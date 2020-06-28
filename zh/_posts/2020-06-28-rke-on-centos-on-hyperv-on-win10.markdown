---
layout: single
title:  "在 Centos 上安裝 RKE"
date:   2020-06-28 01:00:00 +0800
categories: study
author: "Conan Chang"
lang: zh
toc: true
toc_label: "目錄"
---

背景環境
===
Centos 7(2003) 是安裝在 Hyper-V 上的, 沒有把 `Windows 安全開機`關掉的話會出現 `The unsigned images hash is not allowed` , name server (`/etc/resolv.conf`) 沒設好都害我忙了一下。

iso 用的是 <http://isoredirect.centos.org/centos/7/isos/x86_64/> 的 dvd 版，不安裝GUI

```bash
[root@localhost ~]# hostnamectl
Static hostname: localhost.localdomain
     Icon name: computer-vm
          Chassis: vm
     Machine ID: 22a6fecfb61b444a816d82000fa0fe8d
          Boot ID: 8dd351014d3348c7996aca42ebdb920d
Virtualization: microsoft
Operating System: CentOS Linux 8 (Core)
     CPE OS Name: cpe:/o:centos:centos:8
          Kernel: Linux 4.18.0-193.el8.x86_64
     Architecture: x86-64
```

下載 RKE 
===
我用的是`rke_linux-amd64`，其他環境去release裡找 <https://github.com/rancher/rke/releases>

```bash
curl -L https://github.com/rancher/rke/releases/download/v1.1.3/rke_linux-amd64 -o rke_linux-amd64
mv rke_linux-amd64 rke
```

讓 RKE 可執行

```bash
chmod +x rke
# 搬到usr/bin裡
mv rke /usr/bin/rke
```

然後確定能用

```bash
[docker_user@localhost rke]$ rke --version
rke version v1.1.3
```

啟動 linux module
===
這邊用一個 shell script 來執行。

```bash
nano check_module.sh
```
把以下東西都貼進去

```bash
#!/bin/sh
for module in br_netfilter ip6_udp_tunnel ip_set ip_set_hash_ip ip_set_hash_net iptable_filter iptable_nat iptable_mangle iptable_raw nf_conntrack_netlink nf_conntrack nf_conntrack_ipv4   nf_defrag_ipv4 nf_nat nf_nat_ipv4 nf_nat_masquerade_ipv4 nfnetlink udp_tunnel veth vxlan xt_addrtype xt_conntrack xt_comment xt_mark xt_multiport xt_nat xt_recent xt_set  xt_statistic;
     do
     if ! lsmod | grep -q $module; then
          modprobe $module;
     fi;
done

# 有兩個 module 不在 lsmod 裡 -> https://forums.rancher.com/t/rke-kernel-requirements-centos/12074/6
for module in x_tables xt_tcpudp;
     do
     if ! cat /lib/modules/$(uname -r)/modules.builtin | grep -q $module; then
          echo "$module $module is not present";
     fi;
done
```

最後執行它

```bash
sh check_module.sh
```

修改 sysctl 參數
===
打開 /etc/sysctl.conf，然後加上

```bash
net.bridge.bridge-nf-call-iptables=1
```

成品(/etc/sysctl.conf)大概像這樣

```bash
# sysctl settings are defined through files in
# /usr/lib/sysctl.d/, /run/sysctl.d/, and /etc/sysctl.d/.
#
# Vendors settings live in /usr/lib/sysctl.d/.
# To override a whole file, create a new file with the same in
# /etc/sysctl.d/ and put new settings there. To override
# only specific settings, add a file with a lexically later
# name in /etc/sysctl.d/ and put new settings there.
#
# For more information, see sysctl.conf(5) and sysctl.d(5).
net.bridge.bridge-nf-call-iptables=1
```

安裝 docker
===
這裡使用的版本是18.09.2，安裝前可以先看看 Rancher有沒有給出新的版本 <https://rancher.com/docs/rke/latest/en/os/#installing-docker>。

