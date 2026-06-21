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

### 5.2 层级来源总结

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

## 8. 关键文件索引

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

---

## 9. 协作路径总结（单条用户请求视角）

```
用户点击跳转 → <Link :href="后端生成的URL">
        ↓
浏览器请求 URL (SPA 下是 XHR)
        ↓
Laravel 匹配 routes/web.php 中的路由定义
        ↓
Controller 方法接收到路由参数（vaultId / contactId 等）
        ↓
Controller 调用 2 个 ViewHelper：
  ├─ VaultIndexViewHelper::layoutData($vault)   → 生成 layoutData（含一级导航URL）
  └─ XxxShowViewHelper::data($entity, $user)    → 生成 data（含实体数据+实体URL）
        ↓
Inertia::render('组件名', compact('layoutData', 'data'))
        ↓
前端 Vue 组件接收 props
        ↓
页面顶部 <nav> 内联渲染面包屑：
  第1层链接 → layoutData.vault.url.xxx
  第2层链接 → data.url.back / data.url.show
  最后一层  → data.name / data.title (纯文本)
```

---

## 10. 设计特点与注意点

1. **无面包屑状态管理**：面包屑完全是声明式的静态结构，不依赖 Vuex/Pinia，直接由当前页的 props 决定。
2. **URL 后端中心化**：前端从不拼接路径字符串，所有 URL 由后端 `route()` 生成，保证路由改定义时前端自动适配。
3. **面包屑不使用通用组件**：虽然定义了 `Breadcrumb.vue`，但实际页面全部内联。这意味着新增页面需复制面包屑结构，存在样式和结构不一致的潜在风险。
4. **移动设备隐藏**：面包屑 `<nav>` 使用 `sm:mt-20 sm:border-b md:block` 类，在小屏设备上完全隐藏，仅桌面端显示。
5. **权限与可见性耦合**：Tab 导航的显示开关（`show_journal_tab` 等）由后端传入 `layoutData.vault.visibility`，前端据此过滤 Tab；面包屑不做权限判断，直接依赖当前页是否可达。
