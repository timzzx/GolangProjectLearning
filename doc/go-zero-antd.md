# 后台开发实战

## 目录
[Ubuntu环境搭建](#环境搭建)<br />
[Vscode 远程连接设置](#vscode-远程连接设置)<br />

## Ubuntu环境搭建

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