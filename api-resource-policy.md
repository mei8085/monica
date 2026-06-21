# API Resource 与授权策略配合机制分析

## 一、整体架构脉络

Monica CRM 的权限体系设计了 **两条平行但不同的执行路径**，分别服务于 Web 界面和 API 接口：

```
┌─────────────────────────────────────────────────────────────────────┐
│                         请求进入点                                    │
├─────────────────────┬───────────────────────────────────────────────┤
│   Web 路由 (web)    │            API 路由 (api)                     │
│  (session/auth)     │     (auth:sanctum + Token Abilities)          │
└─────────┬───────────┴───────────────────────┬───────────────────────┘
          │                                   │
          ▼                                   ▼
┌─────────────────────┐            ┌─────────────────────────┐
│  Web 控制器          │            │   API 控制器             │
│  - authorizeResource│            │  - abilities 中间件       │
│  - Gate::authorize  │            │  - 作用域查询过滤         │
│  (Policy → Gate)    │            │  - 调用 Service          │
└─────────┬───────────┘            └────────────┬────────────┘
          │                                     │
          ▼                                     ▼
┌─────────────────────┐            ┌─────────────────────────┐
│      Gate 层         │            │   BaseService 层         │
│  (AuthServiceProvider│            │  (自定义权限依赖图)       │
│   定义 8 个 Gate)    │            │  - 权限依赖校验          │
└─────────────────────┘            │  - 作用域存在性校验       │
                                   └────────────┬────────────┘
                                                │
                                                ▼
                                   ┌─────────────────────────┐
                                   │    API Resource 层       │
                                   │  (纯数据格式化，无权限)   │
                                   │  VaultResource          │
                                   │  UserResource           │
                                   └─────────────────────────┘
```

---

## 二、核心组件详解

### 2.1 Gate 定义层 - [AuthServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Providers/AuthServiceProvider.php#L28-L115)

Gate 在 `boot()` 方法中集中定义，共 8 个权限门，分为三类：

| Gate 名称 | 用途 | 判断逻辑 |
|-----------|------|----------|
| `administrator` | 账户管理员 | `$user->is_account_administrator` |
| `vault-viewer` | Vault 查看者 | 用户通过中间表关联该 vault 即可 |
| `vault-editor` | Vault 编辑者 | 中间表 `permission <= 200` |
| `vault-manager` | Vault 管理者 | 中间表 `permission <= 100` |
| `contact-owner` | 接触归属权 | contact.vault_id == vault_id |
| `group-owner` | 分组归属权 | group.vault_id == vault_id |
| `journal-owner` | 日志归属权 | journal.vault_id == vault_id |
| `post-owner` | 文章归属权 | post.journal_id == journal_id |
| `sliceOfLife-owner` | 生活片段归属权 | sliceOfLife.journal_id == journal_id |

