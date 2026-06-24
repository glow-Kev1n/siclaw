# Siclaw 部署教程（Docker / K8s 零基础版）

> 目标：从 **DevOps 构建好镜像、拿到 registry 镜像地址** 开始，一步步把 Siclaw
> 部署到开发机能访问的 K8s 集群里，并通过隧道在 **自己电脑的浏览器** 打开它。
>
> 读者假设：你**完全没用过** Docker / Kubernetes / Helm。本文每一步都解释「这是
> 什么、为什么要做」。照着抄就能复现。

---

## 0. 先建立几个最基本的概念（5 分钟）

部署前，先用大白话理解几个词，后面命令才不会像天书：

| 名词 | 一句话理解 | 类比 |
|---|---|---|
| **镜像 (image)** | 把程序和它的运行环境打包成的一个「只读光盘」 | 一张装好系统的安装盘 |
| **Registry** | 存放镜像的「网盘」，集群从这里下载镜像 | App Store |
| **容器 (container)** | 镜像「跑起来」后的一个运行实例 | 用安装盘装出来的一台电脑 |
| **Pod** | K8s 里最小的运行单位，里面跑一个或多个容器 | 一台正在运行的机器 |
| **Deployment** | 「我要长期跑 N 个一样的 Pod」的声明，Pod 挂了它自动拉新的 | 自动续命的看门人 |
| **Service (svc)** | 给一组 Pod 一个固定的内部地址（Pod 的 IP 会变，svc 不变） | 公司总机转分机 |
| **Namespace (ns)** | 集群里的「文件夹」，把不同项目的资源隔开 | 一个独立工作区 |
| **Helm** | K8s 的「安装包管理器」，一条命令装好一堆资源 | apt / brew |
| **Chart** | Helm 的「安装包模板」，里面是参数化的 K8s 配置 | 安装包的源码 |
| **kubectl** | 操作 K8s 集群的命令行工具 | 集群的遥控器 |
| **port-forward** | 把集群里某个服务的端口，临时映射到你能访问的端口 | 临时拉一根网线 |

**Siclaw 跑起来长什么样**（独立部署形态）：

```
namespace (你的工作区, 例: t-k8s-yzi02)
├── Deployment siclaw-portal   → Service siclaw-portal:3003   (Web 前端 + 控制面)
├── Deployment siclaw-runtime  → Service siclaw-runtime:3001  (Agent 运行时大脑)
├── Deployment siclaw-mysql    → Service siclaw-mysql:3306    (测试用数据库)
└── agentbox-* Pod             → 由 runtime 在你发起对话时「动态创建」
```

- **Portal** = 你用浏览器打开的网页，登录、建 agent、配置都在这。
- **Runtime** = 后台大脑，连回 Portal，按需创建 agentbox。
- **agentbox** = 真正干活的 agent，**不是固定 Deployment**，是 runtime 在你聊天时临时起的 Pod。
- 所以你只需要 **3 个镜像**：portal、runtime、agentbox（agentbox 不用手动部署，runtime 会用）。

---

## 1. 你需要从 DevOps 拿到什么

DevOps 构建完会给你 3 个镜像地址，长这样：

```
registry-ap-southeast.scitix.ai/k8s/siclaw-portal:yzi02
registry-ap-southeast.scitix.ai/k8s/siclaw-agentbox:yzi02
registry-ap-southeast.scitix.ai/k8s/siclaw-runtime:yzi02
```

拆开看，每个地址都是 `<Registry 前缀>/siclaw-<组件名>:<tag>`：

| 部分 | 例子 | 含义 |
|---|---|---|
| Registry 前缀 | `registry-ap-southeast.scitix.ai/k8s` | 镜像网盘地址 |
| 组件名 | `portal` / `runtime` / `agentbox` | 三个组件 |
| tag（标签） | `yzi02` | 这一批镜像的版本号 |

