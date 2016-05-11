# LoRaWAN 1.0 协议笔记

标签（空格分隔）： 协议 

---

##**终端设备需要遵守的约定**：
- 终端设备在每一次数据传输中通过伪随机数的方式来改变信道。这个得到的通信频率使系统在干扰的情况下更稳定。
- 终端设备遵循在当地法规允许的子频段中最大传输占空比。
- 终端设备遵循在当地法规允许的子频段中最大传输持续时间或停滞时间。

----------
##**LoRaWAN Classes**：
- **Class A** 双向终端设备：支持Class A的终端设备在双向传输中每一次终端设备uplink传输后会有两个短的downlink接收窗口。这个传输时间间隙通过终端设备基于自身传输需要一个随机时间基础上微小的变化确定(**ALOHA**型协议)。Class A是为超低功耗终端设备系统只需要终端发送了一次uplink传输后服务器马上downlink传输的应用。来自服务器的任何downlink通信必须等到下一次计划的uplink。
- **Class B** 绑定接收时隙的双向终端设备：支持 Class B的终端设备拥有更多的接收时隙。在Class A随机接收窗口上增加，Class B设备在计划时间里打开额外的接收窗口。来自gateway的指令让终端设备在计划时间里打开接收窗口接收时间同步信标(time synchronized Beacon)。这样能让服务器知道终端设备在监听中。
- **Class C** 拥有最大接收时隙的双向终端设备：支持Class C的终端设备有一个几乎一直开启接收的窗口，只有传输中的时候才关闭。对比Class A或Class B终端设备将使用更多的电量来运行Class C，但是服务器到终端设备的通信Class C提供最低的延迟。

**更高的类别支持低类别的所有功能。所有支持LoRaWAN的终端设备都必须支持Class A**

----------

##**Physical Message Formats**
-**Uplink Messages**： 终端设备->网关(1 or More)->服务器
*Uplink PHY structure:*

|Preamble|PHDR|PHDR_CRC|PHYPayload|CRC|
|:-:|:-:|:-:|:-:|:-:|

-**Downlink Message**: 服务器->网关->终端
*Downlink PHY structure:*

|Preamble|PHDR|PHDR_CRC|PHYPayload|
|:-:|:-:|:-:|:-:|

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
|:-:|:-:|:-:|:-:|


----------
###**MAC Layer(PHYPayload)**

|Size(byte)|1|1..M|4|
|:--:|:-:|:-:|:-:|
|**PHPayload**|MHDR|MACPayload|MIC|

M的最大值根据具体命令来决定。

###MAC Header(MHDR field)

|Bit#|7..5|4..2|1..0|
|:-:|:-:|:-:|:-:|
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
**FHDR**包含一个终端设备的4字节设备地址(**DevAddr**)，一个字节的帧控制(**FCtrl**)，一个两字节的帧计数(**FCnt**)，和最多15个字节的帧配置字段(**FOpts**)用来传输MAC命令。

|Size(bytes)|4|1|2|0..15|
|:-:|:-:|:-:|:-:|:-:|
|**FHDR**|DevAddr|FCtrl|FCnt|FOpts|

downlink 帧时FCtrl在帧头的内容为：

|Bit#|7|6|5|4|[3..0]|
|:-:|:-:|:-:|:-:|:-:|:-:|
|**FCtrl bits**|ADR|ADRACKReq|ACK|FPending|FOptsLen|

uplink 帧时FCtrl在帧头的内容为：

|Bit#|7|6|5|4|[3..0]|
|:-:|:-:|:-:|:-:|:-:|:-:|
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
每个终端设备有两个帧计数器，一个用来记录发送到网络服务器的uplink(FCntUp),由终端增加FCntUp计数；另一个用来记录网络服务器发送给终端设备的downlink(FCntDown), 由网络服务器增加FCntDown计数。 网络服务器记录uplink帧计数并且生成downlink计数分别给每一个终端设备。入网激活成功后，终端设备的帧计数和网络服务器对应该终端设备的帧计数都将清零。之后FCntUp和FCntDown在数据帧发送的每个方向上以1每次递增。接收方这边相应的计数值与接收到的值同步，接收到的值是递增当前值并且

