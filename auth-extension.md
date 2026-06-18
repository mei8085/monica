# Monica 登录认证扩展层代码走向

Monica 的认证体系基于 Laravel 生态的三层扩展框架搭建：**Fortify**（无 UI 的后端认证管线）、**Jetstream**（Inertia 前端脚手架）和 **Laravel Webauthn**（asbiin/laravel-webauthn v5.3.0，Passkey / 安全密钥认证）。在此基础上，项目通过 Socialite 接入了外部 OAuth/SAML 提供商。

下面按代码执行顺序，讲清 **Webauthn 安全密钥登录**的完整链路（从登录页、双因素挑战页到服务端断言校验），严格区分项目内可验证事实与依赖包机制，并对比三条登录管线的关系。

---

## 一、架构总览与关键配置

### 1. 认证 Guard 与 User Provider

`config/auth.php#L38-L43` 定义了唯一的 guard `web`，使用 `session` 驱动。

**关键设计点**：User Provider 的驱动被设为 `webauthn`（而非默认的 `eloquent`）：

`config/auth.php#L62-L66`：
```php
'providers' => [
    'users' => [
        'driver' => 'webauthn',
        'model' => env('AUTH_MODEL', App\Models\User::class),
    ],
],
```

**可验证事实（包源码）**：`webauthn` 驱动由 `WebauthnServiceProvider::passwordLessWebauthn()` 注册，对应类为 `LaravelWebauthn\Auth\EloquentWebAuthnProvider`。它继承 `EloquentUserProvider`，在 `retrieveByCredentials()` 中增加了凭据 ID 查用户的能力，在 `validateCredentials()` 中增加了 WebAuthn 断言校验和密码 fallback 逻辑。

### 2. Fortify Pipeline 自定义

`config/fortify.php#L145-L151` 中 `pipelines.login` 被显式覆盖，替换了 Fortify 默认的登录管线：

```php
'pipelines' => [
    'login' => [
        \App\Actions\Fortify\RedirectIfTwoFactorAuthenticatable::class,
        \Laravel\Fortify\Actions\AttemptToAuthenticate::class,
        \Laravel\Fortify\Actions\PrepareAuthenticatedSession::class,
    ],
],
```

Fortify 默认管线中的 `RedirectIfTwoFactorAuthenticatable` 是包内置的，这里替换成了项目自定义的版本。这是邮箱密码登录路径中 2FA 拦截的关键切入点。

### 3. Webauthn 配置要点

`config/webauthn.php` 的关键参数：
- `enable` 为 `true`，Webauthn 全局启用
- `guard` 为 `web`，与 Fortify 使用同一个 guard
- `prefix` 为 `webauthn`，所有 Webauthn 路由将挂载在 `/webauthn/*` 路径下
- `middleware` 为 `['web']`，应用 session 中间件
- `limiters.login` 为 `'login'`（非 null），这意味着 Webauthn 认证管线中的 `EnsureLoginIsNotThrottled` 会被跳过，改由路由中间件 `throttle:login` 处理限流
- `userless` 由 `WEBAUTHN_USERLESS` 环境变量控制，默认为 `true`——即支持"无用户名登录"（Passkey 一键登录，无需输入邮箱）
- `redirects.login` 指向 `/vaults`
- `views.authenticate` 和 `views.register` 均为 `null`，表示不使用包内置的 Blade 视图，全部走 Inertia 前端
- `session_name` 为 `webauthn_auth`，用于存储已通过 Webauthn 验证的状态
- `model` 使用自定义的 `App\Models\WebauthnKey`（扩展了 `used_at` 字段）
- **没有 `pipelines.login` 配置项**——这是决定 Webauthn 认证管线是否被自定义的关键配置

### 4. Webauthn 中间件注册

`bootstrap/app.php#L29` 将 `LaravelWebauthn\Http\Middleware\WebauthnMiddleware` 注册为别名 `webauthn`。这个中间件是包内部使用的，用于在需要 Webauthn 第二因素验证时拦截请求并重定向到验证流程。

### 5. Webauthn 响应类绑定

`app/Providers/AppServiceProvider.php#L154-L155` 在 boot() 中将 Webauthn 密钥更新和删除的响应类绑定到了项目自定义实现：
```php
Webauthn::updateViewResponseUsing(WebauthnUpdateResponse::class);
Webauthn::destroyViewResponseUsing(WebauthnDestroyResponse::class);
```

**可验证事实**：项目**没有**调用 `Webauthn::loginSuccessResponseUsing()`、`Webauthn::loginViewResponseUsing()`、`Webauthn::authenticateThrough()`、`Webauthn::authenticateUsing()` 中的任何一个。这意味着 Webauthn 登录流程完全使用包内置的默认行为。

### 6. 外部登录提供商

`config/auth.php#L127` 中 `login_providers` 通过 `LOGIN_PROVIDERS` 环境变量动态注入。`config/services.php` 中配置了 azure、facebook、github、google、linkedin、saml2、kanidm、keycloak 共 8 种。各提供商的 Socialite 扩展在 `app/Providers/AppServiceProvider.php#L159-L165` 中通过监听 `SocialiteWasCalled` 事件注册。

---

## 二、Webauthn 路由注册（包源码验证）

**可验证事实（包源码 `routes/routes.php`）**：asbiin/laravel-webauthn v5.3.0 注册了以下路由：

| 路由名 | 方法 | 路径 | 控制器 | 中间件 |
|--------|------|------|--------|--------|
| `webauthn.auth.options` | POST | `/webauthn/auth/options` | `AuthenticateController::create` | `web` + `throttle:login` |
| `webauthn.auth` | POST | `/webauthn/auth` | `AuthenticateController::store` | `web` + `throttle:login` |
| `webauthn.login` | GET | `/webauthn/auth` | `AuthenticateController::create` | `web` + `throttle:login`（**仅当** `config('webauthn.views.authenticate') !== null` 时注册） |
| `webauthn.store.options` | POST | `/webauthn/keys/options` | `WebauthnKeyController::create` | `web` + `auth:web` |
| `webauthn.store` | POST | `/webauthn/keys` | `WebauthnKeyController::store` | `web` + `auth:web` |
| `webauthn.destroy` | DELETE | `/webauthn/keys/{id}` | `WebauthnKeyController::destroy` | `web` + `auth:web` |
| `webauthn.update` | PUT | `/webauthn/keys/{id}` | `WebauthnKeyController::update` | `web` + `auth:web` |
| `webauthn.key.confirm` | POST | `/webauthn/confirm-key` | `ConfirmableKeyController::store` | `web` + `auth:web` |

**注意**：`webauthn.login` GET 路由因为 `config('webauthn.views.authenticate')` 为 `null`，**不会被注册**。项目只使用 `webauthn.auth.options` 和 `webauthn.auth` 两个认证路由。

路由注册代码位于 `WebauthnServiceProvider::configureRoutes()`：
```php
$this->app['router']->group([
    'namespace' => 'LaravelWebauthn\Http\Controllers',
    'domain' => config('webauthn.domain', null),
    'prefix' => config('webauthn.prefix', 'webauthn'),
], fn () => $this->loadRoutesFrom(__DIR__.'/../routes/routes.php'));
```

---

## 三、Webauthn 登录路径一：登录页 Passkey 首因登录

这是"无密码登录"场景——用户在登录页直接使用 Passkey/安全密钥完成认证，无需输入邮箱和密码。

### 第 1 步：登录页加载与公钥挑战生成

```
用户访问 /login
  → LoginController::__invoke()
    ├─ 读取 config('auth.login_providers') 构建外部登录按钮
    ├─ 判断 Webauthn::userless() === true
    │    └─ 调用 Webauthn::prepareAssertion(null) 生成无用户名公钥挑战
    ├─ 读取 cookie('return') 决定是否自动触发 Passkey 登录
    └─ 渲染 Auth/Login.vue，传递 publicKey、userless、autologin、providers 等数据
```

**代码证据**（`app/Http/Controllers/Auth/LoginController.php#L33-L37`）：
```php
if (Webauthn::userless()) {
    $data['publicKey'] = Webauthn::prepareAssertion(null);
    $data['userless'] = true;
    $data['autologin'] = $request->cookie('return') === 'true';
}
```

这里 `Webauthn::prepareAssertion(null)` 传入 `null` 表示**无用户名模式**（userless / discoverable credential），包会生成一个不绑定特定用户的公钥挑战，让浏览器的密码管理器列出所有可用的 Passkey 供用户选择。

### 第 2 步：前端条件判断与 WebauthnLogin 组件激活

`resources/js/Pages/Auth/Login.vue#L29-L31` 中根据后端传入的参数计算 `useSecurityKey`：
```js
const useSecurityKey = computed(
  () => (publicKeyRef.value !== null && props.autologin && platformAuthenticatorAvailable.value) || webauthn.value,
);
```

