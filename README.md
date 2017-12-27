# Gitlab and Gitlab runner with Docker

本文將介紹最基本的 CI 建置方式。過程極緻簡單，沒有什麼高能的操作。

### 環境需求

* Docker

### 建立一個專屬網路

```bash
$ docker network create --subnet=172.18.0.0/16 cicd
```

在 Docker 中建立一個網路，其網路位址範圍為：`172.18.0.0/16`，並命名為`cicd`。

### 建立 Gitlab-ce Container

```bash
$ docker run --network=cicd --add-host="gitlab.sean.com:172.18.0.2"  --ip=172.18.0.2 --hostname=gitlab.sean.com --name=gitlab -p 8080:80 -P -d gitlab/gitlab-ce
```

##### 指令介紹：

* `--network=cicd`：選擇要加入了網路`cicd`。
* `--add-host="gitlab.sean.com:172.18.0.2"`：將主機名稱及對應的 IP 加入此 Container 中（效果同 `/etc/hosts`）。`"gitlab.sean.com:172.18.0.2"`代表的就是 Gitlab-ce 自己。
* `--hostname=gitlab.sean.com`：設定此 Contrainer 的主機名稱。
* `--ip`：設定此 Contrainer 的 IP。
* `--name`：設定 Container 的代稱，方便往後的操作。
* `-p 8080:80`：將 Container 中的 80 Port 導向實體主機的 8080 Port。
* `-P`：其他需對外的 Port 則讓 Docker 隨機分配。
* `gitlab/gitlab-ce`：指定的 image。
* `-d`：直接進入背景執行。

### 建立 Gitlab-runner Container

```bash
$ docker run --network=cicd --add-host="gitlab.sean.com:172.18.0.2" --ip=172.18.0.3 --hostname=runner.sean.com --name=gitlab-runner -d gitlab/gitlab-runner
```

建立時的指令跟 Gitlab-ce 的雷同，小差別在於 runner 不需要設定對外的 Port。

### 設定 Gitlab

在瀏覽器上輸入`127.0.0.1:8080`，即可導向建置好的 Gitlab-ce 上。第一個畫面，就是要你設定管理者的密碼，設定好後，即可登入（`root/your password`）。接下來按上方的`板手`圖示，進入管理者設定頁面。

在管理這頁面中，可以在左邊找 Runners（Overview -> Runners ）。進去此頁面，就可以看到 Gitlab-ce 的`URL`及`Token`。接下來，就是設定 Gitlab-runner。

### 設定 Gitlab-runner

在 Gitlab-runner 上的設定也相當簡單。首先，向 Gitlab-ce 註冊服務。

##### 進入 Gitlab-runner

```bash
$ docker exec -it gitlab-runner bash
```

##### 開始註冊

```
root@runner:/# gitlab-runner register
Running in system-mode.
# 注意，這裡不是用 Gitlab-ce 所顯示的 URL ，而是建立 runner 時所設定的對應 URL
Please enter the gitlab-ci coordinator URL (e.g.https://gitlab.com/):
http://gitlab.sean.com
Please enter the gitlab-ci token for this runner:
xxxxxxxxx
Please enter the gitlab-ci description for this runner:
[runner.sean.com]: py27
Please enter the gitlab-ci tags for this runner (comma separated):
py2-tag,py3-tag
Whether to run untagged builds [true/false]:
true
Whether to lock the Runner to current project [true/false]:
true
Registering runner... succeeded                     runner=xxxxxx
# 在此選擇 shell 的意思，就是往後下達的指令直接由此 runner 進行。如設定 Docker 則會由這台 runner 執行 Docker 開啟指定 image 運行（gitlab-runner 本身沒有 docker 只有 docker machine）。
Please enter the executor: docker-ssh, shell, ssh, virtualbox, kubernetes, docker, parallels, docker+machine, docker-ssh+machine:
shell
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
```

執行完後，及註冊完畢！

##### 開啟權限

接下來開啟在執行 gitlab-runner 時的權限。如果沒有開啟，在往後執行`.gitlab.ci.yml`內的指令會常常遇到權限不足，無法執行的狀況（`sudo: no tty present and no askpass program specified`）。

###### 編輯 visudo

```bash
sudo visudo
```

###### 在其中加入

```bash
...
gitlab-runner ALL=(ALL) NOPASSWD: ALL
...
```

加入後儲存，即可完成設定。

> 在寫入時確認檔名是`/etc/sudoers`