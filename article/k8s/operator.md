# Kubernetes Operator 与 CRD 开发 —— 从 Reconcile Loop 到生产级 Operator

> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你只想刷面试或快速上手，看 [Kubernetes 实战](/article/k8s/k8s-essentials) 那一节就够了。
> 这里要回答的，是**写 Operator 上线遇到坑、半夜被 CR 卡死告警叫起来**时你需要知道的那部分。

---

## 0. 读这篇之前

先自检 8 个问题，能秒答 6 个以上，可以跳过本文：

1. Operator 和 Controller 是同一个东西吗？kube-controller-manager 里那一堆是 Operator 吗？
2. 你写的 CRD 和 Aggregated API Server 都是"扩展 API"，**为什么 99% 的场景都选 CRD**？两者的边界在哪？
3. `Reconcile()` 函数被同一个 CR 触发了 5 次，业务表里多了 5 条记录 —— 错在哪？幂等的最小约束是什么？
4. 你给 CR 加了 Finalizer，结果 `kubectl delete` 一直卡着，`deletionTimestamp` 已经有了但资源就是不消失 —— 排查的第一步看哪儿？
5. 你在 Reconcile 里 `r.Update(ctx, cr)` 同时改了 `cr.Spec.Replicas` 和 `cr.Status.Phase`，发现 Status 没生效 —— 为什么？
6. CR 删掉时，它创建的 Pod / ConfigMap / Service 没被自动删掉 —— OwnerReference 哪个字段错了？
7. ValidatingWebhook 配错了 `failurePolicy: Fail`，导致整个集群 API 都 503 —— 怎么救？为什么这是 Operator 上线最常翻车的一步？
8. Operator 部署了 3 副本想做高可用，结果 3 个副本同时调用 Reconcile，业务表被并发写炸 —— Leader Election 的最小配置是什么？

如果有半数答不上，往下看。

---

## 1. 底层模型：Operator、CRD、Reconcile Loop

绝大多数人对 Operator 的认知止于"用 Kubebuilder 生成代码、填 Reconcile 函数"。这层抽象解释不了上面 8 个问题。要看懂它们，必须下沉到 Operator 模式与 Kubernetes API 机制的耦合点。

### 1.1 Operator 模式的本质：声明式 + Controller

Operator 不是一个新东西，它是 **CRD（Custom Resource Definition）+ Controller** 的组合，把"运维知识"编码成代码。

```
传统方式：人 → 写脚本 → 调命令 → 改状态 → 出问题就跑去看
Operator： 用户 → 改 CR.spec → Operator 持续 Reconcile → 让 actual 收敛到 desired
```

CoreOS 在 2016 年首次提出这个名字，但模型本身在 kube-controller-manager 里早就存在 —— Deployment、StatefulSet、Job 这些都是"内置 Operator"。

> **口诀**：Operator = CRD + Controller + 领域知识。"Controller" 是机制，"Operator" 是模式。

### 1.2 CRD：Custom Resource Definition

CRD 让你在不改 kube-apiserver 代码的前提下，**注册一个新资源类型**。注册后，apiserver 自动给你提供：

- REST API：`/apis/<group>/<version>/namespaces/<ns>/<plural>/<name>`
- 持久化（etcd 存）
- Watch 机制（Informer 能订阅）
- kubectl 交互（`kubectl get / describe / edit`）
- RBAC、AuditLog、字段校验（OpenAPI v3 schema）

一个最小 CRD：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: redisclusters.cache.example.com
spec:
  group: cache.example.com
  names:
    plural: redisclusters
    singular: rediscluster
    kind: RedisCluster
    shortNames: [rc]
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              replicas: { type: integer, minimum: 1 }
              version:  { type: string }
          status:
            type: object
            properties:
              phase:    { type: string }
              readyReplicas: { type: integer }
    subresources:
      status: {}     # 关键：把 status 拆成独立子资源
      scale:         # 让 HPA 能按 spec.replicas 缩容
        specReplicasPath: .spec.replicas
        statusReplicasPath: .status.readyReplicas
```

注意 `subresources.status: {}`。开了之后，对 `/redisclusters/foo` 的 PATCH 不会改 status；想改 status 必须 PATCH `/redisclusters/foo/status`。这是后面案例 ③ "Status 与 Spec 混用导致数据丢失"的根。

### 1.3 Reconcile Loop：从 desired state 到 actual state

Reconcile 是 Operator 的"主循环"。它的语义不是"事件回调"，而是"**告诉我对 X 这个资源，请把当前状态调整到与 spec 一致**"。

```
                ┌─────────────────────────────────┐
                │  watch event (Add/Update/Del)   │
                └────────────────┬────────────────┘
                                 ▼
                ┌─────────────────────────────────┐
                │  workqueue.Add(key)             │
                │  key = "namespace/name"         │
                └────────────────┬────────────────┘
                                 ▼
                ┌─────────────────────────────────┐
                │  worker.processNextItem()       │
                │   → Reconcile(ctx, req)         │
                └────────────────┬────────────────┘
                                 ▼
                ┌─────────────────────────────────┐
                │  err == nil   → forget(key)     │
                │  err != nil   → requeueRateLimit│
                │  Result.Requeue → requeueAfter  │
                └─────────────────────────────────┘
```

**两条铁律**：

- **Reconcile 必须幂等**：同一个 CR 被调用 N 次，最终状态必须一致。Watch 机制做不到 exactly-once，重试随时发生。
- **Reconcile 不接收 event payload**：函数签名只有 `req types.NamespacedName`，**它不知道这次是 Add 还是 Update**，必须自己 Get 当前状态后比对。

这两条是后面案例 ① "Reconcile 不幂等导致死循环" 的根。

### 1.4 Finalizer 与级联删除

Kubernetes 的删除是 **两阶段**：

```
用户调 DELETE
      │
      ▼
apiserver 设置 metadata.deletionTimestamp = now()
      │ （此时资源还在！不会真正删除）
      ▼
所有 finalizer 被移除后
      │
      ▼
apiserver 真正从 etcd 删除
```

`metadata.finalizers: ["cache.example.com/cleanup"]` 这种字符串列表本质是**死亡前钩子**。Operator 在 Reconcile 里检测到 `deletionTimestamp != nil` 后，做完清理（释放外部资源、调用云 API、删依赖等），主动把对应 finalizer 字符串从列表里移除。列表空了，apiserver 才删行。

这套机制解决的核心问题：**Operator 必须在 CR 真正消失前完成清理**。否则 CR 一删，你就再也看不到它的 spec，外部资源（云盘、外部 DB user 等）泄漏。

> **口诀**：finalizer = 死亡前的最后一次 Reconcile 机会。卡住不删 → 100% 是你 finalizer 移除逻辑挂了。

### 1.5 Status vs Spec 的精确分工

| 字段 | 谁写 | 谁读 | 持久化 |
| --- | --- | --- | --- |
| `spec` | **用户**（kubectl / GitOps） | Operator | etcd |
| `status` | **Operator** | 用户、HPA、其他 Controller | etcd |
| `metadata.labels/annotations` | 都可以 | 都可以 | etcd |

**绝对不要**：

- 用户改 status（HPA / GitOps 可能覆盖你的 status）
- Operator 改 spec（用户期望的状态被改了，违反声明式语义）
- spec 和 status 混在同一次 Update 里发出（status subresource 开了之后这是隐藏陷阱，案例 ③）

### 1.6 Webhook：Mutating / Validating Admission

Webhook 是 apiserver 在持久化 CR 之前的"拦路点"，分两种：

```
client → apiserver
            │
            ▼
       MutatingWebhook（可以改 obj，比如填默认值）
            │
            ▼
       ValidatingWebhook（只读校验，不能改）
            │
            ▼
       persist to etcd
```

CRD 也可以用 OpenAPI schema 做基础校验（type / minimum / pattern），但**跨字段校验、跨资源校验**只能用 webhook。

webhook 是 HTTPS 服务，apiserver 通过 `MutatingWebhookConfiguration` / `ValidatingWebhookConfiguration` 找到它。

陷阱：webhook 服务不可用时，`failurePolicy` 决定 apiserver 行为：
- `Ignore`：放过（可能写入非法数据）
- `Fail`：直接拒绝（可能让整个集群不可写，案例 ④）

### 1.7 Aggregated API Server vs CRD

两条扩展 API 的路径：

| 维度 | CRD | Aggregated API Server |
| --- | --- | --- |
| 实现 | 写 yaml + 写 controller | 自己写一个完整的 apiserver 后端 |
| 存储 | etcd（apiserver 帮你存） | 自己存（可以是非 etcd） |
| 子资源 | 只能 `status` 和 `scale` | 任意自定义 |
| Watch | 自动支持 | 自己实现 |
| 复杂度 | 低 | 高，要自建 etcd 接入或自己存 |
| 典型例子 | 90% 的 Operator | metrics-server、knative serving |

绝大多数业务 Operator 用 CRD 就够了。需要 Aggregated 的几乎只有：自定义存储后端、需要复杂子资源、API 调用频率极高（CRD 走 apiserver，QPS 受限）。

> **口诀**：能用 CRD 就别上 Aggregated，复杂度差一个数量级。

---

## 2. 源码切片（基于 Kubebuilder / controller-runtime）

> 基于 Kubebuilder v3 + controller-runtime v0.16 源码。注释为本文添加。

### 2.1 CRD 的 Go 类型定义

Kubebuilder 用 `// +kubebuilder:` 注释驱动代码生成。一个典型 CR 类型：

