# TinyVirt 需求规格说明书

| 项目 | 内容 |
| --- | --- |
| 文档名称 | TinyVirt 需求规格说明书 |
| 文档编号 | TVL-SRS-001 |
| 项目名称 | TinyVirtualLinux / TinyVirt |
| 版本 | 1.0 |
| 状态 | 待评审 |
| 编制日期 | 2026-11-08 |
| 需求基线 | 本文件 |
| 适用阶段 | 设计、开发、测试、验收 |

## 修订记录

| 版本 | 日期 | 修订人 | 修订内容 |
| --- | --- | --- | --- |
| 1.0 | 2026-11-08 | TinyVirt 项目组 | 初版：明确 QMP 管理、SQLite 持久化、端口映射、9P、快照、备份和管理台需求 |

## 审批记录

| 角色 | 姓名 | 审批意见 | 日期 |
| --- | --- | --- | --- |
| 需求提出人 | 待填写 | 待评审 | 待填写 |
| 开发负责人 | 待填写 | 待评审 | 待填写 |
| 测试负责人 | 待填写 | 待评审 | 待填写 |

---

## 1. 引言

### 1.1 目的

本文档定义 TinyVirt 的产品需求、功能边界、数据持久化要求、接口要求、质量要求和验收标准。本文档是后续设计、开发、测试和验收的唯一需求基线。

本项目的核心目标：在现有 TinyVirtualLinux 仓库基础上，提供一个可独立运行的 Go 服务端和浏览器管理台，通过 QEMU QMP 协议管理本机 x86_64 虚拟机，支持虚拟机参数与镜像配置、创建、端口映射、9P 文件共享、状态管理、启动停止、qcow2 内部快照和完整镜像备份。

### 1.2 范围

本期范围包括：

1. Go 服务端，默认监听 `127.0.0.1:8080`，内嵌并提供浏览器管理台。
2. QMP 客户端，通过 TCP loopback 与 QEMU 通信，管理 VM 生命周期和状态。
3. SQLite 持久化：设置、VM 配置、运行状态、端口映射、9P 共享、快照元数据、备份元数据、操作审计。
4. VM 镜像管理：新建 qcow2、引用已有 qcow2/raw 镜像、可选 ISO 光盘。
5. 端口映射：QEMU user-mode 网络 `hostfwd`，创建时配置，运行中动态增删。
6. 9P 文件共享：多宿主目录、mount tag、只读、安全模型。
7. VM 操作：启动、ACPI 关机、强制退出、暂停、恢复、重启、删除。
8. qcow2 内部快照：创建（含/不含 VM 状态）、列表、恢复、删除。
9. 完整镜像备份：创建、列表、下载、恢复、删除。
10. 日志查看、系统信息、QEMU 能力检测、文件浏览、操作审计。
11. 服务端重启后接管仍在运行的 VM。
12. 构建脚本、CI、README、API 文档、自动化测试。

### 1.3 术语与缩写

| 术语 | 说明 |
| --- | --- |
| QEMU | 本项目的虚拟机监控器，使用 qemu-system-x86_64 |
| QMP | QEMU Machine Protocol，JSON-RPC 风格的 QEMU 控制协议 |
| qemu-img | QEMU 镜像管理工具，用于创建、探测、快照、转换 |
| VM | 虚拟机 |
| QMP 端口 | 服务端与 QEMU 通信的本地 TCP 端口 |
| hostfwd | QEMU user-mode 网络的宿主到客户机端口转发规则 |
| 9P / virtfs | QEMU 的 virtio-9p 文件共享机制 |
| qcow2 | QEMU 的写时复制磁盘格式，支持内部快照 |
| raw | 原始磁盘镜像格式，不支持内部快照 |
| 内部快照 | 保存在 qcow2 文件内部的快照 |
| 完整备份 | 独立、可单独使用的 qcow2 镜像文件 |
| SQLite | 嵌入式关系型数据库，本项目唯一结构化数据存储 |
| WAL | SQLite Write-Ahead Logging 日志模式 |
| SRS | Software Requirements Specification，需求规格说明书 |
| managed 镜像 | 由本服务在 `<data-dir>/images` 下创建和管理的镜像 |
| 外部镜像 | 用户提供绝对路径、由本服务引用但不负责删除的镜像 |

### 1.4 参考资料

| 资料 | 说明 |
| --- | --- |
| QEMU QMP 文档 | `qemu/qapi/*.json`、QEMU 源码中的 QMP 定义 |
| QEMU 9P/virtfs 文档 | `patches/README.md` 及 QEMU `-fsdev`/`-virtfs` 选项 |
| qcow2 快照文档 | QEMU `snapshot-save`、`snapshot-load`、`snapshot-delete`、`qemu-img snapshot` |
| SQLite 文档 | https://www.sqlite.org/docs.html |
| modernc.org/sqlite | https://pkg.go.dev/modernc.org/sqlite |
| 本仓库 QEMU 构建工作流 | `.github/workflows/build-qemu-9p.yml` |
| 本仓库 Windows 9P 补丁 | `patches/qemu-v11.1.3-windows-9p.patch`、`patches/README.md` |

### 1.5 读者对象

- 需求提出人和产品负责人：确认范围和验收标准。
- 后端开发人员：实现 Go 服务端、QMP、SQLite、QEMU 交互。
- 前端开发人员：实现 React/Ant Design 管理台。
- 测试人员：依据需求编号和验收矩阵设计测试用例。
- 运维/最终用户：了解部署、使用、安全限制和故障排查。

---

## 2. 总体描述

### 2.1 产品定位

TinyVirt 是一个面向本机开发和测试场景的轻量级 QEMU 虚拟机管理服务。它不替代完整的云平台，而是提供：

- 开箱即用的 Web 管理界面；
- 通过 QMP 对 QEMU 的可靠控制；
- 面向 TinyVirtualLinux 场景的 9P 目录共享；
- qcow2 快照和独立镜像备份；
- 用 SQLite 持久化所有管理数据，服务重启不丢配置和状态。

### 2.2 目标用户

| 用户类型 | 主要诉求 |
| --- | --- |
| 个人开发者 | 快速创建 Linux VM、共享宿主目录、映射 SSH/HTTP 端口 |
| 测试工程师 | 创建/复制测试环境、创建快照、回滚、备份镜像 |
| 教学/演示用户 | 通过浏览器管理 VM，查看状态和日志 |
| 本机运维人员 | 在本机集中管理多个 QEMU 进程和镜像 |

### 2.3 运行环境

| 项目 | 要求 |
| --- | --- |
| 宿主操作系统 | Windows 10/11 x86_64 优先；Linux x86_64 兼容 |
| QEMU | 由本仓库构建的 `qemu-system-x86_64`，Windows 版本需含 9P 补丁；版本以 `.gitmodules` 固定的 QEMU 为准 |
| qemu-img | 与 QEMU 同目录或通过设置指定 |
| Go | 1.26 或更高兼容版本 |
| Node.js | 24 或更高兼容版本 |
| 包管理器 | pnpm |
| 数据库 | 嵌入式 SQLite 3（由 Go 驱动内置，无外部数据库服务） |
| 浏览器 | 支持 ES2020 的现代浏览器（Chrome/Edge/Firefox/Safari 近期版本） |
| 权限 | 普通用户即可；WHPX/KVM 等加速器可能需要系统权限或启用相应功能 |

### 2.4 系统上下文

```text
┌──────────────────────────────┐
│ 浏览器                        │
│ React 18 + Ant Design 5      │
└──────────────┬───────────────┘
               │ HTTP JSON /api
               ▼
┌──────────────────────────────┐
│ Go 服务端                     │
│  - HTTP API / 内嵌静态资源     │
│  - VM Manager / 状态机         │
│  - QMP Client                 │
│  - SQLite Store               │
│  - qemu-img 调用              │
└───────┬───────────────┬──────┘
        │ QMP TCP       │ 文件系统
        ▼               ▼
┌──────────────┐  ┌──────────────────────────┐
│ QEMU 进程     │  │ <data-dir>               │
│ qemu-system  │  │  tinyvirt.db             │
│ -x86_64      │  │  images/ backups/ logs/  │
└──────────────┘  └──────────────────────────┘
```

### 2.5 设计约束

1. 默认仅监听回环地址，不内置认证；非回环监听必须显式开启并提示风险。
2. QMP 和 VNC 均只绑定 `127.0.0.1`。
3. 所有外部命令通过 argv 数组执行，禁止拼接 shell 命令。
4. 结构化数据必须持久化到 SQLite；镜像、备份、日志等大文件保留文件系统，数据库保存路径和元数据，不把多 GB 镜像以 BLOB 写入数据库。
5. 使用纯 Go SQLite 驱动 `modernc.org/sqlite`，避免 Windows 下 CGO/MinGW 依赖。
6. 前端采用 React 18 + TypeScript + Vite + Ant Design 5；用户给出的 React/Vue 二选一中本期选择 React。
7. 不修改 QEMU 源码子模块；服务端消费构建产物。
8. 9P 配置只在 VM 关机时修改；端口映射支持运行中动态增删。
9. 完整备份要求 VM 关机或暂停；恢复要求关机。
10. 内部快照仅支持 qcow2；raw 镜像可运行和完整备份，但不支持内部快照。
11. 单实例、单用户场景；不实现多租户、RBAC、登录认证。
12. 不内嵌浏览器 VNC/noVNC 控制台；本期提供 VNC 地址供外部客户端连接。

