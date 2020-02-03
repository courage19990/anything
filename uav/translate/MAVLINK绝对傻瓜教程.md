本文翻译自__Shyam Balasubramanian__所写的__MavLink Tutorial for Absolute Dummies (Part –I) __，仅供学习与交流。

# MAVLink绝对傻瓜教程（Part –I）

__MavLink__是什么鬼？它是一种通信协议。当看到这个概念后，大伙就开始畏惧它。这个教程将迫使你深刻地领悟它，并通俗易懂地阐述它究竟是什么，它是如何工作的，最重要的是它到底是如何工作的！！我将试着解释Mission Planner如何与APM/ PX4通信，反之亦然。这将有助于你的扩展，并激发你潜在的天才程序员天赋，如果你还没激发的话！！

PS. Mission Planner是一款开源的无人机地面站软件，APM/ PX4则是一种开源飞控

本教程假定：

1. 你是个小白☹我也曾经是，但现在不再是了!
2. 你至少在C语言方面具备一定的编程技能（例如，在C/C++/C#/Java中编写过简单的switch cases）。如果你已经是专业级，那么return 0;
3. 你很严肃地打算去学习知识，因为你将为此失去一些睡眠！

但不管怎样，请始终保持学习的意愿，我衷心希望你永远不会忘记这一点🙂我可以开始了吗？

## 我为你而来，MavLink

Mavlink消息基本上是由Mission Planner（MP）编码，并通过USB串行或遥测发送到APM的一个字节流（两者不能同时使用。如果同时插入，则优先选择USB，而忽略遥测）。这里的__编码__并没有什么特别之处，只是把数据包放入一个数据结构中，然后通过信道以字节的形式发送出去，同时加上一些错误纠正。

## MavLink消息的结构：

每个MavLink数据包的长度为17字节，结构如下：

```
消息长度 = 17 (6 bytes header + 9 bytes payload + 2 bytes checksum)
```

```
长度为6个字节的首部 header
0. message header, 永远为 0xFE
1. message length (9)
2. sequence number -- 在 255 和 0 之间轮转(0x4e，前一个是 0x4d)
3. System ID - 什么系统在发送这个消息 (1)
4. Component ID- 系统的哪个组件正在发送消息 (1)
5. Message ID (例如 0 = heartbeat 等等! 别害羞，你也可以添加..)

可变长的有效负载 Payload (由八位组构成, 范围是 0..255)
** Payload (我们感兴趣的实际数据)

校验和 Checksum: 用于错误检测
```

PS. “由八位组构成”在原文中是specified in octet 1

软件所做的事情是，检查它是否为一个有效的消息（通过检查Checksum判断消息是否已损坏，丢弃已经损坏的消息）。这是遥测的波特率为什么设置为57,600而不是115,200 bps的原因之一。该数值越低，软件容易发生的错误就越少，虽然消息更新到地面站的速度会慢一些。如果你想在采用MavLink协议的同时取得更远点的距离，进一步降低波特率可能是一个好主意。然而，需要注意的是，经过测试，57,600bps在理论上可以通过3DR遥测无线电提供大约一英里半径的覆盖范围。还记得高中时的信噪比（SNIR）的概念吗?

现在，根据上面的内容，我们感兴趣的东西是：

* System ID（亦称消息的来源）：这是指通过无线遥测或USB端口向APM发送消息的发送源（也就是MP），软件会定期进行检查，以确认消息是发送给它的。
* Component ID （亦称系统中的子系统）：主系统中的任何子系统。目前，这里并没有子系统，我们也没有真正利用这个字段。
* Message ID：标识这是条关于什么的信息。在本教程中，我们将其称为“主消息（main message）”。
* Payload（实际数据）：这就是肉！这就是你想要的！！

## MavLink到底是如何工作的 

MavLink啥也不是就是一条消息。MavLink也可以用于地面机器人，所以微型飞行器链路（Micro Aerial Vehicle Link，也就是MavLink） 并不是一个十分贴切的名字。之所以采取这种方式命名呢，是因为它起始于直升机（如果我没猜错的话）。

“消息”是一个包含“常量字节数”（如前所述，即17字节）的数据包。APM（从空中）获取流字节，将其传输到硬件接口（例如，通过UART或遥测），并在软件中对消息进行解码。注意，<u>__消息__包含着我们即将提取的有效负载（payload）</u>。

