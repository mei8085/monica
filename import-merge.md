# 联系人导入与重复合并路径

本文档梳理 Monica 系统中联系人导入后的字段映射、重复识别和合并策略三者的协作关系，以及 CardDAV / CalDAV 拉取推送链路。

---

## 〇、两套导入体系

Monica 的联系人相关数据导入分为 **VCard（CardDAV）** 和 **VCalendar（CalDAV）** 两套完全独立的管道。它们有各自的入口服务、接口、基类和导入器集合，互不交叉。

| | VCard 管道（CardDAV） | VCalendar 管道（CalDAV） |
|---|---|---|
| **入口服务** | [ImportVCard](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Services/ImportVCard.php) | [ImportVCalendar](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Services/ImportVCalendar.php) |
| **导入器接口** | [ImportVCardResource](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/ImportVCardResource.php) | [ImportVCalendarResource](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/ImportVCalendarResource.php) |
| **导入器基类** | [Importer](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Importer.php) | [VCalendarImporter](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/VCalendarImporter.php) |
| **资源基类** | [VCardResource](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/VCardResource.php)（Contact / Group） | [VCalendarResource](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/VCalendarResource.php)（ContactTask / ContactImportantDate） |
| **协议** | CardDAV（地址簿同步） | CalDAV（日历同步） |
| **文件扩展** | `.vcf` | `.ics` |
| **远程同步 Job** | [GetVCard](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Jobs/GetVCard.php) / [UpdateVCard](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Jobs/UpdateVCard.php) | 通过 CalDAVBackend 直接调用 [UpdateVCalendar](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Jobs/UpdateVCalendar.php) |

### 为什么任务和日历日期不在 VCard 管道里

VCard 标准只覆盖联系人名片信息（姓名、地址、邮箱等）。**任务（VTODO）和独立的重要日期事件（VEVENT）** 属于日历数据，走 CalDAV 协议。但 VCard 中嵌在联系人里的 `BDAY`（生日）则由 VCard 管道的 `ImportImportantDates` 处理——这就是两条管道的边界。

---

## 一、字段映射

### 1.1 VCard 管道导入器

所有导入器实现 `ImportVCardResource` 接口，由 `Order` 注解控制执行顺序。`ImportVCard` 在运行时通过 `subClasses()` 自动发现所有实现类，按 Order 排序后依次执行。

每个 VCard 只会匹配其中 **一组** 导入器：`KIND=individual` 走联系人链，`KIND=group` 走群组链。两者互斥。

#### 联系人链（KIND = individual）

当 `handle()` 返回 `kind($vcard) === 'individual'` 时，以下导入器参与处理：

| Order | 导入器 | 映射的 VCard 字段 | 系统实体 |
|-------|--------|-------------------|----------|
| 1 | [ImportContact](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php) | N, FN, NICKNAME, GENDER, UID | Contact |
| 40 | [ImportAddress](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportAddress.php) | ADR | Address |
| 40 | [ImportLabels](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportLabels.php) | CATEGORIES | Label |
| 40 | [ImportContactInformation](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php) | EMAIL, TEL, IMPP, URL, X-SOCIAL-PROFILE | ContactInformation |
| 40 | [ImportImportantDates](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportImportantDates.php) | BDAY | ContactImportantDate |

#### 群组链（KIND = group）

当 `handle()` 返回 `kind($vcard) === 'group'` 时：

| Order | 导入器 | 映射的 VCard 字段 | 系统实体 |
|-------|--------|-------------------|----------|
| 10 | [ImportGroup](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageGroups/Dav/ImportGroup.php) | N, FN, UID | Group |
| 11 | [ImportMembers](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageGroups/Dav/ImportMembers.php) | MEMBER, X-ADDRESSBOOKSERVER-MEMBER | Group ↔ Contact 关系 |

**关键依赖**：`ImportMembers`（Order=11）必须在 `ImportGroup`（Order=10）之后执行，因为成员导入的前提是群组已存在。`ImportMembers` 通过自己的 `getExistingGroup()` 重新查找群组——而不是依赖 `import()` 的 `$result` 参数——以确保即使群组是本次刚创建的也能找到。

### 1.2 VCalendar 管道导入器

同样互斥：有 `VTODO` 走任务链，有 `VEVENT` 走日期链。

| Order | 导入器 | 匹配条件 | 映射的 iCal 字段 | 系统实体 |
|-------|--------|----------|-----------------|----------|
| 1 | [ImportContactTask](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageTasks/Dav/ImportContactTask.php) | `$vcalendar->VTODO !== null` | SUMMARY, DESCRIPTION, DUE, STATUS, COMPLETED, DTSTAMP, UID | ContactTask |
| 1 | [ImportCalendarContactImportantDates](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportCalendarContactImportantDates.php) | `$vcalendar->VEVENT !== null` | SUMMARY, DTSTART, DTSTAMP, UID | ContactImportantDate |

