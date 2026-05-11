# Docker与Kubernetes容器化面试题

> 本面试题基于WMS仓库管理系统（inventory-service-backend）实战代码，涵盖25道高频面试题。
> 项目使用Spring Boot + MyBatis-Plus + RabbitMQ技术栈，通过Dockerfile多环境部署（dev/stage/prod/aws/aliyun）。

---

## 第1题：Docker核心概念（镜像/容器/仓库，容器vs虚拟机区别，Namespace/Cgroups）

### 题目

请解释Docker的三大核心概念：镜像、容器、仓库。并说明容器与虚拟机的核心区别，以及Docker是如何通过Namespace和Cgroups实现资源隔离的。

### 核心答案

**镜像（Image）**：只读模板，包含应用程序及依赖。镜像通过分层结构存储，每一层只记录与前一层的差异（CoW机制）。

**容器（Container）**：镜像的运行实例，是一个进程。容器是镜像的可写层加上运行时状态（网络、存储挂载等）。容器在启动时在镜像层上创建一个薄的可写层。

**仓库（Registry）**：存储和分发镜像的仓库服务。Docker Hub是官方公共仓库；企业通常使用Harbor、阿里云ACR、AWS ECR等私有仓库。

**容器 vs 虚拟机**：

| 维度 | 容器 | 虚拟机 |
|------|------|--------|
| 隔离级别 | 进程级隔离，共享宿主机内核 | 完整操作系统级隔离 |
| 启动速度 | 秒级（~100ms） | 分钟级（~1min） |
| 资源占用 | 极小（MB级） | 较大（GB级） |
| 性能 | 接近原生 | 有虚拟化开销（~5-10%） |
| 安全性 | 共享内核，攻击面较大 | 强隔离，安全性高 |

**Namespace（命名空间）**：Linux内核提供的资源隔离机制。Docker使用以下6种Namespace：
- `pid`：进程隔离（容器内进程看不到宿主机进程）
- `net`：网络隔离（容器有独立的网络栈）
- `ipc`：System V IPC对象隔离
- `mnt`：挂载点隔离（容器有独立的文件系统视图）
- `uts`：主机名/域名隔离
- `user`：用户/组ID映射隔离

**Cgroups（Control Groups）**：限制和隔离进程组的资源使用（CPU、内存、I/O、网络等）。每个容器对应一个或多个Cgroup，Docker通过Cgroups实现：
- 资源限制（`--memory`, `--cpus`）
- 优先级分配
- 资源计量
- 进程控制（冻结/恢复）

### 追问方向

1. 为什么Docker容器比虚拟机轻量？容器启动时发生了什么？
2. 如果容器内想访问宿主机资源（如网络端口），应该如何处理？
3. Namespace和Cgroups是由Docker自动管理的，能否手动查看某个容器的Namespace ID？

### 避坑提示

- 很多候选人把容器和镜像概念混淆，强调"容器是镜像的运行实例，镜像不会变化，容器在其可写层上记录变化"
- 关于Namespace和Cgroups，避免只背名字，要能说清物理含义（比如"mount namespace让容器看到自己独立的文件系统根目录"）

---

## 第2题：Docker镜像（分层结构、写时复制CoW、镜像大小优化、多阶段构建）

### 题目

Docker镜像采用分层结构，请解释什么是写时复制（Copy-on-Write），以及如何在实际项目中优化镜像大小。项目中使用了多环境Dockerfile，请说明多阶段构建的适用场景。

### 核心答案

**分层结构**：Docker镜像由多个只读层叠加而成。每条Dockerfile指令（除CMD/ENTRYPOINT外）都会创建一个新层。层与层之间共享相同的文件（通过内容寻址），节省存储空间。

**写时复制（CoW）**：
- 镜像层是只读的，容器启动时在镜像层上附加一个薄的可写层。
- 当容器需要修改某个文件时，不是直接在镜像层上改，而是将该文件从下层复制到可写层进行修改（CoW策略）。
- 通过`docker history`命令可以看到每层的构建过程和大小。
- CoW确保了多个容器共享同一个镜像时，各自的数据修改互不影响。

**镜像大小优化策略**：
1. **选用更小的基础镜像**：如`alpine`（~5MB）、`distroless`、scratch
2. **减少层数**：将多条RUN指令合并，减少镜像层数量
3. **清理中间产物**：在RUN指令末尾加`&& rm -rf /var/lib/apt/lists/*`清理缓存
4. **使用多阶段构建**：构建阶段使用完整工具链，运行阶段只复制产物
5. **.dockerignore**：排除不必要的构建上下文文件
6. **依赖顺序优化**：将不常变化的层放在前面，利用构建缓存

**多阶段构建**：
```dockerfile
# 阶段1：构建
FROM maven:3.8-openjdk-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
RUN mvn package -DskipTests

# 阶段2：运行
FROM openjdk:17-jre-slim
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**项目中的Dockerfile示例**（`/usr/src/inventory-service-backend/deploy/prod/aws/Dockerfile`）：
```dockerfile
FROM 571600829038.dkr.ecr.us-west-2.amazonaws.com/prod-item/app-baseimage:v7.0-alpha
RUN mkdir -p /sync/log && mkdir -p /data/inventory-service-backend/ && mkdir -p /archive/logs/inventory
COPY inventory-app/build/libs/inventory-app-0.0.1-PROD-SNAPSHOT.jar /data/inventory-service-backend/inventory-app-0.0.1-SNAPSHOT.jar
COPY deploy/prod/aws/supervisord.conf /etc/supervisor/supervisord.conf
CMD ["/usr/bin/supervisord","-c","/etc/supervisor/supervisord.conf"]
```

### 追问方向

1. 项目中Dockerfile只有4条指令，分析为什么这样设计？
2. 如果构建阶段需要编译（Java/Go/C++），运行阶段需要JRE/运行时，如何通过多阶段构建减少最终镜像大小？
3. 如何通过`docker history`分析镜像各层大小占比？

### 避坑提示

- 提到多阶段构建时，强调构建阶段和运行阶段的**职责分离**，构建产物通过`--from=builder`复制，而非RUN打包
- 关于CoW，不要简单说"复制"，要说清楚"首次修改时才复制"的惰性语义

---

## 第3题：Dockerfile常用指令

### 题目

请列举Dockerfile中常用指令的含义和用法：`FROM`、`RUN`、`COPY`、`ADD`、`EXPOSE`、`WORKDIR`、`ENV`、`ENTRYPOINT`、`CMD`。并说明`CMD`和`ENTRYPOINT`的区别。

### 核心答案

**FROM**：指定基础镜像，必须是第一条非注释指令。
```dockerfile
FROM openjdk:17-jre-slim
```

**RUN**：在构建时执行命令，创建新层。
```dockerfile
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
```

**COPY**：将构建上下文中的文件/目录复制到镜像内。
```dockerfile
COPY inventory-app/build/libs/app.jar /data/app/app.jar
COPY deploy/prod/aws/supervisord.conf /etc/supervisor/supervisord.conf
```

**ADD**：功能类似COPY，但多了两个特性：
- 可以从URL下载文件
- 可以自动解压tar包（对压缩文件）

> ⚠️ **推荐优先使用COPY**，ADD语义更复杂，不透明。除非需要URL下载或tar自动解压，才用ADD。

**EXPOSE**：声明容器运行时监听的端口（仅文档作用，不实际绑定端口）。
```dockerfile
EXPOSE 8080 8443
```

**WORKDIR**：设置工作目录，类似`cd`。如果目录不存在会自动创建。
```dockerfile
WORKDIR /data/inventory-service-backend
```

**ENV**：设置环境变量。
```dockerfile
ENV JAVA_OPTS="-Xms512m -Xmx1024m"
ENV SPRING_PROFILES_ACTIVE=prod
```

**ENTRYPOINT**：指定容器启动时执行的命令，**不可被覆盖**（除非加`--entrypoint`参数）。容器作为可执行程序时使用。
```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**CMD**：指定容器启动时执行的默认命令，**可被docker run参数覆盖**。提供默认参数。
```dockerfile
# 形式1：exec形式（推荐）
CMD ["java", "-jar", "app.jar"]

# 形式2：shell形式
CMD java -jar app.jar

# 形式3：作为ENTRYPOINT的参数
CMD ["--server.port=8080"]
```

**CMD vs ENTRYPOINT 区别**：
- `ENTRYPOINT`定义容器"是什么"（可执行程序）
- `CMD`定义"默认参数"，可被命令行覆盖
- 组合使用：`ENTRYPOINT`定义命令，`CMD`定义默认参数

```dockerfile
# java -jar app.jar --server.port=8080
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--server.port=8080"]
```

### 追问方向

