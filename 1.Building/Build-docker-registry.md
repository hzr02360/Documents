**作成日付:2024/07/15**

# 仮想マシン上に Docker Private Registry 環境を構築

## 参照サイト

- https://www.server-world.info/query?os=Ubuntu_22.04&p=docker&f=8
- http://bluearth.cocolog-nifty.com/blog/2019/08/post-720ccd.html

## Docker をインストール

### install

```bash
bravog@docker:/home/share$ sudo apt-get remove docker docker-engine docker.io containerd runc
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
E: Unable to locate package docker-engine
bravog@docker:/home/share$ sudo apt-get update
bravog@docker:/home/share$  sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
bravog@docker:/home/share$
bravog@docker:/home/share$ sudo mkdir -p /etc/apt/keyrings
bravog@docker:/home/share$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
bravog@docker:/home/share$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
bravog@docker:/home/share$ sudo apt-get update
bravog@docker:/home/share$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### Docker のバージョン確認と動作確認

```bash
bravog@docker:/home/share$ sudo docker -v
Docker version 27.0.3, build 7d4bcd8
bravog@docker:/home/share$ sudo docker run hello-world
bravog@docker:/home/share$
```

### 一般ユーザで Docker を実行できるようにする

```bash
bravog@docker:/home/share$ getent group | grep docker
docker:x:999:
bravog@docker:/home/share$ sudo usermod -aG docker $USER
bravog@docker:/home/share$ getent group | grep docker
docker:x:999:bravog
bravog@docker:/home/share$ exit
logout
Connection to docker closed.
bravog@docker:/home/share$
```

```bash
bravog@docker:~$ id
uid=1000(bravog) gid=1000(bravog) groups=1000(bravog),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),110(lxd),999(docker)
bravog@docker:~$ docker run hello-world
```

## Docker-registry

### Docker-registry をインストール

```bash
root@docker:/etc/ssl/private# apt -y install docker-registry
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  docker-registry
0 upgraded, 1 newly installed, 0 to remove and 59 not upgraded.
Need to get 4,701 kB of archives.
After this operation, 17.7 MB of additional disk space will be used.
Get:1 http://jp.archive.ubuntu.com/ubuntu jammy/universe amd64 docker-registry amd64 2.8.0+ds1-4 [4,701 kB]
Fetched 4,701 kB in 2s (2,011 kB/s)
Selecting previously unselected package docker-registry.
(Reading database ... 110410 files and directories currently installed.)
Preparing to unpack .../docker-registry_2.8.0+ds1-4_amd64.deb ...
Unpacking docker-registry (2.8.0+ds1-4) ...
Setting up docker-registry (2.8.0+ds1-4) ...
Adding system user `docker-registry' (UID 114) ...
Adding new group `docker-registry' (GID 119) ...
Adding new user `docker-registry' (UID 114) with group `docker-registry' ...
Not creating home directory `/var/lib/docker-registry'.
Created symlink /etc/systemd/system/multi-user.target.wants/docker-registry.service → /lib/systemd/system/docker-registr
y.service.
Scanning processes...
Scanning linux images...

Running kernel seems to be up-to-date.

No services need to be restarted.

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.
root@docker:/etc/ssl/private#
```

## 認証なしで docker-registry にアクセスする

### /etc/docker/registry/config.yml を編集

- tls と auth をコメントアウトする

```yaml
version: 0.1
log:
  fields:
    service: registry
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/docker-registry
  delete:
    enabled: true
http:
  addr:
    :5000
    #    tls:
    #      certificate: /etc/docker/certs.d/docker.virt.net:5000/server.crt
    #      key: /etc/docker/certs.d/docker.virt.net:5000/server.key
  headers:
    X-Content-Type-Options:
      [nosniff]
      #auth:
      #  htpasswd:
      #    realm: basic-realm
      #    path: /etc/docker/registry
health:
  storagedriver:
    enabled: true
    interval: 10s
    threshold: 3
```

### docker と docker-registry を再起動する

```bash
bravog@docker:~$ sudo systemctl restart docker
bravog@docker:~$ sudo systemctl restart docker-registry
bravog@docker:~$
```

### 動作確認(docker-registry を構築したサーバで確認)

