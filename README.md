# GIITOJ
🏛️ GIITOJ
GIITOJ 是基于开源项目 QDUOJ 深度定制的在线评测系统。本项目专为 桂林信息科技学院 (GIIT) 的教学与竞赛场景设计，旨在提供一个更符合校内师生使用习惯、视觉更加现代化的算法竞技平台。

主要改进：

校色定制：深度融入桂林信息科技学院视觉识别元素。

架构调优：针对校内服务器环境进行 Docker 部署优化。

功能精简：移除冗余模块，聚焦于核心判题与教务管理。

## 🛠️ 快速部署

为了方便用户快速上手，我们提供了专门的部署仓库。请前往以下地址获取完整的 Docker 配置文件及部署脚本：

👉 **部署仓库地址**：[kiffylike/XOJ-ProDeploy](https://github.com/kiffylike/XOJ-ProDeploy)

### 快速启动指令：
```bash
# 克隆部署仓库
git clone https://github.com/kiffylike/XOJ-ProDeploy.git && cd XOJ-ProDeploy

# 启动服务
docker compose -f docker-compose.xoj-pro.yml up -d
