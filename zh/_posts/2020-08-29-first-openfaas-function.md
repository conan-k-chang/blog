---
layout: single
title:  "第一個簡單的 FaaS-function"
date:   2020-08-29 01:00:00 +0800
categories: study
author: "Conan Chang"
lang: zh
toc: true
toc_label: "目錄"
---

## 很久很久以前 (可以直接跳過這段)

OpenFaaS 的玩法是真的寫一個 Command 就結束了，原理是 FaaS-function 裡頭的主程式是一個叫 of-watchdog 的東西在運行，它負責接 http request，然後呼叫開發人員寫的 Command。這個 Command 所產生的 stdout 就會是被 of-watchdog 當作 http response body 回傳。不過這都是以前的東西的，我這系列都不會有這種模式的 FaaS-function。一開始(也很久以前了XD)不知道這個事情讓我有點搞混到底它是怎麼運作的。

## FaaS-Function 的原理

安裝篇有提到 Openfaas 其中一個功能是讓開發人員可以專心寫 Function (Serverless)，那麼 Server 到底哪來呢？就是來自 Openfaas 提供的 Template，這個 Template 裡頭已經寫好了 Web server，包含 requests 進來之後怎麼進入到開發人員寫的 Function 裡。

OpenFaaS 的 Template 有各種語言版本：[https://github.com/openfaas/of-watchdog#1-http-modehttp](https://github.com/openfaas/of-watchdog#1-http-modehttp)。

本身我是寫 Python 的，所以本篇會用python-flask-template：[https://github.com/openfaas-incubator/python-flask-template](https://github.com/openfaas-incubator/python-flask-template)

## 開發人員所負責的部分 — 理想中的版本

如果沒甚麼事當然就是把 Template 拿下來，修改裡面核心 function ，最後打包成 Docker Image 就可以了。

## 開發人員所負責的部分 — 現實

現實就是你還是要了解整個 Docker container 在幹嘛

OpenFaaS 的這個 Template 裡面會有一個叫 of-watchdog 的主程序，它本身會啟一個用 Go 寫的 http server，扮演 Reverse proxy，它負責接 Request ，再把 Request 傳給 Template 裡頭已經寫好的 Web server(本例子是 Flask )，Web server 再呼叫核心 function 。

來到這邊大概就會想到了：

- of-watchdog 會不會有甚麼 timeout 的設定呢
- Flask 前面的 WSGI 呢 (例如 gunicorn，uWSGI)

## 實作一下 — 理想

下載 Template，這邊我們只會用到 python3-flask ，所以其實其他都可以刪掉

```console
[root@localhost openfaas_function]# faas template pull https://github.com/openfaas-incubator/python-flask-template
Fetch templates from repository: https://github.com/openfaas-incubator/python-flask-template at master
2020/08/16 01:25:12 Attempting to expand templates from https://github.com/openfaas-incubator/python-flask-template
2020/08/16 01:25:14 Fetched 7 template(s) : [python27-flask python3-flask python3-flask-armhf python3-flask-debian python3-http python3-http-armhf python3-http-debian] from https://github.com/openfaas-incubator/python-flask-template

[root@localhost openfaas_function]# ls template/
python27-flask  python3-flask  python3-flask-armhf  python3-flask-debian  python3-http  python3-http-armhf  python3-http-debian
```

這邊使用的是一個回應現在時間的例子

```console
[root@localhost openfaas_function]# faas new --lang python3-flask time_now
Function name can only contain a-z, 0-9 and dashes
[root@localhost openfaas_function]# faas new --lang python3-flask time-now
Folder: time-now created.
  ___                   _____           ____
 / _ \ _ __   ___ _ __ |  ___|_ _  __ _/ ___|
| | | | '_ \ / _ \ '_ \| |_ / _` |/ _` \___ \
| |_| | |_) |  __/ | | |  _| (_| | (_| |___) |
 \___/| .__/ \___|_| |_|_|  \__,_|\__,_|____/
      |_|

Function created in folder: time-now
Stack file written: time-now.yml

[root@localhost openfaas_function]# ls
template  time-now  time-now.yml
```

### 來改一下程式碼

把 time-now/handler.py 改成這樣

```python
# time-now/handler.py
import requests

def handle(req):
    """handle a request to the function
    Args:
        req (str): request body
    """
    time_response = requests.get('http://worldtimeapi.org/api/timezone/Asia/Taipei')
    return time_response.json()
```

而 time-now/requirements.txt 裡頭加個 requests

```bash
# time-now/requirements.txt
requests
```

改一下 time-now.yml 檔的 gateway ip，可以透過  `kubectl get service/gateway -n openfaas -ojsonpath='{.spec.clusterIP}' 取得`，我的 gateway ip 是 10.43.210.200

```yaml
# time-now.yml
version: 1.0
provider:
  name: openfaas
  gateway: http://10.43.210.200:8080
functions:
  time-now:
    lang: python3-flask
    handler: ./time-now
    image: time-now:latest
```

準備好可以部署了，這邊加了一個 --skip-push 是因為它預設會推到 docker-hub。因為不想又額外講這個，就直接不 push 了。(如果是有這個需求的話可以將 image 加上你的 docker registry 就可以了，例如 image: your-company.com/time-now:latest)

```console
[root@localhost openfaas_function]# faas up --skip-push -f time-now.yml
...
會有 build image 的 Log (是直接 Docker API 來 Build 的，不是 docker command，所以我們影響不了他 Build 的行為)
...
Deploying: time-now.
WARNING! Communication is not secure, please consider using HTTPS. Letsencrypt.org offers free SSL/TLS certificates.

Deployed. 202 Accepted.
URL: http://10.43.210.200:8080/function/time-now.openfaas-fn
```

最後看一下結果，這邊可以不打 namespace

```console
[root@localhost openfaas_function]# curl http://10.43.210.200:8080/function/time-now
{"abbreviation":"CST","client_ip":"*.*.*.*","datetime":"2020-08-16T03:07:49.712847+08:00","day_of_week":0,"day_of_year":229,"dst":false,"dst_from":null,"dst_offset":0,"dst_until":null,"raw_offset":28800,"timezone":"Asia/Taipei","unixtime":1597518469,"utc_datetime":"2020-08-15T19:07:49.712847+00:00","utc_offset":"+08:00","week_number":33}
```



## 實作一下 — 現實

功能一樣是顯示現在 Asia/Taipei 的時間，不過我希望在 Flask 前面多加一個 gunicorn。

### 複製一份 template 出來

因為需要修改 faas-function 的 template ，而剛在用的是 template/python3-flask，所以我們複製一份出來叫 python3-flask-gunicorn

```bash
cp -r template/python3-flask template/python3-flask-gunicorn
```

修改 template/python3-flask-gunicorn/requirements.txt，把 gunicorn 加進去

```bash
# template/python3-flask-gunicorn/requirements.txt
flask
waitress
gunicorn
```

### 修改time-now.yml

把 lang 改成 python3-flask-gunicorn

```yaml
# time-now.yml
version: 1.0
provider:
  name: openfaas
  gateway: http://10.43.210.200:8080
functions:
  time-now:
    lang: python3-flask-gunicorn                         # <----
    handler: ./time-now
    image: time-now:latest
```

### 修改 fprocess 這個環境變數

fprocess 是當 of-watchdog 啟動之後會啟動的程序。流程是 docker run 會把 of-watchdog 啟動，不過它只負責收轉發 requests 到我們真正的程序裡面。這裡要告訴它我們的程序要怎麼啟動。

而前面理想中的例子是因為 fprocess 已經定義在 Dockerfile 裡頭了，所以用起來沒有感覺，要修改到這一層的時候就不得不搞清楚它的運作方式。

那 fprocess 是一個環境變數，不管你用甚麼方法，只要能把環境變數蓋得進去 K8S 的 pod → container 裡頭就可以了。當然可以修改 template/python3-flask-gunicorn/Dockerfile 把 fprocess 改掉。不過這裡我希望用K8S的 container.env 把它蓋掉。

打開 time-now.yml 加上環境變數，這個 environment 會在部署的時候放在 K8S 的 container.env 裡。留意我們是開 127.0.0.1:5000 ，跟理想中的版本裡 Flask 所開的 port 是一樣的。如果你 gunicorn 開在別的 port 的話，那還要再修改 upstream_url 這個環境變數 (一樣可以在 Dockerfile 找到它，不過還是建議用環境變數的方式設定，Dockerfile 那個就打個註解不要混淆自己)

```yaml
# time-now.yml
version: 1.0
provider:
  name: openfaas
  gateway: http://10.43.210.200:8080
functions:
  time-now:
    lang: python3-flask-gunicorn
    handler: ./time-now
    image: time-now:latest
    environment:                                         # <----
        fprocess: gunicorn index:app -b 127.0.0.1:5000   # <----
```

### 部署

```bash
faas up --skip-push -f time-now.yml
```

看一下 log ，看看 gunicorn 有沒有順利起來

```console
[root@localhost openfaas_function]# kubectl logs -n openfaas-fn deploy/time-now
Forking - gunicorn [index:app -b 127.0.0.1:5000]
2020/08/23 09:47:41 Started logging stderr from function.
2020/08/23 09:47:41 Started logging stdout from function.
2020/08/23 09:47:41 OperationalMode: http
2020/08/23 09:47:41 Timeouts: read: 10s, write: 10s hard: 10s.
2020/08/23 09:47:41 Listening on port: 8080
2020/08/23 09:47:41 Writing lock-file to: /tmp/.lock
2020/08/23 09:47:41 Metrics listening on port: 8081
2020/08/23 09:47:41 stderr: [2020-08-23 09:47:41 +0000] [12] [INFO] Starting gunicorn 20.0.4
2020/08/23 09:47:41 stderr: [2020-08-23 09:47:41 +0000] [12] [INFO] Listening at: http://127.0.0.1:5000 (12)
2020/08/23 09:47:41 stderr: [2020-08-23 09:47:41 +0000] [12] [INFO] Using worker: sync
2020/08/23 09:47:41 stderr: [2020-08-23 09:47:41 +0000] [17] [INFO] Booting worker with pid: 17
```

這樣就完成了，不過要注意的東西還是很多啦，這只是其中一樣而已，之後會講其他會需要注意跟修改的東西，也會有我修它 Bug 的分享。