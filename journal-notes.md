# 联系人笔记与 Journal 展示关系梳理

## 一、核心数据模型

### 1. Note（联系人笔记）

**模型**：[Note.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Models/Note.php)

**表结构**（`notes` 表）：
| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| contact_id | uuid | 所属联系人（外键，级联删除） |
| vault_id | uuid | 所属 Vault（外键，级联删除） |
| author_id | uuid | 作者 User（外键，nullOnDelete） |
| emotion_id | int | 关联情绪（外键，nullOnDelete） |
| title | string(255) | 标题，可空 |
| body | text | 正文内容，必填，最大 65535 |
| created_at / updated_at | timestamp | 时间戳 |

**关联关系**：
- `contact()` → BelongsTo Contact
- `author()` → BelongsTo User
- `emotion()` → BelongsTo Emotion
- `feedItem()` → MorphOne ContactFeedItem（多态关联）

### 2. Journal（日记本）

**模型**：[Journal.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Models/Journal.php)

**表结构**（`journals` 表）：
| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| vault_id | uuid | 所属 Vault |
| name | string | 日记本名称 |
| description | string | 描述 |

**关联关系**：
- `vault()` → BelongsTo Vault
- `posts()` → HasMany Post
- `slicesOfLife()` → HasMany SliceOfLife
- `journalMetrics()` → HasMany JournalMetric

### 3. Post（日记条目）

**模型**：[Post.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Models/Post.php)

**表结构**（`posts` 表）：
| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| journal_id | bigint | 所属日记本（外键，级联删除） |
| slice_of_life_id | bigint | 关联 SliceOfLife |
| title | string(255) | 标题 |
| view_count | int | 阅读次数 |
| published | boolean | 是否发布 |
| written_at | datetime | 撰写日期 |

**关联关系**：
- `journal()` → BelongsTo Journal
- `postSections()` → HasMany PostSection
- `contacts()` → BelongsToMany Contact（通过 `contact_post` 中间表）
- `feedItem()` → MorphOne ContactFeedItem（多态关联）
- `tags()` → BelongsToMany Tag
- `files()` → MorphMany File

### 4. PostSection（日记条目段落）

**模型**：[PostSection.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Models/PostSection.php)

**表结构**（`post_sections` 表）：
| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| post_id | bigint | 所属 Post（外键，级联删除） |
| position | int | 排序位置 |
| label | string | 段落标签名 |
| content | text | 段落内容，可空 |

### 5. ContactFeedItem（联系人动态/时间线条目）

**模型**：[ContactFeedItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Models/ContactFeedItem.php)

**表结构**（`contact_feed_items` 表）：
| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| author_id | uuid | 操作者 User（nullOnDelete） |
| contact_id | uuid | 所属联系人（级联删除） |
| action | string | 动作类型常量 |
| description | string | 描述，可空 |
| feedable_id | numeric | 多态关联 ID，可空 |
| feedable_type | string | 多态关联类型，可空 |

**与笔记相关的 action 常量**：
- `ACTION_NOTE_CREATED = 'note_created'`
- `ACTION_NOTE_UPDATED = 'note_updated'`
- `ACTION_NOTE_DESTROYED = 'note_destroyed'`

**与 Journal Post 相关的 action 常量**：
- `ACTION_ADDED_TO_POST = 'added_to_post'`
- `ACTION_REMOVED_FROM_POST = 'removed_from_post'`

---

## 二、Note 与 Journal 的关系——两套并行系统

**Note 和 Journal 没有直接的数据关联**，它们是两套独立的系统：

```
Contact ──── 1:N ──── Note（直接归属，contact_id 外键）
   │
   └── N:M ──── Post（通过 contact_post 中间表关联）
                    │
                    └── N:1 ──── Journal
```

- **Note** 直接属于 Contact，是联系人的附属记录
- **Post** 属于 Journal，Post 可以通过 `contact_post` 中间表与 Contact 关联
- 联系人详情页同时展示 Notes 模块和 Posts 模块（二者并列展示）

---

## 三、富文本内容：存储与渲染

### 3.1 Note 的富文本

**输入端**：
- 控制器：[ContactModuleNoteController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Web/Controllers/ContactModuleNoteController.php) — `store`/`update` 方法接收 `body` 字段
- 前端组件：[Notes.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Shared/Modules/Notes.vue) 使用 `<text-area :markdown="true">` 组件
- `TextArea.vue` 组件在 `markdown=true` 时会显示 Markdown 格式提示，但输入本身仍是纯文本

**存储端**：
- `notes.body` 为 TEXT 类型，存储原始纯文本/Markdown 源码
- [CreateNote.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/CreateNote.php) 和 [UpdateNote.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/UpdateNote.php) 直接将 body 存入数据库

**展示端**：
- [Notes.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Shared/Modules/Notes.vue) 第 105 行：`<div v-else class="p-3 whitespace-pre-line">{{ note.body }}</div>`
- **使用 `{{ }}` 插值（而非 `v-html`），Markdown 不会被渲染为 HTML**
- 仅通过 `whitespace-pre-line` CSS 保留换行
- 超过 200 字符时显示摘要（`body_excerpt`），点击 "View all" 展开全文

**结论**：Note 支持 Markdown 输入提示，但 **不渲染 Markdown**，仅以纯文本 + 保留换行的方式展示。

### 3.2 Post 的富文本

**输入端**：
- 控制器：[PostController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Vault/ManageJournals/Web/Controllers/PostController.php) — `update` 方法接收 `sections` 数组
- 前端组件：[Post/Edit.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Pages/Vault/Journal/Post/Edit.vue) 使用 `<text-area :markdown="true">` 对每个 section 的 content 进行编辑

**存储端**：
- `post_sections.content` 为 TEXT 类型，存储原始 Markdown 源码
- [UpdatePost.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Vault/ManageJournals/Services/UpdatePost.php) 的 `updateSections()` 方法直接保存 content