触发 Passkey 登录的两种方式：
1. **自动触发**：`autologin` 为 true（有 `return` cookie）且设备支持平台认证器（Windows Hello / Touch ID）
2. **手动点击**：用户点击"Sign in with a passkey"按钮，设置 `webauthn.value = true`

满足任一条件后，渲染 `resources/js/Pages/Webauthn/WebauthnLogin.vue` 组件。

### 第 3 步：WebauthnLogin 组件调用浏览器认证

`resources/js/Pages/Webauthn/WebauthnLogin.vue#L35-L49` 的 onMounted() 中：
1. 先用 `browserSupportsWebAuthn()` 检查浏览器支持
2. 检查 `platformAuthenticatorIsAvailable()` 判断是否有平台认证器
3. 调用 `loginWaitForKey(props.publicKey)` 发起认证

`loginWaitForKey()` 方法（`resources/js/Pages/Webauthn/WebauthnLogin.vue#L74-L84`）：
```js
const loginWaitForKey = (publicKey) => {
  processing.value = true;
  browserSupportsWebAuthnAutofill()
    .then((available) =>
      startAuthentication({ optionsJSON: publicKey, useBrowserAutofill: props.autofill && available }),
    )
    .then((data) => webauthnLoginCallback(data))
    .catch((error) => { ... });
};
```

关键参数：
- `useBrowserAutofill: props.autofill`：在登录页场景下 `autofill` 为 `true`，支持浏览器自动填充 Passkey；在 2FA 挑战页为 `false`
- `startAuthentication()` 来自 `@simplewebauthn/browser`，会弹出浏览器的安全密钥/Passkey 选择器

### 第 4 步：前端提交断言数据到服务端

`webauthnLoginCallback()`（`resources/js/Pages/Webauthn/WebauthnLogin.vue#L86-L101`）将浏览器返回的断言数据 POST 到 `route('webauthn.auth')`：
```js
authForm
  .transform(() => ({
    ...data,  // 包含 id, rawId, response(含 clientDataJSON, authenticatorData, signature, userHandle), type
    remember: props.remember ? 'on' : '',
  }))
  .post(route('webauthn.auth'), { ... });
```

### 第 5 步：服务端 Webauthn 认证管线

POST 请求到达 `/webauthn/auth`，由包的 `AuthenticateController::store()` 处理。

**可验证事实（包源码 `AuthenticateController::store()`）**：
```php
public function store(WebauthnLoginRequest $request): LoginSuccessResponse
{
    return $this->loginPipeline($request)->then(function ($request) {
        Webauthn::login($request->user());
        return app(LoginSuccessResponse::class);
    });
}
```

**可验证事实（包源码 `AuthenticateController::loginPipeline()`）**：管线选择逻辑按优先级：
1. 如果 `Webauthn::$authenticateThroughCallback` 不为 null → 使用回调返回的管道数组
2. 如果 `config('webauthn.pipelines.login')` 是数组 → 使用配置的管道数组
3. 否则使用默认管线

**项目实际走第 3 条**（可验证：`config/webauthn.php` 无 `pipelines.login` 键，项目无 `authenticateThrough` 调用）。

默认管线代码：
```php
return (new Pipeline(app()))->send($request)->through(array_filter([
    config('webauthn.limiters.login') !== null ? null : EnsureLoginIsNotThrottled::class,
    AttemptToAuthenticate::class,
    PrepareAuthenticatedSession::class,
]));
```

因为 `config('webauthn.limiters.login')` 为 `'login'`（非 null），`EnsureLoginIsNotThrottled` 被 `array_filter` 移除。**实际管线为 `[AttemptToAuthenticate, PrepareAuthenticatedSession]`**。

### 第 6 步：包内置 AttemptToAuthenticate 的 OR 短路逻辑

**可验证事实（包源码 `LaravelWebauthn\Actions\AttemptToAuthenticate::handle()`）**：
```php
public function handle(Request $request, Closure $next): mixed
{
    if (Webauthn::$authenticateUsingCallback !== null) {
        return $this->handleUsingCustomCallback($request, $next);
    }
    if ($this->attemptValidateAssertion($request)
        || $this->attemptLogin($this->filterCredentials($request), $request->boolean('remember'))) {
        return $next($request);
    }
    $this->throwFailedAuthenticationException($request);
    return null;
}
```

项目没有设置 `Webauthn::$authenticateUsingCallback`，所以走 OR 短路逻辑：

**模式 A：`attemptValidateAssertion()`**（`$request->user()` 非空时）
- 检查 `$request->user()` 是否非空
- 如果已登录，调用 `WebauthnFacade::validateAssertion($user, $credentials)` 验证断言
- 用于"已登录用户再次验证 WebAuthn"的场景（如敏感操作确认）

**模式 B：`attemptLogin()`**（`$request->user()` 为 null 时）
- 调用 `$this->guard->attempt($credentials, $remember)`
- `$credentials` 来自 `filterCredentials()`：`$request->only(['id', 'rawId', 'response', 'type'])`
- Guard 使用 `webauthn` 驱动的 `EloquentWebAuthnProvider`

在**登录页 Passkey 登录场景**中，用户尚未登录，`$request->user()` 为 null，`attemptValidateAssertion()` 返回 false，走 `attemptLogin()` 分支。

### 第 7 步：EloquentWebAuthnProvider 的凭据 ID 查找与断言校验

**可验证事实（包源码 `EloquentWebAuthnProvider`）**：

`retrieveByCredentials()`：
```php
public function retrieveByCredentials(array $credentials): ?User
{
    if ($this->isSignedChallenge($credentials)) {
        try {
            $webauthnKey = (Webauthn::model())::where('credentialId', Base64UrlSafe::encode(Base64::decode($credentials['id'])))
                ->orWhere('credentialId', Base64UrlSafe::encodeUnpadded(Base64::decode($credentials['id'])))
                ->firstOrFail();
            return $this->retrieveById($webauthnKey->user_id);
        } catch (ModelNotFoundException $e) {
            return null;
        }
    }
    return parent::retrieveByCredentials($credentials);
}
```

当 `$credentials` 包含 `id`、`rawId`、`type`、`response`（即 WebAuthn 断言数据）时，`isSignedChallenge()` 返回 true，Provider 根据 `credentialId` 查找 `webauthn_keys` 表，找到关联的密钥记录，再通过 `user_id` 返回用户。

`validateCredentials()`：
```php
public function validateCredentials(User $user, array $credentials): bool
{
    if ($this->isSignedChallenge($credentials)
        && Webauthn::validateAssertion($user, $credentials)) {
        WebauthnLogin::dispatch($user, true);
        return true;
    }
    if ($this->fallback) {
        return parent::validateCredentials($user, $credentials);
    }
    return false;
}
```

验证通过后派发 `WebauthnLogin` 事件（带第二个参数 `true`，表示来自断言校验）。如果凭据不是 WebAuthn 格式且 `fallback` 为 `true`（默认），则回退到密码校验。

`$guard->attempt()` 内部会依次调用 `retrieveByCredentials()` 和 `validateCredentials()`，两者都通过后调用 `Auth::login()` 建立登录 session。

### 第 8 步：管线后续与登录完成

管线后续管道：
1. **`PrepareAuthenticatedSession`**（包内置）：调用 `$request->session()->regenerate()` 再生 session、清除速率限制
2. **管线 then 回调**：调用 `Webauthn::login($request->user())`

**可验证事实（包源码 `Webauthn::login()`）**：
```php
public static function login(?User $user): void
{
    session([static::sessionName() => true]);
    if ($user !== null) {
        WebauthnLogin::dispatch($user);
    }
}
```

`Webauthn::login()` 做两件事：
- 将 `webauthn_auth = true` 写入 session（标记"已通过 Webauthn 验证"）
- 派发 `WebauthnLogin` 事件（不带第二个参数，与 `validateCredentials` 中的派发不同）

**注意**：`WebauthnLogin` 事件在此流程中被**派发了两次**——一次在 `EloquentWebAuthnProvider::validateCredentials()` 中（带 `true` 参数），一次在 `Webauthn::login()` 中（不带 `true` 参数）。

3. **`LoginSuccessResponse`**（包内置，未被项目覆盖）：对于 Inertia 请求（wantsJson），返回 JSON `{ result: true, callback: '/vaults' }`；对于普通请求，重定向到 `/vaults`。

`Login` 事件触发后，`app/Listeners/LoginListener.php#L15-L19` 会检查：如果用户勾选了"记住我"且拥有 Webauthn 密钥，则设置 `return` cookie（有效期一年），下次自动触发 Passkey 登录。

此外，`app/Providers/AppServiceProvider.php#L157` 订阅了包内的 `LoginViaRemember` 监听器，用于处理 Laravel 记住我 cookie 触发的自动登录中的 Webauthn 相关逻辑。

---

## 四、Webauthn 登录路径二：双因素挑战页的第二因素验证

