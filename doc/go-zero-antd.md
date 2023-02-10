# 后台开发实战

## 目录
[Ubuntu环境搭建](#环境搭建)<br />
[Vscode 远程连接设置](#vscode-远程连接设置)<br />
[安装Docker](#安装docker)<br />
[安装docke-compose](#安装docker-compose)<br />

## Ubuntu环境搭建

> 安装版本 ubuntu-22.04.1-live-server-amd64.iso

1. 安装Ubuntu虚拟机

    选择minimize最小化安装，其他后续再安装

    内存使用4096Mb，磁盘100G

2. 允许root ssh登录

    这个涉及到vscode远端开发的时候，如果不是root会非常麻烦

    ```
    // 先设置root密码
    sudo passwd

    // 切换到root
    su

    // 编辑
    vi /etc/ssh/sshd_config

    // 找到PermitRootLogin without-password 修改为PermitRootLogin yes

    // 重启sshd
    service ssh restart
    ```    
    


3. 配置服务器
    
    修改静态ip

    ```
    vim /etc/netplan/00-installer-config.yaml

    ```

    修改为
    ```
    # This is the network config written by 'subiquity'
    network:
    ethernets:
        enp0s3:
        dhcp4: false
        addresses: [192.168.1.13/24]
        optional: true
        routes:
            - to: default
            via: 192.168.1.1
        nameservers:
            addresses: [192.168.1.1]
    version: 2
    ```
    应用配置
    ```
    netplan apply
    ```
Ubuntu基础环境安装配置完成

## Vscode 远程连接设置

1. 新建远程连接，弹出框中输入
    ```
    ssh root@192.168.1.13 -A
    ```
    回车后记得刷新一下列表

2. 进入之后会自动安装配置vscode远端环境
    ![image](.././timg/1.png)
    然后显示
    ![image](.././timg/2.png)

3. 进入远端后终端自动连接到远端服务可以直接执行命令
    ```
    // 创建一个开发目录
    mkdir /home/code
    ```

4. 点击打开文件夹，打开刚刚创建的目录

    ![image](.././timg/3.png)
    在SSH列表中就有刚刚连接的目录了，下次直接点击连接。

## 安装Docker

官方 apt 源中就有 Docker，我们可以直接通过 apt 来安装(其他版本的自行研究)

```
sudo apt update
sudo apt install docker.io

// 安装完查看
root@tdev:~# docker ps 
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
root@tdev:~# docker -v
Docker version 20.10.12, build 20.10.12-0ubuntu4
```

配置docker国内镜像源

增加Docker的镜像源配置文件 /etc/docker/daemon.json，如果没有配置过镜像该文件默认是不存的，在其中增加如下内容：
```
{       
    "registry-mirrors": ["http://hub-mirror.c.163.com"]
}
```

```
// 重启docker服务
service docker restart

// 查看镜像
root@tdev:~# docker info|grep Mirrors -A 1
 Registry Mirrors:
  http://hub-mirror.c.163.com/
```

## 安装docker-compose

1. 通过github下载 docker-compose-linux-x86_64 

    网上有很多通过命令下载，不过有时候下载不下来，直接下载然后通过vscode直接拖拽进服务器。
    ![image](.././timg/4.png)

    ```
    mv docker-compose-linux-x86_64 /usr/local/bin/docker-compose

    sudo chmod +x /usr/local/bin/docker-compose

    // 检查是否安装完成

    root@tdev:/home/code# docker-compose 

    Usage:  docker compose [OPTIONS] COMMAND

    Docker Compose

    Options:
        --ansi string                Control when to print ANSI control
                                    characters ("never"|"always"|"auto")
                                    (default "auto")

    ....
    ```

## 编写Dockerfile
