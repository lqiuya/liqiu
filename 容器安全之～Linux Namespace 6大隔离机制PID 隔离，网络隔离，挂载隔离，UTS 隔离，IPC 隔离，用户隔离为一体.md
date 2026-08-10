#  最详细的 Linux Namespace 6大隔离机制讲解，万字详细讲解，包含讲解+图片，深度解析

# Linux Namespace 6大机制
| 英文全称              | 中文名称          | 简称     |
| ----------------- | ------------- | ------ |
| PID Namespace     | **进程命名空间**    | PID 隔离 |
| Network Namespace | **网络命名空间**    | 网络隔离   |
| Mount Namespace   | **挂载命名空间**    | 挂载隔离   |
| UTS Namespace     | **主机名命名空间**   | UTS 隔离 |
| IPC Namespace     | **进程间通信命名空间** | IPC 隔离 |
| User Namespace    | **用户命名空间**    | 用户隔离   |

> Namespace 到底隔离了什么？容器？为什么要隔离？
>
>
> 如果你也想搞懂容器到底怎么隔离的，或者说正在学习云原生安全领域的知识，这篇文章可以提供原理和实验讲解

---

## 阅前了解

容器不是虚拟机。这句话被说烂了，但我一开始也没理解它的重量。

虚拟机的隔离是硬件级的——Hypervisor 切的是 CPU、内存、IO，每台虚拟机以为自己独占一台物理机。容器的隔离是内核级的——所有容器共享同一个内核，内核用 Namespace 给每个容器画了一道边界。

**Namespace 不是墙，是视角。** 同一个进程，在不同 Namespace 里看到的世界完全不同。内核没有复制进程，没有复制文件系统，它只是给进程换了一副眼镜，容器需要这个机制

这副眼镜有六片镜片：UTS、PID、Network、Mount、IPC、USER。本文讲我怎么看这六片镜片，以及怎么亲手验证它们。

**实验环境**：Debian 13 (trixie)，内核 6.12.94+。所有命令在隔离测试环境中执行，不要在生产服务器上跑，最好是x86系统，每一个人的环境不同，实验步骤可能会错误

**需要安装的工具包**：`util-linux`（提供 `unshare`）、`iproute2`（提供 `ip`）、`procps`（提供 `ps`）、`coreutils`（提供 `ipcmk`/`ipcs`）。Debian 默认应该下载了，如果出现问题可以检查这些工具是否存在。

---

## 0. Namespace 是怎么被创建和销毁的

内核管理 Namespace 只有三个入口：

| 机制 | 作用 | 场景 |
|------|------|------|
| `fork()` | 子进程继承父进程的 Namespace | 容器内启动新进程 |
| `unshare()` | 当前进程创建新 Namespace 并加入 | `unshare --pid` 实验 |
| `setns()` | 当前进程加入已有 Namespace | `docker exec` 进入容器 |

**一个容易踩的坑**：`setns()` 进入 Mount Namespace 后，之前打开的文件描述符仍然指向旧 Namespace 的文件系统对象。Namespace 切换了，但 fd 没跟着换。

**验证方法**：看 `/proc/[pid]/ns/` 下的符号链接，同一 Namespace 内的所有进程，inode 号相同。

```bash
# 查看当前进程的 Namespace
ls -la /proc/self/ns/
# 预期输出: uts:[4026531838]  pid:[4026531836]  .
..
# 启动一个后台进程
sleep 1000 &
BG_PID=$!

# 验证两个进程是否在同一个 PID Namespace
ls -li /proc/self/ns/pid /proc/$BG_PID/ns/pid
```
Namespace 的销毁采用引用计数。当最后一个进程退出或 `setns` 离开只是进程不再属于该 Namespace，真正销毁需要引用计数归零——没有进程属于它，也没有挂载点等外部引用。这就是容器进程全部退出后 Namespace 自动清理的底层原理。

---

## 一、UTS Namespace：hostname 的独立副本

### 1.1 一句话理解

UTS Namespace 给每个容器分配了一块独立的空间，存自己的 hostname。各改各的，互不干扰。

内核实现上，这是真正的**独立数据结构**——每个 UTS Namespace 有自己的 `struct uts_namespace`，存 hostname 和 NIS domain。不是过滤视图，是独立变量。

```
宿主机 UTS ns：hostname = "debian-host"
LQ1 UTS ns：  hostname = "container-a"  ← 独立变量
LQ2 UTS ns：  hostname = "container-b"  ← 独立变量
```

### 1.2 动手实验

```bash
# 终端1：查看当前 hostname
hostname
# 预期输出: debian-host

# 创建新的 UTS Namespace（不需要 sudo！）
unshare --uts /bin/bash

# 在新 Namespace 里修改
hostname LQ1-uts
hostname
# 预期输出: LQ1-uts

# 终端2（宿主机）：hostname 不变
hostname
# 预期输出: debian-host
```

### 1.3 验证隔离边界

```bash
# 在 LQ1 内
cat /proc/sys/kernel/hostname
# 预期输出: LQ1-uts

# 在宿主机
cat /proc/sys/kernel/hostname
# 预期输出: debian-host
```

同一个文件路径，不同 Namespace 返回不同值。内核根据当前进程的 UTS Namespace 指针，返回对应的数据结构。

### 1.4 与 Docker 的关联

```bash
# Docker 默认独立 UTS Namespace
docker run --rm ubuntu:22.04 hostname
# 预期输出: 随机字符串，如：a1b2c3d4e5f6

# 共享宿主机 UTS（危险：容器改 hostname 会影响宿主机）
docker run --rm --uts=host ubuntu:22.04 hostname
# 预期输出: debian-host
```

**⚠️ 生产环境严禁 `--uts=host`**。容器内 `hostname` 修改会直接影响宿主机。

### 1.5 小结

| 属性 | 值 |
|------|-----|
| 隔离内容 | hostname、NIS domain |
| 隔离方式 | **独立数据结构**（`struct uts_namespace` 独立分配） |
| 是否需要 root | 否 |
| Docker 默认 | ✅ 独立 |
| 危险参数 | `--uts=host` |

> **金句**：UTS 是内核给容器画了一块独立空间放 hostname。最简单，也最真实——因为它真的复制了一份数据。

---

## 二、PID Namespace：进程表的视角魔术

### 2.1 一句话理解

PID Namespace 不复制进程，只复制进程表的视角。内核维护独立的 PID 分配空间，让容器拥有一张"本地进程表"的错觉。

技术本质：内核为每个 PID Namespace 维护独立的 `struct pid_namespace`，包含独立的 PID 分配器。同一进程在不同 Namespace 中有不同的 PID 编号。内核通过 `struct pid` 中的 `level` 和 `numbers[]` 数组管理跨 Namespace 的 PID 映射。

