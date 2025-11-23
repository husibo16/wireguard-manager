##  一套面向生产环境的 WireGuard 一体化安装、管理与多子网路由脚本

---

设计一套适用于 **Debian / Ubuntu** 的 **WireGuard 一体化安装与管理脚本**。脚本必须采用 **统一的整体架构**，在同一逻辑框架下分别实现服务端与客户端功能，确保结构一致、行为可预测、维护成本低。

脚本启动后通过 **交互式菜单** 让用户选择运行角色：

* **服务端（必须有公网 IP）**
* **客户端（公司内网节点 / 员工终端）**

> 设计原则：  
> - 服务端 = 控制平面 + 配置中心 + 路由枢纽  
> - 客户端 = 执行节点 + 连接端，不做策略  
> - 所有复杂性、策略和配置生成，全部集中在服务端统一管理。

---

## 🔵 1. 服务端：角色明确（中心节点 + 配置管理者）

服务端必须具备公网 IP，是整个 WireGuard 网络的中心节点，负责所有配置生成、路由策略与客户端管理。

### 核心职责

* 初始化服务端环境（安装、密钥、wg 接口、基础配置）
* **统一创建客户端：自动生成密钥 / 配置文件 / 分配地址 / 二维码**
* 管理所有客户端（新增 / 删除 / 列表 / 配置导出）
* 控制 WireGuard 连接（启动、停止、重载、状态查询）

> 说明：  
> - 所有客户端密钥与配置由服务端统一生成，避免地址冲突与策略分裂。  
> - 服务端掌握所有节点的公钥与 AllowedIPs，才能做统一访问控制。  
> - 客户端“零配置”，是降低使用门槛和减少错误的关键设计。

### 网络职责

* 启用 IPv4 / IPv6 转发
* 设置 NAT、iptables、防火墙规则
* 维护多子网（如公司内网 `192.168.x.0/24`）
* 实现跨子网访问（员工通过 VPS 访问公司内网）：

```text
员工 → VPS（服务端） → 公司内网节点 → 公司服务器
````

### 架构要求

* 服务端承担所有策略决策与配置生成
* 客户端不做密钥、不做策略、不做 NAT
* 服务端配置结构必须稳定、清晰、可扩展
* 支持长期 7×24 小时运行，重启与异常后可自动恢复

---

## 🔵 2. 客户端：角色明确（执行节点 + 轻量逻辑）

客户端无需公网 IP，只负责执行服务端生成的配置。

### 客户端职责

* 安装 WireGuard
* 导入服务端生成的配置（文件或二维码）
* 执行启动 / 停止 / 状态
* 使用服务端提供的路由策略访问资源

### 特殊客户端（公司内网节点）

用于“公司内网穿透”，额外职责：

* 向服务端声明本地内网子网（如 `192.168.50.0/24`）
* 自动设置本地路由
* 允许外出员工通过 VPS 访问公司服务器（文件服务器 / ERP / 内网服务等）

### 客户端架构特点

* 不生成密钥
* 不生成配置
* 不做策略决策
* 逻辑极轻，仅负责连接与本地路由执行
* **服务端统一生成，客户端只执行**：保证架构稳定、可控、易维护

---

## 🔵 3. 架构统一（Agent 架构：UI → Facade → Controller → EventBus → StateEngine → WGAdapter → WireGuard）

本项目采用 **Bash + Python 组合的 Agent 架构**，将“人机交互”和“内核逻辑”彻底分层，整体调用链为：

```text
UI (Bash)
  ↓
Facade (Python CLI 门面)
  ↓
Controller (业务流程控制器)
  ↓
EventBus (事件总线)
  ↓
StateEngine (状态引擎 / 期望态 vs 实际态)
  ↓
WGAdapter (WireGuard 适配层)
  ↓
