# 逻辑与表现分离玩法架构方案

## 0. 文档定位

本文是当前阶段关于“游戏玩法中逻辑与表现分离、Replay、Do/Undo、ECA、OperationBatch”等问题的滚动总结文档。

它的目的不是记录每一次原始讨论的完整历史，而是沉淀当前已经修正后的有效结论。后续如果继续讨论并产生更好的方案，新文档应继续沿用这个原则：

```text
新文档 = 旧文档有效结论 + 新讨论形成的修正结论 + 新增共识
```

不再采用的旧思路不需要完整保留。只有当某个“不要这样做”的结论对后续设计仍有提醒价值时，才以简短约束的形式保留。

本阶段仍处在架构方案讨论期，暂不展开以下内容：

- 具体类图
- 正向执行时序图
- Undo/Redo 时序图
- 最小伪代码骨架
- 具体工程目录与类接口设计

本文重点回答：

- 为什么要做逻辑与表现分离
- Replay 与 Undo/Redo 的本质区别
- Command、Trigger、DomainEvent、StateDiff 的边界
- ECA 如何与可撤销状态修改体系结合
- AtomicOperation 与 OperationBatch 的职责
- 表现层如何接入正向表现、反向表现与异步播放
- 中大型 Unity 玩法项目落地时需要提前约束哪些问题

---

## 1. 核心目标

这套方案要服务的是“玩法核心规则”的长期可维护性，而不只是目录上的分层。

目标包括：

- 逻辑与表现分离
- 逻辑层可独立测试
- 支持 Replay
- 支持 Undo/Redo
- 支持复杂规则连锁
- 支持表现异步播放
- 支持存档与调试
- 未来可扩展到 AI、自动化测试、网络同步或服务器校验

核心思想可以概括为：

```text
输入数据化
规则确定化
状态修改原子化
一次操作事务化
表现结果投影化
外部副作用边界化
```

---

## 2. 逻辑与表现分离真正要分离什么

逻辑与表现分离，真正要分离的不是文件夹，也不是类名，而是职责和权力。

### 2.1 真状态

真状态属于逻辑层。

包括：

- 胜负
- 资源
- 步数
- 位置
- CD
- 任务进度
- 关卡状态
- 生成与删除
- 合成、消除、碰撞、命中等规则结果
- 随机结果
- ID 分配状态

这些数据决定游戏“事实上发生了什么”。

### 2.2 表现状态

表现状态属于表现层。

包括：

- 动画进度
- 特效存活
- 音效播放
- 镜头状态
- UI 展开状态
- 飘字
- 震屏
- 临时高亮
- 过渡演出
- 对象池中的可视对象状态

这些数据决定玩家“看到了什么、听到了什么、感受到了什么”。

### 2.3 输入翻译

点击、拖拽、按键、AI 决策、网络消息、Replay 数据，都不应该直接修改逻辑状态。

它们应该先被翻译成逻辑层能理解的数据化意图，再交给玩法逻辑处理。

### 2.4 外部副作用

存档、音频、广告、网络、BI、支付、资源加载、系统时间等都属于外部边界。

逻辑层可以产出“发生了什么”的结果，但不应该直接调用这些外部系统。

---

## 3. 推荐分层

当前推荐的整体分层如下：

```text
Input Layer
    捕获点击、拖拽、按键、AI、Replay、Network 输入

Application / UseCase Layer
    输入门禁、流程编排、Undo/Redo、表现队列、存档边界、外部副作用协调

Logic Core
    GameState、规则求解、OperationBatch、DomainEvent、StateDiff

Projection / ReadModel Layer
    把逻辑状态和变化投影成表现/UI 友好的只读数据

Presentation Layer
    View、Animation、Effect、Audio、UI

Infrastructure Layer
    存档、网络、BI、广告、支付、资源加载、平台能力
```

依赖方向：

```text
外层可以依赖内层
内层不依赖外层
Logic Core 不依赖 UnityEngine
```

Unity 项目里尤其要注意：

```text
Logic Core 尽量是纯 C# 逻辑，不直接引用 GameObject、Transform、Animator、AudioSource、Time、UnityEngine.Random。
```

这样才能支持：

- EditMode 单元测试
- Headless Replay
- 服务器校验
- 自动化测试
- 逻辑性能测试
- 多表现层复用

---

## 4. 当前推荐的数据流

推荐主链路：

```text
RawInput
-> IntentCommand
-> Application UseCase
-> RootTrigger
-> RuleEngine / ECA
-> RuleAction
-> AtomicOperation / DomainEvent / StateDiff
-> OperationBatch
-> GameState
-> CommandResult
-> Projection / Presentation / UndoManager / SaveScheduler / DebugTrace
```