```go
// api/v1/rediscluster_types.go

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:shortName=rc
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
type RedisCluster struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   RedisClusterSpec   `json:"spec,omitempty"`
    Status RedisClusterStatus `json:"status,omitempty"`
}

type RedisClusterSpec struct {
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=99
    Replicas int32  `json:"replicas"`
    Version  string `json:"version"`
}

type RedisClusterStatus struct {
    Phase          string             `json:"phase,omitempty"`
    ReadyReplicas  int32              `json:"readyReplicas,omitempty"`
    Conditions     []metav1.Condition `json:"conditions,omitempty"`
}
```

**逐段点透**：

- `// +kubebuilder:subresource:status`：必须加。否则你 `Update` CR 时 status 改不进去。这条注释直接对应 CRD yaml 的 `subresources.status: {}`。
- `metav1.Condition`：K8s 的标准状态字段。**强烈推荐**用它而不是自己定义 phase 字符串，因为 `kubectl wait --for=condition=Ready` 这类工具就是看这个。
- 所有字段必须带 `json` tag，并且首字母大写（公开） —— Go reflection 才能序列化。
- 字段加 `omitempty` 是为了让 `kubectl get -o yaml` 输出干净，但要注意：**zero value 的字段会消失**（int 0、bool false、空切片）。如果业务上"显式 0"和"未设置"语义不同，要用指针类型。

### 2.2 Reconciler.Reconcile 的核心结构

controller-runtime 调度的入口：

```go
// internal/controller/rediscluster_controller.go
func (r *RedisClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    // ① Get 当前 CR（注意：可能已被删，返回 NotFound 不是错误）
    var cr cachev1.RedisCluster
    if err := r.Get(ctx, req.NamespacedName, &cr); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // ② 处理 deletion + finalizer
    if !cr.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, &cr)
    }
    if !controllerutil.ContainsFinalizer(&cr, finalizerName) {
        controllerutil.AddFinalizer(&cr, finalizerName)
        return ctrl.Result{}, r.Update(ctx, &cr)
    }

    // ③ Reconcile 创建 / 更新 子资源
    if err := r.reconcileStatefulSet(ctx, &cr); err != nil {
        return ctrl.Result{}, err
    }
    if err := r.reconcileService(ctx, &cr); err != nil {
        return ctrl.Result{}, err
    }

    // ④ 更新 status
    return r.updateStatus(ctx, &cr)
}
```

**逐段点透**：

- **`client.IgnoreNotFound(err)`**：CR 已被删除时 Get 会返回 404。这不是错误 —— 直接返回 nil 让 workqueue 把这个 key 忘掉。如果你 `return err`，会进入指数退避重试，永远在重试一个不存在的资源。
- **`DeletionTimestamp.IsZero()` 必须先判断**：在加 finalizer 之前就要判断删除，否则用户 `kubectl delete` 后你又给它加 finalizer，资源永远删不掉。
- **加 finalizer 后立即 return**：`r.Update()` 修改了 metadata，会触发 ResourceVersion 变化。如果继续往下走，后面所有写操作都会因为 ResourceVersion 不匹配而冲突。直接 return，等下一次 Reconcile（很快就到，因为 Update 又触发了 watch）。
- **Spec 写和 Status 写必须分开**：见 2.7 节。

### 2.3 client-go 的 Informer / Lister

Reconciler 里的 `r.Get(ctx, key, obj)` 看起来像调 apiserver，**实际上读的是本地 Informer 缓存**。

```go
// k8s.io/client-go/tools/cache/shared_informer.go 简化
type SharedIndexInformer interface {
    AddEventHandler(handler ResourceEventHandler)
    GetStore() Store     // 本地缓存
    GetIndexer() Indexer // 带索引的缓存
    Run(stopCh <-chan struct{})
}

// 启动时干这两件事
func (s *sharedIndexInformer) Run(stopCh <-chan struct{}) {
    // ① 全量 List 拉一次，灌进本地 cache
    // ② Watch 增量更新 cache（HTTP/2 长连接）
    // ③ cache 变化 → 触发 EventHandler → 把 key 塞进 workqueue
}
```

**为什么是这套架构？**

1. **省 apiserver 资源**：所有 controller 都从本地 cache 读，apiserver 只承担 watch 一份长连接。
2. **本地 List 是 O(1)**：cache 内部是 thread-safe 的 map + 索引（支持按 label / namespace 高效过滤）。
3. **代价**：cache **可能滞后** —— Watch 通道延迟、压测时延迟可达秒级。这是案例 ⑥ "Informer 缓存陈旧"的根。

如果你需要"强一致读"（必须读到最新值），调 `r.APIReader.Get(...)` 直连 apiserver，绕过 cache。但代价是 QPS 上限低很多。

### 2.4 Workqueue 的限速重试

controller-runtime 默认用一个 **rate-limited workqueue**：

```go
// k8s.io/client-go/util/workqueue/default_rate_limiters.go
func DefaultControllerRateLimiter() RateLimiter {
    return NewMaxOfRateLimiter(
        // ① 指数退避：5ms, 10ms, 20ms, ..., 上限 1000s
        NewItemExponentialFailureRateLimiter(5*time.Millisecond, 1000*time.Second),
        // ② 全局令牌桶：10 QPS，桶 100
        &BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(10), 100)},
    )
}
```

**逐段点透**：

- `MaxOfRateLimiter` 取**两者最大**：单 key 退避 + 全局 QPS。任何一个限流就生效。
- 单 key 失败 N 次后退避 = `5ms * 2^N`。N=20 时 ≈ 87 分钟，N=27 已经 1000s 上限。
- 含义：**Reconcile 一直返回 error，重试间隔会越来越长**。如果你的失败是"等外部资源就绪"（如等 PVC bound），用 `ctrl.Result{RequeueAfter: 30 * time.Second}` 而不是 `return err`，避免被指数退避惩罚。

```go
// ✅ 业务等待
return ctrl.Result{RequeueAfter: 30 * time.Second}, nil

// ❌ 当成错误，会被退避到很长间隔
return ctrl.Result{}, fmt.Errorf("pvc not ready")
```

### 2.5 Finalizer 的添加与删除

完整的 finalizer 生命周期：

```go
const finalizerName = "rediscluster.cache.example.com/finalizer"

func (r *RedisClusterReconciler) reconcileDelete(ctx context.Context, cr *cachev1.RedisCluster) (ctrl.Result, error) {
    if !controllerutil.ContainsFinalizer(cr, finalizerName) {
        return ctrl.Result{}, nil   // 不归我管
    }

    // ① 执行外部清理（必须幂等！可能被调多次）
    if err := r.cleanupExternalResources(ctx, cr); err != nil {
        return ctrl.Result{}, err   // 失败就重试
    }

    // ② 清理完了再移除 finalizer
    controllerutil.RemoveFinalizer(cr, finalizerName)
    if err := r.Update(ctx, cr); err != nil {
        // 注意：可能撞 ResourceVersion 冲突，让外层重试
        return ctrl.Result{}, err
    }
    // ③ 移除成功后，apiserver 会真正删 CR，下次再触发 Reconcile 就 404
    return ctrl.Result{}, nil
}

func (r *RedisClusterReconciler) cleanupExternalResources(ctx context.Context, cr *cachev1.RedisCluster) error {
    // 调云 API 释放 ELB / 删 OBS / 删外部 DB user
    // 这里必须做：找不到外部资源 = 已经删了，return nil 不是 error
    return nil
}
```

**逐段点透**：

- **必须先做清理再移 finalizer**：反过来就裸奔了。
- **清理函数必须幂等**：finalizer 移除前的每次 Reconcile 都会调它一次。
- **"找不到 = 成功"**：外部资源已经删过了，再删返回 NotFound，不要当错误。
- **只能移除自己的 finalizer**：不要碰别人加的（如 `kubernetes.io/pv-protection`）。

### 2.6 Webhook 的实现

```go
// api/v1/rediscluster_webhook.go

// +kubebuilder:webhook:path=/mutate-cache-example-com-v1-rediscluster,mutating=true,failurePolicy=fail,sideEffects=None,groups=cache.example.com,resources=redisclusters,verbs=create;update,versions=v1,name=mrediscluster.kb.io,admissionReviewVersions=v1
var _ webhook.Defaulter = &RedisCluster{}

func (r *RedisCluster) Default() {
    if r.Spec.Replicas == 0 {
        r.Spec.Replicas = 3       // 填默认值
    }
    if r.Spec.Version == "" {
        r.Spec.Version = "7.2"
    }
}

// +kubebuilder:webhook:path=/validate-cache-example-com-v1-rediscluster,mutating=false,failurePolicy=fail,sideEffects=None,groups=cache.example.com,resources=redisclusters,verbs=create;update,versions=v1,name=vrediscluster.kb.io,admissionReviewVersions=v1
var _ webhook.Validator = &RedisCluster{}

func (r *RedisCluster) ValidateCreate() (admission.Warnings, error) {
    if r.Spec.Replicas%2 == 0 {
        return nil, fmt.Errorf("replicas must be odd for raft quorum")
    }
    return nil, nil
}

func (r *RedisCluster) ValidateUpdate(old runtime.Object) (admission.Warnings, error) {
    oldCR := old.(*RedisCluster)
    if oldCR.Spec.Version != r.Spec.Version && !versionUpgradable(oldCR.Spec.Version, r.Spec.Version) {
        return nil, fmt.Errorf("version downgrade not allowed: %s -> %s", oldCR.Spec.Version, r.Spec.Version)
    }
    return nil, nil
}

func (r *RedisCluster) ValidateDelete() (admission.Warnings, error) {
    return nil, nil
}
```