> 🔑 **关键**：三个镜像的 **Registry 前缀** 和 **tag** 必须一致。Helm 部署时只填这两个值，
> 它会自动拼出 `前缀/siclaw-portal:tag` 这种完整地址。这是 chart 的命名约定
> （见 `helm/siclaw/templates/_helpers.tpl` 里的 `siclaw.image`）。

记下两个值，后面要用：
- `REGISTRY = registry-ap-southeast.scitix.ai/k8s`
- `TAG = yzi02`

---

## 2. 登录开发机，检查工具齐不齐

集群的访问凭证（kubeconfig）和工具（kubectl/helm）都在开发机上，所以我们在开发机操作。

```bash
# 在你自己的电脑终端，SSH 进开发机
ssh -p 27003 yzi02@dev02.scitix-inner.ai
```

> `-p 27003` 是端口，`yzi02` 是用户名，`dev02.scitix-inner.ai` 是开发机地址。
> 第一次连会问 "Are you sure...?"，输 `yes`。

进去后，检查三样东西（缺哪个找运维装）：

```bash
kubectl version --client      # 集群遥控器
helm version                  # 安装包管理器
ls -l ~/.kube/config          # 集群访问凭证（kubeconfig），有这个才能连集群
```

确认能连上集群、且知道自己在哪个 namespace：

```bash
kubectl get nodes                                    # 能列出节点 = 集群通了
kubectl config current-context                       # 当前用的是哪套集群凭证
kubectl config view --minify -o jsonpath='{..namespace}'; echo   # 当前默认 namespace
```

> 本教程的目标 namespace 是 **`t-k8s-yzi02`**（你自己的工作区）。下面命令都用 `-n t-k8s-yzi02`
> 显式指定 ns，所以默认 ns 是什么不影响。**换成你自己的 ns 名即可。**

设一个变量，后面少打字（**在开发机的终端里执行**）：

```bash
export NS=t-k8s-yzi02
```

---

## 3.（可选）清理同名的旧部署

如果你的 namespace 里之前部署过 siclaw（或别的测试垃圾），先清掉，避免冲突。
**全新 namespace 可跳过本节，直接到第 4 节。**

先看看现在 ns 里有什么：

```bash
helm -n $NS list                      # 有哪些 helm 安装包
kubectl -n $NS get deploy,svc,pod      # 有哪些运行中的资源
```

如果看到旧的 `siclaw` helm release，卸载它：

```bash
helm -n $NS uninstall siclaw
```

> `helm uninstall` 会自动删掉这个安装包创建的 Deployment / Service / Secret 等。
> 但有些资源被标记了「保留」（比如 mTLS 的 CA secret `siclaw-runtime-ca`），需要手动删：

```bash
# 删 helm 没带走的残留（名字按实际情况，--ignore-not-found 表示不存在也不报错）
kubectl -n $NS delete secret siclaw-runtime-ca --ignore-not-found
kubectl -n $NS delete cm siclaw-grafana-dashboard --ignore-not-found
```

> ⚠️ **危险操作提醒**：`uninstall` / `delete` 会真的删数据。删之前务必确认 namespace 没填错、
> 要删的东西确实是废弃的。删生产环境的东西是不可逆的。

确认干净了：

```bash
kubectl -n $NS get all | grep -i siclaw || echo "已清空"
```

---

## 4. 把 Helm Chart 弄到开发机上

镜像在 registry 上，但「怎么部署这些镜像」的说明书（Helm Chart）在 siclaw 代码仓库里。
开发机上需要有这个 Chart。**两种方式选一个：**

### 方式 A：在开发机上直接 git clone（推荐，最简单）

```bash
# 在开发机执行。换成你 fork 的仓库地址和分支
cd ~
git clone https://github.com/glow-Kev1n/siclaw.git siclaw-deploy-src
cd siclaw-deploy-src
git checkout <你构建镜像用的那个分支>     # 比如 chore/deploy-probe-test

# Chart 就在 helm/siclaw 目录
ls helm/siclaw/Chart.yaml helm/siclaw/values-standalone.yaml
```

