# Monica 登录认证扩展层代码走向

Monica 的认证体系基于 Laravel 生态的三层扩展框架搭建：**Fortify**（无 UI 的后端认证管线）、**Jetstream**（Inertia 前端脚手架）和 **Laravel Webauthn**（asbiin/laravel-webauthn，Passkey / 安全密钥认证）。在此基础上，项目通过 Socialite 接入了外部 OAuth/SAML 提供商。

下面按代码执行顺序，重点讲清 **Webauthn 安全密钥登录**的完整链路（从登录页、双因素挑战页到服务端断言校验），并对比三条登录管线的关系。

---

## 一、架构总览与关键配置

### 1. 认证 Guard 与 User Provider

`config/auth.php#L38-L43` 定义了唯一的 guard `web`，使用 `session` 驱动。

**关键设计点**：User Provider 的驱动被设为 `webauthn`（而非默认的 `eloquent`）——这意味着用户检索逻辑经过了 Laravel Webauthn 包的包装。实质上仍以 `App\Models\User` 为模型，但在用户认证环节增加了 Webauthn 凭据的感知与校验能力。

`config/auth.php#L62-L66`：
```php
'providers' => [
    'users' => [
        'driver' => 'webauthn',
        'model' => env('AUTH_MODEL', App\Models\User::class),
    ],
],
```

这个 `webauthn` 驱动来自 asbiin/laravel-webauthn 包，它会在 `Guard::attempt()` 时判断提交的凭据是密码还是 Webauthn 断言数据，并据此走不同的认证分支。

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
- `userless` 由 `WEBAUTHN_USERLESS` 环境变量控制，默认为 `true`——即支持"无用户名登录"（Passkey 一键登录，无需输入邮箱）
- `redirects.login` 指向 `/vaults`
- `views.authenticate` 和 `views.register` 均为 `null`，表示不使用包内置的 Blade 视图，全部走 Inertia 前端
- `session_name` 为 `webauthn_auth`，用于存储已通过 Webauthn 验证的状态
- `model` 使用自定义的 `App\Models\WebauthnKey`（扩展了 `used_at` 字段）

### 4. Webauthn 中间件注册

`bootstrap/app.php#L29` 将 `LaravelWebauthn\Http\Middleware\WebauthnMiddleware` 注册为别名 `webauthn`。这个中间件是包内部使用的，用于在需要 Webauthn 第二因素验证时拦截请求并重定向到验证流程。

### 5. Webauthn 响应类绑定

`app/Providers/AppServiceProvider.php#L154-L155` 在 boot() 中将 Webauthn 密钥创建和删除的响应类绑定到了项目自定义实现：
```php
Webauthn::updateViewResponseUsing(WebauthnUpdateResponse::class);
Webauthn::destroyViewResponseUsing(WebauthnDestroyResponse::class);
```

这两个类 (`app/Http/Controllers/Profile/WebauthnUpdateResponse.php` 和 `WebauthnDestroyResponse.php`) 实现了包提供的契约，控制 Inertia 请求下的重定向行为。

### 6. 外部登录提供商

`config/auth.php#L127` 中 `login_providers` 通过 `LOGIN_PROVIDERS` 环境变量动态注入。`config/services.php` 中配置了 azure、facebook、github、google、linkedin、saml2、kanidm、keycloak 共 8 种。各提供商的 Socialite 扩展在 `app/Providers/AppServiceProvider.php#L159-L165` 中通过监听 `SocialiteWasCalled` 事件注册。

---

## 二、Webauthn 登录路径一：登录页 Passkey 首因登录

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

**代码证据**：`app/Http/Controllers/Auth/LoginController.php#L33-L37`
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

路由 `webauthn.auth` 由 Laravel Webauthn 包自动注册，挂载在 `/webauthn/auth`（路径前缀来自 `config/webauthn.php#L55`）。

### 第 5 步：服务端 Webauthn 管线与自定义 AttemptToAuthenticateWebauthn

Laravel Webauthn 包在内部有类似 Fortify 的 Pipeline 机制，将登录请求按管道顺序处理。**项目自定义的 `app/Actions/AttemptToAuthenticateWebauthn.php` 就是这个管线中的核心认证管道**（替代或扩展了包内置的对应动作）。

