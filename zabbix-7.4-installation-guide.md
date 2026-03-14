# Zabbix 7.4 安装指南及学习排查最佳实践

## 目录
- [系统要求](#系统要求)
- [安装方法](#安装方法)
- [安装后配置](#安装后配置)
- [基础使用](#基础使用)
- [常见问题排查](#常见问题排查)
- [最佳实践](#最佳实践)

---

## 系统要求

### 硬件要求
- **CPU**: 最小 2核心，推荐 4核心或以上
- **内存**: 最小 2GB，推荐 8GB或以上（取决于监控规模）
- **存储**: 最小 10GB，推荐 50GB或以上（数据库存储）

### 支持的操作系统
- Ubuntu 20.04/22.04/24.04 LTS
- Debian 10/11/12
- CentOS/RHEL 8/9
- Rocky Linux 8/9
- AlmaLinux 8/9

### 数据库支持
- MySQL 8.0+
- MariaDB 10.5+
- PostgreSQL 13+
- TimescaleDB（推荐用于大规模部署）

---

## 安装方法

### 方法一：包管理器安装（推荐）

#### Ubuntu/Debian 系统

**1. 安装 Zabbix 仓库**
```bash
wget https://repo.zabbix.com/zabbix/7.4/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.4-1+ubuntu$(lsb_release -rs)_all.deb
sudo dpkg -i zabbix-release_7.4-1+ubuntu$(lsb_release -rs)_all.deb
sudo apt update
```

**2. 安装 Zabbix Server、前端和 Agent**
```bash
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

**3. 创建初始数据库**
```bash
# 安装 MySQL/MariaDB（如果未安装）
sudo apt install mysql-server
sudo mysql_secure_installation

# 创建 Zabbix 数据库和用户
sudo mysql -uroot -p
```

在 MySQL 命令行中执行：
```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'your_strong_password';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**4. 导入初始架构和数据**
```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

**5. 配置 Zabbix Server**
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```

修改以下参数：
```ini
DBPassword=your_strong_password
```

**6. 启动 Zabbix Server 和 Agent**
```bash
sudo systemctl restart zabbix-server zabbix-agent apache2
sudo systemctl enable zabbix-server zabbix-agent apache2
```

#### CentOS/RHEL/Rocky/AlmaLinux 系统

**1. 安装 Zabbix 仓库**
```bash
sudo rpm -Uvh https://repo.zabbix.com/zabbix/7.4/rhel/$(rpm -E %rhel)/x86_64/zabbix-release-7.4-1.el$(rpm -E %rhel).noarch.rpm
sudo dnf clean all
```

**2. 安装 Zabbix Server、前端和 Agent**
```bash
sudo dnf install zabbix-server-mysql zabbix-web-mysql zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

**3. 创建初始数据库**
```bash
# 安装 MySQL/MariaDB
sudo dnf install mysql-server
sudo systemctl start mysqld
sudo mysql_secure_installation

# 创建数据库
sudo mysql -uroot -p
```

在 MySQL 命令行中执行：
```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'your_strong_password';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**4. 导入初始架构和数据**
```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

**5. 配置 Zabbix Server**
```bash
sudo vi /etc/zabbix/zabbix_server.conf
```

修改以下参数：
```ini
DBPassword=your_strong_password
```

**6. 启动服务**
```bash
sudo systemctl restart zabbix-server zabbix-agent httpd php-fpm
sudo systemctl enable zabbix-server zabbix-agent httpd php-fpm
```

---

### 方法二：Docker 安装

**1. 创建 Docker Compose 文件**
```bash
mkdir ~/zabbix-docker && cd ~/zabbix-docker
nano docker-compose.yml
```

**2. docker-compose.yml 内容**
```yaml
version: '3.8'
services:
  zabbix-server:
    image: zabbix/zabbix-server-mysql:7.4-ubuntu-latest
    container_name: zabbix-server
    environment:
      - DB_SERVER_HOST=mysql-server
      - MYSQL_DATABASE=zabbix
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=your_strong_password
      - MYSQL_ROOT_PASSWORD=your_root_password
    ports:
      - "10051:10051"
    depends_on:
      - mysql-server
    restart: always

  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:7.4-ubuntu-latest
    container_name: zabbix-web
    environment:
      - DB_SERVER_HOST=mysql-server
      - MYSQL_DATABASE=zabbix
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=your_strong_password
      - MYSQL_ROOT_PASSWORD=your_root_password
      - ZBX_SERVER_HOST=zabbix-server
      - PHP_TZ=Asia/Shanghai
    ports:
      - "80:8080"
    depends_on:
      - mysql-server
      - zabbix-server
    restart: always

  mysql-server:
    image: mysql:8.0
    container_name: zabbix-mysql
    environment:
      - MYSQL_DATABASE=zabbix
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=your_strong_password
      - MYSQL_ROOT_PASSWORD=your_root_password
    volumes:
      - mysql-data:/var/lib/mysql
    restart: always

  zabbix-agent:
    image: zabbix/zabbix-agent:7.4-ubuntu-latest
    container_name: zabbix-agent
    environment:
      - ZBX_HOSTNAME=Zabbix server
      - ZBX_SERVER_HOST=zabbix-server
    ports:
      - "10050:10050"
    depends_on:
      - zabbix-server
    restart: always

volumes:
  mysql-data:
```

**3. 启动容器**
```bash
docker-compose up -d
```

---

## 安装后配置

### 1. 访问 Web 界面
打开浏览器访问：`http://your-server-ip`

**默认登录凭据：**
- 用户名：`Admin`
- 密码：`zabbix`

### 2. 完成安装向导
1. 检查先决条件
2. 配置数据库连接
3. 配置 Zabbix Server 详情
4. 完成安装

### 3. 修改默认密码
登录后立即修改默认管理员密码：
- 导航到：Administration → Users → Admin
- 点击 "Change password"
- 设置强密码

### 4. 配置 PHP 时区**
```bash
sudo nano /etc/php/*/apache2/php.ini
# 或
sudo nano /etc/php.ini
```

修改：
```ini
date.timezone = Asia/Shanghai
```

重启服务：
```bash
sudo systemctl restart apache2  # Ubuntu/Debian
sudo systemctl restart httpd    # CentOS/RHEL
```

---

## 基础使用

### 1. 添加主机
1. 导航到：Configuration → Hosts
2. 点击 "Create host"
3. 填写主机信息：
   - Host name：主机名
   - Groups：选择主机组
   - Interfaces：添加接口（Agent、SNMP等）
4. 关联模板
5. 点击 "Add"

### 2. 配置监控项
- 使用模板快速配置
- 自定义监控项：Configuration → Hosts → Items → Create item

### 3. 设置触发器
- Configuration → Hosts → Triggers → Create trigger
- 定义告警条件

### 4. 配置告警
1. **配置媒介类型**：Administration → Media types
2. **配置用户媒介**：Administration → Users → Media
3. **配置动作**：Configuration → Actions → Trigger actions

---

## 常见问题排查

### 1. Zabbix Server 无法启动

**检查日志：**
```bash
sudo tail -f /var/log/zabbix/zabbix_server.log
```

**常见原因：**
- 数据库连接失败
- 配置文件错误
- 权限问题

**解决方案：**
```bash
# 检查数据库连接
mysql -u zabbix -p -h localhost zabbix

# 检查配置文件语法
sudo zabbix_server -c /etc/zabbix/zabbix_server.conf -R config_cache_reload

# 检查权限
sudo chown -R zabbix:zabbix /var/log/zabbix
sudo chown -R zabbix:zabbix /var/run/zabbix
```

### 2. Agent 无法连接

**检查 Agent 状态：**
```bash
sudo systemctl status zabbix-agent
sudo tail -f /var/log/zabbix/zabbix_agentd.log
```

**测试连接：**
```bash
# 在 Server 端测试
zabbix_get -s agent-ip -k system.uname

# 检查端口
sudo netstat -tlnp | grep 10050
```

**常见解决方案：**
```bash
# 检查防火墙
sudo firewall-cmd --add-port=10050/tcp --permanent
sudo firewall-cmd --reload

# 或 Ubuntu UFW
sudo ufw allow 10050/tcp
```

### 3. Web 界面问题

**白屏或 500 错误：**
```bash
# 检查 PHP 错误日志
sudo tail -f /var/log/apache2/error.log  # Ubuntu/Debian
sudo tail -f /var/log/httpd/error_log    # CentOS/RHEL

# 检查 PHP 配置
php -i | grep "Configuration File"

# 检查权限
sudo chown -R www-data:www-data /usr/share/zabbix  # Ubuntu/Debian
sudo chown -R apache:apache /usr/share/zabbix      # CentOS/RHEL
```

### 4. 数据库性能问题

**检查数据库连接：**
```bash
# MySQL/MariaDB
sudo mysqladmin -u root -p processlist
sudo mysqladmin -u root -p status

# 查看慢查询
sudo mysql -u root -p -e "SHOW VARIABLES LIKE 'slow_query%';"
```

**优化建议：**
```sql
-- 增加 Zabbix 数据库参数
SET GLOBAL innodb_buffer_pool_size = 1073741824;  -- 1GB
SET GLOBAL max_connections = 500;
```

### 5. 性能问题排查

**检查 Zabbix 内部状态：**
```bash
# 查看 Zabbix Server 状态
zabbix_server -R ha_status

# 查看队列
zabbix_server -R queue_view

# 查看缓存使用
zabbix_server -R info
```

**监控 Zabbix 自身性能：**
- 使用内置模板 "Zabbix server health"
- 监控关键指标：
  - Zabbix data gathering process busy %
  - Zabbix configuration cache usage
  - Zabbix history cache usage

---

## 最佳实践

### 1. 架构设计

**小规模部署（<1000 台主机）：**
- 单 Server 架构
- 本地数据库

**中等规模（1000-10000 台主机）：**
- Zabbix Server + Proxy
- 独立数据库服务器

**大规模部署（>10000 台主机）：**
- Zabbix Server 集群
- 多个 Proxy
- 数据库集群/分片
- TimescaleDB

### 2. 数据库优化

**MySQL/MariaDB 配置优化：**
```ini
[mysqld]
innodb_buffer_pool_size = 物理内存的 70-80%
innodb_log_file_size = 256M
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
max_connections = 500
```

**定期维护：**
```sql
-- 优化表
OPTIMIZE TABLE history;
OPTIMIZE TABLE history_uint;
OPTIMIZE TABLE trends;
OPTIMIZE TABLE trends_uint;

-- 分区管理（使用 TimescaleDB）
SELECT drop_chunks('history', older_than => INTERVAL '30 days');
```

### 3. 监控策略

**模板管理：**
- 创建通用基础模板
- 按应用类型创建专用模板
- 使用模板继承减少重复配置

**监控项优化：**
- 合理设置采集间隔
- 使用主动模式减少 Server 压力
- 避免过于频繁的采集

**触发器设计：**
- 使用表达式避免误报
- 设置合理的依赖关系
- 使用宏提高灵活性

### 4. 告警管理

**分级告警：**
- 严重程度分级（Not classified → Disaster）
- 不同级别使用不同通知方式
- 设置告警升级机制

**告警抑制：**
- 维护期间自动抑制
- 使用依赖关系减少告警风暴
- 合理设置触发器恢复条件

### 5. 安全加固

**网络安全：**
```bash
# 配置防火墙规则
sudo firewall-cmd --permanent --add-port=10051/tcp  # Zabbix Server
sudo firewall-cmd --permanent --add-port=10050/tcp  # Zabbix Agent
sudo firewall-cmd --permanent --add-port=80/tcp     # Web UI
sudo firewall-cmd --reload
```

**加密通信：**
```bash
# 生成 TLS 证书
openssl req -x509 -sha256 -days 365 -nodes -newkey rsa:2048 \
  -keyout /etc/zabbix/zabbix_server.key \
  -out /etc/zabbix/zabbix_server.crt

# 配置 Server
# /etc/zabbix/zabbix_server.conf
TLSCAFile=/etc/zabbix/zabbix_server.crt
TLSCertFile=/etc/zabbix/zabbix_server.crt
TLSKeyFile=/etc/zabbix/zabbix_server.key
```

**访问控制：**
- 使用最小权限原则
- 定期审计用户权限
- 启用双因素认证（2FA）

### 6. 备份策略

**备份内容：**
```bash
#!/bin/bash
# Zabbix 备份脚本

BACKUP_DIR="/backup/zabbix"
DATE=$(date +%Y%m%d_%H%M%S)

# 备份数据库
mysqldump -u zabbix -p'password' --single-transaction zabbix > $BACKUP_DIR/zabbix_db_$DATE.sql

# 备份配置文件
tar -czf $BACKUP_DIR/zabbix_config_$DATE.tar.gz /etc/zabbix

# 备份 Web 配置
tar -czf $BACKUP_DIR/zabbix_web_$DATE.tar.gz /usr/share/zabbix

# 保留最近 7 天的备份
find $BACKUP_DIR -type f -mtime +7 -delete
```

### 7. 性能监控

**关键指标监控：**
- Zabbix Server 进程状态
- 数据库性能指标
- 系统资源使用（CPU、内存、磁盘）
- 网络延迟和带宽

**容量规划：**
- 监控 NVPS（New Values Per Second）
- 定期评估数据库增长
- 预估未来监控规模

### 8. 故障恢复

**高可用配置：**
- 使用 Zabbix HA 集群
- 数据库主从复制
- 负载均衡 Web 界面

**灾难恢复计划：**
1. 定期测试备份恢复
2. 文档化恢复流程
3. 准备备用服务器
4. 监控监控系统本身

---

## 学习资源

### 官方文档
- Zabbix 官方文档：https://www.zabbix.com/documentation/7.4/
- Zabbix 手册：https://www.zabbix.com/manuals

### 社区资源
- Zabbix 论坛：https://www.zabbix.com/forum
- Zabbix GitHub：https://github.com/zabbix/zabbix
- Zabbix 分享模板：https://share.zabbix.com

### 推荐学习路径
1. **入门阶段**：掌握基本概念和安装配置
2. **进阶阶段**：学习模板创建、触发器配置、告警设置
3. **高级阶段**：性能优化、高可用架构、自定义脚本
4. **专家阶段**：源码分析、插件开发、大规模部署

---

## 快速参考命令

```bash
# 服务管理
sudo systemctl start zabbix-server
sudo systemctl stop zabbix-server
sudo systemctl restart zabbix-server
sudo systemctl status zabbix-server

# 日志查看
sudo tail -f /var/log/zabbix/zabbix_server.log
sudo tail -f /var/log/zabbix/zabbix_agentd.log

# 配置测试
zabbix_server -c /etc/zabbix/zabbix_server.conf -t
zabbix_agentd -c /etc/zabbix/zabbix_agentd.conf -t

# 运行时控制
zabbix_server -R config_cache_reload
zabbix_server -R housekeeper_execute
zabbix_server -R log_level_increase

# Agent 测试
zabbix_get -s 127.0.0.1 -k agent.ping
zabbix_get -s 127.0.0.1 -k system.uname
```

---

## 总结

Zabbix 7.4 是一个功能强大的企业级监控解决方案。通过本指南，您应该能够：

1. 成功安装和配置 Zabbix 7.4
2. 理解基本监控概念和操作
3. 排查常见问题
4. 应用最佳实践优化部署

建议按照实际需求和环境逐步实施，从小规模开始，逐步扩展。持续学习和实践是掌握 Zabbix 的关键。

---

**文档版本**: 1.0
**最后更新**: 2026-03-14
**适用版本**: Zabbix 7.4
