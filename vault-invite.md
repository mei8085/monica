# 联系人保险柜邀请流程代码分析

## 概述

Monica 系统中，保险柜（Vault）邀请用户加入并指定权限的过程涉及三大核心模块：

1. **账号邀请（Account Invitation）** - 基于 User 模型的邀请码机制，创建待激活账号
2. **邮件通知（Email Notification）** - 邀请邮件的触发与发送
3. **保险柜授权（Vault Authorization）** - 通过 `user_vault` 中间表实现的多对多权限关系

**关键认知：这三个步骤不是同一事务，而是三个彼此独立的操作，由不同角色在不同时间发起。**

---

## 一、三步骤非同一事务

### 1.1 三个独立操作

| 步骤 | 操作 | 服务类 | 发起者 | 前置条件 |
|------|------|--------|--------|----------|
| ① 账号邀请 | 创建 User 记录 + 发邮件 | `InviteUser` | 账户管理员 | 无 |
| ② 接受邀请 | 设置密码 + 激活账号 | `AcceptInvitation` | 被邀请者本人 | ① 已完成 |
| ③ 保险柜授权 | 建立用户-保险柜权限关联 | `GrantVaultAccessToUser` | 保险柜管理员 | ① 已完成（② 非必须） |

三者之间不存在数据库事务包裹，也不存在级联触发。每个步骤是独立的 HTTP 请求，由不同的控制器方法处理，调用不同的服务类。

### 1.2 操作顺序的灵活性

实际使用中存在两种合法路径：

**路径 A：先激活再授权（典型流程）**
```
① 账号邀请 → ② 接受邀请 → ③ 保险柜授权
```

**路径 B：先授权再激活（预配置流程）**
```
① 账号邀请 → ③ 保险柜授权 → ② 接受邀请
```

代码层面，`GrantVaultAccessToUser` 不检查 `invitation_accepted_at`，因此步骤 ③ 可以在步骤 ② 之前执行。但前端视图存在间接限制（见第 1.3 节）。

### 1.3 前端对授权对象的筛选

在 `app/Domains/Vault/ManageVaultSettings/Web/ViewHelpers/VaultSettingsIndexViewHelper.php` 第 31 行：

```php
$usersInAccount = $vault->account->users()->whereNotNull('email_verified_at')->get();
```

前端"可授权用户"列表通过 `whereNotNull('email_verified_at')` 过滤，只展示已完成邮箱验证的用户。由于接受邀请时 `AcceptInvitation::updateUser()` 会同时设置 `email_verified_at`（见第二节），未接受邀请的用户不会出现在前端的候选列表中。

**但这是前端展示层的限制，不是服务层的强制约束。** 如果通过 API 直接调用 `GrantVaultAccessToUser`，传入未激活用户的 `user_id`，服务不会拒绝。

---

## 二、账号邀请与管理员标记变化

### 2.1 邀请时的初始状态

`InviteUser` 服务（`app/Domains/Settings/ManageUsers/Services/InviteUser.php`）的 `createUser()` 方法：

```php
$this->user = User::create([
    'account_id' => $this->data['account_id'],
    'email' => $this->data['email'],
    'invitation_code' => (string) Str::uuid(),
    'is_account_administrator' => $this->data['is_administrator'],
]);
```

| 字段 | 值 | 说明 |
|------|-----|------|
| `email` | 被邀请者邮箱 | 唯一约束 |
| `invitation_code` | UUID | 邀请标识 |
| `invitation_accepted_at` | null | 未接受 |
| `email_verified_at` | null | 未验证 |
| `password` | null | 无密码，无法登录 |
| `first_name` / `last_name` | null | 尚未填写 |
| `is_account_administrator` | 由邀请者指定 | **关键：可为 true** |

邀请者在调用 `InviteUser` 时通过 `is_administrator` 参数决定被邀请者是否为账户管理员。这意味着在邀请阶段，一个待激活账号可能已经被标记为管理员。

### 2.2 接受邀请时的标记重置

`AcceptInvitation` 服务（`app/Domains/Settings/ManageUsers/Services/AcceptInvitation.php`）的 `updateUser()` 方法：

