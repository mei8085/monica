# 实例全局设置读写回路分析

## 一、设置分层架构

Monica 项目中的设置体系分为四个层级，从高到低分别是：

### 1. 应用/实例级配置
- **存储位置**：`.env` 文件 + `config/` 目录
- **典型配置**：
  - `disable_signup` - 是否禁用注册
  - `default_storage_limit_in_mb` - 默认存储限制
  - `mapbox_api_key` - Mapbox API 密钥
  - `location_iq_api_key` - 地理位置服务密钥
- **特点**：影响整个实例部署，不能通过 Web 界面修改，需要通过环境变量配置
- **代码位置**：[monica.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/config/monica.php)

### 2. 账户级设置
- **存储位置**：数据库多张表
- **核心表**：`accounts` 表 - 存储账户基本信息（如 `storage_limit_in_mb`）
- **关联表**：
  - `genders` - 性别选项
  - `pronouns` - 代词选项
  - `relationship_types` - 关系类型
  - `address_types` - 地址类型
  - `templates` / `template_pages` / `modules` - 联系人模板
  - 等等...
- **特点**：每个 Account（账户/租户）有独立的设置，由账户管理员管理
- **代码位置**：[Account.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/Account.php)

### 3. 用户级偏好
- **存储位置**：`users` 表
- **主要设置项**：
  - `name_order` - 姓名显示顺序
  - `date_format` - 日期格式
  - `timezone` - 时区
  - `number_format` - 数字格式
  - `distance_format` - 距离单位
  - `default_map_site` - 默认地图服务
  - `locale` - 语言
  - `help_shown` - 是否显示帮助
  - `contact_sort_order` - 联系人排序
