# Monica 认证流程分析：Fortify 与 Jetstream 扩展钩子

## 概述

Monica 项目基于 Laravel 框架，使用 **Laravel Fortify** 作为后端认证逻辑、**Laravel Jetstream**（Inertia 栈）作为前端脚手架，并深度集成了 **WebAuthn**（无密码认证）和 **Socialite**（社交登录）能力。本文档从守卫替换、双因素验证、登录速率限制、记住我/会话刷新、2FA 管理动作、社交登录绑定（含 SAML2）六个维度逐层剖析扩展实现。

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

### 5. Sanctum 用户设置中间件（SanctumSetUser）

[SanctumSetUser](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Middleware/SanctumSetUser.php) 中间件用于将当前已认证用户同步到 Sanctum guard，并附加一个 `TransientToken`（瞬时令牌），使得 Web 会话用户可以通过 Sanctum 授权检查。

```php
// SanctumSetUser.php
$this->sanctum()->setUser($request->user()->withAccessToken(new TransientToken));
```

#### 入栈的门控条件

SanctumSetUser **并非全局 web 中间件**，也不在默认 `web` 中间件组中。它唯一的使用位置是 [EnsureDavRequestsAreStateful](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Middleware/EnsureDavRequestsAreStateful.php#L38-L43) 内部的条件 Pipeline：

```php
// EnsureDavRequestsAreStateful.php
public function handle(Request $request, Closure $next)
{
    return (new Pipeline($this->app))->send($request)->through(
        $this->auth->guard()->check()   // 前置条件 1：用户已认证
            ? $this->authenticated()     //   → 走 authenticated 管道（含 SanctumSetUser）
            : $this->authenticate()      //   → 走 authenticate 管道（Basic Auth 等）
    )->then(fn (Request $request) => $next($request));
}

private function authenticated(): array
{
    return $this->app->environment('local')  // 前置条件 2：本地开发环境
        ? [SanctumSetUser::class]            //   → local 环境才注入 SanctumSetUser
        : [];                                 //   → 生产环境无任何中间件
}
```

三层门控：
1. **仅 DAV 请求路径**：只有经过 `EnsureDavRequestsAreStateful` 的 `/dav`（CardDAV/CalDAV）请求才可能触发
2. **用户已认证**：`$this->auth->guard()->check()` 必须返回 true（即 session 已登录或 Basic Auth 已通过）
3. **本地开发环境**：`$this->app->environment('local')` 必须成立

生产环境下 DAV 的已认证请求不会执行 SanctumSetUser，因为 DAV 请求依赖 Basic Auth 或 Bearer Token 而非 sanctum guard 做授权。

#### 常规 Web 请求的 Sanctum 用户同步机制

对于受 `auth:sanctum` 保护的 Inertia 页面路由（[web.php#L184-L188](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/routes/web.php#L184-L188)），Sanctum 的 `Guard::user()` 会自动回退到 `web` session guard 获取用户，因此**不需要** SanctumSetUser。只有在 DAV 这种绕过常规 auth 中间件、手动构建 Pipeline 的场景下，才需要手动同步。

### 6. User 模型的认证 Trait

[User 模型](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Models/User.php#L21-L29) 组合了三个认证相关 Trait：

- `Laravel\Fortify\TwoFactorAuthenticatable` — Fortify 双因素认证（含恢复码、TOTP secret 管理）
- `Laravel\Sanctum\HasApiTokens` — Sanctum API 令牌
- `LaravelWebauthn\WebauthnAuthenticatable` — WebAuthn 认证

---

## 二、双因素验证插入环节

Monica 支持**两种**双因素认证方式：TOTP（标准 2FA）和 WebAuthn（硬件密钥/生物识别），两者在登录管道中并行检查。

### 1. Fortify 登录管道自定义：与默认顺序的差异

Monica 在 [fortify.php#L145-L151](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/fortify.php#L145-L151) 中重定义了 `login` 管道。**这里必须澄清一个关键事实：**

| 管道顺序 | 默认 Fortify 管道 | Monica 自定义管道 |
|---------|------------------|------------------|
| 第 1 步 | `AttemptToAuthenticate`（认证并登录） | `RedirectIfTwoFactorAuthenticatable`（仅验证凭证，不登录） |
| 第 2 步 | `RedirectIfTwoFactorAuthenticatable`（检查是否需 2FA，已登录则登出） | `AttemptToAuthenticate`（实际登录） |
| 第 3 步 | `PrepareAuthenticatedSession` | `PrepareAuthenticatedSession` |

Monica 自定义管道：

```php
'pipelines' => [
    'login' => [
        \App\Actions\Fortify\RedirectIfTwoFactorAuthenticatable::class,  // 自定义：先验证凭证并决定是否跳 2FA
        \Laravel\Fortify\Actions\AttemptToAuthenticate::class,            // 无 2FA 则实际登录
        \Laravel\Fortify\Actions\PrepareAuthenticatedSession::class,      // 准备会话
    ],
],
```

**顺序差异的本质原因：** 默认 Fortify 的 `AttemptToAuthenticate` 会直接调用 `Auth::attempt()` 将用户登录；若用户启用了 2FA，默认的 `RedirectIfTwoFactorAuthenticatable` 需要**先登录再登出**，仅保留 session 中的 `login.id`。Monica 将 2FA 检查前置到实际登录之前，避免了"登录 → 立刻登出"的往返，也减少了不必要的会话事件触发。

### 2. RedirectIfTwoFactorAuthenticatable：validateCredentials 而非 Auth::attempt 的关键机制

[RedirectIfTwoFactorAuthenticatable](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php) 是 Monica 自定义类（替代 Fortify 原生同名类），核心区别在于它**不使用 `Auth::attempt()`，而是通过 guard 的 provider 直接调用 `validateCredentials()`**。

核心逻辑如下：

```php
// 第 28-38 行
public function handle(Request $request, Closure $next): mixed
{
    $user = $this->validateCredentials($request);  // ① 验证密码，但不登录

    if ((optional($user)->two_factor_secret && ! is_null(optional($user)->two_factor_confirmed_at))
        || Webauthn::enabled($user)) {             // ② 检查是否启用 2FA
        return $this->twoFactorChallengeResponse($request, $user);  // ③ 需要 2FA：跳转挑战页
    }

    return $next($request);  // ④ 不需要 2FA：传递给管道下一步 AttemptToAuthenticate
}

// 第 43-52 行
protected function validateCredentials(Request $request): ?User
{
    return tap(User::where('email', $request->email)->first(), function ($user) use ($request) {
        // 关键点：使用 guard 的 UserProvider 直接 validateCredentials，
        // 而非 Auth::attempt()——因此不会写入 session、不会触发 Login 事件
        if (! $user || ! $this->guard->getProvider()->validateCredentials($user, ['password' => $request->password])) {
            $this->fireFailedEvent($request, $user);
            $this->throwFailedAuthenticationException($request);
        }
    });
}
```

**`validateCredentials()` vs `Auth::attempt()` 的根本区别：**

| 维度 | `guard->getProvider()->validateCredentials()` | `Auth::attempt()` |
|-----|----------------------------------------------|------------------|
| 验证密码 | ✅ 是 | ✅ 是 |
| 写入 session / cookie | ❌ 否 | ✅ 是 |
| 触发 Login 事件 | ❌ 否 | ✅ 是 |
| 设置 remember cookie | ❌ 否 | ✅ 是（若 remember=true） |
| 返回值 | `bool`（密码是否匹配） | `bool`（整体是否成功） |
| 用户状态 | 仍为"访客" | 变为"已认证" |

Monica 采用此机制的目的：**在决定是否需要 2FA 挑战之前，先确认凭证有效；若需要 2FA，则将用户 ID 和 remember 标志暂存到 session，绝不提前登录。** 只有用户完成 2FA 挑战后，才由 Fortify 的 `TwoFactorAuthenticatedSessionController` 实际执行登录。

### 3. session 暂存：login.id 与 login.remember

当需要 2FA 挑战时，[RedirectIfTwoFactorAuthenticatable@twoFactorChallengeResponse](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php#L84-L96) 将以下信息暂存到 session：

```php
$request->session()->put([
    'login.id' => $user->getKey(),           // 待认证的用户 ID
    'login.remember' => $request->boolean('remember'),  // 是否需要记住我
]);
```

之后 Fortify 的 2FA 挑战控制器（TwoFactorAuthenticatedSessionController）会从 session 中取出 `login.id` 对应的用户，验证 TOTP 或恢复码，成功后再用 `login.remember` 作为参数调用 `$guard->login()`。

社交登录路径也使用相同机制，见 [AttemptToAuthenticateSocialite@twoFactorChallengeResponse](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateSocialite.php#L195-L207)。

### 4. 双因素挑战视图

双因素挑战页面由 [TwoFactorChallengeView](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Fortify/TwoFactorChallengeView.php) 渲染，在 Fortify 中注册：

- [FortifyServiceProvider.php#L56](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/FortifyServiceProvider.php#L56)

挑战视图同时准备两种 2FA 的数据：
- `twoFactor`：TOTP 是否已启用，决定是否显示 OTP 输入框
- `publicKey`：WebAuthn 的断言挑战数据（由 `Webauthn::prepareAssertion($user)` 生成），用于前端 WebAuthn API 调用
- `remember`：从 session 取出的记住我标志，透传到 2FA 表单提交

### 5. 社交登录中的 2FA 检查

社交登录路径同样插入了 2FA 检查，在 [AttemptToAuthenticateSocialite](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateSocialite.php#L51-L54) 中：

```php
if ((optional($user)->two_factor_secret && ! is_null(optional($user)->two_factor_confirmed_at))
    || Webauthn::enabled($user)) {
    return $this->twoFactorChallengeResponse($request, $user);
}
```

逻辑与邮箱密码登录一致，确保所有登录入口都经过 2FA 关卡。

### 6. WebAuthn 认证动作

[AttemptToAuthenticateWebauthn](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateWebauthn.php) 处理 WebAuthn 断言验证，支持两种场景：

- **已登录用户的二次验证**（`attemptValidateAssertion`：用 `Webauthn::validateAssertion()` 校验）
- **直接登录验证**（`attemptLogin`：委托给 `$this->guard->attempt()`，由 webauthn user provider 处理）

### 7. WebAuthn 认证事件监听

[WebauthnAuthenticateListener](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Listeners/WebauthnAuthenticateListener.php) 监听 `AuthenticatorAssertionResponseValidationSucceededEvent` 事件，每次成功的 WebAuthn 认证后更新密钥的 `used_at` 时间戳。

---

## 三、主登录速率限制器（LoginRateLimiter）

Monica 使用两套登录速率限制器：**Fortify 自带的 `LoginRateLimiter`**（用于邮箱密码和社交登录）和 **WebAuthn 包自带的 `LoginRateLimiter`**（用于 WebAuthn 登录），两者都是对 `Illuminate\Cache\RateLimiter` 的封装。

### 1. Fortify LoginRateLimiter 工作机制

`Laravel\Fortify\LoginRateLimiter` 通过构造注入到自定义动作类中：

- [RedirectIfTwoFactorAuthenticatable](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php#L20-L23)：`protected LoginRateLimiter $limiter`
- [AttemptToAuthenticateSocialite](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateSocialite.php#L34-L37)：`protected LoginRateLimiter $limiter`

当认证失败时（密码错误或社交账号冲突），调用 `$this->limiter->increment($request)` 累加失败次数：

```php
// RedirectIfTwoFactorAuthenticatable 第 59-66 行
protected function throwFailedAuthenticationException(Request $request): void
{
    $this->limiter->increment($request);  // 限流 +1
    throw ValidationException::withMessages([...]);
}
```

`increment()` 内部根据 `$request->input(Fortify::username()) . '|' . $request->ip()` 生成限流 key。

### 2. 速率限制定义

在 [FortifyServiceProvider.php#L58-L64](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/FortifyServiceProvider.php#L58-L64) 中定义了两个 RateLimiter：

**主登录速率限制（`login` limiter）：**

```php
RateLimiter::for('login', function (Request $request) {
    $email = (string) $request->email;
    return Limit::perMinute(5)->by($email.$request->ip());
});
```

- 限流维度：邮箱 + IP 组合
- 频率：每分钟 5 次
- 对应配置文件引用：[fortify.php#L104-L107](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/fortify.php#L104-L107)

**2FA 挑战速率限制（`two-factor` limiter）：**

```php
RateLimiter::for('two-factor', fn (Request $request) => Limit::perMinute(5)->by($request->session()->get('login.id')));
```

- 限流维度：session 中的 `login.id`（即待 2FA 验证的用户 ID）
- 频率：每分钟 5 次

### 3. 社交登录速率限制

社交登录路由使用独立的 `throttle:oauth2-socialite` 中间件，在 [AppServiceProvider.php#L152](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/AppServiceProvider.php#L152) 定义：

```php
RateLimiter::for('oauth2-socialite', fn (Request $request) => Limit::perMinute(5)->by(optional($request->user())->id ?: $request->ip()));
```

应用于 [web.php#L168](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/routes/web.php#L168) 的社交登录路由组。

### 4. WebAuthn 登录速率限制

WebAuthn 使用 `LaravelWebauthn\Services\LoginRateLimiter`，注入到 [AttemptToAuthenticateWebauthn](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateWebauthn.php#L21-L24)，使用 webauthn 配置中 `limiters.login` 定义的限制器。

---

## 四、会话刷新主线（Session Refresh Main Line）

本节沿着"**建立会话 → 逐请求验证 → 过渡态清理 → 延长会话 → 配置项 → 高权限二次确认**"的完整生命周期，逐层拆解三条核心机制的代码级实现。

---

### 1. 登录尾段：PrepareAuthenticatedSession 的会话建立

`Laravel\Fortify\Actions\PrepareAuthenticatedSession` 是 Fortify 登录管道的**最后一步**，Monica 未自定义此动作，在两条登录路径中复用：

- **邮箱密码路径**：[fortify.php#L149](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/fortify.php#L149) — login pipeline 第 3 步（紧接 `AttemptToAuthenticate` 之后）
- **社交登录路径**：[SocialiteCallbackController.php#L64](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Auth/SocialiteCallbackController.php#L64) — 社交 loginPipeline 第 2 步（紧接 `AttemptToAuthenticateSocialite` 之后）

它按顺序执行四个**原子动作**（不可拆分，中间无异常捕获）：

| 步骤 | 动作（Fortify v1.28.0 源码） | 安全意义 |
|-----|------------------------------|---------|
| ① | `$request->session()->regenerate()` | **防会话固定攻击**。销毁旧 session ID、生成新 ID，旧 ID 立即在 session 后端（database）失效，浏览器收到新 `laravel_session` cookie |
| ② | `$request->session()->put([$this->passwordHashSessionKey() => $user->getAuthPassword()])` | 写入密码哈希快照到 session，key 由 Fortify 配置的 guard 名决定：`login_password_hash_web`（Monica 的 fortify.guard = `web`，见 [fortify.php#L18](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/fortify.php#L18)）。供后续 AuthenticateSession 逐请求比对 |
| ③ | `Hash::needsRehash($user->getAuthPassword()) ? $user->forceFill(['password' => Hash::make($request->password)])->save() : null` | **透明哈希升级**：如果 bcrypt cost 参数或算法改变，在用户下次登录时自动用新参数重哈希并写回数据库。此步骤依赖 `AttemptToAuthenticate` 已经验证过明文密码，所以可以安全地用 `$request->password` 重做哈希 |
| ④ | `event(new Authenticated($this->config->get('fortify.guard'), $user))` | 触发 `Illuminate\Auth\Events\Authenticated` 事件，通知会话已完整就绪 |

**事件时机差异必须明确**：
- **`Login` 事件**：由 `Auth::attempt()` / `$guard->login()` 触发，在管道第 ② 步（`AttemptToAuthenticate` 或 2FA 控制器）触发 — 此时 session ID 尚未 regenerate、password_hash 尚未写入
- **`Authenticated` 事件**：在本动作第 ④ 步触发 — 此时四条会话准备工作（regenerate、hash写入、rehash、登录）全部完成

Monica 的 [LoginListener](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Listeners/LoginListener.php) 故意监听 `Login` 而非 `Authenticated`，因为它需要 `$event->remember` 参数（`Login` 事件携带，`Authenticated` 事件不携带）。

---

### 2. 逐请求验证：AuthenticateSession 的 password_hash 失效比对

Jetstream 提供的 `AuthenticateSession` 中间件在**每个受保护请求**上执行密码哈希快照比对，一旦不一致立即销毁会话。

- 配置绑定：[jetstream.php#L34](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/jetstream.php#L34) → `'auth_session' => AuthenticateSession::class`
- 中间件层叠：[web.php#L184-L188](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/routes/web.php#L184-L188)，受保护路由组三层中间件为 `auth:sanctum` → `config('jetstream.auth_session')` → `verified`

#### 完整执行流程（Jetstream v5.3.8）

```
请求进入 AuthenticateSession::handle()
│
├─ 1. $guard = $this->getGuardForRequest($request)
│     └─ 解析当前 guard（写入时用 web，比对时用同一 guard 保证 key 一致）
│
├─ 2. $user = $request->user($guard)  // 获取已认证用户
│     └─ 若 null（未认证），直接 $next($request) 跳过
│
├─ 3. $sessionHash = $request->session()->get('login_password_hash_'.$guard)
│     └─ 从 session 取出登录时写入的哈希快照
│
├─ 4. if ($user && $sessionHash !== null && $sessionHash !== $user->getAuthPassword())
│     │
│     ├─ 不一致 → 执行失效链路：
│     │    ├─ Auth::guard($guard)->logout()       // 注销 guard
│     │    ├─ $request->session()->invalidate()    // 销毁整个 session
│     │    ├─ $request->session()->regenerateToken()  // 重新生成 CSRF token
│     │    └─ redirect('/login')                  // 踢回登录页
│     │
│     └─ 一致（或 $sessionHash 为 null / $user 为 null）→ 通过，$next($request)
│
└─ 返回响应前，若有 SESSION_AUTHENTICATED_AT 则更新
```

#### 四种失效触发场景

| 场景 | 根因 | DB 侧变化 | session 侧状态 | 逐请求比对结果 |
|-----|------|----------|---------------|--------------|
| 用户改密码 | Profile 页 `UpdateUserPassword` | `$user->password` 变为新 bcrypt hash | `login_password_hash_web` 仍为旧值 | `旧值 !== 新值` → 失效 |
| 管理员重置密码 | 后台直接改 users.password | 同上 | 同上 | 同上 → 失效 |
| bcrypt cost 升级 | `config/hashing.php` 中 rounds 提高 | 用户下次登录时，PrepareAuthenticatedSession 第③步 rehash 后 DB 写入新 hash | 该用户**本次登录**的 session 在写入时已用新 hash（通过）；**其他浏览器旧会话**存的是旧 hash | 旧会话 → 失效 |
| 登出其他会话 | `LogoutOtherBrowserSessionsForm.vue` 触发 Fortify `logout-other-devices` 路由 | 内部强制改 users.password 的"密码确认 hash" | 当前浏览器 session 立即重新写 hash；其他浏览器不变 | 其他浏览器 → 失效 |

#### 无密码用户的兼容逻辑

纯社交登录用户 `$user->password === null`：
- 登录时，PrepareAuthenticatedSession 写入 `login_password_hash_web` = `null`
- 逐请求时，AuthenticateSession 比对：`null !== null` → **false**（通过）
- 即使某场景把 DB 中 password 改回非 null 值，也会触发失效 — 这是预期行为

这与 [FortifyServiceProvider.php#L44-L50](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/FortifyServiceProvider.php#L44-L50) 中 `confirmPasswordsUsing` 对无密码用户返回 `true` 的设计哲学一致：**无密码用户不存在"密码泄露后改密码踢下线"的语义**。

---

### 3. 过渡态清理：login.id / login.remember 在 2FA 通过后的 forget 时机

`login.id` 和 `login.remember` 是**纯临时状态**：仅在"凭证验证通过 → 等待 2FA 挑战"这一短暂窗口内存在于 session 中。2FA 验证通过后必须彻底清理，否则可能被重放攻击利用。

Monica 代码中这两个键在三条登录路径中的**写入 → 读取 → 清理**链路如下：

#### 路径 A：邮箱密码 → TOTP/WebAuthn 2FA 通过（Fortify 内置控制器）

**写入**：[RedirectIfTwoFactorAuthenticatable@twoFactorChallengeResponse](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php#L84-L96)

```php
$request->session()->put([
    'login.id' => $user->getKey(),
    'login.remember' => $request->boolean('remember'),
]);
// 然后 redirect → /two-factor-challenge
```

**读取与清理**：Fortify v1.28.0 内置 `TwoFactorAuthenticatedSessionController::store()`（POST `/two-factor-challenge`），依次执行：

```
1. 校验 TOTP code 或 recovery code（通过 TwoFactorLoginRequest）
   └─ 内部通过 session()->get('login.id') 取出待验证用户（注意：是 get 不是 pull，key 仍留存在 session）

2. $remember = $request->session()->pull('login.remember', false)
   └─ 用 pull() 取出 remember 并同时删除 login.remember

3. $guard->login($user, $remember)  ← 正式登录，触发 Login 事件

4. $request->session()->forget(['login.id', 'login.remember']);
   └─ 显式 forget 两个键（防御性清理：login.id 之前没被 pull 过，必须显式删；login.remember 虽已 pull 但再次删做双保险）

5. $request->session()->regenerate();
   └─ 重新生成会话 ID，彻底斩断 2FA 挑战阶段与正式登录阶段的 session 关联
```

**防御性设计**：第 4 步显式 `forget` + 第 5 步 `regenerate` 双重保险，确保即使某一步漏清理，新 session ID 也不会携带旧的 challenge 数据。

#### 路径 B：社交登录 → 2FA 通过

**写入前的过渡**：[SocialiteCallbackController](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Auth/SocialiteCallbackController.php#L33-L35) 先把 `remember` 暂存到 session：
```php
session()->put('login.remember', true);
```

**写入 challenge 键**：[AttemptToAuthenticateSocialite@twoFactorChallengeResponse](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateSocialite.php#L195-L207) 执行**原子搬运**：
```php
$request->session()->put([
    'login.id' => $user->getKey(),
    'login.remember' => $request->session()->pull('login.remember', $request->filled('remember')),
    //               ↑ 先 pull 删除旧的 login.remember，再作为值重新 put 到同一 key
]);
```
这里 `pull('login.remember', ...)` 的作用是：**从社交暂存区取出值，同时删除旧位置，再写入到 challenge 区的同一 key 名**——保证路径 B 与路径 A 的 session 键名完全一致。

**读取与清理**：与路径 A 完全相同，因为两条路径最终都 redirect 到 `route('two-factor.login')`，由同一个 `TwoFactorAuthenticatedSessionController::store()` 处理。

#### 路径 C：社交登录 → 无 2FA，直通登录

[AttemptToAuthenticateSocialite.php#L56](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateSocialite.php#L56)：

```php
$this->guard->login($user, $request->session()->pull('login.remember', false));
```

- 直接 `pull('login.remember')` 取值并清理（社交路径无 2FA，所以 `login.id` 从未被写入，无需清理）
- 之后进入 `PrepareAuthenticatedSession`，由 `regenerate()` 刷新会话 ID

#### login.id / login.remember 生命周期总览

| 阶段 | 发生位置 | login.id 状态 | login.remember 状态 |
|-----|---------|--------------|-------------------|
| 1. 进入登录管道前 | POST /login 或 社交回调前 | 不存在 | 不存在 |
| 2. 凭证验证通过后需 2FA | RedirectIfTwoFactorAuthenticatable / AttemptToAuthenticateSocialite | put → 存在 | put → 存在 |
| 3. 用户浏览 2FA 挑战页 | GET /two-factor-challenge | 存在 | 存在 |
| 4. 2FA 代码验证通过 | TwoFactorAuthenticatedSessionController::store() | pull 取出后 forget 双保险 | pull 取出后 forget 双保险 |
| 5. 正式登录后 | PrepareAuthenticatedSession | 已不存在 | 已不存在 |
| 6. 后续受保护请求 | AuthenticateSession 逐请求比对 | 保持不存在 | 保持不存在 |

---

### 4. 记住我（Remember Me）扩展

除了上述三条核心刷新机制，Monica 还对"记住我"做了 WebAuthn 专项扩展。`remember` 标志在三条登录路径中的传递链路：

- **邮箱密码登录**：`$request->boolean('remember')` → 管道第 2 步 `AttemptToAuthenticate` 传给 `Auth::attempt()`
- **社交登录**：[SocialiteCallbackController](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Auth/SocialiteCallbackController.php#L33-L35) `session()->put('login.remember', true)` → [AttemptToAuthenticateSocialite](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateSocialite.php#L56) `pull()` 取出传给 `$guard->login()`
- **2FA 通过后登录**：从 session `login.remember` pull 取出 → 传给 `$guard->login($user, $remember)`

### 2. WebAuthn 记住我扩展

WebAuthn 与记住我的交互有两处扩展：

#### a. LoginViaRemember 事件订阅

在 [AppServiceProvider.php#L157](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/AppServiceProvider.php#L157) 中注册：

```php
Event::subscribe(LoginViaRemember::class);
```

这是 `laravel-webauthn` 包提供的监听器，当用户通过 "记住我" cookie（`Auth::viaRemember()` 为 true）自动登录时，触发 WebAuthn 会话状态的自动填充。

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

参考：[LoginController.php#L33-L37](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Auth/LoginController.php#L33-L37)

### 3. PrepareAuthenticatedSession：登录尾段的会话刷新动作

`Laravel\Fortify\Actions\PrepareAuthenticatedSession` 是 Fortify 登录管道的最后一步，Monica 未自定义此动作，在两条登录路径中复用：

- **邮箱密码路径**：[fortify.php#L149](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/fortify.php#L149) 作为 login pipeline 的第 3 步
- **社交登录路径**：[SocialiteCallbackController.php#L64](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Auth/SocialiteCallbackController.php#L64) 作为社交 loginPipeline 的第 2 步

它执行以下四个原子动作（顺序执行）：

| 步骤 | 动作 | 代码含义 |
|-----|------|---------|
| ① | `$request->session()->regenerate()` | 重新生成会话 ID，**防止会话固定攻击**。旧 session ID 立即失效，浏览器收到新 cookie |
| ② | `$request->session()->put([...])` | 写入 `login_password_hash`（见下一节 AuthenticateSession 比对机制） |
| ③ | `$user->forceFill(...)` + `$user->save()` | 若用户密码哈希需要升级（如 bcrypt cost 参数调整），调用 `Hash::make()` 重新哈希并写回数据库。由 `Hash::needsRehash($user->getAuthPassword())` 判断 |
| ④ | `event(new Authenticated(...))` | 触发 `Illuminate\Auth\Events\Authenticated` 事件，通知其他监听器（如审计日志、最近登录时间更新等） |

Monica 的 [LoginListener](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Listeners/LoginListener.php) 监听的是 **`Login` 事件**（由 `Auth::attempt()` 或 `$guard->login()` 触发，发生在管道第 2 步），而非此处的 `Authenticated` 事件——两者触发时机不同：
- `Login`：表示登录成功（可能会话尚未准备好）
- `Authenticated`：表示登录+会话准备全部完成

### 4. 会话配置

会话配置见 [session.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/session.php)：

| 配置项 | 默认值 | 说明 |
|-------|--------|------|
| `driver` | `database` | 会话存储驱动 |
| `lifetime` | `120` 分钟 | 会话闲置过期时间 |
| `expire_on_close` | `false` | 关闭浏览器时是否销毁会话 |
| `cookie` | 基于 app name | 会话 cookie 名称 |
| `same_site` | `lax` | SameSite 属性 |

### 5. AuthenticateSession：password_hash 会话失效比对机制

Jetstream 提供的 `AuthenticateSession` 中间件用于在已认证会话中**逐请求**验证用户身份的有效性，核心机制是比对 session 中存储的密码哈希与数据库当前的密码哈希。

- 配置别名：[jetstream.php#L34](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/jetstream.php#L34) → `'auth_session' => AuthenticateSession::class`
- 使用位置：[web.php#L184-L188](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/routes/web.php#L184-L188)，作为受保护路由组的第二层中间件

受保护的路由组使用三层中间件：`auth:sanctum` + `config('jetstream.auth_session')` + `verified`。

#### password_hash 的写入

在登录尾段的 `PrepareAuthenticatedSession` 第 ② 步，session 被写入：

```php
// PrepareAuthenticatedSession 内部（Fortify 内置）
$request->session()->put([
    'login_password_hash_sanctum' => $user->getAuthPassword(),  // sanctum guard 对应的 key
    // 或 'login_password_hash_web' 对应 web guard
]);
```

key 名称由当前 guard 名决定（Monica 的受保护路由用 `auth:sanctum`，所以写入的是 `login_password_hash_sanctum`）。

#### password_hash 的逐请求比对

`AuthenticateSession::handle()` 在每个受保护请求上执行以下逻辑：

```
1. $user = $request->user()
   └─ 若未认证，跳过（交由 auth:sanctum 中间件处理）

2. $sessionHash = $request->session()->get('login_password_hash_'.$guardName)
   └─ 从 session 取出登录时存储的密码哈希

3. if ($sessionHash !== $user->getAuthPassword())
   ├─ 不一致 → 密码被修改过（或哈希算法升级重算过）
   │    ├─ Auth::logout()  ← 强制登出当前会话
   │    ├─ session invalidate + regenerate token
   │    └─ 跳转 /login
   └─ 一致 → 通过，继续请求
```

#### 失效场景

| 场景 | 触发条件 | 后果 |
|-----|---------|------|
| 用户修改密码 | `$user->password` 字段更新为新哈希 | 所有旧会话的 `login_password_hash_*` 与新值不等，逐请求被强制登出 |
| 管理员重置用户密码 | 同上 | 同上 |
| 哈希算法升级 | `Hash::needsRehash()` 为 true 时 PrepareAuthenticatedSession 自动重算并写回 | 该用户自己的当前会话在下次登录时写入新 hash；其他旧会话逐请求失效 |
| 登出其他浏览器会话 | 前端 [LogoutOtherBrowserSessionsForm.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/resources/js/Pages/Profile/Partials/LogoutOtherBrowserSessionsForm.vue) 触发 Fortify 的 logout-other-sessions 路由，内部重算密码哈希并强制重新登录 | 当前浏览器以外的所有会话失效 |

#### Monica 无密码用户的特殊情况

对于纯社交登录用户（`$user->password` 为 `null`）：
- `getAuthPassword()` 返回 `null`
- session 中存储的 `login_password_hash_*` 也是 `null`
- 比对 `null !== null` 为 `false`（即比对通过）
- 因此无密码用户的会话不会因"密码变更"而被意外失效

这与 [FortifyServiceProvider.php#L44-L50](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/FortifyServiceProvider.php#L44-L50) 中 `confirmPasswordsUsing` 对无密码用户放行的设计保持一致。

### 6. 密码确认（Password Confirmation）

敏感操作前的密码确认机制：

- 超时时间：`password_timeout = 10800` 秒（3 小时），见 [auth.php#L113](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/auth.php#L113)
- 自定义确认逻辑：[FortifyServiceProvider.php#L44-L50](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/FortifyServiceProvider.php#L44-L50)

```php
Fortify::confirmPasswordsUsing(fn ($user, ?string $password = null) => $user->password
    ? app(StatefulGuard::class)->validate([
        'email' => $user->email,
        'password' => $password,
    ])
    : true  // 没有密码的用户（如仅社交登录）直接通过
);
```

> 关键设计：没有设置密码的用户（纯社交登录用户）可以跳过密码确认，直接返回 `true`。

---

## 五、2FA 管理动作：启用 / 确认 / 禁用 / 恢复码

Monica 直接使用 Fortify 内置的 `TwoFactorAuthenticationController` 和 `TwoFactorQrCodeController` 处理 2FA 设置的全部生命周期动作，未替换这些控制器，仅通过 trait 和配置驱动。

### 1. 数据表字段

迁移文件 [2014_10_12_200000_add_two_factor_columns_to_users_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/database/migrations/2014_10_12_200000_add_two_factor_columns_to_users_table.php) 在 `users` 表添加了三个字段：

| 字段 | 说明 |
|-----|------|
| `two_factor_secret` | 加密存储的 TOTP 密钥（base32 编码） |
| `two_factor_recovery_codes` | 加密存储的恢复码数组（JSON 序列化） |
| `two_factor_confirmed_at` | 用户确认启用 2FA 的时间戳（需 `confirmsTwoFactorAuthentication` 为 true 才存在） |

是否创建 `two_factor_confirmed_at` 字段取决于 [Fortify::confirmsTwoFactorAuthentication()](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/database/migrations/2014_10_12_200000_add_two_factor_columns_to_users_table.php#L26)，该方法读取 fortify 配置中的 `twoFactorAuthentication['confirm']` 值（Monica 设为 `true`，见 [fortify.php#L139-L142](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/fortify.php#L139-L142)）。

### 2. TwoFactorAuthenticatable Trait

所有 2FA 数据操作逻辑封装在 `Laravel\Fortify\TwoFactorAuthenticatable` trait 中，被 [User 模型](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Models/User.php#L27) use。它提供：

- `twoFactorSecret()` / `setTwoFactorSecretAttribute()` — 自动加解密 secret
- `two_factor_recovery_codes` — 自动加解密恢复码
- `recoveryCodes()` — 解密并返回恢复码数组
- `generateRecoveryCodes()` — 生成新的 8 个恢复码
- `two_factor_enabled`（accessor）— 判断 TOTP 是否启用（secret 非空且 confirmed_at 非空）

### 3. 启用 2FA（POST /user/two-factor-authentication）

**路由名：** `two-factor.enable`

**前端触发：** [TwoFactorAuthenticationForm.vue#L39-L54](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/resources/js/Pages/Profile/Partials/TwoFactorAuthenticationForm.vue#L39-L54) 中的 `enableTwoFactorAuthentication`。

**处理流程（Fortify 内置控制器）：**
1. 验证用户已通过密码确认（session `auth.password_confirmed_at` 在有效期内）
2. 调用 `$user->forceFill(['two_factor_secret' => encrypt($secret)])` 写入加密后的 TOTP 密钥
3. 调用 `$user->generateRecoveryCodes()` 生成 8 个随机恢复码，加密后存入 `two_factor_recovery_codes`
4. **重要**：由于启用了 `confirm => true`，此时 `two_factor_confirmed_at` 仍为 `null`，2FA **尚未生效**
5. 返回后前端并行请求 QR 码、setup key 和恢复码

**启用后的 3 个 GET 接口：**
- `GET /user/two-factor-qr-code`（`two-factor.qr-code`）：返回内联 SVG 二维码，内容为 `otpauth://totp/...` URI
- `GET /user/two-factor-secret-key`（`two-factor.secret-key`）：返回明文 base32 密钥，供无法扫码的用户手动输入
- `GET /user/two-factor-recovery-codes`：返回解密后的 8 个恢复码

### 4. 确认 2FA（POST /user/two-factor-authentication/confirm）

**路由名：** `two-factor.confirm`

**前端触发：** [TwoFactorAuthenticationForm.vue#L74-L85](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/resources/js/Pages/Profile/Partials/TwoFactorAuthenticationForm.vue#L74-L85) 中的 `confirmTwoFactorAuthentication`，用户输入手机 App 生成的 6 位 OTP 码后提交。

**处理流程（Fortify 内置控制器）：**
1. 再次验证密码确认状态
2. 使用 `paragonie/constant_time_encoding` 和 `spomky-labs/otphp` 验证 OTP 是否与加密的 secret 匹配
3. 若验证通过，将 `two_factor_confirmed_at` 设为当前时间戳
4. 此时 `two_factor_enabled` accessor 返回 `true`，2FA 正式生效

**这一步的安全意义：** 强制用户用实际 TOTP 设备生成一次验证码，证明用户已正确配置了验证器 App，避免因用户未扫码导致启用后无法登录。

### 5. 禁用 2FA（DELETE /user/two-factor-authentication）

**路由名：** `two-factor.disable`

**前端触发：** [TwoFactorAuthenticationForm.vue#L91-L104](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/resources/js/Pages/Profile/Partials/TwoFactorAuthenticationForm.vue#L91-L104) 中的 `disableTwoFactorAuthentication`。

**处理流程（Fortify 内置控制器）：**
1. 验证密码确认状态
2. 将 `two_factor_secret`、`two_factor_recovery_codes`、`two_factor_confirmed_at` 全部置为 `null`

### 6. 重新生成恢复码（POST /user/two-factor-recovery-codes）

**路由名：** `two-factor.recovery-codes`

**前端触发：** [TwoFactorAuthenticationForm.vue#L87-L89](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/resources/js/Pages/Profile/Partials/TwoFactorAuthenticationForm.vue#L87-L89) 中的 `regenerateRecoveryCodes`。

**处理流程（Fortify 内置控制器）：**
1. 验证密码确认状态
2. 调用 `$user->generateRecoveryCodes()` 生成全新的 8 个恢复码，加密后覆盖旧值
3. 旧恢复码立即失效

### 7. 测试佐证

[TwoFactorAuthenticationSettingsTest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/tests/Feature/Auth/TwoFactorAuthenticationSettingsTest.php) 覆盖了上述全部场景：
- `two_factor_authentication_can_be_enabled`：验证启用后 secret 非空且生成 8 个恢复码
- `recovery_codes_can_be_regenerated`：验证重新生成后新旧恢复码数组无交集
- `two_factor_authentication_can_be_disabled`：验证禁用后 secret 被清空

---

## 六、社交登录绑定动作

### 1. 支持的社交登录提供商

配置于 [auth.php#L127](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/auth.php#L127) 的 `login_providers` 数组，通过环境变量 `LOGIN_PROVIDERS` 配置（逗号分隔）。

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
| `saml2` | —（特殊） | SAML 2.0，见下文 |

### 2. Socialite 扩展注册

在 [AppServiceProvider.php#L159-L165](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/AppServiceProvider.php#L159-L165) 中，通过监听 `SocialiteWasCalled` 事件注册各个第三方 Socialite 扩展：

```php
Event::listen(SocialiteWasCalled::class, AzureExtendSocialite::class);
Event::listen(SocialiteWasCalled::class, FacebookExtendSocialite::class);
Event::listen(SocialiteWasCalled::class, GitHubExtendSocialite::class);
Event::listen(SocialiteWasCalled::class, GoogleExtendSocialite::class);
Event::listen(SocialiteWasCalled::class, LinkedInExtendSocialite::class);
Event::listen(SocialiteWasCalled::class, KanidmExtendSocialite::class);
Event::listen(SocialiteWasCalled::class, KeycloakExtendSocialite::class);
```

### 3. SAML2 登录通路

**注意：SAML2 在当前代码中仅配置了凭证和多语言名称，未注册 Socialite 扩展，也未发现 `Saml2ExtendSocialite` 或专门的 SAML2 Socialite 驱动被 `Event::listen` 注册。** 这意味着 SAML2 的实际登录通路处于**配置预留但未接通**的状态。

已配置但缺失的部分：

- ✅ 已有配置：[services.php#L73-L81](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/services.php#L73-L81) 中定义了 `name`、`metadata`、`acs`、`entityid`、`certificate`、`sp_acs`、`logo`
- ✅ 已有多语言：[lang/en/auth.php#L15](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/lang/en/auth.php#L15) 定义了 `login_provider_saml2`
- ✅ 已有 Logo：`public/img/auth/saml2.svg`
- ❌ 缺少：`SocialiteWasCalled` 事件监听注册 SAML2 驱动
- ❌ 缺少：composer.json 中 `socialiteproviders/saml2` 等 SAML2 专用包
- ❌ 缺少：`Saml2ExtendSocialite` 类或等价的驱动扩展

如果在 `LOGIN_PROVIDERS` 环境变量中包含 `saml2`，登录页会显示 SAML2 按钮，但点击后 `Socialite::driver('saml2')` 将抛出 "Driver [saml2] not supported." 异常。

### 4. 社交登录路由

路由定义于 [web.php#L168-L172](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/routes/web.php#L168-L172)，受 `throttle:oauth2-socialite` 速率限制（每分钟 5 次）：

```php
Route::get('auth/{driver}', [SocialiteCallbackController::class, 'login'])->name('login.provider');
Route::get('auth/{driver}/callback', [SocialiteCallbackController::class, 'callback']);
Route::post('auth/{driver}/callback', [SocialiteCallbackController::class, 'callback']);
```

`GET /auth/{driver}` 跳转到 OAuth 服务商；`GET/POST /auth/{driver}/callback` 接收回调。

### 5. 社交登录控制器

[SocialiteCallbackController](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Auth/SocialiteCallbackController.php) 处理登录跳转和回调：

- **login 方法**（第 24 行）：校验 provider 有效性 → 保存 redirect URL → 保存 remember 到 session → 跳转到 OAuth 服务商
- **callback 方法**（第 43 行）：校验 provider → 检查 OAuth 错误 → 通过自定义管道登录 → 跳转目标页
- **loginPipeline 方法**（第 60 行）：使用 Laravel Pipeline 模式构建认证流程

### 6. 社交登录认证管道

回调处理使用自定义的两步管道：

```php
// SocialiteCallbackController.php
protected function loginPipeline(Request $request): Pipeline
{
    return (new Pipeline(app()))->send($request)->through([
        AttemptToAuthenticateSocialite::class,  // 社交认证
        PrepareAuthenticatedSession::class,      // 准备会话（复用 Fortify 的会话准备动作）
    ]);
}
```

### 7. AttemptToAuthenticateSocialite 核心逻辑

[AttemptToAuthenticateSocialite](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateSocialite.php) 是社交登录的核心动作，处理三种绑定场景：

#### 场景 1：已有绑定 → 直接登录

- 第 80-88 行：通过 `driver_id` + `driver` 在 `user_tokens` 表查找
- 如果找到关联记录，获取对应 `user`
- 冲突检测（第 102-107 行）：若当前已登录用户与 token 所属用户不同，抛出错误

#### 场景 2：新用户 + 未登录 → 创建账号并绑定

- 第 91-93 行：通过 `CreateNewUser` 创建新用户（复用注册逻辑）
- 通过 `createUserToken` 创建绑定记录
- 触发 `Registered` 事件（第 134 行）

#### 场景 3：已登录用户 + 新提供商 → 追加绑定

- 第 112-116 行 `getUserOrCreate()`：如果当前已认证，直接使用当前用户
- 为该用户添加新的社交令牌

#### 冲突检测

- 第 102-107 行：如果已登录用户尝试绑定的社交账号已属于另一个用户，抛出 `This provider is already associated with another account` 错误

#### 2FA 检查

- 第 51-54 行：社交登录同样经过 TOTP/WebAuthn 的 2FA 检查，逻辑与邮箱密码登录一致

#### 速率限制

- 第 171-180 行：认证失败时通过 `$this->limiter->increment($request)` 累加计数，并抛出 `ValidationException`

### 8. 用户令牌模型

[UserToken](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Models/UserToken.php) 模型存储社交登录绑定信息，字段包括：

- `driver` / `driver_id` — 提供商名称和用户在提供商处的 ID
- `email` — 社交账号邮箱
- `format` — `oauth1` 或 `oauth2`
- `token` / `token_secret` / `refresh_token` / `expires_in` — OAuth 令牌

令牌的加密字段在 `$hidden` 属性中，API 不返回。

### 9. 解绑（取消绑定）

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

### 10. 用户资料页展示

[UserProfile](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Jetstream/UserProfile.php) 是 Jetstream `whenRendering` 钩子，在 Profile 页面注入社交登录提供商列表、已绑定令牌、WebAuthn 密钥数据：

- 通过 `Jetstream::inertia()->whenRendering('Profile/Show', new UserProfile)` 注册，见 [JetstreamServiceProvider.php#L33](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/JetstreamServiceProvider.php#L33)

---

## 七、Fortify / Jetstream 其他自定义钩子

### Fortify 自定义汇总

注册位置：[FortifyServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/FortifyServiceProvider.php)

| 钩子 | 自定义类/回调 | 用途 |
|-----|-------------|------|
| `loginView` | LoginController | 自定义登录页视图（Inertia，含 WebAuthn 自动登录逻辑） |
| `registerView` | RegisterController | 自定义注册页视图 |
| `createUsersUsing` | CreateNewUser | 用户创建逻辑（委托给 CreateAccount 服务） |
| `updateUserProfileInformationUsing` | UpdateUserProfileInformation | 更新用户资料 |
| `updateUserPasswordsUsing` | UpdateUserPassword | 更新密码 |
| `resetUserPasswordsUsing` | ResetUserPassword | 重置密码 |
| `confirmPasswordsUsing` | 闭包 | 密码确认逻辑（支持无密码用户直接通过） |
| `twoFactorChallengeView` | TwoFactorChallengeView | 2FA 挑战视图（含 TOTP + WebAuthn 双模式） |
| 注册路由补丁 | `patchFortifyRoutes()` | 给注册路由加 `monica.signup_is_enabled` 中间件 |

### Jetstream 自定义汇总

注册位置：[JetstreamServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/JetstreamServiceProvider.php)

| 钩子 | 自定义类/值 | 用途 |
|-----|------------|------|
| `deleteUsersUsing` | DeleteUser | 用户删除逻辑 |
| `whenRendering('Profile/Show')` | UserProfile | 资料页注入 2FA、社交登录、WebAuthn 数据 |
| `whenRendering('API/Index')` | 闭包 | API 页注入数据 |
| `defaultApiTokenPermissions` | `['read']` | 默认 API 令牌权限 |
| `permissions` | `['read', 'write']` | 可用权限列表 |

### 其他全局认证钩子

| 位置 | 钩子类型 | 用途 |
|-----|---------|------|
| [AppServiceProvider.php#L137](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/AppServiceProvider.php#L137) | `RedirectIfAuthenticated::redirectUsing` | 已认证用户重定向目标（vault 列表） |
| [AppServiceProvider.php#L154-L155](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/AppServiceProvider.php#L154-L155) | Webauthn::updateViewResponseUsing / destroyViewResponseUsing | WebAuthn 操作后的响应 |
| [AppServiceProvider.php#L157](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/AppServiceProvider.php#L157) | `Event::subscribe(LoginViaRemember::class)` | WebAuthn 记住我登录 |

---

## 八、登录流程图（邮箱密码路径）

```
用户 POST /login (email, password, remember)
    │
    ▼
Fortify login pipeline（Monica 自定义顺序）
    │
    ├─ ① RedirectIfTwoFactorAuthenticatable
    │      │
    │      ├─ User::where('email')->first()       ← 按邮箱取用户
    │      ├─ guard->getProvider()->validateCredentials($user, $password)
    │      │      ↑ 仅验证密码哈希，不登录、不写 session
    │      ├─ 失败 → limiter->increment() + 抛 ValidationException
    │      │
    │      ├─ 检查 TOTP (two_factor_secret && two_factor_confirmed_at)
    │      │   或 WebAuthn (Webauthn::enabled($user))
    │      │
    │      ├─ 需要 2FA：
    │      │    ├─ session(['login.id' => $user->id, 'login.remember' => true/false])
    │      │    ├─ event(TwoFactorAuthenticationChallenged)
    │      │    └─ redirect /two-factor-challenge
    │      │
    │      └─ 不需要 2FA：
    │           └─ return $next($request)  ← 传递给管道下一步
    │
    ├─ ② AttemptToAuthenticate（Fortify 内置）
    │      │
    │      └─ Auth::attempt($credentials, $remember)
    │           ↑ 实际登录：写 session、触发 Login 事件、写 remember cookie
    │
    └─ ③ PrepareAuthenticatedSession（Fortify 内置）
           ├─ regenerate session
           ├─ 更新 password_hash_session 字段（若密码哈希算法升级）
           └─ 触发 Authenticated 事件
```

---

## 九、关键文件索引

| 文件路径 | 说明 |
|---------|------|
| [config/fortify.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/fortify.php) | Fortify 配置（自定义管道、特性开关、guard） |
| [config/jetstream.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/jetstream.php) | Jetstream 配置（guard、AuthenticateSession 中间件） |
| [config/auth.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/auth.php) | 认证配置（guards、webauthn provider、密码超时、社交提供商） |
| [config/webauthn.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/webauthn.php) | WebAuthn 配置 |
| [config/services.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/services.php) | 社交登录凭证（含 SAML2） |
| [config/session.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/config/session.php) | 会话配置 |
| [app/Providers/FortifyServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/FortifyServiceProvider.php) | Fortify 钩子注册 + 速率限制定义 |
| [app/Providers/JetstreamServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/JetstreamServiceProvider.php) | Jetstream 钩子注册 |
| [app/Providers/AppServiceProvider.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Providers/AppServiceProvider.php) | 全局钩子（Socialite、WebAuthn、重定向、速率限制） |
| [app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Fortify/RedirectIfTwoFactorAuthenticatable.php) | 自定义 2FA 前置检查（validateCredentials 机制） |
| [app/Actions/Fortify/TwoFactorChallengeView.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Fortify/TwoFactorChallengeView.php) | 2FA 挑战视图（TOTP + WebAuthn） |
| [app/Actions/Fortify/CreateNewUser.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Fortify/CreateNewUser.php) | 用户创建（注册和社交登录共用） |
| [app/Actions/AttemptToAuthenticateSocialite.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateSocialite.php) | 社交登录认证动作（绑定/登录/2FA） |
| [app/Actions/AttemptToAuthenticateWebauthn.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/AttemptToAuthenticateWebauthn.php) | WebAuthn 认证动作 |
| [app/Actions/Jetstream/UserProfile.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Actions/Jetstream/UserProfile.php) | Profile 页面注入数据（含 2FA、社交绑定） |
| [app/Http/Controllers/Auth/LoginController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Auth/LoginController.php) | 登录页（含 WebAuthn autologin 检测） |
| [app/Http/Controllers/Auth/SocialiteCallbackController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Auth/SocialiteCallbackController.php) | 社交登录控制器 |
| [app/Http/Controllers/Profile/UserTokenController.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Controllers/Profile/UserTokenController.php) | 社交账号解绑 |
| [app/Listeners/LoginListener.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Listeners/LoginListener.php) | Login 事件 → 设置 return cookie |
| [app/Listeners/WebauthnAuthenticateListener.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Listeners/WebauthnAuthenticateListener.php) | WebAuthn 认证 → 更新 used_at |
| [app/Http/Middleware/SanctumSetUser.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Http/Middleware/SanctumSetUser.php) | 同步用户到 Sanctum guard（TransientToken） |
| [app/Models/User.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Models/User.php) | User 模型（TwoFactorAuthenticatable + HasApiTokens + WebauthnAuthenticatable） |
| [app/Models/UserToken.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Models/UserToken.php) | 社交令牌模型 |
| [app/Models/WebauthnKey.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/app/Models/WebauthnKey.php) | WebAuthn 密钥模型 |
| [resources/js/Pages/Profile/Partials/TwoFactorAuthenticationForm.vue](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/resources/js/Pages/Profile/Partials/TwoFactorAuthenticationForm.vue) | 2FA 管理前端（启用/确认/禁用/恢复码） |
| [database/migrations/2014_10_12_200000_add_two_factor_columns_to_users_table.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/database/migrations/2014_10_12_200000_add_two_factor_columns_to_users_table.php) | 2FA 字段迁移 |
| [tests/Feature/Auth/TwoFactorAuthenticationSettingsTest.php](file:///d:/fz/0601-1/solo-dogfeeding/code/83-monica/tests/Feature/Auth/TwoFactorAuthenticationSettingsTest.php) | 2FA 管理功能测试 |
