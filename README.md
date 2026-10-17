# BulletMovement 2.0 使用手册

[English](README.en.md) · [更新与升级](CHANGELOG.zh-CN.md)

BulletMovement 是一个用 C++ 编写的子弹移动组件，适合射击、弹幕和动作游戏。给普通 Actor 添加 **BulletMovement** 组件，设置速度、射程和飞行时间，运行后就能自动移动，不需要开启刚体模拟。

你可以继续配置加减速、重力、目标追踪、返回和反弹，也可以在飞行途中改变弹道、按命中次数减速，或在时间阶段切换时触发自己的效果。组件提供 Niagara 拖尾和蓝图接口，方便接到现有的子弹、武器和对象池中；伤害、爆炸及武器规则由游戏处理。

2.0 在 UE5 中加入了 Mass，由它批量调度子弹运动。使用方式仍然是添加组件、配置参数，组件会自动处理 Mass 的接入。UE4 版本使用同一套弹道计算，通过组件 Tick 更新。

> 本文对应 2.0 更新，提供 Win64 的 UE 4.26、4.27 及 UE 5.0–5.8 共 11 个引擎分支源码包。Fab 上当前可用版本以商品页状态为准；各版本的构建与运行覆盖范围见[验证说明](CHANGELOG.zh-CN.md#验证与性能说明)。

## 安装

通过 Fab 安装与你的引擎版本对应的插件，或将购买取得的 `BulletMovement` 文件夹放入项目的 `Plugins` 目录。在编辑器的 **Edit → Plugins** 中启用 BulletMovement，按提示启用依赖，然后重启编辑器。UE5 版本需要 Mass，UE4 版本不依赖 Mass；Niagara 用来播放可选的拖尾效果。

使用源码包需要配置与目标引擎匹配的 C++ 编译环境。引擎目录和项目目录中的 BulletMovement 保留一份即可，避免同时加载。本文档仓库只提供使用说明，插件源码请通过购买渠道取得。

## 五分钟：创建一颗会飞、会停止的子弹

1. 新建 Actor 蓝图，例如 `BP_Bullet`。添加 Sphere Collision 或 Box Collision，将它设为根组件。设置 **Mobility = Movable**、**Simulate Physics = false**、**Collision Enabled = Query Only**，并让它与测试墙体互相响应为 Block。将显示用的网格放在碰撞根下面，关闭网格自身的碰撞即可。
2. 添加 **BulletMovement** 组件，保持 **Auto Activate** 开启。无需连接 BeginPlay 来启动移动。
3. 设置 `InitialSpeed = 3000`、`MaxSpeed = 0`、`MinSpeed = 0`、`MaxMoveDistance = 6000`，将 `DefaultBulletTimeLimits` 设为单元素数组 `[3.0]`。保持 `GravityScale = 0`、`RefractType = None`。速度单位为 cm/s，距离单位为 cm，时间单位为秒。
4. 在场景中放置或通过 Spawn Actor 生成子弹，让 Actor 的 +X 轴指向发射方向。在它前方放置一堵具有阻挡碰撞的墙，然后运行。
5. 绑定组件的 `OnBulletStop`，在这个示例中调用 Destroy Actor 来销毁子弹。子弹碰到墙会以 `Hited` 原因停止；没有碰墙时，会在到达射程或时间上限后停止。组件只负责停下移动，Actor 的销毁或回收由你决定。

默认初速度为 0，默认飞行时间只有一个 1 秒的阶段。添加组件后，先设置非零初速度，再按需要调整飞行时间。新内容统一填写 `DefaultBulletTimeLimits`；如果旧字段 `BulletTimeLimits` 也填了内容，BeginPlay 会用它覆盖 `DefaultBulletTimeLimits`。

如果每发子弹需要不同的发射者、忽略对象或目标，关闭 Auto Activate，先填好本发参数，再调用 `Activate`（Reset = true）或 `ActiveBulletMovement`。也可以使用延迟生成，在完成 Actor 生成前设置参数。

批量发射时，`GetShotAngle` 可以按数量、锥角和随机设置生成散射变换，`SpawnBulletActor` 可以直接生成指定 Actor 类。需要复用现有子弹时，使用后文的[对象池流程](#尾迹停止与对象池)；这两个辅助函数本身不负责对象池、伤害或联网生成。

## 碰撞与忽略规则

要让子弹正确碰墙，先把实际需要检测的碰撞体设为根。组件移动时会对 `UpdatedComponent` 执行扫掠，普通用法中它就是这个碰撞根；跟随移动的子组件不会各自执行一次完整扫掠。

对于保留默认根的蓝图，插件也会自动整理一种常见结构：当根还是 `DefaultSceneRoot`，其下**恰好只有一个启用 Query 碰撞的 PrimitiveComponent**时，组件会将它提升为根，保持世界变换不变。Query Only 碰撞体也适用。如果有多个候选碰撞体、已经换过根，或显式指定了其他 `UpdatedComponent`，请自行设置需要移动的碰撞根，插件会保留你的选择。

子弹自身会自动加入 `IgnoreActors`。需要穿过发射者时，把发射者也加入这个数组；组件不会自动把发射者、Owner 和 Instigator 全部加入。运行中修改数组，会同步影响移动扫掠和自动定向反弹的目标查询。

`IgnoreActors` 按整个 Actor 忽略，不能用来单独过滤其中某个组件。它只控制本组件的查询和移动；另一物体主动扫掠子弹时，仍使用对方的碰撞设置。

调用 `SwitchCollision(false)` 可以关闭去程移动的扫掠，返回过程则由 `bSweepReturn` 单独控制。这些开关不修改组件本身的 Collision Enabled；Overlap 和阻挡仍取决于 UE 碰撞通道及双方响应。把伤害、爆炸等游戏逻辑接到需要的碰撞事件上，插件不会自动调用 Apply Damage。

## 速度、射程与时间阶段

| 参数 | 用法 |
| --- | --- |
| `InitialSpeed` / `CurrentSpeed` | 激活初速度 / 当前标量速度，cm/s。`UpdateInitialSpeed` 同时更新两者及当前速度向量。 |
| `MaxSpeed` | 加速度更新时使用的速度上限，0 表示不限制。外力产生的速度、直接写入的速度不受这个上限统一限制。 |
| `MinSpeed` | 当前标量速度小于或等于此值时，以 `Speed` 原因停止。 |
| `MaxMoveDistance` | 正值限制本次去程距离，cm；0 表示不设距离上限。`bCheckDistance = false` 可关闭距离检查。 |
| `DefaultBulletTimeLimits` | 每个时间阶段各自的持续秒数，不是累计时间点。激活时复制为运行时阶段数组。 |
| `GravityScale` | 世界重力的倍率，0 关闭重力；不需要启用刚体模拟。 |

例如 `[0.5, 1.0, 2.0]` 表示三个阶段分别持续 0.5、1、2 秒。每次切换到下一个有效阶段时，会触发 `OnBulletStepUp(Step)`，你可以在事件中改变弹道或触发自己的效果。阶段从 0 开始，第一次切换输出 1，全部阶段用完后以 `Time` 原因停止。

时间检查随运动更新执行，会受到更新间隔影响，需要精确计时的游戏逻辑应使用自己的计时方式。空数组会在更新检查时停止，不表示无限飞行。

`MovedDistance` 是已实际移动的距离，`MoveableDistance` 是本次剩余距离。射程限制用于去程；返回时使用返回完成条件。

## 加速度、命中减速与偏转

要让子弹逐渐加速或减速，在 `AcceleInfo` 中添加配置。默认 `bScaleByDelta = true`，常量 `DefaultAccle` 按 cm/s² 填写。例如初速 1000、加速度 500，表示每秒增加约 500 cm/s，直到正值 `MaxSpeed` 上限；填写负值则会减速。可以添加多个配置，它们会共同影响速度。

需要随时间变化时，可以使用曲线：

- CurveFloat 资产填入 `AccleCurve`，X 是该项曲线经过的秒数，Y 乘以 `DefaultAccle`。记得为 `DefaultAccle` 设置需要的倍率，默认 0 会让这项没有加速效果。
- 内嵌曲线选择 `AccleType = Curve`，编辑 `RuntimeAccleCurve`，Y 直接表示加速度。两种曲线都设置时，优先使用 `AccleCurve` 资产。

`StartAcceleration` 和 `StopAcceleration` 用来继续或暂停加速，暂停会保留配置与曲线时间。`bScaleByDelta = false` 时，数值改为每次更新增加的速度量，结果会随更新频率变化，按秒计算的弹道保持默认设置即可。

命中后减速使用 `HitDeceleration`。`Decele` 是每次有效命中扣除的速度量，单位 cm/s；`MaxTime` 是这一项可以触发的命中次数，**不是秒数**。它适合与反弹配合，让子弹在命中后继续飞行；普通无反弹的阻挡命中会直接停止。

每次命中使用数组首项，次数用完后由尾项补位。配置三个及以上条目时，后续执行顺序可能与原数组顺序不同；需要严格按自定顺序变化时，可以在阶段变化后由游戏逻辑补充下一项。`OnBulletDecele` 会提供扣除后的速度和减速阶段。

改变飞行中的弹道，可以调用 `AddForce`：名称为 None 时，Vector 是下一次更新使用的一次性世界空间速度增量；填写名称时，Vector 作为世界空间加速度持续作用，同名调用会替换原值。用 `RemoveForce(Name)` 移除一项，或用 `ClearForce()` 清空全部。这些数值直接用于弹道计算，不按刚体质量计算受力。

## 追踪与初始瞄准

让子弹发射时朝向一个位置，将 `bFollowAim` 设为 true，并填写世界坐标 `AimPoint`。组件激活时会从子弹当前位置朝向这个点。

让子弹在飞行中持续追踪，设置正值 `DefaultHoming`，然后调用 `SetupHoming(TargetComponent)`；也可以在激活前填好 `HomingTarget`。目标可以是 Actor 上任意有效的 SceneComponent。

`DefaultHoming` 越大，每次更新时越快转向目标。组件根据这个强度与 DeltaTime 混合方向，因此它表示转向强度，不是严格的角速度上限，也不提供确定性的网络模拟。

需要变化的追踪强度时，设置 `HomingCurve`：X 是开始追踪后经过的秒数，Y 是强度倍率。`SwitchHomingCurve` 可以开关或替换曲线。`HomingEndRange` 按 cm 填写，进入这个距离后停止继续转向。调用 `SwitchHoming(false)` 可以暂停追踪，`HomingLost()` 则主动结束追踪并触发丢失事件。

目标销毁等失效情况也会触发 `OnBulletHomingLost` 并关闭追踪，子弹继续按其他移动设置飞行。需要切换目标时，由游戏选择新目标并再次设置追踪。

## 返回

调用 `SetupReturn(ReturnComponent, SocketName)`，让子弹返回指定组件的插槽位置；无需插槽时，SocketName 使用 None。开始返回时会触发 `OnBulletRetun`，这个名称保留了旧版拼写。

返回时，当前速度会设为 `MaxSpeed`，请为它设置正值，例如 3000。普通加速中 `MaxSpeed = 0` 表示不限速，但开始返回时会把当前速度设为 0。

`ReturnRange` 是完成返回的半径，单位 cm，例如 100 表示一米。默认 `bSweepReturn = false`，返回时可以穿过障碍；设为 true 后，返回也会使用扫掠碰撞。

进入完成半径后，子弹会以 `Return` 原因触发 `OnBulletStop`；返回目标在途中失效也使用同一原因，便于统一回收。返回期间不再检查去程的射程和时间阶段，也没有单独的返回超时，需要限时结束时由游戏中断。单次移动距离远大于完成半径时，可能越过目标；可以降低速度、增大半径或缩短更新间隔。

## 反弹与定向反弹

普通反弹将 `RefractType` 设为 `Normal`，清空定向目标列表，再用 `RefractTime` 设置反弹次数。没有定向目标列表时，0 表示不反弹，正值会逐次消耗，-1 表示次数不耗尽。子弹需要命中阻挡碰撞才会反弹，发生时通过 `OnBulletRefracted` 返回命中结果和反弹后的速度向量。

定向反弹使用 `Target` 模式，命中后会转向指定目标组件。调用 `SetRefractTarget` 填入目标列表，或用 `AddRefractTarget` 追加目标，并保持这些组件引用有效。

需要自动寻找目标时，留空指定目标列表，设置 `RefractTargetType` 的对象通道和 `RefractTargetMaxRange` 搜索半径。`RefractTargetLimitAngle` 可以限制前方的搜索锥角，0 表示不按角度筛选。自动查询的结果不按距离排序；如果要优先攻击最近敌人，应由游戏选好目标再传入列表。

定向目标列表和剩余次数都会影响反弹。需要关闭反弹时，将 `RefractType` 设为 None；列表非空时，只把次数改为 0 可能仍会继续反弹。对象池中要重复使用同一条反弹链，每发前重新填入列表和次数，激活时会清除上一发的当前反弹目标。目标筛选、敌我判断和伤害冷却由游戏处理。

## 尾迹、停止与对象池

在 `TrailVFX` 选择 Niagara System，用 `TrailScale` 调整缩放。运行时需要替换拖尾，可以调用 `SetupTrailVFX`。尾迹会跟随子弹根组件，停止时停用，Actor 销毁时清理。

拖尾如何消退取决于 Niagara 资产设置。希望粒子自然结束时，可以先停止子弹、延迟销毁，或由游戏生成独立的结束效果；不要依赖 Actor 销毁后尾迹仍然保留。

停止移动有两个入口：`StopBulletMovement()` 直接停止移动和尾迹，不广播停止原因；`BulletStop(Reason)` 先触发 `OnBulletMovementStop`、`OnBulletStop`，再停止。对同一个伤害或回收动作，绑定一个停止事件即可，避免执行两次。

自动停止常见的原因是 `Time`、`Distance`、`Speed`、`Return` 和 `Hited`。枚举中的 `Dameged`、`Direct` 可由游戏主动传入，受伤逻辑仍由游戏实现。`Hited`、`Dameged` 和 `OnBulletRetun` 都保留了旧拼写，查找或调用 API 时使用原名。

对象池建议按以下顺序复用：

1. 创建池对象时关闭 Auto Activate。回收时调用 `Deactivate()` 或 `StopBulletMovement()`，再处理隐藏和碰撞。
2. 再次发射前设置世界变换、初速度、射程、`DefaultBulletTimeLimits`、目标与本发的 `IgnoreActors`，保持子弹自身也在忽略列表中。
3. 重新填入本发需要的 `HitDeceleration`、`RefractTime` 和反弹目标列表，清除不再需要的持久力。按需要重设加速度项的 `AccleTime` 及追踪开关。
4. 恢复显示和碰撞，调用 `Activate`（Reset = true）、`ActiveBulletMovement()` 或 `SwitchBulletMovement(true, true)`，再设置需要在激活后应用的追踪或返回行为。激活会重置速度、剩余距离、时间阶段、返回状态和当前反弹目标，并重新开启去程移动扫掠。

激活只重置本次飞行需要的部分状态，不会把全部参数还原成类默认值。`OnBulletDecele` 的内部减速阶段计数也不会逐发归零；蓝图需要从零统计时，可以维护自己的本发计数。

在 UE5 中，组件会随激活、停止、注销和销毁管理对应的 Mass 实体，你只需按上面的组件流程使用。同一颗子弹应由一套移动逻辑控制位置，避免再创建第二个 Mass 运动实体，或同时使用 ProjectileMovement、刚体模拟来移动它。

UE4.26–UE5.1 的部分事件输出引脚名称与较新版本不同。跨引擎版本迁移时，请按[升级说明](CHANGELOG.zh-CN.md)检查事件连线；事件传出的数据含义相同。

## UE4 与 UE5 的执行差异

各版本使用同一套弹道计算和组件接口。UE4.26/4.27 通过组件 Tick 计算并移动子弹，不包含 Mass。UE5 则由组件自动接入 Mass：先在游戏线程读取 Actor 和组件数据，再由 Mass 计算弹道，最后在游戏线程完成场景移动、碰撞和事件。这些注册和调度由插件处理，不用另建管理器、专用子弹基类或 Mass Trait 资产。

UE5.0–5.3 在 Mass 运动处理器的工作线程任务内顺序处理数据块（chunks），UE5.4 及以后使用并行 chunk 查询。UE5.8 还针对该版 Chaos 接口加入场景发布和查询批处理；其他版本使用普通组件场景移动。

组件默认约每 0.016 秒更新一次。更新间隔、帧率、碰撞体尺寸和速度都会影响轨迹及事件时机。Mass 负责调度弹道计算，实际帧耗时还包括 Actor、碰撞、事件、Niagara 和游戏逻辑；评估性能时，应在目标设备上使用实际弹量，测量完整帧时间。已完成的检查范围见[验证与性能说明](CHANGELOG.zh-CN.md#验证与性能说明)。

## 平台与联网边界

本次发行面向 **Win64**。联网游戏需要自行处理子弹生成和伤害的权威端、状态复制与客户端表现；插件不提供子弹复制、RPC、客户端预测、重演或服务器校正。Mass 处理器可以在客户端和服务器世界运行，但不会因此自动同步子弹。其他平台、专用服务器和打包运行的检查范围，请查看对应发行包的验证说明。

## 常见问题

| 现象 | 先检查什么 |
| --- | --- |
| 添加组件后子弹不动 | `InitialSpeed` 是否大于 0，Auto Activate 是否开启；手动激活时是否已调用 `Activate` 或 `ActiveBulletMovement`。 |
| 子弹很快就停止 | 默认时间阶段只有 1 秒；检查 `DefaultBulletTimeLimits`、旧 `BulletTimeLimits`、射程和 `MinSpeed`。 |
| 子弹穿墙 | 碰撞体是否为根，双方是否响应为 Block，Query 碰撞和移动扫掠是否开启；返回时检查 `bSweepReturn`。 |
| 加速曲线没有效果 | `AccleCurve` 资产使用 `DefaultAccle` 作倍率，默认 0 不产生加速；同时检查加速是否暂停。 |
| 返回时不动或反复越过目标 | 返回会取 `MaxSpeed` 作为当前速度；设置正值，并根据速度调整 `ReturnRange` 和更新间隔。 |
| 对象池第二发与第一发不同 | 检查是否重新填入反弹次数、目标列表、命中减速和本发忽略对象，是否清理了上发的持久力及自定义状态。 |

旧项目升级前请阅读[更新与升级](CHANGELOG.zh-CN.md)，尤其是时间数组、消耗参数、碰撞根及 C++ 头文件的变化。