```bash
# 用 centos 8 執行 rancher 的 script 會出錯, 這也是一開始使用 centos 7 的原因
curl https://releases.rancher.com/install-docker/18.09.2.sh | sh
```
     
確定版本

```bash
rpm -q docker-ce
```
     
應該會回傳 `docker-ce-18.09.2-3.el7.x86_64`


然後順便把 docker 變開機啟動吧

都要安裝 RKE (kubernetes)了嘛，docker 開機啟動也是必需的，
先檢查 docker 狀態 `systemctl status docker`，
會看到類似這個

```bash
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabled)
     Active: active (running) since Sun 2020-06-28 17:40:49 CST; 44min ago
     Docs: https://docs.docker.com
Main PID: 9826 (dockerd)
     Tasks: 29
     Memory: 71.2M
     CGroup: /system.slice/docker.service
          ├─9826 /usr/bin/dockerd -H fd://
          └─9839 containerd --config /var/run/docker/containerd/containerd.toml --log-level info
```

有看到 `/usr/lib/systemd/system/docker.service` 後面是 `disabled`，
要把它變成開機啟動

```bash
systemctl enable docker
```


建立一個 non-root 的 docker 使用者
===

```bash
# 建使用者，叫 docker_user
adduser docker_user
# 設定密碼
passwd docker_user
# 建立 user group，叫 docker
groupadd docker
# 把 docker_user 加進 docker 裡
usermod -aG docker docker_user
```

然後切換成 docker_user 來測試看看能不能使用 docker

```bash
su docker_user
docker run hello-world
```

* 話說之前很常打開一些機器發現如果要用 docker 指令都要切到 root。原來是少了一個步驟


準備 RKE 的 config 檔: cluster.yml
===
利用 RKE 產生設定檔

```bash
# 因為操作的使用者是 `docker_user`，所以我也順便切換了一下使用者
su docker_user
rke config --name cluster.yml
```

然後就會有一堆資料讓你填，用來設定。
有幾個重點

 - `Cluster Level SSH Private Key Path`: 等下會建個 ssh key，就是放在這個路徑底下(這是預設路徑)
 - `SSH Address of host`: 機器 IP
 - `SSH User of host`: 這個放 docker_user，也就是剛專門建出來操作 docker 的

完整設定:

```bash
[docker_user@localhost rke]$ rke config --name cluster.yml
[+] Cluster Level SSH Private Key Path [~/.ssh/id_rsa]:
[+] Number of Hosts [1]:
[+] SSH Address of host (1) [none]: 172.28.161.72
[+] SSH Port of host (1) [22]:
[+] SSH Private Key Path of host (172.28.161.72) [none]: ~/.ssh/id_rsa
[+] SSH User of host (172.28.161.72) [ubuntu]: docker_user
[+] Is host (172.28.161.72) a Control Plane host (y/n)? [y]: y
[+] Is host (172.28.161.72) a Worker host (y/n)? [n]: y
[+] Is host (172.28.161.72) an etcd host (y/n)? [n]: y
[+] Override Hostname of host (172.28.161.72) [none]: master
[+] Internal IP of host (172.28.161.72) [none]:
[+] Docker socket path on host (172.28.161.72) [/var/run/docker.sock]:
[+] Network Plugin Type (flannel, calico, weave, canal) [canal]: calico
[+] Authentication Strategy [x509]:
[+] Authorization Mode (rbac, none) [rbac]:
[+] Kubernetes Docker image [rancher/hyperkube:v1.18.3-rancher2]:
[+] Cluster domain [cluster.local]:
[+] Service Cluster IP Range [10.43.0.0/16]:
[+] Enable PodSecurityPolicy [n]:
[+] Cluster Network CIDR [10.42.0.0/16]:
[+] Cluster DNS Service IP [10.43.0.10]:
[+] Add addon manifest URLs or YAML files [no]:
```

