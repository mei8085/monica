# 异步任务与邮件失败恢复代码分析

---

## 关键纠正说明（含三次修正）

| 初版错误 | 首次纠正 | 二次补充 | 三次补充（本次） |
|---------|---------|---------|----------------|
| 取数含 `triggered_at IS NULL` | 取数只看 `scheduled_at <= NOW()` | —— | —— |
| triggered_at 是主防护 | triggered_at 仅审计/前端用 | —— | —— |
| 循环提醒创建新记录 | 更新同一条记录的 scheduled_at | Reschedule 只改 scheduled_at，**不重置 triggered_at** | —— |
| 一次性提醒标记完成 | 一次性提醒直接 DELETE | —— | —— |
| Reschedule 在 triggered_at 之后 | 顺序：发送 → Reschedule → updateTriggeredAt | —— | —— |
| 加 triggered_at 过滤就够了 | —— | ❌ 会导致循环提醒只触发一次，需三处联动修改 | —— |
| 通知同步发送，notify() 返回 = 邮件已发出 | —— | —— | ❌ 通知 use Queueable，队列驱动下 notify() 只是入队，邮件发送是异步的 |
| UserNotificationSent 是"已发送"记录 | —— | —— | ❌ **写在 toMail() 里，发生在实际发送之前**，不能证明发送成功 |
| catch 块中 error 记录和成功记录二选一 | —— | —— | ❌ sync 驱动下可能两条都有：toMail 先写一条"成功"记录，发送失败后 catch 又写一条 error 记录 |

---

## 1. 队列三参数：retry_after、attempts、tries 的关系

### 1.1 参数来源与定义

| 参数 | 定义位置 | 含义 | 数据类型 |
|------|---------|------|---------|
| **tries** | Job类属性 `public int $tries` 或 Worker启动参数 `--tries=N` | **最多允许尝试次数**（含首次） | int |
| **attempts** | `jobs.attempts` 字段（DB） | **已实际尝试次数**，Worker每次领取自增 | unsignedTinyInteger |
| **retry_after** | `config/queue.php` 连接配置 | 任务被领取后，若 `reserved_at + retry_after < NOW()`，视为"Worker可能挂了"，可被其他Worker重新领取 | int（秒） |

**代码位置**：
- queue.php retry_after 配置：[config/queue.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/config/queue.php)
- Worker启动参数：[scripts/docker/queue.sh](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/scripts/docker/queue.sh)
- jobs表attempts字段：[2022_01_22_183321_create_jobs_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/database/migrations/2022_01_22_183321_create_jobs_table.php)

### 1.2 tries 的优先级与本项目取值

tries 有三层来源，**优先级从高到低**：

```
1. Job类 $tries 属性          → 最高
   例：QueuableService::tries=1, SynchronizeAddressBooks::tries=1

2. Worker启动参数 --tries=N    → 默认兜底
   本项目：--tries=3
   影响：SendVerificationEmailChannel, ProcessScheduledContactReminders, PushVCard 等

3. 未设置以上两者              → Laravel默认值 1（本项目不会走到）
```

**本项目各Job的实际 tries 值**：

| Job类 | 实际 tries | 来源 |
|-------|-----------|------|
| ProcessScheduledContactReminders | 3 | Worker --tries=3 |
| SendVerificationEmailChannel | 3 | Worker --tries=3 |
| SetupAccount | 1 | 继承 QueuableService::$tries=1 |
| UpdateVCard | 1 | 继承 QueuableService::$tries=1 |
| SynchronizeAddressBooks | 1 | 自身属性 $tries=1 |
| PushVCard | 3 | Worker --tries=3 |
| UserInvited（Mailable） | 3 | Worker --tries=3 |

### 1.3 三者的交互时序（正常失败路径）

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

### 1.4 retry_after 静默重试（并发重复的根源）

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
- retry_after 触发时 `attempts 仍然会自增`（之前的分析"不增加attempts"是错误的）
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

### 1.5 人工重试时的参数重置

执行 `php artisan queue:retry <uuid>` 后：
- 从 `failed_jobs.payload` 反序列化出原始Job
- 重新 INSERT 到 `jobs` 表，**attempts 被重置为 0**
- tries 限制重新生效（相当于又有 tries 次机会）

