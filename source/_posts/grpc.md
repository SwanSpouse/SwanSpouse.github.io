---
title: go grpc 样例
date: 2018-10-14 16:32:12
tags:
  - go
---

### Go gRPC 基础

通过route_guide($GOPATH/src/google.golang.org/grpc/examples/route_guide/routeguide
)这个样例来了解grpc的使用，主要包括
* 在 .proto 文件中定义service。
* 使用protocol buffer compiler来生成server和client的代码。
* 使用Go gRPC API来写一个简单的样例。

在route_guide样例中，使用protocol buffer 的版本是proto3。

### 关于为什么使用gRPC

route_guide样例是一个地图应用，client可以查询它们路线上的景点、创建线路概要、或者和服务器以及其他client交换如交通状况等信息。

使用gRPC，我们可以在 .proto 文件中定义我们的服务并使用任何gRPC支持的语言来进行实现。 这样使得我们的程序可以跨平台的运行。所有关于不同语言、环境的交互问题，都由gRPC来进行处理。 同时，通过使用protocol buffers，可以简化我们序列化、IDL和接口更新等工作。

### 获取样例代码

样例代码在 grpc/grpc-go/examples/route_guide， 可以使用如下命令来进行获取：

```shell
$ go get google.golang.org/grpc
```

并改变当前目录到 grpc-go/examples/route_guide

```shell
$ cd $GOPATH/src/google.golang.org/grpc/examples/route_guide
```

### 定义service

第一步我们需要使用 protocol buffers 来定义gRPC service 和 方法的request、response。 可以参考 examples/route_guide/routeguide/route_guide.proto 文件

在 .proto 文件中定义 service的名字

``` go
service RouteGuide {
   ...
}
```

然后在service中定义 rpc 方法，同时指定方法的request和response类型。gRPC支持4中service方法。这4种方法在 RouteGuide service中均有使用。

#### A Simple RPC 简单RPC

A Simple RPC: 一个 client request对象对应一个server response对象。

```go
// Obtains the feature at a given position.
rpc GetFeature(Point) returns (Feature) {}
```

#### A Server-side Streaming RPC 服务端流式RPC

A server-side streaming RPC: 一个client request对象对应多个server response对象。 通过在 response 类型之前加上stream 关键字来实现服务端流式方法。

```go
// Obtains the Features available within the given Rectangle.  Results are
// streamed rather than returned at once (e.g. in a response message with a
// repeated field), as the rectangle may cover a large area and contain a
// huge number of features.
rpc ListFeatures(Rectangle) returns (stream Feature) {}
```

#### A Client-side streaming RPC 客户端流式RPC

A client-side streaming RPC: 多个client request对应一个server response。 通过在 request 类型之前加上stream 关键字来实现客户端流式方法。

```go
// Accepts a stream of Points on a route being traversed, returning a
// RouteSummary when traversal is completed.
rpc RecordRoute(stream Point) returns (RouteSummary) {}
```

#### A Bidirectional Streaming RPC  双向流式RPC

A bidirectional streaming RPC: 多个 client request对应多个server response。通过在 request 和 response类型前都加上 stream关键字来实现双向流式RPC。

```go
// Accepts a stream of RouteNotes sent while a route is being traversed,
// while receiving other RouteNotes (e.g. from other users).
rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
```

#### request response 类型定义

.proto 文件中包含了所有 protocol buffer中request和response类型的定义。例如:

```go
// Points are represented as latitude-longitude pairs in the E7 representation
// (degrees multiplied by 10**7 and rounded to the nearest integer).
// Latitudes should be in the range +/- 90 degrees and longitude should be in
// the range +/- 180 degrees (inclusive).
message Point {
  int32 latitude = 1;
  int32 longitude = 2;
}
```

### 生成 client 和 server 代码

接下来我们需要通过 .proto 文件生成gRPC client 和 server 接口。 我们可以使用 gRPC Go 插件来完成这件事情。

```shell
$ protoc --go_out=plugins=grpc:. route_guide.proto
```

通过执行上述命令，会在当前目录中生成 route_guide.pb.go 文件。这个文件中包含了

All the protocol buffer code to populate, serialize, and retrieve our request and response message types
An interface type (or stub) for clients to call with the methods defined in the RouteGuide service.
An interface type for servers to implement, also with the methods defined in the RouteGuide service.

* 所有用于发送、序列化、接收request和response的protocol buffer代码。
* 所有调用 RouteGuide 服务的客户端接口。
* 所有 server 需要实现的 RouteGuide 的接口。


### 创建Server

实现RouteGuide服务，主要有两部分工作：

* 实现服务中定义的所有接口: 在方法中对服务真正要做的工作进行处理。
* 启动一个gRPC server 来监听并分发所有来自clients的request。

server的相关代码在 grpc-go/examples/route_guide/server/server.go 中。

