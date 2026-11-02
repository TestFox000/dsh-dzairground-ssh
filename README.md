# dsh-dzairground-ssh

> 版本 **0.7.0-s6**（package.json:3）。本文档以代码为准重写（2026-10-24）：每条功能断言都带
> `文件:行号` 出处；旧版（S2/S3 时代）存档于同目录 `README.md.bak-20261024`。

在 DeepSeek Harness 里同时照看两台机器：**dzrobot 小车**（RK3588，默认 `duzhong.local`，
src/store.js:30）与 **dzuav 无人机**（Jetson，默认 `dzsmart.local`，src/nodes.js:41）。
侧栏单图标进入「空地协同」面板（client.js:51,104，注册于 client.js:3210-3223）。

## 边界（先读这个）

- **本插件是「监控 + 仿真 + 导出台」，不发控车指令。** 唯一的下发类出口是仿真台的
  「导出下发指令」（`/sim/route-export`），且是**半步版**：只把一条仿真台账记录翻译成
  车端 `/nav/waypoint` 指令文本 + 发布行 + 证据（status/finished/time_s/routeSource/
  checkExit/engine/configMd5/mapMd5），**不直发车端**——人工核对 evidence 里的 config/map
  md5 与车端一致后，由人在车端手动执行 publish 行（src/simbench.js:786-787,789-819；
  路由 index.js:481-494；页面提示 sim.html:359-361）。
- **SSH 免密是前置。** 探测走 `BatchMode` 非交互登录，免密不成立就停在 `needs-password`
  状态（src/ssh.js:178），不弹密码框、不试密码。设置页有「探测」按钮（client.js:1190-1193）。
- **无人机离线时，各面板给诚实空态，不伪造数据：**
  - 状态浮窗：车/机两半各自轮询、各自失败——机掉线只空机半的格子，车的数字不受影响
    （float.html:12-14；client.js:2114-2115,2241-2246）；飞控电量格按四态显示：
    有数据 / 飞控无数据 / 数据异常 / 无飞控（client.js:1951-1968,2346-2364）。
  - 出车自检：`drone-ssh` 红 = **degraded**（不是 blocked，车仍可单跑）；`ros-domain`
    一端不在线时显灰「读不到域号」（src/preflight.js:66-69,82-84）。
  - 文件 API：节点不可达返回 **503 + `transport:true`**，是可命名的离线，不是 400
    「路径错了」（index.js:263-268）。
  - 无人机 VNC 标签照常能开；连不上时显示桥给的具体原因而不是哑失败（client.js:469-470）。

## 安装与生效

- 真源目录就是本目录（`D:\User\Desktop\ZhiYuan\dsh_plugin\dsh-dzairground-ssh`）。
  desktop profile 以 **`link:` 依赖**挂载本目录，安装/登记/校验由管理工具
  `D:\User\Desktop\ZhiYuan\dsh_plugin\admin.mjs` 完成（link: 是符号链接，不存在
  「同步副本」这一步；**不要**改回 `file:`——那会打包成副本并砸掉链接）。
- 挂载方式：Host 半身由 `cordis.patch.yml` 的一行 insert 挂进 bundle（cordis.patch.yml:5-8）；
  浏览器半身由 `package.json` 的 `dsh.client` 声明（package.json:40-44）。
- **改动生效路径：**

  | 改的是 | 生效方式 |
  | --- | --- |
  | `client.js` / `sim.html` / `vnc-*.html` / `float.html` / `novnc/*`（浏览器半身） | 刷新页面即生效 |
  | `index.js` / `src/*`（Host 半身） | **重启 DSH**（Node ESM 按 URL 缓存，index.js:349-352）；或 `POST /api/dzairground-ssh/reload` 热重载（index.js:348-356，失败回滚旧版，src/reload.js:108） |

- `scripts/deploy.mjs` 是 file: 打包副本时代的同步脚本（scripts/deploy.mjs:1-12），
  link: 挂载下**不需要**也不再是部署路径；留作遗留参考。

## 面板结构

主视图（client.js:2642-2683）自上而下：**协同入口**（COORD_TOOL，外链 sim.html#drone，
client.js:2490）→ 五行工具（TOOLS，client.js:2463-2480）→ 默认折叠的「高级」区
（仿真台 + 64 图压测，client.js:2676-2680,2491）。每行点「打开」：

| 行 | 打开为 | 出处 |
| --- | --- | --- |
| SSH 终端 · 车 / · 机 | 复用 DSH 原生 `terminal` 标签（回显/补全/Ctrl-C/缩放全是应用自带）；Host 拼命令行，Client 写进终端 | client.js:2470-2471；index.js:11-16 |
| 文件目录 | 右栏标签 `dzag-files`：浏览/预览/下载/上传（走已信任的 SSH） | client.js:2472；注册 client.js:3263-3268,3284-3289 |
| VNC 桌面 · 车 / · 机 | 右栏标签 `dzag-vnc-car` / `dzag-vnc-drone`（`keepMounted`，切标签不断桌面；两台可同时开） | client.js:2473-2474；注册 client.js:3237-3261 |
| 状态浮窗 | 不开标签，翻转 `shell.overlay` 上的浮卡开关（不需要会话，启动即可用） | client.js:2475,3299-3306 |
| 仿真台（高级区内） | 外链独立文档 `/api/dzairground-ssh/sim.html` | client.js:2476-2479 |

