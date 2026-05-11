# Linux操作系统实战面试题（第13部分）

> 本系列面试题结合WMS仓库管理系统项目实战，覆盖20道高频考点。

---

## 第1题：Linux目录结构与FHS规范

### 题目
描述Linux根目录下 `/etc`、`/home`、`/var`、`/usr`、`/tmp` 各存放什么内容？什么是FHS规范？在WMS项目中如何体现？

### 核心答案

| 目录 | 用途 | WMS项目示例 |
|------|------|------------|
| `/etc` | 系统配置文件 | `/etc/my.cnf`（MySQL配置）、`/etc/nginx/nginx.conf` |
| `/home` | 普通用户家目录 | WMS应用运行用户如 `tomcat` 的home目录 |
| `/var` | 变长数据（日志、缓存、数据库） | `/var/log/wms/`（wms-backend日志）、`/var/lib/mysql` |
| `/usr` | 只读共享资源（程序、库、文档） | `/usr/local/java/`（JDK）、`/usr/local/tomcat/` |
| `/tmp` | 临时文件 | 文件上传临时目录、报表生成临时文件 |

**FHS（Filesystem Hierarchy Standard）** 是Linux目录结构的规范标准，定义了各目录的用途和权限：
- `/bin` → 基础命令（所有用户可用）
- `/sbin` → 系统管理命令（root专用）
- `/lib` → 系统库
- `/opt` → 第三方大型软件（如Oracle）
- `/srv` → 服务数据（如Web服务）

### 追问方向
1. 为什么数据库文件通常放在 `/var` 而不是 `/usr`？
2. `/run` 和 `/var/run` 的关系是什么？（`/run` 是现代系统的tmpfs）
3. 如何查看某个目录的磁盘占用？（`du -sh /var/log`）

### 避坑提示
- ❌ 不能说"想放哪就放哪"——FHS是面试必考的基础规范
- ❌ 不能混淆 `/tmp` 和 `/var/tmp`：重启后 `/tmp` 清空，`/var/tmp` 保留
- ✅ WMS项目日志配置在 `logback-spring.xml` 中指定的路径要符合FHS

---

## 第2题：Linux文件权限与特殊权限位

### 题目
解释Linux文件权限的三组权限位（rwx）、`chmod/chown/umask` 的用法，以及SUID、SGID、StickyBit三种特殊权限位的区别。

### 核心答案

**基础权限（数字法）：**
```bash
chmod 755 file      # rwxr-xr-x（所有者rwx，组rx，其他rx）
chmod 644 file      # rw-r--r--
chown user:group file   # 改所有者和组
chmod +x script.sh      # 添加执行权限
```

**umask（默认权限掩码）：**
```bash
umask 0022           # 新建文件默认644，目录默认755
umask 0002           # 允许组内用户写（新建文件664，目录775）
```
新文件实际权限 = `0666 - umask`（执行权限不默认授予）

**三种特殊权限：**

| 权限位 | 数字 | 作用 | 典型场景 |
|--------|------|------|---------|
| **SUID** | 4xxx | 执行时以文件所有者身份运行 | `passwd`（修改自己的密码需要root权限写/etc/shadow） |
| **SGID** | 2xxx | 执行时以文件所属组身份运行；目录内新建文件继承目录组 | 共享开发目录 |
| **StickyBit** | 1xxx | 只允许所有者删除自己的文件 | `/tmp`（所有人可写但只能删自己的） |

```bash
chmod 4755 /usr/bin/somebinary   # SUID
chmod 2755 /sharedDir            # SGID
chmod 1777 /tmp                   # StickyBit
ls -l /tmp                        # 显示 drwxrwxrwt
```

### 追问方向
1. SUID为什么不能用在shell脚本上？（shell脚本的SUID会被忽略，存在安全风险）
2. 如何查找具有SUID权限的文件？→ `find / -perm -4000 -type f`
3. umask在 `/etc/profile` 和用户 `~/.bashrc` 中的区别？

### 避坑提示
- ❌ 不要混淆SUID和有效用户ID（euid）——SUID在执行瞬间提升权限
- ❌ Java程序运行时通常以tomcat/root用户运行，不要轻易给Java jar设置SUID
- ✅ WMS项目中，`/var/log/wms/` 目录权限应允许应用用户写入日志

---

## 第3题：Linux进程管理

### 题目
描述Linux中查看和管理进程的主要命令：`ps`、`top`、`htop`、`pstree`、`kill`、`pkill` 的用法，以及如何使用正则表达式匹配进程名。

### 核心答案

**进程查看：**
```bash
ps aux                    # 查看所有进程（BSD风格）
ps -ef                    # 查看所有进程（标准风格，显示PPID）
ps -ef | grep java        # 查找Java进程
ps -ef | grep -E 'java|nginx'   # 正则匹配多个进程名
```

**进程监控：**
```bash
top                      # 动态监控（默认3秒刷新）
htop                     # 更友好的交互式进程查看（需安装）
pstree -p                # 树状显示进程及其PID
pstree -ap | grep java   # 查看Java进程树
```

**进程终止：**
```bash
kill PID                 # 发送SIGTERM（优雅停止）
kill -9 PID              # 发送SIGKILL（强制杀死）
kill -15 PID             # 等同于kill，SIGTERM

pkill java               # 杀死所有java进程
pkill -f "inventory-service"   # 匹配完整命令行
pkill -9 -u tomcat       # 强制杀死tomcat用户的所有进程
```

**正则匹配进程名的完整示例：**
```bash
ps -ef | grep -E '^.*java.*inventory.*$'
# 或者用pgrep：
pgrep -f "inventory-service"    # 返回PID列表
pgrep -a -f "inventory-service" # 显示完整命令行
```

### 追问方向
1. `kill -9` 和 `kill -15` 的区别？（SIGKILL不可捕获，SIGTERM可被Java的shutdown hook捕获）
2. 如何查看进程的工作目录？→ `ls -l /proc/PID/cwd`
3. `ps aux` 和 `ps -ef` 的输出有什么区别？

### 避坑提示
- ❌ 不要用 `kill -9` 杀Java进程——会导致OOM Killer触发或数据库连接未关闭
- ❌ `pkill` 是全局匹配，可能误杀其他不相关的java进程
- ✅ WMS项目中，停止服务应使用 `systemctl stop wms-app`，而非直接kill

