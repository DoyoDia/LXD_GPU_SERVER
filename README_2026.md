# 实验室 GPU 服务器的 LXD 虚拟化（2026 现代化版）

> 这份文档是对旧版 README 的现代化重写。
> 目标不是改几句措辞，而是把整套流程从“2019~2021 年可用”升级到“2026 年仍然能直接照着部署”。

## 1. 为什么要重写

旧版文档里最容易让今天的读者踩坑的地方主要有这些：

- 宿主机和容器默认围绕 Ubuntu 18.04 展开，今天已经明显过时。
- `apt install lxd` 这条路会让很多人装到发行版自带的旧包，而不是当前推荐的 LXD 发行方式。
- 文档默认从第三方镜像站复制很老的 Ubuntu 镜像，容易把环境直接锁死在旧版本。
- 容器桌面和 XRDP 依赖老脚本，不稳定也不好维护。
- ZFS 在旧文档里像“必选项”，但其实它只是存储后端的一种选择。
- GPU 部分更适合改成“宿主机装驱动 + LXD 透传 + 容器按需装用户态”的思路，而不是继续照搬早年的 `.run` 安装方式。

## 2. 本版推荐架构

### 宿主机

- **推荐系统**：Ubuntu Server 24.04 LTS
- **可接受**：Ubuntu Server 22.04 LTS
- **GPU 驱动**：先在宿主机把 NVIDIA 驱动装好，确保 `nvidia-smi` 正常
- **LXD**：使用 snap 安装，推荐 `5.0/stable`（更适合生产）

### 容器镜像

- **首选**：`ubuntu:24.04`
- **兼容场景**：`ubuntu:22.04`
- **不再默认推荐**：`ubuntu:18.04`

### 远程使用方式

按推荐程度排序：

1. **SSH / VS Code Remote SSH**（默认推荐）
2. **RDP 桌面**（有图形需求时再开）
3. **X11 转发**（只适合少量 GUI 程序）

### 存储后端

- **快速开始**：`dir`
- **正式多用户环境**：`zfs` 或 `lvm`
- **结论**：ZFS 很好，但不是必须

---

## 3. 宿主机准备

### 3.1 安装并验证 NVIDIA 驱动

先把宿主机显卡驱动装好，再碰 LXD。无论你后面是让容器直接跑 Python，还是想在容器里再套 Docker，这一步都必须先通。

建议先做到：

```bash
nvidia-smi
```

如果这里都不通，后面 LXD 里的 GPU 一定也不通。

### 3.2 安装 LXD（推荐 snap）

```bash
sudo apt update
sudo apt install -y snapd
sudo snap install lxd --channel=5.0/stable
sudo usermod -aG lxd "$USER"
newgrp lxd
lxc version
```

说明：

- `5.0/stable` 是长期支持分支，适合实验室服务器。
- 如果你明确想跟进最新功能，也可以改成 `sudo snap install lxd`。
- snap 版 LXD 的数据目录默认在 `/var/snap/lxd/common/lxd/`。

---

## 4. 初始化 LXD

第一次部署建议直接跑：

```bash
sudo lxd init
```

### 4.1 存储建议

#### 方案 A：快速开始（最省事）

- 使用 `dir`
- 给 LXD 单独准备一个挂载目录，例如 `/srv/lxd`

这种方式最容易上手，适合先把整套流程跑通。

#### 方案 B：正式部署（推荐）

- 如果你有一块独立数据盘，优先考虑 `zfs` 或 `lvm`
- 如果你非常看重快照、克隆和空间利用率，`zfs` 体验很好
- 如果你更熟悉逻辑卷管理，也可以用 `lvm`

> 建议不要再把“ZFS 是必须”写进文档。
> 更准确的说法应该是：**ZFS 是很适合 LXD 的高级存储后端，但不是唯一正确答案。**

### 4.2 网络建议

如果你所在的校园网 / 机房网络不允许桥接或 macvlan，把容器放在默认的 `lxdbr0` 后面即可，然后通过 **proxy device** 暴露 SSH / RDP 端口。

这也是旧文档最实用、最容易复现的思路，建议保留。

---

## 5. 镜像来源：改用官方 Ubuntu remote

先看本机已有 remotes：

```bash
lxc remote list
```

LXD 默认就带有常用 remote。今天最推荐的 Ubuntu 镜像来源是：

- `ubuntu:` —— 官方稳定 Ubuntu 镜像
- `ubuntu-minimal:` —— 官方最小化 Ubuntu 镜像
- `images:` —— 社区镜像源，适合其他发行版或桌面版测试

查看可用镜像：

```bash
lxc image list ubuntu:
lxc image list ubuntu-minimal:
```