右栏标签属于会话：点「打开」会先把主视图切到会话再在右侧开标签（面板顶部提示，
client.js:59）。探测失败**不再**挡住终端打开，只给警告（client.js:937,2617）。

**加新工具的方法**（这套结构的目的）：TOOLS 加一行 → `apply()` 里注册标签类型与标签体
（`sidebarRightTabs.register` + `slots.inject('sidebar.right.pane.tab')`，key 必须是类型 id）
→ 字典加文案（client.js:2452-2461,3228-3289）。

**设置页**（Settings →「空地协同」，client.js:3199-3205）：编辑**车节点**的主机/用户/端口/
ssh 别名/附加 ssh 参数、VNC 主机/端口/通道/密码、文件起始目录、浮窗不透明度
（client.js:1206-1260）。无人机的 VNC 目标在此只读展示（取自 `/drone/config`，
client.js:1097-1101）；机节点配置的读写走 `/drone/config` API（节点派发见下）。

## 双节点模型

`src/nodes.js` 是统一节点注册表：`car`（kind=ground，gauges=batteryPercent/cpuPercent/
uptimeSec）与 `drone`（kind=air，gauges=temp/cpuPercent/uptimeSec/batteryPercent）
（src/nodes.js:64-91）。每个节点 = 自己的 settings/status 存储 + 探测 + metrics 采集器
（index.js:127-154）：车的落在 `settings.json`/`status.json`，机的落在
`settings-drone.json`/`status-drone.json`（index.js:88-114）。

路由派发：`/api/dzairground-ssh/car/...` 与 `/drone/...` 跑**同一条 handler 链**、各自打向
本节点状态（index.js:205-227）；无前缀路由保持 S1 形状、默认打向车节点（index.js:216-218）；
未知角色 404 `unknown_node`（index.js:220-223）。`/preflight` 是唯一的例外——它天生跨节点，
只挂在无前缀路由上（index.js:230-235）。

## 连通性诚实契约（probeView 七态）

探测本身有 5 个状态：`ready` / `unreachable` / `needs-password` / `host-key` / `error`
（src/ssh.js:168-211）；`probeView` 再加 `idle`（从未探测）与 `stale`，构成面板圆点/徽标的
七态契约（client.js:537-544）：

- **cached 快照必灰。** `/status` 应答的是持久化快照，并把它标成 `source:'cached'`
  （src/nodes.js:135-137）；只有刚跑完的活探测（`source:'live'`，index.js:137）可以画绿点。
- **超龄必灰。** 即使是 live 探测，年龄超过 **PROBE_STALE_MS = 35 秒**（client.js:530）也降级
  为 `stale`：灰点 + 「上次探测 N 前」，绝不冒充绿（client.js:540-542）。35 s = 一个 30 s
  轮询周期 + 余量（client.js:524-528；后台每 30 s 补探、页面隐藏时不探，client.js:3344-3352）。
- 读到 cached 就立即补一次活探测，不拿昨天的答案装今天的（client.js:2562-2567,3321-3323）。
- 浮窗读数另有 **UPTIME_STALE_MS = 15 秒**（client.js:1903）：断线后标题显「小车未连接」，
  不拿旧锚点继续走字（client.js:1920-1932）。

## 出车自检 /preflight

`GET /api/dzairground-ssh/preflight`（index.js:233-235）返回 `{verdict, checks}`，
**六项检查**（src/preflight.js:131-140）：

| id | 内容 | 四态 |
| --- | --- | --- |
| `car-ssh` / `drone-ssh` | 双节点免密 SSH 活探测 | 绿=就绪（带延迟与目标）；红=失败原因 |
| `ros-domain` | 两端 `ROS_DOMAIN_ID` 对账 | 绿=一致；**红=不一致（blocked 级）**；灰=一端不在线读不到 |
| `vnc-car` / `vnc-drone` | 该端 5901 是否在监听 | 绿=在听；黄=没在听（不影响跑）；灰=节点不在线未检查 |
| `link-telemetry` | 空地链路心跳 | 恒灰「未接入（T12 在办）」——不存在的检查不许装绿 |

verdict 三态（src/preflight.js:63-70）：`car-ssh` 红 **或 `ros-domain` 红** → **blocked**；
其余任意红/黄 → **degraded**（单车仍可跑）；灰不计入 verdict；全绿 → **ready**。

