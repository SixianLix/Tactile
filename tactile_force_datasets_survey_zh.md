# 力与触觉信息在具身基础模型中的作用、瓶颈与数据硬件路线综述

## TL;DR

- 这条研究线正在经历一条很清晰的演进：`触觉作为额外观测` -> `触觉作为高频闭环反馈` -> `触觉/力作为 VLA 的 token 或预测目标` -> `触觉作为 world model 需要显式建模的 contact dynamics`。
- 现阶段“是否需要力/触觉”的答案不是非黑即白。对一般抓取、搬运、开阔空间运动，视觉和本体感觉已经能覆盖大量需求；但对 `插接、擦拭、剥离、脆弱物体、手内状态、遮挡下操作、受扰恢复` 这类接触丰富任务，触觉/力信息已经被多篇工作证明是关键增益来源。
- 主流具身基础模型过去不建模这类信息，不只是因为“数据不够、硬件不够”，更因为它们还同时面对 `动作条件性强、频率不匹配、传感器异构、跨本体迁移难、评测收益不集中` 这五个约束。
- 最可能的下一步不是“所有事情都交给一个慢速大模型端到端完成”，而是 `VLA 负责语义和长时序意图` + `触觉/力表征或 world model 负责 contact state` + `20-60 Hz 甚至更高频的 reflex / hybrid force-position control 负责闭环执行`。
- 如果用于汇报，建议压成 5 个大主题：`为什么值得做` -> `力与触觉到底怎么表示` -> `系统化落地：数据、采集、硬件与建模实现` -> `核心瓶颈与为什么主流 FM 还没做` -> `最终判断与下一步`。


## 0. 口径、范围与给定论文状态

### 0.1 范围说明

本文围绕你给定的论文与工作，按 10 个问题组织，不按论文逐篇罗列。
为了补齐逻辑链，我额外参考了少量补充工作，例如：
- `TaF-VLA`：把 tactile 从“视觉纹理”转成“力对齐表征”
- `HapticVLA`：训练时学触觉、推理时不依赖触觉硬件
- `Towards Forceful Robotic Foundation Models`：给“反方观点”提供系统性论据


### 0.2 关于 `Vitacworld`

未查到正式题名叫 `Vitacworld` 的论文。本文按最接近、且内容匹配的工作处理为：
- `Visuo-Tactile World Models (VT-WM, arXiv:2602.06001)`


### 0.3 给定论文与工作清单

| 工作 | 时间 | 会议/状态 | 在整条路线中的角色 |
|---|---|---|---|
| `3D-ViTac: Learning Fine-Grained Manipulation with Visuo-Tactile Sensing` (`arXiv:2410.24091`) | `2024-10-31` 提交，`2025-01-06` 修订 | `CoRL 2024` 接收 | 低成本触觉硬件 + 3D visuo-tactile 表示，证明触觉对脆弱物体和手内状态确实重要 |
| `Reactive Diffusion Policy / TactAR` (`arXiv:2503.02881`) | `2025-03-04` 提交，`2025-04-23` 修订 | `RSS 2025` 接收 | 把触觉从“观测补充”推进到“慢-快闭环控制” |
| `Tactile-VLA` (`arXiv:2507.09160`) | `2025-07-12` 提交 | 截至 `2026-06-11` 未查到正式中稿信息 | 研究如何把 VLM/VLA 的语义先验落到触觉感知与力控制上 |
| `TA-VLA` (`arXiv:2509.07962`) | `2025-09-09` 提交 | `CoRL 2025` 接收 | 系统研究 torque 如何进入 VLA：放哪里、用几 token、是否做预测目标 |
| `UniVTAC` (`arXiv:2602.10093`) | `2026-02-10` 提交 | 官方页面写 `Under Review`；获 `ICRA 2026 ViTac Workshop Best Paper Award` | 用仿真做大规模 visuo-tactile 数据生成、表征学习与基准评测 |
| `Visuo-Tactile World Models (VT-WM)` (`arXiv:2602.06001`) | `2026-02-05` 提交 | `Preprint` | 触觉 world model 路线：把 touch 作为接触物理想象与规划的关键模态 |
| `OmniVTA` (`arXiv:2603.19201`) | `2026-03-19` 提交，`2026-03-23` 修订 | `Preprint` / 项目页公开 | 大规模 visuo-tactile-action 数据 + 60 Hz reflexive controller，代表“预测式 contact modeling + 高频闭环” |
| `TAMEn` (`arXiv:2604.07335`) | `2026-04-08` 提交 | `Preprint` | 不只是学策略，而是把“高质量触觉数据怎么采”做成闭环引擎 |
| `Learning Versatile Humanoid Manipulation with Touch Dreaming (HTD)` (`arXiv:2604.13015`) | `2026-04-14` 提交，`2026-04-27` 修订 | `Preprint` | 在 humanoid 上把触觉作为核心模态，并把未来 hand-joint force / tactile latent 作为辅助预测目标 |


### 0.4 补充参考工作

| 工作 | 时间 | 状态 | 用途 |
|---|---|---|---|
| `TaF-VLA` (`arXiv:2601.20321`) | `2026-01-28` | `Preprint` | 说明触觉不是简单图像模态，而可以与 `6-axis force/torque`、`matrix force map` 对齐 |
| `HapticVLA` (`arXiv:2603.15257`) | `2026-03-16` | `Preprint` | 提供“训练时用触觉、部署时不用触觉”的折中思路 |
| `Towards Forceful Robotic Foundation Models` (`arXiv:2504.11827`) | `2025-04-16` | Survey | 为“反方观点”提供系统性论据 |


### 0.5 一句话主线

如果只用一句话概括这批工作：
> 研究重点已经从“给机器人多一个触觉输入”转向“如何把接触动力学变成一个可采集、可表征、可预测、可闭环控制、可接入基础模型的统一问题”。
---

## 1. 价值判断：为什么力与触觉值得重新进入具身基础模型

### 1.1 先给结论：需要，但不是所有任务都同等需要

反方观点并不荒谬：
`Towards Forceful Robotic Foundation Models` 的一个核心判断就是，当前很多 imitation/VLA benchmark 还没有把机器人逼到“非显式力觉不可”的动力学难度；对于很多抓取、搬运、摆放类任务，视觉、本体状态和精心调好的控制器已经足够，额外引入力/触觉的边际收益未必立刻覆盖数据与系统成本。

但正方观点也同样成立：
这批论文共同证明，一旦任务包含 `接触、遮挡、滑移、柔顺性、摩擦、微小力变化、手内状态`，视觉 alone 很快会碰到上限。触觉不是“再加一个模态”，而是 `contact state` 的直接观测。


### 1.2 这批工作通过什么任务证明“视觉不够”

这批工作大致通过三类任务证明必要性：


#### A. 脆弱物体与细粒度力控制

`3D-ViTac` 是最直接的证据。它把任务明确分成两类：
- `需要细粒度力信息`：如 `Egg Steaming`、`Fruit Preparation`
- `需要手内状态信息`：如 `Hex Key Collection`、`Sandwich Serving`

