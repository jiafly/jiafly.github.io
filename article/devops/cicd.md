# DevOps 与 CI/CD 流水线 —— 从 Jenkins 到 GitOps 的工程实战

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试，看 [DevOps 综述](/article/devops/devops) 那一节就够了。
> 这里要回答的，是**真的把流水线搭起来、半夜被「上线失败」叫起来排查**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. 你说自己在做"持续部署"，那 **Continuous Integration / Continuous Delivery / Continuous Deployment** 这三个词，你能用一句话各自精确定义吗？哪个是必须人审、哪个不需要？
2. 公司新项目要选 CI 系统：Jenkins / GitLab CI / GitHub Actions / Tekton / ArgoCD —— 它们里面**只有一个不是 CI** 你知道是哪个吗？为什么经常被混在一起讨论？
3. 你写 Dockerfile 用了多阶段构建（multi-stage build），最终镜像从 1.2GB 降到 80MB —— **真正起作用的机制**是什么？仅仅是"少 copy 几个文件"吗？
4. BuildKit 的 `--mount=type=cache` 和传统的 Docker 层缓存（layer cache）解决的是同一个问题吗？前者为什么能加速 `npm install` / `mvn package` 而 layer cache 不行？
5. 蓝绿、金丝雀、滚动 —— 这三种部署策略，**哪一种回滚最快**？哪一种最容易出现"两个版本同时存在导致数据库脏数据"？
6. GitOps（ArgoCD / Flux）说自己用的是 **pull 模型**，传统 CD（Jenkins push 到 K8s）用的是 push 模型。这两者除了"谁主动"以外，**在权限模型、审计、灾备**上有什么本质差异？
7. Helm 和 Kustomize 都能管 K8s 配置，区别在"模板 vs overlay"。但实际生产里，**为什么大公司经常两者并用**？纯 Helm 或纯 Kustomize 各有什么坑？
8. 你的流水线已经全绿，集成测试通过、镜像推到仓库、部署也完成了 —— 但**生产用户立即报错**。哪三个环节最容易在"流水线全绿"的情况下漏掉？

如果有半数答不上，往下看。

---

## 1. 底层模型：流水线、制品、部署策略

### 1.1 CI / CD / CD 三件事

这三个词被混用得太厉害了。先把语义钉死：

| 缩写 | 全称 | 触发频率 | 是否人审 | 输出 |
| --- | --- | --- | --- | --- |
| **CI** | Continuous Integration | 每次 push | 否 | 通过测试的 commit + 制品（artifact） |
| **CD₁** | Continuous **Delivery** | 每次 CI 通过 | **是**（prod 前必须） | 随时可上线的镜像 + 部署描述 |
| **CD₂** | Continuous **Deployment** | 每次 CI 通过 | **否** | 自动部署到生产 |

CI 解决"代码合并即破坏"的问题；CD₁ 解决"明明能上线但要等一周才上"的问题；CD₂ 是 CD₁ 的极端化。

绝大多数公司号称自己做"CI/CD"，**实际只做了 CI + 半自动 Delivery**。Continuous Deployment 需要极强的测试 + 监控 + 回滚能力支撑，金融和医疗几乎不可能做到。

> **口诀**：CI 是"代码到制品"，Delivery 是"制品随时可上"，Deployment 是"制品自动上"。三件事 ≠ 一件事。

### 1.2 流水线的 6 个阶段

绝大多数现代流水线都能拆成下面 6 段：

```
commit → build → test → image → deploy → verify
   │        │       │       │        │         │
   │        │       │       │        │         └─ 健康检查 / 烟测 / SLO 校验
   │        │       │       │        └─ 蓝绿 / 金丝雀 / 滚动
   │        │       │       └─ Docker build + push registry
   │        │       └─ 单测 / 集成 / 静态分析 / 安全扫描
   │        └─ 编译 / 打包（jar、wheel、binary）
   └─ Webhook / cron / 手动触发
```

每一段都可能成为整条流水线的瓶颈。**90% 的"流水线慢"问题都集中在 test 和 image 两段**（案例 ① 和 ② 会展开）。

### 1.3 制品（Artifact）管理

"制品"是 CI 的输出物，常见三类：

| 类型 | 例子 | 仓库 |
| --- | --- | --- |
| 二进制 | jar / war / wheel / .so | Nexus / Artifactory / GitHub Packages |
| 镜像 | OCI image | Harbor / ECR / GCR / Docker Hub |
| 配置 / Helm chart | values.yaml / chart.tgz | Helm repo / OCI registry / Git |

**制品必须不可变（immutable）**。同一个 tag 不能指向不同内容 —— 这是后面 GitOps 能成立的前提。

错误模式：直接用 `latest` tag 部署。**永远不要这么做**。原因后面案例 ③ 会讲。

### 1.4 部署策略：蓝绿 / 金丝雀 / 滚动 / Feature Flag

| 策略 | 一图概括 | 资源开销 | 回滚速度 | 数据库兼容性要求 |
| --- | --- | --- | --- | --- |
| **滚动（rolling）** | N 个 Pod 一批批替换 | 低（仅多 1 个 Pod） | 慢（要再滚一次） | 中（双版本共存） |
| **蓝绿（blue/green）** | 老的全留着，新的全起好，切流量 | 高（双倍资源） | **极快**（切回 service） | 低（可瞬间切） |
| **金丝雀（canary）** | 先开 1% 流量给新版，逐步放量 | 低（多 1~2 个 Pod） | 快（删 canary） | **高**（双版本长期共存） |
| **Feature Flag** | 代码已部署，功能开关控制 | 0 | **瞬间**（关 flag） | 取决于实现 |

口诀：**蓝绿吃资源换速度，金丝雀吃复杂度换风险，滚动吃时间换简单**。Feature Flag 是部署与发布解耦的终极武器，但需要业务代码深度配合。

后面案例 ④ 会展开"金丝雀流量切不动"的真实事故。

### 1.5 GitOps 的 pull 模型

传统 CD（Jenkins push）：

```
CI → Jenkins → kubectl apply → K8s API Server
                   ▲
                 持有 kubeconfig（高权限）
```

GitOps（ArgoCD / Flux pull）：

```
CI → 改 Git 仓库的 yaml → 提 PR → 合入
                                  │
                          ArgoCD 监听 Git
                                  │
                          (集群内) 主动拉取并 apply
```

差异不仅是"谁主动"。本质差异：

| 维度 | push（Jenkins） | pull（ArgoCD） |
| --- | --- | --- |
| **权限** | CI 系统持有集群凭证 | 集群自己持有 Git 凭证（只读） |
| **审计** | Jenkins 日志（容易丢） | Git 历史（永久、可签名） |
| **真相源** | 不明确（可能是镜像、可能是 Helm release） | **Git 仓库唯一真相源** |
| **漂移检测** | 没有 | ArgoCD 持续 reconcile，发现漂移 |
| **灾备** | 重建要重新配 Jenkins job | 重建集群 + 指 Git → 自动恢复 |

GitOps 的"声明式 + Git 真相源"是它压倒 push 模型的核心优势。

### 1.6 IaC：Terraform / Pulumi / CloudFormation

