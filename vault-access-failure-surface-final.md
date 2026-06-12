# Monica CRM 成员移除后的故障面完整分析报告（Final）

> **前置假设**：成员 A 被管理者从 Vault X 移除，`RemoveVaultAccess` 服务执行了软删除操作（`Contact::find($contactId)->delete()`）。此时：
> - `user_vault` 表：记录**仍然存在**（软删除不触发 CASCADE）
> - `contacts` 表：对应「本人联系人」的 `deleted_at` 被设置
> - 所有权限 Gate 判断：继续返回 `true`（只查 user_vault）
> - `getContactInVault()`：返回 `null`（`Contact::findOrFail` 捕获 ModelNotFoundException）

---

## 一、核心链路与三种机制总览

### 1.1 数据源的双重标准（根因）

| 判断维度 | 数据源 | 软删除影响 | 移除后结果 |
|---------|--------|-----------|-----------|
| **权限判断** | `user_vault` 中间表 | ❌ 无影响 | ✅ 继续放行 |
| **联系人映射** | `contacts` 表 + Eloquent SoftDeletes 全局 Scope | ✅ 自动过滤 | ❌ 返回 null |

代码中存在隐含假设：「只要权限判断通过，`getContactInVault()` 就一定能返回非空 Contact」。软删除破坏了这个不变量。

### 1.2 三种故障机制定义

| 类别 | 标签 | 定义 | 用户感知 |
|-----|------|------|---------|
| 🔴 崩溃级 | ERROR | `null->method()` 或 `null->property` 触发 `TypeError`/`Error` | 白屏 / 500 / 接口报错 |
| 🟡 异常级 | EMPTY | 有判空保护 `if (!$contact) return null`，或 UserHelper 内部降级 | 局部信息缺失 / 显示空白 / 列表少记录 |
| 🟢 放行级 | PASS | 完全不依赖 `getContactInVault()`，仅依赖权限 Gate | 功能正常，用户完全不知道权限已被撤销 |

---

## 二、🔴 崩溃级故障完整清单（11处）

### 🔴 CRASH-01：日历月视图 / 日详情

