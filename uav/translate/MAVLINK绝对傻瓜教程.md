__MavLink__是什么鬼？它是一种通信协议。当看到这个概念后，大伙就开始畏惧它。这个教程将迫使你深刻地领悟它，并通俗易懂地阐述它究竟是什么，它是如何工作的，最重要的是它到底是如何工作的！！我将试着解释Mission Planner如何与APM/ PX4通信，反之亦然。这将有助于你的扩展，并激发你潜在的天才程序员天赋，如果你还没激发的话！！

PS. Mission Planner是一款开源的无人机地面站软件，APM/ PX4则是一种开源飞控

本教程假定：

1. 你是个小白☹我也曾经是，但现在不再是了!
2. 你至少在C语言方面具备一定的编程技能（例如，在C/C++/C#/Java中编写过简单的switch cases)
   ）。如果你已经是专业级，那么return 0;
3. 你很严肃地打算去学习知识，因为你将为此失去一些睡眠！

但不管怎样，请保持学习的意愿，我衷心希望你永远不会忘记它🙂我可以开始了吗？

## 我为你而来，MavLink

Mavlink消息基本上是由Mission Planner（MP）编码，并通过USB串行或遥测发送到APM的一个字节流（两者不能同时使用。如果同时插入，则优先选择USB，而忽略遥测）。这里的编码并没有什么特别之处，只是把数据包放入一个数据结构中，然后通过信道以字节的形式发送出去，同时加上一些错误纠正。

## MAVLINK消息的结构：

每个MavLink数据包的长度为17字节，结构如下：

```
消息长度 = 17 (6 bytes header + 9 bytes payload + 2 bytes checksum)
```

```
6 bytes header
0. message header, 永远为 0xFE
1. message length (9)
2. sequence number -- 在 255 和 0 之间轮转(0x4e，前一个是 0x4d)
3. System ID - 什么系统在发送这个消息 (1)
4. Component ID- 系统的哪个组件正在发送消息 (1)
5. Message ID (e.g. 0 = heartbeat and many more! Don’t be shy, you can add too..)
Variable Sized Payload (specified in octet 1, range 0..255)
** Payload (the actual data we are interested in)
Checksum: 用于错误检测
```








6 bytes header

0. message header, always 0xFE
1. message length (9)
2. sequence number -- rolls around from 255 to 0 (0x4e, previous was 0x4d)
3. System ID - what system is sending this message (1)
4. Component ID- what component of the system is sending the message (1)
5. Message ID (e.g. 0 = heartbeat and many more! Don’t be shy, you can add too..)
   Variable Sized Payload (specified in octet 1, range 0..255)
   ** Payload (the actual data we are interested in)
   Checksum: For error detection. '



