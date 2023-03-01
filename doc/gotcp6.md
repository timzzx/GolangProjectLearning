# golang tcp服务-6 (解决路由不存在报错的问题，修改消息接收方式)

## tnet路由不存在报错
```
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x18 pc=0x4cf02d]

goroutine 11 [running]:
github.com/timzzx/tnet.(*Server).doHandler(0x0?, 0x0?, {0xc000018320, 0x4, 0x4}, {0x522e50, 0xc00005e0c0})
        /home/code/tnet/Server.go:126 +0x4d
created by github.com/timzzx/tnet.(*Server).proceess
        /home/code/tnet/Server.go:84 +0x185
exit status 2
```

## 修复

修改tnet/Server.go
```go
// 根据路由调用处理逻辑
func (s *Server) doHandler(id int, data []byte, agent types.Connection) {
	handler, ok := s.Handlers[id]
	if !ok {
		// 发送消息
		msg, _ := Pack(10000, []byte("router id: "+strconv.Itoa(id)+" error"))
		agent.Send(msg)
		return
	}

	handler.Do(data, agent)
}
```

## tnet-chat 更新 tnet
tnet代码提交后需要生成releases

```
// 在tnet项目根目录运行
go get github.com/timzzx/tnet
// 用了goproxy有缓存要等一等(等了20分钟)

```

## 修改消息接收方式

把消息发送到channel，修改singleTest/agent/agent.go
```go
package agent

import (
	"chat/singleTest/directives"
	"chat/singleTest/types"
	"fmt"
	"net"

	"github.com/timzzx/tnet"
)

type agent struct {
	conn       net.Conn
	resp       chan []byte
	Directives map[string]types.Directives
}

func NewAgent(conn net.Conn) *agent {
	return &agent{
		conn:       conn,
		Directives: make(map[string]types.Directives),
		resp:       make(chan []byte),
	}
}

// 发送消息
func (a *agent) Send(rid int, data []byte) error {
	msg, err := tnet.Pack(rid, data)
	_, err = a.conn.Write(msg)
	return err
}

// 接收消息
func (a *agent) Reader() {
	defer a.Stop()
	for {
		// 接收消息
		rid, data, err := tnet.Unpack(a.conn)
		if err != nil {
			fmt.Println("消息收回", err)
			return
		}

		if rid != 0 {
			// fmt.Println("Resp:" + string(data))
			// fmt.Print("> ")
            // 把消息发送到channel
			a.resp <- data
		}
	}
}

// 获取消息
func (a *agent) GetResp() <-chan []byte {
	return a.resp
}

// 停止
func (a *agent) Stop() {
	a.conn.Close()
}

// 初始化指令
func (a *agent) InitDirectives() {
	a.Directives["login"] = &directives.Login{}
}

```

修改singleTest/directives/login.go

```go
func (h *Login) Do(agent types.Agent) error {
	// 登录
	req, _ := json.Marshal(jsontype.LoginReq{
		Name:     "timzzx",
		Password: "123456",
	})
	if err := agent.Send(122, req); err != nil {
		return err
	}
	msg := <-agent.GetResp()
	fmt.Println(">Resp:" + string(msg))
	return nil
}
```

## 源码

[地址](https://github.com/timzzx/tnet-chat)