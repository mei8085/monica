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

- `note_created` / `note_updated` / `note_destroyed` → 使用 [FeedItems/Note.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Shared/Modules/FeedItems/Note.vue)
- 数据由 [ActionFeedNote::data()](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageContactFeed/Web/ViewHelpers/Actions/ActionFeedNote.php) 生成：
  ```php
  'note' => [
      'object' => $note ? ['id' => ..., 'title' => ..., 'body' => Str::limit($note->body, 30)] : null,
      'description' => $item->description,
  ]
  ```
- **如果笔记仍存在**：显示 title 和 body（截断到 30 字符）
- **如果笔记已删除**（`feedable` 为 null）：显示 description（创建/更新/删除时保存的 10 词摘要）

**Post 在 Feed 中的展示**：

- `added_to_post` / `removed_from_post` → 使用通用 description 展示（无专用子组件）
- 只显示 `feedItem.description`（即 post 的 title）
- 在 [Feed.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/resources/js/Shared/Modules/Feed.vue) 第 146-152 行：
  ```html
  <div v-if="feedItem.description" class="ms-6">
      <div class="rounded-lg border border-gray-300 px-3 py-2">
          <span class="text-sm">{{ feedItem.description }}</span>
      </div>
  </div>
  ```

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

## 五、编辑和删除对 Journal/Feed 的影响

### 5.1 Note 编辑

**服务**：[UpdateNote.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Contact/ManageNotes/Services/UpdateNote.php)

**流程**：
1. 验证权限和数据
2. 更新 note 的 body、title、emotion_id
3. 更新 `contact.last_updated_at`
4. **创建新的 ContactFeedItem**（action = `note_updated`，description = body 摘要 10 词）
5. 通过 `$this->note->feedItem()->save($feedItem)` 将 feedItem 与 Note 关联

**影响**：
- Feed 中新增一条 "edited a note" 记录
- Note 原来创建时的 feedItem 不变（追加式，不替换）
- Note 自身的 `feedItem()` 是 MorphOne 关系，**新的 feedItem 会替换旧的**（MorphOne 只保留一条）

### 5.2 Note 删除

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
- 该 feedItem 的 `feedable` 为 null → 前端 FeedItems/Note.vue 显示 description（10 词摘要）而非完整笔记
- Note 原来创建/更新时关联的 feedItem 因 Note 被删除，其 `feedable` 也变为 null
- **Note 数据完全丢失，无法恢复**
- 不影响 Journal 和 Post

### 5.3 Post 编辑

**服务**：[UpdatePost.php](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Vault/ManageJournals/Services/UpdatePost.php)

**流程**：
1. 验证权限
2. 更新 post 的 title、written_at
3. 更新各 postSection 的 content

**影响**：
- **不创建 ContactFeedItem**，编辑 Post 不会在联系人 Feed 中留下记录
- 前端使用 500ms 防抖自动保存（Auto save）
- Contact 与 Post 的关联（contact_post 中间表）在 [PostController::update()](file:///d:/fz/0601-2/solo-dogfeeding/code/51-monica/app/Domains/Vault/ManageJournals/Web/Controllers/PostController.php#L109-L146) 中处理：先 `detach` 所有联系人，再逐个 `AddContactToPost`

### 5.4 Post 删除

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

### 5.5 Contact-Post 关联变更

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

## 六、核心差异总结

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
