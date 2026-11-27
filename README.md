# 最小化 Debian 镜像构建

GitHub Actions 工作流：构建最小化 Debian 系统并自动发布 Release。工作流文件为 [`.github/workflows/build-minimal-debian.yml`](.github/workflows/build-minimal-debian.yml)。

默认基于 Debian testing（`forky`）构建，默认登录 `root` / `9527`。

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
| `suite` | `forky` | 可选 `trixie`、`bookworm`；testing 无 `-updates` / `-security` 套件，源配置已按套件区分 |
| `arch` | `amd64` | `arm64` 需公开仓库或 arm64 大规格 runner |
| `root_password` | `9527` | 初始 root 密码 |
| `image_size_mb` | `2048` | 镜像总容量（MiB），根分区约为减去 515 MiB |

---

## 构建产物

| 文件 | 说明 |
| --- | --- |
| `debian-<suite>-<arch>.img.xz` | 可启动磁盘镜像，GPT + GRUB（amd64 支持 BIOS/UEFI，arm64 为 UEFI） |
| `debian-<suite>-<arch>-rootfs.tar.xz` | 根文件系统压缩包，用于已有分区或 chroot |
| `SHA256SUMS` | 两个产物的 SHA-256 校验值 |

构建后自动创建 GitHub Release 并上传以上文件；已存在同名 Release 时覆盖上传。

## 镜像构建内容

- **基础**：`debootstrap --variant=minbase`，含对应架构通用内核、initramfs 与 GRUB。
- **SSH**：`openssh-server`，允许 root 密码登录，主机密钥构建时生成。
- **网络**：`systemd-networkd` 对常见网卡名启用 DHCPv4（`10-dhcp.network`），预装 `dhclient`，静态公共 DNS，`systemd-timesyncd`。未安装 `systemd-resolved`（其 postinst 会破坏 chroot 构建）。
- **系统前置**：`/sys/fs/bpf` 开机挂载，`tun` / `nf_conntrack` 开机加载，开启 IP 转发、关闭 `rp_filter`、增大 socket 缓冲区，内置 `iptables` / `kmod` / `bpftool` / `libcap2-bin`。镜像内 apt 源使用清华 TUNA 国内镜像。
- **工具**：`curl`、`wget`、`jq`、`tar`、`xz-utils`、`unzip`、`nano`、`iproute2`、`iputils-ping`、`procps`、`isc-dhcp-client`。
- **校验**：构建末段执行 tar/xz 解包测试与 SHA-256 校验和生成。

---

## 修改构建

| 目标 | 修改位置 |
| --- | --- |
| 增删软件包 | `debootstrap --include` 列表 |
| 默认套件 | workflow 顶层 `env.SUITE` 与 `workflow_dispatch` 输入 |
| 网络配置 | `10-dhcp.network` |
| apt 国内源 | `debian.sources` 生成段（按 suite 区分） |
| 内核参数 | `99-network-tuning.conf` |
| 分区与引导 | `sgdisk` / `mkfs` / `grub-install` 步骤 |
| 发布逻辑 | `Publish GitHub Release` 步骤 |

## 构建限制

- arm64 镜像面向 UEFI；U-Boot 设备需自行配置引导。
- arm64 runner 仅公有仓库可用，私有仓库需改用大规格 runner。
- 未内置 QEMU 启动测试。
