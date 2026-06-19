# 异步任务与邮件失败恢复代码分析

## 关键纠正说明（对照初版的事实修正）

本版对初版分析中以下**事实错误**进行了纠正：

| 初版错误 | 代码事实 |
|---------|----------|
| 提醒取数条件包含 `triggered_at IS NULL` | **取数只看 `scheduled_at <= NOW()`，完全不依赖 triggered_at** |
| triggered_at 是重复发送的主防护 | triggered_at 仅作审计记录和前端展示用，不参与生产调度取数 |
| 循环提醒"创建下一次调度记录" | 循环提醒**更新同一条记录**的 scheduled_at 到未来时间点，不创建新记录 |
| 一次性提醒通过 triggered_at 标记完成 | 一次性提醒**直接 DELETE 整条记录** |
| Reschedule 在 triggered_at 更新之后执行 | 执行顺序：发送 → Reschedule(改scheduled_at/删记录) → updateTriggeredAt |

---

## 1. 整体架构概览

### 1.1 队列基础设施

**配置文件**：[config/queue.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/config/queue.php)

```
默认连接: sync (生产环境应切换为 database/redis)
重试超时: retry_after = 90秒 (任务执行超过90秒会被重新放回队列)
失败驱动: database-uuids (使用UUID标识失败任务)
```

**Worker启动参数**：[scripts/docker/queue.sh](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/scripts/docker/queue.sh)

```bash
php artisan queue:work --sleep=10 --timeout=0 --tries=3 --queue=high,default,low
```

| 参数 | 值 | 含义 |
|------|----|------|
| sleep | 10 | 无任务时休眠10秒 |
| timeout | 0 | 任务执行无超时限制 |
| tries | 3 | **默认重试3次**（Job类可覆盖） |
| queue | high,default,low | 按优先级顺序处理 |

### 1.2 数据表结构

