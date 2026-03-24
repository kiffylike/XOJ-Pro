# GIITOJ 服务端口速查表

> 这份文档只记录当前仓库编排里会用到的端口，避免部署时遗忘。

## 1) 主业务栈（`docker-compose.yml`）

| 服务 | 容器名/模块 | 宿主机端口 | 容器端口 | 对外访问 | 说明 |
|---|---|---:|---:|---|---|
| Frontend (Nginx) | `oj-frontend` | `80` | `80` | 是 | OJ HTTP 入口 |
| Frontend (Nginx) | `oj-frontend` | `443` | `443` | 是 | OJ HTTPS 入口 |
| Backend (Django/Gunicorn) | `oj-backend` | - | `8000` | 否（仅容器网络） | 前端/Judge 通过服务名访问 |
| Judge Server | `oj-judge` | - | `8080` | 否（仅容器网络） | 心跳/判题接口 |
| PostgreSQL | `oj-postgres` | - | `5432` | 否（仅容器网络） | 数据库 |
| Redis | `oj-redis` | - | `6379` | 否（仅容器网络） | 缓存/队列 |

## 2) ELK 日志栈（`elk-stack/docker-compose.yml`）

| 服务 | 宿主机端口 | 容器端口 | 对外访问 | 说明 |
|---|---:|---:|---|---|
| Elasticsearch (HTTP) | `9200` | `9200` | 是 | ES API |
| Elasticsearch (Transport) | `9300` | `9300` | 是（一般内网） | 集群通信 |
| Logstash Beats Input | `5044` | `5044` | 是（按需） | Filebeat 上报入口 |
| Kibana | `5601` | `5601` | 是 | Web 管理界面 |
| Filebeat | - | - | 否 | 采集端，不对外监听 |

## 3) 监控栈（`prometheus-monitor/docker-compose.yml`）

| 服务 | 宿主机端口 | 容器端口 | 对外访问 | 说明 |
|---|---:|---:|---|---|
| Prometheus | `9090` | `9090` | 是 | 指标采集与查询 |
| Grafana | `3000` | `3000` | 是 | 可视化看板 |
| node-exporter | `9100` | `9100` | 是（通常内网） | 主机指标 |
| Alertmanager | `9093` | `9093` | 是（通常内网） | 告警路由 |
| dingtalk-webhook | `8060` | `8060` | 是（通常内网） | 告警转钉钉 |

## 4) Jenkins（`docker-compose.jenkins.yml`）

| 服务 | 宿主机端口 | 容器端口 | 对外访问 | 说明 |
|---|---:|---:|---|---|
| Jenkins Web UI | `8080` | `8080` | 是 | Jenkins 页面 |
| Jenkins Agent | `50000` | `50000` | 按需 | 分布式 Agent 连接端口 |

## 5) 示例/可选编排（默认不常驻）

### `JudgeServer-master/docker-compose.example.yml`
- `12358 -> 8080`（示例 judge_server 对外端口）

### `OnlineJudge-master/deploy/test_case_rsync/docker-compose.yml`
- `873 -> 873`（`oj-rsync-master`）

## 6) 常用访问地址

- OJ 前台：`http://<服务器IP>:80` 或 `https://<服务器IP>:443`
- Kibana：`http://<服务器IP>:5601`
- Prometheus：`http://<服务器IP>:9090`
- Grafana：`http://<服务器IP>:3000`
- Jenkins：`http://<服务器IP>:8080`

## 7) 一条命令查看实际端口映射

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}"
```