產生 ssh 金鑰
===
指令是

```bash
ssh-keygen
```

然後全部用預定，會得到一個 RSA 2048 的金鑰配對

```bash
[docker_user@localhost rke]$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/docker_user/.ssh/id_rsa):
Created directory '/home/docker_user/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/docker_user/.ssh/id_rsa.
Your public key has been saved in /home/docker_user/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:FFCb/cNcNPv09ociZPyAgP49DTxA9bR/3bPWpP5OEQY docker_user@localhost.localdomain
The key's randomart image is:
+===[RSA 2048]===-+
|      o++ .  Eo  |
|     o   B . ..o |
|    . o + +   oo.|
|   .   = o = ..++|
|    .   S = * .oB|
|     . . * o o =*|
|      . o o o oo=|
|         . . o...|
|              .oo|
+===-[SHA256]===--+
```

順便確定一下金鑰都有生出來

```bash
[docker_user@localhost rke]$ ls ~/.ssh
id_rsa  id_rsa.pub
```

安裝金鑰

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub docker_user@172.28.161.72
```

終於可以安裝 RKE 啦
===

```bash
[docker_user@localhost rke]$ rke up
INFO[0000] Running RKE version: v1.1.3
INFO[0000] Initiating Kubernetes cluster
INFO[0000] [certificates] GenerateServingCertificate is disabled, checking if there are unused kubelet certificates
...
INFO[0175] [addons] Executing deploy job rke-ingress-controller
INFO[0180] [ingress] ingress controller nginx deployed successfully
INFO[0180] [addons] Setting up user addons
INFO[0180] [addons] no user addons defined
INFO[0180] Finished building Kubernetes cluster successfully
```

看這寫著安裝成功的精美訊息，好讓人欣慰。
同時也得到了 `kube_config_cluster.yml` 和 `cluster.rkestate`

安裝 kubectl
===

kubectl 的部分就不像 docker 那樣, rancher 它沒有提供的客製化安裝腳本。
所以這裡就跟一般安裝 kubectl 的方法一樣。
不過要先回到 root 使用者，把 `docker_user` 加到 wheel group 裡

```bash
usermod -aG wheel docker_user
```

開始安裝

```bash
# 下載
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
# 改成可執行
chmod +x ./kubectl
# 放進 /usr/bin 也就是這裡需要 sudoer 權限
sudo mv ./kubectl /usr/local/bin/kubectl
# 檢查能不能用
kubectl version --client
```

最後我拿到的版本號訊息是這樣

```bash
[docker_user@localhost rke]$ kubectl version --client
Client Version: version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.5", GitCommit:"e6503f8d8f769ace2f338794c914a96fc335df0f", GitTreeState:"clean", BuildDate:"2020-06-26T03:47:41Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}
```

設定 kubectl
===

直接把 kube_config_cluster.yml 丟到 kubectl 預設的設定檔路徑

```bash
mv kube_config_cluster.yml ~/.kube/config
```

打個指令看看能不能用

```bash
[docker_user@localhost rke]$ kubectl get node
NAME     STATUS   ROLES                      AGE   VERSION
master   Ready    controlplane,etcd,worker   54m   v1.18.3
```

最後來稍微看一下預設一開始 RKE 有哪些資源被啟動吧
===

```bash
[docker_user@localhost rke]$ kubectl get all -A
NAMESPACE       NAME                                           READY   STATUS      RESTARTS   AGE
ingress-nginx   pod/default-http-backend-598b7d7dbd-zddlp      1/1     Running     0          56m
ingress-nginx   pod/nginx-ingress-controller-szlzm             1/1     Running     0          56m
kube-system     pod/calico-kube-controllers-7fbc6b86cc-czsqs   1/1     Running     0          56m
kube-system     pod/calico-node-nkwbz                          1/1     Running     0          56m
kube-system     pod/coredns-849545576b-jgcxb                   1/1     Running     0          56m
kube-system     pod/coredns-autoscaler-5dcd676cbd-x2bmp        1/1     Running     0          56m
kube-system     pod/metrics-server-697746ff48-4fw5n            1/1     Running     0          56m
kube-system     pod/rke-coredns-addon-deploy-job-v99wb         0/1     Completed   0          56m
kube-system     pod/rke-ingress-controller-deploy-job-xcwxn    0/1     Completed   0          56m
kube-system     pod/rke-metrics-addon-deploy-job-9jjvd         0/1     Completed   0          56m
kube-system     pod/rke-network-plugin-deploy-job-qgrhk        0/1     Completed   0          56m

