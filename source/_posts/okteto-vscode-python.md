---
title: 使用 VS Code + Oketo 在 Kubernetes 遠端開發 - 以 Python 為例
date: 2020-06-21 23:30:46
tags:
---

本文介紹如何使用 VS Code 上的 [Remote - Kubernetes](https://marketplace.visualstudio.com/items?itemName=okteto.remote-kubernetes) (Okteto) 插件來在 K8S Cluster 內遠端開發 Python application。

（本文的範例程式碼也可以在 GitHub 找到：https://github.com/lockercho/okteto-python-example ）

## Okteto 介紹
Okteto 這個工具會幫你在 Kubernetes 設定好開發用的 deployment、同步檔案至遠端、以及設定完成後自動開啟 Remote-SSH 連線至遠端環境。

## 為什麼要使用遠端開發？

假設使用這個開發中的 application 未來也會 deploy 在 k8s cluster 上，遠端開發可以提供以下好處：
- 不用再多準備一份差很大 local config：在開發時可以直接使用 cluster config（儘管這份 config 可能是 dev 環境的，通常也會比 local config 更接近 production 環境）。
- 若開發的 application 依賴於 k8s 內的多個其他 service，直接在 cluster 內開發可以省去很多額外的 port forward 設定。
- 邪惡 debug 之術 :smiling_imp:：若使用運行中的 deployment 的名稱來建立環境，則 Okteto 會暫時取代掉原本的 deployment，開發結束才換回來（千萬別用在 production 環境，除非你百分之兩億知道自己在做什麼。)

接下來就開始手把手講解如何建立一個 Kubernetes 內開發環境。

## 設定流程
### Step 1: 準備一個 K8S Cluster

要設定 K8S 遠端開發，首先要準備一個你有權限跑 `kubectl apply` 的 K8S cluster。