#### 实现 RouteGuide

如下所示，routeGuideServer 实现了 RouteGuideServer 接口

```golang
type routeGuideServer struct {
        ...
}
...

func (s *routeGuideServer) GetFeature(ctx context.Context, point *pb.Point) (*pb.Feature, error) {
        ...
}
...

func (s *routeGuideServer) ListFeatures(rect *pb.Rectangle, stream pb.RouteGuide_ListFeaturesServer) error {
        ...
}
...

func (s *routeGuideServer) RecordRoute(stream pb.RouteGuide_RecordRouteServer) error {
        ...
}
...

func (s *routeGuideServer) RouteChat(stream pb.RouteGuide_RouteChatServer) error {
        ...
}
...
```

#### Simple RPC

```go
func (s *routeGuideServer) GetFeature(ctx context.Context, point *pb.Point) (*pb.Feature, error) {
	for _, feature := range s.savedFeatures {
		if proto.Equal(feature.Location, point) {
			return feature, nil
		}
	}
	// No feature was found, return an unnamed feature
	return &pb.Feature{"", point}, nil
}
```

该方法的输入参数有两个: RPC的上下文对象和 client的 Point protocol buffer request。返回了一个带有response 信息的Feature protocol buffer对象和error。 在这个方法中，server返回合适的Feature对象，同时返回一个空error来通知gRPC，server已经完成本次RPC请求，Feature可以被发送回client。

#### Server-side streaming RPC

在服务器端流式RPC中，我们需要返回多个Feature给client。

```go
func (s *routeGuideServer) ListFeatures(rect *pb.Rectangle, stream pb.RouteGuide_ListFeaturesServer) error {
	for _, feature := range s.savedFeatures {
		if inRange(feature.Location, rect) {
			if err := stream.Send(feature); err != nil {
				return err
			}
		}
	}
	return nil
}
```

在上述方法中，server端将所有符合条件的Feature返回给client，使用Send()方法将它们写入 RouteGuide_ListFeaturesServer 中。 在最后，和Simple RPC中的一样，server返回一个空error来通知gRPC已经写完response的数据了。如果在调用过程中出现了任何错误，serve会返回一个非空的error，gRPC会将error转换成一个合适的RPC状态返回给client。

#### Client-side streaming RPC

在RecordRoute中，server从client端接收到多个Point对象并返回一个RouteSummary对象。如下代码所示，在本次请求中只有一个RouteGuide_RecordRouteServer对象，使用RouteGuide_RecordRouteServer中的Recv()方法接受对象，并通过SendAndClose()方法返回结果。

```go
func (s *routeGuideServer) RecordRoute(stream pb.RouteGuide_RecordRouteServer) error {
	var pointCount, featureCount, distance int32
	var lastPoint *pb.Point
	startTime := time.Now()
	for {
		point, err := stream.Recv()
		if err == io.EOF {
			endTime := time.Now()
			return stream.SendAndClose(&pb.RouteSummary{
				PointCount:   pointCount,
				FeatureCount: featureCount,
				Distance:     distance,
				ElapsedTime:  int32(endTime.Sub(startTime).Seconds()),
			})
		}
		if err != nil {
			return err
		}
		pointCount++
		for _, feature := range s.savedFeatures {
			if proto.Equal(feature.Location, point) {
				featureCount++
			}
		}
		if lastPoint != nil {
			distance += calcDistance(lastPoint, point)
		}
		lastPoint = point
	}
}
```

在上述方法中，server使用RouteGuide_RecordRouteServers Recv()来反复读取client的请求，server需要在每次调用Recv()方法之后对返回的error进行检查，如果err为空，则说明当前的数据流是好的，并且可以继续从中读取数据；直到 error == io.EOF， 说明stream中已经没有数据，server可以返回RouteSummary。如果error 还有其他的值，server端会返回error，和Server-side streaming RPC中一样，error会被gRPC转换成特殊的RPC状态，返回给client。

#### Bidirectional streaming RPC

```go
func (s *routeGuideServer) RouteChat(stream pb.RouteGuide_RouteChatServer) error {
	for {
		in, err := stream.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			return err
		}
		key := serialize(in.Location)
    ... // look for notes to be sent to client
		for _, note := range s.routeNotes[key] {
			if err := stream.Send(note); err != nil {
				return err
			}
		}
	}
}
```

在双向流式RPC中读写request和response的方法和其方法类似。

#### Starting the server

在实现了所有方法时候，我们需要启动一个gRPC server以便client可以进行调用。

```go
flag.Parse()
lis, err := net.Listen("tcp", fmt.Sprintf("localhost:%d", *port))
if err != nil {
        log.Fatalf("failed to listen: %v", err)
}
grpcServer := grpc.NewServer()
pb.RegisterRouteGuideServer(grpcServer, &routeGuideServer{})
... // determine whether to use TLS
grpcServer.Serve(lis)

```