**逐段点透**：

- `failurePolicy=fail`：webhook 调用失败时拒绝请求。**生产首选 `Ignore` 或缩小 scope**（见案例 ④）。
- `sideEffects=None`：必须声明。如果你的 webhook 真有副作用（写外部系统），声明 `Some` 或 `NoneOnDryRun` —— apiserver 在 dry-run 时会跳过它。
- `Default()` 与 `Validate*()` 是 controller-runtime 的接口，对应 mutating / validating 两类 webhook。
- 跨字段校验放 `Validate*`，单字段格式校验放 CRD 的 OpenAPI schema —— schema 校验在 apiserver 自己做，不依赖 webhook 服务。

### 2.7 Status Subresource 的 Update

这是 Operator 新手最容易踩的坑。看代码：

```go
// ❌ 错误用法
cr.Status.Phase = "Ready"
cr.Spec.Replicas = 5
r.Update(ctx, cr)        // status 部分被丢弃！

// ✅ 正确用法
cr.Spec.Replicas = 5
if err := r.Update(ctx, cr); err != nil { return err }

cr.Status.Phase = "Ready"
if err := r.Status().Update(ctx, cr); err != nil { return err }
```

**为什么？** apiserver 看到 `subresources.status: {}` 的 CRD，把 `/redisclusters/foo` 的 PUT 请求**忽略 status 字段**，反过来 `/redisclusters/foo/status` 的 PUT 请求**忽略 spec 字段**。

`controller-runtime` 的 `r.Status()` 返回的是另一个 client，它的 Update 走 `/status` 子资源路径。**不混用**，否则 status 修改全部丢失。

更安全的写法用 patch（避免 ResourceVersion 冲突）：

```go
patch := client.MergeFrom(cr.DeepCopy())
cr.Status.Phase = "Ready"
cr.Status.ReadyReplicas = 3
return r.Status().Patch(ctx, cr, patch)
```

> **口诀**：spec 用 `r.Update`，status 用 `r.Status().Update` 或 `r.Status().Patch`。**永不混用**。

---

## 3. 实测数据：Reconcile 周期与缓存的真实成本

口诀化的"声明式"几乎所有文章都讲，但**生产场景下 Reconcile 周期、CR 数量、Webhook 延迟的真实代价**很少有人量化。下面给一组对比，结论给在前面。

### 3.1 Reconcile 周期对资源使用的影响

测试设置：单个 Operator 管理 100 个 CR，Reconcile 内部仅做 1 次 Get + 1 次 Status Update。

| `RequeueAfter` | CPU (avg) | 内存 (RSS) | apiserver QPS | 备注 |
| --- | --- | --- | --- | --- |
| 不设置（仅 watch 触发） | 0.5% | 80 MB | 0.5 | 推荐，事件驱动 |
| 30s | 1.2% | 85 MB | 7 | 折中，常用值 |
| 10s | 3% | 90 MB | 22 | 偏高，慎用 |
| 1s | 25% | 120 MB | 220 | **危险**，会拖垮 apiserver |

结论：**默认依赖 watch 触发**，只有"必须周期性检查的"（如证书过期、外部资源轮询）才设 `RequeueAfter`，且至少 30s。

### 3.2 大规模 CR（10000+）的内存占用

Informer 把所有 CR 拉到本地内存。10000 个 CR，每个 spec ~2KB：

| CR 数量 | Informer 内存 | List 全量耗时 | Watch 重连耗时 |
| --- | --- | --- | --- |
| 100 | 5 MB | 10 ms | < 1s |
| 1000 | 25 MB | 80 ms | 2s |
| 10000 | 220 MB | 800 ms | 15s |
| 100000 | 2.1 GB | 8s | 90s |

**反直觉的点**：当 CR 数到 10000+ 时，**Operator 重启的 cold start** 会让 apiserver 在那一刻产生 List 全量的尖峰。如果你 3 副本 Operator 同时重启，apiserver 可能短暂崩溃。

应对：
- **使用 namespace scope** 缩小 watch 范围（`Cache.Namespaces`）
- **使用 label selector** 只 watch 自己关心的（`Cache.ByObject[T].Label`）
- **滚动重启**（K8s Deployment 默认就是，但要确认 `maxUnavailable`）

### 3.3 Webhook 延迟对 API 整体的影响

Webhook 是同步调用，**每个写操作都要等 webhook 返回**。

| Webhook P99 延迟 | apiserver Create P99 | 影响范围 |
| --- | --- | --- |
| 5 ms | 20 ms | 几乎无感 |
| 50 ms | 80 ms | kubectl 偶尔感觉慢 |
| 500 ms | 600 ms | **生产告警**，Pod 创建堆积 |
| 5 s | timeout（默认 10s） | **集群级故障** |

webhook 的 `timeoutSeconds` 默认 10s，最大 30s。配合 `failurePolicy=Fail`，5s+ 延迟就开始让 Pod 创建大面积失败。

### 3.4 Operator 高可用（Leader Election）

3 副本 Operator + leader election：

| 模式 | 谁在跑 Reconcile | 切换时长 | 适用场景 |
| --- | --- | --- | --- |
| 单副本 | 唯一一个 | 重启时停 5~30s | 简单场景，CR 少 |
| 3 副本 + LeaderElection | 仅 leader 一个 | leader 挂后 ~15s | **生产首选** |
| 3 副本无 LeaderElection | 全部一起跑 | 0 | **错误**，会并发改 status |

leader election 基于 Lease 资源（`coordination.k8s.io/v1`）：

```go
// main.go
mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
    LeaderElection:             true,
    LeaderElectionID:           "rediscluster-operator-leader",
    LeaderElectionResourceLock: "leases",
    LeaseDuration:              &leaseDuration,    // 默认 15s
    RenewDeadline:              &renewDeadline,    // 默认 10s
    RetryPeriod:                &retryPeriod,      // 默认 2s
})
```

参数语义：
- `LeaseDuration`：lease 的有效期，过期就被认为失去 leader
- `RenewDeadline`：leader 必须在此时间内续约成功，否则主动放弃
- `RetryPeriod`：非 leader 的副本检查 lease 的周期

**生产建议**：LeaseDuration 不要小于 30s，否则 GC STW（Go runtime gc）和网络抖动会频繁触发主备切换。

### 3.5 不同 SDK 的对比

| SDK | 语言 | 生态 | 学习曲线 | 适用 |
| --- | --- | --- | --- | --- |
| **Kubebuilder** | Go | 官方 SIG，事实标准 | 中 | 99% 场景 |
| **Operator SDK** | Go / Ansible / Helm | RedHat 主导，含 OLM 集成 | 中 | 需要 OLM、需要 Ansible |
| **Java Operator SDK** | Java | Quarkus + Fabric8 | 中 | Java 团队 |
| **kopf** | Python | 装饰器风格，简洁 | 低 | 脚本化、原型 |
| **Metacontroller** | Webhook 驱动 | 不写 Go，纯 webhook 实现 | 低 | 简单 CRUD 场景 |

> **口诀**：90% 团队选 Kubebuilder，需要 RedHat 生态选 Operator SDK，Java 团队选 Java Operator SDK，原型快速验证选 kopf。

### 3.6 火焰图能告诉你什么

`pprof` 抓 Operator 的火焰图：

```bash
go tool pprof http://localhost:8080/debug/pprof/profile?seconds=30
```

典型问题信号：
- `cache.(*Reflector).ListAndWatch` 占栈高 → 全量 List 太频繁，看 watch 是不是断了
- `Reconcile` 内部调 `Update` 频繁 → 没做 diff，每次都全量更新（案例 ① 死循环）
- `json.Unmarshal` 占大头 → CR 太大，考虑拆分或用 `metadata.managedFields` 减负
- `runtime.gcBgMarkWorker` 占 30%+ → 内存压力，看 cache 大小

---

## 4. 生产案例 9 则

> 每个案例固定四块：**现象 / 根因 / 修复 / 验证**。这一节是本文长尾价值的核心。

### 案例 ① Reconcile 不幂等导致死循环

#### 现象

新写的 Operator 上线后，apiserver QPS 飙到 5000+，监控看到单个 CR 每秒 Reconcile 几十次。CPU 100%，整个集群变慢。

#### 根因

代码：

```go
func (r *RedisClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var cr cachev1.RedisCluster
    if err := r.Get(ctx, req.NamespacedName, &cr); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    sts := &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{Name: cr.Name, Namespace: cr.Namespace},
        Spec:       buildStsSpec(&cr),
    }
    // ★ 每次都覆盖更新
    return ctrl.Result{}, r.Update(ctx, sts)
}
```

`buildStsSpec(&cr)` 里有一行用了 `time.Now()` 给某个 annotation 赋值。每次构造的 `sts.Spec` 都不一样：

1. Reconcile 第 1 次：Update sts，spec 里有 `lastUpdated: t1`
2. apiserver 写入触发 watch → workqueue.Add
3. Reconcile 第 2 次：Update sts，spec 里有 `lastUpdated: t2`（≠ t1）
4. apiserver 又触发 watch → workqueue.Add
5. **死循环**

