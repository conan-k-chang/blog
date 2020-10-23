---
layout: single
title:  "openfaas 寫log (python-flask)"
date:   2020-10-23 01:00:00 +0800
categories: study
author: "Conan Chang"
lang: zh
toc: true
toc_label: "目錄"
---

對，如果你是一開始用 python 寫 openfaas function 的話，你會發現你的 print() 沒有作用
所有程式碼都在這裡：[https://github.com/conan-k-chang/openfaas-example/tree/main/write_log](https://github.com/conan-k-chang/openfaas-example/tree/main/write_log)

### 實際做個實驗來看看是怎麼一回事

```bash
# 下載 template
faas template store pull python3-flask
# 新增一個 function
faas new --lang python3-flask write-log
```

然後我們改一下我們的 function ，加個 print()

```bash
# write-log/handler.py
def handle(req):
    """handle a request to the function
    Args:
        req (str): request body
    """
    print('can you see me?')  # add message here
    return req
```

部署

```bash
# 獲取 openfaas gateway 的 IP 和 port
GATEWAY_IP=$(kubectl get service/gateway -n openfaas -ojsonpath='{.spec.clusterIP}')
GATEWAY_PORT=$(kubectl get service/gateway -n openfaas -ojsonpath='{.spec.ports[0].port}')
# 部署
faas up -f write-log.yml --skip-push --gateway=http://${GATEWAY_IP}:${GATEWAY_PORT}
```

測試一下。因為這個 template 的 function 是會把 request data 回傳給 client，所以可以看到 send back to me 被回傳了

```bash
[root@localhost write_log]# curl http://10.43.210.200:8080/function/write-log -d 'send back to me'
send back to me[root@localhost write_log]#
```

然後重點來了，檢查一下它的 log 

```bash
[root@localhost write_log]# kubectl logs -n openfaas-fn deploy/write-log
Forking - python [index.py]
2020/10/23 12:50:58 Started logging stderr from function.
2020/10/23 12:50:58 Started logging stdout from function.
2020/10/23 12:50:58 OperationalMode: http
2020/10/23 12:50:58 Timeouts: read: 10s, write: 10s hard: 10s.
2020/10/23 12:50:58 Listening on port: 8080
2020/10/23 12:50:58 Writing lock-file to: /tmp/.lock
2020/10/23 12:50:58 Metrics listening on port: 8081
2020/10/23 12:56:39 POST / - 200 OK - ContentLength: 4
```

can you see me? 沒有被 print 出來，這就要了解它的架構了。它事實上是會啟動 of-watchdog ，而 of-watch 會啟動 fprocess(我會以它的 repo 名稱 of-watchdog 來稱呼) ，而 fprocess 這個參數就是我們的 python flask

### 想要真實了解的話 TL;DR

首先看一下它的 Dockerfile

```bash
[root@localhost write_log]# cat template/python3-flask/Dockerfile
FROM openfaas/of-watchdog:0.7.7 as watchdog
FROM python:3.7-alpine
...
CMD ["fwatchdog"]
```

可以看到 Docker 事實上是啟動 fwatchdog 這個程式 (然後 fwatchdog 會啟動 fprocess)

看一下 container 裡面的程式

```bash
[root@localhost write_log]# kubectl -n openfaas-fn exec deploy/write-log -- pstree
fwatchdog---python
```

就是 fwatchdog 把 python 啟動

至於我們 python 的 log 是怎麼被吃掉的呢，其實從 of-watchdog 的 log 看得出來

```bash
[root@localhost write_log]# kubectl logs -n openfaas-fn deploy/write-log
Forking - python [index.py]
2020/10/23 12:50:58 Started logging stderr from function.
2020/10/23 12:50:58 Started logging stdout from function.
```

fwatchdog 只會監聽 stderr 和 stdout。雖然這樣就能大概就知原理了，不過我是直接看他的 source code 來確認的

### 解決方法

使用 logging 來把 output 打到 stdout，那這篇也不是在講 python 的 logging。所以搞懂原理你也可以用自己希望的方法來寫 log 。這邊我提供我的例子

```bash
# write-log/handler.py
import logging
import sys

logger = logging.getLogger()
logger.setLevel(logging.INFO)
handler = logging.StreamHandler(sys.stdout) # logging to stdout
logger.addHandler(handler)

def handle(req):
    """handle a request to the function
    Args:
        req (str): request body
    """
    logger.info('can you see me?')
    return req
```

接下來就是重新部署，發 request ，檢查 log 了。步驟都做過我就直接一口氣寫下來吧

```bash
GATEWAY_IP=$(kubectl get service/gateway -n openfaas -ojsonpath='{.spec.clusterIP}')
GATEWAY_PORT=$(kubectl get service/gateway -n openfaas -ojsonpath='{.spec.ports[0].port}')
# 部署
faas up -f write-log.yml --skip-push --gateway=http://${GATEWAY_IP}:${GATEWAY_PORT}
# 發 request
curl http://10.43.210.200:8080/function/write-log -d 'send back to me'
```

```bash
[root@localhokubectl logs -n openfaas-fn deploy/write-log
Forking - python [index.py]
2020/10/23 13:44:34 Started logging stderr from function.
2020/10/23 13:44:34 Started logging stdout from function.
2020/10/23 13:44:34 OperationalMode: http
2020/10/23 13:44:34 Timeouts: read: 10s, write: 10s hard: 10s.
2020/10/23 13:44:34 Listening on port: 8080
2020/10/23 13:44:34 Writing lock-file to: /tmp/.lock
2020/10/23 13:44:34 Metrics listening on port: 8081
2020/10/23 13:47:38 stdout: Serving on http://0.0.0.0:5000
2020/10/23 13:47:38 stdout: can you see me?
2020/10/23 13:47:38 POST / - 200 OK - ContentLength: 15
```

### 小備註

其實每次 faas up 都會讓 pod 重新啟動，也就是新的 code 就已經上去了。不過你也知道，kubernetes 的博大精深哪是你我能夠完全掌控的呢

如果 faas up 完覺得 Pod 沒有重新啟動的話可以這樣檢查

```bash
# 看一下 pod 是不是新鮮長出來的
kubectl get pod -n openfaas-fn -l faas_function=write-log
```

或者手動讓他重新啟動

```bash
# 把 pod 刪掉讓它長新的出來
kubectl delete pod -n openfaas-fn -l faas_function=write-log
```