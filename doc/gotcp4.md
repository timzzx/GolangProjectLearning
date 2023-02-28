# golang tcp服务-4(项目设计参考go-zero，tcp基于tnet)

## 前言

> chat项目，基于go-zero的api项目目录布局，使用了go-zero的config，来使用配置文件，具体可以看项目源码基本和go-zero类似。

## 项目源码和目录
[源码地址](https://github.com/timzzx/tnet-chat)

```
root@tdev:/home/code/tnet-chat# tree
.
├── Client
│   └── main.go // 客户端测试用的，后续准备写个singleTest
├── LICENSE
├── chat.go // main 启动server
├── chatmodel // gorm-gen 文件不需要修改，都是gentool工具生成的
│   └── dao
│       ├── model
│       │   └── user.gen.go
│       └── query
│           ├── gen.go
│           └── user.gen.go
├── etc  // 配置文件
│   └── chat.yaml
├── go.mod
├── go.sum
├── internal 
│   ├── config // 配置结构
│   │   └── config.go
│   ├── handler // 这部分固定的
│   │   └── loginhandler.go
│   ├── logic // 逻辑写在这里
│   │   └── loginlogic.go
│   └── svc // 全局
│       └── servicecontext.go
├── jsontype
│   └── Login.go // 登录的请求返回的结构体
├── makefile // make命令
├── singleTest // 后续要做的
└── varx
    └── routervar.go // 路由id设置
```

## tnet Context.Done 监听问题

解决方式具体查看[地址](https://github.com/timzzx/tnet/commit/0eb1bdb170d46052df3512add95f2f373fbee6dd)

## chat登录
> 从零完成一个登录请求

启动文件 chat.go

```go
package main

import (
	"chat/internal/config"
	"chat/internal/handler"
	"chat/internal/svc"
	"chat/varx"
	"flag"
	"os"
	"os/signal"
	"syscall"

	"github.com/timzzx/tnet"
	"github.com/zeromicro/go-zero/core/conf"
)

var configFile = flag.String("f", "etc/chat.yaml", "the config file")

func main() {
	flag.Parse()
    // 利用go-zero conf包来读取配置
	var c config.Config
	conf.MustLoad(*configFile, &c)
	ctx := svc.NewServiceContext(c)
	s := tnet.NewServer()
	cs := make(chan os.Signal)
	signal.Notify(cs, syscall.SIGTERM, syscall.SIGINT)
	go func() {
		select {
		case <-cs:
			{
				s.Stop() // 服务器退出
				os.Exit(1)
			}
		}
	}()

	// 增加路由 （这里需要增加一个登录loginHandler）
	s.AddHandlers(varx.LOGIN, handler.LoginHandler(varx.LOGIN, ctx))
	// 服务启动
	s.Start()
}

```

增加 internal/handler/loginhandler.go 这个文件后期可以做自动生成 固定设置
```go
package handler

import (
	"chat/internal/logic"
	"chat/internal/svc"

	"github.com/timzzx/tnet/types"
)

func LoginHandler(id int, svc *svc.ServiceContext) types.Handler {
	return logic.NewLoginLogic(id, svc)
}

```

现在先增加mysql配置操作

修改etc/chat.yaml
```yaml
Mysql:
  DataSource: root:123456@tcp(192.168.1.13:3306)/chat?charset=utf8mb4&parseTime=true&loc=Asia%2FShanghai

Redis:
  Host: 192.168.1.13:6379
  Type: node
```
修改internal/config/config.go
```go
package config

type Config struct {
	Mysql struct {
		DataSource string
	}

	Redis struct {
		Host string
		Type string
	}
}
```
修改internal/svc/servicecontext.go
```go
package svc

import (
	"chat/chatmodel/dao/query"
	"chat/internal/config"

	"github.com/redis/go-redis/v9"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

type ServiceContext struct {
	Config config.Config

	Redis     *redis.Client
	ChatModel *query.Query
}

func NewServiceContext(c config.Config) *ServiceContext {
	db, _ := gorm.Open(mysql.Open(c.Mysql.DataSource), &gorm.Config{})
	rdb := redis.NewClient(&redis.Options{
		Addr:     c.Redis.Host,
		Password: "", // no password set
		DB:       0,  // use default DB
	})
	return &ServiceContext{
		Config:    c,
		Redis:     rdb,
		ChatModel: query.Use(db),
	}
}

```
> 上面的设置和go-zero api的方式基本类似

下面完成功能修改internal/logic/loginlogic.go
```go
package logic

import (
	"chat/internal/svc"
	"chat/jsontype"
	"encoding/json"
	"fmt"

	"github.com/timzzx/tnet"
	"github.com/timzzx/tnet/types"
)

type LoginLogic struct {
	id     int
	svcCtx *svc.ServiceContext
}

func NewLoginLogic(id int, svcCtx *svc.ServiceContext) types.Handler {
	return &LoginLogic{
		id:     id,
		svcCtx: svcCtx,
	}
}
// 登录逻辑
func (l *LoginLogic) Do(req []byte, agent types.Connection) {
	// 获取参数
	var Req jsontype.LoginReq
	if err := json.Unmarshal(req, &Req); err != nil {
		resp, _ := json.Marshal(jsontype.LoginRsp{
			Code: 500,
			Msg:  err.Error(),
		})
		msg, _ := tnet.Pack(l.id, resp)
		agent.Send(msg)
		return
	}
	// 数据库获取
	u := l.svcCtx.ChatModel.User
	user, err := u.WithContext(agent.Ctx()).Where(u.Name.Eq(Req.Name)).Debug().First()
	if err != nil {
		resp, _ := json.Marshal(jsontype.LoginRsp{
			Code: 500,
			Msg:  err.Error(),
		})
		msg, _ := tnet.Pack(l.id, resp)
		agent.Send(msg)
		return
	}

	// 密码错误
	if user.Password != Req.Password {
		fmt.Println(user.Password)
		resp, _ := json.Marshal(jsontype.LoginRsp{
			Code: 500,
			Msg:  "密码错误",
		})
		msg, _ := tnet.Pack(l.id, resp)
		agent.Send(msg)
		return
	}
	resp, _ := json.Marshal(jsontype.LoginRsp{
		Code: 200,
		Msg:  "登录成功",
	})
	msg, _ := tnet.Pack(l.id, resp)
	agent.Send(msg)
	return
}

```
功能完成了，后续只要增加handler和logic和chat中addHandler

## 客户端

创建一个简单的makefile
```makefile
# 命令
help:
	@echo 'Usage:'
	@echo '     db 生成sql执行代码'
	@echo '     dev 运行'
db:
	gentool -dsn 'root:123456@tcp(192.168.1.13:3306)/chat?charset=utf8mb4&parseTime=True&loc=Local' -outPath './chatmodel/dao/query'
dev:
	go run chat.go -f etc/chat.yaml
```

编辑Client/main.go
```go
package main

import (
	"chat/jsontype"
	"encoding/json"
	"fmt"
	"net"

	"github.com/timzzx/tnet"
)

func main() {
	conn, err := net.Dial("tcp", "192.168.1.13:9999")
	if err != nil {
		fmt.Println("连接失败", err)
	}
	defer conn.Close()

	// 发送错误结构的消息
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
	// 发送密码错误的消息
	req, _ := json.Marshal(jsontype.LoginReq{
		Name:     "timzzx",
		Password: "123",
	})
	msg, err = tnet.Pack(1, req)
	conn.Write(msg)
	if err != nil {
		fmt.Println("消息发送失败", err)
		return
	}
	// 接收消息
	_, data, err = tnet.Unpack(conn)
	if err != nil {
		fmt.Println("消息收回", err)
		conn.Close()
		return
	}

	fmt.Println(string(data))
	// 发送正确的消息
	req, _ = json.Marshal(jsontype.LoginReq{
		Name:     "timzzx",
		Password: "123456",
	})
	msg, err = tnet.Pack(1, req)
	conn.Write(msg)
	if err != nil {
		fmt.Println("消息发送失败", err)
		return
	}

	// 接收消息
	_, data, err = tnet.Unpack(conn)
	if err != nil {
		fmt.Println("消息收回", err)
		conn.Close()
		return
	}

	fmt.Println(string(data))

}

```

## 测试

启动sever
```
root@tdev:/home/code/tnet-chat# make dev
go run chat.go -f etc/chat.yaml
TCP服务启动成功...
```

启动客户单
```
root@tdev:/home/code/tnet-chat/Client# go run main.go 
{"code":500,"msg":"invalid character 'e' in literal true (expecting 'r')"}
{"code":500,"msg":"密码错误"}
{"code":200,"msg":"登录成功"}
```

服务端的打印
```
root@tdev:/home/code/tnet-chat# make dev
go run chat.go -f etc/chat.yaml
TCP服务启动成功...
连接建立成功

2023/02/28 16:51:52 /home/code/tnet-chat/chatmodel/dao/query/user.gen.go:234
[0.948ms] [rows:1] SELECT * FROM `user` WHERE `user`.`name` = 'timzzx' ORDER BY `user`.`id` LIMIT 1
123456

2023/02/28 16:51:52 /home/code/tnet-chat/chatmodel/dao/query/user.gen.go:234
[0.718ms] [rows:1] SELECT * FROM `user` WHERE `user`.`name` = 'timzzx' ORDER BY `user`.`id` LIMIT 1
消息解析失败： EOF
```