CI/CD 解决的是"应用层"的部署。但应用跑在哪？数据库、负载均衡、VPC、Kubernetes 集群本身怎么来？

| 工具 | 语法 | 状态管理 | 多云支持 |
| --- | --- | --- | --- |
| **Terraform** | HCL（DSL） | tfstate 文件（远程后端） | 全 |
| **Pulumi** | TypeScript / Python / Go | Pulumi 服务 / S3 | 全 |
| **CloudFormation** | JSON / YAML | AWS 内部管理 | 仅 AWS |
| **CDK** | TS/Py/Java，编译成 CFN | AWS 内部 | 仅 AWS |

IaC 的核心是 **"声明式 + 状态机"**。你写"我要 3 个 EC2 实例"，工具对比当前状态（state）和目标状态（desired），算出 diff（plan），然后 apply。

> **口诀**：CI/CD 管"应用上线"，IaC 管"基础设施成型"。两者都是声明式 + 状态机 + Git 驱动 → **整个 DevOps 是同一种思想的不同面**。

---

## 2. 工具切片：能跑的真实代码

### 2.1 GitHub Actions YAML 设计

GitHub Actions 是当下中小项目最快的入门工具。一个生产级 Java 项目的完整流水线：

```yaml
name: ci

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

# 关键：并发控制，同一个分支新 push 取消旧的
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

env:
  IMAGE: ghcr.io/${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    services:
      postgres:
        image: postgres:16-alpine
        env: { POSTGRES_PASSWORD: test }
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports: ['5432:5432']
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # 全历史，给 SonarQube / 增量分析用

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'   # ★ 自动缓存 ~/.m2

      - name: Run tests
        run: mvn -B verify -Dspring.profiles.active=ci

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: target/site/jacoco/

  build-image:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
      id-token: write   # ★ for OIDC keyless signing
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3   # ★ 启用 BuildKit

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE }}:${{ github.sha }}
            ${{ env.IMAGE }}:latest
          cache-from: type=gha   # ★ GitHub Actions cache backend
          cache-to: type=gha,mode=max
          provenance: true       # SLSA provenance
```

**关键点逐条点透**：

- `concurrency` + `cancel-in-progress`：同一个 PR 新 push 后旧的立即取消，省 CI 资源。**没配这个的项目 CI 排队是常态**。
- `services.postgres`：起一个真实 Postgres，比 H2 / Mock 更接近生产。`health-cmd` 必配，否则 job 还没等 DB ready 就开始跑测试。
- `cache: 'maven'`：actions/setup-java 内置的依赖缓存。**省 80% 的 mvn 时间**。
- `cache-from: type=gha` + `cache-to: type=gha,mode=max`：BuildKit 把构建缓存推到 GHA 自带的 cache 后端，跨 workflow 复用。

### 2.2 GitLab CI 的 pipeline / stages / artifacts

GitLab CI 的核心抽象是 stage 和 job：

```yaml
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_BUILDKIT: "1"
  IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

unit-test:
  stage: test
  image: maven:3.9-eclipse-temurin-21
  cache:
    key:
      files: [pom.xml]
    paths: [.m2/repository]
  variables:
    MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  script:
    - mvn -B test
  artifacts:
    when: always
    reports:
      junit: target/surefire-reports/TEST-*.xml
    paths: [target/]
    expire_in: 1 day

integration-test:
  stage: test
  image: maven:3.9-eclipse-temurin-21
  services:
    - postgres:16-alpine
  variables:
    POSTGRES_PASSWORD: test
    SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/postgres
  script:
    - mvn -B verify -Pintegration

build-image:
  stage: build
  image: docker:25
  services:
    - docker:25-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker buildx create --use
    - docker buildx build
        --cache-from type=registry,ref=$CI_REGISTRY_IMAGE:cache
        --cache-to type=registry,ref=$CI_REGISTRY_IMAGE:cache,mode=max
        --push
        --tag $IMAGE
        .
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

deploy-prod:
  stage: deploy
  image: alpine/k8s:1.30.0
  environment:
    name: production
    url: https://app.example.com
  script:
    - kubectl set image deployment/app app=$IMAGE -n prod
    - kubectl rollout status deployment/app -n prod --timeout=5m
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual   # ★ 手动触发，等同 Continuous Delivery 的语义
```

**点透**：
- `cache.key.files: [pom.xml]`：缓存 key 绑定 pom.xml 内容 hash —— pom 没改就一直用同一个缓存。
- `artifacts.reports.junit`：GitLab UI 会自动展示测试结果，PR 页面可见，省去翻日志。
- `services: docker:25-dind`：Docker-in-Docker。生产推荐改用 BuildKit 远程后端或 Kaniko（避免特权容器）。
- `rules.when: manual`：上 prod 必须手动确认 —— 这就是 Continuous Delivery 而非 Deployment。

### 2.3 Jenkinsfile 声明式流水线

老牌玩家。生产里仍大量存在，必须会写：

```groovy
pipeline {
  agent { label 'docker' }
  options {
    timeout(time: 30, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '50'))
    disableConcurrentBuilds()
    timestamps()
  }
  environment {
    IMAGE = "registry.internal/app:${env.BUILD_NUMBER}"
    SLACK_WEBHOOK = credentials('slack-webhook')
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Test') {
      parallel {
        stage('Unit') {
          steps { sh 'mvn -B test' }
          post {
            always { junit 'target/surefire-reports/*.xml' }
          }
        }
        stage('Lint') {
          steps { sh 'mvn -B spotbugs:check' }
        }
        stage('SCA') {
          steps { sh 'mvn -B org.owasp:dependency-check-maven:check' }
        }
      }
    }

    stage('Build & Push') {
      when { branch 'main' }
      steps {
        sh """
          docker buildx build \
            --cache-from type=registry,ref=registry.internal/app:cache \
            --cache-to type=registry,ref=registry.internal/app:cache,mode=max \
            --push \
            --tag $IMAGE \
            .
        """
      }
    }

    stage('Deploy Staging') {
      when { branch 'main' }
      steps {
        sh "kubectl -n staging set image deployment/app app=$IMAGE"
        sh "kubectl -n staging rollout status deployment/app --timeout=5m"
      }
    }

    stage('Approve Prod') {
      when { branch 'main' }
      steps {
        timeout(time: 24, unit: 'HOURS') {
          input message: 'Deploy to prod?', ok: 'Deploy'
        }
      }
    }

    stage('Deploy Prod') {
      when { branch 'main' }
      steps {
        sh "kubectl -n prod set image deployment/app app=$IMAGE"
        sh "kubectl -n prod rollout status deployment/app --timeout=10m"
      }
    }
  }

  post {
    failure {
      sh """curl -X POST -H 'Content-type: application/json' \
        --data '{"text":"❌ Build ${env.BUILD_URL} failed"}' \
        $SLACK_WEBHOOK"""
    }
  }
}
```

**点透**：
- `agent { label 'docker' }`：通过 label 选 agent —— Jenkins 多 agent 必须用 label 隔离，否则某些 job 会跑在没有 docker 的节点上失败。
- `parallel`：单测 / 静态分析 / SCA 可以并行跑，省一半时间。
- `disableConcurrentBuilds()`：同一分支同时只能跑一个 build —— 防止并发污染共享资源（如数据库迁移）。
- `input` 卡人审：`timeout` 必须配，否则 build 永远挂着占 executor。

