# Vault Breadcrumb 与上下文路由协作路径分析

## 1. 架构总览

Monica 的 Vault 模块采用 **后端驱动路由 + 前端内联面包屑** 的协作模式：

```
后端 (PHP/Laravel)                          前端 (Vue3/Inertia)
─────────────────────────────────           ─────────────────────────
routes/web.php 定义路由层级                  每个 Vue 页面内联手写面包屑
        ↓                                            ↑
Controller 接收路由参数 → 调用 ViewHelper          layoutData (来自后端)
        ↓                                            ↑
ViewHelper 调用 route() 生成 URL ────────→      data.url  (来自后端)
        ↓
Inertia::render() 传递 layoutData + data
```

---

## 2. 面包屑的两种形态

### 2.1 通用组件（定义了但极少使用）

- **文件**：[Breadcrumb.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Components/Breadcrumb.vue)
- **接口**：接收 `items: Array`，每个元素包含 `{ name, url }`
- **现状**：虽然有通用组件定义，但**实际所有 Vault 页面都采用内联方式手写面包屑**，并未复用此组件。

### 2.2 页面内联手写（实际采用的方案）

每个 Vue 页面在 `<template>` 顶部通过 `<nav>` + `<ul>/<li>` 结构直接定义面包屑，样式统一，结构手写。

