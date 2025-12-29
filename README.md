# 使用 Ansible 在 Ubuntu 22.04 上部署高可用 Kubernetes 集群

本项目提供了一套 Ansible Playbook，用于在 Ubuntu 22.04 LTS 服务器上自动化部署 Kubernetes (K8s) 集群。它具有高度的灵活性和自动化能力，支持从简单的单 Master 节点到生产级的多 Master 高可用 (HA) 集群的多种部署场景。

## ✨ 核心特性

- **操作系统支持**: 专为 `Ubuntu 22.04 LTS` 设计和优化。

- **灵活的部署模式**:
  - **单 Master 模式**: 快速搭建一个用于开发和测试的 K8s 环境。
  - **高可用 (HA) 模式**: 部署一个包含3个 Master 节点、外部 ETCD 集群以及 Keepalived + Nginx 负载均衡的生产级高可用集群。
  
- **智能容器运行时**:
  - 根据您在 `group_vars/all.yml` 中定义的 `k8s_version` 自动选择容器运行时：
    - `k8s_version < 1.24.0`: 自动部署 **Docker**。
    - `k8s_version >= 1.24.0`: 自动部署 **containerd**。
  
- **智能选择高可用部署**
  
  - 根据您在 `group_vars/all.yml` 中定义的 `keepalived` 选项自动选择是否启用高可以用部署，以下配置留空为单 master 节点部署
  
    ```
    nic: 'ens160'                                 # 网关网卡
    Virtual_Router_ID: '21'                       # keeplaived 唯一路由 ID
    vip: '192.168.198.65'                         # 高可用虚拟IP
    api_vip_hosts: 'apiserver.cluster.local'      # 高可用集群通信域名
    lb_port: '16443'                              # 负载均衡端口
    ```
  
- **智能选择ETCD二进制部署**
  
  - 根据 hosts.ini 文件中是否添加 ETCD 组节点来自动选择二进制部署

- **完整的生命周期管理**:
  - `single-master-deploy.yml`: 部署单 Master 集群。
  - `multi-master-ha-deploy.yml`: 部署高可用 Master 集群。
  - `add-node.yml`: 向现有集群中添加新的 Worker 节点。
  - `remove-k8s.yml`: 安全、彻底地卸载和清理整个 K8s 集群。
- **高度可定制**: 通过 `group_vars/all.yml` 文件可以轻松定制 Kubernetes 版本、网络插件、镜像仓库等核心参数。

---

## 🚀 快速开始

### 1. 环境准备

- **Ansible 控制节点**: 一台安装了 Ansible 的机器（可以是集群中的某个 Master 节点）。
- **目标服务器**: 若干台纯净安装的 Ubuntu 22.04 LTS 服务器。
- **网络**: 所有服务器之间需要网络互通，并能访问互联网（或您指定的内部镜像源）。
- **SSH 免密登录**: 从 Ansible 控制节点到所有目标服务器的 SSH key-based 免密登录必须配置好，且如果使用普通用户需配置sudo权限。

> 如果不配置免密登陆也是可以的

### 2. 配置主机清单 (`hosts.ini`)

`hosts.ini` 文件是 Ansible 的库存文件，用于定义您的服务器角色和连接信息。请根据您的部署模式进行配置。

#### 示例 1: 单 Master 节点部署

对于单 Master 部署，您**不需要**配置 `[etcd]` 和 `[ha]` 组。

```ini
[all]
k8s-master1 ansible_host=192.168.198.62 ip=192.168.198.62 ansible_port=22 ansible_user=tianxiang
k8s-node1 ansible_host=192.168.198.63 ip=192.168.198.63 ansible_port=22 ansible_user=tianxiang

[master]
k8s-master1

[node]
k8s-node1

#24小时token过期后添加node节点
[newnode]
[k8s:children]
master
node
newnode
```

#### 示例 2: 高可用 (HA) 多 Master 部署

对于高可用部署，您需要配置所有分组，包括 `[master]`, `[etcd]`, 和 `[ha]`。

```ini
[all]
k8s-master1 ansible_host=192.168.198.62 ip=192.168.198.62 ansible_port=22 ansible_user=tianxiang
k8s-master2 ansible_host=192.168.198.63 ip=192.168.198.63 ansible_port=22 ansible_user=tianxiang
k8s-master3 ansible_host=192.168.198.64 ip=192.168.198.64 ansible_port=22 ansible_user=tianxiang
#k8s-node1 ansible_host=192.168.198.65 ip=192.168.198.65 ansible_port=22 ansible_user=tianxiang
#k8s-node2 ansible_host=192.168.198.66 ip=192.168.198.66 ansible_port=22 ansible_user=tianxiang
etcd1 ansible_host=192.168.198.62 ip=192.168.198.62 ansible_port=22 ansible_user=tianxiang
etcd2 ansible_host=192.168.198.63 ip=192.168.198.63 ansible_port=22 ansible_user=tianxiang
etcd3 ansible_host=192.168.198.64 ip=192.168.198.64 ansible_port=22 ansible_user=tianxiang

[master]
k8s-master1
k8s-master2
k8s-master3

[node]
#k8s-node1
#k8s-node2

[etcd]
etcd1
etcd2
etcd3

[ha]
k8s-master1 ha_name=ha-master
k8s-master2 ha_name=ha-backup
k8s-master3 ha_name=ha-backup

#24小时token过期后添加node节点
[newnode]
[k8s:children]
master
node
newnode
```

