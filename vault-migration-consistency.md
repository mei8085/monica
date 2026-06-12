# 联系人跨保险库迁移与成员移除一致性分析

## 1. 概述

本文基于 Monica 项目源码，深入分析以下两个核心问题：
1. **联系人跨保险库移动后，除公司外还有哪些关联数据会"串"到旧保险库**
2. **成员移除保险库时，权限清理到底靠什么生效**

---

## 2. 联系人跨保险库移动的关联数据影响

### 2.1 移动操作的现状

当前 [MoveContactToAnotherVault.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Services/MoveContactToAnotherVault.php#L49-L115) 只处理了两件事：
1. 更新联系人的 `vault_id` 字段
2. 处理关联公司的迁移（移动或复制）

```php
public function execute(array $data): Contact
{
    $this->data = $data;
    $this->validate();
    $this->move();                    // 仅更新 vault_id
    $this->moveCompanyInformation();  // 仅处理公司
    $this->updateLastEditedDate();
    return $this->contact;
}
```

### 2.2 受影响的关联数据全景

根据数据模型分析，以下关联数据在联系人移动后会出现归属不一致问题：

#### 2.2.1 直接属于保险库的实体（Vault-scoped Entities）

这类实体自身带有 `vault_id` 字段，与联系人通过中间表关联。联系人移动后，**中间表记录仍然指向旧保险库的实体**。

| 实体类型 | 模型文件 | 中间表 | 影响说明 |
|---------|---------|--------|---------|
| **标签 (Label)** | [Label.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Label.php) | `contact_label` | 标签属于保险库，联系人移动后标签关系留在旧保险库 |
| **分组 (Group)** | [Group.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Group.php) | `contact_group` | 分组属于保险库，联系人移动后分组关系留在旧保险库 |
| **地址 (Address)** | [Address.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Address.php) | `contact_address` | 地址属于保险库，联系人移动后地址关系留在旧保险库 |
| **借贷 (Loan)** | [Loan.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Loan.php) | `contact_loan` | 借贷属于保险库，联系人移动后借贷关系留在旧保险库 |
| **时间线事件 (TimelineEvent)** | [TimelineEvent.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/TimelineEvent.php) | `timeline_event_participants` | 时间线属于保险库，移动后参与者关系失效 |
| **生活事件 (LifeEvent)** | [LifeEvent.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/LifeEvent.php) | `life_event_participants` | 生活事件通过时间线归属保险库，移动后参与者关系失效 |
| **日志/帖子 (Post)** | [Post.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Post.php) | `contact_post` | 帖子属于保险库，移动后关联失效 |
| **文件 (File)** | [File.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/File.php) | 多态关联 | 文件属于保险库，移动后文件仍在旧保险库 |

**标签（Label）数据结构示例** [2021_10_19_192432_create_labels_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/database/migrations/2021_10_19_192432_create_labels_table.php#L17-L32)：

```php
Schema::create('labels', function (Blueprint $table) {
    $table->id();
    $table->foreignIdFor(Vault::class)->constrained()->cascadeOnDelete();  // 标签属于保险库
    $table->string('name');
    // ...
});

Schema::create('contact_label', function (Blueprint $table) {
    $table->foreignIdFor(Label::class)->constrained()->cascadeOnDelete();
    $table->foreignIdFor(Contact::class)->constrained()->cascadeOnDelete();  // 中间表关联
    // ...
});
```

> **问题本质**：联系人的 `vault_id` 变了，但中间表记录还指向旧保险库的标签/分组等实体。这些旧保险库的标签/分组在新保险库中不存在，也无法访问。

#### 2.2.2 直接属于联系人的实体（Contact-scoped Entities）

这类实体通过 `contact_id` 外键直接关联联系人，**没有独立的 `vault_id`**。联系人移动后，这些数据会跟着联系人走，不会出现归属不一致。

| 实体类型 | 模型文件 | 是否有 vault_id | 移动后状态 |
|---------|---------|----------------|-----------|
| **笔记 (Note)** | [Note.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Note.php) | ✅ 有 | 有独立 vault_id，移动后仍指向旧保险库 ⚠️ |
| **联系人信息 (ContactInformation)** | - | ❌ 无 | 跟随联系人，正常 |
| **重要日期 (ContactImportantDate)** | - | ❌ 无 | 跟随联系人，正常 |
| **任务 (ContactTask)** | - | ❌ 无 | 跟随联系人，正常 |
| **通话记录 (Call)** | - | ❌ 无 | 跟随联系人，正常 |
| **目标 (Goal)** | [Goal.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Goal.php) | ❌ 无 | 跟随联系人，正常 |
| **宠物 (Pet)** | - | ❌ 无 | 跟随联系人，正常 |
| **心情追踪 (MoodTrackingEvent)** | - | ❌ 无 | 跟随联系人，正常 |
| **快速事实 (QuickFact)** | [QuickFact.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/QuickFact.php) | ❌ 无 | 跟随联系人，正常 |
| **关系 (Relationship)** | - | ❌ 无 | 跟随联系人，但对方可能在旧保险库 ⚠️ |

**特别注意 - 笔记（Note）** [Note.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Note.php#L23-L30)：

```php
protected $fillable = [
    'contact_id',
    'vault_id',    // 笔记同时带有 contact_id 和 vault_id
    'author_id',
    'emotion_id',
    'title',
    'body',
];
```

> ⚠️ **笔记是双重归属**：既有 `contact_id` 又有 `vault_id`。联系人移动后，笔记的 `vault_id` 不会自动更新，导致笔记在数据库层面仍属于旧保险库。

#### 2.2.3 用户级别的关联数据（User-scoped Entities）

这类数据与用户和联系人都有关联，存储用户对联系人的个性化操作记录。

| 实体类型 | 表名 | 影响说明 |
|---------|------|---------|
| **收藏标记** | `contact_vault_user` | 收藏记录同时带有 vault_id，移动后旧保险库的收藏记录失效 |
| **查看次数** | `contact_vault_user` | 查看次数统计留在旧保险库 |
| **联系人提醒调度** | `contact_reminder_scheduled` | 提醒是按用户通知渠道调度的，移动后可能仍会触发 |

**contact_vault_user 表结构** [2020_04_25_133132_create_contacts_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L69-L76)：

```php
Schema::create('contact_vault_user', function (Blueprint $table) {
    $table->foreignIdFor(Contact::class)->constrained()->cascadeOnDelete();
    $table->foreignIdFor(Vault::class)->constrained()->cascadeOnDelete();    // 带有保险库ID
    $table->foreignIdFor(User::class)->constrained()->cascadeOnDelete();
    $table->integer('number_of_views');   // 查看次数
    $table->boolean('is_favorite')->default(false);  // 收藏标记
    $table->timestamps();
});
```

**查看次数更新逻辑** [UpdateContactView.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Services/UpdateContactView.php#L51-L72)：

```php
private function updateView(): void
{
    $contact = [
        'contact_id' => $this->data['contact_id'],
        'vault_id' => $this->data['vault_id'],    // 使用的是当前请求的 vault_id
        'user_id' => $this->data['author_id'],
    ];

    $exists = DB::table('contact_vault_user')->where($contact)->exists();

    if ($exists) {
        DB::table('contact_vault_user')->where($contact)->increment('number_of_views');
    } else {
        DB::table('contact_vault_user')->insert($contact + [
            'number_of_views' => 1,
        ]);
    }
}
```

> ⚠️ **收藏/查看统计的数据孤岛问题**：`contact_vault_user` 表同时带有 `contact_id`、`vault_id`、`user_id` 三个外键。联系人移动后：
> - 旧保险库的收藏/查看记录仍然存在，但联系人已不在旧保险库
> - 新保险库没有该用户对该联系人的收藏/查看记录
> - 用户在新保险库看到的是"全新"的联系人（无收藏、零次查看）

#### 2.2.4 联系人关系（Relationships）

联系人之间的关系（如朋友、同事、家人等）通过 `relationships` 中间表建立。联系人移动后：

- 如果关系的另一方在**同一个保险库**，移动后关系仍然有效（但两边保险库不同了）
- 如果关系的另一方在**不同保险库**，移动后关系可能变成跨保险库的引用

> ⚠️ **关系数据的跨库问题**：关系表中只存了 `contact_id` 和 `related_contact_id`，没有 `vault_id`。但联系人本身有 `vault_id`，所以关系隐式受到保险库边界限制。联系人移动后，关系记录仍然存在，但实际上变成了跨保险库的关系引用，可能导致访问异常。

#### 2.2.5 动态流（Feed Items）

[ContactFeedItem.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/ContactFeedItem.php) 记录联系人的所有操作动态。

```php
protected $fillable = [
    'author_id',
    'contact_id',
    'action',
    'description',
    'feedable_id',
    'feedable_type',
];
```

> Feed Item 本身没有 `vault_id`，通过 `contact_id` 间接关联保险库。联系人移动后，历史 Feed 记录会跟着联系人走，不会出现归属不一致。

### 2.3 关联数据影响汇总表

| 类别 | 数据项 | 是否有独立 vault_id | 移动后是否自动迁移 | 风险等级 |
|------|--------|-------------------|-------------------|---------|
| **公司** | Company | ✅ 是 | ✅ 已处理（移动/复制） | 低 |
| **笔记** | Note | ✅ 是 | ❌ 未处理 | 🔴 高 |
| **标签** | Label + contact_label | ✅ 是（标签） | ❌ 未处理 | 🔴 高 |
| **分组** | Group + contact_group | ✅ 是（分组） | ❌ 未处理 | 🔴 高 |
| **地址** | Address + contact_address | ✅ 是（地址） | ❌ 未处理 | 🟡 中 |
| **借贷** | Loan + contact_loan | ✅ 是（借贷） | ❌ 未处理 | 🟡 中 |
| **时间线事件** | TimelineEvent + participants | ✅ 是 | ❌ 未处理 | 🟡 中 |
| **生活事件** | LifeEvent + participants | ❌（通过时间线） | ❌ 未处理 | 🟡 中 |
| **帖子** | Post + contact_post | ✅ 是 | ❌ 未处理 | 🟡 中 |
| **文件** | File | ✅ 是 | ❌ 未处理 | 🟡 中 |
| **收藏标记** | contact_vault_user | ✅ 是 | ❌ 未处理 | 🟡 中 |
| **查看统计** | contact_vault_user | ✅ 是 | ❌ 未处理 | 🟡 中 |
| **联系人关系** | relationships | ❌ | 隐式跨库 | 🟡 中 |
| **联系人信息** | ContactInformation | ❌ | ✅ 跟随联系人 | 低 |
| **重要日期** | ContactImportantDate | ❌ | ✅ 跟随联系人 | 低 |
| **任务** | ContactTask | ❌ | ✅ 跟随联系人 | 低 |
| **通话记录** | Call | ❌ | ✅ 跟随联系人 | 低 |
| **目标** | Goal | ❌ | ✅ 跟随联系人 | 低 |
| **宠物** | Pet | ❌ | ✅ 跟随联系人 | 低 |
| **心情追踪** | MoodTrackingEvent | ❌ | ✅ 跟随联系人 | 低 |
| **快速事实** | QuickFact | ❌ | ✅ 跟随联系人 | 低 |
| **动态流** | ContactFeedItem | ❌ | ✅ 跟随联系人 | 低 |

---

## 3. 成员移除时权限清理机制

### 3.1 清理入口

移除保险库成员的服务是 [RemoveVaultAccess.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php)：

```php
public function execute(array $data): void
{
    $this->data = $data;
    $this->validate();
    $this->remove();                              // 1. 删除关联联系人
    $this->removeAllRemindersForThisUserInThisVault();  // 2. 清理提醒调度
}
```

### 3.2 清理机制详解

#### 3.2.1 第一层：数据库级联删除（Database Cascade）

这是最核心的清理机制，通过数据库外键约束自动完成。

**user_vault 表的级联关系** [2020_04_25_133132_create_contacts_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/database/migrations/2020_04_25_133132_create_contacts_table.php#L61-L67)：

```php
Schema::create('user_vault', function (Blueprint $table) {
    $table->foreignIdFor(Vault::class)->constrained()->cascadeOnDelete();    // 删保险库 → 删关联
    $table->foreignIdFor(User::class)->constrained()->cascadeOnDelete();     // 删用户 → 删关联
    $table->foreignIdFor(Contact::class)->constrained()->cascadeOnDelete();  // 删联系人 → 删关联 ★
    $table->integer('permission');
    $table->timestamps();
});
```

**移除操作的实现** [RemoveVaultAccess.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php#L73-L82)：

```php
private function remove(): void
{
    $vault = $this->user->vaults()
        ->wherePivot('vault_id', $this->vault->id)
        ->first();

    if ($vault !== null) {
        // ★ 关键：删除联系人会级联删除 user_vault 记录
        Contact::find($vault->pivot->contact_id)->delete();
    }
}
```

> **设计巧妙之处**：不直接删除 `user_vault` 记录，而是删除关联的联系人。利用外键级联删除特性，联系人删除会自动级联删除 `user_vault` 中的关联记录。

#### 3.2.2 第二层：应用层手动清理

数据库级联无法覆盖所有场景，需要应用层补充清理。

**提醒调度清理** [RemoveVaultAccess.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php#L88-L95)：

```php
private function removeAllRemindersForThisUserInThisVault(): void
{
    $this->user->notificationChannels->each(function (UserNotificationChannel $notificationChannel) {
        $notificationChannel->contactReminders->each(function ($reminder) {
            $reminder->pivot->delete();  // 删除 contact_reminder_scheduled 记录
        });
    });
}
```

> ⚠️ **当前实现的局限性**：
> - 代码注释说"移除该保险库下的所有提醒"，但实际上是删除**所有**通知渠道的**所有**提醒调度
> - 没有按保险库过滤，会把用户在所有保险库的提醒都清掉
> - 如果用户有多个保险库的访问权，移除一个保险库会导致所有保险库的提醒都被清除

**contact_reminder_scheduled 表结构** [2022_02_18_215852_create_reminders_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/database/migrations/2022_02_18_215852_create_reminders_table.php#L47-L54)：

```php
Schema::create('contact_reminder_scheduled', function (Blueprint $table) {
    $table->id();
    $table->foreignIdFor(UserNotificationChannel::class)->constrained()->cascadeOnDelete();
    $table->foreignIdFor(ContactReminder::class)->constrained()->cascadeOnDelete();
    $table->datetime('scheduled_at');
    $table->datetime('triggered_at')->nullable();
    $table->timestamps();
});
```

> 表本身没有 `vault_id`，无法直接按保险库过滤。要判断提醒属于哪个保险库，需要通过 `contact_reminder → contact → vault` 关联追溯。

#### 3.2.3 第三层：其他隐式清理

还有一些数据的清理是通过其他级联关系间接完成的：

| 数据项 | 清理方式 | 触发条件 |
|--------|---------|---------|
| **user_vault 关联** | 外键级联删除 | 删除关联联系人时触发 |
| **联系人本身** | 软删除/硬删除 | 直接调用 delete() |
| **联系人的所有关联数据** | 外键级联 | 联系人删除时触发（notes, tasks, calls 等） |
| **contact_vault_user 记录** | 外键级联 | 联系人或用户或保险库删除时 |
| **contact_reminder_scheduled** | 应用层手动删除 | 移除权限时调用 |
| **user_notification_sent** | 外键级联 | 删除通知渠道时触发 |

### 3.3 权限清理的完整链路

```
RemoveVaultAccess::execute()
│
├─ validate()                 ← 权限校验
│
├─ remove()                   ← 删除关联联系人
│   │
│   └─ Contact::delete()      ← 触发数据库级联
│       │
│       ├─ user_vault 记录被删  ← 外键：contact_id → cascade
│       ├─ 笔记被删              ← 外键：contact_id → cascade
│       ├─ 任务被删              ← 外键：contact_id → cascade
│       ├─ 通话记录被删           ← 外键：contact_id → cascade
│       ├─ 重要日期被删           ← 外键：contact_id → cascade
│       ├─ 联系人信息被删         ← 外键：contact_id → cascade
│       ├─ contact_vault_user 被删 ← 外键：contact_id → cascade
│       └─ ...其他关联数据...
│
└─ removeAllRemindersForThisUserInThisVault()
    │
    └─ 遍历所有通知渠道
        └─ 遍历所有提醒调度
            └─ 删除 pivot 记录（contact_reminder_scheduled）
```

### 3.4 清理机制的问题与风险

#### 问题1：提醒清理范围过大

[RemoveVaultAccess.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php#L88-L95) 的方法名和注释都说是"该保险库下的"，但实际代码清掉了用户所有保险库的提醒：

```php
// 注释说的是"在这个保险库中"
private function removeAllRemindersForThisUserInThisVault(): void
{
    // 但实际上遍历的是用户的所有通知渠道的所有提醒
    $this->user->notificationChannels->each(function (UserNotificationChannel $notificationChannel) {
        $notificationChannel->contactReminders->each(function ($reminder) {
            $reminder->pivot->delete();
        });
    });
}
```

#### 问题2：缺少事务保护

整个移除操作没有数据库事务包裹。如果在删除联系人后、清理提醒前发生异常，会导致：
- 用户权限已被移除（联系人已删）
- 但提醒调度没有被清理
- 可能继续收到已移除保险库的提醒

#### 问题3：Scout 搜索索引未同步清理

联系人删除时触发模型事件，但 Scout 搜索索引的更新依赖模型的 `boot` 方法。保险库级别的搜索数据可能需要额外处理。

---

## 4. 设计模式与架构思考

### 4.1 "用户即联系人"模式

系统中一个关键设计是：每个用户在每个保险库中都有一个对应的联系人记录（"自己"）。

**创建保险库时** [CreateVault.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVault/Services/CreateVault.php#L80-L94)：
```php
private function createUserContact(): void
{
    $contact = Contact::create([
        'vault_id' => $this->vault->id,
        'first_name' => $this->author->first_name,
        'last_name' => $this->author->last_name,
        'can_be_deleted' => false,    // 不能删除
        'template_id' => $this->vault->default_template_id,
    ]);

    $this->vault->users()->save($this->author, [
        'permission' => Vault::PERMISSION_MANAGE,
        'contact_id' => $contact->id,  // 关联到这个联系人
    ]);
}
```

**这种设计的好处**：
1. 用户在保险库中也是一个"联系人"，可以参与关系、分组等所有联系人功能
2. 权限关联通过联系人级联删除，简化了清理逻辑
3. 统一了数据模型

**代价**：
1. 移除用户时会删除其联系人记录，导致历史数据（如动态流）丢失
2. 联系人移动逻辑也需要考虑"用户自己"的联系人

### 4.2 保险库边界的设计选择

Monica 选择了**严格的保险库隔离**模型：
- 大部分数据（标签、分组、地址、公司等）都直接属于保险库
- 联系人通过 `vault_id` 明确归属
- 跨保险库操作需要显式处理（移动、复制）

这种设计的利弊：
- ✅ 数据隔离清晰，权限模型简单
- ✅ 删除保险库时清理简单（级联删除即可）
- ❌ 跨保险库移动联系人代价高，需要处理大量关联数据
- ❌ 数据共享困难（如标签需要在每个保险库重复创建）

---

## 5. 改进建议

### 5.1 联系人移动的改进

1. **补充关联数据迁移**：至少应该迁移笔记、标签、分组、地址等高频数据
2. **使用数据库事务**：包裹整个移动操作，保证原子性
3. **处理关系数据**：明确跨保险库关系的处理策略（保留 / 断开 / 复制）
4. **迁移收藏和查看记录**：将 `contact_vault_user` 中的记录同步更新 `vault_id`

### 5.2 成员移除的改进

1. **修复提醒清理范围**：只清理目标保险库的提醒，而不是所有提醒
2. **添加事务保护**：将删除联系人和清理提醒放在同一事务中
3. **补充更多清理**：如收藏记录、查看统计等用户级数据也应清理

---

## 6. 关键文件索引

| 文件 | 功能 |
|------|------|
| [MoveContactToAnotherVault.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Services/MoveContactToAnotherVault.php) | 联系人移动服务 |
| [CopyContactToAnotherVault.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Services/CopyContactToAnotherVault.php) | 联系人复制服务 |
| [RemoveVaultAccess.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVaultSettings/Services/RemoveVaultAccess.php) | 移除保险库成员服务 |
| [GrantVaultAccessToUser.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Vault/ManageVaultSettings/Services/GrantVaultAccessToUser.php) | 授予保险库访问权 |
| [Note.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Note.php) | 笔记模型（双重归属） |
| [Label.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Label.php) | 标签模型（保险库归属） |
| [Group.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Group.php) | 分组模型（保险库归属） |
| [Contact.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/Contact.php) | 联系人模型 |
| [User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Models/User.php) | 用户模型 |
| [ToggleFavoriteContact.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Services/ToggleFavoriteContact.php) | 收藏切换服务 |
| [UpdateContactView.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/app/Domains/Contact/ManageContact/Services/UpdateContactView.php) | 更新查看次数服务 |
| [2020_04_25_133132_create_contacts_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/database/migrations/2020_04_25_133132_create_contacts_table.php) | 核心表结构迁移 |
| [2022_02_18_215852_create_reminders_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/28-monica/database/migrations/2022_02_18_215852_create_reminders_table.php) | 提醒表结构迁移 |