### 2.4 Docker 多阶段构建优化

很多人写 Dockerfile：

```dockerfile
# ❌ 单阶段：jar 200MB + JDK 400MB + Maven 200MB + 操作系统 100MB ≈ 900MB
FROM maven:3.9-eclipse-temurin-21
WORKDIR /app
COPY . .
RUN mvn package
CMD ["java", "-jar", "target/app.jar"]
```

多阶段后：

```dockerfile
# 阶段 1：构建（含 Maven、源码）
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
# ★ 先 copy pom，单独 download 依赖 → 利用 layer 缓存
COPY pom.xml .
RUN mvn -B dependency:go-offline
COPY src ./src
RUN mvn -B package -DskipTests

# 阶段 2：运行（仅含 JRE + jar）
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
# ★ 只 copy 阶段 1 的产物，不带 Maven 也不带源码
COPY --from=builder /app/target/app.jar app.jar
# 非 root 运行
RUN addgroup -S app && adduser -S app -G app
USER app
EXPOSE 8080
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "app.jar"]
```

最终镜像：~ 180MB。

**真正起作用的机制**：
- 多阶段不是"少 copy 几个文件"，而是 **`FROM` 第二次出现时整个文件系统重置**。第二阶段从一个干净的 `eclipse-temurin:21-jre-alpine` 开始，Maven、源码、构建中间产物全都不存在 —— 它们在阶段 1 的临时镜像里，最终镜像不会包含。
- `COPY --from=builder` 只拷贝指定路径，相当于跨"两个独立镜像"做文件复制。

更进一步，用 distroless：

```dockerfile
FROM eclipse-temurin:21-jdk AS builder
# ... 同上构建步骤

FROM gcr.io/distroless/java21-debian12:nonroot
COPY --from=builder /app/target/app.jar /app.jar
USER nonroot
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

distroless 镜像没有 shell、没有包管理器、没有 ls，攻击面极小，常用于安全要求高的场景。

> **代价**：distroless 没法 `kubectl exec -it sh` 进容器排查。生产前要确保观测性（日志、metrics、trace）已经 cover 所有调试需求。

### 2.5 BuildKit 缓存挂载

传统 Dockerfile 的痛：每次改一行代码，`mvn package` 重头跑，下一万个依赖。

BuildKit 的 `--mount=type=cache`：

```dockerfile
# syntax=docker/dockerfile:1.7
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 \
    mvn -B package -DskipTests
```

`--mount=type=cache,target=/root/.m2`：把 `/root/.m2` 挂载到一个**跨 build 持久的缓存卷**。不会进最终镜像，但下次 build 直接复用。

适用场景：
- Maven / Gradle：`/root/.m2` 或 `/root/.gradle`
- npm / pnpm：`/root/.npm` 或 `/root/.pnpm-store`
- pip：`/root/.cache/pip`
- apt：`/var/cache/apt` + `/var/lib/apt`（注意 Debian 默认有 `/etc/apt/apt.conf.d/docker-clean`，要先删）

```dockerfile
RUN rm -f /etc/apt/apt.conf.d/docker-clean
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y curl
```

第一次构建：3 分钟。改一行 Java 代码后再构建：**20 秒**。这就是 BuildKit cache mount 的威力。

与 layer 缓存的区别：layer 缓存是"这一层指令完全相同就复用整层"，cache mount 是"层内的某个目录持久化"—— 能缓存"包管理器下载的依赖"这种"不属于任何文件系统层"的状态。

### 2.6 ArgoCD 的 Application + Project 模型

ArgoCD 的核心 CRD：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-prod
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # ★ 删 Application 时同步删集群资源
spec:
  project: prod
  source:
    repoURL: https://github.com/example/k8s-manifests
    targetRevision: main
    path: apps/my-app/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true        # 删除 Git 里没了的资源
      selfHeal: true     # 集群被人手动改了，自动改回 Git 状态
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
    retry:
      limit: 5
      backoff: { duration: 5s, factor: 2, maxDuration: 3m }
```

**关键决策点**：

- `automated.selfHeal: true`：集群里有人 `kubectl edit` 改了配置，ArgoCD 5 分钟内自动改回 Git 上的版本。**这是 GitOps 防漂移的核心机制**。但也意味着紧急修复必须走 Git PR（案例 ⑧ 会展开）。
- `automated.prune: true`：你从 Git 里删了一个 Service yaml，集群里对应的 Service 也被删。不开 prune 会导致 Git 与集群越发不一致。

