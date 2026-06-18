# Monica 邮件与通知系统代码链路分析

## 一、系统整体架构

Monica 的通知系统由以下核心模块组成，形成从模板定义到最终发送的完整链路：

```
模板定义层 → 变量替换层 → 收件人匹配层 → 调度/队列层 → 发送执行层 → 日志记录层
```

---

## 二、模板定义层

### 2.1 邮件模板（Mailable）

位于 [app/Mail](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Mail) 目录下，共有 3 个邮件模板类：

| 模板类 | 用途 |
|--------|------|
| [UserNotificationChannelEmailCreated](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Mail/UserNotificationChannelEmailCreated.php#L10-L39) | 邮箱渠道验证邮件 |
| [TestEmailSent](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Mail/TestEmailSent.php#L10-L34) | 测试邮件 |
| [UserInvited](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Mail/UserInvited.php#L11-L42) | 用户邀请邮件 |

所有 Mailable 类均继承自 `Illuminate\Mail\Mailable`，并使用 `Queueable` 和 `SerializesModels` Trait 以支持队列序列化。

**模板定义示例**（以验证邮件为例）：

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

### 2.2 通知模板（Notification）

系统核心的提醒通知通过 [ReminderTriggered](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Notifications/ReminderTriggered.php#L13-L100) 类实现，它继承自 Laravel 的 `Notification` 基类，支持多渠道分发。

**支持的渠道类型**：
- `email`：通过 `toMail()` 方法构建 `MailMessage`
- `telegram`：通过 `toTelegram()` 方法构建 `TelegramMessage`

```php
class ReminderTriggered extends Notification
{
    use Queueable;

    public function __construct(
        private UserNotificationChannel $channel,
        private string $content,
        private string $contactName
    ) {}

    // 根据渠道类型决定分发方式
    public function via($notifiable)
    {
        switch ($this->channel->type) {
            case UserNotificationChannel::TYPE_EMAIL:
                return ['mail'];
            case UserNotificationChannel::TYPE_TELEGRAM:
                return ['telegram'];
        }
        return [];
    }
}
```

### 2.3 Blade 视图模板

邮件内容的 Blade 视图位于 `resources/views/emails/` 目录下：

| 视图文件 | 对应 Mailable |
|----------|---------------|
| [validate-email.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/resources/views/emails/notifications/validate-email.blade.php#L1-L14) | UserNotificationChannelEmailCreated |
| [test-notification.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/resources/views/emails/notifications/test-notification.blade.php#L1-L10) | TestEmailSent |
| `emails.user.invitation` | UserInvited |

视图中使用 `@lang()` 进行国际化翻译，使用 `{{ $variable }}` 输出变量，通过 `@component('mail::message')` 套用邮件布局。

---

## 三、模板变量替换机制

### 3.1 Laravel 内置替换机制

#### 3.1.1 Mailable 的 `build()` 方法变量注入

通过 `markdown()` 或 `view()` 的第二个参数，或 `with()` 方法将变量传递给 Blade 视图：

```php
// 方式1：markdown/view 的第二参数
->markdown('emails.notifications.validate-email', [
    'url' => $url,
]);

// 方式2：with() 链式调用
->with('userName', $this->user->name)
->with('url', $invitationRoute);
```

#### 3.1.2 Notification 的消息构建变量

在 `toMail()` 方法中，通过 Laravel 的 `MailMessage` fluent API 构建邮件内容，变量直接嵌入 `trans()` 函数：

```php
public function toMail($notifiable)
{
    return (new MailMessage)
        ->subject(trans('Reminder for :name', ['name' => $this->contactName]))
        ->line(trans('You wanted to be reminded of the following:'))
        ->line($this->content)
        ->line($this->contactName);
}
```

### 3.2 国际化变量替换（`trans()` 函数）

系统大量使用 `trans()` 函数进行带变量的翻译，格式为：

```php
trans('Reminder for :name', ['name' => $this->contactName])
trans('🔔 Reminder: :label for :contactName', [
    'label' => $this->content,
    'contactName' => $this->contactName,
])
```

其中 `:name`、`:label`、`:contactName` 是占位符，第二个参数数组提供替换值。

### 3.3 自定义变量替换：NameHelper

[NameHelper::formatContactName()](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Helpers/NameHelper.php#L19-L68) 实现了联系人姓名的自定义变量模板替换。

用户可以在 `name_order` 配置中使用如 `%first_name% %last_name%` 的变量格式，该方法逐个字符扫描，遇到 `%...%` 变量时从 Contact 模型中取出对应字段值进行替换。

```php
public static function formatContactName(User $user, Contact $contact): string
{
    $allCharacters = str_split($user->name_order);
    // ... 扫描 %variable% 并替换为 $contact->$variable
}
```

---

## 四、收件人匹配层

### 4.1 核心数据模型

#### 4.1.1 UserNotificationChannel 模型

[UserNotificationChannel](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Models/UserNotificationChannel.php#L11-L83) 是收件人匹配的核心模型，存储用户的通知渠道信息：

| 字段 | 说明 |
|------|------|
| `user_id` | 关联用户 UUID |
| `type` | 渠道类型：`email` 或 `telegram` |
| `label` | 用户自定义标签 |
| `content` | **收件人地址**：邮箱地址或 Telegram chat_id |
| `active` | 是否激活 |
| `verified_at` | 邮箱验证时间 |
| `preferred_time` | 偏好发送时间 |
| `verification_token` | 邮箱验证 Token |
| `fails` | 连续失败次数 |

**关键关系**：
- `user()`：BelongsTo → 用户
- `userNotificationSent()`：HasMany → 发送历史记录
- `contactReminders()`：BelongsToMany → 调度的提醒（通过 `contact_reminder_scheduled` 中间表，含 `scheduled_at`、`triggered_at`）

#### 4.1.2 User 模型的渠道关联

[User 模型](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Models/User.php#L220-L223) 定义了与通知渠道的一对多关系：

```php
public function notificationChannels(): HasMany
{
    return $this->hasMany(UserNotificationChannel::class);
}
```

### 4.2 渠道创建与收件人校验

[CreateUserNotificationChannel](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Settings/ManageNotificationChannels/Services/CreateUserNotificationChannel.php#L12-L90) 服务类负责创建通知渠道：

```php
private function validate(): void
{
    $this->validateRules($this->data);
    // 校验 content（邮箱/chat_id）全局唯一性
    $exists = UserNotificationChannel::where('content', $this->data['content'])->exists();
    if ($exists) {
        throw ValidationException::withMessages(['content' => trans('The email is already taken.')]);
    }
}
```

**收件人匹配原则**：
- 渠道的 `content` 字段即为实际收件人地址
- 邮箱渠道：`Mail::to($channel->content)`
- Telegram 渠道：`Notification::route('telegram', $channel->content)`

### 4.3 邮箱验证流程

1. 创建邮箱渠道时生成 `verification_token` UUID
2. 分发 [SendVerificationEmailChannel](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Settings/ManageNotificationChannels/Jobs/SendVerificationEmailChannel.php#L18-L53) 队列任务（high 优先级）
3. 用户点击邮件中的验证链接 → 调用 [VerifyUserNotificationChannelEmailAddress](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Settings/ManageNotificationChannels/Services/VerifyUserNotificationChannelEmailAddress.php#L10-L74) 设置 `verified_at`
4. 验证成功后触发 `ScheduleAllContactRemindersForNotificationChannel` 批量调度该渠道的所有提醒

### 4.4 渠道激活/停用

[ToggleUserNotificationChannel](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Settings/ManageNotificationChannels/Services/ToggleUserNotificationChannel.php#L9-L94) 服务：

- **停用**：删除该渠道所有已调度的提醒（`$channel->contactReminders->each->delete()`）
- **激活**：清零失败次数 + 重新调度所有提醒

---

## 五、调度与队列流程

### 5.1 提醒调度机制

#### 5.1.1 ContactReminder 模型

[ContactReminder](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Models/ContactReminder.php#L10-L74) 定义了联系人提醒：

| 字段 | 说明 |
|------|------|
| `contact_id` | 关联联系人 |
| `label` | 提醒内容标签 |
| `day/month/year` | 提醒日期（year 可为空表示每年重复） |
| `type` | 类型：`one_time`、`recurring_day`、`recurring_month`、`recurring_year` |

#### 5.1.2 新建提醒时的调度

[ScheduleContactReminderForUser](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Services/ScheduleContactReminderForUser.php#L12-L90) 服务在创建提醒时，为用户的每个通知渠道分别计算调度时间：

```php
private function schedule(): void
{
    // 计算下一次触发日期（处理过去日期、年重复等）
    if ($this->upcomingDate->isPast()) {
        $this->upcomingDate->year = Carbon::now()->year;
        if ($this->upcomingDate->isPast()) {
            $this->upcomingDate->year = Carbon::now()->addYear()->year;
        }
    }

    // 遍历用户的每个通知渠道
    foreach ($notificationChannels as $channel) {
        // 转换到用户时区 + 设置用户偏好时间
        $this->upcomingDate->shiftTimezone($this->user->timezone ?? config('app.timezone'));
        $this->upcomingDate->hour = $channel->preferred_time->hour;
        $this->upcomingDate->minute = $channel->preferred_time->minute;

        // 写入 contact_reminder_scheduled 中间表
        $this->contactReminder->userNotificationChannels()->syncWithoutDetaching([$channel->id => [
            'scheduled_at' => $this->upcomingDate->tz('UTC'),
        ]]);
    }
}
```

#### 5.1.3 新建渠道时的批量调度

[ScheduleAllContactRemindersForNotificationChannel](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Settings/ManageNotificationChannels/Services/ScheduleAllContactRemindersForNotificationChannel.php#L11-L101) 服务为新渠道批量调度所有账户下的联系人提醒，逻辑同上。

#### 5.1.4 发送后的重调度

[RescheduleContactReminderForChannel](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannel.php#L13-L88) 在每次提醒成功发送后，根据提醒类型计算下一次触发时间：

```php
switch ($this->contactReminder->type) {
    case ContactReminder::TYPE_RECURRING_DAY:
        $this->upcomingDate = $this->upcomingDate->addDay();
        break;
    case ContactReminder::TYPE_RECURRING_MONTH:
        $this->upcomingDate = $this->upcomingDate->addMonth();
        break;
    case ContactReminder::TYPE_RECURRING_YEAR:
        $this->upcomingDate = $this->upcomingDate->addYear();
        break;
}
```

一次性提醒（`one_time`）发送后直接删除调度记录。

### 5.2 定时触发：Cron 调度

[routes/console.php](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/routes/console.php#L27) 中定义了每分钟执行的定时任务：

```php
Schedule::job(ProcessScheduledContactReminders::class, 'minutes', 1);
```

### 5.3 队列执行：ProcessScheduledContactReminders

[ProcessScheduledContactReminders](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L21-L126) 是核心调度 Job，实现 `ShouldQueue` 接口。

**执行流程**：

```
1. 获取当前时间（秒清零，避免时间窗口问题）
2. 查询 contact_reminder_scheduled 中 scheduled_at <= 当前时间 的所有记录
3. 逐条处理：
   a. 加载 UserNotificationChannel 和 ContactReminder、Contact
   b. 检查渠道是否 active
   c. 通过 NameHelper 格式化联系人姓名
   d. 根据渠道类型调用 Notification::route() 发送
   e. 更新 triggered_at 标记已触发
   f. 递增提醒触发次数 number_times_triggered
   g. 调用 RescheduleContactReminderForChannel 安排下一次
4. 异常处理：
   a. 记录错误日志
   b. 在 user_notification_sent 中记录错误
   c. 累加渠道 fails 计数
   d. 失败次数 >= max_notification_failures（默认10）时自动停用渠道
```

**核心发送代码**：

```php
Notification::route($type, $channel->content)
    ->notify((new ReminderTriggered($channel, $contactReminder->label, $contactName))
        ->locale($channel->user->locale));
```

### 5.4 队列配置

[config/queue.php](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/config/queue.php) 支持多种驱动：

| 驱动 | 说明 |
|------|------|
| `sync` | 同步执行（默认，开发环境） |
| `database` | 数据库队列（使用 `jobs` 表） |
| `redis` | Redis 队列 |
| `beanstalkd` / `sqs` | 其他队列服务 |

验证邮件 Job 指定了 `high` 队列优先级：
```php
SendVerificationEmailChannel::dispatch($this->userNotificationChannel)->onQueue('high');
```

---

## 六、发送记录与日志

### 6.1 UserNotificationSent 模型

[UserNotificationSent](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Models/UserNotificationSent.php#L9-L46) 记录每次通知的发送情况：

| 字段 | 说明 |
|------|------|
| `user_notification_channel_id` | 关联渠道 |
| `sent_at` | 发送时间 |
| `subject_line` | 邮件主题或提醒内容 |
| `payload` | 附加数据 |
| `error` | 错误信息（发送失败时记录） |

### 6.2 记录时机

- **Mailable 路径**：由服务类显式调用 `log()` 方法记录（如 [SendTestEmail@log()](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Settings/ManageNotificationChannels/Services/SendTestEmail.php#L74-L81)）
- **Notification 路径**：在 `toMail()` 和 `toTelegram()` 方法开头创建记录（如 [ReminderTriggered@toMail()](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Notifications/ReminderTriggered.php#L54-L58)）
- **异常路径**：在 Job 的 catch 块中记录错误信息

---

## 七、完整代码链路总览

### 7.1 链路一：邮箱验证邮件发送

```
CreateUserNotificationChannel.execute()
  ├─ create() → 写入 user_notification_channels，生成 verification_token
  └─ verifyChannel()
      └─ SendVerificationEmailChannel::dispatch($channel)->onQueue('high')
          └─ Queue Worker 处理
              └─ SendVerificationEmailChannel.handle()
                  └─ Mail::to($channel->content)->send(new UserNotificationChannelEmailCreated($channel))
                      └─ UserNotificationChannelEmailCreated.build()
                          └─ markdown('emails.notifications.validate-email', ['url' => ...])
                              └─ Blade 渲染 + trans() 变量替换
```

### 7.2 链路二：联系人提醒通知（核心链路）

```
CreateContactReminder / UpdateContactReminder
  └─ ScheduleContactReminderForUser.execute()
      └─ schedule() → 为每个 UserNotificationChannel 写入 contact_reminder_scheduled

Cron（每分钟）
  └─ ProcessScheduledContactReminders::dispatch()
      └─ Queue Worker 处理
          └─ ProcessScheduledContactReminders.handle()
              ├─ 查询所有到期的 contact_reminder_scheduled 记录
              └─ triggerNotification()
                  ├─ NameHelper::formatContactName() → 变量替换格式化姓名
                  ├─ Notification::route($type, $channel->content)
                  │   └─ notify(new ReminderTriggered($channel, $label, $contactName))
                  │       ├─ via() → 根据 type 返回 ['mail'] 或 ['telegram']
                  │       ├─ toMail() / toTelegram()
                  │       │   ├─ 创建 UserNotificationSent 记录
                  │       │   └─ 构建 MailMessage / TelegramMessage（含 trans() 变量替换）
                  │       └─ 实际发送（SwiftMailer / Telegram Bot API）
                  ├─ increment('number_times_triggered')
                  └─ RescheduleContactReminderForChannel.execute() → 安排下一次
```

### 7.3 链路三：测试通知发送

```
NotificationsTestController
  ├─ SendTestEmail.execute() → Mail::to()->send(new TestEmailSent($channel)) + 记录日志
  └─ SendTestTelegramNotification.execute() → Notification::route('telegram', ...)->notify(...)
```

---

## 八、关键设计要点

1. **多渠道抽象**：通过 Laravel Notification 的 `via()` 方法统一分发，新增渠道只需添加类型常量和对应的 `toXxx()` 方法
2. **时区处理**：调度时将 UTC 时间转换为用户时区，结合用户偏好时间（`preferred_time`）计算实际触发时间
3. **容错机制**：单条提醒发送失败不影响整体流程，连续失败超过阈值自动停用渠道
4. **重复调度策略**：周期性提醒在每次发送后自动计算下一次触发时间，支持按天/月/年重复
5. **发送可追溯**：所有发送行为（含失败）均写入 `user_notification_sent` 表，便于审计和排查
