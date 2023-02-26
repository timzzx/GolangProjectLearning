# golang tcp服务-1(开始)

## 源码

[地址](https://github.com/timzzx/tnet/tree/tnet-v0.1)

## tcp基本实现

```go
ln, err := net.Listen("tcp", ":8080")
if err != nil {
    // handle error
}
for {
    conn, err := ln.Accept()
    if err != nil {
        // handle error
    }
    go handleConnection(conn)
}
```

## 目标

+ 1. 消息自定义，消息体组成 routerId 4字节 | datalen 4字节| data 
+ 2. 自定义不同消息路由到自定义的Handler进行逻辑处理

## 解包封包

```go
package tnet

import (
	"bytes"
	"encoding/binary"
	"io"
	"net"
)

// 消息解包
func Unpack(conn net.Conn) (int, string, error) {
	// 消息组成 routerId 4字节 | datalen 4字节| data
	// 获取路由id
	routerID := make([]byte, 4)
	_, err := io.ReadFull(conn, routerID)
	if err != nil {
		return 0, "", err
	}
	rid := int(binary.LittleEndian.Uint32(routerID))

	// 获取消息长度
	dataLen := make([]byte, 4)
	_, err = io.ReadFull(conn, dataLen)
	if err != nil {
		return 0, "", err
	}
	msgLen := int(binary.LittleEndian.Uint32(dataLen))

	// 获取消息
	data := make([]byte, msgLen)
	_, err = io.ReadFull(conn, data)
	if err != nil {
		return 0, "", err
	}
	msg := bytes.NewBuffer([]byte{})
	binary.Write(msg, binary.LittleEndian, data)

	return rid, string(data), nil
}

// 消息封包并发送
func PackSend(rid int, data string, conn net.Conn) ([]byte, error) {
	databuff := bytes.NewBuffer([]byte{})

	// 写msgID
	if err := binary.Write(databuff, binary.LittleEndian, uint32(rid)); err != nil {
		return nil, err
	}

	//写dataLen
	if err := binary.Write(databuff, binary.LittleEndian, uint32(len(data))); err != nil {
		return nil, err
	}

	//写data数据
	if err := binary.Write(databuff, binary.LittleEndian, []byte(data)); err != nil {
		return nil, err
	}

	return databuff.Bytes(), nil
}


```

## server主体逻辑

```go
package tnet

import (
	"fmt"
	"github/timzzx/tnet/types"
	"net"
	"sync"
)

type Server struct {
	Name        string
	Port        string
	Connections map[string]net.Conn
    // 自定义handler集合，可以根据路由id获取到自定义handler
	Handlers    map[int]types.Handler

	mu sync.Mutex
}

func NewServer() *Server {
	return &Server{
		Name:        "tcp",
		Port:        "9999",
		Connections: make(map[string]net.Conn),
		Handlers:    make(map[int]types.Handler),
	}
}

func (s *Server) Start() {
	listen, err := net.Listen("tcp", "0.0.0.0:"+s.Port)
	if err != nil {
		fmt.Println("监听失败：", err)
		return
	}

	fmt.Println("TCP服务启动成功...")

	for {
		conn, err := listen.Accept()
		if err != nil {
			fmt.Println("建立连接失败", err)
			continue
		}
		fmt.Println("连接建立成功")

		// 获取用户id
		uid := conn.RemoteAddr().String()
		// 连接加入全局
		s.addConnections(uid, conn)

		// 逻辑控制
		go s.proceess(conn)

	}
}

// 逻辑控制
func (s *Server) proceess(conn net.Conn) {
	defer conn.Close()
	for {
		// 获取消息
		routerID, data, err := Unpack(conn)
		if err != nil {
			fmt.Println("消息解析失败：", err)
			return
		}

		// 根据路由id调用处理逻辑
		s.doHandler(routerID, data, conn)
	}
}

// 根据路由调用处理逻辑
func (s *Server) doHandler(id int, data string, conn net.Conn) {
	s.Handlers[id].Do(data, conn)
}

// 连接加入全局
func (s *Server) addConnections(uid string, conn net.Conn) {
	s.mu.Lock()
	defer s.mu.Unlock()

	s.Connections[uid] = conn

}

// 添加handler
func (s *Server) AddHandlers(id int, handler types.Handler) {
	s.mu.Lock()
	defer s.mu.Unlock()

	s.Handlers[id] = handler
}

func (s *Server) Stop() {

}

```


这里定义了Handler接口
```go
package types

import "net"

type Handler interface {
	Do(string, net.Conn)
}

```

增加一个自定义的handler

```go
package handlers

import (
	"github/timzzx/tnet"
	"github/timzzx/tnet/types"
	"net"
)

type TestHandler struct {
	id int
}

func NewTestHandler(id int) types.Handler {
	return &TestHandler{id: id}
}

func (h *TestHandler) Do(data string, conn net.Conn) {

	// fmt.Println("handlerID:", h.id, "消息:", )
	// 封包并发送
	msg, _ := tnet.PackSend(h.id, "handler发送:"+data, conn)
	conn.Write(msg)
}

```

## 测试

server启动
```go
package main

import (
	"github/timzzx/tnet"

	"github/timzzx/tnet/handlers"
)

func main() {
    // 初始化server
	s := tnet.NewServer()
    //上面自定义的handler
	h := handlers.NewTestHandler(1)
    // 注册到服务中
	s.AddHandlers(1, h)
    // 服务启动
	s.Start()
}


```
client启动

```go
package main

import (
	"fmt"
	"github/timzzx/tnet"
	"net"
	"time"
)

func main() {
	conn, err := net.Dial("tcp", "192.168.1.13:9999")
	if err != nil {
		fmt.Println("连接失败", err)
	}
	defer conn.Close()
	for {
		// 发送消息 这里的1就是路由id.
        msg, err := tnet.PackSend(1, "test", conn)
		conn.Write(msg)
		if err != nil {
			fmt.Println("消息发送失败", err)
			return
		}

		// 接收消息
		_, data, err := tnet.Unpack(conn)
		if err != nil {
			fmt.Println("消息收回", err)
			return
		}

		fmt.Println(data)
		time.Sleep(time.Second)
	}
}

```

## 总结

这个版本写的很粗糙，目的是实现根据路由id执行自定义的handler。

```
.
├── Client
│   └── main.go // 客户端实现
├── LICENSE
├── MsgPack.go
├── README.md
├── Server
│   └── server.go // 服务启动
├── Server.go // 服务实现
├── go.mod
├── go.sum
├── handlers // 自定义handler
│   ├── Test2Handler.go
│   └── TestHandler.go
└── types // handler接口
    └── Handler.go
```