### 2.2 为什么需要这层隔离

容器本质上就是宿主机上的一组普通进程。内核里有一张全局进程表，所有进程都在上面。如果没有 PID Namespace，容器里的 `ps` 一查，直接看到全局表——宿主机的 sshd、systemd、其他容器的进程，全暴露在视野里。

PID Namespace 给容器伪造了一张"本地进程表"。**容器查的是 Namespace 内的本地视图，宿主机查的是全局视图——底层是同一个进程，但编号体系独立。**

### 2.3 双重视角：同一进程，两个世界

假设 LQ1 容器正在运行，容器内的 bash 进程同时存在于两个视角中：

**LQ1 内部视角（`ps aux`）：**

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   4628  3840 pts/0    Ss   16:20   0:00 bash
root         8  0.0  0.0   8900  5120 pts/0    R+   16:21   0:00 ps aux
```

- PID 1 是你的 bash，你觉得这是整个世界
- 你看不到宿主机的 systemd、sshd
- 你看不到其他容器的进程

**宿主机视角（`ps aux | grep bash`）：**

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      2847  0.0  0.0   4628  3840 ?        Ss   16:20   0:00 bash
```

- 同一个 bash 进程，在宿主机上是 PID 2847
- 宿主机能看到全局进程表，包括 LQ1 的 bash、LQ2 的 bash、systemd、sshd
- 宿主机知道 LQ1 里的 "PID 1" 其实是全局的 PID 2847

**各看各的，底层同一个事实。**

### 2.4 真正的隔离需要什么

PID Namespace 切分了进程编号的命名空间，但 `ps` 命令实际读的是 `/proc` 文件系统——这是窗户。

- 如果 `/proc` 还是宿主机的全局 proc → 容器透过窗户看到了外面的世界
- 如果 `/proc` 被重新挂载为当前 Namespace 的本地 proc → 窗户封死，只能看到楼里

`--mount-proc` 就是封窗户的动作。

| 配置 | 结果 |
|------|------|
| 独立 PID Namespace + 重新挂载 proc | 各开一栋楼，完全隔离 |
| 独立 PID Namespace，忘记挂载 proc | 楼分开了，窗户没封，`ps` 还能看到外面 |
| 多个容器共享同一个 PID Namespace | 同一栋楼，楼道互通，互相杀伤 |

即使 Namespace 和 proc 都配好了，如果容器拿到了 `CAP_SYS_PTRACE` 权限，攻击者仍然可能突破操作隔离。**PID Namespace 隔离的是"视图"，操作权限还需要 Capabilities 和 seccomp 来限制。**

### 2.5 PID 1 的特殊语义

PID Namespace 里的 PID 1 不是普通进程，内核赋予了它三个特殊语义：

1. **信号忽略**：未显式设置处理函数的信号默认被忽略（普通进程默认终止）
2. **孤儿收养**：父进程先于子进程退出时，孤儿进程被 PID 1 收养
3. **Namespace 锚点**：PID 1 退出 → 内核发送 SIGKILL 给 Namespace 内所有进程 → Namespace 销毁

**容器安全关联**：
- Docker 默认把容器入口进程作为 PID 1
- 如果 PID 1 是 `bash`，收到 SIGTERM 不会优雅退出（因为 bash 没处理 SIGTERM），导致 `docker stop` 超时后强制 SIGKILL
- 攻击者如果能成为 PID 1，就能控制整个 Namespace 的生命周期

**实验验证**：
```bash
# 容器内，PID 1 是 bash
docker run -it --rm --name test-init ubuntu:22.04 bash
# 在容器内
kill -TERM 1
# bash 不会被终止！（因为未处理 SIGTERM，默认忽略）

# 换成 init 系统
docker run -it --rm --init ubuntu:22.04 bash
# --init 会插入 tini 作为 PID 1，bash 变成 PID 2
# tini 会正确处理信号，容器能优雅退出
```

### 2.6 实验：LQ1 和 LQ2 共享一个 PID Namespace

**实验目的**：验证"同一个 PID Namespace 内的进程互相可见，但看不到宿主机进程"。

**步骤 1：创建 LQ1**

```bash
docker run -it --name LQ1 --rm ubuntu:22.04 bash
```

在 LQ1 内执行 `ps aux`，预期：只能看到 LQ1 内的进程，PID 1 是 bash。

**步骤 2：创建 LQ2，共享 LQ1 的 PID Namespace**

```bash
# 在另一个终端执行，确保 LQ1 已启动
docker run -it --name LQ2 --pid=container:LQ1 --rm ubuntu:22.04 bash
```

在 LQ2 内执行 `ps aux`，预期：能看到 LQ1 和 LQ2 的所有进程。LQ1 的 bash 是 PID 1，LQ2 的 bash 是另一个 PID。**两个容器在同一栋楼，楼道互通。**

**步骤 3：在 LQ2 中向 LQ1 的进程发信号**

```bash
kill -9 1  # 仅在容器内执行！
```

预期：LQ1 容器被强制终止。**同一栋楼，互相杀伤。**

**步骤 4：验证看不到宿主机进程**

```bash
ps aux | grep systemd
```

预期：看不到宿主机的 systemd 进程。**Namespace 隔离了外面的视野，但同一 Namespace 内的进程互相透明。**

**步骤 5：清理**

```bash
docker stop LQ1 LQ2
```


### 2.7 踩坑记录：我漏了 `--mount-proc`

第一次跑 PID Namespace 实验时，我自信地敲了：

```bash
sudo unshare --pid --fork /bin/bash
ps aux
```

结果 `ps` 仍然列出了宿主机的所有进程——sshd、systemd、其他容器的 bash，全在。

** Namespace 不是隔离了吗？

排查了十分钟，才发现 `/proc` 还是宿主机的全局 proc。PID Namespace 切分了进程编号空间，但 `ps` 读的是 `/proc` 里的文件——**窗户没封**。

```bash
# 错误：只切了 Namespace，没换 proc
sudo unshare --pid --fork /bin/bash
ps aux
# 预期输出: 仍然看到宿主机所有进程（窗户没封）

# 正确：切 Namespace + 重新挂载 proc
sudo unshare --pid --fork --mount-proc /bin/bash
ps aux
# 预期输出: PID 1 是 bash，只有 Namespace 内的进程
```

**教训**：PID Namespace 隔离的是"编号空间"，`/proc` 是另一回事。两者必须同时配，缺一不可。

### 2.8 常见错误排查

