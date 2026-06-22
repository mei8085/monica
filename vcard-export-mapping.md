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

[ExportVCard::permissions()](app/Domains/Contact/Dav/Services/ExportVCard.php#L39-L48) 声明了 5 个权限钩子，由 [BaseService::validateRules()](app/Services/BaseService.php#L96-L119) 执行校验。

#### 鉴权遍历方式

BaseService 的鉴权**不是按 `permissions()` 声明的顺序执行**，而是按 `self::$permissionDependencies` 数组的**固定键顺序**逐条遍历。遍历逻辑：

```
foreach (self::$permissionDependencies as $key => $values) {
    if ($permissions->contains($key)) {
        // 先校验依赖是否都在 permissions 中（缺则抛异常）
        // 再调用 validatePermission($key) 执行实际校验
    }
}
```

也就是说，`$permissionDependencies` 数组的键顺序**硬编码了校验执行顺序**，调用方只需声明"我需要哪些权限"，BaseService 自动按固定拓扑顺序展开执行，并自动校验依赖完整性。

#### 权限声明与执行顺序对应表

| `$permissionDependencies` 固定顺序 | 若 ExportVCard 声明了 | 副作用（挂载对象属性） |
|----------------------------------|---------------------|---------------------|
| 1. `author_must_belong_to_account` | ✅ 声明了 | `$this->author` = 从 account 下查找到的 User |
| 2. `vault_must_belong_to_account` | ✅ 声明了 | `$this->vault` = 从 account 下查找到的 Vault |
| 3. `author_must_be_account_administrator` | ❌ 未声明 | — |
| 4. `author_must_be_vault_manager` | ❌ 未声明 | — |
| 5. `author_must_be_vault_editor` | ❌ 未声明 | — |
| 6. `author_must_be_in_vault` | ✅ 声明了 | 校验 author 在 vault 中有 VIEW 权限（含以上） |
| 7. `contact_must_belong_to_vault` | ✅ 声明了 | **有 contact_id 时**：`$this->contact` = 从 vault 下查找到的 Contact，并再校验 `contact.vault_id` 一致性 |
| 8. `group_must_belong_to_vault` | ✅ 声明了 | **有 group_id 时**：`$this->group` = 从 vault 下查找到的 Group，并再校验 `group.vault_id` 一致性 |

#### contact 与 group 的触发顺序

在 `$permissionDependencies` 中，**`contact_must_belong_to_vault` 排在 `group_must_belong_to_vault` 之前**。若某次调用同时传了 `contact_id` 和 `group_id`（虽然 `required_if` 规则不鼓励这样做），contact 的校验和挂载会先于 group 发生。

> 但对 ExportVCard 正常调用而言，`rules()` 中 `required_if` 保证二者选其一，因此实际只会触发其中一个校验分支。

> 因此后续 `execute()` 中 `$this->contact` 和 `$this->group` 并非从 execute 自行查询，而是**鉴权过程的副作用**——由 BaseService 在校验 `contact_must_belong_to_vault` / `group_must_belong_to_vault` 时已经赋值好。鉴权全部通过后才进入真正的导出逻辑。

### withoutTimestamps 持久化的深层原因

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

#### 与 prepareCard 缓存机制的关联

withoutTimestamps 的真正目的是配合 CardDAV 后端的缓存机制。[CardDAVBackend::prepareCard()](app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L180-L215) 有一段"脏检查"逻辑：

```php
$carddata = $resource->vcard;
if ($carddata) {
    $timestamp = $this->rev($carddata);  // 解析 VCard 的 REV 字段
}
if ($carddata === null || empty($carddata) || $timestamp === null || $timestamp < $resource->updated_at) {
    $carddata = $this->refreshObject($resource);  // 重新导出
}
```

判断条件是：**vcard 缓存的 REV 时间 < 模型 updated_at → 缓存过期，需要重新导出**。

如果导出回写时不使用 `withoutTimestamps`，会发生"缓存雪崩死循环"：
1. 联系人真实内容被修改 → `updated_at` 前进
2. CardDAV 请求触发 `prepareCard()`，发现 `REV < updated_at` → 调用 `refreshObject()` 重新导出
3. 导出完成后 `$obj->save()` 会再次更新 `updated_at`（比 REV 里写的时间还新一点）
4. 下一次 `prepareCard()` 又会发现 `REV < updated_at` → 又重新导出
5. 无限循环，vcard 缓存永远判定为过期

`withoutTimestamps` 保证了：导出写回 vcard 字段时，`updated_at` 保持不变，使得 `REV == updated_at`，缓存有效。

#### withoutTimestamps 不阻止模型事件

Laravel/Eloquent 的 `withoutTimestamps` 闭包只做一件事：**跳过 `updated_at` / `created_at` 的自动赋值**。它不阻止任何模型事件。

在 ExportVCard::execute 中调用 `$obj->save()` 时：
- ❌ 被禁用的：`saving` 前自动设置 `updated_at` / `created_at`、`updating` 前自动刷新 `updated_at`
- ✅ 正常触发的：`saving` / `saved` / `updating` / `updated` 事件全部按正常生命周期派发

**风险**：如果存在监听 `saved` / `updated` 的模型观察者或事件订阅者（审计日志、缓存刷新、Elasticsearch 索引、第三方通知等），每次 vCard 导出缓存回写都会触发这些监听器，产生"只读操作产生大量副作用"的反模式。

#### REV 秒级精度 vs. 亚秒级 updated_at：仍然判过期

缓存脏检查核心比较：[CardDAVBackend::prepareCard()](app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L191-L192)

```php
$timestamp = $this->rev($carddata);
if ($timestamp === null || $timestamp < $resource->updated_at) { /* 重新导出 */ }
```

**精度不对称导致的 bug**：

| 变量 | 来源 | 精度 |
|------|------|------|
| `$timestamp`（REV） | VCard 中 `20240622T103025Z` 经 `Carbon::parse()` | 秒级（微秒位=0） |
| `$resource->updated_at` | 数据库 `datetime` / `datetime(6)` 字段 | **可能带微秒**（取决于数据库配置） |

导出时 [ExportTimestamp](app/Domains/Contact/ManageContact/Dav/ExportTimestamp.php#L29) 的 REV 格式是 `Ymd\THis\Z`，**截断到秒**，微秒信息丢失。

**场景**：用户在 10:30:25.456789（含微秒）编辑联系人 → `updated_at = 2024-06-22 10:30:25.456789` → 导出时 REV = `20240622T103025Z`（微秒截断） → 下次 prepareCard 时：`Carbon(10:30:25.000000) < Carbon(10:30:25.456789)` = **true** → 每次都判定缓存过期 → 每次都重新导出 → withoutTimestamps 的防死循环设计**完全失效**。

> 触发条件取决于数据库是否启用了 datetime 微秒精度（MySQL 5.6+ 支持 `DATETIME(6)`，PostgreSQL 默认支持）。如果数据库字段是秒级精度则不会触发。

#### vCard 导出的两类入口

vCard 导出在代码中有两类独立调用入口：

| 入口 | 调用路径 | 是否写回 vcard 缓存 | 用途 |
|------|---------|-------------------|------|
| Web 前端下载 | [ContactVCardController::download()](app/Domains/Contact/ManageContact/Web/Controllers/ContactVCardController.php#L17-L26) → `exportVCard()` → `app(ExportVCard::class)->execute()` | ✅ 写回（因为 ExportVCard::execute 内部 always 写回） | 用户点击"下载 vCard"按钮 |
| CardDAV 同步 | [CardDAVBackend::prepareCard()](app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L180-L215) → `refreshObject()` → `app(ExportVCard::class)->execute()` | ✅ 写回 | CardDAV 客户端拉取地址簿 |

另外 [SyncDAVBackend::getModified()](app/Domains/Contact/Dav/Web/Backend/SyncDAVBackend.php#L172-L185) 也会调用 `refreshObject()` 为修改过的对象预热 vcard 缓存，配合同步令牌机制使用。

> 注意：Web 控制器的 `exportVCard()` 方法只取返回值用 `serialize()`，**并不会跳过写回**——ExportVCard::execute 内部无条件写回数据库。两次调用路径最终都落到同一个 ExportVCard 服务，都会产生持久化副作用。

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

ReadVObject 在代码中有**两处独立调用点**：

#### 调用点 1：ExportVCard 增量导出
[ExportVCard::export() L81-L83](app/Domains/Contact/Dav/Services/ExportVCard.php#L81-L83) — 反序列化已有 vcard 字段以在其基础上增量修改字段。

#### 调用点 2：CardDAVBackend 脏检查
[CardDAVBackend::rev()](app/Domains/Contact/Dav/Web/Backend/CardDAV/CardDAVBackend.php#L217-L229) — 只为解析 VCard 中的 REV 字段值用于缓存过期判断。

两处调用点都使用了相同的 [ReadVObject::execute()](app/Domains/Contact/Dav/Services/ReadVObject.php#L34-L43)，其结构：

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

**两处调用点 null 路径的级联效果**：

| 调用点 | null 结果时的影响 |
|-------|----------------|
| ExportVCard | `$vcard` 为 null → 跳过"补 UID"分支 → 进入 `!isset($vcard)` 分支重建全新 VCard → 旧 vcard 自定义字段全部丢失，且新 UID 退化为 uuid/id 两级（无 distant_uuid） |
| CardDAVBackend::rev() | prepareCard 中 `$timestamp === null` → 判定缓存过期 → 重新导出 |

### 导出器发现与排序

[ExportVCard::exporters()](app/Domains/Contact/Dav/Services/ExportVCard.php#L132-L142) 方法负责发现和排序导出器：

- 通过 `subClasses(ExportVCardResource::class)` 反射获取所有实现了 [ExportVCardResource](app/Domains/Contact/Dav/ExportVCardResource.php) 接口的类
- 按 [Order](app/Domains/Contact/Dav/Order.php) 属性标注的整数值升序排列
- 按 `getType()` 返回值过滤，只保留与当前资源类型匹配的导出器
- 结果静态缓存（`self::$exporters`），避免重复反射

#### 静态缓存生命周期与常驻进程风险

`self::$exporters` 是 PHP 类静态属性。

**传统 FPM 模式**：**生命周期 = 单次 PHP 请求**
- 首次调用 `exporters()` 时为 null → 触发 `subClasses()` 扫描
- `subClasses()` 遍历 `app_path()` 下所有 `*.php` 文件，逐个反射判断是否为目标接口的非抽象子类
- 扫描结果排序后存入 `self::$exporters`
- 同一次请求内后续调用直接返回缓存，零开销
- 请求结束时内存释放，不跨请求、不跨进程

**常驻进程模式（Octane / RoadRunner / Swoole）：上述断言不成立**
- 类静态属性会在整个 Worker 进程生命周期内持久化，跨请求存活
- `if (self::$exporters === null)` 在后续请求中永远不成立 → 不再重新扫描
- **问题 1**：如果开发环境中新增了导出器类（或通过 Composer 安装了新包），必须重启 Worker 才会生效
- **问题 2**：缓存中还保留了已 `newInstance()` 后的 ExportVCardResource 对象实例，如果对象内部持有状态（单例化容器依赖），可能出现状态泄漏
- **问题 3**：缓存不区分用户/账号/vault——所有请求共享同一份对象实例（这本身是 OK 的，因为导出器是无状态的），但如果未来某个导出器内部有成员变量缓存，就可能产生跨请求串数据

开销说明：文件扫描 + 反射是相对较重的操作，每个请求每个服务类（ExportVCard / ExportVCalendar 各有一份独立静态缓存）各执行一次。

> 注：`ExportVCalendar` 服务也有完全相同的静态缓存模式，缓存 VCalendar 导出器。两份缓存互相独立。

#### subClasses：Generator 与硬编码排除项

[helpers.php::subClasses()](app/Helpers/helpers.php#L88-L110) 返回 `Generator`，遍历策略：

1. 使用 `Symfony\Component\Finder\Finder` 扫描 `app_path()` 下所有 `*.php` 文件
2. **硬编码排除项**：`->notName(['helpers.php', 'TelescopeServiceProvider.php'])`
   - `helpers.php`：避免把全局函数注册文件作为类反射（否则 `new ReflectionClass('helpers')` 会抛 ReflectionException 因为 helpers 不是类）
   - `TelescopeServiceProvider.php`：Laravel Telescope 的服务提供者，不需要反射
   - 风险：**排除项是硬编码的**，如果未来新增其他"非类文件"或全局 helper 文件，需要手动补到此列表
3. 对每个文件构造完整命名空间类名，`new ReflectionClass($file)`
4. 判断 `isSubclassOf($className) && !isAbstract()`，符合则 `yield`

**Generator 的惰性特性**：
- 每次 `foreach (subClasses(...))` 都会重新执行 Finder 扫描（除非外部缓存，如 self::$exporters 就是做这件事）
- 如果调用方没有在外部缓存结果，会产生重复扫描开销
- ExportVCard / ExportVCalendar 通过 `self::$exporters = collect(subClasses(...))` 先一次性消费 Generator 转为 Collection，再缓存，避免重复扫描

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

### Order 20: ExportAddress — 地址（槽位语义易错点 + 前端错位）

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

#### 前端标签与 vCard 槽位错位问题

前端 [Addresses.vue](resources/js/Shared/Modules/Addresses.vue) 中两个输入框的 label：

| 数据库字段 | 前端 label（英文原文） | 用户理解 |
|-----------|----------------------|---------|
| `line_1` | "Address" | 街道地址（主地址） |
| `line_2` | "Apartment, suite, etc…" | 公寓号/套房号（补充地址） |

而 ExportAddress 映射到 vCard 时：

| 数据库字段 | vCard 槽位 | RFC 语义 |
|-----------|-----------|---------|
| `line_1` | ADR[1] = Extended Address | 公寓号/扩展地址（补充地址） |
| `line_2` | ADR[2] = Street Address | 街道地址（主地址） |

**结论：存在前后端错位 bug**。用户在前端 line_1 填"123 Main St"（街道）、line_2 填"Apt 4B"（公寓号），导出 vCard 后在其他客户端（如苹果通讯录）打开时，街道地址会显示成 "Apt 4B"，公寓号会显示成 "123 Main St"——两者颠倒。

这是因为：
- 前端按用户直觉：line_1 = 第一行 = 主街道 = "Address"
- 后端 vCard 导出按 RFC 直觉：ADR[1] = 扩展地址 = line_1，ADR[2] = 街道地址 = line_2
- 两者对 line_1 / line_2 的语义理解恰好反过来了

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

#### 参数不对称说明
1. **只有 email 和 phone 两种类型消费了 `kind` 和 `pref`** 字段；IMPP、X-SOCIAL-PROFILE、URL 三种类型即便是 ContactInformation 记录有 kind 或 pref=true，也不会体现在 vCard 参数中
2. **X-SOCIAL-PROFILE 的值位置特殊**：`$vcard->add('X-SOCIAL-PROFILE', '', ['TYPE' => ..., 'X-USER' => $contactInformation->data])` —— VALUE 部分写空字符串，实际用户名放在 `X-USER` 参数中。这与其他四种类型（直接把 data 写在 VALUE 部分）不同
3. **URL 分支的值拼接**：不是直接写 `data`，而是 `escape($type . $contactInformation->data)`，将 type（协议前缀/URL前缀）与 data 直接字符串拼接后作为 VALUE；同时再把原始 type 写进 TYPE 参数（出现两次）

#### kind 经 Str::upper 的大写漂移（导→导回不可逆）

Export 时对 kind 做了大写化：
```php
// ExportContactInformation L48 / L59
$parameters['TYPE'] = Str::upper($contactInformation->kind);
```

Import 时直接存回，**不做小写化还原**：
```php
// ImportContactInformation::createContactInformation L186
'contact_information_kind' => self::getParameter($data),
// getParameter 读取 TYPE 参数的原始值（已被 Str::upper 为大写）
```

**漂移路径**：
1. 初始值（数据库）：`kind = "work"` / `kind = "home"` / `kind = "cell"`（小写，来自前端预设下拉）
2. 导出 → `Str::upper` → vCard 中 `TYPE=WORK` / `TYPE=HOME` / `TYPE=CELL`（大写）
3. 导回（Import）→ 直接存回 → 数据库中 `kind = "WORK"` / `kind = "HOME"` / `kind = "CELL"`（变成大写）
4. 再导出：`Str::upper("WORK") = "WORK"`（已稳定，不再变化）

**前端补偿**：[ModuleContactInformationViewHelper](resources/js/Shared/Modules/ContactInformationViewHelpers/ModuleContactInformationViewHelper.php#L83) 中用 `Str::lower($info->kind)` 匹配预设种类 id——这说明前端已经"知道"kind 可能被大写化，做了补偿。但这是补丁而不是设计：如果有区分大小写的自定义 kind 或其他代码路径直接做精确匹配，会失败。

#### URL：在 Import 的 keys 列表里却被 getTypeId 全程屏蔽

[ImportContactInformation 的 $keys](app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php#L33-L39) 包含 'URL'：
```php
private $keys = ['EMAIL', 'IMPP', 'TEL', 'URL', 'X-SOCIAL-PROFILE'];
```

但 [ImportContactInformation::getTypeId()](app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php#L110-L155) 中：
- `EMAIL`、`TEL`：有专门分支
- `IMPP`：有专门分支（X-SERVICE-TYPE 匹配）
- `X-SOCIAL-PROFILE`：有专门分支（TYPE 参数匹配）
- **`URL`：落到 `else` 分支，直接 `return null`**（L133-L135 注释 "Type not supported"）

**导致的后果**：
1. 导入 vCard 中的 URL 属性时：`getTypeId() === null` → [L92 `if ($typeId === null) { continue; }`](app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php#L92-L94) → **整项被跳过不导入**
2. 现有记录同步判断时：[`getContactInformations()`](app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php#L58-L74) 中自定义 URL 类型落入 `default` → 键名是 `'OTHER'`，但 $key 循环到 `'URL'` 时 `$contactInformations->get('URL', collect())` 得到空集合 → **无法和现有 OTHER 类型记录对齐**
3. **导→导回结果**：Export 时自定义类型成功写到 URL 字段 → Import 时 URL 字段不识别、OTHER 键不匹配 → **整项联系方式被当成新增失败（null typeId）+ 原有 OTHER 记录被当成删除**

### Import 兜底新建 type 膨胀

[getTypeId() L145-L152](app/Domains/Contact/ManageContactInformation/Dav/ImportContactInformation.php#L145-L152) 有三级兜底策略，逐级失败后会**新建 ContactInformationType 记录**：

```
第 1 级：按专门分支查找（EMAIL/TEL/IMPP/X-SOCIAL-PROFILE）
  ↓ 失败
第 2 级：通用兜底：UPPER(type) LIKE %PROPERTY_NAME% 且 name = $name
  ↓ 失败
第 3 级：CreateContactInformationType 执行新建
```

**新建时的参数**：
```php
'name' => $name,         // EMAIL/TEL 时 $name=null；IMPP/X-SOCIAL-PROFILE 时为参数值（如 "WhatsApp"）
'type' => Str::upper($property->name),  // 如 "EMAIL" / "TEL" / "X-SOCIAL-PROFILE"
```

**膨胀风险**：
1. **name=null 时重复创建**：如果某次同步中 EMAIL 类型被误删（外键级联风险）或查找因字符集问题不命中，第 2 级查找 `where('name', null)` 可能不命中（因数据库 `NULL = NULL` 语义），每同步一次就新建一条 `{ name: null, type: 'EMAIL' }` 类型——type 字段没有唯一约束，可无限重复
2. **自定义 IMPP/社交平台参数值**：vCard 发送方传任意 X-SERVICE-TYPE / TYPE 参数值（如 "WeChat"、"Telegram"、拼写错误的 "Whatsapp"），Monica 没有预设 name_translation_key 对应 → 第三级兜底每次都新建 → 类型表无限增长
3. **跨账号各自新建**：查找和新建都在账号（account）维度隔离，1000 个账号各自同步一次就会产生 1000 条类似的新类型记录

> 与 Export 的对比：Export 对无法识别的类型"降级为 URL 导出"，Import 对 URL"直接跳过"，而对 IMPP/X-SOCIAL-PROFILE 查找失败"直接新建类型"——三者行为完全不对称。

#### contactInformationType 的 null 边界问题

导出代码中直接访问 `$contactInformation->contactInformationType->type`，**没有 `optional()` 或 null 检查**。

##### 数据库层面的保障

迁移文件中 `type_id` 定义为：
```php
$table->foreignIdFor(ContactInformationType::class, 'type_id')
    ->constrained('contact_information_types')
    ->cascadeOnDelete();
```

- 有外键约束 + 级联删除：当一个 ContactInformationType 被删除时，关联的 ContactInformation 记录会被数据库级联删除
- 因此正常情况下，`contactInformationType` 关系**不会为 null**

##### 仍存在的风险

1. **N+1 查询性能问题**：`$contact->contactInformations` 没有预加载 `contactInformationType`（没有 `with('contactInformationType')`），每条 ContactInformation 都会触发一次单独查询。大量联系人导出时性能较差
2. **type 字段本身为空字符串**：`ContactInformationType` 的 `type` 字段是 fillable 的，如果为空字符串，`empty($type = ...)` 为 true，该条联系方式会被整个跳过（不导出任何 vCard 字段），属于静默丢失
3. **软删除或数据损坏**：如果未来 type 表引入软删除，或外键被绕过（如直接 DB 操作），关系可能为 null，此时访问 `->type` 会抛 `Error: Attempt to read property "type" on null`

> 对比：Import 路径中 `getTypeId()` 方法大量使用 `optional($type)->id` 做防御性编程，但 Export 路径没有对应防御。

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

## Import / Export 对称性与 RFC 矛盾

vCard 的导入（Import）与导出（Export）共用同一套数据模型，但并非完全对称。以下对照二者的差异。

### Import/Export 结构对应

| 模块 | Export 导出器（Contact → vCard） | Import 导入器（vCard → Contact） |
|------|--------------------------------|--------------------------------|
| 基础信息 | ExportNames, ExportGender | ImportContact（内含 importNames, importGender, importUid） |
| 地址 | ExportAddress | ImportAddress |
| 联系方式 | ExportContactInformation | ImportContactInformation |
| 工作信息 | ExportWorkInformation | —（无对应导入器） |
| 重要日期 | ExportImportantDates | ImportImportantDates |
| 标签 | ExportLabels | ImportCategories |
| 时间戳 | ExportTimestamp（写 REV） | —（导入不更新 REV） |
| UID | ExportVCard 初始化/补 UID | ImportContact::importUid |

**缺失项**：
- **工作信息（ORG/TITLE）**：有 Export 无 Import，导入时公司和职位字段会丢失
- **SOURCE / VERSION**：导出时写入，导入时不处理
- **REV**：导出时写入 updated_at，导入时不读取更新

### 联系方式 Import/Export 不对称细节

以最复杂的联系方式为例，两边映射关系如下：

| vCard 字段 | Export 方向（Monica → vCard） | Import 方向（vCard → Monica） | 是否对称 |
|-----------|------------------------------|------------------------------|---------|
| `EMAIL` | TYPE=kind, PREF=1 if pref | TYPE 映射到 kind，PREF 不处理 | ⚠️ 半对称：kind 互通，pref 只出不进 |
| `TEL` | TYPE=kind, PREF=1 if pref | TYPE 映射到 kind，PREF 不处理 | ⚠️ 半对称：同上 |
| `IMPP` | X-SERVICE-TYPE = 类型名称 | X-SERVICE-TYPE 匹配 name_translation_key | ⚠️ 基本对称，但查找逻辑复杂 |
| `X-SOCIAL-PROFILE` | TYPE=类型名称, X-USER=用户名 | TYPE 匹配 name_translation_key, X-USER 取值 | ✅ 基本对称 |
| `URL`（其他 type） | VALUE = type + data, TYPE = type | **无对应导入分支** | ❌ 完全不对称：导出有，导入丢 |

关键不对称点：
1. **PREF 参数单向**：Export 时若 `pref=true` 写 `PREF=1`，Import 时完全不读 PREF 参数，导→导回环会丢失 pref 标记
2. **URL 类型单向丢失**：Export 时"其他 type"统一走 URL 分支（值为 type+data 拼接），Import 时没有对应 URL 导入逻辑，`getTypeId()` 对未知 vCard 属性名返回 null，整项被 skip
3. **type 匹配方式不同**：Export 时按 type 字符串精确匹配 `Str::is(..., true)`；Import 时按 `UPPER(type) LIKE %PROPERTY_NAME%` + name/name_translation_key 匹配，逻辑不对称可能导致"导出能识别的类型，导回来识别不了"，反之亦然

### 与 RFC 6350 的矛盾 / 偏离

| 字段 | RFC 6350 规定 | 实际实现 | 偏差说明 |
|------|-------------|---------|---------|
| `N` 结构 | 5 个组件：Family, Given, Additional, Honorific Prefixes, Honorific Suffixes | 只传 3 个组件（姓氏, 名字, 中间名） | 缺少前后缀，Sabre VObject 会自动补空值 |
| `ADR` 结构 | 7 个组件，第二个=Extended Address（公寓等补充信息），第三个=Street Address（主街道） | line_1 填第二槽（Extended），line_2 填第三槽（Street） | 与前端语义错位（详见"地址槽位错位"章节） |
| `BDAY` 格式 | 推荐完整日期格式 `YYYYMMDD` 或 ISO 格式，部分日期有明确语法 | 自定义占位：年仅 padLeft 2 位、月缺用 `-`、可能产出 `---22` 等非标准格式 | 部分日期格式可能不被所有 vCard 客户端正确解析 |
| `X-SOCIAL-PROFILE` | 非标准字段，RFC 中不存在 | 自定义扩展，值为空串，用户名放 `X-USER` 参数 | 非标准扩展，跨客户端兼容性差 |
| `PREF` 参数 | 值应为 1-100 的整数，表示优先级 | 只有 0/1 两种状态（pref=true → PREF=1） | 简化了 RFC 的 100 级优先级为布尔值 |
| `KIND: group` | RFC 6350 §6.1.4 标准 KIND 字段 | 优先用 `X-ADDRESSBOOKSERVER-KIND: group` 扩展 | 兼容苹果 CardDAV 的非标准扩展 |
| `MEMBER` | RFC 6350 §6.6.5 标准 MEMBER 字段 | 优先用 `X-ADDRESSBOOKSERVER-MEMBER` 扩展 | 同上，苹果扩展优先于标准 |

---

## 关键设计要点 & 易错点速览

1. **鉴权遍历方式**：BaseService 不按 `permissions()` 声明顺序校验，而是按 `$permissionDependencies` 固定键顺序逐条遍历，调用方只需声明需要哪些权限
2. **鉴权副作用**：contact/group 对象的获取发生在 BaseService 权限校验阶段，并非 ExportVCard::execute() 自己查询——`$this->contact` / `$this->group` 是校验链的副产品
3. **contact/group 校验顺序**：`contact_must_belong_to_vault` 在 `group_must_belong_to_vault` 之前，contact 校验先于 group
4. **withoutTimestamps 持久化**：vCard 序列化结果保存时不触发模型时间戳，防止与 `prepareCard()` 的 REV/updated_at 脏检查形成"缓存雪崩死循环"
5. **withoutTimestamps 不阻 saved 事件**：只禁用 `updated_at/created_at` 自动赋值，不阻止 `saving/saved/updating/updated` 事件派发，观察者可能被意外触发
6. **REV 秒级 vs 亚秒 updated_at**：REV 格式化为秒级（`YmdTHisZ`），若数据库 `datetime(6)` 带微秒，`Carbon(秒级 REV) < Carbon(含微秒 updated_at)` 恒真，缓存永远判过期，withoutTimestamps 失效
7. **两类导出入口**：Web 前端下载（ContactVCardController）与 CardDAV 同步（CardDAVBackend）两条路径，最终都调用 ExportVCard 服务，且都会写回 vcard 缓存
8. **prepareCard 脏检查**：CardDAV 的 prepareCard 先比对 VCard 中 REV 与模型 updated_at，过期才调用 refreshObject 重新导出
9. **UID 两条路径不对称**：补 UID 用 distant_uuid → uuid → id 三级，新建 VCard 时只有 uuid → id 两级，首次导出或 vcard 损坏时 distant_uuid 可能丢失
10. **ReadVObject 两处调用**：除了 ExportVCard 增量导出，CardDAVBackend::rev() 还有第二处调用，两处都有 null 路径
11. **ReadVObject null 路径数据丢失**：ExportVCard 中触发 → 降级为重建全新 VCard（丢失自定义字段 + distant_uuid）；CardDAVBackend 中触发 → 判定缓存过期 → 重导
9. **静态缓存生命周期**：`self::$exporters` 为类静态属性，生命周期 = 单次 PHP 请求（FPM 模式）；**常驻进程（Octane/RoadRunner/Swoole）下该断言不成立**——缓存跨请求存活，新增导出器类需重启 Worker；ImportVCard 的 `self::$importers` 同理
13. **subClasses Generator 与硬编码 skip**：扫描 `app_path()`，硬编码排除 helpers.php、TelescopeServiceProvider.php，新增非类文件需手动补排除项
14. **先删后写原则**：所有导出器先删再写，保证导出结果纯净；ExportContactInformation 更一次性删五个字段，即便部分字段当前无数据也要清空
15. **五联系方式参数不对称**：只有 email/phone 消费 kind 和 pref 字段，IMPP/X-SOCIAL-PROFILE/URL 即便数据库中有这些属性也不写入 vCard 参数
16. **kind 大写漂移**：Export `Str::upper($kind)`，Import 直接存回不还原，导一次后所有小写 kind 永久变大写，靠前端 ViewHelper `Str::lower()` 打补丁
17. **URL 在 Import keys 中却被屏蔽**：`$keys` 含 'URL' 但 `getTypeId()` 中 URL 落 else→`return null`，被全程跳过；且现有 URL 类型记录在 `getContactInformations()` 中归入 'OTHER' 键，无法对齐
18. **Import 兜底新建 type 膨胀**：三级查找失败就 `CreateContactInformationType`，name=null 时 `NULL = NULL` 语义可能每次新建；自定义 IMPP/社交参数跨账号各自新建 → 类型表无限增长
19. **contactInformationType null 边界**：数据库外键级联删除保证关系不会为 null，但存在 N+1 查询性能问题，且 type 字段为空字符串时会静默跳过该条联系方式
20. **地址槽位反直觉**：line_1 → Extended Address（公寓号槽），line_2 → Street Address（主街道槽），与用户对"line_1=第一行=主地址"的直觉相反
21. **前端地址错位 bug**：前端 label 中 line_1 = "Address"（主街道）、line_2 = "Apartment, suite"（公寓号），与 vCard 导出时的槽位分配恰好颠倒
22. **getVCardDate 占位细节**：年仅 padLeft 到 2 位、月日占位用单 `-`、输出紧凑格式而非 ISO 横杠分隔，部分日期组合（如 `---22`）可能不符合 RFC 预期
23. **非生日重要日期丢失**：只有 birthdate 类型映射到 BDAY，其余日期类型不导出
24. **Import/Export 不完全对称**：工作信息（ORG/TITLE）、PREF 参数、URL 类型等存在"导出有、导入丢"或"导出写、导入不读"的不对称
25. **与 RFC 的多处偏离**：N 结构缺前后缀、BDAY 部分日期非标准、X-SOCIAL-PROFILE 非标准扩展、PREF 简化为布尔值、优先使用苹果 X-ADDRESSBOOKSERVER-* 扩展而非标准 KIND/MEMBER
26. **Group 完整导出器列表**：ExportKind (1) + ExportNames (10) + ExportMembers (20) + ExportTimestamp (1000)，共四个导出器
27. **成员增量同步**：Group 成员不采用"先全删再重写"，而是用集合对比增删，避免破坏已有属性节点上可能附带的参数信息
28. **X-SOCIAL-PROFILE 值位置异常**：用户名不写在 VALUE 部分（VALUE 为空串），而放在 `X-USER` 参数中；URL 分支值由 `type + data` 直接拼接生成
