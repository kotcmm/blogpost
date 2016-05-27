##前言

随着net core rc2的发布，园子里面关于net core的入门文章也也多了起来，但是大多数都是在一个平台上面来写几个简单的例子，或者是在解释代码本身，并没有体现说在一个平台上面创建一个项目，然后通过源码管理签出到另一个平台上继续开发。还有就是一次编译到处运行是如何体现的？这样类型的文章好像还没有在哪里看到过，于是我就想自己来一遍，并把这个过程分享出来给大家。

首先来个官方的介绍：

[.NET Core - .NET 使用 .NET Core 跨平台运行](https://msdn.microsoft.com/zh-cn/magazine/mt694084.aspx)

[Announcing .NET Core RC2 and .NET Core SDK Preview 1](https://blogs.msdn.microsoft.com/dotnet/2016/05/16/announcing-net-core-rc2/)


## 一、mac os x 上的开发

本来环境安装是不想写的，因为有太多的文章，但是不写的话又感觉少了些什么。然后我只有一个mac和一个安装win10的pc，所以就只针对这两个平台来，linux等我有钱买新电脑再来折腾。废话不多说，下面开始介绍。

####mac os x环境下 dotnet core 的安装


参考链接： https://www.microsoft.com/net/core#macosx

首先要先安装一个  [OpenSSL](https://www.openssl.org/) 这个是以后 dotnet core 要下载一些依赖库什么的会用到的。可以使用 [brew](http://brew.sh/) 来安装[OpenSSL](https://www.openssl.org/) 

打开命令终端输入[brew](http://brew.sh/)安装命令

`/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"`

安装好[brew](http://brew.sh/) 之后继续输入[OpenSSL](https://www.openssl.org/)安装命令

`brew install openssl`

`brew link --force openssl`

这两个东西安装后，我们就可以来安装我们我们的core了，下载这么一个安装包 [点击下载](https://go.microsoft.com/fwlink/?LinkID=798400) 这是一个pkg安装包。`我现在的时候是 dotnet-dev-osx-x64.1.0.0-preview1-002702.pkg` 只要双击一下安装包，然后一直点击继续就可以了

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160524222803897-1787170527.png)

安装完之后可以输入命令查看dotnet core 有没有安装成功

`dotnet -v`

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160524223226475-1006207901.png)

####命令行下的项目建立与运行

安装完环境后，我们就可以来开始我第一个项目。我们新建一个文件夹叫做`LearnDotnetCore`然后打开命令窗口 cd 到这个文件夹下面。然后用命令 `dotnet new`来建立初始化一个项目。初始化完后我们看看到文件夹下面多了两个文件 Program.cs	project.json 

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160524230326303-571634466.png)

`Program.cs 是c#源码文件，可以用来实现我们的具体逻辑`

`project.json 是配置依赖框架，依赖包，版本号，还有其他一些编译会用到相关的东西。这个作用和nodejs的模式是相类似的。`

项目创建完之后需要 `dotnet restore` 来下载我们配置好的依赖包。下载完后呢文件夹里面多出了一个 project.lock.json 文件，打开来看，可以看到我们下载了那些依赖库，还有一些编译的引用等。

下载完依赖库后，我们可以通过运行命令 `dotnet run` 来编译运行我们的程序。

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160524232412663-1850524485.png)

编译完成后，文件夹内会多出两个文件夹 bin 和 obj 。 

`bin 是我们编译的完成的文件`

`obj 里面是一些编译输出和动态库连接等的一些信息。`

然后运行我们可以看到输出 Hello World! 到这来，我们的命令行建立、编译和运行程序就到这来结束了。

#### 使用vscode调试运行

参考链接：https://docs.asp.net/en/latest/tutorials/your-first-mac-aspnet.html#running-locally-using-kestrel

单纯使用命令行来开发的话，那么肯定是没有那么方便了，但是在但是目前好像也没有什么特别好的选，所以我们就使用 [Visual Studio Code](https://code.visualstudio.com/) 然后再安装一个插件[C# extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode.csharp)

还有一个叫做[Xamarin Studio.](https://www.xamarin.com/studio)，貌似还没有支持到 dotnet core的样子

当然还有一个叫做 [Project Rider](https://www.jetbrains.com/rider/)的东西，之前 net core 用 DNX 命令的时候下载来玩过一下，支持什么的还可以，不过很久没有玩了，最近一次编译更新是在 5月17号，也不知道现在支持如何。有兴趣的同学自己下载来玩下，内容不在本文章中介绍了。

打开vscode然后把项目的文件夹拖拉到vscode的窗口上面去，会在左边的工作空间栏目中显示项目的树形结构。然后我们选择project.json，编辑器会提示你是否要加入调试信息，然后我们点击yes，目录结构就会多出一个.vscode文件夹里面有两文件 [launch.json](https://code.visualstudio.com/docs/editor/debugging#_launch-configurations) 和 [tasks.json](https://code.visualstudio.com/docs/editor/tasks#_mapping-gulp-grunt-and-jake-output-to-problem-matchers)

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160524235532444-568159065.png)


然后我们去修改 launch.json 里面的内容，把program里面的内容改成我们编译完成后的dll路径，当然有的有时候会给你自动生成好的，不需要自己修改。

"program": "${workspaceRoot}/bin/Debug/netcoreapp1.0/LearnDotnetCore.dll",

还有就是args也修改一下，这个是main函数的传入值。

"args": ["我试试看传入了什么"],

修改完后在Program.cs里面加个断点，接着直接按f5，或者点开debug面板，然后点击那个调试按钮开始调试

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160525003740459-8401405.png)

我们可以看到，一个变量的监视数据，还有就是控制台打印出了 Hello World!

####代码上传
我选择了用github来保存代码，并且在window上通过git下载到[vs](https://www.visualstudio.com/)里面进行继续开发

首先我们打开git面板来初始化git储存库
![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160527013539881-2072818395.png)

初始化完后，我们可以看到我们有15个文件要提交，但是我们并不需要把这么多个文件都给提交了，我们只要提交Program.cs	project.json 这两个文件，然后在其他地方下载后来生成其他文件就可以了。
![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160527013557756-594067097.png)

于是，我们就添加了一个.gitignore文件，文件内容添加要排除的东西，然后我们就剩下三个文件提交了
![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160527013616428-409467087.png)

最后，输入提交的信息然后点击那个勾符号，就提交到本地的库里面了。
![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160527013629897-1512197612.png)

完成了上面的步骤后，我们还需要把库同步到[github](https://github.com)上面去。首先，我们去注册一个账户，然后新建一个repository，建立过程自己摸索，就不写的那么详细。建完后如下图
![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160527015731897-747914593.png)
然后到我们的目录下输入下面的两条命令

>git remote add origin https://github.com/kotcmm/LearnDotnetCore.git

>git push -u origin master

如果有用户名密码，就输入用户名密码，然后存储库就同步到了[github](https://github.com)上面去了。记住地址什么的要改成自己的。

##二、window 上的开发

切到win10上面来继续完成这个博文。

####开发环境的安装

参考资料：https://www.microsoft.com/net/core#windows

首先要先下载下面的工具：

[Visual Studio Community 2015](https://www.visualstudio.com/products/visual-studio-community-vs)

[DotNetCore.1.0.0.RC2-VS2015Tools.Preview1](https://go.microsoft.com/fwlink/?LinkId=798481)

这两个软件的安装就不需要截图了，先安装Visual Studio Community 2015再安装DotNetCore.1.0.0.RC2-VS2015Tools.Preview1。安装完之后我们打开vs 2015，然后把我们之前在mac上面创建的项目给下载下来。

打vs后我们选择 open from source control 然后出现左边的面板，如果没有登录你的github账号，那么就先登录你的账号，然后再点击Clone出现一个窗口选择我们前面上传的那个项目。

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160527131940741-1680327735.png)

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160527133430913-124986553.png)

下载完后双击我们的git项目，然后在弹出的窗口里面选择project.json，就会生成左边的那样项目工程了。

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160527133501006-59647658.png)

然后我们编译成功并运行起来。

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160527133714834-922331963.png)

到这里我们已经成功的把在mac上面创建的项目拉到win下不改任何代码，然后成功编译运行了。


#### 发布运行

参考资料：[两种部署方式](http://dotnet.github.io/docs/core-concepts/app-types.html)

上面的开发过程中都是直接在id里面运行的，或者用 dotnet run 来直接运行的，那么我们编译好之后如何脱离IDE来运行呢？

1. 我们也可以到编译成功的目录 netcoreapp1.0 下面输入命令 dotnet LearnDotnetCore.dll  来运行我们的程序。
2. 可以在project.json加个 runtimes 属性，然后编译出一个exe文件，然后直接运行exe。

`要编译成Release的话，可以用命令 dotnet.exe build --configuration Release --no-dependencies --no-incremental  当然build的命令以后可以有单独的文章来讲解`

第一种编译的配置

    {
    "version": "1.0.0-*",
    "buildOptions": {
        "emitEntryPoint": true
    },
    "dependencies": {
        "Microsoft.NETCore.App": {
        "type": "platform",
        "version": "1.0.0-rc2-3002702"
        }
    },
    "frameworks": {
        "netcoreapp1.0": {
        "imports": "dnxcore50"
        }
    }
    }
    
第二种编译的配置

    {
    "version": "1.0.0-*",
    "buildOptions": {
        "emitEntryPoint": true
    },
    "dependencies": {
        "Microsoft.NETCore.App": {
        //"type": "platform",
        "version": "1.0.0-rc2-3002702"
        }
    },
    "frameworks": {
        "netcoreapp1.0": {
        "imports": "dnxcore50"
        }
    },
    "runtimes": {
        "win10-x64": {},
        "osx.10.11-x64": {}
    }
    }

第二种配置使用dotnet publish会把core的支持运行一起打包处理，然后相关的平台就不用安装net core,但是不同平台需要不同编译。具体的效果和目录结构我就不进行截图了，有兴趣的自己编译看看。

##三、改造成web项目

前面介绍了如何编译和运行控制台的程序，但是在这个互联网的时代，我感觉要做成一个web项目可能会更有趣，我们看看如何直接把前面控制台的项目变成web项目。

参考资料：https://docs.asp.net/en/latest/getting-started.html

####vs修改运行

首先修改配置 project.json，在属性dependencies添加Kestrel依赖。

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160527181439631-12509684.png)

第二添加一个类Startup

    using System;
    using Microsoft.AspNetCore.Builder;
    using Microsoft.AspNetCore.Hosting;
    using Microsoft.AspNetCore.Http;

    namespace LearnDotnetCore
    {
        public class Startup
        {
            public void Configure(IApplicationBuilder app)
            {
                app.Run(context =>
                {
                    return context.Response.WriteAsync("Hello from ASP.NET Core!");
                });
            }
        }
    }

然后修改Program.cs

    using System;
    using Microsoft.AspNetCore.Hosting;

    namespace LearnDotnetCore
    {
        public class Program
        {
            public static void Main(string[] args)
            {
                var host = new WebHostBuilder()
                    .UseKestrel()
                    .UseStartup<Startup>()
                    .Build();

                host.Run();
            }
        }
    }

然后点击运行按钮，程序成功的运行起来

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160527191849069-618253433.png)

打开浏览器访问 http://localhost:5000/ 我们可以看到浏览器打印出 Hello from ASP.NET Core!证明我们的程序已经编译并运行成功。

####发布并部署iis

虽然说直接运行可以访问成功，但是有的人就是喜欢部署到IIS上面去，所以这里也尝试一下部署到IIS上面是什么样的。

参考资料：https://docs.asp.net/en/latest/publishing/iis.html

安装完IIS后需要安装 [DotNetCore.1.0.0.RC2-WindowsHosting](http://go.microsoft.com/fwlink/?LinkId=798480)来支持 net core 在iis上面运行

然后修改project.json的内容,添加下面三个东西，来支持iis运行

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160527202115913-1198565477.png)

然后修改Program.cs,添加iis支持

    using System;
    using System.IO;
    using Microsoft.AspNetCore.Hosting;

    namespace LearnDotnetCore
    {
        public class Program
        {
            public static void Main(string[] args)
            {
                var host = new WebHostBuilder()
                    .UseKestrel()
                    
                    .UseContentRoot(Directory.GetCurrentDirectory())
                    .UseIISIntegration()
                    
                    .UseStartup<Startup>()
                    .Build();

                host.Run();
            }
        }
    }

添加文件 web.config

    <?xml version="1.0" encoding="utf-8"?>
    <configuration>

    <!--
        Configure your application settings in appsettings.json. Learn more at http://go.microsoft.com/fwlink/?LinkId=786380
    -->

    <system.webServer>
        <handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModule" resourceType="Unspecified"/>
        </handlers>
        <aspNetCore processPath="%LAUNCHER_PATH%" arguments="%LAUNCHER_ARGS%" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" forwardWindowsAuthToken="false"/>
    </system.webServer>
    </configuration>

上面三个修改完之后，就可以来发布，选择项目邮件，然后选择发布会出现一个窗口

![](http://images2015.cnblogs.com/blog/248834/201605/248834-20160527203320663-1112836804.png)

输入名称，然后next或者直接publish，操作完成后会生成一堆文件。然后到iis里面创建 Web 站点，然后绑定发布目录就行，应用程序池的模式需要改为“无代码托管”。

启动网站运行后，浏览器输入http://localhost/ 可以看到浏览器打印出 Hello from ASP.NET Core!和上面没有在iis里面运行的效果是一样的。

##四、申请vps并上传项目运行


--------------------
由于本人水平有限，知识有限，文章难免会有错误，欢迎大家指正。如果有什么问题也欢迎大家回复交流。要是你觉得本文还可以，那么点击一下推荐。
