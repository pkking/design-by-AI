# 系统设计文档: GitOps CI/CD 后台系统

## 1. 系统架构

本系统采用微服务架构，由多个协作的后台组件构成，以实现高可用性和可扩展性。底层计算资源将由 Kubernetes 进行统一调度和管理。

### 1.1. 核心组件

1.  **Webhook Gateway (入口网关):**
    *   一个暴露在公网的 HTTP 服务。
    *   **职责:** 接收来自 Git 平台的 Webhook 请求，验证请求的签名/Secret Token，将合法事件（如 Push, Pull Request）格式化后推送到消息队列。

2.  **Orchestrator (编排器):**
    *   系统的核心大脑，订阅消息队列中的事件。
    *   **职责:**
        *   消费事件，解析仓库信息、Commit ID 等。
        *   使用预置凭证（SSH Key 或 OAuth Token）克隆指定版本的代码。
        *   解析仓库中的 `ci.yaml` 文件。
        *   根据 `ci.yaml` 的定义，生成一个有向无环图 (DAG) 的任务执行计划。
        *   对于每个待执行的任务 (Job)，向 `Scheduler` 发起调度请求。

3.  **Scheduler (调度器):**
    *   接收来自 `Orchestrator` 的任务调度请求。
    *   **职责:**
        *   解析任务所需的 `runner` 别名，并从 `Admin Config Service` 获取对应的物理资源规格（CPU, Memory）。
        *   解析任务所需的 `secrets` 名称，并从 `Secret Store` 获取加密后的秘钥值。
        *   **生成一个 Kubernetes Pod 定义 (PodSpec)**，该定义包含了：
            *   用户指定的 `image`。
            *   转换后的 `resources` (requests/limits)。
            *   需要注入的环境变量 (`env`) 和秘钥 (`secrets`)。
            *   待执行的 `steps` 命令。
        *   调用 Kubernetes API Server 创建该 Pod。

4.  **Admin Config Service (管理员配置服务):**
    *   一个内部服务，提供 API 或从中心配置文件加载配置。
    *   **职责:** 存储并提供 `runner` 别名与物理资源规格的映射关系。

5.  **Secret Store (秘钥存储):**
    *   一个安全的存储系统，如 HashiCorp Vault 或 Kubernetes Secrets。
    *   **职责:** 加密存储管理员为各个仓库配置的秘钥，并提供给 `Scheduler` 在创建 Pod 时安全地注入。

6.  **Status Reporter (状态报告器):**
    *   一个订阅任务状态变更事件的服务。
    *   **职责:** 当任务状态（开始、成功、失败）发生变化时，调用 Git 平台的 Status API 或 Checks API，将结果回传到对应的 Commit 或 Pull Request 页面。

7.  **Log Collector (日志收集器):**
    *   负责从 Kubernetes 中运行的 Pod 收集实时日志。
    *   **职责:** 将日志流式传输到集中的日志存储系统（如 Elasticsearch, Loki），并提供一个简单的只读 API 供 Git 平台状态链接跳转查看。

### 1.2. 数据流 (Push 事件示例)

1.  开发者 `git push` 到 GitHub。
2.  GitHub 向 **Webhook Gateway** 发送一个带有 `X-Hub-Signature` 的 POST 请求。
3.  **Gateway** 验证签名，成功后将事件推送到消息队列 (e.g., RabbitMQ)。
4.  **Orchestrator** 从队列中消费该事件，克隆代码。
5.  **Orchestrator** 解析 `ci.yaml`，发现一个名为 `build-job` 的任务，该任务需要 `runner: medium` 并引用了秘钥 `DOCKER_PASSWORD`。
6.  **Orchestrator** 向 **Status Reporter** 发送 "pending" 状态更新请求。
7.  **Status Reporter** 调用 GitHub Checks API，在 Commit 旁显示一个黄色的 "pending" 标记。
8.  **Orchestrator** 向 **Scheduler** 发起调度请求，包含任务元数据。
9.  **Scheduler** 查询 **Admin Config Service** 获得 `medium` 对应的 `{"cpu": "2", "memory": "4Gi"}`。
10. **Scheduler** 查询 **Secret Store** 获得 `DOCKER_PASSWORD` 的引用路径。
11. **Scheduler** 构建 Pod 定义，并通过 Kubernetes API 创建 Pod。
12. Kubernetes 在某个节点上启动该 Pod，并将秘钥作为环境变量注入。
13. Pod 内的 Agent 开始执行 `ci.yaml` 中定义的 `steps`。日志被 **Log Collector** 收集。
14. Pod 执行完毕，**Orchestrator** 收到完成通知（成功/失败）。
15. **Orchestrator** 向 **Status Reporter** 发送最终状态。
16. **Status Reporter** 调用 GitHub Checks API，将标记更新为绿色的 "success" 或红色的 "failure"。

## 2. 核心概念定义

### 2.1. `ci.yaml` 文件结构

```yaml
# ci.yaml

# 定义工作流的名称
name: My Application CI/CD

# 定义触发规则
on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main
  schedule:
    - cron: '0 2 * * *' # 每天凌晨2点执行

# 定义任务集合
jobs:
  build:
    # 任务名称
    name: Build and Test
    # 物理资源别名
    runner: medium
    # 运行环境的 Docker 镜像
    image: maven:3.8-openjdk-17
    # 环境变量
    env:
      MAVEN_OPTS: -Dmaven.repo.local=.m2/repository
    # 引用的秘钥名称
    secrets:
      - source: SONAR_TOKEN
        target: SONAR_LOGIN_TOKEN # 在容器内作为 SONAR_LOGIN_TOKEN 环境变量
    # 执行步骤
    steps:
      - name: Checkout Code
        # 'uses' 关键字可以用于引用预定义的通用操作，如代码检出
        uses: actions/checkout@v3

      - name: Build with Maven
        run: mvn clean install

      - name: Analyze with SonarQube
        run: |
          mvn sonar:sonar \
            -Dsonar.host.url=https://sonarqube.example.com \
            -Dsonar.token=$SONAR_LOGIN_TOKEN

  deploy:
    name: Deploy to Staging
    # 'needs' 关键字定义任务依赖，确保 build 成功后才执行 deploy
    needs: build
    runner: small
    image: alpine/helm:3.10
    secrets:
      - KUBECONFIG_STAGING
    steps:
      - name: Deploy with Helm
        run: |
          echo "$KUBECONFIG_STAGING" | base64 -d > ./kubeconfig
          export KUBECONFIG=./kubeconfig
          helm upgrade --install my-app ./charts/my-app