它的任务设计很典型：
- 鸡蛋会因为夹持力过大而破裂，过小又会掉落。
- 葡萄在塑料袋里高度遮挡，视觉看不见是否真正抓住。
- 六角扳手插入前，需要知道它在手里的真实姿态，而不是只知道“手大概在目标附近”。
- 铲蛋、翻勺这种任务，工具会在手里被动旋转，视觉很难持续跟踪手内状态。

`3D-ViTac` 报告，在这些任务上，带 3D tactile points 的策略相对 vision-only baseline 有稳定提升；从表格可以看出，vision-only whole-task success 常落在 `0.45-0.60` 区间，而 visuo-tactile 3D 表示能把关键任务拉到 `0.80-0.85` 左右。这不是边缘改进，而是把“经常失败”推到“多数时候成功”。


#### B. 接触遮挡与世界模型幻觉

`VT-WM` 和 `OmniVTA` 证明的不是“触觉让 grasp 更稳一点”，而是更强的命题：
- vision-only world model 在接触和遮挡下会出现 `object disappearance`、`teleportation`、`violate laws of motion`
- 触觉可以让模型在 imagination 中保持 `object permanence` 和 `causal compliance`

`VT-WM` 的结果特别有代表性：
- `object permanence` 提升约 `33%`
- `laws-of-motion compliance` 提升约 `29%`
- zero-shot real-robot planning 在 contact-rich task 上最高提升约 `35%`

这说明触觉的价值不只是“多一个输入”，而是改变了模型对物理接触的内在表征质量。


#### C. 高频闭环与扰动恢复

`RDP / TactAR` 和 `OmniVTA` 进一步说明，触觉的真正价值常发生在执行时，而不是规划时。

`RDP` 的核心批评非常准确：
很多视觉 imitation learning / VLA 策略靠 `action chunking` 建模复杂行为，但 chunk 执行期间往往是开环的；接触一旦发生偏差，系统不能在足够高频率上及时纠正。

所以它提出：
- 慢层：低频 latent diffusion，负责复杂动作块
- 快层：高频 tactile/force 反馈闭环，负责即时修正

`RDP` 在三类 contact-rich 任务上相对强基线提升超过 `35%`；更重要的是，它还量化了触觉反馈对采集质量的改善：
- peeling length 从 `0.72 -> 0.91`
- stable contact force ratio 从 `0.58 -> 0.87`

也就是说，触觉不只是改善 policy，还改善 demo 本身。

`OmniVTA` 则把这个思路推进到更完整的层级结构：
- slow policy 用 world model 做短时 contact prediction
- fast reflexive controller 用 `60 Hz` 触觉闭环修正执行偏差


### 1.3 在 humanoid 上为什么更需要

`HTD` 把问题又推进了一层。
在 humanoid loco-manipulation 里，接触误差不只影响手，还会影响整个身体执行稳定性。作者明确指出：
- humanoid manipulation 同时要求 whole-body stability、end-effector dexterity、frequent contact changes
- 纯视觉 BC 在接触快速变化时会出现明显盲区

`HTD` 在五个真实接触丰富 humanoid 任务上，相对更强 baseline 平均成功率提升 `90.9%`。
这里最重要的 insight 不是“humanoid 也能用触觉”，而是：
> 接触不确定性会沿着身体耦合传播，所以触觉在 humanoid 上的重要性往往高于固定基座双臂。


### 1.4 对“是否需要”的平衡判断

如果把问题改写得更严格：
- 如果目标是“今天就把仓储 pick-and-place 做到 80 分”，触觉未必是最高 ROI。
- 如果目标是“让一个具身基础模型真正掌握 contact-rich dexterity、遮挡下 manipulation、受扰恢复、细粒度物理交互”，那触觉和力觉不是锦上添花，而是能力边界所在。

我的判断是：
> 现阶段触觉/力觉还不是所有具身公司的第一优先级，但它已经是决定下一代具身基础模型是否真正跨入“物理交互智能”的关键分水岭。
---

## 2. 基本情况：表示、开源数据集与硬件数采设施

这一部分建议在汇报里先回答一个基础问题：力/触觉到底“长什么样”，以及它最终应该被抽象成什么样的中间对象。现有工作已经形成一个相对清晰的判断：真正值得建模的不是原始传感器读数本身，而是由这些读数恢复出来的 `contact state`、`contact dynamics` 和 `future contact consequences`。

### 2.1 力与触觉信息的表示形式总表

| 表示层级 | 典型形式 | 代表工作 | 最适合解决什么问题 | 主要优点 | 主要局限 | 对 FM / VLA 的启发 |
|---|---|---|---|---|---|---|
| 低维力/力矩/本体力觉 | `joint torque`、`6D wrench`、`gripper force`、`hand-joint force` | `TA-VLA`、`RDP`、`HTD` | 接触是否发生、力是否过大/过小、高频闭环修正 | 边际成本低、时间分辨率高、直接服务控制 | 缺少局部接触几何，强依赖本体与标定，跨 embodiment 迁移差 | 更适合作为 `history token`、`decoder-side adapter` 或 `auxiliary prediction target`，而不是单帧静态输入 |
| 触觉图像 / 触觉视频 | `GelSight / Digit / Xense / ViTai / DW-Tac` 图像或视频流 | `VT-WM`、`TAMEn`、`UniVTAC` | 局部形变、滑移、接触轮廓、材料属性 | 与视觉 backbone 兼容，适合 pretraining 和 cross-modal alignment | 跨传感器域差异大，物理含义依赖接触历史与标定 | 可直接 token 化，但最好先学统一 tactile encoder，而不是把 raw frame 直接塞进大模型 |
| 3D deformation / 3D force field | marker deformation、局部形变场、3D force field | `RDP / TactAR` | 跨传感器统一表示、AR 可视化反馈、高频 reactive control | 更接近接触物理，比 raw 图像更适合表达力的方向与变化 | 标定、坐标变换和实时跟踪复杂 | 适合作为 contact physics 的中间表示，而不是最终用户态输入 |
| 3D tactile point cloud | 将 tactile taxel 投影到统一 3D 坐标系中的带强度点云 | `3D-ViTac` | 接触位置与场景几何必须联合建模的任务，如手内状态、遮挡操作、精细插接 | 让视觉和触觉在同一空间对齐，接触位置可解释性强 | 依赖准确运动学与标定，传感器覆盖和分辨率受硬件限制 | 对 contact-rich 操作，触觉更自然的表示有时不是“再加一张图”，而是“把接触点放回 3D 空间” |
| 力对齐表征 / force map | `6-axis force/torque`、`matrix force map`、tactile-force aligned latent | `TaF-VLA` | 把触觉从“局部视觉纹理”转成“显式受力语义” | 能同时保留接触分布与整体受力，更接近 controller 需要的量 | 需要额外标定和多传感器对齐，公开数据仍少 | 适合把 tactile 与 force semantics 统一，再接入 VLA 或 world model |
| token / latent / predictive target | tactile token、torque token、future tactile latent、future torque target | `Tactile-VLA`、`TA-VLA`、`HTD`、`OmniVTA`、`VT-WM`、`UniVTAC` | 把力/触觉纳入 Transformer、VLA、world model 主干 | 便于跨任务复用，能把 raw 读数压成更稳定的 contact state | 可解释性差，传感器语义可能被 latent 吞掉 | 未来主流路线不是直接建模 raw tactile，而是建模 `contact state / contact dynamics / future contact` |

