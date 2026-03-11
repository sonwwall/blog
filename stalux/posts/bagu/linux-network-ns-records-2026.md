---
title: 第七节：Linux中的网络指令：如何查看一个域名有哪些NS记录？
abbrlink: linux-network-ns-records-2026
date: 2026-03-11T10:21:18
updated: 2026-03-11T10:38:32
tags:
  - 操作系统
  - 学习
  - 八股
categories:
  - 操作系统
desc: 第七节：Linux中的网络指令：如何查看一个域名有哪些 NS 记录？
---

# 第七节：Linux中的网络指令：如何查看一个域名有哪些NS记录？

这一节主要围绕 5 个点展开：

1. 远程操作相关命令有哪些
2. 如何查看本机网络接口和网络状态
3. 如何做基础网络测试
4. Linux 中常见的 DNS 查询命令有哪些
5. 如何查看一个域名有哪些 `NS` 记录

## 1. 远程操作指令

这一部分先看两类很常见的远程操作命令：`ssh` 和 `scp`。

### 1.1 `ssh`：远程登录

`ssh` 用来远程登录另一台 Linux 机器。

常见写法：

```bash
ssh user@host
```

例如：

```bash
ssh ramroll@u1
```

它的作用可以直接理解成：

- 在本机打开一个安全的远程终端会话
- 登录到目标主机
- 在目标主机上执行命令

这里的 `u1` 可以是：

- 一个域名
- 一个主机名
- 一个 IP 地址

如果不是公网 DNS 解析出来的名字，也可能是通过 `/etc/hosts` 做的本地映射。

### 1.2 `/etc/hosts`：本地主机名映射

例如可以通过下面的命令查看 `/etc/hosts`：

```bash
cat /etc/hosts
```

例如：

```text
127.0.0.1 localhost
127.0.1.1 u2
192.168.199.128 u1
```

这个文件的作用是：

- 手工维护“主机名 -> IP 地址”的映射
- 在 DNS 解析之前，系统通常会先参考本地配置
- 适合测试环境、小型局域网环境

所以 `ssh ramroll@u1` 能成功，未必说明 `u1` 一定是 DNS 记录，也可能只是 `/etc/hosts` 里配了映射。

### 1.3 `scp`：远程拷贝文件

`scp` 基于 SSH 协议，用来在本机和远程主机之间复制文件。

例如：

```bash
scp ~/a.txt ramroll@u2:/home/ramroll/a.txt
```

这个命令表示：

- 把本机的 `~/a.txt`
- 拷贝到远程主机 `u2`
- 保存为 `/home/ramroll/a.txt`

它的特点是：

- 语法简单
- 走 SSH，传输过程加密
- 适合直接传单个文件或少量目录

## 2. 查看本地网络状态

这一类命令主要包括 `ifconfig` 和 `netstat`。

### 2.1 `ifconfig`：查看网络接口信息

`ifconfig` 用来查看网络接口的基本状态。

例如：

```bash
ifconfig
```

常见可以看到：

- 网卡名称，例如 `ens33`
- `inet`：IPv4 地址
- `netmask`：子网掩码
- `broadcast`：广播地址
- `inet6`：IPv6 地址
- `ether`：MAC 地址
- 收发包统计

在虚拟机场景里，经常会看到虚拟网卡，这个理解是对的。  
虚拟机里的网卡，本质上是由虚拟化软件模拟出来的网络设备，对操作系统来说它仍然表现为普通网卡。

所以通过 `ifconfig`，我们最常做的是：

- 看当前机器 IP 是多少
- 看网卡是否启动
- 看有没有异常丢包

### 2.2 `netstat`：查看网络连接和端口状态

`netstat` 用来查看：

- 当前网络连接
- 监听端口
- 协议统计信息
- 路由信息

它背后和 socket 有关，但更准确地说，**它展示的是套接字对应的网络连接、监听状态和统计结果**，而不是简单理解成“查看 socket 文件”。

例如：

```bash
netstat
```

这个命令默认会输出很多内容，所以经常会配合筛选一起使用。

#### 查看 TCP 连接

```bash
netstat -t
```

或写成：

```bash
netstat -t tcp
```

常见字段含义：

- `Local Address`：本地地址和端口
- `Foreign Address`：远端地址和端口
- `State`：连接状态

例如输出中如果出现：

- 本地 `u1:ssh`
- 远端 `u2:48768`
- 状态 `ESTABLISHED`

这说明当前有一个已经建立好的 SSH TCP 连接。

#### 查看端口占用

例如：

```bash
sudo netstat -ntlp | grep 22
```

这个命令常用来排查某个端口是否被监听。

可以这样理解参数：

- `-n`：直接显示数字地址和端口，不做名字解析
- `-t`：只看 TCP
- `-l`：只看监听中的连接
- `-p`：显示对应进程

如果看到：

```text
0.0.0.0:22 ... LISTEN
```

说明当前机器正在监听 `22` 端口，也就是 SSH 服务通常已经启动。

#### 统计连接数量

再比如：

```bash
netstat | wc -l
```

这只是粗略统计输出行数，能大概感受当前连接信息多少，但并不适合做精确统计。

#### 如何查看正在 `TIME_WAIT` 状态的连接数量

先解释一下什么是 `TIME_WAIT`。

`TIME_WAIT` 是 TCP 连接断开过程中的一种状态，通常出现在**主动关闭连接的一方**。

它存在的主要目的有两个：

- 确保最后的确认报文如果丢失，还能有机会重新发送
- 让网络中残留的旧报文彻底过期，避免影响后续同样四元组的新连接

所以可以把它简单理解成：

