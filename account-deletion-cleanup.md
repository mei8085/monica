# 账号删除数据清理流程梳理

## 概述

Monica 系统中有两种删除场景：

1. **删除单个用户** — 管理员删除账户下的某个用户
2. **注销整个账户** — 管理员注销整个账户，所有数据全部清除

两种场景的数据清理都依赖**数据库外键级联删除** + **手动清理特殊资源**的组合方式。

---

## 一、删除单个用户（DestroyUser）

### 1.1 触发入口

- Jetstream 动作入口：[DeleteUser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Actions/Jetstream/DeleteUser.php)
- 业务服务类：[DestroyUser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Domains/Settings/ManageUsers/Services/DestroyUser.php)

### 1.2 删除流程

```
权限校验 → 不能删除自己 → 删除用户作为唯一管理员的 Vault → 删除用户记录
```

#### 步骤详解：

1. **权限校验**
   - 操作者必须属于该账户
   - 操作者必须是账户管理员 (`author_must_be_account_administrator`)

2. **防止自删**
   - 不能删除当前登录用户自己

3. **清理独占 Vault**
   - 找出用户拥有 `MANAGE` 权限的所有 Vault
   - 对每个 Vault 检查：如果没有其他管理员，则删除该 Vault
   - Vault 删除会级联清理其下所有数据（详见下文「Vault 级联删除数据」）

4. **删除用户记录**
   - 调用 `$user->delete()`
   - 通过数据库外键级联删除所有关联数据

### 1.3 删除用户时级联清理的数据

通过 `user_id` 外键级联删除的表：

| 数据表 | 说明 | 关联关系 |
|--------|------|----------|
| `user_tokens` | 第三方登录令牌 | 一对多 |
| `personal_access_tokens` | API 访问令牌 (Sanctum) | 一对多 |
| `webauthn_keys` | WebAuthn 密钥 | 一对多 |
| `user_vault` | 用户-Vault 关联表 | 多对多中间表 |
| `contact_vault_user` | 联系人-用户收藏/浏览记录 | 多对多中间表 |
| `address_book_subscriptions` | 地址簿订阅 | 一对多 |
| `sync_tokens` | 同步令牌 | 一对多 |

**注意**：`user_notification_channels` 表的 `user_id` 字段**没有外键约束**（仅 `nullable()` 无 `constrained()`），删除用户时不会自动级联删除，会留下悬空引用。详见附录 A.4.3。

另外，用户作为作者的记录（通过 `author_id` 关联）：
- `notes` — 笔记（`author_id` 置空，记录保留）
- `contact_tasks` — 联系人任务（`author_id` 置空，记录保留）
- `calls` — 通话记录（`author_id` 置空，记录保留）
- `contact_feed_items` — 联系人动态（`author_id` 置空，记录保留）

**注意**：所有 `author_id` 字段均使用 `nullOnDelete` 策略，删除用户不会导致这些记录被删除，仅作者关联被清空。详见下文「附录：author_id 字段级联规则分析」。

---

## 二、注销整个账户（CancelAccount）

### 2.1 触发入口

- 控制器：[CancelAccountController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Domains/Settings/CancelAccount/Web/Controllers/CancelAccountController.php)
- 业务服务类：[CancelAccount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Domains/Settings/CancelAccount/Services/CancelAccount.php)

### 2.2 删除流程

```
权限校验 → 手动删除所有文件（触发存储清理） → 删除账户（级联清理所有数据）
```

#### 步骤详解：

1. **权限校验**
   - 操作者必须属于该账户
   - 操作者必须是账户管理员

2. **手动删除所有文件（关键特殊处理）**
   - 遍历账户下所有 Vault 的所有文件
   - 逐个调用 `$file->delete()`
   - **目的**：触发 `FileDeleted` 事件，由监听器从 Uploadcare 云存储中删除实际文件

3. **删除账户记录**
   - 调用 `$account->delete()`
   - 通过数据库外键级联删除所有关联数据

### 2.3 为什么文件需要手动删除？

File 模型配置了 `deleted` 事件分发：
```php
// File.php
protected $dispatchesEvents = [
    'deleted' => FileDeleted::class,
];
```

事件监听器 [DeleteFileInStorage.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Domains/Contact/ManageDocuments/Listeners/DeleteFileInStorage.php) 会调用 Uploadcare API 删除云端文件。

**如果直接通过数据库级联删除 files 表记录，不会触发 Eloquent 模型事件，导致云端文件残留。** 因此 CancelAccount 和 DestroyVault 都选择了遍历删除文件的方式。

---

## 三、Account 级联删除数据全景

当 `Account` 被删除时，通过 `account_id` 外键的 `cascadeOnDelete` 级联清理以下数据：

### 3.1 账户级配置数据

| 数据表 | 模型 | 说明 |
|--------|------|------|
| `users` | [User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Models/User.php) | 所有用户（进一步级联用户相关数据） |
| `templates` | [Template.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Models/Template.php) | 联系人模板 |
| `template_pages` | - | 模板页面（通过模板级联） |
| `modules` | - | 模块（通过模板级联） |
| `module_rows` | - | 模块行 |
| `module_row_fields` | - | 模块行字段 |
| `genders` | - | 性别选项 |
| `pronouns` | - | 称谓选项 |
| `contact_information_types` | - | 联系信息类型 |
| `address_types` | - | 地址类型 |
| `pet_categories` | - | 宠物分类 |
| `emotions` | - | 情绪选项 |
| `call_reason_types` | - | 通话原因类型 |
| `call_reasons` | - | 通话原因（通过类型级联） |
| `gift_occasions` | - | 礼物场合 |
| `gift_states` | - | 礼物状态 |
| `religions` | - | 宗教 |
| `group_types` | - | 分组类型 |
| `group_type_roles` | - | 分组类型角色 |
| `relationship_group_types` | - | 关系组类型 |
| `relationship_types` | - | 关系类型（通过组类型级联） |
| `post_templates` | - | 日志帖子模板 |
| `post_template_sections` | - | 帖子模板分区 |
| `sync_tokens` | - | 同步令牌 |
| `account_currency` | - | 账户-货币多对多关联表 |

