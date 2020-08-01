---
layout: single
title:  "用 Helm template 安裝 OpenFaaS"
date:   2020-07-26 01:00:00 +0800
categories: study
author: "Conan Chang"
lang: zh
---

作者的OS
===
其實網路上已經有一大堆安裝的教學了，不過聽了幾篇發現講的都不太深入。
本身公司是有在用 OpenFaaS的，所以我也研究了不少，就打算寫一系列關於我對 OpenFaaS 的了解

簡介
===
OpenFaaS 就是一個在 Kubernetes 上建購像 AWS Lambda 一樣的 (Serverless) 平台。
不過老實說，我更傾向用 Function as a Service (FaaS) 來理解它。因為所謂的 Serverless 就是把 Server 這一層本來開發人員要處理的事情外包，讓開發人員可以只專心寫 Function。用 Serverless 這個字感覺把事情變得很複雜。
用 FaaS 來理解還能跟 IaaS, PaaS, SaaS 連上關係。 ~~不好意思扯遠了XD~~

功能
===
`OpenFaaS` 這個平台提供的事情就是讓你在開發的時候省下很多時間去做一些每個 Web Application 都需要做的事情。
例如：
 - Serverless 背後的 Server：你開發的功能一定是放在一個 Web server 上的，OpenFaaS 讓你可以只寫 Function ，然後就可以部署這個 Function 來使用了。
 - 提供 async function 的架構，讓你不用管 queue server 那些事情。反正部下去就能有 async function 能用了。
 - Metrics 的收集：openfaas 自帶 prometheus ，例如 Server 收到多少 Requests，回傳了多少個 500 之類的都不用開發者煩腦
 - K8S yaml 的撰寫：每個放在 K8S 上的 Application 大概都需要一個 Service, Deployment，OpenFaaS 可以讓開發者用 deploy function 的概念完成這個動作。

 p.s. 看起來 OpenFaaS 蠻方便的，但是蠻方便跟客製化當然存在一定的抵觸。像是 K8S 的 Deployment 想要來點特殊的設定就還是要再修改，或者直接放棄它的 Deploy 功能，自己寫自己要的 yaml 。