| 现象 | 原因 | 解决 |
|------|------|------|
| `unshare --pid` 后 `ps` 仍显示宿主机进程 | 未 `--mount-proc` | 加 `--mount-proc` 或手动 `mount -t proc proc /proc` |
| `unshare --pid` 报错 Operation not permitted | 非 root 用户且无 CAP_SYS_ADMIN | 用 sudo 或检查 capabilities |
| 共享 PID Namespace 后看不到对方进程 | 容器实际未共享 | 确认 `--pid=container:xxx` 参数正确 |

### 2.9 实验结论

| 场景 | 能否看到对方进程 | 能否操作对方进程 |
|------|----------------|----------------|
| 独立 PID Namespace（默认） | ❌ | ❌（受 capabilities 限制） |
| 共享 PID Namespace | ✅ | ✅（同一空间内无额外隔离） |
| 宿主机进程 | ❌ | ❌（视图隔离生效） |

### 2.10 小结

| 属性 | 值 |
|------|-----|
| 隔离内容 | 进程编号 |
| 隔离方式 | **视图隔离**（独立的 PID 分配空间） |
| 是否需要 root | 创建需 root |
| Docker 默认 | ✅ 独立 |
| 危险参数 | `--pid=host` |

> **金句**：PID Namespace 维护独立的 PID 分配空间，让容器拥有"本地进程表"。同一 Namespace 内的进程互相透明，不同 Namespace 之间的进程互相隔离——但隔离的是"视图"，操作权限还需 Capabilities 限制。

---

## 三、Network Namespace：独立的网络协议栈

### 3.1 一句话理解

Network Namespace 给每个进程分配了一套独立的网络协议栈——独立的网卡、路由表、防火墙规则、socket 哈希表。刚创建时是"毛坯房"：只有 `lo`，其他什么都没有，也连不上网。

技术本质：内核维护独立的 `struct net` 实例。容器之间的通信靠 **veth 虚拟网卡对** 和 **Linux Bridge 网桥** 实现。

### 3.2 核心组件
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/d51918c2a2474c3680cf1517724a297f.png)
```

**veth pair**：一根虚拟网线，两头各一个虚拟网卡。A 和 B 直接通信，但只能连两个。

**veth + br0**：把多个 veth 的一端统一插到 br0 网桥上，实现多容器互通。br0 同时是主机的虚拟网卡，可配 IP 当网关。

> Docker 默认 bridge 模式就是这套架构。

### 3.3 完整路径：容器包怎么走到外网？

以 LQ1（IP 10.0.0.1）ping 8.8.8.8 为例：

```
LQ1（10.0.0.1）
  → 查路由表（外网 → 交给网关 10.0.0.254）
  → veth0 → br0（10.0.0.254，网关）
  → 主机内核路由决策（外网 → 走 eth0）
  → IP 转发开关（net.ipv4.ip_forward=1）
  → iptables NAT（MASQUERADE，换公网 IP）
  → eth0 → 路由器 → 外网（8.8.8.8）
```

三层路由关系：

| 层级 | 谁的路由表 | 决定什么 |
|-----|-----------|---------|
| 网络空间层 | LQ1/LQ2 各自 | "我的包先交给谁" |
| 二层（网桥） | br0 无路由表 | 纯 MAC 交换，不认 IP |
| 主机层 | 主机全局 | "包最终从哪张网卡出去" |

**核心**：br0 是"传达室窗口"（收包转交），真正做三层转发的是主机内核路由子系统 + IP 转发 + NAT。

### 3.4 与 Docker 的关联

```bash
# 默认独立 Network Namespace
docker run --rm ubuntu:22.04 ip link
# 预期输出: 只有 lo

# --network host：直接使用宿主机网络栈（高危）
docker run --rm --network host ubuntu:22.04 ip link
# 预期输出: 显示宿主机所有网卡
```

**⚠️ `--network host`** 让容器绑定宿主机端口、抓包所有流量。

### 3.5 实验 A：veth 直连

```bash
# 1. 创建两个 Network Namespace
sudo ip netns add ns0
sudo ip netns add ns1

# 2. 创建 veth pair，两端分别插入
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth0 netns ns0
sudo ip link set veth1 netns ns1

# 3. 配置 IP 并启动
sudo ip netns exec ns0 ip addr add 10.0.0.1/24 dev veth0
sudo ip netns exec ns0 ip link set veth0 up
sudo ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth1
sudo ip netns exec ns1 ip link set veth1 up

# 4. 验证互通
sudo ip netns exec ns0 ping 10.0.0.2

# 5. 清理（注意：veth0 已在 ns0 中，需用 -n 指定命名空间）
sudo ip -n ns0 link del veth0
```

### 3.6 实验 B：veth + br0 + NAT 出网

⚠️ **实验前必读**：会修改系统网络配置和 iptables。请在隔离环境执行，结束后务必清理，然后一定要准备好手机，防止操作断网没招了

```bash
# 0. 确认物理网卡名称（将下文 eth0 替换为实际名称）
ip link show

# 1. 创建 br0 网桥并配置网关
sudo ip link add br0 type bridge
sudo ip addr add 10.0.0.254/24 dev br0
sudo ip link set br0 up

# 2. 创建 veth pair，一端插命名空间，一端插网桥
sudo ip link add veth0-ns0 type veth peer name veth0-br
sudo ip link set veth0-ns0 netns ns0
sudo ip link set veth0-br master br0
sudo ip link set veth0-br up

sudo ip link add veth1-ns1 type veth peer name veth1-br
sudo ip link set veth1-ns1 netns ns1
sudo ip link set veth1-br master br0
sudo ip link set veth1-br up

# 3. 配置命名空间 IP
sudo ip netns exec ns0 ip addr add 10.0.0.1/24 dev veth0-ns0
sudo ip netns exec ns0 ip link set veth0-ns0 up
sudo ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth1-ns1
sudo ip netns exec ns1 ip link set veth1-ns1 up

# 4. 配置默认网关
sudo ip netns exec ns0 ip route add default via 10.0.0.254
sudo ip netns exec ns1 ip route add default via 10.0.0.254

# 5. 开启 IP 转发 + NAT
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 ! -o br0 -j MASQUERADE

# 6. 验证出网
sudo ip netns exec ns0 ping 8.8.8.8

# 7. 抓包验证 NAT（eth0 替换为实际网卡名）
sudo tcpdump -i eth0 icmp -n
# 另一终端：sudo ip netns exec ns0 ping 8.8.8.8