---

## 第4题：Linux内存管理

### 题目
解释Linux内存相关命令 `free`、`top`、`vmstat` 的输出含义，以及buffer与cache的区别。什么是OOM Killer？

### 核心答案

**free命令：**
```bash
free -h                  # 人类可读单位显示
free -m                  # 单位MB

# 输出示例：
#               total        used        free      shared  buff/cache   available
# Mem:          32768       24576        2048         128        6144       7168
# Swap:          8192           0        8192
```

**关键指标解释：**
- **total**：总物理内存
- **used**：已使用（不含cache）
- **free**：完全空闲
- **buff/cache**：缓冲区 + 页缓存（可回收）
- **available**：真正可分配内存（free + 可回收的cache）

**vmstat监控：**
```bash
vmstat 1 5               # 每秒1次，共5次
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
#  2  0      0 2048000  1024 6144000    0    0     0     0    0    0  5  2 93  0  0
```

**buffer vs cache（高频混淆点）：**
- **buffer（缓冲区）**：写操作临时存储，**即将写入磁盘**的数据（块设备）
- **cache（缓存）**：读操作临时存储，**从磁盘读取**的数据（页缓存）
- 两者都是Linux页缓存机制的一部分，统称为 `buff/cache`，可被回收用于分配给进程

**OOM Killer（Out of Memory Killer）：**
当物理内存耗尽时，Linux内核触发OOM Killer，选择一个进程杀掉并释放其占用的内存：
```bash
dmesg | grep -i "out of memory"   # 查看OOM日志
/var/log/messages                # 系统日志中的OOM记录
```
被杀死进程的退出码为137（128+9，即SIGKILL）

### 追问方向
1. 为什么不直接看 `free` 列，而要看 `available`？
2. Java进程heap外还占用哪些内存？（JVM metaspace、direct buffer、native memory）
3. 如何调整OOM Killer的策略？→ `/proc/sys/vm/overcommit_memory`

### 避坑提示
- ❌ 不能说"buffer和cache是释放不掉的"——它们在需要时会被回收
- ❌ Java进程的 `-Xmx` 只是堆内存，JVM实际占用还包括native memory
- ✅ WMS项目中如果内存不足，inventory-service可能因OOM被杀掉

---

## 第5题：Linux磁盘管理

### 题目
描述Linux磁盘管理命令 `df`、`du`、`fdisk`、`mkfs`、`fsck` 的用法，以及如何评估磁盘IO性能。

### 核心答案

**磁盘使用查看：**
```bash
df -h                    # 查看所有挂载点使用情况
df -h /var/log           # 查看指定目录所在磁盘
du -sh /var/log/*        # 查看/var/log下各子目录大小
du -h --max-depth=1 /    # 限制深度1层
du -ah --max-depth=2 /usr/src/wms-backend   # WMS项目代码目录大小
```

**磁盘分区与格式化：**
```bash
fdisk -l                 # 查看磁盘分区表
fdisk /dev/sdb           # 交互式分区
mkfs.ext4 /dev/sdb1      # 格式化为ext4
mkfs.xfs /dev/sdb1       # 格式化为xfs（大型文件性能更好）
```

**文件系统检查与修复：**
```bash
fsck /dev/sdb1           # 检查并修复（需先卸载）
fsck -n /dev/sdb1        # 只读检查，不修复
mount /dev/sdb1 /data    # 挂载
```

**IO性能评估：**
```bash
iostat -x 1 5            # 扩展统计，每秒1次共5次
# %util 接近100%说明IO饱和

iotop                    # 交互式查看各进程IO占用（需root）
sar -b 1 3               # IO和传输率统计

# 简单测速：
dd if=/dev/zero of=/tmp/test bs=1M count=1024 oflag=direct
```

### 追问方向
1. ext4和xfs的区别？（xfs在大文件和高并发下性能更好，但救援工具不如ext4完善）
2. 如何查看inode使用情况？→ `df -i`
3. LVM逻辑卷的优势？（动态扩容、快照）

### 避坑提示
- ❌ 不要在生产环境对挂载中的磁盘执行 `fsck`
- ❌ `dd` 命令oflag要用direct，否则被cache缓存导致测试不准
- ✅ WMS项目中 `/var/log/wms/` 日志过多可能导致磁盘满，要定期清理

---

## 第6题：Linux网络管理

### 题目
描述Linux网络配置和诊断命令 `ip`、`ifconfig`、`netstat`、`ss`、`lsof` 的用法，以及如何排查端口占用问题。

### 核心答案

**网络配置（旧 vs 新）：**
```bash
ifconfig eth0                 # 查看网卡（旧命令）
ip addr show eth0              # 查看网卡（新命令）
ip link set eth0 up/down       # 启用/禁用网卡
ip addr add 192.168.1.10/24 dev eth0   # 添加IP
```

**端口和连接查看：**
```bash
netstat -tlnp                 # 查看监听TCP端口（显示进程名和PID）
netstat -anp | grep 8080      # 查看8080端口占用
ss -tlnp                      # 比netstat更快的替代品
ss -tlnp | grep 8080

lsof -i:8080                  # 查看8080端口被谁占用
lsof -iTCP:8080               # TCP协议
lsof -i -P -n                 # -P禁止端口转换，-n禁止DNS解析
```

**完整排查流程（以WMS的8080端口为例）：**
```bash
1. 查看端口监听：ss -tlnp | grep 8080
2. 查看进程详情：lsof -i:8080
3. 查看进程PID：ps -ef | grep java
4. 查看进程启动时间：ps -eo pid,lstart,cmd | grep PID
5. 查看网络连接数：ss -s
```

### 追问方向
1. `netstat -a` 和 `netstat -t` 的区别？
2. TIME_WAIT状态过多如何处理？→ `net.ipv4.tcp_tw_reuse`
3. 如何查看UDP端口？→ `ss -u -a`

### 避坑提示
- ❌ 不要混淆 `ss` 和 `netstat`——`ss` 是新命令，性能更好
- ❌ Java程序无法绑定端口时，不一定是端口占用，可能是权限或IP地址问题
- ✅ WMS项目中多个微服务同时启动可能导致端口冲突

---

## 第7题：Linux日志管理