### 2.2 一个更重要的共识：学的不是 raw tactile，而是 `contact state`

从这些工作里可以抽出一个比“传感器类型”更关键的共识：

- `TA-VLA` 不只是喂 torque，还要预测 `future torque`
- `HTD` 不只是喂 touch，还要预测 `future tactile latent`
- `VT-WM` 和 `OmniVTA` 不只是看 touch，还要预测 `contact evolution`
- `RDP` 不只是读触觉，还要把它变成高频 correction signal

所以真正要进入基础模型的，不该只是传感器原始读数，而应是三类中间对象：

- `contact state`：现在是否碰到、碰在什么位置、接触是否稳定
- `contact dynamics`：接触如何随动作演化，是否会滑、卡、破、失稳
- `force consequences`：如果下一步继续推进，力会如何变化
---

### 2.3 开源数据集、硬件与数采设施

这一部分回答一个更现实的问题：即便算法真的需要力/触觉，数据从哪里来，硬件怎么配，成本是否可控，公开数据是否足以支撑后续研究。

#### 2.3.1 采集方式

这一部分回答“数据从哪里来”。当前路线大致分成机器人本体直采、带触觉反馈的遥操作采集、可穿戴 / robot-free 采集和仿真生成四类。

##### A. 机器人本体上直接采

代表：
- `3D-ViTac`
- `VT-WM`
- `OmniVTA`
- `HTD`

典型方式：
- tactile sensor 安装在 gripper / fingertip / dexterous hand 上
- 同步采 RGB / RGB-D、proprioception、action、tactile
- 通过 teleoperation、kinesthetic teaching、gravity compensation 采集

例如：
- `3D-ViTac`：双臂 teleoperation，三台 `RealSense`，四个 tactile sensor pads，数据按时间戳对齐，训练数据采集频率 `10 Hz`
- `OmniVTA`：同时使用 `kinesthetic teaching` 和 `GELLO teleoperation`
- `VT-WM`：`Franka Panda + Allegro Hand + Digit 360 fingertip sensors`


##### B. VR / AR 触觉可视化采集

代表：
- `RDP / TactAR`
- `HTD`
- `TAMEn`

这类方法的关键不是“戴个头显”，而是：
- 人类操作者在采集时能看到或感受到 contact feedback
- 能更自然地采到精细接触动作
- 能在 robot execution 时继续采 recovery data

`RDP` 的 `TactAR` 用 `Meta Quest 3` 把 `3D deformation field` 直接渲染到 AR 里。
`TAMEn` 更进一步，把 recovery teleoperation 做进数据飞轮里。


##### C. 可穿戴 / handheld / robot-free 采集

代表：
- `TAMEn`
- `FreeTacMan`（作为数据层补充，被 `TAMEn` 明确使用）

这条路线的核心动机是：
- 真实机器人采数据太慢、太贵
- tactile data 又必须在接触中产生

所以改成：
- 人手拿着带传感器的 handheld interface
- 记录末端位姿、视觉、触觉
- 再把数据映射回机器人动作空间

`TAMEn` 的“precision mode + portable mode”其实就是在解决这件事：
- precision mode：高保真
- portable mode：低门槛、可扩展、可采 recovery


##### D. 仿真生成

代表：
- `UniVTAC`

它的意义不是替代真实世界，而是把现实里最难扩展的部分先在仿真里规模化：
- 支持三类常见 visuo-tactile sensors
- 自动生成带 supervision 的 synthetic visuo-tactile data
- 提供统一 benchmark 做系统性分析

这条路线很重要，因为“数据不够”在 tactile 里不是一句空话，而是真正限制 FM 的第一瓶颈。


##### E. 当前采集方式的共识变化

从这批工作能看到一个很明显的转向：
> 采集不再被视为“拿到 demo 就行”，而被视为需要包含 `nominal demos + feasibility checking + disturbance cases + recovery data + multi-stage pretraining` 的闭环系统工程。

`TAMEn` 是这点最彻底的体现。
---

#### 2.3.2 硬件方案

可以把当前方案大致分成五类。

##### A. 内置关节力矩 / 腕部 F/T

代表：
- `TA-VLA`
- `RDP`

特点：
- 直接、低维、对闭环友好
- 如果机器人已有估计接口，边际成本最低
- 但容易受本体动力学误差和高速运动噪声影响

`TA-VLA` 的路线很有代表性：
它甚至不依赖外置 F/T sensor，而是用电机电流估算 `joint torque`，说明力觉并不一定等于额外昂贵硬件。


##### B. 光学触觉传感器

代表：
- `GelSight Mini`
- `MCTac`
- `Digit 360`
- `Xense`
- `ViTai`
- `DW-Tac`

特点：
- 分辨率高，能看到形变、滑移、接触轮廓
- 很适合做 tactile image / tactile video / latent encoding
- 但传感器体积、凝胶磨损、光照和标定问题比较突出

这类硬件是 `RDP`、`VT-WM`、`TAMEn`、`UniVTAC` 的主力路线。


##### C. 柔性 taxel 阵列 / 压阻类触觉皮肤

代表：
- `3D-ViTac`

特点：
- 便宜、薄、可弯曲、覆盖面积大
- 适合 gripper surface 和大面积接触
- 空间分辨率和力精度通常低于高端光学触觉

`3D-ViTac` 的意义在于，它证明了这类便宜方案不是“只能凑合用”，而是可以在精细任务上真正产生系统级增益。


##### D. 可穿戴采集与跟踪设备

代表：
- `Meta Quest 3`
- `Pico 4 Ultra`
- `NOKOV` motion capture

它们不是触觉传感器本身，但决定了数据采集是否能规模化。

`RDP` 和 `TAMEn` 的一个共同 insight 是：
> 触觉学习的瓶颈很大程度上不在 sensor 本体，而在“人能不能高效地采出带真实 contact feedback 的好数据”。


##### E. 仿真平台

代表：
- `UniVTAC`
- `TacEx`
- `NVIDIA Isaac Sim / Isaac Lab`

这类方案的价值在于：
- 低成本规模化生成 contact-rich trajectories
- 做跨传感器对比
- 给 tactile encoder / policy / benchmark 提供统一土壤
---

#### 2.3.3 成本：单个硬件成本与系统总成本

##### A. 论文里能确认的可量化成本

