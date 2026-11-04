# 《Godot + Trae + AI 复刻 FC 像素游戏》教程大纲（v1 草案）

> 教材项目：Crimson_Shadow（FC《忍者龙剑传》风格 2D 横版动作平台跳跃复刻）
> 文档用途：课程结构评审 / 章节拆分讨论 / 后续扩写脚本。请直接在本文上批注。

---

## 一、课程定位

### 1.1 一句话定位

用复活一款 8 位机横版动作游戏作为主线，把 Godot 4 的核心机制、面向对象设计、设计模式和 AI 协作开发串成一条可落地的工程路径。

### 1.2 目标读者

| 读者类型 | 已有基础 | 学完能得到什么 |
|---|---|---|
| 有编程基础、没做过游戏 | 会一门语言 | 能独立用 Godot 做出可玩横版关卡 |
| 会 Godot 基础、做不出完整游戏 | 知道节点和信号 | 掌握状态机 / 组件化 / 数据驱动，能拆解复杂动作游戏 |
| 想用 AI 提效的开发者 | 会写代码 | 掌握「让 AI 读懂项目」的上下文工程与协作流程 |

### 1.3 前置要求

- 装好 Godot 4.x（本项目使用 4.6），会开编辑器、能运行场景。
- 会任一面向对象语言（C#/Java/Python/TS 均可），理解类、继承、接口。
- 不需要像素美术能力：教材使用成品素材，重点在工程实现。

### 1.4 学完的产出

1. 一个可运行的 3 关以内横版复刻 Demo（移动 / 攻击 / 忍术 / 杂兵 / Boss / 存档）。
2. 一套可复用的项目骨架：Autoload 层、组件层、状态机层、资源层。
3. 一份属于自己的 `AGENTS.md`，让 AI 在你的项目里稳定输出。

### 1.5 教学原则

- **先跑通，再抽象**：每章先让功能动起来，再讲为什么要抽成组件 / 状态 / 资源。
- **理论必须有源码落点**：不空讲模式，每个概念都指向教材项目里的真实文件。
- **AI 不是代写**：教「怎么把需求拆成 AI 能执行的粒度」，以及「怎么验证 AI 的产出」。
- **保留 8 位机的约束**：480×270 分辨率、整数缩放、手感优先，不用现代引擎的「什么都自动」。

---

## 二、教材项目全景（写作素材库）

### 2.1 规模与结构

| 项 | 实际数据 |
|---|---|
| GDScript | 656 个 `.gd` |
| 场景 | 364 个 `.tscn` |
| 关卡 | `scenes/levels/` 100+ 个关卡场景（Level_1 ~ Level_9-13）+ `scenes/stages/Level_newgame.tscn` 总入口 |
| 敌人脚本 | 352 个（其中 Boss 相关 210 个） |
| 自定义 Resource 类 | 26 个 `.gd` |
| Autoload | 12 个 |

目录职责：

```
addons/godot_mcp/     编辑器 MCP 插件（让 AI 能被编辑器调用）
assets/               美术、音频原始文件
autoload/             全局单例层（12 个）
characters/           角色：定义 / 数值 / 素材 / 帧动画（long 龍、shinku 真紅）
resources/            自定义 Resource 定义 + 音频事件表（bgm / se / player）
scenes/               levels / stages / player / enemy / ui / effects / environment / item
scripts/              player(28) components(19) enemy(352) environment(20)
                      item(6) ninjutsu(5) projectile(5) ui(11) camera(2) effects(2)
```

### 2.2 全局单例层（Autoload）

