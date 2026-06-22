# Timezone 与 Date Format 偏好传递流程

## 1. 数据存储层：User 模型

**文件**: [User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Models/User.php)

`users` 表有两个关键字段（[migration](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/database/migrations/2014_10_12_000000_create_users_table.php#L26-L27)）：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `date_format` | string | `'MMM DD, YYYY'` | 用户偏好的日期显示格式 |
| `timezone` | string (nullable) | `null` | 用户时区，如 `America/New_York` |

两个字段均在 `$fillable` 中（允许批量赋值），`timezone` 还在 `$visible` 中（序列化时可见）。

创建账户时默认值来自 [CreateAccount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/CreateAccount/Services/CreateAccount.php#L62)：
- `timezone` => `'UTC'`
- `date_format` 使用数据库默认值 `'MMM DD, YYYY'`

---

## 2. 偏好设置保存流程

### 2.1 路由定义

**文件**: [web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/routes/web.php#L554-L562)

```
POST /settings/preferences/date      → PreferencesDateFormatController@store
POST /settings/preferences/timezone  → PreferencesTimezoneController@store
```

### 2.2 Timezone 保存

**Controller**: [PreferencesTimezoneController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesTimezoneController.php)

```
请求 → 取 timezone 值 → StoreTimezone 服务 → 直接写入 user.timezone 字段
```

**Service**: [StoreTimezone.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreTimezone.php)
- 验证规则：`timezone` 为 required|string|max:255
- 执行：`$this->author->timezone = $this->data['timezone']; $this->author->save();`
- 返回 User 模型，Controller 再用 `UserPreferencesIndexViewHelper::dtoTimezone($user)` 包装为 JSON 返回

### 2.3 Date Format 保存

**Controller**: [PreferencesDateFormatController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesDateFormatController.php)

```
请求 → 取 dateFormat 值 → StoreDateFormatPreference 服务 → 直接写入 user.date_format 字段
```

**Service**: [StoreDateFormatPreference.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreDateFormatPreference.php)
- 验证规则：`date_format` 为 required|string|max:255
- 执行：`$this->author->date_format = $this->data['date_format']; $this->author->save();`

**注意**：前端表单字段名是 `dateFormat`（camelCase），在 Controller 中映射为 `date_format` 传给 Service。

---

## 3. 偏好传到页面：后端 → 前端

### 3.1 全局共享属性（HandleInertiaRequests）

**文件**: [HandleInertiaRequests.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Http/Middleware/HandleInertiaRequests.php)

该中间件通过 `parent::share($request)` 调用 Laravel 惯例的 Inertia 共享属性。默认情况下，Inertia 会自动共享 `auth.user` 属性。由于 User 模型设置了 `$visible = [..., 'timezone']`，**`timezone` 会被包含在 `$page.props.auth.user.timezone`** 中，但 `date_format` 不在 `$visible` 中，**不会**通过全局 auth 共享到前端。

这意味着：
- 前端全局可访问：`$page.props.auth.user.timezone`
- 前端全局**不可**访问：`$page.props.auth.user.date_format`

### 3.2 页面级传递（ViewHelper 模式）

Monica 采用 **ViewHelper 模式** 将数据从 Controller 传到页面。每个页面 Controller 调用对应的 ViewHelper，ViewHelper 内部使用 `Auth::user()` 获取当前用户及其偏好。

**典型流程（以 Vault Dashboard 为例）**：

```
VaultController::show()
  → Auth::user()                        // 获取当前登录用户
  → VaultShowViewHelper::upcomingReminders($vault, Auth::user())
  → VaultShowViewHelper::dueTasks($vault, Auth::user())
  → VaultShowViewHelper::moodTrackingEvents($vault, Auth::user())
  → Inertia::render('Vault/Dashboard/Index', [...])
```

**文件**: [VaultController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L66-L89)

### 3.3 偏好设置页面

**Controller**: [PreferencesController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesController.php)

```php
return Inertia::render('Settings/Preferences/Index', [
    'layoutData' => VaultIndexViewHelper::layoutData(),
    'data' => UserPreferencesIndexViewHelper::data(Auth::user()),
]);
```

**ViewHelper**: [UserPreferencesIndexViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Settings/ManageUserPreferences/Web/ViewHelpers/UserPreferencesIndexViewHelper.php)

`data()` 方法返回的 `date_format` 数据结构：
```php
'date_format' => [
    'dates' => [         // 可选的4种格式，每项含 id/format/value
        ['id' => 1, 'format' => 'MMM DD, YYYY',  'value' => '实际格式化后的当前日期'],
        ['id' => 2, 'format' => 'DD MMM YYYY',   'value' => ...],
        ['id' => 3, 'format' => 'YYYY/MM/DD',    'value' => ...],
        ['id' => 4, 'format' => 'DD/MM/YYYY',    'value' => ...],
    ],
    'date_format' => $user->date_format,           // 用户当前选择的格式字符串
    'human_date_format' => Carbon::now()->isoFormat($user->date_format),  // 格式化后的可读预览
    'url' => ['store' => route('settings.preferences.date.store')],
]
```

`data()` 方法返回的 `timezone` 数据结构：
```php
'timezone' => [
    'timezone' => $user->timezone,    // 用户当前时区字符串，如 'UTC'
    'url' => ['store' => route('settings.preferences.timezone.store')],
]
```

---

## 4. 前端组件消费

### 4.1 Preferences 页面

**文件**: [Index.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Pages/Settings/Preferences/Index.vue)

```vue
<date-format :data="data.date_format" />
<timezone :data="data.timezone" />
```

#### DateFormat.vue

**文件**: [DateFormat.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Pages/Settings/Preferences/Partials/DateFormat.vue)

- `mounted()` 时读取 `this.data.date_format` 和 `this.data.human_date_format` 到本地变量
- 编辑模式：用 `v-model="form.dateFormat"` 绑定 radio 值为 `date.format`（即 `MMM DD, YYYY` 等格式字符串）
- 提交：`axios.post(this.data.url.store, this.form)`，表单字段名为 `dateFormat`
- 保存成功后：`this.localHumanDateFormat = response.data.data.human_date_format`（从后端返回的 DTO 更新）

#### Timezone.vue

**文件**: [Timezone.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/resources/js/Pages/Settings/Preferences/Partials/Timezone.vue)

- `mounted()` 时读取 `this.data.timezone` 到本地变量
- 编辑模式：用硬编码的 `<select>` 列表（涵盖全球时区），`v-model="form.timezone"`
- 提交：`axios.post(this.data.url.store, this.form)`，表单字段名为 `timezone`
- 保存成功后：`this.localTimezone = response.data.data.timezone`

### 4.2 其他页面：日期直接以后端格式化好的字符串传到前端

在业务页面中，日期**不需要前端做任何格式化**。ViewHelper 在后端已经把日期格式化为字符串，前端直接渲染即可。

例如 Vault Dashboard 的 reminders：
```
后端: DateHelper::format($scheduledAtDate, $user) → "Jun 22, 2026"
前端: <span>{{ reminder.scheduled_at }}</span>  直接显示
```

---

## 5. 日期格式化规则详解

### 5.1 DateHelper（核心格式化工具）

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

### 5.2 ImportantDateHelper（特殊日期格式化）

**文件**: [ImportantDateHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Helpers/ImportantDateHelper.php)

用于 `ContactImportantDate` 的格式化，逻辑根据日期类型分三种：

| 日期类型 | 条件 | 格式化方式 |
|---------|------|-----------|
| `TYPE_FULL_DATE` (fullDate) | day+month+year 都有 | `Carbon::parse(y-m-d)->isoFormat($user->date_format)` |
| `TYPE_YEAR` (year) | 仅 year | 直接返回 `$date->year` 数字 |
| `TYPE_MONTH_DAY` (monthDay) | day+month 但无 year | 根据 `$user->date_format` 匹性选择短格式 |

对于 monthDay 类型的弹性映射：
```
'MMM DD, YYYY' → 'MMM DD'   (如 "Jul 29")
'DD MMM YYYY' → 'DD MMM'    (如 "29 Jul")
'YYYY/MM/DD'  → 'MM/DD'     (如 "07/29")
default       → 'DD/MM'     (如 "29/07")
```

---

## 6. 时区的实际应用位置

时区 `$user->timezone` 在 ViewHelper 中只在极少数场景被直接使用：

| 位置 | 代码 | 用途 |
|------|------|------|
| [VaultShowViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L182) | `Carbon::now($user->timezone)->format('Y-m-d')` | 获取 mood tracking 的当前日期 |
| [ModuleLifeEventViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/78-monica/app/Domains/Contact/ManageLifeEvents/Web/ViewHelpers/ModuleLifeEventViewHelper.php#L28-L29) | `Carbon::now($user->timezone)->format('Y-m-d')` 和 `DateHelper::format(Carbon::now($user->timezone), $user)` | Life event 当前日期和可读格式 |

**注意**：在这两处，`Carbon::now($user->timezone)` 用时区创建了 Carbon 实例，然后再传给 `DateHelper::format()`，这样日期值本身就是用户时区的了，所以 `format()` 不需要再额外做时区转换。但这是一个**局部解决方案**——只对 "当前时间" 起作用，对数据库中存储的历史日期不适用。

### 6.1 时区不生效的场景

`DateHelper::format(Carbon $date, User $user)` 是项目中最常用的日期格式化方法（约 30+ 处调用），它接收的 `$date` 参数来自数据库（如 `$task->due_at`、`$note->created_at`），这些日期在数据库中通常以 UTC 存储。但 `format()` 方法只做了 `isoFormat($user->date_format)`，**没有**做 `$date->setTimezone($user->timezone)`，这意味着**用户设置的 timezone 偏好对大部分日期显示没有实际影响**。

唯一做了时区转换的 `formatDate()` 和 `formatShortDateWithTime()` 方法接受 `?string $timezone` 参数，但它们使用 `trans('format.date')`（locale 格式）而非 `$user->date_format`（用户偏好格式），且只有少数地方调用时传了 timezone。

---

## 7. i18n 格式文件

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

## 8. 完整数据流总结

```
┌─────────────────────────────────────────────────────────────────────┐
│  数据库 users 表                                                      │
│  timezone: "America/New_York" (nullable, 默认 null)                  │
│  date_format: "MMM DD, YYYY" (默认 "MMM DD, YYYY")                   │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
   ┌──────────────┐         ┌──────────────┐
   │ 全局共享属性   │         │ 页面级传递     │
   │ (Inertia)    │         │ (ViewHelper) │
   └──────┬───────┘         └──────┬───────┘
          │                        │
          │ $page.props.auth.user  │ ViewHelper::method($vault, Auth::user())
          │   .timezone ✓          │   → DateHelper::format($date, $user)
          │   .date_format ✗       │     → $user->date_format (isoFormat)
          │ (不在 $visible 中)      │   → DateHelper::formatDate($date, $tz)
          │                        │     → trans('format.date') + setTimezone
          ▼                        ▼
   ┌──────────────────────────────────────────┐
   │  前端 Vue 页面                             │
   │                                          │
   │  1. Preferences 页面:                     │
   │     - Timezone.vue: 设置/显示时区           │
   │     - DateFormat.vue: 设置/显示日期格式      │
   │                                          │
   │  2. 业务页面:                              │
   │     - 日期已是格式化字符串，直接渲染            │
   │     - 前端不做日期格式化                     │
   └──────────────────────────────────────────┘
```

---

## 9. 潜在问题

1. **`DateHelper::format()` 未应用时区**：方法注释说 "according to the timezone of the user"，但代码实际只用 `$user->date_format` 做 `isoFormat()`，没有 `$date->setTimezone($user->timezone)`。数据库中存储的 UTC 时间不会自动转换为用户时区。

2. **`date_format` 不在 User `$visible` 中**：全局 Inertia 共享属性不包含 `date_format`，前端无法通过 `$page.props.auth.user.date_format` 获取用户偏好格式。如果前端需要做本地格式化，目前没有现成的获取路径。

3. **Timezone.vue 的时区列表硬编码**：所有时区选项直接写在模板中，不是动态生成。如需更新时区列表，需手动修改 Vue 文件。

4. **`formatDate()` 与 `format()` 的格式来源不同**：`formatDate()` 用 locale 翻译字符串 `trans('format.date')`，`format()` 用用户偏好 `$user->date_format`。两者可能产生不同的格式输出。
