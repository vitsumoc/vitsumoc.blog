---
title: vmq-从零开始的MQTT客户端-3-CONNECT
url: vmq-mqttclient-3
date: 2024-03-18 13:42:26
categories:
- vmq
tags:
- vmq
- MQTT
- 网络编程
- Go
---

MQTT 5 定义了 15 种控制包，CONNECT 是其中的第一种，他也很适合作为我们实现的起点。

<!-- more -->

# 包结构

参考[文档](/mqtt-v5-0-chinese.html#3-1-CONNECT-连接请求)实现 CONNECT 的包结构，他由固定头、可变头和载荷构成：

```go connect.go
type CONNECT_PACKET struct {
	FixHeader      CONNECT_FIX_HEADER
	VariableHeader CONNECT_VARIABLE_HEADER
	Payload        CONNECT_PAYLOAD
}

type CONNECT_FIX_HEADER struct {
	PacketType      t.MQTT_BYTE
	RemainingLength t.MQTT_VAR_INT
}

type CONNECT_VARIABLE_HEADER struct {
	ProtocolName      t.MQTT_UTF8
	ProtocolLevel     t.MQTT_BYTE
	ConnectFlags      t.MQTT_BYTE
	KeepAlive         t.MQTT_U16
	ConnectProperties PROPERTIES
}

type CONNECT_PAYLOAD struct {
	ClientID       t.MQTT_UTF8
	WillProperties PROPERTIES
	WillTopic      t.MQTT_UTF8
	WillPayload    t.MQTT_BIN
	UserName       t.MQTT_UTF8
	Password       t.MQTT_BIN
}
```

在这里几乎所有的成员都使用我们之前定义的 `MQTT基本类型`，但出现了 `PROPERTIES` 这样一个新的类型，比如 `ConnectProperties` 和 `WillProperties`。

`PROPERTIES` 表示属性集，因为在很多不同的 MQTT 包种都出现了属性集，我将他单独封装成为了一个结构体：

```go properties.go
type PROPERTIES struct {
	PropertyLength t.MQTT_VAR_INT              // length of Properties + UserProperties, not self
	Properties     map[t.MQTT_BYTE]t.MQTT_TYPE // any types of MQTT_*, depending on key byte
	UserProperties []t.MQTT_UTF8_PAIR          // The User Property is allowed to appear multiple times
}
```

MQTT 中的属性集类似于 Go 中的 map，使用键值对的方式存放一组属性。和 Go 中的 map 的主要区别是属性集中有一种被称为 `用户属性` 的特殊属性，他的键可以重复出现多次，因此我使用一个列表来存储用户属性。

这里还可以发现，在 `Properties map[*t.MQTT_BYTE]t.MQTT_TYPE` 中，我将所有的 `MQTT基本类型` 作为了一个接口 `t.MQTT_TYPE`，这个接口的定义是这样的：

```go types.go
type MQTT_TYPE interface {
	Length() int
	FromStream(io.Reader) (int, error)
	ToStream(io.Writer) (int, error)
}
```

这个接口极大的方便了属性集的操作，例如用于计算属性集长度的函数 `calLength()`：

```go properties.go
func (p *PROPERTIES) calLength() {
	length := 0
	for _, v := range p.Properties {
		// key length
		length += 1
		// value length
		length += v.Length()
	}
	// user properties length
	for _, u := range p.UserProperties {
		length += 1
		length += u.Length()
	}
	p.PropertyLength.FromValue(length)
}
```

# 包函数

PROPERTIES 包含了这些方法：

| 签名 | 用途 |
| --- | --- |
| `func NewProperties() *PROPERTIES` | 创建一个新的 PROPERTIES |
| `func (p *PROPERTIES) SetProperty(key MQTT_PROPERTY_KEY, v1 any, v2 any) error` | 设置属性 |
| `func (p *PROPERTIES) Length() int` | 输出 PROPERTIES 在字节流中的长度 |
| `func (p *PROPERTIES) FromStream(input io.Reader) (int, error)` | 从字节流中解析 PROPERTIES |
| `func (p *PROPERTIES) ToStream(output io.Writer) (int, error)` | 将 PROPERTIES 写入字节流 |
| `func (p *PROPERTIES) calLength()` | 内部使用，计算 `PropertyLength` 的值 |

在 `SetProperty` 函数中，`MQTT_PROPERTY_KEY` 是参考[文档](/mqtt-v5-0-chinese.html#2-2-2-2-属性)定义的属性键枚举，v1 和 v2 是属性的值，这里接收的值是 Go 中的基本类型，例如当键对应的值类型为 `4字节整数` 时，这里接收的值类型就是 `uint32`。v2 只在键是 `PROPERTY_USER_PROPERTY` 时有意义，因为此时的值应该是两个 `string`。

而 CONNECT 只有两个方法、构造和序列化：

| 签名 | 用途 |
| --- | --- |
| `func NewConnectPacketFromConf(cc *ConnectConf) *CONNECT_PACKET` | 从配置类 `ConnectConf` 构造`CONNECT` |
| `func (c *CONNECT_PACKET) ToStream(output io.Writer) (int, error)` | 将 `CONNECT` 写入字节流 |

这里使用一个 `ConnectConf` 作为 `CONNECT` 包的配置类，因为直接让用户操作 `CONNECT` 包过于麻烦，通过配置类封装一些Set函数，可以让用户使用时的代码更加简单，配置类的定义和包函数如下：

```go connectconf.go
// all member will be private, so SetXXX() will be the only entrance
type ConnectConf struct {
	// connect flags
	cfCleanStart bool
	cfWillFlag   bool
	cfWillQos    int
	cfWillRetain bool
	cfPassword   bool
	cfUsername   bool

	// variable header
	keepAlive         uint16
	connectProperties PROPERTIES

	// payload
	clientID       string
	willProperties PROPERTIES
	willTopic      string
	willPayload    []byte
	username       string
	password       []byte
}
```

| 签名 | 用途 |
| --- | --- |
| `func NewConnectConf() *ConnectConf` | 初始化配置类 |
| `func (cc *ConnectConf) SetCleanStart(b bool)` | 设置全新开始标志 |
| `func (cc *ConnectConf) SetWill(qos int, retain bool, properties PROPERTIES, topic string, payload []byte)` | 设置遗嘱 |
| `func (cc *ConnectConf) SetPassword(password []byte)` | 设置密码 |
| `func (cc *ConnectConf) SetUsername(username string)` | 设置用户名 |
| `func (cc *ConnectConf) SetKeepAlive(keepAlive uint16)` | 设置保活时间 |
| `func (cc *ConnectConf) SetConnectProperties(pp *PROPERTIES)` | 直接设置属性集 |
| `func (cc *ConnectConf) SetProperty(key MQTT_PROPERTY_KEY, v1 any, v2 any) error` | 设置某属性 |
| `func (cc *ConnectConf) SetClientID(clientID string)` | 设置客户端ID |

这些Set函数涵盖了 `CONNECT` 包中所有可能的配置项，如果没有设置，配置类也会按照协议提供默认值。

# 测试

通过配置类创建 `CONNECT` 包，然后将其输入到 buffer 中，直接打印观察 buffer 的内容是否符合协议的规范和我们设置的参数：

```go vmq_test.go
func TestConnectPacket(test *testing.T) {
	cc := p.NewConnectConf()
	cc.SetClientID("imvc")
	cc.SetKeepAlive(10)
	err := cc.SetProperty(p.PROPERTY_SESSION_EXPIRY_INTERVAL, uint32(10), nil)
	if err != nil {
		test.Error()
	}

	cp := p.NewConnectPacketFromConf(cc)
	buffer := bytes.NewBuffer(nil)
	n, err := cp.ToStream(buffer)
	if err != nil || n != 24 || buffer.Bytes()[2] != 0x00 {
		test.Error()
	}
	fmt.Println(buffer.Bytes())
}
```

# 结尾

有了 CONNECT 包之后，我们就可以将他发往 MQTT 服务器，观察 MQTT 连接的建立情况。这可能包含客户端状态机的管理，网络连接的管理，接收 CONNACK 包等等其他内容。

不论如何，我们还是迈出了一步，现在我们的项目应该是这样的：

```text
vmq/
 ├── packets/
 │    ├── connect.go
 │    ├── connectconf.go
 │    └── properties.go
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