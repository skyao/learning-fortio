---
title: "grpcrunner.go"
linkTitle: "grpcrunner.go"
weight: 10
date: 2022-03-28
description: >
  Fortio的grpcrunner.go源码
---



## GRPCRunnerResults 结构体定义

GRPCRunnerResults是 GRPCRunner 的结果聚合。也是每个线程/goroutine使用的内部类型。

```go
// GRPCRunnerResults is the aggregated result of an GRPCRunner.
// Also is the internal type used per thread/goroutine.
type GRPCRunnerResults struct {
	periodic.RunnerResults
	clientH     grpc_health_v1.HealthClient					// 用于 health
	reqH        grpc_health_v1.HealthCheckRequest   // 用于 health
	clientP     PingServerClient										// 用于 ping
	reqP        PingMessage													// 用于 ping
	RetCodes    HealthResultMap											// 用于 health
	Destination string
	Streams     int
	Ping        bool
}
```

## GRPCRunnerOptions 结构体定义

GRPCRunnerOptions 包括基本的 RunnerOptions 和 Grpc 特定的选项。

```go
// GRPCRunnerOptions includes the base RunnerOptions plus grpc specific
// options.
type GRPCRunnerOptions struct {
	periodic.RunnerOptions
	Destination        string
	Service            string        // Service to be checked when using grpc health check
	Profiler           string        // file to save profiles to. defaults to no profiling
	Payload            string        // Payload to be sent for grpc ping service
	Streams            int           // number of streams. total go routines and data streams will be streams*numthreads.
	Delay              time.Duration // Delay to be sent when using grpc ping service
	CACert             string        // Path to CA certificate for grpc TLS
	CertOverride       string        // Override the cert virtual host of authority for testing
	Insecure           bool          // Allow unknown CA / self signed
	AllowInitialErrors bool          // whether initial errors don't cause an abort
	UsePing            bool          // use our own Ping proto for grpc load instead of standard health check one.
	UnixDomainSocket   string        // unix domain socket path to use for physical connection instead of Destination
}
```





## Dial()方法实现

当 serverAddr 有 HTTPS 前缀或有提供 cert 时，Dial 使用不安全或 tls 传输安全的grpc。如果 override 被设置为一个非空字符串。它将覆盖请求中权威的虚拟主机名。

```go
// Dial dials grpc using insecure or tls transport security when serverAddr
// has prefixHTTPS or cert is provided. If override is set to a non empty string,
// it will override the virtual host name of authority in requests.
func Dial(o *GRPCRunnerOptions) (conn *grpc.ClientConn, err error) {
	var opts []grpc.DialOption
	switch {
	case o.CACert != "":
		var creds credentials.TransportCredentials
		creds, err = credentials.NewClientTLSFromFile(o.CACert, o.CertOverride)
		if err != nil {
			log.Errf("Invalid TLS credentials: %v\n", err)
			return nil, err
		}
		log.Infof("Using CA certificate %v to construct TLS credentials", o.CACert)
		opts = append(opts, grpc.WithTransportCredentials(creds))
	case strings.HasPrefix(o.Destination, fnet.PrefixHTTPS):
		creds := credentials.NewTLS(&tls.Config{InsecureSkipVerify: o.Insecure}) // nolint: gosec // explicit flag
		opts = append(opts, grpc.WithTransportCredentials(creds))
	default:
		opts = append(opts, grpc.WithInsecure())
	}
	serverAddr := grpcDestination(o.Destination)
  
  // 支持 UnixDomainSocket
	if o.UnixDomainSocket != "" {
		log.Warnf("Using domain socket %v instead of %v for grpc connection", o.UnixDomainSocket, serverAddr)
		opts = append(opts, grpc.WithContextDialer(func(ctx context.Context, addr string) (net.Conn, error) {
			return net.Dial(fnet.UnixDomainSocket, o.UnixDomainSocket)
		}))
	}
  
  // 开始建立grpc连接
	conn, err = grpc.Dial(serverAddr, opts...)
	if err != nil {
		log.Errf("failed to connect to %s with certificate %s and override %s: %v", serverAddr, o.CACert, o.CertOverride, err)
	}
	return conn, err
}
```



## Run()方法实现

Run 方法以目标 QPS 进行 GRPC 健康检查或PING。要在RunnerOptions中设置为功能。