**调度表 contact_reminder_scheduled**：[2022_02_18_215852_create_reminders_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/database/migrations/2022_02_18_215852_create_reminders_table.php#L47-L54)

| 字段 | 类型 | 用途 |
|------|------|------|
| id | bigIncrements | 主键 |
| user_notification_channel_id | FK | 通知渠道ID |
| contact_reminder_id | FK | 提醒ID |
| scheduled_at | datetime | **下次触发时间（也是唯一的取数条件）** |
| triggered_at | datetime nullable | **上次触发时间（仅记录，不参与取数过滤）** |
| created_at / updated_at | timestamps | 时间戳 |

**注意**：contact_reminder_scheduled 是 BelongsToMany 的 pivot 表，通过 `(user_notification_channel_id, contact_reminder_id)` 组合唯一标识一条调度记录。

**任务表 jobs**：[2022_01_22_183321_create_jobs_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/database/migrations/2022_01_22_183321_create_jobs_table.php)

**失败任务表 failed_jobs**：[2019_08_19_000000_create_failed_jobs_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/database/migrations/2019_08_19_000000_create_failed_jobs_table.php)

---

## 2. 提醒调度取数是否依赖触发标记？—— 不依赖

### 2.1 生产代码取数逻辑

**ProcessScheduledContactReminders::handle()**：[ProcessScheduledContactReminders.php#L38-L40](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L38-L40)

```php
$currentDate = Carbon::now();
$currentDate->second = 0;

$scheduledContactReminders = DB::table('contact_reminder_scheduled')
    ->where('scheduled_at', '<=', $currentDate)
    ->get();
```

**取数条件只有一个：`scheduled_at <= 当前时间`**，完全没有 `triggered_at IS NULL`。

这意味着：
- triggered_at 是 NULL 还是有值，**不影响**这条记录是否被取出
- 只要 scheduled_at 还没到未来，每分钟调度都会再次命中这条记录

### 2.2 triggered_at 字段的真实用途

triggered_at 只在以下三个场景使用，**均不参与生产调度取数**：

#### 场景1：前端展示过滤（ViewHelper）

**VaultShowViewHelper.php#L44-L51**：[VaultShowViewHelper.php#L44-L51](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L44-L51)

```php
$channel->contactReminders()
    ->wherePivot('scheduled_at', '<=', $currentDate->addDays(30))
    ->wherePivot('triggered_at', null)    // 前端展示只看"未触发过"的
    ->orderByPivot('scheduled_at', 'asc')
    ->get()
```

这只是在 Vault 详情页展示"即将到来的提醒"时过滤已触发的记录，不影响调度任务。

#### 场景2：测试命令过滤（非生产代码）

**TestReminders.php#L43-L45**：[TestReminders.php#L43-L45](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Console/Commands/TestReminders.php#L43-L45)

```php
$scheduledContactReminders = DB::table('contact_reminder_scheduled')
    ->where('triggered_at', null)
    ->get();
```

这是手动测试用的 Artisan 命令（只在非 production 环境可用），同样不参与生产调度。

#### 场景3：审计记录

triggered_at 记录这条调度最近一次被触发处理的时间，用于问题排查。

### 2.3 调度频率

**routes/console.php**：[console.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/routes/console.php)

```php
Schedule::job(ProcessScheduledContactReminders::class, 'minutes', 1);
```

**每分钟执行一次**。注意：代码注释写的是 "every five minutes"，但实际 `minutes(1)` 是每分钟，注释与代码不一致。

---

## 3. 循环提醒怎么重排？—— 更新同一条记录的 scheduled_at

### 3.1 处理流程完整链路

**ProcessScheduledContactReminders::handle()** 核心流程：

```
取数（scheduled_at <= NOW()）
    ↓
foreach 循环处理每条记录
    ├─ try {
    │   ├─ 查询 ContactReminder 和 Contact
    │   ├─ triggerNotification()
    │   │   ├─ 检查 channel->active（不活跃直接return）
    │   │   ├─ Notification::route()->notify()  ← 实际发送邮件/Telegram
    │   │   ├─ increment('number_times_triggered')  ← 触发次数+1
    │   │   └─ RescheduleContactReminderForChannel::execute()  ← 重排（改scheduled_at或删记录）
    │   │
    │   └─ updateScheduledContactReminderTriggeredAt()  ← 设置 triggered_at = NOW()
    │
    └─ } catch (\Exception $e) {
        ├─ Log::error()
        ├─ UserNotificationSent::create(..., error => ...)  ← 记录失败
        ├─ channel->fails++
        ├─ fails >= 10 ? 禁用channel并删除其所有调度 : 继续
        └─ channel->save()
```

**关键执行顺序**：
1. **先发送通知**
2. **然后 Reschedule**（把 scheduled_at 推到未来 / 删除一次性记录）
3. **最后更新 triggered_at**

### 3.2 RescheduleContactReminderForChannel 重排逻辑

**类定义**：[RescheduleContactReminderForChannel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannel.php)

```php
public function execute(array $data): void
{
    if (! $this->userNotificationChannel->active) {
        throw new ModelNotFoundException('The user notification channel is not active anymore.');
    }

    if ($this->contactReminder->type !== ContactReminder::TYPE_ONE_TIME) {
        $this->schedule();  // 循环提醒：更新 scheduled_at
    } else {
        // 一次性提醒：直接 DELETE 整条记录
        DB::table('contact_reminder_scheduled')
            ->where('id', $this->data['contact_reminder_scheduled_id'])
            ->delete();
    }
}
```

### 3.3 循环提醒的 scheduled_at 更新方式

**schedule() 方法**：[RescheduleContactReminderForChannel.php#L62-L87](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannel.php#L62-L87)

```php
private function schedule(): void
{
    $record = DB::table('contact_reminder_scheduled')
        ->where('id', $this->data['contact_reminder_scheduled_id'])
        ->first();

    // 基于当前 scheduled_at 计算下一次时间
    $this->upcomingDate = Carbon::createFromFormat('Y-m-d H:i:s', $record->scheduled_at);

    switch ($this->contactReminder->type) {
        case ContactReminder::TYPE_RECURRING_DAY:
            $this->upcomingDate = $this->upcomingDate->addDay();      // +1天
            break;
        case ContactReminder::TYPE_RECURRING_MONTH:
            $this->upcomingDate = $this->upcomingDate->addMonth();    // +1月
            break;
        case ContactReminder::TYPE_RECURRING_YEAR:
            $this->upcomingDate = $this->upcomingDate->addYear();     // +1年
            break;
    }

    // 用 syncWithoutDetaching 更新 pivot 表（不创建新记录）
    $this->contactReminder->userNotificationChannels()
        ->syncWithoutDetaching([
            $this->userNotificationChannel->id => [
                'scheduled_at' => $this->upcomingDate,
            ]
        ]);
}
```

**重点**：
- `syncWithoutDetaching` 通过 `(contact_reminder_id, user_notification_channel_id)` 组合定位 pivot 记录
- **更新的是同一条记录**的 scheduled_at 字段，不插入新记录
- 只更新 scheduled_at，triggered_at 字段保持不变（由随后的 updateTriggeredAt 设置）

### 3.4 一次性提醒的处理

对于 `TYPE_ONE_TIME`：
```php
DB::table('contact_reminder_scheduled')
    ->where('id', $this->data['contact_reminder_scheduled_id'])
    ->delete();
```

直接 DELETE 整条记录，不再保留。

### 3.5 测试验证（RescheduleContactReminderForChannelTest）

测试文件确认了上述行为：[RescheduleContactReminderForChannelTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/tests/Unit/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannelTest.php)

以日循环提醒为例：
```php
// 初始：scheduled_at = 2018-01-01
$id = DB::table('contact_reminder_scheduled')->insertGetId([
    'scheduled_at' => '2018-01-01 00:00:00',
    ...
]);

// 执行 Reschedule
(new RescheduleContactReminderForChannel)->execute([...]);

// 断言：同一条记录的 scheduled_at 更新为 2018-01-02
$this->assertDatabaseHas('contact_reminder_scheduled', [
    'scheduled_at' => '2018-01-02 00:00:00',
    'triggered_at' => null,   // syncWithoutDetaching 不改 triggered_at
]);
```

### 3.6 真正的"去重"机制

既然取数不看 triggered_at，那么防止重复发送的真正机制是：

**Reschedule 把 scheduled_at 推到未来，使下一次查询 `scheduled_at <= NOW()` 不再命中这条记录。**

这意味着防重复完全依赖 **Reschedule 是否在发送成功后被执行**。如果发送成功但 Reschedule 未执行，scheduled_at 仍是过去时间，下次调度（每分钟）会再次命中。

---

## 4. 人工重试会不会重复发送？—— 视 Reschedule 是否成功而定

### 4.1 ProcessScheduledContactReminders 的 Job 载荷特点

这个Job的构造函数**没有任何参数**：

```php
class ProcessScheduledContactReminders implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable;
    // 无 SerializesModels
    // 构造函数无参数
    // handle() 内部每次都重新查询 DB
}
```

**载荷极其轻量**，不携带任何提醒数据。每次执行（包括人工重试）都会在 `handle()` 内重新执行：

```php
$scheduledContactReminders = DB::table('contact_reminder_scheduled')
    ->where('scheduled_at', '<=', $currentDate)
    ->get();
```

### 4.2 人工重试的重复发送场景判定

人工执行 `php artisan queue:retry <uuid>` 后，Job 会重新跑一遍完整的 handle()。是否重复发送，取决于**上一次执行时 Reschedule 有没有成功**：

#### 场景A：发送前失败（取数/查询阶段异常）

```
DB::table(...) 或 findOrFail 抛出异常
    ↓
Reschedule 未执行，scheduled_at 仍是过去时间
    ↓
人工重试 → 重新查询 → 命中 → 正常发送（不算重复）
```
**结果**：✅ 不重复（之前根本没发出去）

#### 场景B：发送成功，Reschedule 成功，进程在 updateTriggeredAt 之后正常结束

```
发送成功
    ↓
Reschedule 成功 → scheduled_at 被推到未来（或记录被删除）
    ↓
updateTriggeredAt 成功
    ↓
Job 正常完成
```
人工重试时：
- 循环提醒：scheduled_at 在未来 → 不命中 → ✅ 不重复
- 一次性提醒：记录已被 DELETE → 不命中 → ✅ 不重复

**结果**：✅ 不重复

#### 场景C：发送成功，但 Reschedule 失败/未执行，Job 整体异常

```
Notification::route()->notify() → 邮件已成功发出
    ↓
RescheduleContactReminderForChannel::execute() 抛出异常（如 channel 突然被禁用）
    ↓
Reschedule 未完成 → scheduled_at 仍是过去时间
    ↓
异常未被catch（Reschedule 在 try 块内但抛异常了？——实际上有catch）
```

**⚠️ 注意**：ProcessScheduledContactReminders 的 try-catch 包裹了整个单条记录的处理逻辑，包括 Reschedule。Reschedule 失败会被 catch 住，不会冒泡到队列层。因此：

```
Reschedule 异常 → catch 捕获 → 记录错误 → channel->fails++
    ↓
Job 继续处理下一条记录，不会整体失败
    ↓
但这条记录的 scheduled_at 仍是过去时间，triggered_at 未被设置
    ↓
下一分钟调度再次命中 → 又尝试发送 → 可能重复！
```

熔断器（fails >= 10 自动禁用 channel）最多能兜住 10 次重复。

#### 场景D：发送成功，Reschedule 成功，但进程在 Job 完成前被杀 / 超过 retry_after=90秒

```
发送成功（邮件已发出）
    ↓
Reschedule 成功 → scheduled_at 已推到未来
    ↓
进程被杀或超过90秒
    ↓
队列检测到 retry_after 超时 → 把 Job 重新放回队列
    ↓
人工重试或队列自动重试 → handle() 重新执行
    ↓
查询 scheduled_at <= NOW() → 这条记录 scheduled_at 已在未来 → 不命中
```

**结果**：✅ 不重复（因为 Reschedule 已成功把 scheduled_at 推到未来）

#### 场景E：发送成功，Reschedule 未执行，进程被杀

```
发送成功（邮件已发出）
    ↓
还没执行 Reschedule → 进程被杀
    ↓
scheduled_at 仍是过去时间
    ↓
重试 → 重新查询 → 命中 → 再次发送！
```

**结果**：❌ **重复发送**

### 4.3 重复发送场景总结

| 场景 | Reschedule是否成功 | 人工重试是否重复 | 最多重复次数 |
|------|-------------------|-----------------|-------------|
| A：发送前失败 | 未执行 | 否（没发出去） | 0 |
| B：正常完成 | 成功 | 否 | 0 |
| C：发送成功、Reschedule失败、catch住 | 失败 | **是（每分钟调度都会命中）** | ≤10（熔断器） |
| D：发送成功、Reschedule成功、进程被杀 | 成功 | 否 | 0 |
| E：发送成功、Reschedule未执行、进程被杀 | 未执行 | **是** | ≤3（队列tries） |

### 4.4 ProcessScheduledContactReminders 的队列重试

这个Job没有设置 `$tries`，使用 Worker 默认值 `--tries=3`。队列层面最多自动重试 2 次（加上首次执行共 3 次）。

但更危险的是 `retry_after=90` 秒的静默重试：
- 如果 handle() 处理大量提醒，执行超过 90 秒
- 队列认为 Worker 挂了，把 Job 重新放回队列
- 此时原来的 Worker 可能还在运行，同时新的 Worker 也开始执行
- 两个 Worker 同时取数，可能命中同一条记录 → **并发重复发送**

---

## 5. 其他邮件任务的重试与恢复

### 5.1 验证邮件（SendVerificationEmailChannel）

**类定义**：[SendVerificationEmailChannel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Settings/ManageNotificationChannels/Jobs/SendVerificationEmailChannel.php)

```php
class SendVerificationEmailChannel implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected UserNotificationChannel $channel;  // 序列化只存ID

    public function handle()
    {
        if ($this->channel->type !== UserNotificationChannel::TYPE_EMAIL) {
            return;
        }
        Mail::to($this->channel->content)
            ->send(new UserNotificationChannelEmailCreated($this->channel));
    }
}
```

**重试与重复分析**：
- 无 `$tries` → 默认 3 次队列重试
- 无 `$backoff` → 重试无间隔
- **无任何幂等性检查**（没有检查 `verified_at`，没有检查是否已发送过）
- 人工重试 → 反序列化 `$channel` → 重新发送 → **肯定重复**
- 所有邮件中的验证链接相同（同一个 `verification_token`），用户点击任意一封即可验证

### 5.2 邀请邮件（UserInvited）

**类定义**：[UserInvited.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Mail/UserInvited.php)

```php
class UserInvited extends Mailable implements ShouldQueue
{
    use Queueable, SerializesModels;
    // 无 $tries → 默认3次
    // 无幂等性检查
}
```

Mailable 本身实现 `ShouldQueue`，邮件自动进入队列。人工重试会重复发送。

### 5.3 测试邮件（SendTestEmail）

**类定义**：[SendTestEmail.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Settings/ManageNotificationChannels/Services/SendTestEmail.php)

- **同步执行**（不实现 ShouldQueue），不走队列
- 失败直接抛异常，无重试
- 不存在人工重试问题

---

## 6. 其他重试机制

### 6.1 QueuableService 基类（tries=1，不重试）

**QueuableService.php**：[QueuableService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Services/QueuableService.php)

```php
abstract class QueuableService extends BaseService implements ShouldQueue
{
    public int $tries = 1;  // 只跑1次，不重试
}
```

子类：SetupAccount、UpdateVCard 等。人工重试不受此限制（从 failed_jobs 恢复后 attempts 重置）。

### 6.2 DAV任务的业务内重试

**PushVCard**：[PushVCard.php#L80-L105](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/DavClient/Jobs/PushVCard.php#L80-L105)

```php
private function pushDistant(int $depth = 1): string
{
    try {
        $response = $this->subscription->getClient()
            ->request('PUT', $this->uri, $this->card, $this->headers());
        return $response->header('Etag');
    } catch (RequestException $e) {
        if ($depth > 0 && $e->response->status() === 412) {
            $this->mode = self::MODE_MATCH_NONE;
            return $this->pushDistant(--$depth);  // 412时内部重试1次
        } else {
            $this->fail($e);
            throw $e;
        }
    }
}
```

业务逻辑内对 412 状态码重试1次，与队列层面的重试是**叠加关系**。

### 6.3 失败清理机制

```php
Schedule::command('queue:prune-failed --hours=48', 'daily');
```
失败任务最多保留 48 小时，之后自动清除，无法再人工重试。

---

## 7. 关键风险点总结

### 7.1 提醒邮件的防重机制实际很脆弱

**错误认知**：triggered_at 过滤 + 异常捕获 = 可靠防重

**代码事实**：
- 取数不依赖 triggered_at（这是最大的误解）
- 真正防重靠 Reschedule 把 scheduled_at 推到未来
- 如果发送成功但 Reschedule 未执行，每分钟调度都会再次命中

**重复发送的时间窗口**：

```
发送成功 → Reschedule 执行之间的时间窗
          |<-------->|
          这段时间内进程被杀/异常，就会重复

另外：C调度每分钟执行，如果 Reschedule 失败（被catch），
      下次调度会重新命中，直到熔断器触发（最多10次）
```

### 7.2 重复发送风险矩阵

| 任务类型 | 队列tries | 幂等防护 | 人工重试是否重复 | 风险等级 |
|---------|-----------|----------|-----------------|----------|
| 提醒邮件 | 3 | Reschedule推scheduled_at到未来 | **视Reschedule是否成功而定** | ⚠️⚠️ 中高 |
| 验证邮件 | 3 | 无 | **是** | ⚠️⚠️⚠️ 高 |
| 邀请邮件 | 3 | 无 | **是** | ⚠️⚠️⚠️ 高 |
| 测试邮件 | 0（同步） | 无 | 不适用 | ✅ 低 |
| DAV同步 | 1+业务重试1次 | etag机制 | 否（幂等） | ✅ 低 |
| SetupAccount | 1 | 数据初始化幂等 | 低风险 | ✅ 低 |

### 7.3 retry_after 静默重试风险

```php
'retry_after' => 90,  // config/queue.php
```

- 超过 90 秒自动释放回队列，**不增加 attempts**，不受 tries 限制
- 如果 ProcessScheduledContactReminders 处理大量提醒超过 90 秒，可能并发执行多个实例
- 原 Worker 和新 Worker 可能同时处理同一条 scheduled 记录（并发重复）

---

## 8. 代码走向流程图

### 8.1 提醒邮件完整链路（纠正后）

```
Schedule 每分钟触发
    ↓
ProcessScheduledContactReminders::handle()
    ├─ DB查询 contact_reminder_scheduled WHERE scheduled_at <= NOW()
    │                                      ↑ 注意：没有 triggered_at 条件
    ├─ foreach 循环
    │   ├─ try {
    │   │   ├─ findOrFail(ContactReminder, UserNotificationChannel)
    │   │   ├─ triggerNotification()
    │   │   │   ├─ if (!channel->active) return
    │   │   │   ├─ Notification::route()->notify(ReminderTriggered)
    │   │   │   │   └─ ReminderTriggered::toMail()
    │   │   │   │       ├─ UserNotificationSent::create() ← 记录日志（发送前）
    │   │   │   │       └─ 返回 MailMessage ← 实际发送
    │   │   │   ├─ increment number_times_triggered
    │   │   │   └─ RescheduleContactReminderForChannel.execute()
    │   │   │       ├─ 循环提醒: syncWithoutDetaching 更新 scheduled_at → 未来
    │   │   │       └─ 一次性提醒: DELETE 整条记录
    │   │   │
    │   │   └─ updateScheduledContactReminderTriggeredAt() ← triggered_at = NOW()
    │   │         ↑ 注意：Reschedule 在这之前就执行完了
    │   │
    │   └─ } catch (\Exception $e) {
    │       ├─ Log::error()
    │       ├─ UserNotificationSent::create(..., error => ...)
    │       ├─ channel->fails++
    │       ├─ fails >= 10 ? channel->active = false, 删除所有调度 : 继续
    │       └─ channel->save()
    └─ 结束
```

### 8.2 人工重试对提醒邮件的影响判定

```
人工 queue:retry <uuid>
    ↓
handle() 重新执行，重新查询 DB
    ↓
这条提醒的 scheduled_at 是否已被 Reschedule 推到未来？
    ├─ 是（Reschedule 成功过） → 不命中 → ✅ 不重复
    ├─ 否（Reschedule 失败/未执行） → 命中 → ❌ 重复发送
    │       └─ 如果 channel->active = false → triggerNotification 直接 return → 不发送但也不 Reschedule → 僵尸记录
```

---

## 9. 优化建议

### 9.1 短期优化（低风险）

1. **取数增加 triggered_at 条件**（真正把 triggered_at 作为幂等边界）

   ```php
   // ProcessScheduledContactReminders::handle()
   $scheduledContactReminders = DB::table('contact_reminder_scheduled')
       ->where('scheduled_at', '<=', $currentDate)
       ->whereNull('triggered_at')          // ← 增加这行
       ->get();
   ```

   这是最直接、最低风险的修复，把之前"假设存在"的机制补回来。

2. **为邮件Job添加超时限制**（小于 retry_after）

   ```php
   // ProcessScheduledContactReminders, SendVerificationEmailChannel 等添加
   public int $timeout = 60;  // 小于 queue.retry_after = 90
   ```

3. **验证邮件添加幂等性检查**

   ```php
   public function handle()
   {
       if ($this->channel->verified_at !== null) {
           return;
       }
       // ... 发送
   }
   ```

### 9.2 中期优化（中风险）

1. **Reschedule 与发送使用数据库事务包裹**，保证"发送成功+Reschedule成功"的原子性
   - 注意：邮件发送是外部IO，无法与DB事务真正原子化，需要用"事务消息"或"发件箱模式"

2. **为 contact_reminder_scheduled 的 (user_notification_channel_id, contact_reminder_id) 增加唯一索引**，防止 syncWithoutDetaching 产生重复记录（目前虽然逻辑上不会重复，但表结构没有唯一约束）

3. **提醒调度使用悲观锁** `WHERE ... FOR UPDATE SKIP LOCKED`，防止并发 Worker 处理同一条记录

### 9.3 长期优化（高风险）

1. **发件箱模式（Transactional Outbox）**：先写"待发送邮件"到DB（在业务事务内），异步任务扫描发送，保证"业务操作+邮件发送"原子性
2. **接入邮件服务商 Webhook**：准确追踪真实投递状态，而非仅记录"尝试发送"
3. **分布式锁**（Redis）防止 ProcessScheduledContactReminders 并发执行

---

## 10. 相关文件索引

| 类型 | 文件路径 | 关键内容 |
|------|---------|----------|
| 提醒调度Job | [ProcessScheduledContactReminders.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php) | 取数无triggered_at条件、执行顺序 |
| 重排服务 | [RescheduleContactReminderForChannel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannel.php) | 循环更新scheduled_at / 一次性DELETE |
| 调度表迁移 | [2022_02_18_215852_create_reminders_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/database/migrations/2022_02_18_215852_create_reminders_table.php) | contact_reminder_scheduled 字段定义 |
| 调度配置 | [routes/console.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/routes/console.php) | 每分钟调度一次 |
| 队列配置 | [config/queue.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/config/queue.php) | retry_after=90秒 |
| Worker脚本 | [scripts/docker/queue.sh](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/scripts/docker/queue.sh) | --tries=3 默认值 |
| 验证邮件Job | [SendVerificationEmailChannel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Settings/ManageNotificationChannels/Jobs/SendVerificationEmailChannel.php) | 无幂等检查 |
| 邀请邮件 | [app/Mail/UserInvited.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Mail/UserInvited.php) | ShouldQueue无幂等 |
| 熔断器配置 | [config/monica.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/config/monica.php) | max_notification_failures=10 |
| 测试命令（非生产） | [TestReminders.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Console/Commands/TestReminders.php) | 有triggered_at条件（非生产） |
| 前端ViewHelper | [VaultShowViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php) | 前端展示过滤triggered_at |
