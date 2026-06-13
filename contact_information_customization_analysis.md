# Monica 联系人信息类型自定义实现分析

## 一、整体架构概览

Monica 的联系人信息系统采用 **"类型元数据 + 实例数据"** 的分离设计：

- **ContactInformationType**（元数据层）：定义"有哪些信息类型"（如邮箱、电话、社交账号等），每个账户独立配置
- **ContactInformation**（实例数据层）：存储联系人的具体信息条目，通过 `type_id` 关联到类型

这种设计使得管理员可以灵活增删信息类型，而无需改动数据库结构。

---

## 二、数据模型设计

### 2.1 核心表结构

#### `contact_information_types` 表（类型定义表）

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint | 主键 |
| `account_id` | uuid | 所属账户（外键 → accounts） |
| `name` | string, nullable | 自定义名称（用户覆盖时使用） |
| `name_translation_key` | string, nullable | 翻译键（系统默认类型使用，支持多语言） |
| `protocol` | string, nullable | URL 协议前缀，如 `mailto:`、`tel:` |
| `can_be_deleted` | boolean | 是否可删除（系统内置类型为 false） |
| `type` | string, nullable | 分组标识，如 `email`、`phone`、`IMPP`、`X-SOCIAL-PROFILE` |

对应迁移文件：[2021_10_20_004100_create_contact_fields_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/database/migrations/2021_10_20_004100_create_contact_fields_table.php)

#### `contact_information` 表（实例数据表）

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | bigint | 主键 |
| `contact_id` | uuid | 关联联系人（外键 → contacts） |
| `type_id` | bigint | 关联信息类型（外键 → contact_information_types） |
| `data` | string | 具体内容，如 `user@example.com` |
| `kind` | string, nullable | 子类型标识，如 `work`、`home`、`cell` |
| `pref` | boolean | 是否为首选联系方式（默认 true） |

对应迁移文件：[2021_10_20_004100_create_contact_fields_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/database/migrations/2021_10_20_004100_create_contact_fields_table.php)，kind/pref 字段由 [2025_07_07_213621_add_contact_information_kind.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/database/migrations/2025_07_07_213621_add_contact_information_kind.php) 追加。

### 2.2 模型关系