### 1.3 姓名字段映射详解（ImportContact）

[importNames()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L134-L147) 的实际优先级：

```
hasFirstnameInN()  →  importNameFromN()      优先级最高
        ↓ 否则
hasFN()            →  importNameFromFN()     其次
        ↓ 否则
hasNICKNAME()      →  importNameFromNICKNAME()  兜底
```

注意 `hasFirstnameInN()` 的判断条件是 **N 字段的第 2 部分（名）非空**，不是 N 字段本身存在。即 `N:Doe;John;;;` 会匹配，而 `N:Doe;;;;` 不会。

`can()` 方法的检查顺序 `hasFN || hasNICKNAME || hasFirstnameInN` 只是「是否能导入」的充分条件判断，与 `importNames()` 的使用优先级是两回事。

#### 三种姓名映射的具体逻辑

| 来源 | 方法 | 映射逻辑 |
|------|------|----------|
| N 字段 | [importNameFromN()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L164-L179) | `N[0]→last_name`, `N[1]→first_name`, `N[2]→middle_name`；若 NICKNAME 存在则同步映射 |
| FN 字段 | [importNameFromFN()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L188-L211) | 按空白拆分为 2 段；根据 `author.name_order` 决定哪段是名/姓；若 NICKNAME 存在则同步映射 |
| NICKNAME | [importNameFromNICKNAME()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L181-L186) | 直接映射为 `first_name`，不设置 last_name |

### 1.4 联系方式映射（ImportContactInformation）

VCard 属性 → 系统类型映射，每个属性还可能带参数（`TYPE`、`X-SERVICE-TYPE`）用于区分子类型：

| VCard 属性 | 系统类型 | 参数 | 说明 |
|------------|----------|------|------|
| `EMAIL` | `email` | `TYPE` → kind | 邮箱 |
| `TEL` | `phone` | `TYPE` → kind | 电话 |
| `IMPP` | `IMPP` | `X-SERVICE-TYPE` → name_translation_key | 即时通讯 |
| `X-SOCIAL-PROFILE` | `X-SOCIAL-PROFILE` | `TYPE` → name_translation_key | 社交账号 |
| `URL` | `URL` | - | 网址 |

找不到匹配的 `ContactInformationType` 时会自动创建。

---

## 二、重复识别

重复识别在导入流程的早期执行，用于确定当前输入对应的是已有记录还是新记录。核心逻辑在各个导入器的 `getExisting*` 方法中，两套管道各有一套查找策略。

### 2.1 VCard 管道

#### 联系人（ImportContact）

[getExistingContact()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L80-L111)

查找顺序（找到即停止）：

1. **CardDAVBackend 按 URI 查找** — 仅当 `context.data.uri` 非空时
   - `$backend->getObject(vault_id, uri)`
   - 内部通过 `distant_uuid` / `uuid` / `id` 三级解码匹配
2. **distant_uri 字段查找** — 若上一步未找到
   - `Contact::firstWhere(['vault_id' => …, 'distant_uri' => uri])`
3. **UID 作为主键查找** — 若前两步均未找到
   - 从 VCard 的 `UID` 提取，验证是否有效 UUID
   - `Contact::firstWhere(['vault_id' => …, 'id' => uid])`

#### 群组（ImportGroup / ImportMembers）

