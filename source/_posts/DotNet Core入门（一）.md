---
title: .Net Core 入门（一）
url_name: dotnet-core-01
date: 2018-05-28 19:50:55
categories:
  - .Net Core
tags:
  - .Net Core
  - C#
  - 入门
  - 很久以前
---

公司新的.Net Core 项目需要修改站点的认证授权方式，以实现集团的 SSO 登录。

通过对.Net Core 认证授权机制的学习，我目前知道了可以使用的两种认证方式： `CookieAuthentication` 和 `JwtTokenAuthentication` 。

<!-- more -->

其中 Cookie 的方式是为 MVC 站点提供的，当站点登录之后站点请求登录的服务获取用户的登录信息，存在 Cookie 中作为每次站点请求的登录依据；JwtToken 的方式适用于 Web Api 项目，请求的移动端设备，通过请求站点获取授权的 Token 来调用站点服务。并且由于目标是为了实现公司内部的 SSO，所以并没有遵循 OAuth2 的协议，就是完全通过请求登录 API 种下 Cookie 来实现单点登录。因此，授权的方式就使用了 `CookieAuthentication`。

## 项目实现

但是在刚开发的时候存在一个业务逻辑问题，集团虽然确定了站点需要实现 SSO 但并没有给出具体的登录服务从何处提供，有可能是其他组做，也有可能我们自己查数据源。但是项目需要继续开发啊，所以在做这个逻辑的时候肯定就会伴随着需求的变更。但是整个站点的登录流程其实都可以简化为这样：
![登录流程](https://image.dunbreak.cn/past/login-process.png)

从请求的流程中可以知道，有可能随业务改变的只有 `查询用户信息` 和 `验证用户信息` 这两个步骤，其余的步骤 `CookieAuthentication` 确定了之后就基本不会大改动了。因此写了 4 个登录相关的接口： `IAuthorityService` 权限验证服务、 `IDataAnalyzeService` 权限数据解析服务、 `IDataQueryService` 权限数据查询服务、 `ICookieService` Cookie 操作服务。
![IAuthorityService](https://image.dunbreak.cn/past/auth-UML.png)

从 UML 图中可以看到，其他服务基本都是为 `IAuthorityService` 服务的。 `IAuthorityService` 定义了登录的相关操作，用来站点的登录、登出、验证是否登录等和权限相关的操作。但是验证登录需要有用户的登录信息，因此定义了 `IDataQueryService` 服务来提供登录的信息；在登录时需要写入 Cookie 信息，因此需要定义 `ICookieService` 服务来对 Cookie 进行读写操作； 根据[CookieAuthentication](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/cookie?view=aspnetcore-2.0&tabs=aspnetcore2x) 规定，写入的 Cookie 信息必须是一个 `Claim` 的集合，因此还需要 `IDataAnalyzeService` 服务来解析用户的登录信息为 `Claim` 对象写入 Cookie。此外，`IAuthorityService` 和 `ICookieService` 实现了 `IServiceProvider` ，因此这两个服务是提供给上层调用的。站点需要登录时，只需要 Resolve 出 `IAuthorityService` 服务就可以了， 并且也可以单独操作 Cookie 信息。

在刚开始的版本中，由于还没有确定登录信息的数据源，为了赶上项目的测试进度，登录的信息还是按照久站点的逻辑直接从数据库中查询的。在日后权限组提供登录信息微服务后，直接重新实现了一个新的 `IDataQueryService` 整个站点就可以直接运行了。从这个方面，第一次体验到了控制反转的强大和好处。

## 存在的问题

项目虽然顺利的开发完毕了，但是在日后上线正式使用 `CookieAuthentication` 时却存在一个很致命的问题。由于正式站点是多机轮询的，因此每次访问的服务器都是不一样的，这就直接导致了服务器生成的认证 Cookie 是不一样的，这就直接导致了站点登录失败，每次登录都会跳转到登录失败的页面。这个问题出现的时候我很不理解，不明白为什么会出现这种错误，还以为是自己的代码写的有问题，但是在测试和本地都没问题，之后通过抓包发现每次的种下的 Cookie 的值是不一样的，查了一下发知道 `CookieAuthentication` 中，Cookie 的生成规则是根据服务器的机器码为种子加密而成的，多机环境下，每台服务器的机器码都不相同，因此导致每台服务器生成的 Cookie 都不一样，因此站点会验证失败。

这个问题是在上线正式的前一晚测试发现了，我记得很清楚那晚加班到凌晨 5 点。Google 到了这个 [Issue](https://github.com/aspnet/Security/issues/624)（竟然有一个国内的大哥和我遇到了一样的问题，真是泪目啊，哈哈），告诉我 `多机情况下需要配置CookieShareable，并且提供相同的数据保护机制`。按照 Issue 里给解答的帖子 [Share authentication cookies among ASP.NET Core apps](https://docs.microsoft.com/en-us/aspnet/core/security/cookie-sharing?view=aspnetcore-2.0&tabs=aspnetcore2x#share-authentication-cookies-among-aspnet-core-apps) 给出的方法改了一下没有效果。之后发现也和题主一样没有效果，之后按照 Issue 说的 `修改多机的机器码来实现数据加密一致`，还是没法解决问题。

最终这个问题被遗留了，上线前无法解决，只能申请运维同事为多机加入了 `Session保持` 的机制，强制性通过 Cookie 对多机的 `Session进行同步` 暂时性的解决了这个问题。但是公司规定服务器是不能开 `Session同步` 功能的，因此这个问题还需要解决。

之后有看到一个贴子说（时间太久找不到了）可以改 `CookieAuthentication` 为 `JwtTokenAuthentication` ，单独出一台认证授权服务器来解决这个问题。很遗憾，这样改工程量浩大，需要动底层的不少逻辑，并且需要单独服务器部署，如果那台服务器挂了整个站点也都挂了，组长不想承担这么大的风险，并且业务逻辑也在不断推进，没有这么多时间来改这个逻辑。

机缘巧合，在某个.Net Core 的学习群里问了一下这个问题，有位大哥说可以自定义 Cookie 的生成规则来解决这个问题，因为微软默认的生成规则是根据机器码加密的，可以改这个生成规则为自定义的，这样多台服务器的 Cookie 就会生成一致了。 并且自定义方法也很简单，只要在 `CookieAuthentication` 初始化的时候重新赋值 `TicketDataFormat` 属性就可以了：

```csharp
services.AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
        .AddCookie(options =>
        {
            options.Cookie.Domain = cookieDomain;
            options.Cookie.Name = "CookieName";
            options.LoginPath = "/Home/Index/";
            options.AccessDeniedPath = "/Home/Index/";
            options.ExpireTimeSpan = new TimeSpan(0, cookieTimeSpan, 0);
            //只需要自定义这个属性即可
            options.TicketDataFormat = new CustomCookieAuthentication(Configuration, services);
        });
```

`TicketDataFormat` 是一个 `ISecureDataFormat<TData>` 属性，其中有四个方法：

```csharp
namespace Microsoft.AspNetCore.Authentication
{
    public interface ISecureDataFormat<TData>
    {
        string Protect(TData data);
        string Protect(TData data, string purpose);
        TData Unprotect(string protectedText);
        TData Unprotect(string protectedText, string purpose);
    }
}
```

自定义的生成规则只需要实现 `ISecureDataFormat<AuthenticationTicket>` 即可。使用上述的方法确实解决了多机 Cookie 生成不一致的问题，但是却也带来了另外一个问题：由于修改了 `认证的规则` ，直接导致了 DataProtection 功能失效了，导致 `授权` 失效了。不管在 Controller 类中加 `Authorize` 还是 `AllowAnonymous` 特性都得不到 `授权` 的效果了。这个原因我没有查到，但是我认为应该是我修改数据格式化之后和 `DataProtection` 规则不一致导致的。很遗憾这个问题目前还没有解决，我只能通过修改 `BaseController` 中的登录信息验证来临时性解决这个遗留问题。

## 未来改进方向

上一节最后说了，自定义了 `TicketDataFormat` 导致了一个站点 `授权失效` 的问题。这个问题我目前能想到两种解决方案：

1. 学习 `CookieAuthentication` 和 `DataProtection` 源码，了解其授权规则并且对应修改 `TicketDataFormat`
2. 搭建微服务项目，将用户认证做成一个站点内部的微服务进行调用

我尝试了阅读源码，但是代码涉及的项目太多，导致我完全找不到真正的数据保护机制的核心代码和生成规则，因此我希望用搭建微服务框架的方式解决这个问题，类似于之前说过的 `JwtToken` 的方式。因为我最近也正在学习.Net Core 的微服务项目搭建，所以这个逻辑正好可以作为我的入门起手项目。