语义：每次调用**并行发起双节点活探测**（不吃缓存快照，src/preflight.js:112-115），且只对
刚答 ready 的节点做一轮**只读**事实采集（`printf` 读 `ROS_DOMAIN_ID` + `ss` 数 5901 监听，
9 s 超时；src/preflight.js:28-30,117-124）——离线节点只耗一次探测超时。所有远程命令均为
只读（src/preflight.js:15-16）。探测结果照例刷新该节点的 status 文件（index.js:136-139）。

## 状态浮窗与读数来源

浮窗挂在 `shell.overlay`（全界面浮层，切视图不滚走；client.js:3299-3306）：一张卡两行，
行标签点名是哪台机器——车行电量/CPU，机行温度/CPU/飞控电量（client.js:2402-2420，
能力表 src/nodes.js:70-89）；标题栏带开机时间（client.js:2384-2392）。面板内浮窗每 **5 s**
同拉 `/metrics` + `/drone/metrics`（client.js:2159,2114-2115）；独立文档
`/api/dzairground-ssh/float.html`（index.js:412-414）每 **3 s** 一刷（float.html:197,204），
可放独立 OS 窗口。不透明度 30–100 持久化（滑杆防抖 POST `/config`，client.js:2206-2219；
夹取 src/store.js:142-150）。

读数为什么敢显示「没有」（缺失词汇表 10 词，src/gauges.js:29-40；UI 映射 client.js:504-516）：

- **车电量**藏在 ROS 后面：底盘节点发 `/Battery_Percentage`（0..1）与 `/PowerVoltage`，
  sysfs 里没有（src/metrics.js:70-71）。栈没跑 → `battery=null` + `batteryNote=no_stack`，
  显示「—」，**绝不编一个 0%**；越界读数 → `bad_value`，显示「数据异常」；只有电压时按
  12 V 满 / 10 V 空折算并标 `derived`（src/metrics.js:105,227-252；client.js:1976-1982）。
- **机温度**来自 `/sys/class/thermal/thermal_zone*` 逐 zone 读取（src/metrics-drone.js:64）。
- **机飞控电量**来自 PX4 经 uXRCE-DDS 的 `/fmu/out/battery_status`，需要串口板 + agent
  两样都在：板在而话题沉默 → `dds_silent`/`no_ros`（显示「飞控无数据」，不是「无飞控」）；
  越界 → `bad_value`；真没板 → `no_fc`（src/metrics-drone.js:71-81,216-257；
  四态渲染 client.js:1951-1968）。

## VNC 桌面

每个节点一条独立 WebSocket 字节隧道：`/api/dzairground-ssh/car/vnc` 与 `/drone/vnc`，
外加 legacy `/vnc`（= 车，旧链接不坏）（index.js:517-545）。桥是纯字节泵，不解析 RFB
（src/vnc.js:15）；面板里的 VNC 标签就是通用连接器文档 `vnc-connect.html`（index.js:409-411）
预填本节点网桥（client.js:2786,3135），改主机/端口/路径可连任意 TigerVNC+websockify 端点。

画面放在 **iframe 独立文档**里：面板页跑在 `dsh-app://` 自定义 scheme 下，动态 import 与
`location.host` 拼 WebSocket 都会失败；Host 用「请求到达的 socket 端口」反推 http origin
（index.js:560-574），iframe 内是普通同源 http。

两个必须知道的运维事实：

- **mDNS 抖 + 地址缓存 + 横幅闸门**：`dialOrdered` 缓存「最近真正答上话的地址」（10 min TTL，
  src/vnc.js:126-128）；车换网后旧地址可能已被同网段别的设备占用，所以转发任何字节之前先读
  对端的 RFB 版本横幅验明正身——不像 VNC 就删缓存、断开、零字节泄漏（src/vnc.js:257,290-293）。
  横幅被扣下后必须补发，否则握手卡死（src/vnc.js:276-278）。
- **TigerVNC 黑名单**：「连上却没完成认证就断开」会被车端 TigerVNC 拉黑（约 1 分钟恢复，
  重复违规翻倍）——所以面板里的 VNC **不会自动连接**，必须点一下；被拉黑时面板直说原因与
  等待时间。想根治可在车端 VNC 单元加 `-BlacklistThreshold=0`（降安全，自行决定）。

车侧现状：TigerVNC 在显示 `:1` 即端口 **5901**（不是 5900），分辨率 1080p；改分辨率用
`scripts/restart-vnc-1080p.sh`（杀 :1 上所有 X 客户端，慎用）。VNC 密码独立于 Linux 登录
密码：设置页填一次即可免输入（src/store.js:51-52），或车上 `vncpasswd` 重设。

## 文件目录