### 2.6 假设与依赖

| 编号 | 假设/依赖 |
| --- | --- |
| A-01 | 用户已获得可运行的 QEMU 与 qemu-img，或者在设置页配置正确路径 |
| A-02 | Windows 9P 功能依赖本仓库的 virtfs 补丁；标准 Windows QEMU 不保证支持 |
| A-03 | 用户以单实例方式运行服务端，不并发启动多个服务进程管理同一 data-dir |
| A-04 | 用户理解无认证的本地管理台风险，不将其暴露到不可信网络 |
| A-05 | 用户提供的镜像、9P 目录和固件路径存在且当前用户有访问权限 |
| A-06 | 宿主文件系统支持 SQLite WAL 与所需文件锁 |
| A-07 | 快照/备份所需的磁盘空间由用户保证 |
| A-08 | 开发/验收环境可下载 Go、Node、pnpm 依赖；QEMU 集成测试为可选 |

### 2.7 范围外

本期不包含：

- 浏览器内嵌 VNC/noVNC 控制台；
- 多主机调度、集群、迁移；
- 用户登录、RBAC、审计签名；
- 9P 设备热插拔；
- 在线整盘备份（VM 运行中）；
- 外部快照链、块提交、快照树管理；
- 数据库层面的自动备份/复制；
- 非 x86_64 客户机架构的专门适配；
- 网络桥接/TAP、自定义虚拟网络、SDN；
- 公网部署安全加固。

---

## 3. 功能需求

功能需求编号规则：`FR-XXX`。每条需求包含优先级、描述、输入、处理、输出、异常和验收要点。

### 3.1 FR-001 服务启动与浏览器访问

- 优先级：高
- 描述：服务端启动后应在配置的监听地址提供 HTTP 服务，默认 `127.0.0.1:8080`，浏览器访问根路径即可打开管理台。
- 输入：`--listen`、`--data-dir` 等命令行参数或环境变量。
- 处理：
  1. 解析参数，创建/校验 data-dir。
  2. 打开 SQLite 数据库并执行迁移。
  3. 加载设置和 VM 列表，进行状态恢复。
  4. 启动 HTTP 服务，注册 API 和静态资源。
  5. 打印访问 URL、data-dir、数据库路径和 QEMU 检测结果。
- 输出：浏览器可访问的管理台；`GET /api/health` 返回健康状态。
- 异常：
  - 端口被占用：启动失败并给出明确错误。
  - data-dir 不可写：启动失败。
  - 数据库无法打开/迁移失败：启动失败并保留原库。
  - QEMU 未配置：服务仍启动，UI 提示并禁用需要 QEMU 的操作。
- 验收：
  - AC-01：启动后浏览器打开默认地址能看到管理台。
  - 非回环监听未加 `--allow-remote` 时启动被拒绝；加上后有显著安全警告。

### 3.2 FR-002 QEMU/qemu-img 配置与检测

- 优先级：高
- 描述：服务端应能自动发现或通过设置指定 `qemu-system-x86_64` 和 `qemu-img`，并检测版本、加速器和设备能力。
- 输入：设置页路径、命令行参数、环境变量、自动发现候选路径。
- 处理：
  1. 发现顺序：`--qemu-bin`/`TVL_QEMU_BIN` > SQLite 设置 > 可执行文件同目录 > `tools/qemu/` > `qemu/` > `build/` > `qemu/build/` > PATH。
  2. qemu-img 默认取 QEMU 同目录，再查 PATH。
  3. 执行 `--version` 解析版本；执行 `-accel help` 解析加速器；执行 `-device help` 检测 `virtio-9p-pci`、`virtio-net-pci`、`virtio-blk-pci`。
  4. 设置保存时验证路径；提供“测试”功能不保存即可验证候选路径。
  5. 检测结果缓存，设置变更后失效。
- 输出：`GET /api/system` 返回路径、版本、可用性、加速器、设备能力；设置持久化到 SQLite。
- 异常：
  - 路径不存在或不可执行：设置保存返回 400；系统信息显示不可用。
  - 版本命令超时/失败：标记不可用，记录错误。
  - QEMU 缺少 9P 设备：UI 明确提示无法使用 9P，但其他功能可用。
- 验收：
  - AC-02：设置正确路径后系统信息显示版本和 `virtio-9p-pci` 能力。
  - 错误路径保存被拒绝，原设置不被覆盖。

### 3.3 FR-003 虚拟机参数配置

- 优先级：高
- 描述：用户可配置 VM 的基础硬件参数，支持创建时设置和关机时编辑。
- 输入：VM 名称、vCPU、内存、machine、CPU 模型、加速器、固件、启动顺序、显示模式、自动启动、关机超时。
- 处理与规则：
  - 名称：1–64 字符，允许字母、数字、空格、`.`、`_`、`-`，不区分大小写唯一；禁止逗号和控制字符。
  - vCPU：1–64。
  - 内存：128–1048576 MB。
  - machine：默认 `q35`，只允许 `[A-Za-z0-9._-]+`。
  - CPU 模型：可空（QEMU 默认），非空时允许 `[A-Za-z0-9._+-]+`。
  - 加速器：`auto`、`whpx`、`kvm`、`tcg`、`hvf`；`auto` 不传 `-accel`。
  - 固件：可选 `-bios` 文件绝对路径，必须存在且可读。
  - 启动顺序：`disk`、`cdrom`、`cdrom-first`、`disk-first`，映射为 QEMU `-boot order`。
  - 显示：`none` 或 `vnc`；VNC 端口 0 表示自动分配，绑定 `127.0.0.1`。
  - 自动启动：0/1。
  - 关机超时：10–600 秒，默认 60。
- 输出：参数写入 SQLite，并在启动时转换为 QEMU 参数。
- 异常：非法参数返回 400 和字段级错误；运行中编辑返回 409。
- 验收：
  - AC-03：创建时设置参数，启动参数与配置一致。
  - 运行中修改配置被拒绝。

### 3.4 FR-004 镜像路径与磁盘配置

- 优先级：高
- 描述：用户可为 VM 新建 qcow2 磁盘，或引用已有镜像文件；可选挂载 ISO 作为只读光盘。
- 输入：
  - 磁盘模式：新建 qcow2 / 引用已有镜像。
  - 新建：大小 1–65536 GB，默认 20 GB；路径由服务端生成在 `<data-dir>/images/<vm-id>/disk.qcow2`。
  - 已有镜像：绝对路径，必须存在、是普通文件、可读；使用 qemu-img 探测格式和虚拟大小。
  - ISO：可选绝对路径，只读，必须存在。
- 处理：
  1. 新建模式调用 `qemu-img create -f qcow2` 创建 managed 镜像。
  2. 已有镜像调用 `qemu-img info --output=json -U` 获取格式、大小、快照信息；支持 qcow2 和 raw。
  3. 禁止同一镜像路径被多个 VM 使用。
  4. 禁止把 `.iso` 作为可写磁盘；ISO 只能通过光盘字段挂载。
  5. managed 镜像删除 VM 时可按用户选择删除；外部镜像永不自动删除。
- 输出：磁盘路径、格式、大小、是否 managed 写入 SQLite；启动时生成 `-blockdev` JSON。
- 异常：
  - 镜像不存在/不可读/格式不支持：400。
  - 新建大小越界：400。
  - qemu-img 不可用：503，禁止创建需要探测或创建磁盘的 VM。
  - 路径含 QemuOpts 无法安全转义的字符（如逗号）时拒绝。
- 验收：
  - AC-03：新建 qcow2 成功，VM 可启动。
  - AC-04：引用已有镜像成功，格式/大小显示正确；ISO 作为光盘挂载。

### 3.5 FR-005 VM 创建、编辑与删除

- 优先级：高
- 描述：提供 VM 的完整 CRUD；创建时在同一数据库事务中写入 VM、端口映射和 9P 共享。
- 输入：创建/编辑表单、删除选项。
- 处理：
  1. 生成 VM ID 和 UUID。
  2. 校验名称、MAC、磁盘路径、端口、9P、镜像等。
  3. 创建磁盘（如需要）。
  4. 在事务中写入 `vms`、`port_forwards`、`shares`。
  5. 编辑仅允许 `stopped` 状态；编辑不删除旧磁盘文件。
  6. 删除仅允许 `stopped` 或已无存活进程的 `error` 状态；支持 `deleteDisk`（仅 managed）和 `deleteBackups`。
  7. 删除时先删数据库记录（级联子表），再尽力删除文件；外部镜像永不删除。
- 输出：新 VM 详情；编辑后的配置；删除确认。
- 异常：
  - 名称/UUID/MAC/磁盘路径重复：409。
  - 运行中编辑/删除：409。
  - 文件删除失败：记录警告/事件，不回滚数据库。