这是"用户先输入邮箱密码，然后再用安全密钥做第二因素"的场景。

### 第 1 步：从邮箱密码登录被 2FA 拦截

用户提交邮箱密码到 `/login`，请求进入 Fortify Pipeline：
1. **管道 1**：`app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php#L28-L38`
   - `validateCredentials()` 通过 email 查用户并验证密码（此时不登录，只验证凭据）
   - 判断是否需要 2FA（`app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php#L32-L34`）：
     ```php
     if ((optional($user)->two_factor_secret && ! is_null(optional($user)->two_factor_confirmed_at))
         || Webauthn::enabled($user)) {
         return $this->twoFactorChallengeResponse($request, $user);
     }
     ```
   - `Webauthn::enabled($user)` 来自 Laravel Webauthn 包，检查用户是否有关联的 WebauthnKey 记录
   - 如果需要 2FA：将 `login.id`（用户 ID）和 `login.remember`（记住我标记）写入 session，触发 `TwoFactorAuthenticationChallenged` 事件，重定向到 `two-factor.login`

2. **管道 2**：只有无需 2FA 时才会执行 `AttemptToAuthenticate` 完成真正登录

**外部 OAuth 登录路径同样做了 2FA 检查**：`app/Actions/AttemptToAuthenticateSocialite.php#L51-L54` 中用了完全相同的判断逻辑，如果用户有 Webauthn 或 TOTP 就重定向到挑战页，然后在 `twoFactorChallengeResponse()` 里把 `login.remember` 从 session 中 pull 出来重新写入。

### 第 2 步：双因素挑战页渲染

Fortify 的 `two-factor.login` 路由（GET）绑定到 `app/Actions/Fortify/TwoFactorChallengeView.php`——它实现了 `TwoFactorChallengeViewResponse` 契约，替代了 Fortify 默认的 Blade 视图。

`TwoFactorChallengeView::toResponse()`（`app/Actions/Fortify/TwoFactorChallengeView.php#L18-L32`）：
```php
public function toResponse($request)
{
    $userId = $request->session()->get('login.id');
    $user = User::find($userId);

    $data = [];
    if ($user !== null) {
        $data['publicKey'] = Webauthn::prepareAssertion($user);
    }

    return Inertia::render('Auth/TwoFactorChallenge', $data + [
        'twoFactor' => optional($user)->two_factor_secret && ! is_null(optional($user)->two_factor_confirmed_at),
        'remember' => $request->session()->get('login.remember'),
    ])->toResponse($request);
}
```

**与登录页的关键区别**：此处调用 `Webauthn::prepareAssertion($user)` 传入了**具体用户对象**，生成的是针对该用户的断言挑战（非 userless 模式）。浏览器只会查找属于该用户的安全密钥，不会列出所有 Passkey。

**`login.id` session 的作用**：仅用于 `TwoFactorChallengeView` 渲染挑战页时获取用户信息，以便调用 `Webauthn::prepareAssertion($user)` 生成绑定该用户的公钥挑战。`login.id` **不参与**后续的 WebAuthn 断言校验流程。

### 第 3 步：双因素挑战页前端逻辑

`resources/js/Pages/Auth/TwoFactorChallenge.vue` 根据后端参数渲染：

1. **如果 `publicKey !== null`**（用户有 Webauthn 密钥）：渲染 Webauthn 挑战区
   - 展示安全密钥提示文本
   - 嵌入 `WebauthnLogin.vue` 组件，但 `autofill` 设为 `false`（不使用浏览器自动填充，因为用户身份已确定）
   - WebauthnLogin 组件拿到断言数据后，同样 POST 到 `route('webauthn.auth')`

2. **如果 `twoFactor === true`**（用户启用了 TOTP）：渲染 TOTP 验证码输入区
   - 支持普通验证码与恢复码切换
   - 提交到 `route('two-factor.login')`（Fortify 的 POST 路由），由 Fortify 内置的 `TwoFactorAuthenticateAction` 处理

两种方式可以**同时存在**：用户既有用 TOTP 也有 Webauthn，可以二选一通过。

### 第 4 步：服务端断言校验（2FA 场景）

POST 到 `webauthn.auth` 后，进入与首因登录**完全相同**的管线和控制器逻辑。

关键问题：此时用户尚未被 guard 登录（Fortify 只验证了密码，未调用 `Auth::login()`），`$request->user()` 为 null。流程如何找到正确用户？

**可验证事实（包源码）**：`AttemptToAuthenticate::handle()` 中：
1. `attemptValidateAssertion()` → `$request->user()` 为 null → 返回 false
2. `attemptLogin()` → 调用 `$this->guard->attempt($credentials, $remember)`
3. `$credentials` 为 `{id, rawId, response, type}` — 来自前端 WebAuthn 断言
4. `EloquentWebAuthnProvider::retrieveByCredentials()` 检测到 `isSignedChallenge()` 为 true，**根据 `credentials['id']`（凭据 ID）查 `webauthn_keys` 表**，找到密钥记录的 `user_id`，再通过 `retrieveById()` 返回用户
5. `EloquentWebAuthnProvider::validateCredentials()` 调用 `Webauthn::validateAssertion($user, $credentials)` 验证签名
6. 验证通过后 `$guard->attempt()` 调用 `Auth::login()` 建立登录 session

**核心结论**：2FA 场景下用户的查找**完全依赖断言数据中的凭据 ID**，而非 session 中的 `login.id`。`login.id` 仅用于渲染挑战页时生成绑定用户的公钥参数。

后续步骤与首因登录一致：`PrepareAuthenticatedSession` → `Webauthn::login()` → `LoginSuccessResponse`。

---

## 五、`AttemptToAuthenticateWebauthn` 的定位与事实校准

### 项目内的引用关系

`app/Actions/AttemptToAuthenticateWebauthn.php` 在项目代码中**仅被以下文件引用**：
- 其自身类定义文件
- `tests/Unit/Actions/AttemptToAuthenticateWebauthnTest.php`（单元测试）

**可验证事实**：该类没有出现在任何配置文件、服务提供者、控制器或其他 Action 中。具体来说：
- `config/webauthn.php` 无 `pipelines.login` 键 → 未通过配置注入管线
- 项目无 `Webauthn::authenticateThrough()` 调用 → 未通过回调注入管线
- 项目无 `Webauthn::authenticateUsing()` 调用 → 未通过回调替换认证逻辑
- 该类不被任何服务提供者绑定为契约实现

### 与包内置 `AttemptToAuthenticate` 的关系

**可验证事实（包源码）**：项目的 `AttemptToAuthenticateWebauthn` 是包内置 `LaravelWebauthn\Actions\AttemptToAuthenticate` 的**简化副本**：
- 相同的 OR 短路逻辑：`attemptValidateAssertion() || attemptLogin()`
- 相同的 `filterCredentials()`、`attemptLogin()`、`fireFailedEvent()` 方法
- 差异：项目版本**移除了** `authenticateUsingCallback` 分支，将 `trans()` 改为 `trans_ignore()`

由于管线未被自定义，实际运行时使用的是**包内置版本**，`AttemptToAuthenticateWebauthn` 是未被接入管线的**死代码**。它保留了完整的认证逻辑和单元测试，但不会在运行时被调用。

---

## 六、Webauthn 密钥注册（安全密钥创建）流程

用户登录后，在个人设置页面可以添加新的安全密钥。这是 Webauthn 的"注册"（Attestation）流程，与登录的"断言"（Assertion）是两个不同阶段。

### 第 1 步：用户设置页加载密钥管理

`app/Actions/Jetstream/UserProfile.php#L25-L32` 向 Inertia 页面注入 `webauthnKeys` 列表：
```php
$webauthnKeys = $request->user()->webauthnKeys
    ->map(fn (WebauthnKey $key) => [
        'id' => $key->id,
        'name' => $key->name,
        'type' => $key->type,
        'last_used' => optional($key->used_at)->diffForHumans(),
    ])
    ->toArray();
```

### 第 2 步：点击"注册新密钥"按钮

在 `resources/js/Pages/Webauthn/WebauthnKeys.vue#L177-L181`，点击按钮首先经过 `JetConfirmsPassword` 密码确认（Jetstream 的安全确认机制），然后弹出注册模态。

### 第 3 步：获取注册选项（Attestation Options）

`resources/js/Pages/Webauthn/Partials/RegisterKey.vue#L41-L49` 中，输入密钥名称并提交后：
```js
axios.post(route('webauthn.store.options'))
  .then((response) => {
    registerWaitForKey(response.data.publicKey);
  })
```

路由 `webauthn.store.options` 由 Laravel Webauthn 包注册（需要 `auth:web` 中间件），返回用于创建新密钥的公钥挑战参数。

### 第 4 步：浏览器生成新密钥对并提交注册响应

