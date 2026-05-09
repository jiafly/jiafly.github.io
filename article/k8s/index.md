# Kubernetes 与云原生

> K8s 与 Service Mesh 不是 YAML 教程。这一节要讲的是：**Pod 怎么挂、流量怎么不通、Sidecar 怎么吃 CPU、xDS 怎么爆配置**。

---

## 已完成的深文

- ✅ **[Kubernetes 实战 —— 控制平面到生产排查](/article/k8s/k8s-essentials)** · 1470 行
- ✅ **[Service Mesh 与 Istio 深入](/article/k8s/service-mesh)** · 1428 行
- ✅ **[Operator 与 CRD 开发 —— Reconcile Loop 到生产级 Operator](/article/k8s/operator)** · 1825 行
- ✅ **[eBPF 与 Cilium —— 内核态可编程与下一代数据面](/article/k8s/ebpf-cilium)** · 1652 行

---

## 规划中的深文

### 3. Operator 与 CRD 开发 ✅ 已完成 → [阅读深文](/article/k8s/operator)
**核心问题**：什么时候自己写 Operator？Reconcile loop 的真实代价？kubebuilder 与 Operator SDK 怎么选？

### 4. Kubernetes 网络模型深入
**核心问题**：CNI 插件（Calico / Cilium / Flannel）的本质差别？eBPF 数据面凭什么取代 iptables？NetworkPolicy 在 100 节点规模下的代价？

### 5. Kubernetes 存储深入
**核心问题**：CSI 驱动的工作原理？StatefulSet 的 PVC 在节点故障时的恢复？块存储 vs 文件存储 vs 对象存储 在 K8s 里的对接？

### 6. GitOps 与 ArgoCD 实战
**核心问题**：pull 模型 vs push 模型的本质差别？ArgoCD 与 Flux 怎么选？多集群 GitOps？

### 7. K8s 多集群与边缘
**核心问题**：Karmada / Submariner / Open Cluster Management 怎么选？边缘 K8s（K3s / KubeEdge）的真实场景？

---

## 关联现有文章
- [容器与 cgroup](/article/linux-perf/container-cgroup)
- [JVM GC 深入](/article/jvm/gc-deep) — JVM 在容器中的内存管理
- [DevOps CI/CD](/article/devops/cicd)
- [注册中心对比](/article/distributed/service-discovery) — 与 Service Mesh 的边界

[告诉我](/article/about) 你最想先看哪个主题。