- **Account 模型**：`hasMany(ContactInformationType::class)` — 一个账户拥有多个信息类型
  见 [Account.php#L100-L103](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Models/Account.php#L100-L103)

- **Contact 模型**：`hasMany(ContactInformation::class)` — 一个联系人拥有多条信息
  见 [Contact.php#L204-L207](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Models/Contact.php#L204-L207)

- **ContactInformation 模型**：`belongsTo(ContactInformationType::class, 'type_id')` — 每条信息关联一个类型
  见 [ContactInformation.php#L44-L47](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Models/ContactInformation.php#L44-L47)

---

## 三、联系人信息类型的自定义配置

### 3.1 配置分层：硬编码配置 + 数据库动态配置

Monica 将类型配置分为三层：

#### 第一层：全局硬编码配置（config/app.php）

在 [app.php#L141-L279](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/config/app.php#L141-L279) 中定义了三类配置：

**`social_protocols`** — 社交平台协议映射：
```php
'social_protocols' => [
    'Facebook' => [
        'name_translation_key' => trans_key('Facebook'),
        'url' => 'https://www.facebook.com/',
        'type' => 'X-SOCIAL-PROFILE',
    ],
    // ... 共 11 个平台（Facebook, Whatsapp, Telegram, LinkedIn, Instagram, Twitter, Hangouts, Mastodon, Bluesky, Threads, GitHub）
]
```

**`contact_information_groups`** — 类型分组（用于前端 optgroup 展示）：
```php
'contact_information_groups' => [
    'email'            => ['name_translation_key' => trans_key('Email address')],
    'phone'            => ['name_translation_key' => trans_key('Phone number')],
    'IMPP'             => ['name_translation_key' => trans_key('Instant messaging')],
    'X-SOCIAL-PROFILE' => ['name_translation_key' => trans_key('Social profile')],
]
```

**`contact_information_kinds`** — 子类型（仅 email 和 phone 有）：
```php
'contact_information_kinds' => [
    'email' => [
        ['id' => 'work',     'name_translation_key' => trans_key('🏢 Work')],
        ['id' => 'home',     'name_translation_key' => trans_key('🏡 Home')],
        ['id' => 'personal', 'name_translation_key' => trans_key('🧑🏼 Personal')],
        ['id' => 'other',    'name_translation_key' => trans_key('❔ Other')],
    ],
    'phone' => [
        ['id' => 'work',  'name_translation_key' => trans_key('🏢 Work')],
        ['id' => 'home',  'name_translation_key' => trans_key('🏡 Home')],
        ['id' => 'cell',  'name_translation_key' => trans_key('📱 Mobile')],
        ['id' => 'fax',   'name_translation_key' => trans_key('📠 Fax')],
        ['id' => 'pager', 'name_translation_key' => trans_key('📟 Pager')],
        ['id' => 'other', 'name_translation_key' => trans_key('❔ Other')],
    ],
]
```

#### 第二层：账户初始化时的默认类型

创建账户时，[SetupAccount.php#L965-L995](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Settings/CreateAccount/Jobs/SetupAccount.php#L965-L995) 的 `addContactInformation()` 方法会为新账户批量创建默认类型：

```php
private function addContactInformation(): void
{
    // 1. 创建 Email 类型（不可删除）
    $information = (new CreateContactInformationType)->execute([
        'name_translation_key' => trans_key('Email address'),
        'protocol' => 'mailto:',
        'type' => 'email',
    ]);
    $information->can_be_deleted = false;
    $information->save();

    // 2. 创建 Phone 类型（不可删除）
    $information = (new CreateContactInformationType)->execute([
        'name_translation_key' => trans_key('Phone'),
        'protocol' => 'tel:',
        'type' => 'phone',
    ]);
    $information->can_be_deleted = false;
    $information->save();

    // 3. 遍历配置中的所有社交平台，逐一创建类型
    foreach (config('app.social_protocols') as $socialProtocol) {
        (new CreateContactInformationType)->execute([
            'name_translation_key' => $socialProtocol['name_translation_key'],
            'type' => $socialProtocol['type'],
        ]);
    }
}
```

**关键点**：
- `email` 和 `phone` 被标记为 `can_be_deleted = false`，是系统内置不可删除类型
- 社交平台类型均可删除，用户可按需管理

#### 第三层：用户自定义类型（运行时增删改）

用户可在"设置 → 个性化 → 联系人信息类型"页面管理自定义类型，对应服务类：

| 操作 | 服务类 | 权限要求 |
|------|--------|----------|
| 创建 | [CreateContactInformationType.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Settings/ManageContactInformationTypes/Services/CreateContactInformationType.php) | 账户管理员 |
| 更新 | [UpdateContactInformationType.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Settings/ManageContactInformationTypes/Services/UpdateContactInformationType.php) | 账户管理员 |
| 删除 | [DestroyContactInformationType.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Settings/ManageContactInformationTypes/Services/DestroyContactInformationType.php) | 账户管理员 |

**创建逻辑**（[CreateContactInformationType.php#L40-L51](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Settings/ManageContactInformationTypes/Services/CreateContactInformationType.php#L40-L51)）：
```php
public function execute(array $data): ContactInformationType
{
    $this->validateRules($data);
    return ContactInformationType::create([
        'account_id'           => $data['account_id'],
        'name'                 => $data['name'] ?? null,
        'name_translation_key' => $data['name_translation_key'] ?? null,
        'type'                 => $this->valueOrNull($data, 'type'),
        'protocol'             => $this->valueOrNull($data, 'protocol'),
    ]);
}
```

### 3.2 多语言名称的智能解析

[ContactInformationType.php#L53-L65](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Models/ContactInformationType.php#L53-L65) 中的 `name()` 访问器实现了优先级策略：

```php
protected function name(): Attribute
{
    return Attribute::make(
        get: function ($value, $attributes) {
            // 若用户设置了自定义 name，直接使用
            if ($value === null || $value == '') {
                // 否则使用翻译键进行本地化
                return __($attributes['name_translation_key']);
            }
            return $value;
        },
    );
}
```

**优先级**：用户自定义 `name` > 翻译键 `name_translation_key` 的本地化结果

---

## 四、联系人信息的录入流程

### 4.1 核心服务类

| 操作 | 服务类 | 主要职责 |
|------|--------|----------|
| 创建 | [CreateContactInformation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Contact/ManageContactInformation/Services/CreateContactInformation.php) | 验证类型归属 → 创建记录 → 更新联系人编辑时间 → 创建动态流 |
| 更新 | [UpdateContactInformation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Contact/ManageContactInformation/Services/UpdateContactInformation.php) | 验证类型归属 → 更新字段 → 更新编辑时间 → 创建动态流 |
| 删除 | [DestroyContactInformation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Contact/ManageContactInformation/Services/DestroyContactInformation.php) | 验证权限 → 删除记录 → 更新编辑时间 → 创建动态流 |

### 4.2 创建流程详解

[CreateContactInformation.php#L52-L96](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Contact/ManageContactInformation/Services/CreateContactInformation.php#L52-L96)：

```
执行步骤：
1. validateRules()        — 参数校验（必填、类型存在等）
2. validate()             — 业务校验：确保 contact_information_type_id 属于当前账户
3. create()               — 写入 contact_information 表
4. updateLastEditedDate() — 更新 contact.last_updated_at
5. createFeedItem()       — 写入 contact_feed_items，记录"信息已创建"
```

**关键校验**（[CreateContactInformation.php#L63-L69](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Contact/ManageContactInformation/Services/CreateContactInformation.php#L63-L69)）：
```php
private function validate(): void
{
    $this->validateRules($this->data);
    // 确保类型属于当前账户，防止越权
    $this->contactInformationType = $this->account()->contactInformationTypes()
        ->findOrFail($this->data['contact_information_type_id']);
}
```

### 4.3 Web 控制器层

[ContactInformationController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Contact/ManageContactInformation/Web/Controllers/ContactInformationController.php) 提供三个 JSON API 端点：

| 方法 | 路由 | 说明 |
|------|------|------|
| `store()` | POST `/vault/{vault}/contact/{contact}/information` | 创建联系人信息 |
| `update()` | PUT `/vault/{vault}/contact/{contact}/information/{info}` | 更新联系人信息 |
| `destroy()` | DELETE `/vault/{vault}/contact/{contact}/information/{info}` | 删除联系人信息 |

三个端点均通过 `Auth::user()->account_id` 注入账户上下文，保证数据隔离。

---

## 五、联系人信息的展示逻辑

### 5.1 数据组装：按类型分组

[ModuleContactInformationViewHelper.php#L14-L56](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Contact/ManageContactInformation/Web/ViewHelpers/ModuleContactInformationViewHelper.php#L14-L56) 的 `data()` 方法是核心展示逻辑：

```php
public static function data(Contact $contact, User $user): array
{
    // 1. 拉取该联系人的所有信息，并按类型分组
    $infos = $contact->contactInformations()
        ->with('contactInformationType')
        ->get()
        ->groupBy(fn (ContactInformation $info) => $info->contactInformationType->type)
        ->map(fn (Collection $collection) => $collection
            ->map(fn (ContactInformation $info) => self::dto($info))
        );

    // 2. 拉取当前账户所有可用的信息类型（用于下拉选择），同样按 type 分组
    $infoTypes = $user->account
        ->contactInformationTypes()
        ->get()
        ->groupBy('type')
        ->map(fn (Collection $collection) => [
            'optgroup' => $groups[$collection[0]->type],  // 分组显示名
            'options'  => $collection->map(...)->sortByCollator('name')->toArray(),
        ]);

    return [
        'contact_information'         => $infos,
        'contact_information_types'   => $infoTypes,
        'contact_information_kinds'   => self::infoKinds(),
        'contact_information_groups'  => self::infoGroups(),
        'protocols'                   => config('app.social_protocols'),
        'url' => ['store' => route(...)],
    ];
}
```

**分组逻辑**：使用 `ContactInformationType.type` 字段（如 `email`、`phone`、`IMPP`、`X-SOCIAL-PROFILE`）作为分组键，前端按组渲染。

### 5.2 单条信息 DTO 转换

[ModuleContactInformationViewHelper.php#L77-L117](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Contact/ManageContactInformation/Web/ViewHelpers/ModuleContactInformationViewHelper.php#L77-L117) 中的 `dto()` 方法：

```php
public static function dto(ContactInformation $info): array
{
    return [
        'id'                     => $info->id,
        'label'                  => $info->name,              // 显示标签
        'protocol'               => $info->contactInformationType->protocol,
        'data'                   => $info->data,              // 原始数据
        'data_with_protocol'     => $info->dataWithProtocol,  // 带协议的完整 URL
        'contact_information_type' => [...],
        'contact_information_kind' => $kindLabel,             // 子类型（如"工作"、"手机"）
        'url' => ['update' => ..., 'destroy' => ...],
    ];
}
```

### 5.3 智能标签显示

[ContactInformation.php#L56-L70](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Models/ContactInformation.php#L56-L70) 的 `name()` 访问器：

```php
protected function name(): Attribute
{
    return Attribute::make(
        get: function () {
            $type = $this->contactInformationType;
            // 系统内置类型（email/phone）：显示 data 本身
            if (! $type->can_be_deleted) {
                return $this->data;
            }
            // 自定义/社交类型：显示类型名称（如"Facebook"）
            return $type->name;
        },
    );
}
```

**效果**：
- 邮箱/电话显示 `user@example.com`（data）
- Facebook 等社交类型显示 `Facebook`（类型名），data 则作为可点击链接展示

### 5.4 协议与 URL 生成

[ContactInformation.php#L77-L92](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Models/ContactInformation.php#L77-L92) 的 `dataWithProtocol` 访问器：

```php
protected function dataWithProtocol(): Attribute
{
    return Attribute::get(function () {
        // 1. 优先使用数据库中存储的 protocol 字段（如 mailto:、tel:）
        if ($this->contactInformationType->protocol !== null) {
            return $this->contactInformationType->protocol . $this->data;
        }
        // 2. 否则根据 name_translation_key 从配置 social_protocols 中匹配
        $protocols = collect(config('app.social_protocols'));
        if (($protocol = $protocols->firstWhere('name_translation_key', $this->contactInformationType->name_translation_key)) !== null) {
            return $protocol['url'] . $this->data;
        }
        return null;
    });
}
```

**示例**：
- Email：`mailto:user@example.com`（来自 protocol 字段）
- Phone：`tel:+1234567890`（来自 protocol 字段）
- Facebook：`https://www.facebook.com/zuck`（从 social_protocols 配置中匹配 URL）

---

## 六、前端交互（Vue 组件）

### 6.1 类型管理页面

[resources/js/Pages/Settings/Personalize/ContactInformationTypes/Index.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/resources/js/Pages/Settings/Personalize/ContactInformationTypes/Index.vue)

**功能点**：
1. 按分组（Email、Phone、即时通讯、社交资料）展示所有类型
2. 新建类型表单：选择所属分组 → 填写名称 → 可选填写协议
3. 编辑/删除操作：系统内置类型（email/phone）的"删除"按钮被隐藏（通过 `can_be_deleted` 控制）
4. 删除前二次确认：提示"将从所有联系人中移除该类型，但不会删除联系人"

### 6.2 联系人信息录入与展示模块

[resources/js/Shared/Modules/ContactInformation.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/resources/js/Shared/Modules/ContactInformation.vue)

**动态响应式交互**：
```javascript
// 监听选中的类型 ID，动态切换子类型选项和协议提示
watch(
  () => form.contact_information_type_id,
  (newValue) => {
    for (const optgroup in props.data.contact_information_types) {
      let types = props.data.contact_information_types[optgroup].options;
      let type = types.find((x) => x.id === newValue);
      kinds.value = props.data.contact_information_kinds[type.type] ?? null;
      protocol.value = props.data.protocols[type.name_translation_key]?.url ?? null;
      return;
    }
  }
)
```

**录入表单**：
- **类型下拉**：使用 optgroup 分组展示（邮箱/电话/即时通讯/社交资料）
- **子类型（Kind）**：仅当类型为 email 或 phone 时显示 ComboBox，允许"工作/家/手机/..."或自定义
- **数据输入框**：placeholder 动态显示对应协议的 URL 前缀（如 `https://twitter.com/`）

**展示**：
- 按分组标题分块渲染（"Email address"、"Phone number" 等）
- 每条信息显示可点击链接（`data_with_protocol`）+ 子类型标签（如"— 🏢 Work"）+ 类型名（如 `(Facebook)`）
- 内联编辑/删除操作

---

## 七、配置性对录入与展示的影响总结

| 配置维度 | 影响录入 | 影响展示 |
|----------|----------|----------|
| **新增自定义类型** | 下拉框出现新选项，可选择录入 | 自动出现在对应分组下展示 |
| **删除类型** | 该类型不再可选择（级联删除已有数据，外键 `cascadeOnDelete`） | 对应信息消失 |
| **修改类型名称** | 下拉框显示新名称 | 已有信息的标签同步更新 |
| **修改协议** | 输入框 placeholder 变化 | 链接点击跳转地址变化 |
| **type 分组标识** | 决定下拉框归入哪个 optgroup | 决定展示时归入哪个标题分块 |
| **kind 子类型配置** | 仅 email/phone 显示子类型选择 | 展示"— 🏢 Work"等标签 |
| **name 翻译键** | - | 系统类型支持多语言显示 |
| **can_be_deleted** | 禁止删除系统关键类型（email/phone） | 决定该类型显示 data 还是 type.name |

---

## 八、关键文件索引

| 文件 | 作用 |
|------|------|
| [ContactInformation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Models/ContactInformation.php) | 信息实例模型，含 name 和 dataWithProtocol 访问器 |
| [ContactInformationType.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Models/ContactInformationType.php) | 信息类型模型，含多语言 name 访问器 |
| [CreateContactInformationType.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Settings/ManageContactInformationTypes/Services/CreateContactInformationType.php) | 创建类型服务 |
| [UpdateContactInformationType.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Settings/ManageContactInformationTypes/Services/UpdateContactInformationType.php) | 更新类型服务 |
| [DestroyContactInformationType.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Settings/ManageContactInformationTypes/Services/DestroyContactInformationType.php) | 删除类型服务 |
| [CreateContactInformation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Contact/ManageContactInformation/Services/CreateContactInformation.php) | 创建信息服务 |
| [UpdateContactInformation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Contact/ManageContactInformation/Services/UpdateContactInformation.php) | 更新信息服务 |
| [DestroyContactInformation.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Contact/ManageContactInformation/Services/DestroyContactInformation.php) | 删除信息服务 |
| [ModuleContactInformationViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Contact/ManageContactInformation/Web/ViewHelpers/ModuleContactInformationViewHelper.php) | 联系人页展示数据组装 |
| [PersonalizeContactInformationTypeIndexViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Settings/ManageContactInformationTypes/Web/ViewHelpers/PersonalizeContactInformationTypeIndexViewHelper.php) | 设置页展示数据组装 |
| [ContactInformationController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Contact/ManageContactInformation/Web/Controllers/ContactInformationController.php) | 联系人信息 CRUD API |
| [PersonalizeContatInformationTypesController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Settings/ManageContactInformationTypes/Web/Controllers/PersonalizeContatInformationTypesController.php) | 类型管理 API |
| [SetupAccount.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/app/Domains/Settings/CreateAccount/Jobs/SetupAccount.php) | 账户初始化时创建默认类型 |
| [config/app.php](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/config/app.php) | 全局配置（social_protocols、分组、子类型） |
| [ContactInformation.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/resources/js/Shared/Modules/ContactInformation.vue) | 前端联系人信息模块 |
| [Index.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/56-monica/resources/js/Pages/Settings/Personalize/ContactInformationTypes/Index.vue) | 前端类型管理页面 |