1. 项目中使用了`supervisord`，为什么？CMD为什么用exec形式数组？
2. 如果在Dockerfile中同时写了ENTRYPOINT和CMD，哪个生效？
3. `RUN`、`COPY`、`ADD`指令产生的层有什么区别？`docker history`如何验证？

### 避坑提示

- CMD和ENTRYPOINT的区别是高频考点。强调**覆盖行为不同**：docker run的指令参数会覆盖CMD，但不会覆盖ENTRYPOINT（除非加--entrypoint）
- shell形式的CMD/ENTRYPOINT会经过shell解析，可能丢失信号，导致容器无法优雅停止

---

## 第4题：Dockerfile优化（减少层数、顺序优化、.dockerignore、构建缓存）

### 题目

请说明如何优化Dockerfile以提升构建速度和减小镜像体积。项目中如何通过.dockerignore和构建上下文优化构建？

### 核心答案

**减少层数**：
- 合并多条RUN指令：多条命令用`&&`连接，减少层数
- 清理不必要的文件：在同一条RUN中完成安装和清理（`&& rm -rf`）

```dockerfile
# 优化前（3层）
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y nginx

# 优化后（1层）
RUN apt-get update && apt-get install -y curl nginx && rm -rf /var/lib/apt/lists/*
```

**指令顺序优化（利用构建缓存）**：
- 不常变化的指令放在前面（如基础镜像、依赖安装）
- 变化频繁的指令放在后面（如代码复制、应用启动）

```dockerfile
# 优化顺序：依赖层放前面
COPY pom.xml /app/
RUN mvn dependency:go-offline   # 利用缓存，pom.xml不变时不重新下载
COPY src /app/src               # 代码变化时，只需重新这一层
RUN mvn package
```

**.dockerignore**：
- 排除不需要放入构建上下文的文件，减小构建包体积，避免意外文件覆盖镜像内容
- 常见排除项：`.git`、`.gradle`、`build/`、`node_modules/`、`*.md`、`.dockerignore`自身

```
# .dockerignore示例
.git
.gitignore
*.md
build/
.gradle/
node_modules/
.env
.idea/
*.iml
```

**构建缓存**：
- Docker会缓存每一层的结果
- 当某层指令变化时，该层及之后所有层缓存失效
- `--no-cache=true`可强制不使用缓存

**多阶段构建优化**：
- 构建阶段使用完整工具链（编译环境）
- 运行阶段使用精简运行时（JRE/Node运行时）

### 项目中的实践

项目Dockerfile结构分析（`/usr/src/inventory-service-backend/deploy/prod/aws/Dockerfile`）：
```dockerfile
FROM 571600829038.dkr.ecr.us-west-2.amazonaws.com/prod-item/app-baseimage:v7.0-alpha
RUN mkdir -p /sync/log && mkdir -p /data/inventory-service-backend/ && mkdir -p /archive/logs/inventory
COPY inventory-app/build/libs/inventory-app-0.0.1-PROD-SNAPSHOT.jar /data/inventory-service-backend/
COPY deploy/prod/aws/supervisord.conf /etc/supervisor/supervisord.conf
CMD ["/usr/bin/supervisord","-c","/etc/supervisor/supervisord.conf"]
```

**优化建议**：
1. 当前RUN指令可以合并，减少层数
2. 建议增加`.dockerignore`，排除`build/`目录（构建产物应在COPY时已存在）
3. 建议使用多阶段构建，将编译阶段和运行阶段分离

### 追问方向

1. 如果pom.xml没变，但代码变了，Docker缓存还能利用吗？为什么？
2. `docker build --cache-from`的作用是什么？适用于什么场景？
3. 如何诊断Docker构建缓存失效的原因？

### 避坑提示

- 强调构建缓存是Dockerfile优化的核心，顺序错误的Dockerfile每次都需要重新构建所有层，导致构建时间从几秒变成十几分钟
- 清理缓存要放在同一层RUN中，否则清理操作本身产生的层不会被清理

---

## 第5题：Docker网络（bridge/host/none/container模式，容器通信）

### 题目

Docker提供多种网络模式：`bridge`、`host`、`none`、`container`。请解释各模式的工作原理，以及跨容器通信如何实现。

### 核心答案

**Docker网络模式**：

| 模式 | 说明 | 使用场景 |
|------|------|----------|
| `bridge`（默认） | 容器连接到docker0网桥，拥有独立的Network Namespace | 绝大多数场景 |
| `host` | 容器直接使用宿主机网络，无隔离 | 性能敏感、无端口冲突场景 |
| `none` | 容器有独立的Network Namespace，但无网络配置 | 离线计算 |
| `container:<name>` | 容器共享另一个容器的网络栈 | sidecar模式 |

**bridge模式原理**：
- Docker daemon启动时创建`docker0`网桥（172.17.0.0/16）
- 每创建一个容器，Docker在宿主机上创建veth pair（一端在容器内，一端在docker0上）
- 容器通过veth pair连接到docker0网桥，实现与宿主机及其他容器的通信
- 容器访问外部网络需要NAT（网络地址转换）

**容器间通信**：
1. **通过容器名（推荐）**：同一bridge网络下，Docker内置DNS会将容器名解析为容器IP
2. **通过Link（已废弃）**：显式声明依赖关系，单向通信
3. **通过Service名**：在Docker Compose或K8s中，服务名即DNS名

```bash
# 查看容器网络
docker inspect --format='{{json .NetworkSettings.Networks}}' <container_id>

# 查看网络列表
docker network ls
```

**自定义bridge网络**：
```bash
docker network create --driver bridge my-network
docker run --network my-network --name app-container my-app
```

### 追问方向

1. 容器内如何访问宿主机？宿主机如何访问容器？
2. 容器间想通过127.0.0.1:port访问，为什么不行？
3. 如何让容器使用宿主机网络的特定端口（如8080）？

### 避坑提示

- bridge模式下容器通过Docker内置DNS解析服务名，不是系统DNS
- `docker run -p 8080:8080`是将宿主机8080端口映射到容器8080端口，这是NAT端口映射，不是"让容器使用宿主机端口"

---

## 第6题：Docker存储（数据卷/Volume、挂载宿主目录、存储驱动overlay2）

### 题目

Docker容器内的数据在容器删除时会丢失。请说明Docker的持久化存储方案：数据卷（Volume）、挂载宿主目录、存储驱动。项目中哪些数据需要持久化？

### 核心答案

**数据卷（Volume）**：
- Docker管理的持久化存储，不依赖容器生命周期
- 在宿主机存储于`/var/lib/docker/volumes/`
- 多个容器可以共享同一个Volume
- 容器对Volume的读写直接绕过容器文件系统

```bash
# 创建命名卷
docker volume create my-data

# 容器使用卷
docker run -v my-data:/data/app my-app

# 查看卷信息
docker volume inspect my-data
```

**挂载宿主目录（Bind Mount）**：
- 将宿主机指定目录映射到容器内
- 适用于配置文件、日志输出、开发环境代码挂载

```bash
docker run -v /host/logs:/container/logs my-app
docker run -v $(pwd)/config:/app/config:ro my-app  # 只读挂载
```

**存储驱动（Storage Driver）**：
- Docker使用存储驱动管理镜像层和容器可写层
- 常见驱动：`overlay2`（生产推荐）、`aufs`、`devicemapper`、`btrfs`
- `overlay2`：将容器层和镜像层合并展示，性能好，磁盘空间占用低

```bash
# 查看当前存储驱动
docker info | grep "Storage Driver"
```

**项目中的持久化需求**（从Dockerfile分析）：
```dockerfile
RUN mkdir -p /sync/log && mkdir -p /data/inventory-service-backend/ && mkdir -p /archive/logs/inventory
```
- `/sync/log`：同步日志目录
- `/data/inventory-service-backend/`：应用数据目录
- `/archive/logs/inventory`：归档日志

这些目录在容器删除后会丢失，生产环境需要：
1. 挂载宿主目录或NFS存储
2. 使用Docker Volume
3. 考虑日志收集方案（EFK/ELK）

### 追问方向

1. Volume和Bind Mount的核心区别是什么？什么时候用哪个？
2. 如果容器需要写大量小文件（如缓存），选择哪种存储方案？
3. 如何查看Docker Volume的实际存储路径和占用空间？

### 避坑提示

- 挂载宿主目录时，注意权限问题（UID/GID匹配）
- 生产环境避免使用匿名卷（不指定名称的`-v /data`），难以管理

---

## 第7题：Docker Compose（编排多个容器，docker-compose.yml语法）

### 题目

Docker Compose用于编排多个容器。请说明docker-compose.yml的核心配置项，并以项目中的微服务架构为例，设计一个简化的编排方案。

### 核心答案

**docker-compose.yml核心配置**：

