# PX4 v1.18.0-rc1 飞控数据链（代码级整理）

把 PX4 的四条数据链——GPS、遥控、MAVLink/QGC、uXRCE-DDS/ROS 2 Offboard——以及它们背后的失效保护，按"数据从哪来、经过谁、到哪去"逐段落到源码文件、函数和行号上。每个结论都标了证据来源，可以直接复核。

- **源码基线**：PX4-Autopilot `v1.18.0-rc1`（commit `fca3df865a`）。文中所有 `文件:行号` 都以这个 tag 为准，`git show v1.18.0-rc1:<path>` 即可对上。
- **运行基线**：`make px4_sitl gz_netdrone`（Gazebo Harmonic）+ `MicroXRCEAgent udp4 -p 8888` + QGroundControl + `ros2 run ax_circular_flight ax_circular_flight`。
- **规模**：96 页 A4，49 张表，12 幅图。

## 阅读入口

| 格式 | 文件 | 说明 |
|---|---|---|
| 在线版 | <https://smalltanxx.github.io/px4-datalink-doc/> | 浏览器直接读，图已内嵌，不用下载 |
| PDF | [PX4_1.18_数据链代码级文档.pdf](PX4_1.18_数据链代码级文档.pdf) | 96 页，打印、引用用这版 |
| Word | [PX4_1.18_数据链代码级文档.docx](PX4_1.18_数据链代码级文档.docx) | 原生目录域，可二次编辑 |
| Markdown | [PX4_1.18_数据链代码级文档.md](PX4_1.18_数据链代码级文档.md) | 源文，便于全文检索与 diff |

## 章节结构

| 章 | 主题 |
|---|---|
| 0 | 结论速览：总体链路图 + 五个不直观的结论 |
| 1 | 实验复现与观测方法 |
| 2 | 启动链路：`make` → `rcS` → `gz` → `gz_bridge` |
| 3 | 仿真链路：Gazebo ↔ PX4 的传感器与执行器边界 |
| 4 | GPS / GNSS 链路：NavSat → `sensor_gps` → EKF2 |
| 5 | 遥控 / 手动控制链路：`input_rc` → `manual_control_setpoint` |
| 6 | MAVLink / QGC 链路：入站、出站与任务/参数 |
| 7 | ROS 2 / uXRCE-DDS 链路：话题版本、QoS、握手时序 |
| 8 | Offboard 控制闭环：设定点 → 姿态 → 角速率 → 电机 |
| 9 | 本次实验的时序：从启动到圆周飞行 |
| 10 | 参数、端口与排障总表 |
| 11 | 真机映射：SITL 的每一段在真机上换成什么 |
| 12 | 代码索引（按链路） |
| 13 | 图表清单 |
| 14 | 链路时延：本地 SITL 测试（非实机） |
| 附录 A–C | 一页纸因果链；未展开的部分；证据与复现原始输出 |

第 8 章分两层：8.1–8.6 是正常控制链路，8.7 是失效保护这一监督层——两者共用 `vehicle_status`，但入口和状态变量不同。

## 图

| 图 | 内容 |
|---|---|
| [00_总体链路.png](diagrams/hires/00_总体链路.png) | 进程 / 端口 / 协议总体拓扑 |
| [01_启动时序.png](diagrams/hires/01_启动时序.png) | `make` → `rcS` → `gz` → `gz_bridge` |
| [02_GPS链路.png](diagrams/hires/02_GPS链路.png) | NavSat → `sensor_gps` → EKF2 → QGC / ROS 2 |
| [03_RC链路.png](diagrams/hires/03_RC链路.png) | RC / QGC 手动输入 → `manual_control_setpoint` |
| [04_MAVLink链路.png](diagrams/hires/04_MAVLink链路.png) | MAVLink 入站 / 出站消息 |
| [05_ROS2链路.png](diagrams/hires/05_ROS2链路.png) | XRCE/DDS 话题版本与 QoS |
| [06_控制闭环.png](diagrams/hires/06_控制闭环.png) | `trajectory_setpoint` → 电机转速 |
| [07_失效保护全链.png](diagrams/hires/07_失效保护全链.png) | 检测 → 决策 → 动作七阶段 |
| [08_电机指令链.png](diagrams/hires/08_电机指令链.png) | `control_allocator` → `FunctionMotors` → `MixingOutput` → Gazebo |
| [09_节点状态机.png](diagrams/hires/09_节点状态机.png) | `ax_circular_flight` 八状态与转移 |
| [10_排障决策树.png](diagrams/hires/10_排障决策树.png) | 按 Offboard 数据流十级排查 |
| [11_失效保护时序.png](diagrams/hires/11_失效保护时序.png) | `FailsafeBase::update()` 一帧之内的 8 步 |

`diagrams/` 下另有可编辑的 drawio 源文件；`diagrams/hires/` 是 4× 导出 PNG（宽 6402 px），正文和 HTML 用的就是这些。

## 第 14 章的实验数据

第 14 章「链路时延」的四个实验（E1 传感器上行、E2 控制链、E3 ROS 2/uXRCE-DDS、
E4 MAVLink）的原始 CSV、采集脚本和逐项推导保存在作者本地 `实验/2026-11-14_latency/`，
未随本仓库发布；需要复核具体数字可联系作者。所有数字均为**本地 SITL 测试，非实机**。

## 证据等级

正文里每个关键结论都标了来源：

- **源码**：`git show v1.18.0-rc1:<path>` 直接可见，行号为 tag 行号；
- **实测**：本次 SITL 里 `param show` / `listener` / `gz topic -i` 的输出，附录 C 给了原始结果；
- **推导**：由源码与参数优先级推出，文中写明"推导"，不当实测写；
- **本地**：来自未提交进 tag 的工作区文件（`4050_gz_netdrone`、`netdrone/model.sdf`、ROS 2 包），行号只对当前工作区成立。

## 复核方式

```bash
git clone --depth 1 --branch v1.18.0-rc1 https://github.com/PX4/PX4-Autopilot.git
cd PX4-Autopilot && git rev-parse HEAD   # fca3df865af36124a28c9d607e850f111dbaaea9
git show v1.18.0-rc1:src/modules/commander/failsafe/framework.cpp | sed -n '78,90p'
```

行号只在这个 tag 下成立；升级 PX4 后可能漂移，但函数名、话题名和调用关系通常稳定。

## 许可与声明

- 文中引用的 PX4 源码片段版权归 PX4 Development Team 所有（BSD-3-Clause）。
- 本文档是独立整理，不是 PX4 官方文档，也不代表 PX4 维护者意见。
- 引用具体结论时，请连同 `文件:行号` 与 tag（`v1.18.0-rc1` / `fca3df865a`）一起引，便于复核。
