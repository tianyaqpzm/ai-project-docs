# 问题复盘：2026-04-28 - 服务更名导致的网关路由失效

## 问题描述
API 网关在尝试将请求路由到 Python Agent 时报错 `500`，并记录日志 `No servers available for service: python-agent`。

## 根因分析
1. **服务更名**：Python Agent 服务在代码和网关配置中已从 `python-agent` 更名为 `ms-py-agent`。
2. **陈旧部署**：VPS 上仍运行着名为 `python-agent` 的旧 Docker 容器，可能导致其继续在 Nacos 中注册旧名称，或产生端口冲突。
3. **CI/CD 配置缺陷**：Docker 部署工作流仅停止并删除了名为 `ms-py-agent` 的容器，导致旧的 `python-agent` 容器在多次部署后依然存活。
4. **镜像堆积**：旧镜像（如 `python-agent:pr-1`）未被清理，因为 `docker image prune -f` 仅删除无标签（dangling）镜像。

## 解决方案
1. **增强日志可见性**：将网关和 Python Agent 中间件的日志提升至 `INFO` 级别，以捕获完整的请求-响应周期。
2. **CI/CD 工作流同步**：
   - 更新了 `ms-java-biz` 和 `ms-py-agent` 的工作流，使其与 `ms-java-gateway` 的逻辑一致。
   - 开启了向 master 分支发起 `feature_*` PR 时的自动部署。
   - 重构了标签处理逻辑，动态拉取 `pr-N` 或 `master` 镜像。
3. **Docker 卫生管理**：
   - 切换到 `docker image prune -af` 以确保彻底删除所有未使用的镜像。
   - （建议手动步骤）手动停止并删除 VPS 上的遗留 `python-agent` 容器。

## 经验教训
- **更名时的清理逻辑**：在重命名微服务或其容器名称时，必须在部署脚本中明确增加一次对旧名称的清理逻辑。
- **强制镜像清理**：使用 `-af` 进行镜像清理是防止旧 PR 镜像耗尽磁盘空间的必要手段。
- **日志级别规范**：关键的路由事件和认证通过信息应保持在 `INFO` 级别，以便在生产类环境中快速定位问题。
