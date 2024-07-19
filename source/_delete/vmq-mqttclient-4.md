---
title: vmq-从零开始的MQTT客户端-4-MQTT包
url: vmq-mqttclient-4
date: 2024-03-21 18:56:06
categories:
- vmq
tags:
- vmq
- MQTT
- 网络编程
- Go
---

实现了 `CONNECT` 包后，可以按照相似的方法实现其他几个重要的数据包，例如： `CONNACK` `DISCONNECT` `SUBSCRIBE` `SUBACK` `PUBLISH`，有了这些包就可以进行基础的 MQTT 通信了。

<!-- more -->

# 分类与接口

在实现这些包之前，我们可以将这些包进行分类，站在 MQTT 客户端的角度，这些包可以分为三类：

- 只需发送的：`CONNECT` `SUBSCRIBE`
- 只需接收的：`CONNACK` `SUBACK`
- 需要发送和接收的：`DISCONNECT` `PUBLISH`

然后可以为发送和接收分别制定一套简易流程，对于要发送的包：

1. 提供配置类，用户使用配置类设定自定义参数，例如：`ConnectConf`
2. vmq提供操作函数，用户将配置类作为参数，通过操作函数实现发包动作，例如：`vmq.Connect(*ConnectConf)`
3. 在操作函数中，使用一个转换函数将配置类转为包，例如：`NewConnectPacketFromConf(*ConnectConf)`
4. 操作函数完成发包，提供返回

对于要接收的包，则是这样的流程：

1. vmq提供设置回调函数的入口，例如：`v.OnMessage(func(v *vmq, topic string, message []byte))`
2. vmq接收到数据流后，解析第一个字节 `packetType` 和随后的 `remaningLength`，之后根据包类型使用该包的组包函数，例如：`NewPublishPacket(packetType *t.MQTT_BYTE, remainingLength *t.MQTT_VAR_INT) *PUBLISH_PACKET`
3. 组包后，根据包类型触发 vmq 内的 handle 函数，例如：`handlePublish(v *vmq, pub *p.PUBLISH_PACKET) error`，并在 handle 中触发被设置的回调函数

通过上述流程，可以看到要发送和要接收的包只需要实现几个特定的函数就可以实现收发功能，例如对于既需要发送又需要接收的 `PUBLISH` 包，发送流程是这样的：

- 提供 `PublishConf` 作为配置类，提供构造函数和各种Set函数
- 提供 `NewPublishPacketFromConf(pc *PublishConf) *PUBLISH_PACKET`，用于通过配置制作包
- 提供 `(p *PUBLISH_PACKET) ToStream(output io.Writer) (int, error)`，用于实际发送
- vmq侧提供 `(v *vmq) Publish(pc *p.PublishConf) error`，作为用户操作的入口

```go vmq.go
func (v *vmq) Publish(pc *p.PublishConf) error {
	if v.status != STATUS_CONNECTED {
		return errors.New("vmq status error: not connected")
	}

	publish := p.NewPublishPacketFromConf(pc)
	// packetId if qos > 0
	if publish.Qos() > p.PUBLISH_FLAG_QOS_0 {
		publish.VariableHeader.PacketId.FromValue(v.getPID())
	}
	_, err := publish.ToStream(v.conn)
	if err != nil {
		return err
	}
	return nil
}
```

```go publishconf.go
type PublishConf struct {
	dup                bool
	qos                PUBLISH_FLAG_QOS
	retain             bool
	topic              string
	publishProperties  PROPERTIES
	applicationMessage []byte
}

func NewPublishConf() *PublishConf {
	return &PublishConf{
		qos:                PUBLISH_FLAG_QOS_0,
		publishProperties:  *NewProperties(),
		applicationMessage: make([]byte, 0),
	}
}
```

