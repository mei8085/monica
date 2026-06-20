# 异步任务与邮件失败恢复代码分析

---

## 关键纠正说明（含四次修正）

| 初版错误 | 首次纠正 | 二次补充 | 三次补充 | 四次纠正（本次） |
|---------|---------|---------|---------|----------------|
| 取数含 `triggered_at IS NULL` | 取数只看 `scheduled_at <= NOW()` | —— | —— | —— |
| triggered_at 是主防护 | triggered_at 仅审计/前端用 | —— | —— | —— |
| 循环提醒创建新记录 | 更新同一条记录的 scheduled_at | Reschedule 只改 scheduled_at，**不重置 triggered_at** | —— | —— |
| 一次性提醒标记完成 | 一次性提醒直接 DELETE | —— | —— | —— |
| Reschedule 在 triggered_at 之后 | 顺序：发送 → Reschedule → updateTriggeredAt | —— | —— | —— |
| 加 triggered_at 过滤就够了 | —— | ❌ 会导致循环提醒只触发一次，需三处联动修改 | —— | —— |
| 通知异步发送，notify() 返回 ≠ 邮件已发出 | —— | —— | —— | ❌ **完全错误！ReminderTriggered 只 use Queueable，不 implements ShouldQueue → notify() 是同步执行，无论什么驱动** |
| database 驱动下 catch 捕获不到发送失败 | —— | —— | —— | ❌ **完全错误！notify() 同步执行，发送失败直接抛异常，catch 能捕获，熔断器正常生效** |
| UserNotificationSent 是"已发送"记录 | —— | —— | 写在 toMail() 里，发送前写入 | —— | 确认：确实在发送前写，不能证明发送成功，但之前基于异步推论的"catch不到"是错的 |
| UserInvited / TestEmailSent 等同理异步 | —— | —— | —— | ❌ **只有 UserInvited implements ShouldQueue → 异步；TestEmailSent 和 UserNotificationChannelEmailCreated 只 use Queueable → 同步！** |

---

## 1. Queueable 与 ShouldQueue 的本质区别（本次核心纠正）

### 1.1 代码事实：本项目各通知/Mailable 的接口实现

先看实际代码，逐个核对：

| 类名 | 类型 | extends | implements | use | 实际行为 |
|------|------|---------|-----------|-----|---------|
| **ReminderTriggered** | Notification | Notification | ❌ 无 | Queueable | **同步发送** |
| **UserInvited** | Mailable | Mailable | ✅ ShouldQueue | Queueable, SerializesModels | **异步入队** |
| **TestEmailSent** | Mailable | Mailable | ❌ 无 | Queueable, SerializesModels | **同步发送** |
| **UserNotificationChannelEmailCreated** | Mailable | Mailable | ❌ 无 | Queueable, SerializesModels | **同步发送** |

**代码位置**：
- [ReminderTriggered.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Notifications/ReminderTriggered.php)：`class ReminderTriggered extends Notification` + `use Queueable`，**无 ShouldQueue**
- [UserInvited.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Mail/UserInvited.php)：`class UserInvited extends Mailable implements ShouldQueue`
- [TestEmailSent.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Mail/TestEmailSent.php)：`class TestEmailSent extends Mailable`，**无 ShouldQueue**
- [UserNotificationChannelEmailCreated.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Mail/UserNotificationChannelEmailCreated.php)：同上，**无 ShouldQueue**

### 1.2 Laravel 语义：Queueable Trait ≠ ShouldQueue 接口

这是之前分析的核心认知错误：

| 机制 | 作用 | 自动异步？ |
|------|------|----------|
| `use Queueable` (Trait) | 提供 `onQueue()`, `delay()`, `onConnection()`, `throughMiddleware()` 等链式调用方法 | ❌ **否**，只是提供配置能力 |
| `implements ShouldQueue` (接口) | 告诉 Laravel Dispatcher：把这个 Notification/Mailable 推到队列，异步执行 | ✅ **是**，这才是异步开关 |

**Laravel 内部判断逻辑（伪代码）**：
```php
// Illuminate/Notifications/ChannelManager::sendNow() / send()
if ($notification instanceof ShouldQueue) {
    // 推到队列，异步执行
    $this->bus->dispatch(
        (new SendQueuedNotifications($notifiables, $notification))
            ->onConnection($notification->connection ?? null)
            ->onQueue($notification->queue ?? null)
    );
} else {
    // 同步执行，直接调用 toMail()/send()
    $this->sendNow($notifiables, $notification, $channels);
}
```

### 1.3 本项目的实际同步/异步分类

**同步执行（notify() 返回 = 邮件已完成发送，成功或失败）**：
- ✅ `Notification::route()->notify(new ReminderTriggered(...))` —— 提醒邮件
- ✅ `Mail::to(...)->send(new TestEmailSent(...))` —— 测试邮件
- ✅ `Mail::to(...)->send(new UserNotificationChannelEmailCreated(...))` —— 验证邮件（但 SendVerificationEmailChannel Job 本身是异步的）

**异步执行（notify()/Mail::send() 返回 = 邮件已入队，实际发送稍后）**：
- ✅ `Mail::to(...)->send(new UserInvited(...))` —— 邀请邮件（UserInvited implements ShouldQueue）

**外层 Job 本身异步的（和内部邮件发送同步/异步是两层）**：
- ✅ `ProcessScheduledContactReminders implements ShouldQueue` —— 整个提醒调度 Job 入队，但 Job 内部调用 notify() 时，ReminderTriggered 是同步发送
- ✅ `SendVerificationEmailChannel implements ShouldQueue` —— 整个验证 Job 入队，Job 内部调用 Mail::send() 时，UserNotificationChannelEmailCreated 是同步发送

### 1.4 两层队列：外层 Job vs 内部 Notification/Mailable

这是最容易混淆的地方。以提醒邮件为例：

```
┌─────────────────────────────────────────────────────────────┐
│ 第1层：ProcessScheduledContactReminders (ShouldQueue)        │
│   - 整个 handle() 方法是异步的，由 Worker 从 jobs 表领取执行  │
│   - 由每分钟调度触发 dispatch() 入队                         │
│                                                             │
│   ┌──────────────────────────────────────────────────────┐  │
│   │ 第2层：Notification::route()->notify(ReminderTriggered)│  │
│   │   - ReminderTriggered ❌ 不 implements ShouldQueue     │  │
│   │   - **同步执行**，在当前 Worker 进程内直接发送邮件      │  │
│   │   - 发送成功/失败都立即有结果，异常直接抛出            │  │
│   └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**结论**：ReminderTriggered 的邮件发送是**同步阻塞**的，在 ProcessScheduledContactReminders 的 handle() 执行线程内完成。

---

## 2. 队列三参数：retry_after、attempts、tries 的关系

### 2.1 参数来源与定义

| 参数 | 定义位置 | 含义 | 数据类型 |
|------|---------|------|---------|
| **tries** | Job类属性 `public int $tries` 或 Worker启动参数 `--tries=N` | **最多允许尝试次数**（含首次） | int |
| **attempts** | `jobs.attempts` 字段（DB） | **已实际尝试次数**，Worker每次领取自增 | unsignedTinyInteger |
| **retry_after** | `config/queue.php` 连接配置 | 任务被领取后，若 `reserved_at + retry_after < NOW()`，视为"Worker可能挂了"，可被其他Worker重新领取 | int（秒） |

**代码位置**：
- queue.php retry_after 配置：[config/queue.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/config/queue.php)
- Worker启动参数：[scripts/docker/queue.sh](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/scripts/docker/queue.sh)

### 2.2 tries 的优先级与本项目取值

tries 有三层来源，**优先级从高到低**：

```
1. Job类 $tries 属性          → 最高
   例：QueuableService::tries=1, SynchronizeAddressBooks::tries=1

