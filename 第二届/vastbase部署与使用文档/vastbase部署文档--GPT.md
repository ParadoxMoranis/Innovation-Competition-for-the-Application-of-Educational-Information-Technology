# vastbase部署文档--GPT

# 1) 安装前准备（硬件、系统、账号与目录）

**硬件最低建议**（按测试/生产区分，生产更保守）：内存单机 8G+（建议≥128G），CPU 16 核+，安装程序约 800MB、元数据约 750MB，预留≥70%磁盘存放业务数据；网络：单机千兆+、集群万兆+。

**依赖与系统支持**：不同发行版需准备的依赖包与 Python 版本（openEuler 3.7.x、CentOS 3.6.x、Kylin 3.7.x…；通用依赖含 zlib-devel、libaio、libuuid、readline-devel、krb5-libs、libicu、libxslt、tcl、perl、openldap、pam、openssl-devel、libxml2、bzip2、libnsl 等；注：libnsl 仅 openEuler+x86 需要）。

**创建运行用户与目录规划**（推荐专用户/组与分离的软件/数据目录）：

```bash
# 以root
groupadd dbgrp
useradd -g dbgrp vastbase
id vastbase

# 规划目录（示例）
mkdir -p /vastdata/app/vastbase/2.2.15
mkdir -p /vastdata/data/vastbase
chown -R vastbase:dbgrp /vastdata
```

---

# 2) 系统参数与内核调优（必须/可选）

**必须项**

- 关闭防火墙、禁用 SELinux：
  
  ```bash
  systemctl stop firewalld && systemctl disable firewalld
  # /etc/selinux/config: 设 SELINUX=disabled ；重启生效
  ```

- 禁用 RemoveIPC（避免用户退出时清理 IPC 导致实例崩溃）：  
  在 `/etc/systemd/logind.conf` 明确 `RemoveIPC=no`，并在 `/usr/lib/systemd/system/systemd-logind.service` 里也明确设置；随后：
  
  ```bash
  systemctl daemon-reload
  systemctl restart systemd-logind
  ```
  
  安装器的环境检查也会提示并给出同样的修改建议。

**可选项（按需做）**

- 统一字符集与时区/NTP，同步各节点时钟；关闭 shell 历史；内存足够时可关闭 swap；集群建议禁用 `avahi-daemon` 以免 VIP 不稳。对应示例：
  
  ```bash
  # 统一字符集（/etc/profile）
  export LANG=<Unicode编码>
  # 设置时区
  cp /usr/share/zoneinfo/<Region>/<City> /etc/localtime
  # 关闭history
  echo 'HISTSIZE=0' >> /etc/profile
  # 关闭swap（内存足够）
  swapoff -a
  # 停用 avahi（如启用会影响VIP）
  systemctl disable --now avahi-daemon
  ```

**内核参数（推荐值 & 计算方法）**  
将以下写入 `/etc/sysctl.conf` 并 `sysctl -p` 生效（示例值按文档建议；`shmmax/shmall` 需按机器内存计算）：

```conf
fs.aio-max-nr = 1048576            # 最大异步IO请求数
fs.file-max   = 2097152            # 视库规模而定，这里示例
fs.nr_open    = 12000000           # 单进程最大FD，库对象多可适当增大

# 信号量(SEMMSL SEMMNS SEMOPM SEMMNI)
kernel.sem = 4096 2097152000 4096 5120

# 共享内存：shmmax=物理内存50%，shmall=物理内存的80%/PAGE_SIZE
kernel.shmmax  = <RAM_bytes * 0.5>
kernel.shmall  = <RAM_bytes * 0.8 / 4096>   # 若页大小4KB
kernel.shmmni  = 4096                       # 可用默认或按需
vm.swappiness  = 10                         # 10~20，尽量少用swap
```

这些参数及推荐值/计算口径见“系统配置：内核参数”。

---

# 3) 安装（字符界面安装器 `vastbase_installer`）

**解包并启动安装器**

```bash
# 以 vastbase 用户
tar -xzvf Vastbase-installer-2.2_Build15-openEuler_22.03sp3-x86_64.tar.gz
cd vastbase-installer
./vastbase_installer
```

安装前确认端口未被占用。

**依赖检查**：缺包会中止，按提示用包管理器安装后重试（示例提示里缺 tcl，用 `yum -y install tcl`）。

**环境检查**：如提示 IPC 配置不合规，按屏幕建议在两个位置都设置 `RemoveIPC=no`，并执行 `daemon-reload` 与重启 `systemd-logind`。

**是否初始化实例**：

- 选 `Y` = 安装完自动初始化一个数据库实例（初学者/常规部署推荐）。

- 选 `N` = 仅装软件，之后用 `vb_initdb` 手动初始化。

**安装类型**：

- `Typical`（典型安装）：用默认参数初始化（推荐入门/快速验证）。

- `Custom`（自定义）：手工配置目录、端口、兼容模式等（生产推荐根据规范选择）。

**设置初始化用户密码**：按提示两次输入（初始化用户为 `vastbase`）。

**选择目录**（强烈建议与“准备工作”的规划保持一致）：

- 软件安装目录（不存数据，如 `/vastdata/app/vastbase/2.2.15`）

- 数据初始化目录（存放数据，如 `/vastdata/data/vastbase`）  
  安装器界面会逐一询问上述目录。

**选择兼容模式**（**一经选择不可更改，实例内所有 database 统一兼容**）：

- `A`=Oracle、`B`=MySQL、`PG`=PostgreSQL、`MSSQL`=SQL Server  
  示例输入 `B` 表示 MySQL 兼容模式。**生产务必确认后再选。**

> 提示：Vastbase 也支持通过 GSDP 图形化/静默安装、Docker 镜像、RPM 包等多种方式；集群安装也支持一键化方案。