**帧配置(FOptsLen in FCtrl, FOpts)**

- FOpts的长度有FOptsLen决定。如果FOptsLen为0则FOpts字段将不存在，如果FOptsLen不为0，如果MAC命在FOpts字段，port 0在这时不能使用(这时的port必须是不为0或没有)。
- MAC命令不能同时在payload字段和帧配置字段。

**端口字段(FPort)**

- 如果帧payload字段不为空，则端口字段必须存在。如果**FPort**值为0说明**FRMPayload**只包含MAC命令；**FPort**值为1..223(0x01..0xDF)时表示特殊应用。**FPort**值224..255(0xE0..0xFF)是为以后标准化应用扩展而保留的。

|Size(bytes)|7..23|0..1|0..N|
|:-:|:-:|:-:|:-:|
|MACPayload|FHDR|FPort|FRMPayload|

N必须小于或等于：N ≤ M - 1 - (**FHDR**长度) 这里的M是MAC payload的最大长度。单位：字节。

**MAC帧Payload加密(FRMPayload)**

- 如果数据帧携带有payload，**FRMPayload**必须在消息完整性代码(**MIC**)计算之前进行加密。
- 加密方案使用的是IEEE 802.15.4/2006 Annex B[IEEE802154] 128位key的AES加密。
- 默认是所有的FPort都由LoRaWAN层进行加密/解密。FPort为0时除外。


----------


##**MAC Commands**

- 一个单一数据的帧能包含MAC命令的任何序列，或者捎带在**FOpts**字段里，当发送一个数据在**FRMPayload**字段的帧时，**FPort**字段要设置为0。发送数据**FOpts**字段带有MAC命令时，必须不加密和不超过15个字节长度。MCA命令在**FRMPayload**字段时，必须加密和不能超过**FRMPayload**字段最大限制长度。
- MAC指令为了不被监听者破解，必须通过在一个单独的数据帧里的**FRMPayload**字段来发送MAC指令。

MAC 命令表 **CID**看后八位。

|CID|Command|End-device|Gateway|Short Description|
|:-:|:-:|:-:|:-:|:-:|
|0x02|**LinkCheckReq**|*| |终端设备验证是否连接到一个网络上。|
|0x02|**LinkCheckAns**| |*|应答 LinkCheckReq 命令。|
|0x03|**LinkADDRReq**| |*|请求改变终端设备的数据速率、发射功率、重复率或信道。|
|0x03|**LinkADDRAns**|*| |应答 LinkADDRReq 命令。|
|0x04|**DutyCycleReq**| |*|设置一个设备的最大总传输占空比。|
|0x04|**DutyCycleAns**|*| |应答 DutyCycleReq 命令。|
|0x05|**RXParamSetupReq**| |*|设置接收时隙参数。|
|0x05|**RXParamSetupAns**|*| |应答 RXParameSetupReq 命令。|
|0x06|**DevStatusReq**| |*|请求终端设备状态。|
|0x06|**DevStatusAns**|*| |返回终端设备的电池电量和Demodulation margin(SNR)。|
|0x07|**NewChannelReq**| |*|创建或修改一个Radio Channel的定义。|
|0x07|**NewChannelAns**|*| |应答 NewChannelReq 命令。|
|0x08|**RXTimingSetupReq**| |*|设置接收时隙的定时时间(Timing)？|
|0x08|**RXTimingSetupAns**|*| |应答 RXTimingSetupReq 命令。|
|0x80~0xFF|所有的(Proprietary)|\*|*|保留用于专有网络命令扩展。|


