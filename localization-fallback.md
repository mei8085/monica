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

## 八、关键注意事项

1. **英语作为基准**：`en.json` 是所有翻译的基准，键和值相同。
2. **双重 fallback**：模型层有自己的 fallback（用户值 → 翻译键），翻译系统又有一层 fallback（当前语言 → fallback 语言 → 键本身）。
3. **最终保底**：无论哪层 fallback，**最终都不会返回空值**，最差情况返回翻译键本身（即原始英文串）。
4. **locale 和 fallback_locale 相同**：当前配置中两者均为 `'en'`，这意味着英文环境下不会触发 fallback（但对其他语言仍有效）。
5. **前端默认配置**：项目中未显式配置 `fallbackMissingTranslations`，需要注意 `laravel-vue-i18n` 版本的默认行为。