Project 用来做权限隔离：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata: { name: prod, namespace: argocd }
spec:
  description: Production apps
  sourceRepos:
    - 'https://github.com/example/k8s-manifests'   # 限制只能从这个 repo 来
  destinations:
    - namespace: 'my-app'
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceBlacklist:    # 禁止部署 Secret（Secret 走 SealedSecret / ESO）
    - group: ''
      kind: Secret
  roles:
    - name: deployer
      policies:
        - p, proj:prod:deployer, applications, sync, prod/*, allow
      groups:
        - org:devops-team
```

### 2.7 Helm chart 的模板与 values

Helm 的核心是 **Go template + values 注入**。一个最小 chart：

```
my-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── _helpers.tpl
```

`templates/deployment.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  labels: {{- include "my-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels: {{- include "my-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels: {{- include "my-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            {{- range $k, $v := .Values.env }}
            - name: {{ $k }}
              value: {{ $v | quote }}
            {{- end }}
```

`values.yaml`（默认值）：

```yaml
replicaCount: 2
image:
  repository: ghcr.io/example/my-app
  tag: ""   # 空就用 Chart.AppVersion
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 500m, memory: 512Mi }
env:
  LOG_LEVEL: INFO
```

不同环境的 values：

```yaml
# values-prod.yaml
replicaCount: 10
resources:
  requests: { cpu: 500m, memory: 1Gi }
  limits:   { cpu: 2000m, memory: 2Gi }
env:
  LOG_LEVEL: WARN
```

部署：

```bash
helm upgrade --install my-app ./my-app \
  -n prod -f values-prod.yaml \
  --set image.tag=$GIT_SHA
```

Helm 的优势：参数化能力强、社区 chart 多（bitnami / prometheus-community）、有 release 生命周期管理。

劣势：模板嵌套深了难调试；values 覆盖路径多了易混乱；`helm rollback` 跨大版本经常出问题。

### 2.8 Kustomize 的 overlay

Kustomize 的哲学：**没有模板，只有 base + 多层 overlay**。

```
my-app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    └── prod/
        ├── kustomization.yaml
        ├── replica-patch.yaml
        └── resource-patch.yaml
```

`base/kustomization.yaml`：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
commonLabels:
  app.kubernetes.io/name: my-app
```

`base/deployment.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: my-app }
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: app
          image: ghcr.io/example/my-app:latest
          resources:
            requests: { cpu: 100m, memory: 256Mi }
```

`overlays/prod/kustomization.yaml`：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod
resources:
  - ../../base
images:
  - name: ghcr.io/example/my-app
    newTag: v1.2.3   # ★ 不改 base，覆盖镜像 tag
patches:
  - path: replica-patch.yaml
    target: { kind: Deployment, name: my-app }
  - path: resource-patch.yaml
    target: { kind: Deployment, name: my-app }
```

`overlays/prod/replica-patch.yaml`（strategic merge）：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: my-app }
spec:
  replicas: 10
```

构建：

```bash
kubectl kustomize overlays/prod
# 或直接 apply
kubectl apply -k overlays/prod
```

优势：纯 yaml、无模板语法；diff 友好（可以直接看到 base 和 overlay 的具体变化）。

劣势：复杂场景的"条件渲染"做不了（比如"prod 才需要 HPA"）。

实际生产经常 **Helm 包成 chart + Kustomize 做最后一层 overlay**：

```yaml
# kustomization.yaml
helmCharts:
  - name: my-app
    repo: oci://ghcr.io/example
    version: 1.2.3
    releaseName: my-app
    valuesFile: values-prod.yaml
patches:
  - path: pod-disruption-budget.yaml   # Helm chart 没提供的资源，手动加
```

> **口诀**：Helm 强在参数化和复用；Kustomize 强在透明和可读。复杂场景两者并用，让 Helm 出"标准化模块"，Kustomize 出"环境差异 overlay"。

---

## 3. 实测数据：流水线性能基线

### 3.1 单阶段 vs 多阶段镜像大小

测试项目：Spring Boot 3.2 + Java 21，jar 大小 45MB，依赖约 80 个。

| 镜像类型 | 镜像大小 | 推送时间（GHA → GHCR） | 拉取时间（K8s 节点） |
| --- | --- | --- | --- |
| 单阶段（maven:3.9-eclipse-temurin-21） | **882 MB** | 47s | 38s |
| 多阶段（jre-alpine） | 178 MB | 11s | 9s |
| 多阶段（distroless/java21） | 198 MB | 12s | 10s |
| jlink + custom JRE（仅打需要的模块） | **94 MB** | 7s | 5s |

镜像大小直接影响：
- 节点拉镜像速度（首次部署 / 扩容）
- 镜像仓库存储成本
- 跨区域复制带宽

**结论**：能多阶段就多阶段，能 distroless 就 distroless。jlink 适合极致优化场景，复杂度高。

### 3.2 BuildKit cache mount 加速效果

同样的 Spring Boot 项目，连续 build 测试：

| 场景 | 无任何缓存 | 仅 layer 缓存 | layer + cache mount |
| --- | --- | --- | --- |
| 首次 build | 3m 12s | 3m 12s | 3m 12s |
| 改 1 行 Java 代码 | 3m 10s | 2m 50s | **18s** |
| 改 pom.xml 加一个依赖 | 3m 15s | 3m 15s | **42s** |
| 完全干净 build（CI 冷启） | 3m 12s | 3m 12s（cache miss） | **35s**（cache hit） |

`mode=max` 保证所有中间层都进缓存（相比 `mode=min` 只缓存最终 stage）。CI 场景必须开 `mode=max`。

### 3.3 不同 CI 系统的并发能力

测试场景：100 个 PR 同时 push，每个 PR 跑 5 分钟测试。

| CI 系统 | 配置 | 100 个 PR 完成总时长 | 排队最长 |
| --- | --- | --- | --- |
| GitHub Actions（默认） | 20 并发（Free tier） | ~25 min | 20 min |
| GitHub Actions（企业版） | 180 并发 | 8 min | 3 min |
| GitLab CI（self-hosted runner × 10） | 10 并发 | 50 min | 45 min |
| GitLab CI（K8s runner，弹性） | 100 并发 | 6 min | 1 min |
| Jenkins（10 agent） | 10 并发 | 50 min | 45 min |
| Jenkins（K8s 动态 agent） | 100 并发 | 7 min | 1 min |

**结论**：核心瓶颈不是 CI 系统本身，而是**runner 池大小**。Self-hosted 的核心优势是可以接 K8s 动态 agent，按需起 Pod 跑 build，理论无限并发。

### 3.4 滚动 vs 金丝雀的发布时长 + 回滚速度

部署 30 个副本的服务：

| 策略 | 完成发布耗时 | 异常发现到回滚生效 | 资源开销 |
| --- | --- | --- | --- |
| 滚动（maxSurge=25%, maxUnavail=0） | 8 min | 5~8 min（要再滚一次） | +25% Pod |
| 蓝绿（service 切换） | 10 min（先全起完） | **30s**（service selector 改回） | +100% Pod |
| 金丝雀（1% → 10% → 50% → 100%） | 25 min（含观察期） | 1 min（删 canary） | +3% Pod |
| 金丝雀 + Argo Rollouts 自动分析 | 20~40 min | **自动**（基于 SLO 触发） | +5% Pod |

口诀：**蓝绿快但贵，金丝雀慢但稳，滚动均衡但回滚最痛**。

### 3.5 Helm vs Kustomize 的可维护性

主观打分（5 分制），基于 3 年生产经验：

| 维度 | Helm | Kustomize |
| --- | --- | --- |
| 入门门槛 | 3（学 Go template） | 4（纯 yaml） |
| 参数化能力 | 5 | 3 |
| 可读性 / diff 友好 | 2 | 5 |
| 社区 chart 复用 | 5 | 2 |
| 复杂条件渲染 | 4 | 1 |
| 跨环境一致性 | 3 | 5 |
| 调试模板错误 | 1（嵌套深） | 4（错误清晰） |

**结论**：自研业务用 Kustomize；引入第三方组件用 Helm；两者结合是最常见的成熟模式。

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① 流水线慢 —— "单测跑 30 分钟"

#### 现象
团队抱怨：从 push 到 PR 上能 review 要等 30 分钟。监控发现 GHA 的 test job 平均 28 分钟。

#### 根因

进入 job 看具体耗时：

```
mvn test            27m 20s
checkout            18s
setup-java          8s
upload artifact     12s
```

查 surefire-reports 发现：3000+ 个单测，**串行**跑。其中 1500 个是依赖 Spring Context 的"集成测试"（每个都启动一次 Spring，每次 ~1.2s）。

更糟的是 `maven-surefire-plugin` 默认 `forkCount=1`，单进程单 JVM 跑。

#### 修复

**第一刀：分层测试**。
- 真正的单测（不启 Spring）：`*Test.java`，毫秒级
- 集成测试：`*IT.java`，秒级
- 端到端：`*E2E.java`，分钟级

`pom.xml`：

```xml
<plugin>
  <artifactId>maven-surefire-plugin</artifactId>
  <configuration>
    <includes>
      <include>**/*Test.java</include>
    </includes>
    <forkCount>1C</forkCount>          <!-- 每核 1 个 fork -->
    <reuseForks>true</reuseForks>
    <parallel>classesAndMethods</parallel>
    <threadCount>4</threadCount>
  </configuration>
</plugin>

