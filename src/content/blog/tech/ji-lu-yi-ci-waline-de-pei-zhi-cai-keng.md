---
title: 记录一次Waline的配置踩坑
draft: false
sticky: false
tocNumbering: true
excludeFromSummary: false
math: false
quiz: false
date: 2026-02-21 02:49:49
updated: 2026-02-21 02:50:06
categories:
  - 技术
tags:
  - waline
  - vercel
  - 踩坑
---

最近在博客上部署 Waline 评论系统，顺手配置了 Turnstile 人机验证，结果第二天想登录后台发现 API 请求基本全返回 403，无法登录也无法注册。折腾了半天，最终定位到可能是 Vercel 和 Waline 配合的不是很好。记录一下完整的排查过程，希望能帮到有相同困扰的人。

一开始服务端没有配置`TURNSTILE_KEY`，Turnstile 控制台也没配置后台的域名，所以我以为是 Turnstile 验证失败导致的 403。于是按照文档配置了环境变量，结果依然 403。

怀疑是`SERVER_URL`自动生成的地址错误，于是我自己填了一个环境变量，又出现了`Unexpected token '<', "<!doctype "... is not valid JSON`的错误，但是实际上是因为漏掉了 https:// 前缀，导致请求这个 URL 被当成了 API 端点接到了自动生成的地址后面。把`SERVER_URL`设置正确之后就又回到了 403 的错误。

下面是!!走上正轨的!!{.blur}排查过程。

## 环境信息

- Waline 服务端：部署在 Vercel
- 配置的环境变量：`TURNSTILE_KEY`、`TURNSTILE_SECRET`、`SECURE_DOMAINS`

## 现象

配置完上述环境变量后，访问 Waline 服务端，所有登录相关 API 请求都返回 **403 Forbidden**。同时浏览器控制台还出现了两个奇怪的请求失败：

- `https://challenges.cloudflare.com/cdn-cgi/challenge-platform/h/g/pat/...` 返回 401
- `https://waline.hotaron.top/cdn-cgi/challenge-platform/h/g/rc/...` 返回 404

---

## 排查过程

### 第一步：搞清楚 /api/token 在什么情况下会返回 403

翻了一下 Waline 的源码，403 的触发逻辑全部集中在 `packages/server/src/logic/base.js` 的 `__before()` 钩子中，在每个请求处理之前执行。一共有三种情况会触发 403：

**情况一：`referrerCheck` 失败**

```javascript
async __before() {
  const referrerCheckResult = this.referrerCheck();
  if (!referrerCheckResult) {
    return this.ctx.throw(403);
  }
  // ...
}
```

当配置了 `SECURE_DOMAINS` 后，每个请求的 `Referer` 或 `Origin` 头都必须在白名单中，否则直接 403。

**情况二：配置了验证码但请求体中缺少 token**

```javascript
async useRecaptchaOrTurnstileCheck({ secret, token, api, method }) {
  if (!token) {
    return this.ctx.throw(403);
  }
  // ...
}
```

**情况三：验证码 token 校验失败**

```javascript
if (!response.success) {
  return this.ctx.throw(403);
}
```

### 第二步：排除 Turnstile 的嫌疑

两个"奇怪"的请求报错其实都不是 Waline 的问题：

- `challenges.cloudflare.com/cdn-cgi/challenge-platform/h/g/pat/` 返回 **401**：这是 Cloudflare Turnstile 内部的 PAT（Private Access Token）流程，浏览器不支持时会自动降级，**属于正常现象**。
- `waline.hotaron.top/cdn-cgi/challenge-platform/h/g/rc` 返回 **404**：这个路径只存在于 Cloudflare 边缘网络，Vercel 上当然没有，**与 Waline 代码无关**。

于是把 `TURNSTILE_KEY` 和 `TURNSTILE_SECRET` 都删掉，依然 403。所以问题根本不在 Turnstile。

### 第三步：定位 SECURE_DOMAINS 的问题

继续深扒 `referrerCheck()` 的逻辑：

```javascript
referrerCheck() {
  let { secureDomains } = this.config();
  if (!secureDomains) {
    return true; // 未配置则放行所有请求
  }

  const referrer = this.ctx.referrer(true); // 获取请求的 Referer 头（hostname 部分）
  let { origin } = this.ctx;
  if (origin) {
    try {
      const parsedOrigin = new URL(origin);
      origin = parsedOrigin.hostname; // 解析 Origin 为纯 hostname
    } catch (err) { ... }
  }

  // ...白名单处理...

  // 有 referrer 检查 referrer，没有则检查 origin
  const checking = referrer || origin;
  const isSafe = secureDomains.some((domain) =>
    think.isFunction(domain.test) ? domain.test(checking) : domain === checking,
  );

  if (!isSafe) {
    return this.ctx.throw(403); // ← 这里有问题
  }
}
```

这时我意识到，我的请求中的 `Referer` 和 `Origin` 可能都被修改了，导致 `checking` 变量是空字符串，和白名单中的域名完全不匹配，直接被判定为非法来源。

当请求没有携带 `Referer` 或 `Origin` 头时（比如直接用 curl 调用、服务端代理转发、浏览器隐私策略裁掉了裁掉了 Referer 等场景），`referrer` 和 `origin` 都为空：

```plain
checking = "" || "" = ""
```

然后用空字符串去和白名单中所有域名比较，全部不匹配，直接 403。这就是问题所在。

我配置的 `SECURE_DOMAINS` 域名本身没有问题，但 Vercel Serverless / Cloudflare 代理的环境下，可能存在部分请求没有 `Referer`/`Origin` 头或者被修改的情况，导致所有这些请求全部被误判为非法来源，返回 403。

---

## 解决方案

**直接删掉 `SECURE_DOMAINS` 环境变量。**

`SECURE_DOMAINS` 本来就是可选的安全加固配置，不配置时默认允许所有来源，这是 Waline 的正常行为。在 Vercel 环境下，由于这个问题的存在，配置了反而会导致合法请求被误拦截。

删掉之后，Turnstile 依然可以正常配置和使用，两者完全独立。

~~!!可恶的云服务商又来剥夺我的睡眠时间!!{.blur}~~