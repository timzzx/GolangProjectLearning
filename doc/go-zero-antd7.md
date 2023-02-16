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

> 后台系统一般都会有任务队列的需求，所以加入[asynq](https://github.com/hibiken/asynq)