---

# 4) License（试用到期后无法启动的处理）

新版安装包自带**免费试用期**。到期未导入有效 license 会报错并导致库无法启动；license 默认查找路径示例：`/home/vastbase/.license/vastbase_license`。上线前务必导入正式 license。

---

# 5) 启动与登录

**登录**（数据库已启动时）：

```bash
# 本机
vsql -r -d vastbase

# 或指定服务信息
vsql -r -h <server_ip> -p <port> -U vastbase -d <db>
```

登录成功会看到 `vastbase=#` 提示符。

> 数据目录（$PGDATA）内关键文件：`postgresql.conf`、`pg_hba.conf`、`pg_xlog/`（WAL）、`global/`、`base/` 等，日常改参数、放归档/备份时会用到。

---

# 6) 远程访问与基础安全

**开放远程访问**：在 `$PGDATA/pg_hba.conf` 添加允许的网段与账号，并重载配置（或重启）。示例（为备份用户开放登录）：

```conf
# 允许 172.16.55.0/24 网段的 vbuser 使用密码登录
host  all  vbuser  172.16.55.0/24  sha256
```

> 提醒：按最小权限原则为具体网段/业务账号放行；**不要**直接放行 `0.0.0.0/0` 到所有用户。

---

# 7) 归档与物理备份（必会）

**准备归档/备份目录**

```bash
mkdir -p /vastdata/archivelog
mkdir -p /vastdata/backup
```

**创建备份专用账号**（示例口令）：

```sql
CREATE USER vbuser WITH SYSADMIN IDENTIFIED BY 'Vastdata#1234';
```

**启用归档**（以初始化用户登录执行）：

```sql
ALTER SYSTEM SET archive_dest = '/vastdata/archivelog';
ALTER SYSTEM SET archive_mode = on;
```

**执行基线物理备份**：

```bash
vb_basebackup -D /vastdata/backup
ls -l /vastdata/backup
```

（未指定 `-F` 时为“平面文件”完整拷贝）。

---

# 8) 客户端与工具连接

## 8.1 vsql（内置命令行）

- 见“启动与登录”一节。常用：`\l` 列库、`\dt` 列表、`\i file.sql` 执行脚本等（与 PG 类似）。

## 8.2 VDS（Vastbase DataStudio）

- **安装**：下载解压即用，需要 **JDK 11**。

- **运行**：解压目录直接启动 VDS，界面“新建数据库连接”，填写主机/IP、端口、数据库、用户名/密码即可（与 vsql 参数一致）。

## 8.3 DBeaver（桌面客户端）

- **驱动选择与版本建议**：
  
  - **P 版** Vastbase：使用 DBeaver **7.3** + PostgreSQL 驱动 **42.2.9**。
  
  - **V 版** Vastbase：使用 DBeaver **8.3** + PostgreSQL 驱动 **42.5.1**。  
    在 DBeaver 里新建 PG 连接，按主机/端口/库名/用户/密码填写即可。

## 8.4 JDBC/ODBC/Libpq/Psycopg

- Vastbase **支持 JDBC、ODBC、Libpq、Psycopg** 等方式与应用集成；实际配置与 PostgreSQL 家族基本一致（使用对应驱动、按主机/端口/数据库/用户/密码配置）。

> 说明：文档强调“支持 JDBC/ODBC 等驱动”，实际项目中通常采用 **PostgreSQL 驱动** 与 Vastbase 对接；连接参数与 DBeaver 中 PG 驱动一致。

---

# 9) 常用排查与小工具

- **vb_config**：查看安装路径、编译配置等（示例）：
  
  ```bash
  vb_config --bindir
  vb_config --docdir
  vb_config --configure
  ```

- **vb_controldata**：读取控制文件要点、实例状态、checkpoint 等（需指明数据目录）：
  
  ```bash
  vb_controldata -D /vastdata/data/vastbase
  ```

- **License 到期**：启动报 `License expired, database start failed`，请导入有效 license（默认路径见上文）。

- **端口被占用**：安装前检查端口占用，必要时换端口或停止占用进程。

- **RemoveIPC 导致退出崩溃**：确认两个位置都设置 `RemoveIPC=no` 并重启 `systemd-logind`。安装器会给出明确的三步提示（修改→daemon-reload→restart）。

---

# 10) 数据目录速查（$PGDATA）

- 关键路径与文件（节选）：`base/`（各 DB 数据文件）、`global/`（全局系统表与控制文件）、`pg_xlog/`（WAL）、`postgresql.conf`、`pg_hba.conf` 等，见原文结构图与说明。

---

## 落地建议（可直接照此执行）

1. 按“**1/2章**”完成**用户与目录**、**SELinux/防火墙/RemoveIPC**、**内核参数**与**可选系统项**。

2. 以 `vastbase` 用户运行安装器，**选 Y 初始化** → **典型安装（或自定义目录）** → **慎选兼容模式**。

3. 首次登录用 `vsql` 验证；随后配置 `pg_hba.conf` 放行所需网段；**启用归档并做一次基线 `vb_basebackup`**。

4. 客户端侧：**优先 DBeaver（按 P/V 版选对驱动版本）** 或 **VDS（JDK11、解压即用）**；应用侧用 **JDBC/ODBC/Libpq** 与 PG 兼容驱动对接。

5. 上线前校验 **License** 状态，避免试用到期导致实例起不来。

> 以上步骤与命令全部来自你提供的官方 PDF，并尽量压缩成「可落地的操作手册」。如果你给我目标 OS/版本与兼容模式偏好，我可以把以上命令替换成**你环境的“拷贝即用版”**（含 `sysctl.conf` 计算好的 `shmmax/shmall`）。
