---

tags: Cloud Native Security News,云漏洞案例,Azure
author: tarimoe
spec_version: v0.1.2
version: v0.1.1
changelog:
  - v0.1.1: update filename, metadata

---

# Azure容器实例首次跨租户容器接管漏洞(下)

> 原文地址: https://unit42.paloaltonetworks.com/azure-container-instances/

[Azure容器实例中首次跨租户容器接管漏洞(上)](./2023-12-12_Azure容器实例首次跨租户容器接管漏洞(上).md) 文中提及 CVE-2018-1002102 可能在其他场景下被利用，

## 2.1 另一条拿下K8S集群路径 —— Bridge SSRF

在Unit 42报告了（上）篇的问题之后，Microsoft 减少了 k8s 上运行的 ACI 容器的比例；只有约 10% 的区域默认使用 k8s 集群。然而，一些功能只在 k8s 上支持，例如 [gitRepo 卷](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-volume-gitrepo)。如果 ACI 容器使用了这些功能，它会被部署在 k8s 集群上。其他特性也意味着容器可能被部署到 k8s。私有[虚拟网络](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-vnet)中的容器就是这种情况。

Unit 42发现的第二个问题是 bridge pod 中的服务器端请求伪造（SSRF）漏洞。

当 bridge pod 处理一个 `az container exec <ctr> <cmd>` 命令时，它会向相应的 Kubelet 的 `/exec` 接口发送请求。bridge 根据 [Kubelet 的 /exec 接口的 API 规范构建请求](https://github.com/kubernetes/kubernetes/blob/8088b3e67d3f917a94b4ac530579c22cd7688fe6/pkg/kubelet/server/server.go#L421)，生成以下 URL：

```
https://<nodeIP>:10250/exec/<customer-namespace>/<customer-pod>/<customer-ctr>?command=<url-encoded-cmd>&error=1&input=1&output=1&tty=1
```

bridge 需要以某种方式填充尖括号 `<>` 中缺失的参数，其中 `<nodeIP>` 的值是从客户 pod 的 `status.hostIP` 字段中获取的。当节点被授权更新其 pod 的状态（例如，更新其 pod 的 `status.state` 字段为 `Running`、`Terminated` 等）时就比较有用了。

尝试使用受控节点的token 更改我们 pod 的 `status.hostIP` 字段，hostIP 字段更新了，但在一两秒后，api-server 将 `hostIP` 字段更正为原始值。虽然更改没有持久化，但并没有什么能阻止我们反复更新这个字段。

Unit 42 编写了一个小脚本，不断更新 pod 的状态，并将 `status.hostIP` 字段设置为 `1.1.1.1`，然后执行了一个 `az container exec` 命令。虽然失败了，但证命实了 bridge 将 `exec` 请求发送到了 `1.1.1.1` 而不是真实的节点 IP。接下来就是考虑如何利用特殊构造的 `hostIP` 欺骗 bridge 执行对其他 pod 的命令。

仅将 pod 的 `status.hostIP` 设置为另一个节点的 IP 不起作用，因为 Kubelets 只接受指向它们托管的容器的请求。即便 bridge 将 `exec` 请求发送到另一个 Kubelet 的 IP，URL 仍然会指向我们的命名空间、pod 名称和容器名称。

Unit 42随后发现到 api-server 实际上并不验证 `status.hostIP` 的值是否为有效的 IP，它会接受任何字符串 —— 包括 URL 部分。经过几次尝试，Unit 42 找到了一个 `hostIP` 值，可以让 bridge 执行对 api-server 容器的命令，而非我们自己的容器：

```
<apiserver-nodeIP>:10250/exec/kube-system/<apiserver-pod>/<apiserver-container>?command=<url-encoded-command>&error=1&input=1&output=1&tty=1#
```

`hostIP` 值会让 bridge 将 `exec` 请求发送到以下 URL：

`https://**<apiserver-nodeIP>:10250/exec/kube-system/<apiserver-pod-name>/<apiserver-ctr>?command=<url-encoded-command>&error=1&input=1&output=1&tty=1**<u>#</u>:10250/exec/<customer-namespace>/<customer-pod-name>/<customer-ctr-name>?command=<command>&error=1&input=1&output=1&tty=1`

使用 # 后缀确保 URL 的其余部分被视为[URI fragment](https://en.wikipedia.org/wiki/URI_fragment)，从而忽略后面的异常部分。我们将我们的 pod 的 `status.hostIP` 设置为这个值，并通过 `az container exec` 执行命令，利用成功并获得 api-server 容器的 shell。

![patch_hostIP](./image/2023-12-14_Azure容器实例中首次跨租户容器接管漏洞(下)/Figure16.png)

![get_apiserver_shell](./image/2023-12-14_Azure容器实例中首次跨租户容器接管漏洞(下)/Figure17.png)

链接为视频演示：
https://youtu.be/7Alea_9oZgU

这次攻击的影响与之前的攻击完全相同——对多租户集群拥有完全的管理控制权。Unit 42也向 MSRC 报告了这个问题，随后 ACI 出了补丁：bridge 在发送 exec 请求之前会验证 pod 的 status.hostIP 字段是否为有效的 IP 地址

## 2.2 总结
跨账户漏洞通常被认为是公有云面临的“噩梦”级威胁。Azurescape 的发现证实，这些威胁比我们所认为的更加真实。云服务提供商在加强平台安全方面投入了巨大的努力，但不可避免地，存在未知的 0Day 漏洞，这可能会使客户面临风险。因此，云用户应采取多层防御策略，确保无论威胁来自外部还是来自云平台本身，都能有效控制和检测到安全漏洞。

作为 Palo Alto Networks 对推动公有云安全的承诺的一部分，Unit 42积极进行公有云研究，包括高级威胁建模和对云平台及相关技术的漏洞测试，希望这种研究能展示跨账户攻击在现实环境中可能的形式，从而转化为适当的缓解措施和检测机制。

Unit 42也要感谢微软安全响应中心（MSRC）迅速修复我们报告的问题，并专业地处理了整个披露过程，同时感谢他们提供的奖金奖励。合作式的渗透测试和漏洞赏金计划对于保护我们所有人依赖的云服务起着重要作用。

## 2.3 防范 k8s 环境中的类似攻击
对于 k8s 的安全建设来说，以下这些最佳实践、缓解措施和政策非常关键，它们有助于预防或识别类似攻击的迹象：

1. 保持集群基础设施的及时更新，根据安全漏洞的严重程度和具体环境来优先打补丁。
2. 避免向除了 apiserver 之外的任何人发送具有特权的服务账户令牌。如果接收者被入侵，攻击者可以伪装成令牌的拥有者。
3. 启用 [BoundServiceAccountTokenVolume](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#bound-service-account-token-volume) 这个新特性。这个特性确保令牌的有效期与其所属的 Pod 绑定，当 Pod 结束时，其令牌同时失效，这样就能最大限度地减少令牌被盗的影响。
4. 部署策略执行器来监控集群中的可疑活动，并阻止它们。设置这些执行器，使其能够警告那些查询 SelfSubjectAccessReview 或 SelfSubjectRulesReview API 来获取权限的服务账户或节点。Prisma Cloud 的用户可以下载一个相关的规则模板，并通过 k8s 的内置准入控制来实施它。建议将规则设置为警告模式。其他用户则可以使用开源工具，比如 [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper)。
5. 一般攻击者都会利用 SelfSubjectReview API 来查看他们获取到的 k8s 凭证的权限。研究员 Daniel Prizmant 最近发现了 Siloscape 恶意软件，它利用这些 API 来查询被攻击的节点的权限，然后决定是否继续对集群进行攻击。Unit 42 已经将这一行为报告给了 MITRE，并且这将在 [ATT&CK for Containers](https://attack.mitre.org/matrices/enterprise/containers/) 的下一个版本中作为 [Permission Group Discovery](https://attack.mitre.org/techniques/T1069) 技术被纳入。

在 k8s 中实现安全多租户是一项挑战，即使对云服务提供商而言也是如此。那些在 k8s 上实现多租户的托管服务、云服务提供商和 CI/CD 服务在设计其平台时，应考虑以下几点：

- 不要在租户边界内暴露集群凭证。恶意租户可能会滥用这些凭证来进一步扩大攻击范围或收集其他租户以及平台本身的信息。
- 要假设恶意租户能够从其容器或沙箱中逃逸。站在攻击者的角度思考 - 他们的目标是什么？他们的首要行动是什么？据此部署相应的检测机制。在多租户环境中，这样的检测机制是对抗高级持续威胁（APT）和 0 Day 的必要条件。
- 即便一个节点遭到攻击，也不应该允许它横向移动并入侵其他节点。要确保节点具有最少的必要权限，可以通过启用 NodeRestiction 准入控制器来实现。还需要设置防火墙规则，以阻止托管客户容器的节点之间的通信。

## 2.4 个人总结
本SSRF漏洞攻击路径总结
1. bridge**Pod的SSRF漏洞**：当Bridge Pod执行命令 **`az container exec <ctr> <cmd>`** 时，会向相应 Kubelet 的 **`/exec`** 端点发起请求。这个请求的 URL 是由多个参数拼接而成的，其中 **`<nodeIP>`** 的值来自客户 Pod 的 **`status.hostIP`** 字段。
2. **篡改Pod的`status.hostIP`字段**：尝试利用受控Node节点的凭证来修改其 Pod 的 **`status.hostIP`** 字段。尽管 apiserver 服务器会迅速纠正该字段的值，但Unit 42发现可以不断地对其进行更新。
3. **构造异常的`hostIP`值**：尽管节点会对 **`status.hostIP`** 进行有效性校验，但 apiserver 并不会验证该字段是否为有效的 IP 地址，而是接受任何字符串，包括 URL 组件。因此，我们可以构造了一个特殊的 **`hostIP`** 值，使得 bridge Pod 的执行请求被发送到 apiserver 容器，而非自己的容器。
4. **成功实施攻击**：将我们的 Pod 的 **`status.hostIP`** 设置为构造好的值，并通过 **`az container exec`** 发起命令。利用成功后即获得了对 apiserver 容器的 shell 访问权限。

通过 Azurescape 学习到的一些点：
1. whoC在黑盒场景下利用作收集runC版本信息
2. CVE-2018-1002102利用思路，虽然最终没有利用成功，有空可以看看如果能利用要怎么搞（挖坑）
3. 抓取 apiserver 请求 node kubelet 的包，之前一直想搞，但没想到很好的方式，因为apiserver和其他节点的通信基于具有DH（前向安全）特性的HTTPS（普通的RSA不一样）
4. 任意URL跳转漏洞在云场景下的危害升级
5. 针对此类漏洞的缓存措施


----

本文发布已获得"云原生安全资讯"项目授权, 同步发布于以下平台

* github: [https://github.com/cloud-native-security-news/cloud-native-security-news](https://github.com/cloud-native-security-news/cloud-native-security-news)
* blog: [tari Blog](https://tari.moe)

欢迎加入 "云原生安全资讯"项目 👏 阅读、学习和总结云原生安全相关资讯