> 🔑 **重要**：Chart 的版本要和构建镜像的代码**对应同一个分支**，否则配置可能对不上镜像。

### 方式 B：从你自己电脑 scp 上去（我当时用的方式）

如果你本地已经有这个仓库（比如 `~/gpu-cloud/siclaw`）：

```bash
# 在你自己电脑（不是开发机）执行
ssh -p 27003 yzi02@dev02.scitix-inner.ai 'mkdir -p ~/siclaw-deploy'
scp -P 27003 -r ~/gpu-cloud/siclaw/helm yzi02@dev02.scitix-inner.ai:~/siclaw-deploy/
```

> 注意：SSH 用小写 `-p` 指定端口，scp 用大写 `-P`。两个工具不一样，别搞混。

不管哪种方式，记下 Chart 路径（下面叫 `$CHART`）：

```bash
# 在开发机执行，按你的实际路径改
export CHART=~/siclaw-deploy-src/helm/siclaw      # 方式A
# 或 export CHART=~/siclaw-deploy/helm/siclaw      # 方式B

helm lint $CHART        # 检查 chart 语法没问题，看到 "0 chart(s) failed" 就 OK
```

---

## 5. 生成密码和密钥

Siclaw 需要几个密钥（数据库密码、登录 token 签名密钥、Portal↔Runtime 通信密钥）。
用 `openssl` 随机生成，**在开发机执行**：

```bash
export JWT=$(openssl rand -hex 32)        # 登录 token 签名用
export PSEC=$(openssl rand -hex 32)       # Portal 和 Runtime 互相认证用
export MYSQLPW=$(openssl rand -hex 16)    # 测试数据库密码

# 拼出数据库连接串。siclaw-mysql 是数据库的 service 名（chart 自动创建）
export DBURL="mysql://siclaw:${MYSQLPW}@siclaw-mysql:3306/siclaw"

# 打印出来确认一下（生产环境别这么干，这里是测试）
echo "JWT=$JWT"; echo "PSEC=$PSEC"; echo "DBURL=$DBURL"
```

> - `JWT_SECRET`：给登录的「门票」盖章用的印章。
> - `PORTAL_SECRET`：Portal 和 Runtime 对暗号用的，**两边必须填一样的值**，否则 Runtime 连不上 Portal。
> - `DATABASE_URL`：告诉 Portal「数据库在哪、用什么账号密码」。格式：`mysql://用户:密码@地址:端口/库名`。

---

## 6. 一条命令部署（Helm install）

这是核心步骤。一条 `helm` 命令把 portal + runtime + mysql 全装好：

```bash
helm upgrade --install siclaw "$CHART" -n "$NS" \
  -f "$CHART/values-standalone.yaml" \
  --set image.registry="registry-ap-southeast.scitix.ai/k8s" \
  --set image.tag="yzi02" \
  --set mysql.enabled=true \
  --set mysql.password="$MYSQLPW" \
  --set database.url="$DBURL" \
  --set portal.jwtSecret="$JWT" \
  --set runtime.jwtSecret="$JWT" \
  --set portal.portalSecret="$PSEC" \
  --set runtime.portalSecret="$PSEC" \
  --set ocr.enabled=false
```

**逐行解释：**

| 参数 | 作用 |
|---|---|
| `helm upgrade --install siclaw` | 装一个叫 `siclaw` 的 release，已存在就升级、不存在就新装 |
| `"$CHART"` | Chart 在哪个目录 |
| `-n "$NS"` | 装到哪个 namespace |
| `-f .../values-standalone.yaml` | 用「独立部署」的默认配置文件 |
| `--set image.registry=...` | Registry 前缀（第 1 节记的） |
| `--set image.tag=yzi02` | 镜像 tag（第 1 节记的）。chart 自动拼成 `前缀/siclaw-portal:yzi02` 等 |
| `--set mysql.enabled=true` | 顺便起一个测试用 MySQL（**emptyDir，重启丢数据**） |
| `--set mysql.password=...` | 测试 MySQL 的密码 |
| `--set database.url=...` | Portal 连数据库的地址 |
| `--set portal/runtime.jwtSecret` | 登录签名密钥（两个填一样） |
| `--set portal/runtime.portalSecret` | Portal↔Runtime 暗号（两个**必须**一样） |
| `--set ocr.enabled=false` | 关掉 OCR 组件（我们没有 OCR 镜像，不关会拉不到镜像报错） |

