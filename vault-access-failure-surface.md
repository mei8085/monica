# 保险库成员移除故障面（Failure Surface）完整分析

## 1. 概述

本文系统梳理成员被软删除式移除保险库后，系统各入口的故障表现。按严重程度分为三类：

| 级别 | 标识 | 说明 |
|------|------|------|
| 🔴 崩溃级 | `ERROR` | 直接触发 PHP 错误 / 异常，页面白屏或接口 500 |
| 🟡 异常级 | `EMPTY` | 读到空值或不一致数据，功能异常但不崩溃 |
| 🟢 放行级 | `PASS` | 权限判断通过，功能基本正常（但逻辑上不该放行） |

> **前提假设**：成员移除采用当前实现方式（软删除"用户本人"的联系人记录，`user_vault` 中间表记录仍存在）。

---

## 2. 🔴 崩溃级故障（直接报错）

### 2.1 保险库 Dashboard - 生活事件（Life Events）

**代码位置**：[VaultController.php L68-L78](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L68-L78)

```php
public function show(Request $request, Vault $vault)
{
    $contact = Auth::user()->getContactInVault($vault);  // 返回 null

    return Inertia::render('Vault/Dashboard/Index', [
        // ...
        'lifeEvents' => ModuleLifeEventViewHelper::data($contact, Auth::user()),
        //     传入 null ↑                           ↓ 期望 Contact 对象
        'lifeMetrics' => VaultLifeMetricsViewHelper::data($vault, Auth::user(), Carbon::now()->year),
    ]);
}
```

**故障机制**：
- `getContactInVault()` 返回 `null`（联系人被软删）
- `ModuleLifeEventViewHelper::data()` 直接使用 `$contact` 调用方法
- 触发：`Call to a member function ... on null`

**故障等级**：🔴 ERROR

---

### 2.2 保险库 Dashboard - 生活指标（Life Metrics）

**代码位置**：[VaultLifeMetricsViewHelper.php L21-L26](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L21-L26)

```php
public static function data(Vault $vault, User $user, int $year): array
{
    $lifeMetrics = $vault->lifeMetrics;
    $contact = $user->getContactInVault($vault);  // null

    $lifeMetricsCollection = $lifeMetrics->map(
        fn (LifeMetric $lifeMetric) => self::dto($lifeMetric, $year, $contact)
        // 传给 dto 的 $contact 是 null ↑
    );
}
```

再看 `dto()` 方法 [VaultLifeMetricsViewHelper.php L38-L42](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L38-L42)：

```php
public static function dto(LifeMetric $lifeMetric, int $year, Contact $contact): array
//                                      类型声明是 Contact ↑
```

**故障机制**：
- PHP 类型声明 `Contact $contact` 会在传入 `null` 时触发类型错误
- 或者方法内部 `$contact->lifeMetrics()` 触发空引用

**故障等级**：🔴 ERROR

---

### 2.3 生活指标增量接口

