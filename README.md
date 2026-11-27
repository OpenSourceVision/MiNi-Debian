# 最小化 Debian 镜像构建

GitHub Actions 工作流：构建最小化 Debian 系统并自动发布 Release。工作流文件为 [`.github/workflows/build-minimal-debian.yml`](.github/workflows/build-minimal-debian.yml)。

固定基于 Debian testing（`forky`）构建，默认登录 `root` / `9527`。

---

## 触发构建

- 手动：Actions 页面运行 `Build minimal Debian image` → `Run workflow`
- 标签：推送 `debian-` 开头的标签

```sh
git tag debian-build-001 && git push origin debian-build-001
```

工作流已声明 `contents: write`，无需额外配置即可创建 Release。

## 输入参数

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `arch` | `amd64` | `arm64` 需公开仓库或 arm64 大规格 runner |
| `root_password` | `9527` | 初始 root 密码 |
| `image_size_mb` | `2048` | 镜像总容量（MiB），根分区约为减去 515 MiB |

---

## 构建产物

| 文件 | 说明 |
| --- | --- |
| `debian-forky-<arch>.img.xz` | 可启动磁盘镜像，GPT + GRUB（amd64 支持 BIOS/UEFI，arm64 为 UEFI） |
| `debian-forky-<arch>-rootfs.tar.xz` | 根文件系统压缩包，用于已有分区或 chroot |
| `SHA256SUMS` | 两个产物的 SHA-256 校验值 |

构建后自动创建 GitHub Release 并上传以上文件；已存在同名 Release 时覆盖上传。Release 页面顶部展示镜像内**实际安装的内核版本**（构建时从 `/boot/vmlinuz-*` 提取）。

## 镜像构建内容

- **基础**：`debootstrap --variant=minbase`。内核与 GRUB 在完整挂载的 chroot 阶段安装（避免 debootstrap 内 initramfs-tools 的 postinst 失败），并生成 initramfs。
- **SSH**：`openssh-server`，允许 root 密码登录，主机密钥构建时生成。
- **网络**：`systemd-networkd` 对常见网卡名启用 DHCPv4（`10-dhcp.network`），预装 `dhcpcd` 作为手动后备，静态公共 DNS。未安装 `systemd-resolved`（其 postinst 会破坏 chroot 构建）。
- **时间同步**：`systemd-timesyncd` 使用 IPv4 字面地址 NTP（阿里云 `203.107.6.88`、授时中心 `1.82.219.234`，Cloudflare 兜底），配置在 `10-ipv4-only.conf`；DHCP 的 `UseNTP=no`，避免 AAAA 解析导致的 IPv6 超时拖慢启动。systemd 262 的 timesyncd 无 `IPFamily=` 选项，故采用此方案。
- **系统前置**：`/sys/fs/bpf` 由 systemd 作为 API 文件系统自动挂载（不创建自定义 mount 单元，否则 systemd 262+ 会报 bad-setting 错误）；`tun` / `nf_conntrack` 开机加载，开启 IP 转发、关闭 `rp_filter`、增大 socket 缓冲区，内置 `iptables` / `kmod` / `bpftool` / `libcap2-bin`。安装 `dbus` 以支持 `timedatectl` 等管理工具。镜像内 apt 源使用清华 TUNA 国内镜像。
- **工具**：`curl`、`wget`、`jq`、`tar`、`xz-utils`、`unzip`、`nano`、`iproute2`、`iputils-ping`、`procps`、`dhcpcd-base`。
- **校验**：构建末段执行 tar/xz 解包测试与 SHA-256 校验和生成。

---

## 软件包清单（模块化）

软件包不在工作流中写死，而是由两个 YAML 清单提供，**构建时自动读取**：

| 文件 | 用途 |
| --- | --- |
| [`config/packages-required.yml`](config/packages-required.yml) | 系统必需包，除非必要不要删。分三组：`bootstrap`（debootstrap 阶段，纯工具）、`chroot`（完整 chroot 阶段）、`arch`（按架构的内核/GRUB） |
| [`config/packages-optional.yml`](config/packages-optional.yml) | 可选插件。后期只需在 `packages:` 下添加 `- 包名`，重新构建即随镜像安装 |

工作原理：

- `Load package manifests` 步骤解析两个清单，校验包名合法性并去重。
- `bootstrap` 包进入 `debootstrap --include`；`chroot` + `arch.[当前架构]` + 可选 `packages` 在完整挂载的 chroot 内用 apt 安装。
- 安装阶段用 policy-rc.d 阻止服务启动，避免插件 postinst 卡住构建。
- 包名只允许小写字母、数字及 `+ . -`，非法名称会使构建失败并明确指出文件与条目。

添加插件示例（编辑 `config/packages-optional.yml`）：

```yaml
packages:
  - docker.io
  - git
  - vim
```

## 修改构建

| 目标 | 修改位置 |
| --- | --- |
| 增删必需包 | `config/packages-required.yml` |
| 增删插件 | `config/packages-optional.yml` |
| 切换套件 | workflow 顶层 `env.SUITE`（注意 apt 源套件名需同步） |
| 网络配置 | `10-dhcp.network` |
| NTP 服务器 | `10-ipv4-only.conf` |
| 发布页内核版本 | `Build bootable raw disk image` 末尾的 `KERNEL_VERSION` 提取段 |
| apt 国内源 | `debian.sources` 生成段 |
| 内核参数 | `99-network-tuning.conf` |
| 分区与引导 | `sgdisk` / `mkfs` / `grub-install` 步骤 |
| 发布逻辑 | `Publish GitHub Release` 步骤 |

## 构建限制

- arm64 镜像面向 UEFI；U-Boot 设备需自行配置引导。
- arm64 runner 仅公有仓库可用，私有仓库需改用大规格 runner。
- 未内置 QEMU 启动测试。
