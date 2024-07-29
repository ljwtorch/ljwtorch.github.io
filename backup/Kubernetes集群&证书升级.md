
### 集群升级

> **[[官网升级步骤](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)，注意阅读升级说明**

#### 1.更改所有主机软件包存储库地址

> 这里使用的是[[清华镜像源地址](https://mirrors.tuna.tsinghua.edu.cn/help/kubernetes/)](https://mirrors.tuna.tsinghua.edu.cn/help/kubernetes/)，宿主机使用的包管理器为apt，在以下 URL 中，所有仓库的公钥均相同，只需要将仓库地址中的 `v1.28` 修改为所需的版本

```shell
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

vim /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/kubernetes/core:/stable:/v1.27/deb/ /
# deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.tuna.tsinghua.edu.cn/kubernetes/addons:/cri-o:/stable:/v1.27/deb/ /
```

#### 2.确定要升级到哪个版本

```shell
# Find the latest 1.30 version in the list.
# It should look like 1.30.x-*, where x is the latest patch.
sudo apt update
sudo apt-cache madison kubeadm
```

#### 3.升级control-plane节点（kubeadm）

> 该节点必须具有`/etc/kubernetes/admin.conf`文件，**v1.28版本是一个分界点，之前的版本会立即升级所有插件（包括coredns和kube-proxy），需要注意是否还有未升级的其他control-plane节点**

```shell
# replace x in 1.30.x-* with the latest patch version
# 此时目标升级至 v1.27.16 需替换下面的版本
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.30.x-*' && \
sudo apt-mark hold kubeadm

# 验证是否为预期版本
kubeadm version

# 验证升级计划
kubeadm upgrade plan
	# 以下为输出内容
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.27.7
[upgrade/versions] kubeadm version: v1.27.16
I0728 14:52:53.611082 1225480 version.go:256] remote version is much newer: v1.30.3; falling back to: stable-1.27
[upgrade/versions] Target version: v1.27.16
[upgrade/versions] Latest version in the v1.27 series: v1.27.16

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       TARGET
kubelet     2 x v1.27.4   v1.27.16

Upgrade to the latest version in the v1.27 series:

COMPONENT                 CURRENT   TARGET
kube-apiserver            v1.27.7   v1.27.16
kube-controller-manager   v1.27.7   v1.27.16
kube-scheduler            v1.27.7   v1.27.16
kube-proxy                v1.27.7   v1.27.16
CoreDNS                   v1.10.1   v1.10.1
etcd                      3.5.7-0   3.5.12-0

You can now apply the upgrade by executing the following command:

        kubeadm upgrade apply v1.27.16

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________


# 执行对应的升级命令
kubeadm upgrade apply v1.27.16
	# 以下为输出内容
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade/version] You have chosen to change the cluster version to "v1.27.16"
[upgrade/versions] Cluster version: v1.27.7
[upgrade/versions] kubeadm version: v1.27.16
[upgrade] Are you sure you want to proceed? [y/N]: y
[upgrade/prepull] Pulling images required for setting up a Kubernetes cluster
[upgrade/prepull] This might take a minute or two, depending on the speed of your internet connection
[upgrade/prepull] You can also perform this action in beforehand using 'kubeadm config images pull'
W0728 14:54:24.900816 1226355 checks.go:835] detected that the sandbox image "registry.aliyuncs.com/google_containers/pause:3.6" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.aliyuncs.com/google_containers/pause:3.9" as the CRI sandbox image.
[upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.27.16" (timeout: 5m0s)...
[upgrade/etcd] Upgrading to TLS for etcd
[upgrade/staticpods] Preparing for "etcd" upgrade
[upgrade/staticpods] Renewing etcd-server certificate
[upgrade/staticpods] Renewing etcd-peer certificate
[upgrade/staticpods] Renewing etcd-healthcheck-client certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/etcd.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-07-28-14-54-44/etcd.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
[apiclient] Found 1 Pods for label selector component=etcd
[upgrade/staticpods] Component "etcd" upgraded successfully!
[upgrade/etcd] Waiting for etcd to become available
[upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests97929804"
[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Renewing apiserver certificate
[upgrade/staticpods] Renewing apiserver-kubelet-client certificate
[upgrade/staticpods] Renewing front-proxy-client certificate
[upgrade/staticpods] Renewing apiserver-etcd-client certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-07-28-14-54-44/kube-apiserver.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
[apiclient] Found 1 Pods for label selector component=kube-apiserver
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Renewing controller-manager.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-07-28-14-54-44/kube-controller-manager.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
[apiclient] Found 1 Pods for label selector component=kube-controller-manager
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] Renewing scheduler.conf certificate
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2024-07-28-14-54-44/kube-scheduler.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
[apiclient] Found 1 Pods for label selector component=kube-scheduler
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upgrade] Backing up kubelet config file to /etc/kubernetes/tmp/kubeadm-kubelet-config2956970837/config.yaml
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.27.16". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.

# 升级CNI驱动插件，此时使用的是flannel，无需升级即可兼容；并且CNI驱动作为DS运行，无需在其他控制平面节点上执行此步骤
# 如果有其他控制平面节点，需要执行
sudo kubeadm upgrade node
```

#### 4.升级control-plane节点（kubelet & kubectl）

```shell
# 腾空该节点 xxx 为控制平面的节点名称
kubectl drain xxx --ignore-daemonsets
	# 下面为输出内容 省略了不关键的输出内容
node/xxx already cordoned
Warning: ignoring DaemonSet-managed Pods: kube-flannel/kube-flannel-ds-mkn4j, kube-system/kube-proxy-8mf4k, loggie/loggie-tzxtz
evicting pod xxx

.....

pod/xxx evicted
node/xxx drained

# 升级kubectl和kubelet
# 用最新的补丁版本替换 1.30.x-* 中的 x
sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.30.x-*' kubectl='1.30.x-*' && \
sudo apt-mark hold kubelet kubectl
# 重启
sudo systemctl daemon-reload
sudo systemctl restart kubelet
# 解除节点保护
# 将 <node-to-uncordon> 替换为你的节点名称
kubectl uncordon <node-to-uncordon>
```

#### 5.升级工作节点

```shell
# 将 1.30.x-* 中的 x 替换为最新的补丁版本
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.30.x-*' && \
sudo apt-mark hold kubeadm
sudo kubeadm upgrade node
# 腾空节点
kubectl drain xxx --ignore-daemonsets
# 重启
sudo systemctl daemon-reload
sudo systemctl restart kubelet
# 在控制平面节点上执行此命令
# 将 <node-to-uncordon> 替换为你的节点名称
kubectl uncordon <node-to-uncordon>
```



### 证书手动更新

> **[[官网升级步骤](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#manual-preparation-of-component-credentials)](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#manual-preparation-of-component-credentials)，需要查看文档中的警告和笔记**，当集群升级后，证书会自行更新

#### 1.检查证书是否过期

```shell
kubeadm certs check-expiration
```

#### 2.续订证书

```shell
kubeadm certs renew all
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 3.当kubelet客户端证书轮换失败

> kube-apiserver报错 `x509: certificate has expired or is not yet valid`

```shell
# 从故障节点备份和删除 /etc/kubernetes/kubelet.conf 和 /var/lib/kubelet/pki/kubelet-client*
# 在集群中具有 /etc/kubernetes/pki/ca.key 的正常工作的控制平面节点上执行($NODE为故障节点的名称)
kubeadm kubeconfig user --org system:nodes --client-name system:node:$NODE > kubelet.conf
# 复制生成的kubelet.conf到/etc/kubernetes/kubelet.conf
# 在故障节点上执行后，/var/lib/kubelet/pki/kubelet-client-current.pem会重新创建
systemctl restart kubelet
# 手动编辑/etc/kubernetes/kubelet.conf 将生成的 client-certificate-data 和 client-key-data 替换为：
client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
# 重新启动kubelet
systemctl restart kubelet
```

#### 4.更新后处理

```shell
# 重建 apiserver etcd controller-manager scheduler
kubectl -n kube-system delete po etcd-xxx kube-apiserver-xxx kube-controller-manager-xxx kube-scheduler-xxx
# 查看对应pod日志，如果仍然报错：x509: certificate has expired or is not yet valid，则：
# 临时将清单文件从 /etc/kubernetes/manifests/ 移除并等待 20 秒，并再次重建pod
# 之后你可以将文件移回去，kubelet 可以完成 Pod 的重建，而组件的证书更新操作也得以完成
```