# Monica Vault 邀请协作者与赋权流程分析

## 一、整体架构概览

Monica 的 Vault 协作体系涉及两个层级的邀请与赋权：

1. **账户层级**：管理员邀请用户加入 Account（系统级别）
2. **Vault 层级**：Vault 管理员将同 Account 下的用户添加到 Vault 并赋予权限

核心数据表关系：

```
users ──多对多──> vaults (通过 user_vault 中间表)
                中间表字段: vault_id, user_id, contact_id, permission, timestamps

contacts ──多对多──> users (通过 contact_vault_user 中间表)
                中间表字段: contact_id, vault_id, user_id, number_of_views, is_favorite, timestamps
```

---

## 二、邀请链接的生成（账户层级）

### 2.1 入口

管理员在 **Settings > Users** 页面点击"Invite user"，前端页面为 [Create.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/resources/js/Pages/Settings/Users/Create.vue)。

### 2.2 服务层：InviteUser

文件：[InviteUser.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Settings/ManageUsers/Services/InviteUser.php)

**流程：**

1. **权限校验**：调用者必须是 `author_must_belong_to_account` + `author_must_be_account_administrator`（即账户管理员）
2. **创建 User 记录**：

```php
User::create([
    'account_id'       => $this->data['account_id'],
    'email'            => $this->data['email'],
    'invitation_code'  => (string) Str::uuid(),  // 核心：生成 UUID 作为邀请码
    'is_account_administrator' => $this->data['is_administrator'],
]);
```

关键点：此时用户记录已创建，但 `invitation_accepted_at` 为 null，`password` 为空，用户还不能登录。

3. **发送邀请邮件**：通过 [UserInvited.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Mail/UserInvited.php) 发送异步邮件

```php
$invitationRoute = route('invitation.show', [
    'code' => $this->invitedUser->invitation_code,
]);
```

邮件模板为 [invitation.blade.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/resources/views/emails/user/invitation.blade.php)，包含一个链接按钮，指向 `/invitation/{code}`。

### 2.3 邀请链接的消费

