---
title: Go, 从入门到忘记
url_name: go-startup
date: 2021-11-14 15:21:38
hidden: true
categories:
  - Go
tags:
  - 入门
  - 基础知识
---

差不多整整一年前，当时在云组做了一个[Docker 和 Kubernetes 的培训](https://www.dunbreak.cn/2020/12/31/docker-and-kubernetes/)，当时就发现，云原生相关的组件几乎都是用 Go 写的，本着努力成为一个云原生程序员（笑）的目标，当时夸下海口说未来半年的考评任务要学会 Go 去看 Kubernetes 源码，然后就没有然后了……

<!--more-->

Go 最开始吸引我的是它号称`互联网时代的C语言`，第一次听到这个宣传语时，我下意识的以为 Go 是一门和 C 语言一样性能强大并且且可以直接操作硬件的语言。好家伙，学了之后才发现这句话的意思是说 Go 和 C 一样原始……

![hello-go](https://image.dunbreak.cn/go/hello-go.jpg)

作为一个 C#er，第一次看到 Go 的语法真是难受的浑身不爽，先不说别的，就单看这个 main 函数我就能发现奇怪的宝藏：`printf` + `%s`这种语法我可能大学毕业就再也没有用过了，然而这种过时的语法既然还能在 Go 这种 2007 年才发布的语言中看到，并且这种写法还是唯一支持的写法……

加之想到之前由于没有学习 Go Module 相关的内容导致我一个 main 函数都跑不起来的尴尬过往，说实话一年前夸下海口之后我是打算放弃的。

直到今年下半年，Barry 说要我做一个 Go 语言的培训我才重新开始学习 Go（果然 Deadline 才是第一生产力）。不过还好，当把所有 Go 的语法都学完并且自己动手写了一个小项目之后，发现 Go 语言中独特的抽象设计完全是给我打开了一扇新的大门，并且一切极简自己动手的风格也完全契合我想自己动手写点东西提高自己的想法，果然人人都是王境泽。

## 还是从头说起

### Go 是怎样诞生的？

果然我就是一个学任何东西都喜欢刨根问底的人（看书必看序，学习必须从诞生开始…），所以最开始还是来看看 Go 到底是如何诞生的。

话说早在 2007 年 9 月的一天，Google 工程师 [Rob Pike](https://en.wikipedia.org/wiki/Rob_Pike) 和往常一样启动了一个 C++项目的构建，按照他之前的经验，这个构建应该需要持续 1 个小时左右。这时他就和 Google 公司的另外两个同事 [Ken Thompson](https://en.wikipedia.org/wiki/Ken_Thompson) 以及 [Robert Griesemer](https://en.wikipedia.org/wiki/Robert_Griesemer) 开始吐槽并且说出了自己想搞一个新语言的想法。当时 Google 内部主要使用 C++构建各种系统，但 C++复杂性巨大并且原生缺少对并发的支持，使得这三位大佬苦恼不已。

![authors](https://image.dunbreak.cn/go/authors.png)

第一天的闲聊初有成效，他们迅速构想了一门新语言：能够给程序员带来快乐，能够匹配未来的硬件发展趋势以及满足 Google 内部的大规模网络服务。并且在第二天，他们又碰头开始认真构思这门新语言。第二天会后，Robert Griesemer 发出了如下的一封邮件：

![plan-email](https://image.dunbreak.cn/go/plan-email.webp)

可以从邮件中看到，他们对这个新语言的期望是：**在 C 语言的基础上，修改一些错误，删除一些诟病的特性，增加一些缺失的功能。**比如修复 Switch 语句，加入 import 语句，增加垃圾回收，支持接口等。而这封邮件，也成了 Go 的第一版设计初稿。

在这之后的几天，Rob Pike 在一次开车回家的路上，为这门新语言想好了名字`Go`。在他心中，"Go"这个单词短小，容易输入并且可以很轻易地在其后组合其他字母，比如 Go 的工具链：goc 编译器、goa 汇编器、gol 连接器等，并且这个单词也正好符合他们对这门语言的设计初衷：简单。

> Go 从头到尾一直其实只有这一个名字，而现在人们知道的名字大多是 Golang，这其实是错的。Golang 其实是因为 Go 语言原本想要的官网域名 `https://www.go.org` 被占用了，无奈使用了 `https://www.golang.org`。此外，早起由于 Go 这个单词太普世了，导致搜索引擎很难搜索到 Go 语言相关的内容，所以 Golang 也有 SEO 的作用。但是不管这些，这门语言的名字就是 Go。

### 逐步成型

在统一了 Go 的设计思路之后，Go 语言就正式开启了语言的设计迭代和实现。

2008 年，C 语言之父，大佬肯·汤普森实现了第一版的 Go 编译器，这个版本的 Go 编译器还是使用 C 语言开发的，其主要的工作原理是将 Go 编译成 C，之后再把 C 编译成二进制文件。到 2008 年年中，Go 的第一版设计就基本结束了。这时，同样在谷歌工作的伊恩·泰勒（Ian Lance Taylor）为 Go 语言实现了一个 gcc 的前端，这也是 Go 语言的第二个编译器。

伊恩·泰勒的这一成果不仅仅是一种鼓励，也证明了 Go 这一新语言的可行性 。有了语言的第二个实现，对 Go 的语言规范和标准库的建立也是很重要的。随后，伊恩·泰勒以团队的第四位成员的身份正式加入 Go 语言开发团队，后面也成为了 Go 语言，以及其工具设计和实现的核心人物之一。

罗斯·考克斯（Russ Cox）是 Go 核心开发团队的第五位成员，也是在 2008 年加入的。进入团队后，罗斯·考克斯利用函数类型是“一等公民”，而且它也可以拥有自己的方法这个特性巧妙设计出了 http 包的 HandlerFunc 类型。这样，我们通过显式转型就可以让一个普通函数成为满足 http.Handler 接口的类型了。不仅如此，罗斯·考克斯还在当时设计的基础上提出了一些更泛化的想法，比如 io.Reader 和 io.Writer 接口，这就奠定了 Go 语言的 I/O 结构模型。后来，罗斯·考克斯成为 Go 核心技术团队的负责人，推动 Go 语言的持续演化。到这里，Go 语言最初的核心团队形成，Go 语言迈上了稳定演化的道路。

### 正式发布

2009 年 10 月 30 日，罗伯·派克在 Google Techtalk 上做了一次有关 Go 语言的演讲，这也是 Go 语言第一次公之于众。十天后，也就是 2009 年 11 月 10 日，谷歌官方宣布 Go 语言项目开源，之后这一天也被 Go 官方确定为 Go 语言的诞生日。

![published](https://image.dunbreak.cn/go/published.webp)

在 Go 开源后，它也有了一个属于自己的吉祥物 Gopher：

![Gopher](https://image.dunbreak.cn/go/gopher.webp)

Gopher 是一种地鼠，是 Rob Pike 的老婆设计的（写代码的人还有家人的支持，真是羡慕了笑），从此 Gopher 也成了世界各地 Go 程序员的象征。

### 运营至今

其实由于 Google 公司做背书，Go 在刚发布开始还是比较火热的。但是随时认识了解它的人越来越多，Go 的各种问题也逐步暴露了出来，比如：Go 最开始的设计只是为了解决 Google 公司自己内部的一些项目构建问题，其内部的系统是一个巨石架构，几乎不存在包依赖的问题，因此包依赖这个问题并没有在最开始成为 Go 开发团队的主要任务。

然而没有包管理的项目是恐怖的，在最开始的版本，Go 只支持 Go Env 的包管理模式，这种模式下，我们写的代码必须放在 Go 安装时设置的 Go Env 目录下。这几乎简直是一个反人类的设计，也是最开始我连一个 hello, world 都编译不成功的主要原因。

除此之外，由于没有包依赖管理，最开始 Go 项目的包是没有版本的概念的，也就导致了大多数的项目很难运行起来。随后，Go 团队引入了 vendor 模式，试图去解决这个问题，但是都只是杯水车薪。直到 Go1.11 的 Go Module 发布之后，Go 才彻底解决了包依赖的问题。

![main-version](https://image.dunbreak.cn/go/main-version.webp)

从上面的 Go 的几个主要版本可以看到，明年 2 月左右发布的 Go 1.18 版本将支持泛型，这也说明 Go 正在逐步成为一门更主流的语言。

## 学会打洞

Go 是一门致力于简单的语言，从最开始的 Hello, World 函数中我们可以大概知道如下几个概念：

1. 和 C#一样，块和块之前是由大括号区分的（不像 Python 那种面向游标卡尺编程的语言）
2. 和 C#不同，语句之后不用跟分号，如果加了，会自动被 Go 原生自带的代码格式化工具 gofmt 删掉 (go 从源头上根治了左括号到底换不换行这一世纪难题)
3. 同样和 C#不同，函数是一等公民，程序以包作为拆分载体，而不是 C#中以类和程序集。在 Go 中，文件可以大概等同于 C#的 Class；包可以大概等同于 C#的程序集。这个概念以我浅显的理解应该和 JavaScript 相同

好了，有了上述这些基础的概念，我们就可以正式学习如何当一名合格的地鼠了。不过，既然我们都已经是成熟的 C#er 了，那学习 Go 肯定不能从 int 是什么意思说起了，如何找一个切入点呢？

### 关键字

Go 的简单从关键字的个数上就可以很好的体现出来了。

Go 一共只有可怜的`25个关键字`，而我数了数 C#的，好家伙，基础关键字就有`77个`，除此之外还有`42个上下文关键字！`一共有 119 个关键字，这几乎是 Go 的 5 倍之多！

Go 语言中的 25 个关键字：

| break    | default     | func   | interface | select |
| -------- | ----------- | ------ | --------- | ------ |
| case     | defer       | go     | map       | struct |
| chan     | else        | goto   | package   | switch |
| const    | fallthrough | if     | range     | type   |
| continue | for         | import | return    | var    |

既然这样，那就以关键字为切入点，主要对比学习一下 Go 和 C#的关键字有哪些差别，进而来全面地学习一下 Go 的语法。为此，我主要把 Go 和 C#的关键字进行了分类，来按类别学习这些关键字的异同。

#### 咱俩都有的

- break
- case
- const
- continue
- default
- else
- for
- goto
- if
- interface
- return
- struct
- switch
- var

以上的 12 个关键字是 Go 和 C#共有的，它们之中大部分的用法都是完全相同的，这里主要说一下 Go 中有特殊语法的关键字。

##### var

首先，让我们来看看参数定义关键字 var。

var 的意思同样也是定义变量，但是定义变量的语法和 C#有所不同。C#中只有一种定义变量的方法，而 Go 中有两种，它们分别是：

- 普通方式

```go
var i int = 1
```

这种方式是 Go 的原始变量定义方式，一般包级别的变量都是这样定义的，并且如果定义那些编译器可以自动推断的类型，比如上述的例子，其后的类型可以省略。

由此可知，Go 中 var 就是申明一个变量，而这个变量是什么类型，需要在变量名之后指明。

- 语法糖(是的，Go 中也有语法糖…)

```go
i, j := 1, "hello"
```

上述的代码可以简写为这种语法糖形式，Go 代码中，90%的变量都是以这样的方式定义的，因为 Go 中几乎所有的函数都会存在多余一个的返回值，这样的定义可以省去很大功夫。

##### switch-case-default

switch-case 是一个连用的子句，但是 case 和 default 这两个关键字在 Go 中除了可以和 switch 连用，还可以和后面会讲到的 select 语句连用。

Go 中默认把 switch 语句的一个弊端修复了，即 switch 子句中不用再写 break 了：

```go
switch n := "a"; n {
  case n == "a":
    fmt.Println("a")
  case n == "b":
    fmt.Println("b")
  case n == "c":
    fmt.Println("c")
}
```

> 上面这段代码的 fmt 是标准输出包，其中的`Println`函数等同于 C#中的`Console.WriteLine`方法。

这段代码的最终结果只会输出`a`，而 C#中这样的代码会把 abc 全部输出出来，这样的改动确实比较省心。

除此之外，switch 语句后面出现了一个比较奇怪的写法`n := "a"; n`，在 Go 中的控制语句（if, else if, switch-case, for）都可以加入这样的控制块作用域代码，如此这样写的话，这个变量 n 就只能在 switch 语句中生效。分号之前是变量的定义，分号之后是定义的判断条件。

这种语法优点类似于 C#中的普通 for 循环的前两个子句。

最后，我们可以看到 switch 之后没有跟小括号，在 Go 中，控制块的子句后面都是不需要写小括号的，如果写了同样会被 gofmt 自动格式化掉。

##### for

Go 中的循环控制语句`有且只有`一个 for 关键字。而 C#中的 while、foreach 等在 Go 中都是通过 for 的各种变形达成的。

- while 语句

```go
for true {
  // ...
}
```

- for 语句

```go
for i := 0; i < n; i++ {
  // ...
}
```

Go 中普通的 for 循环和 C#几乎没有差别，唯一的差别就是 `i++从表达式变成了语句`。

这句话的意思是，Go 的代码中不能写`i = i++`这样的代码。

此外，也没有`++i`这样的语法，只有`i++`

- foreach 语句

```go
array := [5]int{1, 2, 3, 4, 5}
for index, value := range array {
  // ...
}
```

> 这个例子中用到了数组的初始化代码，数组会在后面单独说明，这里只是作为一个例子展示。

foreach 语句的写法和 C#中很不相同，上述的例子是 foreach 遍历一个 int 类型的数组，其中用到了一个`range`关键字，这个关键字会把数组拆分成两个迭代子对象 index 和 value，这个语法同样类似于 JavaScript 的循环语法。

##### struct

Go 中的 struct 关键字和 C#中的作用是相同的，即定义一个结构体。

上面我们提到了，Go 中是没有类这个概念的，那唯一承载 C#中 class 作用的定义就是 struct 了。和 C#相同，struct 在 Go 中同样也是值类型变量，因此使用的时候一定需要注意案值传递导致的复制问题。

此外 Go 中的 struct 内只能定义字段，不能定义函数，这些概念会在后面的面向对象小节中主要说明。

##### 其他

1. return 关键字和 C#功能是相同的，这里拿出来是要说一下，Go 中的 return 同样也可以同时返回多个结果，Java 没有这样的语法代码写起来实在是费劲。
2. interface 没讲，这个关键字和 C#相同都是定义接口的，而 Go 中的接口实现方式和 C#完全不同，需要在后面详细说明，所以这里就简单带过。

其余没有说明的关键字用法和 C#完全相同，就不一一说明了。

#### 虽然不太一样但是意思差不多的

- package
- import
- type
- defer

##### package 和 namespace

上面已经提到了，Go 的函数是一等公民，因此，Go 比 C#少了一层类的封装，相对的，对函数的封装就体现在了包里。

Go 中的 package 就是定义组织一个包的，其目的和 C#的 namespace 基本是相同的，主要是对代码模块进行隔离。但是有一点和 C#不同，C#十分灵活，即是不在一个文件夹下的代码都可以定义为相同的 namespace。但是 Go 不行，Go 中 package 内的文件，都需要在相同的文件夹内才能被正确编译，并且一个文件夹内只能出现最多一个包名。

这里再额外说一下 main 包，类似于 C#中的 Main 方法，Go 中可运行程序的执行入口也是一个 main 函数，并且如果想要让程序顺利执行，main 函数必须定义在`package main`下。

##### import 和 using

这俩用法也基本是相同的，都是用来导入其他模块的代码来使用的。和 C#using 的是 namespace 相同，Go 中 import 的同样也是其他包的名字。

这里要说一个 Go 的强制要求：没有在代码模块中使用的 import 或者定义了但是没有使用的变量等，在 Go 编译时会直接报错。

这样做除了使代码看起来更简洁以外，最主要的原因是 import 语句除了引用其他包还有另一个功能就是调用包中的`init()`函数。比如如下代码：

```go
package demo

var me string

func init() {
	me = "jeffery"
}

func SayHello() {
	fmt.Printf("hello, %s", me)
}
```

> 这段代码中，定义了一个包级变量`me`，这种变量可以类比于 C#中的静态变量。

上述的程序定义了一个 demo 包，当 demo 包第一次被 import 关键字加载到其他包时，会自动调用其`init()`函数，这就优点类似于 C#中类的构造方法，这时就会把`变量me`赋值为`jeffery`，之后调用`SayHello()`函数时，返回的就都是"hello, jeffery"了。

也正是因为`init`函数的存在，不使用的 import 需要被删除，因为很有可能会自动调用到对应包内的 init 函数。

##### type 和 class

- 常规用法

把 type 和 class 对比其实是不太合理的。因为 C#中 class 关键字是定义一个类型并且定义这个类型的具体实现，比如下述的代码在 C#中的意思是定义一个名为 People 的类，并且定义了这类中有一个属性 Age。

```csharp
class interface IAnimal {
  public void Move();
}

class People {
  public int Age { get;set; }
}
```

然而 Go 中的 type 关键字仅仅是用来定义类型名称的，如果想要定义其实现，必须后面再更上具体实现的关键字。比如上述的代码定义在 Go 中就变成了如下：

```go
type IAnimal interface {
  Move()
}

type People struct {
  Age int
}
```

上述只是 type 的最常用用法，除此之外 type 还有两个其他的用法：

- 以一个基准类型定义一个新类型

```go
type Human People
```

这样的语句相当于用`People`类型定义了一个`Human`的新类型。**注意，这里是一个新类型，而不是 C#中的继承**。因此如果 People 内有一个 Move 函数，那 Human 对象是无法调用这个 Move 函数的，如果非要使用，则需要强制类型转换。

> Go 中的强制类型转换是类型 + ()，比如上述的例子 Human(People)就可以把 People 类型强转为 Human 类型。

- 定义类型别名

```go
type Human = People
```

如果使用了`等号`进行定义，那就相当于给类型 People 定义了一个别名 Human，这种情况下 People 中的代码 Human 也是可以正常使用的。

上面两种用法基本都不常用，这里只做了解即可。

##### defer 和 finally

Go 中的 defer 作用就是 C#的 finally，在一个方法执行结束退出之前，可以干一件事。

而和 C#不太一样的是，Go 中的 defer 语句不用必须写在最后，比如我们会经常看到这样风格的代码：

```go
var mutex sync.Mutex

func do() {
  mutex.Lock()
  defer mutex.Unlock()
  // ...
}
```

上面这个例子的意思是定义一个全局锁，在 do 函数进入时，加锁，在退出时解锁。之后再去写自己的业务逻辑。

除此之外，defer 也可以写多个，但最终的执行顺序是从下向上执行，也就是最后定义的 defer 先执行。

#### Go 有而 C#没有的

- chan
- go
- fallthrough
- func
- map
- range
- select

##### go 和 chan

这两个关键字是 Go 并发编程定义的关键字，具体的这部分会在后面的并发编程小节重点讲解。

##### fallthrough

这个关键字估计连职业 Go 开发工程师一辈子也用不了一次的那种。这个关键字完全是为了兼容 C 语言中的 fallthrough，其目的是是在 switch-case 语句中再向下跳一个 case，比如：

```go
switch n := "a"; n {
  case n == "a":
    fmt.Println("a")
    fallthrough
  case n == "b":
    fmt.Println("b")
  case n == "c":
    fmt.Println("c")
}
```

那这个例子的最终输出结果就是：

```bash
a
b
```

##### func

和其他函数是一等公民的语言（比如 JavaScript 的 function，Python 中的 def）一样，Go 中的 func 就是用来定义函数的。

此外，Go 中的 func 同样也可以配合 type 使用定义 C#中的委托，比如我们可以在 C#中定义一个.Net Core 的中间件：

```csharp
public delegate void handleFunc(HttpContext httpContext);
public delegate handleFunc middleware(handleFunc next);
```

这样的代码可以在 Go 中这样实现：

```go
type handleFunc func(httpContext HttpContext)
type middleware func(next handleFunc) handleFunc
```

##### map

map 关键字是用来定义 map 类型对象的。是的，Go 中原生了哈希对象，这个 map 创建出来的对象就类似于 C#中的`Dictionary<TKey, TValue>`对象。

这个 map 对象会在后面的 Go 中的特殊语法小节中专门讲解。

##### range

range 关键字在之前说 for 的时候就已经演示过功能了，就是用来遍历集合的。

##### select

select 关键字是一个并发编程的概念，会在后面讲并发编程的小节中主要讲解。

至此，我已经把 Go 中的 25 个关键字简单的一一概括了一下。到此为止，我们虽然还不能写出优雅的 Go 代码，但是至少我们应该可以看懂 Go 语言的一些语法概念了。