`registerWaitForKey()`（`resources/js/Pages/Webauthn/WebauthnKeys.vue#L68-L74`）调用 `@simplewebauthn/browser` 的 `startRegistration(publicKey)`，浏览器弹出安全密钥交互提示，生成新密钥对后，由 `webauthnRegisterCallback()` 将注册响应连同密钥名称一起 POST 到 `route('webauthn.store')`。

### 第 5 步：服务端保存密钥

`webauthn.store` 路由由包处理，内部流程：
1. 校验注册响应的签名（证明认证器是真实的）
2. 将公钥和密钥元数据存入 `webauthn_keys` 表（使用自定义的 `App\Models\WebauthnKey` 模型，扩展了 `used_at` 字段）
3. 返回响应时使用自定义的 `WebauthnUpdateResponse`（在 `AppServiceProvider` 中绑定），Inertia 请求下重定向到 `/user/profile`

密钥更新（`webauthn.update`，修改名称）和删除（`webauthn.destroy`）也使用了类似的响应类绑定。

---

## 七、三条登录管线的关系与差异对比

| 维度 | 邮箱密码登录 | Webauthn Passkey 登录 | 外部 OAuth/SAML 登录 |
|------|-------------|---------------------|---------------------|
| **前端入口** | `Login.vue` 表单提交 | `Login.vue` / `TwoFactorChallenge.vue` → `WebauthnLogin.vue` | `ExternalProviders.vue` 按钮点击 |
| **POST 路由** | `POST /login` (Fortify) | `POST /webauthn/auth` (Laravel Webauthn 包) | `GET /auth/{driver}` → 第三方 → `GET/POST /auth/{driver}/callback` |
| **后端管线** | Fortify Pipeline（配置于 `config/fortify.php#L145-L151`） | Webauthn 包内置 Pipeline（`AuthenticateController::loginPipeline()`，**未被项目自定义**） | 独立 `Illuminate\Pipeline\Pipeline` 实例（在 `SocialiteCallbackController#L62-L66` 中手动构建） |
| **实际认证动作** | `RedirectIfTwoFactorAuthenticatable`（项目自定义，替换 Fortify 默认） | `LaravelWebauthn\Actions\AttemptToAuthenticate`（**包内置，未被替换**） | `AttemptToAuthenticateSocialite`（项目自建） |
| **用户密码校验** | `guard->validateCredentials()` 在 2FA 拦截中完成 | 不涉及（用密钥断言替代） | 不涉及（由第三方保证） |
| **2FA 检查位置** | Fortify Pipeline 的第一个管道（登录前拦截） | **不检查**——Webauthn 本身既是首因也可作为二因 | 自定义管线 `AttemptToAuthenticateSocialite` 中，在 `guard->login()` 之前 |
| **2FA 判断条件** | TOTP 已启用 OR Webauthn 有密钥（两处代码完全一致） | 无——Webauthn 包自行根据凭据 ID 查找用户 | 同左 |
| **用户查找方式** | `EloquentWebAuthnProvider::retrieveByCredentials()` 按 email 查（回退到密码） | `EloquentWebAuthnProvider::retrieveByCredentials()` 按 credentialId 查 `webauthn_keys` 表 | 通过 `UserToken` 按 driver_id + driver 查关联用户 |
| **登录成功后** | `AttemptToAuthenticate` → `PrepareAuthenticatedSession` | `PrepareAuthenticatedSession` → `Webauthn::login()` 写 session 标记 → `LoginSuccessResponse` | `PrepareAuthenticatedSession`（复用 Fortify 内置） |
| **WebAuthn session 标记** | 不写入（非 WebAuthn 认证） | `webauthn_auth = true`（由 `Webauthn::login()` 写入） | 不写入 |
| **触发 Login 事件** | Fortify 内置动作触发 | `$guard->attempt()` → `Auth::login()` 触发 + `LoginListener` 设置 `return` cookie | `guard->login()` 触发 |
| **速率限制** | Fortify 的 `login` limiter | 路由中间件 `throttle:login`（因为 `config('webauthn.limiters.login')` 非空，包内置 `EnsureLoginIsNotThrottled` 被跳过） | `oauth2-socialite` limiter（`AppServiceProvider#L152`，每分钟 5 次） |

### 三者的连接点

1. **共享同一个 `web` guard 和 `EloquentWebAuthnProvider`**：无论走哪条路径，最终都通过同一个 guard 建立 session，登录态完全互通。Provider 的 `fallback = true`（默认）意味着它同时支持 WebAuthn 断言和密码两种凭据格式。

2. **共享同一套 2FA 判断逻辑**：
   - `app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php#L32-L34`
   - `app/Actions/AttemptToAuthenticateSocialite.php#L51-L54`
   两处用了完全相同的条件表达式：
   ```php
   (optional($user)->two_factor_secret && ! is_null(optional($user)->two_factor_confirmed_at))
   || Webauthn::enabled($user)
   ```

3. **2FA 挑战页是统一出口**：无论从邮箱密码还是 OAuth 被拦截，最终都跳转到 `two-factor.login`，该页面同时支持 TOTP 和 Webauthn。

4. **Webauthn 可以作为首因（Passkey）或第二因素**：
   - 首因：登录页 Passkey，用户跳过邮箱密码输入，直接使用安全密钥
   - 二因：邮箱密码或 OAuth 登录后，再使用安全密钥通过 2FA 挑战

5. **两种场景共享同一个服务端认证端点**：`POST /webauthn/auth` → `AuthenticateController::store()`。首因登录和 2FA 二因验证走的是**完全相同的代码路径**——区别仅在于前端传入的 `publicKey` 参数不同（userless vs 绑定用户），以及 `autofill` 参数不同。

---

## 八、注册场景

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

**代码走向细节**：

1. 前端 `resources/js/Pages/Auth/Register.vue#L27-L31` 通过 `form.post(route('register'))` 提交到 Fortify 注册端点。

2. Fortify 根据 `config/fortify.php#L134` 中 `Features::registration()` 的配置，将请求路由到 `CreateNewUser` Action。这个 Action 实现了 `CreatesNewUsers` 契约。

3. `app/Actions/Fortify/CreateNewUser.php#L19-L35` 先做输入校验（first_name, last_name, email, password, terms），然后委托给领域服务 `app/Domains/Settings/CreateAccount/Services/CreateAccount.php#L38-L49`。

4. `CreateAccount` 做了两次校验（`validateRules` 是 BaseService 的逻辑），然后：
   - 创建 `Account` 记录（设置默认存储上限）
   - 创建 `User` 记录（密码通过 `Hash::make` 加密，**若 password 为 null 则允许无密码用户**——这是为 OAuth 首次登录即注册的场景预留的）
   - 派发 `SetupAccount` 队列任务，异步初始化账户的默认数据（模板、模块、性别、代词、关系类型等，共 100+ 项默认记录）

5. 注册页面的可见性由 `app/Helpers/SignupHelper.php#L22-L25` 控制：当配置 `monica.disable_signup` 为 true 且数据库中已有至少一个 Account 时，注册入口关闭。

6. `app/Http/Controllers/Auth/RegisterController.php` 只负责渲染注册页面视图，向 Vue 传递外部登录提供商列表。

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

邀请注册不经过 Fortify 管线。`app/Http/Controllers/Auth/AcceptInvitationController.php#L32-L55` 的 `store` 方法自行校验后调用 `AcceptInvitation` 领域服务创建用户，然后直接调用 `Auth::login()` 登录——**绕过了双因素验证检查**（这是一个值得注意的设计点）。

### 入口 C：OAuth 首次登录即注册

在 `app/Actions/AttemptToAuthenticateSocialite.php#L112-L119` 的 `getUserOrCreate()` 中，如果 Socialite 返回的用户在系统中没有对应记录（无 UserToken），且当前没有用户登录，则调用 `createUser()`：

```php
private function createUser(SocialiteUser $socialite): User
{
    $data = [
        'email' => $socialite->getEmail(),
        'first_name' => $names[0],
        'last_name' => $names[1] ?? $names[0],
        'terms' => true,
    ];
    return tap(app(CreateNewUser::class)->create($data), fn (User $user) => event(new Registered($user)));
}
```

复用了 `CreateNewUser`，但**不传递 password**，因此创建的用户 `password` 字段为 null（这就是为什么 `CreateAccount` 允许 password 为 null 的原因）。

---

## 九、扩展层关键文件索引

