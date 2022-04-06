---
title: "main.go"
linkTitle: "main.go"
weight: 100
date: 2022-03-28
description: >
  Fortio的main.go实现
---



## main函数入口



```go
// nolint: funlen // well yes it's fairly big and lotsa ifs.
func main() {
	flag.Var(&proxiesFlags, "P",
		"Tcp proxies to run, e.g -P \"localport1 dest_host1:dest_port1\" -P \"[::1]:0 www.google.com:443\" ...")
	flag.Var(&httpMultiFlags, "M", "Http multi proxy to run, e.g -M \"localport1 baseDestURL1 baseDestURL2\" -M ...")
	bincommon.SharedMain(usage)
	if len(os.Args) < 2 {
		usageErr("Error: need at least 1 command parameter")
	}
  // 命令行传进来的第一个参数是fortio命令，其他参数是这个命令的参数
	command := os.Args[1]
	os.Args = append([]string{os.Args[0]}, os.Args[2:]...)
	flag.Parse()
  // help命令
	if *bincommon.HelpFlag {
		usage(os.Stdout)
		os.Exit(0)
	}
  // quit 命令
	if *bincommon.QuietFlag {
		log.SetLogLevelQuiet(log.Error)
	}
	confDir := *bincommon.ConfigDirectoryFlag
	if confDir != "" {
		if _, err := configmap.Setup(flag.CommandLine, confDir); err != nil {
			log.Critf("Unable to watch config/flag changes in %v: %v", confDir, err)
		}
	}
	fnet.ChangeMaxPayloadSize(*newMaxPayloadSizeKb * fnet.KILOBYTE)
	percList, err := stats.ParsePercentiles(*percentilesFlag)
	if err != nil {
		usageErr("Unable to extract percentiles from -p: ", err)
	}
	baseURL := strings.Trim(*baseURLFlag, " \t\n\r/") // remove trailing slash and other whitespace
	sync := strings.TrimSpace(*syncFlag)
	if sync != "" {
		if !ui.Sync(os.Stdout, sync, *dataDirFlag) {
			os.Exit(1)
		}
	}
  
  // 处理各种子命令，有部分命令是作为服务器端的，isServer 参数用来做标记
	isServer := false
	switch command {
	case "curl":
		fortioLoad(true, nil)
	case "nc":
		fortioNC()
	case "load":
		fortioLoad(*curlFlag, percList)
	case "redirect":
		isServer = true
		fhttp.RedirectToHTTPS(*redirectFlag)
	case "report":
		isServer = true
		if *redirectFlag != disabled {
			fhttp.RedirectToHTTPS(*redirectFlag)
		}
		if !ui.Report(baseURL, *echoPortFlag, *dataDirFlag) {
			os.Exit(1) // error already logged
		}
	case "tcp-echo":
		isServer = true
		fnet.TCPEchoServer("tcp-echo", *tcpPortFlag)
		startProxies()
	case "udp-echo":
		isServer = true
		fnet.UDPEchoServer("udp-echo", *udpPortFlag, *udpAsyncFlag)
		startProxies()
	case "proxies":
		if len(flag.Args()) != 0 {
			usageErr("Error: fortio proxies command only takes -P / -M flags")
		}
		isServer = true
		if startProxies() == 0 {
			usageErr("Error: fortio proxies command needs at least one -P / -M flag")
		}
	case "server":
		isServer = true
		if *tcpPortFlag != disabled {
			fnet.TCPEchoServer("tcp-echo", *tcpPortFlag)
		}
		if *udpPortFlag != disabled {
			fnet.UDPEchoServer("udp-echo", *udpPortFlag, *udpAsyncFlag)
		}
		if *grpcPortFlag != disabled {
			fgrpc.PingServer(*grpcPortFlag, *bincommon.CertFlag, *bincommon.KeyFlag, fgrpc.DefaultHealthServiceName, uint32(*maxStreamsFlag))
		}
		if *redirectFlag != disabled {
			fhttp.RedirectToHTTPS(*redirectFlag)
		}
		if !ui.Serve(baseURL, *echoPortFlag, *echoDbgPathFlag, *uiPathFlag, *dataDirFlag, percList) {
			os.Exit(1) // error already logged
		}
		startProxies()
	case "grpcping":
		grpcClient()
	default:
		usageErr("Error: unknown command ", command)
	}
	if isServer {
		if confDir == "" {
			log.Infof("Note: not using dynamic flag watching (use -config to set watch directory)")
		}
		serverLoop(sync)
	}
}
```

