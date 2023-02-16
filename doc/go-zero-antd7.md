# go-zero-antd实战-3（go-zero添加cobra命令行工具）

> 对于一个后台来说，肯定在一些情况下需要使用命令行执行一些命令，所以添加了cobra包来实现需要的功能

## 安装

```
// 安装cobra命令
go install github.com/spf13/cobra-cli@latest

// 在项目下创建command文件夹用来初始化cobra
mkdir command
cd command
// 初始cobra项目
cobra-cli init
```

## 添加一个命令看看
```
cobra-cli init userlist

// 回显
root@tdev:/home/code/go-zero-antd-backend/api/command# cobra-cli add userlist
userlist created at /home/code/go-zero-antd-backend/api/command


// 运行命令看看效果
go run main.go userlist

// 返回
userlist called

```

项目目录
```
.
├── LICENSE
├── cmd
│   ├── root.go
│   └── userlist.go
└── main.go
```

## 添加go-zero serverctx

> 添加这个目的就是可以使用go-zero的一些配置，保证一致性

修改 command/cmd/root.go
```go
// 定义全局变量
var svcCtx *svc.ServiceContext
...
unc init() {
    ...
	var c config.Config
	conf.MustLoad("../etc/backend.yaml", &c)
	svcCtx = svc.NewServiceContext(c)
}
```

## 测试是否可以使用
修改 command/cmd/userlist.go
```go
Run: func(cmd *cobra.Command, args []string) {
		u := svcCtx.BkModel.User
		d, err := u.WithContext(context.Background()).Debug().First()
		if err != nil {
			fmt.Println(err)
		}
		fmt.Println(d)
	},
```
执行命令

```
go run main.go userlist

// 回显
[0.463ms] [rows:1] SELECT * FROM `user` ORDER BY `user`.`id` LIMIT 1
&{1 tim 0 123456 1 0 0}
```

## 源码已上传

