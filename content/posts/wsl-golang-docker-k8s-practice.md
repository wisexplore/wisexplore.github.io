---
title: "WSL + Git + Go + Docker + K8s 实战回顾"
date: 2026-05-10
draft: false
description: "从 WSL 环境配置到 Go Web 服务、Docker 容器化、kind 本地 Kubernetes 部署的一次完整实践复盘。"
tags: ["WSL", "Git", "Go", "Docker", "Kubernetes", "kind"]
categories: ["编程环境", "云原生"]
ShowToc: true
TocOpen: false
---

> 本文记录一次从 Windows/WSL 环境开始，逐步完成 GoLand 开发、GitHub 推送、Go Web 服务开发、Docker 容器化、kind 本地 Kubernetes 部署，以及 Linux 代理问题排查的完整实践过程。

<!--more-->

---

## 0. 实践目标

本次实践的目标不是单纯“把一个程序跑起来”，而是打通一条接近真实后端发布流程的链路：

```text
Windows
  ↓
WSL Ubuntu
  ↓
GoLand 打开 WSL 项目
  ↓
Go 编写 Web 服务
  ↓
Git 管理代码并推送 GitHub
  ↓
Dockerfile 构建镜像
  ↓
Docker 运行容器
  ↓
kind 创建本地 Kubernetes 集群
  ↓
Deployment 创建 Pod
  ↓
Service 暴露服务
  ↓
port-forward 本地访问
  ↓
验证 Service 到多个 Pod 的转发
```

最终我们成功访问：

```bash
curl http://localhost:8080
curl http://localhost:8080/health
curl http://localhost:8080/version
```

返回：

```text
Hello from Go + Docker + WSL!
ok
{"arch":"amd64","hostname":"godemo1-545bbfc47-9f6qs","os":"linux","time":"2026-05-08 15:10:55","version":"v1"}
```

---

# 1. WSL 环境

## 1.1 查看 WSL 发行版

在 Windows PowerShell 中执行：

```bash
wsl -l -v
```

作用：查看当前安装的 WSL 发行版，以及版本是否为 WSL2。

理想状态类似：

```text
Ubuntu    Running    2
```

其中 `2` 表示 WSL2。

---

## 1.2 项目目录

本次项目放在 WSL 的 Linux 文件系统中：

```bash
~/project/godemo1
```

进入项目目录：

```bash
cd ~/project/godemo1
pwd
```

输出：

```text
~/project/godemo1
```

建议项目放在：

```text
~/project/xxx
```

不建议长期放在：

```text
/mnt/c/...
```

原因是：WSL 访问 Windows 文件系统会有额外开销，开发体验和文件监听性能都可能变差。

---

# 2. Go 环境

## 2.1 查看 Go 位置

```bash
which go
```

输出类似：

```text
/usr/bin/go
```

作用：查看当前系统使用的是哪个 `go` 命令。

---

## 2.2 查看 Go 版本

```bash
go version
```

本次环境：

```text
go version go1.26.0 linux/amd64
```

说明当前是在 Linux/amd64 环境下使用 Go。

---

## 2.3 查看 Go SDK 根目录

```bash
go env GOROOT
```

作用：查看 Go SDK 的真实安装目录。  
GoLand 配置 WSL Go SDK 时，通常参考这个路径。

---

## 2.4 配置 Go 模块代理

安装 kind 时，Go 需要下载依赖。为了避免下载慢，配置了 Go 代理：

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

查看配置：

```bash
go env GOPROXY
```

输出：

```text
https://goproxy.cn,direct
```

含义：

- 优先从 `goproxy.cn` 下载 Go 模块；
- 如果代理没有，再尝试 `direct` 直连源站。

---

# 3. Go 项目

## 3.1 项目结构

最终项目结构大致如下：

```text
godemo1/
├── go.mod
├── main.go
├── Dockerfile
├── .gitignore
├── .dockerignore
└── k8s/
    ├── deployment.yaml
    └── service.yaml
```

---

## 3.2 初始化 Go 模块

```bash
go mod init godemo1
```

作用：创建 `go.mod`，声明当前目录是一个 Go 模块。

`go.mod` 是 Go 项目的核心文件，应该提交到 Git。

---

## 3.3 一开始的命令行程序

最初程序只是打印信息：

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	fmt.Println("Hello, GoLand + WSL!")
	fmt.Println("当前时间：", time.Now().Format("2006-01-02 15:04:05"))
	fmt.Println("操作系统：", runtime.GOOS)
	fmt.Println("CPU架构：", runtime.GOARCH)
}
```

运行：

```bash
go run main.go
```

这种程序运行完就退出。  
它适合测试 Go 环境，但不适合放进 Kubernetes 的 `Deployment` 长期运行。

---

## 3.4 改造成 Web 服务

为了适配 Docker 和 K8s，后来改成 Web 服务：

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"runtime"
	"time"
)

var version = "v1"

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Hello from Go + Docker + WSL!")
	})

	http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "ok")
	})

	http.HandleFunc("/version", func(w http.ResponseWriter, r *http.Request) {
		data := map[string]string{
			"version":  version,
			"os":       runtime.GOOS,
			"arch":     runtime.GOARCH,
			"time":     time.Now().Format("2006-01-02 15:04:05"),
			"hostname": getHostname(),
		}

		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(data)
	})

	fmt.Println("Server started at :8080")
	http.ListenAndServe(":8080", nil)
}

func getHostname() string {
	name, _ := os.Hostname()
	return name
}
```

核心：

```go
http.ListenAndServe(":8080", nil)
```

作用：监听 8080 端口，让程序持续运行。

---

## 3.5 本地运行 Go Web 服务

```bash
go run main.go
```

另开终端测试：

```bash
curl http://localhost:8080
curl http://localhost:8080/health
curl http://localhost:8080/version
```

预期：

```text
Hello from Go + Docker + WSL!
ok
{"arch":"amd64","hostname":"...","os":"linux","time":"...","version":"v1"}
```

---

# 4. Git 与 GitHub

## 4.1 查看 Git 版本

```bash
git --version
```

输出：

```text
git version 2.53.0
```

说明 WSL 中已经可以使用 Git。

---

## 4.2 GitHub SSH 测试

```bash
ssh -T git@github.com
```

第一次连接时会提示：

