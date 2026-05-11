# CI/CD与DevOps 面试题（第12部分）

> 本系列覆盖 Java 后端开发核心技术栈，结合 WMS 仓库管理系统（wms-backend / inventory-service-backend / item-master-backend / xms-starter）真实项目代码。
> 项目使用 Gradle 构建（见 `inventory-service-backend/build.gradle`），多环境配置（dev/stage/prod），Nacos 配置中心，Spring Boot 3.x，Oracle/MySQL 双数据库驱动。

---

## 目录

1. [CI/CD概念：持续集成/持续交付/持续部署区别，完整流程](#q1)
2. [主流CI/CD工具对比：Jenkins/GitLab CI/GitHub Actions/CircleCI/ArgoCD](#q2)
3. [Jenkins核心概念：Master/Slave架构、节点管理、Agent配置](#q3)
4. [Jenkins Pipeline：声明式vs脚本式Pipeline，Jenkinsfile语法](#q4)
5. [Jenkinsfile完整示例：stages/steps/agent/environment/post](#q5)
6. [Jenkins常用插件：Git/Maven/Docker/Kubernetes/Parameterized Trigger](#q6)
7. [流水线构建全流程：代码拉取/编译/测试/打包/部署，SonarQube质量门禁](#q7)
8. [GitLab CI：.gitlab-ci.yml语法，stages/jobs/image/services/dependencies](#q8)
9. [GitLab Runner：注册方式，executor类型](#q9)
10. [GitHub Actions：workflow语法，on/triggers/jobs/steps，act本地运行](#q10)
11. [环境管理：dev/test/staging/prod环境分离，配置中心管理](#q11)
12. [Docker镜像构建：Maven/Gradle多阶段构建，构建优化](#q12)
13. [Kubernetes部署：Helm模板、kubectl apply、服务滚动更新](#q13)
14. [配置加密：SOPS/Sealed Secrets/KSOPS，敏感信息管理](#q14)
15. [制品管理：Harbor/Nexus/JFrog Artifactory，镜像签名/扫描](#q15)
16. [蓝绿部署：原理、流量切换、回滚策略](#q16)
17. [金丝雀发布：Canary Release，Istio/Argo Rollouts灰度策略](#q17)
18. [特性开关：Feature Toggle，Unleash/LaunchDarkly，代码解耦](#q18)
19. [监控告警：Prometheus+Grafana+AlertManager，告警收敛、静默期](#q19)
20. [DevOps文化：DevOps三步工作法，团队协作，测量（MTTR/MTBF/CI频率）](#q20)

---

<a id="q1"></a>

## Q1: CI/CD概念——持续集成/持续交付/持续部署的区别？完整流程是什么？

### 核心答案

**持续集成（CI — Continuous Integration）**
- 开发者频繁提交代码（建议每天多次），每次提交自动触发**构建+单元测试**
- 目标：尽早发现集成错误，避免"集成地狱"
- 核心实践：自动化构建、统一代码库、测试自动化、快速构建

**持续交付（CD — Continuous Delivery）**
- CI 通过后，代码自动部署到 **staging 环境**，经过预生产验证后手动批准上线
- 目标：代码始终处于**可部署状态**，但上线需要人工决策
- 在 staging 环境完成：集成测试、端到端测试、UAT

**持续部署（CD — Continuous Deployment）**
- CI 通过后，代码**自动部署到生产环境**，无需人工干预
- 前提：质量门禁（SonarQube、自动化测试）必须近乎完美

**三者关系**
```
CI  →  持续交付  →  持续部署
↑         ↑            ↑
代码集成  自动到staging  自动到prod
```

**完整流程（以 WMS 项目为例）**
```
开发者提交PR
    ↓
GitLab CI / GitHub Actions 触发 CI Pipeline
    ↓
1. 代码拉取（git clone）
2. 依赖安装（npm install / ./gradlew dependencies）
3. 编译构建（npm run build:prod / ./gradlew build）
4. 单元测试（./gradlew test）
5. 静态代码分析（SonarQube）
    ↓ 质量门禁通过
6. 镜像构建（docker build）
7. 推送镜像到 Harbor
    ↓
8. 部署到 DEV 环境（自动）
9. 部署到 STAGING 环境（自动）
10. 部署到 PROD 环境
   ├── 蓝绿部署 或 金丝雀发布
   └── 监控 + 告警
```

### 追问方向
- 你们的 CI 流程多久跑一次？平均构建时长是多少？
- 如何保证 CI 环境的**一致性**（避免"在我机器上能跑"）？
- 质量门禁的具体指标是什么（代码覆盖率、SonarQube 评分阈值）？

### 避坑提示
- 不要混淆"持续交付"和"持续部署"——前者需要人工批准，后者完全自动
- 如果构建时间超过 10 分钟，CI 的反馈价值会大幅降低，需要优化（并行测试、缓存依赖）
- 很多团队只做了 CI（自动构建+测试），但没有做到 CD（自动化部署到 staging）

---

<a id="q2"></a>

## Q2: 主流CI/CD工具对比——Jenkins / GitLab CI / GitHub Actions / CircleCI / ArgoCD

### 核心答案

| 维度 | Jenkins | GitLab CI | GitHub Actions | CircleCI | ArgoCD |
|------|---------|-----------|----------------|----------|--------|
| **定位** | 通用自动化服务器 | GitLab 内置 CI/CD | GitHub 内置 CI/CD | 云原生 CI/CD | Kubernetes 原生 CD |
| **配置方式** | Jenkinsfile + UI | `.gitlab-ci.yml` | `workflows/*.yml` | `circleci/config.yml` | GitOps YAML |
| **安装维护** | 自托管，运维成本高 | 自托管或 SaaS | SaaS（也可自托管） | SaaS 或自托管 | Kubernetes 集群内 |
| **插件生态** | 5000+ 插件，极度丰富 | 内置 + 市场 | Marketplace 丰富 | Orb 市场 | 扩展 CRD |
| **学习曲线** | 陡峭（Jenkinsfile DSL） | 平缓 | 平缓（YAML直观） | 平缓 | 需要懂 GitOps |
| **UI 体验** | 传统，界面老旧 | 现代，集成良好 | 现代，GitHub 深度集成 | 现代 | K8s 风格 Dashboard |
| **执行环境** | Master/Slave 节点 | GitLab Runner | GitHub-hosted / 自托管 Runner | 容器化执行器 | Kubernetes Pod |
| **价格** | 开源免费 | 免费版有限制，企业版收费 | 免费额度，超出收费 | 免费额度有限 | 开源免费，企业版收费 |
| **适用场景** | 企业复杂构建，多工具集成 | 与 GitLab 代码库深度集成 | GitHub 项目，快速上手 | 快速云原生 CI | Kubernetes 持续部署 |

**WMS 项目实际用法**
- `wms-web` 使用 Jenkinsfile（见 `wms-web/jenkinsfile/JenkinsfileStage`），调用自定义 plumber 工具完成镜像构建和 K8s 部署
- `inventory-service-backend` 使用 Gradle 多阶段构建，制品发布到阿里云 Maven 仓库

### 追问方向
- 你们为什么选择 Jenkins 而不是 GitLab CI/GitHub Actions？
- 如何实现 Jenkins 的高可用（Master 主备）？
- ArgoCD 的 GitOps 模式和传统 Push 模式部署有什么区别？

### 避坑提示
- 不要说"Jenkins 什么都好"——它的插件安全性和维护成本是真实痛点
- GitHub Actions 很方便，但如果代码在 GitLab 上就不适用
- ArgoCD 是 CD 工具，不是 CI 工具，需要配合 CI 系统使用

---

<a id="q3"></a>

## Q3: Jenkins核心概念——Master/Slave架构、节点管理、Agent配置

### 核心答案

**Master 节点**
- Jenkins 主服务，负责：调度构建任务、提供 Web UI、管理配置、存储构建历史
- 协调各 Agent 的工作负载，本身不执行具体构建
- 关键配置：`Manage Jenkins` → `Manage Nodes` → `New Node`

**Agent 节点**
- 执行具体构建任务的 Worker
- 可以是物理机、虚拟机或 Docker 容器
- 通过 JNLP（Java Web Start）或 SSH 连接到 Master
- 支持标签（Label）机制，用于定向调度

**架构图**
```
[Developer]
     ↓ git push
[Jenkins Master]  ←  Web UI / API / 调度
     ↓
[Agent 1]          [Agent 2]          [Agent 3]
Docker Build        Maven Build        kubectl Deploy
```

**节点配置示例（管理节点）**
- **Permanent Agent**：长期运行的物理机/虚拟机
- **Docker Agent**：按需启动的容器，执行完销毁
- **Kubernetes Agent**：Jenkins Kubernetes 插件，在 K8s Pod 中运行构建（推荐生产使用）

**Agent 配置关键参数**
- `远程工作目录`（Remote root directory）：如 `/home/jenkins/agent`
- `启动方式`：JNLP / SSH / 容器
- `标签`：如 `docker`, `maven`, `kubernetes`，用于 job 定向
- `可用性`：按需启动 vs 保持在线

### 追问方向
- 如何配置 Jenkins 的 Kubernetes Agent？（需要安装 `kubernetes` 插件）
- Jenkins Agent 失联（offline）怎么排查？
- 如何控制并发构建数量，避免资源争抢？

### 避坑提示
- 不要在 Master 上直接运行构建（消耗 Master 资源，影响稳定性）
- Agent 的工作目录必须预先创建且有正确权限
- JNLP 启动方式需要 Master 可访问 Agent 的端口，安全组需要规划

---

<a id="q4"></a>

## Q4: Jenkins Pipeline——声明式 vs 脚本式 Pipeline，Jenkinsfile 语法

### 核心答案

**声明式 Pipeline（Declarative Pipeline）**
- 结构化语法，更易读，适合大多数场景
- 以 `pipeline` 块为根，所有内容用 `{}` 包裹
- 关键指令：`agent`, `stages`, `steps`, `environment`, `post`, `when`

**脚本式 Pipeline（Scripted Pipeline）**
- 基于 Groovy DSL，更灵活强大
- 以 `node` 块为根，内部是 Groovy 代码
- 适合复杂逻辑（循环、条件分支）

**对比**
| 维度 | 声明式 | 脚本式 |
|------|--------|--------|
| 可读性 | 高（结构化） | 较低（Groovy 代码） |
| 学习门槛 | 低 | 较高 |
| 灵活性 | 有限制 | 几乎无限制 |
| 调试 | 较难 | Groovy 调试能力强 |
| 建议场景 | 90% 的标准构建 | 复杂条件逻辑 |

**声明式 Pipeline 示例结构**
```groovy
pipeline {
    agent { label 'maven' }          // 或 agent any / agent none
    
    environment {
        MAVEN_OPTS = '-Xmx512m'
    }
    
    options {
        timeout(time: 1, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    
    stages {
        stage('Build') {
            steps {
                sh './gradlew build'
            }
        }
        stage('Test') {
            steps {
                sh './gradlew test'
            }
        }
    }
    
    post {
        always {
            junit 'build/test-results/**/*.xml'
        }
        failure {
            emailext body: 'Build failed', to: 'team@example.com'
        }
    }
}
```

**脚本式 Pipeline 示例**
```groovy
node('maven') {
    stage('Build') {
        def mvn = tool 'Maven-3.9'
        sh "${mvn}/bin/mvn clean package"
    }
    
    if (env.BRANCH_NAME == 'main') {
        stage('Deploy') {
            sh 'kubectl apply -f k8s/'
        }
    }
}
```

### 追问方向
- `agent any` 和 `agent none` 的区别是什么？
- `when` 指令如何实现条件执行（如只在 `main` 分支执行部署）？
- 如何在 Pipeline 中使用 `credentials()` 读取密文？

### 避坑提示
- 声明式 Pipeline 中不支持所有 Groovy 语法，需要用到复杂逻辑时用 `script` 块包裹
- `environment` 变量在 steps 中可以直接引用，但在 sh 中使用 `${VAR}` 格式
- 不要忘记 `post` 块，可以做清理、通知、归档等收尾工作

---

<a id="q5"></a>

## Q5: Jenkinsfile 完整示例——stages / steps / agent / environment / post

### 核心答案

参考 WMS 项目 `wms-web/jenkinsfile/JenkinsfileStage`：

```groovy
pipeline {
  // 1. Agent：执行节点
  agent any
  
  // 2. 环境变量
  environment {
      NODE_ENV = 'development'
      imageTag = ''  // 会在后续steps中重新赋值
  }
  
  // 3. 选项配置
  options {
      timeout(time: 30, unit: 'MINUTES')
      disableConcurrentBuilds()
      buildDiscarder(logRotator(numToKeepStr: '10'))
  }
  
  // 4. 参数化构建（可选）
  parameters {
      string(name: 'DEPLOY_ENV', defaultValue: 'stage', description: '部署环境')
      booleanParam(name: 'SKIP_SONAR', defaultValue: false, description: '跳过SonarQube')
  }
  
  // 5. Stages：阶段定义
  stages {
      stage('Checkout') {
          steps {
              script {
                  echo "构建版本: ${params.DEPLOY_ENV}"
              }
              checkout scm
          }
      }
      
      stage('Package') {
          agent { docker {
              image 'node:18.16'
              args '-u root:root'
              reuseNode true       // 复用同一节点，保留工作目录
          } }
          steps {
              sh 'npm --version'
              sh 'npm cache clean --force'
              sh 'npm install'
              sh 'npm run build:stage'
          }
      }
      
      stage('Build Image & Push') {
          environment {
              // 注意：这里的赋值方式与environment block不同
              imageTag = sh(script: 'openssl rand -base64 3 | md5sum | cut -c1-30', returnStdout: true).trim()
          }
          steps {
              sh """
                /data/plumber/plumber \
                  -dd=/data/plumber/wms-web_stage \
                  -step=Build \
                  -pt=Web \
                  -pn=wms \
                  -pe=stage_v2 \
                  -ad=${WORKSPACE} \
                  -itag=${imageTag}
              """
          }
      }
      
      stage('Deploy to K8s') {
          steps {
              sh """
                /data/plumber/plumber \
                  -dd=/data/plumber/wms-web_stage \
                  -kubeConfig=/var/lib/jenkins/.awskube/config \
                  -step=Deploy \
                  -pt=Generic \
                  -k8sYml=open \
                  -pn=wms \
                  -appName=wms \
                  -appImageRepo=stage.logisticsteam.com:5000/stage_v2-wms-web \
                  -itag=${imageTag}
              """
          }
      }
  }
  
  // 6. Post：构建后处理
  post {
      always {
          echo "清理构建环境..."
          cleanWs()
      }
      success {
          echo "构建成功，镜像标签: ${imageTag}"
          // 通知企业微信/钉钉
      }
      failure {
          echo "构建失败！"
          emailext (
              subject: "Jenkins 构建失败: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
              body: "构建日志: ${env.BUILD_URL}",
              to: 'devops@example.com'
          )
      }
      unstable {
          echo "构建不稳定（测试失败）"
      }
  }
}
```

**关键指令说明**

| 指令 | 作用 | 示例 |
|------|------|------|
| `agent` | 指定执行节点 | `agent any` / `agent { label 'docker' }` / `agent { docker { image 'node:18' } }` |
| `environment` | 定义环境变量 | `imageTag = sh(...)` 或 `VAR = 'value'` |
| `stages` / `stage` | 构建阶段 | 逻辑分组，如 Build / Test / Deploy |
| `steps` | 具体执行步骤 | `sh` / `bat` / `echo` / `withCredentials` |
| `post` | 后置处理 | `always` / `success` / `failure` / `unstable` |
| `options` | Pipeline 配置 | 超时、并发、日志保留 |
| `parameters` | 参数化构建 | 让用户输入构建参数 |

### 追问方向
- `reuseNode true` 什么时候用？和 `reuseNode false` 有什么区别？
- 如何在 `post` 块中使用 `input` 暂停 Pipeline 等待人工审批？
- 如何传递敏感的 Docker registry 账号密码给 Pipeline？

### 避坑提示
- `environment` block 中使用 `sh` 赋值时，变量只在当前 stage 生效，需在 Pipeline 顶层定义
- `cleanWs()` 会清理工作区，确保 `archiveArtifacts` 等步骤在 cleanWs 之前
- Docker agent 的 `reuseNode true` 可以复用节点缓存依赖（如 `node_modules`），加快构建速度

---

<a id="q6"></a>

## Q6: Jenkins常用插件——Git / Maven / Docker / Kubernetes / Parameterized Trigger

### 核心答案

**1. Git 插件（git）**
- 核心功能：代码克隆、分支管理、Tag 处理
- 常用配置：`checkout scm`（根据 Jenkinsfile 所在仓库自动克隆）
- 支持：Submodules、Git LFS、SSH 认证、Git hooks

**2. Maven 插件（Maven Integration）**
- 自动识别项目中的 `pom.xml`，提供 "构建一个 Maven 项目" 模板
- 配置 Maven 构建步骤、运行时参数（`-Xmx`）、私有仓库认证
- 与 SonarQube 插件配合做代码质量分析

**3. Docker 插件**
- `docker-workflow`：在 Pipeline 中使用 Docker
- `docker-build-step`：构建、推送镜像
- 常用语法：
  ```groovy
  agent { docker { image 'maven:3.9', reuseNode true } }
  ```

**4. Kubernetes 插件（Kubernetes）**
- 在 Jenkins Master 所在 K8s 集群中动态创建销毁 Pod 作为 Agent
- 配置 Kubernetes Cloud：API 地址、命名空间、Pod 模板（镜像、CPU/内存限制）
- 大规模 Jenkins 集群推荐方案，按需扩展

**5. Parameterized Trigger 插件**
- 允许一个 Pipeline 触发另一个 Job，并传递参数
- 场景：主 Pipeline 构建镜像 → 触发部署 Pipeline
  ```groovy
  build job: 'deploy-to-staging', parameters: [
      string(name: 'IMAGE_TAG', value: "${imageTag}"),
      string(name: 'ENV', value: 'staging')
  ]
  ```

**6. 其他高频插件**
| 插件 | 用途 |
|------|------|
| `Pipeline` | Pipeline 核心支持 |
| `Credentials Binding` | 管理密码、SSH 密钥、证书 |
| `SSH Agent` | SSH 远程执行命令 |
| `SonarQube Scanner` | 代码质量扫描 |
| `JUnit` | 测试结果聚合展示 |
| `Email Extension` | 增强邮件通知 |
| `Slack Notification` | Slack 通知 |
| `Config File Provider` | 管理 Maven/NPM 配置文件 |

### 追问方向
- 如何在 Kubernetes 插件中配置 Pod 模板的持久化卷（PVC）？
- 如何通过插件管理 Docker-in-Docker（DinD）构建？
- Git 插件的 `checkout scm` 和手动 `git clone` 有什么区别？

### 避坑提示
- Jenkins 插件升级可能导致兼容性问题，生产环境升级前先在测试环境验证
- 插件数量过多会影响 Jenkins 启动速度，只安装必要的
- `docker-workflow` 插件的 `reuseNode true` 参数很关键，不加会导致 Docker 找不到构建产物

---

<a id="q7"></a>

## Q7: 流水线构建全流程——代码拉取 / 编译 / 测试 / 打包 / 部署，SonarQube 质量门禁

### 核心答案

**完整流水线（以 inventory-service-backend 为例）**

```groovy
pipeline {
    agent { label 'maven' }
    
    environment {
        MAVEN_OPTS = '-Xmx1024m'
        SONAR_TOKEN = credentials('sonar-token')
    }
    
    stages {
        // 阶段1：代码拉取
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        // 阶段2：依赖解析（可与Checkout合并）
        stage('Dependencies') {
            steps {
                sh './gradlew dependencies --refresh-dependencies'
            }
        }
        
        // 阶段3：编译
        stage('Compile') {
            steps {
                sh './gradlew compileJava -x test'
            }
        }
        
        // 阶段4：单元测试
        stage('Unit Test') {
            steps {
                sh './gradlew test'
            }
            post {
                always {
                    junit '**/build/test-results/**/*.xml'
                }
            }
        }
        
        // 阶段5：代码质量扫描（SonarQube）
        stage('SonarQube Analysis') {
            steps {
                sh '''
                    ./gradlew sonar \
                        -Dsonar.host.url=http://sonarqube:9000 \
                        -Dsonar.token=${SONAR_TOKEN} \
                        -Dsonar.projectKey=inventory-service \
                        -Dsonar.java.binaries=build/classes
                '''
            }
        }
        
        // 阶段6：安全扫描（可选）
        stage('Security Scan') {
            steps {
                sh './gradlew dependencyCheckAnalyze'
            }
        }
        
        // 阶段7：构建镜像
        stage('Build Docker Image') {
            steps {
                sh '''
                    ORG=stage.logisticsteam.com:5000
                    APP=inventory-service
                    VERSION=${GIT_COMMIT:0-8}
                    docker build -t $ORG/$APP:$VERSION \
                        -f deploy/stage/aws/Dockerfile .
                    docker push $ORG/$APP:$VERSION
                '''
            }
        }
        
        // 阶段8：部署到 STAGING
        stage('Deploy to Staging') {
            when { branch 'main' }
            steps {
                sh './gradlew k8sDeploy -Penv=staging'
            }
        }
    }
    
    post {
        failure {
            emailext subject: "构建失败: ${env.JOB_NAME}", body: "${env.BUILD_URL}", to: 'devops@item.com'
        }
    }
}
```

**SonarQube 质量门禁（Quality Gate）**

质量门禁是 SonarQube 的通过/失败规则，通常检查：

| 指标 | 阈值 | 说明 |
|------|------|------|
| 代码覆盖率 | ≥ 80% | JaCoCo 覆盖率报告 |
| 严重程度为 Bug | 0 | 不允许新增 Bug |
| 漏洞（Vulnerabilities） | 0 | 不允许新增安全漏洞 |
| 代码异味（Code Smell） | 等级 B 或以上 | 可接受轻微问题 |
| 重复率 | ≤ 3% | 低代码重复 |
| 安全性热点 | 无 | 无安全热点 |

**质量门禁在 Pipeline 中强制执行**
```groovy
stage('Quality Gate') {
    steps {
        waitForQualityGate abortPipeline: true
    }
}
```

### 追问方向
- 如何在 CI Pipeline 中并行执行测试以缩短构建时间？
- SonarQube 扫描很慢（10分钟+），怎么优化？
- 如何处理测试覆盖率不达标但必须上线的紧急情况？

### 避坑提示
- `./gradlew dependencies` 的 `--refresh-dependencies` 会强制刷新缓存，每次构建都加会导致速度极慢，生产环境慎用
- SonarQube 质量门禁设置要合理，过于严格会导致团队绕过（关掉门禁）
- Docker 构建最好在专用的 Docker Agent 节点或 Kaniko 中执行，避免权限问题

---

<a id="q8"></a>

## Q8: GitLab CI——.gitlab-ci.yml 语法，stages / jobs / image / services / dependencies

### 核心答案

**基础结构**
```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - deploy

variables:
  MAVEN_OPTS: "-Xmx512m"
  DOCKER_DRIVER: overlay2

# Job 模板复用
.base_job: &base_job
  image: maven:3.9-eclipse-temurin-21
  before_script:
    - chmod +x mvnw

# Job 定义
maven-build:
  <<: *base_job              # 继承模板
  stage: build
  script:
    - ./mvnw clean package -DskipTests
  artifacts:
    paths:
      - target/*.jar
    expire_in: 1 day
  only:
    - main
    - develop

gradle-build:
  stage: build
  image: gradle:8.5-jdk21
  script:
    - gradle clean build -x test
  artifacts:
    paths:
      - build/libs/*.jar
    expire_in: 1 day

unit-test:
  stage: test
  image: gradle:8.5-jdk21
  script:
    - gradle test
  coverage: '/Total.*? (\\d+\\.\\d+)%/'
  artifacts:
    reports:
      junit: build/test-results/**/*.xml
      coverage_report:
        coverage_format: cobertura
        path: build/reports/cobertura/coverage.xml

# Docker 构建 Job
docker-build:
  stage: build
  image: docker:24-dind
  services:
    - docker:24-dind
  script:
    - docker build -t registry.example.com/app:$CI_COMMIT_SHORT_SHA .
    - docker push registry.example.com/app:$CI_COMMIT_SHORT_SHA

# 部署 Job
deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context staging
    - kubectl apply -f k8s/
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

deploy-prod:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context prod
    - kubectl apply -f k8s/
  environment:
    name: production
    url: https://prod.example.com
  when: manual        # 手动触发
  only:
    - main
```

**关键语法解释**

| 关键字 | 作用 | 示例 |
|--------|------|------|
| `stages` | 定义全局流水线阶段，按顺序执行 | `stages: [build, test, deploy]` |
| `stage` | Job 所属阶段 | `stage: build` |
| `image` | 容器镜像 | `image: maven:3.9` |
| `services` | 关联服务容器（如 DinD） | `- docker:24-dind` |
| `script` | 执行的命令 | `script: ["./gradlew build"]` |
| `before_script` | 每个 Job 执行前的脚本 | 初始化环境 |
| `after_script` | Job 执行后的脚本 | 清理 |
| `artifacts` | 构建产物传递 | 供下游 Job 使用 |
| `dependencies` | 依赖的上游 Job（下载其 artifacts） | `dependencies: [build]` |
| `only` / `except` | 分支策略 | `only: [main, develop]` |
| `when` | 触发条件 | `when: manual`（手动）/`when: on_failure` |
| `variables` | CI/CD 变量 | 加密/环境配置 |
| `coverage` | 代码覆盖率正则提取 | 解析测试输出 |
| `extends` | 继承另一个 Job | `extends: .base_job` |

**artifacts 与 dependencies 的区别**
- `artifacts`：定义当前 Job 向下游传递的产物
- `dependencies`：声明当前 Job 需要从哪些上游 Job 获取 artifacts（必须在当前 stage 之前或同 stage）

### 追问方向
- GitLab CI 如何实现 Job 之间的并行执行？
- `cache` 和 `artifacts` 的区别是什么？各自适用什么场景？
- 如何使用 `rules` 表达式实现更复杂的触发条件？

### 避坑提示
- `before_script` 在每个 Job 都会执行，如果 Job 很多会有重复开销，可以提取到 `extends` 模板中
- `artifacts` 默认在所有 stage 结束后清理，不会自动传递给后续 stage 的 Job，需要用 `dependencies`
- `docker:24-dind` 服务需要开启 GitLab Runner 的 Docker-in-Docker 特权模式，安全风险需评估

---

<a id="q9"></a>

## Q9: GitLab Runner——注册方式，executor 类型

### 核心答案

**GitLab Runner 架构**
```
[GitLab Server] ← 注册 → [GitLab Runner] ← 执行 → [具体 Executor]
```

**注册命令**
```bash
gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.example.com" \
  --registration-token "xxxxx" \
  --executor "docker" \
  --docker-image "docker:24-dind" \
  --docker-privileged \
  --description "docker-runner" \
  --tag-list "docker,maven,k8s" \
  --locked="false"
```

**Executor 类型对比**

| Executor | 特点 | 适用场景 | 隔离性 |
|----------|------|----------|--------|
| **Shell** | 直接在 Runner 主机执行 | 简单脚本，测试环境 | 差（共享主机） |
| **Docker** | 容器中执行，镜像隔离 | 主流选择，依赖一致 | 中等 |
| **Docker-Machine** | 动态创建销毁 Docker 主机 | 大规模并发构建 | 好 |
| **Kubernetes** | 在 K8s Pod 中执行 | K8s 原生环境 | 好 |
| **VirtualBox** | 虚拟机中执行 | 跨平台测试 | 好（但资源重） |
| **SSH** | 通过 SSH 连接远程主机执行 | 连接特定物理机 | 取决于远程主机 |

**Kubernetes Executor 配置**
```bash
gitlab-runner register \
  --non-interactive \
  --url "https://gitlab.example.com" \
  --registration-token "K8s_TOKEN" \
  --executor "kubernetes" \
  --kubernetes-image "ubuntu:22.04" \
  --kubernetes-namespace "gitlab-runner" \
  --kubernetes-cpu-limit "2" \
  --kubernetes-memory-limit "4Gi" \
  --kubernetes-service-cpu-limit "1" \
  --kubernetes-service-memory-limit "2Gi"
```

**Runner 生命周期管理**
```bash
# 查看 Runner 状态
gitlab-runner list

# 注销 Runner
gitlab-runner unregister --url https://gitlab.example.com --token xxx

# 启动 Runner（生产环境使用 systemd）
gitlab-runner run --user gitlab-runner --working-directory /home/gitlab-runner

# 标签机制：Job 通过 tags 选择特定 Runner
# Job 中：tags: [docker, maven]
```

**配置 Runner 缓存（以 Docker 为例）**
```toml
# /etc/gitlab-runner/config.toml
[[runners]]
  name = "docker-runner"
  executor = "docker"
  [runners.docker]
    image = "docker:24-dind"
    privileged = true
    volumes = ["/cache"]
  [runners.cache]
    Type = "s3"
    Shared = true
    [runners.cache.s3]
      Bucket = "gitlab-runner-cache"
      BucketLocation = "us-west-2"
```

### 追问方向
- Docker Executor 的 `privileged` 模式有什么安全风险？
- 如何让多个 Runner 分担不同类型的构建任务（标签策略）？
- Runner 的并发数如何配置，如何避免超过宿主机资源上限？

### 避坑提示
- Shell Executor 最简单但隔离性最差，不同 Job 可能互相影响（如端口冲突、磁盘空间耗尽）
- Docker Executor 的 `privileged` 模式在生产环境要慎用，考虑使用 Kaniko 或 BuildKit 替代 DinD
- Kubernetes Executor 的 Pod 默认使用 `Always` 重启策略，失败重试可能导致资源浪费

---

<a id="q10"></a>

## Q10: GitHub Actions——workflow 语法，on/triggers/jobs/steps，act 本地运行

### 核心答案

**基础 workflow 示例**
```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'        # 每天凌晨2点
  workflow_dispatch:           # 手动触发
    inputs:
      env:
        description: '部署环境'
        required: true
        default: 'staging'

env:
  JAVA_VERSION: '21'
  MAVEN_OPTS: -Xmx1024m

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          lfs: true

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Cache Gradle packages
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*') }}
          restore-keys: ${{ runner.os }}-gradle-

      - name: Build with Gradle
        run: ./gradlew clean build -x test

      - name: Run tests
        run: ./gradlew test

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: build/test-results/**/*.xml

      - name: Publish to Maven Repository
        if: github.ref == 'refs/heads/main'
        run: ./gradlew publish
        env:
          NEXUS_USERNAME: ${{ secrets.NEXUS_USERNAME }}
          NEXUS_PASSWORD: ${{ secrets.NEXUS_PASSWORD }}

  docker-build:
    needs: build              # 依赖 build job
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: docker-build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Kubernetes
        uses: azure/k8s-deploy@v4
        with:
          manifests: k8s/
          images: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
```

**核心语法解释**

| 语法 | 说明 | 示例 |
|------|------|------|
| `on` | 触发条件 | `push`, `pull_request`, `schedule`, `workflow_dispatch` |
| `jobs` | 所有并行 job 定义 | 顶级对象 |
| `runs-on` | 执行环境 | `ubuntu-latest`, `self-hosted`, `windows-latest` |
| `steps` | 具体步骤 | 数组，每个 step 执行一个 action 或命令 |
| `uses` | 引用 Action | `actions/checkout@v4` |
| `run` | 执行 Shell 命令 | `run: ./gradlew build` |
| `with` | Action 参数 | `with: { java-version: '21' }` |
| `env` | 环境变量 | job 级别或 step 级别 |
| `needs` | 依赖关系 | `needs: [build, test]` |
| `if` | 条件执行 | `if: github.ref == 'refs/heads/main'` |
| `secrets` | 加密密钥 | `secrets.GITHUB_TOKEN` 或自定义 |

**act 本地运行 GitHub Actions**
```bash
# 安装
brew install act

# 运行 workflow（交互式选择）
act

# 指定 event 类型
act push

# 指定作业
act -j build

# 使用本地 .env 文件
act --env-file .env.local

# 查看所有可用 events
act -l
```

### 追问方向
- `GITHUB_TOKEN` 和自定义 `secrets` 的区别是什么？
- GitHub Actions 如何实现矩阵构建（同一 Job 并行跑多个配置）？
- `actions/cache` vs `actions/upload-artifact` 的区别和使用场景？

### 避坑提示
- `ubuntu-latest` 是 GitHub 托管的虚拟机，每次运行后数据清空，不要在 `run` 中存储持久化数据
- `secrets` 无法在 `run` 的 `echo` 中直接打印（会被脱敏），调试时注意
- `actions/upload-artifact@v4` 的默认过期时间是 90 天，需要长保存要指定 `retention-days`

---

<a id="q11"></a>

## Q11: 环境管理——dev/test/staging/prod 环境分离，配置中心管理

### 核心答案

**环境分层架构**

```
开发代码提交
    ↓
DEV 环境（开发人员自助部署，自动/手动）
    ↓ 通过后
TEST 环境（QA 团队测试）
    ↓ 通过后
STAGING 环境（预生产，镜像 PROD 配置）
    ↓ 通过后
PROD 环境（正式生产）
```

**WMS 项目的多环境配置**

从 `inventory-service-backend` 的 `build.gradle` 可以看到：
```groovy
def env = providers.gradleProperty('env').orNull ?: 'DEV'
version = "0.0.1-${env.toUpperCase()}-SNAPSHOT"
```

配置文件分层（见 `wms-backend/common/src/test/resources/`）：
- `application-dev.yml` — DEV 环境
- `application-stage-local.yml` — STAGING 本地
- `application.yml` — 公共配置

Spring Boot 多环境激活：
```bash
./gradlew bootRun -Penv=STAGE -Pprofile=local
# 等价于 --spring.profiles.active=local
```

**配置中心（Nacos）实践**

WMS 项目使用 Nacos 作为配置中心（见 `bootstrap-stage.yml`）：
```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_DISCOVERY_HOST:nacos-discovery-staging.item.com:8848}
        namespace: ${NACOS_DISCOVERY_NS:item}
        username: ${NACOS_DISCOVERY_USER:item}
        password: ${NACOS_DISCOVERY_PASSWD:YjZzJWXpSPTiXkGB}
      config:
        file-extension: yml
        server-addr: ${NACOS_CONFIG_CONFIG_HOST:nacos-config-staging.item.com:8848}
```

Bootstrap 加载顺序：
```yaml
# application.yml
spring:
  config:
    import:
      - optional:classpath:bootstrap-stage.yml    # 本地 bootstrap
      - nacos:shared-${spring.profiles.active}.yml  # Nacos 共享配置
      - optional:nacos:${spring.application.name}-${spring.profiles.active}.yml  # 服务专属配置
```

**环境差异管理策略**

| 环境 | 特点 | 配置来源 |
|------|------|----------|
| DEV | 开放、频繁变更、日志详细 | 本地或配置中心 dev namespace |
| TEST | 稳定、QA 使用 | 配置中心 test namespace |
| STAGING | 镜像 PROD，数据脱敏 | 配置中心 stage namespace |
| PROD | 严格变更管理，密文配置 | 配置中心 prod namespace，高权限 |

### 追问方向
- 如何保证 DEV/STAGING/PROD 配置的一致性，避免"配置漂移"？
- Nacos 配置变更后，应用如何感知（长轮询/配置推送）？
- 敏感配置（如数据库密码、API Key）如何在配置中心安全管理？

### 避坑提示
- 不要把生产密码明文写在 Nacos 或代码仓库中，使用加密或引用 Vault
- DEV 环境的 `spring.config.import` 如果用 `optional:` 前缀，配置中心挂了不会影响启动
- 多环境配置优先级：`命令行参数 > 环境变量 > application-{profile}.yml > application.yml`

---

<a id="q12"></a>

## Q12: Docker镜像构建——Maven/Gradle 多阶段构建，构建优化

### 核心答案

**多阶段构建原理**
```
Stage 1: Builder（编译环境）
  ├── FROM maven:3.9-eclipse-temurin-21 AS builder
  ├── COPY pom.xml
  ├── RUN ./mvnw dependency:go-offline
  ├── COPY src
  └── RUN ./mvnw package -DskipTests

Stage 2: Runtime（运行时）
  ├── FROM eclipse-temurin:21-jre-alpine
  ├── COPY --from=builder /app/target/*.jar app.jar
  └── CMD ["java", "-jar", "app.jar"]
```

**WMS 项目 Dockerfile 示例**

参考 `inventory-service-backend/deploy/stage/aws/Dockerfile`：
```dockerfile
FROM stage.logisticsteam.com:5000/app-base:aplpha2.0
RUN mkdir -p /data/gc/
RUN mkdir -p /sync/log && mkdir -p /data/inventory-service-backend/ && mkdir -p /archive/logs/inventory
COPY inventory-app/build/libs/inventory-app-0.0.1-STAGE-SNAPSHOT.jar /data/inventory-service-backend/inventory-app-0.0.1-SNAPSHOT.jar
COPY deploy/stage/aws/supervisord.conf /etc/supervisor/supervisord.conf
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf"]
```

生产级 Dockerfile（多阶段构建）：
```dockerfile
# Stage 1: Build
FROM gradle:8.5-jdk21 AS builder
WORKDIR /app
COPY build.gradle settings.gradle ./
RUN gradle dependencies --configuration runtimeClasspath
COPY src ./src
RUN ./gradlew bootJar -x test

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# 安全：非 root 用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=builder /app/build/libs/*.jar app.jar

# 入口点
ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

**构建优化策略**

| 优化手段 | 说明 | 效果 |
|----------|------|------|
| **依赖缓存** | 先 COPY `pom.xml` / `build.gradle`，再 COPY 源码 | 依赖不变时跳过下载 |
| **并行构建** | `./gradlew build -x test --parallel` | 加速 |
| **无干扰编译** | `./gradlew bootJar` vs `./gradlew jar` | 仅打包必要类 |
| **镜像大小** | 使用 alpine / distroless 基础镜像 | 镜像体积缩小 5-10x |
| **分层优化** | Maven/Gradle 的 `.m2` / `.gradle` 缓存层 | 复用依赖 |
| **BuildKit 并行** | `DOCKER_BUILDKIT=1 docker build` | 自动分析依赖并行执行 |

**镜像构建缓存优化示例**
```dockerfile
# 利用 Docker 缓存优化层级
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app

# 依赖层（不变时不重新下载）
COPY pom.xml .
RUN mvn dependency:go-offline -B

# 源码层
COPY src ./src
RUN mvn package -DskipTests -B

# 运行时层
FROM eclipse-temurin:21-jre-alpine
COPY --from=builder /app/target/*.jar app.jar
```

### 追问方向
- Docker BuildKit 的 `--mount=type=cache` 如何加速 Maven/Gradle 依赖下载？
- 如何在多平台（AMD64/ARM64）同时构建镜像？
- 容器内如何正确设置 JVM 堆内存大小（`UseContainerSupport`）？

### 避坑提示
- 生产环境不要用 `latest` 标签，应该用 Git Commit SHA 或语义化版本号
- 不要在 Dockerfile 中执行 `apt-get upgrade`，会导致镜像体积增大且不安全
- 多阶段构建中，`COPY --from=builder` 不会继承 builder 的 Dockerfile 中的 `USER`，需要重新指定

---

<a id="q13"></a>

## Q13: Kubernetes部署——Helm模板、kubectl apply、服务滚动更新

### 核心答案

**Helm 基础概念**
- Helm = K8s 的包管理器（类似 yum / npm）
- Chart = K8s 应用的打包格式（模板 + values）
- Release = Chart 的运行时实例

**Helm 常用命令**
```bash
# 安装 chart
helm install wms-app ./charts/wms -n production

# 升级
helm upgrade wms-app ./charts/wms -n production

# 回滚
helm rollback wms-app 1 -n production

# 查看 release 列表
helm list -n production

# 模板调试（不实际部署）
helm template wms-app ./charts/wms -f values-prod.yaml

# 使用指定 values 文件
helm upgrade wms-app ./charts/wms \
  -f values-prod.yaml \
  --set image.tag=v1.2.3
```

**Helm Chart 目录结构**
```
charts/wms/
├── Chart.yaml          # Chart 元信息
├── values.yaml        # 默认配置
├── values-prod.yaml   # 生产覆盖
├── templates/         # K8s 资源模板
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl   # 公共模板函数
└── .helmignore
```

**values.yaml 示例**
```yaml
# values.yaml
replicaCount: 2

image:
  repository: stage.logisticsteam.com:5000/wms-app
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 4Gi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env:
  SPRING_PROFILES_ACTIVE: prod
  JAVA_OPTS: "-Xmx2g -Xms2g"
```

**Deployment + Service YAML**
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "wms.fullname" . }}
  labels:
    {{- include "wms.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  selector:
    matchLabels:
      {{- include "wms.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "wms.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: {{ .Values.service.port }}
            initialDelaySeconds: 60
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: {{ .Values.service.port }}
            initialDelaySeconds: 30
            periodSeconds: 5
          env:
            {{- toYaml .Values.env | nindent 12 }}
---
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "wms.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  selector:
    {{- include "wms.selectorLabels" . | nindent 4 }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.port }}
```

**kubectl apply 完整流程**
```bash
# 部署（create or update）
kubectl apply -f deployment.yaml

# 查看部署状态
kubectl rollout status deployment/wms-app -n production

# 查看 Pod
kubectl get pods -n production -l app=wms-app

# 滚动更新（修改镜像版本）
kubectl set image deployment/wms-app wms-app=registry/wms:v1.2.3 -n production

# 查看历史版本
kubectl rollout history deployment/wms-app -n production

# 回滚到上一个版本
kubectl rollout undo deployment/wms-app -n production

# 回滚到指定版本
kubectl rollout undo deployment/wms-app -n production --to-revision=3
```

**滚动更新策略**
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 最多超出期望副本数
      maxUnavailable: 0    # 最少保留的可用副本数
```

### 追问方向
- Helm 3 的 `template` vs `install --dry-run` 有什么区别？
- 如何在 Helm Hook 中处理数据库迁移（Flyway/Liquibase）？
- `readinessProbe` 和 `livenessProbe` 的区别，以及何时需要两个都用？

### 避坑提示
- `kubectl apply` 是幂等的，但 `kubectl create` 不是，多次运行会报错
- Helm upgrade 默认不会自动回滚，失败时需要手动 `helm rollback`
- 生产环境部署应该使用 `--atomic` 参数，失败时自动回滚

---

<a id="q14"></a>

## Q14: 配置加密——SOPS / Sealed Secrets / KSOPS，敏感信息管理

### 核心答案

**敏感信息管理的重要性**

WMS 项目中 `bootstrap-stage.yml` 包含明文密码（示例中为测试环境密码）：
```yaml
password: ${NACOS_DISCOVERY_PASSWD:YjZzJWXpSPTiXkGB}
```
生产环境中这些密码必须加密管理，不能明文出现在 Git 仓库中。

**方案一：SOPS（Secret Management）**

Mozilla SOPS（Secrets OPerationS）：
```bash
# 安装
brew install sops

# 创建加密文件（使用 PGP 或 GCP KMS）
sops --encrypt secrets.yaml > secrets.enc.yaml

# 编辑加密文件
sops secrets.enc.yaml

# 解密
sops --decrypt secrets.enc.yaml

# 在 CI/CD 中使用（不暴露明文）
sops --decrypt --in-place secrets.enc.yaml
kubectl apply -f secrets.enc.yaml
```

`.sops.yaml` 配置：
```yaml
creation_rules:
  - path_regex: secrets/.*
    pgp: A3C3F3C4D3E3...
    gcp_kms: projects/my-project/locations/global/keyRings/my-ring/cryptoKeys/sops-key
```

**方案二：Sealed Secrets（Bitnami）**

Kubernetes 原生方案，使用非对称加密：
```bash
# 安装 Sealed Secrets Controller
helm install sealed-secrets bitnami/sealed-secrets -n kube-system

# 公钥自动保存在集群中
# 私钥由 Controller 持有，用于解密

# 客户端加密（需要公钥）
kubeseal --cert sealed-secrets-pub-cert.pem --format=yaml < secrets.yaml > sealed-secrets.yaml

# SealedSecret 资源（可安全提交到 Git）
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: wms-db-credentials
  namespace: production
spec:
  encryptedData:
    password: AgA...  # 加密后的值
```

**方案三：KSOPS（Kustomize + SOPS）**

结合 Kustomize 的 GitOps 流程：
```bash
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
generators:
  - secretGenerator.yaml

# secretGenerator.yaml
apiVersion: secrets.doppler.com/v1alpha1
kind: DopplerSecret
metadata:
  name: wms-secrets
spec:
  project: wms-backend
  config: production
  secrets:
    DB_PASSWORD: ""
    API_KEY: ""
```

**CI/CD 中的敏感信息注入最佳实践**

```yaml
# GitLab CI 示例
variables:
  SECRETS_ENC_FILE: secrets.enc.yaml

before_script:
  - which sops || apt-get install -y sops
  - sops --decrypt $SECRETS_ENC_FILE > app-secrets.yaml

deploy:
  stage: deploy
  script:
    - kubectl apply -f app-secrets.yaml
    - kubectl apply -f deployment.yaml
```

### 追问方向
- Sealed Secrets 的密钥轮换机制是什么？如何定期更新？
- SOPS 的 PGP 和 GCP KMS 方案各有什么优缺点？
- GitOps 场景下（ArgoCD/Flux），如何让 Sealed Secrets 自动解密？

### 避坑提示
- **永远不要**将加密密钥直接提交到 Git 仓库
- Sealed Secrets Controller 挂了之后，已加密的 SealedSecret 无法解密，HA 部署时要注意
- SOPS 的 `.sops.yaml` 配置文件本身不需要加密，但其中的 PGP fingerprint 或 KMS 资源名要保护好

---

<a id="q15"></a>

## Q15: 制品管理——Harbor / Nexus / JFrog Artifactory，镜像签名/扫描

### 核心答案

**三大制品仓库对比**

| 维度 | Harbor | Sonatype Nexus 3 | JFrog Artifactory |
|------|---------|------------------|-------------------|
| **定位** | Docker 镜像仓库 + CNCF 项目 | 通用制品仓库（Java/npm/Docker） | 通用制品仓库 + 安全扫描 |
| **镜像管理** | 强大（镜像签名、分发、复制） | 支持 | 支持 |
| **开源** | Apache 2.0，开源免费 | 开源版功能受限 | 开源版 Artifactory OSS |
| **Helm Chart** | 支持 | 支持 | 支持 |
| **安全扫描** | Trivy / Clair 内置 | Nexus Firewall | Xray（企业版） |
| **高可用** | 企业版 | 企业版 | 企业版 |
| **适用** | 专注 Docker 镜像 | 混合语言制品 | 企业级大规模 |

**WMS 项目的制品仓库**

从 `inventory-service-backend/build.gradle` 可见：
```groovy
repositories {
    maven {
        name = 'ali'
        url = project.findProperty("mvn.lt-maven-repo.ali.snapshot.url")
        credentials(PasswordCredentials) {
            username project.findProperty("mvn.lt-maven-repo.ali.username")
            password project.findProperty("mvn.lt-maven-repo.ali.password")
        }
    }
}
```

镜像推送（Dockerfile）：
```dockerfile
# stage 环境镜像
FROM stage.logisticsteam.com:5000/app-base:aplpha2.0
# 推送到阿里云私有镜像仓库
```

**Harbor 镜像签名与验证（Cosign）**
```bash
# 安装 Cosign
brew install cosign

# 签名镜像
cosign sign --key cosign.key stage.logisticsteam.com:5000/wms-app:v1.2.3

# 验证镜像
cosign verify --key cosign.pub stage.logisticsteam.com:5000/wms-app:v1.2.3

# CI/CD 中集成
cosign sign --yes --upload=false \
  --key=${COSIGN_KEY} \
  ${IMAGE_URL}:${IMAGE_TAG}
```

**Harbor 安全扫描（Trivy）**
```bash
# 直接扫描镜像
trivy image stage.logisticsteam.com:5000/wms-app:v1.2.3

# 输出格式
trivy image --format json --output result.json \
  stage.logisticsteam.com:5000/wms-app:v1.2.3

# 扫描漏洞级别
trivy image --severity HIGH,CRITICAL \
  stage.logisticsteam.com:5000/wms-app:v1.2.3
```

**Harbor 的复制策略（多站点分发）**
```yaml
# Harbor 复制规则示例
replication:
  - name: replicate-to-aliyun
    src_registry:
      url: https://harbor.source.com
      credential_ref:
        name: harbor-credential
    dest_registry:
      url: https://harbor.aliyun.com
      credential_ref:
        name: harbor-credential
    filters:
      - name: name
        value: ".*/wms-.*"
    trigger:
      type: event_based   # 实时同步
    deletion: false
    override: true
```

### 追问方向
- Harbor 的 "Notary 签名" 和 "Cosign 签名" 有什么区别？
- 如何在 CI Pipeline 中自动扫描镜像漏洞并设置质量门禁？
- 多区域镜像复制的一致性如何保证？

### 避坑提示
- Harbor 的 `docker-compose` 部署方式不支持横向扩展，生产环境建议用 Kubernetes 部署
- 镜像 tag 不要用 `latest`，用 Git SHA 或语义化版本，否则无法回滚
- Nexus 3 的 Docker Bearer Token Realm 需要手动启用，否则 Docker login 会失败

---

<a id="q16"></a>

## Q16: 蓝绿部署——原理、流量切换、回滚策略

### 核心答案

**蓝绿部署原理**

```
[路由器/负载均衡器]
       ↓
  ┌─────────┬─────────┐
  │  Blue   │  Green  │
  │ (当前)   │ (新版本) │
  │ v1.0.0  │ v1.1.0  │
  └─────────┴─────────┘
```

两套完全相同的生产环境：
- **Blue**：当前生产环境（v1.0.0）
- **Green**：新版本环境（v1.1.0）

**蓝绿部署流程**
```
1. 部署 v1.1.0 到 Green 环境
2. 验证 Green 环境健康
3. 切换流量：LB 指向 Green
4. 监控新版本指标
5. 如有问题，立即回滚：LB 切回 Blue
```

**在 Kubernetes 中的实现**

使用 `Service + Deployment` 分离：
```yaml
# Blue Deployment (当前生产 v1.0.0)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wms-app-blue
  labels:
    version: blue
    active: "true"
spec:
  replicas: 10
  selector:
    matchLabels:
      app: wms-app
      slot: blue
  template:
    metadata:
      labels:
        app: wms-app
        version: v1.0.0
        slot: blue
---
# Green Deployment (新版本 v1.1.0)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wms-app-green
  labels:
    version: green
    active: "false"
spec:
  replicas: 10
  selector:
    matchLabels:
      app: wms-app
      slot: green
---
# Service 指向 Blue（活动环境）
apiVersion: v1
kind: Service
metadata:
  name: wms-app
spec:
  selector:
    app: wms-app
    slot: blue    # 切换为 green 即完成蓝绿
  ports:
    - port: 80
      targetPort: 8080
```

**切换流量**
```bash
# 切换到 Green（新版本）
kubectl patch service wms-app -p '{"spec":{"selector":{"slot":"green"}}}'

# 验证 Green
kubectl rollout status deployment wms-app-green

# 回滚到 Blue（旧版本）
kubectl patch service wms-app -p '{"spec":{"selector":{"slot":"blue"}}}'
```

**回滚策略**
```
立即回滚（生产故障）：
  kubectl patch service wms-app -p '{"spec":{"selector":{"slot":"blue"}}}'

蓝绿回滚特点：Blue 环境保留旧版本镜像，不需要重新部署，
直接切换流量即可，回滚时间 < 1 分钟
```

**蓝绿部署的优缺点**

| 优点 | 缺点 |
|------|------|
| 回滚速度快（切换 LB 即可） | 需要双倍资源（两套环境） |
| 零停机部署 | 数据库变更需要额外处理 |
| 易于验证新版本 | 流量切换有窗口期 |
| 出现问题可立即切回 | 资源成本高 |

### 追问方向
- 蓝绿部署中，数据库 schema 变更怎么处理（两套环境共用 DB）？
- 如何处理 session 保持问题（用户 IP 哈希 vs 最小连接数）？
- 蓝绿和金丝雀如何结合使用？

### 避坑提示
- Blue/Green 环境必须完全隔离，包括数据库、缓存、消息队列，否则新版本的破坏性变更会影响旧版本
- 流量切换前必须确认 Green 环境**预热完成**（JVM 启动、连接池建立），否则切换后会瞬间高延迟
- Blue 环境在确认 Green 稳定前**不要删除**，用于紧急回滚

---

<a id="q17"></a>

## Q17: 金丝雀发布——Canary Release，Istio / Argo Rollouts 灰度策略

### 核心答案

**金丝雀发布原理**

```
              ┌──────────────┐
              │  10% 流量   │ ← 灰度（新版本）
              └──────┬───────┘
                     ↓
[用户流量] → [路由器/LB] → 90% → [v1.0.0 Blue]  ──→ 旧版本（稳定）
                     └──→ 10% → [v1.1.0 Canary] ──→ 新版本（验证）
```

逐步放量：5% → 20% → 50% → 100%，每个阶段观察业务指标。

**Argo Rollouts 灰度策略**

Argo Rollouts 是 Kubernetes 原生的渐进式交付控制器：
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: wms-app
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 5       # 第一步：5% 流量
        - pause: {}           # 手动暂停，等待确认
        - setWeight: 20      # 第二步：20%
        - pause: {duration: 10m}  # 自动暂停10分钟后继续
        - setWeight: 50      # 第三步：50%
        - pause: {}          # 手动暂停
        - setWeight: 100     # 全量
      canaryMetadata:
        labels:
          role: canary
      stableMetadata:
        labels:
          role: stable
      trafficRouting:
        istio:
          virtualService:
            name: wms-app-vs
            routes:
              - primary
      analysis:
        templates:
          - templateName: success-rate
        startingStep: 1
        args:
          - name: service-name
            value: wms-app-canary
---
# 分析模板（自动验证）
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
    - name: service-name
  metrics:
    - name: success-rate
      interval: 2m
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_server_requests_seconds_count{
              job="{{args.service-name}}",
              status=~"2.."
            }[2m])) /
            sum(rate(http_server_requests_seconds_count{
              job="{{args.service-name}}"
            }[2m]))
```

**Istio 灰度（VirtualService + DestinationRule）**
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: wms-app
spec:
  hosts:
    - wms-app
  http:
    - route:
        - destination:
            host: wms-app
            subset: v1
          weight: 90
        - destination:
            host: wms-app
            subset: v2
          weight: 10
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: wms-app
spec:
  host: wms-app
  subsets:
    - name: v1
      labels:
        version: v1.0.0
    - name: v2
      labels:
        version: v1.1.0
```

**渐进式放量命令**
```bash
# Argo Rollouts 手动控制
kubectl argo rollouts set-weight wms-app 20    # 调整流量权重
kubectl argo rollouts pause wms-app            # 暂停
kubectl argo rollouts resume wms-app           # 继续
kubectl argo rollouts abort wms-app            # 中止，回滚
kubectl argo rollouts status wms-app           # 查看状态
```

**灰度策略的判断指标**

| 指标类型 | 示例 | 阈值 |
|----------|------|------|
| 成功率 | HTTP 2xx 比例 | ≥ 99.5% |
| 延迟 | P99 响应时间 | ≤ 500ms |
| 错误率 | 5xx 错误比例 | ≤ 0.1% |
| 业务 | 订单转化率 | 波动 ≤ 5% |

### 追问方向
- 金丝雀的"金丝雀"典故是什么？为什么叫这个名字？
- Argo Rollouts 和 Flagger 有什么区别？
- 金丝雀发布中如何处理数据库迁移（先迁移还是后迁移）？

### 避坑提示
- 金丝雀 Pod 的资源配额要和 Stable 一致，否则资源限制会导致性能差异
- 灰度期间监控要同时看新旧两个版本，**对比指标**才能发现问题
- `pause: {}` 是无限期暂停，必须手动 `resume`，生产环境要设置合理的 `duration`

---

<a id="q18"></a>

## Q18: 特性开关——Feature Toggle，Unleash / LaunchDarkly，代码解耦

### 核心答案

**Feature Toggle 概念**

特性开关（Feature Flag / Feature Toggle）允许在**不重新部署**的情况下，动态控制功能开启/关闭。

```
传统发布：          特性开关发布：
代码合并 → 构建 → 部署 → 上线（all-or-nothing）
                              ↓
              代码合并 → 构建 → 部署 → 开关打开 → 灰度用户 → 全量
```

**典型应用场景**

| 场景 | 说明 |
|------|------|
| 灰度发布 | 10% 用户先体验新功能 |
| A/B 测试 | 不同用户看到不同 UI |
| 快速回滚 | 问题出现立即关闭开关，无需重新部署 |
| 护栏 | 新功能上线但默认关闭，稳定性验证后再开启 |
| 长期分支避免 | 新功能在 main 分支开发，用开关控制 |

**Unleash（开源）部署**
```bash
# Docker 快速部署
docker run -d \
  -e DATABASE_URL=sqlite:/unleash/unleash.db \
  -p 4242:4242 \
  unleashorg/unleash

# 前端 SDK
import { useUnleash } from 'unleash-proxy-client';

const unleash = new Unleash({
  url: 'http://localhost:4242/api/',
  appName: 'wms-web',
  clientKey: 'client-key-123',
});

await unleash.start();

// 判断特性开关
if (unleash.isEnabled('new-inventory-view')) {
  return <NewInventoryView />;
} else {
  return <LegacyInventoryView />;
}
```

**LaunchDarkly（商业化）集成**
```java
import com.launchdarkly.sdk.*;
import com.launchdarkly.sdk.server.*;

LDClient client = new LDClient("sdk-key-xxx");

User user = User.builder("user-id-123")
    .email("user@example.com")
    .attribute("tenant", "premium")
    .build();

// 判断开关
boolean showNewFeature = client.boolVariation("new-order-flow", user, false);

// 实时变更监听
client.getFlagTracker().addFlagChangeListener(event -> {
    System.out.println("Feature flag changed: " + event.getKey());
});
```

**Spring Boot 集成**
```java
@Configuration
public class FeatureToggleConfig {
    @Bean
    public UnleashConfig unleashConfig() {
        return UnleashConfig.builder()
            .appName("wms-backend")
            .instanceUrl("http://unleash:4242/api/")
            .apiKey("unleash-client-key")
            .build();
    }

    @Bean
    public Unleash unleash(UnleashConfig config) {
        return new DefaultUnleash(config);
    }
}

@Service
public class OrderService {
    @Autowired
    private Unleash unleash;

    public Order create(OrderCmd cmd) {
        // 护栏：新功能默认关闭
        if (unleash.isEnabled("async-order-processing")) {
            return processOrderAsync(cmd);
        }
        return processOrderSync(cmd);
    }
}
```

**开关管理生命周期**

```
开发阶段：  开关已创建，默认关闭
          → 代码合并到 main，但不暴露给用户

测试阶段：  在测试环境开启，QA 验证功能

发布阶段：  逐步放量（5% → 20% → 50% → 100%）

稳定阶段：  全量开启，监控无异常

废弃阶段：  删除旧代码 + 删除开关定义
          注意：废弃后必须清理旧代码，否则技术债务累积
```

### 追问方向
- Feature Toggle 和环境配置（application.yml）的本质区别是什么？
- 如何避免"永久开关"（永远不删除的开关）？
- Feature Toggle 对测试有什么影响（单元测试/集成测试）？

### 避坑提示
- **开关不是永久解决方案**，功能验证后必须删除旧代码（废弃开关），否则代码库会被废弃逻辑污染
- 开关越多，维护成本越高，建议用 `unleash cleanup` 工具定期清理
- 开关状态变更后，SDK 会通过 SSE/WebSocket 推送更新，本地无需轮询

---

<a id="q19"></a>

## Q19: 监控告警——Prometheus + Grafana + AlertManager，告警收敛、静默期

### 核心答案

**Prometheus 架构**

```
[应用服务] ──→ [Prometheus Server] ──→ [AlertManager] ──→ [Email/Slack/钉钉]
     ↓               ↓
  指标暴露           时序数据库
(metrics端点)        + 抓取配置
     ↓
[Exporters] ──→ [Node Exporter] / [MySQLD Exporter] / [Kafka Exporter]
```

**Spring Boot 集成 Micrometer + Prometheus**
```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
```

Prometheus 抓取配置：
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - "alerts/*.yml"

scrape_configs:
  - job_name: 'wms-backend'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['wms-app:8080']
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.+):.*'
        target_label: instance
```

**告警规则示例**
```yaml
# alerts/wms-backend.yml
groups:
  - name: wms-backend
    rules:
      # JVM 堆内存告警
      - alert: JVMHighMemoryUsage
        expr: |
          jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.85
        for: 5m
        labels:
          severity: warning
          team: backend
        annotations:
          summary: "JVM 堆内存使用率超过 85%"
          description: "{{ $labels.application }} 实例 {{ $labels.instance }} 堆内存使用率 {{ $value | humanizePercentage }}"

      # 服务不可用告警
      - alert: ServiceDown
        expr: up{job="wms-backend"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "服务实例不可用"
          description: "{{ $labels.application }} 所有实例均不可用"

      # 5xx 错误率告警
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{
            job=~"wms-.*",
            status=~"5.."
          }[5m])) / 
          sum(rate(http_server_requests_seconds_count{
            job=~"wms-.*"
          }[5m])) > 0.01
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "HTTP 5xx 错误率超过 1%"
```

**AlertManager 配置**
```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname', 'severity']
  group_wait: 30s        # 告警聚合等待时间
  group_interval: 5m     # 相同 group 的重复告警间隔
  repeat_interval: 4h    # 告警重复发送间隔
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'critical-alerts'
      continue: true
    - match:
        severity: warning
      receiver: 'warning-alerts'

receivers:
  - name: 'default'
    email_configs:
      - to: 'devops@item.com'
        send_resolved: true

  - name: 'critical-alerts'
    webhook_configs:
      - url: 'http://dingtalk-hook:8060/dingtalk/webhook'
    pagerduty_configs:
      - service_key: 'xxx'

  - name: 'warning-alerts'
    email_configs:
      - to: 'team@item.com'
```

**Grafana Dashboard 关键面板**

| 面板 | PromQL | 说明 |
|------|--------|------|
| 请求 QPS | `sum(rate(http_server_requests_seconds_count[1m]))` | 每秒请求数 |
| P99 延迟 | `histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m]))` | 99%分位延迟 |
| 错误率 | `sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) / sum(rate(http_server_requests_seconds_count[5m]))` | 5xx 占比 |
| JVM 堆使用 | `jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}` | 堆内存使用率 |
| 活跃线程 | `jvm_threads_live_threads` | JVM 活跃线程数 |

**告警收敛（Alert Silencing / Inhibition）**

```yaml
# AlertManager Inhibition（抑制规则）
# 当某服务整体 down 时，抑制该服务下所有实例的告警
inhibit_rules:
  - source_match:
      severity: critical
    source_labels: [alertname]
    target_match:
      severity: warning
    target_labels:
      service: critical  # 匹配相同 service 标签
```

**静默期（Mute / Silencing）**

```bash
# 使用 amtool 创建静默期（AlertManager 自带工具）
amtool silence add \
  --alertname "JVMHighMemoryUsage" \
  --reason "计划内维护" \
  --duration "2h"

# 在 Grafana UI 中创建静默（方便非运维人员）
# Grafana → Alerting → Silences → New silence
```

### 追问方向
- Prometheus 的 `for` 参数是什么作用？为什么重要？
- AlertManager 的 `group_wait` 和 `group_interval` 的区别？
- 如何防止告警疲劳（Alert Fatigue）？

### 避坑提示
- 告警 `for` 设为 0 会导致任何瞬时抖动都触发告警，生产环境建议至少 `5m`
- 告警没有 `for` 和有 `for` 的行为完全不同：Prometheus 会在 `for` 持续期间才发送 `firing` 状态
- Grafana Dashboard 不要只依赖平均值，P50/P90/P99 才能反映真实用户体验

---

<a id="q20"></a>

## Q20: DevOps文化——DevOps三步工作法，团队协作，测量（MTTR/MTBF/CI频率）

### 核心答案

**DevOps 三步工作法（The Three Ways）**

DevOps 之父 Gene Kim 提出的三步工作法：

**第一步：流动原则（The Flow）**
- 缩短从代码提交到生产的**前置时间**（Lead Time）
- 消除工作中的浪费（Waiting、Handoffs、Defects）
- 实践：持续集成、自动化测试、流水线化部署、小批量工作

**第二步：反馈原则（The Feedback）**
- 建立快速、持续的反馈循环
- 从右向左看：生产 → 运营 → 开发 → 客户
- 实践：监控告警、灰度发布、Code Review、Postmortem

**第三步：持续学习原则（The Learning）**
- 建立学习型组织文化
- 实践：事后分析（Postmortem）、Blameless Review、混沌工程

**DevOps 关键度量指标**

| 指标 | 全称 | 定义 | 目标 |
|------|------|------|------|
| **MTTR** | Mean Time To Recovery | 平均恢复时间（故障到恢复） | < 1 小时 |
| **MTBF** | Mean Time Between Failures | 平均故障间隔 | 越长越好 |
| **MTTF** | Mean Time To Failure | 平均故障发生时间 | 越长越好 |
| **部署频率** | Deployment Frequency | 代码部署到生产的频率 | 每天多次 |
| **变更前置时间** | Lead Time for Changes | 代码提交到生产的平均时间 | < 1 周 |
| **变更失败率** | Change Failure Rate | 变更导致生产失败的百分比 | < 15% |
| **CI 频率** | Integration Frequency | 每天/每周代码集成次数 | 每天多次 |
| **恢复时间** | Time to Restore | 从故障恢复到正常服务的时间 | < 1 小时 |

**DORA 指标（DORA State of DevOps Report）**

DORA（DevOps Research and Assessment）将团队分为四类：

| 精英团队（Elite） | 高绩效团队 | 中等绩效 | 低绩效 |
|------------------|-----------|---------|--------|
| 部署频率：按需（每天多次） | 每周/每天多次 | 每周一次 | 每月一次 |
| 前置时间：< 1 小时 | < 1 周 | 1-6 个月 | > 6 个月 |
| 恢复时间：< 1 小时 | < 1 天 | < 1 周 | > 6 个月 |
| 变更失败率：0-15% | 16-30% | 16-30% | > 30% |

**DevOps 团队协作模式**

```
传统 Silo：                      DevOps 模式：
┌─────┐  ┌─────┐  ┌─────┐       ┌─────────────────────┐
│ Dev │  │  QA │  │ Ops │       │   SRE / DevOps   │
└──┬──┘  └──┬──┘  └──┬──┘       │   跨职能团队        │
   │        │        │           │                    │
   │ handoff        │            │ Dev + QA + Ops     │
   └───────┴────────┘            └────────┬────────────┘
         低效、延迟                         高效、协作
```

**DevOps 文化实践**

| 实践 | 工具 | 说明 |
|------|------|------|
| 基础设施即代码 | Terraform, Ansible, Pulumi | 版本化、可重复、可测试 |
| GitOps | ArgoCD, Flux | Git 作为唯一事实来源 |
| 监控驱动开发 | Prometheus, Grafana | OODA 循环（Observe-Orient-Act） |
| 混沌工程 | Chaos Monkey, Gremlin | 主动注入故障，验证韧性 |
| Postmortem | 文档 + 复盘会 | 无责复盘，持续改进 |
| SRE | SLO/SLI/SLA | 基于错误预算的发布决策 |

### 追问方向
- SRE 和 DevOps 有什么区别？SRE 是 DevOps 的一种实现吗？
- 你们的 MTTR 是多少？如何测量的？
- 如何在团队中推广 DevOps 文化，避免"墙上制度"？

### 避坑提示
- **DevOps 不是工具，不是岗位，是文化**——招一个 DevOps 工程师不能解决团队协作问题
- MTTR 的计算要从**故障发现**开始，而不是从**故障发生**开始，因为很多故障发现时已过了一段时间
- CI 频率不是越高越好，关键是**有意义的集成**——提交代码质量差、破坏构建的高频 CI 没有价值

---

## 参考项目代码索引

| 面试题 | 项目文件 | 说明 |
|--------|----------|------|
| Q5, Q7 | `wms-web/jenkinsfile/JenkinsfileStage` | Jenkinsfile 完整示例 |
| Q7, Q12 | `inventory-service-backend/build.gradle` | Gradle 构建配置，多环境版本管理 |
| Q11 | `inventory-service-backend/inventory-app/src/main/resources/bootstrap-stage.yml` | Nacos 配置中心集成 |
| Q11 | `wms-backend/common/src/test/resources/application-dev.yml` | 多环境配置文件 |
| Q12 | `inventory-service-backend/deploy/stage/aws/Dockerfile` | Docker 多阶段构建示例 |
| Q12 | `wms-backend/wms-app/deploy/prod/Dockerfile` | 生产环境 Dockerfile |
| Q5 | `wms-web/jenkinsfile/JenkinsfileDev` | DEV 环境 Jenkinsfile |

> 建议补充：以上项目中尚未发现 `.gitlab-ci.yml`、GitHub Actions workflows、Helm charts、Istio 配置、Kubernetes YAML、SOPS/Sealed Secrets 配置，建议在实际项目中补充这些真实文件以进一步增强面试说服力。
