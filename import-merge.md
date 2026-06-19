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

适用：标签、重要日期（BDAY）

**逻辑**：以 key（标签名 / 日期字符串）为匹配依据，做集合运算。

```
toAdd    = vcard_items->diffKeys(local_items)    → 新增
toRemove = local_items->diffKeys(vcard_items)    → 删除
intersect = local_items->intersectByKeys(vcard)  → 可能更新
```

实现位置：
- [ImportLabels::import()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportLabels.php#L31-L53)
- [ImportImportantDates::import()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportImportantDates.php#L36-L64)

### 3.4 成员关系合并（ImportMembers）

[updateGroupMembers()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageGroups/Dav/ImportMembers.php#L99-L142)

这是一个特殊的合并策略——先按 `distant_uuid` 或 `id` 将现有成员分为 keep / remove 两组，再对 VCard 中的 MEMBER 列表筛选出不在 keep 中的新成员。

```
已有成员：
  keep   = distant_uuid 或 id 在 MEMBER 列表中的成员
  remove = 不在 MEMBER 列表中的成员 → RemoveContactFromGroup

新增成员：
  MEMBER 列表中不在 keep 里的 → 查找 Contact（先 distant_uuid 再 id）→ AddContactToGroup
```

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

```
AddressBookSynchronizer
    │
    ├─ SyncDAVBackend::getChangesForAddressBook()
    │   └─ 对比 sync_token 之后的 created_at / updated_at / deleted_at
    │       → { added: [...], modified: [...], deleted: [...] }
    │
    ▼
PrepareJobsContactPush
    │
    ├─ preparePushAddedContacts()
    │   └─ 所有新增联系人 → PushVCard(MODE_MATCH_ANY)
    │
    ├─ preparePushChangedContacts()
    │   ├─ 排除刚从远程拉取的联系人（避免循环同步）
    │   │   └─ 对比 refreshIds（本次拉取的联系人 UUID）
    │   └─ 有 distant_etag → PushVCard(MODE_MATCH_ETAG)
    │      无 distant_etag → PushVCard(MODE_MATCH_ANY)
    │
    └─ prepareDeletedContacts()
        └─ 所有已删除联系人 → DeleteVCard
    │
    ▼
PushVCard
    │
    ├─ PUT 请求上传 VCard
    │   ├─ MODE_MATCH_ETAG → If-Match: <etag>   （乐观锁，仅当远程版本一致时更新）
    │   ├─ MODE_MATCH_ANY  → If-Match: *         （只要远程存在就更新）
    │   └─ MODE_MATCH_NONE → 无 If-Match 头      （强制覆盖）
    │
    ├─ 412 Precondition Failed 时自动降级为 MODE_MATCH_NONE 重试一次
    │
    └─ 成功后更新本地 distant_etag
```

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