<plugin>
  <artifactId>maven-failsafe-plugin</artifactId>
  <configuration>
    <includes>
      <include>**/*IT.java</include>
    </includes>
  </configuration>
</plugin>
```

**第二刀：Spring 上下文缓存**。Spring Test 默认会缓存 `@SpringBootTest` 的 context，但**每个不同的配置组合都会重建一次**。统一基类：

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@ActiveProfiles("test")
@AutoConfigureMockMvc
public abstract class BaseIT {
    // 所有 IT 继承这个基类，共享同一个 context
}
```

**第三刀：CI 层并行**。把 test 拆成多个 job，按测试包分片：

```yaml
test:
  strategy:
    matrix:
      shard: [1, 2, 3, 4]
  steps:
    - run: mvn test -Dtest='**/shard${{ matrix.shard }}/*Test'
```

**第四刀：依赖缓存** —— actions/setup-java 的 `cache: 'maven'` 必开。

#### 验证

测试矩阵：

| 阶段 | 优化前 | 优化后 |
| --- | --- | --- |
| Unit test（仅 Test） | 12 min | **45s**（4 fork 并行） |
| Integration（IT） | 16 min | 6 min（context 缓存） |
| 总耗时（4 shard 并行） | 28 min | **7 min** |

> **口诀**：流水线慢三板斧 —— **分层（unit/integration/e2e）+ 进程内并行（forkCount + parallel）+ CI 间并行（matrix shard）**。

---

### 案例 ② Docker 镜像膨胀 —— "500MB+ 的 Node 镜像"

#### 现象

前端项目镜像 580MB，每次发布都要等 5 分钟拉镜像。同事说"前端项目就是大"。

#### 根因

原 Dockerfile：

```dockerfile
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

问题逐条数：
1. `node:20` 基础镜像 1.1GB（含完整 Debian + dev 工具）
2. `npm install` 装了 dev 依赖（webpack、jest、storybook…），`node_modules` ~ 800MB
3. 构建产物（`build/` 或 `dist/`）几百 KB，但**所有源码、test、node_modules 都还在镜像里**
4. `CMD ["npm", "start"]` 实际是 `node` 跑构建产物，根本不需要 npm

#### 修复

```dockerfile
# syntax=docker/dockerfile:1.7

# 阶段 1：装依赖（含 dev）
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# 阶段 2：构建
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# 阶段 3：运行（仅含 node 二进制 + 构建产物 + 生产依赖）
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production

# ★ 重新装一次仅生产依赖
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci --omit=dev

COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public

USER node
EXPOSE 3000
CMD ["node_modules/.bin/next", "start"]
```

更激进的方案 —— Next.js 14 的 `output: 'standalone'`：

```js
// next.config.js
module.exports = { output: 'standalone' }
```

```dockerfile
FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
CMD ["node", "server.js"]
```

standalone 模式只打包 server 真正需要的依赖，最终镜像 ~ 80MB。

#### 验证

```bash
# 镜像分析工具
docker images my-app
dive my-app:latest    # 一层层看哪里大
docker history my-app:latest --no-trunc
```

| 优化阶段 | 镜像大小 |
| --- | --- |
| 原版 | 580 MB |
| 多阶段 + alpine | 220 MB |
| 多阶段 + 仅生产依赖 | 165 MB |
| Next.js standalone | **82 MB** |

> **口诀**：镜像优化三步 —— **多阶段砍中间产物 + 选小基础镜像（alpine/distroless）+ 仅装运行时依赖**。

---

### 案例 ③ 部署失败回滚不干净 —— "用 latest tag 的悲剧"

#### 现象

线上服务报错率突然飙升。值班 `kubectl rollout undo deployment/app -n prod`，状态显示 "rolled back successfully"，但**报错率没下降**。`kubectl describe pod` 看 image 还是新的。

#### 根因

Deployment yaml 是这样的：

```yaml
spec:
  template:
    spec:
      containers:
        - name: app
          image: registry.internal/app:latest   # ❌
          imagePullPolicy: Always
```

发布流程：

```bash
docker build -t registry.internal/app:latest .
docker push registry.internal/app:latest
kubectl rollout restart deployment/app
```

回滚的本质：`kubectl rollout undo` 找到上一个 ReplicaSet 的 spec，apply 回去。但**上一个 ReplicaSet 的 image 也是 `app:latest`**！回滚后 Pod 还是 `latest` tag，`imagePullPolicy: Always` 又拉一遍 —— 拉到的还是新版本。

回滚等于没回滚。

#### 修复

**铁律：永远用不可变 tag**。

```yaml
spec:
  template:
    spec:
      containers:
        - name: app
          image: registry.internal/app:v1.2.3-abc1234   # ★ 含 git SHA
          imagePullPolicy: IfNotPresent                  # 节点已有就别再拉
```

CI 流程：

```bash
TAG="$(git describe --tags --always)-${GITHUB_SHA::8}"
docker build -t registry.internal/app:$TAG .
docker push registry.internal/app:$TAG

# 用 kustomize / helm 注入 tag
kustomize edit set image app=registry.internal/app:$TAG
git commit -am "deploy $TAG" && git push
```

GitOps 模式下，回滚就是 `git revert`：

```bash
git revert HEAD
git push
# ArgoCD 检测到变更，自动同步到旧版本
```

回滚动作可审计、可追溯、可重现。

镜像仓库侧：开启**不可变 tag 策略**（Harbor / ECR / GCR 都支持），从源头禁止 `latest`、`v1.2.3` 这类 tag 被覆盖。

#### 验证

```bash
# 1. 看 Deployment 的 image 是不是带 SHA
kubectl get deploy/app -o jsonpath='{.spec.template.spec.containers[0].image}'

# 2. 模拟回滚
kubectl rollout undo deployment/app
kubectl rollout status deployment/app
# 检查 image 字段确实变了
```

> **口诀**：**生产镜像永远带 git SHA，从不使用 `latest`**。回滚的语义必须是"指向另一份不可变制品"。

---

### 案例 ④ 灰度卡住 —— "流量不切换"

#### 现象

线上用 Argo Rollouts 做金丝雀。新版本部署后，按计划应该 10% → 30% → 50% → 100%。结果在 10% 卡了两小时不动，但状态显示 "Healthy"。

#### 根因

Rollouts 的 yaml：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      canaryService: app-canary
      stableService: app-stable
      trafficRouting:
        istio:
          virtualService: { name: app-vs }
      steps:
        - setWeight: 10
        - pause: { duration: 5m }
        - analysis:
            templates: [{ templateName: success-rate }]
        - setWeight: 30
        - pause: {}                # ★ 无 duration → 等手动恢复
        - setWeight: 100
```

`pause: {}` 是**无限暂停**，必须手动 `kubectl argo rollouts promote app` 才会继续。值班的同事不知道这个语义，以为 Rollouts 会自动推进。

更隐蔽的根因：金丝雀流量为什么"看起来切了 10%"，但 grafana 上 canary Pod 几乎没流量？

```bash
kubectl get virtualservice app-vs -o yaml
```

```yaml
spec:
  http:
    - route:
        - destination: { host: app-stable }
          weight: 90
        - destination: { host: app-canary }
          weight: 10
```

config 没问题。但 `kubectl describe svc app-canary`：

```
Selector: app=app,rollouts-pod-template-hash=abc123
Endpoints: <none>     ← ★
```

canary service 选不到任何 Pod！原因是 Argo Rollouts 给 canary Pod 打了 hash 标签，但**有人手动改了 Pod 模板的 labels**，导致 selector 失配。

#### 修复

**修复 1**：明确 pause 语义。

```yaml
steps:
  - setWeight: 10
  - pause: { duration: 5m }    # 5 分钟自动推进
  - setWeight: 30
  - pause: { duration: 10m }   # 不要无限 pause，除非真的要人审
```

**修复 2**：禁止手动改 Rollouts 管理的 Pod template，所有变更走 Git。

**修复 3**：加自动化分析（automated analysis），异常自动回滚：

```yaml
- analysis:
    templates: [{ templateName: success-rate }]
    args:
      - name: service-name
        value: app-canary
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata: { name: success-rate }
spec:
  metrics:
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[2m]))
            /
            sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
```

成功率 < 95% 持续 3 次 → 自动回滚到 stable。

#### 验证

```bash
# 模拟坏版本
kubectl argo rollouts set image app app=registry.internal/app:bad-version

# 观察是否自动回滚
kubectl argo rollouts get rollout app --watch
```

预期看到：promote 到 10% → 5xx 上升 → analysis 失败 3 次 → 自动 abort → 流量切回 stable。

> **口诀**：**金丝雀必须配自动分析**，否则等于人肉值班。`pause` 无 duration = 无限挂起，是常见反模式。

---

### 案例 ⑤ 配置漂移 —— "人手改了集群配置"

#### 现象

值班同事为了应急调高了某个 Deployment 的 replicas 从 5 → 20。一天后排查"诡异的 OOM"，发现 replicas 又变回 5 了，且时间点正好是某次 ArgoCD sync。

#### 根因

Application 配了 `automated.selfHeal: true`：

```yaml
syncPolicy:
  automated:
    selfHeal: true
```

selfHeal 的语义：集群实际状态与 Git 不一致时，**3 分钟内自动改回 Git 状态**。值班的临时改动 3 分钟后就被 ArgoCD 重置了。

更糟：值班同事之后没看到自己的改动了，以为是别人改回去的，又改了一遍 → 又被改回去 → 来回博弈到天明。

#### 修复

**正确的应急流程**：临时变更也必须走 Git。

```bash
# 1. 改 Git
cd k8s-manifests
sed -i 's/replicas: 5/replicas: 20/' overlays/prod/replica-patch.yaml
git commit -am "URGENT: scale up app to 20 replicas"
git push

# 2. ArgoCD 自动 sync（或手动触发 sync 加速）
argocd app sync app-prod
```

如果 Git 流程实在等不及（半夜罕见情况），用 ArgoCD 的临时关闭机制：

```bash
# 暂停 self-heal（注意：必须事后恢复）
argocd app set app-prod --sync-policy none

# 紧急修改
kubectl scale deployment/app --replicas=20

# 修复后改 Git，然后恢复 auto-sync
git push
argocd app set app-prod --sync-policy automated --self-heal
```

更好的设计：**业务级别的弹性应该走 HPA（HorizontalPodAutoscaler）**，不应该靠人手改 replicas。HPA 改的是 Deployment 的 status，不会被 ArgoCD 当作 spec 漂移。

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: app }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: app }
  minReplicas: 5
  maxReplicas: 50
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

注意：Deployment 的 yaml 中**不要写 replicas**（让 HPA 接管）：

```yaml
spec:
  # replicas: 5    # ★ 删掉，否则 ArgoCD 与 HPA 互相覆盖
```

#### 验证

模拟 selfHeal 行为：

```bash
kubectl scale deployment/app --replicas=999   # 故意改坏
sleep 300
kubectl get deploy/app   # 应该被 ArgoCD 改回 Git 状态
```

> **口诀**：**GitOps 之后所有变更都必须走 Git**。HPA 之类的 status 变更不算漂移；spec 变更必须走 PR。

---

### 案例 ⑥ Secret 泄漏 —— "数据库密码进了代码仓库"

#### 现象

GitHub 自动告警："发现 AWS access key 泄漏在 commit abc1234"。同时收到 AWS 账单异常告警 —— 有人用泄漏的 key 起了几十台 GPU 实例挖矿。

#### 根因

某个新人提交代码时，把本地调试用的 `application-dev.yml` 误加进 git：

```yaml
spring:
  datasource:
    password: P@ssw0rd123!
aws:
  access-key: AKIAIOSFODNN7EXAMPLE
  secret-key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

更糟的是：即使后续发现了把这个文件 `git rm` 删掉，**Git 历史里它还在**。`git log -p application-dev.yml` 一查就有。攻击者扫描 GitHub 公开仓库的速度通常在几分钟内。

#### 修复

**事后**：
1. 立即 revoke 泄漏的所有凭证
2. 不要 `git revert`（不删历史），用 `git filter-repo` 或 BFG 重写历史：
   ```bash
   bfg --delete-files application-dev.yml
   git reflog expire --expire=now --all && git gc --prune=now --aggressive
   git push --force
   ```
3. 通知所有协作者重新 clone
4. 审计被泄漏的资源使用情况

**事前预防**：

第一道防线 — `.gitignore`：

```gitignore
**/application-*.yml
!**/application.yml
!**/application-ci.yml
*.pem
*.key
.env
.env.*
!.env.example
```

第二道防线 — pre-commit hook + secret scanner：

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks
```

