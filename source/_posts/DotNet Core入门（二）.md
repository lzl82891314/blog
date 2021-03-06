---
title: .Net Core 入门（二）
url_name: dotnet-core-02
subtitle: 对 ORM 框架 Dapper 的封装
date: 2018-05-29 19:52:51
categories:
  - .Net Core
tags:
  - .Net Core
  - C#
  - 入门
  - 很久以前
---

在新的.Net Core 站点搭建时，肯定是需要一个 ORM 框架来操作数据库的。由于需要旧项目的直接迁移，因此新的.Net Core 站点依旧选择了 StackExchange.Dapper 作为轻量级的 ORM 框架。

<!-- more -->

然而旧项目中，对 Dapper 的操作都是直接使用的，无论增删改查都需要自行写 Sql 自行创建 Dto，而且最主要的，并没有统一管理数据库连接，每次使用都是需要在 `using` 语句下 new 一个新的 `IDbConnection` 来操作数据库。这种写法很不优雅，并且没有可控性，由于.Net Core 项目内置了 DI 容器，加入了 `Service` 的概念，因此自然而然我就想自己封装一遍 Dapper 使其服务化提供项目使用。

## 第一版——直接封装为 IQueryService

在项目开始做的时候，我并没有想太多东西，其实目的就是为了封装一遍 Dapper 不用每次都写 `using` 语句，所以直接封装了一个 `IQueryService` 封装 Dapper。
![第一版IQueryService](https://image.dunbreak.cn/past/QueryService_Edition1.png)

虽然这么叫但其实它就是提供 Dapper 封装的，里面有增删改查多少操作，方法有很多，随便列举几个：

```csharp
/// <summary>
/// 配置数据库连接串
/// </summary>
/// <param name="connNode">数据库节点</param>
void ConfigConnNode(string connNode);

/// <summary>
/// 其他数据库执行方法（如删除操作）
/// </summary>
/// <param name="sql">执行Sql</param>
/// <param name="param">参数</param>
/// <param name="transaction">事务</param>
/// <param name="timeOut">超时时间</param>
/// <param name="commandType">执行Sql类型</param>
/// <returns>操作结果</returns>
int Execute(string sql, object param = null, IDbTransaction transaction = null, int? timeOut = null, CommandType? commandType = null);

/// <summary>
/// 其他数据库执行方法（如删除操作）——异步
/// </summary>
/// <param name="sql">执行Sql</param>
/// <param name="param">参数</param>
/// <param name="transaction">事务</param>
/// <param name="timeOut">超时时间</param>
/// <param name="commandType">执行Sql类型</param>
/// <returns>操作结果</returns>
Task<int> ExecuteAsync(string sql, object param = null, IDbTransaction transaction = null, int? timeOut = null, CommandType? commandType = null);

/// <summary>
/// 非泛型查询方法
/// </summary>
/// <param name="sql">查询Sql</param>
/// <param name="param">参数</param>
/// <param name="transaction">事务</param>
/// <param name="timeOut">超时时间</param>
/// <param name="commandType">执行Sql类型</param>
/// <returns>查询结果集合</returns>
IEnumerable<dynamic> Query(string sql, object param = null, IDbTransaction transaction = null, int? timeOut = null, CommandType? commandType = null);

/// <summary>
/// 非泛型查询方法——异步
/// </summary>
/// <param name="sql">查询Sql</param>
/// <param name="param">参数</param>
/// <param name="transaction">事务</param>
/// <param name="timeOut">超时时间</param>
/// <param name="commandType">执行Sql类型</param>
/// <returns>查询结果集合</returns>
Task<IEnumerable<dynamic>> QueryAsync(string sql, object param = null, IDbTransaction transaction = null, int? timeOut = null, CommandType? commandType = null);
```

当时我的想法就是，需要连接数据库时，直接从 DI 容器中 Resolve 出一个 IQueryService，然后使用其方法直接加入 Sql 语句和参数就可以了，省去了创建连接串、using 语句等操作。所以 `IQueryService` 的实现大概就是这样的：

```csharp
using Dapper;

namespace QueryServiceImpl
{
    /// <summary>
    /// 其他数据库执行方法（如删除操作）
    /// </summary>
    /// <param name="sql">执行Sql</param>
    /// <param name="param">参数</param>
    /// <param name="transaction">事务</param>
    /// <param name="timeOut">超时时间</param>
    /// <param name="commandType">执行Sql类型</param>
    /// <returns>操作结果</returns>
    public int Execute(string sql, object param = null, IDbTransaction transaction = null, int? timeOut = null, CommandType? commandType = null)
    {
        //IDbConnection对象直接写死为new SqlConnection
        //_connStr是通过ConfigConnNode()方法获取的
        using (var dbConnection = new SqlConnection(_connStr))
        {
            return dbConnection.Execute(
                        sql: sql,
                        param: param,
                        transaction: transaction,
                        commandTimeout: timeOut,
                        commandType: commandType
                    );
        }
    }
}
```

但是这样设计有一个很致命的错误，就是数据库连接对象 `IDbConnection` 被写死为 `SqlConnection` ，耦合在了 `IQueryService` 的实现中，这就导致了无法切换数据库。新项目的数据库全部要求 MySql，而老的迁移逻辑依然有一部分需要 SqlServer，所以这样写肯定是没法用的，因此有了第二次封装。

## 第二次封装——封装 IDbConnection

所以现在问题的核心就是封装 `IDbConnection` 了，只要把它能通过参数生成等方式生成出来就可以解决这个问题了。但是怎么做呢？我想到了当初初学 `ENode` 的时候作者对 Dapper 的一个封装。第一次看他的代码的时候因为工作经验少觉得好高大上，所以对其印象深刻。
![ENode事件入库](https://image.dunbreak.cn/past/ENode_Dapper01.png)
![ENode数据库操作封装](https://image.dunbreak.cn/past/ENode_Dapper02.png)

可以看到作者使用委托的方式进行数据库事件处理，而 `IDbConnection` 则是通过方法来获取的。作者这么做是为了满足 DDD 中的事件驱动，但是我们的项目不需要这么复杂，所以不会这么写。但是还是有可以借鉴的地方，那就是 `委托` 。既然 ENode 的作者用委托封装查询逻辑，那我是不是可以反过来用委托封装 `IDbConnection` 的生成呢？所以以这个思路，我把 `IDbConnection` 对象的生成改成了委托，在 `IQueryService` 中删除了 `ConfigConnNode()` 方法加入了一个属性 `ConnStrCreator` ：

```csharp
/// <summary>
/// 数据库连接串创建委托
/// </summary>
Func<string, IDbConnection> ConnStrCreator { get; set; }
```

因此上述的 `Execute` 方法实现就可以写成这样了：

```csharp
using Dapper;

namespace QueryServiceImpl
{
    /// <summary>
    /// 其他数据库执行方法（如删除操作）
    /// </summary>
    /// <param name="sql">执行Sql</param>
    /// <param name="param">参数</param>
    /// <param name="transaction">事务</param>
    /// <param name="timeOut">超时时间</param>
    /// <param name="commandType">执行Sql类型</param>
    /// <returns>操作结果</returns>
    public int Execute(string sql, object param = null, IDbTransaction transaction = null, int? timeOut = null, CommandType? commandType = null)
    {
        //new SqlConnection()使用ConnStrCreator委托替换
        using (var dbConnection = ConnStrCreator(_connStr))
        {
            return dbConnection.Execute(
                        sql: sql,
                        param: param,
                        transaction: transaction,
                        commandTimeout: timeOut,
                        commandType: commandType
                    );
        }
    }
}
```

这样写确实舒服了很多，解决了 `IDbConnection` 创建和 `IQueryService` 耦合的问题，但是又引出了另一个问题。 委托 `ConnStrCreator` 从哪来？如果按照现在的设计来使用，就必须在使用前加入对 `ConnStrCreator` 的赋值，整个数据库操作基本是这样的：

```csharp
public class DemoClass
{
    /// <summary>
    /// 数据库操作服务
    /// </summary>
    private IQueryService _queryService;

    public DemoClass(IQueryService queryService)
    {
        _queryService = queryService;
    }

    public void Query()
    {
        string connStr = "XXXX";
        string sql = "XXX";
        //操作SqlServer
        _queryService.ConnStrCreator = new SqlConnection(connStr);
        _queryService.Excute(sql);

        //... ...

        //操作MySql
        _queryService.ConnStrCreator = new MySqlConnection(connStr);
        _queryService.Excute(sql);

        // ... ...
    }
}
```

可以看到为了解决 `using` 的问题结果又多写了对 `ConnStrCreator` 的配置，并且数据库连接串还需要自己读取，可以说是完全没有改观，一点都不优雅。如何解决即能创建 SqlServer 又能创建 MySql 的问题呢？自然就想到了工厂方法。

加入一个服务 `IQueryFactory` ，用来获取查询服务，内部有两个方法一个属性：

```csharp
/// <summary>
/// 数据库连接串
/// </summary>
string ConnStr { get; protected set; }

/// <summary>
/// 获取数据库服务
/// </summary>
/// <param name="connNode">数据库连接串节点</param>
/// <returns>查询服务</returns>
IQueryService GetQueryService(string connNode);

/// <summary>
/// 创建数据库连接对象
/// </summary>
/// <returns>数据库连接对象</returns>
IDbConnection CreateDbConnection();
```

继而将 `IQueryService` 的 `IServiceProvider` 删除加在 `IQueryFactory` 上。这样修改之后，创建数据库连接对象 `IQueryService` 的任务就交到了 `IQueryFactory` 上。
![第二版IQueryService](https://image.dunbreak.cn/past/QueryService_Edition2.png)

这样看起来是很完美了，创建 `IDbConnection` 的工作交给工厂来做，查询的服务使用 `IQueryService` 。但是在写实现的时候发现，把工厂作为服务，那注入 DI 容器的实现要用谁啊……所以这样设计还是存在问题的，并且这也只是一个简单工厂，并没有完整使用到工厂模式。

但是这样其实已经可以用了，因为旧项目大部分是用 SqlServer 做的，所以直接注入 SqlServer 工厂就可以了，如果要用到 MySql 那手动 new 一个应该也是可以的。然而，在使用的时候，发现另一个很严重的问题。如果按照下面这种方式写代码：

```csharp
public void Query()
{
    //connNode1连接的是数据库A
    string connNode1 = "AAAAA";
    string querySql1 = "XXXX";
    var queryService = _queryFactory.GetQueryService(connNode1);
    var queryResult1 = queryService.Query<object>(querySql1);

    //queryResult2调用InnerQuery()方法获取
    var queryResult2 = InnerQuery();

    string querySql3 = "XXXX";
    //由于在InnerQuery()方法中又创建了一个新的IQueryFactory
    //因此此时的queryService的连接节点其实已经被改成了connNode1
    //所以此时查询直接报错表不存在。。。
    var queryResult3 = queryService.Query<object>(querySql3);
}

public object InnerQuery()
{
    //connNode1连接的是数据库B
    string connNode2 = "BBBBB";
    string querySql2 = "XXXX";
    var queryService = _queryFactory.GetQueryService(connStr2);
    return queryService.Query<object>(querySql2);
}
```

会出现 `queryResult3` 报出 `数据库表不存在` 的异常。调试发现，在 `InnerQuery()` 方法中， `ConnStr` 已经被改成了 B 数据库，方法执行完毕之后，外层的 `queryService` 的连接串竟然被改成了 B 数据库，这样就导致了表不存在的问题……

这个问题的产生完全是我个人能力的问题， `IQueryFactory` 是被我用 `Singleton` 的模式注入到 DI 容器中的，因此，每一次对 `queryFacotry` 属性的修改都会被替换，没有作用域的区分。导致这个问题完全是我想当然了，没有去查 `Singleton`，`Scoped` 和 `Transient` 的区别就直接埋头胡写，所以出现了这么一个 bug。当然，这个问题的本质不是使用 `Singleton` 注入 DI 容器，而是我属性关系搞错了， `ConnStr` 这种属性，就不应该被工厂聚合，工厂的作用就是创建 `IQueryService` ，而 `ConnStr` 是查询连接串，所以这个属性就应该是 `IQueryService` 的。所以还是能力问题。也因为上述两个问题，出现了第三次改版。

## 第三次封装——加入 IFactoryService

为了解决上述两个问题，我加入了 `IFactoryService` 服务用来创建数据库连接对象工厂，内部就一个方法：

```csharp
/// <summary>
/// 创建数据库连接对象工厂
/// </summary>
/// <param name="factoryTypeEnum">工厂类型</param>
/// <returns>数据库对象工厂</returns>
IQueryFactory GetQueryFactory(FactoryTypeEnum factoryTypeEnum);
```

然后将其注入到 DI 容器中，并且将 `IQueryFactory` 中的 `ConnStr` 数据下放到 `IQueryService` 中。并且又由于此时公司运维数据库权限收回的需求，在 `IQueryFactory` 中加入 `DbConnStrCreator` 委托（这个问题不是重点）。因此，整个封装变成了这样：
![第三版IQueryService](https://image.dunbreak.cn/past/QueryService_Edition3.png)

## 其他封装

其实至此这个数据库操作服务已经可以用了，但是原生的 Dapper 使用起来很麻烦，增删改查都需要自己写 Sql 语句，如果有地方写错了或者数据库字段更改了都不好修改。所以我继续封装了一遍 Dapper 的使用，使其可以通过对象来简化 Sql。为此定义了一个空的接口 `IQueryServiceConstraint` 用来约束想使用对象操作的模型类，并且为这个接口写了很多扩展方法，以反射的方法获取对象中的属性来做到拼接 `增删改` Sql 的效果。例如一个获取 `Insert Sql` 的方法可以写成这样：

```csharp
/// <summary>
/// 获取该模型类的插入Sql
/// </summary>
/// <param name="serviceConstraint">数据库操作服务约束接口</param>
/// <returns>数据库插入Sql，如果查询失败返回NULL</returns>
public static string GetInsertSql(this IQueryServiceConstraint serviceConstraint)
{
    var tableName = serviceConstraint.GetTableName();
    if (!string.IsNullOrWhiteSpace(tableName))
    {
        var type = serviceConstraint.GetType();
        var propList = type.GetProperties();
        List<string> insertFieldList = new List<string>();
        List<string> insertParamList = new List<string>();

        foreach (PropertyInfo prop in propList)
        {
            var ignoreAttr = prop.GetCustomAttribute<IgnoreAttribute>(true);
            if (ignoreAttr != null && ignoreAttr.IsInsertIgnore)
            {
                continue;
            }

            string field = prop.Name;
            string param = $"@{ prop.Name }";
            var mappingNameAttrs = prop.GetCustomAttributes<MappingNameAttribute>(true);
            if (mappingNameAttrs != null && mappingNameAttrs.Count() > 0)
            {
                var mappingNameAttr = mappingNameAttrs.First(item => item.OperationType == SqlOperationTypeEnum.INSERT);
                if (mappingNameAttr != null && !string.IsNullOrWhiteSpace(mappingNameAttr.MappingName))
                {
                    field = mappingNameAttr.MappingName;
                }
            }

            insertFieldList.Add(field);
            insertParamList.Add(param);
        }

        if (insertParamList.Count > 0 && insertFieldList.Count > 0)
        {
            string insertField = string.Join(",", insertFieldList);
            string insertParam = string.Join(",", insertParamList);
            return $"INSERT INTO { tableName }({ insertField }) VALUES({ insertParam })";
        }
    }
    return null;
}
```

因为这些方法，插入数据库对象的时候就不必一个一个写 Insert 语句这么麻烦了，通过泛型直接传递查询对象类型就可以了。这其实就是偷懒的修改，并没有什么好说的，在此就不长篇大论了。

至此，整个数据库查询服务封装完毕。

## 未来改进方向

虽然当前版本已经可以进行开发了，但是依然存在问题。

1. 服务中不存在数据库对象连接池，每次都是创建新的对象，这样在高访问量的时候对性能损耗太大

2. 对对象的操作只能支持 `增删改` ，对 `Query` 没办法做到 `Lambda表达式` 级别操作

以上两个问题还需要我不断学习才能改善。对数据库连接池的概念，.Net 中我很少看到，Java 中却有很多，现在不清楚是不是平台的差别导致的这个问题以后会着重研究。对 `Lambda表达式` 的支持就需要研究 `Expression` 与对象的联动了，这一点我还没怎么看过，但是之前机缘巧合做过一个其他集团的项目，但是的 ORM 就支持这样的操作，有机会去学习一下他们的源码实现。
