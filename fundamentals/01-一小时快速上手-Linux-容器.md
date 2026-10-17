# 一小时快速上手 Linux 容器

从 Linux 进程出发，亲手组装一个容器，再看到容器的安全边界。

---

## 1. Coding Agent 为什么必须运行在 sandbox 中

以 Claude Code 为代表的 Coding Agent，在深刻改变世界和程序员工作方式的同时，也带来了一大类安全问题。具体来说，没有人能 100% 保证 Agent 写出来的代码不会删库跑路，不会下载病毒文件，也不会越权访问不该访问的文件。

在这个过程中，不管你是前端、后端还是算法工程师，都越来越多地或主动、或被动地和沙箱、权限、鉴权等概念打交道。尤其是作为 AI 工程师，你可能还要选择、使用，甚至设计各种沙箱。因此，理解沙箱背后的原理就显得尤为重要。

这个课程会从第一性原理出发，先讲为什么需要沙箱，再以完全实操、交互式的方式，深入浅出地介绍最有代表性的一类沙箱实现——Linux 容器。

### 1.1 Coding Agent 执行的代码为什么不可信

这里所说的“不可信”，并不等于代码一定是恶意的。更多时候，代码只是写错了，但错误代码和恶意代码可能造成完全相同的后果。

第一类是无心之失。Agent 生成清理命令时算错了路径，可能删除工作目录之外的文件；为了恢复一个“干净”的 Git 状态，它可能运行 `git clean -fdx`，顺手删掉尚未提交的代码、测试数据和模型权重。一个没有退出条件的循环可能占满 CPU；一个不断向列表追加数据的 `for` loop 可能吃光内存；一段反复创建子进程的测试代码，则可能让整台机器无法再创建新的进程。

第二类是安装和运行代码时产生的副作用。Agent 为了解决一个 import error，可能直接安装一个名字相似但并不可信的软件包；`npm install`、`pip install` 和各种构建工具也可能自动执行第三方提供的安装脚本。这些脚本可以下载和运行新的程序、修改配置文件，甚至在用户毫不知情的情况下常驻后台。即使 Agent 自己写出的那几行代码没有问题，它拉取和执行的依赖也未必可信。

第三类是代码或输入本身带有恶意。Agent 处理的仓库、README、issue、测试数据和网页内容都可能由攻击者控制。其中可以藏有 prompt injection，诱导 Agent 读取环境变量、SSH key 或云凭证，也可以直接包含恶意测试、构建脚本和二进制文件。一段恶意代码甚至不需要删除任何东西，只要悄悄读取源码并上传出去，就已经造成了严重后果。

第四类是权限被滥用。Agent 启动的程序通常会继承当前用户的文件权限、环境变量和网络访问能力。如果开发机能够访问公司内网、云服务、Docker socket 或 Kubernetes 配置，那么不可信代码也可能拥有相同的入口。它未必需要利用漏洞“逃出”任何地方，只需要使用我们主动交给它的权限，就足以产生破坏。

Coding Agent 让这个问题变得更加突出，是因为“生成代码”和“执行代码”被放进了同一个自动循环。过去，错误代码可能要经过人工阅读、编译和运行才会产生影响；现在，Agent 可以在几秒钟内连续生成命令、安装依赖、运行测试，并根据结果继续采取下一步行动。因此，我们不能把安全建立在“Agent 应该不会写出危险代码”这个假设上。

### 1.2 Sandbox 要回答的四个问题

在一个完全自由的执行环境里，上面这些事故完全有可能发生。所谓沙箱，不是不让 Agent 生成这些代码，而是确保当 Agent 生成这些代码之后，它们要么无法按照 Agent 的意图执行，要么只能造成有限的危害。

具体来说，设计一个沙箱时，我们主要需要回答四个问题：

1. **它看得到什么？** 不可信代码能否看到宿主机上的其他进程、源代码、配置文件、环境变量和凭证？看不到，通常比看得到但依靠程序自觉不去访问更安全。
2. **它改得了什么？** 哪些目录可以写，哪些文件只能读，系统文件和沙箱外的工作区是否可能被修改？即使代码必须修改项目文件，也不代表它应该能够修改整台机器。
3. **它能联系谁？** 它能否访问互联网、localhost、公司内网和云服务？完全断网最容易控制；如果业务必须联网，就需要进一步限制它能够访问的目标。
4. **它能消耗多少资源？** 它最多能使用多少 CPU、内存和进程？一个程序即使看不到任何秘密，也仍然可能通过死循环、不断申请内存或创建进程拖垮整台机器。

因此，沙箱不需要判断一段代码究竟是恶意代码，还是一段不小心写错的代码。它只需要划定边界：无论代码出于什么意图，最坏能够造成多大的影响。

---

## 2. 准备一次性的 Linux 实验机

这门课程会以 Lima 作为实验环境。Lima 是一个可以在 macOS 上创建和管理轻量级 Linux 虚拟机的工具，它让我们能够在一台随时可以删除和重建的 Linux 机器中放心地进行实验。

接下来，我们会在 Linux 中一步步“手搓”出容器，并在里面模拟删除文件、消耗资源、访问网络和尝试越权等操作。相比只阅读概念，这些实验会让你更直观地感受到每一种隔离机制解决了什么问题，以及它没有解决什么问题。

在开始之前，有必要先区分“沙箱”和“容器”这两个经常被混用的名词。在开发 AI 应用时，很多人会把两者当成同一件事，但它们实际上处于不同的层次。

沙箱是一个基于目标的概念：它的目标是为不可信代码划定边界，限制代码能够看到、修改、访问和消耗的资源。至于这道边界具体如何实现，并没有唯一答案。

Linux 容器则是一组具体的操作系统隔离机制，也是实现沙箱最常见的方式之一。但容器并不天然等于安全沙箱：一个拥有过多权限、挂载了宿主目录且没有资源限制的容器，可能无法有效隔离不可信代码。只有正确组合文件系统、网络、用户权限、资源限制和系统调用过滤等机制之后，容器才能真正承担沙箱的作用。

我们会先从 Linux 容器讲起，因为它最适合用来理解隔离的基本原理。后面的课程还会继续介绍 gVisor、独立虚拟机等其他沙箱实现，并比较它们在隔离强度、兼容性和性能上的取舍。

### 2.1 安装 Lima

首先在 macOS 的终端中通过 Homebrew 安装 Lima：

```bash
brew install lima
```

安装完成后，运行下面的命令创建并启动课程专用的 Linux VM：

```bash
limactl start --name=sandbox-lab --mount-none --containerd=none --cpus=4 --memory=4 template:default
```

这个命令的主要参数如下：

- `limactl start`：启动一个 Lima VM；如果指定的 VM 还不存在，Lima 会先创建它。
- `--name=sandbox-lab`：把这台 VM 命名为 `sandbox-lab`，后续进入、停止和删除它时都会使用这个名字。
- `--mount-none`：不把 macOS 上的目录挂载进 VM。这样即使实验中的命令删除文件，也无法直接删除 Mac 上的真实文件。
- `--containerd=none`：暂时不安装和启动 containerd。我们希望先从 Linux 原理出发手工构造容器，而不是一开始就使用现成的容器运行时。
- `--cpus=4`：给 VM 分配 4 个虚拟 CPU。
- `--memory=4`：给 VM 分配 4 GiB 内存。
- `template:default`：使用 Lima 默认的 Ubuntu 模板。