# 8. 清理（务必执行！）
sudo iptables -t nat -D POSTROUTING -s 10.0.0.0/24 ! -o br0 -j MASQUERADE
sudo ip netns del ns0
sudo ip netns del ns1
sudo ip link del br0
sudo ip link del veth0-br 2>/dev/null
sudo ip link del veth1-br 2>/dev/null
如果上面的还是删不了，执行
sudo ip link show | grep -E "veth.*br" | awk -F: '{print $2}' | xargs -r sudo ip link del
```

### 3.7 踩坑记录：忘了 `ip_forward`

实验 B 跑完后，ns0 能 ping 通 ns1，但 ping 外网不通。

排查链：
- `ip route` 正常 ✓
- `iptables -t nat -L` 有 MASQUERADE 规则 ✓
- `ping 8.8.8.8` 没回包 ✗

最后发现 `net.ipv4.ip_forward=0`。主机默认只管自己的包，不管"别人的"。

```bash
# 错误：IP 转发未开启
sysctl net.ipv4.ip_forward
# 预期输出: net.ipv4.ip_forward = 0
# ping 8.8.8.8 无回包

# 正确：开启转发
sudo sysctl -w net.ipv4.ip_forward=1
# ping 8.8.8.8 正常
```

**教训**：Network Namespace 隔离了协议栈，但"出网"是主机的责任。IP 转发是主机当"快递中转站"的许可证，不开就等于拒收。

### 3.8 常见错误排查

| 现象 | 原因 | 解决 |
|------|------|------|
| `ip link del veth0` 报错 | veth 已移到 ns 中 | `ip -n ns0 link del veth0` |
| `ping 8.8.8.8` 不通 | IP 转发未开启 | `sysctl -w net.ipv4.ip_forward=1` |
| 外网不回包 | NAT 规则缺失 | 检查 `iptables -t nat -L` |
| tcpdump 报错接口不存在 | 网卡名不是 eth0 | `ip link show` 查看实际名称 |

### 3.9 从实验到生产：Docker 在背后做了什么？

| 你手动做的 | Docker 自动做的 |
|-----------|----------------|
| `ip netns add` | 创建容器网络空间 |
| `ip link add veth...` | 创建 veth，一头插容器，一头插 docker0 |
| `ip addr add` | 分配 IP（来自 docker0 子网） |
| `ip route add default` | 设置默认网关为 docker0 IP |
| `iptables -t nat ...` | 自动配置 MASQUERADE |

**理解了这个实验，你就理解了 Docker 网络 80% 的原理。**

### 3.10 小结

| 属性 | 值 |
|------|-----|
| 隔离内容 | 网卡、路由表、防火墙规则、端口 |
| 隔离方式 | **独立协议栈**（独立的 `struct net`） |
| 是否需要 root | 创建需 root |
| Docker 默认 | ✅ 独立（bridge 模式） |
| 危险参数 | `--network host` |

> **金句**：Network Namespace 给每个进程发了一间独立的网络办公室。办公室之间靠 veth 网线连到 br0 交换机，出网靠主机内核当快递中转站，NAT 换马甲。

## 四、Mount Namespace：文件系统的挂载视图

### 4.1 一句话理解

Mount Namespace 让每个进程拥有自己的"挂载视图"（挂载表），视图独立但底层文件系统可能共享。容器在此基础上多了一步——不仅换视图，还换根目录（`pivot_root`）。

技术本质：内核为每个 Mount Namespace 维护独立的挂载树副本（`struct mnt_namespace`）。创建时复制当前挂载树，之后各自独立演化。但底层文件系统是共享还是独立，取决于挂载的是同一个设备还是不同设备。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/4890a51877114312b63b2bcc0930f29e.jpeg)


### 4.2 目录是门，文件系统是房间

**目录本身不存储数据，它只是一扇门。** 门后面是什么房间，取决于你在这扇门上"挂载"了什么。

```bash
# 造一扇门
mkdir /mnt/door
ls /mnt/door
# 预期输出: （空）

# 在门后挂一个房间
mount -t tmpfs tmpfs /mnt/door
echo "我在房间里" > /mnt/door/msg.txt
cat /mnt/door/msg.txt
# 预期输出: 我在房间里

# 拆下房间，门又变空了
umount /mnt/door
cat /mnt/door/msg.txt
# 预期输出: 报错：没有那个文件或目录
```

> **文件系统必须挂载到目录上才能使用。目录是唯一的入口。**

### 4.3 什么是 Mount Namespace

Linux 内核的 Mount Namespace 让每个进程拥有自己的"房间清单"（挂载表），进程之间互相独立。

同一个门 `/mnt/data`，进程 A 的清单上写着通向房间 X，进程 B 的清单上可能写着通向房间 Y，或者什么都没写。

**不是房间变了，是进程手里的清单不同。**

创建新挂载视图时，内核会复制一份当前的房间清单给新视图。之后两边的清单各自更新，互不影响。

> ⚠️ **重要**：Mount Namespace 隔离的是"房间清单"（挂载表），不是房间里的文件内容。如果两个视图都挂载了同一个房间（比如都指向同一个硬盘分区），修改文件会互相看到。

### 4.4 三个视图实验

**第一视图（宿主机）**：所有进程共享同一份房间清单。

**第二视图（独立清单）**：

```bash
sudo unshare --mount --propagation private /bin/bash
```

内核复制了一份房间清单给你。从此你的清单和宿主机的清单各自独立。

> `--propagation private` 显式设置挂载传播为 private，确保挂载事件不会穿透 Namespace 边界。这是手写容器时最容易忽略的安全点。

```bash
# 你在第二视图
mkdir /mnt/room2
mount -t tmpfs tmpfs /mnt/room2
echo "第二视图的文件" > /mnt/room2/secret.txt

# 退出第二视图，回到第一视图
exit

# 第一视图看不到
ls /mnt/room2
# 预期输出: （空）

