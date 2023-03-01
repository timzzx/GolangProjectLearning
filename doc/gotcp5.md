# golang tcp服务-5(tcp长连接基于cobra完成singleTest tcp命令行客户端)

## 目的

> 在写tcp长连接的时候，测试需要一个命令行测试工具，所以基于cobra来实现

## cobra初始化项目

```
cd singleTest
// 初始化项目
cobra-cli init
// 生成子命令
cobra-cli init agent
```
项目目录
```
├── LICENSE
├── cmd
│   ├── agent.go // 基于这个命令文件来实现需求
│   └── root.go
├── main.go
```

命令执行
```
go run main.go agent

```

## 实现tcp client
先写接口types/types.go
```go
package types
// 指令接口
type Directives interface {
	Do(agent Agent)
}
// tcp client 发送接口
type Agent interface {
	Send(rid int, data []byte) error
}

```
增加指令directives/login.go
```go
package directives

import (
	"chat/jsontype"
	"chat/singleTest/types"
	"encoding/json"
	"fmt"
)

type Login struct {
}
// 请求逻辑
func (h *Login) Do(agent types.Agent) {
	// 登录
	req, _ := json.Marshal(jsontype.LoginReq{
		Name:     "timzzx",
		Password: "123456",
	})
	if err := agent.Send(1, req); err != nil {
		fmt.Println("消息发送失败", err)
	}
}

```
> 后续只需要增加类似login.go指令文件即可。

增加agent/agent.go
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
	Directives map[string]types.Directives
}

func NewAgent(conn net.Conn) *agent {
	return &agent{
		conn:       conn,
		Directives: make(map[string]types.Directives),
	}
}

// 发送消息
func (a *agent) Send(rid int, data []byte) error {
	msg, err := tnet.Pack(1, data)
	_, err = a.conn.Write(msg)
	return err
}

// 接收消息（统一消息接收处理）
func (a *agent) Reader() {
	for {
		// 接收消息
		rid, data, err := tnet.Unpack(a.conn)
		if err != nil {
			fmt.Println("消息收回", err)
			a.conn.Close()
			return
		}

		if rid != 0 {
			fmt.Println("Resp:" + string(data))
			fmt.Print("> ")
		}
	}
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

修改cobra agent文件就是修改Run方法 cmd/agent.go

```go
/*
Copyright © 2023 NAME HERE <EMAIL ADDRESS>
*/
package cmd

import (
	"bufio"
	"chat/singleTest/agent"
	"fmt"
	"net"
	"os"
	"strings"

	"github.com/spf13/cobra"
)

// agentCmd represents the agent command
var agentCmd = &cobra.Command{
	Use:   "agent",
	Short: "A brief description of your command",
	Long: `A longer description that spans multiple lines and likely contains examples
and usage of using your command. For example:

Cobra is a CLI library for Go that empowers applications.
This application is a tool to generate the needed files
to quickly create a Cobra application.`,
    // cmd接收指令处理 
	Run: func(cmd *cobra.Command, args []string) {
		// 连接服务器
		conn, err := net.Dial("tcp", "192.168.1.13:9999")
		if err != nil {
			fmt.Println("连接失败", err)
			return
		}
		defer conn.Close()

		// 创建agent
		client := agent.NewAgent(conn)
		client.InitDirectives()

		// 监听消息接收
		go client.Reader()

		buf := bufio.NewReader(os.Stdin)
		// 监听输入
		for {
			fmt.Print("> ")
			directives, err := buf.ReadBytes('\n')
			if err != nil {
				fmt.Println("命令操作", err)
				return
			}
			// 去除换行符
			str := strings.Replace(string(directives), "\n", "", -1)
			// 处理指令
			handeler, ok := client.Directives[str]
			if !ok {
				fmt.Println("指令不存在")
			} else {
				handeler.Do(client)
			}

		}
	},
}

func init() {
	rootCmd.AddCommand(agentCmd)

	// Here you will define your flags and configuration settings.

	// Cobra supports Persistent Flags which will work for this command
	// and all subcommands, e.g.:
	// agentCmd.PersistentFlags().String("foo", "", "A help for foo")

	// Cobra supports local flags which will only run when this command
	// is called directly, e.g.:
	// agentCmd.Flags().BoolP("toggle", "t", false, "Help message for toggle")
}

```

## 测试

先启动tcp server 这个看之前的文档[golang tcp服务](https://github.com/timzzx/GolangProjectLearning/blob/main/doc/gotcp1.md)

```
// 启动server
make dev 
```

启动singleTest

```
cd singleTest
// 启动
go run main.go agent
```

输入指令
```
root@tdev:/home/code/tnet-chat/singleTest# go run main.go agent
> login
> Resp:{"code":200,"msg":"登录成功"}
> 
```

## 源码

[地址](https://github.com/timzzx/tnet-chat)

