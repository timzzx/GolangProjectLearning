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