2. Worker启动参数 --tries=N    → 默认兜底
   本项目：--tries=3
   影响：SendVerificationEmailChannel, ProcessScheduledContactReminders, PushVCard 等

3. 未设置以上两者              → Laravel默认值 1（本项目不会走到）
```

### 2.3 三者的交互时序（正常失败路径）

以 ProcessScheduledContactReminders（tries=3）为例，首次执行抛异常的完整路径：

```
T=0   调度触发 → dispatch(Job) → 写入 jobs 表
        jobs {id:1, attempts:0, reserved_at:null, available_at: T}
        ↓
T+Δ   Worker1 领取：
        UPDATE jobs SET attempts=1, reserved_at=T+Δ WHERE id=1
        ↑ attempts 此时为 1（首次执行 = 第1次尝试）
        ↓
      handle() 执行 → 抛出 Exception
        ↓
      检查：attempts (1) < tries (3) ?
        ├─ YES → 释放回队列
        │        UPDATE jobs SET
        │          reserved_at = null,
        │          available_at = NOW() + backoff(本项目=0)
        │        WHERE id=1
        │        ↓
        │      Worker2 领取：attempts=2, reserved_at=T2 ...
        │        ... 再失败 ...
        │        ↓
        │      Worker3 领取：attempts=3, reserved_at=T3 ...
        │        handle() 再抛异常
        │        ↓
        │      检查：attempts (3) < tries (3) ?  → NO
        └─ NO → 调用 Job::failed() 回调
                → INSERT INTO failed_jobs ...
                → DELETE FROM jobs WHERE id=1
```

**关键结论 1**：`attempts == tries` 时最后一次尝试仍会执行，失败后才进入 failed_jobs。

### 2.4 retry_after 静默重试（并发重复的根源）

这是三者中最容易被忽略、也是**重复发送风险最大**的机制。

#### 触发条件

```
reserved_at IS NOT NULL
AND
reserved_at + retry_after < NOW()
```

即：任务被某Worker领取了，但超过 `retry_after` 秒还没完成（没删除也没释放）。

#### 本项目的值：retry_after = 90 秒

```php
// config/queue.php
'database' => [
    'retry_after' => 90,
],
```

#### 静默重试的完整时序（ProcessScheduledContactReminders 处理大量数据时）

```
T=0    Worker1 领取：attempts=1, reserved_at=0
       ↓
       handle() 开始循环处理 500 条 scheduled 提醒
       （每条提醒都是同步发邮件，比较慢）
       处理到第 200 条时 ...
       ↓
T=90   (90秒后) retry_after 超时触发
       ↓
       Laravel 自动把这条 jobs 记录的 reserved_at = null
       （相当于"Worker1可能挂了，释放给别人"）
       ↓
T=91   Worker2 领取：
         UPDATE jobs SET attempts=2, reserved_at=T=91
         注意：attempts 被自增了（和正常重试一样）
       ↓
       此时 Worker1 和 Worker2 **同时在跑 handle()！**
       ├─ Worker1：继续处理第201~500条
       └─ Worker2：重新查询DB，处理所有 scheduled_at<=NOW() 的
           ↓
           两边的查询结果有交集 → **并发重复发送同一条提醒**
       ↓
T=180  (又过了90秒) 如果两个Worker都没跑完，可能再次触发 retry_after
       attempts=3，继续可能 Worker3 也加入
       ↓
T=X    当 attempts(3) == tries(3) 之后，若还没跑完并再触发 retry_after，
       此时会直接进入 failed_jobs（因为attempts不再小于tries）
```

**关键结论 2**：
- retry_after 触发时 `attempts 仍然会自增`
- 但 retry_after 可以在 attempts < tries 的范围内**无限重复触发并发执行**
- 直到 attempts 增加到 attempts >= tries，之后再触发 retry_after 就直接进 failed_jobs

#### 静默重试 vs 正常重试的区别

| 维度 | 正常异常重试 | retry_after 静默重试 |
|------|------------|-------------------|
| 触发时机 | handle() 抛出异常 | 执行超时 + reserved_at 过期 |
| attempts 自增 | ✅ 是 | ✅ 是 |
| 最大次数 | tries | tries - 1（减去已经占用的次数） |
| 是否并发 | 否（串行） | ✅ **是**（多个Worker同时执行同一份handle） |
| 对邮件的影响 | 最多重复 tries 次 | tries 次以内**并发重复** |

### 2.5 人工重试时的参数重置

执行 `php artisan queue:retry <uuid>` 后：
- 从 `failed_jobs.payload` 反序列化出原始Job
- 重新 INSERT 到 `jobs` 表，**attempts 被重置为 0**
- tries 限制重新生效（相当于又有 tries 次机会）

这意味着：人工重试可以绕开 tries 限制，**理论上可以无限次重试**。

---

## 3. notify() 是否同步？发送失败能否进 catch 和熔断器？（本次核心纠正）

### 3.1 结论先行：ReminderTriggered 完全同步，失败一定进 catch

这是之前分析最严重的错误，现在用代码事实严格证明：

**事实1**：[ReminderTriggered.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Notifications/ReminderTriggered.php) 不 `implements ShouldQueue`
```php
class ReminderTriggered extends Notification
{
    use Queueable;   // 只是 trait，不是接口
    // 没有 implements ShouldQueue ← 关键！
}
```

**事实2**：Laravel Notification 发送逻辑中，只有 `instanceof ShouldQueue` 才会入队，否则同步执行（见第1.2节）。

**推论**：
- `Notification::route('mail', ...)->notify(new ReminderTriggered(...))` → **同步阻塞执行**
- 邮件发送成功 → notify() 正常返回
- 邮件发送失败（SMTP 超时、网络错误等）→ **立即抛出异常**，直接进入 catch 块
- 熔断器计数 **正常生效**

### 3.2 notify() 的完整同步时序（逐行拆解）

以提醒邮件为例，无论 QUEUE_CONNECTION=sync 还是 database：

```
ProcessScheduledContactReminders::handle() 运行在 Worker 线程中
  ↓
