# Monica CRM 活动时间线（Activity Timeline）代码解析

Monica CRM 的联系人详情页和 Vault 仪表盘都需要展示"活动时间线"——将笔记、地址、标签、宠物、目标、心情追踪等多种类型的操作记录合并为一条按时间倒序排列的流，并分页展示。本文从领域模型、资源映射层、列表查询三个维度，逐步追踪其代码组织。

---

## 一、两套并行的"时间线"概念

在深入代码之前，需要先厘清 Monica 中存在两套容易混淆但职责不同的"时间线"机制：

| 维度 | Feed（动态流） | Timeline Event（时间线事件） |
|---|---|---|
| 模型 | `ContactFeedItem` | `TimelineEvent` + `LifeEvent` |
| 性质 | 自动产生的操作日志 | 用户手动创建的生活事件记录 |
| 排序依据 | `created_at`（操作发生时间） | `started_at` / `happened_at`（事件发生日期） |
| 作用域 | 单联系人 or 整个 Vault | 单联系人的生活事件模块 |
| 数据库表 | `contact_feed_items` | `timeline_events` + `life_events` + 多张 pivot 表 |
| 入口路由 | `contact.feed.show` / `vault.feed.show` | `contact.timeline_event.index` |

**Feed** 更像是 Git 的 commit log——每次对联系人做了任何修改，系统自动追加一条记录；**Timeline Event** 更像是日记本里的一篇篇记事——用户主动记录"南极之旅"这种可能跨越多天的生活事件。

下文将分别解析，再说明它们在前端如何被统一呈现。

---

## 二、活动领域模型（Activity Domain Model）

### 2.1 Feed 领域：ContactFeedItem

文件：[ContactFeedItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/ContactFeedItem.php)

`ContactFeedItem` 是 Feed 体系的唯一聚合根。其表结构由迁移 [2021_10_19_022411_create_contact_feed_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/database/migrations/2021_10_19_022411_create_contact_feed_table.php) 定义：

```
contact_feed_items
├── id
├── author_id        → users.id        (nullable, 谁做的操作)
├── contact_id       → contacts.id     (哪个联系人)
├── action           → string          (操作类型常量)
├── description       → string          (nullable, 人类可读摘要)
├── feedable_type     → string          (nullable, 多态类型)
├── feedable_id       → bigint          (nullable, 多态 ID)
├── created_at
└── updated_at
```

**核心设计：action 常量 + feedable 多态关联**

- `action` 字段用一个字符串常量表示"发生了什么"，在 `ContactFeedItem` 模型中定义了约 30 种操作类型：

```php
public const ACTION_NOTE_CREATED = 'note_created';
public const ACTION_NOTE_UPDATED = 'note_updated';
public const ACTION_NOTE_DESTROYED = 'note_destroyed';
public const ACTION_PET_CREATED = 'pet_created';
public const ACTION_GOAL_CREATED = 'goal_created';
public const ACTION_LABEL_ASSIGNED = 'label_assigned';
public const ACTION_CONTACT_ADDRESS_CREATED = 'address_created';
public const ACTION_MOOD_TRACKING_EVENT_CREATED = 'mood_tracking_event_added';
// ... 更多
```

- `feedable` 是 Laravel 的多态关联（`MorphTo`），指向被操作的实体对象（Note、Address、Goal、Pet、Label、MoodTrackingEvent、ContactInformation 等）。这使得在渲染 Feed 条目时，可以直接拿到原始对象的完整数据。

```php
public function feedable(): MorphTo
{
    return $this->morphTo();
}
```

**多态关联的反向方向：** 每个可被 Feed 追踪的模型都声明了 `feedItem()` 方法（`MorphOne`），例如：

