# Monica CRM 活动时间线（Activity Timeline）代码解析

Monica CRM 的联系人详情页和 Vault 仪表盘都需要展示"活动时间线"——将笔记、地址、标签、宠物、目标、心情追踪、联系人更新、重要日期、群组、文章、头像、收藏归档等多种类型的操作记录合并为一条按时间倒序排列的流，并分页展示。本文从领域模型、动态来源、关联边界、资源映射层、列表查询六个维度，逐步追踪其代码组织。

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

- `action` 字段用一个字符串常量表示"发生了什么"，在 `ContactFeedItem` 模型中定义了 **38 个** `ACTION_*` 常量（完整清单见 2.3 节）。

- `feedable` 是 Laravel 的多态关联（`MorphTo`），指向被操作的实体对象。这使得在渲染 Feed 条目时，可以直接拿到原始对象的完整数据。

```php
public function feedable(): MorphTo
{
    return $this->morphTo();
}
```

### 2.2 关联边界：12 个反向关联模型，3 个定义了未实际使用

每个可被 Feed 追踪的模型都声明了 `feedItem()` 方法（`MorphOne`），形成一对一的反向关联。

Grep 核准结果（共 12 个）：

| 模型 | `feedItem()` 位置 | 是否有对应 action 常量 | Service 是否实际绑定 |
|---|---|---|---|
| `Note` | [Note.php#L92-L95](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Note.php#L92-L95) | ✅ note_created/updated/destroyed | ✅ create/update 绑定 |
| `Address` | [Address.php#L61-L64](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Address.php#L61-L64) | ✅ address_created/updated/destroyed | ✅ create/remove 绑定 |
| `Goal` | [Goal.php#L60-L63](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Goal.php#L60-L63) | ✅ goal_created/updated/destroyed | ✅ create/update 绑定 |
| `ContactInformation` | [ContactInformation.php#L99-L102](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/ContactInformation.php#L99-L102) | ✅ contact_information_created/updated/destroyed | ✅ create/update 绑定 |
| `Label` | [Label.php#L56-L59](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Label.php#L56-L59) | ✅ label_assigned/removed | ✅ assigned/removed 绑定 |
| `MoodTrackingEvent` | [MoodTrackingEvent.php#L63-L66](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/MoodTrackingEvent.php#L63-L66) | ✅ mood_tracking_event_added/updated/deleted | ✅ create/update 绑定 |
| `Pet` | [Pet.php#L52-L59](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Pet.php#L52-L59) | ✅ pet_created/updated/destroyed | ✅ create/update 绑定 |
| `ContactImportantDate` | [ContactImportantDate.php#L95-L98](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/ContactImportantDate.php#L95-L98) | ✅ important_date_created/updated/destroyed | ✅ create/update 绑定 |
| `Group` | [Group.php#L98-L101](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Group.php#L98-L101) | ✅ added_to_group/removed_from_group | ✅ added/removed 绑定 |
| `Post` | [Post.php#L92-L95](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Post.php#L92-L95) | ✅ added_to_post/removed_from_post | ✅ added/removed 绑定 |
| `Loan` | [Loan.php#L108](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Loan.php#L108) | ✅ loan_created/updated | ❌ **定义了但 Service 未调用** |
| `Tag` | [Tag.php#L51-L54](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Models/Tag.php#L51-L54) | ❌ 无对应 action 常量 | ❌ **定义了但完全未使用** |

**关联边界的关键事实**：
- `Loan`：模型有 `feedItem()` 关联，`ContactFeedItem` 也定义了 `ACTION_LOAN_CREATED/UPDATED` 常量，但 [CreateLoan.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageLoans/Services/CreateLoan.php) 整个文件中没有 `createFeedItem()`，也没有 `ContactFeedItem::create()` 调用——属于典型的"骨架已搭、实现未完成"。
- `Tag`：属于 Journal/Post 模块的标签分类（`tag_post` pivot 表），模型上定义了 `feedItem()` 但 `ContactFeedItem` 中**没有任何 `ACTION_TAG_*` 常量**——完全是前置定义，尚未接入 Feed 体系。

### 2.3 Timeline Event 领域：TimelineEvent + LifeEvent

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

### 2.4 Feed 产生的完整入口：38 个常量 × 35 个 Service

Feed 条目不是在 Controller 中直接创建的，而是在各领域 Service 的 `execute()` 方法末尾调用 `createFeedItem()` 私有方法。这是一种 **"写时记录"（Write-Ahead Logging）** 模式——每次对联系人的写操作，都会自动追加一条 Feed 条目。

#### 2.4.1 典型流程（以创建笔记为例）

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

#### 2.4.2 完整的 38 个 action 常量清单与核准

`ContactFeedItem` 模型中定义了 38 个 `ACTION_*` 常量。下表逐一列举，分布在 35 个 Service 类中实际使用了其中 **35 个**，**3 个**仅定义未使用。

| 分类 | Action 常量与值 | 实际使用的 Service | bind feedable |
|---|---|---|---|
| **联系人基础操作（10/10 被使用）** | | | |
| 创建联系人 | `ACTION_CONTACT_CREATED` = `'contact_created'` | [CreateContact.php#L126](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/CreateContact.php#L126) | 否 |
| 更新联系人信息 | `ACTION_INFORMATION_UPDATED` = `'information_updated'` | [UpdateContact.php#L116](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/UpdateContact.php#L116) | 否 |
| 归档联系人 | `ACTION_ARCHIVED_CONTACT` = `'archived'` | [ToggleArchiveContact.php#L74](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/ToggleArchiveContact.php#L74) | 否 |
| 取消归档 | `ACTION_UNARCHIVED_CONTACT` = `'unarchived'` | [ToggleArchiveContact.php#L74](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/ToggleArchiveContact.php#L74) | 否 |
| 收藏联系人 | `ACTION_FAVORITED_CONTACT` = `'favorited'` | [ToggleFavoriteContact.php#L103](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/ToggleFavoriteContact.php#L103) | 否 |
| 取消收藏 | `ACTION_UNFAVORITED_CONTACT` = `'unfavorited'` | [ToggleFavoriteContact.php#L103](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContact/Services/ToggleFavoriteContact.php#L103) | 否 |
| 更换头像 | `ACTION_CHANGE_AVATAR` = `'changed_avatar'` | [UpdatePhotoAsAvatar.php#L96](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageAvatar/Services/UpdatePhotoAsAvatar.php#L96) + [DestroyAvatar.php#L81](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageAvatar/Services/DestroyAvatar.php#L81) | 否 |
| 更新工作信息 | `ACTION_JOB_INFORMATION_UPDATED` = `'job_information_updated'` | [UpdateJobInformation.php#L65](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageJobInformation/Services/UpdateJobInformation.php#L65) + [ResetJobInformation.php#L55](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageJobInformation/Services/ResetJobInformation.php#L55) | 否 |
| 更新宗教信仰 | `ACTION_RELIGION_UPDATED` = `'religion_updated'` | [UpdateReligion.php#L61](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageReligion/Services/UpdateReligion.php#L61) | 否 |
| **重要日期（3/3 被使用）** | | | |
| 创建重要日期 | `ACTION_IMPORTANT_DATE_CREATED` = `'important_date_created'` | [CreateContactImportantDate.php#L94](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactImportantDates/Services/CreateContactImportantDate.php#L94) | 是 → ContactImportantDate |
| 更新重要日期 | `ACTION_IMPORTANT_DATE_UPDATED` = `'important_date_updated'` | [UpdateContactImportantDate.php#L95](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactImportantDates/Services/UpdateContactImportantDate.php#L95) | 是 → ContactImportantDate |
| 删除重要日期 | `ACTION_IMPORTANT_DATE_DESTROYED` = `'important_date_destroyed'` | [DestroyContactImportantDate.php#L70](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactImportantDates/Services/DestroyContactImportantDate.php#L70) | 否（先删后记） |
| **笔记（3/3 被使用）** | | | |
| 创建笔记 | `ACTION_NOTE_CREATED` = `'note_created'` | [CreateNote.php#L79](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageNotes/Services/CreateNote.php#L79) | 是 → Note |
| 更新笔记 | `ACTION_NOTE_UPDATED` = `'note_updated'` | [UpdateNote.php#L79](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageNotes/Services/UpdateNote.php#L79) | 是 → Note |
| 删除笔记 | `ACTION_NOTE_DESTROYED` = `'note_destroyed'` | [DestroyNote.php#L65](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageNotes/Services/DestroyNote.php#L65) | 否（先记后删，不 bind） |
| **联系方式（3/3 被使用）** | | | |
| 创建联系方式 | `ACTION_CONTACT_INFORMATION_CREATED` = `'contact_information_created'` | [CreateContactInformation.php#L92](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactInformation/Services/CreateContactInformation.php#L92) | 是 → ContactInformation |
| 更新联系方式 | `ACTION_CONTACT_INFORMATION_UPDATED` = `'contact_information_updated'` | [UpdateContactInformation.php#L91](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactInformation/Services/UpdateContactInformation.php#L91) | 是 → ContactInformation |
| 删除联系方式 | `ACTION_CONTACT_INFORMATION_DESTROYED` = `'contact_information_destroyed'` | [DestroyContactInformation.php#L81](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactInformation/Services/DestroyContactInformation.php#L81) | 否（先删后记） |
| **地址（3/2 被使用）** | | | |
| 添加地址 | `ACTION_CONTACT_ADDRESS_CREATED` = `'address_created'` | [AssociateAddressToContact.php#L81](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactAddresses/Services/AssociateAddressToContact.php#L81) | 是 → Address |
| 更新地址 | `ACTION_CONTACT_ADDRESS_UPDATED` = `'address_updated'` | **❌ 未使用**（地址更新在 [UpdateAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Vault/ManageAddresses/Services/UpdateAddress.php)，为 Vault 级通用服务，不创建 Feed） | — |
| 删除地址 | `ACTION_CONTACT_ADDRESS_DESTROYED` = `'address_destroyed'` | [RemoveAddressFromContact.php#L82](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactAddresses/Services/RemoveAddressFromContact.php#L82) | 是 → Address（**条件删除，即使地址被删仍 bind**） |
| **标签（2/2 被使用）** | | | |
| 分配标签 | `ACTION_LABEL_ASSIGNED` = `'label_assigned'` | [AssignLabel.php#L67](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageLabels/Services/AssignLabel.php#L67) | 是 → Label |
| 移除标签 | `ACTION_LABEL_REMOVED` = `'label_removed'` | [RemoveLabel.php#L67](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageLabels/Services/RemoveLabel.php#L67) | 是 → Label（**仅 detach pivot，Label 不删**） |
| **宠物（3/3 被使用）** | | | |
| 创建宠物 | `ACTION_PET_CREATED` = `'pet_created'` | [CreatePet.php#L72](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManagePets/Services/CreatePet.php#L72) | 是 → Pet |
| 更新宠物 | `ACTION_PET_UPDATED` = `'pet_updated'` | [UpdatePet.php#L75](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManagePets/Services/UpdatePet.php#L75) | 是 → Pet |
| 删除宠物 | `ACTION_PET_DESTROYED` = `'pet_destroyed'` | [DestroyPet.php#L65](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManagePets/Services/DestroyPet.php#L65) | 否（先删后记） |
| **目标（3/3 被使用）** | | | |
| 创建目标 | `ACTION_GOAL_CREATED` = `'goal_created'` | [CreateGoal.php#L81](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageGoals/Services/CreateGoal.php#L81) | 是 → Goal |
| 更新目标 | `ACTION_GOAL_UPDATED` = `'goal_updated'` | [UpdateGoal.php#L72](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageGoals/Services/UpdateGoal.php#L72) | 是 → Goal |
| 删除目标 | `ACTION_GOAL_DESTROYED` = `'goal_destroyed'` | [DestroyGoal.php#L65](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageGoals/Services/DestroyGoal.php#L65) | 否（先删后记） |
| **群组（2/2 被使用）** | | | |
| 加入群组 | `ACTION_ADDED_TO_GROUP` = `'added_to_group'` | [AddContactToGroup.php#L89](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageGroups/Services/AddContactToGroup.php#L89) | 是 → Group |
| 离开群组 | `ACTION_REMOVED_FROM_GROUP` = `'removed_from_group'` | [RemoveContactFromGroup.php#L67](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageGroups/Services/RemoveContactFromGroup.php#L67) | 是 → Group（**仅 detach pivot**） |
| **文章（2/2 被使用）** | | | |
| 加入文章 | `ACTION_ADDED_TO_POST` = `'added_to_post'` | [AddContactToPost.php#L83](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Vault/ManageJournals/Services/AddContactToPost.php#L83) | 是 → Post |
| 离开文章 | `ACTION_REMOVED_FROM_POST` = `'removed_from_post'` | [RemoveContactFromPost.php#L85](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Vault/ManageJournals/Services/RemoveContactFromPost.php#L85) | 是 → Post（**仅 detach pivot**） |
| **心情追踪（3/3 被使用）** | | | |
| 添加心情 | `ACTION_MOOD_TRACKING_EVENT_CREATED` = `'mood_tracking_event_added'` | [CreateMoodTrackingEvent.php#L76](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageMoodTrackingEvents/Services/CreateMoodTrackingEvent.php#L76) | 是 → MoodTrackingEvent |
| 更新心情 | `ACTION_MOOD_TRACKING_EVENT_UPDATED` = `'mood_tracking_event_updated'` | [UpdateMoodTrackingEvent.php#L79](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageMoodTrackingEvents/Services/UpdateMoodTrackingEvent.php#L79) | 是 → MoodTrackingEvent |
| 删除心情 | `ACTION_MOOD_TRACKING_EVENT_DESTROYED` = `'mood_tracking_event_deleted'` | [DestroyMoodTrackingEvent.php#L65](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageMoodTrackingEvents/Services/DestroyMoodTrackingEvent.php#L65) | 否（先删后记） |
| **借贷（2/0 被使用）** | | | |
| 创建借贷 | `ACTION_LOAN_CREATED` = `'loan_created'` | **❌ 未使用**（Loan Service 未创建 FeedItem） | — |
| 更新借贷 | `ACTION_LOAN_UPDATED` = `'loan_updated'` | **❌ 未使用**（无对应 Service） | — |

**核准统计**：

- **38 个** action 常量定义在 `ContactFeedItem` 模型中
- **35 个** Service 文件实际调用 `ContactFeedItem::create()`
- **35 个** action 常量被实际使用
- **3 个** 常量定义了但未被使用：
  - `ACTION_LOAN_CREATED` — CreateLoan Service 中没有 `createFeedItem()`
  - `ACTION_LOAN_UPDATED` — 没有对应的 UpdateLoan Service 创建 feed
  - `ACTION_CONTACT_ADDRESS_UPDATED` — 地址在 Vault 级 UpdateAddress Service 不创建 feed

#### 2.4.3 统一分类口径：8 大类 × feedable 保留状态（全文基准）

35 个实际使用的 action 按**技术性质**（而非业务模块）归为 8 类，所有后续章节（2.5 节删除语义、6.2 节总览图、第七章总结）均以本分类为准：

| 类 # | 技术分类 | 子类说明 | 数量 | feedable 保留状态 | action 清单 |
|---|---|---|---|---|---|
| ① | **创建实体** | 新建可被多态追踪的实体对象 | 7 | ✅ **有效绑定**（实体存在，关联有效） | `important_date_created`, `note_created`, `contact_information_created`, `address_created`, `pet_created`, `goal_created`, `mood_tracking_event_added` |
| ② | **更新实体** | 修改已存在的可追踪实体字段 | 6 | ✅ **有效绑定**（实体存在，关联有效） | `important_date_updated`, `note_updated`, `contact_information_updated`, `pet_updated`, `goal_updated`, `mood_tracking_event_updated` |
| ③ | **绑定关联（attach）** | 把联系人关联到已有实体，新增 pivot 行，实体本身不新增 | 3 | ✅ **实体保留**（pivot 新增，实体原已存在） | `label_assigned`, `added_to_group`, `added_to_post` |
| ④ | **解绑关联（仅 detach）** | 只拆除 pivot 关联行，**实体本身不删除** | 3 | ✅ **实体保留**（pivot 删除，实体仍存在） | `label_removed`, `removed_from_group`, `removed_from_post` |
| ⑤ | **解绑 + 条件删除** | 先 detach pivot；若无其他引用则真正删除实体；但无论删不删都 bind | 1 | ⚠️ **有值但实体可能不存在**（存在 dangling 风险） | `address_destroyed` |
| ⑥ | **纯删除实体** | 直接删除实体对象，无论先删后记还是先记后删，均不 bind | 6 | ❌ **为 NULL**（完全丢失多态关联） | `important_date_destroyed`, `note_destroyed`, `contact_information_destroyed`, `pet_destroyed`, `goal_destroyed`, `mood_tracking_event_deleted` |
| ⑦ | **联系人级操作（无实体）** | 直接操作联系人本身或元信息，无可关联的 feedable 实体 | 5 | ❌ **本就无实体可关联**（语义上不需要） | `contact_created`, `information_updated`, `changed_avatar`, `job_information_updated`, `religion_updated` |
| ⑧ | **Toggle 双向标记** | 切换 bool 标记位（收藏/归档），无可关联实体 | 4 | ❌ **本就无实体可关联**（语义上不需要） | `archived`, `unarchived`, `favorited`, `unfavorited` |

**合计**：7 + 6 + 3 + 3 + 1 + 6 + 5 + 4 = **35** ✓

**bind 状态图例（全文表格统一使用）**：
- ✅ **有效绑定 / 实体保留**：`feedable_id/type` 有值，且 `$item->feedable` 能正常返回对象
- ⚠️ **有值但实体可能不存在（dangling）**：`feedable_id/type` 有值，但实体可能已被删除，查询时可能返回 `null`
- ❌ **为 NULL**：`feedable_id/type` 字段为 `NULL`，无任何多态关联
- ❌ **本就无实体可关联**：语义上不需要关联实体的联系人级/Toggle 操作

### 2.5 关键：删除/解绑/移除时 feedable 关联是否保留

**与 2.4.3 节 8 大类基准的对应**：本节覆盖的是 8 大类中涉及"删除/解除关联"的三类——**④ 解绑关联（仅 detach，3 个）、⑤ 解绑 + 条件删除（1 个）、⑥ 纯删除实体（6 个）**，合计 10 个 action。类①②③⑦⑧不涉及删除语义，不在此展开。

这 10 个操作对多态关联的处理方式并不一致。以下是逐一核准后的完整对照表（bind 状态图例同 2.4.3 节）：

| action | Service | 操作性质 | 执行顺序 | `$entity->feedItem()->save()` | feedable 数据库状态 |
|---|---|---|---|---|---|
| **note_destroyed** | [DestroyNote.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageNotes/Services/DestroyNote.php#L46-L58) | 删除实体 | **① createFeedItem ② delete()** | ❌ **不调用**（特例：先记后删仍不 bind） | ❌ feedable_id/type 为 NULL |
| **pet_destroyed** | [DestroyPet.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManagePets/Services/DestroyPet.php#L45-L68) | 删除实体 | ① delete() ② createFeedItem | ❌ 不调用 | ❌ feedable_id/type 为 NULL |
| **goal_destroyed** | [DestroyGoal.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageGoals/Services/DestroyGoal.php#L45-L68) | 删除实体 | ① delete() ② createFeedItem | ❌ 不调用 | ❌ feedable_id/type 为 NULL |
| **contact_information_destroyed** | [DestroyContactInformation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactInformation/Services/DestroyContactInformation.php#L47-L84) | 删除实体 | ① delete() ② createFeedItem | ❌ 不调用 | ❌ feedable_id/type 为 NULL |
| **important_date_destroyed** | [DestroyContactImportantDate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactImportantDates/Services/DestroyContactImportantDate.php#L50-L73) | 删除实体 | ① delete() ② createFeedItem | ❌ 不调用 | ❌ feedable_id/type 为 NULL |
| **mood_tracking_event_deleted** | [DestroyMoodTrackingEvent.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageMoodTrackingEvents/Services/DestroyMoodTrackingEvent.php#L45-L68) | 删除实体 | ① delete() ② createFeedItem | ❌ 不调用 | ❌ feedable_id/type 为 NULL |
| **address_destroyed** | [RemoveAddressFromContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactAddresses/Services/RemoveAddressFromContact.php#L48-L87) | **条件删除** | ① detach() ② 若无其他关联则 delete() ③ createFeedItem + **save** | ✅ **调用** | ⚠️ **feedable_id/type 有值，但 address 实体可能已不存在** |
| **label_removed** | [RemoveLabel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageLabels/Services/RemoveLabel.php#L45-L71) | **解绑 pivot** | ① detach() ② createFeedItem + **save** | ✅ 调用 | ✅ **feedable 正常可用**（Label 实体不删） |
| **removed_from_group** | [RemoveContactFromGroup.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageGroups/Services/RemoveContactFromGroup.php#L43-L71) | **解绑 pivot** | ① detach() ② createFeedItem + **save** | ✅ 调用 | ✅ **feedable 正常可用**（Group 实体不删） |
| **removed_from_post** | [RemoveContactFromPost.php](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Vault/ManageJournals/Services/RemoveContactFromPost.php#L48-L89) | **解绑 pivot** | ① detach() ② createFeedItem + **save** | ✅ 调用 | ✅ **feedable 正常可用**（Post 实体不删） |

**可以归纳为四种删除/解绑语义（对应 2.4.3 节 8 大类编号）**：

1. **⑥ 纯删除实体（6 种）**：对应类⑥的全部 6 个 action。先删实体再记 Feed（DestroyNote 虽为先记后删特例，但仍不 bind），完全不绑定 feedable → ❌ `feedable_id/type` 均为 NULL，实体详情丢失，仅靠 `description` 字段保存文本摘要
2. **⑤ 解绑 pivot + 条件删除（1 种 RemoveAddress）**：对应类⑤的 `address_destroyed`。地址是多对多多态（可被多个联系人共享），解绑后若没有其他联系人使用该地址则真正删除；但无论是否删除实体，都会先 bind feedable → ⚠️ 导致数据库中 `feedable_id/type` 有值，但查询时 `$item->feedable` 可能返回 null（实体被删的情况，dangling 关联）
3. **④ 仅解绑 pivot（3 种 Label/Group/Post）**：对应类④的全部 3 个 action。实体本身保留，仅解除中间表关联；bind feedable → ✅ `feedable_id/type` 正常可用，前端可以看到群组名称、标签名称、文章标题等完整数据
4. **①②③⑦⑧ 无删除语义**：创建/更新/绑定关联/联系人级/Toggle 共 25 个 action 不涉及删除，feedable 状态见 2.4.3 节总表

### 2.6 特殊的双向动态操作（Toggle 模式）

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

`getSentence()` 使用 PHP 8 的 `match` 表达式，将 **35 个命名分支**（34 个 model 常量 + 1 个额外的 `author_deleted`） + `default` 兜底 = 共 36 个分支映射为翻译字符串（[ModuleFeedViewHelper.php#L37-L77](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L37-L77)）：

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

**核准 getSentence 覆盖情况**：

| 覆盖状态 | 数量 | 具体 action |
|---|---|---|
| ✅ 已覆盖 | 34 个 model 常量 | 除 loan_created/updated、mood_tracking_event_updated/deleted 外的全部 |
| ➕ 额外分支（非 model 常量） | 1 个 | `author_deleted`（遗留分支，无 Service 实际产生） |
| ❌ 未覆盖（走 default → "unknown action"） | 4 个 model 常量 | `loan_created`、`loan_updated`、`mood_tracking_event_updated`、`mood_tracking_event_deleted` |

**设计要点**：
1. **硬编码字符串匹配**：使用硬编码字符串而非 `ContactFeedItem::ACTION_*` 常量，新增 action 时需同步修改两处
2. **前后不一致**：`mood_tracking_event_updated/deleted` 在 `getData()` 中有专属 Helper 映射，但在 `getSentence()` 中缺失——用户会看到句子为"unknown action"但 data 字段有完整对象
3. **`address_updated` 已覆盖但无实际来源**：`getSentence()` 和 `getData()` 都已准备好处理 `address_updated`，但没有 Service 产生这个 action——属于典型的"预留接口"

#### 3.1.2 操作者信息：getAuthor()

`getAuthor()` 处理操作者可能已被删除的情况（[ModuleFeedViewHelper.php#L79-L105](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L79-L105)）：

- 如果 `$item->author` 存在 → 通过 `UserHelper::getInformationAboutContact()` 返回正常用户信息
- 如果 `$item->author` 为 `null` → 返回一个内置的 Monica SVG 头像和 "Deleted author" 文本

#### 3.1.3 数据映射分发：getData()

`getData()` 使用 `switch` 语句，按 action 类型将数据映射分发到 **7 个 Action Feed Helper + 1 个通用 Helper**（[ModuleFeedViewHelper.php#L107-L147](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L107-L147)）。

#### 3.1.4 完整的 Action → Helper 映射表（核准版）

| Helper 类 | 覆盖的 action | 分支数 | **实际会产生 Feed 的数量** | 产出数据结构 |
|---|---|---|---|---|
| [ActionFeedLabelAssigned](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedLabelAssigned.php) | `label_assigned`, `label_removed` | 2 | **2** | `{ label: { object, description }, contact }` |
| [ActionFeedAddress](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedAddress.php) | `address_created`, `address_updated`, `address_destroyed` | 3 | **2**（address_updated 无来源） | `{ address: { object, description }, contact }` |
| [ActionFeedContactInformation](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedContactInformation.php) | `contact_information_created/updated/destroyed` | 3 | **3** | `{ information: { object, description }, contact }` |
| [ActionFeedPet](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedPet.php) | `pet_created`, `pet_updated`, `pet_destroyed` | 3 | **3** | `{ pet: { object, description }, contact }` |
| [ActionFeedNote](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedNote.php) | `note_created`, `note_updated`, `note_destroyed` | 3 | **3** | `{ note: { object, description }, contact }` |
| [ActionFeedGoal](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedGoal.php) | `goal_created`, `goal_updated`, `goal_destroyed` | 3 | **3** | `{ goal: { object, description }, contact }` |
| [ActionFeedMoodTrackingEvent](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedMoodTrackingEvent.php) | `mood_tracking_event_added/updated/deleted` | 3 | **3** | `{ mood_tracking_event: { object, description }, contact }` |
| **专属 Helper 小计** | - | **20** | **19**（address_updated 无来源） | - |
| [ActionFeedGenericContactInformation](file:///d:/fz/0601-2/solo-dogfeeding/code/15-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedGenericContactInformation.php) | default 兜底（其余所有） | 18（逻辑上） | **16**（loan_created/updated 未使用） | `{ contact }`（仅联系人基础信息） |
| **总计** | - | **38** | **35**（3 个未使用） | - |

#### 3.1.5 通用动作映射：16 个实际会产生 Feed 的 default 动作

进入 `default` 分支的 18 个 action 中，**16 个实际会产生 FeedItem**，**2 个仅定义常量未使用**：

**实际会产生 Feed 的 16 个通用动作**（都走 ActionFeedGenericContactInformation）：

| 编号 | action | 性质 | description 字段是否有内容 |
|---|---|---|---|
| 1 | `contact_created` | 创建联系人 | 否（无 description） |
| 2 | `information_updated` | 更新联系人基础信息 | 否 |
| 3 | `important_date_created` | 创建重要日期 | ✅ **有**（"标签 + 格式化日期"） |
| 4 | `important_date_updated` | 更新重要日期 | ✅ **有**（"标签 + 格式化日期"） |
| 5 | `important_date_destroyed` | 删除重要日期 | ✅ **有**（"标签 + 格式化日期"） |
| 6 | `added_to_group` | 加入群组 | ✅ **有**（群组名称） |
| 7 | `removed_from_group` | 离开群组 | ✅ **有**（群组名称） |
| 8 | `added_to_post` | 加入文章 | ✅ **有**（文章标题） |
| 9 | `removed_from_post` | 离开文章 | ✅ **有**（文章标题） |
| 10 | `archived` | 归档联系人 | 否 |
| 11 | `unarchived` | 取消归档 | 否 |
| 12 | `favorited` | 收藏联系人 | 否 |
| 13 | `unfavorited` | 取消收藏 | 否 |
| 14 | `changed_avatar` | 更换头像 | 否 |
| 15 | `job_information_updated` | 更新工作信息 | 否 |
| 16 | `religion_updated` | 更新宗教信仰 | 否 |

**仅定义常量未使用的 2 个**：
- `loan_created`（CreateLoan Service 未实现）
- `loan_updated`（无对应 Service）

**ActionFeedGenericContactInformation 本身的实现**：

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

**映射层的覆盖缺口**：

上述编号 3-9 共 **7 个通用动作**的 Service 实际上在 `description` 字段中写入了有意义的信息（重要日期、群组名、文章标题），但 `ActionFeedGenericContactInformation` 没有读取和返回 `description` 字段，导致前端无法展示这些摘要。这是一个明确可以改进的设计点。

#### 3.1.6 每个 Action Helper 的统一输出结构

对于有专属 Helper 的 action，输出结构统一为：

```json
{
  "<entity_type>": {
    "object": { /* feedable 实体详情, 可能为 null */ },
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

`feedable` 实体为 `null` 的场景：
- 各种 destroy 操作（feedable_id/type 本身为 NULL）
- `address_destroyed` 中地址被条件删除（feedable_id/type 有值，但查询时关联返回 null）
- Note 的 `note_destroyed`：虽然先记后删，但由于没有调用 feedItem()->save()，feedable_id/type 仍为 NULL

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
┌─────────────────────── 写操作（产生 Feed，35 个 Service） ────────────────────────┐
│                                                                                    │
│  用户操作 → Controller → Service::execute()                                        │
│                          ├── 业务逻辑（创建/更新/删除实体 / detach pivot）          │
│                          ├── contact->last_updated_at = now()                      │
│                          └── createFeedItem()                                      │
│                                ├── ContactFeedItem::create([...])                  │
│                                │   ├── create/update + pivot类: feedItem()->save() │
│                                │   ├── 纯delete类:     仅create不save               │
│                                │   ├── address_destroyed: detach(+maybe delete)+save│
│                                │   └── note_destroyed:  特例:先记后删仍不save        │
│                                └── 更新 last_updated_at                              │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────── 读操作（展示时间线） ───────────────────────┐
│                                                                    │
│  前端请求 → Controller                                              │
│              ├── ContactFeedController (单联系人 Feed)              │
│              │     ContactFeedItem::where('contact_id', ...)       │
│              │       ->with(['author', 'contact.importantDates'])  │
│              │       ->orderBy('created_at', 'desc')->paginate(15) │
│              │                                                      │
│              ├── VaultFeedController (Vault 级 Feed)                │
│              │     ContactFeedItem::whereIn('contact_id', [...])   │
│              │       ->with(['author', 'contact.importantDates'])  │
│              │       ->orderBy('created_at', 'desc')->paginate(15) │
│              │                                                      │
│              └── ContactModuleTimelineEventController (生活事件)     │
│                    $contact->timelineEvents()                      │
│                      ->orderBy('started_at', 'desc')->paginate(15) │
│                                                                    │
│            → ModuleFeedViewHelper / ModuleLifeEventViewHelper      │
│                ├── getSentence() → match(35命名 + default)         │
│                ├── getAuthor()  → UserHelper / 默认 Monica SVG     │
│                └── getData()    → switch(7专属 + default兜底)       │
│                                                                    │
│            → PaginatorHelper::getData($paginator)                  │
│                                                                    │
│            → JSON Response { data: {...}, paginator: {...} }       │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 6.2 8 大类技术分类 × 35 个 Action × 多态关联保留状态（以 2.4.3 节为唯一基准）

```
Action 常量 (38 个, 定义在 ContactFeedItem)
    │
    ├─ 实际被 Service 使用: 35 个（按 2.4.3 节 8 大类技术分类）
    │     │
    │     ├─ ① 创建实体 (7 个)  →  均调用 feedItem()->save()
    │     │     · important_date_created, note_created, contact_information_created
    │     │     · address_created, pet_created, goal_created, mood_tracking_event_added
    │     │     └─► feedable_id/type  ✅ 有效绑定（实体存在）
    │     │
    │     ├─ ② 更新实体 (6 个)  →  均调用 feedItem()->save()
    │     │     · important_date_updated, note_updated, contact_information_updated
    │     │     · pet_updated, goal_updated, mood_tracking_event_updated
    │     │     └─► feedable_id/type  ✅ 有效绑定（实体存在）
    │     │
    │     ├─ ③ 绑定关联（attach，新增 pivot，3 个）→  均调用 feedItem()->save()
    │     │     · label_assigned, added_to_group, added_to_post
    │     │     └─► feedable_id/type  ✅ 实体保留（pivot 新增，Label/Group/Post 原已存在）
    │     │
    │     ├─ ④ 解绑关联（仅 detach pivot，不删实体，3 个）→  均调用 feedItem()->save()
    │     │     · label_removed, removed_from_group, removed_from_post
    │     │     └─► feedable_id/type  ✅ 实体保留（pivot 删除，Label/Group/Post 仍存在）
    │     │
    │     ├─ ⑤ 解绑 + 条件删除（1 个 address_destroyed）→  调用 feedItem()->save()
    │     │     · detach 关联 pivot → 无其他引用则 delete 实体 → 最后 save feedable
    │     │     └─► feedable_id/type  ⚠️ 有值但实体可能不存在（dangling 关联）
    │     │
    │     ├─ ⑥ 纯删除实体（6 个，无论先删后记还是先记后删均不 bind）
    │     │     · important_date_destroyed, note_destroyed（先记后删特例，仍不 bind）
    │     │     · contact_information_destroyed, pet_destroyed
    │     │     · goal_destroyed, mood_tracking_event_deleted
    │     │     └─► feedable_id/type  ❌ 为 NULL（完全丢失多态关联）
    │     │
    │     ├─ ⑦ 联系人级操作（无实体可关联，5 个）→  无 feedable 绑定
    │     │     · contact_created, information_updated, changed_avatar
    │     │     · job_information_updated, religion_updated
    │     │     └─► feedable_id/type  ❌ 语义上不需要关联实体
    │     │
    │     └─ ⑧ Toggle 双向标记（无实体可关联，4 个）→  无 feedable 绑定
    │           · archived, unarchived, favorited, unfavorited
    │           └─► feedable_id/type  ❌ 语义上不需要关联实体
    │
    └─ 未被使用（仅定义常量）: 3 个
          · loan_created（CreateLoan Service 未实现 createFeedItem）
          · loan_updated（无对应 UpdateLoan Service 创建 feed）
          · address_updated（地址更新在 Vault 级 UpdateAddress，不创建 Feed）

12 个 feedItem() 反向关联模型
    │
    ├─ 10 个实际使用，与类①~⑤的实体一一对应：
    │     Note, Address, Goal, ContactInformation, Label,
    │     MoodTrackingEvent, Pet, ContactImportantDate, Group, Post
    │
    └─ 2 个定义了但未实际接入：
          · Loan（有 ACTION_LOAN_CREATED/UPDATED 常量，但 Service 未实现）
          · Tag（模型上有 feedItem()，但 ContactFeedItem 中无任何 ACTION_TAG_* 常量）

ModuleFeedViewHelper::getData()
      │
      ├─ 7 个专属 Helper (声明 20 个 action, 实际产生 19 个)
      │     ├─ ActionFeedNote/Pet/Goal/ContactInformation (各 3, 全覆盖)
      │     ├─ ActionFeedLabelAssigned (2, 全覆盖)
      │     ├─ ActionFeedMoodTrackingEvent (3, 全覆盖)
      │     └─ ActionFeedAddress (3 个声明 → 2 个实际产生, address_updated 无来源)
      │
      └─ 通用 Helper (default 兜底: 18 个 → 16 个实际产生)
            ├─ 7 个: Service 写入了 description 字段但 Helper 不返回 (重要日期/群组/文章)
            └─ 9 个: 纯联系人级操作, 无 description

ModuleFeedViewHelper::getSentence()
      │
      ├─ 35 个命名分支 (34 个 model 常量 + 1 个 author_deleted 遗留)
      │
      └─ 4 个 model 常量未覆盖 (loan_created/updated, mood_updated/deleted)
```

---

## 七、设计要点总结（全文统一口径：以 2.4.3 节 8 大类技术分类为基准）

1. **Feed 是"操作日志"，Timeline Event 是"生活记事"**——前者由系统自动产生，后者由用户手动创建。两者在模型、查询、映射层都完全独立，仅在 Vault 仪表盘前端通过 Tab 切换统一呈现。

2. **多态关联实现统一 Feed 表**：`ContactFeedItem.feedable` 使用 Laravel 的 `nullableNumericMorphs`，将 8 大类中类①~⑤涉及的 10 种实体（Note、Address、Goal、ContactInformation、Label、MoodTrackingEvent、Pet、ContactImportantDate、Group、Post）统一收纳进同一张表。这是"不同类型活动条目合并成时间线"的核心数据库机制。

3. **关联边界的缺口**：12 个模型定义了 `feedItem()` 反向关联，但 `Loan`（有常量无实现）和 `Tag`（无常量无实现）实际未接入；38 个 action 常量中有 3 个未使用。

4. **38 个常量 × 35 个 Service × 8 大类技术分类**：模型定义了 38 个 action 常量，35 个 Service 文件实际调用 `ContactFeedItem::create()`。按技术性质归为 ①~⑧ 共 8 类（见 2.4.3 节总表）。Feed 条目的创建散布在各领域 Service 中（而非事件监听器）。

5. **8 大类 × 多态关联保留精确对应**：
   - ① 创建实体（7 个）、② 更新实体（6 个）→ ✅ feedable_id/type 有效绑定
   - ③ 绑定关联（3 个）、④ 解绑关联仅 detach（3 个）→ ✅ 实体保留，关联有效
   - ⑤ 解绑 + 条件删除（1 个 address_destroyed）→ ⚠️ 有值但实体可能不存在（dangling）
   - ⑥ 纯删除实体（6 个，含 DestroyNote 先记后删特例）→ ❌ feedable_id/type 为 NULL
   - ⑦ 联系人级操作（5 个）、⑧ Toggle 双向标记（4 个）→ ❌ 语义上本就无需关联实体
   - **总计 feedable 关联情况**：13 个有效绑定 + 6 个实体保留 = 19 个 ✅；1 个 dangling ⚠️；6 个 NULL + 9 个无需关联 = 15 个 ❌

6. **映射层覆盖缺口**：
   - `getData()` 中 **16 个实际产生 Feed 的 action** 走 default 分支，其中 7 个（重要日期 3 个、群组/文章 added/removed 各 2 个）Service 实际写入了 `description` 字段但 `ActionFeedGenericContactInformation` 未读取返回
   - `getSentence()` 有 4 个 model 常量未覆盖（loan_created/updated + mood_updated/deleted，后者会在前端显示 "unknown action" 但 data 字段完整）
   - `address_updated` 是死代码（ViewHelper 全覆盖但无 Service 产生）

7. **前后不一致的隐患**：
   - `getSentence()` 中 mood_tracking_event_updated/deleted 缺失但 `getData()` 有专属 Helper → 前端显示"unknown action" + 完整数据
   - 硬编码字符串匹配而非 model 常量 → 新增 action 时容易遗漏

8. **排序策略差异**：Feed 按 `created_at` 倒序（操作发生时间），Timeline Event 按 `started_at` 倒序（事件开始日期）。

9. **分页统一**：Feed 和 Timeline Event 都固定每页 15 条，都通过 `PaginatorHelper` 生成标准化的分页元数据。

10. **`default_activity_tab`**：Vault 模型上的 `default_activity_tab` 字段让用户可以选择仪表盘默认展示 Feed 还是 Timeline Event。
