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

---

## 七、异常处理链路与 HTTP 状态码映射

### 7.1 异常处理器 - [Handler.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Exceptions/Handler.php)

Monica 的异常处理器**极度精简**——没有覆盖 `render()` 方法，也没有注册任何自定义 `renderable` 回调。唯一的自定义逻辑是将异常上报到 Sentry。

这意味着：**所有异常到 HTTP 响应的转换，完全依赖 Laravel 框架的默认行为**，外加少量控制器级别的手动捕获。

### 7.2 各类异常 → HTTP 状态码完整映射

#### 场景 A：Sanctum Token 无效或缺失

| 触发点 | 抛出的异常 | 最终 HTTP 状态码 | 响应体 |
|--------|-----------|-----------------|--------|
| `auth:sanctum` 中间件 | `AuthenticationException` | **401** | Laravel 默认 JSON: `{"message": "Unauthenticated."}` |
| `abilities:read` / `abilities:write` | `MissingAbilityException` | **403** | Laravel Sanctum 默认 JSON: `{"message": "Invalid ability provided."}` |

> `CheckAbilities` 中间件（[bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/bootstrap/app.php#L27)）在 Token 缺少所需能力时抛出 `MissingAbilityException`，Laravel 默认将其转为 403。

#### 场景 B：API 控制器中的数据查询失败

ApiController 的 `callAction()` [L54-L65](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Http/Controllers/ApiController.php#L54-L65) 手动捕获了三种异常：

| 触发点 | 捕获的异常 | 最终 HTTP 状态码 | 响应体 |
|--------|-----------|-----------------|--------|
| `findOrFail()` 在关联查询中 | `ModelNotFoundException` | **404** | `{"error": {"message": "The resource has not been found", "error_code": 31}}` |
| SQL 约束违反等 | `QueryException` | **500** | `{"error": {"message": "Invalid query", "error_code": 40}}` |
| 请求验证失败 | `ValidationException` | **422** | `{"error": {"message": [...], "error_code": 32}}` |

#### 场景 C：BaseService 权限不足

| 触发点 | 抛出的异常 | 最终 HTTP 状态码 | 响应体 |
|--------|-----------|-----------------|--------|
| `validateUserPermissionInVault()` 失败 | `NotEnoughPermissionException` | **500**（⚠️ 见下文分析） | Laravel 默认: `{"message": "Server Error"}` |
| `findOrFail()` 在 Service 校验中 | `ModelNotFoundException` | 被 ApiController 捕获 → **404** | 同场景 B |
| `validateAuthorIsAccountAdministrator()` 失败 | `NotEnoughPermissionException` | **500**（⚠️ 见下文分析） | Laravel 默认: `{"message": "Server Error"}` |

> ⚠️ **关键发现**：`NotEnoughPermissionException` 继承自 `\Exception`，不是 `AccessDeniedHttpException` 或 `AuthorizationException`。Laravel 框架无法将其自动识别为 403，默认转为 **500 Internal Server Error**。这在 API 通道中是一个设计缺陷——Web 通道通过 `Gate::authorize()` 抛出的 `AuthorizationException`（Laravel 内置类）会正确转为 403，但 Service 层的 `NotEnoughPermissionException` 走了不同的异常层次结构。

#### 场景 D：Web 通道 Gate/Policy 授权失败

| 触发点 | 抛出的异常 | 最终 HTTP 状态码 | 响应体 |
|--------|-----------|-----------------|--------|
| `$this->authorizeResource()` / `Gate::authorize()` | `AuthorizationException` | **403** | Inertia 渲染 403 页面（Web），或 JSON `{"message": "This action is unauthorized."}`（AJAX） |
| 路由中间件 `can:vault-viewer,vault` | `AuthorizationException` | **403** | 同上 |

#### 场景 E：Web 通道未登录

| 触发点 | 抛出的异常 | 最终 HTTP 状态码 | 行为 |
|--------|-----------|-----------------|------|
| `auth:sanctum` + `verified` 中间件 | `AuthenticationException` | **302** → 重定向 | 重定向到登录页 [Authenticate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Http/Middleware/Authenticate.php#L15-L18) |
| JSON 请求时的 `AuthenticationException` | - | **401** | `{"message": "Unauthenticated."}` |

### 7.3 状态码速查总表

| 状态码 | 含义 | 触发场景 | 通道 |
|--------|------|---------|------|
| **401** | 未认证 | Token 无效/缺失；Web 未登录 + JSON 请求 | API + Web |
| **403** | 无权限 | Gate/Policy 拒绝；Token 缺少 ability | API + Web |
| **404** | 资源不存在 | 关联查询 findOrFail 失败（含隐式权限过滤） | API |
| **422** | 数据校验失败 | ValidationException | API + Web |
| **500** | 服务器错误 | NotEnoughPermissionException ⚠️；QueryException | API |

---

## 八、多种入口通道的中间件链路

### 8.1 API Token 通道（REST API）

**入口**：`/api/*` 路由

```
请求 → api 中间件组
  ├─ throttleApi (限流)
  ├─ EnsureFrontendRequestsAreStateful (Sanctum，使前端 Cookie 请求也能通过)
  ├─ SubstituteBindings (路由模型绑定)
  │
  ├─ auth:sanctum (Token 认证，无 Token → 401)
  │
  ├─ abilities:read 或 abilities:write (Token 能力，不足 → 403)
  │
  └─ ApiController 构造函数 (limit 校验 → 400)
```

配置来源：[bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/bootstrap/app.php#L41-L43)，[routes/api.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/routes/api.php#L18)

### 8.2 DAV 同步通道（CardDAV / CalDAV）

**入口**：`/dav/*` 路由（由 LaravelSabre 注册）

DAV 通道有**独特的双重认证机制**——同时支持 HTTP Basic Auth 和 Session Cookie。

```
请求 → laravelsabre 中间件组 [配置: laravelsabre.php]
  ├─ api (API 中间件组基础)
  ├─ EnsureDavRequestsAreStateful ← 核心分叉逻辑
  │   ├─ 已有 Session 登录？
  │   │   ├─ 是 (本地环境) → SanctumSetUser
  │   │   │     给当前用户附加 TransientToken（虚拟 Token，拥有全部能力）
  │   │   └─ 否 → 进入 Basic Auth 流程:
  │   │         ├─ 修改 Sanctum 使其同时从 HTTP Password 字段提取 Token
  │   │         ├─ EncryptCookies → AddQueuedCookies → StartSession → VerifyCsrfToken
  │   │         └─ AuthenticateWithTokenOnBasicAuth
  │   │               ├─ 用 Sanctum 验证 Token → 成功则设置用户
  │   │               └─ 失败 → 抛出 UnauthorizedHttpException → 401
  │   │                     响应头: WWW-Authenticate: Basic realm="..."
  │   │
  ├─ abilities:read,write (DAV 需要 Token 同时有读写能力)
  │
  └─ Sabre DAV Server 内部
      ├─ AuthPlugin (AuthBackend::check → Auth::check())
      │     未认证 → 401 + WWW-Authenticate: Bearer
      ├─ AclPlugin (allowUnauthenticatedAccess = false)
      └─ 业务处理 (CardDAV / CalDAV)
```

关键文件：
- [laravelsabre.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/config/laravelsabre.php#L52-L56) - DAV 中间件配置
- [EnsureDavRequestsAreStateful.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Http/Middleware/EnsureDavRequestsAreStateful.php) - DAV 认证分叉
- [AuthenticateWithTokenOnBasicAuth.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Http/Middleware/AuthenticateWithTokenOnBasicAuth.php) - Basic Auth Token 映射
- [AuthBackend.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Contact/Dav/Web/Auth/AuthBackend.php) - Sabre 层认证
- [DAVServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Providers/DAVServiceProvider.php) - Sabre 插件注册

> **DAV 认证的特殊之处**：CalDAV/CardDAV 客户端（如 macOS Contacts、Thunderbird）不支持 Bearer Token，只能通过 HTTP Basic Auth 发送凭据。`AuthenticateWithTokenOnBasicAuth` 将 Basic Auth 的 password 字段映射为 Sanctum Token，兼容标准 DAV 客户端。

### 8.3 Web + WebAuthn 通道（浏览器）

**入口**：Web 路由 + Fortify 登录

```
请求 → web 中间件组
  ├─ EncryptCookies
  ├─ AddQueuedCookies
  ├─ StartSession
  ├─ SetLocale (CodeZero Localizer)
  ├─ SubstituteBindings (路由模型绑定)
  ├─ HandleInertiaRequests (Inertia 协议头处理)
  ├─ AddLinkHeadersForPreloadedAssets
  │
  ├─ auth:sanctum (Session 认证)
  ├─ jetstream.auth_session (AuthenticateSession，验证 Session 绑定)
  ├─ verified (邮箱验证)
  │
  └─ 控制器层权限:
      ├─ authorizeResource(Vault::class, 'vault') → Policy → Gate
      ├─ Route::middleware('can:vault-viewer,vault') → Gate 直接检查
      └─ Route::middleware('can:administrator') → Gate 直接检查
```

#### WebAuthn 二次验证流程

WebAuthn 不改变上述中间件链路，而是**插入登录流程内部**。

```
用户提交密码 → Fortify Login Pipeline
  ├─ RedirectIfTwoFactorAuthenticatable [L28-L38]
  │   ├─ 验证邮箱+密码 → 失败: ValidationException (422)
  │   ├─ 检查是否启用了 2FA 或 WebAuthn
  │   │   ├─ 是 → 重定向到二次验证页面
  │   │   │   (two-factor.login 或 webauthn.authenticate)
  │   │   └─ 否 → 直接登录
  │   └─
  └─ 二次验证:
      ├─ TOTP 验证码 (标准 2FA)
      └─ WebAuthn 断言验证
          └─ AttemptToAuthenticateWebauthn [L32-L42]
              ├─ validateAssertion(user, credentials)
              │   ├─ 通过 → 登录成功
              │   └─ 失败 → ValidationException (422)
              └─ guard->attempt() 兜底
```

关键文件：
- [RedirectIfTwoFactorAuthenticatable.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php#L28-L38) - 登录流程分叉
- [AttemptToAuthenticateWebauthn.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Actions/AttemptToAuthenticateWebauthn.php) - WebAuthn 断言验证

> **WebAuthn 不走独立的中间件通道**。`webauthn` 中间件别名已在 [bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/bootstrap/app.php#L29) 注册，但项目路由中**未实际使用**该中间件。WebAuthn 仅作为登录流程的一环，通过 Fortify Pipeline 集成。

#### auth 配置中的 WebAuthn 用户提供者

[auth.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/config/auth.php#L62-L66) 配置用户提供者为 `webauthn` driver：

```php
'providers' => [
    'users' => [
        'driver' => 'webauthn',
        'model' => User::class,
    ],
],
```

这使得 `web` guard 的用户查询支持 WebAuthn 凭据验证，但**不影响中间件链路**——它只决定了如何从数据库加载用户。

---

## 九、四种通道对比总览

| 通道 | 认证方式 | 授权中间件 | 权限不足时状态码 | 响应格式 |
|------|---------|-----------|----------------|---------|
| **REST API** | Sanctum Bearer Token | `auth:sanctum` + `abilities:read/write` | 401 (无Token) / 403 (能力不足) / 404 (资源不属于你) / 500 ⚠️ (Service权限不足) | JSON |
| **DAV 同步** | Basic Auth (password=Token) 或 Session Cookie | `EnsureDavRequestsAreStateful` + `abilities:read,write` + Sabre AuthPlugin | 401 (WWW-Authenticate: Basic/Bearer) | Sabre XML |
| **Web 浏览器** | Session + Cookie | `auth:sanctum` + `can:*` 中间件 / `authorizeResource` | 302 → 登录页 (未登录) / 403 (Gate拒绝) | Inertia HTML |
| **WebAuthn 二次验证** | 内嵌于 Web 登录流程 | 无独立中间件（通过 Fortify Pipeline） | 422 (断言失败) | Inertia HTML / JSON |

---

## 十、异常链路时序图（API 通道写操作）

以 `PUT /api/vaults/{id}` 权限不足为例，展示完整异常传播路径：

```
PUT /api/vaults/{id}
  │
  ├─ auth:sanctum ─── Token 无效 ──→ AuthenticationException
  │                                    ↓
  │                              Laravel Handler 默认渲染
  │                                    ↓
  │                              401 {"message":"Unauthenticated."}
  │
  ├─ abilities:write ── Token 缺少 write ──→ MissingAbilityException
  │                                           ↓
  │                                     Laravel Handler 默认渲染
  │                                           ↓
  │                                     403 {"message":"Invalid ability provided."}
  │
  ├─ UpdateVault->execute($data)
  │   │
  │   ├─ validateAuthorBelongsToAccount() ──→ findOrFail 失败
  │   │                                       ↓
  │   │                                 ModelNotFoundException
  │   │                                       ↓
  │   │                              ApiController::callAction 捕获
  │   │                                       ↓
  │   │                              404 {"error":{"message":"The resource has not been found","error_code":31}}
  │   │
  │   └─ validateUserPermissionInVault(100) ──→ 用户只有 EDIT(200) 权限
  │                                            ↓
  │                                      NotEnoughPermissionException
  │                                            ↓
  │                                  ⚠️ 无自定义 render，Laravel 默认
  │                                            ↓
  │                                  500 {"message":"Server Error"}
  │                                  (debug 模式下会暴露堆栈)
  │
  └─ 成功 → 200 + VaultResource JSON
```

> **设计隐患**：`NotEnoughPermissionException` 在 API 通道中应返回 403 而非 500。修复方案有两种：
> 1. 在 `Handler.php` 中注册 `renderable` 回调，将 `NotEnoughPermissionException` 映射为 403
> 2. 让 `NotEnoughPermissionException` 继承 `AuthorizationException` 而非 `Exception`
