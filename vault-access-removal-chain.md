# 保险库成员移除链路与软删除影响分析

## 1. 概述

本文完整梳理成员被移除保险库后，"用户在保险库中的本人联系人映射"的失效时机、权限判断链路、各展示入口的状态，以及软删除（SoftDeletes）对整条链路的影响。

---

## 2. 核心概念回顾

### 2.1 "用户在保险库中的本人联系人映射"是什么

Monica 采用"用户即联系人"模式：每个用户在每个有权限的保险库中，都会创建一个对应的"自己"的联系人记录。这个映射关系存储在 `user_vault` 中间表中。

**建立时机** [CreateVault.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Services/CreateVault.php#L80-L94)：
```php
private function createUserContact(): void
{
    // 1. 创建"用户本人"的联系人
    $contact = Contact::create([
        'vault_id' => $this->vault->id,
        'first_name' => $this->author->first_name,
        'last_name' => $this->author->last_name,
        'can_be_deleted' => false,  // 标记为不可删除
        'template_id' => $this->vault->default_template_id,
    ]);

    // 2. 建立 user_vault 关联，关联到这个联系人
    $this->vault->users()->save($this->author, [
        'permission' => Vault::PERMISSION_MANAGE,
        'contact_id' => $contact->id,  // ★ 关键：映射关系
    ]);
}
```

**数据结构** [2020_04_25_133132_create_contacts_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L61-L67)：
```php
Schema::create('user_vault', function (Blueprint $table) {
    $table->foreignIdFor(Vault::class)->constrained()->cascadeOnDelete();
    $table->foreignIdFor(User::class)->constrained()->cascadeOnDelete();
    $table->foreignIdFor(Contact::class)->constrained()->cascadeOnDelete();  // ★ 级联删除
    $table->integer('permission');
    $table->timestamps();
});
```

### 2.2 软删除机制

**Contact 模型启用了 SoftDeletes** [Contact.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Contact.php#L19-L29)：
```php
use SoftDeletes;

// 数据库表有 deleted_at 字段 [迁移文件 L47]
$table->softDeletes();
```

这意味着调用 `$contact->delete()` 时：
- **不是真正删除**记录，而是设置 `deleted_at` 字段为当前时间
- 所有常规查询（`where`、`find`、`findOrFail`）会自动过滤掉 `deleted_at IS NOT NULL` 的记录
- 级联删除**不会触发**，因为记录本身没有被 `DELETE`
- 只有 `forceDelete()` 才会真正删除并触发数据库级联

> ⚠️ **关键发现**：移除成员时调用的是软删除，所以 `user_vault` 表的记录**不会被数据库级联删除**，仍然存在于数据库中！

---

## 3. 移除操作的完整链路

### 3.1 移除操作的代码路径

[RemoveVaultAccess.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php#L48-L95)：
```php
public function execute(array $data): void
{
    $this->data = $data;
    $this->validate();
    $this->remove();                              // Step 1
    $this->removeAllRemindersForThisUserInThisVault();  // Step 2
}
```

### 3.2 Step 1: 删除关联联系人（触发软删除）

```php
private function remove(): void
{
    $vault = $this->user->vaults()
        ->wherePivot('vault_id', $this->vault->id)
        ->first();

    if ($vault !== null) {
        // ★ 这是软删除！deleted_at 被设置，记录仍在数据库
        Contact::find($vault->pivot->contact_id)->delete();
    }
}
```

**执行后各表状态**：

| 表 | 状态变化 | 记录是否存在 |
|----|---------|------------|
| `contacts` | `deleted_at` 被设置为当前时间 | ✅ 仍存在（软删除） |
| `user_vault` | 无变化（软删除不触发级联） | ✅ 仍存在，`contact_id` 指向被软删的联系人 |
| `contact_vault_user` | 无变化 | ✅ 仍存在 |
| `contact_reminder_scheduled` | 待 Step 2 处理 | - |

### 3.3 Step 2: 清理提醒调度

```php
private function removeAllRemindersForThisUserInThisVault(): void
{
    $this->user->notificationChannels->each(function (UserNotificationChannel $channel) {
        $channel->contactReminders->each(function ($reminder) {
            $reminder->pivot->delete();  // 删除 contact_reminder_scheduled 记录
        });
    });
}
```

> ⚠️ 这个方法名和注释说"该保险库下的"，但实际清掉了用户**所有**保险库的提醒。

---

## 4. 权限判断链路会读到什么状态

权限判断有两条并行链路：**Laravel Gate（控制器层）** 和 **BaseService（服务层）**。

### 4.1 链路一：Laravel Gate（控制器层）

**定义位置** [AuthServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Providers/AuthServiceProvider.php#L36-L54)：
```php
Gate::define('vault-viewer', function (User $user, $vault): bool {
    return $user->vaults()
        ->wherePivotIn('vault_id', [static::id($vault)])
        ->exists();
});

Gate::define('vault-editor', function (User $user, $vault): bool {
    return $user->vaults()
        ->wherePivotIn('vault_id', [static::id($vault)])
        ->wherePivot('permission', '<=', 200)
        ->exists();
});

Gate::define('vault-manager', function (User $user, $vault): bool {
    return $user->vaults()
        ->wherePivotIn('vault_id', [static::id($vault)])
        ->wherePivot('permission', '<=', 100)
        ->exists();
});
```

**关键查询**：`$user->vaults()->wherePivotIn('vault_id', ...)->exists()`

这里的核心是 **`$user->vaults()` 关联** [User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/User.php#L186-L191)：
```php
public function vaults(): BelongsToMany
{
    return $this->belongsToMany(Vault::class)
        ->withPivot('permission', 'contact_id')
        ->withTimestamps();
}
```

> **这个关联查询的是 `user_vault` 中间表**，**不会** 自动过滤 `contacts.deleted_at`。因为中间表本身存在记录，且 Eloquent 的 `BelongsToMany` 不会自动关联检查 `Contact` 的软删除状态。

**结论**：Gate 判断时，**`user_vault` 记录仍然存在**，所以 **Gate 会返回 `true`（用户仍有权限）**！

### 4.2 链路二：BaseService 验证（服务层）

[BaseService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Services/BaseService.php#L190-L200)：
```php
public function validateUserPermissionInVault(int $permission): void
{
    $exists = $this->author->vaults()
        ->where('vaults.id', $this->vault->id)
        ->wherePivot('permission', '<=', $permission)
        ->exists();

    if (! $exists) {
        throw new NotEnoughPermissionException;
    }
}
```

**同样的查询方式**：`$this->author->vaults()->where(...)->exists()`

**结论**：与 Gate 一样，**也会返回 `true`（用户仍"有权限"）**。

### 4.3 链路三：VaultHelper（缓存层）

[VaultHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Helpers/VaultHelper.php#L40-L53)：
```php
public static function getPermission(User $user, Vault $vault): ?int
{
    return Cache::store('array')
        ->remember("Permission:{$user->id}:{$vault->id}", 5, fn () => self::internalGetPermission($user, $vault));
}

private static function internalGetPermission(User $user, Vault $vault): ?int
{
    $entry = $user->vaults()
        ->wherePivot('vault_id', $vault->id)
        ->first();

    return $entry !== null ? $entry->pivot->permission : null;
}
```

**结论**：
1. 同样通过 `$user->vaults()` 查询，**会查到被软删后的关联记录**
2. 还会缓存 5 秒，即使后续有其他操作，缓存也会继续返回旧权限

### 4.4 权限判断小结

| 判断链路 | 代码位置 | 移除后的返回值 | 原因 |
|---------|---------|--------------|------|
| Gate: vault-viewer | AuthServiceProvider L36-40 | ✅ true | user_vault 记录仍存在 |
| Gate: vault-editor | AuthServiceProvider L42-47 | ✅ true（编辑者/管理者） | 同上 |
| Gate: vault-manager | AuthServiceProvider L49-54 | ✅ true（管理者） | 同上 |
| BaseService 权限验证 | BaseService.php L190-200 | ✅ 通过 | 同上 |
| VaultHelper::getPermission | VaultHelper.php L40-53 | ✅ 返回权限值（非 null） | 同上，还有 5 秒缓存 |

> 🔴 **重大问题**：成员移除后（软删除联系人），所有权限判断链路**仍然认为用户有权限**！用户仍然可以访问保险库、查看联系人、执行操作（直到后续某个环节失败）。

---

## 5. 各展示入口读取到什么状态

接下来分析用户在各入口会看到什么。

### 5.1 入口一：保险库列表页（Vault Index）

**路由**：`vault.index`
**代码路径**：
[VaultController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L33-L39) →
[VaultIndexViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultIndexViewHelper.php#L90-L129)

```php
public static function data(User $user): array
{
    $vaults = $user->vaults()
        ->where('account_id', $user->account_id)
        ->with('contacts')
        ->get()
        // ...
}
```

**用户看到的**：
- ✅ **保险库仍然出现在列表中**（因为 `user_vaults` 记录还在）
- 能看到保险库的名称、描述
- 但"随机联系人预览"使用 `$vault->contacts`，该关联会过滤软删除记录，所以联系人数量会减少（用户本人的联系人被软删了）

### 5.2 入口二：保险库 Dashboard（Vault Show）

**路由**：`vault.show`
**代码路径**：
[VaultController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L66-L89)

```php
public function show(Request $request, Vault $vault)
{
    // ★ 这里会调用 getContactInVault，下面详细分析
    $contact = Auth::user()->getContactInVault($vault);

    return Inertia::render('Vault/Dashboard/Index', [
        'lastUpdatedContacts' => VaultShowViewHelper::lastUpdatedContacts($vault),
        'upcomingReminders' => VaultShowViewHelper::upcomingReminders($vault, Auth::user()),
        'favorites' => VaultShowViewHelper::favorites($vault, Auth::user()),
        // ...
        'lifeEvents' => ModuleLifeEventViewHelper::data($contact, Auth::user()),  // ★ 使用了 $contact
    ]);
}
```

#### 5.2.1 `getContactInVault` 的行为

[User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/User.php#L267-L282)：
```php
public function getContactInVault(Vault $vault): ?Contact
{
    // Step 1: 查 user_vault 关联 —— ✅ 仍然能查到
    $entry = $this->vaults()
        ->wherePivot('vault_id', $vault->id)
        ->first();

    if ($entry === null) {
        return null;
    }

    try {
        // Step 2: 用 pivot 中的 contact_id 查联系人
        // ★ 这里会触发软删除过滤！
        return Contact::findOrFail($entry->pivot->contact_id);
    } catch (ModelNotFoundException) {
        return null;  // ★ 软删除后走到这里，返回 null
    }
}
```

**关键差异**：
- Step 1 查 `user_vault` → ✅ 查到
- Step 2 用 `findOrFail` 查联系人 → ❌ **查不到（软删除过滤）**，返回 `null`

#### 5.2.2 Dashboard 各组件行为

| 组件 | 代码位置 | 移除后状态 |
|------|---------|-----------|
| `lastUpdatedContacts` | VaultShowViewHelper L17-34 | ✅ 正常显示其他联系人 |
| `upcomingReminders` | VaultShowViewHelper L36-97 | ✅ 正常（Step 2 已清理） |
| `favorites` | VaultShowViewHelper L99-116 | ⚠️ 下面详细分析 |
| `dueTasks` | VaultShowViewHelper L118+ | ✅ 正常显示 |
| `lifeEvents` | 传入 `$contact`（值为 `null`） | 🔴 **可能崩溃**，因为 ModuleLifeEventViewHelper 期望传入 Contact 对象 |
| `lifeMetrics` | VaultLifeMetricsViewHelper | 依赖 `$contact` → 🔴 可能崩溃 |

> 🔴 **问题**：`getContactInVault()` 返回 `null` 后，后续代码没有判空，直接传给视图 Helper，可能导致 "Call to a member function on null" 错误。

### 5.3 入口三：联系人列表页（Contact Index）

**路由**：`contact.index`
**代码路径**：
[ContactController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L28-L53)

```php
public function index(Request $request, Vault $vault)
{
    $contacts = $vault->contacts()
        ->where('listed', true);  // ★ 会过滤软删除
    // ...
}
```

**用户看到的**：
- ✅ 能正常访问列表页（Gate 通过）
- ✅ 联系人列表正常显示（软删除的"本人联系人"不显示）
- ✅ 标签统计正常（标签统计基于 `$vault->labels()`）

### 5.4 入口四：联系人详情页（Contact Show）

**路由**：`contact.show`
**代码路径**：
[ContactController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L96-L126)

```php
public function show(Request $request, string $vaultId, string $contactId)
{
    $vault = Vault::findOrFail($vaultId);
    $contact = Contact::with([...])->findOrFail($contactId);  // ✅ 正常
    // ...
}
```

**关键判空**：[ContactShowViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php#L62-L65)：
```php
'options' => [
    'can_be_archived' => $user->getContactInVault($contact->vault)->id !== $contact->id,
    'can_be_deleted' => $user->getContactInVault($contact->vault)->id !== $contact->id,
],
```

> 🔴 **问题**：`getContactInVault()` 返回 `null`，调用 `->id` 会触发错误。

### 5.5 入口五：收藏（Favorites）

**代码路径**：[VaultShowViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L99-L116)：
```php
public static function favorites(Vault $vault, User $user): Collection
{
    return $user->contacts()  // ★ 这是 contact_vault_user 关联
        ->wherePivot('vault_id', $vault->id)
        ->wherePivot('is_favorite', true)
        ->get();
}
```

**关联定义** [User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/User.php#L198-L203)：
```php
public function contacts(): BelongsToMany
{
    return $this->belongsToMany(Contact::class, 'contact_vault_user')
        ->withPivot('is_favorite')
        ->withTimestamps();
}
```

**行为分析**：
- `belongsToMany` 会查询 `contact_vault_user` → `contacts`
- 因为 Contact 使用了 SoftDeletes，Eloquent **会自动过滤**软删除的联系人
- 但对于用户收藏的**其他**联系人，`contact_vault_user` 记录仍在，所以 ✅ 正常显示其他收藏

**结论**：收藏功能**基本正常**，仅会丢失用户自己"本人联系人"的收藏（如果有的话）。

### 5.6 入口六：最常访问（Most Consulted）

**代码路径**：[VaultMostConsultedViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultMostConsultedViewHelper.php#L13-L39)：
```php
public static function data(Vault $vault, User $user): Collection
{
    // Step 1: 直接查 contact_vault_user 原生表（不走 Eloquent 关联）
    $records = DB::table('contact_vault_user')
        ->where('vault_id', $vault->id)
        ->where('user_id', $user->id)
        ->orderBy('number_of_views', 'desc')
        ->select('contact_id')
        ->limit(5)
        ->get()
        ->toArray();

    // Step 2: 逐个用 Contact::find 查
    foreach ($records as $record) {
        $contact = Contact::find($record->contact_id);  // ★ 会过滤软删除
        // ...
    }
}
```

**行为**：
- Step 1 可能查到"本人联系人"的查看记录
- Step 2 中 `Contact::find` 会返回 `null`（软删除过滤）
- 后续访问 `$contact->id` 等属性会报错

> 🔴 **问题**：Step 2 没有判空，可能崩溃。

### 5.7 入口七：搜索（Search）

#### 5.7.1 Scout 全文搜索

[VaultContactSearchViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultContactSearchViewHelper.php#L11-L31)：
```php
public static function data(Vault $vault, string $term): Collection
{
    $contacts = Contact::search($term)
        ->where('vault_id', $vault->id)
        ->take(5)
        ->get();
    // ...
}
```

**Scout 搜索机制**：
- 搜索走的是搜索索引（如 Meilisearch、Algolia 等）
- `shouldBeSearchable()` 决定是否索引 [Contact.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Contact.php#L104-L107)：
  ```php
  public function shouldBeSearchable()
  {
      return $this->listed;  // 仅检查 listed 字段，不检查 deleted_at
  }
  ```
- 但 Laravel Scout 会自动监听 `deleted` 事件，软删除时会从搜索索引中**移除**文档

**结论**：
- ✅ 搜索结果中不会出现被软删除的"本人联系人"
- ✅ 搜索其他联系人正常

#### 5.7.2 通用搜索入口

[VaultSearchIndexViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultSearchIndexViewHelper.php) 会综合搜索笔记、标签、分组等，行为类似。

### 5.8 入口八：保险库设置页（成员管理）

保险库管理者在成员列表中看到什么？

需要通过 `$vault->users()` 查询，定义在 [Vault.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Vault.php#L129-L134)：
```php
public function users(): BelongsToMany
{
    return $this->belongsToMany(User::class)
        ->withPivot('permission', 'contact_id')
        ->withTimestamps();
}
```

**管理者看到的**：
- 🔴 **被移除的用户仍然显示在成员列表中**（因为 `user_vault` 记录还在）
- 但点击该用户的"本人联系人"时会失败（联系人已被软删）

---

## 6. 失效时机完整梳理

### 6.1 各环节失效时序图

```
执行 RemoveVaultAccess::execute()
│
├─ remove() —— 软删除联系人
│   ├─ contacts.deleted_at = NOW()
│   ├─ user_vault 记录 → 仍然存在（⚠️ 关键不一致点）
│   └─ Scout 索引自动移除该联系人（模型事件）
│
└─ removeAllRemindersForThisUserInThisVault()
    └─ 清空 contact_reminder_scheduled（用户所有保险库的）

● 权限判断（所有链路）→ 仍然通过 ❌（不一致）
│
● 保险库列表 → 保险库仍可见 ❌（不一致）
│
● 保险库 Dashboard
│   ├─ 权限检查 Gate::authorize('view', $vault) → 通过
│   ├─ getContactInVault() → 返回 null
│   ├─ lifeEvents / lifeMetrics → 可能崩溃 ❌
│   ├─ 其他组件（联系人列表、任务、收藏）→ 基本正常
│   └─ VaultHelper 权限缓存 → 5 秒内继续返回旧权限值
│
● 联系人列表 → 正常（软删除自动过滤）
│
● 联系人详情 → 访问 getContactInVault()->id → 可能崩溃 ❌
│
● 最常访问 → Contact::find 返回 null → 可能崩溃 ❌
│
● 搜索 → 正常（Scout 已移除软删除记录）
│
● 管理者成员列表 → 被移除用户仍显示 ❌（不一致）
│
● 用户本人"联系人映射" → 通过 Contact::find 查询时失效
```

### 6.2 真正的"完全失效"条件

| 失效条件 | 何时达成 |
|---------|---------|
| 权限 Gate 返回 false | **永不**（直到 user_vault 记录被物理删除） |
| 保险库从列表中消失 | **永不**（同上） |
| getContactInVault() 返回 null | ✅ **立即**（软删除后 findOrFail 查不到） |
| "本人联系人" 从列表消失 | ✅ **立即**（软删除自动过滤） |
| 从搜索索引中移除 | ✅ **立即**（Scout 监听 deleted 事件） |
| 提醒停止发送 | ✅ **立即**（Step 2 清理了调度） |
| 收藏记录消失 | ⚠️ 仅本人联系人的收藏消失，其他正常 |
| 管理者看到成员被移除 | ❌ **永不**（user_vault 仍存在） |

---

## 7. 核心问题总结

### 问题 1：权限不一致（最严重）

**现象**：移除成员后，用户仍然可以访问保险库、查看/编辑联系人。

**根本原因**：
- 移除操作使用软删除（`delete()`），不触发数据库级联
- `user_vault` 记录仍然存在于数据库
- 所有权限判断（Gate、BaseService、VaultHelper）都只查 `user_vault` 表，不检查关联联系人的 `deleted_at` 状态

**修复建议**：
```php
// 方案 A：在权限判断时加入软删除检查
Gate::define('vault-viewer', function (User $user, $vault): bool {
    return $user->vaults()
        ->wherePivotIn('vault_id', [static::id($vault)])
        // 加入对联系人软删除的检查
        ->whereHasMorph('pivot.contact')  // 或用 join 方式
        ->exists();
});

// 方案 B：移除时使用 forceDelete 硬删除
private function remove(): void
{
    $vault = $this->user->vaults()
        ->wherePivot('vault_id', $this->vault->id)
        ->first();

    if ($vault !== null) {
        $contact = Contact::find($vault->pivot->contact_id);
        DB::transaction(function () use ($contact) {
            // 先物理删除 user_vault 记录
            $contact->vault->users()->detach($contact->vault->pivot->user_id);
            // 再硬删除联系人（触发级联清理其他关联）
            $contact->forceDelete();
        });
    }
}
```

### 问题 2：缺少事务保护

**现象**：如果 Step 1 成功、Step 2 失败，权限已移除但提醒仍会触发。

**修复建议**：使用 `DB::transaction()` 包裹整个移除逻辑。

### 问题 3：视图层未判空（getContactInVault 返回 null）

**现象**：Dashboard、联系人详情页等多处直接使用 `getContactInVault()->id`，可能触发空引用错误。

**根本原因**：[ContactShowViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php#L62-L65)：
```php
'can_be_archived' => $user->getContactInVault($contact->vault)->id !== $contact->id,
```

**修复建议**：所有调用 `getContactInVault()` 的地方都需要判空：
```php
$userContact = $user->getContactInVault($contact->vault);
'can_be_archived' => $userContact ? $userContact->id !== $contact->id : true,
```

### 问题 4：提醒清理范围过大

**现象**：移除用户某个保险库的权限，会清掉该用户所有保险库的提醒。

**根本原因**：[RemoveVaultAccess.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php#L88-L95) 没有按 `vault_id` 过滤。

---

## 8. 关键文件索引

| 文件 | 功能 |
|------|------|
| [RemoveVaultAccess.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php) | 移除保险库成员的核心服务 |
| [Contact.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Contact.php) | 联系人模型（使用 SoftDeletes） |
| [User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/User.php) | 用户模型（vaults 关联、getContactInVault 方法） |
| [AuthServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Providers/AuthServiceProvider.php) | Gate 权限定义（vault-viewer/editor/manager） |
| [BaseService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Services/BaseService.php) | 服务层统一权限验证 |
| [VaultHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Helpers/VaultHelper.php) | 权限获取（带 5 秒缓存） |
| [VaultController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php) | 保险库 Dashboard 入口 |
| [ContactController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php) | 联系人列表/详情入口 |
| [VaultIndexViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultIndexViewHelper.php) | 保险库列表/Dashboard 布局数据 |
| [VaultShowViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php) | Dashboard 组件数据（收藏、任务等） |
| [ContactShowViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php) | 联系人详情页（调用 getContactInVault） |
| [VaultMostConsultedViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultMostConsultedViewHelper.php) | 最常访问联系人 |
| [2020_04_25_133132_create_contacts_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/database/migrations/2020_04_25_133132_create_contacts_table.php) | contacts、user_vault、contact_vault_user 表结构 |