- 验收：
  - AC-03/AC-04：创建成功。
  - AC-19：重复名称、运行中删除等错误被正确拒绝。

### 3.6 FR-006 虚拟机生命周期操作

- 优先级：高
- 描述：提供启动、ACPI 关机、强制退出、暂停、恢复、重启操作。
- 输入：VM ID 和动作。
- 处理：
  - 启动：校验配置/镜像/9P/端口/镜像未被占用；分配 QMP/VNC 端口；启动 QEMU；等待 QMP 握手；查询状态。
  - 关机：QMP `system_powerdown`；超过 `shutdownTimeoutSec` 后 `quit`；再超过 10 秒强制终止。
  - 强制退出：QMP `quit`；超时强制终止。
  - 暂停/恢复：QMP `stop`/`cont`。
  - 重启：QMP `system_reset`。
  - 自动启动：服务端恢复完成后，对 `autoStart=true` 且 `stopped` 的 VM 后台启动。
- 输出：状态变化写入 SQLite；动作结果返回前端；操作期间 `Runtime.Operation` 可见。
- 异常：
  - 非法状态动作：409。
  - QEMU 启动失败：返回 QEMU stderr 尾部，状态置 error/stopped。
  - QMP 不可达：明确错误并提示查看日志/强制终止。
- 验收：
  - AC-05：启动/暂停/恢复/重启/关机/强制退出均生效，状态正确。
  - 启动失败有可诊断错误。

### 3.7 FR-007 QMP 对接与状态管理

- 优先级：高
- 描述：所有 VM 控制通过 QMP 完成；服务端维护 QMP 连接、状态轮询、事件处理和断线重连。
- 输入：VM 的 QMP 端口、命令、事件。
- 处理：
  1. TCP 连接 `127.0.0.1:<qmpPort>`，读取 greeting，发送 `qmp_capabilities`。
  2. 请求使用唯一 id，按 id 匹配响应；事件单独分发。
  3. 处理 `SHUTDOWN`、`POWERDOWN`、`STOP`、`RESUME`、`RESET`、`GUEST_PANICKED`、`BLOCK_IO_ERROR`、`JOB_STATUS_CHANGE`。
  4. 周期性 `query-status`（默认 2 秒）兜底；事件驱动状态更新优先。
  5. 连接断开后按需重连；QEMU 端 QMP server 允许重新接受连接。
  6. QMP 错误包含 class/desc，转换为 API 可读错误。
- 输出：实时状态、状态详情、事件记录。
- 异常：
  - 握手失败/超时：启动失败或标记连接异常。
  - 命令超时：返回错误，不无限阻塞。
  - 未预期断开：尝试重连；失败则标记 error 并提示。
- 验收：
  - AC-06：QMP 状态与操作一致；事件/轮询能反映 guest 关机。
  - 不提供任意 QMP/HMP 透传接口。

### 3.8 FR-008 端口映射

- 优先级：高
- 描述：配置 QEMU user-mode 网络的宿主到客户机端口转发，支持创建时配置和运行中动态增删。
- 输入：协议（tcp/udp）、宿主地址、宿主端口、客户机地址（可空）、客户机端口。
- 处理：
  1. 校验协议、端口 1–65535、IPv4 地址、同一 VM `(协议,宿主地址,宿主端口)` 唯一。
  2. 检查与 HTTP/QMP/VNC 端口不冲突；启动前用 `net.Listen`/`ListenPacket` 预检。
  3. 启动时生成 `-netdev user,id=net0,hostfwd=...`。
  4. 运行中通过 QMP `human-monitor-command` 调用 HMP `hostfwd_add`/`hostfwd_remove`。
  5. 成功后写 SQLite；失败返回错误，不静默修改。
- 输出：端口映射列表、启动参数、运行中增删结果。
- 异常：
  - 端口冲突/格式错误：400/409。
  - HMP 返回错误或 “not found”：返回失败；删除时若确认已不存在，可在返回警告后清理 DB 记录。
  - 运行中修改失败：保留原 DB 配置。
- 验收：
  - AC-07：启动后宿主端口可连接；运行中增删生效并持久化。
  - 端口冲突被拒绝。

### 3.9 FR-009 9P 文件映射

- 优先级：高
- 描述：将宿主目录通过 virtio-9p 共享给 Linux 客户机。
- 输入：宿主目录绝对路径、mount tag、安全模型、只读标志。
- 处理：
  1. 校验目录存在、是目录、路径不含逗号、mount tag 唯一且匹配 `^[A-Za-z0-9][A-Za-z0-9._-]{0,31}$`。
  2. 每个共享生成 `-fsdev local,id=fsdevN,path=...,security_model=...[,readonly=on]` 和 `-device virtio-9p-pci,fsdev=fsdevN,mount_tag=...`。
  3. 默认 `security_model=none`；可选 `mapped-xattr`、`passthrough`。
  4. 最多 16 个共享。
  5. 禁止共享 `<data-dir>` 及其子目录。
  6. 仅关机状态可修改；运行中修改返回 409。
  7. UI 显示挂载命令。
- 输出：启动参数、共享列表、挂载说明；启动后可通过 `query-pci` 自检 9P 设备。
- 异常：
  - 目录不存在/无权限/tag 冲突：400/409。
  - QEMU 不支持 9P：启动失败或能力检测提示。
- 验收：
  - AC-08：`query-pci` 含 `virtio-9p-pci`；有 Linux guest 时可实际 mount。
  - 运行中修改被拒绝。

### 3.10 FR-010 内部快照

- 优先级：高
- 描述：对 qcow2 磁盘创建内部快照，可选择是否包含 VM 运行状态，并支持列表、恢复、删除。
- 输入：快照名称、是否包含 VM 状态、VM 状态。
- 处理：
  - 仅 qcow2 支持；raw 返回 `supported=false` 或 400。
  - 运行/暂停 + 包含 VM 状态：QMP `snapshot-save`，`vmstate` 和 `devices` 均使用 `disk0`；轮询 `query-jobs`，完成后 `job-dismiss`；超时 `job-cancel`。
  - 运行/暂停 + 不包含 VM 状态：QMP `blockdev-snapshot-internal-sync`。
  - 关机：`qemu-img snapshot -c`；请求包含 VM 状态时返回 400。
  - 恢复：
    - 含 VM 状态且运行/暂停：先创建 `pre-restore-<UTC>` 安全快照，再 QMP `snapshot-load`。
    - 磁盘快照或关机状态：要求关机，使用 `qemu-img snapshot -a`；运行中恢复磁盘快照返回 409。
    - 安全快照失败则中止恢复。
  - 删除：运行/暂停 QMP `snapshot-delete`；关机 `qemu-img snapshot -d`。
  - 名称：1–64 字符，唯一，保留 `pre-restore-` 前缀给系统。
  - 每次创建/恢复/删除后同步 SQLite `snapshots` 表；镜像为最终权威，DB 为 UI 读取来源。
- 输出：快照列表、操作结果、Operation 进度。
- 异常：
  - 重复名称/非法名称：409/400。
  - 快照不存在：404。
  - 非 qcow2/磁盘空间不足/QMP 作业失败：返回明确错误。
  - DB 同步失败：返回 500；下一次列表同步可恢复一致。
- 验收：
  - AC-09：运行中创建含 VM 状态快照、列表、恢复、删除成功；关机磁盘快照成功；raw 被拒绝。

### 3.11 FR-011 完整镜像备份

- 优先级：高
- 描述：将 VM 磁盘导出为独立 qcow2 备份，支持创建、列表、下载、恢复、删除。
- 输入：备份名称、是否压缩、备份 ID。
- 处理：
  - 创建要求 VM `stopped` 或 `paused`；运行中返回 409。
  - 使用 `qemu-img convert -f <fmt> -O qcow2 [-c] [-U] <source> <tmp>`；paused 加 `-U`；完成后原子改名到 `<data-dir>/backups/<vm-id>/<backup-id>.qcow2`。
  - 创建成功后写 SQLite `backups`；DB 写失败删除文件并报错。
  - 列表读 DB 并校验文件存在，返回 `available`。
  - 下载使用流式响应和 `Content-Disposition`。
  - 恢复要求关机；先为当前磁盘创建安全备份，再转换到临时文件并原子替换；失败清理临时文件并保留安全备份。
  - 删除先删 DB 行，再删文件；文件删除失败记录警告。
  - 外部镜像恢复前 UI 必须提示会覆盖原镜像；服务端不删除外部镜像。
- 输出：备份列表、文件、下载流、恢复结果。
- 异常：
  - 运行中创建/恢复：409。
  - qemu-img 失败/空间不足：500/507（可用 500 + 详细错误）。
  - 备份文件缺失：`available=false`，恢复/下载返回 404/409。
- 验收：
  - AC-10：关机/暂停状态创建备份、下载、恢复、删除成功；运行中创建被拒绝；恢复前自动安全备份。

### 3.12 FR-012 日志查看

