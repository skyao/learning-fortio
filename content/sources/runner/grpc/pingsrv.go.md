---
title: "pingsrv.go"
linkTitle: "pingsrv.go"
weight: 100
date: 2022-03-28
description: >
  Fortio的pingsrv.go源码
---



## 常量和结构体定义

 ```go
 const (
 	// DefaultHealthServiceName is the default health service name used by fortio.
 	DefaultHealthServiceName = "ping"
 	// Error indicates that something went wrong with healthcheck in grpc.
 	Error = "ERROR"
 )
 
 type pingSrv struct{}
 ```



## Ping() 方法实现

Ping() 方法的服务器端实现，没啥特殊。

```go
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



## PingServer()方法实现

PingServer 方法启动 grpc ping（和 health）echo 服务器。

返回被绑定的端口（当传递 "0" 作为端口以获得一个动态服务器时非常有用）。

传递 healthServiceName 以用于 grpc 服务名称的健康检查（或传递 DefaultHealthServiceName），以标记为SERVING。

传递 maxConcurrentStreams > 0 来设置该选项。

```go
// PingServer starts a grpc ping (and health) echo server.
// returns the port being bound (useful when passing "0" as the port to
// get a dynamic server). Pass the healthServiceName to use for the
// grpc service name health check (or pass DefaultHealthServiceName)
// to be marked as SERVING. Pass maxConcurrentStreams > 0 to set that option.
func PingServer(port, cert, key, healthServiceName string, maxConcurrentStreams uint32) net.Addr {
	socket, addr := fnet.Listen("grpc '"+healthServiceName+"'", port)
	if addr == nil {
		return nil
	}
	var grpcOptions []grpc.ServerOption
	if maxConcurrentStreams > 0 {
		log.Infof("Setting grpc.MaxConcurrentStreams server to %d", maxConcurrentStreams)
		grpcOptions = append(grpcOptions, grpc.MaxConcurrentStreams(maxConcurrentStreams))
	}
	if cert != "" && key != "" {
		creds, err := credentials.NewServerTLSFromFile(cert, key)
		if err != nil {
			log.Fatalf("Invalid TLS credentials: %v\n", err)
		}
		log.Infof("Using server certificate %v to construct TLS credentials", cert)
		log.Infof("Using server key %v to construct TLS credentials", key)
		grpcOptions = append(grpcOptions, grpc.Creds(creds))
	}
  
  // 创建 grpc server
	grpcServer := grpc.NewServer(grpcOptions...)
	reflection.Register(grpcServer)
  
  // 在 grpc server上注册 healthServer
	healthServer := health.NewServer()
	healthServer.SetServingStatus(healthServiceName, grpc_health_v1.HealthCheckResponse_SERVING)
	grpc_health_v1.RegisterHealthServer(grpcServer, healthServer)
  
  // 在 grpc server上注册 pingServer
	RegisterPingServerServer(grpcServer, &pingSrv{})
  
  // 启动 grpc server
	go func() {
		if err := grpcServer.Serve(socket); err != nil {
			log.Fatalf("failed to start grpc server: %v", err)
		}
	}()
	return addr
}
```



## PingServerTCP()方法实现

PingServerTCP 是假设 tcp 而不是可能的 unix domain socket 端口的 PingServer() ，返回数字端口。

```go
// PingServerTCP is PingServer() assuming tcp instead of possible unix domain socket port, returns
// the numeric port.
func PingServerTCP(port, cert, key, healthServiceName string, maxConcurrentStreams uint32) int {
	addr := PingServer(port, cert, key, healthServiceName, maxConcurrentStreams)
	if addr == nil {
		return -1
	}
	return addr.(*net.TCPAddr).Port
}
```



## PingClientCall() 方法实现

PingClientCall调用ping服务（估计是作为PingServer在目的地运行）。返回以秒为单位的平均往返行程。

```go
func PingClientCall(serverAddr, cacert string, n int, payload string, delay time.Duration, insecure bool) (float64, error) {
	o := GRPCRunnerOptions{Destination: serverAddr, CACert: cacert, Insecure: insecure}
  // 建立连接
	conn, err := Dial(&o) // somehow this never seem to error out, error comes later
	if err != nil {
		return -1, err // error already logged
	}
	msg := &PingMessage{Payload: payload, DelayNanos: delay.Nanoseconds()}
  // 创建 ping client
	cli := NewPingServerClient(conn)
  
	// Warm up: 热身
	_, err = cli.Ping(context.Background(), msg)
	if err != nil {
		log.Errf("grpc error from Ping0 %v", err)
		return -1, err
	}
  
	skewHistogram := stats.NewHistogram(-10, 2)
	rttHistogram := stats.NewHistogram(0, 10)
	for i := 1; i <= n; i++ {
    // 执行一次 ping 
		msg.Seq = int64(i)
		t1a := time.Now().UnixNano()
		msg.Ts = t1a
		res1, err := cli.Ping(context.Background(), msg)
		t2a := time.Now().UnixNano()
		if err != nil {
			log.Errf("grpc error from Ping1 iter %d: %v", i, err)
			return -1, err
		}
    
    // 再执行一次 ping 
		t1b := res1.Ts
		res2, err := cli.Ping(context.Background(), msg)
		t3a := time.Now().UnixNano()
		t2b := res2.Ts
		if err != nil {
			log.Errf("grpc error from Ping2 iter %d: %v", i, err)
			return -1, err
		}
    
    // 计算结果
		rt1 := t2a - t1a
		rttHistogram.Record(float64(rt1) / 1000.)
		rt2 := t3a - t2a
		rttHistogram.Record(float64(rt2) / 1000.)
		rtR := t2b - t1b
		rttHistogram.Record(float64(rtR) / 1000.)
		midR := t1b + (rtR / 2)
		avgRtt := (rt1 + rt2 + rtR) / 3
		x := (midR - t2a)
		log.Infof("Ping RTT %d (avg of %d, %d, %d ns) clock skew %d",
			avgRtt, rt1, rtR, rt2, x)
		skewHistogram.Record(float64(x) / 1000.)
		msg = res2
	}
	skewHistogram.Print(os.Stdout, "Clock skew histogram usec", []float64{50})
	rttHistogram.Print(os.Stdout, "RTT histogram usec", []float64{50})
	return rttHistogram.Avg() / 1e6, nil
}
```



## GrpcHealthCheck 实现

GrpcHealthCheck对标准的grpc健康检查服务进行grpc客户端调用。

```go
// GrpcHealthCheck makes a grpc client call to the standard grpc health check
// service.
func GrpcHealthCheck(serverAddr, cacert string, svcname string, n int, insecure bool) (*HealthResultMap, error) {
	log.Debugf("GrpcHealthCheck for %s svc '%s', %d iterations", serverAddr, svcname, n)
	o := GRPCRunnerOptions{Destination: serverAddr, CACert: cacert, Insecure: insecure}
  
  // 建立连接
	conn, err := Dial(&o)
	if err != nil {
		return nil, err
	}
  
  
	msg := &grpc_health_v1.HealthCheckRequest{Service: svcname}
	cli := grpc_health_v1.NewHealthClient(conn)
	rttHistogram := stats.NewHistogram(0, 10)
	statuses := make(HealthResultMap)

	for i := 1; i <= n; i++ {
    // 执行 check
		start := time.Now()
		res, err := cli.Check(context.Background(), msg)
		dur := time.Since(start)
		log.LogVf("Reply from health check %d: %+v", i, res)
		if err != nil {
			log.Errf("grpc error from Check %v", err)
			return nil, err
		}
		statuses[res.Status.String()]++
		rttHistogram.Record(dur.Seconds() * 1000000.)
	}
	rttHistogram.Print(os.Stdout, "RTT histogram usec", []float64{50})
	for k, v := range statuses {
		fmt.Printf("Health %s : %d\n", k, v)
	}
	return &statuses, nil
}
```

GrpcHealthCheck()是单线程循环调用 check，因此如果要多线程并发，就需要多线程下调用 GrpcHealthCheck()。
