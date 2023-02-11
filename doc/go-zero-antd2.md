# 后台开发实战-2

## 目录

[返回-全部目录](../README.md#目录)<br />
[待办列表](#待办列表)<br />
[目的](#目的)<br />
[数据库表设计](#数据库表设计)<br />
[使用gorm gen](#使用gorm-gen)<br />

## 待办列表

- [x] 目的  
- [ ] 设计数据库表
- [ ] go-zero api
- [ ] antd pro 配合 go-zero

## 目的

> 完成基于go-zero单体服务+antd pro后台，登录，用户管理，权限管理

## 数据库表设计

1. 基于[GoDockerDev](https://github.com/timzzx/GoDockerDev)启动mysql，redis

```
// 创建compose目录
mkdir dev_compose

// 下载GoDockerDev
git clone https://github.com/timzzx/GoDockerDev.git

cd GoDockerDev/

// 启动docker-compose
docker-compose up -d

// 显示
root@tdev:/home/code/dev_compose/GoDockerDev# docker-compose up -d
[+] Running 2/2
 ⠿ Container godockerdev-redis-1  Started                                                    1.1s
 ⠿ Container godockerdev-mysql-1  Started                                                    1.0s
```

创建bk数据库并且创建user表

```sql
CREATE TABLE `user` (
  `id` bigint(20) unsigned zerofill NOT NULL AUTO_INCREMENT COMMENT '用户表主键',
  `name` varchar(128) NOT NULL COMMENT '用户名',
  `password` varchar(64) NOT NULL COMMENT '密码',
  `status` int(11) NOT NULL DEFAULT '1' COMMENT '是否有效1.有效 2.无效',
  `ctime` int(11) NOT NULL DEFAULT '0' COMMENT '创建时间',
  `utime` int(11) DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表'
```

## 使用gorm gen

安装生成
```
// 安装Gen Tool
go install gorm.io/gen/tools/gentool@latest

// 报错了
cgo: C compiler "gcc" not found: exec: "gcc": executable file not found in $PATH

// 安装Ubuntu开发包(省事)
sudo apt install build-essential

// 再次安装Gen Tool
go install gorm.io/gen/tools/gentool@latest

cd tapi/
mkdir bkmodel

// 生成gen
gentool -dsn "root:123456@tcp(192.168.1.13:3306)/bk?charset=utf8mb4&parseTime=True&loc=Local" -outPath "./bkmodel/dao/query"

// 更新依赖
go mod tidy

```

> 注意 -outPath "./bkmodel/dao/query" 能改变的只能是bkmodel这个目录，后面的是固定的。

> 测试表增加一个字段，增加一张表，重新执行命令后新的表和字段都会生成。bkmodel下所有文件都不要修改直接使用就好

创建makefile
```
# 命令
gen:
	gentool -dsn "root:123456@tcp(192.168.1.13:3306)/bk?charset=utf8mb4&parseTime=True&loc=Local" -outPath "./bkmodel/dao/query"
```

以后要生成model直接 make gen 即可