- 优先级：中
- 描述：查看 QEMU stderr 和串口日志的最后若干行。
- 输入：VM ID、日志类型 `qemu|serial`、尾部行数（默认 200，最大 2000）。
- 处理：
  1. 每次启动向 QEMU 日志追加分隔头；日志超过 10 MB 时启动前截断。
  2. 从文件尾部读取最后 N 行，避免整文件加载。
  3. 返回行数组和是否截断。
- 输出：`GET /api/vms/{id}/logs` JSON；UI 日志页可自动刷新。
- 异常：文件不存在返回空列表；读取失败返回 500；VM 不存在 404。
- 验收：
  - AC-11：启动日志、串口日志可见；自动刷新正常。

### 3.13 FR-013 SQLite 持久化

- 优先级：高
- 描述：所有结构化数据持久化到 SQLite 数据库，服务重启后数据完整保留。
- 输入：设置、VM 配置、运行状态、端口、9P、快照、备份、审计事件。
- 处理：
  - 数据库文件 `<data-dir>/tinyvirt.db`。
  - 使用 WAL、外键、busy_timeout、synchronous=NORMAL。
  - 使用迁移表管理 schema 版本。
  - 写操作使用事务；VM 与子表同事务。
  - 快照/备份元数据按第 4 章同步规则维护。
  - 结构化数据不得回退到 JSON 文件持久化。
- 输出：重启后设置、VM、状态、端口、9P、快照、备份、审计均存在。
- 异常：
  - 数据库无法打开/迁移失败：启动失败，不自动重建。
  - 数据库损坏：报错并给出路径，不静默清空。
  - 写失败：返回 500 `db_error`，不伪造成功。
- 验收：
  - AC-12：重启服务后数据一致；DB 中可查询到对应记录。
  - AC-22：迁移幂等、外键级联、唯一约束生效。

### 3.14 FR-014 服务重启接管

- 优先级：高
- 描述：服务端退出再启动后，能接管仍在运行的 QEMU 进程。
- 输入：SQLite 中保存的 PID、QMP 端口、VM 配置。
- 处理：
  1. 启动时遍历非 stopped 或有 PID/QMP 端口的 VM。
  2. 尝试连接 QMP，成功则查询状态并接管。
  3. QMP 不可达时检查 PID 是否存活；进程已死置 stopped，进程存活置 error 并提示孤儿进程。
  4. 仅在进程镜像名匹配 qemu-system 时允许强制终止，避免误杀。
  5. 服务端退出默认不停止 VM；可选 `--stop-vms-on-exit` 优雅停止。
- 输出：恢复后的状态、可继续操作的 VM。
- 异常：端口被其他进程占用、QMP 无响应、PID 重用；需通过进程名校验和错误提示降低风险。
- 验收：
  - AC-13：运行中的 VM 在服务重启后显示 running，可继续暂停/停止/快照。
  - 服务退出后 VM 继续运行（默认）。

### 3.15 FR-015 文件浏览

- 优先级：中
- 描述：为镜像、ISO、9P 目录、固件路径选择提供宿主文件系统浏览。
- 输入：路径（空表示根/盘符列表）、类型过滤 `dir|file|all`。
- 处理：
  1. 空路径返回 Windows 盘符或 Unix `/`。
  2. 返回当前路径、父目录、条目名称、绝对路径、是否目录、大小、修改时间。
  3. 目录优先排序；最多 2000 条。
  4. 校验路径为绝对路径并 Clean。
- 输出：JSON 目录列表；前端 Modal 选择。
- 异常：路径不存在/不是目录：400；无权限：403/500；非绝对路径：400。
- 验收：
  - AC-14：可选择盘符/目录/文件，镜像和 9P 表单可回填路径。

### 3.16 FR-016 系统信息

- 优先级：中
- 描述：提供应用、QEMU、qemu-img、SQLite、data-dir 和监听信息。
- 输入：无。
- 处理：汇总版本、路径、能力、DB 路径/大小/schema 版本、监听地址。
- 输出：`GET /api/system` JSON；设置页展示。
- 异常：QEMU 不可用时返回 `available=false` 和错误原因，不影响服务运行。
- 验收：
  - AC-15：系统信息与实际环境一致；DB 路径和 schema 版本可见。

### 3.17 FR-017 操作审计

- 优先级：中
- 描述：记录 VM 的关键操作和状态变化，持久化到 `vm_events` 表。
- 输入：动作（create/update/delete/start/stop/pause/resume/reset/port-add/port-remove/snapshot/backup/state-change/error 等）、状态、详情。
- 处理：
  1. 操作成功/失败后写入事件。
  2. 每个 VM 最多保留 10000 条，插入后裁剪旧记录。
  3. 提供 `GET /api/vms/{id}/events?limit=100`。
- 输出：事件列表；UI“操作记录”展示。
- 异常：事件写入失败不应阻断主操作，但需记录服务端日志。
- 验收：
  - AC-16：关键操作产生事件；重启后仍可查询。

### 3.18 FR-018 Web 管理台

- 优先级：高
- 描述：提供中文管理台，覆盖 Dashboard、创建/编辑、详情、设置。
- 输入：用户交互。
- 处理：
  - Dashboard：汇总卡片 + VM 表格 + 搜索/状态筛选 + 2 秒轮询。
  - 创建页：基本信息、磁盘/镜像、网络/端口、9P、显示、高级；支持“保存并启动”。
  - 详情页 Tabs：Overview、Ports、Shares、Snapshots、Backups、Logs/操作记录。
  - 设置页：QEMU/qemu-img、默认参数、系统信息、能力提示。
  - 状态颜色、危险操作确认、错误 toast、加载态、按状态/操作禁用按钮。
  - 路径输入带文件浏览器。
- 输出：可完成全部功能操作的浏览器界面。
- 异常：API 错误展示后端 message/details；404 时返回列表。
- 验收：
  - AC-17：全部页面可访问，关键操作可完成，移动端基本可用。

### 3.19 FR-019 REST API

- 优先级：高
- 描述：提供版本化前缀 `/api` 的 JSON API，供前端和自动化调用。
- 输入/输出：见 6.1 端点表。
- 处理：
  - JSON camelCase；时间 RFC3339 UTC。
  - 错误结构统一。
  - 请求体限制 1 MB，拒绝未知字段和尾部多余 JSON。
  - 静态资源与 API 路由分离；`/api/` 未匹配返回 JSON 404。
- 异常：见 FR-020 和附录 B。
- 验收：
  - AC-18：API 响应结构、状态码、错误格式符合约定。

### 3.20 FR-020 错误处理、状态冲突与并发

- 优先级：高
- 描述：所有操作必须有明确、可诊断、可恢复的错误处理；同一 VM 的长操作串行化。
- 输入：并发请求、非法参数、QEMU/DB/文件系统故障。
- 处理：
  - 每 VM 一把操作锁；启动/停止/快照/备份/恢复串行。
  - 长操作设置 Operation；其他动作返回 409。
  - 参数错误 400；不存在 404；状态冲突 409；QEMU/qemu-img 不可用 503；DB/内部错误 500。
  - QEMU 启动失败返回 stderr 尾部；QMP 错误保留 class/desc。
  - DB 写失败不伪造成功；事务回滚。
- 输出：统一错误 JSON，前端可显示。
- 异常：错误信息不得泄露敏感内容（本机路径可显示，因为用户需要诊断）。
- 验收：
  - AC-19：错误场景返回正确状态码和可读消息。
  - AC-20：并发操作不产生竞态或数据损坏。

---

## 4. 数据与持久化需求

### 4.1 数据分类

| 数据类型 | 存储位置 | 说明 |
| --- | --- | --- |
| 设置 | SQLite `settings` | QEMU/qemu-img 路径、默认参数 |
| VM 配置 | SQLite `vms` | 硬件、磁盘、ISO、显示、网络、超时、自动启动 |
| VM 运行状态 | SQLite `vms` | state、state_detail、pid、qmp_port、runtime_vnc_port、时间、last_error |
| 端口映射 | SQLite `port_forwards` | 协议、地址、端口 |
| 9P 共享 | SQLite `shares` | 宿主路径、mount tag、安全模型、只读 |
| 快照元数据 | SQLite `snapshots` | 以镜像为准，DB 缓存/查询 |
| 备份元数据 | SQLite `backups` | 文件路径、大小、来源 |
| 操作审计 | SQLite `vm_events` | 动作、状态、详情、时间 |
| 数据库版本 | SQLite `schema_migrations` | 迁移版本 |
| 磁盘镜像 | 文件系统 `<data-dir>/images` | 大文件，不入库 |
| 备份文件 | 文件系统 `<data-dir>/backups` | 大文件，不入库 |
| 日志 | 文件系统 `<data-dir>/logs` | QEMU/串口日志 |
| 运行 PID 文件 | 文件系统 `<data-dir>/run` | QEMU `-pidfile` |

### 4.2 SQLite 通用要求