**辅助方法** `id()` [L112-L115](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Providers/AuthServiceProvider.php#L112-L115):
支持传入 Model 对象或原始 ID，自动提取主键，使 Gate 调用更灵活。

---

### 2.2 Policy 策略层 - [VaultPolicy.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Policies/VaultPolicy.php)

Policy 作为 Gate 的**外观包装层**，主要供 Web 控制器的 `authorizeResource` 使用。

| Policy 方法 | 对应 Gate |
|-------------|-----------|
| `viewAny()` | 直接返回 true（账户级查询由控制器作用域限制） |
| `view()` | `Gate::allows('vault-viewer', $vault)` |
| `create()` | 直接返回 true |
| `update()` | `Gate::allows('vault-editor', $vault)` |
| `delete()` | `Gate::allows('vault-manager', $vault)` |

> **关键点**：Policy 本身不做具体判断，全部委托给 Gate 实现。Policy 的存在是为了适配 Laravel 的 `authorizeResource` 自动授权约定。

---

### 2.3 BaseService 自定义权限层 - [BaseService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Services/BaseService.php)

这是 **API 通道的核心权限机制**，与 Gate 体系平行但独立。

#### 权限依赖图 [L41-L67](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Services/BaseService.php#L41-L67)

```
author_must_belong_to_account (无前置)
    ↓
author_must_be_account_administrator (需要 author_must_belong_to_account)

vault_must_belong_to_account (无前置)
    ↓
    ├── author_must_be_vault_manager  (需要 vault_must + author_must_belong)
    ├── author_must_be_vault_editor   (需要 vault_must + author_must_belong)
    └── author_must_be_in_vault       (需要 vault_must + author_must_belong)
          ↓
          ├── contact_must_belong_to_vault (需要前置 3 项)
          └── group_must_belong_to_vault   (需要前置 3 项)
```

#### 权限校验执行流程 - `validateRules()` [L96-L119](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Services/BaseService.php#L96-L119)

1. 先用 Laravel Validator 校验数据规则 `rules()`
2. 遍历 `permissionDependencies`，对 Service 声明的每个权限：
   - 先校验其依赖权限是否都已声明（防止遗漏）
   - 调用 `validatePermission()` 执行具体校验
3. 最后检查是否有未知权限名，防止拼写错误

#### 各权限具体校验逻辑

| 权限名 | 校验方法 [L159-L230] | 核心逻辑 |
|--------|---------------------|----------|
| `author_must_belong_to_account` | `validateAuthorBelongsToAccount()` | `User::where(account_id, author_id)->findOrFail()` |
| `author_must_be_account_administrator` | `validateAuthorIsAccountAdministrator()` | 检查 `$this->author->is_account_administrator` |
| `vault_must_belong_to_account` | `validateVaultExists()` | `Vault::where(account_id, vault_id)->findOrFail()` |
| `author_must_be_vault_manager` | `validateUserPermissionInVault(100)` | pivot `permission <= 100` |
| `author_must_be_vault_editor` | `validateUserPermissionInVault(200)` | pivot `permission <= 200` |
| `author_must_be_in_vault` | `validateUserPermissionInVault(300)` | pivot `permission <= 300` |
| `contact_must_belong_to_vault` | `validateContactBelongsToVault()` | `$vault->contacts()->findOrFail()` + 归属检查 |
| `group_must_belong_to_vault` | `validateGroupBelongsToVault()` | `$vault->groups()->findOrFail()` + 归属检查 |

> **设计优点**：通过依赖图自动保证校验顺序，先加载 author/vault 模型后，下游校验可复用 `$this->author` 和 `$this->vault` 属性。

---

### 2.4 Vault 权限常量 - [Vault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Models/Vault.php#L19-L23)

```php
PERMISSION_MANAGE = 100  // 管理（最高）
PERMISSION_EDIT   = 200  // 编辑
PERMISSION_VIEW   = 300  // 查看（最低）
```

数值设计采用**反向等级**：数值越小权限越高，判断统一用 `<=` 运算符。

---

### 2.5 API Resource 层 - 纯数据转换

| Resource 文件 | 输出字段 |
|---------------|----------|
| [VaultResource.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Http/Resources/VaultResource.php) | id, name, description, created_at, updated_at, links.self |
| [UserResource.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Http/Resources/UserResource.php) | id, name, email, created_at, updated_at, links.self |

> **重要特征**：Resource 层 **完全不包含权限判断**。所有权限校验都在进入 Resource 之前完成。Resource 只负责 DTO 转换。

---

## 三、完整流程分析（以 Vault 为例）

### 3.1 Web 通道 - 查询单个 Vault

**路径**：[Web VaultController](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php)

```
1. 构造函数执行 $this->authorizeResource(Vault::class, 'vault')
   ↓ 自动映射
2. 调用 VaultPolicy::view($user, $vault)
   ↓ 委托
3. Gate::forUser($user)->allows('vault-viewer', $vault)
   ↓ 查中间表
4. 用户vaults关联中是否存在该vault_id
   ├─ 通过 → 继续执行 show()，渲染 Inertia 页面
   └─ 不通过 → 抛出 AuthorizationException (403)
```

### 3.2 API 通道 - 查询单个 Vault (GET /api/vaults/{id})

**路径**：[API VaultController::show()](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Vault/ManageVault/Api/Controllers/VaultController.php#L73-L79)

```
1. 路由中间件 auth:sanctum → 验证 Token
   ↓
2. 控制器中间件 abilities:read → 验证 Token 具有 read 能力
   ↓
3. show() 方法执行作用域查询:
   $request->user()->account->vaults()->findOrFail($vaultId)
   ├─ 隐式过滤：只在当前账户的vaults中查找
   ├─ 找到 → 继续
   └─ 未找到 → ModelNotFoundException → ApiController 捕获返回 404
   ↓
4. 用返回的 $vault 构造 new VaultResource($vault)
   ↓
5. VaultResource::toArray() 格式化输出
```

> **API 设计特点**：用 `findOrFail()` + 关联查询实现"隐式权限"。不属于当前账户的 vault 直接返回 404（而非 403），避免信息泄露。

### 3.3 API 通道 - 更新 Vault (PUT /api/vaults/{id})

**路径**：[API VaultController::update()](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Vault/ManageVault/Api/Controllers/VaultController.php#L92-L105)

```
1. 路由中间件 auth:sanctum → Token 验证
   ↓
2. 控制器中间件 abilities:write → Token 具有 write 能力
   ↓
3. 调用 (new UpdateVault)->execute($data)
   ↓ 进入 BaseService::validateRules()
   ├─ 步骤A: 校验数据规则 (account_id/author_id/vault_id/name 等)
   ├─ 步骤B: 按 UpdateVault::permissions() 声明顺序校验
   │   ├─ 权限1: author_must_belong_to_account
   │   │      → 加载 $this->author，确保属于该账户
   │   ├─ 权限2: vault_must_belong_to_account
   │   │      → 加载 $this->vault，确保属于该账户
   │   └─ 权限3: author_must_be_vault_manager
   │          → 查中间表 pivot permission <= 100 (MANAGE)
   │          ├─ 通过 → 继续
   │          └─ 不通过 → 抛出 NotEnoughPermissionException
   └─ 步骤C: 全部通过后进入实际 execute() 逻辑
   ↓
4. 更新 $this->vault 属性并 save()
   ↓
5. 返回 new VaultResource($vault)
```

UpdateVault 声明的权限 [L28-L35](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Vault/ManageVault/Services/UpdateVault.php#L28-L35)：
```php
['author_must_belong_to_account', 'vault_must_belong_to_account', 'author_must_be_vault_manager']
```

### 3.4 Web 通道 - 创建 Journal (示例 Gate::authorize 用法)

**路径**：[JournalController::store()](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Vault/ManageJournals/Web/Controllers/JournalController.php#L43-L61)

```
1. Gate::authorize('vault-editor', $vaultId)
   → 先确保用户至少有编辑权限
   ↓
2. 调用 CreateJournal Service
   → Service 内部再次用 BaseService 机制校验
   → 双重检查（Web 层防御 + Service 层强制）
```

---

## 四、两种权限体系对比

| 维度 | Gate + Policy 体系 | BaseService 自定义体系 |
|------|-------------------|----------------------|
| 主要服务对象 | Web 控制器 | API / Service 内部调用 |
| 触发方式 | authorizeResource / Gate::authorize | Service::permissions() 声明 |
| 错误响应 | AuthorizationException (403 页面) | NotEnoughPermissionException |
| 模型加载 | 依赖路由模型绑定（先加载再校验） | 校验过程中主动 findOrFail |
| 归属检查方式 | Gate 闭包中查询 | Service 按依赖图逐步加载 |
| 可复用性 | 控制器层 | 任何调用 Service 的地方（CLI / Queue 等） |

---

## 五、API Token Abilities 中间件 - [bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/bootstrap/app.php#L27-L28)

```php
'abilities' => CheckAbilities::class,      // 需要具备所有能力
'ability'   => CheckForAnyAbility::class,  // 任一能力即可
```

在 API 控制器中使用：
```php
// VaultController 构造函数
$this->middleware('abilities:read')->only(['index', 'show']);
$this->middleware('abilities:write')->only(['store', 'update', 'delete']);
```

这是 API 的 **第一道粗粒度防线**，后续由 Service 层做细粒度校验。

---

## 六、总结：资源输出前的完整权限判断链

以 **API GET /api/vaults** 返回 `VaultResource::collection()` 为例，完整判断链路：

```
                     请求进入
                        │
      ┌─────────────────┴─────────────────┐
      │  1. auth:sanctum 中间件            │
      │     Token 是否有效？               │
      └─────────────────┬─────────────────┘
                        │ 通过
      ┌─────────────────┴─────────────────┐
      │  2. abilities:read 中间件          │
      │     Token 是否有 read 能力？       │
      └─────────────────┬─────────────────┘
                        │ 通过
      ┌─────────────────┴─────────────────┐
      │  3. ApiController 构造函数         │
      │     limit 参数校验(可选)           │
      └─────────────────┬─────────────────┘
                        │ 通过
      ┌─────────────────┴─────────────────┐
      │  4. 控制器方法 index()             │
      │     $user->account->vaults()      │
      │     ↳ 只查当前账户的 vaults        │
      │     ↳ 分页 paginate(limit)        │
      └─────────────────┬─────────────────┘
                        │ 返回模型集合
      ┌─────────────────┴─────────────────┐
      │  5. VaultResource::collection()   │
      │     ↳ 遍历每个模型调用 toArray()  │
      │     ↳ 纯格式化，不做权限判断       │
      └─────────────────┬─────────────────┘
                        │
                  JSON 响应输出
```

> **结论**：API Resource 本身是"哑巴层"，只负责 JSON 序列化。权限的 **三道防线** 分别在：
> 1. **路由层**：Sanctum 认证 + Token Abilities（粗粒度）
> 2. **控制器层**：通过关联关系 `$user->account->...` 做账户级数据隔离
> 3. **Service 层**：BaseService 权限依赖图做细粒度权限 + 归属双重检查（写操作必走）

Web 通道则使用 `Policy → Gate` 链条实现同等效果，但写操作最终也会落到 Service 层的二次校验，形成"双重保险"。