| Autoload | 职责 | 可讲的点 |
|---|---|---|
| `UIManager` | 统一弹提示、血条、Boss 血条 | 单例 + 跨场景常驻 |
| `SoundRegistry` | 音效名 → `SoundEventResource` 的注册表 | 注册表模式、别名解耦 |
| `AudioManager` | 播放 BGM/SFX、音量总线 | 服务对象、SFX/BGM 分总线 |
| `LevelManager` | 关卡跳转与入口点 | 场景流控制 |
| `PlayerStateManager` | 跨关卡保存玩家 HP/MP/TP | 会话状态与场景解耦 |
| `SceneTransition` | 转场遮罩与过渡 | 异步流程 + 输入锁 |
| `GameProgress` | 通关进度、解锁状态，存 `user://game_progress.cfg` | 持久化 / ConfigFile |
| `SettingsManager` | 难度、音量、自动连发、语言，存 `user://settings.cfg` | 配置项 + 变更信号 |
| `LocaleManager` | 多语言 | 本地化 |
| `CharacterRegistry` | 角色花名册（long / shinku） | 注册表 + 工厂入口 |
| `CheatManager` | F1 超忍 / F2 切角色 / F3 无敌，带发布开关 | 开发期工具与正式版开关 |
| `MCPRuntimeProbe` | 运行时探测，供 AI 调试 | AI 工具链 |

### 2.3 玩家模块（组件化 + 状态机）

- 主节点 `scripts/player/Player.gd`：只做编排，把输入、移动、动画、忍术、剑、手里剑拆成组件。
- 组件：`InputComponent`（输入 + 跳跃/攻击输入缓冲）、`MovementComponent`（重力/加速/冰面/跳跃）、`AnimationComponent`（统一播动画 + 兜底）、`SwordComponent`、`NinjutsuComponent`、`ShurikenComponent`、`PlayerAudioComponent`、各类 HitBox/HurtBox。
- 状态机：`scripts/player/states/StateMachine.gd` 自动收集子节点中的 `State`，`State.gd` 定义 `enter / exit / update / physics_update` 生命周期，共 24 个状态（Idle/Run/Jump/Fall/Wall/Hang/Crouch/Hurt/各类攻击/必杀/灭杀链/收刀等）。

### 2.4 敌人与 Boss

- 杂兵基类 `scripts/enemy/l1/BaseEnemy.gd`（`CharacterBody2D`）：统一朝向、受伤、死亡流程，子类只写自己的 AI 行为。
- 敌人按关卡分目录：`scripts/enemy/l1 ~ l9`，每个敌人配套 `resources/enemy/*Data.gd` 数据资源。
- 数据类继承树：`BaseEnemyData` → `ChaserNinjaData / CrouchNinjaData / PatrolEnemyData / PatrolNinjaData / SoldierData`；`BossData` → `BossData_2 ~ BossData_10`。
- Boss：`Boss.gd` + `BossStateMachine.gd` + `BossState.gd` 基类 + 11 个通用 Boss 状态；`BossAIComponent.gd` 用「距离/玩家是否在空中/血量阶段」计算动作权重表。
- 最终 Boss 是特殊拆分案例：`boss10_*` 系列（手、眼、黑洞、火雨、黑暗激光、屏障充能……）共 210 个脚本。

### 2.5 表现层与流程层

- 音频：`resources/audio/SoundEventResource.gd`，每个音效一个 `.tres`（`se/baoqi.tres`、`player/gongji.tres` 等）；角色可覆盖音效（真紅把部分语音换成女性版），实现「一套逻辑两套声音」。
- 总线：`Master / SFX / BGM` 三条。
- 输入：14 个动作（`nav_left/right/up/down`、`jump`、`attack`、`finishing_move`、`ninjutsu`、`switch_ninjutsu`、`special_move`、`pass`、`exterminate`、`ui_confirm`、`ui_cancel`）。
- 物理分组（global_group）：`enemy`、`fall_vulnerable`、`blockable`、`wall_blocker`、`projectile`。
- 显示：480×270 viewport，`stretch/mode = viewport`，窗口 1280×720。
- 场景元素：`TileMapLayer`（102 个场景使用）、`ClimbableWall`、`moving_platform`、`IceGround`、`wind_zone`、`DeathZone`、`TransitionZone`、`BossGateTrigger`。

### 2.6 值得写进教程的 12 个实战案例（均绑定真实文件）