```php
private function updateUser(): void
{
    $this->user->is_account_administrator = false;   // ← 强制重置为 false
    $this->user->first_name = $this->data['first_name'];
    $this->user->last_name = $this->data['last_name'];
    $this->user->invitation_accepted_at = Carbon::now();
    $this->user->email_verified_at = Carbon::now();
    $this->user->password = Hash::make($this->data['password']);
    $this->user->timezone = 'UTC';
    $this->user->save();
}
```

**`is_account_administrator` 被硬编码重置为 `false`，无论邀请时是否被指定为 `true`。**

这个设计意味着：
- 邀请阶段设置的 `is_administrator = true` 仅为临时标记
- 接受邀请后，该标记被无条件清除
- 如果确实需要该用户成为管理员，必须在接受邀请后由现有管理员重新设置

### 2.3 管理员标记的含义与影响范围

`is_account_administrator` 控制的是**账户级别**的权限，与保险柜级别权限是两套独立体系：

| 标记 | 影响范围 | 检查位置 |
|------|----------|----------|
| `is_account_administrator` | 账户设置、用户管理、修改保险柜权限（`ChangeVaultAccess` 额外要求） | `BaseService::validateAuthorIsAccountAdministrator()`、`Gate::define('administrator')` |
| `user_vault.permission` | 保险柜内的数据操作 | `BaseService::validateUserPermissionInVault()`、`Gate::define('vault-*')` |

两套体系独立运行，保险柜管理员（permission=100）不一定是账户管理员，反之亦然。

---

## 三、保险柜授权是否需要用户先接受邀请

### 3.1 服务层：不要求

`GrantVaultAccessToUser`（`app/Domains/Vault/ManageVaultSettings/Services/GrantVaultAccessToUser.php`）的 `validate()` 方法：

```php
private function validate(): void
{
    $this->validateRules($this->data);

    $user = $this->account()->users()
        ->findOrFail($this->data['user_id']);
    $this->user = $user;

    if ($this->user->id === $this->author->id) {
        throw new SameUserException;
    }
}
```

验证逻辑仅确认：
1. 参数合法性（`validateRules`）
2. 目标用户属于同一账户（`account()->users()->findOrFail()`）
3. 不是给自己授权（`SameUserException`）

**没有检查 `invitation_accepted_at` 或 `email_verified_at`。** 只要用户记录存在于 `users` 表且属于同一账户，就可以授权。

### 3.2 前端展示层：间接要求

如第 1.3 节所述，前端通过 `whereNotNull('email_verified_at')` 过滤候选用户列表，未接受邀请的用户不会出现在界面上。这是展示层面的软限制，不是业务规则的硬约束。

### 3.3 路由中间件：保护操作者而非目标用户

保险柜授权路由（`routes/web.php` 第 485-491 行）：

```php
Route::middleware('can:vault-manager,vault')->group(function () {
    Route::post('settings/users', [VaultSettingsUserController::class, 'store']);
    // ...
});
```

而整个 vault 路由组位于更外层的中间件保护下（`routes/web.php` 第 184-188 行）：

```php
Route::middleware([
    'auth:sanctum',
    config('jetstream.auth_session'),
    'verified',
])->group(function () {
```

中间件保证的是**操作者**（发起授权的 vault manager）必须已登录且邮箱已验证，对**被授权的目标用户**无任何要求。

---

## 四、权限挂到待激活账号时的影响

### 4.1 数据库层面：权限记录立即生效

当 `GrantVaultAccessToUser` 对一个未接受邀请的用户执行授权时：

1. `user_vault` 表中立即插入一条记录，包含 `permission` 值和 `contact_id`
2. 权限数据在数据库中是完整的，对后续查询完全可见
3. `AuthServiceProvider` 中定义的 Gate（`vault-viewer`、`vault-editor`、`vault-manager`）只查询 `user_vault` 表的 `permission` 列，不检查用户的激活状态

**结论：从数据模型角度看，权限已绑定，但无法被行使。**

### 4.2 实际操作层面：待激活用户无法行使权限

