# 异步任务与邮件失败恢复代码分析

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

**任务表**：[jobs](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/database/migrations/2022_01_22_183321_create_jobs_table.php)

| 字段 | 类型 | 用途 |
|------|------|------|
| id | bigIncrements | 主键 |
| queue | string | 队列名称（high/default/low） |
| payload | longText | 任务序列化数据（JSON格式） |
| attempts | unsignedTinyInteger | 已尝试次数 |
| reserved_at | unsignedInteger nullable | 被worker领取的时间戳 |
| available_at | unsignedInteger | 任务可执行时间 |
| created_at | unsignedInteger | 创建时间 |

**失败任务表**：[failed_jobs](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/database/migrations/2019_08_19_000000_create_failed_jobs_table.php)

| 字段 | 类型 | 用途 |
|------|------|------|
| id | id | 主键 |
| uuid | string unique | 唯一标识（用于重试命令） |
| connection | text | 队列连接名 |
| queue | text | 队列名称 |
| payload | longText | 完整任务载荷 |
| exception | longText | 异常堆栈信息 |
| failed_at | timestamp | 失败时间 |

---

## 2. 任务载荷（Payload）结构

### 2.1 序列化机制

所有队列Job类使用以下核心Trait：

- `SerializesModels` - 模型序列化（只存ID，执行时重新查询）
- `Queueable` - 队列配置（连接、队列、延迟等）
- `InteractsWithQueue` - 与队列交互（删除、重试、失败等）
- `Dispatchable` - 任务分发

**载荷JSON结构示例**：

```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  "displayName": "App\\Domains\\Settings\\ManageNotificationChannels\\Jobs\\SendVerificationEmailChannel",
  "job": "Illuminate\\Queue\\CallQueuedHandler@call",
  "maxTries": null,
  "maxExceptions": null,
  "failOnTimeout": false,
  "backoff": null,
  "timeout": null,
  "retryUntil": null,
  "data": {
    "commandName": "App\\Domains\\Settings\\ManageNotificationChannels\\Jobs\\SendVerificationEmailChannel",
    "command": "O:73:\"App\\Domains\\Settings\\ManageNotificationChannels\\Jobs\\SendVerificationEmailChannel\":2:{...}"
  }
}
```

### 2.2 典型邮件任务载荷分析

#### SendVerificationEmailChannel 任务

