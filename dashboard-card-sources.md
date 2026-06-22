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

### 数据来源

初始数据由 ViewHelper 提供（基于当前用户在 Vault 中的 contact）：

```
User → getContactInVault($vault) → vault → lifeEventCategories() (HasMany)
      → with('lifeEventTypes')
      → orderBy('position', 'asc')
```

时间线事件列表由前端 mounted 时异步请求：

```
GET contact.timeline_event.index → TimelineEvent 列表
      → 每个 TimelineEvent 包含 lifeEvents (HasMany)
```

- **数据库表**:
  - `life_event_categories`（Vault 级别）
  - `life_event_types`（属于 Category）
  - `timeline_events`（字段 `vault_id`, `started_at`, `label`, `collapsed`）
  - `life_events`（字段 `timeline_event_id`, `emotion_id`, `summary`, `description`, `happened_at`, `costs`, `currency_id`, `distance`, `distance_unit`, `from_place`, `to_place`, `place`, `life_event_type_id`, `collapsed`）
  - `timeline_event_participants`（中间表）
- **返回数据**: `contact`, `current_date`, `life_event_categories[{id, label, life_event_types}]`, `url.{load, store}`

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

### 展示逻辑

- 每个 Metric 显示：标签名 + 周/月/年统计（格式：周 / 月 / 年）+ "+1" 按钮
- 点击 "+1" 调用 `vault.life_metrics.contact.store` API 记录一次事件
- 点击统计数字可展开月度柱状图
- 支持 CRUD 操作（创建/编辑/删除 Metric）

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
   - `lastUpdatedContacts`, `upcomingReminders`, `favorites`, `dueTasks`, `moodTrackingEvents`, `lifeEvents`, `lifeMetrics` 全部在服务端查询完成

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

---

## 修订记录

| 版本 | 修订内容 |
|------|---------|
| v2 | 1. LifeMetrics 明确按**登录用户 contact** 聚合（非 Vault 全量），dto/stats/years 各自独立查询（每 Metric 共 5 次 DB 查询）<br>2. Feed 标注 **VaultFeedController 直接分页**查询，不经 Service 层<br>3. Reminders 与 DueTasks 明确**只设上界、含过期**（scheduled_at / due_at 均只有 <= now+30d 条件）<br>4. 补齐 **8 个 ActionFeed\* 类**的派发分支明细及数据结构 |
