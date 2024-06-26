---

tags: Cloud Native Security News, k8s
version: v0.1.2
changelog:
  - v0.1.2: update filename, metadata
  - v0.1.1: add publish suffix

---

# 云原生安全资讯: Wiz: K8s 安全态势分析

* 原文地址: https://www.wiz.io/blog/key-takeaways-from-the-wiz-2023-kubernetes-security-report

Wiz 威胁研究团队分析了 200,000 个云帐号，得出了 Kubernetes 安全态势。本文作了一些翻译和总结。

安全研究者可以参考相关数据来指导研究方向。

## 1. Kubernetes 攻击链

Wiz 将典型的k8s集群攻击链归纳为以下环节，报告分析的数据也只分析相关环节。

![image](https://github.com/cloud-native-security-news/cloud-native-security-news/assets/16935049/eca034fc-4820-405f-a622-b162a1de95f4)

##  2. 版本分布

**自管理 vs 云厂商托管**

接近 85% 的集群由云厂商托管。

**云厂商**

云厂商托管的集群中，AWS的EKS最受欢迎，占比接近一半，AKS, GKE 大致相当。

**版本**

k8s每年大致发布并维护3个主要版本，云厂商稍晚会跟进。其中 GKE 跟进最快, EKS和AKS分别约有14%,26% 的EOL版本。AKS允许用户运行很旧的版本，最老的版本是 1.7(2018.4 EOL)。

## 3. 初始访问

数据显示，控制面的不安全配置导致初始访问的情况相对少，攻击者倾向于使用控制面漏洞，可能是因为容易利用。Wiz预计，随着控制面的的默认安全的设置、意识的提高、控制平面误配置的减少，攻击者将转向攻击面更大的数据面。

### 3.1 控制面

**公开集群 vs 私有集群**

公开集群的占比为 69%。公开集群中，EKS, GKE 默认开启了匿名认证, AKS未默认开启(默认情况下匿名用户被绑定至 system:viewer 角色，该角色只允许访问关于集群的基本信息，如显示集群版本。)

少于 1% 的集群开启了匿名认证的同时，system:anonymous 用户还被绑定到了非常规权限。

### 3.2 数据面

在所有暴露的 pods 中，52% 包含有已知漏洞的容器镜像，44% 包含有高危漏洞的镜像。暴露容器中的漏洞应该比数据平面其余部分的漏洞更优先处理。

0.27% 的集群包含具有暴露 SSH 访问的容器。直接进入容器的管理接口可以让攻击者轻易绕过传统的 Kubernetes 级别的安全控制。

## 4. 横向移动和权限提升

数据显示攻击者有很多机会可以在集群内横向移动。

### 4.1 RBAC 

有8%的 pods 拥有超出了操作需求的 RBAC 权限，非 kube-system pods 中，有5.9%的 pods 拥有过高的 RBAC 权限。

有0.1%的 pods 使用带有明文云凭证的镜像。

### 4.2 容器逃逸

**敏感主机路径挂载**

18% 的 pod 挂载了主机的敏感路径。

**特权容器**

16% 的 pod可能因为capability 导致逃逸： 其中 10% 的 pod 以特权模式运行，6% 的 pod 拥有特定的危险capability。

**root**

约 18% 的 pod 以root权限运行。

上述提到的错误配置的pod 总计占比为 22% 。

### 4.3 集群内隔离

**网络隔离**

Kubernetes Network Policy 在策略级别定义了集群内的网络限制，而只有 9% 的集群使用带有网络策略的 namespace 。

**namespace**

约 7.5% 的集群，数据面中只有一个 namespace (即 default)

## 5. 影响

### 5.1 拒绝服务

19% 的 pod 没有设置资源配额限制。

### 5.2 由集群跳转到云

有15%的 pod 拥有可以跳出集群的权限，其中 60%  是通过工作节点身份获取的。

## 6. 安全控制和缓解措施

**PSP,PSS**

只有 6% 的集群没有使用 PSP、外部准入控制器，或者至少有一个命名空间没有强制执行 PSS。然而，运行 1.25 及以上版本的集群（在这些版本中，PodSecurityPolicy（PSP）已经不再可用），只有 39% 的集群在所有数据平面命名空间中使用了第三方准入控制器或内置的 Pod Security 准入控制器。（值得注意的是，即使客户从 v1.24 版本迁移到 v1.25 版本会“失去” PSP，EKS也允许迁移。这可以部分解释这种现象。）

**级别**

* 用户更倾向于 Baseline 级别的规则，使用 Audit/Warning 和 Enforce 模式的比例相当。
* Restricted级别上, 用户大部分使用Audit/Warning 模式，而不愿意使用 Enforce 模式。仅有 0.13% 的命名空间执行 Enforce 模式，与同级别的审计/警告模式相比。这表明用户通常认为限制模式对于生产工作负载的使用并不实际。

**User Namespace**

在实际使用中，没有任何 pod 实例使用这个特性。

## 7. 结论

根据分析数据，Wiz得出以下结论：

* 初始访问：k8s 控制面配置错误或漏洞相对较少，数据面漏洞更多
* 横向移动和权限提升：一旦攻击者通过了初始访问，他们就有大量的机会在集群内进行横向移动和权限提升
* 影响：在影响环节，特别是关于从k8s影响到云，安全实践不足
* 现有安全控制措施在各个攻击阶段的利用不足，社区应优先考虑采用已有安全功能，而不是引入新的安全功能。

----

本文发布已获得"云原生安全资讯"项目授权, 同步发布于以下平台

* github: [https://github.com/cloud-native-security-news/cloud-native-security-news](https://github.com/cloud-native-security-news/cloud-native-security-news)
* 知乎专栏: [云原生安全资讯](https://www.zhihu.com/column/c_1694733563684151296)
* 公众号: 石头的安全料理屋

欢迎加入 "云原生安全资讯"项目 👏 阅读、学习和总结云原生安全相关资讯