1. 数据库文件：`<data-dir>/tinyvirt.db`。
2. 必须启用 WAL、外键、busy_timeout=5000ms、synchronous=NORMAL。
3. 单进程内使用单连接（`SetMaxOpenConns(1)`），避免写锁竞争；所有事务短小。
4. 迁移在启动时执行，失败则启动失败；不得自动删除或重建数据库。
5. 所有结构化数据必须入库；不得用 JSON 文件代替数据库。
6. 布尔值存 0/1；时间存 RFC3339Nano UTC 文本；快照 `icount` 以十进制文本保存，避免无符号 64 位溢出。
7. 不允许把镜像/备份等大文件以 BLOB 存入数据库。
8. 数据库损坏时给出明确错误和文件路径，保留现场供人工恢复。

### 4.3 数据实体

| 表 | 用途 | 关键字段 | 主要约束 |
| --- | --- | --- | --- |
| `schema_migrations` | 迁移版本 | version, name, applied_at | version 主键 |
| `settings` | 全局设置 | qemu_bin, qemu_img_bin, defaults, updated_at | 单行 `id=1` |
| `vms` | VM 主数据 | id, name, uuid, vcpus, memory_mb, machine, accelerator, disk_*, iso_path, display_*, network_*, mac, state, state_detail, pid, qmp_port, runtime_vnc_port, started_at, stopped_at, last_error, created_at, updated_at | id 主键；name/uuid/disk_path/mac 唯一；state 枚举；vcpus/memory 范围 |
| `port_forwards` | 端口映射 | id, vm_id, protocol, host_addr, host_port, guest_addr, guest_port | vm_id 外键级联；`(vm_id,protocol,host_addr,host_port)` 唯一；端口范围 |
| `shares` | 9P 共享 | id, vm_id, host_path, mount_tag, security_model, read_only | vm_id 外键级联；`(vm_id,mount_tag)` 唯一；security_model 枚举 |
| `snapshots` | 快照元数据 | vm_id, name, vm_state_size, date_*, vm_clock_*, icount, source, created_at, synced_at | `(vm_id,name)` 主键；vm_id 外键级联 |
| `backups` | 备份元数据 | id, vm_id, name, file_path, source_image, format, size_bytes, created_at | id 主键；vm_id 外键级联；file_path 唯一 |
| `vm_events` | 操作审计 | id, vm_id, action, status, detail, created_at | id 自增主键；vm_id 外键级联 |

关系：

```text
vms 1 ── N port_forwards
vms 1 ── N shares
vms 1 ── N snapshots       (复合主键 vm_id + name)
vms 1 ── N backups
vms 1 ── N vm_events
settings 单行
```

### 4.4 完整性与一致性

1. 唯一约束：
   - VM 名称不区分大小写唯一。
   - VM UUID 唯一。
   - VM 磁盘路径唯一，禁止同一镜像被多个 VM 使用。
   - VM MAC 唯一。
   - 同一 VM 端口映射 `(protocol,host_addr,host_port)` 唯一。
   - 同一 VM mount tag 唯一。
   - 同一 VM 快照名称唯一。
   - 备份文件路径唯一。
2. 外键：端口、共享、快照、备份、事件在 VM 删除时级联删除。
3. 事务：
   - 创建 VM + 端口 + 共享同一事务。
   - 更新 VM + 替换端口/共享同一事务。
   - 快照同步（删除旧行 + 插入新行）同一事务。
   - 备份创建：先写文件，再写 DB；DB 失败删除文件。
   - 备份删除：先删 DB，再删文件；文件失败记录警告。
4. 快照同步规则：
   - 镜像中的快照列表是最终权威。
   - 创建/恢复/删除/列表后调用 `SyncSnapshots`，事务内替换该 VM 的全部快照行。
   - `pre-restore-` 前缀记为 `source=safety`，其余为 `user`。
   - DB 同步失败时操作返回 500，但镜像变更保留，下次列表同步恢复一致。
5. 备份一致性：
   - DB 为主元数据源；列表时校验文件存在，缺失时返回 `available=false`。
   - 备份文件必须位于 `<data-dir>/backups/<vm-id>/` 下；删除/下载/恢复前校验路径前缀，防止目录穿越。
6. 设置：数据库只保留一行；不存在时由迁移插入默认值；更新使用 upsert。

### 4.5 迁移与版本

- 迁移文件位于 `internal/store/migrations/`，通过 `go:embed` 嵌入。
- 迁移版本从 1 开始，按版本号升序执行。
- 每次迁移在事务中执行，成功后写入 `schema_migrations`。
- 启动时比较已应用版本与嵌入文件版本；缺少的迁移执行，多余的版本视为不兼容并报错。
- 迁移必须幂等、可回滚（回滚以备份/人工处理为主，不自动破坏数据）。
- 迁移失败时服务端不启动，打印失败版本和错误。

### 4.6 删除与保留

| 对象 | 规则 |
| --- | --- |
| VM | 仅 stopped 或无存活进程的 error 可删；DB 级联删除；managed 镜像可选删除；外部镜像永不删除 |
| 端口/共享 | 随 VM 级联删除，或用户显式修改时替换/删除 |
| 快照 | 删除后同步 DB；`pre-restore-*` 可删除 |
| 备份 | 删除 DB 行后删除文件；文件删除失败记录警告 |
| 日志 | 每次启动可截断超过 10 MB 的文件；删除 VM 时尽力删除该 VM 日志 |
| 事件 | 每 VM 最多 10000 条，超出裁剪最旧记录 |
| 数据库 | 不自动清理；用户可备份整个 data-dir |

### 4.7 数据库故障处理

1. 打开失败：启动失败，输出路径和驱动错误。
2. 迁移失败：启动失败，保留原库。
3. 读写失败：API 返回 500 `db_error`，不伪造成功。
4. SQLITE_BUSY：依赖 busy_timeout 重试；仍失败返回 503/500 并提示重试。
5. 数据库损坏：启动失败并提示用户从 data-dir 备份恢复；不自动删除/重建。
6. 服务端退出：先停止 HTTP，再关闭 QMP 连接，最后关闭数据库；默认不停止 VM。

---

## 5. 非功能需求

### NFR-001 安全

- 默认监听 `127.0.0.1:8080`；非回环监听必须显式 `--allow-remote`，并打印“无认证”警告。
- QMP/VNC 仅绑定 `127.0.0.1`。
- 所有外部命令使用 argv，不经过 shell；禁止用户输入直接进入 shell。
- 磁盘参数使用 JSON blockdev；QemuOpts 路径拒绝逗号。
- 严格校验名称、端口、MAC、IP、枚举、路径。
- 不提供任意 QMP/HMP 透传接口。
- 禁止同一镜像多 VM 共用；外部镜像永不删除；9P 禁止共享 data-dir。
- 请求体限制 1 MB；错误信息不包含密钥（本项目无密钥）。
- 验收：AC-21。

### NFR-002 可靠性

- 数据库写入使用事务与唯一/外键约束。
- VM 状态机明确；非法迁移被拒绝。
- 长操作串行化；失败可诊断、可重试。
- 服务重启可接管运行中的 VM。
- QEMU 启动失败、QMP 断开、DB 错误均有明确处理。
- 验收：AC-12、AC-13、AC-19、AC-20。

### NFR-003 性能

- `GET /api/health` 在正常环境下 < 100 ms。
- 50 台 VM 的 `GET /api/vms` 在正常环境下 < 500 ms。
- VM 状态轮询间隔默认 2 秒；页面隐藏时可暂停轮询。
- QEMU 启动等待最多 20 秒；QMP 命令默认 30 秒超时；快照 10 分钟、备份 30 分钟。
- 长备份/快照不阻塞其他 VM 的 HTTP 请求；同一 VM 内串行。
- 日志尾部读取不加载整个文件。

### NFR-004 兼容性

- Windows 10/11 x86_64 优先；Linux x86_64 兼容。
- Go 1.26+；Node 24+；pnpm。
- SQLite 驱动 `modernc.org/sqlite` 纯 Go，无 CGO。
- 前端支持现代 Chromium/Firefox/Safari。
- QEMU 版本以本仓库固定 tag 为准；9P 需要 Windows 补丁。
- 验收：Windows 上构建/运行/集成；Linux 至少单元测试和构建通过。

### NFR-005 可维护性

- 分层：api / qemu / qmp / store / model / config / webui。
- SQL 与迁移显式可审计，不引入 ORM。
- 关键逻辑（参数构建、校验、QMP、快照/备份）有单元测试。
- 需求编号可追踪到实现与测试。
- 代码通过 `go vet`、`go test`、前端类型检查和构建。

### NFR-006 易用性

- 中文界面，Ant Design 组件，状态颜色和图标清晰。
- 危险操作二次确认。
- 错误显示可读 message 和 details。
- 路径选择提供文件浏览器。
- 9P 提供挂载命令，端口映射提供常用预设。
- 响应式布局，窄屏可用。

### NFR-007 可观测性

- 服务端结构化日志：时间、级别、请求、VM、动作、结果。
- QEMU stderr 和串口日志保留，可在 UI 查看尾部。
- 操作审计写入 `vm_events`。
- 启动失败包含 QEMU stderr 尾部；QMP 错误包含 class/desc。
- `GET /api/health`、`GET /api/system` 可用于诊断。