1. 480×270 像素画工程配置：`project.godot` 的 `[display]` 段如何决定画面清晰度。
2. 输入缓冲手感：`components/InputComponent.gd` 的 jump/attack buffer。
3. 移动手感数学：`components/MovementComponent.gd`（加速、冰面滑行、重力、跳跃）。
4. 状态机落地：`states/StateMachine.gd` + `State.gd` + `IdleState.gd`。
5. 复杂技能拆状态：`states/DragonFlashState.gd` 的 fade-in → 斩击波 → fade-out 三段。
6. 组件编排：`Player.gd` 如何在 `_process` 里按顺序驱动组件。
7. 伤害判定体系：`components/HurtBox.gd` + `AttackHitBox.gd`，靠 signal 解耦、靠 group 过滤。
8. 敌人基类模板方法：`BaseEnemy.gd` 统一死亡、子类只写 AI。
9. 数据驱动敌人：`resources/enemy/BaseEnemyData.gd` 及子类，Inspector 绑定数据。
10. Boss 权重 AI：`components/BossAIComponent.gd` 的权重表 + 血量阶段。
11. 弹幕对象池：`components/projectile_pool.gd`（`prewarm/acquire/release` + 可选钩子方法）。
12. 角色注册表与热切换：`autoload/CharacterRegistry.gd` + `player/PlayerSpawn.gd` + `CharacterDefinition.gd`。

---

## 三、课程结构：每章固定五拍

1. **理论（30%）**：概念、为什么需要、常见错误做法。
2. **源码解剖（30%）**：教材项目真实文件、关键行、设计取舍。
3. **动手实战（30%）**：当堂实现一个最小可跑版本。
4. **AI 协作（10%）**：给 Trae 的提示词模板 + 验收方式。
5. **检查清单**：自查项 + 常见坑。

---

## 四、章节大纲

### 第 0 篇 起步与工程底座

**第 0 章 课程地图与复刻方法论**
- 理论：什么叫「复刻」（机制对拍 ≠ 美术临摹）、参考游戏拆解法、垂直切片。
- 实战：跑起 `scenes/stages/Level_newgame.tscn`，用 F1/F2/F3 观察系统边界。
- AI：如何让 AI 先「读项目」再「写代码」。

**第 1 章 Godot 工程与像素画基础**
- 理论：Project Settings、渲染器选择、viewport 与 window 的区别、整数缩放。
- 实战：从零配置 480×270 + `stretch/mode=viewport` + 1280×720 窗口；解释「编辑器内嵌窗口」为什么不是全屏。
- 规范：目录先行约定（assets / scripts / scenes / resources / autoload）。

**第 2 章 节点、场景与生命周期**
- 理论：节点树、场景即预制体、`_ready/_process/_physics_process` 顺序、`@onready` 与 `$Path`。
- 实战：做一个会跑会跳的最小 Player（CharacterBody2D + AnimatedSprite2D + CollisionShape2D）。
- 源码：`Player.gd` 的 `_ready` 初始化顺序、`@onready` 组件引用。

**第 3 章 信号与解耦**
- 理论：观察者模式、signal 的强/弱连接、`set_deferred` 解决物理回调里改状态的问题。
- 实战：HurtBox → took_damage → 敌人扣血 → 死亡通知。
- 源码：`BaseEnemy.gd` 的伤害链路；`SettingsManager` 的 `difficulty_changed` 等变更信号。

**第 4 章 自定义 Resource 与数据驱动**
- 理论：Resource 是「可存档的类」、检查器编辑、`class_name` 注册、资源复用。
- 实战：把敌人血量/速度/音效搬进 `BaseEnemyData`，用继承派生 5 种杂兵数据。
- 源码：`resources/enemy/`、`resources/player/PlayerData.gd`、`resources/audio/SoundEventResource.gd`。