### 题目
描述Linux主要日志文件 `/var/log/messages`、`/var/log/secure`、`/var/log/cron` 的作用，以及日志轮转工具 `logrotate` 的配置。

### 核心答案

**主要日志文件：**

| 文件 | 内容 |
|------|------|
| `/var/log/messages` | 系统常规日志（内核、启动、服务等） |
| `/var/log/secure` | 安全相关（认证、SSH登录、sudo使用） |
| `/var/log/cron` | 定时任务执行日志 |
| `/var/log/nginx/access.log` | Nginx访问日志 |
| `/var/log/wms/wms-app.log` | WMS应用日志 |

**查看日志：**
```bash
tail -f /var/log/messages           # 实时跟踪
grep "ERROR" /var/log/wms/wms-app.log | tail -100
journalctl -u wms-app.service -f    # 查看systemd服务的日志
```

**logrotate配置：**
```bash
/etc/logrotate.conf                 # 全局配置
/etc/logrotate.d/wms               # 应用自定义配置

# 示例：/etc/logrotate.d/wms
/var/log/wms/*.log {
    daily                    # 每天轮转
    missingok               # 日志不存在不报错
    rotate 30               # 保留30天
    compress                # 压缩旧日志
    delaycompress           # 延迟压缩（保留最近一个不压缩）
    notifempty              # 空日志不轮转
    create 0644 tomcat tomcat  # 轮转后创建新文件权限
    postrotate
        systemctl reload wms-app > /dev/null 2>&1 || true
    endscript
}
```

手动触发：`logrotate -f /etc/logrotate.conf`

### 追问方向
1. 如何确认logrotate是否正常工作？→ `cat /var/lib/logrotate/status`
2. 为什么日志轮转后应用还在写旧文件？（没有发送SIGUSR1信号）
3. `/var/log/maillog` 是做什么的？（邮件系统日志）

### 避坑提示
- ❌ 不要删除日志文件而不重启服务——进程仍持有已删除文件的fd
- ❌ logrotate的 `sharedscripts` 关键字容易忽略，导致postrotate脚本执行多次
- ✅ WMS项目日志配置在 `logback-spring.xml`，建议配置滚动策略

---

## 第8题：Linux防火墙

### 题目
描述Linux三大防火墙工具 `iptables`、`firewalld`、`ufw` 的基本用法，以及防火墙规则链式匹配的优先级。

### 核心答案

**iptables（传统包过滤）：**
```bash
iptables -L -n -v                  # 查看所有规则
iptables -A INPUT -p tcp --dport 22 -j ACCEPT    # 允许SSH
iptables -A INPUT -p tcp --dport 8080 -j ACCEPT  # 允许8080
iptables -A INPUT -j DROP          # 默认拒绝

# 保存规则
service iptables save
```

**firewalld（CentOS 7+默认）：**
```bash
firewall-cmd --list-all            # 查看所有规则
firewall-cmd --add-port=8080/tcp   # 临时开放8080端口
firewall-cmd --permanent --add-port=8080/tcp   # 永久开放
firewall-cmd --reload              # 重载配置
```

**ufw（Ubuntu默认）：**
```bash
ufw status                         # 查看状态
ufw allow 22/tcp                   # 允许SSH
ufw allow 8080/tcp                 # 允许8080
ufw delete allow 8080/tcp          # 删除规则
ufw enable                         # 启用防火墙
```

**链式匹配顺序（数据包经过的路径）：**
```
入站数据：PREROUTING → INPUT（本地进程）
出站数据：OUTPUT → POSTROUTING（离开系统）
转发数据：PREROUTING → FORWARD → POSTROUTING
```

### 追问方向
1. iptables的 filter 表的INPUT链和OUTPUT链的区别？
2. Docker容器端口映射后，宿主机防火墙需要额外配置吗？
3. 如何查看哪个规则阻止了连接？→ `iptables -L -n --line-numbers`

### 避坑提示
- ❌ 不要在生产环境随意执行 `iptables -F`——会清空所有规则导致服务器失联
- ❌ 远程操作防火墙前要开一个备用SSH连接
- ✅ WMS部署前要确认服务器防火墙已开放相应端口

---

## 第9题：Linux用户管理

### 题目
描述Linux用户管理命令 `useradd`、`usermod`、`groupadd` 的用法，以及 `/etc/passwd`、`/etc/shadow`、`sudoers` 文件的结构。

### 核心答案

**用户和组管理：**
```bash
useradd -m -s /bin/bash wmsuser          # 创建用户并创建家目录
useradd -r -s /sbin/nologin tomcat       # 创建系统用户（不可登录）
usermod -aG docker wmsuser                # 将用户加入docker组
usermod -L wmsuser                       # 锁定用户
groupadd wmsgroup                        # 创建组

id wmsuser                               # 查看用户信息
passwd wmsuser                           # 设置密码
```

**/etc/passwd结构：**
```
root:x:0:0:root:/root:/bin/bash
wmsuser:x:1000:1000::/home/wmsuser:/bin/bash
# 用户名:密码占位符:UID:GID:描述:家目录:登录shell
```

**sudoers配置（`/etc/sudoers` 或 `/etc/sudoers.d/`）：**
```bash
# 允许wmsuser以root身份运行所有命令
wmsuser ALL=(ALL) ALL

# 免密sudo
wmsuser ALL=(ALL) NOPASSWD:ALL

# 只允许执行特定命令
wmsuser ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart wms-app

# 配置后验证
visudo -c                              # 检查语法
```

### 追问方向
1. `/etc/shadow` 的结构是什么？（加密密码、最后一次修改时间、密码策略）
2. UID 0的root用户和UID 1000+的普通用户有什么区别？
3. `userdel` 和 `usermod -L` 的区别？

### 避坑提示
- ❌ 不要手动编辑 `/etc/passwd`——使用 `vipw` 命令（会加锁）
- ❌ 不要给普通用户sudo ALL权限——权限过大
- ✅ WMS项目部署应创建专用应用用户，而非用root运行Java进程

---

## 第10题：Linux定时任务

### 题目
描述Linux定时任务 `crontab` 的语法，以及 `/etc/cron.d/` 目录的作用。解释分、时、日、月、周五个时间字段。

### 核心答案

