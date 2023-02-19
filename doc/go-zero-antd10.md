# go-zero-antd实战-8（golang断言遇到的问题）

> 问题描述:今天在写用户列表功能，需要先用go-zero写一个用户列表接口。发现报错了

```
2023/02/19 19:03:49 interface conversion: interface {} is nil, not json.Number
goroutine 26 [running]:
runtime/debug.Stack()
        /usr/local/go/src/runtime/debug/stack.go:24 +0x65
main.main.func2.1.1()
        /home/code/go-zero-antd-backend/api/backend.go:53 +0xca
panic({0xee9060, 0xc0003fa780})
        /usr/local/go/src/runtime/panic.go:884 +0x212
tapi/internal/middleware.(*LoginMiddleMiddleware).Handle.func1({0x11b1b58, 0xc0004fb560}, 0xc000243600)
        /home/code/go-zero-antd-backend/api/internal/middleware/loginmiddlemiddleware.go:26 +0x139a
net/http.HandlerFunc.ServeHTTP(0x0?, {0x11b1b58?, 0xc0004fb560?}, 0x0?)
        /usr/local/go/src/net/http/server.go:2109 +0x2f
net/http.HandlerFunc.ServeHTTP(...)
        /usr/local/go/src/net/http/server.go:2109
main.main.func2.1({0x11b1b58?, 0xc0004fb560?}, 0x10?)
        /home/code/go-zero-antd-backend/api/backend.go:61 +0x73
net/http.HandlerFunc.ServeHTTP(0x0?, {0x11b1b58?, 0xc0004fb560?}, 0x0?)
        /usr/local/go/src/net/http/server.go:2109 +0x2f
github.com/zeromicro/go-zero/rest/handler.GunzipHandler.func1({0x11b1b58, 0xc0004fb560}, 0xc000243600)
        /root/go/pkg/mod/github.com/zeromicro/go-zero@v1.4.4/rest/handler/gunziphandler.go:26 +0xe9
net/http.HandlerFunc.ServeHTTP(0xedb83f915?, {0x11b1b58?, 0xc0004fb560?}, 0x601662113e?)
        /usr/local/go/src/net/http/server.go:2109 +0x2f
github.com/zeromicro/go-zero/rest/handler.MaxBytesHandler.func2.1({0x11b1b58?, 0xc0004fb560?}, 0x1870b20?)
        /root/go/pkg/mod/github.com/zeromicro/go-zero@v1.4.4/rest/handler/maxbyteshandler.go:24 +0xff
net/http.HandlerFunc.ServeHTTP(0x0?, {0x11b1b58?, 0xc0004fb560?}, 0x0?)
        /usr/local/go/src/net/http/server.go:2109 +0x2f
github.com/zeromicro/go-zero/rest/handler.MetricHandler.func1.1({0x11b1b58, 0xc0004fb560}, 0x0?)
        /root/go/pkg/mod/github.com/zeromicro/go-zero@v1.4.4/rest/handler/metrichandler.go:21 +0xd5
net/http.HandlerFunc.ServeHTTP(0x0?, {0x11b1b58?, 0xc0004fb560?}, 0x0?)
        /usr/local/go/src/net/http/server.go:2109 +0x2f
github.com/zeromicro/go-zero/rest/handler.RecoverHandler.func1({0x11b1b58?, 0xc0004fb560?}, 0x0?)
        /root/go/pkg/mod/github.com/zeromicro/go-zero@v1.4.4/rest/handler/recoverhandler.go:21 +0x83
net/http.HandlerFunc.ServeHTTP(0x0?, {0x11b1b58?, 0xc0004fb560?}, 0x0?)
        /usr/local/go/src/net/http/server.go:2109 +0x2f
github.com/zeromicro/go-zero/rest/handler.(*timeoutHandler).ServeHTTP.func1()
        /root/go/pkg/mod/github.com/zeromicro/go-zero@v1.4.4/rest/handler/timeouthandler.go:79 +0x7c
created by github.com/zeromicro/go-zero/rest/handler.(*timeoutHandler).ServeHTTP
        /root/go/pkg/mod/github.com/zeromicro/go-zero@v1.4.4/rest/handler/timeouthandler.go:73 +0x350
```

通过trace发现错误在loginmiddlemiddleware中间件
```
tapi/internal/middleware.(*LoginMiddleMiddleware).Handle.func1({0x11b1b58, 0xc0004fb560}, 0xc000243600)
        /home/code/go-zero-antd-backend/api/internal/middleware/loginmiddlemiddleware.go:26 +0x139a
```

loginmiddlemiddleware中间件是自定义的，查看代码

```go
func (m *LoginMiddleMiddleware) Handle(next http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		u := varx.Ctx.User
		rp := varx.Ctx.RolePermissionResource
		p := varx.Ctx.PermissionResource
		ctx := r.Context()
		// 就是这句报错了
        uid, err := ctx.Value("uid").(json.Number).Int64()
		if err != nil {
			httpx.OkJson(w, &types.CodeErrorResponse{
				Code: 500,
				Msg:  err.Error(), //string(debug.Stack()),
			})
			return
		}
		var resource []struct {
			Name string
			Url  string
		}
		u.WithContext(ctx).Where(u.ID.Eq(uid)).LeftJoin(rp, u.RoleID.EqCol(rp.RoleID)).LeftJoin(p, rp.Prid.EqCol(p.ID)).Where(u.Status.Eq(1)).Where(rp.Status.Eq(1)).Where(p.Status.Eq(1)).Where(p.URL.Eq(r.URL.Path)).Select(p.Name, p.URL).Scan(&resource)

		if resource == nil {
			httpx.OkJson(w, &types.CodeErrorResponse{
				Code: 500,
				Msg:  "没有权限", //string(debug.Stack()),
			})
			return
		}

		next(w, r)
	}
}
```


uid, err := ctx.Value("uid").(json.Number).Int64() 这句报错发现涉及两个问题。（这句是JWT需要使用的）
## 1.断言没有进行判断就直接使用

这句需要修改成
```go
v, ok := ctx.Value("uid").(json.Number)
if !ok {
    httpx.OkJson(w, &types.CodeErrorResponse{
        Code: 500,
        Msg:  "ctx没有uid", //string(debug.Stack()),
    })
    return
}
uid, err := v.Int64()

```

## 2.go-zero 中间件配置的问题

> 上一个问题之前没有暴露出来的原因就是因为我以为@server中间件只需要写一次，后面就都会调用，后面发现每写一次@server都需要加上需要的所有中间件。（和gin还是有区别的要注意）

```go
@server(
	jwt: Auth // 开启auth验证
)
// 这部分只验证jwt
service Backend {
	// 用户信息
	@handler UserInfo
	post /api/user/info(UserInfoRequest) returns (UserInfoResponse)
	
	@handler UserInfo2
	post /api/user/info2(UserInfoRequest) returns (UserInfoResponse)
	// 修改用户密码
	@handler EditPassword
	post /api/user/password(EditPasswordRequest) returns(EditPasswordResponse)
}

@server(
	jwt: Auth // 这个地方也需要加上，不然后面只会使用LoginMiddle中间件
	middleware: LoginMiddle
)
// 后面需要权限验证
service Backend {
	
	// 创建用户
	@handler UserAdd
	post /api/user/add(UserAddRequest) returns (UserAddResponse)
	
	// 查询用户列表
.....
}

```
