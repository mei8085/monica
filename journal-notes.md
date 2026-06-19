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
| feedable_id | 被删 note 的 id | 被删 note 的 id | `null` |
| feedable_type | `App\Models\Note` | `App\Models\Note` | `null` |
| description（DB字段） | 创建时 body 的前 10 词 | 更新时 body 的前 10 词 | 删除时 body 的前 10 词 |
| `$item->feedable` 返回值（笔记删除后） | `null`（notes 表行已硬删除，有 id 和 type 也找不到对应行） | `null`（同上） | `null`（id 和 type 本就是 null） |

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

> 💡 **笔记存在时**，这条记录的 `object` 不为 null（因为 feedable_id=N 有效，能加载到 Note），会显示 title + body 30 字截断。删除后才退化为 10 词摘要。

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

如果笔记被编辑了 N 次，就会有 N 条 note_updated 记录，每条都能看到对应修改时刻的 10 词快照（删除后）或 30 字 body（笔记存在时）。

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

### 5.5 MorphOne 的真实行为：查询时 first()，保存时不强制一对一

**之前的理解是错误的**：MorphOne `save()` 不会把旧记录的 feedable_id/type 置空。下面是基于 Laravel 12 源码的纠正。

#### Note 模型的关系定义

[Note.php:87-95](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Models/Note.php#L87-L95)：

```php
public function feedItem(): MorphOne
{
    return $this->morphOne(ContactFeedItem::class, 'feedable');
}
```

#### 继承链与 save() 方法的实际源码

```
MorphOne
  extends MorphOneOrMany   ← 没有重写 save()
    extends HasOneOrMany   ← save() 源码如下
```

**Laravel 12 `HasOneOrMany::save()` 真实源码**（已从官方仓库确认）：

```php
public function save(Model $model)
{
    $this->setForeignAttributesForCreate($model);  // 只设置外键
    return $model->save() ? $model : false;         // 只调用 save()
}
```

**Laravel 12 `MorphOneOrMany::setForeignAttributesForCreate()` 真实源码**：

```php
protected function setForeignAttributesForCreate(Model $model)
{
    $model->{$this->getForeignKeyName()} = $this->getParentKey();
    $model->{$this->getMorphType()} = $this->morphClass;
    // ... pendingAttributes 处理
}
```

**结论**：`$note->feedItem()->save($feedItem)` 的完整 DB 操作就是：

```
1. 设置 $feedItem->feedable_id   = note.id
2. 设置 $feedItem->feedable_type = 'App\Models\Note'
3. $feedItem->save()  ← UPDATE 或 INSERT
```

**没有任何 UPDATE 旧记录的 SQL！不会把任何旧记录的 feedable_id/type 置空！**

#### MorphOne 语义到底是什么：查询时的 first()，不是保存时的强制唯一

MorphOne 的 "One" 体现在**查询端**，不是**保存端**：

- **查询** `$note->feedItem` 时，`MorphOne::getResults()` 调用 `$this->query->first()`，所以永远只返回一条（取决于 DB 默认排序，通常是 id ASC 最早的那条）
- **保存** `$note->feedItem()->save()` 时，只是给新记录设置关联字段然后 save，**不会清理或修改旧记录**

#### 实际 DB 状态：多条记录共享相同的 feedable_id/type

以操作序列「创建 → 编辑 → 再编辑 → 删除」为例：

| 步骤 | 操作 | F1 note_created | F2 note_updated① | F3 note_updated② | Fd note_destroyed |
|------|------|-----------------|------------------|------------------|-------------------|
| 1 | 创建笔记 | id=N, type=Note | — | — | — |
| 2 | 第一次编辑 | id=N, type=Note | id=N, type=Note | — | — |
| 3 | 第二次编辑 | id=N, type=Note | id=N, type=Note | id=N, type=Note | — |
| 4 | 删除笔记 | **id=N, type=Note（残留）** | **id=N, type=Note（残留）** | **id=N, type=Note（残留）** | id=NULL, type=NULL |

**步骤 2 和 3 结束时，F1、F2、F3 的 feedable_id 都是 N，feedable_type 都是 Note！**三条记录共享同一个关联目标。

#### MorphOne 查询的歧义：`$note->feedItem` 返回哪一条？

因为三条记录都有 feedable_id=N、feedable_type=Note，而 `MorphOne::getResults()` 用的是 `$this->query->first()`，在没有显式 `orderBy` 的情况下，返回结果取决于：
- Query Builder 的默认排序（通常是 DB 原生顺序）
- MySQL InnoDB 默认是主键 id 升序 → 返回 **F1 最早的 note_created**
- PostgreSQL 默认也是插入顺序 → 返回最早的一条

但 **Note 模型和项目代码中从未使用过 `$note->feedItem` 这个动态属性**，只用它的 `save()` 方法建立关联。实际从 ContactFeedItem → Note 方向加载是走 `$feedItem->feedable`（MorphTo），它**精确按 id 查找**不存在歧义——只要有有效的 feedable_id 和 feedable_type，就一定能 find 到对应的 Note（如果 Note 还存在的话）。

> **对本项目的影响**：MorphOne 保存时不强制唯一这个事实，意味着笔记存在时，**所有** note_created / note_updated 记录都能通过 `$item->feedable` 成功加载到完整 Note，而不是只有最近一条。

---

### 5.6 总结：删除后到底能看到什么？

| Feed 中的记录 | 笔记存在时显示 | 笔记删除后显示 |
|--------------|---------------|----------------|
| note_created（创建） | ✅ title + body 30 字（通过 feedable 加载完整 Note） | ✅ 10 词摘要（description 快照） |
| note_updated（每次编辑） | ✅ title + body 30 字（**所有记录都能加载到完整 Note**） | ✅ 10 词摘要（每条各自的快照） |
| note_destroyed（删除操作本身） | — | ❌ **空**（前端 Bug，action 名不匹配） |

**核心问题总结**：

1. **前端 Bug**：[Feed.vue:139](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Shared/Modules/Feed.vue#L139) 条件写的是 `'note_deleted'`，而后端实际使用的是 `'note_destroyed'`，导致删除操作那条记录的 <note> 组件完全不渲染
2. **数据链路正确**：后端 `ActionFeedNote::data()` 对三种 action 都返回了正确的结构，笔记存在时 object 有值，删除后 object 为 null 但 description 有值
3. **历史记录可恢复性有限**：删除后只能看到每次操作时保存的 10 词摘要快照，完整 body 已随硬删除永久丢失
4. **MorphOne 语义澄清**：MorphOne 的 save() 不会断开旧关联，所有历史记录都持有有效的 feedable_id/type，**笔记存在时每条操作记录都能加载到完整 Note**（而不是只有最近一条）

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
- **MorphOne save() 不会断开旧记录**：note_created 和之前所有 note_updated 的 feedable_id/type 仍然有效，都指向同一个 Note（值都为 N / 'App\Models\Note'）
- 新旧 feedItem 都能通过 `$item->feedable` 加载到完整 Note（笔记存在时），所有操作记录都能显示 title + body 30 字截断

### 6.2 Note 删除

**服务**：[DestroyNote.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/DestroyNote.php)

**流程**：
1. 验证权限
2. **先创建 ContactFeedItem**（action = `note_destroyed`，description = body 摘要 10 词）
   - 注意：此处使用 `ContactFeedItem::create()` 而非 `$this->note->feedItem()->save()`
   - 因此该 feedItem **没有 feedable_id/feedable_type**（因为 Note 马上要被删除，关联也没有意义）
3. 删除 Note 记录（硬删除）
4. 更新 `contact.last_updated_at`

**影响**：
- Feed 中新增一条 "deleted a note" 记录
- **由于前端 Bug**（action 名 `note_deleted` vs `note_destroyed` 不匹配），该记录的 <note> 组件不渲染，**看不到 description 中的 10 词摘要**，只能看到 "deleted a note" 这句话
- Note 原来创建/更新时关联的 feedItem：feedable_id 仍然是 N，feedable_type 仍然是 'App\Models\Note'，但 `$item->feedable` 返回 null（因为 notes 表行已被硬删除，find(N) 找不到）
- 因此这些旧记录的 <note> 组件走 else 分支，显示各自保存的 description（10 词摘要）
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

CreateNote 和 UpdateNote 都用这段代码把 feedItem 与 Note 关联。根据 Laravel 12 的实际源码，这个方法做了**两步**，而不是之前错误说的三步：

```
调用：$this->note->feedItem()->save($feedItem)
          │
          ▼
继承链：MorphOne → MorphOneOrMany → HasOneOrMany

HasOneOrMany::save() 源码（Laravel 12 官方仓库确认）：
public function save(Model $model)
{
    $this->setForeignAttributesForCreate($model);  // ↓ 进入此方法
    return $model->save() ? $model : false;
}

MorphOneOrMany::setForeignAttributesForCreate() 源码：
protected function setForeignAttributesForCreate(Model $model)
{
    $model->{$this->getForeignKeyName()} = $this->getParentKey();
    // 即 $feedItem->feedable_id = note.id

    $model->{$this->getMorphType()} = $this->morphClass;
    // 即 $feedItem->feedable_type = 'App\Models\Note'

    // ... pendingAttributes 处理
}
          │
          ▼
实际执行的 SQL：
1. 设置内存中 $feedItem 的两个属性值（无 SQL）
2. $feedItem->save()
   ├─ 因为之前已经 ContactFeedItem::create() 过，$feedItem 存在主键
   └─ 执行 UPDATE contact_feed_items
          SET feedable_id   = N,
              feedable_type = 'App\Models\Note'
        WHERE id = <new_feedItem_id>
```

**关键结论**：
- **只做了"设置新记录关联字段 + 保存"两件事**
- **没有任何 UPDATE 旧记录的 SQL**，不会把旧记录的 feedable_id/type 置空
- MorphOne 的 "One" 语义仅体现在**查询端**（`$note->feedItem` 返回 `first()`），**不体现在保存端**

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
              └─ （MorphOne 不会修改旧记录，这是 HasOneOrMany::save() 源码的既定行为，并非"副作用没触发"）
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

### 8.4 编辑笔记：完整代码追踪

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
        │    // 此时 $this->note->body 是新正文
        │  ]);
        │
        └─ $this->note->feedItem()->save($feedItem);
              │
              ├─ 设置 F2.feedable_id   = N
              ├─ 设置 F2.feedable_type = 'App\Models\Note'
              └─ F2->save();                      ← UPDATE F2
              │
              └─ ✅ 没有任何副作用！F1（note_created）保持不变
                   F1.feedable_id   仍然是 N
                   F1.feedable_type 仍然是 'App\Models\Note'