这条链路表达了几个关键结论：

- 输入不是规则结果，只是外部意图。
- ECA 负责根据当前状态求解规则。
- 真正写入状态的是 AtomicOperation。
- 一次外部操作引发的所有变化应归入同一个 OperationBatch。
- View 不直接参与规则判定。
- Replay 重放的是输入和确定性逻辑。
- Undo/Redo 回滚的是已经产生的状态变化，不重新跑规则。

---

## 5. 常见方案与当前取舍

逻辑与表现分离不是某一种固定模式，不同方案适合不同复杂度。

### 5.1 Headless Core + Presenter

适合：

- 关卡制
- 回合制
- 卡牌
- 棋盘
- 解谜
- Tile / Grid 玩法

价值：

- 逻辑可独立测试
- 容易 Replay
- 容易断点调试
- 表现可替换

当前方案的基础方向与它一致：Logic Core 是唯一真相，Presentation 只消费结果。

### 5.2 Command + Event

适合：

- 回放
- AI 输入
- 网络同步
- 自动化测试
- 统一输入入口

当前方案采用这个方向，但会进一步区分：

- IntentCommand：外部意图
- Trigger：逻辑内部规则信号
- DomainEvent：已经发生的业务事实
- StateDiff：底层状态差异
- AtomicOperation：可 Apply / Revert 的状态修改

### 5.3 Entity + System + ViewAdapter

适合：

- Merge
- 经营
- 地图交互
- 对象关系复杂的玩法

当前方案可以兼容这种组织方式。逻辑世界可以有 Entity / System，表现世界可以有 View / Adapter。二者通过结果数据、事件、读模型或表现计划连接。

### 5.4 MVVM / MVP

更适合：

- Meta UI
- 商店
- 背包
- 养成面板
- 任务面板

不建议把复杂玩法核心规则直接建在 MVVM / MVP 上。它们可以用于 UI 表现层，但不应该取代玩法 Logic Core。

### 5.5 ECS

ECS 是一种可选实现手段，不是逻辑与表现分离的必要条件。

适合：

- 实体数量极大
- 同构对象非常多
- 性能瓶颈明确
- 数据导向收益明显

当前阶段的推荐判断：

```text
对象少、规则复杂：优先 RuleEngine + OperationBatch
对象多、规则同构、性能瓶颈明显：再考虑 ECS
```

---

## 6. 核心概念边界

这一节是整套方案最重要的语义边界。

### 6.1 RawInput

RawInput 是原始输入。

例如：

- 玩家点击屏幕
- 玩家拖动物体
- 按钮点击
- AI 做出选择
- 网络包到达
- Replay 读到下一条记录

RawInput 不应该直接进入 Logic Core 修改状态。

### 6.2 IntentCommand

IntentCommand 表示外部希望逻辑尝试执行的意图。

例如：

- 拖动物体到某个格子
- 使用某个道具
- 开始关卡
- 点击某个棋子
- 选择某张卡牌

它的特点：

- 数据化
- 可序列化
- 不包含 Unity 对象引用
- 不直接执行状态修改
- 不承担 Undo/Redo 细节

这里的 Command 不是经典 GoF 命令模式中“自己携带 Execute 行为的命令对象”。在当前玩法架构语境里，它更接近“输入意图的数据化封装”，由 Application / UseCase / Logic Core 中的专门流程处理。

一句话：

```text
IntentCommand = “我想做什么”
```

### 6.3 Trigger

Trigger 是逻辑内部规则入口信号。

它通常由 IntentCommand 翻译而来，也可以由规则内部显式追加，用于驱动后续连锁。

例如：

- EntityDropped
- EntityCreated
- MatchResolved
- TurnStarted
- BuffExpired

Trigger 的特点：

- 用于规则求解
- 通常只在 Logic Core 内部流转
- 不对外承诺业务事实
- 不直接等同于 DomainEvent

一句话：

```text
Trigger = “规则系统现在需要检查什么”
```

### 6.4 ECA

ECA 指 Event / Condition / Action。

在当前方案里，更准确地说：

- Event 更接近 Trigger
- Condition 负责纯判断
- Action 负责生成逻辑结果

当前推荐关系：

```text
IntentCommand
-> RootTrigger
-> ECA 规则求解
-> RuleAction
-> AtomicOperation / DomainEvent / StateDiff
-> OperationBatch
```

关键点：

