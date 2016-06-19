##开始

回顾上一篇文章：[dotnet core开发体验之开始MVC](http://www.cnblogs.com/caipeiyu/p/5575158.html) 里面体验了一把mvc,然后我们知道了aspnet mvc是靠Routing来驱动起来的，所以感觉需要研究一下Routing是什么鬼。

##Routing简单使用体验

首先我们用命令`yo aspnet`创建一个新的空web项目。（Yeoman的使用自己研究，参考：https://docs.asp.net/en/latest/client-side/yeoman.html?#building-projects-with-yeoman）

创建完项目后，在project.json里面添加Routing依赖。

    "dependencies": {
        ...
        "Microsoft.AspNetCore.Routing": "1.0.0-*"
    },

添加完依赖后，修改Startup里面的Configure,和ConfigureServices里面添加Routing的使用依赖
修改前：

    public void Configure(IApplicationBuilder app)
    {
            app.Run(async (context) =>
            {
                await context.Response.WriteAsync("Hello World!");
            });
    }

修改后：

    public void ConfigureServices(IServiceCollection services)
    {
            services.AddRouting();
    }

    public void Configure(IApplicationBuilder app)
    {
            var endpoint = new RouteHandler((c) => c.Response.WriteAsync("Hello, I am Routing!"));

            app.UseRouter(endpoint);
    }

dotnet run 然后浏览器访问http://localhost:5000/ 显示为Hello, I am Routing! 接下来我们在http://localhost:5000/ 后面加入一些其他的东西来访问，发现其实还是一样打印Hello, I am Routing!  这让我们感觉好像并没有什么卵用的样子。不着急我们先来看看Routing是怎么运行起来的。在开始这话题之前需要先了解到什么是中间件，参考：https://docs.asp.net/en/latest/fundamentals/middleware.html  

##Routing的最基本运行原理

Routing的驱动入口就是基于middleware的。可以先看看 `app.UseRouter(endpoint)`的内部实现，参考一个扩展方法类[RoutingBuilderExtensions](https://github.com/aspnet/Routing/blob/dev/src/Microsoft.AspNetCore.Routing/BuilderExtensions.cs)，可以看到最后有一句代码`return builder.UseMiddleware<RouterMiddleware>(router)` 。这里可以很明显看到，入口在[RouterMiddleware](https://github.com/aspnet/Routing/blob/dev/src/Microsoft.AspNetCore.Routing/RouterMiddleware.cs)的Invoke方法。

        public async Task Invoke(HttpContext httpContext)
        {
            var context = new RouteContext(httpContext);
            context.RouteData.Routers.Add(_router);

            await _router.RouteAsync(context);

            if (context.Handler == null)
            {
                _logger.RequestDidNotMatchRoutes();
                await _next.Invoke(httpContext);
            }
            else
            {
                httpContext.Features[typeof(IRoutingFeature)] = new RoutingFeature()
                {
                    RouteData = context.RouteData,
                };

                await context.Handler(context.HttpContext);
            }
        }


这个入口的实现是这样的:
> Invoke入口 ===>>> 实例化一个 [RouteContext](https://github.com/aspnet/Routing/blob/dev/src/Microsoft.AspNetCore.Routing.Abstractions/RouteContext.cs) ===>>> 把我们传进来的 [IRouter](https://github.com/aspnet/Routing/blob/dev/src/Microsoft.AspNetCore.Routing.Abstractions/IRouter.cs) 存到 RouteContext里面的 [RouteData](https://github.com/aspnet/Routing/blob/dev/src/Microsoft.AspNetCore.Routing.Abstractions/RouteData.cs)  ===>>> 再执行IRouter的RouteAsync方法把RouteContext对象传进去提供给具体实现使用。===>>> 如果context.Handler没有东西，就执行下一个RequestDelegate，如果有的话就把RouteData保存起来然后执行这个RequestDelegate。


看到了这里我们已经可以明白下面这个代码运行起来的原理。

    public void Configure(IApplicationBuilder app)
    {
            var endpoint = new RouteHandler((c) => c.Response.WriteAsync("Hello, I am Routing!"));

            app.UseRouter(endpoint);
    }

可能还会有人不明白[RouteHandler]()是个什么鬼，既然我们知道了代码的实现运行原理，那么肯定可以猜到RouteHandler是有实现接口IRouter的。我们可以[看RouteHandler代码](https://github.com/aspnet/Routing/blob/dev/src/Microsoft.AspNetCore.Routing/RouteHandler.cs)

        public Task RouteAsync(RouteContext context)
        {
            context.Handler = _requestDelegate;
            return TaskCache.CompletedTask;
        }

这里可以看到把我们的`(c) => c.Response.WriteAsync("Hello, I am Routing!")`赋值给context.Handler，然后由RouterMiddleware来执行我们这个事件方法。于是我们就可以在浏览器上面看到输出 Hello, I am Routing!这么一句话了。

##Routing的一些使用

文章到现在，我们虽然知道了Routing运行起来的一个大概原理，但是我们一直打印出相同内容，确实也没有什么卵用呀。我们要改一下让打印内容能有点改变。这个时候可以使用到Routing提供的[Route](https://github.com/aspnet/Routing/blob/dev/src/Microsoft.AspNetCore.Routing/Route.cs)类来使用。代码修改如下：

        public void Configure(IApplicationBuilder app)
        {
            var endpoint = new RouteHandler((c) => c.Response.WriteAsync($"Hello, I am Routing! your item is {c.GetRouteValue("item")}"));
            var resolver = app.ApplicationServices.GetRequiredService<IInlineConstraintResolver>();
            var runRoute = new Route(endpoint,"{item}",resolver);

            app.UseRouter(runRoute);
        }

修改完代码后，我们再次编译运行，然后输入http://localhost:5000/  我们发现一片空白，然后再输入http://localhost:5000/abc  发现打印出来的是Hello, I am Routing! your item is abc。然后再输入其他的 http://localhost:5000/abc/cc 发现也是一片空白。这是因为我们给路由添加的匹配是`主机地址/+{item}`那其他的路径都是匹配不到，那么肯定就是不会显示任何东西啦。假设我们要给一个默认值，那么可以改成这样

>var runRoute = new Route(endpoint,"{item=home}",resolver);

OK，这个时候我们再输入http://localhost:5000/ 看到的就是Hello, I am Routing! your item is home。

匹配原理相对比较复杂点，想要了解的话可以参考 [RouteBase的源码](https://github.com/aspnet/Routing/blob/dev/src/Microsoft.AspNetCore.Routing/RouteBase.cs)，然后看相关的类，看看我们设置的模板是如何解析的，然后如何和url进行匹配的。如果要要来解释完整个过程的话，这个文章肯定是不够的，所以各位可以自己了解一下。

假如要配置多个路由支持的话，可以使用[RouteCollection](https://github.com/aspnet/Routing/blob/dev/src/Microsoft.AspNetCore.Routing/RouteCollection.cs)

        public void Configure(IApplicationBuilder app)
        {
            var endpoint = new RouteHandler((c) => c.Response.WriteAsync($"Hello, I am Routing! your item is {c.GetRouteValue("item")}"));
            var resolver = app.ApplicationServices.GetRequiredService<IInlineConstraintResolver>();
            var runRoute = new Route(endpoint,"{item=home}",resolver);
            var otherRoute = new Route(endpoint,"other/{item=other_home}",resolver);

            var routeCollection = new RouteCollection();
            routeCollection.Add(runRoute);
            routeCollection.Add(otherRoute);

            app.UseRouter(routeCollection);
        }

修改成上面的代码后就支持两个路由，假如输入的url是 http://localhost:5000/other 那么就是使用runRoute，如果输入的是http://localhost:5000/other/myother 那么使用的就是otherRoute。

这样书写暴露了很多细节东西，我们可以用 Routing提供的[RouteBuilder类](https://github.com/aspnet/Routing/blob/dev/src/Microsoft.AspNetCore.Routing/RouteBuilder.cs)来编写相同的东西。代码修改一下如下：

    public void Configure(IApplicationBuilder app)
        {
            var endpoint = new RouteHandler((c) => c.Response.WriteAsync($"Hello, I am Routing! your item is {c.GetRouteValue("item")}"));
            
            var routeBuilder = new RouteBuilder(app)
            {
                DefaultHandler = endpoint,
            };
            
            routeBuilder.MapRoute("default","{item=home}");
            routeBuilder.MapRoute("other","other/{item=other_home}");


            app.UseRouter(routeBuilder.Build());
        }

如果有一些特殊的的路由配置，我们也可以使用`routeBuilder.Routes.Add(route);`这代码来添加。至于能配置的模板都有些什么，可以看 Routing 的 Template 的测试类：https://github.com/aspnet/Routing/tree/dev/test/Microsoft.AspNetCore.Routing.Tests/Template 看完基本就知道都有些什么样的模板格式可以使用了。

## 实现自己的 RouteHandler

到现在，我们已经知道了Routing大概是怎么运行起来，知道了如何简单的使用。那么接下来可以来创建一个自己的RouteHandler，来加深一下对Routing的使用体验。

创建一个类MyRouteHandler,实现接口IRoute:

    public class MyRouteHandler : IRouter
    {
        public VirtualPathData GetVirtualPath(VirtualPathContext context)
        {
            return null;
        }

        public Task RouteAsync(RouteContext context)
        {

            context.Handler = (c) =>
            {
                var printStr = $"controller:{c.GetRouteValue("controller")}," +
                $"action:{c.GetRouteValue("action")},id:{c.GetRouteValue("id")}";

                return c.Response.WriteAsync(printStr);
            };
            return TaskCache.CompletedTask;
        }
    }

然后我们的路由配置改成这样：

        public void Configure(IApplicationBuilder app)
        {
            var endpoint = new MyRouteHandler();
            
            var routeBuilder = new RouteBuilder(app)
            {
                DefaultHandler = endpoint,
            };
            
            routeBuilder.MapRoute("default","{controller=Home}/{action=Index}/{id?}");

            app.UseRouter(routeBuilder.Build());
        }

然后打开浏览器http://localhost:5000/ 打印出来的内容是 `controller:Home,action:Index,id:` 。这样是不是很像我们去调用mvc的控制器和控制器的行为呢？Routing的体验文章到这来就结束了，谢谢观看。

--------------------
由于本人水平有限，知识有限，文章难免会有错误，欢迎大家指正。如果有什么问题也欢迎大家回复交流。要是你觉得本文还可以，那么点击一下推荐。