```bash
bravog@docker:/etc/docker/registry$ docker images
REPOSITORY                        TAG               IMAGE ID       CREATED          SIZE
docker.virt.net:5000/my-iproute   latest            a1d0f798e055   19 minutes ago   125MB
docker.virt.net:5000/message      1.0               d18f95999a1e   5 hours ago      10.8MB
ubuntu                            latest            edbfe74c41f8   4 weeks ago      78.1MB
nginx                             latest            fffffc90d343   2 months ago     188MB
docker.virt.net:5000/nginx        my-registry       fffffc90d343   2 months ago     188MB
docker.virt.net:5000/nginx        private-registr   fffffc90d343   2 months ago     188MB
registry                          2                 cfb4d9904335   11 months ago    25.4MB
registry                          latest            cfb4d9904335   11 months ago    25.4MB
hello-world                       latest            d2c94e258dcb   16 months ago    13.3kB
bravog@docker:/etc/docker/registry$ docker push docker.virt.net:5000/message:1.0
The push refers to repository [docker.virt.net:5000/message]
538988016e2d: Pushed
1f35d3f5b66c: Pushed
78561cef0761: Pushed
1.0: digest: sha256:099f541466b983d7042cdffca1c8a97f43d4427cd95be2a32881360249334026 size: 946
bravog@docker:/etc/docker/registry$ curl http://docker.virt.net:5000/v2/_catalog
{"repositories":["message","nginx"]}
bravog@docker:/etc/docker/registry$ docker pull docker.virt.net:5000/message:1.0
1.0: Pulling from message
Digest: sha256:099f541466b983d7042cdffca1c8a97f43d4427cd95be2a32881360249334026
Status: Image is up to date for docker.virt.net:5000/message:1.0
docker.virt.net:5000/message:1.0
bravog@docker:/etc/docker/registry$
```

## クライアント(kubernetes 各ノード)で docker-registry にアクセスできるようにする

- kubernetes 各ノードは、containerd のみで docker は存在しない
- 各ノード同じ設定を実施するため、１ノードのみ記載する

### /et/hosts に docker-registry を構築したサーバを追加する

```bash
bravog@k8s-worker1:$ sudo vi /etc/hosts
bravog@k8s-worker1:$
```

- 今回は以下のような設定を追加

```
192.168.122.192 docker.virt.net docker-registry
```

### containerd の設定ファイルを作成する

```bash
bravog@k8s-worker1:/etc$ cd !$
cd containerd
bravog@k8s-worker1:/etc/containerd/certs.d/docker.virt.net:5000$ cd ../../
bravog@k8s-worker1:/etc/containerd$ containerd config default | sudo tee /etc/containerd/config.toml
```

### docker-registry の設定を追加する

```bash
bravog@k8s-worker1:/etc/containerd$ sudo mkdir certs.d
bravog@k8s-worker1:/etc/containerd$ cd !$
cd certs.d
bravog@k8s-worker1:/etc/containerd/certs.d$ sudo mkdir docker.virt.net:5000
bravog@k8s-worker1:/etc/containerd/certs.d$ cd !$
cd docker.virt.net:5000
bravog@k8s-worker1:/etc/containerd/certs.d/docker.virt.net:5000$ sudo vi hosts.toml
bravog@k8s-worker1:/etc/containerd/certs.d/docker.virt.net:5000$
```

- 今回は以下のような設定を追加

```toml
server = "https://registry-1.docker.io"

[host."http://192.168.122.192:5000"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
[host."http://docker.virt.net:5000"]
  capabilities = ["pull", "resolve", "push"]
  skip_verify = true
```

### containerd を再起動

```bash
bravog@k8s-worker1:/etc/containerd$ sudo systemctl restart containerd
Job for containerd.service failed because the control process exited with error code.
See "systemctl status containerd.service" and "journalctl -xeu containerd.service" for details.
bravog@k8s-worker1:/etc/containerd$
```

- 何故か再起動に失敗するのでサーバ自体再起動で対応した

### 動作確認

```bash
bravog@k8s-master:/etc/containerd$ kubectl run busybox-tmp -n default --image=docker.virt.net:5000/nginx:my-registry --command -- tail -f /dev/null
pod/busybox-tmp created
bravog@k8s-master:/etc/containerd$ kubectl get pods
NAME          READY   STATUS    RESTARTS   AGE
busybox-tmp   1/1     Running   0          72s
bravog@k8s-master:/etc/containerd$ kubectl describe pod -n default busybox-tmp | grep -A 32 ^Event
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  86s   default-scheduler  Successfully assigned default/busybox-tmp to k8s-worker2
  Normal  Pulled     85s   kubelet            Container image "docker.virt.net:5000/nginx:my-registry" already present on machine
  Normal  Created    85s   kubelet            Created container busybox-tmp
  Normal  Started    84s   kubelet            Started container busybox-tmp
bravog@k8s-master:/etc/containerd$
```