```go
// Run exercises GRPC health check or ping at the target QPS.
// To be set as the Function in RunnerOptions.
func (grpcstate *GRPCRunnerResults) Run(t int) {
	log.Debugf("Calling in %d", t)
	var err error
	var res interface{}
	status := grpc_health_v1.HealthCheckResponse_SERVING
	if grpcstate.Ping {
    // 如果指定要 ping，则以 ping 方法替代默认的 health check
		res, err = grpcstate.clientP.Ping(context.Background(), &grpcstate.reqP)
	} else {
    // 默认为标准的 health check
		var r *grpc_health_v1.HealthCheckResponse
		r, err = grpcstate.clientH.Check(context.Background(), &grpcstate.reqH)
		if r != nil {
			status = r.Status
			res = r
		}
	}
	log.Debugf("For %d (ping=%v) got %v %v", t, grpcstate.Ping, err, res)
  
  // 保存运行的结果
	if err != nil {
		log.Warnf("Error making grpc call: %v", err)
		grpcstate.RetCodes[Error]++
	} else {
		grpcstate.RetCodes[status.String()]++
	}
}
```





## RunGRPCTest() 方法实现

### 处理参数

```go
// RunGRPCTest runs an http test and returns the aggregated stats.
// nolint: funlen, gocognit
func RunGRPCTest(o *GRPCRunnerOptions) (*GRPCRunnerResults, error) {
	if o.Streams < 1 {
		o.Streams = 1
	}
	if o.NumThreads < 1 {
		// sort of todo, this redoing some of periodic normalize (but we can't use normalize which does too much)
		o.NumThreads = periodic.DefaultRunnerOptions.NumThreads
	}
  // 目前只支持 ping 和 health，可惜
	if o.UsePing {
		o.RunType = "GRPC Ping"
		if o.Delay > 0 {
			o.RunType += fmt.Sprintf(" Delay=%v", o.Delay)
		}
	} else {
		o.RunType = "GRPC Health"
	}
	pll := len(o.Payload)
	if pll > 0 {
		o.RunType += fmt.Sprintf(" PayloadLength=%d", pll)
	}
```

### 开始执行测试

```go
	log.Infof("Starting %s test for %s with %d*%d threads at %.1f qps", o.RunType, o.Destination, o.Streams, o.NumThreads, o.QPS)
	o.NumThreads *= o.Streams
	r := periodic.NewPeriodicRunner(&o.RunnerOptions)
	defer r.Options().Abort()
	numThreads := r.Options().NumThreads // may change
	total := GRPCRunnerResults{
		RetCodes:    make(HealthResultMap),
		Destination: o.Destination,
		Streams:     o.Streams,
		Ping:        o.UsePing,
	}
	grpcstate := make([]GRPCRunnerResults, numThreads)
	out := r.Options().Out // Important as the default value is set from nil to stdout inside NewPeriodicRunner
	var conn *grpc.ClientConn
	var err error
	ts := time.Now().UnixNano()
	for i := 0; i < numThreads; i++ {
		r.Options().Runners[i] = &grpcstate[i]
		if (i % o.Streams) == 0 {
			conn, err = Dial(o)
			if err != nil {
				log.Errf("Error in grpc dial for %s %v", o.Destination, err)
				return nil, err
			}
		} else {
			log.Debugf("Reusing previous client connection for %d", i)
		}
		grpcstate[i].Ping = o.UsePing
		var err error
		if o.UsePing { // nolint: nestif
			grpcstate[i].clientP = NewPingServerClient(conn)
			if grpcstate[i].clientP == nil {
				return nil, fmt.Errorf("unable to create ping client %d for %s", i, o.Destination)
			}
			grpcstate[i].reqP = PingMessage{Payload: o.Payload, DelayNanos: o.Delay.Nanoseconds(), Seq: int64(i), Ts: ts}
			if o.Exactly <= 0 {
				_, err = grpcstate[i].clientP.Ping(context.Background(), &grpcstate[i].reqP)
			}
		} else {
			grpcstate[i].clientH = grpc_health_v1.NewHealthClient(conn)
			if grpcstate[i].clientH == nil {
				return nil, fmt.Errorf("unable to create health client %d for %s", i, o.Destination)
			}
			grpcstate[i].reqH = grpc_health_v1.HealthCheckRequest{Service: o.Service}
			if o.Exactly <= 0 {
				_, err = grpcstate[i].clientH.Check(context.Background(), &grpcstate[i].reqH)
			}
		}
		if !o.AllowInitialErrors && err != nil {
			log.Errf("Error in first grpc call (ping = %v) for %s: %v", o.UsePing, o.Destination, err)
			return nil, err
		}
		// Setup the stats for each 'thread'
		grpcstate[i].RetCodes = make(HealthResultMap)
	}
```