- ECA Action 不等于 AtomicOperation。
- RuleAction 是高层语义动作。
- AtomicOperation 是最小状态修改。
- RuleAction 可以展开成多个 AtomicOperation。

例如一次“合成”在语义上是一个 MergeAction，但底层可能包含移动、删除、创建、扣步数、更新任务等多个状态修改。

### 6.5 Condition

Condition 是规则判断。

必须满足：

- 只读状态
- 不写状态
- 不发事件
- 不调用外部服务
- 不消耗随机数
- 不读取系统时间
- 不依赖不稳定遍历顺序

一句话：

```text
Condition 必须是纯判断。
```

### 6.6 RuleAction

RuleAction 是规则命中后的语义动作。

它负责描述“当前规则下应该产生哪些逻辑结果”，但不应该绕过统一入口直接改状态。

必须满足：

- 不直接修改 GameState
- 不直接调用 View
- 不直接播放音效
- 不直接存档
- 不直接上报 BI
- 不直接发网络请求
- 不直接读取 Unity 表现对象

RuleAction 应该通过统一上下文产出：

- AtomicOperation
- DomainEvent
- StateDiff
- 必要时追加后续 Trigger

### 6.7 AtomicOperation

AtomicOperation 表示最小可回滚状态修改。

例如：

- 移动实体
- 创建实体
- 删除实体
- 修改数值
- 修改任务进度
- 修改资源数量
- 推进随机状态
- 分配逻辑 ID
- 设置标记位

AtomicOperation 的核心要求：

- 能正向 Apply
- 能反向 Revert
- 保存回滚所需数据
- 只处理状态变化，不表达完整业务语义

一句话：

```text
AtomicOperation = “状态具体怎么变”
```

### 6.8 OperationBatch

OperationBatch 表示一次外部意图引发的所有逻辑变化。

它是多个系统的共同边界：

- Undo/Redo 边界
- 存档脏标记边界
- 调试追踪边界
- 表现计划构建边界
- Replay 校验边界
- 网络同步候选边界

一句话：

```text
OperationBatch = “一次玩家操作或外部意图造成的完整事务”
```

### 6.9 DomainEvent

DomainEvent 表示业务事实已经发生。

例如：

- 实体已经移动
- 实体已经合成
- 任务进度已经变化
- 关卡已经成功
- 奖励已经获得

DomainEvent 的特点：

- 用过去式或已发生语义
- 表达业务意义
- 不表示请求
- 不携带 GameObject / Transform
- 不等同于底层字段变化

一句话：

```text
DomainEvent = “业务上发生了什么”
```

### 6.10 StateDiff

StateDiff 表示状态差异。

它比 DomainEvent 更底层，适合：

- UI 局部刷新
- 调试
- 增量同步
- 精确表现映射
- 状态校验

区别：

```text
DomainEvent：发生了一次合成
StateDiff：A 被删除，B 被删除，C 被创建，步数减少 1，任务进度增加
```

DomainEvent 与 StateDiff 应同时存在，各自服务不同目的。

### 6.11 EffectRequest / VisualCommand

EffectRequest 或 VisualCommand 表示表现请求。

例如：

- 播放动画
- 播放音效
- 生成特效
- 镜头震动
- 显示飘字
- 高亮 UI

更推荐的方向：

```text
Logic Core 产出 DomainEvent / StateDiff
PresentationBuilder 根据它们生成 EffectRequest / VisualPlan
```

这样 Logic Core 更纯。

如果项目强依赖玩法表现绑定，也可以让逻辑层产出抽象表现请求，但这些请求仍然不能依赖 UnityEngine。

---

## 7. Command、Trigger、DomainEvent 的关系

这三者最容易混。

当前最终边界如下：

```text
IntentCommand：外部输入意图
Trigger：逻辑内部规则信号
DomainEvent：对外发布的业务事实
```

不要把三者混用。

推荐流向：

```text
IntentCommand
-> RootTrigger
-> RuleEngine / ECA
-> RuleAction
-> AtomicOperation / DomainEvent / StateDiff
-> OperationBatch / CommandResult
```

后续连锁规则优先通过 RuleAction 显式追加 Trigger，而不是让外部系统监听 DomainEvent 后反向驱动逻辑。

原因是：

- 可以避免规则循环
- 可以避免表现层反向影响逻辑
- 可以让规则链路更容易调试
- 可以保证 ECA 求解过程仍在 Logic Core 内部闭环

当前约束：

```text
逻辑内部连锁：使用 TriggerQueue
对外事实通知：使用 DomainEvent
表现/UI/音效层：不得监听 DomainEvent 后直接修改逻辑
```

