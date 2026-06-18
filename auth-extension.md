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