```

**操作完成后的 DB 快照（与创建后对比）**：

| 表 | id | 关键字段变化 |
|----|----|--------------|
| notes | N | body = 新正文（UPDATE 覆盖） |
| contact_feed_items | F1（note_created） | **feedable_id=N，不变**<br>**feedable_type='App\Models\Note'，不变**<br>description 仍然是旧正文前10词 |
| contact_feed_items | F2（note_updated） | feedable_id=N<br>feedable_type='App\Models\Note'<br>description=新正文前10词 |

**此时的摘要来源**：
- F1.description = 创建时刻 body 的前 10 词（历史快照，永远不变）
- F2.description = 编辑时刻 body 的前 10 词（也是历史快照，下次编辑也不变）
- **F1 和 F2 都可以通过 `$item->feedable` 加载到同一个 Note N**（因为 feedable_id/type 都有效）

> **如果再编辑一次**（第二次编辑产生 F3）：F3 也被设置 feedable_id=N / Note，F1、F2 仍然保持不变。三条记录都指向同一个 Note N。三条记录都能加载到完整 Note。

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
  │          // 故意不传 feedable_id/type
  │          // 也故意不调用 $note->feedItem()->save()
  │          // 因为 note 马上就要被 DELETE 了，关联也没意义
  │        ]);
  │
  ├─ 4. $this->note->delete();              ← DELETE FROM notes WHERE id=N
  │     （硬删除，行消失。notes 表已查不到 N）
  │
  └─ 5. touch contact.last_updated_at
```