**第 5 章 Autoload 与全局服务**
- 理论：单例的边界（什么时候该用、什么时候是灾难）、服务对象 vs 上帝类。
- 实战：写一个 `AudioManager` + `SoundRegistry`，音效名与文件路径解耦。
- 源码：12 个 autoload 的职责切分；为什么 `PlayerStateManager` 要独立于 Player。

### 第 1 篇 面向对象与代码规范

**第 6 章 继承、组合与「什么时候该拆」**
- 理论：is-a 与 has-a、继承层级过深的信号、组合优先原则。
- 实战：对比 `BaseEnemy` 继承方案与 `Player` 组件方案，回答「为什么敌人用继承、玩家用组合」。
- 源码：`BaseEnemy` 派生 5 类杂兵；`Player.gd` + 7 个组件。

**第 7 章 封装与多态**
- 理论：封装不是「加 getter」，而是收窄可变状态；多态在 GDScript 里怎么落地（虚方法 + 重写 + duck typing）。
- 实战：`State.gd` 定义空实现，24 个状态各自重写；`BossData_2..10` 覆盖基类行为。
- 源码：`states/State.gd`、`states/StateMachine.gd`、`resources/enemy/boss/BossData.gd`。

**第 8 章 依赖注入与「谁认识谁」**
- 理论：`@export` 注入、场景内引用 vs 全局查找、避免 `get_node("../../..")`。
- 实战：把硬编码的敌人参数改为 Inspector 绑定数据资源；把全局名查找改为 Autoload 注入。
- 源码：`BaseEnemy.gd` 的 `@export var data`；`CharacterDefinition` 由 `PlayerSpawn` 注入 Player。

**第 9 章 代码规范与可维护性**
- 理论：命名约定、类型标注、常量集中、注释分层。
- 实战：制定本项目规范并写进 `AGENTS.md`：
  - 文件头写 `res://` 路径注释；分节线 `# ── xxx ──`
  - 常量 `UPPER_SNAKE`、函数/变量 `snake_case`、类名 `PascalCase`
  - signal 命名 `xxx_changed` / `xxx_requested`
  - 写路径用 `&"idle"`（StringName）避免字符串开销
  - 发布开关集中在 `CheatManager`，正式版一键关闭
- 源码：`CheatManager.gd` 顶部开关区、`projectile_pool.gd` 的使用约定注释。

### 第 2 篇 设计模式（每个模式都配源码 + 实战）

**第 10 章 状态机（本项目第一模式）**
- 理论：有限状态机 vs 行为树 vs 分层状态；状态退出/进入必须成对。
- 实战：给 Player 写 Idle/Run/Jump/Fall 四状态，再扩展 Wall/Attack。
- 源码：`StateMachine.gd` 自动收集子节点；`DragonFlashState.gd` 三段式技能；Boss 版 `BossStateMachine.gd`。

**第 11 章 组件模式**
- 理论：把「会变的维度」拆成组件；组件间通信走主节点。
- 实战：拆出 Input/Movement/Animation 三组件并让 Player 只做编排。
- 源码：`Player.gd` 的 `_process` 驱动顺序；`AnimationComponent` 的容错播放。

**第 12 章 注册表与工厂**
- 理论：注册表解决「按名字拿对象」；工厂解决「按配置造对象」。
- 实战：`CharacterRegistry` 花名册 + `PlayerSpawn` 按角色定义生成玩家；再扩展到 Boss 刷新器。
- 源码：`autoload/CharacterRegistry.gd`、`scripts/player/PlayerSpawn.gd`、`scripts/enemy/boss/BossSpawner.gd`。

**第 13 章 模板方法与策略**
- 理论：模板方法固定骨架、策略替换算法。
- 实战：`BaseEnemy` 固定「受击→死亡」骨架；`BossAIComponent` 用权重表当策略，按血量阶段换策略。
- 源码：`BaseEnemy.gd`、`BossAIComponent.gd`（权重摇号）。

**第 14 章 对象池与性能**
- 理论：节点创建/销毁的成本、池化适用边界（高频短命对象）。
- 实战：弹幕池化，对比池化前后的帧率与 GC 抖动。
- 源码：`components/projectile_pool.gd` 及 `activate_from_pool` 约定。

