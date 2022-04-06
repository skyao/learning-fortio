---
title: "health测试"
linkTitle: "health测试"
weight: 200
date: 2022-03-28
description: >
  Fortio的gRPC health测试
---





#### 执行 health 测试

```go
			grpcstate[i].clientH = grpc_health_v1.NewHealthClient(conn)
			if grpcstate[i].clientH == nil {
				return nil, fmt.Errorf("unable to create health client %d for %s", i, o.Destination)
			}
			grpcstate[i].reqH = grpc_health_v1.HealthCheckRequest{Service: o.Service}
			if o.Exactly <= 0 {
				_, err = grpcstate[i].clientH.Check(context.Background(), &grpcstate[i].reqH)
			}
```
