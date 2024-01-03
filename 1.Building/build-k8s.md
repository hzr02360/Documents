**作成日付:2023/12/29**

# 仮想マシン上に kubernetes 環境を構築

## 仮想マシンのスペック

- Ubuntu22.04：10GB/4cpu/2GB/サーバ構成
- 192.168.122.1/24
- 以下 3 台構成
  192.168.122.181 k8s-master.localdomain k8s-master
  192.168.122.182 k8s-worker1.localdomain k8s-worker1
  192.168.122.183 k8s-worker2.localdomain k8s-worker2
- kubernetes の構成
  - containerd
  - carico

## 仮想マシン雛形作成

### コンテナランタイム

```shell
$ apt -y install containerd
$ cat > /etc/sysctl.d/99-k8s-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
$ sysctl --system
$ modprobe overlay; modprobe br_netfilter
$ echo -e overlay\\nbr_netfilter > /etc/modules-load.d/k8s.conf
$ update-alternatives --config iptables

There are 2 choices for the alternative iptables (providing /usr/sbin/iptables).

  Selection    Path                       Priority   Status
------------------------------------------------------------
* 0            /usr/sbin/iptables-nft      20        auto mode
  1            /usr/sbin/iptables-legacy   10        manual mode
  2            /usr/sbin/iptables-nft      20        manual mode

Press <enter> to keep the current choice[*], or type selection number: 1
update-alternatives: using /usr/sbin/iptables-legacy to provide /usr/sbin/iptables (iptables) in manual mode
```

### その他設定

- swap 無効化

```shell
$ swapoff -a
```

- swap 行はコメントアウト

```shell
$ vi /etc/fstab
→#/swap.img      none    swap    sw      0       0
```

- Cgroup v1 への切り替え (デフォルトは v2)

```shell
$ vi /etc/default/grub
→11行目に以下の定義を追記
GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=0"
$ update-grub
```

### install kubeadm kubelet kubectl

- NG→Expired key - "Google Cloud Packages Automatic Signing Key"

```shell
$ curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg -o /etc/apt/keyrings/kubernetes-keyring.gpg
$ echo "deb [signed-by=/etc/apt/keyrings/kubernetes-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
$ apt update
```

- OK

```shell
$ mkdir -p /etc/apt/keyrings
$ echo "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
$ curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg
$ apt update
$ apt -y install kubeadm kubelet kubectl
$ apt-mark hold kubelet kubeadm kubectl
```

### cgroup ドライバーの設定

```shell
$ vi /etc/default/kubelet
=====
KUBELET_EXTRA_ARGS=--cgroup-driver=systemd --container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock
=====
```

### containerd 設定

```shell
$ systemctl edit containerd.service
=====
[Service]
KillMode=
KillMode=mixed
=====
$ systemctl restart containerd.service
```

## 仮想マシンのコピー

### イメージのコピー

- 以下の条件での例を記載
  - コピー元のイメージ名:ubuntu22.04
  - コピー先のイメージ名:k8s-master
  - イメージファイル名:/var/lib/libvirt/images/k8s-master.qcow2

### イメージの確認

```shell
$ virsh list --all
 Id   Name          State
------------------------------
 -    ubuntu22.04   shut off
```

### イメージコピー

```shell
$ virt-clone --original ubuntu22.04 --name k8s-master --file /var/lib/libvirt/images/k8s-master.qcow2
'k8s-master.qcow2' を割り当てています                   8% [==-                                ]    0 B/s | 593 MB  --:-'k8s-master.qcow2' を割り当てています                  12%
[====                               ] 123 MB/s | 865 MB  00:0'k8s-master.qcow2' を割り当てています                  17%
[======                             ] 172 MB/s | 1.2 GB  00:0'k8s-master.qcow2' を割り当てています                  20%
[=======                            ] 191 MB/s | 1.5 GB  00:0'k8s-master.qcow2' を割り当てています                  26%
[=========                          ] 240 MB/s | 1.9 GB  00:0'k8s-master.qcow2' を割り当てています                  31%
[===========                        ] 258 MB/s | 2.2 GB  00:0'k8s-master.qcow2' を割り当てています                                                                     | 2.2 GB  00:00:06 ...

'k8s-master' の複製に成功しました。
```

### コピー結果確認

