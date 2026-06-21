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
              → wherePivot('scheduled_at', '<=', now + 30 days)
              → wherePivot('triggered_at', null)
              → orderByPivot('scheduled_at', 'asc')
```

然后对每个 reminder 检查其 `contact.vault_id` 是否匹配当前 Vault，过滤掉不属于当前 Vault 的提醒，最后按 `contact_reminder_id` 去重。

- **涉及数据库表**:
  - `vaults` → `vault_user`（中间表）→ `users`
  - `users` → `user_notification_channels`
  - `user_notification_channels` → `contact_reminder_scheduled`（中间表）→ `contact_reminders`
  - `contact_reminders` → `contacts`
- **关键过滤条件**:
  - `scheduled_at <= 当前时间 + 30天`
  - `triggered_at IS NULL`（尚未触发的提醒）
  - `contact.vault_id == 当前 Vault ID`
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
      → where('due_at', '<=', now + 30 days)
      → sortBy('due_at')
```

- **查询**: 该 Vault 下所有联系人的未完成任务，且截止日期在 30 天以内
- **数据库表**: `contact_tasks`（字段 `completed`, `due_at`）
- **关联**: `Contact` hasMany `ContactTask`
- **关键过滤条件**:
  - `completed = false`
  - `due_at <= 当前时间 + 30天`
- **返回数据**: `id`, `label`, `description`, `completed`, `completed_at`, `due_at.{formatted, value, is_late}`, `url.toggle`, `contact.{id, name, avatar, url.show}`
- **统计值 `is_late`**: `due_at->isPast()` 判断任务是否已逾期

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

### 展示逻辑

- 标题："Record your mood"（带 CloudSun 图标）
- 默认展示"How are you?"提示 + "Record your mood"按钮
- 点击后展开表单：选择心情参数（单选）+ 可选日期/备注/睡眠时长
- 提交后调用 `contact.mood_tracking_event.store` API
- 成功后显示🎉 + "Your mood has been recorded!"

---

## 6. Activity Feed（活动动态，中栏 Tab 1）

**前端组件**: [Feed.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Pages/Vault/Dashboard/Partials/Feed.vue)（通过 [Feed.vue 共享模块](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Shared/Modules/Feed.vue)）

