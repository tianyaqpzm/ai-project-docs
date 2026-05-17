# 事故复盘 (RCA): MCP-PG 插件连接云端 PostgreSQL 失败

**日期**: 2026-05-16
**问题点**: MCP-PG 插件无法连接至 Aiven Cloud 托管的 PostgreSQL 数据库
**状态**: 已解决 (教科书级排查案例)

---

## 1. 现象描述
在 `docker-compose` 环境中部署 `mcp-pg` 插件时，频繁出现 `self-signed certificate in certificate chain` 或连接被拒绝的情况。尽管使用了 `?sslmode=disable` 尝试降级，但在 Aiven 等严格的云数据库面前依然无法建立连接。

## 2. 根因分析 (教训总结)

### A. 跨进程参数传递的“绞肉机”陷阱 (CLI Args vs. ENV Vars)
试图通过命令行附加参数传递复杂的 `DATABASE_URL` 是一场灾难。特殊字符（`@`、`?`、`&`）在 `docker-compose` -> `mcp-proxy` -> `postgres-mcp-server` 的多层嵌套中，极易被 Shell 转义或截断。
**正解**: **环境变量是穿透隔离层最安全、最防呆的机制**。使用 `DB_HOST`、`DB_USER` 等独立的环境变量，彻底避开命令行字符转义雷区。

### B. Node.js TLS 机制与底层驱动的安全博弈
粗暴地注入 `NODE_TLS_REJECT_UNAUTHORIZED=0` 往往失效。成熟的数据库驱动（如 `pg`）在面对云端数据库时，其内部安全策略会强制接管 TLS 握手。
**正解**: **不要尝试绕过安全机制，而是去满足它**。通过 `NODE_EXTRA_CA_CERTS` 注入 CA 证书是 Node.js 处理自定义或严格证书的唯一标准解法。

### C. 云原生数据库的“零容忍”安全底线
Aiven 等托管实例在网关层面物理阻断了任何非安全握手。必须放弃本地测试思维，老老实实配置 `verify-ca` 模式。

---

## 3. 最终成功配置 (Golden Template)

以下配置具备企业级的工程鲁棒性，完美处理了凭证安全、网络穿透和 TLS 证书链校验。

```yaml
services:
  mcp-pg:
    build: .
    container_name: mcp-pg-service
    restart: always
    ports:
      - "6277:6277"
    volumes:
      # 将宿主机的 ca.pem 挂载到容器内部
      - ./ca.pem:/app/ca.pem:ro
    environment:
      DB_HOST: "pg-aiven-dark.c.aivencloud.com"
      DB_PORT: "15228"
      DB_USER: "postgres_user"
      DB_PASSWORD: "YourSecurePassword"
      DB_NAME: "postgres"
      DB_SSL: "true"
      PGSSLMODE: "verify-ca"  # 严格验证 CA 证书
      NODE_EXTRA_CA_CERTS: "/app/ca.pem" # 告诉 Node.js 信任这个自定义 CA
    entrypoint:
      - "mcp-proxy"
      - "--host"
      - "0.0.0.0"
      - "--port"
      - "6277"
      - "--"
      - "postgres-mcp-server"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    networks:
      - pei-network

networks:
  pei-network:
    external: true
```

---

## 4. 经验教训与改进建议
1. **诊断客观性**: 系统行为和日志永远比理论推断更真实。在处理开源组件时，必须依赖确凿的源码或测试反馈，摒弃臆测。
2. **标准模板化**: 此配置应作为未来所有微服务连接公有云数据库的标准架构模板。
3. **监控前置**: 在插件初始化阶段增加对证书文件可读性的检查日志。