mount | grep room2
# 预期输出: （无输出，确认隔离生效）
```

**第三视图（嵌套隔离）**：在第二视图内部再复制一份清单。

```bash
# 在第二视图里
sudo unshare --mount --propagation private /bin/bash
mkdir /mnt/room2/room3
mount -t tmpfs tmpfs /mnt/room2/room3
echo "第三视图的文件" > /mnt/room2/room3/deep.txt
```

| 查看内容 | 第一视图 | 第二视图 | 第三视图 |
|----------|----------|----------|----------|
| `/mnt/room2/secret.txt` | ❌ | ✅ | ✅（复制时带过来的） |
| `/mnt/room2/room3/deep.txt` | ❌ | ❌ | ✅ |
| 修改 `/etc/hosts` | ✅ | ✅ | ✅（底层共享） |

> **规律**：每多一个视图，就多一层清单隔离。外层看不到内层的新挂载，除非显式进入内层。

### 4.5 挂载传播：为什么需要 `--propagation private`？

Linux 挂载点有四种传播类型：

| 传播类型 | 行为 | 典型场景 |
|---------|------|---------|
| **shared** | 挂载事件双向传播 | 系统默认根挂载 |
| **slave** | 挂载事件单向传播（只接收） | 某些容器场景 |
| **private** | 挂载事件不传播 | 完全隔离的容器 |
| **unbindable** | 不可绑定挂载 | 防止递归挂载问题 |

**为什么实验要加 `--propagation private`？**

因为宿主机的根目录 `/` 默认是 shared 传播。如果你在 Mount Namespace 里挂载了新目录，这个事件会传播回宿主机——宿主机也会看到这个挂载！这就破坏了隔离。

```bash
# 错误示范（不加 --propagation private）
sudo unshare --mount /bin/bash
mount -t tmpfs tmpfs /mnt/leak
# 宿主机也会看到 /mnt/leak！因为根挂载是 shared
```

```bash
# 正确示范
sudo unshare --mount --propagation private /bin/bash
mount -t tmpfs tmpfs /mnt/isolated
# 宿主机看不到，隔离生效
```

> **Docker 的默认行为**：Docker 创建容器时会自动处理挂载传播，将容器的根设置为 private，所以你不需要手动操心。但手写容器时必须注意这一点。

### 4.6 从视图到容器：换根目录

前面的视图只隔离了清单，没换房间。第二视图、第三视图的根目录 `/` 里的内容还是和第一视图一样。

**容器需要**：不仅清单独立，根目录 `/` 也要换成新房间。

容器内部三步：
1. 创建 Mount Namespace（第二视图，清单隔离）
2. 挂载容器镜像文件系统（新房间）
3. 用 `pivot_root` 把根目录 `/` 指向新房间（换根操作）

```
容器的文件系统隔离 = Mount Namespace（清单隔离）+ 新文件系统 + pivot_root（换根）
```

**pivot_root vs chroot**：
- `chroot` 只是改当前进程的根目录，但 `cd ..` 可以逃逸
- `pivot_root` 修改 mount namespace 内调用进程的 rootfs，把当前 rootfs 移到 `put_old` 目录，`new_root` 变成新的 `/`。`cd ..` 不会逃逸，因为当前工作目录已被重新解析到新的 rootfs 下

| 攻击场景 | 文件系统隔离机制（标准容器） |
|----------|----------------------------|
| 容器内读取宿主机 `/etc/shadow` | ✅ 换根后看不到 |
| 容器内修改宿主机文件 | ✅ 清单隔离保护 |


#### 实验：chroot 逃逸 vs pivot_root 安全

```bash
# 1. 创建 jail 目录和文件
mkdir -p /tmp/jail/bin
cp /bin/bash /tmp/jail/bin/
cp /bin/ls /tmp/jail/bin/
mkdir -p /tmp/jail/lib /tmp/jail/lib64
# 复制必要的动态库（略，实际需 ldd 检查）

# 2. chroot 逃逸演示
sudo chroot /tmp/jail /bin/bash
# 在 chroot 内
cd ..
cd ..
ls /
# 预期输出: 看到宿主机的根目录！chroot 只改了进程的根目录指针，没有隔离 mount namespace，当前工作目录的解析逻辑仍可逃逸

# 3. pivot_root 安全演示（需要 Mount Namespace）
sudo unshare --mount --propagation private /bin/bash
mkdir /tmp/newroot /tmp/oldroot
mount -t tmpfs tmpfs /tmp/newroot
# 在 newroot 里准备最小根文件系统...
pivot_root /tmp/newroot /tmp/oldroot
cd /
ls /tmp/oldroot
# 预期输出: 看到旧的根被挂到了 /tmp/oldroot，但 cd .. 不会逃逸
# 因为整个 mount namespace 的 rootfs 已经切换
```

**关键区别**：`chroot` 只修改了当前进程的根目录指针，`pivot_root` 修改了整个 mount namespace 的根文件系统。`cd ..` 在 `pivot_root` 后仍然指向新的根，不会逃逸。

> **注意**：通过 volume 显式挂载宿主机目录，容器可以访问。这是配置行为，不是隔离机制的绕过。

### 4.7 与 Docker 的关联

```bash
# Docker 默认独立 Mount Namespace
docker run --rm ubuntu:22.04 ls /
# 预期输出: 显示 Ubuntu 镜像的根目录

# 共享宿主机文件系统（危险）
docker run --rm -v /:/host ubuntu:22.04 ls /host
# 预期输出: 显示宿主机的根目录

# 只读挂载
docker run --rm -v /:/host:ro ubuntu:22.04 cat /host/etc/shadow
# 预期输出: 可以读取，但不能修改
```

**⚠️ 安全提示**：`-v /:/host` 这种挂载直接把宿主机根目录塞进容器，是容器逃逸的最短路径之一。


### 4.8 踩坑记录：`--propagation private` 不是默认的

第一次写 Mount Namespace 实验脚本时，我没加 `--propagation private`：

```bash
sudo unshare --mount /bin/bash
mount -t tmpfs tmpfs /mnt/leak
```

然后在宿主机终端执行 `mount | grep leak`，居然看到了 `/mnt/leak`！

**为什么？** 因为宿主机根目录 `/` 的默认传播类型是 `shared`。Mount Namespace 里的事件会穿透回宿主机。

```bash
# 错误：挂载事件泄漏到宿主机
sudo unshare --mount /bin/bash
mount -t tmpfs tmpfs /mnt/leak
# 宿主机：mount | grep leak → 能看到！

# 正确：显式设置 private，切断传播
sudo unshare --mount --propagation private /bin/bash
mount -t tmpfs tmpfs /mnt/isolated
# 宿主机：mount | grep isolated → 看不到
```

**教训**：手写容器时必须显式处理挂载传播。Docker 自动做了这件事，但你自己写的时候不会自动发生。

### 4.9 小结

| 属性 | 值 |
|------|-----|
| 隔离内容 | 文件系统挂载点 |
| 隔离方式 | **挂载视图**（复制挂载树） |
| 是否需要 root | 创建需 root |
| Docker 默认 | ✅ 独立 |
| 危险参数 | `-v /:/host` |

> **金句**：Mount Namespace 让每个进程拥有自己的"房间清单"。容器比纯 Namespace 多了一层：不仅"换清单"，还"换房间"。

---

## 五、IPC Namespace：全局表上的归属标签

### 5.1 一句话理解

IPC Namespace 是**逻辑隔离**。内核在全局 IPC 对象表上增加了一列"归属哪个 Namespace"，查找时按 Namespace 过滤。容器内只能看到自己的过滤视图，宿主机 root 能看到所有 Namespace 的 IPC 对象。

技术本质：内核为每个 IPC Namespace 分配独立的 `struct ipc_namespace`，包含独立的信号量数组、消息队列和共享内存段的管理结构。但这些数据结构物理上存在于内核全局空间中，内核通过 `ipc_namespace` 指针区分归属。
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/53b8c2c9cc1f4b769f9e9b6e24697015.jpeg)


### 5.2 动手实验

```bash
# 步骤1：宿主机创建共享内存
ipcmk -M 1024
# 预期输出: Shared memory id: 0