**crontab基本语法：**
```bash
crontab -e                             # 编辑当前用户的crontab
crontab -l                             # 查看当前用户的crontab
crontab -r                             # 删除当前用户的crontab
```

**时间字段格式：**
```
┌───────────── 分钟 (0-59)
│ ┌───────────── 小时 (0-23)
│ │ ┌───────────── 日 (1-31)
│ │ │ ┌───────────── 月 (1-12)
│ │ │ │ ┌───────────── 周 (0-7，0和7都是周日)
│ │ │ │ │
* * * * * command
```

**常用示例：**
```bash
# 每天凌晨2点清理日志
0 2 * * * /usr/bin/find /var/log/wms -name "*.log" -mtime +7 -delete

# 每周一、三、五下午5点执行备份
0 17 * * 1,3,5 /opt/backup/backup.sh

# 每月1日凌晨0点执行
0 0 1 * * /opt/backup/monthly.sh

# 每5分钟检查服务状态
*/5 * * * * /opt/monitor/check.sh

# 每年的1月1日0点
0 0 1 1 * /opt/annual.sh
```

**特殊字符：**
- `*`：任意值
- `,`：多个值（如 1,3,5）
- `-`：范围（如 1-5）
- `/`：步长（如 */10 表示每10个单位）

**系统级定时任务（/etc/cron.d/）：**
```bash
# /etc/cron.d/wms-backup
0 2 * * * root /opt/backup/backup.sh
# 格式：时间 用户 命令
```

### 追问方向
1. `crontab -e` 和在 `/etc/cron.d/` 创建文件的区别？
2. cron任务的时间是以服务器本地时区还是UTC计算？
3. 如何查看cron任务执行失败的原因？→ `grep CRON /var/log/cron`

### 避坑提示
- ❌ 不要忘记cron任务的环境变量问题——建议使用绝对路径
- ❌ cron任务的输出默认发邮件，要重定向到文件或 `/dev/null`
- ✅ WMS项目中，定时同步库存数据的任务可用cron实现

---

## 第11题：Linux服务管理

### 题目
描述Linux服务管理命令 `systemctl` 和 `service` 的区别，以及systemd的unit文件和target概念。

### 核心答案

**service（SysV init兼容）：**
```bash
service nginx start
service nginx stop
service nginx restart
service nginx status
```

**systemctl（systemd新方式）：**
```bash
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl status nginx
systemctl enable nginx           # 开机自启
systemctl disable nginx          # 取消开机自启
systemctl is-active nginx        # 检查是否运行
systemctl list-units --type=service  # 列出所有服务
```

**Unit文件示例（`/etc/systemd/system/wms-app.service`）：**
```ini
[Unit]
Description=WMS Application Service
After=network.target mysql.service
Wants=mysql.service

[Service]
Type=simple
User=tomcat
ExecStart=/usr/local/java/jdk/bin/java -jar /opt/wms/wms-app.jar
ExecStop=/bin/kill -SIGTERM $MAINPID
Restart=on-failure
RestartSec=10
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=wms-app

[Install]
WantedBy=multi-user.target
```

**Target（运行级别）：**
```bash
systemctl get-default               # 查看当前默认target
systemctl set-default multi-user.target   # 改为多用户无图形模式
systemctl isolate graphical.target  # 切换到图形界面
```

常见target：
- `multi-user.target`：多用户命令行模式（类似runlevel 3）
- `graphical.target`：图形界面模式（类似runlevel 5）
- `rescue.target`：单用户救援模式
- `emergency.target`：紧急救援模式

### 追问方向
1. `Type=simple` 和 `Type=forking` 的区别？
2. `After=` 和 `Wants=` 的区别？（强依赖 vs 弱依赖）
3. `systemctl daemon-reload` 什么时候需要执行？

### 避坑提示
- ❌ 修改unit文件后必须执行 `systemctl daemon-reload` 才能生效
- ❌ Java进程应使用 `Type=simple` 并在ExecStart中直接启动jar包
- ✅ WMS项目建议使用systemd管理，方便日志收集和自动重启

---

## 第12题：Linux管道与重定向

### 题目
描述Linux管道（`|`）和重定向（`>`、`>>`、`2>&1`）的用法，以及 `tee` 命令的作用。

### 核心答案

**基础重定向：**
```bash
command > file          # stdout重定向到file（覆盖）
command >> file         # stdout追加重定向
command 2> file         # stderr重定向到file（覆盖）
command 2>> file        # stderr追加重定向
command &> file         # stdout和stderr都重定向（覆盖）
command &>> file        # stdout和stderr都追加重定向
```

**组合重定向（2>&1）：**
```bash
command > file 2>&1     # 先重定向stdout到file，再把stderr重定向到stdout（file）
# 等价于：command &> file

# 常见错误写法（顺序很重要）：
command > file 2>&1     # ✓ 正确
command 2>&1 > file     # ✗ 错误——stderr仍输出到终端
```

**管道（|）：**
```bash
cat /var/log/wms/wms-app.log | grep ERROR | wc -l
ps -ef | grep java | grep -v grep
```

**tee（同时输出到屏幕和文件）：**
```bash
command | tee file.txt           # 屏幕和file都输出（覆盖）
command | tee -a file.txt        # 屏幕和file都输出（追加）
command | tee file.txt | other   # 屏幕→other，文件→file

# 配合sudo写入系统目录：
echo "127.0.0.1 wms.local" | sudo tee -a /etc/hosts
```

**实际应用（WMS日志分析）：**
```bash
# 实时查看并保存ERROR日志
tail -f /var/log/wms/wms-app.log | tee /tmp/wms-error.log

# 统计ERROR出现次数
grep ERROR /var/log/wms/wms-app.log | tee /tmp/errors.txt | wc -l
```

### 追问方向
1. `|` 和 `xargs` 的区别？什么时候必须用 `xargs`？
2. `command > /dev/null 2>&1` 是什么意思？
3. `tee` 和 `script` 命令的区别？

### 避坑提示
- ❌ 不要写 `command > file 2>&1 &`——后台重定向顺序仍有问题
- ❌ shell脚本中管道失败不会影响整体命令退出码（除非set -o pipefail）
- ✅ tee可以解决"既想看输出又需要保存"的问题

---

## 第13题：Linux文本处理

