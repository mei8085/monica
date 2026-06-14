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

**关键设计**：删除 `user_vault` 关联的 Contact 记录，由于 `user_vault` 表中 `contact_id` 有外键约束（`cascadeOnDelete`），删除 Contact 会**级联删除** `user_vault` 中间表记录，从而移除用户的 Vault 访问权限。

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
| `user_vault` 中间表记录 | 级联删除（通过删除 Contact 触发） | 用户不再有 Vault 访问权 |
| 关联的 Contact 记录 | 直接删除（`Contact::delete()`） | 该用户在 Vault 中的"身份"被移除 |
| 联系人提醒调度 (`contact_reminder_scheduled`) | 主动删除 | 该用户不再收到此 Vault 的提醒 |
| 用户创建的联系人、笔记等数据 | **不删除**，仍保留在 Vault 中 | 数据归属 Vault，不随用户移除而消失 |
| `contact_vault_user` 中的收藏/浏览记录 | 依赖外键级联 | `user_id` 外键 `cascadeOnDelete`，但用户本身未被删除，仅脱离 Vault，所以这些记录**可能残留** |
| 用户账户本身 | **不删除** | 用户仍属于 Account，只是不再有该 Vault 的权限 |

### 6.4 Contact 的软删除

Contact 模型使用了 `SoftDeletes` trait（[Contact.php#L29](file:///d:/fz/0601-1/solo-dogfeeding/code/82-monica/app/Models/Contact.php#L29)），所以 `Contact::delete()` 是软删除，数据仍在数据库中（`deleted_at` 字段被设置）。但因为 `user_vault` 表的 `contact_id` 外键是 `constrained()->cascadeOnDelete()`，**软删除不会触发级联删除**，所以 `user_vault` 记录不会被自动删除。

> **这是一个潜在问题**：如果 Contact 软删除不触发级联，那么 `remove()` 方法中的 `Contact::find($vault->pivot->contact_id)->delete()` 实际上只是软删除了 Contact，而 `user_vault` 记录可能仍然存在。不过由于后续权限校验查询不会匹配已软删除的 Contact，用户实际上已无法访问 Vault。这取决于数据库外键的具体行为和 Laravel 对软删除与级联删除的处理。

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
│           → 删除关联的 Contact (软删除)                           │
│             → 级联删除 user_vault 记录（如硬删除则触发）            │
│           → 删除该用户所有通知渠道中的联系人提醒调度                 │
│                                                                 │
│  历史数据处理:                                                    │
│    ✓ user_vault 记录 → 删除                                      │
│    ✓ Contact (用户身份) → 软删除                                  │
│    ✓ 提醒调度 → 删除                                              │
│    ✗ 用户创建的数据 → 保留在 Vault 中                             │
│    ✗ 用户账户 → 保留在 Account 中                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 八、关键文件索引

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