WireGuard (系统内核 / wg / ip / iptables)
```

> 设计目标：
>
> * Bash 只负责交互（菜单）与简单入口，不再承载复杂逻辑。
> * 所有真正的业务逻辑、状态管理、系统调用由 Python Agent 完成。
> * 架构可进化：后续换成 Web UI、API、面板，只需替换 UI + Facade，不动底层逻辑。

---

### 3.1 目录结构（Agent 版）

在原有 `/opt/wireguard-manager/` 基础上，重构为：

```text
/opt/wireguard-manager/
├── wireguard.sh                  # UI 层：Bash 主脚本（入口，交互菜单）
│
├── agent/                        # Python Agent（核心逻辑）
│   ├── __init__.py
│   ├── facade.py                 # Facade：对外统一调用入口（CLI / 子命令）
│   ├── controller.py             # Controller：具体业务流程编排
│   ├── event_bus.py              # EventBus：事件总线（发布/订阅）
│   ├── state_engine.py           # StateEngine：状态引擎（期望配置 → 实际系统）
│   ├── wg_adapter.py             # WGAdapter：封装 wg/ip/iptables 等底层命令
│   ├── models.py                 # 数据模型（Server/Client/Route/Config 等）
│   ├── storage.py                # 配置持久化（wg-data/meta.json 等）
│   ├── logger.py                 # 日志中枢（供所有层使用）
│   └── utils.py                  # 通用工具函数（校验/随机端口/MTU 探测/公网 IP 检测等）
│
├── wg-data/                      # WireGuard 数据（与原设计一致）
│   ├── server.key
│   ├── server.pub
│   ├── wg0.conf
│   └── clients/
│       └── client01/
│           ├── private.key
│           ├── public.key
│           ├── config.conf
│           ├── qrcode.png
│           └── meta.json
│
└── logs/
    └── wireguard-manager.log     # 所有层共用的统一日志输出
