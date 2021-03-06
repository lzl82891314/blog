---
title: .Net Core 入门（四）
url_name: dotnet-core-04
subtitle: 总结.Net Core 碰到的零碎问题
date: 2018-05-31 19:55:02
categories:
  - .Net Core
tags:
  - .Net Core
  - C#
  - 入门
  - 很久以前
---

.Net Core 的新项目已经在正式运行了，但是开发期间还是遇到了很多小问题，这些问题不会影响整个开发的流程和使用，但是会影响开发效率和使用体验。

<!-- more -->

## 问题一：Unicode 编码

这是项目刚开始开发的时候遇到的问题，页面数据展示都没任何问题，但是浏览器右键查看源代码时，html 标签中绑定的数据源都是 Unicode 乱码。
![Unicode编码问题](https://image.dunbreak.cn/past/unicode-encoding-error.png)

很显然这是一个配置项的问题，查询找到解决方案，在 `Startup` 配置的 `ConfigureServices` 服务配置方法中，注入一个编码服务即可：

```csharp
//解决Unicode编码问题服务
services.AddSingleton(HtmlEncoder.Create(UnicodeRanges.All));
```

## 问题二：Action 中 string 类型默认值问题

在 Asp.Net MVC 框架中，UI 端的请求参数可以通过 Action 的形参接收：

```csharp
public IActionResult Demo(string userName = "")
{
    return View();
}
```

在 Framework 版本中， `string` 类型的默认返回值是空字符串 `""` ，但是在.Net Core 中， `string` 类型的默认返回值变成了 `null` 。并且通过上面那种赋值默认参数的形式依然是 `null`，这个问题就导致了旧项目迁移的时候代码很多就会报错空引用，因为很多的 string 类型都是按照空字符串处理的。

这很显然也是一个配置问题，查到的解决方案是：注入 `MVC service` 的时候，需要在 MVC 的 `ModelMetadataDetailsProviders` 配置项集合中加入对 `IDisplayMetadataProvider` 的自定义支持，将默认的 `ConvertEmptyStringToNull` 属性赋值为 `false` 。即需要加入如下自定义配置：

```csharp
/// <summary>
/// 自定义MetadataProvider
/// 专门解决Action字符串类型返回值默认为Null为不是String.Empty的难问题
/// </summary>
public class CustomMetadataProvider : IMetadataDetailsProvider, IDisplayMetadataProvider
{
    /// <summary>
    /// 解决MVC的Action中的String参数默认返回NULL而不是string.Empty的问题
    /// </summary>
    /// <param name="context"></param>
    public void CreateDisplayMetadata(DisplayMetadataProviderContext context)
    {
        context.DisplayMetadata.ConvertEmptyStringToNull = false;
    }
}
```

然后在 MVC 的服务中加入如下 option：

```csharp
//注入MVC服务
services.AddMvc(options =>
{
    options.ModelMetadataDetailsProviders.Add(new CustomMetadataProvider());
});
```

这样就可以解决 string 类型的返回值问题了。

## 问题三：Redis 的 Sentinel 模式

这个问题没有解决掉，只是提供了一种临时处理方案。因公司 DBA 需求，所有的 Redis 服务器都要改直接连接方式为 `Sentinel` 模式，原来的连接方式废弃。然而，我们的项目中 Redis 的连接是直接使用.Net core MVC 提供的 `IDistributedCache` 接口实现的，但是这个服务在注入的时候就提供了一个 `Configuration` 的配置项，怎么使用这个配置项实现 `Sentinel` 模式呢？Google 也找不到任何解决方案。因此，这个问题当时只能想到两个解决方案：

1. 弃用 `IDistributedCache` 接口，而是直接使用 Redis 的第三方 API 如 `StackExchange.Redis` 来连接 Redis

2. 查看 `AddDistributedRedisCache` 服务的源码，从源码中了解 Redis 的连接过程，尝试找到解决方案

方案 1 中，由于出现这个需求的时候，业务代码早都已经完成了，对 Redis 的配置等都已经完成，调用端直接从 DI 容器中 Resolve 出 `IDistributedCache` 进行读写，因此这种解决方案成本很大。因此先尝试使用方案 2 来找找解决方案。

没办法，只能去查看 `AddDistributedRedisCache` 的源代码了。从源码中可以发现， `IDistributedCache` 连接 Redis 其实内部也使用到了 `StackExchange.Redis` 这个类库。然而很遗憾，从源代码中的连接过程来看，仅仅提供 `Configuration` 的配置项失无法支持 `Sentinel` 模式的。因此当前只想到一种临时的解决方案。

简单地了解了一下什么是 Redis 的 `Sentinel` 模式，发现其实就是改直接连接 Redis 服务器为首先连接 Redis 哨兵服务器，然后通过哨兵服务器查询到真正的 Redis 主从服务器进行连接。那这样就好办了，我先使用 `StackExchange.Redis` 的 `Sentinel` 模式连接到 Redis 的哨兵服务器，然后通过哨兵服务器获取到真正的 Redis 主从服务器的 IP 地址，然后通过 `IDistributedCache` 的 `Configuration` 属性进行直接连接就行了，相当于使用 `StackExchnage.Redis` 做了一个适配器。

这里代码很多，因此就全部贴了，只是说一下主要的流程：

1. 通过 `StackExchange.Redis` 连接到哨兵服务器

2. 连接成功后，获取哨兵后端的正式主从服务器 IP 端口

3. 排除无用的从属服务器 IP 和端口

4. 通过获取到的主从服务器 IP 端口，创建 `ConfigurationOptions` 对象

5. 通过 `ConfigurationOptions.ToString()` 赋值 `AddDistributedRedisCache` 下 `Option` 的 `Configuration` 属性

在这贴一下步骤 3 的无用从属服务器 IP 端口排重的代码：

```csharp
//通过哨兵服务器获取Redis主从服务器端口
var slaveRedisEndpointConfigs = sentinelServer.SentinelSlaves(sentinelConfig.ServiceName);
string ip, port, flags;
var slaveRedisEndpoints = from slaveEndpointConfig in slaveRedisEndpointConfigs
                            let endpointDic = slaveEndpointConfig.ToDictionary()
                            where endpointDic != null
                                && endpointDic.TryGetValue("ip", out ip)
                                && endpointDic.TryGetValue("port", out port)
                                && endpointDic.TryGetValue("flags", out flags)
                                && !flags.Contains("s_down")
                                && !flags.Contains("o_down")
                            let singleIp = endpointDic.GetValueOrDefault("ip", "")
                            let singlePort = endpointDic.GetValueOrDefault("port", "")
                            select singleIp == "127.0.0.1" ? (sentinelConfig.EndPoints.First() as IPEndPoint).Address.ToString() : $"{ singleIp }:{ singlePort }";
```

排重规则我是从网上找的，只是在这用 Linq 重写了一遍。

这样，就可以临时解决 `Sentinel` 模式的问题了。为什么说临时呢，因为我在这固定写死了哨兵服务器，其实真正的哨兵服务器是一个集群，在 Java 中，有专门的类库可以提供连接到所有的集群池中，而.Net 没有相似的类库，所以只能从哨兵服务器中获取一个来直接连接。这个问题也无法通过轮询的方式解决。因为 `AddDistributedRedisCache` 服务实在站点启动的时候就已经注册完毕的，只有在站点结束下一次启动才会重新注入，所以站点启动了就无法更改，要说每次连不通的哨兵服务器，这就要耗费很大的物力了，吃力不讨好。所以，目前的打算是在微软支持 `IDistributedCache` 使用 `Sentinel` 模式之前，先这样临时使用吧。
