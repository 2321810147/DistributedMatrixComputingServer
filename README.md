# DistributedMatrixComputingServer

> 分布式矩阵计算服务：C++ 调度/API 网关 + 独立计算节点 + 纯前端矩阵计算器。
> A distributed matrix-computing service built from a C++ gateway/scheduler, a separate compute node, and a vanilla-JS matrix calculator frontend.

本项目 fork 自 [ddd-y/FirstServer](https://github.com/ddd-y/FirstServer)。

---

## 中文

### 项目简介

这是一个**分布式矩阵计算平台**的服务端与前端实现。系统把"接收用户请求"和"执行矩阵运算"拆成两层：

- **FirstServer** 同时运行两套服务：
  - **HTTP API（Crow，端口 8081）**：用户注册/登录、JWT 鉴权、计算历史的写入、分页查询与二进制下载；
  - **TCP 调度服务（自建 epoll 反应堆，端口 8080）**：管理计算节点，按**预测负载**挑选最空闲的节点分给客户端。
- **计算节点（SecondServer）**：独立进程，向 FirstServer 注册并定时上报负载，对外提供 `POST /api/matrix/calculate` 执行矩阵运算。
- **前端**：原生 HTML/CSS/JavaScript，包含登录、矩阵计算器、历史记录三个页面。

### 主要功能

| 模块 | 说明 |
|---|---|
| 用户系统 | 注册（用户名/邮箱/密码）、登录、登出；登录成功返回 JWT（HS256，默认 24 小时有效） |
| 计算历史 | 按用户保存每次计算的输入/输出、运算类型、矩阵尺寸与状态；支持分页、按类型/运算/日期筛选 |
| 矩阵数据 | 基于 Eigen 的稠密/稀疏矩阵二进制序列化；支持把历史记录中的矩阵下载为 `.bin` 文件 |
| 负载调度 | 计算节点 `Join` 注册、`Update<load>` 上报负载、`Leave` 退出；调度器用加权移动平均预测下一次负载，优先选择预测负载最低的节点 |
| 异常处理 | 客户端通过 `FindE<ip>` 上报计算异常，调度器会把对应节点标记为可疑，超时未更新则自动摘除 |
| 并发模型 | 自建线程池（20 个工作线程）+ 非阻塞 socket + epoll 边缘触发；60 秒无活动的连接自动清理 |
| Web 界面 | 矩阵加法、减法、乘法、转置、求逆、特征值；历史记录分页、筛选、下载 |

### 系统架构

```
┌──────────────────────────┐
│  Browser                 │
│  HTML / CSS / JavaScript │
└────────────┬─────────────┘
             │ HTTP (8081)
             ▼
┌──────────────────────────┐        ┌──────────────────┐
│ FirstServer              │        │ MySQL            │
│  ├─ Crow HTTP API :8081  │◄──────►│ 用户 / 历史记录   │
│  └─ epoll 调度器 :8080   │        └──────────────────┘
└────────────┬─────────────┘
             │ TCP :8080（Join / Update / Leave）
             ▼
┌──────────────────────────┐
│ SecondServer（计算节点）   │
│  POST /api/matrix/calculate│
└──────────────────────────┘
```

调度协议以 `.` 作为消息结束符，共五种指令：

| 指令 | 发送方 | 作用 |
|---|---|---|
| `Request.` | 客户端 | 请求一个可用的计算节点 |
| `Join.` | 计算节点 | 注册自己（IP/端口从连接中获取） |
| `Update<load>.` | 计算节点 | 上报当前负载 |
| `Leave.` | 计算节点 | 主动退出 |
| `FindE<ip>.` | 客户端 | 上报某个节点计算失败 |

### 目录结构

```
.
├── FirstServer/                    # 网关、调度器、HTTP API、鉴权与持久化
│   ├── mainfunction.cpp            # 入口：启动 HTTP 服务与 epoll 调度服务
│   ├── crow_api.cpp / .h           # Crow 路由、业务逻辑、CORS
│   ├── MyInternet.* / Acceptor.*   # epoll 反应堆与连接接入
│   ├── ThreadPool.* / Handler.*    # 线程池与协议处理
│   ├── ProcessPool.* / metaProcess.* # 计算节点池与负载预测
│   ├── users_database.* / database.* # MySQL 访问与分页
│   ├── jwt_service.*               # JWT 签发与校验
│   ├── UnifiedData / EigenMatrixSerializer # 矩阵数据抽象与序列化
│   └── logger.*                    # 基于 spdlog 的日志
├── SecondServer/                   # 计算节点（当前仓库只有解决方案文件）
├── include/                        # vendored：nlohmann/json、spdlog
├── matrix-calculator-html-css-js/  # 前端：登录 / 计算器 / 历史记录
├── FirstServer.sln
└── b.txt
```

### 技术栈

| 层 | 技术 |
|---|---|
| 语言 | C++17 |
| HTTP / 网络 | [Crow](https://github.com/CrowCpp/Crow)、epoll、非阻塞 socket、线程池 |
| 数学库 | [Eigen](https://eigen.tuxfamily.org/)（稠密矩阵、稀疏矩阵） |
| 序列化 / 日志 | nlohmann/json、spdlog |
| 鉴权 | jwt-cpp（HS256） |
| 数据库 | MySQL C API |
| 前端 | HTML5 / CSS3 / 原生 JavaScript |

### 构建与运行

**环境要求**

- Linux（源码使用了 `epoll`、`unistd.h` 等 POSIX 接口）
- 支持 C++17 的编译器；仓库自带 `FirstServer.sln`，使用 Visual Studio 的 **Linux 工具链**（`ApplicationType=Linux`）
- Crow、Eigen、jwt-cpp 头文件，以及 MySQL 客户端开发库

**步骤**

1. 安装 C++ 工具链和 MySQL 客户端库，例如 Ubuntu：
   ```bash
   sudo apt install g++ make libmysqlclient-dev libssl-dev
   ```
2. 准备 Crow、Eigen、jwt-cpp 头文件，并修改 `FirstServer/FirstServer.vcxproj` 中的
   `AdditionalIncludeDirectories` / `AdditionalLibraryDirectories`，使其指向本机路径
   （当前配置中的路径为开发机上的 `/home/yzyz/projects/include` 等）。
3. 在 `FirstServer/crow_api.cpp` 中配置 MySQL 连接参数与 JWT 密钥，然后编译运行 `FirstServer`。
   启动后会同时监听：
   - `8080`：TCP 调度服务
   - `8081`：HTTP API
4. 单独启动计算节点（`SecondServer`）。计算节点需实现 `POST /api/matrix/calculate`，
   并向调度器发送 `Join.` / `Update<load>.` / `Leave.`。登录接口返回的 `ip` 字段会把节点地址下发给前端。
5. 打开前端页面（可直接用浏览器打开，或用任意静态服务器托管），按需修改
   `矩阵计算器.html` 里的 `BASE_URL` 与 `secondServerUrl`。

### HTTP API

| 方法 | 路径 | 说明 | 鉴权 |
|---|---|---|---|
| GET | `/api/health` | 健康检查 | 否 |
| GET | `/api/messages` | 获取消息列表 | 否 |
| POST | `/api/auth/register` | 注册（`name`、`email`、`password`） | 否 |
| POST | `/api/auth/login` | 登录（`username`、`password`） | 否 |
| POST | `/api/auth/logout` | 登出 | Bearer |
| GET | `/api/auth/get_records?page=&limit=` | 分页获取计算记录 | Bearer |
| GET | `/api/get_history?page=&page_size=&record_type=&operation=&date_from=&date_to=` | 历史记录列表与筛选 | Bearer |
| GET | `/api/get_history/<id>` | 历史记录详情（含输入/输出） | Bearer |
| POST | `/api/auth/add_history` | 写入一条计算历史 | Bearer |
| GET | `/api/download_history/<id>/matrix/<input\|output>` | 下载矩阵二进制文件 | Bearer |

鉴权方式：请求头 `Authorization: Bearer <jwt>`。

> `POST /api/auth/add_records` 目前只做令牌校验，实际写入使用 `add_history`。

### 矩阵二进制格式

- **稠密矩阵**：头部 `{rows, cols, storage_order, data_size}` + 按行/列连续存储的 `double`
- **稀疏矩阵**：头部 `{rows, cols, non_zeros, indices_size, values_size}` + 行索引数组 + 列索引数组 + 数值数组（COO 形式）
- `compress` 参数目前为占位实现，尚未启用实际压缩

### 配置与安全说明

- 数据库连接信息与 JWT 密钥目前**硬编码**在 `FirstServer/crow_api.cpp` 中。公开仓库中不应保留真实凭据，建议改用环境变量或配置文件，并**轮换已经提交过的数据库口令**。
- JWT 密钥当前为示例占位值，部署前务必替换。
- CORS 中间件当前允许 `Access-Control-Allow-Origin: *`，生产环境建议收紧到可信来源。
- 前端中的服务端地址是写死的内网 IP，部署到其他环境时需要同步修改。

### 已知限制

- `SecondServer/` 目前只有解决方案文件，计算节点的实现未包含在本仓库中。
- 项目依赖 Linux 的 `epoll`，不能直接在 Windows 上编译运行。
- 没有提供 CMake/Makefile，构建依赖 Visual Studio 的 Linux 工具链配置。

---

## English

### Overview

A distributed matrix-computing platform split into a gateway/scheduler, one or more compute nodes, and a browser frontend.

- **FirstServer** runs two services:
  - a **Crow HTTP API on port 8081** for registration, login, JWT-authenticated history management, and binary matrix downloads;
  - a **custom epoll TCP dispatcher on port 8080** that tracks compute nodes and assigns clients to the node with the lowest *predicted* load.
- **SecondServer** (compute node) registers with the dispatcher, reports its load periodically, and serves `POST /api/matrix/calculate`.
- **Frontend**: plain HTML/CSS/JavaScript with login, calculator, and history pages.

This repository is forked from [ddd-y/FirstServer](https://github.com/ddd-y/FirstServer).

### Features

- User registration/login with HS256 JWTs (24-hour expiry by default).
- Per-user calculation history with pagination and filtering by type, operation, and date.
- Dense and sparse matrix serialization powered by Eigen, including binary `.bin` downloads.
- Load-aware scheduling using an exponentially weighted moving average of reported load.
- Suspicious-node detection: client-reported failures (`FindE<ip>`) and stale heartbeats can remove a node from the pool.
- 20-worker thread pool, non-blocking sockets, edge-triggered epoll, and idle-connection reaping after 60 seconds.
- Matrix operations: addition, subtraction, multiplication, transpose, inverse, and eigenvalues.
- CORS middleware for cross-origin frontend access.

### Architecture

```
Browser ──HTTP:8081──► FirstServer ──► MySQL (users, history)
                           │
                           └──TCP:8080──► SecondServer (compute node)
                                              POST /api/matrix/calculate
```

Dispatcher protocol (messages end with `.`): `Request.`, `Join.`, `Update<load>.`, `Leave.`, `FindE<ip>.`

### Layout

| Path | Description |
|---|---|
| `FirstServer/` | Gateway, scheduler, HTTP API, JWT auth, persistence |
| `SecondServer/` | Compute-node solution file (implementation not included) |
| `include/` | Vendored nlohmann/json and spdlog |
| `matrix-calculator-html-css-js/` | Frontend pages |

### Tech stack

C++17 · Crow · Eigen · nlohmann/json · spdlog · jwt-cpp · MySQL C API · HTML/CSS/JavaScript.

### Build and run

1. Use a Linux environment with a C++17 compiler, Crow, Eigen, jwt-cpp, and the MySQL client development library.
2. Point `AdditionalIncludeDirectories` / `AdditionalLibraryDirectories` in `FirstServer/FirstServer.vcxproj` at your local dependency paths (the checked-in values reference the original developer's machine).
3. Configure the MySQL connection and JWT secret in `FirstServer/crow_api.cpp`, then build and run `FirstServer`. It listens on `8080` (dispatcher) and `8081` (HTTP API).
4. Start a compute node that implements `POST /api/matrix/calculate` and speaks the `Join.` / `Update<load>.` / `Leave.` protocol. The login response exposes the selected node's address to the frontend.
5. Open the frontend and adjust `BASE_URL` / `secondServerUrl` in `矩阵计算器.html`.

### API summary

`GET /api/health`, `GET /api/messages`, `POST /api/auth/register`, `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/auth/get_records`, `GET /api/get_history`, `GET /api/get_history/<id>`, `POST /api/auth/add_history`, `GET /api/download_history/<id>/matrix/<input|output>`. Authenticated routes expect `Authorization: Bearer <jwt>`.

### Security notes

- Database credentials and the JWT secret are currently hard-coded in `FirstServer/crow_api.cpp`. Move them to environment variables or a config file, and rotate any credential that has been committed to a public repository.
- Replace the placeholder JWT secret before deployment.
- CORS currently allows any origin (`*`).
- Frontend service URLs are hard-coded to internal IP addresses.

### Limitations

- The compute-node implementation is not included in this repository.
- The project relies on Linux `epoll` and cannot be built natively on Windows.
- No CMake/Makefile is provided; the build depends on the Visual Studio Linux toolchain.

---

### 第三方组件 / Third-party components

Crow, Eigen, nlohmann/json, spdlog, jwt-cpp, and the MySQL C API. Please refer to their respective licenses.

本项目未附带开源许可证文件。/ No license file is included in this repository.
