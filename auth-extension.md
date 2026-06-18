# Monica 登录认证扩展层代码走向

Monica 的认证体系基于 Laravel 生态的三层扩展框架搭建：**Fortify**（无 UI 的后端认证管线）、**Jetstream**（Inertia 前端脚手架）和 **Laravel Webauthn**（Passkey / 安全密钥认证）。在此基础上，项目通过 Socialite 接入了外部 OAuth/SAML 提供商。下面按代码执行顺序梳理三个核心场景。

---

## 一、架构总览与关键配置

### 1. 认证 Guard 与 User Provider

[auth.php](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/config/auth.php#L38-L43) 定义了唯一的 guard `web`，使用 `session` 驱动。值得注意的是 User Provider 的驱动被设为 `webauthn`（而非默认的 `eloquent`），这意味着用户检索逻辑经过了 Laravel Webauthn 包的包装——但实质上仍然以 `App\Models\User` 为模型，只是在此基础上增加了 Webauthn 认证能力的感知。

### 2. Fortify Pipeline 自定义

[fortify.php](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/config/fortify.php#L145-L151) 中 `pipelines.login` 被显式覆盖，替换了 Fortify 默认的登录管线：

```php
'pipelines' => [
    'login' => [
        \App\Actions\Fortify\RedirectIfTwoFactorAuthenticatable::class,
        \Laravel\Fortify\Actions\AttemptToAuthenticate::class,
        \Laravel\Fortify\Actions\PrepareAuthenticatedSession::class,
    ],
],
```

Fortify 默认管线中的 `RedirectIfTwoFactorAuthenticatable` 是包内置的，这里替换成了 `App\Actions\Fortify\RedirectIfTwoFactorAuthenticatable`，这是整个扩展层最关键的切入点——它在凭据验证通过后、真正登录之前，拦截请求判断是否需要进入双因素验证流程。

### 3. Jetstream 功能开关

[jetstream.php](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/config/jetstream.php#L60-L66) 启用了 `termsAndPrivacyPolicy` 和 `api`，未启用 `profilePhotos`、`teams`、`accountDeletion`。Jetstream 的 guard 设置为 `sanctum`，用于 API token 认证。

### 4. Webauthn 配置

[webauthn.php](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/config/webauthn.php) 关键点：
- `enable` 为 `true`，Webauthn 全局启用
- `userless` 由环境变量 `WEBAUTHN_USERLESS` 控制，默认为 `true`——即支持"无用户名登录"（Passkey 一键登录）
- `redirects.login` 指向 `/vaults`
- `views.authenticate` 和 `views.register` 均为 `null`，表示不使用包内置的 Blade 视图，而是走 Inertia 前端

### 5. 外部登录提供商

[auth.php](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/config/auth.php#L127) 中 `login_providers` 通过 `LOGIN_PROVIDERS` 环境变量动态注入，支持逗号分隔的多个提供商。实际在 [services.php](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/config/services.php) 中配置了 azure、facebook、github、google、linkedin、saml2、kanidm、keycloak 共 8 种。各提供商的 Socialite 扩展在 [AppServiceProvider](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Providers/AppServiceProvider.php#L159-L165) 中通过监听 `SocialiteWasCalled` 事件注册。

---

## 二、注册场景

注册有两条入口：**标准注册**和**邀请注册**。

### 入口 A：标准注册

```
前端表单提交
  → POST /register（Fortify 自动注册的路由）
    → App\Actions\Fortify\CreateNewUser::create()
      → App\Domains\Settings\CreateAccount\Services\CreateAccount::execute()
        → 创建 Account 记录
        → 创建 User 记录（关联 Account，标记为 is_account_administrator）
        → 派发 SetupAccount 队列任务
```

**代码走向细节：**

1. 前端 [Register.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/resources/js/Pages/Auth/Register.vue#L27-L31) 通过 `form.post(route('register'))` 提交到 Fortify 注册端点。

2. Fortify 根据 [fortify.php](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/config/fortify.php#L134) 中 `Features::registration()` 的配置，将请求路由到 `CreateNewUser` Action。这个 Action 实现了 `CreatesNewUsers` 契约。

3. [CreateNewUser](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Actions/Fortify/CreateNewUser.php#L19-L35) 先做输入校验（first_name, last_name, email, password, terms），然后委托给领域服务 [CreateAccount](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Domains/Settings/CreateAccount/Services/CreateAccount.php#L38-L49)。

4. `CreateAccount` 做了两次校验（`validateRules` 是 BaseService 的逻辑），然后：
   - 创建 `Account` 记录（设置默认存储上限）
   - 创建 `User` 记录（密码通过 `Hash::make` 加密，若 password 为 null 则允许无密码用户——这是为 OAuth 注册预留的）
   - 派发 [SetupAccount](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Domains/Settings/CreateAccount/Jobs/SetupAccount.php) 队列任务，异步初始化账户的默认数据（模板、模块、性别、代词、关系类型等）

5. 注册页面的可见性由 [SignupHelper](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Helpers/SignupHelper.php#L22-L25) 控制：当配置 `monica.disable_signup` 为 true 且数据库中已有至少一个 Account 时，注册入口关闭。

6. [RegisterController](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Http/Controllers/Auth/RegisterController.php) 只负责渲染注册页面视图，向 Vue 传递外部登录提供商列表。

### 入口 B：邀请注册

```
用户访问邀请链接
  → GET /invitation/{code}
    → AcceptInvitationController::show()
      → 查找有效的邀请码
      → 渲染 AcceptInvitation.vue
  → POST /invitation
    → AcceptInvitationController::store()
      → AcceptInvitation 服务执行
      → Auth::login() 直接登录
```

邀请注册不经过 Fortify 管线。[AcceptInvitationController](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Http/Controllers/Auth/AcceptInvitationController.php#L32-L55) 的 `store` 方法自行校验后调用 `AcceptInvitation` 领域服务创建用户，然后直接调用 `Auth::login()` 登录——绕过了双因素验证检查。这是一个潜在的设计注意点。

---

## 三、登录场景

登录有**三条并行入口**：邮箱密码登录、Webauthn Passkey 登录、外部 OAuth/SAML 登录。

### 入口 A：邮箱密码登录（Fortify 管线）

```
前端表单提交
  → POST /login（Fortify 路由）
    → Fortify 登录 Pipeline（按 fortify.php 中 pipelines.login 配置）
      ├─ 1. RedirectIfTwoFactorAuthenticatable（自定义扩展）
      │     ├─ 验证凭据 → 失败则抛 ValidationException
      │     ├─ 检查 two_factor_secret + two_factor_confirmed_at → 有则重定向到 2FA 挑战页
      │     ├─ 检查 Webauthn::enabled($user) → 有则重定向到 2FA 挑战页
      │     └─ 无 2FA → 传递给下一个 Pipeline 管道
      ├─ 2. AttemptToAuthenticate（Fortify 内置）
      │     └─ 调用 Guard::attempt() 正式登录
      └─ 3. PrepareAuthenticatedSession（Fortify 内置）
            └─ 处理 session 再生成等安全操作
```

**代码走向细节：**

1. 前端 [Login.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/resources/js/Pages/Auth/Login.vue#L60-L69) 提交到 `route('login')`。

2. 请求进入 Fortify 管线，第一个管道是自定义的 [RedirectIfTwoFactorAuthenticatable](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php#L28-L38)。它的核心逻辑：
   - 先调用 `validateCredentials()` 通过 email 查找用户并验证密码（注意：此时并未真正登录，只是验证凭据是否匹配）
   - 如果凭据无效：触发 `Failed` 事件、增加速率限制、抛出 `ValidationException`
   - 如果凭据有效但用户启用了 TOTP 二步验证（`two_factor_secret` 非空且 `two_factor_confirmed_at` 非空）**或**启用了 Webauthn 密钥（`Webauthn::enabled($user)`）：将 `login.id` 和 `login.remember` 写入 session，触发 `TwoFactorAuthenticationChallenged` 事件，重定向到 `two-factor.login` 路由
   - 如果无 2FA 需求：`return $next($request)` 传递给下一个管道

3. [LoginController](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Http/Controllers/Auth/LoginController.php#L20-L47) 只负责渲染登录页面，它做了以下准备工作：
   - 读取 `auth.login_providers` 配置构建外部登录按钮列表
   - 如果 Webauthn userless 模式启用，调用 `Webauthn::prepareAssertion(null)` 生成无用户名的公钥挑战（用于 Passkey 自动填充登录）
   - 读取 `return` cookie 判断是否自动触发 Passkey 登录

### 入口 B：Webauthn Passkey 登录

```
浏览器 WebAuthn API 认证
  → POST /webauthn/auth（Laravel Webauthn 包路由）
    → Webauthn 包内部处理断言验证
    → 登录成功 → 重定向到 /vaults
```

**代码走向细节：**

1. 在 [Login.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/resources/js/Pages/Auth/Login.vue#L152-L159) 中，当 `userless` 为 true 或 `publicKey` 存在时，显示"使用 Passkey 登录"按钮或直接渲染 [WebauthnLogin.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/resources/js/Pages/Webauthn/WebauthnLogin.vue) 组件。

2. `WebauthnLogin.vue` 调用 `@simplewebauthn/browser` 的 `startAuthentication()` 与浏览器安全密钥交互，拿到认证数据后 `POST` 到 `route('webauthn.auth')`。

3. 这条路径完全由 Laravel Webauthn 包处理，**不经过 Fortify 管线**，因此也**不触发双因素验证的二次检查**——因为 Webauthn 本身就是第二因素。

4. 登录成功后触发 `Login` 事件，被 [LoginListener](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Listeners/LoginListener.php#L15-L19) 监听：如果用户勾选了"记住我"且拥有 Webauthn 密钥，则设置 `return` cookie（有效期一年），下次访问登录页时自动触发 Passkey 登录。

5. 此外，`AppServiceProvider` 中注册了 `LoginViaRemember` 事件订阅者（来自 Laravel Webauthn 包），用于处理"记住我"cookie 触发的自动登录。

### 入口 C：外部 OAuth/SAML 登录（Socialite 管线）

```
用户点击外部登录按钮
  → GET /auth/{driver}（SocialiteCallbackController::login）
    → 重定向到 OAuth 提供商授权页
  → GET /auth/{driver}/callback（SocialiteCallbackController::callback）
    → 自定义登录 Pipeline：
      ├─ 1. AttemptToAuthenticateSocialite（自定义扩展）
      │     ├─ 获取 Socialite 用户信息
      │     ├─ 查找 UserToken 关联
      │     │     ├─ 已关联 → 校验当前登录用户一致性
      │     │     └─ 未关联 → getUserOrCreate()
      │     │           ├─ 已登录 → 关联到当前用户
      │     │           └─ 未登录 → createUser() → CreateNewUser + 派发 Registered 事件
      │     ├─ 创建 UserToken 记录
      │     ├─ 检查 2FA（同 RedirectIfTwoFactorAuthenticatable 逻辑）
      │     │     ├─ 有 2FA → 重定向到 two-factor.login
      │     │     └─ 无 2FA → Guard::login() 直接登录
      │     └─ 传递给下一个管道
      └─ 2. PrepareAuthenticatedSession（Fortify 内置）
```

**代码走向细节：**

1. 前端 [ExternalProviders.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/resources/js/Pages/Auth/ExternalProviders.vue#L15-L27) 通过 `GET route('login.provider', { driver })` 发起外部登录。

2. [SocialiteCallbackController::login()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Http/Controllers/Auth/SocialiteCallbackController.php#L24-L38) 先检查 provider 是否在配置中启用，处理 redirect 和 remember 参数，然后通过 `Socialite::driver($driver)->redirect()` 跳转到外部授权页。

3. 回调进入 [SocialiteCallbackController::callback()](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Http/Controllers/Auth/SocialiteCallbackController.php#L43-L55)，它构建了一个自定义 Pipeline（注意：**不是 Fortify 的管线**，而是独立实例化的 `Illuminate\Pipeline\Pipeline`）。

4. [AttemptToAuthenticateSocialite](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Actions/AttemptToAuthenticateSocialite.php#L44-L59) 是 Socialite 管线的核心扩展，它：
   - 获取 Socialite 提供商实例（本地环境跳过 SSL 验证）
   - 获取外部用户信息后调用 `authenticateUser()`
   - `authenticateUser()` 根据 `UserToken` 表查找已有的 driver_id + driver 关联：
     - **已有关联**：取出关联的用户，检查当前登录用户是否与关联用户一致（防止绑定冲突）
     - **无关联**：调用 `getUserOrCreate()`：
       - 如果当前已登录（`Auth::user()` 非空），则为当前用户创建新的 UserToken 关联——这是**关联已有账户**的场景
       - 如果未登录，调用 `createUser()` 创建新用户——这会复用 [CreateNewUser](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Actions/Fortify/CreateNewUser.php) 并触发 `Registered` 事件
   - 创建 `UserToken` 记录，区分 OAuth1 和 OAuth2 格式存储 token 信息
   - 最后检查 2FA：判断逻辑与 `RedirectIfTwoFactorAuthenticatable` 一致，但这里是**手动调用** `$this->guard->login()` 而非依赖 Fortify 管线

5. Socialite 路由组使用了 `throttle:oauth2-socialite` 限流（每分钟 5 次），见 [web.php](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/routes/web.php#L168-L172)。

---

## 四、双因素验证场景

2FA 挑战页面同时承载了 TOTP 验证码和 Webauthn 安全密钥两种二步验证方式。

### 挑战页面渲染

```
用户被重定向到 two-factor.login 路由
  → TwoFactorChallengeView::toResponse()
    → 从 session 取出 login.id 查找用户
    → 检查用户是否启用了 Webauthn → 生成 prepareAssertion 公钥
    → 渲染 TwoFactorChallenge.vue（携带 publicKey、twoFactor、remember）
```

[TwoFactorChallengeView](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Actions/Fortify/TwoFactorChallengeView.php#L18-L32) 实现了 `TwoFactorChallengeViewResponse` 契约，替代了 Fortify 默认的挑战页视图。它根据用户状态准备不同的验证数据：

- 如果用户有 Webauthn 密钥：生成 `Webauthn::prepareAssertion($user)` 公钥参数
- `twoFactor` 布尔值指示用户是否启用了 TOTP

### 前端挑战页面结构

[TwoFactorChallenge.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/resources/js/Pages/Auth/TwoFactorChallenge.vue) 的渲染逻辑：

1. **如果 `publicKey` 不为 null**（用户有 Webauthn 密钥）：渲染 [WebauthnLogin.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/resources/js/Pages/Webauthn/WebauthnLogin.vue) 组件，提示用户"请通过安全密钥确认访问"，禁用 autofill
2. **如果 `twoFactor` 为 true**（用户启用了 TOTP）：渲染 TOTP 验证码输入表单，支持切换到恢复码模式

两种方式可以同时存在——页面上会优先展示 Webauthn 挑战，同时也展示 TOTP 输入框。

### 二步验证提交

- **Webauthn 方式**：`WebauthnLogin.vue` 提交到 `route('webauthn.auth')`，由 Laravel Webauthn 包验证断言
- **TOTP 方式**：`TwoFactorChallenge.vue` 提交到 `route('two-factor.login')`，由 Fortify 内置的 `TwoFactorAuthenticateAction` 验证

两种方式验证成功后，都会完成登录流程并重定向到 `/vaults`。

---

## 五、扩展层关键文件索引

| 文件 | 角色 |
|------|------|
| [fortify.php](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/config/fortify.php) | Fortify 功能与管线配置 |
| [auth.php](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/config/auth.php) | Guard、Provider、外部登录提供商配置 |
| [webauthn.php](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/config/webauthn.php) | Webauthn 全局配置 |
| [jetstream.php](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/config/jetstream.php) | Jetstream 功能开关 |
| [services.php](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/config/services.php) | 各 OAuth 提供商凭证配置 |
| [RedirectIfTwoFactorAuthenticatable](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php) | 邮箱密码登录管线中的 2FA 拦截点 |
| [TwoFactorChallengeView](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Actions/Fortify/TwoFactorChallengeView.php) | 2FA 挑战页视图响应（含 Webauthn 公钥准备） |
| [CreateNewUser](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Actions/Fortify/CreateNewUser.php) | 注册 Action（Fortify 契约实现） |
| [CreateAccount](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Domains/Settings/CreateAccount/Services/CreateAccount.php) | 注册领域服务（创建 Account + User + 初始化） |
| [AttemptToAuthenticateSocialite](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Actions/AttemptToAuthenticateSocialite.php) | OAuth 登录管线核心（含注册/关联/2FA 检查） |
| [SocialiteCallbackController](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Http/Controllers/Auth/SocialiteCallbackController.php) | OAuth 登录/回调控制器 |
| [LoginController](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Http/Controllers/Auth/LoginController.php) | 登录页渲染（含 Webauthn 准备） |
| [RegisterController](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Http/Controllers/Auth/RegisterController.php) | 注册页渲染 |
| [AcceptInvitationController](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Http/Controllers/Auth/AcceptInvitationController.php) | 邀请注册控制器 |
| [User](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Models/User.php) | 用户模型（组合 TwoFactorAuthenticatable + WebauthnAuthenticatable） |
| [WebauthnKey](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Models/WebauthnKey.php) | Webauthn 密钥模型（扩展 used_at 字段） |
| [UserToken](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Models/UserToken.php) | 外部登录提供商关联令牌模型 |
| [LoginListener](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Listeners/LoginListener.php) | 登录事件监听（设置 return cookie 以启用 Passkey 自动登录） |
| [AppServiceProvider](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/app/Providers/AppServiceProvider.php) | Socialite 提供商注册 + Webauthn 响应类绑定 |
| [Login.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/resources/js/Pages/Auth/Login.vue) | 登录页前端（邮箱密码 + Passkey + 外部提供商） |
| [Register.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/resources/js/Pages/Auth/Register.vue) | 注册页前端 |
| [TwoFactorChallenge.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/resources/js/Pages/Auth/TwoFactorChallenge.vue) | 2FA 挑战页前端（Webauthn + TOTP） |
| [WebauthnLogin.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/resources/js/Pages/Webauthn/WebauthnLogin.vue) | Webauthn 登录组件 |
| [ExternalProviders.vue](file:///d:/fz/0601-2/solo-dogfeeding/code/34-monica/resources/js/Pages/Auth/ExternalProviders.vue) | 外部登录提供商按钮组件 |

---

## 六、代码走向的"为什么让人困惑"

以下是导致扩展层代码走向难以直观理解的几个设计因素：

1. **管线分散**：邮箱密码登录走 Fortify Pipeline，OAuth 登录走独立实例化的 `Illuminate\Pipeline\Pipeline`，Webauthn Passkey 登录走 Laravel Webauthn 包自己的路由。三条路径各自独立，2FA 检查逻辑分别在 `RedirectIfTwoFactorAuthenticatable` 和 `AttemptToAuthenticateSocialite` 中重复实现。

2. **自定义与内置混合**：`RedirectIfTwoFactorAuthenticatable` 替换了 Fortify 默认管道，但 `AttemptToAuthenticate` 和 `PrepareAuthenticatedSession` 仍使用 Fortify 内置版本。这种"替换其中一环"的做法需要知道 Fortify 的管线机制才能理解。

3. **职责双重性**：`AttemptToAuthenticateSocialite` 同时承担了"认证已有用户"和"注册新用户"两个职责，还包含了 2FA 检查——一个 Action 里承载了三个不同场景的逻辑分支。

4. **隐式契约绑定**：`TwoFactorChallengeView` 和 `WebauthnUpdateResponse` / `WebauthnDestroyResponse` 分别在 `fortify.php` 的 `features` 配置和 `AppServiceProvider` 的 `boot()` 中通过契约接口绑定到框架，这种"在配置文件或服务提供者中替换实现"的模式不容易通过代码追踪发现。

5. **User Provider 的 `webauthn` 驱动**：`auth.providers.users.driver` 设为 `webauthn` 而非 `eloquent`，这个细微的改动影响了整个用户检索链路，但在代码中没有显式调用点，容易被忽略。