**为何 DeleteNote 故意不走 MorphOne？**

如果走关联 `$this->note->feedItem()->save($feedItem)`，Fd 会被设置 `feedable_id=N, feedable_type='App\Models\Note'`。然后 `$this->note->delete()` 硬删除 Note N，由于 contact_feed_items 没有外键约束，Fd 的 `feedable_id=N` 会残留，变成一条指向不存在记录的"悬空死引用"。

直接 `ContactFeedItem::create()` 不设置关联字段，Fd 的 `feedable_id=NULL, feedable_type=NULL`，干净利落——语义上明确表示"目标已被删除，不再有有效关联"。

**操作完成后的 DB 快照**：

| 表 | id | 关键字段 |
|----|----|----------|
| notes | N | **行已消失**（DELETE） |
| contact_feed_items | F1（note_created） | **feedable_id=N（残留），feedable_type='App\Models\Note'（残留）**<br>description=创建时刻前10词 |
| contact_feed_items | F2（note_updated①） | **feedable_id=N（残留），feedable_type='App\Models\Note'（残留）**<br>description=编辑时刻前10词 |
| contact_feed_items | F3（note_updated②） | **feedable_id=N（残留），feedable_type='App\Models\Note'（残留）**<br>description=再编辑时刻前10词 |
| contact_feed_items | Fd（note_destroyed） | **feedable_id=NULL，feedable_type=NULL**<br>description=删除时刻前10词 |