**展示端**：
- [PostShowViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/PostShowViewHelper.php) 第 128 行：
  ```php
  'content' => (string) Str::of($section->content)->markdown([
      'html_input' => 'strip',
      'allow_unsafe_links' => false,
  ]),
  ```
  **服务端将 Markdown 转为 HTML**
- [Post/Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Pages/Vault/Journal/Post/Show.vue) 第 142 行：`<div class="mb-6" v-html="section.content"></div>`
- **使用 `v-html` 渲染，Markdown 确实被渲染为 HTML**
- 安全措施：`html_input => 'strip'` 和 `allow_unsafe_links => false`

**结论**：Post 支持 Markdown 输入且 **真正渲染 Markdown 为 HTML**，在服务端做转换，前端用 v-html 展示。

### 3.3 两者富文本差异对比

| 维度 | Note | Post |
|------|------|------|
| 内容存储字段 | 单个 `body` 字段 | 多个 `post_sections.content` 字段 |
| 输入组件 | TextArea(:markdown="true") | TextArea(:markdown="true") |
| 服务端 Markdown 转 HTML | ❌ 不转换 | ✅ Str::markdown() 转换 |
| 前端渲染方式 | `{{ note.body }}` 纯文本插值 | `v-html="section.content"` HTML 渲染 |
| 换行处理 | CSS `whitespace-pre-line` | Markdown → HTML `<p>`/`<br>` |
| 安全性 | 天然安全（纯文本） | strip HTML input + 禁止不安全链接 |

---

## 四、时间线展示逻辑

### 4.1 联系人 Feed（动态时间线）

Feed 是联系人详情页的一个模块，展示该联系人的所有操作历史。

**数据加载流程**：

