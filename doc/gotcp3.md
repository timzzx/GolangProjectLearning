# golang tcp服务-3(github发布自己的包和chat项目初始化)

> 前面写了一个简单的tcp服务包，现在准备利用这个包完成一个聊天功能，正式项目这个包不适合使用，只是为了学习了解golang 开发一个完整的tcp项目

## tnet包发布（遇到的问题）
tnet包[地址](https://github.com/timzzx/tnet)

问题：之前发布tnet包时候go mod init github/timzzx/tnet写成了这样了，下面会报错
```
root@tdev:/home/code/tnet-chat# go get github.com/timzzx/tnet
go: github.com/timzzx/tnet@v0.0.0-20230227150359-57595e9fc47a: parsing go.mod:
        module declares its path as: github/timzzx/tnet
                but was required as: github.com/timzzx/tnet
```

解决问题：
+ [gitCommit](https://github.com/timzzx/tnet/commit/e4e822634e0fd05c5b514db5674d7baf4ada0031) 把项目中github/timzzx/tnet替换成github.com/timzzx/tnet

+ 发布一个tag

但是按照上面解决完还是会报错，最后发现goproxy 有缓存要等一等才能正常

## chat项目目录
项目源码 [地址](https://github.com/timzzx/tnet-chat)
```
.
├── Client
│   └── main.go // 命令行客户端
├── Handlers
│   └── TestHandler.go // 测试handler
├── LICENSE
├── go.mod
├── go.sum
└── main.go // 启动server
```

### 项目启动

main.go

```go
package main

import (
	handlers "chat/Handlers"

	"github.com/timzzx/tnet"
)

func main() {
	s := tnet.NewServer()
	// 添加一个handler
	s.AddHandlers(1, handlers.NewTestHandler(1))
	// 服务启动
	s.Start()
}

```

handler
```go
package handlers

import (
	"github.com/timzzx/tnet"
	"github.com/timzzx/tnet/types"
)

type TestHandler struct {
	id int
}

func NewTestHandler(id int) types.Handler {
	return &TestHandler{id: id}
}

func (h *TestHandler) Do(data []byte, agent types.Connection) {

	// fmt.Println("handlerID:", h.id, "消息:", )
	// 封包并发送
	msg, _ := tnet.Pack(h.id, data)
	agent.Send(msg)
	// agent.Cancel()
}

```

启动server
```
root@tdev:/home/code/tnet-chat# go run main.go 
TCP服务启动成功...
连接建立成功

```

客户端
```go
package main

import (
	"fmt"
	"net"
	"time"

	"github.com/timzzx/tnet"
)

func main() {
	conn, err := net.Dial("tcp", "192.168.1.13:9999")
	if err != nil {
		fmt.Println("连接失败", err)
	}
	defer conn.Close()
	for {
		// 发送消息
		msg, err := tnet.Pack(1, []byte("test"))
		conn.Write(msg)
		if err != nil {
			fmt.Println("消息发送失败", err)
			return
		}

		// 接收消息
		_, data, err := tnet.Unpack(conn)
		if err != nil {
			fmt.Println("消息收回", err)
			conn.Close()
			return
		}

		fmt.Println(string(data))
		time.Sleep(time.Second)
	}
}

```