- 任何服务器调整的值直到终端设备下一次Join时才会失效。因此，在每次成功的Join，设备都是使用默认参数，是否需要服务器重新调整参数，取决于需求。

###**Link Check commands**(LinkCheckReq, LinkCheckAns)
- ***LinkCheckReq*** 命令是终端设备用来验证是否连接上网，该命令没有payload。
- 当 ***LinkCheckReq*** 命令被网络服务器收到，将通过一个或多个网关来响应一条 ***LinkCheckAns***命令。

|Size(bytes)|1|1|
|:-:|:-:|:-:|
|LinkCheckAns Payload|Margin|GwCnt|


- **Margin**(demodulation margin)为8位无符号整型，取值范围0～254，用来说明最后成功收到的 ***LinkCheckReq*** 命令的link margin 值，单位dB(幅度)。
- **GwCnt** 是表示成功接收到最后的 ***LinkCheckReq*** 命令的网关数量。
###**Link ADR command**(LinkADRReq, LinkADRAns)
- ***LinkADRReq*** 命令是网络服务器请求一个终端设备执行速率适配。

|Size(bytes)|1|2|1|
|:-:|:-:|:-:|:-:|
|LinkAddrReq Payload|DataRate_TXPower|ChMask|Redundancy|

|Bits|[7:4]|[3:0]|
|:-:|:-:|:-:|
|DataRate_TXPower|DataRate|TXPower|


- **ChMask**(channel mask)用于uplink接入的channel。
- Channel state table:

|Bit#|Usable channels|
|:-:|:-:|
|0|Channel 1|
|1|Channel 2|
|2|Channel 3|
|...|...|
|15|Channel 16|


- 在**ChMask**字段的相应位设置为1表示相应的信道可以用于uplink传输。

|Bits|7|[6:4]|[3:0]|
|:-:|:-:|:-:|:-:|
|Redundancy bits|RFU|ChMaskCntl|NbRep|


- **NbRep** 表示每一次uplink消息重复次数。仅用于 unconfirmed uplink frames。默认值是1, 有效范围1～15，如果**NbRep**== 0，终端设备应该使用默认值1。每一次重复都是在接收窗口过期后，跳频也会在这期间进行。
- The channel mask control(**ChMaskCntl**)field controls the interpretation of the previously defined **Chmask** bit mask. This field will only be non-zero values in networks where more than 16 channels are implemented. It controls the block of 16 channels to which the **ChMask** applies. It can also be used to globally turn on or off all channels using specific modulation. This field usage is region specific and is defined in Chapter 7.
- 信道的频率是区域专用的在第6章中定义它们。一个终端设备用***LinkADRAns***命令来应答***LinkADRReq***命令。

|Size(bytes)|1|
|:-:|:-:|
|LinkADRAns Payload|Status|

|Bits|[7:3]|2|1|0|
|:-:|:-:|:-:|:-:|:-:|
|Status bits|RFU|Power ACK|Data rate ACK|Channel mask ACK|


- ***LinkADRAns Status*** bits具有以下含义：

| Field |Bit = 0|Bit = 1|
|:-:|:-:|:-:|
|Channel mask ACK|发送的Channel mask使能一个尚未定义的channel，该命令被丢弃并且终端设备的状态没有改变。|发送的Channel mask成功解析，当前所有定义的channel状态根据channel mask来设置。|
|Data rate ACK|请求的数据率是终端设备未知的或任何使能的channel不被支持。该命令被丢弃并且终端设备的状态没有改变。|数据率成功设置。|
|Power ACK|请求的功率级别在终端设备中没有实现。该命令被丢弃并且终端设备的状态没有改变。|功率级别成功设置。|


- 如果这3个字段中的任何一个为0，则该命令没有设置成功，节点一直保持以前的状态。
###**End-Device Transmit Duty Cycle**(DutyCycleReq, DutyCycleAns)
- 该DutyCycleReq命令通过网络协调器来限制一个终端设备的最大总发射占空比。总发射占空比对应于发射占空比所有子频段。

