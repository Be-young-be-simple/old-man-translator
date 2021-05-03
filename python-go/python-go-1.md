原文：https://www.ardanlabs.com/blog/2020/06/python-go-grpc.html （译：[@xwjdsh](https://github.com/xwjdsh)）

---

## Python 和 Go：第一部分 - gRPC

### 引言

你可以用小刀来拧紧螺丝，但用螺丝刀会更好，也更安全。像工具一样，编程语言也倾向于解决其设计要解决的问题。

Go 适合编写高吞吐量的服务，而 Python 在数据科学领域大放异彩。在接下来的一系列的文章中，我们将探讨几种 Go 和 Python 通信的方式，使它们做各自更适合的部分。

*注意：在项目中使用多种语言是有代价的。如果你可以只用 Go 或 Python 编写所有内容 - 那一定要这样做。但是，在某些情况下，使用正确的语言来完成工作会减少在可读性，维护成本以及性能方面的总体开销。*

在这篇文章中，我们将学习使用 [grpc]() 来完成在 Go 和 Python 程序间的通信。这篇文章假设你具有 Go 和 Python 的基础知识。

### gRPC 概述

gRPC 是 Google 开发的远程过程调用（RPC）框架。它使用 [Protocol Buffers](https://developers.google.com/protocol-buffers) 作为序列化格式，并使用 [HTTP2 ](https://en.wikipedia.org/wiki/HTTP/2)作为传输介质。通过使用这两种完善的技术，你可以获得许多有效的信息和工具。我咨询过的许多公司都使用 gRPC 来连接内部服务。

使用 Protocol Buffers 的另一个优点是，你只需编写一次消息定义，然后就可以从同一来源生成各个语言的实现。这意味着可以用不同的编程语言来编写各个服务，并且各个服务有一致的消息格式。

Protocol Buffers 也是一种高效的二进制格式：你可以获得更少的序列化时间和传输字节数，仅此一项就可以节省金钱。在我的机器上进行基准测试，与 JSON 相比，序列化时间要快 7.5 倍左右，生成的数据要小 4 倍左右。



### 示例：异常检测

异常检测是一种在数据中查找可疑值的方法。现代系统会从其服务中收集大量指标，很难通过简单的阈值来找到出现故障的服务，这意味着会在凌晨 2 点把值班的开发人员叫醒多次...

首先我们会实现一个 Go 的指标收集服务，然后通过 gRPC，我们把这些指标发送到一个 Python 的服务，它会基于这些指标做异常检测。

### 项目结构

在此项目中，我们采用一种简单的方法，即在源代码中将 Go 作为主项目，将 Python 作为子项目。

```bash
.
├── client.go
├── gen.go
├── go.mod
├── go.sum
├── outliers.proto
├── pb
│   └── outliers.pb.go
└── py
    ├── Makefile
    ├── outliers_pb2_grpc.py
    ├── outliers_pb2.py
    ├── requirements.txt
    └── server.py
```

以上展示了项目结构，这个项目使用 [Go modules](https://blog.golang.org/using-go-modules)，模块名为 `github.com/ardanlabs/python-go/grpc`，我们会在一些地方引用该模块名。

```go
01 module github.com/ardanlabs/python-go/grpc
02
03 go 1.14
04
05 require (
06     github.com/golang/protobuf v1.4.2
07     google.golang.org/grpc v1.29.1
08     google.golang.org/protobuf v1.24.0
09 )
```

以上是项目的 go.mod 文件内容，你可以看到 01 行是定义模块名称的地方。

### 定义消息和服务

在 gRPC 中，你首先要编写一个 `.proto` 文件，该文件定义要发送的消息和 RPC 方法。

```protobuf
01 syntax = "proto3";
02 import "google/protobuf/timestamp.proto";
03 package pb;
04
05 option go_package = "github.com/ardanlabs/python-go/grpc/pb";
06
07 message Metric {
08    google.protobuf.Timestamp time = 1;
09    string name = 2;
10    double value = 3;
11 }
12
13 message OutliersRequest {
14    repeated Metric metrics = 1;
15 }
16
17 message OutliersResponse {
18    repeated int32 indices = 1;
19 }
20
21 service Outliers {
22    rpc Detect(OutliersRequest) returns (OutliersResponse) {}
23 }
```

以上是 `outliers.proto` 的文件内容，需要注意的是第 02 行，我们导入了 Protocol Buffers 中定义的 `timestamp`，然后在 05 行定义了完整的 Go 包名 - `github.com/ardanlabs/python-go/grpc/pb`。

指标是对资源使用情况的度量，用于监视和诊断系统。我们在第 07 行定义了 `Metric`，它具有时间戳，名称 (例如 "CPU") 和对应的浮点数值。例如，我们可以说在 `2020-03-14T12:30:14` 我们测得的 CPU 利用率为 `41.2％`。

每个 RPC 方法都有一个（或多个）输入类型和一个输出类型，我们的方法 `Detect`（第 22 行）使用 `OutliersRequest` 消息类型（第 13 行）作为输入，并使用 `OutliersResponse` 消息类型（第 17 行）作为输出。

`OutliersRequest` 消息类型是 `Metric` 的列表/切片，`OutliersResponse` 消息类型是发现的异常值索引的列表/切片。例如，如果我们具有 `[1、2、100、1、3、200、1]` 的值，则结果将是 `[2、5]`，这是异常值 100 和 200 的索引。

### Python 服务

在本节中，我们将介绍 Python 服务的代码。

```bash
.
├── client.go
├── gen.go
├── go.mod
├── go.sum
├── outliers.proto
├── pb
│   └── outliers.pb.go
└── py
    ├── Makefile
    ├── outliers_pb2_grpc.py
    ├── outliers_pb2.py
    ├── requirements.txt
    └── server.py
```

以上显示了 Python 服务的代码位于项目根目录下的 `py` 目录中。

要生成 proto 对应的 Python 代码，你需要安装 `protoc` 编译器，你可以在 [这里](https://developers.google.com/protocol-buffers/docs/downloads) 下载。你还可以使用软件包管理器来安装。（例如 `apt-get`, `brew` …）

安装编译器后，你还需要安装 Python 的 `grpcio-tools` 包。

*注意：我强烈建议你为所有 Python 项目使用虚拟环境。阅读 [此内容](https://docs.python.org/3/tutorial/venv.html) 以了解更多信息。*

```bash
$ cat requirements.txt

OUTPUT:
grpcio-tools==1.29.0
numpy==1.18.4

$ python -m pip install -r requirements.txt
```

以上展示了如何检查和安装我们 Python 项目的外部依赖，`requirements.txt` 指定了 Python 项目的外部依赖项，类似于`go.mod` 为 Go 项目指定依赖项。

从 `cat` 命令的输出中可以看到，我们需要两个外部依赖项：`grpcio-tools` 和 [numpy](https://numpy.org/)。好的做法是将此文件置于源代码管理中，并始终对依赖项进行版本控制（例如 `numpy == 1.18.4`），类似于 Go 项目使用 `go.mod` 进行的操作。

一旦安装好了依赖项，就可以生成 `proto` 对应的Python 代码了。

```bash
$ python -m grpc_tools.protoc \
    -I.. --python_out=. --grpc_python_out=. \
    ../outliers.proto
```

以上命令展示了如何生成支持 gRPC 的 Python 代码，我们来分解一下这个长命令：

* `python -m grpc_tools.protoc` 将 `grpc_tools.protoc` 模块作为脚本运行。
* `-I ..` 告诉工具在哪里找到 `.proto` 文件。
* `--python_out=.` 告诉工具在当前目录中生成 protocol buffers 序列化代码。
* `--grpc_python_out=.` 告诉工具在当前目录中生成 gRPC 代码。
* `../outliers.proto` 是我们定义 protocol buffers + gRPC 的文件名。

运行此 Python 命令没有任何输出，最后，你会看到两个新文件：`outliers_pb2.py`（protocol buffers 代码）和 `outliers_pb2_grpc.py`（gRPC 客户端和服务器代码）。

*注意：我通常使用 `Makefile` 在 Python 项目中自动执行任务，并创建一条 `make` 规则来运行此命令。我将生成的文件添加到源代码管理中，以便部署计算机无需安装 `protoc` 编译器。*

要编写 Python 服务，你需要继承从 `outliers_pb2_grpc.py` 中定义的 `OutliersServicer` 并重写 `Detect` 方法。我们将使用 `numpy` 包，并使用一种简单的方法来选择所有与 [均值](https://en.wikipedia.org/wiki/Mean) 相差两个 [标准差](https://en.wikipedia.org/wiki/Standard_deviation) 以上的值。

```python
01 import logging
02 from concurrent.futures import ThreadPoolExecutor
03
04 import grpc
05 import numpy as np
06
07 from outliers_pb2 import OutliersResponse
08 from outliers_pb2_grpc import OutliersServicer, add_OutliersServicer_to_server
09
10
11 def find_outliers(data: np.ndarray):
12     """Return indices where values more than 2 standard deviations from mean"""
13     out = np.where(np.abs(data - data.mean()) > 2 * data.std())
14     # np.where returns a tuple for each dimension, we want the 1st element
15     return out[0]
16
17
18 class OutliersServer(OutliersServicer):
19     def Detect(self, request, context):
20          logging.info('detect request size: %d', len(request.metrics))
21          # Convert metrics to numpy array of values only
22          data = np.fromiter((m.value for m in request.metrics), dtype='float64')
23          indices = find_outliers(data)
24          logging.info('found %d outliers', len(indices))
25          resp = OutliersResponse(indices=indices)
26          return resp
27
28
29 if __name__ == '__main__':
30     logging.basicConfig(
31          level=logging.INFO,
32          format='%(asctime)s - %(levelname)s - %(message)s',
33	)
34     server = grpc.server(ThreadPoolExecutor())
35     add_OutliersServicer_to_server(OutliersServer(), server)
36     port = 9999
37     server.add_insecure_port(f'[::]:{port}')
38     server.start()
39     logging.info('server ready on port %r', port)
40     server.wait_for_termination()
```

以上显示了 `server.py` 文件中的代码，这就是我们编写 Python 服务的全部代码。在第 19 行中，我们覆写了生成的`OutlierServicer `中的 `Detect` 方法，编写了实际的异常值检测代码。在第 34 行，我们创建了一个使用 `ThreadPoolExecutor` 并行运行请求的 gRPC 服务器，在第 35 行，我们注册了 `OutliersServer` 来处理服务器的请求。

```bash
$ python server.py

OUTPUT:
2020-05-23 13:45:12,578 - INFO - server ready on port 9999
```

执行以上命令，我们成功运行了 Python 服务。

### Go 客户端

现在我们已经运行了 Python 服务，我们可以编写将与其通信的 Go 客户端。

我们将从生成支持 gRPC 的 Go 代码开始。为了使这个过程自动化，我通常使用 `go:generate` 命令创建一个名为 `gen.go` 的文件来生成。

你需要下载 `github.com/golang/protobuf/protoc-gen-go` 模块，这是 Go 的 gRPC 插件。

```go
01 package main
02
03 //go:generate mkdir -p pb
04 //go:generate protoc --go_out=plugins=grpc:pb --go_opt=paths=source_relative outliers.proto
```

以上是 `gen.go` 文件内容，我们能看到是如何使用 `go:generate` 来生成 gRPC 的 Go 代码。

让我们分解 04 行的命令，该命令生成 Go 代码：

* `protoc` 是 protocol buffer 编译器。
* `--go-out=plugins=grpc:pb` 告诉 `protoc` 使用 gRPC 插件并将文件放置在 `pb` 目录中。
* `--go_opt=source_relative` 告诉 `protoc` 在相对于当前目录的 `pb` 目录中生成代码。
* `outliers.proto` 是我们定义 protocol buffers + gRPC 的文件名。

当你运行 `go generate` 命令，应该看不到任何输出，但是 `pb` 目录中将有一个名为 `outliers.pb.go` 的新文件。

```bash
.
├── client.go
├── gen.go
├── go.mod
├── go.sum
├── outliers.proto
├── pb
│   └── outliers.pb.go
└── py
    ├── Makefile
    ├── outliers_pb2_grpc.py
    ├── outliers_pb2.py
    ├── requirements.txt
    └── server.py
```

以上显示了 `pb` 目录和 `go generate` 调用后生成的新文件 `outliers.pb.go`。我将 `pb` 目录添加到源代码管理中，因此如果将项目克隆到其他的计算机上，则该项目可以直接运行而无需在该计算机上再安装 `protoc`。

现在，我们可以编译并运行 Go 客户端。

```go
01 package main
02
03 import (
04     "context"
05     "log"
06     "math/rand"
07     "time"
08
09     "github.com/ardanlabs/python-go/grpc/pb"
10     "google.golang.org/grpc"
11     pbtime "google.golang.org/protobuf/types/known/timestamppb"
12 )
13
14 func main() {
15     addr := "localhost:9999"
16     conn, err := grpc.Dial(addr, grpc.WithInsecure(), grpc.WithBlock())
17     if err != nil {
18          log.Fatal(err)
19     }
20     defer conn.Close()
21
22     client := pb.NewOutliersClient(conn)
23     req := pb.OutliersRequest{
24          Metrics: dummyData(),
25     }
26
27     resp, err := client.Detect(context.Background(), &req)
28     if err != nil {
29          log.Fatal(err)
30     }
31     log.Printf("outliers at: %v", resp.Indices)
32 }
33
34 func dummyData() []*pb.Metric {
35     const size = 1000
36     out := make([]*pb.Metric, size)
37     t := time.Date(2020, 5, 22, 14, 13, 11, 0, time.UTC)
38     for i := 0; i < size; i++ {
39          m := pb.Metric{
40               Time: Timestamp(t),
41               Name: "CPU",
42               // Normally we're below 40% CPU utilization
43               Value: rand.Float64() * 40,
44          }
45          out[i] = &m
46          t.Add(time.Second)
47     }
48     // Create some outliers
49     out[7].Value = 97.3
50     out[113].Value = 92.1
51     out[835].Value = 93.2
52     return out
53 }
54
55 // Timestamp converts time.Time to protobuf *Timestamp
56 func Timestamp(t time.Time) *pbtime.Timestamp {
57     return &pbtime.Timestamp {
58         Seconds: t.Unix(),
59         Nanos: int32(t.Nanosecond()),
60     }
61 }
```

上面是 `client.go` 的代码。代码在第 23 行的 `OutliersRequest` 中填充了一些虚拟数据（由第 34 行的 `dummyData` 函数生成），然后在第 27 行调用 Python 服务，返回一个 `OutliersResponse` 值。

让我们进一步分解代码：

* 在第 16 行，我们使用 `WithInsecure` 选项来连接到 Python 服务，因为我们编写的 Python 服务不支持 `HTTPS`。
* 在第 22 行，我们使用在第 16 行建立的连接创建一个新的 `OutliersClient`。
* 在第 23 行，我们构造了 gPRC 请求。
* 在第 27 行，我们执行了 gRPC 调用。每个 gRPC 调用都有一个 `context.Context` 作为第一个参数，允许你控制超时和取消。
* gRPC 拥有自己的 `Timestamp` 结构实现。在第 56 行，我们写了一个工具函数，可将 Go 的 `time.Time` 值转换为gRPC `Timestamp` 值。

```bash
$ go run client.go

OUTPUT:
2020/05/23 14:07:18 outliers at: [7 113 835]
```

然后我们运行 Go 客户端，这里假设和 Python 服务器在同一台计算机上运行。

### 总结

gRPC 使得将消息从一项服务传递到另一项服务变得容易且安全。你可以维护一个定义所有数据类型和方法的地方，并且gRPC 框架具有出色的工具和最佳实践。

所有代码：`outliers.proto`，`py/server.py` 和 `client.go` 少于 100 行。你可以在 [此处](https://github.com/ardanlabs/python-go/tree/master/grpc) 查看项目代码。

gRPC 还有很多功能，例如超时，负载平衡，TLS 和流式传输。我强烈建议你浏览 [官方网站](https://grpc.io/)，阅读文档并试试它提供的示例。

在本系列的下一篇文章中，我们将调换角色，让 Python 调用 Go 服务。

