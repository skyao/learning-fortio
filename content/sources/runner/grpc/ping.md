---
title: "ping测试"
linkTitle: "ping测试"
weight: 100
date: 2022-03-28
description: >
  Fortio的gRPC ping测试
---



整体看一下 gRPC ping 的实现。

## proto 定义

`fgrpc/ping.proto` 文件定义了 PingServer 服务和 Ping 方法：

```protobuf
syntax = "proto3";
package fgrpc;

message PingMessage {
  int64 seq      = 1; // sequence number
  int64 ts       = 2; // src send ts / dest receive ts  //这个是timestamp
  string payload = 3; // extra packet data
  int64 delayNanos = 4; // delay the response by x nanoseconds
}

service PingServer {
  rpc Ping (PingMessage) returns (PingMessage) {}
}
```

生成的对应的 go 代码在 `fgrpc/ping.pb.go` 中。

这是一个非常简单的方法，参数也足够简单。





## 客户端发起 ping 测试请求

```go
ts := time.Now().UnixNano()
// 创建 PingServerClient
grpcstate[i].clientP = NewPingServerClient(conn)
if grpcstate[i].clientP == nil {
  return nil, fmt.Errorf("unable to create ping client %d for %s", i, o.Destination)
}
// 组装请求的message
grpcstate[i].reqP = PingMessage{Payload: o.Payload, DelayNanos: o.Delay.Nanoseconds(), Seq: int64(i), Ts: ts}
if o.Exactly <= 0 {
  // 调用 ping 方法
  _, err = grpcstate[i].clientP.Ping(context.Background(), &grpcstate[i].reqP)
}
```

很正统的 gRPC 调用方式。

> 疑虑：为什么要为每个线程都建立一个 gRPC 连接？其实可以多线程共用一个 gRPC 连接的。



## 服务器端接收并响应 ping 测试请求

`fgrpc/pingsrv.go` 中实现了一个简单的 ping 方法：

```bash
func (s *pingSrv) Ping(c context.Context, in *PingMessage) (*PingMessage, error) {
	log.LogVf("Ping called %+v (ctx %+v)", *in, c)
	out := *in // copy the input including the payload etc
	out.Ts = time.Now().UnixNano()
	if in.DelayNanos > 0 {
		s := time.Duration(in.DelayNanos)
		log.LogVf("GRPC ping: sleeping for %v", s)
		time.Sleep(s)
	}
	return &out, nil
}
```



