# Vastbase 数据库使用文档--Grok

## 1. 硬件和系统要求

### 依赖软件包要求

安装程序会检查依赖包是否已安装，并列出具体版本要求，例如：

- readline : 8.1
- libicu : 72.1
- cracklib : 2.9.8
- libxslt : 1.1.37
- tcl : 8.6.12
- perl : 5.34.0
- openldap : 2.6.0
- pam : 1.5.2
- systemd-libs : 249
- bzip2 : 1.0.8
- gettext : 0.21.1
- libaio : 0.3.113
- ncurses-libs : 6.3
- python2 : line

如果有缺失包，安装程序将中止。配置 yum 源后，使用 `yum -y install <package>` 安装缺失包，例如 `yum -y install tcl`。安装后重新运行安装程序。

### Linux 内核参数调整（环境配置）

安装前检查和调整系统参数：

- **IPC 参数**：如果检查失败，根据提示修改。系统提示 "The systemd-logind parameter setting is incorrect" 时，进行以下步骤：
  
  - 在 `/etc/systemd/logind.conf` 中增加 `RemoveIPC=no`。
  - 在 `/usr/lib/systemd/system/systemd-logind.service` 中增加 `RemoveIPC=no`（如果提示 show-session 参数问题）。
  - 执行 `systemctl daemon-reload`。
  - 执行 `systemctl restart systemd-logind`。

- **前提条件**：
  
  - 已完成数据库用户组和数据库用户的创建（例如，使用 root 用户执行 `groupadd vastbase` 和 `useradd -g vastbase vastbase`）。
  - vastbase 用户对数据库包解压路径、安装路径有读、写、执行权限，并且安装路径必须为空。
  - 安装前检查指定的 Vastbase 端口（默认 5432）是否被占用。如果被占用，请更改端口或停止当前使用端口的进程（使用 `netstat -tuln | grep 5432` 检查）。

- **VDS 工具系统要求**（如果使用 Vastbase Data Studio 等图形工具）：
  
  | 硬件项目 | 要求                                            |
  | ---- | --------------------------------------------- |
  | 内存   | 2G+ Memory                                    |
  | CPU  | Intel X86、Kunpeng920、Hygon X86、ft2500、ft2000  |
  | 存储   | 1GB 用于安装 VDS 的应用程序包。<br>100MB 以上空间用于 Home 目录。 |
  | 网络   | 千兆网络。                                         |
  
  支持操作系统：
  
  - Intel X86: Windows 7/10 (64 位), Windows Server 2012 (64 位), Windows Server 2008 R2 Enterprise。
  - Kunpeng920: openEuler 20.03 (LTS-SP2) Server, UOS V20 1050e Server, Kylin V10 SP1, Kylin-GFB V10。
  - Hygon X86: Kylin V10 SP3 Server。
  - ft2500: Kylin V10 GFB Desktop。
  - ft2000: Kylin V10 GFB Desktop, Kylin V10 涉密专用版 Server。

## 2. 数据库部署（安装）

Vastbase 使用响应式安装程序进行部署。支持单节点字符安装。

### 操作步骤

1. **解压安装包**（使用 vastbase 用户）：
   
   ```
   tar –xzvf Vastbase-installer-2.2_Build15-openEuler_22.03sp3-x86_64.tar.gz
   ```
   
   这会生成 `vastbase-installer` 目录。

2. **执行安装程序**：
   
   ```
   cd vastbase-installer
   ./vastbase_installer
   ```

### 安装时的选项含义及推荐选项

安装程序提供交互式提示。典型安装使用默认，自定义允许手动配置。推荐初学者使用典型安装（选项 1），生产环境根据需求自定义。

- **是否实例化数据库 (Y/N)**：Y - 安装后初始化实例（推荐）；N - 只安装软件，后用 `vb_initdb` 手动初始化。
- **选择安装类型**：1 - Typical installation（使用默认参数初始化，推荐）；2 - Custom installation（手动配置参数）。
- **初始用户密码**：为 vastbase 用户设置密码（必须复杂，如 Vastdata#1234）。
- **软件安装目录**：数据库软件路径，默认 `/home/vastbase/local/vastbase`。推荐自定义为 `/vastdata/app/vastbase/2.2.15`（不存储数据，不需大空间）。
- **数据库初始化目录**：数据存储路径，默认 `/home/vastbase/data/vastbase`。推荐自定义为 `/vastdata/data/vastbase`（根据业务规划大空间）。
- **兼容模式**：A (Oracle，默认，推荐迁移 Oracle)；B (MySQL)；PG (PostgreSQL)；MSSQL (SQL Server)。一旦选择不可更改，所有数据库统一模式。
- **License 配置**（安装后）：编辑 `postgresql.conf` 添加 `license_path = '/home/vastbase/.license/vastbase_license'`。最新版本有免费试用期，无 License 可启动但到期后失败。

安装完成后：

- 执行 `source ~/.bashrc` 初始化环境。
- 启动数据库：`vb_ctl start`。
- 如果失败，检查 License 或参数。

## 3. 数据库参数调整

### 本地缓冲区调整

在 `postgresql.conf` 中配置，重启生效：

