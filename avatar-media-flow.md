# 联系人头像上传和媒体裁剪链路分析

本文档按代码执行顺序，拆解从**页面渲染 → 上传选择 → 后端入库 → 关联头像 → 展示裁剪**的完整链路。

---

## 0. 核心架构说明：使用 Uploadcare 做第三方文件托管

Monica 不自建文件存储服务，而是使用 **Uploadcare**（第三方文件 CDN 平台）。因此：

- **文件实际上传到 Uploadcare 的 CDN**（前端 JS SDK 直接上传，不经过后端服务器）
- 后端只保存文件的**元数据**（uuid、名称、URL、大小等）到 `files` 表
- **裁剪/缩放等图片处理不发生在服务器端**，而是利用 Uploadcare 提供的 URL 语法，由 CDN 在取图时实时处理（详见第 8 节）

---

## 1. 页面渲染阶段：准备上传环境

### 1.1 后端 ViewHelper 注入上传参数

当打开联系人详情页时，入口在：

[ContactShowViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php#L57-L78)

```php
'avatar' => [
    'uploadcare'    => StorageHelper::uploadcare(),
    'canUploadFile' => StorageHelper::canUploadFile($contact->vault->account),
    'hasFile'       => $contact->avatar['type'] === 'url',
],
'url' => [
    'update_avatar'  => route('contact.avatar.update',  [...]),
    'destroy_avatar' => route('contact.avatar.destroy', [...]),
],
```

- `uploadcare` 参数由 [StorageHelper::uploadcare()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Helpers/StorageHelper.php#L34-L45) 生成，包含：
  - `publicKey`：Uploadcare 公钥（环境变量 `UPLOADCARE_PUBLIC_KEY`）
  - `signature` / `expire`：基于私钥的安全签名和过期时间（使用 `Uploadcare\Security\Signature` 类），防止越权上传
- `hasFile`：通过访问 `$contact->avatar` 魔术属性，判断当前是否已有真实头像

### 1.2 路由定义

[web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/routes/web.php#L282-L283)

```php
Route::put('avatar',    [ModuleAvatarController::class, 'update'])->name('contact.avatar.update');
Route::delete('avatar', [ModuleAvatarController::class, 'destroy'])->name('contact.avatar.destroy');
```

---

## 2. 头像展示阶段：SVG 或 CDN URL

### 2.1 前端组件链

页面 [Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Pages/Vault/Contact/Show.vue#L209) 使用：

```vue
<ContactAvatar v-if="module.type === 'avatar'" :data="module.data" />
```

[ContactAvatar.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Shared/Modules/ContactAvatar.vue) 内部再调用通用组件：

```vue
<avatar :data="data.avatar" ... />
```

[Avatar.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Shared/Avatar.vue) 根据 `data.type` 决定渲染方式：

```vue
<div v-if="data.type === 'svg'" v-html="data.content" />
<img v-else :src="data.content" alt="avatar" />
```

### 2.2 后端头像数据构造：Contact::avatar Attribute

[Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/Contact.php#L482-L500) 中定义了 `avatar` 访问器：

```php
protected function avatar(): Attribute
{
    return Attribute::make(
        get: function ($value) {
            $type    = self::AVATAR_TYPE_SVG;
            $content = AvatarHelper::generateRandomAvatar($this);

            if ($this->file) {
                $type    = self::AVATAR_TYPE_URL;
                $content = 'https://ucarecdn.com/' . $this->file->uuid
                    . '/-/scale_crop/300x300/smart/'
                    . '/-/format/auto/'
                    . '/-/quality/smart_retina/';
            }

            return ['type' => $type, 'content' => $content];
        }
    );
}
```

#### 关键点：
1. **无头像（file_id 为空）→ SVG 默认头像**：调用 [AvatarHelper::generateRandomAvatar()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Helpers/AvatarHelper.php#L19-L30)，基于 MultiAvatar 库（使用姓名做种子，姓名为空时用 Faker 伪造姓名），生成一段内联 SVG 字符串，无需任何网络请求。

2. **有头像（存在 file 关联）→ CDN URL 带裁剪参数**：拼接 Uploadcare URL 处理语法：
   - `/-/scale_crop/300x300/smart/`：等比缩放后智能中心裁剪到 300×300 正方形
   - `/-/format/auto/`：根据浏览器自动选择 WebP/AVIF 等现代格式
   - `/-/quality/smart_retina/`：视网膜屏智能质量压缩

### 2.3 Contact 与 File 的关联关系

[Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/Contact.php#L318-L332)

```php
// 多态一对多：一个联系人可挂多个文件（照片、文档等）
public function files(): MorphMany
{
    return $this->morphMany(File::class, 'ufileable', 'fileable_type');
}

// 一对一：当前作为头像的那一个文件（file_id 外键直接存在 contacts 表）
public function file(): BelongsTo
{
    return $this->belongsTo(File::class);
}
```

---

## 3. 用户触发上传：前端上传组件

### 3.1 页面中的上传入口

在 [Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Pages/Vault/Contact/Show.vue#L235-L247)：

```vue
<li v-if="!data.avatar.hasFile">
  <Uploadcare
    :public-key="data.avatar.uploadcare.publicKey"
    :secure-signature="data.avatar.uploadcare.signature"
    :secure-expire="data.avatar.uploadcare.expire"
    :tabs="'file'"           // 只显示本地文件选择，不显示 URL/相机等其他源
    :preview-step="false"    // 跳过 Uploadcare 自带预览步骤
    @success="onSuccess"
    @error="onError">
    <span>Upload photo as avatar</span>
  </Uploadcare>
</li>
```

### 3.2 Uploadcare.vue 组件：JS SDK 封装

[Uploadcare.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Components/Uploadcare.vue) 封装了官方 `uploadcare-widget`：

```javascript
onClick() {
    this.fileGroup = uploadcare.openDialog([], options);

    this.fileGroup.done((filePromise) => {
        filePromise.done((file) => {
            // file 是 Uploadcare 返回的对象，包含：
            //   uuid, name, originalUrl, cdnUrl, mimeType, size 等
            this.$emit('success', file);
        });
    });
}
```

**核心要点**：`uploadcare.openDialog()` 打开官方弹窗，用户选文件后 **SDK 直接把文件 POST 到 Uploadcare 的服务器**（完全绕过 Monica 后端），成功后通过回调拿到 CDN 文件信息。

### 3.3 回调后 POST 到 Monica 后端

在 [Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Pages/Vault/Contact/Show.vue#L111-L121)（`onSuccess` 组装数据后调用 `upload()`）：

```javascript
const upload = () => {
  axios
    .put(props.data.url.update_avatar, form)
    .then((response) => {
      router.visit(response.data.data); // Inertia 跳转刷新页面
      flash('The photo has been added', 'success');
    });
};
```

`form` 里包含的字段（从 Uploadcare file 对象映射而来）：
- `uuid`、`name`、`original_url`、`cdn_url`、`mime_type`、`size`

---

## 4. 后端入库：两步原子操作

### 4.1 Controller 入口

[ModuleAvatarController::update()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageAvatar/Web/Controllers/ModuleAvatarController.php#L15-L49)

```php
public function update(Request $request, string $vaultId, string $contactId)
{
    // ── 第一步：在 files 表中创建记录 ──
    $data = [
        'account_id'  => Auth::user()->account_id,
        'author_id'   => Auth::id(),
        'vault_id'    => $vaultId,
        'uuid'        => $request->input('uuid'),
        'name'        => $request->input('name'),
        'original_url'=> $request->input('original_url'),
        'cdn_url'     => $request->input('cdn_url'),
        'mime_type'   => $request->input('mime_type'),
        'size'        => $request->input('size'),
        'type'        => File::TYPE_AVATAR,   // 类型标记：头像
    ];

    $file = (new UploadFile)->execute($data);

    // ── 第二步：把文件 ID 关联到联系人 ──
    $data = [
        'account_id' => Auth::user()->account_id,
        'author_id'  => Auth::id(),
        'vault_id'   => $vaultId,
        'contact_id' => $contactId,
        'file_id'    => $file->id,
    ];

    (new UpdatePhotoAsAvatar)->execute($data);

    return response()->json(['data' => route('contact.show', [...])], 200);
}
```

### 4.2 UploadFile Service：保存元数据

[UploadFile.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Services/UploadFile.php#L54-L88)

```php
public function execute(array $data): File
{
    $this->data = $data;
    $this->validate();   // 校验 Uploadcare 公私钥是否配置 + 字段规则
    $this->save();
    return $this->file;
}

private function save(): void
{
    $this->file = File::create([
        'vault_id'     => $this->data['vault_id'],
        'uuid'         => $this->data['uuid'],
        'name'         => $this->data['name'],
        'original_url' => $this->data['original_url'],
        'cdn_url'      => $this->data['cdn_url'],
        'mime_type'    => $this->data['mime_type'],
        'size'         => $this->data['size'],
        'type'         => $this->data['type'],   // TYPE_AVATAR / TYPE_PHOTO / TYPE_DOCUMENT
    ]);
}
```

**注意**：这里只是「写表」，不做任何文件 IO——文件早已存在于 Uploadcare CDN。

### 4.3 UpdatePhotoAsAvatar Service：关联为头像

[UpdatePhotoAsAvatar.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageAvatar/Services/UpdatePhotoAsAvatar.php#L48-L98)

```php
public function execute(array $data): Contact
{
    $this->data = $data;
    $this->validate();

    $this->deleteCurrentAvatar();   // 先删旧头像（触发模型事件→删 CDN 文件）
    $this->setAvatar();             // 设置新头像
    $this->updateLastEditedDate();  // 更新 contact.last_updated_at
    $this->createFeedItem();        // 写动态流：ACTION_CHANGE_AVATAR

    return $this->contact;
}

private function validate(): void
{
    $this->validateRules($this->data);

    // 只允许类型为 TYPE_AVATAR 的文件被设为头像
    $this->file = $this->vault->files()
        ->where('type', File::TYPE_AVATAR)
        ->findOrFail($this->data['file_id']);
}

private function setAvatar(): void
{
    $this->contact->file_id = $this->file->id;  // 保存到 contacts.file_id 外键
    $this->contact->save();

    $this->contact->files()->save($this->file); // 同时建立 morphMany 多态关联（ufileable）
}
```

---

## 5. 删除旧头像：级联触发 CDN 文件删除

### 5.1 File 模型的 deleted 事件

[File.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/File.php#L54-L56)

```php
protected $dispatchesEvents = [
    'deleted' => FileDeleted::class,
];
```

每当 `File::delete()` 被调用，就会派发 `FileDeleted` 事件。

### 5.2 DeleteFileInStorage Listener：调 API 删 CDN

[DeleteFileInStorage.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Listeners/DeleteFileInStorage.php#L34-L68)

```php
public function handle(FileDeleted $event)
{
    $this->file = $event->file;
    $this->checkAPIKeyPresence();       // 校验公私钥
    $this->getFileFromUploadcare();     // 调 Uploadcare Api 拿到文件信息对象
    $this->deleteFile();                // 调 Uploadcare Api 删除远程文件
}

private function deleteFile(): void
{
    $this->api->file()->deleteFile($this->fileInUploadcare);
}
```

**删除链路闭环**：`contact.file->delete()` → 触发 `deleted` 事件 → `DeleteFileInStorage` → Uploadcare REST API → CDN 上的真实文件被删。

---

## 6. 删除头像（用户主动移除）

### 6.1 前端触发

[Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Pages/Vault/Contact/Show.vue#L123-L130)

```javascript
const destroyAvatar = () => {
  axios
    .delete(props.data.url.destroy_avatar)
    .then((response) => { router.visit(response.data.data); });
};
```

### 6.2 DestroyAvatar Service

[DestroyAvatar.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageAvatar/Services/DestroyAvatar.php#L44-L83)

```php
public function execute(array $data): Contact
{
    $this->validate();
    $this->deleteCurrentAvatar();   // → $this->contact->file->delete() → 走第 5 节的级联链
    $this->updateLastEditedDate();
    $this->createFeedItem();
    return $this->contact;
}

private function deleteCurrentAvatar(): void
{
    if ($this->contact->file) {
        $this->contact->file->delete();  // 触发事件删 CDN
    }
    $this->contact->file_id = null;      // 置空外键 → 下次访问 avatar Attribute 退回 SVG
}
```

---

## 7. 照片（Photo）链路 vs 头像（Avatar）链路对比

**照片上传**的代码和头像几乎完全一致，区别仅在 3 点：

| 维度                  | 头像（Avatar）                                          | 照片（Photo）                                            |
| --------------------- | ------------------------------------------------------- | -------------------------------------------------------- |
| `File.type`           | `File::TYPE_AVATAR`                                     | `File::TYPE_PHOTO`                                       |
| Controller            | `ModuleAvatarController`                                | `ContactModulePhotoController`                           |
| 关联方式              | 写入 `contacts.file_id` 外键 + morphMany 关联          | 只写入 morphMany 关联（不设置外键）                     |
| Service               | `UpdatePhotoAsAvatar`                                   | 直接 `$contact->files()->save($file)`                    |
| 路由                  | `contact.avatar.update/destroy`                         | `contact.photo.store/destroy`                            |
| 展示时裁剪参数        | 固定 300×300 方形裁剪                                   | 按 ViewHelper 不同用途构造 URL                           |

照片对应的 Controller：[ContactModulePhotoController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/Controllers/ContactModulePhotoController.php)，照片列表展示在 [ContactPhotoController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/Controllers/ContactPhotoController.php)

---

## 8. 「裁剪」链路真相：Uploadcare URL 处理（非服务端图像处理）

Monica 项目中**没有任何 Intervention/ImageMagick/GD 等 PHP 图像扩展依赖**。所谓的「裁剪/缩放」完全依赖 Uploadcare 的 **URL-based Image Processing** 能力。

### 8.1 URL 语法结构

```
https://ucarecdn.com/{uuid}/-/operation/params/-/operation/params/...
```

### 8.2 头像展示使用的处理链（Contact::avatar）

```
/-/scale_crop/300x300/smart/   # 先等比缩放到覆盖 300×300，再智能居中裁剪
/-/format/auto/                # 自动判断浏览器支持的最优格式（WebP/AVIF/JPEG）
/-/quality/smart_retina/       # 智能质量，兼顾视网膜屏清晰度与文件大小
```

### 8.3 为什么这是最优设计

1. **零服务器 CPU 消耗**：图像处理全在 CDN 边缘节点完成，PHP-FPM 不做任何像素运算
2. **按需生成 + CDN 缓存**：每种尺寸第一次请求时生成，之后命中 CDN 缓存
3. **无存储成本爆炸**：不需要预先生成 thumb/medium/large 多份尺寸存本地
4. **前端按需改尺寸**：只要改 URL 参数即可获得任意尺寸，无需改后端代码

---

## 9. 完整链路时序图（文字版）

```
用户打开联系人页
  │
  ├─► ContactShowViewHelper
  │     ├─ StorageHelper::uploadcare()  生成公钥+安全签名
  │     └─ $contact->avatar  Attribute 返回 SVG 或 CDN_URL_300x300
  │
  ├─► Show.vue 渲染 <ContactAvatar> → <Avatar> 显示图片
  │
  └─► （若未上传头像）渲染 <Uploadcare> 组件
           │
           ▼  用户点击「Upload photo as avatar」
  Uploadcare.openDialog()
           │
           ▼  用户选文件
  uploadcare-widget SDK → 直接 POST 到 Uploadcare CDN
           │
           ▼  上传成功回调 (file 对象含 uuid/url/size)
  onSuccess(file) → axios.put(contact.avatar.update, form)
           │
           ▼
  ModuleAvatarController::update()
    ├─► UploadFile::execute()        → INSERT files 表（type=avatar）
    └─► UpdatePhotoAsAvatar::execute()
          ├─ deleteCurrentAvatar()   → 旧 File::delete() → FileDeleted 事件
          │                                └─► DeleteFileInStorage → Uploadcare API 删 CDN
          ├─ setAvatar()             → contacts.file_id = 新 ID + morphMany 关联
          ├─ updateLastEditedDate()  → 更新 last_updated_at
          └─ createFeedItem()        → 写入 contact_feed_items
           │
           ▼
  返回路由 → Inertia 刷新页面 → Avatar Attribute 返回新 CDN URL（带 300×300 裁剪参数）
           │
           ▼
  用户看到裁剪后的正方形头像 🎉
```

---

## 10. 涉及的关键文件速查表

| 分层       | 文件                                                         | 作用                                         |
| ---------- | ------------------------------------------------------------ | -------------------------------------------- |
| **前端**   | [Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Pages/Vault/Contact/Show.vue) | 联系人详情页，包含上传/删除按钮              |
| 前端       | [Uploadcare.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Components/Uploadcare.vue) | 封装 uploadcare-widget，处理文件选择+回调    |
| 前端       | [ContactAvatar.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Shared/Modules/ContactAvatar.vue) | 头像模块容器组件                             |
| 前端       | [Avatar.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Shared/Avatar.vue) | 根据 type 渲染 SVG 或 img 标签               |
| **路由**   | [web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/routes/web.php) | 注册 contact.avatar.* 和 contact.photo.* 路由 |
| **控制器** | [ModuleAvatarController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageAvatar/Web/Controllers/ModuleAvatarController.php) | 头像增删入口，串联 UploadFile + UpdatePhotoAsAvatar |
| 控制器     | [ContactModulePhotoController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/Controllers/ContactModulePhotoController.php) | 照片增删入口                                 |
| **Service**| [UploadFile.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Services/UploadFile.php) | 写入 files 表（所有类型文件共用）            |
| Service    | [DestroyFile.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Services/DestroyFile.php) | 删除 files 表记录（通用）                    |
| Service    | [UpdatePhotoAsAvatar.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageAvatar/Services/UpdatePhotoAsAvatar.php) | 把已上传文件设为头像                         |
| Service    | [DestroyAvatar.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageAvatar/Services/DestroyAvatar.php) | 取消头像（删文件+置空外键）                  |
| **Helper** | [StorageHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Helpers/StorageHelper.php) | 生成 Uploadcare 公钥/签名/过期时间           |
| Helper     | [AvatarHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Helpers/AvatarHelper.php) | 调用 MultiAvatar 生成默认 SVG 头像           |
| **模型**   | [Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/Contact.php) | file()/files() 关联 + avatar Attribute（构造 CDN 裁剪 URL） |
| 模型       | [File.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/File.php) | TYPE_* 常量 + deleted 事件派发               |
| 模型       | [MultiAvatar.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/MultiAvatar.php) | 第三方 SVG 头像生成库封装                    |
| **事件**   | [DeleteFileInStorage.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Listeners/DeleteFileInStorage.php) | FileDeleted 监听 → 调 Uploadcare API 删远程文件 |
| **视图帮助** | [ContactShowViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageContact/Web/ViewHelpers/ContactShowViewHelper.php) | 准备上传参数 + 上传/删除路由 URL             |

---

## 11. 缺名联系人每次刷新都换一张默认头像的成因

### 11.1 问题现象

当联系人没有设置 `first_name`（即匿名联系人）时，页面每刷新一次，SVG 默认头像就变一张——颜色、发型、表情全都不同。而有名字的联系人则始终显示同一种头像。

### 11.2 根因追踪

调用链：`Contact::avatar` Attribute → `AvatarHelper::generateRandomAvatar()` → `MultiAvatar`

[AvatarHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Helpers/AvatarHelper.php#L19-L30)

```php
public static function generateRandomAvatar(Contact $contact): string
{
    $multiavatar = new MultiAvatar;

    if (is_null($contact->first_name)) {
        $name = Faker::create()->name();   // ← 问题根源
    } else {
        $name = $contact->first_name.' '.$contact->last_name;
    }

    return $multiavatar($name, null, null);
}
```

**三层原因叠加**：

1. **Faker 不可复现**：`Faker::create()->name()` 每次调用都生成不同的随机姓名（如 "Prof. Janick Collins"、"Miss Alaina Lehner"）。Faker 没有固定种子（seed），也没有使用联系人的 `id` 做伪随机种子。

2. **MultiAvatar 是纯确定性函数，但种子每次不同**：[MultiAvatar.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/MultiAvatar.php#L26-L29) 的 `__invoke` 接受 `avatarId` 参数，同一个 `avatarId` 永远产出同一张 SVG。对于有名字的联系人，`"John Doe"` 每次传入都一样，所以头像稳定。但对于缺名联系人，每次传入不同的 Faker 姓名作为种子，输出自然不同。

3. **avatar Attribute 是实时计算的访问器，无缓存**：[Contact::avatar](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/Contact.php#L482-L500) 是一个 Eloquent Attribute `get` 访问器，每次被访问都会重新执行。Inertia 在每次页面渲染时序列化联系人数据，都会调用此访问器，触发 `AvatarHelper::generateRandomAvatar()`，触发新的 Faker 调用。

### 11.3 有名字联系人不换头像的原因

有 `first_name` 的联系人，种子字符串是 `first_name + ' ' + last_name`（如 `"Alice Smith"`），这是一个**确定性的字符串**——不会因刷新而变化，MultiAvatar 对相同输入始终产出相同输出。

### 11.4 影响范围

- 同一个缺名联系人在不同请求、不同用户、不同时间看到的默认头像都不同
- 列表页中同一联系人在翻页后头像也会变
- 即使同一页面，如果 `avatar` 属性被访问多次，理论上也可能得到不同结果（Faker 实例非全局单例）

### 11.5 可选修复方向

| 方案 | 做法 | 优劣 |
| ---- | ---- | ---- |
| A. 用 contact id 做种子 | `Faker::create( crc32($contact->id) )->name()` | 简单；同一联系人总是同一 Faker 姓名→同一 SVG |
| B. 省掉 Faker，直接用 id | `$name = $contact->id` 传给 MultiAvatar | 最简；但 MultiAvatar 对纯数字字符串的视觉效果可能不佳 |
| C. 首次生成后持久化 | 创建联系人时生成 SVG 存到 `default_avatar` 列 | 最稳定，但增加存储和迁移成本 |

---

## 12. files 表两套关联列并存现状：fileable 与 ufileable

### 12.1 迁移中的表结构

[2022_02_24_002342_create_files_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/database/migrations/2022_02_24_002342_create_files_table.php#L18-L34)

```php
Schema::create('files', function (Blueprint $table) {
    $table->id();
    $table->foreignIdFor(Vault::class)->constrained()->cascadeOnDelete();

    // 第一套：nullableNumericMorphs → fileable_id (BIGINT UNSIGNED NULL) + fileable_type (STRING NULL)
    $table->nullableNumericMorphs('fileable');

    // 第二套：ufileable_id (UUID NULL) + 复用 fileable_type 作为类型列
    $table->uuid('ufileable_id')->nullable();
    $table->index(['fileable_type', 'ufileable_id']);

    $table->string('uuid');
    // ...
});
```

### 12.2 两套关联的语义对比

| 列 | 类型 | 用途 | 来源 |
| -- | ---- | ---- | ---- |
| `fileable_id` | `BIGINT UNSIGNED NULL` | Laravel 标准 `nullableNumericMorphs`，存**数字型**外键 | 迁移自动生成 |
| `fileable_type` | `STRING NULL` | 多态类型名，如 `App\Models\Contact`、`App\Models\Post` | 两套共用 |
| `ufileable_id` | `UUID NULL` | 存 **UUID 型**外键，专给使用 UUID 主键的模型用 | 手动添加 |

**核心矛盾**：Monica 的 `Contact` 模型使用 UUID 主键（`HasUuids` trait），而 Laravel 的 `nullableNumericMorphs` 只能存 `BIGINT`。因此原有的 `fileable_id` 列无法存储 UUID 值，必须另加 `ufileable_id` 列来承载 UUID。

### 12.3 两套 MorphTo 关系在 File 模型中的定义

[File.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/File.php#L68-L82)

```php
// 第一套：标准 MorphTo，查找 fileable_id + fileable_type
public function fileable(): MorphTo
{
    return $this->morphTo();
}

// 第二套：自定义 MorphTo，查找 ufileable_id + fileable_type
public function ufileable(): MorphTo
{
    return $this->morphTo(type: 'fileable_type');
}
```

第二套 `ufileable()` 的写法 `morphTo(type: 'fileable_type')` 利用了 Laravel MorphTo 的参数重载：
- 第一个参数 `name` 默认是方法名（即 `ufileable`），Laravel 会查找 `ufileable_type` 和 `ufileable_id` 列
- 传入 `type: 'fileable_type'` 后，类型列被改为 `fileable_type`（与第一套共用），而 ID 列仍由方法名推得为 `ufileable_id`

### 12.4 Contact 模型使用的是第二套

[Contact.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/Contact.php#L318-L321)

```php
public function files(): MorphMany
{
    return $this->morphMany(File::class, 'ufileable', 'fileable_type');
}
```

第三个参数 `'fileable_type'` 告诉 Laravel 用 `fileable_type` 列而不是 `ufileable_type` 列来存储多态类型名。这样当调用 `$contact->files()->save($file)` 时，写入的是：
- `ufileable_id` = 联系人的 UUID
- `fileable_type` = `App\Models\Contact`
- `fileable_id` = NULL（不写入）

### 12.5 两套并存的现状总结

| 模型 | 主键类型 | 使用哪套关联 | fileable_id | ufileable_id | fileable_type |
| ---- | -------- | ------------ | ----------- | ------------ | ------------- |
| Contact | UUID | 第二套 (ufileable) | NULL | UUID 值 | `App\Models\Contact` |
| Post | UUID | 第二套 (ufileable) | NULL | UUID 值 | `App\Models\Post` |
| 其他可能的数字 ID 模型 | BIGINT | 第一套 (fileable) | BIGINT 值 | NULL | 类名 |

**实际影响**：
- `fileable_id` 列在当前 Contact/Photo/Document 流程中**始终为 NULL**，因为所有用到多态关联的模型都使用 UUID
- 查询时 `File::where('ufileable_id', $contactId)->where('type', File::TYPE_PHOTO)` 直接走 UUID 列
- `fileable()` 关系方法在当前业务中**未被调用**，属于迁移遗留

**潜在问题**：
- 查询照片列表时（[ContactPhotoController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/Controllers/ContactPhotoController.php#L23-L26)），直接用 `where('ufileable_id', $contactId)` 而非通过 Eloquent 关系，绕过了多态类型过滤——如果未来有其他模型也使用 `ufileable` 关联，可能查出非 Contact 的文件
- 两套关联列共存增加理解成本，`fileable_id` 在 UUID 场景下是死列

---

## 13. 删除文件时同步请求第三方 CDN 的链路与风险

### 13.1 完整同步链路

```
用户操作（更换头像/删除头像/删除照片/删除文档）
  │
  ▼
Service 层调用 $file->delete()
  │
  ▼  Eloquent deleted 事件
FileDeleted 事件被派发
  │
  ▼  同步 Listener（注册在 AppServiceProvider）
DeleteFileInStorage::handle()
  │
  ├─► checkAPIKeyPresence()     ← 纯本地检查
  ├─► getFileFromUploadcare()   ← HTTP 请求 1: GET /files/{uuid}/
  └─► deleteFile()              ← HTTP 请求 2: DELETE /files/{uuid}/
  │
  ▼  返回给 Service 层
响应返回给前端
```

### 13.2 事件注册方式

[AppServiceProvider.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Providers/AppServiceProvider.php#L158)

```php
Event::listen(FileDeleted::class, DeleteFileInStorage::class);
```

这行注册的是**同步监听器**，不是 Queued Listener。`FileDeleted` 事件本身也**没有实现 `ShouldQueue` 接口**：

[FileDeleted.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Events/FileDeleted.php)

```php
class FileDeleted
{
    use Dispatchable;
    use InteractsWithSockets;
    use SerializesModels;
    // 没有 ShouldQueue
}
```

这意味着 `DeleteFileInStorage::handle()` 在**同一个 HTTP 请求生命周期内同步执行**。

### 13.3 同步链路的风险分析

| 风险 | 分析 | 严重度 |
| ---- | ---- | ------ |
| **请求延迟叠加** | 删除一个文件 = 2 次同步 HTTP 往返（先查 fileInfo 再 delete）。如果用户更换头像，要先删旧头像（2 次 HTTP）再建新关联——前端等待时间 = 本地 DB 操作 + 2 次 Uploadcare API 调用 | 中 |
| **Uploadcare 宕机 = 删除操作失败** | `getFileFromUploadcare()` 中 HTTP 请求失败时抛出 `BadRequestHttpException`，这会导致整个删除流程中断——本地 DB 记录已标记删除，但 CDN 文件可能未被真正删除 | 高 |
| **无重试机制** | 同步 Listener 不走队列，失败后没有自动重试。如果 API 调用超时或 5xx，CDN 上的文件成为孤儿——本地记录已删，CDN 文件残留，持续计费 | 高 |
| **DB 事务与事件时序** | Eloquent 的 `deleted` 事件在 `DELETE` SQL 执行**之后**触发。如果 Listener 抛异常，DB 记录已物理删除（不在事务中回滚），但 CDN 文件未删 | 高 |
| **并发删除同一文件** | 两个请求同时删同一 File 记录，第二个 `fileInfo` 调用可能返回 404（Uploadcare 上已被第一个请求删除），异常被抛出 | 低 |

### 13.4 一个值得注意的细节：两步 HTTP 调用的必要性存疑

[DeleteFileInStorage.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Listeners/DeleteFileInStorage.php#L53-L68)

```php
private function getFileFromUploadcare(): void
{
    $configuration = Configuration::create(...);
    $this->api = new Api($configuration);
    $this->fileInUploadcare = $this->api->file()->fileInfo($this->file->uuid);
}

private function deleteFile(): void
{
    $this->api->file()->deleteFile($this->fileInUploadcare);
}
```

`deleteFile()` 接受的是一个 `FileInfoInterface` 对象，而不是 UUID 字符串。所以必须先调 `fileInfo()` 拿到完整对象再传给 `deleteFile()`。这是 Uploadcare PHP SDK 的 API 设计限制——如果能直接按 UUID 删除，可以省掉一次 HTTP 往返。

### 13.5 改进方向

| 方向 | 做法 | 效果 |
| ---- | ---- | ---- |
| 异步化 | 让 `DeleteFileInStorage` 实现 `ShouldQueue`，用 Redis/数据库队列异步处理 | 请求立即返回，CDN 删除后台进行 |
| 幂等重试 | 队列 + 指数退避重试，404 视为成功（文件已删） | 解决孤儿文件问题 |
| 简化调用 | 如果 SDK 支持直接按 UUID 删除，去掉 `fileInfo()` 预查询 | 少一次 HTTP 往返 |

---

## 14. 上传校验对类型与大小的限制

### 14.1 后端校验层：UploadFile Service

[UploadFile.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Services/UploadFile.php#L20-L31)

```php
public function rules(): array
{
    return [
        'account_id'  => 'required|uuid|exists:accounts,id',
        'vault_id'    => 'required|uuid|exists:vaults,id',
        'author_id'   => 'required|uuid|exists:users,id',
        'uuid'        => 'required|string',
        'name'        => 'required|string',
        'original_url'=> 'required|string',
        'cdn_url'     => 'required|string',
        'mime_type'   => 'required|string',
        'size'        => 'required|integer',
        'type'        => 'required|string',
    ];
}
```

**关键发现**：

| 字段 | 校验规则 | 缺失的限制 |
| ---- | -------- | ---------- |
| `mime_type` | `required\|string` | **没有 `mimetypes` 或 `mimes` 规则**，不检查是否为 image/* 等合法类型 |
| `size` | `required\|integer` | **没有 `min`/`max` 规则**，0 字节或数十 GB 的 size 值都能通过 |
| `type` | `required\|string` | **没有 `in:document,avatar,photo` 规则**，任意字符串都能写入 |
| `uuid` | `required\|string` | **没有 `uuid` 格式校验**（对比 `account_id` 有 `uuid` 规则），任意字符串都能作为 Uploadcare UUID 传入 |
| `name` | `required\|string` | 没有长度限制、没有文件扩展名白名单 |

也就是说，**后端 UploadFile Service 不校验文件类型、不限制文件大小、不验证 UUID 格式**。它只确保这些字段「存在且为字符串/整数」。

### 14.2 后端校验层：UpdatePhotoAsAvatar Service

[UpdatePhotoAsAvatar.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageAvatar/Services/UpdatePhotoAsAvatar.php#L21-L29)

```php
public function rules(): array
{
    return [
        'account_id' => 'required|uuid|exists:accounts,id',
        'vault_id'   => 'required|uuid|exists:vaults,id',
        'author_id'  => 'required|uuid|exists:users,id',
        'contact_id' => 'required|uuid|exists:contacts,id',
        'file_id'    => 'nullable|integer|exists:files,id',
    ];
}
```

这里额外校验了 `file_id` 必须存在于 `files` 表，且在 `validate()` 方法中进一步限定 `where('type', File::TYPE_AVATAR)`——只有 type 为 avatar 的文件才能被设为头像。这是唯一一层对文件类型的间接约束。

### 14.3 存储配额校验：StorageHelper::canUploadFile

[StorageHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Helpers/StorageHelper.php#L15-L31)

```php
public static function canUploadFile(Account $account): bool
{
    if ($account->storage_limit_in_mb == 0) {
        return true;   // 0 = 不限量
    }

    $vaultIds = $account->vaults()->select('id')->get()->toArray();
    $totalSizeInBytes = File::whereIn('vault_id', $vaultIds)->sum('size');
    $accountLimit = $account->storage_limit_in_mb * 1024 * 1024;

    return $totalSizeInBytes < $accountLimit;
}
```

**这是 Monica 唯一的大小限制机制**：
- 在前端渲染时计算 `canUploadFile`，决定是否显示上传按钮
- 如果账户已用空间 >= 配额，上传按钮不渲染
- **但后端没有二次校验**：即使前端隐藏了按钮，直接 `PUT /avatar` 仍可绕过此限制，因为 `UploadFile` Service 不检查配额

### 14.4 前端 Uploadcare Widget 的限制

[Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Pages/Vault/Contact/Show.vue#L236-L244) 中头像上传组件的配置：

```vue
<Uploadcare
  :public-key="data.avatar.uploadcare.publicKey"
  :secure-signature="data.avatar.uploadcare.signature"
  :secure-expire="data.avatar.uploadcare.expire"
  :tabs="'file'"
  :preview-step="false"
  @success="onSuccess"
  @error="onError">
```

而 [Uploadcare.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Components/Uploadcare.vue) 支持的 props：

| Prop | 默认值 | 头像上传时传的值 | 作用 |
| ---- | ------ | ---------------- | ---- |
| `imagesOnly` | `false` | **未传（默认 false）** | 如果为 true，Uploadcare 弹窗只允许选图片 |
| `crop` | `''` | **未传（空字符串）** | 如果设置如 `"1:1"`，Uploadcare 会强制裁剪 |
| `imageShrink` | `false` | **未传** | 如果为 true，大图会被自动缩小 |
| `multipartMinSize` | `26214400`（25MB） | **未传（默认 25MB）** | 超过此大小启用分片上传 |

**关键发现**：
- 头像上传时 `imagesOnly` 没有设为 `true`——理论上用户可以上传 PDF、ZIP 等非图片文件作为头像
- `crop` 没有设置——上传头像时不强制 1:1 裁剪，最终展示时的方形裁剪完全由 CDN URL 参数 `scale_crop/300x300` 完成，这意味着原始文件可能是任意比例的图
- `imageShrink` 没有启用——大图（如 8000×6000 的相机原片）会原封不动地传到 CDN，虽然 CDN 取图时会缩放，但原始文件占用 Uploadcare 存储空间

### 14.5 对比：照片上传和文档上传

[ContactModulePhotoController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/Controllers/ContactModulePhotoController.php#L16-L40) 的上传逻辑与头像完全一致——同样调用 `UploadFile`，同样无类型/大小校验，仅 `type` 字段改为 `TYPE_PHOTO`。

照片页面的 Uploadcare 配置（在照片列表页 Vue 中）可能不同（可能启用了 `imagesOnly` 和 `crop`），但后端校验层同样没有约束。

### 14.6 校验缺失的风险汇总

| 风险 | 场景 | 后果 |
| ---- | ---- | ---- |
| 无 MIME 类型校验 | 恶意请求传 `mime_type: application/pdf` | 非图片文件被当作头像/照片存入，avatar Attribute 的 CDN 裁剪 URL 返回错误 |
| 无文件大小上限 | 传 `size: 9999999999` | 绕过前端配额检查，Uploadcare 存储被滥用（实际 Uploadcare 账户可能有自己的大小限制） |
| 无 type 枚举校验 | 传 `type: malicious` | files 表出现非预期类型值，查询过滤可能遗漏 |
| UUID 无格式校验 | 传 `uuid: ../../../etc/passwd` | 低风险——此 uuid 只用于拼接 CDN URL，不用于本地文件路径 |
| 前端不强制 imagesOnly | 用户在 Uploadcare 弹窗选了非图片文件 | 文件能成功上传到 CDN，但 CDN 的 scale_crop 操作会失败，头像展示异常 |

---

## 15. Uploadcare 签名鉴权算法的完整实现

### 15.1 算法本质：HMAC-SHA256

Uploadcare 的签名上传（Signed Uploads）鉴权核心是 **HMAC-SHA256**。算法的输入只有两个：

- **密钥**：Uploadcare 项目的 Private Key（存储在 `config/services.php` 的 `uploadcare.private_key`）
- **消息**：过期时间的 Unix 时间戳字符串

签名公式为：

```
signature = HMAC-SHA256(key=PRIVATE_KEY, message=str(expire_timestamp))
```

### 15.2 PHP SDK 中的 Signature 类源码

Monica 依赖的是 `uploadcare/uploadcare-php` v4.2.0（见 [composer.lock](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/composer.lock#L11619)）。其 `Signature` 类位于 `Uploadcare\Security\Signature`：

```php
class Signature implements SignatureInterface
{
    private string $secretKey;
    private \DateTimeInterface $expired;

    public function __construct(string $secretKey, int $ttl = null)
    {
        $this->secretKey = $secretKey;
        if ($ttl === null || $ttl > self::MAX_TTL) {
            $ttl = self::MAX_TTL;        // MAX_TTL = 3600 秒（1 小时）
        }
        $ts = \date_create()->getTimestamp() + $ttl;
        $this->expired = \date_create()->setTimestamp($ts);
    }

    public function getSignature(): string
    {
        $signString = $this->getExpire()->getTimestamp();
        return \hash_hmac(
            SignatureInterface::SIGN_ALGORITHM,   // 'sha256'
            (string) $signString,                  // 过期时间戳字符串
            $this->secretKey                       // 私钥
        );
    }

    public function getExpire(): \DateTimeInterface
    {
        return $this->expired;
    }
}
```

**关键细节**：

1. **签名消息就是 `expire` 时间戳本身**，不是 `expire + UUID + 其他参数` 的组合。这意味着同一个签名可以用于同一个项目内的**任何文件上传**——签名不绑定具体文件
2. **MAX_TTL = 3600 秒**：构造函数未传 TTL 时默认 1 小时有效期，传入超过 3600 的值也会被截断到 3600
3. **签名是 hex 编码的**：`hash_hmac()` 默认返回 64 字符的十六进制小写字符串

### 15.3 StorageHelper 如何调用

[StorageHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Helpers/StorageHelper.php#L36-L45)

```php
public static function uploadcare(): array
{
    $signature = config('services.uploadcare.private_key') != ''
        ? new Signature(config('services.uploadcare.private_key'))
        : null;

    return [
        'publicKey' => config('services.uploadcare.public_key'),
        'signature' => optional($signature)->getSignature(),
        'expire'    => optional(optional($signature)->getExpire())->getTimestamp(),
    ];
}
```

**调用链**：

1. 从 `config/services.php` 读取私钥，构造 `Signature` 对象（未传 TTL → 默认 3600 秒）
2. 调用 `getSignature()` 得到 HMAC-SHA256 签名字符串
3. 调用 `getExpire()->getTimestamp()` 得到过期时间戳（当前时间 + 3600）
4. 将公钥、签名、过期时间三值返回给前端

**重要：每次调用 `StorageHelper::uploadcare()` 都会生成新的签名和过期时间**。因为 `Signature` 构造函数在实例化时就计算了 `now + ttl`，所以不同请求拿到的 `expire` 不同，`signature` 也不同。

### 15.4 前端如何消费签名

[Uploadcare.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Components/Uploadcare.vue#L109-L134) 把 `secureSignature` 和 `secureExpire` 直接传入 `uploadcare.openDialog()` 的 options：

```javascript
const options = {
    publicKey,
    secureSignature,    // HMAC-SHA256 签名
    secureExpire,       // Unix 时间戳（秒）
    // ...
};
this.fileGroup = uploadcare.openDialog([], options);
```

uploadcare-widget SDK 在向 Uploadcare CDN 上传文件时，会自动把 `secureSignature` 和 `secureExpire` 附加到上传请求中。Uploadcare 服务器收到请求后：

1. 用自己保存的私钥 + 请求中的 `expire` 值，重新计算 `HMAC-SHA256`
2. 对比计算结果与请求中的 `signature`
3. 检查 `expire` 是否大于当前时间（防过期）
4. 两项均通过才允许上传

### 15.5 签名的安全特性与局限

| 特性 | 说明 |
| ---- | ---- |
| **防未授权上传** | 没有私钥就无法伪造合法签名，攻击者无法直接向 Uploadcare 项目上传文件 |
| **有时效性** | 签名最多 1 小时有效，过期后无法使用 |
| **不绑定文件** | 签名只含时间戳，不含文件 UUID 或类型——合法签名可用于上传任意文件到该项目 |
| **不绑定用户** | 签名不包含用户 ID——拿到签名的任何人都可以使用（包括被截获后重放，但在 1 小时内） |
| **私钥在 Monica 后端** | 私钥只存在于服务端 `config/services.php`，不会传给前端，不会被暴露 |

---

## 16. 照片上传组件对图片类型的限制——全场景对比

### 16.1 Uploadcare.vue 组件支持的图片相关 props

[Uploadcare.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Components/Uploadcare.vue#L26-L55) 提供了以下与文件类型限制相关的 props：

| Prop | 类型 | 默认值 | 作用 |
| ---- | ---- | ------ | ---- |
| `imagesOnly` | Boolean | `false` | Uploadcare 弹窗只允许选择图片文件 |
| `crop` | String | `''` | 设置裁剪比例，如 `"1:1"`、`"3:2"` 等 |
| `imageShrink` | Boolean | `false` | 上传大图时自动缩小（减少 CDN 存储） |
| `inputAcceptTypes` | String | — | HTML `<input accept>` 属性值，如 `"image/*"` |
| `preferredTypes` | String | — | 优先选择的文件类型列表 |

### 16.2 全部调用场景的 props 对照

| 调用场景 | 组件文件 | `imagesOnly` | `crop` | `imageShrink` | `tabs` | `inputAcceptTypes` |
| -------- | -------- | ------------ | ------ | ------------- | ------ | ------------------ |
| 头像上传 | [Show.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Pages/Vault/Contact/Show.vue#L236-L246) | ❌ 未传 (false) | ❌ 未传 ('') | ❌ 未传 (false) | `'file'` | ❌ 未传 |
| 照片上传（模块） | [Photos.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Shared/Modules/Photos.vue#L12-L22) | ❌ 未传 (false) | ❌ 未传 ('') | ❌ 未传 (false) | `'file'` | ❌ 未传 |
| 照片上传（列表页） | [Index.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Pages/Vault/Contact/Photos/Index.vue#L84-L94) | ❌ 未传 (false) | ❌ 未传 ('') | ❌ 未传 (false) | `'file'` | ❌ 未传 |
| 文档上传 | [Documents.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Shared/Modules/Documents.vue#L12-L22) | ❌ 未传 (false) | ❌ 未传 ('') | ❌ 未传 (false) | `'file'` | ❌ 未传 |
| Journal 封面图 | [Show.vue (Slices)](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Pages/Vault/Journal/Slices/Show.vue#L123-L149) | ❌ 未传 (false) | ❌ 未传 ('') | ❌ 未传 (false) | `'file'` | ❌ 未传 |

**结论：全场景无差异**——Monica 中所有 Uploadcare 调用点都没有设置 `imagesOnly`、`crop`、`imageShrink`、`inputAcceptTypes` 中的任何一个。头像、照片、文档、Journal 封面图的上传弹窗行为完全相同，用户可以上传任意类型的文件。

### 16.3 这意味着什么

1. **头像上传**：用户可以选 PDF 文件上传 → CDN 存了 PDF → `Contact::avatar` Attribute 拼接的 `scale_crop/300x300/smart/` URL 对 PDF 无效 → 头像位置显示破损图片
2. **照片上传**：用户可以选 ZIP 文件上传 → 同样 CDN URL 裁剪失败 → 照片展示异常
3. **文档上传**：这个场景**不需要** `imagesOnly`，因为文档本身就可以是任意类型——但前端和后端都没有阻止用户把 ZIP 当照片上传
4. **Journal 封面图**：与头像同理，上传非图片文件后封面展示异常

---

## 17. 文档上传与照片上传的保护机制差异

### 17.1 Controller 层差异

**照片上传**：[ContactModulePhotoController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/Controllers/ContactModulePhotoController.php#L16-L40)

```php
public function store(Request $request, string $vaultId, string $contactId)
{
    $data = [
        // ...
        'type' => File::TYPE_PHOTO,   // 硬编码类型
    ];
    $file = (new UploadFile)->execute($data);
    $contact = Contact::where('vault_id', $vaultId)->findOrFail($contactId);
    $contact->files()->save($file);   // 只建 morphMany 关联，不设 file_id
    return response()->json([...], 201);
}
```

**文档上传**：[ContactModuleDocumentController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Web/Controllers/ContactModuleDocumentController.php#L16-L40)

```php
public function store(Request $request, string $vaultId, string $contactId)
{
    $data = [
        // ...
        'type' => File::TYPE_DOCUMENT,   // 硬编码类型
    ];
    $file = (new UploadFile)->execute($data);
    $contact = Contact::where('vault_id', $vaultId)->findOrFail($contactId);
    $contact->files()->save($file);   // 同样只建 morphMany 关联
    return response()->json([...], 201);
}
```

两者的 Controller 代码结构**完全一致**，唯一区别是 `File::TYPE_PHOTO` vs `File::TYPE_DOCUMENT`。

### 17.2 保护机制对比

| 保护层 | 照片上传 | 文档上传 | 差异 |
| ------ | -------- | -------- | ---- |
| **UploadFile Service 校验** | `mime_type: required\|string`，无类型限制 | 同左 | 无差异 |
| **UploadFile Service 校验** | `size: required\|integer`，无大小限制 | 同左 | 无差异 |
| **Controller 硬编码 type** | `File::TYPE_PHOTO` | `File::TYPE_DOCUMENT` | 仅类型标记不同 |
| **前端 imagesOnly** | ❌ 未设 | ❌ 未设 | 无差异 |
| **前端 crop** | ❌ 未设 | ❌ 不需要 | 合理差异 |
| **前端 imageShrink** | ❌ 未设 | ❌ 不需要 | 合理差异 |
| **展示时 CDN 裁剪** | ✅ 使用 `scale_crop` 等 URL 参数 | ❌ 直接展示/下载原文件 | **关键差异** |
| **存储配额检查** | `canUploadFile` 前端挡位 | `canUploadFile` 前端挡位 | 无差异 |
| **删除链路** | 同步调 CDN API 删除 | 同步调 CDN API 删除 | 无差异 |

### 17.3 关键差异分析：展示层才是真正的保护分水岭

照片和文档上传在后端和前端的保护机制**完全相同**——都没有对文件类型和大小做校验。但两者在**展示层**有本质区别：

- **照片**的展示 URL 包含 CDN 图像处理操作（`scale_crop`、`format/auto` 等）。如果用户上传了非图片文件（如 PDF），Uploadcare CDN 在执行 `scale_crop` 时会返回错误响应，前端 `<img>` 标签显示破损图标
- **文档**的展示是直接下载或预览原始文件，不做任何 CDN 图像处理。非图片文件在文档场景下**完全正常**——PDF、Word、ZIP 都可以正常下载

换句话说，**照片展示层的 CDN 处理反而是唯一一层对文件类型的隐式校验**——虽然它不是主动拒绝上传，而是让错误类型的文件在展示时「自然失败」。

### 17.4 差异化保护应该是怎样的

合理的保护策略应该是：

| 保护 | 照片/头像 | 文档 |
| ---- | --------- | ---- |
| 前端 `imagesOnly` | ✅ 应设为 `true` | ❌ 不设 |
| 后端 `mime_type` 校验 | ✅ 应校验 `image/*` | ❌ 不限 |
| 前端 `crop` (头像) | ✅ 应设为 `"1:1"` | ❌ 不设 |
| 前端 `imageShrink` | ✅ 应设为 `true` | ❌ 不设 |
| 后端 `size` 上限 | ✅ 应设上限（如 10MB） | ✅ 应设上限（如 50MB） |

---

## 18. Faker 确定性种子头像的修复方案及现行 API 写法不正确之处

### 18.1 现行 API 写法的问题

[AvatarHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Helpers/AvatarHelper.php#L19-L30) 的当前实现：

```php
public static function generateRandomAvatar(Contact $contact): string
{
    $multiavatar = new MultiAvatar;

    if (is_null($contact->first_name)) {
        $name = Faker::create()->name();
    } else {
        $name = $contact->first_name.' '.$contact->last_name;
    }

    return $multiavatar($name, null, null);
}
```

**问题 1：Faker 无种子，导致不可复现**

`Faker::create()` 等价于 `Faker\Factory::create()`，内部调用 `new Generator()` 时**不传入种子**。PHP 的 `mt_rand()` 每次进程启动时自动播种，所以同一个 PHP-FPM worker 进程内的连续请求可能恰好一致，但不同 worker 之间、不同请求之间结果都是随机的。

**问题 2：方法签名不正确——`generateRandomAvatar` 不是「随机」的应有语义**

方法名说的是「生成随机头像」，但实际需求是「生成确定性默认头像」——同一联系人每次应看到同一张。真正随机是当前实现的 bug，而非 feature。方法名和实现恰好颠倒了意图。

**问题 3：Faker 产生的是人类姓名，而不是最佳种子**

MultiAvatar 的算法（[MultiAvatar.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/MultiAvatar.php#L562-L570)）是：

```php
$sha256Hash = hash('sha256', $avatarId);
$sha256Numbers = preg_replace('/[^0-9]/', '', $sha256Hash);
$hash = substr($sha256Numbers, 0, 12);
```

它先对输入字符串做 SHA-256，再提取数字位。这意味着**任何输入字符串都能产生有效的头像**——不需要是人类姓名。Faker 的姓名（如 "Prof. Janick Collins"）和联系人的 UUID（如 `"550e8400-e29b-41d4-a716-446655440000"`）在经过 SHA-256 后效果完全等价——都是高质量的伪随机散列输入。

### 18.2 正确的修复方案

**方案 A：直接用 `$contact->id` 作为种子（推荐）**

```php
public static function generateRandomAvatar(Contact $contact): string
{
    $multiavatar = new MultiAvatar;

    $name = $contact->first_name
        ? $contact->first_name . ' ' . $contact->last_name
        : $contact->id;

    return $multiavatar($name, null, null);
}
```

优势：
- `Contact` 使用 UUID 主键（`HasUuids` trait），`$contact->id` 是确定性值，同一联系人永远相同
- 无需引入 Faker 依赖
- MultiAvatar 的 SHA-256 算法对 UUID 字符串同样能产生高质量散列
- 改动最小——只删了 Faker 分支，改为一个三元表达式

**方案 B：用 Faker 但固定种子**

```php
if (is_null($contact->first_name)) {
    $name = Faker::create(crc32($contact->id))->name();
} else {
    $name = $contact->first_name.' '.$contact->last_name;
}
```

这种方式也能保证确定性——`crc32($contact->id)` 对同一 UUID 总是产出相同整数，Faker 对相同种子总是产出相同姓名。但这是**不必要的间接层**：Faker 姓名只是 MultiAvatar SHA-256 的输入，为什么不直接把 UUID 传给 SHA-256？

**方案 C：首次生成后持久化**

在 `contacts` 表加 `default_avatar` 列，创建联系人时生成一次 SVG 存入。这最稳定但成本最高，对于默认头像这种纯装饰性内容来说过度设计。

### 18.3 方案 A 的可行性验证

[MultiAvatar.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/MultiAvatar.php#L562-L631) 的 `generate()` 方法对 `avatarId` 做了：

1. `hash('sha256', $avatarId)` — 无论输入是姓名还是 UUID，SHA-256 都是 64 位十六进制
2. 提取数字位 — UUID 格式 `"550e8400-e29b-41d4-a716-446655440000"` 的 SHA-256 数字位足够多
3. 取前 12 位数字 → 分成 6 组（env/clo/head/mouth/eyes/top），每组 2 位 → 映射到 0-47 的范围

也就是说，**UUID 输入与姓名输入在 MultiAvatar 算法中的效果完全对等**，不会出现「纯数字字符串视觉效果不佳」的情况——因为输入根本不是直接映射，而是经过 SHA-256 散列。

### 18.4 还需考虑的边界情况

| 边界 | 当前行为 | 修复后行为 |
| ---- | -------- | ---------- |
| `first_name` 为空字符串 `""` | `is_null("")` = false → 拼出 `"  Smith"`（前导空格） | 同上（三元表达式 `""` 不走 null 分支） |
| `first_name` 为 null，`last_name` 有值 | Faker 随机 → MultiAvatar 不稳定 | 用 `$contact->id` → 稳定 |
| `first_name` 和 `last_name` 都为 null | 同上 | 同上 |
| 联系人 `id` 为空（未持久化） | Faker 随机 | `$contact->id` 为 null → MultiAvatar 收到空字符串 → 返回空 SVG |

最后一个边界情况需要注意：**未持久化的 Contact 没有 id**。如果 `generateRandomAvatar()` 在 `Contact::create()` 之前被调用（虽然当前代码不存在此场景），UUID 主键尚未生成，会传入 null。但当前所有调用点都在联系人已存在之后，所以这不是实际问题。

---

## 19. DestroyFile Service 被多控制器共用但未按类型守门的权限问题

### 19.1 四个控制器共用同一个 DestroyFile

通过代码搜索发现，[DestroyFile](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Services/DestroyFile.php) Service 被以下 4 个控制器的 destroy 方法调用：

| 控制器 | 路由 | 删除场景 | 预期删除的 type |
| ------ | ---- | -------- | ---------------- |
| [ContactModulePhotoController](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/Controllers/ContactModulePhotoController.php#L42-L59) | `contact.photo.destroy` | 联系人照片模块中删除某张照片 | `TYPE_PHOTO` |
| [ContactModuleDocumentController](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Web/Controllers/ContactModuleDocumentController.php#L42-L56) | `contact.document.destroy` | 联系人文档模块中删除某个文档 | `TYPE_DOCUMENT` |
| [PostPhotoController](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Web/Controllers/PostPhotoController.php#L47-L61) | `post.photos.destroy` | Journal 的 Post 中删除某张内嵌照片 | `TYPE_PHOTO` |
| [VaultFileController](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageFiles/Web/Controllers/VaultFileController.php#L85-L99) | `vault.files.destroy` | Vault 级文件管理页（所有类型汇总） | 任意 type（合理） |

### 19.2 DestroyFile 的权限与校验缺口

[DestroyFile.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Services/DestroyFile.php#L20-L61) 中的校验逻辑：

```php
public function rules(): array
{
    return [
        'account_id' => 'required|uuid|exists:accounts,id',
        'vault_id'   => 'required|uuid|exists:vaults,id',
        'author_id'  => 'required|uuid|exists:users,id',
        'file_id'    => 'required|integer|exists:files,id',  // ← 只校验 file_id 存在
    ];
}

public function permissions(): array
{
    return [
        'author_must_belong_to_account',
        'vault_must_belong_to_account',
        'author_must_be_vault_editor',  // ← 只校验 Vault Editor 权限
    ];
}

private function validate(): void
{
    $this->validateRules($this->data);
    // 只校验文件属于该 vault，不校验 type
    $this->file = $this->vault->files()->findOrFail($this->data['file_id']);
}
```

**关键发现：DestroyFile Service 内部完全不知道自己被哪种场景调用**——它不接收、也不校验 `type` 字段。

### 19.3 具体越权场景

由于缺失 `type` 守门，可以构造以下越权请求：

| 场景 | 攻击方式 | 后果 |
| ---- | -------- | ---- |
| 跨类型删除（照片路由删头像） | 知道某头像文件 id=123，调用 `DELETE /vault/xxx/contact/yyy/photo/123` | 该联系人头像被删，而当前用户可能**没有编辑该联系人详情的权限**，只有查看照片模块的权限 |
| 跨类型删除（文档路由删照片） | 调用 `DELETE /vault/xxx/contact/yyy/document/456` 传的是照片文件 id=456 | 照片被删，同样不校验文件是否属于该文档模块 |
| Post 照片删 Contact 照片 | 调 `DELETE /vault/xxx/journal/1/post/2/photo/123`，传一个属于 Contact 的 TYPE_PHOTO 的 file_id=123 | Contact 的照片被删，请求来自 Journal Post 路由，但文件实际归属 Contact |

**根本原因**：四个路由从 RESTful 设计上看应该各自限定只能删除自己「归属」的文件，但控制器只是把 `file_id` 原样传给 DestroyFile，没有在 Service 层或 Controller 层做「文件归属 + 文件类型」的双重校验。

### 19.4 唯一的隐式限制

DestroyFile 的唯一过滤是：

```php
$this->file = $this->vault->files()->findOrFail($this->data['file_id']);
```

这确保文件属于该 Vault——但 Vault 是一个很宽的边界。同一个 Vault 下的所有联系人、所有 Journal 共享同一个文件池，V 编辑器可以看到全部，因此**跨联系人、跨 Journal 的删除是可能的**。

### 19.5 建议的修复方案

在 DestroyFile 中增加可选的 `type` 参数校验：

```php
public function rules(): array
{
    return [
        ...
        'file_id' => 'required|integer|exists:files,id',
        'expected_type' => 'nullable|string|in:avatar,photo,document', // 新增
    ];
}

private function validate(): void
{
    $this->validateRules($this->data);
    $query = $this->vault->files();
    if (isset($this->data['expected_type'])) {
        $query = $query->where('type', $this->data['expected_type']);
    }
    $this->file = $query->findOrFail($this->data['file_id']);
}
```

同时控制器调用时传入类型：

```php
// ContactModulePhotoController::destroy
(new DestroyFile)->execute([... , 'expected_type' => File::TYPE_PHOTO]);
```

---

## 20. 删除路径中 fileable_type 列的真相：仍在被使用（修正第 12 章结论）

第 12 章中曾得出「`fileable_id` 列在 Contact/Photo/Document 流程中始终为 NULL」的结论是**部分正确但不完整的**。结合 AddPhotoToPost 和 SetSliceOfLifeCoverImage 两个 Service，重新梳理：

### 20.1 fileable_type 在删除路径的实际使用

[DestroyFile::updateLastEditedDate()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Services/DestroyFile.php#L63-L69)

```php
private function updateLastEditedDate(): void
{
    // ← 这里的 fileable_type 绝对不可为 NULL
    if ($this->file->fileable_type == Contact::class) {
        // ← 这里用的是 ufileable 关系，但判断条件是 fileable_type！
        $this->file->ufileable->last_updated_at = Carbon::now();
        $this->file->ufileable->save();
    }
}
```

**关键点**：`$this->file->fileable_type == Contact::class` 这个判断必须依赖 `fileable_type` 列有正确的值。如果它为 NULL 或存了错误的值，`updateLastEditedDate` 会静默跳过（删 Contact 照片后不更新 last_updated_at），但不会报错。

### 20.2 两种主键类型的模型实际使用的列

| 模型 | 主键类型 | 写入的 ID 列 | fileable_type 值 | 写入位置 |
| ---- | -------- | ------------ | ---------------- | -------- |
| **Contact** | UUID (HasUuids) | `ufileable_id`（UUID） | `Contact::class` | 由 `Contact::files()->save($file)` 自动写入（Laravel MorphMany） |
| **Post** | INT (BIGINT auto_increment) | `fileable_id`（BIGINT） | `Post::class` | [AddPhotoToPost::execute()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Services/AddPhotoToPost.php#L53-L56) 手动赋值 |
| **SliceOfLife** | INT (BIGINT auto_increment) | `fileable_id`（BIGINT） | `SliceOfLife::class` | [SetSliceOfLifeCoverImage::execute()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Services/SetSliceOfLifeCoverImage.php#L56-L58) 手动赋值 |

**修正后的对照表**（替换第 12 章的表）：

| 模型 | 主键类型 | 使用哪套关联 | fileable_id | ufileable_id | fileable_type |
| ---- | -------- | ------------ | ----------- | ------------ | ------------- |
| Contact | UUID | 第二套 (ufileable) | NULL | UUID 值 | `App\Models\Contact` ✅ 有值 |
| Post | INT | 第一套 (fileable) | BIGINT 值 | NULL | `App\Models\Post` ✅ 有值 |
| SliceOfLife | INT | 第一套 (fileable) | BIGINT 值 | NULL | `App\Models\SliceOfLife` ✅ 有值 |

### 20.3 fileable_type 还有哪些读取方

| 读取方 | 用途 |
| ------ | ---- |
| DestroyFile::updateLastEditedDate() | 判断归属是否为 Contact，决定是否更新联系人的 `last_updated_at` |
| [VaultFileIndexViewHelper::getObjectDetails()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageFiles/Web/ViewHelpers/VaultFileIndexViewHelper.php#L79-L96) | Vault 文件管理页中，用 `match ($file->fileable_type)` 判断文件归属类型，展示跳转链接（Contact 展示对应联系人链接，其他类型返回空数组） |
| File::ufileable() 关系 | `morphTo(type: 'fileable_type')`——共享的类型列 |
| File::fileable() 关系 | `morphTo()`——默认使用类型列，对应 Post/SliceOfLife |

### 20.4 第 12 章潜在问题的修正

原第 12.5 节提到：
> `fileable()` 关系方法在当前业务中未被调用，属于迁移遗留

**此结论错误**。实际上，`fileable()` 关系虽然没有在 Contact 场景下被使用，但在 Post 和 SliceOfLife 场景下：
- 由 Service 层手动写入 `fileable_id` + `fileable_type`
- 对应的关联查询（如 `$post->files()`）在 `Post` 模型中必然定义为 `morphMany(File::class, 'fileable')`，即使用第一套关联

### 20.5 删除路径中的一个隐藏 bug

在 DestroyFile::updateLastEditedDate 中：

```php
if ($this->file->fileable_type == Contact::class) {
    $this->file->ufileable->last_updated_at = Carbon::now();
}
```

如果文件归属是 `Post::class` 或 `SliceOfLife::class`，**不会更新归属对象的 last_updated_at**——这是合理的，因为 Post 和 SliceOfLife 可能没有 `last_updated_at` 字段。但如果未来新增一个 Contact 以外的模型也使用 `ufileable_id`（UUID 主键 + 第二套关联），这里的判断条件 `== Contact::class` 就会漏掉，导致删除后不更新归属对象的时间戳。

更通用的写法应该是：判断是否存在 `ufileable` 关联对象，且该对象有 `last_updated_at` 字段，而不是硬编码比较 `Contact::class`。

---

## 21. 照片下载/展示 URL 的构造：直接 CDN URL vs 裁剪参数

### 21.1 两种 URL 模式的清晰划分

Monica 所有涉及图片的 ViewHelper 中，URL 严格分为两类：

- **`url.display`**：供前端 `<img>` 标签展示用，**拼接 CDN 裁剪/缩放参数**
- **`url.download`**：供用户下载原文件用，**直接用 `$file->cdn_url`，不做任何处理**

### 21.2 全场景 display URL 的裁剪参数一览

| 场景 | ViewHelper | 尺寸 | CDN 处理链 |
| ---- | ---------- | ---- | ---------- |
| 联系人头像（Attribute） | [Contact::avatar](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/Contact.php#L492-L497) | 300×300 | `scale_crop/300x300/smart` → `format/auto` → `quality/smart_retina` |
| 照片模块缩略图（Contact 模块） | [ModulePhotosViewHelper::dto](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/ViewHelpers/ModulePhotosViewHelper.php#L44-L46) | 300×300 | `scale_crop/300x300/smart` → `format/auto` → `quality/smart_retina` |
| 照片列表页网格图（Index） | [ContactPhotosIndexViewHelper::dto](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/ViewHelpers/ContactPhotosIndexViewHelper.php#L45-L47) | 400×400 | `scale_crop/400x400/smart` → `format/auto` → `quality/smart_retina` |
| 照片详情页大图（Show） | [ContactPhotosShowViewHelper::data](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/ViewHelpers/ContactPhotosShowViewHelper.php#L21-L23) | 1700px 宽 | `resize/1700x` → `format/auto` → `quality/smart_retina` |
| Post 编辑页缩略图 | [PostEditViewHelper::dtoPhoto](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/PostEditViewHelper.php#L186-L188) | 75×75 | `scale_crop/75x75/smart` → `format/auto` → `quality/smart_retina` |
| Post 展示页缩略图 | [PostShowViewHelper::getPhotos()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/PostShowViewHelper.php#L154-L156) | 100×100 | `scale_crop/100x100/smart` → `format/auto` → `quality/smart_retina` |
| Journal 列表页 Post 封面 | [JournalShowViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/JournalShowViewHelper.php#L87-L89) | 75×75 | `scale_crop/75x75/smart` → `format/auto` → `quality/smart_retina` |
| Journal 相册页索引 | [JournalPhotoIndexViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/JournalPhotoIndexViewHelper.php#L53-L55) | 200×200 | `scale_crop/200x200/smart` → `format/auto` → `quality/smart_retina` |
| Journal 列表页 Slice 封面 | [JournalShowViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/JournalShowViewHelper.php#L206-L208) | 200×100 | `scale_crop/200x100/smart` → `format/auto` → `quality/smart_retina` |
| Slice 详情页封面大图 | [SliceOfLifeShowViewHelper::dtoSlice](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/SliceOfLifeShowViewHelper.php#L79) | 800×100 | `scale_crop/800x100/smart` → `format/auto` → `quality/smart_retina` |

### 21.3 download URL 的统一模式

所有 ViewHelper 中 download URL 的构造完全一致：

| ViewHelper | 代码 |
| ---------- | ---- |
| [ModulePhotosViewHelper::dto](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/ViewHelpers/ModulePhotosViewHelper.php#L46) | `'download' => $file->cdn_url` |
| [ModuleDocumentsViewHelper::dto](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Web/ViewHelpers/ModuleDocumentsViewHelper.php#L43) | `'download' => $file->cdn_url` |
| [ContactPhotosIndexViewHelper::dto](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/ViewHelpers/ContactPhotosIndexViewHelper.php#L47) | `'download' => $file->cdn_url` |
| [ContactPhotosShowViewHelper::data](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/ViewHelpers/ContactPhotosShowViewHelper.php#L22) | `'download' => $file->cdn_url` |
| [VaultFileIndexViewHelper::data](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageFiles/Web/ViewHelpers/VaultFileIndexViewHelper.php#L26) | `'download' => $file->cdn_url` |

### 21.4 `$file->cdn_url` 从何而来

`cdn_url` 字段是**前端上传时传过来的**，不是后端自己拼的：

- 前端 Uploadcare Widget 上传成功后，`file.cdnUrl` 属性（通常形如 `https://ucarecdn.com/{uuid}/`）被 axios PUT/POST 到后端
- [UploadFile::save()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Services/UploadFile.php#L77-L88) 原样存入 `files.cdn_url` 列

所以 download URL 实际上就是**用户上传的原始文件在 CDN 上的根 URL**，不含任何图像转换参数。

### 21.5 两种 URL 模式的设计意图

| | display URL | download URL |
| - | ----------- | ------------ |
| **目的** | 给 `<img>` 标签快速加载缩略图 | 给用户下载原始文件 |
| **体积** | 小（300-1700px 且格式压缩，通常几十 KB） | 大（原始分辨率，可能几 MB 到几十 MB） |
| **格式** | 自动选择 WebP/AVIF | 保留用户上传的原始格式 |
| **质量损耗** | 有（smart_retina 级别压缩） | 无（完全无损） |
| **生成位置** | ViewHelper 实时拼接字符串 | 前端上传时写入数据库，后端原样返回 |
| **代码重复度** | 10 处硬编码，每次拼接 3 段相同参数（`scale_crop/...` + `format/auto` + `quality/smart_retina`） | 完全一致，无重复 |

### 21.6 设计改进建议

`format/auto` + `quality/smart_retina` 这两段在 10 处 display URL 中重复出现，建议抽到 `FileHelper` 静态方法中：

```php
public static function displayUrl(string $uuid, string $size = '300x300'): string
{
    return 'https://ucarecdn.com/'.$uuid.'/-/scale_crop/'.$size.'/smart/-/format/auto/-/quality/smart_retina/';
}
```

---

## 22. Uploadcare 签名直接嵌入 HTML 的暴露风险

### 22.1 签名如何到达用户浏览器

完整链路：

```
StorageHelper::uploadcare()
  ├─► new Signature($privateKey)         ← PHP 内计算签名
  ├─► getSignature() → HMAC-SHA256 字符串
  └─► getExpire() → Unix 时间戳
       │
       ▼
ContactShowViewHelper / ModulePhotosViewHelper / ...
  └─► 'uploadcare' => StorageHelper::uploadcare()   ← 放入返回数组
       │
       ▼
Inertia::render() → 序列化为 JSON
  └─► 注入到 HTML 页面的 <div id="app" data-page="{...}"> 属性
       │
       ▼
浏览器接收 HTML → Vue/Inertia 解析 data-page JSON
  └─► props.data.avatar.uploadcare.signature 可用
       │
       ▼
Uploadcare.vue 中
  └─► secureSignature, secureExpire 传入 uploadcare.openDialog()
```

**签名以明文形式出现在 HTML 源码中**。任何打开 DevTools、查看页面源码、抓 HTTP 包（或使用中间人攻击读取明文 HTTP）的人都能直接看到这 64 个十六进制字符的 HMAC 签名。

### 22.2 签名泄露后能做什么

结合第 15 章对签名算法的分析，当前签名设计的特点是：
- ✅ 不绑定私钥（不暴露私钥本身）
- ✅ 有时效性（最多 1 小时）
- ❌ **不绑定文件**：同一签名可用于任意文件的上传
- ❌ **不绑定用户**：拿到签名的任何人都能用
- ❌ **不绑定 IP**：跨网络重放有效

具体攻击场景：

| 场景 | 可行性 | 效果 |
| ---- | ------ | ---- |
| **跨用户上传** | 完全可行 | 同一 Vault 的另一个合法用户 B 查看页面得到签名后，可以用自己的程序直接上传文件到 Monica 的 Uploadcare 项目，再通过构造合法请求写入 Contact A 的照片列表——只要 B 有 Vault Editor 权限，整个流程完全合法 |
| **签名截获重放** | 在 HTTP 明文传输下可行（通常 Monica 是 HTTPS，降低风险） | 攻击者在有效期内用截获的签名上传任意内容 |
| **上传非法内容** | 完全可行 | 签名不约束文件类型/内容，攻击者可以上传超大文件、恶意软件、色情内容到 Monica 的 Uploadcare 项目空间，产生账单风险和法律风险 |
| **签名滥用上传后不入库** | 完全可行 | 攻击者直接用签名向 Uploadcare 上传文件，不回调 Monica 后端——CDN 文件存在但 Monica 无记录，产生无主的存储消耗 |

### 22.3 风险的前提条件与边界

| 前提 | 是否通常满足 |
| ---- | ------------ |
| Monica 站点有多个信任边界不同的用户（如家庭共享、团队使用） | 取决于部署方式 |
| 站点通过 HTTPS 服务 | 是（任何现代部署都是 HTTPS） |
| 用户使用公共电脑 / 共享浏览器 Session | 低概率 |
| Uploadcare 账户有存储上限或按用量计费 | 是（商业 SaaS） |
| 站点对外公开注册 | 通常不是（Monica 是自部署私密 CRM） |

**综合评价**：在典型自部署场景（单用户/家庭使用 + HTTPS）下，此风险等级为**低**。但在多租户 SaaS 部署或团队共享部署场景下，风险等级提升为**中**。

### 22.4 为什么不能把签名放在上传请求时动态获取

直觉上，用户点击上传按钮时再 AJAX 请求一个签名会更安全——但这实际上和嵌入 HTML 在风险上等价，因为用户点击上传按钮的时刻同样是在浏览器中，同样能被 DevTools 截获。签名生成必须在服务端完成，而任何传送到浏览器的数据本质上都是用户可读取的。

**真正有效的加固手段**（而非隐藏签名）：

| 方案 | 效果 |
| ---- | ---- |
| **缩短 TTL** | 把默认 3600 秒改为 300-600 秒。上传操作通常在几十秒内完成，1 小时过于宽松 |
| **Uploadcare 后端 Webhook 校验** | 在 Uploadcare Dashboard 配置文件上传 Webhook，Monica 收到后校验上传来源，删除未通过 Monica 后端注册的「无主文件」 |
| **上传完成后后端再校验 type/mime/size** | UploadFile Service 中增加规则（如第 14 章建议的），即使签名被滥用上传非法文件，也写不进 Monica 数据库 |
| **上传速率限制** | 对 `/contact/*/store` 和 `/avatar/update` 等上传入库路由加 Throttle 中间件 |
| **Uploadcare 项目层面加限制** | 在 Uploadcare Dashboard 配置文件大小上限、允许的 MIME 类型白名单——在 CDN 侧第一层拦截 |

### 22.5 与典型 CSRF Token 的安全性对比

Monica 的 Uploadcare 签名和 Laravel 的 CSRF Token 在暴露形式上类似（都嵌入 HTML），但本质不同：

| | Laravel CSRF Token | Uploadcare 签名 |
| - | ------------------ | --------------- |
| **生命周期** | 整个 Session（2 小时或更长） | 最多 1 小时（SDK MAX_TTL） |
| **每次刷新是否更新** | 同一 Session 内不更新 | ✅ 每次页面刷新重新生成 |
| **泄露后能做什么** | 在当前 Session 内伪造表单提交 | 有效期内上传任意文件到 Uploadcare 项目 |
| **是否绑定用户/会话** | ✅ 绑定当前 Session | ❌ 不绑定 |
| **服务端校验** | Laravel 每个非 GET 请求强制校验 | Uploadcare 上传端点校验 |

值得注意的是 Uploadcare 签名**每次页面刷新都重新生成**，所以同一用户多次打开页面会得到多个不同且都有效的签名。签名过期的时间点是各自独立计算的（`now + ttl`），这会让有效签名数量随页面访问次数线性增长。

---

## 23. Slice 封面上传：写入老 fileable_id 列但端模型未定义对应关系

### 23.1 SetSliceOfLifeCoverImage 如何写入关联

[SetSliceOfLifeCoverImage::execute()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Services/SetSliceOfLifeCoverImage.php#L48-L61)

```php
$this->slice->file_cover_image_id = $this->file->id;  // ← 1. 写入 SliceOfLife 自身的外键
$this->slice->save();

$this->file->fileable_id = $this->slice->id;          // ← 2. 写入 fileable 多态关联（INT 主键那套）
$this->file->fileable_type = SliceOfLife::class;
$this->file->save();
```

同时维护了两套关联：
- **正向 BelongsTo**：`slice_of_lives.file_cover_image_id` → `files.id`（由 [SliceOfLife::file()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/SliceOfLife.php#L52-L55) 定义，指定键名 `file_cover_image_id`）
- **反向 MorphMany**：`files.fileable_id = slice.id` + `files.fileable_type = SliceOfLife::class`（使用 INT 主键那套老关联）

### 23.2 但 SliceOfLife 模型**未定义反向 morphMany 关系**

对比 [Post::files()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/Post.php#L112-L115)：

```php
// Post 模型有定义
public function files(): MorphMany
{
    return $this->morphMany(File::class, 'fileable');
}
```

而 [SliceOfLife 模型](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Models/SliceOfLife.php#L10-L56) 的全部关系只有 `journal()`、`posts()`、`file()` ——**没有 `files()` 方法**。

这意味着：

| 操作 | Post（有定义） | SliceOfLife（无定义） |
| ---- | ------------- | -------------------- |
| `$model->files` 动态属性读取 | ✅ 返回该 Post 的全部照片 | ❌ 报错：`Undefined property` |
| `$model->files()->save($file)` | ✅ 自动写入 fileable_id/type | ❌ 报错：`Call to undefined method` |
| `$model->files()->where(...)` 查询 | ✅ 可以按 Post 过滤照片 | ❌ 无法使用 |
| 通过 `File::where('fileable_type', SliceOfLife::class)->where('fileable_id', $sliceId)` 查询 | — | ✅ 手写 SQL 可以查到 |

### 23.3 为何业务没有因此报错

SetSliceOfLifeCoverImage 和 AddPhotoToPost 都**绕过了 Eloquent 关系**，直接手动写入 `$file->fileable_id` 和 `$file->fileable_type`，而不是调用 `$slice->files()->save($file)`。所以缺失关系定义在写入时不会暴露问题。

但查询时，以下代码都**没有使用** `$slice->files`：
- 展示封面用 `$slice->file`（BelongsTo，存在）
- 删除封面用 `$slice->file->delete()`（同上）
- Vault 文件管理页用 `File::where('vault_id', ...)` 全局查询

因此这个缺失只影响「列出某个 Slice 的全部封面历史」这类需求，但由于 Slice 只有一张当前封面（`file_cover_image_id` 一对一），历史封面在业务上其实不被查询，所以 bug 被掩盖。

### 23.4 修复建议

在 SliceOfLife 模型中补加：

```php
public function files(): MorphMany
{
    return $this->morphMany(File::class, 'fileable');
}
```

虽然当前业务不直接使用，但这是与 Post 保持一致的正确模型定义，可避免后续开发踩坑。

---

## 24. files.original_url：强校验写入但读取方为零

### 24.1 写入路径

所有 5 个上传入口（Avatar / Contact Photo / Contact Document / Post Photo / Slice Cover）都要求前端传入 `original_url`：

| 上传场景 | Controller |
| -------- | ---------- |
| 头像 | [ModuleAvatarController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageAvatar/Web/Controllers/ModuleAvatarController.php#L24) |
| 联系人照片 | [ContactModulePhotoController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManagePhotos/Web/Controllers/ContactModulePhotoController.php#L24) |
| 联系人文档 | [ContactModuleDocumentController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Web/Controllers/ContactModuleDocumentController.php#L24) |
| Post 照片 | [PostPhotoController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Web/Controllers/PostPhotoController.php#L24) |
| Slice 封面 | [SliceOfLifeCoverImageController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Web/Controllers/SliceOfLifeCoverImageController.php#L25) |

[UploadFile::rules()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Services/UploadFile.php#L26) 强制校验：

```php
'original_url' => 'required|string',   // ← required，不可省略
'cdn_url'      => 'required|string',
```

前端 [Uploadcare.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/resources/js/Components/Uploadcare.vue) 的 `onSuccess` 回调中，把 `file.originalUrl` 和 `file.cdnUrl` 都传给后端。

### 24.2 全局搜索：original_url 的读取方

对 `app/` 目录和 `resources/js/` 目录做全局搜索：

- PHP 端：**8 处** —— 全部是写入（`$request->input('original_url')` 或 `fillable` 声明或 `rules()`），**零处读取**
- JS 端：**6 处** —— 全部是写入 form（`this.form.original_url = file.originalUrl`），**零处消费**

读取到 `$file->original_url` 的 ViewHelper / Controller / Service 数量 = **0**。

对比之下，`cdn_url` 有 **5 处读取**（全部是 `'download' => $file->cdn_url`）。

### 24.3 original_url 和 cdn_url 的区别（Uploadcare 语义）

根据 Uploadcare SDK 语义：
- `file.originalUrl`：用户上传前本地文件的原始路径（浏览器中通常是 `blob:` URL 或临时对象 URL），**不保证可公网访问**
- `file.cdnUrl`：文件上传到 Uploadcare CDN 后的正式 URL，形如 `https://ucarecdn.com/{uuid}/`，**可永久公网访问**

也就是说，`original_url` 是**前端浏览器本地的临时 URL**，传到后端存进数据库完全没有意义——后端和其他用户根本无法访问这个本地 blob URL。

### 24.4 问题总结

| 问题 | 说明 |
| ---- | ---- |
| **存储浪费** | 每条 File 记录多存一个无用字符串字段（通常是几十到几百字节的 blob URL） |
| **校验冗余** | `required\|string` 校验强制前端必须传一个无用字段 |
| **可能的误解** | 后续开发者看到此字段可能误以为可以用它访问原始文件，实际却不行 |
| **与 cdn_url 功能重叠** | Uploadcare 真正可用的下载 URL 是 cdn_url，original_url 在后端无任何用途 |

### 24.5 修复建议

- 短期：从 UploadFile `rules()` 中去掉 `required`，改为 `nullable|string`，避免未来重构时前端不传就报错
- 长期：新建迁移移除 `files.original_url` 列，同时移除 Controller 中的 `$request->input('original_url')` 和前端 form 中的 `original_url` 字段

---

## 25. Slice 封面与联系人照片的隔离：共用 TYPE_PHOTO 标签，仅靠 fileable_type 区分

### 25.1 Slice 封面的 type 标签

[SliceOfLifeCoverImageController::update()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Web/Controllers/SliceOfLifeCoverImageController.php#L29)：

```php
$data = [
    // ...
    'type' => File::TYPE_PHOTO,   // ← 与联系人照片完全相同的常量值 'photo'
];
$file = (new UploadFile)->execute($data);
```

没有 `TYPE_COVER` 或 `TYPE_SLICE_COVER` 之类的独立常量，Slice 封面直接复用 `TYPE_PHOTO`。

### 25.2 仅靠两层隔离区分

| 隔离层 | 联系人照片 | Slice 封面 | Post 内嵌照片 |
| ------ | ---------- | ---------- | ------------- |
| 第一层：`files.type` | `'photo'` | `'photo'` | `'photo'` |
| 第二层：`files.fileable_type` | `Contact::class` (UUID 走 ufileable_id) | `SliceOfLife::class` (INT 走 fileable_id) | `Post::class` (INT 走 fileable_id) |
| 第三层：外键 | `contacts.file_id` (仅头像) + `ufileable_id` | `slice_of_lives.file_cover_image_id` + `fileable_id` | `fileable_id` |

### 25.3 隔离不充分带来的问题

**问题 1：Vault 文件管理页混在一起，无法分辨**

[VaultFileController::photos()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageFiles/Web/Controllers/VaultFileController.php#L34-L49) 和 `avatars()` / `documents()` 的查询方式：

```php
// 仅按 type 过滤
$files = File::where('vault_id', $vaultId)
    ->where('type', File::TYPE_PHOTO)   // ← 只看 type=photo
    ->orderBy('created_at', 'desc')
    ->paginate(25);
```

这意味着**联系人照片、Post 内嵌照片、Slice 封面全部出现在同一个「照片」Tab 下**，用户在 Vault 文件管理页中完全看不出这张照片是属于某个 Contact 还是某个 Slice。

[VaultFileIndexViewHelper::getObjectDetails()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageFiles/Web/ViewHelpers/VaultFileIndexViewHelper.php#L79-L96) 用 match 处理归属：

```php
return match ($file->fileable_type) {
    Contact::class => [ 'type' => 'contact', ... ], // 仅 Contact 有处理
    default => [],   // ← Post 和 SliceOfLife 的文件都返回空数组，显示成「未知归属」
};
```

**问题 2：DestroyFile 的 updateLastEditedDate 只处理 Contact**

[DestroyFile::updateLastEditedDate()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Contact/ManageDocuments/Services/DestroyFile.php#L63-L69)

```php
if ($this->file->fileable_type == Contact::class) {
    $this->file->ufileable->last_updated_at = Carbon::now();
    $this->file->ufileable->save();
}
```

删除 Slice 封面或 Post 照片时，**归属对象的时间戳完全不更新**——既不检查 SliceOfLife 也不检查 Post，即使它们可能有 `updated_at` 字段。

**问题 3：DestroyFile 不区分 type，任何路由可以删除任意 type**

第 19 章已分析：4 个控制器共用 DestroyFile，Service 内部不校验 type。因此用联系人照片的删除路由可以删掉一张 Slice 封面，只要它在同 Vault 下。

### 25.4 建议的隔离方案

| 方案 | 做法 | 优劣 |
| ---- | ---- | ---- |
| A. 细化 File::type 常量 | 新增 `TYPE_COVER = 'cover'` 或 `TYPE_SLICE_COVER`，Slice 封面写入时用新 type | 清晰；但需同步修改所有过滤查询和 Vault 文件管理页的 Tab |
| B. 保留 type=photo，查询时同时按 fileable_type 过滤 | 联系人照片查询加 `where('fileable_type', Contact::class)` | 无需迁移；但每个查询点都要补 where 条件 |
| C. DestroyFile 中 expected_type 校验同时校验 fileable_type | Service 层新增 `expected_fileable_type` 参数 | 解决跨归属删除越权；但不解决展示层混合 |

---

## 26. Slice 封面误删后 SliceOfLife.file_cover_image_id 外键的级联效果

### 26.1 迁移中的外键约束

[2022_12_15_004442_create_slices_of_life_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/database/migrations/2022_12_15_004442_create_slices_of_life_table.php#L22)：

```php
$table->foreignIdFor(File::class, 'file_cover_image_id')
    ->nullable()
    ->constrained('files')
    ->nullOnDelete();   // ← 关键：文件被删时，自动把此外键设为 NULL
```

`nullOnDelete()` 是 Laravel 对外键 `ON DELETE SET NULL` 的封装。

### 26.2 两条删除路径的对比

**路径 A：正常移除（通过 RemoveSliceOfLifeCoverImage Service）**

[RemoveSliceOfLifeCoverImage::execute()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Services/RemoveSliceOfLifeCoverImage.php#L44-L57)：

```php
if ($this->slice->file) {
    $this->slice->file->delete();      // ① 先删 File 记录
                                       // → DB 级外键 nullOnDelete 自动把
                                       //   slice_of_lives.file_cover_image_id 置为 NULL
                                       // → 同时触发 FileDeleted 事件 → DeleteFileInStorage
                                       //   → 调 Uploadcare API 删 CDN 文件
}

$this->slice->file_cover_image_id = null;  // ② 再显式写一次 NULL（重复但无害）
$this->slice->save();
```

注意这里有一个**重复操作**：第 ① 步删 File 后，外键 `nullOnDelete` 已经把 DB 中的 `file_cover_image_id` 置为 NULL；第 ② 步再写一次是冗余的，但不会出错。

**路径 B：误删（通过 DestroyFile / Vault 文件管理删了封面文件）**

假设用户在 Vault 文件管理页（[VaultFileController::destroy()](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageFiles/Web/Controllers/VaultFileController.php#L85-L99)）看到了某张封面照片，直接点删除：

```
DELETE FROM files WHERE id = {coverFileId}
        │
        ▼  外键 ON DELETE SET NULL
slice_of_lives.file_cover_image_id = NULL
        │
        ▼  Eloquent deleted 事件
FileDeleted 事件 → DeleteFileInStorage → Uploadcare API → CDN 文件删除
        │
        ▼  DestroyFile 的 updateLastEditedDate
fileable_type == SliceOfLife::class != Contact::class → 不更新任何时间戳
```

结果：
- ✅ 数据库完整性：外键自动 `NULL`，不会出现悬垂指针
- ✅ CDN 文件同步删除：DeleteFileInStorage Listener 正常工作
- ✅ Slice 再次读取时 `$slice->file` 返回 null，ViewHelper 已做 `optional()` 处理（[SliceOfLifeShowViewHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/71-monica/app/Domains/Vault/ManageJournals/Web/ViewHelpers/SliceOfLifeShowViewHelper.php#L79) 写的是 `$slice->file ? ... : null`）
- ❌ SliceOfLife 的 `updated_at` 不更新（DestroyFile 只处理 Contact）
- ❌ 没有写动态流或日志，用户可能不知道封面为什么不见了

### 26.3 与联系人头像删除的对比

| | 头像删除（DestroyAvatar Service） | Slice 封面误删（DestroyFile → DB 级联） |
| - | -------------------------------- | --------------------------------------- |
| 外键处理 | Service 显式 `$contact->file_id = null` + `save()` | DB 级 `nullOnDelete` 自动处理 |
| 是否触发 FileDeleted 事件 | ✅ 是（`$this->contact->file->delete()`） | ✅ 是（`$this->file->delete()`） |
| CDN 文件是否同步删除 | ✅ 是 | ✅ 是 |
| Contact/Slice 时间戳 | ✅ `updateLastEditedDate()` 写 `last_updated_at` | ❌ 不更新 |
| 动态流（Feed Item） | ✅ 写 `ACTION_CHANGE_AVATAR` | ❌ 没有 |
| 是否可以从任意路由触发 | ❌ 只有 `contact.avatar.destroy` 调 DestroyAvatar | ✅ 4 个 DestroyFile 入口都可以删 |

### 26.4 隐藏问题：RemoveSliceOfLifeCoverImage 的显式 NULL 可能在事务回滚时不一致

RemoveSliceOfLifeCoverImage 中：
1. `$this->slice->file->delete()` 触发 `deleted` 事件 → Uploadcare REST API（HTTP，不在事务内）
2. `$this->slice->file_cover_image_id = null` → `save()`（DB 写入）

如果第 ② 步 DB 写入失败（如死锁、连接断开），会出现：
- CDN 文件已被删（HTTP 请求已完成，无法回滚）
- 但 `slice_of_lives.file_cover_image_id` 仍指向已删除文件的 id —— **除非外键 nullOnDelete 在同一 DB 事务内自动生效**

实际上第 ① 步 Eloquent `delete()` 执行的 SQL `DELETE FROM files` 和第 ② 步 `UPDATE slice_of_lives` 在同一个请求中通常共享同一个 DB 连接，`nullOnDelete` 的 `ON DELETE SET NULL` 是 MySQL 内部机制，与 DELETE 在同一事务中自动完成——所以 `file_cover_image_id` 在 DELETE 后立即就是 NULL，第 ② 步只是覆盖写 NULL，失败不会导致不一致。

但这不是一个稳健的模式：**Service 应该只维护一套语义，不要同时依赖 DB 级联和显式赋值。**
