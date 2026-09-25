# Google Agent Substrate 系统调研报告

> 调研对象：[agent-substrate/substrate](https://github.com/agent-substrate/substrate)（Apache-2.0，Go 编写）
> 关联项目：[google/ax](https://github.com/google/ax)（上层编排器 Agent Executor）
> 调研日期：2026-09-26
> 目标：系统理解 Substrate 的设计动机、架构、核心机制、源码结构，以及它与 K8s、AX、各类 Agent 框架的关系。

---

## 目录

1. [一句话概括](#1-一句话概括)
2. [它要解决什么问题](#2-它要解决什么问题)
3. [核心概念](#3-核心概念)
4. [整体架构](#4-整体架构)
5. [核心机制深入](#5-核心机制深入)
6. [性能指标](#6-性能指标)
7. [与 AX（Agent Executor）的关系](#7-与-axagent-executor-的关系)
8. [生态兼容性](#8-生态兼容性)
9. [源码结构导读](#9-源码结构导读)
10. [与同类系统对比](#10-与同类系统对比)
11. [学习路径建议](#11-学习路径建议)
12. [参考资料](#12-参考资料)

---

## 1. 一句话概括

**Agent Substrate 是 Google 开源的、跑在 Kubernetes 之上的"Agent 时代的虚拟内存 + 进程调度器"**：它把成千上万个长时运行、大部分时间在空闲的 Agent（或任何"长空闲 + 突发活跃 + 需要保留状态"的进程）抽象成 **Actor**，通过快照（checkpoint）挂起和 sub-second 恢复（restore），把它们高密度地复用到一小撮预先预热好的 **Worker** Pod 上，配合一层 Agent 感知的路由网关（atenet），让调用方完全不用关心 Agent 当前是否在内存中。

- 它**不是** Agent 开发 SDK，**不是** K8s 替代品，**不是** serverless 函数框架；
- 它是**运行时底座**：K8s 继续管节点、Worker Pod 生命周期、自动伸缩；Substrate 自己把高频的 suspend/resume 和 actor 调度从 K8s API server 的关键路径上摘掉。

官方公布的硬指标：**sub-500ms 恢复、500+ 次 suspend/resume/秒、单机 1000+ dormant actor、相对传统容器 10x 密度**。

---

## 2. 它要解决什么问题

### 2.1 Agent 负载的形态

现代 Agent（Claude Code、Codex、Hermes、MCP server、RL rollout 沙箱……）的运行模式和传统微服务根本不同：

| 特征 | 传统微服务 | Agent 负载 |
|---|---|---|
| 生命周期 | 长驻，请求-响应 | 长时存活，但**绝大多数时间在等**（等模型推理、等工具返回、等人类确认） |
| 每次活跃时长 | ms ~ 秒 | ms ~ 几十秒 |
| 隔离要求 | 较低，多副本共享 | **极高**——模型生成的是从未被人看过的不可信代码 |
| 实例数量 | 几十~几千 | **百万~十亿级** actor |
| 状态 | 无状态或少状态 | **有状态**：文件系统、进程内存、会话上下文都要保留 |

### 2.2 用 K8s 直接跑会遇到什么

设计文档 `docs/architecture.md` 列得很直白：

1. **空闲 Pod 也吃资源**——给每个 Agent 留一个 Pod，CPU/内存/IP 全浪费；
2. **K8s API server 不是为百万资源设计的**——它擅长异步调和，不擅长存海量离散对象、扛每秒上万次写；
3. **Pod 启动延迟秒级**——对要跑几毫秒就再次进入等待的工作负载不可接受；
4. **PV 不适合百万级卷高频挂载/卸载**。

### 2.3 Substrate 的回答

> 把"机器管理"留给 K8s，把"actor 生命周期"留给自己。

- **预热 Worker 池**：提前起好一批 sandbox Pod，空转等活；
- **挂起即快照**：actor 空闲时把整个进程内存 + 可写层磁盘打成 snapshot 写到 GCS/S3，把 Worker 让出来；
- **请求即唤醒**：来流量时路由层先把 actor 从 snapshot 恢复到某个空闲 Worker，再把请求转过去；
- **安全默认**：gVisor 或 microVM 内核级隔离 + egress 默认全拒 + 凭证不进沙箱。

这本质上是把操作系统里"交换分区 + 调度器"的思路搬到了集群层面：Actor 是进程，Worker 是 CPU 核，快照就是换页文件。

---

## 3. 核心概念

| 概念 | 是什么 | 类比 |
|---|---|---|
| **Actor** | 一个具体的应用实例（一个 Agent、一个 shell 沙箱、一个 MCP server）。状态：SUSPENDED / RESUMING / RUNNING / SUSPENDING / PAUSED / CRASHED | 进程 |
| **Atespace** | Actor 的分组单位，类似 K8s namespace；actor 地址是 `<atespace>/<actor>` | namespace |
| **ActorTemplate** | 不可变的 actor 镜像/配置定义（OCI 镜像、资源、环境变量、sandbox 配置），首次实例化时基于它生成 **golden snapshot** | 容器镜像 |
| **WorkerPool** | K8s CRD，定义一组预热好的 Worker Pod（节点选择、toleration、sandbox class） | CPU 核池 |
| **Worker** | 一个真正在跑的 Pod，IDLE/BUSY，当前托管 0 或 1 个 actor | CPU 核 |
| **Sandbox class** | 隔离技术选型：`gVisor`（默认）或 `microVM`（Kata + Cloud Hypervisor） | 隔离级别 |
| **Snapshot** | actor 的内存 + 可写层磁盘的版本化镜像，存 GCS/S3；可打 tag 做不可变别名 | 交换页 |

资源分两类存：
- **系统配置（声明式，走 K8s）**：WorkerPool、SandboxConfig；
- **动态实例状态（高频，走 PostgreSQL）**：Actor、Worker 记录。

这种"双层"是为了不让每秒上千次的 actor 状态变更打爆 K8s etcd。

---

## 4. 整体架构

```
                          ┌─────────────────────────────────────┐
                          │         调用方 / AX / 框架          │
                          └──────────────┬──────────────────────┘
                                         │ HTTP / gRPC
                                         │ header: ate-target-actor: <atespace>/<actor>
                                         ▼
                          ┌─────────────────────────────┐
                          │   atenet-router (Envoy +     │
                          │   ext_proc)  ← 数据面入口     │
                          │   - 解析 actor，触发 resume   │
                          │   - Request parking (1024)   │
                          └──────┬───────────────┬───────┘
                  gRPC ResumeActor│               │ mTLS tunnel :443
                                 ▼               ▼
                ┌────────────────────────┐  ┌──────────────────────────┐
                │   ate-api-server        │  │  Worker Pod (warm)        │
                │   (控制面，PostgreSQL)   │  │  ┌────────────────────┐  │
                │   - Scheduler           │  │  │ atunnel :443       │  │
                │   - Workflow engine     │  │  │  (TLS 终结/转发)    │  │
                │   - 凭证颁发             │  │  └─────────┬──────────┘  │
                └────┬───────────┬───────┘  │  ┌─────────▼──────────┐  │
                     │           │          │  │ ateom-gvisor /      │  │
                     │           │ snapshot │  │ ateom-microvm       │  │
                     │           ▼          │  │  - runsc checkpoint │  │
                     │   ┌────────────┐     │  │  - Cloud Hypervisor │  │
                     │   │ GCS / S3   │     │  │    memory snapshot  │  │
                     │   │ (快照存储)  │     │  └─────────┬──────────┘  │
                     │   └────────────┘     │  ┌─────────▼──────────┐  │
                     │                      │  │  Actor (OCI 容器)   │  │
                     │                      │  │  私有 veth          │  │
                     ▼                      │  └────────────────────┘  │
              ┌──────────────┐               └──────────────────────────┘
              │ atelet      │  每个节点一个 DaemonSet
              │ (节点监督)    │  - 调 ateom Restore/Checkpoint
              │             │  - 流式上传/下载 snapshot
              └──────────────┘
                     ▲
                     │ K8s 管：节点、Worker Pod 伸缩、自愈
              ┌──────┴───────┐
              │  atecontroller │  调和 WorkerPool CRD → Deployment
              └───────────────┘
```

### 4.1 组件清单（`cmd/` 下的二进制）

| 二进制 | 职责 |
|---|---|
| `ateapi` | 控制面 gRPC API server，PostgreSQL 存 actor/worker 状态，内置 scheduler 和 workflow engine |
| `atelet` | 节点级 DaemonSet，"牧羊人"，监督本机 Worker Pod、协调 snapshot 上传/下载、调 ateom |
| `atecontroller` | K8s controller，调和 WorkerPool CRD → Deployment/HPA |
| `atenet` | 数据面网络：Envoy 路由 + ext_proc + proxy sidecar |
| `ateom-gvisor` | Worker Pod 内的 sandbox herder，跑 `runsc` checkpoint/restore |
| `ateom-microvm` | 同上，跑 Kata + Cloud Hypervisor，memory-only snapshot + userfaultfd demand paging |
| `credential-provider` | 凭证注入网关（见 §5.5） |
| `podcertcontroller` | Pod 证书签发（上游 K8s 还没有的 polyfill） |
| `kubectl-ate` | kubectl 插件，管理 actor/template/workerpool |

### 4.2 一次请求的完整时序（来自 architecture.md）

```
Client → atenet-router: HTTP 请求，带 ate-target-actor: foo/bar
atenet-router → ate-api-server: ResumeActor(foo, bar)
ate-api-server → atelet: Restore（挑一个空闲 Worker）
Store(GCS) ──快照下载──► atelet
atelet → ateom: RestoreWorkload
ateom: 恢复 sandbox，激活 actor 网络栈
ateom → atelet: ready
atelet → ate-api-server: 返回 Worker Pod IP
ate-api-server → atenet-router: 返回分配
atenet-router → atunnel(Worker:443): mTLS 隧道
atunnel → Actor: 转发原始请求
Actor → atunnel → atenet-router → Client: 响应
...actor 空闲一段时间后，SuspendActor 把内存+磁盘 checkpoint 回 GCS，Worker 归还池
```

关键：**路由层直接调控制面做 resume，不经过 K8s scheduler**，这是 sub-100ms 激活延迟的根本原因。

---

## 5. 核心机制深入

### 5.1 两种沙箱后端：gVisor vs microVM

Substrate 不绑死一种隔离技术，`WorkerPool.spec.sandboxClass` 二选一：

**gVisor（默认，`ateom-gvisor`）**
- 用 `runsc` 在用户态拦截 syscall；
- suspend/resume 直接复用 gVisor 原生的 checkpoint/restore（CRIU 思路）；
- 开销低，但系统调用兼容面有边界。

**microVM（`ateom-microvm`）**
- 跑在 Kata + Cloud Hypervisor 里，完整 Linux 内核；
- snapshot 只存 **memory-only VM snapshot**，恢复时用 `userfaultfd` 做按需分页（demand paging）——不必把整个 VM 镜像加载完就能跑；
- 容器 rootfs 由 host 组装（只读 OCI lower + per-actor 可写 upper），通过一条 virtio-fs 共享给 guest，写操作走 host page cache，不占 guest RAM；
- `DurableDir` 卷也走这条共享通道，作为 tar 随 snapshot 一起传；microVM class 因此取消了 gVisor 下"只能挂一个 DurableDir"的限制。

> 工程细节：gVisor 后端目前需要带 `--allow-connect-on-save` 补丁的 runsc，绕开 checkpoint 时网络恢复的 bug。

### 5.2 快照与状态管理

- **两类状态**绑在同一个版本化 snapshot 里：进程内存 + 容器可写层磁盘（"working memory"）；
- snapshot 上传到 **GCS**（本地 kind 开发用 rustfs，也兼容 S3）；
- 一个 actor 同一时刻只持有一份 external snapshot；suspend 时新 snapshot 写完才删旧的；
- snapshot 可以打 **tag**：不可变别名 + 保留 pin，跨 atespace 复用；
- 恢复严格绑定 ActorTemplate 版本——snapshot manifest 里钉住 sandbox 二进制 digest，升级运行时也能恢复老 snapshot。

Actor 状态机：

```
[*] --CreateActor--> SUSPENDED
SUSPENDED --ResumeActor--> RESUMING --> RUNNING
RUNNING --SuspendActor--> SUSPENDING --> SUSPENDED
RUNNING --PauseActor--> PAUSING --> PAUSED（节点本地 checkpoint，不跨节点）
PAUSED --ResumeActor--> RESUMING（钉在原节点）
PAUSED --SuspendActor--> SUSPENDING（上传节点本地 snapshot）
RUNNING/PAUSED/CRASHED --RevertActor--> SUSPENDED（丢弃进行中的状态，回到上次成功 snapshot）
SUSPENDED/CRASHED --DeleteActor--> [*]（GC 掉 snapshot）
```

### 5.3 路由层 atenet 与 `ate-target-actor`

- 入口是 `atenet-router`，跑 Envoy，挂一个 `ext_proc`（external processor）；
- 调用方发 HTTP/gRPC，**唯一寻址方式**是请求头：
  ```
  ate-target-actor: <atespace>/<actor>
  ```
  `Host` 头仍然由应用自己决定，不参与路由；
- ext_proc 截到请求后调 `ate-api-server.ResumeActor`，拿到当前 Worker IP；
- 然后 atenet-router 和 Worker 内的 `atunnel:443` 建一条 **mTLS 隧道**，原始请求通过隧道送到 Actor 的私有 veth；
- Worker Pod 的 80 端口**不是**直接入口——所有流量必须过隧道，这样 actor 漂移到别的 Worker 时路由自动跟着变；
- 非 80 端口访问走 HTTP `CONNECT <actor-dns>:9090`，路由在同一个隧道里重新进入，长连接里每个请求独立 resume/re-route。

### 5.4 Request Parking（请求暂存）

这是用户特别提到的"资源池满了不返回 503"的实现，文档 `docs/request-parking.md` 写得非常细：

**触发条件**：`ResumeActor` 返回以下可重试错误时，请求不直接 503，而是 park 住指数退避重试：
- `ResourceExhausted`（池里没空闲 Worker）
- `FailedPrecondition`（actor 处于瞬态）
- `Unavailable`（控制面在滚动重启）
- `Aborted`（并发 resume 冲突）

**硬参数**：
- `--parked-request-max`：parking lot 容量，**默认 1024**；lot 满了才返回 `503 "router at capacity"`；
- `--parked-request-budget`：等待预算，**默认 5s**；超时仍失败则返回 503；
- Envoy ext_proc 消息超时 = budget + 5s，是客户端被挂住的硬上限；
- ext_proc 集群熔断值默认是 `parked-request-max × 2`（下限 1024），保证 lot 满时不会饿死已经 RUNNING 的 actor 的快路径请求。

**两个聪明的设计**：
1. **同 actor 请求去重**：同一个 actor 的并发请求共享一个 in-flight `ResumeActor`（per-actor flight registry），N 个请求占 N 个 parking slot，但只产生 1 次控制面 RPC；
2. **预算是 per-flight 而不是 per-request**：后到的请求共享前面那个 flight 的剩余预算，晚到的请求可能只等了 1 秒就看到 `budget_exhausted`——这是把热点 actor 折叠成一次控制面调用的代价。

**不会被 park 的错误**（fail fast）：`NotFound→404`、`DeadlineExceeded→504`、`PermissionDenied→403`、`Unauthenticated→401`。

### 5.5 网络隔离与出网策略（egress）

`docs/egress-traffic.md` 规定了 GA 版本支持的出网行为：

- Actor 的 TCP 出向流量（除 53 DNS）被重定向到 atunnel，atunnel 向 egress gateway 发 CONNECT，由 gateway 按策略放行；
- DNS-over-TCP、UDP、其他协议由 **nftables** 直接过滤，根本到不了 gateway。

| 协议 | 端口 | 行为 |
|---|---|---|
| HTTP(S) 1.1 / 2 | 任意 | **支持**，按主机名/端口 allowlist 放行，拒绝回 `403` |
| WebSocket | 任意 | **阻断**，回 403 |
| HTTP CONNECT 转发代理 | 任意 | **阻断**，回 403 |
| DNS | 53 | 允许（nftables → 节点 DNS） |
| 其他 TCP | 任意 | **接受连接后立即关闭**，不返回任何字节 |
| 其他 UDP | 任意 | **静默丢包**（不发 ICMP port-unreachable，客户端挂到自己超时） |
| 非 TCP/UDP | — | 静默丢包 |

也就是说：**默认全拒，只放 HTTP/HTTPS，按主机名+端口白名单**；WebSocket、原始 TCP、UDP 一律不行。这正好对应"Agent 不应该能随便连数据库、连内部服务、起反向 shell"的安全目标。

要做 header 注入这类 egress 改写，还得在 Actor 里开 MITM 信任包（`docs/egress-trust-bundle.md`），由 gateway 终止 TLS、代持证书。

### 5.6 凭证注入与防 Prompt 注入

威胁模型 `docs/threat-model.md` 里 **T-29** 是核心：

> *"Agent leaks credentials exposed in sandbox, because LLMs are unreliable. Due to prompt injection or just agent silliness."*

对策（`cmd/credential-provider` + `internal/credbundle`）：

- **默认不在沙箱里放任何凭证**；
- 需要凭证时走 **credential injecting proxy**：gateway 终止 TLS、代持 x509 私钥，token 在出网请求时才注入；
- Secrets 走 K8s Secret 官方路径，env 或 tmpfs（内存文件系统）注入，落盘即清零；
- snapshot 访问用独立的、只发给 atelet 的 JWT，带 audience claim，Actor 自己拿到也用不了（T-23/T-24/T-32）；
- snapshot 要加密、要签名校验 digest（T-25）；
- Worker 复用时必须彻底清掉上一个 actor 的进程/文件/env/网络策略残留（T-27），还在边界上布蜜罐检测状态泄露。

这整套思路就是"零信任 + blast radius 最小化"：Agent 写的代码能跑，但碰不到密钥、扫不到内网、连不上除白名单外的任何地址。

---

## 6. 性能指标

**North Star（设计目标，`docs/architecture.md`）**：

| 指标 | 目标 |
|---|---|
| 激活延迟（wakeup → actor 可收流量） | p99 **100ms** |
| 单集群 actor 总数（活跃+挂起） | **10 亿** |
| 单集群 wakeup 事件吞吐 | **1000/秒** |

**官方已公布实测（2026-09-16 GKE GA 公告）**：

- sub-500ms resume；
- 500+ suspend/resume activations/秒；
- 单 Worker 上 **1000+ dormant actor**；
- 相对传统容器运行时 **10x 密度**；
- Counter demo：8 个物理 Pod 多路复用 **250 个有状态 actor**，30x+ oversubscription。

优化点：
- snapshot 写本地盘 + GCS 双写，恢复时就近取；
- microVM 用 userfaultfd demand paging，VM 不整个加载完就能服务；
- 同 actor 并发请求折叠成一次控制面 RPC；
- Worker 池预热，永远不现场拉起 Pod。

---

## 7. 与 AX（Agent Executor）的关系

[google/ax](https://github.com/google/ax)（Apache-2.0，官网 agentexecutor.io）是 Google 在 2026-09 开源的**上层编排器**，跑在 Substrate 之上。它把 agent 当有状态 actor 而不是微服务/batch job，提供四个声明式 CRD（`ax.io/v1alpha1`）：

| CRD | 作用 |
|---|---|
| **Task** | 一次 agent 执行的生命周期、资源限制、sandbox 约束 |
| **Workspace** | 执行前环境组装：挂 Git repo、配 MCP server、装 skill bundle，甚至写一段自然语言 goal 让初始化 agent 自己装工具链 |
| **Gateway** | 出网策略 allowlist + 凭证注入 |
| **Model** | LLM provider 参数、secrets 的统一控制点，滚动升级模型版本 |

CLI 是 Go 写的 `ax`：`ax apply` / `ax watch` / `ax ssh` / `ax suspend` / `ax resume`。控制面用 ko 部署到 `ax-system` namespace，依赖 Redis。

**分层关系**：

```
┌─────────────────────────────────────┐
│  你的 Agent 应用 / ADK / LangChain   │
├─────────────────────────────────────┤
│  AX (google/ax)                     │  ← 声明式编排：Task/Workspace/Gateway/Model
├─────────────────────────────────────┤
│  Agent Substrate (agent-substrate)  │  ← 运行时：suspend/resume、路由、隔离、凭证
├─────────────────────────────────────┤
│  Kubernetes (GKE)                   │  ← 节点、Pod、自动伸缩、自愈
└─────────────────────────────────────┘
```

AX 论文/博客里有个很形象的比喻：传统 Pod 是"每个 agent 一个大容器，浪费空间"；AX actor 是"几十个唯一的、小巧的 axolotl（六角恐龙，AX 的吉祥物）住在同一个 microVM 里"。

---

## 8. 生态兼容性

Substrate 是**框架无关**的——它在 gVisor 内核级跑标准 OCI 容器，所以任何能容器化的 agent harness 都能上：

- **Google ADK**（Agent Development Kit）：session state 作为 actor state 跨调用保留；
- **LangChain / LangGraph**：agent 和 tool call 的执行环境；
- **Claude Code / Codex / Antigravity**：高密度有状态编码环境，文件系统和系统状态跨会话保留（官方有 `demos/claude-code-multiplex`）；
- **MCP**：把 MCP server 作为 durable actor 部署，给任意模型当持久化工具；
- **kagent**（CNCF Sandbox 项目）：K8s 原生 agent 框架，直接用 Substrate 跑沙箱化工作负载；
- **Nous Research Hermes**：早期设计伙伴，OpenRouter 上用量第一的 agent，已在生产上验证 per-agent 隔离和资源复用。

---

## 9. 源码结构导读

Go 项目，根目录即 Go module：

```
substrate/
├── cmd/                        # 可执行入口
│   ├── ateapi/                 #   控制面 gRPC server
│   ├── atelet/                 #   节点 DaemonSet
│   ├── atecontroller/          #   WorkerPool CRD controller
│   ├── atenet/                 #   路由/代理
│   ├── ateom-gvisor/            #   gVisor herder
│   ├── ateom-microvm/           #   microVM herder
│   ├── credential-provider/    #   凭证注入
│   ├── podcertcontroller/      #   Pod 证书
│   ├── kubectl-ate/             #   kubectl 插件
│   ├── ate-setup/              #   安装工具
│   └── benchmarking/           #   压测（glutton 等合成负载）
├── internal/
│   ├── atelet/  ateom*/        # 节点/sandbox 管理
│   ├── atenet/                 # 路由、ext_proc、parking lot
│   ├── egresspolicy/           # 出网策略
│   ├── objectstore/            # GCS/S3/rustfs 抽象
│   ├── actoridjwt/ localjwtauthority/ substratex509/  # 身份/证书
│   ├── credbundle/  ateapiauth/ # 凭证包、控制面鉴权
│   ├── atunnel/                # worker 内 TLS 隧道
│   ├── k8sresolver/  resources/  volume/  volume path/
│   ├── sizing/  deviceplugin/  cdi/
│   ├── wakeupprobe/  actorlock/  actorevent/  actorlog/
│   └── otlprelay/  observability 相关
├── pkg/
│   ├── api/                    # Go API 类型
│   ├── client/                 # Go client SDK
│   └── proto/                  # ateapi protobuf 定义
├── docs/                       # 架构/API/安全/运维文档（本报告主要依据）
├── demos/                      # counter / sandbox / claude-code-multiplex /
│                               # multi-template / parking / autoscaled-workerpool
├── manifests/                  # K8s 部署 manifest
├── hack/                       # kind 集群、安装脚本
└── tools/setup-gcp/            # GKE/GCS/IAM 一键拉起
```

**本地跑起来**（官方 quickstart）：

```bash
hack/create-kind-cluster.sh
hack/install-ate-kind.sh --deploy-ate-system
hack/install-ate-kind.sh --deploy-demo-counter
go install ./cmd/kubectl-ate
kubectl ate create actor my-counter-1 -a ate-demo-counter --template counter
kubectl port-forward -n ate-system svc/atenet-router 8000:80
curl -X POST -H "ate-target-actor: ate-demo-counter/my-counter-1" http://localhost:8000/
```

---

## 10. 与同类系统对比

| 维度 | Agent Substrate | 普通 K8s Pod | AWS Fargate/Lambda | Kata Containers | Ray Serve |
|---|---|---|---|---|---|
| 调度单位 | Actor（可挂起） | Pod（长驻） | 函数/任务 | Pod | actor/actor handle |
| 空闲时 | **快照到 GCS，资源全释放** | 占着 CPU/内存 | 缩到 0，但冷启动慢 | 占着 | 占着 |
| 恢复延迟 | **<500ms（目标 100ms）** | 秒级 | 冷启动百 ms~秒 | 秒级 | 秒级 |
| 隔离 | gVisor 或 microVM | 共享内核 | VM/Fargate microVM | Kata microVM | 进程级 |
| 状态 | 内存+磁盘 snapshot | PV | 外部存储 | PV | 对象存储 |
| 路由 | agent-aware，自动 resume | Service/Ingress | URL path | Service | Ray router |
| 出网安全 | 默认全拒，HTTP allowlist | NetworkPolicy | SG | NetworkPolicy | 自有 |
| 适合 | **百万级长空闲有状态 agent** | 普通微服务 | 短请求/事件 | 安全敏感通用负载 | Python/ML 服务 |

**JuiceFS/3FS 那类分布式 FS 解决的是"数据怎么存"，Substrate 解决的是"进程怎么跑"**——两者不冲突，Substrate 的 Workspace 可以挂 NFS/Filestore，MCP server 的数据照样放在 3FS 上。

---

## 11. 学习路径建议

### 阶段 1：建立心智模型（0.5 天）
- 读这份报告 + `docs/architecture.md`（它自己开头就写"很多还是 aspirational"，要看懂目标态）；
- 看官方 demo 视频（README 里的 YouTube），重点理解"250 actor 跑在 8 Pod 上"这个 30x 复用；
- 能画出：调用方 → atenet → ate-api-server → atelet → ateom → Actor → GCS 这条链。

### 阶段 2：本地跑起来（半天）
- 装 Go、kubectl、docker、kind；
- 跑 `hack/create-kind-cluster.sh` + `hack/install-ate-kind.sh --deploy-ate-system`；
- 起 counter demo，用 `kubectl ate create actor` 建几个 actor，curl 打过去，观察：
  - 第一次请求慢（restore）；
  - 第二次快（已经 RUNNING）；
  - 停一会再打，又慢（已被 suspend）；
- 故意把 WorkerPool 缩到 1，打并发，看 parking lot 行为（1024 / 5s / 指数退避）。

### 阶段 3：读关键源码（2-3 天）
按这个顺序：
1. `pkg/proto/` + `pkg/api/`：先看资源模型；
2. `cmd/atenet/` + `internal/atenet/`：路由、ext_proc、parking lot；
3. `cmd/ateapi/` + `internal/` 里 workflow engine：ResumeActor/SuspendActor 的步骤编排；
4. `cmd/atelet/` + `internal/ateomsuspend/`：快照怎么落 GCS；
5. `cmd/ateom-gvisor/` 和 `cmd/ateom-microvm/`：两种 sandbox 的 restore 差异；
6. `internal/egresspolicy/` + `cmd/credential-provider/`：出网和凭证。

### 阶段 4：对照安全模型（半天）
- 读 `docs/threat-model.md`（T-01 ~ T-40 那张表非常有料）；
- 重点看 T-23/24/25/27/29/32/36 这几个 Critical/High，理解"为什么 actor 不能直接碰 snapshot"。

### 延伸方向
- AX（google/ax）怎么把 Substrate 的原生 API 包装成 Task/Workspace/Gateway/Model 四个 CRD；
- microVM 的 userfaultfd demand paging 实现；
- 同 actor 请求折叠（per-actor flight registry）的一致性细节；
- Filestore agent volume（毫秒级挂/卸 NFS，RWX + POSIX lock，多 agent 协作）。

---

## 12. 参考资料

- 核心仓库：<https://github.com/agent-substrate/substrate>
- 架构文档：<https://github.com/agent-substrate/substrate/blob/main/docs/architecture.md>
- Request Parking：<https://github.com/agent-substrate/substrate/blob/main/docs/request-parking.md>
- Egress Traffic：<https://github.com/agent-substrate/substrate/blob/main/docs/egress-traffic.md>
- 威胁模型：<https://github.com/agent-substrate/substrate/blob/main/docs/threat-model.md>
- Google Cloud 官方博客（2026-09-16 GA）：<https://cloud.google.com/blog/products/containers-kubernetes/agent-substrate-available-on-gke>
- Google Cloud 早期介绍（2026-05-21）：<https://cloud.google.com/blog/products/containers-kubernetes/bringing-you-agent-sandbox-on-gke-and-agent-substrate>
- Agent Executor 博客：<https://cloud.google.com/blog/products/ai-machine-learning/agent-executor-googles-distributed-agent-runtime/>
- AX 仓库：<https://github.com/google/ax>
- InfoQ 对 AX 的报道：<https://www.infoq.com/news/2026/09/google-ax-orchestrator/>
- gVisor：<https://gvisor.dev>；Cloud Hypervisor：<https://www.cloudhypervisor.org>；Kata Containers：<https://katacontainers.io>

---

*本报告基于 2026-09-26 当天 clone 的 agent-substrate/substrate main 分支整理。项目 pre-1.0，API 和行为仍在快速演进，以上游仓库为准。*
