##开始
在上一篇文章：[dotnet core多平台开发体验](http://www.cnblogs.com/caipeiyu/p/5537021.html) ，体验了一把dotnet core 之后，现在想对之前做的例子进行改造，想看看加上mvc框架是一种什么样的体验，于是我就要开始诞生今天的这篇文章来分享我的感受了。

##一、项目改造加入mvc框架

首先我们先修改project.json文件，加入mvc框架的依赖、添加发布时候的发布的包含内容、还有就是编译选项，修改前后的对比如下图。

![](http://images2015.cnblogs.com/blog/248834/201606/248834-20160605005316446-70105496.png)

第二我们对Startup.cs进行修改，以让能加载mvc框架进行工作。如下图，把我们原来输出到页面的内容给删除掉，修改成使用mvc并配置mvc的路由信息。同时还需要把mvc给加入到services里面。

![](http://images2015.cnblogs.com/blog/248834/201606/248834-20160605010529164-221246368.png)

第三我们要添加下 视图和控制器，创建Controllers和Views两个目录。并且添加HomeController控制器和它对应的视图。

![](http://images2015.cnblogs.com/blog/248834/201606/248834-20160605011919774-2108680145.png)

HomeController.cs的内容

    using Microsoft.AspNetCore.Mvc;

    namespace LearnDotnetCore.Controllers
    {
        public class HomeController : Controller
        {
            public IActionResult Index()
            {
                return View();
            }
        }
    }
    
Home/Index.cshtml的内容

    Hello from ASP.NET Core MVC!   
    
到这里改造加入的内容已经完成了，然后输入命令来下载依赖包和编译运行

>dotnet restore

>dotnet run

    Project LearnDotnetCore (.NETCoreApp,Version=v1.0) will be compiled because inputs were modified
    Compiling LearnDotnetCore for .NETCoreApp,Version=v1.0

    Compilation succeeded.
        0 Warning(s)
        0 Error(s)

    Time elapsed 00:00:04.0228759
    

    Hosting environment: Production
    Content root path: /Users/caipeiyu/Documents/code/c#/LearnDotnetCore
    Now listening on: http://localhost:5000
    Application started. Press Ctrl+C to shut down.
    
输入上面两个命令后，显示编译成功，并运行起来。 然后打开浏览器输入地址http://localhost:5000/ 浏览器显示内容 `Hello from ASP.NET Core MVC! `。到这里我们已经成功的给添加上mvc框架，并且很好的运行起来，但是界面好像挺单调，可以添加点其他的内容让显示充实的。添加mvc框架进来如此的简单，内容如此的少，文章到这里结束肯定不是我的风格了，篇幅不够放首页等下不小心就被移除了，这肯定是不能忍受的啦。下面可以来研究下这个mvc的简单运行原理然后分享一下，怎么说也得凑点字数。

##二、ASP.NET Core MVC运行原理初探

首先我们先了解一下什么是mvc模式，什么是mvc框架。其实像我们经常说的mvc很多讲的都是这么一个mvc框架，而不是讲的什么mvc模式。
需要明白的一点是 `MVC模式（Model–view–controller）是软件工程中的一种软件架构模式,而aps.net mvc是一种对mvc模式实现的一个框架。`

参考资料：https://zh.wikipedia.org/wiki/MVC

#### 自己实现简单的mvc
在一开始，假设我们没有 `asp.net mvc 框架` 那么我们自己想要实现一个mvc模式的话，那么我们能如何去实现呢？

首先我们来新建一个web项目，如果像前面那样用dotnet new建立一个项目然后再改造，感觉还是比较麻烦。这里有另一种建议，使用Yeoman 来新建立项目，参考：https://docs.asp.net/en/latest/client-side/yeoman.html?#building-projects-with-yeoman
安装细节这里不做翻译介绍，请自行研究。安装完这个东西后，我们可以新建立一个Empty Web Application项目。建立完成项目后，我们可以来建立一个mvc实现代码。

首先我们可以建立一个 Model

    public class MyModel
    {
        public string Name{get;set;}
    }

然后建立一个 View

    using Microsoft.AspNetCore.Http;

    public class MyView
    {
        private MyModel _model;

        public MyView(MyModel model){
            _model = model;
        }

        public async System.Threading.Tasks.Task Show(HttpContext context)
        {
            await context.Response.WriteAsync($"Hello {_model.Name}");
        }
    }

最后建立一个Controller

    public class MyController
    {
        public MyView Index(){
            MyModel model = new MyModel(){
                Name="I am Home!"
            };
            return new MyView(model);
        }

        public MyView About(){
            MyModel model = new MyModel(){
                Name="I am About"
            };
            return new MyView(model);
        }

    }

建立完成后，我们就可以在入口处调用我们Controller,入口为Startup类的方法Configure里面的app.Run修改代码如下

        public void Configure(IApplicationBuilder app)
        {
            MyController controller = new MyController();
            app.Run(async (context) =>
            {
                switch (context.Request.Path.Value)
                {
                    case "/about":
                        await controller.About().Show(context);
                        break;
                    default:
                        await controller.Index().Show(context);
                        break;
                }

            });
        }

再运行 dotnet run,启动成功后，在浏览器里面输入http://localhost:5000/ 就显示 Hello I am Home! 输入 http://localhost:5000/about  就显示 Hello I am About，如上，就是我们实现了最简单的一个mvc。看到这里，可能还是很多人一头雾水那asp.net mvc里面是怎么一回事呀。

#### aspnet mvc运行初步解析

想知道aspnet mvc的模型是如何工作起来的，最简单有效的办法就是去看它的源码，参考源码：https://github.com/aspnet/Mvc

程序的一个顺序如下：
1. 先添加所有mvc运行需要依赖的东西到servicecollection里面去。包括Controller、Views等。[MvcServiceCollectionExtensions.cs](https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNetCore.Mvc/MvcServiceCollectionExtensions.cs)
2. 添加路由支持，然后根据路由实现的中间件来驱动控制器的动作。[MvcApplicationBuilderExtensions.cs](https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNetCore.Mvc.Core/Builder/MvcApplicationBuilderExtensions.cs)

在添加路由支持的源码里面我们可以看到有一句代码

    var routes = new RouteBuilder(app)
    {
        DefaultHandler = app.ApplicationServices.GetRequiredService<MvcRouteHandler>(),
    };

从这个代码我们不难看到路由的驱动入口就是 [MvcRouteHandler](https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNetCore.Mvc.Core/Internal/MvcRouteHandler.cs),在这个类里面调用方法public Task RouteAsync(RouteContext context)，这个方法主要是根据路由提供的数据来选择是要执行哪个控制器和控制器里面的方法。具体的实现可以参考[ControllerActionInvoker.cs](https://github.com/aspnet/Mvc/blob/dev/src/Microsoft.AspNetCore.Mvc.Core/Internal/ControllerActionInvoker.cs)

整个mvc运行起来的大概原理也就是这样，感觉好像也没有特别的复杂。至于这个mvc框架里面的依赖是如何注入，如何在调用到时取出来就不属于本文的范围。然后对路由内部的实现有兴趣的可以参考源码：https://github.com/aspnet/Routing

由于每一个人对知识的掌握范围的不同，可能单纯阅读代码是无法明白是什么意思，那么可以把源码下载下来，然后在我说到的这几个方法里面加断点看看每一个变量是怎么样的，可能会更好的辅助理解。更仔细的内容我将会在今后的文章里面进行介绍。本文结束。

--------------------
由于本人水平有限，知识有限，文章难免会有错误，欢迎大家指正。如果有什么问题也欢迎大家回复交流。要是你觉得本文还可以，那么点击一下推荐。