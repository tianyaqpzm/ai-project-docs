# 问题复盘：ms-java-gateway Docker 容器健康检查失败 (unhealthy)

## 基本信息
- **日期**: 2026-04-27
- **问题描述**: `ms-java-gateway` 容器启动后状态显示为 `unhealthy`，导致在部分环境下请求被拦截或容器被重启。
- **涉及工程**: `ms-java-gateway`, `ms-java-biz`
- **故障原因**: 
    1. 缺少 `spring-boot-starter-actuator` 依赖，导致 `/actuator/health` 接口不存在。
    2. 运行镜像 `eclipse-temurin:17-jre` 中未安装 `curl`，导致 Dockerfile 中的 `HEALTHCHECK` 指令无法执行。

## 排查过程
1. **现象确认**: 使用 `docker ps` 观察到容器状态为 `Up 22 minutes (unhealthy)`。
2. **源码分析**: 检查 `Dockerfile` 发现 `HEALTHCHECK` 指令为 `curl -f http://localhost:8281/actuator/health || exit 1`。
3. **依赖检查**: 检查 `pom.xml` 发现未引入 Spring Boot Actuator 模块。
4. **镜像分析**: 确认 `eclipse-temurin:17-jre` 基础镜像不包含 `curl` 工具。

## 解决方案
1. **添加依赖**: 在 `ms-java-gateway` 和 `ms-java-biz` 的 `pom.xml` 中引入 `spring-boot-starter-actuator`。
2. **安装工具**: 修改 `Dockerfile`，在运行时阶段使用 `apt-get` 安装 `curl`。
3. **补充配置**: 在 `ms-java-biz` 的 `application.yaml` 中显式暴露 `health` 端点。
4. **安全加固**: 确认 `SecurityConfig` 中已放行 `/actuator/**` 路径，避免健康检查被拦截。

## 根因分析 (RCA)
- **开发疏忽**: 在将项目容器化并添加 `HEALTHCHECK` 指令时，未同步检查运行时环境（镜像）是否具备必要的工具（curl），且未确认应用本身是否开启了对应的监控端点。
- **模板不一致**: `ms-java-gateway` 和 `ms-java-biz` 在初始化时对监控模块的集成程度不一致。

## 经验总结
- **容器化检查**: 所有的 `HEALTHCHECK` 指令必须在对应的镜像中通过单元测试或手动验证，确保检测工具可用且接口可达。
- **基础依赖规范**: 所有的 Java 微服务必须默认集成 `spring-boot-starter-actuator` 以支持云原生环境下的存活（Liveness）和就绪（Readiness）检查。
- **镜像预装**: 若使用精简镜像，必须显式安装业务运维所需的工具（如 `curl`, `telnet` 等）。

---
**记录人**: Antigravity
**状态**: 已解决