| 文件（仓库相对路径） | 角色 |
|------|------|
| `config/fortify.php` | Fortify 功能与登录管线配置，自定义了 `pipelines.login` |
| `config/auth.php` | Guard、User Provider（`webauthn` driver）、外部登录提供商配置 |
| `config/webauthn.php` | Webauthn 全局配置（路由前缀、userless、redirects、limiters 等），**无 `pipelines.login` 键** |
| `config/jetstream.php` | Jetstream 功能开关 |
| `config/services.php` | 各 OAuth/SAML 提供商凭证配置 |
| `bootstrap/app.php` | WebauthnMiddleware 别名注册 |
| `app/Providers/AppServiceProvider.php` | Socialite 提供商事件监听注册、Webauthn 响应类绑定（仅 update/destroy，**未覆盖 login 管线**）、LoginViaRemember 订阅 |
| `app/Providers/AuthServiceProvider.php` | Gate 定义（administrator、vault-viewer 等权限） |
| `app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php` | **Fortify 登录管线自定义 2FA 拦截点**，替换包内置版本 |
| `app/Actions/Fortify/TwoFactorChallengeView.php` | 2FA 挑战页视图响应（含 Webauthn 公钥准备），替代 Fortify 默认 |
| `app/Actions/Fortify/CreateNewUser.php` | 注册 Action，实现 Fortify 的 CreatesNewUsers 契约 |
| `app/Actions/AttemptToAuthenticateWebauthn.php` | **死代码**——是包内置 `AttemptToAuthenticate` 的简化副本，未被接入任何管线，仅有单元测试引用 |
| `app/Actions/AttemptToAuthenticateSocialite.php` | OAuth 登录管线核心（含注册/关联已有账户/2FA 检查） |
| `app/Http/Controllers/Auth/LoginController.php` | 登录页渲染（userless 模式下生成无用户名断言公钥） |
| `app/Http/Controllers/Auth/RegisterController.php` | 注册页渲染 |
| `app/Http/Controllers/Auth/SocialiteCallbackController.php` | OAuth 登录/回调控制器，手动构建独立 Pipeline |
| `app/Http/Controllers/Auth/AcceptInvitationController.php` | 邀请注册控制器（直接 Auth::login，绕过 2FA 检查） |
| `app/Http/Controllers/Profile/WebauthnUpdateResponse.php` | Webauthn 密钥创建/更新后的 Inertia 重定向响应 |
| `app/Http/Controllers/Profile/WebauthnDestroyResponse.php` | Webauthn 密钥删除后的 Inertia 重定向响应 |
| `app/Domains/Settings/CreateAccount/Services/CreateAccount.php` | 注册领域服务（创建 Account + User + 派发 SetupAccount） |
| `app/Domains/Settings/CreateAccount/Jobs/SetupAccount.php` | 账户初始化队列任务（100+ 条默认数据） |
| `app/Models/User.php` | 用户模型，组合 `TwoFactorAuthenticatable` + `WebauthnAuthenticatable` trait |
| `app/Models/WebauthnKey.php` | Webauthn 密钥模型，扩展 `used_at` 字段 |
| `app/Models/UserToken.php` | 外部登录提供商关联令牌模型（OAuth1/OAuth2 格式区分） |
| `app/Listeners/LoginListener.php` | Login 事件监听（设置 `return` cookie 启用 Passkey 自动登录） |
| `app/Helpers/SignupHelper.php` | 判断注册入口是否开放 |
| `resources/js/Pages/Auth/Login.vue` | 登录页前端（邮箱密码表单 + Passkey 区 + 外部登录按钮区） |
| `resources/js/Pages/Auth/Register.vue` | 注册页前端 |
| `resources/js/Pages/Auth/TwoFactorChallenge.vue` | 2FA 挑战页前端（Webauthn 挑战 + TOTP 输入） |
| `resources/js/Pages/Auth/ExternalProviders.vue` | 外部登录提供商按钮组件 |
| `resources/js/Pages/Webauthn/WebauthnLogin.vue` | Webauthn 登录组件（调用 `@simplewebauthn/browser`，POST 到 `webauthn.auth`） |
| `resources/js/Pages/Webauthn/WebauthnKeys.vue` | 用户设置页的密钥管理组件 |
| `resources/js/Pages/Webauthn/Partials/RegisterKey.vue` | 新密钥注册向导（POST 到 `webauthn.store.options` + `webauthn.store`） |
| `resources/js/Pages/Webauthn/WebauthnTest.vue` | 密钥确认组件（用于 `webauthn.key.confirm`，应用内敏感操作再验证） |

---

## 十、事实层级标注：项目可验证 vs 包源码可验证 vs 推断

| 描述 | 事实层级 |
|------|---------|
| `config/webauthn.php` 无 `pipelines.login` 键 | 项目可验证 |
| 项目无 `authenticateThrough` / `authenticateUsing` 调用 | 项目可验证（grep 验证） |
| `AttemptToAuthenticateWebauthn` 仅被自身和单元测试引用 | 项目可验证（grep 验证） |
| `EloquentWebAuthnProvider` 是 `webauthn` 驱动的实现类 | 包源码可验证（`WebauthnServiceProvider::passwordLessWebauthn()`） |
| `EloquentWebAuthnProvider::retrieveByCredentials()` 按 credentialId 查用户 | 包源码可验证 |
| `EloquentWebAuthnProvider::validateCredentials()` 同时支持 WebAuthn 断言和密码 fallback | 包源码可验证 |
| WebAuthn 认证管线实际为 `[AttemptToAuthenticate, PrepareAuthenticatedSession]` | 包源码可验证（`loginPipeline()` + 项目 `limiters.login` 配置） |
| `Webauthn::login()` 写 `webauthn_auth = true` 到 session 并派发 `WebauthnLogin` 事件 | 包源码可验证 |
| `WebauthnLogin` 事件在认证流程中被派发两次 | 包源码可验证（`validateCredentials` 一次 + `Webauthn::login()` 一次） |
| `LoginSuccessResponse` 对 Inertia 请求返回 JSON、对普通请求重定向到 `/vaults` | 包源码可验证 |
| 2FA 场景下 `login.id` 不参与 WebAuthn 断言校验 | 包源码可验证（`AuthenticateController::store()` 和 `AttemptToAuthenticate` 中无 `login.id` 读取逻辑） |
| `login.id` 仅用于 `TwoFactorChallengeView` 生成绑定用户的公钥挑战 | 项目可验证（`TwoFactorChallengeView` 源码中读取 `login.id` 仅用于 `User::find()` + `Webauthn::prepareAssertion($user)`） |

---

## 十一、代码走向的"为什么让人困惑"

1. **`AttemptToAuthenticateWebauthn` 是死代码但看起来像核心动作**：它完整实现了认证逻辑，有单元测试，放在 `app/Actions/` 目录下——但实际运行时使用的是包内置的同名类 `LaravelWebauthn\Actions\AttemptToAuthenticate`。项目没有通过任何机制将自定义类接入管线，导致阅读者误以为它参与了认证流程。

2. **三条管线各自独立**：邮箱密码走 Fortify Pipeline（`config/fortify.php`），OAuth 走独立实例化的 `Illuminate\Pipeline\Pipeline`（`SocialiteCallbackController` 内手动 new），Webauthn 走包内置 Pipeline（`AuthenticateController::loginPipeline()`）。三条管线的配置位置、管道顺序、2FA 检查实现方式都不同，需要分别追踪。

3. **2FA 检查逻辑重复实现**：完全相同的条件表达式（TOTP OR Webauthn）在 `RedirectIfTwoFactorAuthenticatable` 和 `AttemptToAuthenticateSocialite` 中各写了一份。

4. **自定义与内置管道混用**：Fortify 管线中只有第一个管道被项目自定义，后两个仍使用 Fortify 内置版本；Webauthn 管线则**完全没有被自定义**。这种"有的换有的不换"的模式容易让人误以为 Webauthn 管线也被换了。

5. **隐式契约绑定**：`TwoFactorChallengeView`（通过 Fortify features 配置自动绑定）、`WebauthnUpdateResponse` / `WebauthnDestroyResponse`（在 AppServiceProvider 的 `updateViewResponseUsing` / `destroyViewResponseUsing` 中绑定）等都属于"在配置文件或服务提供者中替换接口实现"的模式，IDE 的"查找引用"通常无法直接跳到调用方。

6. **User Provider 的 `webauthn` driver**：`config/auth.php` 中 `providers.users.driver` 从默认的 `eloquent` 改成了 `webauthn`，这改变了 `Guard::attempt()` 的内部行为——从纯密码校验变成"密码或断言二选一"。但因为改动只在配置文件中，代码搜索无法直接找到关联。

7. **Webauthn 路由全部由包注册**：项目代码中看不到 `/webauthn/auth`、`/webauthn/store` 等路由的定义，它们由 `WebauthnServiceProvider::configureRoutes()` 自动注册，挂载在 `config/webauthn.php#L55` 的 `prefix` 下。对于不熟悉该包的开发者，找路由会有些"魔法"感。

8. **`WebauthnLogin` 事件被派发两次**：一次在 `EloquentWebAuthnProvider::validateCredentials()` 中（带 `true` 第二参数），一次在 `Webauthn::login()` 中（不带）。两次派发的语义差异需要查阅包源码才能理解，项目代码中无法直接看到这一行为。

---

## 十二、敏感操作确认链路（ConfirmsPassword + WebauthnTest + webauthn.key.confirm）

