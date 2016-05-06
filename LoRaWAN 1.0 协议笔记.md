# LoRaWAN 1.0 协议笔记

标签（空格分隔）： 协议 

---

##终端设备需要遵守的约定：
- 终端设备在每一次数据传输中通过伪随机数的方式来改变信道。这个得到的通信频率使系统在干扰的情况下更稳定。
- 终端设备遵循在当地法规允许的子频段中最大传输占空比。
- 终端设备遵循在当地法规允许的子频段中最大传输持续时间或停滞时间。

----------
##LoRaWAN Classes：
- **Class A** 双向终端设备：支持Class A的终端设备在双向传输中每一次终端设备uplink传输后会有两个短的downlink接收窗口。这个传输时间间隙通过终端设备基于自身传输需要一个随机时间基础上微小的变化确定(**ALOHA**型协议)。Class A是为超低功耗终端设备系统只需要终端发送了一次uplink传输后服务器马上downlink传输的应用。来自服务器的任何downlink通信必须等到下一次计划的uplink。
- **Class B** 绑定接收时隙的双向终端设备：支持 Class B的终端设备拥有更多的接收时隙。在Class A随机接收窗口上增加，Class B设备在计划时间里打开额外的接收窗口。来自gateway的指令让终端设备在计划时间里打开接收窗口接收时间同步信标(time synchronized Beacon)。这样能让服务器知道终端设备在监听中。
- **Class C** 拥有最大接收时隙的双向终端设备：支持Class C的终端设备有一个几乎一直开启接收的窗口，只有传输中的时候才关闭。对比Class A或Class B终端设备将使用更多的电量来运行Class C，但是服务器到终端设备的通信Class C提供最低的延迟。

**更高的类别支持低类别的所有功能。所有支持LoRaWAN的终端设备都必须支持Class A**
----------
##Physical Message Formats
-**Uplink Messages**： 终端设备->网关(1 or More)->服务器
*Uplink PHY structure:*
|Preamble|PHDR|PHDR_CRC|PHYPayload|CRC|
|--|

-**Downlink Message**: 服务器->网关->终端
*Downlink PHY structure:*
|Preamble|PHDR|PHDR_CRC|PHYPayload|
|--|

##Receive Window
###每一个uplink传输终端设备会打开两个短的接收窗口。


----------
**PHYPayload:**
|MHDR|MACPayload|MIC|
|:-:|:-:|:-:|

**MACPayload:**
|FHDR|FPort|FRMPayload|
|:-:|:-:|:-:|

**FHDR:**
|DevAddr|FCtrl|FCnt|FOpts|
|:-:|:-:|:-:|


----------
### MAC Layer(PHYPayload)

|Size(byte)|1|1..M|4|
|-:|:-:|:-:|:-:|
|**PHPayload**|MHDR|MACPayload|MIC|

M的最大值根据具体命令来决定。

###MAC Header(MHDR field)

|Bit#|7..5|4..2|1..0|
|-:|:-:|:-:|:-:|
|**MHDR bits**|MType|RFU|Major|

Message type(MType bit filed)
|MType|Description|
|:-:|:-:|
|000|Join Request|
|001|Join Accept|
|010|Unconfirmed Data Up|
|011|Unconfirmed Data Down|
|100|Confirmed Data Up|
|101|Confirmed Data Down|
|110|RFU|
|111|Proprietary|


- Join-request and join-accept message
这两个消息类型用于over-the-air activation(空中激活)
- Data message
Data message用来传输MAC命令和应用数据，两种数据可以在一个Message中组合传输。confirmed-data message需要接收确认，而unconfirmed-data message不需要确认。Proprietary messages用来发送不标准的消息格式，不能与标准消息格式之间相互通信，但是能用于拥有专有扩展(Proprietary extensions)的设备之间进行通信。

Major version of data message(Major bit field)
|Major bits|Description|
|:-:|:-:|
|00|LoRaWAN R1|
|01..11|RFU|

###MAC Payload of Data Messages(MACPayload)
包含一个帧头(**FHDR**)其次有一个可选的Port字段(**FPort**)和可选的帧Payload字段(**FRMPayload**)。

- **Frame header(FHDR)**
**FHDR**包含一个终端设备的4字节设备地址(**DevAddr**)，一个字节的帧控制(**FCtrl**)，一个两字节的帧计数(**FCnt**)，和最多15个字节的帧可选字段(**FOpts**)用来传输MAC命令。
|Size(bytes)|4|1|2|0..15|
|-:|:-:|:-:|:-:|:-:|
|**FHDR**|DevAddr|FCtrl|FCnt|FOpts|

downlink 帧时FCtrl在帧头的内容为：
|Bit#|7|6|5|4|[3..0]|
|-:|:-:|:-:|:-:|:-:|:-:|
|**FCtrl bits**|ADR|ADRACKReq|ACK|FPending|FOptsLen|

