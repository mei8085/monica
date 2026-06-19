# HTTP Resource 与 Domain Action 职责边界分析

> **文档版本**: v1.2
> **仓库根目录**: `d:/fz/0601-2/solo-dogfeeding/code/49-monica/`
> **路径说明**: 本文档使用**仓库相对路径**引用文件，格式为 `[显示名](相对路径#行号范围)`，同时附带完整绝对路径用于点击跳转。

---

## 1. 整体架构分层

项目采用领域驱动设计 (DDD) 的分层架构，核心层次如下：

```
+------------------------------------------+
|        HTTP Resource 层 (表现层)           |
|  UserResource / VaultResource            |
+------------------------------------------+
                    | 数据序列化
                    v
+------------------------------------------+
|        Controller 层 (应用层)              |
|  ApiController / Web Controller          |
+------------------------------------------+
                    | 调用 + 异常转换
                    v
+------------------------------------------+
|     Domain Service/Action 层 (领域层)      |
|  CreateContact / UpdateContact 等        |
+------------------------------------------+
                    | 数据持久化
                    v
+------------------------------------------+
|        Model 层 (数据层)                   |
|  Contact / Vault / User 等               |
+------------------------------------------+
```

---

## 2. HTTP Resource 层

### 2.1 定位与职责

**所在目录**: `app/Http/Resources/`

**核心职责**: 负责将领域模型对象转换为 API 响应格式，属于**表现层**。

### 2.2 具体功能

1. **字段映射与格式化**
   - 选择需要暴露给 API 的字段
   - 对字段值进行格式化 (如日期格式化)
   - 隐藏敏感字段 (未在 Resource 中声明的字段不会暴露)

2. **链接生成 (HATEOAS)**
   - 生成资源的自引用链接
   - 提供资源间的导航能力

3. **数据序列化**
   - 支持单个资源和集合资源
   - 统一 API 输出格式

### 2.3 典型代码示例

以 [VaultResource.php](app/Http/Resources/VaultResource.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Http/Resources/VaultResource.php)) 为例：

```php
class VaultResource extends JsonResource
{
    public function toArray($request): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'description' => $this->description,
            'created_at' => DateHelper::getTimestamp($this->created_at),
            'updated_at' => DateHelper::getTimestamp($this->updated_at),
            'links' => [
                'self' => route('api.vaults.show', $this),
            ],
        ];
    }
}
```

### 2.4 不做的事情

- 不做字段验证
- 不做权限检查
- 不执行业务逻辑
- 不修改数据
- 不抛出业务异常

---

## 3. Domain Service (Action) 层

### 3.1 定位与职责

**所在目录**: `app/Domains/*/Services/`

**核心职责**: 封装领域业务逻辑，属于**领域层**，是业务逻辑的核心载体。

每个 Service 类代表一个领域操作 (Action)，遵循**命令-查询分离**的思想。

### 3.2 具体功能

#### 3.2.1 字段验证 (rules 方法)

定义输入数据的验证规则，包括：
- 字段必填/可选
- 字段类型
- 字段长度限制
- 存在性验证 (exists 规则)
- UUID 格式验证

示例来自 [CreateContact.php](app/Domains/Contact/ManageContact/Services/CreateContact.php#L18-L37) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Services/CreateContact.php#L18-L37))：

```php
public function rules(): array
{
    return [
        'account_id' => 'required|uuid|exists:accounts,id',
        'vault_id' => 'required|uuid|exists:vaults,id',
        'author_id' => 'required|uuid|exists:users,id',
        'first_name' => 'nullable|string|max:255',
        'gender_id' => 'nullable|integer|exists:genders,id',
        'listed' => 'required|boolean',
    ];
}
```

#### 3.2.2 权限验证 (permissions 方法)

定义执行该操作所需的权限，由 [BaseService.php](app/Services/BaseService.php#L80-L83) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Services/BaseService.php#L80-L83)) 统一处理：

```php
public function permissions(): array
{
    return [
        'author_must_belong_to_account',
        'vault_must_belong_to_account',
        'author_must_be_vault_editor',
    ];
}
```

支持的权限类型 (定义在 [BaseService.php](app/Services/BaseService.php#L41-L67) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Services/BaseService.php#L41-L67)))：

| 权限标识 | 含义 | 前置依赖 |
|---------|------|---------|
| `author_must_belong_to_account` | 作者必须属于账户 | 无 |
| `author_must_be_account_administrator` | 作者必须是账户管理员 | author_must_belong_to_account |
| `vault_must_belong_to_account` | Vault 必须属于账户 | 无 |
| `author_must_be_vault_manager` | 作者必须是 Vault 管理员 | vault_must_belong_to_account + author_must_belong_to_account |
| `author_must_be_vault_editor` | 作者必须是 Vault 编辑者 | vault_must_belong_to_account + author_must_belong_to_account |
| `author_must_be_in_vault` | 作者必须在 Vault 中 | vault_must_belong_to_account + author_must_belong_to_account |
| `contact_must_belong_to_vault` | 联系人必须属于 Vault | vault_must_belong_to_account + author_must_belong_to_account |
| `group_must_belong_to_vault` | 分组必须属于 Vault | vault_must_belong_to_account + author_must_belong_to_account |

#### 3.2.3 领域业务逻辑 (execute 方法)

核心业务操作，包含：
- 实体创建/更新/删除
- 关联数据处理 (如创建 FeedItem、更新最后编辑时间)
- 领域规则校验 (如性别必须属于当前账户)
- 默认值处理 (如使用默认模板)

#### 3.2.4 领域一致性验证 (validate 私有方法)

除了基础的 rules 验证外，Service 内部还有额外的领域验证逻辑：

```php
// CreateContact.php#L66-L84
private function validate(): void
{
    $this->validateRules($this->data);

    if ($this->valueOrNull($this->data, 'gender_id')) {
        $this->account()->genders()
            ->findOrFail($this->data['gender_id']);
    }
    // ...
}
```

### 3.3 Service 接口契约

[ServiceInterface.php](app/Interfaces/ServiceInterface.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Interfaces/ServiceInterface.php)) 定义了 Service 的公共接口：

```php
interface ServiceInterface
{
    public function rules(): array;
    public function permissions(): array;
}
```

### 3.4 BaseService 基类

[BaseService.php](app/Services/BaseService.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Services/BaseService.php)) 提供了：

1. **通用属性**: `$author`、`$vault`、`$contact`、`$group` 等常用对象
2. **验证流程**: `validateRules()` 统一调用验证器和权限检查
3. **权限依赖**: `$permissionDependencies` 定义权限间的依赖关系
4. **辅助方法**: `valueOrNull()`、`valueOrTrue()`、`valueOrFalse()` 等工具方法
5. **权限验证方法**:
   - `validateAuthorBelongsToAccount()`
   - `validateVaultExists()`
   - `validateUserPermissionInVault()`
   - `validateContactBelongsToVault()`
   - `validateGroupBelongsToVault()`

### 3.5 QueuableService 子类

[QueuableService.php](app/Services/QueuableService.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Services/QueuableService.php)) 继承 BaseService 并实现 `ShouldQueue`，使 Service 可作为队列任务派发：

- 构造函数中即调用 `validateRules($data)` 进行验证
- `handle()` 方法调用 `execute()`
- `failed()` 方法处理任务失败 (当前为空实现)
- 典型示例: [DestroyContact.php](app/Domains/Contact/ManageContact/Services/DestroyContact.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Services/DestroyContact.php)) 通过 `dispatchSync` 或 `dispatch` 派发