| 项目 | 内容 |
|-----|------|
| **入口** | `vault.calendar.month` / `vault.calendar.day` 路由 |
| **控制器** | [VaultCalendarController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageCalendar/Web/Controllers/VaultCalendarController.php) |
| **崩溃点** | [VaultCalendarIndexViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageCalendar/Web/ViewHelpers/VaultCalendarIndexViewHelper.php#L98-L116) `getMood()` 第100-102行 |
| **代码** | `$contact = $user->getContactInVault($vault); return $contact->moodTrackingEvents()...` |
| **触发** | 月视图 `buildMonth()` 每天调用一次 `getMood()`；日查询 `getDayInformation()` 也调用 |
| **感知** | 日历页完全白屏，AJAX 接口返回 500 |

### 🔴 CRASH-02：情绪年度报表页

| 项目 | 内容 |
|-----|------|
| **入口** | `vault.reports.mood_tracking_events.index` 路由 |
| **控制器** | [ReportMoodTrackingEventController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageReports/Web/Controllers/ReportMoodTrackingEventController.php) |
| **崩溃点** | [ReportMoodTrackingEventIndexViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageReports/Web/ViewHelpers/ReportMoodTrackingEventIndexViewHelper.php#L28-L36) `year()` 第30-32行 |
| **代码** | `$contact = $user->getContactInVault($vault); $moodTrackingEvents = $contact->moodTrackingEvents()...` |
| **感知** | 「Vault > Reports > Mood Tracking」页整页白屏 |

### 🔴 CRASH-03：Dashboard 心情记录 URL 生成

| 项目 | 内容 |
|-----|------|
| **入口** | Dashboard 渲染 |
| **调用链** | `VaultController@show` → `VaultShowViewHelper::moodTrackingEvents()` |
| **崩溃点** | [VaultShowViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L168-L193) 第189行 |
| **代码** | `'contact' => $user->getContactInVault($vault)->id` |
| **感知** | Dashboard 整页白屏（渲染时崩溃） |

### 🔴 CRASH-04：生活事件 Feed 接口

| 项目 | 内容 |
|-----|------|
| **入口** | Dashboard 异步加载：`GET /vault/{vaultId}/life-events` |
| **控制器** | [VaultLifeEventController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/Controllers/VaultLifeEventController.php#L14-L28) 第17-20行 |
| **代码** | `$contact = Auth::user()->getContactInVault($vault); $timelineEvents = $contact->timelineEvents()...` |
| **感知** | Dashboard 中生活事件模块接口 500，该模块显示错误 |

### 🔴 CRASH-05：Dashboard 生活指标模块

| 项目 | 内容 |
|-----|------|
| **入口** | Dashboard 渲染 |
| **调用链** | `VaultController@show` → `VaultLifeMetricsViewHelper::data()` |
| **崩溃点** | [VaultLifeMetricsViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageLifeMetrics/Web/ViewHelpers/VaultLifeMetricsViewHelper.php#L21-L94) 第24-26行 + 第38行 |
| **代码** | `self::dto($lifeMetric, $year, $contact)` 传入 null，但 `dto()` 参数类型声明为 `Contact $contact` |
| **触发条件** | Vault 中存在至少一条 lifeMetric 记录 |
| **感知** | Dashboard 整页白屏（与 CRASH-03 谁先触发取决于渲染顺序） |

### 🔴 CRASH-06：联系人详情页（影响范围最广）

| 项目 | 内容 |
|-----|------|
| **入口** | 任何联系人详情页 `contact.show`、`contact.page.show` |
| **崩溃点** | [ContactShowViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php#L41-L151) 共 4 处 |
| **代码位置** | data() 第63-64行、dataForTemplatePage() 第117-118行 |
| **代码** | `$user->getContactInVault($contact->vault)->id !== $contact->id`（判断是否能归档/删除） |
| **感知** | **所有联系人详情页白屏**——包括被移除成员自己的「本人联系人」和其他普通联系人 |

### 🔴 CRASH-07：Dashboard 最常访问（概率性）

| 项目 | 内容 |
|-----|------|
| **入口** | Dashboard 「Most Consulted」模块 |
| **崩溃点** | [VaultMostConsultedViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/Search/Web/ViewHelpers/VaultMostConsultedViewHelper.php#L13-L39) 第26-29行 |
| **代码** | `DB::table('contact_vault_user')` 查原生表（绕过软删除过滤），拿到被软删的 contact_id，后续 `Contact::find()` 返回 null → `$contact->id` 崩溃 |
| **触发条件** | 该用户曾将「本人联系人」加入收藏/查看过，且该记录在 contact_vault_user 表中 |
| **感知** | 概率性 Dashboard 白屏，取决于最常访问列表是否包含被软删的 ID |

### 🔴 CRASH-08：生活指标增量接口

| 项目 | 内容 |
|-----|------|
| **入口** | `POST /vault/{vaultId}/life-metrics/{lifeMetricId}/contact` |
| **服务层** | [IncrementLifeMetric.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageLifeMetrics/Services/IncrementLifeMetric.php#L39-L51) 第46-48行 |
| **代码** | `$contact = $this->author->getContactInVault($this->vault); $contact->lifeMetrics()->save($lifeMetric);` |
| **前置** | BaseService 权限校验通过（因为 Gate 放行），执行到业务代码才崩溃 |
| **感知** | 接口 500，操作失败 |

### 🔴 CRASH-09：生活指标 CRUD 响应构建

| 项目 | 内容 |
|-----|------|
| **入口** | LifeMetric `store` / `update` / `contact.store` |
| **控制器** | [LifeMetricController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageLifeMetrics/Web/Controllers/LifeMetricController.php) + [LifeMetricContactController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageLifeMetrics/Web/Controllers/LifeMetricContactController.php) |
| **崩溃点** | 三处 `VaultLifeMetricsViewHelper::dto($lifeMetric, $year, $contact)` 传 null |
| **特点** | 业务操作（Create/Update/Increment）**实际已完成**，数据库已写入，但构造响应时崩溃 |
| **感知** | 前端看到 500 以为失败了，刷新后发现数据其实已改动 |

### 🔴 CRASH-10：CalDAV 任务同步（静默失败）

| 项目 | 内容 |
|-----|------|
| **入口** | 后台 CalDAV 同步任务队列 |
| **代码点** | [ImportContactTask.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageTasks/Dav/ImportContactTask.php#L132-L144) 第138行 |
| **代码** | `'contact_id' => optional($task)->contact_id ?? $this->author()->getContactInVault($this->vault())->id` |
| **触发条件** | 待导入任务无对应 contact_task 记录（新任务场景） |
| **感知** | 队列 Job 失败，日志出现错误，用户侧感知为「日历同步不更新」 |

### 🔴 CRASH-11：CalDAV 重要日期同步（静默失败）

| 项目 | 内容 |
|-----|------|
| **入口** | 后台 CalDAV 同步任务队列 |
| **代码点** | [ImportCalendarContactImportantDates.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportCalendarContactImportantDates.php#L130-L144) 第136行 |
| **代码** | `'contact_id' => optional($importantDate)->contact_id ?? $this->author()->getContactInVault($this->vault())->id` |
| **感知** | 与 CRASH-10 相同，同步任务静默失败 |

---

## 三、🟡 异常级故障完整清单（7处）

### 🟡 EMPTY-01：权限系统整体放行（源头）

| 项目 | 内容 |
|-----|------|
| **影响范围** | Gate（3个）、BaseService、VaultHelper 三层 |
| **代码位置** | [AuthServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Providers/AuthServiceProvider.php) 的 Gate 定义 |
| **行为** | `$user->vaults()->wherePivot(...)->exists()` —— 只查 user_vault，不检测联系人软删除 |
| **后果** | 所有受权限保护的路由/服务全部放行，是后续一切崩溃/异常的前置条件 |

### 🟡 EMPTY-02：保险库列表残留

| 项目 | 内容 |
|-----|------|
| **入口** | 首页 Vault 列表 |
| **ViewHelper** | `VaultIndexViewHelper` 中 `$user->vaults()` |
| **行为** | user_vault 记录仍存在，已被移除的 Vault 仍显示在列表中 |
| **感知** | 用户仍能看到 Vault 卡片并点击 → 进入后触发 Dashboard 崩溃（CRASH-03/05） |

### 🟡 EMPTY-03：成员设置页显示虚假成员（管理者视角）

| 项目 | 内容 |
|-----|------|
| **入口** | 管理者访问 `vault.settings.index` |
| **ViewHelper** | `VaultSettingsIndexViewHelper` 中 `$vault->users()->get()` |
| **行为** | 被移除用户仍出现在成员列表中，权限等级也显示原值 |
| **感知** | 管理者以为该成员还在 Vault 中，再次操作移除会发生什么需实测 |

### 🟡 EMPTY-04：笔记作者信息空白

| 项目 | 内容 |
|-----|------|
| **入口** | 联系人详情笔记模块 / 笔记列表页 |
| **ViewHelper** | [ModuleNotesViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageNotes/Web/ViewHelpers/ModuleNotesViewHelper.php#L45-L72) 第57行 + [NotesIndexViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageNotes/Web/ViewHelpers/NotesIndexViewHelper.php#L47-L74) 第55行 |
| **代码** | `'author' => $note->author ? UserHelper::getInformationAboutContact($note->author, $contact->vault) : null` |
| **降级逻辑** | [UserHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Helpers/UserHelper.php#L21-L38) 第25行 `if (! $contact) { return null; }` |
| **感知** | 笔记卡片上作者头像/名字/链接区域为空，但笔记内容正常显示 |

### 🟡 EMPTY-05：Feed 动态作者信息降级

| 项目 | 内容 |
|-----|------|
| **入口** | 联系人详情 Feed 模块 |
| **ViewHelper** | [ModuleFeedViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L79-L105) |
| **两层降级** | ① `$item->author` 为 null → 返回内置 SVG「Deleted author」占位符；② author 存在但联系人软删 → `UserHelper` 返回 null |
| **感知** | 动态条目中作者区域显示空白或「已删除作者」图标，其余内容正常 |

### 🟡 EMPTY-06：日志帖子心情关联不显示

| 项目 | 内容 |
|-----|------|
| **入口** | 日志帖子详情页 `post.show` |
| **ViewHelper** | [PostShowViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/PostShowViewHelper.php#L160-L181) 第160-166行 |
| **代码** | `if (! $contact) { return null; }` —— 有显式判空 |
| **感知** | 帖子详情页中，帖子发布当天的心情记录块不渲染，但正文/标签/图片等内容全部正常 |

### 🟡 EMPTY-07：收藏列表少一条记录

| 项目 | 内容 |
|-----|------|
| **入口** | Dashboard 收藏模块 |
| **ViewHelper** | [VaultShowViewHelper.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L99-L116) |
| **代码** | `$user->contacts()->wherePivot('is_favorite', true)->get()` —— Eloquent 关联自动过滤软删除 |
| **感知** | 如果用户收藏了被移除成员的「本人联系人」，该联系人在收藏列表中消失，用户以为数据丢了 |

---

## 四、🟢 放行级故障完整清单（6类，无异常感知）

### 🟢 PASS-01：联系人列表 / 搜索

| 项目 | 内容 |
|-----|------|
| **入口** | `vault.contact.index` + 全局搜索 |
| **机制** | `$vault->contacts()` 走 vault_id 查询 + 软删除 Scope；Scout 索引已在 softDelete 时移除 |
| **感知** | 完全正常，联系人列表可正常浏览、搜索、分页 |

### 🟢 PASS-02：标签 / 分组 / 公司管理

| 项目 | 内容 |
|-----|------|
| **入口** | 联系人详情标签模块、Vault 分组管理、公司模块 |
| **机制** | 依赖 vault_id 外键关联，不涉及 getContactInVault |
| **感知** | 增删改查完全正常，**被移除用户甚至可以继续给联系人打标签、分分组** |

### 🟢 PASS-03：联系人详情大多数子模块

| 模块 | ViewHelper | 说明 |
|-----|-----------|------|
| 基本信息 | ModuleContactNameViewHelper 等 | 完全不依赖 getContactInVault |
| 笔记列表 | ModuleNotesViewHelper | 作者信息降级为 null（见 EMPTY-04），列表本身正常 |
| Feed 动态 | ModuleFeedViewHelper | 作者信息降级（见 EMPTY-05），列表本身正常 |
| 生活事件 | ModuleLifeEventViewHelper | 完全不依赖 getContactInVault |
| 标签 | ModuleLabelViewHelper | 完全正常 |
| 分组 | ModuleGroupsViewHelper | 完全正常 |
| 公司 | ModuleCompanyViewHelper | 完全正常 |
| 任务 | ModuleContactTasksViewHelper | 完全正常 |
| 文档 | ModuleDocumentsViewHelper | 完全正常 |
| 照片 | ModulePhotosViewHelper | 完全正常 |
| 宠物 | ModulePetsViewHelper | 完全正常 |
| 目标 | ModuleGoalsViewHelper | 完全正常 |

> **注意**：上述模块的正常渲染被 ContactShowViewHelper 的 CRASH-06 拦截了。修复 CRASH-06 后，这些模块全部可以正常工作。

### 🟢 PASS-04：报表首页 / 地址报表 / 日期摘要

| 报表 | ViewHelper | 依赖 |
|-----|-----------|------|
| 报表首页 | ReportIndexViewHelper | 仅生成 URL，不查数据 |
| 地址报表 | ReportAddressIndexViewHelper | 查 contact_addresses，不涉及 getContactInVault |
| 国家报表 | ReportCountriesShowViewHelper | 地址聚合查询 |
| 城市报表 | ReportCitiesShowViewHelper | 地址聚合查询 |
| 日期摘要 | ReportImportantDateSummaryIndexViewHelper | 查 contact_important_dates 聚合 |

> **注意**：情绪报表（ReportMoodTrackingEvent）是 CRASH-02，不属于此类。

### 🟢 PASS-05：CalDAV 推送过滤逻辑

| 项目 | 内容 |
|-----|------|
| **类** | [PrepareJobsContactPush.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/DavClient/Services/Utils/PrepareJobsContactPush.php#L103-L118) |
| **特殊实现** | 自有 `getContactInVault()` 方法直接从 pivot 表取 contact_id 字符串，**不经过 Contact 模型查询** |
| **机制** | `optional($pivot)->contact_id` 返回 UUID 字符串，与后端返回 UUID 做字符串比较 |
| **感知** | CalDAV 推送同步完全正常（虽然被移除成员的操作已不应该再推送到这个 Vault，但代码没拦截） |

### 🟢 PASS-06：提醒 / 任务 / 日志列表

| 项目 | 内容 |
|-----|------|
| **Dashboard 提醒** | `VaultShowViewHelper::upcomingReminders()`：查 reminder_channel_pivot → 关联 contact，软删除联系人的提醒会被 `if ($contact->vault_id != $vault->id) return null` 过滤掉（不崩溃但该提醒消失） |
| **Dashboard 待办** | `VaultShowViewHelper::dueTasks()`：通过 vault.contacts 关联查询，软删除联系人的任务自动排除，其余正常 |
| **日志列表** | 走 journals 表 + vault_id，不涉及 getContactInVault |

---

## 五、三类机制对应入口索引表

### 5.1 按用户操作路径索引

| 用户操作路径 | 故障类型 | 编号 |
|-------------|---------|------|
| 登录 → 查看首页 Vault 列表 | 🟡 EMPTY | EMPTY-02 |
| 点击已移除 Vault → 进入 Dashboard | 🔴 CRASH | CRASH-03/05/07 |
| Dashboard → 点击「Calendar」菜单 | 🔴 CRASH | CRASH-01 |
| Dashboard → 点击「Reports → Mood Tracking」 | 🔴 CRASH | CRASH-02 |
| Dashboard → 点击任意联系人 | 🔴 CRASH | CRASH-06 |
| 联系人详情 → 查看笔记 | 🟡 EMPTY | EMPTY-04（先过了 CRASH-06 才能看到） |
| 联系人详情 → 查看 Feed | 🟡 EMPTY | EMPTY-05（先过了 CRASH-06 才能看到） |
| 联系人详情 → 操作标签/分组 | 🟢 PASS | PASS-02（先过了 CRASH-06 才能操作） |
| 打开日志帖子详情 | 🟡 EMPTY | EMPTY-06 |
| 管理后台 → 成员设置 | 🟡 EMPTY | EMPTY-03 |
| 调用生活指标增量 API | 🔴 CRASH | CRASH-08 |
| CalDAV 后台同步（任务） | 🔴 CRASH | CRASH-10 |
| CalDAV 后台同步（日期） | 🔴 CRASH | CRASH-11 |

### 5.2 按调用 getContactInVault() 的代码位置统计

**全部调用点（21处）分类结果**：

| 调用结果处理方式 | 数量 | 对应故障 |
|----------------|-----|---------|
| 直接 `->id` 或 `->method()` 链式调用 | 13 | 🔴 CRASH |
| `UserHelper` 内部有判空 | 3 | 🟡 EMPTY |
| 显式 `if (!$contact) return null` | 2 | 🟡 EMPTY |
| `optional()` 包装 / 直接返回 pivot 字符串 | 3 | 🟢 PASS |

---

## 六、修复优先级与建议方案

### P0（阻断性，必须立即修复）

1. **修复权限判断源头**：在 Gate / BaseService / VaultHelper 三层中加入软删除检查，确保 user_vault 存在但联系人已软删时返回 false。从根源阻止放行。

2. **所有链式调用加判空**：覆盖 CRASH-01 ~ CRASH-11 的 11 处崩溃点，统一使用 `optional()` 或显式 `if (!$contact)` 保护。

3. **修复 RemoveVaultAccess 软删除逻辑**：改用 `forceDelete()` 硬删除，或在软删除后手动删除 `user_vault` 关联记录，触发数据库 CASCADE 清理。

### P1（体验性，第二优先级）

4. **修复列表显示异常**：VaultIndexViewHelper 和 VaultSettingsIndexViewHelper 中加入条件过滤已软删联系人的成员记录。

5. **提醒清理范围修正**：`removeAllRemindersForThisUserInThisVault()` 方法名与实际行为不符（清了所有 Vault 的提醒），需要加 vault_id 限定。

### P2（结构性，长期优化）

6. **统一数据源设计**：权限判断与联系人映射使用同一数据源（比如以 user_vault 为准，删除 user_vault 记录即视为移除，不依赖软删除级联）。

7. **单测覆盖**：为 `getContactInVault()` 返回 null 的场景为每个涉及的 Controller / ViewHelper 编写单元测试。

8. **事务包裹**：RemoveVaultAccess 服务加入 DB::transaction()，确保软删除与提醒清理等步骤原子化。
