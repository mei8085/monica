# vCard Export 字段映射规则

## 整体架构

vCard 导出采用**策略模式**，由核心服务类 [ExportVCard](app/Domains/Contact/Dav/Services/ExportVCard.php) 统一调度，根据资源类型（Contact 或 Group）自动发现并执行对应的导出器（Exporter）。每个导出器负责映射一组相关字段。

### 核心流程

```
Contact/Group 模型 → ExportVCard::execute() → ExportVCard::export() → 遍历所有 Exporter → 生成 VCard 对象
```

1. [ExportVCard::execute()](app/Domains/Contact/Dav/Services/ExportVCard.php#L53-L73) — 入口方法，校验权限，获取 Contact 或 Group 对象，调用 `export()`
2. [ExportVCard::export()](app/Domains/Contact/Dav/Services/ExportVCard.php#L78-L107) — 核心导出逻辑：
   - 若资源已有 `vcard` 字段（之前序列化过的），通过 [ReadVObject](app/Domains/Contact/Dav/Services/ReadVObject.php) 反序列化已有 VCard
   - 若已有 VCard 缺少 `UID`，则补充
   - 若无已有 VCard，创建新 VCard 4.0 对象，初始化 `UID`、`SOURCE`、`VERSION`
   - 按顺序遍历所有匹配的导出器，每个导出器对 VCard 对象进行字段写入
3. 导出完成后，将 VCard 序列化写回资源的 `vcard` 字段

### 导出器发现与排序

[ExportVCard::exporters()](app/Domains/Contact/Dav/Services/ExportVCard.php#L132-L142) 方法负责发现和排序导出器：

- 通过 `subClasses(ExportVCardResource::class)` 反射获取所有实现了 [ExportVCardResource](app/Domains/Contact/Dav/ExportVCardResource.php) 接口的类
- 按 [Order](app/Domains/Contact/Dav/Order.php) 属性标注的整数值升序排列
- 按 `getType()` 返回值过滤，只保留与当前资源类型匹配的导出器
- 结果静态缓存（`self::$exporters`），避免重复反射

### 通用模式：先删后写

每个导出器在写入字段前，都先调用 `$vcard->remove('XXX')` 删除该字段已有的所有值，再重新写入。这是为了确保导出结果是当前数据的真实映射，而不是在旧数据上追加。

### 基类工具方法

[Exporter](app/Domains/Contact/Dav/Exporter.php) 提供两个工具方法：

| 方法 | 作用 |
|------|------|
| `escape($value)` | 将值转为字符串，若非空则 `trim()`，否则返回 `null`。用于 FN、N、ORG 等字段 |
| `formatValue($value)` | 将值中的 `\;` 还原为 `;`，用于读取已有 VCard 值时的反转义 |

---

## Contact 导出器映射表

以下为 Contact 类型资源的所有导出器，按 `#[Order]` 值排列：

### Order 1: ExportNames — 姓名与昵称

文件: [ExportNames](app/Domains/Contact/ManageContact/Dav/ExportNames.php)

| 数据库字段 | vCard 字段 | RFC 6350 章节 | 说明 |
|-----------|-----------|-------------|------|
| `contact.name` | `FN` | §6.2.1 | 全名（Formatted Name），转义后写入 |
| `contact.last_name` | `N[0]` | §6.2.2 | 姓氏（Family Name） |
| `contact.first_name` | `N[1]` | §6.2.2 | 名字（Given Name） |
| `contact.middle_name` | `N[2]` | §6.2.2 | 中间名（Additional Names） |
| `contact.nickname` | `NICKNAME` | §6.2.3 | 昵称，仅在非空时写入 |

> 注意：vCard `N` 结构标准为 `[姓氏, 名字, 中间名, 前缀, 后缀]`，此处只映射了前三个元素，前缀和后缀未使用。

### Order 10: ExportGender — 性别

文件: [ExportGender](app/Domains/Contact/ManageContact/Dav/ExportGender.php)

| 数据库字段 | vCard 字段 | RFC 6350 章节 | 说明 |
|-----------|-----------|-------------|------|
| `gender.type` | `GENDER` | §6.2.7 | 优先使用 `gender.type` 字段 |
| `gender.name`（回退） | `GENDER` | §6.2.7 | 当 `type` 为空时，根据 `name` 翻译值推断 |

**性别映射回退逻辑**：
1. 若 `gender` 关系为 `null`，不输出 `GENDER` 字段
2. 优先使用 `gender.type`（数据库中的 `type` 字段）
3. 若 `type` 为空，按 `name` 的翻译值映射：
   - `trans('Male')` → `M` ([Gender::MALE](app/Models/Gender.php#L19))
   - `trans('Female')` → `F` ([Gender::FEMALE](app/Models/Gender.php#L26))
   - 其他 → `O` ([Gender::OTHER](app/Models/Gender.php#L33))

### Order 20: ExportAddress — 地址

文件: [ExportAddress](app/Domains/Contact/ManageContact/Dav/ExportAddress.php)

| 数据库字段 | vCard ADR 元素索引 | RFC 6350 章节 | 说明 |
|-----------|-------------------|-------------|------|
| — | `ADR[0]` | §6.3.1 | 邮箱地址（PO Box），始终为空字符串 |
| `address.line_1` | `ADR[1]` | §6.3.1 | 扩展地址（如公寓号） |
| `address.line_2` | `ADR[2]` | §6.3.1 | 街道地址 |
| `address.city` | `ADR[3]` | §6.3.1 | 城市（Locality） |
| `address.province` | `ADR[4]` | §6.3.1 | 省/州（Region） |
| `address.postal_code` | `ADR[5]` | §6.3.1 | 邮政编码 |
| `address.country` | `ADR[6]` | §6.3.1 | 国家 |
| `address.addressType.type` | `ADR 参数 TYPE` | — | 地址类型（如 home、work） |

> 注意：只导出 `is_past_address = false` 的地址（非过去地址）。每个地址生成一条 `ADR` 记录。

### Order 40: ExportContactInformation — 联系方式

文件: [ExportContactInformation](app/Domains/Contact/ManageContactInformation/Dav/ExportContactInformation.php)

这是最复杂的导出器，根据 `contactInformationType.type` 的值，将不同的联系方式映射到不同的 vCard 字段：

| `contactInformationType.type` | vCard 字段 | RFC 章节 | 参数 | 说明 |
|------------------------------|-----------|---------|------|------|
| `email` | `EMAIL` | §6.4.2 | `TYPE`（kind 大写）, `PREF`（若 pref=true 则为 1） | 邮箱地址 |
| `phone` | `TEL` | §6.4.1 | `TYPE`（kind 大写）, `PREF`（若 pref=true 则为 1） | 电话号码 |
| `IMPP` | `IMPP` | RFC 4770 | `X-SERVICE-TYPE`（类型名称） | 即时通讯 |
| `X-SOCIAL-PROFILE` | `X-SOCIAL-PROFILE` | — | `TYPE`（类型名称）, `X-USER`（用户名） | 社交档案 |
| 其他非空 type | `URL` | — | `TYPE`（原始 type 值） | 值为 `type + data` 拼接 |

**联系方式参数说明**：
- `kind`：联系方式的子类型（如 mobile、work、home），大写后写入 `TYPE` 参数
- `pref`：布尔值，若为 `true` 则设置 `PREF=1`，表示首选联系方式
- `X-SOCIAL-PROFILE` 的值部分为空字符串，实际用户名放在 `X-USER` 参数中
- URL 类型会将 `contactInformationType.type`（协议/URL前缀）与 `data` 拼接

### Order 40: ExportWorkInformation — 工作信息

文件: [ExportWorkInformation](app/Domains/Contact/ManageContact/Dav/ExportWorkInformation.php)

| 数据库字段 | vCard 字段 | RFC 6350 章节 | 说明 |
|-----------|-----------|-------------|------|
| `company.name` | `ORG` | §6.6.4 | 组织名称，仅在 company 关系存在时写入 |
| `contact.job_position` | `TITLE` | §6.6.1 | 职位头衔，仅在非空时写入 |

### Order 40: ExportImportantDates — 重要日期

文件: [ExportImportantDates](app/Domains/Contact/ManageContactImportantDates/Dav/ExportImportantDates.php)

| 数据库字段 | vCard 字段 | RFC 6350 章节 | 说明 |
|-----------|-----------|-------------|------|
| 仅 `internal_type === 'birthdate'` 的日期 | `BDAY` | §6.2.5 | 生日 |

**日期格式化逻辑**（[getVCardDate()](app/Models/ContactImportantDate.php#L117-L128)）：
- 有年份：`YYYYMMDD` 或 `YYYYMM` 或 `YYYY`
- 无年份：`--MMDD` 或 `--MM`
- 月或日为空时用 `-` 占位

> 注意：只有 `internal_type` 为 `birthdate` 的重要日期才会映射到 `BDAY`。其他类型的重要日期（如纪念日）**不会**导出到 vCard 中。

### Order 40: ExportLabels — 标签

文件: [ExportLabels](app/Domains/Contact/ManageContact/Dav/ExportLabels.php)

| 数据库字段 | vCard 字段 | RFC 6350 章节 | 说明 |
|-----------|-----------|-------------|------|
| `labels[].name` | `CATEGORIES` | §6.7.1 | 所有标签名以数组形式写入，仅在标签数 > 0 时输出 |

### Order 1000: ExportTimestamp — 时间戳

文件: [ExportTimestamp](app/Domains/Contact/ManageContact/Dav/ExportTimestamp.php)

| 数据库字段 | vCard 字段 | RFC 6350 章节 | 说明 |
|-----------|-----------|-------------|------|
| `contact.updated_at` | `REV` | §6.7.4 | 最后修订时间，格式 `Ymd\THis\Z` |

---

## Group 导出器映射表

以下为 Group 类型资源的所有导出器：

### Order 1: ExportKind — 类型标识

文件: [ExportKind](app/Domains/Contact/ManageGroups/Dav/ExportKind.php)

| 条件 | vCard 字段 | 值 | 说明 |
|------|-----------|----|------|
| 已有 `X-ADDRESSBOOKSERVER-KIND` | `X-ADDRESSBOOKSERVER-KIND` | `group` | 兼容 CardDAV 扩展格式 |
| 否则 | `KIND` | `group` | 标准 RFC 6350 格式 |

> 优先使用 `X-ADDRESSBOOKSERVER-KIND` 是为了兼容某些 CardDAV 服务器（如 macOS AddressBook Server）的扩展协议。

### Order 10: ExportNames — 组名

文件: [ExportNames (Group)](app/Domains/Contact/ManageGroups/Dav/ExportNames.php)

| 数据库字段 | vCard 字段 | 说明 |
|-----------|-----------|------|
| `group.name` | `FN` | 组全名 |
| `group.name` | `N[0]` | 结构化姓名（仅姓氏位填入组名） |

### Order 20: ExportMembers — 成员

文件: [ExportMembers](app/Domains/Contact/ManageGroups/Dav/ExportMembers.php)

| 数据库字段 | vCard 字段 | 说明 |
|-----------|-----------|------|
| `contacts[].distant_uuid` ?? `contacts[].id` | `MEMBER` 或 `X-ADDRESSBOOKSERVER-MEMBER` | 组内成员的 UUID 列表 |

**成员导出逻辑**：
- 若 VCard 中存在 `X-ADDRESSBOOKSERVER-KIND`，使用 `X-ADDRESSBOOKSERVER-MEMBER` 字段；否则使用标准 `MEMBER` 字段
- 优先使用 `distant_uuid`（来自 CardDAV 同步的远端 ID），若无则使用本地 `id`
- 采用增量更新策略：对比当前 VCard 中已有的成员和新成员列表，只添加新增的、只删除移除的

---

## 初始化字段（非导出器生成）

在 [ExportVCard::export()](app/Domains/Contact/Dav/Services/ExportVCard.php#L91-L97) 中，新建 VCard 时设置的基础字段：

| vCard 字段 | 值来源 | 说明 |
|-----------|--------|------|
| `UID` | `resource.uuid ?? resource.id` | 唯一标识符，优先 uuid |
| `SOURCE` | `route('contact.show', ...)` 或 `route('group.show', ...)` | 联系人/组的 URL |
| `VERSION` | `4.0` | vCard 版本固定为 4.0 |

若资源已有 `vcard` 字段，则从已有数据反序列化，仅补充缺失的 `UID`。

---

## 完整映射速查表

### Contact → vCard

| Monica 数据 | vCard 字段 | 导出器 | Order |
|------------|-----------|--------|-------|
| `uuid` / `id` | `UID` | ExportVCard (初始化) | — |
| 联系人 URL | `SOURCE` | ExportVCard (初始化) | — |
| — | `VERSION` | ExportVCard (初始化) | — |
| `name` | `FN` | ExportNames | 1 |
| `last_name` | `N[0]` | ExportNames | 1 |
| `first_name` | `N[1]` | ExportNames | 1 |
| `middle_name` | `N[2]` | ExportNames | 1 |
| `nickname` | `NICKNAME` | ExportNames | 1 |
| `gender.type` / `gender.name` | `GENDER` | ExportGender | 10 |
| 地址（非过去） | `ADR` | ExportAddress | 20 |
| 邮箱 | `EMAIL` | ExportContactInformation | 40 |
| 电话 | `TEL` | ExportContactInformation | 40 |
| 社交档案 | `X-SOCIAL-PROFILE` | ExportContactInformation | 40 |
| 即时通讯 | `IMPP` | ExportContactInformation | 40 |
| 其他 URL | `URL` | ExportContactInformation | 40 |
| 公司名 | `ORG` | ExportWorkInformation | 40 |
| 职位 | `TITLE` | ExportWorkInformation | 40 |
| 生日 | `BDAY` | ExportImportantDates | 40 |
| 标签 | `CATEGORIES` | ExportLabels | 40 |
| `updated_at` | `REV` | ExportTimestamp | 1000 |

### Group → vCard

| Monica 数据 | vCard 字段 | 导出器 | Order |
|------------|-----------|--------|-------|
| `uuid` / `id` | `UID` | ExportVCard (初始化) | — |
| 组 URL | `SOURCE` | ExportVCard (初始化) | — |
| — | `VERSION` | ExportVCard (初始化) | — |
| `group` | `KIND` / `X-ADDRESSBOOKSERVER-KIND` | ExportKind | 1 |
| `name` | `FN` / `N` | ExportNames | 10 |
| 成员 UUID | `MEMBER` / `X-ADDRESSBOOKSERVER-MEMBER` | ExportMembers | 20 |

---

## 关键设计要点

1. **先删后写**：每个导出器在写入前先清除对应字段的所有已有值，确保导出结果反映当前状态，避免重复或残留
2. **增量 VCard**：若资源已有序列化的 VCard，导出时基于已有 VCard 进行更新而非重建，保留可能存在的自定义字段
3. **Order 控制顺序**：`#[Order]` 属性控制导出器执行顺序，数值越小越先执行。Timestamp（1000）最后执行确保 `REV` 反映最终状态
4. **非生日日期丢失**：只有 `internal_type === 'birthdate'` 的重要日期映射到 `BDAY`，其他日期类型（纪念日等）不会出现在 vCard 中
5. **地址过滤**：只导出当前地址（`is_past_address = false`），过去地址被排除
6. **性别回退**：`gender.type` 优先，若为空则根据翻译后的 `gender.name` 推断标准值
7. **成员增量同步**：Group 成员导出采用增量对比策略，只增删差异部分
8. **社交档案特殊处理**：`X-SOCIAL-PROFILE` 值部分为空字符串，用户名放在 `X-USER` 参数中
9. **URL 拼接**：无法识别为已知类型的联系方式，将 `type`（协议前缀）与 `data` 拼接后作为 `URL` 输出