这意味着：人工重试可以绕开 tries 限制，**理论上可以无限次重试**。

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

**取数条件只有 `scheduled_at <= 当前时间`，完全没有 `triggered_at IS NULL`。**

### 2.2 triggered_at 字段的真实用途（三处使用均非生产调度）

| 场景 | 代码位置 | 用途 | 是否参与生产取数 |
|------|---------|------|----------------|
| 前端Vault展示 | [VaultShowViewHelper.php#L48](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L48) | 只展示"待触发"的即将到来的提醒 | ❌ |
| 前端Reminder索引 | [VaultReminderIndexViewHelper.php#L39](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultReminderIndexViewHelper.php#L39) | 同上 | ❌ |
| 测试命令（非生产） | [TestReminders.php#L44](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Console/Commands/TestReminders.php#L44) | `where('triggered_at', null)` 手动触发用 | ❌ |
| 审计记录 | [ProcessScheduledContactReminders.php#L83-L85](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L83-L85) | `updateScheduledContactReminderTriggeredAt()` 设置触发时间 | ❌（只写不读） |

---

## 3. 循环提醒怎么重排？—— 更新同一条记录的 scheduled_at，且不重置 triggered_at

### 3.1 处理流程与执行顺序

```
取数（scheduled_at <= NOW()）
    ↓
foreach 每条记录
    ├─ try {
    │   ├─ 查询 ContactReminder, Contact
    │   ├─ triggerNotification()
    │   │   ├─ if (!channel->active) return
    │   │   ├─ Notification::route()->notify()      ← ① 发送
    │   │   ├─ increment number_times_triggered      ← ② 计数+1
    │   │   └─ RescheduleContactReminderForChannel   ← ③ 重排 scheduled_at
    │   └─ updateScheduledContactReminderTriggeredAt() ← ④ 设 triggered_at = NOW()
    │
    └─ } catch { ... }
```

**顺序：①发送 → ②计数 → ③Reschedule（改scheduled_at/删记录） → ④triggered_at**

### 3.2 Reschedule 只改 scheduled_at，不改 triggered_at

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

### 3.3 一次性提醒 vs 循环提醒的区别

| 类型 | Reschedule 行为 | triggered_at |
|------|----------------|-------------|
| TYPE_ONE_TIME | DELETE 整条记录 | ——（记录已删除） |
| TYPE_RECURRING_* | UPDATE scheduled_at → 未来，triggered_at 不变 | 随后被设为 NOW() |

### 3.4 测试代码印证

[RescheduleContactReminderForChannelTest.php#L81-L86](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/tests/Unit/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannelTest.php#L81-L86)：

```php
// 初始 scheduled_at=2018-01-01，triggered_at=null
(new RescheduleContactReminderForChannel)->execute([...]);

// 断言：scheduled_at → 2018-01-02，triggered_at 仍为 null
$this->assertDatabaseHas('contact_reminder_scheduled', [
    'scheduled_at' => '2018-01-02 00:00:00',
    'triggered_at' => null,   // Reschedule 不改变它
]);
```

### 3.5 真正的防重机制

防重复发送不靠 triggered_at，而是：
> **Reschedule 把 scheduled_at 推到未来，使下一次查询 `scheduled_at <= NOW()` 不再命中。**

脆弱点：如果 ①发送成功 但 ③Reschedule 没执行（进程被杀/异常），scheduled_at 仍是过去时间，下次调度（每分钟）会再次命中。

---

## 4. 三大边界深度分析：发送时序 / 并发取数 / 发送记录可信度

### 4.1 发送已发生但触发标记未写入？—— 不，是"通知发了但 scheduled 状态没更新"

首先澄清：不是"邮件发了但 triggered_at 没写"那么简单。实际有**五层操作**，每一层之间都可能中断，产生不同的边界状态。

#### 五层操作的完整时序（代码逐行拆解）

在 `ProcessScheduledContactReminders::handle()` 的单条记录处理中，按执行顺序有五层关键操作：

```
Layer 1: notify() 发送通知
  └─ Notification::route('mail', ...)->notify(new ReminderTriggered(...))
       └─ 内部会调用 toMail()，toMail() 里会写 UserNotificationSent

Layer 2: increment number_times_triggered
  └─ UPDATE contact_reminders SET number_times_triggered = number_times_triggered + 1

Layer 3: RescheduleContactReminderForChannel
  └─ syncWithoutDetaching 更新 scheduled_at（循环提醒）或 DELETE（一次性提醒）

Layer 4: updateTriggeredAt
  └─ UPDATE contact_reminder_scheduled SET triggered_at = NOW()

Layer 5: UserNotificationSent （注意：这层是在 Layer 1 的 toMail() 里写的，不是在最后）
```

**关键事实**：UserNotificationSent 的写入发生在 **toMail() 方法内**，也就是邮件真正发送**之前**。

#### sync 驱动 vs database 驱动的时序差异

**sync 驱动（本地开发）：**
```
notify() 调用
  → toMail() 执行 → UserNotificationSent::create() （写DB，此时邮件还没发）
  → 返回 MailMessage 对象
  → Laravel 调用邮件驱动真正发送
  → 发送成功 → notify() 返回
  → 发送失败 → 抛出异常 → 进入 catch 块
```
sync 下：notify() 返回 = 邮件已发出。toMail() 内的 UserNotificationSent 是在发送**前**写的。

**database 驱动（生产环境）：**
```
notify() 调用
  → toMail() 执行 → UserNotificationSent::create() （写DB）
  → 返回 MailMessage 对象
  → Laravel 检测到通知 use Queueable
  → 把通知 push 到队列（jobs 表）
  → notify() 返回（此时邮件还没发！）
```
database 下：notify() 返回 ≠ 邮件已发出。邮件可能还在队列里，甚至可能永远发不出去。

#### 各层中断后的边界状态

| 中断点 | 邮件状态 | UserNotificationSent | number_times_triggered | scheduled_at | triggered_at | 下次是否重复 |
|--------|---------|---------------------|----------------------|-------------|-------------|------------|
| Layer 1 之前 | 未发 | ❌ 无 | ❌ 未变 | 过去 | null | ✅ 是（正常） |
| Layer 1 中（toMail里） | 未发 | ⚠️ 可能有也可能无 | ❌ 未变 | 过去 | null | ✅ 是 |
| Layer 1 后（notify返回） | sync:已发<br>database:队列中 | ✅ 有 | ❌ 未变 | 过去 | null | ❌ 重复发送 |
| Layer 2 后 | 同上 | ✅ 有 | ✅ +1 | 过去 | null | ❌ 重复发送 |
| Layer 3 后（Reschedule） | 同上 | ✅ 有 | ✅ +1 | 未来 | null | ✅ 否 |
| Layer 4 后（triggered_at） | 同上 | ✅ 有 | ✅ +1 | 未来 | 有值 | ✅ 否 |

**结论**：
- database 驱动下，notify() 返回时邮件**可能还没发**，catch 块捕获不到发送异常
- 如果在 Layer 1 之后、Layer 3 之前中断，邮件可能已发（或在队列中），但 scheduled_at 还是过去，下次调度**会重复**
- 熔断器（channel->fails++）也不生效，因为 catch 块捕获不到发送异常

---

### 4.2 并发取数竞态—— 两个来源导致同批数据被重复处理

并发取数的问题有两个来源：

#### 来源1：retry_after 静默重试导致的并发

回顾第1节：`retry_after = 90`，如果 Job 执行超过90秒，Laravel 会认为 Worker 死了，自动把任务放回队列，另一个 Worker 会领取并执行。

对于 `ProcessScheduledContactReminders`：
- 每分钟调度一次
- 如果一批提醒很多，处理超过90秒
- retry_after 触发，新 Worker 并发执行**同一批数据**
- 取数 SQL：`WHERE scheduled_at <= NOW()`，**没有任何锁**
- 两个 Worker 同时取到同一份数据 → 同时发送 → 重复邮件

#### 来源2：每分钟调度叠加 retry_after 重试

更极端的情况：
- 第0分钟：Job A 启动，开始处理100条提醒
- 第1分钟：调度器又触发 Job B（因为每分钟一次），此时 Job A 还在运行
- 第1.5分钟：Job A 还没处理完，scheduled_at 还是过去，Job B 也取到了同样的数据
- 两个 Job 并发处理同批数据

注意：Laravel 的 withoutOverlapping 调度模式可以防止这个问题，但需要确认代码里有没有用。

#### syncWithoutDetaching 的并发行为

如果两个 Worker 同时处理同一条提醒，且都走到了 Reschedule：

```
Worker A: syncWithoutDetaching → scheduled_at = Day2
Worker B: syncWithoutDetaching → scheduled_at = Day2 （相同的结果，因为都是基于原 scheduled_at 计算）
```

两个都执行 syncWithoutDetaching，结果是一样的（因为计算出的 upcomingDate 相同），所以 scheduled_at 不会有问题。

但问题在于：**两个 Worker 都会执行 notify()**，导致重复发送邮件。

更坏的情况：Worker A 执行到 Reschedule 之后（scheduled_at 已推到未来），Worker B 才开始取数。

- 如果 Worker A 的事务还没提交，Worker B 可能还是读到旧的 scheduled_at（过去）→ 重复处理
- 如果 Worker A 的事务已提交，Worker B 读到新的 scheduled_at（未来）→ 不重复

**结论**：取数无锁 + 多并发源 → 重复发送是可能的，概率取决于提醒数量和处理速度。

---

### 4.3 发送记录可信度—— UserNotificationSent 不能证明邮件真的发出去了

这是最关键的认知修正。

#### UserNotificationSent 的写入时机

**ReminderTriggered::toMail()**：[ReminderTriggered.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Notifications/ReminderTriggered.php)

```php
public function toMail($notifiable)
{
    UserNotificationSent::create([
        ...
        'notification_channel_id' => $this->notificationChannel->id,
        'type' => 'reminder',
        'error' => null,
    ]);

    return (new MailMessage)
        ->subject(...)
        ->line(...);
}
```

**写在 toMail() 里，返回 MailMessage 之前**。也就是说：

> UserNotificationSent 记录存在 ≠ 邮件发送成功
> UserNotificationSent 记录存在 = 邮件**准备发送**的日志被写了

#### sync 驱动下的可信度

sync 驱动是同步的，发送失败会抛异常：

```
toMail() 执行
  → UserNotificationSent::create() 成功（error=null）
  → 返回 MailMessage
  → 邮件驱动真正发送
  → 发送失败 → 抛出异常
  → 进入 catch 块
  → 又写一条 UserNotificationSent（error=错误信息）
```

结果：**两条记录**
- 一条 error=null（假成功，toMail 里写的）
- 一条 error=xxx（真失败，catch 里写的）

#### database 驱动下的可信度

database 驱动下更混乱：

```
toMail() 执行
  → UserNotificationSent::create() 成功（error=null）
  → 返回 MailMessage
  → 通知被 push 到队列
  → notify() 返回（此时邮件还没发！）

... 稍后 Worker 从队列里取到邮件发送任务 ...
  → 发送成功 → 没有回调，不更新 UserNotificationSent
  → 发送失败 → 队列重试，还是不更新 UserNotificationSent
  → 最终失败 → 进入 failed_jobs，还是不更新 UserNotificationSent
```

结果：**只有一条 error=null 的记录，但邮件可能根本没发出去**。

#### catch 块里的 UserNotificationSent

catch 块里也会写一条 UserNotificationSent：

```php
catch (\Throwable $e) {
    UserNotificationSent::create([
        'error' => $e->getMessage(),
        ...
    ]);
}
```

但 catch 块捕获的是 **handle() 方法内的异常**。对于 database 驱动：
- notify() 只是把邮件推到队列，不会抛发送异常
- catch 块捕获不到真正的邮件发送失败
- 熔断器（channel->fails++）也只对 handle() 内的异常生效，不对队列里的发送失败生效

#### 结论：目前没有可靠的方式判断邮件是否发送成功

| 场景 | UserNotificationSent | 邮件实际状态 | 熔断器是否计数 |
|------|---------------------|-------------|--------------|
| sync 发送成功 | 1条 error=null | 成功 | 否 |
| sync 发送失败 | 2条（1条成功+1条error） | 失败 | 是（catch住） |
| database 队列中 | 1条 error=null | 待发送 | 否 |
| database 发送成功 | 1条 error=null | 成功 | 否 |
| database 发送失败 | 1条 error=null | 失败 | 否（catch不到） |

**发送记录完全不可靠**，既不能用来做幂等判断，也不能用来统计真实发送成功率。

---

## 5. triggered_at 过滤会不会影响循环提醒？—— 会！（加过滤必须同时重置）

### 5.1 初版建议的错误：只加过滤不重置 = 循环提醒只触发一次

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

### 5.2 为什么 TestReminders 有过滤却能正常工作？

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

### 5.3 正确的 triggered_at 幂等方案：过滤 + Reschedule 时重置

如果要用 triggered_at 作为幂等边界（防 retry_after 并发重复、防人工重试），必须**同时改两处**：

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

#### 修改2的补充：一次性提醒不用改（直接 DELETE）

```php
// TYPE_ONE_TIME 的分支已经是 DELETE，无需额外处理
DB::table('contact_reminder_scheduled')
    ->where('id', $this->data['contact_reminder_scheduled_id'])
    ->delete();
```

#### 加了两处修改后的完整生命周期（日循环提醒）

```
Day1:
  初始：scheduled=01-01 09:00, triggered=null
  取数：scheduled<=09:00 AND triggered IS NULL → ✅ 命中
  发送 → Reschedule: scheduled=01-02 09:00, triggered=null（被重置）
       → updateTriggeredAt: triggered=01-01 09:00（最后设置）
  最终：scheduled=01-02 09:00, triggered=01-01 09:00 ← 有值

Day2:
  状态：scheduled=01-02 09:00, triggered=01-01 09:00
  取数：scheduled<=09:00 AND triggered IS NULL
        → triggered 有值 = ❌ 不命中 ← 这里还是不命中啊！？

等一下，这里有问题。让我再梳理一遍顺序...
```

**⚠️ 顺序问题**：即使 Reschedule 中把 triggered_at 重置为 null，紧接着的 `updateTriggeredAt` 又把它设回 NOW() 了！

让我重排时序：
```
③ Reschedule 执行：
   UPDATE contact_reminder_scheduled
   SET scheduled_at = '2024-01-02 09:00',
       triggered_at = NULL            ← 先清空
   WHERE id = ?
   ↓
④ updateTriggeredAt 执行：
   UPDATE contact_reminder_scheduled
   SET triggered_at = '2024-01-01 09:00'   ← 又设置了
   WHERE id = ?
```

**结果**：triggered_at 最终还是有值。Day2 还是不命中。

所以还需要**第3处修改**：

#### 修改3：把 updateTriggeredAt 移到 Reschedule 之前执行

即把顺序从 `发送 → 计数 → Reschedule → updateTriggeredAt` 改为 `发送 → 计数 → updateTriggeredAt → Reschedule`

```php
// 在 triggerNotification() 内，或者在 try 块中：
private function triggerNotification(...)
{
    ...
    Notification::route(...)->notify(...);   // ① 发送
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

## 6. 人工重试会不会重复发送？—— 分情况（含并发场景）

### 6.1 ProcessScheduledContactReminders 的特性

这个Job无构造参数、每次 handle() 都重新查询DB。人工重试 = 重新跑一遍完整的取数+处理逻辑。

### 6.2 各场景判定矩阵

| 场景 | Reschedule 状态 | scheduled_at 状态 | channel 状态 | 人工重试结果 | 最多重复 |
|------|----------------|------------------|-------------|-------------|---------|
| A：发送前异常 | 未执行 | 过去 | active | ✅ 不重复（没发出去） | 0 |
| B：正常完成 | 成功 | 未来/已删 | — | ✅ 不重复（取数命中不了） | 0 |
| C：发送成、Reschedule败，catch住 | 失败 | 过去 | active | ❌ **重复**（每分钟调度+人工重试都会命中），直到熔断 | ≤10次 |
| D：发送成、Reschedule成，进程被杀 | 成功 | 未来 | — | ✅ 不重复 | 0 |
| E：发送成、Reschedule未执行，进程被杀 | 未执行 | 过去 | active | ❌ **重复** | ≤3(tries)×N |
| F：retry_after 并发 + Reschedule 部分成功 | 不同Worker执行到不同阶段 | — | — | ❌ **不确定**（取决于哪条Worker先改DB） | 高并发下难预估 |

### 6.3 加上 triggered_at 幂等方案后的人工重试

假设已实施了第5节的三处修改：

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

## 7. 其他邮件任务的重试与恢复

### 7.1 验证邮件（SendVerificationEmailChannel）

- 无 `$tries` → 默认 tries=3
- **无任何幂等性检查**：不检查 `verified_at`，不检查发送记录
- retry_after 静默重试：可重复发送（1 < attempts < 3 范围内）
- 人工重试 → 反序列化 $channel → 肯定重复发送
- 缓解：添加 `verified_at` 检查

### 7.2 邀请邮件（UserInvited）

- Mailable 自带 ShouldQueue
- 无 tries、无幂等、人工重试必重复

### 7.3 测试邮件（SendTestEmail）

- 同步执行，不走队列，无重试问题

---

## 8. 风险矩阵与关键风险点

### 8.1 重复发送风险矩阵

| 任务类型 | tries | 幂等防护 | retry_after 并发风险 | 人工重试是否重复 |
|---------|-------|----------|-------------------|-----------------|
| 提醒邮件 | 3 | Reschedule推scheduled_at（脆弱） | ⚠️ 高（无悲观锁） | 视 Reschedule 成功否 |
| 验证邮件 | 3 | 无 | ⚠️ 中（单任务执行快） | ✅ 是 |
| 邀请邮件 | 3 | 无 | ⚠️ 中（单任务执行快） | ✅ 是 |
| DAV同步类 | 1 | etag/幂等接口 | ✅ 低 | ✅ 否 |

### 8.2 最高风险：ProcessScheduledContactReminders 的 retry_after 并发

- 每分钟调度一次，大量提醒时处理可能超过90秒
- retry_after 触发后 attempts 自增，新 Worker 并发执行同一批数据
- **没有悲观锁（FOR UPDATE SKIP LOCKED）、没有 triggered_at 条件、没有唯一键校验**
- 结果：同一批提醒可能被2~3个Worker并发处理 → 群发重复邮件

**缓解（低成本）**：
1. 给 ProcessScheduledContactReminders 加 `$timeout = 60`（小于 retry_after=90，超时直接杀进程而非并发）
2. 取数SQL改用悲观锁（database驱动支持）：
   ```php
   $scheduledContactReminders = DB::table('contact_reminder_scheduled')
       ->where('scheduled_at', '<=', $currentDate)
       ->lockForUpdate()         // ← 加悲观锁
       ->get();
   ```
   （但注意：laravel database 队列的 Worker 取 jobs 时也有自己的锁机制，这个是业务表的锁）

---

## 9. 代码走向流程图（完整纠正版）

### 9.1 循环提醒的一次完整生命周期

```
[初始状态]
  contact_reminder_scheduled:
    id=123, channel_id=7, reminder_id=42
    scheduled_at = 2024-01-01 09:00:00
    triggered_at = null

Schedule 每分钟触发 ProcessScheduledContactReminders
    ↓
handle()
  DB查询: WHERE scheduled_at <= NOW()  AND triggered_at IS NULL?
          ↑ 注意：生产代码只有第一个条件，第二个条件不存在！
    ↓
  命中 id=123
    ↓
  try {
    findOrFail(ContactReminder=42, UserNotificationChannel=7)
      ↓
    triggerNotification()
      ├─ if (!channel->active) return
      ├─ Notification::route('mail', ...)->notify(ReminderTriggered)
      │     └─ toMail(): UserNotificationSent::create() → 写日志 → 返回 MailMessage
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
    Log::error
    UserNotificationSent::create(error => ...)
    channel->fails++
    fails >= 10 → channel->active = false + 删除其所有调度
    channel->save()
  }

[最终状态（循环提醒）]
  scheduled_at = 2024-01-02 09:00:00  ← 推到未来
  triggered_at = 2024-01-01 09:00:xx  ← 标记这次触发时间
                        ↑ 注意：下次查询 (WHERE scheduled_at<=2024-01-02 09:00)
                                时，triggered_at 不参与过滤！
                                只看 scheduled_at 已 ≤ NOW()，所以会命中。
```

### 9.2 retry_after / attempts / tries 交互图

```
                         ┌──────────────────────┐
                         │  dispatch() → jobs表  │
                         │  attempts=0           │
                         └──────────┬───────────┘
                                    ↓
                         ┌──────────────────────┐
                         │ Worker领取            │
                         │ attempts++            │
                         │ reserved_at = NOW()   │
                         └──────────┬───────────┘
                                    ↓
               ┌─── attempts < tries ? ───┐
               │ YES                       NO │
               ↓                            ↓
    ┌──────────────────────┐      ┌──────────────────────┐
    │  handle() 执行中...  │      │  → failed_jobs       │
    │                      │      │  → DELETE jobs       │
    │  ┌─ 超过 retry_after? │      └──────────────────────┘
    │  │ YES                │
    │  │                    │
    │  ↓                    │
    │ reserved_at = null    │
    │ （释放给其他Worker）   │
    │  ──→ 新Worker领取     │
    │      attempts++      │
    │      → 回到判断框     │
    └─── 正常完成/异常 ────┘
         异常时：
           释放回队列(reserved_at=null)
           → Worker再次领取
           → attempts++
           → 回到判断框
```

---

## 10. 优化建议（纠正版）

### 10.1 短期优化（低风险）

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
$scheduledContactReminders = DB::table('contact_reminder_scheduled')
    ->where('scheduled_at', '<=', $currentDate)
    ->lockForUpdate()    // 事务内加悲观锁
    ->get();
```
需要包在 DB::transaction() 中。并发Worker会排队而不是重复取数。

### 10.2 中期优化（需同时修改多处）

#### ④ 三处联动修改，引入 triggered_at 作为幂等边界

**修改1**：取数加 `triggered_at IS NULL`

**修改2**：RescheduleContactReminderForChannel::schedule() 在 syncWithoutDetaching 时同步设置 `triggered_at => null`

**修改3**：在 ProcessScheduledContactReminders 中，把 `updateScheduledContactReminderTriggeredAt()` 的调用从 try 块末尾移到 `Reschedule` 调用之前（即 triggerNotification 方法内）

三处必须联动，缺一不可（详见第5.3节）。

#### ⑤ contact_reminder_scheduled 表加唯一索引

```sql
ALTER TABLE contact_reminder_scheduled
ADD UNIQUE KEY uk_channel_reminder
  (user_notification_channel_id, contact_reminder_id);
```
防止 syncWithoutDetaching 在并发下产生重复 pivot 记录（目前代码逻辑上不会重复，但表结构无约束）。

### 10.3 长期优化（高风险，架构调整）

- 发件箱模式（Transactional Outbox）
- 邮件服务商 Webhook 追踪投递状态
- Redis 分布式锁防同一份提醒并发处理

---

## 11. 相关文件索引

| 类型 | 文件路径 | 关键内容 |
|------|---------|----------|
| 队列配置 | [config/queue.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/config/queue.php) | retry_after=90 |
| Worker脚本 | [scripts/docker/queue.sh](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/scripts/docker/queue.sh) | --tries=3 |
| jobs表迁移 | [2022_01_22_183321_create_jobs_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/database/migrations/2022_01_22_183321_create_jobs_table.php) | attempts, reserved_at 字段 |
| 提醒调度Job | [ProcessScheduledContactReminders.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php) | 取数无triggered_at条件、执行顺序 |
| 重排服务 | [RescheduleContactReminderForChannel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannel.php) | syncWithoutDetaching 只改scheduled_at |
| 调度表迁移 | [2022_02_18_215852_create_reminders_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/database/migrations/2022_02_18_215852_create_reminders_table.php) | contact_reminder_scheduled 字段 |
| 测试命令 | [TestReminders.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Console/Commands/TestReminders.php) | 有triggered_at过滤，但不调用updateTriggeredAt |
| 前端展示过滤 | [VaultShowViewHelper.php#L48](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Vault/ManageVault/Web/ViewHelpers/VaultShowViewHelper.php#L48) | wherePivot('triggered_at', null) |
| QueuableService基类 | [QueuableService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Services/QueuableService.php) | tries=1 |
| 验证邮件Job | [SendVerificationEmailChannel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Settings/ManageNotificationChannels/Jobs/SendVerificationEmailChannel.php) | 无幂等检查 |
| 熔断器配置 | [config/monica.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/config/monica.php) | max_notification_failures=10 |
| 重排测试 | [RescheduleContactReminderForChannelTest.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/tests/Unit/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannelTest.php) | 断言triggered_at在Reschedule后保持null |
