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

---

## 十一、Token Abilities 颁发与存储链路

### 11.1 Abilities 定义与默认值 - [JetstreamServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Providers/JetstreamServiceProvider.php#L48-L56)

Monica 的 Token abilities 由 Jetstream 管理，仅定义了 **两种能力**：

```php
Jetstream::defaultApiTokenPermissions(['read']);  // 新 Token 默认只有 read
Jetstream::permissions([
    'read',   // 读操作权限
    'write',  // 写操作权限
]);
```

| 能力 | 用途 | 对应接口 |
|------|------|---------|
| `read` | 只读查询 | GET 接口 |
| `write` | 数据修改 | POST/PUT/DELETE 接口 |

### 11.2 数据库存储结构 - [2019_12_14_000001_create_personal_access_tokens_table.php](file:///d:/fz/0601-2\solo-dogfeeding\code\70-monica\database\migrations\2019_12_14_000001_create_personal_access_tokens_table.php#L16-L25)

Token 存储在 `personal_access_tokens` 表中，核心字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `tokenable_type` | morphs | 关联模型类型（通常是 `App\Models\User`） |
| `tokenable_id` | morphs | 关联模型 ID |
| `name` | string | Token 名称（用户备注用） |
| `token` | string(64) | 哈希后的 Token 串（唯一索引） |
| `abilities` | text, nullable | **JSON 格式的能力数组**，如 `["read","write"]` |
| `last_used_at` | timestamp | 最后使用时间 |
| `expires_at` | timestamp | 过期时间 |

> **重要**：`abilities` 字段以 JSON 字符串存储，Laravel Sanctum 会自动将其序列化为数组。

### 11.3 Token 创建流程

Monica 使用 Jetstream 内置的 API Token 管理功能，Token 创建完全通过 Jetstream 标准路由完成，**项目代码中无自定义创建逻辑**。

```
用户在 Web 界面创建 Token
  ↓
POST /user/api-tokens  (Jetstream 标准路由)
  ↓
Jetstream 内部处理:
  ├─ 读取 JetstreamServiceProvider 配置
  ├─ defaultApiTokenPermissions = ['read']  ← 用户未勾选 write 时
  │  或 permissions 用户勾选的 ['read','write'] ← 用户勾选 write 时
  ├─ 生成随机 40 字符明文 Token (仅首次显示给用户)
  ├─ SHA-256 哈希后存入 `token` 字段
  ├─ abilities 以 JSON 数组存入 `abilities` 字段
  ↓
返回明文 Token 给用户 (只显示一次)
```

创建后的数据库记录示例：
```
tokenable_type: App\Models\User
tokenable_id:   {uuid}
name:           "My Script Token"
token:          sha256('abc123...') (64位哈希)
abilities:      ["read","write"]    (JSON)
```

### 11.4 Token Abilities 更新流程 - [ApiTokenPermissionsTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/tests/Feature/Auth/ApiTokenPermissionsTest.php#L31-L41)

Token 权限可通过 Web 界面修改，走 Jetstream 标准路由：

```
PUT /user/api-tokens/{tokenId}
  ↓
Jetstream 内部处理:
  ├─ 验证提交的 permissions 数组
  ├─ 只保留在 Jetstream::permissions() 中声明过的有效能力
  │  (无效能力如 'missing-permission' 会被静默过滤)
  ├─ 更新 `abilities` 字段为新的 JSON 数组
  ↓
返回成功
```

测试中的示例：
```php
$token = $user->tokens()->create([
    'name' => 'Test Token',
    'token' => Str::random(40),
    'abilities' => ['read'],           // 初始只有 read
]);

// 提交更新: ['write', 'missing-permission']
// 结果: abilities 变为 ["write"]（invalid 被过滤）
$token->can('write');      // true
$token->can('read');       // false (被覆盖了)
$token->can('missing-permission');  // false
```

### 11.5 Token Abilities 验证流程

**第一步：Sanctum 解析 Token**

请求携带 `Authorization: Bearer {plainTextToken}`，`auth:sanctum` 中间件：
1. 对 plainTextToken 做 SHA-256 哈希
2. 在 `personal_access_tokens` 表中匹配 `token` 字段
3. 加载关联的 User 模型并设置到 `Auth::user()`
4. Token 模型挂载到 `$user->currentAccessToken()`

**第二步：CheckAbilities 中间件验证**