权限能否被实际行使，取决于用户能否通过身份认证进入系统。待激活用户面临以下障碍：

| 障碍 | 原因 | 代码位置 |
|------|------|----------|
| 无法登录 | `password` 为 null，Fortify 无法验证凭据 | `InviteUser::createUser()` 未设置密码 |
| 无法通过 API 认证 | Sanctum token 需要先登录才能获取 | `auth:sanctum` 中间件 |
| 无法通过邮箱验证 | `email_verified_at` 为 null | `verified` 中间件拦截 |
| 不在前端候选列表 | `whereNotNull('email_verified_at')` 过滤 | `VaultSettingsIndexViewHelper::data()` |

**因此，虽然权限数据已写入数据库，但用户在未接受邀请（设置密码、激活账号）之前，无法通过任何途径登录系统，自然无法行使保险柜权限。**

### 4.3 接受邀请后权限自动可用

用户接受邀请后（`AcceptInvitation` 执行完毕），`password`、`email_verified_at`、`invitation_accepted_at` 均已设置，可以正常登录。此时如果在步骤 ② 之前已执行了步骤 ③，用户登录后即可立即行使已绑定的保险柜权限——无需再次授权。

这是"先授权再激活"路径的实际价值：管理员可以预先配置好新用户的保险柜权限，用户接受邀请后开箱即用。

### 4.4 联系人记录的特殊情况

`GrantVaultAccessToUser::grant()` 会为被授权用户创建一个 `can_be_deleted = false` 的联系人记录：

```php
$contact = Contact::create([
    'vault_id' => $this->vault->id,
    'first_name' => $this->user->first_name,
    'last_name' => $this->user->last_name,
    'can_be_deleted' => false,
    'template_id' => $this->vault->default_template_id,
]);
```

如果目标用户尚未接受邀请，`first_name` 和 `last_name` 均为 null，创建的联系人记录的姓名字段为空。这不影响权限功能，但会导致保险柜中出现一个空名的不可删除联系人。

---

## 五、邮件通知（Email Notification）

### 5.1 邮件类

`UserInvited`（`app/Mail/UserInvited.php`）实现 `ShouldQueue` 接口，支持队列异步发送：

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

### 5.2 邮件内容

模板位于 `resources/views/emails/user/invitation.blade.php`：

- 标题："Please join Monica"
- 正文：邀请者姓名 + 邀请说明
- 按钮："Accept invitation and create your account"
- 链接指向 `/invitation/{code}`

### 5.3 触发时机

邮件在 `InviteUser::sendEmail()` 中触发，与 `createUser()` 在同一 `execute()` 调用中，但两者是顺序执行的独立操作，不在同一数据库事务中。如果邮件队列出错，用户记录已经创建，不会回滚。

### 5.4 邮件与授权的无关性

邮件仅服务于账号邀请步骤，与保险柜授权完全无关。保险柜授权不触发任何邮件通知——授权后需由操作者自行告知被授权者。

---

## 六、成员权限映射（Member Permission Mapping）

### 6.1 权限等级

定义于 `app/Models/Vault.php` 第 19-23 行的常量：

| 权限 | 数值 | 说明 |
|------|------|------|
| `PERMISSION_MANAGE` | 100 | 管理员 - 可管理成员权限、删除保险柜 |
| `PERMISSION_EDIT` | 200 | 编辑者 - 可修改数据、更新保险柜 |
| `PERMISSION_VIEW` | 300 | 查看者 - 只能查看 |

**数值越小权限越高。** 权限校验使用 `<=` 比较，例如 permission=100（管理者）自动满足 `<= 200` 和 `<= 300` 的条件，即高权限自动包含低权限。

### 6.2 中间表结构

`user_vault` 表定义于 `database/migrations/2020_04_25_133132_create_contacts_table.php` 第 61-67 行：

```php
Schema::create('user_vault', function (Blueprint $table) {
    $table->foreignIdFor(Vault::class)->constrained()->cascadeOnDelete();
    $table->foreignIdFor(User::class)->constrained()->cascadeOnDelete();
    $table->foreignIdFor(Contact::class)->constrained()->cascadeOnDelete();
    $table->integer('permission');
    $table->timestamps();
});
```

