# GIITOJ ELK 日志系统部署与使用教程

## 目录

- [简介](#简介)
- [前置条件](#前置条件)
- [目录结构](#目录结构)
- [快速启动](#快速启动)
- [Kibana 初始化配置](#kibana-初始化配置)
- [常用 KQL 查询示例](#常用-kql-查询示例)
- [日志索引说明](#日志索引说明)
- [性能调优](#性能调优)
- [常见问题排查](#常见问题排查)

---

## 简介

本 ELK 栈为 GIITOJ（桂林信息科技学院在线评测平台）提供集中式日志管理能力：

```
各服务日志文件
    ↓ Filebeat 采集（轻量、断点续传）
    ↓ Logstash 解析（grok 结构化）
    ↓ Elasticsearch 存储索引
    ↓ Kibana 可视化（:5601）
```

| 组件 | 版本 | 端口 | 作用 |
|---|---|---|---|
| Elasticsearch | 8.13.4 | 9200 | 日志存储与全文索引 |
| Logstash | 8.13.4 | 5044 | 日志解析与路由 |
| Kibana | 8.13.4 | **5601** | Web 可视化界面 |
| Filebeat | 8.13.4 | - | 日志文件采集代理 |

---

## 前置条件

### 系统要求

- Docker >= 20.10
- Docker Compose >= 2.0
- 可用内存 >= **4GB**（Elasticsearch 需要至少 2GB）
- 已正常运行 GIITOJ 主服务（`docker-compose.yml`）

### 检查内存设置（Linux 必做）

Elasticsearch 需要关闭内存交换，执行以下命令（重启后失效，如需永久生效写入 `/etc/sysctl.conf`）：

```bash
# 临时生效
sudo sysctl -w vm.max_map_count=262144

# 永久生效（推荐）
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## 目录结构

```
elk-stack/
├── docker-compose.yml          # ELK 四服务编排
├── filebeat/
│   └── filebeat.yml            # Filebeat 采集规则（7 类日志源）
├── logstash/
│   ├── logstash.yml            # Logstash 基础配置
│   └── pipeline/
│       └── giitoj.conf         # Pipeline：grok 解析 + index 路由
└── kibana/
    └── kibana.yml              # Kibana 连接配置（中文界面）
```

---

## 快速启动

### 第一步：确认主服务日志目录存在

在 GIITOJ 根目录执行：

```bash
# 确认后端日志目录已被主服务创建
ls data/backend/log/

# 手动创建前端日志目录（首次运行需要）
mkdir -p data/frontend/log

# 手动创建判题服务日志目录（若不存在）
mkdir -p data/judge_server/log
```

### 第二步：重启前端容器（使日志挂载生效）

`docker-compose.yml` 已为 `oj-frontend` 新增了日志目录挂载。需重建前端容器使其生效：

```bash
# 在 GIITOJ 根目录执行
docker-compose up -d --force-recreate oj-frontend
```

### 第三步：启动 ELK 栈

```bash
# 在 GIITOJ 根目录执行
docker-compose -f elk-stack/docker-compose.yml up -d

# 查看启动状态（等待约 1-2 分钟）
docker-compose -f elk-stack/docker-compose.yml ps
```

### 第四步：验证服务健康状态

```bash
# 检查 Elasticsearch（返回 JSON 即正常）
curl http://localhost:9200/_cluster/health?pretty

# 检查 Kibana（返回 200 即正常）
curl -I http://localhost:5601

# 查看 Filebeat 日志（确认日志采集正常）
docker logs giitoj-filebeat --tail 30

# 查看 Logstash 日志（确认 pipeline 加载正常）
docker logs giitoj-logstash --tail 30
```

---

## Kibana 初始化配置

### 1. 打开 Kibana

浏览器访问：`http://<服务器IP>:5601`

首次打开会提示"请等待 Kibana 准备就绪"，稍等约 30 秒刷新即可。

### 2. 创建 Data View（索引模式）

1. 点击左侧菜单 → **Management（管理）** → **Stack Management**
2. 左侧选择 **Kibana** → **Data Views**
3. 点击右上角 **Create data view（创建数据视图）**
4. 填写：
   - **Name**：`GIITOJ 所有日志`
   - **Index pattern**：`giitoj-*`（匹配所有 GIITOJ 索引）
   - **Timestamp field**：`@timestamp`
5. 点击 **Save data view to Kibana（保存）**

> 也可以按服务分别创建：
> - `giitoj-nginx-access-*` → Nginx 访问日志
> - `giitoj-nginx-error-*` → Nginx 错误日志
> - `giitoj-app-*` → 后端应用日志
> - `giitoj-judge-*` → 判题服务日志
> - `giitoj-frontend-*` → 前端访问日志

### 3. 查看日志（Discover）

1. 点击左侧菜单 → **Analytics** → **Discover**
2. 左上角选择刚创建的 **Data View**
3. 右上角调整时间范围（如"最近 1 小时"）
4. 即可看到实时流入的日志

---

## 常用 KQL 查询示例

在 Discover 页面顶部搜索栏输入以下 KQL 查询：

### 查询 HTTP 错误请求（5xx）

```kql
tags: "nginx-access" AND status_code >= 500
```

### 查询某个 IP 的所有请求

```kql
client_ip: "192.168.1.100"
```

### 查询所有 POST 请求

```kql
tags: "nginx-access" AND http_method: "POST"
```

### 查询提交（Submit）API 请求

```kql
tags: "nginx-access" AND request_uri: *submission*
```

### 查询判题失败相关日志

```kql
tags: "judge" AND message: *error*
```

### 查询后端 ERROR 级别日志

```kql
tags: "gunicorn" AND log_level: "ERROR"
```

### 查询某用户登录相关日志

```kql
request_uri: *login* OR request_uri: *api/account*
```

### 查询慢请求（响应体 > 100KB）

```kql
tags: "nginx-access" AND bytes_sent > 102400
```

### 查询特定时间段内的日志

```kql
@timestamp >= "2024-01-01T00:00:00" AND @timestamp <= "2024-01-02T00:00:00"
```

---

## 日志索引说明

| 索引名 | 对应服务 | 关键字段 |
|---|---|---|
| `giitoj-nginx-access-YYYY.MM.DD` | 后端 Nginx 访问 | `client_ip`, `http_method`, `request_uri`, `status_code`, `bytes_sent` |
| `giitoj-nginx-error-YYYY.MM.DD` | 后端 Nginx 错误 | `log_level`, `error_message` |
| `giitoj-app-YYYY.MM.DD` | Gunicorn + Dramatiq + Supervisord | `log_level`, `log_message` |
| `giitoj-judge-YYYY.MM.DD` | 判题服务 | `message` |
| `giitoj-frontend-YYYY.MM.DD` | 前端 Nginx 访问 | `client_ip`, `http_method`, `request_uri`, `status_code` |

---

## 性能调优

### 调整 Elasticsearch 堆内存

服务器内存 ≥ 8GB 时，可适当增大：

编辑 `elk-stack/docker-compose.yml`，修改 `ES_JAVA_OPTS`：

```yaml
- ES_JAVA_OPTS=-Xms1g -Xmx1g  # 1GB（推荐总内存的 50%，最多不超过 32GB）
```

### 日志保留策略（定期清理旧索引）

ES 默认不自动删除索引，长期运行会占满磁盘。建议设置 ILM 策略或定期手动清理：

```bash
# 删除 30 天前的 Nginx 访问日志索引（示例）
curl -X DELETE "http://localhost:9200/giitoj-nginx-access-$(date -d '30 days ago' +%Y.%m.%d)"

# 查看所有 GIITOJ 索引大小
curl "http://localhost:9200/_cat/indices/giitoj-*?v&s=index"
```

---

## 常见问题排查

### Q1: Elasticsearch 启动后立即退出，日志显示 "max virtual memory areas vm.max_map_count [65530] is too low"

**解决方案：**

```bash
sudo sysctl -w vm.max_map_count=262144
```

### Q2: Kibana 打开后显示 "Kibana server is not ready yet"

ES 仍在初始化，等待 1-3 分钟后刷新。也可检查 ES 是否正常：

```bash
curl http://localhost:9200/_cluster/health
# status 为 "green" 或 "yellow" 均正常
```

### Q3: Filebeat 日志中看到 "permission denied" 错误

Filebeat 容器以 `root` 用户运行，但宿主机日志目录权限过严。执行：

```bash
chmod -R 755 data/backend/log/
chmod -R 755 data/judge_server/log/
chmod -R 755 data/frontend/log/
```

### Q4: Kibana 中没有数据 / 索引不存在

1. 确认 Filebeat 已采集到数据：`docker logs giitoj-filebeat --tail 50`
2. 确认 Logstash 正常处理：`docker logs giitoj-logstash --tail 50`
3. 检查 ES 中是否有索引：`curl http://localhost:9200/_cat/indices?v`
4. 主服务运行一段时间后才会产生日志，可先访问 OJ 平台触发日志生成

### Q5: 修改 pipeline 后如何生效

`giitoj.conf` 修改后，Logstash 已开启自动重载（`config.reload.automatic: true`），等待约 10 秒自动生效，无需重启。

### Q6: 停止 / 重启 ELK 服务

```bash
# 停止（保留数据）
docker-compose -f elk-stack/docker-compose.yml stop

# 重启
docker-compose -f elk-stack/docker-compose.yml restart

# 完全销毁（包括 ES 数据，谨慎操作）
docker-compose -f elk-stack/docker-compose.yml down -v
```

### Q7: 如何查看某个容器的实时日志

```bash
docker logs -f giitoj-elasticsearch
docker logs -f giitoj-logstash
docker logs -f giitoj-kibana
docker logs -f giitoj-filebeat
```

---

## 升级说明

升级时四个组件版本必须保持一致，编辑 `docker-compose.yml` 中所有镜像 tag 为同一版本后重新 `up -d` 即可。
