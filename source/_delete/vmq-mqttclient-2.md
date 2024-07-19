---
title: vmq-从零开始的MQTT客户端-2-数据表示
url: vmq-mqttclient-2
date: 2024-03-10 15:21:20
categories:
- vmq
tags:
- vmq
- MQTT
- 网络编程
- Go
---

MQTT 5.0 规范中定义了 7 种[数据表示](/mqtt-v5-0-chinese.html#1-5-数据表示)，分别是：

- `比特位`
- `2字节整数`
- `4字节整数`
- `UTF-8字符串`
- `变长整数`
- `二进制数据`
- `UTF-8字符串对`

<!-- more -->

这 7 种数据有些比较简单，有些比较复杂，但是为了方便统一处理，我还是为每一种数据设置了数据类型：

```go types.go
type MQTT_BYTE struct {
	data byte
}
type MQTT_U16 struct {
	data uint16
}
type MQTT_U32 struct {
	data uint32
}
type MQTT_UTF8 struct {
	len  MQTT_U16
	data []byte
}
type MQTT_VAR_INT struct {
	data []byte
}
type MQTT_BIN struct {
	len  MQTT_U16
	data []byte
}
type MQTT_UTF8_PAIR struct {
	key   MQTT_UTF8
	value MQTT_UTF8
}
```

- `比特位`：只是描述字节内部高低位顺序，可以用一个普通的字节来表示
- `2字节整数`：等同于 Go 中的 `uint16`
- `4字节整数`：等同于 Go 中的 `uint32`
- `UTF-8字符串`：使用 `2字节整数` 表示内容长度，使用 `[]byte` 存储字符串内容
- `变长整数`：1-4位的 `[]byte`，用来表示一个正整数值
- `二进制数据`：使用 `2字节整数` 表示内容长度，使用 `[]byte` 存储二进制数据的内容
- `UTF-8字符串对`：两个 `UTF-8字符串`，表示一个键值对

# 接口

上述的 7 种数据表示都有一个共同点，就是使用这些数据时都需要在 3 种不同的状态之间来回转换，这三种状态分别是：

- 值：例如 "i'm MQTT"，在业务代码中使用，类型为 `string`
- 变量：在 MQTT 包中使用，类型为 `MQTT_UTF8`
- 数据流：在网络传输中使用，类型为 `[]byte`

因此，可以为这些数据表示定义统一的使用接口：

- NewXXX：创建XXX类型的变量
- FromStream：从数据流向变量赋值
- ToStream：从变量向数据流赋值
- FromValue：将值赋值给变量
- ToValue：将变量转换为可以用的值

# 比特位

比特位是一个最简单的例子，实现如下：

```go mqttbyte.go
func NewByte() *MQTT_BYTE {
	return &MQTT_BYTE{}
}

func (mb *MQTT_BYTE) FromStream(input io.Reader) (*MQTT_BYTE, int, error) {
	err := binary.Read(input, binary.BigEndian, &mb.data)
	if err != nil {
		return nil, 0, err
	}
	return mb, 1, nil
}

func (mb *MQTT_BYTE) ToStream(output io.Writer) (int, error) {
	err := binary.Write(output, binary.BigEndian, mb.data)
	if err != nil {
		return 0, err
	}
	return 1, nil
}

func (mb *MQTT_BYTE) FromValue(b byte) *MQTT_BYTE {
	mb.data = b
	return mb
}

func (mb *MQTT_BYTE) ToValue() byte {
	return mb.data
}
```

# UTF-8字符串

`2字节整数` 和 `4字节整数` 的实现和 `比特位` 几乎一致，不再赘述。而 `UTF-8字符串` 的实现略微有些不同，因为他需要使用一个 `2字节整数` 来表示字符串的长度，还需要考虑一些越界问题：

```go mqttutf8.go
const MQTT_UTF_8_MAX = 65535 // 0xFFFF

func NewUtf8() *MQTT_UTF8 {
	return &MQTT_UTF8{}
}

func (mutf8 *MQTT_UTF8) FromStream(input io.Reader) (*MQTT_UTF8, int, error) {
	err := binary.Read(input, binary.BigEndian, &mutf8.len.data)
	if err != nil {
		return nil, 0, err
	}
	mutf8.data = make([]byte, mutf8.len.data)
	err = binary.Read(input, binary.BigEndian, mutf8.data)
	if err != nil {
		return nil, 0, err
	}
	return mutf8, 2 + int(mutf8.len.data), nil
}

func (mutf8 *MQTT_UTF8) ToStream(output io.Writer) (int, error) {
	n, err := mutf8.len.ToStream(output)
	if err != nil {
		return n, err
	}
	err = binary.Write(output, binary.BigEndian, mutf8.data)
	if err != nil {
		return n, err
	}
	return 2 + int(mutf8.len.data), nil
}

func (mutf8 *MQTT_UTF8) FromValue(s string) (*MQTT_UTF8, error) {
	if len(s) > MQTT_UTF_8_MAX {
		return nil, errors.New("MQTT_UTF8 length error")
	}
	mutf8.len.data = uint16(len(s))
	mutf8.data = []byte(s)
	return mutf8, nil
}

func (mutf8 *MQTT_UTF8) ToValue() string {
	return string(mutf8.data)
}
```

# 变长整数

`变长整数` 是这些数据表示中最复杂的，好在文档中提供了能作为参考的伪代码，实现后如下：

```go
const (
	MQTT_VAR_INT_MIN = 0         // 0x00
	MQTT_VAR_INT_MAX = 268435455 // 0xFF 0xFF 0xFF 0x7F
)

func NewVarInt() *MQTT_VAR_INT {
	return &MQTT_VAR_INT{}
}

func (mvi *MQTT_VAR_INT) FromStream(input io.Reader) (*MQTT_VAR_INT, int, error) {
	mvi.data = make([]byte, 0)
	buf := make([]byte, 1)
	x := 0
	// maximum 4bytes
	for ; x < 4; x++ {
		_, err := input.Read(buf)
		if err != nil {
			return nil, x, err
		}
		b := buf[0]
		// height byte can't be 0
		if x > 0 && b == 0 {
			return nil, x, errors.New("MQTT_VAR_INT read error")
		}
		mvi.data = append(mvi.data, b)
		// is there after
		if b&0x80 > 0 {
			if x == 3 {
				return nil, x, errors.New("MQTT_VAR_INT read error, length > 4")
			}
			continue
		}
		// over
		break
	}
	return mvi, x + 1, nil
}

func (mvi *MQTT_VAR_INT) ToStream(output io.Writer) (int, error) {
	return output.Write(mvi.data)
}

func (mvi *MQTT_VAR_INT) FromValue(v int) (*MQTT_VAR_INT, error) {
	if v < MQTT_VAR_INT_MIN || v > MQTT_VAR_INT_MAX {
		return nil, errors.New("MQTT_VAR_INT parse error, value out of range")
	}
	mvi.data = make([]byte, 0)
	for {
		var b byte = (byte)(v % 0x80)
		v = v / 0x80
		// set top bit
		if v > 0 {
			b = b | 0x80
		}
		// put
		mvi.data = append(mvi.data, b)
		// over
		if v == 0 {
			break
		}
	}
	return mvi, nil
}

func (mvi *MQTT_VAR_INT) ToValue() int {
	multiplier := 1
	value := 0
	for x := 0; x < len(mvi.data); x++ {
		b := mvi.data[x]
		value += int(b&0x7F) * multiplier
		multiplier *= 0x80
	}
	return value
}
```

# 结尾

这里我没有列出所有数据表示实现的源码，因为这些代码大体相似，只是细节不同，您可以在仓库中自行查看。

有了基础数据，接下来就可以组织 MQTT 包，此时我们的项目结构是这样的：

```text
vmq/
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