| 方案 | 论文/工作 | 公开成本信息 | 含义 |
|---|---|---|---|
| 柔性 taxel 触觉皮肤 | `3D-ViTac` | 单个 `sensor pad + reading board` 约 `20 USD`，`不含 Arduino`；采样约 `32.2 FPS` | 证明 dense tactile skin 的 BOM 可以很低 |
| AR 头显 | `RDP / TactAR` | `Meta Quest 3` 约 `499-500 USD` | 低成本触觉可视化与 teleoperation 入口 |
| 商用光学触觉 | `RDP / TactAR` | `GelSight Mini Robotics Package` 约 `549 USD` | 对研究实验室可接受，但大规模部署仍有成本压力 |
| 自研光学触觉 | `RDP / TactAR` | 改进版 `MCTac` 约 `50 USD` 实验室制造 | 说明 optical tactile 并非必然昂贵 |
| 参考 6D F/T | `RDP / TactAR` | `ATI Mini45` 约 `3000 USD` | 解释了为什么不少工作偏向 tactile 或 joint torque，而不是外置高端 F/T |
| 便携式 recovery 设备 | `TAMEn` | `Pico 4 Ultra` 约 `630 USD` | 便携 recovery teleop 的现实门槛 |
| 双臂 portable 采集系统 | `TAMEn` | dual-arm portable hardware 约 `700+ USD` | 指的是 handheld / VR 侧，不含整套机器人 |


##### B. 更关键的是：单个硬件成本不等于系统成本

这一点经常被低估。

对 tactile/force 系统来说，真正昂贵的往往不是单个 sensor，而是：
- 机器人本体
- 标定
- 同步
- MoCap 或 tracking 基础设施
- 传感器维护与更换
- 数据清洗
- 人工采集时间
- recovery data 的闭环执行成本

例如：
- `3D-ViTac` 的触觉 pad 很便宜，但仍需要 `RealSense`、双臂 teleoperation、时间同步和训练流水线。
- `TAMEn` 的 portable mode 可以低成本部署，但 precision mode 依赖 `NOKOV` motion capture。
- `HTD` 没有公开完整系统 BOM，但 humanoid 本体、dexterous hand、distributed tactile sensing、VR teleop 的总成本显然远高于单个传感器。


##### C. 成本问题的真实结论

所以更准确的结论不是：
- “触觉硬件太贵，所以不现实”

而是：
- `低成本 sensor` 已经出现
- 但 `低成本、大规模、高质量、跨平台` 的触觉数据系统仍然很难

换句话说，瓶颈已经从 `can we build a sensor`，转向 `can we build a scalable tactile data engine`。
---

#### 2.3.4 开源数据：从表征预训练到动作对齐轨迹

##### A. 先分两类：表征数据 vs 动作轨迹数据

如果不分这两类，讨论很容易失真。

- `表征预训练数据`：适合 tactile encoder / tactile-language / tactile VLM
- `动作对齐轨迹数据`：适合 VLA / policy / world model


##### B. 被使用最广泛的原始触觉数据源

| 数据集 | 规模/内容 | 主要表示 | 状态 | 常见用途 |
|---|---|---|---|---|
| `Touch-and-Go (TAG)` | 大规模 in-the-wild GelSight 数据 | tactile image/video + egocentric vision | `released` | tactile pretraining、Touch100k、FoTa、AnyTouch、Sparsh |
| `VisGel` | 机器人触碰 `195` 个物体 | GelSight tactile + RGB video | `released` | visual-tactile pretraining、TVL、FoTa |
| `ObjectFolder-Real` | `100` 个真实物体，多模态 | tactile + visual + audio + mesh | `released` | 多模态物体表征、FoTa、AnyTouch |
| `YCB-Slide` | `10` 个 YCB 物体滑动交互 | DIGIT tactile + pose + RGB + sim labels | `released` | tactile localization、Sparsh、AnyTouch |
| `SSVTP` | `4500` 对严格对齐样本 | spatially aligned visual-tactile pairs | `released` | tactile-language / alignment |

如果只问“目前最常被复用的数据源是什么”，答案仍然是：
- `TAG`
- `VisGel`
- `ObjectFolder-Real`
- `YCB-Slide`


##### C. 聚合型 / foundation tactile 数据集

| 数据集 | 规模/内容 | 主要表示 | 状态 | 作用 |
|---|---|---|---|---|
| `FoTa` | `300万+` tactile images，`13` 类 camera-based tactile sensors，`11` 类任务 | tactile images | `released` | foundation tactile pretraining |
| `TVL` | `44k` vision-tactile-language observations | tactile image + visual image + language | `released` | tactile-language alignment |
| `Touch100k` | `100,147` touch-language-vision entries | tactile image + visual image + language | `released` | tactile-language VLM |
| `VTV150K` | `150k` tactile video frames，`100` objects，`3` sensors | tactile video + attributes + QA | `released` | tactile video reasoning |
| `TacQuad` | `72,606` contact frames，跨 4 类 sensor | tactile image/video + visual + text | `released` | cross-sensor tactile representation |


##### D. 面向具身基础模型更关键的动作对齐数据

| 数据集 | 规模/内容 | 主要表示 | 状态 | 备注 |
|---|---|---|---|---|
| `RH20T` | `110,000+` contact-rich manipulation sequences | RGB/RGB-D/IR + joint angle + joint torque + 6D F/T + audio；部分配置含 fingertip tactile | `released` | 当前最重要的开放 contact-rich robot dataset 之一 |
| `FuSe` | `27k` robot trajectories | vision + touch + audio + proprioception + language + action | `released` | 很适合 VLA 后训练 |
| `FreeTacMan` | `3000k+` visuo-tactile pairs，`10k+` trajectories，`50` tasks | wearable visuo-tactile + end-effector pose + trajectory | `released` | 低成本、大规模 tactile trajectory 采集路线 |
| `OmniViTac` | `21,879` trajectories，`86` tasks，`100+` objects | RGB-D + tactile + action streams | `announced` | 对 world model 很关键 |
| `TaF-Dataset` | `10 million+` synchronized tactile observations | tactile + `6-axis F/T` + `matrix force map` | `to verify` | 力对齐 VLA 的关键补充数据 |


#### 2.3.5 一个必须强调的判断：当前公开数据既不够大，也不够统一，更不够 action-aligned

从数据角度看，当前力/触觉表示至少包括：
- tactile image / tactile video
- taxel pressure map
- 3D deformation field
- 6D force-torque
- joint torque
- gripper force
- tactile latent / tactile token
- language-annotated touch description

这正是为什么“数据不够”不能简单理解为“样本数太少”。
更准确的说法是：
> 目前公开数据既 `不够大`，也 `不够统一`，而且很多数据 `不够 action-aligned`。
---

## 3. 建模与学习：这些信息如何进入 VLA、world model 与 controller

### 3.1 建模与学习路线

如果只看你给定的几篇论文，这一节很容易被理解成 5 条彼此并列的技术路线。但从“综述引用生态”看，需要先做一个区分：前面列到的 `2025-2026` 综述大多是**新综述**，对 FM / VLA / world model 的连接很新，但还谈不上“高引综述”；真正高引、能用来支撑路线完备性的，主要还是更早的 tactile sensing、dexterous hand、LfD 和 contact-rich manipulation 综述。

#### 先回答一个问题：前面那几篇综述是高引综述吗？

不是，至少目前还不是。按 `2026-06-11` 查询到的 OpenAlex `cited_by_count`，它们的引用沉淀还很有限：