`pre-commit install` 后每次 commit 前自动扫描。

第三道防线 — CI 阶段扫描：

```yaml
- uses: gitleaks/gitleaks-action@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

第四道防线 — **生产 Secret 永远不进 Git**。三种主流方案：

**方案 A：External Secrets Operator**（推荐）

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: { name: db-credentials }
spec:
  refreshInterval: 1h
  secretStoreRef: { name: vault-backend, kind: ClusterSecretStore }
  target: { name: db-secret }
  data:
    - secretKey: password
      remoteRef: { key: prod/db, property: password }
```

ESO 从 Vault / AWS Secrets Manager / GCP Secret Manager 拉密文，在集群内创建 K8s Secret。**Git 里只有引用，没有明文**。

**方案 B：SealedSecret**

```bash
echo -n "P@ssw0rd123!" | kubeseal --raw --from-file=/dev/stdin \
  --namespace=prod --name=db-secret > sealed-password.txt
```

SealedSecret 用集群内 controller 的公钥加密，Git 里存密文。controller 解密后创建 Secret。

**方案 C：SOPS + age/KMS**

适合 GitOps 友好的小团队。

#### 验证

```bash
# 1. 扫描历史
gitleaks detect --source . --redact -v

# 2. 测试 pre-commit
echo "AKIAIOSFODNN7EXAMPLE" >> test.txt
git add test.txt
git commit -m "test"   # 应该被 hook 拦下
```

> **口诀**：Secret 防泄漏四件套 —— **gitignore + pre-commit 扫描 + CI 扫描 + Secret 全走 Vault/ESO**。出事后必须重写历史 + 全量轮换。

---

### 案例 ⑦ Jenkins 主节点磁盘满 —— "全队阻塞"

#### 现象