foreach 某条 scheduled 记录
  ↓
  try {
    triggerNotification()
      ├─ Notification::route('mail', $channel->content)
      │    ->notify(new ReminderTriggered($channel, $label, $name))
      │
      │    Laravel Dispatcher 内部判断：
      │    ├─ ReminderTriggered instanceof ShouldQueue ? → ❌ 否
      │    └─ 走 sendNow() 分支，**同步执行**：
      │
      │        ① 调用 $notification->toMail($notifiable)
      │           → 执行 UserNotificationSent::create(...)  ← 写DB（发送前！）
      │           → 返回 MailMessage 对象
      │
      │        ② 调用 MailChannel::send()：
      │           → 从 MailMessage 构建 Swift_Message
      │           → 调用 Mailer::send()
      │           → 通过配置的 MAIL_MAILER（smtp/log/ses...）真正发送
      │
      │           ├─ 发送成功 → 正常返回，notify() 返回 null
      │           └─ 发送失败 → 抛出 Exception（TransportException 等）
      │
      ├─ （如果发送失败，下面代码不执行，直接跳到 catch）
      ├─ UPDATE contact_reminders SET number_times_triggered += 1
      └─ RescheduleContactReminderForChannel::execute()
           → UPDATE/DELETE contact_reminder_scheduled
    ↓
    updateScheduledContactReminderTriggeredAt()
      → UPDATE contact_reminder_scheduled SET triggered_at = NOW()
  }
  catch (\Exception $e) {
    ← **邮件发送失败会直接跳到这里**
    Log::error(...)
    UserNotificationSent::create(['error' => $e->getMessage(), ...])
    $channel->fails += 1                    ← 熔断器计数 +1
    if ($channel->fails >= 10) {
        $channel->active = false;           ← 熔断：禁用通道
        $channel->contactReminders->each->delete();  ← 清掉所有调度
    }
    $channel->save();
  }
```

### 3.3 为什么 Queue 驱动不影响 Notification 的同步性？

之前混淆了两个独立的概念：

| 概念 | 作用 | 影响对象 |
|------|------|---------|
| **QUEUE_CONNECTION** (sync/database/redis) | 决定 Job（implements ShouldQueue 的类）是同步执行还是推到队列 | ProcessScheduledContactReminders（外层Job）、UserInvited（implements ShouldQueue 的 Mailable） |
| **Notification 是否 implements ShouldQueue** | 决定 notify() 内部是同步执行还是把 SendQueuedNotifications Job 推到队列 | ReminderTriggered、其他 Notification |

**两层独立的队列决策**：

```
场景 A：QUEUE_CONNECTION=sync，ReminderTriggered 不 implements ShouldQueue
  → ProcessScheduledContactReminders dispatch() → 同步执行 handle()
    → handle() 内 notify(ReminderTriggered) → 同步执行 sendNow() → 同步发邮件
  → 全部同步，最直接

场景 B：QUEUE_CONNECTION=database，ReminderTriggered 不 implements ShouldQueue
  → ProcessScheduledContactReminders dispatch() → 推到 jobs 表
    → Worker 领取后执行 handle()
      → handle() 内 notify(ReminderTriggered) → 同步执行 sendNow() → 同步发邮件
                                                                   （在 Worker 进程内完成）
  → 外层 Job 异步，内层邮件发送同步

场景 C（假设）：ReminderTriggered implements ShouldQueue，QUEUE_CONNECTION=database
  → ProcessScheduledContactReminders dispatch() → 推到 jobs 表
    → Worker 领取后执行 handle()
      → handle() 内 notify(ReminderTriggered) → Laravel 检测到 ShouldQueue
        → dispatch(SendQueuedNotifications Job) → 再推一次 jobs 表
          → 另一个 Worker 领取后才真正发邮件
  → 两次入队，纯异步，handle() 里 catch 不到邮件发送异常

现实：本项目是 场景 B，不是 场景 C。
```

### 3.4 熔断器的实际生效范围

由于 notify() 是同步执行的，**熔断器完全有效**：

| 异常场景 | 是否进入 catch | channel->fails 是否 +1 | 连续10次是否熔断 |
|---------|--------------|---------------------|----------------|
| SMTP 连接失败 | ✅ 是 | ✅ 是 | ✅ 是 |
| 邮件被服务商拒收 | ✅ 是 | ✅ 是 | ✅ 是 |
| toMail() 内 UserNotificationSent::create() 失败（DB异常） | ✅ 是 | ✅ 是 | ✅ 是 |
| Reschedule 失败 | ✅ 是 | ✅ 是 | ✅ 是 |
| updateTriggeredAt 失败 | ✅ 是 | ✅ 是 | ✅ 是 |
| findOrFail 找不到记录 | ✅ 是 | ✅ 是 | ✅ 是 |

**注意**：`$this->updateNumberOfTimesTriggered()` 和 Reschedule 也是在 notify() 之后执行的。notify() 成功后（邮件已发出），后续步骤的异常**仍然会进入 catch**，导致熔断器计数，即使邮件已经成功发送。

### 3.5 验证邮件场景的两层队列

再看验证邮件的代码，更清楚地体现两层队列：

[SendVerificationEmailChannel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Settings/ManageNotificationChannels/Jobs/SendVerificationEmailChannel.php)：
```php
class SendVerificationEmailChannel implements ShouldQueue   // 第1层：整个Job异步
{
    public function handle()
    {
        Mail::to($this->channel->content)
            ->send(new UserNotificationChannelEmailCreated($this->channel));
            // 第2层：UserNotificationChannelEmailCreated 只 use Queueable
            // ❌ 不 implements ShouldQueue → 同步发送
    }
}
```

[UserNotificationChannelEmailCreated.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Mail/UserNotificationChannelEmailCreated.php)：
```php
class UserNotificationChannelEmailCreated extends Mailable
{
    use Queueable, SerializesModels;   // 只是 trait
    // 没有 implements ShouldQueue → 同步
}
```

时序：
```
触发 dispatch(SendVerificationEmailChannel)
  → QUEUE_CONNECTION=database → 推到 jobs 表
    → Worker 领取后执行 handle()
      → Mail::to(...)->send(new UserNotificationChannelEmailCreated(...))
        → Mailable 不 implements ShouldQueue → 同步构建 + 同步发送
        → 发送成功/失败都在当前 handle() 内抛出/返回
```

---

## 4. 提醒调度取数是否依赖触发标记？—— 不依赖

### 4.1 生产代码取数逻辑

**ProcessScheduledContactReminders::handle()**：[ProcessScheduledContactReminders.php#L38-L40](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L38-L40)

```php
$currentDate = Carbon::now();
$currentDate->second = 0;