- 连接虽然已经“逻辑上断开”
- 但内核不会立刻把这条连接记录完全删除
- 而是会保留一小段时间，进入 `TIME_WAIT`

如果面试里问：**如何查看处于 `TIME_WAIT` 状态的连接数量？**  
常见写法是：

```bash
netstat -ant | grep TIME_WAIT | wc -l
```

其中：

- `-a`：显示所有连接和监听端口
- `-n`：数字方式显示
- `-t`：只看 TCP

这条命令的含义是：

- 先列出所有 TCP 连接
- 过滤出状态为 `TIME_WAIT` 的连接
- 最后统计数量

如果想看具体是哪些连接处于 `TIME_WAIT`，可以先不加 `wc -l`：

```bash
netstat -ant | grep TIME_WAIT
```

## 3. 网络测试命令

### 3.1 `ping`：测试网络连通性和时延

`ping` 是最常用的网络测试命令之一。

例如：

```bash
ping www.lagou.com
```

它主要用来：

- 测试目标主机是否可达
- 观察往返时延
- 粗略判断网络是否稳定

`ping` 底层使用的是 `ICMP` 协议。  
如果目标写的是域名，那么在真正发包前，系统通常会先做一次 DNS 解析，把域名解析成 IP 地址。

这里还要注意 `ttl`，也就是 `Time To Live`，可以理解为：

- 数据包在网络中的生存时间上限
- 每经过一个路由器通常会减一
- 减到 `0` 时，数据包会被丢弃

所以 `ttl` 的意义之一，是防止数据包在网络里无限转发。

### 3.2 `telnet`：测试服务端口能否建立连接

虽然 `telnet` 现在很少用来做正式远程登录，但仍然常被用来做简单的端口连通性测试。

例如：

```bash
telnet www.lagou.com 443
```

如果出现：

```text
Connected to ...
```

通常说明：

- 目标主机能访问到
- 对应端口是打开的
- TCP 三次握手已经成功

所以 `telnet` 常被拿来判断：

- 某个服务端口通不通
- 是网络问题，还是应用层问题

## 4. DNS 查询命令

这一部分和标题最相关，重点是 `host` 和 `dig`。

### 4.1 `host`：快速查询 DNS 记录

`host` 命令适合做快速查询。

例如：

```bash
host www.lagou.com
```

可以看到：

- 域名是否有 `CNAME`
- 最终解析到哪些 `A` 记录

如果要指定查询记录类型，可以使用 `-t` 参数。

例如查询 `AAAA` 记录：

```bash
host -t AAAA www.lagou.com
```

### 4.2 `dig`：更专业、更详细的 DNS 查询工具

`dig` 比 `host` 输出更完整，适合查看 DNS 查询的详细信息。

例如：

```bash
dig www.lagou.com
```

通常可以看到：

- `QUESTION SECTION`
- `ANSWER SECTION`
- `AUTHORITY SECTION`
- `ADDITIONAL SECTION`
- 查询耗时
- 使用的是哪个 DNS 服务器

如果只是想看结果，不想看那么多细节，也可以后面再配合精简参数使用。

## 5. HTTP 相关命令

### 5.1 `curl`：发送 HTTP/HTTPS 请求

`curl` 是 Linux 里非常通用的网络请求工具。

例如：

```bash
curl https://www.lagou.com | head -n 10
```

这个命令表示：

- 用 `curl` 请求一个 HTTPS 页面
- 把返回内容输出到终端
- 再用 `head -n 10` 只看前 10 行

它常用来：

- 测试接口是否可访问
- 查看 HTTP 响应内容
- 调试接口请求

例如发送 POST 请求时，可以写成：

```bash
curl -d '{"x":1}' -H "Content-Type: application/json" -X POST http://localhost:3000/api
```

这说明 `curl` 不只能发 GET，也可以发：

- POST
- PUT
- DELETE

所以它在后端调试里非常常见。

## 6. 如何查看一个域名有哪些 NS 记录

这是本节标题对应的核心问题。

先记结论：**`NS` 记录表示一个域名由哪些权威 DNS 服务器负责解析。**

也就是说，`NS` 记录不是网站服务器地址，而是：

- 这个域名的名字服务器是谁
- 应该去问哪些权威 DNS 服务器

### 6.1 用 `host` 查询 NS 记录

`host` 提供了 `-t` 参数来指定记录类型，所以可以直接这样查：

```bash
host -t ns example.com
```

这里的 `ns` 也可以写成大写：

```bash
host -t NS example.com
```

### 6.2 用 `dig` 查询 NS 记录

`dig` 也可以完成同样的事：

```bash
dig example.com NS
```

或者：

```bash
dig -t NS example.com
```

### 6.3 面试里怎么答更完整

如果面试官问：**如何查看一个域名有哪些 `NS` 记录？**

可以直接答：

1. `NS` 记录表示域名对应的权威 DNS 服务器
2. 可以用 `host -t NS 域名` 查询
3. 也可以用 `dig 域名 NS` 或 `dig -t NS 域名` 查询
4. `host` 输出更简洁，`dig` 输出更详细

例如：

```bash
host -t NS example.com
dig example.com NS
```

## 7. 这一节的整体串联

可以把这一节理解成几类常见网络命令：

- `ssh`：远程登录
- `scp`：远程拷贝
- `ifconfig`：查看网络接口信息
- `netstat`：查看连接、监听端口和网络状态
- `ping`、`telnet`：做基础连通性测试
- `host`、`dig`：做 DNS 查询
- `curl`：发 HTTP/HTTPS 请求

其中最容易被单独提问的一个点就是：

**查看一个域名有哪些 NS 记录，可以使用 `host -t NS 域名`，也可以使用 `dig 域名 NS`。**
