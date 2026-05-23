# 闲鱼自动回复系统

本文档面向运维与部署人员，详细说明本项目的部署流程、依赖版本、环境变量、常用运维命令与故障排查方法。

> **个人备注**：本仓库为个人学习/使用目的 fork 自 [zhinianboke/xianyu-auto-reply](https://github.com/zhinianboke/xianyu-auto-reply)，如需官方支持请前往原仓库。

## 交流群

| 微信群 | 微信群1 | QQ群 |
|:---:|:---:|:---:|
| ![微信群](https://github.com/zhinianboke/xianyu-auto-reply/blob/20260423_old/static/wechat-group.png?raw=true) | ![微信群1](https://github.com/zhinianboke/xianyu-auto-reply/blob/20260423_old/static/wechat-group1.png?raw=true) | ![QQ群](https://github.com/zhinianboke/xianyu-auto-reply/blob/20260423_old/static/qq-group.png?raw=true) |

---

## 目录

- [一、项目简介](#一项目简介)
- [二、Python 版本要求](#二python-版本要求重要)
- [三、部署方式总览](#三部署方式总览)
- [四、方式一：Docker 加密源码构建](#四方式一docker-加密源码构建推荐)
- [五、方式二：本地源码运行](#五方式二本地源码运行开发调试)
- [六、方式三：Windows EXE 打包](#六方式三windows-exe-打包)
- [七、推广返佣子系统](#七推广返佣子系统可选模块)
- [八、环境变量配置说明](#八环境变量配置说明)
- [九、常用运维命令](#九常用运维命令)
- [十、数据持久化与备份](#十数据持久化与备份)
- [十一、故障排查](#十一故障排查)
- [十二、安全建议](#十二安全建议)
- [十三、目录结构参考](#十三目录结构参考)
- [十四、版本与许可](#十四版本与许可)

---

## 一、项目简介

本项目是一个基于微服务架构的闲鱼自动回复与运营管理系统，由前端、Backend-Web、WebSocket、Scheduler、MySQL、Redis 共 6 个核心服务组成，支持 Docker Compose 一键部署，也支持 Windows EXE 单机打包运行。

### 1.1 服务拓扑

| 服务名 | 容器名 | 默认端口 | 说明 |
| --- | --- | --- | --- |
| `frontend` | `xianyu-frontend` | `9000` | 前端站点（Nginx + 前端构建产物） |
| `backend-web` | `xianyu-backend-web` | `8089` | 主业务后端（FastAPI） |
| `websocket` | `xianyu-websocket` | `8090` | 闲鱼 WebSocket 接入与浏览器自动化 |
| `scheduler` | `xianyu-scheduler` | `8091` | 定时任务（重发、限流统计等） |
| `mysql` | `xianyu-mysql` | 仅内网 `3306` | MySQL 8.0 |
| `redis` | `xianyu-redis` | 仅内网 `6379` | Redis 7（开启密码与 AOF） |

### 1.2 功能特性

#### 核心功能
- **智能自动回复**：通过 WebSocket 实时接入闲鱼 IM 消息，支持 AI（OpenAI）和关键词匹配两种回复模式
- **多账号管理**：支持管理多个闲鱼账号，通过 Cookie 或扫码登录
- **在线聊天**：WebSocket 实时在线聊天界面，支持人工介入

#### 运营功能
- **订单管理**：自动获取和管理闲鱼订单
- **商品管理**：商品信息管理、搜索、批量操作
- **自动发货**：卡密自动发货、自动确认收货
- **自动评价**：自动给买家好评
- **自动重新上架**：商品定时重发/擦亮
- **商品发布**：批量发布商品到闲鱼
- **Cookie 管理**：自动刷新 Cookie 保持登录状态

#### 管理功能
- **用户系统**：多用户、管理员/普通用户角色
- **激活码系统**：软件授权激活（含硬件绑定）
- **仪表盘**：数据统计和可视化
- **风险控制**：风控日志记录
- **公告/广告**：系统公告和广告位管理
- **通知系统**：消息通知渠道配置
- **验证码处理**：极验（Geetest）滑动验证码自动识别

#### 推广分销（可选）
- **淘宝联盟**：淘宝联盟推广集成
- **分销体系**：代理商、子代理商管理
- **对账结算**：资金流和结算管理

### 1.3 技术栈

| 层 | 技术 |
|----|------|
| 前端 | React 18 + TypeScript + Vite + Tailwind CSS + Zustand |
| 后端 | Python 3.11/3.12 + FastAPI + SQLAlchemy + Uvicorn |
| 数据库 | MySQL 8.0 + Redis 7 |
| 浏览器自动化 | Playwright (Chromium) |
| AI | OpenAI API |
| 消息接入 | WebSocket（闲鱼 IM） |
| 部署 | Docker Compose / Windows EXE（Nuitka） |
| 源码保护 | Cython 编译 |

### 1.4 核心仓库目录

| 目录 | 作用 |
| --- | --- |
| `backend-web/` | Backend-Web 服务源码与 `Dockerfile` |
| `websocket/` | WebSocket 服务源码与 `Dockerfile` |
| `scheduler/` | Scheduler 服务源码与 `Dockerfile` |
| `common/` | 跨服务共享的数据库、工具与服务模块 |
| `frontend/` | 主前端（React + Vite） |
| `promotion/` | 推广返佣子系统（可选模块，存在时自动启用） |
| `launcher/` | EXE 单机版统一启动器入口 |
| `docker/` | 前端镜像构建上下文 |
| `scripts/` | 辅助脚本（按端口停止服务等） |

---

## 二、Python 版本要求（重要）

不同部署方式对 Python 版本要求**不一样**，请严格按照下表选择：

| 部署方式 | Python 版本 | 说明 |
| --- | --- | --- |
| Docker 加密源码构建 | **不需要本机 Python** | 镜像内统一使用 Python 3.11 |
| 本地源码运行（Windows） | **必须 Python 3