`AttemptToAuthenticateWebauthn::handle()`（`app/Actions/AttemptToAuthenticateWebauthn.php#L32-L42`）：
```php
public function handle(Request $request, $next)
{
    if ($this->attemptValidateAssertion($request)
        || $this->attemptLogin($this->filterCredentials($request), $request->boolean('remember'))) {
        return $next($request);
    }

    $this->throwFailedAuthenticationException($request);
    return null;
}
```

这里用了一个短路 OR `||`，按顺序尝试两种认证模式：

**模式 A：`attemptValidateAssertion()`（2FA 场景，用户已登录）**
- 检查 `$request->user()` 是否非空
- 如果已登录，调用 `WebauthnFacade::validateAssertion($user, $credentials)` 验证断言——此时用户身份已知，只校验密钥签名
- 验证失败则触发 Failed 事件并抛异常

**模式 B：`attemptLogin()`（Passkey 首因登录，用户未登录）**
- 调用 `$this->guard->attempt($credentials, $remember)`
- 因为 auth.providers.users.driver 是 `webauthn`，Guard 的 attempt() 会调用 Webauthn UserProvider
- UserProvider 先根据断言中的 `userHandle` 或邮箱查找用户，再验证签名有效性
- 验证通过后完成标准 Laravel session 登录

在**登录页 Passkey 登录场景**中，用户尚未登录，所以 `attemptValidateAssertion()` 返回 `false`，自动进入 `attemptLogin()` 分支。

`filterCredentials()`（`app/Actions/AttemptToAuthenticateWebauthn.php#L109-L112`）只保留 Webauthn 断言字段：
```php
protected function filterCredentials(Request $request): array
{
    return $request->only(['id', 'rawId', 'response', 'type']);
}
```

### 第 6 步：管线后续处理与登录成功

`AttemptToAuthenticateWebauthn` 认证通过后，`return $next($request)` 将请求传递给 Webauthn 管线的后续管道。包内后续的默认管道通常包含：
- 记录 WebauthnKey 的最后使用时间（`used_at` 字段）
- 写入 `webauthn_auth` session（`config/webauthn.php#L158` 的 session_name）
- 触发 `Login` 事件
- 重定向到 `redirects.login`（`/vaults`）

`Login` 事件触发后，`app/Listeners/LoginListener.php#L15-L19` 会检查：如果用户勾选了 "记住我" 且拥有 Webauthn 密钥，则设置 `return` cookie（有效期一年），下次自动触发 Passkey 登录。

此外，`app/Providers/AppServiceProvider.php#L157` 订阅了包内的 `LoginViaRemember` 监听器，用于处理 Laravel 记住我 cookie 触发的自动登录中的 Webauthn 相关逻辑。

---

## 三、Webauthn 登录路径二：双因素挑战页的第二因素验证

这是"用户先输入邮箱密码，然后再用安全密钥做第二因素"的场景。

### 第 1 步：从邮箱密码登录被 2FA 拦截

用户提交邮箱密码到 `/login`，请求进入 Fortify Pipeline：
1. **管道 1**：`app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php#L28-L38`
   - `validateCredentials()` 通过 email 查用户并验证密码（此时不登录，只验证凭据）
   - 判断是否需要 2FA（`config/fortify.php#L32-L34`）：
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

**注意与登录页的区别**：此处调用 `Webauthn::prepareAssertion($user)` 传入了**具体用户对象**，生成的是针对该用户的断言挑战（非 userless 模式）。浏览器只会查找属于该用户的安全密钥，不会列出所有 Passkey。

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

POST 到 `webauthn.auth` 后，再次进入 Laravel Webauthn 的管线和自定义 `AttemptToAuthenticateWebauthn`。

但这次走**模式 A**：
- `attemptValidateAssertion()` 中 `$request->user()` 为 `null`（用户还未登录，session 中只有 `login.id`）
- 等等，这里有个关键点需要理清：此时用户到底有没有登录？

