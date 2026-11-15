<p align="center">
  <a href="https://github.com/sleep1223/"><img src="web/public/favicon.svg" width="180" height="180" alt="FastSoyAdmin"></a>
</p>

# 汽机启停智能分析系统

---

## 项目简介

本系统用于对汽轮发电机组的启动和停机过程进行智能化分析，包括：

- **公共配置** — 单元/机组/实时点/指标分类/性能指标/公式验证/系统参数
- **启停配置** — 操作节点、启停过程、关联参数三大配置
- **过程评价** — 操作评价、总时长评价、能耗评价（实时计算 + 历史补算）
- **过程指导** — 趋势分析、参数统计、能耗统计（基于 InfluxDB 时序数据）
- **异常提醒** — 知识库、WebSocket 实时告警、语音播报、启停次数统计

### 技术亮点

- **双库架构** — PostgreSQL 存配置/记录/评价，InfluxDB 存高频实时采集值，通过 `point_code` 关联
- **三层 RBAC** — 菜单/API/按钮权限 + 行级 data_scope
- **CLI 代码生成** — 定义 Tortoise 模型后一键生成前后端 CRUD
- **业务模块自动发现** — 放入 `app/business/<name>/` 即自动注册路由，无需手动接线
- **事件总线** — 跨模块解耦通信，禁止互相 import
- **定时任务** — 内置 PeriodicTask（数据采集/评分计算/告警检测），无需 Celery

---

## 分支说明

采用 Git Flow 分支策略：

| 分支 | 用途 |
|------|------|
| `main` | 生产分支，永远可部署 |
| `develop` | 开发主分支，所有功能分支从这里拉出 |
| `feature/phase{N}-{模块名}` | 功能开发分支，各开发者按 Phase 认领 |
| `fix/{描述}` | 缺陷修复分支 |
| `hotfix/{描述}` | 紧急修复分支，从 main 拉出 |

**功能分支对应**：

| 分支 | 模块 |
|------|------|
| `feature/phase2-models` / `feature/phase3-common` | 基础架构 + 公共模块（已完成） |
| `feature/phase4-config` | 启停配置模块 |
| `feature/phase5-evaluation` | 评价模块 |
| `feature/phase6-guidance` | 指导模块 |
| `feature/phase7-alarm` | 告警模块 |

详见 `启停规划文件/team.md` 第五章 Git 协作规范。

---

## 环境要求

| 工具 | 版本 | 用途 |
|------|------|------|
| Python | >= 3.12 | 后端运行时 |
| Node.js | >= 20.19 | 前端运行时 |
| uv | 最新 | 后端包管理器（替代 pip） |
| pnpm | 最新 | 前端包管理器 |
| just | 最新 | 工作流命令封装 |
| Docker Desktop | 24+ | 容器化运行 PostgreSQL / InfluxDB / Redis |
| Git | 2.40+ | 版本控制 |

---

## 快速开始

### 1. 启动基础服务

```bash
docker compose up -d postgres influxdb redis
```

### 2. 后端

```bash
uv sync              # 安装后端依赖
cp .env.example .env # 复制环境变量模板，按需修改
just db-init         # 首次建表 + 超级管理员 + 基础数据
just dev             # 启动后端 (:9999)
```

### 3. 前端

```bash
cd web
pnpm install         # 安装前端依赖
pnpm dev             # 启动前端 (:9527)
```

### 4. 访问

- 后端 API 文档：`http://localhost:9999/docs`
- 前端页面：`http://localhost:9527`

---

## 常用命令

所有命令封装在 `justfile` 中，运行 `just --list` 查看全部。

| 命令 | 说明 |
|------|------|
| `just run` | 并行启动前后端开发服务器 |
| `just run backend` / `just run frontend` | 仅启动后端 / 前端 |
| `just db-init` | 首次初始化数据库 + 超级管理员 + 基础菜单 |
| `just mm` | 生成并应用数据库迁移（模型变更时执行） |
| `just cli-init <module>` | 创建后端业务模块骨架 |
| `just cli-crud <module> <中文名>` | 一键生成前后端 CRUD 代码 |
| `just fmt` | 格式化 + 安全 lint 修复 |
| `just check` | 提交前质量门禁（类型检查 + lint + 测试） |

> Windows 注意：`just` 需配置 `windows-shell`（PowerShell），PowerShell 5.1 不支持 `&&`，用 `;` 分隔。

---

## 架构

### 后端分层

```
app/
├── core/               # 框架基础设施（CRUDBase / CRUDRouter / RBAC / 事件总线 / PeriodicTask）
├── system/             # 系统模块（认证 / 用户 / 角色 / 菜单 / API / 字典 / Radar 监控）
├── business/           # 业务模块（自动发现）
│   ├── common/         #   公共配置（单元/机组/实时点/指标/分类/公式/系统参数）
│   ├── setonoff/       #   启停业务模型（监测配置/子过程/记录/统计/补算）
│   ├── config/         #   启停配置模块（Phase 4）
│   ├── evaluation/     #   评价模块（Phase 5）
│   ├── guidance/       #   指导模块（Phase 6）
│   └── alarm/          #   告警模块（Phase 7）
├── cli/                # 代码生成器
└── utils/              # 业务模块统一导入入口（含 influx_client）
```

**分层职责**：`api/` → `services/` → `controllers/` → `models + schemas`