$scheduledContactReminders = DB::table('contact_reminder_scheduled')
    ->where('scheduled_at', '<=', $currentDate)
    ->get();
```

**取数条件只有 `scheduled_at <= 当前时间`，完全没有 `triggered_at IS NULL`。**

### 4.2 triggered_at 字段的真实用途（三处使用均非生产调度）

| 场景 | 代码位置 | 用途 | 是否参与生产取数 |
|------|---------|------|----------------|
| 前端Vault展示 | [VaultShowViewHelper.php#L48](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L48) | 只展示"待触发"的即将到来的提醒 | ❌ |
| 测试命令（非生产） | [TestReminders.php#L44](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Console/Commands/TestReminders.php#L44) | `where('triggered_at', null)` 手动触发用 | ❌ |
| 审计记录 | [ProcessScheduledContactReminders.php#L83-L85](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L83-L85) | `updateScheduledContactReminderTriggeredAt()` 设置触发时间 | ❌（只写不读） |

---

## 5. 循环提醒怎么重排？—— 更新同一条记录的 scheduled_at，且不重置 triggered_at

### 5.1 处理流程与执行顺序

```
取数（scheduled_at <= NOW()）
    ↓
foreach 每条记录
    ├─ try {
    │   ├─ 查询 ContactReminder, Contact
    │   ├─ triggerNotification()
    │   │   ├─ if (!channel->active) return
    │   │   ├─ Notification::route()->notify()      ← ① 发送（同步！）
    │   │   ├─ increment number_times_triggered      ← ② 计数+1
    │   │   └─ RescheduleContactReminderForChannel   ← ③ 重排 scheduled_at
    │   └─ updateScheduledContactReminderTriggeredAt() ← ④ 设 triggered_at = NOW()
    │
    └─ } catch { ... }
```

**顺序：①发送 → ②计数 → ③Reschedule（改scheduled_at/删记录） → ④triggered_at**

### 5.2 Reschedule 只改 scheduled_at，不改 triggered_at

**RescheduleContactReminderForChannel::schedule()**：[RescheduleContactReminderForChannel.php#L190-L218](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannel.php#L190-L218)

```php
private function schedule(): void
{
    $record = DB::table('contact_reminder_scheduled')
        ->where('id', $this->data['contact_reminder_scheduled_id'])
        ->first();
    $this->upcomingDate = Carbon::createFromFormat('Y-m-d H:i:s', $record->scheduled_at);

    // +1天 / +1月 / +1年（基于原 scheduled_at 计算，非当前时间）
    switch ($this->contactReminder->type) { ... }

    // 只传 scheduled_at，不传 triggered_at
    $this->contactReminder->userNotificationChannels()
        ->syncWithoutDetaching([
            $this->userNotificationChannel->id => [
                'scheduled_at' => $this->upcomingDate,
                // 注意：没有 triggered_at => null  ← 关键！
            ]
        ]);
}
```

**syncWithoutDetaching 的行为**：对传入的 pivot 字段做 UPDATE，**不传的字段保持原值不变**。
所以 Reschedule 之后，triggered_at 仍保持之前的值（由后面的 updateTriggeredAt 设置为 NOW()）。

### 5.3 一次性提醒 vs 循环提醒的区别

| 类型 | Reschedule 行为 | triggered_at |
|------|----------------|-------------|
| TYPE_ONE_TIME | DELETE 整条记录 | ——（记录已删除） |
| TYPE_RECURRING_* | UPDATE scheduled_at → 未来，triggered_at 不变 | 随后被设为 NOW() |

### 5.4 真正的防重机制

防重复发送不靠 triggered_at，而是：
> **Reschedule 把 scheduled_at 推到未来，使下一次查询 `scheduled_at <= NOW()` 不再命中。**

脆弱点：如果 ①发送成功 但 ③Reschedule 没执行（进程被杀/异常），scheduled_at 仍是过去时间，下次调度（每分钟）会再次命中。

---

## 6. 三大边界深度分析：发送时序 / 并发取数 / 发送记录可信度

### 6.1 发送已发生但触发标记未写入？—— 四层操作时序

首先明确：因为 notify() 是同步执行的，**notify() 返回就意味着邮件已经发送成功（或者抛出了异常）**。

所以之前"database 驱动下 notify() 返回 ≠ 邮件已发出"是错误的。实际有四层关键操作，每一层之间都可能中断，产生不同的边界状态：

```
Layer 1: notify() 同步发送通知
  └─ Notification::route('mail', ...)->notify(new ReminderTriggered(...))
       ├─ toMail() 执行 → UserNotificationSent::create()  （写DB，发送前！）
       └─ MailChannel 同步调用邮件驱动发送
            ├─ 成功 → notify() 正常返回
            └─ 失败 → 抛出异常 → 直接进入 catch

Layer 2: increment number_times_triggered
  └─ UPDATE contact_reminders SET number_times_triggered = number_times_triggered + 1

Layer 3: RescheduleContactReminderForChannel
  └─ syncWithoutDetaching 更新 scheduled_at（循环提醒）或 DELETE（一次性提醒）

Layer 4: updateTriggeredAt
  └─ UPDATE contact_reminder_scheduled SET triggered_at = NOW()
