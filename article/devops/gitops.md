# GitOps 实战 —— ArgoCD / Flux / 多集群部署的工程化

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想速记，看 [DevOps CI/CD](/article/devops/cicd) 的部署一节够了。
> 这里要回答的，是**你已经把 ArgoCD 装上了、跑了几个 Application 之后才会撞到的问题** —— 多集群、密钥、同步漂移、紧急修复的"绕过"模式。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. GitOps 的"4 个原则"是哪 4 个？为什么"声明式"是其他三条的前提，而不是并列关系？
2. Pull 模型 vs Push 模型 —— 给一个**安全角度**的差别和一个**一致性角度**的差别。为什么 ArgoCD/Flux 都选 Pull？
3. ArgoCD 里 `Application` 和 `AppProject` 各管什么？为什么生产环境**不能**让所有 Application 都用 default Project？
4. `ApplicationSet` 和 `Application` 是什么关系？Cluster Generator 和 Git Generator 各解决什么问题？
5. ArgoCD 和 Flux v2 在"**同步谁拥有 manifest**"这件事上设计差别是什么？哪个更适合 platform team / 哪个更适合 app team？
6. Helm + Kustomize + GitOps 三者怎么协作？为什么 ArgoCD 推荐 "render-then-commit" 模式而不是直接渲染？
7. Sealed Secrets 和 SOPS 解决的是同一个问题吗？密钥泄漏后哪个更容易轮换？
8. 多集群 GitOps 部署里，**控制平面集群**自己怎么 GitOps 管理？"app of apps" 模式和 ApplicationSet 的边界在哪？

如果有半数答不上，往下看。

---

## 1. 底层模型：声明式、Pull、调和循环

绝大多数 GitOps 文章止于"把 YAML 推到 Git，ArgoCD 自动同步"。但这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到 GitOps 的几个核心抽象。

### 1.1 GitOps 的 4 个原则

Weaveworks 在 2017 年提出 GitOps 时，给了 4 条形式化原则。后来 OpenGitOps 1.0 把它们写成规范。先看原文意图：

| 原则 | 一句话定义 | 反面是什么 |
| --- | --- | --- |
| **Declarative**（声明式） | 整个系统**最终状态**用声明式数据描述 | 命令式脚本（`kubectl create`、Ansible playbook） |
| **Versioned & Immutable**（版本化 & 不可变） | 期望状态存在 Git，**所有变更都是 commit** | 数据库里存配置、控制台里改 |
| **Pulled Automatically**（自动拉取） | Agent 自动从 Git 拉，而不是被 push | CI 流水线 `kubectl apply` |
| **Continuously Reconciled**（持续调和） | Agent 持续比对实际 vs 期望，**漂移自动纠正** | 一次性部署后不再管 |

四条不是平等的。**声明式是其他三条的前提**：
- 不声明式 → 没有"最终状态"概念 → 没办法版本化（diff 没意义）
- 不版本化 → Pull 没东西可拉
- 不 Pull → Agent 不知道源在哪
- 不调和 → Pull 一次后偏移仍然无法纠正

读者最容易混淆的是"声明式 vs 幂等"。Kubernetes manifest 是声明式的；Ansible playbook 是**幂等的命令式**。两者都能"重复跑不出错"，但只有声明式能 diff —— 这是 GitOps 工具能"自动调和"的根本。

> **口诀**：4 条原则不是并列，是**声明式 → 版本化 → Pull → 持续调和**的链条。断哪一环，剩下的都不成立。

### 1.2 Pull 模型 vs Push 模型

CI/CD 时代主流是 Push：CI 服务器拿到 kube config，主动 `kubectl apply` 到集群。GitOps 翻转了这个方向 —— 集群里跑一个 Agent，**它自己**去 Git 拉。

```
Push 模型（Jenkins / GitLab CI）
    Git ─push trigger─► CI Runner ──kubectl apply──► Cluster
                            ↑
                       (持有 kube config)

Pull 模型（ArgoCD / Flux）
    Git ◄──poll/webhook── Agent
                              │
                              ▼
                          Cluster (Agent 在集群内)
```

差别不是"谁主动谁被动"这种表面，而是**信任边界**和**一致性保证**：

**安全角度**：
- Push：CI 服务器**持有所有目标集群的 admin 凭证**。CI 系统被攻破 = 所有集群被攻破。这是 SolarWinds 之后大家都怕的链。
- Pull：集群只需要 Git 的**只读凭证**，且这个凭证可以在集群内 Vault / Secret 管理。Git 平台被攻破 = 攻击者能改代码，但还需要 PR review 才能 merge。

**一致性角度**：
- Push：CI 是事件驱动 —— commit 触发部署，**不 commit 就不会同步**。如果有人手 `kubectl edit` 改了集群，Push 模型永远发现不了。
- Pull：Agent 是循环驱动 —— 每 N 秒比对实际 vs Git，**漂移会被发现**（甚至自动修复）。

**网络角度**：
- Push：CI 必须能访问集群 API Server。如果集群在 VPC 里，要么开公网，要么 CI 也进 VPC。
- Pull：Agent 在集群内主动出向访问 Git，**不需要任何入向通路**。这对私有云、合规要求高的场景是关键优势。

GitOps 选 Pull 不是技术偏好，是**把"谁能改集群"的权力收回到集群自己手里**。

> **口诀**：Push 是"我推给你"（凭证在外），Pull 是"我去拿"（凭证在内）。前者快但脆，后者慢一拍但能扛 SolarWinds 级攻击。

### 1.3 ArgoCD 架构

ArgoCD 不是"一个进程"。它是一组协作的 controller，理解清楚它们的职责才能定位"为什么 sync 慢"、"为什么内存爆"这类问题。

```
┌─────────────────────────────────────────────────────────────┐
│                       ArgoCD Namespace                      │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  API Server  │  │ Repo Server  │  │ App Controller   │  │
│  │              │  │              │  │                  │  │
│  │ gRPC/REST    │  │ git clone    │  │ Reconcile loop   │  │
│  │ Web UI       │  │ helm template│  │ Diff & Sync      │  │
│  │ RBAC         │  │ kustomize    │  │ Health check     │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│         │                  │                   │            │
│         └──────────────────┼───────────────────┘            │
│                            │                                │
│                  ┌─────────▼─────────┐                      │
│                  │  Redis (state)    │                      │
│                  └───────────────────┘                      │
└─────────────────────────────────────────────────────────────┘
                             │
                             │ kube API
                             ▼
                ┌──────────────────────────────┐
                │   Target Cluster(s)          │
                │   (可以是自己也可以是远程)    │
                └──────────────────────────────┘
```

三个核心组件：

**API Server**：
- 暴露 REST/gRPC，给 CLI/UI/CI 用
- RBAC、SSO、审计在这里
- 不直接干"同步"的活

**Repo Server**：
- 克隆 Git 仓库（可以是 https/ssh/oci）
- 渲染 manifest（执行 `helm template`、`kustomize build`、jsonnet 等）
- **无状态**，但内存吃得最多 —— 大 chart 渲染、上千个 Application 同时缓存
- 注意：renders happen here，不在目标集群

**Application Controller**：
- 真正的"调和循环"
- 比对 Repo Server 渲染出的 manifest vs 集群实际状态
- 计算 diff、决定 sync action、调用目标集群 API
- 健康检查（用内置/自定义 lua 脚本判断 Resource 是否 healthy）
- 默认每 3 分钟一轮 reconcile（可配置 `timeout.reconciliation`）

**Redis**：
- 缓存 manifest 渲染结果（避免每次都 helm template）
- 缓存集群资源元数据
- **生产必须独立部署**，不能用 ArgoCD 自带的单 Pod Redis

为什么这样切？因为三种工作的资源画像不一样：API Server 是请求驱动（突发）、Repo Server 是 CPU/IO 密集（git clone + helm render）、App Controller 是常驻 watch + 调度。每一种独立扩展才不会互相挤。

> **口诀**：ArgoCD 不是单体。Repo Server 吃内存、App Controller 吃 CPU/watch、API Server 吃请求 —— 出问题先看是哪个组件。

### 1.4 Application 与 ApplicationSet

ArgoCD 里"被部署的东西"用 `Application` CRD 描述。一个 Application = 一个"源 → 目标"的映射。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
spec:
  project: payments              # 属于哪个 AppProject（RBAC 边界）
  source:
    repoURL: https://github.com/org/manifests
    path: services/order
    targetRevision: HEAD          # 跟哪条分支/tag/commit
  destination:
    server: https://kubernetes.default.svc
    namespace: payments
  syncPolicy:
    automated:
      prune: true                 # 删除 Git 里不存在的资源
      selfHeal: true              # 集群被人改回来
```

`AppProject` 是**RBAC + 安全策略**的边界：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: payments
  namespace: argocd
spec:
  sourceRepos:
    - https://github.com/org/manifests
  destinations:
    - namespace: payments
      server: https://kubernetes.default.svc
  clusterResourceWhitelist: []    # 不允许部署集群级资源
  namespaceResourceBlacklist:
    - group: ""
      kind: ResourceQuota         # 不允许改 quota
  roles:
    - name: payments-deployer
      policies:
        - p, proj:payments:payments-deployer, applications, sync, payments/*, allow
```