```shell
bravog@bravog:~ $ virsh list --avirsh list --all
 Id   Name          State
------------------------------
 -    k8s-master    shut off
 -    ubuntu22.04   shut off
```

## ホスト OS とのファイル共有の設定（必要に応じて）(guest)

```shell
root@ubuntu2204:/home$ mkdir share
root@ubuntu2204:/home$ mount -t 9p -o trans=virtio share /home/share
root@ubuntu2204:/home$ ls -l
total 5
drwxr-x--- 4 bravog bravog 4096 Nov 24 06:56 bravog
drwxrwxr-x 2 bravog bravog    2 Nov 26 12:49 share
root@ubuntu2204:/home$ df -hkl
Filesystem                        1K-blocks     Used Available Use% Mounted on
tmpfs                                400592     1140    399452   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   5327504  3608584   1427492  72% /
tmpfs                               2002948        0   2002948   0% /dev/shm
tmpfs                                  5120        0      5120   0% /run/lock
tmpfs                                  4096        0      4096   0% /sys/fs/cgroup
/dev/vda2                           1768056   131944   1527980   8% /boot
tmpfs                                400588        4    400584   1% /run/user/1000
share                             220534656 16460800 204073856   8% /home/share
root@ubuntu2204:/home$ vi /etc/fstab
======
share /home/share/ 9p trans=virtio,version=9p2000.L,nobootwait,rw,_netdev 0 0
======
```

## IP アドレスの変更＋固定化(guest)

_netplan を書くときに dhcp を無効(false)にすることを忘れずに_

```shell
root@ubuntu2204:/home$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:13:02:cf brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.182/24 metric 100 brd 192.168.122.255 scope global dynamic enp1s0
       valid_lft 2909sec preferred_lft 2909sec
    inet6 fe80::5054:ff:fe13:2cf/64 scope link
       valid_lft forever preferred_lft forever
root@ubuntu2204:/home$ cd /etc/netplan/
root@ubuntu2204:/etc/netplan$ ls -l
total 4
-rw-r--r-- 1 root root 117 Nov 24 05:58 00-installer-config.yaml
root@ubuntu2204:/etc/netplan$ vi 99-config.yaml
=====
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.122.181/24
      routes:
        - to: default
          via: 192.168.122.1
      nameservers:
          addresses: [8.8.8.8, 8.8.4.4]
=====
root@ubuntu2204:/etc/netplan$ netplan apply
root@ubuntu2204:/etc/netplan$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:13:02:cf brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.181/24 brd 192.168.122.255 scope global enp1s0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe13:2cf/64 scope link
       valid_lft forever preferred_lft forever
```

## ホスト名の変更(guest)

```shell
root@ubuntu2204:/etc/netplan$ hostnamectl
 Static hostname: ubuntu2204
       Icon name: computer-vm
         Chassis: vm
      Machine ID: 732a1bead3d64bf891f9478869b4c744
         Boot ID: 3c74a969b28c46ca972d0edfd0a8e0b0
  Virtualization: kvm
Operating System: Ubuntu 22.04.3 LTS
          Kernel: Linux 5.15.0-89-generic
    Architecture: x86-64
 Hardware Vendor: QEMU
  Hardware Model: Standard PC _Q35 + ICH9, 2009_
root@ubuntu2204:/etc/netplan$ hostnamectl set-hostname k8s-master
root@ubuntu2204:/etc/netplan$ hostnamectl
 Static hostname: k8s-master
       Icon name: computer-vm
         Chassis: vm
      Machine ID: 732a1bead3d64bf891f9478869b4c744
         Boot ID: 3c74a969b28c46ca972d0edfd0a8e0b0
  Virtualization: kvm
Operating System: Ubuntu 22.04.3 LTS
          Kernel: Linux 5.15.0-89-generic
    Architecture: x86-64
 Hardware Vendor: QEMU
  Hardware Model: Standard PC _Q35 + ICH9, 2009_
```

## 仮想マシンの領域拡張（参考）

### ホスト