**代码位置**：[IncrementLifeMetric.php L46-L48](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageLifeMetrics/Services/IncrementLifeMetric.php#L46-L48)

```php
$contact = $this->author->getContactInVault($this->vault);  // null

$contact->lifeMetrics()->save($lifeMetric);
//   ↑ 空引用
```

**故障机制**：
- BaseService 权限验证通过（`user_vault` 记录还在）
- 但 `getContactInVault()` 返回 `null`
- 调用 `->lifeMetrics()` 时崩溃

**故障等级**：🔴 ERROR

---

### 2.4 生活事件 Feed 流接口

**代码位置**：[VaultLifeEventController.php L17-L25](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultLifeEventController.php#L17-L25)

```php
public function show(Request $request, string $vaultId)
{
    $vault = Vault::findOrFail($vaultId);
    $contact = Auth::user()->getContactInVault($vault);  // null

    $timelineEvents = $contact
        ->timelineEvents()  // ← 空引用
        ->orderBy('started_at', 'desc')
        ->paginate(15);
    // ...
}
```

**故障等级**：🔴 ERROR

---

### 2.5 心情记录提交 URL 生成

**代码位置**：[VaultShowViewHelper.php L187-L190](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L187-L190)

```php
'store' => route('contact.mood_tracking_event.store', [
    'vault' => $vault->id,
    'contact' => $user->getContactInVault($vault)->id,
    //                                    ↑ 空引用
]),
```

**故障机制**：
- Dashboard 渲染心情记录模块时
- 直接链式调用 `->id`
- `getContactInVault()` 返回 `null` 后崩溃

**故障等级**：🔴 ERROR

---

### 2.6 联系人详情页 - 操作选项判断

**代码位置**：[ContactShowViewHelper.php L62-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php#L62-L65)（两处相同代码 L117-L118）

```php
'options' => [
    'can_be_archived' => $user->getContactInVault($contact->vault)->id !== $contact->id,
    //                                                         ↑ 空引用
    'can_be_deleted' => $user->getContactInVault($contact->vault)->id !== $contact->id,
],
```

**故障机制**：
- 访问任何联系人详情页都会触发
- `getContactInVault()` 返回 `null`
- 链式调用 `->id` 崩溃

**影响范围**：所有联系人详情页（包括"本人"以外的联系人）

**故障等级**：🔴 ERROR

---

### 2.7 最常访问联系人

**代码位置**：[VaultMostConsultedViewHelper.php L15-L36](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultMostConsultedViewHelper.php#L15-L36)

```php
public static function data(Vault $vault, User $user): Collection
{
    // Step 1: 直接查原生表（不走 Eloquent，不会过滤软删除）
    $records = DB::table('contact_vault_user')
        ->where('vault_id', $vault->id)
        ->where('user_id', $user->id)
        ->orderBy('number_of_views', 'desc')
        ->select('contact_id')
        ->limit(5)
        ->get();

    // Step 2: 逐个查联系人
    foreach ($records as $record) {
        $contact = Contact::find($record->contact_id);
        // 如果是"本人联系人"被软删，这里返回 null

        $contactsCollection->push([
            'id' => $contact->id,  // ← 空引用
            // ...
        ]);
    }
}
```

**故障机制**：
- Step 1 用 `DB::table()` 直接查，可能查到被软删的"本人联系人"
- Step 2 用 `Contact::find()` 会过滤软删除，返回 `null`
- 然后访问 `$contact->id` 崩溃

**触发条件**："本人联系人"恰好排在前 5 个最常访问的位置

**故障等级**：🔴 ERROR（概率性）

---

### 2.8 生活指标 CRUD 控制器

**代码位置**：
- [LifeMetricController.php L29-L33](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageLifeMetrics/Web/Controllers/LifeMetricController.php#L29-L33)（store）
- [LifeMetricController.php L48-L51](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageLifeMetrics/Web/Controllers/LifeMetricController.php#L48-L51)（update）
- [LifeMetricContactController.php L27-L30](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageLifeMetrics/Web/Controllers/LifeMetricContactController.php#L27-L30)（increment）

```php
$contact = Auth::user()->getContactInVault($vault);  // null

return response()->json([
    'data' => VaultLifeMetricsViewHelper::dto($lifeMetric, Carbon::now()->year, $contact),
    //  传给 dto 的是 null ↑
], 201);
```

**故障机制**：`dto()` 方法参数类型声明为 `Contact $contact`，传入 null 触发类型错误

**故障等级**：🔴 ERROR

---

## 3. 🟡 异常级故障（读到空值/不一致）

### 3.1 权限判断系统（全部链路）

**涉及代码**：
- [AuthServiceProvider.php L36-L54](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Providers/AuthServiceProvider.php#L36-L54)（Gate）
- [BaseService.php L190-L200](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Services/BaseService.php#L190-L200)（服务层）
- [VaultHelper.php L40-L53](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Helpers/VaultHelper.php#L40-L53)（缓存层）

**异常表现**：
- 所有权限判断都返回 `true` / 权限值
- 用户仍然被认为是保险库成员
- 但实际上"用户本人联系人"已被软删

**根本原因**：
- 权限判断只查 `user_vault` 中间表
- 软删除联系人不触发数据库级联，`user_vault` 记录仍在
- 不会检查关联联系人的 `deleted_at` 状态

**异常等级**：🟡 EMPTY（逻辑不一致，但不崩溃）

---

### 3.2 保险库列表页

**代码位置**：[VaultIndexViewHelper.php L90-L129](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultIndexViewHelper.php#L90-L129)

```php
public static function data(User $user): array
{
    $vaults = $user->vaults()  // 查 user_vault，仍能查到
        ->where('account_id', $user->account_id)
        ->with('contacts')
        ->get()
        // ...
}
```

**异常表现**：
- 被移除的保险库仍然出现在用户的保险库列表中
- 能看到保险库名称、描述
- 随机联系人预览数量减少（本人联系人被软删了）
- 可以点击进入保险库

**异常等级**：🟡 EMPTY

---

### 3.3 保险库设置 - 成员列表

**代码位置**：[VaultSettingsIndexViewHelper.php L31-L35](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVaultSettings/Web/ViewHelpers/VaultSettingsIndexViewHelper.php#L31-L35)

```php
$usersInVault = $vault->users()->get();  // 仍能查到被移除的用户
```

**异常表现**（从管理者视角）：
- 被移除的用户仍然显示在成员列表中
- 但点击该用户的"本人联系人"链接会 404
- 再次执行"移除"操作会再次软删联系人（无实际效果）

**异常等级**：🟡 EMPTY

---

### 3.4 笔记作者信息显示

**代码位置**：[ModuleNotesViewHelper.php L57](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageNotes/Web/ViewHelpers/ModuleNotesViewHelper.php#L57)

```php
'author' => $note->author ? UserHelper::getInformationAboutContact($note->author, $contact->vault) : null,
```

**异常表现**：
- 如果笔记作者是被移除的成员
- `UserHelper::getInformationAboutContact()` 内部调用 `getContactInVault()` 返回 `null`
- 但该方法有判空，最终返回 `null`
- 笔记作者信息区域显示为空

**UserHelper 的防护**：[UserHelper.php L25-L27](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Helpers/UserHelper.php#L25-L27)
```php
if (! $contact) {
    return null;
}
```

**异常等级**：🟡 EMPTY（有防护，不崩溃）

---

### 3.5 Feed 动态流作者信息

**代码位置**：[ModuleFeedViewHelper.php L104](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L104)

```php
return UserHelper::getInformationAboutContact($author, $vault);
```

**异常表现**：与笔记作者相同，被移除用户的动态会显示作者为空。

**异常等级**：🟡 EMPTY

---

### 3.6 日志帖子的心情关联

**代码位置**：[PostShowViewHelper.php L160-L166](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/PostShowViewHelper.php#L160-L166)

```php
private static function getMood(User $user, Post $post): ?Collection
{
    $contact = $user->getContactInVault($post->journal->vault);

    if (! $contact) {
        return null;  // 有判空
    }
    // ...
}
```

**异常表现**：被移除后，帖子详情页不显示关联心情记录。

**异常等级**：🟡 EMPTY（有防护）

---

### 3.7 收藏功能

**代码位置**：[VaultShowViewHelper.php L99-L116](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L99-L116)

```php
public static function favorites(Vault $vault, User $user): Collection
{
    return $user->contacts()  // contact_vault_user 关联
        ->wherePivot('vault_id', $vault->id)
        ->wherePivot('is_favorite', true)
        ->get();
}
```

**异常表现**：
- 用户收藏的其他联系人正常显示
- 如果用户收藏了"本人联系人"，该条会因为软删除被 Eloquent 自动过滤
- 收藏总数减少 1（如果收藏了自己）

**异常等级**：🟡 EMPTY（轻微）

---

## 4. 🟢 放行级故障（功能正常但逻辑不该放行）

### 4.1 联系人列表页

**代码位置**：[ContactController.php L28-L53](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L28-L53)

```php
public function index(Request $request, Vault $vault)
{
    $contacts = $vault->contacts()
        ->where('listed', true);  // 自动过滤软删除
    // ...
}
```

**放行表现**：
- Gate 判断通过（`user_vault` 记录还在）
- 联系人列表正常显示
- 看不到"本人联系人"（被软删过滤了）
- 其他功能完全正常

**放行原因**：权限判断只查中间表，不检查联系人软删除状态

**放行等级**：🟢 PASS

---

### 4.2 联系人详情页（除操作选项外）

**代码位置**：[ContactController.php L96-L126](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L96-L126)

**放行表现**：
- 能正常访问其他联系人详情（如果 options 部分修好了的话）
- 能查看笔记、标签、分组等所有模块
- 只有 `getContactInVault()->id` 那几行代码会崩溃

**放行等级**：🟢 PASS（部分崩溃）

---

### 4.3 搜索功能

**代码位置**：[VaultContactSearchViewHelper.php L11-L20](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultContactSearchViewHelper.php#L11-L20)

```php
$contacts = Contact::search($term)
    ->where('vault_id', $vault->id)
    ->take(5)
    ->get();
```

**放行表现**：
- 搜索功能完全正常
- 搜不到"本人联系人"（Scout 监听 deleted 事件已移除索引）
- 其他联系人搜索正常

**放行等级**：🟢 PASS

---

### 4.4 标签、分组、公司等保险库级功能

这些功能不依赖 `getContactInVault()`，所以完全正常：
- 标签管理
- 分组管理
- 公司管理
- 文件管理
- 任务管理
- 提醒管理（已被清理）
- 日历视图

**放行等级**：🟢 PASS

---

## 5. 故障面全景图

### 5.1 按入口汇总

| 入口 / 模块 | 故障等级 | 故障类型 | 触发方式 |
|------------|---------|---------|---------|
| 保险库列表 | 🟡 EMPTY | 列表还能看到已移除的保险库 | 直接访问 |
| Dashboard 首页 | 🔴 ERROR | lifeEvents / lifeMetrics 传 null | 进入 Dashboard |
| 心情记录模块 | 🔴 ERROR | URL 生成时空引用 | Dashboard / 报告 |
| 生活指标模块 | 🔴 ERROR | dto 方法类型不匹配 | Dashboard / 详情 |
| 生活事件 Feed | 🔴 ERROR | timelineEvents() 空引用 | 访问生活事件页 |
| 联系人列表 | 🟢 PASS | 功能正常但不该放行 | 浏览联系人 |
| 联系人详情 | 🔴 ERROR | options 处空引用 | 查看任何联系人 |
| 笔记模块 | 🟡 EMPTY | 作者信息显示为空 | 查看笔记 |
| Feed 动态 | 🟡 EMPTY | 作者信息显示为空 | 查看动态 |
| 收藏 | 🟡 EMPTY | 可能少一个（本人） | Dashboard |
| 最常访问 | 🔴 ERROR | 概率性空引用 | 搜索 / Dashboard |
| 搜索 | 🟢 PASS | 功能正常 | 搜索联系人 |
| 成员设置（管理者视角） | 🟡 EMPTY | 被移除用户仍在列表 | 查看成员列表 |
| 标签/分组/公司 | 🟢 PASS | 功能正常 | 日常使用 |
| 生活指标增量接口 | 🔴 ERROR | 服务层空引用 | 点击指标按钮 |
| 心情记录提交 | 🟡 ~ 🔴 | 取决于入口是否先崩溃 | 提交心情 |

---

### 5.2 故障传播链

```
RemoveVaultAccess（软删除联系人）
│
├─ 第一层：权限层（全部放行）
│   ├─ Gate: vault-viewer → ✅ true
│   ├─ Gate: vault-editor → ✅ true
│   ├─ BaseService 验证 → ✅ 通过
│   └─ VaultHelper 缓存 → ✅ 返回权限值（5秒缓存）
│
├─ 第二层：入口层（部分放行）
│   ├─ 保险库列表 → ✅ 还能看到
│   ├─ Dashboard 入口 → ✅ Gate 通过
│   ├─ 联系人列表 → ✅ 正常显示
│   ├─ 搜索 → ✅ 正常工作
│   └─ 设置页 → ✅ 还能进
│
└─ 第三层：功能层（开始崩溃）
    ├─ Dashboard: lifeEvents → 💥 ERROR
    ├─ Dashboard: lifeMetrics → 💥 ERROR
    ├─ Dashboard: 心情记录 URL → 💥 ERROR
    ├─ 联系人详情: options → 💥 ERROR
    ├─ 最常访问 → 💥 概率性 ERROR
    ├─ 笔记作者 → ⚠️ 显示为空
    ├─ Feed 作者 → ⚠️ 显示为空
    └─ 其他模块（标签/分组等）→ ✅ 正常
```

---

## 6. 根因分析

### 6.1 为什么会产生这么多故障？

**核心矛盾**：权限系统和"联系人系统"是两套独立的判断逻辑。

```
权限判断（user_vault 中间表）
    ↓ 只查这张表
    ↓ 软删除不影响它
    ↓ 认为用户还有权限
                    → 不一致 → 后续代码假设 getContactInVault 一定能取到
联系人映射（contacts 表软删除）
    ↓ deleted_at 被设置
    ↓ 所有 Contact 查询都过滤
    ↓ getContactInVault() 返回 null
```

**设计假设**：代码中大量使用 `getContactInVault()->xxx` 的链式调用，隐含了一个假设 —— **"只要有权限，就一定能取到本人联系人"**。软删除式移除打破了这个假设。

### 6.2 为什么用软删除？

可能的设计考量：
1. 保留历史数据（动态流、笔记作者等）
2. 可恢复（撤销移除）
3. 符合 Laravel 最佳实践

但这套实现**没有配套更新权限判断逻辑**，导致了"权限还在但人没了"的不一致状态。

---

## 7. 修复建议优先级

### P0（立即修复）- 崩溃级故障

1. **修复权限判断**：Gate 和 BaseService 中加入对联系人软删除状态的检查
2. **或者改用硬删除**：移除用户时 `forceDelete()` 联系人，触发级联删除 `user_vault` 记录
3. **为所有 getContactInVault 调用加判空**：快速止血

### P1（尽快修复）- 体验异常

1. 成员列表显示异常
2. 保险库列表显示异常
3. 笔记/Feed 作者信息缺失

### P2（规划修复）- 设计层面

1. 统一权限判断的数据源（只以 `user_vault` 表为准，还是要检查联系人状态）
2. 明确"软删除联系人"的语义（是移除权限，还是仅隐藏）
3. 补充单元测试覆盖移除后的各种入口

---

## 8. 关键文件索引

| 文件 | 故障类型 | 说明 |
|------|---------|------|
| [RemoveVaultAccess.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php) | - | 移除操作的源头 |
| [User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/User.php) | - | getContactInVault 方法定义 |
| [AuthServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Providers/AuthServiceProvider.php) | 🟡 PASS | Gate 权限定义 |
| [BaseService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Services/BaseService.php) | 🟡 PASS | 服务层权限验证 |
| [VaultHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Helpers/VaultHelper.php) | 🟡 PASS | 权限缓存 |
| [UserHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Helpers/UserHelper.php) | 🟡 EMPTY | 联系人信息辅助（有判空） |
| [VaultController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php) | 🔴 ERROR | Dashboard 入口 |
| [VaultShowViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php) | 🔴 ERROR | 心情记录 URL 生成 |
| [ContactShowViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php) | 🔴 ERROR | 联系人详情 options |
| [VaultLifeMetricsViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php) | 🔴 ERROR | 生活指标 dto 方法 |
| [VaultLifeEventController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultLifeEventController.php) | 🔴 ERROR | 生活事件 Feed |
| [IncrementLifeMetric.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageLifeMetrics/Services/IncrementLifeMetric.php) | 🔴 ERROR | 生活指标增量 |
| [VaultMostConsultedViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultMostConsultedViewHelper.php) | 🔴 ERROR | 最常访问（概率性） |
| [VaultSettingsIndexViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVaultSettings/Web/ViewHelpers/VaultSettingsIndexViewHelper.php) | 🟡 EMPTY | 成员列表（管理者视角） |
| [ModuleNotesViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageNotes/Web/ViewHelpers/ModuleNotesViewHelper.php) | 🟡 EMPTY | 笔记作者显示 |
| [ModuleFeedViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php) | 🟡 EMPTY | Feed 作者显示 |
| [PostShowViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/PostShowViewHelper.php) | 🟡 EMPTY | 帖子心情关联 |
| [VaultContactSearchViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultContactSearchViewHelper.php) | 🟢 PASS | 搜索功能 |
