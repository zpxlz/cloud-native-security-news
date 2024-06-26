---

tags: 云原生安全资讯
version: v0.1.1
changelog:
  - v0.1.1: update filename, metadata
---

文章转载自：[https://chenxy.me/a-journey-to-runc](https://chenxy.me/a-journey-to-runc) （已获作者授权）

# 云原生安全资讯: runc 主流程简读

平时天天跟容器打交道，但自己对其背后的机制却并不是多么了解。今天我们先跳过上层让人眼花缭乱的 [containerd](https://containerd.io/)、[CRI-O](https://cri-o.io/) 以及各种 xxx-shim，直接进入到最深处的组件：[runc](https://github.com/opencontainers/runc)。所谓“容器的三大核心技术”——namespace、cgroup、unionfs，runc 就负责着前两者。


本文主要关注 runc 里一些关键函数的流程，因此会跳过大量细节，我只能尽量保持不丢失最重要的几个部分~

## 前置背景

在进入 runc 代码之前，我们先了解下 OCI Specs。OCI 标准全称是 [Open Container Initiative 开放容器倡议](https://opencontainers.org/)，包含了三个子规范：Image Specs、Distribution Specs、Runtime Specs。

* [Image Specs](https://github.com/opencontainers/image-spec) 规定了一个 OCI 镜像应该遵守的格式；
* [Distribution Specs](https://github.com/opencontainers/distribution-spec) 规定了一个镜像分发系统应该提供的 API 接口（基本与 Docker Registry 一致）；
* [Runtime Specs](https://github.com/opencontainers/runtime-spec) 规定了一个容器的配置格式、执行环境与生命周期；



runc 实现了 OCI Runtime Specs，与另外两个规范关系不大。简要来说，OCI Runtime Specs 主要定义了这些东西：

* 一个符合 OCI Runtime Specs 的配置格式需要包含一个 config.json 文件，并定义了其中的结构（包含了 rootfs、cmd、env、sysctl、cap、cgroup、hooks 等各种配置）；
* 一个容器应当拥有几种状态，以及各个状态的定义；
* 一个容器应当支持的几种操作（创建、开启、关闭等）；



下面这段代码展示了我使用 docker 启动一个容器后，其生成的典型的 config.json 结构： [https://gist.github.com/oyiadin/f10a08c4b8c134516ecedf160a0241f0](https://gist.github.com/oyiadin/f10a08c4b8c134516ecedf160a0241f0)

## 环境准备

在开始阅读代码之前，先自己编译一个 runc 并尝试运行起来一个最简单的容器，以获得一点感性认知吧。


> 本文章使用编写时最新的 runc main 分支的版本，准确来说，commit id 为 [ac78d13271f777ced2855fa6f4552c8ba7a4ed15](https://github.com/opencontainers/runc/commit/ac78d13271f777ced2855fa6f4552c8ba7a4ed15)。



要编译+调试 runc，除了源代码之外、常规的编译工具之外，还需要安装 `libseccomp-devel`，当然还有 Go、delve。准备好这些东西之外，直接 `make` 一把梭，稍等半分钟就编译好了。不过为了调试方便，还需要在 `Makefile` 里给 `GO_BUILD` 加上 `-gcflags "all=-N -l"` 的参数，然后重新 `make` 即可。



完成之后即可使用 `dlv` 进行调试。不过由于 runc 的特殊性，过程中会创建好几个子进程，其中还有用 C 语言写成的。`dlv` 目前貌似不支持自动 follow 子进程，因此这些子进程需要自行用 `dlv attach` 或者 `gdb attach` 上去。



做好这些准备之后，runc 的运行需要一个 filesystem bundle。一个最小的 bundle 只需要 `config.json` 与 `rootfs` 即可。先创建一个新目录用以放置我们的 bundle，然后使用 `runc spec` 命令即可生成一个默认的 `config.json` 文件，无须修改其中的任何配置即可直接使用。


而 `rootfs` 的选择就比较多了。如果你有安装 docker，使用以下命令就可以导出一个 alpine 环境作为 `rootfs`：

```bash
mkdir rootfs
docker export `docker run -d alpine:latest` | tar -x -C rootfs
```



如果没有 docker，也可以选择直接从 [alpine 官网](https://alpinelinux.org/downloads/#:~:text=MINI%20ROOT%20FILESYSTEM)下载并解压：

```bash
mkdir rootfs
url="https://dl-cdn.alpinelinux.org/alpine/v3.18/releases/$(uname -p)/alpine-minirootfs-3.18.4-$(uname -p).tar.gz"
wget -qO- $url | tar -C rootfs -xz
```



随后就可以使用 runc 运行一个容器了，命令如下：

```bash
hsiaoxychen@hsiaoxychen-PC:~/runc-bundle$ ll
total 16
drwxr-xr-x  3 hsiaoxychen hsiaoxychen 4096 Oct 14 22:34 ./
drwxr-x--- 13 hsiaoxychen hsiaoxychen 4096 Oct 14 15:48 ../
-rw-r--r--  1 hsiaoxychen hsiaoxychen 2560 Oct 14 11:31 config.json
drwxr-xr-x 19 hsiaoxychen hsiaoxychen 4096 Sep 28 19:18 rootfs/

hsiaoxychen@hsiaoxychen-PC:~/runc-bundle$ sudo ../runc/runc run test-container-id
/ # hostname
runc
/ #
hsiaoxychen@hsiaoxychen-PC:~/runc-bundle$
```

如果要调试，使用 `dlv exec` 即可。另外建议打开 runc 的 `--debug` 选项，可以输出详尽的日志，方便调试。

## libcontainer 外的部分流程

```plain
             +-----------+
             | main.main |
             +-----+-----+
                   |
       +-----------v------------+
       |  run.runCommand.Action |
       +-----------+------------+
                   |
     +-------------v--------------+
     | utils_linux.startContainer |
     +-------------+--------------+
                   |
     +-------------v---------------+
     | utils_linux.createContainer |
     +-------------+---------------+
                   |
+------------------v-------------------+
| utils_linux.runner.run(spec.Process) |
+--------------------------------------+
```

流程图如上。其中 `createContainer` 这个函数先构造了一个 `libcontainer.Config` 结构体，随后就调用了 `libcontainer.Create`。后者也是做了一些简单的处理后返回了一个 `libcontainer.Container` 结构体。


可以看到在 `runc` 的外层模块里主要用来做一些参数转换，输入、输出的处理等，核心的逻辑都在 `libcontainer` 包下边，这种结构可以让其他模块直接调用 `libcontainer` 包做其他的功能扩展，非常灵活。


`runner.run` 函数的流程如下：

```plain
   +------------------------+
   | utils_linux.newProcess |
   +-----------+------------+
               |
+--------------v---------------+
| utils_linux.newSignalHandler |
+--------------+---------------+
               |
    +----------v----------+
    | utils_linux.setupIO |
    +----------+----------+
               |
 +-------------v--------------+
 | libcontainer.Container.Run |
 +-------------+--------------+
               |
      +--------v---------+
      | handler.forward  |
      +------------------+
```



`libcontainer.Container.Run` 里边是创建、运行容器的核心逻辑，它执行完成之后容器就已经运行起来了，在等待着我们把命令通过 `tty` 传递给它，下一节会继续分析。现在先看一眼 `handler.forward`：`handle.forward` 里边只是一个循环，用以接收信号，并在完成一些必要的处理后传递给子进程（容器进程）。不过这里对几个特殊信号的处理倒是可以说道说道：

* `SIGWINCH` 信号的产生代表了窗口大小发生改变，无需传递到容器内部，直接对 `tty` 进行 `resize` 就可以；

```go
	for s := range h.signals {
		switch s {
		case unix.SIGWINCH:
			// Ignore errors resizing, as above.
			_ = tty.resize()
```

* `SIGURG` 信号被 Go runtime 用来实现抢占式的调度，这会导致 runc 时不时就会收到这个信号，忽略即可，无需传递给子进程；

```go
		case unix.SIGURG:
			// SIGURG is used by go runtime for async preemptive
			// scheduling, so runc receives it from time to time,
			// and it should not be forwarded to the container.
			// Do nothing.
```

* `SIGCHLD` 信号在子进程退出时由操作系统触发。父进程需要调用 `wait` 系列的 syscall 以 `reap` 子进程，否则子进程会成为一个 `defunct 进程`，也就是我们常说的僵尸进程。这个信号也无需传递给子进程（也不应该传递）；

```go
		case unix.SIGCHLD:
			exits, err := h.reap()
			if err != nil {
				logrus.Error(err)
			}
			for _, e := range exits {
				logrus.WithFields(logrus.Fields{
					"pid":    e.pid,
					"status": e.status,
				}).Debug("process exited")
				if e.pid == pid1 {
					// call Wait() on the process even though we already have the exit
					// status because we must ensure that any of the go specific process
					// fun such as flushing pipes are complete before we return.
					_, _ = process.Wait()
					return e.status, nil
				}
			}
```

```go
// reap runs wait4 in a loop until we have finished processing any existing exits
// then returns all exits to the main event loop for further processing.
func (h *signalHandler) reap() (exits []exit, err error) {
	var (
		ws  unix.WaitStatus
		rus unix.Rusage
	)
	for {
		pid, err := unix.Wait4(-1, &ws, unix.WNOHANG, &rus)
		if err != nil {
			if err == unix.ECHILD {
				return exits, nil
			}
			return nil, err
		}
		if pid <= 0 {
			return exits, nil
		}
		exits = append(exits, exit{
			pid:    pid,
			status: utils.ExitStatus(ws),
		})
	}
}
```

* 其他信号，则统一使用 `kill` syscall 发送给子进程即可。

```go
		default:
			us := s.(unix.Signal)
			logrus.Debugf("forwarding signal %d (%s) to %d", int(us), unix.SignalName(us), pid1)
			if err := unix.Kill(pid1, us); err != nil {
				logrus.Error(err)
			}
```

## 准备发起 runc init 子进程

我们使用的 `runc run` 命令其实是 `create + start` 的集合，其中 `create` 对应着 `libcontainer.Container.Start`，`start` 则对应着 `libcontainer.Container.exec`。

```go
	switch r.action {
	case CT_ACT_CREATE:
		err = r.container.Start(process)
	case CT_ACT_RESTORE:
		err = r.container.Restore(process, r.criuOpts)
	case CT_ACT_RUN:
		err = r.container.Run(process)
	default:
		panic("Unknown action")
	}
```

```go
func (c *Container) Run(process *Process) error {
	// Run 函数的背后，只是把两个流程拼接了起来：
	// 先调用了一下 Start（对应命令 runc create），然后调用 exec 触发容器执行
	if err := c.Start(process); err != nil {
		return err
	}
	if process.Init {
		return c.exec()
	}
	return nil
}
```

`libcontainer.Container.Start` 负责完成整个容器的准备过程，并向一个名为 `exec.fifo` 的文件写入数据（被阻塞），等待 runc 主进程从中读取数据，这将作为容器正式启动的信号。`libcontainer.Container.exec` 的逻辑非常简单，它正是 `exec.fifo` 的接收者，负责打开 fifo 文件以使得（正在等待 fifo 文件的）容器进程继续执行。


`libcontainer.Container.Start` 先创建了上文所说的 `exec.fifo`，然后进入了 `libcontainer.Container.start`。后者先调用 `newParentProcess` 创建了一个 `libcontainer.initProcess` 结构体，随后调用了这个结构体示例的 `start` 函数。


> 执行完 `initProcess.start` 之后，就是 `postStart` 的 hook 点。



### newParentProcess

这个函数先是借助 `newProcessComm` 函数创建了一个 `libcontainer.processComm` 结构体以及初始化 `init`、`sync` 与 `log` 共三对 `socket pair`，用以后续跟子进程同步数据使用。


随后构造了一个 `exec.Cmd` 对象，命令为 `/proc/self/exe init`，并且工作目录在 rootfs 下。


构造这个命令的过程中，会将上文提及的那几对 `socket pair` 的 `fd` 号（包括 `exec.fifo`）通过环境变量的形式传递进去。子进程由于默认继承 `fd`，直接读取 `fd` 即可还原 `socket` 对象。


随后又借助 `newInitProcess` 函数，根据上文构造的各个结构图，构造一个 `libcontainer.initProcess` 对象。在这个函数中，会将一些启动所必须的信息以 `netlink` 的格式封装起来，以供后续传递给 `runc init` 子进程。

### initProcess.start

此函数一开头就先把上文那个 `/proc/self/exe init` 以子进程的形式运行了起来：

```go
	defer p.comm.closeParent()
	err := p.cmd.Start()
	p.process.ops = p
	// close the child-side of the pipes (controlled by child)
	p.comm.closeChild()
	if err != nil {
		p.process.ops = nil
		return fmt.Errorf("unable to start init: %w", err)
	}
```

随后将借助上文创建的各个 `socket pair` 与子进程进行状态同步、数据传递。由于这段逻辑与子进程联系紧密，需要在后文与 `runc init` 的代码一同分析。

## runc init 进程初印象

既然这也是一个命令，那在 `main.go` 里应该能找到对应的 `action` 吧……居然没有？！好吧，其实是放到 `init.go` 里边去了。由于 `runc init` 的定位是串起容器启动前所有与操作系统紧密相关的环境准备流程，并负责最终 `execve` 容器入口程序，runc 自身的绝大多数逻辑都是没必要执行的。也由于 `nsexec` 的特殊性，需要在绝大多数 Go runtime 逻辑之前处理，因此整个给放到了 `func init` 之下。


先看一眼代码：

```go
package main

import (
	"os"

	"github.com/opencontainers/runc/libcontainer"
	_ "github.com/opencontainers/runc/libcontainer/nsenter"
)

func init() {
	if len(os.Args) > 1 && os.Args[1] == "init" {
		// This is the golang entry point for runc init, executed
		// before main() but after libcontainer/nsenter's nsexec().
		libcontainer.Init()
	}
}
```

非常简单。先 `import nsenter`，然后判断命令参数并进入 `libcontainer.Init` 函数。但是，别被 `func init` 给骗了，其实比它更早的还有一个 `nsenter` 要执行。`nsenter` 由于其功能的特殊性质，借助 `cgo` 主要以 C 语言实现。先看看 Go 这边的入口：

```go
//go:build linux && !gccgo
// +build linux,!gccgo

package nsenter

/*
#cgo CFLAGS: -Wall
extern void nsexec();
void __attribute__((constructor)) init(void) {
	nsexec();
}
*/
import "C"
```

为了使得 `nsexec` 函数突破 Go 的机制尽早被执行，这段代码借助了 `gcc` 的 `constructor` 特性，实现了在 `go.main` 函数之前执行。`nsexec` 函数由 `C` 语言编写而成，逻辑比较复杂，请继续往下阅读：

## nsexec 阶段零：准备工作

为了使得日志能够正常传递给主进程，`nsexec` 最先开始读取并配置 `log` pipe。随后借助 `init` pipe 等待主进程准备完毕：

```c
	/*
	 * Get the init pipe fd from the environment. The init pipe is used to
	 * read the bootstrap data and tell the parent what the new pids are
	 * after the setup is done.
	 */
	pipenum = getenv_int("_LIBCONTAINER_INITPIPE");
	if (pipenum < 0) {
		/* We are not a runc init. Just return to go runtime. */
		return;
	}

	/*
	 * Inform the parent we're past initial setup.
	 * For the other side of this, see initWaiter.
	 */
	if (write(pipenum, "", 1) != 1)
		bail("could not inform the parent we are past initial setup");

	write_log(DEBUG, "=> nsexec container setup");
```

主进程未准备完毕时，子进程将阻塞在倒数第二行的 `write` 函数中。此时主进程在 `initProcess.start` 中，紧接着上文的地方：

```go
	waitInit := initWaiter(p.comm.initSockParent)
	// ... 省略一部分内容 ...
	if _, err := io.Copy(p.comm.initSockParent, p.bootstrapData); err != nil {
		return fmt.Errorf("can't copy bootstrap data to pipe: %w", err)
	}
	err = <-waitInit
	if err != nil {
		return err
	}
```

可以看到在打开 `init` pipe 实现与子进程的同步之后，主进程紧接着借助同一个管道，把 `bootstrapData` 传递给了子进程。


此时回到子进程，子进程将输出 `=> nsexec container setup` 的调试日志，此时即标志着子进程的初始化流程正式开始。


子进程（runc init）会先解析主进程发过来的 `bootstrapData` 数据并存到内存中。然后依次创建 `sync_child` 与 `sync_grandchild` 两对管道，用以后续跟 runc init 的子进程以及子进程的子进程交互。简单画个流程就是：

```plain
runc run
--> 创建了子进程 runc init (runc:[0:PARENT])
--> 创建了子进程 runc init stage-child (runc:[1:CHILD])
--> 创建了子进程 runc init stage-init (runc:[2:INIT])
```

也就是一次容器的创建+运行一共涉及了四个进程。中间的两个进程流程比较短暂，很快就退出了。如果开了 `detach` 模式，第一个主进程也会很快退出。只有最后一个 stage-init 的进程会负责 `execve` 容器真正的入口程序，并成为容器内的 `pid=1` 根进程。

## nsexec 阶段一：STAGE_PARENT

扯远了，继续回到代码。到了这个地方后，`runc init` 进程就正式进入了各个子进程的分叉点，此时我们先会进入 `STAGE_PARENT` 的分支（也即是 `0`）：

```c
	switch (setjmp(env)) {
		/*
		 * Stage 0: We're in the parent. Our job is just to create a new child
		 *          (stage 1: STAGE_CHILD) process and write its uid_map and
		 *          gid_map. That process will go on to create a new process, then
		 *          it will send us its PID which we will send to the bootstrap
		 *          process.
		 */
	case STAGE_PARENT:{
			// ...
```

`setjmp` 函数标记了一处跳转点，执行流程有点类似 `fork`。首次执行时，`setjmp` 将保存当前程序的 `PC` 寄存器等状态，记录到 `env` 参数中，然后返回 `0`，也即是 `STAGE_PARENT`，进入 `runc init` 主进程的分支。后续创建其他子进程时，会借助 `longjmp` 跳回 `setjmp` 的标记点，并返回其他取值，以进入对应的其他 switch-case 分支。


STAGE_PARENT 的逻辑比较简单，主要负责同步各个进程间的状态。先看一眼整体流程：

```plain
                             +--------------------------+
                             | spawn stage-1 subprocess |
                             +--------------------------+
                                          |
                            +-------------v--------------+
                            | read requests from stage-1 |
                            |    via sync_child pipe     |
                            +-------------+--------------+
                                          |
                           +--------------v---------------+
+--------------------------> process request from stage-1 <---------------------------+
|                          ++-------------+--------------++                           |
|                           |             |              |                            |
|  +------------------------v+   +--------v----------+  +v-----------------+          |
|  |     SYNC_USERMAP_PLS    |   | SYNC_CHILD_FINISH |  | SYNC_RECVPID_PLS |          |
|  | and other data requests |   |                   |  |                  |          |
|  +-----------+-------------+   +--------+----------+  +---------+--------+          |
|              |                          |                       |                   |
|   +----------v------------+     +-------v---------+  +----------v----------+        |
^---+ send datas to stage-1 |     | stops the loop  |  | send ack to stage-1 |        |
    +-----------------------+     +-------+---------+  +----------+----------+        |
                                          |                       |                   |
                                          |      +----------------v----------------+  |
                                          |      | forward stage-1 and stage-2 PID |  |
                                          |      |            to parent            +--^
                                          |      +---------------------------------+
                                          |
                  +-----------------------v----------------------+
                  |       send SYNC_GRANDCHILD to stage-2        |
                  | to tell it that "you can start working now!" |
                  +-----------------------+----------------------+
                                          |
                       +------------------v------------------+
                       | recv SYNC_CHILD_FINISH from stage-2 |
                       +------------------+------------------+
                                          |
                                      +---v--+
                                      | exit |
                                      +------+
```

STAGE_PARENT 在克隆出 STAGE_CHILD 之后就进入一个循环中，不断处理 STAGE_CHILD 从 `sync_child` pipe 发过来的请求。主要有三种：

* 数据类的请求，会从刚才从 netlink 格式的 `bootstrapData` 中提取出来并发给 STAGE_CHILD；
* SYNC_RECVPID_PLS，会接收从 STAGE_CHILD 发过来的 STAGE_INIT 的 PID，并随自己记录的 STAGE_CHILD 的 PID 一起发送给 runc 主进程；
* SYNC_CHILD_FINISH，表明 STAGE_CHILD 已完成所有工作，可进入 STAGE_INIT 的交互流程中。



完成上述处理后，会先发送一个通知给 STAGE_INIT，告知其可继续执行了。STAGE_INIT 的流程很简单，无须更多的交互。等其流程结束后会发送一个 SYNC_CHILD_FINISH，此时 STAGE_PARENT 就完成其所有工作了，直接一个 `exit(0)` 退出。

## nsexec 阶段二：STAGE_CHILD

nsexec 几个阶段（子进程）的执行时间是互相有重叠的，并利用 `pipe` 来同步相互之间的执行进度以及数据传递。


STAGE_CHILD 的入口在 STAGE_PARENT 的 `stage1_pid = clone_parent(&env, STAGE_CHILD);` 处，先进入 `clone_parent` 函数看一眼：

```c
struct clone_t {
	/*
	 * Reserve some space for clone() to locate arguments
	 * and retcode in this place
	 */
	char stack[4096] __attribute__((aligned(16)));
	char stack_ptr[0];

	/* There's two children. This is used to execute the different code. */
	jmp_buf *env;
	int jmpval;
};

static int clone_parent(jmp_buf *env, int jmpval)
{
	struct clone_t ca = {
		.env = env,
		.jmpval = jmpval,
	};

	return clone(child_func, ca.stack_ptr, CLONE_PARENT | SIGCHLD, &ca);
}
// clone 函数的签名：
// https://linux.die.net/man/2/clone
/*
int clone(int (*fn)(void *), void *child_stack,
          int flags, void *arg, ... );
*/

static int child_func(void *arg)
{
	struct clone_t *ca = (struct clone_t *)arg;
	longjmp(*ca->env, ca->jmpval);
}
```

`clone` 克隆出一个子进程，子进程将从 `fn` 函数处开始执行，也就是 `child_func`，做的事情很简单，就是跳转到先前 `setjmp` 的地方并指定 `setjmp` 的新的返回值。


子进程使用的栈空间由父进程为其指定，由于子进程的入口函数永远也不会 `return`，导致突破栈的边界，这里直接使用 `clone_parent` 里的局部栈空间并没有关系。


`CLONE_PARENT` 的参数使得子进程的父进程与当前进程的父进程一致，也就是 STAGE_CHILD 与 STAGE_PARENT 并无父子关系，STAGE_CHILD 直接由 runc 主进程管辖。`SIGCHLD` 参数使得子进程退出时会发送 `SIGCHLD` 给父进程。


最后一个参数携带了 `longjmp` 所需要的跳转点信息，以及跳转过去后要使用什么返回值。在此处的话就是 `clone_parent(&env, STAGE_CHILD)` 所指定的 `STAGE_CHILD`，因此子进程将进入 `switch (setjmp(env))` 的 `STAGE_CHILD` 分支：

```c
	switch (setjmp(env)) {
	case STAGE_PARENT: /* ... */

		/*
		 * Stage 1: We're in the first child process. Our job is to join any
		 *          provided namespaces in the netlink payload and unshare all of
		 *          the requested namespaces. If we've been asked to CLONE_NEWUSER,
		 *          we will ask our parent (stage 0) to set up our user mappings
		 *          for us. Then, we create a new child (stage 2: STAGE_INIT) for
		 *          PID namespace. We then send the child's PID to our parent
		 *          (stage 0).
		 */
	case STAGE_CHILD: /* ... */
```

STAGE_CHILD 主要负责将当前进程加入配置的各个 namespace 中，并离开原有的 namespace。流程比较长，核心就是 `setns` 与 `unshare` 函数，后边有机会的话再做下深入分析。


值得一提的是，按道理到这里之后，所有准备工作就已经全部完成，无需再 `clone` 子进程的。为什么 runc 还要再引入一个 STAGE_INIT 呢？注释里有针对这个问题的说明：

```c
			/*
			 * TODO: What about non-namespace clone flags that we're dropping here?
			 *
			 * We fork again because of PID namespace, setns(2) or unshare(2) don't
			 * change the PID namespace of the calling process, because doing so
			 * would change the caller's idea of its own PID (as reported by getpid()),
			 * which would break many applications and libraries, so we must fork
			 * to actually enter the new PID namespace.
			 */
			write_log(DEBUG, "spawn stage-2");
			stage2_pid = clone_parent(&env, STAGE_INIT);
			if (stage2_pid < 0)
				bail("unable to spawn stage-2");
```

如注释中所说明的，如果 PID namespace 在当前进程立即生效的话，会导致当前进程发现自己的 PID 发生了改变，这将给现有的应用程序、库的逻辑带来很严重的问题。因此 PID namespace 的设计是需要在子进程里才开始生效。如果我们在此时去查看 STAGE_CHILD 的 namespace 信息，会发现其 `pid` 与 `pid_for_children` 的 ns ID 是不一样的：

```plain
root@hsiaoxychen-PC:/home/hsiaoxychen# ll /proc/8696/ns
total 0
dr-x--x--x 2 root root 0 Oct 14 15:48 ./
dr-xr-xr-x 9 root root 0 Oct 14 15:17 ../
lrwxrwxrwx 1 root root 0 Oct 14 15:48 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Oct 14 15:48 ipc -> 'ipc:[4026532365]'
lrwxrwxrwx 1 root root 0 Oct 14 15:48 mnt -> 'mnt:[4026532362]'
lrwxrwxrwx 1 root root 0 Oct 14 15:48 net -> 'net:[4026532367]'
lrwxrwxrwx 1 root root 0 Oct 14 15:48 pid -> 'pid:[4026532366]'
lrwxrwxrwx 1 root root 0 Oct 14 15:48 pid_for_children -> 'pid:[4026532373]'
lrwxrwxrwx 1 root root 0 Oct 14 15:48 time -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Oct 14 15:48 time_for_children -> 'time:[4026531834]'
lrwxrwxrwx 1 root root 0 Oct 14 15:48 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Oct 14 15:48 uts -> 'uts:[4026532363]'
```

由于此时 STAGE_CHILD 还在 root PID namespace 的视角下，它可以正常拿到 STAGE_INIT 在 root ns 视角下的 PID。将这个 PID 传递给 STAGE_PARENT 之后，STAGE_CHILD 也完成了自己的使命，直接 `exit(0)` 退出。

## nsexec 阶段三：STAGE_INIT

同样是回到了 `setjmp` 处，这一次将进入到 STAGE_INIT 的分支：

```c
		/*
		 * Stage 2: We're the final child process, and the only process that will
		 *          actually return to the Go runtime. Our job is to just do the
		 *          final cleanup steps and then return to the Go runtime to allow
		 *          init_linux.go to run.
		 */
	case STAGE_INIT: /* ... */
```

这个子进程仅仅只是为了 PID namespace 的机制而存在，几乎没有其他需要完成的事情了，逻辑非常简单。先借助 `sync_grandchild` 等待 STAGE_PARENT 发来“开始执行”的信号后，完成一些简单的初始化、清理逻辑之后，通知 STAGE_PARENT 自己已经初始化完毕，随后借助一个 `return` 终于退出了漫长的 `nsexec` 函数。

## 别忘了 libcontainer.Init

退出 `nsexec` 函数之后……就没了？别忘了，刚才所有的 `nsexec` 逻辑，只是借助 gcc 的 constructor 机制抢在 Go runtime 之前的部分！现在 constructor 执行完了，该继续正常进入 Go 的世界了……


> 值得一提的是，在进入 Go 的逻辑之前，由于是 C 的代码，需要用 gdb attach 进行调试。进入 Go 的逻辑之后，gdb 就不太方便调试了，可以 detach 出来后再使用 dlv attach 继续调试。


```go
package main

import (
	"os"

	"github.com/opencontainers/runc/libcontainer"
	// 刚才那么多 C 的代码，只“相当于”走完了这个 import…
	_ "github.com/opencontainers/runc/libcontainer/nsenter"
)

func init() {
	if len(os.Args) > 1 && os.Args[1] == "init" {
		// This is the golang entry point for runc init, executed
		// before main() but after libcontainer/nsenter's nsexec().
		libcontainer.Init()
	}
}
```

一路跟进去到达一个 `libcontainer.startInitialization` 与 `libcontainer.containerInit` 函数。还是熟悉的流程，runc 将从环境变量里取出各个用来传递信息的 pipe fd。该拿的都拿出来后，一个 `os.Clearenv` 调用把所有 runc 内部的环境变量全部清空，随后再把 `config.Env` 中指定的环境变量依次设置上。之后来到 `libcontainer.linuxStandardInit.Init` 函数：

```go
// 省略了一些不（kan）重（bu）要（dong）的代码
func (l *linuxStandardInit) Init() error {
	// 设置网络与路由，一般只是把 loopback 给开起来
	// 时刻记得：现在已经在“容器”里边了，我们现在是容器里的 1 号进程！
	if err := setupNetwork(l.config); err != nil {
		return err
	}
	if err := setupRoute(l.config.Config); err != nil {
		return err
	}

	// We don't need the mount nor idmap fds after prepareRootfs() nor if it fails.
	// 这里边会完成 proc 等各个目录的 mount，并完成 pivot_root，把 mount 点的根目录都切换到 rootfs 下
	// 在 pivot_root 之前查看一下 mount 信息的话，长这个样子：
	/*
root@hsiaoxychen-PC:/home/hsiaoxychen# mount -N /proc/8696/ns/mnt|grep hsi
/dev/sdc on /home/hsiaoxychen/runc-bundle/rootfs type ext4 (rw,relatime,discard,errors=remount-ro,data=ordered)
proc on /home/hsiaoxychen/runc-bundle/rootfs/proc type proc (rw,relatime)
tmpfs on /home/hsiaoxychen/runc-bundle/rootfs/dev type tmpfs (rw,nosuid,size=65536k,mode=755)
devpts on /home/hsiaoxychen/runc-bundle/rootfs/dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,pt
shm on /home/hsiaoxychen/runc-bundle/rootfs/dev/shm type tmpfs (rw,nosuid,nodev,noexec,relatime,size=65536k)
mqueue on /home/hsiaoxychen/runc-bundle/rootfs/dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime)
sysfs on /home/hsiaoxychen/runc-bundle/rootfs/sys type sysfs (ro,nosuid,nodev,noexec,relatime)
tmpfs on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup type tmpfs (rw,nosuid,nodev,noexec,relatime,mode=75
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/cpuset type cgroup (ro,nosuid,nodev,noexec,relatim
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/cpu type cgroup (ro,nosuid,nodev,noexec,relatime,c
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/cpuacct type cgroup (ro,nosuid,nodev,noexec,relati
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/blkio type cgroup (ro,nosuid,nodev,noexec,relatime
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/memory type cgroup (ro,nosuid,nodev,noexec,relatim
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/devices type cgroup (ro,nosuid,nodev,noexec,relati
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/freezer type cgroup (ro,nosuid,nodev,noexec,relati
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/net_cls type cgroup (ro,nosuid,nodev,noexec,relati
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/perf_event type cgroup (ro,nosuid,nodev,noexec,rel
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/net_prio type cgroup (ro,nosuid,nodev,noexec,relat
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/hugetlb type cgroup (ro,nosuid,nodev,noexec,relati
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/pids type cgroup (ro,nosuid,nodev,noexec,relatime,
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/rdma type cgroup (ro,nosuid,nodev,noexec,relatime,
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/misc type cgroup (ro,nosuid,nodev,noexec,relatime,
cgroup on /home/hsiaoxychen/runc-bundle/rootfs/sys/fs/cgroup/systemd type cgroup (ro,nosuid,nodev,noexec,relati
*/
	err := prepareRootfs(l.pipe, l.config, l.mountFds)
	if err != nil {
		return err
	}

	// 设置主机名
	if hostname := l.config.Config.Hostname; hostname != "" {
		if err := unix.Sethostname([]byte(hostname)); err != nil {
			return &os.SyscallError{Syscall: "sethostname", Err: err}
		}
	}
	if domainname := l.config.Config.Domainname; domainname != "" {
		if err := unix.Setdomainname([]byte(domainname)); err != nil {
			return &os.SyscallError{Syscall: "setdomainname", Err: err}
		}
	}

	// 设置 sysctl
	for key, value := range l.config.Config.Sysctl {
		if err := writeSystemProperty(key, value); err != nil {
			return err
		}
	}

	// 通知父进程我们已经 ready 了，可以继续后边的流程了
	// （别忘了上边的 CLONE_PARENT 选项，我们的父进程是 runc 主进程！）
	// Tell our parent that we're ready to Execv. This must be done before the
	// Seccomp rules have been applied, because we need to be able to read and
	// write to a socket.
	// 正如注释所说，我们要在 seccomp 生效之前把各种该干的活都给干了
	// 不然受到 seccomp 的限制之后，很多 syscall 都可能被禁用掉了
	if err := syncParentReady(l.pipe); err != nil {
		return fmt.Errorf("sync ready: %w", err)
	}

	// 检查对应目录下是否有配置的入口程序文件
	// 当入口程序不存在时，其实是这里报的错（比如尝试进入一个不携带 sh 的容器时）：
	// ERRO[0000] runc run failed:
	// unable to start container process: 
	// error during container init: 
	// exec: "a-fake-path": executable file not found in $PATH

	// Check for the arg before waiting to make sure it exists and it is
	// returned as a create time error.
	name, err := exec.LookPath(l.config.Args[0])
	if err != nil {
		return err
	}

	// 等待 runc 主进程通知可以继续执行（利用往 exec.fifo 写入数据导致的阻塞来同步这一信息）
	// Wait for the FIFO to be opened on the other side before exec-ing the
	// user process. We open it through /proc/self/fd/$fd, because the fd that
	// was given to us was an O_PATH fd to the fifo itself. Linux allows us to
	// re-open an O_PATH fd through /proc.
	fifoPath := "/proc/self/fd/" + strconv.Itoa(l.fifoFd)
	fd, err := unix.Open(fifoPath, unix.O_WRONLY|unix.O_CLOEXEC, 0)
	if err != nil {
		return &os.PathError{Op: "open exec fifo", Path: fifoPath, Err: err}
	}
	if _, err := unix.Write(fd, []byte("0")); err != nil {
		return &os.PathError{Op: "write exec fifo", Path: fifoPath, Err: err}
	}

	// 更新容器状态为 Created 并执行相应的 hook
	// 值得再次注意的是这个 hook 将在容器内部 1 号进程执行
	s := l.config.SpecState
	s.Pid = unix.Getpid()
	s.Status = specs.StateCreated
	if err := l.config.Config.Hooks.Run(configs.StartContainer, s); err != nil {
		return err
	}

	// 一切就绪，直接 exec syscall 让入口程序鸠占鹊巢，成为容器内的 1 号进程！
	return system.Exec(name, l.config.Args, os.Environ())
}
```

至此，在 `exec` syscall 之后，这个 1 号进程就彻底成为了容器配置的入口进程。并且 runc 在这之前已经把该清理的全都给清理了（环境变量、fd 等），把该修改的配置（namespace、cgroup、mount 等）也都设置了。我们的进程将与一个正常运行的进程无样，但是所有常规的资源都已被限制在了容器之中。