```text
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

输入：

```text
yes
```

成功后显示：

```text
Hi wisexplore! You've successfully authenticated, but GitHub does not provide shell access.
```

这表示 SSH 认证成功。  
后半句不用管，它只是说明 GitHub 不提供远程 shell 登录。

---

## 4.3 初始化 Git 仓库

```bash
git init
```

作用：把当前目录变成 Git 仓库。

---

## 4.4 查看状态

```bash
git status
```

作用：查看哪些文件被修改、哪些文件还未跟踪。

这是 Git 排错最常用命令。

---

## 4.5 添加文件

```bash
git add .
```

作用：把当前目录的修改加入暂存区。

也可以指定文件：

```bash
git add main.go go.mod Dockerfile
```

---

## 4.6 提交

```bash
git commit -m "feat: containerize go web service"
```

作用：创建一次提交记录。

常用提交信息示例：

```text
feat: containerize go web service
chore: add ignore files
fix: remove idea files from git
```

---

## 4.7 绑定远程仓库

查看远程仓库：

```bash
git remote -v
```

本次远程仓库：

```text
origin  git@github.com:wisexplore/godemo1.git (fetch)
origin  git@github.com:wisexplore/godemo1.git (push)
```

添加远程仓库：

```bash
git remote add origin git@github.com:wisexplore/godemo1.git
```

如果已经存在，则修改：

```bash
git remote set-url origin git@github.com:wisexplore/godemo1.git
```

---

## 4.8 推送 master 分支

因为习惯使用 `master`，所以没有改成 `main`。

首次推送：

```bash
git push -u origin master
```

含义：

- `origin`：远程仓库名称；
- `master`：本地分支；
- `-u`：建立本地 `master` 与远程 `origin/master` 的默认关联。

以后可以直接：

```bash
git push
```

---

## 4.9 已提交 `.idea` 后的处理

GoLand 可能生成 `.idea/`。  
一般不建议提交 IDE 配置。

如果 `.idea` 已经被 Git 跟踪，可以执行：

```bash
git rm -r --cached .idea
```

作用：

```text
从 Git 仓库中移除 .idea
但本地 .idea 文件仍然保留
```

然后提交：

```bash
git add .gitignore
git commit -m "chore: remove idea files from git"
git push
```

---

# 5. .gitignore

`.gitignore` 的作用：告诉 Git 哪些文件不要提交。

推荐内容：

```gitignore
# Go build output
godemo1
*.exe
*.exe~
*.dll
*.so
*.dylib
*.test
*.out

# Go workspace
go.work

# IDE
.idea/
.vscode/

# OS files
.DS_Store
Thumbs.db

# Logs
*.log