### 题目
描述Linux文本处理工具 `grep`、`sed`、`awk`、`cut`、`sort`、`uniq`、`wc`、`xargs` 的高级用法。

### 核心答案

**grep（文本搜索）：**
```bash
grep -E "ERROR|WARN" /var/log/wms/wms-app.log    # 正则匹配
grep -v "DEBUG" /var/log/wms/wms-app.log          # 反向匹配（排除）
grep -n "NullPointerException" app.log            # 显示行号
grep -C 3 "ERROR" app.log                         # 前后3行上下文
grep -r "wms-backend" /usr/src/                   # 递归搜索
grep -c "ERROR" app.log                           # 计数
```

**sed（流编辑器）：**
```bash
sed -n '10,20p' file                    # 打印第10-20行
sed 's/old/new/g' file                  # 全局替换
sed -i 's/ERROR/FATAL/g' file          # 直接修改文件
sed '/ERROR/d' file                     # 删除匹配行
sed -n '/ERROR/,/WARN/p' file         # 打印ERROR到WARN之间的行
```

**awk（报表生成器）：**
```bash
awk '{print $1, $3}' file              # 打印第1、3列
awk -F: '{print $1}' /etc/passwd       # 指定分隔符
awk '/ERROR/ {count++} END {print count}' app.log   # 统计ERROR出现次数
awk '{sum+=$NF} END {print sum}' file  # 统计最后一列总和
awk 'NR==5' file                       # 打印第5行
awk -F',' '{print $1, $2}' data.csv    # 处理CSV
```

**cut（列提取）：**
```bash
cut -d: -f1 /etc/passwd                # 提取第一列
cut -c1-10 file                        # 提取1-10个字符
```

**sort与uniq（排序去重）：**
```bash
sort file | uniq                       # 去重
sort file | uniq -c                    # 去重并计数
sort -t: -k3 -n /etc/passwd            # 按第3列数字排序
sort -u file                           # 等价于 sort | uniq
```

**wc（统计）：**
```bash
wc -l file                             # 行数
wc -w file                             # 单词数
wc -c file                             # 字节数
```

**xargs（参数构建）：**
```bash
find /var/log/wms -name "*.log" | xargs rm   # 删除所有日志
find . -name "*.java" | xargs grep "Entity"  # 搜索Java文件
echo "1 2 3" | xargs -n1                      # 每行一个参数
find . -name "*.tmp" | xargs -I {} rm {}     # -I替换占位符
```

**综合示例（WMS日志分析）：**
```bash
# 统计每小时ERROR数量
awk '/ERROR/ {split($3, t, ":"); print t[1]}' app.log | sort | uniq -c

# 提取10点到12点之间的日志
awk '/11:00/,/13:00/' app.log | grep ERROR

# 查找WMS项目中所有包含@Transactional的Java文件
grep -r "@Transactional" /usr/src/wms-backend --include="*.java" | \
  awk -F: '{print $1}' | sort -u
```

### 追问方向
1. `grep -E` 和 `egrep` 的区别？（现代Linux已统一）
2. awk如何处理带引号的CSV？（如 `"张三","男"`）
3. `xargs -0` 和 `-d` 参数的作用？

### 避坑提示
- ❌ `sed -i` 会直接修改文件，操作不可逆，要先备份
- ❌ awk默认按空格分割，遇到连续空格会合并列
- ✅ 复杂文本处理优先用awk——更强大且更安全

---

## 第14题：Linux SSH与远程文件传输

### 题目
描述Linux SSH密钥登录配置、远程文件传输命令 `scp` 和 `rsync`，以及SSH Tunnel的用法。

### 核心答案

**SSH密钥登录：**
```bash
# 1. 本地生成密钥对
ssh-keygen -t ed25519 -C "wms-server-key"
# 或 RSA：ssh-keygen -t rsa -b 4096

# 2. 上传公钥到服务器
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
# 等价于：
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# 3. 免密码登录
ssh user@server
```

**SSH配置文件（`~/.ssh/config`）：**
```bash
Host wms-prod
    HostName 192.168.1.100
    User tomcat
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    ForwardAgent yes
```

**scp（安全复制）：**
```bash
# 复制到远程
scp /local/file.txt user@server:/remote/path/
scp -r /local/dir user@server:/remote/path/     # 递归复制

# 从远程下载
scp user@server:/remote/file.txt /local/path/

# 指定密钥
scp -i ~/.ssh/custom_key file.txt user@server:/path/

# 远程到远程（经过本地中转）
scp user1@server1:/file.txt user2@server2:/path/
```

**rsync（增量同步）：**
```bash
rsync -avz /local/dir/ user@server:/remote/dir/   # -a保留属性，-v显示详情，-z压缩
rsync -avz --delete /local/dir/ user@server:/remote/dir/  # 删除目标多余文件
rsync -avz -e "ssh -i /path/to/key" /local/ user@server:/remote/  # 指定SSH密钥
rsync -avz --exclude '*.log' /local/ user@server:/remote/  # 排除日志
rsync -avz --dry-run /local/ user@server:/remote/  # 模拟运行（不实际复制）
```

**rsync vs scp：**
- scp：简单粗暴，每次全量复制
- rsync：增量同步，只传差异，网络带宽利用率高

**SSH Tunnel（端口转发）：**
```bash
# 本地端口转发（访问远程服务的远程服务）
ssh -L 8080:remote-service:80 user@jump-server
# 本地8080 → jump-server → remote-service:80

# 远程端口转发（反向隧道，穿透NAT）
ssh -R 8080:localhost:80 user@public-server
# public-server:8080 → 当前机器:80

# 动态端口转发（SOCKS代理）
ssh -D 1080 user@server
# 本地1080作为SOCKS代理，隧道所有流量
```

### 追问方向
1. SSH密钥的权限要求？（`~/.ssh` 700，`~/.ssh/authorized_keys` 600）
2. `ssh-agent` 的作用？（避免多次输入密码）
3. 如何使用SSH Tunnel连接远程数据库？→ `ssh -L 3306:localhost:3306 user@db-server`

### 避坑提示
- ❌ scp在传输大文件时中断需要重传，rsync更适合大文件
- ❌ SSH密钥不能有太宽松的权限（chmod 600），否则会被拒绝
- ✅ WMS项目部署建议使用rsync + SSH KEY实现自动化发布

