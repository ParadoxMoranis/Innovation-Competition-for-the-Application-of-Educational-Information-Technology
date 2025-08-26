# 海量数据VastBase部署

此前，我们已经准备好了CentOS 7.9,并且更换了yum源，也上传了需要的文件。这份文档是VastBase在CentOS 7.9上的部署教程。

## 连接至服务器

上次的文档我们写到

![](C:\Users\Moranis\AppData\Roaming\marktext\images\2025-07-21-18-20-07-image.png)

我么们可以通过ssh连接到虚拟机。这里我推荐一个工具叫做`HexHub`，这个软件的功能包括但不限于管理ssh链接，非常好用。

![](C:\Users\Moranis\AppData\Roaming\marktext\images\2025-07-21-18-54-00-image.png)

点击创建资产，如果你的虚拟机网络端口转发和我文档的设置相同，你可以这样填写：

![](C:\Users\Moranis\AppData\Roaming\marktext\images\2025-07-21-18-55-14-image.png)

点击测试链接，没问题保存即可。

![](C:\Users\Moranis\AppData\Roaming\marktext\images\2025-07-21-18-56-29-image.png)

上述操作均可以在Windows终端完成。后续的操作当然也可以。

### 创建用户组并创建数据库普通用户与设置密码

执行命令：

```bash
groupadd vastbase
useradd -g vastbase vastbase
useradd vastbase
passwd vastbase
```

跟随提示设置密码。

### 准备安装数据库

规划软件安装目录：我采用`/vastdata/app/vastbase`

规划数据库目录：我采用`/vastdata/data/vastbase`

创建目录，这里要用到的命令是我曾经在一篇文档中写过的。

```bash
mkdir -p /vastdata/app/vastbase
mkdir -p /vastdata/data/vastbase
```

授予数据库用户权限，这里我也有介绍过**chown**命令

```bash
chown -R vastbase:vastbase /vastdata
```

### 调整系统配置

#### 1.关闭防火墙

```bash
systemctl stop firewalld.service
systemctl disable firewalld.service
```

#### 2.禁用SELINUX

使用Vim或者Vi修改`/etc/selinux/config`,修改**SELINUX=enforcing**修改为**SELINUX=disabled**

然后重启系统。

#### 关闭RemoveIPC

1. **编辑 logind.conf 文件**：
   使用vi或者vim编辑 `/etc/systemd/logind.conf`：
   
   ```bash
   sudo vim /etc/systemd/logind.conf
   ```

2. **修改 RemoveIPC 设置**：
   找到 `RemoveIPC` 配置项（如果不存在，可以添加）。将其设置为 `no`：
   
   ![](C:\Users\Moranis\AppData\Roaming\marktext\images\2025-07-21-19-33-07-image.png)
   
   去掉`#`解除注释。
   
   ```bash
   RemoveIPC=no
   ```

3. **重启 systemd-logind 服务**：
   保存文件后，重启 `systemd-logind` 服务以应用更改：
   
   ```bash
   systemctl restart systemd-logind
   ```

4. **验证设置**：
   检查配置是否生效：
   
   ```bash
   cat /etc/systemd/logind.conf | grep RemoveIPC
   ```
   
   应返回 `RemoveIPC=no`。
   
   #### 调整内核参数
   
   ### 1. 文件句柄相关 (fs.aiomax-nr, fs.file-max, fs.nr_open)
   
   - **fs.aiomax-nr**: 建议值 1048576
     
     ```bash
     echo "fs.aio-max-nr = 1048576" >> /etc/sysctl.conf
     sudo sysctl -p
     ```
   
   - **fs.file-max**: 建议值 无限大 (无具体数值，但应最大化)
     
     ```bash
     echo "fs.file-max = 2097152" >> /etc/sysctl.conf
     sudo sysctl -p
     ```
   
   - **fs.nr_open**: 建议值 无限大 (无具体数值，但应最大化)
     
     ```bash
     echo "fs.nr_open = 2097152" >> /etc/sysctl.conf
     sudo sysctl -p
     ```
   
   ### 2. 进程间通信 (IPC) 相关 (kernel.sem)
   
   ```bash
   echo "kernel.sem = 250 32000 100 128" >> /etc/sysctl.conf
   sudo sysctl -p
   ```
   
   ### 3. 共享内存相关 (kernel.shmall, kernel.shmmax, kernel.shmmni)
   
   - **kernel.shmall**: 推荐值 80%/PAGE_SIZE
     
     ```bash
     echo "kernel.shmall = 800000" >> /etc/sysctl.conf
     sudo sysctl -p
     ```
   
   - **kernel.shmmax**:  
     
     ```bash
     echo "kernel.shmmax = 8,589,934,592" >> /etc/sysctlctl.conf
     sudo sysctl -p
     ```
   
   - **kernel.shmmni**: 推荐值 4096
     
     ```bash
     echo "kernel.shmmni = 4096" >> /etc/sysctl.conf
     sudo sysctl -p
     ```
   
   ### 4. 虚拟内存相关 (vm.swappiness)
   
   ```bash
   echo "vm.swappiness = 10" >> /etc/sysctl.conf
   sudo sysctl -p
   ```

你可以把每个echo执行后再sysctl -p

![](C:\Users\Moranis\AppData\Roaming\marktext\images\2025-07-21-19-45-22-image.png)

> 官方课程的可选配置我们不做设置。

## 安装

将文件移动到vastbase用户的目录下（如果你scp的用户就是vastbase则不用）。

```bash
mv ./VB/Vastbase-installer-2.2_Build15-centos_7-x86_64.tar.gz /home/vastbase
```

然后切换用户

```bash
su - vastbase
```

解压压缩包

```bash
tar -zxvf Vastbase-installer-2.2_Build15-centos_7-x86_64.tar.gz
```

等待解压完成。

 进入vastbase-installer目录并执行安装脚本

```bash
cd vastbase-installer
./vastbase_installer
```

根据提示进行配置

如果缺少以来就手动安装

![](C:\Users\Moranis\AppData\Roaming\marktext\images\2025-07-21-20-08-28-image.png)

安装依赖后继续安装数据库。

此时我遇到了一个问题，RemoveIPC需要在另一个文件中添加。路径是 **/usr/lib/systemd/system/systemd-login.service** 在这个文件中添加一行`RemoveIPC=0` 。

![](C:\Users\Moranis\AppData\Roaming\marktext\images\2025-07-21-20-28-11-image.png)

在设置路径时，输入我们曾经规划的第一个路径。

![](C:\Users\Moranis\AppData\Roaming\marktext\images\2025-07-21-20-37-30-image.png)

兼容模式选择B，MySQL的兼容模式。（比赛时按照要求做）。

![](C:\Users\Moranis\AppData\Roaming\marktext\images\2025-07-21-20-39-29-image.png)

安装完成，后面我们会重点介绍ExBase数据库迁移工具。
