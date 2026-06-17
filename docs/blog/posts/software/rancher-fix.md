---
layout: post
date: 2026-06-17
tags: [kubernetes,network,rke2,rancher,nginx,traefik,software]
categories:
    - software
---

Q:删除nginx webhook后更新rancher 的helm 配置,改为traefik 问题解决了,帮我复盘下 


A:
可以，整个问题链路其实很典型：**Ingress Controller 切换过程中，旧 nginx 没清干净，新 Traefik 又一开始没完全可用，导致 Rancher 入口中断。**   

复盘如下。

------

# **1. 最初的调度失败原因**

你最开始看到的是：

```text
0/3 nodes are available:
1 node(s) didn't have free ports for the requested pod ports,
2 node(s) didn't satisfy plugin(s) [NodeAffinity]
```

后面看 Traefik Pod YAML，发现它是 DaemonSet 创建的，并且每个 Pod 会被固定调度到对应节点，例如：

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchFields:
      - key: metadata.name
        operator: In
        values:
        - host-203
```

这说明：

```text
这个 Traefik Pod 只能调度到 host-203
```

同时 Traefik 配了：

```yaml
hostPort: 80
hostPort: 443
```

所以它必须占用宿主机的 80 和 443。

------

# **2. 真正的端口冲突来源**

你用命令查到：

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{" "}{.metadata.name}{" node="}{.spec.nodeName}{" hostPorts="}{range .spec.containers[*].ports[*]}{.hostPort}{"/"}{.protocol}{" "}{end}{"\n"}{end}' | grep -E '80/TCP|443/TCP'
```

结果里有：

```text
kube-system rke2-ingress-nginx-controller-7h97n node=host-201 hostPorts=80/TCP 443/TCP
kube-system rke2-ingress-nginx-controller-vwfgs node=host-202 hostPorts=80/TCP 443/TCP
kube-system rke2-ingress-nginx-controller-sqjft node=host-203 hostPorts=80/TCP 443/TCP
```

同时 Traefik 也要：

```text
default rke2-traefik-* hostPorts=80/TCP 443/TCP
```

所以根因是：

```text
rke2-ingress-nginx-controller 已经占用了每个节点的 80/443
rke2-traefik 也想占用每个节点的 80/443
同一个 Node 上同一个 hostPort 不能被两个 Pod 同时使用
```

因此 Traefik 的 Pod 一开始无法调度。

------

# **3. 为什么删除 Service 没用？**

你删除了 nginx 的 Service，但 nginx controller Pod 还是重新调度。

原因是：

```text
Service 只负责暴露访问入口
不负责管理 Pod 生命周期
```

真正管理这些 Pod 的是：

```text
DaemonSet/rke2-ingress-nginx-controller
```

所以：

```text
删除 Service：不会删除 Pod
删除 Pod：DaemonSet 会重建
删除 DaemonSet：Pod 才会停止，但如果 RKE2 Addon/HelmChart 还在，也可能再次重建
```

所以要彻底不用 nginx ingress，需要处理它背后的：

```text
DaemonSet
HelmChart
RKE2 内置 addon 配置
admission webhook
```

------

# **4. Rancher 为什么打不开？**

你的 Rancher Ingress 是：

```text
Namespace: cattle-system
Name: rancher
Ingress Class: <none>
Address: 192.168.31.201,192.168.31.202,192.168.31.203
Events From: nginx-ingress-controller
```

虽然它没有显式写：

```yaml
ingressClassName: nginx
```

但从 Events 可以看出，之前实际是 **nginx ingress controller 在处理 Rancher 的 Ingress**：

```text
nginx-ingress-controller Scheduled for sync
```

所以 Rancher 访问链路原来大概是：

```text
浏览器访问 rancher.edelman999.com:443
    ↓
节点 443
    ↓
rke2-ingress-nginx-controller
    ↓
Ingress/cattle-system/rancher
    ↓
Service/cattle-system/rancher
    ↓
Rancher Pod
```

当你删除或破坏 nginx ingress 后，如果 Traefik 还没有正常接管，那么链路就断了：

```text
没有可用的 80/443 Ingress Controller
Rancher 域名自然打不开
```

------

# **5. Helm upgrade 为什么失败？**

你执行：

```bash
helm upgrade rancher rancher-latest/rancher \
  -n cattle-system \
  --reuse-values \
  --set ingress.ingressClassName=traefik
```

报错：

```text
failed calling webhook "validate.nginx.ingress.kubernetes.io"
service "rke2-ingress-nginx-controller-admission" not found
```

这个是关键转折点。

它说明：

```text
nginx ingress admission webhook 配置还在
但 webhook 指向的 Service 已经被删了
```

也就是集群进入了一个“半删除”状态：

```text
ValidatingWebhookConfiguration 还在
rke2-ingress-nginx-controller-admission Service 不在
```

Kubernetes API Server 在创建/更新 Ingress 前，会调用这个 webhook 做校验：

```text
helm upgrade 修改 Rancher Ingress
    ↓
API Server
    ↓
调用 validate.nginx.ingress.kubernetes.io
    ↓
访问 rke2-ingress-nginx-controller-admission.kube-system.svc
    ↓
Service 不存在
    ↓
Ingress 更新失败
    ↓
Helm upgrade 失败
```