**第 15 章 观察者与事件总线**
- 理论：signal 与全局事件总线的区别；什么时候该引入「事件中心」。
- 实战：用信号替换直接引用，串联「设置改变 → UI/音频/玩家」三方响应。
- 源码：`SettingsManager` 的 5 个变更信号；`CheatManager.switch_character_requested`。

### 第 3 篇 复刻实战：从手感到底层战斗

**第 16 章 平台跳跃手感**
- 理论：重力曲线、最大速度、加速度、空中控制、输入缓冲、可变跳跃高度。
- 实战：调出「像 FC」的手感；用输入缓冲修掉「明明按了却没跳」。
- 源码：`MovementComponent.gd`、`InputComponent.gd`（jump/attack buffer）。

**第 17 章 攻击、判定与无敌帧**
- 理论：HitBox/HurtBox 模型、主动/被动碰撞、group 过滤、无敌帧与受击硬直。
- 实战：实现「挥刀命中 → 敌人扣血 → 玩家受伤闪白 → 击退」。
- 源码：`AttackHitBox.gd`、`HurtBox.gd`、`SwordHitBox.gd`、`EnemyHitBox.gd`。

**第 18 章 忍术与投射物**
- 理论：投射物生命周期、发射器与弹体分离、对象池接入。
- 实战：手里剑 + 火球 + 长刀波三种弹体，共用同一套池与命中逻辑。
- 源码：`scripts/ninjutsu/`（5 个）、`scripts/projectile/`（5 个）、`ShurikenComponent`。

**第 19 章 杂兵 AI**
- 理论：巡逻、追击、飞行、远程的差异建模；AI 参数外置。
- 实战：用 `BaseEnemy` + 数据资源派生 5 种杂兵，尽量不复制粘贴。
- 源码：`scripts/enemy/l1/` 系列 + `resources/enemy/*Data.gd`。

**第 20 章 Boss 设计**
- 理论：阶段化血量、动作权重、预警（telegraph）、死亡表演。
- 实战：做一个两阶段 Boss（近战突进 + 激光），再接最终 Boss 的拆分思路。
- 源码：`scripts/enemy/boss/Boss*.gd`、`BossDeathDirector.gd`、`boss10_*` 家族的分层。

**第 21 章 关卡、相机与场景流程**
- 理论：关卡单元（Level_1-2 式命名）、相机边界、转场锁输入、检查点。
- 实战：搭一关包含攀爬墙、移动平台、死亡区、Boss 门触发。
- 源码：`CameraController.gd`、`CameraRegion.gd`、`LevelManager.gd`、`SceneTransition.gd`、`environment/` 20 个脚本。

**第 22 章 存档与进度**
- 理论：全局进度（解锁）vs 会话状态（当前 HP）分层持久化。
- 实战：用 ConfigFile 保存难度、解锁、音量，并能读取恢复。
- 源码：`GameProgress.gd`、`SettingsManager.gd`、`PlayerStateManager.gd`。

### 第 4 篇 表现层与工程化

**第 23 章 音频架构**
- 理论：BGM 与 SFX 分总线、音效资源化、随机变体避免疲劳。
- 实战：`SoundEventResource` + `SoundRegistry` + 角色音效覆盖（男/女忍者同一套逻辑两套声音）。
- 源码：`resources/audio/`、`CharacterAssets.gd` 的 `sound_overrides`。

**第 24 章 UI 与本地化**
- 理论：CanvasLayer、UI 与游戏逻辑解耦、字体与像素对齐。
- 实战：标题 / 暂停 / 选关 / 设置 / 结算五个界面，统一由 `UIManager` 调度。
- 源码：`scripts/ui/` 11 个脚本、`LocaleManager`。