看到 `STATUS: deployed` 就说明 Helm 把资源都创建了。

> 💡 **生产/持久化提示**：本教程用的是测试 MySQL（`emptyDir`，Pod 重启数据没了）。
> 要数据不丢，应该用外部 MySQL：去掉 `mysql.enabled` 那两行，把 `database.url` 指向
> 真实数据库地址。

---

## 7. 等 Pod 起来 + 排查启动竞争

部署完不代表马上能用，要等容器下载镜像、启动。查看状态：

```bash
kubectl -n $NS get pods
```

**你大概率会看到这个，别慌：**

```
NAME                             READY   STATUS             RESTARTS   AGE
siclaw-mysql-xxx                 0/1     Running            0          15s
siclaw-portal-xxx                0/1     CrashLoopBackOff   1          15s
siclaw-runtime-xxx               0/1     CrashLoopBackOff   1          15s
```

> 😱 `CrashLoopBackOff` 看着吓人，但这里通常是**正常的启动竞争**，不是真错误：
> - MySQL 还没就绪，Portal 连不上数据库 → 退出重启
> - Portal 还没就绪，Runtime 连不上 Portal → 退出重启（这是 siclaw 的设计：Runtime 必须等 Portal）
>
> K8s 会自动不停重试。**等 20~40 秒**，它们会依次自愈。

等一会再看：

```bash
sleep 30
kubectl -n $NS get pods
```

变成这样就成功了（`READY 1/1`，`STATUS Running`）：

```
NAME                             READY   STATUS    RESTARTS      AGE
siclaw-mysql-xxx                 1/1     Running   0             70s
siclaw-portal-xxx                1/1     Running   2 (63s ago)   70s
siclaw-runtime-xxx               1/1     Running   3 (46s ago)   70s
```

> `RESTARTS` 不是 0 没关系，那是启动竞争时重试留下的计数。只要现在 `READY 1/1` 且
> `RESTARTS` 不再增长，就是稳定了。

**如果等很久还是 CrashLoop**，看日志找真正原因：

```bash
kubectl -n $NS logs deploy/siclaw-portal --tail=30
kubectl -n $NS logs deploy/siclaw-runtime --tail=30
```

常见真错误：
| 现象 | 原因 | 解决 |
|---|---|---|
| `ImagePullBackOff` | 镜像拉不到 | 检查 registry/tag 拼对没、镜像是否 amd64 架构 |
| Portal 一直连不上 DB | `database.url` 填错 / MySQL 没起来 | 看 MySQL pod 状态和密码是否一致 |
| Runtime 报 `Failed to connect to Portal` 且不自愈 | `portalSecret` 两边不一致 | 重新部署，确保两个 portalSecret 相同 |

---

## 8. 验证部署

确认镜像用对了 + 服务都在：

```bash
# 确认三个 Deployment 用的是你的镜像
kubectl -n $NS get deploy -o custom-columns="NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image"

# 确认 runtime 知道用哪个 agentbox 镜像（动态创建 agent pod 时用）
kubectl -n $NS get deploy siclaw-runtime -o jsonpath='{range .spec.template.spec.containers[0].env[*]}{.name}={.value}{"\n"}{end}' | grep AGENTBOX_IMAGE

# 确认 service 都在
kubectl -n $NS get svc | grep siclaw
```

预期能看到 `svc/siclaw-portal` 暴露 3003 端口。

