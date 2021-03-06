---
title: .Net Core 入门（三）
url_name: dotnet-core-03
subtitle: 编写微服务接口请求服务
date: 2018-05-30 19:52:57
categories:
  - .Net Core
tags:
  - .Net Core
  - C#
  - 入门
  - 很久以前
---

由于.Net Core 项目中一半以上的逻辑被微服务实现，因此为了调用这些微服务就需要一个微服务的调用服务。当前的微服务全都是 RESTful 的 Http 协议 Api，所以调用微服务其实就是实现一个 WebRequest 的方法，技术总监还要求方法内需要加入日志监控等功能。

<!-- more -->

但是同样的，我还是不想这么简单地实现一个 WebRequest 的封装的方法，这样没有任何扩展性，并且使用起来也很不方便。如果只是封装一个 WebRequest，那这个方法干的事肯定就是：输入请求接口地址和参数获取结果 string，然后再处理 string 为一个对象或者什么。这样做使用性太差了，大量的微服务接口的配置项散落在各个逻辑代码中很难管理，并且调用方法很难复用，有可能一个逻辑中调用了一个微服务接口，另一个地方要调用只能把代码复制一遍，日后如果要修改成本也很高。本着易用的目的，我开始编写一个微服务请求的服务。

## 确定调用方法

我想了一下想调用一个微服务的接口至少需要确定一下几个元素：

- 请求参数

- 响应结果

- 请求地址

- 请求方式

- 权限验证

- 超时时间

- 作者

其中前五个是必须存在的信息，没有这些信息一个微服务是无法被调用的，后两个是用于日志记录跟踪使用的额外参数。那么，先来看一下现有的微服务的 `请求参数` 和 `响应结果` 可以发现：

- 请求参数：大部分的微服务中规定 GET 请求方式参数就是直接通过 Url 地址传递过去；POST 请求方式参数是一个模型，通过 Json 序列化之后传过去。那既然是这样，其实 GET 请求方式也可以把参数封装成一个模型类，只不过在调用的时候把类中的参数序列化一下就好

- 响应结果：大部分微服务 GET/POST 的输出都是一个模型的 Json 序列化结果；小部分是 XML 格式的，但是也是一个模型的序列化结果

因此，微服务的请求方法其实可以简单定义成这样：

```csharp
/// <summary>
/// 微服务请求方法
/// </summary>
/// <param name="requestModel">请求参数</param>
/// <returns>响应结果</returns>
ResponseModel Invoke(RequestModel requestModel);
```

但是光这样肯定是不够的，有了请求响应，没有其他的请求地址、请求方式等还是无法调用微服务接口，但是这些信息不应该加在请求的 `RequestModel` 中，因为这个 Model 是需要序列化传递的，加入了这些参数就影响了序列化结果，并且也不好获取结果。但其实可以发现，无论是请求地址、请求方式、权限验证等参数都是微服务接口自身的属性，不会因为调用参数的改变而改变，也就是说他们是和特定的微服务接口绑定的，不管怎么调用这几个参数都这么传递。与之相似的，每次调用都需要传递 `RequestModel` ，只不过参数的值不一样，所以我想到了将这些参数加入 `RequestModel` 的类特性进行传递，于是有了这样一个特性 `InterfaceRequestAttribute`：

```csharp
/// <summary>
/// 接口请求特性
/// </summary>
[AttributeUsage(AttributeTargets.Class)]
public class InterfaceRequestAttribute : Attribute
{
    /// <summary>
    /// 所调用的接口在配置文件中的文件Key，接口排重唯一标识
    /// </summary>
    public string ConfigKey { get; private set; }
    /// <summary>
    /// 接口名称
    /// </summary>
    public string InterfaceName { get; private set; }
    /// <summary>
    /// 在没有配置的情况下默认的接口Url地址
    /// </summary>
    public string DefaultUrl { get; private set; }
    /// <summary>
    /// 接口是否记录日志
    /// </summary>
    public bool IsLog { get; private set; }
    /// <summary>
    /// 接口请求方式，默认为GET
    /// </summary>
    public InvokeMethodEnum InvokeMethod { get; private set; }
    /// <summary>
    /// 接口请求类型，默认为JSON
    /// </summary>
    public InvokeContentTypeEnum InvokeType { get; private set; }

    /// <summary>
    /// 接口独立超时时间，如果小于默认配置则走默认配置
    /// </summary>
    public int TimeOut { get; private set; }

    /// <summary>
    /// 响应数据格式，默认JSON
    /// </summary>
    public ResponseDataTypeEnum ResponseType { get; private set; }

    /// <summary>
    /// 参数编码方式，默认是UTF-8格式
    /// </summary>
    public Encoding EncodingType { get; private set; }
}
```