```go publish.go
type PUBLISH_FLAG_QOS byte

const PUBLISH_FLAG_RETAIN byte = 0x01
const PUBLISH_FLAG_QOS_0 PUBLISH_FLAG_QOS = 0x00
const PUBLISH_FLAG_QOS_1 PUBLISH_FLAG_QOS = 0x02
const PUBLISH_FLAG_QOS_2 PUBLISH_FLAG_QOS = 0x04
const PUBLISH_FLAG_DUP byte = 0x09

type PUBLISH_PACKET struct {
	FixHeader      PUBLISH_FIX_HEADER
	VariableHeader PUBLISH_VARIABLE_HEADER
	Payload        PUBLISH_PAYLOAD
}

type PUBLISH_FIX_HEADER struct {
	PacketType      t.MQTT_BYTE // pak type dup qos retain
	RemainingLength t.MQTT_VAR_INT
}

type PUBLISH_VARIABLE_HEADER struct {
	TopicName         t.MQTT_UTF8
	PacketId          t.MQTT_U16
	PublishProperties PROPERTIES
}

type PUBLISH_PAYLOAD struct {
	ApplicationMessage []byte
}

func NewPublishPacketFromConf(pc *PublishConf) *PUBLISH_PACKET {
	packet := PUBLISH_PACKET{}
	// fix header
	packetType := byte(PACKET_TYPE_PUBLISH)
	if pc.retain {
		packetType |= PUBLISH_FLAG_RETAIN
	}
	packetType |= byte(pc.qos)
	if pc.dup {
		packetType |= PUBLISH_FLAG_DUP
	}
	packet.FixHeader.PacketType.FromValue(packetType)
	remainingLen := 0

	// variable header
	packet.VariableHeader.TopicName.FromValue(pc.topic)
	remainingLen += packet.VariableHeader.TopicName.Length()
	// only qos1 or qos2, there is packetId
	if packet.Qos() > PUBLISH_FLAG_QOS_0 {
		// PID is 0, then vmq will set it
		packet.VariableHeader.PacketId.FromValue(0)
		remainingLen += packet.VariableHeader.PacketId.Length()
	}
	packet.VariableHeader.PublishProperties = pc.publishProperties
	remainingLen += packet.VariableHeader.PublishProperties.Length()

	// payload
	packet.Payload.ApplicationMessage = pc.applicationMessage
	remainingLen += len(packet.Payload.ApplicationMessage)

	// set remainingLen
	packet.FixHeader.RemainingLength.FromValue(remainingLen)

	return &packet
}

func (p *PUBLISH_PACKET) ToStream(output io.Writer) (int, error) {
	var err error
	var buffer = bytes.NewBuffer(nil)
	// fix header
	_, err = p.FixHeader.PacketType.ToStream(buffer)
	if err != nil {
		return 0, err
	}
	_, err = p.FixHeader.RemainingLength.ToStream(buffer)
	if err != nil {
		return 0, err
	}

	// variable header
	_, err = p.VariableHeader.TopicName.ToStream(buffer)
	if err != nil {
		return 0, err
	}
	if p.Qos() > PUBLISH_FLAG_QOS_0 {
		_, err = p.VariableHeader.PacketId.ToStream(buffer)
		if err != nil {
			return 0, err
		}
	}
	_, err = p.VariableHeader.PublishProperties.ToStream(buffer)
	if err != nil {
		return 0, err
	}

	// payload
	_, err = buffer.Write(p.Payload.ApplicationMessage)
	if err != nil {
		return 0, err
	}

	// to stream
	return output.Write(buffer.Bytes())
}
```

而他的接收流程是这样的：

- 用户设置回调函数，自定义对应用消息的处理方式
- vmq收到数据，判断包类型
- vmq使用 `PUBLISH` 的组包函数，获得 `PUBLISH_PACKET`
- vmq调用 `PUBLISH` 的 handle 函数，在其中使用用户注册的回调

