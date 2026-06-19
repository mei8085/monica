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
| [invitation.blade.php](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/resources/views/emails/user/invitation.blade.php#L1-L12) | UserInvited |

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

[RescheduleContactReminderForChannel](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannel.php#L13-L88) 在每次提醒成功发送后，根据提醒类型计算下一次触发时间。

**完整执行流程**：

```
execute(array $data)
  ├─ validateRules() → 校验 contact_reminder_id、user_notification_channel_id、contact_reminder_scheduled_id
  ├─ 加载 ContactReminder 和 UserNotificationChannel
  ├─ 检查渠道是否 active，非活跃则抛出异常
  └─ 检查提醒类型：
      ├─ 若为 TYPE_ONE_TIME → DELETE FROM contact_reminder_scheduled WHERE id = ?
      └─ 若为其他类型 → 执行 schedule()
          ├─ 读取当前调度记录的 scheduled_at
          ├─ 根据类型计算下一次时间：
          │   ├─ TYPE_RECURRING_DAY → addDay()
          │   ├─ TYPE_RECURRING_MONTH → addMonth()
          │   └─ TYPE_RECURRING_YEAR → addYear()
          └─ syncWithoutDetaching() → UPDATE 同一条记录的 scheduled_at
```

**关键代码**：

```php
// 计算下一次时间
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

// 更新同一调度记录，不创建新记录
$this->contactReminder->userNotificationChannels()->syncWithoutDetaching([
    $this->userNotificationChannel->id => [
        'scheduled_at' => $this->upcomingDate,
    ]
]);
```

> **重要**：时间计算基于上一次的 `scheduled_at` 而非当前时间，确保周期准确性（例如每月 31 日的提醒不会因为跨月而偏移）。

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
3. 逐条处理（try 块内）：
   a. 加载 UserNotificationChannel 和 ContactReminder、Contact
   b. 如果 contact !== null，调用 triggerNotification()
      ├─ 检查渠道是否 active，非激活则直接返回
      ├─ NameHelper::formatContactName() → 格式化联系人姓名
      ├─ Notification::route($type, $channel->content)
      │   └─ notify(new ReminderTriggered(...))
      │       ├─ via() → 根据 type 返回 ['mail'] 或 ['telegram']
      │       └─ toMail() / toTelegram()
      │           ├─ 创建 UserNotificationSent 记录（sent_at = 当前时间）
      │           └─ 构建 MailMessage / TelegramMessage（含 trans() 变量替换）
      ├─ updateNumberOfTimesTriggered() → 递增 number_times_triggered
      └─ RescheduleContactReminderForChannel.execute() → 安排下一次
   c. updateScheduledContactReminderTriggeredAt() → 更新 triggered_at 时间标记
4. 异常处理（catch 块）：
   a. 记录错误日志
   b. 在 user_notification_sent 中记录错误信息
   c. 累加渠道 fails 计数
   d. 失败次数 >= max_notification_failures（默认10）时自动停用渠道
```

**关键执行顺序（经代码核准）**：

| 顺序 | 操作 | 代码位置 | 说明 |
|------|------|----------|------|
| 1 | 发送通知 | [ProcessScheduledContactReminders@triggerNotification#L115-L116](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L115-L116) | 内部先创建 UserNotificationSent 记录 |
| 2 | 递增触发次数 | [ProcessScheduledContactReminders@triggerNotification#L118](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L118) | `number_times_triggered` + 1 |
| 3 | 重调度 | [ProcessScheduledContactReminders@triggerNotification#L120-L124](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L120-L124) | 一次性提醒删除调度记录，周期性更新 `scheduled_at` |
| 4 | 标记触发时间 | [ProcessScheduledContactReminders@handle#L53](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L53) | 设置 `triggered_at` = 当前时间，**在 triggerNotification 返回后执行** |

> **注意**：即使 `contact === null`（联系人已删除），仍会执行 `updateScheduledContactReminderTriggeredAt()` 标记 `triggered_at`。但由于查询条件不排除 `triggered_at`，**该记录下次仍会被查询到**（详见 5.3.1 边缘情况分析）。

### 5.3.1 边缘情况深度分析

#### 情况一：一次性提醒删除后，触发时间标记是否生效？

**结论：不生效。**

执行顺序如下：

```
triggerNotification() 内部：
  1. 发送通知
  2. 递增次数
  3. 调用 RescheduleContactReminderForChannel.execute()
     └─ 一次性提醒 → DELETE FROM contact_reminder_scheduled WHERE id = ?

handle() 中 triggerNotification() 返回后：
  4. 调用 updateScheduledContactReminderTriggeredAt()
     └─ UPDATE contact_reminder_scheduled SET triggered_at = ? WHERE id = ?
     └─ ❌ 记录已被删除，UPDATE 影响 0 行
```

**代码依据**：
- 删除操作在 [RescheduleContactReminderForChannel@execute#L56-L58](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannel.php#L56-L58)
- 触发时间标记在 [ProcessScheduledContactReminders@handle#L53](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L53)

**功能影响**：不影响。因为记录已被删除，下次 cron 不会再查询到，`triggered_at` 是否设置没有实际意义。

---

#### 情况二：重调度失败是否进入异常分支？

**结论：是，会进入异常分支。**

`triggerNotification()` 整体在 `handle()` 的 try 块内调用，`RescheduleContactReminderForChannel->execute()` 抛出的任何异常都会冒泡到 catch 块。

**可能导致重调度失败的异常**：

| 异常来源 | 触发条件 | 代码位置 |
|----------|----------|----------|
| `validateRules()` | 参数校验失败（如 id 不存在） | [RescheduleContactReminderForChannel@rules#L26-L33](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannel.php#L26-L33) |
| `findOrFail()` | 找不到 ContactReminder 或 UserNotificationChannel | [RescheduleContactReminderForChannel@execute#L46-L47](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannel.php#L46-L47) |
| `ModelNotFoundException` | 渠道已变为非活跃状态 | [RescheduleContactReminderForChannel@execute#L49-L51](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannel.php#L49-L51) |
| `\Exception` | 无效的提醒类型（default 分支） | [RescheduleContactReminderForChannel@schedule#L80-L82](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Services/RescheduleContactReminderForChannel.php#L80-L82) |

**进入异常分支后的后果**：
1. 记录错误日志
2. 创建 `UserNotificationSent` 记录（含 error 字段）
3. 渠道 `fails` 计数 +1
4. 失败次数达到阈值时自动停用渠道并删除所有调度记录

> **注意**：此时通知可能已经发送成功（发送在重调度之前），但因为重调度失败，整个流程被算作失败。

---

#### 情况三：联系人缺失、渠道停用时是否重复处理？

**结论：会重复处理。**

##### 3.1 联系人缺失（`contact === null`）

```
每次 cron 执行：
  查询 contact_reminder_scheduled WHERE scheduled_at <= NOW()
    → 命中该记录（因为 scheduled_at 从未更新）
    → contact === null，跳过 triggerNotification()
    → 执行 updateScheduledContactReminderTriggeredAt() → 更新 triggered_at
    → 不重调度、不删除
  下次 cron：
    → 再次命中（查询条件不检查 triggered_at）
    → 🔄 无限循环
```

**原因**：查询条件只有 `scheduled_at <= $currentDate`，**不排除已触发的记录**。如果联系人被删除但调度记录仍存在，每次 cron 都会重复标记 `triggered_at`，但不会发送通知。

##### 3.2 渠道停用

分两种场景：

| 场景 | 是否重复处理 | 原因 |
|------|-------------|------|
| 手动停用（ToggleUserNotificationChannel） | ❌ 不会 | 停用时调用 `deleteScheduledReminders()` 删除所有调度记录 |
| triggerNotification 中检测到非活跃 | ✅ 会 | 直接 return，不重调度、不删除记录，scheduled_at 保持不变 |

**第二种场景的执行流程**：

```
每次 cron 执行：
  查询 → 命中记录
  triggerNotification() 入口检查：if (! $channel->active) { return; }
    → 直接返回，不发送、不递增、不重调度
  handle() 中继续执行：updateScheduledContactReminderTriggeredAt()
    → 仅标记 triggered_at
  下次 cron：
    → 再次命中（scheduled_at 未变）
    → 🔄 无限循环
```

**代码依据**：
- 查询条件：[ProcessScheduledContactReminders@handle#L38-L40](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L38-L40)
- 渠道活跃检查：[ProcessScheduledContactReminders@triggerNotification#L97-L99](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Contact/ManageReminders/Jobs/ProcessScheduledContactReminders.php#L97-L99)
- 手动停用删除调度：[ToggleUserNotificationChannel@deleteScheduledReminders#L81-L84](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Settings/ManageNotificationChannels/Services/ToggleUserNotificationChannel.php#L81-L84)

> **实际影响评估**：
> - 手动停用渠道时会删除调度记录，因此正常情况下不会出现重复处理
> - 仅在数据不一致或竞态条件下（如渠道在查询后、处理前被停用）才可能发生
> - 重复处理的代价很小：仅更新 `triggered_at` 字段，不发送通知、不产生副作用

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

### 7.2 链路二：用户邀请邮件发送（从创建到排队）

```
UserController@store() [POST /settings/users]
  └─ InviteUser.execute($data)
      ├─ validateRules() → 校验 account_id、author_id、email、is_administrator
      │   └─ 检查 email 在 users 表中唯一性
      ├─ createUser() → 创建受邀用户记录
      │   └─ User::create([
      │          'account_id' => ...,
      │          'email' => $data['email'],
      │          'invitation_code' => (string) Str::uuid(),
      │          'is_account_administrator' => $data['is_administrator'],
      │       ])
      └─ sendEmail() → 将邮件推入队列
          └─ Mail::to($this->user->email)
              └─ queue(new UserInvited($this->user, $this->author))
                  └─ UserInvited 实现 ShouldQueue 接口 → 进入队列系统
                      └─ Queue Worker 处理
                          └─ UserInvited.build()
                              ├─ 生成邀请链接：route('invitation.show', ['code' => $invitedUser->invitation_code])
                              └─ markdown('emails.user.invitation')
                                  ├─ with('userName', $this->user->name)
                                  ├─ with('url', $invitationRoute)
                                  └─ Blade 渲染
                                      └─ invitation.blade.php → trans(':UserName invites you...', ['userName' => $userName])

用户接受邀请（可选后续）：
AcceptInvitation.execute()
  ├─ findUserByInvitationCode() → 根据 invitation_code 查找用户
  ├─ updateUser() → 设置姓名、密码、invitation_accepted_at、email_verified_at
  └─ createNotificationChannel() → 自动创建默认邮件渠道（直接设置 verified_at 和 active）
```

**关键细节说明**：
- [InviteUser](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Domains/Settings/ManageUsers/Services/InviteUser.php#L45-L70) 使用 `Mail::to()->queue()` 而非 `send()`，确保邮件异步发送不阻塞请求
- [UserInvited](file:///d:/fz/0601-2/solo-dogfeeding/code/36-monica/app/Mail/UserInvited.php#L11-L42) 显式实现 `ShouldQueue` 接口，即使调用 `send()` 也会自动入队
- 邀请邮件的收件人直接从 `User.email` 字段获取，不经过 UserNotificationChannel
- 接受邀请时自动创建的邮件渠道跳过验证流程（`verify_email` = false），直接标记为已验证并激活

### 7.3 链路三：联系人提醒通知（核心链路，执行顺序经代码核准）

```
CreateContactReminder / UpdateContactReminder
  └─ ScheduleContactReminderForUser.execute()
      └─ schedule() → 为每个 UserNotificationChannel 写入 contact_reminder_scheduled

Cron（每分钟）
  └─ ProcessScheduledContactReminders::dispatch()
      └─ Queue Worker 处理
          └─ ProcessScheduledContactReminders.handle()
              ├─ $currentDate = Carbon::now(); $currentDate->second = 0;
              ├─ 查询 contact_reminder_scheduled WHERE scheduled_at <= $currentDate
              └─ foreach ($scheduledContactReminders as $scheduledReminder) {
                  try {
                      加载 UserNotificationChannel、ContactReminder、Contact
                      if ($contact !== null) {
                          triggerNotification($channel, $contact, $contactReminder, $scheduledReminder)
                            │
                            ├─ 【顺序 1】发送通知
                            │   ├─ NameHelper::formatContactName() → 变量替换格式化姓名
                            │   ├─ Notification::route($type, $channel->content)
                            │   │   └─ notify(new ReminderTriggered($channel, $label, $contactName))
                            │   │       ├─ via() → 根据 type 返回 ['mail'] 或 ['telegram']
                            │   │       └─ toMail() / toTelegram()
                            │   │           ├─ ✅ 创建 UserNotificationSent 记录（sent_at = 当前时间）
                            │   │           └─ 构建消息（含 trans() 变量替换）
                            │   └─ 实际发送（SwiftMailer / Telegram Bot API）
                            │
                            ├─ 【顺序 2】递增次数
                            │   └─ updateNumberOfTimesTriggered() → number_times_triggered + 1
                            │
                            └─ 【顺序 3】重调度
                                └─ RescheduleContactReminderForChannel.execute()
                                    ├─ 检查渠道是否 active
                                    ├─ 一次性提醒 → DELETE FROM contact_reminder_scheduled
                                    └─ 周期性提醒 → 计算下一次时间 UPDATE scheduled_at
                      }
                      // 【顺序 4】标记触发时间（在 triggerNotification 之后）
                      updateScheduledContactReminderTriggeredAt() → SET triggered_at = NOW()
                  } catch (\Exception $e) {
                      记录错误日志 + UserNotificationSent（含 error 字段）
                      渠道 fails + 1，超过阈值自动停用
                  }
                }
```

### 7.4 链路四：测试通知发送

```
NotificationsTestController
  ├─ SendTestEmail.execute() → Mail::to()->send(new TestEmailSent($channel)) + 记录日志
  └─ SendTestTelegramNotification.execute() → Notification::route('telegram', ...)->notify(...)
```

---

## 八、关键设计要点

### 8.1 通用设计

1. **多渠道抽象**：通过 Laravel Notification 的 `via()` 方法统一分发，新增渠道只需添加类型常量和对应的 `toXxx()` 方法
2. **时区处理**：调度时将 UTC 时间转换为用户时区，结合用户偏好时间（`preferred_time`）计算实际触发时间
3. **容错机制**：单条提醒发送失败不影响整体流程，连续失败超过阈值自动停用渠道
4. **重复调度策略**：周期性提醒在每次发送后自动计算下一次触发时间，支持按天/月/年重复
5. **发送可追溯**：所有发送行为（含失败）均写入 `user_notification_sent` 表，便于审计和排查

### 8.2 用户邀请流程设计

6. **异步非阻塞**：邀请邮件采用 `Mail::to()->queue()` 异步发送，避免阻塞 HTTP 请求
7. **受邀用户预创建**：受邀用户在发送邀请前即创建账户（含 `invitation_code`），接受邀请时补全用户信息
8. **邮件渠道自动创建**：接受邀请时自动创建邮件通知渠道，跳过验证流程直接激活

### 8.3 提醒通知执行顺序设计

9. **发送记录先于统计更新**：`UserNotificationSent` 记录在 `toMail()`/`toTelegram()` 开头创建，确保发送行为可追溯，即使后续步骤失败也有记录
10. **触发时间标记后置**：`triggered_at` 在 `triggerNotification()` 完成后才更新，确保只有被正确执行完毕后才标记为已触发
11. **重调度与发送解耦**：重调度逻辑在发送成功后独立执行，避免发送失败时仍可标记为已触发（即使重调度失败）
12. **容错边界处理**：即使联系人已删除（`contact === null`）仍标记为已触发，避免僵尸调度记录重复执行

### 8.4 重调度机制细节

- **一次性提醒**：发送后直接 `DELETE` 调度记录，不再保留
- **周期性提醒**：`UPDATE` 同一条调度记录的 `scheduled_at` 为下一次时间，避免创建新记录
- **渠道状态校验**：重调度前检查 `userNotificationChannel.active`，确保渠道未被停用
- **类型驱动计算**：基于上一次 `scheduled_at` 计算下一次时间，而非当前时间，确保周期准确
