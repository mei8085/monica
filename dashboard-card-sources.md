# Dashboard Summary Card 数据来源分析

本文档梳理 Monica 应用中 Vault Dashboard 页面每个 Card/板块的统计数据来源，从前端组件一路追踪到后端 Eloquent 查询与数据库表。

---

## 整体架构概览

Dashboard 页面由 `VaultController::show()` 方法渲染，路由为 `vault.show`。控制器通过多个 ViewHelper 方法收集各卡片数据，以 Inertia Props 传递给前端 Vue 组件。

**入口控制器**: [VaultController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L66-L89)

**前端页面**: [Dashboard/Index.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Pages/Vault/Dashboard/Index.vue)

Dashboard 采用三栏布局（左 / 中 / 右），共 6 个卡片组件 + 1 个 Tab 切换区域：

| 位置 | 组件 | Prop 名 | ViewHelper 方法 |
|------|------|---------|----------------|
| 左栏 | Favorites | `favorites` | `VaultShowViewHelper::favorites()` |
| 左栏 | LastUpdated | `lastUpdatedContacts` | `VaultShowViewHelper::lastUpdatedContacts()` |
| 中栏 | Feed / LifeEvent / LifeMetrics（Tab切换） | `url.feed` / `lifeEvents` / `lifeMetrics` | 异步加载 / `ModuleLifeEventViewHelper::data()` / `VaultLifeMetricsViewHelper::data()` |
| 右栏 | MoodTrackingEvents | `moodTrackingEvents` | `VaultShowViewHelper::moodTrackingEvents()` |
| 右栏 | UpcomingReminders | `upcomingReminders` | `VaultShowViewHelper::upcomingReminders()` |
| 右栏 | DueTasks | `dueTasks` | `VaultShowViewHelper::dueTasks()` |

---

## 1. Favorites（收藏联系人）

**前端组件**: [Favorites.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Pages/Vault/Dashboard/Partials/Favorites.vue)

**ViewHelper**: [VaultShowViewHelper::favorites()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L99-L116)

### 数据来源

```
User → contacts() (BelongsToMany, 中间表: contact_vault_user)
      → wherePivot('vault_id', $vault->id)
      → wherePivot('is_favorite', true)
```

- **查询**: 当前用户在指定 Vault 中标记为收藏的所有联系人
- **数据库表**: `contact_vault_user`（中间表，含 `is_favorite` 字段）
- **关联模型**: `User` hasMany `Contact`（通过 `contact_vault_user`）
- **返回数据**: 每个收藏联系人的 `id`, `name`, `avatar`, `url.show`

### 展示逻辑

- 仅在 `favorites.length > 0` 时显示
- 列表展示联系人头像 + 名称（可点击跳转联系人详情）

---

## 2. Last Updated（最近更新的联系人）

**前端组件**: [LastUpdated.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Pages/Vault/Dashboard/Partials/LastUpdated.vue)

**ViewHelper**: [VaultShowViewHelper::lastUpdatedContacts()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L17-L34)

### 数据来源

```
Vault → contacts() (HasMany)
      → orderBy('last_updated_at', 'desc')
      -> take(5)
```

- **查询**: 该 Vault 下按 `last_updated_at` 降序排列的前 5 个联系人
- **数据库表**: `contacts`（字段 `last_updated_at` 为 datetime 类型）
- **返回数据**: `id`, `name`, `avatar`, `url.show`

### 展示逻辑

- 固定显示最近更新的 5 个联系人
- 始终显示（即使没有数据也会渲染空列表）

---

## 3. Upcoming Reminders（未来 30 天提醒）

**前端组件**: [UpcomingReminders.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Pages/Vault/Dashboard/Partials/UpcomingReminders.vue)

**ViewHelper**: [VaultShowViewHelper::upcomingReminders()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L36-L97)

### 数据来源（多层关联查询）

```
Vault → users() (BelongsToMany)
      → flatMap → notificationChannels() (HasMany, 表: user_notification_channels)
      → flatMap → contactReminders() (BelongsToMany, 中间表: contact_reminder_scheduled)
              → wherePivot('scheduled_at', '<=', now + 30 days)   ← 只设上界
              → wherePivot('triggered_at', null)
              → orderByPivot('scheduled_at', 'asc')
```

> **关键说明**：`scheduled_at` **只设上界**（`<= 当前时间 + 30天`），没有下界，因此**包含所有已过期但尚未触发**的提醒。
> 只要 `triggered_at IS NULL`（未触发过）且 `scheduled_at <= now+30d`，就会出现在列表中，哪怕 scheduled_at 是过去的日期。

然后对每个 reminder 检查其 `contact.vault_id` 是否匹配当前 Vault，过滤掉不属于当前 Vault 的提醒，最后按 `contact_reminder_id` 去重（因为同一提醒可能关联多个 notification channel）。

- **涉及数据库表**:
  - `vaults` → `vault_user`（中间表）→ `users`
  - `users` → `user_notification_channels`
  - `user_notification_channels` → `contact_reminder_scheduled`（中间表）→ `contact_reminders`
  - `contact_reminders` → `contacts`