```yaml
version: "3.8"  # Compose文件格式版本

services:
  app:                    # 服务名
    build:                # 从Dockerfile构建
      context: .
      dockerfile: Dockerfile
    image: my-app:1.0     # 或直接指定镜像
    container_name: app   # 固定容器名
    ports:                # 端口映射
      - "8080:8080"
    environment:          # 环境变量
      - SPRING_PROFILES=prod
      - DB_HOST=db
    depends_on:           # 依赖顺序（仅保证启动顺序，不保证健康）
      - db
      - redis
    volumes:              # 存储卷
      - ./logs:/sync/log
      - app-data:/data
    networks:             # 网络
      - backend
    restart: unless-stopped  # 重启策略
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:                    # 数据库服务
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: wms
    volumes:
      - db-data:/var/lib/mysql
    networks:
      - backend

  redis:
    image: redis:7-alpine
    networks:
      - backend

volumes:                 # 声明命名卷
  app-data:
  db-data:

networks:                # 声明自定义网络
  backend:
    driver: bridge
```

**常用命令**：
```bash
docker-compose up -d          # 启动
docker-compose down           # 停止并删除
docker-compose ps             # 查看状态
docker-compose logs -f app    # 查看日志
docker-compose exec app bash  # 进入容器
docker-compose restart app    # 重启服务
docker-compose scale app=3   # 扩容（v3已废弃，用up --scale）
```

### 项目应用场景

WMS系统典型依赖关系：
```
[前端] → [Spring Boot后端] → [MySQL] → [Redis]
                         ↓
                    [RabbitMQ]
```

### 追问方向

1. `depends_on`能保证容器健康吗？如何实现真正的依赖健康等待？
2. 如何使用Docker Compose实现开发/测试/生产环境切换？
3. `restart: always`和`restart: unless-stopped`有什么区别？

### 避坑提示

- v3版本的`docker-compose scale`已被废弃，扩容通过`docker-compose up --scale`实现
- `depends_on`只保证启动顺序，不保证服务就绪。如需健康检查等待，使用`healthcheck`和`condition`

---

## 第8题：Docker命令（run/ps/images/rm/exec/logs/commit/build/stop/start）

### 题目

请写出以下Docker命令的使用方式和常见参数：`docker run`、`docker ps`、`docker images`、`docker rm`、`docker exec`、`docker logs`、`docker commit`、`docker build`、`docker stop`、`docker start`。

### 核心答案

**docker run**：创建并启动容器
```bash
docker run -d \                    # 后台运行
           -p 8080:8080 \           # 端口映射
           -v /data:/data \         # 存储挂载
           -e SPRING_PROFILES=prod \ # 环境变量
           --name my-app \         # 容器名
           --network my-net \       # 网络
           --restart unless-stopped \ # 重启策略
           --memory 512m \          # 内存限制
           --cpus 1.0 \             # CPU限制
           my-app:1.0              # 镜像名
```

**docker ps**：查看运行中的容器
```bash
docker ps              # 运行中
docker ps -a           # 所有容器（包括停止）
docker ps -q           # 只显示容器ID
docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}"  # 格式化输出
```

**docker images**：查看本地镜像
```bash
docker images
docker images -q               # 只显示ID
docker images --filter "dangling=true"  # 悬空镜像
```

**docker rm**：删除容器
```bash
docker rm my-app              # 删除停止的容器
docker rm -f my-app           # 强制删除（运行中）
docker rm $(docker ps -aq)    # 删除所有容器
```

**docker exec**：在运行中的容器内执行命令
```bash
docker exec -it my-app bash       # 进入交互式终端
docker exec my-app ls /data       # 执行单条命令
docker exec -u root my-app bash   # 以指定用户执行
```

**docker logs**：查看容器日志
```bash
docker logs my-app              # 查看日志
docker logs -f my-app           # 实时跟踪
docker logs --tail 100 my-app    # 最后100行
docker logs --since 1h my-app    # 最近1小时
```

**docker commit**：从容器创建镜像（不推荐用于生产）
```bash
docker commit -m "configured" my-app my-app:configured
```

**docker build**：构建镜像
```bash
docker build -t my-app:1.0 .              # 从当前目录构建
docker build -t my-app:1.0 -f Dockerfile.production .  # 指定Dockerfile
docker build --no-cache -t my-app:1.0 .   # 禁用缓存
docker build --build-arg JAR_FILE=app.jar -t my-app:1.0 .  # 构建参数
```

**docker stop/start**：停止/启动容器
```bash
docker stop my-app             # 优雅停止（发送SIGTERM）
docker stop -t 10 my-app       # 等待10秒后强制SIGKILL
docker start my-app            # 启动已停止的容器
docker restart my-app           # 重启
```

### 追问方向

1. `docker run -d`和`docker run`的区别？容器后台运行时如何查看日志？
2. `docker exec`和`docker attach`的区别？
3. 如何通过日志排查容器启动失败的原因？

### 避坑提示

- `docker rm`不能删除运行中的容器，需要先`docker stop`或加`-f`强制删除
- `docker logs`只记录容器主进程（PID 1）的输出，不记录`docker exec`进入后的操作

---

## 第9题：Docker vs Podman（区别、Rootless容器、安全性对比）

### 题目

Podman被称为Docker的无守护进程替代品。请说明Docker与Podman的核心区别，以及Rootless容器的工作原理和安全性考量。

### 核心答案

**核心架构区别**：

| 维度 | Docker | Podman |
|------|--------|--------|
| 守护进程 | 需要dockerd守护进程（root权限） | 无守护进程，fork-exec模型 |
| 容器运行用户 | root或指定用户 | 与启动用户一致 |
| 镜像存储 | `/var/lib/docker` | `/var/lib/containers` |
| 兼容性 | Docker CLI标准 | 兼容Docker CLI（别名`alias docker=podman`） |
| 镜像来源 | Docker Hub/私有仓库 | Docker Hub/ Quay/私有仓库 |

**Rootless容器**：
- Podman设计为无root运行，容器内的root映射为宿主机非特权用户（UID/GID映射）
- 通过`--userns=keep-id`指定容器内用户映射到宿主机特定用户

```bash
# Podman以普通用户运行
podman run -d my-app

# Docker需要额外配置才能rootless
dockerd-rootless-setuptool.sh install
```

**安全性对比**：

| 安全维度 | Docker | Podman |
|----------|--------|--------|
| 守护进程风险 | dockerd是root进程，是攻击目标 | 无root守护进程，风险更低 |
| 容器逃逸 | 需要root权限运行，逃逸风险较高 | 默认非root，隔离更好 |
| CVE攻击面 | Docker daemon暴露API | 无持久监听端口 |
| 用户命名空间 | 需显式开启 | 默认支持 |

**Podman额外特性**：
- **Pod概念**：Podman有Pod（类似K8s Pod），一组容器共享网络/存储
- **Systemd集成**：Podman可生成systemd unit文件管理容器

```bash
# Podman生成systemd服务
podman generate systemd --files --name my-app
```

### 追问方向

1. 为什么企业逐渐采用Podman替代Docker？迁移成本高吗？
2. Kubernetes（K8s）中使用的是containerd还是Docker？为什么？
3. Rootless容器的限制有哪些？哪些功能无法使用？

### 避坑提示

- 不要简单说"Podman更好"，要客观分析两者适用场景
- Kubernetes默认使用containerd或CRI-O，不直接使用Docker，Docker只是提供了封装
- Rootless Podman在用户命名空间映射上有一些限制（如端口映射需要配置）

---

## 第10题：K8s架构（Master/Node节点，API Server/etcd/Controller/Scheduler组件）

### 题目

Kubernetes采用Master-Node架构。请说明Master节点和Node节点的核心组件及其职责，以及各组件之间如何协作完成容器编排。

### 核心答案