[地址](https://github.com/timzzx/go-zero-antd-backend)

# go-zero-antd实战-4（go-zero添加cron定时任务）

> 后台系统一般都会有定时任务的需求，所以加入[cron](https://github.com/robfig/cron)

创建了一个目录cron如下
```
├── cmd
│   ├── root.go
│   ├── schedule.go
│   └── userlist.go
├── cronx
│   └── time.go
└── main.go 

2 directories, 5 files
```
root.go中启动了cron，加入了go-zero srvCtx
```go
package cmd

import (
	"fmt"
	"tapi/internal/config"
	"tapi/internal/svc"

	"github.com/robfig/cron/v3"
	"github.com/zeromicro/go-zero/core/conf"
)

var svcCtx *svc.ServiceContext

func Execute() {
	c := cron.New(cron.WithSeconds())

	// 这里是加入定时任务
	ScheduleRun(c)

	fmt.Println("定时任务启动...")
	go c.Start()
	defer c.Stop()
	select {}
}

func init() {
	var c config.Config
	conf.MustLoad("../etc/backend.yaml", &c)
	svcCtx = svc.NewServiceContext(c)
}


```

schedule.go只要在其中加入任务即可
```go
func ScheduleRun(c *cron.Cron) {
	c.AddFunc(cronx.EveryMinute(), func() {
		fmt.Println("定时任务")
	})
	// 每分钟定时查询用户信息
	c.AddFunc(cronx.Every5s(), userlist)

	// 计划任务执行写在这里
}

```

## 测试cron
写了一个userlist.go
```go
package cmd

import (
	"context"
	"fmt"
)

func userlist() {
	u := svcCtx.BkModel.User
	d, err := u.WithContext(context.Background()).Debug().First()
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(d)
}

```


```
root@tdev:/home/code/go-zero-antd-backend/api/cron# go run main.go 
定时任务启动...

2023/02/16 23:13:40 /home/code/go-zero-antd-backend/api/bkmodel/dao/query/user.gen.go:238
[7.589ms] [rows:1] SELECT * FROM `user` ORDER BY `user`.`id` LIMIT 1
&{1 tim 0 123456 1 0 0}

2023/02/16 23:13:45 /home/code/go-zero-antd-backend/api/bkmodel/dao/query/user.gen.go:238
[0.617ms] [rows:1] SELECT * FROM `user` ORDER BY `user`.`id` LIMIT 1
&{1 tim 0 123456 1 0 0}
```

> cron定时任务加入go-zero项目完成


## 源码已上传

[地址](https://github.com/timzzx/go-zero-antd-backend)





# go-zero-antd实战-5（go-zero添加asynq队列任务）

> 后台系统一般都会有任务队列的需求，本项目加入了[asynq](https://github.com/hibiken/asynq)，具体使用参考官方文档。使用不复杂，这里只是简单的加入到go-zero单体服务的项目中，如果使用微服务可以参考[go-zero-looklook项目](https://github.com/Mikaelemmmm/go-zero-looklook)它相当于单独做了个asynq的服务，设计思路类似go-zero的api，我仔细阅读后完成了这次接入。

## asynq加入项目
创建了aqueue目录，
```
aqueue/
├── handler // task任务
│   └── userlist.go
├── jobtype // task对应的结构体和const
│   └── userlist.go
├── main.go
└── queue
    └── queue.go // asynq 服务器部分的启动文件

3 directories, 4 files
```

首先先看main.go
```go
func main() {
	// 这里还是去引用go-zero的配置
	var c config.Config
	conf.MustLoad("../etc/backend.yaml", &c)
	svcCtx := svc.NewServiceContext(c)
	ctx := context.Background()
	// 这里可以看源码，类似go-zero的rest，也可以看做http 
	job := queue.NewQueue(ctx, svcCtx)
	// 注册路由
	mux := job.Register()
	// 启动asynq服务连接redis
	server := asynq.NewServer(
		asynq.RedisClientOpt{Addr: svcCtx.Config.Redis.Host},
		asynq.Config{
			IsFailure: func(err error) bool {
				fmt.Printf("asynq server exec task IsFailure ======== >>>>>>>>>>>  err : %+v \n", err)
				return true
			},
			Concurrency: 20, //max concurrent process job task num
		},
	)

	if err := server.Run(mux); err != nil {
		logx.WithContext(ctx).Errorf("!!!CronJobErr!!! run err:%+v", err)
		os.Exit(1)
	}
}

```

创建 aqueue/queue/queue.go

```go
package queue

import (
	"context"
	"tapi/aqueue/handler"
	"tapi/aqueue/jobtype"
	"tapi/internal/svc"

	"github.com/hibiken/asynq"
)

type Queue struct {
	ctx    context.Context
	svcCtx *svc.ServiceContext
}

func NewQueue(ctx context.Context, svcCtx *svc.ServiceContext) *Queue {
	return &Queue{
		ctx:    ctx,
		svcCtx: svcCtx,
	}
}

// register job 这里一看就和go-zero的router类似
func (l *Queue) Register() *asynq.ServeMux {

	mux := asynq.NewServeMux()

	// job handler就加入到这里
	// NewUserListHandler这个方法要符合asynq.Handler接口
	/*
	type Handler interface {
		ProcessTask(context.Context, *Task) error
	}
	*/
	mux.Handle(jobtype.DesUserList, handler.NewUserListHandler(l.svcCtx))

	return mux
}

```

> 上面其实算是加入成功了

## server部分创建

创建handler  增加aqueue/handler/userlist.go
```go
package handler

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"tapi/aqueue/jobtype"
	"tapi/internal/svc"

	"github.com/hibiken/asynq"
)

type UserListHandler struct {
	svcCtx *svc.ServiceContext
}

func NewUserListHandler(svcCtx *svc.ServiceContext) *UserListHandler {
	return &UserListHandler{
		svcCtx: svcCtx,
	}
}

func (l *UserListHandler) ProcessTask(ctx context.Context, t *asynq.Task) error {
	// 获取参数
	var p jobtype.PayloadUserList
	if err := json.Unmarshal(t.Payload(), &p); err != nil {
		return errors.New("参数错误")
	}

	u := l.svcCtx.BkModel.User
	d, err := u.WithContext(context.Background()).Where(u.ID.Eq(p.Id)).Debug().First()
	if err != nil {
		return err
	}
	fmt.Println(d)
	return nil
}

```
创建结构aqueue/jobtype/userlist.go
```go
package jobtype
// 任务标识
const DesUserList = "job:user_list"
// 任务接收发送参数
type PayloadUserList struct {
	Id int64
}

```


添加路由 aqueue/queue/queue.go
```go
// job
mux.Handle(jobtype.DesUserList, handler.NewUserListHandler(l.svcCtx))

```
> 服务部分完成了

## 客户端部分
新增internal/svc/asynqClient.go
```go
package svc

import (
	"tapi/internal/config"

	"github.com/hibiken/asynq"
)

// create asynq client.
func newAsynqClient(c config.Config) *asynq.Client {
	return asynq.NewClient(asynq.RedisClientOpt{Addr: c.Redis.Host})
}

```


修改 internal/svc/servicecontext.go
```go
AsynqClient: newAsynqClient(c),
```

### 调用客户端
在userinfo接口添加
```go
	// 测试一下写入job
	// 设置参数
	payload, err := json.Marshal(jobtype.PayloadUserList{Id: 2})
	if err != nil {
		return &types.UserInfoResponse{
			Code: 500,
			Msg:  err.Error(),
		}, nil
	} else {
		// 加入任务
		_, err = l.svcCtx.AsynqClient.Enqueue(asynq.NewTask(jobtype.DesUserList, payload))
		if err != nil {
			return &types.UserInfoResponse{
				Code: 500,
				Msg:  err.Error(),
			}, nil
		}
	}
```

## 测试

启动asynq服务端
```
cd aqueue
go run main.go

// 回显
root@tdev:/home/code/go-zero-antd-backend/api/aqueue# go run main.go 
asynq: pid=8583 2023/02/16 17:14:17.527560 INFO: Starting processing
asynq: pid=8583 2023/02/16 17:14:17.528667 INFO: Send signal TSTP to stop processing new tasks
```

启动go-zero
```go
make dev
```

访问192.168.1.13:8888/api/user/info

asynq服务端显示
```
{"@timestamp":"2023-02-17T01:15:17.522+08:00","caller":"stat/usage.go:61","content":"CPU: 5m, MEMORY: Alloc=1.6Mi, TotalAlloc=5.8Mi, Sys=18.6Mi, NumGC=2","level":"stat"}

2023/02/17 01:15:18 /home/code/go-zero-antd-backend/api/bkmodel/dao/query/user.gen.go:238
[1.098ms] [rows:1] SELECT * FROM `user` WHERE `user`.`id` = 2 ORDER BY `user`.`id` LIMIT 1
&{2 ttt 0 123456 1 1676195522 1676195522}
```

## 源码已上传

[地址](https://github.com/timzzx/go-zero-antd-backend)