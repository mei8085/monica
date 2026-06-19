# 联系人搜索索引与过滤条件链路分析

本文档详细分析 Monica 系统中「联系人搜索索引」与「过滤/权限/列表」的协作方式，解释为什么两者配合起来令人困惑。

---

## 一、搜索索引系统（Laravel Scout）

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

## 六、验证清单（排查搜索/过滤问题时）

1. **确认 Scout 驱动**：`SCOUT_DRIVER` 环境变量 → `ScoutHelper::isActivated()` 是否返回 true
2. **索引是否最新**：如果是 meilisearch/typesense，运行 `scout:setup --flush --import`
3. **listed 字段**：检查目标联系人 `listed=1`，否则不会进入搜索索引
4. **中间件是否保护路由**：路由分组是否在 `can:vault-viewer,vault` 之内
5. **Scout 的 where 条件**：搜索代码是否包含 `where('vault_id', $vault->id)`
6. **Meilisearch/Typesense filterableAttributes**：`scout.php` 中是否声明了 `vault_id` 可过滤
7. **路径一致性**：如果用分组/标签，注意路径C缺失了 `listed` 过滤
8. **权限数字对比**：用户在 pivot 中的 `permission` 值是否 <= 所需阈值（300/200/100）
