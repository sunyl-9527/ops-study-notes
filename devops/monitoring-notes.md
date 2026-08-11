# 监控与告警实战笔记
> **适用对象**：已经掌握 Docker、Docker Compose 基础，并了解常见中间件的运维学习者。  
> **定位**：对应 `cloud-ops-roadmap.md` 第五阶段“监控、日志与排障”的动手实践，与 `cloud-ops-middleware-notes.md` 第十章互补。
本文以一套可运行的容器监控栈为主线，完成“采集、查询、展示、告警、通知、排障”的基础闭环。

## 目录
1. [监控体系与可观测性](#1-监控体系与可观测性)
2. [Prometheus 架构](#2-prometheus-架构)
3. [使用 Docker Compose 搭建监控栈](#3-使用-docker-compose-搭建监控栈)
4. [PromQL 入门](#4-promql-入门)
5. [常用 Exporter](#5-常用-exporter)
6. [Grafana 使用](#6-grafana-使用)
7. [告警规则](#7-告警规则)
8. [Alertmanager](#8-alertmanager)
9. [日志方案简介](#9-日志方案简介)
10. [告警设计原则](#10-告警设计原则)
11. [常见问题排查](#11-常见问题排查)
12. [学习路线与实践任务](#12-学习路线与实践任务)
13. [总结](#13-总结)

---

## 1. 监控体系与可观测性
### 1.1 监控要回答什么
- 系统现在是否可用，用户是否遇到错误或延迟？
- 哪个组件最可能是故障点，异常从何时开始？
- 资源是否即将耗尽，一次发布是否引入性能退化？
- 当前容量还能支持多少增长？
“进程存活”不等于“服务健康”。进程可能仍在运行，却因数据库连接池耗尽而持续返回错误。
### 1.2 监控的四个层次
| 层次 | 对象 | 典型指标 | 回答的问题 |
|---|---|---|---|
| 基础设施 | 主机、虚拟机、容器、网络 | CPU、内存、磁盘、网络、负载 | 资源是否异常 |
| 服务 | Nginx、应用、数据库、缓存 | 请求量、错误率、延迟、连接数 | 服务是否健康 |
| 业务 | 登录、下单、支付、任务 | 成功率、业务量、失败量、耗时 | 业务是否完成 |
| 用户体验 | 浏览器、移动端、外部探测点 | 可用率、页面加载、地域延迟 | 用户感受如何 |
建设常从底层向上推进，告警优先级却应先看用户和业务影响。
### 1.3 可观测性三支柱
| 类型 | 特点 | 适合的问题 | 常见工具 |
|---|---|---|---|
| 指标 | 数值化、易聚合、成本较低 | 是否异常、何时开始、范围多大 | Prometheus、Grafana |
| 日志 | 事件细节丰富、可检索 | 报错是什么、当时发生了什么 | journald、Loki、Elasticsearch |
| 链路 | 描述跨服务调用路径 | 慢在哪一段、哪个下游失败 | OpenTelemetry、Jaeger、Tempo |
典型排障流程是：告警发现异常，指标确定实例和时间，日志寻找错误，链路定位跨服务慢点。
### 1.4 USE 方法
USE 方法用于检查每一种基础资源：
- **使用率**：资源忙碌程度，如 CPU 使用率、磁盘空间使用率。
- **饱和度**：是否出现排队，如运行队列、磁盘等待队列。
- **错误数**：操作是否失败，如网卡丢包、磁盘错误。
| 资源 | 使用率 | 饱和度 | 错误 |
|---|---|---|---|
| CPU | 非空闲时间 | 运行队列 | 硬件异常 |
| 内存 | 已用内存 | 换页、交换活动 | 分配失败、进程被杀 |
| 磁盘 | 空间或输入输出利用率 | 等待、队列长度 | 文件系统、设备错误 |
| 网络 | 带宽使用率 | 连接积压 | 丢包、重传、接口错误 |
### 1.5 RED 方法
RED 方法用于面向请求的服务：
- **请求速率**：每秒请求数。
- **错误数**：每秒错误数或错误率。
- **请求耗时**：平均值和 95、99 分位延迟。
RED 判断用户影响，USE 寻找资源瓶颈，两者应结合使用。

---

## 2. Prometheus 架构
### 2.1 核心组件
- **Prometheus Server**：抓取指标、保存时序数据、执行 PromQL 和告警规则。
- **Exporter**：把操作系统或中间件状态转换为 Prometheus 指标。
- **Alertmanager**：完成告警分组、路由、抑制、静默和通知。
- **Grafana**：查询数据并展示仪表盘，不负责采集。
- **Pushgateway**：接收短生命周期批处理任务推送的指标，不替代常规抓取。
- **服务发现**：动态提供抓取目标，如 Kubernetes、云平台或文件列表。
### 2.2 拉取模型
Prometheus 按固定间隔主动请求目标的 `/metrics`：
- 采集频率由监控端统一控制。
- 抓取失败可直接说明目标不可达。
- Exporter 无需保存监控系统凭据。
- Prometheus 必须能访问目标端口，短任务则需特殊处理。
### 2.3 时序数据
```text
node_cpu_seconds_total{instance="node-exporter:9100",job="node",cpu="0",mode="idle"}
```
- 指标名与整组标签唯一确定一条时间序列。
- 每个样本包含时间戳和值。
- 标签值变化会生成新序列。
- 实例、环境、状态码适合做标签；用户编号、订单编号、请求编号会造成高基数，应避免。
### 2.4 文本架构图
```text
                              PromQL 查询
+-----------+             +----------------------+             +----------+
| Grafana   |------------>| Prometheus Server    |------------>| 时序存储 |
| 仪表盘    |             | 抓取、查询、规则计算 |             +----------+
+-----------+             +----------+-----------+
                                     |
                               拉取 /metrics
                 +-------------------+-------------------+
                 |                   |                   |
                 v                   v                   v
          +-------------+     +--------------+     +-------------+
          |node_exporter|     |MySQL Exporter|     |应用指标接口 |
          +-------------+     +--------------+     +-------------+
                                     |
                               规则满足条件
                                     v
                             +---------------+
                             | Alertmanager  |
                             | 路由与通知    |
                             +------+--------+
                                    |
                              邮件、Webhook
```
一次采集过程：读取配置，生成目标，定时抓取，附加标签，写入时序库，执行查询与规则，把满足条件的告警发给 Alertmanager。

---

## 3. 使用 Docker Compose 搭建监控栈
### 3.1 环境与目录
建议使用 Linux 主机，至少 2 核 CPU、4 GB 内存和 10 GB 可用磁盘。
```bash
docker version
docker compose version
mkdir -p monitoring-lab/{prometheus,alertmanager}
cd monitoring-lab
```
```text
monitoring-lab/
├── compose.yaml
├── prometheus/
│   ├── prometheus.yml
│   └── alert-rules.yml
└── alertmanager/
    └── alertmanager.yml
```
### 3.2 完整 compose.yaml
`latest` 方便实验获取当前镜像；生产环境应固定经过验证的版本。Grafana 密码必须替换。
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=15d
      - --web.enable-lifecycle
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alert-rules.yml:/etc/prometheus/alert-rules.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - monitoring
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: "<请替换为强密码>"
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - grafana-data:/var/lib/grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    networks:
      - monitoring
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --path.rootfs=/rootfs
      - --collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro,rslave
    ports:
      - "9100:9100"
    networks:
      - monitoring
  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    command:
      - --config.file=/etc/alertmanager/alertmanager.yml
      - --storage.path=/alertmanager
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager-data:/alertmanager
    ports:
      - "9093:9093"
    networks:
      - monitoring
networks:
  monitoring:
    name: monitoring
volumes:
  prometheus-data:
  grafana-data:
  alertmanager-data:
```
配置说明：
- 三个命名卷分别保存时序数据、Grafana 配置和 Alertmanager 状态。
- Prometheus 保留 15 天数据，并开启 HTTP 重载接口。
- node_exporter 以只读方式读取宿主机 `/proc`、`/sys` 和根文件系统。
- `rslave` 让容器看到宿主机后续挂载点，仅适用于 Linux。
- 组件通过 `monitoring` 网络和服务名互访；生产环境不要把管理端口直接暴露到公网。
### 3.3 完整 prometheus.yml
创建 `prometheus/prometheus.yml`：
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s
  external_labels:
    environment: lab
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
rule_files:
  - /etc/prometheus/alert-rules.yml
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets:
          - prometheus:9090
        labels:
          service: prometheus
  - job_name: node
    static_configs:
      - targets:
          - node-exporter:9100
        labels:
          service: linux-host
```
分段解释：
- `global`：每 15 秒抓取和计算规则，单次抓取最多 10 秒，并附加实验环境标签。
- `alerting`：声明 Alertmanager 地址；Prometheus 判断条件，Alertmanager 决定通知方式。
- `rule_files`：把告警规则拆到独立文件，便于维护。
- `scrape_configs`：定义 Prometheus 自监控和主机监控任务。
- `job_name` 成为 `job` 标签；`targets` 只写“主机:端口”，不写协议。
### 3.4 初始规则与 Alertmanager 配置
创建 `prometheus/alert-rules.yml`：
```yaml
groups:
  - name: 基础存活检查
    rules:
      - alert: TargetDown
        expr: up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "目标 {{ $labels.instance }} 不可用"
          description: "任务 {{ $labels.job }} 已连续 2 分钟抓取失败。"
```
创建 `alertmanager/alertmanager.yml`：
```yaml
route:
  receiver: default
  group_by:
    - alertname
    - job
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
receivers:
  - name: default
```
此配置可接收告警但不向外通知，第 8 节再加入邮件和 Webhook。
### 3.5 启动与验证
```bash
docker compose config
docker compose pull
docker compose up -d
docker compose ps
curl http://localhost:9100/metrics
curl http://localhost:9090/-/healthy
curl http://localhost:9093/-/healthy
```
访问地址：Grafana `3000`、Prometheus `9090`、Alertmanager `9093`、node_exporter `9100/metrics`。在 Prometheus 的“状态 -> 目标”确认两个任务正常。
修改配置后先检查再重载：
```bash
docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml
docker compose exec prometheus promtool check rules /etc/prometheus/alert-rules.yml
docker compose exec alertmanager amtool check-config /etc/alertmanager/alertmanager.yml
curl -X POST http://localhost:9090/-/reload
docker compose restart alertmanager
```

---

## 4. PromQL 入门
### 4.1 指标类型
- **Counter**：只增不减，进程重启归零，如请求总数、错误总数、CPU 累计时间；常配合 `rate()`、`increase()`。
- **Gauge**：可增可减，如温度、可用内存、队列长度、连接数。
- **Histogram**：用 `_bucket`、`_sum`、`_count` 记录分布，可用 `histogram_quantile()` 估算分位延迟。
瞬时向量表示某时刻的一组序列，如 `up`；范围向量表示一段时间的样本，如 `up[5m]`。
### 4.2 常用函数
```promql
rate(node_network_receive_bytes_total[5m])
```
`rate()` 计算计数器每秒平均增长率，并处理重置。15 秒抓取间隔通常使用 2 至 5 分钟窗口。
```promql
increase(node_network_receive_bytes_total[1h])
```
`increase()` 估算窗口内累计增量，适合“过去一小时共发生多少”。
```promql
avg_over_time(node_load1[15m])
```
`avg_over_time()` 计算 Gauge 在窗口内的平均值，用于平滑波动。
### 4.3 实用查询示例
**1. 服务存活：**
```promql
up
```
`1` 表示最近一次抓取成功，`0` 表示失败；抓取成功不等于业务功能完全健康。
**2. 按任务统计正常目标：**
```promql
sum by (job) (up)
```
**3. CPU 总使用率：**
```promql
100 - avg by (instance) (
  rate(node_cpu_seconds_total{mode="idle"}[5m])
) * 100
```
先求各核心空闲时间增长率的平均值，再用 100 减去空闲百分比。
**4. CPU 输入输出等待占比：**
```promql
avg by (instance) (
  rate(node_cpu_seconds_total{mode="iowait"}[5m])
) * 100
```
持续升高可能表示存储拥塞。
**5. 内存使用率：**
```promql
100 * (
  1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
)
```
Linux 应优先使用 `MemAvailable`，缓存较多不等于内存耗尽。
**6. 可用内存，单位 GiB：**
```promql
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```
**7. 根文件系统使用率：**
```promql
100 * (
  1 - node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs|overlay"}
  / node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs|overlay"}
)
```
`avail` 更接近普通用户可用空间，过滤临时文件系统可减少无效结果。
**8. 预测文件系统 24 小时内耗尽：**
```promql
predict_linear(
  node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[6h],
  24 * 3600
) < 0
```
预测受突发写入和日志清理影响，需与使用率共同判断。
**9. 磁盘读取速率：**
```promql
sum by (instance) (
  rate(node_disk_read_bytes_total[5m])
)
```
**10. 磁盘写入速率：**
```promql
sum by (instance) (
  rate(node_disk_written_bytes_total[5m])
)
```
**11. 网络接收速率：**
```promql
sum by (instance) (
  rate(node_network_receive_bytes_total{device!~"lo|veth.*|docker.*"}[5m])
)
```
**12. 网络发送速率：**
```promql
sum by (instance) (
  rate(node_network_transmit_bytes_total{device!~"lo|veth.*|docker.*"}[5m])
)
```
**13. 网卡接收错误速率：**
```promql
sum by (instance, device) (
  rate(node_network_receive_errs_total{device!="lo"}[5m])
)
```
持续非零时检查链路、驱动和虚拟网络。
**14. 15 分钟负载与 CPU 核心比值：**
```promql
node_load15
/
count by (instance) (node_cpu_seconds_total{mode="idle"})
```
持续大于 1 表示每核心对应的运行或不可中断任务较多，还需结合 CPU 与输入输出等待。
**15. 过去一小时网络接收总量：**
```promql
sum by (instance) (
  increase(node_network_receive_bytes_total{device!="lo"}[1h])
)
```
**16. 过去 15 分钟平均可用内存：**
```promql
avg_over_time(node_memory_MemAvailable_bytes[15m])
```
标签过滤与聚合示例：
```promql
up{job="node"}
up{job=~"node|prometheus"}
node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs"}
sum by (job) (up)
```
聚合时应保留排障需要的 `instance` 等维度。

---

## 5. 常用 Exporter
| Exporter | 采集内容 | 常用端口 | 部署要点 |
|---|---|---:|---|
| node_exporter | Linux CPU、内存、磁盘、文件系统、网络 | 9100 | 每台主机一个；正确挂载宿主机目录；限制端口来源 |
| mysqld_exporter | MySQL 连接、查询、InnoDB、复制 | 9104 | 使用最小权限监控账号；凭据放入密钥或受限文件 |
| redis_exporter | Redis 内存、命令、键空间、持久化、复制 | 9121 | 保护认证信息；区分单机、哨兵、集群模式 |
| blackbox_exporter | HTTP、TCP、ICMP、DNS、证书探测 | 9115 | Prometheus 传入探测目标；适合外部可用性视角 |
| cAdvisor | 容器 CPU、内存、网络、文件系统 | 8080 | 需访问宿主机与运行时；控制权限和标签基数 |
### 5.1 部署关注点
- node_exporter 只暴露指标，不保存数据；文本文件收集器可暴露备份时间、证书期限等自定义指标。
- mysqld_exporter 重点观察连接数、慢查询、缓冲池、锁、复制延迟，禁止使用管理员账号。
- redis_exporter 重点观察内存、碎片率、命中率、拒绝连接、淘汰键、持久化和复制。
- blackbox_exporter 从用户入口探测状态码、DNS、端口和 TLS，与应用内部指标互补。
- cAdvisor 适合单机容器学习；容器多时要评估时间序列规模和是否已有平台指标。

---

## 6. Grafana 使用
### 6.1 添加数据源
1. 登录 `http://主机地址:3000`，使用 Compose 中配置的账号密码。
2. 打开“连接 -> 数据源”，选择 Prometheus。
3. 地址填写 `http://prometheus:9090`，保存并测试。
容器内的 `localhost:9090` 指向 Grafana 自己，必须使用 Compose 服务名。
### 6.2 导入社区仪表盘
Node Exporter Full 的常用编号为 `1860`：
1. 打开“仪表盘 -> 新建 -> 导入”。
2. 输入 `1860`，加载定义。
3. 选择 Prometheus 数据源并导入。
4. 选择正确的 `job` 和 `instance`。
社区仪表盘用于快速学习，生产使用前要检查查询是否匹配当前 Exporter、文件系统和网卡名称。
### 6.3 自建面板
以 CPU 使用率为例：
1. 新建仪表盘并添加时间序列面板。
2. 选择 Prometheus，输入第 4 节 CPU 查询。
3. 单位设为百分比，范围设为 0 至 100。
4. 图例写 `{{instance}}`，按需设置 70、85 阈值。
5. 标题写清对象与单位，并保存。
时间序列适合趋势，单值适合当前状态，表格适合多实例，热力图适合延迟分布。不要在一页堆放过多小图。
### 6.4 变量与模板
创建实例变量：
- 名称：`instance`
- 类型：查询
- 查询：`label_values(up{job="node"}, instance)`
- 多选与“全部”：按需启用
面板查询：
```promql
100 - avg by (instance) (
  rate(node_cpu_seconds_total{instance=~"$instance",mode="idle"}[5m])
) * 100
```
多选变量应使用正则匹配 `=~`。看板第一行优先展示可用率、错误率和延迟，第二行再展示资源原因。

---

## 7. 告警规则
### 7.1 语法
- `groups`、`name`、`rules`：组织规则组和规则。
- `alert`：稳定、清晰的告警名。
- `expr`：返回异常序列的 PromQL。
- `for`：条件持续多久才触发。
- `labels`：等级、团队、服务等路由标签。
- `annotations`：摘要、描述和处理手册地址。
### 7.2 完整规则示例
将 `prometheus/alert-rules.yml` 替换为：
```yaml
groups:
  - name: 主机基础告警
    interval: 30s
    rules:
      - alert: InstanceDown
        expr: up{job="node"} == 0
        for: 2m
        labels:
          severity: critical
          team: ops
        annotations:
          summary: "实例 {{ $labels.instance }} 无法采集"
          description: "已连续 2 分钟无法采集，请检查主机、网络、容器和端口。"
          runbook_url: "<请替换为实例宕机处理手册地址>"
      - alert: HostHighCpuUsage
        expr: |
          100 - avg by (instance) (
            rate(node_cpu_seconds_total{job="node",mode="idle"}[5m])
          ) * 100 > 85
        for: 10m
        labels:
          severity: warning
          team: ops
        annotations:
          summary: "实例 {{ $labels.instance }} CPU 使用率高"
          description: "已连续 10 分钟高于 85%，当前为 {{ printf \"%.2f\" $value }}%。"
          runbook_url: "<请替换为CPU处理手册地址>"
      - alert: HostHighMemoryUsage
        expr: |
          100 * (
            1 - node_memory_MemAvailable_bytes{job="node"}
            / node_memory_MemTotal_bytes{job="node"}
          ) > 90
        for: 10m
        labels:
          severity: warning
          team: ops
        annotations:
          summary: "实例 {{ $labels.instance }} 可用内存不足"
          description: "已连续 10 分钟高于 90%，当前为 {{ printf \"%.2f\" $value }}%。"
          runbook_url: "<请替换为内存处理手册地址>"
      - alert: FilesystemSpaceLow
        expr: |
          100 * (
            1 - node_filesystem_avail_bytes{
              job="node",
              fstype!~"tmpfs|overlay|squashfs"
            }
            / node_filesystem_size_bytes{
              job="node",
              fstype!~"tmpfs|overlay|squashfs"
            }
          ) > 85
        for: 15m
        labels:
          severity: warning
          team: ops
        annotations:
          summary: "实例 {{ $labels.instance }} 文件系统空间不足"
          description: "挂载点 {{ $labels.mountpoint }} 已连续 15 分钟高于 85%，当前为 {{ printf \"%.2f\" $value }}%。"
          runbook_url: "<请替换为磁盘处理手册地址>"
```
### 7.3 持续时间与等级
`for: 10m` 表示表达式连续满足 10 分钟才触发，不是每 10 分钟检查一次。状态依次为未触发、等待中、已触发、恢复。较长持续时间可过滤尖峰，也会延迟通知。
| 等级 | 含义 | 方式 | 示例 |
|---|---|---|---|
| 严重 | 用户已受影响或核心能力不可用 | 电话、即时消息、值班系统 | 核心服务不可用 |
| 警告 | 风险正在形成，需要尽快处理 | 即时消息、邮件 | 磁盘高、持续高负载 |
| 信息 | 低风险状态变化 | 邮件、工单或记录 | 较长周期容量提示 |
等级必须对应响应动作，而不是只根据数值高低机械划分。
```bash
docker compose exec prometheus promtool check rules /etc/prometheus/alert-rules.yml
curl -X POST http://localhost:9090/-/reload
```

---

## 8. Alertmanager
### 8.1 路由、分组、抑制与静默
- **路由树**：根路由接收全部告警，子路由按 `severity`、`team`、`service` 等标签选择接收器。
- **分组**：把同一故障产生的多条告警合并通知，降低噪声。
- **抑制**：根因告警存在时自动隐藏相关次要告警。
- **静默**：维护期间按标签临时停止通知；必须写原因和过期时间，告警仍会计算。
### 8.2 邮件和 Webhook 示例
将 `alertmanager/alertmanager.yml` 替换为以下内容，所有凭据均为占位符：
```yaml
global:
  resolve_timeout: 5m
  smtp_smarthost: "<SMTP服务器地址>:587"
  smtp_from: "<告警发件邮箱>"
  smtp_auth_username: "<SMTP用户名>"
  smtp_auth_password: "<SMTP密码或授权码>"
  smtp_require_tls: true
route:
  receiver: default-webhook
  group_by:
    - alertname
    - job
    - instance
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers:
        - severity="critical"
      receiver: critical-email-and-webhook
      group_wait: 10s
      repeat_interval: 30m
    - matchers:
        - severity="warning"
      receiver: warning-email
      repeat_interval: 4h
inhibit_rules:
  - source_matchers:
      - severity="critical"
    target_matchers:
      - severity="warning"
    equal:
      - alertname
      - instance
receivers:
  - name: default-webhook
    webhook_configs:
      - url: "<Webhook接收地址>"
        send_resolved: true
  - name: critical-email-and-webhook
    email_configs:
      - to: "<严重告警收件邮箱>"
        send_resolved: true
        headers:
          subject: "[严重告警] {{ .CommonLabels.alertname }}"
    webhook_configs:
      - url: "<Webhook接收地址>"
        send_resolved: true
  - name: warning-email
    email_configs:
      - to: "<警告告警收件邮箱>"
        send_resolved: true
        headers:
          subject: "[警告] {{ .CommonLabels.alertname }}"
```
参数含义：
- `group_wait`：首次通知前等待，以收集同批告警。
- `group_interval`：同组内容变化后的最小通知间隔。
- `repeat_interval`：未恢复告警的重复提醒间隔。
- `send_resolved`：发送恢复通知，形成闭环。
- 严重告警同时发邮件与 Webhook，警告只发邮件。
真实密码不能提交到 Git，应使用密钥系统或仅管理员可读的外部文件。
```bash
docker compose exec alertmanager amtool check-config /etc/alertmanager/alertmanager.yml
docker compose restart alertmanager
docker compose exec alertmanager amtool alert query
```
测试实例宕机告警：
```bash
docker compose stop node-exporter
# 等待超过规则持续时间，观察触发和通知
docker compose start node-exporter
```

---

## 9. 日志方案简介
### 9.1 journald 基础
```bash
journalctl -u docker.service
journalctl -u docker.service -f
journalctl --since "1 hour ago"
journalctl -p err
journalctl -b
```
分别用于查看服务日志、持续跟踪、限定时间、只看错误和查看本次启动日志。还应配置持久化、空间上限和轮转，避免重启丢日志或占满系统盘。
### 9.2 Loki 与 Promtail 轻量方案
Loki 主要索引标签，Promtail 读取日志并发送给 Loki，资源消耗通常低于完整 ELK。Promtail 已进入长期支持阶段，新建生产平台应评估 Grafana Alloy；此处只用于理解采集流程。
在 `compose.yaml` 的 `services` 下追加：
```yaml
  loki:
    image: grafana/loki:latest
    container_name: loki
    restart: unless-stopped
    command:
      - -config.file=/etc/loki/local-config.yaml
    ports:
      - "3100:3100"
    networks:
      - monitoring
  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    restart: unless-stopped
    command:
      - -config.file=/etc/promtail/config.yml
    volumes:
      - /var/log:/var/log:ro
      - ./promtail/config.yml:/etc/promtail/config.yml:ro
    depends_on:
      - loki
    networks:
      - monitoring
```
创建 `promtail/config.yml`：
```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0
positions:
  filename: /tmp/positions.yaml
clients:
  - url: http://loki:3100/loki/api/v1/push
scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: system-logs
          host: lab-host
          __path__: /var/log/*.log
```
Grafana 中添加 Loki 数据源，地址为 `http://loki:3100`。请求编号等随机值不应作为日志标签。
### 9.3 Loki 与 ELK 对比
| 项目 | Loki | ELK |
|---|---|---|
| 组件 | Loki、采集器、Grafana | Elasticsearch、采集器、Kibana |
| 资源 | 通常较低 | 通常较高，需规划堆内存与分片 |
| 索引 | 主要索引标签 | 强大全文和字段索引 |
| 查询 | 适合先按标签缩小范围 | 适合复杂全文检索与聚合 |
| 运维 | 相对简单 | 需管理节点、分片、副本、生命周期 |
| 场景 | 中小规模、已有 Grafana | 全文检索和分析要求高 |
选型前先明确每天日志量、保留天数、查询模式、合规要求和维护能力。

---

## 10. 告警设计原则
### 10.1 可执行性
告警应说明对象、当前值、阈值、持续时间、影响、首个检查动作和处理手册。没有响应动作的信号更适合仪表盘或日报。
### 10.2 避免告警疲劳
- 使用合理 `for` 和稳定时间窗口过滤瞬时抖动。
- 通过分组、抑制合并同一根因产生的通知。
- 按服务、团队、等级路由。
- 定期统计数量、误报率和处理时长，删除无行动价值的规则。
### 10.3 分级与值班
等级应根据用户影响、冗余、扩散风险和剩余处理时间确定。要明确接收人、确认时限、升级路径、处理记录、复盘条件和规则负责人。
### 10.4 优先用户结果
推荐顺序：用户失败率或可用率、关键业务流程、容量耗尽风险、基础资源异常、普通状态变化。CPU 高不一定是故障，错误率高通常已经影响用户。

---

## 11. 常见问题排查
### 11.1 Target 显示 Down
1. 在 Prometheus“状态 -> 目标”查看错误。
2. 确认进程、容器、监听端口和 `/metrics`。
3. 从 Prometheus 容器访问目标。
4. 检查 Compose 网络、服务名、防火墙和超时。
```bash
docker compose ps
docker compose logs --tail=100 node-exporter
docker compose exec prometheus wget -qO- http://node-exporter:9100/metrics
docker network inspect monitoring
```
连接被拒绝通常表示未监听；超时重点检查网络或响应速度；解析失败重点检查服务名和容器网络。
### 11.2 rate() 返回空
- 先只查指标名，再逐个增加标签。
- 确认指标是 Counter，目标正常且查询时间包含数据。
- 刚启动时等待样本积累，把 `[1m]` 改为 `[5m]`。
- 在 Prometheus 查询页面直接验证，再放入 Grafana。
### 11.3 时区问题
Prometheus 使用时间戳存储，Grafana 通常按浏览器时区显示。检查浏览器、Grafana 偏好、主机时区、日志时区和时间同步。
```bash
timedatectl status
date
docker compose exec prometheus date
docker compose exec grafana date
```
应使用时间同步服务，不要手工长期修正系统时间。
### 11.4 数据保留与磁盘
磁盘占用由序列数量、抓取间隔、保留天数和标签基数决定。
```bash
docker system df
docker volume inspect monitoring-lab_prometheus-data
docker compose exec prometheus du -sh /prometheus
```
可缩短保留时间、降低非关键抓取频率、丢弃无用指标、修复高基数，并为监控系统自身设置磁盘告警。长期保存应评估远程存储。
### 11.5 配置加载失败
```bash
docker compose exec prometheus promtool check config /etc/prometheus/prometheus.yml
docker compose exec prometheus promtool check rules /etc/prometheus/alert-rules.yml
```
检查 YAML 缩进、列表短横线、PromQL、挂载路径和文件权限，并查看组件日志。
### 11.6 Grafana 连不上 Prometheus
数据源地址应为 `http://prometheus:9090`，不是 `localhost`。同时确认两个容器在同一网络、服务名正确、Prometheus 健康。
### 11.7 告警不通知
依次确认表达式有结果、已超过 `for`、Prometheus 发现 Alertmanager、路由匹配、未被静默或抑制、SMTP 或 Webhook 可达。
```bash
docker compose logs --tail=200 prometheus
docker compose logs --tail=200 alertmanager
```
### 11.8 清理环境
```bash
docker compose down
# 同时删除持久化实验数据
docker compose down -v
```
`-v` 会删除历史指标、Grafana 配置和 Alertmanager 状态，执行前确认数据不再需要。

---

## 12. 学习路线与实践任务
### 12.1 必做任务
- [ ] 从零启动 Prometheus、Grafana、node_exporter、Alertmanager。
- [ ] 在目标页面解释 `job`、`instance` 和 `up`。
- [ ] 导入 Node Exporter Full 1860 并看懂 CPU、内存、磁盘、网络图表。
- [ ] 独立写 5 条 PromQL，覆盖 CPU、内存、磁盘、网络和服务存活。
- [ ] 自建一个包含三个主机核心面板的 Grafana 仪表盘。
- [ ] 配置磁盘告警并在专用测试挂载点故意触发。
- [ ] 观察告警从等待、触发到恢复的过程。
- [ ] 接入一个 Webhook，确认触发和恢复通知。
- [ ] 停止 node_exporter，完成一次 Target Down 排查。
- [ ] 清理容器和卷，再根据笔记重建。
### 12.2 安全触发磁盘告警
不要在生产主机或根文件系统盲目写满磁盘。应使用测试卷、专用挂载点或可回滚虚拟机，并提前准备删除命令。
```bash
fallocate -l 1G /专用测试挂载点/alert-test.bin
rm -f /专用测试挂载点/alert-test.bin
```
实验时可临时降低阈值和持续时间，完成后恢复正式配置。
### 12.3 进阶任务
- [ ] 部署 blackbox_exporter，探测 HTTP 和证书期限。
- [ ] 为 MySQL 或 Redis 部署 Exporter 并制作看板。
- [ ] 为示例应用增加请求数、错误数和延迟直方图。
- [ ] 分别使用 RED、USE 方法设计服务看板和主机排障表。
- [ ] 为严重告警补充处理手册，并统计一周告警噪声。
- [ ] 接入 Loki，从异常指标跳转到同一时间范围日志。
### 12.4 验收问题
1. 为什么 Prometheus 使用拉取模型？
2. Counter 与 Gauge 的查询方式有何不同？
3. `rate()` 窗口为什么不能过短？
4. `for`、`group_wait`、`repeat_interval` 分别做什么？
5. Target 正常为什么不等于业务健康？
6. 为什么用户编号不适合作为标签？
7. 如何从指标异常转到日志排查？
8. 如何安全验证磁盘告警？

---

## 13. 总结
完整监控闭环是：Exporter 或应用暴露指标，Prometheus 抓取和保存，PromQL 计算健康信号，Grafana 展示，告警规则判断异常，Alertmanager 路由通知，日志系统补充事件细节，最后通过故障注入验证触发、通知、恢复和处理流程。
学习重点不是收集尽可能多的指标，而是围绕“用户是否受影响、问题在哪里、值班人员下一步做什么”建立少而有效的信号。先完成单机监控和一次真实告警闭环，再扩展到中间件、应用指标、集中日志和链路追踪。