### profile和测试结果

```go
	if o.Profiler != "" {
		fc, err := os.Create(o.Profiler + ".cpu")
		if err != nil {
			log.Critf("Unable to create .cpu profile: %v", err)
			return nil, err
		}
		if err = pprof.StartCPUProfile(fc); err != nil {
			log.Critf("Unable to start cpu profile: %v", err)
		}
	}
	total.RunnerResults = r.Run()
	if o.Profiler != "" {
		pprof.StopCPUProfile()
		fm, err := os.Create(o.Profiler + ".mem")
		if err != nil {
			log.Critf("Unable to create .mem profile: %v", err)
			return nil, err
		}
		runtime.GC() // get up-to-date statistics
		if err = pprof.WriteHeapProfile(fm); err != nil {
			log.Critf("Unable to write heap profile: %v", err)
		}
		fm.Close()
		fmt.Printf("Wrote profile data to %s.{cpu|mem}\n", o.Profiler)
	}
	// Numthreads may have reduced
	numThreads = r.Options().NumThreads
	keys := []string{}
	for i := 0; i < numThreads; i++ {
		// Q: is there some copying each time stats[i] is used?
		for k := range grpcstate[i].RetCodes {
			if _, exists := total.RetCodes[k]; !exists {
				keys = append(keys, k)
			}
			total.RetCodes[k] += grpcstate[i].RetCodes[k]
		}
		// TODO: if grpc client needs 'cleanup'/Close like http one, do it on original NumThreads
	}
	// Cleanup state:
	r.Options().ReleaseRunners()
	which := "Health"
	if o.UsePing {
		which = "Ping"
	}
	_, _ = fmt.Fprintf(out, "Jitter: %t\n", total.Jitter)
	for _, k := range keys {
		_, _ = fmt.Fprintf(out, "%s %s : %d\n", which, k, total.RetCodes[k])
	}
	return &total, nil
}
````



## grpcDestination() 方法实现

grpcDestination 方法解析 dest，并根据 dest 是一个主机名、IP地址、`hostname:port` 或 `ip:port` 返回 `dest:port`。如果 dest 是一个无效的主机名或无效的 IP 地址，则返回原始 dest。如果存在 http/https 前缀，将从 destination 中删除，如果在 destination 中没有指定http、https或 `:port`，则端口号被设置为StandardHTTPPort，StandardHTTPSPort用于https，或 DefaultGRPCPort。

```go
func grpcDestination(dest string) (parsedDest string) {
	var port string
  // 从dest中剥离任何无意的http/https方案前缀，并设置端口号。
	// strip any unintentional http/https scheme prefixes from dest
	// and set the port number.
	switch {
	case strings.HasPrefix(dest, fnet.PrefixHTTP):
		parsedDest = strings.TrimSuffix(strings.Replace(dest, fnet.PrefixHTTP, "", 1), "/")
		port = fnet.StandardHTTPPort
		log.Infof("stripping http scheme. grpc destination: %v: grpc port: %s",
			parsedDest, port)
	case strings.HasPrefix(dest, fnet.PrefixHTTPS):
		parsedDest = strings.TrimSuffix(strings.Replace(dest, fnet.PrefixHTTPS, "", 1), "/")
		port = fnet.StandardHTTPSPort
		log.Infof("stripping https scheme. grpc destination: %v. grpc port: %s",
			parsedDest, port)
	default:
		parsedDest = dest
		port = fnet.DefaultGRPCPort
	}
  
  
	if _, _, err := net.SplitHostPort(parsedDest); err == nil {
		return parsedDest
	}
	if ip := net.ParseIP(parsedDest); ip != nil {
		switch {
		case ip.To4() != nil:
			parsedDest = ip.String() + fnet.NormalizePort(port)
			return parsedDest
		case ip.To16() != nil:
			parsedDest = "[" + ip.String() + "]" + fnet.NormalizePort(port)
			return parsedDest
		}
	}
	// parsedDest is in the form of a domain name,
	// append ":port" and return.
	parsedDest += fnet.NormalizePort(port)
	return parsedDest
}
```

三个默认的端口分别是：

```go
	// DefaultGRPCPort is the Fortio gRPC server default port number.
	DefaultGRPCPort = "8079"
	// StandardHTTPPort is the Standard http port number.
	StandardHTTPPort = "80"
	// StandardHTTPSPort is the Standard https port number.
	StandardHTTPSPort = "443"
```

