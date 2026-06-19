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
