---
title: vmq-从零开始的MQTT客户端-5-变长整数
url: vmq-mqttclient-5
date: 2024-02-25 11:06:33
categories:
- vmq
tags:
- vmq
- MQTT
- 网络编程
- Go
---

要实现 CONNECT 包，就要先实现 MQTT 固定头，要实现固定头，就要先实现变长整数编码的剩余长度（RemainingLength）。很好，从如此细微的一个地方着手，非常符合本系列的一贯精神，总是从最简单的地方开始，始终让项目处在易于理解易于控制的状态中。

<!-- more -->

一方面是我判断我有充足的时间慢慢完善`vmq`（毕竟工作中已经做了太多赶时间的项目了），另一方面是我对 `vmq` 寄予厚望，希望他能够成长壮大。我想在一开始就为 `vmq` 设定一个较高的标准，比如使用英文文档，编写完善的测试代码等。

# 剩余长度

剩余长度是 MQTT 包固定头中的第二个部分，固定头的介绍参考[这里](/mqtt-v5-0-chinese.html#2-1-1-固定头)。剩余长度使用变长整数编码，变长整数的说明参考[这里](/mqtt-v5-0-chinese.html#1-5-5-变长整数)。

# 数据类型和接口

变长整数的定义不再赘述，接下来就是如何定义和使用的问题，考虑到后续的实际使用场景，打算为变长整数定义四个接口：

- 从字节流生成变长整数  remainingLengthFromBytes
- 从数值生成变长整数    remainingLengthFromValue
- 将变长整数写入字节流  toBytes
- 读取变长整数的数值    toValue

考虑到读写的方便，打算包装一个 `[]byte` 来存储变长整数。

```go remaininglength.go
type remainingLength struct {
	bytes []byte
}

func remainingLengthFromBytes(input io.Reader) (rl *remainingLength, n int, err error) {}

func remainingLengthFromValue(v int) (rl *remainingLength, err error) {}

func (rl remainingLength) toBytes(output io.Writer) (n int, err error) {}

func (rl remainingLength) toValue() int {}
```

这四个接口的名称已经表示了他们的用处，因此我也没有做额外的注释。

# 安全性

剩余长度的创建由两个 `remainingLengthFromxxxx` 函数实现，而且剩余长度只在 `vmq` 内部使用，所以可以把各种安全性检查代码放到创建函数中，这意味着只要剩余长度能够被创建，那么其中的内容就一定是正确的。剩余长度创建和使用时可能的错误大概有这些：

- io读写错误 (从 Reader 创建时、写入 Writer 时)
- 内容错误：
  - 长度超过4位（从 Reader 创建时）
  - 数值超过取值范围（从 Value 创建时）

这些错误都会在对应的函数中抛出。

# 实现

剩余长度的功能实现非常简单，可以参考规范文档中提供的伪代码。

```go remaininglength.go
package vmq

import (
	"errors"
	"io"
)

const (
	REMAININGLENTH_MIN  = 0         // 0x00
	REMAININGLENGTH_MAX = 268435455 // 0xFF 0xFF 0xFF 0x7F
)

type remainingLength struct {
	bytes []byte
}

func remainingLengthFromBytes(input io.Reader) (rl *remainingLength, n int, err error) {
	rl = &remainingLength{}
	rl.bytes = make([]byte, 0)
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
			return nil, x, errors.New("remainingLength parse err, height byte can't be 0")
		}
		rl.bytes = append(rl.bytes, b)
		// is there after
		if b&0x80 > 0 {
			if x == 3 {
				return nil, x, errors.New("remainingLength parse err, length > 4")
			}
			continue
		}
		// over
		break
	}
	return rl, x + 1, nil
}

func remainingLengthFromValue(v int) (rl *remainingLength, err error) {
	if v < REMAININGLENTH_MIN || v > REMAININGLENGTH_MAX {
		return nil, errors.New("remainingLength make err, value ranges err")
	}
	rl = &remainingLength{}
	rl.bytes = make([]byte, 0)
	for {
		var b byte = (byte)(v % 0x80)
		v = v / 0x80
		// set top bit
		if v > 0 {
			b = b | 0x80
		}
		// put
		rl.bytes = append(rl.bytes, b)
		// over
		if v == 0 {
			break
		}
	}
	return rl, nil
}

func (rl remainingLength) toBytes(output io.Writer) (n int, err error) {
	return output.Write(rl.bytes)
}

func (rl remainingLength) toValue() int {
	multiplier := 1
	value := 0
	for x := 0; x < len(rl.bytes); x++ {
		b := rl.bytes[x]
		value += int(b&0x7F) * multiplier
		multiplier *= 0x80
	}
	return value
}

```

# 测试

实话实说我不是很擅长做测试，但是索性 Go 提供了我这种小白也可以理解的测试包，在这里我尽量测试了所有的功能和错误类型：

```go remaininglength_test.go
package vmq

import (
	"bytes"
	"testing"
)

func TestRemainingLengthFromBytes(t *testing.T) {
	testBytes := []byte{0x01, 0x02, 0x03, 0x04, 0x91, 0x92, 0x93, 0x94, 0x95, 0x96, 0x07, 0x98}
	testBytes_2 := []byte{0x80, 0x00}

	// expect [0x01] 1 nil
	reader_1 := bytes.NewReader(testBytes)
	rl_1, n, err := remainingLengthFromBytes(reader_1)
	if len(rl_1.bytes) != 1 || rl_1.bytes[0] != 0x01 || n != 1 || err != nil {
		t.Error()
	}

	// expect length error
	reader_2 := bytes.NewReader(testBytes[4:])
	_, _, err = remainingLengthFromBytes(reader_2)
	if err == nil {
		t.Error()
	}

	// expect [0x94, 0x95, 0x96, 0x07] 4 nil
	reader_3 := bytes.NewReader(testBytes[7:])
	rl_3, n, err := remainingLengthFromBytes(reader_3)
	if len(rl_3.bytes) != 4 || rl_3.bytes[0] != 0x94 || rl_3.bytes[3] != 0x07 || n != 4 || err != nil {
		t.Error()
	}

	// expect EOF error
	reader_4 := bytes.NewReader(testBytes[11:])
	_, _, err = remainingLengthFromBytes(reader_4)
	if err == nil {
		t.Error()
	}

	// expect height byte can't be 0 err
	reader_5 := bytes.NewReader(testBytes_2)
	_, _, err = remainingLengthFromBytes(reader_5)
	if err == nil {
		t.Error()
	}
}

func TestRemainingLengthFromValue(t *testing.T) {
	v1 := 1
	v2 := 399
	v3 := -15
	v4 := REMAININGLENGTH_MAX - 2
	v5 := REMAININGLENGTH_MAX
	v6 := REMAININGLENGTH_MAX + 1

	// expect [0x01] nil
	rl_1, err := remainingLengthFromValue(v1)
	if len(rl_1.bytes) != 1 || rl_1.bytes[0] != 0x01 || err != nil {
		t.Error()
	}

	// expect [0x8F 0x03] nil
	rl_2, err := remainingLengthFromValue(v2)
	if len(rl_2.bytes) != 2 || rl_2.bytes[0] != 0x8F || rl_2.bytes[1] != 0x03 || err != nil {
		t.Error()
	}

	// expect value ranges err
	_, err = remainingLengthFromValue(v3)
	if err == nil {
		t.Error()
	}

	// expect [0xFD, 0xFF, 0xFF, 0x7F] nil
	rl_4, err := remainingLengthFromValue(v4)
	if len(rl_4.bytes) != 4 || rl_4.bytes[0] != 0xFD ||
		rl_4.bytes[1] != 0xFF || rl_4.bytes[2] != 0xFF ||
		rl_4.bytes[3] != 0x7F || err != nil {
		t.Error()
	}

	// expect [0xFF, 0xFF, 0xFF, 0x7F] nil
	rl_5, err := remainingLengthFromValue(v5)
	if len(rl_5.bytes) != 4 || rl_5.bytes[0] != 0xFF ||
		rl_5.bytes[1] != 0xFF || rl_5.bytes[2] != 0xFF ||
		rl_5.bytes[3] != 0x7F || err != nil {
		t.Error()
	}

	// expect value ranges err
	_, err = remainingLengthFromValue(v6)
	if err == nil {
		t.Error()
	}
}

func TestToBytes(t *testing.T) {
	testBytes_1 := []byte{0x01}
	testBytes_2 := []byte{0x94, 0x95, 0x96, 0x07}

	// expect [0x01] nil
	reader_1 := bytes.NewReader(testBytes_1)
	rl_1, _, _ := remainingLengthFromBytes(reader_1)
	buffer := bytes.NewBuffer(nil)
	n, err := rl_1.toBytes(buffer)
	if len(buffer.Bytes()) != 1 || buffer.Bytes()[0] != 0x01 || n != 1 || err != nil {
		t.Error()
	}

	// expect [0x94, 0x95, 0x96, 0x07] nil
	reader_2 := bytes.NewReader(testBytes_2)
	rl_2, _, _ := remainingLengthFromBytes(reader_2)
	buffer_2 := bytes.NewBuffer(nil)
	n, err = rl_2.toBytes(buffer_2)
	if len(buffer_2.Bytes()) != 4 || buffer_2.Bytes()[0] != 0x94 ||
		buffer_2.Bytes()[1] != 0x95 || buffer_2.Bytes()[2] != 0x96 ||
		buffer_2.Bytes()[3] != 0x07 || n != 4 || err != nil {
		t.Error()
	}
}

func TestToValue(t *testing.T) {
	testBytes_1 := []byte{0x01}
	testBytes_2 := []byte{0x94, 0x95, 0x96, 0x07}

	// expect 1
	reader_1 := bytes.NewReader(testBytes_1)
	rl_1, _, _ := remainingLengthFromBytes(reader_1)
	v := rl_1.toValue()
	if v != 1 {
		t.Error()
	}

	// expect 15043220
	reader_2 := bytes.NewReader(testBytes_2)
	rl_2, _, _ := remainingLengthFromBytes(reader_2)
	v = rl_2.toValue()
	// 0x14 * 128^0 + 0x15 * 128^1 + 0x16 * 128^2 + 0x07 * 128^3
	if v != 15043220 {
		t.Error()
	}
}
```

# 未来可能修改

当前将变长整数功能和剩余长度写在了一起，也确实能解决眼下的问题。但是，后续的开发中会出现其他业务字段也采用变长整数编码，到时需要将变长整数实现从剩余长度中抽取出来。

当前的 `toBytes` `toValue` 函数名可能会在此包里产生冲突，后续需要考虑更好的命名方式。