---

## 8. AtomicOperation 与 DomainEvent 的关系

AtomicOperation 和 DomainEvent 可以关联，但不应该一一对应。

原因：

- AtomicOperation 偏底层，表达状态怎么变。
- DomainEvent 偏业务，表达发生了什么。
- 一个业务事实可能对应多个 AtomicOperation。
- 一个 AtomicOperation 也未必值得对外发布为 DomainEvent。

推荐关系：

```text
RuleAction / DomainService
    产出 AtomicOperation
    产出 DomainEvent
    产出 StateDiff

OperationBatch
    收集本次操作的 Operations
    收集本次操作的 DomainEvents
    收集本次操作的 StateDiffs
```

这样表现层既可以理解语义，也可以拿到底层变化。

---

## 9. Replay 与 Undo/Redo 的区别

Replay 和 Undo/Redo 不是同一个问题。

### 9.1 Replay

Replay 的目标是：

```text
在同样初始状态、同样配置、同样随机状态、同样输入序列下，
重新演变出同样的逻辑结果。
```

Replay 依赖：

- InitialSnapshot
- LogicVersion
- ConfigVersion
- RandomSeed / RandomState
- CommandSequence
- 固定执行顺序
- 可校验的 StateHash

常见记录方式：

- Command Replay：记录初始状态和完整 Command 序列。
- Snapshot + Command：定期记录快照，再从最近快照开始重放后续 Command。

典型方式：

```text
初始状态 + 随机状态 + Command 序列 + 确定性逻辑 = 重放结果
```

Replay 适合：

- 闯关
- 推关
- 棋盘
- 回合制
- 解谜
- 卡牌
- 自动化测试
- AI 训练或验证

### 9.2 Undo/Redo

Undo/Redo 的目标是：

```text
回到某次操作执行前或执行后的准确状态。
```

Undo/Redo 不应该重新跑 ECA，不应该重新消耗随机数，也不应该重新请求外部副作用。

它依赖的是：

- OperationBatch
- AtomicOperation 的 Apply / Revert
- 必要的状态快照
- UndoStack / RedoStack

当前结论：

```text
Replay 重放输入。
Undo/Redo 回滚或重放已经求解出来的状态变化。
```

---

## 10. 为什么不要把高层玩法行为做成巨大 Do/Undo

一次玩家操作可能引发很多变化：

- 移动
- 合成
- 删除旧实体
- 创建新实体
- 扣步数
- 更新任务
- 消耗随机结果
- 触发连锁规则
- 产出表现语义

如果把整个高层行为做成一个巨大 Do/Undo，会导致：

- Undo 需要重新复刻所有分支判断
- 高层逻辑和回滚逻辑强耦合
- 新增规则会影响旧的 Undo
- 随机和连锁难以复原
- 调试困难

当前推荐：

```text
高层意图负责触发规则求解。
真正可撤销的是求解后产生的 AtomicOperation 集合。
```

也就是：

```text
IntentCommand 不负责 Undo。
RuleAction 不负责巨大 Undo。
AtomicOperation 负责最小回滚。
OperationBatch 负责一次操作的事务边界。
```

---

## 11. Undo/Redo 的表现接入

Undo/Redo 不只是逻辑回滚，玩家还需要看到合理的反向表现。

但必须区分：

```text
逻辑状态可以立即完成回滚。
表现播放通常是异步过程。
```

因此需要 Application Layer 管理表现事务。

推荐概念：

- InputGate：输入门禁
- PresentationQueue：表现队列
- VisualPlan：表现计划
- Reverse VisualPlan：反向表现计划

Undo 的逻辑原则：

- Application 收到 Undo 请求
- 检查当前是否允许 Undo
- 锁定或限制输入
- Logic 逆序 Revert 上一个 OperationBatch
- Projection 更新 ReadModel
- PresentationBuilder 构建反向表现
- PresentationQueue 播放反向表现
- 播放完成后恢复输入

Redo 的逻辑原则：

- Application 收到 Redo 请求
- 正序 Apply 对应 OperationBatch
- Projection 更新 ReadModel
- PresentationQueue 播放正向表现

关键约束：

```text
Undo/Redo 不重新跑 ECA。
Undo/Redo 不重新消耗随机数。
Undo/Redo 不重新触发真实外部副作用。
View 不决定逻辑是否完成。
```

---

## 12. Application / UseCase 层的职责

Application / UseCase 是非常关键的一层。

它不是 Logic Core，也不是 View。

它负责把输入、逻辑、表现、存档、外部边界编排起来。

职责包括：