> Pod 能 `READY 1/1`，本身就说明 K8s 的健康检查（探活 `/api/health`）通过了，服务是健康的。

---

## 9. 把 Portal 映射到你自己电脑的浏览器（port-forward + SSH 隧道）

这是最容易绕晕的一步，先看清楚「网络要穿过几层」：

```
你的电脑浏览器
   │  ① SSH 隧道（把本地 13003 → 开发机的 13003）
   ▼
开发机 dev02
   │  ② kubectl port-forward（把开发机 13003 → 集群里 Portal 的 3003）
   ▼
集群里的 siclaw-portal Pod:3003
```

因为 Portal 在集群内部，你电脑直接够不着，要**两跳**：先 SSH 隧道到开发机，再在开发机
用 port-forward 接到 Pod。一条命令同时做这两件事（**在你自己电脑执行，不是开发机**）：

```bash
ssh -p 27003 \
  -o ServerAliveInterval=20 -o ServerAliveCountMax=6 \
  -L 13003:localhost:13003 \
  yzi02@dev02.scitix-inner.ai \
  'kubectl -n t-k8s-yzi02 port-forward svc/siclaw-portal 13003:3003'
```

**逐段解释：**

| 部分 | 作用 |
|---|---|
| `ssh -p 27003 ... yzi02@dev02...` | 连开发机 |
| `-L 13003:localhost:13003` | ① 把你电脑的 13003 端口，转发到开发机的 13003 |
| `-o ServerAliveInterval=20` | 每 20 秒发个心跳，防止隧道空闲被掐断 |
| `'kubectl ... port-forward svc/siclaw-portal 13003:3003'` | ② 在开发机上，把开发机 13003 接到集群里 Portal 的 3003 |

这条命令会**一直占着终端**（这是正常的，它在持续转发）。**不要关这个窗口。**

打开浏览器访问：

```
http://127.0.0.1:13003
```

> 第一次进 Portal 是空数据库，需要先注册/登录账号。

**验证健康**（另开一个终端窗口，在你电脑执行）：

```bash
curl http://127.0.0.1:13003/api/health
# 预期返回: {"status":"ok"}
```

---

## 10. 常见隧道问题（重要，我们踩过的坑）

### 坑 1：隧道用一会就断（Broken pipe）

`kubectl port-forward` 和 SSH 长时间空闲都可能断。表现：浏览器突然连不上。
**解决**：回到那个跑隧道的终端，`Ctrl+C` 停掉，重新跑第 9 节那条命令即可。
（加了 `ServerAliveInterval` 已经能缓解，但不能 100% 避免。）

### 坑 2：重连时报「address already in use」（端口被占）

如果你的隧道命令异常退出，**开发机上的 kubectl port-forward 进程可能没被回收**，
还占着 13003，导致你重连时报：

```
Unable to listen on port 13003: ... bind: address already in use
```

**解决**：登开发机，杀掉你自己残留的 port-forward 进程（**只杀自己的，别动别人的**）：

```bash
ssh -p 27003 yzi02@dev02.scitix-inner.ai
# 在开发机上：
ps -u $USER -o pid,cmd | grep "port-forward.*siclaw-portal" | grep -v grep
# 看到自己的进程后，杀掉（把 <PID> 换成上面列出的数字）
kill -9 <PID>
exit
```

> ⚠️ 用 `ps -u $USER` 只列出**你自己**的进程。如果 `kill` 报 "Operation not permitted"，
> 说明那是**别的同事**的进程（可能他也在用同名服务但不同端口），**不要去杀**，换个端口即可。

### 坑 3：换个端口避开冲突

如果 13003 老是被占，换一个不常用的端口，比如 13017。把第 9 节命令里的
`13003:localhost:13003` 和 `13003:3003` 以及浏览器地址里的端口都改成新的即可。

---

## 11. 更新镜像（DevOps 重新构建后）

改了代码、DevOps 重新构建了镜像，怎么更新部署？

### 情况 A：用了新 tag（推荐）