| 字段 | 说明 |
|------|------|
| `vault_id` + `user_id` | 复合关联，标识哪个用户属于哪个保险柜 |
| `permission` | 权限等级数值（100/200/300） |
| `contact_id` | 每个用户在保险柜中对应一个联系人记录（`can_be_deleted = false`） |

### 6.3 模型关系

**Vault 模型**（`app/Models/Vault.php` 第 129-134 行）：

```php
public function users(): BelongsToMany
{
    return $this->belongsToMany(User::class)
        ->withPivot('permission', 'contact_id')
        ->withTimestamps();
}
```

**User 模型**（`app/Models/User.php` 第 186-191 行）：

```php
public function vaults(): BelongsToMany
{
    return $this->belongsToMany(Vault::class)
        ->withPivot('permission', 'contact_id')
        ->withTimestamps();
}
```

### 6.4 授予保险柜访问权限

`GrantVaultAccessToUser`（`app/Domains/Vault/ManageVaultSettings/Services/GrantVaultAccessToUser.php`）：

```php
private function grant(): void
{
    $contact = Contact::create([
        'vault_id' => $this->vault->id,
        'first_name' => $this->user->first_name,
        'last_name' => $this->user->last_name,
        'can_be_deleted' => false,
        'template_id' => $this->vault->default_template_id,
    ]);

    $this->vault->users()->save($this->user, [
        'permission' => $this->data['permission'],
        'contact_id' => $contact->id,
    ]);
}
```

权限要求：调用者必须是保险柜管理员（`author_must_be_vault_manager`）

### 6.5 修改保险柜权限

`ChangeVaultAccess`（`app/Domains/Vault/ManageVaultSettings/Services/ChangeVaultAccess.php`）：

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

权限要求：调用者必须同时是保险柜管理员和账户管理员（双重校验，比授权操作更严格）。

### 6.6 移除保险柜访问权限

`RemoveVaultAccess`（`app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php`）：

```php
private function remove(): void
{
    $vault = $this->user->vaults()
        ->wherePivot('vault_id', $this->vault->id)
        ->first();

    if ($vault !== null) {
        Contact::find($vault->pivot->contact_id)->delete();
    }
}
```

通过删除联系人记录，利用数据库外键 `cascadeOnDelete` 自动删除 `user_vault` 中间表记录，实现权限撤销。

### 6.7 权限 Gate 定义

`app/Providers/AuthServiceProvider.php` 第 36-54 行：

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

### 6.8 BaseService 权限验证

`app/Services/BaseService.php` 第 190-200 行：

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

### 6.9 用户管理控制器

`VaultSettingsUserController`（`app/Domains/Vault/ManageVaultSettings/Web/Controllers/VaultSettingsUserController.php`）：

- `store()` - 添加用户到保险柜（POST），调用 `GrantVaultAccessToUser`
- `update()` - 修改用户权限（PUT），调用 `ChangeVaultAccess`
- `destroy()` - 移除用户（DELETE），调用 `RemoveVaultAccess`

### 6.10 权限值校验边界：只校验 integer 带来的风险

`GrantVaultAccessToUser` 和 `ChangeVaultAccess` 两个服务的 `rules()` 方法中，对 `permission` 字段的校验完全相同：

```php
// GrantVaultAccessToUser::rules() 第 29 行 / ChangeVaultAccess::rules() 第 26 行
'permission' => 'required|integer',
```

**仅校验类型为整数，不校验具体数值范围**。这带来以下边界情况：

| 问题 | 表现 | 影响 |
|------|------|------|
| **非三档值可写入** | 可以传入任意整数（如 1, 50, 99, 150, 250, 999 等） | 权限判断使用 `<=` 比较，数值越小权限越高。写入 50 将获得比 MANAGE(100) 更高的权限，写入 999 将获得比 VIEW(300) 更低的权限（无法通过任何 Gate 校验） |
| **前端限制可绕过** | 前端 [Users.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/resources/js/Pages/Vault/Settings/Partials/Users.vue) 通过单选按钮硬编码 value="100"/"200"/"300"，但直接调用 API 可绕过 | API 层面无防护，恶意调用可能越权 |
| **负数和零值** | 可以传入 0 或负数 | 这些值 < 100，将获得"超级管理员"权限 |