不用 SFTP：每个操作就是一条 `ssh …命令`，复用已认证通道。目录列表用 `find -printf` 的
NUL 分隔字段再 base64 包裹（文件名带空格/引号也能原样往返），读取用 `base64 -w0`，
下载 `cat` 流式、上传 `cat >` 流式（src/files.js；单点收口 `quote()`，src/files.js:39）。读上限
`MAX_READ_BYTES`（2 MB）、写上限 `MAX_UPLOAD_BYTES`（512 MB）（src/files.js:27,30；超限 413，index.js:310-313）。
节点离线回 503 + `transport:true`（index.js:263-268）。UI 只挂车节点一行（client.js:2472）；
API 层 `/drone/files` 等同款可用（节点派发 index.js:212-227）。

## 仿真台（sim.html · 纯 CPU 离线）

默认入口现为**逐格协同仿真**：打开页面只显示固定网格、起终点、隔板、桥洞和无人机起降点；
点击「运行仿真」才生成本局雷 8 格、障碍物 8 格、相邻双格靶板 1 块。桥洞位置沿用固定场地。
生成器按 seed 确定性生成，检查互斥、隔板、桥室及任务点连通性；本局布局随运行记录保存。
这些是场地结构检查，不代表控制器一定能完赛；真实闭环的碰撞、越界和无进展均保留为失败结果。

`POST /sim/session/start` 创建会话，`POST /sim/session/step` 每次推进 **0.5 仿真秒**，
按列往返下发：`N11 → N10 → N9 → N8 → N7 → M7 → M8 → M9 → M10 → M11 → L11 …`，
先扫描 7–11 行的下半场（N 列至 A 列），再从 `A6 → A5 … A1 → B1 … B6` 扫描上半场（A 列至 N 列）。
固定格也发送确认包，全场 154 包共 77 仿真秒。固定元素作为先验，其余 139 格在下发前保持 unknown。
界面区分发送包数和新增已知格数；规划器只读取已知信息，未知格禁止通行；雷沿用必须压过的任务语义。
适配器 `src/cell-session.py` 复用安装工作区的 `StepLoop`、生产段规划、Tracker、Localization、
Mission、Safety 和 `sim_world.World`。规划调用链为 `main.plan_segment_from_layout()` →
`tasks/mission.py::plan_segment_route()` → `route/planner.py::Grid`，算法选择遵循当前 `config/robot.yaml`
的 `planner.engine`（当前为 `tour`），不强制覆盖成 `tracker-safe`，也不调用完整路线入口 `plan_route()`。
每拍内部继续运行控制与物理小步，不会重新从起点跑整局；任务选择使用定位估计和已下发地图，物理真值仅用于仿真与结果判定。
适配层在每次情报被 `DroneTask` 接收后调用生产 `StepLoop._sync_brain()` 同步已知格缓存，
避免行驶中仍用旧规划版本误判已下发格为未知。此同步在运行凭据中单独标记；未知格保护仍开启，外部引擎源文件不变。
未找到可达的已知任务时等待情报；已执行的任务与位姿保留。靶板用于识别与避让，当前不模拟射击评分。

前端提供「小车认知 / 场地实况」、暂停/继续、单步、重跑本局和重新随机。
“重跑本局”保留原布局，使用当前规划器、配置与所选速度；它不是旧代码环境的恢复。
规划统计覆盖候选评估及执行规划的全部调用，分别显示接受、无路、异常及计算耗时；碰撞、越界、控制失败与目标选择耗尽单独分类。
暂停即停止推进会话；正在计算的一拍会完成到下一个 0.5 秒边界。请求携带 revision 防止网络重试重复推进。
`GET /sim/session?id=...` 读取当前状态；`POST /sim/session/stop` 结束会话。
运行记录在 `artifacts/sim/cell/<id>/` 保存布局、完整下发顺序、逐拍事件和终态；
`cell-plans.jsonl` 保存每次规划输入、输出和耗时，`cell-engine.json` 保存实际入口、代码指纹、配置指纹及仿真覆盖参数。
物理终态耗时与每 0.5 秒的界面时钟分别记录，避免用取整时间衡量性能。只写仿真产物目录，不改外部引擎代码及配置。
会话闲置 30 分钟自动释放，达到 1800 仿真秒尚未完成则标记失败。分段协同记录不能导出为整局真机指令。

原有整局仿真、误差注入与手动点位仍作为**高级整局模式**保留；下面 `/sim/run` 等接口描述属于该模式。
新入口的验证命令为 `node scripts/test-session-field.mjs`、`node scripts/test-sim-session.mjs`、
`python scripts/test-cell-session.py`、`node scripts/test-sim-session-ui.mjs` 和 `node scripts/test-sim-session-http.mjs`。
可用 `node scripts/test-sim-session-http.mjs --serve` 启动临时本地验收页面（不探测或连接真机）。
可用 `node scripts/benchmark-cell-session.mjs --seeds 1,2,42 --speed .15 --repeat 2 --max-seconds 1800`
批量测量并比较同布局重跑结果；达到测量上限的运行单独计为未完成，不能据此认定规划器无路。
Host 路由和模块改变后需重载插件或重启 DSH，再刷新仿真页；旧 Host 尚未加载新模块时页面会明确阻止运行。