```

**关键事实**：
- UserNotificationSent 写在 toMail() 内（Layer 1 内部），发生在**真正发送邮件之前**
- Layer 1 成功（邮件已发出）后，后续 Layer 2/3/4 任一失败，都会抛出异常 → 进入 catch → 熔断器计数

#### 各层中断后的边界状态

| 中断点 | 邮件状态 | UserNotificationSent | number_times_triggered | scheduled_at | triggered_at | 下次是否重复 | catch是否执行 | 熔断器是否+1 |
|--------|---------|---------------------|----------------------|-------------|-------------|------------|------------|-----------|
| Layer 1 之前（未调用 notify） | 未发 | ❌ 无 | ❌ 未变 | 过去 | null | ✅ 是（正常） | — | — |
| Layer 1 中（toMail已执行，发送中失败） | 未发/部分发 | ✅ 有（error=null） | ❌ 未变 | 过去 | null | ✅ 是 | ✅ 是 | ✅ 是 |
| Layer 1 之后（notify 正常返回） | ✅ 已发出 | ✅ 有（error=null） | ❌ 未变 | 过去 | null | ❌ 重复发送 | — | — |
| Layer 2 之后 | ✅ 已发出 | ✅ 有 | ✅ +1 | 过去 | null | ❌ 重复发送 | — | — |
| Layer 3 之后（Reschedule） | ✅ 已发出 | ✅ 有 | ✅ +1 | 未来 | null | ✅ 否 | — | — |
| Layer 4 之后（triggered_at） | ✅ 已发出 | ✅ 有 | ✅ +1 | 未来 | 有值 | ✅ 否 | — | — |
| Layer 1 成功但 Layer 2/3/4 中 DB 异常 | ✅ 已发出 | ✅ 有 | 看中断在哪 | 看中断在哪 | 看中断在哪 | ❓可能重复 | ✅ 是 | ✅ 是 |

**最危险边界**：
1. **Layer 1 后、Layer 3 前中断**（进程被杀/DB死锁）→ 邮件已发但 scheduled_at 还是过去 → 下次调度**重复发送**
2. **Layer 1 成功但后续步骤 DB 异常** → 邮件已发出，但 catch 被触发 → **熔断器错误计数**（实际发成功了，但被记为失败），连续10次可能把正常通道误熔断

---

### 6.2 并发取数竞态—— 两个来源导致同批数据被重复处理

并发取数的问题有两个来源：

#### 来源1：retry_after 静默重试导致的并发

回顾：`retry_after = 90`，如果 Job 执行超过90秒，Laravel 会认为 Worker 死了，自动把任务放回队列，另一个 Worker 会领取并执行。

对于 `ProcessScheduledContactReminders`：
- 每分钟调度一次
- 每条提醒都是**同步发邮件**（SMTP 握手+传输，假设每条 0.5~2 秒）
- 90 秒内能处理约 45~180 条
- 如果某批提醒超过 180 条，处理时间超过 90 秒 → retry_after 触发
- 新 Worker 并发执行**同一批数据**
- 取数 SQL：`WHERE scheduled_at <= NOW()`，**没有任何锁**
- 两个 Worker 同时取到同一份数据 → 同时发送 → 重复邮件

**这里需要注意**：因为邮件发送是同步的，Worker 处理速度受邮件服务商响应速度影响很大。如果服务商响应慢（比如每条 5 秒），18 条就够触发 retry_after 了。

#### 来源2：每分钟调度叠加 retry_after 重试

更极端的情况：
- 第0分钟：Job A 启动，开始处理大量提醒
- 第1分钟：调度器又触发 Job B（因为每分钟一次），此时 Job A 还在运行
- 第1.5分钟：Job A 还没处理完，scheduled_at 还是过去，Job B 也取到了同样的数据
- 两个 Job 并发处理同批数据

#### syncWithoutDetaching 的并发行为

如果两个 Worker 同时处理同一条提醒，且都走到了 Reschedule：

```
Worker A: syncWithoutDetaching → scheduled_at = Day2
Worker B: syncWithoutDetaching → scheduled_at = Day2 （相同的结果，因为都是基于原 scheduled_at 计算）
```

两个都执行 syncWithoutDetaching，结果是一样的（因为计算出的 upcomingDate 相同），所以 scheduled_at 不会有问题。

但问题在于：**两个 Worker 都会执行 notify()**，导致重复发送邮件。因为邮件发送是同步的，而且在 Reschedule 之前执行。

更坏的情况：Worker A 执行到 Reschedule 之后（scheduled_at 已推到未来），Worker B 才开始取数。

- 如果 Worker A 的事务还没提交，Worker B 可能还是读到旧的 scheduled_at（过去）→ 重复处理
- 如果 Worker A 的事务已提交，Worker B 读到新的 scheduled_at（未来）→ 不重复

**注意**：当前代码没有显式事务。DB::table() 的 update 是单语句事务（auto-commit），所以 Reschedule 的 scheduled_at 更新提交后，其他连接立刻可见。但问题是 **notify() 在 Reschedule 之前执行**，即使 Reschedule 很快提交，Worker A 的 notify() 和 Worker B 的 notify() 已经并行发生了。

**结论**：取数无锁 + 多并发源 + notify 在 Reschedule 之前 → 重复发送是可能的，概率取决于提醒数量和邮件发送速度。

---

### 6.3 发送记录可信度—— UserNotificationSent 不能证明邮件真的发出去了

这部分之前的分析基本正确，现在补充纠正后的时序细节。

#### UserNotificationSent 的写入时机

**ReminderTriggered::toMail()**：[ReminderTriggered.php#L52-L67](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Notifications/ReminderTriggered.php#L52-L67)

```php
public function toMail($notifiable)
{
    UserNotificationSent::create([
        'user_notification_channel_id' => $this->channel->id,
        'sent_at' => Carbon::now(),
        'subject_line' => $this->content,
    ]);  // ← 写在发送前！

    return (new MailMessage)->subject(...)->line(...);
}
```

**写在 toMail() 里，返回 MailMessage 之前**。也就是说：

> UserNotificationSent 记录存在 ≠ 邮件发送成功
> UserNotificationSent 记录存在 = 邮件**准备发送**的日志被写了

#### 正常成功路径下的记录

```
notify() 调用
  → toMail() 执行 → UserNotificationSent::create() 成功（error=null）
  → 返回 MailMessage
  → 邮件驱动真正发送（同步）
  → 发送成功 → notify() 返回
```

结果：**1 条 error=null 的记录**，和邮件真实状态一致（发送成功）。

#### 发送失败路径下的记录

```
notify() 调用
  → toMail() 执行 → UserNotificationSent::create() 成功（error=null）
  → 返回 MailMessage
  → 邮件驱动真正发送（同步）
  → 发送失败 → 抛出 Exception
  → 进入 catch 块
  → 又写一条 UserNotificationSent（error=错误信息）
```

结果：**两条记录**
- 一条 error=null（假成功，toMail 里写的）
- 一条 error=xxx（真失败，catch 里写的）

#### toMail 里的 create 本身失败

如果 UserNotificationSent::create() 失败（比如 DB 连接超时）：

```
notify() 调用
  → toMail() 执行
    → UserNotificationSent::create() 失败 → 抛出 Exception
    → 还没返回 MailMessage，邮件还没发
  → 异常直接冒泡 → 进入 catch 块
  → catch 里尝试再写一条 UserNotificationSent（error=错误信息）
    → 如果 DB 还挂着 → 这条也写失败 → 0 条记录
    → 如果 DB 恢复了 → 1 条 error=xxx 的记录
```

#### 边界：邮件发送成功但后续步骤异常

```
notify() 调用 → 邮件发送成功 ✅
  → Layer 2: updateNumberOfTimesTriggered → DB 死锁，抛出异常
  → 进入 catch
  → 写一条 UserNotificationSent（error=死锁信息）