`abilities:read` 中间件（[bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/bootstrap/app.php#L27)）的逻辑：

```php
// 伪代码
public function handle($request, $next, ...$abilities)
{
    $token = $request->user()->currentAccessToken();
    
    // 检查 Token 是否具备 ALL 所需能力
    foreach ($abilities as $ability) {
        if (! $token->can($ability)) {
            throw new MissingAbilityException($ability);
            // Laravel 默认转为 403 {"message":"Invalid ability provided."}
        }
    }
    
    return $next($request);
}
```

`$token->can($ability)` 的判断逻辑：
1. 如果 `abilities` 字段为 `null` → 返回 `true`（无限制）
2. 否则检查 `$ability` 是否在 `abilities` 数组中

> **注意**：DAV 通道中的 `TransientToken`（虚拟 Token）的 `can()` 方法始终返回 `true`，因此 DAV 认证后的请求拥有全部能力。

---

## 十二、所有 API 路由的 Abilities 中间件挂载清单

### 12.1 API 路由总览 - [routes/api.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/routes/api.php#L18-L25)

```php
Route::middleware('auth:sanctum')->name('api.')->group(function () {
    Route::get('user', [UserController::class, 'user']);
    Route::apiResource('users', UserController::class)->only(['index', 'show']);
    Route::apiResource('vaults', VaultController::class);
});
```

### 12.2 各控制器的 Abilities 中间件挂载详情

#### [VaultController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Vault/ManageVault/Api/Controllers/VaultController.php#L21-L27)

```php
public function __construct()
{
    $this->middleware('abilities:read')->only(['index', 'show']);
    $this->middleware('abilities:write')->only(['store', 'update', 'delete']);
    parent::__construct();
}
```

| 方法 | HTTP 动词 | 路由 | 需要 ability | 说明 |
|------|----------|------|-------------|------|
| `index` | GET | `/api/vaults` | `read` | 列出所有 vaults |
| `show` | GET | `/api/vaults/{vault}` | `read` | 获取单个 vault |
| `store` | POST | `/api/vaults` | `write` | 创建 vault |
| `update` | PUT | `/api/vaults/{vault}` | `write` | 更新 vault |
| `destroy` | DELETE | `/api/vaults/{vault}` | `write` | 删除 vault |

#### [UserController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Settings/ManageUsers/Api/Controllers/UserController.php#L18-L23)

```php
public function __construct()
{
    $this->middleware('abilities:read');  // 所有方法都需要 read
    parent::__construct();
}
```

| 方法 | HTTP 动词 | 路由 | 需要 ability | 说明 |
|------|----------|------|-------------|------|
| `user` | GET | `/api/user` | `read` | 获取当前登录用户 |
| `index` | GET | `/api/users` | `read` | 列出账户内所有用户 |
| `show` | GET | `/api/users/{user}` | `read` | 获取单个用户 |

### 12.3 挂载总表

| 资源 | 方法 | ability | 读写类型 |
|------|------|---------|---------|
| GET `/api/vaults` | index | read | **读** |
| GET `/api/vaults/{vault}` | show | read | **读** |
| POST `/api/vaults` | store | write | **写** |
| PUT `/api/vaults/{vault}` | update | write | **写** |
| DELETE `/api/vaults/{vault}` | destroy | write | **写** |
| GET `/api/user` | user | read | **读** |
| GET `/api/users` | index | read | **读** |
| GET `/api/users/{user}` | show | read | **读** |

> **目前 API 接口数量较少**，仅 2 个资源、8 个端点。所有写接口（3 个）都正确挂载了 `abilities:write`，所有读接口（5 个）都挂载了 `abilities:read`。

---

## 十三、DAV Principal 与 ACL 判定实现

### 13.1 Principal 定义 - [PrincipalBackend.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Contact/Dav/Web/DAVACL/PrincipalBackend.php)

Sabre DAV 使用 **Principal** 概念表示"可被授权的实体"，Monica 中 Principal 直接对应用户。

Principal URI 格式：
```php
public const PRINCIPAL_PREFIX = 'principals/';

public static function getPrincipalUser(User $user): string
{
    return static::PRINCIPAL_PREFIX.$user->email;
}
```

实际示例：`principals/john@example.com`

Principal 包含的属性：
```php
[
    'uri' => 'principals/john@example.com',
    '{DAV:}displayname' => 'John Doe',
    '{http://sabredav.org/ns}email-address' => 'john@example.com',
]
```

### 13.2 DAV Server 启动时的 Principal 注入 - [DAVServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Providers/DAVServiceProvider.php#L42-L56)

每次 DAV 请求进入 Sabre Server 时，在 `nodes()` 回调中绑定当前用户：

```php
private function nodes(): array
{
    $user = Auth::user();  // 从 Laravel Auth 获取已认证用户
    
    // 注入当前用户到各个 Backend
    $principalBackend = app(PrincipalBackend::class, ['user' => $user]);
    $carddavBackend = app(CardDAVBackend::class)->withUser($user);
    $caldavBackend = app(CalDAVBackend::class)->withUser($user);

    return [
        new PrincipalCollection($principalBackend),  // principals/ 根节点
        new AddressBookRoot($principalBackend, $carddavBackend),  // 通讯录根
        new CalendarRoot($principalBackend, $caldavBackend),      // 日历根
    ];
}
```

### 13.3 WithUser Trait - [WithUser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Contact/Dav/Web/Backend/WithUser.php)

所有 DAV Backend 共享此 trait，用于在 Backend 实例中保存当前用户引用：

```php
trait WithUser
{
    protected User $user;

    public function withUser(User $user): self
    {
        $this->user = $user;
        return $this;
    }
}
```

### 13.4 ACL 插件配置 - [DAVServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Providers/DAVServiceProvider.php#L80-L84)

Sabre AclPlugin 配置：

```php
$aclPlugin = new AclPlugin;
$aclPlugin->allowUnauthenticatedAccess = false;  // 禁止未认证访问
$aclPlugin->hideNodesFromListings = true;        // 无权限则不显示在列表中
yield $aclPlugin;
```

| 配置项 | 值 | 效果 |
|--------|----|------|
| `allowUnauthenticatedAccess` | `false` | 未认证用户直接返回 401，不会进入 ACL 判定 |
| `hideNodesFromListings` | `true` | 对无权限的节点，在 PROPFIND 列表中直接隐藏（而非返回 403） |

### 13.5 基于 Principal 的资源过滤（ACL 第一层）

Sabre AclPlugin 会检查每个 DAV 资源的 ACL 属性。Monica 中 AddressBookHome 通过 principalUri 过滤用户可访问的地址簿：

**[AddressBookHome.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/AddressBookHome.php#L20-L28)**

```php
public function getChildren(): array
{
    return collect($this->carddavBackend->getAddressBooksForUser($this->getPrincipalUri()))
        ->map(fn (array $addressBookInfo) => new AddressBook($this->carddavBackend, $addressBookInfo))
        ->toArray();
}
```

**[CardDAVBackend.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L78-L83)**

```php
public function getAddressBooksForUser($principalUri): array
{
    return $this->vaults()
        ->map(fn (Vault $vault) => $this->getAddressBookDetails($vault))
        ->toArray();
}
```

### 13.6 基于 Vault 权限的过滤（ACL 第二层） - [GetVaults.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Contact/Dav/Web/Backend/GetVaults.php)

这是 DAV ACL 的**核心判定逻辑**，通过 `vaults()` 方法按用户在 vault 中的权限过滤：

```php
public function vaults(?string $collectionId = null, int $permission = Vault::PERMISSION_VIEW): Collection
{
    $vaults = $this->user->vaults()
        ->wherePivot('permission', '<=', $permission);  // 反向等级：数值越小权限越高

    if ($collectionId !== null) {
        $vaults = $vaults->where('id', $collectionId);
    }

    return $vaults->get();
}
```

#### 各操作使用的权限等级

| 操作 | 调用位置 | 权限等级 | 说明 |
|------|---------|---------|------|
| 列出地址簿 | `getAddressBooksForUser()` | `PERMISSION_VIEW (300)` | 只要在 vault 中就能看到 |
| 读取联系人 | `getObjects()` | `PERMISSION_VIEW (300)` | 查看权限 |
| 创建/更新联系人 | `updateCard()` [L439](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L439) | `PERMISSION_EDIT (200)` | 编辑权限 |
| 访问特定 collection | `getObjectUuid()` [L262](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L262) | `PERMISSION_VIEW (300)` | 无权限则抛出 `NotEnoughPermissionException` |

> **关键**：当 `collectionId` 不为空（访问具体地址簿）但用户无权限时，`vaults()` 返回空集合。`getObjectUuid()` 检测到空集合后会抛出 `NotEnoughPermissionException`。

### 13.7 AddressBook 中的 Principal 绑定 - [CardDAVBackend.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L92)

每个 AddressBook 资源上都标记了其所属的 Principal：

```php
private function getAddressBookDetails(Vault $vault): array
{
    return [
        'id' => $vault->id,
        'uri' => $vault->name,
        'principaluri' => PrincipalBackend::getPrincipalUser($this->user),  // 关键绑定
        '{DAV:}displayname' => $vault->name,
    ];
}
```

Sabre AclPlugin 使用 `principaluri` 判定：只有匹配当前登录 Principal 的请求才能访问该资源。

### 13.8 DAV ACL 完整判定流程图

```
PROPFIND /dav/addressbooks/  (列出地址簿)
  │
  ├─ EnsureDavRequestsAreStateful 中间件
  │   ├─ Session 已登录 → TransientToken (全能)
  │   └─ Basic Auth → 验证 Token → 设置 Auth::user()
  │
  ├─ abilities:read,write 中间件
  │   └─ DAV 要求 Token 同时具备 read 和 write 能力
  │
  ├─ Sabre AuthPlugin
  │   └─ AuthBackend::check() → Auth::check() → 确认已登录
  │
  ├─ DAVServiceProvider::nodes()
  │   └─ 注入当前 $user 到 PrincipalBackend / CardDAVBackend
  │
  ├─ Sabre AclPlugin
  │   ├─ 检查当前 Principal: principals/john@example.com
  │   └─ hideNodesFromListings = true (无权限不显示)
  │
  ├─ AddressBookHome::getChildren()
  │   └─ CardDAVBackend::getAddressBooksForUser(principalUri)
  │       └─ GetVaults::vaults(PERMISSION_VIEW)
  │           └─ $user->vaults()->wherePivot('permission', '<=', 300)
  │               └─ 返回用户有权限查看的所有 vault
  │
  └─ 结果: 只返回用户有 VIEW 以上权限的 vault 对应的地址簿
```

```
PUT /dav/addressbooks/{vaultId}/{contactUuid}.vcf  (更新联系人)
  │
  ├─ ... 同上认证流程 ...
  │
  ├─ CardDAVBackend::updateCard($addressBookId, $cardUri, $cardData)
  │   └─ GetVaults::vaults($addressBookId, PERMISSION_EDIT)
  │       ├─ $user->vaults()
  │       │   ->where('id', $addressBookId)
  │       │   ->wherePivot('permission', '<=', 200)
  │       ├─ 找到 vault → 继续执行，队列 UpdateVCard Job
  │       └─ 未找到 → firstOrFail() → ModelNotFoundException
  │           → Sabre 捕获转为 404
  │
  └─ 结果: 只有 EDIT 以上权限才能修改联系人
```

### 13.9 CalDAV 与 CardDAV 的一致性

CalDAV 后端（日历/任务）使用完全相同的 ACL 机制：
- 同样使用 `WithUser` trait 注入用户
- 同样使用 `GetVaults::vaults()` 过滤 vault
- 同样使用 `PrincipalBackend::getPrincipalUser()` 标记 principaluri

区别仅在于：
- CardDAV 操作 Contact / Group 模型
- CalDAV 操作 Reminder（日历）/ Task（任务）模型

---

## 十四、三阶段权限校验总结（全通道统一视图）

| 阶段 | REST API | DAV 同步 | Web 浏览器 |
|------|----------|---------|-----------|
| **1. 认证层** | `auth:sanctum` 解析 Bearer Token | `EnsureDavRequestsAreStateful` (Basic Auth 或 Session) | `auth:sanctum` (Session Cookie) |
| **2. 粗粒度能力层** | `abilities:read/write` 检查 Token 能力 | `abilities:read,write` 检查 Token 同时具备读写 | `verified` 邮箱验证 |
| **3. 细粒度权限层** | 读: 关联查询 `$user->account->vaults()`<br>写: BaseService 依赖图校验 | `GetVaults::vaults()` 按 pivot permission 过滤<br>+ Sabre AclPlugin principal 检查 | `authorizeResource` → Policy → Gate<br>`can:*` 中间件 → Gate |

所有通道最终都汇聚到同一个权限数据源：**`user_vault` 中间表的 `permission` 字段**。

---

## 十五、Web 路由 can 中间件与 Gate 注册名逐一比对

### 15.1 Gate 定义清单 - [AuthServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Providers/AuthServiceProvider.php#L32-L109)

Monica 共定义了 **8 个 Gate**（不含 Telescope/Pulse 等调试工具 Gate）：

| # | Gate 名称 | 参数 | 判定方式 |
|---|----------|------|---------|
| 1 | `administrator` | `User $user` | `$user->is_account_administrator` |
| 2 | `vault-viewer` | `$user, $vault` | pivot 存在即可 |
| 3 | `vault-editor` | `$user, $vault` | pivot `permission <= 200` |
| 4 | `vault-manager` | `$user, $vault` | pivot `permission <= 100` |
| 5 | `contact-owner` | `$user, $vault, $contact` | `contact.vault_id == vault_id` |
| 6 | `group-owner` | `$user, $vault, $group` | `group.vault_id == vault_id` |
| 7 | `journal-owner` | `$user, $vault, $journal` | `journal.vault_id == vault_id` |
| 8 | `post-owner` | `$user, $journal, $post` | `post.journal_id == journal_id` |
| 9 | `sliceOfLife-owner` | `$user, $journal, $sliceOfLife` | `sliceOfLife.journal_id == journal_id` |

### 15.2 Web 路由 can 中间件挂载清单 - [web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/routes/web.php)

| # | 路由位置 | can 中间件 | 路由参数 | 对应 Gate | 匹配状态 |
|---|---------|-----------|---------|----------|---------|
| 1 | [L199](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/routes/web.php#L199) | `can:vault-viewer,vault` | `{vault}` | `vault-viewer` | ✅ 匹配 |
| 2 | [L250](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/routes/web.php#L250) | `can:contact-owner,vault,contact` | `{vault},{contact}` | `contact-owner` | ✅ 匹配 |
| 3 | [L398](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/routes/web.php#L398) | `can:group-owner,vault,group` | `{vault},{group}` | `group-owner` | ✅ 匹配 |
| 4 | [L413](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/routes/web.php#L413) | `can:journal-owner,vault,journal` | `{vault},{journal}` | `journal-owner` | ✅ 匹配 |
| 5 | [L426](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/routes/web.php#L426) | `can:post-owner,journal,post` | `{journal},{post}` | `post-owner` | ✅ 匹配 |
| 6 | [L452](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/routes/web.php#L452) | `can:slice-owner,journal,slice` | `{journal},{slice}` | `sliceOfLife-owner` | ⚠️ **不匹配！** |
| 7 | [L485](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/routes/web.php#L485) | `can:vault-manager,vault` | `{vault}` | `vault-manager` | ✅ 匹配 |
| 8 | [L580](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/routes/web.php#L580) | `can:administrator` | （无参数） | `administrator` | ✅ 匹配 |

### 15.3 ⚠️ 发现 Bug：`slice-owner` vs `sliceOfLife-owner`

**问题**：路由中使用 `can:slice-owner,journal,slice`，但 Gate 实际注册名为 `sliceOfLife-owner`。

**影响范围**：[web.php L452-L459](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/routes/web.php#L452-L459) 内所有 SliceOfLife 相关路由：

```php
Route::middleware('can:slice-owner,journal,slice')->prefix('slices/{slice}')->group(function () {
    Route::get('', [SliceOfLifeController::class, 'show'])->name('slices.show');
    Route::get('edit', [SliceOfLifeController::class, 'edit'])->name('slices.edit');
    Route::put('', [SliceOfLifeController::class, 'update'])->name('slices.update');
    Route::put('cover', [SliceOfLifeCoverImageController::class, 'update'])->name('slices.cover.update');
    Route::delete('cover', [SliceOfLifeCoverImageController::class, 'destroy'])->name('slices.cover.destroy');
    Route::delete('', [SliceOfLifeController::class, 'destroy'])->name('slices.destroy');
});
```

**后果**：Laravel 无法找到名为 `slice-owner` 的 Gate，会返回 **false**，导致所有已登录用户（即使是 vault 管理者）访问 SliceOfLife 页面时都会被拒绝（403）。

**修复方案**：二选一
- 方案 A（推荐）：将路由改为 `can:sliceOfLife-owner,journal,slice`
- 方案 B：在 AuthServiceProvider 中额外注册一个别名 Gate：`Gate::define('slice-owner', ...)`

### 15.4 控制器内 Gate 调用（非路由级 can 中间件）

除了路由级 can 中间件，部分控制器在方法内部显式调用 `Gate::authorize()`：

| 控制器 | 方法 | Gate 调用 | 行号 |
|--------|------|----------|------|
| [VaultController](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L30) | 构造函数 | `$this->authorizeResource(Vault::class, 'vault')` | L30 |
| [JournalController](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Vault/ManageJournals/Web/Controllers/JournalController.php#L35) | create/store | `Gate::authorize('vault-editor', $vault)` | L35, L45 |
| [GroupController](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Contact/ManageGroups/Web/Controllers/GroupController.php#L49) | create/store/update/destroy | `Gate::authorize('vault-editor', $vaultId)` | L49, L65, L88 |
| [ContactController](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L57) | create/store | `Gate::authorize('vault-editor', $vault)` | L57, L67 |

`authorizeResource` 会自动将 CRUD 方法映射到 Policy 的对应方法，最终委托给 Gate：

| Policy 方法 | 对应 Gate |
|-------------|-----------|
| `viewAny()` | 无（直接返回 true） |
| `view($vault)` | `vault-viewer` |
| `create()` | 无（直接返回 true） |
| `update($vault)` | `vault-editor` |
| `delete($vault)` | `vault-manager` |

---

## 十六、Telegram Webhook 免 CSRF 入口的鉴权链路

### 16.1 CSRF 豁免配置 - [bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/bootstrap/app.php#L21-L25)

```php
$middleware->validateCsrfTokens(except: [
    '/dav',
    '/dav/*',
    '/telegram/webhook/*',   // Telegram webhook 免 CSRF
]);
```

共 3 类路径免于 CSRF 校验：DAV 根路径、DAV 子路径、Telegram webhook。

### 16.2 Telegram Webhook 路由注册 - [web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/routes/web.php#L177-L182)

```php
if (config('services.telegram-bot-api.token') !== null) {
    Route::post(
        '/telegram/webhook/'.config('services.telegram-bot-api.webhook'),
        [TelegramWebhookController::class, 'store']
    );
}
```

关键特征：
- **路由位置**：在 `auth:sanctum` 中间件组**之外**，完全不经过认证中间件
- **环境依赖**：仅当配置了 `services.telegram-bot-api.token` 时才注册
- **URL 保密**：路径包含随机 webhook 密钥 (`config('services.telegram-bot-api.webhook')`)，防止未授权访问

### 16.3 Telegram Webhook 控制器鉴权逻辑 - [TelegramWebhookController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Settings/ManageNotificationChannels/Web/Controllers/TelegramWebhookController.php#L19-L63)

该入口不使用 Laravel Auth，而是采用 **"验证码 + URL 保密"双层校验**：

```
Telegram Bot 发送 POST /telegram/webhook/{secret}
  │
  ├─ 第一层：URL 路径中的 webhook secret（由配置提供）
  │   只有 Telegram Bot 知道这个 URL，攻击者无法猜测
  │
  ├─ 解析消息体中的 $request->message['text']
  │
  ├─ 第二层：消息格式校验
  │   正则: /^\/start\s[A-Za-z0-9-]{36}$/
  │   必须是 "/start " 后跟 36 字符 UUID（verification_token）
  │   不匹配 → 返回 202 Accepted（让 Telegram 停止重试）
  │
  ├─ 第三层：verification_token 查库
  │   UserNotificationChannel::where('verification_token', $key)->firstOrFail()
  │   ├─ 找到 → 绑定 chat_id，激活通知通道
  │   └─ 未找到 → 返回 404 Error
  │
  └─ 成功 → 200 Success，绑定 Telegram Chat ID 到用户通知渠道
```

| 阶段 | 校验方式 | 失败响应 |
|------|---------|---------|
| URL 保密 | webhook secret 路径参数 | 路由不匹配 → 404 |
| 消息格式 | 正则匹配 `/start {36位UUID}` | 202 Accepted（静默丢弃） |
| Token 有效性 | `user_notification_channels.verification_token` 查库 | 404 Error |

> **设计要点**：失败时返回 202 而非 4xx，是为了让 Telegram Bot Platform 停止重试（Telegram 对非 2xx 响应会指数退避重试）。

### 16.4 对比：另一个 Telegram 入口（需认证） - [TelegramNotificationsController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Settings/ManageNotificationChannels/Web/Controllers/TelegramNotificationsController.php#L14-L33)

`POST /settings/notifications/telegram` 是 Web 界面创建通知渠道的入口，位于 `auth:sanctum` 中间件组内：
- 需要正常登录
- 通过 `CreateUserNotificationChannel` Service（走 BaseService 权限校验）
- 生成 `verification_token` 并存储
- 返回给用户用于后续 Telegram Webhook 绑定

---

## 十七、Token 生命周期闭环与撤销/到期判定

Monica 项目中存在 **三种完全独立的 Token 体系**，各自有不同的生命周期管理：

### 17.1 Token 体系概览

| 体系 | 存储表 | 模型 | 用途 | 撤销方式 | 过期策略 |
|------|--------|------|------|---------|---------|
| **Sanctum API Token** | `personal_access_tokens` | Laravel 内置 `PersonalAccessToken` | REST API 认证、DAV Basic Auth | Jetstream 路由删除 | 永不过期（未设置 expires_at） |
| **OAuth 社交登录 Token** | `user_tokens` | [UserToken.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Models/UserToken.php) | Google/GitHub 等 OAuth 登录 | `DELETE /auth/{driver}` | 由 OAuth provider 决定（`expires_in` 字段） |
| **DAV Sync Token** | `sync_tokens` | `SyncToken`（Eloquent 模型） | CardDAV/CalDAV 增量同步 | CleanSyncToken 定时任务 | 保留 30 天后自动清理 |

### 17.2 Sanctum API Token 生命周期

#### 创建

走 Jetstream 标准路由 `POST /user/api-tokens`，在 [JetstreamServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Providers/JetstreamServiceProvider.php#L50-L55) 中配置：
- 默认能力：`['read']`（`Jetstream::defaultApiTokenPermissions`）
- 可选能力：`['read', 'write']`（`Jetstream::permissions`）
- 明文 Token 仅在创建时返回一次，后续无法恢复

数据库存储：
```
personal_access_tokens
  ├─ tokenable_type / tokenable_id → 关联 User
  ├─ name: Token 名称（用户备注）
  ├─ token: SHA-256 哈希值
  ├─ abilities: JSON 数组 ["read","write"]
  ├─ last_used_at: 最后使用时间（每次请求自动更新）
  └─ expires_at: NULL（Monica 不使用过期机制）
```

#### 能力更新

`PUT /user/api-tokens/{tokenId}` - [ApiTokenPermissionsTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/tests/Feature/Auth/ApiTokenPermissionsTest.php#L31-L41)
- 覆盖 `abilities` 字段
- 无效能力名被静默过滤（白名单校验）

#### 撤销（删除）

`DELETE /user/api-tokens/{tokenId}` - [DeleteApiTokenTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/tests/Feature/Auth/DeleteApiTokenTest.php#L31-L33)
- Jetstream 内置逻辑，直接删除对应行
- 删除后该 Token 立即失效（下次请求哈希匹配不到）

#### 到期判定

Monica **从不设置 `expires_at`**，Token 永不过期。Sanctum 的过期检查逻辑（Laravel 内置）：
```php
// Sanctum 伪代码
if ($token->expires_at && $token->expires_at->isPast()) {
    throw new AuthenticationException;  // 401
}
```
由于 `expires_at` 始终为 null，该检查直接跳过。

### 17.3 OAuth 社交登录 Token 生命周期 - [UserToken.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Models/UserToken.php)

存储字段：
```php
protected $fillable = [
    'user_id', 'driver', 'driver_id', 'email', 'format',
    'token', 'token_secret', 'refresh_token', 'expires_in',
];
```

#### 创建

用户通过 Socialite 完成 OAuth 登录回调时，由 [AttemptToAuthenticateSocialite.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Actions/AttemptToAuthenticateSocialite.php) 写入 `user_tokens` 表。

#### 撤销

`DELETE /auth/{driver}` - [UserTokenController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Http/Controllers/Profile/UserTokenController.php#L15-L22)
```php
public function destroy(Request $request, string $driver)
{
    $request->user()->userTokens()
        ->where('driver', $driver)
        ->delete();
    return redirect()->route('profile.show');
}
```
按驱动名（`google`/`github` 等）批量删除该用户下所有对应 Provider 的 Token。

#### 到期判定

`expires_in` 字段存储 OAuth Provider 返回的秒数（如 3600 = 1 小时），但项目中**没有代码检查该字段**，仅作为记录保存。Socialite 驱动内部在调用 API 时会自动用 `refresh_token` 续期。

### 17.4 DAV Sync Token 生命周期 - [CleanSyncToken.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Contact/Dav/Jobs/CleanSyncToken.php)

这是 CardDAV/CalDAV 同步协议的增量同步令牌，与认证无关，用于客户端追踪服务端变更。

#### 创建

每次客户端请求 `sync-token` 时，若数据有变化则自动创建新 Token：
- [SyncDAVBackend.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/app/Domains/Contact/Dav/Web/Backend/SyncDAVBackend.php#L61-L69) `createSyncTokenNow()`
- 每个 Token 关联 `user_id` + `name`（如 `contacts-{vaultId}`）+ `timestamp`

#### 到期判定与清理

保留天数配置：[dav.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/config/dav.php#L24)
```php
'sync_token_keep_days' => 30,
```

清理逻辑（CleanSyncToken Job）：
```
1. 按 user_id + name 分组，找出每组最新的 token（max timestamp）
2. 删除所有"老于最新 token 且老于 30 天"的 token
3. 触发 TokenDeleteEvent 事件
4. 保留每个用户+资源类型至少一个 token（即使超 30 天）
```

> **注意**：Sync Token 不是安全凭证，只是同步游标。它不参与认证或授权判定。

### 17.5 Session Token（Cookie）生命周期

Web 浏览器的 Session Cookie 由 Laravel 默认管理：
- 配置文件：[session.php](file:///d:/fz/0601-2/solo-dogfeeding/code/70-monica/config/session.php#L30-L37)
- `expire_on_close`：默认 `false`，浏览器关闭后 Session 仍有效
- 有效期由 `lifetime` 配置控制（分钟）
- 登出时销毁当前 Session：由 Jetstream/Fortify 标准路由 `POST /logout` 处理

### 17.6 Token 撤销/过期时的 HTTP 响应

| Token 类型 | 撤销/过期事件 | 中间件/检查点 | HTTP 状态码 |
|-----------|-------------|-------------|------------|
| Sanctum API Token | 已删除（DB 无匹配） | `auth:sanctum` | 401 Unauthenticated |
| Sanctum API Token | 已过期（expires_at 过去） | `auth:sanctum` | 401 Unauthenticated |
| Sanctum API Token | 能力不足 | `abilities:*` | 403 Invalid ability |
| DAV Sync Token | 已清理（同步游标丢失） | `SyncDAVBackend::getSyncToken()` | 返回 null → 客户端做全量重新同步 |
| Session Cookie | 已登出 / 过期 | `auth:sanctum` (Session) | 302 重定向到登录页（Web） / 401（JSON 请求） |

---

## 十八、所有入口通道的完整权限矩阵

| 入口路径 | 认证中间件 | CSRF 校验 | 授权方式 | 权限数据源 |
|---------|-----------|----------|---------|-----------|
| `/api/*` | `auth:sanctum` | ✅ API 无 Cookie 不需 CSRF | `abilities:*` + 关联查询/BaseService | `personal_access_tokens.abilities` + `user_vault.permission` |
| `/dav`、`/dav/*` | `EnsureDavRequestsAreStateful` | ❌ 豁免 | `abilities:read,write` + GetVaults + Sabre ACL | 同上 + Sabre principal |
| `/telegram/webhook/*` | **无** | ❌ 豁免 | verification_token 查库 | `user_notification_channels.verification_token` |
| `/user/api-tokens` * | `auth:sanctum` | ✅ | 无额外授权（仅需登录） | 登录状态即可 |
| Web `vaults/{vault}/*` | `auth:sanctum` + `verified` | ✅ | `can:vault-viewer,vault` 等 | `user_vault.permission` |
| Web `settings/*` | `auth:sanctum` + `verified` | ✅ | 部分需 `can:administrator` | `users.is_account_administrator` |
| `GET /currencies` | **无** | ✅ | **无（公开）** | - |
| `/auth/{driver}` (Socialite) | **无** | ✅ | OAuth Provider 验证 | - |
| `GET /invitation/{code}` | **无** | ✅ | invitation code 查库 | `invitations.code` |

> \* `/user/api-tokens` 是 Jetstream 内置路由，用于 Token 的 CRUD 管理，必须已登录但无细粒度权限检查。