我们对__有效负载__很感兴趣，但是，嘿，和有效负载在一起的是__Message ID__（见上面），<u>我们可以通过它了解__有效负载__代表着什么</u>。在这之前，先看看程序解释一切MavLink消息的几个步骤：

1. 我们有一个名为handlemessage (msg)的方法。这个就是你需要学习和了解的方法！在GCS_MavLink.pde中找到它（在Arducopter/ ArduPlane里）。

   它基本上是在问信息包：嘿，你是谁？你是为我而来还是试图侵入我的系统？在我给予你许可之前，让我先读一下你的__系统ID和组件ID__。<u>任何使用MavLink的系统都有一个系统ID和组件ID</u>。例如，你的MP和正在飞行的四轴飞行器将具有相同的系统ID。组件ID则用于附加到APM/PX4的“子系统”。

   注意：当前，系统ID和组件ID被__<u>硬编码</u>__为相同的。

   现在，你仅有一个遥测和一架带着APM的直升机，就是这样，去愉快地飞行吧-不用再考虑其它的！这些东西对一群的直升机也许有帮助（在未来），未来将会有不同的系统ID🙂

2. 我们从消息中提取有效负载并放入一个__包（packet）__中。一个包是基于一种“信息类型”的数据结构。我们将不再使用“消息”一词，那玩意儿到这里就结束了。我们基本上只对由“原始数据”打包而成的包感兴趣。

3. 包被放入一个“适当的数据结构”中。有许多数据结构 ，例如用来存放姿态（俯仰，横摇，偏航方向）的，GPS的, 无线电控制信道的等，也就是，把相似的东西组合在一起，形成易于理解的模块。 这些数据结构在发送端和接收端（即在MP和APM端）__“100%完全相同”__。 如果不一样的话，你的直升机就会在奇怪的时间坠毁！

PS. 无线电控制信道原文中为RC channel

此外，这也是MavLink GUI生成器大展身手的地方。为了生成这些数据结构，我们不需要编写任何代码！！嗯，也不完全是，但只有一点点。

到目前为止很简单对不对？如果觉得困难请重读一遍！！当个笨蛋没关系的🙂

好了，现在说正事。我们使用MavLink发送双向消息。

> 从地面控制站（GCS）到APM/PX4或从APM/PX4到地面控制站（GCS）。注意，我说的GCS是指Mission Planner（MP）或者QGroundControl（QGC）、DroidPlanner（DP）或者你自定义的与直升机通信的工具。

## 地面控制站（GCS）到四轴飞行器：

到目前为止，我们知道每个消息（我们叫一个包，里面有对我们有用的信息）都有一个消息ID和有效负载，并将其放入适当类型的数据结构中。我们根据主消息（或MAVLINK_MSG_ID_）进行选择，一旦检测到特定类型的消息，就会执行一些神奇的操作，比如将接收到的信息存储到永久内存中，或者执行我们希望对其执行的任何操作。

截至2013年11月，在Arducopter最新的3.0.1 RC5（候选版本）中，这些是你会发现的参数。我已经试着列出所有可能的MavLink消息的“主消息”ID。

请注意，在每个“主要信息”类别中(如下面粗体部分)你会找到属于那个类别的“子消息”，它们基本上与有效负载信息（真正的肉）及其处理方式密切相关。就像“自行车”类别有雅马哈，铃木，哈雷戴维森等。我列出了所有的主要信息类别，但只指定了一些子类别。你可以自己查一下细节🙂因为如果你明白我到目前为止的意思，你就不再是个傻瓜了。明白我的意思吗？

__MAVLINK_MSG_ID__（主消息块）：

1. MAVLINK_MSG_ID_HEARTBEAT：//0
   * 这是__最重要的信息__。GCS不断地向APM/PX4发送信息，以确定它是否与之相连（每1秒一次）。这是为了确保在更新某些参数时MP与APM是同步的。如果错过了许多心跳信号，则会触发故障保护（可能），直升机着陆、继续执行任务或返回发射（Returns to launch，也称为RTL）。在MP的“配置/设置故障保护选项”下，可以启用/禁用故障保护选项。但你不能停止心跳，对吗？这个名字很有道理！!
