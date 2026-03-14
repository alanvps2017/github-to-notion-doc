# Prometheus 最新版安装指南及学习排查最佳实践

## 目录
- [系统要求](#系统要求)
- [安装方法](#安装方法)
- [安装后配置](#安装后配置)
- [基础使用](#基础使用)
- [Exporter 配置](#exporter-配置)
- [告警配置](#告警配置)
- [常见问题排查](#常见问题排查)
- [最佳实践](#最佳实践)

---

## 系统要求

### 硬件要求
- **CPU**: 最小 2核心，推荐 4核心或以上
- **内存**: 最小 4GB，推荐 8GB或以上（取决于监控规模）
- **存储**: 最小 20GB，推荐 100GB或以上（数据存储）

### 支持的操作系统
- Linux（所有主流发行版）
- macOS
- Windows
- FreeBSD

### 网络要求
- 默认端口：9090（Prometheus Server）
- 9091（Pushgateway）
- 9093（Alertmanager）

---

## 安装方法

### 方法一：二进制安装（推荐）

**1. 下载最新版本**
```bash
cd /tmp
# 访问 https://github.com/prometheus/prometheus/releases 获取最新版本
wget https://github.com/prometheus/prometheus/releases/download/v2.54.1/prometheus-2.54.1.linux-amd64.tar.gz
tar -xzf prometheus-2.54.1.linux-amd64.tar.gz
sudo mv prometheus-2.54.1.linux-amd64 /usr/local/prometheus
```

**2. 创建 Prometheus 用户**
```bash
sudo useradd -r -s /bin/false prometheus
sudo chown -R prometheus:prometheus /usr/local/prometheus
```

**3. 创建数据目录**
```bash
sudo mkdir -p /var/lib/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

**4. 创建 systemd 服务**
```bash
sudo nano /etc/systemd/system/prometheus.service
```

**服务文件内容：**
```ini
[Unit]
Description=Prometheus Monitoring System
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/prometheus/prometheus \
  --config.file=/usr/local/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --storage.tsdb.retention.time=15d \
  --web.console.templates=/usr/local/prometheus/consoles \
  --web.console.libraries=/usr/local/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-admin-api

Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

**5. 启动服务**
```bash
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
sudo systemctl status prometheus
```

---

### 方法二：Docker 安装

**1. 运行 Prometheus 容器**
```bash
docker run -d \
  --name=prometheus \
  -p 9090:9090 \
  -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v prometheus-data:/prometheus \
  prom/prometheus:latest \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/prometheus
```

**2. 使用 Docker Compose（推荐）**
```bash
mkdir ~/prometheus-docker && cd ~/prometheus-docker
nano docker-compose.yml
```

**docker-compose.yml 内容：**
```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-admin-api'
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    restart: unless-stopped
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager-data:/alertmanager
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    ports:
      - "3000:3000"
    volumes:
      - grafana-storage:/var/lib/grafana
    networks:
      - monitoring

volumes:
  prometheus-data:
  alertmanager-data:
  grafana-storage:

networks:
  monitoring:
    driver: bridge
```

**3. 创建配置文件**
```bash
nano prometheus.yml
```

**prometheus.yml 内容：**
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

**4. 启动服务**
```bash
docker-compose up -d
```

---

### 方法三：Kubernetes 安装

**使用 Helm 安装：**
```bash
# 添加 Prometheus Helm 仓库
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 安装 Prometheus
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring \
  --create-namespace \
  --set server.retention=15d \
  --set server.persistentVolume.size=50Gi
```

---

## 安装后配置

### 1. 访问 Web 界面
打开浏览器访问：`http://your-server-ip:9090`

### 2. 基础配置文件

**prometheus.yml 配置详解：**
```yaml
global:
  # 全局抓取间隔
  scrape_interval: 15s
  # 全局评估间隔
  evaluation_interval: 15s
  # 抓取超时时间
  scrape_timeout: 10s

# Alertmanager 配置
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - localhost:9093

# 告警规则文件
rule_files:
  - "alert_rules.yml"
  - "recording_rules.yml"

# 抓取配置
scrape_configs:
  # Prometheus 自身监控
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: 'prometheus'

  # Node Exporter 监控
  - job_name: 'node-exporter'
    static_configs:
      - targets:
        - '192.168.1.10:9100'
        - '192.168.1.11:9100'
        labels:
          group: 'production'

  # 应用监控
  - job_name: 'myapp'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['app1:8080', 'app2:8080']

  # 使用服务发现（Consul）
  - job_name: 'consul-services'
    consul_sd_configs:
      - server: 'localhost:8500'
        services: []

  # 使用服务发现（Kubernetes）
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
```

### 3. 配置文件热重载

**方法一：使用 API**
```bash
curl -X POST http://localhost:9090/-/reload
```

**方法二：使用 systemd**
```bash
sudo systemctl reload prometheus
```

**方法三：使用信号**
```bash
kill -HUP <prometheus-pid>
```

### 4. 配置防火墙

**Ubuntu/Debian (UFW):**
```bash
sudo ufw allow 9090/tcp
sudo ufw reload
```

**CentOS/RHEL (firewalld):**
```bash
sudo firewall-cmd --permanent --add-port=9090/tcp
sudo firewall-cmd --reload
```

---

## 基础使用

### 1. PromQL 基础查询

**即时向量查询：**
```promql
# 查询 CPU 使用率
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 查询内存使用率
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# 查询磁盘使用率
(1 - (node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"})) * 100

# 查询网络流量
rate(node_network_receive_bytes_total{device="eth0"}[5m])
```

**范围向量查询：**
```promql
# 查询过去 1 小时的 CPU 使用率
node_cpu_seconds_total{mode="idle"}[1h]

# 查询过去 30 分钟的内存使用
node_memory_MemAvailable_bytes[30m]
```

**聚合操作：**
```promql
# 按实例分组求和
sum by(instance) (node_cpu_seconds_total)

# 按作业分组求平均值
avg by(job) (node_memory_MemAvailable_bytes)

# 计算百分位数
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

### 2. 常用函数

**rate() - 计算速率：**
```promql
# 每秒请求数
rate(http_requests_total[5m])

# 网络流量速率
rate(node_network_receive_bytes_total[5m])
```

**irate() - 计算瞬时速率：**
```promql
# 瞬时 CPU 使用率
irate(node_cpu_seconds_total[5m])
```

**increase() - 计算增长量：**
```promql
# 过去 1 小时的请求增长量
increase(http_requests_total[1h])
```

**聚合函数：**
```promql
# 求和
sum(node_memory_MemTotal_bytes)

# 平均值
avg(node_memory_MemAvailable_bytes)

# 最小值
min(node_memory_MemAvailable_bytes)

# 最大值
max(node_memory_MemAvailable_bytes)

# 计数
count(node_memory_MemAvailable_bytes)
```

### 3. Web UI 使用

**查询界面：**
- 访问 `http://localhost:9090/graph`
- 输入 PromQL 查询
- 选择时间范围
- 查看图表或表格

**Targets 页面：**
- 访问 `http://localhost:9090/targets`
- 查看所有监控目标状态
- 检查抓取错误

**Alerts 页面：**
- 访问 `http://localhost:9090/alerts`
- 查看告警规则状态
- 查看触发告警

---

## Exporter 配置

### 1. Node Exporter（系统监控）

**安装：**
```bash
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar -xzf node_exporter-1.7.0.linux-amd64.tar.gz
sudo mv node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/
sudo useradd -r -s /bin/false node_exporter
```

**创建 systemd 服务：**
```bash
sudo nano /etc/systemd/system/node_exporter.service
```

**服务文件内容：**
```ini
[Unit]
Description=Node Exporter
Documentation=https://prometheus.io/docs/guides/node-exporter/
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

**启动服务：**
```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

**配置 Prometheus：**
```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
```

### 2. MySQL Exporter

**Docker 安装：**
```bash
docker run -d \
  --name mysql-exporter \
  -p 9104:9104 \
  -e DATA_SOURCE_NAME="user:password@(mysql-host:3306)/" \
  prom/mysqld-exporter:latest
```

**配置 Prometheus：**
```yaml
scrape_configs:
  - job_name: 'mysql-exporter'
    static_configs:
      - targets: ['localhost:9104']
```

### 3. Redis Exporter

**Docker 安装：**
```bash
docker run -d \
  --name redis-exporter \
  -p 9121:9121 \
  -e REDIS_ADDR="redis://redis-host:6379" \
  oliver006/redis_exporter:latest
```

**配置 Prometheus：**
```yaml
scrape_configs:
  - job_name: 'redis-exporter'
    static_configs:
      - targets: ['localhost:9121']
```

### 4. Nginx Exporter

**安装：**
```bash
docker run -d \
  --name nginx-exporter \
  -p 9113:9113 \
  -e NGINX_STATUS_URL="http://nginx-host/nginx_status" \
  nginx/nginx-prometheus-exporter:latest
```

**配置 Prometheus：**
```yaml
scrape_configs:
  - job_name: 'nginx-exporter'
    static_configs:
      - targets: ['localhost:9113']
```

### 5. Blackbox Exporter（网络探测）

**安装：**
```bash
docker run -d \
  --name blackbox-exporter \
  -p 9115:9115 \
  -v $(pwd)/blackbox.yml:/config/blackbox.yml \
  prom/blackbox-exporter:latest \
  --config.file=/config/blackbox.yml
```

**blackbox.yml 配置：**
```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: [200]
      method: GET

  tcp_connect:
    prober: tcp
    timeout: 5s

  icmp:
    prober: icmp
    timeout: 5s
```

**配置 Prometheus：**
```yaml
scrape_configs:
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - http://example.com
        - http://another-site.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

---

## 告警配置

### 1. Alertmanager 安装

**二进制安装：**
```bash
cd /tmp
wget https://github.com/prometheus/alertmanager/releases/download/v0.26.0/alertmanager-0.26.0.linux-amd64.tar.gz
tar -xzf alertmanager-0.26.0.linux-amd64.tar.gz
sudo mv alertmanager-0.26.0.linux-amd64 /usr/local/alertmanager
sudo useradd -r -s /bin/false alertmanager
sudo chown -R alertmanager:alertmanager /usr/local/alertmanager
```

**创建 systemd 服务：**
```bash
sudo nano /etc/systemd/system/alertmanager.service
```

**服务文件内容：**
```ini
[Unit]
Description=Prometheus Alertmanager
Documentation=https://prometheus.io/docs/alerting/alertmanager/
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/alertmanager/alertmanager \
  --config.file=/usr/local/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager

Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

**启动服务：**
```bash
sudo mkdir -p /var/lib/alertmanager
sudo chown alertmanager:alertmanager /var/lib/alertmanager
sudo systemctl daemon-reload
sudo systemctl start alertmanager
sudo systemctl enable alertmanager
```

### 2. Alertmanager 配置

**alertmanager.yml 配置：**
```yaml
global:
  # SMTP 配置
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'your_password'

  # Slack 配置
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

# 路由配置
route:
  receiver: 'default-receiver'
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  group_by: ['alertname', 'severity']

  # 子路由
  routes:
    - match:
        severity: critical
      receiver: 'critical-team'
      continue: false

    - match:
        severity: warning
      receiver: 'warning-team'

# 接收器配置
receivers:
  - name: 'default-receiver'
    email_configs:
      - to: 'team@example.com'
        send_resolved: true

  - name: 'critical-team'
    email_configs:
      - to: 'critical-team@example.com'
        send_resolved: true
    slack_configs:
      - channel: '#critical-alerts'
        send_resolved: true
        title: '{{ .Status | toUpper }}: {{ .CommonLabels.alertname }}'
        text: '{{ .CommonAnnotations.description }}'

  - name: 'warning-team'
    email_configs:
      - to: 'warning-team@example.com'
        send_resolved: true

# 抑制规则
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

### 3. 告警规则配置

**alert_rules.yml 配置：**
```yaml
groups:
  - name: node_alerts
    rules:
      # CPU 使用率告警
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is {{ $value }}% (threshold: 80%)"

      # 内存使用率告警
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value }}% (threshold: 85%)"

      # 磁盘使用率告警
      - alert: HighDiskUsage
        expr: (1 - (node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"})) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High disk usage on {{ $labels.instance }}"
          description: "Disk usage on {{ $labels.mountpoint }} is {{ $value }}% (threshold: 85%)"

      # 服务宕机告警
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down on {{ $labels.instance }}"
          description: "Service has been down for more than 1 minute"

  - name: application_alerts
    rules:
      # HTTP 错误率告警
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100 > 5
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High HTTP error rate"
          description: "HTTP error rate is {{ $value }}% (threshold: 5%)"

      # 响应时间告警
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time"
          description: "95th percentile response time is {{ $value }}s (threshold: 2s)"
```

### 4. 记录规则配置

**recording_rules.yml 配置：**
```yaml
groups:
  - name: node_recording_rules
    rules:
      # CPU 使用率
      - record: node:cpu_usage:rate5m
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

      # 内存使用率
      - record: node:memory_usage:percentage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

      # 磁盘使用率
      - record: node:disk_usage:percentage
        expr: (1 - (node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"})) * 100

      # 网络流量
      - record: node:network_receive:rate5m
        expr: rate(node_network_receive_bytes_total[5m])

      - record: node:network_transmit:rate5m
        expr: rate(node_network_transmit_bytes_total[5m])
```

---

## 常见问题排查

### 1. Prometheus 无法启动

**检查日志：**
```bash
sudo journalctl -u prometheus -f
# 或
tail -f /var/log/prometheus/prometheus.log
```

**常见原因：**
- 配置文件语法错误
- 数据目录权限问题
- 端口被占用

**解决方案：**
```bash
# 验证配置文件
promtool check config /usr/local/prometheus/prometheus.yml

# 检查端口占用
sudo netstat -tlnp | grep 9090

# 检查权限
sudo chown -R prometheus:prometheus /var/lib/prometheus
sudo chown -R prometheus:prometheus /usr/local/prometheus
```

### 2. Target 无法抓取

**检查 Target 状态：**
- 访问 `http://localhost:9090/targets`
- 查看错误信息

**常见问题：**
```bash
# 测试连接
curl http://target-host:9100/metrics

# 检查网络连通性
ping target-host
telnet target-host 9100

# 检查防火墙
sudo firewall-cmd --list-all  # CentOS/RHEL
sudo ufw status               # Ubuntu/Debian
```

**解决方案：**
- 检查目标服务是否运行
- 验证网络连通性
- 检查防火墙规则
- 确认配置文件中的地址和端口

### 3. 数据存储问题

**检查存储使用：**
```bash
# 查看数据目录大小
du -sh /var/lib/prometheus

# 查看数据文件
ls -lh /var/lib/prometheus
```

**清理旧数据：**
```bash
# 使用 API 删除数据（需要启用 admin API）
curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]={job="old-job"}'

# 清理磁盘块
curl -X POST http://localhost:9090/api/v1/admin/tsdb/clean_tombstones
```

**调整保留时间：**
```bash
# 修改启动参数
--storage.tsdb.retention.time=30d

# 或使用大小限制
--storage.tsdb.retention.size=50GB
```

### 4. 查询性能问题

**优化查询：**
- 减少查询时间范围
- 使用记录规则预计算
- 避免使用高基数标签
- 优化 PromQL 查询

**检查查询性能：**
```bash
# 查看查询统计
curl http://localhost:9090/api/v1/status/tsdb

# 查看当前查询
curl http://localhost:9090/api/v1/queries
```

### 5. 告警不触发

**检查告警规则：**
```bash
# 验证告警规则文件
promtool check rules /usr/local/prometheus/alert_rules.yml

# 查看告警状态
curl http://localhost:9090/api/v1/rules
```

**检查 Alertmanager：**
```bash
# 查看 Alertmanager 状态
curl http://localhost:9093/api/v2/status

# 查看告警
curl http://localhost:9093/api/v2/alerts

# 测试告警发送
amtool alert add --alertmanager.url=http://localhost:9093 \
  alertname=TestAlert severity=critical instance=test
```

### 6. 内存使用过高

**检查内存使用：**
```bash
# 查看 Prometheus 进程内存
ps aux | grep prometheus

# 查看系统内存
free -h
```

**优化内存使用：**
```bash
# 限制内存使用
--storage.tsdb.retention.time=15d

# 减少抓取目标
# 优化抓取间隔
scrape_interval: 30s  # 从 15s 增加到 30s

# 减少标签基数
# 避免使用高基数标签如 user_id, session_id 等
```

---

## 最佳实践

### 1. 架构设计

**小规模部署（<100 个目标）：**
- 单 Prometheus 实例
- 本地存储
- 简单告警配置

**中等规模（100-1000 个目标）：**
- Prometheus 联邦
- 远程存储
- Alertmanager 集群

**大规模部署（>1000 个目标）：**
- Prometheus 集群
- Thanos 或 VictoriaMetrics
- 分布式存储
- 多数据中心部署

### 2. 数据管理

**存储优化：**
```bash
# 合理设置保留时间
--storage.tsdb.retention.time=15d

# 使用远程存储
--storage.tsdb.retention.time=2d
--storage.remote.url=http://remote-storage:9090/write

# 压缩数据
--storage.tsdb.min-block-duration=2h
--storage.tsdb.max-block-duration=6h
```

**数据备份：**
```bash
#!/bin/bash
# Prometheus 数据备份脚本

BACKUP_DIR="/backup/prometheus"
DATE=$(date +%Y%m%d_%H%M%S)

# 创建快照
curl -X POST http://localhost:9090/api/v1/admin/tsdb/snapshot

# 备份数据
tar -czf $BACKUP_DIR/prometheus_snapshot_$DATE.tar.gz /var/lib/prometheus/snapshots

# 保留最近 7 天的备份
find $BACKUP_DIR -type f -mtime +7 -delete
```

### 3. 标签设计

**标签命名规范：**
- 使用小写字母和下划线
- 避免使用高基数标签
- 保持标签一致性

**推荐标签：**
```yaml
# 标准标签
job: 应用名称
instance: 实例地址
environment: 环境（production, staging, development）
region: 区域
datacenter: 数据中心
service: 服务名称
team: 团队名称
```

**避免的标签：**
```yaml
# 高基数标签（避免使用）
user_id: 用户ID
session_id: 会话ID
request_id: 请求ID
timestamp: 时间戳
```

### 4. 查询优化

**使用记录规则：**
```yaml
# 预计算常用查询
- record: job:http_requests:rate5m
  expr: sum by(job) (rate(http_requests_total[5m]))

# 在告警中使用
- alert: HighRequestRate
  expr: job:http_requests:rate5m > 1000
```

**优化 PromQL：**
```promql
# 不推荐：使用高基数标签
sum by(user_id) (http_requests_total)

# 推荐：使用低基数标签
sum by(job, instance) (http_requests_total)

# 使用 recording rules
job:http_requests:rate5m > 1000
```

### 5. 告警策略

**告警分级：**
- **Critical**: 需要立即处理
- **Warning**: 需要关注
- **Info**: 仅供参考

**告警规则设计：**
```yaml
# 使用合理的阈值和持续时间
- alert: HighCPUUsage
  expr: cpu_usage > 80
  for: 5m  # 持续 5 分钟才告警
  labels:
    severity: warning
  annotations:
    summary: "High CPU usage"
    description: "CPU usage is {{ $value }}%"
```

**告警分组：**
```yaml
# Alertmanager 配置
route:
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
```

### 6. 安全加固

**认证和加密：**
```bash
# 使用反向代理（Nginx）
server {
    listen 443 ssl;
    server_name prometheus.example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    auth_basic "Prometheus";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://localhost:9090;
    }
}
```

**访问控制：**
```bash
# 禁用 admin API（生产环境）
# --web.enable-admin-api=false

# 限制访问
# --web.listen-address=127.0.0.1:9090
```

### 7. 高可用配置

**Prometheus 联邦：**
```yaml
# 主 Prometheus 配置
scrape_configs:
  - job_name: 'federate'
    scrape_interval: 15s
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job="prometheus"}'
        - '{__name__=~"job:.*"}'
    static_configs:
      - targets:
        - 'prometheus-slave-1:9090'
        - 'prometheus-slave-2:9090'
```

**Alertmanager 集群：**
```bash
# 启动 Alertmanager 集群
alertmanager --cluster.listen-address=0.0.0.0:9094 \
  --cluster.peer=alertmanager-1:9094 \
  --cluster.peer=alertmanager-2:9094
```

### 8. 监控 Prometheus 自身

**关键指标：**
```promql
# Prometheus 内存使用
process_resident_memory_bytes{job="prometheus"}

# Prometheus CPU 使用
rate(process_cpu_seconds_total{job="prometheus"}[5m])

# 抓取成功率
sum(rate(scrape_duration_seconds_count[5m])) by (job)

# 查询延迟
histogram_quantile(0.95, rate(prometheus_http_request_duration_seconds_bucket[5m]))

# 数据存储大小
prometheus_tsdb_storage_blocks_bytes
```

**导入 Dashboard：**
- Dashboard ID: `3662`（Prometheus 2.0 Overview）
- Dashboard ID: `12486`（Prometheus Monitoring）

---

## 学习资源

### 官方资源
- Prometheus 官方文档：https://prometheus.io/docs/
- PromQL 文档：https://prometheus.io/docs/prometheus/latest/querying/basics/
- Alertmanager 文档：https://prometheus.io/docs/alerting/latest/alertmanager/

### 社区资源
- Prometheus GitHub：https://github.com/prometheus/prometheus
- Prometheus 用户邮件列表：https://groups.google.com/forum/#!forum/prometheus-users
- Exporter 列表：https://prometheus.io/docs/instrumenting/exporters/

### 推荐学习路径
1. **入门阶段**：掌握基本概念、安装配置、PromQL 基础
2. **进阶阶段**：Exporter 配置、告警设置、查询优化
3. **高级阶段**：高可用架构、远程存储、性能调优
4. **专家阶段**：自定义 Exporter、服务发现、大规模部署

---

## 快速参考命令

```bash
# 服务管理
sudo systemctl start prometheus
sudo systemctl stop prometheus
sudo systemctl restart prometheus
sudo systemctl status prometheus

# 配置验证
promtool check config prometheus.yml
promtool check rules alert_rules.yml

# 热重载配置
curl -X POST http://localhost:9090/-/reload

# 查看日志
sudo journalctl -u prometheus -f

# 数据管理
# 创建快照
curl -X POST http://localhost:9090/api/v1/admin/tsdb/snapshot

# 删除数据
curl -X POST -g 'http://localhost:9090/api/v1/admin/tsdb/delete_series?match[]={job="old-job"}'

# 清理墓碑
curl -X POST http://localhost:9090/api/v1/admin/tsdb/clean_tombstones

# 查询示例
# 查看所有指标
curl http://localhost:9090/api/v1/label/__name__/values

# 查看目标状态
curl http://localhost:9090/api/v1/targets

# 查看告警规则
curl http://localhost:9090/api/v1/rules
```

---

## 常用 Exporter 列表

### 系统监控
- **Node Exporter** - 系统指标（CPU、内存、磁盘、网络）
- **Process Exporter** - 进程监控
- **JMX Exporter** - Java 应用监控

### 数据库监控
- **MySQL Exporter** - MySQL 监控
- **PostgreSQL Exporter** - PostgreSQL 监控
- **Redis Exporter** - Redis 监控
- **MongoDB Exporter** - MongoDB 监控

### Web 服务器
- **Nginx Exporter** - Nginx 监控
- **Apache Exporter** - Apache 监控
- **HAProxy Exporter** - HAProxy 监控

### 消息队列
- **Kafka Exporter** - Kafka 监控
- **RabbitMQ Exporter** - RabbitMQ 监控

### 容器和编排
- **cAdvisor** - 容器监控
- **Kube-state-metrics** - Kubernetes 监控

### 网络监控
- **Blackbox Exporter** - 网络探测
- **SNMP Exporter** - SNMP 设备监控

---

## 总结

Prometheus 是一个功能强大的监控告警系统，通过本指南，您应该能够：

1. 成功安装和配置 Prometheus 最新版本
2. 理解 PromQL 查询语言
3. 配置各种 Exporter
4. 设置告警规则和通知
5. 排查常见问题
6. 应用最佳实践优化部署

建议按照实际需求逐步实施，从简单的监控开始，逐步掌握高级功能。持续学习和实践是掌握 Prometheus 的关键。

---

**文档版本**: 1.0
**最后更新**: 2026-03-14
**适用版本**: Prometheus 2.x 最新版