- `Towards Forceful Robotic Foundation Models`：约 `0`
- `A Survey on Imitation Learning for Contact-Rich Tasks in Robotics`：约 `1-3`
- `The Developments and Challenges towards Dexterous and Embodied Robotic Manipulation`：约 `1`
- `Survey of learning-based approaches for robotic in-hand manipulation`：约 `18`
- `Towards Robotic Dexterous Hand Intelligence`：过新，公开引用沉淀很少
- `Tactile Robotics: An Outlook`：新近工作，公开索引中的引用统计尚不稳定

所以它们的价值主要在于：**能把话题桥接到最近的 foundation model / VLA / world model 语境**；但如果要保证“路线覆盖完整”，必须补上更早、更高引的基础综述。

#### 如果只看 `2025-2026`，相对影响力最大的几篇触觉 / 力综述有哪些？

严格说，这两年还没有形成传统意义上的“高引综述”。如果把范围限制在 `2025-01-01` 到 `2026-06-11`，更合理的标准是：

- `早期引用量`：看 OpenAlex 的 `cited_by_count`
- `发表状态`：优先正式期刊/接收稿，其次 arXiv
- `主题中心性`：是否直接覆盖 `tactile / force / contact-rich manipulation`
- `作者与机构可核性`：机构、通讯作者都必须能明确查到；机构不明或条目来源不清的，不纳入这张表

按这个口径，并去掉机构或通讯作者不够明确的条目后，更值得补进文档的几篇是：

| 综述 | 时间 / 状态 | 机构 | 通讯作者 | OpenAlex 引用数 | 为什么值得纳入 |
|---|---|---|---|---:|---|
| `Tactile Robotics: An Outlook` | `2025`，`IEEE Transactions on Robotics` 接收/发表 | `King’s College London`、`University of Bristol`、`University of Illinois Urbana-Champaign`、`Queen Mary University of London`、`Technical University of Munich`、`Northeastern University` | `Shan Luo` | `9` | 近两年最像“触觉总纲”的综述之一，强调 `active tactile perception`、仿真大数据和多模态融合 |
| `Compliant Force Control for Robots: A Survey` | `2025`，`Mathematics` | `Southwest Jiaotong University`、`University of Electronic Science and Technology of China`、`Chengdu University` | `Shijie Song` | `10` | 少数直接以 `force control` 为中心、且已有明显早期引用积累的综述，可作为“力控路线”的补充支撑 |
| `A Survey on Imitation Learning for Contact-Rich Tasks in Robotics` | `2025` arXiv，OpenAlex 另索引到 `2026` 文章版 | `Saitama University`、`The University of Tokyo`、`Italian Institute of Technology`、`Purdue University`、`Jožef Stefan Institute`、`Google DeepMind` | `Toshiaki Tsuji` | `3` | 这是最直接把 `contact-rich learning`、`IL/BC/generative methods` 与 tactile/force 场景连起来的综述 |
| `Interactive imitation learning for dexterous robotic manipulation: challenges and perspectives—a survey` | `2025`，`Frontiers in Robotics and AI` | `Karlsruhe Institute of Technology` | `Edgar Welte` | `2` | 更偏 dexterous manipulation + imitation，适合作为“示教策略学习”路线的补充 |
| `Safe Learning for Contact-Rich Robot Tasks: A Survey From Classical Learning-Based Methods to Safe Foundation Models` | `2025`，survey / preprint | `Italian Institute of Technology` | `Heng Zhang` | `1` | 补足了 `contact-rich + safety + foundation model` 视角，适合说明为什么力/触觉在安全约束下更关键 |
| `Towards Forceful Robotic Foundation Models: a Literature Survey` | `2025-04-16`，arXiv survey | `University of Colorado Boulder` | `William Xie` | `0` | 虽然还不高引，但它是少数直接讨论 `forceful robot FM` 的综述，和你的主题最贴近 |
| `The Developments and Challenges towards Dexterous and Embodied Robotic Manipulation: A Survey` | `2025-07-16` 提交，`2025-11-18` 修订，arXiv survey | `Zhejiang University` | `Gaofeng Li` | `1` | 不是纯 tactile/force 综述，但对 `闭环控制 -> embodied intelligent manipulation` 的阶段划分很适合做宏观框架 |
| `Towards Robotic Dexterous Hand Intelligence: A Survey` | `2026-05-13` 提交，`2026-05-15` 修订，arXiv survey | `University of Liverpool`、`Xi’an Jiaotong-Liverpool University`、`Duke Kunshan University`、`The Chinese University of Hong Kong` | `Rui Zhang`、`Kaizhu Huang` | `0` | 这篇过新，但它把 `hand hardware / sensing / learning / datasets / evaluation` 放在一个统一框架里，适合作为 `2026` 年最新视角补充 |

这里有一个很重要的判断：  
如果只看 `2025-2026`，真正**直接聚焦触觉/力**的综述并不多，影响力最大的也还在早期积累期。  
所以更稳的写法不是“近两年已经出现了很多高引触觉/力综述”，而是：
> 近两年真正重要的是出现了几篇把触觉、力控、contact-rich learning、VLA/world model 串起来的新综述；它们的学术影响力还在形成中，但议题设置已经非常关键。

#### 高引综述底座：哪些综述更适合用来保证路线覆盖？

下表中的引用量同样按 `2026-06-11` 的 OpenAlex `cited_by_count` 统计，属于近似值，不同数据库会有小幅差异。

| 综述 | 时间 | 约引用量 | 它覆盖的主路线 |
|---|---:|---:|---|
| `Tactile Sensing—From Humans to Humanoids` | `2009` | `1777` | tactile sensing 总体版图、传感机理、从人到机器的触觉建模 |
| `Tactile sensing for dexterous in-hand manipulation in robotics—A review` | `2011` | `769` | dexterous manipulation、手内操作、触觉在 manipulation 中的作用 |
| `Tactile sensing in dexterous robot hands — Review` | `2015` | `748` | robot hand sensing、触觉硬件与手部 manipulation 的耦合 |
| `Recent Advances in Robot Learning from Demonstration` | `2019` | `717` | LfD / IL 主路线，是 contact-rich learning 的学习底座之一 |
| `A Century of Robotic Hands` | `2019` | `320` | 机器人手的硬件演进、手部能力边界、dexterous manipulation 背景 |
| `Tactile sensing in intelligent robotic manipulation – a review` | `2005` | `280` | 早期 tactile manipulation 总框架，强调触觉在 manipulation 闭环中的位置 |
| `Robot Learning from Demonstration in Robotic Assembly: A Survey` | `2018` | `251` | contact-rich assembly、示教学习与接触任务 |
| `Tactile Sensors for Friction Estimation and Incipient Slip Detection—Toward Dexterous Robotic Manipulation: A Review` | `2018` | `222` | slip / friction / incipient contact，补足局部接触物理感知路线 |
| `Tactile Image Sensors Employing Camera: A Review` | `2019` | `174` | optical tactile / vision-based tactile sensor 路线 |
| `Artificial Sense of Slip—A Review` | `2013` | `119` | slip detection 单独成线，补足“触觉不仅是图像，也是一种局部接触事件检测” |
| `Sensors for Robotic Hands: A Survey of State of the Art` | `2015` | `118` | 手部传感器与 manipulation 系统设计 |