**代码层面没有任何 `in:100,200,300` 或 `min`/`max` 校验规则**，这是一个明显的安全边界漏洞。

### 6.11 重复授权边界：同一用户重复授权产生多条 user_vault 关系

#### 数据库层面：缺少唯一约束

`user_vault` 表定义（`database/migrations/2020_04_25_133132_create_contacts_table.php` 第 61-67 行）：

```php
Schema::create('user_vault', function (Blueprint $table) {
    $table->foreignIdFor(Vault::class)->constrained()->cascadeOnDelete();
    $table->foreignIdFor(User::class)->constrained()->cascadeOnDelete();
    $table->foreignIdFor(Contact::class)->constrained()->cascadeOnDelete();
    $table->integer('permission');
    $table->timestamps();
    // 没有 $table->unique(['vault_id', 'user_id'])
});
```

**没有对 `(vault_id, user_id)` 建立唯一索引**，数据库层面允许多条相同 `vault_id + user_id` 的记录。

#### 代码层面：无存在性检查

`GrantVaultAccessToUser::grant()` 第 72-86 行：

```php
private function grant(): void
{
    $contact = Contact::create([...]);  // 每次都创建新的联系人

    $this->vault->users()->save($this->user, [
        'permission' => $this->data['permission'],
        'contact_id' => $contact->id,
    ]);
}
```

- `validate()` 方法只检查目标用户属于同一账户且不是给自己授权
- **没有检查用户是否已经在该保险柜中**
- `$this->vault->users()->save(...)` 在多对多关系中总是插入新记录（不同于 `sync()`）

**结论：对同一用户重复调用 `GrantVaultAccessToUser`，会产生多条 `user_vault` 记录，同时产生多个 `can_be_deleted = false` 的联系人记录。**

#### 前端层面：间接防止重复

在 [Users.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/resources/js/Pages/Vault/Settings/Partials/Users.vue) 的 `store()` 方法第 288-292 行：

```js
// 成功后从可添加列表中移除
var id = this.localUsersInAccount.findIndex((x) => x.id === this.form.user_id);
this.localUsersInAccount.splice(id, 1);
```

用户添加成功后，前端立即将其从"可添加用户"列表中移除，防止界面上的重复操作。但这是前端逻辑，API 调用仍可绕过。

#### 测试层面：无重复授权测试

[GrantVaultAccessToUserTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/16-monica/tests/Unit/Domains/Vault/ManageVaultSettings/Services/GrantVaultAccessToUserTest.php) 中没有覆盖"同一用户重复授权"的场景，测试用例只验证了正常授权流程。

### 6.12 重复关系下的操作行为分析

当同一用户在 `user_vault` 表中有多条记录时，各操作的行为如下：

#### 6.12.1 权限判断：取最高权限

`AuthServiceProvider` 中的 Gate 使用 `exists()` 判断：

```php
Gate::define('vault-editor', function (User $user, $vault): bool {
    return $user->vaults()
        ->wherePivotIn('vault_id', [static::id($vault)])
        ->wherePivot('permission', '<=', 200)
        ->exists();
});
```

`BaseService::validateUserPermissionInVault()` 同样使用 `exists()`：

```php
public function validateUserPermissionInVault(int $permission): void
{
    $exists = $this->author->vaults()
        ->where('vaults.id', $this->vault->id)
        ->wherePivot('permission', '<=', $permission)
        ->exists();
}
```

**行为**：只要有任意一条记录满足 `permission <= N`，就返回 `true`。

**影响**：如果用户有两条记录，一条 permission=100（MANAGE），一条 permission=300（VIEW），则该用户拥有 MANAGE 权限。重复授权只会"升高"或"保持"权限，不会降低。

#### 6.12.2 修改权限：只改第一条记录