- **特点**：每个用户可自定义，存储在 `users` 表的对应字段中
- **代码位置**：[User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/User.php#L80-L103)

### 4. Vault 级设置
- **存储位置**：数据库 Vault 相关表
- **主要设置**：标签、标签、心情追踪参数、人生事件分类等
- **特点**：每个保险库（Vault）有独立的配置

---

## 二、持久层（数据库）设计

### 核心表结构

#### accounts 表
- 字段：`id` (uuid), `storage_limit_in_mb` (integer), `timestamps`
- 迁移文件：[2013_04_25_132851_create_accounts_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/database/migrations/2013_04_25_132851_create_accounts_table.php)

#### users 表（偏好设置字段）
- 迁移文件：[2014_10_12_000000_create_users_table.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/database/migrations/2014_10_12_000000_create_users_table.php)
- 关键字段：
  - `name_order` - 默认 '%first_name% %last_name%'
  - `date_format` - 默认 'MMM DD, YYYY'
  - `timezone` - 可空
  - `number_format` - 默认 'locale'
  - `default_map_site` - 默认 'open_street_maps'
  - `distance_format` - 默认 'mi'
  - `is_account_administrator` - 是否账户管理员
  - `is_instance_administrator` - 是否实例管理员
  - `locale` - 默认 'en'
  - `help_shown` - 默认 true

---

## 三、运行时缓存机制

### 缓存驱动
使用 `Cache::store('array')` - **请求级内存缓存**，只在单次请求生命周期内有效。

### 缓存应用场景

#### 1. Vault 权限缓存
- **位置**：[VaultHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/VaultHelper.php#L40-L53)
- **缓存键**：`Permission:{userId}:{vaultId}`
- **过期时间**：5 秒
- **作用**：缓存用户在指定 Vault 中的权限等级，避免重复查询

#### 2. 用户关联联系人信息缓存
- **位置**：[UserHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/UserHelper.php#L15-L38)
- **缓存键**：`InformationAboutContact:{userId}:{vaultId}`
- **过期时间**：5 秒
- **作用**：缓存用户在 Vault 中对应的联系人信息

#### 3. 重要日期类型缓存
- **位置**：[ContactImportantDateHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/ContactImportantDateHelper.php#L13-L21)
- **缓存键**：`ImportantDateType:{vaultId}:{type}`
- **过期时间**：5 秒
- **作用**：缓存 Vault 中的重要日期类型

### 缓存特点
- 全部使用 `array` 驱动，请求结束后缓存自动清除
- 过期时间统一为 5 秒，主要用于防止单次请求内的重复查询
- 采用 `remember()` 方法，自动处理缓存读写

---

## 四、读取流程（从数据库到界面）

以**用户日期格式偏好**为例，完整的读取链路：

### 1. 路由层
- **路由定义**：`settings.preferences.index` → `PreferencesController@index`
- **路由文件**：[web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/routes/web.php)

### 2. 控制层
- **控制器**：[PreferencesController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesController.php)
- **核心逻辑**：
  ```php
  public function index()
  {
      return Inertia::render('Settings/Preferences/Index', [
          'layoutData' => VaultIndexViewHelper::layoutData(),
          'data' => UserPreferencesIndexViewHelper::data(Auth::user()),
      ]);
  }
  ```

### 3. 视图模型层（ViewHelper）
- **类**：[UserPreferencesIndexViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/ViewHelpers/UserPreferencesIndexViewHelper.php)
- **作用**：将模型数据转换为前端需要的 DTO 格式
- **日期格式相关方法**：`dtoDateFormat(User $user)`
  - 生成可选的日期格式列表
  - 包含当前用户的设置值
  - 提供后端 API 的 URL

### 4. 前端展示层
- **主页面**：[Index.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/resources/js/Pages/Settings/Preferences/Index.vue)
- **日期格式组件**：[DateFormat.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/resources/js/Pages/Settings/Preferences/Partials/DateFormat.vue)
- **数据流向**：
  1. Inertia 将后端数据注入到页面 props
  2. 父组件将 `data.date_format` 传递给子组件
  3. 子组件在 `mounted()` 中将 props 数据同步到本地 data
  4. 组件根据 `editMode` 切换显示/编辑模式

### 读取流程图

```
用户访问页面
    ↓
路由匹配 → PreferencesController@index
    ↓
Auth::user() 获取当前用户模型
    ↓
UserPreferencesIndexViewHelper::data($user)
    ├─ 从 User 模型读取各字段值
    └─ 组装成 DTO（包含 URL、选项列表等）
    ↓
Inertia::render() 传递给前端
    ↓
Vue 组件接收 props
    ↓
mounted() 同步到本地 data
    ↓
渲染显示
```

---

## 五、写入流程（从界面到数据库）

以**用户日期格式偏好**为例，完整的写入链路：

### 1. 前端交互
- **组件**：[DateFormat.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/resources/js/Pages/Settings/Preferences/Partials/DateFormat.vue#L124-L140)
- **交互流程**：
  1. 用户点击 "Edit" 按钮 → 进入编辑模式
  2. 用户选择新的日期格式（单选框）
  3. 用户点击 "Save" 按钮 → 触发 `submit()` 方法
  4. 使用 axios 发送 POST 请求到 `data.url.store`

### 2. 路由层
- **路由**：`settings.preferences.date.store` → `PreferencesDateFormatController@store`

### 3. 控制层
- **控制器**：[PreferencesDateFormatController.php](file:///d:/fz/0601-2\solo-dogfeeding\code\35-monica\app\Domains\Settings\ManageUserPreferences\Web\Controllers\PreferencesDateFormatController.php)
- **核心逻辑**：
  ```php
  public function store(Request $request)
  {
      $data = [
          'account_id' => Auth::user()->account_id,
          'author_id' => Auth::id(),
          'date_format' => $request->input('dateFormat'),
      ];

      $user = (new StoreDateFormatPreference)->execute($data);

      return response()->json([
          'data' => UserPreferencesIndexViewHelper::dtoDateFormat($user),
      ], 200);
  }
  ```

### 4. 服务层（Service）
- **服务类**：[StoreDateFormatPreference.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreDateFormatPreference.php)
- **基类**：[BaseService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Services/BaseService.php)
- **接口**：`ServiceInterface`

#### 服务执行流程：
1. **验证规则** (`validateRules()`)
   - 检查 `account_id`, `author_id`, `date_format` 等字段
   - 使用 Laravel Validator 进行数据校验

2. **验证权限** (`validatePermission()`)
   - 权限声明：`author_must_belong_to_account`
   - 验证用户是否属于指定账户
   - 自动查询并设置 `$this->author` 用户对象

3. **执行业务逻辑**
   - 更新 `$this->author->date_format`
   - 调用 `$this->author->save()` 保存到数据库

4. **返回结果**
   - 返回更新后的 User 模型

### 5. 响应层
- Controller 调用 `UserPreferencesIndexViewHelper::dtoDateFormat($user)` 重新组装 DTO
- 返回 JSON 响应给前端

### 6. 前端更新
- axios `.then()` 回调中：
  1. 显示成功提示（flash 消息）
  2. 更新本地状态（`localDateFormat`, `localHumanDateFormat`）
  3. 退出编辑模式
  4. 重置加载状态

### 写入流程图

```
用户点击 Save
    ↓
前端表单验证
    ↓
axios.post(url, formData)
    ↓
路由 → PreferencesDateFormatController@store
    ↓
构造 service 数据（account_id, author_id, ...）
    ↓
StoreDateFormatPreference->execute($data)
    ├─ validateRules() - 字段验证
    ├─ validatePermission() - 权限验证（查询 User）
    ├─ updateUser() - 更新模型属性
    └─ save() - 写入数据库
    ↓
返回 User 模型
    ↓
UserPreferencesIndexViewHelper::dtoDateFormat() 组装 DTO
    ↓
返回 JSON 响应
    ↓
前端 .then() 回调
    ├─ 显示成功消息
    ├─ 更新本地状态
    ├─ 退出编辑模式
    └─ 重置加载状态
```

---

## 六、服务层架构详解

### BaseService 基类
所有设置服务都继承自 [BaseService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Services/BaseService.php)，提供统一的验证框架。

### 权限依赖图
```
author_must_belong_to_account
    ↑
author_must_be_account_administrator

vault_must_belong_to_account
    ↑
author_must_be_vault_manager
author_must_be_vault_editor
author_must_be_in_vault
    ↑
contact_must_belong_to_vault
group_must_belong_to_vault
```

### 服务类命名规范
- 创建：`Create{Entity}.php`
- 更新：`Update{Entity}.php`
- 删除：`Destroy{Entity}.php`
- 其他操作：`{Action}{Entity}.php`

---

## 七、账户初始化（默认设置）

新账户创建时，会通过 [SetupAccount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/CreateAccount/Jobs/SetupAccount.php) 任务初始化所有默认设置。

### 初始化内容
1. **货币** - 关联所有可用货币
2. **通知渠道** - 创建默认邮件通知渠道
3. **模板** - 创建默认联系人模板及页面
4. **模块** - 创建各类模块（头像、姓名、联系方式等）
5. **性别** - 男/女/其他
6. **代词** - he/him, she/her, they/them 等
7. **组类型** - 家庭、情侣、俱乐部等
8. **关系类型** - 家人、朋友、同事等
9. **地址类型** - 家、工作、其他等
10. **通话原因类型** - 个人、商务
11. **联系方式类型** - 邮件、电话、社交网络
12. **宠物分类** - 狗、猫、鸟等
13. **情绪** - 负面、中性、正面
14. **礼物场合** - 生日、纪念日、圣诞节等
15. **礼物状态** - 想法、已购买、已送出等
16. **帖子模板** - 普通帖子、励志帖子
17. **宗教** - 基督教、伊斯兰教、佛教等

### 创建入口
- 服务类：[CreateAccount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/CreateAccount/Services/CreateAccount.php)
- 队列任务：`SetupAccount::dispatch($request)->onQueue('high')`

---

## 八、关键代码索引

### 配置文件
- [config/monica.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/config/monica.php) - 应用级配置

### 模型层
- [app/Models/Account.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/Account.php) - 账户模型
- [app/Models/User.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/User.php) - 用户模型（含偏好字段）
- [app/Models/Instance/Cron.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Models/Instance/Cron.php) - 实例级定时任务

### 服务基类
- [app/Services/BaseService.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Services/BaseService.php) - 服务基类

### 缓存辅助类
- [app/Helpers/VaultHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/VaultHelper.php) - Vault 权限缓存
- [app/Helpers/UserHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/UserHelper.php) - 用户信息缓存
- [app/Helpers/ContactImportantDateHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Helpers/ContactImportantDateHelper.php) - 日期类型缓存

### 设置控制器（用户偏好）
- [app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesController.php)
- [app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesDateFormatController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/Controllers/PreferencesDateFormatController.php)

### 设置服务（用户偏好）
- [app/Domains/Settings/ManageUserPreferences/Services/StoreDateFormatPreference.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Services/StoreDateFormatPreference.php)

### 视图模型
- [app/Domains/Settings/ManageUserPreferences/Web/ViewHelpers/UserPreferencesIndexViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageUserPreferences/Web/ViewHelpers/UserPreferencesIndexViewHelper.php)

### 前端组件
- [resources/js/Pages/Settings/Index.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/resources/js/Pages/Settings/Index.vue) - 设置首页
- [resources/js/Pages/Settings/Preferences/Index.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/resources/js/Pages/Settings/Preferences/Index.vue) - 用户偏好页
- [resources/js/Pages/Settings/Preferences/Partials/DateFormat.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/resources/js/Pages/Settings/Preferences/Partials/DateFormat.vue) - 日期格式组件

### 账户创建
- [app/Domains/Settings/CreateAccount/Services/CreateAccount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/CreateAccount/Services/CreateAccount.php)
- [app/Domains/Settings/CreateAccount/Jobs/SetupAccount.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/CreateAccount/Jobs/SetupAccount.php)

### 存储设置
- [app/Domains/Settings/ManageStorage/Web/Controllers/AccountStorageController.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageStorage/Web/Controllers/AccountStorageController.php)
- [app/Domains/Settings/ManageStorage/Web/ViewHelpers/StorageIndexViewHelper.php](file:///d:/fz/0601-2/solo-dogfeeding/code/35-monica/app/Domains/Settings/ManageStorage/Web/ViewHelpers/StorageIndexViewHelper.php)