|Size(bytes)|1|
|:-:|:-:|
|DutyCycleReq Payload|MaxDCycle|

允许的最大终端设备发送占空比:  aggregated duty cycle = $\frac{1}{2^{MaxDCycle}}$  

- **MaxDutyCycle**的有效值是[0：15]。除了为0则占空比没有限制，其他则由区域设置调节(Regional regulation)。
- 值255意味着终端设备应立即变得沉默(silent)。相等于远程切断终端设备。
- 终端设备使用 ***DutyCycleAns*** 来应答 ***DutyCycleReq*** 命令。***DutyCycleAns*** 不含任何Payload。

###**Receive Window Parameters**(RXParamSetupReq, RXParamSetupAns)
- ***RXParamSetupReq***命令允许修改每一个uplink之后的第二接收窗口(RX2)的频率和数据率。该命令还允许uplink和RX1时隙downlink数据速率之间的offset修改

|Size(bytes)|1|3|
|:-:|:-:|:-:|
|RX2SetupReq Payload|DLsettings|Frequency|

|Bits|7|6:4|3:0|
|:-:|:-:|:-:|:-:|
|DLsettings|RFU|RX1DRoffset|RX2DataRate|


- Rx1DRoffset字段设置的offset在uplink数据率和RX1时隙downlink数据速率之间偏移。作为默认此偏移为0。该偏移是用于考虑到最大功率密度约束在某些区域的基站和以平衡上行链路和下行链路的无线电链路余量。
- 数据速率（RX2DataRate）字段定义了使用第二接收窗为LinkADRReq命令相同的约定下列下行链路的数据速率（例如：0意味着，DR0/125kHz）。频率(**Frequency**)字段用于第二接收窗口的信道频率，因此这个频率是在***NewChannelReq***命令定义的约定下。
- ***RXParamSetupAns***命令是用来确认***RXParamSetupReq***命令的接收的。payload只有一个字节的status：

|Size(bytes)|1|
|:-:|:-:|
|RX2SetupAns Payload|Status|

|Bits|7:3|2|1|0|
|:-:|:-:|:-:|:-:|:-:|
|Status bits|RFU|RX1DRoffset ACK|RX2 Data rate ACK|Channel ACK|

|Field|Bit = 0|Bit = 1|
|:-:|:-:|:-:|
|Channel ACK|请求的频率终端设备不可用。|RX2 时隙信道设置成功。|
|RX2 Data rate ACK|请求的数据速率终端设备未知。|RX2 时隙数据速率设置成功。|
|RX1DRoffset ACK|uplink/downlink数据速率偏移不在RX1时隙允许的范围里。|RX1DRoffset设置成功。|


- 如果这3个字段中的任何一个为0，则该命令没有设置成功，保持之前的参数。

###**End-Device Status**(DevStatusReq, DevStatusAns)
- 通过***DevStatusReq***命令向一个终端设备请求状态信息。该命令没有Payload。如果***DevStatusReq***命令的接收者是一个终端设备，终端设备需用***DevStatusAns***命令来响应。

|Size(bytes)|1|1|
|:-:|:-:|:-:|
|DevStatusAns Payload|Battery|Margin|

电池电量等级编码如下：

|Battery|Description|
|:-:|:-:|
|0|终端设备连技到外部电源。|
|1..254|电量等级，1处于最低电量，254处于最高电量。|
|255|终端设备无法测量电量等级|


- **Margin**是最后成功收到***DevStatusReq***命令经过四舍五入到最接近整数值的解调信噪比(SNR)。该值取有符号整型的6位来表示，最小值为-32，最大值为31.

|Bits|7:6|5:0|
|:-:|:-:|:-:|
|Status|RFU|Margin|


