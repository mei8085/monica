# Monica 认证流程分析：Fortify 与 Jetstream 扩展钩子

## 概述

Monica 项目基于 Laravel 框架，使用 **Laravel Fortify** 作为后端认证逻辑、**Laravel Jetstream**（Inertia 栈）作为前端脚手架，并深度集成了 **WebAuthn**（无密码认证）和 **Socialite**（社交登录）能力。本文档梳理认证流程中四个关键扩展点：守卫替换、双因素验证、记住我/会话刷新、社交登录绑定。

---

## 一、认证守卫的替换与包装

### 1. 多 Guard 并存

项目配置了两种认证守卫，分别服务于 Web 页面和 API：

| Guard 名称 | 驱动 | 用户提供者 | 用途 |
|-----------|------|-----------|------|
| `web` | `session` | `users` (webauthn driver) | Fortify 登录、Web 页面认证 |
| `sanctum` | — (Sanctum 内置) | 同 `web` 的 Eloquent 模型 | Jetstream、API 令牌认证 |

配置文件：[auth.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/auth.php#L38-L72)

### 2. WebAuthn 用户提供者替换

默认的 `eloquent` 用户提供者被替换为 `webauthn` 驱动：

```php
// config/auth.php
'providers' => [
    'users' => [
        'driver' => 'webauthn',   // 而非默认的 'eloquent'
        'model' => env('AUTH_MODEL', App\Models\User::class),
    ],
],
```

这是由 `asbiin/laravel-webauthn` 包提供的扩展，使得 `Auth::attempt()` 既能验证密码也能验证 WebAuthn 断言。

- 配置文件：[webauthn.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/webauthn.php)
- 守卫关联：WebAuthn 默认使用 `web` guard ([webauthn.php#L29](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/webauthn.php#L29))

### 3. Fortify Guard 配置

Fortify 明确指定使用 `web` guard：

- [fortify.php#L18](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/fortify.php#L18)

### 4. Jetstream Guard 配置

Jetstream 使用 `sanctum` guard，适用于 Inertia 页面和 API：

- [jetstream.php#L47](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/jetstream.php#L47)

### 5. Sanctum 用户设置中间件

[SanctumSetUser](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Middleware/SanctumSetUser.php) 中间件用于将当前已认证用户同步到 Sanctum guard，并附加一个 `TransientToken`（瞬时令牌），使得 Web 会话用户可以通过 Sanctum 授权检查。

```php
// SanctumSetUser.php
$this->sanctum()->setUser($request->user()->withAccessToken(new TransientToken));
```

### 6. User 模型的认证 Trait

[User 模型](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Models/User.php#L21-L29) 组合了三个认证相关 Trait：

- `Laravel\Fortify\TwoFactorAuthenticatable` — Fortify 双因素认证
- `Laravel\Sanctum\HasApiTokens` — Sanctum API 令牌
- `LaravelWebauthn\WebauthnAuthenticatable` — WebAuthn 认证

---

## 二、双因素验证插入环节

Monica 支持**两种**双因素认证方式：TOTP（标准 2FA）和 WebAuthn（硬件密钥/生物识别），两者在登录管道中并行检查。

### 1. Fortify 登录管道自定义

Fortify 的 `login` 管道被重定义，将双因素检查前置到密码验证之前的位置：

- [fortify.php#L145-L151](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/fortify.php#L145-L151)

```php
'pipelines' => [
    'login' => [
        \App\Actions\Fortify\RedirectIfTwoFactorAuthenticatable::class,  // 自定义：先检查 2FA
        \Laravel\Fortify\Actions\AttemptToAuthenticate::class,            // 密码认证
        \Laravel\Fortify\Actions\PrepareAuthenticatedSession::class,      // 准备会话
    ],
],
```

> 默认 Fortify 管道顺序是 Attempt → 2FA Redirect → Prepare。Monica 将 2FA 检查提到最前面，先验证凭证再判断是否需要 2FA 挑战。

### 2. RedirectIfTwoFactorAuthenticatable

[RedirectIfTwoFactorAuthenticatable](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php) 是核心的 2FA 前置检查动作：

- 第 30 行：先验证用户凭证（不登录）
- 第 32-33 行：检查两种 2FA 方式任一是否启用：
  - `two_factor_secret` 且 `two_factor_confirmed_at` 不为空（TOTP）
  - `Webauthn::enabled($user)`（WebAuthn 密钥）
- 第 34 行：如果启用任一 2FA，则跳转到挑战页面，不继续管道
- 第 37 行：否则继续后续管道（密码登录 + 会话准备）

### 3. 双因素挑战视图

双因素挑战页面由 [TwoFactorChallengeView](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Fortify/TwoFactorChallengeView.php) 渲染，在 Fortify 中注册：

- [FortifyServiceProvider.php#L56](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/FortifyServiceProvider.php#L56)

挑战视图同时准备：
- TOTP 输入框的状态
- WebAuthn 的 `publicKey` 断言数据（用于前端发起 WebAuthn 认证）

### 4. 社交登录中的 2FA 检查

社交登录路径同样插入了 2FA 检查，在 [AttemptToAuthenticateSocialite](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateSocialite.php#L51-L54) 中：

```php
if ((optional($user)->two_factor_secret && ! is_null(optional($user)->two_factor_confirmed_at))
    || Webauthn::enabled($user)) {
    return $this->twoFactorChallengeResponse($request, $user);
}
```

逻辑与邮箱密码登录一致，确保所有登录入口都经过 2FA 关卡。

### 5. WebAuthn 认证动作

[AttemptToAuthenticateWebauthn](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateWebauthn.php) 处理 WebAuthn 断言验证，支持两种场景：

- **已登录用户的二次验证**（如敏感操作前的 WebAuthn 确认）
- **直接登录验证**（无密码登录）

### 6. WebAuthn 认证事件监听

[WebauthnAuthenticateListener](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Listeners/WebauthnAuthenticateListener.php) 监听 `AuthenticatorAssertionResponseValidationSucceededEvent` 事件，每次成功的 WebAuthn 认证后更新密钥的 `used_at` 时间戳。

### 7. 2FA 速率限制

双因素挑战有独立的速率限制：

- [FortifyServiceProvider.php#L64](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/FortifyServiceProvider.php#L64)
- 每分钟 5 次，按 `login.id`（会话中的用户 ID）限流

---

## 三、记住我与会话刷新策略

### 1. 记住我（Remember Me）基础机制

项目使用 Laravel 原生的 `remember` 机制，在以下入口支持：

- **邮箱密码登录**：通过 `$request->boolean('remember')` 传递给 `Auth::attempt()`
- **社交登录**：在 [SocialiteCallbackController](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Auth/SocialiteCallbackController.php#L33-L35) 中将 `remember` 存入 session，后续在 [AttemptToAuthenticateSocialite](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateSocialite.php#L56) 中取出使用

### 2. WebAuthn 记住我扩展

WebAuthn 与记住我的交互有两处扩展：

#### a. LoginViaRemember 事件订阅

在 [AppServiceProvider.php#L157](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/AppServiceProvider.php#L157) 中注册：

```php
Event::subscribe(LoginViaRemember::class);
```

这是 `laravel-webauthn` 包提供的监听器，当用户通过 "记住我" cookie 自动登录时，触发 WebAuthn 相关的登录流程（如自动填充 WebAuthn 会话状态）。

#### b. LoginListener 与 return cookie

[LoginListener](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Listeners/LoginListener.php) 监听 `Illuminate\Auth\Events\Login` 事件：

```php
public function handle(Login $event)
{
    if ($event->remember && $event->user->webauthnKeys()->count() > 0) {
        Cookie::queue('return', 'true', 60 * 24 * 365);
    }
}
```

当满足以下条件时设置有效期 1 年的 `return` cookie：
- 用户勾选了 "记住我"
- 用户拥有至少一个 WebAuthn 密钥

该 cookie 用于登录页检测是否自动发起 WebAuthn 无密码登录（`autologin` 标志）。

参考：[LoginController.php#L36](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Auth/LoginController.php#L36)

### 3. 会话配置

会话配置见 [session.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/session.php)：

| 配置项 | 默认值 | 说明 |
|-------|--------|------|
| `driver` | `database` | 会话存储驱动 |
| `lifetime` | `120` 分钟 | 会话闲置过期时间 |
| `expire_on_close` | `false` | 关闭浏览器时是否销毁会话 |
| `cookie` | 基于 app name | 会话 cookie 名称 |
| `same_site` | `lax` | SameSite 属性 |

### 4. 会话认证中间件（AuthenticateSession）

Jetstream 提供的 `AuthenticateSession` 中间件用于在已认证会话中验证用户身份的有效性（防止会话固定攻击、密码修改后失效等）。

- 配置：[jetstream.php#L34](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/jetstream.php#L34)
- 使用位置：[web.php#L184-L188](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/routes/web.php#L184-L188)

受保护的路由组使用 `auth:sanctum` + `config('jetstream.auth_session')` + `verified` 三层中间件。

### 5. 密码确认（Password Confirmation）

敏感操作前的密码确认机制：

- 超时时间：`password_timeout = 10800` 秒（3 小时），见 [auth.php#L113](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/auth.php#L113)
- 自定义确认逻辑：[FortifyServiceProvider.php#L44-L50](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/FortifyServiceProvider.php#L44-L50)

```php
Fortify::confirmPasswordsUsing(fn ($user, ?string $password = null) => $user->password
    ? app(StatefulGuard::class)->validate([...])
    : true  // 没有密码的用户（如仅社交登录）直接通过
);
```

> 关键设计：没有设置密码的用户（纯社交登录用户）可以跳过密码确认，直接返回 `true`。

---

## 四、社交登录绑定动作

### 1. 支持的社交登录提供商

配置于 [auth.php#L127](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/auth.php#L127) 的 `login_providers` 数组，通过环境变量 `LOGIN_PROVIDERS` 配置。

具体凭证在 [services.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/services.php) 中：

| 提供商 | Socialite 包 | 备注 |
|-------|-------------|------|
| `azure` | socialiteproviders/microsoft-azure | Microsoft Azure AD |
| `facebook` | socialiteproviders/facebook | |
| `github` | socialiteproviders/github | |
| `google` | socialiteproviders/google | |
| `linkedin` | socialiteproviders/linkedin | |
| `kanidm` | socialiteproviders/kanidm | |
| `keycloak` | socialiteproviders/keycloak | |
| `saml2` | — | SAML 2.0 |

### 2. Socialite 扩展注册

在 [AppServiceProvider.php#L159-L165](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/AppServiceProvider.php#L159-L165) 中，通过监听 `SocialiteWasCalled` 事件注册各个第三方 Socialite 扩展：

```php
Event::listen(SocialiteWasCalled::class, AzureExtendSocialite::class);
Event::listen(SocialiteWasCalled::class, FacebookExtendSocialite::class);
// ... 其他提供商
```

### 3. 社交登录路由

路由定义于 [web.php#L168-L172](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/routes/web.php#L168-L172)，受 `throttle:oauth2-socialite` 速率限制（每分钟 5 次）：

```php
Route::get('auth/{driver}', [SocialiteCallbackController::class, 'login'])->name('login.provider');
Route::get('auth/{driver}/callback', [SocialiteCallbackController::class, 'callback']);
Route::post('auth/{driver}/callback', [SocialiteCallbackController::class, 'callback']);
```

### 4. 社交登录控制器

[SocialiteCallbackController](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Auth/SocialiteCallbackController.php) 处理登录跳转和回调：

- **login 方法**（第 24 行）：校验 provider 有效性 → 保存 redirect → 保存 remember → 跳转到 OAuth 服务商
- **callback 方法**（第 43 行）：校验 → 通过自定义管道登录 → 跳转目标页
- **loginPipeline 方法**（第 60 行）：使用 Laravel Pipeline 模式构建认证流程

### 5. 社交登录认证管道

回调处理使用自定义的两步管道：

```php
// SocialiteCallbackController.php
protected function loginPipeline(Request $request): Pipeline
{
    return (new Pipeline(app()))->send($request)->through([
        AttemptToAuthenticateSocialite::class,  // 社交认证
        PrepareAuthenticatedSession::class,      // 准备会话（来自 Fortify）
    ]);
}
```

### 6. AttemptToAuthenticateSocialite 核心逻辑

[AttemptToAuthenticateSocialite](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateSocialite.php) 是社交登录的核心动作，处理三种绑定场景：

#### 场景 1：已有绑定 → 直接登录

- 第 80-88 行：通过 `driver_id` + `driver` 在 `user_tokens` 表查找
- 如果找到关联记录，获取对应 `user`

#### 场景 2：新用户 + 未登录 → 创建账号并绑定

- 第 91-93 行：通过 `CreateNewUser` 创建新用户
- 通过 `createUserToken` 创建绑定记录
- 触发 `Registered` 事件

#### 场景 3：已登录用户 + 新提供商 → 追加绑定

- 第 114-116 行：如果当前已认证，直接使用当前用户
- 为该用户添加新的社交令牌

#### 冲突检测

- 第 102-107 行：如果已登录用户尝试绑定的社交账号已属于另一个用户，抛出错误

### 7. 用户令牌模型

[UserToken](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Models/UserToken.php) 模型存储社交登录绑定信息，字段包括：

- `driver` / `driver_id` — 提供商名称和用户在提供商处的 ID
- `email` — 社交账号邮箱
- `format` — `oauth1` 或 `oauth2`
- `token` / `token_secret` / `refresh_token` / `expires_in` — OAuth 令牌

### 8. 解绑（取消绑定）

[UserTokenController](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Profile/UserTokenController.php) 继承自 Jetstream 的 `UserProfileController`，提供解绑功能：

```php
public function destroy(Request $request, string $driver)
{
    $request->user()->userTokens()
        ->where('driver', $driver)
        ->delete();

    return redirect()->route('profile.show');
}
```

### 9. 用户资料页展示

[UserProfile](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Jetstream/UserProfile.php) 是 Jetstream `whenRendering` 钩子，在 Profile 页面注入社交登录提供商和 WebAuthn 密钥数据：

- 通过 `Jetstream::inertia()->whenRendering('Profile/Show', new UserProfile)` 注册，见 [JetstreamServiceProvider.php#L33](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/JetstreamServiceProvider.php#L33)

---

## 五、Fortify / Jetstream 其他自定义钩子

### Fortify 自定义汇总

注册位置：[FortifyServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/FortifyServiceProvider.php)

| 钩子 | 自定义类/回调 | 用途 |
|-----|-------------|------|
| `loginView` | LoginController | 自定义登录页视图（Inertia） |
| `registerView` | RegisterController | 自定义注册页视图 |
| `createUsersUsing` | CreateNewUser | 用户创建逻辑 |
| `updateUserProfileInformationUsing` | UpdateUserProfileInformation | 更新用户资料 |
| `updateUserPasswordsUsing` | UpdateUserPassword | 更新密码 |
| `resetUserPasswordsUsing` | ResetUserPassword | 重置密码 |
| `confirmPasswordsUsing` | 闭包 | 密码确认逻辑（支持无密码用户） |
| `twoFactorChallengeView` | TwoFactorChallengeView | 2FA 挑战视图 |
| 注册路由补丁 | `patchFortifyRoutes()` | 给注册路由加 `monica.signup_is_enabled` 中间件 |

### Jetstream 自定义汇总

注册位置：[JetstreamServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/JetstreamServiceProvider.php)

| 钩子 | 自定义类/值 | 用途 |
|-----|------------|------|
| `deleteUsersUsing` | DeleteUser | 用户删除逻辑 |
| `whenRendering('Profile/Show')` | UserProfile | 资料页注入数据 |
| `whenRendering('API/Index')` | 闭包 | API 页注入数据 |
| `defaultApiTokenPermissions` | `['read']` | 默认 API 令牌权限 |
| `permissions` | `['read', 'write']` | 可用权限列表 |

### 其他全局认证钩子

| 位置 | 钩子类型 | 用途 |
|-----|---------|------|
| [AppServiceProvider.php#L137](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/AppServiceProvider.php#L137) | `RedirectIfAuthenticated::redirectUsing` | 已认证用户重定向目标（ vault 列表） |
| [AppServiceProvider.php#L154-L155](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/AppServiceProvider.php#L154-L155) | Webauthn::updateViewResponseUsing / destroyViewResponseUsing | WebAuthn 操作后的响应 |
| [AppServiceProvider.php#L157](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/AppServiceProvider.php#L157) | `Event::subscribe(LoginViaRemember::class)` | WebAuthn 记住我登录 |

---

## 六、登录流程图（简化）

```
用户访问登录页
    │
    ├─── 检测 'return' cookie → 自动发起 WebAuthn 无密码登录
    │
    ├─── 邮箱密码登录
    │      │
    │      ▼
    │   Fortify login pipeline
    │      │
    │      ├─ RedirectIfTwoFactorAuthenticatable
    │      │    ├─ 验证密码
    │      │    ├─ 检查 TOTP 或 WebAuthn 是否启用
    │      │    └─ 启用 → 跳转 2FA 挑战页
    │      │
    │      ├─ AttemptToAuthenticate（密码验证）
    │      └─ PrepareAuthenticatedSession（准备会话 + 登录）
    │
    └─── 社交登录
           │
           ▼
        Socialite 跳转 → 回调
           │
           ▼
        loginPipeline
           │
           ├─ AttemptToAuthenticateSocialite
           │    ├─ 查找已有绑定
           │    ├─ 未找到 → 新建用户/绑定到当前用户
           │    ├─ 检查 2FA → 跳转挑战页
           │    └─ 登录用户
           └─ PrepareAuthenticatedSession
```

---

## 七、关键文件索引

| 文件路径 | 说明 |
|---------|------|
| [config/fortify.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/fortify.php) | Fortify 配置（管道、特性、guard） |
| [config/jetstream.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/jetstream.php) | Jetstream 配置（guard、会话中间件） |
| [config/auth.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/auth.php) | 认证配置（guards、providers、密码超时） |
| [config/webauthn.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/webauthn.php) | WebAuthn 配置 |
| [config/services.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/services.php) | 社交登录凭证 |
| [app/Providers/FortifyServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/FortifyServiceProvider.php) | Fortify 钩子注册 |
| [app/Providers/JetstreamServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/JetstreamServiceProvider.php) | Jetstream 钩子注册 |
| [app/Providers/AppServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/AppServiceProvider.php) | 全局认证钩子（Socialite、WebAuthn、重定向） |
| [app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php) | 2FA 前置检查 |
| [app/Actions/Fortify/TwoFactorChallengeView.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Fortify/TwoFactorChallengeView.php) | 2FA 挑战视图 |
| [app/Actions/AttemptToAuthenticateSocialite.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateSocialite.php) | 社交登录认证动作 |
| [app/Actions/AttemptToAuthenticateWebauthn.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateWebauthn.php) | WebAuthn 认证动作 |
| [app/Http/Controllers/Auth/SocialiteCallbackController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Auth/SocialiteCallbackController.php) | 社交登录控制器 |
| [app/Http/Controllers/Profile/UserTokenController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Profile/UserTokenController.php) | 社交账号解绑 |
| [app/Listeners/LoginListener.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Listeners/LoginListener.php) | 登录事件 → 设置 return cookie |
| [app/Listeners/WebauthnAuthenticateListener.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Listeners/WebauthnAuthenticateListener.php) | WebAuthn 认证 → 更新 used_at |
| [app/Http/Middleware/SanctumSetUser.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Middleware/SanctumSetUser.php) | 同步用户到 Sanctum guard |
| [app/Models/User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Models/User.php) | User 模型（含多个认证 Trait） |
| [app/Models/UserToken.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Models/UserToken.php) | 社交令牌模型 |
| [app/Models/WebauthnKey.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Models/WebauthnKey.php) | WebAuthn 密钥模型 |