**整体架构**：
```
┌─────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                 │
│  ┌─────────────┐         ┌─────────────────────┐   │
│  │   Master    │         │       Node 1        │   │
│  │  ┌────────┐ │         │  ┌───────────────┐   │   │
│  │  │API Srv │◄─────────┼──┤ Kubelet        │   │   │
│  │  └────────┘ │         │  │ Kube-Proxy    │   │   │
│  │  ┌────────┐ │         │  │ Container Runtime│  │   │
│  │  │ etcd   │ │         │  │  Pod1  Pod2    │   │   │
│  │  └────────┘ │         │  └───────────────┘   │   │
│  │  ┌────────┐ │         └─────────────────────┘   │
│  │  │Schedulr│ │                                   │
│  │  └────────┘ │         ┌─────────────────────┐   │
│  │  ┌────────┐ │         │       Node 2        │   │
│  │  │CtrlMgr │ │         │  ┌───────────────┐   │   │
│  │  └────────┘ │         │  │ Kubelet        │   │   │
│  └─────────────┘         │  │ Kube-Proxy    │   │   │
│                           │  │ Container Runtime│  │   │
│                           │  │  Pod3  Pod4    │   │   │
│                           │  └───────────────┘   │   │
│                           └─────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

**Master节点组件**：

| 组件 | 职责 |
|------|------|
| **API Server** | K8s统一入口，接收REST请求，提供认证/授权/准入控制，所有组件通过它交互 |
| **etcd** | 高可用键值存储，保存集群所有状态（Pod/Service/Deployment等），RAFT协议保证一致性 |
| **Scheduler** | 监听新建Pod，为Pod选择最优Node（考虑资源、亲和性、污点等） |
| **Controller Manager** | 运行各类控制器（Deployment/ReplicaSet/StatefulSet/Job等），确保期望状态与实际状态一致 |

**Node节点组件**：

| 组件 | 职责 |
|------|------|
| **Kubelet** | 节点代理，负责向API Server注册节点、维护Pod生命周期、监控容器健康 |
| **Kube-Proxy** | 维护节点上的网络规则（iptables/ipvs），实现Service负载均衡 |
| **Container Runtime** | 实际运行容器（containerd/CRI-O），负责下载镜像、启动容器 |

**协作流程**（Pod创建过程）：
1. 用户通过kubectl发送`kubectl apply -f pod.yaml`
2. 请求到API Server，认证/授权/准入后写入etcd
3. Scheduler监听到新建Pod，通过API Server获取未调度的Pod
4. Scheduler根据资源/策略选择Node，通过API Server更新Pod的`.spec.nodeName`
5. 目标Node上的Kubelet监听到分配给自己的Pod
6. Kubelet调用Container Runtime下载镜像、创建容器、启动容器
7. Kubelet将Pod状态汇报给API Server

### 追问方向

1. etcd为什么是K8s最关键的组件？如果etcd集群发生脑裂会怎样？
2. Controller Manager和Scheduler都是Master上的进程，它们是如何感知Pod变化的？（通过API Server的Watch机制）
3. Kube-Proxy的工作模式（iptables vs IPVS）有什么区别？

### 避坑提示

- 不要将Kubelet和Kube-Proxy混淆：Kubelet是Pod的守护者（管理容器生命周期），Kube-Proxy是Service的网络代理
- etcd存储的是期望状态，Kubelet上报的是实际状态，两者不一致时Controller会尝试协调

---

## 第11题：K8s核心概念（Pod/Deployment/Service/Ingress/ConfigMap/Secret/Volume/Namespace）

### 题目

请解释Kubernetes的核心资源对象：Pod、Deployment、Service、Ingress、ConfigMap、Secret、Volume、Namespace，并说明它们之间的关系。

### 核心答案

**Pod**：
- K8s最小调度单元，一个Pod包含一个或多个容器
- Pod内容器共享Network Namespace、UTS（主机名）、IPC
- Pod内容器可以通过localhost互相访问
- 每个Pod有独立IP（Pod IP）

**Deployment**：
- 管理Pod副本数、滚动更新、回滚
- 声明式管理Pod的期望状态（replicas、镜像版本、资源限制）
- Deployment通过ReplicaSet间接管理Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: inventory-app
  template:
    metadata:
      labels:
        app: inventory-app
    spec:
      containers:
      - name: app
        image: inventory-app:1.0
        ports:
        - containerPort: 8080
```

**Service**：
- 为Pod提供稳定的访问入口（ClusterIP/NodePort/LoadBalancer）
- Service通过Label Selector匹配后端Pod
- 提供Pod的负载均衡和DNS发现

**Ingress**：
- HTTP/HTTPS七层路由规则
- 基于域名、路径将请求路由到后端Service
- 需要Ingress Controller（如Nginx Ingress）配合使用

**ConfigMap**：
- 存储非敏感配置（键值对、配置文件）
- 可以作为环境变量、命令行参数、文件挂载

**Secret**：
- 存储敏感信息（密码、Token、证书）
- Base64编码存储（不加密，需配合K8s RBAC限制访问）
- 支持TLS证书等类型

**Volume**：
- Pod级别的存储抽象
- 类型：emptyDir、hostPath、PV/PVC、ConfigMap、Secret等
- 容器挂载Volume后如同访问本地目录

**Namespace**：
- 集群内的虚拟隔离分区
- 用于多租户、资源分组、权限隔离
- 常见命名空间：`default`、`kube-system`、`kube-public`

**资源关系**：
```
Deployment → ReplicaSet → Pod → Volume/ConfigMap/Secret
                ↓
            Service → Ingress
```

### 追问方向

1. Pod内的多个容器是如何共享网络的？为什么可以用localhost访问？
2. Deployment和Service是同一个概念吗？有什么区别？
3. 为什么需要Ingress？Service不是已经能暴露服务了吗？

### 避坑提示

- Pod不是容器，Pod是包装容器的抽象。面试时不要说"Pod就是容器"
- Service的ClusterIP是虚拟IP，不占用实际网络资源，只在K8s内部生效

---

## 第12题：Pod生命周期（Pending/Running/Succeeded/Failed/Unknown，restartPolicy）

### 题目

请说明Pod的完整生命周期流程，包括各种Pod状态（Pending/Running/Succeeded/Failed/Unknown）的含义，以及restartPolicy的作用。

### 核心答案

**Pod生命周期状态**：

| 状态 | 含义 |
|------|------|
| **Pending** | Pod已被K8s接受，但容器镜像未创建/调度未完成（等待调度或拉取镜像） |
| **Running** | Pod已绑定到Node，所有容器已创建，至少有一个容器在运行 |
| **Succeeded** | Pod中所有容器正常终止（exit code=0），不会被重启 |
| **Failed** | Pod中所有容器都已终止，且至少有一个非正常退出（exit code≠0） |
| **Unknown** | 无法获取Pod状态（通常是Node通信问题） |

**Pod生命周期阶段（phase）**：
- `Pending` → `Running` → `Succeeded`/`Failed`
- Job类型Pod：最终状态为Succeeded或Failed
- 长期运行服务（Deployment）：Running状态会被不断保持，容器退出后重启

**restartPolicy**：

| 策略 | 行为 |
|------|------|
| `Always`（默认） | 容器终止后总是重启（适用于Deployment管理的长期运行服务） |
| `OnFailure` | 容器非正常退出（exit code≠0）时重启 |
| `Never` | 不重启（适用于Job/CronJob） |

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: inventory-app
spec:
  restartPolicy: Always
  containers:
  - name: app
    image: inventory-app:1.0
```

**容器状态（Container States）**：
- `Waiting`：等待启动
- `Running`：正在运行
- `Terminated`：已终止

**initContainer**：
- Pod启动前执行一次的容器
- 按顺序执行，所有initContainer成功后才启动主容器
- 常用于等待依赖服务、数据初始化

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z db:3306; do echo waiting; sleep 2; done']
  containers:
  - name: app
    image: inventory-app:1.0
```

### 追问方向

1. Pod处于Pending状态超过5分钟，可能的原因是什么？
2. restartPolicy是Always的Pod，如果容器内Java进程OOM，K8s会怎么处理？
3. 如何通过`kubectl describe pod`判断Pod终止原因？

### 避坑提示

- Pod的phase（Pending/Running等）是聚合状态，Container级别的状态需要通过`kubectl describe`查看
- restartPolicy的默认值是Always，不要误以为没有restartPolicy容器就不会重启

---

## 第13题：K8s ReplicaSet vs Deployment（区别、滚动更新、回滚）

### 题目

ReplicaSet和Deployment有什么关系？Deployment的滚动更新（RollingUpdate）是如何工作的？如何实现回滚？

### 核心答案

**ReplicaSet vs Deployment关系**：

| 维度 | ReplicaSet | Deployment |
|------|------------|------------|
| 层级 | 直接管理Pod | 通过ReplicaSet间接管理Pod |
| 功能 | 维持Pod副本数 | 声明式更新、回滚、滚动更新 |
| 使用方式 | 不单独使用 | 与Deployment一起声明 |
| 版本管理 | 无 | 记录历史版本，支持回滚 |

```
Deployment → ReplicaSet（版本1） → Pod × 3
          → ReplicaSet（版本2） → Pod × 3  （滚动更新中）
```

**滚动更新（RollingUpdate）**：
- 逐步替换旧版本Pod，确保服务高可用
- 关键参数：`maxUnavailable`（最多不可用）、`maxSurge`（最多多创建）

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # 最多1个不可用
      maxSurge: 1         # 最多多创建1个
```

**滚动更新流程**：
1. 创建新ReplicaSet（v2）
2. 减少v1的replicas，增加v2的replicas
3. 逐步完成替换，直到v1 replicas=0
4. 服务始终有minAvailable个Pod可用

**回滚操作**：

```bash
# 查看历史版本
kubectl rollout history deployment/inventory-app

# 回滚到上一版本
kubectl rollout undo deployment/inventory-app