---

## 4. Controller 层

### 4.1 定位与职责

**所在目录**:
- API Controller: `app/Domains/*/Api/Controllers/`
- Web Controller: `app/Domains/*/Web/Controllers/`

**核心职责**: 作为 HTTP 请求的入口，协调各层，属于**应用层**。

### 4.2 API Controller 典型流程

以 [VaultController.php](app/Domains/Vault/ManageVault/Api/Controllers/VaultController.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVault/Api/Controllers/VaultController.php)) 为例：

```php
public function store(Request $request)
{
    // 1. 组装数据: 从 Request 提取 + 补加上下文信息
    $data = [
        'account_id' => $request->user()->account_id,
        'author_id' => $request->user()->id,
        'type' => Vault::TYPE_PERSONAL,
        'name' => $request->input('name'),
        'description' => $request->input('description'),
    ];

    // 2. 调用 Domain Service
    $vault = (new CreateVault)->execute($data);

    // 3. 使用 Resource 转换并返回响应
    return new VaultResource($vault);
}
```

### 4.3 ApiController 基类

[ApiController.php](app/Http/Controllers/ApiController.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Http/Controllers/ApiController.php)) 提供了：

1. **异常捕获与转换**: `callAction()` 方法统一捕获异常并转换为 HTTP 响应：

```php
// ApiController.php#L54-L65
public function callAction($method, $parameters)
{
    try {
        return $this->{$method}(...array_values($parameters));
    } catch (ModelNotFoundException) {
        return $this->respondNotFound();
    } catch (QueryException) {
        return $this->respondInvalidQuery();
    } catch (ValidationException $e) {
        return $this->respondValidatorFailed($e->validator);
    }
}
```

2. **分页参数处理**: `limit` 参数的验证和设置
3. **JSON 响应工具**: 通过 [JsonRespondController.php](app/Traits/JsonRespondController.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Traits/JsonRespondController.php)) trait 提供统一的错误响应格式

### 4.4 Web Controller 特点

Web Controller 与 API Controller 类似，但返回的是 Inertia 视图而不是 JSON：

- 使用 ViewHelper 组装视图数据
- 也会调用 Gate 进行权限检查
- 也会调用 Domain Service 执行业务操作
- **Web Controller 不继承 ApiController**，没有 `callAction()` 的异常捕获逻辑

---

## 5. 字段验证分工

### 5.1 验证层次结构

```
+-----------------------------------------------+
|  1. HTTP 请求层验证 (FormRequest)              |
|     仅认证相关，有限使用 (LoginRequest)        |
|     - 限流验证                                |
|     - 认证逻辑                                |
+-----------------------------------------------+
                    |
                    v
+-----------------------------------------------+
|  2. Domain Service 层验证 (主要验证层)          |
|     - 字段格式验证 (rules 方法)                |
|     - 存在性验证 (exists 规则)                 |
|     - 权限验证 (permissions 方法)              |
|     - 领域一致性验证 (validate 私有方法)       |
+-----------------------------------------------+
                    |
                    v
+-----------------------------------------------+
|  3. 数据库层验证 (最后防线)                     |
|     - 数据库约束 (外键、唯一约束等)            |
+-----------------------------------------------+
```

### 5.2 各层验证详细说明

#### 5.2.1 HTTP 请求层验证 (FormRequest)