```

结果：**两条记录**
- 一条 error=null（toMail 里写的，实际对应成功发送的邮件）
- 一条 error=xxx（catch 里写的，对应后续步骤失败）

#### 各种场景下的可信度矩阵

| 场景 | UserNotificationSent 记录 | 邮件实际状态 | 熔断器是否计数 | 说明 |
|------|---------------------|-------------|--------------|------|
| 完全成功 | 1条 error=null | ✅ 成功 | ❌ 否 | 一致 |
| 发送失败 | 2条（1条null+1条error） | ❌ 失败 | ✅ 是 | 需过滤 error 字段判断 |
| toMail 内 create 失败 | 0或1条（error） | ❌ 未发 | ✅ 是 | 可能无记录 |
| 发送成功但后续DB异常 | 2条（1条null+1条error） | ✅ 成功 | ✅ 是（**误计数**） | 熔断器误判 |
| Layer 1成功后进程被杀 | 1条 error=null | ✅ 成功 | ❌ 否 | 记录存在，但下次可能重复 |

**结论**：UserNotificationSent 不能直接用来判断邮件是否发送成功。需要结合 error 字段和记录条数综合判断，且存在"发送成功但熔断器误计数"的边界情况。

---

## 7. triggered_at 过滤会不会影响循环提醒？—— 会！（加过滤必须同时重置）

### 7.1 初版建议的错误：只加过滤不重置 = 循环提醒只触发一次

**之前的错误建议**：
```php
// ❌ 只加这行会出问题！
$scheduledContactReminders = DB::table('contact_reminder_scheduled')
    ->where('scheduled_at', '<=', $currentDate)
    ->whereNull('triggered_at')   // ← 只加这行
    ->get();
```

**会发生什么（以日循环提醒为例）**：

```
Day1 (2024-01-01)
  初始状态：scheduled_at=2024-01-01 09:00, triggered_at=null
  取数：WHERE scheduled_at <= 09:00 AND triggered_at IS NULL → 命中 ✅
  发送 → Reschedule: scheduled_at=2024-01-02 09:00 → triggered_at=2024-01-01 09:00
  最终状态：scheduled_at=2024-01-02 09:00, triggered_at=2024-01-01 09:00

Day2 (2024-01-02)
  状态：scheduled_at=2024-01-02 09:00, triggered_at=2024-01-01 09:00
  取数：WHERE scheduled_at <= 09:00 AND triggered_at IS NULL
        → triggered_at 有值 = 2024-01-01 09:00 → 不匹配 → ❌ 不命中！
  结果：这一天不发送

Day3 (2024-01-03)
  状态同上，仍不命中 ...
  结果：永远不再发送 = 循环提醒变成一次性提醒！
```

**根因**：Reschedule 只改 scheduled_at，**不重置 triggered_at**。第一次设了 triggered_at = NOW() 之后就再也清不掉了。

### 7.2 为什么 TestReminders 有过滤却能正常工作？

对比看 [TestReminders.php#L43-L71](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Console/Commands/TestReminders.php#L43-L71)：

```php
// TestReminders 有 triggered_at 过滤
$scheduledContactReminders = DB::table('contact_reminder_scheduled')
    ->where('triggered_at', null)
    ->get();

foreach (...) {
    Notification::route(...)->notify(...);   // 发送

    (new RescheduleContactReminderForChannel)->execute([...]);  // Reschedule
    // ↑ 注意：TestReminders 调用了 Reschedule
    // ↑ 注意：TestReminders **没有调用 updateScheduledContactReminderTriggeredAt()**
    // 所以 triggered_at 永远保持 null
}
```

TestReminders 有过滤但**从来不设置 triggered_at**，所以不存在被卡死的问题。但它也因此**失去了 triggered_at 作为幂等边界的意义**。

### 7.3 正确的 triggered_at 幂等方案：过滤 + Reschedule 时重置 + 调整顺序

如果要用 triggered_at 作为幂等边界（防 retry_after 并发重复、防人工重试），必须**同时改三处**：

#### 修改1：取数加过滤（ProcessScheduledContactReminders）

```php
$scheduledContactReminders = DB::table('contact_reminder_scheduled')
    ->where('scheduled_at', '<=', $currentDate)
    ->whereNull('triggered_at')          // ← 加这行
    ->get();
```

#### 修改2：Reschedule 时重置 triggered_at（RescheduleContactReminderForChannel::schedule）

```php
$this->contactReminder->userNotificationChannels()
    ->syncWithoutDetaching([
        $this->userNotificationChannel->id => [
            'scheduled_at' => $this->upcomingDate,
            'triggered_at' => null,      // ← 加这行：把 triggered_at 清空
        ]
    ]);
```

#### 修改3：把 updateTriggeredAt 移到 Reschedule 之前执行

即把顺序从 `发送 → 计数 → Reschedule → updateTriggeredAt` 改为 `发送 → 计数 → updateTriggeredAt → Reschedule`

```php
// 在 triggerNotification() 内：
private function triggerNotification(...)
{
    ...
    Notification::route(...)->notify(...);   // ① 发送（同步）
    $this->updateNumberOfTimesTriggered(...); // ② 计数

    // 把 triggered_at 设置移到这里（Reschedule 之前）
    $this->updateScheduledContactReminderTriggeredAt($scheduledReminder); // ③ 先标记已触发

    // 最后才 Reschedule（会重置 triggered_at = null，因为 scheduled_at 已经推到未来了）
    (new RescheduleContactReminderForChannel)->execute([...]); // ④ 重排 + 重置 triggered_at=null
}

// try 块里就不再重复调用 updateTriggeredAt 了
```

**改完后的时序**：

```
Day1:
  初始：scheduled=01-01 09:00, triggered=null
  取数：✅ 命中
  ① 发送
  ② 计数
  ③ updateTriggeredAt → triggered=01-01 09:00（有值）
  ④ Reschedule → scheduled=01-02 09:00, triggered=null（被重置回 null）
  最终：scheduled=01-02 09:00, triggered=null ← 关键：最终 triggered_at 是 null！

Day2:
  状态：scheduled=01-02 09:00, triggered=null
  取数：scheduled<=09:00 AND triggered IS NULL → ✅ 命中！
  ... 循环继续