早上九点全公司 push 代码，所有 Jenkins job pending，UI 打不开。运维登 Jenkins master 看 `df -h`：

```
/var/jenkins_home   500G  500G   0G  100%
```

#### 根因

Jenkins master 的 `/var/jenkins_home` 装了：
- 所有 build 的完整日志（默认无限保留）
- 所有 build 的 artifact（默认无限保留）
- 工作空间（workspace）—— 即使 build agent 在远程，某些场景 master 也会保留
- 插件、配置、credentials

Jenkins 没有自动清理策略。一年下来，几百个 job × 几千次 build × 几 MB 日志/artifact = 几百 GB。

#### 修复

**临时**：扩盘 + 立即清理。

```bash
# 1. 找出大头
du -sh /var/jenkins_home/jobs/* | sort -rh | head -20

# 2. 通过 Jenkins UI 或脚本清理老 build
# Manage Jenkins → Script Console:
Jenkins.instance.getAllItems(Job.class).each { job ->
    job.getBuilds().each { build ->
        if (build.getTimestamp().before(new GregorianCalendar(2024, 0, 1))) {
            build.delete()
        }
    }
}
```

**长效**：每个 Pipeline 必须配 buildDiscarder：

```groovy
options {
  buildDiscarder(logRotator(
    numToKeepStr: '50',          // 仅保留最近 50 次
    daysToKeepStr: '30',         // 30 天内的
    artifactNumToKeepStr: '10',  // artifact 仅保留 10 次
    artifactDaysToKeepStr: '7'
  ))
}
```

或全局默认（Manage Jenkins → Configure System）。

**架构改进**：Jenkins master 永远不该跑 build。

```groovy
agent { label 'docker' }    // 强制走 agent
```

更进一步：用 K8s Plugin 做动态 agent，每次 build 起一个 Pod，build 完销毁。master 完全不存 workspace。

```yaml
# Jenkins Kubernetes plugin Pod template
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: jnlp
      image: jenkins/inbound-agent:latest
      resources:
        requests: { cpu: 500m, memory: 1Gi }
    - name: docker
      image: docker:25
      command: [cat]
      tty: true
```

监控 + 告警：

```bash
# Prometheus alert
- alert: JenkinsDiskHigh
  expr: node_filesystem_avail_bytes{mountpoint="/var/jenkins_home"} / node_filesystem_size_bytes < 0.2
  for: 10m
```

#### 验证

```bash
# 配 buildDiscarder 后跑几次，确认旧 build 被清
ls /var/jenkins_home/jobs/my-job/builds/ | wc -l   # 应该 ≤ numToKeepStr
```

> **口诀**：Jenkins master = 国王，不能干活。**buildDiscarder + 动态 agent + 磁盘告警** 三件套必配。

---

### 案例 ⑧ ArgoCD pull 模型与紧急修复的冲突

#### 现象

线上服务挂了，需要立即把镜像版本回到 `v1.2.2`。但流程是：

1. 改 Git → 提 PR → 等 reviewer 批 → 合入 main → ArgoCD sync
2. 全流程 ~ 20 分钟

业务损失每分钟几万。值班 `kubectl set image` 直接改了集群，3 分钟恢复。但 5 分钟后 ArgoCD 又改回了"坏版本"。

#### 根因

GitOps 的 selfHeal 与"紧急修复"天然冲突。**真相源是 Git 不是集群**，绕过 Git 的修改一定会被打回原形。

但 PR 流程在线上事故时太慢。

#### 修复

**方案 A：紧急 PR 直通车**。

为 production 仓库设置特殊路径，紧急修复走简化流程：

```yaml
# .github/CODEOWNERS
/overlays/prod/emergency/   @oncall-team
/overlays/prod/             @release-team
```

emergency 路径下的 PR 1 个 oncall-team 成员 approve 即可合入（vs 普通 prod 需要 2 个 release-team）。

实操：

```bash
# 一键脚本
./scripts/emergency-rollback.sh v1.2.2
# 内部做：改 yaml + 提 PR + 自动 approve（需 GH token）+ merge + 触发 ArgoCD 立即 sync
```

**方案 B：ArgoCD 临时关闭 selfHeal**。

```bash
# 1. 关 self-heal
argocd app set app-prod --sync-policy none

# 2. 集群直改
kubectl set image deployment/app app=registry.internal/app:v1.2.2

# 3. 事后必须补 Git PR + 恢复 auto-sync
git commit -am "rollback to v1.2.2 (incident #123)"
git push
argocd app set app-prod --sync-policy automated --self-heal
argocd app sync app-prod
```

**方案 C：用 Argo Rollouts 的 abort + undo**。

```bash
# Rollouts 自身支持快速回滚
kubectl argo rollouts abort app -n prod
kubectl argo rollouts undo app -n prod
```

这是 Rollouts CR 内的状态变更，不算 Git 漂移，不会被 selfHeal 干扰。

**架构层面**：把"镜像 tag"独立成单独的 yaml，设 selfHeal=false 但其他 selfHeal=true：

```yaml
# 镜像 tag 单独成 ApplicationSet，不开 selfHeal
# 应用配置 + 资源等开 selfHeal
```

#### 验证

模拟事故 → 执行紧急回滚脚本 → 全流程必须 < 5 分钟。

> **口诀**：GitOps 与紧急修复的矛盾，靠**流程加速（紧急 PR 直通车）+ 工具加速（Argo Rollouts undo）**化解。绝不要养成"绕过 Git 改集群"的习惯。

---

### 案例 ⑨ Pipeline 太长 —— "日志超 100MB 没法排查"

#### 现象

一个综合性流水线：lint + 单测 + 集成 + 安全扫描 + build + push + 部署 + 烟测，总时长 45 分钟，日志 120MB。某个集成测试 flaky 失败，要从 120MB 日志里翻出失败点。

#### 根因

日志膨胀的几个常见原因：
- Maven `-X` debug 输出全开
- 集成测试输出大量 SQL（`hibernate.show_sql=true`）
- Docker build 每个层的所有命令输出
- 未关闭的进度条（`--progress=plain` 不停刷新）
- 业务日志开了 DEBUG 级别

监控盲区：
- 单一巨大 job，失败时没人知道是哪一步失败
- 日志按时间顺序混杂，多线程输出乱序
- 日志没有结构化，grep 困难

#### 修复

**1. 拆 stage 让失败定位更快**：

```yaml
jobs:
  lint:        { needs: [], timeout-minutes: 5 }
  unit-test:   { needs: [], timeout-minutes: 10 }
  integration: { needs: [unit-test], timeout-minutes: 15 }
  security:    { needs: [], timeout-minutes: 10 }
  build:       { needs: [unit-test, integration, security], timeout-minutes: 10 }
  deploy:      { needs: [build], timeout-minutes: 5 }
```

CI 系统的 UI 会展示每个 job 的状态 —— 失败时一眼看出是哪一步。

**2. 控制日志级别**：

```bash
mvn -B -ntp test                      # -ntp 关闭 transfer progress
docker buildx build --progress=plain  # 替代默认 tty
```

`logback-test.xml`：

```xml
<configuration>
  <root level="WARN"/>
  <logger name="com.mycompany" level="INFO"/>
  <logger name="org.springframework" level="WARN"/>
  <logger name="org.hibernate.SQL" level="WARN"/>
</configuration>
```