# 回滚到指定版本
kubectl rollout undo deployment/inventory-app --to-revision=2

# 暂停滚动（用于检查问题）
kubectl rollout pause deployment/inventory-app

# 恢复滚动
kubectl rollout resume deployment/inventory-app
```

**Deployment配置示例**（项目场景）：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inventory-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: inventory-service
  template:
    metadata:
      labels:
        app: inventory-service
    spec:
      containers:
      - name: app
        image: 571600829038.dkr.ecr.us-west-2.amazonaws.com/prod-item/app:v7.0-alpha
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

### 追问方向

1. `maxUnavailable=0`且`maxSurge=1`时，滚动更新过程中最多会有多少个Pod？
2. 如何判断滚动更新是否成功？有哪些观察指标？
3. 如果滚动更新过程中新版本有bug，如何快速回滚？Pod会中断服务吗？

### 避坑提示

- 滚动更新期间Service会同时负载均衡到新旧版本Pod，如果新版本有问题，会影响部分请求
- 回滚是立即触发的，不需要等待当前滚动更新完成
- 使用`kubectl get rs`可以看到Deployment创建了多个ReplicaSet（每版本一个）

---

## 第14题：K8s Service（ClusterIP/NodePort/LoadBalancer/ExternalName，服务发现）

### 题目

Kubernetes Service有四种类型：ClusterIP、NodePort、LoadBalancer、ExternalName。请说明各类型的适用场景，以及Service如何实现服务发现。

### 核心答案

**Service类型**：

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| **ClusterIP**（默认） | 分配集群内部虚拟IP，只能集群内访问 | K8s集群内部微服务间通信 |
| **NodePort** | 在每个Node的固定端口暴露服务 | 开发环境、小规模集群 |
| **LoadBalancer** | 通过云厂商LB暴露服务 | 生产环境对接云负载均衡器 |
| **ExternalName** | 将Service映射到外部DNS名称 | 访问外部服务（如数据库） |

**ClusterIP**：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: inventory-service
spec:
  type: ClusterIP
  selector:
    app: inventory-service
  ports:
  - port: 80        # Service端口
    targetPort: 8080 # 后端Pod端口
```

**NodePort**：
```yaml
spec:
  type: NodePort
  selector:
    app: inventory-service
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080  # Node上固定端口（30000-32767）
```
访问方式：`http://<Node-IP>:30080`

**LoadBalancer**：
```yaml
spec:
  type: LoadBalancer
  selector:
    app: inventory-service
  ports:
  - port: 80
    targetPort: 8080
```
自动创建云厂商负载均衡器（AWS ELB、阿里云SLB）。

**ExternalName**：
```yaml
spec:
  type: ExternalName
  externalName: db.example.com  # 外部数据库域名
```
集群内访问：`http://inventory-db` → 解析为`db.example.com`

**Headless Service**：
```yaml
spec:
  clusterIP: None  # 无ClusterIP，DNS直接解析Pod IP
  selector:
    app: inventory-service
```
用于有状态服务（StatefulSet）的稳定网络标识。

**服务发现机制**：
1. **环境变量**：Kubelet在Pod启动时注入同Namespace下所有Service的IP和端口（`${SERVICE_NAME}_SERVICE_HOST`）
2. **DNS（推荐）**：`http://inventory-service`自动解析为ClusterIP，DNS由CoreDNS（kube-dns）提供

### 追问方向

1. ClusterIP是虚拟IP，Pod如何通过Service IP访问后端Pod的？
2. 同一个集群中，不同Namespace的Service如何互相访问？
3. ExternalName和Endpoint有什么区别？什么时候用哪个？

### 避坑提示

- ClusterIP是kube-proxy通过iptables/ipvs规则实现的，不是真实网络接口
- 环境变量服务发现依赖Pod启动顺序，后启动的Pod无法通过环境变量访问先启动的Service（DNS方式更可靠）

---

## 第15题：K8s Ingress（HTTP/HTTPS路由，Ingress Controller，路径重写）

### 题目

Ingress用于HTTP/HTTPS七层路由。请说明Ingress的工作原理，以及如何配置基于域名和路径的路由规则。

### 核心答案

**Ingress组成**：
- **Ingress资源**：声明路由规则的K8s对象
- **Ingress Controller**：实际处理请求的反向代理（Nginx/Traefik/Contour等）

**工作原理**：
```
Client → Ingress Controller（Nginx） → Service → Pod
                    ↓
              读取Ingress规则，决定路由目标
```

**Ingress规则示例**：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wms-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - wms.example.com
    secretName: wms-tls
  rules:
  - host: wms.example.com
    http:
      paths:
      - path: /inventory
        pathType: Prefix
        backend:
          service:
            name: inventory-service
            port:
              number: 80
      - path: /order
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service: api-service
          port:
            number: 80