### 3.2 Vault 数据（进一步级联）

| 数据表 | 模型 | 说明 |
|--------|------|------|
| `vaults` | [Vault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Models/Vault.php) | 所有 Vault → 进一步级联 Vault 下所有数据 |

---

## 四、Vault 级联删除数据全景

当 `Vault` 被删除时，通过 `vault_id` 外键的 `cascadeOnDelete` 级联清理以下数据：

### 4.1 联系人核心数据

| 数据表 | 模型 | 说明 |
|--------|------|------|
| `contacts` | [Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Models/Contact.php) | 所有联系人 → 进一步级联联系人相关数据 |
| `contact_information` | - | 联系信息（电话、邮箱等） |
| `contact_important_dates` | - | 重要日期（生日、纪念日等） |
| `contact_reminders` | - | 提醒 |
| `contact_reminder_scheduled` | - | 已计划的提醒 |
| `contact_tasks` | - | 任务 |
| `calls` | - | 通话记录 |
| `call_reasons` | - | 通话原因 |
| `notes` | - | 笔记 |
| `pets` | - | 宠物 |
| `goals` | - | 目标 |
| `mood_tracking_events` | - | 心情追踪记录 |
| `quick_facts` | - | 快速信息 |
| `contact_feed_items` | - | 联系人动态 |
| `relationships` | - | 联系人关系 |
| `loans` | - | 借贷记录 |
| `gifts` | - | 礼物记录 |

### 4.2 组织与分类数据

| 数据表 | 说明 |
|--------|------|
| `labels` | 标签 |
| `contact_label` | 联系人-标签关联 |
| `groups` | 分组 |
| `contact_group` | 联系人-分组关联 |
| `tags` | 标签（帖子用） |
| `post_tag` | 帖子-标签关联 |
| `companies` | 公司 |
| `addresses` | 地址 |
| `contact_address` | 联系人-地址关联 |

### 4.3 日志与时间线

| 数据表 | 模型 | 说明 |
|--------|------|------|
| `journals` | [Journal.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Models/Journal.php) | 日志 |
| `posts` | - | 日志帖子 |
| `post_sections` | - | 帖子分区 |
| `post_metrics` | - | 帖子指标 |
| `journal_metrics` | - | 日志指标 |
| `slices_of_life` | - | 生活片段 |
| `timeline_events` | - | 时间线事件 |
| `timeline_event_participants` | - | 时间线事件参与者 |
| `life_events` | - | 生活事件 |
| `life_event_categories` | - | 生活事件分类 |
| `life_event_types` | - | 生活事件类型 |
| `life_event_participants` | - | 生活事件参与者 |

### 4.4 文件与媒体

| 数据表 | 模型 | 说明 |
|--------|------|------|
| `files` | [File.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Models/File.php) | 文件记录 |
| `mood_tracking_parameters` | - | 心情追踪参数 |
| `life_metrics` | - | 生活指标 |
| `contact_life_metric` | - | 联系人-生活指标关联 |
| `vault_quick_facts_templates` | - | Vault 快速事实模板 |
| `contact_important_date_types` | - | 重要日期类型 |

### 4.5 关联表

| 数据表 | 说明 |
|--------|------|
| `user_vault` | 用户-Vault 权限关联 |
| `contact_vault_user` | 联系人-用户收藏/浏览 |

---

## 五、特殊清理机制

### 5.1 搜索索引清理