**3. 测试报告结构化输出**：

```yaml
- run: mvn -B test
- uses: dorny/test-reporter@v1
  if: always()
  with:
    name: Maven Tests
    path: '**/surefire-reports/*.xml'
    reporter: java-junit
```

PR 页面直接看到失败的 case，不用翻日志。

**4. flaky 测试自动重试 + 隔离**：

```xml
<plugin>
  <artifactId>maven-surefire-plugin</artifactId>
  <configuration>
    <rerunFailingTestsCount>2</rerunFailingTestsCount>
  </configuration>
</plugin>
```

且 flaky 必须有 issue 跟踪，定期清理。Google 的内部数据是大型项目 ~ 1.5% 的测试是 flaky，不治理会失控。

**5. 日志归档到外部**：

```yaml
- name: Upload logs on failure
  if: failure()
  uses: actions/upload-artifact@v4
  with:
    name: logs-${{ github.run_id }}
    path: |
      target/surefire-reports/
      target/failsafe-reports/
      logs/
    retention-days: 7
```

failed run 的日志自动归档，2 周内可下载排查。

#### 验证

```
日志大小：120 MB → 18 MB
排查失败用例耗时：30 min → 2 min（PR 页面直接看 test reporter）
flaky 率：5% → 0.3%
```

> **口诀**：流水线长不是问题，**不可观测才是问题**。拆 stage + 结构化测试报告 + 失败日志归档 + flaky 治理，四件套打通。

---

## 5. 上线检查清单

### 流水线层
- [ ] 每个 PR 的 CI < 10 分钟（超过就拆 / 并行 / 缓存）
- [ ] 同一分支的并发 build 互相取消（`concurrency` / `disableConcurrentBuilds`）
- [ ] 流水线分阶段（lint / unit / integration / build / deploy），失败点可秒级定位
- [ ] 测试报告结构化（JUnit XML / dorny test-reporter），不靠翻日志
- [ ] flaky 测试有跟踪 issue，rerun 不超过 2 次

### 镜像层
- [ ] Dockerfile 必须多阶段构建
- [ ] 基础镜像选 alpine / distroless / slim 系列
- [ ] BuildKit cache mount 用于包管理器（`/root/.m2`、`/root/.npm`）
- [ ] CI cache 配 `mode=max`
- [ ] 镜像 tag 含 git SHA，永远不用 `latest` 部署
- [ ] 镜像仓库开启不可变 tag 策略
- [ ] 容器内非 root 运行（`USER` 指令）
- [ ] 镜像扫描（trivy / grype）作为 CI 必过项

### 部署层
- [ ] 部署策略明确（rolling / blue-green / canary）+ 配套回滚预案
- [ ] 金丝雀必须配自动分析（基于 SLO 的 success rate / latency）
- [ ] 健康检查（liveness / readiness）配置正确，readiness 失败时不接流量
- [ ] `terminationGracePeriodSeconds` + `preStop` 实现优雅停机
- [ ] PodDisruptionBudget 防止全部 Pod 同时挂
- [ ] HPA 配置后 Deployment yaml 删掉 `replicas` 字段

### 安全层
- [ ] gitleaks / trufflehog pre-commit + CI 扫描
- [ ] Secret 全部走 Vault / ESO / SealedSecret，不进 Git
- [ ] CI 凭证用 OIDC 短期 token，禁用长期 PAT
- [ ] SBOM 生成（cyclonedx / syft）+ 漏洞扫描入 CI
- [ ] 镜像签名（cosign keyless）+ 准入控制（cosign / kyverno）
- [ ] DependencyCheck / Snyk / Dependabot 监控依赖漏洞

### 回滚层
- [ ] 回滚是 git revert 或 helm rollback，秒级可执行
- [ ] 数据库变更与代码部署解耦，先发布兼容性 schema
- [ ] 紧急回滚预案文档化，oncall 必须演练过
- [ ] 蓝绿 / 金丝雀场景下，stable 版本至少保留一周再清理

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Progressive Delivery
- **Argo Rollouts**：基于 metric 自动金丝雀 / 蓝绿，与 Istio / Nginx / ALB 集成
- **Flagger**：CNCF 项目，类似定位，与 Flux 配合
- 自动分析的关键 —— 选对 SLI（success rate、P99、错误率），并设合理阈值

### Feature Flag
- **LaunchDarkly**（商业）/ **Unleash**（开源）/ **Flagsmith**：发布与部署解耦
- A/B 测试、灰度按用户群、kill switch
- Flag 治理：旧 flag 必须有过期机制，否则代码里满是 if

### DORA 指标
- **Deployment Frequency**：部署频率
- **Lead Time for Changes**：从 commit 到 prod 的时间
- **MTTR**：故障平均恢复时间
- **Change Failure Rate**：导致回滚 / hotfix 的部署占比
- 这四个指标是衡量 DevOps 成熟度的工业标准（Google DORA 报告）

### SLSA 供应链安全
- SLSA Level 1~4：从"有 build 记录"到"完整可重现 + 隔离 build + 双人审"
- in-toto 框架：定义 build 链路的密码学证明
- 案例：SolarWinds 事件后供应链安全成为 CI/CD 必修课

### Tekton（Cloud Native CI/CD）
- 完全跑在 K8s 上的 CI 系统，每个 step 是一个容器
- 与 ArgoCD 配合做 Cloud Native 端到端流水线
- 复杂度高于 GHA，但可移植性、可扩展性更强

### Bazel / Buck2 增量构建
- Google / Meta 内部 monorepo 的核心：基于依赖图的精准增量
- 改一行 Java 代码，只编译受影响的包
- 适合 monorepo 大项目（10 万 + 文件级别）

### Backstage 与 Internal Developer Platform
- Spotify 开源的开发者门户
- 整合 CI/CD / 文档 / 服务目录 / 监控
- "平台工程（Platform Engineering）"思潮的代表工具

---

## 7. 引用

- 《Continuous Delivery》—— Jez Humble & David Farley（CD 概念奠基书）
- 《Accelerate》—— Forsgren / Humble / Kim（DORA 指标方法论）
- [The Twelve-Factor App](https://12factor.net/)
- [GitHub Actions docs](https://docs.github.com/en/actions)
- [GitLab CI/CD reference](https://docs.gitlab.com/ee/ci/)
- [Jenkins Pipeline syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Docker BuildKit](https://docs.docker.com/build/buildkit/)
- [ArgoCD documentation](https://argo-cd.readthedocs.io/)
- [Argo Rollouts](https://argoproj.github.io/argo-rollouts/)
- [Helm best practices](https://helm.sh/docs/chart_best_practices/)
- [Kustomize](https://kustomize.io/)
- [SLSA framework](https://slsa.dev/)
- [Google DORA Reports](https://dora.dev/)
- [CNCF Landscape - CI/CD](https://landscape.cncf.io/)

---

> 配套：[Kubernetes 实战](/article/k8s/k8s-essentials) | [Git 工作流](/article/devops/git-workflow) | [单元测试与 Mock](/article/testing/unit-testing)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。
