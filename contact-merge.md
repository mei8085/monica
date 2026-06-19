# Contact Merge 代码分析

> 基于 Monica 代码库的事实性分析。**当前代码库不存在 `MergeContact` 服务**，本文档沿现有代码路径（`DestroyContact`、`BaseService`、`SetRelationship` 等）推导合并操作的正确实现方式，并标注哪些是代码事实、哪些是推导结论。

---

## 1. 现有代码中的相关事实

### 1.1 没有合并服务

- **事实**：代码库中**不存在** `MergeContact` / `DuplicateContact` / `ContactMerge` 等合并相关服务。
- **事实**：最接近"去重"的逻辑在 `ImportContact` 中——CardDAV 导入时按 `uri → distant_uri → uid` 查找已存在联系人，找到则**覆盖更新**，而非合并。
  见 [ImportContact.php::getExistingContact()](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L80-L111)

### 1.2 软删除模型清单

使用 `SoftDeletes` 的模型只有 **4 个**：

| 模型 | 文件 |
|------|------|
| Contact | [Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Contact.php#L29) |
| Group | [Group.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Group.php#L21) |
| ContactTask | [ContactTask.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/ContactTask.php#L16) |
| ContactImportantDate | [ContactImportantDate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/ContactImportantDate.php#L21) |

其余模型（Note、Call、Pet、Goal、File、ContactInformation 等）**均为硬删除**。

### 1.3 软删除 ≠ 级联删除

- **事实**：数据库迁移中大量使用 `cascadeOnDelete()`（如 notes、calls、relationships 等表的 `contact_id` 外键）。
- **事实**：`cascadeOnDelete` 是**数据库层**的外键约束，只在 **`DELETE` 语句**（硬删除）时触发。
- **事实**：Laravel 的 `SoftDeletes` 是通过 `UPDATE contacts SET deleted_at = ?` 实现的，**不会触发**数据库外键级联。
- **推导**：因此 `$contact->delete()`（软删除）后，该 Contact 的所有 hasMany / belongsToMany 关联记录**仍然留在数据库中**，只是通过 Eloquent 普通查询无法触达。

### 1.4 DestroyContact 的实际行为

[DestroyContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Services/DestroyContact.php) 只做两件事：

1. 遍历删除所有 `files`（File 模型硬删除，触发 `FileDeleted` 事件）
2. `$this->contact->delete()` —— **软删除**

它**不会**级联删除 notes、calls、tasks 等关联记录。这些记录会成为"孤儿数据"，但因为外键约束是 `cascadeOnDelete`，只有未来硬删除 Contact 时才会被清理。

### 1.5 Contact 的 deleting 事件

[Contact::boot()](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Contact.php#L120-L127) 的 `deleting` 事件只做一件事：

```php
static::deleting(function (self $model) {
    $model->notes()->unsearchable();
});
```

即从 Scout 搜索索引中移除 notes，没有任何级联删除逻辑。

---

## 2. 主联系人选择（Primary Selection）

### 2.1 现有代码中的选择依据

| 场景 | 代码位置 | 选择策略 |
|------|---------|---------|
| CardDAV 导入去重 | [ImportContact::getExistingContact()](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L80-L111) | `uri` 匹配 → `distant_uri` 匹配 → `uid` 匹配，**先命中者为准** |
| Move 跨 Vault | [MoveContactToAnotherVault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Services/MoveContactToAnotherVault.php#L77-L81) | 无选择，整体搬运 |
| Copy 跨 Vault | [CopyContactToAnotherVault.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Services/CopyContactToAnotherVault.php#L77-L84) | `replicate()` 浅拷贝 |

### 2.2 合并场景的选择逻辑（推导）

合并需要从两个同 Vault 的 Contact 中选一个 **survivor**（保留），另一个 **absorbed**（被吸收后删除）。推荐判定优先级：

1. **用户显式指定**（前端传 `survivor_id`）——最可靠，也是现有服务的惯用模式（所有服务都通过参数指定操作对象）。
2. **数据完整度自动兜底**（如果没有指定 survivor）：
   - 有 `distant_uuid` 的优先（保证 CardDAV 同步不中断，见 [ImportContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L62-L77)）
   - `first_name` + `last_name` 非空的优先
   - 关联对象数量更多的优先（notes、informations、reminders 等计数）
   - `last_updated_at` 较新的优先
3. **必须同 Vault**：两个 Contact 的 `vault_id` 必须一致，否则拒绝。

---

## 3. 权限校验

### 3.1 BaseService 的权限机制（事实）

[BaseService::validateRules()](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Services/BaseService.php#L96-L119) 的执行流程：

1. 先做 `Validator::make($data, $this->rules())->validate()`
2. 遍历 `self::$permissionDependencies` 的键顺序，检查权限声明中是否包含该权限
3. 若包含，则执行对应的 `validatePermission()` 方法
4. 权限有依赖关系（如 `contact_must_belong_to_vault` 依赖 `vault_must_belong_to_account`）
5. 未在 `$permissionDependencies` 中声明的权限会抛出 `Unknown permission` 异常

### 3.2 权限校验的关键属性（事实）

- `BaseService` 只有 **一个** `$this->contact` 属性
- 该属性通过 `contact_must_belong_to_vault` 权限校验时设置，数据源是请求参数中的 `contact_id`
- 一次 service 调用只能校验"一个主 Contact"的归属

### 3.3 合并时的权限处理（推导）

合并操作涉及两个 Contact，需要特殊处理：

```
permissions(): array = [
    'author_must_belong_to_account',
    'vault_must_belong_to_account',
    'author_must_be_vault_editor',
    'contact_must_belong_to_vault',  // 用 survivor_id 设 $this->contact
]
```

然后在 `validate()` 中**手动校验**第二个 Contact：

```php
$absorbed = $this->vault->contacts()->findOrFail($data['absorbed_id']);
```

这样可以复用 `author_must_be_vault_editor` 的 Vault 级权限校验，只需确保第二个 Contact 也属于同一个 Vault。

### 3.4 为什么不用两次 contact_must_belong_to_vault？

因为 `BaseService` 只有一个 `$this->contact` 属性，权限校验通过后会覆盖它。声明两次没有意义——第二次会覆盖第一次。

---

## 4. 字段冲突处理

### 4.1 Contact 本体字段（事实）

来自 [Contact.php::$fillable](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Contact.php#L43-L67)，共 22 个可填充字段：

| 字段 | 类型 | 冲突策略 |
|------|------|---------|
| `first_name / last_name / middle_name` | string nullable | survivor 优先，空则补 absorbed |
| `nickname / maiden_name` | string nullable | 同上 |
| `prefix / suffix` | string nullable | 同上 |
| `gender_id` | FK nullable | survivor 优先 |
| `pronoun_id` | FK nullable | survivor 优先 |
| `religion_id` | FK nullable | survivor 优先 |
| `template_id` | FK nullable | survivor 优先 |
| `job_position` | string nullable | survivor 优先 |
| `company_id` | FK nullable | survivor 优先；特殊处理见 MoveContact 逻辑 |
| `file_id` (头像) | FK nullable | survivor 有则保留，无则取 absorbed 的 |
| `can_be_deleted` | boolean | survivor 保留 |
| `listed` | boolean | survivor 保留 |
| `show_quick_facts` | boolean | survivor 保留 |
| `vcard` | mediumText | 合并后重新导出，用 survivor 的 |
| `distant_uuid / distant_etag / distant_uri` | string nullable | ⚠️ 只能保留一组，详见 DAV 章节 |

### 4.2 非 fillable 但需要处理的字段

| 字段 | 说明 |
|------|------|
| `vault_id` | 必须相同，否则拒绝合并 |
| `last_updated_at` | 合并完成时刷新为当前时间（参照 [UpdateContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Services/UpdateContact.php#L83-L85) 的 `updateLastEditedDate()` 模式） |

### 4.3 冲突判定算法（推导）

```
对于每个字段 field:
  if survivor->field 非空:
    保留 survivor->field        // 主联系人优先原则
  elif absorbed->field 非空:
    写入 absorbed->field        // 补白原则
  else:
    保持 null
```

FK 字段（gender、pronoun、religion、company）两者均非空且不同时——默认取 survivor 值，如需用户介入可在 API 层返回冲突列表。

### 4.4 可复用的 UpdateContact 服务

[UpdateContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContact/Services/UpdateContact.php) 已实现：
- 字段逐一赋值
- `last_updated_at` 刷新
- `ContactFeedItem` 写入

但合并时**不建议直接调用**，因为合并需要批量处理且有自己的 feed 语义。建议直接操作 model。

---

## 5. 关联对象迁移（重绑）

这是合并中最复杂的部分。Contact 有 **20+ 种关联**，迁移方式各不相同。

### 5.1 附件（Files）—— 最容易搞错的部分

**事实**：Contact 有 **两套**文件关联，用的字段完全不同：

| 关联 | 关系类型 | 用的字段 | 用途 |
|------|---------|---------|------|
| `file()` | BelongsTo | `contacts.file_id` | 头像（单张） |
| `files()` | MorphMany | `files.ufileable_id` + `files.fileable_type` | 附件/照片（多张） |

**关键事实**：
- `Contact::files()` 用的是 `morphMany(File::class, 'ufileable', 'fileable_type')`，见 [Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Contact.php#L318-L321)
- `files` 表有 **两套** morph 字段：
  - `fileable_id` (int) + `fileable_type` —— 普通 numeric morph
  - `ufileable_id` (uuid) + `fileable_type` —— UUID morph，Contact 用的是这套
- 见迁移 [create_files_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/database/migrations/2022_02_24_002342_create_files_table.php#L22-L24)

**迁移方式（推导）**：
- **头像 `file_id`**：按字段补白逻辑（survivor 有则保留，无则取 absorbed 的）
- **附件 `files()`**：
  - 直接 `UPDATE files SET ufileable_id = ? WHERE ufileable_id = ? AND fileable_type = 'contact'`
  - 无需去重（文件是独立对象，即使内容相同也是两条记录）
  - ⚠️ 注意用的是 `ufileable_id`，**不是** `fileable_id`

### 5.2 HasMany 关系（直接改 contact_id）

| 关联 | 模型 | 表 | 软删除 | 去重依据 |
|------|------|-----|--------|---------|
| `contactInformations` | [ContactInformation](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/ContactInformation.php) | `contact_information` | ❌ 硬删 | `type_id` + `data` |
| `notes` | [Note](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Note.php) | `notes` | ❌ 硬删 | 否（每篇笔记是独立的） |
| `importantDates` | [ContactImportantDate](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/ContactImportantDate.php) | `contact_dates` | ✅ 软删 | `contact_important_date_type_id` + `day/month/year` |
| `reminders` | [ContactReminder](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/ContactReminder.php) | `contact_reminders` | ❌ 硬删 | `label` + `day/month/year/type` |
| `calls` | [Call](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Call.php) | `calls` | ❌ 硬删 | 否 |
| `pets` | [Pet](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Pet.php) | `pets` | ❌ 硬删 | `name` + `pet_category_id` |
| `goals` | [Goal](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/Goal.php) | `goals` | ❌ 硬删 | `name` |
| `tasks` | [ContactTask](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/ContactTask.php) | `contact_tasks` | ✅ 软删 | 否 |
| `moodTrackingEvents` | [MoodTrackingEvent](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/MoodTrackingEvent.php) | `mood_tracking_events` | ❌ 硬删 | 否 |
| `quickFacts` | [QuickFact](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/QuickFact.php) | `vault_quick_facts_template` | ❌ 硬删 | `vault_quick_facts_template_id` |
| `contactFeedItems` | [ContactFeedItem](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Models/ContactFeedItem.php) | `contact_feed` | ❌ 硬删 | 否 |

**迁移 SQL**：
```sql
UPDATE {table} SET contact_id = {survivor_id} WHERE contact_id = {absorbed_id};
```

**去重逻辑**（对需要去重的表）：先查出 survivor 已有的记录，删除 absorbed 中重复的，再更新剩余的。

### 5.3 BelongsToMany 关系（改 pivot 表）

| 关联 | Pivot 表 | Pivot FK | 额外字段 | 幂等操作 |
|------|---------|----------|---------|---------|
| `labels` | `contact_label` | `contact_id` | 无 | `syncWithoutDetaching` |
| `groups` | `contact_group` | `contact_id` | `group_type_role_id` | `syncWithoutDetaching` |
| `addresses` | `contact_address` | `contact_id` | `is_past_address` | 需注意 |
| `posts` | `contact_post` | `contact_id` | 无 | `syncWithoutDetaching` |
| `lifeEvents` | `life_event_participants` | `contact_id` | 无 | `syncWithoutDetaching` |
| `timelineEvents` | `timeline_event_participants` | `contact_id` | 无 | `syncWithoutDetaching` |
| `lifeMetrics` | `contact_life_metric` | `contact_id` | 无 | `syncWithoutDetaching` |

**现有代码中的惯用操作**：
- [AssignLabel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageLabels/Services/AssignLabel.php#L52) 用 `syncWithoutDetaching()`
- [AddContactToGroup.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageGroups/Services/AddContactToGroup.php#L56-L58) 用 `syncWithoutDetaching()`
- [AssociateAddressToContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageContactAddresses/Services/AssociateAddressToContact.php#L68) **用 `attach()`** —— 这意味着地址可以重复关联？

**迁移方式（推导）**：
- 推荐统一用 `UPDATE pivot SET contact_id = ? WHERE contact_id = ?`
- 对于有唯一约束的 pivot 表，更新后可能产生重复行，需要去重
- 或者用更安全的方式：先查出 absorbed 的所有关联，对每条用 `syncWithoutDetaching` 挂到 survivor 上，再删除 absorbed 的 pivot 记录

### 5.4 双向关系（Relationships）—— 最棘手

**事实**：`relationships` 表结构（见 [迁移](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/database/migrations/2021_10_16_184625_create_relationship_types_table.php#L40-L46)）：

| 列 | 说明 |
|----|------|
| `contact_id` | 主动方（FK → contacts.id） |
| `related_contact_id` | 被关联方（FK → contacts.id） |
| `relationship_type_id` | 关系类型 |

两个外键都 `cascadeOnDelete`（但只对硬删除生效）。

**现有代码中的操作模式**：
- [SetRelationship.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageRelationships/Services/SetRelationship.php) 会创建正向和反向两条关系记录
- [UnsetRelationship.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageRelationships/Services/UnsetRelationship.php) 会删除正向和反向两条记录

**合并时的 4 种场景（推导）**：

```
场景 1: absorbed 是主动方 (contact_id = absorbed_id)
  → 更新为 contact_id = survivor_id
  → 若 survivor 已有对同一 related_contact_id + type 的关系，需去重

场景 2: absorbed 是被关联方 (related_contact_id = absorbed_id)
  → 更新为 related_contact_id = survivor_id
  → 同上需去重

场景 3: absorbed 与 survivor 之间的互相关系
  → 如 "survivor 是 absorbed 的父亲"，合并后变成 "survivor 是 survivor 的父亲"
  → 自引用无意义，应删除

场景 4: 其他 Contact 指向 absorbed 的关系
  → 所有 related_contact_id = absorbed_id 的记录都要改成 survivor_id
  → 场景 2 已覆盖此情况
```

**简化 SQL**：
```sql
-- 删除 survivor ↔ absorbed 之间的互相关系（场景 3）
DELETE FROM relationships
WHERE (contact_id = survivor_id AND related_contact_id = absorbed_id)
   OR (contact_id = absorbed_id AND related_contact_id = survivor_id);

-- 场景 1: absorbed 主动方 → survivor 主动方
UPDATE relationships SET contact_id = survivor_id WHERE contact_id = absorbed_id;

-- 场景 2: absorbed 被关联方 → survivor 被关联方
UPDATE relationships SET related_contact_id = survivor_id WHERE related_contact_id = absorbed_id;

-- 去重（上述更新后可能产生相同的关系）
DELETE r1 FROM relationships r1
INNER JOIN relationships r2
  ON r1.contact_id = r2.contact_id
  AND r1.related_contact_id = r2.related_contact_id
  AND r1.relationship_type_id = r2.relationship_type_id
  AND r1.id > r2.id;
```

### 5.5 非对称双端关系（Loans）

**事实**：`contact_loan` pivot 表结构（见 [迁移](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/database/migrations/2022_03_23_005751_create_loans_table.php#L34-L37)）：

| 列 | 说明 |
|----|------|
| `loan_id` | FK → loans |
| `loaner_id` | 借出方 (FK → contacts.id) |
| `loanee_id` | 借入方 (FK → contacts.id) |

两个 contact 外键都 `cascadeOnDelete`。

**现有代码中的操作**：
- [UpdateLoan.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageLoans/Services/UpdateLoan.php#L94-L109) 用事务包裹：先删所有关联，再用 `syncWithoutDetaching` 逐条重建。
- 注意：`loansAsLoaner()` 和 `loansAsLoanee()` 是两个独立的关系。

**合并迁移（推导）**：
```sql
-- absorbed 是借出方
UPDATE contact_loan SET loaner_id = survivor_id WHERE loaner_id = absorbed_id;

-- absorbed 是借入方
UPDATE contact_loan SET loanee_id = survivor_id WHERE loanee_id = absorbed_id;
```

⚠️ **自引用检测**：更新后如果一条记录的 `loaner_id == loanee_id`（自己借给自己），语义上不合理，应删除。

⚠️ **重复行检测**：同一 loan 的 loaner/loanee 组合可能重复。

### 5.6 其他外键引用

| 字段 | 表 | 迁移 | 处理 |
|------|-----|------|------|
| `paid_by_contact_id` | `life_events` | [create_life_events_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/database/migrations/2022_05_17_155546_create_life_events_table.php#L70) → `nullOnDelete` | `UPDATE life_events SET paid_by_contact_id = ? WHERE paid_by_contact_id = ?` |
| `loaner_id / loanee_id` | `gifts` | [create_gifts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/database/migrations/2022_06_09_173049_create_gifts_table.php#L53-L55) → `cascadeOnDelete` | 同上，双端更新 |

### 5.7 ContactFeedItem 的多态关联

**事实**：`contact_feed` 表有两套关联：
1. `contact_id` —— 直接 FK，指向所属 Contact
2. `feedable_id` + `feedable_type` —— MorphTo，指向被记录的对象（Note、Pet、Goal 等）

合并时：
- `contact_id` 按 HasMany 方式迁移
- `feedable` 多态不需要动（因为 Note、Pet 等本身的 id 没变，只是它们的 contact_id 变了）

---

## 6. 软删除与失败恢复

### 6.1 软删除的实际效果（事实）

重申一个关键点：**软删除不会触发数据库级联**。

| 操作 | 结果 |
|------|------|
| `$contact->delete()` | 软删除，只更新 `deleted_at` |
| 软删除后的关联数据 | 全部保留在数据库中 |
| 通过 Eloquent 查关联 | 查不到（因为 Contact 被软删除了，关联查询会走 JOIN 或 WHERE 条件过滤） |
| 直接 SQL 查关联表 | 能查到（孤儿数据） |
| `$contact->forceDelete()` | 硬删除，触发 `cascadeOnDelete`，所有关联数据被清掉 |

### 6.2 合并时的删除策略选择（推导）

有两种策略：

**策略 A：软删除 absorbed**
- 优点：可恢复，数据安全
- 缺点：关联数据成为孤儿，占用空间；未来硬删除时会触发 cascade 清掉所有已迁移的数据（如果有重复的话）
- **不推荐**：因为合并后 absorbed 的关联数据已经迁移到 survivor，软删除 absorbed 后这些数据还在但变成"双份"

**策略 B：先迁移再硬删除**
- 优点：干净，不留孤儿数据
- 缺点：不可恢复
- **推荐**：合并是用户主动操作，迁移成功后 absorbed 的数据已经全部转到 survivor，可以硬删除

**策略 C：迁移 + 软删除 + 延迟清理**
- 先迁移所有数据
- 软删除 absorbed（给用户反悔期）
- 一段时间后通过队列硬删除
- 最安全但最复杂

### 6.3 事务与失败恢复

**事实**：代码库中只有 [UpdateLoan.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Domains/Contact/ManageLoans/Services/UpdateLoan.php#L94) 使用了 `DB::transaction()`，大部分服务**没有事务包装**。

**事实**：`QueuableService` 的 `tries = 1`，失败不重试，`failed()` 方法是空的。见 [QueuableService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/54-monica/app/Services/QueuableService.php#L21-L55)

**推导**：合并操作涉及 20+ 张表的写操作，**必须用事务**包裹，否则中途失败会导致数据不一致。

```php
DB::transaction(function () use ($survivor, $absorbed) {
    // 1. 字段补白
    // 2. HasMany 迁移
    // 3. Pivot 表迁移
    // 4. Relationships 处理
    // 5. Loans 处理
    // 6. Files 处理
    // 7. 其他外键更新
    // 8. 删除 absorbed（硬删 or 软删）
    // 9. 刷新 survivor 的 last_updated_at
    // 10. 创建合并 feed item
});
```

**事务回滚 = 完全恢复**：因为所有修改都在事务内，任何一步异常都会回滚到事务前的状态，两个 Contact 都完好无损。

### 6.4 事务外的风险点

| 风险 | 场景 | 恢复策略 |
|------|------|---------|
| Scout 索引未更新 | 事务内数据变了但索引没刷新 | 事务成功后手动调用 `$survivor->searchable()` 和 `$survivor->notes->searchable()` |
| File 删除事件 | `File::deleted` 触发 `FileDeleted` 事件，可能有副作用 | 如果在事务内删除文件，事件在事务提交前触发；需确保事件监听器是幂等的 |
| CardDAV 同步推送 | 合并后 survivor 的 vcard 需重新推送 | 这是队列任务，不在事务内；推送失败会通过队列重试机制处理 |
| 内存中的 model 状态 | 事务回滚后，Eloquent model 的属性可能是脏的 | 回滚后需 `refresh()` 重新加载 |

### 6.5 幂等性设计

如果合并操作被重复执行（如队列重试），需保证幂等：

- **字段补白**：只有 survivor 字段为空时才写入，重复执行无副作用
- **HasMany 迁移**：先检查 `contact_id` 是否已指向 survivor，是则跳过
- **Pivot 迁移**：用 `syncWithoutDetaching` 思维，不会产生重复
- **删除 absorbed**：如果已经删了，再删一次会抛 `ModelNotFoundException`，需要捕获

---

## 7. 关联对象完整清单

| # | 类型 | 关联对象 | 迁移方式 | 去重 | 自引用风险 | 软删除 |
|---|------|---------|---------|------|-----------|--------|
| 1 | BelongsTo (FK) | File 头像 (`file_id`) | 补白逻辑 | ❌ | ❌ | ❌ |
| 2 | MorphMany (UUID) | File 附件 (`ufileable_id`) | UPDATE ufileable_id | ❌ | ❌ | ❌ |
| 3 | HasMany | ContactInformation | UPDATE contact_id | ✅ type+data | ❌ | ❌ |
| 4 | HasMany | Note | UPDATE contact_id | ❌ | ❌ | ❌ |
| 5 | HasMany | ContactImportantDate | UPDATE contact_id | ✅ type+date | ❌ | ✅ |
| 6 | HasMany | ContactReminder | UPDATE contact_id | ✅ label+date+type | ❌ | ❌ |
| 7 | HasMany | Call | UPDATE contact_id | ❌ | ❌ | ❌ |
| 8 | HasMany | Pet | UPDATE contact_id | ✅ name+category | ❌ | ❌ |
| 9 | HasMany | Goal | UPDATE contact_id | ✅ name | ❌ | ❌ |
| 10 | HasMany | ContactTask | UPDATE contact_id | ❌ | ❌ | ✅ |
| 11 | HasMany | MoodTrackingEvent | UPDATE contact_id | ❌ | ❌ | ❌ |
| 12 | HasMany | QuickFact | UPDATE contact_id | ✅ template_id | ❌ | ❌ |
| 13 | HasMany | ContactFeedItem | UPDATE contact_id | ❌ | ❌ | ❌ |
| 14 | BelongsToMany | Label (contact_label) | UPDATE pivot | sync | ❌ | - |
| 15 | BelongsToMany | Group (contact_group) | UPDATE pivot | sync + role | ❌ | - |
| 16 | BelongsToMany | Address (contact_address) | UPDATE pivot | 需去重 | ❌ | - |
| 17 | BelongsToMany | Post (contact_post) | UPDATE pivot | sync | ❌ | - |
| 18 | BelongsToMany | LifeEvent (participants) | UPDATE pivot | sync | ❌ | - |
| 19 | BelongsToMany | TimelineEvent (participants) | UPDATE pivot | sync | ❌ | - |
| 20 | BelongsToMany | LifeMetric | UPDATE pivot | sync | ❌ | - |
| 21 | BelongsToMany (双向) | Relationship (relationships) | 4 场景 SQL | ✅ | ✅ | - |
| 22 | BelongsToMany (双端) | Loan (contact_loan) | 双端 UPDATE | ✅ | ✅ | - |
| 23 | FK (其他表) | LifeEvent.paid_by_contact_id | UPDATE FK | ❌ | ❌ | - |
| 24 | FK (其他表) | Gift.loaner_id / loanee_id | 双端 UPDATE | ✅ | ✅ | - |

---

## 8. 完整合并流程（推导）

```
MergeContact::execute([
    'account_id',
    'vault_id',
    'author_id',
    'survivor_id',     // 保留的联系人 ID
    'absorbed_id',     // 被吸收的联系人 ID
])
```

### 执行步骤

1. **参数校验**（rules）
   - `survivor_id` / `absorbed_id` 都存在
   - 两者不相同

2. **权限校验**（permissions）
   - `author_must_belong_to_account`
   - `vault_must_belong_to_account`
   - `author_must_be_vault_editor`
   - `contact_must_belong_to_vault` —— 用 `survivor_id` 设置 `$this->contact`

3. **手动校验 absorbed**
   - `$absorbed = $this->vault->contacts()->findOrFail($absorbed_id)`
   - 检查 `can_be_deleted`（参照 DestroyContact 模式）

4. **事务内执行合并**
   - 字段补白（仅 survivor 为空的字段从 absorbed 拷贝）
   - 迁移所有 HasMany 关系
   - 迁移所有 BelongsToMany pivot 表
   - 处理 relationships 双向关系（4 场景）
   - 处理 loans 双端关系
   - 迁移 files（附件 + 头像）
   - 更新其他表的 FK（life_events.paid_by_contact_id 等）
   - 软删除/硬删除 absorbed
   - 刷新 survivor 的 `last_updated_at`
   - 创建 `ContactFeedItem` 记录合并动作

5. **事务后处理**
   - 刷新 Scout 搜索索引
   - 触发 CardDAV 同步（队列任务）
   - 触发 vCard 重新导出