###**Creation / Modification of a Channel**(NewChannelReq, NewChannelAns)
- ***NewChannelReq*** 命令可以用于修改现有信道的参数或者创建一个新的。该命令用于设置这个频道上可用的新的信道的中心频率和数据率的范围：

|Size(bytes)|1|3|1|
|:-:|:-:|:-:|:-:|
|NewChannelReq Payload|ChIndex|Freq|DrRange|


- 信道索引(**ChIndex**)是正在创建或修改的信道的索引值。根据所使用的区域和频带，该LoRaWAN规范规定默认频道必须是共同的所有的设备和不能被 ***NewChannelReq*** 命令（参照第6章）进行修改。如果缺省信道的数目是N，则缺省信道从0到N-1，并且 **ChIndex** 合适的范围是N至15.一种设备必须能够处理至少16个不同的信道的定义。在某些区域中的设备可具有存储16个以上的信道的定义。
- 频率(**Freq**)字段是一个24位无符号整数。这个实际信道频率为100 × **Freq** Hz，由此表示低于100 MHz的频率值被保留供将来使用。这使得在任何信道设置频率在100MHz至1.67GHz之间每100Hz为一个步进。信道的 **Freq** 值不能为0。终端设备能否检查它的实际频率由无线硬件决定否则将返回一个错误。
- 数据速率范围(**DrRange**)字段指定此通道允许的数据速率范围。该字段由两个4位的索引组成：

|Bits|7:4|3:0|
|:-:|:-:|:-:|
|DrRange|MaxDR|MinDR|


- 继第5.2节的最低数据速率(**MinDR**)子定义的约定指定了该信道上的最低数据速率。举个例子：0 指定 DR0 / 125kHz。类似地，最大数据速率(**MaxDR**)指定了最高数据速率。举个例子：DrRange = 0x77 意味着信道上只有50kbps的GFSK是允许的，DrRange = 0x50 意味着能支持的数据速率范围是 DR0 / 125kHz ~ DR5 / 125kHz。
- 新定义的信道被使能，则可以立即使用该信道。
- 终端设备通过发送回一个 ***NewChannelAns*** 命令确认***NewChannelAns*** 命令的接收。此消息的有效载荷包含以下信息：

|Size(bytes)|1|
|:-:|:-:|
|NewChannelAns Payload|Status|

|Bits|7:2|1|0|
|:-:|:-:|:-:|:-:|
|Status|RFU|Data rate range ok|Channel frequency ok|

|Field|Bit = 0|Bit = 1|
|:-:|:-:|:-:|
|Data rate range ok|指定的数据速率范围超出目前为此设备定义的那些。|数据速率的范围可能与终端设备的兼容。|
|Channel frequency ok|这个频率设备无法使用。|这个频率设备能够使用。|


- 如果这2个位中的任何一个为0，则该命令没有成功，新的信道没有创建。

###**Setting delay between TX and RX**(RXTimingSetupReq, RXTimingSetupAns)
- ***RXTimingSetupReq*** 命令允许配置 TX uplink 和 打开第一接收窗口时隙间的延迟(Delay)。在第一接收窗口时隙打开一秒之后第二接收窗口时隙才打开。

|Size(bytes)|1|
|:-:|:-:|
|RXTimingSetupReq Payload|Settings|


- 延迟(**Delay**)字段指定延迟时间。该字段被分割在两个4位的索引：

|Bits|7:4|3:0|
|:-:|:-:|:-:|
|Settings|RFU|Del|


- 延迟(Delay)以秒为单位。**Del** 为 0 对应为 1 秒。

|Del|Delay[s]|
|:-:|:-:|
|0|1|
|1|1|
|2|2|
|3|3|
|...|...|
|15|15|


- 一个终端设备用 ***RXTimingSetupAns*** 来应答 ***RXTimingSetupReq*** 命令,***RXTimingSetupAns***命令没有payload。


----------

##**End-Device Activation**

