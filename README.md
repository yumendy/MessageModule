# 基于socket通信的消息通讯模块

## 消息机制设计（POST，GETDATA，DATA，ACK）

本项目为保证数据安全及可靠性，采用消息队列机制，对有实际载荷的数据包进行短时间缓存。
数据包中带有请求，请求分为4个类型：

- 控制命令类
	- POST：用于告知携带有准备发送的数据包信息摘要（数据包hex(guid)、时间戳）
	- GETDATA: 用于请求数据包（数据包hex(guid)、时间戳）
	- ACK: 告知已经成功接收的数据包（数据包hex(guid)、时间戳）
	```json

	{
		"guid": "cfa2832078304d81a478dfdf23cb4b57",
		"timestamp": 1448998980,
	}

	```
- 实际数据类
	- DATA：拥有实际载荷的数据包（实际数据）
	json示例在接口及功能设计模块给出


## 通信部分架构设计

通信部分架构为:

- Triple Queues:
	- Caching Queue: 
		装载对象：数据包对象， 用于缓存有实际载荷的数据包以备发送。
	- Sending Queue:
		装载对象：元组，（接收地址，数据包），用于存储待发送的数据包
	- Receiving Queue:
		装载对象：元组，（发送地址，数据包），用于存储已接受的数据包
- Double Controllers:
	- Sender:
		轮询Sending Queue，按序将数据包发送到指定地址的2333端口。
	- Receiver:
		监听2333端口，接收数据包，并将数据包组为（发送地址，数据包）元组形式，放置与接收队列
- Single Worker:
	- Worker:
		轮询Receiving Queue，按序处理数据包，根据数据包请求对消息模块进行操作，对DATA类型的数据包进行分析并合理调用其他接口完成命令，包括一个可供上层调用的发送函数，接受参数为接收地址和json，完成组DATA包放入Caching Queue，并组装POST包放入Sending Queue。

## 协议设计（数据包规范）
|内容|长度|说明|
|----|----|----|
|Version|4-Byte|协议版本号，用于确定应用，默认为0x0209|
|Command|8-Byte|请求类型|
|Length|4-Byte|payload长度，小尾序|
|Guid(uuid)|16-Byte|数据包唯一标识符|
|Checksum|8-Byte|SHA256(AES(json))|
|Payload|n/a|AES(json)|