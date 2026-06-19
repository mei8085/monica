# 联系人搜索索引与过滤条件链路分析

本文档详细分析 Monica 系统中「联系人搜索索引」与「过滤/权限/列表」的协作方式，解释为什么两者配合起来令人困惑。

---

## 一、搜索索引系统（Laravel Scout）

> **版本锁定**：根据 `composer.lock:3139-3144`，本项目使用 **`laravel/scout: v10.17.0`**（reference: `66b064ab1f987560d1edfbc10f46557fddfed600`）。以下所有关于 Scout 内部方法链的分析均基于此锁定版本的实现模式。

### 1.1 驱动与总体架构

系统使用 [Laravel Scout](https://laravel.com/docs/scout) 作为搜索抽象层，支持多种后端驱动：

| 驱动 | 适用场景 | 配置位置 |
|------|---------|---------|
| `algolia` | 生产云服务 | [scout.php:119-122](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/config/scout.php#L119-L122) |
| `meilisearch` | 自托管开源搜索 | [scout.php:137-154](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/config/scout.php#L137-L154) |
| `typesense` | 自托管开源搜索 | [scout.php:167-303](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/config/scout.php#L167-L303) |
| `database` | 数据库全文索引（无需额外服务） | - |
| `collection` | 集合级模糊搜索（纯PHP，最慢） | - |
| `null` | 禁用搜索 | - |

索引激活检测逻辑在 [ScoutHelper::isActivated()](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Helpers/ScoutHelper.php#L15-L30)：

```
algolia    → 需要 ALGOLIA_APP_ID
meilisearch→ 需要 MEILISEARCH_KEY
typesense  → 需要 TYPESENSE_API_KEY
database   → 始终启用
collection → 始终启用
```

**关键混淆点**：`database` 和 `collection` 驱动不依赖外部搜索引擎，但仍走 Scout 的 Searchable 代码路径。

### 1.2 可搜索模型与索引字段

以下4个模型使用了 `Laravel\Scout\Searchable` trait：

| 模型 | 索引字段（`toSearchableArray`） | 可过滤字段（Meilisearch） | 可排序字段 |
|------|-------------------------------|--------------------------|-----------|
| [Contact](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Contact.php#L87-L97) | id, vault_id, created_at, updated_at, **first_name, last_name, middle_name, nickname, maiden_name** | id, vault_id | updated_at, first_name, last_name |
| [Group](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Group.php#L45-L51) | id, vault_id, created_at, updated_at, **name** | id, vault_id | updated_at |
| [Note](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Note.php#L37-L45) | id, vault_id, created_at, updated_at, contact_id, **title, body** | id, vault_id, contact_id | updated_at |
| Loan | - | - | - |

**每个模型**还定义了 [ScoutHelper::id()](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Helpers/ScoutHelper.php#L66-L84) 合并的基础字段：
```php
[
    'id'         => (string) $id,           // 主键
    'vault_id'   => (string) $vault_id,     // 保险库ID ⭐ 权限过滤的核心
    'created_at' => (int) $timestamp,
    'updated_at' => (int) $timestamp,
]
```

### 1.3 索引更新机制

索引更新的触发链路：

1. **自动更新（Model 事件）**：通过 `Searchable` trait 监听模型的 `created / updated / deleted` 事件
2. **更新开关**：每个模型的 `searchIndexShouldBeUpdated()` 方法决定是否更新索引 → 调用 `ScoutHelper::isActivated()`
3. **是否可被搜索**：`shouldBeSearchable()` 决定记录是否进入索引

Contact 模型的特殊性在 [Contact.php:104-107](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Contact.php#L104-L107)：

```php
public function shouldBeSearchable()
{
    return $this->listed;   // 只有 listed=true 的联系人进入索引
}
```

**⭐ 这里是第一个混乱点**：`listed` 字段既控制列表页面的显示，又控制搜索索引的收录。但这个信息**不在搜索调用处可见**，需要追到模型方法里才知道。

4. **级联清理**：
   - 删除 Contact 时：删除其关联 Note 的索引 → [Contact.php:124-126](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Contact.php#L124-L126)
   - 删除 Vault 时：删除所有 contacts 及其 notes 的索引 → [Vault.php:76-81](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Vault.php#L76-L81)

5. **手动初始化**：通过 `php artisan scout:setup` → [SetupScout.php](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Console/Commands/SetupScout.php)
   - `scout:sync-index-settings`（仅 isIndexed 驱动）
   - `scout:flush`（清空）
   - `scout:import`（批量导入）

### 1.4 全文索引注解

模型使用 `#[SearchUsingFullText]` 注解标记用于 `database` 驱动的 MATCH AGAINST 查询列，例如 Contact：
```php
#[SearchUsingFullText(['first_name', 'last_name', 'middle_name', 'nickname', 'maiden_name'], ['expanded' => true])]
```

这意味着当使用 `database` 驱动时，Scout 会在数据库层面执行全文索引查询，而不需要外部引擎。数据库迁移时是否创建全文索引由 `scout.full_text_index` 控制。

---

## 二、权限范围（Vault 访问控制）

### 2.1 三级 Vault 权限

权限值定义在 [Vault.php:19-23](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Vault.php#L19-L23)：

| 权限常量 | 值 | 说明 | 允许的 Gate |
|---------|----|------|-------------|
| `PERMISSION_MANAGE` | 100 | 管理员（最高） | vault-manager, vault-editor, vault-viewer |
| `PERMISSION_EDIT` | 200 | 编辑者 | vault-editor, vault-viewer |
| `PERMISSION_VIEW` | 300 | 仅查看（最低） | vault-viewer |

存储在 `user_vault` 中间表的 `permission` 字段（pivot）。

### 2.2 Gate 定义

Gate 在 [AuthServiceProvider.php:32-109](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Providers/AuthServiceProvider.php#L32-L109) 中定义：

```php
vault-viewer   → user->vaults() 存在该 vault_id 记录
vault-editor   → 同上 + permission <= 200
vault-manager  → 同上 + permission <= 100
contact-owner  → contact.vault_id == vault.id  (或通过查询确认)
group-owner    → group.vault_id == vault.id
journal-owner  → journal.vault_id == vault.id
```

**⭐ 第二个混乱点**：`contact-owner` Gate 实际上并不检查「用户是否能访问该vault」，只检查「contact 是否属于这个vault」。真正的用户权限检查由**外层路由中间件** `can:vault-viewer,vault` 完成。

### 2.3 路由层的权限守卫

在 [web.php:199](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/routes/web.php#L199) 中，所有 vault 内的操作都包裹在：

```php
Route::middleware('can:vault-viewer,vault')->prefix('{vault}')->group(function () {
    // 所有联系人/搜索/分组...的路由都在里面
});
```

因此**搜索入口的所有路由**在执行 Controller 前，已经通过了 `vault-viewer` 检查。搜索代码中无需再判断用户是否有权访问该 vault。

**⭐ 第三个混乱点**：搜索 ViewHelper 里只写了 `->where('vault_id', $vault->id)`，从表面看没有任何用户权限检查，但权限实际在**路由层**就完成了 —— 两者是分离的，需要跨越路由→控制器→ViewHelper 三层才能看全。

---

## 三、列表筛选链路（5条不同路径）

系统中**至少有5条不同的路径**获取「筛选后的联系人列表」，每条路径的过滤条件、是否使用索引、排序规则都不相同。

### 3.1 路径A：联系人主列表（按排序浏览）

- **路由**：`contact.index` → GET `/vaults/{vault}/contacts`
- **控制器**：[ContactController@index](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L28-L53)
- **查询方式**：**不使用 Scout**，纯 Eloquent

```php
$contacts = $vault->contacts()        // 关系查询 = 隐式 WHERE vault_id = ?
    ->where('listed', true)           // ⭐ 只显示列出的联系人
    ->orderBy(/* 根据用户偏好 */)     // ASC / DESC / last_updated_at
    ->paginate(25);
```

排序由 `Auth::user()->contact_sort_order` 决定，字段由 `name_order` 通过正则提取。

### 3.2 路径B：按标签筛选联系人

- **路由**：`contact.label.index` → GET `/vaults/{vault}/contacts/labels/{label}`
- **控制器**：[ContactLabelController@index](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactLabelController.php#L17-L33)
- **查询方式**：**不使用 Scout**，多对多关系查询

```php
$contacts = Label::find($labelId)
    ->contacts()                         // 通过 contact_label 中间表
    ->where('vault_id', $vault->id)      // 🔴 二次校验 vault_id
    ->where('listed', true)              // ⭐ 只显示列出的
    ->orderBy('created_at', 'asc')       // 固定排序（和其他路径不同！）
    ->paginate(10);
```

### 3.3 路径C：按分组查看成员

- **路由**：`group.show` → GET `/vaults/{vault}/groups/{group}`
- **控制器**：[GroupController@show](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageGroups/Web/Controllers/GroupController.php#L33-L45)
- **数据组装**：[GroupShowViewHelper::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageGroups/Web/ViewHelpers/GroupShowViewHelper.php#L18-L90)
- **查询方式**：**不使用 Scout**，多对多关系查询

```php
// 按角色（group_type_role_id）分组获取
$group->contacts()->wherePivot('group_type_role_id', $role->id)->get();
// 未分配角色的
$group->contacts()->wherePivotNull('group_type_role_id')->get();
```

**⭐ 第四个混乱点**：这里**没有** `->where('listed', true)`！分组内的联系人即使 archived（listed=false）也会显示。和路径A/B的行为不一致。

### 3.4 路径D：模块内快速搜索（下拉框联想）

- **路由**：`vault.user.search.index` → POST `/vaults/{vault}/search/user/contacts`
- **使用场景**：在贷款、任务、生活事件等模块中选择联系人时的搜索下拉
- **控制器**：[VaultContactSearchController@index](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/Controllers/VaultContactSearchController.php#L16-L23)
- **ViewHelper**：[VaultContactSearchViewHelper::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultContactSearchViewHelper.php#L11-L31)
- **查询方式**：**使用 Scout 搜索**

```php
Contact::search($term)                   // Scout 全文搜索
    ->where('vault_id', $vault->id)      // ⭐ 在搜索引擎端过滤 vault
    ->orderBy('first_name')
    ->orderBy('last_name')
    ->take(5)                            // 只取前5个
    ->get();
```

**⭐ 第五个混乱点**：此处没有 `where('listed', true)`，但搜索结果仍然只包含 listed=true 的联系人，因为 `shouldBeSearchable()` 已经在**入库时**排除了非列出联系人。这个机制完全隐藏在 Scout 内部流程中。

### 3.5 路径E：Vault 全局搜索页面

- **路由**：`vault.search.index/show` → GET/POST `/vaults/{vault}/search`
- **功能**：同时搜索联系人、笔记、分组
- **控制器**：[VaultSearchController](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/Controllers/VaultSearchController.php)
- **ViewHelper**：[VaultSearchIndexViewHelper::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultSearchIndexViewHelper.php#L15-L87)
- **查询方式**：**全部使用 Scout 搜索**

```php
// 联系人
Contact::search($term)->where('vault_id', $vault->id)->get();
// 笔记
Note::search($term)->where('vault_id', $vault->id)->get();
// 分组
Group::search($term)->where('vault_id', $vault->id)->get();
```

---

## 四、索引与过滤条件如何配合（混乱根源分析）

### 4.1 过滤条件分层图

```
请求进入
   │
   ├─ 路由层：'can:vault-viewer,vault' 中间件
   │     → 保证 User 对 Vault 至少有 VIEW 权限
   │
   ├─ Controller 层：构造查询（决定走哪条路径）
   │
   ├─ 非搜索路径（A/B/C）：Eloquent 直接查 DB
   │     ├─ WHERE vault_id = ?（通过关系或显式条件）
   │     ├─ WHERE listed = true（路径C缺失 ⚠️）
   │     ├─ WHERE label_id / group_type_role_id（特定路径）
   │     └─ ORDER BY + paginate()
   │
   └─ 搜索路径（D/E）：先过 Scout 搜索引擎
         │
         ├─ Scout::search($term) 执行全文检索
         │     （前提：模型 mustBeSearchable() → listed=true 的记录才被收录）
         │
         ├─ Scout::where('vault_id', $vault->id)
         │     → 在搜索引擎的 filterableAttributes 中精确匹配
         │     → Meilisearch/Typesense 必须在配置中声明 vault_id 可过滤
         │
         ├─ Scout::orderBy / take
         │
         ├─ Scout::get() 触发实际搜索 → 返回 Model ID 列表
         │
         └─ Scout 用 ID 列表回查 MySQL：SELECT * FROM contacts WHERE id IN (...)
               └─ （⚠️ 这里不会再追加 listed 条件，依赖收录时的保证）
```

### 4.2 双重「listed」过滤机制

| 位置 | 路径A（列表） | 路径D（搜索） |
|------|-------------|--------------|
| 可见性控制 | SQL `WHERE listed=1` | 索引期 `shouldBeSearchable()` 排除 |
| 失败表现 | 直接被SQL排除 | 根本不在索引中，查不到 |
| listed 切换为 false 的即时性 | 立即生效 | 需等 Scout 同步（非queue时同步） |
| 代码可见性 | 查询处一眼可见 | 需要翻阅模型方法 |

**关键**：搜索路径的 `listed` 控制是**索引时**完成的，而不是查询时。这就是为什么代码表面「看起来没有权限过滤」——因为它发生在更早的阶段。

### 4.3 双重「vault隔离」机制

| 位置 | 非搜索路径 | 搜索路径 |
|------|----------|---------|
| 第一级：路由中间件 | vault-viewer Gate | vault-viewer Gate |
| 第二级：查询过滤 | 关系 `$vault->contacts()` 隐式加条件 | Scout `->where('vault_id', ...)` 在引擎内过滤 |
| 第三级：物理索引隔离 | - | ❌ 缺失！不同 Vault 的记录在**同一个索引**里 |

**重要**：Scout 中的所有记录（无论哪个 Vault）默认都在同一个物理索引里，靠 `vault_id` 字段做逻辑隔离。如果搜索引擎的 `where` 条件被漏掉，就会**跨Vault泄漏数据**。

### 4.4 权限 vs 过滤条件的职责分布

| 检查项 | 在哪里检查 | 表现形式 |
|--------|-----------|---------|
| 用户是否登录 | 全局 `auth:sanctum` 中间件 | 未登录跳转 |
| 用户能否访问该Vault | 路由 `can:vault-viewer,vault` | 403 |
| 联系人属于该Vault | Gate `contact-owner` + 查询 `vault_id` 条件 | 403 或 空结果 |
| 联系人已列出(visible) | Eloquent: `where listed=true` / Scout: `shouldBeSearchable()` | 不显示 |
| 按标签/分组等细分过滤 | 对应路径的 pivot JOIN | 不显示 |

### 4.5 为什么让人难以理解？

1. **双轨制设计**：列表页走DB，搜索走Scout，两者代码路径完全不同，却要产出「逻辑一致」的结果。任何一个地方的过滤条件改动，必须同步到另一条路径。

2. **`listed` 语义的双重实现**：在DB查询中是 `WHERE` 条件，在Scout中是索引收录规则。开发者如果只看搜索代码，会疑惑「为什么没有 `where('listed', 1)`」，需要深入 Scout 生命周期才能理解。

3. **权限层层包裹**：路由中间件（vault-viewer）→ 控制器Gate → 查询条件（vault_id）→ 索引收录（shouldBeSearchable）。条件不在同一处，必须跨多个文件才能拼出完整的访问控制图景。

4. **Scout `where` 的虚幻感**：`Contact::search()->where('vault_id', ...)` 看起来和 Eloquent 一样，但执行引擎完全不同。这个 where 不是发送给 MySQL，而是发送给 Algolia/Meilisearch/Typesense，且字段必须预先在 `scout.php` 的 `filterableAttributes` 里注册，否则报错。

5. **搜索结果的ID回查**：Scout 先在搜索引擎中找到 ID，再用 `WHERE id IN (...)` 回 MySQL 取模型。这个回查不附加额外条件，如果搜索引擎侧的过滤失效（配置错误），回查阶段也不会补救。

6. **路径C行为不一致**：Group 展示成员时没有 `listed` 条件，已归档的联系人也会出现在分组详情页，与列表页/搜索页的行为不同。

### 4.6 不同驱动下的行为差异

| 行为 | database / collection | algolia / meilisearch / typesense |
|------|----------------------|----------------------------------|
| Scout `where('vault_id')` 执行位置 | 追加到 SQL WHERE | 发送到搜索引擎的 filter API |
| 索引延迟 | 无（因为是直接查DB） | 有，取决于 queue 配置 |
| `shouldBeSearchable` 强制执行期 | 每次查询时（因为无预建索引）| 写入索引时 |
| 排序字段支持 | 任意表字段 | 必须在 sortableAttributes 中声明 |
| 性能 | 大量数据下较差 | 较好 |

---

## 五、关键代码速查表

| 概念 | 文件 | 行号 |
|------|------|------|
| Scout 总体配置 | [scout.php](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/config/scout.php) | 全文件 |
| 索引激活/字段辅助 | [ScoutHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Helpers/ScoutHelper.php) | 全文件 |
| Scout 初始化命令 | [SetupScout.php](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Console/Commands/SetupScout.php) | 全文件 |
| Contact 索引字段 + shouldBeSearchable | [Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Contact.php) | L87-L137 |
| Vault 权限常量 | [Vault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Vault.php) | L19-L23 |
| Gate 权限定义 | [AuthServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Providers/AuthServiceProvider.php) | L32-L109 |
| VaultPolicy 策略 | [VaultPolicy.php](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Policies/VaultPolicy.php) | 全文件 |
| 路由 + 中间件权限守卫 | [web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/routes/web.php) | L199, L240-L545 |
| 列表页（路径A）| [ContactController@index](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php) | L28-L53 |
| 标签筛选（路径B）| [ContactLabelController@index](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactLabelController.php) | L17-L33 |
| 分组内成员（路径C）| [GroupShowViewHelper::data](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageGroups/Web/ViewHelpers/GroupShowViewHelper.php) | L18-L90 |
| 模块快速搜索（路径D）| [VaultContactSearchViewHelper::data](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultContactSearchViewHelper.php) | L11-L31 |
| 全局搜索（路径E）| [VaultSearchIndexViewHelper::data](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultSearchIndexViewHelper.php) | L30-L87 |
| 最近常用 | [VaultMostConsultedViewHelper::data](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultMostConsultedViewHelper.php) | L13-L39 |

---

## 六、三大核心流程深度剖析

### 6.1 流程一：联系人所有权校验（contact-owner Gate + Service 层双重校验）

这是最容易产生误解的流程，因为「所有权校验」实际上发生在**两个独立的地方**，且职责不同。

#### 6.1.1 路由层 Gate：`can:contact-owner,vault,contact`

**代码位置**：[web.php:250](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/routes/web.php#L250-L255)

```php
Route::middleware('can:contact-owner,vault,contact')->prefix('{contact}')->group(function () {
    Route::get('', [ContactController::class, 'show'])->name('contact.show');
    // ... 所有单联系人操作
});
```

**Gate 定义**：[AuthServiceProvider.php:56-65](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Providers/AuthServiceProvider.php#L56-L65)

```php
Gate::define('contact-owner', function (User $user, $vault, $contact): bool {
    if ($contact instanceof Contact) {
        return $contact->vault_id === static::id($vault);  // 只比较 vault_id 是否相等
    }
    return Contact::where([
        'id' => static::id($contact),
        'vault_id' => static::id($vault),
    ])->exists();
});
```

**⚠️ 关键理解**：
- 这个 Gate **不检查用户的任何权限**，只检查「该 contact 是否属于该 vault」
- 参数顺序是 `($user, $vault, $contact)`，三个参数由路由的 `{vault}` 和 `{contact}` 绑定自动传入
- 它的作用是**防止横向越权**：即使你有 vault 访问权，也不能通过修改 URL 中的 contact_id 访问其他 vault 的联系人

#### 6.1.2 Service 层校验：`contact_must_belong_to_vault`

所有写入操作（创建/更新/删除联系人）都通过 Service 类执行，每个 Service 声明 `permissions()` 数组：

**示例：ToggleArchiveContact** [ToggleArchiveContact.php:31-39](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageContact/Services/ToggleArchiveContact.php#L31-L39)

```php
public function permissions(): array
{
    return [
        'author_must_belong_to_account',
        'vault_must_belong_to_account',
        'contact_must_belong_to_vault',      // ← 这里
        'author_must_be_vault_editor',
    ];
}
```

**校验执行逻辑**：[BaseService.php:205-215](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Services/BaseService.php#L205-L215)

```php
public function validateContactBelongsToVault(array $data): void
{
    if (isset($data['contact_id'])) {
        // 🔴 通过 vault->contacts() 关系查询，隐式包含 WHERE vault_id = ?
        $this->contact = $this->vault->contacts()
            ->findOrFail($data['contact_id']);

        // 🔴 额外的防御性校验，确保不会拿到其他 vault 的 contact
        if ($this->contact->vault_id !== $this->vault->id) {
            throw new ModelNotFoundException;
        }
    }
}
```

#### 6.1.3 权限依赖图

[BaseService.php:41-67](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Services/BaseService.php#L41-L67) 定义了权限依赖顺序：

```
contact_must_belong_to_vault
    ├─ 依赖 vault_must_belong_to_account  → 校验 vault 属于 account
    └─ 依赖 author_must_belong_to_account → 校验 user 属于 account
```

执行顺序是**拓扑排序**的，先执行依赖，再执行权限本身。

**完整校验链路（以 ToggleArchiveContact 为例）**：

```
请求 → 路由中间件 can:contact-owner,vault,contact
          ↓ （检查 contact.vault_id == vault.id）
      ToggleArchiveContact::execute($data)
          ↓
      $this->validateRules($data)
          ├─ author_must_belong_to_account  → 查 user，设置 $this->author
          ├─ vault_must_belong_to_account   → 查 vault，设置 $this->vault
          └─ contact_must_belong_to_vault   → $vault->contacts()->findOrFail($id)
                                              （同时隐式校验 vault_id）
          ↓
      author_must_be_vault_editor → 检查 pivot.permission <= 200
          ↓
      执行业务逻辑：$this->contact->listed = !$this->contact->listed
```

**为什么要有双重校验？**
- 路由层的 Gate 保护所有 GET/POST/DELETE/PUT 路由，防止 URL 篡改
- Service 层的校验是**深度防御**，即使路由层被绕过（如队列任务、Artisan 命令直接调用 Service），也能保证安全
- Service 层还负责加载模型实例（`$this->contact`），供后续业务逻辑使用

---

### 6.2 流程二：分组归档可见性（路径C的 listed 过滤缺失）

这是整个系统中最隐蔽的不一致性。

#### 6.2.1 `listed` 字段的定义

`listed` 字段在 Contact 表中，是布尔型：
- `listed = 1`：正常显示的联系人
- `listed = 0`：已归档（archived）的联系人

**切换归档的服务**：[ToggleArchiveContact.php:44-56](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageContact/Services/ToggleArchiveContact.php#L44-L56)

```php
public function execute(array $data): Contact
{
    $this->data = $data;
    $this->validate();

    $this->contact->listed = ! $this->contact->listed;  // 直接取反
    $this->contact->save();

    $this->updateLastEditedDate();
    $this->createFeedItem();

    return $this->contact;
}
```

**同时触发 Scout 索引更新**：因为 Contact 模型的 `searchIndexShouldBeUpdated()` 返回 true，`shouldBeSearchable()` 返回 `$this->listed`。当 `listed` 从 true 变为 false 时，这条记录会被**从搜索索引中移除**。

#### 6.2.2 五条路径对 `listed` 的处理对比

| 路径 | 查询方式 | 是否过滤 `listed=true` | 代码位置 |
|------|---------|----------------------|---------|
| A. 主列表 | `$vault->contacts()->where('listed', true)` | ✅ 显式过滤 | [ContactController.php:30-31](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L30-L31) |
| B. 标签筛选 | `Label::find()->contacts()->where('listed', true)` | ✅ 显式过滤 | [ContactLabelController.php:21-24](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactLabelController.php#L21-L24) |
| C. 分组成员 | `$group->contacts()` | ❌ **不过滤** | [GroupShowViewHelper.php:26-28](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageGroups/Web/ViewHelpers/GroupShowViewHelper.php#L26-L28) |
| D. 模块内搜索 | `Contact::search($term)->where('vault_id', ...)` | ✅ 收录时过滤（`shouldBeSearchable`） | [Contact.php:104-107](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Contact.php#L104-L107) |
| E. 全局搜索 | `Contact::search($term)->where('vault_id', ...)` | ✅ 收录时过滤（`shouldBeSearchable`） | 同上 |

**Contact 模型其实有 scopeActive，但路径C没用到**：
[Contact.php:112-115](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Contact.php#L112-L115)

```php
public function scopeActive(Builder $query): Builder
{
    return $query->where('listed', 1);
}
```

#### 6.2.3 路径C代码全景

[GroupShowViewHelper.php:18-90](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageGroups/Web/ViewHelpers/GroupShowViewHelper.php#L18-L90)

```php
public static function data(Group $group): array
{
    // 第一部分：按角色分组的成员
    $rolesCollection = $group->groupType === null
        ? collect()
        : $group->groupType->groupTypeRoles()
            ->orderBy('position')
            ->get()
            ->map(function (GroupTypeRole $role) use ($group) {
                // 🔴 这里没有 where('listed', true)！
                $contactsCollection = $group->contacts()
                    ->wherePivot('group_type_role_id', $role->id)
                    ->get()
                    ->map(fn (Contact $contact) => [...]);

                return [...];
            });

    // 第二部分：未分配角色的成员
    $contactsCollection = $group->contacts()
        ->wherePivotNull('group_type_role_id')  // 🔴 这里也没有 where('listed', true)！
        ->get()
        ->map(fn (Contact $contact) => [...]);

    // 如果有未分配角色的成员，追加到 rolesCollection
    if ($contactsCollection->isNotEmpty()) {
        $rolesCollection->push([
            'id' => -1,
            'label' => trans('No role'),
            'contacts' => $contactsCollection,
        ]);
    }

    return [...];
}
```

#### 6.2.4 为什么这是一个问题？

**场景演示**：
1. Vault A 中有名为「家人」的分组，包含成员「张三」（listed=true）
2. 用户将「张三」归档（listed=false）
3. 这时：
   - 主列表：✅ 看不到张三
   - 搜索：✅ 搜不到张三（已从索引移除）
   - 分组详情页：❌ **仍然能看到张三**！因为 `$group->contacts()` 没过滤 listed

**这不是 Bug 就是 Feature 争议**：系统设计时可能有意让分组显示全部成员（包括已归档），但行为不一致容易让用户困惑 — 同一个联系人在列表页消失了，在分组里却还能看到。

---

### 6.3 流程三：全局搜索结果回查（Scout ID 回查机制）

这是整个搜索流程中最关键也最隐蔽的环节。

#### 6.3.1 Scout 搜索执行的完整流程

根据 [Laravel Scout 官方文档](https://laravel.com/docs/scout) 和代码设计，`Contact::search($term)->where('vault_id', $vault->id)->get()` 的执行过程如下：

```
1. Contact::search($term)
   → 创建 Laravel\Scout\Builder 实例
   → 保存搜索词 "John"
   
2. ->where('vault_id', $vault->id)
   → 在 Builder 的 $wheres 数组中保存 ['vault_id' => 'xxx']
   
3. ->get()
   ├─ 调用 Engine::search(Builder $builder)
   │    ├─ Meilisearch/Algolia/Typesense 执行搜索
   │    │   · 全文搜索关键词 "John"
   │    │   · 应用过滤条件 vault_id = "xxx"
   │    │   · 返回匹配文档的 ID 列表 [id1, id2, id3]
   │    │
   │    └─ Scout 用这些 ID 回查 MySQL
   │         → SELECT * FROM contacts WHERE id IN (id1, id2, id3)
   │         → 🔴 这里不会追加任何额外的 WHERE 条件！
   │
   └─ 返回 Eloquent Collection
```

#### 6.3.2 为什么回查不追加条件？

**设计意图**：Scout 的设计哲学是「搜索引擎负责过滤和排序，MySQL 只负责取数据」。所有过滤逻辑应该在搜索引擎侧完成，回查只做 hydration（模型水化）。

**风险点**：如果搜索引擎侧的过滤条件失效，回查阶段不会有任何补救措施。

**失效场景举例**：
1. 开发者忘记写 `->where('vault_id', $vault->id)`
2. `scout.php` 中 `filterableAttributes` 没有包含 `vault_id`，导致过滤条件被搜索引擎忽略
3. 搜索引擎的索引配置错误，导致跨 vault 数据泄漏
4. `shouldBeSearchable()` 的逻辑在索引收录后被修改，但索引没有重建

在这些情况下，**搜索引擎可能返回其他 vault 的 contact ID**，而回查的 `WHERE id IN (...)` 会无条件地把它们取出来，造成**跨 vault 数据泄漏**。

#### 6.3.3 本系统的回查安全保障

本系统通过以下机制降低风险：

| 保障机制 | 作用 | 位置 |
|---------|------|------|
| 路由中间件 `can:vault-viewer,vault` | 保证用户至少能访问该 vault | [web.php:199](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/routes/web.php#L199) |
| Scout 查询显式 `where('vault_id', ...)` | 在搜索引擎侧过滤 | [VaultSearchIndexViewHelper.php:33-35](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultSearchIndexViewHelper.php#L33-L35) |
| 搜索索引收录时 `shouldBeSearchable()` | 保证只有 listed=true 的记录能被搜索到 | [Contact.php:104-107](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Contact.php#L104-L107) |
| `scout.meilisearch.index-settings.filterableAttributes` | 强制声明哪些字段可过滤，防止拼写错误被静默忽略 | [scout.php:141-144](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/config/scout.php#L141-L144) |

**但注意**：这些保障是**纵深防御**，不是 100% 安全。如果搜索引擎侧的 `where('vault_id')` 因为某种原因没有生效（例如 Meilisearch 的 filterableAttributes 漏配），回查阶段没有任何补救。

---

### 6.4 三者之间的关系：一张完整的数据流图

```
用户请求
   │
   ├─ 路由层
   │    ├─ auth:sanctum  → 登录校验
   │    ├─ can:vault-viewer,vault  → 用户能否访问该Vault？
   │    │    （Gate: 用户在 pivot 表中有该 vault 记录）
   │    └─ can:contact-owner,vault,contact  → 该Contact是否属于该Vault？
   │         （Gate: contact.vault_id == vault.id）
   │
   ├─ 操作类型分支
   │    │
   │    ├─ [写入操作] 归档/删除/修改联系人
   │    │    │
   │    │    └─ Service 层（如 ToggleArchiveContact）
   │    │         ├─ validateRules()
   │    │         │   ├─ author_must_belong_to_account → 加载 $this->author
   │    │         │   ├─ vault_must_belong_to_account  → 加载 $this->vault
   │    │         │   ├─ contact_must_belong_to_vault  → $vault->contacts()->findOrFail()
   │    │         │   │                              （隐式 WHERE vault_id = ?）
   │    │         │   └─ author_must_be_vault_editor  → pivot.permission <= 200
   │    │         │
   │    │         ├─ 执行业务：$contact->listed = false
   │    │         └─ 触发 Scout 更新：
   │    │              · shouldBeSearchable() 返回 false
   │    │              · 从搜索索引中删除该文档
   │    │
   │    ├─ [读取操作] 联系人列表/标签筛选
   │    │    │
   │    │    └─ Eloquent 直接查 DB
   │    │         ├─ $vault->contacts()  → 隐式 WHERE vault_id = ?
   │    │         ├─ where('listed', true)  → 只看活跃联系人
   │    │         └─ 分页 + 排序
   │    │
   │    ├─ [读取操作] 分组成员（路径C）
   │    │    │
   │    │    └─ Eloquent 多对多查 DB
   │    │         ├─ $group->contacts()  → JOIN contact_group
   │    │         ├─ ❌ 缺失 where('listed', true)
   │    │         └─ 按角色分组
   │    │
   │    └─ [读取操作] 搜索（路径D/E）
   │         │
   │         └─ Scout 搜索
   │              ├─ Builder 构造：search($term)
   │              ├─ Builder where：where('vault_id', $vault->id)
   │              ├─ Engine::search() → 发送给 Meilisearch/Algolia
   │              │    · 全文检索 + 过滤 vault_id
   │              │    · 返回 ID 列表 [id1, id2, ...]
   │              │
   │              ├─ 🔴 ID 回查 MySQL：SELECT * FROM contacts WHERE id IN (...)
   │              │    · 不追加任何条件
   │              │    · 完全信任搜索引擎返回的 ID
   │              │
   │              └─ 返回 Eloquent Collection
   │
   └─ 响应
```

---

### 6.5 三大流程的交叉影响

#### 6.5.1 归档操作对搜索的影响

```
ToggleArchiveContact::execute()
    ↓
$contact->listed = false
    ↓
Model saved → 触发 Scout observer
    ↓
shouldBeSearchable() 返回 false
    ↓
从搜索索引中删除该文档
    ↓
后续搜索（路径D/E）搜不到该联系人 ✅
但分组详情页（路径C）仍然能看到 ❌
```

#### 6.5.2 所有权校验对搜索的影响

搜索路径（D/E）**不经过** `contact-owner` Gate，因为搜索返回的是列表，不是单个 `{contact}` 路由参数。搜索的所有权保障是：
1. 路由层 `can:vault-viewer,vault` 保证用户能访问 vault
2. Scout `where('vault_id', ...)` 在搜索引擎侧过滤
3. 收录时 `shouldBeSearchable()` 保证只有 listed=true 的记录在索引中

#### 6.5.3 搜索回查对分组可见性的影响

如果某人在分组中有一个已归档的联系人（路径C可见），尝试在搜索框搜他的名字：
- 搜索返回空 ✅（因为 `shouldBeSearchable()` 已将其移除）
- 但分组详情页仍能看到 ❌（不一致）

---

## 七、验证清单（排查搜索/过滤问题时）

1. **确认 Scout 驱动**：`SCOUT_DRIVER` 环境变量 → `ScoutHelper::isActivated()` 是否返回 true
2. **索引是否最新**：如果是 meilisearch/typesense，运行 `scout:setup --flush --import`
3. **listed 字段**：检查目标联系人 `listed=1`，否则不会进入搜索索引
4. **中间件是否保护路由**：路由分组是否在 `can:vault-viewer,vault` 之内
5. **Scout 的 where 条件**：搜索代码是否包含 `where('vault_id', $vault->id)`
6. **Meilisearch/Typesense filterableAttributes**：`scout.php` 中是否声明了 `vault_id` 可过滤
7. **路径一致性**：如果用分组/标签，注意路径C缺失了 `listed` 过滤
8. **权限数字对比**：用户在 pivot 中的 `permission` 值是否 <= 所需阈值（300/200/100）
9. **Service 层权限**：写入操作是否声明了 `contact_must_belong_to_vault` 权限
10. **Gate 参数顺序**：自定义 Gate 时注意参数顺序是 `($user, $vault, $contact)`，不要搞混
11. **搜索回查安全性**：新增搜索路径时，务必添加 `where('vault_id', ...)`，不要只依赖搜索引擎的默认行为

---

## 八、代码顺序逐行分析

### 8.1 搜索框架模型取回：从 `Contact::search()` 到 `get()` 的完整调用链

本章节沿着代码执行顺序，逐行拆解搜索调用链。由于 Scout 代码在 vendor 中，以下分析基于 Laravel Scout 10.x 的标准实现模式 + 本项目的实际配置。

#### 8.1.1 第一步：调用静态方法 `search()`

**代码位置**：通过 `use Searchable;` trait 引入
**触发点**：[VaultSearchIndexViewHelper.php:33](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultSearchIndexViewHelper.php#L33-L35)

```php
$contact = Contact::search($term)   // ① 创建 Builder
    ->where('vault_id', $vault->id)  // ② 追加过滤条件
    ->get();                          // ③ 执行搜索并取回模型
```

**`search()` 方法做了什么**（Searchable trait 提供）：
1. 创建 `Laravel\Scout\Builder` 新实例
2. 传入模型类名和搜索词 `$term`
3. Builder 初始状态：
   - `$model` = Contact 类
   - `$query` = $term
   - `$wheres` = []
   - `$orders` = []
   - `$limit` = null

#### 8.1.2 第二步：`where('vault_id', $vault->id)`

**Builder::where()** 的行为：
1. 将 `['vault_id' => $vault->id]` 存入 `$this->wheres` 数组
2. **注意**：这时候还没有发送给搜索引擎，只是暂存在 Builder 对象里

本项目所有搜索路径都有这一步：
- [VaultSearchIndexViewHelper.php:34](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultSearchIndexViewHelper.php#L34)
- [VaultSearchIndexViewHelper.php:51](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultSearchIndexViewHelper.php#L51)
- [VaultSearchIndexViewHelper.php:76](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultSearchIndexViewHelper.php#L76)
- [VaultContactSearchViewHelper.php:15](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultContactSearchViewHelper.php#L15)

**索引中的 vault_id 是怎么来的？**

[ScoutHelper::id()](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Helpers/ScoutHelper.php#L66-L84) 被每个模型的 `toSearchableArray()` 调用：

```php
public static function id(Model $model): array
{
    if (config('scout.driver') === 'database') {
        return [];  // database 驱动不需要，因为直接查 MySQL
    }
    // ...
    return [
        'id' => $id,
        'vault_id'   => (string) $model->getAttribute('vault_id'),  // 🔴 字符串型
        'created_at' => (int) ...,
        'updated_at' => (int) ...,
    ];
}
```

**注意点**：`vault_id` 在索引中是**字符串**（因为 `(string)` 强制转换），而 `where()` 传入的是 `$vault->id` 也通常是字符串（UUID），所以类型匹配没问题。

#### 8.1.3 第三步：`get()` — 搜索执行的核心

`get()` 方法内部执行流程：

```
Builder::get()
   │
   ├─ 1. 解析 softDelete 配置
   │
   ├─ 2. 调用 Engine::search($builder)
   │    │
   │    ├─ 2.1 构造搜索请求
   │    │   · 全文搜索关键词
   │    │   · 应用 $builder->wheres 中的过滤条件
   │    │   · 应用 $builder->orders 中的排序
   │    │   · 应用 $builder->limit（如果有）
   │    │
   │    └─ 2.2 发送请求到搜索引擎
   │         · Meilisearch: POST /indexes/contacts/search
   │         · Algolia: 对应 search API
   │         · Typesense: 对应 documents/search
   │         · database: 直接执行 SQL MATCH AGAINST
   │
   ├─ 3. 提取搜索结果中的 ID 列表
   │   （搜索引擎返回的 hits 中的 objectID / id）
   │
   ├─ 4. 🔴 关键步骤：模型回查（hydration）
   │   调用 $this->model->getScoutModelsByIds($builder, $ids)
   │    → $this->query()  （获取一个新的 Eloquent 查询构造器）
   │    → whereKey($ids)  （WHERE id IN (...)）
   │    → get()           （执行查询，返回 Collection）
   │
   ├─ 5. 按搜索结果顺序重排模型集合
   │   （因为 WHERE id IN 不保证顺序）
   │
   └─ 6. 返回 Eloquent Collection
```

#### 8.1.4 为什么回查阶段**不追加** `where('vault_id', ...)

**设计哲学**：Scout 假设「搜索引擎已经完成了所有过滤」，回查只负责取数据。

**本项目的影响**：
- 如果搜索引擎侧的 `vault_id` 过滤失效，回查阶段**不会纠正**
- 可能返回其他 vault 的联系人（数据泄漏）

**安全保障依赖的三道防线**：
1. 收录时 `shouldBeSearchable()` — 只让 listed=true 的记录进索引
2. 查询时 `where('vault_id', ...)` — 在搜索引擎侧过滤
3. 路由层 `can:vault-viewer,vault` — 用户至少能访问该 vault

**第4道缺失的防线**：回查时不检查 `listed` 和 `vault_id`

#### 8.1.5 `database` 驱动的特殊情况

当 `SCOUT_DRIVER=database` 时，Scout 的行为不同：

- **没有独立的搜索引擎**，直接在 MySQL 上执行
- `where('vault_id', ...)` 会**追加到 SQL 的 WHERE 子句**
- 不存在「ID回查」的概念，因为查询本身就是完整的 SQL
- 安全边界和普通 Eloquent 查询一样

`isActivated()` 对 `database` 驱动返回 `true`，说明 database 驱动被视为**正常的 Scout 驱动**，不是降级模式。

---

### 8.2 分组所有权校验：从路由到 Service 的双重校验

#### 8.2.1 路由层：`can:group-owner,vault,group`

**代码位置**：[web.php:398](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/routes/web.php#L398-L403)

```php
Route::middleware('can:group-owner,vault,group')->prefix('groups')->group(function () {
    Route::get('{group}', [GroupController::class, 'show'])->name('group.show');
    Route::get('{group}/edit', [GroupController::class, 'edit'])->name('group.edit');
    Route::put('{group}', [GroupController::class, 'update'])->name('group.update');
    Route::delete('{group}', [GroupController::class, 'destroy'])->name('group.destroy');
});
```

**Gate 定义**：[AuthServiceProvider.php:67-76](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Providers/AuthServiceProvider.php#L67-L76)

```php
Gate::define('group-owner', function (User $user, $vault, $group): bool {
    if ($group instanceof Group) {
        return $group->vault_id === static::id($vault);
    }
    return Group::where([
        'id' => static::id($group),
        'vault_id' => static::id($vault),
    ])->exists();
});
```

**逐行解读**：
1. 参数顺序：`($user, $vault, $group)`，对应路由 `{vault}/groups/{group}`
2. 如果 `$group` 已经是 Group 模型（路由模型绑定后），直接比较 `vault_id`
3. 如果是 ID（手动调用），执行 `WHERE id=? AND vault_id=?` 查询
4. **不检查用户权限**，只检查分组是否属于该 vault

**单元测试验证**：[GatesTest.php:79-92](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/tests/Unit/Controllers/GatesTest.php#L79-L92)
- 测试了两种调用方式：传 Group 对象 和 传 id 字符串
- 验证了属于和不属于两种情况

#### 8.2.2 Service 层：`group_must_belong_to_vault`

以 `AddContactToGroup` 为例：[AddContactToGroup.php:37-46](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageGroups/Services/AddContactToGroup.php#L37-L46)

```php
public function permissions(): array
{
    return [
        'author_must_belong_to_account',
        'vault_must_belong_to_account',
        'author_must_be_vault_editor',
        'contact_must_belong_to_vault',
        'group_must_belong_to_vault',
    ];
}
```

**依赖顺序**（BaseService 定义）：
```
group_must_belong_to_vault
    ├─ vault_must_belong_to_account
    └─ author_must_belong_to_account
```

**校验实现**：[BaseService.php:220-230](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Services/BaseService.php#L220-L230)

```php
public function validateGroupBelongsToVault(array $data): void
{
    if (isset($data['group_id'])) {
        $this->group = $this->vault->groups()   // 通过 vault->groups() 关系查询
            ->findOrFail($data['group_id']);    // = WHERE vault_id = ? AND id = ?

        if ($this->group->vault_id !== $this->vault->id) {  // 防御性二次校验
            throw new ModelNotFoundException;
        }
    }
}
```

**逐行解读**：
1. `$this->vault->groups()` 返回 `HasMany` 关系，已经隐含 `WHERE vault_id = $this->vault->id`
2. `findOrFail($data['group_id'])` 在这个约束下查找分组
3. 如果分组不在该 vault 中，抛出 `ModelNotFoundException`
4. **还额外加了一次 `$this->group->vault_id !== $this->vault->id` 校验**，作为深度防御

#### 8.2.3 为什么需要「路由层 Gate + Service 层校验」双重检查？

| 场景 | 路由层 Gate | Service 层校验 |
|------|------------|---------------|
| Web 页面访问（GET/POST） | ✅ 生效 | 写入操作生效 |
| Artisan 命令直接调用 Service | ❌ 不生效 | ✅ 生效 |
| 队列任务调用 Service | ❌ 不生效 | ✅ 生效 |
| API 调用 | ✅ 生效（如果路由定义了） | 写入操作生效 |
| 提供的安全保障 | 防止 URL 篡改 | 确保数据一致性，防御深度 |

**设计模式**：
- 路由层 Gate 是**第一道防线**，保护 HTTP 入口
- Service 层校验是**核心校验**，所有写入路径都必须经过
- 两者重复但目的不同：Gate 偏向「访问控制」，Service 偏向「数据完整性」

---

### 8.3 归档联系人显示差异：五条路径代码逐行对比

本章节逐行对比五条路径对 `listed` 字段的处理，解释为什么行为不一致。

#### 8.3.1 路径A：主列表（显式过滤）

**代码**：[ContactController.php:30-31](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L30-L31)

```php
$contacts = $vault->contacts()
    ->where('listed', true)    // 🔴 显式过滤
    ->orderBy(...)
    ->paginate(25);
```

**逐行分析**：
1. `$vault->contacts()`：Eloquent HasMany 关系 → `WHERE vault_id = ?`
2. `->where('listed', true)`：**显式**追加 `AND listed = 1`
3. 结果：只返回 listed=true 的联系人

#### 8.3.2 路径B：标签筛选（显式过滤）

**代码**：[ContactLabelController.php:21-26](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactLabelController.php#L21-L26)

```php
$contacts = Label::find($labelId)
    ->contacts()
    ->where('vault_id', $request->route()->parameter('vault'))
    ->where('listed', true)    // 🔴 显式过滤
    ->orderBy('created_at', 'asc')
    ->paginate(10);
```

**逐行分析**：
1. `Label::find($labelId)->contacts()`：多对多关系，JOIN contact_label
2. `->where('vault_id', ...)`：二次校验 vault_id
3. `->where('listed', true)`：**显式**追加 `AND listed = 1`
4. 结果：只返回 listed=true 的联系人

**注意**：这里用 `$request->route()->parameter('vault')` 而不是 `$vault->id`，可能是因为控制器没有注入 Vault 模型。

#### 8.3.3 路径C：分组成员（❌ 无过滤）

**代码**：[GroupShowViewHelper.php:26-28](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageGroups/Web/ViewHelpers/GroupShowViewHelper.php#L26-L28)

```php
$contactsCollection = $group->contacts()
    ->wherePivot('group_type_role_id', $role->id)
    ->get();
```

以及 [GroupShowViewHelper.php:48-50](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageGroups/Web/ViewHelpers/GroupShowViewHelper.php#L48-L50)：

```php
$contactsCollection = $group->contacts()
    ->wherePivotNull('group_type_role_id')
    ->get();
```

**逐行分析**：
1. `$group->contacts()`：多对多关系，JOIN contact_group
2. `->wherePivot('group_type_role_id', $role->id)`：只过滤 pivot 表的角色字段
3. **没有** `->where('listed', true)`
4. 结果：返回**所有**联系人，包括 listed=false（已归档）的

**为什么遗漏？可能的原因**：
- 分组被设计为「容器」，不管联系人状态如何都显示
- 开发者的疏忽，因为标签筛选是有的
- 路径C的代码路径和其他路径不同，维护时漏加

#### 8.3.4 路径D：模块内搜索（收录时过滤）

**代码**：[VaultContactSearchViewHelper.php:14-19](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultContactSearchViewHelper.php#L14-L19)

```php
$contacts = Contact::search($term)
    ->where('vault_id', $vault->id)
    ->orderBy('first_name')
    ->orderBy('last_name')
    ->take(5)
    ->get();
```

**逐行分析**：
1. `Contact::search($term)`：Scout 全文搜索
2. `->where('vault_id', $vault->id)`：在搜索引擎侧过滤 vault
3. **没有** `->where('listed', true)`，但结果仍然只包含 listed=true 的联系人
4. 原因：`shouldBeSearchable()` 在索引收录时排除了 listed=false 的记录

**`shouldBeSearchable()` 如何生效**：
- 创建/更新 Contact 时，Scout observer 调用 `shouldBeSearchable()`
- 返回 true：调用 `searchable()` → 加入/更新索引
- 返回 false：调用 `unsearchable()` → 从索引中移除
- 当 `listed` 字段变化时，模型 saved 事件触发，重新判断是否收录

#### 8.3.5 路径E：全局搜索（收录时过滤）

**代码**：[VaultSearchIndexViewHelper.php:33-35](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultSearchIndexViewHelper.php#L33-L35)

和路径D原理相同，不同的是同时搜索三种模型（Contact/Note/Group）。

#### 8.3.6 五条路径过滤方式对比总结

| 路径 | 过滤方式 | `listed` 检查时机 | 代码可见性 | 漏写风险 |
|------|---------|-----------------|-----------|---------|
| A 主列表 | SQL WHERE | 查询时 | 高（查询处可见） | 低 |
| B 标签筛选 | SQL WHERE | 查询时 | 高（查询处可见） | 低 |
| C 分组成员 | ❌ 无 | - | - | 已遗漏 |
| D 模块搜索 | 索引收录期 | 写入时 | 低（模型方法里） | 中（容易忽视） |
| E 全局搜索 | 索引收录期 | 写入时 | 低（模型方法里） | 中（容易忽视） |

#### 8.3.7 为什么这是个问题？

**场景重现**：
1. 用户在 Vault "家庭" 中创建分组 "家人"，添加 "张三"（listed=true）
2. 张三正常出现在：主列表、搜索结果、分组详情
3. 用户将张三归档（ToggleArchiveContact，listed=false）
4. 张三的 Scout 索引被删除（因为 shouldBeSearchable 返回 false）
5. 现在：
   - 主列表：✅ 看不到张三
   - 标签筛选：✅ 看不到张三
   - 搜索：✅ 搜不到张三
   - 分组详情：❌ **仍然能看到张三**

**用户感知**：
- "我明明归档了这个人，为什么在分组里还能看到？"
- 行为不一致，造成困惑

**Contact 模型其实有 scopeActive 但没用**：
[Contact.php:112-115](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Contact.php#L112-L115)

```php
public function scopeActive(Builder $query): Builder
{
    return $query->where('listed', 1);
}
```

如果路径C改成 `$group->contacts()->active()->...`，就能统一行为。但目前没有用。

---

### 8.4 三者的关系：代码级联调用图

```
用户操作：归档联系人
   │
   ├─ HTTP POST /vaults/{vault}/contacts/{contact}/archive
   │
   ├─ 路由层
   │    ├─ can:vault-viewer,vault → Gate: 用户有 vault 访问权
   │    └─ can:contact-owner,vault,contact → Gate: contact 属于该 vault
   │
   ├─ ContactArchiveController@store
   │    └─ ToggleArchiveContact::execute($data)
   │
   ├─ Service 层（ToggleArchiveContact）
   │    ├─ validateRules()
   │    │   ├─ author_must_belong_to_account → 加载 $author
   │    │   ├─ vault_must_belong_to_account  → 加载 $vault
   │    │   ├─ contact_must_belong_to_vault  → $vault->contacts()->findOrFail()
   │    │   └─ author_must_be_vault_editor   → pivot.permission <= 200
   │    │
   │    └─ $this->contact->listed = !$this->contact->listed
   │       $this->contact->save()
   │
   ├─ 模型事件 → Scout Observer
   │    └─ searchIndexShouldBeUpdated() → ScoutHelper::isActivated() = true
   │         └─ shouldBeSearchable() → 返回新的 listed 值
   │              ├─ true  → searchable()  → 加入/更新索引
   │              └─ false → unsearchable() → 从索引删除
   │
   └─ 后续影响
        ├─ 路径A 主列表：SQL WHERE listed=1 → 不显示 ✅
        ├─ 路径B 标签筛选：SQL WHERE listed=1 → 不显示 ✅
        ├─ 路径C 分组成员：无过滤 → 仍然显示 ❌
        ├─ 路径D 模块搜索：索引中已删除 → 搜不到 ✅
        └─ 路径E 全局搜索：索引中已删除 → 搜不到 ✅
```

---

### 8.5 关键代码位置速查

| 概念 | 文件 | 行号 |
|------|------|------|
| Searchable trait 引入（Contact）| [Contact.php:28](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Contact.php#L28) |
| toSearchableArray | [Contact.php:88-97](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Contact.php#L88-L97) |
| shouldBeSearchable | [Contact.php:104-107](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Contact.php#L104-L107) |
| searchIndexShouldBeUpdated | [Contact.php:134-137](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Contact.php#L134-L137) |
| scopeActive | [Contact.php:112-115](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Models/Contact.php#L112-L115) |
| ScoutHelper::isActivated | [ScoutHelper.php:15-30](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Helpers/ScoutHelper.php#L15-L30) |
| ScoutHelper::id | [ScoutHelper.php:66-84](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Helpers/ScoutHelper.php#L66-L84) |
| group-owner Gate 定义 | [AuthServiceProvider.php:67-76](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Providers/AuthServiceProvider.php#L67-L76) |
| contact-owner Gate 定义 | [AuthServiceProvider.php:56-65](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Providers/AuthServiceProvider.php#L56-L65) |
| 路由 group-owner 中间件 | [web.php:398](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/routes/web.php#L398) |
| 路由 contact-owner 中间件 | [web.php:250](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/routes/web.php#L250) |
| Service group_must_belong_to_vault | [BaseService.php:220-230](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Services/BaseService.php#L220-L230) |
| Service contact_must_belong_to_vault | [BaseService.php:205-215](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Services/BaseService.php#L205-L215) |
| 路径A 主列表 | [ContactController.php:30-31](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactController.php#L30-L31) |
| 路径B 标签筛选 | [ContactLabelController.php:21-26](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageContact/Web/Controllers/ContactLabelController.php#L21-L26) |
| 路径C 分组成员 | [GroupShowViewHelper.php:26-28](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageGroups/Web/ViewHelpers/GroupShowViewHelper.php#L26-L28) |
| 路径D 模块搜索 | [VaultContactSearchViewHelper.php:14-19](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultContactSearchViewHelper.php#L14-L19) |
| 路径E 全局搜索 | [VaultSearchIndexViewHelper.php:33-35](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultSearchIndexViewHelper.php#L33-L35) |
| ToggleArchiveContact | [ToggleArchiveContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/app/Domains/Contact/ManageContact/Services/ToggleArchiveContact.php) |
| Gates 单元测试 | [GatesTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/52-monica/tests/Unit/Controllers/GatesTest.php) |