```shell
root@bravog:/home/bravog$ qemu-img info /var/lib/libvirt/images/k8s-master.qcow2
image: /var/lib/libvirt/images/k8s-master.qcow2
file format: qcow2
virtual size: 7 GiB (7516192768 bytes)
disk size: 2.57 GiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: true
    refcount bits: 16
    corrupt: false
    extended l2: false
root@bravog:/home/bravog$ qemu-img resize -f qcow2 /var/lib/libvirt/images/k8s-master.qcow2 +3G
Image resized.
root@bravog:/home/bravog$ qemu-img info /var/lib/libvirt/images/k8s-master.qcow2
image: /var/lib/libvirt/images/k8s-master.qcow2
file format: qcow2
virtual size: 10 GiB (10737418240 bytes)
disk size: 2.57 GiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: true
    refcount bits: 16
    corrupt: false
    extended l2: false
```

### ゲスト

```bash
root@k8s-master:/home/bravog$ sudo fdisk -l
Disk /dev/loop0: 63.45 MiB, 66531328 bytes, 129944 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop1: 111.95 MiB, 117387264 bytes, 229272 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop2: 53.26 MiB, 55844864 bytes, 109072 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop3: 40.86 MiB, 42840064 bytes, 83672 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/loop4: 63.46 MiB, 66547712 bytes, 129976 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
GPT PMBR size mismatch (14680063 != 20971519) will be corrected by write.
The backup GPT table is not on the end of the device.


Disk /dev/vda: 10 GiB, 10737418240 bytes, 20971520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 8B946D30-DFDD-4CC8-926E-B0F77CDC139C

Device       Start      End  Sectors  Size Type
/dev/vda1     2048     4095     2048    1M BIOS boot
/dev/vda2     4096  3674111  3670016  1.8G Linux filesystem
/dev/vda3  3674112 14678015 11003904  5.2G Linux filesystem


Disk /dev/mapper/ubuntu--vg-ubuntu--lv: 5.25 GiB, 5632950272 bytes, 11001856 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
root@k8s-master:/home/bravog$ parted /dev/vda
GNU Parted 3.4
Using /dev/vda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) p
Warning: Not all of the space available to /dev/vda appears to be used, you can fix the GPT to use all of the space (an extra
6291456 blocks) or continue with the current setting?
Fix/Ignore? fix
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 10.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  1881MB  1879MB  ext4
 3      1881MB  7515MB  5634MB

(parted) resizepart 3
End?  [7515MB]? 100%
(parted) p
Model: Virtio Block Device (virtblk)
Disk /dev/vda: 10.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  2097kB  1049kB                     bios_grub
 2      2097kB  1881MB  1879MB  ext4
 3      1881MB  10.7GB  8856MB

(parted) q
Information: You may need to update /etc/fstab.

root@k8s-master:/home/bravog$ lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
  Size of logical volume ubuntu-vg/ubuntu-lv changed from <5.25 GiB (1343 extents) to <8.25 GiB (2111 extents).
  Logical volume ubuntu-vg/ubuntu-lv successfully resized.
root@k8s-master:/home/bravog$ mount -l | grep /dev/mapper/ubuntu--vg-ubuntu--lv
/dev/mapper/ubuntu--vg-ubuntu--lv on / type ext4 (rw,relatime)
root@k8s-master:/home/bravog$ resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
resize2fs 1.46.5 (30-Dec-2021)
Filesystem at /dev/mapper/ubuntu--vg-ubuntu--lv is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 2
The filesystem on /dev/mapper/ubuntu--vg-ubuntu--lv is now 2161664 (4k) blocks long.

root@k8s-master:/home/bravog$ df -h /
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv  8.1G  4.5G  3.2G  59% /
```

## 仮想マシン個別設定

### マスタノードの設定

```shell
root@k8s-master:~$ kubeadm init
[init] Using Kubernetes version: v1.28.4
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
W1127 12:37:49.844012    8031 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.8" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.122.181]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.122.181 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.122.181 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 6.014449 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: r05zoo.e4xr2sx3rs722hwz
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.122.181:6443 --token r05zoo.e4xr2sx3rs722hwz \
	--discovery-token-ca-cert-hash sha256:aa0ccea1084a2c439071a19dec5cd5747055c34964c0f8b187fd321978152ae5
```

```shell
bravog@k8s-master:~$ cd
bravog@k8s-master:~$ mkdir -p $HOME/.kube
bravog@k8s-master:~$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[sudo] password for bravog:
bravog@k8s-master:~$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

```shell
root@k8s-master:/home/bravog$ wget https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml
--2023-11-27 14:19:44--  https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.108.133, 185.199.109.133, 185.199.110.133, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.108.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 252345 (246K) [text/plain]
Saving to: ‘calico.yaml’

