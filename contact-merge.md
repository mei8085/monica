# Contact Merge 分析

基于 Monica 代码库的联系人合并路径分析。当前代码库 **不存在** `MergeContact` 服务，本文档沿现有 `DestroyContact`、`MoveContactToAnotherVault`、`CopyContactToAnotherVault`、`ImportContact` 以及 `BaseService` 事务/权限模式，推导合并操作的完整设计。

---

## 1. 主联系人选择（Primary Contact Selection）

### 1.1 现有代码中的参照

| 场景 | 代码位置 | 策略 |
|------|---------|------|
| VCard 导入去重 | [ImportContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L80-L111) | 依次按 `uri → distant_uri → uid` 查找已存在联系人，**找到即覆盖** |
| Move 跨 Vault | [MoveContactToAnotherVault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Services/MoveContactToAnotherVault.php#L77-L81) | 无选择——直接搬整个 Contact 记录 |
| Copy 跨 Vault | [CopyContactToAnotherVault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Services/CopyContactToAnotherVault.php#L77-L84) | `replicate()` 浅拷贝，无字段取舍 |

### 1.2 合并场景下的主联系人判定依据

需从两个同 Vault 的 Contact 中选一个 **保留**（survivor），另一个 **吸收后删除**（absorbed）。判定优先级建议：

1. **用户显式指定**：前端传入 `survivor_id`，最可靠。
2. **数据完整度**（自动兜底）：按字段非空计数 + 关联对象数量排序：
   - `first_name` / `last_name` 非空
   - `contactInformations` 数量
   - `notes` / `importantDates` / `reminders` 数量
   - `file_id` 是否非空（有头像）
   - `last_updated_at` 最新
3. **外部同步权重**：若一个 Contact 有 `distant_uuid`（来自 CardDAV 同步），优先保留它，否则同步可能中断。

### 1.3 关键约束

- 两个 Contact **必须在同一个 Vault** 内——`MoveContactToAnotherVault` 已验证 `vault_id` 一致性；合并同样需要。
- BaseService 权限链要求 `contact_must_belong_to_vault`，合并需对两个 Contact 都做此校验。

---

## 2. 字段冲突处理（Field Conflict Resolution）

### 2.1 Contact 本体字段全景

来自 [Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Contact.php#L43-L67) 的 `$fillable`：

| 字段 | 类型 | 冲突策略 |
|------|------|---------|
| `first_name` | string nullable | survivor 优先；若 survivor 为空则取 absorbed |
| `last_name` | string nullable | 同上 |
| `middle_name` | string nullable | 同上 |
| `nickname` | string nullable | 同上 |
| `maiden_name` | string nullable | 同上 |
| `prefix` | string nullable | 同上 |
| `suffix` | string nullable | 同上 |
| `gender_id` | FK nullable | survivor 优先；若冲突需用户裁定 |
| `pronoun_id` | FK nullable | survivor 优先 |
| `template_id` | FK nullable | survivor 优先 |
| `company_id` | FK nullable | ⚠️ 复杂——见第 3 节 |
| `job_position` | string nullable | survivor 优先 |
| `religion_id` | FK nullable | survivor 优先 |
| `vault_id` | FK | **必须相同**，否则拒绝合并 |
| `can_be_deleted` | boolean | survivor 保留 |
| `listed` | boolean | survivor 保留；若 survivor archived 而 absorbed active，需用户确认 |
| `show_quick_facts` | boolean | survivor 保留 |
| `file_id` | FK nullable (头像) | survivor 有则保留；无则取 absorbed 的 |
| `vcard` | mediumText | survivor 保留；合并后需重新导出 |
| `distant_uuid` | string nullable | ⚠️ 只能有一个；见第 3 节 DAV 同步 |
| `distant_etag` | string nullable | 跟随 `distant_uuid` |
| `distant_uri` | string nullable | 跟随 `distant_uuid` |
| `last_updated_at` | datetime | 合并完成时刷新为 `Carbon::now()` |

### 2.2 冲突判定模式

```
对于每个标量字段 field:
  if survivor->field 非空:
    保留 survivor->field（主联系人优先原则）
  elif absorbed->field 非空:
    填入 absorbed->field（补白原则）
  else:
    两者都为空，保持 null
```

**例外**：FK 字段（`gender_id`, `pronoun_id`, `religion_id`）两者均非空且不同时——需用户选择或自动取 survivor 值。可在 API 层返回冲突列表让前端展示。

### 2.3 UpdateContact 服务可复用

[UpdateContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Services/UpdateContact.php#L59-L88) 已实现字段逐一赋值 + `last_updated_at` 刷新 + `ContactFeedItem` 写入。合并时可复用此服务将 absorbed 的补白字段写入 survivor。

---

## 3. 关联对象迁移/重绑定（Related Object Rebinding）

这是合并中最复杂的部分。Contact 有 **16 种直接关联**，需逐一分析迁移策略。

### 3.1 HasMany 关系（直接改 `contact_id`）

| 关联 | 模型 | FK 字段 | 特殊处理 |
|------|------|---------|---------|
| `contactInformations` | [ContactInformation](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/ContactInformation.php) | `contact_id` | ⚠️ 需去重——同 `type_id` + `data` 的记录只保留一条 |
| `notes` | [Note](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Note.php) | `contact_id` | 直接迁移，无冲突；但 `vault_id` 需一致 |
| `importantDates` | [ContactImportantDate](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/ContactImportantDate.php) | `contact_id` | ⚠️ 需去重——同 `contact_important_date_type_id` + `day/month/year` |
| `reminders` | [ContactReminder](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/ContactReminder.php) | `contact_id` | ⚠️ 需去重——同 `label` + `day/month/year/type` |
| `calls` | [Call](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Call.php) | `contact_id` | 直接迁移 |
| `pets` | [Pet](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Pet.php) | `contact_id` | ⚠️ 需去重——同 `name` + `pet_category_id` |
| `goals` | [Goal](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Goal.php) | `contact_id` | ⚠️ 需去重——同 `name` |
| `tasks` | [ContactTask](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/ContactTask.php) | `contact_id` | 直接迁移 |
| `moodTrackingEvents` | [MoodTrackingEvent](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/MoodTrackingEvent.php) | `contact_id` | 直接迁移 |
| `quickFacts` | [QuickFact](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/QuickFact.php) | `contact_id` | ⚠️ 需去重——同 `vault_quick_facts_template_id` |
| `contactFeedItems` | [ContactFeedItem](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/ContactFeedItem.php) | `contact_id` | 直接迁移；历史记录不应丢失 |

**迁移方式**：统一 `UPDATE table SET contact_id = survivor_id WHERE contact_id = absorbed_id`。去重时先查询 survivor 已有记录，删除 absorbed 中重复的。

### 3.2 BelongsToMany 关系（改 pivot 表）

| 关联 | Pivot 表 | Pivot FK | 特殊处理 |
|------|---------|----------|---------|
| `labels` | `contact_label` | `contact_id` | `syncWithoutDetaching`——去重自动处理 |
| `groups` | `contact_group` | `contact_id` | 同上；注意 `group_type_role_id` 额外字段 |
| `addresses` | `contact_address` | `contact_id` | 需保留 `is_past_address` pivot 字段；用 `syncWithoutDetaching` |
| `posts` | `contact_post` | `contact_id` | `syncWithoutDetaching` |
| `lifeEvents` | `life_event_participants` | `contact_id` | `syncWithoutDetaching` |
| `timelineEvents` | `timeline_event_participants` | `contact_id` | `syncWithoutDetaching` |
| `lifeMetrics` | `contact_life_metric` | `contact_id` | `syncWithoutDetaching` |

**迁移方式**：直接更新 pivot 表的 `contact_id`。对于 `syncWithoutDetaching`，不会产生重复——Laravel 底层会用 upsert。

### 3.3 双向 BelongsToMany 关系（Relationships）

这是**最棘手**的关联，定义于 [Contact.php#L186](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Contact.php#L184-L187)：

```
relationships → belongsToMany(Contact, 'relationships', 'contact_id', 'related_contact_id')
```

Pivot 表 `relationships` 结构（来自 [migration](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/database/migrations/2021_10_16_184625_create_relationship_types_table.php#L40-L46)）：

| 列 | 说明 |
|----|------|
| `contact_id` | 主动方 |
| `related_contact_id` | 被关联方 |
| `relationship_type_id` | 关系类型 |

**迁移规则**（4 种场景）：

1. **absorbed 作为主动方** (`contact_id = absorbed_id`)：
   - 更新为 `contact_id = survivor_id`
   - ⚠️ 若 survivor 已有对同一 `related_contact_id` 的关系，需去重

2. **absorbed 作为被关联方** (`related_contact_id = absorbed_id`)：
   - 更新为 `related_contact_id = survivor_id`
   - ⚠️ 同上需去重

3. **absorbed 与 survivor 之间的互相关系**：
   - 如 absorbed 是 survivor 的父亲，合并后 `contact_id = survivor_id, related_contact_id = survivor_id` → **自引用**，无意义，应删除

4. **其他 Contact 指向 absorbed 的关系**：
   - 所有 `related_contact_id = absorbed_id` 的记录需改为 `related_contact_id = survivor_id`

SQL 概要：

```sql
-- 场景 1: absorbed 主动方 → 改为 survivor 主动方
UPDATE relationships SET contact_id = ?  -- survivor_id
WHERE contact_id = ?                     -- absorbed_id
  AND related_contact_id != ?;           -- 排除与 survivor 的互相关系

-- 场景 2: absorbed 被关联方 → 改为 survivor 被关联方
UPDATE relationships SET related_contact_id = ?  -- survivor_id
WHERE related_contact_id = ?                     -- absorbed_id
  AND contact_id != ?;                           -- 排除与 survivor 的互相关系

-- 场景 3: 删除 survivor ↔ absorbed 的互相关系
DELETE FROM relationships
WHERE (contact_id = ? AND related_contact_id = ?)  -- survivor → absorbed
   OR (contact_id = ? AND related_contact_id = ?); -- absorbed → survivor
```

去重（场景 1/2 更新后可能产生重复行）：

```sql
-- 删除重复关系，保留 id 最小的
DELETE r1 FROM relationships r1
INNER JOIN relationships r2
  ON r1.contact_id = r2.contact_id
  AND r1.related_contact_id = r2.related_contact_id
  AND r1.relationship_type_id = r2.relationship_type_id
  AND r1.id > r2.id;
```

### 3.4 BelongsToMany 关系（Loans——非对称双端）

[Loan](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Loan.php) 的 pivot 表 `contact_loan` 结构（来自 [migration](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/database/migrations/2022_03_23_005751_create_loans_table.php#L34-L39)）：

| 列 | 说明 |
|----|------|
| `loan_id` | 借贷记录 |
| `loaner_id` | 借出方 (Contact FK) |
| `loanee_id` | 借入方 (Contact FK) |

**迁移规则**：

```sql
-- absorbed 是借出方
UPDATE contact_loan SET loaner_id = ? WHERE loaner_id = ?;

-- absorbed 是借入方
UPDATE contact_loan SET loanee_id = ? WHERE loanee_id = ?;
```

⚠️ **自引用问题**：若同一 Loan 的 loaner 和 loanee 都指向同一个人（合并后），该条记录语义上变成"自己借给自己"，需删除。

⚠️ **去重问题**：若 survivor 和 absorbed 都是同一 Loan 的 loaner（或 loanee），更新后会产生重复行。

### 3.5 MorphMany/MorphTo 关系

| 关联 | 模型 | Polymorphic 字段 | 处理 |
|------|------|-----------------|------|
| `files` | [File](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/File.php) | `fileable_type` + `fileable_id` | `fileable_type = Contact` 时更新 `fileable_id` |
| `avatar (file_id)` | File | `contacts.file_id` FK | survivor 有则保留，无则取 absorbed 的 |
| `feedItems (morphOne)` | ContactFeedItem | `feedable_type` + `feedable_id` | 各子对象的 feedItem 跟随其主体迁移 |

### 3.6 LifeEvent.paid_by_contact_id

[LifeEvent](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/LifeEvent.php#L95-L98) 有独立 FK `paid_by_contact_id` 指向 Contact：

```sql
UPDATE life_events SET paid_by_contact_id = ? WHERE paid_by_contact_id = ?;
```

### 3.7 contact_vault_user（用户维度数据）

来自 [contacts 迁移](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L69-L76)：

| 列 | 说明 |
|----|------|
| `contact_id` | 联系人 |
| `vault_id` | Vault |
| `user_id` | 用户 |
| `number_of_views` | 浏览次数 |
| `is_favorite` | 是否收藏 |

**迁移**：将 absorbed 的浏览次数加到 survivor 上；收藏取 OR（任一为 true 则 true）。更新后删除 absorbed 的记录。

### 3.8 Company 处理

参照 [MoveContactToAnotherVault::moveCompanyInformation()](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Services/MoveContactToAnotherVault.php#L90-L109) 的逻辑：

- 若 absorbed 的 company 只有 absorbed 一个 Contact → 整个 Company 也迁移给 survivor
- 若 absorbed 的 company 有其他 Contact → 只改 `contact.company_id`，不移动 Company

合并时更简单：若 survivor 已有 company 且与 absorbed 不同，按 survivor 优先；否则取 absorbed 的 company_id。

### 3.9 DAV 同步字段（distant_uuid/etag/uri）

合并时 **只能保留一组 DAV 标识**。参照 [ImportContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L62-L77)：

- survivor 有 `distant_uuid` → 保留 survivor 的，合并后重新导出 vcard
- survivor 无 `distant_uuid` 但 absorbed 有 → 迁移 absorbed 的到 survivor
- 两者都有 → survivor 优先；absorbed 的 `distant_uuid` 对应的远端记录需通过 [AddressBookSynchronizer](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/DavClient/Services/Utils/AddressBookSynchronizer.php) 推送删除

---

## 4. 失败恢复路径（Failure Recovery）

### 4.1 现有代码的错误处理模式

| 模式 | 代码位置 | 说明 |
|------|---------|------|
| `QueuableService` + `tries = 1` | [QueuableService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Services/QueuableService.php#L22) | 队列任务只执行一次，失败不重试 |
| `Bus::batch()->allowFailures()` | [AddressBookSynchronizer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/DavClient/Services/Utils/AddressBookSynchronizer.php#L48) | 批量任务允许部分失败 |
| `CantBeDeletedException` | [DestroyContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Services/DestroyContact.php#L44) | `can_be_deleted = false` 时阻止删除 |
| `SoftDeletes` | [Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Contact.php#L29) | Contact 使用软删除 |

### 4.2 合并操作的事务策略

合并是 **多步写操作**，中途失败需回滚。推荐使用数据库事务：

```php
DB::transaction(function () use ($survivor, $absorbed) {
    // 1. 补白 survivor 字段
    // 2. 迁移所有 HasMany 关系
    // 3. 更新所有 Pivot 表
    // 4. 处理 Relationships 双向绑定
    // 5. 处理 Loans 非对称绑定
    // 6. 迁移 contact_vault_user
    // 7. 软删除 absorbed
});
```

**事务内失败**：自动回滚，无数据损失。

### 4.3 事务外的风险点

| 风险 | 场景 | 恢复策略 |
|------|------|---------|
| DAV 同步推送失败 | survivor 的 vcard 需重新推送到 CardDAV 服务器 | `UpdateVCard` 已是队列任务，自动重试；[CardDAVBackend::updateCard()](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L437-L456) 使用 `Bus::batch` 允许失败 |
| Scout 索引未更新 | absorbed 的 `notes` 调用了 `unsearchable()` 但 survivor 的未 `searchable()` | 合并后显式调用 `$survivor->searchable()` 和相关子对象的重新索引 |
| absorbed 的 `distant_uuid` 对应远端记录残留 | 合并后 absorbed 被删除，但远端服务器仍有对应 vCard | 触发 `DestroyContact` 的队列任务，它会被 [CardDAVBackend::deleteCard()](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L464-L489) 处理为推送删除 |

### 4.4 软删除与恢复

Contact 使用 `SoftDeletes`。合并时：

1. **absorbed 应被软删除**，而非硬删除——这样在事务回滚或需要恢复时有据可查。
2. 软删除后，`relationships`、`contact_label` 等 pivot 表中指向 absorbed 的记录因 `cascadeOnDelete` 外键约束会被自动删除——**这是事务回滚的关键依赖**。如果外键是 `cascadeOnDelete`，回滚时这些记录无法自动恢复。
3. **建议**：不在事务内真正删除 absorbed，而是先将 absorbed 的 `listed` 设为 `false` + `can_be_deleted` 设为 `false`，让 absorbed 变为"不可见但存在"状态。待所有迁移确认成功后，再通过 `DestroyContact` 软删除。

### 4.5 幂等性保障

若合并操作因超时等原因被重复执行，需确保幂等：

- HasMany 迁移：先检查 `contact_id` 是否已指向 survivor，是则跳过
- Pivot 更新：使用 `syncWithoutDetaching` 而非 `attach`，天然幂等
- 字段补白：仅当 survivor 字段为空时才写入，重复执行无副作用

---

## 5. 完整合并流程

```
MergeContact::execute([
    'account_id', 'vault_id', 'author_id',
    'survivor_id',       // 保留的联系人
    'absorbed_id',       // 被吸收的联系人
    'field_overrides?',  // 可选：用户指定的字段冲突解决方案
])
```

### 执行步骤

1. **校验**
   - `validateRules`：两个 contact_id 都在同一个 vault
   - survivor 和 absorbed 不是同一个 Contact
   - 权限：`author_must_be_vault_editor`

2. **字段合并**
   - 对 Contact 本体标量字段执行补白逻辑
   - 若传入 `field_overrides`，按用户选择覆盖
   - 处理 `company_id`（参照 MoveContact 逻辑）
   - 处理 `file_id`（头像）
   - 处理 DAV 字段（`distant_uuid/etag/uri`）
   - 刷新 `last_updated_at`

3. **关联迁移**
   - HasMany：批量 `UPDATE ... SET contact_id = survivor_id WHERE contact_id = absorbed_id`
   - HasMany 去重：对 `contactInformations`、`importantDates`、`reminders`、`pets`、`goals`、`quickFacts` 做去重
   - BelongsToMany Pivot：批量更新 `contact_id`
   - Relationships：处理 4 种场景（见 3.3）
   - Loans：处理非对称双端 + 自引用检测
   - LifeEvent `paid_by_contact_id`：更新
   - File polymorphic：更新 `fileable_id`
   - ContactFeedItem：更新 `contact_id`
   - `contact_vault_user`：合并 `number_of_views` 和 `is_favorite`

4. **清理**
   - 将 absorbed 标记为 `listed = false` + `can_be_deleted = false`
   - 触发 `DestroyContact` 队列任务软删除 absorbed
   - 创建 `ContactFeedItem` 记录合并动作

5. **同步**
   - 刷新 survivor 的 `vcard`（通过 `ExportVCard`）
   - 触发 `PushVCard` 通知 CardDAV 服务器

---

## 6. 关联对象清单速查

| # | 关联类型 | 关联对象 | 迁移方式 | 去重 | 自引用风险 |
|---|---------|---------|---------|------|-----------|
| 1 | HasMany | ContactInformation | UPDATE contact_id | ✅ type_id+data | ❌ |
| 2 | HasMany | Note | UPDATE contact_id | ❌ | ❌ |
| 3 | HasMany | ContactImportantDate | UPDATE contact_id | ✅ type+date | ❌ |
| 4 | HasMany | ContactReminder | UPDATE contact_id | ✅ label+date+type | ❌ |
| 5 | HasMany | Call | UPDATE contact_id | ❌ | ❌ |
| 6 | HasMany | Pet | UPDATE contact_id | ✅ name+category | ❌ |
| 7 | HasMany | Goal | UPDATE contact_id | ✅ name | ❌ |
| 8 | HasMany | ContactTask | UPDATE contact_id | ❌ | ❌ |
| 9 | HasMany | MoodTrackingEvent | UPDATE contact_id | ❌ | ❌ |
| 10 | HasMany | QuickFact | UPDATE contact_id | ✅ template_id | ❌ |
| 11 | HasMany | ContactFeedItem | UPDATE contact_id | ❌ | ❌ |
| 12 | BelongsToMany | Label (contact_label) | UPDATE pivot | sync | ❌ |
| 13 | BelongsToMany | Group (contact_group) | UPDATE pivot | sync | ❌ |
| 14 | BelongsToMany | Address (contact_address) | UPDATE pivot | sync | ❌ |
| 15 | BelongsToMany | Post (contact_post) | UPDATE pivot | sync | ❌ |
| 16 | BelongsToMany | LifeEvent (life_event_participants) | UPDATE pivot | sync | ❌ |
| 17 | BelongsToMany | TimelineEvent (timeline_event_participants) | UPDATE pivot | sync | ❌ |
| 18 | BelongsToMany | LifeMetric (contact_life_metric) | UPDATE pivot | sync | ❌ |
| 19 | BelongsToMany(双向) | Relationship (relationships) | 4 场景 SQL | ✅ | ✅ |
| 20 | BelongsToMany(非对称) | Loan (contact_loan) | 双端 UPDATE | ✅ | ✅ |
| 21 | BelongsTo FK | LifeEvent.paid_by_contact_id | UPDATE FK | ❌ | ❌ |
| 22 | MorphMany | File (fileable) | UPDATE fileable_id | ❌ | ❌ |
| 23 | BelongsTo | File (avatar, file_id) | 补白逻辑 | ❌ | ❌ |
| 24 | Pivot | contact_vault_user | 合并+DELETE | ❌ | ❌ |
| 25 | BelongsTo | Company (company_id) | 条件逻辑 | ❌ | ❌ |
| 26 | DAV 字段 | distant_uuid/etag/uri | 单组保留 | ❌ | ❌ |
