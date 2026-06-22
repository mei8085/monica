# vCard Export 字段映射规则

## 整体架构

vCard 导出采用**策略模式**，由核心服务类 [ExportVCard](app/Domains/Contact/Dav/Services/ExportVCard.php) 统一调度，根据资源类型（Contact 或 Group）自动发现并执行对应的导出器（Exporter）。每个导出器负责映射一组相关字段。

### 核心流程

```
Contact/Group 模型 → ExportVCard::execute() → ExportVCard::export() → 遍历所有 Exporter → 生成 VCard 对象 → withoutTimestamps 持久化
```

1. [ExportVCard::execute()](app/Domains/Contact/Dav/Services/ExportVCard.php#L53-L73) — 入口方法，先进行规则验证+权限校验（校验过程中挂载 author/vault/contact/group），获取 Contact 或 Group 对象后调用 `export()`，最后以 `withoutTimestamps` 闭包保存序列化结果
2. [ExportVCard::export()](app/Domains/Contact/Dav/Services/ExportVCard.php#L78-L107) — 核心导出逻辑：
   - 若资源已有 `vcard` 字段，调用 [ReadVObject](app/Domains/Contact/Dav/Services/ReadVObject.php) 反序列化；若解析成功但缺少 `UID`，按 **distant_uuid → uuid → id** 三级优先级补 UID
   - 若资源无 `vcard` 字段 **或 ReadVObject 返回 null（解析失败）**，创建全新 VCard 4.0 对象，按 **uuid → id** 两级优先级设 UID，并初始化 SOURCE、VERSION
   - 按 `#[Order]` 顺序遍历所有匹配类型的导出器，逐一写入字段
3. 导出完成后，以 `Model::withoutTimestamps` 闭包将 `$vcard->serialize()` 写回 `$obj->vcard` 字段并保存（不更新 updated_at / created_at）

### execute 的鉴权流程

[ExportVCard::rules()](app/Domains/Contact/Dav/Services/ExportVCard.php#L25-L34) 定义了字段校验：`account_id`、`author_id`、`vault_id` 必须存在，`contact_id` 与 `group_id` 二者选其一（`required_if`）。

[ExportVCard::permissions()](app/Domains/Contact/Dav/Services/ExportVCard.php#L39-L48) 声明了 5 个权限钩子，由 [BaseService::validateRules()](app/Services/BaseService.php#L96-L119) 按 `$permissionDependencies` 拓扑依赖顺序**逐条展开并执行**：

| 权限声明 | 展开执行顺序 | 副作用（挂载对象属性） |
|---------|-------------|---------------------|
| `author_must_belong_to_account` | 第 1 条 | `$this->author` = 从 account 下查找到的 User |
| `vault_must_belong_to_account` | 第 2 条 | `$this->vault` = 从 account 下查找到的 Vault |
| `author_must_be_in_vault` | 第 3 条（依赖前两条） | 校验 author 在 vault 中有 VIEW 权限（含以上） |
| `contact_must_belong_to_vault` | 第 4 条（依赖 vault/author） | **有 contact_id 时**：`$this->contact` = 从 vault 下查找到的 Contact，并再校验 `contact.vault_id` 一致性 |
| `group_must_belong_to_vault` | 第 4 条（依赖 vault/author） | **有 group_id 时**：`$this->group` = 从 vault 下查找到的 Group，并再校验 `group.vault_id` 一致性 |

> 因此后续 `execute()` 中 `$this->contact` 和 `$this->group` 并非从 execute 自行查询，而是**鉴权过程的副作用**——由 BaseService 在校验 `contact_must_belong_to_vault` / `group_must_belong_to_vault` 时已经赋值好。鉴权全部通过后才进入真正的导出逻辑。

### withoutTimestamps 持久化的意义

[ExportVCard::execute() L67-L70](app/Domains/Contact/Dav/Services/ExportVCard.php#L67-L70)：

```php
$obj::withoutTimestamps(function () use ($obj, $vcard): void {
    $obj->vcard = $vcard->serialize();
    $obj->save();
});
```

- **`withoutTimestamps`** 是 Eloquent 的静态方法，闭包内的所有 `save()` 操作不会触发 `updated_at` / `created_at` 的自动更新
- 导出 vCard 的目的是将当前数据缓存为序列化字符串，这个动作本身不应被视为"修改了联系人/组"
- 但注意 [ExportTimestamp](app/Domains/Contact/ManageContact/Dav/ExportTimestamp.php) 写入的 `REV` 字段**取的是导出前**的 `$resource->updated_at`（即真实的最后修改时间），`REV` 不会被 withoutTimestamps 影响，因为 REV 是写入 VCard 对象的内存属性，而 `withoutTimestamps` 只影响 Eloquent 层面的时间戳

### UID 两条路径的来源差异

UID 的赋值存在**两条路径，优先级不对称**：

#### 路径 A：补 UID（资源有 vcard 字段且解析成功，且原 VCard 无 UID）
代码位置：[ExportVCard::export() L86-L88](app/Domains/Contact/Dav/Services/ExportVCard.php#L86-L88)
```
优先级：distant_uuid → uuid → id（三级）
```

#### 路径 B：新建 UID（资源无 vcard 字段 或 ReadVObject 返回 null）
代码位置：[ExportVCard::export() L93-L94](app/Domains/Contact/Dav/Services/ExportVCard.php#L93-L94)
```
优先级：uuid → id（两级，缺少 distant_uuid）
```

> **偏差风险**：当资源的 `vcard` 字段损坏导致 ReadVObject 解析失败、或首次导出（vcard 字段为空）时，如果资源恰好设置了 `distant_uuid`（来自远端 CardDAV 同步），新建路径不会优先使用它——UID 的值会退化为 uuid 或 id，与补 UID 路径行为不一致。

### ReadVObject 返回 null 的路径

[ReadVObject::execute()](app/Domains/Contact/Dav/Services/ReadVObject.php#L34-L43) 结构：

```php
$this->validateRules($data);  // 先校验 entry 是 string 或 resource，不通过会抛 ValidationException（不被 try/catch 捕获）
try {
    return Reader::read($data['entry'], Reader::OPTION_FORGIVING + Reader::OPTION_IGNORE_INVALID_LINES);
} catch (ParseException $e) {
    return null;  // 唯一返回 null 的路径
}
```

- **validateRules 阶段**：校验失败抛 `ValidationException`，**不被 try 包裹**，会向上冒泡异常，**不会**走到 null 分支
- **try 阶段**：仅捕获 `ParseException`。即使启用了 `OPTION_FORGIVING`（宽容模式）和 `OPTION_IGNORE_INVALID_LINES`（忽略非法行），如果输入字符串完全损坏（不是 VObject 文档、字节流截断等），Sabre/VObject 仍可能抛 `ParseException`

**ReadVObject 返回 null 时的级联效果**：
`$vcard = (new ReadVObject)->execute(...)` 赋值为 null → `if ($vcard !== null && !$vcard->UID)` 判断为 false → 进入 `if (!isset($vcard))` 分支**重建全新 VCard** → 旧 vcard 中所有自定义字段、扩展属性全部丢失。

### 导出器发现与排序

[ExportVCard::exporters()](app/Domains/Contact/Dav/Services/ExportVCard.php#L132-L142)：

- 通过 `subClasses(ExportVCardResource::class)` 反射获取所有实现了 [ExportVCardResource](app/Domains/Contact/Dav/ExportVCardResource.php) 接口的类
- 按 [Order](app/Domains/Contact/Dav/Order.php) 属性标注的整数值升序排列
- 按 `getType()` 返回值过滤，只保留与当前资源类型匹配的导出器
- 结果静态缓存（`self::$exporters`），避免重复反射

### 通用模式：先删后写

每个导出器在写入字段前，都先调用 `$vcard->remove('XXX')` 删除该字段已有的所有值，再重新写入。这确保导出结果是当前数据的真实映射，而非在旧数据上追加。

### 基类工具方法

[Exporter](app/Domains/Contact/Dav/Exporter.php)：

| 方法 | 作用 |
|------|------|
| `escape($value)` | 将值转为字符串，若非空则 `trim()`，否则返回 `null`。用于 FN、N、ORG 等文本字段 |
| `formatValue($value)` | 将值中的 `\;` 还原为 `;`，用于读取已有 VCard 值时的反转义（仅 ExportMembers 使用） |

---

## Contact 导出器映射表

Contact 类型资源共有 **8 个** 导出器，按 `#[Order]` 值排列如下：

### Order 1: ExportNames — 姓名与昵称

文件: [ExportNames](app/Domains/Contact/ManageContact/Dav/ExportNames.php)

| 数据库字段 | vCard 字段 | RFC 6350 章节 | 说明 |
|-----------|-----------|-------------|------|
| `contact.name` | `FN` | §6.2.1 | 全名（Formatted Name），经过 `escape()` 处理 |
| `contact.last_name` | `N[0]` | §6.2.2 | 姓氏（Family Name） |
| `contact.first_name` | `N[1]` | §6.2.2 | 名字（Given Name） |
| `contact.middle_name` | `N[2]` | §6.2.2 | 中间名（Additional Names） |
| `contact.nickname` | `NICKNAME` | §6.2.3 | 昵称，仅在非空时写入 |

> 先删除 FN / N / NICKNAME 三字段。注意：vCard `N` 结构标准为 `[姓氏, 名字, 中间名, 前缀(Honorific Prefixes), 后缀(Honorific Suffixes)]`，此处只映射了前三个元素，前缀和后缀永远为空。

### Order 10: ExportGender — 性别

文件: [ExportGender](app/Domains/Contact/ManageContact/Dav/ExportGender.php)

| 数据库字段 | vCard 字段 | RFC 6350 章节 | 说明 |
|-----------|-----------|-------------|------|
| `gender.type` | `GENDER` | §6.2.7 | 优先使用 `gender.type` 字段 |
| `gender.name`（翻译值回退） | `GENDER` | §6.2.7 | 当 `type` 为空字符串时，根据 `name` 的**翻译结果**推断 |

**性别映射回退逻辑**：
1. 若 `gender` 关系本身为 `null`，先删 GENDER 后直接 return，**不输出 GENDER 字段**
2. 取 `$resource->gender->type` 作为优先值
3. 若 `type` 为空字符串，按 `$resource->gender->name` 的翻译值推断：
   - `trans('Male')` → `M`（[Gender::MALE](app/Models/Gender.php#L19)）
   - `trans('Female')` → `F`（[Gender::FEMALE](app/Models/Gender.php#L26)）
   - 其他任何翻译结果 → `O`（[Gender::OTHER](app/Models/Gender.php#L33)）

> 注意 `Gender` 类还有 `UNKNOWN = 'U'` 和 `NONE = 'N'` 常量，但回退逻辑永远不会产出这两个值。

### Order 20: ExportAddress — 地址（槽位语义易错点）

文件: [ExportAddress](app/Domains/Contact/ManageContact/Dav/ExportAddress.php)

RFC 6350 §6.3.1 规定 ADR 七元素顺序为：
`[PO Box, Extended Address, Street Address, Locality, Region, Postal Code, Country Name]`

Monica 的 [Address](app/Models/Address.php) 模型只有 `line_1` 和 `line_2` 两个命名不明确的文本字段。代码实际映射如下：

| ADR 索引 | RFC 语义 | Monica 字段 | 说明 |
|---------|---------|------------|------|
| `ADR[0]` | PO Box（邮政信箱） | `''`（空字符串） | Monica 无此字段，永远为空 |
| `ADR[1]` | Extended Address（扩展地址/公寓号/房间号） | `address.line_1` | **反直觉槽位**：用户普遍认为 line_1 = "第一行=主街道"，但代码将 line_1 填入了"次要补充地址"槽 |
| `ADR[2]` | Street Address（主街道地址） | `address.line_2` | **反直觉槽位**：line_2 反而被填入"主街道"槽 |
| `ADR[3]` | Locality（城市） | `address.city` | 正常映射 |
| `ADR[4]` | Region（省/州） | `address.province` | 正常映射 |
| `ADR[5]` | Postal Code（邮编） | `address.postal_code` | 正常映射 |
| `ADR[6]` | Country Name（国家） | `address.country` | 正常映射 |
| TYPE 参数 | Address Type | `address.addressType.type` | 如 home / work / other 等 |

> 只导出 `is_past_address = false` 的地址（通过中间表 `contact_address` 的 pivot 字段过滤），`is_past_address = true` 的地址不导出。每个地址产出一条独立 ADR 记录。

### Order 40: ExportContactInformation — 联系方式（一次清五字段 + 参数不对称）

文件: [ExportContactInformation](app/Domains/Contact/ManageContactInformation/Dav/ExportContactInformation.php)

这是最复杂的导出器，有两个重要特性需要特别注意。

#### 一次清五字段的副作用

[L32-L36](app/Domains/Contact/ManageContactInformation/Dav/ExportContactInformation.php#L32-L36) 无条件清空五个 vCard 字段：

```php
$vcard->remove('TEL');
$vcard->remove('EMAIL');
$vcard->remove('X-SOCIAL-PROFILE');
$vcard->remove('IMPP');
$vcard->remove('URL');
```

**副作用风险**：即使 Monica 数据库中当前没有某类联系方式（例如一条 URL 都没有），这行 `remove('URL')` 也会把旧 vcard 中可能存在的（例如由外部 CardDAV 客户端添加的）URL 全部删掉。清除的字段范围远大于"本导出器当前有数据要写"的范围。

#### 字段映射表（五类型参数不对称）

| `contactInformationType.type` | vCard 字段 | RFC 章节 | 使用参数 | `kind` 是否映射 TYPE | `pref` 是否映射 PREF |
|------------------------------|-----------|---------|---------|---------------------|---------------------|
| `email` | `EMAIL` | §6.4.2 | `TYPE` (kind 大写), `PREF=1` (若 pref=true) | ✅ 是 | ✅ 是 |
| `phone` | `TEL` | §6.4.1 | `TYPE` (kind 大写), `PREF=1` (若 pref=true) | ✅ 是 | ✅ 是 |
| `IMPP` | `IMPP` | RFC 4770 | `X-SERVICE-TYPE` (类型名称) | ❌ 否 | ❌ 否 |
| `X-SOCIAL-PROFILE` | `X-SOCIAL-PROFILE` | 非标准 | `TYPE` (类型名称), `X-USER` (用户名) | ❌ kind 被忽略 | ❌ 否 |
| 其他非空 type | `URL` | §6.7.8 | `TYPE` (原始 type 值)，值为 `escape(type + data)` | ❌ kind 被忽略 | ❌ 否 |

**参数不对称说明**：
1. **只有 email 和 phone 两种类型消费了 `kind` 和 `pref`** 字段；IMPP、X-SOCIAL-PROFILE、URL 三种类型即便是 ContactInformation 记录有 kind 或 pref=true，也不会体现在 vCard 参数中
2. **X-SOCIAL-PROFILE 的值位置特殊**：`$vcard->add('X-SOCIAL-PROFILE', '', ['TYPE' => ..., 'X-USER' => $contactInformation->data])` —— VALUE 部分写空字符串，实际用户名放在 `X-USER` 参数中。这与其他四种类型（直接把 data 写在 VALUE 部分）不同
3. **URL 分支的值拼接**：不是直接写 `data`，而是 `escape($type . $contactInformation->data)`，将 type（协议前缀/URL前缀）与 data 直接字符串拼接后作为 VALUE；同时再把原始 type 写进 TYPE 参数（出现两次）

### Order 40: ExportWorkInformation — 工作信息

文件: [ExportWorkInformation](app/Domains/Contact/ManageContact/Dav/ExportWorkInformation.php)

| 数据库字段 | vCard 字段 | RFC 6350 章节 | 说明 |
|-----------|-----------|-------------|------|
| `company.name` | `ORG` | §6.6.4 | 组织名称，仅在 `$resource->company` 关系存在（非 null）时写入，经过 `escape()` |
| `contact.job_position` | `TITLE` | §6.6.1 | 职位头衔，非空字符串时写入，经过 `escape()` |

> 先删除 ORG 和 TITLE。ORG 标准结构支持数组 `[组织名, 部门, 子部门...]`，此处只写入组织名字符串。

### Order 40: ExportImportantDates — 重要日期（getVCardDate 占位细节）

文件: [ExportImportantDates](app/Domains/Contact/ManageContactImportantDates/Dav/ExportImportantDates.php)

| 条件 | vCard 字段 | RFC 6350 章节 |
|------|-----------|-------------|
| 仅 `internal_type === 'birthdate'`（常量 `ContactImportantDate::TYPE_BIRTHDATE`） | `BDAY` | §6.2.5 |

非生日类型的重要日期（如纪念日、入职日等）**完全不导出**。

#### getVCardDate 月日占位算法

[ContactImportantDate::getVCardDate()](app/Models/ContactImportantDate.php#L117-L128)：

```php
$date = $this->year ? Str::padLeft((string) $this->year, 2, '0') : '--';
if ($this->month === null && $this->day === null) {
    return $date;
}
$date .= $this->month ? Str::padLeft((string) $this->month, 2, '0') : '-';
$date .= $this->day   ? Str::padLeft((string) $this->day,   2, '0') : '';
return $date;
```

**所有情况对照**：

| year | month | day | 输出 | 说明 |
|------|-------|-----|------|------|
| 2024 | 6 | 22 | `20240622` | 完整年月日，月日均 padLeft 两位 |
| 2024 | 6 | null | `202406` | 无日，结果 6 位 |
| 2024 | null | 22 | `2024-22` | 月占位为单 `-`，日 padLeft |
| 2024 | null | null | `2024` | 月日均空，提前 return |
| null | 6 | 22 | `--0622` | 年占位 `--`，月日均 padLeft（RFC 预期格式） |
| null | 6 | null | `--06` | 年占位 `--`，月 padLeft，日空拼空串 |
| null | null | 22 | `---22` | 年占位 `--` + 月占位 `-` + 日，共三条横杠 |
| null | null | null | `--` | 三者都空，提前 return |
| 5 | 3 | 8 | `050308` | 年只 padLeft 到 2 位（RFC 通常用 4 位表示完整年） |

> 注意：`Str::padLeft` 不截断。年份不会被格式化为 4 位宽度，只保证至少 2 位（year < 10 时前补 0）。另外输出全是**紧凑格式**（无横杠分隔的 YYMMDD 风格），而非 ISO 格式 `YYYY-MM-DD`。

### Order 40: ExportLabels — 标签

文件: [ExportLabels](app/Domains/Contact/ManageContact/Dav/ExportLabels.php)

| 数据库字段 | vCard 字段 | RFC 6350 章节 | 说明 |
|-----------|-----------|-------------|------|
| `labels[].name` | `CATEGORIES` | §6.7.1 | `$resource->labels->pluck('name')->toArray()`，所有标签名以数组形式一次性写入；仅当标签数量 > 0 时才 add |

### Order 1000: ExportTimestamp — 时间戳

文件: [ExportTimestamp (Contact)](app/Domains/Contact/ManageContact/Dav/ExportTimestamp.php)

| 数据库字段 | vCard 字段 | RFC 6350 章节 | 格式 |
|-----------|-----------|-------------|------|
| `contact.updated_at` | `REV` | §6.7.4 | `Ymd\THis\Z`（UTC 格式字符串，如 `20240622T034512Z`） |

> Order 1000 确保 REV 是最后一个被写入的字段（反映"导出前"的 Contact 修改时间，不因 withoutTimestamps save 而改变）。

---

## Group 导出器映射表

Group 类型资源共有 **4 个** 导出器（此前漏记了 ExportTimestamp）：

### Order 1: ExportKind — 类型标识

文件: [ExportKind](app/Domains/Contact/ManageGroups/Dav/ExportKind.php)

| 条件 | vCard 字段 | 值 | 说明 |
|------|-----------|----|------|
| 旧 vcard 中已有 `X-ADDRESSBOOKSERVER-KIND` | `X-ADDRESSBOOKSERVER-KIND` | `group` | 兼容 Apple/macOS CardDAV 扩展协议 |
| 否则 | `KIND` | `group` | 标准 RFC 6350 §6.1.4 KIND 字段 |

判断方式：`collect($vcard->select('X-ADDRESSBOOKSERVER-KIND'))->first()`。只要旧 vcard 里存在该字段（即便值是别的内容），也会走扩展路径重新写回 `group`。

### Order 10: ExportNames — 组名

文件: [ExportNames (Group)](app/Domains/Contact/ManageGroups/Dav/ExportNames.php)

| 数据库字段 | vCard 字段 | 说明 |
|-----------|-----------|------|
| `group.name` | `FN` | `escape()` 后写入 Formatted Name |
| `group.name` | `N[0]` | 组名写入 N 结构的"姓氏位"，其余 N 元素未设置 |

### Order 20: ExportMembers — 成员

文件: [ExportMembers](app/Domains/Contact/ManageGroups/Dav/ExportMembers.php)

| 条件 | vCard 字段 | 值来源 |
|------|-----------|--------|
| 旧 vcard 有 `X-ADDRESSBOOKSERVER-KIND` | `X-ADDRESSBOOKSERVER-MEMBER` | 每个成员 `contact.distant_uuid ?? contact.id`，排序后逐个对比 |
| 否则 | `MEMBER`（RFC §6.6.5） | 同上 |

增量同步算法：
1. 取出数据库中当前组成员的 UUID 列表并排序
2. 从现有 vcard 中取出已有的 MEMBER/X-ADDRESSBOOKSERVER-MEMBER 值，逐个 `formatValue()` 反转义
3. **新增**：凡数据库中有、vcard 中没有的 → `$vcard->add(...)`
4. **删除**：凡 vcard 中有、数据库中没有的 → `$vcard->remove($member)`

> 这里 `remove($member)` 传的是属性对象（不是属性名字符串），Sabre VObject 会移除这个具体属性节点。

### Order 1000: ExportTimestamp — 时间戳（Group 版，此前漏记）

文件: [ExportTimestamp (Group)](app/Domains/Contact/ManageGroups/Dav/ExportTimestamp.php)

| 数据库字段 | vCard 字段 | 格式 |
|-----------|-----------|------|
| `group.updated_at` | `REV` | `Ymd\THis\Z` |

与 Contact 版完全一致：先 `remove('REV')`，再写入 `$resource->updated_at` 的 UTC 格式字符串。Order 1000 保证最后执行。

---

## 初始化字段（非导出器生成）

[ExportVCard::export()](app/Domains/Contact/Dav/Services/ExportVCard.php#L81-L97) 在核心导出器运行前准备基础字段：

### 路径 A：资源有 vcard 字段 + ReadVObject 解析成功

- 直接反序列化得到的 VCard 对象继续在其上被导出器们修改
- 仅当 `!$vcard->UID`（原 VCard 中 UID 缺失）时补充 UID：
  ```
  distant_uuid → uuid → id（三级优先级）
  ```

### 路径 B：资源无 vcard 字段 或 ReadVObject 返回 null

创建全新 `Sabre\VObject\Component\VCard` 对象，构造时传入三个参数：

| vCard 字段 | 值来源 | 说明 |
|-----------|--------|------|
| `UID` | `resource.uuid ?? resource.id` | 仅两级优先级（**无 distant_uuid**） |
| `SOURCE` | `route('contact.show', ...)` / `route('group.show', ...)` | [getSource()](app/Domains/Contact/Dav/Services/ExportVCard.php#L109-L124) 根据资源类型生成 |
| `VERSION` | `'4.0'` | vCard 版本固定为 4.0 |

---

## 完整映射速查表

### Contact → vCard

| Monica 数据 | vCard 字段 | 导出器 | Order |
|------------|-----------|--------|-------|
| `uuid` / `id`（或补 UID 时 `distant_uuid`优先） | `UID` | ExportVCard 初始化 / 补 UID | — |
| 联系人 URL | `SOURCE` | ExportVCard 初始化 | — |
| — | `VERSION` | ExportVCard 初始化 | — |
| `name` | `FN` | ExportNames | 1 |
| `last_name` / `first_name` / `middle_name` | `N[0..2]` | ExportNames | 1 |
| `nickname` | `NICKNAME` | ExportNames | 1 |
| `gender.type` / `gender.name`翻译回退 | `GENDER` | ExportGender | 10 |
| `address.line_1`→ADR[1]（扩展槽）, `line_2`→ADR[2]（街道槽） | `ADR` | ExportAddress | 20 |
| 邮箱（email/phone 有 kind+pref，其他无） | `EMAIL` / `TEL` / `IMPP` / `X-SOCIAL-PROFILE` / `URL` | ExportContactInformation（一次清五字段） | 40 |
| 公司名 | `ORG` | ExportWorkInformation | 40 |
| 职位 | `TITLE` | ExportWorkInformation | 40 |
| 生日（仅 birthdate 类型，其他丢失） | `BDAY` | ExportImportantDates（getVCardDate 占位） | 40 |
| 标签名数组 | `CATEGORIES` | ExportLabels | 40 |
| `updated_at`（UTC 格式） | `REV` | ExportTimestamp | 1000 |

### Group → vCard

| Monica 数据 | vCard 字段 | 导出器 | Order |
|------------|-----------|--------|-------|
| `uuid` / `id`（或补 UID 时 distant_uuid 优先） | `UID` | ExportVCard 初始化 / 补 UID | — |
| 组 URL | `SOURCE` | ExportVCard 初始化 | — |
| — | `VERSION` | ExportVCard 初始化 | — |
| 固定值 `group` | `KIND` / `X-ADDRESSBOOKSERVER-KIND` | ExportKind | 1 |
| `name` | `FN` / `N` | ExportNames | 10 |
| 成员 UUID 列表（distant_uuid 优先） | `MEMBER` / `X-ADDRESSBOOKSERVER-MEMBER` | ExportMembers（增量） | 20 |
| `updated_at`（UTC 格式） | `REV` | ExportTimestamp（漏记补回） | 1000 |

---

## 关键设计要点 & 易错点速览

1. **鉴权副作用**：contact/group 对象的获取发生在 BaseService 权限校验阶段，并非 ExportVCard::execute() 自己查询——`$this->contact` / `$this->group` 是校验链的副产品
2. **withoutTimestamps 持久化**：vCard 序列化结果保存时不触发模型时间戳，`REV` 字段反映的是"导出前"的真实 `updated_at`
3. **UID 两条路径不对称**：补 UID 用 distant_uuid → uuid → id 三级，新建 VCard 时只有 uuid → id 两级，首次导出或 vcard 损坏时 distant_uuid 可能丢失
4. **ReadVObject null 路径数据丢失**：ParseException 触发时会降级为重建全新 VCard，旧 VCard 中所有自定义字段被清除
5. **先删后写原则**：所有导出器先删再写，保证导出结果纯净；ExportContactInformation 更一次性删五个字段，即便部分字段当前无数据也要清空
6. **五联系方式参数不对称**：只有 email/phone 消费 kind 和 pref 字段，IMPP/X-SOCIAL-PROFILE/URL 即便数据库中有这些属性也不写入 vCard 参数
7. **地址槽位反直觉**：line_1 → Extended Address（公寓号槽），line_2 → Street Address（主街道槽），与用户对"line_1=第一行=主地址"的直觉相反
8. **getVCardDate 占位细节**：年仅 padLeft 到 2 位、月日占位用单 `-`、输出紧凑格式而非 ISO 横杠分隔
9. **非生日重要日期丢失**：只有 birthdate 类型映射到 BDAY，其余日期类型不导出
10. **Group 完整导出器列表**：ExportKind (1) + ExportNames (10) + ExportMembers (20) + ExportTimestamp (1000)，共四个导出器
11. **成员增量同步**：Group 成员不采用"先全删再重写"，而是用集合对比增删，避免破坏已有属性节点上可能附带的参数信息
12. **X-SOCIAL-PROFILE 值位置异常**：用户名不写在 VALUE 部分（VALUE 为空串），而放在 `X-USER` 参数中；URL 分支值由 `type + data` 直接拼接生成