所有 F1/F2/F3 的 feedable_id/type 都保留了原值（因为 MorphOne save 不会修改旧记录），但指向的 Note N 已被硬删除，所以 `$item->feedable` 都会返回 null（find(N) 找不到）。

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
| **F1 note_created** | **id=N, type='Note'**（有效） | `Note N`（通过 `Note::find(N)` 找到） | `{object: {id, title, body（Str::limit 30字）}, description: "今天和张三…"}` | Feed.vue action 匹配 ✅ → `<note>` 渲染 → `v-if` 分支 → **显示 title + body 30 字截断** |
| **F2 note_updated** | **id=N, type='Note'**（有效） | `Note N`（同上，同一个 Note） | `{object: {id, title, body（Str::limit 30字）}, description: "讨论之后确定…"}` | Feed.vue action 匹配 ✅ → `<note>` 渲染 → `v-if` 分支 → **显示 title + body 30 字截断** |
| **F3 note_updated** | **id=N, type='Note'**（有效） | `Note N`（同上） | `{object: {id, title, body（Str::limit 30字）}, description: "最终决定A项目…"}` | Feed.vue action 匹配 ✅ → `<note>` 渲染 → `v-if` 分支 → **显示 title + body 30 字截断** |
| **Fd** | 不存在（还没删） | — | — | — |

**⚠️ 关键事实纠正**：与之前错误的分析不同，**笔记存在时，F1/F2/F3 三条记录都能加载到完整 Note**，都显示最新 Note 的 title + body 30 字。这意味着：

- 所有历史操作记录在笔记存在时，都展示的是**最新的** body，而不是操作时刻的 body
- 因为所有记录的 feedable_id 都是 N，都指向同一个最新的 Note
- 如果想看到操作时刻的 body 快照，只能看每条记录各自独立保存的 `description` 10词摘要字段（但 v-if 分支不展示 description，只展示 object.body）

**展示效果总结（笔记存在时）**：

```
F3 第二次编辑：
  John  edited a note          2025-01-03
  ┌─────────────────────────────────────┐
  │ （如果有 title 先显示 title）         │
  │ 最终决定A项目Q3启动，相关资源已协调…  │  ← Str::limit 30字符，来自 Note 最新 body
  └─────────────────────────────────────┘

F2 第一次编辑：
  John  edited a note           2025-01-02
  ┌─────────────────────────────────────┐
  │ （如果有 title 先显示 title）         │
  │ 最终决定A项目Q3启动，相关资源已协调…  │  ← 也是最新 body（不是历史快照！）
  └─────────────────────────────────────┘

F1 创建：
  John  wrote a note            2025-01-01
  ┌─────────────────────────────────────┐
  │ （如果有 title 先显示 title）         │
  │ 最终决定A项目Q3启动，相关资源已协调…  │  ← 也是最新 body（不是创建时的原始 body！）
  └─────────────────────────────────────┘
```

**设计含义**：MorphOne 不会断开旧关联，所以所有操作记录都指向最新的 Note 对象。历史操作记录在 Note 存在时展示的是最新状态，而不是操作发生时的状态。只有 Note 被删除后才会 fallback 到各自保存的 10词摘要快照。

---

#### 场景 B：笔记已被删除（执行了 DeleteNote）

此时 notes 表中 N 行已被 DELETE。各 feedItem 状态：

| 记录 | feedable_id / type | ViewHelper 中 `$note = $item->feedable` | ViewHelper 输出 data.note | 前端渲染 |
|------|-------------------|----------------------------------------|--------------------------|---------|
| **F1 note_created** | id=N**（残留）**, type='Note'**（残留）** | `null`（Note::find(N) 找不到，行已删除） | `{object: null, description: "今天和张三讨论了A项目的需求和下一步…"}` | Feed.vue action 匹配 ✅ → `<note>` 渲染 → `v-else` 分支 → **灰色标签显示 10 词摘要（创建时刻的快照）** |
| **F2 note_updated** | id=N（残留）, type='Note'（残留） | `null` | `{object: null, description: "讨论之后确定了A项目的详细规格文档的编写…"}` | Feed.vue action 匹配 ✅ → `<note>` 渲染 → `v-else` 分支 → **灰色标签显示 10 词摘要（第一次编辑时刻的快照）** |
| **F3 note_updated** | id=N（残留）, type='Note'（残留） | `null` | `{object: null, description: "最终决定A项目Q3启动…"}` | Feed.vue action 匹配 ✅ → `<note>` 渲染 → `v-else` 分支 → **灰色标签显示 10 词摘要（第二次编辑时刻的快照）** |
| **Fd note_destroyed** | id=NULL, type=NULL（故意不传） | `null`（id 和 type 本就是 null） | `{object: null, description: "最终决定A项目Q3启动…"}` | **Feed.vue action 匹配 ❌**（条件写的是 `'note_deleted'`，实际 action 是 `'note_destroyed'`）→ **不渲染 `<note>` 组件**，也不走顶层 description fallback → **只显示 sentence，什么内容都没有** |