- [Note.php#L92-L95](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Note.php#L92-L95)
- [Address.php#L61-L64](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Address.php#L61-L64)
- [Goal.php#L60-L63](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Goal.php#L60-L63)
- [ContactInformation.php#L99-L102](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/ContactInformation.php#L99-L102)
- [Label.php#L56-L59](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Label.php#L56-L59)
- [MoodTrackingEvent.php#L63-L66](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/MoodTrackingEvent.php#L63-L66)

这些模型不一定有 `author_id`，但都有 `feedItem()` 方法指向 `ContactFeedItem`，形成一对一的反向关联。

### 2.2 Timeline Event 领域：TimelineEvent + LifeEvent

文件：[TimelineEvent.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/TimelineEvent.php)、[LifeEvent.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/LifeEvent.php)

这是一个两层结构：

```
TimelineEvent (时间线事件，如"南极之旅")
├── id
├── vault_id
├── started_at        → date   (用于排序)
├── label             → string (事件标签)
├── collapsed         → bool
├── participants[]    → BelongsToMany<Contact> (via timeline_event_participants)
└── lifeEvents[]      → HasMany<LifeEvent>     (子事件)
    LifeEvent (生活事件，如"开了100公里"、"吃了披萨")
    ├── id
    ├── timeline_event_id
    ├── life_event_type_id
    ├── emotion_id
    ├── happened_at    → date
    ├── summary, description
    ├── costs, currency_id, paid_by_contact_id
    ├── duration_in_minutes, distance, distance_unit
    ├── from_place, to_place, place
    ├── participants[] → BelongsToMany<Contact> (via life_event_participants)
    └── collapsed      → bool
```

迁移定义在 [2022_05_17_155546_create_life_events_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/database/migrations/2022_05_17_155546_create_life_events_table.php)，涉及 6 张表：

- `life_event_categories` — 事件分类（Vault 级别）
- `life_event_types` — 分类下的具体类型
- `timeline_events` — 时间线事件主表
- `timeline_event_participants` — 时间线事件与联系人的多对多
- `life_events` — 生活事件主表
- `life_event_participants` — 生活事件与联系人的多对多

`TimelineEvent` 模型有一个计算属性 `range`，通过查询其下所有 `LifeEvent` 的最早和最晚 `happened_at`，自动生成日期范围字符串（如"2024-01-01 — 2024-01-15"）：

```php
protected function range(): Attribute
{
    return Attribute::make(
        get: function ($value) {
            $lifeEvents = $this->lifeEvents()->get();
            $firstEvent = $lifeEvents->sortBy('happened_at')->first();
            $lastEvent = $lifeEvents->sortByDesc('happened_at')->first();
            // ...
        }
    );
}
```

### 2.3 Feed 产生的入口：Service 层

Feed 条目不是在 Controller 中直接创建的，而是在各领域 Service 的 `execute()` 方法末尾调用 `createFeedItem()` 私有方法。这是一种 **"写时记录"（Write-Ahead Logging）** 模式——每次对联系人的写操作，都会自动追加一条 Feed 条目。

典型流程以创建笔记为例，见 [CreateNote.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageNotes/Services/CreateNote.php)：

```
execute()
  ├── validateRules()
  ├── Note::create()
  ├── contact->last_updated_at = now()    ← 同时更新联系人的最后编辑时间
  └── createFeedItem()
        ├── ContactFeedItem::create([
        │     'author_id'   => $this->author->id,
        │     'contact_id'  => $this->contact->id,
        │     'action'      => ContactFeedItem::ACTION_NOTE_CREATED,
        │     'description' => Str::words($this->note->body, 10, '…'),
        │   ])
        └── $this->note->feedItem()->save($feedItem)   ← 通过多态关联绑定
```

同样的模式存在于所有产生 Feed 条目的 Service 中：

| Service | Action 常量 | feedable 类型 |
|---|---|---|
| [CreateContactInformation](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactInformation/Services/CreateContactInformation.php) | `ACTION_CONTACT_INFORMATION_CREATED` | ContactInformation |
| [AssociateAddressToContact](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactAddresses/Services/AssociateAddressToContact.php) | `ACTION_CONTACT_ADDRESS_CREATED` | Address |
| [CreateGoal](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageGoals/Services/CreateGoal.php) | `ACTION_GOAL_CREATED` | Goal |
| [CreatePet](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManagePets/Services/CreatePet.php) | `ACTION_PET_CREATED` | Pet |
| [AssignLabel](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageLabels/Services/AssignLabel.php) | `ACTION_LABEL_ASSIGNED` | Label |
| [CreateMoodTrackingEvent](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageMoodTrackingEvents/Services/CreateMoodTrackingEvent.php) | `ACTION_MOOD_TRACKING_EVENT_CREATED` | MoodTrackingEvent |

**关键设计观察：** Feed 条目的创建被散布在各个领域 Service 中，而非集中在某个 Event Listener 里。这意味着如果要新增一种 Feed 来源，必须在该 Service 的 `createFeedItem()` 方法中显式编码。这牺牲了一定的集中管理性，但保证了每个领域对自身 Feed 行为的完全控制。

---

## 三、资源映射层（Resource Mapping Layer）

Monica 使用 Inertia.js（Vue + SRR）作为前端框架，后端不使用 Laravel Resource 类来序列化 Feed 数据，而是通过专门的 **ViewHelper** 类完成映射。整个映射层分为两套独立体系。

### 3.1 Feed 映射体系

核心文件：[ModuleFeedViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php)

映射流程：

```
ContactFeedItem (ORM)
    │
    ▼
ModuleFeedViewHelper::data($items, $user, $vault)
    │
    ├── getSentence($item)     → 根据 action 常量返回人类可读的句子
    ├── getAuthor($item, $vault) → 返回操作者信息（可能已删除用户）
    ├── getData($item, $user)   → 根据 action 类型分发到不同 Action* helper
    │
    ▼
JSON 响应
```

**`getSentence()`** 使用 PHP 8 的 `match` 表达式，将约 30 种 action 常量映射为翻译字符串：

```php
return match ($item->action) {
    'note_created' => trans('wrote a note'),
    'goal_created' => trans('created a goal'),
    'label_assigned' => trans('assigned a label'),
    // ...
};
```

**`getData()`** 使用 `switch` 语句，按 action 类型将数据映射分发到 8 个 Action Feed Helper：

| Action 前缀 | Helper 类 | 产出数据 |
|---|---|---|
| `label_assigned`, `label_removed` | [ActionFeedLabelAssigned](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedLabelAssigned.php) | label object + contact |
| `address_*` | [ActionFeedAddress](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedAddress.php) | address object + contact |
| `contact_information_*` | [ActionFeedContactInformation](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedContactInformation.php) | info object + contact |
| `pet_*` | [ActionFeedPet](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedPet.php) | pet object + contact |
| `note_*` | [ActionFeedNote](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedNote.php) | note object + contact |
| `goal_*` | [ActionFeedGoal](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedGoal.php) | goal object + contact |
| `mood_tracking_event_*` | [ActionFeedMoodTrackingEvent](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedMoodTrackingEvent.php) | mood event + contact |
| 其他（default） | [ActionFeedGenericContactInformation](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedGenericContactInformation.php) | 仅 contact |

**每个 Action Helper 的统一输出结构：**

```json
{
  "<entity_type>": {
    "object": { /* feedable 实体详情, 可能为 null（实体已被删除） */ },
    "description": "来自 ContactFeedItem.description 的摘要"
  },
  "contact": {
    "id": "...",
    "name": "...",
    "age": "...",
    "avatar": { "type": "svg|url", "content": "..." },
    "url": "/vault/{id}/contacts/{id}"
  }
}
```

注意 `feedable` 实体可能为 `null`（比如一个 Note 被删除后，ContactFeedItem 仍存在但 `feedable` 关联查不到对象）。所有 Action Helper 都做了 `null` 安全处理。

### 3.2 Timeline Event 映射体系

核心文件：[ModuleLifeEventViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageLifeEvents/Web/ViewHelpers/ModuleLifeEventViewHelper.php)

映射流程：

```
TimelineEvent (ORM)
    │
    ▼
ModuleLifeEventViewHelper::dtoTimelineEvent($event, $user, $contact)
    │
    ├── id, label, collapsed
    ├── happened_at → $timelineEvent->range (计算属性)
    ├── life_events[] → dtoLifeEvent() 逐一映射
    │     ├── 基础字段: summary, description, happened_at, costs, ...
    │     ├── participants[] → ContactCardHelper::data()
    │     ├── life_event_type → { id, label, category: { id, label } }
    │     └── url: { toggle, edit, destroy }
    └── url: { store, toggle, destroy }
```

`dtoLifeEvent()` 方法（[ModuleLifeEventViewHelper.php#L104-L156](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageLifeEvents/Web/ViewHelpers/ModuleLifeEventViewHelper.php#L104-L156)）展开了 LifeEvent 的所有富字段（费用、距离、时长、起止地点等），以及关联的 Emotion、LifeEventType 及其 Category。

---

## 四、列表查询与分页（List Query & Pagination）

### 4.1 Feed 的查询与分页

有两个入口 Controller，查询逻辑几乎相同，区别仅在于数据范围：

**单联系人 Feed：** [ContactFeedController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/Controllers/ContactFeedController.php)

```php
$items = ContactFeedItem::where('contact_id', $contactId)
    ->with(['author', 'contact' => ['importantDates']])
    ->orderBy('created_at', 'desc')
    ->paginate(15);
```

**Vault 级 Feed：** [VaultFeedController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultFeedController.php)

```php
$contactIds = Contact::where('vault_id', $vaultId)->select('id')->get()->toArray();
$items = ContactFeedItem::whereIn('contact_id', $contactIds)
    ->with(['author', 'contact' => ['importantDates']])
    ->orderBy('created_at', 'desc')
    ->paginate(15);
```

Vault 级 Feed 先查出 Vault 下所有联系人 ID，再用 `whereIn` 过滤。两者都：
- 使用 Eager Loading 预加载 `author` 和 `contact.importantDates`，避免 N+1
- 按 `created_at` 降序排列（最新操作在前）
- 固定每页 15 条

### 4.2 Timeline Event 的查询与分页

[ContactModuleTimelineEventController.php#L18-L31](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageLifeEvents/Web/Controllers/ContactModuleTimelineEventController.php#L18-L31)

```php
$timelineEvents = $contact
    ->timelineEvents()
    ->orderBy('started_at', 'desc')
    ->paginate(15);
```

通过 Contact 模型的 `timelineEvents()` BelongsToMany 关联（pivot 表 `timeline_event_participants`），按 `started_at` 降序排列，同样每页 15 条。

### 4.3 分页元数据

两个查询体系都使用统一的 [PaginatorHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Helpers/PaginatorHelper.php) 来生成分页元信息：

```php
'paginator' => PaginatorHelper::getData($items),
```

`PaginatorHelper::getData()` 封装了 Laravel `LengthAwarePaginator` 的全部元数据：当前页、总页数、每页数量、上/下一页 URL、总条目数等。前端拿到这些信息后自行渲染分页控件。

### 4.4 路由注册

Feed 和 Timeline Event 的路由在 [routes/web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/routes/web.php) 中注册：

```php
// Vault 级 Feed
Route::get('feed', [VaultFeedController::class, 'show'])->name('vault.feed.show');

// 联系人级 Feed
Route::get('feed', [ContactFeedController::class, 'show'])->name('contact.feed.show');

// 联系人 Timeline Events (CRUD)
Route::get('timelineEvents', [...])->name('contact.timeline_event.index');
Route::post('timelineEvents', [...])->name('contact.timeline_event.store');
Route::post('timelineEvents/{timelineEvent}/toggle', [...])->name('contact.timeline_event.toggle');
Route::delete('timelineEvents/{timelineEvent}', [...])->name('contact.timeline_event.destroy');

// 联系人 Life Events (CRUD, 嵌套在 TimelineEvent 下)
Route::post('timelineEvents/{timelineEvent}/lifeEvents', [...])->name('contact.life_event.store');
Route::put('timelineEvents/{timelineEvent}/lifeEvents/{lifeEvent}', [...])->name('contact.life_event.edit');
Route::post('.../lifeEvents/{lifeEvent}/toggle', [...])->name('contact.life_event.toggle');
Route::delete('.../lifeEvents/{lifeEvent}', [...])->name('contact.life_event.destroy');
```

---

## 五、Vault 仪表盘：统一呈现

在 [VaultController::show()](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L66-L89) 中，Vault 仪表盘同时预加载了 Feed 和 Timeline Event 的初始数据：

```php
return Inertia::render('Vault/Dashboard/Index', [
    'lastUpdatedContacts' => ...,
    'upcomingReminders'   => ...,
    'favorites'           => ...,
    'dueTasks'            => ...,
    'moodTrackingEvents'  => ...,
    'defaultTab'          => $vault->default_activity_tab,  // ← 用户选择的默认 Tab
    'lifeEvents'          => ModuleLifeEventViewHelper::data(...),  // Timeline Event 初始数据
    'lifeMetrics'         => ...,
    'url' => [
        'feed' => route('vault.feed.show', ['vault' => $vault]),  // Feed 异步加载端点
        'default_tab' => route('vault.default_tab.update', ...),
    ],
]);
```

**`default_activity_tab`** 字段（见 [UpdateVaultDashboardDefaultTab](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Vault/ManageVault/Services/UpdateVaultDashboardDefaultTab.php)）允许用户选择仪表盘默认展示哪个 Tab（Feed 还是 Timeline Event），其值存储在 `vaults.default_activity_tab` 中。

前端通过 Inertia 拿到初始数据和异步加载的 URL 后，使用 Vue 组件渲染 Tab 切换界面：Feed Tab 通过 AJAX 调用 `vault.feed.show` 或 `contact.feed.show` 加载更多分页数据；Timeline Event Tab 通过 `contact.timeline_event.index` 加载。

---

## 六、代码协作全景图

下面将整个数据流梳理为一条完整的链路：

```
┌─────────────────────────── 写操作（产生 Feed） ───────────────────────────┐
│                                                                           │
│  用户操作 → Controller → Service::execute()                              │
│                          ├── 业务逻辑（创建/更新/删除实体）                │
│                          ├── contact->last_updated_at = now()             │
│                          └── createFeedItem()                             │
│                                ├── ContactFeedItem::create(...)            │
│                                └── $entity->feedItem()->save($feedItem)   │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────── 读操作（展示时间线） ──────────────────────────┐
│                                                                           │
│  前端请求 → Controller                                                    │
│              ├── ContactFeedController (单联系人 Feed)                     │
│              │     ContactFeedItem::where('contact_id', ...)              │
│              │       ->with(['author', 'contact.importantDates'])         │
│              │       ->orderBy('created_at', 'desc')->paginate(15)        │
│              │                                                           │
│              ├── VaultFeedController (Vault 级 Feed)                      │
│              │     ContactFeedItem::whereIn('contact_id', [...])          │
│              │       ->with(['author', 'contact.importantDates'])         │
│              │       ->orderBy('created_at', 'desc')->paginate(15)        │
│              │                                                           │
│              └── ContactModuleTimelineEventController (生活事件)            │
│                    $contact->timelineEvents()                             │
│                      ->orderBy('started_at', 'desc')->paginate(15)       │
│                                                                           │
│            → ModuleFeedViewHelper / ModuleLifeEventViewHelper              │
│                ├── getSentence() → match(action) → trans(...)             │
│                ├── getAuthor()  → UserHelper / 默认 Monica SVG            │
│                └── getData()    → switch(action) → ActionFeed* Helper     │
│                                                                           │
│            → PaginatorHelper::getData($paginator)                         │
│                                                                           │
│            → JSON Response { data: {...}, paginator: {...} }              │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 七、设计要点总结

1. **Feed 是"操作日志"，Timeline Event 是"生活记事"**——前者由系统自动产生，后者由用户手动创建。两者在模型、查询、映射层都完全独立，仅在 Vault 仪表盘前端通过 Tab 切换统一呈现。

2. **多态关联实现统一 Feed 表**：`ContactFeedItem.feedable` 使用 Laravel 的 `nullableNumericMorphs`，将 Note、Address、Goal、Pet、Label、ContactInformation、MoodTrackingEvent 等不同实体统一收纳进同一张 `contact_feed_items` 表。这是"不同类型活动条目合并成时间线"的核心数据库机制。

3. **写时记录模式**：Feed 条目的创建散布在各领域 Service 中（而非事件监听器），每个 Service 在完成业务操作后显式调用 `createFeedItem()`。这保证了领域自治，但新增 Feed 来源时需要侵入对应 Service。

4. **ViewHelper 替代 Resource**：Monica 使用 ViewHelper 模式（静态方法）而非 Laravel Eloquent Resource 来完成 ORM → JSON 的映射。`ModuleFeedViewHelper` 充当调度中心，根据 `action` 类型分发到 8 个 ActionFeed* 子 Helper，每个子 Helper 只负责一种实体类型的序列化。

5. **排序策略差异**：Feed 按 `created_at` 倒序（操作发生时间），Timeline Event 按 `started_at` 倒序（事件开始日期）。这反映了两种时间线的本质区别——一个关注"何时操作的"，另一个关注"何时发生的"。

6. **分页统一**：Feed 和 Timeline Event 都固定每页 15 条，都通过 `PaginatorHelper` 生成标准化的分页元数据。Vault 级 Feed 通过 `whereIn('contact_id', ...)` 聚合 Vault 下所有联系人的动态。

7. **`default_activity_tab`**：Vault 模型上的 `default_activity_tab` 字段让用户可以选择仪表盘默认展示 Feed 还是 Timeline Event，该偏好通过 `UpdateVaultDashboardDefaultTab` Service 持久化到 `vaults` 表。