**第 25 章 调试与作弊工具**
- 理论：开发期功能必须有开关，正式版零残留。
- 实战：F1 超忍、F2 切角色、F3 无敌；热切换角色不重建节点、保留状态。
- 源码：`CheatManager.gd` 发布开关、`Player._apply_character()`。

**第 26 章 Trae + AI 协作工作流**
- 理论：上下文工程（Context Engineering）——AI 的产出质量取决于你给的约束。
- 实战：
  - 装 `addons/godot_mcp`（编辑器 MCP：node/scene/script/resource/project/editor/debug 工具集），让 AI 能直接操作场景与资源。
  - 写 `AGENTS.md`：目录约定、命名规范、禁止事项（例如「不要改动 `addons/`」「新功能必须带发布开关」）。
  - 任务拆解粒度：一次只让 AI 做「一个模式 / 一个状态 / 一个组件」。
  - 验证闭环：让 AI 给出「改哪个文件、哪一行、怎么验收」。
  - 反面案例：AI 幻觉造 API、越权重构、悄悄改配置。

**第 27 章 综合项目：从零复刻第一关**
- 需求 → 拆解 → 骨架 → 手感 → 战斗 → 敌人 → Boss → 存档 → 打包。
- 验收标准：能跑通 1 关，60FPS 稳定，无内存增长，存档可恢复。
- 交付：可执行包 + 源码 + `AGENTS.md` + 复盘文档。

### 附录

- A. 术语表（viewport / 拉伸模式 / 状态机 / 组件 / 资源驱动 / 对象池 / 权重 AI）
- B. 常见坑清单（拉伸模式选错、`_ready` 顺序、物理回调里改状态、信号重复连接、资源被共享修改、AI 改坏 project.godot）
- C. 每章检查清单模板
- D. 提示词模板库（新建状态 / 新建敌人 / 重构组件 / 排查报错 / 生成关卡数据）
- E. 参考资料（Godot 官方文档对应章节）

---

## 五、章节依赖关系

```
第 0-5 章（底座：工程 / 节点 / 信号 / 资源 / 单例）
        ↓
第 6-9 章（OOP 与规范：继承组合 / 多态 / 注入 / 命名）
        ↓
第 10-15 章（模式：状态机 / 组件 / 注册表工厂 / 模板策略 / 对象池 / 观察者）
        ↓
第 16-22 章（实战：手感 / 战斗 / 忍术 / 杂兵 / Boss / 关卡 / 存档）
        ↓
第 23-25 章（表现层与工具）
        ↓
第 26-27 章（AI 工作流 + 综合项目）
```

弹性建议：
- 只想做小游戏：0-5 + 10 + 11 + 16 + 17 + 21。
- 想学架构：0-15 全做，实战只做第 27 章。
- 想学 AI 协作：0-3 + 26-27，其余作为阅读材料。

---

## 六、每章统一交付物

| 交付物 | 说明 |
|---|---|
| 可运行场景 | 每章结束必须能跑，且不破坏前面章节 |
| 讲义正文 | 理论 + 源码解剖 + 步骤 |
| 代码对照 | 「教材项目原版」与「学员版本」差异说明 |
| 提示词 | 本章对应的一段 Trae 提示词 |
| 检查清单 | 5-10 条自查项 |
| 常见错误 | 3 条以上真实踩坑 |

---

## 七、待确认事项（请批注）

1. **受众定位**：以「会编程但没做过游戏」为主，还是「会 Godot 但做不出完整项目」为主？两者章节配比不同。
2. **教材语言**：GDScript 单语言，还是附带 C# 对照？
3. **实战载体**：直接用 Crimson_Shadow 精简版做教材，还是另建一个 `NinjaClone_Demo` 骨架工程？（建议后者：教材仓库保持「章节 = 分支」）
4. **是否包含美术流程**（aseprite → 导入 → SpriteFrames 切片），还是只讲工程侧？
5. **AI 部分深度**：只讲 Trae 对话式开发，还是展开 MCP 工具链与自动化？
6. **配套形式**：纯图文，还是需要每章短视频 / 可交互 HTML 讲义？
