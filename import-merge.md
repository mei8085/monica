# 联系人导入与重复合并路径

本文档梳理 Monica 系统中联系人导入后的字段映射、重复判断和合并任务三者的协作关系。

## 整体架构概览

```
VCard 输入
    │
    ▼
┌─────────────────────────────────────────────────┐
│              ImportVCard 核心服务                │
│  (app/Domains/Contact/Dav/Services/ImportVCard) │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
        按 Order 排序的导入器链
                    │
    ┌───────────────┼───────────────┐
    ▼               ▼               ▼
 ImportContact  ImportAddress  ImportLabels ...
 (字段映射)     (字段映射)     (字段映射)
    │               │               │
    └───────────────┼───────────────┘
                    │
                    ▼
          重复判断 (getExisting*)
                    │
                    ▼
          合并逻辑 (各导入器自行实现)
                    │
                    ▼
          持久化到数据库
```

## 一、字段映射

字段映射由多个 `Import*` 类共同完成，每个类负责一组 VCard 字段到系统内部数据结构的映射。所有导入器都实现 `ImportVCardResource` 接口，通过 `Order` 属性控制执行顺序。

### 核心入口：ImportVCard 服务

[ImportVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Services/ImportVCard.php)

- 自动发现所有 `ImportVCardResource` 的子类
- 按 `Order` 注解排序（数字越小越先执行）
- 依次调用每个导入器的 `handle()` → `can()` → `import()` 方法
- 支持两种行为模式：
  - `BEHAVIOUR_ADD`：添加模式（用于全新导入）
  - `BEHAVIOUR_REPLACE`：替换模式（用于同步更新）

### 主要导入器列表

| 顺序 | 导入器类 | 负责映射的 VCard 字段 | 对应系统实体 |
|------|---------|---------------------|------------|
| 1 | [ImportGroup](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageGroups/Dav/ImportGroup.php) | KIND:group, N, FN, UID | Group |
| 1 | [ImportContact](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php) | KIND:individual, N, FN, NICKNAME, GENDER, UID | Contact |
| 10 | [ImportMembers](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageGroups/Dav/ImportMembers.php) | MEMBER | Group 成员关系 |
| 40 | [ImportAddress](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportAddress.php) | ADR | Address |
| 40 | [ImportLabels](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportLabels.php) | CATEGORIES | Label |
| 40 | [ImportContactInformation](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php) | EMAIL, TEL, IMPP, URL, X-SOCIAL-PROFILE | ContactInformation |
| 40 | [ImportImportantDates](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportImportantDates.php) | BDAY | ContactImportantDate |
| - | [ImportContactTask](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageTasks/Dav/ImportContactTask.php) | VTODO | ContactTask |

### 字段映射示例

#### 1. 姓名字段映射（ImportContact）

VCard 的姓名字段有三种来源，优先级从高到低：
1. **N 字段**（结构化姓名）：`N:姓;名;中间名;前缀;后缀`
2. **FN 字段**（格式化全名）：按用户的 `name_order` 偏好拆分
3. **NICKNAME 字段**（昵称）：直接作为 first_name

#### 2. 联系方式映射（ImportContactInformation）

VCard 属性 → 系统类型映射：
- `EMAIL` → `email` 类型
- `TEL` → `phone` 类型
- `IMPP` → 即时通讯（带 `X-SERVICE-TYPE` 参数区分具体服务）
- `X-SOCIAL-PROFILE` → 社交账号（带 `TYPE` 参数区分平台）
- `URL` → 网址

## 二、重复判断

重复判断在导入流程的早期执行，用于确定当前 VCard 对应的是已有联系人还是新联系人。核心逻辑在各个导入器的 `getExisting*` 方法中。

### 联系人重复判断（ImportContact）

[getExistingContact()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L80-L111)

判断顺序（找到即返回，不再继续）：

1. **通过 URI 查找**（CardDAV 协议层）
   - 调用 `CardDAVBackend::getObject(vault_id, uri)`
   - 适用于已知 DAV URI 的场景（如 CardDAV 同步）

2. **通过 distant_uri 查找**
   - `Contact::firstWhere(['vault_id', 'distant_uri' => uri])`
   - 远程服务器的 URI 作为唯一标识

3. **通过 UID 查找**
   - 从 VCard 的 `UID` 字段提取 UUID
   - 如果是有效 UUID，直接用 `id` 字段查找
   - `Contact::firstWhere(['vault_id', 'id' => contactId])`

### 群组重复判断（ImportGroup）

