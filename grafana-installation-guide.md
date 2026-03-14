# Grafana 最新版安装指南及学习排查最佳实践

## 目录
- [系统要求](#系统要求)
- [安装方法](#安装方法)
- [安装后配置](#安装后配置)
- [基础使用](#基础使用)
- [数据源配置](#数据源配置)
- [常见问题排查](#常见问题排查)
- [最佳实践](#最佳实践)

---

## 系统要求

### 硬件要求
- **CPU**: 最小 1核心，推荐 2核心或以上
- **内存**: 最小 255MB，推荐 1GB或以上
- **存储**: 最小 255MB，推荐 1GB或以上

### 支持的操作系统
- Ubuntu 18.04/20.04/22.04/24.04 LTS
- Debian 10/11/12
- CentOS/RHEL 7/8/9
- Rocky Linux 8/9
- AlmaLinux 8/9
- Fedora
- openSUSE/SLES
- macOS
- Windows

### 支持的数据库（用于存储配置）
- SQLite（默认，适合小规模部署）
- MySQL 5.7+/8.0+
- MariaDB 10.2+
- PostgreSQL 10+

---

## 安装方法

### 方法一：包管理器安装（推荐）

#### Ubuntu/Debian 系统

**1. 安装依赖包**
```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
```

**2. 添加 Grafana GPG 密钥**
```bash
sudo mkdir -p /etc/apt/keyrings/
wget -q -O /etc/apt/keyrings/grafana.key https://apt.grafana.com/gpg.key
```

**3. 添加 Grafana 仓库**
```bash
echo "deb [signed-by=/etc/apt/keyrings/grafana.key] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```

**4. 更新并安装 Grafana**
```bash
sudo apt-get update
sudo apt-get install grafana
```

**5. 启动 Grafana 服务**
```bash
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
sudo systemctl status grafana-server
```

#### CentOS/RHEL/Rocky/AlmaLinux 系统

**1. 添加 Grafana 仓库**
```bash
sudo tee /etc/yum.repos.d/grafana.repo <<EOF
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF
```

**2. 安装 Grafana**
```bash
sudo dnf install grafana  # RHEL 8/9
# 或
sudo yum install grafana  # RHEL 7
```

**3. 启动 Grafana 服务**
```bash
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
sudo systemctl status grafana-server
```

---

### 方法二：Docker 安装

**1. 运行 Grafana 容器**
```bash
docker run -d \
  --name=grafana \
  -p 3000:3000 \
  -v grafana-storage:/var/lib/grafana \
  grafana/grafana:latest
```

**2. 使用 Docker Compose（推荐）**
```bash
mkdir ~/grafana-docker && cd ~/grafana-docker
nano docker-compose.yml
```

**docker-compose.yml 内容：**
```yaml
version: '3.8'

services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-piechart-panel
    ports:
      - "3000:3000"
    volumes:
      - grafana-storage:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning
    networks:
      - monitoring

volumes:
  grafana-storage:

networks:
  monitoring:
    driver: bridge
```

**3. 启动服务**
```bash
docker-compose up -d
```

---

### 方法三：二进制安装

**1. 下载最新版本**
```bash
cd /tmp
wget https://dl.grafana.com/oss/release/grafana-latest.linux-amd64.tar.gz
tar -zxvf grafana-latest.linux-amd64.tar.gz
sudo mv grafana-* /opt/grafana
```

**2. 创建 Grafana 用户**
```bash
sudo useradd -r -s /bin/false grafana
sudo chown -R grafana:grafana /opt/grafana
```

**3. 创建 systemd 服务**
```bash
sudo nano /etc/systemd/system/grafana-server.service
```

**服务文件内容：**
```ini
[Unit]
Description=Grafana instance
Documentation=http://docs.grafana.org
Wants=network-online.target
After=network-online.target

[Service]
User=grafana
Group=grafana
Type=notify
ExecStart=/opt/grafana/bin/grafana-server \
  -homepath /opt/grafana \
  -config /opt/grafana/conf/defaults.ini
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**4. 启动服务**
```bash
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

---

## 安装后配置

### 1. 访问 Web 界面
打开浏览器访问：`http://your-server-ip:3000`

**默认登录凭据：**
- 用户名：`admin`
- 密码：`admin`

首次登录会要求修改密码。

### 2. 修改默认配置

**编辑配置文件：**
```bash
sudo nano /etc/grafana/grafana.ini
```

**常用配置项：**
```ini
[server]
# 访问地址和端口
http_addr = 0.0.0.0
http_port = 3000
domain = your-domain.com
root_url = http://your-domain.com:3000/

[database]
# 数据库配置（默认使用 SQLite）
type = sqlite3
# 或使用 MySQL
# type = mysql
# host = 127.0.0.1:3306
# name = grafana
# user = grafana
# password = your_password

[security]
# 安全配置
admin_user = admin
admin_password = your_strong_password
secret_key = your_secret_key
disable_gravatar = true

[users]
# 用户配置
allow_sign_up = false
auto_assign_org = true
auto_assign_org_role = Viewer

[auth.anonymous]
# 匿名访问（生产环境建议禁用）
enabled = false

[auth.basic]
enabled = true

[smtp]
# 邮件配置
enabled = true
host = smtp.example.com:587
user = your-email@example.com
password = your_email_password
from_address = grafana@example.com
from_name = Grafana
```

**重启服务使配置生效：**
```bash
sudo systemctl restart grafana-server
```

### 3. 配置防火墙

**Ubuntu/Debian (UFW):**
```bash
sudo ufw allow 3000/tcp
sudo ufw reload
```

**CentOS/RHEL (firewalld):**
```bash
sudo firewall-cmd --permanent --add-port=3000/tcp
sudo firewall-cmd --reload
```

### 4. 配置 HTTPS（推荐）

**使用 Nginx 反向代理：**
```bash
sudo apt install nginx  # Ubuntu/Debian
sudo dnf install nginx  # CentOS/RHEL
```

**Nginx 配置：**
```nginx
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /path/to/your/cert.pem;
    ssl_certificate_key /path/to/your/key.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 基础使用

### 1. 创建第一个 Dashboard

**步骤：**
1. 登录 Grafana
2. 点击左侧菜单 "Dashboards" → "New Dashboard"
3. 选择 "Add new panel"
4. 配置数据源和查询
5. 选择可视化类型
6. 点击 "Apply" 保存

### 2. 添加 Panel

**Panel 类型：**
- Time series（时间序列图）
- Stat（统计值）
- Table（表格）
- Gauge（仪表盘）
- Bar chart（柱状图）
- Pie chart（饼图）
- Heatmap（热力图）
- GeoMap（地理地图）

### 3. 配置告警

**步骤：**
1. 在 Panel 编辑页面，切换到 "Alert" 标签
2. 点击 "Create alert rule from this panel"
3. 配置告警条件：
   - 评估频率
   - 告警条件
   - 持续时间
4. 配置通知渠道
5. 保存规则

### 4. 用户和权限管理

**创建用户：**
- Administration → Users → New user

**创建团队：**
- Administration → Teams → New team

**权限级别：**
- Admin：完全管理权限
- Editor：可以编辑 Dashboard
- Viewer：只读权限

---

## 数据源配置

### 1. Prometheus

**配置步骤：**
1. Configuration → Data sources → Add data source
2. 选择 "Prometheus"
3. 配置 URL：`http://prometheus-server:9090`
4. 点击 "Save & Test"

**常用查询示例：**
```promql
# CPU 使用率
100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 内存使用率
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# 磁盘使用率
(1 - (node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"})) * 100
```

### 2. InfluxDB

**配置步骤：**
1. Configuration → Data sources → Add data source
2. 选择 "InfluxDB"
3. 配置连接信息：
   - URL：`http://influxdb:8086`
   - Database：数据库名
   - User：用户名
   - Password：密码
4. 点击 "Save & Test"

### 3. MySQL/PostgreSQL

**配置步骤：**
1. Configuration → Data sources → Add data source
2. 选择 "MySQL" 或 "PostgreSQL"
3. 配置数据库连接：
   - Host：数据库地址
   - Database：数据库名
   - User：用户名
   - Password：密码
4. 点击 "Save & Test"

**SQL 查询示例：**
```sql
SELECT
  time_column as time,
  value_column as value,
  metric_column as metric
FROM your_table
WHERE time_column > now() - interval '1 hour'
ORDER BY time_column
```

### 4. Elasticsearch

**配置步骤：**
1. Configuration → Data sources → Add data source
2. 选择 "Elasticsearch"
3. 配置连接：
   - URL：`http://elasticsearch:9200`
   - Index name：索引名称
   - Time field：时间字段
4. 点击 "Save & Test"

### 5. Zabbix

**安装插件：**
```bash
grafana-cli plugins install alexanderzobnin-zabbix-app
sudo systemctl restart grafana-server
```

**配置步骤：**
1. Configuration → Plugins → Zabbix → Enable
2. Configuration → Data sources → Add data source
3. 选择 "Zabbix"
4. 配置 Zabbix API：
   - URL：`http://zabbix-server/zabbix/api_jsonrpc.php`
   - Zabbix API credentials
5. 点击 "Save & Test"

---

## 常见问题排查

### 1. Grafana 无法启动

**检查日志：**
```bash
sudo journalctl -u grafana-server -f
# 或
sudo tail -f /var/log/grafana/grafana.log
```

**常见原因：**
- 端口被占用
- 权限问题
- 配置文件错误

**解决方案：**
```bash
# 检查端口占用
sudo netstat -tlnp | grep 3000

# 检查权限
sudo chown -R grafana:grafana /var/lib/grafana
sudo chown -R grafana:grafana /var/log/grafana

# 验证配置文件
grafana-server -config /etc/grafana/grafana.ini -homepath /usr/share/grafana -verify-config
```

### 2. 无法访问 Web 界面

**检查服务状态：**
```bash
sudo systemctl status grafana-server
```

**检查防火墙：**
```bash
# Ubuntu/Debian
sudo ufw status

# CentOS/RHEL
sudo firewall-cmd --list-all
```

**检查监听地址：**
```bash
sudo netstat -tlnp | grep grafana
```

**解决方案：**
```bash
# 确保配置文件中 http_addr 设置正确
# /etc/grafana/grafana.ini
[server]
http_addr = 0.0.0.0  # 监听所有接口
http_port = 3000
```

### 3. 数据源连接失败

**Prometheus 连接问题：**
```bash
# 测试 Prometheus 连接
curl http://prometheus-server:9090/api/v1/query?query=up

# 检查网络连通性
ping prometheus-server
telnet prometheus-server 9090
```

**数据库连接问题：**
```bash
# 测试 MySQL 连接
mysql -h mysql-server -u grafana -p

# 测试 PostgreSQL 连接
psql -h postgres-server -U grafana -d grafana
```

**解决方案：**
- 检查数据源 URL 是否正确
- 验证用户名和密码
- 检查网络连通性
- 确认防火墙规则

### 4. Dashboard 加载缓慢

**性能优化：**
```ini
# /etc/grafana/grafana.ini
[database]
# 使用外部数据库提高性能
type = mysql
host = mysql-server:3306
name = grafana
user = grafana
password = your_password

[dataproxy]
# 数据代理超时设置
timeout = 30
dialTimeout = 10

[cache]
# 启用缓存
enabled = true
```

**查询优化：**
- 减少查询时间范围
- 使用查询缓存
- 优化数据源查询语句
- 减少面板数量

### 5. 告警不工作

**检查告警状态：**
- Alerting → Alert rules → 查看告警状态
- Alerting → Contact points → 测试通知渠道

**检查通知渠道：**
```bash
# 测试邮件发送
echo "Test email" | mail -s "Test" your-email@example.com

# 测试 Webhook
curl -X POST http://webhook-url -d '{"test": "data"}'
```

**常见问题：**
- 通知渠道配置错误
- 告警条件设置不当
- 权限问题
- 网络连接问题

### 6. 插件安装失败

**手动安装插件：**
```bash
# 列出可用插件
grafana-cli plugins list-remote

# 安装插件
grafana-cli plugins install plugin-id

# 安装特定版本
grafana-cli plugins install plugin-id version

# 更新插件
grafana-cli plugins update plugin-id

# 卸载插件
grafana-cli plugins uninstall plugin-id
```

**离线安装插件：**
```bash
# 下载插件 zip 文件
wget https://grafana.com/api/plugins/plugin-id/versions/latest/download -O plugin.zip

# 解压到插件目录
sudo unzip plugin.zip -d /var/lib/grafana/plugins/

# 重启 Grafana
sudo systemctl restart grafana-server
```

---

## 最佳实践

### 1. 架构设计

**小规模部署：**
- 单 Grafana 实例
- SQLite 数据库
- 本地数据源

**中等规模：**
- 单 Grafana 实例
- 外部数据库（MySQL/PostgreSQL）
- 多数据源

**大规模部署：**
- Grafana 高可用集群
- 负载均衡
- 外部数据库集群
- 分布式数据源

### 2. Dashboard 设计原则

**设计原则：**
- 简洁明了，避免信息过载
- 使用变量提高复用性
- 合理使用颜色和图例
- 按业务逻辑组织 Dashboard
- 添加说明文档

**Dashboard 组织：**
```
- Overview Dashboard（概览）
  - 系统概览
  - 业务概览
  
- Detailed Dashboard（详细）
  - 系统详细指标
  - 应用详细指标
  
- Troubleshooting Dashboard（故障排查）
  - 日志分析
  - 性能分析
```

### 3. 变量使用

**常用变量类型：**
- Query：从数据源查询
- Custom：自定义值列表
- Constant：常量
- Text box：文本输入框
- Data source：数据源选择

**变量示例：**
```promql
# 主机变量
label_values(node_cpu_seconds_total, instance)

# 应用变量
label_values(app_requests_total, app)

# 使用变量
node_cpu_seconds_total{instance=~"$instance"}
```

### 4. 告警策略

**告警分级：**
- Critical：严重告警，需要立即处理
- Warning：警告告警，需要关注
- Info：信息告警，仅供参考

**告警规则设计：**
- 使用合理的阈值
- 设置持续时间避免抖动
- 配置告警依赖关系
- 使用告警分组减少通知

**告警通知策略：**
```yaml
# 通知策略示例
route:
  receiver: 'default'
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
  - match:
      severity: critical
    receiver: 'critical-team'
    continue: false
  - match:
      severity: warning
    receiver: 'warning-team'
```

### 5. 安全加固

**访问控制：**
```ini
# /etc/grafana/grafana.ini
[security]
# 禁用匿名访问
disable_gravatar = true

[users]
# 禁用用户注册
allow_sign_up = false

[auth.anonymous]
enabled = false

[auth.basic]
enabled = true
```

**HTTPS 配置：**
```ini
[server]
protocol = https
cert_key = /path/to/cert.key
cert_file = /path/to/cert.crt
```

**数据库安全：**
- 使用专用数据库用户
- 限制数据库权限
- 定期备份数据库
- 加密敏感数据

### 6. 备份策略

**备份内容：**
```bash
#!/bin/bash
# Grafana 备份脚本

BACKUP_DIR="/backup/grafana"
DATE=$(date +%Y%m%d_%H%M%S)

# 备份数据库
mysqldump -u grafana -p'password' grafana > $BACKUP_DIR/grafana_db_$DATE.sql

# 备份配置文件
tar -czf $BACKUP_DIR/grafana_config_$DATE.tar.gz /etc/grafana

# 备份数据目录
tar -czf $BACKUP_DIR/grafana_data_$DATE.tar.gz /var/lib/grafana

# 保留最近 7 天的备份
find $BACKUP_DIR -type f -mtime +7 -delete
```

**恢复步骤：**
```bash
# 恢复数据库
mysql -u grafana -p'password' grafana < grafana_db_backup.sql

# 恢复配置
tar -xzf grafana_config_backup.tar.gz -C /

# 恢复数据
tar -xzf grafana_data_backup.tar.gz -C /

# 重启服务
sudo systemctl restart grafana-server
```

### 7. 性能优化

**Grafana 配置优化：**
```ini
[database]
# 使用连接池
max_open_conn = 100
max_idle_conn = 100

[dataproxy]
# 数据代理优化
timeout = 30
dialTimeout = 10
keep_alive_seconds = 30

[cache]
# 启用缓存
enabled = true
max_age = 10m
```

**Dashboard 优化：**
- 减少面板数量
- 优化查询语句
- 使用查询缓存
- 合理设置刷新间隔
- 使用变量减少重复查询

### 8. 监控 Grafana 自身

**关键指标：**
- Grafana 进程状态
- HTTP 请求延迟
- 数据库连接数
- 内存使用
- CPU 使用

**使用 Prometheus 监控：**
```ini
# /etc/grafana/grafana.ini
[metrics]
enabled = true
interval_seconds = 10

[metrics.graphite]
address = localhost:2003
prefix = prod.grafana.%(instance_name)s.
```

**导入 Grafana 监控 Dashboard：**
- Dashboard ID: `3590`（Grafana Metrics）
- Dashboard ID: `4270`（Grafana Prometheus）

---

## 学习资源

### 官方资源
- Grafana 官方文档：https://grafana.com/docs/grafana/latest/
- Grafana 官方博客：https://grafana.com/blog/
- Grafana University：https://grafana.com/academy/

### 社区资源
- Grafana 社区：https://community.grafana.com/
- Grafana GitHub：https://github.com/grafana/grafana
- Grafana 插件市场：https://grafana.com/grafana/plugins/
- Grafana Dashboard 市场：https://grafana.com/grafana/dashboards/

### 推荐学习路径
1. **入门阶段**：掌握基本概念、安装配置、Dashboard 创建
2. **进阶阶段**：数据源配置、查询优化、告警设置
3. **高级阶段**：插件开发、API 使用、性能优化
4. **专家阶段**：高可用架构、自定义插件、集成开发

---

## 快速参考命令

```bash
# 服务管理
sudo systemctl start grafana-server
sudo systemctl stop grafana-server
sudo systemctl restart grafana-server
sudo systemctl status grafana-server

# 日志查看
sudo journalctl -u grafana-server -f
sudo tail -f /var/log/grafana/grafana.log

# 插件管理
grafana-cli plugins list-remote
grafana-cli plugins install plugin-id
grafana-cli plugins update plugin-id
grafana-cli plugins uninstall plugin-id

# 配置验证
grafana-server -config /etc/grafana/grafana.ini -homepath /usr/share/grafana -verify-config

# 重置管理员密码
grafana-cli admin reset-admin-password new-password

# 数据迁移
grafana-cli admin data-migration encrypt-datasource-passwords
```

---

## 常用 Dashboard 推荐

### 系统监控
- **Node Exporter Full** (ID: 1860) - Linux 系统监控
- **Node Exporter for Prometheus** (ID: 11074) - 简洁版系统监控
- **Windows Node** (ID: 11538) - Windows 系统监控

### 容器监控
- **Docker and system monitoring** (ID: 893) - Docker 监控
- **Kubernetes Cluster Monitoring** (ID: 315) - Kubernetes 集群监控

### 数据库监控
- **MySQL Overview** (ID: 7362) - MySQL 监控
- **PostgreSQL Database** (ID: 9628) - PostgreSQL 监控
- **Redis Dashboard** (ID: 763) - Redis 监控

### 应用监控
- **Nginx VTS Stats** (ID: 2949) - Nginx 监控
- **Apache Status** (ID: 3894) - Apache 监控
- **JVM Dashboard** (ID: 11504) - Java 应用监控

---

## 总结

Grafana 是一个功能强大的可视化平台，通过本指南，您应该能够：

1. 成功安装和配置 Grafana 最新版本
2. 理解基本概念和操作
3. 配置各种数据源
4. 创建专业的 Dashboard
5. 排查常见问题
6. 应用最佳实践优化部署

建议按照实际需求逐步实施，从简单的 Dashboard 开始，逐步掌握高级功能。持续学习和实践是掌握 Grafana 的关键。

---

**文档版本**: 1.0
**最后更新**: 2026-03-14
**适用版本**: Grafana 最新版（11.x+）
