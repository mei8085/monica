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
