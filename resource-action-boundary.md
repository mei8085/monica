# HTTP Resource 与 Domain Action 职责边界分析

## 1. 整体架构分层

项目采用领域驱动设计（DDD）的分层架构，核心层次如下：

```
┌─────────────────────────────────────────┐
│          HTTP Resource 层 (表现层)        │
│  UserResource / VaultResource      │
└─────────────────────────────────────────┘
                      ↓ 数据序列化
┌─────────────────────────────────────────┐
│          Controller 层 (应用层)     │
│  ApiController / Web Controller      │
└─────────────────────────────────────────┘
                      ↓ 调用 + 异常转换
┌─────────────────────────────────────────┐
│     Domain Service/Action 层 (领域层)    │
│  CreateContact / UpdateContact 等     │
└─────────────────────────────────────────┘
                      ↓ 数据持久化
┌─────────────────────────────────────────┐
│          Model 层 (数据层)               │
│  Contact / Vault / User 等         │
└─────────────────────────────────────────┘
```

---

## 2. HTTP Resource 层

### 2.1 定位与职责

**所在目录：[app/Http/Resources/

**核心职责**：负责将领域模型对象转换为 API 响应格式，属于**表现层**。

### 2.2 具体功能

1. **字段映射与格式化**
   - 选择需要暴露给 API 的字段
   - 对字段值进行格式化（如日期格式化）
   - 隐藏敏感字段（不会在 Resource 中不包含的字段不会暴露）

2. **链接生成（HATEOAS）
   - 生成资源的自引用链接
   - 提供资源间的导航能力

3. **数据序列化**
   - 支持单个资源和集合资源
   - 统一 API 输出格式

### 2.3 典型代码示例

以 [VaultResource.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Http/Resources/VaultResource.php)：

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

- ❌ 不做字段验证
- ❌ 不做权限检查
- ❌ 不执行业务逻辑
- ❌ 不修改数据
- ❌ 不抛出业务异常

---

## 3. Domain Service (Action) 层

### 3.1 定位与职责

**所在目录**：[app/Domains/*/Services/

**核心职责**：封装领域业务逻辑，属于**领域层**，是业务逻辑的核心载体。

每个 Service 类代表一个领域操作（Action），遵循**命令-查询分离**的思想。

### 3.2 具体功能

#### 3.2.1 字段验证（rules 方法）

定义输入数据的验证规则，包括：
- 字段必填/字段类型
- 字段长度限制
- 存在性验证（exists 规则）
- UUID 格式验证

示例来自 [CreateContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Services/CreateContact.php#L18-L37)：

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

#### 3.2.2 权限验证（permissions 方法）

定义执行该操作所需的权限，由 [BaseService](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Services/BaseService.php#L80-L83) 统一处理：

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

支持的权限类型：
- `author_must_belong_to_account - 作者必须属于账户
- `author_must_be_account_administrator - 作者必须是账户管理员
- `vault_must_belong_to_account - Vault 必须属于账户
- `author_must_be_vault_manager - 作者必须是 Vault 管理员
- `author_must_be_vault_editor - 作者必须是 Vault 编辑者
- `author_must_be_in_vault - 作者必须在 Vault 中
- `contact_must_belong_to_vault - 联系人必须属于 Vault
- `group_must_belong_to_vault - 分组必须属于 Vault

#### 3.2.3 领域业务逻辑（execute 方法）

核心业务操作，包含：
- 实体创建/更新/删除
- 关联数据处理（如创建 FeedItem、更新最后编辑时间）
- 领域规则校验（如性别必须属于当前账户）
- 默认值处理（如使用默认模板）

#### 3.2.4 领域一致性验证（validate 私有方法）

除了基础的 rules 验证外，Service 内部还有额外的领域验证逻辑：

```php
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

[ServiceInterface.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Interfaces/ServiceInterface.php) 定义了 Service 的公共接口：

```php
interface ServiceInterface
{
    public function rules(): array;
    public function permissions(): array;
}
```

### 3.4 BaseService 基类

[BaseService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Services/BaseService.php) 提供了：

1. **通用属性**：`$author`、`$vault`、`$contact`、`$group` 等常用对象

2. **验证流程**：`validateRules()` 统一调用验证器和权限检查

3. **权限依赖**：`$permissionDependencies` 定义权限间的依赖关系

4. **辅助方法**：`valueOrNull()`、`valueOrTrue()`、`valueOrFalse()` 等工具方法

5. **权限验证方法**：
   - `validateAuthorBelongsToAccount()`
   - `validateVaultExists()`
   - `validateUserPermissionInVault()`
   - `validateContactBelongsToVault()`
   - `validateGroupBelongsToVault()`

---

## 4. Controller 层

### 4.1 定位与职责

**所在目录**：
- API Controller：[app/Domains/*/Api/Controllers/
- Web Controller：[app/Domains/*/Web/Controllers/

**核心职责**：作为 HTTP 请求的入口，协调各层之间的协调者，属于**应用层**。

### 4.2 API Controller 典型流程

以 [VaultController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVault/Api/Controllers/VaultController.php) 为例：

```php
public function store(Request $request)
{
    // 1. 组装数据（从 Request 提取 + 补加上下文信息
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

[ApiController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Http/Controllers/ApiController.php) 提供了：

1. **异常捕获与转换**：`callAction()` 方法统一捕获异常并转换为 HTTP 响应：

```php
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

2. **分页参数处理**：`limit` 参数的验证和设置

3. **JSON 响应工具**：通过 [JsonRespondController](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Traits/JsonRespondController.php) trait 提供统一的错误响应格式

### 4.4 Web Controller 特点

Web Controller 与 API Controller 类似，但返回的是 Inertia 视图而不是 JSON：

- 使用 ViewHelper 组装视图数据
- 也会调用 Gate 进行权限检查
- 也会调用 Domain Service 执行业务操作

---

## 5. 字段验证分工

### 5.1 验证层次结构

```
┌───────────────────────────────────────────────────────┐
│  ① HTTP 请求层验证 (FormRequest)                    │
│  仅认证相关，有限使用 (LoginRequest)               │
│  - 限流验证                                            │
│  - 认证逻辑                                          │
└───────────────────────────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────────────┐
│  ② Domain Service 层验证 (主要验证层)                    │
│  - 字段格式验证 (rules 方法)                          │
│  - 存在性验证 (exists 规则)                         │
│  - 权限验证 (permissions 方法)                       │
│  - 领域一致性验证 (validate 私有方法)                 │
└───────────────────────────────────────────────────────┘
                        ↓
┌───────────────────────────────────────────────────────┐
│  ③ 数据库层验证 (最后防线)                             │
│  - 数据库约束 (外键约束、唯一约束等                  │
└───────────────────────────────────────────────────────┘
```

### 5.2 各层验证详细说明

#### 5.2.1 HTTP 请求层验证 (FormRequest)

**所在位置**：[app/Http/Requests/

**使用场景**：仅用于认证相关的特殊场景

**职责**：
- 请求限流（Rate Limiting）
- 认证逻辑
- 仅限认证失败时的特殊验证

**示例**：[LoginRequest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Http/Requests/Auth/LoginRequest.php)

```php
class LoginRequest extends FormRequest
{
    public function rules()
    {
        return [
            'email' => ['required', 'string', 'email'],
            'password' => ['required', 'string'],
        ];
    }

    public function authenticate()
    {
        $this->ensureIsNotRateLimited();
        // ... 认证逻辑
    }
}
```

**特点**：
- 仅用于认证相关的有限场景
- 大部分业务验证不在此层验证

#### 5.2.2 Domain Service 层验证 (主要验证层)

**所在位置**：每个 Service 类内部

**使用场景**：所有业务操作的主要验证层

**职责**：

1. **字段格式验证**（rules 方法）
   - 必填/可选
   - 数据类型
   - 长度限制
   - 格式验证（UUID、email 等）

2. **存在性验证**（exists 规则）
   - 外键存在性检查
   - 确保关联数据存在性

3. **权限验证**（permissions 方法）
   - 用户身份验证
   - 操作权限检查

4. **领域一致性验证**（validate 私有方法）
   - 领域规则验证
   - 业务规则验证
   - 关联数据属于当前账户

**示例**：[CreateContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Contact/ManageContact/Services/CreateContact.php#L66-L84)

```php
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

**特点**：
- 是验证的核心层
- 所有业务操作都必须经过这层验证
- 验证逻辑与业务逻辑紧密结合

#### 5.2.3 数据库层验证 (最后防线)

**所在位置**：数据库表结构

**职责**：
- 外键约束
- 唯一约束
- 字段类型约束

**特点**：
- 是最后一道防线
- 不应依赖这层验证来保证数据完整性
- 异常由数据库异常会被上层捕获转换

---

## 6. 领域操作分工

### 6.1 领域操作完全在 Domain Service 层

领域操作（创建、更新、删除等）完全封装在 Domain Service 中，Controller 不直接操作 Model。

### 6.2 典型领域操作内容

以 [CreateVault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Domains/Vault/ManageVault/Services/CreateVault.php) 为例，一个创建操作包含：

1. **主实体创建**
   ```php
   private function createVault(): void
   {
       $this->vault = Vault::create([...]);
   }
   ```

2. **关联数据创建**
   ```php
   private function createUserContact(): void
   {
       $contact = Contact::create([...]);
       $this->vault->users()->save($this->author, [...
   }
   ```

3. **默认数据填充**
   ```php
   private function populateDefaultContactImportantDateTypes(): void
   {
       (new CreateContactImportantDateType)->execute([...]);
   }
   ```

4. **副作用操作
   - 更新最后编辑时间
   - 创建 Feed 项
   - 触发事件等

### 6.3 领域操作的特点

1. **原子性**：一个 Service 的 execute 方法代表一个完整的业务操作
2. **封装性**：Controller 不需要知道内部细节
3. **可复用性**：可以被 Web Controller、API Controller、Job 等调用
4. **可测试性**：可以独立测试

---

## 7. 异常验证分工

### 7.1 异常层次结构

```
┌───────────────────────────────────────────────────────┐
│  Domain Service 层（抛出异常）                    │
│  - ValidationException - 验证失败                    │
│  - ModelNotFoundException - 模型不存在           │
│  - NotEnoughPermissionException - 权限不足       │
│  - 其他领域异常                                 │
└───────────────────────────────────────────────────────┘
                        ↓ 抛出
┌───────────────────────────────────────────────────────┐
│  Controller 层（捕获并转换）                          │
│  - ModelNotFoundException → 404 Not Found          │
│  - QueryException → 500 Invalid Query            │
│  - ValidationException → 422 Validation Failed     │
└───────────────────────────────────────────────────────┘
                        ↓ 响应
┌───────────────────────────────────────────────────────┐
│  全局异常处理器（报告异常）                      │
│  - 异常报告 (Sentry)                            │
│  - 不做响应转换                               │
└───────────────────────────────────────────────────────┘
```

### 7.2 各层异常详细说明

#### 7.2.1 Domain Service 层（抛出异常）

**所在位置**：Domain Service 内部

**抛出的异常类型**：

1. **ValidationException**
   - 来源：Laravel Validator
   - 触发：字段验证失败时
   - 示例：`Validator::make($data, $this->rules())->validate()

2. **ModelNotFoundException**
   - 来源：Eloquent ORM
   - 触发：`findOrFail()` 找不到记录时
   - 示例：`$this->account()->genders()->findOrFail(...)

3. **NotEnoughPermissionException**
   - 来源：自定义异常
   - 触发：权限不足时
   - 位置：[NotEnoughPermissionException.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Exceptions/NotEnoughPermissionException.php)

4. **其他领域异常**：
   - `CantBeDeletedException` - 不可删除
   - `EntryAlreadyExistException` - 条目已存在
   - `MaximumNumberOfUsersInVaultException` - Vault 用户数超限
   - `SameUserException` - 相同用户

**特点**：
- 异常是领域语言描述，与 HTTP 状态码无关
- 异常表达业务含义
- 由 Service 不关心异常如何被处理

#### 7.2.2 Controller 层（捕获并转换）

**所在位置**：[ApiController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Http/Controllers/ApiController.php)

**捕获转换逻辑**：

```php
public function callAction($method, $parameters)
{
    try {
        return $this->{$method}(...array_values($parameters));
    } catch (ModelNotFoundException) {
        return $this->respondNotFound();          // 404
    } catch (QueryException) {
        return $this->respondInvalidQuery();      // 500
    } catch (ValidationException $e) {
        return $this->respondValidatorFailed($e->validator); // 422
    }
}
```

**转换规则**：

| 异常类型 | HTTP 状态码 | 错误码 | 说明 |
|---------|------------|--------|------|
| ModelNotFoundException | 404 | 31 | 资源不存在 |
| ValidationException | 422 | 32 | 验证失败 |
| QueryException | 500 | 40 | 查询错误 |
| - | 400 | 30 | 参数错误（limit 超限 |
| - | 401 | 42 | 未授权 |
| - | 422 | 41 | 无效参数 |

**特点**：
- 将领域异常转换为 HTTP 响应
- 统一错误响应格式
- 包含错误码和错误消息

#### 7.2.3 全局异常处理器（报告异常）

**所在位置**：[Handler.php](file:///d:/fz/0601-2/solo-dogfeeding/code/49-monica/app/Exceptions/Handler.php)

**职责**：
- 异常报告（发送到 Sentry）
- 不做具体的响应转换

**特点**：
- 主要用于监控和日志
- 不处理未被 Controller 捕获的异常
- 是最后一道异常处理

---

## 8. 三层职责总结对比

| 职责 | HTTP Resource | Domain Service | Controller |
|-----|--------------|---------------|------------|
| **字段验证 | ❌ | ✅ 主要验证层 | ❌ |
| **权限检查** | ❌ | ✅ 主要层 | ⚠️ Gate 辅助 |
| **业务逻辑** | ❌ | ✅ 核心层 | ❌ |
| **数据格式化** | ✅ 主要层 | ❌ | ❌ |
| **数据组装** | ❌ | ❌ | ✅ |
| **异常抛出** | ❌ | ✅ 领域异常 | ❌ |
| **异常转换** | ❌ | ❌ | ✅ HTTP 状态码 |
| **HTTP 响应** | ✅ 数据部分 | ❌ | ✅ 完整响应 |
| **路由链接** | ✅ HATEOAS | ❌ | ❌ |
| **可复用性** | 低 | 高 | 中 |

---

## 9. 调用流程示例

### 9.1 API 创建 Vault 完整流程

```
1. HTTP 请求 → POST /api/vaults
   │
   ▼
2. Route → VaultController::store()
   │  ├─ 从 Request 提取 name、description
   │  ├─ 从 Auth 获取 user()->account_id,
   │  │  user()->id
   │  └─ 组装 $data 数组
   │
   ▼
3. CreateVault::execute($data)
   │
   ├─ validateRules($data)
   │  ├─ 调用 rules() 验证字段
   │  ├─ 调用 permissions() 检查权限
   │  └─ 验证失败抛出异常
   │
   ├─ createVault() → 创建 Vault
   ├─ createUserContact() → 创建用户联系人
   ├─ populateDefault...() → 填充默认数据
   │
   └─ 返回 Vault 模型
   │
   ▼
4. new VaultResource($vault)
   │
   └─ 转换为 API 响应格式
      ├─ id, name, description
      ├─ created_at, updated_at
      └─ links.self
   │
   ▼
5. 返回 JSON 响应
```

### 9.2 异常流程示例

```
1. 请求 → 调用 Service
   │
   ▼
2. Service 内部 validate()
   │
   ├─ findOrFail() 找不到记录
   │
   ▼
3. 抛出 ModelNotFoundException
   │
   ▼
4. ApiController::callAction() 捕获
   │
   ├─ catch (ModelNotFoundException)
   │
   ▼
5. 调用 respondNotFound()
   │
   ├─ 设置 HTTP 404
   ├─ 设置错误码 31
   └─ 返回 JSON 错误响应
   │
   ▼
6. 返回 404 JSON 响应
```

---

## 10. 设计原则与优点

### 10.1 单一职责原则

- Resource 只负责数据展示
- Service 只负责业务逻辑
- Controller 只负责请求协调

### 10.2 依赖倒置原则

- 上层依赖下层
- 下层不依赖上层
- Service 不知道 Resource 和 Controller

### 10.3 可测试性

- Service 可以独立测试
- 不依赖 HTTP 环境
- 可以被多种调用方式（Web、API、Job、CLI）

### 10.4 可维护性

- 业务逻辑集中在 Service 层
- 修改业务只需要修改 Service
- 表现层变化影响业务逻辑

---

## 11. 常见疑问点澄清

### 11.1 为什么验证不在 Controller 层？

- 业务验证与业务逻辑紧密相关
- Service 可以被多种调用方式复用（Web、API、Job）
- 保证所有入口都经过同样的验证

### 11.2 为什么有 FormRequest？

- FormRequest 是 Laravel 的推荐方式
- 但项目选择了 Service 层验证
- 可能是为了更好的复用性和领域驱动设计的选择

### 11.3 为什么 Resource 为什么叫 Action 是同一个概念吗？

- Domain Service = Domain Action
- 项目中叫 Service
- 但本质上是 Action 是领域操作
- 遵循命令模式的概念类似

### 11.4 异常为什么不在 Service 层处理？

- Service 抛出领域异常
- Controller 转换为 HTTP 异常
- 保持领域层与 HTTP 解耦
- Service 可以被非 HTTP 场景使用