# 查看
ipcs -m
# 预期输出: ------ Shared Memory Segments --------
# 预期输出: key        shmid      owner      perms      bytes      nattch     status
# 预期输出: 0x00000000 0          root       644        1024       0

# 步骤2：创建新的 IPC Namespace
sudo unshare --ipc /bin/bash

# 步骤3：在 Namespace 内查看共享内存
ipcs -m
# 预期输出: （空！看不到宿主机的）

# 步骤4：在 Namespace 内创建新的共享内存
ipcmk -M 512
# 预期输出: Shared memory id: 0

ipcs -m
# 预期输出: key        shmid      owner      perms      bytes      nattch     status
# 预期输出: 0x00000000 0          root       644        512        0
```

### 5.3 验证隔离边界

```bash
# 宿主机再看（另一个终端）
ipcs -m
# 预期输出: ------ Shared Memory Segments --------
# 预期输出: key        shmid      owner      perms      bytes      nattch     status
# 预期输出: 0x00000000 0          root       644        1024       0          ← 宿主机的
# 预期输出: 0x00000000 0          root       644        512        0          ← 容器的？！
```

**关键发现**：`shmid` 是 Namespace 内的本地编号。宿主机 root 通过 `/proc/sysvipc/shm` 等接口能看到所有 Namespace 的 IPC 对象（因为内核全局表中包含 ns 指针字段），但普通容器进程只能看到自己的过滤视图。

### 5.4 内核实现：全局表 + 过滤条件

IPC Namespace 的内核实现基于 `struct ipc_namespace`。每个 IPC Namespace 拥有独立的 System V 信号量数组、消息队列和共享内存段的管理结构。

但这些数据结构**不是物理隔离的**——它们都存在于内核全局空间中，内核通过 `ipc_namespace` 指针来区分归属：

```
内核全局 IPC 对象表（简化示意）：
┌─────────────┬─────────────────────┬─────────────┐
│ key         │ IPC ns 指针地址      │ 数据地址     │
├─────────────┼─────────────────────┼─────────────┤
│ "nginx.stats"│ 0xffff...a1b2（LQ1）│ 0xabc...    │
│ "nginx.stats"│ 0xffff...c3d4（LQ2）│ 0xdef...    │
└─────────────┴─────────────────────┴─────────────┘

LQ1 查找 "nginx.stats"
  └── 内核确认 LQ1 的 IPC ns 指针 = 0xffff...a1b2
  └── 匹配 (key="nginx.stats", ns=0xffff...a1b2)
  └── 返回 LQ1 的数据
```

### 5.5 独立数据结构 vs 逻辑隔离

| 维度 | UTS Namespace | IPC Namespace |
|---|---|---|
| **隔离方式** | 独立数据结构 | 全局表 + 过滤条件 |
| **物理位置** | 容器独立空间 | 宿主机内核共享 |
| **同名冲突** | 不存在（各自独立） | 存在但通过指针区分 |

> **为什么区分这两种隔离方式很重要？**
> 
> 因为这决定了突破隔离的方式不同：
> - UTS：没有"全局表"可以遍历，宿主机 root 也无法直接看到所有 UTS Namespace 的内容
> - IPC：宿主机 root 可以通过内核接口遍历所有 IPC 对象，只是按 Namespace 过滤显示

### 5.6 与 Docker 的关联

```bash
# Docker 默认独立 IPC Namespace
docker run --rm ubuntu:22.04 ipcs -m
# 预期输出: （空）

# 共享宿主机 IPC（危险）
docker run --rm --ipc=host ubuntu:22.04 ipcs -m
# 预期输出: 显示宿主机的所有 IPC 对象

# 两个容器共享 IPC Namespace
docker run --rm --ipc=container:LQ1 ubuntu:22.04 ipcs -m
# 预期输出: 显示 LQ1 的 IPC 对象
```

`--ipc=host` 会让容器直接访问宿主机的共享内存、消息队列、信号量，是容器逃逸的高危路径之一。

### 5.7 小结

| 属性 | 值 |
|------|-----|
| 隔离内容 | 共享内存、消息队列、信号量 |
| 隔离方式 | **逻辑隔离**（全局表 + Namespace 指针过滤） |
| 是否需要 root | 创建需要 |
| Docker 默认 | ✅ 独立 |
| 危险参数 | `--ipc=host` |

> **金句**：IPC 是内核在全局表上贴了个"属于谁"的标签，通过 Namespace 指针区分归属。不是切空间，是贴标签。

---

## 六、USER Namespace：身份翻译器

### 6.1 一句话理解

USER Namespace 是**身份翻译器**，不是权限提升工具。容器内看到的 `root`（UID 0），在内核眼里是映射后的普通用户（如 UID 1000）。内核始终用**真实 UID** 做权限检查。

技术本质：USER Namespace 通过 `uid_map` 和 `gid_map` 建立容器内外 UID/GID 的映射关系。`struct user_namespace` 是核心数据结构。映射在内核权限检查时实时转换。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/882278c5b9e54ac49bcf8bb577bfd4ab.jpeg)

### 6.2 两层视角

容器内进程看到的自己，和内核看到的它，是两个完全不同的身份：

| 视角 | 看到的身份 | 实际权限 |
|------|-----------|---------|
| 容器内 | `root`（UID 0） | 以为自己有最高权限 |
| 宿主机内核 | 普通用户（如 UID 1000） | 按普通用户做权限检查 |

内核始终用**真实 UID** 执行权限判定，容器内的 root 只是**映射后的身份**。

### 6.3 动手实验（普通用户即可！）

```bash
# 关键：USER Namespace 不需要 sudo！
# 前置检查：确保内核允许非特权 USER Namespace
sysctl kernel.unprivileged_userns_clone
# 应输出 1。如果输出 0，执行：
# sudo sysctl -w kernel.unprivileged_userns_clone=1

unshare --user --map-root-user --fork /bin/bash

# 容器内视角
whoami
# => root
id
# => uid=0(root) gid=0(root) groups=0(root)

# 创建文件
touch /tmp/userns-test.txt
ls -l /tmp/userns-test.txt
# => -rw-r--r-- 1 root root ...（容器内看是 root）
```

```bash
# 宿主机视角（另一个终端）
ps aux | grep bash
# 预期输出: 1000     2847  ...  /bin/bash  （实际 UID 1000）

