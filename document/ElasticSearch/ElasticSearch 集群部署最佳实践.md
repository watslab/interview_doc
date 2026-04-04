# ElasticSearch 集群部署最佳实践

## 概述

本文档介绍 ElasticSearch 集群部署的两种方式和两种环境：

| 部署方式 | 环境 | 适用场景 |
|----------|------|----------|
| 软件包方式 | Linux | 生产环境 |
| 软件包方式 | Windows | 开发测试 |
| Docker方式 | Linux | 生产/测试环境 |
| Docker方式 | Windows | 开发测试 |

---

## 一、Linux 软件包方式部署

### 1.1 环境准备

#### 系统要求

| 项目 | 要求 |
|------|------|
| 操作系统 | CentOS 7+ / Ubuntu 18.04+ |
| Java | JDK 11 或 JDK 17（ES 8.x 推荐） |
| 内存 | 至少 4GB，推荐 8GB+ |
| 磁盘 | SSD 推荐，至少 50GB |

#### 系统参数优化

```bash
# 1. 修改文件句柄数
vim /etc/security/limits.conf

# 添加以下内容
* soft nofile 65536
* hard nofile 65536
* soft nproc 4096
* hard nproc 4096

# 2. 修改内存映射限制
vim /etc/sysctl.conf

# 添加以下内容
vm.max_map_count=262144
vm.swappiness=1

# 使配置生效
sysctl -p

# 3. 关闭Swap
swapoff -a

# 永久关闭，编辑 /etc/fstab，注释 swap 行
```

### 1.2 安装 ElasticSearch

#### 下载并解压

```bash
# 下载（以 8.11.1 为例）
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.11.1-linux-x86_64.tar.gz

# 解压
tar -xzf elasticsearch-8.11.1-linux-x86_64.tar.gz -C /opt/
mv /opt/elasticsearch-8.11.1 /opt/elasticsearch

# 创建数据和日志目录
mkdir -p /opt/elasticsearch/data
mkdir -p /opt/elasticsearch/logs
```

#### 创建 ES 用户（重要）

```bash
# ES 不能以 root 用户运行，必须创建专用用户
groupadd es
useradd es -g es -p elasticsearch

# 授权
chown -R es:es /opt/elasticsearch
```

### 1.3 配置 ElasticSearch

#### JVM 配置

```bash
vim /opt/elasticsearch/config/jvm.options

# 修改堆内存（建议设置为物理内存的 50%，但不超过 32GB）
-Xms4g
-Xmx4g

# 使用 G1 垃圾回收器
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```

#### elasticsearch.yml 配置（3节点集群示例）

**节点1配置**：

```yaml
# /opt/elasticsearch/config/elasticsearch.yml

# 集群名称（所有节点必须一致）
cluster.name: es-cluster

# 节点名称（每个节点唯一）
node.name: node-1

# 数据和日志路径
path.data: /opt/elasticsearch/data
path.logs: /opt/elasticsearch/logs

# 网络配置
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300

# 集群发现配置
discovery.seed_hosts: ["192.168.1.101", "192.168.1.102", "192.168.1.103"]

# 初始主节点（首次启动时指定）
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]

# 节点角色
node.roles: [master, data]

# 关闭安全认证（生产环境建议开启）
xpack.security.enabled: false
```

**节点2配置**：

```yaml
cluster.name: es-cluster
node.name: node-2
path.data: /opt/elasticsearch/data
path.logs: /opt/elasticsearch/logs
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300
discovery.seed_hosts: ["192.168.1.101", "192.168.1.102", "192.168.1.103"]
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
node.roles: [master, data]
xpack.security.enabled: false
```

**节点3配置**：

```yaml
cluster.name: es-cluster
node.name: node-3
path.data: /opt/elasticsearch/data
path.logs: /opt/elasticsearch/logs
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300
discovery.seed_hosts: ["192.168.1.101", "192.168.1.102", "192.168.1.103"]
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
node.roles: [master, data]
xpack.security.enabled: false
```

### 1.4 启动 ElasticSearch

```bash
# 切换到 es 用户
su es

# 启动（前台运行）
/opt/elasticsearch/bin/elasticsearch

# 后台运行
/opt/elasticsearch/bin/elasticsearch -d

# 查看日志
tail -f /opt/elasticsearch/logs/es-cluster.log
```

### 1.5 验证集群

```bash
# 检查集群状态
curl -X GET "http://192.168.1.101:9200/_cluster/health?pretty"

# 查看节点信息
curl -X GET "http://192.168.1.101:9200/_cat/nodes?v"

# 查看集群信息
curl -X GET "http://192.168.1.101:9200"
```