- `work_mem`：当前 work mem 大小，单位 KB，默认 65536KB。用于排序、哈希等。推荐：根据内存调整，生产 128MB，避免 OOM。
- `temp_buffers`：临时缓冲区大小，单位 8KB，默认 1MB。用于临时表。推荐：根据负载 8MB。
- `maintenance_work_mem`：维护操作缓冲区大小，单位 KB，默认 16MB。用于 VACUUM 等。推荐：64MB。

查看/设置（vsql 中）：

```
SHOW work_mem;
SET work_mem = '128MB';  -- 会话级
```

### 创建数据库参数

```
CREATE DATABASE mydb OWNER myuser TEMPLATE template0 ENCODING 'UTF8' LC_COLLATE 'en_US.utf8' LC_CTYPE 'en_US.utf8' TABLESPACE mytablespace CONNECTION LIMIT 100;
```

- `ENCODING`：默认 UTF-8。
- `LC_COLLATE/LC_CTYPE`：使用 `locale -a` 查看支持值，推荐 'en_US.utf8'。
- `CONNECTION LIMIT`：最大连接，推荐根据负载 100-500。

## 4. 使用工具连接和操作数据库

### vsql（命令行客户端）

- 本地连接：`vsql -r -d vastbase`（默认用户 vastbase、端口 5432）。
- 远程连接：`vsql -r -d vastbase -h 192.168.1.105 -p 5432 -U vbuser -W Vastdata#1234`。
- 创建用户：`CREATE USER vbuser IDENTIFIED BY 'Vastdata#0730' LOGIN;`。
- 执行 SQL：创建表、插入、查询示例如前所述。
- 元命令：`\l` (列数据库)、`\d tablename` (描述表)、`\q` (退出)。

### vb_ctl（服务控制）

- 启动：`vb_ctl start -D /vastdata/data/vastbase -s -l /vastdata/history/start.log`。
- 停止：`vb_ctl stop -D /vastdata/data/vastbase -w -t 30`（等待事务，超时 30s）。
- 状态：`vb_ctl status`。
- 重启：`vb_ctl restart`。

### vb_guc（参数管理）

- 重载：`vb_guc reload -D $PGDATA -c "behavior_compat_options = 'bind_procedure_searchpath, display_leading_zero'"`。

## 5. JDBC 配置和驱动

使用 PostgreSQL 兼容 JDBC 驱动（e.g., postgresql-42.2.18.jar）。

- 连接 URL：`jdbc:postgresql://192.168.1.105:5432/vastbase?user=vbuser&password=Vastdata#1234`。
- Vastbase 支持 JDBC 用于 Java 连接，处理增删改查。

## 6. 备份和恢复操作

### 物理备份：vb_basebackup

格式：`vb_basebackup [OPTION]...`
参数：

- `-D directory`：输出目录（必选）。
- `-c fast|spread`：检查点模式（默认 spread）。
- `-F plain|tar`：格式（默认 plain，推荐 tar 压缩）。
- `-X fetch|stream`：XLOG 方法（默认 stream）。
- `-Z level`：压缩级别 (0-9，推荐 6)。

示例：`vb_basebackup -D /backup/dir -F tar -X stream -v`。

### 逻辑备份：vb_dump

格式：`vb_dump [OPTION]... [DBNAME]`
参数：

- `-f FILENAME`：输出文件（c/d/t 格式必选）。
- `-F c|d|t|p`：格式（推荐 c 自定义）。
- `-Z 0-9`：压缩（推荐 6）。
- `-a`：只数据。
- `-s`：只模式。
- `-t TABLE`：指定表。

示例：`vb_dump -f backup.sql -F p mydb`。

### 逻辑备份：vb_dumpall

格式：`vb_dumpall [OPTION]...`
参数：

- `-f FILENAME`：输出文件。
- `-g`：只全局对象。
- `-a`：只数据。
- `-s`：只模式。
- `-c`：先清理。

示例：`vb_dumpall -f all.sql -c`。

### 恢复：vsql

外部：`vsql -p 5432 -f backup.sql`。
内部：`\i backup.sql`。

### 恢复：vb_restore

格式：`vb_restore [OPTION]... FILE`
参数：

- `-f FILENAME`：输出脚本。
- `-l`：列出内容。
- `-a`：只数据。
- `-s`：只模式。
- `-c`：先清理。

示例：`vb_restore -d mydb -c backup.dump`。

## 7. 插件安装示例（PostGIS）

1. 下载 Vastbase 专用 PostGIS 插件包。
2. root 用户解压，将 postgis 文件夹复制到 $GAUSSHOME。
3. yum 安装依赖：`yum install -y gmp gmp-devel mpfr mpfr-devel boost boost-devel`（指定版本）。
4. 设置参数：`vb_guc reload -D $PGDATA -c "behavior_compat_options = 'bind_procedure_searchpath, display_leading_zero'"`。
5. 重启：`vb_ctl restart`。
6. 创建扩展：`CREATE EXTENSION postgis; CREATE EXTENSION postgis_sfcgal; CREATE EXTENSION postgis_raster; CREATE EXTENSION postgis_topology;`。
7. 测试：`SELECT Box3D(ST_GeomFromEWKT('LINESTRING(1 2 3, 3 4 5, 5 6 5)'));`。
8. 删除：`DROP EXTENSION postgis CASCADE;`。

生产环境监控日志，调整参数优化性能。
