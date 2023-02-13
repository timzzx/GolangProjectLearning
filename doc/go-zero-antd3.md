# go-zero单体服务（权限管理）

> 基于tapi项目[文档地址](go-zero-antd2.md#后台开发实战-2)，可以使用casbin做权限管理，为了多练习使用go-zero本文档从零开发权限管理

## 目录

[数据库表设计](#数据库表设计)<br />
[go-zero添加go-redis包](#go-zero增加redis)<br />
[获取路由URl](#获取路由url)<br />
[角色表新增，编辑，删除，列表](#角色表新增编辑删除列表)<br />
[权限资源新增删除列表](#权限资源新增删除列表)<br />

## 数据库表设计

创建role表
```sql
CREATE TABLE `role` (
  `id` bigint(20) unsigned zerofill NOT NULL AUTO_INCREMENT COMMENT '角色表',
  `name` varchar(64) NOT NULL DEFAULT '' COMMENT '角色名',
  `type` int(11) NOT NULL DEFAULT '1' COMMENT '1.普通用户，2.管理员',
  `status` int(11) NOT NULL DEFAULT '1' COMMENT '1.有效，2.无效',
  `ctime` int(11) NOT NULL DEFAULT '0' COMMENT '创建时间',
  `utime` int(11) NOT NULL DEFAULT '0' COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色表'
```
权限资源表
```sql
CREATE TABLE `permission_resource` (
  `id` bigint(20) unsigned zerofill NOT NULL AUTO_INCREMENT COMMENT '权限资源表主键',
  `name` varchar(64) NOT NULL DEFAULT '' COMMENT '资源名称',
  `url` varchar(256) NOT NULL DEFAULT '' COMMENT '资源url',
  `status` int(11) NOT NULL DEFAULT '1' COMMENT '1.有效，2.无效',
  `ctime` int(11) NOT NULL DEFAULT '0' COMMENT '创建时间',
  `utime` int(11) NOT NULL DEFAULT '0' COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='权限资源表'
```
角色权限资源关联表表
```sql
CREATE TABLE `role_permission_resource` (
  `id` bigint(20) unsigned zerofill NOT NULL AUTO_INCREMENT COMMENT '角色和权限资源关联表主键',
  `role_id` bigint(20) NOT NULL DEFAULT '0' COMMENT '角色id',
  `prid` bigint(20) NOT NULL DEFAULT '0' COMMENT '权限资源id',
  `status` int(11) NOT NULL DEFAULT '1' COMMENT '1.有效，2.无效',
  `ctime` int(11) NOT NULL DEFAULT '0' COMMENT '创建时间',
  `utime` int(11) NOT NULL DEFAULT '0' COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色和权限资源关联表'
```
用户表增加role_id字段，关联角色表
```sql
CREATE TABLE `user` (
  `id` bigint(20) unsigned zerofill NOT NULL AUTO_INCREMENT COMMENT '用户表主键',
  `name` varchar(128) NOT NULL COMMENT '用户名',
  `role_id` bigint(20) NOT NULL DEFAULT '0' COMMENT '角色id',
  `password` varchar(64) NOT NULL COMMENT '密码',
  `status` int(11) NOT NULL DEFAULT '1' COMMENT '是否有效1.有效 2.无效',
  `ctime` int(11) NOT NULL DEFAULT '0' COMMENT '创建时间',
  `utime` int(11) NOT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表'
```
![image](../timg3/1.png)
这个model显示高亮截图不好弄，只能这样

### 生成gen代码

```
make db
```

## go-zero增加redis

> 使用go-redis

修改 etc/backend.yaml
```yaml
...
# 增加redis配置（测试环境没有密码就不加了）
Redis:
  Host: 192.168.1.13:6379
  Type: node
```

修改 internal/config/config.go

```go
// 增加
Redis struct {
		Host string
		Type string
}
```

修改 internal/svc/servicecontext.go
```go
package svc

import (
	"tapi/bkmodel/dao/query"
	"tapi/internal/config"

	"github.com/redis/go-redis/v9"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

type ServiceContext struct {
	Config config.Config

	BkModel *query.Query

	Redis *redis.Client
}

func NewServiceContext(c config.Config) *ServiceContext {
	db, _ := gorm.Open(mysql.Open(c.Mysql.DataSource), &gorm.Config{})
	rdb := redis.NewClient(&redis.Options{
		Addr:     c.Redis.Host,
		Password: "", // no password set
		DB:       0,  // use default DB
	})
	return &ServiceContext{
		Config:  c,
		BkModel: query.Use(db),
		Redis:   rdb,
	}
}

```
go-redis添加完成

## 获取路由url

新建api/routers.api
```go
type (
    RouterListResquest {
		
    }
    Router {
        Method  string `json:"method"`
		Path    string  `json:"path"`
    }
    RouterListResponse {
        Code int64 `json:"code"`
        Msg string  `json:"msg"`
        Data []Router   `json:"data,optional"`
    }

)
```
编辑project.api
```go
syntax = "v1"

info(
	title: "tapi"
	desc: "接口"
	author: "tim"
	version: 1.0
)
// 新增
import "./api/routers.api"

type (
  ...
  
```

```go
service Backend {
	// 用户信息
	@handler UserInfo
	post /api/user/info(UserInfoRequest) returns (UserInfoResponse)
	
	// 创建用户
	@handler UserAdd
	post /api/user/add(UserAddRequest) returns (UserAddResponse)
	
	// 获取路由列表   新增
	@handler RouterList
	get /api/router/list(RouterListResquest) returns (RouterListResponse)
	
}
```
运行make api  发现报错了 make: 'api' is up to date.

修改makefile
```makefile
# 命令
help:
	@echo 'Usage:'
	@echo '     db 生成sql执行代码'
	@echo '     api 根据api文件生成go-zero api代码'
	@echo '     dev 运行'
db:
	gentool -dsn 'root:123456@tcp(192.168.1.13:3306)/bk?charset=utf8mb4&parseTime=True&loc=Local' -outPath './bkmodel/dao/query'
# 增加这一句
.PHONY:api 
api:
	goctl api go -api project.api -dir ./ -style gozero
dev:
	go run backend.go -f etc/backend.yaml
```
重新运行 make api 生成代码

> go-zero获取路由列表通过阅读源码得知rest/server.go中
```go
func (s *Server) Routes() []Route {
	var routes []Route

	for _, r := range s.ngin.routes {
		routes = append(routes, r.routes...)
	}

	return routes
}
```
由于没找到在handler中直接获取路由列表，那么直接在backend.go中直接调用存入一个全局变量中，再利用

新建common/varx/vars.go
```go
package varx

import "github.com/zeromicro/go-zero/rest"

var RouterList []rest.Route

```
然后在backend.go增加
``` go
package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"net/http"
	"runtime/debug"

	"tapi/common/varx"
	"tapi/internal/config"
	"tapi/internal/handler"
	"tapi/internal/svc"
	"tapi/internal/types"

	"github.com/zeromicro/go-zero/core/conf"
	"github.com/zeromicro/go-zero/rest"
	"github.com/zeromicro/go-zero/rest/httpx"
)

var configFile = flag.String("f", "etc/backend.yaml", "the config file")

func main() {
	flag.Parse()

	var c config.Config
	conf.MustLoad(*configFile, &c)

	server := rest.MustNewServer(c.RestConf)
	defer server.Stop()

	ctx := svc.NewServiceContext(c)
	handler.RegisterHandlers(server, ctx)

	httpx.SetErrorHandlerCtx(func(ctx context.Context, err error) (int, interface{}) {
		fmt.Println(err.Error())
		return http.StatusOK, &types.CodeErrorResponse{
			Code: 500,
			Msg:  err.Error(),
		}
	})

	// 全局recover中间件
	server.Use(func(next http.HandlerFunc) http.HandlerFunc {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			defer func() {
				if result := recover(); result != nil {
					log.Println(fmt.Sprintf("%v\n%s", result, debug.Stack()))
					httpx.OkJson(w, &types.CodeErrorResponse{
						Code: 500,
						Msg:  "服务器错误", //string(debug.Stack()),
					})
				}
			}()

			next.ServeHTTP(w, r)
		})
	})
	// 路由列表  增加这一句
	varx.RouterList = server.Routes()

	fmt.Printf("Starting server at %s:%d...\n", c.Host, c.Port)
	server.Start()
}

```

修改internal/logic/routerlistlogic.go
```go
package logic

import (
	"context"

	"tapi/common/varx"
	"tapi/internal/svc"
	"tapi/internal/types"

	"github.com/zeromicro/go-zero/core/logx"
)

type RouterListLogic struct {
	logx.Logger
	ctx    context.Context
	svcCtx *svc.ServiceContext
}

func NewRouterListLogic(ctx context.Context, svcCtx *svc.ServiceContext) *RouterListLogic {
	return &RouterListLogic{
		Logger: logx.WithContext(ctx),
		ctx:    ctx,
		svcCtx: svcCtx,
	}
}

func (l *RouterListLogic) RouterList(req *types.RouterListResquest) (resp *types.RouterListResponse, err error) {
	// todo: add your logic here and delete this line
	list := varx.RouterList
	if list == nil {
		return &types.RouterListResponse{
			Code: 500,
			Msg:  "获取失败",
		}, nil
	}

	var data []types.Router

	for _, item := range list {
		d := types.Router{
			Method: item.Method,
			Path:   item.Path,
		}
		if d.Path != "/api/login" {
			data = append(data, d)
		}
	}

	return &types.RouterListResponse{
		Code: 200,
		Msg:  "成功",
		Data: data,
	}, nil
}

```
执行 make dev 访问192.168.1.13:8888/api/router/list
```json
{
    "code": 200,
    "msg": "成功",
    "data": [
        {
            "method": "POST",
            "path": "/api/user/info"
        },
        {
            "method": "POST",
            "path": "/api/user/add"
        },
        {
            "method": "GET",
            "path": "/api/router/list"
        }
    ]
}
```
## 角色表新增编辑删除列表
新增api/user.api
```go
type (
	// 创建用户
	UserAddRequest {
		Name     string `form:"name"`
		Password string `form:"password"`
	}
	UserAddResponse {
		Code int64  `json:"code"`
		Msg  string `json:"msg"`
	}

    // 角色列表
    RoleListRequest {

    }
    Role {
        Id int64 `json:"id"`
        Nmae string `json:"name"`
        Type int64 `json:"type"`
        Ctime int64 `json:"ctime"`
        Utime int64 `json:"utime"`
    }
    RoleListResponse {
        Code int64    `json:"code"`
		Msg  string   `json:"msg"`
		Data []Role `json:"data,optional"`
    }

    // 角色编辑
    RoleEditRequest {
        Id int64 `form:"id"`
        Name     string `form:"name"`
        Type int64 `form:"type"`
    }
    RoleEditResponse {
        Code int64  `json:"code"`
		Msg  string `json:"msg"`
    }

    // 角色删除
    RoleDeleteRequest {
        Id int64 `form:"id"`
    }
    RoleDeleteResponse {
        Code int64  `json:"code"`
		Msg  string `json:"msg"`
    }

)
```
编辑project.api
```go
...
import (
	"./api/routers.api"
	"./api/user.api"  //新增
)

...
  // 获取路由列表
	@handler RouterList
	get /api/router/list(RouterListResquest) returns (RouterListResponse)
	
	// 角色列表 +
	@handler RoleList
	get /api/role/list(RoleListRequest) returns (RoleListResponse)
	
	// 角色编辑 +
	@handler RoleEdit
	post /api/role/edit(RoleEditRequest) returns (RoleEditResponse)
	
	// 角色删除 +
	@handler RoleDelete
	post /api/role/delete(RoleDeleteRequest) returns (RoleDeleteResponse)
	

```

执行make api生成代码


修改internal/rolelistlogic.go
```go
package logic

import (
	"context"

	"tapi/internal/svc"
	"tapi/internal/types"

	"github.com/zeromicro/go-zero/core/logx"
	"gorm.io/gorm"
)

type RoleListLogic struct {
	logx.Logger
	ctx    context.Context
	svcCtx *svc.ServiceContext
}

func NewRoleListLogic(ctx context.Context, svcCtx *svc.ServiceContext) *RoleListLogic {
	return &RoleListLogic{
		Logger: logx.WithContext(ctx),
		ctx:    ctx,
		svcCtx: svcCtx,
	}
}

func (l *RoleListLogic) RoleList(req *types.RoleListRequest) (resp *types.RoleListResponse, err error) {
	// 数据表
	role := l.svcCtx.BkModel.Role
	// 查询
	list, err := role.WithContext(l.ctx).Where(role.Status.Eq(1)).Order(role.ID).Find()
	if err != nil {
		if err != gorm.ErrRecordNotFound {
			return &types.RoleListResponse{
				Code: 500,
				Msg:  "查询失败",
			}, nil
		}
	}

	var data []types.Role

	if list != nil {
		for _, item := range list {
			d := types.Role{
				Id:    item.ID,
				Nmae:  item.Name,
				Type:  int64(item.Type),
				Ctime: int64(item.Ctime),
				Utime: int64(item.Utime),
			}
			data = append(data, d)
		}
	}

	return &types.RoleListResponse{
		Code: 200,
		Msg:  "成功",
		Data: data,
	}, nil
}

```
修改internal/roleeditlogic.go
```go
package logic

import (
	"context"
	"time"

	"tapi/bkmodel/dao/model"
	"tapi/internal/svc"
	"tapi/internal/types"

	"github.com/zeromicro/go-zero/core/logx"
)

type RoleEditLogic struct {
	logx.Logger
	ctx    context.Context
	svcCtx *svc.ServiceContext
}

func NewRoleEditLogic(ctx context.Context, svcCtx *svc.ServiceContext) *RoleEditLogic {
	return &RoleEditLogic{
		Logger: logx.WithContext(ctx),
		ctx:    ctx,
		svcCtx: svcCtx,
	}
}

func (l *RoleEditLogic) RoleEdit(req *types.RoleEditRequest) (resp *types.RoleEditResponse, err error) {
	// 数据表
	role := l.svcCtx.BkModel.Role.WithContext(l.ctx)
	if req.Id == 0 {
		// 查询看是否存在
		rs, _ := role.Where(l.svcCtx.BkModel.Role.Name.Eq(req.Name)).Where(l.svcCtx.BkModel.Role.Status.Eq(1)).First()
		if rs != nil {
			return &types.RoleEditResponse{
				Code: 500,
				Msg:  "角色已存在，请重新创建",
			}, nil
		}
		// 新增
		err = role.Create(&model.Role{
			Name:  req.Name,
			Type:  int32(req.Type),
			Ctime: int32(time.Now().Unix()),
			Utime: int32(time.Now().Unix()),
		})
		if err != nil {
			return &types.RoleEditResponse{
				Code: 500,
				Msg:  err.Error(),
			}, nil
		}
	} else {
		// 更新
		r, err := role.Where(l.svcCtx.BkModel.Role.ID.Eq(req.Id)).Updates(model.Role{
			Name:  req.Name,
			Utime: int32(time.Now().Unix()),
		})
		if err != nil {
			return &types.RoleEditResponse{
				Code: 500,
				Msg:  err.Error(),
			}, nil
		}
		if r.Error != nil {
			return &types.RoleEditResponse{
				Code: 500,
				Msg:  r.Error.Error(),
			}, nil
		}
	}

	return &types.RoleEditResponse{
		Code: 200,
		Msg:  "成功",
	}, nil
}

```
修改internal/roledeletelogic.go
```go
package logic

import (
	"context"
	"time"

	"tapi/bkmodel/dao/model"
	"tapi/internal/svc"
	"tapi/internal/types"

	"github.com/zeromicro/go-zero/core/logx"
)

type RoleDeleteLogic struct {
	logx.Logger
	ctx    context.Context
	svcCtx *svc.ServiceContext
}

func NewRoleDeleteLogic(ctx context.Context, svcCtx *svc.ServiceContext) *RoleDeleteLogic {
	return &RoleDeleteLogic{
		Logger: logx.WithContext(ctx),
		ctx:    ctx,
		svcCtx: svcCtx,
	}
}

func (l *RoleDeleteLogic) RoleDelete(req *types.RoleDeleteRequest) (resp *types.RoleDeleteResponse, err error) {
	// 数据表
	role := l.svcCtx.BkModel.Role
	// 删除(标记删除)
	r, err := role.WithContext(l.ctx).Where(role.ID.Eq(req.Id)).Updates(model.Role{
		Status: 2,
		Utime:  int32(time.Now().Unix()),
	})
	if err != nil {
		return &types.RoleDeleteResponse{
			Code: 500,
			Msg:  err.Error(),
		}, nil
	}
	if r.Error != nil {
		return &types.RoleDeleteResponse{
			Code: 500,
			Msg:  r.Error.Error(),
		}, nil
	}

	return &types.RoleDeleteResponse{
		Code: 200,
		Msg:  "成功",
	}, nil
}

```

完成后运行make dev

### 测试
#### 新增
192.168.1.13:8888/api/role/edit?name=admin&type=2&id=0
```json
{
    "id": 0,
    "code": 200,
    "msg": "成功"
}
// 失败
{
    "id": 0,
    "code": 500,
    "msg": "角色已存在，请重新创建"
}

```
> 问题：暂时没有处理lastinsertId,gorm要用事务处理。
#### 编辑
192.168.1.13:8888/api/role/edit?name=admin1&type=2&id=1
```json
{
    "id": 0,
    "code": 200,
    "msg": "成功"
}
```
#### 列表
192.168.1.13:8888/api/role/list
```json
{
    "code": 200,
    "msg": "成功",
    "data": [
        {
            "id": 1,
            "name": "admin1",
            "type": 2,
            "ctime": 1676290849,
            "utime": 1676291007
        }
    ]
}
```
#### 删除
192.168.1.13:8888/api/role/delete?id=1
```json
{
    "code": 200,
    "msg": "成功"
}
```


## 权限资源新增删除列表

> go-zero单体服务（权限管理-1）获取路由列表就是为了这个接口，提供可用的URL，其实也可以手动编写URL来传入这个接口。（手动不安全还是通过代码获取比较合理）

新建 api/permission_resource.api
```
type(
    PermissionResourceListRequest {
    }
    PermissionResource {
        Name string `form:"name"`
        Url string `form:"url"`
        Ctime int64 `form:"ctime"`
    }
    PermissionResourceListResponse {
        Code int64 `json:"code"`
        Msg string `json:"msg"`
        Data []PermissionResource `json:"data"`
    }
    PermissionResourceEditRequest {
        Id int64 `form:"id"`
        Name string `form:"name"`
        Url string `form:"url"`
    }
    PermissionResourceEditResponse {
        Code int64 `json:"code"`
        Msg string `json:"msg"`
    }
    PermissionResourceDeleteRequest {
        Id int64 `form:"id"`
    }
    PermissionResourceDeleteResponse {
        Code int64 `json:"code"`
        Msg string `json:"msg"`
    }
)
```

修改project.api

```
...
import (
	"./api/routers.api"
	"./api/user.api"
	"./api/permission_resource.api"
)
...
// 权限资源列表
@handler PermissionResourceList
post /api/permission/resource/list(PermissionResourceListRequest) returns (PermissionResourceListResponse)

// 权限资源列表
@handler PermissionResourceEdit
post /api/permission/resource/edit(PermissionResourceEditRequest) returns (PermissionResourceEditResponse)

// 权限资源删除
@handler PermissionResourceDelete
post /api/permission/resource/delete(PermissionResourceDeleteRequest) returns (PermissionResourceDeleteResponse)
	

```

运行 make api 生成代码

修改 internal/logic/permissionresourcelistlogic.go

``` go
func (l *PermissionResourceListLogic) PermissionResourceList(req *types.PermissionResourceListRequest) (resp *types.PermissionResourceListResponse, err error) {
	p := l.svcCtx.BkModel.PermissionResource
	pr, err := p.WithContext(l.ctx).Where(p.Status.Eq(1)).Find()

	if err != nil {
		return &types.PermissionResourceListResponse{
			Code: 500,
			Msg:  err.Error(),
		}, nil
	}
	var data []types.PermissionResource
	if pr != nil {
		for _, item := range pr {
			d := types.PermissionResource{
				Name:  item.Name,
				Url:   item.URL,
				Ctime: int64(item.Ctime),
			}
			data = append(data, d)
		}
	}

	return &types.PermissionResourceListResponse{
		Code: 200,
		Msg:  "成功",
		Data: data,
	}, nil
}
```
修改 internal/logic/permissionresourceeditlogic.go
```go
func (l *PermissionResourceEditLogic) PermissionResourceEdit(req *types.PermissionResourceEditRequest) (resp *types.PermissionResourceEditResponse, err error) {
	p := l.svcCtx.BkModel.PermissionResource

	// 只做新增，不做修改
	// 查询数据是否存在
	rp, _ := p.WithContext(l.ctx).Where(p.URL.Eq(req.Url)).Where(p.Status.Eq(1)).First()
	if rp != nil {
		return &types.PermissionResourceEditResponse{
			Code: 500,
			Msg:  "资源已存在",
		}, nil
	}

	// 新建
	err = p.WithContext(l.ctx).Create(&model.PermissionResource{
		Name:  req.Name,
		URL:   req.Url,
		Ctime: int32(time.Now().Unix()),
		Utime: int32(time.Now().Unix()),
	})

	if err != nil {
		return &types.PermissionResourceEditResponse{
			Code: 500,
			Msg:  "新建失败",
		}, nil
	}

	return &types.PermissionResourceEditResponse{
		Code: 200,
		Msg:  "成功",
	}, nil
}
```

修改 internal/logic/permissionresourcedeletelogic.go
```go
func (l *PermissionResourceDeleteLogic) PermissionResourceDelete(req *types.PermissionResourceDeleteRequest) (resp *types.PermissionResourceDeleteResponse, err error) {
	p := l.svcCtx.BkModel.PermissionResource

	// 标记删除
	_, err = p.WithContext(l.ctx).Where(p.ID.Eq(req.Id)).Updates(&model.PermissionResource{
		Status: 2,
		Utime:  int32(time.Now().Unix()),
	})

	if err != nil {
		return &types.PermissionResourceDeleteResponse{
			Code: 500,
			Msg:  err.Error(),
		}, nil
	}

	return &types.PermissionResourceDeleteResponse{
		Code: 200,
		Msg:  "成功",
	}, nil
}

```

运行 make:dev

### 创建
192.168.1.13:8888/api/permission/resource/edit?name=用户信息&url=/api/user/info&id=0

``` json
{
    "code": 200,
    "msg": "成功"
}
```

### 查询
192.168.1.13:8888/api/permission/resource/list
```json
{
    "code": 200,
    "msg": "成功",
    "data": [
        {
            "Name": "用户信息",
            "Url": "/api/user/info",
            "Ctime": 1676297045
        }
    ]
}
```

### 删除
192.168.1.13:8888/api/permission/resource/delete?id=1

``` json
{
    "code": 200,
    "msg": "成功"
}
```

## 角色关联权限资源新增，删除


## 用户新增，编辑，删除


## 用户登录（修改）