更隐蔽的版本：把 `metav1.Time` 类型放进 spec，反序列化时纳秒精度丢失，导致前后构造的对象 `DeepEqual` 永远 false。

#### 修复

**铁律一：Reconcile 必须幂等。** "幂等"在 Operator 语境下是：**给定相同的 spec，构造出的子资源必须 byte-for-byte 一致**。

修复模板：

```go
func (r *RedisClusterReconciler) reconcileStatefulSet(ctx context.Context, cr *cachev1.RedisCluster) error {
    desired := buildStsSpec(cr)        // 期望的 spec
    desired.Name = cr.Name
    desired.Namespace = cr.Namespace
    if err := controllerutil.SetControllerReference(cr, desired, r.Scheme); err != nil {
        return err
    }

    var current appsv1.StatefulSet
    err := r.Get(ctx, client.ObjectKeyFromObject(desired), &current)
    if errors.IsNotFound(err) {
        return r.Create(ctx, desired)   // 不存在 → 创建
    }
    if err != nil {
        return err
    }

    // ★ 比较 spec，相同则不写
    if equality.Semantic.DeepEqual(current.Spec, desired.Spec) {
        return nil   // 幂等的关键
    }
    current.Spec = desired.Spec
    return r.Update(ctx, &current)
}
```

**铁律二：spec 里禁止放"每次都变"的字段**：时间戳、随机数、UUID。如果业务必须有时间戳，放 status 或 annotation。

更好的工具：`controllerutil.CreateOrUpdate`：

```go
op, err := controllerutil.CreateOrUpdate(ctx, r.Client, sts, func() error {
    sts.Spec = buildStsSpec(cr)
    return controllerutil.SetControllerReference(cr, sts, r.Scheme)
})
log.Info("reconciled sts", "op", op)   // op = "created" | "updated" | "unchanged"
```

它内部就是 Get + Diff + Update 的封装，幂等性由你 mutate 函数保证。

#### 验证

写一个测试，连续调 10 次 Reconcile，期望只有第 1 次产生写：

```go
func TestReconcileIdempotent(t *testing.T) {
    r := setupReconciler()
    cr := &cachev1.RedisCluster{...}
    require.NoError(t, r.Create(ctx, cr))

    var calls int32
    r.Client = &countingClient{Client: r.Client, updates: &calls}

    for i := 0; i < 10; i++ {
        _, err := r.Reconcile(ctx, ctrl.Request{NamespacedName: types.NamespacedName{Name: cr.Name, Namespace: cr.Namespace}})
        require.NoError(t, err)
    }
    assert.LessOrEqual(t, atomic.LoadInt32(&calls), int32(2), "expected at most 2 updates, got %d", calls)
}
```

监控层面：`workqueue_depth` 和 `workqueue_adds_total` 是否成线性增长。死循环时 adds 会爆炸。

> **口诀**：Reconcile 必须幂等。**Get → Diff → Update**，相同则跳过。spec 里禁放任何"每次都变"的字段。

---

### 案例 ② Finalizer 没移除导致资源无法删除

#### 现象

`kubectl delete rediscluster foo` 卡住不返回。`kubectl get rediscluster foo -o yaml` 看到 `metadata.deletionTimestamp` 已经设了，但资源就是不消失。

#### 根因

代码：

```go
func (r *RedisClusterReconciler) reconcileDelete(ctx context.Context, cr *cachev1.RedisCluster) (ctrl.Result, error) {
    if err := r.deleteExternalRedisInstance(ctx, cr); err != nil {
        return ctrl.Result{}, err   // ★ 失败就直接 return
    }
    controllerutil.RemoveFinalizer(cr, finalizerName)
    return ctrl.Result{}, r.Update(ctx, cr)
}

func (r *RedisClusterReconciler) deleteExternalRedisInstance(ctx context.Context, cr *cachev1.RedisCluster) error {
    return r.cloudClient.Delete(ctx, cr.Status.InstanceID)
}
```

外部资源**已经被人工删过了**（运维直接登录云平台点了删除按钮）。`cloudClient.Delete` 返回 `404 Not Found` 错误，被当成"删除失败"，finalizer 永远不移除。

更常见的版本：清理函数依赖某个 ConfigMap 里的凭证，但 ConfigMap 在 Operator 启动后被删了，清理函数永远报 `secret not found`。

#### 修复

**铁律：清理函数必须把"找不到 = 已删除 = 成功"作为基本契约。**

```go
func (r *RedisClusterReconciler) deleteExternalRedisInstance(ctx context.Context, cr *cachev1.RedisCluster) error {
    err := r.cloudClient.Delete(ctx, cr.Status.InstanceID)
    if isNotFoundErr(err) {
        return nil   // ★ 已经被删过，视为成功
    }
    return err
}
```

逃生通道（手动救场）：

```bash
# 移除 finalizer 强制让 apiserver 真正删
kubectl patch rediscluster foo -p '{"metadata":{"finalizers":null}}' --type=merge
```

**警告**：这个操作绕过了清理逻辑，**外部资源会泄漏**，只在你确认外部资源已经手动清理过时使用。

更系统的做法：给 finalizer 加超时退出：

```go
func (r *RedisClusterReconciler) reconcileDelete(ctx context.Context, cr *cachev1.RedisCluster) (ctrl.Result, error) {
    timeout := 30 * time.Minute
    if cr.DeletionTimestamp.Add(timeout).Before(time.Now()) {
        log.FromContext(ctx).Error(nil, "finalizer timeout, force remove", "cr", cr.Name)
        controllerutil.RemoveFinalizer(cr, finalizerName)
        return ctrl.Result{}, r.Update(ctx, cr)
    }
    if err := r.deleteExternalRedisInstance(ctx, cr); err != nil {
        return ctrl.Result{RequeueAfter: 30 * time.Second}, err
    }
    controllerutil.RemoveFinalizer(cr, finalizerName)
    return ctrl.Result{}, r.Update(ctx, cr)
}
```

30 分钟还清理不掉，自动放弃 + 告警，让人工介入而不是永远卡住。

#### 验证

```go
func TestFinalizerHandlesNotFound(t *testing.T) {
    r := setupReconciler()
    r.cloudClient = &fakeCloudClient{returnNotFound: true}

    cr := &cachev1.RedisCluster{
        ObjectMeta: metav1.ObjectMeta{
            Name:              "foo",
            DeletionTimestamp: &metav1.Time{Time: time.Now()},
            Finalizers:        []string{finalizerName},
        },
    }
    require.NoError(t, r.Create(ctx, cr))

    _, err := r.Reconcile(ctx, req(cr))
    require.NoError(t, err)

    var got cachev1.RedisCluster
    err = r.Get(ctx, req(cr).NamespacedName, &got)
    assert.True(t, errors.IsNotFound(err) || !controllerutil.ContainsFinalizer(&got, finalizerName))
}
```

线上排查清单：
- `kubectl describe rediscluster foo` 看 events，Operator 通常会写错误日志
- `kubectl get events --field-selector involvedObject.name=foo`
- `kubectl logs -l app=rediscluster-operator --tail=200 | grep foo`

> **口诀**：finalizer 卡死 = 100% 是清理函数返回 error。**"找不到外部资源 = 成功"是 OPERATOR 清理函数的第一公理**。

---

### 案例 ③ Status 与 Spec 混用导致数据丢失

#### 现象

Operator 跑了一段时间后，CR 的 `status.phase` 字段经常"突然变空"，但 Reconcile 日志里明明刚写过 `phase=Ready`。

#### 根因

```go
func (r *RedisClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var cr cachev1.RedisCluster
    r.Get(ctx, req.NamespacedName, &cr)

    cr.Spec.Replicas = computeDesiredReplicas(&cr)
    cr.Status.Phase = "Ready"
    cr.Status.ReadyReplicas = 3

    return ctrl.Result{}, r.Update(ctx, &cr)   // ★ 错
}
```

CRD 开了 `subresources.status: {}`。`r.Update(ctx, &cr)` 走的是 `/redisclusters/foo` 路径，**status 字段被 apiserver 整个忽略**。结果是：spec 改了，status 没改。

但日志里你看到 `cr.Status.Phase = "Ready"` 这行执行了 —— 那只是改了内存里的 struct，没写进 etcd。下次 Get 出来还是空的。

更隐蔽的版本：

```go
cr.Status.Phase = "Ready"
r.Status().Update(ctx, &cr)   // 这步对的

cr.Spec.Replicas = 5
r.Update(ctx, &cr)            // ★ 这步用的还是上面那个 cr 对象
```

第二个 Update 把"我以为已经是 Ready"的 status 又写回去 —— 但因为走的是 `/redisclusters/foo`，apiserver 忽略 status，只更新 spec。这看起来没问题，但下面这种更糟：

```go
cr.Status.Phase = "Ready"
r.Status().Update(ctx, &cr)   // status 已经是 Ready

cr.Status.ReadyReplicas = 5   // 想再改一个 status 字段
r.Status().Update(ctx, &cr)   // ★ ResourceVersion 冲突
```

第二次 Status().Update 因为没重新 Get，对象的 ResourceVersion 是旧的，apiserver 会返回 409 Conflict。

#### 修复

模板：spec 写完，必须 re-Get；status 写完，必须 re-Get。