典型结构（以 [Contact/Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Pages/Vault/Contact/Show.vue#L172-L192) 为例）：

```
You are here:  Contacts  →  Profile of John Doe
```

---

## 3. 层级来源：数据注入链路

### 3.1 顶层上下文：`layoutData`（全局共享）

**来源**：每个 Controller 渲染时通过 `VaultIndexViewHelper::layoutData($vault)` 生成。

- **核心文件**：[VaultIndexViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultIndexViewHelper.php#L17-L88)
- **作用**：为整个 Layout 提供用户信息、当前 Vault 信息、以及 Vault 内部一级导航的 URL。

关键结构：

```
layoutData
├── user                 (全局用户信息)
├── vault                (当前 Vault 上下文，非 vault 内页面为 null)
│   ├── id
│   ├── name
│   ├── permission       (at_least_editor, at_least_manager)
│   ├── visibility       (各 Tab 开关：show_journal_tab 等)
│   └── url              (一级导航的路由集合)
│       ├── dashboard    → vault.show
│       ├── contacts     → contact.index
│       ├── journals     → journal.index
│       ├── groups       → group.index
│       ├── reports      → vault.reports.index
│       ├── ...
│
└── url                  (全局 URL)
    ├── vaults           → vault.index
    ├── settings         → settings.index
    └── logout           → logout
```

**前端消费位置**：
- [Layout.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Layouts/Layout.vue#L55-L66) 的顶部导航（用户名 → Vault 名）
- [Layout.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Layouts/Layout.vue#L200-L271) 的二级 Tab 导航
- 每个页面的面包屑第一层（如 `Contacts` 链接）

### 3.2 页面特定：`data`（本页专属）

**来源**：由各模块的 ViewHelper 生成，结构各异。

典型 ViewHelper：
- 联系人：[ContactShowViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php#L41-L97)
- 日志：[JournalShowViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/JournalShowViewHelper.php#L20-L59)
- 分组：[GroupShowViewHelper.php]
- 仪表盘：[VaultShowViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php)

`data` 中通常包含：
- 实体自身的显示字段（`name`, `title`, `description` 等）
- `url` 子对象：该实体相关的 CRUD、关联跳转路由

---

## 4. 路由层级结构

### 4.1 路由定义（嵌套式）

**文件**：[routes/web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/routes/web.php#L189-L498)

核心层级嵌套：

```
vaults/                               vault.index     Vault 列表
├── {vault}/                          vault.show      Vault 仪表盘
│   ├── contacts/                     contact.index   联系人列表
│   │   └── {contact}/                contact.show    联系人详情
│   │       ├── notes/                contact.notes.index
│   │       ├── photos/               contact.photos.index
│   │       ├── importantDates/       contact.important_dates.index
│   │       └── ...
│   │
│   ├── journals/                     journal.index   日志列表
│   │   └── {journal}/                journal.show    日志详情
│   │       ├── posts/                post.*          帖子
│   │       ├── metrics/              journal_metrics.*
│   │       └── slices/               slices.*
│   │
│   ├── groups/                       group.index     分组列表
│   │   └── {group}/                  group.show      分组详情
│   │
│   ├── reports/                      vault.reports.index
│   │   ├── addresses/                ...addresses.index
│   │   │   ├── country/{country}/    ...countries.show
│   │   │   └── city/{city}/          ...cities.show
│   │   ├── moodTrackingEvents/       ...mood_tracking_events.index
│   │   └── importantDates/           ...important_dates.index
│   │
│   ├── files/  calendar/  companies/  tasks/  settings/  ...
```

### 4.2 路由参数传递

路由参数通过 **Laravel Route Model Binding + Controller 方法签名** 传递：

以 [ContactController@show](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L96-L126) 为例：

```php
// 路由定义: vaults/{vault}/contacts/{contact} → contact.show
public function show(Request $request, string $vaultId, string $contactId)
{
    $vault = Vault::findOrFail($vaultId);
    $contact = Contact::with(...)->findOrFail($contactId);
    // ...
    return Inertia::render('Vault/Contact/Show', [
        'layoutData' => VaultIndexViewHelper::layoutData($vault),  // 注入 vault 上下文
        'data' => ContactShowViewHelper::data($contact, Auth::user()), // 注入页面数据
    ]);
}
```

---

## 5. 面包屑层级与路由的对应关系

### 5.1 典型页面的面包屑拆解

#### ① 联系人详情页（2 层面包屑）
- **文件**：[Contact/Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Pages/Vault/Contact/Show.vue#L172-L192)
- **路由**：`vaults/{vault}/contacts/{contact}`
- **面包屑**：

| 层级 | 显示文本 | 链接来源 | 链接值 |
|------|---------|---------|--------|
| 1 | Contacts | `layoutData.vault.url.contacts` | route('contact.index') |
| 2 | Profile of :name | 无（终端节点） | - |

#### ② 日志详情页（2 层面包屑）
- **文件**：[Journal/Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Pages/Vault/Journal/Show.vue#L27-L47)
- **路由**：`vaults/{vault}/journals/{journal}`
- **面包屑**：

| 层级 | 显示文本 | 链接来源 | 链接值 |
|------|---------|---------|--------|
| 1 | Journals | `layoutData.vault.url.journals` | route('journal.index') |
| 2 | {journal.name} | 无（终端节点） | - |

#### ③ 帖子详情页（3 层面包屑）
- **文件**：[Journal/Post/Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Pages/Vault/Journal/Post/Show.vue#L14-L58)
- **路由**：`vaults/{vault}/journals/{journal}/posts/{post}`
- **面包屑**：

| 层级 | 显示文本 | 链接来源 | 链接值 |
|------|---------|---------|--------|
| 1 | Journals | `layoutData.vault.url.journals` | route('journal.index') |
| 2 | {journal.name} | `data.url.back` | route('journal.show') |
| 3 | {post.title} | 无（终端节点） | - |

#### ④ 地址报表页（2 层面包屑）
- **文件**：[Reports/Address/Index.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Pages/Vault/Reports/Address/Index.vue#L13-L40)
- **路由**：`vaults/{vault}/reports/addresses`
- **面包屑**：

| 层级 | 显示文本 | 链接来源 | 链接值 |
|------|---------|---------|--------|
| 1 | Reports | `layoutData.vault.url.reports` | route('vault.reports.index') |
| 2 | List of addresses | 无（终端节点） | - |

#### ⑤ 分组详情页（2 层面包屑）
- **文件**：[Group/Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Pages/Vault/Group/Show.vue#L41-L70)
- **路由**：`vaults/{vault}/groups/{group}`
- **面包屑**：

| 层级 | 显示文本 | 链接来源 | 链接值 |
|------|---------|---------|--------|
| 1 | Groups | `layoutData.vault.url.groups` | route('group.index') |
| 2 | {group.name} | 无（终端节点） | - |

#### ⑥ 帖子编辑页（4 层面包屑）
- **文件**：[Journal/Post/Edit.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Pages/Vault/Journal/Post/Edit.vue#L154-L213)
- **路由**：`vaults/{vault}/journals/{journal}/posts/{post}/edit`
- **面包屑**：

| 层级 | 显示文本 | 链接来源 | 链接值 | 后端生成 |
|------|---------|---------|--------|---------|
| 1 | Journals | `layoutData.vault.url.journals` | `journal.index` | layoutData |
| 2 | {journal.name} | `data.url.back` | `journal.show` | [PostEditViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/PostEditViewHelper.php#L113-L116) |
| 3 | {post.title} | `data.url.show` | `post.show` | [PostEditViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/PostEditViewHelper.php#L88-L92) |
| 4 | Edit a post | 无（终端节点） | - | 前端静态文本 |

####  切片详情页（4 层面包屑）
- **文件**：[Journal/Slices/Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Pages/Vault/Journal/Slices/Show.vue#L60-L117)
- **路由**：`vaults/{vault}/journals/{journal}/slices/{slice}`
- **面包屑**：

| 层级 | 显示文本 | 链接来源 | 链接值 | 后端生成 |
|------|---------|---------|--------|---------|
| 1 | Journals | `layoutData.vault.url.journals` | `journal.index` | layoutData |
| 2 | {journal.name} | `data.journal.url.show` | `journal.show` | [SliceOfLifeShowViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/SliceOfLifeShowViewHelper.php#L48-L57) |
| 3 | Slices of life | `data.url.slices_index` | `slices.index` | [SliceOfLifeShowViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/SliceOfLifeShowViewHelper.php#L63-L68) |
| 4 | {slice.name} | 无（终端节点） | - | （通过 `localSlice.name` 本地响应式变量 |

> **注意**：切片详情页的终端层使用 `localSlice` 而非 `data.slice.name`，因为封面图更新后会更新本地响应式变量，面包屑名称随之变化。

####  切片编辑页（5 层面包屑）
- **文件**：[Journal/Slices/Edit.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Pages/Vault/Journal/Slices/Edit.vue#L34-L102)
- **路由**：`vaults/{vault}/journals/{journal}/slices/{slice}/edit`
- **面包屑**：

| 层级 | 显示文本 | 链接来源 | 链接值 | 后端生成 |
|------|---------|---------|--------|---------|
| 1 | Journals | `layoutData.vault.url.journals` | `journal.index` | layoutData |
| 2 | {journal.name} | `data.journal.url.show` | `journal.show` | [SliceOfLifeEditViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/SliceOfLifeEditViewHelper.php#L24-L33) |
| 3 | Slices of life | `data.url.slices_index` | `slices.index` | [SliceOfLifeEditViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/SliceOfLifeEditViewHelper.php#L40-L43) |
| 4 | {slice.name} | `data.slice.url.show` | `slices.show` | [SliceOfLifeEditViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/SliceOfLifeEditViewHelper.php#L12-L23) |
| 5 | Edit slice of life | 无（终端节点） | - | 前端静态文本 |

### 5.2 多层面包屑回链规则模式

通过分析以上 8 个典型页面，面包屑层级来源有明确的规则模式：

```
第 1 层 → 🔗 layoutData.vault.url.xxx
             （永远对应 Vault 一级 Tab 列表页）
             例：Journals / Contacts / Groups / Reports
             ← layoutData.vault.url.journals

第 2 ~ N-2 层 → 🔗 data.*.url.show 或 data.url.xxx_index
             （中间层级：父实体详情页 / 中间列表页
             例：{journal.name} ← data.journal.url.show
             例：Slices of life ← data.url.slices_index

第 N-1 层 → 🔗 data.url.show（编辑/创建页的"前一层
             （当前实体的详情页，仅在编辑/创建页存在）
             例：{post.title} ← data.url.show

第 N 层 → 📝 纯文本（终端节点
             （当前页的标题/操作名）
             例：Edit a post / Profile of John Doe
```

**URL 来源口诀**：**一层 layoutData，深层 data 挖，最后是文本。

### 5.3 `data.url.back` 的两种模式对比

不同 ViewHelper 中，面包屑回链 URL 的命名并不统一，存在两种模式：

| 模式 | 示例 | 所在 ViewHelper |
|------|------|---------------|
| **`data.url.back`** | 帖子详情/编辑页 | [PostShowViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/PostShowViewHelper.php#L111-L114)、[PostEditViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/PostEditViewHelper.php#L113-L116) |
| **`data.journal.url.show`** | 切片详情/编辑页 | [SliceOfLifeShowViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/SliceOfLifeShowViewHelper.php#L48-L57)、[SliceOfLifeEditViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/SliceOfLifeEditViewHelper.php#L24-L33) |

- 两种模式在不同模块约定俗成，并无强制规范。新增页面时需参考同模块其他页面的写法。

### 5.4 层级来源总结

```
面包屑层级来源：
├─ layoutData.vault.url.*     →  总是对应 Vault 下的一级 Tab（第 1 层用它）
├─ data.url.back              →  回到父实体（第 2 层及更深层用它）
├─ data.url.show/edit/index   →  后端 ViewHelper 中 route() 生成的具体实体路由
└─ 纯文本 (无链接)            →  终端当前页（最后一层）
```

---

## 6. 跳转机制

### 6.1 面包屑中的跳转

面包屑的链接使用 **Inertia 的 `<Link>` 组件**（而非普通 `<a>`），实现 SPA 无刷新跳转：

```vue
<Link :href="layoutData.vault.url.contacts" class="text-blue-500 hover:underline">
  {{ $t('Contacts') }}
</Link>
```

### 6.2 后端 URL 生成

所有路由 URL 都在 **后端 ViewHelper 中通过 Laravel 的 `route()` 函数** 生成，前端只消费、不拼接。

以 [VaultIndexViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultIndexViewHelper.php#L40-L80) 为例：

```php
'url' => [
    'contacts' => route('contact.index', ['vault' => $vault->id]),
    'journals' => route('journal.index', ['vault' => $vault->id]),
    // ...
],
```

实体路由（如 [JournalShowViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/JournalShowViewHelper.php#L32-L57)）：

```php
'url' => [
    'edit'    => route('journal.edit',    ['vault' => $journal->vault_id, 'journal' => $journal->id]),
    'destroy' => route('journal.destroy', ['vault' => $journal->vault_id, 'journal' => $journal->id]),
    // ...
],
```

### 6.3 前端程序化跳转

使用 Inertia 的 `router.visit()` 或 `form.delete/post/put`：

- [Contact/Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Pages/Vault/Contact/Show.vue#L66-L81) 删除联系人后的跳转
- [Layout.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Layouts/Layout.vue#L27-L31) 搜索框聚焦跳转

---

## 7. 导航中的三层结构区分

Monica 的 Vault 页面实际上存在 **三层导航结构**，面包屑只是其中一层，需要区分清楚：

| 层级 | 位置 | 作用 | 数据来源 |
|------|------|------|---------|
| ① 全局顶栏 | [Layout.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Layouts/Layout.vue#L51-L66) | 用户名 → 当前 Vault 名 | `layoutData.url.vaults` + `layoutData.vault.name` |
| ② Tab 导航 | [Layout.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Layouts/Layout.vue#L200-L271) | Dashboard/Contacts/Journals/Groups/Reports 等 | `layoutData.vault.url.*` + `visibility` 开关 |
| ③ 面包屑 | 各 Vue 页面 `<nav>` 内 | Tab 内的子层级路径 | `layoutData.vault.url.*` + `data.url.*` |

**关系**：
- ① 和 ② 在 Layout 中统一渲染，所有 Vault 页面共享
- ③ 由每个页面独立内联定义，反映 Tab 内部更深的路由层级

---

---

## 8. 权限与 Tab 显示开关的耦合关系

### 8.1 Vault 三级权限体系

Vault 采用数值型权限分级，数值越小权限越高。权限值存储在 `vault_users` 表的 `permission` 字段。

**权限常量定义**：[Vault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Models/Vault.php#L379-L388)

| 权限级别 | 常量名 | 数值 | 说明 |
|---------|--------|------|------|
| 管理员 | `PERMISSION_MANAGE` | 100 | 最高权限，可管理 Vault 设置和成员 |
| 编辑者 | `PERMISSION_EDIT` | 200 | 可创建、编辑、删除内容 |
| 查看者 | `PERMISSION_VIEW` | 300 | 只读访问 |

**权限比较逻辑**：权限值越小权限越高。判断是否有编辑权限时，比较 `permission <= PERMISSION_EDIT`。

### 8.2 Gate 定义与权限检查

权限通过 Laravel Gate 系统封装，定义在 [AuthServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Providers/AuthServiceProvider.php)。

**主要 Gate 列表**：

| Gate 名称 | 对应权限 | 用途 |
|----------|---------|------|
| `vault-viewer` | PERMISSION_VIEW (300) | 查看 Vault 基本权限 |
| `vault-editor` | PERMISSION_EDIT (200) | 编辑内容权限 |
| `vault-manager` | PERMISSION_MANAGE (100) | 管理 Vault 权限 |
| `contact-owner` | vault-editor | 联系人操作权限 |
| `journal-owner` | vault-editor | 日记操作权限 |
| `post-owner` | vault-editor | 帖子操作权限 |
| `sliceOfLife-owner` | vault-editor | 人生切片操作权限 |
| `group-owner` | vault-editor | 分组操作权限 |

注意：`contact-owner`、`journal-owner` 等实体级 Gate 均直接复用 `vault-editor` 权限，不做更细粒度的实体所有权检查。

### 8.3 VaultHelper 权限辅助

[VaultHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Helpers/VaultHelper.php) 提供静态方法获取当前用户在指定 Vault 中的权限，并带 5 秒数组缓存。

```php
// 获取权限值（带缓存）
public static function getPermission(Vault $vault): int

// 检查权限
public static function can(Vault $vault, string $ability): bool
```

### 8.4 Tab 显示开关字段

Vault 模型中有 7 个布尔型字段控制 Tab 的显示与隐藏，存储在 `vaults` 表中：

| 字段名 | 对应 Tab | 默认值 |
|--------|---------|--------|
| `show_journal_tab` | 日记 (Journals) | true |
| `show_group_tab` | 分组 (Groups) | true |
| `show_tasks_tab` | 任务 (Tasks) | true |
| `show_files_tab` | 文件 (Files) | true |
| `show_companies_tab` | 公司 (Companies) | true |
| `show_reports_tab` | 报告 (Reports) | true |
| `show_calendar_tab` | 日历 (Calendar) | true |

**设置入口**：Vault 设置  Tab 显示开关页面，由 [VaultSettingsTabVisibilityController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageVaultSettings/Web/Controllers/VaultSettingsTabVisibilityController.php) 处理。

**前端设置组件**：[TabVisibility.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Pages/Vault/Settings/Partials/TabVisibility.vue)


> **注意**：Dashboard、Contacts 两个 Tab 始终显示，没有对应的显示开关字段；Settings Tab 由 `permission.at_least_editor` 权限控制，也不受 visibility 开关影响。

### 8.5 layoutData 中的权限与可见性结构

每个 Vault 页面的 `layoutData.vault` 都包含权限、可见性和 URL 三部分数据：

```
layoutData.vault
  permission         当前用户权限布尔判断对象
    at_least_editor     permission <= 200（可编辑内容）
    at_least_manager    permission <= 100（可管理 Vault）
  visibility         7 个 Tab 显示开关布尔值
    show_journal_tab
    show_group_tab
    show_tasks_tab
    show_files_tab
    show_companies_tab
    show_reports_tab
    show_calendar_tab
  url                所有 Tab 的顶层链接（全部生成，不受可见性影响）
      dashboard
      contacts
      calendar
      journals
      groups
      companies
      tasks
      files
      reports
      settings
      search
```

**关键耦合规则**：
1. **URL 始终全量生成**：`layoutData.vault.url.*` 不论 Tab 是否显示，都会生成所有 URL
2. **Tab 按钮按可见性过滤**：Layout 中的 Tab 导航通过 `v-if="visibility.show_xxx_tab"` 控制显示
3. **面包屑不检查可见性**：面包屑链接直接使用 `layoutData.vault.url.*`，不判断 Tab 是否隐藏
4. **权限在路由层拦截**：能否访问页面由路由中间件的 Gate 检查决定，面包屑层面不做权限校验

### 8.6 Tab 显示与路由权限的关系

Tab 显示开关和权限是两个独立的维度：

| 场景 | Tab 显示 | 有访问权限 | 面包屑链接 | 直接访问URL |
|------|---------|-----------|-----------|------------|
| 正常 |  显示 |  有权限 |  可见可点 |  可访问 |
| Tab 隐藏 |  隐藏 |  有权限 |  面包屑仍可跳转 |  可访问 |
| 无权限 |  不显示 |  无权限 |  到不了页面 |  403 |

结论：**面包屑的顶层链接总是可用的，只要用户有权限访问**。Tab 隐藏只是视觉上不显示导航入口，不影响通过面包屑或直接 URL 访问。

### 8.7 面包屑与权限的关系

面包屑本身不包含任何权限判断逻辑，它遵循以下原则：

1. **可达即可用**：如果用户能到达当前页面，那么面包屑中的所有回链都应该是可访问的
2. **权限前置校验**：权限检查发生在路由中间件层（`can:vault-viewer` 等），在进入 Controller 之前完成
3. **数据信任**：面包屑直接信任后端传入的 URL 数据，不做二次验证
4. **可见性不影响面包屑**：即使某个 Tab 被隐藏，面包屑中对应的顶层链接仍然可以点击跳转

---
## 9. 关键文件索引

| 功能 | 文件路径 |
|------|---------|
| 路由定义 | [routes/web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/routes/web.php) |
| Inertia 中间件（共享全局 props） | [HandleInertiaRequests.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Http/Middleware/HandleInertiaRequests.php) |
| layoutData 构建 | [VaultIndexViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultIndexViewHelper.php) |
| 通用 Layout（含顶栏+Tab） | [Layout.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Layouts/Layout.vue) |
| 面包屑通用组件（未广泛使用） | [Breadcrumb.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Components/Breadcrumb.vue) |
| Vault Controller | [VaultController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php) |
| Contact Controller | [ContactController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php) |
| Journal Controller | [JournalController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/Controllers/JournalController.php) |
| Post Controller（帖子 CRUD） | [PostController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/Controllers/PostController.php) |
| SliceOfLife Controller（人生切片 CRUD） | [SliceOfLifeController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/Controllers/SliceOfLifeController.php) |
| Post 详情数据 | [PostShowViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/PostShowViewHelper.php) |
| Post 编辑数据 | [PostEditViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/PostEditViewHelper.php) |
| Post 创建数据 | [PostCreateViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/PostCreateViewHelper.php) |
| Slice 详情数据 | [SliceOfLifeShowViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/SliceOfLifeShowViewHelper.php) |
| Slice 编辑数据 | [SliceOfLifeEditViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/SliceOfLifeEditViewHelper.php) |
| Gate 权限定义 | [AuthServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Providers/AuthServiceProvider.php) |
| Vault 模型（权限常量 + Tab 字段） | [Vault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Models/Vault.php) |
| Vault 权限辅助类 | [VaultHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Helpers/VaultHelper.php) |
| Tab 显示开关设置控制器 | [VaultSettingsTabVisibilityController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/app/Domains/Vault/ManageVaultSettings/Web/Controllers/VaultSettingsTabVisibilityController.php) |
| Tab 显示开关前端组件 | [TabVisibility.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/75-monica/resources/js/Pages/Vault/Settings/Partials/TabVisibility.vue) |

---

## 10. 协作路径总结（单条用户请求视角）

```
用户点击跳转  <Link :href="后端生成的URL">
        
浏览器请求 URL (SPA 下是 XHR)
        
Laravel 匹配 routes/web.php 中的路由定义
        
路由中间件检查：
   auth  登录校验
   can:vault-viewer / vault-editor  Gate 权限校验（不通过则 403）
        
Controller 方法接收到路由参数（vaultId / contactId 等）
        
Controller 调用 2 个 ViewHelper：
   VaultIndexViewHelper::layoutData($vault)    生成 layoutData
       permission   当前用户权限级别（100/200/300）
       visibility   7 个 Tab 显示开关
       url          所有 Tab 顶层链接（全量生成，不受可见性影响）
   XxxShowViewHelper::data($entity, $user)     生成 data
        实体业务数据
        实体 URL（show/edit/back 等）
        
Inertia::render('组件名', compact('layoutData', 'data'))
        
Layout 渲染：
   顶部 Tab 导航  按 visibility 过滤显示（v-if="visibility.show_xxx"）
   面包屑区域  页面组件内联渲染
        
页面顶部 <nav> 内联渲染面包屑：
  第1层链接  layoutData.vault.url.xxx（不检查可见性，直接使用）
  第2层链接  data.url.back / data.url.show / data.*.url.show
  最后一层   data.name / data.title (纯文本)
```

---

## 11. 设计特点与注意点

1. **无面包屑状态管理**：面包屑完全是声明式的静态结构，不依赖 Vuex/Pinia，直接由当前页的 props 决定。
2. **URL 后端中心化**：前端从不拼接路径字符串，所有 URL 由后端 `route()` 生成，保证路由改定义时前端自动适配。
3. **面包屑不使用通用组件**：虽然定义了 `Breadcrumb.vue`，但实际页面全部内联。这意味着新增页面需复制面包屑结构，存在样式和结构不一致的潜在风险。
4. **移动设备隐藏**：面包屑 `<nav>` 使用 `sm:mt-20 sm:border-b md:block` 类，在小屏设备上完全隐藏，仅桌面端显示。
5. **权限与可见性耦合**：Tab 导航的显示开关（`show_journal_tab` 等）由后端传入 `layoutData.vault.visibility`，前端据此过滤 Tab；面包屑不做权限判断，直接依赖当前页是否可达。
6. **URL 命名两种模式并存**：Post 模块采用扁平模式（`data.url.back`），Slice 模块采用嵌套模式（`data.journal.url.show`）。两种模式在代码库中并存，新增页面时需注意与同模块保持一致。
7. **面包屑可达性原则**：面包屑本身不做任何权限或可见性校验，遵循"能到达当前页就能访问所有回链"的原则。权限检查完全前置到路由中间件层，面包屑信任后端传入的 URL 数据。
8. **Tab 可见性与面包屑解耦**：Tab 显示开关只控制顶部导航栏的按钮显示，不影响面包屑中的顶层链接。即使某个 Tab 被隐藏，面包屑中对应的链接仍然可点击跳转，URL 始终全量生成。