实际流程是：**Fortify 的 RedirectIfTwoFactorAuthenticatable 只验证了密码，没有调用 `Auth::login()`**。用户在 session 中的状态只有 `login.id`，并未被 Laravel guard 识别为已认证。

那 Webauthn 包是如何知道验证哪个用户的？答案是：**Webauthn 包内部从 session 中读取 `login.id`，用它来确定目标用户。**这是 asbiin/laravel-webauthn 包的内部机制——包的控制器在处理 `webauthn.auth` 路由时，会优先检查 session 中的 `login.id`（Fortify 设置的），如果存在就以该用户为上下文验证断言，并在验证通过后完成登录流程（写入 session、触发 Login 事件等）。

因此在 2FA 场景下，`AttemptToAuthenticateWebauthn` 的实际行为是：
- `attemptValidateAssertion()` 中 `$request->user()` 返回 null，返回 false
- `attemptLogin()` 被调用，Webauthn UserProvider 不是根据密码而是根据断言中的凭据 ID 查找用户
- 或者（更准确地说是包的内部工作方式）：Webauthn 包的控制器在进入管线之前就已经从 session 的 `login.id` 确定了目标用户，直接调用包内部的 `Webauthn::validateAssertion($userFromSession, $credentials)`，验证通过后再调用 `Auth::login()`

无论哪种方式，最终结果一致：断言验证通过即完成登录，重定向到 `/vaults`。

---

## 四、Webauthn 密钥注册（安全密钥创建）流程

用户登录后，在个人设置页面可以添加新的安全密钥。这是 Webauthn 的"注册"（Attestation）流程，与登录的"断言"（Assertion）是两个不同阶段。

### 第 1 步：用户设置页加载密钥管理

`app/Actions/Jetstream/UserProfile.php` 是 Jetstream 用户资料页的数据钩子，它向 Inertia 页面注入 `webauthnKeys` 列表：

`app/Actions/Jetstream/UserProfile.php#L25-L32`：
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

路由 `webauthn.store.options` 由 Laravel Webauthn 包注册，返回用于创建新密钥的公钥挑战参数（包含用户信息、Relying Party、允许的算法等）。

### 第 4 步：浏览器生成新密钥对并提交注册响应

`registerWaitForKey()`（`resources/js/Pages/Webauthn/WebauthnKeys.vue#L68-L74`）调用 `@simplewebauthn/browser` 的 `startRegistration(publicKey)`，浏览器弹出安全密钥交互提示，生成新密钥对后，由 `webauthnRegisterCallback()` 将注册响应（包含公钥、密钥句柄、认证器信息等）连同密钥名称一起 POST 到 `route('webauthn.store')`。

### 第 5 步：服务端保存密钥

`webauthn.store` 路由由包处理，内部流程：
1. 校验注册响应的签名（证明认证器是真实的）
2. 将公钥和密钥元数据存入 `webauthn_keys` 表（使用自定义的 `App\Models\WebauthnKey` 模型，扩展了 `used_at` 字段）
3. 返回响应时使用自定义的 `WebauthnUpdateResponse`（在 `AppServiceProvider` 中绑定），Inertia 请求下重定向到 `/user/profile`

密钥更新（`webauthn.update`，修改名称）和删除（`webauthn.destroy`）也使用了类似的响应类绑定。

---

## 五、三条登录管线的关系与差异对比