安裝
===
安裝方法其實有三種 [ref](https://docs.openfaas.com/deployment/kubernetes/#b-deploy-with-helm-for-production-most-configurable)
1. arkade: OpenFaaS 自己的工具，而我沒事不想又安裝個東西，所以不用這個
2. Helm: 用 Helm 的話可以有一些 Configuration 可以設，`所以本篇會使用 Helm 來安裝`
3. 寫好的 yaml: 其實就跟第二點類似，不過就是不能改設定

如果沒有 Helm 的話要先安裝
---
https://github.com/openfaas/faas-netes/blob/master/HELM.md#helm-3
```bash
curl -sSLf https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
安裝上大致上跟 OpenFaaS 的 README 一樣
---
https://github.com/openfaas/faas-netes/blob/master/chart/openfaas/README.md

不過它寫的比較完整，而我使用 Helm template 的方式來安裝。以下就是我從中挑出來的安裝步驟

首先來 clone 它
```bash
git clone https://github.com/openfaas/faas-netes.git
cd faas-netes
```

然後改設定 chart/openfaas/value.yaml，
先上結果
```
openfaasImagePullPolicy: "IfNotPresent"
gateway:
  readTimeout : "650s"
  writeTimeout : "650s"
  upstreamTimeout : "600s"
faasnet:
  imagePullPolicy : "IfNotPresent"

queueWorker:
  ackWait : "600s"
faasIdler:
  inactivityDuration: 15m
  reconcileInterval: 1m
  dryRun: false
```
 - openfaasImagePullPolicy： 這個預設是 Always ，會改成 IfNotPresent (~~我本身覺得沒甚麼事就應該用 IfNotPresent~~) 是用來面對之後 Cold start 的問題
 - gateway timeout： 因為 requests 都是透過 gateway 進來的，如果之後 function 要寫比較長時間 job 的話，就不能被 gateway 卡掉。
   - readTimeout 和 writeTimeout 可以參考 Go lang http server 
   - upstreamTimeout 則是願意等後面的 function 多久的 timeout。所以 upstreamTimeout 會比 writeTimeout 小
 - queueWorker.ackWait：前面提到 OpenFaaS 可以寫 async function ，那是透過一個 queue server 和 queue worker 搭配做出來的。queue worker 會從 queue server 拿 message 出來把 requets 丟給 function。ackWait 就是 queue worker 願意等待 function 的 timeout
 - faasIdler：它會把近期沒有被使用的 function 收掉。~~個人認為這是 OpenFaaS 的一個很重要的核心，不過其實做的不太好~~
   - 需要把 dryRun 設成 false 才會有作用
   - inactivityDuration 是說超過多久都沒被使用的 function 就可以被回收了
   - reconcileInterval 是多久檢查一次
   
都設定好之後就能產生 yaml 了
```bash
helm template \
  openfaas chart/openfaas/ \
  --namespace openfaas \
  --set basic_auth=true \
  --set functionNamespace=openfaas-fn > openfaas.yaml
```

先建 namespace
```bash
[root@localhost openfaas_core]# kubectl apply -f namespace.yaml
namespace/openfaas created
namespace/openfaas-fn created
```

產生密碼，這個之後是用來登入 gateway 用的

```bash
# 官網是說建個亂數啦，不過本文章就簡單一點把密碼寫成 password
# generate a random password	
# PASSWORD=$(head -c 12 /dev/urandom | shasum| cut -d' ' -f1)	
[root@localhost faas-netes]# PASSWORD=password
[root@localhost faas-netes]# kubectl -n openfaas create secret generic basic-auth --from-literal=basic-auth-user=admin --from-literal=basic-auth-password="$PASSWORD"
secret/basic-auth created
```

Apply OpenFaaS 所有部件，也就是剛才 Helm template 產生出來的 openfaas.yaml
```bash
[root@localhost faas-netes]# kubectl apply -f openfaas.yaml
serviceaccount/openfaas-controller created
serviceaccount/openfaas-prometheus created
configmap/alertmanager-config created
configmap/prometheus-config created
clusterrole.rbac.authorization.k8s.io/openfaas-prometheus configured
role.rbac.authorization.k8s.io/openfaas-controller created
rolebinding.rbac.authorization.k8s.io/openfaas-controller created
rolebinding.rbac.authorization.k8s.io/openfaas-prometheus created
rolebinding.rbac.authorization.k8s.io/openfaas-prometheus created
service/alertmanager created
service/basic-auth-plugin created
service/gateway-external created
service/gateway created
service/nats created
service/prometheus created
deployment.apps/alertmanager created
deployment.apps/basic-auth-plugin created
deployment.apps/faas-idler created
deployment.apps/gateway created
deployment.apps/nats created
deployment.apps/prometheus created
deployment.apps/queue-worker created
```
OpenFaaS 有一點是做的挺好的，就是它是 Kubernetes Native 的，所以安裝過程我都沒有遇過問題，後續的使用也很簡單。

裝完的話會有這些東西：
值得注意的是預設會有開一個 NodePort 31112，也是方便剛開始測試用的，可以用瀏覽器直接連進去看他的 Portal。這個 NodePort 在之後也應該要被刪除，改用 Ingress 來 expose
```bash
[root@localhost faas-netes]# kubectl get all -n openfaas
NAME                                     READY   STATUS    RESTARTS   AGE
pod/alertmanager-7498c4579b-bfqzk        1/1     Running   0          20d
pod/basic-auth-plugin-7d4956689b-k86xg   1/1     Running   0          20d
pod/faas-idler-67dfd9495d-r9bmd          1/1     Running   2          20d
pod/gateway-54f6c78d45-8vjxn             2/2     Running   0          20d
pod/nats-5cd4dff7c8-95vf6                1/1     Running   0          20d
pod/prometheus-c8f8b8fb4-696ts           1/1     Running   0          20d
pod/queue-worker-6cb888d49c-zrcbj        1/1     Running   0          20d

NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/alertmanager        ClusterIP   10.43.164.248   <none>        9093/TCP         20d
service/basic-auth-plugin   ClusterIP   10.43.84.196    <none>        8080/TCP         20d
service/gateway             ClusterIP   10.43.13.40     <none>        8080/TCP         20d
service/gateway-external    NodePort    10.43.87.54     <none>        8080:31112/TCP   20d
service/nats                ClusterIP   10.43.159.21    <none>        4222/TCP         20d
service/prometheus          ClusterIP   10.43.234.31    <none>        9090/TCP         20d

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/alertmanager        1/1     1            1           20d
deployment.apps/basic-auth-plugin   1/1     1            1           20d
deployment.apps/faas-idler          1/1     1            1           20d
deployment.apps/gateway             1/1     1            1           20d
deployment.apps/nats                1/1     1            1           20d
deployment.apps/prometheus          1/1     1            1           20d
deployment.apps/queue-worker        1/1     1            1           20d

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/alertmanager-7498c4579b        1         1         1       20d
replicaset.apps/basic-auth-plugin-7d4956689b   1         1         1       20d
replicaset.apps/faas-idler-67dfd9495d          1         1         1       20d
replicaset.apps/gateway-54f6c78d45             1         1         1       20d
replicaset.apps/nats-5cd4dff7c8                1         1         1       20d
replicaset.apps/prometheus-c8f8b8fb4           1         1         1       20d
replicaset.apps/queue-worker-6cb888d49c        1         1         1       20d
```


安裝FaaS CLI 
===
OpenFaaS 之所以方便的其中一個原因是在這個工具。後面的文章會用到它的功能

```bash
curl -sSL https://cli.openfaas.com | sudo sh
```

部署第一個 Function
===
先登入 faas gateway。 faas gateway 就是整個 OpenFaaS 的中控，要先登入才能進行操作。小提醒一下，以下操作都是在安裝 OpenFaaS 的主機上操作，所以可以直接用 ClusterIP 連接，如果你是在別台機器操作的話可以選擇用 NodePort 來連。
```bash
# 找一下faas gateway 在哪
gateway_ip=$(kubectl get service/gateway -n openfaas -ojsonpath='{.spec.clusterIP}')
# 登入，密碼就是之前寫的 password ，如果你使用別的密碼的話，這邊記得改成你的
echo 'password' | faas login -u admin --gateway=http://$gateway_ip:8080 --password-stdin
```

挑個展示用的 function 來部部看
```bash
[root@localhost ~]# faas store ls
FUNCTION                                 DESCRIPTION
NodeInfo                                 Get info about the machine that you'r...
Figlet                                   Generate ASCII logos with the figlet CLI
SSL/TLS cert info                        Returns SSL/TLS certificate informati...
Colorization                             Turn black and white photos to color ...
Inception                                This is a forked version of the work ...
Have I Been Pwned                        The Have I Been Pwned function lets y...
Face Detection with Pigo                 Detect faces in images using the Pigo...
curl                                     Use curl for network diagnostics, pas...
SentimentAnalysis                        Python function provides a rating on ...
Tesseract OCR                            This function brings OCR - Optical Ch...
Dockerhub Stats                          Golang function gives the count of re...
QR Code Generator - Go                   QR Code generator using Go
Nmap Security Scanner                    Tool for network discovery and securi...
ASCII Cows                               Generate cartoons of ASCII cows
YouTube Video Downloader                 Download YouTube videos as a function
OpenFaaS Text-to-Speech                  Generate an MP3 of text using Google'...
nslookup                                 Uses nslookup to return any IP addres...
Docker Image Manifest Query              Query an image on the Docker Hub for ...
face-detect with OpenCV                  Detect faces in images. Send a URL as...
Face blur by Endre Simo                  Blur out faces detected in JPEGs. Inv...
Left-Pad                                 left-pad on OpenFaaS
normalisecolor                           Automatically fix white-balance in ph...
mememachine                              Turn any image into a meme.
Business Strategy Generator              Generates a Business Strategy (using ...
Line Drawing Generator from a photograph Automatically generates a sketch like...
Image EXIF Reader                        Reads EXIF information from an URL or...
Open NSFW Model                          Score images for NSFW (nudity) content.
Identicon Generator                      Create an identicon from a provided s...
hey                                      HTTP load generator, ApacheBench (ab)...
```
NodeInfo 排第一，事情又簡單，就拿它來部署好了。這邊要注意就是 gateway 的地址要放好，不然預設是 127.0.0.1 是打不到的
```bash
[root@localhost deploy_store]# faas store deploy 'NodeInfo' --gateway=http://$gateway_ip:8080
WARNING! Communication is not secure, please consider using HTTPS. Letsencrypt.org offers free SSL/TLS certificates.

Deployed. 202 Accepted.
URL: http://10.43.13.40:8080/function/nodeinfo
```
要出現 202 才是部署成功喔

最後來看看結果
```bash
[root@localhost deploy_store]# curl http://10.43.13.40:8080/function/nodeinfo
Hostname: nodeinfo-6cf67cd8-bvcg5

Arch: x64
CPUs: 4
Total mem: 13189MB
Platform: linux
Uptime: 1437007
```
這樣就大功告成了。

之後就是要寫自己的 function 。