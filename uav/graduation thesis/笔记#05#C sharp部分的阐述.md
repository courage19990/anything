## 一、准备工作

　　这部分主要是介绍开始C#编程前的一些准备工作。读者既可以参照本文完全从头做起，也可以基于已经写好的代码继续开发。笔者假设读者已具备一个可以正常编写并运行C#程序的Visual Studio 2017开发平台。开始编程前的第一步，是建立一个新的Windows Forms项目。如下图所示。在这里，唯一需要注意的是.Net Framework的版本。以避免出现一些不兼容的情形。

![img](https://img2020.cnblogs.com/i-beta/1042431/202003/1042431-20200316171533890-490180948.png)

　　建立好项目之后的第一件事情是，用NuGet包管理工具添加开发地面站所需要的外部依赖。这是极为关键的一步，也就是引用由他人写好的供人调用的程序包以方便自己写程序。编程实现本地面站，主要用到了以下几个外部依赖：

1. Fleck包。便于我们与地面站的浏览器端进行WebSocket通信。
2. MAVLink包。它封装着关于MAVLink的一切，便于我们接收、发送MAVLink消息包，而完全不需要深入了解底层的细节。关于MAVLink的详情我在前面的章节已介绍过了。
3. Newtonsoft.Json包。该包在程序中唯一的应用在于，将MAVLink消息包转化为JSON格式。以方便地面站的浏览器端进一步处理。实质上就是将C#中的对象转化为一个特定格式的字符串。再在浏览器端进行反序列化成JS对象。

　　以上几个包在笔者所编写程序中的具体版本，如下图所示，实际自主编程时可选择任意兼容的版本。

![img](https://img2020.cnblogs.com/i-beta/1042431/202003/1042431-20200316170312719-1575009589.png)

　　最后，还有一个重要的细节，右键资源框中的项目名，并在弹出的菜单中点击“property”，在设置页面将项目的Output type更改为Console Application。这样运行地面站时，命令行提示框才会正常显示。一开始创建为Windows Forms是为了方便自动生成一些初始代码。

![img](https://img2020.cnblogs.com/i-beta/1042431/202003/1042431-20200316200504970-101312961.png)

## 二、设计与实现思路

　　编写C#端的程序，笔者主要采用了面向对象的开发思想。面向对象思想的核心要义只有一条，就是将对象视为服务的提供者。良好的抽象可以大大降低编程的复杂程度，提高代码的可读性。

　　笔者将一启动地面站就首先弹出来的那个窗体，也就是创建项目时自动生成的那个类，改名为Entrance。该类将完成地面站软件的所有顶层工作。一旦启动地面站，Entrance类对象会首先被创建。考虑到程序后续的可扩展性，即易于在原基础上添加一个完全由C#实现的视图。笔者在Entrance类中保留了以下代码。这样就可以方便地更换视图的实现，而不用冒险修改大量代码。

```c#
namespace SimpleGCS4MAVLink
{
    // 作为地面站程序的入口
    public partial class Entrance : Form
    {
        ······
        // 定义一个更新GUI的委托类，暂时弃用
        public delegate void UIUpdator(MAVLink.MAVLinkMessage packet);
        // 定义一个委托变量的引用，暂时弃用
        private UIUpdator updator;
        public Entrance()
        {
            InitializeComponent();
            updator = MyUIUpdator; // 暂时弃用
        }
        ······
        // 该方法在GCS开启的子线程中执行
        private void MessageHandler(MAVLink.MAVLinkMessage packet)
        {
            ······
            // BeginInvoke(updator, packet); 若要实现C#的GUI版本，取消该行注释
        }
        // 该方法在主线程中执行
        private void MyUIUpdator(MAVLink.MAVLinkMessage packet)
        {
            // 若要实现C#的GUI版本，在这里更新UI组件
        }
    }
}
```

　　笔者个人选择实现的是一个运行于浏览器的前端视图。所以在创建Entrance类对象的同时，创建了一个MyWebSocketServer类对象。顾名思义，该类抽象表示一个WebSocket服务器。默认的地址为ws://127.0.0.1:8095。该地址可以很容易地在程序中修改。假设我已经创建该类的一个对象，其引用为s。那么我们就可以通过调用s.sendMessage方法向所有与该地面站后端连接的浏览器端口发送WebSocket消息。需要注意的是，在将WebSocket消息发出去之前，我们需要先将MavLink消息包对象转化为JSON格式以便前端进行处理。相关代码如下。

```c#
        // 该方法在GCS开启的子线程中执行
        private void MessageHandler(MAVLink.MAVLinkMessage packet)
        {
            server.SendMessage(JsonConvert.SerializeObject(packet));
        }
```

　　接下来介绍C#端代码的主要部分。下图为即将介绍的各个类之间的关系。

![img](https://img2020.cnblogs.com/i-beta/1042431/202003/1042431-20200316194340388-1814442529.png)

　　Entrance类，上文已经提到过，即地面站入口。GCS4MAVLink类，抽象表示无人机地面站。Autopilot类，抽象表示无人机飞控。MyWebSocketServer类，上文已经说过了，主要用于协助完成数据可视化的任务。飞控之于地面站、地面站之于地面站入口，都是必不可少且不可替换的。而地面站视图的实现方式，也就是WebSocket服务器相对地面站入口而言，必不可少但可以替换。因此前者中的关系较强，是组合关系，后者中的关系较弱，是聚合关系。

　　随着Entrance类对象、MyWebSocketServer类对象的创建，同时还将创建一个GCS4MAVLink类对象。这里保留了未来监控多无人机的可能性。倘若地面站需要监控多个无人机。那么就需要创建多个GCS4MAVLink类对象。而要完成来自多无人机数据的可视化，只需创建多个MyWebSocketServer类对象并绑定不同的端口。然后，通过浏览器开启多个网页分别连接不同的“WebSocket服务器”端口即可实现对多无人机的监控。

　　那么GCS4MAVLink类对象做了什么事情呢？该类对象提供了一个startup方法，在确保物理连接没问题之后，我们只需要向该方法传入相应的端口号和波特率作为参数就可以去连接一个无人机飞控。以及我们必须传入一个消息处理器，对来自无人机的数据进行响应。这里仍然体现着“松耦合”的编程思想，消息处理器是容易替换的，在本地面站实现中，所谓消息处理器只是一个方法而已，在上文的代码中已经有所体现。GCS4MAVLink类对象一旦创建并调用startup方法，就会按确定的频率向无人机飞控发送心跳包并请求除“心跳”外的其它数据，例如飞行姿态、GPS位置等等。倘若需要扩展更多的数据交互方法，扩展此类即可。

　　GCS4MAVLink类对象具体怎样和飞控交互呢？又是借助一个Autopilot类对象。该类对象用于抽象表示一个具体的飞控，实际上是一个SerialPort对象的代理人。但是我们在编写程序的过程中直接将其视为飞控就可以了，这样易于理解。在Autopilot类中，特别的方法只有GetPacket以及ReceiveBytes，前者用于从飞控读取一个MAVLink包，解析字节的过程由MAVLink包代我们完成了。后者则用于地面站向飞控发送MAVLink消息包。只需把创建好的MAVLink消息包所对应的字节串作为参数传入，调用该方法即可。

　　至此，借助于一系列外部依赖，一个总共只有4个自定义类的简易无人机后台就搭建好了。由于充分采取了面向对象的思想，该程序完全具备良好的可读性，以及可扩展性。