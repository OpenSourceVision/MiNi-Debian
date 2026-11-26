# 最小化 Debian 镜像构建（GitHub Actions）

本仓库提供一个 GitHub Actions 工作流，用于在 CI 中构建**开箱可用的最小化 Debian 系统**，构建完成后自动发布为 GitHub Release。

- 工作流文件：[`.github/workflows/build-minimal-debian.yml`](.github/workflows/build-minimal-debian.yml)
- 默认发行版：Debian 13 `trixie`（内核 6.12.x）
- 默认登录：`root` / `9527`
- 定位：作为 honk、mihomo 等代理内核的**基础系统**，内核程序由用户在安装后手动部署

## 产物

每次构建会向 GitHub Release 上传三个文件：

| 文件 | 说明 |
| --- | --- |
| `debian-<suite>-<arch>.img.xz` | 可启动的原始磁盘镜像，GPT 分区表，含 GRUB（amd64 同时支持 BIOS/UEFI，arm64 为 UEFI） |
| `debian-<suite>-<arch>-rootfs.tar.xz` | 根文件系统压缩包，可解压到已有磁盘、分区或用于 chroot |
| `SHA256SUMS` | 上述文件的 SHA-256 校验值 |

## 快速开始

1. 将 `.github/workflows/build-minimal-debian.yml` 放入你的 GitHub 仓库。
2. 确认仓库已启用 Actions，并允许工作流创建 Release（工作流已声明 `contents: write` 权限）。
3. 任选一种触发方式：
   - **手动触发**：Actions 页面选择 `Build minimal Debian image (SSH ready, proxy-kernel friendly)` → `Run workflow`，按需填写参数。
   - **标签触发**：推送以 `debian-` 开头的标签，例如：
     ```sh
     git tag debian-build-001
     git push origin debian-build-001
     ```
4. 构建结束后，在仓库的 Releases 页面下载镜像和校验文件。

### 输入参数

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `suite` | `trixie` | Debian 发行版；可选 `bookworm`，但其 6.1 内核不满足 honk 要求 |
| `arch` | `amd64` | 目标架构；`arm64` 需要公开仓库或 arm64 大规格 runner |
| `root_password` | `9527` | 镜像初始 root 密码 |
| `image_size_mb` | `2048` | 原始磁盘镜像大小（MiB） |

## 镜像内置内容

系统基于 `debootstrap --variant=minbase` 构建，预置以下内容：

- **SSH**：`openssh-server` 已启用，允许 root 密码登录；主机密钥在构建时生成。
- **网络**：`systemd-networkd` 对有线网卡启用 DHCP，`systemd-resolved` 与 `systemd-timesyncd` 已启用。
- **基础工具**：`curl`、`wget`、`jq`、`tar`、`xz-utils`、`unzip`、`nano`、`iproute2`、`iputils-ping`、`procps`。
- **代理内核运行环境**：
  - `/sys/fs/bpf` 通过 systemd mount 单元在开机时挂载（honk/eBPF 依赖）。
  - `tun`、`nf_conntrack` 通过 `modules-load.d` 在开机时加载（mihomo TUN 依赖）。
  - `/etc/sysctl.d/99-proxy-kernels.conf` 开启 IPv4/IPv6 转发、关闭 `rp_filter`、增大 socket 缓冲区。
  - 已安装 `iptables`、`kmod`、`bpftool`、`libcap2-bin` 等部署与排障工具。
- **内核**：Debian 对应架构的通用内核与 initramfs；GRUB 已写入镜像，可直接启动。

## 部署说明

### 写入原始磁盘镜像

> 以下命令会**清空目标磁盘全部数据**，请确认设备名（`/dev/sdX`）无误。

```sh
xz -dc debian-trixie-amd64.img.xz | sudo dd of=/dev/sdX bs=4M status=progress
sync
```

也可直接作为虚拟机磁盘使用，例如：

```sh
qemu-system-x86_64 -m 1024 -drive file=debian-trixie-amd64.img,format=raw -netdev user,id=n0 -device virtio-net-pci,netdev=n0
```

首次启动后通过 DHCP 获取地址，使用 `root` / `9527` 登录。

### 使用 rootfs 压缩包

```sh
sudo mkdir -p /mnt/debian
sudo tar -xJf debian-trixie-amd64-rootfs.tar.xz -C /mnt/debian
```

解压后可作为 chroot 环境，或复制到目标设备的根分区；此时需要自行配置目标设备的引导程序、`/etc/fstab` 与内核/设备树（尤其是 ARM 单板机）。

### 校验下载文件

```sh
sha256sum -c SHA256SUMS
```

## 代理内核兼容性

内核程序**未预装**，需用户在系统安装后手动下载并部署。镜像已准备好其运行前置条件：

- **honk**：要求 Linux ≥ 6.12，并具备 BPF/TC/TPROXY 相关内核选项与 bpffs。 Debian `trixie` 通用内核与预配置的 bpffs 挂载满足基础要求；`bookworm` 不适用。注意 honk 与 dae 不能共存（均使用 `dae0` 接口）。
- **mihomo**：TUN 模式需要 `/dev/net/tun`、`CAP_NET_ADMIN` 与 IP 转发；镜像已预加载 `tun` 模块并开启转发。若容器化运行，需额外挂载 `/dev/net/tun` 并赋予相应 capability。

建议部署顺序：先确认 `sshd` 可登录、网络可用，再部署代理内核； honk 首次启动会修改网络 hook、namespace 与 sysctl，应保留带外管理通道。

## 安全须知

- 默认密码 `9527` 强度很低，且镜像允许 root 密码登录。**首次登录后请立即修改密码或改用 SSH 密钥，并限制 SSH 暴露范围。**
- 不要将该镜像直接部署到公网且不设置防火墙。
- 修改默认密码可调整工作流输入 `root_password`，或在构建后手动执行 `passwd root`。
- 写盘操作不可逆，执行 `dd` 前务必核对目标设备。

## 定制

直接编辑 `.github/workflows/build-minimal-debian.yml`：

- 增加/减少软件包：修改 `debootstrap --include` 列表。
- 调整网络：修改 `10-dhcp.network`。
- 调整内核参数：修改 `99-proxy-kernels.conf`。
- 调整镜像分区：修改 `sgdisk`、`mkfs` 与 `grub-install` 步骤。
- 修改发布逻辑：调整 `Publish GitHub Release` 步骤中的标签与 Release 说明。

## 已知限制

- arm64 的原始镜像面向 UEFI 启动环境；树莓派等使用 U-Boot 的设备建议使用 rootfs 压缩包并自行配置引导。
- 使用公有仓库的 arm64 hosted runner；私有仓库需改用 arm64 大规格 runner。
- 本工作流未内置 QA 环节（如 QEMU 启动测试）；镜像完整性校验包含 tar/xz 解包测试与文件校验和。