**API 控制器**: [VaultFeedController::show()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultFeedController.php#L16-L36)

**ViewHelper**: [ModuleFeedViewHelper::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L21-L35)

### 数据来源

```
Contact::where('vault_id', $vaultId) → pluck('id')
ContactFeedItem::whereIn('contact_id', $contactIds)
      → with(['author', 'contact.importantDates'])
      → orderBy('created_at', 'desc')
      -> paginate(15)
```

- **查询**: 该 Vault 下所有联系人的动态 feed 项，分页每页 15 条
- **数据库表**: `contact_feed_items`（字段 `author_id`, `contact_id`, `action`, `description`, `feedable_id`, `feedable_type`）
- **关联**: `ContactFeedItem` belongsTo `User`(author), belongsTo `Contact`, morphTo `feedable`
- **action 类型**: `contact_created`, `information_updated`, `note_created/updated/deleted`, `address_created/updated/destroyed`, `label_assigned/removed`, `pet_created/updated/destroyed`, `goal_created/updated/destroyed`, `mood_tracking_event_added/updated/deleted` 等
- **返回数据**: `id`, `action`, `author.{name, avatar, url}`, `sentence`, `data`（根据 action 类型不同，由各 ActionFeed* 辅助类构建）, `created_at`

### 展示逻辑

- 时间线样式展示
- 每个 feed 项显示：操作者头像 + 名称 + 动作描述 + 时间
- 部分动作类型附带详情卡片
- 支持分页加载更多

---

## 7. Life Events（人生事件，中栏 Tab 2）

**前端组件**: [LifeEvent.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/resources/js/Shared/Modules/LifeEvent.vue)

**ViewHelper**: [ModuleLifeEventViewHelper::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/77-monica/app/Domains/Contact/ManageLifeEvents/Web/ViewHelpers/ModuleLifeEventViewHelper.php#L18-L42)

**异步加载 API**: `contact.timeline_event.index`

### 数据来源

初始数据由 ViewHelper 提供：

```
Contact(用户在Vault中的联系人) → vault → lifeEventCategories() (HasMany)
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
              → contact → lifeMetrics() (BelongsToMany, 中间表: contact_life_metric)
                        → where('life_metric_id', $id)
                        → 过滤当年事件
```

#### 统计值详情（stats 方法，第 109-137 行）

| 统计值 | 查询逻辑 | 数据库来源 |
|--------|---------|-----------|
| `weekly_events` | `contact_life_metric` 当周记录数（`startOfWeek` ~ `endOfWeek`） | `contact_life_metric.pivot.created_at` |
| `monthly_events` | `contact_life_metric` 当月记录数（`startOfMonth` ~ `endOfMonth`） | `contact_life_metric.pivot.created_at` |
| `yearly_events` | `contact_life_metric` 当年记录数（`startOfYear` ~ `endOfYear`） | `contact_life_metric.pivot.created_at` |

#### 月度分布（dto 方法，第 38-94 行）

对每个 LifeMetric，按当年 1-12 月统计每月事件次数，用于柱状图展示：

| 数据项 | 查询逻辑 |
|--------|---------|
| `months[{id, friendly_name, events}]` | 遍历 1-12 月，统计 `contact_life_metric.pivot.created_at` 落在该月的事件数 |
| `max_number_of_events` | 12 个月中事件数的最大值，用于图表 Y 轴缩放 |

#### 年份列表（years 方法，第 96-107 行）

| 数据项 | 查询逻辑 |
|--------|---------|
| `years[{year}]` | 从 `contact_life_metric.pivot.created_at` 提取所有不重复年份，降序排列 |

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
   - Activity Feed: mounted 时 `GET vault.feed.show`，分页加载更多
   - Life Events 时间线: mounted 时 `GET contact.timeline_event.index`，分页加载更多
   - Life Metrics "+1": `POST vault.life_metrics.contact.store`
   - Mood Record: `POST contact.mood_tracking_event.store`
   - Task Toggle: `PUT contact.task.toggle`
   - Default Tab: `PUT vault.default_tab.update`

---

## 关键统计值速查

| Card | 统计/展示内容 | 汇总方式 | 数据库表/字段 |
|------|-------------|---------|-------------|
| Favorites | 收藏联系人列表 | `contact_vault_user.is_favorite = true` | `contact_vault_user.is_favorite` |
| Last Updated | 最近更新 5 人 | `contacts.last_updated_at DESC LIMIT 5` | `contacts.last_updated_at` |
| Upcoming Reminders | 30 天内未触发提醒 | `contact_reminder_scheduled.scheduled_at <= now+30d AND triggered_at IS NULL` | `contact_reminder_scheduled.scheduled_at, triggered_at` |
| Due Tasks | 30 天内到期未完成任务 | `contact_tasks.completed = false AND due_at <= now+30d` | `contact_tasks.completed, contact_tasks.due_at` |
| Mood Tracking | 心情参数列表 + 录入表单 | `mood_tracking_parameters WHERE vault_id = ?` | `mood_tracking_parameters.*` |
| Activity Feed | 联系人操作动态流 | `contact_feed_items WHERE contact_id IN (vault_contacts) ORDER BY created_at DESC` | `contact_feed_items.*` |
| Life Events | 时间线 + 人生事件 | `timeline_events + life_events WHERE vault_id = ?` | `timeline_events.*, life_events.*` |
| Life Metrics - weekly | 本周事件次数 | `COUNT contact_life_metric WHERE created_at IN current week` | `contact_life_metric.created_at` |
| Life Metrics - monthly | 本月事件次数 | `COUNT contact_life_metric WHERE created_at IN current month` | `contact_life_metric.created_at` |
| Life Metrics - yearly | 本年事件次数 | `COUNT contact_life_metric WHERE created_at IN current year` | `contact_life_metric.created_at` |
| Life Metrics - months | 12 月事件分布 | 遍历 1-12 月统计 `contact_life_metric.created_at` | `contact_life_metric.created_at` |
