本文内容来源于[维基百科](https://en.wikipedia.org/wiki/MAVLink)，仅供学习参考。

# MAVLink

MAVLink（Micro Air Vehicle Link）是一种用于与小型无人机通信的协议。它被设计为一个header-only消息封送处理库。MAVLink由Lorenz Meier在LGPL许可下[2]于2009[1]年初发布。

PS. header-only库是一种“无需编译，包含头文件就可以用”的库。

[TOC]

## 应用

MAVLink主要用于地面控制站(GCS,  Ground Control Station)与无人机之间的通信，以及载具内部子系统间的通信。它可以用来传输载具的方位、GPS位置以及速度。

## 包结构

在1.0版本中包的结构如下：

| 字段名  | 索引（以字节作为长度单位） | 目的                                       |
| :--- | :------------ | :--------------------------------------- |
| 起始标识 | 0             | 表示帧传输的开始（v1.0: 0xFE）                     |
| 负载长度 | 1             | 有效负载的长度（n）                               |
| 包序号  | 2             | 每个组件都会计算它们各自的发送序号（每发送一个消息包，包序号加1），允许接收端通过包序号来检测丢包率。 |
| 系统ID | 3             | 用于唯一标识某网络中的一个发送系统。允许在同一网络上区分不同的系统。       |
| 组件ID | 4             | 用于唯一标识某网络中的一个发送组件。允许区分同一系统的不同组件，例如IMU和自动驾驶仪（即飞控）。 |
| 消息ID | 5             | 标识消息类型——定义有效负载的“含义”以及如何正确解码有效负载。         |
| 有效负载 | 6 到 (n+6)     | 消息包中的数据，其含义取决于消息类型。                      |
| CRC  | (n+7) 到 (n+8) | 整个包的校验和，不包括包的起始标识（LSB to MSB）            |

PS. 0xFE = 254，即一个十进制值为254的字节标识着一个新的MAVLink消息包的开始。

版本2之后，包结构扩展为以下[3]：

| 字段名                                | 索引（以字节作为长度单位）   | 目的                                       |
| ---------------------------------- | --------------- | ---------------------------------------- |
| 起始标识                               | 0               | 表示帧传输的开始（ v2: 0xFD）                      |
| 负载长度                               | 1               | 有效负载的长度（n）                               |
| 不兼容性标志（ incompatibility flag__s__） | 2               | 不兼容性标志用于指示MAVLink库必须支持的特性，如果某个MAVLink实现无法理解incompat_flags字段中的任一标志（flag），则它必须丢弃数据包。 |
| 兼容性标志                              | 3               | MAVLink实现可以安全地忽略compat_flags字段中它不理解的标志。  |
| 包序号                                | 4               | 每个组件都会计算它们各自的发送序号（每发送一个消息包，包序号加1），允许接收端通过包序号来检测丢包率。 |
| 系统ID                               | 5               | 用于唯一标识某网络中的一个发送系统。允许在同一网络上区分不同的系统。       |
| 组件ID                               | 6               | 用于唯一标识某网络中的一个发送组件。允许区分同一系统的不同组件，例如IMU和自动驾驶仪（即飞控）。 |
| 消息ID                               | 7 到 9           | 标识消息类型——定义有效负载的“含义”以及如何正确解码有效负载。         |
| 有效负载                               | 10 到 (n+10)     | 消息包中的数据，其含义取决于消息类型。                      |
| CRC                                | (n+11) 到(n+12)  | 整个包的校验和，不包括包的起始标识（LSB to MSB）            |
| 签名                                 | (n+12) 到 (n+26) | 该字段确保消息来自可信任的发送者。(可选)                    |

PS. 0xFD = 253

### CRC字段

详见原文。[4]\[5]\[6]\[7]\[8]

该字段主要用于确保消息包的完整性。MAVLink的循环冗余检查算法已经在Python和Java等多种语言中实现。

### 消息

上述数据包中的__有效__负载就是MAVLink消息。每条消息都由包上的ID字段进行标识，有效负载包含来自消息的数据。MAVlink源码[9]中的XML文档定义了存储在此有效负载中的数据。

下面是从XML文档中提取的ID为24的消息。

```xml
<message id="24" name="GPS_RAW_INT">
        <description>The global position, as returned by the Global Positioning System (GPS). This is NOT the global position estimate of the system, but rather a RAW sensor value. See message GLOBAL_POSITION for the global position estimate. Coordinate frame is right-handed, Z-axis up (GPS frame).</description>
        <field type="uint64_t" name="time_usec">Timestamp (microseconds since UNIX epoch or microseconds since system boot)</field>
        <field type="uint8_t" name="fix_type">0-1: no fix, 2: 2D fix, 3: 3D fix. Some applications will not use the value of this field unless it is at least two, so always correctly fill in the fix.</field>
        <field type="int32_t" name="lat">Latitude (WGS84), in degrees * 1E7</field>
        <field type="int32_t" name="lon">Longitude (WGS84), in degrees * 1E7</field>
        <field type="int32_t" name="alt">Altitude (WGS84), in meters * 1000 (positive for up)</field>
        <field type="uint16_t" name="eph">GPS HDOP horizontal dilution of position in cm (m*100). If unknown, set to: UINT16_MAX</field>
        <field type="uint16_t" name="epv">GPS VDOP horizontal dilution of position in cm (m*100). If unknown, set to: UINT16_MAX</field>
        <field type="uint16_t" name="vel">GPS ground speed (m/s * 100). If unknown, set to: UINT16_MAX</field>
        <field type="uint16_t" name="cog">Course over ground (NOT heading, but direction of movement) in degrees * 100, 0.0..359.99 degrees. If unknown, set to: UINT16_MAX</field>
        <field type="uint8_t" name="satellites_visible">Number of satellites visible. If unknown, set to 255</field>
</message>
```

注意：XML文档描述了协议字段的逻辑顺序。实际的连线格式（以及典型的内存表示）对字段重新排序[10]，以减少数据结构对齐问题。在阅读从消息定义生成的代码时，这可能会造成混淆。

### MAVLink生态圈

MAVLink在很多项目中作为通信协议使用，这可能意味着它们之间存在一定的兼容性。有人已经编写了一个有趣的教程[11]来解释MAVLink的基础知识。

## 参考

1.  ["Initial commit · mavlink/mavlink@a087528"](https://github.com/mavlink/mavlink/commit/a087528b8146ddad17e9f39c1dd0c1353e5991d5). *GitHub*.
2. **^** [http://qgroundcontrol.org/mavlink/start](http://qgroundcontrol.org/mavlink/start)
3. **^** ["Serialization · MAVLink Developer Guide"](https://mavlink.io/en/guide/serialization.html). *mavlink.io*. Retrieved 2019-08-22.
4. **^** [http://qgroundcontrol.org/mavlink/crc_extra_calculation](http://qgroundcontrol.org/mavlink/crc_extra_calculation)
5. **^** ["GitHub - ArduPilot/pymavlink: python MAVLink interface and utilities"](https://github.com/ArduPilot/pymavlink). August 18, 2019 – via GitHub.
6. **^** ["GitHub - arthurbenemann/droidplanner: Ground Control Station for Android Devices"](https://github.com/arthurbenemann/droidplanner). July 2, 2019 – via GitHub.
7. **^** ["A Java code generator and a Java library for MAVLink: ghelle/MAVLinkJava"](https://github.com/ghelle/MAVLinkJava). August 4, 2019 – via GitHub.
8.  ["GitHub - dronefleet/mavlink: A Java API for MAVLink communication"](https://github.com/dronefleet/mavlink). August 2, 2019 – via GitHub.
9. **^** ["GitHub - mavlink/mavlink: Marshalling / communication library for drones"](https://github.com/mavlink/mavlink). August 20, 2019 – via GitHub.
10. **^**[http://qgroundcontrol.org/mavlink/crc_extra_calculation#field_reordering](http://qgroundcontrol.org/mavlink/crc_extra_calculation#field_reordering)
11. **^** Posted by Shyam Balasubramanian on November 15, 2013 at 2:36pm in ArduCopter User Group; Discussions, Back to ArduCopter User Group. ["MAVLink Tutorial for Absolute Dummies (Part –I)"](https://diydrones.com/forum/topics/mavlink-tutorial-for-absolute-dummies-part-i?groupUrl=arducopterusergroup). *diydrones.com*.