calico.yaml                      100%[=========================================================>] 246.43K  --.-KB/s    in 0.03s

2023-11-27 14:19:44 (9.59 MB/s) - ‘calico.yaml’ saved [252345/252345]

root@k8s-master:/home/bravog$ ls -l
total 252
-rw-r--r-- 1 root root 252345 Nov 27 14:19 calico.yaml
-rw-r--r-- 1 root root   1778 Nov 24 06:42 containerd.service
root@k8s-master:/home/bravog$ export KUBECONFIG=/etc/kubernetes/admin.conf
root@k8s-master:/home/bravog$ kubectl apply -f calico.yaml
poddisruptionbudget.policy/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
serviceaccount/calico-node created
serviceaccount/calico-cni-plugin created
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpfilters.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/caliconodestatuses.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipreservations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/kubecontrollersconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrole.rbac.authorization.k8s.io/calico-cni-plugin created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-cni-plugin created
daemonset.apps/calico-node created
deployment.apps/calico-kube-controllers created
root@k8s-master:/home/bravog$ kubectl get nodes
NAME         STATUS   ROLES           AGE    VERSION
k8s-master   Ready    control-plane   103m   v1.28.2
root@k8s-master:/home/bravog$ kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS      AGE
kube-system   calico-kube-controllers-57758d645c-jswcr   1/1     Running   0             51s
kube-system   calico-node-xxqqm                          1/1     Running   0             51s
kube-system   coredns-5dd5756b68-6vxwp                   1/1     Running   0             103m
kube-system   coredns-5dd5756b68-llrvw                   1/1     Running   0             103m
kube-system   etcd-k8s-master                            1/1     Running   2 (24m ago)   103m
kube-system   kube-apiserver-k8s-master                  1/1     Running   2 (24m ago)   103m
kube-system   kube-controller-manager-k8s-master         1/1     Running   2 (24m ago)   103m
kube-system   kube-proxy-wxnrw                           1/1     Running   1 (24m ago)   103m
kube-system   kube-scheduler-k8s-master                  1/1     Running   2 (24m ago)   103m
```

### ワーカノード 1 の設定

```shell
root@k8s-worker1:/home/bravog$ kubeadm join 192.168.122.181:6443 --token r05zoo.e4xr2sx3rs722hwz        --discovery-token-ca-cert-hash sha256:aa0ccea1084a2c439071a19dec5cd5747055c34964c0f8b187fd321978152ae5
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W1127 14:45:12.793736    2453 common.go:192] WARNING: could not obtain a bind address for the API Server: no default routes found in "/proc/net/route" or "/proc/net/ipv6_route"; using: 0.0.0.0
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

```

### ワーカノード 2 の設定

```shell
root@k8s-worker2:/home/bravog$ kubeadm join 192.168.122.181:6443 --token r05zoo.e4xr2sx3rs722hwz        --discovery-token-ca-cert-hash sha256:aa0ccea1084a2c439071a19dec5cd5747055c34964c0f8b187fd321978152ae5
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W1127 14:48:19.950806    2494 common.go:192] WARNING: could not obtain a bind address for the API Server: no default routes found in "/proc/net/route" or "/proc/net/ipv6_route"; using: 0.0.0.0
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

```

## その他

### コントロールプレーンに NFS サーバをインストール

```shell
$ apt -y install nfs-kernel-server
$ vi /etc/idmapd.conf
===== ドメインを設定
$ Domain = localdomain
Domain = virt.net
=====
$ vi /etc/exports
===== 最終行にマウントポイントを定義
/home/nfsshare 192.168.122.0/24(rw,no_root_squash)
=====
$ mkdir /home/nfsshare
$ systemctl restart nfs-server
```

### ワーカーノードに NFS クライアントをインストール

```shell
$ apt -y install nfs-common
$ vi /etc/idmapd.conf
===== ドメインを設定
$ Domain = localdomain
Domain = virt.net
=====
$ mount -t nfs k8s-master.virt.net:/home/nfsshare /mnt
$ df -hT /mnt
Filesystem                         Type  Size  Used Avail Use% Mounted on
k8s-master.virt.net:/home/nfsshare nfs4  8.1G  6.3G  1.4G  83% /mnt
$ vi /etc/fstab
===== 以下追加
k8s-master.virt.net:/home/nfsshare /mnt nfs defaults 0 0
```