这些高引综述和近两年的新综述放在一起，才能同时回答两个问题：

- 老综述告诉我们：这个领域到底有哪些“完整路线”
- 新综述告诉我们：这些路线今天是如何接到 `foundation model / VLA / world model` 上的

#### 因此，更完整的建模与学习路线不应只有五条，而应至少覆盖八类

1. `经典接触控制与力调节`
   - 这是所有路线的底座，包括 `impedance / admittance / hybrid force-position control`。
   - 高引支撑主要来自 `Tactile sensing in intelligent robotic manipulation`、`A Survey on IL for Contact-Rich Tasks`、`Towards Forceful Robotic Foundation Models`。
   - 它解决的是：机器人在接触发生后，怎样稳定、柔顺、可控地作用于环境。

2. `触觉信号形成与局部接触物理感知`
   - 这一类不是直接学动作，而是恢复 `slip / friction / incipient contact / local geometry / material response`。
   - 高引支撑来自 `Tactile Sensing—From Humans to Humanoids`、`Artificial Sense of Slip—A Review`、`Tactile Sensors for Friction Estimation...`、`Tactile Image Sensors Employing Camera: A Review`。
   - 它解决的是：模型在动作之前，先要知道“接触发生了什么”。

3. `手部与灵巧操作中心的 tactile manipulation`
   - 这一类把触觉放在 robot hand / dexterous hand / in-hand manipulation 场景下理解。
   - 高引支撑来自 `Tactile sensing for dexterous in-hand manipulation in robotics—A review`、`Tactile sensing in dexterous robot hands — Review`、`A Century of Robotic Hands`。
   - 它解决的是：触觉如何与手型、抓取面、手内状态和 finger coordination 耦合。

4. `示教驱动的策略学习`
   - 这是从 manipulation learning 角度最稳定的一条主线，包括 `BC / IL / RL / offline RL`，尤其适用于 contact-rich tasks。
   - 高引支撑来自 `Recent Advances in Robot Learning from Demonstration`、`Robot Learning from Demonstration in Robotic Assembly: A Survey`、`A Survey on Imitation Learning for Contact-Rich Tasks in Robotics`。
   - 它解决的是：在真实接触任务上，如何以可承受的数据成本学到可执行策略。

5. `多模态表征学习与跨模态对齐`
   - 这一类先不急着输出动作，而是把 `vision / touch / force / proprioception` 变成统一的 `contact-aware latent`。
   - 高引支撑来自 tactile sensing 系列综述，近两年则由 `Towards Forceful Robotic Foundation Models`、`Tactile Robotics: An Outlook` 延伸到 foundation-style representation learning。
   - 它解决的是：不同传感器、不同 embodiment 的触觉信息如何被统一表示。

6. `生成式序列建模与 VLA 接入`
   - 这是从点对点动作回归转向 `diffusion / Transformer / VLA adapter / token-based fusion` 的路线。
   - 近年的关键支撑来自 `A Survey on Imitation Learning for Contact-Rich Tasks in Robotics`，它已经把 generative methods 单独列为主类别。
   - 它解决的是：接触操作中的多峰动作分布、长时序技能组合以及如何不破坏已有 VLA 先验。

7. `预测式接触动力学与世界模型`
   - 这是 `VT-WM`、`OmniVTA`、`HTD` 最集中的方向，也是最接近 foundation-model 新阶段的路线。
   - 它学习的不只是当前接触，而是 `future tactile / future force / future contact transition`。
   - 它解决的是：模型能不能提前知道“如果继续这么动，接下来会滑、卡、破，还是稳定接触”。

8. `主动触觉与分层 embodied system`
   - 这是更完整、也更容易被遗漏的一类：触觉不是被动输入，而是主动探索、主动修正、主动闭环的一部分。
   - 高层支撑来自 `Tactile Robotics: An Outlook` 对 `active tactile perception` 的强调，以及 `The Developments and Challenges towards Dexterous and Embodied Robotic Manipulation` 对 embodied intelligent manipulation 的阶段判断。
   - 它解决的是：为什么最终可扩展的系统往往是 `高层语义规划` + `中层 contact state / world model` + `低层 reflex / force control`，而不是一个单块 policy。

把这些路线压缩成一句更稳的结论：
> 触觉/力觉建模的演进，不是从“小传感器”到“大模型”的单线升级，而是从 `接触控制与接触感知`，逐步过渡到 `多模态表征学习`、`生成式策略学习`、`预测式 contact modeling`，最后收敛到 `active touch + hierarchical embodied system` 的系统升级。

在这个更完整的框架下，你给定论文最直接对应的具体路线，可以再压回下面几类近期主线。


#### 路线 A：直接把触觉当额外观测

最直接的做法是：
- 视觉 + 语言 + 本体状态 + tactile/force
- 然后做 BC / diffusion policy / policy finetuning

但这条路线已经显示出明显上限。

`3D-ViTac` 就说明：
- 不是“加了 tactile image 就一定行”
- 更好的做法是把 tactile 转成 `3D tactile points` 再融合

也就是说，关键不只是多一个输入，而是 `输入以什么结构进入模型`。


#### 路线 B：慢-快分层控制

代表：
- `RDP`
- `OmniVTA`

这可能是当前最重要的建模共识。

`RDP`：
- slow latent diffusion 负责复杂动作块
- fast policy 负责高频 tactile/force 修正

`OmniVTA`：
- slow world model / fusion policy
- `60 Hz` reflexive controller

它们共同说明：
> 触觉最有价值的部分往往发生在高频闭环层，而不是低频语义层。

这也解释了为什么“只给 VLA 多拼一个 tactile token”通常不够。


#### 路线 C：作为 VLA 的 token / adapter / auxiliary objective

代表：
- `Tactile-VLA`
- `TA-VLA`

这条路线的核心问题是：
- 如何不破坏 VLA 已有的视觉语言先验
- 又把物理接触信息塞进去

`Tactile-VLA` 的观点是：
- VLM 的 latent space 里已经有一部分“物理交互语义”
- 只要通过少量 tactile demonstrations 把它和真实 force control 连起来，就能激活这部分能力

`TA-VLA` 则把这个问题做成系统设计空间：
- torque 放 encoder 还是 decoder
- 用 single token 还是 multi-token
- immediate torque 还是 history torque
- 只喂输入还是顺带预测未来 torque

它的结论非常清晰：
- `decoder-side` 比 `encoder-side` 更好
- `history` 比 single frame 更重要
- `single aggregated token` 比多 token 更稳
- `predict future torque` 作为 auxiliary objective 会进一步提升性能


#### 路线 D：预测未来触觉 / 力作为学习目标

代表：
- `HTD`
- `TA-VLA`
- `VT-WM`
- `OmniVTA`

这是 2025-2026 最值得注意的变化。

`HTD` 的结论尤其有启发性：
- 直接预测 raw tactile 不如预测 `latent tactile`
- `future hand-joint forces + future tactile latents` 能显著改善成功率

