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
| `user_notification_channels` | 用户通知渠道 | 一对多 |
| `user_notification_sent` | 通知发送记录 | 通过通知渠道级联 |
| `user_tokens` | 第三方登录令牌 | 一对多 |
| `personal_access_tokens` | API 访问令牌 (Sanctum) | 一对多 |
| `webauthn_keys` | WebAuthn 密钥 | 一对多 |
| `user_vault` | 用户-Vault 关联表 | 多对多中间表 |
| `contact_vault_user` | 联系人-用户收藏/浏览记录 | 多对多中间表 |
| `address_book_subscriptions` | 地址簿订阅 | 一对多 |
| `sync_tokens` | 同步令牌 | 一对多 |

另外，用户作为作者的记录（通过 `author_id` 关联）：
- `notes` — 笔记（**注意**：`author_id` 外键需要确认是否级联）
- `contact_tasks` — 联系人任务（**注意**：`author_id` 外键需要确认是否级联）

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