**Vault 删除时**：
- [Vault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Models/Vault.php#L76-L81) 中注册了 `deleting` 事件
- 会将所有联系人的笔记从 Scout 搜索索引中移除
- 将所有联系人从 Scout 搜索索引中移除

**Contact 删除时**：
- [Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Models/Contact.php#L124-L126) 中注册了 `deleting` 事件
- 会将该联系人的笔记从 Scout 搜索索引中移除

### 5.2 云存储文件清理

- File 模型删除时触发 `FileDeleted` 事件
- [DeleteFileInStorage.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Domains/Contact/ManageDocuments/Listeners/DeleteFileInStorage.php) 监听器调用 Uploadcare API 删除云端文件
- **重要**：只有通过 Eloquent 的 `delete()` 方法删除才会触发事件，直接 SQL 删除或数据库级联删除不会触发

### 5.3 软删除注意事项

Contact 模型使用了 `SoftDeletes` trait，但在级联删除场景下，数据库的 `cascadeOnDelete` 是物理删除，会直接删除记录而非软删除。

---

## 六、删除调用链总览

### 6.1 注销账户完整调用链

```
CancelAccountController
  → CancelAccount::execute()
      → destroyAllFiles()  // 遍历删除所有文件（触发 FileDeleted → 删 Uploadcare 文件）
      → $account->delete() // 账户删除
          → 级联删除 users
              → 级联删除 user_vault, user_notification_channels 等
          → 级联删除 vaults
              → 删除前触发 Vault::deleting（清 Scout 索引）
              → 级联删除 contacts, files, journals, ...
                  → contacts 删除触发 Contact::deleting（清笔记索引）
                  → contacts 级联删除 notes, reminders, tasks, ...
          → 级联删除 templates, modules, group_types, ...
```

### 6.2 删除单个用户完整调用链

```
DeleteUser (Jetstream)
  → DestroyUser::execute()
      → validate()  // 权限校验、防自删
      → destroyAllVaults()  // 删除用户作为唯一管理员的 Vault
          → 每个 Vault 的删除：调用 $vault->delete()
              → 触发 Vault::deleting（清 Scout 索引）
              → 级联删除该 Vault 下所有数据（同 6.1 中的 Vault 级联）
      → destroy()  // $user->delete()
          → 级联删除 user_vault, user_notification_channels 等
```

---

## 七、关键代码位置汇总

| 功能 | 文件路径 |
|------|----------|
| Jetstream 删除用户入口 | [DeleteUser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Actions/Jetstream/DeleteUser.php) |
| 删除用户服务 | [DestroyUser.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Domains/Settings/ManageUsers/Services/DestroyUser.php) |
| 注销账户服务 | [CancelAccount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Domains/Settings/CancelAccount/Services/CancelAccount.php) |
| 删除 Vault 服务 | [DestroyVault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Domains/Vault/ManageVault/Services/DestroyVault.php) |
| 文件删除事件 | [FileDeleted.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Domains/Contact/ManageDocuments/Events/FileDeleted.php) |
| 云存储文件删除监听器 | [DeleteFileInStorage.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Domains/Contact/ManageDocuments/Listeners/DeleteFileInStorage.php) |
| Vault 模型（删除事件） | [Vault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Models/Vault.php) |
| Contact 模型（删除事件） | [Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Models/Contact.php) |
| Account 模型 | [Account.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Models/Account.php) |
| User 模型 | [User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/app/Models/User.php) |

---

## 八、总结

1. **数据删除主要依赖数据库外键级联** — 大部分数据通过 `cascadeOnDelete` 自动清理
2. **文件需要手动遍历删除** — 为了触发 Eloquent 事件从而清理 Uploadcare 云存储
3. **搜索索引在模型 deleting 事件中清理** — Vault 和 Contact 删除时会移除 Scout 索引
4. **删除用户不等于删除其所有数据** — 只有当用户是某个 Vault 的唯一管理员时，该 Vault 才会被删除；否则 Vault 保留，用户仅被从 Vault 成员中移除
5. **注销账户是最彻底的删除** — 账户下所有数据（用户、Vault、联系人、文件等）全部级联清除
6. **用户创建的内容不会被删除** — 笔记、任务、通话、动态等 `author_id` 关联的内容仅置空作者，记录本身保留
7. **未使用 restrictOnDelete** — 全系统无任何 `restrictOnDelete`，删除操作不会因存在关联数据而被阻止

---

## 九、外键级联策略全景表

本章节逐张表分析所有迁移文件中的外键级联策略，共统计到 **100 处 cascadeOnDelete**、**28 处 nullOnDelete**、**0 处 restrictOnDelete**。

### 9.1 三种外键策略使用统计

| 策略 | 数量 | 占比 | 说明 |
|------|------|------|------|
| `cascadeOnDelete()` | 100 | 78.1% | 关联记录随父记录一同删除 |
| `nullOnDelete()` | 28 | 21.9% | 外键字段置为 NULL，关联记录保留 |
| `restrictOnDelete()` | 0 | 0% | 全系统未使用，不会阻止删除 |
| **合计** | **128** | **100%** | |

### 9.2 按删除场景梳理策略

#### 场景一：注销账户（Account 删除）→ cascadeOnDelete

##### 第一层：Account 直接关联的表

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `users` | `account_id` | cascade | [create_users_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2014_10_12_000000_create_users_table.php#L19) |
| `vaults` | `account_id` | cascade | [create_vaults_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2014_10_12_000010_create_vaults_table.php#L19) |
| `templates` | `account_id` | cascade | [create_attributes_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2013_04_25_155842_create_attributes_table.php#L21) |
| `modules` | `account_id` | cascade | [create_attributes_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2013_04_25_155842_create_attributes_table.php#L41) |
| `genders` | `account_id` | cascade | [create_genders_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_02_17_224235_create_genders_table.php#L17) |
| `pronouns` | `account_id` | cascade | [create_pronouns_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_02_19_173445_create_pronouns_table.php#L17) |
| `contact_information_types` | `account_id` | cascade | [create_contact_fields_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_20_004100_create_contact_fields_table.php#L19) |
| `address_types` | `account_id` | cascade | [create_address_types_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_03_20_213318_create_address_types_table.php#L17) |
| `pet_categories` | `account_id` | cascade | [create_pets_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_18_000002_create_pets_table.php#L19) |
| `emotions` | `account_id` | cascade | [create_emotions_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_20_163535_create_emotions_table.php#L19) |
| `call_reason_types` | `account_id` | cascade | [create_call_reasons_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_16_184121_create_call_reasons_table.php#L20) |
| `gift_occasions` | `account_id` | cascade | [create_gifts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_06_09_173049_create_gifts_table.php#L22) |
| `gift_states` | `account_id` | cascade | [create_gifts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_06_09_173049_create_gifts_table.php#L31) |
| `religions` | `account_id` | cascade | [create_religions_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_10_30_202904_create_religions_table.php#L19) |
| `group_types` | `account_id` | cascade | [create_group_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_09_204235_create_group_table.php#L25) |
| `relationship_group_types` | `account_id` | cascade | [create_relationship_types_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_16_184625_create_relationship_types_table.php#L20) |
| `post_templates` | `account_id` | cascade | [create_posts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_09_22_111510_create_posts_table.php#L25) |
| `sync_tokens` | `account_id` | cascade | [create_synctokens.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2018_12_29_135516_create_synctokens.php#L20) |
| `account_currency` | `account_id` | cascade | [create_currencies_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_03_22_180407_create_currencies_table.php#L28) |
| `contact_information` | `account_id` | cascade | [create_contact_fields_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_20_004100_create_contact_fields_table.php#L19) |

##### 第二层：模板级联（templates 删除）→ cascadeOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `template_pages` | `template_id` | cascade | [create_attributes_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2013_04_25_155842_create_attributes_table.php#L29) |
| `module_template_page` | `template_page_id` | cascade | [create_attributes_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2013_04_25_155842_create_attributes_table.php#L52-L53) |
| `module_template_page` | `module_id` | cascade | 同上（双边级联） |

##### 第三层：模块级联（modules 删除）→ cascadeOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `module_rows` | `module_id` | cascade | [create_attributes_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2013_04_25_155842_create_attributes_table.php#L60) |
| `module_row_fields` | `module_row_id` | cascade | [create_attributes_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2013_04_25_155842_create_attributes_table.php#L67) |

##### 第四层：分组类型级联（group_types 删除）→ cascadeOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `groups` | `group_type_id` | cascade | [create_group_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_09_204235_create_group_table.php#L34) |
| `call_reasons` | `call_reason_type_id` | cascade | [create_call_reasons_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_16_184121_create_call_reasons_table.php#L28) |
| `relationship_types` | `relationship_group_type_id` | cascade | [create_relationship_types_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_16_184625_create_relationship_types_table.php#L30) |
| `pet_categories` 的关联表 | `pet_category_id` | cascade | [create_pets_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_18_000002_create_pets_table.php#L29) |
| `post_template_sections` | `post_template_id` | cascade | [create_posts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_09_22_111510_create_posts_table.php#L35) |

##### 第五层：联系人信息类型级联 → cascadeOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `contact_information` | `type_id` | cascade | [create_contact_fields_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_20_004100_create_contact_fields_table.php#L31) |

##### 第六层：货币关联级联 → cascadeOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `account_currency` | `currency_id` | cascade | [create_currencies_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_03_22_180407_create_currencies_table.php#L27) |

#### 场景二：Vault 删除 → cascadeOnDelete + nullOnDelete

##### 第一层：Vault 直接关联的表（全部 cascadeOnDelete）

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `contacts` | `vault_id` | cascade | [create_contacts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L25) |
| `companies` | `vault_id` | cascade | [create_companies_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_23_133132_create_companies_table.php#L17) |
| `groups` | `vault_id` | cascade | [create_group_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_09_204235_create_group_table.php#L44) |
| `labels` | `vault_id` | cascade | [create_labels_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_19_192432_create_labels_table.php#L19) |
| `tags` | `vault_id` | cascade | [create_posts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_09_22_111510_create_posts_table.php#L70) |
| `journals` | `vault_id` | cascade | [create_journal_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_09_20_183401_create_journal_table.php#L19) |
| `files` | `vault_id` | cascade | [create_files_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_02_24_002342_create_files_table.php#L20) |
| `loans` | `vault_id` | cascade | [create_loans_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_03_23_005751_create_loans_table.php#L22) |
| `addresses` | `vault_id` | cascade | [create_addresses_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_26_215133_create_addresses_table.php#L20) |
| `notes` | `vault_id` | cascade | [create_notes_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_21_013005_create_notes_table.php#L22) |
| `contact_important_date_types` | `vault_id` | cascade | [create_contact_date_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_02_09_145139_create_contact_date_table.php#L21) |
| `mood_tracking_parameters` | `vault_id` | cascade | [create_mood_tracking_parameters_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2023_01_07_005110_create_mood_tracking_parameters_table.php#L19) |
| `life_event_categories` | `vault_id` | cascade | [create_life_events_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_17_155546_create_life_events_table.php#L26) |
| `life_event_types` | `vault_id` | cascade | [create_life_events_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_17_155546_create_life_events_table.php#L46) |
| `timeline_events` | `vault_id` | cascade | [create_life_events_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_17_155546_create_life_events_table.php#L54) |
| `user_vault` | `vault_id` | cascade | [create_contacts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L62) |
| `contact_vault_user` | `vault_id` | cascade | [create_contacts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L71) |
| `address_book_subscriptions` | `vault_id` | cascade | [create_addressbook_subscription.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2023_07_03_230200_create_addressbook_subscription.php#L22) |
| `life_metrics` | `vault_id` | cascade | [create_life_metrics_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2023_03_31_125903_create_life_metrics_table.php#L19) |
| `vault_quick_facts_templates` | `vault_id` | cascade | [create_vault_quick_facts_template_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2023_02_07_022607_create_vault_quick_facts_template_table.php#L21) |
| `posts` | `vault_id` | cascade | [create_posts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_09_22_111510_create_posts_table.php#L70) |

##### Vault 关联表的 nullOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 | 说明 |
|------|---------|------|---------|------|
| `vaults` | `default_template_id` | null | [create_vaults_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2014_10_12_000010_create_vaults_table.php#L23) | 默认模板被删时置空 |
| `address_book_subscriptions` | `sync_token_id` | null | [create_addressbook_subscription.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2023_07_03_230200_create_addressbook_subscription.php#L33) | 同步令牌被删时置空 |

##### 第二层：Contacts 级联（contacts 删除）→ cascadeOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `contact_information` | `contact_id` | cascade | [create_contact_fields_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_20_004100_create_contact_fields_table.php#L30) |
| `contact_important_dates` | `contact_id` | cascade | [create_contact_date_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_02_09_145139_create_contact_date_table.php#L30) |
| `contact_reminders` | `contact_id` | cascade | [create_reminders_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_02_18_215852_create_reminders_table.php#L22) |
| `contact_tasks` | `contact_id` | cascade | [create_contact_tasks_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_13_201216_create_contact_tasks_table.php#L20) |
| `calls` | `contact_id` | cascade | [create_calls_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_16_193917_create_calls_table.php#L22) |
| `notes` | `contact_id` | cascade | [create_notes_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_21_013005_create_notes_table.php#L21) |
| `pets` | `contact_id` | cascade | [create_pets_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_18_000002_create_pets_table.php#L28) |
| `goals` | `contact_id` | cascade | [create_goals_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_06_02_011219_create_goals_table.php#L20) |
| `streaks` | `goal_id` | cascade | [create_goals_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_06_02_011219_create_goals_table.php#L28) |
| `mood_tracking_events` | `contact_id` | cascade | [create_mood_tracking_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2023_01_08_155554_create_mood_tracking_table.php#L20) |
| `contact_feed_items` | `contact_id` | cascade | [create_contact_feed_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_19_022411_create_contact_feed_table.php#L19) |
| `quick_facts` | `contact_id` | cascade | [create_vault_quick_facts_template_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2023_02_07_022607_create_vault_quick_facts_template_table.php#L31) |
| `user_vault` | `contact_id` | cascade | [create_contacts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L64) |
| `contact_vault_user` | `contact_id` | cascade | [create_contacts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L70) |
| `relationships` | `contact_id` + `related_contact_id` | cascade（双边） | [create_relationship_types_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_16_184625_create_relationship_types_table.php#L42-L44) |
| `loans` | `loaner_id` + `loanee_id` | cascade（双边） | [create_loans_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_03_23_005751_create_loans_table.php#L36-L37) |
| `gifts` | `loaner_id` + `loanee_id` | cascade（双边） | [create_gifts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_06_09_173049_create_gifts_table.php#L54-L55) |
| `contact_label` | `contact_id` + `label_id` | cascade（双边） | [create_labels_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_19_192432_create_labels_table.php#L29-L30) |
| `contact_group` | `contact_id` + `group_id` | cascade（双边） | [create_group_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_09_204235_create_group_table.php#L62-L63) |
| `contact_address` | `contact_id` + `address_id` | cascade（双边） | [create_addresses_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_26_215133_create_addresses_table.php#L34-L35) |
| `contact_post` | `contact_id` + `post_id` | cascade（双边） | [create_posts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_09_22_111510_create_posts_table.php#L63-L64) |
| `life_event_participants` | `contact_id` + `life_event_id` | cascade（双边） | [create_life_events_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_17_155546_create_life_events_table.php#L81-L82) |
| `timeline_event_participants` | `contact_id` + `timeline_event_id` | cascade（双边） | [create_life_events_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_17_155546_create_life_events_table.php#L54-L55) |
| `timeline_event_types` 关联 | `timeline_event_id` + `life_event_type_id` | cascade（双边） | [create_life_events_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_17_155546_create_life_events_table.php#L61-L62) |
| `contact_life_metric` | `contact_id` + `life_metric_id` | cascade（双边） | [create_life_metrics_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2023_03_31_125903_create_life_metrics_table.php#L25-L26) |
| `gifts` | `contact_id` | cascade | [create_gifts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_06_09_173049_create_gifts_table.php#L40) |
| `gifts` | `loan_id` | cascade | [create_gifts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_06_09_173049_create_gifts_table.php#L53) |
| `contact_loan` | `loan_id` + `loaner_id` + `loanee_id` | cascade（三边） | [create_loans_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_03_23_005751_create_loans_table.php#L35-L37) |

##### Contacts 关联表的 nullOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 | 说明 |
|------|---------|------|---------|------|
| `contacts` | `gender_id` | null | [create_contacts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L26) | 性别被删时置空 |
| `contacts` | `pronoun_id` | null | [create_contacts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L27) | 称谓被删时置空 |
| `contacts` | `template_id` | null | [create_contacts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L28) | 模板被删时置空 |
| `contacts` | `company_id` | null | [create_contacts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L29) | 公司被删时置空 |
| `contacts` | `file_id`（avatar） | null | [create_files_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_02_24_002342_create_files_table.php#L37) | 头像文件被删时置空 |
| `contacts` | `religion_id` | null | [add_religion_to_contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_11_01_174411_add_religion_to_contact.php#L18) | 宗教被删时置空 |
| `contact_important_dates` | `contact_important_date_type_id` | null | [create_contact_date_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_02_09_145139_create_contact_date_table.php#L31) | 日期类型被删时置空 |
| `notes` | `emotion_id` | null | [create_notes_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_21_013005_create_notes_table.php#L24) | 情绪被删时置空 |
| `calls` | `emotion_id` | null | [create_calls_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_16_193917_create_calls_table.php#L25) | 情绪被删时置空 |
| `calls` | `call_reason_id` | cascade（注意） | [create_calls_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_16_193917_create_calls_table.php#L23) | 通话原因被删时级联删通话！ |
| `life_events` | `emotion_id` | null | [create_life_events_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_17_155546_create_life_events_table.php#L63) | 情绪被删时置空 |
| `life_events` | `currency_id` | null | [create_life_events_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_17_155546_create_life_events_table.php#L69) | 货币被删时置空 |
| `life_events` | `paid_by_contact_id` | null | [create_life_events_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_17_155546_create_life_events_table.php#L70) | 付款联系人被删时置空 |
| `loans` | `currency_id` | null | [create_loans_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_03_23_005751_create_loans_table.php#L27) | 货币被删时置空 |
| `gifts` | `currency_id` | null | [create_gifts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_06_09_173049_create_gifts_table.php#L45) | 货币被删时置空 |
| `addresses` | `address_type_id` | null | [create_addresses_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_26_215133_create_addresses_table.php#L21) | 地址类型被删时置空 |
| `groups` | `group_type_id` | null | [create_group_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_09_204235_create_group_table.php#L45) | 分组类型被删时置空 |
| `groups` | `group_type_role_id` | null | [create_group_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_09_204235_create_group_table.php#L64) | 分组角色被删时置空 |

##### 第三层：Journals 级联（journals 删除）→ cascadeOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `posts` | `journal_id` | cascade | [create_posts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_09_22_111510_create_posts_table.php#L45) |
| `journal_metrics` | `journal_id` | cascade | [create_post_metrics_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2023_03_16_182310_create_post_metrics_table.php#L19) |
| `slices_of_life` | `journal_id` | cascade | [create_slices_of_life_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_12_15_004442_create_slices_of_life_table.php#L21) |

##### Journals 关联的 nullOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 | 说明 |
|------|---------|------|---------|------|
| `slices_of_life` | `file_cover_image_id` | null | [create_slices_of_life_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_12_15_004442_create_slices_of_life_table.php#L22) | 封面文件被删时置空 |
| `slices_of_life` | `slice_of_life_id`（父切片） | null | [create_slices_of_life_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_12_15_004442_create_slices_of_life_table.php#L29) | 父切片被删时置空 |

##### 第四层：Posts 级联（posts 删除）→ cascadeOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `post_sections` | `post_id` | cascade | [create_posts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_09_22_111510_create_posts_table.php#L55) |
| `post_metrics` | `post_id` | cascade | [create_post_metrics_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2023_03_16_182310_create_post_metrics_table.php#L29) |
| `post_tag` | `post_id` + `tag_id` | cascade（双边） | [create_posts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_09_22_111510_create_posts_table.php#L77-L78) |
| `post_template_sections` | `post_template_id` | cascade | [create_posts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_09_22_111510_create_posts_table.php#L35) |

##### Post 级联的特殊 nullOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 | 说明 |
|------|---------|------|---------|------|
| `posts`（注意是 post_templates 表）| `post_template_id` | cascade | [create_posts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_09_22_111510_create_posts_table.php#L35) | **注意**：帖子模板被删时级联删除所有使用该模板的帖子（与联系人模板策略不同！） |

##### 第五层：Life Events 级联 → cascadeOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `life_event_types` | `life_event_category_id` | cascade | [create_life_events_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_17_155546_create_life_events_table.php#L36) |
| `life_events` | `life_event_category_id` | —（通过 type 间接） | — | 分类被删先删类型，再删事件 |
| `life_events` | `life_event_type_id` | —（通过 timeline 间接） | — | 类型被删时通过中间表级联 |

##### 第六层：Mood Tracking 级联 → cascadeOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `mood_tracking_events` | `mood_tracking_parameter_id` | cascade | [create_mood_tracking_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2023_01_08_155554_create_mood_tracking_table.php#L21) |

##### 第七层：User Notification Channels 级联 → cascadeOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `contact_reminder_scheduled` | `user_notification_channel_id` + `contact_reminder_id` | cascade（双边） | [create_reminders_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_02_18_215852_create_reminders_table.php#L49-L50) |
| `user_notification_sent` | `user_notification_channel_id` | cascade | [create_reminders_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_02_18_215852_create_reminders_table.php#L58) |

#### 场景三：User 删除 → cascadeOnDelete + nullOnDelete

##### 第一层：User 直接关联 cascadeOnDelete

| 表名 | 外键字段 | 策略 | 迁移文件 |
|------|---------|------|---------|
| `user_tokens` | `user_id` | cascade | [create_user_token_socialite.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_07_06_065356_create_user_token_socialite.php#L17) |
| `webauthn_keys` | `user_id` | cascade | [create_webauthn_keys.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2019_03_29_163611_create_webauthn_keys.php#L19) |
| `sync_tokens` | `user_id` | cascade | [create_synctokens.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2018_12_29_135516_create_synctokens.php#L21) |
| `user_vault` | `user_id` | cascade | [create_contacts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L63) |
| `contact_vault_user` | `user_id` | cascade | [create_contacts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L72) |
| `address_book_subscriptions` | `user_id` | cascade | [create_addressbook_subscription.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2023_07_03_230200_create_addressbook_subscription.php#L21) |

##### 第二层：User 直接关联 nullOnDelete（author_id 等）

| 表名 | 外键字段 | 策略 | 迁移文件 | 说明 |
|------|---------|------|---------|------|
| `notes` | `author_id` | null | [create_notes_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_21_013005_create_notes_table.php#L23) | 作者被删时置空，笔记保留 |
| `contact_tasks` | `author_id` | null | [create_contact_tasks_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_13_201216_create_contact_tasks_table.php#L21) | 作者被删时置空，任务保留 |
| `calls` | `author_id` | null | [create_calls_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_05_16_193917_create_calls_table.php#L24) | 作者被删时置空，通话保留 |
| `contact_feed_items` | `author_id` | null | [create_contact_feed_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2021_10_19_022411_create_contact_feed_table.php#L18) | 作者被删时置空，动态保留 |

##### 第三层：无外键约束的 User 关联（潜在悬空引用）

| 表名 | 外键字段 | 字段定义 | 实际行为 | 迁移文件 |
|------|---------|---------|---------|---------|
| `sessions` | `user_id` | `nullable()->index()`（无 constrained） | 记录残留，无数据库处理 | [create_sessions_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_07_31_200647_create_sessions_table.php#L19) |
| `user_notification_channels` | `user_id` | `nullable()`（无 constrained） | 记录残留，无数据库处理 | [create_reminders_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_02_18_215852_create_reminders_table.php#L36) |
| `contact_reminders` | `user_id` | `nullable()`（无 constrained） | 记录残留，无数据库处理 | [create_reminders_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/67-monica/database/migrations/2022_02_18_215852_create_reminders_table.php#L36) |

### 9.3 关键策略差异对比

| 数据类型 | 被删对象 | 策略选择 | 设计意图 |
|---------|---------|---------|---------|
| **用户身份数据** | User | cascadeOnDelete | 令牌、密钥等必须彻底清除 |
| **用户创建的内容** | User（author_id） | nullOnDelete | 笔记/任务/通话有历史价值，保留 |
| **联系人核心数据** | Contact | cascadeOnDelete | 联系人被删则一切从属数据清除 |
| **联系人分类属性** | Gender/Pronoun/Religion | nullOnDelete | 属性被删不影响联系人存在 |
| **Vault 配置数据** | Vault | cascadeOnDelete | Vault 被删则一切数据清除 |
| **Vault 默认模板** | Template | nullOnDelete | 模板被删仅取消默认设置 |
| **帖子模板** | PostTemplate | cascadeOnDelete | **特殊**：模板被删则所有使用该模板的帖子级联删除 |
| **联系人模板** | Template | nullOnDelete | 模板被删仅取消联系人的模板关联 |
| **多对多关联表** | 双边 | 双边 cascadeOnDelete | 任何一方被删则解除关联 |
| **货币关联** | Currency | nullOnDelete | 货币被删不删除借贷/礼物记录 |
| **通话原因** | CallReason | cascadeOnDelete | **特殊**：通话原因被删则级联删除通话记录 |

---

## 十、账号删除完整数据流向图

### 10.1 注销账户（Account 删除）完整流程

```
CancelAccount::execute()
│
├─ destroyAllFiles()  手动遍历删除（触发 FileDeleted → 删 Uploadcare 云端文件）
│   └─ File::chunk(100) → 逐个 $file->delete()
│       └─ FileDeleted 事件 → DeleteFileInStorage 监听器 → Uploadcare API 删除
│
└─ $account->delete()
    │
    ├─ Account 级联 cascadeOnDelete
    │   │
    │   ├─ users（所有用户）
    │   │   └─ 见 10.3 User 删除级联
    │   │
    │   ├─ vaults（所有 Vault）
    │   │   ├─ 触发 Vault::deleting 事件
    │   │   │   ├─ 所有联系人笔记从 Scout 索引移除
    │   │   │   └─ 所有联系人从 Scout 索引移除
    │   │   └─ 见 10.2 Vault 删除级联
    │   │
    │   ├─ templates → template_pages → module_template_page
    │   │                   └─ modules → module_rows → module_row_fields
    │   │
    │   ├─ post_templates → post_template_sections
    │   │   └─ （注意）posts 使用该模板的 → cascadeOnDelete 删除帖子！
    │   │
    │   ├─ genders / pronouns / religions → contacts 中对应字段 nullOnDelete 置空
    │   │
    │   ├─ contact_information_types → contact_information 数据级联删除
    │   │
    │   ├─ address_types → addresses 中 address_type_id nullOnDelete 置空
    │   │
    │   ├─ pet_categories → pets 数据级联删除
    │   │
    │   ├─ emotions → notes/calls/life_events 中 emotion_id nullOnDelete 置空
    │   │
    │   ├─ call_reason_types → call_reasons → calls（特殊：级联删除通话！）
    │   │
    │   ├─ gift_occasions / gift_states
    │   │
    │   ├─ group_types → groups（先 cascade，后如还有 group_type_id 则 null 置空）
    │   │                  └─ group_type_roles → groups 中 role_id nullOnDelete 置空
    │   │
    │   ├─ relationship_group_types → relationship_types → relationships（双边级联）
    │   │
    │   ├─ sync_tokens
    │   │   └─ address_book_subscriptions 中 sync_token_id nullOnDelete 置空
    │   │
    │   └─ account_currency 中间表
    │       └─（注意）Currency 被删 → nullOnDelete 置空 gifts/loans/life_events 中 currency_id
    │
    └─ Account 记录被删除
```

### 10.2 Vault 删除完整级联流程

```
$vault->delete()
│
├─ 触发 Vault::deleting 事件
│   ├─ 所有联系人的 notes → unsearchable() 从 Scout 索引移除
│   └─ 所有 contacts → unsearchable() 从 Scout 索引移除
│
└─ Vault 级联 cascadeOnDelete
    │
    ├─ contacts（所有联系人）
    │   ├─ 触发 Contact::deleting 事件
    │   │   └─ 该联系人的 notes → unsearchable()
    │   ├─ cascadeOnDelete：
    │   │   ├─ contact_information（电话/邮箱等）
    │   │   ├─ contact_important_dates（生日等）
    │   │   │   └─ contact_important_date_type_id nullOnDelete
    │   │   ├─ contact_reminders → contact_reminder_scheduled
    │   │   ├─ contact_tasks（author_id 如被删则 nullOnDelete）
    │   │   ├─ calls（call_reason_id cascade, emotion_id null, author_id null）
    │   │   ├─ notes（emotion_id null, author_id null, vault_id 也 cascade）
    │   │   ├─ pets
    │   │   ├─ goals → streaks
    │   │   ├─ mood_tracking_events（mood_tracking_parameter_id cascade）
    │   │   ├─ contact_feed_items（author_id null）
    │   │   ├─ quick_facts →（通过 vault_quick_facts_template cascade）
    │   │   ├─ relationships（双边级联：contact_id + related_contact_id）
    │   │   ├─ contact_label 中间表（双边级联）→ labels
    │   │   ├─ contact_group 中间表（双边级联）→ groups
    │   │   ├─ contact_address 中间表（双边级联）→ addresses
    │   │   ├─ contact_post 中间表（双边级联）→ posts
    │   │   ├─ loans → contact_loan 中间表（loaner_id/loanee_id 双边级联）
    │   │   │       └─ gifts（通过 loan_id 级联）
    │   │   ├─ contact_life_metric 中间表（双边级联）
    │   │   ├─ life_event_participants 中间表（双边级联）
    │   │   └─ timeline_event_participants 中间表（双边级联）
    │   └─ nullOnDelete（联系人属性字段）：
    │       ├─ gender_id → NULL
    │       ├─ pronoun_id → NULL
    │       ├─ template_id → NULL
    │       ├─ company_id → NULL
    │       ├─ file_id（头像）→ NULL
    │       └─ religion_id → NULL
    │
    ├─ companies → 所有联系人的 company_id nullOnDelete 置空
    │
    ├─ groups → contact_group / groups（级联）
    │   └─ group_type_id（null）/ group_type_role_id（null）
    │
    ├─ labels → contact_label 中间表（级联）
    │
    ├─ tags → post_tag 中间表（级联）
    │
    ├─ journals → posts → post_sections
    │   │              └─ post_metrics
    │   ├─ journal_metrics → post_metrics（通过 journal_metric_id 级联）
    │   └─ slices_of_life → file_cover_image_id（null）/ 父切片（null）
    │
    ├─ files（注意：DestroyVault 先手动逐个删 File！此处是数据库级联兜底）
    │
    ├─ loans → contact_loan / gifts（级联）
    │   └─ currency_id（null）
    │
    ├─ addresses → contact_address 中间表（级联）
    │   └─ address_type_id（null）
    │
    ├─ contact_important_date_types → 所有日期 type_id（null）
    │
    ├─ mood_tracking_parameters → mood_tracking_events（级联）
    │
    ├─ life_event_categories → life_event_types → timeline_event_types 中间表
    │   └─ life_events → 所有事件相关级联
    │       ├─ emotion_id（null）/ currency_id（null）/ paid_by_contact_id（null）
    │
    ├─ life_metrics → contact_life_metric 中间表（级联）
    │
    ├─ vault_quick_facts_templates → quick_facts（级联）
    │
    ├─ posts → post_sections / post_metrics / post_tag / contact_post
    │   └─（注意）posts 还通过 account_id 级联、post_template_id 级联！
    │
    ├─ user_vault 中间表（级联）
    │
    ├─ contact_vault_user 中间表（级联）
    │
    └─ address_book_subscriptions
        └─ sync_token_id（null）/ last_batch（null）
```

### 10.3 User 删除完整级联流程

```
$user->delete()
│
├─ cascadeOnDelete：
│   ├─ user_tokens（第三方登录令牌）
│   ├─ personal_access_tokens（Sanctum API 令牌）
│   ├─ webauthn_keys（WebAuthn 密钥）
│   ├─ sync_tokens（同步令牌）
│   │   └─（fix_synctokens 迁移中）addressbook_subscriptions 中 sync_token_id（null）
│   ├─ user_vault（用户-Vault 成员关系）
│   │   └─ 连带删除该用户在各 Vault 中的「联系人镜像」（通过 contact_id 级联）
│   ├─ contact_vault_user（联系人收藏/浏览记录）
│   └─ address_book_subscriptions（地址簿订阅）
│
├─ nullOnDelete（作者关联置空）：
│   ├─ notes.author_id → NULL（笔记保留）
│   ├─ contact_tasks.author_id → NULL（任务保留，author_name 冗余保留）
│   ├─ calls.author_id → NULL（通话保留，author_name 冗余保留）
│   └─ contact_feed_items.author_id → NULL（动态保留）
│
├─ 无约束残留（无外键，数据库层面不处理）：
│   ├─ sessions.user_id → 悬空引用（依赖会话过期机制清理）
│   ├─ user_notification_channels.user_id → 悬空引用（遗留问题）
│   │   └─ 但 channel 被删时：contact_reminder_scheduled（级联）+ user_notification_sent（级联）
│   └─ contact_reminders.user_id → 悬空引用（遗留问题）
│
└─ 特殊业务处理（DestroyUser::destroyAllVaults）：
    对每个用户有 MANAGE 权限的 Vault：
        若无其他 MANAGE 权限用户 → $vault->delete()（触发 10.2 整个 Vault 删除流程）
        否则 → 仅通过 user_vault 级联，把用户从 Vault 成员中移除
```

---

## 十一、代码证据汇总

### 11.1 cascadeOnDelete 核心定义示例

```php
// accounts → users：账户被删则所有用户被删
// create_users_table.php L19
$table->foreignIdFor(Account::class)->constrained()->cascadeOnDelete();

// accounts → vaults：账户被删则所有 Vault 被删
// create_vaults_table.php L19
$table->foreignIdFor(Account::class)->constrained()->cascadeOnDelete();

// vaults → contacts：Vault 被删则所有联系人被删
// create_contacts_table.php L25
$table->foreignIdFor(Vault::class)->constrained()->cascadeOnDelete();

// contacts → notes：联系人被删则笔记被删
// create_notes_table.php L21
$table->foreignIdFor(Contact::class)->constrained()->cascadeOnDelete();

// 多对多双边级联：联系人或分组被删则关联解除
// create_group_table.php L62-L63
$table->foreignIdFor(Group::class)->constrained()->cascadeOnDelete();
$table->foreignIdFor(Contact::class)->constrained()->cascadeOnDelete();

// 注意！通话原因被删则通话级联删除（与情绪策略不同）
// create_calls_table.php L23
$table->foreignIdFor(CallReason::class)->nullable()->constrained()->cascadeOnDelete();

// 注意！帖子模板被删则帖子级联删除（与联系人模板策略不同）
// create_posts_table.php L35
$table->foreignIdFor(PostTemplate::class)->constrained()->cascadeOnDelete();
```

### 11.2 nullOnDelete 核心定义示例

```php
// author_id 系列：用户被删则作者置空，内容保留
// create_notes_table.php L23
$table->foreignIdFor(User::class, 'author_id')
    ->nullable()
    ->constrained('users')
    ->nullOnDelete();

// 联系人属性字段：属性被删则置空，联系人保留
// create_contacts_table.php L26-L29
$table->foreignIdFor(Gender::class)->nullable()->constrained()->nullOnDelete();
$table->foreignIdFor(Pronoun::class)->nullable()->constrained()->nullOnDelete();
$table->foreignIdFor(Template::class)->nullable()->constrained()->nullOnDelete();
$table->foreignIdFor(Company::class)->nullable()->constrained()->nullOnDelete();

// 货币关联：货币被删则置空，借贷记录保留
// create_loans_table.php L27
$table->foreignIdFor(Currency::class)->nullable()->constrained()->nullOnDelete();

// 情绪关联：情绪被删则置空，内容保留
// create_calls_table.php L25
$table->foreignIdFor(Emotion::class)->nullable()->constrained()->nullOnDelete();

// Vault 默认模板：模板被删则取消默认设置
// create_vaults_table.php L23
$table->foreignIdFor(Template::class, 'default_template_id')
    ->nullable()
    ->constrained('templates')
    ->nullOnDelete();

// 生活事件付款人：付款联系人被删则置空，事件保留
// create_life_events_table.php L70
$table->foreignIdFor(Contact::class, 'paid_by_contact_id')
    ->nullable()
    ->constrained('contacts')
    ->nullOnDelete();
```

### 11.3 无外键约束定义示例（潜在问题）

```php
// sessions：仅有索引，无 constrained()，删除用户时不处理
// create_sessions_table.php L19
$table->foreignIdFor(User::class)->nullable()->index();
// 注意：少了 ->constrained()，无外键！

// user_notification_channels：仅有 nullable()，无 constrained()
// create_reminders_table.php L36
$table->foreignIdFor(User::class)->nullable();
// 注意：少了 ->constrained()，无外键！
```

---

## 附录：author_id 字段级联规则分析
