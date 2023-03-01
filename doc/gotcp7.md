# golang tcp服务-7 (修复tnet阻塞问题chat创建房间加入房间功能)

> 增加了服务器推送消息监听和response消息区分，本版本只是简单实现不是很优雅，只是为了实现功能



## 修复tnet连接监听ctx.Done第二个阻塞了
> 另起一个goroutine
```
go func() {
        select {
        case <-ctx.Done():
            s.claer(agent)
        }
    }()
```

## 增加服务器推送
> 服务端不用修改，只是约定rid > 1000的消息都是推送的仅此而已

修改singleTest/agent/agent.go
```go
type agent struct {
	conn       net.Conn
	resp       chan []byte
	push       chan []byte//增加push channel
	Directives map[string]types.Directives
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
		// 屏蔽掉心跳(主要这里修改了)
		if rid != 0 {
			// 路由大于1000的都是服务器推送过来的
			if rid > 1000 {
				a.push <- data
			} else {
				a.resp <- data
			}

		}
	}
}

func (a *agent) PushShow() {
	for {
		select {
		case msg := <-a.push:
			fmt.Println("==PushShow:" + string(msg))
		}
	}

}
```
启动 PushShow 修改singleTest/cmd/agent.go
```go
// 监听消息接收
go client.Reader()
go client.PushShow() // add
```

> 这个修改其实可以考虑给消息加个类型

> 之前 消息组成 routerId 4字节 | datalen 4字节| data 

> 改成消息组成 msgType 4字节|routerId 4字节 | datalen 4字节| data

> 暂时先完成功能就不优化了

## 房间创建加入

先定义房间结构，创建internal/object/member.go 用户类型
```go
package object

import "github.com/timzzx/tnet/types"

type Member struct {
	UserId int
	Name   string
	Agent  types.Connection
}

func NewMember(uid int, name string, agent types.Connection) *Member {
	return &Member{
		UserId: uid,
		Name:   name,
		Agent:  agent,
	}
}

```

创建 internal/object/room.go

> 整个server只会创建一个房间
```go
package object

import "sync"

type Rooms struct {
	Members map[int]*Member
	mu      sync.Mutex
}

var Room *Rooms

func init() {
	var once sync.Once
	once.Do(func() {
		Room = &Rooms{
			Members: make(map[int]*Member),
		}
	})

}

func (r *Rooms) Add(m *Member) {
	r.mu.Lock()
	defer r.mu.Unlock()

	r.Members[m.UserId] = m
}

func (r *Rooms) Del(uid int) {
	r.mu.Lock()
	defer r.mu.Unlock()
	delete(r.Members, uid)
}

func (r *Rooms) List() map[int]*Member {
	return r.Members
}

```
> 下面设计参考go-zero
### 增加handler
internal/handler/roomaddhandler.go

```go
package handler

import (
	"chat/internal/logic"
	"chat/internal/svc"

	"github.com/timzzx/tnet/types"
)

func RoomAddHandler(id int, svc *svc.ServiceContext) types.Handler {
	return logic.NewRoomAddLogic(id, svc)
}

```