**不使用免密登陆方式**

```
$ sudo apt -y install sshpass
```

```
[all]
k8s-master1 ansible_host=192.168.198.62 ip=192.168.198.62
k8s-master2 ansible_host=192.168.198.63 ip=192.168.198.63
k8s-master3 ansible_host=192.168.198.64 ip=192.168.198.64
#k8s-node1 ansible_host=192.168.198.65 ip=192.168.198.65
#k8s-node2 ansible_host=192.168.198.66 ip=192.168.198.66
etcd1 ansible_host=192.168.198.62 ip=192.168.198.62
etcd2 ansible_host=192.168.198.63 ip=192.168.198.63
etcd3 ansible_host=192.168.198.64 ip=192.168.198.64

[all:vars]
ansible_port=22
ansible_user=tianxiang
ansible_ssh_pass=tx010910.
ansible_become=yes
ansible_become_method=sudo
ansible_become_password=tx010910.
ansible_ssh_common_args='-o StrictHostKeyChecking=no'

[master]
k8s-master1
k8s-master2
k8s-master3

[node]
#k8s-node1
#k8s-node2

[etcd]
etcd1
etcd2
etcd3

[ha]
k8s-master1 ha_name=ha-master
k8s-master2 ha_name=ha-backup
k8s-master3 ha_name=ha-backup

#24小时token过期后添加node节点
[newnode]
[k8s:children]
master
node
newnode
```

### 3. 配置核心变量 (`group_vars/all.yml`)

在执行 Playbook 之前，请务必检查并修改 `group_vars/all.yml` 文件中的以下关键变量：

- `k8s_version`: **(重要)** 定义要安装的 Kubernetes 版本。例如: `"1.23.0"`。这将自动决定容器运行时的选择。
- `image_repository`: Kubernetes 组件镜像的仓库地址，例如: `registry.cn-hangzhou.aliyuncs.com/google_containers`。
- `pod_cidr`: Pod 网络的 CIDR 地址段，例如: `"10.244.0.0/16"`。
- `service_cidr`: Service 网络的 CIDR 地址段，例如: `"10.96.0.0/16"`。

**仅在高可用 (HA) 模式下需要配置以下变量:**

- `vip`: Keepalived 使用的虚拟 IP (VIP)，这是您的 K8s API Server 的统一入口。例如: `"192.168.1.100"`。
- `api_vip_hosts`: 与 VIP 对应的域名。例如: `apiserver.k8s.local`。
- `nic`: Master 节点上用于 Keepalived 绑定 VIP 的网卡名称。例如: `"ens192"`。
- `lb_port`: Nginx 4 层代理端口。例如: `16443`

### 4. 执行 Playbook

根据您选择的部署模式，运行相应的命令：

- **部署单 Master 集群**:
  ```bash
  ansible-playbook -i hosts.ini single-master-deploy.yml
  ```

- **部署高可用 (HA) 集群**:
  ```bash
  ansible-playbook -i hosts.ini multi-master-ha-deploy.yml
  ```

---

## 📦 集群管理

### 添加 Worker 节点

1.  将新的 Worker 节点信息添加到 `hosts.ini` 的 `[newnode]` 组下。
2.  执行 `add-node.yml` 剧本：
    ```bash
    ansible-playbook -i hosts.ini add-node.yml
    ```
3.  成功后，将新节点从 `[newnode]` 组移至 `[node]` 组，以便于后续管理。

### 卸载集群

> **警告**: 这将删除集群中的所有数据和配置，操作不可逆！

如果您需要彻底清理和卸载集群，请执行：
```bash
ansible-playbook -i hosts.ini remove-k8s.yml
```

---

## ✅ 验证集群

部署完成后，您可以在 Master 节点上使用 `kubectl` 命令来验证集群状态：

```bash
# 查看所有节点状态
kubectl get nodes -o wide

# 查看系统 Pod 运行状态
kubectl get pods -A
```

如果所有节点都处于 `Ready` 状态，并且 `kube-system` 命名空间下的所有 Pod 都在正常运行 (`Running`)，那么恭喜您，集群已成功部署！
