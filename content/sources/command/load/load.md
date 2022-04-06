---
title: "load command"
linkTitle: "load command"
weight: 100
date: 2022-03-28
description: >
  Fortio 的 load 命令
---





## 入口

Load 命令的入口在main函数中：

```bash
var (
curlFlag   = flag.Bool("curl", false, "Just fetch the content once")
percentilesFlag = flag.String("p", "50,75,90,99,99.9", "List of pXX to calculate")
)

func main() {
	percList, err := stats.ParsePercentiles(*percentilesFlag)
	......
	
	isServer := false
	switch command {
	......
	case "curl":
		fortioLoad(true, nil)
	case "load":
		fortioLoad(*curlFlag, percList)
	......
}
```

load 命令和 curl 命令的实现是相同的，只是 curl 命令只执行一次请求来获取内容。

## fortioLoad() 的实现

### curl 命令

```go
// nolint: funlen, gocognit // maybe refactor/shorten later.
func fortioLoad(justCurl bool, percList []float64) {
	if len(flag.Args()) != 1 {
		usageErr("Error: fortio load/curl needs a url or destination")
	}
	httpOpts := bincommon.SharedHTTPOptions()
	if justCurl {
    // 对于 curl 命令，实现很简单
		bincommon.FetchURL(httpOpts)
		return
	}
	......
}
```

以下是 load 命令。

### 处理各种参数

先处理各种参数：
  
```go
	url := httpOpts.URL
	prevGoMaxProcs := runtime.GOMAXPROCS(*goMaxProcsFlag)
	out := os.Stderr
	qps := *qpsFlag // TODO possibly use translated <=0 to "max" from results/options normalization in periodic/
	if *calcQPS {
		if *exactlyFlag == 0 || *durationFlag <= 0 {
			usageErr("Error: can't use `-calc-qps` without also specifying `-n` and `-t`")
		}
		qps = float64(*exactlyFlag) / (*durationFlag).Seconds()
		log.LogVf("Calculated QPS to do %d request in %v: %f", *exactlyFlag, *durationFlag, qps)
	}
	_, _ = fmt.Fprintf(out, "Fortio %s running at %g queries per second, %d->%d procs",
		version.Short(), qps, prevGoMaxProcs, runtime.GOMAXPROCS(0))
	if *exactlyFlag > 0 {
		_, _ = fmt.Fprintf(out, ", for %d calls: %s\n", *exactlyFlag, url)
	} else {
		if *durationFlag <= 0 {
			// Infinite mode is determined by having a negative duration value
			*durationFlag = -1
			_, _ = fmt.Fprintf(out, ", until interrupted: %s\n", url)
		} else {
			_, _ = fmt.Fprintf(out, ", for %v: %s\n", *durationFlag, url)
		}
	}
	if qps <= 0 {
		qps = -1 // 0==unitialized struct == default duration, -1 (0 for flag) is max
	}
	labels := *labelsFlag
	if labels == "" {
		hname, _ := os.Hostname()
		shortURL := url
		for _, p := range []string{"https://", "http://"} {
			if strings.HasPrefix(url, p) {
				shortURL = url[len(p):]
				break
			}
		}
		labels = shortURL + " , " + strings.SplitN(hname, ".", 2)[0]
		log.LogVf("Generated Labels: %s", labels)
	}
```
	
### 准备RunnerOptions
	
```go
	ro := periodic.RunnerOptions{
		QPS:         qps,
		Duration:    *durationFlag,
		NumThreads:  *numThreadsFlag,
		Percentiles: percList,
		Resolution:  *resolutionFlag,
		Out:         out,
		Labels:      labels,
		Exactly:     *exactlyFlag,
		Jitter:      *jitterFlag,
		Uniform:     *uniformFlag,
		RunID:       *bincommon.RunIDFlag,
		Offset:      *offsetFlag,
	}
	err := ro.AddAccessLogger(*accessLogFileFlag, *accessLogFileFormat)
	if err != nil {
		// Error already logged.
		os.Exit(1)
	}
```
	
