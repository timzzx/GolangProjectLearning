# go-zero-antd实战-7（antd与go-zero放在一个目录下eslint问题，修改用户密码，前后端完整过程整理）

> 标题写这么长目的就是记录eslint问题,安装正常的开发前后端项目还是单独分开，放在一起vscode有一些问题。

## eslint问题

> 写前端的时候发现，ts代码没有自动格式化，就安装了eslint扩展，antd已经配置好eslint文件，反复尝试不起作用，后来把配置文件放在根目录就可以了

.eslintrc.js
```ts
module.exports = {
    extends: [require.resolve('/home/code/go-zero-antd-backend/web/node_modules/@umijs/lint/dist/config/eslint')],
    globals: {
        page: true,
        REACT_APP_ENV: true,
    },
};
```

## 增加用户修改密码功能

> 记录一下完整的前后端开发过程

### go-zero增加修改密码接口

#### 修改 project.api
```
// 修改密码请求
EditPasswordRequest {
    Password string `json:"password"`
}

// 修改密码返回
EditPasswordResponse {
    Code int64  `json:"code"`
    Msg  string `json:"msg"`
}

...

// 修改用户密码
@handler EditPassword
post /api/user/password(EditPasswordRequest) returns(EditPasswordResponse)

```
goctl api go -api project.api -dir ./ -style gozero

#### 修改/home/code/go-zero-antd-backend/api/internal/logic/editpasswordlogic.go

```go
// 逻辑处理
func (l *EditPasswordLogic) EditPassword(req *types.EditPasswordRequest) (resp *types.EditPasswordResponse, err error) {
	uid, _ := l.ctx.Value("uid").(json.Number).Int64()
	// user表
	u := l.svcCtx.BkModel.User
	// 密码加密
	password := md5x.Md5(req.Password + varx.PasswordSalt)

	// 更新用户密码
	_, err = u.WithContext(l.ctx).Where(u.ID.Eq(uid)).Updates(model.User{
		Password: password,
		Utime:    int32(time.Now().Unix()),
	})

	if err != nil {
		return &types.EditPasswordResponse{
			Code: 500,
			Msg:  "更新失败",
		}, nil
	}
	return &types.EditPasswordResponse{
		Code: 200,
		Msg:  "更新成功",
	}, nil
}
```

> 接口完成

### antd增加service

```ts
// 修改密码参数
type EditPasswordParam = {
    password: string;
} 
// 修改密码结果
type EditPasswordResult = {
    code?: number;
    msg?: string;
}


// 修改密码 json提交
export async function editPassword(body: USER.EditPasswordParam, options?: { [key: string]: any }) {
    return request<USER.EditPasswordResult>('/api/user/password',{
        method: 'POST',
        headers: {
        'Content-Type': 'application/json',
        },
        data: body, 
        ...(options || {}),
    });
}
```

### 新增pages
go-zero-antd-backend/web/src/pages/User/UserInfo.tsx

```ts
import { PageContainer, ProCard, ProColumns, ProForm, ProFormText, ProTable } from '@ant-design/pro-components';
import { message } from 'antd';
import { editPassword } from '@/services/user/api';
import React from 'react';
import { history } from '@umijs/max';

const UserInfo: React.FC = () => {
    return (
        <PageContainer>
            <ProCard title="用户信息编辑">
                <ProForm
                    onFinish={async (values: any) => {
                        const data = await editPassword(values);
                        if (data.code == 200) {
                            message.success("修改成功")
                            localStorage.setItem("token", '');
                            history.push('/user/login');
                        }
                    }}
                >
                    <ProFormText
                        name="password"
                        label="密码"
                        rules={[
                            {
                                required: true,
                                message: '密码长度在6-20',
                                min: 6,
                                max: 20,
                            },
                        ]}
                    >
                    </ProFormText>
                </ProForm>
            </ProCard>
        </PageContainer>

    )
}

export default UserInfo;

```

修改路由go-zero-antd-backend/web/config/routes.ts

```ts
// 用户管理
  {
    path: '/user',
    name: '用户管理',
    icon: 'crown',
    routes: [
      {
        path: '/user/userinfo',
        name: '用户信息',
        component: './User/UserInfo',
      },
    ],
  },
```

## 源码已上传

[地址](https://github.com/timzzx/go-zero-antd-backend)