**链路状态：✅ 真实生效**

这是用户在应用内执行敏感操作（删除密钥、启用 2FA 等）时的二次身份验证流程，支持"密码确认"和"安全密钥确认"两种方式。

### 12.1 前端入口：ConfirmsPassword.vue 的使用

`resources/js/Components/Jetstream/ConfirmsPassword.vue` 在项目中被 8 处使用：

| 位置 | 触发的操作 |
|------|----------|
| `resources/js/Pages/Webauthn/WebauthnKeys.vue#L164` | 删除安全密钥 |
| `resources/js/Pages/Webauthn/WebauthnKeys.vue#L177` | 注册新安全密钥 |
| `resources/js/Pages/Profile/Partials/TwoFactorAuthenticationForm.vue#L205` | 启用双因素认证 |
| `resources/js/Pages/Profile/Partials/TwoFactorAuthenticationForm.vue#L213` | 确认双因素认证 |
| `resources/js/Pages/Profile/Partials/TwoFactorAuthenticationForm.vue#L224` | 重新生成恢复码 |
| `resources/js/Pages/Profile/Partials/TwoFactorAuthenticationForm.vue#L230` | 显示恢复码 |
| `resources/js/Pages/Profile/Partials/TwoFactorAuthenticationForm.vue#L236` | 禁用双因素认证 |
| `resources/js/Pages/Profile/Partials/TwoFactorAuthenticationForm.vue#L242` | 确认禁用双因素 |

### 12.2 ConfirmsPassword.vue 的条件分支逻辑

`resources/js/Components/Jetstream/ConfirmsPassword.vue` 的核心逻辑：

```
点击触发按钮
  → startConfirmingPassword()
    → GET password.confirmation（检查 session 中 auth.password_confirmed_at 是否在有效期内）
    ├─ 已确认 → 直接 emit('confirmed')
    └─ 未确认 → 弹出确认对话框
         ├─ hasKey === true（用户有 Webauthn 密钥）
         │    └─ 渲染"Confirm your passkey or security key"按钮
         │       → 点击 → webauthn.start() → 触发 WebauthnTest 组件
         └─ 始终渲染密码输入框（无论是否有密钥）
              → 输入密码 → confirmPassword()
                 → POST password.confirm（Jetstream 内置路由）
                 → 成功后写 session auth.password_confirmed_at
                 → confirm() → emit('confirmed')
```

**hasKey 的来源**（`app/Http/Middleware/HandleInertiaRequests.php#L34-L40`）：Inertia 全局共享属性，判断当前登录用户是否拥有至少一条 WebauthnKey 记录。

### 12.3 WebauthnTest.vue 的密钥确认流程

`resources/js/Pages/Webauthn/WebauthnTest.vue` 的两步流程：

```
webauthn.start() 被调用
  ├─ 第 1 步：POST route('webauthn.auth.options')
  │     → 包的 AuthenticateController::create() 返回断言挑战公钥
  │     → 注意：与登录不同，此处 POST 不带任何用户参数，
  │       因为当前用户已登录，包内部从 Auth::user() 获取用户身份
  │
  └─ 第 2 步：调用浏览器 startAuthentication({ optionsJSON: publicKey })
       → 用户完成安全密钥交互
       → POST route('webauthn.key.confirm')，提交断言数据
       → 服务端返回 JsonResponse('', 201)（空 body + 201 状态码）
       → 前端 .then(() => { emit('success') }) 检测到 2xx 即触发
```

**成功回调传递链**（3 层 emit 传递）：

```
WebauthnTest.vue
  → emit('success')                                    // 第 1 层：密钥确认成功
  → ConfirmsPassword.vue @success="confirm()"
    → confirm() { closeModal(); nextTick().then(() => emit('confirmed')) }  // 第 2 层：关闭弹窗
    → 父组件 @confirmed="实际操作"
      → 执行删除密钥、启用 2FA 等敏感操作                // 第 3 层：执行实际动作
```

密码确认走的是同样的传递链，只是第 1 层从 `WebauthnTest.emit('success')` 替换为 `confirmPassword()` 中 `axios.post('password.confirm')` 成功后的 `confirm()` 调用。两条路径的区别在于服务端写入 `auth.password_confirmed_at` 的位置不同（密码确认由 `PasswordConfirmationController` 写入，密钥确认由 `ConfirmableKeyController` 写入），但前端最终都汇合到同一个 `confirm()` 方法关闭弹窗并 `emit('confirmed')`。

### 12.4 服务端：ConfirmableKeyController（包源码验证）

**可验证事实（包源码 `ConfirmableKeyController::store()`）**：

```php
public function store(Request $request): Responsable
{
    $confirmed = app(ConfirmKey::class)(
        $this->guard, $request
    );

    if ($confirmed) {
        $request->session()->put('auth.password_confirmed_at', Date::now()->unix());
    }

    return $confirmed
        ? app(KeyConfirmedResponse::class)
        : app(FailedKeyConfirmedResponse::class);
}
```

关键点：
- 调用 `ConfirmKey` action 执行断言验证
- 验证成功后写入 `auth.password_confirmed_at` 到 session（与密码确认使用同一个 session 键）
- 验证失败时返回 `FailedKeyConfirmedResponse`：JSON 请求抛 `ValidationException`（422 状态码，字段 `key`，消息 `__('Invalid key.')`）；普通请求 `back()->withErrors()`

**可验证事实（包源码 `KeyConfirmedResponse::toResponse()`）**：

```php
public function toResponse($request)
{
    return $request->wantsJson()
        ? new JsonResponse('', 201)
        : redirect()->intended(Webauthn::redirects('key-confirmation'));
}
```

- **JSON / Inertia 请求**：返回 `JsonResponse('', 201)`——**空 body** + HTTP 201 Created 状态码。前端 `WebauthnTest.vue#L57` 的 `.then(() => { emit('success') })` 只关心请求成功（2xx），不读取响应体内容
- **普通请求**：重定向到 `config('webauthn.redirects.key-confirmation')`（项目中配置为 `/user/profile`）

### 12.5 ConfirmKey action（包源码验证）

**可验证事实（包源码 `ConfirmKey::__invoke()`）**：

```php
public function __invoke(StatefulGuard $guard, Request $request): bool
{
    return is_null(Webauthn::$confirmKeyUsingCallback)
        ? $guard->attempt($this->getParams($request))
        : $this->confirmKeyUsingCustomCallback($request);
}
```

项目未设置 `$confirmKeyUsingCallback`，因此走 `$guard->attempt()` 分支——**与登录时的断言校验使用完全相同的路径**（`EloquentWebAuthnProvider::retrieveByCredentials()` → `validateCredentials()`）。

**`$guard->attempt()` 在已登录用户上执行的副作用**：

1. `retrieveByCredentials()` 根据断言中的 credentialId 查找密钥 → 找到关联用户
2. `validateCredentials()` 调用 `Webauthn::validateAssertion()` 验证断言签名 → 验证通过 → 派发 `WebauthnLogin` 事件（带 `true` 第二参数）
3. `$guard->attempt()` 内部调用 `Auth::login()` → **重新认证用户**（再生 session ID、派发 `Login` 事件、更新 `viaRemember` 标记）

副作用 3 意味着密钥确认过程中：
- `Login` 事件会被派发 → `LoginListener` 会被触发 → 如果用户有 Webauthn 密钥且 `remember` 为 true，会再次设置 `return` cookie
- Session ID 会被再生 → 之前的 session 数据保持，但 ID 变化

这些副作用在实践中通常无害（用户本已登录），但值得关注：密钥确认并非"只做校验"，它实际上**执行了一次完整的重新登录**。

---

## 十三、密钥最后使用时间（used_at）更新链路

**链路状态：✅ 真实生效**

### 13.1 数据库字段与模型

- 迁移文件 `database/migrations/2025_05_05_101750_webauthn_used_at.php` 为 `webauthn_keys` 表添加了 `used_at` nullable timestamp 字段
- `app/Models/WebauthnKey.php#L16-L18` 将 `used_at` 标记为 fillable、visible、datetime cast
- 展示位置：`app/Actions/Jetstream/UserProfile.php#L30` 通过 `optional($key->used_at)->diffForHumans()` 显示为"last_used"

### 13.2 WebauthnAuthenticateListener

`app/Listeners/WebauthnAuthenticateListener.php` 是更新 `used_at` 的核心：

```php
public function handle(AuthenticatorAssertionResponseValidationSucceededEvent $event)
{
    $webauthnKey = WebauthnKey::where('user_id', $event->userHandle)
        ->where('credentialId', Base64UrlSafe::encode($event->publicKeyCredentialSource->publicKeyCredentialId))
        ->first();

    if ($webauthnKey !== null) {
        $webauthnKey->used_at = now();
        $webauthnKey->save();
    }
}
```

### 13.3 事件来源与监听器注册机制