---

## 二、Windows 软件包方式部署

### 2.1 环境准备

#### 系统要求

| 项目 | 要求 |
|------|------|
| 操作系统 | Windows 10/11 或 Windows Server 2016+ |
| Java | JDK 11 或 JDK 17 |
| 内存 | 至少 4GB |

#### 安装 JDK

1. 下载并安装 JDK 11 或 17
2. 配置环境变量 `JAVA_HOME`

### 2.2 安装 ElasticSearch

#### 下载并解压

1. 访问 https://www.elastic.co/downloads/elasticsearch
2. 下载 Windows ZIP 版本
3. 解压到目录，如 `D:\Software\elasticsearch`

### 2.3 配置 ElasticSearch

#### JVM 配置

编辑 `config/jvm.options`：

```
-Xms2g
-Xmx2g
```

#### elasticsearch.yml 配置（单节点示例）

```yaml
# D:\Software\elasticsearch\config\elasticsearch.yml

cluster.name: es-windows
node.name: node-1
path.data: D:/Software/elasticsearch/data
path.logs: D:/Software/elasticsearch/logs
network.host: 0.0.0.0
http.port: 9200

# 单节点模式
discovery.type: single-node

xpack.security.enabled: false
```

### 2.4 启动 ElasticSearch

```powershell
# 方式一：双击运行
D:\Software\elasticsearch\bin\elasticsearch.bat

# 方式二：命令行运行
cd D:\Software\elasticsearch\bin
.\elasticsearch.bat

# 方式三：安装为 Windows 服务
.\elasticsearch-service.bat install
.\elasticsearch-service.bat start
```

### 2.5 验证

```powershell
# 浏览器访问
http://localhost:9200

# 或使用 curl
curl http://localhost:9200/_cluster/health?pretty
```

---

## 三、Linux Docker 方式部署

### 3.1 环境准备

```bash
# 安装 Docker
curl -fsSL https://get.docker.com | sh

# 安装 Docker Compose
curl -L "https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# 启动 Docker
systemctl start docker
systemctl enable docker
```

### 3.2 创建 Docker Compose 配置

```bash
mkdir -p /opt/es-cluster
cd /opt/es-cluster
```

#### docker-compose.yml

```yaml
version: '3.8'

services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.1
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - elastic

  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.1
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data02:/usr/share/elasticsearch/data
    networks:
      - elastic

  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.1
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data03:/usr/share/elasticsearch/data
    networks:
      - elastic

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.1
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://es01:9200
    ports:
      - 5601:5601
    networks:
      - elastic
    depends_on:
      - es01

volumes:
  data01:
  data02:
  data03:

networks:
  elastic:
    driver: bridge
```

### 3.3 启动集群

```bash
# 启动
docker-compose up -d

# 查看状态
docker-compose ps

# 查看日志
docker-compose logs -f es01

# 停止
docker-compose down

# 停止并删除数据卷
docker-compose down -v
```

### 3.4 验证集群

```bash
curl http://localhost:9200/_cluster/health?pretty
curl http://localhost:9200/_cat/nodes?v
```

### 3.5 单机部署的问题

上述 Docker Compose 配置将三个 ES 节点部署在**同一台服务器**上，存在以下问题：

| 问题 | 说明 |
|------|------|
| **单点故障** | 服务器宕机，整个集群不可用 |
| **资源竞争** | 三个节点共享同一台服务器的 CPU、内存、磁盘 |
| **无高可用** | 无法实现真正的高可用架构 |
| **适用场景** | 仅适用于开发、测试环境 |

---

### 3.6 多机分布式部署（生产环境推荐）

#### 部署架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     多机分布式部署架构                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   服务器 A (192.168.1.101)     服务器 B (192.168.1.102)                 │
│   ┌─────────────────────┐     ┌─────────────────────┐                  │
│   │    Docker 容器      │     │    Docker 容器      │                  │
│   │    es-node-1        │     │    es-node-2        │                  │
│   │    端口: 9200       │     │    端口: 9200       │                  │
│   └─────────────────────┘     └─────────────────────┘                  │
│                                                                         │
│   服务器 C (192.168.1.103)                                              │
│   ┌─────────────────────┐                                              │
│   │    Docker 容器      │                                              │
│   │    es-node-3        │                                              │
│   │    端口: 9200       │                                              │
│   └─────────────────────┘                                              │
│                                                                         │
│   【优势】                                                               │
│   • 真正的高可用：一台服务器故障，集群仍可用                              │
│   • 资源隔离：每个节点独占服务器资源                                      │
│   • 负载分散：查询和写入负载分散到多台服务器                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 服务器准备