这说明触觉不一定要作为 inference-time 的显式模块存在；它也可以作为训练时迫使模型学会接触动力学的辅助任务。


#### 路线 E：世界模型 / 接触动力学建模

代表：
- `VT-WM`
- `OmniVTA`

这条路线的核心不是“看见触觉”，而是“想象接触”。

`VT-WM` 关注：
- object permanence
- causal compliance
- zero-shot planning

`OmniVTA` 关注：
- short-horizon contact evolution
- predictive tactile target
- closed-loop reflex correction

这一阶段的研究目标，已经不是“让 tactile 帮视觉一点忙”，而是：
> 让模型内部显式携带 contact dynamics。


#### 小结：最有效的不是 raw tactile 输入，而是结构化状态与预测目标

如果把这几条路线放在一起，一个结论越来越稳：
> 最有效的触觉学习，不是“把原始 tactile 数组喂给大模型”，而是把它变成 `结构化状态`、`预测目标`、`高频闭环误差信号`。

如果再把几篇综述的视角叠加起来，这个结论还可以再往上抽一层：
> 未来真正有前景的路线，不是 “tactile-enhanced policy”，而是 `contact-aware embodied stack`：低层有显式接触控制，中层有多模态 contact state / world model，高层才是 VLA / foundation model 负责语义、技能组合和长时序决策。
---

### 3.2 围绕建模与学习，当前前沿在集中解决什么问题

#### 问题 1：如何规模化采集高质量 tactile/force 数据

代表：
- `TAMEn`
- `OmniVTA`
- `UniVTAC`

共同关注的问题：
- 数据量不够
- 任务覆盖不够
- 触觉数据 replayability 差
- recovery data 很难采
- tracking 精度和 portability 很难兼得


#### 问题 2：如何统一异构传感器

代表：
- `RDP`：3D deformation field
- `3D-ViTac`：3D tactile points
- `UniVTAC`：统一仿真三类 visuo-tactile sensors
- `TaF-VLA`：tactile-force alignment

这背后的核心问题是：
- GelSight、Digit、ViTai、Xense、joint torque、F/T wrench 并不是同一种东西


#### 问题 3：如何把触觉接进 VLA / FM，而不是单任务小 policy

代表：
- `Tactile-VLA`
- `TA-VLA`
- `HapticVLA`

这里关注的不是“触觉有没有用”，而是：
- 用 token 还是 adapter
- 放 encoder 还是 decoder
- 是否保留 inference-time tactile hardware
- 是否可以蒸馏掉 tactile sensor，保留 tactile-aware policy


#### 问题 4：如何建模 contact dynamics，而不是只感知 contact

代表：
- `VT-WM`
- `OmniVTA`
- `HTD`

这个问题的本质是：
- 能不能从“看到接触”走到“预测接触会怎样演化”


#### 问题 5：如何在高频闭环中真正用起来

代表：
- `RDP`
- `OmniVTA`

触觉/力的价值往往在 `20-60 Hz` 甚至更高频，而主流 VLA 仍偏低频 chunk-level action。
所以当前工作越来越关注：
- 分层
- reflex control
- hybrid position-force control
- disturbance recovery
---

## 4. 核心瓶颈：为什么主流具身 FM 还没有系统建模力与触觉

### 4.1 数据不够，不只是“不够大”

更具体地说，是五个不够：


#### A. 不够 `action-aligned`

很多触觉数据适合训练 encoder，但不能直接训练 policy 或 world model。
VLA 真正需要的是：
- 视觉
- 语言
- 本体状态
- tactile/force
- action
- timestamp alignment

同时存在的轨迹数据。


#### B. 不够 `contact-diverse`

很多数据覆盖的是：
- 分类
- 属性识别
- 轻量接触

而不是：
- sustained contact
- friction transitions
- sliding
- insertion under misalignment
- disturbance recovery


#### C. 不够 `failure-rich`

现实中最需要触觉的时刻，往往不是 nominal success，而是：
- misalignment
- partial contact
- slip
- excessive force
- failed insertion

`TAMEn` 特别强调 recovery data，说明 failure-rich data 才是 contact-aware policy 的关键燃料。


#### D. 不够 `standardized`

视觉可以统一成 image patches；触觉不行。
现在的数据可能是：
- tactile images
- marker flow
- taxel maps
- torques
- wrenches
- tactile latents

统一难度极高。


#### E. 不够 `benchmark-driven`

当前 benchmark 仍高度碎片化：
- 有的测 grasp success
- 有的测插接
- 有的测 tactile-language QA
- 有的测 world-model object permanence

它们共享“触觉”这个名字，但不共享统一能力目标。


### 4.2 硬件不够，不只是“太贵”

更具体地说，是五个不够：


#### A. 不够标准化

同样叫 tactile sensor，不同设备在：
- 输出形态
- 空间分辨率
- 接触面积
- 帧率
- 安装方式
- 标定方法

上都差别很大。


#### B. 不够耐用

尤其光学触觉传感器，常见问题包括：
- 凝胶磨损
- 标记点漂移
- 光照变化
- 视角/压缩变形


#### C. 不够容易集成

很多 gripper / dexterous hand 本来空间就很紧张。
你加了 sensor 之后，可能：
- 改变接触面几何
- 增加体积
- 影响柔顺性
- 带来布线与供电问题


#### D. 不够容易同步

真正可用的 tactile dataset 要同步：
- RGB / RGB-D
- tactile
- robot state
- action
- sometimes audio / language / tracking

同步成本和工程复杂度被严重低估。


#### E. 不够容易跨 embodiment 迁移

同样是 `5N`，在不同手型、不同覆盖面积、不同柔顺性下，物理语义并不相同。
这是 tactile/force 进入 foundation model 时最难的点之一。


### 4.3 模型与控制层面的两个结构性矛盾

#### A. 频率矛盾

- VLA 喜欢低频、大上下文、chunked action
- 接触控制需要高频、局部、事件驱动


#### B. 语义矛盾

- 视觉和语言更擅长“这是什么、应该做什么”
- 触觉和力觉更擅长“现在是不是碰到了、碰得对不对、下一步会不会滑/卡/破”

这两类信号天然处在不同抽象层。
---

### 4.4 为什么现有 FM 预训练通常不建模这些信息

#### 第一层原因：没有互联网式的被动数据

视觉-语言能吃互联网。
触觉不行。

触觉数据天然是：
- 交互产生的
- 动作条件化的
- 强硬件绑定的

同一个物体，不同接触动作会产生完全不同触觉。
这使它很难像 image-text pair 那样被海量、被动地收集。


#### 第二层原因：模态太异构，难以标准化 tokenization

主流 FM 之所以能 scale，一个关键条件是输入可标准化。
而触觉/力觉目前同时包含：
- tactile image
- tactile video
- 3D deformation
- taxel map
- joint torque
- 6D wrench

这使得“统一 tokenization recipe”远不如图像或文本成熟。


#### 第三层原因：频率不匹配

主流具身基础模型大多建在：
- VLM / LLM 语义 backbone
- 低频 action chunking
- 大时间窗口 reasoning