# Env files
.env
```

注意：

```text
go.mod 要提交
go.sum 要提交
main.go 要提交
Dockerfile 要提交
k8s/*.yaml 要提交
.idea/ 不提交
本地编译产物 godemo1 不提交
```

---

# 6. Linux 编辑器小抄

## 6.1 nano

打开文件：

```bash
nano 文件名
```

常用操作：

```text
Ctrl + O    保存
Enter       确认保存
Ctrl + X    退出
Ctrl + K    删除当前行
```

---

## 6.2 vim

进入 vim 后，先记住：

```text
Esc         回到普通模式
dd          删除当前行
:w          保存
:q          退出
:wq         保存并退出
:q!         不保存强制退出
```

`:wq` 的意思是：

```text
write + quit
保存 + 退出
```

---

# 7. WSL 代理问题

## 7.1 问题现象

访问本地服务时：

```bash
curl -v http://localhost:8080
```

出现：

```text
Uses proxy env variable http_proxy == 'http://<proxy-host>:<proxy-port>'
HTTP/1.1 502 Bad Gateway
```

说明 `localhost` 请求也被代理转发了，导致访问失败。

---

## 7.2 临时绕过代理

```bash
curl --noproxy localhost,127.0.0.1 http://localhost:8080
```

成功返回：

```text
Hello from Go + Docker + WSL!
```

---

## 7.3 配置 no_proxy

在 `~/.bashrc` 中加入：

```bash
export http_proxy=http://<proxy-host>:<proxy-port>
export https_proxy=http://<proxy-host>:<proxy-port>
export ALL_PROXY=socks5://<proxy-host>:<proxy-port>

export no_proxy=localhost,127.0.0.1,::1
export NO_PROXY=localhost,127.0.0.1,::1
```

生效：

```bash
source ~/.bashrc
```

查看代理变量：

```bash
env | grep -i proxy
```

含义：

- `http_proxy` / `https_proxy`：访问外网时走代理；
- `ALL_PROXY`：通用代理；
- `no_proxy` / `NO_PROXY`：这些地址不走代理。

---

# 8. Docker 安装与验证

## 8.1 Docker Desktop WSL 集成

在 Docker Desktop 中开启：

```text
Settings → Resources → WSL Integration
```

勾选：

```text
Enable integration with my default WSL distro
Ubuntu
```

然后 `Apply & Restart`。

---

## 8.2 查看 Docker 版本

```bash
docker version
```

成功时会看到：

```text
Client: linux/amd64
Server: Docker Desktop
```

说明 WSL 中的 Docker 客户端已经连上 Docker Desktop 的服务端。

---

## 8.3 测试 Docker

```bash
docker run hello-world
```

第一次会拉取镜像：

```text
Unable to find image 'hello-world:latest' locally
Pulling from library/hello-world
```

成功后：

```text
Hello from Docker!
```

这说明 Docker 安装正常。

---

## 8.4 查看镜像

```bash
docker images
```

作用：查看本地已有镜像。

---

## 8.5 查看容器

查看正在运行的容器：

```bash
docker ps
```

查看所有容器，包括已经退出的：

```bash
docker ps -a
```

本次看到：

```text
hello-world   Exited (0)
godemo1:v1    Exited (0)
```

说明这些容器运行完后已经退出。

`Exited (0)` 表示正常退出。

---

# 9. Dockerfile

## 9.1 Dockerfile 内容

```dockerfile
# 第一阶段：编译 Go 程序
FROM golang:1.26 AS builder

WORKDIR /app

COPY go.mod ./
RUN go mod download

COPY . .

RUN go build -o godemo1

# 第二阶段：运行程序
FROM debian:bookworm-slim

WORKDIR /app

COPY --from=builder /app/godemo1 .

EXPOSE 8080

CMD ["./godemo1"]
```

这是一个多阶段构建。

---

## 9.2 第一阶段：编译

```dockerfile
FROM golang:1.26 AS builder
```

使用 Go 官方镜像作为编译环境。

```dockerfile
WORKDIR /app
```

设置容器内工作目录。

```dockerfile
COPY go.mod ./
RUN go mod download
```

先复制 `go.mod` 并下载依赖，用于利用 Docker 缓存。

```dockerfile
COPY . .
RUN go build -o godemo1
```

复制源码并编译出可执行文件。

---

## 9.3 第二阶段：运行

```dockerfile
FROM debian:bookworm-slim
```

使用精简 Debian 镜像作为运行环境。

```dockerfile
COPY --from=builder /app/godemo1 .
```

从第一阶段复制编译好的程序。

```dockerfile
EXPOSE 8080
```

声明容器内服务监听 8080 端口。

```dockerfile
CMD ["./godemo1"]
```

容器启动时运行 `./godemo1`。

---

## 9.4 为什么使用多阶段构建

如果只用 `golang` 镜像，最终镜像会包含：

```text
Go 编译器
Go 工具链
源码
依赖缓存
```

使用多阶段构建后，最终镜像主要包含：

```text
Debian 精简系统
编译好的 godemo1 可执行文件
```

好处：

```text
镜像更小
运行环境更干净
更接近生产部署方式
```

---

# 10. .dockerignore

`.dockerignore` 的作用：告诉 Docker 构建镜像时哪些文件不要打进构建上下文。

推荐内容：

```gitignore
.git
.gitignore

.idea/
.vscode/

godemo1
*.exe
*.test
*.out
bin/
dist/

.DS_Store
Thumbs.db

*.log
```

`.gitignore` 和 `.dockerignore` 区别：

```text
.gitignore：控制哪些文件不进 Git
.dockerignore：控制哪些文件不进 Docker 构建上下文
```

---

# 11. Docker 构建与运行

## 11.1 构建镜像

```bash
docker build -t godemo1:v1 .
```

解释：

```text
docker build        构建镜像
-t godemo1:v1       镜像名 godemo1，标签 v1
.                   当前目录作为构建上下文
```

---

## 11.2 运行容器

```bash
docker run -d --name godemo1 -p 8080:8080 godemo1:v1
```

解释：

```text
-d                 后台运行
--name godemo1     容器名
-p 8080:8080       宿主机 8080 → 容器 8080
godemo1:v1         使用的镜像
```

---

## 11.3 查看日志

```bash
docker logs godemo1
```

作用：查看容器标准输出。

---

## 11.4 进入容器

```bash
docker exec -it godemo1 sh
```

作用：进入正在运行的容器内部。

---

## 11.5 删除容器

```bash
docker rm -f godemo1
```

作用：强制删除名为 `godemo1` 的容器。

---

# 12. Kubernetes 工具链

本地 K8s 使用：

```text
kind + kubectl
```

其中：

```text
kind：用 Docker 创建本地 Kubernetes 集群
kubectl：操作 Kubernetes 集群的命令行工具
```

---

## 12.1 安装 kind

```bash
go install sigs.k8s.io/kind@v0.31.0
```

安装后配置 PATH：

```bash
echo 'export PATH=$PATH:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc
```

验证：

```bash
kind version
```

输出：

```text
kind v0.31.0 go1.26.0 linux/amd64
```

---

## 12.2 Ctrl + Z 与 Ctrl + C

安装 kind 时误按过 `Ctrl + Z`。

区别：

```text
Ctrl + Z：暂停任务，挂到后台
Ctrl + C：中断任务
```

查看后台任务：

```bash
jobs
```

杀掉挂起任务：

```bash
kill %1
kill %2
```

---

## 12.3 安装 kubectl

下载：

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

安装：

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

验证：

```bash
kubectl version --client
```

---

# 13. 创建 kind 集群

## 13.1 创建集群

```bash
kind create cluster --name godemo
```

成功输出：

```text
Creating cluster "godemo" ...
✓ Ensuring node image (kindest/node:v1.35.0)
✓ Preparing nodes
✓ Writing configuration
✓ Starting control-plane
✓ Installing CNI
✓ Installing StorageClass
Set kubectl context to "kind-godemo"
```

含义：

- 拉取 `kindest/node` 节点镜像；
- 用 Docker 创建一个模拟的 K8s 节点；
- 启动 Kubernetes 控制面；
- 安装 CNI 网络；
- 安装默认 StorageClass；
- 设置 kubectl 当前上下文为 `kind-godemo`。

---

## 13.2 查看集群节点

```bash
kubectl get nodes
```

输出类似：

```text
godemo-control-plane   Ready
```

说明本地 K8s 集群已经运行。

---

## 13.3 kind 与 Docker 的关系

查看 Docker 容器：

```bash
docker ps
```

会看到：

```text
godemo-control-plane
```

它是一个 Docker 容器，但在 kind 中模拟一个 Kubernetes Node。

真实生产环境：

```text
真实服务器 / 云主机 = K8s Node
```

本地 kind 环境：

```text
Docker 容器 = 模拟出来的 K8s Node
```

---

# 14. K8s、Node、Pod、Container、Docker、containerd 的关系

## 14.1 核心层级

```text
Kubernetes Cluster
  ↓
Node，也就是服务器
  ↓
Pod
  ↓
Container
  ↓
应用程序
```

---

## 14.2 生产环境

```text
K8s 集群
├── Node 1：真实服务器 / 云主机
│   ├── Pod A
│   │   └── Container：Go 服务
│   └── Pod B
│       └── Container：Nginx
├── Node 2：真实服务器 / 云主机
│   └── Pod C
│       └── Container：订单服务
└── Node 3：真实服务器 / 云主机
    └── Pod D
        └── Container：支付服务
```

---

## 14.3 kind 本地环境

```text
Windows
  ↓
WSL Ubuntu
  ↓
Docker Desktop
  ↓
godemo-control-plane 容器
  ↓
模拟 K8s Node
  ↓
Pod
  ↓
Container
  ↓
godemo1 程序
```

---

## 14.4 containerd 的作用

K8s 不直接运行容器。  
完整链路是：

```text
K8s 控制面
  ↓
Scheduler 决定 Pod 放哪台 Node
  ↓
Node 上的 kubelet 接收任务
  ↓
kubelet 调用 containerd
  ↓
containerd 启动容器
  ↓
容器运行 godemo1 程序
```

简记：

```text
K8s 是指挥官
kubelet 是节点代理
containerd 是真正启动容器的执行者
容器里跑应用程序
```

---

## 14.5 公司平台概念映射

| 公司平台说法 | K8s 对应                       |
| ------------ | ------------------------------ |
| 应用         | Deployment                     |
| 实例         | Pod                            |
| 实例数       | replicas                       |
| 实例规格     | resources requests / limits    |
| 服务器       | Node                           |
| 镜像版本     | image                          |
| 健康检查     | readinessProbe / livenessProbe |
| 发布         | rollout                        |
| 回滚         | rollout undo                   |
| 服务暴露     | Service / Ingress              |

---

## 14.6 实例规格

公司平台里选择：

```text
1C 2G
2C 4G
```

底层通常会转成：

```yaml
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "1"
    memory: "2Gi"
```

含义：

```text
requests：调度时申请的资源
limits：运行时最多能用的资源
```

Scheduler 会根据 `requests` 把 Pod 放到资源足够的 Node 上。

---

## 14.7 Pod 与 sidecar

一个 Pod 可以有一个或多个容器。

常见：

```text
Pod
└── app 容器
```

带 sidecar：

```text
Pod
├── app 容器
└── sidecar 容器
```

sidecar 本质也是容器，只是用于辅助主容器，例如：

```text
日志采集
流量代理
配置同步
服务网格 Envoy
```

---

# 15. 将镜像导入 kind

本地 Docker 有镜像，不代表 kind 节点内部也能直接使用。  
需要导入：

```bash
kind load docker-image godemo1:v1 --name godemo
```

输出：

```text
Image: "godemo1:v1" with ID "sha256:..." not yet present on node "godemo-control-plane", loading...
```

含义：把本地 Docker 镜像加载到 kind 的节点中。

---

# 16. K8s Deployment

## 16.1 deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: godemo1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: godemo1
  template:
    metadata:
      labels:
        app: godemo1
    spec:
      containers:
        - name: godemo1
          image: godemo1:v1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
```

---

## 16.2 字段解释

```yaml
kind: Deployment
```

表示创建 Deployment。

Deployment 负责：

```text
管理 Pod 副本数
滚动更新
失败恢复
回滚
```

---

```yaml
replicas: 2
```

表示期望运行 2 个 Pod。  
公司平台中的“实例数 2”通常对应这个。

---

```yaml
selector:
  matchLabels:
    app: godemo1
```

Deployment 通过标签管理 Pod。

---

```yaml
template:
  metadata:
    labels:
      app: godemo1
```

创建出来的 Pod 会带有标签：

```text
app=godemo1
```

---

```yaml
image: godemo1:v1
```

Pod 中容器使用的镜像。

---

```yaml
imagePullPolicy: IfNotPresent
```

表示如果节点本地有镜像，就不去远程拉取。  
kind 本地实验建议加这个。

---

```yaml
readinessProbe
```

就绪检查。  
用于判断 Pod 是否可以接收流量。

---

```yaml
livenessProbe
```

存活检查。  
用于判断容器是否还活着；如果检查失败，K8s 会重启容器。

---

# 17. K8s Service

## 17.1 service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: godemo1-service
spec:
  selector:
    app: godemo1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: ClusterIP
```

---

## 17.2 字段解释

```yaml
kind: Service
```

表示创建 Service。

Service 作用：给一组 Pod 提供稳定访问入口。

---

```yaml
selector:
  app: godemo1
```

Service 会选择所有带有 `app=godemo1` 标签的 Pod。

---

```yaml
port: 80
```

Service 暴露的端口。

---

```yaml
targetPort: 8080
```

转发到 Pod 容器内的 8080 端口。

---

```yaml
type: ClusterIP
```

表示该 Service 默认只能在集群内部访问。

本地访问需要使用：

```bash
kubectl port-forward
```

---

# 18. 应用 K8s 配置

```bash
kubectl apply -f k8s/
```

作用：应用 `k8s/` 目录下所有 YAML 文件。

会创建：

```text
Deployment
Service
```

---

# 19. 查看 K8s 资源

## 19.1 查看 Deployment

```bash
kubectl get deploy
```

正常：

```text
NAME      READY   UP-TO-DATE   AVAILABLE
godemo1   2/2     2            2
```

---

## 19.2 查看 Pod

```bash
kubectl get pods
```

正常：

```text
godemo1-545bbfc47-9f6qs   1/1   Running
godemo1-545bbfc47-m4767   1/1   Running
```

---

## 19.3 持续观察 Pod

```bash
kubectl get pods -w
```

`-w` 表示 watch，动态观察资源变化。

退出：

```text
Ctrl + C
```

---

## 19.4 查看 Pod 详细信息

```bash
kubectl get pods -o wide
```

输出：

```text
NAME                      READY   STATUS    IP           NODE
godemo1-545bbfc47-9f6qs   1/1     Running   10.244.0.7   godemo-control-plane
godemo1-545bbfc47-m4767   1/1     Running   10.244.0.8   godemo-control-plane
```

说明两个 Pod 分别有自己的 Pod IP。

---

## 19.5 查看 Service

```bash
kubectl get svc
```

输出：

```text
godemo1-service   ClusterIP   10.96.16.192   <none>   80/TCP
```

---

## 19.6 查看 Service 后端

```bash
kubectl get endpoints godemo1-service -o wide
```

输出：

```text
godemo1-service   10.244.0.7:8080,10.244.0.8:8080
```

说明 Service 后面确实挂了两个 Pod。

新版 K8s 推荐看：

```bash
kubectl get endpointslice
```

---

# 20. 访问 K8s 服务

## 20.1 port-forward

```bash
kubectl port-forward svc/godemo1-service 8080:80
```

作用：

```text
本机 localhost:8080
  ↓
godemo1-service:80
  ↓
Pod:8080
```

另开一个终端访问：

```bash
curl http://localhost:8080
curl http://localhost:8080/health
curl http://localhost:8080/version
```

---

## 20.2 port-forward 与负载均衡

连续请求：

```bash
for i in {1..10}; do curl http://localhost:8080/version; echo; done
```

发现一直打到同一个 Pod。

原因：

```text
kubectl port-forward 更偏调试工具
它可能固定转发到 Service 后面的某一个 Pod
不适合验证 Service 负载均衡
```

---

## 20.3 从集群内部验证 Service 转发

创建临时 curl Pod：

```bash
kubectl run curl-test --image=curlimages/curl -it --rm --restart=Never -- sh
```

进入后执行：

```sh
for i in $(seq 1 10); do curl http://godemo1-service/version; echo; done
```

结果中 hostname 在两个 Pod 之间切换：

```text
godemo1-545bbfc47-m4767
godemo1-545bbfc47-9f6qs
godemo1-545bbfc47-m4767
godemo1-545bbfc47-9f6qs
```

说明：

```text
Service 后面确实有两个 Pod
集群内部访问 Service 时，请求会转发到不同 Pod
```

退出临时容器：

```bash
exit
```

或：

```text
Ctrl + D
```

因为创建时用了 `--rm`，退出后 Pod 自动删除。

---

# 21. CrashLoopBackOff 排查

## 21.1 问题现象

```bash
kubectl get pods
```

输出：

```text
godemo1-xxx   0/1   CrashLoopBackOff
```

含义：

```text
容器启动
  ↓
程序退出
  ↓
K8s 重启
  ↓
再次退出
  ↓
进入 CrashLoopBackOff
```

---

## 21.2 查看日志

```bash
kubectl logs deploy/godemo1
```

当时日志：

```text
Hello, GoLand + WSL!
当前时间：...
操作系统： linux
CPU架构： amd64
```

说明镜像里还是旧版一次性程序，打印完就退出。

---

## 21.3 解决方法

确认代码已经改成 Web 服务后，重新构建镜像：

```bash
docker build -t godemo1:v1 .
```

重新导入 kind：

```bash
kind load docker-image godemo1:v1 --name godemo
```

重启 Deployment：

```bash
kubectl rollout restart deployment/godemo1
```

观察：

```bash
kubectl get pods -w
```

最后变成：

```text
1/1 Running
```

---

# 22. 滚动更新

## 22.1 触发滚动重启

```bash
kubectl rollout restart deployment/godemo1
```

作用：让 Deployment 重新创建 Pod。

本次观察到：

```text
旧 ReplicaSet：godemo1-5ff677dc9
新 ReplicaSet：godemo1-545bbfc47
```

滚动过程：

```text
创建新 Pod
新 Pod Ready
删除旧 Pod
保持期望副本数
```

---

## 22.2 查看滚动状态

```bash
kubectl rollout status deployment/godemo1
```

成功：

```text
deployment "godemo1" successfully rolled out
```

---

## 22.3 发布 v2

修改代码：

```go
var version = "v2"
```

构建新镜像：

```bash
docker build -t godemo1:v2 .
```

导入 kind：

```bash
kind load docker-image godemo1:v2 --name godemo
```

更新镜像：

```bash
kubectl set image deployment/godemo1 godemo1=godemo1:v2
```

查看状态：

```bash
kubectl rollout status deployment/godemo1
```

访问：

```bash
curl http://localhost:8080/version
```

---

## 22.4 回滚

```bash
kubectl rollout undo deployment/godemo1
```

作用：回滚到上一个版本。  
这就是公司平台“回滚”的底层逻辑。

---

# 23. 自愈实验

查看 Pod：

```bash
kubectl get pods
```

删除一个 Pod：

```bash
kubectl delete pod godemo1-545bbfc47-9f6qs
```

观察：

```bash
kubectl get pods -w
```

现象：

```text
旧 Pod 被删除
新 Pod 自动创建
最终仍然保持 2 个 Running
```

原因：

```yaml
replicas: 2
```

Deployment 会持续维护期望状态。

---

# 24. 常用命令汇总

## 24.1 Go

```bash
go version
go env GOROOT
go env GOPROXY
go env -w GOPROXY=https://goproxy.cn,direct
go mod init godemo1
go run main.go
go build -o godemo1
```

---

## 24.2 Git

```bash
git status
git add .
git commit -m "说明"
git push
git remote -v
git remote add origin git@github.com:wisexplore/godemo1.git
git remote set-url origin git@github.com:wisexplore/godemo1.git
git rm -r --cached .idea
```

---

## 24.3 Docker

```bash
docker version
docker run hello-world
docker images
docker ps
docker ps -a
docker build -t godemo1:v1 .
docker run -d --name godemo1 -p 8080:8080 godemo1:v1
docker logs godemo1
docker exec -it godemo1 sh
docker rm -f godemo1
```

---

## 24.4 kind

```bash
kind version
kind create cluster --name godemo
kind get clusters
kind load docker-image godemo1:v1 --name godemo
kind delete cluster --name godemo
```

---

## 24.5 kubectl

```bash
kubectl version --client
kubectl get nodes
kubectl get deploy
kubectl get rs
kubectl get pods
kubectl get pods -w
kubectl get pods -o wide
kubectl get svc
kubectl get endpoints godemo1-service -o wide
kubectl get endpointslice
kubectl apply -f k8s/
kubectl logs deploy/godemo1
kubectl logs pod名字
kubectl logs pod名字 --previous
kubectl describe pod pod名字
kubectl port-forward svc/godemo1-service 8080:80
kubectl rollout restart deployment/godemo1
kubectl rollout status deployment/godemo1
kubectl rollout undo deployment/godemo1
kubectl delete pod pod名字
```

---

# 25. 本次实践的核心理解

## 25.1 Go、Docker、K8s 的关系

```text
Go 负责写服务
Docker 负责把服务打包成镜像
K8s 负责把镜像以 Pod 的形式调度到 Node 上运行
Service 负责给这些 Pod 提供稳定访问入口
```

---

## 25.2 Docker 与 K8s 的区别

```text
Docker：
  构建镜像
  本地运行容器
  查看容器日志
  做开发验证

K8s：
  管理多个 Pod
  维护副本数
  自动恢复
  滚动更新
  服务发现
  暴露服务
```

---

## 25.3 最重要的一条链路

```text
main.go
  ↓
Dockerfile
  ↓
docker build -t godemo1:v1 .
  ↓
kind load docker-image godemo1:v1 --name godemo
  ↓
kubectl apply -f k8s/
  ↓
Deployment 创建 Pod
  ↓
Service 选择 Pod
  ↓
kubectl port-forward
  ↓
curl localhost:8080
```

---

# 26. 下一步可以继续深入的方向

## 方向 1：资源规格

在 Deployment 中增加：

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

理解公司平台中的“实例规格”。

---

## 方向 2：ConfigMap

把配置从代码中拿出来：

```text
版本号
环境名
开关配置
```

用 `ConfigMap` 注入到容器。

---

## 方向 3：Secret

学习密码、Token、数据库连接串如何注入。

---

## 方向 4：Ingress

目前使用 `port-forward`。  
后续可以学习 Ingress，让服务通过域名访问。

---

## 方向 5：HPA 自动扩缩容

学习：

```text
CPU 高时自动扩容
CPU 降低时自动缩容
```

---

## 方向 6：日志与监控

继续学习：

```text
kubectl logs
Prometheus
Grafana
Loki / ELK
```

---

# 27. 总结

这次实践从基础环境开始，一路打通了：

```text
WSL
GitHub SSH
Go Web 服务
Docker 容器化
Dockerfile 多阶段构建
Linux 代理排查
kind 本地 K8s 集群
Deployment
Pod
Service
port-forward
Service 多 Pod 转发验证
CrashLoopBackOff 排查
滚动重启
```

最终得到的核心认知是：

> **公司平台上的“实例”通常就是 Pod；“实例数”对应 replicas；“实例规格”对应 resources；服务器在 K8s 中叫 Node；K8s 负责调度和维护期望状态；containerd/Docker 负责真正运行容器；Service 给多个 Pod 提供稳定入口。**


---

# 附录 A：Windows + WSL 安装与基础配置

这一部分补充从零开始安装 WSL 的流程，适合以后换电脑或重装环境时参考。

## A.1 检查 Windows 版本

在 PowerShell 中执行：

```powershell
winver
```

作用：查看 Windows 版本。  
一般 Windows 10 较新版本或 Windows 11 都可以直接使用 WSL2。

---

## A.2 安装 WSL

以管理员身份打开 PowerShell，执行：

```powershell
wsl --install
```

作用：

```text
安装 WSL
安装默认 Linux 发行版
安装必要的虚拟化组件
```

安装完成后通常需要重启电脑。

---

## A.3 指定安装 Ubuntu

如果想明确安装 Ubuntu：

```powershell
wsl --install -d Ubuntu
```

查看可安装的发行版：

```powershell
wsl --list --online
```

或：

```powershell
wsl -l -o
```

---

## A.4 查看已安装 WSL

```powershell
wsl -l -v
```

作用：查看当前安装的 WSL 发行版、运行状态和 WSL 版本。

示例：

```text
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

---

## A.5 设置默认使用 WSL2

```powershell
wsl --set-default-version 2
```

作用：以后新安装的 Linux 发行版默认使用 WSL2。

如果某个发行版是 WSL1，可以转换：

```powershell
wsl --set-version Ubuntu 2
```

---

## A.6 更新 WSL

```powershell
wsl --update
```

如果 Microsoft Store 有问题，可以用：

```powershell
wsl --update --web-download
```

关闭所有 WSL 实例：

```powershell
wsl --shutdown
```

这个命令很常用。  
当 Docker Desktop、WSL 网络、代理、DNS 状态异常时，可以先执行它。

---

## A.7 进入 Ubuntu

在开始菜单打开 Ubuntu，或者 PowerShell 执行：

```powershell
wsl
```

进入后可以看到类似：

```bash
user@WSL:~$
```

---

## A.8 更新 Ubuntu 软件源

进入 WSL 后，建议先执行：

```bash
sudo apt update
sudo apt upgrade -y
```

含义：

```text
sudo apt update      更新软件源索引
sudo apt upgrade -y  升级已安装的软件包
```

---

## A.9 安装常用基础工具

```bash
sudo apt install -y curl wget vim nano unzip zip ca-certificates gnupg lsb-release net-tools
```

常用工具说明：

| 工具              | 作用                               |
| ----------------- | ---------------------------------- |
| `curl`            | 发送 HTTP 请求，下载文件，测试接口 |
| `wget`            | 下载文件                           |
| `vim`             | 文本编辑器                         |
| `nano`            | 简单文本编辑器                     |
| `unzip` / `zip`   | 解压和压缩                         |
| `ca-certificates` | HTTPS 证书支持                     |
| `gnupg`           | GPG 密钥工具                       |
| `lsb-release`     | 查看 Linux 发行版信息              |
| `net-tools`       | 提供 ifconfig 等传统网络工具       |

---

## A.10 推荐项目目录

```bash
mkdir -p ~/project
cd ~/project
```

以后项目放这里：

```text
~/project
```

不建议把项目放在：

```text
/mnt/c/Users/...
```

原因：WSL 访问 Windows 文件系统较慢，且某些文件权限和监听机制不如 Linux 原生目录稳定。

---

# 附录 B：Git 安装与 GitHub SSH 配置

## B.1 安装 Git

在 WSL Ubuntu 中执行：

```bash
sudo apt update
sudo apt install -y git
```

查看版本：

```bash
git --version
```

示例：

```text
git version 2.53.0
```

---

## B.2 配置 Git 用户名和邮箱

```bash
git config --global user.name "wisexplore"
git config --global user.email "你的GitHub邮箱"
```

查看配置：

```bash
git config --global --list
```

作用：设置 Git 提交记录中的作者信息。

---

## B.3 生成 SSH Key

```bash
ssh-keygen -t ed25519 -C "你的GitHub邮箱"
```

一路回车即可。

生成后查看：

```bash
ls ~/.ssh
```

常见文件：

```text
id_ed25519      私钥，不要泄露
id_ed25519.pub  公钥，可以添加到 GitHub
```

---

## B.4 查看公钥

```bash
cat ~/.ssh/id_ed25519.pub
```

复制输出内容，添加到 GitHub：

```text
GitHub → Settings → SSH and GPG keys → New SSH key
```

---

## B.5 测试 GitHub SSH

```bash
ssh -T git@github.com
```

第一次会提示：

```text
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

输入：

```text
yes
```

成功：

```text
Hi wisexplore! You've successfully authenticated, but GitHub does not provide shell access.
```

---

## B.6 创建本地 Git 仓库

```bash
cd ~/project/godemo1
git init
```

查看状态：

```bash
git status
```

---

## B.7 绑定远程仓库

```bash
git remote add origin git@github.com:wisexplore/godemo1.git
```

如果已经绑定过，修改地址：

```bash
git remote set-url origin git@github.com:wisexplore/godemo1.git
```

查看远程仓库：

```bash
git remote -v
```

---

## B.8 提交与推送

```bash
git add .
git commit -m "init: init wsl godemo1 project"
git push -u origin master
```

如果已经建立关联，以后直接：

```bash
git push
```

---

## B.9 常见 Git 问题

### 1. Repository not found

报错：

```text
ERROR: Repository not found.
fatal: Could not read from remote repository.
```

常见原因：

```text
GitHub 上还没有创建这个仓库
仓库名写错
当前 SSH 账号没有权限
```

检查远程地址：

```bash
git remote -v
```

---

### 2. go.mod 没推上去

查看状态：

```bash
git status
```

如果显示：

```text
Untracked files:
  go.mod
```

执行：

```bash
git add go.mod
git commit -m "add go module file"
git push
```

---

### 3. 已推送 .idea 怎么办

```bash
git rm -r --cached .idea
git add .gitignore
git commit -m "chore: remove idea files from git"
git push
```

`--cached` 的作用：从 Git 仓库移除，但本地文件保留。

---

### 4. rebase 过程中想取消

```bash
git rebase --abort
```

继续 rebase：

```bash
git rebase --continue
```

---

# 附录 C：Go / Golang 安装

## C.1 方法一：apt 安装

```bash
sudo apt update
sudo apt install -y golang-go
```

查看版本：

```bash
go version
```

查看 go 命令位置：

```bash
which go
```

查看 GOROOT：

```bash
go env GOROOT
```

优点：简单。  
缺点：Ubuntu 软件源里的版本可能不是最新。

---

## C.2 方法二：官方压缩包安装

如果需要安装指定版本，可以使用官方 tar 包。示例：

```bash
cd /tmp
wget https://go.dev/dl/go1.26.0.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.26.0.linux-amd64.tar.gz
```

配置 PATH：

```bash
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

查看：

```bash
go version
which go
go env GOROOT
```

如果输出类似：

```text
/usr/local/go/bin/go
```

说明使用的是官方安装版本。

---

## C.3 Go 常用环境变量

查看所有 Go 环境变量：

```bash
go env
```

常看几个：

```bash
go env GOPATH
go env GOROOT
go env GOPROXY
```

含义：

| 变量      | 作用                                  |
| --------- | ------------------------------------- |
| `GOROOT`  | Go SDK 安装目录                       |
| `GOPATH`  | Go 工作区和 `go install` 默认安装目录 |
| `GOPROXY` | Go 模块下载代理                       |

---

## C.4 配置 GOPROXY

```bash
go env -w GOPROXY=https://goproxy.cn,direct
```

查看：

```bash
go env GOPROXY
```

---

## C.5 go install 安装工具

例如安装 kind：

```bash
go install sigs.k8s.io/kind@v0.31.0
```

Go 会把可执行文件安装到：

```text
$HOME/go/bin
```

所以要配置 PATH：

```bash
echo 'export PATH=$PATH:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc
```

---

# 附录 D：Docker Desktop 安装与 WSL 集成

## D.1 安装选择

安装 Docker Desktop for Windows 时，推荐选择：

```text
✅ All users installation
✅ Use WSL 2 instead of Hyper-V
⬜ Allow Windows Containers
✅ Add shortcut to desktop
```

说明：

| 选项                         | 建议 | 说明                                |
| ---------------------------- | ---- | ----------------------------------- |
| All users installation       | 选   | 给所有用户安装                      |
| Use WSL 2 instead of Hyper-V | 必选 | WSL 开发环境推荐                    |
| Allow Windows Containers     | 不选 | 普通 Go/K8s 学习使用 Linux 容器即可 |
| Add shortcut to desktop      | 随意 | 桌面快捷方式                        |

---

## D.2 开启 WSL Integration

Docker Desktop 中进入：

```text
Settings → Resources → WSL Integration
```

打开：

```text
Enable integration with my default WSL distro
Ubuntu
```

然后点击：

```text
Apply & Restart
```

---

## D.3 验证 Docker

在 WSL 中执行：

```bash
docker version
```

如果报：

```text
The command 'docker' could not be found in this WSL 2 distro.
```

一般是 Docker Desktop 没启动，或者 WSL Integration 没打开。

处理：

```powershell
wsl --shutdown
```

然后重新打开 Docker Desktop，并确认 Ubuntu 集成已开启。

---

## D.4 测试 Docker

```bash
docker run hello-world
```

成功：

```text
Hello from Docker!
```

---

## D.5 Docker 常用清理

删除停止的容器：

```bash
docker container prune
```

删除未使用镜像：

```bash
docker image prune
```

查看磁盘占用：

```bash
docker system df
```

谨慎清理所有未使用资源：

```bash
docker system prune
```

---

# 附录 E：kubectl 安装

## E.1 下载 kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

作用：下载当前稳定版本的 kubectl。

---

## E.2 安装 kubectl

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

作用：把 kubectl 安装到系统命令目录。

---

## E.3 验证 kubectl

```bash
kubectl version --client
```

---

## E.4 查看当前上下文

```bash
kubectl config current-context
```

查看所有上下文：

```bash
kubectl config get-contexts
```

切换上下文：

```bash
kubectl config use-context kind-godemo
```

---

# 附录 F：kind 安装与常用命令

## F.1 安装 kind

```bash
go install sigs.k8s.io/kind@v0.31.0
```

配置 PATH：

```bash
echo 'export PATH=$PATH:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc
```

验证：

```bash
kind version
```

---

## F.2 创建集群

```bash
kind create cluster --name godemo
```

查看集群：

```bash
kind get clusters
```

查看节点：

```bash
kubectl get nodes
```

---

## F.3 删除集群

```bash
kind delete cluster --name godemo
```

---

## F.4 加载镜像

```bash
kind load docker-image godemo1:v1 --name godemo
```

作用：把本地 Docker 镜像导入 kind 节点。

---

# 附录 G：Linux 常用命令小抄

## G.1 目录与路径

查看当前目录：

```bash
pwd
```

进入目录：

```bash
cd ~/project/godemo1
```

返回上一级：

```bash
cd ..
```

返回用户主目录：

```bash
cd ~
```

查看目录内容：

```bash
ls
```

详细查看：

```bash
ls -l
```

显示隐藏文件：

```bash
ls -la
```

---

## G.2 ls -l 输出怎么看

示例：

```text
-rw-r--r-- 1 user user 123 May  8 10:00 main.go
drwxr-xr-x 2 user user 4096 May  8 10:00 k8s
lrwxrwxrwx 1 root root 21 May  8 10:00 go -> ../lib/go/bin/go
```

第一位含义：

| 第一位 | 含义     |
| ------ | -------- |
| `-`    | 普通文件 |
| `d`    | 目录     |
| `l`    | 软链接   |

权限部分：

```text
rw-r--r--
```

分三组：

```text
用户权限  用户组权限  其他人权限
rw-       r--        r--
```

---

## G.3 创建、复制、移动、删除

创建目录：

```bash
mkdir k8s
mkdir -p ~/project/godemo1/k8s
```

创建空文件：

```bash
touch .gitignore
```

复制文件：

```bash
cp a.txt b.txt
```

复制目录：

```bash
cp -r dir1 dir2
```

移动或重命名：

```bash
mv old.txt new.txt
```

删除文件：

```bash
rm a.txt
```

删除目录：

```bash
rm -r dir
```

强制删除目录：

```bash
rm -rf dir
```

注意：`rm -rf` 很危险，执行前一定确认路径。

---

## G.4 查看文件内容

查看完整内容：

```bash
cat main.go
```

分页查看：

```bash
less main.go
```

查看前几行：

```bash
head main.go
```

查看后几行：

```bash
tail main.go
```

实时查看日志：

```bash
tail -f app.log
```

---

## G.5 搜索与过滤

在文件中搜索：

```bash
grep ListenAndServe main.go
```

忽略大小写：

```bash
grep -i proxy ~/.bashrc
```

递归搜索目录：

```bash
grep -r "godemo1" .
```

配合管道：

```bash
env | grep -i proxy
docker images | grep godemo1
```

---

## G.6 管道与重定向

管道 `|`：把前一个命令的输出交给后一个命令。

```bash
docker images | grep godemo1
```

覆盖写入文件：

```bash
echo ".idea/" > .gitignore
```

追加写入文件：

```bash
echo "export PATH=$PATH:$HOME/go/bin" >> ~/.bashrc
```

多行写入文件：

```bash
cat > .dockerignore <<'EOF'
.git
.idea/
*.log
EOF
```

---

## G.7 权限与执行

查看权限：

```bash
ls -l
```

添加执行权限：

```bash
chmod +x app
```

运行当前目录程序：

```bash
./app
```

---

## G.8 进程管理

查看进程：

```bash
ps aux
```

搜索进程：

```bash
ps aux | grep go
```

结束进程：

```bash
kill 进程ID
```

强制结束：

```bash
kill -9 进程ID
```

查看后台任务：

```bash
jobs
```

杀掉后台任务：

```bash
kill %1
```

---

## G.9 网络相关

查看监听端口：

```bash
ss -lntp
```

查看 8080 端口：

```bash
ss -lntp | grep 8080
```

测试 HTTP：

```bash
curl http://localhost:8080
```

显示详细请求过程：

```bash
curl -v http://localhost:8080
```

请求不走代理：

```bash
curl --noproxy localhost,127.0.0.1 http://localhost:8080
```

查看 IP：

```bash
ip addr
```

或：

```bash
ip a
```

---

## G.10 环境变量

查看所有环境变量：

```bash
env
```

过滤代理变量：

```bash
env | grep -i proxy
```

临时设置环境变量：

```bash
export no_proxy=localhost,127.0.0.1,::1
```

写入 `.bashrc` 永久生效：

```bash
echo 'export no_proxy=localhost,127.0.0.1,::1' >> ~/.bashrc
source ~/.bashrc
```

---

## G.11 压缩与解压

压缩 zip：

```bash
zip -r project.zip godemo1
```

解压 zip：

```bash
unzip project.zip
```

打 tar.gz 包：

```bash
tar -czf project.tar.gz godemo1
```

解压 tar.gz：

```bash
tar -xzf project.tar.gz
```

---

## G.12 sudo 与 apt

更新软件源：

```bash
sudo apt update
```

安装软件：

```bash
sudo apt install -y git curl vim
```

升级软件：

```bash
sudo apt upgrade -y
```

搜索软件：

```bash
apt search 软件名
```

查看软件是否安装：

```bash
apt list --installed | grep git
```

---

# 附录 H：常用命令速查总表

## H.1 WSL / Windows PowerShell

```powershell
wsl --install
wsl --install -d Ubuntu
wsl -l -v
wsl --set-default-version 2
wsl --set-version Ubuntu 2
wsl --update
wsl --update --web-download
wsl --shutdown
```

---

## H.2 Linux 基础

```bash
pwd
cd ~
cd ..
ls
ls -l
ls -la
mkdir -p dir
touch file
cat file
nano file
vim file
rm file
rm -r dir
cp a b
mv a b
grep keyword file
env | grep -i proxy
ss -lntp | grep 8080
curl http://localhost:8080
```

---

## H.3 Git

```bash
git --version
git config --global user.name "wisexplore"
git config --global user.email "你的GitHub邮箱"
ssh-keygen -t ed25519 -C "你的GitHub邮箱"
cat ~/.ssh/id_ed25519.pub
ssh -T git@github.com
git init
git status
git add .
git commit -m "说明"
git remote -v
git remote add origin git@github.com:wisexplore/godemo1.git
git push -u origin master
git push
git rm -r --cached .idea
```

---

## H.4 Go

```bash
sudo apt install -y golang-go
go version
which go
go env GOROOT
go env GOPATH
go env GOPROXY
go env -w GOPROXY=https://goproxy.cn,direct
go mod init godemo1
go mod tidy
go run main.go
go build -o godemo1
go install sigs.k8s.io/kind@v0.31.0
```

---

## H.5 Docker

```bash
docker version
docker run hello-world
docker images
docker ps
docker ps -a
docker build -t godemo1:v1 .
docker run -d --name godemo1 -p 8080:8080 godemo1:v1
docker logs godemo1
docker exec -it godemo1 sh
docker rm -f godemo1
docker container prune
docker image prune
```

---

## H.6 kind

```bash
kind version
kind create cluster --name godemo
kind get clusters
kind load docker-image godemo1:v1 --name godemo
kind delete cluster --name godemo
```

---

## H.7 kubectl

```bash
kubectl version --client
kubectl config current-context
kubectl config get-contexts
kubectl get nodes
kubectl get deploy
kubectl get rs
kubectl get pods
kubectl get pods -w
kubectl get pods -o wide
kubectl get svc
kubectl get endpoints godemo1-service -o wide
kubectl get endpointslice
kubectl apply -f k8s/
kubectl logs deploy/godemo1
kubectl logs pod名字
kubectl logs pod名字 --previous
kubectl describe pod pod名字
kubectl port-forward svc/godemo1-service 8080:80
kubectl rollout restart deployment/godemo1
kubectl rollout status deployment/godemo1
kubectl rollout undo deployment/godemo1
kubectl delete pod pod名字
```

---

# 附录 I：记忆方法

## I.1 日常开发四步

```bash
git status
git add .
git commit -m "说明"
git push
```

记忆：

```text
看状态 → 加修改 → 做提交 → 推远程
```

---

## I.2 Docker 三步

```bash
docker build -t godemo1:v1 .
docker run -d --name godemo1 -p 8080:8080 godemo1:v1
docker logs godemo1
```

记忆：

```text
构建镜像 → 运行容器 → 看日志
```

---

## I.3 K8s 四步

```bash
kind load docker-image godemo1:v1 --name godemo
kubectl apply -f k8s/
kubectl get pods
kubectl port-forward svc/godemo1-service 8080:80
```

记忆：

```text
导入镜像 → 应用配置 → 看 Pod → 转发访问
```

---

## I.4 排错三件套

Docker 排错：

```bash
docker ps -a
docker logs 容器名
docker images
```

K8s 排错：

```bash
kubectl get pods
kubectl logs pod名字
kubectl describe pod pod名字
```

Linux 网络排错：

```bash
ss -lntp | grep 8080
curl -v http://localhost:8080
env | grep -i proxy
```


---

# 附录 J：补充问答记录

这一部分只补充前文没有展开或容易混淆的问题，便于后续复盘。

## J.1 sidecar 一般用来做什么？

是的，**sidecar 通常用于监控、日志、代理、流量治理等辅助能力**。

sidecar 本质上也是一个容器，只是它不是主业务容器，而是和主容器放在同一个 Pod 里，辅助主容器工作。

常见结构：

```text
Pod
├── app 容器：运行主业务，例如 Go 服务
└── sidecar 容器：运行辅助能力
```

常见用途：

| 用途     | 说明                              |
| -------- | --------------------------------- |
| 日志采集 | 收集 app 容器日志并发送到日志系统 |
| 监控采集 | 采集指标并上报                    |
| 流量代理 | 代理进出流量，例如 Envoy          |
| 服务网格 | 做熔断、限流、灰度、链路追踪      |
| 配置同步 | 动态拉取配置、证书或密钥          |
| 文件辅助 | 共享 volume，辅助主容器处理文件   |

典型例子：

```text
Pod
├── godemo1 容器：业务服务
└── envoy sidecar：代理流量、做观测和治理
```

注意：不要把多个不相关的业务服务塞进一个 Pod。  
sidecar 适合“主容器 + 辅助容器”，不是“多个业务系统混在一起”。

---

## J.2 containerd、sidecar、Pod、服务器是什么 1 对多关系？

简化关系：

```text
Node 服务器
├── containerd
├── Pod A
│   ├── app container
│   └── sidecar container
├── Pod B
│   └── app container
└── Pod C
    ├── app container
    └── sidecar container
```

对应关系：

| 对象 A       | 对象 B      | 关系             |
| ------------ | ----------- | ---------------- |
| K8s 集群     | Node 服务器 | 1 对多           |
| Node 服务器  | containerd  | 通常 1 对 1      |
| Node 服务器  | Pod         | 1 对多           |
| containerd   | 容器        | 1 对多           |
| Pod          | 容器        | 1 对 1 或 1 对多 |
| Pod          | sidecar     | 1 对 0/1/多      |
| Deployment   | Pod 实例    | 1 对多           |
| 公司平台实例 | Pod         | 通常近似 1 对 1  |

关键理解：

```text
containerd 管容器
kubelet 管本机 Pod
K8s 控制面负责调度和期望状态
sidecar 是 Pod 里的辅助容器
```

---

## J.3 公司平台里的“实例”是不是 Pod？

通常可以近似理解为：

```text
公司平台实例 ≈ Pod
```

比如平台配置：

```text
实例数：2
```

底层常见对应：

```yaml
replicas: 2
```

K8s 会创建 2 个 Pod。

但严格来说：

```text
实例是平台/业务口径
Pod 是 Kubernetes 原生对象
```

多数公司平台会把 Pod 包装成“实例”给开发者看。

---

## J.4 公司平台里的“规格”是不是从服务器调度来的？

更准确地说：

```text
不是服务器主动给规格
而是你声明规格，K8s 再根据规格找合适的服务器
```

平台里选择：

```text
1C 2G
2C 4G
```

底层通常转成：

```yaml
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "1"
    memory: "2Gi"
```

含义：

| 字段     | 作用                   |
| -------- | ---------------------- |
| requests | 调度时申请的最低资源   |
| limits   | 运行时最多能使用的资源 |

调度链路：

```text
平台选择实例规格
  ↓
转换成 resources
  ↓
Scheduler 根据 requests 找资源足够的 Node
  ↓
Pod 被调度到某台服务器运行
```

所以：**规格是声明出来的资源需求，调度器根据规格去服务器资源池里找位置。**

---

## J.5 为什么 port-forward 访问 Service 时一直打到同一个 Pod？

现象：

```bash
for i in {1..10}; do curl http://localhost:8080/version; echo; done
```

结果一直返回同一个 hostname。

原因：

```text
kubectl port-forward 更偏调试工具
即使写的是 svc/godemo1-service
它也可能固定转发到 Service 后面的某一个 Pod
```

所以它不适合验证 Service 的负载均衡。

更准确的验证方式是从集群内部访问 Service：

```bash
kubectl run curl-test --image=curlimages/curl -it --rm --restart=Never -- sh
```

进入后：

```sh
for i in $(seq 1 10); do curl http://godemo1-service/version; echo; done
```

如果 hostname 在两个 Pod 之间变化，就说明 Service 后端转发正常。

---

## J.6 endpoints / EndpointSlice 说明了什么？

查看 Service 后端：

```bash
kubectl get endpoints godemo1-service -o wide
```

输出：

```text
godemo1-service   10.244.0.7:8080,10.244.0.8:8080
```

说明：

```text
godemo1-service
  ├── 10.244.0.7:8080
  └── 10.244.0.8:8080
```

也就是 Service 已经选中了两个 Pod。

新版 K8s 会提示：

```text
v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
```

可以用：

```bash
kubectl get endpointslice
```

这不是报错，只是推荐使用新版资源对象。

---

## J.7 curl-test 临时 Pod 怎么退出？

进入临时容器后，提示符类似：

```bash
~ $
```

退出方式：

```bash
exit
```

或：

```text
Ctrl + D
```

因为创建时用了 `--rm`：

```bash
kubectl run curl-test --image=curlimages/curl -it --rm --restart=Never -- sh
```

所以退出后 Pod 会自动删除：

```text
pod "curl-test" deleted from default namespace
```

---

## J.8 为什么 Pod 里显示 hostname 是 Pod 名？

访问：

```bash
curl http://localhost:8080/version
```

返回：

```json
{"hostname":"godemo1-545bbfc47-9f6qs"}
```

这是因为程序里使用了：

```go
os.Hostname()
```

在 K8s 中，容器运行在 Pod 内，默认 hostname 通常就是 Pod 名。  
所以这个字段可以用来观察请求实际打到了哪个 Pod。

---

## J.9 为什么旧版命令行程序在 K8s 里会 CrashLoopBackOff？

旧版程序逻辑：

```text
打印信息
  ↓
程序结束
```

K8s Deployment 期望 Pod 持续运行。  
如果容器主进程退出，K8s 会认为容器结束，然后尝试重启。

循环后就出现：

```text
CrashLoopBackOff
```

适合 Deployment 的服务应该类似：

```go
http.ListenAndServe(":8080", nil)
```

也就是长期运行的 Web 服务。

---

## J.10 这次实践中最重要的概念链路

```text
Go 源码
  ↓
Dockerfile
  ↓
Docker 镜像
  ↓
kind load 导入镜像
  ↓
Deployment 声明副本数
  ↓
ReplicaSet 创建 Pod
  ↓
Pod 里运行容器
  ↓
Service 选择 Pod
  ↓
port-forward / 集群内部访问
```

一句话：

```text
Docker 解决“怎么打包和运行一个容器”
K8s 解决“怎么管理很多容器化应用”
Service 解决“怎么稳定访问一组会变化的 Pod”
```