如果沒有的話，[Okteto Cloud](https://cloud.okteto.com/#/login) 提供了免費的 cluster 可以使用，只要使用 GitHub 帳號註冊即可。（本文是使用 Mac OS X 上的 Docker Desktop 內建的 Kubernetes）

### Step 2: 安裝 VS Code 的 Remote - Kubernetes Extension

[Remote - Kubernetes](https://marketplace.visualstudio.com/items?itemName=okteto.remote-kubernetes)

### Step 3: 設定 Okteto.yaml

假設現在在開發一個 Python 的 hello-world 專案，你需要建立一個 `hello-world/okteto.yml` 檔案，內容如下：
```
# name 會是開發用的 deployment name
name: hello-world

# 使用 Okteto 提供的 Python Image
image: okteto/python:3-dev

command: ["bash"]

# Source Code 會被同步至這個 dir
workdir: "/opt/src/"

# 設定將遠端 port 5000 forward 到本機 port 25000，等等會用到
forward:
  - 25000:5000
```

### Step 3: 啟動遠端開發環境

接著 Cmd + Shift + p 開啟 Command Palette 使用 `Okteto: Up`
<div style="width:100%; text-align:center;">
<img src="/blog/2020/06/21/okteto-vscode-python/okteto-up.png" width="400px" title="Okteto Starting...">
</div>

再選擇你剛剛建立的 `hello-world/okteto.yaml` 設定檔。
<div style="width:100%; text-align:center;">
<img src="/blog/2020/06/21/okteto-vscode-python/okteto-select-manifest.png" width="400px" title="Okteto Starting...">
</div>
<br/>
選完設定檔後 VS Code 右下角會跳出提示正在準備你的 Remote 開發環境：
<div style="width:100%; text-align:center;">
<img src="/blog/2020/06/21/okteto-vscode-python/okteto-starting.png" width="400px" title="Okteto Starting...">
</div>

環境建立好後 terminal 會顯示以下訊息：
```
 ✓  Development container activated
 ✓  Files synchronized
    Namespace: default
    Name:      python-okteto-sample
    SSH:       22100 -> 2222
    Forward:   25000 -> 5000

Welcome to your development environment. Happy coding!
okteto>
```
同時 VS Code 會開啟一個新的視窗，可以看到上面中間的標題與左下狀態都有標記 [SSH: XXX.okteto] 代表這個視窗是遠端環境：
<div style="width:100%; text-align:center;">
<img src="/blog/2020/06/21/okteto-vscode-python/vscode-remote-welcome.png" width="800px" title="Okteto Starting...">
</div>

### Step 4: 設定 Python 環境

由於這個遠端是一個乾淨環境，所以需要重新安裝 Python 開發套件並 reload。
<div style="width:100%; text-align:center;">
<img src="/blog/2020/06/21/okteto-vscode-python/vscode-install-python-small.png" width="400px" title="Okteto Starting...">
</div>

接著選擇 Debugger 要使用的 Python Interpreter，Okteto 預設會幫你開一個 python `venv` 資料夾，但如果在 TERMINAL 下 `which python` 會發現是使用 `/usr/local/bin/python` 而非 `venv/bin/python`。這邊為了讓 debugger 方便抓到 pip 安裝的套件，我直接把 Interpreter 設定為 `/usr/local/bin/python`。
<div style="width:100%; text-align:center;">
<img src="/blog/2020/06/21/okteto-vscode-python/vscode-select-interpreter.png" width="800px" title="VS Code: select interpreter">
</div>
<br>
<div style="width:100%; text-align:center;">
<img src="/blog/2020/06/21/okteto-vscode-python/vscode-select-python-version.png" width="800px" title="VS Code: select interpreter">
</div>

接著使用 `Cmd +j` 叫出 VS Code TERMINAL，跑 `pip install Flask==1.1.2` 安裝 Flask
<div style="width:100%; text-align:center;">
<img src="/blog/2020/06/21/okteto-vscode-python/flask-install.png" width="800px" title="Install flask">
</div>

### Step 5: 建立測試用的 Python application

接下來用 Flask 建立一個簡單的測試 app。

建立 `app.py`：
```
# app.py

from flask import Flask

app = Flask(__name__)

@app.route("/hello")
def hello():
    return "World!"
```

建立 `.vscode/launch.json`
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Flask",
            "type": "python",
            "request": "launch",
            "module": "flask",
            "env": {
                "FLASK_APP": "app.py",
                "FLASK_ENV": "development",
                "FLASK_DEBUG": "0"
            },
            "args": [
                "run",
                "--no-debugger",
                "--no-reload"
            ]
        }
    ]
}
```

設定完了就可以使用 VS Code menu 的 `Run -> Start Debugging` 或 `F5` 開始 debug。
此時可以順便在 `line 7` 設定一個測試用的中斷點：
<div style="width:100%; text-align:center;">
<img src="/blog/2020/06/21/okteto-vscode-python/python-run-success.png" width="800px" title="Install flask">
</div>


由於一開始有設定了 Okteto 把 remote 的 `5000` port forward 到本機的 `25000` port，現在可以從 `localhost:25000` 測試剛剛寫的 `/hello` 這個 API：
```
$ curl http://localhost:25000/hello
```
如果剛剛設定了中斷點，應該會發現這個 curl 不會 return。切回 remote VS Code 視窗會發現中斷點被成功觸發了：
<div style="width:100%; text-align:center;">
<img src="/blog/2020/06/21/okteto-vscode-python/python-breakpoint-success.png" width="800px" title="Install flask">
</div>

到這邊就成功設定且測試我們的 Remote Python 開發環境了 :+1:

## Trouble Shooting 疑難排解
筆者在使用`Remote - Kubernetes` 遠端開發時經常遇到 `Okteto: Up` 指令失敗、逾時，或是雖然指令成功，卻沒有連上遠端環境等狀況，這時會需要分段檢查環境問題。

### 可能性一：遠端環境及 ssh 連線正常，只是 VS Code 沒有自動連上

這時可以先試著使用 Remote-SSH 連回遠端。`Cmd + Shift + P` 開啟 Command Palette，選擇 `Remote-SSH: Connect to Host`：
<div style="width:100%; text-align:center;">
<img src="/blog/2020/06/21/okteto-vscode-python/remote-ssh-connect.png" width="400px" title="Install flask">
</div>

再選擇要連線的遠端環境：
<div style="width:100%; text-align:center;">
<img src="/blog/2020/06/21/okteto-vscode-python/remote-ssh-select-host.png" width="400px" title="Install flask">
</div>

### 可能性二：SSH 連線失效
若直接使用 Remote-SSH 無法連回遠端，則需使用 `Okteto: Down` Deactivate 目前環境，再用 `Okteto: Up` 重開。有時候會發生 `Okteto: Down` 顯示成功，但 `Okteto: Up` 失敗的狀況，必須多跑幾次後才能再次使用 `Okteto: Up`。

### 可能性三：其他 Okteto Error
Okteto 預設會寫 log 進 `~/.okteto/okteto.log`，可以看到詳細的錯誤訊息，見招拆招。
```
tail ~/.okteto/okteto.log
```

## 其他設定方式

我自己測試有一部分的問題是來自於 `Remote - Kubernetes` 這個插件沒有正確的 launch 遠端環境或設定 port。

另外 `Remote - Kubernetes` 有另一個很大的限制是一次只能 launch 一個遠端開發環境 (GitHub 上有發相關的 [issue](https://github.com/okteto/remote-kubernetes/issues/101))。我觀察這是因為 `Okteto: Up` 這個指令在 command line 寫死了 `--remote 22100`，即使在 `okteto.yml` 設定了別的 `remote`，VS Code 插件這邊也會直接無視，希望之後官方會處理這個 bug。

因為上述的種種不穩問題，我現在都是自己分兩段設定：
1. 直接用 [Okteto CLI](https://okteto.com/docs/reference/cli/index.html)，先在 terminal 下  `okteto up -f ./okteto.yml` 確定環境成功跑起
2. 再用 `VS Code Remote SSH` 連上遠端環境（也就是 trouble shooting #1 的方式。）

兩段式用法有一個比較直接的缺點是，連上遠端後 VS Code 不會自動 cd 進 `okteto.yml` 裡設定的 `workdir`，而是會進到 `/root/`，要自己從 VS Code Side Bar 再切過去。


而前面也有說過其實 `Remote - Kubernetes` 就是 `Okteto` + `Remote-SSH`，所以若採用了這個兩段式用法，基本上可以直接移除 `Remote - Kubernetes` 了 XDDD
<br/>

## 結語

本文簡單介紹了如何使用 Okteto 的 `Remote - Kubernetes` VS Code extension 來建立 Kubernetes 中的 Python 開發環境。

除了本文介紹的部分以外，Okteto 其實有提供了更多的選項可以設定，包含了設定環境變數、限制 resources、設定多個 port、、設定 secrets 及 PersistentVolume 等等，詳細可以看官方的 [Manifest Reference](https://okteto.com/docs/reference/manifest/index.html)

遠端開發對相依於 K8S cluster 的 application 來說相當方便，但因為需要事前設定，我自己開發 API 時還是會先以本機開發 + fake data 為主，直到主要邏輯開發完畢後才放上 development cluster 測試，並搭配中斷點快速修正早期問題。但同事似乎也有人是直接以 Remote 開發為主，這個部分就很看個人，用的順手最重要。


## 參考資料
- [Remote Kubernetes Development in Visual Studio Code with Okteto (Golang)](https://medium.com/okteto/remote-kubernetes-development-in-visual-studio-code-with-okteto-8b96015b41a6)
- [How to Develop and Debug Python Applications in Kubernetes (using PyCharm)](https://okteto.com/blog/how-to-develop-python-apps-in-kubernetes/)
- [VS Code Remote - Kubernetes](https://marketplace.visualstudio.com/items?itemName=okteto.remote-kubernetes)
- [VS Code Remote - SSH](https://code.visualstudio.com/docs/remote/ssh)
- [Okteto Manifest Reference](https://okteto.com/docs/reference/manifest/index.html)