- 接收 Input / AI / Network / Replay 的请求
- 判断当前是否允许输入
- 创建命令执行上下文
- 调用 Logic Core
- 接收 CommandResult
- 管理 UndoStack / RedoStack
- 管理表现播放时序
- 管理输入锁
- 决定是否标记存档
- 决定是否触发外部副作用
- 协调 DebugTrace

这一层的存在可以避免两个常见问题：

- View 直接驱动 Logic
- Logic 直接调用 View、Save、Audio、Network

---

## 13. CommandResult

Command 执行后需要有结构化结果。

CommandResult 应该表达：

- 是否成功
- 失败原因
- 本次 OperationBatch
- 本次 DomainEvents
- 本次 StateDiffs
- 需要表现层处理的信息
- 是否需要标记存档
- 是否允许进入 Undo 栈

它让 Application Layer 可以统一处理：

- 命令成功
- 命令失败
- 命令无效
- 命令成功但没有状态变化
- 命令触发表现
- 命令需要存档
- 命令需要调试记录

---

## 14. Projection / ReadModel

复杂玩法中，View 不应该直接读取 GameState 内部结构。

GameState 通常是为规则服务的数据结构，例如：

- EntityId 到实体状态
- Cell 到 EntityId
- TaskId 到进度
- 资源 ID 到数量
- Buff 列表
- 随机状态

而 View / UI 通常需要的是：

- 当前格子显示什么
- 当前任务面板显示什么
- 哪些对象可点击
- 哪些对象需要高亮
- 哪些数值需要滚动表现

如果 View 直接依赖 GameState，容易导致：

- View 绑定逻辑内部结构
- GameState 被显示需求污染
- UI 刷新逻辑散落
- 表现和逻辑虽然目录分开，但数据依赖仍然耦合

推荐增加：

```text
GameState + DomainEvent / StateDiff
-> Projection
-> ReadModel / ViewModel
-> View
```

当前判断：

- 简单玩法可以先不做完整 Projection。
- 复杂 UI / 复杂玩法建议引入 ReadModel。
- 这与 CQRS 中“写模型和读模型分离”的思想相似，但不要把所有玩法都强行事件溯源化。
- 不要为了模式而模式。

---

## 15. 表现层是否需要 GraphicSystem

Graphic 层没有 System，不代表设计不完整。

表现层常见有三种粒度。

### 15.1 View 自治

适合简单表现。

特点：

- 一个逻辑对象对应一个 View
- View 自己刷新基础显示
- View 处理自身动画

### 15.2 View + Service

适合中等复杂度。

Service 可以负责：

- 资源加载
- 对象池
- 音效
- 特效
- 相机
- UI 公共能力

### 15.3 View + GraphicSystem

当表现层需要跨对象协调，就应该引入 GraphicSystem。

适合：

- 批量创建/回收 View
- 全局演出编排
- 多对象联动动画
- 相机、震屏、转场
- 排序层和遮挡
- HUD 布局
- 表现节奏控制
- 对象池和资源预热

当前结论：

```text
逻辑层 System 管真状态推进。
表现层 System 管表现世界协调。
二者名称可以相同，但权力完全不同。
```

---

## 16. 确定性约束

只要要做 Replay，就必须重视确定性。

### 16.1 随机数

要求：

- 随机数统一由逻辑层管理
- 随机种子或随机状态进入 GameState / Replay 记录
- 不允许在逻辑层直接使用 UnityEngine.Random
- 不允许随处创建不可追踪的随机源
- 重要随机消耗最好可记录、可追踪

### 16.2 执行顺序

要求：

- TriggerQueue 顺序固定
- Rule 优先级固定
- Entity 遍历顺序固定
- 不依赖 Dictionary 的非固定遍历顺序
- 同一输入在同一状态下必须得到同一结果

### 16.3 时间

要求：

- 回合制、棋盘、解谜类玩法优先使用逻辑 Tick / Step
- Logic Core 不直接读取 Time.time
- Logic Core 不直接读取 DateTime.Now
- 时间变化应通过输入或上下文显式传入

### 16.4 浮点数

要求：

- 棋盘、格子、资源、CD 尽量使用整数或定点数
- 需要严格 Replay 的逻辑避免依赖浮点误差
- 表现层可以使用浮点，但不作为规则真相

### 16.5 Unity 物理

要求：

- 不要默认把 Unity 物理结果作为可严格 Replay 的规则真相
- 如果玩法强依赖物理，必须单独设计确定性边界
- 对棋盘、Merge、解谜、关卡制玩法，优先使用逻辑坐标和逻辑碰撞

