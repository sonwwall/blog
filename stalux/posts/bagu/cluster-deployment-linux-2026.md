---
title: 第十节：高级指令之集群部署：利用Linux指令同时在多台机器部署程序
abbrlink: cluster-deployment-linux-2026
date: 2026-03-11T21:43:36
updated: 2026-03-11T21:56:17
tags:
  - 操作系统
  - 学习
  - 八股
categories:
  - 操作系统
desc: 第十节：高级指令之集群部署：利用Linux指令同时在多台机器部署程序
---

# 第十节：高级指令之集群部署：利用Linux指令同时在多台机器部署程序

在实际工作里，部署程序往往不是只发到一台机器上，而是需要同时发到多台服务器。

如果每台机器都手工登录、手工执行命令，不仅效率低，而且很容易出错。  
所以这一节的重点，就是利用 Linux 的脚本能力和 SSH 能力，把“单机部署”扩展成“批量部署”。

这一节主要围绕 6 个步骤展开：

1. 搭建一个学习用的集群环境
2. 循环遍历 IP 列表
3. 创建统一的集群管理账户
4. 打通主控机到目标机器的 SSH 权限
5. 先在单机上安装运行环境
6. 再把安装脚本批量执行到远程机器

## 1. 什么是集群部署

这里讲的集群部署，并不是特别复杂的容器编排系统，而是一个更基础、更容易理解的场景：

- 有一台主控机
- 有多台目标服务器
- 主控机通过脚本把命令分发到这些目标机器上执行

图片里的结构可以整理成这样：

- `u1`：本地主控机，使用 Ubuntu 桌面版
- `v1`：目标机器 1，使用 Ubuntu 服务版
- `v2`：目标机器 2，使用 Ubuntu 服务版

所以这节课的核心目标可以概括成一句话：

**让主控机可以批量控制多台服务器，并在这些机器上执行同一套部署动作。**

## 2. 第一步：搭建学习用的集群

为了练习集群部署，最简单的办法就是先准备几台 Linux 机器。

在学习场景中，可以是：

- 一台桌面版 Linux 作为主控机
- 两台服务器版 Linux 作为被管理节点

这里不必一开始就追求很大的规模。

因为这节的重点不是机器数量，而是把下面这条流程跑通：

- 主控机能知道目标机器有哪些
- 主控机能连接这些机器
- 主控机能把脚本发过去
- 主控机能让它们执行同样的安装和部署任务

## 3. 第二步：循环遍历 IP 列表

想要批量操作多台机器，首先得先把目标机器列出来。

最常见的做法，是先准备一个 `iplist` 文件，例如：

```text
192.168.199.130
192.168.199.131
```

这样做的好处是：

- 目标机器一目了然
- 后续脚本可以直接读取
- 新增机器时，只需要补一行 IP

### 3.1 用脚本读取 `iplist`

图片里使用的是 Bash 数组方式：

```bash
#!/usr/bin/bash

readarray -t ips < iplist

for ip in ${ips[@]}
do
    echo $ip
done
```

这个脚本的逻辑很简单：

- `readarray -t ips < iplist`：把 `iplist` 中每一行读进数组 `ips`
- `for ip in ${ips[@]}`：逐个遍历数组中的 IP
- `echo $ip`：把当前 IP 打印出来

执行成功后，就会依次输出：

```text
192.168.199.130
192.168.199.131
```

### 3.2 为什么 `sh foreach.sh` 会报错

图片里还出现了一个常见问题：

```text
./foreach.sh: 2: readarray: not found
./foreach.sh: 4: Bad substitution
```

这通常是因为脚本明明写的是 Bash 语法，却用了 `sh` 来执行。

例如：

```bash
sh ./foreach.sh
```

这里的 `sh` 往往不会按 Bash 语法解释脚本，所以：

- `readarray` 可能不可用
- `${ips[@]}` 这样的数组写法也会报错

正确做法通常有两种：

```bash
bash ./foreach.sh
```

或者直接：

```bash
./foreach.sh
```