我完全可以在 `RequestModel` 中加上这个特性，让其特定表示为一个微服务的请求参数模型，然后在接口请求的 Invoke 方法中，通过反射的方式将这些参数读取出来就可以使用了。但是如果没有使用特性的模型类放入请求参数肯定是不行的，因此加入了一个接口 `IAttributeConstraint` 作为约束。所以，最终确定的调用方式是这样的：

```csharp
/// <summary>
/// 微服务请求方法
/// </summary>
/// <typeparam name="TRequestModel">请求参数模型类</typeparam>
/// <typeparam name="TResponseModel">响应参数模型类</typeparam>
/// <param name="requestModel">请求参数</param>
/// <returns>响应结果</returns>
TResponseModel Invoke<TRequestModel, TResponseModel>(TRequestModel requestModel) where TRequestModel : IAttributeConstraint;
```

而微服务的请求模型定义成这样：

```csharp
/// <summary>
/// XXX服务请求模型
/// </summary>
[InterfaceRequest(
    configKey: "XXXXUrl",
    interfaceName: "XXXX服务",
    defaultUrl: "http://xxx.microservice.test.com/xxx/xxx/",
    isLog: true，
    invokeMethod: InvokeMethodEnum.POST,
    invokeType: InvokeContentTypeEnum.JSON,
    timeOut: 5000,
    responseType: ResponseDataTypeEnum.JSON,
    encodingType: "utf-8"
)]
public class XXXXRequestModel : IAttributeConstraint
{
    /// <summary>
    /// 用户Id
    /// </summary>
    [JsonProperty(PropertyName = "userId")]
    public int UserId { get; set; } = 0;

    /// <summary>
    /// 用户名
    /// </summary>
    [JsonProperty(PropertyName = "userName")]
    public string UserName { get; set; } = string.Empty;

    //... ...
}
```

其中我直接使用 `Newtonsoft.Json` 类库中的 `JsonProperty` 特性来定义属性的序列化名称。加入这个特性是为了解决微服务和站点见命名规则不一致的问题。要知道，.Net 平台下类的属性的命名规则是首字母大写的，而 Java 微服务的命名规则，字段的首字母必须要小写，因此就存在了命名规则，所以加入这个属性来解决这个问题。

## 编写 IInvokeService