```

**三处修改缺一不可**：

| 修改 | 作用 | 不加的后果 |
|------|------|----------|
| 1. 取数加 `triggered_at IS NULL` | 幂等边界：同一次调度不重复处理 | retry_after 并发重复、人工重试重复 |
| 2. Reschedule 同步重置 `triggered_at = null` | 下一周期可重新命中 | 循环提醒只触发一次（Day2起不命中） |
| 3. 把 updateTriggeredAt 移到 Reschedule 之前 | 先标记"本周期已处理"，再重置"下周期待处理" | Reschedule 刚清空，updateTriggeredAt 又覆盖为有值 → Day2 仍不命中 |

---

## 8. 人工重试会不会重复发送？—— 分情况（含并发场景）

### 8.1 ProcessScheduledContactReminders 的特性

这个Job无构造参数、每次 handle() 都重新查询DB。人工重试 = 重新跑一遍完整的取数+处理逻辑。

### 8.2 各场景判定矩阵

| 场景 | Reschedule 状态 | scheduled_at 状态 | channel 状态 | 人工重试结果 | 最多重复 |
|------|----------------|------------------|-------------|-------------|---------|
| A：发送前异常 | 未执行 | 过去 | active | ✅ 不重复（没发出去） | 0 |
| B：正常完成 | 成功 | 未来/已删 | — | ✅ 不重复（取数命中不了） | 0 |
| C：发送成、Reschedule败，catch住 | 失败 | 过去 | active | ❌ **重复**（每分钟调度+人工重试都会命中），直到熔断 | ≤10次 |
| D：发送成、Reschedule成，进程被杀 | 成功 | 未来 | — | ✅ 不重复 | 0 |
| E：发送成、Reschedule未执行，进程被杀 | 未执行 | 过去 | active | ❌ **重复** | ≤3(tries)×N |
| F：retry_after 并发 + Reschedule 部分成功 | 不同Worker执行到不同阶段 | — | — | ❌ **不确定**（取决于哪条Worker先改DB） | 高并发下难预估 |

### 8.3 加上 triggered_at 幂等方案后的人工重试

假设已实施了三处修改：

```
第一次执行（部分成功）：
  → 发送了提醒1、2（成功）
  → triggered_at 分别被设为 NOW()
  → Reschedule 重置 triggered_at=null 之前进程被杀
  → 最终：提醒1、2的 scheduled_at=过去, triggered_at=有值（没被Reschedule重置）
  → 提醒3、4的 scheduled_at=过去, triggered_at=null（还没处理到）

人工重试时：
  取数：WHERE scheduled_at<=NOW() AND triggered_at IS NULL
    → 提醒1、2：triggered_at 有值 → ❌ 不命中 → ✅ 不重复
    → 提醒3、4：triggered_at 是 null → ✅ 命中 → 正常发送
```

**结论**：三处修改都加之后，人工重试**只会补发没处理过的记录**，不会重复发送已经成功发送的。

---

## 9. 其他邮件任务的重试与恢复

### 9.1 验证邮件（SendVerificationEmailChannel）

**两层队列**：
- 外层 Job：`SendVerificationEmailChannel implements ShouldQueue` → 异步入队（tries=3）
- 内层 Mailable：`UserNotificationChannelEmailCreated extends Mailable` → **同步发送**（不 implements ShouldQueue）

所以：
- handle() 内的 Mail::send() 是同步执行
- 发送失败 → 异常抛出 → 进入 Job 重试逻辑（attempts++，放回队列）
- 重试 3 次全失败 → 进入 failed_jobs
- 无幂等检查：不检查 `verified_at`，不检查发送记录 → 每次重试都会重新发邮件
- 人工重试 → 反序列化 $channel → 肯定重复发送
- 缓解：添加 `verified_at` 检查

### 9.2 邀请邮件（UserInvited）

- Mailable `implements ShouldQueue` → 异步入队
- `Mail::to(...)->send(new UserInvited(...))` 只是把 SendQueuedMailable Job 推到队列
- 无 tries、无幂等、人工重试必重复

### 9.3 测试邮件（SendTestEmail Service）

- 同步执行，不走队列，无重试问题
- TestEmailSent Mailable 不 implements ShouldQueue → 同步发送
- Mail::send() 成功/失败都在当前请求内完成

---

## 10. 风险矩阵与关键风险点

### 10.1 重复发送风险矩阵

| 任务类型 | 内层发送 | 外层异步 | 幂等防护 | retry_after 并发风险 | 人工重试是否重复 |
|---------|---------|---------|----------|-------------------|-----------------|
| 提醒邮件 ProcessScheduledContactReminders | **同步**（ReminderTriggered 无 ShouldQueue） | ✅ 是 | Reschedule推scheduled_at（脆弱） | ⚠️ 高（无悲观锁，SMTP慢易超时） | 视 Reschedule 成功否 |
| 验证邮件 SendVerificationEmailChannel | **同步**（Mailable 无 ShouldQueue） | ✅ 是 | 无 | ⚠️ 中（单任务执行快） | ✅ 是 |
| 邀请邮件 UserInvited | **异步**（Mailable implements ShouldQueue） | 本身就是异步 Mailable | 无 | ⚠️ 中（单任务执行快） | ✅ 是 |
| DAV同步类 | — | ✅ 是 | etag/幂等接口 | ✅ 低 | ✅ 否 |

### 10.2 最高风险：ProcessScheduledContactReminders 的 retry_after 并发

- 每分钟调度一次，每条提醒同步发邮件（SMTP 0.5~5秒/条）
- 只要一批提醒 > 18~180 条（取决于 SMTP 速度），处理就可能超过 retry_after=90 秒
- retry_after 触发后 attempts 自增，新 Worker 并发执行同一批数据
- **没有悲观锁（FOR UPDATE SKIP LOCKED）、没有 triggered_at 条件、没有唯一键校验**
- 结果：同一批提醒可能被2~3个Worker并发处理 → 群发重复邮件

**缓解（低成本）**：
1. 给 ProcessScheduledContactReminders 加 `$timeout = 60`（小于 retry_after=90，超时直接杀进程而非并发）
2. 取数SQL改用悲观锁（database驱动支持），但需要 DB::transaction() 包裹

### 10.3 次高风险：熔断器误计数

由于邮件发送是同步的，且在 Layer 1 成功后还有 Layer 2/3/4 的 DB 操作：

- Layer 1：notify() 成功 → 邮件已发出 ✅
- Layer 2/3/4：DB 操作偶尔死锁/超时 → 抛异常 → 进入 catch → channel->fails++
- 连续 10 次这种情况 → 正常的通知通道被**误熔断**（实际邮件都发成功了）

概率：DB 偶尔抖动时可能发生。

---

## 11. 代码走向流程图（完整纠正版）

### 11.1 循环提醒的一次完整生命周期（纠正：notify 同步）

```
[初始状态]
  contact_reminder_scheduled:
    id=123, channel_id=7, reminder_id=42
    scheduled_at = 2024-01-01 09:00:00
    triggered_at = null

Schedule 每分钟触发 dispatch(ProcessScheduledContactReminders)
  QUEUE_CONNECTION=database → 推到 jobs 表
    ↓