| 服务器 | IP 地址 | 角色 |
|--------|---------|------|
| 服务器 A | 192.168.1.101 | ES 节点 1 |
| 服务器 B | 192.168.1.102 | ES 节点 2 |
| 服务器 C | 192.168.1.103 | ES 节点 3 |

#### 前置条件

每台服务器都需要：

```bash
# 1. 安装 Docker
curl -fsSL https://get.docker.com | sh
systemctl start docker
systemctl enable docker

# 2. 修改系统参数
sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" >> /etc/sysctl.conf

# 3. 开放端口
firewall-cmd --permanent --add-port=9200/tcp
firewall-cmd --permanent --add-port=9300/tcp
firewall-cmd --reload

# 或使用 iptables
iptables -A INPUT -p tcp --dport 9200 -j ACCEPT
iptables -A INPUT -p tcp --dport 9300 -j ACCEPT
```

#### 服务器 A 配置（192.168.1.101）

```bash
mkdir -p /opt/es-cluster
cd /opt/es-cluster
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  es-node-1:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.1
    container_name: es-node-1
    environment:
      - node.name=es-node-1
      - cluster.name=es-distributed-cluster
      - network.publish_host=192.168.1.101
      - discovery.seed_hosts=192.168.1.101,192.168.1.102,192.168.1.103
      - cluster.initial_master_nodes=es-node-1,es-node-2,es-node-3
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms4g -Xmx4g"
      - xpack.security.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - /opt/es-cluster/data:/usr/share/elasticsearch/data
      - /opt/es-cluster/logs:/usr/share/elasticsearch/logs
    ports:
      - 9200:9200
      - 9300:9300
    restart: always
```

#### 服务器 B 配置（192.168.1.102）

```bash
mkdir -p /opt/es-cluster
cd /opt/es-cluster
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  es-node-2:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.1
    container_name: es-node-2
    environment:
      - node.name=es-node-2
      - cluster.name=es-distributed-cluster
      - network.publish_host=192.168.1.102
      - discovery.seed_hosts=192.168.1.101,192.168.1.102,192.168.1.103
      - cluster.initial_master_nodes=es-node-1,es-node-2,es-node-3
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms4g -Xmx4g"
      - xpack.security.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - /opt/es-cluster/data:/usr/share/elasticsearch/data
      - /opt/es-cluster/logs:/usr/share/elasticsearch/logs
    ports:
      - 9200:9200
      - 9300:9300
    restart: always
```

#### 服务器 C 配置（192.168.1.103）

```bash
mkdir -p /opt/es-cluster
cd /opt/es-cluster
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  es-node-3:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.1
    container_name: es-node-3
    environment:
      - node.name=es-node-3
      - cluster.name=es-distributed-cluster
      - network.publish_host=192.168.1.103
      - discovery.seed_hosts=192.168.1.101,192.168.1.102,192.168.1.103
      - cluster.initial_master_nodes=es-node-1,es-node-2,es-node-3
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms4g -Xmx4g"
      - xpack.security.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - /opt/es-cluster/data:/usr/share/elasticsearch/data
      - /opt/es-cluster/logs:/usr/share/elasticsearch/logs
    ports:
      - 9200:9200
      - 9300:9300
    restart: always
```

#### 关键配置说明

| 配置项 | 说明 |
|--------|------|
| `network.publish_host` | **关键配置**，告诉其他节点如何连接本节点 |
| `discovery.seed_hosts` | 所有节点的 IP 地址列表 |
| `cluster.initial_master_nodes` | 首次启动时参与选主的节点名称 |
| `cluster.name` | 所有节点必须一致 |
| `ports` | 开放 9200（HTTP）和 9300（Transport） |

#### 启动顺序

```bash
# 在三台服务器上依次启动（间隔约 30 秒）

# 服务器 A
docker-compose up -d

# 等待 30 秒...

# 服务器 B
docker-compose up -d

# 等待 30 秒...

# 服务器 C
docker-compose up -d
```

#### 验证集群

```bash
# 在任意服务器上执行
curl http://192.168.1.101:9200/_cluster/health?pretty

# 查看节点分布
curl http://192.168.1.101:9200/_cat/nodes?v

# 查看分片分布
curl http://192.168.1.101:9200/_cat/shards?v
```

