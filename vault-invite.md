# 联系人保险柜邀请流程代码分析

## 概述

Monica 系统中，保险柜（Vault）邀请用户加入并指定权限的过程涉及三大核心模块协同工作：

1. **邀请实体（Invitation Entity）** - 基于 User 模型的邀请码机制
2. **邮件通知（Email Notification）** - 邀请邮件的触发与发送
3. **成员权限映射（Member Permission Mapping）** - 通过 `user_vault` 中间表实现的多对多权限关系

三者共同保证：只有被邀请的用户才能按指定的权限身份操作保险柜中的数据。

---

## 一、邀请实体（Invitation Entity）

### 1.1 数据模型

邀请实体并非独立的数据库表，而是内建于 [User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Models/User.php) 模型中的两个关键字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `invitation_code` | string (UUID) | 邀请码，用于唯一标识一次邀请 |
| `invitation_accepted_at` | datetime (nullable) | 邀请接受时间，null 表示未接受 |

数据库迁移定义见 [2014_10_12_000000_create_users_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/database/migrations/2014_10_12_000000_create_users_table.php#L33-L34)。

### 1.2 邀请用户服务

**类名**：[InviteUser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Settings/ManageUsers/Services/InviteUser.php)

**核心流程**：
```php
public function execute(array $data): User
{
    $this->data = $data;
    $this->validateRules($data);  // 验证参数和权限
    $this->createUser();           // 创建用户记录并生成邀请码
    $this->sendEmail();            // 发送邀请邮件
    return $this->user;
}
```

**关键实现**：
- 使用 `Str::uuid()` 生成唯一邀请码
- 新创建的用户 `password` 为 null，只有邮箱和邀请码
- `is_account_administrator` 由邀请者指定
- 调用者必须是账户管理员（`author_must_be_account_administrator`）

### 1.3 接受邀请服务

**类名**：[AcceptInvitation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Settings/ManageUsers/Services/AcceptInvitation.php)

**核心流程**：
```php
public function execute(array $data): User
{
    $this->validateRules($data);           // 验证参数
    $this->findUserByInvitationCode();     // 通过邀请码查找未接受的用户
    $this->updateUser();                   // 更新用户信息、设置密码、标记已接受
    $this->createNotificationChannel();    // 创建邮件通知渠道
    return $this->user;
}
```

**安全校验**：
- 邀请码必须是有效的 UUID 格式
- 用户必须存在且 `invitation_accepted_at` 为 null
- 接受后自动将 `is_account_administrator` 设为 false（防止越权）

### 1.4 邀请控制器

**类名**：[AcceptInvitationController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Http/Controllers/Auth/AcceptInvitationController.php)

**路由**（定义于 [web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/routes/web.php#L174-L175)）：
- `GET /invitation/{code}` - 展示接受邀请页面
- `POST /invitation` - 提交接受邀请表单

---

## 二、邮件通知（Email Notification）

### 2.1 邮件类

**类名**：[UserInvited.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Mail/UserInvited.php)

**核心代码**：
```php
public function build()
{
    $invitationRoute = route('invitation.show', [
        'code' => $this->invitedUser->invitation_code,
    ]);

    return $this->markdown('emails.user.invitation')
        ->subject(trans('You are invited to join Monica'))
        ->with('userName', $this->user->name)
        ->with('url', $invitationRoute);
}
```

**设计特点**：
- 实现 `ShouldQueue` 接口，支持队列异步发送
- 接收两个 User 对象：被邀请用户（`$invitedUser`）和邀请发起者（`$user`）
- 邮件链接包含邀请码，点击后直接进入接受邀请页面

### 2.2 邮件模板

**模板路径**：[invitation.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/resources/views/emails/user/invitation.blade.php)

**邮件内容**：
- 标题："Please join Monica"
- 正文：邀请者姓名 + 邀请说明
- 按钮："Accept invitation and create your account"
- 链接指向 `/invitation/{code}`

### 2.3 触发时机

邮件在 [InviteUser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Settings/ManageUsers/Services/InviteUser.php#L66-L70) 的 `sendEmail()` 方法中触发：

```php
private function sendEmail(): void
{
    Mail::to($this->user->email)
        ->queue(new UserInvited($this->user, $this->author));
}
```

---

## 三、成员权限映射（Member Permission Mapping）

### 3.1 权限等级

定义于 [Vault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Models/Vault.php#L19-L23) 模型的常量：

| 权限 | 数值 | 说明 |
|------|------|------|
| `PERMISSION_MANAGE` | 100 | 管理员 - 可管理成员权限 |
| `PERMISSION_EDIT` | 200 | 编辑者 - 可修改数据 |
| `PERMISSION_VIEW` | 300 | 查看者 - 只能查看 |

> **数值越小权限越高**。权限校验使用 `<=` 比较，例如 100（管理者）自动拥有 200（编辑）和 300（查看）的权限。

### 3.2 中间表结构

**表名**：`user_vault`（多对多关系中间表）

定义于 [2020_04_25_133132_create_contacts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L61-L67)：

```php
Schema::create('user_vault', function (Blueprint $table) {
    $table->foreignIdFor(Vault::class)->constrained()->cascadeOnDelete();
    $table->foreignIdFor(User::class)->constrained()->cascadeOnDelete();
    $table->foreignIdFor(Contact::class)->constrained()->cascadeOnDelete();
    $table->integer('permission');  // 权限等级
    $table->timestamps();
});
```

**关键字段说明**：
- `vault_id` + `user_id` 组成复合关联
- `permission` 存储权限等级数值
- `contact_id` - 每个用户在保险柜中对应一个联系人记录（不可删除）

### 3.3 模型关系

**Vault 模型** - [Vault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Models/Vault.php#L129-L134)：
```php
public function users(): BelongsToMany
{
    return $this->belongsToMany(User::class)
        ->withPivot('permission', 'contact_id')
        ->withTimestamps();
}
```

**User 模型** - [User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Models/User.php#L186-L191)：
```php
public function vaults(): BelongsToMany
{
    return $this->belongsToMany(Vault::class)
        ->withPivot('permission', 'contact_id')
        ->withTimestamps();
}
```

### 3.4 授予保险柜访问权限

**类名**：[GrantVaultAccessToUser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Vault/ManageVaultSettings/Services/GrantVaultAccessToUser.php)

**核心流程**：
```php
public function execute(array $data): User
{
    $this->validate();          // 验证参数和权限
    $this->grant();             // 授予访问权限
    $this->scheduleContactReminders();  // 为新用户安排所有联系人提醒
    return $this->user;
}
```

**关键实现 - grant() 方法**：
```php
private function grant(): void
{
    // 1. 在保险柜中创建一个对应的联系人记录
    $contact = Contact::create([
        'vault_id' => $this->vault->id,
        'first_name' => $this->user->first_name,
        'last_name' => $this->user->last_name,
        'can_be_deleted' => false,  // 不可删除
        'template_id' => $this->vault->default_template_id,
    ]);

    // 2. 建立用户-保险柜关联，设置权限
    $this->vault->users()->save($this->user, [
        'permission' => $this->data['permission'],
        'contact_id' => $contact->id,
    ]);
}
```

**权限要求**：调用者必须是保险柜管理员（`author_must_be_vault_manager`）

### 3.5 修改保险柜权限

**类名**：[ChangeVaultAccess.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Vault/ManageVaultSettings/Services/ChangeVaultAccess.php)

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

**权限要求**：
- 调用者必须是保险柜管理员
- 同时必须是账户管理员（额外的安全限制）

### 3.6 移除保险柜访问权限

**类名**：[RemoveVaultAccess.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php)

```php
private function remove(): void
{
    $vault = $this->user->vaults()
        ->wherePivot('vault_id', $this->vault->id)
        ->first();

    if ($vault !== null) {
        // 删除关联的联系人记录，级联删除 user_vault 记录
        Contact::find($vault->pivot->contact_id)->delete();
    }
}
```

**设计巧妙之处**：通过删除联系人记录，利用数据库外键 `cascadeOnDelete` 自动删除 `user_vault` 中间表记录，实现权限撤销。

### 3.7 权限 Gate 定义

定义于 [AuthServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Providers/AuthServiceProvider.php#L36-L54)：

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

### 3.8 BaseService 权限验证

在 [BaseService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Services/BaseService.php#L190-L200) 中实现了统一的权限校验：

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

### 3.9 用户管理控制器

**类名**：[VaultSettingsUserController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Vault/ManageVaultSettings/Web/Controllers/VaultSettingsUserController.php)

提供三个 API 端点：
- `store()` - 添加用户到保险柜（POST）
- `update()` - 修改用户权限（PUT/PATCH）
- `destroy()` - 移除用户（DELETE）

---

## 四、三者协同工作流程

### 4.1 完整邀请时序

```
邀请者 (Vault Manager)
    │
    ▼
1. 邀请用户加入账户 (InviteUser 服务)
    ├── 创建 User 记录（含 invitation_code）
    ├── 触发 UserInvited 邮件（队列发送）
    └── 邮件包含 /invitation/{code} 链接
    │
    ▼
2. 被邀请者接收邮件
    ├── 点击邮件链接
    └── 进入 AcceptInvitationController@show
    │
    ▼
3. 被邀请者填写信息并提交
    ├── AcceptInvitationController@store
    ├── AcceptInvitation 服务执行
    │   ├── 验证邀请码有效性
    │   ├── 设置密码和个人信息
    │   ├── 标记 invitation_accepted_at
    │   └── 创建邮件通知渠道
    └── 自动登录，跳转到首页
    │
    ▼
4. 保险柜管理员授予保险柜权限
    ├── VaultSettingsUserController@store
    ├── GrantVaultAccessToUser 服务执行
    │   ├── 验证调用者是 vault manager
    │   ├── 在保险柜中创建联系人记录
    │   ├── 在 user_vault 表插入记录（permission + contact_id）
    │   └── 为新用户调度所有联系人提醒
    └── 返回用户权限信息
    │
    ▼
5. 被邀请者按权限操作保险柜
    ├── Gate 校验权限等级
    ├── BaseService 统一验证
    └── 按 MANAGE/EDIT/VIEW 三级控制操作
```

### 4.2 安全性保障机制

1. **邀请码机制**：UUID 格式，不可预测，一次性使用
2. **权限层级**：数值越小权限越高，`<=` 比较确保高权限自动包含低权限
3. **服务层权限校验**：每个 Service 通过 `permissions()` 方法声明所需权限，BaseService 统一验证
4. **Gate 门面校验**：控制器和策略层可通过 Gate 快速校验
5. **外键级联删除**：删除用户/保险柜/联系人时自动清理关联
6. **拒绝自我操作**：Grant/Change/Remove 服务均检查 `$this->user->id === $this->author->id`，防止用户修改自己的权限

### 4.3 创建保险柜时的初始权限

在 [CreateVault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Vault/ManageVault/Services/CreateVault.php#L80-L94) 的 `createUserContact()` 方法中：

```php
private function createUserContact(): void
{
    $contact = Contact::create([...]);  // 创建创建者的联系人记录
    
    $this->vault->users()->save($this->author, [
        'permission' => Vault::PERMISSION_MANAGE,  // 默认为管理员
        'contact_id' => $contact->id,
    ]);
}
```

保险柜的创建者自动获得 MANAGE 权限。

---

## 五、辅助工具

### 5.1 VaultHelper

**类名**：[VaultHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Helpers/VaultHelper.php)

提供两个静态方法：
- `getPermission(User $user, Vault $vault): ?int` - 获取用户在保险柜中的权限等级（带 array 缓存）
- `getPermissionFriendlyName(int $permission): string` - 将权限数值转换为友好名称（Manager/Editor/Viewer）

### 5.2 VaultPolicy

**类名**：[VaultPolicy.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Policies/VaultPolicy.php)

Laravel 策略类，封装了对 Vault 模型的权限判断：
- `viewAny` - 任何人都可以查看列表
- `view` - 需要 `vault-viewer` 权限
- `create` - 任何人都可以创建
- `update` - 需要 `vault-editor` 权限
- `delete` - 需要 `vault-manager` 权限

---

## 六、关键文件索引

| 模块 | 文件路径 | 作用 |
|------|----------|------|
| 邀请实体 | [User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Models/User.php) | 用户模型，含邀请字段 |
| 邀请实体 | [InviteUser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Settings/ManageUsers/Services/InviteUser.php) | 邀请用户服务 |
| 邀请实体 | [AcceptInvitation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Settings/ManageUsers/Services/AcceptInvitation.php) | 接受邀请服务 |
| 邀请实体 | [AcceptInvitationController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Http/Controllers/Auth/AcceptInvitationController.php) | 邀请页面控制器 |
| 邮件通知 | [UserInvited.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Mail/UserInvited.php) | 邀请邮件类 |
| 邮件通知 | [invitation.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/resources/views/emails/user/invitation.blade.php) | 邮件模板 |
| 权限映射 | [Vault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Models/Vault.php) | 保险柜模型，权限常量 |
| 权限映射 | [GrantVaultAccessToUser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Vault/ManageVaultSettings/Services/GrantVaultAccessToUser.php) | 授予权限服务 |
| 权限映射 | [ChangeVaultAccess.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Vault/ManageVaultSettings/Services/ChangeVaultAccess.php) | 修改权限服务 |
| 权限映射 | [RemoveVaultAccess.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php) | 移除权限服务 |
| 权限映射 | [VaultSettingsUserController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Domains/Vault/ManageVaultSettings/Web/Controllers/VaultSettingsUserController.php) | 用户权限控制器 |
| 权限映射 | [AuthServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Providers/AuthServiceProvider.php) | Gate 权限定义 |
| 权限映射 | [BaseService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Services/BaseService.php) | 基础服务，统一权限校验 |
| 权限映射 | [2020_04_25_133132_create_contacts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/database/migrations/2020_04_25_133132_create_contacts_table.php) | user_vault 中间表迁移 |
| 辅助工具 | [VaultHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Helpers/VaultHelper.php) | 权限辅助方法 |
| 辅助工具 | [VaultPolicy.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/app/Policies/VaultPolicy.php) | 模型策略类 |