---

## 第15题：Linux进程通信

### 题目
描述Linux进程间通信（IPC）机制：管道（PIPE）、命名管道（FIFO）、信号（Signal）、共享内存（Shared Memory）、消息队列（Message Queue）。

### 核心答案

**管道（PIPE）：**
```bash
# 匿名管道（只能在有亲缘关系的进程间）
ps -ef | grep java
# shell自动创建管道，左边进程stdout连接到右边进程stdin
```

**命名管道（FIFO）：**
```bash
mkfifo /tmp/myfifo

# 进程1：写
echo "data" > /tmp/myfifo

# 进程2：读
cat /tmp/myfifo

# 应用：WMS中库存服务与消息队列的本地缓冲
```

**信号（Signal）：**
```bash
# 查看所有信号
kill -l

# 常用信号：
# SIGTERM (15)：优雅终止，请求进程自行关闭
# SIGKILL (9)：强制杀死，不可捕获
# SIGUSR1 (10)：用户自定义，Java可捕获做日志轮转
# SIGUSR2 (12)：用户自定义
# SIGHUP (1)：挂起，常用于重新加载配置
# SIGPIPE (13)：管道破裂

# 发送信号
kill -SIGTERM PID
kill -15 PID

# Java进程捕获SIGTERM
# Java的shutdown hook对应SIGTERM
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    // 清理资源
}));
```

**共享内存（Shared Memory）：**
```bash
# 创建共享内存
ipcmk -M 1024          # 创建1KB共享内存
ipcs -m                # 查看共享内存

# 程序中使用shmget/shmat（System V）
# 或 mmap（POSIX）：

# 示例：用mmap创建共享内存
mmap /dev/zero 文件映射
```

**消息队列（Message Queue）：**
```bash
# 创建消息队列
ipcmk -Q                # 创建
ipcs -q                  # 查看

# Java中使用：
# ActiveMQ、Kafka（RMS以外的消息队列）
# 或直接使用System V：
# msgget/msgsnd/msgrcv
```

**Socket（套接字）：**
```bash
# Unix Domain Socket
socket -a                # 查看

# 示例：/var/run/docker.sock
```

### 追问方向
1. 管道和FIFO的底层实现区别？（管道是内核缓冲区，FIFO是特殊文件）
2. 信号量（Semaphore）和互斥锁的区别？
3. WMS项目中如何选择IPC方式？（跨服务器用消息队列，本机用共享内存或管道）

### 避坑提示
- ❌ 不要混淆管道和消息队列——管道是字节流，消息队列是消息单元
- ❌ Java进程被SIGKILL杀死时无法执行shutdown hook
- ✅ WMS项目的进程间通信建议使用Kafka等消息队列，而非直接共享内存

---

## 第16题：Linux监控告警

### 题目
描述Linux监控系统 `Nagios`、`Zabbix`、`Prometheus` 的特点和监控指标采集方式。

### 核心答案

**Nagios（老牌监控）：**
- 特点：插件化、主机/服务分组、告警（邮件/短信）
- 指标采集：通过nrpe插件在远程主机执行检查脚本
- 配置文件：`/etc/nagios/`
- 适用场景：中小规模、对告警要求高的环境

**Zabbix（企业级）：**
- 特点：自动发现、模板继承、Web界面、分布式
- 指标采集：agent主动上报 + server轮询
- 组件：Zabbix Server + Zabbix Agent + MySQL
- 配置文件：`/etc/zabbix/`
- 适用场景：大规模、复杂拓扑、多节点

**Prometheus（云原生）：**
- 特点：时序数据库、Pull模式、PromQL查询语言
- 指标采集：exporter暴露metrics，Prometheus定期拉取
- 组件：Prometheus Server + node_exporter + AlertManager
- 告警规则：`groups: - name: rules file: xxx.yml`
- 适用场景：云原生、Kubernetes、微服务

**常用监控指标：**
```bash
# node_exporter暴露的Linux基础指标
# CPU：node_cpu_seconds_total
# 内存：node_memory_MemAvailable_bytes
# 磁盘：node_filesystem_avail_bytes
# 网络：node_network_receive_bytes_total
# 负载：node_load1

# 采集示例
curl http://localhost:9100/metrics | grep node_cpu
```

**Prometheus + Grafana组合：**
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'wms-backend'
    static_configs:
      - targets: ['localhost:9090']
    metrics_path: '/actuator/prometheus'
```

### 追问方向
1. Pull模式 vs Push模式的区别？（Prometheus是Pull，InfluxDB是Push）
2. Zabbix主动模式 vs 被动模式的区别？
3. 如何在Java应用中暴露Prometheus指标？→ Micrometer + actuator

### 避坑提示
- ❌ 监控agent本身也会消耗资源，要避免对监控对象造成干扰
- ❌ 告警阈值设置过低会产生告警风暴，设置过高会漏报
- ✅ WMS项目建议Prometheus + Grafana监控微服务栈

---

## 第17题：Linux内核参数调优

### 题目
描述Linux内核参数调优方法，重点讲解 `/etc/sysctl.conf` 和 `net.ipv4.tcp_*` 系列参数。

### 核心答案

**sysctl命令：**
```bash
sysctl -a                    # 查看所有内核参数
sysctl -w net.core.somaxconn=1024    # 临时生效，重启失效
sysctl -p                    # 从/etc/sysctl.conf加载参数
sysctl -p /etc/sysctl.d/custom.conf  # 加载自定义配置
```

**常用内核参数：**
```bash
# 内存
vm.swappiness=10             # 内存不足时用swap的程度（0-100）
vm.dirty_ratio=60            # 脏页占比超过此值则强制写回
vm.overcommit_memory=1        # 内存分配策略（0=启发式，1=总是允许，2=严格）

# 文件描述符
fs.file-max=65536            # 系统级最大fd数
fs.nr_open=65536             # 单进程最大fd数
```

**TCP参数（最常考）：**
```bash
# 连接队列
net.core.somaxconn=1024       # TCP监听队列上限（Java服务应调大）
net.core.netdev_max_backlog=5000  # 网卡最大队列长度