#### 单机 vs 多机部署对比

| 对比项 | 单机 Docker Compose | 多机分布式部署 |
|--------|---------------------|----------------|
| 高可用 | ❌ 无 | ✅ 有 |
| 单点故障 | ❌ 存在 | ✅ 避免 |
| 资源隔离 | ❌ 共享资源 | ✅ 独占资源 |
| 运维复杂度 | ⭐ 简单 | ⭐⭐⭐ 中等 |
| 适用场景 | 开发/测试 | 生产环境 |
| 配置差异 | 使用容器名通信 | 使用 `network.publish_host` |

---

## 四、Windows Docker 方式部署

### 4.1 环境准备

1. 安装 Docker Desktop for Windows
2. 启用 WSL2 后端
3. 确保 Docker 正常运行

### 4.2 创建配置文件

创建目录 `D:\docker\es-cluster`，创建 `docker-compose.yml`：

```yaml
version: '3.8'

services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.1
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - xpack.security.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - D:/docker/es-cluster/data01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
      - 9300:9300
    networks:
      - elastic

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.1
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://es01:9200
    ports:
      - 5601:5601
    networks:
      - elastic
    depends_on:
      - es01

networks:
  elastic:
    driver: bridge
```

### 4.3 启动集群

```powershell
cd D:\docker\es-cluster

# 启动
docker-compose up -d

# 查看状态
docker-compose ps

# 查看日志
docker-compose logs -f
```

### 4.4 验证

```powershell
# 浏览器访问
http://localhost:9200
http://localhost:5601  # Kibana
```

---

## 五、生产环境最佳实践

### 5.1 JVM 配置建议

| 配置项 | 建议 |
|--------|------|
| 堆内存大小 | 物理内存的 50%，最大不超过 32GB |
| Xms 和 Xmx | 设置为相同值 |
| 垃圾回收器 | 使用 G1GC |
| MaxGCPauseMillis | 200ms |

```bash
# jvm.options
-Xms8g
-Xmx8g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=35
```

### 5.2 系统参数优化

```bash
# /etc/sysctl.conf
vm.max_map_count=262144
vm.swappiness=1
net.core.somaxconn=65535
net.ipv4.tcp_max_syn_backlog=65535

# /etc/security/limits.conf
* soft nofile 65536
* hard nofile 65536
* soft nproc 65536
* hard nproc 65536
* soft memlock unlimited
* hard memlock unlimited
```

### 5.3 集群配置建议

| 配置项 | 建议 |
|--------|------|
| 主节点数 | 奇数个，至少 3 个（避免脑裂） |
| 数据节点数 | 根据数据量决定 |
| 分片数 | 每个节点不超过 20 个分片/GB堆内存 |
| 副本数 | 至少 1 个副本 |

### 5.4 节点角色分离

```yaml
# 主节点（不存储数据）
node.roles: [master]

# 数据节点
node.roles: [data]

# 协调节点
node.roles: []
```

### 5.5 安全配置

```yaml
# 开启安全认证
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

---

## 六、常见问题

### Q1: 启动报错 "max file descriptors too low"

```bash
# 解决方案
vim /etc/security/limits.conf
* soft nofile 65536
* hard nofile 65536

# 重新登录后生效
```

### Q2: 启动报错 "max virtual memory areas vm.max_map_count too low"

```bash
# 解决方案
sysctl -w vm.max_map_count=262144

# 永久生效
echo "vm.max_map_count=262144" >> /etc/sysctl.conf
```

### Q3: 启动报错 "the default discovery settings are unsuitable for production use"

```yaml
# 解决方案：在 elasticsearch.yml 中配置
discovery.seed_hosts: ["192.168.1.101", "192.168.1.102", "192.168.1.103"]
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
```

### Q4: Docker 容器内存不足

```yaml
# 在 docker-compose.yml 中添加内存限制
services:
  es01:
    deploy:
      resources:
        limits:
          memory: 4G
```

---

## 参考资料

- [Elasticsearch 官方文档 - Setup Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html)
- [Elasticsearch 集群安装部署](https://blog.csdn.net/aflyingcat520/article/details/106221184)
- [Elasticsearch集群快速部署(docker)](https://blog.csdn.net/weixin_42953108/article/details/130538877)
- [Elasticsearch 生产环境参数调优指南](https://blog.csdn.net/qq_38238956/article/details/156614631)
- [Elasticsearch Windows 集群部署](https://blog.csdn.net/maoye/article/details/124871494)