所以当时问题已经不是 Helm 参数错，而是 **nginx admission webhook 残留阻塞了所有 Ingress 更新**。

------

# **6. 最终解决动作为什么有效？**

你删除 nginx webhook 后，再更新 Rancher Helm 配置为 Traefik，问题解决。

这说明最终链路变成了：

```text
删除残留 nginx webhook
    ↓
API Server 不再调用不存在的 nginx admission service
    ↓
Rancher Ingress 可以正常更新
    ↓
spec.ingressClassName 改为 traefik
    ↓
Traefik 接管 Rancher Ingress
    ↓
Rancher 恢复访问
```

最终关键配置应该类似：

```yaml
spec:
  ingressClassName: traefik
```

也就是 Rancher 不再被 nginx ingress 处理，而是交给 Traefik。

------

# **7. 整个问题的根因总结**

可以分成 3 层。

## **第一层：端口冲突**

```text
nginx ingress 和 Traefik 同时使用 hostPort 80/443
```

这是最开始 Traefik Pending 的原因。

------

## **第二层：Ingress Controller 切换不完整**

你想从 nginx ingress 切到 Traefik，但一开始：

```text
nginx 没完全清理
Traefik 没完全接管
Rancher Ingress 还处于 nginx 处理逻辑中
```

所以 Rancher 打不开。

------

## **第三层：nginx admission webhook 残留**

删除 nginx 相关 Service 后，还留下了：

```text
ValidatingWebhookConfiguration/validate.nginx.ingress.kubernetes.io
```

它继续拦截 Ingress 更新，但指向的 Service 已经不存在。

所以 Helm upgrade 被阻断。

------

# **8. 这次问题最关键的判断点**

几个关键命令起了作用：

## **查 hostPort 冲突**

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{" "}{.metadata.name}{" node="}{.spec.nodeName}{" hostPorts="}{range .spec.containers[*].ports[*]}{.hostPort}{"/"}{.protocol}{" "}{end}{"\n"}{end}' | grep -E '80/TCP|443/TCP'
```

确认 nginx 占了 80/443。

------

## **查 Rancher Ingress 当前由谁处理**

```bash
kubectl describe ingress -n cattle-system rancher
```

看到：

```text
Ingress Class: <none>
Events From: nginx-ingress-controller
```

说明 Rancher 原来被 nginx 处理。

------

## **Helm upgrade 报 webhook 错**

```text
failed calling webhook "validate.nginx.ingress.kubernetes.io"
service "rke2-ingress-nginx-controller-admission" not found
```

直接定位到：

```text
nginx webhook 残留
```

------

# **9. 以后类似切换建议流程**

如果以后从一个 Ingress Controller 切到另一个，建议按这个顺序：

## **1. 先确认新 controller 正常**

```bash
kubectl get pods -A | grep -i traefik
kubectl get ingressclass
```

确认 Traefik Running，并且有：

```text
traefik
```

IngressClass。

------

## **2. 避免 hostPort 冲突**

如果两个 controller 都要占：

```text
80
443
```

只能保留一个，或者让它们跑在不同节点，或者改一个不用 hostPort。

------

## **3. 修改业务 IngressClass**

例如 Rancher：

```bash
helm upgrade rancher rancher-latest/rancher \
  -n cattle-system \
  --reuse-values \
  --set ingress.ingressClassName=traefik
```

或者临时 patch：

```bash
kubectl patch ingress rancher -n cattle-system \
  --type='merge' \
  -p '{"spec":{"ingressClassName":"traefik"}}'
```

但 Helm 管理的资源，最好用 Helm 改。

------

## **4. 再清理旧 controller**

旧 nginx 不用了再删：

```bash
kubectl delete ds -n kube-system rke2-ingress-nginx-controller
```

如果是 RKE2 内置组件，还要在 RKE2 配置里禁用：

```yaml
disable:
  - rke2-ingress-nginx
```

------

## **5. 清理旧 admission webhook**

不用 nginx 后，确认是否残留：

```bash
kubectl get validatingwebhookconfiguration | grep -i nginx
kubectl get mutatingwebhookconfiguration | grep -i nginx
```

确认不用后再删：

```bash
kubectl delete validatingwebhookconfiguration <nginx-webhook-name>
```

------

# **10. 一句话复盘**

这次问题本质是：

```text
RKE2 集群里 nginx ingress 和 Traefik 同时争抢 80/443，
Rancher 原来由 nginx ingress 暴露；
删除 nginx 相关资源后，nginx admission webhook 残留，
导致 Rancher Ingress 无法通过 Helm 更新到 Traefik；
删除残留 webhook 后，Rancher Ingress 成功切换到 Traefik，访问恢复。
```

最终你做对的关键动作是：

```text
1. 找到 80/443 是 rke2-ingress-nginx-controller 占用
2. 明确 Rancher Ingress 原来被 nginx 处理
3. 删除残留 nginx admission webhook
4. 用 Helm 把 Rancher ingressClassName 改为 traefik
```

这就是问题解决的完整闭环。