# TCP连接参数
net.ipv4.tcp_max_syn_backlog=8192  # SYN队列长度
net.ipv4.tcp_tw_reuse=1       # 允许重用TIME_WAIT连接（client）
net.ipv4.tcp_tw_recycle=1     # 快速回收TIME_WAIT（NAT环境慎用）
net.ipv4.tcp_fin_timeout=30  # FIN_WAIT_2超时时间

# 内存
net.ipv4.tcp_rmem="4096 87380 6291456"   # TCP接收缓冲区
net.ipv4.tcp_wmem="4096 65536 6291456"   # TCP发送缓冲区
net.core.rmem_max=16777216               # socket最大接收缓冲
net.core.wmem_max=16777216               # socket最大发送缓冲

# 连接跟踪
net.netfilter.nf_conntrack_max=1048576   # 连接跟踪表大小
net.nf_conntrack_max=1048576

# 其他
net.ipv4.ip_local_port_range="1024 65535"  # 客户端可用端口范围
net.ipv4.tcp_keepalive_time=600             # TCP保活时间
net.ipv4.tcp_keepalive_intvl=30            # 保活探测间隔
net.ipv4.tcp_keepalive_probes=3             # 保活探测次数
```

**永久配置：**
```bash
# /etc/sysctl.d/99-wms.conf
net.core.somaxconn=2048
net.ipv4.tcp_max_syn_backlog=8192
net.ipv4.tcp_tw_reuse=1
vm.swappiness=10
```

### 追问方向
1. `somaxconn` 和 `tcp_max_syn_backlog` 的关系？（前者是OS级别，后者是TCP级别）
2. Java服务如何读取这些内核参数？（通过NIO的backlog参数）
3. 高并发短连接场景下为什么要调大 `ip_local_port_range`？

### 避坑提示
- ❌ 修改内核参数可能影响系统稳定性，生产环境要逐个验证
- ❌ `tcp_tw_recycle` 在NAT环境下会导致连接异常，要谨慎使用
- ✅ WMS项目在容器化部署时，容器内内核参数受限于宿主机

---

## 第18题：Linux容器支持

### 题目
描述Docker运行在Linux上的内核要求，以及cgroup和Namespace在容器隔离中的作用。

### 核心答案

**Docker运行要求：**
```bash
# 1. 64位Linux内核
uname -r                    # 内核版本 >= 3.10

# 2. cgroup支持
grep cgroup /proc/filesystems   # 应显示cgroup或cgroup2

# 3. Namespace支持
ls /proc/self/ns/           # 应显示各种命名空间
# ipc, uts, mount, net, pid, user, cgroup
```

**cgroup（Control Group，资源限制）：**
- 作用：限制和隔离进程组的资源（CPU、内存、IO、网络）
- v1 vs v2：cgroup v2是新一代接口，统一了资源控制
- 示例：
```bash
# 查看cgroup信息
ls /sys/fs/cgroup/
# 内存限制示例
cat /sys/fs/cgroup/memory/machine.slice/memory.limit_in_bytes
```

**Namespace（命名空间，视图隔离）：**
| Namespace | 作用 | 相关参数 |
|-----------|------|---------|
| PID | 进程ID隔离 | `clone(CLONE_NEWPID)` |
| Network | 网络协议栈隔离 | `clone(CLONE_NEWNET)` |
| Mount | 文件系统挂载隔离 | `clone(CLONE_NEWNS)` |
| IPC | 进程间通信隔离 | `clone(CLONE_NEWIPC)` |
| UTS | 主机名/域名隔离 | `clone(CLONE_NEWUTS)` |
| User | 用户UID/GID隔离 | `clone(CLONE_NEWUSER)` |

**Docker如何利用cgroup和Namespace：**
```bash
# Docker容器的cgroup
ls /sys/fs/cgroup/systemd/docker/

# 查看容器进程在宿主机看到的PID
docker exec container cat /proc/self/proc
# 宿主机上对应的进程PID

# 容器资源限制（docker run）
docker run -m 512m --cpus="1.5" \
  --blkio-weight=500 \
  --oom-kill-disable \
  wms-backend:latest
```

**内核配置检查：**
```bash
# 检查必需的内核模块
modprobe overlay
modprobe br_netfilter

# 检查内核编译选项
grep CONFIG_NAMESPACES /boot/config-$(uname -r)
grep CONFIG_CGROUPS /boot/config-$(uname -r)
```

### 追问方向
1. 容器和虚拟机的本质区别？（容器是进程级隔离，虚拟机是硬件级隔离）
2. Docker的 `--privileged` 模式有什么风险？
3. 如何在容器内查看cgroup限制？→ `cat /sys/fs/cgroup/memory/memory.limit_in_bytes`

### 避坑提示
- ❌ 不要在容器内使用 `--privileged`，除非必要（权限过大）
- ❌ 容器内的root只是容器内的root，映射到宿主机是普通用户
- ✅ WMS项目的Dockerfile中应设置合理的资源限制

---

## 第19题：Linux优雅停止

### 题目
描述Linux信号机制中 `SIGTERM` vs `SIGKILL` 的区别，以及Java进程如何实现graceful shutdown。

### 核心答案

**SIGTERM vs SIGKILL：**

| 特性 | SIGTERM (15) | SIGKILL (9) |
|------|-------------|-------------|
| 默认行为 | 进程自行处理，可被捕获/忽略 | 强制杀死，不可捕获 |
| 优雅停止 | ✅ 可以 | ❌ 不行 |
| 清理资源 | ✅ 可以执行shutdown hook | ❌ 立即终止 |
| 数据库连接 | ✅ 可以正常关闭 | ❌ 连接直接断开 |

**Linux停止进程的方式：**
```bash
kill PID              # 发送SIGTERM（默认）
kill -15 PID          # 显式发送SIGTERM
kill -9 PID           # 强制杀死（SIGKILL）
kill -SIGTERM PID     # 信号名方式

# 查看信号定义
kill -l              # 列出所有信号
```

**Java进程graceful shutdown实现：**
```java
// 1. 注册ShutdownHook
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    logger.info("Received SIGTERM, starting graceful shutdown...");
    
    // 停止接受新请求
    server.shutdown();
    
    // 等待现有请求处理完成（带超时）
    if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
        executor.shutdownNow();
    }
    
    // 关闭数据库连接池
    dataSource.close();
    
    logger.info("Graceful shutdown completed");
}));