```go
func (r *RedisClusterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var cr cachev1.RedisCluster
    if err := r.Get(ctx, req.NamespacedName, &cr); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 处理 spec
    if cr.Spec.Replicas != computeDesiredReplicas(&cr) {
        cr.Spec.Replicas = computeDesiredReplicas(&cr)
        if err := r.Update(ctx, &cr); err != nil {
            return ctrl.Result{}, err
        }
        // ★ 立即 return，下次 Reconcile 再处理 status（apiserver Update 触发了新 watch）
        return ctrl.Result{}, nil
    }

    // 处理 status（用 patch 避免 ResourceVersion 冲突）
    patch := client.MergeFrom(cr.DeepCopy())
    cr.Status.Phase = "Ready"
    cr.Status.ReadyReplicas = 3
    return ctrl.Result{}, r.Status().Patch(ctx, &cr, patch)
}
```

**关键点**：

1. spec 改完立即 return，下一次 Reconcile 再继续。这一是因为 Update 后内存 cr 的 ResourceVersion 落后；二是让逻辑更线性（每次 Reconcile 只做一种修改）。
2. status 用 `Patch(MergeFrom)` 而不是 Update。MergeFrom 用 JSON Patch，**只发改动的字段**，避免覆盖其他人改的部分（如另一个 controller 在同时修同一个 CR 的不同字段）。

更安全的写法（强制重新 Get）：

```go
// helpers
func (r *RedisClusterReconciler) updateStatus(ctx context.Context, key client.ObjectKey, mut func(*cachev1.RedisCluster)) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        var cr cachev1.RedisCluster
        if err := r.Get(ctx, key, &cr); err != nil {
            return err
        }
        mut(&cr)
        return r.Status().Update(ctx, &cr)
    })
}

// 用法
err := r.updateStatus(ctx, req.NamespacedName, func(cr *cachev1.RedisCluster) {
    cr.Status.Phase = "Ready"
    cr.Status.ReadyReplicas = 3
})
```

`retry.RetryOnConflict` 是 client-go 自带的，专门处理 ResourceVersion 冲突重试。

#### 验证

```go
func TestStatusUpdateNotLost(t *testing.T) {
    r := setupReconciler()
    cr := &cachev1.RedisCluster{...}
    require.NoError(t, r.Create(ctx, cr))

    _, err := r.Reconcile(ctx, req(cr))
    require.NoError(t, err)

    var got cachev1.RedisCluster
    require.NoError(t, r.Get(ctx, req(cr).NamespacedName, &got))
    assert.Equal(t, "Ready", got.Status.Phase)        // ★ 必须真的写进去
    assert.Equal(t, int32(3), got.Status.ReadyReplicas)
}
```

> **口诀**：spec 用 `r.Update`，status 用 `r.Status().Patch`。**永不混用，永不复用 cr 对象**，每次写完立即 return + 重新 Get。

---

### 案例 ④ Webhook 性能瓶颈拖慢 API

#### 现象

发布了一个新的 ValidatingWebhook 后，集群里所有 Pod 创建突然变慢，`kubectl create` 经常 5~10s 才返回。监控显示 apiserver 的 admission latency P99 从 50ms 飙到 5s。

#### 根因

webhook 代码：

```go
func (v *RedisClusterValidator) Handle(ctx context.Context, req admission.Request) admission.Response {
    var cr cachev1.RedisCluster
    if err := v.decoder.Decode(req, &cr); err != nil { ... }

    // ★ 每次都查一次外部 DB 看是否有冲突
    exists, err := v.externalDB.QueryClusterName(ctx, cr.Name)
    if err != nil { return admission.Errored(http.StatusInternalServerError, err) }
    if exists { return admission.Denied("cluster name already exists in external DB") }

    return admission.Allowed("")
}
```

外部 DB 查询 P99 = 4s。**apiserver 的每个 redisCluster 写请求都要等这 4s**。更糟：webhook 配的 scope 错了：

```yaml
rules:
- apiGroups: [""]
  apiVersions: ["v1"]
  resources: ["pods", "configmaps", "secrets", ...]   # ★ 误配！
```

本想拦截 `redisclusters`，结果把 Pod / ConfigMap / Secret 全拦截了。**apiserver 的所有写都在等这个 4s 的外部 DB**。

#### 修复

**第一步：scope 必须精准**：

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
webhooks:
- name: vrediscluster.kb.io
  rules:
  - apiGroups:   ["cache.example.com"]    # ★ 只拦自己的
    apiVersions: ["v1"]
    resources:   ["redisclusters"]
    operations:  ["CREATE", "UPDATE"]
  namespaceSelector:                      # 进一步缩小
    matchLabels:
      enable-rediscluster: "true"
  objectSelector:                         # 按对象 label 过滤
    matchLabels:
      managed-by: rediscluster-operator
  failurePolicy: Ignore                   # ★ 失败时放过
  timeoutSeconds: 5                       # 必须设短超时
  sideEffects: None
```

**第二步：webhook 内部禁止做长 IO**：

```go
func (v *RedisClusterValidator) Handle(ctx context.Context, req admission.Request) admission.Response {
    var cr cachev1.RedisCluster
    if err := v.decoder.Decode(req, &cr); err != nil { ... }

    // ★ 用 cache 而不是直查 DB，cache 由后台 goroutine 异步刷新
    if v.nameCache.Exists(cr.Name) {
        return admission.Denied("cluster name already exists")
    }
    return admission.Allowed("")
}

// 后台异步刷新
func (v *RedisClusterValidator) refreshLoop(ctx context.Context) {
    ticker := time.NewTicker(30 * time.Second)
    for {
        select {
        case <-ctx.Done(): return
        case <-ticker.C:
            names, _ := v.externalDB.ListClusterNames(ctx)
            v.nameCache.Replace(names)
        }
    }
}
```

**第三步：failurePolicy 设 Ignore**（除非你完全确定这个校验绝不能跳过）。

**第四步：监控 webhook 自身延迟**：

```bash
# apiserver 暴露的指标
apiserver_admission_webhook_admission_duration_seconds{name="vrediscluster.kb.io"}
apiserver_admission_webhook_request_total{name="vrediscluster.kb.io",rejected="true"}
```

#### 救火操作

如果 webhook 已经把整个集群打爆：

```bash
# 紧急下掉 webhook（apiserver 还能用）
kubectl delete validatingwebhookconfiguration vrediscluster

# 如果 apiserver 已经无法响应（极端情况），登 master 节点
# 直接编辑 /etc/kubernetes/manifests/kube-apiserver.yaml
# 加 --disable-admission-plugins=ValidatingAdmissionWebhook 临时关掉
# kubelet 会自动重启 apiserver
```

#### 验证

```go
func TestWebhookLatency(t *testing.T) {
    handler := newValidator()
    req := admission.Request{...}

    start := time.Now()
    handler.Handle(ctx, req)
    elapsed := time.Since(start)
    assert.Less(t, elapsed, 100*time.Millisecond)   // 必须够快
}
```

压测：

```bash
hey -n 1000 -c 50 -m POST -d '{...}' https://your-webhook/validate
# P99 必须 < 100ms
```

> **口诀**：webhook 是 apiserver 的同步阻塞点。**scope 精准 + 失败 Ignore + 超时短 + 内部不做长 IO**。生产前必须压测延迟。

---

### 案例 ⑤ OwnerReference 错位导致级联删除丢失资源

#### 现象

删一个 RedisCluster，预期它创建的 StatefulSet / Service / ConfigMap 一起被删，结果只删了 StatefulSet，Service 和 ConfigMap 留了下来。一段时间后 namespace 里堆满孤儿资源。

#### 根因

代码：

```go
func (r *RedisClusterReconciler) reconcileService(ctx context.Context, cr *cachev1.RedisCluster) error {
    svc := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      cr.Name + "-svc",
            Namespace: cr.Namespace,
            // ★ 忘了 OwnerReference
        },
        Spec: corev1.ServiceSpec{...},
    }
    return r.Create(ctx, svc)
}
```

K8s 的级联删除靠 `metadata.ownerReferences` 字段。owner 被删除时，gc 控制器会找出所有指向 owner 的 ownee 然后删掉。**没设 OwnerReference = 永远不会被级联删**。

更隐蔽的版本：设了，但 `Controller: false` 或 `BlockOwnerDeletion: false`，或者 owner 跨 namespace（K8s 不允许 namespaced 资源 own cluster-scoped 资源）。

最隐蔽的版本：

```go
controllerutil.SetOwnerReference(cr, svc, r.Scheme)        // 设了
controllerutil.SetOwnerReference(otherCR, svc, r.Scheme)   // 又设了一个
```

`SetOwnerReference` 允许多 owner，但 `SetControllerReference` 只允许一个 controller。多 owner 时**任意一个 owner 还活着，资源就不会被删**。

#### 修复

正确模式：

```go
func (r *RedisClusterReconciler) reconcileService(ctx context.Context, cr *cachev1.RedisCluster) error {
    svc := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      cr.Name + "-svc",
            Namespace: cr.Namespace,
        },
    }
    op, err := controllerutil.CreateOrUpdate(ctx, r.Client, svc, func() error {
        // ★ SetControllerReference（不是 SetOwnerReference）
        if err := controllerutil.SetControllerReference(cr, svc, r.Scheme); err != nil {
            return err
        }
        svc.Spec = corev1.ServiceSpec{...}
        return nil
    })
    return err
}
```

`SetControllerReference` 的含义：
- `controller: true`：表示这是"主控制者"
- `blockOwnerDeletion: true`：owner 删除前必须先删掉 controllee（保证级联完成）
- 同一个资源**只能有一个 controller 引用**

跨 namespace 的限制：

| owner scope | ownee scope | 允许？ |
| --- | --- | --- |
| Cluster | Cluster | ✅ |
| Cluster | Namespaced | ✅ |
| Namespaced | Namespaced（同 ns） | ✅ |
| Namespaced | Namespaced（跨 ns） | ❌ |
| Namespaced | Cluster | ❌ |

如果你的 owner 是 namespaced（绝大多数 CR 都是）但要 own 一个 cluster-scoped 资源（如 ClusterRole），就只能 finalizer 自己清理。

#### 删除策略

`kubectl delete` 默认是 **Background** 模式：apiserver 立即删 owner，gc 异步删 ownee。如果你需要"等所有 ownee 都删完才删 owner"，用 Foreground：

```bash
kubectl delete rediscluster foo --cascade=foreground