路由定义在 [web.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/routes/web.php#L174-L175)：

```
GET  /invitation/{code}  → AcceptInvitationController::show   (展示接受邀请页面)
POST /invitation          → AcceptInvitationController::store  (提交注册信息)
```

控制器：[AcceptInvitationController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Http/Controllers/Auth/AcceptInvitationController.php)

**show 方法**：根据 `invitation_code` 查找用户，且 `invitation_accepted_at` 必须为 null，否则重定向到首页。

**store 方法**：调用 [AcceptInvitation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Settings/ManageUsers/Services/AcceptInvitation.php) 服务：

```php
private function updateUser(): void
{
    $this->user->is_account_administrator = false;  // 强制设为非管理员
    $this->user->first_name = $this->data['first_name'];
    $this->user->last_name = $this->data['last_name'];
    $this->user->invitation_accepted_at = Carbon::now();  // 标记邀请已接受
    $this->user->email_verified_at = Carbon::now();       // 自动验证邮箱
    $this->user->password = Hash::make($this->data['password']);
    $this->user->save();
}
```

完成后自动登录用户（`Auth::login($user)`）。

### 2.4 流程图

```
管理员点击邀请 → InviteUser::execute()
                   ├─ 创建 User (含 invitation_code=UUID, invitation_accepted_at=null)
                   └─ 发送邮件 (含链接 /invitation/{code})

被邀请者点击链接 → AcceptInvitationController::show()
                   └─ 展示注册表单

被邀请者填写信息 → AcceptInvitationController::store()
                   └─ AcceptInvitation::execute()
                       ├─ 根据 invitation_code 找到用户
                       ├─ 更新姓名、密码、invitation_accepted_at
                       └─ 创建默认通知渠道
```

> **重要**：InviteUser 是**账户层级**的邀请，只是让用户加入 Account。用户此时还不属于任何 Vault。

---

## 三、Vault 成员加入与数据写入

### 3.1 入口

Vault 管理员在 **Vault Settings > Users** 页面操作，前端组件为 [Users.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/resources/js/Pages/Vault/Settings/Partials/Users.vue)。

页面显示两类用户列表：
- `users_in_account`：同 Account 下尚未加入该 Vault 的用户
- `users_in_vault`：已在 Vault 中的用户

### 3.2 路由

定义在 [web.php#L485-L491](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/routes/web.php#L485-L491)，所有 vault settings 路由受 `can:vault-manager,vault` 中间件保护：

```
POST   /vaults/{vault}/settings/users        → VaultSettingsUserController::store   (添加用户)
PUT    /vaults/{vault}/settings/users/{user}  → VaultSettingsUserController::update (修改权限)
DELETE /vaults/{vault}/settings/users/{user}  → VaultSettingsUserController::destroy(移除用户)
```

控制器：[VaultSettingsUserController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVaultSettings/Web/Controllers/VaultSettingsUserController.php)

### 3.3 添加用户到 Vault：GrantVaultAccessToUser

文件：[GrantVaultAccessToUser.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVaultSettings/Services/GrantVaultAccessToUser.php)

**流程：**

1. **权限校验**：`author_must_be_vault_manager`（调用者必须是该 Vault 的 Manager）
2. **验证目标用户**：目标用户必须属于同一个 Account，且不能是调用者自己（否则抛出 `SameUserException`）
3. **核心赋权逻辑**（`grant` 方法）：

```php
private function grant(): void
{
    // 1. 为该用户在 Vault 中创建一个 Contact 记录
    $contact = Contact::create([
        'vault_id'      => $this->vault->id,
        'first_name'    => $this->user->first_name,
        'last_name'     => $this->user->last_name,
        'can_be_deleted' => false,              // 该 Contact 不可被删除
        'template_id'   => $this->vault->default_template_id,
    ]);

    // 2. 写入 user_vault 中间表
    $this->vault->users()->save($this->user, [
        'permission' => $this->data['permission'],
        'contact_id' => $contact->id,           // 关联到刚创建的 Contact
    ]);
}
```

4. **调度提醒**（`scheduleContactReminders` 方法）：为新用户调度该 Vault 中所有联系人的提醒

```php
$contactIds = $this->vault->contacts->pluck('id')->toArray();
$contactReminders = ContactReminder::whereIn('contact_id', $contactIds)->get();
// 遍历所有提醒，为该用户调度（跳过已触发的一次性提醒）
```

### 3.4 数据库写入

中间表 `user_vault` 的结构（来自迁移文件 [2020_04_25_133132_create_contacts_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L61-L67)）：

```php
Schema::create('user_vault', function (Blueprint $table) {
    $table->foreignIdFor(Vault::class)->constrained()->cascadeOnDelete();
    $table->foreignIdFor(User::class)->constrained()->cascadeOnDelete();
    $table->foreignIdFor(Contact::class)->constrained()->cascadeOnDelete();  // 关键外键
    $table->integer('permission');
    $table->timestamps();
});
```

**核心设计**：每条 `user_vault` 记录都关联一个 `contact_id`，该 Contact 代表此用户在此 Vault 中的"身份"。这就是为什么移除用户时删除 Contact 就能级联删除中间表记录。

### 3.5 创建 Vault 时自动添加创建者

[CreateVault.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVault/Services/CreateVault.php#L80-L94) 中，创建 Vault 的同时会自动将创建者加入：

```php
private function createUserContact(): void
{
    $contact = Contact::create([
        'vault_id'      => $this->vault->id,
        'first_name'    => $this->author->first_name,
        'last_name'     => $this->author->last_name,
        'can_be_deleted' => false,
        'template_id'   => $this->vault->default_template_id,
    ]);

    $this->vault->users()->save($this->author, [
        'permission' => Vault::PERMISSION_MANAGE,  // 创建者自动成为 Manager
        'contact_id' => $contact->id,
    ]);
}
```

---

## 四、权限体系与访问判断

### 4.1 权限常量定义

在 [Vault.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Models/Vault.php#L19-L23)：

```php
public const PERMISSION_MANAGE = 100;   // 管理员
public const PERMISSION_EDIT   = 200;   // 编辑者
public const PERMISSION_VIEW   = 300;   // 查看者
```

**数值越小权限越大**，这种设计使得权限判断可以用 `<=` 运算符：
- Manager (100) `<=` MANAGE/EDIT/VIEW → 拥有所有权限
- Editor (200) `<=` EDIT/VIEW → 拥有编辑和查看权限
- Viewer (300) `<=` VIEW → 只有查看权限

### 4.2 权限含义

| 权限 | 值 | 说明 |
|------|------|------|
| Manager | 100 | 可做所有事，包括添加/移除用户、修改 Vault 设置 |
| Editor | 200 | 可编辑数据，但不能管理 Vault |
| Viewer | 300 | 可查看数据，但不能编辑 |

### 4.3 服务层权限校验：BaseService

文件：[BaseService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Services/BaseService.php)

所有业务服务继承 `BaseService`，通过 `permissions()` 方法声明所需权限，`validateRules()` 在执行时自动校验。

**权限依赖关系**（[BaseService.php#L41-L67](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Services/BaseService.php#L41-L67)）：

```php
private static array $permissionDependencies = [
    'author_must_belong_to_account' => [],
    'author_must_be_account_administrator' => ['author_must_belong_to_account'],
    'vault_must_belong_to_account' => [],
    'author_must_be_vault_manager' => ['vault_must_belong_to_account', 'author_must_belong_to_account'],
    'author_must_be_vault_editor' => ['vault_must_belong_to_account', 'author_must_belong_to_account'],
    'author_must_be_in_vault' => ['vault_must_belong_to_account', 'author_must_belong_to_account'],
    'contact_must_belong_to_vault' => ['vault_must_belong_to_account', 'author_must_belong_to_account'],
    'group_must_belong_to_vault' => ['vault_must_belong_to_account', 'author_must_belong_to_account'],
];
```

**权限校验的核心方法**（[BaseService.php#L190-L200](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Services/BaseService.php#L190-L200)）：

```php
public function validateUserPermissionInVault(int $permission): void
{
    $exists = $this->author->vaults()
        ->where('vaults.id', $this->vault->id)
        ->wherePivot('permission', '<=', $permission)  // 关键：<= 比较实现权限继承
        ->exists();

    if (! $exists) {
        throw new NotEnoughPermissionException;
    }
}
```

各权限声明的实际校验：

| 权限声明 | 校验的 permission 值 | 效果 |
|----------|----------------------|------|
| `author_must_be_vault_manager` | `<= 100` | 仅 Manager 可通过 |
| `author_must_be_vault_editor` | `<= 200` | Manager 和 Editor 可通过 |
| `author_must_be_in_vault` | `<= 300` | 所有成员可通过 |

### 4.4 Gate 定义：AuthServiceProvider

文件：[AuthServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Providers/AuthServiceProvider.php#L36-L54)

```php
// 只要属于 Vault 即可通过（任何权限）
Gate::define('vault-viewer', function (User $user, $vault): bool {
    return $user->vaults()
        ->wherePivotIn('vault_id', [static::id($vault)])
        ->exists();
});

// permission <= 200 (Manager 或 Editor)
Gate::define('vault-editor', function (User $user, $vault): bool {
    return $user->vaults()
        ->wherePivotIn('vault_id', [static::id($vault)])
        ->wherePivot('permission', '<=', 200)
        ->exists();
});

// permission <= 100 (仅 Manager)
Gate::define('vault-manager', function (User $user, $vault): bool {
    return $user->vaults()
        ->wherePivotIn('vault_id', [static::id($vault)])
        ->wherePivot('permission', '<=', 100)
        ->exists();
});
```

### 4.5 VaultPolicy

文件：[VaultPolicy.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Policies/VaultPolicy.php)

```php
public function view(User $user, Vault $vault): bool
{
    return Gate::forUser($user)->allows('vault-viewer', $vault);
}

public function update(User $user, Vault $vault): bool
{
    return Gate::forUser($user)->allows('vault-editor', $vault);
}

public function delete(User $user, Vault $vault): bool
{
    return Gate::forUser($user)->allows('vault-manager', $vault);
}
```

### 4.6 路由层权限控制

在 [web.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/routes/web.php) 中，Vault 相关路由通过中间件进行权限控制：

| 路由组 | 中间件 | 含义 |
|--------|--------|------|
| `/vaults/{vault}` 基本操作 | 无额外中间件 | 依赖 VaultPolicy 自动授权 |
| `/vaults/{vault}/...` 大部分子路由 | `can:vault-viewer,vault` | Viewer 及以上可访问 |
| `/vaults/{vault}/contacts/{contact}/...` | `can:contact-owner,vault,contact` | 验证 Contact 属于 Vault |
| `/vaults/{vault}/settings/...` | `can:vault-manager,vault` | 仅 Manager 可访问设置 |

### 4.7 权限读取辅助

[VaultHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Helpers/VaultHelper.php) 提供两个辅助方法：

```php
// 获取用户在 Vault 中的权限值（带 5 秒数组缓存）
public static function getPermission(User $user, Vault $vault): ?int

// 获取权限的友好名称
public static function getPermissionFriendlyName(int $permission): string
    // 100 → "Manager", 200 → "Editor", 300 → "Viewer"
```

### 4.8 各操作的权限要求汇总

| 操作 | 所需最低权限 |
|------|-------------|
| 查看 Vault 内容（联系人、日历等） | Viewer (300) |
| 编辑联系人数据 | Editor (200) |
| 移动/复制联系人到其他 Vault | Editor (200) |
| 管理 Vault 设置（标签、分组可见性等） | Manager (100) |
| 添加/移除/修改 Vault 成员 | Manager (100) |
| 删除 Vault | Manager (100) |
| 修改成员权限 | Manager + 账户管理员 |

---

## 五、修改成员权限：ChangeVaultAccess

文件：[ChangeVaultAccess.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVaultSettings/Services/ChangeVaultAccess.php)

**权限要求**：调用者必须同时是 `author_must_be_account_administrator` + `author_must_be_vault_manager`，即必须是账户管理员且是该 Vault 的 Manager。

**核心逻辑**：直接更新 `user_vault` 中间表的 `permission` 字段

```php
private function change(): void
{
    $this->user->vaults()
        ->where('vault_id', $this->vault->id)
        ->first()
        ->pivot
        ->update([
            'permission' => $this->data['permission'],
        ]);
}
```

> 注意：修改权限时不影响关联的 Contact 记录，也不重新调度提醒。

---

## 六、移除成员与历史数据处理

### 6.1 服务层：RemoveVaultAccess

文件：[RemoveVaultAccess.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php)

**权限要求**：`author_must_be_vault_manager`（Vault Manager）

**核心逻辑**：

```php
private function remove(): void
{
    $vault = $this->user->vaults()
        ->wherePivot('vault_id', $this->vault->id)
        ->first();

    if ($vault !== null) {
        Contact::find($vault->pivot->contact_id)->delete();  // 删除关联的 Contact
    }
}
```

**关键设计**：设计意图是删除 `user_vault` 关联的 Contact 记录，利用 `user_vault` 表 `contact_id` 外键的 `cascadeOnDelete` 约束级联删除 `user_vault` 中间表记录，从而移除用户的 Vault 访问权限。**但由于 Contact 使用了 SoftDeletes，实际执行的是 UPDATE 而非真正的 DELETE，级联删除不会触发**——详见 6.4 节详细分析。

### 6.2 清理提醒数据

```php
private function removeAllRemindersForThisUserInThisVault(): void
{
    $this->user->notificationChannels->each(function (UserNotificationChannel $notificationChannel) {
        $notificationChannel->contactReminders->each(function ($reminder) {
            $reminder->pivot->delete();  // 删除 contact_reminder_scheduled 中间表记录
        });
    });
}
```

移除用户时，会遍历该用户的所有通知渠道，删除所有已调度的联系人提醒关联。

### 6.3 历史数据处理说明

移除成员时的数据处理情况：

| 数据 | 处理方式 | 说明 |
|------|----------|------|
| `user_vault` 中间表记录 | **仍然存在**，不会被删除 | 软删除是 UPDATE 语句，不触发数据库外键级联，记录永久残留 |
| 关联的 Contact 记录 | 软删除（`deleted_at` 设为当前时间） | 该用户在 Vault 中的"身份"被软删除，但数据仍在 |
| 联系人提醒调度 (`contact_reminder_scheduled`) | 主动删除 | 该用户不再收到此 Vault 的提醒（通过遍历删除） |
| 用户创建的联系人、笔记等数据 | **不删除**，仍保留在 Vault 中 | 数据归属 Vault，不随用户移除而消失 |
| `contact_vault_user` 中的收藏/浏览记录 | **仍然存在** | 用户本身未被删除，中间表记录不会因外键级联删除 |
| 用户账户本身 | **不删除** | 用户仍属于 Account，只是不再有该 Vault 的身份 Contact |

### 6.4 Contact 软删除与外键级联的实际行为

这是 RemoveVaultAccess 设计的**关键点**，需要结合 Laravel SoftDeletes 行为和数据库外键约束分别分析。

#### 6.4.1 技术背景

- **Contact 模型**使用了 `SoftDeletes` trait（[Contact.php#L19](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Models/Contact.php#L19) 和 [Contact.php#L29](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Models/Contact.php#L29)）
- **迁移定义**中 `user_vault` 表的 `contact_id` 外键使用了 `cascadeOnDelete()`（[2020_04_25_133132_create_contacts_table.php#L64](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L64)）

#### 6.4.2 核心结论：软删除不触发外键级联，且**不影响权限校验通过**，被移除用户仍能访问 Vault

`SoftDeletes` 本质上是 Eloquent 在执行 `DELETE` 查询时改写为 `UPDATE contacts SET deleted_at = NOW() WHERE id = ?`，**不是真正的数据库 DELETE**。而数据库外键的 `ON DELETE CASCADE` 只有在数据库层面执行真正的 `DELETE` 语句时才会触发。

所以：

1. **`Contact::find($id)->delete()` 执行的是软删除（UPDATE）** → **不会**触发 `user_vault.contact_id` 的外键级联 → **`user_vault` 记录仍然存在于数据库中**
2. **权限校验不会因此失效**——关键原因逐层分析如下：

##### 6.4.2.1 权限校验的查询逻辑完全不涉及 Contact 表

**Gate 层**（[AuthServiceProvider.php#L36-L54](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Providers/AuthServiceProvider.php#L36-L54)）：
```php
Gate::define('vault-viewer', function (User $user, $vault): bool {
    return $user->vaults()                         // 查 user_vault 表
        ->wherePivotIn('vault_id', [static::id($vault)])
        ->exists();                                  // 只要 user_vault 中有记录就返回 true
});

Gate::define('vault-editor', function (User $user, $vault): bool {
    return $user->vaults()
        ->wherePivotIn('vault_id', [static::id($vault)])
        ->wherePivot('permission', '<=', 200)       // 只查 permission 字段
        ->exists();
});
```

**服务层**（[BaseService.php#L190-L199](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Services/BaseService.php#L190-L199)）：
```php
public function validateUserPermissionInVault(int $permission): void
{
    $exists = $this->author->vaults()              // 查 user_vault 表
        ->where('vaults.id', $this->vault->id)
        ->wherePivot('permission', '<=', $permission)  // 只查 permission 字段
        ->exists();
    // 不检查 Contact 的任何状态
}
```

**User::vaults() 关系定义**（[User.php#L186-L191](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Models/User.php#L186-L191)）：
```php
public function vaults(): BelongsToMany
{
    return $this->belongsToMany(Vault::class)       // 只关联 user_vault 中间表 + vaults 表
        ->withPivot('permission', 'contact_id')     // 不 join contacts 表
        ->withTimestamps();
}
```

**关键结论**：所有权限校验查询都只查 `user_vault` 中间表的 `permission` 字段，**不 join contacts 表，也不检查 `deleted_at` 状态**。只要 `user_vault` 记录还在，被移除用户的权限校验就仍然通过。

##### 6.4.2.2 唯一会失效的地方：依赖 `getContactInVault()` 的业务逻辑

**[getContactInVault()](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Models/User.php#L267-L282)** 是少数会实际查询 Contact 的方法：
```php
public function getContactInVault(Vault $vault): ?Contact
{
    $entry = $this->vaults()
        ->wherePivot('vault_id', $vault->id)
        ->first();

    if ($entry === null) {
        return null;
    }

    try {
        return Contact::findOrFail($entry->pivot->contact_id);  // 会因 SoftDeletes 全局作用域抛出 ModelNotFoundException
    } catch (ModelNotFoundException) {
        return null;
    }
}
```

这个方法在 [VaultController::show()](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L68) 等页面入口被调用，用于获取当前用户在 Vault 中的身份 Contact。如果该 Contact 已软删除，这里会返回 null，导致部分页面（如 Vault 仪表板）因缺少必要数据而无法正常渲染。

但**不依赖 `getContactInVault()` 的 API 或页面（如联系人列表、笔记列表等）仍然可以正常访问**，因为它们的权限校验只走 Gate 层和服务层。

#### 6.4.3 测试用例的 Bug：断言权限值不一致，永远空过

单元测试 [RemoveVaultAccessTest.php#L107-L152](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/tests/Unit/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccessTest.php#L107-L152) 存在严重缺陷：

**实际写入**（[RemoveVaultAccessTest.php#L33-L36](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/tests/Unit/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccessTest.php#L33-L36)）：
```php
$vault->users()->save($anotherUser, [
    'permission' => Vault::PERMISSION_MANAGE,   // 写入 100
    'contact_id' => $contact->id,
]);
```

**断言检查**（[RemoveVaultAccessTest.php#L137-L141](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/tests/Unit/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccessTest.php#L137-L141)）：
```php
$this->assertDatabaseMissing('user_vault', [
    'vault_id' => $vault->id,
    'user_id'  => $anotherUser->id,
    'permission' => Vault::PERMISSION_VIEW,      // 检查 300
]);
```

**问题**：写入的是 `PERMISSION_MANAGE=100`，但断言检查的是 `PERMISSION_VIEW=300`。数据库里本来就不存在 `permission=300` 的记录，所以无论 `user_vault` 记录是否被删除，这条断言**永远通过**（空过），测试完全没有验证级联删除是否生效。

**正确的断言**应该是：
```php
$this->assertDatabaseMissing('user_vault', [
    'vault_id' => $vault->id,
    'user_id'  => $anotherUser->id,
    'permission' => Vault::PERMISSION_MANAGE,    // 应该和写入一致
]);
```

另外，辅助方法 `setPermissionInVault()`（[TestCase.php#L69-L80](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/tests/TestCase.php#L69-L80)）也有同样的模式——创建独立的 Contact 用于测试，但没有验证级联行为。

#### 6.4.4 最终修正结论

1. **`user_vault` 记录不会被级联删除**：软删除是 UPDATE 语句，不触发数据库外键的 ON DELETE CASCADE，中间表记录**始终存在**。
2. **权限校验不受影响**：Gate 和服务层校验只查 `user_vault` 表，不检查 Contact 的软删除状态，被移除用户**仍然通过权限校验，可以正常访问 Vault 内容**。
3. **只有特定业务页面失效**：依赖 `getContactInVault()` 的页面（如 Vault 仪表板）会因 Contact 软删除返回 null 而无法正常渲染，但不依赖该方法的页面（如联系人列表）仍可正常访问。
4. **测试用例存在缺陷**：断言的权限值与实际写入不一致，导致测试永远通过，没有真正验证级联删除行为。

> **安全隐患**：被移除的用户实际上并未失去访问权，只要他知道 Vault 的 URL，仍然可以查看和操作 Vault 中的数据（取决于权限级别）。这是 RemoveVaultAccess 设计中的一个严重漏洞。

---

## 七、完整流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                      账户层级邀请流程                              │
│                                                                 │
│  管理员 → InviteUser → 创建 User(invitation_code=UUID)          │
│                     → 发送邮件 (含 /invitation/{code} 链接)       │
│                                                                 │
│  被邀请者 → 点击链接 → AcceptInvitationController::show          │
│          → 填写信息 → AcceptInvitationController::store          │
│                     → AcceptInvitation::execute()                │
│                       → 更新用户信息 + 标记 invitation_accepted_at│
│                       → 创建通知渠道 → 自动登录                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Vault 层级赋权流程                           │
│                                                                 │
│  Manager → Vault Settings → 选择用户 + 选择权限                   │
│         → VaultSettingsUserController::store                    │
│         → GrantVaultAccessToUser::execute()                      │
│           → 创建 Contact (用户在 Vault 中的身份, can_be_deleted=false)│
│           → 写入 user_vault (vault_id, user_id, contact_id, permission)│
│           → 调度该 Vault 所有联系人提醒给新用户                    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      权限判断体系                                 │
│                                                                 │
│  路由层: can:vault-viewer/vault-editor/vault-manager 中间件      │
│     ↓                                                           │
│  Gate 定义 (AuthServiceProvider):                                │
│     vault-viewer  → 用户在 user_vault 中存在记录                  │
│     vault-editor  → permission <= 200                            │
│     vault-manager → permission <= 100                            │
│     ↓                                                           │
│  服务层 (BaseService::validateUserPermissionInVault):             │
│     author_must_be_vault_manager  → permission <= 100            │
│     author_must_be_vault_editor   → permission <= 200            │
│     author_must_be_in_vault       → permission <= 300            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      移除成员流程                                 │
│                                                                 │
│  Manager → Vault Settings → 点击 Remove                         │
│         → VaultSettingsUserController::destroy                  │
│         → RemoveVaultAccess::execute()                           │
│           → 删除关联的 Contact (软删除: UPDATE deleted_at)       │
│             → ⚠️  user_vault 记录仍然存在!           │
│             → ⚠️  权限校验仍然通过 (只查 user_vault, 不查 Contact)│
│             → ⚠️  被移除用户仍可访问 Vault!                     │
│           → 删除该用户所有通知渠道中的联系人提醒调度                 │
│                                                                 │
│  历史数据处理:                                                    │
│    ✗ user_vault 记录 → 仍然存在 (权限校验仍然通过!)              │
│    ✓ Contact (用户身份) → 软删除 (deleted_at 已设置)             │
│    ✓ 提醒调度 → 删除                                              │
│    ✗ 用户创建的数据 → 保留在 Vault 中                             │
│    ✗ 用户账户 → 保留在 Account 中                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **⚠️ 安全警告**：移除成员的级联删除设计存在漏洞。被移除用户仍然拥有访问权，可正常访问 Vault 内容。

---

## 八、API 层与 Web 层的赋权与权限校验对比

### 8.1 API 层赋权入口现状

当前 API 路由文件 [api.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/routes/api.php) 中注册的 Vault 相关接口只有：

```php
Route::middleware('auth:sanctum')->name('api.')->group(function () {
    Route::apiResource('vaults', VaultController::class);  // 仅 CRUD
});
```

对应控制器：[Api/VaultController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVault/Api/Controllers/VaultController.php)

**重要发现**：API 层**目前没有暴露** Vault 成员管理（添加/修改/移除成员）的接口。API 只能对 Vault 本身做 CRUD，无法进行成员赋权操作。同样，账户层级的用户邀请在 API 层也没有入口（API 的 [UserController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Settings/ManageUsers/Api/Controllers/UserController.php) 只提供了 read 能力）。

以下对比聚焦于**现有 Web 入口与 API 入口在权限校验和错误返回机制上的差异**：

### 8.2 权限校验方式的差异

| 维度 | Web 层 | API 层 |
|------|--------|--------|
| **认证中间件** | `auth:sanctum` + `verified` + Jetstream session 中间件（[web.php#L184-L188](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/routes/web.php#L184-L188)） | `auth:sanctum` 单一中间件（[api.php#L18](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/routes/api.php#L18)） |
| **路由级权限** | `can:vault-viewer,vault` / `can:vault-manager,vault` 等 Gate 中间件（[web.php#L199](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/routes/web.php#L199)） | 使用 `abilities:read/write` Sanctum Token 能力校验（[Api/VaultController.php#L23-L24](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVault/Api/Controllers/VaultController.php#L23-L24)） |
| **Vault 授权** | `$this->authorizeResource(Vault::class, 'vault')` 走 VaultPolicy（[Web/VaultController.php#L30](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L30)） | **无 Gate/Policy 校验**，仅通过 `$request->user()->account->vaults()` 隐式限定在所属 Account（[Api/VaultController.php#L38-L39](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVault/Api/Controllers/VaultController.php#L38-L39)） |
| **服务层权限** | 调用 BaseService 的 `validateRules()`，通过 `permissions()` 声明做细粒度校验（服务层内部） | **相同**：底层服务一致，最终也走 BaseService 校验 |
| **账户设置入口** | `can:administrator` Gate 中间件保护 `/settings/users/*`（[web.php#L580](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/routes/web.php#L580)） | **无对应入口** |
| **Vault 设置入口** | `can:vault-manager,vault` Gate 中间件保护 `/vaults/{vault}/settings/*`（[web.php#L485](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/routes/web.php#L485)） | **无对应入口** |

### 8.3 错误返回格式的差异

#### Web 层错误返回

Web 控制器（如 [Web/VaultController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php)）**没有**统一异常捕获包装，直接让异常冒泡给 Laravel 全局异常处理器 [Handler.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Exceptions/Handler.php)：

```php
// Handler 没有注册任何自定义 render 回调
public function register()
{
    $this->reportable(function (Throwable $e) {
        if (app()->bound('sentry')) {
            app('sentry')->captureException($e);
        }
    });
}
```

所以 Web 层异常交给 Laravel 默认处理：
- **ValidationException**：返回 422，格式为 `{ message, errors: { field: [msg] } }`
- **ModelNotFoundException**：返回 404（由 Inertia/Laravel 渲染错误页）
- **NotEnoughPermissionException**：返回 403
- 前端 [Users.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/resources/js/Pages/Vault/Settings/Partials/Users.vue#L296-L299) 直接把 `error.response.data` 赋给表单 errors

#### API 层错误返回

API 控制器继承 [ApiController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Http/Controllers/ApiController.php)，使用了 `callAction` 方法包装统一的 try/catch：

```php
public function callAction($method, $parameters)
{
    try {
        return $this->{$method}(...array_values($parameters));
    } catch (ModelNotFoundException) {
        return $this->respondNotFound();        // { error: { message, error_code: 31 } }, HTTP 404
    } catch (QueryException) {
        return $this->respondInvalidQuery();    // { error: { message, error_code: 40 } }, HTTP 500
    } catch (ValidationException $e) {
        return $this->respondValidatorFailed($e->validator);  // { error: { message, error_code: 32 } }, HTTP 422
    }
}
```

错误码定义在 [api.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/config/api.php#L42-L50) 配置文件中：

| HTTP 状态码 | error_code | 含义 |
|-------------|------------|------|
| 400 | 30 | limit 参数过大 |
| 404 | 31 | 资源未找到 |
| 422 | 32 | 校验失败 |
| 500 | 33 | 参数过多 |
| 500 | 40 | 查询错误 |
| 422 | 41 | 参数非法 |
| 401 | 42 | 未授权 |

**统一格式**：
```json
{
  "error": {
    "message": "The resource has not been found",
    "error_code": 31
  }
}
```

#### 错误返回差异总结

| 异常类型 | Web 层（Inertia/Axios） | API 层 |
|---------|------------------------|--------|
| 验证失败 ValidationException | HTTP 422，默认 Laravel errors 对象 | HTTP 422，`{ error: { message: [...], error_code: 32 } }` |
| 模型不存在 ModelNotFoundException | HTTP 404，Laravel/Inertia 错误页 | HTTP 404，`{ error: { message, error_code: 31 } }` |
| 权限不足 NotEnoughPermissionException | HTTP 403，交给 Laravel 默认渲染 | **未捕获**！直接冒泡 → HTTP 500（这是 API 层的潜在不足） |
| SameUserException | HTTP 500 | **未捕获** → HTTP 500 |
| 查询错误 QueryException | HTTP 500 | HTTP 500，`{ error: { message, error_code: 40 } }` |

### 8.4 Sanctum Token 能力校验

API 层还多了一层基于 Sanctum Token 能力的校验：[Api/VaultController.php#L22-L24](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVault/Api/Controllers/VaultController.php#L22-L24)

```php
$this->middleware('abilities:read')->only(['index', 'show']);
$this->middleware('abilities:write')->only(['store', 'update', 'delete']);
```

这意味着即使 API 用户通过了认证和服务层权限校验，其 Token 还必须具备对应的 `read`/`write` 能力（abilities），否则仍然 401。而 Web 层通过 Session 认证，不涉及 abilities 校验。

---

## 九、invitation_code 的有效期、重发与防爆破分析

### 9.1 invitation_code 的有效期

**结论：无有效期限制，永久有效直到被消费。**

代码依据分析：

1. **User 模型字段定义**（[User.php#L84-L103](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Models/User.php#L84-L103)）：
   - `invitation_code` 是普通 UUID 字符串，没有 `expired_at` 等过期时间字段
   - `invitation_accepted_at` 是 Datetime，用于标记是否已接受（接受后设为当前时间，未接受为 null）

2. **生成逻辑**（[InviteUser.php#L56-L64](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Settings/ManageUsers/Services/InviteUser.php#L56-L64)）：
   ```php
   User::create([
       'invitation_code' => (string) Str::uuid(),  // 仅生成 UUID，无任何过期时间
   ]);
   ```

3. **消费时的校验**（[AcceptInvitationController.php#L19-L25](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Http/Controllers/Auth/AcceptInvitationController.php#L19-L25) 和 [AcceptInvitation.php#L47-L52](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Settings/ManageUsers/Services/AcceptInvitation.php#L47-L52)）：
   ```php
   // 控制器 show 方法
   User::where('invitation_code', $code)
       ->whereNull('invitation_accepted_at')  // 仅校验 invitation_accepted_at 为 null（未被消费）
       ->firstOrFail();
   ```
   
   查询条件只检查 `invitation_code` 匹配 + `invitation_accepted_at is null`，**没有任何时间范围条件**。

4. **路由定义**（[web.php#L174-L175](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/routes/web.php#L174-L175)）：
   ```php
   Route::get('invitation/{code}', ...);
   Route::post('invitation', ...);
   ```
   两个路由都**没有** `throttle` 中间件。

### 9.2 重发入口

**结论：有两种变相重发方式，无需删除用户即可实现。**

#### 方式一：管理员从接口响应获取 invitation_code 直接拼接链接（无需删除用户）

这是最便捷的重发方式，代码依据如下：

1. **ViewHelper 返回 invitation_code 字段**（[UserIndexViewHelper.php#L30-L52](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Settings/ManageUsers/Web/ViewHelpers/UserIndexViewHelper.php#L30-L52)）：
   ```php
   public static function dtoUser(User $user, User $loggedUser): array
   {
       return [
           'id' => $user->id,
           'email' => $user->email,
           'invitation_code'         => $user->invitation_code ? $user->invitation_code : null,  // ← 直接返回原始 UUID
           'invitation_accepted_at'  => $user->invitation_accepted_at ? DateHelper::formatDate(...) : null,
           // ...
       ];
   }
   ```
   只要用户还没接受邀请（`invitation_accepted_at is null`），接口就会将 `invitation_code` 明文返回给前端。

2. **前端虽然没有直接显示，但数据已传递**（[Index.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/resources/js/Pages/Settings/Users/Index.vue#L51-L84)）：
   - 前端 UI 只显示 "Invitation sent" 状态（信封图标 + 文字），没有直接显示 `invitation_code`
   - 但通过 Inertia 的响应数据，管理员可以在浏览器开发者工具的 Network 面板中看到完整的 `invitation_code` 字段值

3. **拼接链接直接可用**：
   路由定义（[web.php#L174](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/routes/web.php#L174)）：
   ```php
   Route::get('invitation/{code}', ...)->name('invitation.show');
   ```
   管理员只需将获取到的 `invitation_code` 拼接到 URL：
   ```
   https://your-monica-domain/invitation/{invitation_code}
   ```
   直接发送给被邀请人即可，无需通过系统重新发送邮件。

#### 方式二：删除后重新邀请（会生成新的 invitation_code）

如果希望生成新的邀请码：
- 先删除该 User（[DestroyUser.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Settings/ManageUsers/Services/DestroyUser.php) 会真正删除用户和所有关联）
- 然后再次调用 InviteUser 重新邀请（会生成新的 invitation_code，发送新的邮件）

> **注意**：全库搜索 `ResendInvitation` / `resend.*invitation` 等关键词**没有任何匹配**，说明项目**确实没有提供官方的重发按钮或 API**。上述方式一属于"利用现有数据结构的变通做法"。

### 9.3 防爆破（速率限制）

**结论：几乎没有防爆破保护，存在安全隐患。**

代码依据：

1. **GET `/invitation/{code}` 无 throttle**：可无限次探测 invitation_code 是否有效

2. **POST `/invitation` 无 throttle**：可无限次暴力尝试猜测 invitation_code + 提交注册

3. **对比 OAuth 路由**：只有第三方登录路由加了节流（[web.php#L168](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/routes/web.php#L168)）：
   ```php
   Route::middleware(['throttle:oauth2-socialite'])->group(function () {
       Route::get('auth/{driver}', ...);
   });
   ```
   邀请相关路由没有类似保护。

4. **invitation_code 的熵评估**：`Str::uuid()` 生成的是 RFC 4122 UUID v4，拥有 122 位随机熵，理论上暴力破解的概率极低（猜测成功率约 10^-37）。所以虽然没有 throttle，但凭借 UUID 本身的高熵，爆破难度极大。

5. **账号锁定机制**：项目中也没有发现针对 invitation 错误次数的锁定/封禁逻辑。

### 9.4 invitation_code 的生命周期总览

```
InviteUser::execute()  创建用户并设置:
  ┌─ invitation_code = UUID v4
  └─ invitation_accepted_at = null
        │
        ▼
  发送邮件 (UserInvited Mailable)
  链接: /invitation/{code}
        │
        ├─ 用户点击 → AcceptInvitationController::show()
        │             WHERE invitation_code = ? AND invitation_accepted_at IS NULL
        │                 │
        │                 ├─ 匹配成功 → 渲染注册表单
        │                 └─ 匹配失败 → 重定向到首页（不暴露是 code 无效还是已被使用）
        │
        ▼
  用户提交注册 → AcceptInvitationController::store()
        │
        ▼
  AcceptInvitation::execute():
    ├─ 查找 invitation_code（再次检查 invitation_accepted_at IS NULL）
    ├─ 设置 invitation_accepted_at = Carbon::now()   ← 消费标记，此后该 code 永久失效
    ├─ 设置 email_verified_at = Carbon::now()
    ├─ 设置 password = Hash(密码)
    └─ 强制 is_account_administrator = false（接受邀请后自动降级为普通用户）
        │
        ▼
  Auth::login() → 自动登录用户
```

> **安全细节**：AcceptInvitationController::show() 中当 code 无效或已被使用时都统一重定向到首页，避免了枚举区分"code 存在但已用"和"code 不存在"的时序攻击。

---

## 十、关键文件索引

| 文件 | 职责 |
|------|------|
| [Vault.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Models/Vault.php) | Vault 模型，定义权限常量和用户多对多关系 |
| [User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Models/User.php) | User 模型，定义 vaults 多对多关系和 getContactInVault 方法 |
| [BaseService.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Services/BaseService.php) | 服务基类，权限校验核心逻辑 |
| [VaultHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Helpers/VaultHelper.php) | 权限查询辅助（带缓存） |
| [AuthServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Providers/AuthServiceProvider.php) | Gate 权限定义 |
| [VaultPolicy.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Policies/VaultPolicy.php) | Vault Policy |
| [InviteUser.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Settings/ManageUsers/Services/InviteUser.php) | 账户层级邀请用户 |
| [AcceptInvitation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Settings/ManageUsers/Services/AcceptInvitation.php) | 接受邀请 |
| [AcceptInvitationController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Http/Controllers/Auth/AcceptInvitationController.php) | 邀请链接控制器 |
| [UserInvited.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Mail/UserInvited.php) | 邀请邮件 |
| [CreateVault.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVault/Services/CreateVault.php) | 创建 Vault（含自动添加创建者） |
| [GrantVaultAccessToUser.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVaultSettings/Services/GrantVaultAccessToUser.php) | 添加用户到 Vault |
| [ChangeVaultAccess.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVaultSettings/Services/ChangeVaultAccess.php) | 修改用户 Vault 权限 |
| [RemoveVaultAccess.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php) | 移除用户 Vault 访问权 |
| [VaultSettingsUserController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVaultSettings/Web/Controllers/VaultSettingsUserController.php) | Vault 用户管理控制器 |
| [VaultSettingsIndexViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Domains/Vault/ManageVaultSettings/Web/ViewHelpers/VaultSettingsIndexViewHelper.php) | Vault 设置视图辅助 |
| [Users.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/resources/js/Pages/Vault/Settings/Partials/Users.vue) | Vault 用户管理前端组件 |
| [web.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/routes/web.php) | 路由定义（含权限中间件） |
| [2020_04_25_133132_create_contacts_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/database/migrations/2020_04_25_133132_create_contacts_table.php) | 数据库迁移（含 user_vault 和 contact_vault_user 表） |