为什么生产**不能**让所有 Application 都用 default Project？因为 default Project 默认允许任何源、任何目的、任何资源类型。一旦某个 Application 的 Git 仓库被攻破（或写错了 path），可以在集群里部署任何东西 —— 包括 ClusterRole、PV、CRD。

**ApplicationSet** 是 Application 的模板化生成器。考虑一个场景：你有 30 个微服务，每个服务要在 dev/staging/prod 三个集群部署。手写 90 个 Application 既繁琐又容易漂移。ApplicationSet 用 generator + template 生成。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: services-fanout
spec:
  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/org/manifests
              directories:
                - path: services/*
          - list:
              elements:
                - cluster: dev
                  url: https://dev-cluster.example.com
                - cluster: prod
                  url: https://prod-cluster.example.com
  template:
    metadata:
      name: '{{path.basename}}-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/manifests
        path: '{{path}}'
      destination:
        server: '{{url}}'
        namespace: '{{path.basename}}'
```

Matrix Generator 把 Git directories（30 个服务）和 List（2 个集群）做笛卡尔积，**生成 60 个 Application**。改一个模板，所有实例同步更新。

> **口诀**：Application = 单部署，AppProject = RBAC 边界，ApplicationSet = 批量生成器。三层抽象一个不能混。

### 1.5 Sync 策略

Application 的 `syncPolicy` 看似只有几个字段，但行为差别巨大：

| 配置 | Git 改了 | 集群被人改了 | 删了 Git 的资源 |
| --- | --- | --- | --- |
| **manual**（缺省） | 等人点 Sync | 显示 OutOfSync，不动 | 不删 |
| **automated** | 自动 sync | 显示 OutOfSync，不动 | 不删 |
| **automated + selfHeal** | 自动 sync | **自动改回去** | 不删 |
| **automated + prune** | 自动 sync | 显示 OutOfSync | **自动删** |
| **automated + selfHeal + prune** | 自动 sync | 自动改回去 | 自动删 |

**selfHeal** 是把双刃剑。它是 GitOps 最强的卖点（漂移自动修复），也是事故 ③ 的元凶（紧急 hotfix 被覆盖）。

**prune** 是另一把双刃剑。设想：你不小心在 PR 里删了 Service.yaml，merge 后 ArgoCD 直接把生产的 Service 删了 —— 流量全断。所以生产的 Application 通常配 `prunePropagationPolicy: foreground` + `Annotation: argocd.argoproj.io/sync-options: Prune=false` 显式保护关键资源。

Sync 阶段还有"何时删"的子语义。Kubernetes 的 propagation 有 background / foreground / orphan 三种。ArgoCD 默认 foreground，意味着删 Deployment 时会**等所有 Pod 也被删干净**才返回 success。如果 Pod 有长 graceful shutdown（比如 60s），sync 会卡 60s。

### 1.6 Flux 与 ArgoCD 的设计差别

Flux v2（GitOps Toolkit）和 ArgoCD 都是 CNCF 毕业项目，目标重叠但设计哲学不同。

| 维度 | ArgoCD | Flux v2 |
| --- | --- | --- |
| **抽象** | Application 中心 | Source + Kustomization/HelmRelease 分离 |
| **UI** | 内置富 Web UI | 无内置 UI（社区有 Weave GitOps / Capacitor） |
| **多集群** | 单 ArgoCD 管多集群（hub-spoke） | 每集群一个 Flux（每集群自治） |
| **Sync 模型** | 拉 → 渲染 → 应用 | 同上但解耦：Source controller 拉，Kustomize controller 渲染 |
| **CRD 数量** | 少（Application/AppProject/ApplicationSet） | 多（GitRepository, OCIRepository, Kustomization, HelmRelease, ImagePolicy...） |
| **学习曲线** | 平缓 | 陡（CRD 多、需理解 toolkit 协作） |
| **安全模型** | 中心化 RBAC（AppProject） | 分布式 RBAC（每集群独立 KSA） |
| **典型选型** | App team / 平台需要可视化 | Platform team / 多 tenant 强隔离 |

**核心设计差别**：ArgoCD 是 "**一个管理面**" 思路，所有集群在一个 UI 里看；Flux 是 "**每集群一个 GitOps agent**" 思路，集群之间没有共享控制平面。

后者更符合"把控制权下放到集群"的极端 GitOps 哲学，但对 audit/可视化不友好。前者反过来。

实际选型：
- 你团队 < 50 人 / 集群 < 5 个 / 强烈需要 UI → ArgoCD
- 你是 platform team / 多 tenant / 强隔离 / 不在意 UI → Flux
- 你想用 image automation（图像 tag 自动 PR） → Flux 内置，ArgoCD 需要外部 Image Updater

### 1.7 多集群 GitOps

单集群 GitOps 是简单的；难点全在多集群。考虑一个常见拓扑：

```
        ┌──────────────────────┐
        │   Control Plane      │
        │   ArgoCD + Repos     │
        └──────────┬───────────┘
                   │ kube API (远程)
       ┌───────────┼───────────┬─────────────┐
       ▼           ▼           ▼             ▼
   ┌───────┐  ┌───────┐  ┌───────┐    ┌───────┐
   │ dev-  │  │stage- │  │prod-  │    │prod-  │
   │ us    │  │ us    │  │ us    │    │ eu    │
   └───────┘  └───────┘  └───────┘    └───────┘
```

ArgoCD 用 **Cluster Generator** 自动发现所有注册的集群：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: monitoring-stack
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            env: production         # 只在 label env=production 的集群部署
  template:
    metadata:
      name: 'monitoring-{{name}}'
    spec:
      destination:
        server: '{{server}}'
        namespace: monitoring
      source:
        repoURL: https://github.com/org/manifests
        path: monitoring
        targetRevision: HEAD
```

集群注册：

```bash
argocd cluster add prod-us-context --label env=production --label region=us
argocd cluster add prod-eu-context --label env=production --label region=eu
```

这种 hub-spoke 模式有个**深坑**：控制平面集群挂了 = 所有 GitOps 停摆。所以严肃生产要么：
1. 控制平面集群本身 HA + 跨 AZ
2. **每个 region 一个 ArgoCD**（federation 模式），各自管自己 region 的集群
3. 用 Flux per-cluster 模式

第 2 条又引出新问题：那 ArgoCD 自己怎么 GitOps 部署？答案是 **app of apps** 模式 —— 用一个 root Application 管理所有其他 Application 的定义，包括 ArgoCD 自身的 Helm chart。引导逻辑（"自举"）由 `kubectl apply` 一次性 bootstrap，之后 ArgoCD 自管自。

> **口诀**：单 ArgoCD hub-spoke 适合 < 20 集群；> 20 或跨 region 必须 federation；ArgoCD 自己用 app-of-apps 自举。

---

## 2. 工程切片：从安装到多集群

> 本节所有 YAML 在 ArgoCD 2.10 + K8s 1.28 验证过，可以复制粘贴。

### 2.1 ArgoCD 安装与 RBAC

最小安装：

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.10.0/manifests/install.yaml
```

但生产**不要**用 install.yaml。它有几个问题：
- Redis 是单 Pod，HA 不够
- 资源 limit 默认很小，大 repo 会 OOM
- TLS / ingress 都没配

生产用官方 Helm chart：

```yaml
# values.yaml
global:
  domain: argocd.example.com

controller:
  replicas: 2                          # App Controller HA（>= 2.10 支持）
  resources:
    requests: { cpu: 1000m, memory: 2Gi }
    limits:   { cpu: 4000m, memory: 8Gi }
  env:
    - name: ARGOCD_CONTROLLER_REPLICAS
      value: "2"
    - name: ARGOCD_RECONCILIATION_TIMEOUT
      value: "180s"

repoServer:
  replicas: 3                          # Repo Server 是无状态的，开多副本扛大渲染
  resources:
    requests: { cpu: 500m, memory: 1Gi }
    limits:   { cpu: 2000m, memory: 4Gi }

server:
  replicas: 2
  ingress:
    enabled: true
    hostname: argocd.example.com
    tls: true

redis-ha:
  enabled: true                        # 用 HA Redis 替代默认单 Pod

dex:
  enabled: true                        # 启用 SSO（OIDC/SAML）
```

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace -f values.yaml
```

RBAC 配置 ConfigMap `argocd-rbac-cm`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly        # 默认只读
  policy.csv: |
    # 平台团队：所有项目所有操作
    p, role:platform-admin, applications, *, */*, allow
    p, role:platform-admin, clusters, *, *, allow
    p, role:platform-admin, repositories, *, *, allow
    p, role:platform-admin, projects, *, *, allow

    # 业务团队：只能操作自己 project 下的 Application
    p, role:payments-dev, applications, get, payments/*, allow
    p, role:payments-dev, applications, sync, payments/*, allow
    p, role:payments-dev, applications, action/*, payments/*, allow
    p, role:payments-dev, applications, override, payments/*, allow

    # 紧急 hotfix 角色：能 override 但要审计
    p, role:hotfix, applications, override, */*, allow
    p, role:hotfix, applications, sync, */*, allow

    # 绑定到 SSO group
    g, payments-team@example.com, role:payments-dev
    g, sre-oncall@example.com, role:hotfix
    g, platform@example.com, role:platform-admin
```

注意 `policy.default: role:readonly` —— 默认所有未匹配用户**只读**，避免误操作。

### 2.2 Application 配置（含 Helm / Kustomize）

最简单的 raw YAML：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-demo
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # 删 Application 时连资源一起删
spec:
  project: default
  source:
    repoURL: https://github.com/org/manifests
    path: demo/nginx
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true             # namespace 不存在自动建
      - PruneLast=true                   # 先 apply 再 prune（避免依赖错误）
      - ApplyOutOfSyncOnly=true          # 只 apply 变化的资源
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

Helm chart：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: redis
spec:
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: redis
    targetRevision: 18.6.0
    helm:
      releaseName: redis
      valueFiles:
        - $values/redis/values-prod.yaml   # $values 引用第二个 source
      parameters:
        - name: image.tag
          value: 7.2.4
  sources:
    - repoURL: https://charts.bitnami.com/bitnami
      chart: redis
      targetRevision: 18.6.0
    - repoURL: https://github.com/org/manifests
      targetRevision: HEAD
      ref: values                          # 这个 source 只是为了 valueFiles 引用
  destination:
    server: https://kubernetes.default.svc
    namespace: cache
```

注意 `sources`（多源，2.6+）的设计：chart 来自 chart repo，values 来自 Git。这样能力强但复杂度高，团队小的话**用单源 + 把 chart 整个 vendor 进 Git 更省心**。

Kustomize：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: api-gateway-prod
spec:
  source:
    repoURL: https://github.com/org/manifests
    path: gateway/overlays/prod
    targetRevision: HEAD
    kustomize:
      images:
        - api-gateway:v1.2.3              # 覆盖 image tag
      commonLabels:
        env: prod
      namePrefix: prod-
```

Helm vs Kustomize 在 GitOps 场景的取舍：

| 维度 | Helm | Kustomize |
| --- | --- | --- |
| 模板表达力 | 强（go template） | 弱（patch） |
| 可读性 | 复杂模板难读 | overlay 直观 |
| 环境差异 | values.yaml | overlays/ |
| Lint/校验 | helm lint | kubectl kustomize \| kubeval |
| ArgoCD 渲染成本 | 高（exec helm） | 低（内置） |
| 推荐场景 | 第三方 chart 复用 | 自家应用多环境 |

很多团队两者并用：第三方组件用 Helm，自家应用用 Kustomize。

### 2.3 ApplicationSet 多环境模板

最常见的场景：**一个微服务，多套环境**。用 Git Generator + 目录约定：

```
repo/
├── services/
│   ├── order/
│   │   ├── base/
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       ├── staging/
│   │       └── prod/
│   └── payment/
│       └── overlays/{dev,staging,prod}/
└── applicationsets/
    └── all-services.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: all-services
  namespace: argocd
spec:
  goTemplate: true                         # 启用 go template（2.6+）
  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/org/manifests
              revision: HEAD
              directories:
                - path: services/*/overlays/*
          - list:
              elements:
                - env: dev
                  cluster: https://dev.example.com
                  autoSync: true
                - env: staging
                  cluster: https://staging.example.com
                  autoSync: true
                - env: prod
                  cluster: https://prod.example.com
                  autoSync: false             # prod 手动触发
  template:
    metadata:
      name: '{{index .path.segments 1}}-{{.env}}'
      labels:
        env: '{{.env}}'
        service: '{{index .path.segments 1}}'
    spec:
      project: '{{index .path.segments 1}}'
      source:
        repoURL: https://github.com/org/manifests
        path: '{{.path.path}}'
        targetRevision: HEAD
      destination:
        server: '{{.cluster}}'
        namespace: '{{index .path.segments 1}}'
      syncPolicy:
        automated:
          prune: '{{.autoSync}}'
          selfHeal: '{{.autoSync}}'
```

**模板渲染的坑**：`{{ }}` 在 YAML 里要特别小心。go template 启用前用 `{{path.basename}}` 是 fasttemplate 语法；启用后必须写 `{{.path.basename}}`。混用会渲染失败 —— 见案例 ⑥。

Pull Request Generator 是另一个利器 —— 每个 PR 自动起一个 preview 环境：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: pr-previews
spec:
  generators:
    - pullRequest:
        github:
          owner: org
          repo: app
          tokenRef:
            secretName: github-token
            key: token
          labels:
            - preview                     # 仅 PR 带 preview label 时触发
        requeueAfterSeconds: 60
  template:
    metadata:
      name: 'preview-pr-{{number}}'
    spec:
      source:
        repoURL: https://github.com/org/app
        targetRevision: '{{branch}}'
        path: deploy
        kustomize:
          namePrefix: 'pr-{{number}}-'
      destination:
        server: https://kubernetes.default.svc
        namespace: 'preview-pr-{{number}}'
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
        automated:
          prune: true
          selfHeal: true
```

PR 关闭时 ApplicationSet controller 自动删除对应 Application（因为 PR 不在 generator 输出里了）。

### 2.4 Sync Hooks

很多场景需要"在 sync 之前/之中/之后做点事"：DB migration、cache warmup、通知。ArgoCD 用 annotation 标记 Hook：

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync             # 先于业务资源 apply
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: org/db-migrate:v1.2.3
          command: ["./migrate", "up"]
```

Hook 类型：

| Hook | 何时跑 | 典型用途 |
| --- | --- | --- |
| **PreSync** | sync 之前 | DB migration、备份 |
| **Sync** | 与主资源一起 | 默认值；几乎不用作 hook |
| **PostSync** | sync 完成且健康 | 缓存预热、smoke test、通知 |
| **SyncFail** | sync 失败 | 回滚通知 |

Hook delete policy：

| 策略 | 行为 |
| --- | --- |
| **HookSucceeded** | Hook 成功后删（默认） |
| **HookFailed** | Hook 失败后删 |
| **BeforeHookCreation** | 下次创建 Hook 前删旧的 |

**陷阱**：默认 Job 跑完成功就删了，**调试时看不到日志**。生产环境建议：
- 关键 Hook 用 `BeforeHookCreation`，保留最后一次结果
- 配套 logging（loki/elastic）抓 Pod 日志，避免 Pod 删了日志没了

### 2.5 Sealed Secrets / SOPS

GitOps 最敏感的话题：**密钥怎么放进 Git**。明文当然不行；但加密之后**谁来解密**？两个主流方案：

#### Sealed Secrets（Bitnami）

控制器在集群里生成一对非对称密钥，公钥可以分发，私钥在集群保留。`kubeseal` CLI 用公钥加密 Secret，加密后的 SealedSecret CR 可以提交 Git。集群里的 controller 用私钥解密，转成普通 Secret。

```bash
# 安装 controller
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# 拉公钥
kubeseal --fetch-cert > pub-cert.pem

# 加密
echo -n "supersecret" | kubectl create secret generic db-pass \
  --dry-run=client \
  --from-file=password=/dev/stdin \
  -o yaml | \
  kubeseal --cert pub-cert.pem -o yaml > sealed-db-pass.yaml

# 把 sealed-db-pass.yaml 提交到 Git
```

提交后的 SealedSecret 长这样：

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-pass
  namespace: app
spec:
  encryptedData:
    password: AgBy3i4OJSWK...   # 加密后的字符串
```

Sealed Secrets 的限制：
- **私钥绑定到集群** —— 跨集群无法直接复用同一个 SealedSecret
- 私钥丢失 = 所有 SealedSecret 解不开 —— 必须**定期备份**
- 加密单元是单条 Secret，不是文件级

#### SOPS（Mozilla）

SOPS 加密文件级，支持多种 KMS 后端（AWS KMS、GCP KMS、Azure Key Vault、age、PGP）。GitOps 里通常用 age（轻量、免依赖云）：

```bash
# 生成 age 密钥
age-keygen -o key.txt
# Public key: age1xy...
# 私钥放进集群（Vault / SOPS controller）

# 加密
sops --encrypt --age age1xy... secrets.yaml > secrets.enc.yaml

# 提交 secrets.enc.yaml
```

加密后的文件保留结构，只加密 value：

```yaml
db:
  password: ENC[AES256_GCM,data:abc123,iv:...,tag:...,type:str]
sops:
  age:
    - recipient: age1xy...
      enc: |
        -----BEGIN AGE...
```

**ArgoCD 集成**：用 [argocd-vault-plugin](https://github.com/argoproj-labs/argocd-vault-plugin) 或 [helm-secrets](https://github.com/jkroepke/helm-secrets) 插件，在 Repo Server 渲染时解密。

#### 选哪个？

| 维度 | Sealed Secrets | SOPS |
| --- | --- | --- |
| 加密粒度 | Secret 级 | 文件级 |
| 跨集群 | 难（需要复用密钥） | 易（KMS 集中管理） |
| 密钥泄漏处理 | 重新生成密钥 + 重加密所有 SealedSecret | 撤销 KMS key + 重加密 |
| ArgoCD 集成 | 原生（专用 controller） | 需要插件 |
| 学习曲线 | 低 | 中（KMS 概念） |
| 推荐场景 | 单集群 / 小团队 | 多集群 / 已有云 KMS |

**两者都不解决的问题**：动态密钥（每次拉取都不一样的、有 TTL 的）。这种场景必须用 **External Secrets Operator + Vault**：Git 里只放"密钥引用"，运行时 ESO 从 Vault 拉真值。

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h                  # TTL
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-credentials              # 生成的 Secret 名
  data:
    - secretKey: password
      remoteRef:
        key: secret/data/db
        property: password
```

### 2.6 ArgoCD Notifications

部署成功/失败通知是 GitOps 落地的基础需求。argocd-notifications 内置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token

  template.app-deployed: |
    message: |
      Application {{.app.metadata.name}} is now running new version.
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name}}",
          "color": "#18be52",
          "fields": [
            {"title": "Sync Status", "value": "{{.app.status.sync.status}}", "short": true},
            {"title": "Repository", "value": "{{.app.spec.source.repoURL}}", "short": true},
            {"title": "Revision", "value": "{{.app.status.sync.revision}}", "short": true}
          ]
        }]

  template.app-sync-failed: |
    message: |
      Application {{.app.metadata.name}} sync FAILED.
      Error: {{.app.status.operationState.message}}

  trigger.on-deployed: |
    - description: Application is synced and healthy
      send: [app-deployed]
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'

  trigger.on-sync-failed: |
    - description: Application syncing has failed
      send: [app-sync-failed]
      when: app.status.operationState.phase in ['Error', 'Failed']

  subscriptions: |
    - recipients:
        - slack:platform-alerts
      triggers:
        - on-sync-failed
```

Application 上加 annotation 订阅：

```yaml
metadata:
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: payments-deploy
    notifications.argoproj.io/subscribe.on-sync-failed.slack: payments-alerts
```

注意：notifications 默认**对每次状态变化都发**，频繁 sync 会刷屏。生产建议加 throttle 或只在状态变化时发（`when` 加上 `time.Now().Sub(...).Minutes() > 5` 之类的限流）。

### 2.7 多集群部署模式

#### 模式 A：单 ArgoCD Hub-Spoke

最简单的多集群 GitOps：一个中心 ArgoCD 管理多个目标集群。

```bash
# 在 hub 集群安装 ArgoCD
helm install argocd argo/argo-cd -n argocd

# 注册目标集群（CLI 自动创建 Secret + ServiceAccount + ClusterRoleBinding）
argocd cluster add prod-us --label env=prod --label region=us
argocd cluster add prod-eu --label env=prod --label region=eu
argocd cluster add staging --label env=staging
```

集群注册的本质是在 hub 创建一个 Secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster-prod-us
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: prod-us
  server: https://prod-us.example.com
  config: |
    {
      "bearerToken": "xxx",
      "tlsClientConfig": { "caData": "yyy" }
    }
```

适合：**< 20 集群，单 region，无强合规隔离**。

#### 模式 B：每 Region 一个 ArgoCD（Federation）

跨 region 部署时，hub-spoke 的延迟、可用性、故障域都成问题。改成每 region 一个 ArgoCD：

```
us-region/
  ArgoCD-us  ──manages──►  prod-us, staging-us
eu-region/
  ArgoCD-eu  ──manages──►  prod-eu, staging-eu
```

每个 ArgoCD 从同一个 Git 仓库拉，但配置 region filter：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: services-us
spec:
  generators:
    - clusters:
        selector:
          matchLabels:
            region: us         # 只匹配 region=us 的集群（即本 hub 自己注册的）
  template: ...
```

#### 模式 C：每集群一个 Flux

最分散的模式：每个集群跑自己的 GitOps agent，无中心控制平面。Flux 是这种模式的代表。

```bash
# 在每个集群单独 bootstrap
flux bootstrap github \
  --owner=org \
  --repository=manifests \
  --branch=main \
  --path=clusters/prod-us \
  --personal
```

`clusters/prod-us/` 目录下的所有 manifest 由这个集群的 Flux 拉。集群之间互不知道，**没有 Single Point of Failure**。

#### 三种模式选型

| 模式 | 适合 | 优点 | 缺点 |
| --- | --- | --- | --- |
| **A 单 hub** | 团队小、集群少、需要统一 UI | 简单、统一视角 | hub 挂全 GG |
| **B 每 region 一 hub** | 跨 region、合规要求中等 | 故障域隔离、延迟低 | 视角碎片化、需要 federation 工具 |
| **C 每集群 Flux** | 极致隔离、强合规、多 tenant | 无 SPoF、最强隔离 | 没有统一 UI、多套配置 |

> **口诀**：< 5 集群同 region → A；跨 region 但同公司 → B；多租户/极致隔离 → C。

---

## 3. 实测数据：Sync 不是免费的

> 测试环境：AWS EKS 1.28，控制平面集群 m5.xlarge × 3，目标集群 m5.large × 5，ArgoCD 2.10.0。
> 数据仅供方向性参考，你自己环境的绝对值会差很多。

### 3.1 ArgoCD 的 Sync 延迟

测量"Git push commit → 集群里 Pod 实际更新"的端到端延迟。

| 配置 | 触发模式 | 平均延迟 | P99 |
| --- | --- | --- | --- |
| 默认 polling（180s 间隔） | polling | 92s | 178s |
| 默认 polling + webhook | webhook | 11s | 28s |
| 高频 polling（30s） | polling | 16s | 32s |
| 多 Application 同时变更（100 个） | webhook | 64s | 134s |

**结论**：
- 没配 webhook 的 ArgoCD 延迟是按 reconcile 周期算的，平均 = 周期 / 2
- Webhook 把 Git push → sync 的延迟降到秒级
- 大批量同时变更时，受 Repo Server 渲染并发度限制（默认 max 50 并行 manifest 生成）

Webhook 配置（GitHub）：

```bash
# 在 GitHub 仓库 Settings → Webhooks 添加
URL: https://argocd.example.com/api/webhook
Content type: application/json
Events: Just the push event

# ArgoCD 端配置 secret
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {"webhook.github.secret": "your-webhook-secret"}}'
```

### 3.2 大规模 Application 资源消耗

模拟 Application 数量从 100 → 5000，观察 Repo Server 和 App Controller 的 CPU/内存。

| Application 数 | Repo Server 内存 | App Controller 内存 | App Controller CPU |
| --- | --- | --- | --- |
| 100 | 800MB | 500MB | 0.3 cores |
| 500 | 1.4GB | 1.2GB | 0.8 cores |
| 1000 | 2.1GB | 2.0GB | 1.5 cores |
| 2000 | 3.8GB | 3.5GB | 2.8 cores |
| 5000 | 7.2GB（OOM 风险） | 7.0GB | 5.0+ cores |

**关键观察**：
- Repo Server 内存几乎线性增长（缓存所有 manifest 渲染结果）
- 1000+ Application 时**必须**开启 sharding（App Controller 多副本，每个负责一部分 cluster）
- Redis 也成瓶颈 —— 默认配置 Redis maxmemory 是 200MB，1000+ Application 时需要调到 2GB+

调优方向：

```yaml
# values.yaml
controller:
  replicas: 4                           # sharding，每副本约 1500 application
  env:
    - name: ARGOCD_CONTROLLER_REPLICAS
      value: "4"

repoServer:
  replicas: 6
  env:
    - name: ARGOCD_EXEC_TIMEOUT
      value: "180s"                     # helm template 超时
    - name: ARGOCD_GIT_ATTEMPTS_COUNT
      value: "3"

redis-ha:
  enabled: true
  redis:
    config:
      maxmemory: "2gb"
      maxmemory-policy: "allkeys-lru"
```

### 3.3 Pull vs Push 的部署延迟

同一个微服务，部署到同一个集群，对比两种模式：

| 模式 | 平均部署延迟 | P99 | 故障情况 |
| --- | --- | --- | --- |
| Push（GitLab CI + kubectl apply） | 28s | 47s | CI runner 挂 → 部署不出去 |
| Pull（ArgoCD + webhook） | 14s | 31s | ArgoCD 挂 → 部署不出去；但**新代码不会被推坏** |
| Pull（ArgoCD + polling 180s） | 92s | 178s | 同上 |

**反直觉的点**：Pull 加 webhook 比 Push 还快。原因：Push 模型 CI runner 要先 docker pull、helm package、再 apply，串行链条长；Pull 模型 ArgoCD 收到 webhook 时只做 git pull + diff + apply，没 docker 步骤（image 已经 push 到 registry 了）。

### 3.4 不同 Sync 策略的回滚速度

回滚场景：发现 v2 有 bug，需要回到 v1。

| 方式 | 操作 | 生效时间 | 风险 |
| --- | --- | --- | --- |
| Git revert + ArgoCD auto-sync | git revert + push | 15-180s（看 webhook） | 干净，符合 GitOps |
| ArgoCD UI rollback | 点 History → Rollback | < 5s | 与 Git 不一致，下次 reconcile 可能被覆盖 |
| `kubectl rollout undo` | 命令行直接回滚 | < 5s | 同上，且 selfHeal 会马上改回去 |
| 紧急 Sync to specific revision | argocd app sync --revision=v1 | 10-30s | Git 还是 v2，需要后续补 PR |

**推荐**：
- **正常回滚**：永远用 git revert，符合 GitOps 原则
- **紧急回滚**（火烧眉毛）：用 `argocd app sync --revision=` 指定老 commit，**事后必须补 PR 把 Git 拉回到那个状态**，否则下次 reconcile 会再次部署有 bug 的版本

> **数据后结论**：**没有 webhook 的 ArgoCD 是半残的**。生产部署 webhook 是最高 ROI 的优化。1000+ Application 必须开 sharding。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 配置漂移 —— "kubectl apply 改了集群没人知道"

#### 现象

某天 SRE 发现生产集群的 nginx ConfigMap 和 Git 里不一样。问业务，业务说"上周值班的时候为了排查问题手动改了一下，忘记同步回 Git 了"。ArgoCD UI 上显示 OutOfSync，但因为没开 selfHeal，没人主动 sync，漂移持续了一周。

#### 根因

Application 配置：

```yaml
syncPolicy:
  automated:
    prune: false
    selfHeal: false           # ★ 漂移不会自动纠正
```

且 ArgoCD 默认**不阻止集群外的 kubectl 操作**。任何有 kubectl 权限的人都能直接改集群，ArgoCD 只能"发现"漂移，不能阻止。

更深一层：很多公司给业务团队 namespace-scoped 的 kubectl 权限用于排查（看日志、exec 进容器），但 RBAC 里**没有把"修改"权限收掉** —— 看 logs 需要 `pods/log` 但不需要 `update`，可惜业界默认 view role 给了所有 read 权限，dev role 给了所有 write 权限。

#### 修复

**第一步**：所有生产 Application 默认开 selfHeal。

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true               # ★ 漂移立即纠正
```

但 selfHeal 又引出案例 ③ 的问题（hotfix 被覆盖），所以更严格的方案是：

**第二步**：用 RBAC 把生产集群的 kubectl 写权限收紧。

```yaml
# 业务团队只读 + log 查看
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-debugger
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "endpoints"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log", "pods/exec"]    # exec 但不能 update
    verbs: ["get", "create"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
```

**第三步**：监控漂移。ArgoCD Prometheus 指标：

```promql
# 持续 5 分钟 OutOfSync 的 Application
sum by (name, namespace) (
  argocd_app_info{sync_status="OutOfSync"}
) > 0
```

告警接到 PagerDuty / 钉钉。

#### 验证

```bash
# 故意漂移一下
kubectl edit configmap nginx-config -n nginx
# 改一个值

# 等 selfHeal 触发（默认 30s 内）
sleep 60
kubectl get configmap nginx-config -n nginx -o yaml | grep <changed-value>
# 期望：值已被改回 Git 里的版本
```

> **口诀**：**生产环境只能通过 Git 改集群**。要么 RBAC 收紧 kubectl 写权限，要么 selfHeal 兜底 + 监控告警。两条同时做最稳。

---

### 案例 ② Sync 失败 —— "CRD 顺序错误"

#### 现象

部署一个新的 monitoring stack，包含 Prometheus Operator + 一堆 ServiceMonitor。第一次 sync 总失败：

```
no matches for kind "ServiceMonitor" in version "monitoring.coreos.com/v1"
```

但单独看 Application，CRD 是包含的（`prometheus-operator-crds`）。

#### 根因

Kubernetes 的 CRD 注册和 CR 创建是**异步**的：CRD apply 之后，要等 API Server 注册 schema、kube-aggregator 同步，CR 才能用。ArgoCD 默认**一次性 apply 所有资源**，不等 CRD 就绪就开始 apply CR，导致 ServiceMonitor 找不到 kind。

更细的：ArgoCD 的 sync 把资源按 wave 分组并发执行。默认所有资源 wave=0，**同一 wave 内顺序未定义**。

#### 修复

**方案 A**：用 sync wave 显式排序。

```yaml
# CRD 资源
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: servicemonitors.monitoring.coreos.com
  annotations:
    argocd.argoproj.io/sync-wave: "-1"      # ★ wave 越小越早 apply

---
# Operator 部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-operator
  annotations:
    argocd.argoproj.io/sync-wave: "0"

---
# ServiceMonitor（依赖 CRD + Operator）
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

**方案 B**：用 ServerSideApply + Replace。

```yaml
syncPolicy:
  syncOptions:
    - ServerSideApply=true
    - Replace=true
    - SkipDryRunOnMissingResource=true     # ★ CRD 还没就绪时跳过 dry-run
```

`SkipDryRunOnMissingResource` 是关键 —— ArgoCD 默认会 dry-run 所有资源，CR 找不到 kind 时 dry-run 就失败。开了这个 ArgoCD 会跳过 dry-run，直接 apply（API Server 这时通常已经认识 CRD 了）。

**方案 C**：拆成两个 Application。

```yaml
# 第一个 Application：只装 CRD + Operator
---
# 第二个 Application：装 ServiceMonitor 等 CR
# 用 app-of-apps 模式控制顺序
```

#### 验证

```bash
# 删了重 sync
kubectl delete application monitoring -n argocd
kubectl apply -f application.yaml
argocd app sync monitoring

# 观察 sync 日志
argocd app logs monitoring
# 期望：CRD 先 apply，CR 后 apply，无 "no matches for kind" 错误
```

> **口诀**：CRD 必须用 sync-wave 排在 CR 之前；或开 `SkipDryRunOnMissingResource`；或拆 Application。三选一，别让 K8s 自己赌时序。

---

### 案例 ③ Self-Heal 把紧急 hotfix 覆盖 —— "改完三秒被冲掉"

#### 现象

凌晨 2 点生产事故，DBA 紧急 `kubectl edit` 改了 Pod 的 JVM 参数（`-Xmx` 调大），改完 3 秒后被 ArgoCD 改回去了。继续改、继续被改回去。整整折腾 20 分钟才意识到是 selfHeal 干的。

#### 根因

Application 配置 selfHeal=true。任何对受 ArgoCD 管理的资源的"集群侧"修改，都会在下次 reconcile 时被改回 Git 里的状态。ArgoCD 默认 reconcile 周期 180s，但 selfHeal 触发的同步**很多时候在 30s 内**（因为 watch 到了变更立即触发）。

更深的设计：selfHeal 的存在就是为了消除漂移。它**忠诚于 Git**，对人手改完全不留情。这是 GitOps 的核心特性，但**也是紧急场景最大的敌人**。

#### 修复

**禁止人手改集群**这条铁律不能松，但**紧急 hotfix 必须有合法路径**。三种合法绕过模式：

**模式 A**：临时关 selfHeal。

```bash
argocd app set <app-name> --self-heal=false
# 改集群
kubectl edit deployment xxx
# 事后：补 PR 改 Git，merge 后再开 selfHeal
argocd app set <app-name> --self-heal=true
```

**模式 B**：sync 一个 hotfix branch。

```bash
# 在 git push 一个 hotfix branch
git checkout -b hotfix-jvm-mem
# 改 deployment.yaml
git push origin hotfix-jvm-mem

# ArgoCD sync 到这个 branch
argocd app set <app-name> --revision=hotfix-jvm-mem
argocd app sync <app-name>

# 修完事故后，正常 PR 走 review merge 到 main
# 然后 sync 回 main
argocd app set <app-name> --revision=HEAD
```

**模式 C**：用 Application 注解临时禁用 sync。

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-options: "Force=true,Replace=true"
    argocd.argoproj.io/refresh: "true"
```

**最推荐的工程化方案**：**hotfix 流程**写进运维 SOP。

```
紧急 hotfix SOP（< 10 分钟内完成）：
1. 在专用 hotfix branch 上改 manifest，不要 review
2. argocd app set <app> --revision=hotfix/...
3. argocd app sync <app>
4. 修完事故后 24h 内：
   - hotfix branch 提 PR 到 main，标 [HOTFIX]
   - review + merge
   - argocd app set <app> --revision=HEAD
5. 事故复盘必须包含"为什么这个改动不能走正常流程"
```

关键是 **流程上认可"绕过"是合法的，但要可追溯**。

#### 验证

```bash
# 模拟事故场景
kubectl edit deployment my-app -n prod
# 改 image tag
sleep 10
kubectl get deployment my-app -n prod -o jsonpath='{.spec.template.spec.containers[0].image}'
# 期望：被 selfHeal 改回去

# 走 hotfix 流程
git checkout -b hotfix-test
# 改 manifest
git push
argocd app set my-app --revision=hotfix-test
argocd app sync my-app
sleep 30
kubectl get deployment my-app -n prod -o jsonpath='{.spec.template.spec.containers[0].image}'
# 期望：是新 image（没被改回去，因为 Git 里就是新值）
```

> **口诀**：selfHeal 对 GitOps 是好的，对凌晨值班是要命的。流程上必须有"绕过"模式：临时关 selfHeal、或 sync hotfix branch。事后必须补 PR。

---

### 案例 ④ Sealed Secrets 密钥泄漏 —— "Git 公开了那一刻"

#### 现象

某天发现公司一个内部 Git 仓库被错误地设为 public 了一周。仓库里有几百个 SealedSecret。立即问：**这些密钥安全吗**？

#### 根因

SealedSecret 用集群侧的非对称密钥加密，**只要私钥没泄漏，密文公开是安全的**。但实际情况复杂：
1. 集群里的 Sealed Secrets controller 私钥 —— 从未轮换过
2. 集群升级时被备份到了一个 S3 bucket
3. 那个 S3 bucket 的访问日志显示，过去一年有 5 个外部 IP 访问过

虽然没明确证据私钥被复制走，但**安全性已经不能保证**。所有密钥必须当作泄漏处理。

#### 修复

**短期（24 小时内）**：

```bash
# 1. 把所有受影响的 Secret 全部轮换（DB 密码、API key、TLS 证书）
# 这是真正的工作量，跟密钥总数线性相关

# 2. 重新生成 Sealed Secrets controller 的密钥对
kubectl -n kube-system delete secret -l sealedsecrets.bitnami.com/sealed-secrets-key=active
kubectl -n kube-system rollout restart deployment sealed-secrets-controller
# Controller 重启后会生成新密钥对

# 3. 拉新公钥
kubeseal --fetch-cert > new-cert.pem

# 4. 用新密钥重新加密所有 Secret
for f in $(find . -name 'sealed-*.yaml'); do
  kubeseal --re-encrypt --cert new-cert.pem -f $f -o yaml > $f.new
  mv $f.new $f
done
git add . && git commit -m "Re-encrypt all SealedSecrets after key rotation"
```

**长期**：

**方案 A**：定期轮换 Sealed Secrets controller 密钥（建议 90 天一次）。Bitnami 文档有 [key renewal](https://github.com/bitnami-labs/sealed-secrets#secret-rotation) 流程。

**方案 B**：迁移到 SOPS + KMS。云厂商 KMS 支持自动轮换密钥版本，比 Sealed Secrets 的"密钥固定 + 重加密"运维负担小很多。

**方案 C**：迁移到 External Secrets + Vault。Git 里只存"引用"，泄漏 Git 不会泄漏密钥本身。这是最高安全等级的方案，但运维复杂度也最高。

#### 应急预案文档（必须提前写）

```
## SealedSecrets 密钥泄漏应急预案

触发条件：
- Git 仓库异常公开
- Sealed Secrets controller 所在集群被入侵
- 备份私钥的存储被未授权访问

立即行动（< 1 小时）：
1. 评估泄漏窗口（多长时间、哪些密钥可能被访问）
2. 立即轮换"高敏感"密钥（生产 DB、支付 API、TLS 证书）
3. 通知 SRE / 安全团队 / 业务负责人

24 小时内：
4. 轮换所有受影响 Secret 的明文值
5. 生成新的 controller 密钥对
6. 重加密所有 SealedSecret
7. PR + review + sync

事后：
8. 复盘报告
9. 评估是否迁移到 SOPS / Vault
```

#### 验证

```bash
# 验证新密钥生效
echo "test123" | kubectl create secret generic test --dry-run=client \
  --from-file=key=/dev/stdin -o yaml | \
  kubeseal --cert new-cert.pem -o yaml > test-sealed.yaml

kubectl apply -f test-sealed.yaml
kubectl get secret test -o jsonpath='{.data.key}' | base64 -d
# 期望：能正确解密为 test123

# 验证旧密钥已清理
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key
# 期望：只有新密钥，老密钥已 delete
```

> **口诀**：SealedSecret 的安全性 = 私钥的安全性。**私钥必须定期轮换，必须有应急预案，最好别用在公网仓库**。

---

### 案例 ⑤ 多集群版本不一致 —— "us prod 是 v1.2.3，eu prod 是 v1.2.2"

#### 现象

某天发现 us prod 和 eu prod 的某服务版本不一致。监控里 us 的某个指标看不到，eu 的能看到。最终发现 eu 的部署比 us 慢了一个版本。

#### 根因

Application 配置：

```yaml
spec:
  source:
    targetRevision: HEAD               # ★ HEAD 不是固定的
```

`HEAD` 是 git 引用，**永远指向最新 commit**。但每个 ArgoCD 实例去拉的时机不一样：
- us ArgoCD 在 commit C1 push 后 5 秒拉到了 C1
- eu ArgoCD 网络拥塞，30 秒后才拉到，但那时已经有 C2 了
- 结果两个集群拉到不同 commit

更糟的：webhook 触发 sync 时，每个 ArgoCD 独立处理，**不保证同时 sync**。

#### 修复

**方案 A**：targetRevision 用固定 tag/commit，而不是 HEAD。

```yaml
spec:
  source:
    targetRevision: v1.2.3            # 固定 tag
    # 或
    targetRevision: a1b2c3d4         # 固定 commit
```

部署流程改为：
1. CI 打 image tag
2. CI 打 git tag（与 image tag 同名）
3. ArgoCD Application 引用这个 tag

**方案 B**：用 ApplicationSet 强制多集群一致。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: order-multi-region
spec:
  syncPolicy:
    preserveResourcesOnDeletion: true  # ApplicationSet 删了不删 Application
  generators:
    - clusters:
        selector:
          matchLabels:
            env: prod
  template:
    metadata:
      name: 'order-{{name}}'
    spec:
      source:
        repoURL: https://github.com/org/manifests
        path: services/order/overlays/prod
        targetRevision: v1.2.3        # ★ 所有集群同一个 revision
```

ApplicationSet 模板里指定 revision，确保所有目标集群的 Application 都引用同一个版本。

**方案 C**：渐进式发布（Argo Rollouts + Argo Workflows）。

```
us-canary (10%) → 等 5 分钟 SLO → us-full (100%) → eu-canary → eu-full
```

每一步都是显式的、可观测的，不会出现"一个先一个后"的隐式不一致。

#### 验证

```bash
# 检查所有 prod 集群的版本
for cluster in prod-us prod-eu; do
  kubectl --context=$cluster -n payments get deployment order-service \
    -o jsonpath='{.spec.template.spec.containers[0].image}'
  echo
done
# 期望：所有集群打印同一个 image tag
```

写一个 smoke test：

```bash
#!/bin/bash
# version-consistency-check.sh
set -e
EXPECTED=$(cat manifests/services/order/overlays/prod/kustomization.yaml | yq '.images[0].newTag')
for cluster in prod-us prod-eu prod-ap; do
  ACTUAL=$(kubectl --context=$cluster -n payments get deployment order \
    -o jsonpath='{.spec.template.spec.containers[0].image}' | awk -F: '{print $2}')
  if [ "$ACTUAL" != "$EXPECTED" ]; then
    echo "MISMATCH: $cluster has $ACTUAL, expected $EXPECTED"
    exit 1
  fi
done
```

接到 CI 里，每天跑一次。

> **口诀**：targetRevision 用 HEAD 是定时炸弹。**多集群 prod 必须用固定 tag**，配 smoke test 验证一致性。

---

### 案例 ⑥ ApplicationSet 模板渲染错误 —— "{{path.basename}} 渲染成空"

#### 现象

升级 ArgoCD 从 2.5 到 2.7，所有 ApplicationSet 突然生成的 Application 名字不对：

```
Before: order-prod, payment-prod, ...
After:  -prod, -prod, -prod  (重名冲突)
```

#### 根因

ArgoCD 2.6 引入 go template 支持，需要在 ApplicationSet 上显式开 `goTemplate: true`。**默认仍然是 fasttemplate**（旧引擎）。

但旧的 fasttemplate 有限制：不支持 `index`、`sprig` 函数、嵌套字段访问。从 2.6 开始 fasttemplate 被认为是 deprecated，且某些场景下行为变化。

升级到 2.7 后，fasttemplate 对 `{{path.basename}}` 的解析在某些 generator 上行为变了 —— path 变量结构改了。

#### 修复

**方案 A**：显式启用 go template + 重写模板。

```yaml
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]    # 缺字段时报错而不是空字符串
  template:
    metadata:
      name: '{{.path.basename}}-{{.env}}'    # 注意要加点号
```

`missingkey=error` 是关键 —— 默认 go template 缺字段静默渲染成空字符串，导致案例里的现象。设为 error 后会立即失败，比静默错误好得多。

**方案 B**：迁移期保留 fasttemplate，但锁版本。

```yaml
spec:
  # goTemplate: false（默认）
  template:
    metadata:
      name: '{{path.basename}}-{{values.env}}'
```

注意 fasttemplate 不支持 `.path.basename` 写法，必须 `{{path.basename}}`。

**方案 C**：用 ApplicationSet 的 dry-run 模式验证。

```bash
# 用 argocd CLI dry-run
argocd appset get my-appset --output json | jq '.spec.template'

# 或直接看生成的 Application
kubectl get applications -n argocd -l argocd.argoproj.io/application-set-name=my-appset
```

#### 升级前必须做的 checklist

```
ArgoCD 主版本升级 checklist：
1. 通读 release notes 找 Breaking Changes
2. 在 staging 环境完整复现现有 ApplicationSet
3. 用 --dry-run 验证生成的 Application 数量、名字、内容
4. 准备回滚方案（保存当前 Helm values + 备份 Application CR）
5. 升级期间禁用 selfHeal（避免老版本 sync 老配置）
```

#### 验证

```bash
# 看 ApplicationSet controller 日志
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-applicationset-controller

# 检查生成的 Application 名字
kubectl get applications -n argocd
# 期望：名字符合 <service>-<env> 格式，没有空字段

# 用 missingkey=error 触发：故意删掉一个字段
# 期望：ApplicationSet status 报 template error，不是静默生成错误的 Application
```

> **口诀**：ApplicationSet 升级是 ArgoCD 升级最大的风险点。开 `goTemplate: true` + `missingkey=error`，不要靠静默渲染。

---

### 案例 ⑦ ArgoCD 自身升级风险 —— "管 ArgoCD 的 ArgoCD 把自己改坏了"

#### 现象

用 app-of-apps 模式让 ArgoCD 管理自己的安装。某次升级 ArgoCD chart 从 5.46.0 到 5.51.0，Helm template 渲染时新版本对某些字段做了破坏性变更，新的 ArgoCD Pod CrashLoopBackOff。但**改 ArgoCD 配置的 ArgoCD 也是这个 ArgoCD**，没法用 ArgoCD 改回去。

#### 根因

GitOps 自管自的"自举悖论"：管 X 的 GitOps 工具是 X 自己。X 挂了，连"修改 X 让它别挂"的能力都没了。

```
Bootstrap-ArgoCD ──manages──► Production-ArgoCD ──manages──► All other apps
                                       │
                                       └──manages──► itself  (循环依赖)
```

如果 Production-ArgoCD 自己挂了，要么手动 kubectl 改、要么有一个独立的 Bootstrap-ArgoCD 救场。

#### 修复

**方案 A**：双 ArgoCD 互管。

```
Cluster A: argocd-a   ←── managed by argocd-b
Cluster B: argocd-b   ←── managed by argocd-a
```

任何一个挂了，另一个能救。

**方案 B**：保留手动 bootstrap 能力。把 ArgoCD 安装 manifest（Helm template 渲染后的）也提交到 Git，应急时 kubectl apply 直接装：

```bash
# 渲染当前版本的 manifest 备份
helm template argocd argo/argo-cd -f values.yaml > backup/argocd-manifest.yaml
git add backup/argocd-manifest.yaml
git commit -m "Backup current ArgoCD manifest"

# 应急恢复
kubectl apply -f backup/argocd-manifest.yaml
```

**方案 C**：升级 ArgoCD 时用蓝绿。

```bash
# 装一个 argocd-staging
helm install argocd-staging argo/argo-cd --version 5.51.0 -n argocd-staging

# 在 staging 验证 N 天
# 通过后，把 production 切到 5.51.0
helm upgrade argocd argo/argo-cd --version 5.51.0 -n argocd
```

**方案 D**：版本固定 + 谨慎升级 SOP。

```yaml
# 不要用 chart 的 latest
helm:
  chart: argo-cd
  version: "5.51.0"          # 固定 minor
  # 不要写 5.x 或 5.51.*
```

#### 升级 SOP

```
ArgoCD 升级 SOP：
1. 读 release notes 找 Breaking Changes
2. 在 staging 集群升级 + 验证 1 周
3. 在 prod 升级前：
   a. helm template 当前版本，commit 到 git 作为 fallback
   b. 备份 ConfigMap (argocd-cm, argocd-rbac-cm, argocd-secret)
   c. 备份所有 Application + ApplicationSet
4. helm upgrade --atomic --timeout 10m
5. 升级后 5 分钟内观察 controller 日志
6. 应急回滚：helm rollback argocd <previous-revision>
```

#### 验证

```bash
# 升级后必须验证的事
# 1. ArgoCD 自己 healthy
kubectl get pods -n argocd
# 全部 Running

# 2. 能正常 sync 一个测试 Application
argocd app sync test-app

# 3. UI 能登录
curl -k https://argocd.example.com/api/version

# 4. ApplicationSet 没生成莫名其妙的 Application
kubectl get applications -n argocd | wc -l
# 数量应该和升级前一致
```

> **口诀**：管 ArgoCD 的 ArgoCD 是自举循环。要么双互管，要么留手动 bootstrap 备份，要么蓝绿升级。任何一个，不能没有。

---

### 案例 ⑧ Repo Server 内存爆炸 —— "OOM killer 把 ArgoCD 揍死"

#### 现象

ArgoCD 管理的 Application 数从 200 涨到 1500，Repo Server 频繁 OOM。Pod restart 后又涨上来。日志看到很多 `git clone` 卡住的栈。

#### 根因

Repo Server 的内存吃在三处：
1. **Git clone 本地副本**：每个 repo 一份，多 branch / tag 切换时还会增长
2. **Helm template / Kustomize build 缓存**：每个 Application 一份渲染结果
3. **并行渲染**：默认最多 50 个 Application 并行渲染，每个高峰可能用 200MB

1500 Application × 平均 50MB 缓存 = 75GB —— 单 Pod 当然扛不住。

更深的：Repo Server 默认 git 仓库存在 PVC 上，但**默认没有 PVC，只用 emptyDir**。Pod 重启后所有缓存丢失，第一波启动会**集体重新 git clone**，瞬间 IO + CPU 爆炸。

#### 修复

**第一步**：水平扩展 Repo Server。

```yaml
repoServer:
  replicas: 6                            # 多副本，自动负载均衡
  resources:
    requests: { memory: 2Gi, cpu: 500m }
    limits:   { memory: 8Gi, cpu: 2000m }
```

**第二步**：限制并行渲染。

```yaml
repoServer:
  env:
    - name: ARGOCD_GIT_MODULES_ENABLED
      value: "false"
    - name: ARGOCD_REPO_SERVER_PARALLELISM_LIMIT
      value: "10"                        # 默认 50 → 10
    - name: ARGOCD_EXEC_TIMEOUT
      value: "180s"
```

**第三步**：用 PVC 持久化 git cache。

```yaml
repoServer:
  volumes:
    - name: gpg-keys
      emptyDir: {}
    - name: git-credentials
      emptyDir: {}
  volumeMounts:
    - name: argocd-repo-server-cache
      mountPath: /tmp/_argocd-repo
  persistence:
    enabled: true
    storageClass: gp3
    size: 50Gi
```

**第四步**：清理无用的 Helm values 缓存。配置 `argocd-cm`：

```yaml
data:
  reposerver.parallelism.limit: "10"
  helm.repositories: |              # 限制只能用配置过的 chart repo
    - url: https://charts.bitnami.com/bitnami
    - url: https://prometheus-community.github.io/helm-charts
```

**第五步**（重要）：拆 ArgoCD。如果 Application 数 > 2000 且增长还快，单 ArgoCD 实例已经不合适，应该按业务/region 拆多个 ArgoCD。

#### 监控指标

```promql
# Repo Server 内存使用率
container_memory_working_set_bytes{namespace="argocd",pod=~"argocd-repo-server.*"}
  / container_spec_memory_limit_bytes{namespace="argocd",pod=~"argocd-repo-server.*"}

# Manifest 渲染耗时
histogram_quantile(0.99,
  sum by (le) (rate(argocd_git_request_duration_seconds_bucket[5m])))
```

告警：
- 内存使用率 > 80% 持续 10 分钟
- P99 渲染耗时 > 30s

#### 验证

```bash
# 模拟大量并发 sync
for app in $(argocd app list -o name | head -100); do
  argocd app sync $app --async &
done
wait

# 观察 Repo Server pod
kubectl top pods -n argocd -l app.kubernetes.io/name=argocd-repo-server
# 期望：单 pod 内存不超过 limit 的 80%

# 检查 OOM 历史
kubectl get events -n argocd --field-selector reason=OOMKilling
# 期望：无新增
```

> **口诀**：Repo Server 是 ArgoCD 最容易爆的组件。多副本 + 并行限制 + PVC 缓存 + 监控告警，四件套缺一不可。

---

### 案例 ⑨ 紧急修复绕过 —— "GitOps 流程的合法'后门'"

#### 现象

线上事故：某接口 P99 飙到 30s。根因是某个配置项写错。正常 GitOps 流程要走 PR → review → merge → sync，全套走完最快 15 分钟。但事故每分钟在烧钱（支付订单失败）。值班想直接 `kubectl edit` 改完先恢复服务。

但前面案例 ① 和 ③ 已经讲了：**生产不能 kubectl edit**。怎么办？

#### 根因（不是 bug，是流程）

GitOps 强调"所有变更走 Git"，这在常规情况下是对的，但**事故场景下"快速止损"和"流程合规"会冲突**。如果工具/流程不允许快速止损，团队会被迫自己想"野路子"（隐藏的 break-glass 账号、绕过 ArgoCD 直接 kubectl），反而更危险。

#### 修复（流程设计 + 工具支持）

**第一步**：定义合法的"紧急修复模式"。

```
Severity 1 事故定义：
- 影响生产业务核心链路
- 损失 > $1000/分钟
- 持续 > 5 分钟未恢复

合法绕过措施：
- 可以 sync 到 hotfix branch（不需要 review）
- 可以 argocd app set --sync-policy=none 临时关 selfHeal
- 但不可以 kubectl edit / kubectl patch 生产资源
- 24 小时内必须补 PR + 复盘
```

**第二步**：工具支持紧急 sync。

```bash
# 紧急 sync 到任意 commit
argocd app sync my-app --revision=abc1234 --force

# 或用 hotfix branch
git checkout -b hotfix/SEV1-payment-config
# 改 manifest
git commit -m "[HOTFIX] fix payment timeout"
git push origin hotfix/SEV1-payment-config

# 不走 PR，直接 sync 到这个 branch
argocd app set payment-prod --revision=hotfix/SEV1-payment-config
argocd app sync payment-prod --force

# 等服务恢复后
# 1. PR 把 hotfix branch merge 到 main
# 2. argocd app set payment-prod --revision=HEAD
# 3. 写复盘
```

**第三步**：审计 + 通知。所有"绕过"操作必须有审计记录 + 实时通知。

```yaml
# argocd-cm
data:
  audit.policy: |
    - level: Metadata
      verbs: ["sync", "rollback", "override"]
      users: ["sre-oncall"]
      apiGroups: ["argoproj.io"]
      resources: ["applications"]
```

通知：

```yaml
# argocd-notifications-cm
data:
  trigger.on-emergency-sync: |
    - description: Emergency sync (out of normal flow)
      send: [emergency-sync-alert]
      when: |
        app.spec.source.targetRevision matches '^hotfix/.*' or
        sync.metadata.user == 'sre-oncall'

  template.emergency-sync-alert: |
    message: |
      🚨 EMERGENCY SYNC by {{.user}}
      App: {{.app.metadata.name}}
      Revision: {{.app.spec.source.targetRevision}}
      Time: {{.time}}
    slack:
      channel: incident-bridge
```

任何紧急 sync 实时进 incident channel，PMO 知道、安全知道、复盘有依据。

**第四步**：演练。每季度跑一次紧急修复演练，确保 SOP 真能用。

#### 验证

```bash
# 演练：模拟事故，全程计时
date
# 1. 发现问题
# 2. 决策走紧急流程
# 3. 创建 hotfix branch
# 4. argocd sync
# 5. 服务恢复
date
# 期望：< 5 分钟从决策到服务恢复

# 演练后自动检查
# - hotfix PR 是否 24h 内 merge
# - 复盘文档是否提交
# - 审计日志是否完整
```

> **口诀**：GitOps 不能僵化。**正常情况严格走 Git，事故情况合法绕过 + 事后补 PR + 强审计**。流程必须支持快速止损，否则团队会自己想坏办法。

---

## 5. 上线检查清单

打印贴墙 / 加进 ArgoCD 平台准入清单。

### 安装层
- [ ] 用官方 Helm chart 部署，不用 install.yaml
- [ ] App Controller / Repo Server / API Server 都开 HA（至少 2 副本）
- [ ] Redis 用 redis-ha，不用单 Pod
- [ ] 资源 limit 显式设置（参考案例 ⑧ 的数值）
- [ ] Ingress 启用 TLS，证书自动轮换（cert-manager）

### 安全层
- [ ] SSO 接入（OIDC/SAML），禁用 admin 密码
- [ ] RBAC `policy.default: role:readonly`
- [ ] AppProject 严格限制 sourceRepos / destinations / clusterResourceWhitelist
- [ ] 不让任何 Application 用 default Project（生产）
- [ ] 集群 kubectl 写权限通过 RBAC 收紧（防止漂移源头）

### 同步层
- [ ] 生产 Application 默认开 selfHeal + prune
- [ ] 但有定义的"紧急修复 SOP"允许临时关 selfHeal
- [ ] targetRevision 用固定 tag/commit，不用 HEAD
- [ ] CRD 用 sync-wave 排在 CR 前
- [ ] Webhook 配置（GitHub/GitLab）替代 polling

### 多集群层
- [ ] < 20 集群用 hub-spoke，> 20 或跨 region 用 federation
- [ ] ApplicationSet 模板用 `goTemplate: true` + `missingkey=error`
- [ ] 多集群部署用 ApplicationSet 强制版本一致
- [ ] 集群 label 规范化（env/region/team），ApplicationSet 用 label 选择

### 密钥层
- [ ] Sealed Secrets 私钥定期轮换（建议 90 天）
- [ ] 私钥有备份 + 应急预案文档
- [ ] 高敏感场景用 SOPS + KMS 或 External Secrets + Vault

### 监控层
- [ ] Prometheus 接入 ArgoCD metrics
- [ ] 告警：Application 持续 OutOfSync > 5 分钟
- [ ] 告警：Sync 失败、Repo Server OOM、Application Controller CPU > 80%
- [ ] 紧急 sync 操作实时通知 incident channel
- [ ] 部署成功/失败有 Slack/钉钉通知

### 升级层
- [ ] ArgoCD 版本固定 minor
- [ ] 升级前在 staging 验证 1 周
- [ ] 升级前备份当前 Helm template 到 Git 作为 fallback
- [ ] ArgoCD chart 升级在事故回滚 SOP 里

### 自管自层
- [ ] ArgoCD 自身用 app-of-apps 管理
- [ ] 但有 Bootstrap 备份 manifest 应急（防自举死锁）
- [ ] 升级 ArgoCD 自身用 helm 而不是 ArgoCD（避免循环）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是 ArgoCD 的入门。再深一层的方向：

### Flux v2 / GitOps Toolkit
- Source Controller / Kustomize Controller / Helm Controller / Notification Controller 各自职责
- ImagePolicy + ImageUpdateAutomation：image tag 变化自动 PR
- 与 ArgoCD 的最大差异：**Flux 没有"中心化"，每集群独立**
- 推荐场景：多 tenant 强隔离 / 不需要 UI 的 platform team

### Crossplane 基础设施 GitOps
- 用 K8s CRD 描述云资源（RDS、S3、IAM）
- ArgoCD/Flux 同步 Crossplane CR → 实际创建云资源
- 真正的"基础设施 + 应用同一套 GitOps"
- 坑：Provider 升级、CRD 版本兼容、StateLocking

### Argo Rollouts（渐进式部署）
- Canary / Blue-Green / 实验组的 K8s 原生实现
- 与 ArgoCD 集成：sync 后由 Rollouts controller 接管发布过程
- 集成 Prometheus 做 SLO-driven 自动判断 promote / abort
- 替代 Spinnaker / Flagger 的轻量方案

### Argo Workflows（K8s 原生工作流）
- DAG / steps 描述复杂部署流程
- 比 Tekton 更"应用导向"
- 与 ArgoCD 的关系：Rollouts/Workflows 是 sync 后的"动作"，ArgoCD 是 sync 本身

### Backstage 平台工程
- 把 ArgoCD/Flux 包装成"自助服务门户"
- 业务团队不直接接触 ArgoCD UI，通过 Backstage 模板化创建 Application
- 配套：Software Templates、TechDocs、Catalog

### Open Policy Agent + GitOps
- ArgoCD admission webhook 用 OPA Rego 验证 Application
- 例：禁止 Application 使用 latest tag、强制资源必须有 limits、强制必须有 owner label
- ConfTest 在 CI 阶段就校验，避免错误进集群

### GitOps 与 Trunk-based Development
- 单一 main branch + 短生命周期 feature branch
- 与 GitOps 的协作：每个 PR 起 preview 环境（PR Generator）
- Feature flag 替代 long-lived branch
- 推荐配套：LaunchDarkly / Unleash + GitOps

### Renovate / Dependabot 自动化
- 自动 PR 升级 chart 版本、image tag、依赖
- 配合 ArgoCD ApplicationSet 实现"全集群升级"自动化
- 与 image automation 的边界

---

## 7. 引用

- [OpenGitOps Principles 1.0](https://opengitops.dev/) —— GitOps 4 原则的形式化定义
- [ArgoCD 官方文档](https://argo-cd.readthedocs.io/)
- [Flux 官方文档](https://fluxcd.io/flux/)
- [ApplicationSet 文档](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
- [Bitnami Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [Mozilla SOPS](https://github.com/getsops/sops)
- [External Secrets Operator](https://external-secrets.io/)
- [Argo Rollouts](https://argo-rollouts.readthedocs.io/)
- [Crossplane](https://www.crossplane.io/)
- [Backstage](https://backstage.io/)
- *Patterns of Distributed Systems* —— Martin Kleppmann（理论补充）
- Weaveworks 原始论文：*GitOps: Operations by Pull Request*（2017）

---

> 配套：[Kubernetes 实战](/article/k8s/k8s-essentials) | [DevOps CI/CD](/article/devops/cicd) | [Git 工作流](/article/devops/git-workflow)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果觉得这篇深度对，告诉我下一篇该写哪个 GitOps 周边主题（Flux v2 / Argo Rollouts / Crossplane / Backstage）。