### 增加logic并完成功能
```go
package logic

import (
	"chat/internal/object"
	"chat/internal/svc"
	"chat/jsontype"
	"encoding/json"

	"github.com/timzzx/tnet"
	"github.com/timzzx/tnet/types"
)

type RoomAddLogic struct {
	id     int
	svcCtx *svc.ServiceContext
}

func NewRoomAddLogic(id int, svcCtx *svc.ServiceContext) types.Handler {
	return &RoomAddLogic{
		id:     id,
		svcCtx: svcCtx,
	}
}

func (l *RoomAddLogic) Do(req []byte, agent types.Connection) {
	// 请求参数
	var Req jsontype.RoomAddReq
	if err := json.Unmarshal(req, &Req); err != nil {
		resp, _ := json.Marshal(jsontype.RoomAddResp{
			Code: 500,
			Msg:  err.Error(),
		})
		msg, _ := tnet.Pack(l.id, resp)
		agent.Send(msg)
		return
	}
	// 从数据库获取用户信息
	u := l.svcCtx.ChatModel.User
	user, err := u.WithContext(agent.Ctx()).Where(u.ID.Eq(int64(Req.UserId))).First()
	if err != nil {
		resp, _ := json.Marshal(jsontype.RoomAddResp{
			Code: 500,
			Msg:  err.Error(),
		})
		msg, _ := tnet.Pack(l.id, resp)
		agent.Send(msg)
		return
	}

	m := object.NewMember(int(user.ID), user.Name, agent)
	object.Room.Add(m)

	// 发消息给自己
	resp, _ := json.Marshal(jsontype.RoomAddResp{
		Code: 200,
		Msg:  "加入成功",
	})
	msg, _ := tnet.Pack(l.id, resp)
	agent.Send(msg)

	// 发消息给所有人(这里就利用了前面加的服务器推送)
	resp, _ = json.Marshal(jsontype.RoomAddResp{
		Code: 200,
		Msg:  user.Name + "加入成功",
	})
    // 路由id > 1000
	msg, _ = tnet.Pack(1002, resp)
	for _, member := range object.Room.List() {
		member.Agent.Send(msg)
	}

	return
}

```
修改 chat.go
```go
// 添加路由
s.AddHandlers(varx.ROOMADD, handler.RoomAddHandler(varx.ROOMADD, ctx))
```

## singleTest功能

singleTest/agent/agent.go 
> 增加用户id这样登录后就可以设置agent的用户id，这个以后其他指令就可以不用传递了
```go
type agent struct {
	userId     int
	conn       net.Conn
	resp       chan []byte
	push       chan []byte
	Directives map[string]types.Directives
}
...
// 获取userid
func (a *agent) GetUserId() int {
	return a.userId
}

// 设置userId
func (a *agent) SetUserId(userId int) {
	a.userId = userId
}
```

增加指令singleTest/directives/roomadd.go
```go
package directives

import (
	"chat/jsontype"
	"chat/singleTest/types"
	"encoding/json"
	"fmt"
)

type RoomAdd struct {
}

func (h *RoomAdd) Do(agent types.Agent) error {
	if agent.GetUserId() == 0 {
		fmt.Println("请先登录")
		return nil
	}
	// 加入房间
	req, _ := json.Marshal(jsontype.RoomAddReq{
		UserId: agent.GetUserId(),
	})
	if err := agent.Send(2, req); err != nil {
		return err
	}
	msg := <-agent.GetResp()
	fmt.Println("==Resp:" + string(msg))

	return nil
}

```

## 测试

服务启动
```
root@tdev:/home/code/tnet-chat# make dev
go run chat.go -f etc/chat.yaml
TCP服务启动成功...
```

客户端1
```
root@tdev:/home/code/tnet-chat/singleTest# go run main.go agent
> login
==Resp:{"user_id":1,"code":200,"msg":"登录成功"}
UserId:1
> roomadd
==PushShow:{"code":200,"msg":"timzzx加入成功"}
==Resp:{"code":200,"msg":"加入成功"}
> ==PushShow:{"code":200,"msg":"ttt加入成功"}

```

客户端2
```
root@tdev:/home/code/tnet-chat/singleTest# go run main.go agent
> login
==Resp:{"user_id":1,"code":200,"msg":"登录成功"}
UserId:1
> roomadd
==PushShow:{"code":200,"msg":"timzzx加入成功"}
==Resp:{"code":200,"msg":"加入成功"}
> ==PushShow:{"code":200,"msg":"ttt加入成功"}

```

## 源码
[tnet地址](https://github.com/timzzx/tnet)

[tnet-chat地址](https://github.com/timzzx/tnet-chat)

## 总结

> 这次从服务端到客户端singleTest的实现，文档很难说明白，具体还是需要查看源码