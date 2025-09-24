# 系统设计文档: GitOps CI/CD 后台系统 (Rust + Argo Workflows)

## 1. 系统架构

本系统以 Rust 构建后台服务，并利用 Argo Workflows 作为核心的 CI/CD 执行引擎。系统通过动态生成 Argo `Workflow` 或 `CronWorkflow` Custom Resource Definitions (CRDs) 来响应 Git 事件，将复杂的任务调度、依赖管理、资源分配和执行逻辑完全委托给 Argo。

### 1.1. 核心组件

1.  **Webhook Gateway (Rust Service):**
    *   一个基于 Rust (例如 Axum, Actix Web) 构建的高性能、异步 HTTP 服务。
    *   **职责:** 接收、验证 Git Webhook 请求，并将合法的、已解析的事件推送到消息队列。

2.  **Workflow Generator (Rust Service):**
    *   **替代了原设计中的 `Orchestrator` 和 `Scheduler`。** 这是系统的核心业务逻辑所在。
    *   **职责:**
        *   消费消息队列中的事件。
        *   克隆代码，解析 `ci.yaml` 文件。
        *   **将 `ci.yaml` 的声明式配置转换为一个 Argo `Workflow` 或 `CronWorkflow` 的 YAML/JSON manifest。**
        *   查询 `Admin Config Service` 获取 `runner` 对应的资源规格，并填充到 Workflow manifest 中。
        *   查询 `Secret Store`，将 `secrets` 引用转换为 Argo 所支持的 Kubernetes Secret 引用。
        *   使用 Kubernetes API Client (例如 `kube-rs`) 将生成的 Workflow manifest `apply` 到 Kubernetes 集群中。

3.  **Status Reporter (Rust Service):**
    *   **职责:**
        *   监听 Kubernetes API Server 中 `Workflow` 资源的状态变更。
        *   或者，配置 Argo Workflows 使用 Webhook 或事件通知机制，将 `Workflow` 的状态（Running, Succeeded, Failed）推送给 `Status Reporter`。
        *   调用 Git 平台的 API 将状态回传到对应的 Commit 或 Pull Request。

4.  **Admin Config Service (Rust Service or Static Config):**
    *   一个简单的 Rust 服务或由 ConfigMap 挂载的配置文件。
    *   **职责:** 提供 `runner` 别名与物理资源规格（CPU, Memory）的映射。Argo Workflows 直接支持在 Pod 模板中定义 `resources`，因此映射非常直接。

5.  **Underlying Infrastructure (底层基础设施):**
    *   **Kubernetes Cluster:** 所有服务的运行环境。
    *   **Argo Workflows:** 已安装并在此集群中运行的执行引擎。
    *   **Kubernetes Secrets:** 作为默认的秘钥存储机制，与 Argo Workflows 无缝集成。
    *   **Message Queue:** (e.g., NATS, RabbitMQ) 用于解耦 `Webhook Gateway` 和 `Workflow Generator`。

### 1.2. 数据流 (Push 事件示例 with Argo)

1.  开发者 `git push`。
2.  Git 平台向 **Webhook Gateway (Rust)** 发送 Webhook。
3.  **Gateway** 验证后将事件推入消息队列。
4.  **Workflow Generator (Rust)** 消费事件，克隆代码，解析 `ci.yaml`。
5.  **Generator** 开始构建一个 Argo `Workflow` manifest:
    *   将 `ci.yaml` 中的 `jobs` 转换为 Argo DAG `templates`。
    *   将 `job.needs` 转换为 `template.dependencies`。
    *   对于每个 `job`，创建一个 `container` 模板。
    *   设置 `container.image` 为 `job.image`。
    *   设置 `container.command` 为 `["/bin/sh", "-c"]`，`args` 为 `job.run` 的脚本内容。
    *   查询 **Admin Config Service**，将 `job.runner` 对应的资源规格填充到 `container.resources`。
    *   将 `job.secrets` 转换为 `container.envFrom` 或 `env` 中的 `secretKeyRef`。
6.  **Generator** 使用 `kube-rs` 客户端将完整的 `Workflow` manifest 提交到 Kubernetes API Server。
7.  **Argo Workflows Controller** 检测到新的 `Workflow` 资源。
8.  Argo Controller 根据 `Workflow` 定义，开始调度并创建执行器 Pods。
9.  **Status Reporter (Rust)** 通过监听 `Workflow` 资源或接收 Argo 的回调，获取任务状态。
10. **Reporter** 将 `Succeeded` / `Failed` 状态回传给 Git 平台。

## 2. `ci.yaml` 到 Argo Workflow 的映射

这是本设计的核心。我们的 `Workflow Generator` 负责执行以下转换。

| `ci.yaml` 概念 | Argo Workflow 概念 | 示例 |
| :--- | :--- | :--- |
| `jobs` 集合 | 一个 `Workflow` 中的 DAG `template` | `spec.templates` 中类型为 `dag` 的模板 |
| 单个 `job` | DAG 中的一个 `task`，指向另一个执行 `template` | `dag.tasks` 中的一项 |
| `job.needs: [a, b]` | `task.dependencies: [a, b]` | `dependencies: [build, test]` |
| `job.image` | `container` 模板中的 `image` | `container: { image: "node:18" }` |
| `job.runner: large` | `container` 模板中的 `resources` | `resources: { requests: { cpu: "4", memory: "16Gi" } }` |
| `job.run: "..."` | `container` 模板中的 `script` (或 `command`/`args`) | `script: | echo "hello"` |
| `job.secrets` | `container` 模板中的 `env` 或 `envFrom` (引用 `Secret`) | `env: - name: MY_SECRET valueFrom: { secretKeyRef: ... }` |
| `on.schedule.cron` | 生成一个 `CronWorkflow` 资源而非 `Workflow` | `kind: CronWorkflow` |

**示例 `ci.yaml`:**
```yaml
jobs:
  build:
    runner: medium
    image: golang:1.19
    steps:
      - run: go build ./...