NAMESPACE       NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
default         service/kubernetes             ClusterIP   10.43.0.1       <none>        443/TCP                  56m
ingress-nginx   service/default-http-backend   ClusterIP   10.43.204.251   <none>        80/TCP                   56m
kube-system     service/kube-dns               ClusterIP   10.43.0.10      <none>        53/UDP,53/TCP,9153/TCP   56m
kube-system     service/metrics-server         ClusterIP   10.43.186.178   <none>        443/TCP                  56m

NAMESPACE       NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
ingress-nginx   daemonset.apps/nginx-ingress-controller   1         1         1       1            1           <none>                   56m
kube-system     daemonset.apps/calico-node                1         1         1       1            1           kubernetes.io/os=linux   56m

NAMESPACE       NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
ingress-nginx   deployment.apps/default-http-backend      1/1     1            1           56m
kube-system     deployment.apps/calico-kube-controllers   1/1     1            1           56m
kube-system     deployment.apps/coredns                   1/1     1            1           56m
kube-system     deployment.apps/coredns-autoscaler        1/1     1            1           56m
kube-system     deployment.apps/metrics-server            1/1     1            1           56m

NAMESPACE       NAME                                                 DESIRED   CURRENT   READY   AGE
ingress-nginx   replicaset.apps/default-http-backend-598b7d7dbd      1         1         1       56m
kube-system     replicaset.apps/calico-kube-controllers-7fbc6b86cc   1         1         1       56m
kube-system     replicaset.apps/coredns-849545576b                   1         1         1       56m
kube-system     replicaset.apps/coredns-autoscaler-5dcd676cbd        1         1         1       56m
kube-system     replicaset.apps/metrics-server-697746ff48            1         1         1       56m

NAMESPACE     NAME                                          COMPLETIONS   DURATION   AGE
kube-system   job.batch/rke-coredns-addon-deploy-job        1/1           1s         56m
kube-system   job.batch/rke-ingress-controller-deploy-job   1/1           2s         56m
kube-system   job.batch/rke-metrics-addon-deploy-job        1/1           1s         56m
kube-system   job.batch/rke-network-plugin-deploy-job       1/1           13s        56m
[docker_user@localhost rke]$ kubectl get namespace
NAME              STATUS   AGE
default           Active   57m
ingress-nginx     Active   56m
kube-node-lease   Active   57m
kube-public       Active   57m
kube-system       Active   57m


[docker_user@localhost rke]$ kubectl get namespace
NAME              STATUS   AGE
default           Active   57m
ingress-nginx     Active   56m
kube-node-lease   Active   57m
kube-public       Active   57m
kube-system       Active   57m
```

心情舒發
===
眾所周知，Kubernetes的安裝的很複雜。普羅大眾都會想要有個簡單的試驗環境做一些測試之類的事情。
而 `minikube` 這種很好上手，但是只適合很簡單的學習 Kubernetes。要拿來安裝各種工具會遇到問題。
之前也有用過 `micork8s`，不過也是在 docker 路徑、kubelet路徑各種踩雷。
所以也多嘗試一種，只是安裝過程還真的蠻多步驟的，沒有很友善 XD。

不過以後可能會有些需要在 k8s cluster 上的測試/學習，希望用這套能順利。