[getExistingGroup()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageGroups/Dav/ImportGroup.php#L67-L90)

判断顺序：

1. **通过 distant_uri 查找**
2. **通过 distant_uuid 查找**（VCard 的 UID 字段）

### 关键标识字段

Contact 模型中有三个与远程同步相关的字段：

| 字段 | 说明 |
|------|------|
| `distant_uuid` | 远程服务器的唯一标识（VCard 的 UID） |
| `distant_uri` | 远程服务器上的资源路径 |
| `distant_etag` | 远程资源的 ETag，用于乐观并发控制 |

## 三、合并任务

当重复判断找到已有联系人后，进入合并流程。合并采用 **BEHAVIOUR_REPLACE** 模式，由各个导入器自行实现合并策略。

### 合并策略分类

#### 1. 整体替换策略（基本信息）

适用于联系人基本信息（姓名、性别等）。

**实现位置**：[ImportContact::import()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php#L44-L78)

逻辑：
- 获取现有联系人数据
- 用 VCard 数据构建新数据数组
- 比较 `$data !== $original`，有变化则调用 `UpdateContact` 服务

#### 2. 索引对齐策略（有序列表）

适用于地址、联系方式等有序列表数据。

**实现位置**：
- [ImportAddress::import()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportAddress.php#L37-L62)
- [ImportContactInformation::importContacts()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php#L76-L108)

逻辑（按位置一一对应）：
```
for i from 0 to max(vcard_count, local_count):
    if i < vcard_count and i < local_count:
        更新第 i 条
    elif i < vcard_count:
        新增第 i 条
    else:
        删除第 i 条
```

#### 3. 集合差集策略（无序集合）

适用于标签、重要日期等无序集合数据。

**实现位置**：
- [ImportLabels::import()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportLabels.php#L31-L53)
- [ImportImportantDates::import()](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportImportantDates.php#L36-L64)

逻辑：
- `toAdd = vcard_items - local_items` → 新增
- `toRemove = local_items - vcard_items` → 删除
- `intersect` → 可能需要更新

## 四、三者协作完整流程

### 场景一：CardDAV 远程同步（拉取）

```
AddressBookSynchronizer
    │
    ├─ 获取远程变化（sync-collection 或 PROPFIND）
    │
    ├─ PrepareJobsContactUpdater
    │   └─ 生成 GetVCard / GetMultipleVCard Job
    │
    ▼
GetVCard Job
    │
    ├─ GET 请求下载 VCard
    │
    ▼
UpdateVCard Job
    │
    ├─ 调用 ImportVCard 服务
    │   ├─ behaviour = BEHAVIOUR_REPLACE
    │   ├─ external = true
    │   └─ 传入 uri、etag
    │
    ▼
ImportVCard 服务
    │
    ├─ 1. 发现并排序所有导入器
    │
    ├─ 2. ImportContact（Order=1）
    │   ├─ handle() → 检查 KIND 是否为 individual
    │   ├─ can() → 检查是否有姓名信息
    │   └─ import()
    │       ├─ 重复判断：getExistingContact()
    │       │   ├─ 通过 uri 查找
    │       │   ├─ 通过 distant_uri 查找
    │       │   └─ 通过 UID 查找
    │       ├─ 字段映射：importNames(), importGender()
    │       └─ 合并：存在则 UpdateContact，不存在则 CreateContact
    │
    ├─ 3. ImportAddress（Order=40）
    │   ├─ 字段映射：ADR → Address
    │   └─ 合并：索引对齐策略
    │
    ├─ 4. ImportLabels（Order=40）
    │   ├─ 字段映射：CATEGORIES → Label
    │   └─ 合并：集合差集策略
    │
    ├─ 5. ImportContactInformation（Order=40）
    │   ├─ 字段映射：EMAIL/TEL/... → ContactInformation
    │   └─ 合并：索引对齐策略
    │
    ├─ 6. ImportImportantDates（Order=40）
    │   ├─ 字段映射：BDAY → ContactImportantDate
    │   └─ 合并：集合差集策略
    │
    └─ ... 其他导入器
```

### 场景二：本地修改推送（CardDAV 上传）

```
AddressBookSynchronizer
    │
    ├─ 获取本地变化（SyncDAVBackend::getChangesForAddressBook）
    │
    ├─ PrepareJobsContactPush
    │   ├─ 过滤刚从远程拉取的联系人（避免循环同步）
    │   └─ 生成 PushVCard / DeleteVCard Job
    │
    ▼
PushVCard Job
    │
    ├─ PUT 请求上传 VCard
    ├─ 使用 If-Match 头进行乐观并发控制
    │   ├─ MODE_MATCH_ETAG：匹配指定 ETag（更新已有）
    │   ├─ MODE_MATCH_ANY：匹配任意 ETag（新建远程）
    │   └─ MODE_MATCH_NONE：不匹配（强制覆盖）
    │
    └─ 更新本地 distant_etag
```

## 五、关键代码文件索引

### 核心服务
- [ImportVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Services/ImportVCard.php) - 导入核心服务
- [Importer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Importer.php) - 导入器基类
- [ImportVCardResource.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/ImportVCardResource.php) - 导入器接口
- [Order.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Order.php) - 执行顺序注解

### 联系人导入器
- [ImportContact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportContact.php) - 基本信息
- [ImportAddress.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportAddress.php) - 地址
- [ImportLabels.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContact/Dav/ImportLabels.php) - 标签
- [ImportContactInformation.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php) - 联系方式
- [ImportImportantDates.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/ManageContactImportantDates/Dav/ImportImportantDates.php) - 重要日期

### DavClient 同步
- [AddressBookSynchronizer.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Services/Utils/AddressBookSynchronizer.php) - 地址簿同步器
- [PrepareJobsContactUpdater.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Services/Utils/PrepareJobsContactUpdater.php) - 拉取任务准备
- [PrepareJobsContactPush.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Services/Utils/PrepareJobsContactPush.php) - 推送任务准备
- [GetVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Jobs/GetVCard.php) - 拉取 VCard Job
- [PushVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/DavClient/Jobs/PushVCard.php) - 推送 VCard Job
- [UpdateVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/Jobs/UpdateVCard.php) - 更新本地 VCard Job

### 模型
- [Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Models/Contact.php) - 联系人模型
- [VCardResource.php](file:///d:/fz/0601-2/solo-dogfeeding/code/50-monica/app/Domains/Contact/Dav/VCardResource.php) - VCard 资源基类