上。
但触觉/力觉的价值往往在：
- 接触瞬间
- 滑移开始
- 卡住的前 100 ms
- 力过大或过小的微小变化

这类高频局部时刻。

所以很多模型即便“加了 tactile input”，也未必真正吃到了它最关键的价值。


#### 第四层原因：经济收益还不够集中

这是反方观点最强的一点。

对于很多现阶段具身公司：
- 最容易扩规模的是视觉 demo
- 最容易形成产品的是 pick-place / navigation / basic manipulation
- 最容易量产的是少传感器、低维护方案

而触觉/力觉最擅长的任务恰恰是：
- 插接
- 擦拭
- 柔顺物体
- fragile object
- 工具操作
- 受扰恢复

这些任务重要，但未必是所有公司当前最紧迫的商业路径。


#### 第五层原因：很多“力信息”其实被系统隐式吸收了

这点容易被忽略。

很多机器人系统虽然没有显式 tactile token，但仍可能通过以下路径部分吸收物理信息：
- joint torque estimation
- motor current
- controller compliance
- motion failure signatures
- visual after-effects

这会让触觉/力觉的边际收益在某些任务上显得没那么显著。


#### 第六层原因：过去的 benchmark 没有足够惩罚“没有触觉”

如果 benchmark 主要奖励：
- 指令跟随
- open-vocabulary understanding
- broad manipulation coverage

而不奖励：
- contact fidelity
- force regulation
- insertion under occlusion
- slip recovery

那研究资源自然会优先流向视觉语言路线。


#### 但这个结论正在变化

截至 `2026-06-11`，公开主流 embodied FM 仍主要是视觉/语言/本体状态路线，例如：
- `Octo` (`2024`)
- `OpenVLA` (`2024`)
- `GR00T N1` (`2025`)
- `Gemini Robotics` (`2025`)

它们的公开主干都不是触觉/力控优先。

但趋势已经开始变化：
- `Tactile-VLA`
- `TA-VLA`
- `VT-WM`
- `OmniVTA`
- `HTD`

已经说明 tactile/force 正在从“专用策略模块”往“主干表征与预测目标”移动。

商业侧也开始出现信号。例如 Figure 官方在 `2026-01-27` 发布的 `Helix 02` 明确把 `fingertip tactile sensors`、`force-modulated grasping` 纳入系统描述，说明这一方向已经开始从学术走向产品架构层。
---

## 5. 汇报结论：正反观点、核心 insight 与下一步

### 5.1 反方观点是否成立？

成立一半。

反方观点：
1. 只有少数精细操作需要力信息。
2. 需要额外硬件投入，对现阶段公司 ROI 不高。

这在 `2025-2026` 的现实里并不荒谬。
很多 benchmark 和业务场景确实还没逼到“没有触觉就做不成”的程度。


### 5.2 正方观点为什么也成立？

同样成立，而且是长期更强的观点。

1. 从第一性原理看，触觉是物理交互体的原生模态。
2. 从智能角度看，触觉是对接触、摩擦、滑移、柔顺性最直接的观测。
3. 一旦硬件和数据流水线成本被压低，触觉会对 `数据采集方式`、`训练目标`、`控制架构` 形成高杠杆重构。

尤其是当触觉不只提升某个 task success，而是：
- 改善 world model 的物理一致性
- 提升 VLA 的 contact grounding
- 让模型学到更强的 future interaction prediction

那它就不再是一个局部 feature，而可能成为新的数据-算法共设计支点。


### 5.3 这批工作的共同 insight

如果把这些论文放在一起，它们的共同 insight 可以压缩成四条：
1. **触觉/力觉最重要的不是“观测更多”，而是“观测到视觉看不见的接触状态”。**
2. **真正高价值的不是 raw tactile，而是 contact state、contact dynamics、future contact。**
3. **触觉/力觉最应该落在分层系统里，而不是强行塞进一个统一低频大模型。**
4. **数据和算法必须闭环设计；旧的视觉 demo 流水线，无法自然长出好的 tactile foundation model。**


### 5.4 下一步最值得做什么

从研究路线看，下一步最值得做的不是“再做一个触觉小模型”，而是下面四件事：
1. 建立统一的 `visuo-tactile-action schema`
   - 同时保存 raw tactile、校准后的 force/contact labels、robot state、action、task language、failure/recovery tag。

2. 把训练目标从“识别触觉”推进到“预测接触后果”
   - 如 future tactile latent、future force、contact transition、slip onset、contact success/failure。

3. 采用分层架构
   - `慢速 VLA / VLM` 做语义与规划
   - `tactile/force state estimator or world model` 做 contact grounding
   - `20-60 Hz or faster reflex / hybrid controller` 做执行闭环

4. 用仿真和真实世界共同推进
   - `UniVTAC` 这类平台解决规模与标准化
   - `TAMEn / OmniViTac / HTD` 这类真实系统解决接触真实性和 recovery


### 5.5 最终判断

如果问题是：
- “现在是否所有具身基础模型都必须建模力、触觉？”

答案是：
- `还不是`

如果问题是：
- “它是否是下一代具身基础模型最有可能带来本质能力跃迁的模态之一？”

答案是：
- `是`

更具体地说，触觉/力觉最可能改变的不是单一任务性能，而是：
- 模型如何理解接触
- 数据如何被采集
- 控制如何分层
- world model 如何避免接触幻觉

一旦这四件事形成闭环，它就可能不是“增加一个模态”，而是重写一套新的 `数据-表示-控制` 范式。
---

## 附录 A：一个更清晰的时间线

### 阶段 1：触觉作为观测补充

- `3D-ViTac`

关注点：
- 低成本硬件
- 统一 3D 表示
- 证明触觉能显著提升 contact-rich manipulation


### 阶段 2：触觉作为高频闭环反馈

- `RDP / TactAR`

关注点：
- 慢-快控制
- AR 反馈采集
- 数据质量与高频 reactive policy


### 阶段 3：触觉作为 VLA 的 token / 预测目标

- `Tactile-VLA`
- `TA-VLA`
- `HTD`

关注点：
- 如何不破坏 pretrained VLA
- 如何让触觉/torque 进入模型主干
- 如何通过预测 future tactile/torque 学 contact-aware latent


### 阶段 4：触觉作为 world model 的接触动力学

- `VT-WM`
- `OmniVTA`

关注点：
- 物理一致 imagination
- zero-shot planning
- reflexive contact control


### 阶段 5：触觉数据引擎与仿真标准化

- `TAMEn`
- `UniVTAC`

关注点：
- 数据如何规模化
- benchmark 如何统一
- 仿真和真实世界如何共同驱动 tactile foundation model


## 附录 B：这份综述最适合怎么用

如果你接下来要把这份内容写进更正式的综述、proposal 或飞书文档，我建议直接沿用下面这个主判断：
> 当前力/触觉研究的真正分歧不在“它有没有用”，而在“它的收益何时超过其数据与系统成本”。最新一批工作表明，这个拐点正在逼近：触觉已经从一个局部感知增强模态，逐步变成 contact-rich manipulation 中用于数据采集、预测学习、世界建模和高频闭环控制的核心结构性变量。
