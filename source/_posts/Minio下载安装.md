---
title: Minio下载安装
top: false
cover: false
toc: true
mathjax: true
date: 2026-04-01 22:03:00
password:
summary:
tags: 
- minio
- 存储服务
categories: 运维
---

# 					Minio下载安装

## 1. 下载安装包

```linux
wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio

wget https://dl.min.io/client/mc/release/linux-amd64/archive/mc
```

## 2. 赋予执行权限

```
sudo chmod +x /usr/bin/{minio,mcli}
```

## 3. 配置环境变量文件

1. 创建数据目录

   ```
   mkdir -p /data/service/minio
   ```

2. 配置文件

   ```
   vim /etc/default/minio
   ```

   ```
   # 访问MinIO的用户名和密码
   MINIO_ROOT_USER=leejie 
   MINIO_ROOT_PASSWORD=leejie123
   # 数据存储目录
   MINIO_VOLUMES="/data/service/minio"
   # 配置服务监听端口9000,控制台端口 9001
   MINIO_OPTS="--address :9000 --console-address :9001"
   ```

## 4. 集成systemd

> **systemd**是Linux的系统与服务管理器，它可以管理系统中的各种服务和进程，包括启动、停止和重启服务，还可以监测各服务的运行状态，并在服务异常退出时，自动拉起服务，以保证服务的稳定性。

修改配置文件

```
vim /etc/systemd/system/minio.service
```

```
[Unit]
Description=MinIO Server
Documentation=https://docs.min.io
After=network-online.target

[Service]
Type=simple
EnvironmentFile=-/etc/default/minio
ExecStart=/usr/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
ExecStop=/bin/kill -SIGTERM $MAINPID
Restart=on-failure
RestartSec=30
TimeoutStartSec=120
TimeoutStopSec=180
StartLimitIntervalSec=600
StartLimitBurst=3
KillMode=control-group
KillSignal=SIGTERM
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
EOF
```

## 常用命令

- 查看运行状态

  ```
  systemctl status minio
  ```

- 启动

  ```
  systemctl start minio
  ```

- 重启

  ```
  systemctl restart minio
  ```

- 开机自启

  ```
  systemctl enable minio
  ```

  