比如新 tag 叫 `yzi03`，重跑第 6 节的 helm 命令，把 `image.tag` 改成 `yzi03` 即可。
也可以只更新单个组件：

```bash
kubectl -n $NS set image deploy/siclaw-portal  portal=registry-ap-southeast.scitix.ai/k8s/siclaw-portal:yzi03
kubectl -n $NS set image deploy/siclaw-runtime runtime=registry-ap-southeast.scitix.ai/k8s/siclaw-runtime:yzi03
kubectl -n $NS set env   deploy/siclaw-runtime SICLAW_AGENTBOX_IMAGE=registry-ap-southeast.scitix.ai/k8s/siclaw-agentbox:yzi03
```

### 情况 B：tag 没变（还是 yzi02，只是内容重建了）

K8s 看 tag 没变会以为没更新，不会重新拉。需要强制重启来重新拉镜像：

```bash
kubectl -n $NS rollout restart deploy/siclaw-portal deploy/siclaw-runtime
```

> 💡 所以**建议每次构建用不同的 tag**（如带时间戳），省得纠结要不要强制重启。

更新后等 rollout 完成：

```bash
kubectl -n $NS rollout status deploy/siclaw-portal --timeout=180s
kubectl -n $NS rollout status deploy/siclaw-runtime --timeout=180s
```

> 注意：已经在跑的 `agentbox-*` Pod 还是旧镜像。runtime 的 `SICLAW_AGENTBOX_IMAGE` 更新后，
> **新发起的会话**才用新 agentbox 镜像。

---

## 12. 日志和排障速查

```bash
# 看某个组件的日志
kubectl -n $NS logs deploy/siclaw-portal  --tail=100
kubectl -n $NS logs deploy/siclaw-runtime --tail=100

# 看 namespace 最近发生了什么（拉镜像失败、调度失败等都在这）
kubectl -n $NS get events --sort-by=.lastTimestamp | tail -40

# 看某个 pod 的详细情况（为什么起不来）
kubectl -n $NS describe pod <pod名>

# 进容器内部看看（调试用）
kubectl -n $NS exec -it deploy/siclaw-portal -- sh
```

---

## 附录：完整命令清单（复制粘贴版）

```bash
# ===== 在开发机执行 =====
export NS=t-k8s-yzi02
export CHART=~/siclaw-deploy-src/helm/siclaw     # 按实际路径改

# (可选) 清理旧部署
helm -n $NS uninstall siclaw 2>/dev/null
kubectl -n $NS delete secret siclaw-runtime-ca --ignore-not-found

# 生成密钥
export JWT=$(openssl rand -hex 32)
export PSEC=$(openssl rand -hex 32)
export MYSQLPW=$(openssl rand -hex 16)
export DBURL="mysql://siclaw:${MYSQLPW}@siclaw-mysql:3306/siclaw"

# 部署
helm upgrade --install siclaw "$CHART" -n "$NS" \
  -f "$CHART/values-standalone.yaml" \
  --set image.registry="registry-ap-southeast.scitix.ai/k8s" \
  --set image.tag="yzi02" \
  --set mysql.enabled=true --set mysql.password="$MYSQLPW" \
  --set database.url="$DBURL" \
  --set portal.jwtSecret="$JWT"  --set runtime.jwtSecret="$JWT" \
  --set portal.portalSecret="$PSEC" --set runtime.portalSecret="$PSEC" \
  --set ocr.enabled=false

# 等待并验证
sleep 30
kubectl -n $NS get pods

# ===== 在你自己电脑执行（开隧道，会占住终端）=====
ssh -p 27003 -o ServerAliveInterval=20 -o ServerAliveCountMax=6 \
  -L 13003:localhost:13003 yzi02@dev02.scitix-inner.ai \
  'kubectl -n t-k8s-yzi02 port-forward svc/siclaw-portal 13003:3003'

# 浏览器打开 http://127.0.0.1:13003
```
