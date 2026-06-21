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