- **关键过滤条件**:
  - `scheduled_at <= 当前时间 + 30天`（仅上界，含过期）
  - `triggered_at IS NULL`（尚未触发的提醒）
  - `contact.vault_id == 当前 Vault ID`
- **去重逻辑**: 按 `contact_reminder.id` 去重，避免同一提醒因多个通知渠道重复显示
- **返回数据**: `id`, `label`, `scheduled_at`（格式化后的日期）, `contact.{id, name, avatar, url.show}`

### 展示逻辑

- 标题："Reminders for the next 30 days"
- 有提醒时：列表展示日期 + 联系人头像名称 + 提醒标签
- 无提醒时：显示空白占位图 `dashboard_blank_reminders.svg`
- 底部"View all"链接跳转提醒列表页

---

## 4. Due Tasks（到期和即将到期的任务）

**前端组件**: [DueTasks.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Pages/Vault/Dashboard/Partials/DueTasks.vue)

**ViewHelper**: [VaultShowViewHelper::dueTasks()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L118-L166)

### 数据来源

```
Vault → contacts() (HasMany, with('tasks'))
      → flatMap → tasks (HasMany, 表: contact_tasks)
      → where('completed', false)
      → where('due_at', '<=', now + 30 days)   ← 只设上界
      → sortBy('due_at')
```

> **关键说明**：`due_at` **只设上界**（`<= 当前时间 + 30天`），没有下界，因此**包含所有已过期但未完成**的任务。
> 凡是 `completed = false` 且 `due_at <= now+30d` 的任务都会显示，哪怕 due_at 是过去很久的日期。

- **查询**: 该 Vault 下所有联系人的未完成任务，且截止日期在 30 天以内（含已过期）
- **数据库表**: `contact_tasks`（字段 `completed`, `due_at`）
- **关联**: `Contact` hasMany `ContactTask`
- **关键过滤条件**:
  - `completed = false`
  - `due_at <= 当前时间 + 30天`（仅上界，含过期）
- **返回数据**: `id`, `label`, `description`, `completed`, `completed_at`, `due_at.{formatted, value, is_late}`, `url.toggle`, `contact.{id, name, avatar, url.show}`
- **统计值 `is_late`**: `due_at->isPast()` 判断任务是否已逾期（逾期显示红色标签）

### 展示逻辑

- 标题："Due and upcoming tasks"
- 每个任务显示：复选框 + 标签 + 截止日期标签（逾期为红色，未逾期为天蓝色）+ 联系人
- 可直接勾选完成任务（调用 `toggle` API）
- 无任务时显示空白占位图 `dashboard_blank_tasks.svg`
- 底部"View all"链接

---

## 5. Mood Tracking Events（记录心情）

**前端组件**: [MoodTrackingEvents.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Pages/Vault/Dashboard/Partials/MoodTrackingEvents.vue)

**ViewHelper**: [VaultShowViewHelper::moodTrackingEvents()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L168-L193)

### 数据来源

```
Vault → moodTrackingParameters() (HasMany, 表: mood_tracking_parameters)
      → orderBy('position', 'asc')
```

- **查询**: 该 Vault 下所有心情追踪参数（如"开心"、"难过"等选项）
- **数据库表**: `mood_tracking_parameters`（字段 `vault_id`, `label`, `label_translation_key`, `hex_color`, `position`）
- **返回数据**: `mood_tracking_parameters: [{id, label, hex_color}]`, `current_date`, `url.{history, store}`
- **注意**: 此卡片不返回已有心情事件列表，只提供参数供用户录入新心情
- **提交目标**: `contact.mood_tracking_event.store`（作用于当前用户在 Vault 中的 contact）

### 展示逻辑

- 标题："Record your mood"（带 CloudSun 图标）
- 默认展示"How are you?"提示 + "Record your mood"按钮
- 点击后展开表单：选择心情参数（单选）+ 可选日期/备注/睡眠时长
- 提交后调用 `contact.mood_tracking_event.store` API
- 成功后显示🎉 + "Your mood has been recorded!"

---

## 6. Activity Feed（活动动态，中栏 Tab 1）

**前端组件**: [Feed.vue 共享模块](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Shared/Modules/Feed.vue)