### NFR-008 数据一致性

- 唯一约束、外键、事务保证核心数据一致性。
- 快照以镜像为权威，DB 同步；备份以 DB 元数据 + 文件存在性为准。
- 删除 VM 时 DB 级联；文件删除失败不回滚 DB，但记录警告。
- 外部镜像和 data-dir 共享保护规则防止数据被误删。
- 验收：AC-12、AC-22。

---

## 6. 外部接口需求

### 6.1 REST API

所有 API 以 `/api` 为前缀，返回 JSON。错误结构：

```json
{
  "error": {
    "code": "conflict",
    "message": "虚拟机正在运行",
    "details": "当前状态 running，无法编辑 9P 配置"
  }
}
```

端点总表：

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/health` | 健康检查 |
| GET | `/api/system` | 系统、QEMU、qemu-img、SQLite 信息 |
| GET | `/api/settings` | 读取设置 |
| PUT | `/api/settings` | 保存并验证设置 |
| POST | `/api/system/test-qemu` | 测试候选 QEMU/qemu-img 路径 |
| GET | `/api/fs/list` | 文件/目录浏览 |
| GET | `/api/vms` | VM 列表 |
| POST | `/api/vms` | 创建 VM |
| GET | `/api/vms/{id}` | VM 详情 |
| PUT | `/api/vms/{id}` | 更新配置（关机/error 且无进程） |
| DELETE | `/api/vms/{id}` | 删除 VM，支持 `deleteDisk`、`deleteBackups` |
| POST | `/api/vms/{id}/start` | 启动 |
| POST | `/api/vms/{id}/shutdown` | ACPI 关机 |
| POST | `/api/vms/{id}/poweroff` | 强制退出 |
| POST | `/api/vms/{id}/pause` | 暂停 |
| POST | `/api/vms/{id}/resume` | 恢复 |
| POST | `/api/vms/{id}/reset` | 重启 |
| GET | `/api/vms/{id}/status` | 实时 QMP 状态 |
| GET | `/api/vms/{id}/port-forwards` | 端口映射列表 |
| POST | `/api/vms/{id}/port-forwards` | 新增端口映射 |
| DELETE | `/api/vms/{id}/port-forwards/{pfId}` | 删除端口映射 |
| GET | `/api/vms/{id}/snapshots` | 内部快照列表 + 支持信息 |
| POST | `/api/vms/{id}/snapshots` | 创建快照 `{name, includeVmState}` |
| POST | `/api/vms/{id}/snapshots/{name}/restore` | 恢复快照 |
| DELETE | `/api/vms/{id}/snapshots/{name}` | 删除快照 |
| GET | `/api/vms/{id}/backups` | 备份列表 |
| POST | `/api/vms/{id}/backups` | 创建备份 `{name, compress}` |
| POST | `/api/vms/{id}/backups/{backupId}/restore` | 恢复备份 |
| DELETE | `/api/vms/{id}/backups/{backupId}` | 删除备份 |
| GET | `/api/vms/{id}/backups/{backupId}/download` | 下载备份 |
| GET | `/api/vms/{id}/logs` | QEMU/串口日志尾部 |
| GET | `/api/vms/{id}/events` | 操作记录 |

约定：

- 请求体上限 1 MB。
- `Content-Type: application/json`。
- 使用 `DisallowUnknownFields`，拒绝尾部多余 JSON。
- 时间格式 RFC3339 UTC。
- 静态资源 `/`、`/assets/*` 由内嵌前端提供；SPA 路由回退到 `index.html`；`/api/` 未匹配返回 JSON 404。
- 长操作（快照/备份）在 HTTP 请求内完成并返回结果；期间通过 VM 列表/详情的 `operation` 展示进度；服务端不设置过短的写超时。

### 6.2 QMP 接口

| 命令/事件 | 用途 |
| --- | --- |
| `qmp_capabilities` | 握手 |
| `query-status` | 查询运行状态 |
| `system_powerdown` | ACPI 关机 |
| `quit` | 退出 QEMU |
| `stop` / `cont` | 暂停/恢复 |
| `system_reset` | 重启 |
| `snapshot-save` | 创建含 VM 状态的内部快照（异步 job） |
| `snapshot-load` | 恢复含 VM 状态的内部快照（异步 job） |
| `snapshot-delete` | 删除内部快照（异步 job） |
| `blockdev-snapshot-internal-sync` | 创建磁盘内部快照 |
| `query-jobs` | 查询快照任务状态 |
| `job-dismiss` / `job-cancel` | 结束/取消任务 |
| `query-named-block-nodes` | 获取 `disk0` 的快照列表 |
| `query-block` | 回退获取块设备信息 |
| `query-pci` | 自检 9P 设备 |
| `human-monitor-command` | 调用 `hostfwd_add`/`hostfwd_remove` |
| `SHUTDOWN` | guest 关机事件 |
| `POWERDOWN` | 电源事件 |
| `STOP` / `RESUME` | 暂停/恢复事件 |
| `RESET` | 重启事件 |
| `GUEST_PANICKED` | guest panic |
| `BLOCK_IO_ERROR` | 块 I/O 错误 |
| `JOB_STATUS_CHANGE` | 快照任务状态变化 |

### 6.3 qemu-img 接口

| 操作 | 典型命令 | 用途 |
| --- | --- | --- |
| 创建 | `qemu-img create -f qcow2 <path> <size>` | 新建 managed qcow2 |
| 探测 | `qemu-img info --output=json -U <path>` | 获取格式、大小、快照 |
| 关机快照创建 | `qemu-img snapshot -c <name> <path>` | 磁盘内部快照 |
| 关机快照恢复 | `qemu-img snapshot -a <name> <path>` | 恢复磁盘快照 |
| 关机快照删除 | `qemu-img snapshot -d <name> <path>` | 删除快照 |
| 完整备份 | `qemu-img convert -f <fmt> -O qcow2 [-c] [-U] <src> <dst>` | 独立 qcow2 备份 |
| 恢复备份 | `qemu-img convert -O <targetFormat> <backup> <tmp>` | 恢复磁盘 |
| 安全备份 | 同上，来源为当前磁盘 | 恢复前保护 |

### 6.4 命令行参数与环境变量

| 参数 | 环境变量 | 说明 |
| --- | --- | --- |
| `--listen` | `TVL_LISTEN` | 默认 `127.0.0.1:8080` |
| `--data-dir` | `TVL_DATA_DIR` | 默认 `./data` |
| `--qemu-bin` | `TVL_QEMU_BIN` | QEMU 可执行文件 |
| `--qemu-img` | `TVL_QEMU_IMG` | qemu-img 可执行文件 |
| `--allow-remote` | 无 | 允许非回环监听，打印安全警告 |
| `--stop-vms-on-exit` | 无 | 退出时优雅停止 VM，默认关闭 |
| `--open` | 无 | 启动后尝试打开浏览器 |

### 6.5 文件系统布局

```text
<data-dir>/
  tinyvirt.db
  tinyvirt.db-wal
  tinyvirt.db-shm
  images/
    <vm-id>/
      disk.qcow2
  backups/
    <vm-id>/
      <backup-id>.qcow2
  logs/
    <vm-id>-qemu.log
    <vm-id>-serial.log
  run/
    <vm-id>.pid
```

- 外部镜像可位于任意绝对路径。
- 9P 宿主目录可位于任意绝对路径，但不得位于 `<data-dir>` 下。
- 删除 VM 时：DB 记录级联删除；managed 镜像、备份、日志按用户选项/策略尽力删除。

---

## 7. 状态机与业务规则

### 7.1 VM 状态

| 状态 | 含义 |
| --- | --- |
| `stopped` | 未运行，无 QEMU 进程 |
| `starting` | 进程已启动，等待 QMP 就绪 |
| `running` | QMP 报告 running |
| `paused` | QMP 报告 paused/suspended/debug |
| `stopping` | 已请求关机/退出，等待进程结束 |
| `error` | 启动失败、QMP 不可达、guest panic、进程异常等 |

`state_detail` 保存 QMP 原始 status，便于诊断。

### 7.2 状态迁移

```text
stopped --start--> starting --QMP就绪--> running
running --pause--> paused --resume--> running
running/paused --shutdown/poweroff--> stopping --> stopped
starting/running/paused --失败--> error
error --start--> starting
```

禁止的迁移返回 409。

### 7.3 动作矩阵

| 动作 | stopped | starting | running | paused | stopping | error（无进程） | error（有进程） |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 启动 | 允许 | 拒绝 | 拒绝 | 拒绝 | 拒绝 | 允许 | 拒绝，先强制退出 |
| ACPI 关机 | 拒绝 | 拒绝 | 允许 | 允许 | 拒绝 | 拒绝 | 拒绝 |
| 强制退出 | 拒绝 | 拒绝 | 允许 | 允许 | 允许 | 拒绝 | 允许 |
| 暂停 | 拒绝 | 拒绝 | 允许 | 拒绝 | 拒绝 | 拒绝 | 拒绝 |
| 恢复 | 拒绝 | 拒绝 | 拒绝 | 允许 | 拒绝 | 拒绝 | 拒绝 |
| 重启 | 拒绝 | 拒绝 | 允许 | 允许 | 拒绝 | 拒绝 | 拒绝 |
| 编辑配置 | 允许 | 拒绝 | 拒绝 | 拒绝 | 拒绝 | 允许（无进程） | 拒绝 |
| 删除 VM | 允许 | 拒绝 | 拒绝 | 拒绝 | 拒绝 | 允许（无进程） | 拒绝 |
| 新增/删除端口 | 允许 | 拒绝 | 允许 | 允许 | 拒绝 | 拒绝 | 拒绝 |
| 编辑 9P | 允许 | 拒绝 | 拒绝 | 拒绝 | 拒绝 | 允许（无进程） | 拒绝 |
| 创建快照 | 允许（仅磁盘） | 拒绝 | 允许 | 允许 | 拒绝 | 拒绝 | 拒绝 |
| 恢复快照 | 允许（qcow2） | 拒绝 | 允许（含 VM 状态） | 允许（含 VM 状态） | 拒绝 | 拒绝 | 拒绝 |
| 删除快照 | 允许 | 拒绝 | 允许 | 允许 | 拒绝 | 拒绝 | 拒绝 |
| 创建备份 | 允许 | 拒绝 | 拒绝 | 允许 | 拒绝 | 拒绝 | 拒绝 |
| 恢复备份 | 允许 | 拒绝 | 拒绝 | 拒绝 | 拒绝 | 拒绝 | 拒绝 |
| 删除备份 | 允许 | 允许 | 允许 | 允许 | 允许 | 允许 | 允许 |
| 查看日志/事件 | 允许 | 允许 | 允许 | 允许 | 允许 | 允许 | 允许 |

说明：

- 运行中恢复“仅磁盘快照”不允许，需先关机，因为 `qemu-img snapshot -a` 不能安全用于运行中的镜像。
- 运行中恢复含 VM 状态的快照由 QMP `snapshot-load` 完成，QEMU 会暂停/恢复 CPU。
- 恢复前自动安全快照失败则中止恢复。

### 7.4 并发规则

1. 每个 VM 一把操作锁，启动、停止、快照、备份、恢复互斥。
2. 同一 VM 长操作期间，其他动作返回 409 `operation_in_progress`。
3. 不同 VM 的操作可并行。
4. 全局 Store 锁只保护短事务，不覆盖 QMP 长调用。
5. 服务端重启恢复期间，API 可读；针对未完成恢复的 VM 的动作返回 409。

### 7.5 配置与校验规则

- 名称：1–64 字符，允许字母/数字/空格/`.`/`_`/`-`，禁止逗号和控制字符，不区分大小写唯一。
- vCPU 1–64；内存 128–1048576 MB。
- 端口 1–65535；hostAddr/guestAddr 为 IPv4 或空。
- MAC 格式 `XX:XX:XX:XX:XX:XX`，单播，VM 间唯一；默认自动生成。
- mount tag：`^[A-Za-z0-9][A-Za-z0-9._-]{0,31}$`，同一 VM 唯一。
- 安全模型：`none`、`mapped-xattr`、`passthrough`。
- 加速器：`auto`、`whpx`、`kvm`、`tcg`、`hvf`。
- 显示：`none`、`vnc`；VNC 绑定 loopback。
- 路径：绝对路径；镜像/ISO/固件必须存在；9P 必须是目录；QemuOpts 路径拒绝逗号。
- 禁止同一磁盘路径多 VM；禁止 9P 共享 data-dir。
- 删除外部镜像永远禁止；managed 镜像仅在删除 VM 且用户勾选时删除。

### 7.6 快照/备份决策表

| VM 状态 | 快照含 VM 状态 | 创建方式 | 恢复方式 |
| --- | --- | --- | --- |
| running | 是 | QMP `snapshot-save` | QMP `snapshot-load`（先安全快照） |
| running | 否 | QMP `blockdev-snapshot-internal-sync` | 需先关机，`qemu-img snapshot -a` |
| paused | 是 | QMP `snapshot-save` | QMP `snapshot-load`（先安全快照） |
| paused | 否 | QMP `blockdev-snapshot-internal-sync` | 需先关机，`qemu-img snapshot -a` |
| stopped | 否 | `qemu-img snapshot -c` | `qemu-img snapshot -a` |
| stopped | 是 | 不允许，返回 400 | — |
| raw 磁盘 | 任意 | 不支持 | 不支持 |

| VM 状态 | 创建完整备份 | 恢复完整备份 |
| --- | --- | --- |
| running | 拒绝（409） | 拒绝（409） |
| paused | 允许（`qemu-img -U`） | 拒绝（409） |
| stopped | 允许 | 允许（先安全备份） |

---

## 8. 验收标准

### 8.1 验收场景

| 编号 | 场景 | 预期结果 |
| --- | --- | --- |
| AC-01 | 启动服务并访问浏览器 | 管理台加载；health 正常；默认回环监听 |
| AC-02 | 配置 QEMU/qemu-img 并测试 | 显示版本、加速器、`virtio-9p-pci` 能力；错误路径被拒绝 |
| AC-03 | 新建 qcow2 VM | VM 写入 SQLite；可启动；参数与配置一致 |
| AC-04 | 引用已有镜像并挂载 ISO | 格式/大小正确；ISO 作为只读光盘；外部镜像不被删除 |
| AC-05 | 启动/暂停/恢复/重启/关机/强制退出 | 状态正确；超时策略生效 |
| AC-06 | QMP 状态与事件 | 状态与 QEMU 一致；guest 关机可被感知 |
| AC-07 | 端口映射 | 启动后宿主端口可连接；运行中增删生效并持久化；冲突被拒绝 |
| AC-08 | 9P 共享 | `query-pci` 含 `virtio-9p-pci`；UI 显示挂载命令；运行中修改被拒绝 |
| AC-09 | 内部快照 | 运行中与关机场景符合决策表；恢复前自动安全快照；raw 被拒绝 |
| AC-10 | 完整备份 | 关机/暂停可创建；可下载/恢复/删除；运行中创建被拒绝；恢复前安全备份 |
| AC-11 | 日志 | QEMU/串口日志尾部可见；自动刷新正常 |
| AC-12 | SQLite 持久化 | 重启后设置、VM、状态、端口、9P、快照、备份、事件均保留 |
| AC-13 | 服务重启接管 | 运行中的 VM 显示 running，可继续操作；默认不杀 VM |
| AC-14 | 文件浏览 | 可浏览盘符/目录/文件并回填路径 |
| AC-15 | 系统信息 | QEMU/DB/data-dir/监听信息正确 |
| AC-16 | 操作审计 | 关键操作产生事件；重启后仍可查询 |
| AC-17 | Web 管理台 | 页面可访问；关键操作可完成；危险操作有确认 |
| AC-18 | REST API 约定 | JSON/时间/错误结构/状态码符合约定 |
| AC-19 | 错误场景 | 参数、冲突、QEMU/DB 故障返回正确错误，不静默失败 |
| AC-20 | 并发/长操作 | 同一 VM 动作互斥；不同 VM 可并行；无数据损坏 |
| AC-21 | 安全约束 | 非回环需 `--allow-remote`；QMP/VNC 回环；无 shell 拼接 |
| AC-22 | 数据一致性 | 迁移幂等、外键级联、唯一约束、事务回滚、快照/备份同步正确 |

### 8.2 需求追踪矩阵

| 需求 | 验收场景 | 测试类型 |
| --- | --- | --- |
| FR-001 | AC-01、AC-21 | 人工、API |
| FR-002 | AC-02 | API、人工 |
| FR-003 | AC-03、AC-19 | 单元、API |
| FR-004 | AC-03、AC-04 | 单元、集成、人工 |
| FR-005 | AC-03、AC-19 | 单元、仓储、API |
| FR-006 | AC-05 | 集成、人工 |
| FR-007 | AC-06 | 单元（fake QMP）、集成 |
| FR-008 | AC-07 | 单元、集成、人工 |
| FR-009 | AC-08 | 单元、集成、人工 |
| FR-010 | AC-09 | 单元、集成、人工 |
| FR-011 | AC-10 | 单元、集成、人工 |
| FR-012 | AC-11 | API、人工 |
| FR-013 | AC-12、AC-22 | 仓储、集成、人工 |
| FR-014 | AC-13 | 集成、人工 |
| FR-015 | AC-14 | API、人工 |
| FR-016 | AC-15 | API、人工 |
| FR-017 | AC-16 | 仓储、API |
| FR-018 | AC-17 | 前端构建、人工 |
| FR-019 | AC-18 | API |
| FR-020 | AC-19、AC-20 | 单元、API、人工 |
| NFR-001 | AC-21 | 人工、代码审查 |
| NFR-002 | AC-12、AC-13、AC-19、AC-20 | 仓储、集成 |
| NFR-003 | AC-06、AC-17 | 性能抽测、人工 |
| NFR-004 | AC-01、AC-22 | CI、人工 |
| NFR-005 | 全部 | CI、代码审查 |
| NFR-006 | AC-17 | 人工 |
| NFR-007 | AC-11、AC-16 | 人工、API |
| NFR-008 | AC-12、AC-22 | 仓储、集成 |

### 8.3 测试策略

1. **单元测试**：参数构建、校验、QMP 协议、快照/备份纯逻辑、错误映射。
2. **仓储测试**：临时 SQLite 文件、迁移幂等、唯一/外键约束、事务回滚、快照同步、事件裁剪。
3. **API 测试**：httptest，覆盖健康、空列表、404、400、设置测试、错误 JSON、静态回退。
4. **集成测试**：使用本仓库 Release QEMU 产物（可选下载）执行启动/暂停/快照/备份/重启接管；无 QEMU 时跳过。
5. **前端测试**：`tsc` 类型检查、`vite build`；人工走查关键页面。
6. **人工验收**：按 8.1 场景逐项执行；在 Windows 上验证 9P 设备启动和 SQLite 持久化。

---

## 9. 约束、风险与限制

### 9.1 已知限制

1. 浏览器不提供 VM 图形/串口控制台；VNC 需外部客户端连接 loopback 地址。
2. 9P 配置只在关机时生效；运行中修改返回 409。
3. 内部快照仅限 qcow2；raw 只能完整备份。
4. 完整备份要求关机/暂停；恢复要求关机。
5. 端口映射仅支持 QEMU user-mode 网络，不支持桥接/TAP。
6. Windows 9P 下 POSIX 符号链接、权限、xattr 语义受限；默认 `security_model=none`。
7. 无登录认证；仅适合本机可信环境。
8. 单实例管理同一 data-dir；不支持多服务进程并发管理。
9. 快照/备份为同步 HTTP 操作，超大镜像可能耗时较长；期间 UI 显示 Operation。
10. HMP `hostfwd_add` 的失败信息可能只写 stderr，服务端采用预检 + 输出解析双保险，极端竞态下仍需查看日志。
11. SQLite 数据库不自动备份；用户应备份整个 data-dir。

### 9.2 风险

| 风险 | 影响 | 缓解 |
| --- | --- | --- |
| 本仓库 Windows 9P 补丁未在目标机验证 | 9P 不可用 | 用 Release 产物集成验证；能力检测提示 |
| QEMU 未安装/路径错误 | 无法启动 VM | 设置页自动发现、测试、明确提示 |
| WHPX/KVM 不可用 | 启动失败或性能低 | 支持 `auto`/`tcg`；返回 QEMU stderr |
| 端口被占用 | 启动或映射失败 | 启动前预检；409/500 明确错误 |
| 镜像被其他进程占用 | QEMU 启动失败 | 返回锁错误；禁止同镜像多 VM |
| 磁盘空间不足 | 快照/备份/创建失败 | 错误返回；不删除已有数据 |
| SQLite 损坏/迁移失败 | 服务无法启动 | 保留原库、明确错误、建议备份 data-dir |
| 服务端崩溃 | VM 可能继续运行 | 启动时 QMP 接管；PID 存活检查 |
| 无认证 | 被本机其他用户控制 | 默认回环；非回环显式开关+警告 |
| 长备份 HTTP 超时 | 前端等待失败 | 服务端放宽写超时；UI 显示进度；文档说明 |

---

## 10. 未来扩展

- 浏览器内嵌 noVNC 控制台。
- 登录认证、RBAC、操作审批。
- 在线整盘备份（QMP `blockdev-backup`）和外部快照链。
- 定时快照/备份、保留策略。
- 多主机管理、VM 迁移。
- 网络桥接/TAP、端口映射可视化。
- SQLite 数据库自动备份/导出/恢复。
- 快照树、快照导出为独立镜像。
- Vue 前端替代（API 不变）。
- 资源配额、镜像库、模板/克隆。

---

## 附录 A：API 端点明细

| 方法 | 路径 | 请求/参数 | 响应 |
| --- | --- | --- | --- |
| GET | `/api/health` | 无 | `{status, version}` |
| GET | `/api/system` | 无 | 系统/QEMU/DB/能力信息 |
| GET | `/api/settings` | 无 | 设置 |
| PUT | `/api/settings` | 设置对象 | 保存后的设置 |
| POST | `/api/system/test-qemu` | `{qemuBin,qemuImgBin}` | 版本/能力/错误 |
| GET | `/api/fs/list` | `path`, `kind` | 目录条目 |
| GET | `/api/vms` | 无 | VM 摘要列表 |
| POST | `/api/vms` | VM 配置 | 创建的 VM |
| GET | `/api/vms/{id}` | 无 | VM 详情 |
| PUT | `/api/vms/{id}` | VM 配置 | 更新后的 VM |
| DELETE | `/api/vms/{id}` | `deleteDisk`, `deleteBackups` | 204/结果 |
| POST | `/api/vms/{id}/start` | 无 | 操作结果 |
| POST | `/api/vms/{id}/shutdown` | 无 | 操作结果 |
| POST | `/api/vms/{id}/poweroff` | 无 | 操作结果 |
| POST | `/api/vms/{id}/pause` | 无 | 操作结果 |
| POST | `/api/vms/{id}/resume` | 无 | 操作结果 |
| POST | `/api/vms/{id}/reset` | 无 | 操作结果 |
| GET | `/api/vms/{id}/status` | 无 | QMP 状态 |
| GET | `/api/vms/{id}/port-forwards` | 无 | 映射列表 |
| POST | `/api/vms/{id}/port-forwards` | 映射对象 | 新增结果 |
| DELETE | `/api/vms/{id}/port-forwards/{pfId}` | 无 | 204/结果 |
| GET | `/api/vms/{id}/snapshots` | 无 | `{supported, reason, snapshots}` |
| POST | `/api/vms/{id}/snapshots` | `{name, includeVmState}` | 快照/结果 |
| POST | `/api/vms/{id}/snapshots/{name}/restore` | 无 | 恢复结果 |
| DELETE | `/api/vms/{id}/snapshots/{name}` | 无 | 204/结果 |
| GET | `/api/vms/{id}/backups` | 无 | 备份列表 |
| POST | `/api/vms/{id}/backups` | `{name, compress}` | 备份/结果 |
| POST | `/api/vms/{id}/backups/{backupId}/restore` | 无 | 恢复结果 |
| DELETE | `/api/vms/{id}/backups/{backupId}` | 无 | 204/结果 |
| GET | `/api/vms/{id}/backups/{backupId}/download` | 无 | 文件流 |
| GET | `/api/vms/{id}/logs` | `kind`, `tail` | 日志行 |
| GET | `/api/vms/{id}/events` | `limit` | 审计事件 |

## 附录 B：错误码

| 错误码 | HTTP | 含义 |
| --- | --- | --- |
| `invalid_argument` | 400 | 参数/路径/端口/名称等校验失败 |
| `not_found` | 404 | VM/快照/备份/文件不存在 |
| `conflict` | 409 | 状态冲突、名称/端口/镜像重复 |
| `operation_in_progress` | 409 | 同一 VM 有长操作进行中 |
| `permission_denied` | 403 | 文件/目录无权限 |
| `qemu_unavailable` | 503 | QEMU/qemu-img 不可用 |
| `qmp_error` | 502/500 | QMP 命令失败 |
| `db_error` | 500 | SQLite 读写/迁移失败 |
| `unsupported` | 400 | raw 快照等不支持的操作 |
| `internal_error` | 500 | 未分类内部错误 |

## 附录 C：术语表

| 术语 | 解释 |
| --- | --- |
| ACPI 关机 | 通过 `system_powerdown` 向 guest 发送电源事件 |
| 强制退出 | 通过 QMP `quit` 结束 QEMU 进程 |
| 安全快照 | 恢复前自动创建的 `pre-restore-*` 快照 |
| 安全备份 | 恢复完整备份前对当前磁盘创建的独立备份 |
| Operation | VM 当前长操作的类型/进度，仅在内存中，API 可见 |
| managed 镜像 | 服务端创建并管理的镜像 |
| 外部镜像 | 用户提供路径、服务端只引用的镜像 |
| loopback | 127.0.0.1 / ::1 本机回环 |
| WAL | SQLite 预写日志模式 |

## 附录 D：Web 页面清单

| 路由 | 页面 | 主要内容 |
| --- | --- | --- |
| `/` | Dashboard | 汇总卡片、VM 表格、搜索/筛选、操作 |
| `/vms/new` | 创建 VM | 基本信息、磁盘/镜像、网络/端口、9P、显示、高级 |
| `/vms/:id` | VM 详情 | 状态/动作、Overview、Ports、Shares、Snapshots、Backups、Logs/操作记录 |
| `/settings` | 设置 | QEMU/qemu-img、默认参数、系统信息、能力提示 |
| 全局 | 文件浏览器 Modal | 盘符/目录/文件选择 |
| 全局 | 错误/确认弹窗 | 错误详情、危险操作确认 |

---

**文档结束。** 本文件为 TinyVirt 需求基线；任何需求变更必须先更新本文档并重新评审。
