+++
date = '2023-07-19T21:35:00+08:00'
draft = false
title = '裸写TCP通信的正确方式'
+++

> TCP 是一种面向连接的、可靠的、基于字节流的传输层通信协议。

由于TCP的特性，在裸写TCP通信的时候，是不能直接把数据序列化为字节后直接发送的，否则就可能会遇到数据包被“截断”或“粘包”的问题。(粘包警察正在赶来。)

### “截断”的原因

**数据包之所以被“截断”，是因为网卡的 mtu。**

MTU（Maximum Transmission Unit，MTU）的中文名称是【最大传输单元】，它是指网络能够传输的最大数据包大小，以字节为单位。MTU的大小决定了发送端一次能够发送报文的最大字节数。如果MTU超过了接收端所能够承受的最大值，或者是超过了发送路径上途经的某台设备所能够承受的最大值，就会造成报文分片甚至丢弃，加重网络传输的负担。

普通网卡常见的mtu值是1500，vxlan 等隧道协议的 mtu 则会小于1500，这是因为 vxlan 等隧道协议需要对原始数据进行一层封装，并加上一个包头。

### “粘包”的原因

**而数据包之所以被“粘包”，则是因为不确定消息的边界，接收端根本不知到要拿多少数据。**

你说我知道网卡的 mtu 是 1500，那我一次取1500个字节吧，但你拿到的数据报文里面往往会“粘”着下一条消息的一部分。

### 解决方案

因此我们需要一个固定长度的包头，写入消息的时候，把正文的长度写入到包头里面再打包发出，每次读取消息的时候，则先读取固定长度的包头，解析出来正文的长度，再使用正文长度取读取正文。

代码示例：

```golang
const MaxBodyLen = 1<<32 - 1

type Header [8]byte

func (h Header) Version() int {
    return int(h[0])
}

func (h Header) Flag() int {
    return int(h[1])
}

func (h Header) BodyLen() uint32 {
    return binary.BigEndian.Uint32(h[4:])
}

func Read(conn net.Conn) (Header, []byte, error) {
    h := Header{}
    _, err := io.ReadFull(conn, h[:])
    if err != nil {
        return h, nil, err
    }

    bodyLen := h.BodyLen()
    if bodyLen <= 0 {
        return h, nil, nil
    }

    body := make([]byte, bodyLen)
    _, err = io.ReadFull(conn, body)
    if err != nil {
        return h, nil, err
    }

    return h, body, nil
}

func Write(conn net.Conn, flag int, body []byte) error {
    if len(body) > MaxBodyLen {
        return fmt.Errorf("too large body, body length must be <= %d", MaxBodyLen)
    }
    bodyLen := make([]byte, 4)
    binary.BigEndian.PutUint32(bodyLen, uint32(len(body)))

    hdr := []byte{0x01, byte(flag)}
    hdr = append(hdr, make([]byte, 2)...) // 补2个0让消息头总长度为8
    hdr = append(hdr, bodyLen...)

    writeBody := make([]byte, 0)
    writeBody = append(writeBody, hdr...)
    writeBody = append(writeBody, body...)
    _, err := conn.Write(writeBody)
    return err
}
```