- 要加入到 LoRaWAN 网络中来，每一个终端设备必须 personalized(个性化)和激活(activated)。
- 一个终端设备的激活可以通过两种方式来实现，无论是当一个终端设备被部署或复位后通过 **Over-The-Air Activation** （OTAA），或通过 **Activation By Personalization** (激活由个性化)（ABP），其中终端设备的个性化和激活的两个步骤为一个步骤完成。

###**Data Stored in the End-device after Activation**
- 激活后，将下面的信息存储在终端设备里：一个设备地址(**DevAddr**)，一个应用标识符(**AppEUI**)，一个网络会话密钥(**NwkSKey**)，和一个应用会话密钥(**AppSKey**)。

###**End-device address(DevAddr)**
- 当前网络内的终端设备的32位标识码就是设备的 **DevAddr** 。格式如下：

|Bit#|31:25|24:0|
|:-:|:-:|:-:|
|DevAddr bits|NwkID|NwkAddr|


- 最有用的7位是网络标识符(**NwkID**)用于区分不同地区范围的网络运营商的网络地址，以纠正漫游的问题。另外25位为终端设备的网络地址(**NwkAddr**)，可以由任意的网络管理器分配。

###**Application identifier(AppEUI)**
- **AppEUI** 是在IEEE EUI64地址范围唯一标识终端设备的应用程序提供商的全球应用ID。
- 在激活过程之前这个 **AppEUI** 就存储在终端设备中了。

###**Network session key(NwkSKey)**
- **NwkSKey** 是一个 网络会话密钥，指定用于终端设备。它是用于网络服务器和终端设备来计算和验证所有数据消息的 **MIC**（消息完整性代码），以确保数据的完整性。它还用于加密和解密仅数据消息的MAC payload。

###**Application session key(AppSKey)**
- **AppSKey** 是一个应用会话密钥，指定用于终端设备。它用于由网络服务器和终端设备都进行加密和解密应用程序特定数据的消息的payload字段。它还用来计算和验证可以包括在应用程序特定数据的消息的Payload的应用程序级的MIC。

###**Over-the-Air Activation**
- 用于空中激活，终端设备必须遵循Join步骤才能与网络服务器进行数据交换。每当终端设备丢失了网络连接，必须重新Join网络。
- 在Join之前以下信息需要设置：全球唯一终端设备标识符(**DevEUI**)，程序标识符(**AppEUI**)，一个AES-128密钥(**AppKey**)。
- 使用空中激活时，终端设备不用设置任何形式的网络密钥。相反，当一个终端设备加入网络，网络会话密钥将用于加密和验证网络级的传输。这种方式便于不同的供应商的网络之间的终端设备漫游。同时使用网络会话密钥和应用会话密钥还允许在其中应用的数据不能由网络提供商来读取或篡改联合网络服务器。

###**End-device identifier(DevEUI)**
- **DevEUI** 是终端设备在IEEE EUI64地址空间的全球终端设备ID唯一标识。

###**Application key(AppKey)**
- **AppKey** 用于生成 NwkSKey 和 AppSKey。

###**Join procedure**
- 对于终端设备，Join 过程包括 **Join request** 和 **Join accept** 两个过程与服务器的MAC消息交互。

###**Join-request message**
- Join 过程总是从终端设备发起join-request请求开始。

|Size(bytes)|8|8|2|
|:-:|:-:|:-:|:-:|
|Join Request|AppEUI|DevEUI|DevNonce|


- **DevNonce** 是一个随机值。对于每一个终端设备，网络服务器跟踪终端设备过去使用的一定数量的**DevNonce** 和忽略来自终端设备join请求时的**DevNonce**。
- 这种使用随机数的机制能防止重复发送被记录的join请求消息的攻击。
- Join消息的**MIC**计算方法：
*cmac* = aes128_cmac(AppKey, MHDR|AppEUI|DevEUI|DevNonce)
MIC = *cmac*[0..3]
- **Join-request消息没有加密**。