**依赖方向铁律**：
- `app/core/` 不依赖 system / business
- `app/system/` 仅依赖 `app/core/`
- `app/business/<x>/` 仅依赖 `app/utils`，**不得**反向 import `app.system.*`，**不得**互相 import 兄弟模块
- 跨模块联动走事件总线（`emit` / `on`）

### 前端分层

```
web/src/
├── views/              # 页面（Elegant Router 基于文件系统自动路由）
├── service/api/        # Alova HTTP 封装
├── store/modules/      # Pinia 状态管理
├── typings/api/        # TypeScript 类型定义
├── router/             # Elegant Router + 路由守卫
└── locales/            # vue-i18n 国际化
```

### 双数据库架构

```
PostgreSQL（关系库）                    InfluxDB（时序库）
┌─────────────────────────────┐        ┌─────────────────────────────┐
│ biz_common_cell   单元      │        │ measurement = "PT001"       │
│ biz_common_unit   机组      │        │ 时间戳 → value: 480.5       │
│ biz_common_point  实时点    │◄──────►│ 时间戳 → value: 481.2       │
│ biz_common_index  指标      │ point_  │ （每10秒一条，高频）         │
│ biz_setonoff_*    启停业务  │ code    │                              │
│ 配置/记录/评价都存这里       │ 关联    │ 实时采集值/趋势都查这里       │
└─────────────────────────────┘        └─────────────────────────────┘
```

| 数据类型 | 存储位置 | 访问方式 |
|---------|---------|---------|
| 配置数据（编码/名称/单位/阈值等） | PostgreSQL | Tortoise ORM（`just cli-crud` 生成 CRUD） |
| 高频实时采集值（传感器每10秒读数） | InfluxDB | `app/utils/influx_client.py` 工具函数 |
| 评价/记录结果（评分/统计） | PostgreSQL | Tortoise ORM |

---

## 数据库

默认使用 **PostgreSQL**（asyncpg 驱动），通过 `.env` 的 `DB_URL` 配置：

```
DB_URL="postgres://fastsoyadmin:fastsoyadmin@localhost:5432/fastsoyadmin"
```

**InfluxDB** 配置（时序数据）：

```
INFLUXDB_URL="http://localhost:8086"
INFLUXDB_TOKEN="yanan-influxdb-token-2026"
INFLUXDB_ORG="yanan"
INFLUXDB_BUCKET="turbine_realtime"
```

数据库迁移使用 Tortoise ORM 自带工具：

```bash
just mm    # = makemigrations + migrate
```

---

## 响应码

所有接口统一返回 `{"code": "xxxx", "msg": "...", "data": ...}`，HTTP 状态恒为 200。

业务异常使用 `BizError(code, msg)` 抛出，由全局异常处理器统一包装。

| 码段 | 含义 | 前端典型行为 |
|------|------|-------------|
| `0000` | 成功 | 正常处理 |
| `1xxx` | 系统内部错误 / 序列化失败 | 框架自动弹错 |
| `21xx` | 认证失败（token / session） | 登出 / 弹窗 / 自动刷新 |
| `22xx` | 授权失败（RBAC / 按钮 / 角色） | 显示错误消息 |
| `23xx` | 资源冲突（唯一键重复） | 显示错误消息 |
| `24xx` | 通用业务失败 | 显示错误消息 |
| `25xx` | 限流 / 安全策略 | 显示错误消息 |
| `26xx` | Schema 必填字段兜底 | 显示错误消息 |
| `4000+` | 业务模块自定义（本项目从 4000 起） | 业务自行处理 |

业务码定义在 `app/core/code.py`，新增业务场景在末尾追加唯一码。

---

## 技术栈

### 后端

- **FastAPI** — 异步 Web 框架，原生 OpenAPI 文档
- **Tortoise ORM** — 原生 async ORM（asyncpg 驱动）
- **Pydantic v2** — 数据校验
- **Redis** — 缓存 / 会话 / Leader 锁 / Pub-Sub
- **Granian** — ASGI 服务器（多 Worker + 内存保护）
- **fastapi-guard** — 限流 / 安全防护
- **influxdb-client** — InfluxDB 2.x 异步客户端
- **uv** — 包管理器

### 前端

- **Vue 3 + TypeScript** — 组合式 API
- **Naive UI** — 组件库
- **ECharts** — 工业监控图表
- **Elegant Router** — 基于文件系统的自动路由
- **Alova** — HTTP 客户端（请求缓存/类型推断）
- **Pinia** — 状态管理
- **UnoCSS** — 原子化 CSS
- **pnpm** — 包管理器

### 基础设施

- **PostgreSQL 18** — 关系数据库（配置/记录/评价）
- **InfluxDB 2.7** — 时序数据库（实时采集值）
- **Redis 7** — 缓存 / 消息
- **Docker Compose** — 一键编排
- **Nginx** — 反向代理

---

## 项目文档

| 文档 | 位置 | 内容 |
|------|------|------|
| 团队开发指南 | `启停规划文件/team.md` | 角色分工、Git 协作、开发规范 |
| 技术设计 | `启停规划文件/design.md` | 架构、API 定义、数据模型、技术决策 |
| 任务清单 | `启停规划文件/tasks.md` | Phase 0-7 任务拆分与验收标准 |
| 功能模块树 | `启停规划文件/demotree.md` | 功能范围、业务规则、表结构映射 |
| 公共配置说明 | `启停规划文件/说明.md` | 前端公共配置 7 个页面的使用逻辑 |