### 推荐做法

#### 直接从远端启动容器

```bash
lxc launch ubuntu:24.04 gpu-dev-01
```

或者：

```bash
lxc launch ubuntu:22.04 cuda-legacy-01
```

#### 如果你想做本地模板缓存

```bash
lxc image copy ubuntu:24.04 local: --alias ubuntu-24.04 --auto-update
lxc launch local:ubuntu-24.04 gpu-dev-02
```

> 不再建议默认从第三方镜像站复制老旧 `18.04` 镜像，更不建议把“老镜像 copy 到 local 再慢慢配”当成主路径。

---

## 6. 容器基础初始化

进入容器：

```bash
lxc exec gpu-dev-01 -- bash
```

在容器内做基础准备：

```bash
apt update
apt install -y openssh-server curl git vim htop tmux
adduser dev
usermod -aG sudo dev
systemctl enable --now ssh
```

建议：

- 不要依赖默认密码账户。
- 第一时间创建自己的普通用户。
- 日常开发优先使用普通用户，不要一直用 root。

---

## 7. 为容器添加 GPU

### 7.1 宿主机先确认 GPU 正常

```bash
nvidia-smi
```

### 7.2 给容器打开 NVIDIA runtime 注入

```bash
lxc config set gpu-dev-01 nvidia.runtime true
```

### 7.3 添加 GPU 设备

#### 把宿主机上的全部 GPU 暴露给容器

```bash
lxc config device add gpu-dev-01 gpu0 gpu
```

#### 只给容器一块指定 GPU（推荐按 PCI 地址绑定）

先看资源：

```bash
lxc info --resources
```

再按 PCI 地址绑定，例如：

```bash
lxc config device add gpu-dev-01 gpu0 gpu pci=0000:65:00.0
```

### 7.4 在容器里验证

```bash
lxc exec gpu-dev-01 -- bash
nvidia-smi
```

如果 `nvidia-smi` 正常，就说明 GPU 透传已经通了。

### 7.5 关于 CUDA / PyTorch / TensorFlow 的建议

现代化的推荐思路是：

- **驱动装在宿主机**
- **容器里优先装框架自己的 wheel / conda 包**
- **只有在需要 nvcc 编译或特定 CUDA Toolkit 时，才额外装对应 toolkit**

换句话说：

- 训练 PyTorch / TensorFlow 时，很多情况下并不需要在容器里再装一整套 `.run` 版驱动
- 不要默认在容器里重复安装内核驱动
- 真需要 CUDA Toolkit 时，也尽量用包管理器或 Conda，而不是把旧教程里的若干 `.run` 文件继续传来传去

---

## 8. 远程访问：优先 SSH，需要桌面再开 RDP

### 8.1 SSH 端口转发

先看宿主机 IP：

```bash
ip addr
```

再看容器 IP：

```bash
lxc list
```

假设：

- 宿主机 IP：`192.168.1.10`
- 容器 IP：`10.0.0.123`
- 你给这个容器分配 SSH 端口 `22001`

则：

```bash
lxc config device add gpu-dev-01 ssh proxy \
  listen=tcp:192.168.1.10:22001 \
  connect=tcp:10.0.0.123:22 \
  bind=host
```

然后从外部机器连接：

```bash
ssh -p 22001 dev@192.168.1.10
```

### 8.2 VS Code Remote SSH（强烈推荐）

今天最推荐的开发方式其实不是完整桌面，而是：

- 容器内跑 SSH
- 本地 VS Code 装 Remote SSH
- 直接打开远程目录

这种方式通常比 XRDP 更稳、更轻、更省资源。

### 8.3 需要桌面时再装 XRDP

如果确实要图形桌面，建议改成 **XFCE + XRDP**，不要再把完整 GNOME 桌面当成默认路径。

在容器内：

```bash
apt update
apt install -y xfce4 xfce4-goodies xrdp
systemctl enable --now xrdp
echo xfce4-session > /home/dev/.xsession
chown dev:dev /home/dev/.xsession
```

然后在宿主机增加 RDP 端口代理，例如 `23389`：

```bash
lxc config device add gpu-dev-01 rdp proxy \
  listen=tcp:192.168.1.10:23389 \
  connect=tcp:10.0.0.123:3389 \
  bind=host
```

Windows 用 `mstsc`，macOS / Linux 用支持 RDP 的客户端连接即可。

---

## 9. 容器资源限制

建议把资源限制写进 profile，而不是每次手工乱改。

先创建一个 profile：

```bash
lxc profile create lab-user
lxc profile edit lab-user
```

示例：

