---
title: Go与串口设备在项目中的运用
url: GoSerial
date: 2023-11-22 11:06:28
categories:
- 项目实践
tags:
- 项目实践
- 网络编程
- Go
- 串口通讯
---

# 需求简述

硬件设备使用485Modbus通讯，需使用 Go 编写采集程序，将数据采集至平台。

<!--more-->

# 方案简述

使用串口服务器将485Modbus通讯转为TCPModbus，并将串口服务器设置为TCPserver。

使用 Go 编写采集器，定期向串口服务器建立TCP链接，采集数据。

# 技术点与实现

## 点表

通过```struct```实现点位表

这里使用Raw表示原始点表数据，和通讯协议一一对应

后续可将Raw结构封装至更高层的业务结构，用来实现业务数据的表示、嵌套能其他功能

```go go
type StructMcuRaw struct {
	Ver               [4]uint8 `json:"ver"`                                                                               // 软件版本[4]char
	Id                uint16   `json:"id"`                                                                                // 通信箱id
	TargetAngle       uint16   `json:"targetAngle" IEC104:"yc" IEC104Name:"子阵目标角度" IEC104Unit:"°" IEC104Factor:"0.1"` // 对整个子阵设置目标角度
	InitSnowDepth     uint16   `json:"initSnowDepth" IEC104:"yx" IEC104Name:"标定初始雪深标志"`                              // 标定初始雪深标志
	PrecipitationType uint16   `json:"precipitationType"`                                                                 // 降水类型
}
```

## 读取二进制数据

通过```binary```包，可以实现从buffer中读取数据向```struct```赋值

```go go
func (p *StructMcuRaw) MCUFromByte(res *bytes.Buffer) {
	// 软件版本[4]char
	for x := 0; x < 4; x++ {
		binary.Read(res, binary.BigEndian, p.Ver[x])
	}
	// 通信箱id
	binary.Read(res, binary.BigEndian, &p.Id)
	// 对整个子阵设置目标角度
	binary.Read(res, binary.BigEndian, &p.TargetAngle)
	// 标定初始雪深标志
	binary.Read(res, binary.BigEndian, &p.InitSnowDepth)
	// 降水类型
	binary.Read(res, binary.BigEndian, &p.PrecipitationType)
}
```

## 封装为query

在本项目中，query指对单个设备的采集方法

将数据读取封装成query方法，包括TCP采集过程、日志记录、包格式处理等

```go go
func MCUQuery(conn *net.Conn, reader *bufio.Reader, buffer *[]byte, cb *rs.StructCommBox, mcu *rs.StructMcu) error {
	// 查询地址
	addInt, err := strconv.Atoi(mcu.Addr)
	if err != nil {
		return err
	}
	// 包编号
	tcpSeq := TcpSeq()
	var query = []byte{
		uint8(tcpSeq / 0x100), uint8(tcpSeq % 0x100), // 编号
		0x00, 0x00, 0x00, 0x06, // 长度
		byte(addInt), 0x03, 0x00, 0xa0, 0x00, 0x29} // 指令
	(*conn).SetWriteDeadline(time.Now().Add(rs.QUERY_DEFAULT_TIMEOUT))
	_, err = (*conn).Write(query)
	if err != nil {
		log.Log(true, cb.IpAddr, cb.Port, []byte{})
		return err
	}
	// 日志
	log.Log(true, cb.IpAddr, cb.Port, query)
	// 接收
	(*conn).SetReadDeadline(time.Now().Add(rs.QUERY_DEFAULT_TIMEOUT))
	n, err := (*reader).Read(*buffer)
	if err != nil {
		log.Log(false, cb.IpAddr, cb.Port, []byte{})
		return err
	}
	// 日志
	log.Log(false, cb.IpAddr, cb.Port, (*buffer)[:n])
	// 解析
	res := bytes.NewBuffer(*buffer)
	// TCP头
	var tcpHeader rs.StructTCPHeader
	tcpHeader.TCPHeaderFromByte(res)
	if tcpHeader.Seq != tcpSeq {
		return errors.New("TCP异常")
	}
	if tcpHeader.Len != 85 {
		return errors.New("TCP长度异常")
	}
	// modbus头
	var mbHeader rs.StructMudbusHeader
	mbHeader.MudbusHeaderFromByte(res)
	// mcu内容
	mcu.Raw.MCUFromByte(res)
	mcu.VUpdate = true // 标记更新
	return nil
}
```

## 封装为采集过程

最后需要将所有的采集query放置在统一的采集过程中

在一次采集过程中，创建一条TCP链接，完成所有采集动作，最后断开链接

```go go
func Collect(cb *rs.StructCommBox) {
	// 记录网络占用
	NetCh <- true
	defer func() {
		<-NetCh
	}()
	// 初始化采集标识
	eraseFlag(cb)
	// 采集结束后更新时标
	defer func() {
		updateTs(cb)
	}()
	// 建链
	conn, err := net.DialTimeout("tcp", cb.IpAddr+":"+cb.Port, rs.QUERY_DEFAULT_TIMEOUT)
	if err != nil {
		log.Log(true, cb.IpAddr, cb.Port, []byte{})
		return
	}
	cb.VUpdate = true
	defer conn.Close()
	// 读写缓存
	readBuf := bufio.NewReader(conn)
	buffer := make([]byte, 256)
	// 按mcu查询
	for x := 0; x < len(cb.Mcus); x++ {
		// 切换MCU预留时间, 提高成功率
		time.Sleep(rs.QUERY_MCU_INTERVAL)
		// 查mcu信息
		mcu := cb.Mcus[x]
		err = cmd.MCUQuery(&conn, readBuf, &buffer, cb, mcu)
		if err != nil {
			return
		}
		// 分次查跟踪器信息
		for y := 0; y < mcu.TracerNum; {
			// 切换Tracer预留时间, 提高成功率
			time.Sleep(rs.QUERY_TRACER_INTERVAL)
			// 查询长度
			tracerLen := rs.QUERY_TRACER_COUNT
			if mcu.TracerNum-y < rs.QUERY_TRACER_COUNT {
				tracerLen = mcu.TracerNum - y
			}
			err = cmd.TracerQuery(&conn, readBuf, &buffer, cb, mcu, y, tracerLen)
			if err != nil {
				return
			}
			y += tracerLen
		}
	}
}
```

# 总结

使用 Go + 串口服务器 进行串口通讯，非常的简单、直观，易于开发维护。

在本次项目实践中，由于 Go 提供了方便的并发编程与控制机制，高负载环境下的性能也得到了充分保障。