# 或者 yaml
kubectl delete rediscluster foo --cascade=orphan   # 不删 ownee
```

Foreground 的实现：apiserver 给 owner 加上 `metadata.finalizers: ["foregroundDeletion"]` 这个特殊 finalizer，gc 删完 ownee 后才移除。

#### 验证

```go
func TestOwnerReferenceSet(t *testing.T) {
    r := setupReconciler()
    cr := &cachev1.RedisCluster{...}
    require.NoError(t, r.Create(ctx, cr))
    _, err := r.Reconcile(ctx, req(cr))
    require.NoError(t, err)

    var svc corev1.Service
    require.NoError(t, r.Get(ctx, types.NamespacedName{Name: cr.Name + "-svc", Namespace: cr.Namespace}, &svc))
    require.Len(t, svc.OwnerReferences, 1)
    assert.Equal(t, cr.Name, svc.OwnerReferences[0].Name)
    assert.True(t, *svc.OwnerReferences[0].Controller)
    assert.True(t, *svc.OwnerReferences[0].BlockOwnerDeletion)
}
```

线上排查孤儿资源：

```bash
# 找出指定 namespace 里 OwnerReference 指向已不存在的资源
kubectl get all,configmap,secret -n redis -o json | \
    jq '.items[] | select(.metadata.ownerReferences != null) | .metadata.ownerReferences[].name' | \
    sort -u
```

> **口诀**：子资源用 `SetControllerReference`（不是 SetOwnerReference）。跨 namespace own = 不可能，要靠 finalizer 自己清。

---

### 案例 ⑥ Informer 缓存陈旧

#### 现象

用户改了一个 ConfigMap 后立刻执行 `kubectl annotate rediscluster foo trigger=now`，预期 Operator 看到新 ConfigMap 然后重启 Pod。结果 Operator Reconcile 时拿到的是旧 ConfigMap，Pod 没重启。

#### 根因

```go
func (r *RedisClusterReconciler) Reconcile(...) (...) {
    var cm corev1.ConfigMap
    r.Get(ctx, key, &cm)   // ★ 走 informer cache
    if cm.Data["password"] == cr.Status.LastAppliedPassword {
        return ctrl.Result{}, nil
    }
    ...
}
```

`r.Get` 走的是本地 informer cache。cache 由后台 watch 异步刷新，**与最新 etcd 状态有几十毫秒到几秒的差距**。用户刚改的 ConfigMap，watch 通道还没把更新推到 cache 里。

加上：默认情况下 controller-runtime 只 watch 你声明 Owns / Watches 的资源类型。如果 ConfigMap 不在 Owns/Watches 列表里，Operator 完全不会被 ConfigMap 变化触发，只会被 RedisCluster 自己的变化触发。

#### 修复

**方案 A**：把依赖资源加进 Watches，并设置 EventHandler 反向触发 Reconcile：

```go
func (r *RedisClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&cachev1.RedisCluster{}).
        Owns(&appsv1.StatefulSet{}).
        Owns(&corev1.Service{}).
        // ★ ConfigMap 不是 owned 的，但要监听
        Watches(&corev1.ConfigMap{}, handler.EnqueueRequestsFromMapFunc(r.findCRsForConfigMap)).
        Complete(r)
}

// ConfigMap 变更时，找出所有依赖它的 CR
func (r *RedisClusterReconciler) findCRsForConfigMap(ctx context.Context, obj client.Object) []reconcile.Request {
    cm := obj.(*corev1.ConfigMap)
    var crs cachev1.RedisClusterList
    r.List(ctx, &crs, client.InNamespace(cm.Namespace), client.MatchingFields{"spec.configMapName": cm.Name})
    res := make([]reconcile.Request, len(crs.Items))
    for i, cr := range crs.Items {
        res[i] = reconcile.Request{NamespacedName: client.ObjectKeyFromObject(&cr)}
    }
    return res
}
```

`Watches + EnqueueRequestsFromMapFunc` 的组合让 ConfigMap 变化能反向触发对应 CR 的 Reconcile。

**方案 B**：需要强一致读时绕过 cache：

```go
func (r *RedisClusterReconciler) Reconcile(...) (...) {
    var cm corev1.ConfigMap
    // ★ 用 APIReader，直连 apiserver
    if err := r.APIReader.Get(ctx, key, &cm); err != nil { ... }
}
```

APIReader 不走 cache，每次都打 apiserver。**慎用**，QPS 上去会被 apiserver 限流。

**方案 C**：让用户在 CR 上加 annotation 触发（已经在用了），但**确保 Reconcile 是从 cr.Annotations 而不是 cm.Data 决策**：

```go
// 用 hash 而不是直接读 cm 内容
hash := computeConfigHash(cm)
if cr.Annotations["last-config-hash"] == hash {
    return ctrl.Result{}, nil
}
```

#### 验证

```go
func TestConfigMapTriggersReconcile(t *testing.T) {
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{Name: "redis-config", Namespace: "default"},
        Data:       map[string]string{"password": "old"},
    }
    require.NoError(t, mgr.GetClient().Create(ctx, cm))

    cr := &cachev1.RedisCluster{...}
    require.NoError(t, mgr.GetClient().Create(ctx, cr))

    // 等第一次 Reconcile 完成
    Eventually(func() string {
        var got cachev1.RedisCluster
        mgr.GetClient().Get(ctx, key(cr), &got)
        return got.Status.LastAppliedPassword
    }).Should(Equal("old"))

    // 改 ConfigMap，期望触发再次 Reconcile
    cm.Data["password"] = "new"
    require.NoError(t, mgr.GetClient().Update(ctx, cm))

    Eventually(func() string {
        var got cachev1.RedisCluster
        mgr.GetClient().Get(ctx, key(cr), &got)
        return got.Status.LastAppliedPassword
    }, 10*time.Second).Should(Equal("new"))
}
```

监控指标：
- `controller_runtime_reconcile_total{result="success"}` 应该在 ConfigMap 改后立即跳一下
- `workqueue_depth` 应该短暂上升

> **口诀**：默认走 informer cache，要强一致读用 APIReader。**依赖资源必须加 Watches**，否则永远不会被反向触发。

---

### 案例 ⑦ CRD 升级（v1alpha1 → v1）的兼容性

#### 现象

把 CRD 从 v1alpha1 升到 v1，新增了 `spec.tlsEnabled` 字段、删了 `spec.legacyAddr`。升级后老的 v1alpha1 CR 报错无法读取，新创建的 v1 CR 也时不时报"unknown field"。

#### 根因

CRD 升级要解决三个独立问题：

**问题 1：版本共存**。同一个 CRD 可以同时 served 多个版本，但只有一个是 storage 版本（etcd 里的存储格式）：

```yaml
spec:
  versions:
  - name: v1alpha1
    served: true        # 仍提供 API
    storage: false      # 不再存
  - name: v1
    served: true
    storage: true       # 新存储版本
```

**问题 2：版本转换**。client 用 v1alpha1 读，etcd 里存的是 v1 —— apiserver 怎么转？答案是**Conversion Webhook**。如果你只声明了多版本但没写 conversion webhook，apiserver 会 fallback 到 `None` 策略 = **直接报错**。

**问题 3：字段废弃**。删除字段不能直接干。OpenAPI schema 里删掉 `legacyAddr` 后，老 CR 上的这个字段会被 apiserver 在写入时静默丢弃，丢失业务数据。

#### 修复

**完整的升级流程**：

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: redisclusters.cache.example.com
spec:
  conversion:
    strategy: Webhook
    webhook:
      clientConfig:
        service:
          name: redis-operator-webhook
          namespace: redis-system
          path: /convert
        caBundle: <base64-ca>
      conversionReviewVersions: ["v1"]
  versions:
  - name: v1alpha1
    served: true
    storage: false
    schema:
      openAPIV3Schema: {...}    # 老 schema
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema: {...}    # 新 schema
```

Conversion webhook 实现：

```go
// internal/conversion/rediscluster.go
func ConvertTo(src, dst runtime.Object) error {
    in := src.(*v1alpha1.RedisCluster)
    out := dst.(*v1.RedisCluster)
    out.ObjectMeta = in.ObjectMeta
    out.Spec.Replicas = in.Spec.Replicas
    out.Spec.Version = in.Spec.Version
    // ★ 新字段填默认值
    out.Spec.TLSEnabled = false
    // ★ 老字段映射到 annotation 保留信息
    if in.Spec.LegacyAddr != "" {
        if out.Annotations == nil { out.Annotations = map[string]string{} }
        out.Annotations["legacy-addr"] = in.Spec.LegacyAddr
    }
    return nil
}

func ConvertFrom(src, dst runtime.Object) error {
    in := src.(*v1.RedisCluster)
    out := dst.(*v1alpha1.RedisCluster)
    out.ObjectMeta = in.ObjectMeta
    out.Spec.Replicas = in.Spec.Replicas
    out.Spec.Version = in.Spec.Version
    // ★ 反向：从 annotation 还原老字段
    if v, ok := in.Annotations["legacy-addr"]; ok {
        out.Spec.LegacyAddr = v
    }
    return nil
}
```

