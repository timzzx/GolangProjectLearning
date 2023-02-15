# go-zero-antd实战-1（项目启动）

## 前言

> 阅读本文档之前先阅读之前的[文档](https://github.com/timzzx/GolangProjectLearning),本文档的[源码地址](https://github.com/timzzx/go-zero-antd-backend)

```
.
├── LICENSE
├── README.md
├── api // go-zero单体服务目录
│   ├── LICENSE
│   ├── README.md
│   ├── api
│   ├── backend.go
│   ├── bkmodel
│   ├── common
│   ├── doc
│   ├── etc
│   ├── go.mod
│   ├── go.sum
│   ├── internal
│   ├── makefile
│   └── project.api
├── dockerdev // docker-compose目录
│   ├── LICENSE
│   ├── README.md
│   ├── docker-compose.yml
│   ├── mysql
│   └── redis
└── web // antd目录
    ├── LICENSE
    ├── README.md
    ├── config
    ├── jest.config.ts
    ├── jsconfig.json
    ├── mock
    ├── node_modules
    ├── package.json
    ├── pnpm-lock.yaml
    ├── public
    ├── src
    ├── tests
    ├── tsconfig.json
    ├── types
    └── yarn.lock

18 directories, 20 files
```

## 项目下载启动
```
mkdir go-zero-antd-backend
cd go-zero-backend
mkdir api
mkdir web
mkdir dockerdev

// 下载go-zero项目
git clone https://github.com/timzzx/tapi.git api/

// 下载antd pro项目
git clone https://github.com/timzzx/antds.git web

// 下载docker-compose项目
git clone https://github.com/timzzx/GoDockerDev.git dockerdev

// 启动docker-compose
cd dockerdev
docker-compose -d

// 启动go-zero单体服务
cd api
make dev

// 启动antd
cd web
yarn dev
```

以上项目启动完成。