ls -l /tmp/userns-test.txt
# 预期输出: -rw-r--r-- 1 1000 1000 ...（宿主机看是 UID 1000）
```


### 6.4 验证隔离边界：映射表

```bash
# 在容器内查看映射
cat /proc/self/uid_map
# 预期输出: 0       1000          1
# 格式：容器内UID  宿主机UID  映射范围

# 解读：容器内的 UID 0 → 宿主机的 UID 1000
#      映射范围 1，表示只映射了 1 个 UID
```

#### 为什么映射表需要"外面的人"来写？

这是 USER Namespace 最核心的安全设计：**只有 Namespace 外的 root 才能给 Namespace 内的 root 发面具。**

```
父进程（宿主机，UID 0）
  ├── fork() → 子进程
  │
  ├── unshare(CLONE_NEWUSER) → 子进程进入新 USER Namespace
  │   └── 子进程在新 NS 内是 UID 0（但内核知道它是"假的"）
  │
  └── 父进程写 /proc/[子进程pid]/uid_map
      └── "0 1000 1" → 容器内 UID 0 映射到宿主机 UID 1000
```

**如果容器自己能写 `uid_map`**，它就可以写 `"0 0 1"`（映射到宿主机 root），面具就成真了。

内核的设计是：
- 创建 USER Namespace 的进程，在新 Namespace 内获得所有 capabilities
- 但写 `uid_map` 的权限在**父进程**手里（父进程在旧 Namespace 中）
- 父进程通常是容器运行时（runc/dockerd），由系统管理员控制

> 这就是为什么 Docker 默认不启用 USER Namespace：启用后需要处理镜像文件权限（镜像内文件所有者 UID 0 映射后可能无法读取），兼容性成本高。



### 6.5 安全价值：逃逸了也是普通人

```bash
# 模拟逃逸：在容器内尝试读取宿主机 /etc/shadow
cat /etc/shadow
# 预期输出: cat: /etc/shadow: Permission denied

# 为什么？因为内核看到的是 UID 1000，不是 root
```

**核心结论**：即使容器进程通过某种方式回到了宿主机 PID Namespace（逃逸），内核记住的是它的**真实 UID 1000**，不是 root。对 `/etc/shadow` 没权限，创建的文件所有者也是 UID 1000。

> **但注意**：如果容器通过 volume 挂载了宿主机目录，且该目录对映射后的 UID 可写，容器仍然可以修改宿主机文件。USER Namespace 防的是"权限提升"，不是"文件访问"。

### 6.6 ⚠️ 危险配置警告

```bash
# 以下配置是危险的：把容器 root 映射到宿主机 root
echo "0 0 1" > /proc/self/uid_map  # 容器内 UID 0 → 宿主机 UID 0

# Docker 默认不会这样做，但某些"特权容器"场景可能接近这个效果
docker run --rm --privileged ubuntu:22.04 id
# 预期输出: uid=0(root) gid=0(root)  ← 这是真正的 root！
```

**`--privileged` 和 USER Namespace 是两种互斥的安全模型**：
- `--privileged`：容器内是真正的 root，没有 USER Namespace 保护
- USER Namespace：容器内是"映射 root"，逃逸无危害

> 注意：技术上 `--privileged` 和 `--userns-remap` 可以同时指定，但结果混乱且危险，应避免。

### 6.7 为什么 Docker 默认不开 USER Namespace？

```bash
# 检查 Docker 是否启用 USER Namespace
docker info | grep userns
# 预期输出: 通常为空，表示未启用
```

原因：**文件系统权限冲突**。

如果镜像里的文件所有者是 root（UID 0），而容器 root 映射到 UID 100000，那容器内读取这些文件时，内核检查的是 UID 100000 的权限——大概率没有读权限。

解决方案：
- 方案A：镜像构建时把所有文件改为非 root 拥有（最佳实践）
- 方案B：Docker 启用 `--userns-remap=default`，自动映射到一个子 UID 范围

```bash
# 启用 USER Namespace（需修改 /etc/docker/daemon.json）
{
  "userns-remap": "default"
}
```

### 6.8 容器内"创造"UID 的真相

容器内执行 `useradd` 创建新用户，这个 UID 在宿主机视角：
- 如果没在 uid_map 里映射 → 就是 `nobody`（65534）
- 创建的文件所有者 → 实际是 nobody
- 这个"用户"在宿主机上不存在

**创造出来的全是空权限**，建立在容器自己拥有的普通权限之上，不可能靠近真正的 root。

> 容器在这个空间里**注定只能是普通权限**。这不是限制它的地盘，而是限制它以为自己有的权限。

### 6.9 小结

| 属性 | 值 |
|------|-----|
| 隔离内容 | UID/GID 身份映射 |
| 隔离方式 | **身份映射**（`uid_map`/`gid_map` 转换） |
| 是否需要 root | **不需要！** 普通用户可创建 |
| Docker 默认 | ❌ 不启用（兼容性原因） |
| 安全价值 | 逃逸后仍是普通用户权限 |
| 危险配置 | 映射到宿主机 UID 0、`--privileged` |

> **金句**：USER Namespace 给容器内的 root 戴了一副"我是 root"的面具。面具摘了（逃逸），还是普通用户。不是更高级，也不是更低级，就是同一个普通用户。

---

## 七、六面墙，一栋楼

| Namespace | 隔离什么 | 隔离方式 | 难度 | 一句话 |
|-----------|---------|---------|------|--------|
| **UTS** | hostname | 独立数据结构 | ⭐ | 各改各的 hostname |
| **PID** | 进程编号 | 视图隔离 | ⭐⭐ | 独立的本地进程表 |
| **Network** | 网络资源 | 独立协议栈 | ⭐⭐ | 各走一路，出网靠主机中转 |
| **Mount** | 文件系统挂载 | 挂载视图 | ⭐⭐⭐ | 换清单，容器还换根 |
| **IPC** | 进程间通信 | 逻辑隔离 | ⭐⭐⭐ | 贴标签，不是切空间 |
| **USER** | 用户身份 | 身份映射 | ⭐⭐⭐⭐⭐ | 面具摘了还是普通人 |

> Linux 内核后续还加入了 **Cgroup Namespace**（隔离 cgroup 视图，内核 4.6+）和 **Time Namespace**（隔离系统时间，内核 5.6+）。这两个 Namespace 在容器生态中应用尚不广泛，但了解它们的存在有助于完整理解 Linux 的隔离体系。

---

## 八、组合实验：真实容器的隔离栈

```bash
# 同时启用 UTS + IPC + USER + PID + Network + Mount
# 这就是 Docker 默认的隔离配置（除了 USER）