Kubebuilder 的实现入口：

```go
// api/v1alpha1/conversion.go
func (src *RedisCluster) ConvertTo(dstRaw conversion.Hub) error {
    dst := dstRaw.(*v1.RedisCluster)
    return ConvertTo(src, dst)
}

func (dst *RedisCluster) ConvertFrom(srcRaw conversion.Hub) error {
    src := srcRaw.(*v1.RedisCluster)
    return ConvertFrom(src, dst)
}
```

`Hub` 是中转版本（推荐用最新 storage 版本作 hub），所有版本互转都走 hub，避免 N×N 转换函数。

**升级步骤**（生产）：

1. 部署新版 Operator + 新 CRD（包含 conversion webhook）
2. 验证 conversion 工作（创建 v1alpha1 CR，用 `kubectl get -o yaml redisclusters.v1.cache.example.com foo` 看输出）
3. 用 `kubectl convert` 或脚本把所有 CR 显式重写一次（强制走 storage 版本）
4. 一段时间后，从 versions 列表里下掉 v1alpha1（先 `served: false`，再删除）

```bash
# 强制把所有 v1alpha1 的 CR 重写成 storage 版本
for ns in $(kubectl get ns -o name); do
  for cr in $(kubectl get rediscluster -n ${ns#*/} -o name); do
    kubectl get -n ${ns#*/} $cr -o yaml | kubectl replace -f -
  done
done
```

#### 验证

```bash
# v1alpha1 读
kubectl get redisclusters.v1alpha1.cache.example.com -o yaml

# v1 读（同一个底层对象）
kubectl get redisclusters.v1.cache.example.com -o yaml

# 字段映射对吗？
diff <(kubectl get redisclusters.v1alpha1... -o yaml) <(kubectl get redisclusters.v1... -o yaml)
```

测试：

```go
func TestConversionRoundTrip(t *testing.T) {
    src := &v1alpha1.RedisCluster{Spec: v1alpha1.RedisClusterSpec{Replicas: 3, LegacyAddr: "old:6379"}}
    var hub v1.RedisCluster
    require.NoError(t, src.ConvertTo(&hub))

    var back v1alpha1.RedisCluster
    require.NoError(t, back.ConvertFrom(&hub))
    assert.Equal(t, src.Spec, back.Spec)   // 不能丢字段
}
```

> **口诀**：CRD 多版本必须配 conversion webhook，否则 apiserver 报错。**老字段映射到 annotation 保留**，留升级回退余地。

---

### 案例 ⑧ Operator 自身的 RBAC 配置过宽

#### 现象

安全审计报告：`rediscluster-operator` ServiceAccount 拥有 `cluster-admin` 等同权限，能删除任意 namespace 的所有资源。

#### 根因

Kubebuilder 默认生成的 ClusterRole 标记：

```go
// +kubebuilder:rbac:groups="*",resources="*",verbs="*"
func (r *RedisClusterReconciler) Reconcile(...) {...}
```

或者懒人写法：

```yaml
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

把通配符全开，相当于把 cluster-admin 给了 Operator。一旦 Operator 进程被攻陷（CVE、镜像污染、配置错误），攻击者拿到的就是全集群权限。

#### 修复

**最小权限原则**。Kubebuilder 的注释式 RBAC 一定要精准：

```go
// +kubebuilder:rbac:groups=cache.example.com,resources=redisclusters,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=cache.example.com,resources=redisclusters/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=cache.example.com,resources=redisclusters/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="",resources=services;configmaps,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="",resources=secrets,verbs=get;list;watch
// +kubebuilder:rbac:groups="",resources=events,verbs=create;patch
func (r *RedisClusterReconciler) Reconcile(...) {...}
```

跑 `make manifests` 生成的 ClusterRole 只包含上述权限。

**关键检查项**：

1. **不要 verbs="*"**：明确列出 get/list/watch/create/update/patch/delete
2. **不要 resources="*"**：列出确实需要的
3. **secrets 通常只需要 get/list/watch**：除非业务真的要创建/更新 secret，否则别给 write
4. **`/finalizers` 子资源单独列**：要 update finalizer 必须有它的权限
5. **能用 Role（namespace scope）就别用 ClusterRole**

namespace scope 的 Operator：

```go
mgr, _ := ctrl.NewManager(cfg, ctrl.Options{
    Cache: cache.Options{
        DefaultNamespaces: map[string]cache.Config{
            "redis-system": {},
            "app-prod":     {},
        },
    },
})
```

配套的 RBAC 用 RoleBinding（不是 ClusterRoleBinding）。

#### 审计工具

```bash
# 列出 SA 的所有权限
kubectl auth can-i --list --as=system:serviceaccount:redis-system:rediscluster-operator

# 检查 ClusterRole 里有没有 *
kubectl get clusterrole rediscluster-operator-manager-role -o yaml | grep -E '"\*"'
```

集成到 CI：

```bash
# 用 rbac-tool（或 kubectl-who-can）
rbac-tool analysis --service-account redis-system/rediscluster-operator | grep -E "danger|warn"
```

#### 验证

```bash
# 应该 yes
kubectl auth can-i get rediscluster --as=system:serviceaccount:redis-system:rediscluster-operator

# 应该 no
kubectl auth can-i delete pod --as=system:serviceaccount:redis-system:rediscluster-operator -n kube-system
kubectl auth can-i create secret --as=system:serviceaccount:redis-system:rediscluster-operator
```

线上加一个准入策略（PSA / OPA Gatekeeper / Kyverno），禁止部署带 `*` 通配权限的 ClusterRole：

```yaml
# Kyverno
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: no-wildcard-rbac
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-clusterrole
    match:
      resources:
        kinds: [ClusterRole, Role]
    validate:
      message: "wildcard verbs/resources are not allowed"
      pattern:
        rules:
        - resources:
            resources: "X(*)"   # X = not equal
          verbs: "X(*)"
```

> **口诀**：Operator 的 RBAC 永远是 **明确列出的最小集**。verbs 不要 *，resources 不要 *。生产前用 `kubectl auth can-i --list` 审一遍。

---

### 案例 ⑨ Operator 在多集群下的部署模式

#### 现象

公司有 5 个 K8s 集群（生产、灾备、测试 ×3），同一个 RedisCluster CRD 想统一管理。要不要每个集群都装一份 Operator？

#### 三种主流模式

**模式 A：每集群独立 Operator**

```
cluster-A: [operator-A] → 管 cluster-A 的 CR
cluster-B: [operator-B] → 管 cluster-B 的 CR
...
```

- ✅ 故障域隔离：A 集群 Operator 挂了不影响 B
- ✅ RBAC 简单：每个 Operator 只需要本集群权限
- ❌ 配置漂移：N 份 Operator 版本可能不一致
- ❌ 跨集群 CR 共享需要额外同步机制

**模式 B：中心 Operator，远程操作多集群**

```
管控集群: [central-operator]
                │
        ┌───────┼───────┐
        ▼       ▼       ▼
   cluster-A  cluster-B  cluster-C
```

- ✅ 单点管理，CR 在管控集群上，Operator 用 kubeconfig 连各业务集群
- ✅ 适合"管控+数据"分层架构
- ❌ 网络故障：管控集群 → 业务集群断开，业务集群上的 CR 无法 reconcile
- ❌ kubeconfig 凭证管理复杂

实现要点：

```go
// 给每个目标集群创建独立的 Manager
for _, cluster := range targetClusters {
    cfg, _ := clientcmd.BuildConfigFromFlags("", cluster.Kubeconfig)
    mgr, _ := ctrl.NewManager(cfg, ctrl.Options{...})
    go mgr.Start(ctx)
}
```

或用 [Cluster API](https://cluster-api.sigs.k8s.io/) / [Karmada](https://karmada.io/) 这类多集群框架。

**模式 C：Hub-Spoke + 远程 CR 同步**

```
hub: [hub-operator] + [karmada/argocd]
       │ 把 CR 同步到 spoke
       ▼
spoke-A: [spoke-agent] → 在本地 reconcile
```

- ✅ CR 集中定义（hub）但执行在本地（spoke）
- ✅ 网络故障时 spoke 仍能继续 reconcile（最终一致）
- ❌ 复杂度最高，需要专门的同步框架

#### 选型建议

| 场景 | 推荐模式 |
| --- | --- |
| 集群数 ≤ 3，CR 不需要跨集群 | A |
| 集群数 5+，CR 需要中心管理但不要求强一致 | C（Karmada / ArgoCD） |
| 严格分离管控面 / 数据面 | B（自研或 Cluster API） |
| 跨云、跨地域 | C |

#### 通用注意事项

1. **CRD 必须在每个集群都装**：CRD 是集群作用域，不能跨集群。版本必须一致。
2. **Operator 镜像版本必须一致**：避免 v1.0 在 A 集群、v1.1 在 B 集群导致行为差异。
3. **Webhook 服务发现**：webhook 是 in-cluster Service，跨集群部署时每个集群都要有自己的 webhook 实例。
4. **Leader Election 是 per-cluster 的**：A 集群的 Operator leader election 与 B 集群无关。

#### 验证

```bash
# 验证多集群 CRD 版本一致
for cluster in $clusters; do
    kubectl --context $cluster get crd redisclusters.cache.example.com -o jsonpath='{.spec.versions[*].name}'