独立文档 `/api/dzairground-ssh/sim.html`（index.js:416-418），运行根由插件目录的
`sim-root.json` 指到 dzrobot_v2 真源（sim-root.json:1-4）；真源缺失时 `/sim/run` 回 409
并点名怎么补（src/simbench.js:825-831）。**纯 CPU 确定性离线计算：无 GPU 路径、
不 import rclpy、不连小车**（sim.html:240-241）。同一时刻只跑一个仿真（与压测互斥 409，
src/simbench.js:823-824,107-119）。

- **单跑 `POST /sim/run`**（index.js:419-424）：巡航速度档 0.15/0.35/0.40 m/s 或自定义
  （→ `--set pursuit.cruise_speed_mps`，sim.html:162-177）；误差注入 `speed_scale_err` /
  `yaw_drift` / `pos_noise` / `latency` / `anchor`（白名单一一对应 sim_cli 参数，
  src/simbench.js:122-128）；`maxSteps`、`seed`（留空 = SimConfig 默认；同 seed+同参数 =
  逐位同轨迹，sim.html:206-215）、`label`；参数清洗 src/simbench.js:624-678。
- **改点位**：棋盘拖拽为主编辑通道（雷 8 / 障 8 / 桥洞 2 / 靶板 1 定容硬顶，
  sim.html:275-281）；保存只写运行目录 `artifacts/sim/<id>/map_override/`，然后**只读**调
  `tools/plan_route.py` 重规划路线——仓库 `config/` 与 `src/dzbrain/` 一个字节都不写
  （sim.html:316-318；src/simbench.js:660-671）。场地真值 `GET /sim/map`（index.js:439-441）。
  勾选「无人机协同」后车路线硬避让无人机起降格（override 图 obstacles 注入 + 重规划，
  sim.html:222-231；src/simbench.js:654-659）。
- **预检 `POST /sim/check`**：`config_selfcheck` + `route_check` 两个只读脚本
  （src/simbench.js:1010-1031；index.js:432-434）。
- **历史 `GET /sim/history`**：台账 `artifacts/sim/run_meta.jsonl` 追加不覆盖，取最近
  50 条（src/simbench.js:772-778,986-989）；历史表里勾两行可对比；产物图懒加载经
  `GET /sim/artifact`（index.js:425-431），页面带 artifact 请求计数徽标自证懒加载
  （sim.html:386-387）。
- **64 图压测**：`POST /sim/stress` 校验+生图后回 **202 + batchId**，整批后台跑
  （index.js:443-450；默认 64 图、并发池 4，src/stressbench.js:247-253；批 seed 派生
  单图 seed = `(seed*1000+seq) mod 2^31` 可复现，src/stressbench.js:272）；进度与结果走
  `GET /sim/stress/status` / `/sim/stress/result?id=`（index.js:451-459）；页面弹窗展
  64 张路线缩略网格 + 底部均值（sim.html:411-421）。压测期间手动单跑 409
  （src/simbench.js:823-824）。
- **空地协同 `POST /sim/drone/run`**（index.js:461-472）：无人机桥洞优先扫描（0.5 s/格，
  3×3 视场，src/dronesim.js:23,284-287），情报到达触发小车**情报先导的滚动全量重规划**
  ——每半场至多一次、全场 ≤2 次，失败降级走先验默认（src/dronesim.js:289-330）；返回
  timeline/reroutes/confirmed/record + 无人机航线 scan 与情报 intel 供拟真地图渲染。
  协同历史 `GET /sim/drone/history`（最近 50 条 `kind:'drone'`，index.js:473-475；
  src/dronesim.js:397-399）。
- **「最近情报」面板 `GET /intel/latest`**（index.js:477-479）：`source` 诚实三态——
  `sim` = 最近一次协同仿真的 intel.json sidecar；`none` = 一份都没有（诚实空态，不伪造）；
  `live` 预留给 T12 真链路，契约形状不变（src/dronesim.js:401-435）。
- **「导出下发指令」`GET /sim/route-export?id=`**：见上文「边界」——半步版，不直发车端，
  人工执行（src/simbench.js:789-819；sim.html:359-361,2380-2390）。

## HTTP API

前缀 `/api/dzairground-ssh`（index.js:509）；**仅本机回环**可访问（src/trust.js:19,34；
index.js:202-203；VNC 隧道同样只收回环，index.js:642-643）。除 `/preflight` 外，
下列每条都可带 `/car` 或 `/drone` 前缀打向对应节点（index.js:212-227）。

