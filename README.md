# BulletMovement 2.0 使用手册

[English](README.en.md) · [更新与升级](CHANGELOG.zh-CN.md)

BulletMovement 为普通 Actor 提供参数驱动的弹道移动，包括加速度、曲线、重力、追踪、返回、反弹、命中减速、距离与时间限制，以及 Niagara 尾迹。将 **BulletMovement** 组件加入子弹 Actor 并设置参数，即可自动开始移动；无需创建专用子弹基类、管理器 Actor 或 Mass Trait 资产。伤害、爆炸、武器规则和对象池由你的游戏负责。

> 本文对应 2.0 更新，提供 Win64 的 UE 4.26、4.27 及 UE 5.0–5.8 共 11 个引擎分支源码包。Fab 上当前可用版本以商品页状态为准；各版本的构建与运行覆盖范围见[验证说明](CHANGELOG.zh-CN.md#验证与性能说明)。

## 安装

通过 Fab 将对应引擎版本的插件安装到引擎，或将购买取得的 `BulletMovement` 文件夹放入项目的 `Plugins` 目录。在编辑器的 **Edit → Plugins** 中启用 BulletMovement，按提示启用依赖并重启编辑器。UE5 版本使用 Mass，UE4 版本不依赖 Mass；Niagara 用于可选尾迹。

使用源码包时，需要与目标引擎匹配的 C++ 编译环境。不要在同一项目同时启用引擎目录和项目目录中的两份 BulletMovement。本文档仓库只提供说明，商业插件源码由购买渠道提供。

## 五分钟：创建一颗会飞、会停止的子弹

1. 新建 Actor 蓝图，例如 `BP_Bullet`。添加 Sphere Collision 或 Box Collision，将它设为根组件。设置 **Mobility = Movable**、**Simulate Physics = false**、**Collision Enabled = Query Only**，并让它与测试墙体互相响应为 Block。将可见网格作为碰撞根的子组件；如果只用于显示，可关闭网格碰撞。
2. 添加 **BulletMovement** 组件，保持 **Auto Activate** 开启。无需连接 BeginPlay 来启动移动。
3. 设置 `InitialSpeed = 3000`、`MaxSpeed = 0`、`MinSpeed = 0`、`MaxMoveDistance = 6000`，将 `DefaultBulletTimeLimits` 设为单元素数组 `[3.0]`。保持 `GravityScale = 0`、`RefractType = None`。速度单位为 cm/s，距离单位为 cm，时间单位为秒。
4. 在场景中放置或通过 Spawn Actor 生成子弹，让 Actor 的 +X 轴指向发射方向。在它前方放置一堵具有阻挡碰撞的墙，然后运行。
5. 绑定组件的 `OnBulletStop`。对最小示例，可在事件中调用 Destroy Actor。未碰墙时，子弹将因距离或时间限制停止；碰到阻挡物时，停止原因是 `Hited`。停止移动本身不会销毁 Actor。

默认初速度为 0、默认单阶段时间为 1 秒，因此添加组件后仍需设置非零初速度。若同时填写旧字段 `BulletTimeLimits`，其非空数组会在 BeginPlay 覆盖 `DefaultBulletTimeLimits`；新内容建议统一使用后者。

需要在首次运动前设置发射者、忽略对象、目标或其他逐发参数时，关闭 Auto Activate，在配置完成后调用 `Activate`（Reset = true）或 `ActiveBulletMovement`。也可以在延迟生成 Actor 时先设置参数，再完成生成。

## 碰撞与忽略规则

移动对 `UpdatedComponent` 执行扫掠，普通用法中它是 Actor 的碰撞根。子组件不会因为跟随根移动而各自承担完整的移动扫掠；请把实际需要检测的碰撞体设为根。

插件兼容常见的蓝图初始结构：当根是未经替换的 `DefaultSceneRoot`，并且其下**恰好只有一个启用 Query 碰撞的 PrimitiveComponent**时，组件会自动将该碰撞体提升为根并保持世界变换。Query Only 碰撞体可以使用此路径。如果存在多个候选、使用自定义根，或已经显式指定其他 `UpdatedComponent`，插件会保留你的选择；这时请自行明确碰撞根。

`IgnoreActors` 自动加入子弹自身；**发射者、Owner 或 Instigator 不会自动全部加入**。需要穿过发射者时，将发射者及其他需要忽略的 Actor 加入该数组。数组用于移动扫掠和自动定向反弹目标查询，并会随运行时修改同步。它按 Actor 忽略，不是单个组件过滤，也不代表双向忽略：另一物体主动扫掠子弹时仍由对方的碰撞规则决定。

`SwitchCollision(false)` 关闭本组件运动时的扫掠，不会全局修改所有组件的 Collision Enabled。返回过程另由 `bSweepReturn` 控制。Overlap、阻挡和可命中的对象仍取决于 UE 的碰撞通道及双方响应。伤害和爆炸可接到游戏自己的碰撞事件；插件不会自动 Apply Damage。

## 速度、射程与时间阶段

| 参数 | 当前含义 |
| --- | --- |
| `InitialSpeed` / `CurrentSpeed` | 激活初速度 / 当前标量速度，cm/s。`UpdateInitialSpeed` 同时更新两者及当前速度向量。 |
| `MaxSpeed` | 加速度更新使用的速度上限，0 表示不限制。它不是对任意外力或直接写入速度的全局限速器。 |
| `MinSpeed` | 当前标量速度小于或等于此值时，以 `Speed` 原因停止。 |
| `MaxMoveDistance` | 正值限制本次去程距离，cm；0 表示不设距离上限。`bCheckDistance = false` 可关闭距离检查。 |
| `DefaultBulletTimeLimits` | 每个时间阶段各自的持续秒数，不是累计时间点。激活时复制为运行时阶段数组。 |
| `GravityScale` | 世界重力的倍率，0 关闭重力；不需要启用刚体模拟。 |

例如 `[0.5, 1.0, 2.0]` 表示三个阶段分别持续 0.5、1、2 秒。进入仍有下一阶段的阶段时触发 `OnBulletStepUp(Step)`，阶段索引从 0 开始，首次升段输出 1；全部阶段消耗后以 `Time` 原因停止。检查发生在运动更新处，不作为精确计时器使用；空数组会在更新检查时停止。

`MovedDistance` 是已实际移动的距离，`MoveableDistance` 是本次剩余距离。射程限制用于去程；返回时使用返回完成条件。

## 加速度、命中减速与偏转

在 `AcceleInfo` 中添加配置。默认 `bScaleByDelta = true`，常量 `DefaultAccle` 的单位为 cm/s²，每次按实际时间增加或减少标量速度。例如初速 1000、加速度 500，表示每秒增加约 500 cm/s，直到正值 `MaxSpeed` 上限。多个配置会叠加。

使用 CurveFloat 资产时，设置 `AccleCurve`，X 为该项曲线经过的秒数，Y 乘以 `DefaultAccle`。使用内嵌曲线时，选择 `AccleType = Curve` 并编辑 `RuntimeAccleCurve`，Y 直接作为加速度。存在 `AccleCurve` 资产时优先使用它。`StartAcceleration` / `StopAcceleration` 控制应用开关；暂停不会自动清空配置或重置曲线时间。`bScaleByDelta = false` 改成每次更新的速度增量，结果受更新频率影响，仅在你明确需要这种行为时使用。

`HitDeceleration` 中的 `Decele` 是每次有效命中扣除的速度量，单位 cm/s；`MaxTime` 是该项可触发的命中次数，**不是秒数**。它主要配合可以继续运动的反弹使用；普通无反弹阻挡命中会直接停止。每次消耗当前首项，首项耗尽后以尾项补位，因此不要依赖三个及以上配置按原数组顺序执行。`OnBulletDecele` 提供扣除后的速度和减速阶段。需要严格自定义顺序时，可在游戏逻辑中按阶段补充下一项。

`AddForce(None, Vector)` 为下一次运动更新添加一次世界空间速度增量；命名的 `AddForce` 按世界空间加速度持续作用，同名调用替换该项。使用 `RemoveForce(Name)` 移除一项，或 `ClearForce()` 清空全部。这些是数值弹道偏转，不是刚体 Add Force，不使用质量计算。

## 追踪与初始瞄准

`bFollowAim = true` 时，激活方向由子弹位置指向世界坐标 `AimPoint`。这是初始瞄准；持续追踪请设置 `DefaultHoming` 为正值，并调用 `SetupHoming(TargetComponent)`，或在激活前赋值 `HomingTarget`。目标可以是 Actor 的任意有效 SceneComponent。

`DefaultHoming` 控制每次更新向目标方向混合的强度；数值越大，转向越快。该公开追踪模式的混合量与 DeltaTime 相乘，不应把它当作严格角速度上限或网络确定性模拟。`HomingCurve` 的 X 为追踪经过的秒数，Y 为强度倍率；用 `SwitchHomingCurve` 开关或替换曲线。`HomingEndRange` 为停止继续转向的接近距离，cm。`SwitchHoming(false)` 暂停追踪，`HomingLost()` 主动结束追踪并通知事件。

目标销毁等失效情况触发 `OnBulletHomingLost` 并关闭追踪，子弹继续按其余移动规则运行；不会因此自动生成新目标。

## 返回

调用 `SetupReturn(ReturnComponent, SocketName)` 开始朝指定组件的插槽位置返回；无需插槽时使用 None。调用时会触发名称保留为 `OnBulletRetun` 的事件，它表示**开始返回**。

返回速度取 `MaxSpeed`，因此使用返回功能时请设置正值，例如 3000；`MaxSpeed = 0` 在普通加速中表示不限速，但不能充当返回速度。`ReturnRange` 是完成返回的半径，单位 cm，例如 100 表示一米。默认 `bSweepReturn = false`，返回可以穿过障碍；设为 true 后使用扫掠碰撞。

进入完成半径后，`OnBulletStop` 的原因是 `Return`；返回目标在途中失效也按该原因结束，方便走同一回收路径。返回期间不再用去程射程或时间阶段结束运动。返回没有独立超时，游戏可按需求自行中断。若单次移动距离远大于完成半径，建议降低速度、增大半径或缩短更新间隔。

## 反弹与定向反弹

将 `RefractType` 设为 `Normal`、清空定向目标列表，并设置正值 `RefractTime`，即可按命中法线反射移动方向。无定向目标列表时，次数为 0 不反弹；-1 用作不耗尽的反弹次数。反弹仍需要有效的阻挡碰撞。反弹发生时 `OnBulletRefracted` 返回命中结果和反射后的速度向量。

`Target` 模式会在命中后转向目标组件。用 `SetRefractTarget` 替换指定目标列表，或 `AddRefractTarget` 追加目标；保留有效的目标引用。没有指定目标时，设置 `RefractTargetType` 的对象通道、`RefractTargetMaxRange` 的搜索半径，以及可选 `RefractTargetLimitAngle` 的前方锥角限制。角度为 0 表示不做角度筛选。自动候选来自场景查询，不保证按最近距离排序；请不要依赖它自动选择最近敌人。

定向目标列表与剩余反弹次数都会影响后续行为。列表非空时，仅将次数设为 0 不一定关闭反弹；使用 `RefractType = None` 关闭反弹功能。若要每次发射相同的链式反弹，重新填入列表和次数；目标筛选、敌我判断与伤害冷却由游戏自行实现。

## 尾迹、停止与对象池

在 `TrailVFX` 选择 Niagara System，用 `TrailScale` 设置缩放；也可通过 `SetupTrailVFX` 运行时替换。尾迹随子弹根组件移动，停止时停用，最终销毁 Actor 时清理组件。拖尾粒子的消退由 Niagara 资产控制；不能保证 Actor 销毁后仍保留尾迹。需要尾迹自然结束时，可先停用子弹并延迟销毁，或由游戏创建独立结束效果。

`StopBulletMovement()` 静默停止移动及尾迹，不广播停止原因事件。`BulletStop(Reason)` 先广播 `OnBulletStop` 和另一个停止事件 `OnBulletMovementStop`，再停止；同一业务通常只需绑定其中一个，避免重复扣伤或回收。常见自动原因是 `Time`、`Distance`、`Speed`、`Return`、`Hited`。枚举中的 `Dameged`、`Direct` 可由游戏主动传入，插件不会自动实现受伤逻辑。兼容拼写 `Hited`、`Dameged`、`OnBulletRetun` 保留不变。

对象池建议按以下顺序复用：

1. 创建池对象时关闭 Auto Activate。回收时调用 `Deactivate()` 或 `StopBulletMovement()`，再处理隐藏和碰撞。
2. 再次发射前设置世界变换、初速度、射程、`DefaultBulletTimeLimits`、目标与本发的 `IgnoreActors`，保持子弹自身也在忽略列表中。
3. 根据本发配置恢复会消耗的 `HitDeceleration`、`RefractTime` 和反弹目标列表；清除不需要的持久力，并按需要重设加速度项的 `AccleTime` 及追踪开关。`OnBulletDecele` 的内部减速阶段计数不会在激活时自动归零；蓝图若需要逐发从零计数，可维护自己的逐发计数。激活重置不等于整个组件回到类默认值。
4. 恢复显示及需要的碰撞设置，调用 `Activate`（Reset = true）、`ActiveBulletMovement()` 或 `SwitchBulletMovement(true, true)`，然后应用需要在激活后设置的追踪/返回行为。激活会重置速度、剩余距离、时间阶段、返回状态，并重新开启移动扫掠。

UE5 的实体创建、停用释放、注销与销毁由组件自动管理。不要额外为同一子弹建立第二套 Mass 运动实体，也不要再添加会同时写入位置的 ProjectileMovement 或刚体运动。

## UE4 与 UE5 的执行差异

两条后端共用弹道数值内核和组件使用入口。UE4.26/4.27 在组件 Tick 中运行内核及场景移动，不包含 Mass。UE5 在游戏线程采集 Actor/组件输入，使用 Mass 运动处理器计算纯数值，再回到游戏线程执行场景移动、碰撞和事件；你不需要手动注册实体。

UE5.0–5.3 在 Mass 运动处理器的 worker 任务内顺序遍历 chunks；UE5.4 及以后使用并行 chunk 查询。UE5.8 还包含针对相应 Chaos 接口的场景发布与查询批处理支持，其他版本使用普通组件场景移动。不要将 5.8 的该项优化理解成所有版本都具有相同实现。

插件保留组件更新间隔，默认约 0.016 秒；间隔、帧率、碰撞体尺寸与速度会影响轨迹和事件时机。Mass 负责运动任务调度，Actor、碰撞、事件、Niagara 和你编写的蓝图逻辑仍有成本。当前没有跨版本统一的吞吐、净加速或“零卡顿”保证；应在目标设备及实际弹量下测量完整帧时间。

## 平台与联网边界

本次发行目标平台为 **Win64**。插件没有提供子弹状态复制、RPC、客户端预测、重演或服务器校正；不会因为 Mass 处理器可在客户端/服务器世界运行就自动同步网络。联网游戏需要自行决定生成与伤害的权威端、复制数据和视觉表现。其他平台、专用服务器及打包运行的完成情况，以对应发行包的实际验证范围为准。

辅助蓝图函数 `GetShotAngle` 可生成散射变换，`SpawnBulletActor` 可按数量、锥角与随机选项生成指定 Actor；它们不会代替对象池、网络生成权威或伤害系统。

旧项目升级前请阅读[更新与升级](CHANGELOG.zh-CN.md)，尤其是时间数组、消耗参数、碰撞根及 C++ 头文件的变化。
