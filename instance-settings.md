# 实例全局设置读写回路分析

本文档分别阐述 Monica 项目中三个层级设置数据的完整生命周期：读取路径、运行时缓存机制及界面写回方式。

---

## 一、环境配置（实例级配置）

### 1.1 概述
环境配置属于整个部署实例的全局配置，影响所有账户和用户。此层级配置**不提供 Web 界面修改能力**。

### 1.2 存储位置
- **源文件**：项目根目录 `.env` 文件
- **配置层**：`config/` 目录下的 PHP 配置文件，主要包括：
  - [config/monica.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/config/monica.php) - Monica 应用专用配置
  - [config/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/config/app.php) - Laravel 应用基础配置
  - [config/cache.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/config/cache.php) - 缓存配置
  - [config/filesystems.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/config/filesystems.php) - 文件系统配置
  - [config/services.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/config/services.php) - 第三方服务配置

### 1.3 读取路径
```
.env 环境变量
    ↓
Laravel 框架启动时加载（DotEnv）
    ↓
config/*.php 文件通过 env() 函数读取环境变量并组装配置数组
    ↓
业务代码通过 config('key') 或 Config Facade 读取
```

### 1.4 典型配置项及代码读取位置

| 配置项 | .env 变量 | 读取代码位置 |
|--------|-----------|-------------|
| 是否禁用注册 | `APP_DISABLE_SIGNUP` | [SignupHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/SignupHelper.php#L27-L30) |
| 默认存储限制(MB) | `DEFAULT_STORAGE_LIMIT` | [CreateAccount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/CreateAccount/Services/CreateAccount.php#L43-L45) |
| Mapbox API Key | `MAPBOX_API_KEY` | [MapHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/MapHelper.php) |
| LocationIQ API Key | `LOCATION_IQ_API_KEY` | [GetGPSCoordinate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Vault/ManageAddresses/Services/GetGPSCoordinate.php) |
| 帮助链接 | 硬编码于 monica.php | [HandleInertiaRequests.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Http/Middleware/HandleInertiaRequests.php#L31-L32) |
| Uploadcare 密钥 | `UPLOADCARE_PUBLIC_KEY`, `UPLOADCARE_PRIVATE_KEY` | [StorageHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/StorageHelper.php#L36-L45) |
| 应用版本号 | `APP_VERSION` | [HandleInertiaRequests.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Http/Middleware/HandleInertiaRequests.php#L56-L71) |
| 强制 HTTPS URL | `APP_FORCE_URL` | [AppServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Providers/AppServiceProvider.php#L132-L135) |

### 1.5 读取代码示例

**SignupHelper（检查是否允许注册）**：
[SignupHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/SignupHelper.php)
```php
// 通过依赖注入 Config Repository
public function __construct(private Config $config) {}

private function isDisabledByConfig(): bool
{
    return $this->config->get('monica.disable_signup', false);
}
```

**HandleInertiaRequests（共享全局数据到前端）**：
[HandleInertiaRequests.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Http/Middleware/HandleInertiaRequests.php)
```php
public function share(Request $request)
{
    return [
        ...parent::share($request),
        'help_links' => fn () => config('monica.help_links'),
        'help_url' => fn () => config('monica.help_center_url'),
        'sentry' => fn () => [
            'dsn' => config('sentry.dsn'),
            'release' => config('sentry.release'),
            // ...
        ],
        // ...
    ];
}
```

### 1.6 运行时缓存机制
- **Laravel 配置缓存**：框架层面提供 `php artisan config:cache` 命令，将所有配置编译为单一文件，加速读取。
- **内存驻留**：配置加载后存储于 Laravel 服务容器中，单次请求生命周期内直接从内存读取，无额外 IO。
- **本项目未使用**：`Cache` Facade 对此层级配置做额外缓存。

### 1.7 界面写回
**不支持界面写回。** 此层级配置修改方式：
1. 直接编辑服务器上的 `.env` 文件
2. 通过容器编排（Docker/K8s）的环境变量注入
3. 修改后需执行 `php artisan config:clear` 或重启应用生效

---

## 二、账户数据（Account 级设置）

### 2.1 概述
账户级数据属于每个 Account（租户）独立拥有的配置，存储于数据库中。账户管理员可管理大部分关联数据。

### 2.2 存储位置
核心表和关联表均通过 `account_id` 外键关联到账户：

| 表名 | 用途 | 模型文件 |
|------|------|---------|
| `accounts` | 账户主表（含存储限制） | [Account.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/Account.php) |
| `genders` | 性别选项 | [Gender.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/Gender.php) |
| `pronouns` | 代词选项 | [Pronoun.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/Pronoun.php) |
| `address_types` | 地址类型 | [AddressType.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/AddressType.php) |
| `relationship_types` / `relationship_group_types` | 关系类型 | [RelationshipType.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/RelationshipType.php) |
| `templates` / `template_pages` / `modules` | 联系人模板及模块 | [Template.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/Template.php) |
| `currencies` 多对多 | 启用的货币 | [Currency.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/Currency.php) |
| ... 更多 | ... | ... |

**accounts 表结构**：
迁移文件 [2013_04_25_132851_create_accounts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/database/migrations/2013_04_25_132851_create_accounts_table.php)
- `id` (uuid, 主键)
- `storage_limit_in_mb` (integer, 默认 0 - 0 表示不限制)
- `timestamps`

### 2.3 读取路径

#### 2.3.1 账户存储限制读取（以 Storage 页面为例）
```
用户访问 /settings/storage
    ↓
路由 → AccountStorageController@index
    ↓
Auth::user()->account  // 通过用户的 account_id 关联查询
    ↓
StorageIndexViewHelper::data($account)
    ├─ 读取 $account->storage_limit_in_mb
    ├─ 查询 files 表统计当前使用量
    └─ 组装统计数据 DTO
    ↓
Inertia::render() 传给前端
    ↓
Vue 组件渲染显示
```

**控制器代码**：
[AccountStorageController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageStorage/Web/Controllers/AccountStorageController.php)
```php
public function index()
{
    return Inertia::render('Settings/Storage/Index', [
        'layoutData' => VaultIndexViewHelper::layoutData(),
        'data' => StorageIndexViewHelper::data(Auth::user()->account),
    ]);
}
```

**视图模型代码**：
[StorageIndexViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageStorage/Web/ViewHelpers/StorageIndexViewHelper.php)
```php
public static function data(Account $account): array
{
    $vaultIds = $account->vaults()->select('id')->get()->toArray();
    $totalSizeInBytes = File::whereIn('vault_id', $vaultIds)->sum('size');
    $accountLimit = $account->storage_limit_in_mb * 1024 * 1024;
    // ... 组装统计数据
}
```

#### 2.3.2 存储限制业务校验（上传文件时）
[StorageHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/StorageHelper.php)
```php
public static function canUploadFile(Account $account): bool
{
    if ($account->storage_limit_in_mb == 0) {
        return true;  // 0 表示不限制
    }
    $vaultIds = $account->vaults()->select('id')->get()->toArray();
    $totalSizeInBytes = File::whereIn('vault_id', $vaultIds)->sum('size');
    $accountLimit = $account->storage_limit_in_mb * 1024 * 1024;
    return $totalSizeInBytes < $accountLimit;
}
```

### 2.4 运行时缓存机制
**账户主表数据无专门的缓存层。**
- 每次请求通过 Eloquent ORM 直接查询数据库
- 依赖 Eloquent 模型的「同一请求内对象缓存」（identity map），相同 ID 的模型不会重复查询
- 本项目的 `Cache::store('array')` 请求级缓存不用于缓存 Account 模型本身，仅用于缓存衍生数据（如 Vault 权限、关联联系人信息）

**当前项目使用 Cache 的 Helper 类（均为请求级 array 缓存）**：

| Helper 类 | 缓存内容 | 缓存键 | 过期时间 |
|-----------|---------|--------|---------|
| [VaultHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/VaultHelper.php#L40-L53) | 用户在 Vault 中的权限 | `Permission:{userId}:{vaultId}` | 5 秒 |
| [UserHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/UserHelper.php#L15-L38) | 用户在 Vault 中的关联联系人 | `InformationAboutContact:{userId}:{vaultId}` | 5 秒 |
| [ContactImportantDateHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/ContactImportantDateHelper.php#L13-L21) | Vault 中的重要日期类型 | `ImportantDateType:{vaultId}:{type}` | 5 秒 |

### 2.5 界面写回

#### 2.5.1 storage_limit_in_mb（账户存储限制）
**无 Web 界面修改入口。** 仅在创建新账户时写入一次：

[CreateAccount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/CreateAccount/Services/CreateAccount.php#L43-L45)
```php
$this->account = Account::create([
    'storage_limit_in_mb' => config('monica.default_storage_limit_in_mb'),
]);
```

修改方式：需在数据库中直接操作，或由实例管理员通过代码/Artisan 命令修改。

#### 2.5.2 账户关联数据（性别、关系类型、模板等）
此类设置提供完整的 Web 界面 CRUD 能力，由账户管理员操作。

**通用写入流程（以性别管理为例）**：
```
界面操作（创建/更新/删除 性别选项）
    ↓
axios 请求
    ↓
路由 → Personalize*Controller
    ↓
对应 Service 类（CreateGender / UpdateGender / DestroyGender）
    ├─ validateRules() - 字段校验
    ├─ validatePermission() - 校验 author_must_be_account_administrator
    └─ DB 操作
    ↓
返回 JSON 响应
    ↓
前端刷新本地状态
```

**服务基类权限检查**：
[BaseService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Services/BaseService.php#L170-L175)
```php
private function validateAuthorIsAccountAdministrator(): void
{
    if (! $this->author->is_account_administrator) {
        throw new NotEnoughPermissionException;
    }
}
```

### 2.6 账户初始化（首次创建时）
新账户创建后，通过队列任务 [SetupAccount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/CreateAccount/Jobs/SetupAccount.php) 异步初始化所有默认关联数据，包括：货币、通知渠道、模板、模块、性别、代词、组类型、关系类型、地址类型、通话原因、联系方式类型、宠物分类、情绪、礼物场合、礼物状态、帖子模板、宗教等 17 类默认数据。

---

## 三、用户数据（User 偏好设置）

### 3.1 概述
用户偏好设置存储于 `users` 表中，每个用户独立维护，用户本人可修改。

### 3.2 存储位置
**users 表偏好字段**：
迁移文件 [2014_10_12_000000_create_users_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/database/migrations/2014_10_12_000000_create_users_table.php)

| 字段 | 类型 | 默认值 | 含义 |
|------|------|--------|------|
| `name_order` | string | '%first_name% %last_name%' | 姓名显示顺序 |
| `date_format` | string | 'MMM DD, YYYY' | 日期显示格式 |
| `timezone` | string | nullable | 时区 |
| `number_format` | string(8) | 'locale' | 数字格式 |
| `default_map_site` | string | 'open_street_maps' | 默认地图服务 |
| `distance_format` | string | 'mi' | 距离单位 |
| `locale` | string | 'en' | 界面语言 |
| `help_shown` | boolean | true | 是否显示帮助提示 |
| `contact_sort_order` | string | - | 联系人排序方式 |
| `is_account_administrator` | boolean | false | 是否账户管理员 |
| `is_instance_administrator` | boolean | false | 是否实例管理员 |

模型定义见 [User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/User.php#L80-L155)。

### 3.3 读取路径
```
用户访问 /settings/preferences
    ↓
路由 → PreferencesController@index
    ↓
Auth::user()  // 获取当前认证用户模型（从数据库或 session 解析）
    ↓
UserPreferencesIndexViewHelper::data($user)
    ├─ dtoHelp()       - 从 $user->help_shown 读取
    ├─ dtoNameOrder()  - 从 $user->name_order 读取
    ├─ dtoDateFormat() - 从 $user->date_format 读取
    ├─ dtoTimezone()   - 从 $user->timezone 读取
    ├─ dtoNumberFormat() - 从 $user->number_format 读取
    ├─ dtoDistanceFormat() - 从 $user->distance_format 读取
    ├─ dtoMapsPreferences() - 从 $user->default_map_site 读取
    └─ dtoLocale()     - 从 $user->locale 读取
    ↓
Inertia::render() 将 DTO 传给前端
    ↓
Vue 父组件 Index.vue 接收 props.data
    ↓
分发给各子组件（DateFormat.vue、Locale.vue 等）
    ↓
子组件 mounted() 将 props 同步到本地 data，然后渲染
```

**控制器代码**：
[PreferencesController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesController.php)
```php
public function index()
{
    return Inertia::render('Settings/Preferences/Index', [
        'layoutData' => VaultIndexViewHelper::layoutData(),
        'data' => UserPreferencesIndexViewHelper::data(Auth::user()),
    ]);
}
```

**视图模型示例（日期格式）**：
[UserPreferencesIndexViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/ViewHelpers/UserPreferencesIndexViewHelper.php#L62-L96)
```php
public static function dtoDateFormat(User $user): array
{
    $date = Carbon::now();
    $collection = collect();
    // 组装可选格式列表（4种选项）...
    return [
        'dates' => $collection,
        'date_format' => $user->date_format,
        'human_date_format' => Carbon::now()->isoFormat($user->date_format),
        'url' => ['store' => route('settings.preferences.date.store')],
    ];
}
```

### 3.4 运行时缓存机制
**用户偏好字段本身无专门缓存**，读取流程：
1. Laravel 的 Auth 系统解析当前用户时执行一次数据库查询
2. 同一请求内通过 `Auth::user()` 重复获取返回同一模型对象
3. 没有使用 `Cache` Facade 对用户偏好字段做跨请求缓存

**实例管理员权限读取（Gate 定义）**：
[AppServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Providers/AppServiceProvider.php#L169)
```php
Gate::define('viewPulse', fn (User $user) => 
    $user->is_instance_administrator || $this->app->environment('local')
);
```

### 3.5 界面写回
所有 8 项用户偏好均提供独立的修改接口，采用完全一致的架构模式。

#### 3.5.1 通用写入流程
```
Vue 子组件（如 DateFormat.vue）
    ↓ 用户点击 Edit → 选择新值 → 点击 Save
submit() 方法
    ↓
axios.post(data.url.store, formData)
    ↓
路由匹配 → 对应 Controller（PreferencesDateFormatController@store）
    ↓
Controller 构造 $data = [account_id, author_id, 目标字段值]
    ↓
实例化 Service 类并执行 execute($data)
    ├─ 1. rules() 定义并校验字段
    ├─ 2. permissions() 声明权限：author_must_belong_to_account
    ├─ 3. validatePermission 自动查询并设置 $this->author
    ├─ 4. 赋值：$this->author->{字段} = $this->data['{字段}']
    └─ 5. $this->author->save() 写入数据库
    ↓
返回更新后的 User 模型
    ↓
Controller 调用 ViewHelper::dtoXxx($user) 重新组装 DTO
    ↓
返回 JSON 响应
    ↓
前端 axios.then() 回调
    ├─ flash 成功提示
    ├─ 更新本地 data（无需刷新页面）
    └─ 退出编辑模式
```

#### 3.5.2 各偏好对应的 Controller 和 Service

| 偏好项 | 路由名称 | Controller | Service |
|--------|---------|-----------|---------|
| 帮助显示开关 | settings.preferences.help.store | [PreferencesHelpController](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesHelpController.php) | [StoreHelpPreference](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreHelpPreference.php) |
| 语言 | settings.preferences.locale.store | [PreferencesLocaleController](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesLocaleController.php) | [StoreLocale](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreLocale.php) |
| 姓名顺序 | settings.preferences.name.store | [PreferencesNameOrderController](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesNameOrderController.php) | [StoreNameOrderPreference](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreNameOrderPreference.php) |
| 日期格式 | settings.preferences.date.store | [PreferencesDateFormatController](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesDateFormatController.php) | [StoreDateFormatPreference](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreDateFormatPreference.php) |
| 数字格式 | settings.preferences.number.store | [PreferencesNumberFormatController](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesNumberFormatController.php) | [StoreNumberFormatPreference](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreNumberFormatPreference.php) |
| 距离单位 | settings.preferences.distance.store | [PreferencesDistanceFormatController](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesDistanceFormatController.php) | [StoreDistanceFormatPreference](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreDistanceFormatPreference.php) |
| 时区 | settings.preferences.timezone.store | [PreferencesTimezoneController](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesTimezoneController.php) | [StoreTimezone](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreTimezone.php) |
| 地图服务 | settings.preferences.maps.store | [PreferencesMapsPreferenceController](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesMapsPreferenceController.php) | [StoreMapsPreference](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreMapsPreference.php) |

#### 3.5.3 Service 写入代码示例（StoreLocale 含特殊逻辑）
[StoreLocale.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreLocale.php)
```php
public function execute(array $data): User
{
    $this->data = $data;
    $this->validateRules($data);
    $this->updateUser();
    return $this->author;
}

private function updateUser(): void
{
    $this->author->locale = $this->data['locale'];
    $this->author->save();
    App::setLocale($this->data['locale']);  // 额外同步当前请求的应用语言
}
```

#### 3.5.4 前端写入代码示例（DateFormat.vue）
[DateFormat.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/resources/js/Pages/Settings/Preferences/Partials/DateFormat.vue#L119-L141)
```javascript
methods: {
    submit() {
        this.loadingState = 'loading';
        axios
            .post(this.data.url.store, this.form)
            .then((response) => {
                this.flash(this.$t('Changes saved'), 'success');
                this.localDateFormat = this.form.dateFormat;
                this.localHumanDateFormat = response.data.data.human_date_format;
                this.editMode = false;
                this.loadingState = null;
            })
            .catch((error) => {
                this.loadingState = null;
                this.form.errors = error.response.data;
            });
    },
}
```

---

## 四、三阶段总结对比

| 维度 | 环境配置（实例级） | 账户数据（Account 级） | 用户数据（User 偏好） |
|------|-------------------|----------------------|---------------------|
| **存储介质** | `.env` + `config/*.php` | 数据库（accounts 表 + 关联表） | 数据库 users 表字段 |
| **读取入口** | `config('monica.xxx')` | `Auth::user()->account` 或 `Account::find()` | `Auth::user()->{field}` |
| **运行时缓存** | Laravel 配置缓存（可选）+ 服务容器内存 | Eloquent 同请求对象缓存 + 部分 Helper 的 array 缓存（5s） | Eloquent 同请求对象缓存 |
| **跨请求缓存** | 有（config:cache） | 无（storage_limit_in_mb）/ 视业务而定 | 无 |
| **界面写回** | ❌ 不支持 | ✅ 关联数据支持（需账户管理员）；❌ storage_limit_in_mb 不支持 | ✅ 完全支持（用户本人操作） |
| **写回 Service** | N/A | CreateGender / UpdateGender / DestroyGender 等 CRUD 服务 | Store{Preference} 系列服务（仅更新） |
| **权限控制** | N/A | `author_must_be_account_administrator` | `author_must_belong_to_account` |

---

## 五、关键代码索引

### 配置层
- [config/monica.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/config/monica.php) - 实例级配置定义
- [config/cache.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/config/cache.php) - 缓存驱动配置

### 环境配置使用
- [app/Helpers/SignupHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/SignupHelper.php) - 读取 monica.disable_signup
- [app/Helpers/StorageHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/StorageHelper.php) - 读取 services.uploadcare
- [app/Http/Middleware/HandleInertiaRequests.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Http/Middleware/HandleInertiaRequests.php) - 共享帮助链接、版本号等
- [app/Providers/AppServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Providers/AppServiceProvider.php) - 强制 URL、Gate 定义等

### 账户数据
- [app/Models/Account.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/Account.php) - Account 模型
- [app/Domains/Settings/ManageStorage/Web/Controllers/AccountStorageController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageStorage/Web/Controllers/AccountStorageController.php) - 存储页面入口
- [app/Domains/Settings/ManageStorage/Web/ViewHelpers/StorageIndexViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageStorage/Web/ViewHelpers/StorageIndexViewHelper.php) - 存储数据组装
- [app/Domains/Settings/CreateAccount/Services/CreateAccount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/CreateAccount/Services/CreateAccount.php) - 创建账户（读取默认存储限制）
- [app/Domains/Settings/CreateAccount/Jobs/SetupAccount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/CreateAccount/Jobs/SetupAccount.php) - 初始化账户默认关联数据

### 缓存 Helper
- [app/Helpers/VaultHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/VaultHelper.php) - Vault 权限缓存
- [app/Helpers/UserHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/UserHelper.php) - 用户关联联系人缓存
- [app/Helpers/ContactImportantDateHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/ContactImportantDateHelper.php) - 重要日期类型缓存

### 服务基类
- [app/Services/BaseService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Services/BaseService.php) - 统一验证与权限框架

### 用户偏好设置（读取）
- [app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesController.php)
- [app/Domains/Settings/ManageUserPreferences/Web/ViewHelpers/UserPreferencesIndexViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/ViewHelpers/UserPreferencesIndexViewHelper.php)

### 用户偏好设置（写入 - Service 层）
- [app/Domains/Settings/ManageUserPreferences/Services/StoreDateFormatPreference.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreDateFormatPreference.php)
- [app/Domains/Settings/ManageUserPreferences/Services/StoreLocale.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreLocale.php)
- [app/Domains/Settings/ManageUserPreferences/Services/StoreTimezone.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreTimezone.php)

### 前端组件
- [resources/js/Pages/Settings/Index.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/resources/js/Pages/Settings/Index.vue) - 设置首页
- [resources/js/Pages/Settings/Preferences/Index.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/resources/js/Pages/Settings/Preferences/Index.vue) - 用户偏好页
- [resources/js/Pages/Settings/Preferences/Partials/DateFormat.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/resources/js/Pages/Settings/Preferences/Partials/DateFormat.vue) - 日期格式编辑组件
- [resources/js/Pages/Settings/Storage/Index.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/resources/js/Pages/Settings/Storage/Index.vue) - 存储统计页（只读）
