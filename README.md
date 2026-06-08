# Claude Code 可观测性快速搭建

一套开箱即用的 Docker Compose 栈，用于采集、存储和可视化 Claude Code 的
OpenTelemetry 遥测数据（metrics + logs）。

```
Claude Code ──OTLP/http──▶ OTel Collector ──┬─ metrics ─▶ Prometheus ──┐
                          (4318)            └─ logs ────▶ Loki ────────┴─▶ Grafana
```

## 组件

| 服务 | 镜像 | 宿主机端口 | 作用 |
| --- | --- | --- | --- |
| OTel Collector | `otel/opentelemetry-collector-contrib:0.118.0` | 4318 | 统一 OTLP 接收端，分流 metrics / logs |
| Prometheus | `prom/prometheus:v3.5.0` | 9090 | 存储 metrics（原生 OTLP receiver，保留 30d） |
| Loki | `grafana/loki:3.3.2` | 3200 → 容器 3100 | 存储 logs（原生 OTLP receiver） |
| Grafana | `grafana/grafana:12.2.0` | 3000 | 仪表盘可视化，数据源已自动注入 |

> Claude 2.1 的 per-signal endpoint split 实测不稳定，因此让 Claude 把所有信号
> 推到统一端点 4318，由 Collector 负责分流。

## 快速开始

### 1. 启动栈

```bash
docker compose up -d
```

打开 Grafana：<http://localhost:3000>（账号 `admin` / 密码 `admin`，已开启匿名 Viewer）。
仪表盘在 **Claude Code** 文件夹下自动加载。

### 2. 让 Claude Code 上报遥测

在运行 Claude Code 的 shell 中设置环境变量，指向本地 Collector：

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318

# 可选：附带可用于关联/分组的 resource 属性
export OTEL_RESOURCE_ATTRIBUTES="team=my-team,process.pid=$$"
```

随后正常使用 Claude Code，数据会持续流入 Grafana。

## 仪表盘

`grafana/provisioning/dashboards/` 下已内置：

- **Overview** — 总览
- **Sessions** / **Sessions & Prompts** — 会话与提示词
- **Prompt Detail** — 提示词明细
- **Agents (Subagents)** — 子代理
- **Skills** — 技能调用
- **MCP Servers** — MCP 服务
- **Cost & Efficiency** — 成本与效率
- **Team Analytics** — 团队分析

## 设计要点

- **高基数控制**：`process.pid`、`session.id` 在 metrics 管道中被删除（防止
  Prometheus series 爆炸），logs 管道保留它们用于关联。详见
  `otel-collector-config.yaml`。
- **Loki 编码**：logs 用 `http+json` 导出 —— 实测 protobuf 在 Loki 3.3 会被静默丢弃。
- **Prometheus 命名**：使用默认 `UnderscoreEscapingWithSuffixes`，OTel 名中的
  `.` → `_`，counter 自动加 `_total`，带单位的 metric 自动加单位后缀
  （如 `token.usage` → `token_usage_tokens_total`）。
- **端口让位**：Loki 容器内 3100，映射到宿主机 3200，避免与其他服务冲突。

## 停止与清理

```bash
docker compose down          # 停止
docker compose down -v       # 停止并删除数据卷（prom_data / loki_data / grafana_data）
```

## 配置文件

| 文件 | 说明 |
| --- | --- |
| `docker-compose.yml` | 服务编排 |
| `otel-collector-config.yaml` | Collector 接收 / 处理 / 导出管道 |
| `prometheus.yml` | Prometheus OTLP 接收与命名策略 |
| `loki-config.yaml` | Loki 单体配置 |
| `grafana/provisioning/` | Grafana 数据源与仪表盘自动注入 |