| 维度 | 邮箱密码登录 | Webauthn Passkey 登录 | 外部 OAuth/SAML 登录 |
|------|-------------|---------------------|---------------------|
| **前端入口** | `Login.vue` 表单提交 | `Login.vue` / `TwoFactorChallenge.vue` → `WebauthnLogin.vue` | `ExternalProviders.vue` 按钮点击 |
| **POST 路由** | `POST /login` (Fortify) | `POST /webauthn/auth` (Laravel Webauthn 包) | `GET /auth/{driver}` → 第三方 → `GET/POST /auth/{driver}/callback` |
| **后端管线** | Fortify Pipeline（配置于 `config/fortify.php#L145-L151`） | Webauthn 包内置 Pipeline（含自定义 `AttemptToAuthenticateWebauthn`） | 独立 `Illuminate\Pipeline\Pipeline` 实例（在 `SocialiteCallbackController#L62-L66` 中手动构建） |
| **自定义认证动作** | `RedirectIfTwoFactorAuthenticatable`（替换 Fortify 默认） | `AttemptToAuthenticateWebauthn`（替换/扩展包默认） | `AttemptToAuthenticateSocialite`（项目自建） |
| **用户密码校验** | `guard->validateCredentials()` 在 2FA 拦截中完成 | 不涉及（用密钥断言替代） | 不涉及（由第三方保证） |
| **2FA 检查位置** | Fortify Pipeline 的第一个管道（登录前拦截） | **不检查**——Webauthn 本身既是首因也可作为二因 | 自定义管线 `AttemptToAuthenticateSocialite` 中，在 `guard->login()` 之前 |
| **2FA 判断条件** | TOTP 已启用 OR Webauthn 有密钥（两处代码完全一致） | 无——Webauthn 包自行处理 session 中的 login.id | 同左 |
| **用户 Provider** | `webauthn` driver，但在 2FA 拦截阶段已手动查用户 | `webauthn` driver，处理断言认证 | `webauthn` driver，但用户由 Socialite driver_id 查 UserToken 关联得到 |
| **登录成功后** | `AttemptToAuthenticate` → `PrepareAuthenticatedSession` | 包内管道：更新 used_at、写 session、触发 Login 事件 | `PrepareAuthenticatedSession`（复用 Fortify 内置） |
| **触发 Login 事件** | Fortify 内置动作触发 | Webauthn 包触发 + 自定义 `LoginListener` 设置 `return` cookie | `guard->login()` 触发 |
| **速率限制** | Fortify 的 `login` limiter | Webauthn 的 `login` limiter（`config/webauthn.php#L94-L96`） | `oauth2-socialite` limiter（`AppServiceProvider#L152`，每分钟 5 次） |

### 三者的连接点

1. **共享同一个 `web` guard 和 `webauthn` user provider**：无论走哪条路径，最终都通过同一个 guard 建立 session，所以登录态完全互通。

2. **共享同一套 2FA 判断逻辑**：
   - `RedirectIfTwoFactorAuthenticatable.php#L32-L34`
   - `AttemptToAuthenticateSocialite.php#L51-L54`
   两处用了完全相同的条件表达式：
   ```php
   (optional($user)->two_factor_secret && ! is_null(optional($user)->two_factor_confirmed_at))
   || Webauthn::enabled($user)
   ```

3. **2FA 挑战页是统一出口**：无论从邮箱密码还是 OAuth 被拦截，最终都跳转到 `two-factor.login`，该页面同时支持 TOTP 和 Webauthn。

4. **Webauthn 可以作为首因（Passkey）或第二因素**：
   - 首因：登录页 Passkey，用户跳过邮箱密码输入，直接使用安全密钥
   - 二因：邮箱密码或 OAuth 登录后，再使用安全密钥通过 2FA 挑战

5. **AttemptToAuthenticateWebauthn 设计的巧妙之处**：handle() 方法中先试 `attemptValidateAssertion()`（用户已登录时的断言验证）再试 `attemptLogin()`（用户未登录时走 guard attempt），使得同一个类可以同时服务于"2FA 二因验证"和"Passkey 首因登录"两种使用模式。虽然 2FA 场景下 `$request->user()` 可能为 null（因为 Fortify 的 2FA 拦截只验证凭据不登录），但包的路由控制器会从 session `login.id` 中恢复用户上下文，最终效果等价。

---

## 六、注册场景

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

## 七、注册场景（Webauthn 密钥注册）

见第四节的"Webauthn 密钥注册流程"，该流程在用户已登录的设置页面进行。

---

## 八、扩展层关键文件索引