### 运行测试

#### gRPC测试
	
```go
	var res periodic.HasRunnerResult
	if *grpcFlag {
		o := fgrpc.GRPCRunnerOptions{
			RunnerOptions:      ro,
			Destination:        url,
			CACert:             *bincommon.CACertFlag,
			Insecure:           bincommon.TLSInsecure(),
			Service:            *healthSvcFlag,
			Streams:            *streamsFlag,
			AllowInitialErrors: *allowInitialErrorsFlag,
			Payload:            httpOpts.PayloadString(),
			Delay:              *pingDelayFlag,
			UsePing:            *doPingLoadFlag,
			UnixDomainSocket:   httpOpts.UnixDomainSocket,
		}
		res, err = fgrpc.RunGRPCTest(&o)
	} 
```

#### TCP测试

```go
    else if strings.HasPrefix(url, tcprunner.TCPURLPrefix) {
		o := tcprunner.RunnerOptions{
			RunnerOptions: ro,
		}
		o.ReqTimeout = httpOpts.HTTPReqTimeOut
		o.Destination = url
		o.Payload = httpOpts.Payload
		res, err = tcprunner.RunTCPTest(&o)
	}
```

#### UDP测试

```go
  else if strings.HasPrefix(url, udprunner.UDPURLPrefix) {
		o := udprunner.RunnerOptions{
			RunnerOptions: ro,
		}
		o.ReqTimeout = *udpTimeoutFlag
		o.Destination = url
		o.Payload = httpOpts.Payload
		res, err = udprunner.RunUDPTest(&o)
	}
}
```

#### HTTP测试

```go
  else {
		o := fhttp.HTTPRunnerOptions{
			HTTPOptions:        *httpOpts,
			RunnerOptions:      ro,
			Profiler:           *profileFlag,
			AllowInitialErrors: *allowInitialErrorsFlag,
			AbortOn:            *abortOnFlag,
		}
		res, err = fhttp.RunHTTPTest(&o)
	}
```

### 处理测试结果

```go
	if err != nil {
		_, _ = fmt.Fprintf(out, "Aborting because of %v\n", err)
		os.Exit(1)
	}
	rr := res.Result()
	warmup := *numThreadsFlag
	if ro.Exactly > 0 {
		warmup = 0
	}
	_, _ = fmt.Fprintf(out, "All done %d calls (plus %d warmup) %.3f ms avg, %.1f qps\n",
		rr.DurationHistogram.Count,
		warmup,
		1000.*rr.DurationHistogram.Avg,
		rr.ActualQPS)
	jsonFileName := *jsonFlag
	if *autoSaveFlag || len(jsonFileName) > 0 { //nolint: nestif // but probably should breakup this function
		var j []byte
		j, err = json.MarshalIndent(res, "", "  ")
		if err != nil {
			log.Fatalf("Unable to json serialize result: %v", err)
		}
		var f *os.File
		if jsonFileName == "-" {
			f = os.Stdout
			jsonFileName = "stdout"
		} else {
			if len(jsonFileName) == 0 {
				jsonFileName = path.Join(*dataDirFlag, rr.ID()+".json")
			}
			f, err = os.Create(jsonFileName)
			if err != nil {
				log.Fatalf("Unable to create %s: %v", jsonFileName, err)
			}
		}
		n, err := f.Write(append(j, '\n'))
		if err != nil {
			log.Fatalf("Unable to write json to %s: %v", jsonFileName, err)
		}
		if f != os.Stdout {
			err := f.Close()
			if err != nil {
				log.Fatalf("Close error for %s: %v", jsonFileName, err)
			}
		}
		_, _ = fmt.Fprintf(out, "Successfully wrote %d bytes of Json data to %s\n", n, jsonFileName)
	}
}
```