---

## 17. 配置数据与规则代码的边界

游戏玩法规则往往同时来自代码和配置。

例如：

- 棋盘生成配置
- 合成表
- 掉落表
- 任务表
- 道具效果表
- 关卡目标
- 权重表
- 新手引导条件

当前推荐折中：

```text
稳定底层规则：代码实现
变化频繁参数：配置驱动
少量组合规则：可配置规则表
复杂行为：代码 Action + 配置参数
```

不建议一开始就做过度通用的规则编辑器。

原因：

- 配置 DSL 容易过度复杂
- 调试困难
- 类型不安全
- 性能不可控
- 表达能力最终仍然不够

当前方向：

```text
代码负责规则能力。
配置负责规则参数。
工具负责可视化和校验。
```

---

## 18. 异常、事务与提交

OperationBatch 是事务边界，因此要考虑异常情况。

需要明确：

- 执行到一半失败怎么办
- 某个 Operation 无法 Apply 怎么办
- RuleAction 生成非法变化怎么办
- Batch 为空是否进入 Undo 栈
- Undo 栈什么时候入栈
- DomainEvent 什么时候发布
- 存档什么时候标记

当前推荐原则：

```text
优先先构建计划，再提交变化。
```

也就是：

```text
Validate
-> Build OperationBatch / ExecutionPlan
-> PreCheck
-> Commit
-> Publish Result
```

如果项目为了简单选择边求解边 Apply，也必须具备失败回滚能力。

当前约束：

- 未成功提交的 Batch 不进入 Undo 栈。
- 空 Batch 默认不进入 Undo 栈，除非它代表有意义的玩家操作。
- 外部副作用必须在逻辑提交成功后，由 Application Layer 决定是否执行。
- 存档应在 Batch 成功提交后标记，而不是每个 Operation 都存。

---

## 19. Snapshot / Memento

Undo/Redo 可以主要依赖 AtomicOperation，但不要求所有状态都手写精细反向逻辑。

对于复杂状态，可以混合使用快照。

当前推荐：

- 简单字段：记录 old / new
- 删除对象：记录完整对象快照
- 创建对象：记录创建所需快照
- 复杂对象：记录 before / after 快照
- 大状态：Checkpoint Snapshot + 后续 Operation

这不是和 AtomicOperation 冲突，而是 AtomicOperation 的一种实现策略。

核心目标仍然是：

```text
每个 Operation 必须知道如何恢复自己造成的状态变化。
```

---

## 20. 存档边界

存档不应该发生在每个 AtomicOperation 内部。

推荐方式：

```text
Batch committed
-> StateVersion++
-> MarkDirty
-> AutoSave at safe point
```

适合客户端游戏的存档内容：

- GameState Snapshot
- SchemaVersion
- ConfigVersion
- LastStateVersion
- 可选的最近 CommandLog / BatchLog

存档时机：

- 一次玩家操作完成后标记 Dirty
- 表现安全点后自动保存
- 关卡完成时强制保存
- 退出或切后台时强制保存

当前约束：

```text
Logic Core 不直接 Save。
Save Boundary 在 Application / Infrastructure。
```

---

## 21. 版本与 ID

如果要支持存档、Replay、热更新、调试和网络同步，必须管理版本和 ID。

### 21.1 SchemaVersion

用于表示存档结构版本。

当 GameState 结构变化时，需要通过 SchemaVersion 做迁移或兼容。

### 21.2 StateVersion

每次成功提交 OperationBatch 后递增。

用途：

- 调试
- Replay 校验
- 网络同步
- 断言状态是否被意外修改

### 21.3 ConfigVersion

用于标识规则配置版本。

Replay 时必须知道使用的是哪一版配置。

### 21.4 LogicVersion

用于标识逻辑代码版本。

当规则代码改变时，旧 Replay 可能不再可严格复现，需要有版本标记。

### 21.5 LogicIdAllocator

逻辑 ID 不能随意生成。

要求：

- ID 分配器状态进入 GameState 或初始快照
- CreateEntity 不能依赖 Unity InstanceID
- Replay 和 Undo/Redo 中 ID 必须稳定

---

## 22. 外部副作用边界

外部副作用不进入可撤销链路。

不适合直接放进 Undo/Redo 的内容：

- BI 上报
- 广告播放
- 支付
- 网络请求
- 真实存档提交
- 系统时间变化
- 资源异步加载

当前推荐：

```text
Logic Core 产出业务事实。
Application Layer 判断是否触发外部副作用。
Infrastructure 执行具体副作用。
```

这样可以保证：