| 文件（仓库相对路径） | 角色 |
|------|------|
| `config/fortify.php` | Fortify 功能与登录管线配置，自定义了 `pipelines.login` |
| `config/auth.php` | Guard、User Provider（`webauthn` driver）、外部登录提供商配置 |
| `config/webauthn.php` | Webauthn 全局配置（路由前缀、userless、redirects、views 等） |
| `config/jetstream.php` | Jetstream 功能开关 |
| `config/services.php` | 各 OAuth/SAML 提供商凭证配置 |
| `bootstrap/app.php` | WebauthnMiddleware 别名注册 |
| `app/Providers/AppServiceProvider.php` | Socialite 提供商事件监听注册、Webauthn 响应类绑定、LoginViaRemember 订阅 |
| `app/Providers/AuthServiceProvider.php` | Gate 定义（administrator、vault-viewer 等权限） |
| `app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php` | **Fortify 登录管线自定义 2FA 拦截点**，替换包内置版本 |
| `app/Actions/Fortify/TwoFactorChallengeView.php` | 2FA 挑战页视图响应（含 Webauthn 公钥准备），替代 Fortify 默认 |
| `app/Actions/Fortify/CreateNewUser.php` | 注册 Action，实现 Fortify 的 CreatesNewUsers 契约 |
| `app/Actions/AttemptToAuthenticateWebauthn.php` | **Webauthn 登录管线自定义认证动作**，同时支持首因登录和 2FA 断言验证 |
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

---

## 九、代码走向的"为什么让人困惑"

以下是导致扩展层代码走向难以直观理解的几个设计因素：

1. **三条管线各自独立**：邮箱密码走 Fortify Pipeline（`config/fortify.php`），OAuth 走独立实例化的 `Illuminate\Pipeline\Pipeline`（`SocialiteCallbackController` 内手动 new），Webauthn 走 asbiin/laravel-webauthn 包的内部管线。三条管线的配置位置、管道顺序、2FA 检查实现方式都不同，需要分别追踪。

2. **2FA 检查逻辑重复实现**：完全相同的条件表达式（TOTP OR Webauthn）在 `RedirectIfTwoFactorAuthenticatable` 和 `AttemptToAuthenticateSocialite` 中各写了一份——如果有朝一日增加第三种 2FA 方式，需要同步修改两处。

3. **自定义与内置管道混用**：Fortify 管线中只有第一个管道被项目自定义，后两个仍使用 Fortify 内置版本；Webauthn 管线中 `AttemptToAuthenticateWebauthn` 是自定义的，但后续管道（更新 used_at、写 session、重定向）仍是包内置。这种"只换一环"的模式需要对框架内部机制有了解才能看全。

4. **AttemptToAuthenticateWebauthn 的 OR 短路逻辑**：`handle()` 中用 `attemptValidateAssertion() || attemptLogin()` 这个看似简单的 OR 同时支持"已登录用户的 2FA 断言验证"和"未登录用户的 Passkey 首因登录"两种场景，但 `attemptValidateAssertion()` 中的 `$request->user()` 与 2FA 实际流程（session 中只有 login.id、guard 未登录）之间存在语义差异，实际依赖包的路由控制器在管线执行前先从 session 恢复用户上下文——这个隐含依赖单看项目代码难以察觉。

5. **隐式契约绑定**：`TwoFactorChallengeView`（通过 Fortify features 配置自动绑定）、`WebauthnUpdateResponse` / `WebauthnDestroyResponse`（在 AppServiceProvider 的 `updateViewResponseUsing` / `destroyViewResponseUsing` 中绑定）、`CreateNewUser`（Fortify 自动解析契约实现）等都属于"在配置文件或服务提供者中替换接口实现"的模式，IDE 的"查找引用"通常无法直接跳到调用方。

6. **User Provider 的 `webauthn` driver**：`config/auth.php` 中 `providers.users.driver` 从默认的 `eloquent` 改成了 `webauthn`，这改变了 `Guard::attempt()` 的内部行为——从纯密码校验变成"密码或断言二选一"。但因为改动只在配置文件中，代码搜索无法直接找到关联。

7. **Webauthn 路由全部由包注册**：项目代码中看不到 `/webauthn/auth`、`/webauthn/store` 等路由的定义，它们由 `asbiin/laravel-webauthn` 包的 ServiceProvider 自动注册，挂载在 `config/webauthn.php#L55` 的 `prefix` 下。对于不熟悉该包的开发者，找路由会有些"魔法"感。