done

# 验证镜像版本一致
for cluster in $clusters; do
    kubectl --context $cluster get deploy -n redis-system rediscluster-operator -o jsonpath='{.spec.template.spec.containers[0].image}'
done
```

> **口诀**：3 个以下集群 → 每集群独立 Operator；中心化诉求强 → Karmada / ArgoCD 同步 CR；严格分层 → Cluster API。**永远确认 CRD 和镜像版本在多集群间一致**。

---

## 5. 上线检查清单

打印贴墙 / 加进 Code Review checklist。

### CRD 设计层
- [ ] CRD 启用 `subresources.status: {}`，spec/status 严格分离
- [ ] OpenAPI schema 标注 minimum / maximum / pattern / required，杜绝运行时校验
- [ ] 用 `metav1.Condition` 而不是自定义 phase 字符串
- [ ] `printcolumn` 配齐，`kubectl get` 出来一眼能看到关键状态
- [ ] 字段命名遵循 K8s 惯例（camelCase, 不用动词前缀）
- [ ] 涉及缩放的 CR 配 `subresources.scale`（HPA 才能用）

### Reconcile 层
- [ ] Reconcile 函数完全幂等（spec 不变 → 多次调用结果完全一致）
- [ ] 用 `Get → Diff → Update` 模式，不无脑覆盖
- [ ] spec 写完立即 return，不与 status 写在同一次 Reconcile
- [ ] status 用 `r.Status().Patch(MergeFrom)` 而不是 Update
- [ ] 业务等待用 `RequeueAfter` 而不是 `return error`
- [ ] 处理 NotFound 用 `client.IgnoreNotFound(err)`
- [ ] 子资源用 `SetControllerReference`（不是 SetOwnerReference）

### Finalizer 层
- [ ] CR 类型有 finalizer，外部资源清理逻辑有兜底
- [ ] 清理函数把"NotFound = 成功"作为契约
- [ ] 加 finalizer 和清理逻辑都判断 `DeletionTimestamp`
- [ ] 给 finalizer 加超时退出（避免永远卡死）

### Webhook 层
- [ ] webhook 的 scope 精准（apiGroups / resources / verbs 别用 *）
- [ ] `failurePolicy` 默认 Ignore，除非校验绝不能跳过
- [ ] `timeoutSeconds` 设短（≤ 5s）
- [ ] webhook 内部不做长 IO，必要时用 cache + 后台异步刷新
- [ ] 提前压测 webhook P99 延迟（应 < 100ms）
- [ ] 配 `objectSelector` / `namespaceSelector` 进一步缩小

### RBAC 层
- [ ] verbs 不用 *，明确列出 get/list/watch/create 等
- [ ] resources 不用 *，明确列出
- [ ] 能用 Role 就不用 ClusterRole
- [ ] CI 跑 `kubectl auth can-i --list` 验证最小权限

### 高可用层
- [ ] Operator Deployment ≥ 2 副本 + LeaderElection 开启
- [ ] LeaseDuration ≥ 30s，避免 GC STW 频繁切换
- [ ] PodDisruptionBudget 配 `minAvailable: 1`
- [ ] 健康探针 readiness / liveness 都有

### 性能与资源层
- [ ] CR 数量 1000+ 时配 label selector 缩小 watch 范围
- [ ] Operator 容器有 resources.requests/limits（Go 服务推荐 200m CPU / 256Mi mem 起）
- [ ] 大量 CR 场景跑 pprof 看内存占用
- [ ] `RequeueAfter` 不要 < 30s，除非有强业务理由

### 监控与可观测层
- [ ] 暴露 controller-runtime 默认指标（`/metrics`）
- [ ] 关键指标告警：`controller_runtime_reconcile_errors_total`、`workqueue_depth`、`workqueue_unfinished_work_seconds`
- [ ] 业务事件用 `r.Recorder.Event(cr, "Normal", "Reason", "msg")` 写到 K8s Events
- [ ] 日志带 `logr` 风格的结构化字段（namespace / name / generation）

### 升级层
- [ ] CRD 多版本必须配 Conversion Webhook
- [ ] 老字段不直接删，先映射到 annotation 留 1~2 个版本
- [ ] 镜像 tag 用 semver，禁止 `latest`
- [ ] Helm chart 提供 `pre-install` / `pre-upgrade` hook 跑兼容性检查

### 安全层
- [ ] Operator 容器以非 root 用户运行（`runAsNonRoot: true`）
- [ ] `readOnlyRootFilesystem: true`
- [ ] 镜像签名（cosign）+ admission policy 验签
- [ ] webhook 证书自动轮转（cert-manager）

---

## 6. 还能往下挖什么

读完上面 9 个案例只是起点。如果想再深一层：

### Java Operator SDK（Quarkus + Fabric8）
- Java 团队的福音：用 Quarkus + [java-operator-sdk](https://github.com/operator-framework/java-operator-sdk) 写 Operator，Fabric8 客户端比 client-go 在 Java 生态更成熟
- 注意：Java 启动慢、内存占用高，建议配合 GraalVM native-image 编译
- 适合公司技术栈以 Java 为主、Operator 业务逻辑复杂（需要复用 Java 业务代码）的场景

### OperatorHub / OLM (Operator Lifecycle Manager)
- [OLM](https://olm.operatorframework.io/) 是 RedHat 的 Operator 生命周期管理框架
- 提供 Operator 的安装 / 升级 / 依赖管理 / 多版本共存
- 配合 [OperatorHub.io](https://operatorhub.io/) 发布给广义生态用
- 复杂度高，简单业务别上 —— 你公司就那几个 Operator，CD 工具直接搞定

### Crossplane（基础设施 Operator）
- [Crossplane](https://crossplane.io/) 把 Operator 模式应用到基础设施层
- 用 K8s CR 定义云资源（RDS / S3 / VPC），让 K8s 成为统一的"基础设施 API"
- Composition / XRD（Composite Resource Definition）实现"高阶 CR" 抽象
- 看它的 Provider 实现是学习 Operator 高级用法的好材料

### Kubebuilder v4 新特性
- 集成 Helm 生成器（不用自己手写 chart）
- ComponentConfig：Manager 配置从命令行 flag 改成 yaml 文件
- 多 Group 项目原生支持
- 升级文档：[Migration Guide](https://book.kubebuilder.io/migration/v3vsv4)

### Webhook 的服务发现
- in-cluster Service 是默认方式，但要解决证书签发（cert-manager 是事实标准）
- URL 模式：webhook 部署在集群外，apiserver 用 `clientConfig.url` 直连
- 选择标准：高可用要求 → Service；调试期 → URL

### 多版本 CRD 的转换 Webhook
- conversion webhook 走的是 webhook 服务，与 admission webhook 共享同一套部署
- Hub & Spoke 模式：所有版本互转都先转 hub 版本，避免 N×N 转换矩阵
- Kubebuilder 提供 `--conversion` flag 自动生成骨架

### Operator 与 GitOps 的协作
- ArgoCD / Flux 把 CR 当成"应用配置"管理
- CR 的 spec 由 Git 管理，status 由 Operator 写 → **GitOps 工具不要 sync status**
- 配套：`.argoignore` 或 ArgoCD 的 `ignoreDifferences` 排除 status 字段

### Controller-runtime 内部机制
- Source / EventHandler / Predicate 三件套：watch 哪个资源、怎么转 reconcile.Request、过不过滤
- Cache 的 SelectorOptions：精确控制每种资源的 watch 范围
- ManagedFields 与 Server-Side Apply：避免多 controller 互相覆盖字段

### Reconcile 的高级模式
- **Phase machine**：CR 的 status 显式建模成有限状态机（Pending → Provisioning → Ready → Failed）
- **Sub-reconciler**：把大的 Reconcile 拆成多个 sub-reconciler，每个负责一类子资源
- **Result.Requeue 的细颗粒控制**：根据失败类型决定退避时长，而不是依赖默认指数退避

---

## 7. 引用

- 官方文档：[Kubebuilder Book](https://book.kubebuilder.io/)
- [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime) 源码
- [client-go](https://github.com/kubernetes/client-go) Informer / Workqueue 实现
- CoreOS, *Introducing Operators*（2016 年原始博客，定义了 Operator 模式）
- [Operator SDK](https://sdk.operatorframework.io/)
- [Java Operator SDK](https://github.com/operator-framework/java-operator-sdk)
- [Crossplane](https://crossplane.io/) Composition 设计文档
- KEP-95: [Custom Resource Definitions](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/95-custom-resource-definitions)
- KEP-1610: [CRD Defaulting](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/1610-crd-defaulting)
- 《Programming Kubernetes》— Michael Hausenblas, Stefan Schimanski（O'Reilly，深入 client-go 与 Operator 内部）
- [Awesome Operators](https://github.com/operator-framework/awesome-operators) 社区精选

---

> 配套：[Kubernetes 实战](/article/k8s/k8s-essentials) | [Service Mesh](/article/k8s/service-mesh) | [DevOps CI/CD](/article/devops/cicd)
> 这是按 [桃花岛深度模板](/STYLE) 写的深文。如果你觉得这个深度对，告诉我下一篇该写哪个。