```

> 原来的 `core/*.sh`、`server/*.sh`、`client/*.sh` 中的逻辑，逐步迁移/收敛到 Python agent 中，由 `wg_adapter.py` 等模块统一管理真正的系统调用。

---

### 3.2 各层职责说明（精细版）

#### 3.2.1 UI 层（Bash：`wireguard.sh`）

**定位：** 纯交互壳，负责与用户对话，所有实质操作都转发给 Python Agent。

**主要职责：**

* 显示交互式菜单（服务端 / 客户端 / 添加客户端 / 查看状态 / 日志等）
* 收集用户输入（比如客户端名称、类型、内网子网）
* 将用户选择转换为统一命令调用 Python：

  ```bash
  python3 -m agent.facade server init
  python3 -m agent.facade server add-client --name client01 --type lan-node --lan-subnet 192.168.50.0/24
  python3 -m agent.facade client import --from-file client01.conf
  ```
* 不直接操作配置文件，不直接调用 wg/ip/iptables

> 作用：
>
> * 把 Bash 限制在“UI”范围，避免 Shell 逻辑越写越乱。
> * 如果未来改成 Web 界面/前端，只要替换 UI，下面整个 Agent 架构不用动。

---

#### 3.2.2 Facade 层（Python：`agent/facade.py`）

**定位：** 对外的统一“门面”，封装所有命令入口，是 Bash / 其他调用者唯一需要认识的 Python 入口。

**主要职责：**

* 解析 CLI 参数 → 统一转换为内部命令
* 做最薄的一层权限与环境检查（例如 Python 版本、当前是否是 root）
* 将命令转发给对应的 Controller 方法

示例接口：

```bash
# 初始化服务端
python3 -m agent.facade server init

# 添加客户端
python3 -m agent.facade server add-client --name client01 --type normal

# 客户端操作
python3 -m agent.facade client status
```

> 作用：
>
> * 隔离 UI 与内部实现细节，避免 UI 直接绑死在 Controller/StateEngine 上。
> * 为未来 REST API / RPC 接口预留空间（Facade 也可以换成 HTTP 层）。

---

#### 3.2.3 Controller 层（Python：`agent/controller.py`）

**定位：** 业务流程 orchestrator，负责“做一件完整事”的流程编排。

**例子：添加客户端（伪流程）**

```text
UI → Facade → Controller.add_client()

Controller.add_client() 做的事情：
1. 校验参数（名称、类型、子网等）
2. 通过 StateEngine 读取当前状态（现有客户端、已用 IP/端口）
3. 生成新客户端的期望状态（desired client object）
4. 发布事件：CLIENT_CREATE_REQUESTED
5. 调用 StateEngine.apply_desired_state()
6. 等待 StateEngine 完成对 WGAdapter 的调用并返回结果
7. 根据结果记录日志，并返回给 Facade / UI
```

**主要职责：**

* 将“业务动作”打包成明确的流程（init server / add client / delete client / reload / route update）
* 与 EventBus 协同，将关键步骤（如客户端创建成功/失败）转化为事件
* 不直接操作文件系统、不直接执行系统命令，这些全都交给 StateEngine + WGAdapter

> 作用：
>
> * 把业务语义（“添加一个 client”）和实现细节分离。
> * 以后要支持 WebHook、审计、回调，只要在 Controller 层挂事件/钩子就行。

---

#### 3.2.4 EventBus 层（Python：`agent/event_bus.py`）

**定位：** 内部事件总线，用于解耦 Controller / StateEngine / 监控组件。

**典型事件：**

* `CLIENT_CREATE_REQUESTED`
* `CLIENT_CREATED`
* `CLIENT_CREATE_FAILED`
* `SERVER_INIT_COMPLETED`
* `WG_RELOADED`
* `HEALTH_CHECK_FAILED`

**职责：**

* 提供订阅/发布（subscribe / publish）接口
* Controller 发布业务事件；
  StateEngine、监控模块、日志模块等可以订阅这些事件
* 可用于：

  * 统计（多少客户端创建成功）
  * 告警（某个操作频繁失败）
  * 将来扩展 Webhook / MQ 等

> 作用：
>
> * 内部组件之间不再直接互相调用，减少耦合度。
> * 方便后续增加“旁路”能力，比如：外部监控/报警/审计。

---

#### 3.2.5 StateEngine 层（Python：`agent/state_engine.py`）

**定位：** 整个系统的“状态大脑”，负责：**期望状态（desired state） → 实际状态（real world）** 的一致性。

**核心概念：**

* Desired State（期望态）：
  从 meta.json / 控制命令得出的应有配置：

  * 有哪些客户端
  * 每个客户端的 IP、端口、AllowedIPs
  * 有哪些子网、路由、NAT 规则
* Actual State（实际态）：
  从系统读取的当前状态：

  * 当前 wg0.conf 内容
  * 当前 iptables 规则
  * 当前 ip route 表
  * 当前 wg show 输出

**职责：**

* 从存储（storage.py）加载当前期望配置（meta.json / wg-data）
* 对比系统实际状态（wg_adapter.get_current_state()）
* 生成差异（diff），决定：

  * 需要新增哪些 Peer / 客户端
  * 需要删除哪些过期配置
  * 需要追加哪些路由 / NAT
* 调用 WGAdapter 进行实际变更
* 支持“自愈”：发现规则丢失时自动补回

> 作用：
>
> * 把整个 WireGuard + 系统网络看作一个“状态机”，而不是一堆命令。
> * 以后要做“声明式配置（像 K8s 那样）”时，只需要扩展 StateEngine。

---

#### 3.2.6 WGAdapter 层（Python：`agent/wg_adapter.py`）

**定位：** 对底层系统命令的统一适配，让上层不直接接触 `wg` / `ip` / `iptables` 等命令。

**职责：**

* 封装：

  * `wg genkey` / `wg pubkey` / `wg show`
  * `wg-quick up/down`
  * `ip addr` / `ip route`
  * `iptables` / `nftables`
* 所有命令必须通过统一的执行函数（带日志 / 超时 / 返回码处理）：

  ```python
  run_cmd(["wg", "show"], timeout=3)
  ```
* 提供语义化方法：

  * `create_interface(config)`
  * `add_peer(peer)`
  * `set_nat_rule(rule)`
  * `enable_forwarding()`

> 作用：
>
> * 如果未来想从 iptables 换成 nftables，只需要在 WGAdapter 中改实现，不动上层。
> * 便于做命令级别的统一日志记录与错误处理。

---

#### 3.2.7 WireGuard 层（系统）

**定位：** 实际的内核模块与用户态工具（`wg`、`wg-quick`），本项目不改动 WireGuard，只是通过 WGAdapter 调用。

---

### 3.3 Bash 与 Python 的分工总结

* **Bash：**

  * 只负责 UI/菜单/交互
  * 调用 Python：`python3 -m agent.facade ...`
  * 不做复杂逻辑，不操作文件、命令

* **Python（Agent）：**

  * 全面负责业务、状态、网络、配置、日志
  * 通过分层架构（Facade → Controller → EventBus → StateEngine → WGAdapter）组织复杂逻辑
  * 易于测试、易于扩展

---

### 3.4 与原有功能要求的映射关系（非常关键）

* 原来文档里的：

  * “服务端初始化” → `Controller.server_init()` + `StateEngine.init_server()` + `WGAdapter.*`
  * “添加客户端” → `Controller.add_client()` → 发布事件 → `StateEngine.apply_desired_state()`
  * “多子网路由 / NAT / 转发” → 封装在 `WGAdapter` 中，由 `StateEngine` 统一调度
  * “智能识别公网 IP / 动态 MTU / 随机端口” → 封装在 `utils.py`，由 Controller/StateEngine 调用
  * “详细日志” → 统一通过 `logger.py`，内部各层都不直接 `print`

这样，你原有的需求全部被“投射”到了 Agent 分层结构中，
**既保留了你当前已经设计好的业务逻辑，又升级到了一个更现代、更可维护的架构。**

---