[getExistingGroup()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageGroups/Dav/ImportGroup.php#L67-L90)

查找顺序：

1. **distant_uri 字段查找** — 仅当 `context.data.uri` 非空时
2. **distant_uuid 字段查找** — 从 VCard 的 `UID` 提取

注意：群组没有像联系人那样先走 CardDAVBackend 查找，而是直接查数据库。

### 2.2 VCalendar 管道

#### 任务（ImportContactTask）

[getExistingTask()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageTasks/Dav/ImportContactTask.php#L81-L114)

查找顺序：

1. **CalDAVTasks 按 URI 查找** — 仅当 `context.data.uri` 非空时
2. **distant_uri 字段查找** — 遍历 vault 下所有联系人的 tasks
3. **uuid 字段查找** — 从 VCalendar 的 `VTODO.UID` 提取，遍历 vault 下所有联系人的 tasks

#### 重要日期（ImportCalendarContactImportantDates）

[getExistingImportantDate()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportCalendarContactImportantDates.php#L79-L112)

查找顺序与任务相同，只是后端换成 `CalDAVDates`，实体换成 `ContactImportantDate`。

### 2.3 关键标识字段

以下四个模型都有 `distant_uuid`、`distant_uri`、`distant_etag` 三个远程同步字段：

| 模型 | 远程标识 | 远程路径 | 远程版本 |
|------|----------|----------|----------|
| Contact | `distant_uuid` | `distant_uri` | `distant_etag` |
| Group | `distant_uuid` | `distant_uri` | `distant_etag` |
| ContactTask | `distant_uuid` | `distant_uri` | `distant_etag` |
| ContactImportantDate | `distant_uuid` | `distant_uri` | `distant_etag` |

`distant_uuid` / `distant_uri` 在 `external=true`（远程同步场景）时写入；`distant_etag` 用于乐观并发控制。

---

## 三、合并策略

当重复识别找到已有记录后，各导入器自行决定合并方式。下面按策略类型分类。

### 3.1 整体替换（基本信息）

适用：Contact 基本信息、Group 基本信息、ContactTask、ContactImportantDate（VCalendar 管道）

**逻辑**：构建新数据数组 → 与原始数据比较 → 有变化则调用 Update 服务，无变化则跳过。

```php
$data = $this->getContactData($contact);   // 取现有数据
$original = $data;
$data = $this->importNames($data, $vcard); // 覆写映射字段
$data = $this->importGender($data, $vcard);

if ($contact === null) {
    $contact = CreateContact::execute($data);   // 新建
} elseif ($data !== $original) {
    $contact = UpdateContact::execute($data);   // 更新
}
```

实现位置：
- [ImportContact::import()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L44-L78)
- [ImportGroup::import()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageGroups/Dav/ImportGroup.php#L30-L62)
- [ImportContactTask::import()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageTasks/Dav/ImportContactTask.php#L35-L79)
- [ImportCalendarContactImportantDates::import()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportCalendarContactImportantDates.php#L35-L77)

### 3.2 索引对齐（有序列表）

适用：地址、联系方式

**逻辑**：将 VCard 中的条目与本地已有条目按下标一一对应，逐位增删改。

```
for i from 0 to max(vcard_count, local_count):
    if i < vcard_count and i < local_count:
        比较并更新第 i 条
    elif i < vcard_count:
        新增第 i 条
    else:
        删除第 i 条
```

实现位置：
- [ImportAddress::import()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportAddress.php#L37-L62)
- [ImportContactInformation::importContacts()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php#L76-L108)

`ImportContactInformation` 还会在更新前做值比较，只有 `value` 或 `kind` 真正变化时才调用 `UpdateContactInformation`。

### 3.3 集合差集（无序集合）

标签和重要日期虽然都用了 `diffKeys` / `intersectByKeys`，但**只有重要日期是真正的差集策略**，标签的实现由于 key 类型不同，实际上是全量删除再全量添加。

#### 重要日期（真正的差集）

[ImportImportantDates::import()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportImportantDates.php#L36-L64)

两边都用 **日期字符串** 作为 key：
- 本地：`$contactImportantDates` 通过 `mapWithKeys(fn ($d) => [$d->getVCardDate() => $d])` 构建
- VCard：`$bdays` 通过 `mapWithKeys(fn ($b) => [$b->getValue() => ...])` 构建

因此 `diffKeys` / `intersectByKeys` 能正确匹配：

```
toAdd     = bdays - contactImportantDates   （按日期字符串取差集）→ 新增
toRemove  = contactImportantDates - bdays   （按日期字符串取差集）→ 删除
intersect = contactImportantDates ∩ bdays   （按日期字符串取交集）→ 可能更新
```

交集部分会检查 `contactImportantDateType` 是否需要从 null 更新为 birthdate 类型。

#### 标签（全量删除再添加）

[ImportLabels::import()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportLabels.php#L31-L53)

两边的 key 类型不一致：
- 本地：`$labels` 通过 `mapWithKeys(fn ($label) => [$label->name => $label])` 构建，key 是 **标签名字符串**
- VCard：`$categories` 通过 `collect($categories->getParts())` 构建，key 是 **数字索引**（0, 1, 2...）

由于数字键和字符串键永远不会匹配，`diffKeys` 的结果是：
- `$toAdd = $categories->diffKeys($labels)` → 返回**所有** VCard 中的分类
- `$toRemove = $labels->diffKeys($categories)` → 返回**所有**本地标签

**实际效果**：每次导入都会先移除联系人身上所有现有的标签关联，再重新添加 VCard 中所有的标签。标签本身（Label 记录）不会被删除，因为 `addLabel()` 会先通过 `getLabel($name)` 查找已有的同名标签，找不到才创建。

### 3.4 成员关系合并（ImportMembers）

[updateGroupMembers()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageGroups/Dav/ImportMembers.php#L99-L142)

成员的 MEMBER 字段是一个 UUID 字符串列表，既可以是联系人的 `distant_uuid`（远程标识），也可以是 `id`（本地标识）——导出时 [ExportMembers](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageGroups/Dav/ExportMembers.php#L34-L57) 优先用 `distant_uuid`，没有时才回退到 `id`。因此匹配时需要**双标识同时考虑**。

#### Step 1：删除旧成员

对群组当前所有联系人，按 MEMBER 列表分为两组：

```php
$contacts = $group->contacts
    ->groupBy(fn (Contact $contact): string =>
        $members->contains($contact->distant_uuid) || $members->contains($contact->id)
            ? 'keep'
            : 'remove'
    );
```

- **keep**：`distant_uuid` 或 `id` 任意一个在 MEMBER 列表中 → 保留
- **remove**：两个都不在 MEMBER 列表中 → 通过 `RemoveContactFromGroup` 从群组移除（high 队列）

`RemoveContactFromGroup` 内部用 `$group->contacts()->detach([$contact_id])`，安全且幂等。

#### Step 2：添加新成员

从 MEMBER 列表中筛选出需要新增的成员，这里的判断条件用了 `||`（或）：

```php
$members->filter(fn (string $member): bool =>
    ! $keep->contains('distant_uuid', $member)
    || ! $keep->contains('id', $member)
)
```

根据德摩根定律 `!A || !B = !(A && B)`，意思是：只有当 keep 集合中存在一个 Contact，它的 `distant_uuid == member` **并且** `id == member` 时，才跳过。

**实际效果**：由于同一个 Contact 的 `distant_uuid` 和 `id` 几乎不可能相同（都是 UUID，但各自独立生成），**几乎所有 MEMBER 中的条目都会进入新增流程**。

#### Step 3：新增时的重复避免

进入新增流程并不代表会产生重复关系。每个 member 值都会按**先远程后本地**的顺序查找 Contact：

```php
$contact = Contact::firstWhere('distant_uuid', $member);
if ($contact === null) {
    $contact = Contact::find($member);
}
```

找不到对应的 Contact（例如 member 指向的联系人尚未同步）则直接跳过。

找到后调用 `AddContactToGroup`，其内部用 `syncWithoutDetaching` 实现**幂等**：

```php
$this->group->contacts()->syncWithoutDetaching([
    $this->contact->id => ['group_type_role_id' => ...],
]);
```

`syncWithoutDetaching` 在 pivot 关系已存在时不会重复插入，也不会抛异常。因此即使同一个成员在 MEMBER 中重复出现，或者因前述 `||` 条件而「误」入新增流程，也不会产生重复关系。

> 总结：成员合并不依赖 filter 条件做精确去重，而是靠 `AddContactToGroup` 服务层的 `syncWithoutDetaching` 提供最终幂等保障。

---

## 四、CardDAV 拉取/推送链路

### 4.1 拉取链路（远程 → 本地）

```
SynchronizeAddressBook
    │
    ▼
AddressBookSynchronizer
    │
    ├─ 获取远程变化
    │   ├─ 有 sync-collection 能力 → sync-collection 请求
    │   └─ 无 → PROPFIND 请求
    │
    ├─ 过滤：只保留 text/vcard 类型 + 新增或 etag 变化的卡片
    │
    ├─ 区分 updated / deleted
    │
    ▼
PrepareJobsContactUpdater
    │
    ├─ 支持 addressbook-multiget → GetMultipleVCard（批量下载）
    └─ 不支持 → GetVCard（逐个下载）
    │
    ▼
GetVCard / GetMultipleVCard
    │
    ├─ GET / addressbook-multiget 请求下载 VCard 原文
    │
    ▼
UpdateVCard（Job）
    │
    ├─ 调用 ImportVCard::execute()
    │   ├─ behaviour = BEHAVIOUR_REPLACE
    │   ├─ external = true
    │   └─ 传入 uri、etag
    │
    ├─ ImportVCard 内部流程（见下方详图）
    │
    └─ 比对新 etag 与预期 etag，不一致则记录警告
```

#### ImportVCard 内部流程（单个 VCard）

```
ImportVCard::execute(data)
    │
    ├─ ReadVObject 解析 VCard 字符串 → VCard 对象
    │
    ├─ canImportCurrentEntry(entry)
    │   ├─ 按 Order 排序所有 ImportVCardResource 实例
    │   ├─ filter: 只保留 handle(entry) === true 的导入器
    │   │   ├─ KIND=individual → ImportContact + 所有 Order=40 的 individual 导入器
    │   │   └─ KIND=group     → ImportGroup + ImportMembers
    │   └─ 遍历匹配的导入器，全部 can(entry) === true 才继续
    │
    └─ importEntry(entry)
        │
        │  以下以 KIND=individual 为例：
        │
        ├─ ImportContact (Order=1)
        │   ├─ getExistingContact() → 重复识别
        │   ├─ importUid() → 若非 external 且 UID 有效则设 id
        │   ├─ importNames() → 姓名映射（hasFirstnameInN > hasFN > hasNICKNAME）
        │   ├─ importGender() → 性别映射
        │   ├─ 不存在 → CreateContact / 存在且变化 → UpdateContact
        │   └─ external 时写入 distant_uuid / distant_etag / distant_uri
        │   返回 Contact 对象 → 作为 $result 传给后续导入器
        │
        ├─ ImportAddress (Order=40)
        │   ├─ 字段映射：ADR → line_1/line_2/city/province/postal_code/country
        │   └─ 索引对齐合并
        │
        ├─ ImportLabels (Order=40)
        │   ├─ 字段映射：CATEGORIES → Label.name
        │   └─ 集合差集合并
        │
        ├─ ImportContactInformation (Order=40)
        │   ├─ 字段映射：EMAIL/TEL/IMPP/URL/X-SOCIAL-PROFILE → ContactInformation
        │   └─ 索引对齐合并（同类型按位置对齐）
        │
        └─ ImportImportantDates (Order=40)
            ├─ 字段映射：BDAY → ContactImportantDate
            └─ 集合差集合并（以日期字符串为 key）

最后：将原始 VCard 序列化内容写入 contact.vcard 字段
```

### 4.2 推送链路（本地 → 远程）

推送链路有两套入口：**普通同步** 走 `PrepareJobsContactPush`，**强制同步（force sync）** 走 `PrepareJobsContactPushMissed`。两者的条件头策略和回写行为有重要差异。

#### 推送前统一获取卡片数据

无论哪种入口，推送前都通过 `CardDAVBackend::getCard()` / `prepareCard()` 获取卡片数据，返回数组包含两组 ETag：
- `etag` — **本地计算的**内容 ETag（`GetEtag` 服务基于当前 VCard 内容生成）
- `distant_etag` — **远程服务器上次返回的** ETag（保存在 `contact.distant_etag` 字段）

传给 `PushVCard` 构造函数的是 **`distant_etag`**（上一次从远程拿到的 ETag），不是本地 `etag`。

---

#### 4.2.1 普通同步：PrepareJobsContactPush

普通同步由 [AddressBookSynchronizer::sync()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Services/Utils/AddressBookSynchronizer.php#L66-L92) 触发，通过 `SyncDAVBackend::getChangesForAddressBook()` 对比 sync_token 之后的变更，分为 **新增**、**变更**、**删除** 三类。

```
AddressBookSynchronizer::sync()
    │
    ├─ SyncDAVBackend::getChangesForAddressBook(sync_token_id, depth=1)
    │   └─ 对比 created_at / updated_at / deleted_at
    │       → { added: [...], modified: [...], deleted: [...] }
    │
    ▼
PrepareJobsContactPush::execute(localChanges, changes)
    │
    ├─ preparePushAddedContacts(added)
    │   └─ 所有新增联系人
    │       → new PushVCard(subscription, uri, card['distant_etag'],
    │                         carddata, card['contact_id'])
    │       // 不传 mode → 默认 MODE_MATCH_NONE
    │
    ├─ preparePushChangedContacts(modified, changes)
    │   ├─ 排除刚从远程拉取的联系人（避免循环同步）
    │   │   refreshIds = changes.map(contactDto → backend.getUuid(uri))
    │   │   reject: uri 在 refreshIds 中
    │   │
    │   ├─ card['distant_etag'] !== null
    │   │   → PushVCard(..., MODE_MATCH_ETAG)
    │   │
    │   └─ card['distant_etag'] === null
    │       → PushVCard(..., MODE_MATCH_ANY)
    │
    └─ prepareDeletedContacts(deleted)
        └─ DeleteVCard
    │
    ▼
PushVCard Job → 推送 → 状态回写（见 4.2.3）
```

---

#### 4.2.2 强制同步：PrepareJobsContactPushMissed

强制同步由 [AddressBookSynchronizer::forcesync()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Services/Utils/AddressBookSynchronizer.php#L97-L130) 触发（`--force` 参数）。它不依赖本地 sync_token，而是做**全量对比**：把本地所有联系人与远程所有联系人比对，找出「本地有但远程没有/没匹配上」的漏推联系人。

```
AddressBookSynchronizer::forcesync()
    │
    ├─ $localContacts = backend.getObjects(vault_id)    // 本地所有
    │  $distContacts = getAllContactsEtag()              // 远程所有（addressbookQuery）
    │
    ├─ 拉取漏的：远程有本地没有的 → GetVCard（拉取）
    │
    ▼
PrepareJobsContactPushMissed::execute(localChanges, distContacts, localContacts)
    │
    ├─ 第 1 部分：正常变更推送（同普通同步）
    │   app(PrepareJobsContactPush)->execute(localChanges)
    │
    └─ 第 2 部分：漏推联系人推送
       preparePushMissedContacts(added, distContacts, localContacts)
       │
       ├─ distUuids = distContacts.map(uri → getUuid(uri))
       │  addedUuids = added.map(uri → getUuid(uri))
       │
       ├─ 筛选漏推：localContacts 中 reject
       │   distUuids 包含 resource.id   // 远程有的排除
       │   || addedUuids 包含 resource.id  // 已在新增列表的排除
       │
       └─ 每个漏推联系人 → PushVCard
           - 有 distant_etag   → MODE_MATCH_ANY
           - 无 distant_etag   → MODE_MATCH_NONE
```

**漏推联系人的判定**：本地存在，但既不在远程联系人列表里，也不在本次新增列表中的联系人。可能的原因：
- 之前推送失败
- 远程被他人删除但本地还保留
- sync_token 不准确导致 added 列表遗漏

**漏推的条件头策略比普通变更更「宽松」**：有 distant_etag 时用 `MODE_MATCH_ANY`（只要远程存在就更新），而不是 `MODE_MATCH_ETAG`（精确 etag 匹配）。因为既然已经是「漏推」，远程状态不确定，用更宽松的条件。

---

#### 4.2.3 三种推送场景的条件头对比

| | 新增联系人（普通同步） | 变更联系人（有 distant_etag） | 变更联系人（无 distant_etag） | 漏推联系人（有 distant_etag） | 漏推联系人（无 distant_etag） |
|---|---|---|---|---|---|
| **入口** | `preparePushAddedContacts` | `preparePushChangedContacts` | `preparePushChangedContacts` | `preparePushMissedContacts` | `preparePushMissedContacts` |
| **mode** | `MODE_MATCH_NONE`（默认 0） | `MODE_MATCH_ETAG`（1） | `MODE_MATCH_ANY`（2） | `MODE_MATCH_ANY`（2） | `MODE_MATCH_NONE`（0） |
| **If-Match 头** | **不发送** | `If-Match: <distant_etag>` | `If-Match: *` | `If-Match: *` | 不发送 |
| **etag 参数实际使用** | 不用 | 用作 If-Match 值 | 不用（写死 `*`） | 不用（写死 `*`） | 不用 |
| **远程行为** | PUT 直接创建/覆盖 | 仅当远程 ETag 匹配时更新 | 只要远程存在就更新 | 只要远程存在就更新 | 直接 PUT |
| **412 降级重试** | 不会触发 | 触发一次，降级 MODE_MATCH_NONE | 触发一次，降级 MODE_MATCH_NONE | 触发一次，降级 MODE_MATCH_NONE | 不会触发 |
| **推送成功后回写** | 不回写 distant_etag | 回写 distant_etag | 不回写 distant_etag | 不回写 distant_etag | 不回写 distant_etag |

> 注意：上表最后一行「回写」的差异不是 mode 决定的，而是由 `contact.distant_uri` 是否为 null 决定的——详见 4.2.5 节。

---

#### 4.2.4 PushVCard 执行流程与 412 降级

[PushVCard::pushDistant()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Jobs/PushVCard.php#L80-L105) 递归执行，`depth` 初始为 1，最大重试 1 次：

```
pushDistant(depth=1)
    │
    ├─ headers() 根据 mode 构造请求头
    │   ├─ MODE_MATCH_ETAG → ['If-Match' => $this->etag]
    │   ├─ MODE_MATCH_ANY  → ['If-Match' => '*']
    │   └─ MODE_MATCH_NONE → []
    │
    ├─ PUT 请求上传 VCard
    │
    ├─ 成功 → 返回响应头 ETag → 更新本地 distant_etag
    │
    └─ 失败（RequestException）
        ├─ status === 412 且 depth > 0
        │   → mode = MODE_MATCH_NONE
        │   → pushDistant(--depth)  递归重试一次
        │
        └─ 其他错误 → 记录日志 → fail() → 抛出
```

412 降级相当于乐观锁失败后的「强制覆盖」策略。降级后不再有 If-Match 约束，远程无论什么版本都会被覆盖。

---

#### 4.2.5 推送成功后的状态回写

**重要发现**：PushVCard 推送成功后**只更新 `distant_etag`**，而且**仅当 `contact.distant_uri !== null` 时才更新**：

```php
// PushVCard::run()
$etag = $this->pushDistant();

if ($contact->distant_uri !== null) {
    Contact::withoutTimestamps(function () use ($contact, $etag): void {
        $contact->distant_etag = empty($etag) ? null : $etag;
        $contact->save();
    });
}
```

- **不会设置** `distant_uri`
- **不会设置** `distant_uuid`
- **只会更新** `distant_etag`（前提是 distant_uri 已存在）

`distant_uri` 和 `distant_uuid` 只在**拉取时**设置——也就是 `ImportContact::import()` 中 `$this->context->external == true` 的分支里：

```php
// ImportContact::import()
if ($this->context->external && $contact->distant_uuid === null) {
    $contact->distant_uuid = $this->getUid($vcard);
    $contact->save();
}

return Contact::withoutTimestamps(function () use ($contact): Contact {
    $uri = Arr::get($this->context->data, 'uri');
    if ($this->context->external) {
        $contact->distant_etag = Arr::get($this->context->data, 'etag');
        $contact->distant_uri = $uri;
        $contact->save();
    }
    return $contact;
});
```

**实际影响**：本地新建的联系人推送到远程后，`distant_uri` 仍然是 null，`distant_etag` 也不会被更新。要等到**下一次拉取**（远程 → 本地）时，这三个远程同步字段才会被填充。

---

#### 4.2.6 三个远程同步字段的完整生命周期

| 字段 | 拉取时（ImportContact, external=true） | 推送成功后（PushVCard） |
|------|--------------------------------------|-----------------------|
| `distant_uri` | 设置为 DAV 资源 URI | **不修改**（只在 distant_uri 非空时才更新 etag） |
| `distant_uuid` | 设置为 VCard 的 UID | **不修改** |
| `distant_etag` | 设置为远程响应头 ETag | 若 distant_uri 非空，则更新为新的响应头 ETag |

```
初始状态（本地新建联系人）：
  distant_uri   = null
  distant_uuid  = null
  distant_etag  = null
        │
        ▼  PushVCard（新增推送，MODE_MATCH_NONE）
        ▼  推送成功
  distant_uri   = null    ← 不变！因为条件是 distant_uri !== null
  distant_uuid  = null    ← 不变！PushVCard 不碰这个字段
  distant_etag  = null    ← 不变！
        │
        ▼  下一次拉取（UpdateVCard → ImportVCard, external=true）
        ▼  ImportContact::import()
  distant_uri   = <远程 URI>
  distant_uuid  = <VCard UID>
  distant_etag  = <远程响应 ETag>
        │
        ▼  再次推送（变更推送，MODE_MATCH_ETAG）
        ▼  推送成功
  distant_uri   = 不变
  distant_uuid  = 不变
  distant_etag  = <新响应 ETag>   ← 只有这个更新
```

---

#### 4.2.7 避免循环同步的机制

普通同步的 `preparePushChangedContacts()` 会将本次拉取中远程已更新的联系人从推送列表排除：

```php
$refreshIds = $changes->map(fn (ContactDto $contact): string => $this->backend()->getUuid($contact->uri));

return $this->filterContacts($contacts)
    ->reject(fn (string $uri): bool => $refreshIds->contains($this->backend()->getUuid($uri)))
```

`$changes` 是本次远程拉取的变更列表。这样刚从远程拉下来的变更不会又被推回去，避免「拉 → 推 → 再拉 → 再推」的死循环。强制同步没有这个排除机制，因为强制同步的目的就是全量对齐。

### 4.3 CalDAV 链路（本地服务端）

CalDAV 不走 DavClient 的远程同步 Job 体系。当 CalDAV 客户端直接向 Monica 推送 iCal 数据时：

```
CalDAV 客户端 PUT 请求
    │
    ▼
CalDAVBackend::updateCalendarObject()
    │
    ▼
AbstractCalDAVBackend::updateOrCreateCalendarObject()
    │
    ▼
UpdateVCalendar Job
    │
    ▼
ImportVCalendar::execute()
    │
    ├─ VTODO 存在 → ImportContactTask
    │   ├─ getExistingTask() → 重复识别
    │   ├─ SUMMARY → label, DESCRIPTION → description, DUE → due_at
    │   └─ 整体替换合并
    │
    └─ VEVENT 存在 → ImportCalendarContactImportantDates
        ├─ getExistingImportantDate() → 重复识别
        ├─ SUMMARY → label, DTSTART → day/month/year
        └─ 整体替换合并
```

CalDAV 后端分为两个子后端，分别处理不同组件类型：

| 子后端 | 组件类型 | 资源 |
|--------|----------|------|
| [CalDAVTasks](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Web/Backend/CalDAV/CalDAVTasks.php) | VTODO | ContactTask |
| [CalDAVDates](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Web/Backend/CalDAV/CalDAVDates.php) | VEVENT | ContactImportantDate |

---

## 五、关键代码文件索引

### 两套管道的入口与基础设施

| 文件 | 作用 |
|------|------|
| [ImportVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Services/ImportVCard.php) | VCard 导入入口服务 |
| [ImportVCalendar.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Services/ImportVCalendar.php) | VCalendar 导入入口服务 |
| [Importer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Importer.php) | VCard 导入器基类 |
| [VCalendarImporter.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/VCalendarImporter.php) | VCalendar 导入器基类 |
| [ImportVCardResource.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/ImportVCardResource.php) | VCard 导入器接口 |
| [ImportVCalendarResource.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/ImportVCalendarResource.php) | VCalendar 导入器接口 |
| [Order.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Order.php) | 执行顺序注解 |

### VCard 管道导入器

| 文件 | Order | KIND | 职责 |
|------|-------|------|------|
| [ImportContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php) | 1 | individual | 姓名、性别、UID |
| [ImportGroup.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageGroups/Dav/ImportGroup.php) | 10 | group | 群组名称、UID |
| [ImportMembers.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageGroups/Dav/ImportMembers.php) | 11 | group | 群组成员关系 |
| [ImportAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportAddress.php) | 40 | individual | 地址 |
| [ImportLabels.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportLabels.php) | 40 | individual | 标签 |
| [ImportContactInformation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php) | 40 | individual | 联系方式 |
| [ImportImportantDates.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportImportantDates.php) | 40 | individual | 生日（BDAY） |

### VCalendar 管道导入器

| 文件 | Order | 匹配条件 | 职责 |
|------|-------|----------|------|
| [ImportContactTask.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageTasks/Dav/ImportContactTask.php) | 1 | VTODO | 任务 |
| [ImportCalendarContactImportantDates.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportCalendarContactImportantDates.php) | 1 | VEVENT | 重要日期事件 |

### DavClient 远程同步（CardDAV）

| 文件 | 职责 |
|------|------|
| [SynchronizeAddressBook.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Services/SynchronizeAddressBook.php) | 同步入口 |
| [AddressBookSynchronizer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Services/Utils/AddressBookSynchronizer.php) | 同步编排 |
| [PrepareJobsContactUpdater.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Services/Utils/PrepareJobsContactUpdater.php) | 拉取任务准备 |
| [PrepareJobsContactPush.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Services/Utils/PrepareJobsContactPush.php) | 推送任务准备 |
| [GetVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Jobs/GetVCard.php) | 单个拉取 |
| [GetMultipleVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Jobs/GetMultipleVCard.php) | 批量拉取 |
| [PushVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Jobs/PushVCard.php) | 推送 |
| [DeleteVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Jobs/DeleteVCard.php) | 远程删除 |
| [DeleteLocalVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Jobs/DeleteLocalVCard.php) | 本地删除 |
| [UpdateVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Jobs/UpdateVCard.php) | VCard 导入 Job |
| [UpdateVCalendar.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Jobs/UpdateVCalendar.php) | VCalendar 导入 Job |

### DAV 后端

| 文件 | 职责 |
|------|------|
| [CardDAVBackend.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php) | CardDAV 服务端后端 |
| [CalDAVBackend.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Web/Backend/CalDAV/CalDAVBackend.php) | CalDAV 服务端后端（路由层） |
| [CalDAVTasks.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Web/Backend/CalDAV/CalDAVTasks.php) | CalDAV 子后端：VTODO |
| [CalDAVDates.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Web/Backend/CalDAV/CalDAVDates.php) | CalDAV 子后端：VEVENT |
| [AbstractCalDAVBackend.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Web/Backend/CalDAV/AbstractCalDAVBackend.php) | CalDAV 子后端基类 |
| [SyncDAVBackend.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Web/Backend/SyncDAVBackend.php) | 同步令牌与变更检测 |

### 模型

| 文件 | 职责 |
|------|------|
| [Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Models/Contact.php) | 联系人 |
| [Group.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Models/Group.php) | 群组 |
| [ContactTask.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Models/ContactTask.php) | 任务 |
| [ContactImportantDate.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Models/ContactImportantDate.php) | 重要日期 |
| [VCardResource.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/VCardResource.php) | VCard 资源基类 |
| [VCalendarResource.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/VCalendarResource.php) | VCalendar 资源基类 |