**关键事实**：该监听器监听的**不是** Laravel Webauthn 包的 `WebauthnLogin` 事件，而是 **webauthn-lib 底层库**的 `Webauthn\Event\AuthenticatorAssertionResponseValidationSucceededEvent`。

这是 web-authn/webauthn-lib（FIDO2/WebAuthn 规范的 PHP 实现）在断言验证成功时派发的底层事件。

**注册机制**：项目中没有 `App\Providers\EventServiceProvider.php`，也没有在 `AppServiceProvider` 中显式调用 `Event::listen()` 注册该监听器。事件自动发现通过 Laravel 11 的 `Application::configure()` 默认机制工作。

**可验证事实（框架源码 `Illuminate\Foundation\Application::configure()`）**：

```php
public static function configure(?string $basePath = null)
{
    return (new Configuration\ApplicationBuilder(new static($basePath)))
        ->withKernels()
        ->withEvents()     // ← 默认调用
        ->withCommands()
        ->withProviders();
}
```

**可验证事实（框架源码 `ApplicationBuilder::withEvents()`）**：

```php
public function withEvents(array|bool $discover = [])
{
    if (is_array($discover) && count($discover) > 0) {
        AppEventServiceProvider::setEventDiscoveryPaths($discover);
    }
    if ($discover === false) {
        AppEventServiceProvider::disableEventDiscovery();
    }
    if (! isset($this->pendingProviders[AppEventServiceProvider::class])) {
        $this->app->booting(function () {
            $this->app->register(AppEventServiceProvider::class);
        });
    }
    $this->pendingProviders[AppEventServiceProvider::class] = true;
    return $this;
}
```

**完整的事件发现链路**：

1. `bootstrap/app.php` 中 `Application::configure()` 默认调用 `->withEvents()`
2. `withEvents()` 在 `app->booting()` 回调中注册 `Illuminate\Foundation\Support\Providers\EventServiceProvider`（框架内置的 `AppEventServiceProvider`）
3. 该服务提供者的 `register()` 方法调用 `getEvents()`
4. `getEvents()` 检查事件缓存（`bootstrap/cache/events.php`）：
   - **有缓存** → 加载缓存中的事件映射
   - **无缓存** → 调用 `discoveredEvents()` → `shouldDiscoverEvents()` → `discoverEvents()`
5. `shouldDiscoverEvents()` 对基类 `EventServiceProvider` 本身返回 `true`（因为 `get_class($this) === __CLASS__`）
6. `discoverEvents()` 使用 `DiscoverEvents::within()` 扫描 `app/Listeners` 目录，通过 `handle()` 方法的参数类型提示推断事件-监听器映射

**结论**：虽然项目没有自定义 `EventServiceProvider`，但 `Application::configure()` → `withEvents()` 会自动注册框架内置的 `EventServiceProvider`。该提供者扫描 `app/Listeners/` 目录，根据 `handle()` 方法的参数类型提示自动发现 `WebauthnAuthenticateListener`（监听 `AuthenticatorAssertionResponseValidationSucceededEvent`）和 `LoginListener`（监听 `Login`）等监听器并注册。

**生产环境注意**：`shouldDiscoverEvents()` 在非 local 环境下仍返回 `true`（对基类本身始终如此），但如果事件已被缓存（`php artisan event:cache`），则直接从缓存加载，跳过扫描。

**可验证事实（包源码 `WebauthnServiceProvider::bindWebAuthnPackage()`）**：

```php
$this->app->resolving(CanDispatchEvents::class,
    fn (CanDispatchEvents $object) => $object->setEventDispatcher(
        $this->app[EventDispatcherInterface::class]
    ));
```

包在 `WebauthnServiceProvider` 中将 PSR-14 `EventDispatcherInterface` 绑定到 `LaravelWebauthn\Events\EventDispatcher`（适配 Laravel 事件系统），并在解析所有 `CanDispatchEvents` 对象时注入这个 dispatcher。这样 webauthn-lib 底层派发的 PSR-14 事件会被桥接到 Laravel 的事件系统，从而触发 `WebauthnAuthenticateListener`。

### 13.4 触发场景与生效边界

`WebauthnAuthenticateListener` 的触发取决于 webauthn-lib 的 `AuthenticatorAssertionResponseValidator` 是否成功完成了一次断言验证。事件派发点在 `Webauthn::validateAssertion()` 内部（由 `EloquentWebAuthnProvider::validateCredentials()` 调用），通过 PSR-14 → `LaravelWebauthn\Events\EventDispatcher` 桥接到 Laravel 事件系统。

**✅ 会触发 `used_at` 更新的场景**：

1. **Passkey 首因登录**（`POST /webauthn/auth`）
   - `AuthenticateController::store()` → `loginPipeline()` → `AttemptToAuthenticate::attemptLogin()` → `$guard->attempt()` → `validateCredentials()` → `Webauthn::validateAssertion()` ✓

2. **Webauthn 二因验证**（`POST /webauthn/auth`，从 2FA 挑战页提交）
   - 同首因登录，走完全相同的路径 ✓

3. **密钥确认（敏感操作）**（`POST /webauthn/confirm-key`）
   - `ConfirmableKeyController::store()` → `ConfirmKey::__invoke()` → `$guard->attempt()` → `validateCredentials()` → `Webauthn::validateAssertion()` ✓

**❌ 不会触发 `used_at` 更新的场景**：

1. **密钥注册/更新**（Attestation 流程）：使用 `AuthenticatorAttestationResponseValidator`，派发的是 `AuthenticatorAttestationResponseValidationSucceededEvent`（不同事件类）
2. **密码登录**：不经过 `Webauthn::validateAssertion()`
3. **OAuth 登录**：不经过 `Webauthn::validateAssertion()`
4. **TOTP 二因验证**：走 Fortify 内置的 `TwoFactorAuthenticateAction`，不涉及断言校验

---

## 十四、WebauthnMiddleware 实际使用情况分析

**链路状态：❌ 残留/未接入——仅注册别名，未被任何路由使用**

### 14.1 注册位置

`bootstrap/app.php#L9` 和 `#L29`：

```php
use LaravelWebauthn\Http\Middleware\WebauthnMiddleware;
// ...
$middleware->alias([
    // ...
    'webauthn' => WebauthnMiddleware::class,
]);
```

### 14.2 实际使用情况验证

对项目所有 PHP 文件执行 grep `middleware.*webauthn` 和 `webauthn.*middleware`，结果为 **0 匹配**。具体检查：

| 检查范围 | 结果 |
|---------|------|
| `routes/web.php`（含所有路由分组） | 无任何 `->middleware('webauthn')` 或 `'middleware' => 'webauthn'` |
| `routes/api.php` | 无 |
| 所有控制器的 `$middleware` 属性 | 无 |
| 所有控制器的构造函数 `$this->middleware()` | 无 |

### 14.3 包源码中的 WebauthnMiddleware 作用（推测）

包的 `WebauthnMiddleware`（`LaravelWebauthn\Http\Middleware\WebauthnMiddleware`）通常用于以下场景：
- 拦截需要 Webauthn 二次验证的请求
- 检查 session 中 `webauthn_auth` 标记是否存在
- 如果未通过 Webauthn 验证，重定向到挑战页

但项目中所有需要 Webauthn 验证的场景都通过**前端主动触发**（`WebauthnLogin.vue` 或 `WebauthnTest.vue`），不依赖后端中间件拦截。因此该中间件别名只是注册了但从未使用，属于**残留代码**。

---

## 十五、自动登录 cookie 命名分析：return vs webauthn_remember

**链路状态：✅ 生产代码真实生效，❌ 测试代码过时不匹配**

### 15.1 生产代码的 cookie 命名：`return`

**写入位置**（`app/Listeners/LoginListener.php#L15-L19`）：

```php
public function handle(Login $event)
{
    if ($event->remember && $event->user->webauthnKeys()->count() > 0) {
        Cookie::queue('return', 'true', 60 * 24 * 365);
    }
}
```

- cookie 名称：**`return`**
- cookie 值：**`'true'`**（字符串布尔值）
- 有效期：1 年（60 × 24 × 365 分钟）
- 触发条件：用户登录时勾选了"记住我"（`$event->remember === true`）**且**用户拥有至少一个 Webauthn 密钥

**读取位置**（`app/Http/Controllers/Auth/LoginController.php#L36`）：

```php
$data['autologin'] = $request->cookie('return') === 'true';
```

读取后传递给前端 `Login.vue`，当 `autologin === true` 且设备支持平台认证器时，自动触发 Passkey 登录流程。

### 15.2 测试代码的 cookie 命名：`webauthn_remember`（过时/错误）

`tests/Feature/Controllers/Auth/LoginControllerTest.php#L47`：

```php
$response = $this->withCookie('webauthn_remember', $user->id)
    ->get('/login', [/* Inertia headers */]);
```

- cookie 名称：**`webauthn_remember`**
- cookie 值：**`$user->id`**（用户 ID，整数）