| 方法 | 路径 | 作用 | 出处 |
| --- | --- | --- | --- |
| GET | `/preflight` | 跨节点出车自检（verdict 三态 + 六项四态；仅无前缀） | index.js:233；src/preflight.js:111-141 |
| GET | `/status` | 配置、cached 探测快照、描述、等价命令行、VNC 目标、origin、文件快捷方式 | index.js:236-261 |
| POST | `/probe[?car=0]` | 活探测（`source:'live'`；`car=0` 跳过描述采集） | index.js:319-321,128-140 |
| POST | `/describe` | 采集节点信息（用户/主机名/内核/运行时长） | index.js:322-326 |
| GET | `/metrics` | 浮窗读数（永远 200；节点不可达 = `ok:false` + 原因） | index.js:327-331 |
| GET/POST | `/config` | 读 / 存配置（sanitize 夹取，`floatOpacity` 30–100） | index.js:332-347；src/store.js:115-152 |
| GET | `/files?path=` | 列目录（NUL+base64） | index.js:269-273 |
| GET | `/file?path=` | 读文本（二进制识别而非乱码；上限 MAX_READ_BYTES） | index.js:274-279 |
| GET | `/stat?path=` | 大小/类型/时间 | index.js:280-285 |
| GET | `/download?path=` | 流式下载 | index.js:286-306 |
| POST | `/upload?path=` | 流式上传（超 MAX_UPLOAD_BYTES → 413） | index.js:307-318 |
| POST | `/reload`；GET `/reload` | 热重载 Host 半身（重入 409；失败回滚）/ 查上次重载 | index.js:348-362；src/reload.js |
| GET/POST | `/selftest` | 一次性自检开关（GET 读完即消费） | index.js:363-375 |
| POST/GET | `/diag` | 浏览器半身上报诊断 / 读回（尾部 200 行，`diag.jsonl`） | index.js:376-384,691-707 |
| GET | `/graph` | 查询 web 启动图是否加载了本插件浏览器半身 | index.js:385-401 |
| GET | `/vnc.html` | 小车 VNC 屏幕文档（iframe 嵌入，postMessage 控制） | index.js:404-406 |
| GET | `/vnc-connect.html` | 通用 VNC 连接器文档（noVNC 直连 ws://host:port/path） | index.js:409-411 |
| GET | `/float.html` | 状态浮窗独立文档（3 s 自刷） | index.js:412-414；float.html:197 |
| GET | `/sim.html` | 仿真台页面 | index.js:416-418 |
| POST | `/sim/run` | 跑一局仿真（409=互斥/工作区缺失） | index.js:419-424；src/simbench.js:822-833 |
| GET | `/sim/history` | 仿真台账最近 50 条 | index.js:425-427；src/simbench.js:986-989 |
| GET | `/sim/artifact?path=` | 取某局产物（懒加载） | index.js:428-431；src/simbench.js:991 |
| POST | `/sim/check` | 只读预检（config_selfcheck + route_check） | index.js:432-434；src/simbench.js:1010-1031 |
| GET | `/sim/root` | 仿真工作区路径与可用性 | index.js:435-437 |
| GET | `/sim/map` | 场地真值（画横置地图 + 预填改点位表单） | index.js:439-441 |
| POST | `/sim/stress` | 64 图压测批次（202 + batchId，后台跑） | index.js:443-450；src/stressbench.js:247 |
| GET | `/sim/stress/status` | 压测进度轮询 | index.js:451-453 |
| GET | `/sim/stress/result?id=` | 压测结果（批次/统计/逐图记录） | index.js:454-459 |
| POST | `/sim/drone/run` | 空地协同仿真（timeline/reroutes/scan/intel） | index.js:461-472 |
| GET | `/sim/drone/history` | 协同台账最近 50 条 | index.js:473-475 |
| GET | `/intel/latest` | 最近一份情报（`source` ∈ sim\|none，live 预留 T12） | index.js:477-479；src/dronesim.js:412-435 |
| GET | `/sim/route-export?id=` | 半步版路线导出：生成 `/nav/waypoint` 指令 + 证据，**不直发车端** | index.js:481-494；src/simbench.js:789-819 |
| GET | `/novnc/*` | 自带 noVNC 静态资源（白名单类型；路径穿越 403） | index.js:495-497,605-632 |
| WS | `/car/vnc`、`/drone/vnc` | 每节点 VNC 字节隧道（upgrade 座位） | index.js:523-533 |
| WS | `/vnc` | legacy 车通道（= `/car/vnc`，旧链接不坏） | index.js:534-540 |

以上 `/sim*` 路由共 **13 条**（index.js:416-494；含 `/sim/route-export`）。

## 设置项与数据目录

数据目录 `~/.dsh/dzairground-ssh/`（src/store.js:21）：`settings.json` / `status.json`（车）、
`settings-drone.json` / `status-drone.json`（机）、`diag.jsonl`（诊断，index.js:88-156）。