`ChangeVaultAccess::change()` 第 69-79 行：

```php
private function change(): void
{
    $this->user->vaults()
        ->where('vault_id', $this->vault->id)
        ->first()  // ← 只取第一条
        ->pivot
        ->update([
            'permission' => $this->data['permission'],
        ]);
}
```

**行为**：`first()` 只返回按主键升序排列的第一条匹配记录。

**影响**：
- 如果存在多条记录，只有最旧的那条（id 最小）的 permission 会被更新
- 其他记录保持原值不变
- 如果第一条记录权限被降低（如 100 → 300），但第二条记录仍为 100，则用户实际权限仍然是 MANAGE
- **前端显示可能与实际权限不符**

#### 6.12.3 移除权限：只删第一条记录

`RemoveVaultAccess::remove()` 第 73-82 行：

```php
private function remove(): void
{
    $vault = $this->user->vaults()
        ->wherePivot('vault_id', $this->vault->id)
        ->first()  // ← 只取第一条
        ->first();

    if ($vault !== null) {
        Contact::find($vault->pivot->contact_id)->delete();
    }
}
```

**行为**：`first()` 只返回第一条匹配记录，删除其关联的 Contact（级联删除 user_vault 记录）。

**影响**：
- 如果存在多条记录，只有第一条被删除
- 其他记录仍然存在，用户权限不变
- **出现"已移除但仍有权限"的不一致状态**

#### 6.12.4 提醒调度：重复调度但幂等

每次调用 `GrantVaultAccessToUser` 都会触发 `scheduleContactReminders()`（第 92-111 行），遍历所有联系人提醒并调用 `ScheduleContactReminderForUser`。

`ScheduleContactReminderForUser::schedule()` 第 85 行使用 `syncWithoutDetaching`：

```php
$this->contactReminder->userNotificationChannels()->syncWithoutDetaching([$channel->id => [
    'scheduled_at' => $this->upcomingDate->tz('UTC'),
]]);
```

**行为**：`syncWithoutDetaching` 是幂等操作——如果关联已存在则更新 `scheduled_at`，不存在则创建。

**影响**：重复授权不会产生重复的提醒调度记录，只会刷新调度时间。这是唯一在重复关系下行为正确的操作。

---

## 七、完整协同流程

### 7.1 两种合法路径对比

```
路径 A（先激活再授权）                  路径 B（先授权再激活）
━━━━━━━━━━━━━━━━━━━━                  ━━━━━━━━━━━━━━━━━━━━
① InviteUser                          ① InviteUser
   创建 User (invitation_code)            创建 User (invitation_code)
   发送邮件                               发送邮件
       ↓                                    ↓
② AcceptInvitation                    ③ GrantVaultAccessToUser
   is_account_administrator = false      创建 Contact (first_name=null!)
   设置密码                               写入 user_vault (permission)
   email_verified_at = now                调度提醒
   invitation_accepted_at = now              ↓
       ↓                                ② AcceptInvitation
③ GrantVaultAccessToUser                is_account_administrator = false
   创建 Contact (姓名已填)               设置密码
   写入 user_vault (permission)          email_verified_at = now
   调度提醒                              invitation_accepted_at = now
```

路径 B 的副作用：联系人记录的 `first_name`/`last_name` 为空（因为用户尚未填写个人信息）。

### 7.2 安全性保障机制

1. **邀请码机制**：UUID 格式，不可预测，一次性使用（`whereNull('invitation_accepted_at')` 防止重复接受）
2. **管理员标记重置**：接受邀请时 `is_account_administrator` 强制置 false，防止邀请阶段的管理员标记被继承
3. **权限层级**：数值越小权限越高，`<=` 比较确保高权限自动包含低权限
4. **服务层权限校验**：每个 Service 通过 `permissions()` 方法声明所需权限，`BaseService` 统一验证
5. **Gate 门面校验**：控制器和策略层可通过 Gate 快速校验
6. **路由中间件**：`auth:sanctum` + `verified` 确保操作者已认证，`can:vault-manager,vault` 确保操作者有保险柜管理权限
7. **外键级联删除**：删除用户/保险柜/联系人时自动清理关联
8. **拒绝自我操作**：Grant/Change/Remove 服务均检查 `$this->user->id === $this->author->id`，防止用户修改自己的权限
9. **双重权限校验**：修改保险柜权限（`ChangeVaultAccess`）需同时满足 vault manager 和 account administrator

