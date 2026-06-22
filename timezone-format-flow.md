# Timezone 与 Date Format 偏好传递流程

## 1. 数据存储层：User 模型

**文件**: [User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Models/User.php)

`users` 表有两个关键字段（[migration](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/database/migrations/2014_10_12_000000_create_users_table.php#L26-L27)）：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `date_format` | string | `'MMM DD, YYYY'` | 用户偏好的日期显示格式 |
| `timezone` | string (nullable) | `null` | 用户时区，如 `America/New_York` |

User 模型对序列化的控制（决定了哪些字段能出现在 JSON / Inertia 共享属性中）：

```php
// User.php#L122-L132
protected $visible = [
    'name', 'first_name', 'last_name', 'email',
    'help_shown', 'locale', 'locale_ietf',
    'is_account_administrator', 'timezone',   // ← timezone 在此
    // ⚠️ date_format 不在 $visible 中！
];

protected $appends = [
    'name',        // accessor: first_name + ' ' + last_name
    'locale_ietf', // accessor: Str::replace('_', '-', $locale)
];
```

**关键差异**：`timezone` 在 `$visible` 中，`date_format` 不在。这决定了两者在 Inertia 全局共享属性中的可见性完全不同。

创建账户时默认值来自 [CreateAccount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/CreateAccount/Services/CreateAccount.php#L62)：
- `timezone` => `'UTC'`
- `date_format` 使用数据库默认值 `'MMM DD, YYYY'`

---

## 2. Inertia 全局共享属性的真实来源

### 2.1 两个 JetstreamServiceProvider 的区别

项目中涉及两个不同的 JetstreamServiceProvider，容易混淆：

| 提供者 | 命名空间 | 位置 | 注册方式 | 职责 |
|--------|---------|------|---------|------|
| **项目自身的** JetstreamServiceProvider | `App\Providers\JetstreamServiceProvider` | [app/Providers/JetstreamServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Providers/JetstreamServiceProvider.php) | [bootstrap/providers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/bootstrap/providers.php#L8) 显式注册 | 配置权限、whenRendering 回调等项目自定义逻辑 |
| **Jetstream 包自带的** JetstreamServiceProvider | `Laravel\Jetstream\JetstreamServiceProvider` | vendor/laravel/jetstream 目录 | Composer 包自动发现（Package Discovery） | 注册路由、中间件、共享 Inertia 数据等核心功能 |

> **重要**：下文提到"Jetstream 包"时，指的是 `laravel/jetstream` Composer 包及其 `Laravel\Jetstream\JetstreamServiceProvider`，而非项目自身的 `App\Providers\JetstreamServiceProvider`。

### 2.2 两条共享路径：Jetstream 包 + HandleInertiaRequests

Monica 的 Inertia 全局共享数据来自**两条独立路径**：

```
HTTP 请求
  │
  ├─ 路径①：Jetstream 包的 Inertia 数据共享
  │    └─ 共享：auth.user, jetstream 等
  │
  └─ 路径②：App\Http\Middleware\HandleInertiaRequests
       └─ parent::share() → ['errors' => ...]
       └─ 自定义共享：help_links, help_url, footer, hasKey, ziggy, sentry
```

#### 路径①：Jetstream 包的 auth.user 共享

**来源**：`laravel/jetstream` 包（v5.x）的 Inertia 栈。

**可验证的证据**（项目代码中可直接确认）：

1. **依赖存在**：[composer.json](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/composer.json#L27) 声明 `"laravel/jetstream": "^5.0"`
2. **配置为 Inertia 栈**：[config/jetstream.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/config/jetstream.php#L19) 中 `'stack' => 'inertia'`
3. **项目使用 Jetstream Inertia API**：项目的 [JetstreamServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Providers/JetstreamServiceProvider.php#L33) 调用 `Jetstream::inertia()->whenRendering(...)`
4. **前端能访问 auth.user**：多个 Vue 组件读取 `$page.props.auth.user`
5. **前端能访问 jetstream**：[Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Pages/Vault/Contact/Show.vue#L139) 读取 `$page.props.jetstream.flash.filename`
6. **HandleInertiaRequests 不共享 auth**：[HandleInertiaRequests.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Http/Middleware/HandleInertiaRequests.php#L25-L54) 的 `share()` 方法中没有 `auth` 键

**共享机制**：Jetstream 包的 ServiceProvider 在 boot 阶段通过 `Inertia::share()` 方法注册共享数据回调。当 Inertia 渲染页面时，这些回调被执行，将 `auth.user`、`jetstream` 等数据合并到共享 props 中。

> 注：Jetstream 包的具体实现位于 vendor 目录中（本环境未安装 vendor，无法直接查看源码），以上基于 Jetstream Inertia 栈的标准行为和项目代码中的证据推断。

#### auth.user 的数据结构与 $visible 的关系

Jetstream 共享 `auth.user` 时，基础数据来自 `$user->toArray()`，而 `toArray()` 受 User 模型的 `$visible` 属性控制。Jetstream 还会追加一些额外字段（如 `two_factor_enabled`、`profile_photo_url` 等）。

| 字段 | 在 auth.user 中 | 原因 |
|------|:--------------:|------|
| `name` | ✅ | `$visible` + `$appends` (accessor) |
| `first_name` | ✅ | `$visible` |
| `last_name` | ✅ | `$visible` |
| `email` | ✅ | `$visible` |
| `help_shown` | ✅ | `$visible` |
| `locale` | ✅ | `$visible` |
| `locale_ietf` | ✅ | `$visible` + `$appends` (accessor) |
| `is_account_administrator` | ✅ | `$visible` |
| **timezone** | **✅** | **`$visible` 中显式声明** |
| **date_format** | **❌** | **不在 `$visible` 中** |
| `two_factor_enabled` | ✅ | Jetstream 追加（非 User 模型属性） |
| `profile_photo_url` | ✅ | Jetstream 追加（accessor） |
| `password` | ❌ | `$hidden` 中排除 |

#### 路径②：HandleInertiaRequests 中间件

**文件**: [HandleInertiaRequests.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Http/Middleware/HandleInertiaRequests.php)

**注册方式**：[bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/bootstrap/app.php#L38) 通过 `$middleware->web(append: [...])` 追加到 web 中间件组。

```php
public function share(Request $request)
{
    $this->storeCurrentUrl($request);

    return [
        ...parent::share($request),   // ← Inertia v2 Middleware: 只含 ['errors' => ...]
        'help_links' => fn () => config('monica.help_links'),
        'help_url' => fn () => config('monica.help_center_url'),
        'footer' => fn () => $this->footer(),
        'hasKey' => fn () => function () use ($request) { ... },
        'ziggy' => fn () => [...],
        'sentry' => fn () => [...],
        // ⚠️ 注意：这里没有 'auth' 键！auth 由 Jetstream 包共享
    ];
}
```

**关键纠正**：`parent::share()` 即 Inertia v2 `Middleware` 的 `share()` 方法，只返回 `['errors' => ...]`，**不包含 auth.user**。auth.user 完全由 Jetstream 包的共享机制提供。

### 2.3 前端实际使用的 auth.user 字段

搜索所有 Vue 文件中 `$page.props.auth.user` 的引用（按出现次数排序）：

| 字段 | 使用位置数 | 用途 |
|------|:----------:|------|
| `locale_ietf` | 9 处 | DatePicker 组件的 locale 设置（日历显示语言） |
| `name` | 3 处 | AppLayout 中显示用户名 |
| `help_shown` | 2 处 | Help.vue 等控制帮助提示显示 |
| `email` | 2 处 | AppLayout 中显示邮箱 |
| `profile_photo_url` | 2 处 | AppLayout 中显示头像 |
| `instance_administrator` | 2 处 | AppLayout 中管理员入口 |
| `two_factor_enabled` | 1 处 | TwoFactorAuthenticationForm.vue |

**关键发现**：
- ✅ `$page.props.auth.user.timezone` 存在（因为 timezone 在 `$visible` 中），但**没有任何 Vue 组件读取它**
- ❌ `$page.props.auth.user.date_format` 不存在（不在 `$visible` 中）

---

## 3. 偏好设置的真实数据路径

### 3.1 概览：三条完全独立的路径

```
                    ┌──────────────────────────────┐
                    │  数据库 users 表               │
                    │  timezone: "America/New_York" │
                    │  date_format: "MMM DD, YYYY"  │
                    └──────────────┬───────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                     ▼
   ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐
   │ 路径A: Jetstream│  │ 路径B: ViewHelper │  │ 路径C: ViewHelper │
   │ 全局共享 auth    │  │ 偏好设置页面       │  │ 业务页面日期格式化 │
   │ .user           │  │                  │  │                  │
   └────────┬────────┘  └────────┬─────────┘  └────────┬─────────┘
            │                     │                      │
            ▼                     ▼                      ▼
   $page.props.auth.user   props.data.timezone    props.xxx.scheduled_at
     .timezone ✅ 存在      props.data.date_format   = "Jun 22, 2026"
     .date_format ❌ 缺失    = "MMM DD, YYYY"        (后端已格式化)
     但从未被读取           (ViewHelper 直接读        前端直接渲染
                          $user->timezone
                          和 $user->date_format)
```

### 3.2 路径A：Jetstream 全局共享（auth.user）

```
Jetstream 包 ServiceProvider boot()
  → Inertia::share('auth.user', function () use ($request) {
      $user = $request->user();
      return array_merge($user->toArray(), [/* Jetstream 追加字段 */]);
    })
  → 前端: $page.props.auth.user
      .timezone      ✅ 存在（$visible 中），但从未被读取
      .date_format   ❌ 不存在（不在 $visible 中）
      .locale_ietf   ✅ 存在，被 DatePicker 大量使用
```

**特点**：
- 数据受 User 模型 `$visible` / `$hidden` / `$appends` 控制
- 是"全局"的——每个页面都能访问
- timezone 和 date_format 在此路径上的可见性不一致

### 3.3 路径B：偏好设置页面（ViewHelper → page props）

```
PreferencesController::index()
  → Auth::user()                               // 直接获取 User 模型对象
  → UserPreferencesIndexViewHelper::data($user) // 绕过 $visible 限制
      → dtoTimezone($user): ['timezone' => $user->timezone, ...]
      → dtoDateFormat($user): [
            'dates' => [...],                   // 4种可选格式
            'date_format' => $user->date_format, // 用户当前格式
            'human_date_format' => Carbon::now()->isoFormat($user->date_format),
            'url' => [...],
         ]
  → Inertia::render('Settings/Preferences/Index', ['data' => ...])
  → 前端: props.data.timezone     = "America/New_York"
          props.data.date_format   = "MMM DD, YYYY"
          props.data.human_date_format = "Jun 22, 2026"
```

**文件**：
- Controller: [PreferencesController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesController.php)
- ViewHelper: [UserPreferencesIndexViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/ManageUserPreferences/Web/ViewHelpers/UserPreferencesIndexViewHelper.php)

**关键点**：ViewHelper 直接读取 `$user->timezone` 和 `$user->date_format` 属性（PHP 对象属性访问），**不受 `$visible` 限制**。`$visible` 只在序列化为数组/JSON 时才起作用。

### 3.4 路径C：业务页面（ViewHelper → DateHelper → 预格式化字符串）

```
各业务 Controller::method()
  → Auth::user()                               // 直接获取 User 模型对象
  → XxxViewHelper::method($model, Auth::user())// 传入 User 对象
      → DateHelper::format($date, $user)       // 使用 $user->date_format
      → ImportantDateHelper::formatDate($date, $user)
      → Carbon::now($user->timezone)           // 少数场景使用时区
  → Inertia::render('Xxx/Page', ['data' => [...]])
  → 前端: props.xxx.scheduled_at = "Jun 22, 2026"  (已格式化字符串)
          props.xxx.current_date = "2026-06-22"     (已按时区计算)
```

**关键点**：业务页面的前端组件**从不接触** timezone 和 date_format 原始值。所有日期在后端已被格式化为字符串，前端只是 `{{ reminder.scheduled_at }}` 直接渲染。

---

## 4. 偏好设置保存流程

### 4.1 路由定义

**文件**: [web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/routes/web.php#L554-L562)

```
POST /settings/preferences/date      → PreferencesDateFormatController@store
POST /settings/preferences/timezone  → PreferencesTimezoneController@store
```

### 4.2 Timezone 保存

**Controller**: [PreferencesTimezoneController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesTimezoneController.php)

**Service**: [StoreTimezone.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreTimezone.php)

```
前端 axios.post(url, { timezone }) 
  → Controller: $request->input('timezone')
  → StoreTimezone 服务: $this->author->timezone = $data['timezone']; save()
  → 返回 JSON: UserPreferencesIndexViewHelper::dtoTimezone($user)
```

验证规则：`timezone` 为 required|string|max:255

### 4.3 Date Format 保存

**Controller**: [PreferencesDateFormatController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesDateFormatController.php)

**Service**: [StoreDateFormatPreference.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreDateFormatPreference.php)

```
前端 axios.post(url, { dateFormat })
  → Controller: $request->input('dateFormat') → 映射为 'date_format'
  → StoreDateFormatPreference 服务: $this->author->date_format = $data['date_format']; save()
  → 返回 JSON: UserPreferencesIndexViewHelper::dtoDateFormat($user)
```

验证规则：`date_format` 为 required|string|max:255

**注意**：前端表单字段名是 `dateFormat`（camelCase），在 Controller 中手动映射为 `date_format` 传给 Service。

---

## 5. 前端组件消费细节

### 5.1 Preferences 页面

**文件**: [Index.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Pages/Settings/Preferences/Index.vue)

```vue
<date-format :data="data.date_format" />
<timezone :data="data.timezone" />
```

#### DateFormat.vue

**文件**: [DateFormat.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Pages/Settings/Preferences/Partials/DateFormat.vue)

- `mounted()`: 读取 `this.data.date_format` → `localDateFormat`，读取 `this.data.human_date_format` → `localHumanDateFormat`
- 编辑模式：`v-model="form.dateFormat"` 绑定 radio，值为 `date.format`（如 `MMM DD, YYYY`）
- 提交：`axios.post(this.data.url.store, this.form)`，字段名 `dateFormat`
- 保存成功后：`this.localHumanDateFormat = response.data.data.human_date_format`（后端 DTO 返回新的可读预览）

#### Timezone.vue

**文件**: [Timezone.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Pages/Settings/Preferences/Partials/Timezone.vue)

- `mounted()`: 读取 `this.data.timezone` → `localTimezone`
- 编辑模式：硬编码 `<select>` 列表（涵盖全球时区），`v-model="form.timezone"`
- 提交：`axios.post(this.data.url.store, this.form)`，字段名 `timezone`
- 保存成功后：`this.localTimezone = response.data.data.timezone`

### 5.2 业务页面：日期直接以后端格式化好的字符串传到前端

在业务页面中，日期**不需要前端做任何格式化**。ViewHelper 在后端已经把日期格式化为字符串，前端直接渲染即可。

例如 Vault Dashboard 的 reminders：
```
后端: DateHelper::format($scheduledAtDate, $user) → "Jun 22, 2026"
前端: <span>{{ reminder.scheduled_at }}</span>  直接显示
```

### 5.3 前端日期选择器（v-calendar DatePicker）

多个 Vue 组件使用 `v-calendar` 的 `DatePicker` 组件：

```vue
<DatePicker
  :timezone="'UTC'"                          ← 硬编码 UTC
  :locale="$page.props.auth.user?.locale_ietf"  ← 从 auth.user 读取（路径A）
  v-model.string="form.date"
/>
```

**出现位置**（共 9 处）：
- [MoodTrackingEvents.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Pages/Vault/Dashboard/Partials/MoodTrackingEvents.vue#L149-L155)
- [CreateLifeEvent.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Shared/Modules/CreateLifeEvent.vue#L256-L261)
- [Reminders.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Shared/Modules/Reminders.vue)
- [Loans.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Shared/Modules/Loans.vue)
- [Calls.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Shared/Modules/Calls.vue)
- [CreateOrEditTask.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Shared/Modules/TaskItems/CreateOrEditTask.vue)
- [PostEdit.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Pages/Vault/Journal/Post/Edit.vue)
- [CreateOrEditImportantDate.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Pages/Vault/Contact/ImportantDates/Partials/CreateOrEditImportantDate.vue)

**注意**：
- DatePicker 的 `:timezone="'UTC'"` 是**硬编码**的，没有使用用户的 timezone 偏好
- `:locale` 来自 `$page.props.auth.user.locale_ietf`（路径A，Jetstream 全局共享），只影响日历的显示语言，不影响时区

---

## 6. 日期格式化规则详解

### 6.1 DateHelper（核心格式化工具）

**文件**: [DateHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Helpers/DateHelper.php)

DateHelper 提供两类格式化方法：

#### A. 使用用户 `date_format` 偏好的方法

| 方法 | 签名 | 格式化规则 | 是否应用时区 |
|------|------|-----------|-------------|
| `format()` | `(Carbon $date, User $user)` | `$user->date_format` | ❌ 不应用 |
| `formatDate()` | `(Carbon $date, ?string $timezone)` | `trans('format.date')` | ✅ 如果传了 timezone |

#### B. 使用 i18n 格式字符串的方法（locale 相关，与用户 date_format 偏好无关）

| 方法 | 格式来源 | 示例输出 |
|------|---------|---------|
| `formatDate()` | `trans('format.date')` | "Oct 29, 1981" |
| `formatShortDateWithTime()` | `trans('format.short_date_year_time')` | "Oct 29, 1981 07:32 PM" |
| `formatMonthAndDay()` | `trans('format.long_month_day')` | "July 29th" |
| `formatMonthAndYear()` | `trans('format.short_month_year')` | "Jul 2020" |
| `formatLongMonthAndYear()` | `trans('format.long_month_year')` | "September 2020" |
| `formatShortMonthAndDay()` | `trans('format.short_date')` | "Jul 29" |
| `formatShortDay()` | `trans('format.short_day')` | "Mon" |
| `formatDayAndMonthInParenthesis()` | `trans('format.day_month_parenthesis')` | "Monday (Jul 29th)" |
| `formatFullDate()` | `trans('format.full_date')` | "Monday, Jul 29th 2020" |
| `formatDayNumber()` | `trans('format.day_number')` | "03" |
| `formatMonthNumber()` | `trans('format.short_month')` | "Jul" |
| `getTimestamp()` | `config('api.timestamp_format')` | "2026-06-22T12:00:00Z" |

**关键发现**：`format()` 方法只用了 `$user->date_format` 做 isoFormat，但**完全没有应用时区转换**。虽然方法注释写了 "according to the timezone of the user"，但代码实现并未做 `$date->setTimezone($user->timezone)`。

### 6.2 ImportantDateHelper（特殊日期格式化）

**文件**: [ImportantDateHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Helpers/ImportantDateHelper.php)

用于 `ContactImportantDate` 的格式化，逻辑根据日期类型分三种：

| 日期类型 | 条件 | 格式化方式 |
|---------|------|-----------|
| `TYPE_FULL_DATE` (fullDate) | day+month+year 都有 | `Carbon::parse(y-m-d)->isoFormat($user->date_format)` |
| `TYPE_YEAR` (year) | 仅 year | 直接返回 `$date->year` 数字 |
| `TYPE_MONTH_DAY` (monthDay) | day+month 但无 year | 根据 `$user->date_format` 弹性选择短格式 |

对于 monthDay 类型的弹性映射：
```
'MMM DD, YYYY' → 'MMM DD'   (如 "Jul 29")
'DD MMM YYYY' → 'DD MMM'    (如 "29 Jul")
'YYYY/MM/DD'  → 'MM/DD'     (如 "07/29")
default       → 'DD/MM'     (如 "29/07")
```

---

## 7. 时区的实际应用位置

时区 `$user->timezone` 在 ViewHelper 中只在极少数场景被直接使用：

| 位置 | 代码 | 用途 |
|------|------|------|
| [VaultShowViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L182) | `Carbon::now($user->timezone)->format('Y-m-d')` | 获取 mood tracking 的当前日期 |
| [ModuleLifeEventViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Contact/ManageLifeEvents/Web/ViewHelpers/ModuleLifeEventViewHelper.php#L28-L29) | `Carbon::now($user->timezone)->format('Y-m-d')` 和 `DateHelper::format(Carbon::now($user->timezone), $user)` | Life event 当前日期和可读格式 |

**注意**：在这两处，`Carbon::now($user->timezone)` 用时区创建了 Carbon 实例，然后再传给 `DateHelper::format()`，这样日期值本身就是用户时区的了，所以 `format()` 不需要再额外做时区转换。但这是一个**局部解决方案**——只对 "当前时间" 起作用，对数据库中存储的历史日期不适用。

### 7.1 时区不生效的场景

`DateHelper::format(Carbon $date, User $user)` 是项目中最常用的日期格式化方法（约 30+ 处调用），它接收的 `$date` 参数来自数据库（如 `$task->due_at`、`$note->created_at`），这些日期在数据库中通常以 UTC 存储。但 `format()` 方法只做了 `isoFormat($user->date_format)`，**没有**做 `$date->setTimezone($user->timezone)`，这意味着**用户设置的 timezone 偏好对大部分日期显示没有实际影响**。

唯一做了时区转换的 `formatDate()` 和 `formatShortDateWithTime()` 方法接受 `?string $timezone` 参数，但它们使用 `trans('format.date')`（locale 格式）而非 `$user->date_format`（用户偏好格式），且只有少数地方调用时传了 timezone。

---

## 8. i18n 格式文件

**文件**: [lang/en/format.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/lang/en/format.php)

该文件定义了所有 locale 相关的日期格式模板，使用 Carbon 的 ISO 格式语法。不同 locale（如 `zh_CN`、`fr` 等）有各自的 format.php，实现日期显示的本地化。

| 键 | 英文值 | 示例 |
|----|--------|------|
| `date` | `MMM DD, YYYY` | Oct 29, 1981 |
| `short_date_year_time` | `MMM DD, YYYY hh:mm A` | Oct 29, 1981 07:32 PM |
| `short_date` | `MMM DD` | Jul 29 |
| `short_month_year` | `MMM Y` | Jul 2020 |
| `long_month_year` | `MMMM Y` | July 2020 |
| `long_month_day` | `MMMM Do` | July 29th |
| `day_month_parenthesis` | `dddd (MMM Do)` | Monday (Jul 29th) |
| `full_date` | `dddd, MMM Do YYYY` | Monday, Jul 29th 2020 |
| `short_day` | `ddd` | Mon |
| `day_number` | `DD` | 03 |
| `short_month` | `MMM` | Jul |

API 时间戳格式来自 [config/api.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/config/api.php#L24)：
- `timestamp_format` => `'Y-m-d\TH:i:s\Z'`（ISO 8601 UTC 格式）

---

## 9. 完整数据流总结

```
┌─────────────────────────────────────────────────────────────────────┐
│  数据库 users 表                                                      │
│  timezone: "America/New_York" (nullable, 默认 null)                  │
│  date_format: "MMM DD, YYYY" (默认 "MMM DD, YYYY")                   │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
       ┌───────────────┼───────────────────────┐
       ▼               ▼                        ▼
┌──────────────┐ ┌──────────────┐      ┌──────────────┐
│ 路径A         │ │ 路径B         │      │ 路径C         │
│ Jetstream 包  │ │ ViewHelper    │      │ ViewHelper    │
│ 全局共享 auth │ │ 偏好设置页面   │      │ 业务页面       │
│ .user         │ │              │      │              │
└──────┬───────┘ └──────┬───────┘      └──────┬───────┘
       │                │                      │
       │ $user->toArray()│ 直接读 $user->xxx     │ DateHelper::format()
       │ 受 $visible 控制 │ 不受 $visible 限制    │ + Carbon::now($tz)
       │                │                      │
       ▼                ▼                      ▼
  $page.props.auth   props.data.timezone    props.xxx.scheduled_at
    .user:           props.data.date_format   = "Jun 22, 2026"
      timezone ✅    = "America/New_York"      (预格式化字符串)
      date_format ❌  = "MMM DD, YYYY"         前端直接渲染
      locale_ietf ✅  human_date_format
      但 timezone     = "Jun 22, 2026"
      从未被读取
```

---

## 10. 关键不一致与潜在问题

### 10.1 Inertia 共享与页面实际使用的脱节

| 问题 | 说明 |
|------|------|
| **timezone 在 auth.user 中但从未被读取** | `$page.props.auth.user.timezone` 存在（在 `$visible` 中），但所有 Vue 组件都通过 ViewHelper 的 `props.data.timezone` 获取偏好值，没有任何组件从 auth.user 读取 timezone |
| **date_format 不在 auth.user 中** | 由于不在 `$visible` 中，`$page.props.auth.user.date_format` 不存在。偏好设置页面通过 ViewHelper 的 `props.data.date_format` 获取 |
| **两者走完全不同的路径** | auth.user 由 Jetstream 包通过 `$user->toArray()` 序列化（受 `$visible` 控制）；偏好值由 ViewHelper 通过 PHP 对象属性直接读取（不受 `$visible` 限制） |
| **$visible 设计意图不明确** | `timezone` 被放入 `$visible` 但从未通过 auth.user 被消费；`date_format` 不在 `$visible` 中但通过 ViewHelper 被大量使用。两者在序列化策略上的不一致没有功能上的理由 |

### 10.2 其他问题

1. **`DateHelper::format()` 未应用时区**：方法注释说 "according to the timezone of the user"，但代码实际只用 `$user->date_format` 做 `isoFormat()`，没有 `$date->setTimezone($user->timezone)`。数据库中存储的 UTC 时间不会自动转换为用户时区。

2. **DatePicker 硬编码 UTC**：前端所有 DatePicker 组件的 `:timezone="'UTC'"` 是硬编码的，没有使用用户设置的 timezone 偏好。这意味着用户在 DatePicker 中选择的日期不受时区影响，但后端存储和显示的日期也未做时区转换。

3. **Timezone.vue 的时区列表硬编码**：所有时区选项直接写在模板中，不是动态生成。如需更新时区列表，需手动修改 Vue 文件。

4. **`formatDate()` 与 `format()` 的格式来源不同**：`formatDate()` 用 locale 翻译字符串 `trans('format.date')`，`format()` 用用户偏好 `$user->date_format`。两者可能产生不同的格式输出。

5. **两条路径的数据一致性风险**：偏好保存后，ViewHelper 路径立即反映新值（因为保存后返回新的 DTO），但 auth.user 路径（Jetstream 全局共享）在下一次页面刷新前可能仍为旧值。不过由于前端从未从 auth.user 读取 timezone/date_format，这个问题目前不影响功能。