**所在位置**: `app/Http/Requests/`

**使用场景**: 仅用于认证相关的特殊场景

**职责**:
- 请求限流 (Rate Limiting)
- 认证逻辑
- 仅限认证失败时的特殊验证

**示例**: [LoginRequest.php](app/Http/Requests/Auth/LoginRequest.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Http/Requests/Auth/LoginRequest.php))

**特点**:
- 仅用于认证相关的有限场景
- 大部分业务验证不在此层

#### 5.2.2 Domain Service 层验证 (主要验证层)

**所在位置**: 每个 Service 类内部

**使用场景**: 所有业务操作的主要验证层

**职责**:

1. **字段格式验证** (rules 方法): 必填/可选、数据类型、长度限制、格式验证
2. **存在性验证** (exists 规则): 外键存在性检查
3. **权限验证** (permissions 方法): 用户身份验证、操作权限检查
4. **领域一致性验证** (validate 私有方法): 领域规则、业务规则、关联数据归属

**特点**:
- 是验证的核心层
- 所有业务操作都必须经过这层验证
- 验证逻辑与业务逻辑紧密结合

#### 5.2.3 数据库层验证 (最后防线)

**所在位置**: 数据库表结构

**职责**: 外键约束、唯一约束、字段类型约束

**特点**:
- 是最后一道防线
- 不应依赖这层验证来保证数据完整性
- 触发的异常会被上层捕获转换

---

## 6. 领域操作分工

### 6.1 领域操作完全在 Domain Service 层

领域操作 (创建、更新、删除等) 完全封装在 Domain Service 中，Controller 不直接操作 Model。

### 6.2 典型领域操作内容

以 [CreateVault.php](app/Domains/Vault/ManageVault/Services/CreateVault.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVault/Services/CreateVault.php)) 为例，一个创建操作包含：

1. **主实体创建** - Vault::create(...)
2. **关联数据创建** - 创建用户联系人 Contact
3. **默认数据填充** - 调用其他 Service 创建默认日期类型、情绪参数、生活事件类别等
4. **副作用操作** - 更新最后编辑时间、创建 Feed 项、触发事件等

### 6.3 领域操作的特点

1. **原子性**: 一个 Service 的 execute 方法代表一个完整的业务操作
2. **封装性**: Controller 不需要知道内部细节
3. **可复用性**: 可以被 Web Controller、API Controller、Job 等调用
4. **可测试性**: 可以独立测试

---

## 7. 异常处理完整流程

### 7.1 Domain Service 抛出的异常全景

通过代码扫描，Domain Service 层抛出以下 6 种领域异常：