**展示效果总结（笔记被删后）**：

```
F1 创建记录：
  John  wrote a note            2025-01-01
  ┌──────────────────────────────────────┐
  │ [今天和张三讨论了A项目的需求和下一步…] │  ← 灰色标签，10词摘要（创建时刻的快照）
  └──────────────────────────────────────┘
    笔记存在时这里展示的是最新 body（可能和这完全不同），删除后才还原为历史快照

F2 第一次编辑记录：
  John  edited a note           2025-01-02
  ┌─────────────────────────────────────────┐
  │ [讨论之后确定了A项目的详细规格文档的编写…] │  ← 灰色标签，10词摘要（第一次编辑时刻快照）
  └─────────────────────────────────────────┘

F3 第二次（最近一次）编辑记录：
  John  edited a note           2025-01-03
  ┌──────────────────────────────────────┐
  │ [最终决定A项目Q3启动，相关资源已协调…] │  ← 灰色标签，10词摘要（第二次编辑时刻快照）
  └──────────────────────────────────────┘
    笔记存在时显示的是 title + 30字 body，删除后退化为灰色标签的 10 词摘要

Fd 删除记录（前端 Bug）：
  John  deleted a note          2025-01-04
    （空。。。什么内容预览都没有）
    （虽然 data.note.description 有值，但组件根本没渲染）
```

**重要对比（笔记存在 vs 删除后）**：

- **存在时**：所有 note_created / note_updated 都展示**同一个最新 Note** 的 body（30 字截断），历史操作的上下文差异仅体现在 sentence 动词不同（wrote vs edited）和时间戳不同
- **删除后**：每条记录才分别展示各自操作时刻的 10 词摘要快照，能看到每次修改的内容演进轨迹
- 某种程度上，只有删除后才能看到"真正的历史轨迹"；笔记存在时所有记录都只反映最新状态

---

### 8.8 结论与代码缺陷汇总

| 问题 | 具体表现 | 位置 |
|------|---------|------|
| **前端 action 名 Bug** | `note_destroyed` 不匹配 `'note_deleted'`，删除操作的 <note> 组件完全不渲染 | [Feed.vue:139](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Shared/Modules/Feed.vue#L139) |
| **MorphOne 语义澄清** | 之前误以为 MorphOne save() 会断开旧关联。**实际不会**：所有 note_created / note_updated 的 feedable_id/type 都保持有效，全部指向同一个 Note。只有查询端 `$note->feedItem` 用 `first()` 只返回一条 | — |
| **历史记录展示"最新值"而非"操作时刻快照"** | 笔记存在时，所有操作记录都加载同一个最新 Note 对象并展示它的 body（30字截断），只有删除后才 fallback 到各自的 10词历史摘要。这意味着用户查看 Feed 时，无法通过"查看完整内容"的方式看到操作时刻的原始 body | [ActionFeedNote.php:17-21](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedNote.php#L17-L21) — `$note = $item->feedable` 加载最新 Note |
| **MorphOne vs MorphMany 的思考** | feedItem() 定义为 MorphOne，但实际上 ContactFeedItem 是操作日志（多条记录对一），MorphOne 查询语义下 `$note->feedItem` 只会返回一条（取决于 DB 默认排序）。但因为本项目从不从 Note 方向查询 feedItem，所以这个设计差异没有造成实际影响 | [Note.php:92-95](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Models/Note.php#L92-L95) |
| **顶层 description fallback 不起作用** | Feed.vue 有一个 `<div v-if="feedItem.description">` 兜底分支，但 ModuleFeedViewHelper 返回结构中没有顶层 description，对 note / post 类 action 永远不触发 | [ModuleFeedViewHelper.php:21-35](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/ModuleFeedViewHelper.php#L21-L35) |
| **删除后完整内容不可恢复** | Note 硬删除后，body 的完整内容永久丢失，只能从各 feedItem 的 10 词摘要中看到碎片化信息 | [DestroyNote:54](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/DestroyNote.php#L54) — `$this->note->delete()` |

