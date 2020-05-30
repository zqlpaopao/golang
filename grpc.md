# RPC 服务端的创建



## 引言

我们介绍 grpc quick start 时，通过快速启动一个 grpc server 端和 client 端，然后以 rpc 调用的方式输出一个 hello world。 背后的调用链路是什么，如何处理的



```go
func main() {
    lis, err := net.Listen("tcp", port)
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    s := grpc.NewServer()
    pb.RegisterGreeterServer(s, &server{})
    if err := s.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}

```

server端链接建立的重要三个步骤

1. 创建server
2. server的注册
3. 调用server监听端口并处理请求

## ==创建server==

```
func NewServer(opt ...ServerOption) *Server {
    opts := defaultServerOptions
    for _, o := range opt {
        o.apply(&opts)
    }
    s := &Server{
        lis:    make(map[net.Listener]bool),
        opts:   opts,
        conns:  make(map[transport.ServerTransport]bool),
        m:      make(map[string]*service),
        quit:   grpcsync.NewEvent(),
        done:   grpcsync.NewEvent(),
        czData: new(channelzData),
    }
    s.cv = sync.NewCond(&s.mu)
    if EnableTracing {
        _, file, line, _ := runtime.Caller(1)
        s.events = trace.NewEventLog("grpc.Server", fmt.Sprintf("%s:%d", file, line))
    }
    if channelz.IsOn() {
        s.channelzID = channelz.RegisterServer(&channelzServer{s}, "")
    }
    return s
}
```

创建结构体和赋值操作

server的结构体

```
// Server is a gRPC server to serve RPC requests.
type Server struct {
    // serverOptions 就是描述协议的各种参数选项，包括发送和接收的消息大小、buffer大小等等各种，跟 http Headers 类似，我们这里就暂时先不管
    opts serverOptions
    // 一个互斥锁
    mu     sync.Mutex // guards following
    // listener map
    lis    map[net.Listener]bool
    // connections map
    conns  map[transport.ServerTransport]bool
    // server 是否在处理请求的一个状态位
    serve  bool
    drain  bool
    cv     *sync.Cond          // signaled when connections close for GracefulStop
    // service map
    m      map[string]*service // service name -> service info
    events trace.EventLog
    quit               *grpcsync.Event
    done               *grpcsync.Event
    channelzRemoveOnce sync.Once
    serveWG            sync.WaitGroup // counts active Serve goroutines for GracefulStop
    channelzID int64 // channelz unique identification number
    czData     *channelzData
}
```

<font color=red>比较重要的无非就是三个 map 表分别用来存放多个 listener 、connection 和 service</font>。其他字段都是为了实现协议描述或者并发控制的功能。我们重点关注下

```
m      map[string]*service 

type service struct {
    server interface{} // the server for service methods
    md     map[string]*MethodDesc
    sd     map[string]*StreamDesc
    mdata  interface{}
}
```

这个结构，service 中主要包含了 MethodDesc 和 StreamDesc 这两个 map

