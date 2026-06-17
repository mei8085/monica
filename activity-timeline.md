# Monica CRM 活动时间线（Activity Timeline）代码解析

Monica CRM 的联系人详情页和 Vault 仪表盘都需要展示"活动时间线"——将笔记、地址、标签、宠物、目标、心情追踪、联系人更新、重要日期、群组、文章、头像、收藏归档等多种类型的操作记录合并为一条按时间倒序排列的流，并分页展示。本文从领域模型、动态来源、资源映射层、列表查询四个维度，逐步追踪其代码组织。

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

- `action` 字段用一个字符串常量表示"发生了什么"，在 `ContactFeedItem` 模型中定义了 **30 种操作类型**（完整列表见下文 2.3 节）。

- `feedable` 是 Laravel 的多态关联（`MorphTo`），指向被操作的实体对象。这使得在渲染 Feed 条目时，可以直接拿到原始对象的完整数据。

```php
public function feedable(): MorphTo
{
    return $this->morphTo();
}
```

**多态关联的反向方向：** 每个可被 Feed 追踪的模型都声明了 `feedItem()` 方法（`MorphOne`），形成一对一的反向关联。目前共有 **11 个模型** 拥有此关联：

| 模型 | 文件位置 |
|---|---|
| `Note` | [Note.php#L92-L95](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Note.php#L92-L95) |
| `Address` | [Address.php#L61-L64](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Address.php#L61-L64) |
| `Goal` | [Goal.php#L60-L63](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Goal.php#L60-L63) |
| `ContactInformation` | [ContactInformation.php#L99-L102](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/ContactInformation.php#L99-L102) |
| `Label` | [Label.php#L56-L59](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Label.php#L56-L59) |
| `MoodTrackingEvent` | [MoodTrackingEvent.php#L63-L66](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/MoodTrackingEvent.php#L63-L66) |
| `Pet` | [Pet.php#L56-L59](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Pet.php#L56-L59) |
| `ContactImportantDate` | [ContactImportantDate.php#L95-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/ContactImportantDate.php#L95-L98) |
| `Group` | [Group.php#L98-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Group.php#L98-L101) |
| `Post` | [Post.php#L92-L95](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Post.php#L92-L95) |
| `Loan` | [Loan.php#L76-L79](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Loan.php#L76-L79) |

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

迁移定义在 [2022_05_17_155546_create_life_events_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/database/migrations/2022_05_17_155546_create_life_events_table.php)，涉及 6 张表。

`TimelineEvent` 模型有一个计算属性 `range`，通过查询其下所有 `LifeEvent` 的最早和最晚 `happened_at`，自动生成日期范围字符串。

### 2.3 Feed 产生的完整入口：30 种操作 × Service 层

Feed 条目不是在 Controller 中直接创建的，而是在各领域 Service 的 `execute()` 方法末尾调用 `createFeedItem()` 私有方法。这是一种 **"写时记录"（Write-Ahead Logging）** 模式——每次对联系人的写操作，都会自动追加一条 Feed 条目。

#### 2.3.1 典型流程（以创建笔记为例）

[CreateNote.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageNotes/Services/CreateNote.php):

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

#### 2.3.2 完整的 30 种 Feed 来源清单

`ContactFeedItem` 模型中定义了 30 个 `ACTION_*` 常量，对应 30 种操作类型。这些操作分布在 26 个 Service 类中：

| 操作分类 | Action 常量 | Service 文件 | 是否绑定 feedable |
|---|---|---|---|
| **联系人基础操作** | | | |
| 创建联系人 | `ACTION_CONTACT_CREATED` | [CreateContact.php#L121-L128](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/CreateContact.php#L121-L128) | 否 |
| 更新联系人信息 | `ACTION_INFORMATION_UPDATED` | [UpdateContact.php#L111-L118](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/UpdateContact.php#L111-L118) | 否 |
| 归档联系人 | `ACTION_ARCHIVED_CONTACT` | [ToggleArchiveContact.php#L69-L76](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/ToggleArchiveContact.php#L69-L76) | 否 |
| 取消归档联系人 | `ACTION_UNARCHIVED_CONTACT` | [ToggleArchiveContact.php#L69-L76](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/ToggleArchiveContact.php#L69-L76) | 否 |
| 收藏联系人 | `ACTION_FAVORITED_CONTACT` | [ToggleFavoriteContact.php#L98-L105](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/ToggleFavoriteContact.php#L98-L105) | 否 |
| 取消收藏联系人 | `ACTION_UNFAVORITED_CONTACT` | [ToggleFavoriteContact.php#L98-L105](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/ToggleFavoriteContact.php#L98-L105) | 否 |
| 更新头像 | `ACTION_CHANGE_AVATAR` | [UpdatePhotoAsAvatar.php#L91-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageAvatar/Services/UpdatePhotoAsAvatar.php#L91-L98) | 否 |
| 删除头像 | `ACTION_CHANGE_AVATAR` | [DestroyAvatar.php#L76-L83](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageAvatar/Services/DestroyAvatar.php#L76-L83) | 否 |
| 更新工作信息 | `ACTION_JOB_INFORMATION_UPDATED` | [UpdateJobInformation.php#L62-L66](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageJobInformation/Services/UpdateJobInformation.php#L62-L66) | 否 |
| 更新宗教信仰 | `ACTION_RELIGION_UPDATED` | [UpdateReligion.php#L58-L62](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageReligion/Services/UpdateReligion.php#L58-L62) | 否 |
| **重要日期** | | | |
| 创建重要日期 | `ACTION_IMPORTANT_DATE_CREATED` | [CreateContactImportantDate.php#L89-L99](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactImportantDates/Services/CreateContactImportantDate.php#L89-L99) | 是 → ContactImportantDate |
| 更新重要日期 | `ACTION_IMPORTANT_DATE_UPDATED` | [UpdateContactImportantDate.php#L90-L100](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactImportantDates/Services/UpdateContactImportantDate.php#L90-L100) | 是 → ContactImportantDate |
| 删除重要日期 | `ACTION_IMPORTANT_DATE_DESTROYED` | [DestroyContactImportantDate.php#L65-L73](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactImportantDates/Services/DestroyContactImportantDate.php#L65-L73) | 否（实体已删） |
| **笔记** | | | |
| 创建笔记 | `ACTION_NOTE_CREATED` | [CreateNote.php#L74-L83](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageNotes/Services/CreateNote.php#L74-L83) | 是 → Note |
| 更新笔记 | `ACTION_NOTE_UPDATED` | [UpdateNote.php#L74-L83](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageNotes/Services/UpdateNote.php#L74-L83) | 是 → Note |
| 删除笔记 | `ACTION_NOTE_DESTROYED` | [DestroyNote.php#L60-L68](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageNotes/Services/DestroyNote.php#L60-L68) | 否（先记再删） |
| **联系方式** | | | |
| 创建联系方式 | `ACTION_CONTACT_INFORMATION_CREATED` | [CreateContactInformation.php#L87-L96](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactInformation/Services/CreateContactInformation.php#L87-L96) | 是 → ContactInformation |
| 更新联系方式 | `ACTION_CONTACT_INFORMATION_UPDATED` | [UpdateContactInformation.php#L86-L95](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactInformation/Services/UpdateContactInformation.php#L86-L95) | 是 → ContactInformation |
| 删除联系方式 | `ACTION_CONTACT_INFORMATION_DESTROYED` | [DestroyContactInformation.php#L76-L84](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactInformation/Services/DestroyContactInformation.php#L76-L84) | 否（实体已删） |
| **地址** | | | |
| 添加地址 | `ACTION_CONTACT_ADDRESS_CREATED` | [AssociateAddressToContact.php#L73-L86](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactAddresses/Services/AssociateAddressToContact.php#L73-L86) | 是 → Address |
| 更新地址 | `ACTION_CONTACT_ADDRESS_UPDATED` | *（无独立 Service，地址本身在 Address 管理）* | 是 → Address |
| 删除地址 | `ACTION_CONTACT_ADDRESS_DESTROYED` | [RemoveAddressFromContact.php#L74-L87](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactAddresses/Services/RemoveAddressFromContact.php#L74-L87) | 否（实体已删） |
| **标签** | | | |
| 分配标签 | `ACTION_LABEL_ASSIGNED` | [AssignLabel.php#L71-L78](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageLabels/Services/AssignLabel.php#L71-L78) | 是 → Label |
| 移除标签 | `ACTION_LABEL_REMOVED` | [RemoveLabel.php#L62-L71](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageLabels/Services/RemoveLabel.php#L62-L71) | 是 → Label |
| **宠物** | | | |
| 创建宠物 | `ACTION_PET_CREATED` | [CreatePet.php#L79-L88](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManagePets/Services/CreatePet.php#L79-L88) | 是 → Pet |
| 更新宠物 | `ACTION_PET_UPDATED` | [UpdatePet.php#L72-L80](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManagePets/Services/UpdatePet.php#L72-L80) | 是 → Pet |
| 删除宠物 | `ACTION_PET_DESTROYED` | [DestroyPet.php#L65-L73](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManagePets/Services/DestroyPet.php#L65-L73) | 否（实体已删） |
| **目标** | | | |
| 创建目标 | `ACTION_GOAL_CREATED` | [CreateGoal.php#L76-L85](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageGoals/Services/CreateGoal.php#L76-L85) | 是 → Goal |
| 更新目标 | `ACTION_GOAL_UPDATED` | [UpdateGoal.php#L77-L85](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageGoals/Services/UpdateGoal.php#L77-L85) | 是 → Goal |
| 删除目标 | `ACTION_GOAL_DESTROYED` | [DestroyGoal.php#L66-L74](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageGoals/Services/DestroyGoal.php#L66-L74) | 否（实体已删） |
| **群组** | | | |
| 加入群组 | `ACTION_ADDED_TO_GROUP` | [AddContactToGroup.php#L84-L93](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageGroups/Services/AddContactToGroup.php#L84-L93) | 是 → Group |
| 离开群组 | `ACTION_REMOVED_FROM_GROUP` | [RemoveContactFromGroup.php#L62-L71](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageGroups/Services/RemoveContactFromGroup.php#L62-L71) | 是 → Group |
| **文章** | | | |
| 加入文章 | `ACTION_ADDED_TO_POST` | [AddContactToPost.php#L78-L87](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Vault/ManageJournals/Services/AddContactToPost.php#L78-L87) | 是 → Post |
| 离开文章 | `ACTION_REMOVED_FROM_POST` | [RemoveContactFromPost.php#L80-L89](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Vault/ManageJournals/Services/RemoveContactFromPost.php#L80-L89) | 是 → Post |
| **心情追踪** | | | |
| 添加心情 | `ACTION_MOOD_TRACKING_EVENT_CREATED` | [CreateMoodTrackingEvent.php#L67-L75](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageMoodTrackingEvents/Services/CreateMoodTrackingEvent.php#L67-L75) | 是 → MoodTrackingEvent |
| 更新心情 | `ACTION_MOOD_TRACKING_EVENT_UPDATED` | [UpdateMoodTrackingEvent.php#L67-L75](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageMoodTrackingEvents/Services/UpdateMoodTrackingEvent.php#L67-L75) | 是 → MoodTrackingEvent |
| 删除心情 | `ACTION_MOOD_TRACKING_EVENT_DESTROYED` | [DestroyMoodTrackingEvent.php#L63-L71](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageMoodTrackingEvents/Services/DestroyMoodTrackingEvent.php#L63-L71) | 否（实体已删） |
| **借贷** | | | |
| 创建借贷 | `ACTION_LOAN_CREATED` | *（注：CreateLoan 中未创建 FeedItem，存在遗漏）* | - |
| 更新借贷 | `ACTION_LOAN_UPDATED` | *（存在常量但未被使用）* | - |

#### 2.3.3 Feed 产生的三种模式

通过分析 26 个 Service 的 `createFeedItem()` 方法，可以归纳出 **三种产生模式**：

**模式一：有 feedable 绑定（大多数 create/update 操作）**

操作对象实体存在，通过多态关联绑定 FeedItem 到实体。例如 [CreateNote.php#L74-L83](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageNotes/Services/CreateNote.php#L74-L83):

```php
private function createFeedItem(): void
{
    $feedItem = ContactFeedItem::create([
        'author_id'   => $this->author->id,
        'contact_id'  => $this->contact->id,
        'action'      => ContactFeedItem::ACTION_NOTE_CREATED,
        'description' => Str::words($this->note->body, 10, '…'),
    ]);
    $this->note->feedItem()->save($feedItem);  // ← 绑定多态关联
}
```

**模式二：无 feedable 绑定（联系人基础操作）**

针对联系人本身的操作（更新信息、收藏、归档、头像），没有对应的实体对象，只记录动作本身。例如 [UpdateContact.php#L111-L118](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/UpdateContact.php#L111-L118):

```php
private function createFeedItem(): void
{
    ContactFeedItem::create([
        'author_id'  => $this->author->id,
        'contact_id' => $this->contact->id,
        'action'     => ContactFeedItem::ACTION_INFORMATION_UPDATED,
        // 无 feedable 绑定，也无 description
    ]);
}
```

**模式三：删除前记录 Feed（DestroyNote 的特殊处理）**

对于删除操作，有两种子模式：

- **先删后记**（大多数 destroy 操作）：先删除实体，再创建 FeedItem，此时 feedable 关联无法绑定（实体已不存在）。例如 [DestroyContactInformation.php#L76-L84](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactInformation/Services/DestroyContactInformation.php#L76-L84)。

- **先记后删**（仅 DestroyNote）：先创建 FeedItem 并绑定 feedable，再删除实体。这种方式保证了 feedable_id/type 被写入数据库，虽然后续查询时 feedable 关联会返回 null，但 id 仍然保留。见 [DestroyNote.php#L46-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageNotes/Services/DestroyNote.php#L46-L58):

```php
public function execute(array $data): void
{
    $this->validateRules($data);
    $this->note = $this->contact->notes()->findOrFail($data['note_id']);

    $this->createFeedItem();  // ← 先记录 Feed（此时 note 还存在）
    $this->note->delete();    // ← 再删除实体

    $this->contact->last_updated_at = Carbon::now();
    $this->contact->save();
}
```

#### 2.3.4 特殊的双向动态操作

某些操作具有"切换"性质，同一个 Service 会根据状态产生不同的 action：

**Toggle 模式——收藏/取消收藏**（[ToggleFavoriteContact.php#L98-L105](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/ToggleFavoriteContact.php#L98-L105)）：

```php
private function createFeedItem(): void
{
    ContactFeedItem::create([
        'author_id'  => $this->author->id,
        'contact_id' => $this->contact->id,
        // 三目运算：当前是收藏状态 → 产生"取消收藏"action，反之亦然
        'action'     => $this->isFavorite
            ? ContactFeedItem::ACTION_UNFAVORITED_CONTACT
            : ContactFeedItem::ACTION_FAVORITED_CONTACT,
    ]);
}
```

**Toggle 模式——归档/取消归档**（[ToggleArchiveContact.php#L69-L76](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/ToggleArchiveContact.php#L69-L76)）：

```php
private function createFeedItem(): void
{
    ContactFeedItem::create([
        'author_id'  => $this->author->id,
        'contact_id' => $this->contact->id,
        'action'     => $this->contact->refresh()->listed
            ? ContactFeedItem::ACTION_UNARCHIVED_CONTACT
            : ContactFeedItem::ACTION_ARCHIVED_CONTACT,
    ]);
}
```

**关键设计观察：** Feed 条目的创建被散布在各个领域 Service 中，而非集中在某个 Event Listener 里。这意味着如果要新增一种 Feed 来源，必须在该 Service 的 `createFeedItem()` 方法中显式编码。这牺牲了一定的集中管理性，但保证了每个领域对自身 Feed 行为的完全控制。

---

## 三、资源映射层（Resource Mapping Layer）

Monica 使用 Inertia.js（Vue + SSR）作为前端框架，后端不使用 Laravel Resource 类来序列化 Feed 数据，而是通过专门的 **ViewHelper** 类完成映射。整个映射层分为两套独立体系。

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

#### 3.1.1 动态句子生成：getSentence()

`getSentence()` 使用 PHP 8 的 `match` 表达式，将 **30 种 action 常量** 映射为翻译字符串（[ModuleFeedViewHelper.php#L37-L77](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L37-L77)）：

```php
private static function getSentence(ContactFeedItem $item): mixed
{
    return match ($item->action) {
        'contact_created'              => trans('created the contact'),
        'author_deleted'               => trans('Deleted author'),
        'information_updated'          => trans('updated the contact information'),
        'important_date_created'       => trans('added an important date'),
        'important_date_updated'       => trans('updated an important date'),
        'important_date_destroyed'     => trans('deleted an important date'),
        'address_created'              => trans('added an address'),
        'address_updated'              => trans('updated an address'),
        'address_destroyed'            => trans('deleted an address'),
        'pet_created'                  => trans('added a pet'),
        'pet_updated'                  => trans('updated a pet'),
        'pet_destroyed'                => trans('deleted a pet'),
        'contact_information_created'  => trans('added a contact information'),
        'contact_information_updated'  => trans('updated a contact information'),
        'contact_information_destroyed'=> trans('deleted a contact information'),
        'label_assigned'               => trans('assigned a label'),
        'label_removed'                => trans('removed a label'),
        'note_created'                 => trans('wrote a note'),
        'note_updated'                 => trans('edited a note'),
        'note_destroyed'               => trans('deleted a note'),
        'job_information_updated'      => trans('updated the job information'),
        'religion_updated'             => trans('updated the religion'),
        'goal_created'                 => trans('created a goal'),
        'goal_updated'                 => trans('updated a goal'),
        'goal_destroyed'               => trans('deleted a goal'),
        'added_to_group'               => trans('added the contact to a group'),
        'removed_from_group'           => trans('removed the contact from a group'),
        'added_to_post'                => trans('added the contact to a post'),
        'removed_from_post'            => trans('removed the contact from a post'),
        'archived'                     => trans('archived the contact'),
        'unarchived'                   => trans('unarchived the contact'),
        'favorited'                    => trans('added the contact to the favorites'),
        'unfavorited'                  => trans('removed the contact from the favorites'),
        'changed_avatar'               => trans('updated the avatar of the contact'),
        'mood_tracking_event_added'    => trans('logged the mood'),
        default                        => trans('unknown action'),
    };
}
```

注意几点设计：
- 这里使用了硬编码字符串匹配（如 `'note_created'`），而非引用 `ContactFeedItem::ACTION_NOTE_CREATED` 常量。这在新增 action 时需要同时修改两处。
- 有一个 `'author_deleted'` 的特殊处理，但在 Service 代码中未找到其产生来源。
- `default` 兜底分支返回 "unknown action"，保证向后兼容性。

#### 3.1.2 操作者信息：getAuthor()

`getAuthor()` 处理操作者可能已被删除的情况（[ModuleFeedViewHelper.php#L79-L105](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L79-L105)）：

- 如果 `$item->author` 存在 → 通过 `UserHelper::getInformationAboutContact()` 返回正常用户信息
- 如果 `$item->author` 为 `null` → 返回一个内置的 Monica SVG 头像和 "Deleted author" 文本

#### 3.1.3 数据映射分发：getData()

`getData()` 使用 `switch` 语句，按 action 类型将数据映射分发到 **8 个 Action Feed Helper**（[ModuleFeedViewHelper.php#L107-L147](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L107-L147)）：

```php
private static function getData(ContactFeedItem $item, User $user)
{
    switch ($item->action) {
        case 'label_assigned':
        case 'label_removed':
            return ActionFeedLabelAssigned::data($item);

        case 'address_created':
        case 'address_updated':
        case 'address_destroyed':
            return ActionFeedAddress::data($item, $user);

        case 'contact_information_created':
        case 'contact_information_updated':
        case 'contact_information_destroyed':
            return ActionFeedContactInformation::data($item);

        case 'pet_created':
        case 'pet_updated':
        case 'pet_destroyed':
            return ActionFeedPet::data($item);

        case 'note_created':
        case 'note_updated':
        case 'note_destroyed':
            return ActionFeedNote::data($item);

        case 'goal_created':
        case 'goal_updated':
        case 'goal_destroyed':
            return ActionFeedGoal::data($item);

        case 'mood_tracking_event_added':
        case 'mood_tracking_event_updated':
        case 'mood_tracking_event_deleted':
            return ActionFeedMoodTrackingEvent::data($item, $user);

        default:
            return ActionFeedGenericContactInformation::data($item);
    }
}
```

#### 3.1.4 完整的 Action → Helper 映射表

| Action 类型 | Helper 类 | 产出数据结构 |
|---|---|---|
| `label_assigned`, `label_removed` | [ActionFeedLabelAssigned](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedLabelAssigned.php) | `{ label: { object, description }, contact }` |
| `address_created`, `address_updated`, `address_destroyed` | [ActionFeedAddress](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedAddress.php) | `{ address: { object, description }, contact }` |
| `contact_information_created/updated/destroyed` | [ActionFeedContactInformation](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedContactInformation.php) | `{ information: { object, description }, contact }` |
| `pet_created`, `pet_updated`, `pet_destroyed` | [ActionFeedPet](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedPet.php) | `{ pet: { object, description }, contact }` |
| `note_created`, `note_updated`, `note_destroyed` | [ActionFeedNote](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedNote.php) | `{ note: { object, description }, contact }` |
| `goal_created`, `goal_updated`, `goal_destroyed` | [ActionFeedGoal](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedGoal.php) | `{ goal: { object, description }, contact }` |
| `mood_tracking_event_added/updated/deleted` | [ActionFeedMoodTrackingEvent](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedMoodTrackingEvent.php) | `{ mood_tracking_event: { object, description }, contact }` |
| `important_date_created/updated/destroyed` | **default → ActionFeedGenericContactInformation** | `{ contact }`（无实体详情） |
| `added_to_group`, `removed_from_group` | **default → ActionFeedGenericContactInformation** | `{ contact }`（无实体详情） |
| `added_to_post`, `removed_from_post` | **default → ActionFeedGenericContactInformation** | `{ contact }`（无实体详情） |
| `contact_created`, `information_updated` | **default → ActionFeedGenericContactInformation** | `{ contact }`（无实体详情） |
| `job_information_updated`, `religion_updated` | **default → ActionFeedGenericContactInformation** | `{ contact }`（无实体详情） |
| `archived`, `unarchived`, `favorited`, `unfavorited` | **default → ActionFeedGenericContactInformation** | `{ contact }`（无实体详情） |
| `changed_avatar` | **default → ActionFeedGenericContactInformation** | `{ contact }`（无实体详情） |
| `loan_created`, `loan_updated` | **default → ActionFeedGenericContactInformation** | `{ contact }`（无实体详情） |
| 未知 action | **default → ActionFeedGenericContactInformation** | `{ contact }`（无实体详情） |

#### 3.1.5 通用动作映射：ActionFeedGenericContactInformation

**约 17 种 action** 会进入 `default` 分支，使用 [ActionFeedGenericContactInformation](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedGenericContactInformation.php) 作为兜底映射：

```php
public static function data(ContactFeedItem $item): array
{
    $contact = $item->contact;

    return [
        'id'      => $contact->id,
        'name'    => $contact->name,
        'age'     => $contact->age,
        'avatar'  => $contact->avatar,
        'url'     => route('contact.show', [
            'vault'   => $contact->vault_id,
            'contact' => $contact->id,
        ]),
    ];
}
```

它只返回联系人的基础信息（id、姓名、年龄、头像、跳转链接），不包含被操作实体的详情。这意味着：

- **重要日期的 Feed 条目在前端无法看到日期的具体内容**（虽然数据库中 `description` 字段保存了"标签 + 日期"文本）
- **群组/文章的 Feed 条目无法看到群组/文章的名称**（虽然 `description` 字段保存了名称）
- 所有联系人基础操作（更新信息、收藏、归档、头像）的 Feed 条目只显示联系人信息

**注意：** 实际上 `ContactFeedItem.description` 字段保存了这些信息（例如重要日期的"生日 1990-01-01"），但 `ActionFeedGenericContactInformation` 没有读取和返回该字段。这是一个可以改进的点——可以在通用 Helper 中加入 `description` 字段的返回。

#### 3.1.6 每个 Action Helper 的统一输出结构

对于有专属 Helper 的 action，输出结构统一为：

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

### 6.1 完整数据流

```
┌─────────────────────────── 写操作（产生 Feed） ───────────────────────────┐
│                                                                           │
│  用户操作 → Controller → Service::execute()                              │
│                          ├── 业务逻辑（创建/更新/删除实体）                │
│                          ├── contact->last_updated_at = now()             │
│                          └── createFeedItem()                             │
│                                ├── ContactFeedItem::create(...)            │
│                                │   ├── 有实体：$entity->feedItem()->save() │
│                                │   ├── 无实体：仅保存基本信息              │
│                                │   └── 删除操作：先记后删 or 先删后记       │
│                                └── 更新 last_updated_at                   │
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
│                ├── getSentence() → match(action) → trans(...)  (30 种)    │
│                ├── getAuthor()  → UserHelper / 默认 Monica SVG            │
│                └── getData()    → switch(action) → 8 个 ActionFeed* Helper│
│                                                                           │
│            → PaginatorHelper::getData($paginator)                         │
│                                                                           │
│            → JSON Response { data: {...}, paginator: {...} }              │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

### 6.2 30 种操作 × 3 种模式 × 2 种映射方式

```
Action 常量 (30 种)
    │
    ├─► Service::createFeedItem() (26 个 Service)
    │     │
    │     ├─ 模式一：有 feedable 绑定 (17 种 action)
    │     │     └─► Note, Address, Goal, Pet, Label, ContactInformation,
    │     │         MoodTrackingEvent, ContactImportantDate, Group, Post, Loan
    │     │
    │     ├─ 模式二：无 feedable 绑定 (10 种 action)
    │     │     └─► 联系人基础操作: 创建/更新/收藏/归档/头像/工作/宗教
    │     │
    │     └─ 模式三：删除操作 (10 种 action)
    │           ├─ 先记后删 (仅 DestroyNote)
    │           └─ 先删后记 (其余 9 种)
    │
    └─► ModuleFeedViewHelper::getData()
          │
          ├─ 专属 Helper (13 种 action → 7 个 Helper)
          │     ├─ ActionFeedNote, ActionFeedAddress, ActionFeedGoal
          │     ├─ ActionFeedPet, ActionFeedLabelAssigned
          │     ├─ ActionFeedContactInformation
          │     └─ ActionFeedMoodTrackingEvent
          │
          └─ 通用 Helper (17 种 action → ActionFeedGenericContactInformation)
                └─ 仅返回联系人基础信息 (无实体详情)
```

---

## 七、设计要点总结

1. **Feed 是"操作日志"，Timeline Event 是"生活记事"**——前者由系统自动产生，后者由用户手动创建。两者在模型、查询、映射层都完全独立，仅在 Vault 仪表盘前端通过 Tab 切换统一呈现。

2. **多态关联实现统一 Feed 表**：`ContactFeedItem.feedable` 使用 Laravel 的 `nullableNumericMorphs`，将 11 种不同实体（Note、Address、Goal、Pet、Label、ContactInformation、MoodTrackingEvent、ContactImportantDate、Group、Post、Loan）统一收纳进同一张 `contact_feed_items` 表。这是"不同类型活动条目合并成时间线"的核心数据库机制。

3. **30 种操作 × 26 个 Service**：Feed 条目的创建散布在各领域 Service 中（而非事件监听器），每个 Service 在完成业务操作后显式调用 `createFeedItem()`。这保证了领域自治，但新增 Feed 来源时需要侵入对应 Service。

4. **三种 Feed 产生模式**：
   - 有 feedable 绑定（大多数 create/update）
   - 无 feedable 绑定（联系人基础操作）
   - 删除操作（先记后删 / 先删后记）

5. **ViewHelper 替代 Resource**：Monica 使用 ViewHelper 模式而非 Laravel Eloquent Resource 完成 ORM → JSON 映射。`ModuleFeedViewHelper` 充当调度中心，根据 `action` 类型分发到 8 个 ActionFeed* 子 Helper。

6. **映射层的覆盖缺口**：约 17 种 action（重要日期、群组、文章、联系人基础操作等）进入 `default` 分支使用 `ActionFeedGenericContactInformation`，仅返回联系人基础信息，丢失了 `description` 字段中已保存的实体摘要。这是可以改进的设计点。

7. **排序策略差异**：Feed 按 `created_at` 倒序（操作发生时间），Timeline Event 按 `started_at` 倒序（事件开始日期）。这反映了两种时间线的本质区别——一个关注"何时操作的"，另一个关注"何时发生的"。

8. **分页统一**：Feed 和 Timeline Event 都固定每页 15 条，都通过 `PaginatorHelper` 生成标准化的分页元数据。Vault 级 Feed 通过 `whereIn('contact_id', ...)` 聚合 Vault 下所有联系人的动态。

9. **`default_activity_tab`**：Vault 模型上的 `default_activity_tab` 字段让用户可以选择仪表盘默认展示 Feed 还是 Timeline Event。