* 首先指定需要进行监听的端口号, err := net.Listen("tcp", fmt.Sprintf("localhost:%d", \*port))。
* 使用 grpc.NewServer()创建一个gPRC server的实例。
* 注册实现的gRPC server 。
* 在server端调用Serve()方法开始提供服务，直到进程被杀掉或者调用了Stop()方法。

### 创建 client

client 的代码在 grpc-go/examples/route_guide/client/client.go。

#### 创建stub

为了调用service的方法，我们首先需要通过grpc.Dail()创建一个gRPC通道来和server进行通信。

```go
conn, err := grpc.Dial(*serverAddr)
if err != nil {
    ...
}
defer conn.Close()
```

如果service需要的话，可以在DialOptions中设定授权信息，

当 gRPC 通过创建完之后，我们需要一个client stub 来进行RPC调用。 可以使用通过 .proto 文件生成的pb包中的NewRouteGuideClient方法来创建client stub。

```go
client := pb.NewRouteGuideClient(conn)
```

#### Calling service methods

在gRPC-GO中，RPC调用都是同步阻塞模式。这意味着，client进行RPC调用之后会一直阻塞，直到server返回结果或者错误。

#### Simple RPC

```go
feature, err := client.GetFeature(ctx, &pb.Point{409146138, -746188906})
if err != nil {
        ...
}
```

如上述代码所示，通过client stub调用GetFeature()方法。在输入参数中，除request protocol buffer对象外，还有一个context.Context对象，通过context对象，可以在需要的时候改变RPC的一些参数。如超时时间等。 如果方法调用没有返回错误，client可以在response的第一个返回值中读取server返回的结果。

#### Server-side streaming RPC

```go
rect := &pb.Rectangle{ ... }  // initialize a pb.Rectangle
stream, err := client.ListFeatures(ctx, rect)
if err != nil {
    ...
}
for {
    feature, err := stream.Recv()
    if err == io.EOF {
        break
    }
    if err != nil {
        log.Fatalf("%v.ListFeatures(_) = _, %v", client, err)
    }
    log.Println(feature)
}
```

和simple RPC一样，输入参数是一个context和一个request对象。但是返回值为RouteGuide_ListFeaturesClient，通过RouteGuide_ListFeaturesClient的 Recv()方法，client 可以获取server所有的response。同理， client在调用Recv()方法之后，需要对返回的error进行检查。

#### Client-side streaming RPC

```go
// Create a random number of random points
r := rand.New(rand.NewSource(time.Now().UnixNano()))
pointCount := int(r.Int31n(100)) + 2 // Traverse at least two points
var points []*pb.Point
for i := 0; i < pointCount; i++ {
	points = append(points, randomPoint(r))
}
log.Printf("Traversing %d points.", len(points))
stream, err := client.RecordRoute(ctx)
if err != nil {
	log.Fatalf("%v.RecordRoute(_) = _, %v", client, err)
}
for _, point := range points {
	if err := stream.Send(point); err != nil {
		log.Fatalf("%v.Send(%v) = %v", stream, point, err)
	}
}
reply, err := stream.CloseAndRecv()
if err != nil {
	log.Fatalf("%v.CloseAndRecv() got error %v, want %v", stream, err, nil)
}
log.Printf("Route summary: %v", reply)
```

client 通过RouteGuide_RecordRouteClient的Send()方法向server发送request，当request发送结束之后，需要调用CloseAndRecv()方法来通知gRPC client已经发送完所有的request，并开始等待接受response。 当client接收到CloseAndRecv()的返回值后，如果error是空， 那么CloseAndRecv()就是一个有效的server返回值。

#### Bidirectional streaming RPC

```go
stream, err := client.RouteChat(ctx)
waitc := make(chan struct{})
go func() {
	for {
		in, err := stream.Recv()
		if err == io.EOF {
			// read done.
			close(waitc)
			return
		}
		if err != nil {
			log.Fatalf("Failed to receive a note : %v", err)
		}
		log.Printf("Got message %s at point(%d, %d)", in.Message, in.Location.Latitude, in.Location.Longitude)
	}
}()
for _, note := range notes {
	if err := stream.Send(note); err != nil {
		log.Fatalf("Failed to send a note: %v", err)
	}
}
stream.CloseSend()
<-waitc
```

在双向流式RPC中，除了调用CloseSend()方法，读写请求的方法和client-side streaming中的方法类似。在双向流式RPC中，读写请求是完全独立的。

### 运行sample

```shell
cd $GOPATH/src/google.golang.org/grpc/examples/route_guide
```

启动server，启动client

```shell
$ go run server/server.go
$ go run client/client.go

```

### reference
* https://github.com/grpc/grpc-go/blob/master/examples/gotutorial.md