unshare --uts --ipc --user --map-root-user --pid --fork --mount-proc --net --mount /bin/bash

# 验证各层隔离
hostname container-full
hostname
# 预期输出: container-full

id
# 预期输出: uid=0(root) gid=0(root)  ← USER Namespace 映射

ps aux
# 预期输出: PID 1 是 bash  ← PID Namespace 隔离

ip link
# 预期输出: 只有 lo  ← Network Namespace 隔离

ipcs -m
# 预期输出: （空）← IPC Namespace 隔离

ls /
# 预期输出: 标准 Linux 目录  ← Mount Namespace + pivot_root
```

---

## 九、性能：Namespace 不是免费的，但也没那么贵

根据 2026 年 2 月发表的系统性测量研究（详见参考文献[1]），Linux namespace 创建在总启动时间中仅占 **8–10 ms（<1.5%）**。容器启动的主要开销来自运行时层和存储层，而非 Namespace 本身。

| Namespace 类型 | 创建开销 | 主要开销来源 |
|---------------|---------|-------------|
| UTS | 最小 | 分配 `struct uts_namespace` |
| IPC | 较小 | 分配 `struct ipc_namespace` + 初始化信号量数组 |
| PID | 中等 | 分配 pid 哈希表 + 挂载 proc |
| Network | 较大 | 分配 net_device + 路由表 + iptables 规则 |
| Mount | 最大 | 复制挂载树（与挂载点数量成正比） |
| USER | 较小 | 分配 `struct user_namespace` + uid_map 验证 |

**/proc 遍历过滤开销**：PID Namespace 隔离后，`ps` 遍历 `/proc` 时内核需要检查每个进程的 Namespace 归属。宿主机进程数越多（>1000），过滤开销越明显。

---

## 十、安全底线

> **Namespace 解决的是"视图隔离"，不是"权限控制"**

| 机制 | 解决什么问题 | 不解决什么问题 |
|------|-------------|---------------|
| Namespace | 进程"看不到"外面的资源 | 进程"不能操作"外面的资源 |
| Capabilities | 细粒度权限控制 | 系统调用过滤 |
| Seccomp | 限制可使用的系统调用 | 文件访问控制 |
| AppArmor/SELinux | 强制访问控制（MAC） | 资源限制 |
| cgroups | 资源限制（CPU/内存/IO） | 安全隔离 |

**真正的安全边界需要 Capabilities + Seccomp + AppArmor/SELinux 组合。**

**USER Namespace 是最后一道防线**：即使逃逸，也是普通人。

---

## 附录 A：runc spec 中的 Namespace 配置

```json
{
  "linux": {
    "namespaces": [
      {"type": "pid"},
      {"type": "network"},
      {"type": "ipc"},
      {"type": "uts"},
      {"type": "mount"}
    ],
    "uidMappings": [
      {
        "containerID": 0,
        "hostID": 1000,
        "size": 1
      }
    ],
    "gidMappings": [
      {
        "containerID": 0,
        "hostID": 1000,
        "size": 1
      }
    ]
  }
}
```

生成和验证：
```bash
runc spec
cat config.json | jq '.linux.namespaces'
cat config.json | jq '.linux.uidMappings'
runc run mycontainer
```

---

## 附录 B：危险参数速查表

| 参数 | 风险等级 | 影响 | 何时可用 |
|------|---------|------|---------|
| `--uts=host` | 🔴 高危 | 容器改 hostname 影响宿主机 | 几乎不用 |
| `--pid=host` | 🔴 高危 | 容器看到/操作宿主机所有进程 | 调试时短暂使用 |
| `--network host` | 🔴 高危 | 容器绑定宿主机端口，可抓包所有流量 | 性能敏感型应用 |
| `--ipc=host` | 🟠 中危 | 容器访问宿主机共享内存 | 遗留应用兼容 |
| `-v /:/host` | 🔴 高危 | 容器直接读写宿主机根目录 | 几乎不用 |
| `--privileged` | 🔴 极高危 | 容器获得真正 root + 所有 capabilities | 绝对避免 |

---

## 附录 C：术语表

| 术语 | 含义 |
|------|------|
| 视图隔离 | 内核维护了独立的分配空间/协议栈，给你看不同的表 |
| 逻辑隔离 | 全局表 + 过滤条件 |
| 身份映射 | UID/GID 映射表转换 |
| 挂载视图 | 每个进程独立的挂载点列表 |
| 独立协议栈 | 完整的网络环境隔离 |
| pivot_root | 切换 mount namespace 的 rootfs |
| 独立数据结构 | 内核分配独立的数据结构副本 |

---

## 附录 D：常见误区速查表

| ❌ 错误认知 | ✅ 正确理解 |
|-----------|-----------|
| "PID Namespace 切断了进程间的所有联系" | "是视图隔离，同一 Namespace 内进程完全互通" |
| "Mount Namespace 隔离了文件内容" | "隔离的是挂载点清单，底层文件可能共享" |
| "USER Namespace 让容器更安全" | "默认不启用，启用后需处理文件权限冲突" |
| "IPC Namespace 是真隔离" | "是逻辑隔离，基于全局表过滤" |
| "Network Namespace 完全隔离网络" | "veth 和网桥让隔离有条件，出网依赖主机" |
| "USER Namespace 需要 root 才能创建" | "普通用户即可创建，需内核支持" |
| "Mount Namespace 创建后自动完全隔离" | "默认传播类型可能是 shared，需显式设置 private" |
| "br0 做三层转发" | "br0 是二层交换机，三层转发由主机内核路由子系统处理" |

---

---

**系列导航**：
- 本文：Namespace 隔离机制（六面墙）
- 下一篇：cgroups 资源限制（配重块）
- 计划：Capabilities 权限管理 → Seccomp 系统调用过滤 → AppArmor/SELinux 强制访问控制

---

## 最后

Namespace 不是魔法，是内核给进程换了一副眼镜。六副镜片，六种视角。

**但记住：眼镜换得再好，也挡不住有人把门踹开。**

Namespace 隔离的是"看到什么"，Capabilities、Seccomp、AppArmor 才管"能做什么"。USER Namespace 是最后一道保险——即使有人踹开门逃出来，他也只是个普通人。

这副眼镜能遮住视线，但遮不住贪婪。

> 如果这篇文章对你有帮助，欢迎 在评论区留下你的实验结果。我踩过的坑，希望你不用重踩一遍，尤其是断网，可能是我电脑的原因，但是最好还是提防，核心原文是我自己对原理的了解，AI修改文章建立在我了解的基础上，如果有错误可以纠正，正在持续更新高质量文章.......

---