**类定义**：[SendVerificationEmailChannel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Settings/ManageNotificationChannels/Jobs/SendVerificationEmailChannel.php#L18-L53)

```php
class SendVerificationEmailChannel implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected UserNotificationChannel $channel;

    public function __construct(UserNotificationChannel $channel) {
        $this->channel = $channel;
    }

    public function handle() {
        if ($this->channel->type !== UserNotificationChannel::TYPE_EMAIL) {
            return;
        }
        Mail::to($this->channel->content)
            ->send(new UserNotificationChannelEmailCreated($this->channel));
    }
}
```

**载荷特点**：
- `$channel` 是 `UserNotificationChannel` 模型，使用 `SerializesModels` 序列化后只存ID
- 无 `$tries` 属性，使用Worker默认值 **3次重试**
- 无 `$backoff`，重试无延迟
- 无幂等性检查

---

## 3. 重试机制详解

### 3.1 重试层级

系统存在三个层级的重试配置，优先级从高到低：

#### 层级1：Job类属性（最高优先级）

**QueuableService 基类**：[QueuableService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Services/QueuableService.php#L14-L56)

```php
abstract class QueuableService extends BaseService implements ShouldQueue
{
    public int $tries = 1;  // 只尝试1次，不重试
}
```

**继承关系**：
- `SetupAccount` 继承 `QueuableService` → tries=1
- `UpdateVCard` 继承 `QueuableService` → tries=1

**SynchronizeAddressBooks**：[SynchronizeAddressBooks.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/DavClient/Jobs/SynchronizeAddressBooks.php#L15-L79)

```php
class SynchronizeAddressBooks implements ShouldQueue
{
    public $tries = 1;  // 显式设置只尝试1次
}
```

#### 层级2：Worker启动参数（默认值）

```bash
--tries=3  # 未显式设置$tries的Job使用此值
```

受影响的Job类：
- `SendVerificationEmailChannel` → 重试3次
- `ProcessScheduledContactReminders` → 重试3次
- `PushVCard` → 重试3次
- `GetVCard`/`DeleteVCard` 等 → 重试3次

#### 层级3：队列配置 retry_after

```php
// config/queue.php
'database' => [
    'retry_after' => 90,  // 任务执行超过90秒被视为失败，自动放回队列
],
```

**⚠️ 重要边界**：
- 这是**静默重试**，不增加 `attempts` 计数
- 即使 `tries=1`，超过 `retry_after` 仍会被重新执行
- 这是**重复发送的主要风险点**

### 3.2 重试间隔（Backoff）

所有Job类均未设置 `$backoff` 属性，使用Laravel默认值：
- 第1次重试：无延迟
- 第2次重试：无延迟
- 第3次重试：无延迟

### 3.3 代码内自定义重试

**PushVCard 特殊重试逻辑**：[PushVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/DavClient/Jobs/PushVCard.php#L80-L105)

```php
private function pushDistant(int $depth = 1): string
{
    try {
        $response = $this->subscription->getClient()
            ->request('PUT', $this->uri, $this->card, $this->headers());
        return $response->header('Etag');
    } catch (RequestException $e) {
        if ($depth > 0 && $e->response->status() === 412) {
            // 412 Precondition Failed 时重试1次（使用MODE_MATCH_NONE）
            $this->mode = self::MODE_MATCH_NONE;
            return $this->pushDistant(--$depth);
        } else {
            $this->fail($e);
            throw $e;
        }
    }
}
```

**特点**：
- 这是**业务逻辑内的重试**，不经过队列系统
- 不增加 `jobs.attempts` 计数
- 只对 412 状态码重试，depth=1 意味着最多重试1次
- 与队列层面的重试是叠加关系（总重试次数 = 队列重试 × 业务重试）

---

## 4. 失败恢复流程

### 4.1 失败检测与记录

**失败触发条件**：
1. Job执行抛出未捕获异常 → 立即标记为失败
2. 执行时间超过 `retry_after` (90秒) → Worker自动释放回队列
3. 超过 `maxTries` → 标记为永久失败

**失败流程**：

```
Job执行异常
    ↓
attempts++
    ↓
attempts < maxTries ?
    ├─ 是 → 重新放回队列（available_at = now() + backoff）
    └─ 否 → 调用 failed() 方法 → 写入 failed_jobs 表
```

### 4.2 失败清理机制

**调度配置**：[console.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/routes/console.php#L20-L29)

```php
Schedule::command('queue:prune-batches --hours=48 --unfinished=72 --cancelled=72', 'daily');
Schedule::command('queue:prune-failed --hours=48', 'daily');
```

**清理规则**：
- 每天执行一次
- 48小时前的失败任务自动删除
- 超过72小时未完成/已取消的批处理自动删除
- **失败任务最多保留48小时用于人工重试**

### 4.3 人工重试

**重试命令**（Laravel内置，代码中无自定义封装）：

```bash
# 重试所有失败任务
php artisan queue:retry all

# 重试指定UUID的失败任务
php artisan queue:retry 550e8400-e29b-41d4-a716-446655440000

# 查看失败任务列表
php artisan queue:failed
```

**⚠️ 重试风险**：
- 从 `failed_jobs.payload` 反序列化后重新执行
- 无幂等性检查的任务可能重复执行
- 原任务上下文（如模型状态）可能已变化

---

## 5. 重复发送边界分析

### 5.1 提醒邮件（ProcessScheduledContactReminders）

**类定义**：[ProcessScheduledContactReminders.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L21-L126)

**调度频率**：每1分钟执行1次
```php
Schedule::job(ProcessScheduledContactReminders::class, 'minutes', 1);
```

**执行流程**：

```
查询 contact_reminder_scheduled
WHERE scheduled_at <= NOW() AND triggered_at IS NULL
    ↓
遍历每条记录
    ├─ 检查 channel->active
    ├─ 发送通知（邮件/Telegram）
    ├─ 更新 triggered_at = NOW()  ← 幂等边界
    ├─ 递增 number_times_triggered
    └─ 创建下一次调度记录
```

**重复发送防护**：

1. **triggered_at 机制**（主防护）
   ```php
   private function updateScheduledContactReminderTriggeredAt($scheduledReminder): void
   {
       DB::table('contact_reminder_scheduled')
           ->where('id', $scheduledReminder->id)
           ->update(['triggered_at' => Carbon::now()]);
   }
   ```
   - SQL条件：`triggered_at IS NULL`
   - 即使任务重试，已处理的记录不会被重复查询

2. **异常捕获**（次防护）
   ```php
   try {
       // 发送逻辑
       $this->updateScheduledContactReminderTriggeredAt($scheduledReminder);
   } catch (\Exception $e) {
       Log::error(...);
       UserNotificationSent::create([..., 'error' => $e->getMessage()]);
       $channel->fails += 1;
       if ($channel->fails >= 10) {
           $channel->active = false;  // 熔断器
       }
   }
   ```
   - 单条失败不影响其他提醒
   - 失败记录到 `user_notification_sents` 表
   - 连续失败10次自动禁用channel（熔断器）

3. **熔断器配置**：[monica.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/config/monica.php#L142)
   ```php
   'max_notification_failures' => 10,
   ```

**⚠️ 仍存在的重复风险**：

```
时序风险：
1. Worker1: 查询 scheduled (id=1, triggered_at=null)
2. Worker1: 发送邮件 → 成功
3. Worker1: 进程卡住/网络延迟 > 90秒
4. Worker2: 检测到 retry_after 超时 → 重新获取任务
5. Worker2: 查询 scheduled (id=1, triggered_at=null) ← 此时triggered_at还未更新
6. Worker2: 发送邮件 → 重复发送
7. Worker1: 恢复 → 更新 triggered_at = NOW()
```

### 5.2 验证邮件（SendVerificationEmailChannel）

**幂等性分析**：

```
❌ 无 triggered_at 机制
❌ 无唯一索引防重复
❌ 无发送状态检查
✅ 有 tries=3 限制（来自Worker默认值）
```

**重复发送场景**：

| 场景 | 是否重复 | 原因 |
|------|----------|------|
| 正常执行1次 | 否 | - |
| 邮件服务超时 < 90秒，抛出异常 | 最多3次 | 队列重试 |
| 邮件服务超时 > 90秒，未抛出异常 | 可能无限次 | retry_after 静默重试，不增加attempts |
| 任务执行中Worker进程被杀 | 最多3次 | 重启后重新执行 |
| 人工执行 queue:retry | 可能多次 | 无发送记录检查 |

**验证邮件载荷**：[UserNotificationChannelEmailCreated.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Mail/UserNotificationChannelEmailCreated.php#L10-L39)

```php
class UserNotificationChannelEmailCreated extends Mailable
{
    use Queueable, SerializesModels;

    public function __construct(public UserNotificationChannel $channel) {}

    public function build()
    {
        return $this->subject(trans('Please validate your email address'))
            ->markdown('emails.notifications.validate-email', [
                'url' => route('settings.notifications.verification.store', [
                    'notification' => $this->channel->id,
                    'uuid' => $this->channel->verification_token,
                ]),
            ]);
    }
}
```

**重复发送的后果**：
- 用户收到多封验证邮件
- 所有邮件中的验证链接相同（同一个 `verification_token`）
- 点击任意一封都可以完成验证
- 属于**可接受的重复**（用户体验影响较小）

### 5.3 邀请邮件（UserInvited）

**类定义**：[UserInvited.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Mail/UserInvited.php#L11-L42)

```php
class UserInvited extends Mailable implements ShouldQueue
{
    use Queueable, SerializesModels;

    public function __construct(protected User $invitedUser, protected User $user) {}
}
```

**特点**：
- `Mailable` 本身实现 `ShouldQueue`，邮件发送自动进入队列
- 无 `$tries` 设置 → 使用默认3次重试
- 无幂等性检查
- 无发送记录

### 5.4 测试邮件（SendTestEmail）

**类定义**：[SendTestEmail.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Settings/ManageNotificationChannels/Services/SendTestEmail.php#L14-L82)

```php
class SendTestEmail extends BaseService implements ServiceInterface
{
    public function execute(array $data): UserNotificationChannel
    {
        $this->validate();
        $this->send();      // 同步发送，不进队列
        $this->log();       // 记录发送日志
        return $this->userNotificationChannel;
    }

    private function send(): void
    {
        Mail::to($this->userNotificationChannel->content)->send(
            new TestEmailSent($this->userNotificationChannel)
        );
    }

    private function log(): void
    {
        UserNotificationSent::create([
            'user_notification_channel_id' => $this->userNotificationChannel->id,
            'sent_at' => Carbon::now(),
            'subject_line' => trans('Test email for Monica'),
        ]);
    }
}
```

**特点**：
- **同步执行**，不进入队列
- 无重试机制（失败直接抛出异常）
- 有发送日志记录

---

## 6. 通知发送日志（UserNotificationSent）

**记录时机**：

1. **提醒通知**：在 `toMail`/`toTelegram` 方法中记录
   [ReminderTriggered.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Notifications/ReminderTriggered.php#L52-L58)

   ```php
   public function toMail($notifiable)
   {
       UserNotificationSent::create([
           'user_notification_channel_id' => $this->channel->id,
           'sent_at' => Carbon::now(),
           'subject_line' => $this->content,
       ]);
       // 返回 MailMessage
   }
   ```

2. **测试邮件**：发送成功后记录
   [SendTestEmail.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Settings/ManageNotificationChannels/Services/SendTestEmail.php#L74-L81)

3. **提醒发送失败**：catch块中记录（带error）
   [ProcessScheduledContactReminders.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L61-L66)

   ```php
   UserNotificationSent::create([
       'user_notification_channel_id' => $userNotificationChannel->id,
       'sent_at' => Carbon::now(),
       'subject_line' => '',
       'error' => $e->getMessage(),
   ]);
   ```

**⚠️ 日志边界问题**：
- 日志记录在 `toMail` 方法开头，**发送动作之前**
- 如果日志写入成功但邮件发送失败，会出现"已记录但未实际发送"
- 无法通过此表判断邮件是否真实送达（只能判断是否尝试发送）

---

## 7. 关键风险点总结

### 7.1 重复发送风险矩阵

| 任务类型 | 最大重试次数 | 幂等防护 | 重复风险 | 风险等级 |
|---------|-------------|----------|---------|----------|
| 提醒邮件 | 3（队列）+ N（retry_after） | triggered_at + 异常捕获 | 低（时序漏洞） | ⚠️ 中 |
| 验证邮件 | 3（队列）+ N（retry_after） | 无 | 中 | ⚠️⚠️ 高 |
| 邀请邮件 | 3（队列）+ N（retry_after） | 无 | 中 | ⚠️⚠️ 高 |
| 测试邮件 | 0（同步） | 无 | 无 | ✅ 低 |
| DAV同步任务 | 1 | etag机制 | 低 | ✅ 低 |
| SetupAccount | 1 | 无（数据初始化） | 低 | ✅ 低 |

### 7.2 retry_after 静默重试风险

**风险描述**：
- `retry_after = 90` 秒是隐形的重试机制
- 不增加 `attempts` 计数，不受 `tries` 限制
- 如果任务实际在运行但超过90秒，会被重复执行

**理论最大执行次数**：
```
实际重试次数 = min(tries, ∞) × retry_after触发次数
```

**缓解建议**：
1. 为邮件发送类添加 `$timeout`，小于 `retry_after`：
   ```php
   public int $timeout = 60;  // 小于 90
   ```
2. 为关键邮件添加幂等性检查（如发送前检查 `verified_at`）

### 7.3 失败任务人工重试风险

**风险描述**：
- `php artisan queue:retry` 直接从 `failed_jobs` 恢复执行
- 无业务层面的状态检查
- 对于验证邮件，用户可能已验证但仍会重发

**缓解建议**：
1. 批量重试前先筛选任务类型
2. 为可重复发送的任务添加业务幂等性检查

---

## 8. 代码走向流程图

### 8.1 邮件发送完整链路（以提醒邮件为例）

```
Schedule 每分钟触发
    ↓
ProcessScheduledContactReminders::handle()
    ├─ 查询 scheduled_at <= NOW() AND triggered_at IS NULL
    ├─ 循环处理每条记录
    │   ├─ try {
    │   │   ├─ 检查 channel->active
    │   │   ├─ Notification::route(...)->notify(new ReminderTriggered(...))
    │   │   │   └─ ReminderTriggered::toMail()
    │   │   │       ├─ UserNotificationSent::create()  ← 记录日志
    │   │   │       └─ 返回 MailMessage  ← 实际发送
    │   │   ├─ updateScheduledContactReminderTriggeredAt()  ← 标记已处理
    │   │   └─ RescheduleContactReminderForChannel  ← 创建下次调度
    │   └─ } catch (\Exception $e) {
    │       ├─ Log::error()
    │       ├─ UserNotificationSent::create(..., error => ...)
    │       ├─ channel->fails++
    │       └─ fails >= 10 ? channel->active = false : 继续
    └─ 结束循环
```

### 8.2 队列任务生命周期

```
dispatch() → 写入 jobs 表 (attempts=0, reserved_at=null)
    ↓
Worker pick up → reserved_at = NOW(), attempts++
    ↓
handle() 执行
    ├─ 成功 → delete from jobs
    └─ 失败 → 检查 attempts < maxTries
        ├─ 是 → available_at = NOW() + backoff, reserved_at = null
        │          ↓
        │        Worker再次pick up（重试）
        └─ 否 → 调用 failed() → 写入 failed_jobs → delete from jobs
                ↓
              人工 queue:retry → 重新写入 jobs 表
```

---

## 9. 优化建议

### 9.1 短期优化（低风险）

1. **为邮件Job添加超时限制**（小于 retry_after）
   ```php
   // SendVerificationEmailChannel 等添加
   public int $timeout = 60;
   ```

2. **验证邮件添加幂等性检查**
   ```php
   public function handle()
   {
       if ($this->channel->verified_at !== null) {
           return;  // 已验证，不重复发送
       }
       // ... 发送逻辑
   }
   ```

3. **调整 UserNotificationSent 记录时机**
   - 移到邮件发送成功后（需重写通知发送逻辑）

### 9.2 中期优化（中风险）

1. **为邮件发送添加唯一键防重**
   - 使用 `(channel_id, purpose, scheduled_at)` 唯一索引
   - 发送前 INSERT IGNORE，成功才实际发送

2. **添加邮件发送状态机**
   - PENDING → SENDING → SENT/FAILED
   - 用数据库事务保证状态变更原子性

### 9.3 长期优化（高风险）

1. **引入分布式锁**（如Redis）防止时序漏洞
2. **使用事务消息** 保证"业务操作+消息发送"原子性
3. **接入邮件服务商Webhook** 准确追踪投递状态

---

## 10. 相关文件索引

| 类型 | 文件路径 | 关键内容 |
|------|---------|----------|
| 队列配置 | [config/queue.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/config/queue.php) | retry_after, failed配置 |
| Worker脚本 | [scripts/docker/queue.sh](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/scripts/docker/queue.sh) | 启动参数 |
| 调度配置 | [routes/console.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/routes/console.php) | 任务频率 |
| 应用配置 | [config/monica.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/config/monica.php) | max_notification_failures |
| 任务基类 | [app/Services/QueuableService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Services/QueuableService.php) | tries=1 |
| 验证邮件Job | [SendVerificationEmailChannel.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Settings/ManageNotificationChannels/Jobs/SendVerificationEmailChannel.php) | 无重试限制 |
| 提醒处理Job | [ProcessScheduledContactReminders.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php) | triggered_at机制 |
| 提醒通知 | [ReminderTriggered.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Notifications/ReminderTriggered.php) | 发送日志 |
| 邀请邮件 | [app/Mail/UserInvited.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Mail/UserInvited.php) | ShouldQueue |
| DAV同步Job | [PushVCard.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/app/Domains/Contact/DavClient/Jobs/PushVCard.php) | 业务内重试 |
| jobs表迁移 | [2022_01_22_183321_create_jobs_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/database/migrations/2022_01_22_183321_create_jobs_table.php) | 表结构 |
| failed_jobs表迁移 | [2019_08_19_000000_create_failed_jobs_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/53-monica/database/migrations/2019_08_19_000000_create_failed_jobs_table.php) | 表结构 |
