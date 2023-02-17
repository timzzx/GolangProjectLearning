# go-zero-antd实战-6（antd关闭国际化，用户密码加密存储，增加用户退出接口）

## antd关闭国际化

> 国际化对于编写antd太麻烦了，尤其是路由部分，必须编写国际化字符串，一般对于普通内部使用后台也用不上国际化，所以决定关闭它

修改web/config/config.ts

```ts
// 原来
locale: {
    // default zh-CN
    default: 'zh-CN',
    antd: true,
    // default true, when it is true, will use `navigator.language` overwrite default
    baseNavigator: true,
  },

// 修改成
locale: false,

```

修改完会发现有一些报错去掉使用国际化的组件即可，这样国际化关闭完成

## 用户密码加密存储go-zero部分

> 之前密码是明文存储的，不安全修改掉

先新增md5方法
新增api/common/md5x/md5.go

```go
package md5x

import (
	"crypto/md5"
	"fmt"
)

// 生成MD5
func Md5(s string) string {
	m := md5.Sum([]byte(s))
	return fmt.Sprintf("%x", m)
}

```

新增加密盐值,修改api/common/varx/vars.go
```go
const PasswordSalt = "tapiSalt"
```

处理逻辑部分修改api/internal/logic/loginlogic.go
```go
// 密码加密验证
password := md5x.Md5(req.Password + varx.PasswordSalt)

// 判断密码是否正确
if user.Password != password {
    return &types.LoginResponse{
        Code: 500,
        Msg:  "密码错误",
    }, nil
}
```

> 密码盐值也可以每个用户使用不同，这里使用统一的一个值，这种自行选择处理

## 增加用户退出接口

### 处理antd部分

先去查看调用退出的地方

go-zero-antd-backend/web/src/components/RightContent/AvatarDropdown.tsx 中第62行

```ts
await outLogin();
// 这里是后加的退出完提示一下，也可以判断返回值，不过这里涉及之前全局错误处理那边已经统一处理这里只需要增加这一句即可
message.success("退出成功");

```

通过这个直接点击跳转到go-zero-antd-backend/web/src/services/ant-design-pro/api.ts

```ts
/** 退出登录接口 POST /api/login/outLogin */
export async function outLogin(options?: { [key: string]: any }) {
    // 修改了请求地址 /api/loginout
  return request<Record<string, any>>('/api/loginout', {
    method: 'POST',
    ...(options || {}),
  });
}
```

### go-zero增加退出接口
修改api/project.api
```go
// 退出请求
LoginOutRequest {
}
// 退出返回
LoginOutResponse {
    Code int64  `json:"code"`
    Msg  string `json:"msg"`
}
// 退出
@handler LoginOut
post /api/loginout(LoginOutRequest) returns (LoginOutResponse)

```
运行 make api 生成代码

修改api/internal/logic/loginoutlogic.go

```go
func (l *LoginOutLogic) LoginOut(req *types.LoginOutRequest) (resp *types.LoginOutResponse, err error) {
	// todo: add your logic here and delete this line
	// 退出 可以做一些清除处理

	return &types.LoginOutResponse{
		Code: 200,
		Msg:  "退出成功",
	}, nil
}
```

> go-zero-antd 退出操作前后端都完成了