- Replay 不会重复支付
- Undo 不会撤销真实 BI
- 单元测试不依赖外部系统
- 逻辑层可 Headless 运行

---

## 23. 调试工具链

ECA + OperationBatch 如果没有调试工具，后期容易变成黑盒。

建议从早期就设计调试数据。

### 23.1 Command Log

记录：

- 输入命令
- 命令来源
- 执行时状态版本
- 执行结果

来源包括：

- 玩家
- AI
- Replay
- Network
- 自动化测试

### 23.2 Rule Trace

记录：

- 哪些 Rule 被匹配
- 哪些 Condition 通过
- 哪些 Condition 失败
- 哪些 Action 被执行
- 后续 Trigger 如何追加

### 23.3 Operation Log

记录：

- 每个 Batch 包含哪些 Operation
- Operation 的执行顺序
- 修改前后值
- 是否成功提交

### 23.4 DomainEvent Log

记录：

- 发布了哪些业务事实
- 哪些系统消费了它们
- 是否生成表现计划

### 23.5 State Snapshot Diff

记录：

- 执行前状态摘要
- 执行后状态摘要
- StateHash
- StateVersion

### 23.6 Replay Export / Import

目标：

- 能导出一次操作序列
- 能在编辑器里重放
- 能比较最终 StateHash
- 能定位 Replay 第一次不一致的位置

---

## 24. 网络与多人扩展方向

当前方案不要求立即实现网络，但需要避免未来无法扩展。

需要提前避免：

- Logic Core 依赖 UnityEngine
- 随机不可控
- ID 不稳定
- 输入和表现混杂
- 物理结果直接作为唯一真相
- View 直接修改状态

常见方向：

### 24.1 Command 同步

客户端发送 IntentCommand，由权威端执行逻辑。

适合：

- 回合制
- 棋盘
- 卡牌
- 弱实时玩法

### 24.2 StateDiff 同步

权威端计算结果，客户端应用 StateDiff 并播放表现。

适合：

- 客户端不完全可信
- 规则复杂
- 需要防作弊

### 24.3 Snapshot 同步

定期同步完整或局部状态。

适合：

- 状态较小
- 断线重连
- 校验修正

当前方案最自然的扩展是：

```text
客户端输入 IntentCommand
权威端执行 Logic Core
返回 OperationBatch / StateDiff / DomainEvent
客户端 Presentation 播放表现
```

---

## 25. 适用玩法类型

这套方案非常适合：

- Merge
- 棋盘
- 消除
- 卡牌
- 回合制
- 解谜
- 关卡制
- 推关
- 经营类局内逻辑
- 复杂连锁但状态离散的玩法

这些玩法通常具备：

- 状态可离散表达
- 规则可以确定执行
- 输入可以序列化
- Replay 和 Undo/Redo 有明确价值
- 表现可以滞后于逻辑结果播放

需要谨慎裁剪的类型：

- 高频实时动作
- 强物理驱动玩法
- 大规模单位战斗
- 帧同步强实时竞技
- 强依赖连续浮点模拟的玩法

这些项目仍然应该遵守逻辑与表现分离原则，但内部实现可能更偏向：

- Simulation Tick
- 定点数
- ECS
- 服务器权威
- 帧同步
- 专门的物理确定性方案

---

## 26. 当前阶段不展开的实现内容

为了避免过早进入实现细节，当前文档暂不讨论：

- 具体类图
- 接口定义
- 代码骨架
- 文件目录
- 完整时序图
- 具体 Unity 组件绑定方式
- 具体规则编辑器设计
- 具体序列化格式

这些内容应该等当前架构边界、职责拆分、数据流和工程约束继续讨论清楚后，再进入下一阶段。

当前阶段更重要的是把下面这些问题想清楚：

- 真状态边界在哪里
- 表现状态边界在哪里
- 哪些输入应该进入逻辑
- 哪些结果应该对外发布
- 哪些变化必须可撤销
- 哪些副作用不可撤销
- Replay 的确定性要求有多强
- 表现异步期间是否允许继续输入
- ReadModel 是否必要
- 配置化要做到什么程度

---

## 27. 当前核心共识

### 27.1 Logic Core 是唯一真相

胜负、资源、位置、步数、任务、随机、生成删除都属于逻辑层。

### 27.2 View 只表现结果

View 可以发起输入请求，但不能直接决定规则，也不能直接修改真状态。

### 27.3 输入必须数据化

RawInput 需要翻译成 IntentCommand。

IntentCommand 不包含 GameObject、Transform、Animator 等表现对象引用。

