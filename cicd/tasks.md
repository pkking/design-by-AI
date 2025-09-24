
# 任务分解: GitOps CI/CD 后台系统 (Rust + Argo Workflows)

---

### Epic 1: 核心转换引擎 (MVP)

**目标:** 能够将一个基础的 `ci.yaml` 文件手动转换为一个有效的 Argo Workflow manifest 并通过 `kubectl` 成功执行。

*   **Task 1.1:** 设计并实现 `ci.yaml` v0.1 版本的 Rust 解析器 (使用 `serde`)。
    *   *要求:* 支持 `jobs`, `image`, `run`。
*   **Task 1.2:** 开发一个 Rust 模块，该模块能将解析后的 `ci.yaml` 结构体转换为 Argo `Workflow` 的 Rust 结构体 (使用 `k8s-openapi` 或自定义 `serde` 结构体)。
*   **Task 1.3:** 编写一个简单的 Rust 命令行工具，输入为 `ci.yaml` 文件路径，输出为标准的 Argo `Workflow` YAML。
*   **Task 1.4:** 部署 Argo Workflows 到开发 Kubernetes 集群。
*   **Task 1.5:** 手动测试：使用 Task 1.3 生成的 YAML，通过 `kubectl apply -f` 确认任务能被 Argo 成功执行。

---

### Epic 2: Git 平台集成与自动提交

**目标:** 系统能够通过 Webhook 接收 Git 事件，并自动生成 Argo Workflow 提交到集群。

*   **Task 2.1:** 开发 `Webhook Gateway` Rust 服务。
    *   *要求:* 提供一个公开的 HTTP 端点，能处理 JSON payload，并实现 Webhook Secret 验证。
*   **Task 2.2:** 集成消息队列 (如 NATS)，将验证通过的事件推送到队列中。
*   **Task 2.3:** 开发 `Workflow Generator` Rust 服务，能够消费队列事件。
*   **Task 2.4:** 在 `Workflow Generator` 中集成代码克隆和 `ci.yaml` 解析逻辑。
*   **Task 2.5:** 集成 `kube-rs` 客户端到 `Workflow Generator`，使其能够将生成的 `Workflow` manifest 提交到 Kubernetes。
*   **Task 2.6:** 配置 Kubernetes RBAC，为 `Workflow Generator` 的 ServiceAccount 授予创建 `argoproj.io/workflows` 资源的权限。
*   **Task 2.7:** 开发 `Status Reporter` Rust 服务，能够监听 `Workflow` 资源的状态变更，并调用 Git 平台 API 进行状态回传。

---

### Epic 3: 资源与秘钥管理

**目标:** 实现对物理资源别名和秘钥的安全管理与使用。

*   **Task 3.1:** 设计并实现 `Admin Config Service` (或基于 ConfigMap 的配置加载)，用于管理 `runner` 别名。
*   **Task 3.2:** 改造 `Workflow Generator`，使其在生成 manifest 时：
    *   调用 `Admin Config Service` 获取资源规格。
    *   将资源规格填充到 `container.resources` 字段。
*   **Task 3.3:** 设计管理 Kubernetes Secrets 的流程 (例如，通过一个安全的管理脚本或内部工具)。
*   **Task 3.4:** 改造 `Workflow Generator`，将 `ci.yaml` 中的 `secrets` 声明转换为 Argo `Workflow` 中对 Kubernetes `Secret` 的引用 (`secretKeyRef`)。
*   **Task 3.5:** 实现对无效 `runner` 或 `secret` 名称的错误处理，并通过 `Status Reporter` 回传失败状态。

---

### Epic 4: 高级工作流功能

**目标:** 支持任务依赖、定时触发等复杂场景。

*   **Task 4.1:** 扩展 `ci.yaml` 解析器和 `Workflow Generator`，支持 `needs` 关键字，并将其转换为 Argo DAG `dependencies`。
*   **Task 4.2:** 扩展 `ci.yaml` 解析器，支持 `on.schedule.cron` 语法。
*   **Task 4.3:** 改造 `Workflow Generator`，使其在检测到 `schedule` 触发时，生成 `CronWorkflow` 类型的 CRD 而不是 `Workflow`。
*   **Task 4.4:** 配置 `Workflow Generator` 的 ServiceAccount 以拥有创建 `CronWorkflow` 资源的权限。
*   **Task 4.5:** 实现对环境变量 (`env`) 的支持，将其直接映射到 `container.env` 字段。

---

### Epic 5: 系统部署与运维

**目标:** 实现整个系统的自动化部署、配置和监控。

*   **Task 5.1:** 为所有 Rust 服务编写 `Dockerfile` (推荐使用 multi-stage builds 减小镜像体积)。
*   **Task 5.2:** 编写 Helm Chart 用于一键部署所有 Rust 服务及其相关配置 (ConfigMaps, RBAC, etc.)。
*   **Task 5.3:** 编写或使用官方 Helm Chart 部署 Argo Workflows。
*   **Task 5.4:** (推荐) 创建一个 Argo CD Application，用于以 GitOps 的方式管理本 CI/CD 系统自身的部署。
*   **Task 5.5:** 引入日志和监控。集成 `tracing` 或 `log` crate 到 Rust 服务中，并通过 Prometheus 和 Grafana 进行监控。