设置键（src/store.js:28-62，机侧默认 src/nodes.js:39-55）：`host`、`port`、`user`、
`sshAlias`（填了优先于 user@host）、`probeTimeoutMs`、`sshOptions`、`vncHost`（空=跟随 host）、
`vncPort`、`vncMode`（`direct` 直连 / `tunnel` 走 `ssh -L`，src/store.js:45-50）、`vncPassword`、
`filesRoot`、`floatOpacity`（30–100）、`lastConnectedAt`。车默认 `duzhong.local`/`duzhong`/22/5901；
机默认 `dzsmart.local`/`duzhong`/22/5901，`vncPassword` 刻意为空（凭据不进出厂默认，
src/nodes.js:33-51）。默认值假设本机 `~/.ssh/config` 已可免密到车。

## 自带的 noVNC

`novnc/` 是 npm `@novnc/novnc` 1.7.0 **原样**拷贝的 `core/` + `vendor/`，浏览器当原生 ES
模块直接 import，无打包步骤；许可证 MPL-2.0（`novnc/LICENSE.txt`）。升级 = 整目录替换。

## 自检脚本（scripts/）

```bash
# Host/Client/契约
node scripts/test-host.mjs         # Host 半身：全部路由 + 电量解析 + noVNC 资源 + VNC 隧道（本机替身）
node scripts/test-client.mjs       # Client 半身：座位注册与渲染 + 浮窗结构契约 + 完整连接流程
node scripts/test-s2.mjs           # S2/S3 契约：统一 gauge 词汇 + 节点注册表 + B-1 面板布局 + 路由保留数
node scripts/test-preflight.mjs    # 出车自检纯逻辑：事实解析/判定矩阵/双端就绪（stub 节点，无网络）
node scripts/test-intel-route.mjs  # B1/C1 防回归：/intel/latest 三态 + /sim/route-export（含 P1-2 **写侧**：intel.json 落盘/摘要同源/失败不静默）
node scripts/verify-s1.mjs         # S1 验收取证：与旧车插件的同构证明

# 仿真台
node scripts/test-sim-parity.mjs        # 页面重规划 ↔ 车端路线对账 + 引擎口径 + 注入幂等 + 确定性
node scripts/test-sim-map-render.mjs    # 横置地图视图的离线渲染核对
node scripts/test-sim-drone-render.mjs  # 无人机拟真地图的离线渲染核对
node scripts/test-sim-incremental.mjs   # 增量规划透传（from_cell / already_covered）
node scripts/test-sim-overlay-ab.mjs    # 地图「叠加/乱跑」渲染层重复应用的判定性 A/B
node scripts/test-sim-overlay-fix.mjs   # 「乱算+叠加」修复的断言式自检

# VNC
node scripts/test-vnc-guard.mjs    # 缓存地址闸门：非 VNC 对端被拒、零字节转发、失败即清缓存
node scripts/test-vnc-wrapper.mjs  # 浏览器侧 WebSocket 探针能过 noVNC 自己的通道检查
node scripts/test-vnc-connect.mjs  # 连接器文档逻辑无头验证：URL 构造/分辨率/缩放/密码事件
node scripts/test-live-ws.mjs      # 对运行中的插件 + 小车打一发真实 VNC 横幅（显式开启）

# 勘察与一致性
node scripts/check-ssh.mjs         # SSH 层：真实免密登录、描述
node scripts/check-files.mjs       # 文件层：列举/读取/二进制识别/奇怪文件名往返/符号链接
node scripts/check-css-namespace.mjs # CSS 命名空间隔离（两插件同 DOM 不互踩）
node scripts/check-divergence.mjs  # 防「两棵树都像是真源」漂移
node scripts/test-package-files.mjs # 安装位置与源码树逐字节一致（file: 时代的盲区守卫，link: 下恒真）
node scripts/debug-files.mjs       # 目录列不出时打印远程命令原始退出码/stderr
node scripts/debug-ws-handshake.mjs # 浏览器同款请求头打握手，打印原始响应
scripts/ubuntu-vnc-websockify.sh   # 任意 Ubuntu 机装 TigerVNC + websockify（供通用连接器）
scripts/drone-autostart-probe.sh   # 无人机下次在线时的只读盘点（自启前置）
# 车侧（scp 过去再 bash）：probe-car.sh / probe-vnc-config.sh / restart-vnc-1080p.sh /
#   check-vnc-suggestion.sh / vnc-log-1617.sh / vnc-log-1617b.sh / vnc-log-families.sh
```

测试都用临时 `DSH_HOME`，不动真实 `settings.json`。对着真车跑 VNC 测试要显式开
`DZR_TEST_CAR_VNC=1`（黑名单机制下「连上读个横幅就挂」本身是一次认证失败）。

## 改动约定（看 diff 之前先读这三条）

1. **看插件 diff 一律 `git diff --ignore-all-space`**。本仓存量文件是 CRLF
   （`sim.html` 实测 2,772 换行里 2,767 是 CRLF），一次真实改动曾显示成「4587 行假 diff」。
   `.gitattributes` 已把 `*.js/*.mjs/*.html/*.json/*.sh` 钉成 `text eol=lf`，
   但**存量归一单独成一个「纯行尾」提交**，别和功能改动混在一起。
