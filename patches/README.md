# QEMU Windows 9P (virtfs) 补丁

`qemu-v11.1.3-windows-9p.patch` 为 QEMU **v11.1.3** 添加 Windows 宿主的 virtio-9p 支持，
使 `qemu-system-x86_64.exe` 可以在 Windows 上通过 `-fsdev local,...` 和
`-device virtio-9p-pci` 把宿主目录共享给 Linux 客户机。

## 来源分析

| 来源 | 内容 |
| --- | --- |
| [davidkmg1996/QEMU-9P-Scoop-Install-for-Windows-](https://github.com/davidkmg1996/QEMU-9P-Scoop-Install-for-Windows-) | Scoop 分发包：只有 `bucket/qemu-9p.json` 和 README，**不含源码或补丁**。README 说明它把既有 Windows 9P 实现集成进 QEMU 11.1.1 源码树，并列出集成要点（`local_fid_fd()` 在 Windows 返回 `ENOTSUP`、Windows errno/xattr 处理、跳过 `symlink-install-tree.py` 等）。 |
| [cmspam/winq-emu-qemu](https://github.com/cmspam/winq-emu-qemu)（`alpha10`，基于 QEMU 11.0.0） | 实际源码改动所在：`hw/9pfs/9p-util-win32.c`（1357 行）、`9p-linux-errno.h`、`9p-local.c` / `9p-util.h` / `9p.c` 的 Windows 分支、fsdev 与 meson 调整。WINQ-EMU 的 Windows 9P 实现源自 Bin Meng、Guohuai Shi 等人的上游工作。 |

本补丁把上述 Windows 9P 改动移植到本仓库 submodule 固定的 QEMU **v11.1.3**
（commit `005c61c0ef6f6ae615283440b4bb7be51b2c6ddd`），并处理了 11.0 → 11.1 的接口差异。

## 补丁内容

| 文件 | 改动 |
| --- | --- |
| `hw/9pfs/9p-util-win32.c` | 新增：Windows 版 `*at()` 系列、目录句柄/缓存实现、NTFS ADS 扩展属性、`statfs_win32`、`qemu_futimens_win32` |
| `hw/9pfs/9p-linux-errno.h` | 新增：Linux errno 常量表（`L_*`），供 `errno_to_dotl()` 使用 |
| `hw/9pfs/9p-util.h` | Windows 函数声明、`qemu_*at` 宏映射、`errno_to_dotl()`、`qemu_stat_rdev/blksize/dirent_off` |
| `hw/9pfs/9p-local.c` | 目录/文件句柄操作改用 `qemu_*` 包装；Windows 下禁用 symlink/link/chmod/chown/mknod 等；`LocalData` 增加 root 路径与块大小 |
| `hw/9pfs/9p-local.h` | `LocalData` 定义移入头文件，增加 `root_path` / `block_size` |
| `hw/9pfs/9p.c` | Windows 条件编译：open flags、`S_IFSOCK`/`S_IFIFO`、xattr 创建、fd 上限、`st_blocks` 计算 |
| `hw/9pfs/9p.h` | Windows 下补充 `NAME_MAX`、`AT_*`、`O_DIRECTORY`、`makedev/major/minor`、`S_IS*` 等宏 |
| `hw/9pfs/9p-synth.c` | 避免 Windows 上不存在的 `st_blksize`/`st_blocks`/`d_off` |
| `hw/9pfs/9p-util-generic.c` | `O_*` 标志字符串转换的可用性判断 |
| `hw/9pfs/codir.c` | `qemu_dirent_off()` 新签名 |
| `hw/9pfs/meson.build` | `host_os == 'windows'` 时加入 `9p-util-win32.c` |
| `fsdev/file-op-9p.h` | Windows 下补充 `uid_t`/`gid_t`/`struct statfs` 等 POSIX 类型 |
| `fsdev/meson.build` | Windows 也编译 fsdev 后端 |
| `meson.build` | 允许 Windows 启用 `virtfs` |
| `scripts/symlink-install-tree.py` | Windows 无软链接权限时回退为复制（便携打包） |
| `tests/qtest/libqos/virtio-9p-client.h` | Windows 下提供 `getuid()` |

## 应用

```powershell
git submodule update --init qemu

# 补丁使用 LF 换行；若 QEMU 工作树被 CRLF 检出，先切换后再应用
git -C qemu config core.autocrlf false
git -C qemu rm -r --cached . ; git -C qemu reset --hard

git -C qemu apply --check ..\patches\qemu-v11.1.3-windows-9p.patch
git -C qemu apply ..\patches\qemu-v11.1.3-windows-9p.patch
```

## 构建（MSYS2 UCRT64）

```bash
mkdir -p build && cd build
../qemu/configure --target-list=x86_64-softmmu \
  --enable-virtfs --enable-whpx --enable-slirp \
  --disable-sdl --disable-gtk --disable-opengl --disable-docs
make -j"$(nproc)"
```

## CI 构建

[`../.github/workflows/build-qemu-9p.yml`](../.github/workflows/build-qemu-9p.yml) 提供 Windows 构建流水线：

- **手动触发**：Actions → *Build QEMU with virtio-9p for Windows* → *Run workflow*。QEMU 版本固定取自 `.gitmodules` 的 `branch`（当前 `v11.1.3`），无需手动指定；可选参数：
  - `targets`：QEMU target 列表，逗号分隔（默认 `x86_64-softmmu`）；
  - `configure_args`：追加到默认参数之后的额外 `./configure` 选项；
  - `jobs`：`make -j` 并行度（留空用 CPU 数）；
  - `minimal_devices`：启用 `--without-default-devices`，产物更小、构建更快，但所有设备都必须显式 `-device` 指定（默认关闭）。
- **打 tag 触发**：推送 `qemu-v*` tag 时构建，并自动创建 GitHub Release 附加 zip。
- 产物：名为 `qemu-9p-windows-x86_64` 的 artifact（zip），保留 30 天。

工作流会浅克隆 `.gitmodules` 指定的 QEMU tag、应用 `patches/*.patch`、用 MSYS2 UCRT64
构建（WHPX + virtfs + slirp），并打包 `bin/*.exe`、运行时 DLL 与固件。

为缩短构建时间，默认关闭了 VNC、SPICE、Xen、curses、guest-agent、vhost-net/user/vdpa、
capstone 以及 SDL/GTK/OpenGL/VirGL 等与 9P 场景无关的子系统；同时用 ccache
（`actions/cache` 持久化）缓存重复构建。

## 使用

```text
qemu-system-x86_64.exe -accel whpx -m 4096 -smp 4 ^
  -fsdev local,id=fsdev0,path=C:/9p-share,security_model=none ^
  -device virtio-9p-pci,fsdev=fsdev0,mount_tag=hostshare
```

Linux 客户机：

```text
sudo mkdir -p /mnt/hostshare
sudo mount -t 9p -o trans=virtio hostshare /mnt/hostshare
```

## 限制

- Windows 宿主上 `security_model=none/passthrough` 时，`symlink`/`readlink`/`link`/`chmod`/`chown`/`mknod`
  返回 `ENOTSUP`（Windows 无对应 POSIX 语义）；`mapped` 模式依赖 NTFS 的 ADS 存 xattr。
- 目录文件描述符在 Windows 上不可用，`local_fid_fd()` 对目录返回 `ENOTSUP`。
- 上游项目验证的组合是 Windows x86-64 + WHPX + `x86_64-softmmu`。
- 本补丁尚未在 Windows 工具链上编译验证（当前环境无 MSYS2/mingw），建议在 CI 中构建验证。

## 11.0 → 11.1.3 适配说明

- v11.1.3 的 `meson.build` 仍要求 virtfs 仅在 Linux/macOS/FreeBSD 上启用，本补丁放开 Windows。
- 保留 v11.1.3 `local_fid_fd()` 新增的 `P9_FID_FILE` 分支，只给目录分支加 Windows `ENOTSUP`。
- v11.1.3 已简化 `stat_to_v9stat_dotl()` 的时间戳赋值；本补丁仅在 `CONFIG_WIN32` 下计算
  `st_blocks`（512 字节单位），其他平台保持 11.1.3 原逻辑，不引入 FreeBSD 回归。
- 上游 WINQ-EMU 补丁把若干 `#else` 改成 `#error Missing ... for this host system`，
  本补丁补齐了 `CONFIG_FREEBSD` 分支，避免破坏 FreeBSD 构建。