**API 控制器**: [VaultFeedController::show()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultFeedController.php#L16-L36) — **直接分页查询，不经 Service 层**

**ViewHelper**: [ModuleFeedViewHelper::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L21-L35)

### 数据来源

```php
// VaultFeedController::show() 直接操作
$contactIds = Contact::where('vault_id', $vaultId)->select('id')->get()->toArray();

$items = ContactFeedItem::whereIn('contact_id', $contactIds)
    ->with(['author', 'contact.importantDates'])
    ->orderBy('created_at', 'desc')
    ->paginate(15);   // ← 直接用 Eloquent 分页
```

> **关键说明**：`VaultFeedController` 直接在控制器层用 `ContactFeedItem::whereIn(...)->paginate(15)` 分页查询，**没有经过 Service 层**。每页固定 15 条。

- **查询**: 该 Vault 下所有联系人的动态 feed 项，分页每页 15 条
- **数据库表**: `contact_feed_items`（字段 `author_id`, `contact_id`, `action`, `description`, `feedable_id`, `feedable_type`）
- **关联**: `ContactFeedItem` belongsTo `User`(author), belongsTo `Contact`, morphTo `feedable`
- **返回数据**: `id`, `action`, `author.{name, avatar, url}`, `sentence`, `data`（根据 action 类型不同，由各 ActionFeed* 辅助类构建）, `created_at`

### ActionFeed* 派发分支

`ModuleFeedViewHelper::getData()` 根据 `$item->action` 值，通过 switch 派发到 8 个不同的 ActionFeed 类构建 `data` 字段：

| Action 类型 | 派发类 | feedable 模型 | data 返回结构 |
|------------|--------|-------------|-------------|
| `label_assigned` / `label_removed` | [ActionFeedLabelAssigned](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedLabelAssigned.php) | `Label` | `{ label: { object:{id,name,bg_color,text_color,url}, description }, contact:{...} }` |
| `address_created` / `address_updated` / `address_destroyed` | [ActionFeedAddress](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedAddress.php) | `Address` | `{ address: { object:{id,line_1,line_2,city,province,postal_code,country,type,image,url}, description }, contact:{...} }` |
| `contact_information_created` / `contact_information_updated` / `contact_information_destroyed` | [ActionFeedContactInformation](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedContactInformation.php) | `ContactInformation` | `{ information: { object:{id,label,data,contact_information_type}, description }, contact:{...} }` |
| `pet_created` / `pet_updated` / `pet_destroyed` | [ActionFeedPet](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedPet.php) | `Pet` | `{ pet: { object:{id,name,pet_category}, description }, contact:{...} }` |
| `note_created` / `note_updated` / `note_destroyed` | [ActionFeedNote](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedNote.php) | `Note` | `{ note: { object:{id,title,body(前30字)}, description }, contact:{...} }` |
| `goal_created` / `goal_updated` / `goal_destroyed` | [ActionFeedGoal](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedGoal.php) | `Goal` | `{ goal: { object:{id,name}, description }, contact:{...} }` |
| `mood_tracking_event_added` / `mood_tracking_event_updated` / `mood_tracking_event_deleted` | [ActionFeedMoodTrackingEvent](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedMoodTrackingEvent.php) | `MoodTrackingEvent` | `{ mood_tracking_event: { object:{id,rated_at,note,number_of_hours_slept}, description }, contact:{...} }` |
| **default**（其他所有 action） | [ActionFeedGenericContactInformation](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedGenericContactInformation.php) | —（仅 contact 信息） | `{ id,name,age,avatar,url }` |

> **兜底逻辑**：`contact_created`, `information_updated`, `job_information_updated`, `religion_updated`, `archived`, `unarchived`, `favorited`, `unfavorited`, `changed_avatar`, `important_date_created/updated/destroyed`, `added_to_group`, `removed_from_group`, `added_to_post`, `removed_from_post`, `author_deleted` 等 action 都走 `ActionFeedGenericContactInformation` 兜底分支，只返回 contact 基本信息。

### 展示逻辑

- 时间线样式展示
- 每个 feed 项显示：操作者头像 + 名称 + 动作描述 + 时间
- 部分动作类型附带详情卡片（地址、标签、宠物、笔记等）
- 支持分页加载更多（通过 `paginator.nextPageUrl`）

---

## 7. Life Events（人生事件，中栏 Tab 2）

**前端组件**: [LifeEvent.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Shared/Modules/LifeEvent.vue)

**ViewHelper**: [ModuleLifeEventViewHelper::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageLifeEvents/Web/ViewHelpers/ModuleLifeEventViewHelper.php#L18-L42)

**异步加载 API**: `contact.timeline_event.index`

### 服务端返回结构：Categories 和 Types

`ModuleLifeEventViewHelper::data(Contact $contact, User $user)` 返回的 SSR 初始数据中包含完整的 life_event_categories 树结构，供前端"新增人生事件"弹窗选择使用：

```php
// ModuleLifeEventViewHelper::data() 第 20-24 行
$lifeEventCategoriesCollection = $contact->vault->lifeEventCategories()
    ->with('lifeEventTypes')
    ->orderBy('position', 'asc')
    ->get()
    ->map(fn (LifeEventCategory $lifeEventCategory) => self::dtoLifeEventCategory($lifeEventCategory));
```

嵌套 dto 调用链：
- **dtoLifeEventCategory** ([第44-54行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageLifeEvents/Web/ViewHelpers/ModuleLifeEventViewHelper.php#L44-L54))：`{ id, label, life_event_types: [...] }`
- **dtoLifeEventType** ([第56-62行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageLifeEvents/Web/ViewHelpers/ModuleLifeEventViewHelper.php#L56-L62))：`{ id, label }`

前端拿到的完整结构示例：

```json
{
  "life_event_categories": [
    {
      "id": 1,
      "label": "生活事件",
      "life_event_types": [
        { "id": 1, "label": "搬家" },
        { "id": 2, "label": "旅行" }
      ]
    }
  ]
}
```

数据库层级为三级：`life_event_categories` (Vault level) → `life_event_types` (Category level)，都带 `position` 字段排序。

### TimelineEvent 的前端异步加载

**SSR 初始加载**（`VaultController::show()`）时：
- 仅返回 categories/types 静态结构 + `url.load` 路由
- **不返回任何 timeline events 列表**

**前端 `onMounted` 异步请求**（[LifeEvent.vue 第25-36行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Shared/Modules/LifeEvent.vue#L25-L36)）：

```js
axios.get(props.data.url.load)  // contact.timeline_event.index 路由
  .then((response) => {
    response.data.data.timeline_events.forEach((entry) => {
      localTimelines.value.push(entry);
    });
    paginator.value = response.data.paginator;
  })
```

请求命中 [ContactModuleTimelineEventController::index()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageLifeEvents/Web/Controllers/ContactModuleTimelineEventController.php#L18-L31)：

```php
$timelineEvents = $contact
    ->timelineEvents()
    ->orderBy('started_at', 'desc')
    ->paginate(15);
```

后端通过 [ModuleLifeEventViewHelper::timelineEvents()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageLifeEvents/Web/ViewHelpers/ModuleLifeEventViewHelper.php#L64-L73) 和 `dtoTimelineEvent()` 嵌套返回每个 TimelineEvent 下已有的 life_events。

**"Load previous entries"分页加载更多**（[LifeEvent.vue 第38-49行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Shared/Modules/LifeEvent.vue#L38-L49)）：

```js
axios.get(paginator.value.nextPageUrl)
```

每页固定 15 条 timeline events，通过 `paginator.hasMorePages` 控制底部按钮是否显示。

> **NPE 风险**：后端 `ModuleLifeEventViewHelper::data(Contact $contact, ...)` 签名要求 `Contact` 类型，调用方 [VaultController::show() 第78行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L78) 将 `Auth::user()->getContactInVault($vault)` 的返回值（可能为 null）直接传入，若当前用户在该 Vault 中没有对应的 contact，则 PHP 会抛 TypeError。详见下方"空 contact 导致的 NPE 问题"章节。

### 展示逻辑

- 每个时间线事件可展开/折叠
- 展开后显示包含的人生事件列表
- 每个人生事件显示：摘要 + 类别 > 类型标签、日期、距离、参与者
- 支持 CRUD 操作

---

## 8. Life Metrics（人生指标，中栏 Tab 3）

**前端组件**: [LifeMetrics.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Pages/Vault/Dashboard/Partials/LifeMetrics.vue)

**ViewHelper**: [VaultLifeMetricsViewHelper::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L21-L36)

### 数据来源

```
Vault → lifeMetrics() (HasMany, 表: life_metrics)
      → map → dto(lifeMetric, year, contact)
              → contact = $user->getContactInVault($vault)  ← 登录用户自己的 contact
              → contact → lifeMetrics() (BelongsToMany, 中间表: contact_life_metric)
                        → where('life_metric_id', $id)
```

> **关键说明 1**：Life Metrics **只按登录用户自己的 contact 聚合**，不是 Vault 下所有联系人的总和。每个用户在每个 Vault 中有一个对应的 contact（通过 `$user->getContactInVault($vault)` 获取），统计值基于该 contact 的 `contact_life_metric` 中间表记录。
>
> **关键说明 2**：dto() 方法内部调用了 `self::stats()` 和 `self::years()`，加上自身的查询，**每个 LifeMetric 共触发 5 次数据库查询**：
> - `dto()` 自身：查询该 metric 的所有事件（用于 12 月统计）—— 第 1 次
> - `stats()` 内：weekly / monthly / yearly 各一次 COUNT 查询—— 第 2、3、4 次
> - `years()` 内：查询所有事件提取年份—— 第 5 次

#### 统计值详情（stats 方法，第 109-137 行）

[VaultLifeMetricsViewHelper::stats()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L109-L137)

| 统计值 | 查询逻辑 | 数据库来源 |
|--------|---------|-----------|
| `weekly_events` | 当周记录数（`startOfWeek` ~ `endOfWeek`），独立 COUNT 查询 | `contact_life_metric.pivot.created_at` |
| `monthly_events` | 当月记录数（`startOfMonth` ~ `endOfMonth`），独立 COUNT 查询 | `contact_life_metric.pivot.created_at` |
| `yearly_events` | 当年记录数（`startOfYear` ~ `endOfYear`），独立 COUNT 查询 | `contact_life_metric.pivot.created_at` |

> stats() 方法的三个统计值都是**独立的数据库 COUNT 查询**，不依赖 dto() 中已获取的 events 集合。

#### 月度分布（dto 方法，第 38-94 行）

[VaultLifeMetricsViewHelper::dto()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L38-L94)

对每个 LifeMetric，先查询该 contact 的所有事件，然后在 PHP 层遍历 1-12 月统计每月次数：

| 数据项 | 查询逻辑 |
|--------|---------|
| `months[{id, friendly_name, events}]` | 遍历 1-12 月，在 PHP 层统计 `contact_life_metric.pivot.created_at` 落在该月的事件数 |
| `max_number_of_events` | 12 个月中事件数的最大值，用于图表 Y 轴缩放 |

#### 年份列表（years 方法，第 96-107 行）

[VaultLifeMetricsViewHelper::years()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L96-L107)

| 数据项 | 查询逻辑 |
|--------|---------|
| `years[{year}]` | 独立查询所有事件，从 `contact_life_metric.pivot.created_at` 提取所有不重复年份，降序排列 |

> **NPE 风险**：
> - [VaultLifeMetricsViewHelper::data() 第24行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L24) `$contact = $user->getContactInVault($vault)` 返回值直接传给 `self::dto()`
> - [dto() 第38行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L38) 签名 `dto(LifeMetric $lifeMetric, int $year, Contact $contact)` 强类型要求 Contact，若传入 null 直接 TypeError
> - 内部 `$contact->lifeMetrics()` 调用前也无 null 检查

### 展示逻辑

- 每个 Metric 显示：标签名 + 周/月/年统计（格式：周 / 月 / 年）+ "+1" 按钮
- 点击 "+1" 调用 `vault.life_metrics.contact.store` API 记录一次事件
- 点击统计数字可展开月度柱状图
- 支持 CRUD 操作（创建/编辑/删除 Metric）

---

## 9. Default Tab（默认 Tab 切换）

**前端处理**: [Dashboard/Index.vue 第29-40行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Pages/Vault/Dashboard/Index.vue#L29-L40)

**数据库字段**: [Vault::$fillable](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Models/Vault.php#L39-L52) 中 `'default_activity_tab'`

**Service**: [UpdateVaultDashboardDefaultTab](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Services/UpdateVaultDashboardDefaultTab.php)

**API Controller**: [VaultDefaultTabOnDashboardController::update()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultDefaultTabOnDashboardController.php#L12-L26)

### 数据流

```
SSR 渲染时:
VaultController::show() → $vault->default_activity_tab → Inertia prop 'defaultTab'
                                ↓
                         Index.vue: const currentTab = ref(props.defaultTab)
                                ↓
前端切换时 (changeTab):
  currentTab.value = tab
  form.default_activity_tab = tab
  axios.put(props.url.default_tab, form)
                                ↓
VaultDefaultTabOnDashboardController::update()
  → (new UpdateVaultDashboardDefaultTab)->execute($data)
    → $this->vault->default_activity_tab = $data['default_activity_tab']
    → save()
```

- 可能值：`'activity'` | `'life_events'` | `'life_metrics'`
- 注意：`$vault->default_activity_tab` 可为 null（数据库默认值），此时前端 `ref(props.defaultTab)` 会得到 `null`，导致初始没有任何 Tab 被选中高亮
- 前端切换 Tab 后立即发送 PUT 请求持久化，用户下次进入 Dashboard 时直接显示上次选择的 Tab

---

## 10. 空 Contact 导致的 NPE 问题

### 根源：`User::getContactInVault()` 可返回 null

[User::getContactInVault()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Models/User.php#L267-L282) 的返回类型是 `?Contact`，在以下两种情况下返回 `null`：

```php
public function getContactInVault(Vault $vault): ?Contact
{
    $entry = $this->vaults()
        ->wherePivot('vault_id', $vault->id)
        ->first();

    if ($entry === null) {      // ← 情况1: 用户不在该 Vault 中
        return null;
    }

    try {
        return Contact::findOrFail($entry->pivot->contact_id);
    } catch (ModelNotFoundException) {  // ← 情况2: pivot 指向的 contact 已被删除
        return null;
    }
}
```

### Dashboard 中所有 NPE 风险点

| 位置 | 代码 | 风险 |
|------|------|------|
| [VaultController.php:68](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L68) | `$contact = Auth::user()->getContactInVault($vault)` | 变量本身可 null，后续传入多个 ViewHelper |
| [VaultController.php:78](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L78) | `ModuleLifeEventViewHelper::data($contact, Auth::user())` | 签名要求 `Contact`，传 null → TypeError |
| [VaultLifeMetricsViewHelper.php:24](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L24) → 第38行 dto() 签名 | `dto(LifeMetric, int, Contact $contact)` | 签名要求 `Contact`，传 null → TypeError |
| [VaultShowViewHelper.php:189](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L189) | `'contact' => $user->getContactInVault($vault)->id` | 直接 `->id`，无 null 检查 |
| [VaultLifeEventController.php:17](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultLifeEventController.php#L17) → 第19行 | `$contact->timelineEvents()` | 直接调用方法，无 null 检查 |
| [LifeMetricController.php:29](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/Controllers/LifeMetricController.php#L29) → 第32行 | `VaultLifeMetricsViewHelper::dto($lifeMetric, ..., $contact)` | 签名要求 Contact |
| [LifeMetricController.php:48](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/Controllers/LifeMetricController.php#L48) → 第51行 | `VaultLifeMetricsViewHelper::dto($lifeMetric, ..., $contact)` | 同上 |
| [LifeMetricContactController.php:27](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/Controllers/LifeMetricContactController.php#L27) | 后续使用 $contact | 同上 |

### 触发场景

1. **新创建的 Vault**：用户被加入 Vault 后，如果 Vault 初始化流程（创建用户自身的 contact 记录）未完成或失败，则 `contact_vault_user` pivot 中的 `contact_id` 指向不存在的 contact
2. **用户被移除后重新加入**：旧的 pivot contact 记录可能已被清理
3. **数据一致性问题**：手动删除了用户在 Vault 中的 contact 记录而未更新 pivot

### 当前代码中的防护

- 部分 Controller 有 `authorizeResource(Vault::class)` 策略检查（保证用户在 Vault 中），但仅保证 pivot 行存在，**不保证 pivot 指向的 contact 存在**
- `getContactInVault()` 内部对 `findOrFail` 做了 try/catch 返回 null，但**调用方几乎都没处理 null**

---

## 11. Timezone 与 Carbon Mutable 边界

### DateHelper：Mutable Carbon 作为事实标准，timezone 就地修改

[DateHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Helpers/DateHelper.php) 所有方法都接受 `Carbon`（非 CarbonImmutable）参数：

```php
public static function format(Carbon $date, User $user): string
{
    return $date->isoFormat($user->date_format);  // ← 不转换时区，用调用方传入的时区
}

public static function formatDate(Carbon $date, ?string $timezone = null): string
{
    if ($timezone) {
        $date->setTimezone($timezone);   // ← ⚠️ 就地修改 Mutable Carbon!
    }
    return $date->isoFormat(trans('format.date'));
}
```

> **Mutable 风险**：`DateHelper::formatDate()` / `formatShortDateWithTime()` / `formatDayAndMonthInParenthesis()` 都调用 `$date->setTimezone()`，**就地修改传入的 Mutable Carbon 对象**。如果调用方后续还要使用该 Carbon 对象，其时区已经被改变。

### Dashboard 中的 timezone 使用场景

| 位置 | 代码 | 说明 |
|------|------|------|
| [VaultShowViewHelper.php:182](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L182) | `Carbon::now($user->timezone)->format('Y-m-d')` | Mood card 当天日期，传入 timezone 构造新 Carbon |
| [ModuleLifeEventViewHelper.php:28](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageLifeEvents/Web/ViewHelpers/ModuleLifeEventViewHelper.php#L28) | `Carbon::now($user->timezone)->format('Y-m-d')` | Life events 当前日期 |
| [ModuleLifeEventViewHelper.php:29](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageLifeEvents/Web/ViewHelpers/ModuleLifeEventViewHelper.php#L29) | `DateHelper::format(Carbon::now($user->timezone), $user)` | Life events 人性化日期 |
| [VaultShowViewHelper.php:38](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L38) | `Carbon::now()->copy()` + `$currentDate->second = 0` | Reminders 基准时间，**未使用用户时区** |
| [VaultShowViewHelper.php:62](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L62) | `Carbon::createFromFormat('Y-m-d H:i:s', $reminder->pivot->scheduled_at)` | Reminder 显示时间，从 DB 字符串解析，**未转换用户时区** |
| [VaultShowViewHelper.php:125](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L125) | `Carbon::now()->addDays(30)` | DueTasks 30 天窗口上界，**未使用用户时区** |
| [VaultShowViewHelper.php:136](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L136) | `$task->due_at->isPast()` | Eloquent cast 出来的 Carbon，使用 DB/App 配置时区 |

### VaultLifeMetricsViewHelper 中 Mutable / Immutable 混用

同一文件中**混用 Carbon 和 CarbonImmutable**，存在不一致风险：

| 位置 | 类型 | 用途 |
|------|------|------|
| [第45行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L45) | `Carbon` (Mutable) | `Carbon::parse(...)->year` 过滤年度事件 |
| [第53行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L53) | `CarbonImmutable` | `CarbonImmutable::parse(...)->month` 在循环中判断月份 |
| [第62行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L62) | `CarbonImmutable` | 构造月度日期对象 |
| [第103行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L103) | `Carbon` (Mutable) | 提取年份 |
| [第114-115行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L114-L115) | `CarbonImmutable` | stats() 中 weekly 时间窗口 |
| [第121-122行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L121-L122) | `CarbonImmutable` | stats() 中 monthly 时间窗口 |
| [第128-129行](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L128-L129) | `CarbonImmutable` | stats() 中 yearly 时间窗口 |

> **边界提醒**：
> - `DateHelper` 只接受 **Mutable Carbon**，不能传 CarbonImmutable（会 TypeError）
> - `Carbon::now()` 默认使用 `config('app.timezone')`（通常是 UTC），而不是用户时区
> - 当需要转换时区时，如果传入的是 Mutable Carbon，**DateHelper 会就地修改**，后续使用同一对象时需注意
> - 对于数据库中的 `created_at` / `due_at` 等时间戳，Eloquent 默认按 `app.timezone` 解析；与用户时区不一致时需显式 `setTimezone`

---

## 12. Controller 与 ViewHelper 的职责划分

### 分层概览

Monica 的 Dashboard 数据流遵循三层架构：

```
┌─────────────────────────────────────────────────────────┐
│ Controller (HTTP 层 / 路由层)                            │
│  • 接收 Request、Auth 鉴权、参数提取                      │
│  • 调用 Service 处理写操作                               │
│  • 编排多个 ViewHelper 收集读数据                         │
│  • 返回 Inertia Render 或 JSON Response                  │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│ Service (领域层 / 写操作层)                              │
│  • 验证 rules()、权限 permissions()                      │
│  • 执行业务写入（创建/更新/删除）                          │
│  • 返回 Model 或 true/false                              │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│ ViewHelper (展示层 / 读操作层)                           │
│  • 纯静态方法，无副作用（不写入 DB）                      │
│  • 接收 Model 作为入参，组装前端所需 array                │
│  • 负责 DTO 转换：Eloquent Model → 前端 props 结构        │
│  • 负责拼接 route URL                                    │
└─────────────────────────────────────────────────────────┘
```

### Dashboard 中各 Controller 的职责

| Controller | 职责 | 调用的 Service | 调用的 ViewHelper |
|-----------|------|----------------|------------------|
| [VaultController::show()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultController.php#L66-L89) | 渲染 Dashboard 页面（SSR），一次性编排所有卡片数据 | —（只读） | `VaultShowViewHelper::*`, `ModuleLifeEventViewHelper::data()`, `VaultLifeMetricsViewHelper::data()` |
| [VaultFeedController::show()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultFeedController.php#L16-L36) | Feed AJAX 分页接口，**直接在 Controller 层做 Eloquent 查询**（越过 Service 层） | —（只读） | `ModuleFeedViewHelper::data()` |
| [VaultLifeEventController::show()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultLifeEventController.php#L14-L28) | Life Events AJAX 分页接口，直接做查询 | —（只读） | `ModuleLifeEventViewHelper::timelineEvents()` |
| [VaultDefaultTabOnDashboardController::update()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultDefaultTabOnDashboardController.php#L12-L26) | 保存默认 Tab | `UpdateVaultDashboardDefaultTab` | — |
| [LifeMetricController::store/update/destroy](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/Controllers/LifeMetricController.php) | Life Metric CRUD | `CreateLifeMetric`, `UpdateLifeMetric`, `DestroyLifeMetric` | `VaultLifeMetricsViewHelper::dto()` |
| [LifeMetricContactController](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/Controllers/LifeMetricContactController.php) | Life Metric "+1" 记录一次事件 | `IncrementLifeMetric` | — |

### ViewHelper 规范

所有 Dashboard 相关 ViewHelper 都遵循以下模式：

```php
class XxxViewHelper
{
    // 入口方法：接收 Model 入参，返回 array
    public static function data(Vault $vault, User $user, ...): array
    {
        // 1. Eloquent 查询（只读）
        // 2. 嵌套调用 dto() 做单条记录转换
        // 3. 拼接 route URL
        // 4. 返回前端所需完整结构
    }

    // DTO 辅助方法：单条 Model → array
    public static function dto(SomeModel $model): array { ... }
}
```

Dashboard 中涉及的 ViewHelper 文件：

| ViewHelper | 所在域 | 职责 |
|-----------|--------|------|
| [VaultShowViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php) | Vault | favorites / lastUpdated / upcomingReminders / dueTasks / moodTrackingEvents |
| [ModuleFeedViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php) | Contact\Feed | Feed 列表 DTO + ActionFeed\* 派发 |
| [ModuleLifeEventViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageLifeEvents/Web/ViewHelpers/ModuleLifeEventViewHelper.php) | Contact\LifeEvents | life_event_categories 树 + timeline events DTO |
| [VaultLifeMetricsViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php) | Vault\LifeMetrics | Life Metrics 统计（dto/stats/years） |

### 例外：Feed / Timeline 分页查询越过 Service 层

不同于标准 CRUD 流程（Controller → Service → DB），**读操作的分页查询直接在 Controller 层完成**：

- `VaultFeedController::show()` 直接调用 `ContactFeedItem::whereIn(...)->paginate(15)`
- `VaultLifeEventController::show()` 直接调用 `$contact->timelineEvents()->paginate(15)`

这是代码库的约定：**Service 层只处理写操作（CUD），读操作（R）可以直接在 Controller 中用 Eloquent 查询后交给 ViewHelper 转 DTO**。

---

## 数据库表关系图

```
vaults
 ├── contacts (1:N)
 │    ├── contact_reminders (1:N)
 │    │    └── contact_reminder_scheduled (M:N with user_notification_channels)
 │    ├── contact_tasks (1:N)
 │    ├── mood_tracking_events (1:N)
 │    ├── contact_feed_items (1:N)
 │    └── contact_life_metric (M:N with life_metrics, pivot: created_at)
 │
 ├── mood_tracking_parameters (1:N)
 ├── life_metrics (1:N)
 ├── life_event_categories (1:N)
 │    └── life_event_types (1:N)
 ├── timeline_events (1:N)
 │    ├── life_events (1:N)
 │    └── timeline_event_participants (M:N with contacts)
 │
 └── vault_user (M:N with users, pivot: permission, contact_id)
      └── contact_vault_user (M:N with contacts, pivot: is_favorite)
```

---

## 请求时序汇总

1. **页面初始加载**（SSR，Inertia Props）：
   - `VaultController::show()` 一次性调用所有 ViewHelper 方法
   - `lastUpdatedContacts`, `upcomingReminders`, `favorites`, `dueTasks`, `moodTrackingEvents`, `lifeEvents`, `lifeMetrics`, `defaultTab` 全部在服务端查询完成

2. **前端异步请求**：
   - Activity Feed: mounted 时 `GET vault.feed.show`（VaultFeedController 直接分页，15条/页），分页加载更多
   - Life Events 时间线: mounted 时 `GET contact.timeline_event.index`，分页加载更多
   - Life Metrics "+1": `POST vault.life_metrics.contact.store`
   - Mood Record: `POST contact.mood_tracking_event.store`
   - Task Toggle: `PUT contact.task.toggle`
   - Default Tab: `PUT vault.default_tab.update`

---

## 关键统计值速查

| Card | 统计/展示内容 | 汇总方式 | 数据库表/字段 |
|------|-------------|---------|-------------|
| Favorites | 收藏联系人列表 | `contact_vault_user.is_favorite = true`（当前用户） | `contact_vault_user.is_favorite` |
| Last Updated | 最近更新 5 人 | `contacts.last_updated_at DESC LIMIT 5` | `contacts.last_updated_at` |
| Upcoming Reminders | 30 天内未触发提醒（含过期） | `contact_reminder_scheduled.scheduled_at <= now+30d AND triggered_at IS NULL`（只设上界） | `contact_reminder_scheduled.scheduled_at, triggered_at` |
| Due Tasks | 30 天内到期未完成任务（含过期） | `contact_tasks.completed = false AND due_at <= now+30d`（只设上界） | `contact_tasks.completed, contact_tasks.due_at` |
| Mood Tracking | 心情参数列表 + 录入表单 | `mood_tracking_parameters WHERE vault_id = ?` | `mood_tracking_parameters.*` |
| Activity Feed | 联系人操作动态流 | `ContactFeedItem::whereIn(contact_id IN vault_contacts) ORDER BY created_at DESC` 直接分页 | `contact_feed_items.*` |
| Life Events | 时间线 + 人生事件 | `timeline_events + life_events WHERE vault_id = ?` | `timeline_events.*, life_events.*` |
| Life Metrics - weekly | 本周事件次数（登录用户 contact） | 独立 COUNT 查询：`contact_life_metric WHERE created_at IN current week` | `contact_life_metric.created_at` |
| Life Metrics - monthly | 本月事件次数（登录用户 contact） | 独立 COUNT 查询：`contact_life_metric WHERE created_at IN current month` | `contact_life_metric.created_at` |
| Life Metrics - yearly | 本年事件次数（登录用户 contact） | 独立 COUNT 查询：`contact_life_metric WHERE created_at IN current year` | `contact_life_metric.created_at` |
| Life Metrics - months | 12 月事件分布（登录用户 contact） | dto 查询全部事件 + PHP 层遍历 1-12 月统计 | `contact_life_metric.created_at` |
| Default Tab | 用户偏好的默认 Tab | `vaults.default_activity_tab` 字段（可 null） | `vaults.default_activity_tab` |

---

## 修订记录

| 版本 | 修订内容 |
|------|---------|
| v3 | 1. 补充 **Life Events categories/types 服务端返回结构**（dtoLifeEventCategory → dtoLifeEventType 嵌套 DTO）<br>2. 补充 **TimelineEvent 前端异步加载流程**（SSR 只返回结构 + onMounted 异步拉取 + 分页 loadMore）<br>3. 补充 **Default Tab 机制**（Vault.default_activity_tab 字段 + 前端 changeTab PUT 持久化）<br>4. 补充 **空 Contact NPE 问题**（getContactInVault 返回 null 的所有风险点、触发场景、防护情况）<br>5. 补充 **Timezone 与 Carbon Mutable 边界**（DateHelper 就地修改 Mutable Carbon、timezone 使用不一致、VaultLifeMetricsViewHelper 中 Mutable/Immutable 混用）<br>6. 补充 **Controller 与 ViewHelper 职责划分**（三层架构概览、各 Controller 职责表、ViewHelper 规范、读操作越过 Service 层的约定） |
| v2 | 1. LifeMetrics 明确按**登录用户 contact** 聚合（非 Vault 全量），dto/stats/years 各自独立查询（每 Metric 共 5 次 DB 查询）<br>2. Feed 标注 **VaultFeedController 直接分页**查询，不经 Service 层<br>3. Reminders 与 DueTasks 明确**只设上界、含过期**（scheduled_at / due_at 均只有 <= now+30d 条件）<br>4. 补齐 **8 个 ActionFeed\* 类**的派发分支明细及数据结构 |
