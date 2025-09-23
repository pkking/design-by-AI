
---

### `tasks.md`

```markdown
# 任务分解: GitOps CI/CD 后台系统

本文档将系统开发工作分解为多个史诗 (Epics) 和可执行的任务 (Tasks)。

---

### Epic 1: 核心执行引擎 (MVP)

**目标:** 能够在一个预配置的环境中，手动触发并成功执行一个最简单的 `ci.yaml` 文件。

*   **Task 1.1:** 设计并实现 `ci.yaml` v0.1 版本的解析器 (Parser)。
    *   *要求:* 支持 `jobs`, `image`, `steps`, `run`。
*   **Task 1.2:** 开发一个基础的 `Orchestrator`，能够读取解析后的任务计划。
*   **Task 1.3:** 开发一个基础的 `Scheduler`，能够接收任务并硬编码创建一个 Kubernetes Pod 来执行 `run` 命令。
*   **Task 1.4:** 实现 Pod 内的执行 Agent，负责按顺序执行 `steps` 并捕获日志和退出码。
*   **Task 1.5:** 建立一个基础的日志收集和存储机制，能够查看 Pod 的完整执行日志。

---

### Epic 2: Git 平台集成与 Webhook

**目标:** 系统能够接收来自 Git 平台的 Webhook，并根据事件自动触发工作流。

*   **Task 2.1:** 开发 `Webhook Gateway` 服务。
    *   *要求:* 提供一个公开的 HTTP 端点，能处理 JSON payload。
*   **Task 2.2:** 实现 Webhook Secret Token 的验证逻辑。
*   **Task 2.3:** 集成消息队列，将验证通过的合法事件推送到队列中。
*   **Task 2.4:** 改造 `Orchestrator` 以消费消息队列中的事件，而非手动触发。
*   **Task 2.5:** 实现代码克隆逻辑，能够根据 Webhook payload 中的信息拉取正确的仓库和 Commit。
*   **Task 2.6:** 开发 `Status Reporter` 服务，实现向 Git 平台（首先支持 GitHub）回传 `pending`, `success`, `failure` 状态。

---

### Epic 3: 资源管理 (Runner 别名)

**目标:** 实现用户通过别名选择物理资源，管理员可配置这些别名的功能。

*   **Task 3.1:** 设计 `runner` 别名的中心化配置文件格式。
*   **Task 3.2:** 开发 `Admin Config Service`，能够加载并提供 `runner` 配置。
*   **Task 3.3:** 扩展 `ci.yaml` 解析器，支持 `runner` 关键字。
*   **Task 3.4:** 改造 `Scheduler`，使其在创建 Pod 前调用 `Admin Config Service`，获取 CPU/Memory 规格并应用到 PodSpec 的 `resources` 字段中。
*   **Task 3.5:** 实现默认 `runner` 机制和无效 `runner` 名称的错误处理及状态回传。

---

### Epic 4: 安全与秘钥管理

**目标:** 实现一套安全的秘钥管理和注入机制。

*   **Task 4.1:** 集成一个安全的秘钥存储后端 (如 HashiCorp Vault)。
*   **Task 4.2:** 设计并开发一个安全的管理接口（API 或 CLI），供管理员添加/更新与仓库关联的秘钥。
*   **Task 4.3:** 扩展 `ci.yaml` 解析器，支持 `secrets` 关键字。
*   **Task 4.4:** 改造 `Scheduler`，在创建 Pod 时，从 `Secret Store` 中获取秘钥引用，并配置 PodSpec 以安全地将秘钥作为环境变量注入到容器中。
*   **Task 4.5:** 确保在日志输出中自动屏蔽秘钥值。

---

### Epic 5: 增强功能与易用性

**目标:** 增加环境变量、任务依赖和定时触发等高级功能。

*   **Task 5.1:** 扩展 `ci.yaml` 解析器，支持 `env` 关键字，并实现环境变量的注入。
*   **Task 5.2:** 扩展 `ci.yaml` 解析器，支持 `needs` 关键字。
*   **Task 5.3:** 改造 `Orchestrator`，使其能够理解任务间的依赖关系，并按正确的顺序调度任务。
*   **Task 5.4:** 扩展 `ci.yaml` 解析器，支持 `on.schedule.cron` 语法。
*   **Task 5.5:** 开发一个定时任务触发器 (Cron Job)，能够定期生成事件并推送到消息队列，以触发定时工作流。

---

### Epic 6: 系统运维与部署

**目标:** 确保系统可以被轻松地部署和维护。

*   **Task 6.1:** 为所有服务编写 Dockerfile。
*   **Task 6.2:** 编写 Kubernetes 部署清单 (Deployments, Services, etc.) 或 Helm Chart。
*   **Task 6.3:** 建立监控和告警系统 (e.g., Prometheus + Alertmanager)。
*   **Task 6.4:** 编写管理员手册，说明如何部署系统、配置 `runner` 和 `secret`。