### 7.3 创建保险柜时的初始权限

`CreateVault`（`app/Domains/Vault/ManageVault/Services/CreateVault.php`）的 `createUserContact()` 方法：

```php
private function createUserContact(): void
{
    $contact = Contact::create([
        'vault_id' => $this->vault->id,
        'first_name' => $this->author->first_name,
        'last_name' => $this->author->last_name,
        'can_be_deleted' => false,
        'template_id' => $this->vault->default_template_id,
    ]);

    $this->vault->users()->save($this->author, [
        'permission' => Vault::PERMISSION_MANAGE,
        'contact_id' => $contact->id,
    ]);
}
```

保险柜的创建者自动获得 MANAGE 权限。

---

## 八、辅助工具

### 8.1 VaultHelper

`app/Helpers/VaultHelper.php`：

- `getPermission(User $user, Vault $vault): ?int` - 获取用户在保险柜中的权限等级（带 array 缓存，5 秒 TTL）
- `getPermissionFriendlyName(int $permission): string` - 将权限数值转换为友好名称（Manager/Editor/Viewer）

### 8.2 VaultPolicy

`app/Policies/VaultPolicy.php`，Laravel 策略类：

- `viewAny` - 任何人都可以查看列表
- `view` - 需要 `vault-viewer` 权限
- `create` - 任何人都可以创建
- `update` - 需要 `vault-editor` 权限
- `delete` - 需要 `vault-manager` 权限

---

## 九、关键文件索引

| 模块 | 文件路径 | 作用 |
|------|----------|------|
| 账号邀请 | `app/Models/User.php` | 用户模型，含 invitation_code/invitation_accepted_at 字段 |
| 账号邀请 | `app/Domains/Settings/ManageUsers/Services/InviteUser.php` | 邀请用户服务 |
| 账号邀请 | `app/Domains/Settings/ManageUsers/Services/AcceptInvitation.php` | 接受邀请服务 |
| 账号邀请 | `app/Http/Controllers/Auth/AcceptInvitationController.php` | 邀请页面控制器 |
| 邮件通知 | `app/Mail/UserInvited.php` | 邀请邮件类 |
| 邮件通知 | `resources/views/emails/user/invitation.blade.php` | 邮件模板 |
| 保险柜授权 | `app/Models/Vault.php` | 保险柜模型，权限常量 |
| 保险柜授权 | `app/Domains/Vault/ManageVaultSettings/Services/GrantVaultAccessToUser.php` | 授予权限服务 |
| 保险柜授权 | `app/Domains/Vault/ManageVaultSettings/Services/ChangeVaultAccess.php` | 修改权限服务 |
| 保险柜授权 | `app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php` | 移除权限服务 |
| 保险柜授权 | `app/Domains/Vault/ManageVaultSettings/Web/Controllers/VaultSettingsUserController.php` | 用户权限控制器 |
| 保险柜授权 | `app/Domains/Vault/ManageVaultSettings/Web/ViewHelpers/VaultSettingsIndexViewHelper.php` | 前端视图辅助（含用户过滤逻辑） |
| 权限校验 | `app/Providers/AuthServiceProvider.php` | Gate 权限定义 |
| 权限校验 | `app/Services/BaseService.php` | 基础服务，统一权限校验 |
| 权限校验 | `app/Policies/VaultPolicy.php` | 模型策略类 |
| 数据库 | `database/migrations/2014_10_12_000000_create_users_table.php` | users 表迁移（含邀请字段） |
| 数据库 | `database/migrations/2020_04_25_133132_create_contacts_table.php` | user_vault 中间表迁移 |
| 辅助工具 | `app/Helpers/VaultHelper.php` | 权限辅助方法 |
| 路由 | `routes/web.php` | 路由定义（含中间件保护） |