1. [ContactShowViewHelper::modules()](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php#L229-L313) 识别 `TYPE_FEED` 模块时，只传递一个 API URL：
   ```php
   $data = route('contact.feed.show', ['vault' => ..., 'contact' => ...]);
   ```

2. [Feed.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Shared/Modules/Feed.vue) 组件在 `mounted()` 时异步请求数据：
   ```js
   axios.get(this.url).then(response => {
       response.data.data.items.forEach(entry => this.feed.push(entry));
   });
   ```

3. 后端通过 [ModuleFeedViewHelper::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php) 将每个 `ContactFeedItem` 映射为：
   - `id`、`action`、`author`（头像+姓名+链接）
   - `sentence`（动作描述，如 "wrote a note"、"added the contact to a post"）
   - `data`（动作详细数据，不同 action 类型由不同 Helper 处理）
   - `created_at`（格式化时间）

4. 根据 `action` 值渲染不同的子组件

**Note 在 Feed 中的展示**：

- `note_created` / `note_updated` → ✅ 使用 [FeedItems/Note.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Shared/Modules/FeedItems/Note.vue)
- `note_destroyed` → ❌ **前端 Bug：不渲染专用组件**（Feed.vue 中写的是 `'note_deleted'`，与后端 `'note_destroyed'` 不匹配），只显示 "deleted a note" sentence，无内容预览
- 数据由 [ActionFeedNote::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedNote.php) 生成：
  ```php
  'note' => [
      'object' => $note ? ['id' => ..., 'title' => ..., 'body' => Str::limit($note->body, 30)] : null,
      'description' => $item->description,
  ]
  ```
- **如果笔记仍存在且 feedable 关联未断开**：显示 title 和 body（截断到 30 字符）
- **如果笔记已删除或 feedable 关联已断开**（`feedable` 为 null）：显示 description（创建/更新时保存的 10 词摘要），灰色标签样式

**Post 在 Feed 中的展示**：

- `added_to_post` / `removed_from_post` → 无专用子组件，Feed.vue 中走 default 分支
- [ModuleFeedViewHelper::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L21-L35) 返回结构中**没有顶层 description 字段**，因此 Feed.vue 第 145-152 行的 `<div v-if="feedItem.description">` fallback 对这些 action 也不生效
- 实际效果：仅显示 sentence（如 "added the contact to a post"），不显示 post title 等内容预览

### 4.2 Journal 页面的时间线展示

[Journal Show 页面](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Pages/Vault/Journal/Show.vue) 以**年-月**维度组织 Posts：

1. [JournalShowViewHelper::postsInYear()](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/JournalShowViewHelper.php#L67-L144) 按 `written_at` 的月份分组，12 个月倒序排列
2. 每个月的 Posts 展示：日期块、标题、excerpt（第一个有内容的 section 的前 200 字符）、缩略图
3. 左侧栏展示年份列表（带各年 post 数量），可按年切换
4. 右侧栏展示 Slices of Life 列表

**注意**：Journal 页面**不展示 Note**，只展示属于该 Journal 的 Posts。

### 4.3 联系人详情页的 Posts 模块

[ContactShowViewHelper::modules()](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php#L297-L299) 中 `TYPE_POSTS` 模块调用 `ModulePostsViewHelper::data()`：

- 列出与该 Contact 关联的所有 Posts（通过 `contact_post` 中间表）
- 每条 Post 展示：title、journal 名称和链接、written_at 日期、链接到 post.show
- 前端组件 [Posts.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Shared/Modules/Posts.vue) 是一个简单的列表

---

## 五、删除笔记后在 Feed 中的完整展示链路

本节专门回答"删除笔记后，联系人动态中能不能看到笔记内容"这个问题。结论先行：

- **删除操作那条 feedItem（note_destroyed）——看不到任何摘要内容，只能看到 "deleted a note" 这句话**
- **删除前的创建/编辑记录（note_created / note_updated）——有条件地看到 10 词摘要**
- **根本原因是前端 Bug：action 名 `note_deleted` ≠ 后端常量 `note_destroyed`**

---

### 5.1 数据流总览：从删除操作到前端渲染

```
用户点击删除
    │
    ▼
DestroyNote::execute()
    ├─► 创建 ContactFeedItem（action = 'note_destroyed', description = body 前10词）
    │    └─► 注意：用 ContactFeedItem::create() 直接创建，未走 morph 关联
    │    └─► 因此 feedable_id = null, feedable_type = null
    ├─► $this->note->delete()  ← 硬删除，notes 表行彻底消失
    └─► 更新 contact.last_updated_at
    │
    ▼
用户打开联系人 Feed 页面
    │
    ▼
Feed.vue mounted() → axios.get(this.url)
    │
    ▼
ModuleFeedViewHelper::data() 把每个 DB 行映射为前端数据结构
    ├─► id / action / author / sentence / data / created_at
    ├─► 注意：返回数据中**没有顶层 description 字段**
    └─► action 为 note_created / note_updated / note_destroyed 时
         └─► getData() → ActionFeedNote::data($item)
              ├─► $note = $item->feedable  ← 通过 morphTo 加载
              ├─► 'object' => $note ? ['id','title','body（30字截断）'] : null
              └─► 'description' => $item->description  ← DB 中保存的 10 词摘要
    │
    ▼
Feed.vue 渲染每条 feedItem
    ├─► 头部：作者头像 + 姓名 + sentence（如 "wrote a note" / "deleted a note"） + 时间
    │
    ├─► 【专用组件路径】根据 feedItem.action 渲染对应子组件
    │    └─► v-if="action === 'note_created' || 
    │              action === 'note_updated' || 
    │              action === 'note_deleted'"
    │         └─► <note :data="feedItem.data">
    │              ├─► v-if="data.note.object" → 显示 title + body（30字截断）
    │              └─► v-else → 灰色标签显示 data.note.description（10词摘要）
    │
    └─► 【通用 description fallback】v-if="feedItem.description"
         └─► 灰色边框显示 feedItem.description
         └─► 注：ViewHelper 不返回顶层 description，此分支对 note 类 action 始终为假
```

---

### 5.2 后端：三个 note 相关 action 的 feedItem 状态对比

**后端常量**定义于 [ContactFeedItem.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Models/ContactFeedItem.php#L33-L38)：

```php
ACTION_NOTE_CREATED   = 'note_created';
ACTION_NOTE_UPDATED   = 'note_updated';
ACTION_NOTE_DESTROYED = 'note_destroyed';
```

三种 action 在删除笔记后的数据状态：

| 维度 | note_created（旧记录） | note_updated（旧记录） | note_destroyed（删除时新记录） |
|------|-------------------------|-------------------------|----------------------------------|
| 创建位置 | [CreateNote.php:74-83](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/CreateNote.php#L74-L83) | [UpdateNote.php:74-83](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/UpdateNote.php#L74-L83) | [DestroyNote.php:60-68](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/DestroyNote.php#L60-L68) |
| 关联方式 | `$note->feedItem()->save()` | `$note->feedItem()->save()` | `ContactFeedItem::create()` |
| feedable_id | **取决于是否被更新过**（见 5.4） | 被删 note 的 id | `null` |
| feedable_type | 同上 | `App\Models\Note` | `null` |
| description（DB字段） | 创建时 body 的前 10 词 | 更新时 body 的前 10 词 | 删除时 body 的前 10 词 |
| `$item->feedable` 返回值（笔记删除后） | `null`（DB 行已不存在或关联已断开） | `null`（DB 行已不存在） | `null`（id 和 type 本就是 null） |

**ActionFeedNote::data() 的映射**，代码在 [ActionFeedNote.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedNote.php#L10-L35)：

```php
$note = $item->feedable;   // 笔记被删后始终为 null
return [
    'note' => [
        'object'      => $note ? ['id', 'title', 'body（30字截断）'] : null,  // 始终为 null
        'description' => $item->description,  // DB 中的 10 词摘要，始终有值
    ],
    'contact' => [...],
];
```

所以删除笔记后，三种 action 的 `data.note` 都是：
```js
{ object: null, description: "这是笔记内容的前 10 个词…" }
```

---

### 5.3 前端 Bug：action 名不匹配导致 note_destroyed 专用组件不渲染

**Feed.vue 中的匹配条件**（[Feed.vue:135-142](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Shared/Modules/Feed.vue#L135-L142)）：

```html
<note
  v-if="
    feedItem.action === 'note_created' ||
    feedItem.action === 'note_updated' ||
    feedItem.action === 'note_deleted'
  "
  :data="feedItem.data"
  :contact-view-mode="contactViewMode" />
```

问题在于：
- 前端写的是 `'note_deleted'`
- 后端常量是 `'note_destroyed'`（见 `ContactFeedItem::ACTION_NOTE_DESTROYED`）
- 二者不相等 → `note_destroyed` 这条 feedItem **不会进入 <note> 组件**

同时，[ModuleFeedViewHelper::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L21-L35) 的返回结构中**没有顶层的 description 字段**：

```php
return [
    'id'         => $item->id,
    'action'     => $item->action,
    'author'     => self::getAuthor($item, $vault),
    'sentence'   => self::getSentence($item),
    'data'       => self::getData($item, $user),
    'created_at' => DateHelper::format($item->created_at, $user),
];
```

因此 Feed.vue 第 145-152 行的通用 description fallback：

```html
<div v-if="feedItem.description" class="ms-6">
    <div class="rounded-lg border border-gray-300 px-3 py-2">
        <span class="text-sm">{{ feedItem.description }}</span>
    </div>
</div>
```

对 note 类 action 而言 `feedItem.description` 始终为 `undefined`（ViewHelper 没传），所以这个 fallback 也不起作用。

---

### 5.4 三种 action 删除后的实际渲染效果

假设用户做了以下操作序列：
1. 创建笔记（body = "一二三四五六七八九十十一十二十三十四十五"）
2. 编辑笔记（body = "修改后第一二三四五六七八九十"）
3. 删除笔记

删除后 Feed 中会出现三条相关记录，渲染效果如下：

#### A. note_created（创建笔记那条）

**渲染链路**：
1. Feed.vue：`action === 'note_created'` → ✅ 匹配 → 渲染 `<note>` 组件
2. Note.vue：`data.note.object` → 为 null（笔记已被硬删除）→ 走 else 分支
3. 显示：灰色标签中的 `description`（创建时的前 10 词）

**实际显示**：
```
[头像] John  wrote a note          2025-01-01
  ┌──────────────────────────┐
  │  一二三四五六七八九十…    │  ← 灰色标签，来自 description
  └──────────────────────────┘
```

⚠️ **但有个 MorphOne 陷阱**（见 5.5）：如果笔记被编辑过，note_created 的 feedable 关联会被 MorphOne 自动断开。不过这不影响删除后的显示——因为删除后 feedable 本来就是 null，显示的还是 description。

#### B. note_updated（编辑笔记那条）

**渲染链路**：
1. Feed.vue：`action === 'note_updated'` → ✅ 匹配 → 渲染 `<note>` 组件
2. Note.vue：`data.note.object` → 为 null → 走 else 分支
3. 显示：灰色标签中的 `description`（编辑时的前 10 词）

**实际显示**：
```
[头像] John  edited a note          2025-01-02
  ┌──────────────────────────────┐
  │  修改后第一二三四五六七八…    │  ← 灰色标签
  └──────────────────────────────┘
```

如果笔记被编辑了 N 次，就会有 N 条 note_updated 记录，每条都能看到对应修改时刻的 10 词快照。

#### C. note_destroyed（删除笔记那条）

**渲染链路**：
1. Feed.vue：`action === 'note_destroyed'`，但匹配条件写的是 `'note_deleted'` → ❌ 不匹配 → **不渲染 <note> 组件**
2. Feed.vue 通用 fallback：`feedItem.description` → undefined → ❌ 也不渲染
3. 结果：**只显示 sentence，没有任何笔记内容**

**实际显示**：
```
[头像] John  deleted a note         2025-01-03
  （空，什么内容都没有）
```

虽然后端 `data.note.description` 中明明存了删除时的前 10 词摘要，但因为前端 action 名写错了，这段内容完全不会被渲染出来。

---

### 5.5 MorphOne 陷阱：多次编辑会断开旧记录的 feedable 关联

Note 模型上的 `feedItem()` 关系定义为 **MorphOne**（[Note.php:87-95](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Models/Note.php#L87-L95)）：

```php
public function feedItem(): MorphOne
{
    return $this->morphOne(ContactFeedItem::class, 'feedable');
}
```

MorphOne 是一对一关系。当 `$note->feedItem()->save($newFeedItem)` 被调用时，Laravel 会：
1. 给 `$newFeedItem` 设置 `feedable_id` 和 `feedable_type`
2. 把**之前**关联的那条 feedItem 的 `feedable_id` / `feedable_type` 置为 null

所以操作序列的实际 DB 状态是：

| 步骤 | 操作 | note_created 行 feedable_id | note_updated 行 feedable_id |
|------|------|------------------------------|------------------------------|
| 1 | 创建笔记 | = note.id | — |
| 2 | 编辑笔记 | 被 MorphOne **置为 null** | = note.id |
| 3 | 删除笔记 | 保持 null | 仍为 note.id（但 notes 表行已消失） |

这意味着：
- 即使笔记没被删除，note_created 的 feedItem 通过 `$item->feedable` 也加载不到 Note（关联被 MorphOne 断开了），只能 fallback 到 description
- 最近一次编辑的 note_updated，在笔记未删除时能通过 feedable 加载到完整 Note（显示 title + 30 字 body）
- 笔记删除后，所有记录的 feedable 都返回 null，全部 fallback 到 description

---

### 5.6 总结：删除后到底能看到什么？

| Feed 中的记录 | 能看到内容吗？ | 显示内容来源 | 显示样式 |
|--------------|----------------|--------------|----------|
| note_created（创建） | ✅ 能 | DB description（创建时前 10 词） | 灰色标签 |
| note_updated（每次编辑） | ✅ 能 | DB description（每次编辑时前 10 词） | 灰色标签 |
| note_destroyed（删除操作本身） | ❌ 不能 | — | 只显示 "deleted a note"，无内容 |

**核心问题总结**：

1. **前端 Bug**：[Feed.vue:139](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Shared/Modules/Feed.vue#L139) 条件写的是 `'note_deleted'`，而后端实际使用的是 `'note_destroyed'`，导致删除操作那条记录的 <note> 组件完全不渲染
2. **数据链路正确**：后端 `ActionFeedNote::data()` 对三种 action 都返回了 `{object: null, description: "10词摘要"}`，数据是齐全的
3. **历史记录可恢复性有限**：删除后只能看到每次操作时保存的 10 词摘要快照，完整 body 已随硬删除永久丢失
4. **MorphOne 副作用**：多次编辑会自动断开更早记录的 feedable 关联，导致即使笔记未删除，旧的 note_created / note_updated 也只能 fallback 到 description 而无法加载完整笔记

---

## 六、编辑和删除对 Journal/Feed 的影响

### 6.1 Note 编辑

**服务**：[UpdateNote.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/UpdateNote.php)

**流程**：
1. 验证权限和数据
2. 更新 note 的 body、title、emotion_id
3. 更新 `contact.last_updated_at`
4. **创建新的 ContactFeedItem**（action = `note_updated`，description = body 摘要 10 词）
5. 通过 `$this->note->feedItem()->save($feedItem)` 将 feedItem 与 Note 关联

**影响**：
- Feed 中新增一条 "edited a note" 记录
- Note 原来创建时的 feedItem 的 feedable_id/type 会被 MorphOne 自动置空（关联被断开，见第五节 5.5）
- 新的 feedItem 通过 feedable 关联到 Note，是唯一能通过 `$item->feedable` 加载到完整 Note 的记录

### 6.2 Note 删除

**服务**：[DestroyNote.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/DestroyNote.php)

**流程**：
1. 验证权限
2. **先创建 ContactFeedItem**（action = `note_destroyed`，description = body 摘要 10 词）
   - 注意：此处使用 `ContactFeedItem::create()` 而非 `$this->note->feedItem()->save()`
   - 因此该 feedItem **没有 feedable_id/feedable_type**（因为 Note 马上要被删除）
3. 删除 Note 记录（硬删除）
4. 更新 `contact.last_updated_at`

**影响**：
- Feed 中新增一条 "deleted a note" 记录
- **由于前端 Bug**（action 名 `note_deleted` vs `note_destroyed` 不匹配），该记录的 <note> 组件不渲染，**看不到 description 中的 10 词摘要**，只能看到 "deleted a note" 这句话
- Note 原来创建/更新时关联的 feedItem：其 `$item->feedable` 返回 null（notes 表行已被硬删除），但这些记录的 <note> 组件仍会渲染并显示各自保存的 description（10 词摘要）
- **Note 数据完全丢失，无法恢复**，只能从历史 feedItem 的 10 词摘要中看到片段
- 不影响 Journal 和 Post

### 6.3 Post 编辑

**服务**：[UpdatePost.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Vault/ManageJournals/Services/UpdatePost.php)

**流程**：
1. 验证权限
2. 更新 post 的 title、written_at
3. 更新各 postSection 的 content

**影响**：
- **不创建 ContactFeedItem**，编辑 Post 不会在联系人 Feed 中留下记录
- 前端使用 500ms 防抖自动保存（Auto save）
- Contact 与 Post 的关联（contact_post 中间表）在 [PostController::update()](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Vault/ManageJournals/Web/Controllers/PostController.php#L109-L146) 中处理：先 `detach` 所有联系人，再逐个 `AddContactToPost`

### 6.4 Post 删除

**服务**：[DestroyPost.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Vault/ManageJournals/Services/DestroyPost.php)

**流程**：
1. 验证权限
2. 删除 Post 关联的 Files
3. 删除 Post 记录（级联删除 PostSections 和 contact_post 中间记录）

**影响**：
- **不创建 ContactFeedItem**，删除 Post 不会在联系人 Feed 中留下记录
- Post 之前产生的 feedItem（`added_to_post`/`removed_from_post`）仍保留在 Feed 中，但其 `feedable` 引用变为悬空（Post 被删除后无法加载）
- `contact_post` 中间表记录级联删除 → Contact 的 Posts 模块列表自动更新
- Journal 的 Show 页面自动更新（Post 不再出现）
- 不影响 Note

### 6.5 Contact-Post 关联变更

**添加联系人到 Post**：[AddContactToPost.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Vault/ManageJournals/Services/AddContactToPost.php)
- `syncWithoutDetaching` 添加关联
- 创建 ContactFeedItem（action = `added_to_post`，description = post title）
- feedItem 通过 `$this->post->feedItem()->save($feedItem)` 与 Post 关联
- 更新 `contact.last_updated_at`

**从 Post 移除联系人**：[RemoveContactFromPost.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Vault/ManageJournals/Services/RemoveContactFromPost.php)
- `detach` 移除关联
- 创建 ContactFeedItem（action = `removed_from_post`，description = post title）
- feedItem 通过 `$this->post->feedItem()->save($feedItem)` 与 Post 关联
- 更新 `contact.last_updated_at`

---

## 七、核心差异总结

| 维度 | Note（联系人笔记） | Post（日记条目） |
|------|---------------------|-------------------|
| 所属对象 | Contact | Journal |
| 与 Contact 关系 | 直接外键（1:N） | 多对多（contact_post 中间表） |
| 内容结构 | 单个 body 字段 | 多个 PostSection（label + content） |
| Markdown 渲染 | ❌ 纯文本展示 | ✅ 服务端转 HTML，前端 v-html 渲染 |
| 在 Contact Feed 中 | note_created/updated/destroyed（有专用组件） | added_to_post/removed_from_post（仅显示 title） |
| 编辑对 Feed 影响 | ✅ 创建新 feedItem | ❌ 无影响 |
| 删除对 Feed 影响 | ✅ 创建 feedItem，Note 硬删除 | ❌ 无 feedItem，Post 级联删除 |
| 在 Journal 页面展示 | ❌ 不展示 | ✅ 按年月组织展示 |
| 在 Contact 页面展示 | ✅ Notes 模块 | ✅ Posts 模块（列出关联 Post） |
| 搜索索引 | ✅ Scout 全文搜索（title + body） | ❌ 无搜索索引 |

---

## 八、一对一多态关联（MorphOne）的代码级行为详解

本节顺着代码路径，把笔记动态记录的 `feedItem()` 一对一多态关联到底怎么保存、每次编辑时如何影响旧记录、删除时为何不走关联、以及笔记存在/被删两种状态下的摘要来源与展示结果讲清楚。

---

### 8.1 关联定义的两端

#### 从 Note 角度（多态的 "一" 端）

定义于 [Note.php:87-95](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Models/Note.php#L87-L95)：

```php
public function feedItem(): MorphOne
{
    return $this->morphOne(ContactFeedItem::class, 'feedable');
}
```

含义：一个 Note **有一个** `feedItem`。底层通过 `contact_feed_items` 表的 `feedable_id` + `feedable_type` 两个字段找到对应的行。

#### 从 ContactFeedItem 角度（多态的 "多" 端）

定义于 [ContactFeedItem.php:129-135](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Models/ContactFeedItem.php#L129-L135)：

```php
public function feedable(): MorphTo
{
    return $this->morphTo();
}
```

含义：一个 ContactFeedItem **属于一个** `feedable`（可以是 Note、Post、Address、Label... 任何东西）。读取这个属性时，Laravel 根据 `feedable_type`（如 `App\Models\Note`）确定模型类，再根据 `feedable_id` 去对应表 find。

> **数据库字段设计**（见 [2021_10_19_022411_create_contact_feed_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/database/migrations/2021_10_19_022411_create_contact_feed_table.php)）：
> - `feedable_id` → `numeric`，可空
> - `feedable_type` → `string`，可空
> - 两个字段都**没有外键约束**，可以自由置空而不会产生 DB 级错误

---

### 8.2 `$note->feedItem()->save($feedItem)` 的精确 DB 行为

CreateNote 和 UpdateNote 都用这段代码把 feedItem 与 Note 关联。MorphOne 的 `save()` 方法在 Laravel 内部做了**三件事**：

```
调用：$this->note->feedItem()->save($feedItem)
          │
          ▼
1. 给 $feedItem 设置两个字段值：
     $feedItem->feedable_id   = $this->note->id;
     $feedItem->feedable_type = 'App\Models\Note';
   （注意：fillable 里已经包含了这两个字段，所以允许批量赋值）

2. $feedItem->save();  ← 持久化这两个字段到 DB
          │
          ▼
3. ⚠️  MorphOne 的副作用：把 "之前" 关联的那条 feedItem 记录断开
     UPDATE contact_feed_items
     SET    feedable_id = NULL,
            feedable_type = NULL
     WHERE  feedable_id   = <note.id>
       AND  feedable_type = 'App\Models\Note'
       AND  id           != <new_feedItem.id>
```

**关键点**：步骤 3 把旧记录的 `feedable_id` 和 `feedable_type` 置为 `null`，但**不会删除旧记录**——那条 note_created / 早期 note_updated 的行仍然留在表里，只是无法通过 feedable 加载到 Note 了。

这是 "一对一" 的必然结果：在任意时刻，一个 Note 只能有一条 feedItem 记录通过 feedable_id/type 关联到它。新的来了，旧的必须断开。

---

### 8.3 创建笔记：完整代码追踪

**代码路径**：[CreateNote.php:48-83](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/CreateNote.php#L48-L83)

```
execute($data)
  │
  ├─ 1. validateRules()
  │     ├─ rules(): account_id, vault_id, author_id, contact_id, emotion_id?, title?, body
  │     ├─ permissions(): author + vault + contact + author_must_be_vault_editor
  │     └─ BaseService 会自动注入 $this->author / $this->vault / $this->contact
  │
  ├─ 2. Note::create([...])                     ← notes 表新增一行（note.id = N）
  │     contact_id, vault_id, author_id, title, body, emotion_id
  │
  ├─ 3. touch contact.last_updated_at
  │
  └─ 4. createFeedItem()
        │
        ├─ ContactFeedItem::create([            ← contact_feed_items 新增一行（id=F1）
        │    author_id    => author.id,
        │    contact_id   => contact.id,
        │    action       => 'note_created',
        │    description  => Str::words($this->note->body, 10, '…'),
        │    // ⚠️  这里没有 feedable_id/feedable_type！所以此时为 NULL
        │  ]);
        │
        └─ $this->note->feedItem()->save($feedItem);
              │
              ├─ 设置 $feedItem->feedable_id   = N
              ├─ 设置 $feedItem->feedable_type = 'App\Models\Note'
              ├─ $feedItem->save();             ← UPDATE F1，把两个关联字段写入
              └─ （没有旧记录，所以 MorphOne 副作用不触发）
```

**操作完成后的 DB 快照**：

| 表 | id | 关键字段 |
|----|----|----------|
| notes | N | body = 正文原文 |
| contact_feed_items | F1 | action='note_created'<br>description=正文前10词…<br>feedable_id=N, feedable_type='App\Models\Note' |

**此时的摘要来源**：
- `description`（DB 字段）= `Str::words($this->note->body, 10, '…')` → **创建时刻 body 的前 10 词快照**
- 快照时机：`Note::create()` 之后、`createFeedItem()` 调用之时，`$this->note->body` 就是刚写入 DB 的值

---

### 8.4 编辑笔记：完整代码追踪 + MorphOne 副作用

**代码路径**：[UpdateNote.php:49-83](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/UpdateNote.php#L49-L83)

```
execute($data)
  │
  ├─ 1. validateRules() — 同上，额外 require note_id
  │
  ├─ 2. $this->note = $contact->notes()->findOrFail(note_id)
  │     （note.id 仍然是 N，body 还是旧值）
  │
  ├─ 3. $this->note->body  = $data['body'];    ← 内存中的 body 已变为新值
  │    $this->note->title = ...
  │    $this->note->save();                    ← UPDATE notes N，body 变为新正文
  │
  ├─ 4. touch contact.last_updated_at
  │
  └─ 5. createFeedItem()
        │
        ├─ ContactFeedItem::create([           ← 新增一行 F2
        │    action       => 'note_updated',
        │    description  => Str::words($this->note->body, 10, '…'),
        │    // ⚠️  此时 $this->note->body 是"新正文"（已 save 到 DB 的值）
        │  ]);
        │
        └─ $this->note->feedItem()->save($feedItem);   ← 关键！MorphOne 副作用触发
              │
              ├─ 设置 F2.feedable_id=N, F2.feedable_type='App\Models\Note'
              ├─ F2->save();
              │
              └─ ⚠️  MorphOne 自动执行：
                   UPDATE contact_feed_items
                   SET    feedable_id = NULL,
                          feedable_type = NULL
                   WHERE  feedable_id   = N
                     AND  feedable_type = 'App\Models\Note'
                     AND  id           != F2;
                   └─ 命中了 F1（note_created）的行！F1 的关联字段被置空
```

**操作完成后的 DB 快照**（与创建后对比）：

| 表 | id | 关键字段变化 |
|----|----|--------------|
| notes | N | body = 新正文（UPDATE 覆盖） |
| contact_feed_items | F1（note_created） | feedable_id=**NULL**, feedable_type=**NULL**<br>description 仍然是**旧正文**前10词（不变） |
| contact_feed_items | F2（note_updated） | feedable_id=N, feedable_type='App\Models\Note'<br>description=新正文前10词 |

**此时的摘要来源**：
- F1.description = 创建时刻 body 的前 10 词（历史快照，**永远不再变化**）
- F2.description = 编辑时刻 body 的前 10 词（也是历史快照，下次编辑也不再变化）
- 注意：编辑时即使标题改了，两个 description 中的**词级截断快照**也不会互相影响，它们各自独立

> **如果再编辑一次**（第二次编辑产生 F3）：F3 获得 feedable 关联，F2 的关联字段被置空。F1 / F2 的 description 都是各自时刻的快照，永远不变。

---

### 8.5 删除笔记：完整代码追踪 + 为何不走 MorphOne

**代码路径**：[DestroyNote.php:46-68](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/DestroyNote.php#L46-L68)

```
execute($data)
  │
  ├─ 1. validateRules() — require note_id，确认权限
  │
  ├─ 2. $this->note = $contact->notes()->findOrFail(note_id)
  │     （此时 note.body 仍然可读，是最后一次编辑保存的值）
  │
  ├─ 3. createFeedItem()                    ← 先创建 feedItem！再删 note！
  │     │
  │     └─ ContactFeedItem::create([        ← 新增一行 Fd
  │          author_id    => author.id,
  │          contact_id   => contact.id,
  │          action       => 'note_destroyed',
  │          description  => Str::words($this->note->body, 10, '…'),
  │          // ⚠️  这里故意不传 feedable_id/type
  │          //     也故意不调用 $note->feedItem()->save()
  │          //     因为 note 马上就要被 DELETE 了，关联也没意义
  │        ]);
  │
  ├─ 4. $this->note->delete();              ← DELETE FROM notes WHERE id=N
  │     （硬删除，行消失。notes 表已查不到 N）
  │
  └─ 5. touch contact.last_updated_at
```

**为何 DeleteNote 故意不走 MorphOne？**

两种可能性分析：
1. 如果调用 `$this->note->feedItem()->save($feedItem)`：Fd 会被设置 `feedable_id=N, feedable_type='App\Models\Note'`，同时把 F2（最近一次编辑的 feedItem）关联置空。然后 `$this->note->delete()` 也不会对 contact_feed_items 产生级联（因为没有外键约束），所以 Fd 的 `feedable_id=N, feedable_type='Note'` 会残留在表里，变成一条指向不存在记录的"死引用"。
2. 直接 `ContactFeedItem::create()` 不设置关联字段：Fd 的 `feedable_id=NULL, feedable_type=NULL`，干净利落，指向性明确——这就是一条"目标已被删除"的记录。

代码选择了方案 2。

**操作完成后的 DB 快照**：

| 表 | id | 关键字段 |
|----|----|----------|
| notes | N | **行已消失**（DELETE） |
| contact_feed_items | F1（note_created） | feedable_id=NULL, feedable_type=NULL<br>description=创建时刻前10词 |
| contact_feed_items | F2（note_updated） | feedable_id=N（残留）, feedable_type='Note'（残留）<br>description=编辑时刻前10词 |
| contact_feed_items | Fd（note_destroyed） | feedable_id=**NULL**, feedable_type=**NULL**<br>description=删除时刻前10词 |

注意 F2 的 `feedable_id=N` 因为没有外键约束而残留，这就是之前说的"悬空引用"。

---

### 8.6 摘要来源的精确对照

每次 `Str::words($this->note->body, 10, '…')` 都是从**当时内存中**的 `$this->note->body` 取前 10 个词。调用时机不同，快照值也不同：

| 操作 | Service 中调用位置 | `$this->note->body` 来源 | 快照保存到 |
|------|-------------------|-------------------------|-----------|
| 创建 | [CreateNote:80](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/CreateNote.php#L80) — `Note::create()` 之后 | `Note::create()` 刚写入 DB 的新 body | F1.description |
| 编辑 | [UpdateNote:80](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/UpdateNote.php#L80) — `$note->save()` 之后 | `$note->save()` 刚 UPDATE 的新 body | F2.description |
| 再编辑 | 同上，每次编辑都取最新 body | 最新提交的 body | F3、F4... 各自的 description |
| 删除 | [DestroyNote:66](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/DestroyNote.php#L66) — `findOrFail()` 之后、`delete()` 之前 | `findOrFail()` 从 DB 读到的最后保存的 body | Fd.description |

结论：每条 feedItem 的 description 都是**操作发生瞬间** body 的前 10 词独立快照，之后永远不变。不会因为后续编辑或删除而更新。

---

### 8.7 展示结果矩阵：笔记存在 vs 被删后

**前提**：假设用户依次做了 创建 → 编辑 → 再编辑 → 删除，共产生 4 条 feedItem 记录：
- F1 `note_created`，body="今天和张三讨论了A项目的需求和下一步计划"（15词）→ 10词摘要="今天和张三讨论了A项目的需求和下一步…"
- F2 `note_updated`，body="讨论之后确定了A项目的详细规格文档的编写安排" → 10词摘要="讨论之后确定了A项目的详细规格文档的编写…"
- F3 `note_updated`，body="最终决定A项目Q3启动，相关资源已协调完毕" → 10词摘要="最终决定A项目Q3启动，相关资源已协调…"
- Fd `note_destroyed`，body 仍是 F3 的内容（最后保存的）→ 10词摘要="最终决定A项目Q3启动，相关资源已协调…"

> 注：以下所有 "ViewHelper 输出" 指 [ActionFeedNote::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedNote.php#L10-L35) 的映射结果。

---

#### 场景 A：笔记仍然存在（第三次编辑后，尚未删除）

此时 notes 表中 N 行仍然存在。各 feedItem 状态：

| 记录 | feedable_id / type | ViewHelper 中 `$note = $item->feedable` | ViewHelper 输出 data.note | 前端渲染 |
|------|-------------------|----------------------------------------|--------------------------|---------|
| **F1 note_created** | id=NULL, type=NULL（被 MorphOne 在 F2 时断开） | `null`（找不到关联，就算 Note 存在也没用） | `{object: null, description: "今天和张三讨论了A项目的需求和下一步…"}` | Feed.vue action 匹配 ✅ → `<note>` 渲染 → `v-else` 分支 → **灰色标签显示 10 词摘要** |
| **F2 note_updated** | id=NULL, type=NULL（被 MorphOne 在 F3 时断开） | `null` | `{object: null, description: "讨论之后确定了A项目的详细规格文档的编写…"}` | Feed.vue action 匹配 ✅ → `<note>` 渲染 → `v-else` 分支 → **灰色标签显示 10 词摘要** |
| **F3 note_updated** | id=N, type='Note'（当前持有 MorphOne 关联） | `Note N`（通过 find(N) 找到） | `{object: {id, title, body（Str::limit 30字）}, description: "最终决定A项目Q3启动…"}` | Feed.vue action 匹配 ✅ → `<note>` 渲染 → `v-if` 分支 → **显示 title + body 30 字截断**（不是 10 词摘要！） |
| **Fd** | 不存在（还没删） | — | — | — |

**展示效果总结（笔记存在时）**：

```
F3 最近一次编辑（持有关联）：
  John  edited a note          2025-01-03
  ┌─────────────────────────────────────┐
  │ （如果有 title 先显示 title）         │
  │ 最终决定A项目Q3启动，相关资源已协调…  │  ← Str::limit 30字符，来自 Note 实时 body
  └─────────────────────────────────────┘

F1 / F2 早期记录（关联已断开）：
  John  wrote a note            2025-01-01
  ┌──────────────────────────────────────┐
  │ [今天和张三讨论了A项目的需求和下一步…] │  ← 灰色标签，Str::words 10词快照
  └──────────────────────────────────────┘
```

**一个值得注意的细节**：即使笔记存在，**除了最近一次编辑之外**的所有历史记录也只能看到 10 词摘要，因为 MorphOne 的副作用把它们的 feedable 关联断开了。这意味着 MorphOne 在这里的语义其实是"当前最新版本"指针，而不是"每条记录都能回溯完整内容"。

---

#### 场景 B：笔记已被删除（执行了 DeleteNote）

此时 notes 表中 N 行已被 DELETE。各 feedItem 状态：

| 记录 | feedable_id / type | ViewHelper 中 `$note = $item->feedable` | ViewHelper 输出 data.note | 前端渲染 |
|------|-------------------|----------------------------------------|--------------------------|---------|
| **F1 note_created** | id=NULL, type=NULL | `null` | `{object: null, description: "今天和张三讨论了A项目的需求和下一步…"}` | Feed.vue action 匹配 ✅ → `<note>` 渲染 → `v-else` 分支 → **灰色标签显示 10 词摘要**（与笔记存在时相同，因为 description 是 DB 快照，不受 Note 删除影响） |
| **F2 note_updated** | id=NULL, type=NULL | `null` | `{object: null, description: "讨论之后确定了A项目的详细规格文档的编写…"}` | 同上 → **灰色标签显示 10 词摘要** |
| **F3 note_updated** | id=N**（残留值）**, type='Note'**（残留）** | `null`（因为 find(N) 找不到 Note 行，即使有 id 和 type 也没用） | `{object: null, description: "最终决定A项目Q3启动…"}` | Feed.vue action 匹配 ✅ → `<note>` 渲染 → `v-else` 分支 → **灰色标签显示 10 词摘要**（⚠️ 笔记存在时这里显示的是 30 字 body，删除后退化为 10 词摘要） |
| **Fd note_destroyed** | id=NULL, type=NULL（故意不传） | `null` | `{object: null, description: "最终决定A项目Q3启动…"}` | **Feed.vue action 匹配 ❌**（条件写的是 `'note_deleted'`，实际 action 是 `'note_destroyed'`）→ **不渲染 `<note>` 组件**，也不走顶层 description fallback（ViewHelper 不返回顶层 description 字段）→ **只显示 sentence，什么内容都没有** |

**展示效果总结（笔记被删后）**：

```
F1 创建记录：
  John  wrote a note            2025-01-01
  ┌──────────────────────────────────────┐
  │ [今天和张三讨论了A项目的需求和下一步…] │  ← 灰色标签，10词摘要（DB 快照）
  └──────────────────────────────────────┘

F2 第一次编辑记录：
  John  edited a note           2025-01-02
  ┌─────────────────────────────────────────┐
  │ [讨论之后确定了A项目的详细规格文档的编写…] │  ← 灰色标签，10词摘要
  └─────────────────────────────────────────┘

F3 第二次（最近一次）编辑记录：
  John  edited a note           2025-01-03
  ┌──────────────────────────────────────┐
  │ [最终决定A项目Q3启动，相关资源已协调…] │  ← ⚠️ 退化为灰色标签，10词摘要
  └──────────────────────────────────────┘
    （笔记存在时这里会显示更长的 30 字 body，删除后变短了）

Fd 删除记录（前端 Bug）：
  John  deleted a note          2025-01-04
    （空。。。什么内容预览都没有）
    （虽然 data.note.description 有值，但组件根本没渲染）
```

---

### 8.8 结论与代码缺陷汇总

| 问题 | 具体表现 | 位置 |
|------|---------|------|
| **前端 action 名 Bug** | `note_destroyed` 不匹配 `'note_deleted'`，删除操作的 <note> 组件完全不渲染 | [Feed.vue:139](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Shared/Modules/Feed.vue#L139) |
| **MorphOne 历史记录断链** | 每次编辑都会把更早记录的 feedable_id/type 置空，导致即使笔记存在也只能从最近一次加载完整 body，历史记录全部 fallback 到 10 词摘要 | [UpdateNote:82](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/UpdateNote.php#L82) — `$note->feedItem()->save()` |
| **MorphOne 设计语义不符** | feedItem() 是一对一（MorphOne），但 ContactFeedItem 是**操作日志**应当是一对多（MorphMany）。一对多的设计下每条历史操作都能通过 feedable 回溯到完整 Note（只要 Note 还在） | [Note.php:92-95](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Models/Note.php#L92-L95) — `morphOne` 应改为 `morphMany` |
| **顶层 description fallback 不起作用** | Feed.vue 有一个 `<div v-if="feedItem.description">` 兜底分支，但 ModuleFeedViewHelper 返回结构中没有顶层 description，对 note / post 类 action 永远不触发 | [ModuleFeedViewHelper.php:21-35](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L21-L35) |
| **删除后完整内容不可恢复** | Note 硬删除后，body 的完整内容永久丢失，只能从各 feedItem 的 10 词摘要中看到碎片化信息 | [DestroyNote:54](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/DestroyNote.php#L54) — `$this->note->delete()` |