uplink 帧时FCtrl在帧头的内容为：
|Bit#|7|6|5|4|[3..0]|
|-:|:-:|:-:|:-:|:-:|:-:|
|**FCtrl bits**|ADR|ADRACKReq|ACK|RFU|FOptsLen|

**自适应数据速率(ADR)控制在帧头(ADR, ADRACKReq in FCtrl)**

- LoRa网络允许终端设备单独使用任何可能的数据传输速率。这个特性用来让LoRaWAN为静止的终端设备的数据速率能够适应和优化。简称自适应数据速率，当这个特性使能时网络能优化可能使用的最高的数据率。
- 用ADR来管理移动的终端设备的数据率是不实用的，在快速变化的无线环境下移动的终端设备应使用其固定的缺省数据率。
- 如果ADR位被置1，网络将使用适当的MAC命令来控制终端设备的速据率。如果ADR位没有被置1，网络将无论接收信号强度如何也不尝试控制终端设备的速据率。ADR有助于延长终端设备的电池寿命和最大化网络容量。
- 即使是移动的终端设备也可以根据流动性的状态来使用ADR优化其数据率。
- 如果一个终端设备谁的数据速率是由网络优化使用一个比其默认的数据速率高的数据速率，它定期需要验证因为网络仍然接收uplink帧。每一次uplink帧计数递增(仅新的uplink帧，重复传输不会计数递增)，这个设备增加一个**ADR_ACK_CNT**计数。ADR_ACK_LIMIT uplinks(ADR_ACK_CNT >= ADR_ACK_LIMIT)后不会任何downlink响应，设置ADR应答请求位(**ADRACKReq**)。网络需要在ADR_ACK_DELAY时间内响应一个downlink帧，任何收到downlink帧后一个uplink帧重置这个ADR_ACK_CNT计数。这个downlink ACK位并不需要被置1，如在终端设备接收时隙期间的任何响应，说明网关仍然接收到来自这个设备的uplink帧。uplink帧在下一个ADR_ACK_DELAY如果没有收到应答(i.e., ADR_ACK_LIMIT + ADR_ACK_DELAY的总和之后)，终端设备可能尝试通过切换到下一个较低数据率来重新连接因为低数据率提供了一个较长的无线电范围。终端设备在每一次ADR_ACK_LIMIT时间到达时将进一步降低其数据速率。如果终端设备使用默认的数据速率这个**ADRACKReq**不能置1，因为在这种情况下没有改善连接范围的作用。

- 不要求一个ADR确认请求立即回复，为网络以最佳调度其downlink提供了灵活性。
- 在uplink传输如果ADR_ACK_CNT >= ADR_ACK_LIMIT 并且当前的数据速率大于设备定义的最小数据速率**ADRACKReq**位被置1，在其它条件下被置0。

**消息应答位和应答过程(ACK in FCtrl)**

- 当收到一个确认数据消息，这个接收方将应答一个数据帧，数据帧的应答位(**ACK**)置1。如果发送方是一个终端设备，终端设备发送操作之后打开一个接收窗口来接收网络将发送的应答。如果发送方是网关，终端设备发送一个自行决定的应答。
- 应答仅在接收到的最新消息中发送，并且不重发。
- 为了让终端设备尽可能有简单和少的状态，在接收到一个需要确认的数据消息后可能需要立即发送一个明确的应答数据消息(可能为空的数据消息)。另外终端设备可能推迟到发送下一个数据消息才带上这个应答消息。

**重传过程**
The number of retransmissions (and their timing) for the same message where an acknowlegment is requested but not received is at the discretion of the end-device and may be different for each end-device, it can also be set or adjusted from the network server.
重传的次数(和重传时间)可以设置和网络服务器来调整。对于相同的请求应答消息

如果终端设备的重传次数达到最大值还没有接收到应答，可以降低数据速率继续尝试。终端设备可以继续重传消息或者放弃这个消息并继续。

如果网络服务器的重传次数达到最大值还没有收到应答，通常会认为这个终端设备无法访问，直到在重新收到来自这个终端的消息。如果终端设备再次连接上网络服务器会重新发一次消息或者放弃这个消息并继续。

**帧填挂起位(FPending in FCtrl, downlink only)**
帧挂起位仅在downlink通讯中使用，说明网关有更多的数据等待被发送并且因此询问这个终端设备是否能发送另一个uplink消息尽可能快的打开另一个接收窗口。

**帧计数(FCnt)**
每个终端设备有两个帧计数器，一个用来记录发送到网络服务器的uplink(FCntUp),由终端增加FCntUp计数；另一个用来记录网络服务器发送给终端设备的downlink(FCntDown), 由网络服务器增加FCntDown计数。 网络服务器记录uplink帧计数并且生成downlink计数分别给每一个终端设备。入网激活成功后，终端设备的帧计数和网络服务器对应该终端设备的帧计数都将清零。之后FCntUp和FCntDown在数据帧发送的每个方向上以1每次递增。