第一次执行时，Lima 需要下载 Linux 镜像并创建虚拟磁盘，因此可能需要等待几分钟。当终端显示 `READY` 时，VM 已经启动完成。

进入 VM：

```bash
limactl shell sandbox-lab
```

执行后，当前终端会从 macOS shell 进入 VM 中的 Linux shell。接下来课程中的命令，除非特别注明，都应该在这个 Linux shell 中执行。

可以运行下面的命令确认环境：

```bash
uname -a
```

预期会看到包含 `Linux` 的 kernel 信息，而不是 macOS 的 `Darwin`：

```text
Linux sandbox-lab ...
```

再查看 Linux 发行版：

```bash
cat /etc/os-release
```

预期会看到 Ubuntu 的名称和版本信息。此时，我们已经进入了一台与 macOS 隔离的 Linux 虚拟机。后续即使实验破坏了这台机器，也可以删除它并重新创建，而不会直接破坏 macOS 系统。

参考：[Lima `limactl start` 文档](https://lima-vm.io/docs/reference/limactl_start/)

### 2.2 安装课程工具

进入 Lima VM 后，安装后续实验需要的工具：

```bash
sudo apt update && sudo apt install -y util-linux procps iproute2 curl strace libcap2-bin systemd-container debootstrap
```

其中，`util-linux` 提供 `unshare` 等 namespace 工具，`procps` 和 `iproute2` 用于观察进程与网络，`strace` 和 `libcap2-bin` 用于观察系统调用与 capabilities，`systemd-container` 和 `debootstrap` 则用于最后的 `systemd-nspawn` 容器实验。`apt update` 只更新软件包索引，`apt install -y` 会自动确认并安装这些软件。

如果你现在还不理解 namespace、系统调用、capabilities 或 `systemd-nspawn` 是什么，完全没有关系。这里仅仅是在一次性准备实验环境，后面我们会通过具体实验逐一解释这些概念和工具。

### 2.3 确认没有挂载 macOS Home

```bash
mount | grep '/Users/' || echo "macOS home is not mounted"
```

“挂载 macOS Home”是指让 VM 直接看到 Mac 上 `/Users/...` 目录里的真实文件。如果存在这种挂载，实验代码即使不能修改这些文件，也可能读取并泄露它们。因为创建 VM 时使用了 `--mount-none`，这里预期只会看到：

```text
macOS home is not mounted
```

这表示后续实验看到和修改的都是 VM 内部文件，不是 Mac 上的真实文件。从下一节开始，除非特别注明，所有命令都应该在 `sandbox-lab` 的 Linux shell 中执行。

后面大多数实验都会在命令完成后自动回到当前 shell，不需要每节手动执行 `exit`。只有当我们明确进入一个交互式容器 shell 时，才需要用 `exit` 返回 Lima VM。不要在其他位置习惯性地输入 `exit`，否则可能直接退出 Lima，让后续 Linux 命令误跑在 macOS 上。

---

## 3. 没有隔离的进程能做什么

这一节的所有命令都在 Lima VM 中运行。我们先不添加任何隔离措施，看看一个普通进程默认能够做什么。这会成为后续实验的比较基准。

### 3.1 删除文件

先创建一个临时目录，并在其中放入一个名为 `important.txt` 的测试文件。随后，我们用一个新的 shell 模拟 Agent 启动的程序，让它删除目录中的文件：

```bash
mkdir -p /tmp/agent-work && echo important >/tmp/agent-work/important.txt && sh -c 'rm -f /tmp/agent-work/*'
```

检查目录：

```bash
ls -la /tmp/agent-work
```

预期目录仍然存在，但 `important.txt` 已经消失。整个过程中，操作系统并不知道这条删除命令是不是 Agent 误生成的；它只知道当前用户拥有删除这个文件的权限。

对操作系统来说，Agent 生成的代码就是普通进程。只要当前用户有权修改一个文件，它启动的程序通常也拥有相同的权限。

### 3.2 读取“凭证”

```bash
echo 'FAKE_TOKEN=super-secret' >/tmp/fake-credentials && sh -c 'cat /tmp/fake-credentials'
```

### 3.3 占用 CPU

```bash
timeout 5 sh -c 'while :; do :; done' & sleep 1; top -b -n 1 -o %CPU | head -n 12; wait
```

`while :` 是一个永远不会主动结束的循环。`timeout 5` 会在五秒后终止它，避免实验一直运行；中间的 `top` 会打印当前最消耗 CPU 的进程。预期可以看到运行循环的 `sh` 进程占用接近一个完整 CPU 核心，五秒后命令自动退出。

### 3.4 创建大量进程

除了 CPU 和内存，进程本身也是有限资源。下面的循环会一次启动 100 个后台 `sleep` 进程，用它们模拟一段程序在短时间内大量创建子进程：

```bash
sh -c 'for i in $(seq 1 100); do sleep 3 & done; echo "child processes: $(pgrep -P $$ | wc -l)"; wait'
```

`&` 让每个 `sleep` 在后台运行，因此 shell 不会等前一个结束后才创建下一个；`pgrep -P $$` 查找当前 shell 的直接子进程，预期会看到接近 100 的数量。最后的 `wait` 会等待它们全部结束，大约三秒后终端恢复正常。

这个例子有明确的进程数量和存活时间，不会像真正的 fork bomb 那样无限复制。但它已经足以说明：如果没有限制，一个普通程序可以快速创建大量进程，消耗 PID、内存和 CPU 调度资源。

### 3.5 观察它仍然只是普通进程

最后查看这个 shell 的进程身份，以及它当前所属的几个 namespace：

```bash
sh -c 'echo "PID=$$"; id; for ns in pid mnt net user; do printf "%-5s " "$ns"; readlink "/proc/$$/ns/$ns"; done'
```

第一行会打印 shell 的 PID。`id` 会显示它仍然使用 Lima VM 中当前用户的 UID 和 GID。后面几行会出现类似下面的内容：

```text
pid   pid:[4026531836]
mnt   mnt:[4026531841]
net   net:[4026531992]
user  user:[4026531837]
```

方括号里的数字可以看作 kernel 中这个 namespace 的标识。具体数字每台机器可能不同。再次运行同一条命令时，PID 会变化，但这些 namespace 标识通常保持不变，因为两个 shell 都处于 Lima VM 原来的 PID、mount、network 和 user namespace 中。下一节使用 `unshare` 创建新 namespace 后，其中一部分标识就会发生变化。

到这里可以看到，在加入任何隔离机制之前，不可信代码和开发者自己的命令处于同一个操作系统世界。它继承了当前用户的文件、进程和资源访问能力，而操作系统不会替我们判断它的意图。

---

## 4. namespace——让进程看到不同的世界

上一章中的程序之所以能够看到其他进程、访问相同的文件目录和使用同一套网络，是因为它和 Lima VM 中的其他程序共享同一个系统视图。Linux namespace 要解决的，正是“不同进程应该看到哪个世界”这个问题。

namespace 会把某一类系统资源的视图划分开来。例如，PID namespace 可以让一组进程只看到自己的进程列表，Mount namespace 可以让它们拥有不同的文件系统挂载方式，Network namespace 则可以提供独立的网卡和路由。进程仍然由同一个 Linux kernel 调度，但它们从 kernel 得到的信息和能够操作的资源范围已经不同。

Linux 提供了多种 namespace。这里不追求全部覆盖，而是选择几个最容易建立容器直觉的类型：我们先用 UTS namespace 做一个简单热身，然后依次观察 PID、Mount、User 和 Network namespace。

实验中会反复使用 `unshare` 命令。它的作用很直接：为指定类型创建一个新的 namespace，然后在里面启动一条命令或一个 shell。`unshare` 本身不是另一种隔离机制，而是让我们能够直接操作和观察 namespace 的实验工具。

还要提前注意，namespace 主要改变进程能够看到和操作的系统视图，却不负责限制它能消耗多少 CPU 或内存，也不负责过滤 syscall。后面的 cgroup、capabilities 和 seccomp 会继续补上这些边界。

### 4.1 UTS namespace：最简单的热身

`hostname` 是一台机器在操作系统和网络中的名字。它会出现在终端提示符、日志和一些网络服务中，用来帮助人们区分不同的机器。我们先查看这台 Lima VM 当前的 hostname：

```bash
hostname
```

预期会看到类似 `sandbox-lab` 的名字。通常来说，修改 hostname 会改变整台 Linux 机器的名字；但下面这条命令会先创建一个新的 UTS namespace，再在里面把 hostname 改成 `agent-sandbox`：

```bash
sudo unshare --uts sh -c 'hostname agent-sandbox; hostname'
```

`unshare --uts` 创建新的 UTS namespace，`sh -c` 则在这个 namespace 中依次执行改名和查看名称两个操作。命令输出应该是：

```text
agent-sandbox
```

命令执行完毕后，我们已经离开这个 namespace。再次查看 VM 的 hostname：

```bash
hostname
```

它仍然是原来的 `sandbox-lab`。也就是说，同一个 Linux kernel 中的两个进程，可以分别认为自己处于不同名字的机器中。namespace 通常不会复制一整套操作系统，而是让不同进程看到不同的系统视图。UTS namespace 的功能相对简单，我们把它作为热身，接下来再看几个对容器更重要的 namespace。

### 4.2 PID namespace：让进程以为自己是 PID 1

Linux 中运行的每个程序都是一个进程，每个进程都有一个编号，也就是 PID（Process ID）。`ps`、`top` 和 `kill` 等命令，都是通过 PID 来查看或操作进程的。在普通环境中，机器上的进程共享同一套 PID 编号，因此一个进程通常可以看到系统中的许多其他进程。

PID namespace 会为一组进程提供一套独立的进程编号和进程视图。同一个进程可以在 namespace 外拥有一个 PID，在 namespace 内又拥有另一个 PID。新 PID namespace 中的第一个进程会看到自己是 PID 1，就像它是一个独立 Linux 系统中的第一个进程一样。

下面创建一个新的 PID namespace，并在其中启动 shell：

```bash
sudo unshare --pid --fork --mount-proc sh -c 'echo "inside PID=$$"; ps -o pid,ppid,cmd'
```

预期现象：

- shell 认为自己是 PID 1；
- `ps` 只能看到 namespace 内的进程；
- 从 Lima VM 的另一个终端仍然能看到它在外部的真实 PID。

可选双终端实验：

```bash
sudo unshare --pid --fork --mount-proc sh -c 'sleep 60 & echo "inside sleep PID=$!"; wait'
```

另一个终端：

```bash
pgrep -a sleep
```

这几个现象放在一起，可以看出 PID namespace 的边界：

- PID namespace 隔离 PID 编号和进程视图；
- 它没有减少系统中的真实进程数量；
- 它不会阻止 fork bomb；
- namespace 中的 PID 1 还承担回收孤儿进程等特殊职责。

### 4.3 Mount namespace：不同的文件系统视图

Linux 把磁盘、临时文件系统和其他存储资源组织成一棵从 `/` 开始的目录树。所谓 mount，可以简单理解为把一个文件系统或目录“接到”这棵目录树中的某个位置。通常，一个进程执行的 mount 会改变整台机器的文件系统视图，其他进程也能看到这个变化。

Mount namespace 为进程提供独立的挂载表。一个进程可以在自己的 namespace 中挂载或卸载文件系统，而不影响外面的进程。不过，它隔离的是“目录树如何拼装”这件事，并不会自动复制底层文件，也不会自动把原有文件变成只读。

下面先准备一个空目录，稍后把临时文件系统挂载到这里：

```bash
sudo mkdir -p /mnt/sandbox-demo
```

只在新 namespace 中挂载 tmpfs：

```bash
sudo unshare --mount sh -c 'mount -t tmpfs tmpfs /mnt/sandbox-demo; touch /mnt/sandbox-demo/inside-only; ls -l /mnt/sandbox-demo'
```

在新 namespace 内，应该能看到刚创建的文件，输出大致如下：

```text
total 0
-rw-r--r-- 1 root root 0 ... inside-only
```

这里的 `total 0` 不是说目录中没有文件，而是说这些文件占用的数据块总量为 0。`touch` 创建的是一个零字节文件，所以即使 `inside-only` 已经存在，第一行仍可能显示 `total 0`。

退出后查看：

```bash
ls -l /mnt/sandbox-demo
```

这一次通常只会看到：

```text
total 0
```

`inside-only` 消失了。它刚才被创建在新 namespace 中挂载的 tmpfs 里；命令结束后，我们回到了原来的 mount namespace。这里的 `/mnt/sandbox-demo` 仍然是原本那个空目录，从未挂载过刚才的 tmpfs。这说明 mount namespace 隔离的是进程看到的挂载关系：同一个路径，在两个 namespace 中可以指向不同的文件系统。

#### 反例：mount namespace 本身不会保护文件

```bash
mkdir -p /tmp/shared-work && echo important >/tmp/shared-work/canary
```

```bash
sudo unshare --mount sh -c 'rm /tmp/shared-work/canary'
```

外部检查：

```bash
ls /tmp/shared-work/canary
```

文件仍然被删除。虽然 mount table 已经不同，但进程看到的目录仍然指向相同的底层文件。

#### 加入只读保护

上一个实验已经删除了 `canary`。先重新创建这个测试文件，并在里面写入一行 `important`：

```bash
echo important >/tmp/shared-work/canary
```

这里的 `>` 会把 `echo` 输出的文字写入文件；如果文件不存在就创建它，如果已经存在则覆盖原内容。`important` 没有特殊含义，只是用来代表一份不希望被沙箱内程序删除或修改的数据。

```bash
sudo unshare --mount sh -c 'mount --bind /tmp/shared-work /tmp/shared-work; mount -o remount,bind,ro /tmp/shared-work; rm /tmp/shared-work/canary'
```

这里的只读保护分成了两步。

第一步：

```bash
mount --bind /tmp/shared-work /tmp/shared-work
```

bind mount 可以把一个已有目录挂载到另一个位置。这里的源路径和目标路径相同，看上去什么都没变，但它会让 `/tmp/shared-work` 成为一个独立的挂载点。可以把它理解为：先在这个目录外面单独套一层“挂载外壳”，这样下一步就能只修改这一层的属性。

第二步：

```bash
mount -o remount,bind,ro /tmp/shared-work
```

`remount` 表示修改现有挂载，`bind` 表示修改的是刚才创建的 bind mount，`ro` 则把它标记为 read-only。随后执行 `rm` 时，预期会失败：

```text
rm: cannot remove '/tmp/shared-work/canary': Read-only file system
```

因为这两次挂载都发生在新 mount namespace 中，所以只读属性只对里面的进程生效。命令结束后回到外面，文件仍然存在，而且原来的目录仍然可以写入：

```bash
cat /tmp/shared-work/canary
echo still-writable >/tmp/shared-work/outside
```

这里并没有复制 `/tmp/shared-work` 里的文件，也没有修改文件自身的权限。内外两边看到的仍是同一批底层文件，只是 namespace 内的进程要经过一个只读的挂载点，因此写入和删除都会被 kernel 拒绝。

这个实验也说明，mount namespace 不只是决定进程能看到哪些挂载，还能让同一个目录在不同 namespace 中具有不同的挂载属性。只读挂载与普通文件权限会共同决定进程最终能否修改文件。

### 4.4 User namespace：里面的 root 不等于外面的 root

Linux 使用 UID（User ID）和 GID（Group ID）识别用户与用户组，并据此判断一个进程能否读取文件、修改系统配置或执行特权操作。UID 0 通常就是 root，拥有普通用户没有的系统权限。

User namespace 允许内外使用不同的 UID 和 GID 映射。一个进程在 namespace 内可以看到自己是 UID 0，但在外部仍然只是普通用户。这样，程序可以在自己的隔离环境里获得类似 root 的管理能力，却不自动获得外部系统的完整 root 权限。

Ubuntu 24.04 默认还会通过 AppArmor 限制普通程序使用 user namespace。这是一项额外的安全加固：user namespace 可以帮助普通用户构造沙箱，但也会让他们接触更多 kernel 功能，因此 Ubuntu 默认收紧了它的使用。这里不展开研究 AppArmor；为了让下面的实验能够运行，我们只在这台一次性 Lima VM 中临时关闭该限制：

```bash
echo 0 | sudo tee /proc/sys/kernel/apparmor_restrict_unprivileged_userns
```

输出应为 `0`。这个修改不会永久保存，重启 VM 后便会恢复默认值，也不应该作为生产服务器上的通用配置。

下面创建一个新的 user namespace，并把当前用户映射成里面的 root：

```bash
unshare --user --map-root-user sh -c 'id; cat /proc/self/uid_map'
```

这一次应该成功。`id` 会显示 namespace 内的 UID 和 GID 都是 `0`。`uid_map` 的输出大致如下：

```text
         0        501          1
```

具体数字可能不同。三列分别表示“namespace 内从 UID 0 开始”“映射到外部当前用户的 UID”“一共映射 1 个 UID”。可以运行 `id -u`，确认中间一列正是自己在外部的真实 UID。

创建文件，并在 namespace 内外观察 owner：

```bash
rm -f /tmp/userns-file && unshare --user --map-root-user sh -c 'touch /tmp/userns-file; ls -ln /tmp/userns-file' && ls -ln /tmp/userns-file
```

准备只有真正 root 可以读取的文件：

```bash
sudo sh -c 'echo HOST_SECRET >/root/fake-secret && chmod 600 /root/fake-secret'
```

以 namespace 中的 root 尝试读取：

```bash
unshare --user --map-root-user sh -c 'id; cat /root/fake-secret'
```

因此，一个进程即使在自己的 user namespace 中显示为 UID 0，也不代表它拥有外部 Linux 系统中的完整 root 权限。

完成这一节后，可以立即恢复 Ubuntu 的默认保护：

```bash
echo 1 | sudo tee /proc/sys/kernel/apparmor_restrict_unprivileged_userns
```

### 4.5 Network namespace：最容易证明的断网方式

程序访问网络时，需要使用 Linux kernel 提供的网卡、IP 地址、路由表和端口。普通进程默认共享同一套网络环境，因此它不仅可能访问互联网，也可能访问 localhost、Lima VM 中的其他服务，以及 VM 能够到达的内网地址。

Network namespace 可以给一组进程提供独立的网络环境，包括自己的网卡、地址、路由和端口。一个刚创建的 network namespace 没有可用的外部网卡和路由，因此其中的程序默认无法访问外部网络。这为“不可信代码默认断网”提供了一个很直接的基础。

我们先在普通网络环境中访问一个网页，确认 VM 当前可以联网：

```bash
curl -I --max-time 3 https://example.com
```

新 network namespace：

```bash
sudo unshare --net sh -c 'ip addr; ip route; curl -I --max-time 3 https://example.com'
```

`unshare --net` 创建一个新的 network namespace，并在其中启动 `sh`。`ip addr` 查看里面有哪些网卡和 IP 地址，`ip route` 查看路由表，`curl` 则尝试访问外网。你会看到里面只有一块处于 DOWN 状态的 `lo`，路由表也是空的；由于没有能够通向外部的网卡和路由，`curl` 会在三秒内失败。这里的 `-I` 表示只请求网页的响应头，`--max-time 3` 用来避免命令长时间等待。

启用 namespace 自己的 loopback：

```bash
sudo unshare --net sh -c 'ip link set lo up; ip addr show lo'
```

这条命令又创建了一个新的 network namespace，然后用 `ip link set lo up` 启用其中的 loopback 网卡。输出中应该能看到 `lo` 处于 `UP` 状态，并拥有 `127.0.0.1` 和 `::1` 等本地地址。loopback 只允许同一个 network namespace 内的程序通过 localhost 相互通信；启用它并不会让程序获得外网访问能力。

从这组实验可以看到，network namespace 给进程提供了一套独立的网卡、IP 地址、路由和端口空间。新建出来的空间默认无法访问外网，但它只负责隔离网络环境，并不能直接表达“只允许访问某个域名”这样的精细规则；这类策略通常还要配合 proxy 或 firewall。

---

## 5. cgroup——限制进程能消耗多少资源

前面的 namespace 可以让不可信代码看不到外部进程、文件挂载和网络，但“看不到”不等于“用不到”。一个处于独立 PID namespace 中的程序，仍然可以创建大量真实进程；一个已经断网的程序，也仍然可以通过死循环占满 CPU，或者不断申请内存拖垮整台机器。

cgroup 是 control group 的缩写。它是 Linux kernel 提供的资源管理机制，可以把一组进程放进同一个层级中，然后统一限制和统计它们使用的 CPU、内存和进程数量等资源。限制会作用于整个进程组及其子进程，因此程序不能简单地通过再启动一个子进程逃避限制。

现代 Linux 通常使用 cgroup v2。先确认我们的实验机也是如此：

```bash
stat -fc %T /sys/fs/cgroup
```

预期输出：

```text
cgroup2fs
```

这里的 `cgroup2fs` 表示 `/sys/fs/cgroup` 挂载的是 cgroup v2 文件系统。cgroup 的配置最终表现为这个目录下的一组文件，kernel 通过这些文件接收限制并提供统计信息。

直接操作这些文件很容易与系统自己的资源管理发生冲突，因此下面使用 `systemd-run` 创建临时的 cgroup。systemd 帮我们创建和清理分组，真正执行资源限制的仍然是 Linux kernel。

### 5.1 CPU

CPU 限制控制的是一个进程组能够获得多少 CPU 执行时间，而不是降低虚拟 CPU 本身的频率。我们先运行一个纯计算任务，并用 `time` 测量它需要多久：

```bash
time python3 -c 'sum(i*i for i in range(50_000_000))'
```

然后把完全相同的任务放进一个 CPU quota 为 20% 的 cgroup：

```bash
time sudo systemd-run --quiet --wait --pipe -p CPUQuota=20% python3 -c 'sum(i*i for i in range(50_000_000))'
```

`CPUQuota=20%` 表示这组进程最多使用相当于单个 CPU 核心 20% 的执行时间。由于它经常需要等待下一轮调度额度，第二次运行的实际时间应该明显更长，具体倍数会随机器性能和当时负载变化。

命令中的 `--wait` 让 `systemd-run` 等待任务结束，`--pipe` 把程序输出接回当前终端，`-p` 则用于设置这个临时服务的资源属性。

### 5.2 Memory

内存耗尽比 CPU 死循环更危险：CPU 忙时系统通常只是变慢，内存耗尽时 kernel 可能不得不杀死进程来回收空间。如果没有单独限制，不可信程序的内存压力会和系统中其他进程混在一起。

先让 Python 正常申请 200 MiB 内存：

```bash
python3 -c 'x=bytearray(200*1024*1024); print(len(x))'
```

这条命令应该成功输出 `209715200`。接下来仍然申请 200 MiB，但把整个进程组的内存上限设为 100 MiB：

```bash
sudo systemd-run --wait --pipe -p MemoryMax=100M python3 -c 'x=bytearray(200*1024*1024); print(len(x))'
```

`MemoryMax=100M` 是一个硬上限。Python 连同解释器自身使用的内存一旦超过限制，kernel 就会在这个 cgroup 内触发 OOM，并终止其中的进程。终端可能显示 `Killed`、`oom-kill` 或任务失败，但 Lima VM 和其他 shell 应该继续正常工作。这正是资源隔离想要达到的效果：允许单个任务失败，而不是让它拖垮整个环境。

### 5.3 进程数量

一个程序还可能通过不断创建进程或线程耗尽系统的 PID、内存和调度资源，这类攻击通常被称为 fork bomb。下面的例子不会无限复制自己，而是有控制地尝试启动 100 个短暂的 `sleep` 进程，同时把整个进程组允许存在的 task 数量限制为 20：

```bash
sudo systemd-run --wait --pipe -p TasksMax=20 sh -c 'for i in $(seq 1 100); do sleep 3 & done; wait'
```

这里的 task 包括进程，也包括线程，而且 shell 和命令本身也会占用名额，因此不一定恰好能创建 20 个 `sleep`。达到上限后，shell 会报告无法 fork 或资源暂时不可用；与此同时，VM 中其他 cgroup 的进程仍然可以正常运行。

cgroup 的能力远不止这三个例子。它还可以控制进程之间竞争 CPU 时的相对权重、限定进程只能在哪些 CPU 核心上运行、限制磁盘读写带宽或 IOPS、限制 swap 和 huge pages，并持续记录每组进程的资源用量与压力。我们不需要逐一尝试这些参数；只要先建立一个整体印象：凡是“这组进程最多能用多少系统资源”的问题，通常都应该先想到 cgroup。

---

## 6. Capabilities——root 权限可以被拆开

cgroup 解决了“一个进程最多能消耗多少资源”，但它不决定进程有权执行哪些操作。一个严格限制了 CPU 和内存的进程，如果仍然拥有完整 root 权限，依然可能修改网络配置、挂载磁盘或改变其他文件的所有者。

在传统印象中，Linux 权限似乎只有两种状态：普通用户权限有限，root 则无所不能。现代 Linux kernel 把原本集中在 root 身上的特权拆成了许多相对独立的能力，这些能力就叫 capabilities。一个进程可以拥有修改网络配置的能力，却没有加载 kernel module 的能力；也可以保留绑定特殊端口的能力，同时放弃修改文件所有者的能力。

每个进程都有几组 capability。现阶段不需要记住它们之间的所有细节，只需要先认识两个直觉：effective 集合表示当前正在生效的能力，bounding 集合则像一个上限，限制当前进程及其后代未来最多能够获得哪些能力。

下面查看一个 root shell 当前拥有的 capabilities：

```bash
sudo capsh --print
```

输出中会出现 `Current`、`Bounding set` 等字段，以及一串以 `cap_` 开头的名称。我们这次只关注其中的 `cap_sys_chroot`。

`chroot` 可以改变一个进程眼中的根目录，是构造隔离文件系统时会用到的基础操作。即使下面只是把 `/` 再次设为根目录，执行这个动作仍然需要 `CAP_SYS_CHROOT`。root 默认拥有这个能力，因此命令会成功：

```bash
sudo chroot / /bin/true && echo success
```

接下来仍然使用 root 身份，但先从 capability bounding set 中删除 `CAP_SYS_CHROOT`，然后执行相同命令：

```bash
sudo capsh --drop=cap_sys_chroot -- -c 'chroot / /bin/true'
```

这次 `chroot` 应该返回 `Operation not permitted`。进程仍然是 UID 0，但因为缺少对应 capability，kernel 拒绝了这个特权操作。前后两次不同的结果说明，Linux root 并不是一个简单的全开或全关开关，我们可以只删除或保留其中一部分特权。

除了这里使用的 `CAP_SYS_CHROOT`，容器中经常关注的 capability 还包括：

- `CAP_CHOWN`：改变文件的所有者和用户组；
- `CAP_DAC_OVERRIDE`：绕过普通文件读写权限检查；
- `CAP_NET_ADMIN`：修改网卡、路由和防火墙等网络配置；
- `CAP_NET_RAW`：创建 raw socket；
- `CAP_SYS_PTRACE`：检查和控制其他进程；
- `CAP_SETUID` 和 `CAP_SETGID`：改变进程身份；
- `CAP_SYS_ADMIN`：覆盖 mount、namespace 等大量管理操作，是范围尤其广的一项能力。

不需要记住完整列表，更重要的是掌握最小权限原则：先删除所有 capability，再只加回程序确实需要的少数能力。同时也要注意，capability 限制的是特权操作，并不会取消进程原本就拥有的普通文件权限。如果一个 Agent 本来就拥有工作目录的写权限，即使删除全部 capability，它仍然可以删除这个目录中的文件。

即便把 root 特权拆得足够细，程序仍然能够调用大量普通系统调用。下一节的 seccomp 会继续从另一个方向缩小边界：限制一个进程可以进入哪些 kernel 接口。

---

## 7. seccomp——缩小能够调用的 kernel API

上一章的 capabilities 把 root 特权拆成了更小的权限单元，但一个没有任何 capability 的普通进程，仍然可以打开文件、创建进程、连接网络和申请内存。这些操作最终都需要进入 kernel，而程序进入 kernel 的入口，就是系统调用，也叫 syscall。

用户程序不能直接操作磁盘、网卡和内存页。当程序想读取文件时，它会调用 `openat`、`read` 等 syscall；想创建进程时，会调用 `clone` 或相关 syscall；想访问网络时，则会使用 `socket`、`connect` 等 syscall。Linux 为程序提供了数百个这样的入口。程序用到的入口越多，能够接触的 kernel attack surface 也越大。

seccomp 是 Linux 提供的系统调用过滤机制。我们可以为一个进程安装过滤规则，允许、拒绝或直接终止特定 syscall。规则还会约束它后续创建的子进程，因此特别适合限制只需要完成少数固定工作的程序。

这里要区分 capabilities 和 seccomp 的作用。capability 表示进程是否拥有执行某类特权操作的资格；seccomp 则可以让某个 syscall 根本无法通过。即使一个进程拥有对应 capability，只要 seccomp 拦住了 syscall，操作仍然不能执行。这两层限制可以叠加使用。

我们继续使用前面见过的 mount 操作来观察这种区别。`strace` 可以显示一个程序执行过程中调用了哪些 syscall。下面让它只打印名为 `mount` 的 syscall：

```bash
sudo strace -e mount mount -t tmpfs tmpfs /mnt/sandbox-demo
```

命令成功后，`strace` 输出中应该出现一次 `mount(...) = 0`，其中 `0` 表示 syscall 执行成功。实验结束后把刚才挂载的 tmpfs 卸载：

```bash
sudo umount /mnt/sandbox-demo
```

现在通过 systemd 安装一个 seccomp filter，禁止与 mount 相关的 syscall：

```bash
sudo systemd-run --wait --pipe -p 'SystemCallFilter=~@mount' -p SystemCallErrorNumber=EPERM mount -t tmpfs tmpfs /mnt/sandbox-demo
```

`SystemCallFilter=~@mount` 中的 `@mount` 是 systemd 预先整理的一组 mount 相关 syscall，前面的 `~` 表示拒绝这一组。`SystemCallErrorNumber=EPERM` 则告诉 kernel：遇到被拒绝的 syscall 时，不要直接杀死进程，而是返回一个“没有权限”的错误。再次运行 mount 后，预期会看到 `Operation not permitted`。

这里的 systemd 并不是一个站在程序和 kernel 之间、负责逐次检查请求的代理。它只是在启动程序之前，帮我们把 seccomp 规则安装到这个进程上。之后每当进程发起 syscall，真正执行检查和拦截的都是 Linux kernel。因此，即使程序绕过 shell、直接从 C、Python 或其他语言调用 `mount`，也无法绕过这条规则。

所谓缩小 kernel attack surface，可以把它想象成减少一栋建筑对外开放的门。Linux kernel 提供了数百个 syscall，但一个只负责解析文本的程序，可能根本不需要挂载文件系统、加载 kernel module 或调试其他进程。如果提前关掉这些永远用不到的入口，那么即使某个入口中存在尚未发现的漏洞，攻击者也无法从这个进程触达它。

不过，seccomp 看到的主要是 syscall 编号和少量直接参数，而不是程序的完整意图。例如，读取 `/workspace/code.py` 和读取 `/root/secret` 最终都可能使用同一个 `openat` syscall。文件路径以指针的形式存放在进程内存中，普通 seccomp filter 不适合根据路径内容表达“允许前者、拒绝后者”。如果彻底禁止 `openat`，程序连正常文件也无法打开；如果允许它，具体能打开哪些文件就要继续依靠 mount namespace、文件权限或 AppArmor、SELinux 等机制控制。

网络策略也是类似的。seccomp 可以禁止创建 socket，却不擅长表达“可以访问 GitHub，但不能访问公司内网”这样的高层规则。域名、IP 和连接方向通常需要交给 network namespace、firewall 或受控代理处理。

seccomp 的底层规则最终会表示成 kernel 可以执行的 BPF filter，但这门课程不要求编写 BPF 或 C 程序。使用 systemd 等现成工具就足以理解它的核心作用。更重要的是记住，seccomp 只负责其中一层：它擅长关闭不需要的 kernel 入口，却不能独自处理文件视图、用户身份、网络范围和资源上限。一个完整的沙箱，需要把前面学过的多层机制组合起来。

---

## 8. systemd-nspawn——把这些零件组装起来

到目前为止，我们一直在分别实验 Linux 的底层机制：namespace 改变进程看到的系统视图，cgroup 限制资源消耗，capabilities 拆分 root 特权，seccomp 则关闭不需要的 kernel 入口。但其中任何一种机制单独拿出来，都不能构成一个完整容器。

容器并不是 kernel 中某个名为“container”的独立功能。它更像一份组装方案：创建进程时，同时为它准备新的 namespace、文件系统视图和用户身份，再把它放进受限的 cgroup，删除不需要的 capabilities，并安装 seccomp filter。只有这些机制组合在一起，我们才得到一个看起来像独立 Linux 系统的受限执行环境。

这一节使用 `systemd-nspawn` 帮我们完成组装。它是 systemd 提供的轻量级容器工具，可以把一个 Linux root filesystem 作为容器启动，并通过命令行逐步加入只读文件系统、私有网络、capability 和 syscall 等限制。它没有隐藏太多底层细节，因此很适合用来观察前面学过的机制如何协同工作。

`systemd-nspawn` 是这一轮组装实验的阶段性成果，但还不是整部分课程的终点。等我们真正得到一个容器后，还要反过来检查它的边界，看看这些机制没有隔离什么。

### 8.1 一次性准备 rootfs

容器首先需要一套自己的文件系统，也就是 root filesystem，通常简称 rootfs。它不是一个正在运行的虚拟机，也不包含 kernel；它只是一个普通目录，里面按照 Linux 的习惯放着 `/bin`、`/etc`、`/lib`、`/usr` 等文件。容器启动后，这个目录会成为容器进程眼中的 `/`。

我们使用 `debootstrap` 下载一套最小化的 Debian 用户空间，并把它安装到 `/var/lib/machines/agent-rootfs`：

```bash
sudo debootstrap --variant=minbase stable /var/lib/machines/agent-rootfs https://deb.debian.org/debian
```

`--variant=minbase` 表示只安装维持基本系统所需的软件包，`stable` 表示使用 Debian 当前稳定版，最后的 URL 是软件包下载源。这一步需要下载和解压文件，可能等待几分钟。完成后可以查看这个目录：

```bash
sudo ls /var/lib/machines/agent-rootfs
```

这里需要使用 `sudo`，因为 `/var/lib/machines` 中的容器文件系统由 root 管理，当前普通用户可能没有读取权限。如果上一条命令已经显示 `Base system installed successfully`，随后不加 `sudo` 的 `ls` 报出 `Permission denied`，并不表示 rootfs 创建失败。

预期会看到 `bin`、`etc`、`usr`、`var` 等熟悉的 Linux 目录。值得注意的是，我们的 Lima VM 运行 Ubuntu，而这个 rootfs 来自 Debian。这没有问题，因为容器会使用自己的用户空间文件，却继续共享外部的 Linux kernel。

为后续 bind mount 准备目录：

```bash
sudo mkdir -p /var/lib/machines/agent-rootfs/work
```

### 8.2 启动最简单的容器

现在让 `systemd-nspawn` 把刚才的目录当成一个独立系统的根目录，并在其中启动 Bash：

```bash
sudo systemd-nspawn -D /var/lib/machines/agent-rootfs /bin/bash
```

`-D` 指定容器使用哪个目录作为 rootfs。启动过程中，nspawn 会帮我们创建必要的 namespace、切换根目录、挂载 `/proc`，然后运行 `/bin/bash`。终端提示符通常也会发生变化，表示当前 shell 已经位于容器内部。

在容器中观察 PID、hostname 和 kernel 版本：

```bash
echo "PID=$$"; hostname; uname -r
```

这个 Bash 通常会认为自己是 PID 1，hostname 也可能变成与容器名称相关的值。但 `uname -r` 显示的 kernel 版本仍然和 Lima VM 一样，因为 nspawn 创建的是容器，不是拥有独立 kernel 的虚拟机。

这里启动的是一个交互式 shell，实验完成后需要退出它：

```bash
exit
```

执行 `exit` 后会从 nspawn 容器返回 Lima VM，而不是返回 macOS。后面使用 `sh -c '...'` 的 nspawn 实验只执行一条命令，完成后会自动退出，不需要再次输入 `exit`。

### 8.3 只读 rootfs

默认情况下，容器内的 root 用户可以修改 rootfs。如果不可信代码并不需要安装软件或改变系统文件，我们可以把整个 rootfs 设为只读，只另外开放真正需要写入的工作目录。

下面以只读方式启动容器，并尝试在根目录创建文件：

```bash
sudo systemd-nspawn -qD /var/lib/machines/agent-rootfs --read-only sh -c 'touch /should-fail'
```

`-q` 只用于减少 nspawn 自己打印的提示信息。`touch` 应该得到 `Read-only file system`，随后命令自动退出。这个限制保护的是 rootfs；如果之后主动挂入一个可写目录，容器仍然可以修改那个目录中的内容。

### 8.4 私有网络

最简单可靠的网络策略通常是完全断网。`--private-network` 会让 nspawn 创建新的 network namespace，不再把 Lima VM 的网卡和路由直接交给容器。

下面查看私有网络中的路由表：

```bash
sudo systemd-nspawn -qD /var/lib/machines/agent-rootfs --private-network sh -c 'cat /proc/net/route'
```

预期只会看到表头，或者至少看不到通往外部网络的默认路由。容器因此无法直接访问互联网和 Lima VM 所能访问的内网。如果应用确实需要联网，后续可以再有选择地添加虚拟网卡、路由或受控代理，而不是默认继承全部网络能力。

### 8.5 删除 capabilities

容器中的进程可以显示为 root，但通常没有必要保留 root 的全部特权。`--drop-capability=all` 会把所有 capability 从容器进程的 capability bounding set 中删除。

下面让这个“没有任何 capability 的 root”再次尝试 `chroot`：

```bash
sudo systemd-nspawn -qD /var/lib/machines/agent-rootfs --drop-capability=all sh -c 'chroot / /bin/true'
```

预期会看到 `Operation not permitted`。这和前面的 capabilities 实验是同一条原理，只是这次 capability 限制已经由容器 runtime 在启动时配置完成。

### 8.6 加入 seccomp

最后加入 seccomp。为了让结果容易观察，我们禁止 `uname` syscall，然后在容器中运行同名命令：

```bash
sudo systemd-nspawn -qD /var/lib/machines/agent-rootfs --system-call-filter='~uname' uname -a
```

`uname` 程序本身仍然存在，也能够正常启动，但当它向 kernel 请求系统信息时，seccomp 会拦截这个 syscall，因此命令应该返回 `Operation not permitted`。这个规则只是为了演示，实际策略应该根据容器中程序真正需要的 syscall 来设计。

### 8.7 回顾 nspawn 做了什么

现在可以把几层限制放进同一次启动中：

```bash
sudo systemd-nspawn -qD /var/lib/machines/agent-rootfs --read-only --private-network --drop-capability=all --system-call-filter='~uname' sh -c 'echo "PID=$$"; id; grep -E "CapEff|CapBnd|Seccomp" /proc/self/status; cat /proc/net/route; touch /blocked'
```

这条命令看起来较长，但它只是把前面已经分别试过的选项写在一起。输出中可以依次观察到：

- shell 在自己的 PID namespace 中运行；
- `id` 仍可能显示 UID 0；
- `CapEff` 和 `CapBnd` 表明 capabilities 已被删除；
- `Seccomp` 表明系统调用过滤已经启用；
- 路由表中没有通向外部网络的默认路由；
- `touch` 因 rootfs 只读而失败。

前面分别实验过的机制，现在同时作用在同一个进程上。它们彼此并不替代：network namespace 不会让文件系统变成只读，删除 capability 不会限制 CPU，而 seccomp 也不会自动隐藏宿主目录。容器的隔离能力来自这些限制的叠加。

参考：[systemd-nspawn 文档](https://www.freedesktop.org/software/systemd/man/systemd.nspawn.html)

到这里，我们已经不再是分别测试某个 namespace 或限制参数，而是得到了一套相对完整的隔离环境。容器 runtime 的主要工作，正是可靠地组合前面分别实验过的这些 kernel primitives。

从底层隔离原理来看，Docker、Podman 等 Linux 容器工具，主要也是依赖 namespace、cgroup、capabilities、seccomp 和文件系统隔离。因此，可以把这一节看成是在“手搓一个基础版的 Docker 容器”。

当然，我们手工拼出的容器与真实 Docker 还有很多差别。Docker 还会负责镜像构建与分层、registry 分发、虚拟网络、volume、daemon API、日志以及容器的创建、停止和清理等生命周期管理。但通过前面的实验，我们已经大致理解了 Linux 容器最核心的隔离原理，也知道了常见容器参数最终是在配置哪些底层机制。

值得一提的是，Meta 最近推出的个人 Agent Muse，也在实际安全架构中使用了 `systemd-nspawn`。Muse 的核心 agent harness、工作区，以及 agent 代替用户执行的程序和工具，都运行在一个 nspawn runtime cell 中；真正的凭据、权限审批和网络出口控制等敏感服务则放在 cell 外。这个 runtime cell 还组合了独立 rootfs、User 和 Network namespace、受限 capabilities 与 syscall filter——也就是我们在这一节中刚刚逐层组装起来的这些机制。这说明 nspawn 不只是教学工具，同样可以成为真实 Agent 系统纵深防御的一层。[Meta 对 Muse 安全架构的介绍](https://research.meta.ai/blog/security-and-safety-for-ai-agents-our-approach-with-muse)

---

## 9. 容器的根本边界——共享 kernel

到目前为止，我们已经让容器拥有不同的进程列表、文件系统视图、用户身份和网络环境，还限制了它能使用的资源、特权和 syscall。从容器内部看，它确实很像一台独立的 Linux 机器。

但这个“像”有一个根本边界：容器没有自己的 kernel。容器中的进程仍然是 Lima VM 中的普通进程，它发起的 syscall 最终仍由 Lima VM 的 kernel 处理。namespace、capabilities 和 seccomp 也不是容器外面额外套上的另一层操作系统，它们本身就是由这个共享 kernel 实现和执行的。

### 9.1 一眼看到共享 kernel

先在 Lima VM 中查看 kernel 版本：

```bash
uname -r
```

再让 nspawn 容器执行同样的命令：

```bash
sudo systemd-nspawn -qD /var/lib/machines/agent-rootfs uname -r
```

两次输出应该完全相同。容器可以使用 Debian rootfs，而外部 Lima VM 使用 Ubuntu 用户空间，但它们的 `uname -r` 仍然一致，因为两边运行在同一个 kernel 上。

这也是容器轻量、启动快的主要原因之一：创建容器不需要重新启动一个完整操作系统，也不需要为每个容器运行一份 kernel。但从安全角度看，这也意味着 host kernel 成了所有容器共同依赖的一道边界。

### 9.2 共享 kernel 会带来哪些风险

第一个风险是 kernel 漏洞可能变成 container escape。容器中的程序每次调用 `openat`、`socket`、`clone` 等 syscall，都会进入 host kernel。如果其中某个 syscall 或相关 kernel 子系统存在可利用漏洞，攻击者就可能从“只能控制容器中的进程”升级为“控制 host kernel 或宿主机”。这并不意味着任何 kernel 漏洞都一定能逃逸：攻击者还必须能够触达相应接口，并满足漏洞的具体条件。capability 和 seccomp 的重要价值之一，就是减少这些可触达的入口。但为了运行 Python、Node.js、编译器和构建工具，真实沙箱通常仍然需要开放相当多的 syscall，因此无法把风险完全消除。

第二个风险是故障范围仍然共享。如果某段代码触发了 kernel panic、kernel deadlock 或严重的全局资源问题，影响的可能不是一个容器，而是共享这个 kernel 的所有容器。cgroup 可以限制许多常见的资源耗尽问题，却无法把 kernel 本身拆成多份彼此独立的故障域。

第三个风险是所有容器共享 host 的 kernel 版本和 kernel 配置。更新容器里的 Debian 或 Ubuntu 软件包，并不会更新真正处理 syscall 的 host kernel。修复 kernel 漏洞通常需要升级宿主机 kernel，有时还必须重启宿主机；同一台机器上的多个容器也会同时受到这个 kernel 安全状态的影响。

最后，容器边界还依赖 runtime 正确完成组装。runtime 在启动容器时需要创建 namespace、处理 mount、打开文件描述符、设置用户身份并安装安全策略。即使单个 kernel 机制本身没有问题，runtime 在这些步骤中犯错，也可能意外把宿主资源带进容器。因此，host kernel 和容器 runtime 都属于这套安全边界中必须信任的部分。

### 9.3 历史漏洞如何穿过这道边界

下面几个真实案例不需要记住漏洞编号，重点是观察：攻击者究竟利用了哪一层边界。

#### Dirty Pipe：read-only 最终仍由 kernel 执行

Dirty Pipe（CVE-2022-0847）是 Linux kernel 的 pipe 和 page cache 处理漏洞。正常情况下，进程不应该修改一个没有写权限或以只读方式挂载的文件；但这个漏洞可以在特定条件下直接修改 page cache 中的数据，绕过原本的文件权限和只读限制。

这说明 read-only mount 和文件权限并不是物理屏障，它们是 kernel 执行的规则。如果负责执行规则的 kernel 本身出现漏洞，容器内看到的“只读”就可能不再可靠。对这类问题，修改容器配置通常不够，真正的修复是升级并重启 host kernel。[Ubuntu 对 Dirty Pipe 的说明](https://ubuntu.com/security/vulnerabilities/dirtypipe)

#### CVE-2022-0492：用于限制容器的 cgroup 也可能成为入口

cgroup v1 曾提供一个名为 `release_agent` 的机制：当某个 cgroup 变空时，kernel 可以在用户空间执行一个清理程序。CVE-2022-0492 的问题在于，kernel 对设置这个程序路径的权限检查不够严格。在满足特定权限和配置条件时，容器中的攻击者可能诱导 host kernel 在容器外执行指定命令。

我们前面把 cgroup 当作限制 CPU、内存和进程数量的安全工具；这个案例提醒我们，任何暴露给容器的复杂 kernel 接口也可能成为攻击面。安全机制可以降低一种风险，但它自己的实现仍然需要正确。[Ubuntu CVE-2022-0492 安全说明](https://ubuntu.com/security/CVE-2022-0492)

#### runc：组装容器时泄漏了一条通往 host 的路径

2019 年的 CVE-2019-5736 允许恶意容器借助 `/proc/self/exe` 覆盖 host 上的 runc binary。当管理员下一次使用 runc 操作容器时，被修改的 binary 会以 host root 身份执行。这里出问题的不是某个 namespace 概念，而是 runtime 在 host 与容器之间切换和执行进程时，没有安全处理这条特殊的文件引用。[CVE-2019-5736 漏洞记录](https://github.com/advisories/GHSA-gxmr-w5mj-v8hh)

2024 年的 CVE-2024-21626 又展示了另一种相似风险：runc 的内部初始化阶段保留了指向 host 资源的文件描述符，而工作目录和路径处理没有充分确认最终位置仍在容器 rootfs 内。攻击者在特定条件下可以借此访问 host 文件系统，甚至进一步完成容器逃逸。[runc CVE-2024-21626 官方公告](https://github.com/opencontainers/runc/security/advisories/GHSA-xr7r-f8xq-vfvv)

这两个案例共同说明，namespace 隔离的不只是“功能是否打开”，还依赖 runtime 在非常复杂的启动过程中不泄漏任何已经打开的 host 资源。容器 runtime 本身也必须及时更新，不能只更新容器镜像。

### 9.4 容器仍然很有价值，但不是最强边界

共享 kernel 并不意味着容器没有安全价值。对于误删文件、普通依赖脚本、意外联网、CPU 死循环和内存耗尽等常见问题，正确配置的 namespace、cgroup、capabilities 和 seccomp 可以显著缩小影响范围。很多 Coding Agent 面对的首先也是这类“代码可能出错，但不一定专门研究 kernel 漏洞”的风险。

但如果沙箱要面对的是明确恶意的任意代码、互不信任的多租户，或者攻击者会主动寻找 kernel 和 runtime 漏洞，那么“所有人共享一个 kernel”就会成为需要认真评估的风险。此时，我们会希望在不可信代码和 host kernel 之间再增加一道更独立的边界。

后面的课程会介绍两条主要路线。gVisor 在应用程序和 host kernel 之间实现一层 userspace kernel，让大量应用 syscall 不再直接进入 host kernel；microVM 则通过硬件虚拟化为不可信代码提供独立的 guest kernel。它们通常能提供更强的隔离，但也会带来兼容性、性能、启动速度和资源开销上的取舍。

完成本部分实验后，可以先退出 Lima VM，返回 macOS：

```bash
exit
```

然后在 macOS 终端中停止 VM：

```bash
limactl stop sandbox-lab
```

停止只会关闭 VM，实验文件仍然保留，下次可以使用 `limactl start sandbox-lab` 继续。如果确定不再需要实验数据，也可以彻底删除 VM：

```bash
limactl delete sandbox-lab
```
