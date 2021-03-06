# Istio 和 Service Mesh

Istio 是 Google、IBM 和 Lyft 联合开源的微服务 Service Mesh 框架，旨在解决大量微服务的发现、连接、管理、监控以及安全等问题。Istio 对应用是透明的，不需要改动任何服务代码就可以实现透明的服务治理。

Istio 的主要特性包括：

- HTTP、gRPC 和 TCP 网络流量的自动负载均衡
- 丰富的路由规则，细粒度的网络流量行为控制
- 流量加密、服务间认证，以及强身份声明
- 全范围（Fleet-wide）策略执行
- 深度遥测和报告

> 注意：Istio 目前还处于 Alpha 阶段，不建议在生产环境中使用。

## Istio 原理

Istio 从逻辑上可以分为数据平面和控制平面：

- 数据平面主要由一系列的智能代理（默认为 Envoy）组成，管理微服务之间的网络通信
- 控制平面负责管理和配置这些智能代理，并动态执行策略

Istio 架构可以如下图所示

![](images/istio.png)

主要由以下组件构成

- [Envoy](https://lyft.github.io/envoy/)：Lyft 开源的高性能代理总线，支持动态服务发现、负载均衡、TLS 终止、HTTP/2 和 gPRC 代理、健康检查、性能测量等功能。Envoy 以 sidecar 的方式部署在相关的服务的 Pod 中。
- Mixer：负责访问控制、执行策略并从 Envoy 代理中收集遥测数据。Mixer 支持灵活的插件模型，方便扩展（支持 GCP、AWS、Prometheus、Heapster 等多种后端）
- Istio-Auth：提供服务间和终端用户的双向 TLS 认证
- Pilot：动态管理 Envoy 实例的生命周期，提供服务发现、流量管理、智能路由以及超时、熔断等弹性控制的功能。其与 Envoy 的关系如下图所示

![](images/istio-service.png)

在数据平面上，除了 [Envoy](https://www.envoyproxy.io)，还可以选择使用 [nginxmesh](https://github.com/nginmesh/nginmesh) 和 [linkerd](https://linkerd.io/getting-started) 作为网络代理。比如，使用 nginxmesh 时，Istio 的控制平面（Pilot、Mixer、Auth）保持不变，但用 Nginx Sidecar 取代 Envoy：

![](images/nginx_sidecar.png)

## 安装

Istio 的安装部署步骤见 [这里](istio-deploy.md)。

## 示例应用

> 以下步骤假设命令行终端在[安装部署](istio-deploy.md) 时下载的 `istio-${ISTIO_VERSION}` 目录中。

### 手动注入 sidecar 容器

在部署应用时，可以通过 `istioctl kube-inject` 给 Pod 手动插入 Envoy sidecar 容器，即

```sh
$  kubectl apply -f <(istioctl kube-inject --debug -f samples/bookinfo/kube/bookinfo.yaml)
service "details" configured
deployment.extensions "details-v1" configured
service "ratings" configured
deployment.extensions "ratings-v1" configured
service "reviews" configured
deployment.extensions "reviews-v1" configured
deployment.extensions "reviews-v2" configured
deployment.extensions "reviews-v3" configured
service "productpage" configured
deployment.extensions "productpage-v1" configured
ingress.extensions "gateway" configured
```

原始应用如下图所示

![](images/bookinfo.png)

`istioctl kube-inject` 在原始应用的每个 Pod 中插入了一个 Envoy 容器

![](images/bookinfo2.png)

服务启动后，可以通过 Ingress 地址 `http://<ingress-address>/productpage` 来访问 BookInfo 应用：

```sh
$ kubectl describe ingress
Name:			gateway
Namespace:		default
Address:		192.168.0.77
Default backend:	default-http-backend:80 (10.8.0.4:8080)
Rules:
  Host	Path	Backends
  ----	----	--------
  *
    	/productpage 	productpage:9080 (<none>)
    	/login 		productpage:9080 (<none>)
    	/logout 	productpage:9080 (<none>)
Annotations:
Events:	<none>
```

![](images/productpage.png)

默认情况下，三个版本的 reviews 服务以负载均衡的方式轮询。

### 自动注入 sidecar 容器

首先确认 `admissionregistration` API 已经开启：

```sh
$ kubectl api-versions | grep admissionregistration
admissionregistration.k8s.io/v1beta1
```

然后确认 istio-sidecar-injector 正常运行

```sh
# Conform istio-sidecar-injector is working
$ kubectl -n istio-system get deploy istio-sidecar-injector
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
istio-sidecar-injector   1         1         1            1           4m
```

为需要自动注入 sidecar 的 namespace 加上标签 `istio-injection=enabled`：

```sh
# default namespace 没有 istio-injection 标签
$ kubectl get namespace -L istio-injection
NAME           STATUS        AGE       ISTIO-INJECTION
default        Active        1h
istio-system   Active        1h
kube-public    Active        1h
kube-system    Active        1h

# 打上 istio-injection=enabled 标签
$ kubectl label namespace default istio-injection=enabled
```

这样，在 default namespace 中创建 Pod 后自动添加 istio sidecar 容器。

## 参考文档

- <https://istio.io/>
- [Istio - A modern service mesh](https://istio.io/talks/istio_talk_gluecon_2017.pdf)
- <https://lyft.github.io/envoy/>
- <https://github.com/nginmesh/nginmesh>
- [WHAT’S A SERVICE MESH? AND WHY DO I NEED ONE?](https://buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one/)
- [A SERVICE MESH FOR KUBERNETES](https://buoyant.io/2016/10/04/a-service-mesh-for-kubernetes-part-i-top-line-service-metrics/)
- [Service Mesh Pattern](http://philcalcado.com/2017/08/03/pattern_service_mesh.html)
- [Request Routing and Policy Management with the Istio Service Mesh](http://blog.kubernetes.io/2017/10/request-routing-and-policy-management.html)