```yaml
config:
  limits.cpu: "8"
  limits.memory: 16GiB
  nvidia.runtime: "true"
description: Lab user baseline profile
devices:
  root:
    path: /
    pool: default
    size: 200GiB
    type: disk
```

给容器加 profile：

```bash
lxc profile add gpu-dev-01 lab-user
```

> 注意：`limits.memory=16GiB` 是“上限”，不是“预留”。
> 你可以逻辑上把多台容器都设成 16GiB，但物理内存并不会因此凭空变多。

---

## 10. 制作母本镜像

当你把基础环境配置好之后，建议立即制作模板，不要每个用户都从零重复配置。

停止容器：

```bash
lxc stop gpu-dev-01
```

发布成镜像：

```bash
lxc publish gpu-dev-01 --alias ubuntu-24.04-gpu-xfce
```

以后直接从模板创建：

```bash
lxc launch local:ubuntu-24.04-gpu-xfce user01
```

模板适合预装这些东西：

- openssh-server
- git / vim / tmux / htop
- 常用中文 locale
- VS Code Remote SSH 友好环境
- 可选的 xfce4 + xrdp
- 统一的管理员说明文件

---

## 11. 容器内再套 Docker（可选）

如果你确实要在 LXD 容器里跑 Docker：

```bash
lxc config set gpu-dev-01 security.nesting true
lxc restart gpu-dev-01
```

然后在容器内安装 Docker。

### 重要建议

- 先问自己是不是真的需要“LXD 里再套 Docker”
- 如果只是给学生 / 用户跑 PyTorch、Jupyter、训练脚本，**直接在 LXD 容器里运行通常更简单**
- 嵌套 Docker + GPU 不是不能做，但复杂度明显更高

### 需要 GPU Docker 时

容器内再按照 **Docker 官方文档 + NVIDIA Container Toolkit 官方文档** 走一遍，重点是：

- 安装 Docker
- 安装 `nvidia-container-toolkit`
- 运行：

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
docker run --rm --gpus all ubuntu nvidia-smi
```

> 某些宿主机 / 驱动 / AppArmor / cgroup 组合下，嵌套 GPU Docker 仍可能需要额外调整。
> 如果不是明确有容器编排需求，建议优先避免多套一层。

---

## 12. 共享目录

共享目录建议优先使用普通 `disk` device，不要默认为了省事就把容器切成特权模式。

```bash
lxc config device add gpu-dev-01 shared-data disk source=/data/shared path=/mnt/shared
```

如果容器里权限不合适，请先从：

- 宿主机目录属主
- 宿主机目录权限
- 容器 UID/GID 映射

这些方向排查。

只有在你完全知道风险时，才去考虑特权容器。

---

## 13. 从旧版文档迁移到新版的建议

| 旧做法 | 新建议 |
|---|---|
| Ubuntu 18.04 | Ubuntu 24.04 / 22.04 |
| `apt install lxd` | `snap install lxd --channel=5.0/stable` |
| 第三方旧镜像源 | `ubuntu:` / `ubuntu-minimal:` 官方 remote |
| 默认完整 GNOME + XRDP | 默认 SSH / VS Code，桌面改成可选 XFCE + XRDP |
| ZFS 写死为必须项 | ZFS / LVM / dir 按场景选择 |
| 容器里手工装老驱动 | 宿主机装驱动，容器优先用 runtime + 框架 wheel |
| 靠老脚本装桌面远程 | 用发行版自己的包管理器维护 |

---

## 14. 推荐的最小可复现流程

如果你只想要一条**今天最稳、最短**的路线，可以按下面走：

1. 宿主机装 Ubuntu Server 24.04
2. 宿主机装好 NVIDIA 驱动并确认 `nvidia-smi` 正常
3. `sudo snap install lxd --channel=5.0/stable`
4. `sudo lxd init`
5. `lxc launch ubuntu:24.04 gpu-dev-01`
6. `lxc config set gpu-dev-01 nvidia.runtime true`
7. `lxc config device add gpu-dev-01 gpu0 gpu`
8. 容器里装 `openssh-server git vim tmux`
9. 用 proxy 暴露 SSH 端口
10. 日常开发统一走 SSH / VS Code Remote SSH
11. 只有确实需要桌面时，再装 XFCE + XRDP
12. 一切配置稳定后，`lxc publish` 做母本

---

## 15. 配套文档

- `lxd容器使用说明/管理员-2026.md`
- `lxd容器使用说明/普通用户使用-2026.md`

如果你准备把仓库彻底升级，可以考虑：

- 保留旧 README 作为历史文档
- 把本文件改名替换成新的主 README
- 在仓库首页明确标注“旧版文档仅供历史参考”
