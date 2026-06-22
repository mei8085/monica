# Localization Resource Fallback 取值路径分析

本文档详细说明 Monica 项目中翻译资源缺失时的 fallback 处理流程，从代码层到界面层的完整取值路径。

## 一、整体架构概览

Monica 的本地化系统由三层组成：

| 层级 | 技术栈 | 主要职责 |
|------|--------|----------|
| 语言检测层 | `asbiin/laravel-localizer` 中间件 | 确定当前请求的语言环境 |
| 后端翻译层 | Laravel 内置翻译系统 + 模型 Accessor | 处理 PHP 代码中的翻译 |
| 前端翻译层 | `laravel-vue-i18n` Vue 插件 | 处理 Vue 组件中的翻译 |

---

## 二、语言环境检测流程 (Locale Detection)

### 2.1 中间件注册
在 [bootstrap/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/bootstrap/app.php#L35-L40) 中，`web` 中间件组注册了 `SetLocale` 中间件：

```php
$middleware->web(append: [
    \CodeZero\Localizer\Middleware\SetLocale::class,
    // ...
]);
```

### 2.2 语言检测器顺序
在 [config/localizer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L26-L35) 中定义了检测顺序，按优先级从高到低：

```
RouteActionDetector → UrlDetector → OmittedLocaleDetector → UserDetector 
→ SessionDetector → CookieDetector → BrowserDetector → AppDetector
```

| 检测器 | 说明 | 代码位置 |
|--------|------|----------|
| RouteActionDetector | 从路由动作中获取 locale | [localizer.php:27](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L27) |
| UrlDetector | 从 URL 路径段获取 locale (段索引=1) | [localizer.php:28](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L28) |
| OmittedLocaleDetector | 默认省略的 locale | [localizer.php:29](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L29) |
| UserDetector | 从用户模型 `locale` 属性获取 | [localizer.php:30](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L30) |
| SessionDetector | 从 Session 中读取 | [localizer.php:31](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L31) |
| CookieDetector | 从 Cookie 中读取 | [localizer.php:32](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L32) |
| BrowserDetector | 从 `Accept-Language` 请求头解析 | [localizer.php:33](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L33) |
| AppDetector | 使用配置文件中的默认值 | [localizer.php:34](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L34) |

### 2.3 支持的语言列表
[config/localizer.php:9-12](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L9-L12) 定义了支持的语言，**英语必须是第一个**：

```php
'supported_locales' => ['en', 'ar', 'bn', 'ca', 'da', 'de', 'es', 'el',
    'fr', 'he', 'hi', 'it', 'ja', 'ml', 'nl', 'nn', 'pa', 'pl', 'pt',
    'pt_BR', 'ro', 'ru', 'sv', 'te', 'tr', 'ur', 'vi', 'zh_CN', 'zh_TW',
],
```

---

## 三、后端翻译 Fallback 路径 (PHP/Laravel)

### 3.1 核心配置
在 [config/app.php](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/app.php#L85-L87) 中定义了默认语言和 fallback 语言：

```php
'locale' => env('APP_LOCALE', 'en'),
'fallback_locale' => env('APP_FALLBACK_LOCALE', 'en'),
```

**重要**：默认 locale 和 fallback_locale 均为 `'en'`。

### 3.2 Laravel 内置翻译 Fallback 流程
当调用 `__('key')` 或 `trans('key')` 时，Laravel 的翻译器按以下路径查找：

```
调用 __($key)
    ↓
查找当前 locale 的翻译文件
    ├─ 格式1 (短键): lang/{locale}/{file}.php → 数组键匹配
    └─ 格式2 (整串键): lang/{locale}.json → JSON 键匹配
    ↓
找到？→ 返回翻译值
    ↓ 未找到
查找 fallback_locale 的翻译文件
    ├─ lang/en/{file}.php
    └─ lang/en.json
    ↓
找到？→ 返回翻译值
    ↓ 未找到
返回 $key 本身
```

### 3.3 翻译函数说明
在 [app/Helpers/helpers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Helpers/helpers.php) 中定义了两个辅助函数：

#### `trans_key()` - 仅返回键名
```php
function trans_key(?string $key = null): ?string
{
    return $key;
}
```
**用途**：用于配置文件中标记翻译键，不实际执行翻译（用于 `monica:localize` 命令提取）。

#### `trans_ignore()` - 不被提取的翻译
```php
function trans_ignore(?string $key = null, array $replace = [], ?string $locale = null): string
{
    return __($key, $replace, $locale);
}
```
**用途**：执行翻译但不被翻译提取命令扫描。

### 3.4 模型层翻译 Fallback (关键路径)

Monica 中大量模型使用 **"用户自定义值 → 翻译键 → Laravel fallback"** 的三级 fallback 模式。

#### 典型实现 - [Religion.php](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/Religion.php#L43-L55)
```php
protected function name(): Attribute
{
    return Attribute::make(
        get: function ($value, $attributes) {
            if (is_null($attributes['name'])) {
                return __($attributes['translation_key']);
            }
            return $attributes['name'];
        },
        set: fn ($value) => $value,
    );
}
```

#### Fallback 链完整路径
```
访问 $model->name
    ↓
检查 attributes['name'] 是否非空
    ↓ 非空 → 返回用户自定义值
    ↓ 为空
调用 __($attributes['translation_key'])
    ↓
进入 Laravel 翻译流程 (见 3.2)
    ├─ 当前 locale 翻译文件查找
    ├─ fallback_locale 翻译文件查找
    └─ 最终返回 translation_key 本身
```

#### 应用此模式的模型列表
| 模型 | 翻译键字段 | 代码位置 |
|------|-----------|----------|
| Religion | `translation_key` | [Religion.php:48](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/Religion.php#L48) |
| Gender | `name_translation_key` | [Gender.php:85](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/Gender.php#L85) |
| RelationshipType | `name_translation_key`, `name_reverse_relationship_translation_key` | [RelationshipType.php:70](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/RelationshipType.php#L70) |
| RelationshipGroupType | `name_translation_key` | [RelationshipGroupType.php:78](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/RelationshipGroupType.php#L78) |
| LifeEventCategory | `label_translation_key` | [LifeEventCategory.php:62](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/LifeEventCategory.php#L62) |
| LifeEventType | `label_translation_key` | [LifeEventType.php:62](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/LifeEventType.php#L62) |
| Module | `name_translation_key` | [Module.php:135](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/Module.php#L135) |
| Template | `name_translation_key` | [Template.php:78](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/Template.php#L78) |
| PostTemplate | `label_translation_key` | [PostTemplate.php:71](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/PostTemplate.php#L71) |
| Emotion | `name_translation_key` | [Emotion.php:59](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/Emotion.php#L59) |
| GroupType | `label_translation_key` | [GroupType.php:61](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/GroupType.php#L61) |
| GiftOccasion | `label_translation_key` | [GiftOccasion.php:50](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/GiftOccasion.php#L50) |
| GiftState | `label_translation_key` | [GiftState.php:50](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/GiftState.php#L50) |
| MoodTrackingParameter | `label_translation_key` | [MoodTrackingParameter.php:62](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/MoodTrackingParameter.php#L62) |
| PetCategory | `name_translation_key` | [PetCategory.php:49](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/PetCategory.php#L49) |
| Pronoun | `name_translation_key` | [Pronoun.php:47](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/Pronoun.php#L47) |
| AddressType | `name_translation_key` | [AddressType.php:50](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/AddressType.php#L50) |
| CallReason | `name_translation_key` | [CallReason.php:39](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/CallReason.php#L39) |
| CallReasonType | `name_translation_key` | [CallReasonType.php:50](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/CallReasonType.php#L50) |
| ContactInformationType | `name_translation_key` | [ContactInformationType.php:48](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/ContactInformationType.php#L48) |
| GroupTypeRole | `label_translation_key` | [GroupTypeRole.php:50](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/GroupTypeRole.php#L50) |
| PostTemplateSection | `label_translation_key` | [PostTemplateSection.php:59](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/PostTemplateSection.php#L59) |
| TemplatePage | `name_translation_key` | [TemplatePage.php:82](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/TemplatePage.php#L82) |
| VaultQuickFactsTemplate | `label_translation_key` | [VaultQuickFactsTemplate.php:50](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Models/VaultQuickFactsTemplate.php#L50) |

---

## 四、前端翻译 Fallback 路径 (Vue)

### 4.1 前端翻译初始化
在 [resources/js/app.js](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/resources/js/app.js#L31-L33) 中初始化 `laravel-vue-i18n` 插件：

```javascript
.use(i18nVue, {
    resolve: (lang) => resolvePageComponent(`../../lang/${lang}.json`, import.meta.glob('../../lang/*.json')),
})
```

### 4.2 前端语言环境确定
1. **后端传递**：Blade 模板 [app.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/resources/views/app.blade.php#L2) 设置 HTML lang 属性：
   ```html
   <html lang="{{ str_replace('_', '-', app()->getLocale()) }}" ...>
   ```

2. **前端读取**：`laravel-vue-i18n` 从 `<html lang="xx">` 标签自动读取当前语言。

3. **Fallback 语言**：插件默认 `fallbackLang = 'en'`。

### 4.3 前端翻译 Fallback 流程
`laravel-vue-i18n` 支持 `fallbackMissingTranslations` 选项（可选，默认行为取决于版本）：

```
调用 $t('key') 或 trans('key')
    ↓
查找当前 locale 的 JSON 文件 (lang/{locale}.json)
    ↓
找到？→ 返回翻译值，进行参数替换
    ↓ 未找到
检查 fallbackMissingTranslations 是否启用
    ↓ 启用
查找 fallbackLang (en) 的 JSON 文件 (lang/en.json)
    ↓
找到？→ 返回翻译值
    ↓ 未找到 / 未启用
返回 $key 本身
```

### 4.4 前端使用方式
| 方式 | 示例 | 位置 |
|------|------|------|
| 模板中 `$t()` | `{{ $t('Save') }}` | Vue 组件 template |
| JS 中 `trans()` | `trans('The task has been created')` | `methods.js` 或组件 script |
| 带参数 | `$t(':count contact|:count contacts', { count: 5 })` | 支持复数形式 |

**代码引用**：
- [resources/js/methods.js](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/resources/js/methods.js#L2) - 导入 `trans` 函数
- [resources/js/Shared/Modules/Tasks.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/resources/js/Shared/Modules/Tasks.vue) - 大量使用 `$t()` 和 `trans()`

---

## 五、完整的翻译缺失 Fallback 链

### 5.1 后端渲染场景 (Blade 模板 / 邮件)

以邮件模板 [reminder.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/resources/views/emails/notifications/reminder.blade.php) 为例：

```
@lang('Hi :name', ['name' => $name])
    ↓
Laravel 翻译器查找
    ├─ 当前 locale: lang/fr.json → "Hi :name" 存在？→ 返回
    └─ 不存在 → fallback: lang/en.json → 存在？→ 返回
        └─ 不存在 → 返回 "Hi :name"
```

### 5.2 API / Inertia 响应场景

```
模型属性访问 $contact->religion->name
    ↓
模型 Accessor 检查
    ├─ name 字段非空？→ 返回用户自定义值
    └─ name 为空 → 调用 __($attributes['translation_key'])
        ↓
        Laravel 翻译器查找
            ├─ 当前 locale: lang/fr.json → "Sikhism" 存在？→ 返回
            └─ 不存在 → fallback: lang/en.json → 存在？→ 返回
                └─ 不存在 → 返回 "Sikhism" (translation_key 本身)
    ↓
通过 Inertia 传递到 Vue 组件
    ↓
界面显示最终值
```

### 5.3 前端直接翻译场景

```
Vue 模板: {{ $t('There are no tasks yet.') }}
    ↓
laravel-vue-i18n 查找
    ├─ 当前 locale: lang/fr.json → 存在？→ 返回翻译
    └─ 不存在 → (如启用 fallbackMissingTranslations) → lang/en.json → 存在？→ 返回
        └─ 不存在 → 返回 "There are no tasks yet."
```

---

## 六、翻译资源文件结构

### 6.1 文件组织
```
lang/
├── en/                    ← 短键格式 (PHP 数组)
│   ├── actions.php
│   ├── auth.php
│   ├── currencies.php
│   ├── format.php
│   ├── http-statuses.php
│   ├── pagination.php
│   ├── passwords.php
│   └── validation.php
├── fr/                    ← 其他语言相同结构
│   └── ...
├── en.json                ← 整串键格式 (JSON)
├── fr.json
└── zh_CN.json
```

### 6.2 两种翻译格式对比
| 格式 | 键示例 | 文件 | 调用方式 |
|------|--------|------|----------|
| 短键 | `format.date` | `lang/en/format.php` | `__('format.date')` |
| 整串键 | `"Save"` | `lang/en.json` | `__('Save')` |

**注意**：`en.json` 中键和值通常相同，作为其他语言翻译的基准。

---

## 七、翻译提取工具

项目使用 `amirami/localizator` 包进行翻译字符串提取，配置在 [config/localizator.php](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizator.php)：

- 扫描目录：`app`, `config`, `resources/js`, `resources/views`
- 扫描模式：`*.php`, `*.vue`, `*.js`
- 提取函数：`__`, `trans`, `trans_choice`, `trans_key`, `@lang`, `$t`, `$tChoice`

---

---

## 八、Locale 持久化：Stores 三件套

当检测器确定当前 locale 后，`asbiin/laravel-localizer` 会通过配置的 `stores` 将 locale 持久化，确保后续请求能保持语言设置。

### 8.1 Stores 配置
在 [config/localizer.php:49-53](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L49-L53) 中定义了三个存储：

```php
'stores' => [
    CodeZero\Localizer\Stores\SessionStore::class,
    CodeZero\Localizer\Stores\CookieStore::class,
    CodeZero\Localizer\Stores\AppStore::class,
],
```

### 8.2 各 Store 的职责与配置

| Store | 作用 | 相关配置 | 代码位置 |
|-------|------|----------|----------|
| **SessionStore** | 将 locale 存入 Session，单次会话有效 | `session_key = 'locale'` | [localizer.php:77](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L77) |
| **CookieStore** | 将 locale 存入 Cookie，长期持久化 | `cookie_name = 'locale'`, `cookie_minutes = 60*24*365` (1年) | [localizer.php:83-89](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L83-L89) |
| **AppStore** | 调用 `App::setLocale()` 设置当前请求运行时 locale | - | 框架内置 |

### 8.3 持久化执行时序
```
检测器确定 locale 后
    ↓
按顺序执行所有 stores
    ├─ SessionStore: session(['locale' => $locale])
    ├─ CookieStore:  Cookie::queue('locale', $locale, $minutes)
    └─ AppStore:    App::setLocale($locale)
    ↓
后续请求
    ↓
检测器按优先级读取
    ├─ SessionDetector: 从 session('locale') 读取
    ├─ CookieDetector:  从 Cookie::get('locale') 读取
    └─ ...其他检测器
```

---

## 九、Trusted Detectors 开关

### 9.1 配置与作用
在 [config/localizer.php:42-44](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/config/localizer.php#L42-L44) 中定义了受信任检测器：

```php
'trusted_detectors' => [
    //
],
```

**关键特性**：
- 当某个检测器被加入 `trusted_detectors`，其返回的 locale 会被**直接使用**，跳过 `supported_locales` 的白名单校验
- 普通检测器返回的 locale 必须在 `supported_locales` 列表中，否则会被忽略并继续下一个检测器
- 当前配置为空数组，表示所有检测器都需要经过白名单校验

### 9.2 安全意义
```
普通检测器流程：
detector->detect() → $locale
    ↓
in_array($locale, $supported_locales)?
    ↓ 是 → 使用
    ↓ 否 → 忽略，尝试下一个检测器

受信任检测器流程：
detector->detect() → $locale
    ↓
★ 跳过白名单校验
    ↓
直接 App::setLocale($locale)
```

---

## 十、用户偏好语言切换：运行时完整链路

### 10.1 后端流程
用户在设置页面切换语言时，完整的后端处理流程：

#### 1. 控制器入口
[PreferencesLocaleController.php:13-26](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesLocaleController.php#L13-L26)

```php
public function store(Request $request)
{
    $data = [
        'account_id' => Auth::user()->account_id,
        'author_id' => Auth::id(),
        'locale' => $request->input('locale'),
    ];

    $user = (new StoreLocale)->execute($data);

    return response()->json([
        'data' => UserPreferencesIndexViewHelper::dtoLocale($user),
    ], 200);
}
```

#### 2. 业务逻辑层
[StoreLocale.php:45-61](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreLocale.php#L45-L61)

```php
public function execute(array $data): User
{
    $this->data = $data;
    $this->validateRules($data);  // locale 必须在 supported_locales 中
    $this->updateUser();
    return $this->author;
}

private function updateUser(): void
{
    $this->author->locale = $this->data['locale'];  // 持久化到用户表
    $this->author->save();
    App::setLocale($this->data['locale']);  // 设置当前请求运行时 locale
}
```

#### 3. DTO 组装
[UserPreferencesIndexViewHelper.php:192-208](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Domains/Settings/ManageUserPreferences/Web/ViewHelpers/UserPreferencesIndexViewHelper.php#L192-L208)

```php
public static function dtoLocale(User $user): array
{
    return [
        'id' => $user->locale,
        'name' => self::language($user->locale),  // 关键：用目标语言翻译语言名
        'dir' => htmldir(),
        'locales' => collect(config('localizer.supported_locales'))
            ->map(fn (string $locale) => [
                'id' => $locale,
                'name' => self::language($locale),
            ])
            ->sortByCollator('name'),
        'url' => [
            'store' => route('settings.preferences.locale.store'),
        ],
    ];
}
```

**特殊技巧 - 语言名称自翻译**：
[UserPreferencesIndexViewHelper.php:210-213](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Domains/Settings/ManageUserPreferences/Web/ViewHelpers/UserPreferencesIndexViewHelper.php#L210-L213)

```php
public static function language(?string $code): string
{
    return $code !== null ? __('auth.lang', [], $code) : '';
}
```

这里 `__('auth.lang', [], $code)` 传递第三个参数 `$code` 作为 locale，**强制以目标语言来翻译语言名称**：
- `__('auth.lang', [], 'fr')` → 读取 `lang/fr/auth.php` 的 `'lang'` 键 → 返回 `'Français'`
- `__('auth.lang', [], 'zh_CN')` → 读取 `lang/zh_CN/auth.php` 的 `'lang'` 键 → 返回 `'中文(中华人民共和国)'`

实现效果：在语言下拉列表中，每个语言名称用**该语言本身**显示。

### 10.2 前端运行时切换
[Locale.vue:4-44](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/resources/js/Pages/Settings/Preferences/Partials/Locale.vue#L4-L44)

```javascript
import { loadLanguageAsync, getActiveLanguage, trans } from 'laravel-vue-i18n';

const submit = () => {
    // ...
    axios.post(props.data.url.store, form.data())
        .then((response) => {
            // ...
            if (getActiveLanguage() !== form.locale) {
                loadLanguageAsync(response.data.data.id);  // ★ 运行时加载新语言包
                document.getRootNode().querySelector('html').setAttribute('dir', response.data.data.dir);
            }
        })
    // ...
};
```

### 10.3 完整切换时序图
```
用户在前端选择语言并提交
    ↓
POST /settings/preferences/locale
    ↓
StoreLocale 服务
    ├─ 验证 locale 在 supported_locales 中
    ├─ 保存到 users.locale 字段
    └─ App::setLocale() 设置当前请求
    ↓
dtoLocale() 组装响应
    ├─ id: 'fr'
    ├─ name: 'Français' (用 fr  locale 翻译得到)
    ├─ dir: 'ltr'
    └─ locales: 所有支持语言列表
    ↓
前端收到响应
    ↓
getActiveLanguage() !== form.locale?
    ↓ 是
    ├─ loadLanguageAsync('fr') → 动态加载 lang/fr.json
    └─ 更新 HTML dir 属性
    ↓
后续页面翻译立即生效（无需刷新）
```

---

## 十一、Inertia Share 未传递 Locale 的事实

### 11.1 验证
检查 [HandleInertiaRequests.php:25-54](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/app/Http/Middleware/HandleInertiaRequests.php#L25-L54) 的 `share()` 方法：

```php
public function share(Request $request)
{
    return [
        ...parent::share($request),
        'help_links' => fn () => config('monica.help_links'),
        'help_url' => fn () => config('monica.help_center_url'),
        'footer' => fn () => $this->footer(),
        'hasKey' => fn () => function () use ($request) { ... },
        'ziggy' => fn () => [ ... ],
        'sentry' => fn () => [ ... ],
    ];
}
```

**确认**：`share()` 方法中**没有传递 `locale`** 给前端。

### 11.2 前端如何获取 Locale
前端通过以下方式间接获取 locale，而不依赖 Inertia share：

1. **初始加载**：从 `<html lang="xx">` 标签读取（由 Blade 模板设置）
   - [app.blade.php:2](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/resources/views/app.blade.php#L2)
   - `laravel-vue-i18n` 插件自动读取此属性

2. **运行时切换**：通过 `loadLanguageAsync(locale)` 主动设置（用户切换语言时）
   - [Locale.vue:36](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/resources/js/Pages/Settings/Preferences/Partials/Locale.vue#L36)

### 11.3 影响分析
- ✅ **不影响正常翻译**：翻译由后端处理或前端 JSON 文件提供
- ✅ **不影响语言切换**：切换通过 `loadLanguageAsync` 完成
- ⚠️ **前端无法直接感知当前 locale**：如需在 JS 逻辑中判断当前语言，需使用 `getActiveLanguage()`
- ⚠️ **SSR 场景需额外处理**：服务端渲染时需显式传递 locale 给前端

---

## 十二、前端 i18n 插件默认值详解

### 12.1 实际初始化配置
[resources/js/app.js:31-33](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/resources/js/app.js#L31-L33)

```javascript
.use(i18nVue, {
    resolve: (lang) => resolvePageComponent(`../../lang/${lang}.json`, import.meta.glob('../../lang/*.json')),
})
```

**显式配置项**：仅配置了 `resolve` 函数，用于动态加载 JSON 语言文件。

### 12.2 插件默认配置值
根据 `laravel-vue-i18n@2.8.0` 文档，未显式配置的项使用以下默认值：

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `lang` | 自动检测 | 从 `<html lang="xx">` 标签读取，如不存在则使用 `fallbackLang` |
| `fallbackLang` | `'en'` | 当语言无效或未提供时的 fallback 语言 |
| `fallbackMissingTranslations` | **未启用** (v2.8.0 默认关闭) | 翻译键缺失时是否 fallback 到 `fallbackLang` |

### 12.3 关键结论：缺失翻译的行为
由于 `fallbackMissingTranslations` **默认关闭**，当前项目前端的实际行为是：

```
调用 $t('missing.key')
    ↓
查找当前 locale 的 JSON 文件
    ↓ 未找到
★ 不进行 fallback！
    ↓
直接返回 'missing.key' 本身
```

**与后端行为不一致**：
- 后端：`__('missing.key')` → 自动 fallback 到 `en` → 找到则返回，否则返回键
- 前端：`$t('missing.key')` → **不 fallback** → 直接返回键

**代码验证**：前端项目中未配置 `fallbackMissingTranslations: true`，因此缺失翻译不会自动 fallback 到英文。

### 12.4 改进建议
如需前端与后端行为一致，应在 [app.js](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/resources/js/app.js#L31-L33) 中显式启用：

```javascript
.use(i18nVue, {
    resolve: (lang) => resolvePageComponent(`../../lang/${lang}.json`, import.meta.glob('../../lang/*.json')),
    fallbackMissingTranslations: true,  // 启用缺失翻译 fallback
})
```

---

## 十三、复数翻译退化路径

Monica 项目使用 Laravel 标准的 `|` 分隔符定义复数形式，支持复杂的数量区间匹配。

### 13.1 复数格式定义
#### 后端：`trans_choice()`
虽然项目中未直接使用 `trans_choice()`，但翻译文件中已定义复数格式：

[lang/en.json:3](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/lang/en.json#L3)
```json
"(and :count more errors)": "(and :count more error)|(and :count more errors)|(and :count more errors)"
```

#### 前端：`$tChoice()`
在多个 Vue 组件中使用 `$tChoice()` 处理复数：

- [MoodTrackingEvent.vue:31-35](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/resources/js/Shared/Modules/FeedItems/MoodTrackingEvent.vue#L31-L35)
  ```javascript
  $tChoice(
      'Slept :count hour|Slept :count hours',
      data.mood_tracking_event.object.number_of_hours_slept,
      { count: data.mood_tracking_event.object.number_of_hours_slept },
  )
  ```

- [Tags.vue:53](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/resources/js/Pages/Vault/Settings/Partials/Tags.vue#L53)
  ```javascript
  $tChoice(':count post|:count posts', tag.count, { count: tag.count })
  ```

### 13.2 Laravel 复数规则
Laravel 支持三种复数格式：

| 格式 | 示例 | 匹配逻辑 |
|------|------|----------|
| **简单二选一** | `'apple|apples'` | count=1 → 左边，其他 → 右边 |
| **区间匹配** | `'{0} no apples|[1,19] some apples|[20,*] many apples'` | 精确匹配区间 |
| **多区间** | `'{1} :count error|[2,*] :count errors'` | 支持多个区间 |

**项目中使用的特殊三区间格式**：
[lang/en.json:3](file:///d:/fz/0601-2/solo-dogfeeding/code/68-monica/lang/en.json#L3)
```json
"(and :count more errors)": "(and :count more error)|(and :count more errors)|(and :count more errors)"
```

这是处理俄语等有复杂复数规则语言的标准写法，三个区间分别对应：
- 区间1：count = 1 → "error" (单数)
- 区间2：count 为 2-4 或某些特殊数字 → "errors" 
- 区间3：count ≥ 5 或其他 → "errors"

在英语中虽然区间2和3相同，但为了与其他语言的翻译文件结构保持一致，仍保留三区间格式。

### 13.3 复数翻译 Fallback 完整路径
```
调用 $tChoice('key', $count, $replace) 或 trans_choice('key', $count, $replace)
    ↓
查找当前 locale 的翻译文件
    ├─ 后端: lang/{locale}.json 或 lang/{locale}/{file}.php
    └─ 前端: lang/{locale}.json
    ↓
找到翻译字符串？
    ├─ 是 → 解析 `|` 分隔的复数区间
    │       ↓
    │       根据 $count 匹配正确区间
    │       ↓
    │       替换参数 → 返回结果
    │
    └─ 否 → 进入 fallback 流程
            ↓
            ★ 后端: 查找 fallback_locale (en) 的翻译
            │       ↓
            │       找到？→ 解析复数 → 返回
            │       ↓ 未找到
            │       返回 key 本身（但参数替换仍会进行）
            │
            ★ 前端: 不 fallback（默认配置）
                    ↓
                    返回 key 本身（参数替换仍会进行）
```

### 13.4 复数翻译缺失的特殊处理
**关键行为**：即使翻译键缺失，参数替换（`count` 等占位符）仍会执行。

示例：
```javascript
// 假设 ':count apple|:count apples' 完全缺失
$tChoice(':count apple|:count apples', 5, { count: 5 })

// 前端默认行为：不 fallback，直接返回 key 并替换参数
// 结果：':count apple|:count apples' → 参数替换后 → '5 apple|5 apples'
```

---

## 十四、关键注意事项（补充）

1. **英语作为基准**：`en.json` 是所有翻译的基准，键和值相同。
2. **双重 fallback**：模型层有自己的 fallback（用户值 → 翻译键），翻译系统又有一层 fallback（当前语言 → fallback 语言 → 键本身）。
3. **最终保底**：无论哪层 fallback，**最终都不会返回空值**，最差情况返回翻译键本身（即原始英文串）。
4. **locale 和 fallback_locale 相同**：当前配置中两者均为 `'en'`，这意味着英文环境下不会触发 fallback（但对其他语言仍有效）。
5. **前端默认配置**：项目中未显式配置 `fallbackMissingTranslations`，需要注意 `laravel-vue-i18n` 版本的默认行为。
6. **前后端 fallback 不一致**：后端自动 fallback 到英文，前端默认不 fallback，翻译缺失时行为有差异。
7. **Stores 持久化三件套**：Session（会话级）+ Cookie（长期）+ App（运行时）三级存储，确保 locale 正确持久化。
8. **`trusted_detectors` 安全开关**：当前为空，所有 locale 都经过白名单校验，避免恶意 locale 注入。
9. **Inertia share 不传递 locale**：前端通过 HTML lang 属性和 `loadLanguageAsync` 管理语言，不依赖 Inertia 共享数据。
10. **复数翻译参数替换**：即使翻译键缺失，占位符参数仍会被替换，避免显示原始 `:count` 等占位符。
11. **语言名称自翻译技巧**：`__('auth.lang', [], $code)` 通过第三个参数强制指定 locale，实现语言名称用自身显示。