Worker 领取 jobs，执行 handle()
  ↓
  DB查询: WHERE scheduled_at <= NOW()
          ↑ 注意：没有 triggered_at IS NULL 条件！
    ↓
  命中 id=123
    ↓
  try {
    findOrFail(ContactReminder=42, UserNotificationChannel=7)
      ↓
    triggerNotification()
      ├─ if (!channel->active) return
      ├─ Notification::route('mail', ...)->notify(ReminderTriggered)
      │     ├─ Laravel 判断：ReminderTriggered instanceof ShouldQueue? → ❌ 否
      │     ├─ 同步执行 sendNow()
      │     │   ├─ toMail(): UserNotificationSent::create()  ← 写日志（发送前！）
      │     │   ├─ 返回 MailMessage
      │     │   └─ MailChannel 同步调用 SMTP 发送
      │     │        ├─ 成功 → notify() 返回
      │     │        └─ 失败 → 抛异常 → 跳到 catch
      │     └─ （以下代码只有发送成功才执行）
      ├─ UPDATE contact_reminders SET number_times_triggered += 1
      └─ RescheduleContactReminderForChannel::execute()
            ├─ TYPE != ONE_TIME:
            │   原 scheduled_at = 2024-01-01
            │   upcomingDate = 2024-01-02 (+1 day)
            │   syncWithoutDetaching:
            │     UPDATE pivot SET scheduled_at = 2024-01-02
            │           (triggered_at 不变，仍是 null)
            └─ TYPE == ONE_TIME:
                  DELETE FROM pivot WHERE id=123
    ↓
    updateScheduledContactReminderTriggeredAt():
      UPDATE pivot SET triggered_at = NOW() WHERE id=123
  }
  catch (Exception $e) {
    ← **邮件发送失败、后续DB失败都跳到这里**
    Log::error
    UserNotificationSent::create(error => ...)
    channel->fails++
    fails >= 10 → channel->active = false + 删除其所有调度
    channel->save()
  }

[最终状态（循环提醒）]
  scheduled_at = 2024-01-02 09:00:00  ← 推到未来
  triggered_at = 2024-01-01 09:00:xx  ← 标记这次触发时间
```

---

## 12. 优化建议（纠正版）

### 12.1 短期优化（低风险）

#### ① 给 ProcessScheduledContactReminders 加 `$timeout = 60`

```php
class ProcessScheduledContactReminders implements ShouldQueue
{
    public int $timeout = 60;  // 小于 retry_after=90，避免并发
    ...
}
```
60秒超时直接杀进程，不是并发多个Worker。代价是：大量提醒时部分提醒可能在被杀时没处理完（但有tries=3兜底，下次重试）。

#### ② 验证邮件添加 `verified_at` 检查

```php
// SendVerificationEmailChannel::handle()
if ($this->channel->verified_at !== null) {
    return;
}
```

#### ③ 取数加悲观锁（避免并发Worker重复处理）

```php
DB::transaction(function () use ($currentDate) {
    $scheduledContactReminders = DB::table('contact_reminder_scheduled')
        ->where('scheduled_at', '<=', $currentDate)
        ->lockForUpdate()    // 事务内加悲观锁
        ->get();
    // ... 处理 ...
});
```
并发Worker会排队而不是重复取数。注意：需要整个处理逻辑包在事务内才有意义，否则 lockForUpdate 执行后立刻释放。

### 12.2 中期优化（需同时修改多处）

#### ④ 三处联动修改，引入 triggered_at 作为幂等边界

**修改1**：取数加 `triggered_at IS NULL`

**修改2**：RescheduleContactReminderForChannel::schedule() 在 syncWithoutDetaching 时同步设置 `triggered_at => null`

**修改3**：在 ProcessScheduledContactReminders 中，把 `updateScheduledContactReminderTriggeredAt()` 的调用从 try 块末尾移到 `Reschedule` 调用之前（即 triggerNotification 方法内）

三处必须联动，缺一不可（详见第7.3节）。

#### ⑤ contact_reminder_scheduled 表加唯一索引

```sql
ALTER TABLE contact_reminder_scheduled
ADD UNIQUE KEY uk_channel_reminder
  (user_notification_channel_id, contact_reminder_id);
```
防止 syncWithoutDetaching 在并发下产生重复 pivot 记录（目前代码逻辑上不会重复，但表结构无约束）。

#### ⑥ 修复熔断器误计数问题

把 Layer 1（邮件发送）和 Layer 2/3/4（DB 更新）分开 try-catch：

```php
try {
    // Layer 1：邮件发送
    Notification::route(...)->notify(new ReminderTriggered(...));
} catch (\Exception $e) {
    // 只有邮件发送失败才熔断器计数
    Log::error('Reminder email send failed', ...);
    UserNotificationSent::create(['error' => $e->getMessage(), ...]);
    $channel->fails++;
    if ($channel->fails >= 10) { ... }
    $channel->save();
    continue;  // 跳过这条，继续处理下一条
}

// 邮件已发送成功，后续 DB 操作单独 try-catch
try {
    $this->updateNumberOfTimesTriggered(...);
    (new RescheduleContactReminderForChannel)->execute(...);
    $this->updateScheduledContactReminderTriggeredAt(...);
} catch (\Exception $e) {
    // DB 更新失败，记录日志，但**不熔断器计数**（邮件已经发了）
    Log::error('Reminder post-send DB update failed', ...);
}
```

### 12.3 长期优化（高风险，架构调整）

- 发件箱模式（Transactional Outbox）：把"标记已发送"和"发送邮件"用事务保证一致性
- 邮件服务商 Webhook 追踪投递状态：只有收到服务商的 delivered 回调才算发送成功
- Redis 分布式锁防同一份提醒并发处理
- 发送记录（UserNotificationSent）移到发送成功回调中写入，而不是 toMail() 里

---

## 13. 相关文件索引

| 类型 | 文件路径 | 关键内容 |
|------|---------|----------|
| 队列配置 | [config/queue.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/config/queue.php) | retry_after=90, default=sync |
| Worker脚本 | [scripts/docker/queue.sh](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/scripts/docker/queue.sh) | --tries=3 |
| 提醒Notification | [ReminderTriggered.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Notifications/ReminderTriggered.php) | use Queueable, **无 ShouldQueue** → 同步发送 |
| 邀请邮件Mailable | [UserInvited.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Mail/UserInvited.php) | implements ShouldQueue → 异步 |
| 测试邮件Mailable | [TestEmailSent.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Mail/TestEmailSent.php) | use Queueable, **无 ShouldQueue** → 同步 |
| 验证邮件Mailable | [UserNotificationChannelEmailCreated.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Mail/UserNotificationChannelEmailCreated.php) | use Queueable, **无 ShouldQueue** → 同步 |
| 提醒调度Job | [ProcessScheduledContactReminders.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php) | implements ShouldQueue, 取数无triggered_at条件, notify 同步 |
| 验证邮件Job | [SendVerificationEmailChannel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Settings/ManageNotificationChannels/Jobs/SendVerificationEmailChannel.php) | implements ShouldQueue, 内部 Mail::send 同步 |
| 重排服务 | [RescheduleContactReminderForChannel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannel.php) | syncWithoutDetaching 只改scheduled_at |
| 测试命令 | [TestReminders.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Console/Commands/TestReminders.php) | 有triggered_at过滤，但不调用updateTriggeredAt |
| QueuableService基类 | [QueuableService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Services/QueuableService.php) | tries=1, implements ShouldQueue |
| 熔断器配置 | [config/monica.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/config/monica.php) | max_notification_failures=10 |
| .env示例 | [.env.example](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/.env.example) | QUEUE_CONNECTION=sync, MAIL_MAILER=log |