###**Join-accept message**
- 如果**Join-request**被允许，将发送**Join-accept**来应答。Join-accept消息像普通的downlink消息一样但是使用的延迟是JOIN_ACCEPT_DELAY1 和 JOIN_ACCEPT_DELAY2。Join-accept使用的频率和数据速率和RX1、RX2的相同。
- 如果Join-request没有响应，表明不允许加入该网络。

|Size(bytes)|3|3|4|1|1|(16)Optional|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|Join Accept|AppNonce|NetID|DevAddr|DLSettings|RxDelay|CFList|


- **AppNonce** 是一个随机数或由网络服务器提供的唯一ID，和用于终端设备来生成**NwkSKey** 和 **AppSKey** :
NwkSKey = aes128_encrypt(AppKey, 0x01|AppNonce|NetID|DevNonce|$pad_{16}$)
AppSKey = aes128_encrypt(AppKey, 0x02|AppNonce|NetID|DevNonce|$pad_{16}$)
- MIC计算：
*cmac* = aes128_cmac(AppKey, MHDR|AppNonce|NetID|DevAddr|RFU|RxDelay|CFList)
MIC = *cmac*[0..3]
- Join-accept 消息使用**AppKey**加密：
aes128_decrypt(AppKey, AppNonce|NetID|DevAddr|RFU|RxDelay|CFList|MIC)
- 网络服务器使用ECB模式的AES解密操作来加密Join-accept消息，所以终端设备可以使用AES加密运算来解密该消息。这样终端设备只需要实现AES加密而不不需要实现AES解密。
- 使用这两个会话密钥，使得网络运营商无法窃取应用数据，这样的话，在网络上应用程序提供者必须支持终端设备在加入网络的过程中创建用于终端设备的NwkSKey。与此同时应用提供者承诺给网络运营商因终端设备产生的任何流量费用和保留用于保护其应用数据的AppSKey的完全控制。
- **NetID**的格式如下：七位LSB为NwkID，七位MSB为用于终端设备的短地址。相邻或重叠的网络必须有不同的**NwkID**。剩下的17个MSB由网络运营商自由选择。原文： the format of the NetID is as follows: The seven LSB of the NetID are called NwkID and match the seven MSB of the short address of an end-device as described before. Neighboring or overlapping networks must have different NwkIDs. the remaining 17 MSB can be freely chosen by the network operator.
- DLsetting字段包含downlink配置：

|Bits|7|6:4|3:0|
|:-:|:-:|:-:|:-:|
|DLsettings|RFU|RX1DRoffset|RX2 Data rate|


- RX1DRoffset 字段设置在终端设备的第一接收时隙通讯时的uplink数据速率和downlink数据速率之间offset。默认情况该offset为0。downlink数据速率总是低于或等于uplink数据速率。这个offset用于约束某些区域的基站最大功率密度和平衡uplink与downlink的radio link margins（余量）。
- uplink和downlink数据速率之间的实际关系和具体区域有关。
- **RxDelay** 延迟遵循**RXTimingSetupReq**命令中的**Delay**字段一样的延迟约定。

###**Activation by Personalization**
- 某些情况下，终端设备可以由Personalization（个性化）激活。终端设备通过Personalization绕过 **Join-request Join-accept** 过程来加入特定的网络。
- 使用Personalization激活一个终端设备，意味着 **DevAddr**、**NwkSKey**和**AppSKey**直接存储在终端设备里替换OTA激活的**DevEUI**、**AppEUI**和**AppKey**。终端设备配置了开始加入一个特定的LoRa网络时所需的信息。
- 每一个设备必须有唯一的NwkSKey和AppSKey。这样一个设备的密钥被破解了也不会造成其他设备的安全性危险。创建密钥过程不应该被公开(比如节点地址)。

##**Physical Layer**