2. **写 `.js`/`.html` 用 `edit`/`write` 工具或 node 脚本**，不许 PowerShell
   `Set-Content`/`Out-File`（那条路会把多行压成一行，注释吞掉代码）。
3. **工作区根只有一份解析器**：`src/workspace-root.js` 的 `resolveSimRoot()`
   （`DZROBOT_SIM_ROOT` → `sim-root.json` 指针 → 向上走 4 级；判据 = `src/dzbrain`
   且 `config/robot.yaml` 都在）。新代码请 import 它，**不要再长第四份内联副本**——
   历史上四份语义不一致，导致"设了 env 仍然测试红"。

## 排错

- **侧栏图标灰点 + 「上次探测 N 前」**：不是新故障——是缓存快照或活探测超龄（>35 s）
  被诚实标灰（client.js:530-544）。面板挂起后会自动补探（client.js:2562-2567）；
  一直灰说明 30 s 轮询连续失败，看设置页「探测」给的 detail。
- **黄点 / 「需要密码」**：免密不成立（src/ssh.js:178）。装公钥：
  `ssh-copy-id -i ~/.ssh/id_rsa.pub duzhong@duzhong.local`。
- **「主机密钥不符」**：确认是换机/换网卡后清 `known_hosts` 里对应行（src/ssh.js:181）。
- **VNC 连不上**：确认车上 `vncserver@:1.service` 在跑、端口是 **5901**；VNC 只监听本机时
  把设置里的通道改成「走 SSH 隧道」（`vncMode: tunnel`，src/store.js:45-50）。
- **VNC 报「不是这台车的 VNC 服务（缓存已失效）」**：车换网、旧地址被占；缓存已自动删除，
  等 mDNS 恢复后重连即可（src/vnc.js:257,290-293）。
- **「连接异常中断」其实是车掉线**：面板显示的是桥自己给的关闭原因；`无法连接 …
  （N 个地址均失败）` = 车不可达；只有 `WebSocket code N` 时看 `diag.jsonl` 同条目的 ws 数组。
- **面板点了没反应**：看 `~/.dsh/dzairground-ssh/diag.jsonl`（每一步都有记录，
  index.js:376-384,691-707）；或在设置页复制等价命令直接进终端粘贴。
- **仿真台 409「工作区不可用」**：安装副本缺 `sim-root.json` 指针——在插件目录放一份
  `{"root":"<dzrobot_v2 绝对路径>"}`（src/simbench.js:825-831；本目录已带 sim-root.json）。

## 已移除（旧 README 有、代码里已无）

- **浏览器控制台 `dzag-console`（Eruda 内嵌调试面板）**：2026-10-09 整条链路移除——标签
  类型/标签体注册、TOOLS 行、`/eruda.js` 路由、`vendor/eruda.min.js`（约 500 KB）与全部
  文案键。排查 JS 错误请用 DSH 自己的开发者工具。
- **小车操作台**（内嵌车端 `cam_viewer.py :8080`）：2026-10-09 随车端工具一起下线。
  它是全车唯一调参通道（`/api/param` 白名单 + 回滚）与可视化遥控（接管仲裁 + 失效保护）
  的 UI；**遥控至今没有替代品**——底盘 1.0 s 无指令强制停车仍在，但「人接管」要自己写节点
  订阅 `/nav/manual_override`。调参改走车端 `config/robot.yaml`。出车前确认这点可接受。
- **file: 打包副本安装 + deploy.mjs 同步流程**：2026-10-23 起由 link: 挂载取代（见「安装与
  生效」）；`scripts/deploy.mjs` 与 `test-package-files.mjs` 保留作遗留与回归守卫。

## 参考：通用 VNC 连接器接外部机器

VNC 标签页的连接器形态同样适用任意 TigerVNC + websockify 端点：协议（ws/wss）、主机、
WebSocket 端口、路径（默认 `websockify`）、密码、分辨率；配置存浏览器 localStorage（密码
也在其中，但永不出现在 URL 里）。自适应缩放 = noVNC `scaleViewport`；固定分辨率经
`SetDesktopSize` 扩展请求远端改分辨率（TigerVNC + RandR；不支持的服务器静默忽略）。
Ubuntu 侧一键装：`scripts/ubuntu-vnc-websockify.sh`（tigervnc + websockify + systemd 单元）。
VNC 协议明文，推荐把 websockify 绑 127.0.0.1 再走 SSH 隧道：`ssh -N -L 6080:127.0.0.1:6080
user@server`，连接器填 `ws://127.0.0.1:6080/websockify`；wss 也支持（协议下拉）。
「车桥」按钮一键把表单指回本节点网桥（client.js:462-468,3135）。
