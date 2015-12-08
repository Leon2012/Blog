title: 初学go的几个问题
date: 2015/12/6 20:00:00
tags: [golang]
categories: golang
---

初学go, 基于已有的Java IM服务端写了一个简单的客户端.
用go实现本身也比较简单, 但是涉及跟已有的私有通讯协议兼容比较麻烦(私有通讯协议是自定义报文, 并且报文经过了加密和压缩), 中间碰到一些问题记录一下.

## 3DES ECB
3DES加密有ECB和CBC两种模式.
因为ECB模式有缺陷, go默认只支持CBC模式, 所以如果需要用到ECB模式就要自己去实现.

http://stackoverflow.com/questions/24072026/golang-aes-ecb-encryption
https://code.google.com/p/go/issues/detail?id=5597

## JZLIB
Go默认是支持jzlib压缩的, 但是不知道什么原因, 始终比我们用Java JZLIB压缩出来的多两个字节.
最后没找到原因于是用了个比较挫的办法, Go压缩以后每次固定去除最后两个字节.
解压缩是没问题的.

## unsigned rigth shift
Java中有无符号右移, 但是go没有, 需要自己实现.
```
func unsignedShift(val int, n uint) int {
	return (val >> n) & ((1 << (32 - n)) - 1)
}

```
http://stackoverflow.com/questions/9071797/ruby-unsigned-right-shift

## 整形和字节数组转换

```
func Int64ToBytes(i int64) []byte {
    var buf = make([]byte, 8)
    binary.BigEndian.PutUint64(buf, uint64(i))
    return buf
}

func BytesToInt64(buf []byte) int64 {
    return int64(binary.BigEndian.Uint64(buf))
}
```

http://studygolang.com/topics/22

需要注意下这组方法有BigEndian/LittleEndian和Uint16/32/64多种组合.
要参考源数据看下用那种组合去做转换.