### 27.4 Trigger 与 DomainEvent 不能混用

Trigger 是逻辑内部规则信号。

DomainEvent 是对外发布的业务事实。

### 27.5 Condition 必须纯读

Condition 不写状态、不发副作用、不消耗随机、不读取系统时间。

### 27.6 RuleAction 不直接改状态

所有状态变化都必须通过 AtomicOperation 或统一 OperationSink。

### 27.7 AtomicOperation 必须可回滚

每个 AtomicOperation 必须保存 Revert 所需数据。

### 27.8 OperationBatch 是事务边界

Undo/Redo、存档、表现计划、调试、Replay 校验都围绕 Batch。

### 27.9 Replay 依赖确定性

初始状态、配置、随机、输入序列、执行顺序都必须稳定。

### 27.10 Undo/Redo 不重新跑规则

Undo/Redo 只回滚或重放已经产生的 OperationBatch。

### 27.11 外部副作用不进入可撤销链路

BI、支付、网络、真实存档、广告等由 Application / Infrastructure 处理。

### 27.12 表现是异步事务

逻辑可以立即完成，表现由 PresentationQueue 播放，输入策略由 Application 控制。

### 27.13 复杂 UI 建议引入 Projection / ReadModel

避免 View 直接依赖 GameState 内部结构。

---

## 28. 当前推荐一句话总结

```text
IntentCommand 表达输入意图，
Application 编排执行流程，
ECA 负责规则求解，
AtomicOperation 负责可逆状态修改，
OperationBatch 负责一次操作的事务边界，
DomainEvent 表达业务事实，
StateDiff 表达状态差异，
Projection 生成显示数据，
PresentationQueue 播放异步表现，
Infrastructure 处理不可撤销副作用。
```

再压缩成主链路：

```text
Input
-> IntentCommand
-> Application
-> Trigger
-> ECA
-> RuleAction
-> AtomicOperation / DomainEvent / StateDiff
-> OperationBatch
-> GameState
-> Projection / Presentation / Undo / Replay / Save / Debug
```

这是当前阶段最适合作为后续继续讨论的总骨架。

---

## 29. 设计自检清单

每当讨论一个新玩法或新系统时，可以用下面的问题做自检：

- 输入是否统一翻译成 IntentCommand
- IntentCommand 是否不包含表现对象引用
- 规则是否通过 Trigger / Condition / RuleAction 求解
- Condition 是否纯读、无随机、无时间、无副作用
- RuleAction 是否不绕过统一入口直接改状态
- 状态写入是否统一形成 AtomicOperation
- 一次外部意图是否归档成 OperationBatch
- Undo/Redo 是否只处理 OperationBatch
- Undo/Redo 是否没有重跑规则求解
- Replay 所需的初始状态、配置、随机和输入序列是否可记录
- 表现层是否只消费逻辑结果
- 随机、任务、步数、生成删除是否都在 Logic Core 闭环
- 外部副作用是否通过 Application / Infrastructure 边界隔离

如果这些问题大多数答案都是“是”，说明这套玩法的逻辑与表现分离边界通常比较扎实。

---

## 30. 后续讨论优先级

后续不建议一次性讨论所有细节，可以按下面顺序推进。

### 30.1 最小可用分离

先确认：

- GameState 与 View 的边界
- Input 到 IntentCommand 的翻译边界
- Application / UseCase 的编排职责
- View 只监听结果，不直接改真状态

### 30.2 可撤销状态体系

再确认：

- 哪些变化需要 AtomicOperation
- OperationBatch 的事务边界
- Undo/Redo 栈的语义
- 删除、创建、复杂对象是否需要 Snapshot / Memento

### 30.3 规则求解与连锁

继续确认：

- TriggerQueue 的顺序
- Rule 优先级
- Condition 纯读约束
- RuleAction 与 AtomicOperation 的边界
- 后续连锁如何显式追加 Trigger

### 30.4 表现事务化

然后确认：

- 表现异步播放期间是否锁输入
- 是否允许输入排队
- Undo/Redo 的反向表现如何构建
- PresentationQueue 与 InputGate 的职责边界

### 30.5 Replay 与调试

最后确认：

- CommandLog 记录格式
- InitialSnapshot / RandomState / StateHash
- RuleTrace / OperationLog / DomainEventLog
- Replay 不一致时如何定位第一处差异

### 30.6 配置化与工具化

在规则边界稳定后再讨论：

- 哪些规则写代码
- 哪些参数进配置
- 是否需要规则配置表
- 是否需要 Batch 查看器、StateDiff 查看器、Replay 工具