前提是脚本首行已经写好解释器，并且文件有执行权限。

这一点可以顺手记住：

**如果脚本里用了 Bash 特性，就不要用 `sh` 去执行。**

## 4. 第三步：创建集群管理账户

如果主控机要批量管理多台服务器，最好不要直接拿 root 账号到处操作。  
更常见、更安全的做法，是统一创建一个专门的管理用户。

图片里使用的账户名是 `lagou`。

### 4.1 创建用户

例如：

```bash
sudo useradd -m -d /home/lagou lagou
```

这里：

- `-m` 表示如果家目录不存在就自动创建
- `-d /home/lagou` 指定用户家目录

如果直接执行 `useradd` 而没有足够权限，就会看到类似报错：

```text
useradd: Permission denied.
useradd: cannot lock /etc/passwd; try again later.
```

这说明：

- 创建用户需要管理员权限
- 所以应该使用 `sudo`

### 4.2 设置密码

创建完用户后，需要给它设置密码：

```bash
sudo passwd lagou
```

系统会提示输入并确认新密码。

### 4.3 加入 sudo 组

如果希望这个用户具备管理能力，可以把它加入 `sudo` 组：

```bash
sudo usermod -G sudo lagou
```

这样它后面就能用 `sudo` 执行管理命令。

### 4.4 设置默认 Shell 和基础环境

图片里还补充了几步初始化动作，可以整理成下面这组命令：

```bash
sudo useradd -m -d /home/lagou lagou
sudo passwd lagou
sudo usermod -G sudo lagou
sudo usermod --shell /bin/bash lagou
sudo cp ~/.bashrc /home/lagou/
sudo chown lagou.lagou /home/lagou/.bashrc
```

这样做的目的主要是：

- 让这个用户使用 `bash`
- 给它准备基础的 shell 配置文件

图片最后还给出了一条免密 `sudo` 的方式：

```bash
sudo sh -c 'echo "lagou ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers'
```

它的作用是让 `lagou` 执行 `sudo` 时不再输入密码。

不过要注意：

**这种配置在学习环境很方便，但在线上生产环境里要谨慎使用。**

## 5. 第四步：打通集群权限

如果主控机每次连接远程机器都要手工输用户名和密码，那就很难做自动化部署。

因此这一节的关键动作，就是：

**打通从主控机到所有目标机器的 SSH 免密登录权限。**

### 5.1 先在主控机生成 SSH 密钥

先准备 `.ssh` 目录并生成密钥对：

```bash
mkdir -p ~/.ssh
cd ~/.ssh
ssh-keygen -t rsa
```

执行后通常会生成两个文件：

- `id_rsa`：私钥
- `id_rsa.pub`：公钥

可以用下面的命令查看：

```bash
ls ~/.ssh
cat ~/.ssh/id_rsa.pub
```

这里要记住一条原则：

- 私钥留在主控机本地
- 公钥分发到目标机器

### 5.2 把公钥写入目标机的 `authorized_keys`

图片里给出的思路，是写一个 `transfer_key.sh` 脚本，接收一个 IP 参数，然后把本机公钥写到远程机器上。

逻辑可以整理成下面这样：

```bash
ip=$1
pubkey=$(cat ~/.ssh/id_rsa.pub)

echo "execute on .. $ip"

ssh lagou@$ip "
mkdir -p ~/.ssh
echo $pubkey >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
"
```

这个脚本做了 4 件事：

- 读取主控机上的公钥
- SSH 登录目标机器
- 确保目标机器存在 `~/.ssh` 目录
- 把公钥追加进 `authorized_keys`

### 5.3 循环给所有机器分发密钥

有了 `iplist` 和 `transfer_key.sh` 后，就可以再包一层循环脚本：

```bash
#!/usr/bin/bash

readarray -t ips < iplist

for ip in ${ips[@]}
do
    sh ./transfer_key.sh $ip
done
```

它表示：

- 逐个遍历 IP
- 对每台机器执行一次密钥分发脚本

第一次连接某台机器时，SSH 往往会提示：

