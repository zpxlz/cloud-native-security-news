---

tags: Cloud Native Security News, unikernel
version: v0.1.1
changelog:
  - v0.1.1: update filename, metdata

---

# 了解 unikernel
Unikernel 是一种为云化而设计的轻量级操作系统构建方式，具有以下特性：

1. 专用单应用系统：Unikernel 仅运行单一应用，相比虚拟机和容器能减少不必要的功能和代码，提高安全性​​。例如，只包含网络堆栈的 Unikernel DNS 服务，并去掉了文件系统、多媒体处理、其他网络服务等不必要的系统服务和驱动程序，这些通常存在于完整的操作系统中，就算存在RCE，也没法执行命令和读取文件
2. 单地址空间模型：Unikernel 将应用和操作系统功能集成在同一地址空间中，与传统多地址空间操作系统不同，减少了系统调用，提高性能​​。
3. 库操作系统：Unikernel 通过库操作系统与应用代码直接编译，减少了额外的操作系统代码，与基于模块的微内核和包含多功能的单体内核相比，降低了复杂性和潜在的安全风险​​。
4. 减少攻击面：由于结构简化，Unikernel 的攻击面比传统虚拟机和容器小，减少了安全漏洞的可能性​​。
5. 不变性：Unikernel 遵循不变基础设施的原则，更新和更改通过部署新版本实现，而不是修改运行中的实例，这减少了运行时被攻击的风险​​。

安全性对比
+ 优点：与虚拟机和容器相比，Unikernel 提供了更好的隔离和安全性。虚拟机虽然提供完全隔离，但由于其复杂性和大量操作系统代码，可能存在更多安全漏洞。容器分享宿主的内核，可能导致宿主操作系统的漏洞影响到所有容器。Unikernel 通过裁减内核功能，降低了这些风险​​。
+ 缺点：单地址空间模型导致应用是直接运行在 kernel ring0 空间，如果存在二进制漏洞，直接影响到内核态，并且过度裁剪会导致Linux内核原有的安全保护机制缺失，使得漏洞更容易利用。



| 技术           | 优点                                               | 缺点                                                   |
| -------------- | -------------------------------------------------- | ------------------------------------------------------ |
| 虚拟机         | - 可在单个主机上部署不同操作系统<br>- 与主机完全隔离<br>- 有编排解决方案 | - 需要与实例数量成比例的计算能力<br>- 需要大型基础设施<br>- 每个实例都加载完整的操作系统 |
| Linux 容器     | - 轻量级虚拟化<br>- 快速启动<br>- 有编排解决方案<br>- 动态资源分配 | - 由于共享内核，主机和客户端之间的隔离性降低<br>- 灵活性较差（如：依赖于主机内核）<br>- 网络灵活性较低 |
| Unikernels     | - 轻量级镜像<br>- 专业化应用<br>- 与主机完全隔离<br>- 裁减功能提高安全性（如：远程命令执行） | - 尚未足够成熟以用于生产环境<br>- 需要从内核开始开发应用程序<br>- 部署可能性有限<br>- 缺乏完整的 IDE 支持<br>- 静态资源分配<br>- 缺乏编排工具 |



个人认为 unikernel 不像容器这么流行有一部分原因是它的开发成本和门槛比较高，尽管开源社区有很多实现帮助开发者降低成本，如 [InlcudeOS](https://github.com/includeos/IncludeOS) 和 [solo5](https://github.com/Solo5/solo5) ，但也需要开发者深入了解操作系统内核和应用程序之间的交互、根据特定应用程序的需要进行高度定制化开发（可复用也不好）

总的来说，unikernel 代表了一种紧凑、高效、安全的运行应用程序的方法。

## 参考
[1] https://github.com/cetic/unikernels

[2] https://www.jianshu.com/p/aa42225da211

[3] https://zhuanlan.zhihu.com/p/622751025

[4] https://blog.51cto.com/u_15127630/2801715

----

本文发布已获得"云原生安全资讯"项目授权, 同步发布于以下平台

* github: [https://github.com/cloud-native-security-news/cloud-native-security-news](https://github.com/cloud-native-security-news/cloud-native-security-news)

欢迎加入 "云原生安全资讯"项目 👏 阅读、学习和总结云原生安全相关资讯