调用方式既然已经确定了，那就可以开始定义微服务调用服务 `IInvokeService` 了。其实按照上面的逻辑， `IInvokeService` 就包括一个 `Invoke` 方法就足够了，我可以在方法内解析参数，请求结果，记录日志，返回响应等，但是如果这样设计的话就又出现了另一个问题——没有扩展性。 `IInvokeService` 里肯定将会支持几个默认的请求方式，比如 GET 请求方式参数 Url 传递， POST 请求方式参数 Json 传递等，但是以后总会碰到一切奇奇怪怪的接口，比如 GET 需要 JSON 传递参数，POST 需要 Url 传递，返回响应不是一个模型是一个 string 等等，如果碰到 `IInvokeService` 不支持的接口，那这个方法就彻底不能使用了。因为这个原因，我把 `Invoke` 方法进行了拆分。
![IInvokeService](https://image.dunbreak.cn/past/IInvokeService.png)

我把 `IInvokeService` 中的请求方法拆分成了多个不同的服务：

- IRequestService：用于解析 RequestModel 中的参数提供微服务请求使用

- IServiceInvokeService：用于请求微服务，设计这个服务的目的是为了日后扩展，当前的逻辑中微服务都是 Http 协议的，所以内部依赖于 IWebRequestService，如果以后有可能存在 RPC 的微服务，那么直接扩展这个服务的实现即可

- IResponseService：用于解析请求的响应信息为想要的类型

- IInvokeLogService：用于记录请求的日志信息，当前版本的日志都是存在 DB 中进行分析的，有了这个接口，方便以后扩展文件等日志

- IWebRequestService：即 Http 请求的封装，其中 Get 请求使用 HttpClient 实现，Post 请求通过 WebRequest 实现

- IInjectedInvokeService：相当于对 IInvokeService 的一个代理，当调用 IInvokeService 的 InjectService 方法时，返回这个代理进行请求操作

拆分这么多方法一个是为了扩展性，另一个目的是为了服务于 `IInvokeService` 中的 `InjectService()` 方法：

```csharp
/// <summary>
/// 请求内部服务注入
/// </summary>
/// <param name="requestService">请求数据解析服务</param>
/// <param name="responseService">响应数据解析服务</param>
/// <param name="serviceInvokeService">接口请求服务</param>
/// <param name="invokeLogService">请求日志服务</param>
IInjectedInvokeService InjectServie(IRequestService requestService = null, IResponseService responseService = null, IServiceInvokeService serviceInvokeService = null, IInvokeLogService invokeLogService = null);
```

加入这个方法的目的就是为了解决对整个请求参数的可自定义功能，例如：如果当前 IInvokeService 中不支持解析返回值类型是 XML 格式的功能，那么就可以这样做：

```csharp
/// <summary>
/// 自定义响应服务
/// </summary>
public class CustomResponseServiceImpl : IResponseService
{
    /// <summary>
    /// 自定义解析响应数据
    /// </summary>
    /// <typeparam name="TResultModel">响应数据类型</typeparam>
    /// <param name="responseModel">请求结果模型</param>
    /// <returns>解析结果模型</returns>
    public TResultModel AnalysisResponseData<TResultModel>(ResponseDataModel responseModel)
    {
        //实现代码
    }
}

/// <summary>
/// 请求方法
/// </summary>
public void RequestFunction()
{
    // ...
    var responseModel = _invokeService.InjectService(responseService: new CustomResponseServiceImpl()).Invoke<TRequestModel, TResponseModel>(requestModel);
    // ...
}
```

这样就可以通过代理服务 `IInjectedInvokeService` 来自定义响应的解析获取微服务的请求结果。

至此整个接口请求服务就设计完毕了，但是其内部的实现还有很多逻辑，因为需要支持不同的接口请求方式，解析不同的响应结果等，总是服务的实现也是一个浩大的工程量，在此就不多说了。

## 遇到的问题

接口请求服务在使用的过程中遇到了两个很大的问题：

1. WebRequest 的 Get 请求方式在 Linux 环境下无法工作，获取不到请求结果

2. HttpClient 在 Linux 下存在很大的性能问题

对于第一个问题， `IWebRequestService` 在第一个版本中 Get 方式是使用 `WebRequest` 来实现了，本地测试通过之后传去 Linux 服务器发现获取不到结果，一直在超时。通过查找发现
![.Net Core Web请求](https://image.dunbreak.cn/past/webrequest-error-on-linux.png)

所以在 Linux 系统下请求 Http 只能通过 HttpClient 的方式实现，也由此引出了第二个问题。

这个问题很奇怪，因为如果是用 IP 的形式在 Linux 下通过 HttpClient 请求接口，是没有什么性能问题的，但是使用域名就会慢 `50-100倍` !这个问题在当时来说是非常严重和棘手的了，一个微服务请求本来只需要 10-100ms，但是在 Linux 下就需要 1000-5000ms，这个时长是无法接受的。但是使用 IP 没问题，所以当时解决这个问题的方向和思路就错了，一直以为是运维搞的 DNS 解析有问题，也因此耗费了大量的时间和经历与运维同事撕逼，但是最终都没有结果，因为他们在服务器使用 Curl 访问是很明显没有那么慢的。无解，最终就是用了临时方案：所有微服务都通过 IP 访问。

但是这种方式解决不了根本问题，因为随着业务的扩展，请求的接口也越来越多，内部也会调用大量其他集团的接口，这样我们就获取不到 IP 了。也是那个时候，我们逐渐发现，这其实不是服务器环境的问题而有可能是.Net Core 在 Linux 下的兼容问题。因此以这个思路开始查找解决方案，发现了这个 Issue：[HttpClient Performance Slow Compared to .NET 4.7](https://github.com/dotnet/corefx/issues/23401)，在这个 Issue 底下发现了一个哥们碰到了和我完全相同的问题：
![HttpClientIssue](https://image.dunbreak.cn/past/HttpClientIssue.png)

查了一下我们所有服务器的.Net Core Runtime 的版本：2.0.5。。。看来真的是微软留的一个坑而不是我们自身的问题，这么多天错怪运维同事了，现在想想都有点丢人。。。底下微软工程师的回复就是——升级.Net Core Runtime！最终升级.Net Core Runtime 为 2.0.7 解决了这个问题。。。其实升级的过程也很曲折，同样也就不多说了，因为升级了 2.0.7 之前删除了 2.0.5 版本的 runtime，所以支持的类库版本高于现有代码的类库版本，因此 Linux 服务器下的 lib 都无法找到了，为了解决这个问题，在解决方案文件中加入了如下属性：

```XML
  <PropertyGroup>
    <PublishWithAspNetCoreTargetManifest>false</PublishWithAspNetCoreTargetManifest>
  </PropertyGroup>
```

加入这个配置节点之后，发布的文件夹中就会自己包含所有依赖的类库，而不是从 lib 文件夹下去找，由此解决了这两个问题。