```text
The authenticity of host '192.168.199.130' can't be established.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

输入 `yes` 后，这台机器会加入本机的 `known_hosts`，以后就不会再重复提示。

第一次写入公钥时，通常还需要输入一次远程用户密码。  
但密钥写好后，后续再 `ssh 192.168.199.130` 时，就能直接登录了。

这一步完成后，主控机就真正拥有了对各节点的批量控制能力。

## 6. 第五步：单机安装 Java 环境

在批量部署之前，通常应该先在一台机器上把流程走通。

图片里的示例程序依赖 Java，所以先检查远程机器上有没有 Java：

```bash
which java
java --version
```

如果机器已经装过 Java，还可以继续查看软链接指向：

```bash
ls -l /usr/bin/java
ls -l /etc/alternatives/java
```

这样可以确认：

- `java` 命令是否存在
- 当前系统实际在用哪一套 JDK

### 6.1 安装 OpenJDK

如果机器没有 Java，可以安装：

```bash
sudo apt -y install openjdk-11-jdk
```

这里图片里还创建了一个专门运行 Java 的用户，例如 `ujava`：

```bash
sudo useradd -m -d /opt/ujava ujava
sudo usermod --shell /bin/bash ujava
```

### 6.2 配置 `JAVA_HOME`

安装完之后，可以把 Java 环境变量写入用户配置文件，例如：

```bash
sudo sh -c 'echo "export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64/" >> /opt/ujava/.bash_profile'
```

这样以后这个用户登录后，就能直接使用对应的 Java 环境。

所以这一步的意义是：

**先在单机上把环境安装过程验证清楚，再考虑批量化。**

## 7. 第六步：远程批量安装 Java 环境

当单机脚本已经验证没问题之后，就可以让主控机批量把脚本发到远程执行。

图片里给出的做法，是再写一个通用脚本，让它接收一个脚本文件路径，然后遍历所有 IP 去执行：

```bash
#!/usr/bin/bash

readarray -t ips < iplist

script=$1
for ip in ${ips[@]}
do
    ssh $ip 'bash -s' < $script
done
```

这段命令的核心思想是：

- `script=$1`：拿到待执行的安装脚本
- `ssh $ip 'bash -s' < $script`：把本地脚本内容通过标准输入传给远程机器执行

这样一来，只要你已经准备好了一个安装脚本，例如 `install_java.sh`，就可以：

```bash
./remote.sh install_java.sh
```

然后主控机会自动：

- 遍历所有目标机器
- 逐台 SSH 登录
- 在远程机器上执行同一份安装脚本

这就是“批量部署”的基本雏形。

## 8. 这一套流程的核心思路

如果把整套过程压缩一下，本质上只有 3 层：

### 8.1 第一层：拿到机器列表

也就是：

- 用一个 `iplist` 文件维护所有目标机器

### 8.2 第二层：打通访问权限

也就是：

- 为所有机器准备统一账号
- 主控机生成 SSH 密钥
- 把公钥分发到所有目标机

### 8.3 第三层：批量执行脚本

也就是：

- 先把单机安装步骤写成脚本
- 再通过循环 + SSH 的方式在所有机器上执行

所以集群部署并不神秘，本质就是：

**文件列表管理 + SSH 免密登录 + Shell 批量执行。**

## 9. 总结

这一节可以重点记住下面几件事：

- 集群部署的第一步，是把要管理的机器整理成 IP 列表
- Bash 脚本可以通过 `readarray` 和 `for` 循环批量遍历目标机器
- 统一创建管理账号后，更方便做自动化运维
- 想做自动部署，必须先打通 SSH 免密登录
- 真正的批量部署，本质上是“先写好单机脚本，再远程批量执行”

如果只记最终方法，可以压缩成下面这个流程：

1. 准备 `iplist`
2. 遍历 IP
3. 给所有机器创建统一管理账户
4. 配置 SSH 公钥登录
5. 在单机验证安装脚本
6. 使用 `ssh 'bash -s' < script.sh` 批量远程执行