// 2. Spring Boot的优雅停止
// application.yml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

**Spring Boot Actuator配置：**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: shutdown
  endpoint:
    shutdown:
      enabled: true
```

**Docker容器优雅停止：**
```dockerfile
# Dockerfile
STOPSIGNAL SIGTERM

# docker-compose.yml
stop_grace_period: 30s
```

**Systemd优雅停止（unit文件）：**
```ini
[Service]
ExecStop=/bin/kill -SIGTERM $MAINPID      # 发送SIGTERM
TimeoutStopSec=60                          # 60秒内未停止则SIGKILL
```

### 追问方向
1. `kill -9` 为什么Java进程无法捕获？（操作系统直接终止，不经过信号处理）
2. 数据库连接池的 `close()` 和 `shutdown()` 区别？
3. Kubernetes中Pod的 `preStop` 钩子的作用？

### 避坑提示
- ❌ 不要用 `kill -9` 停止Java服务——会导致OOM killer记录和连接池未关闭
- ❌ Java的Runtime shutdown hook只保证在有限时间内执行，超过后进程会被强制终止
- ✅ WMS项目建议使用systemd管理，配合合理的TimeoutStopSec实现优雅停止

---

## 第20题：Linux常见故障排查

### 题目
描述Linux常见故障的排查思路：CPU高、内存满、磁盘满、网络不通。

### 核心答案

**排查总原则：由外到内、由易到难、证据先行**

### 1. CPU高排查
```bash
# Step 1：确认CPU使用情况
top                        # 按Cpu排序，看%CPU列
htop                       # 更直观

# Step 2：找到CPU占用最高的进程
ps -eo pid,%cpu,cmd --sort=-%cpu | head -10

# Step 3：如果是Java进程，看线程
jps                        # 找到Java进程PID
top -Hp PID                # 查看该进程的所有线程CPU占用
printf '%x\n' TID          # 将线程ID转为十六进制

# Step 4：查看具体线程堆栈
jstack PID | grep TID_HEX  # 定位CPU高的代码位置

# Step 5：如果是系统问题
sar -u 1 5                 # CPU使用率
```

### 2. 内存满排查
```bash
# Step 1：确认内存使用情况
free -h
top                        # 按MEM排序

# Step 2：找到内存占用最高的进程
ps -eo pid,%mem,cmd --sort=-%mem | head -10

# Step 3：如果是Java进程，检查堆内存
jmap -heap PID             # 查看JVM堆使用情况
jmap -histo PID | head -20 # 查看对象占用

# Step 4：检查OOM Killer
dmesg | grep -i "out of memory"
journalctl -k | grep -i oom

# Step 5：查看内存分配趋势
vmstat 1 10
```

### 3. 磁盘满排查
```bash
# Step 1：确认磁盘使用情况
df -h

# Step 2：找到最大的目录/文件
du -sh /* 2>/dev/null | sort -h | tail -10
du -h --max-depth=1 /       # 限制深度1层

# Step 3：找到大文件
find / -type f -size +100M 2>/dev/null
find / -type f -mtime -7    # 最近7天修改的文件

# Step 4：WMS项目常见问题
# 日志文件过多
du -sh /var/log/wms/
# 临时文件未清理
du -sh /tmp/
# Docker镜像占用
docker system df
docker system prune -a

# Step 5：定位是inode满还是block满
df -i                       # inode使用情况
```

### 4. 网络不通排查
```bash
# Step 1：确认网络配置
ip addr show
ip route show

# Step 2：测试基本连通性
ping 8.8.8.8                # 确认外网连通
ping gateway_ip             # 确认网关可达

# Step 3：测试端口连通性
telnet target_ip port       # 测试TCP
nc -zv target_ip port        # 更简洁的测试

# Step 4：检查防火墙
iptables -L -n | grep target_ip
firewall-cmd --list-all

# Step 5：检查DNS解析
nslookup hostname
dig hostname

# Step 6：抓包分析
tcpdump -i eth0 host target_ip and port port
tcpdump -i eth0 -w /tmp/capture.pcap

# Step 7：查看网络连接状态
ss -s                       # 汇总统计
netstat -an | grep ESTABLISHED | wc -l
```

### WMS项目故障排查示例
```bash
# WMS应用无法访问（8080端口）
1. ss -tlnp | grep 8080              # 确认端口监听
2. lsof -i:8080                       # 查看进程
3. systemctl status wms-app           # 查看服务状态
4. journalctl -u wms-app -n 50        # 查看最近日志
5. ps -ef | grep wms                  # 确认进程存在
6. jstack PID > /tmp/jstack.log      # 导出线程堆栈
```

### 追问方向
1. CPU使用率100%但top显示idle是100%是怎么回事？（可能是用户态和内核态混淆）
2. 内存used很高但available也高说明什么？（cache可以回收）
3. 磁盘写满但df显示还有空间是什么情况？（inode耗尽）

### 避坑提示
- ❌ 排查过程不要直接重启服务——应该先保留证据（进程堆栈、日志）
- ❌ `kill -9` 不是解决方案——要找到根本原因
- ✅ 建议写一个故障排查checklist脚本，标准化操作流程

---

## 附录：WMS项目Linux运维实战场景

**场景1：WMS服务内存溢出**
```
现象：Java进程被OOM Killer杀死
排查：
  1. dmesg | grep java
  2. 查看OOM Killer日志
  3. jmap -heap 导出堆内存
  4. 分析MATdump
解决：调整-Xmx参数，或优化代码内存泄漏
```

**场景2：inventory-service磁盘写满**
```
现象：库存同步失败，日志报错"No space left on device"
排查：
  1. df -h 发现/var/log所在磁盘100%
  2. du -sh /var/log/wms/*
  3. 发现历史日志未压缩
解决：logrotate + 手动清理 + 告警配置
```

**场景3：多实例部署端口冲突**
```
现象：第二台WMS实例启动失败，8080端口已被占用
排查：
  1. ss -tlnp | grep 8080
  2. 发现残留的java进程
解决：kill残留进程，或修改实例端口
```

---

> 本面试题库结合WMS仓库管理系统项目，涵盖Linux运维高频考点。建议配合实际服务器环境练习命令，结合项目代码理解内核参数调优和监控告警的落地实践。