```go vmq.go
// set callback
func (v *vmq) OnMessage(f func(v *vmq, topic string, message []byte)) {
	v.onMessage = f
}

func handleData(v *vmq) {
	packetType := t.NewByte()
	remainingLength := t.NewVarInt()
	for {
		// read fix header: PacketType + RemainingLength
		_, err := packetType.FromStream(v.conn)
		// EOF means the server is disconnected
		if err == io.EOF {
			v.conn.Close()
			v.setStatus(STATUS_DISCONNECTED)
			break
		} else if err != nil {
			v.conn.Close()
			v.setStatus(STATUS_IDLE)
			break
		}
		_, err = remainingLength.FromStream(v.conn)
		if err != nil {
			v.conn.Close()
			v.setStatus(STATUS_IDLE)
			break
		}
		// handle packet
		err = handlePacket(v, packetType, remainingLength)
		if err != nil {
			v.conn.Close()
			v.setStatus(STATUS_IDLE)
			break
		}
	}
}

func handlePacket(v *vmq, packetType *t.MQTT_BYTE, remainingLength *t.MQTT_VAR_INT) error {
	// slect packet by type
	// TODO need more packet type
	if packetType.ToValue() == byte(p.PACKET_TYPE_CONNACK) {
		// stream to packet
		ca := p.NewConnackPacket(packetType, remainingLength)
		_, err := ca.FromStream(v.conn)
		if err != nil {
			return err
		}
		// handle connACK
		return handleConnAck(v, ca)
	}
	if packetType.ToValue() == byte(p.PACKET_TYPE_DISCONNECT) {
		dc := p.NewDisconnectPacket(packetType, remainingLength)
		_, err := dc.FromStream(v.conn)
		if err != nil {
			return err
		}
		// handle connACK
		return handleDisconn(v, dc)
	}
	if packetType.ToValue() == byte(p.PACKET_TYPE_SUBACK) {
		sa := p.NewSubackPacket(packetType, remainingLength)
		_, err := sa.FromStream(v.conn)
		if err != nil {
			return err
		}
		return handleSuback(v, sa)
	}
	if packetType.ToValue() == byte(p.PACKET_TYPE_PUBLISH) {
		pub := p.NewPublishPacket(packetType, remainingLength)
		_, err := pub.FromStream(v.conn)
		if err != nil {
			return err
		}
		return handlePublish(v, pub)
	}
	return errors.New("can't metch packet type")
}

func handlePublish(v *vmq, pub *p.PUBLISH_PACKET) error {
	v.onMessage(v, pub.VariableHeader.TopicName.ToValue(), pub.Payload.ApplicationMessage)
	return nil
}
```

这样的流程看上去有些麻烦，但是暂时所有的包都能够通过这样一套流程实现，后续随着会话状态的引入，肯定会有所变化。

# 测试

到此为止，已经可以直接测试连接、订阅、发布、断开的过程：

```go vmq_test.go
func TestPublish(test *testing.T) {
	// vmq
	v := New("test", "tcp", "36.33.24.191", 1883)
	v.OnMessage(func(v *vmq, topic string, message []byte) {
		fmt.Println(topic)
		fmt.Println(string(message))
	})
	// conn conf
	cc := p.NewConnectConf()
	cc.SetClientID("impub")
	// conn to server
	err := v.Connect(cc)
	if err != nil {
		test.Error()
	}
	// do sub after 3 second
	for x := 0; x < 10; x++ {
		time.Sleep(1 * time.Second)
		if x == 3 {
			subconf := p.NewSubscribeConf()
			subconf.SetTopic("vctest/vc")
			subconf.SetTopicAndOption("vctest/vc2", p.SUBSCRIBE_OPTION_QOS_2, true, true, p.SUBSCRIBE_OPTION_RETAIN_HANDLING_1)
			subconf.SetTopic("vctest/vc3")
			v.Subscribe(subconf)
		}
		if x == 5 {
			pc := p.NewPublishConf()
			pc.SetTopic("vctest/vc")
			pc.SetApplicationMessage([]byte("hello !!!! i ! am ! vmq !!!"))
			v.Publish(pc)
		}
	}
	v.Disconnect(p.NewDisconnectConf())
	time.Sleep(1 * time.Second)
}
```

# 结尾

此时的 vmq 已经具备了连接、订阅、发布、断开的基本能力，但是离我们的目标还很远。例如，vmq 目前没有提供会话状态的能力，没有实现 PUBLISH 包的 QOS1 和 QOS2 的流程，对外发布的使用接口也混乱不堪。

此时我们的项目结构是这样的：

```text
vmq/
 ├── packets/
 │    ├── connack.go
 │    ├── connect.go
 │    ├── connectconf.go
 │    ├── disconnect.go
 │    ├── disconnectconf.go
 │    ├── packets.go
 │    ├── pingreq.go
 │    ├── properties_test.go
 │    ├── properties.go
 │    ├── publish.go
 │    ├── publishconf.go
 │    ├── suback.go
 │    ├── subscribe.go
 │    ├── packets.go
 │    ├── subscribeconf.go
 │    └── unsubscribe.go
 ├── types/
 │    ├── mqttbin.go
 │    ├── mqttbyte.go
 │    ├── mqttu16.go
 │    ├── mqttu32.go
 │    ├── mqttutf8.go
 │    ├── mqttutf8pair.go
 │    ├── mqttvarint.go
 │    ├── types_test.go
 │    └── types.go
 ├── go.mod
 └── vmq.go
```