### 15.3 不一致分析

| 维度 | 生产代码（LoginListener + LoginController） | 测试代码（LoginControllerTest） |
|------|------------------------------------------|-------------------------------|
| cookie 名称 | `return` | `webauthn_remember` |
| cookie 值 | `'true'`（字符串） | `$user->id`（整数） |
| 匹配性 | ✅ 写入和读取一致 | ❌ 与生产代码不匹配 |

**结论**：测试使用的是早期版本的 cookie 命名方案（`webauthn_remember` + 用户 ID），后来生产代码改为了更简洁的 `return` + `'true'`，但测试代码未同步更新。该测试目前应该**无法通过**（因为 `LoginController` 读取 `cookie('return')` 而非 `cookie('webauthn_remember')`），除非测试框架有特殊的 cookie 处理逻辑。

---

## 十六、链路生效状态总览：真实生效 vs 残留/未接入

### ✅ 真实生效的链路

| 链路 | 入口代码 | 核心实现 |
|------|---------|---------|
| **Passkey 首因登录** | 登录页 `WebauthnLogin.vue` → `webauthn.auth` | 包内置管线 `[AttemptToAuthenticate, PrepareAuthenticatedSession]` → `EloquentWebAuthnProvider` |
| **Webauthn 二因验证** | 2FA 挑战页 `WebauthnLogin.vue` → `webauthn.auth` | 同上，共享同一端点和管线 |
| **邮箱密码登录 + 2FA 拦截** | `POST /login` → Fortify 管线 | 自定义 `RedirectIfTwoFactorAuthenticatable` 替换 Fortify 默认 |
| **OAuth 登录 + 2FA 拦截** | `auth/{driver}/callback` → 独立 Pipeline | 自定义 `AttemptToAuthenticateSocialite` |
| **敏感操作密钥确认** | `ConfirmsPassword.vue` → `WebauthnTest.vue` → `webauthn.key.confirm` | 包的 `ConfirmableKeyController` → `ConfirmKey` action → `guard->attempt()`（⚠️ 副作用：重新登录 + 派发 Login 事件） |
| **敏感操作密码确认** | `ConfirmsPassword.vue` → `POST password.confirm` | Jetstream 内置 |
| **密钥 used_at 时间更新** | 任何断言校验成功时自动触发 | `WebauthnAuthenticateListener` 监听 webauthn-lib 的 `AuthenticatorAssertionResponseValidationSucceededEvent` |
| **自动登录 cookie（return）** | 登录成功后 `LoginListener` 写入 | `Cookie::queue('return', 'true')` + `LoginController` 读取 |
| **Webauthn 密钥注册/更新/删除** | 设置页 `WebauthnKeys.vue` → `webauthn.store/update/destroy` | 包控制器 + 自定义响应类 `WebauthnUpdateResponse` / `WebauthnDestroyResponse` |

### ❌ 残留/未接入/死代码

| 代码 | 位置 | 状态说明 |
|------|------|---------|
| **`AttemptToAuthenticateWebauthn`** | `app/Actions/AttemptToAuthenticateWebauthn.php` | 是包内置 `AttemptToAuthenticate` 的简化副本，但未通过任何配置或回调接入 Webauthn 管线，仅自身和单元测试引用 |
| **`WebauthnMiddleware` 别名** | `bootstrap/app.php#L29` 注册别名 `'webauthn'` | 全项目无任何路由或控制器使用该中间件别名，仅注册未使用 |
| **测试中的 `webauthn_remember` cookie** | `tests/Feature/Controllers/Auth/LoginControllerTest.php#L47` | 测试使用旧命名 `webauthn_remember` + 用户 ID，生产代码已改为 `return` + `'true'`，测试与生产不一致 |

---

## 十七、事实层级标注（续：新增核对点）

| 描述 | 事实层级 |
|------|---------|
| `ConfirmsPassword.vue` 在 8 处使用（WebauthnKeys 2 处 + TwoFactorAuthenticationForm 6 处） | 项目可验证（grep） |
| `WebauthnTest.vue` POST 到 `webauthn.auth.options` 再 POST 到 `webauthn.key.confirm` | 项目可验证（组件源码） |
| `ConfirmableKeyController::store()` 成功后写 `auth.password_confirmed_at` session | 包源码可验证 |
| `ConfirmKey` action 调用 `$guard->attempt()` 与登录共享校验路径 | 包源码可验证 |
| `KeyConfirmedResponse::toResponse()` 对 JSON 请求返回 `JsonResponse('', 201)`（空 body + 201 状态码） | 包源码可验证 |
| `FailedKeyConfirmedResponse::toResponse()` 对 JSON 请求抛 `ValidationException`（422，字段 key，消息 "Invalid key."） | 包源码可验证 |
| `WebauthnTest.vue` 的 `.then()` 只检测 2xx 状态码即 emit('success')，不读取响应体 | 项目可验证（组件源码） |
| `ConfirmKey` 调用 `$guard->attempt()` 会导致 `Auth::login()` 重新认证已登录用户（再生 session、派发 Login 事件） | 包源码 + 框架源码可验证 |
| `ConfirmsPassword.vue` 密码确认与密钥确认共享同一个 `confirm()` → `emit('confirmed')` 出口 | 项目可验证（组件源码） |
| `WebauthnAuthenticateListener` 监听 webauthn-lib 的 `AuthenticatorAssertionResponseValidationSucceededEvent`（非包的 `WebauthnLogin`） | 项目可验证（监听器源码 use 语句 + handle 参数类型） |
| 事件自动发现通过 `Application::configure()` → `withEvents()` 注册框架内置 `EventServiceProvider` 实现 | 框架源码可验证（`ApplicationBuilder::withEvents()` + `EventServiceProvider::register()` + `discoverEvents()`） |
| `shouldDiscoverEvents()` 对基类 `EventServiceProvider` 本身返回 `true`，因此自动发现默认启用 | 框架源码可验证（`get_class($this) === __CLASS__` 条件） |
| 包通过 `CanDispatchEvents` 解析回调将 PSR-14 事件桥接到 Laravel 事件系统 | 包源码可验证（`WebauthnServiceProvider::bindWebAuthnPackage()`） |
| `bootstrap/app.php` 注册了 `webauthn` 中间件别名但无任何路由使用 | 项目可验证（grep 0 匹配） |
| `LoginListener` 写入 `cookie('return', 'true')`，`LoginController` 读取 `cookie('return')` | 项目可验证 |
| 测试使用 `cookie('webauthn_remember', $user->id)` 与生产代码不一致 | 项目可验证（测试源码 vs 生产源码对比） |

---

## 十八、代码走向的"为什么让人困惑"（续）

9. **Webauthn 两种确认流程共享同一校验路径但入口完全不同**：登录（首因/二因）使用 `webauthn.auth` 路由走完整的登录管线（含 session 再生、Webauthn::login() 等），而敏感操作确认使用 `webauthn.key.confirm` 路由只做断言校验 + 写 `password_confirmed_at`。两者底层都通过 `guard->attempt()` → `EloquentWebAuthnProvider` 校验断言，但外层包装完全不同。

10. **WebauthnAuthenticateListener 监听的是底层库事件而非包事件**：直觉上会以为它监听 Laravel Webauthn 包的 `WebauthnLogin` 事件，但实际上它监听的是 web-authn/webauthn-lib 的 `AuthenticatorAssertionResponseValidationSucceededEvent`——这是一个 PSR-14 事件，通过包的 `EventDispatcher` 桥接才能被 Laravel 事件系统捕获。不了解这层桥接会疑惑"这个监听器到底能不能被触发"。

11. **事件发现机制是隐式的**：项目中没有 `App\Providers\EventServiceProvider`，也没有任何 `Event::listen()` 调用注册 `LoginListener` 和 `WebauthnAuthenticateListener`。它们的注册完全依赖 `Application::configure()` → `withEvents()` 自动注册的框架内置 `EventServiceProvider`。这个机制在项目代码中完全看不到，需要追踪框架源码才能理解。

12. **密钥确认的 `$guard->attempt()` 有副作用**：直觉上密钥确认应该"只做校验"，但 `ConfirmKey` action 调用的 `$guard->attempt()` 实际上执行了一次完整的重新登录（包括 `Auth::login()`、再生 session、派发 `Login` 事件），在已登录用户身上产生意料之外的副作用。

13. **测试与生产的 cookie 命名不一致**：测试使用 `webauthn_remember`，生产使用 `return`，如果只看测试会误以为系统使用 `webauthn_remember` cookie，但实际运行的代码完全不同。

14. **WebauthnMiddleware 注册了但不用**：作为中间件别名注册在 `bootstrap/app.php` 中，看起来像是项目使用了这个中间件，但实际上全项目没有任何路由挂载它。它的存在容易让阅读者以为存在"通过中间件自动拦截 Webauthn 验证"的链路。