```

**路径重写**：
```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - http:
      paths:
      - path: /inventory(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: inventory-service
            port:
              number: 80
```
访问`/inventory/api/users`会被重写为`/api/users`。

**HTTPS配置**：
```yaml
spec:
  tls:
  - hosts:
    - wms.example.com
    secretName: wms-tls  # 通过Secret存储证书
```

**多域名Ingress**：
```yaml
spec:
  rules:
  - host: app1.example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.example.com
    http:
      paths:
      - path: /
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

### 追问方向

1. Ingress Controller和Ingress资源的关系是什么？没有Ingress Controller，Ingress能用吗？
2. Ingress的TLS是如何工作的？需要手动续期证书吗？（结合cert-manager）
3. Ingress的性能瓶颈在哪里？大规模集群如何优化？

### 避坑提示

- Ingress是七层HTTP/HTTPS代理，无法处理TCP/UDP四层流量（用Service的LoadBalancer）
- Ingress Controller需要单独部署，不是K8s自带组件

---

## 第16题：K8s ConfigMap和Secret（创建方式、环境变量注入、Volume挂载）

### 题目

ConfigMap和Secret是K8s中存储配置和敏感信息的资源。请说明它们的创建方式，以及如何在Pod中通过环境变量和Volume挂载使用。

### 核心答案

**ConfigMap创建方式**：

```bash
# 1. 键值对创建
kubectl create configmap app-config --from-literal=env=prod --from-literal=debug=false

# 2. 从文件创建
kubectl create configmap app-config --from-file=config.yaml

# 3. 从目录创建
kubectl create configmap app-config --from-file=./configs/

# 4. YAML创建
kubectl apply -f configmap.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  SPRING_PROFILES_ACTIVE: "prod"
  DATABASE_HOST: "mysql.default.svc.cluster.local"
  LOG_LEVEL: "INFO"
  application.yml: |
    spring:
      application:
        name: inventory-service
    server:
      port: 8080
```

**Secret创建方式**：

```bash
# 1. 键值对创建（Base64编码）
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=secret123

# 2. TLS Secret
kubectl create secret tls tls-cert \
  --cert=tls.crt --key=tls.key

# 3. YAML创建（Base64手动编码）
echo -n "admin" | base64
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=
  password: c2VjcmV0MTIz
```

**环境变量注入**：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: inventory-app
spec:
  containers:
  - name: app
    image: inventory-app:1.0
    env:
    - name: SPRING_PROFILES_ACTIVE
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: SPRING_PROFILES_ACTIVE
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    envFrom:
    - configMapRef:
        name: app-config
```

**Volume挂载**：

```yaml
spec:
  containers:
  - name: app
    image: inventory-app:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /config
      readOnly: true
    - name: secret-volume
      mountPath: /secrets
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: secret-volume
    secret:
      secretName: db-credentials
```

挂载后文件内容：
- ConfigMap：`/config/SPRING_PROFILES_ACTIVE` = "prod"
- Secret：`/secrets/username`、`/secrets/password`

**ConfigMap作为命令行参数**：
```yaml
env:
- name: ARGS
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: extra-args
command: ["/app"]
args: ["$(ARGS)"]
```

### 追问方向

1. ConfigMap修改后，Pod内的值会自动更新吗？哪种方式可以自动更新？
2. Secret的Base64编码是加密吗？实际生产中如何保证Secret安全？
3. Pod同时通过环境变量和Volume使用同一个ConfigMap，两者有什么区别？

### 避坑提示

- 环境变量注入的值是静态的，ConfigMap更新后环境变量不会变；Volume挂载可以通过TTL自动更新（默认约1分钟）
- Secret虽然是Base64编码，但etcd存储默认不加密，生产环境需要启用K8s加密或使用Vault

---

## 第17题：K8s持久化存储（PV/PVC/StorageClass，动态供给）

### 题目

K8s的持久化存储通过PV（PersistentVolume）、PVC（PersistentVolumeClaim）和StorageClass实现。请说明它们的关系，以及动态供给的工作流程。

### 核心答案

**核心概念**：

| 资源 | 作用 |
|------|------|
| **PV** | 集群层面的存储资源，类似"存储池" |
| **PVC** | Pod对存储的请求，类似"存储券" |
| **StorageClass** | 存储类型抽象，支持动态供给 |

**关系图**：
```
Pod → PVC → PV → 存储后端（NFS/云盘/ Ceph）
                ↑
         StorageClass（动态创建PV）
```

**PV示例**：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-inventory
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce      # 单节点读写
    # - ReadOnlyMany      # 多节点只读
    # - ReadWriteMany     # 多节点读写（需要支持）
  persistentVolumeReclaimPolicy: Retain  # Retain/Delete/Recycle
  storageClassName: standard
  nfs:
    server: nfs-server.default.svc.cluster.local
    path: /data/inventory
```

**PVC示例**：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-inventory
spec:
  resources:
    requests:
      storage: 5Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
```

**Pod使用PVC**：
```yaml
spec:
  containers:
  - name: app
    image: app:1.0
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-inventory
```

**StorageClass与动态供给**：
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-disk
provisioner: kubernetes.io/aws-ebs  # 云厂商供给器
parameters:
  type: gp3
  fsType: ext4
  replication-type: Regional
volumeBindingMode: WaitForFirstConsumer  # 延迟绑定
allowVolumeExpansion: true
```

动态供给流程：
1. 用户创建PVC，指定StorageClass
2. PVC触发StorageClass的provisioner
3. Provisioner调用云厂商API创建云盘（如AWS EBS）
4. 自动创建PV并绑定到PVC
5. Pod使用PVC

**项目中的存储需求**（从Dockerfile分析）：
```dockerfile
RUN mkdir -p /data/inventory-service-backend/
```
应用需要持久化存储：`/data`（业务数据）、`/sync/log`（日志）、`/archive/logs/inventory`（归档日志）

### 追问方向

1. `accessModes`中`ReadWriteOnce`和`ReadWriteMany`有什么区别？为什么NFS支持RWX而云盘不支持？
2. `persistentVolumeReclaimPolicy`的三种策略（Retain/Delete/Recycle）分别适用什么场景？
3. `volumeBindingMode: WaitForFirstConsumer`的作用是什么？为什么推荐使用？

### 避坑提示

- PVC不指定StorageClass时，使用集群默认StorageClass（如果没有默认则会Pending）
- 云盘（如AWS EBS）只能单节点挂载（ReadWriteOnce），无法跨节点共享

---

## 第18题：K8s调度（调度器工作流程、节点亲和性、污点容忍、硬亲和vs软亲和）

### 题目

Kubernetes Scheduler负责为Pod选择最优节点。请说明调度器的工作流程，以及如何通过节点亲和性（Node Affinity）、污点容忍（Taints/Tolerations）控制调度。

### 核心答案

**调度器工作流程**：

```
┌──────────────────────────────────────────────────────┐
│                 Pod创建请求                           │
└──────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────┐
│  1. Filtering（预选）：筛选出所有可调度Node            │
│     - 资源是否足够（CPU/内存）                         │
│     - 端口是否冲突                                    │
│     - 存储是否能满足                                  │
│     - 节点是否被标记不可调度                           │
└──────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────┐
│  2. Scoring（优选）：对可调度Node打分                  │
│     - 最低负载节点                                    │
│     - 资源分配最均衡                                  │
│     - Pod Affinity/Anti-Affinity                     │
│     - 标签选择器                                      │
└──────────────────────────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────┐
│  3. Binding：将Pod绑定到最优Node                      │
│     - 更新Pod.spec.nodeName                          │
└──────────────────────────────────────────────────────┘
```

**节点亲和性（Node Affinity）**：
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:  # 硬亲和（必须满足）
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      preferredDuringSchedulingIgnoredDuringExecution:  # 软亲和（尽量满足）
      - weight: 10
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - us-west-1a
```

**污点（Taints）和容忍（Tolerations）**：

Taints设置在Node上，表示"这个节点排斥某些Pod"：
```bash
# 标记节点有特殊硬件
kubectl taint nodes node1 special=true:NoSchedule

# 标记节点即将维护
kubectl taint nodes node2 dedicated=gpu:NoExecute
```

Tolerations设置在Pod上，表示"Pod能够容忍某些污点"：
```yaml
spec:
  tolerations:
  - key: "special"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"      # 容忍NoSchedule污点
  - key: "dedicated"
    operator: "Exists"
    effect: "NoExecute"       # 容忍NoExecute（包括不可调度+驱逐）
    tolerationSeconds: 3600   # 驱逐前等待3600秒
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300    # 容忍节点NotReady最多5分钟
```

**污点Effect类型**：

| Effect | 行为 |
|--------|------|
| `NoSchedule` | 不调度新Pod到该节点，不影响已运行Pod |
| `PreferNoSchedule` | 尽量不调度，无节点可用时仍可调度 |
| `NoExecute` | 不调度新Pod，且驱逐已运行Pod（可设置tolerationSeconds延迟） |

**Pod Anti-Affinity（同Zone分散部署）**：
```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - inventory-app
        topologyKey: topology.kubernetes.io/zone
```

### 追问方向

1. 节点亲和性和污点容忍什么关系？实际使用场景是什么？
2. 软亲和`preferredDuringScheduling`如果没有任何节点满足会怎样？
3. 如何实现"同一Deployment的Pod必须分布在不同节点"？

### 避坑提示

- 区分`nodeSelector`和`nodeAffinity`：前者是简单标签匹配，后者支持复杂表达式
- 污点容忍不满足时，Pod不会调度到该节点；但硬亲和不满足时，Pod直接Pending

---

## 第19题：K8s健康检查（livenessProbe/readinessProbe/exec/httpGet/tcpSocket）

### 题目

Kubernetes提供三种探针用于健康检查：livenessProbe、readinessProbe、startupProbe。请说明它们的作用、区别，以及如何配置exec、httpGet、tcpSocket检测方式。

### 核心答案

**探针类型**：

| 探针 | 作用 | 失败后果 |
|------|------|----------|
| **livenessProbe** | 检测容器是否存活（应用是否存活） | 杀死容器，重启 |
| **readinessProbe** | 检测容器是否就绪（能否接收流量） | 从Service endpoints移除 |
| **startupProbe** | 检测容器是否启动完成（慢启动容器） | startupProbe成功前，禁用liveness和readiness |

**配置方式**：

**exec方式**：
```yaml
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
```

**httpGet方式**：
```yaml
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
    httpHeaders:
    - name: X-Custom-Header
      value: value
  initialDelaySeconds: 30
  periodSeconds: 10
  successThreshold: 1
  failureThreshold: 3
```

**tcpSocket方式**：
```yaml
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

**通用参数**：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `initialDelaySeconds` | 0 | 启动后多久开始检测 |
| `periodSeconds` | 10 | 检测间隔 |
| `timeoutSeconds` | 1 | 检测超时时间 |
| `failureThreshold` | 3 | 连续失败多少次判定为失败 |
| `successThreshold` | 1 | 连续成功多少次判定为成功（readinessProbe专用） |

**典型配置示例**（Spring Boot应用）：
```yaml
containers:
- name: app
  image: inventory-app:1.0
  ports:
  - containerPort: 8080
  startupProbe:
    httpGet:
      path: /actuator/health/liveness
      port: 8080
    failureThreshold: 30
    periodSeconds: 10
  livenessProbe:
    httpGet:
      path: /actuator/health/liveness
      port: 8080
    initialDelaySeconds: 60
    periodSeconds: 10
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /actuator/health/readiness
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 5
    failureThreshold: 3
```

### 追问方向

1. `initialDelaySeconds`设置过短有什么风险？
2. `startupProbe`和`livenessProbe`同时存在时，调度逻辑是什么？
3. readinessProbe失败时，Pod会被驱逐吗？流量会中断吗？

### 避坑提示

- `initialDelaySeconds`必须大于应用启动时间，否则容器会在启动过程中被误杀
- Spring Boot的`/actuator/health`端点需要配置`management.endpoint.health.probes.enabled=true`

---

## 第20题：K8s资源限制（requests/limits，LimitRange，ResourceQuota）

### 题目

Kubernetes通过requests和limits控制容器资源使用。请说明它们的含义和区别，以及LimitRange和ResourceQuota的作用。

### 核心答案

**requests vs limits**：

| 维度 | requests | limits |
|------|----------|--------|
| 含义 | 调度时参考的"保证资源" | 运行时上限 |
| 调度影响 | 影响Pod调度（节点资源是否足够） | 影响运行时（超限被限流/OOM） |
| 超出后果 | 调度失败（Pod Pending） | CPU限流/内存OOM Kill |

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

**CPU理解**：
- `cpu: 1` = 1个CPU核心 = `1000m`（millicores）
- `cpu: 500m` = 0.5个核心
- CPU是可压缩资源，超限会被内核限流（throttled）

**内存理解**：
- `memory: 512Mi` = 512 MiB
- 内存是不可压缩资源，超限会被OOM Kill

**LimitRange**：
- 设置Namespace内Pod/容器的默认和最大资源限制
- 避免单个Pod耗尽整个集群

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - type: Container
    default:
      memory: 512Mi
      cpu: 500m
    defaultRequest:
      memory: 256Mi
      cpu: 250m
    max:
      memory: 2Gi
      cpu: 2
    min:
      memory: 128Mi
      cpu: 100m
  - type: Pod
    max:
      memory: 4Gi
      cpu: 4
```

**ResourceQuota**：
- 设置Namespace内资源总量上限
- 防止Namespace过度使用集群资源

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
spec:
  hard:
    requests.cpu: "10"
    requests.memory: "20Gi"
    limits.cpu: "20"
    limits.memory: "40Gi"
    pods: "100"
    services: "10"
```

### 追问方向

1. Pod设置了limits.memory=512Mi，容器实际使用了600Mi会怎样？
2. 如果容器只设置了limits没有设置requests，requests会如何取值？
3. 为什么建议将requests和limits设置为相同值（Guaranteed QoS）？

### 避坑提示

- 内存超限会导致OOM Kill，这是强制行为，不是限流
- CPU超限是软限制（throttling），容器不会被杀，只是变慢

---

## 第21题：K8s集群安全（RBAC权限、ServiceAccount、SecurityContext、NetworkPolicy）

### 题目

Kubernetes提供多层安全机制：RBAC、ServiceAccount、SecurityContext、NetworkPolicy。请说明各机制的作用，以及如何实现最小权限原则。

### 核心答案

**RBAC（Role-Based Access Control）**：

| 资源 | 作用域 |
|------|--------|
| Role/ClusterRole | 定义权限（verbs: get, list, create...） |
| RoleBinding/ClusterRoleBinding | 将权限绑定到Subject |

```yaml
# 定义只读权限
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# 将权限绑定到ServiceAccount
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
subjects:
- kind: ServiceAccount
  name: inventory-app
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**ServiceAccount**：
- Pod访问K8s API的身份
- 每个Pod都有一个ServiceAccount（默认是default）
- Pod通过挂载ServiceAccount的Token访问API

```yaml
spec:
  serviceAccountName: inventory-app-sa
  containers:
  - name: app
    image: inventory-app:1.0
```

**SecurityContext**：
- 设置容器级别或Pod级别的安全上下文
- 包括运行用户、组、权限、Capabilities等

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
  containers:
  - name: app
    image: inventory-app:1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

**NetworkPolicy**：
- 声明式控制Pod级别的网络流量
- 默认拒绝，声明允许的入站/出站流量

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
spec:
  podSelector:
    matchLabels:
      app: inventory-api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: mysql
    ports:
    - protocol: TCP
      port: 3306
```

**最小权限原则实践**：
1. 为每个应用创建专用ServiceAccount
2. 仅授予应用需要的RBAC权限
3. Pod使用非root用户运行
4. 通过NetworkPolicy限制服务间访问（只允许必要通信）

### 追问方向

1. Role和ClusterRole的区别？什么时候用哪个？
2. 如何通过ServiceAccount让Pod读取其他Namespace的Secret？
3. NetworkPolicy在没有CNI插件支持的情况下有效吗？

### 避坑提示

- RBAC权限默认是累积的（additive），没有deny机制
- NetworkPolicy需要CNI插件支持（如Calico），基础bridge网络不支持

---

## 第22题：K8s监控（Prometheus + Grafana，Metrics Server，HPA自动扩缩容）

### 题目

Kubernetes监控体系通常使用Prometheus + Grafana。请说明Metrics Server的作用，以及HPA（Horizontal Pod Autoscaler）如何基于指标实现自动扩缩容。

### 核心答案

**Metrics Server**：
- K8s核心监控组件，采集Pod和Node的资源指标（CPU/内存）
- 提供`kubectl top`命令查看资源使用
- 是HPA的指标数据来源

```bash
kubectl top nodes
kubectl top pods
```

**Prometheus + Grafana监控体系**：

| 组件 | 作用 |
|------|------|
| **Prometheus** | 指标采集、存储、查询（TSDB） |
| **Grafana** | 可视化仪表盘、告警 |
| **kube-state-metrics** | 采集K8s对象状态（Deployment/Service等） |
| **node-exporter** | 采集Node级别指标 |

**Prometheus架构**：
```
┌─────────────┐     ┌──────────────┐     ┌─────────┐
│  Targets    │────▶│  Prometheus  │────▶│ Grafana │
│  (Pod/Node) │     │  (Pull模型)  │     │         │
└─────────────┘     └──────────────┘     └─────────┘
```

**HPA（Horizontal Pod Autoscaler）**：
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: inventory-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: inventory-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
```

**HPA工作流程**：
1. Metrics Server采集Pod实际CPU/内存使用
2. HPA Controller计算：需要副本数 = ceil(当前副本数 × 当前CPU% ÷ 目标CPU%)
3. HPA更新Deployment的replicas字段
4. Deployment Controller创建/删除Pod

**自定义指标HPA（KEDA）**：
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: inventory-app-scaled
spec:
  scaleTargetRef:
    name: inventory-app
  triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: http_requests_per_second
      threshold: "100"
```

### 追问方向

1. Prometheus是Push还是Pull模型？如何发现监控目标？
2. HPA的`stabilizationWindowSeconds`的作用是什么？为什么需要？
3. 如何配置HPA在业务高峰期前预扩容？

### 避坑提示

- HPA需要Metrics Server提供指标，没有Metrics Server的话HPA无法工作
- CPU使用率计算基于requests，不是节点总资源

---

## 第23题：K8s日志（ELK/EFK日志方案，fluentd/fluent-bit日志收集）

### 题目

Kubernetes中容器日志如何收集和分析？请说明EFK（Elasticsearch + Fluentd + Kibana）日志方案，以及Fluent Bit相比Fluentd的优势。

### 核心答案

**容器日志生命周期**：

| 存储位置 | 说明 |
|----------|------|
| **容器stdout/stderr** | Kubelet将日志写入`/var/log/pods/<namespace>/<pod>/<container>/.log` |
| **Docker log driver** | json-file driver（默认），日志存储在`/var/lib/docker/containers/` |
| **应用程序文件** | 应用写入文件（如`/sync/log/app.log`） |

**EFK架构**：
```
┌────────────┐    ┌────────────┐    ┌───────────────┐    ┌─────────┐
│ Containers │───▶│  Fluentd   │───▶│ Elasticsearch │───▶│ Kibana  │
│ (日志输出)  │    │ (DaemonSet)│    │  (存储+索引)   │    │ (可视化)│
└────────────┘    └────────────┘    └───────────────┘    └─────────┘
```

**Fluentd/Fluent Bit部署为DaemonSet**：
- 每个Node运行一个Fluentd/Fluent Bit Pod
- 读取该Node上所有容器的日志
- 过滤、转换、发送到Elasticsearch

**Fluent Bit vs Fluentd**：

| 维度 | Fluent Bit | Fluentd |
|------|-----------|---------|
| 资源占用 | 轻量（~10MB内存） | 较重（~100MB内存） |
| 性能 | 高（用C编写） | 中（用Ruby编写） |
| 功能 | 日志收集、转发 | 丰富的插件、数据处理 |
| 适用场景 | 边缘节点、大规模集群 | 需要复杂处理的场景 |

**Fluent Bit配置示例**：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Log_Level     info
        Daemon        off

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        Refresh_Interval  5

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token

    [OUTPUT]
        Name            es
        Match           kube.*
        Host            elasticsearch.default.svc
        Port            9200
        Logstash_Format On
        Retry_Limit     False
```

**项目日志目录分析**（从Dockerfile）：
```dockerfile
RUN mkdir -p /sync/log && mkdir -p /data/inventory-service-backend/ && mkdir -p /archive/logs/inventory
```
- `/sync/log`：应用日志同步目录（EFK收集源）
- `/archive/logs/inventory`：归档日志

### 追问方向

1. 应用程序写入文件的日志如何通过EFK收集？需要什么配置？
2. 如何在Kibana中搜索特定Pod的日志？
3. 日志保留策略如何设置？Elasticsearch磁盘空间如何管理？

### 避坑提示

- EFK默认收集stdout/stderr，应用文件日志需要额外配置tail路径
- 日志量随集群规模增长，需要提前规划Elasticsearch存储容量

---

## 第24题：K8s网络（CNI插件Flannel/Calico/Canal，Pod网络模型）

### 题目

Kubernetes网络模型要求所有Pod可以跨节点互相通信。请说明CNI插件的作用，以及主流CNI插件（Flannel、Calico、Canal）的区别。

### 核心答案

**CNI（Container Network Interface）**：
- K8s网络标准接口，定义容器网络配置
- Kubelet通过CNI调用插件配置容器网络
- CNI插件负责：Pod IP分配、网络策略、跨节点通信

**K8s网络模型三大原则**：
1. 所有Pod可以跨Node互相通信（无需NAT）
2. 所有Node可以与所有Pod互相通信（无需NAT）
3. Pod看到的IP与其他人看到的IP一致

**主流CNI插件对比**：

| 插件 | 底层技术 | 网络模型 | 优势 | 劣势 |
|------|---------|----------|------|------|
| **Flannel** | VXLAN overlay | Overlay网络 | 简单、轻量、易部署 | 无网络策略支持 |
| **Calico** | BGP路由 | 纯三层路由 | 高性能、网络策略强大 | 配置复杂 |
| **Canal** | Flannel + Calico | Overlay + 网络策略 | 结合两者优点 | 组件较多 |

**Flannel工作原理**：
- 跨节点通信使用VXLAN封装（UDP封装二层帧）
- 每个Node有独立网段（如Node1: 10.244.0.0/24, Node2: 10.244.1.0/24）
- 跨Node通信：Pod → veth → docker0 → flannel.1（VXLAN）→ 对端flannel.1 → 目标Pod
- 性能损耗约10-15%

**Calico工作原理**：
- 直接使用三层路由（无需封装）
- 每个Node运行BIRD（BGP客户端），广播路由到其他Node
- 跨Node通信：Pod → veth → eth0 → 物理网卡 → 对端eth0 → 目标Pod
- 性能接近物理网络

**Calico网络策略示例**：
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}  # 空选择器，匹配所有Pod
  policyTypes:
  - Ingress
  - Egress
```

**IPAM（IP Address Management）**：
- CNI插件负责为每个Node分配Pod网段
- Flannel使用etcd协调分配
- Calico使用etcd或BGP RR（Route Reflector）同步

### 追问方向

1. 为什么K8s要求Pod IP全网可达？这与Docker bridge网络有何不同？
2. Calico的BGP模式适用于什么规模？小型集群是否需要BGP RR？
3. 如何排查Pod跨节点网络不通的问题？

### 避坑提示

- Flannel的VXLAN是overlay网络，实际跨Node流量会被封装，性能低于Calico
- Calico的NetworkPolicy需要设置`policyTypes`，否则默认不生效

---

## 第25题：K8s实战问题（Pod一直Pending/CrashLoopBackOff/ImagePullBackOff排查）

### 题目

请说明以下三种常见Pod异常状态的排查思路：Pod一直处于Pending状态、Pod处于CrashLoopBackOff状态、Pod处于ImagePullBackOff状态。

### 核心答案

**1. Pod一直处于Pending状态**

**排查步骤**：

```bash
# 1. 查看Pod详情
kubectl describe pod <pod-name>

# 2. 查看Events
kubectl describe pod <pod-name> | grep -A 10 "Events:"
```

**常见原因及解决方案**：

| 原因 | 现象（Events） | 解决方案 |
|------|---------------|----------|
| 资源不足 | `FailedScheduling: Insufficient cpu/memory` | 减少其他Pod资源/扩容节点/降低requests |
| 无可用节点 | `FailedScheduling: No nodes available` | 检查节点状态`kubectl get nodes` |
| 亲和性不满足 | `FailedScheduling: 0/N nodes match pod affinity rules` | 调整affinity规则 |
| 污点不容忍 | `FailedScheduling: node(s) had taints` | 添加tolerations |
| PVC未绑定 | `FailedScheduling: persistentvolumeclaim not bound` | 检查PVC状态 |
| 调度被禁用 | `FailedScheduling: nodes are not schedulable` | `kubectl uncordon node` |

```bash
# 检查节点污点
kubectl get nodes -o json | jq '.items[].spec.taints'

# 检查PVC绑定状态
kubectl get pvc
kubectl describe pvc <pvc-name>
```

**2. Pod处于CrashLoopBackOff状态**

**排查步骤**：

```bash
# 1. 查看容器日志
kubectl logs <pod-name> --previous

# 2. 查看Pod详情
kubectl describe pod <pod-name>

# 3. 进入容器调试（如容器未完全崩溃）
kubectl exec -it <pod-name> -- /bin/sh
```

**常见原因及解决方案**：

| 原因 | 日志/现象 | 解决方案 |
|------|----------|----------|
| 应用启动失败 | `Error: java.net.UnknownHostException` | 检查应用配置、依赖服务 |
| 健康检查失败 | `Liveness probe failed` | 调整initialDelaySeconds |
| OOMKilled | `Last Exit State: OOMKilled` | 增加内存limits |
| 端口冲突 | `java.net.BindException: Address in use` | 修改containerPort |
| 配置错误 | `FileNotFoundException` | 检查配置文件挂载 |

```bash
# 检查OOMKill（需要开启audit）
dmesg | grep -i "killed process"
kubectl top pod <pod-name> --containers
```

**3. Pod处于ImagePullBackOff状态**

**排查步骤**：

```bash
# 1. 查看详情
kubectl describe pod <pod-name>

# 2. 检查镜像名称
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[0].image}'
```

**常见原因及解决方案**：

| 原因 | Events | 解决方案 |
|------|--------|----------|
| 镜像名称错误 | `ErrImagePull` | 修正镜像名称 |
| 镜像不存在 | `ErrImageNotFound` | 确认镜像仓库中是否存在 |
| 认证失败 | `ImagePullBackOff: unauthorized` | 创建imagePullSecret |
| 网络问题 | `ImagePullBackOff: dial tcp` | 配置代理/检查网络策略 |
| 仓库访问限制 | `ImagePullBackOff: denied` | 配置imagePullSecrets |

```bash
# 创建Secret保存镜像仓库认证信息
kubectl create secret docker-registry my-registry-key \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<password> \
  --docker-email=<email>

# Pod引用Secret
spec:
  imagePullSecrets:
  - name: my-registry-key
```

**通用排查流程图**：

```
Pod异常
    │
    ├─ Pending → kubectl describe pod → 检查Events
    │             ├─ 资源不足 → 扩容/降低requests
    │             ├─ 调度失败 → 检查node/taint/affinity
    │             └─ PVC未绑定 → 检查存储供给
    │
    ├─ CrashLoopBackOff → kubectl logs --previous
    │                      ├─ 应用错误 → 修复应用配置
    │                      ├─ OOM → 增加内存
    │                      └─ 健康检查 → 调整probe
    │
    └─ ImagePullBackOff → kubectl describe pod
                         ├─ 镜像名称 → 确认仓库路径
                         ├─ 认证失败 → 创建imagePullSecret
                         └─ 网络不通 → 检查网络策略/DNS
```

### 追问方向

1. Pod处于`Terminating`状态无法删除，如何处理？
2. `kubectl logs`显示容器正常启动，但Pod仍CrashLoopBackOff，为什么？
3. 如何在Pod启动前预检查（preflight check）避免部署失败？

### 避坑提示

- `kubectl describe pod`显示的Events是排查的第一步，很多候选人跳过这步直接猜原因
- CrashLoopBackOff的`--previous`参数很重要，查看的是上一次运行（崩溃时）的日志

---

## 附录：WMS项目Dockerfile实战分析

基于项目实际代码（`/usr/src/inventory-service-backend/deploy/prod/aws/Dockerfile`）：

```dockerfile
FROM 571600829038.dkr.ecr.us-west-2.amazonaws.com/prod-item/app-baseimage:v7.0-alpha
RUN mkdir -p /sync/log && mkdir -p /data/inventory-service-backend/ && mkdir -p /archive/logs/inventory
COPY inventory-app/build/libs/inventory-app-0.0.1-PROD-SNAPSHOT.jar /data/inventory-service-backend/inventory-app-0.0.1-SNAPSHOT.jar
COPY deploy/prod/aws/supervisord.conf /etc/supervisor/supervisord.conf
CMD ["/usr/bin/supervisord","-c","/etc/supervisor/supervisord.conf"]
```

**优化建议**：
1. **RUN层可合并**：3个`mkdir`可以合并到1个RUN
2. **构建缓存优化**：JAR包构建应在COPY前先COPY pom.xml/gradle依赖文件
3. **无多阶段构建**：建议使用多阶段构建，将编译阶段移到镜像内
4. **日志目录**：建议通过Volume挂载持久化日志目录
5. **无健康检查**：建议添加HEALTHCHECK指令

---

> 本文档涵盖Docker与Kubernetes核心知识点，建议配合实际操作加深理解。
> 项目代码路径：`/usr/src/inventory-service-backend/deploy/`