| 异常类 | 抛出位置 | 业务含义 |
|-------|---------|---------|
| `ValidationException` | BaseService::validateRules() 内的 Validator::make()->validate() | 字段验证失败 |
| `ModelNotFoundException` | BaseService 内各 findOrFail() 调用 | 关联记录不存在 |
| `NotEnoughPermissionException` | [BaseService.php#L173](app/Services/BaseService.php#L173) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Services/BaseService.php#L173)) (2处)、[MoveContactToAnotherVault.php#L73](app/Domains/Contact/ManageContact/Services/MoveContactToAnotherVault.php#L73) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Services/MoveContactToAnotherVault.php#L73))、[CopyContactToAnotherVault.php#L73](app/Domains/Contact/ManageContact/Services/CopyContactToAnotherVault.php#L73) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Services/CopyContactToAnotherVault.php#L73))、[CardDAVBackend.php#L263](app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L263) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L263)) | 权限不足 |
| `CantBeDeletedException` | [DestroyContact.php#L45](app/Domains/Contact/ManageContact/Services/DestroyContact.php#L45) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Services/DestroyContact.php#L45))、[DestroyLifeEventCategory.php#L47](app/Domains/Vault/ManageVaultSettings/Services/DestroyLifeEventCategory.php#L47) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVaultSettings/Services/DestroyLifeEventCategory.php#L47)) 等 | 资源不可删除 |
| `SameUserException` | [GrantVaultAccessToUser.php#L68](app/Domains/Vault/ManageVaultSettings/Services/GrantVaultAccessToUser.php#L68) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVaultSettings/Services/GrantVaultAccessToUser.php#L68))、[RemoveVaultAccess.php#L64](app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php#L64) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php#L64))、[ChangeVaultAccess.php#L65](app/Domains/Vault/ManageVaultSettings/Services/ChangeVaultAccess.php#L65) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVaultSettings/Services/ChangeVaultAccess.php#L65)) | 操作目标与操作者相同 |
| `EnvVariablesNotSetException` | [GetGPSCoordinate.php#L67](app/Domains/Vault/ManageAddresses/Services/GetGPSCoordinate.php#L67) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageAddresses/Services/GetGPSCoordinate.php#L67))、[UploadFile.php#L66](app/Domains/Contact/ManageDocuments/Services/UploadFile.php#L66) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageDocuments/Services/UploadFile.php#L66)) 等 | 环境变量未配置 |
| `MaximumNumberOfUsersInVaultException` | (定义于 [MaximumNumberOfUsersInVaultException.php](app/Exceptions/MaximumNumberOfUsersInVaultException.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Exceptions/MaximumNumberOfUsersInVaultException.php))) | Vault 用户数超限 |
| `EntryAlreadyExistException` | (定义于 [EntryAlreadyExistException.php](app/Exceptions/EntryAlreadyExistException.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Exceptions/EntryAlreadyExistException.php))) | 条目已存在 |

此外，BaseService 自身还可能抛出通用 `\Exception`：

| 抛出位置 | 代码 | 含义 |
|---------|------|------|
| [BaseService.php#L106](app/Services/BaseService.php#L106) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Services/BaseService.php#L106)) | `throw new \Exception("$key requires $value")` | 权限配置错误: 缺少前置权限 |
| [BaseService.php#L115](app/Services/BaseService.php#L115) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Services/BaseService.php#L115)) | `throw new \Exception('Unknown permission: '.$e->first())` | 权限配置错误: 未知权限标识 |
| [BaseService.php#L152](app/Services/BaseService.php#L152) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Services/BaseService.php#L152)) | `throw new \Exception("Unknown permission: $permission")` | switch-case 兜底: 不应到达的分支 |

> 上述三个 `\Exception` 属于**开发期防御性异常**，正常情况下不应触发，表示 Service 的 permissions() 配置有误。

### 7.2 异常流转的两条路径

异常从 Domain Service 抛出后，根据调用环境不同，走完全不同的处理路径：

```
                          Domain Service 抛出异常
                                    |
                     +--------------+--------------+
                     |                             |
                     v                             v
            API 请求环境                    Web / 其他环境
            (ApiController)             (WebController / Job / CLI)
                     |                             |
                     v                             v
          ApiController::callAction()        无 catch 块
          部分异常被捕获                       |
                     |                        v
          +----------+----------+      Laravel 全局 Handler
          |          |          |       (bootstrap/app.php 配置)
          v          v          v            |
    ModelNot    Validation   Query          v
    Found       Exception   Exception   Sentry 报告 +
      |           |           |         Laravel 默认渲染
      v           v           v            |
    404         422         500        +----+----+
    JSON        JSON        JSON      |         |
                                      v         v
                              Web 环境      API 环境
                              HTML 错误页    JSON 错误
```

### 7.3 路径 A: API 请求 -- ApiController::callAction() 的处理

[ApiController.php#L54-L65](app/Http/Controllers/ApiController.php#L54-L65) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Http/Controllers/ApiController.php#L54-L65)) 的 `callAction()` 只 catch 了 **3 种**异常：

```php
public function callAction($method, $parameters)
{
    try {
        return $this->{$method}(...array_values($parameters));
    } catch (ModelNotFoundException) {
        return $this->respondNotFound();              // 404, error_code=31
    } catch (QueryException) {
        return $this->respondInvalidQuery();           // 500, error_code=40
    } catch (ValidationException $e) {
        return $this->respondValidatorFailed($e->validator);  // 422, error_code=32
    }
}
```

**被 ApiController 显式捕获的异常**：

| 异常类型 | HTTP 状态码 | 错误码 | 响应方法 | 响应格式 |
|---------|------------|--------|---------|---------|
| `ModelNotFoundException` | 404 | 31 | `respondNotFound()` | `{"error":{"message":"...","error_code":31}}` |
| `ValidationException` | 422 | 32 | `respondValidatorFailed()` | `{"error":{"message":["字段1: 错误信息"],"error_code":32}}` |
| `QueryException` | 500 | 40 | `respondInvalidQuery()` | `{"error":{"message":"...","error_code":40}}` |

**未被 ApiController 显式捕获的异常** (会继续冒泡到 Laravel 全局 Handler)：

| 异常类型 | 最终 HTTP 响应 | 处理者 |
|---------|--------------|--------|
| `NotEnoughPermissionException` | 500 (Laravel 默认) | Laravel 全局 Handler |
| `CantBeDeletedException` | 500 (Laravel 默认) | Laravel 全局 Handler |
| `SameUserException` | 500 (Laravel 默认) | Laravel 全局 Handler |
| `EnvVariablesNotSetException` | 500 (Laravel 默认) | Laravel 全局 Handler |
| `MaximumNumberOfUsersInVaultException` | 500 (Laravel 默认) | Laravel 全局 Handler |
| `EntryAlreadyExistException` | 500 (Laravel 默认) | Laravel 全局 Handler |
| 通用 `\Exception` | 500 (Laravel 默认) | Laravel 全局 Handler |

> **关键发现**: `NotEnoughPermissionException`、`CantBeDeletedException` 等领域异常在 `ApiController::callAction()` 中**没有对应的 catch 分支**。它们会越过 ApiController，直接冒泡到 Laravel 全局异常处理器。在 API 请求中 (Accept: application/json)，Laravel 默认返回 500 JSON 响应，而非语义化的 403 或 422。

### 7.4 路径 B: Web 请求 -- Web Controller 的处理

Web Controller 继承自 [Controller.php](app/Http/Controllers/Controller.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Http/Controllers/Controller.php))，使用 `AuthorizesRequests` trait：

```php
abstract class Controller extends BaseController
{
    use AuthorizesRequests;
    use DispatchesJobs;
    use ValidatesRequests;
}
```

Web Controller 的异常处理流程：

1. **无 callAction 重写**: Web Controller 不像 ApiController 那样重写 `callAction()`，没有显式的 try-catch
2. **Gate::authorize()**: 部分 Web Controller 方法在调用 Service 前先通过 Gate 检查权限，这会抛出 `AuthorizationException` (Laravel 框架级异常)
3. **所有异常冒泡**: 无论是 Service 抛出的领域异常还是 Gate 抛出的 `AuthorizationException`，都直接冒泡到 Laravel 全局 Handler

以 [ContactController.php](app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php)) 为例：

```php
public function store(Request $request, string $vaultId)
{
    Gate::authorize('vault-editor', $vaultId);  // 可能抛 AuthorizationException

    $data = [...];
    $contact = (new CreateContact)->execute($data);  // 可能抛领域异常

    return response()->json([...], 201);
}
```

Web 请求中 Laravel 全局 Handler 的默认行为：

| 异常类型 | 默认 HTTP 响应 | 说明 |
|---------|--------------|------|
| `ValidationException` | 302 重回表单页 + session 错误 | Laravel 自动处理 |
| `AuthorizationException` | 403 Forbidden 页面 | Laravel 自动处理 |
| `ModelNotFoundException` | 404 Not Found 页面 | Laravel 自动处理 |
| `NotEnoughPermissionException` | 500 Internal Server Error 页面 | **无特殊处理，按通用异常对待** |
| `CantBeDeletedException` | 500 Internal Server Error 页面 | **无特殊处理，按通用异常对待** |
| 其他领域异常 | 500 Internal Server Error 页面 | **无特殊处理，按通用异常对待** |

### 7.5 路径 C: 队列任务 -- QueuableService 的处理

通过 `dispatchSync` 或 `dispatch` 派发的 QueuableService，异常处理遵循 Laravel 队列机制：

1. `dispatchSync` (同步): 异常直接冒泡到调用方 (Controller)，走调用方的异常处理路径
2. `dispatch` (异步队列): 异常由 Laravel 队列 worker 捕获，触发任务重试或标记失败，调用 `failed()` 方法

以 [ContactController.php#L167-L183](app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L167-L183) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L167-L183)) 为例：

```php
public function destroy(Request $request, string $vaultId, string $contactId)
{
    $data = [
        'account_id' => Auth::user()->account_id,
        'author_id' => Auth::id(),
        'vault_id' => $vaultId,
        'contact_id' => $contactId,
    ];

    DestroyContact::dispatchSync($data);  // 同步派发，异常直接冒泡

    return response()->json([...], 200);
}
```

如果 `DestroyContact` 在 `validateRules()` 阶段抛出 `NotEnoughPermissionException`，或 `execute()` 阶段抛出 `CantBeDeletedException`，该异常会直接冒泡到 Controller，然后冒泡到全局 Handler。

### 7.6 Laravel 全局异常处理器

**配置位置**: [bootstrap/app.php#L45-L47](bootstrap/app.php#L45-L47) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/bootstrap/app.php#L45-L47))

```php
->withExceptions(function (Exceptions $exceptions) {
    Integration::handles($exceptions);  // Sentry 集成
})
```

**自定义 Handler**: [Handler.php](app/Exceptions/Handler.php) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Exceptions/Handler.php))

```php
class Handler extends ExceptionHandler
{
    public function register()
    {
        $this->reportable(function (Throwable $e) {
            if (app()->bound('sentry')) {
                app('sentry')->captureException($e);
            }
        });
    }
}
```

**关键事实**: Handler 只注册了 `reportable` 回调 (Sentry 报告)，**没有注册任何 `renderable` 回调**来将领域异常转换为语义化的 HTTP 响应。

这意味着 Laravel 框架默认的异常渲染行为生效：

- 对于 API 请求 (Accept: application/json): Laravel 将未捕获的异常统一渲染为 500 JSON 响应
- 对于 Web 请求: Laravel 将未捕获的异常统一渲染为 500 错误页面 (或 debug 页面)
- `AuthorizationException` 等框架级异常有 Laravel 内置的语义化处理 (403)
- **自定义领域异常全部被当作 500 处理**

### 7.7 领域内部捕获: 两条 DAV 导入路径中的静默吞没

在 2 处 DAV 导入代码中，`NotEnoughPermissionException` 被领域内部 catch 并静默忽略：

1. [ImportContactInformation.php#L104](app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php#L104) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php#L104)):

```php
} catch (NotEnoughPermissionException) {
    continue;  // 跳过无权限的联系人信息，继续处理下一条
}
```

2. [ImportAddress.php#L82](app/Domains/Contact/ManageContact/Dav/ImportAddress.php#L82) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Dav/ImportAddress.php#L82)):

```php
} catch (NotEnoughPermissionException) {
    // catch -- 静默忽略，地址类型创建失败时跳过
}
```

这是项目中仅有的两处在 Domain Service 内部 catch 领域异常的地方。设计意图是在批量导入场景中，单条记录的权限不足不应阻断整个导入流程。

---

## 8. 异常处理流程序列图

### 8.1 场景 1: API 请求 + ValidationException (被 ApiController 显式捕获)

```
API 请求  POST /api/vaults  (name 为空)
  |
  v
VaultController::store()
  |-- 组装 $data
  |-- (new CreateVault)->execute($data)
  |     |
  |     v
  |   CreateVault::validateRules($data)
  |     |-- Validator::make($data, $this->rules())->validate()
  |     |     |-- name 字段为空，违反 required 规则
  |     |     v
  |     |   抛出 ValidationException
  |     |
  |     v (异常冒泡)
  |   ApiController::callAction() 捕获
  |     |-- catch (ValidationException $e)
  |     |     |-- $this->respondValidatorFailed($e->validator)
  |     |     |     |-- 设置 HTTP 422, error_code=32
  |     |     |     v
  |     |     |   返回 JSON:
  |     |     |   {"error":{"message":["name is required"],"error_code":32}}
  |     v
  |   响应返回
  v
HTTP 422 Unprocessable Entity
```

### 8.2 场景 2: API 请求 + NotEnoughPermissionException (未被 ApiController 捕获)

```
API 请求  POST /api/vaults  (用户权限不足)
  |
  v
VaultController::store()
  |-- 组装 $data
  |-- (new CreateVault)->execute($data)
  |     |
  |     v
  |   CreateVault::validateRules($data)
  |     |-- permissions() 返回 ['author_must_belong_to_account']
  |     |-- validatePermission('author_must_belong_to_account', $data)
  |     |     |-- validateAuthorBelongsToAccount($data)
  |     |     |     |-- User::where(...)->findOrFail(...)  -- 找到用户
  |     |     v
  |     |   (假设进一步检查 author_must_be_vault_manager)
  |     |     |-- validateUserPermissionInVault(Vault::PERMISSION_MANAGE)
  |     |     |     |-- 用户不是 vault manager
  |     |     |     v
  |     |     |   抛出 NotEnoughPermissionException
  |     |
  |     v (异常冒泡)
  |   ApiController::callAction()
  |     |-- try { ... } catch (ModelNotFoundException) {...}     -- 不匹配
  |     |-- catch (QueryException) {...}                         -- 不匹配
  |     |-- catch (ValidationException $e) {...}                -- 不匹配
  |     |-- 无匹配的 catch 分支，异常继续冒泡
  |     v
  |   Laravel 全局 ExceptionHandler
  |     |-- report: 发送到 Sentry
  |     |-- render: API 请求 (Accept: application/json)
  |     |     |-- 按通用 Exception 处理
  |     |     v
  |     |   HTTP 500 Internal Server Error
  |     |   {"message":"Internal Server Error"}  (或 debug 详情)
  v
HTTP 500 Internal Server Error
```

### 8.3 场景 3: Web 请求 + CantBeDeletedException (队列同步派发)

```
Web 请求  DELETE /vault/{vaultId}/contact/{contactId}
  |
  v
ContactController::destroy()
  |-- 组装 $data
  |-- DestroyContact::dispatchSync($data)
  |     |
  |     v
  |   QueuableService 构造函数
  |     |-- $this->validateRules($data)
  |     |     |-- 权限检查通过
  |     v
  |   DestroyContact::handle() -> execute($data)
  |     |-- $this->contact->can_be_deleted === false
  |     |     v
  |     |   抛出 CantBeDeletedException
  |     |
  |     v (异常冒泡，dispatchSync 不吃异常)
  |   Controller 层无 catch
  |     v
  |   Laravel 全局 ExceptionHandler
  |     |-- report: 发送到 Sentry
  |     |-- render: Web 请求
  |     |     |-- 按通用 Exception 处理
  |     |     v
  |     |   HTTP 500 Internal Server Error (HTML 错误页)
  v
HTTP 500 错误页面
```

### 8.4 场景 4: Web 请求 + Gate::authorize (AuthorizationException，框架级处理)

```
Web 请求  POST /vault/{vaultId}/contact
  |
  v
ContactController::store()
  |-- Gate::authorize('vault-editor', $vaultId)
  |     |
  |     v
  |   VaultPolicy::create() / 检查用户权限
  |     |-- 用户无编辑权限
  |     |     v
  |     |   抛出 AuthorizationException  (Laravel 框架异常)
  |     |
  |     v (异常冒泡，Web Controller 无 catch)
  |   Laravel 全局 ExceptionHandler
  |     |-- render: AuthorizationException
  |     |     |-- Laravel 内置处理: 403 Forbidden
  |     |     v
  |     |   HTTP 403 Forbidden (HTML 错误页)
  v
HTTP 403 Forbidden 错误页面
```

> 注意: `AuthorizationException` (来自 Gate) 和 `NotEnoughPermissionException` (来自 Service) 虽然业务含义类似，但前者被 Laravel 框架自动映射为 403，后者却被当作 500。

---

## 9. 异常转换汇总表

### 9.1 API 环境 (Accept: application/json)

| 异常类型 | ApiController catch? | 最终 HTTP 状态码 | 处理者 | 响应格式 |
|---------|---------------------|-----------------|--------|---------|
| `ValidationException` | 是 | 422 | ApiController | `{"error":{"message":[...],"error_code":32}}` |
| `ModelNotFoundException` | 是 | 404 | ApiController | `{"error":{"message":"...","error_code":31}}` |
| `QueryException` | 是 | 500 | ApiController | `{"error":{"message":"...","error_code":40}}` |
| `NotEnoughPermissionException` | **否** | **500** | Laravel 默认 | `{"message":"Internal Server Error"}` |
| `CantBeDeletedException` | **否** | **500** | Laravel 默认 | `{"message":"Internal Server Error"}` |
| `SameUserException` | **否** | **500** | Laravel 默认 | `{"message":"Internal Server Error"}` |
| `EnvVariablesNotSetException` | **否** | **500** | Laravel 默认 | `{"message":"Internal Server Error"}` |
| `MaximumNumberOfUsersInVaultException` | **否** | **500** | Laravel 默认 | `{"message":"Internal Server Error"}` |
| `EntryAlreadyExistException` | **否** | **500** | Laravel 默认 | `{"message":"Internal Server Error"}` |
| 通用 `\Exception` | **否** | **500** | Laravel 默认 | `{"message":"Internal Server Error"}` |

### 9.2 Web 环境 (Accept: text/html)

| 异常类型 | Web Controller catch? | 最终 HTTP 状态码 | 处理者 |
|---------|----------------------|-----------------|--------|
| `ValidationException` | 否 | 302 | Laravel 默认 (重回表单+错误) |
| `AuthorizationException` | 否 | 403 | Laravel 默认 |
| `ModelNotFoundException` | 否 | 404 | Laravel 默认 |
| `NotEnoughPermissionException` | **否** | **500** | Laravel 默认 |
| `CantBeDeletedException` | **否** | **500** | Laravel 默认 |
| 其他领域异常 | **否** | **500** | Laravel 默认 |

---

## 10. 权限检查的双重路径

项目中存在**两套独立的权限检查机制**，分别在不同层级工作：

### 10.1 路径 1: Controller 层的 Gate 检查 (Web Controller 专属)

- 调用方式: `Gate::authorize('vault-editor', $vaultId)`
- 抛出异常: `AuthorizationException` (Laravel 框架级)
- 处理方式: Laravel 自动映射为 403 Forbidden
- 适用范围: **仅 Web Controller**，API Controller 不使用

示例: [ContactController.php#L57](app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L57) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L57))

```php
Gate::authorize('vault-editor', $vault);
```

### 10.2 路径 2: Domain Service 层的 permissions 检查 (通用)

- 调用方式: Service 的 `permissions()` + BaseService 的 `validateRules()`
- 抛出异常: `NotEnoughPermissionException` (自定义领域异常)
- 处理方式: **无显式映射**，默认 500
- 适用范围: **所有调用方式** (Web、API、Job、CLI)

示例: [CreateContact.php#L42-L49](app/Domains/Contact/ManageContact/Services/CreateContact.php#L42-L49) ([绝对路径](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Services/CreateContact.php#L42-L49))

```php
public function permissions(): array
{
    return [
        'author_must_belong_to_account',
        'vault_must_belong_to_account',
        'author_must_be_vault_editor',
    ];
}
```

### 10.3 两套机制的差异

| 对比维度 | Gate 检查 | Service permissions |
|---------|----------|-------------------|
| 所在层 | Controller (应用层) | Domain Service (领域层) |
| 抛出异常 | `AuthorizationException` | `NotEnoughPermissionException` |
| HTTP 映射 | 403 (自动) | 500 (未映射) |
| 复用性 | 仅 Web Controller | 所有调用方式 |
| 检查时机 | Service 调用之前 | Service 内部 |
| 检查粒度 | 粗 (资源级别) | 细 (字段级别+上下文) |

> Web Controller 中存在 Gate 和 Service permissions **双重检查**的情况。例如 `ContactController::store()` 先调用 `Gate::authorize()`，然后 `(new CreateContact)->execute()` 内部又执行 permissions 检查。Gate 检查可视为**前置快速失败**，Service permissions 则是**权威的领域级权限守卫**。

---

## 11. 三层职责总结对比

| 职责 | HTTP Resource | Domain Service | Controller |
|-----|--------------|---------------|------------|
| 字段验证 | - | 主要验证层 | - |
| 权限检查 | - | 主要层 | Gate 辅助 (仅 Web) |
| 业务逻辑 | - | 核心层 | - |
| 数据格式化 | 主要层 | - | - |
| 数据组装 | - | - | 主要层 |
| 异常抛出 | - | 领域异常 | - |
| 异常转换 | - | - | 部分 (仅 API 的 3 种) |
| HTTP 响应 | 数据部分 | - | 完整响应 |
| 路由链接 | HATEOAS | - | - |
| 可复用性 | 低 | 高 | 中 |

---

## 12. 设计原则与优点

### 12.1 单一职责原则

- Resource 只负责数据展示
- Service 只负责业务逻辑
- Controller 只负责请求协调

### 12.2 依赖倒置原则

- 上层依赖下层
- 下层不依赖上层
- Service 不知道 Resource 和 Controller

### 12.3 可测试性

- Service 可以独立测试
- 不依赖 HTTP 环境
- 可以被多种调用方式 (Web、API、Job、CLI)

### 12.4 可维护性

- 业务逻辑集中在 Service 层
- 修改业务只需要修改 Service
- 表现层变化不影响业务逻辑

---

## 13. 常见疑问点澄清

### 13.1 为什么验证不在 Controller 层?

- 业务验证与业务逻辑紧密相关
- Service 可以被多种调用方式复用 (Web、API、Job)
- 保证所有入口都经过同样的验证

### 13.2 为什么有 FormRequest 但很少使用?

- FormRequest 是 Laravel 的推荐方式
- 但项目选择了 Service 层验证
- 这是为了更好的复用性和领域驱动设计的一致性

### 13.3 Domain Service 和 Domain Action 是同一个概念吗?

- Domain Service = Domain Action
- 项目中叫 Service
- 本质上是 Action (领域操作)
- 遵循命令模式的概念

### 13.4 异常为什么不在 Service 层处理?

- Service 抛出领域异常
- Controller 转换为 HTTP 响应 (部分)
- 保持领域层与 HTTP 解耦
- Service 可以被非 HTTP 场景使用

### 13.5 为什么 NotEnoughPermissionException 不被 ApiController 捕获?

- `ApiController::callAction()` 只显式捕获了 3 种 Laravel 框架级异常
- 自定义领域异常 (`NotEnoughPermissionException`、`CantBeDeletedException` 等) 没有对应的 catch 分支
- 这些异常会冒泡到 Laravel 全局 Handler，被当作 500 处理
- 这可能是一个**设计缺陷**或**待完善项**，合理的做法应该是在 ApiController 或全局 Handler 中添加对这些领域异常的显式映射 (如 NotEnoughPermissionException -> 403)

---

## 14. 文件引用索引

以下是本文档中引用的所有文件的完整列表，包含仓库相对路径和绝对路径，方便在不同环境中定位：

| 编号 | 文件描述 | 仓库相对路径 | 绝对路径 |
|-----|---------|-------------|---------|
| 1 | Vault 资源类 | `app/Http/Resources/VaultResource.php` | [VaultResource.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Http/Resources/VaultResource.php) |
| 2 | 创建联系人 Service | `app/Domains/Contact/ManageContact/Services/CreateContact.php` | [CreateContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Services/CreateContact.php) |
| 3 | Service 基类 | `app/Services/BaseService.php` | [BaseService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Services/BaseService.php) |
| 4 | Service 接口 | `app/Interfaces/ServiceInterface.php` | [ServiceInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Interfaces/ServiceInterface.php) |
| 5 | 可队列 Service | `app/Services/QueuableService.php` | [QueuableService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Services/QueuableService.php) |
| 6 | 删除联系人 Service | `app/Domains/Contact/ManageContact/Services/DestroyContact.php` | [DestroyContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Services/DestroyContact.php) |
| 7 | Vault API 控制器 | `app/Domains/Vault/ManageVault/Api/Controllers/VaultController.php` | [VaultController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVault/Api/Controllers/VaultController.php) |
| 8 | API 控制器基类 | `app/Http/Controllers/ApiController.php` | [ApiController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Http/Controllers/ApiController.php) |
| 9 | JSON 响应 Trait | `app/Traits/JsonRespondController.php` | [JsonRespondController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Traits/JsonRespondController.php) |
| 10 | 登录请求验证 | `app/Http/Requests/Auth/LoginRequest.php` | [LoginRequest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Http/Requests/Auth/LoginRequest.php) |
| 11 | 创建 Vault Service | `app/Domains/Vault/ManageVault/Services/CreateVault.php` | [CreateVault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVault/Services/CreateVault.php) |
| 12 | 权限不足异常 | `app/Exceptions/NotEnoughPermissionException.php` | [NotEnoughPermissionException.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Exceptions/NotEnoughPermissionException.php) |
| 13 | 移动联系人到其他 Vault | `app/Domains/Contact/ManageContact/Services/MoveContactToAnotherVault.php` | [MoveContactToAnotherVault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Services/MoveContactToAnotherVault.php) |
| 14 | 复制联系人到其他 Vault | `app/Domains/Contact/ManageContact/Services/CopyContactToAnotherVault.php` | [CopyContactToAnotherVault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Services/CopyContactToAnotherVault.php) |
| 15 | CardDAV 后端 | `app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php` | [CardDAVBackend.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php) |
| 16 | 删除生活事件类别 | `app/Domains/Vault/ManageVaultSettings/Services/DestroyLifeEventCategory.php` | [DestroyLifeEventCategory.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVaultSettings/Services/DestroyLifeEventCategory.php) |
| 17 | 授予 Vault 访问权限 | `app/Domains/Vault/ManageVaultSettings/Services/GrantVaultAccessToUser.php` | [GrantVaultAccessToUser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVaultSettings/Services/GrantVaultAccessToUser.php) |
| 18 | 移除 Vault 访问权限 | `app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php` | [RemoveVaultAccess.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php) |
| 19 | 修改 Vault 访问权限 | `app/Domains/Vault/ManageVaultSettings/Services/ChangeVaultAccess.php` | [ChangeVaultAccess.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVaultSettings/Services/ChangeVaultAccess.php) |
| 20 | 获取 GPS 坐标 Service | `app/Domains/Vault/ManageAddresses/Services/GetGPSCoordinate.php` | [GetGPSCoordinate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageAddresses/Services/GetGPSCoordinate.php) |
| 21 | 上传文件 Service | `app/Domains/Contact/ManageDocuments/Services/UploadFile.php` | [UploadFile.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageDocuments/Services/UploadFile.php) |
| 22 | Vault 用户数超限异常 | `app/Exceptions/MaximumNumberOfUsersInVaultException.php` | [MaximumNumberOfUsersInVaultException.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Exceptions/MaximumNumberOfUsersInVaultException.php) |
| 23 | 条目已存在异常 | `app/Exceptions/EntryAlreadyExistException.php` | [EntryAlreadyExistException.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Exceptions/EntryAlreadyExistException.php) |
| 24 | Web 控制器基类 | `app/Http/Controllers/Controller.php` | [Controller.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Http/Controllers/Controller.php) |
| 25 | 联系人 Web 控制器 | `app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php` | [ContactController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php) |
| 26 | 应用启动配置 | `bootstrap/app.php` | [bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/bootstrap/app.php) |
| 27 | 全局异常处理器 | `app/Exceptions/Handler.php` | [Handler.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Exceptions/Handler.php) |
| 28 | 导入联系人信息 DAV | `app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php` | [ImportContactInformation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php) |
| 29 | 导入地址 DAV | `app/Domains/Contact/ManageContact/Dav/ImportAddress.php` | [ImportAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Dav/ImportAddress.php) |
| 30 | Vault 策略类 | `app/Policies/VaultPolicy.php` | [VaultPolicy.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Policies/VaultPolicy.php) |
