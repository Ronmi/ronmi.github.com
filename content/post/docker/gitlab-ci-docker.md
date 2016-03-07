+++
date = "2016-03-07T13:44:17+08:00"
title = "gitlab-ci and docker"
tags = [ "docker", "gitlab" ]
categories = ["docker", "gitlab" ]
+++

最近在學 CI/CD 相關的課題，研究了一下 Jenkins 但是覺得太過肥大。GitLab 本體雖然也是肥肥的 ruby，不過 gitlab-ci 預設的 gitlab-multirunner 到是 go 寫的，相當輕量；能與 GitLab 深度整合這點也還不錯。

gitlab-multirunner 支援 docker。由於我的 GitLab 本體和 ci 都是裝在 docker 裡面的。為了要在 build job 中做 docker build，所以一開始考慮用 docker-in-docker (dind)。不過 dind 有效能和 cgroup namespace 等等問題，所以後來改成將外部 docker socket 掛入 container 入的方式。這衍生了另一個問題：gitlab-multirunner 會針對每一個 job 下載必要的 image、開啟複數個 container，讓外部的 docker 看起來十分複雜，造成管理的困難。

這篇記錄了我的另一個解法：啟動另一個獨立的 docker daemon 專供 build/test/deploy job 使用。

**註：以下適用 docker 1.10 以後的版本，舊版「理論上可以」，但我沒有測試，不保證能通用**

<!--more-->

# 準備工作

由於 Docker 本身的設計並不適合讓兩個 docker daemon 共用資源，所以要先把網路、儲存設備先準備好。

## 網路

Docker 預設使用 `docker0` 這個橋接介面，所以我們針對第二個 docker daemon 建立一個新的橋接器，再用 `-b` 指定給它：

```sh
brctl addbr docker1
ip addr add 10.9.0.1/16 dev docker1
ip link set dev docker1 up
```

或是用 systemd

`/etc/systemd/network/docker1.netdev`:

```
[NetDev]
Name=docker1
Kind=bridge
```

`/etc/systemd/network/docker1.network`:

```
[Match]
Name=docker1

[Network]
Address=10.9.0.1/16
```

Reload

```sh
systemctl daemon-reload
systemctl restart systemd-networkd
```

ip 要記得調開，不要跟已知的網段 (包括第一個 docker) 重覆。

## 儲存設備

```sh
mkdir -p /docker2/graph /docker2/tmp
```

graph 目錄是存放 image/container 用的，tmp 則是暫存目錄；pid/socket 存放在 `/docker2`

# Wrapper

```sh
#!/bin/bash

DOCKER_HOST=unix:///docker2/docker.sock
DOCKER_TMPDIR=/docker2/tmp

export DOCKER_HOST DOCKER_TMPDIR

exec docker "$@"
```

另存成 `/usr/local/bin/docker2`

**註：不要忘了加執行權限**

# init 設定

為了簡便，第二個 docker 會使用 loop devicemapper。

## systemd

先把 `/lib/systemd/system/docker.socket` 和 `/lib/systemd/system/docker.service` 複製一份到 `/etc/systemd/system`，然後把複本檔名裡的 `docker` 改成 `docker2`。

`docker2.socket` 要改的地方有兩個： (說明部份也可以視需要更改)

- PartOf: 把 `docker` 改成 `docker2`
- ListenStream: 改成 `/docket2/docker.sock`

`docker2.service` 要改的有三個: (說明部份也可以視需要更改)

- After: 把 `docker` 改成 `docker2`
- Requires: 把 `docker` 改成 `docker2`
- ExecStart: 改成 `/usr/bin/docker daemon -H fd:// -g /docker2/graph -s devicemapper -p /docker2/docker.pid -b docker1`。視需要加上其他參數，比如 `--dns` 或 `--insecure-registry` 等等

啟動試試

```sh
systemctl daemon-reload
systemctl start docker2
docker2 info
```

正常的話就可以設為預設啟動了

```sh
systemctl enable docker2
```

# 建立 gitlab-ci runner

```sh
docker run -d --name runner1 --hostname runner1 -v /docker2/docker.sock:/var/run/docker.sock --privileged --restart always gitlab/gitlab-runner
docker exec -it runner1 gitlab-runner register -n -r "gitlab-runner-register-token" --url "http://your.gitlab.url/ci" --executor docker --docker-image gitlab/dind --docker-privileged --name runner1 --docker-volumes /var/run/docker.sock:/var/run/docker.sock
```

說明一下 gitlab-runner register 的參數

- `-r`: register token
- `--url`: GitLab 本體提供的 API 位址，如果本體跟 runner 跑在同一台機器的 docker 裡的話，建立 runner container 的時候可能要用 `--link` 連起來
- `--executor`: 指定 job executor
- `--docker-image`: 預設用來執行 job 的 image
- `--docker-privileged`: 不加這個就不能存取外部的 docker 了
- `--docker-volumes`: 把 runner 裡的 docker socket 掛進 job container 裡。由於在建立 runner container 的時候我們把第二個 docker 的 socket 指定給 runner 了，所以 job container 存取到的會是第二個 docker

# 完成

指定一個 build job 給新的 runner，應該就可以用 `docker2 ps -a` 和 `docker2 images` 看到 job container 及相關資訊。
