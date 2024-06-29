> [!IMPORTANT]
> [GitHub原文链接](https://github.com/kelseyhightower/kubernetes-the-hard-way) 本文使用ChatGPT协助翻译

# 先决条件

在本实验中，您将回顾跟随本教程所需的机器要求。

## 虚拟或物理机器

本教程需要四（4）台运行Debian 12（bookworm）的虚拟或物理ARM64机器。下表列出了四台机器及其CPU、内存和存储要求。

| 名称    | 描述               | CPU  | 内存  | 存储 |
| ------- | ------------------ | ---- | ----- | ---- |
| jumpbox | 管理主机           | 1    | 512MB | 10GB |
| server  | Kubernetes服务器   | 1    | 2GB   | 20GB |
| node-0  | Kubernetes工作节点 | 1    | 2GB   | 20GB |
| node-1  | Kubernetes工作节点 | 1    | 2GB   | 20GB |

如何配置这些机器由您决定，唯一的要求是每台机器必须符合上述系统要求，包括机器规格和操作系统版本。一旦您配置好所有四台机器，通过在每台机器上运行 `uname` 命令来验证系统要求：

```bash
uname -mov
```

运行 `uname` 命令后，您应该会看到以下输出：

```bash
#1 SMP Debian 6.1.55-1 (2023-09-29) aarch64 GNU/Linux
```

您可能会惊讶地看到 `aarch64`，但这是Arm架构64位指令集的正式名称。您经常会看到Apple和Linux内核维护者在提到对`aarch64`的支持时使用`arm64`。本教程将始终使用`arm64`以避免混淆。

# 设置Jumpbox

在本实验中，您将设置四台机器之一为`jumpbox`。这台机器将用于在本教程中运行命令。虽然使用专用机器以确保一致性，但这些命令也可以从几乎任何机器运行，包括运行macOS或Linux的个人工作站。

将`jumpbox`视为管理机器，您将使用它作为基础来从头开始设置Kubernetes集群。在开始之前，我们需要安装一些命令行工具并克隆Kubernetes The Hard Way Git库，该库包含一些用于配置各种Kubernetes组件的额外配置文件。

登录到`jumpbox`：

```bash
ssh root@jumpbox
```

所有命令将以`root`用户身份运行。这是为了方便起见，并将帮助减少设置所有内容所需的命令数量。

### 安装命令行工具

现在，您已经以`root`用户身份登录到`jumpbox`机器，您将安装用于在整个教程中执行各种任务的命令行工具。

```bash
apt-get -y install wget curl vim openssl git
```

### 同步GitHub库

现在是下载本教程副本的时候了，其中包含将用于从头构建Kubernetes集群的配置文件和模板。使用`git`命令克隆Kubernetes The Hard Way Git库：

```bash
git clone --depth 1 \
  https://github.com/kelseyhightower/kubernetes-the-hard-way.git
```

进入`kubernetes-the-hard-way`目录：

```bash
cd kubernetes-the-hard-way
```

这将是本教程其余部分的工作目录。如果您迷路了，请运行`pwd`命令以确认在`jumpbox`上运行命令时是否在正确的目录中：

```shell
pwd
/root/kubernetes-the-hard-way
```

### 下载二进制文件

下载列在`downloads.txt`文件中的二进制文件，使用`wget`命令：

```bash
wget -q --show-progress \
  --https-only \
  --timestamping \
  -P downloads \
  -i downloads.txt
```

根据您的互联网连接速度，下载`584`兆字节的二进制文件可能需要一段时间，下载完成后，您可以使用`ls`命令列出它们：

```bash
ls -loh downloads

total 584M
-rw-r--r-- 1 root  41M May  9 13:35 cni-plugins-linux-arm64-v1.3.0.tgz
-rw-r--r-- 1 root  34M Oct 26 15:21 containerd-1.7.8-linux-arm64.tar.gz
-rw-r--r-- 1 root  22M Aug 14 00:19 crictl-v1.28.0-linux-arm.tar.gz
-rw-r--r-- 1 root  15M Jul 11 02:30 etcd-v3.4.27-linux-arm64.tar.gz
-rw-r--r-- 1 root 111M Oct 18 07:34 kube-apiserver
-rw-r--r-- 1 root 107M Oct 18 07:34 kube-controller-manager
-rw-r--r-- 1 root  51M Oct 18 07:34 kube-proxy
-rw-r--r-- 1 root  52M Oct 18 07:34 kube-scheduler
-rw-r--r-- 1 root  46M Oct 18 07:34 kubectl
-rw-r--r-- 1 root 101M Oct 18 07:34 kubelet
-rw-r--r-- 1 root 9.6M Aug 10 18:57 runc.arm64
```

### 安装kubectl

在本节中，您将在`jumpbox`机器上安装`kubectl`，即官方Kubernetes客户端命令行工具。在本教程后面您的集群配置完成后，`kubectl`将用于与Kubernetes控制进行交互。

使用`chmod`命令使`kubectl`二进制文件可执行并将其移动到`/usr/local/bin/`目录：

```bash
{
  chmod +x downloads/kubectl
  cp downloads/kubectl /usr/local/bin/
}
```

此时，`kubectl`已安装，可以通过运行`kubectl`命令进行验证：

```bash
kubectl version --client

Client Version: v1.28.3
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

此时，`jumpbox`已安装完成所有必要的命令行工具和实用程序。

# 配置计算资源

Kubernetes 需要一组机器来托管 Kubernetes 控制平面和最终运行容器的工作节点。在本实验中，您将配置设置 Kubernetes 集群所需的机器。

## 机器数据库

本教程将利用一个文本文件作为机器数据库，存储在设置 Kubernetes 控制平面和工作节点时将使用的各种机器属性。以下模式表示机器数据库中的条目，每行一个条目：

```bash
IPV4_ADDRESS FQDN HOSTNAME POD_SUBNET
```

每一列分别对应一台机器的IP地址 `IPV4_ADDRESS`，完全限定域名 `FQDN`，主机名 `HOSTNAME` 和 IP 子网 `POD_SUBNET`。Kubernetes 为每个 `pod` 分配一个 IP 地址，`POD_SUBNET` 表示分配给集群中每台机器的唯一 IP 地址范围。

以下是创建本教程时使用的机器数据库示例。请注意，IP 地址已被屏蔽。只要每台机器能够相互访问以及访问`jumpbox`，您的机器可以分配任何 IP 地址。

```bash
cat machines.txt

XXX.XXX.XXX.XXX server.kubernetes.local server  
XXX.XXX.XXX.XXX node-0.kubernetes.local node-0 10.200.0.0/24
XXX.XXX.XXX.XXX node-1.kubernetes.local node-1 10.200.1.0/24
```

现在轮到您创建一个`machines.txt`文件，其中包含您将用来创建 Kubernetes 集群的三台机器的详细信息。使用上面的示例机器数据库，并添加您的机器的详细信息。

## 配置 SSH 访问

SSH 将用于配置集群中的机器。请验证您对机器数据库中列出的每台机器都有 `root` SSH 访问权限。您可能需要通过更新 sshd_config 文件并重新启动 SSH 服务器来启用每个节点上的 root SSH 访问。

### 启用 root SSH 访问

如果每台机器上启用了 `root` SSH 访问，您应该可以使用以下命令验证 SSH 访问：

```bash
for host in server node-0 node-1
do ssh root@${host} uname -o -m -n
done

server aarch64 GNU/Linux
node-0 aarch64 GNU/Linux
node-1 aarch64 GNU/Linux
```

## 将 DNS 条目添加到本地机器

在本节中，您将从 `hosts` 文件附加 DNS 条目到 `jumpbox` 机器上的本地 `/etc/hosts` 文件。

将 `hosts` 文件中的 DNS 条目附加到 `/etc/hosts`：

```bash
cat hosts >> /etc/hosts
```

验证 `/etc/hosts` 文件已更新：

```bash
cat /etc/hosts

127.0.0.1       localhost
127.0.1.1       jumpbox

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

# Kubernetes The Hard Way
XXX.XXX.XXX.XXX server.kubernetes.local server
XXX.XXX.XXX.XXX node-0.kubernetes.local node-0
XXX.XXX.XXX.XXX node-1.kubernetes.local node-1
```

此时，您应该可以使用主机名 SSH 访问 `machines.txt` 文件中列出的每台机器。

```bash
for host in server node-0 node-1
do ssh root@${host} uname -o -m -n
done

server aarch64 GNU/Linux
node-0 aarch64 GNU/Linux
node-1 aarch64 GNU/Linux
```

## 将 DNS 条目添加到远程机器

在本节中，您将从 `hosts` 文件附加 DNS 条目到 `machines.txt` 文件中列出的每台机器的 `/etc/hosts` 文件中。

将 `hosts` 文件复制到每台机器，并将内容附加到 `/etc/hosts`：

```bash
while read IP FQDN HOST SUBNET; do
  scp hosts root@${HOST}:~/
  ssh -n \
    root@${HOST} "cat hosts >> /etc/hosts"
done < machines.txt
```

此时，可以在 `jumpbox` 机器或 Kubernetes 集群中的任何三台机器之间使用主机名进行连接。您现在可以使用 `server`、`node-0` 或 `node-1` 等主机名而不是 IP 地址进行连接。	