2. MAVLINK_MSG_ID_REQUEST_DATA_STREAM：//66
   * 传感器，无线电控制信道，GPS位置，状态，额外1／2／3
3. MAVLINK_MSG_ID_COMMAND_LONG: // 76 
4. SET_MODE: //11 
5. MAVLINK_MSG_ID_MISSION_REQUEST_LIST: //43 
6. MAVLINK_MSG_ID_MISSION_REQUEST: //40 
7. MAVLINK_MSG_ID_MISSION_ACK: //47 
8. MAVLINK_MSG_ID_PARAM_REQUEST_LIST: //21 
9. MAVLINK_MSG_ID_PARAM_REQUEST_READ: //20 
10. MAVLINK_MSG_ID_MISSION_CLEAR_ALL: //45 
11. MAVLINK_MSG_ID_MISSION_SET_CURRENT: //41 
12. MAVLINK_MSG_ID_MISSION_COUNT: // 44 
13. MAVLINK_MSG_ID_MISSION_WRITE_PARTIAL_LIST: // 
14. MAVLINK_MSG_ID_SET_MAG_OFFSETS: //151 
15. MAVLINK_MSG_ID_MISSION_ITEM: //39 
16. MAVLINK_MSG_ID_PARAM_SET: //23 
17. MAVLINK_MSG_ID_RC_CHANNELS_OVERRIDE: //70 
18. MAVLINK_MSG_ID_HIL_STATE: //90 
19. MAVLINK_MSG_ID_DIGICAM_CONFIGURE: // 
20. MAVLINK_MSG_ID_MOUNT_CONFIGURE: // 
21. MAVLINK_MSG_ID_MOUNT_CONTROL: // 
22. MAVLINK_MSG_ID_MOUNT_STATUS:// 
23. MAVLINK_MSG_ID_RADIO, MAVLINK_MSG_ID_RADIO_STATUS: // 

## 四轴飞行器到地面控制站（GCS）到四轴飞行器：

好吧，我承认，这变得更有趣了，但这其实容易得多。事实上，GCS只是你和直升机之间的中介物。它反过来从直升机获取数据，显示在GCS上。

如果你打开了Arducopter.pde文件，请查看代码的这一部分：

```c
static const AP_Scheduler::Task scheduler_tasks[] PROGMEM = {
. . .
. . .
{ gcs_send_heartbeat, 100, 150 },
{ update_notify, 2, 100 },
{ one_hz_loop, 100, 420 },
{ gcs_check_input, 2, 550 },
{ gcs_send_heartbeat, 100, 150 },
{ gcs_send_deferred, 2, 720 },
{ gcs_data_stream_send, 2, 950 },
. . .
. . .
. . .
```

别害怕。这比呼吸还容易。这就是实时系统概念发挥作用的地方。我们希望某些任务需要一定的时间，如果到那时它们还没有完成，那么我们就不会继续执行它们。

第一个参数是函数名，

第二个是“它应该花费的时间”（以10毫秒为单位，即2个平均值，20毫秒，即50赫兹，即该功能每秒运行50次）。

第三个参数是“函数不应运行的最长时间”。

我觉得这很简单！你在那里看到的每一个函数，它的未来都是注定的，它运行的时间正是如此之长。这就是为什么对这些讨厌的机器使用实时系统是安全的，让它可以预测而不是不可预测！！

所有这些功能都是精心挑选的，让您知道这些功能与地面军事系统的更新有关。简单地说，进入每一个函数定义，这些函数将始终带您到GCS_Mavlink.pde，在那里GCS通信的实际操作发生了！!

最有趣（也是最重要）的是：

```c
/*
* send data streams in the given rate range on both links
*/
static void gcs_data_stream_send(void)
{
	gcs0.data_stream_send();
	if (gcs3.initialised) {
		gcs3.data_stream_send();
	}
}
```

这样做的目的是通过链路发送数据（gcs0通过USB，gcs3通过遥测）。如果你再深入一点，你就会知道我们会把这些数据结构发送回地面军事系统显示。

例如，当你用手移动你的直升机并看到任务规划器的HUD屏幕时会发生什么？你看到直升机在屏幕上移动。我们得到了每个时间单位的姿态数据（俯仰、横摇和偏航）。同样，我们有IMU数据、GPS数据、电池数据等。