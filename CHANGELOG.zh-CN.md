# BulletMovement 更新与升级

[使用手册](README.md) · [English](CHANGELOG.en.md)

## 2.0

2.0 为 UE5 加入了自动 Mass 调度，用来批量处理子弹运动。使用方式延续 1.0：给 Actor 添加 BulletMovement、配置弹道参数，运行后自动开始移动。UE4 使用同一套弹道计算，通过组件 Tick 更新。

本次更新对应 `VersionName = 2.0`，提供 UE 4.26、4.27 及 UE 5.0–5.8 共 11 个引擎分支的 Win64 源码包。具体构建和运行范围见文末的[验证与性能说明](#验证与性能说明)。

- **自动接入 Mass。** UE5 由组件处理实体的创建、释放和清理，无需另建管理器或 Mass Trait 资产。读取 Actor 数据、移动碰撞和触发事件仍在游戏线程完成，弹道数值由 Mass 计算。
- **按引擎版本适配。** UE5.0–5.3 在 Mass 工作线程任务内顺序处理数据块（chunks），UE5.4 及以后使用并行 chunk 查询。UE5.8 另有针对该版 Chaos 接口的场景发布和查询批处理。
- **保留原有弹道功能。** 加减速、曲线、追踪、返回、反弹、命中减速、时间和射程限制，以及 Niagara 尾迹继续使用。激活与停止接口可以接到现有对象池中。
- **保留常用蓝图入口。** 组件类仍为 `UBulletMovementComponent`，保留 `OnBulletRetun` 等旧名称；补充停止原因和 `OnBulletMovementStop` 事件。
- **完善碰撞设置。** `IgnoreActors` 会同步到移动扫掠，同时保留碰撞体原本已有的原生忽略项。保留 `DefaultSceneRoot` 且下面只有一个启用 Query 的碰撞体时，会自动将该碰撞体提升为根。
- **修复对象池的反弹链。** 每次激活清空 `CurrentRefTarget`，避免上一发的当前目标影响重新填入的目标列表，让下一发从首目标开始。
- **修复发布包编译问题。** 补齐蓝图属性和函数的分类，修复 UE5.1 事件参数重名，并为 UE5.6、5.7 显式包含 Mass 实体句柄头文件。

### 从 1.0 升级

替换插件前，保留一份可恢复的项目副本。替换后使用对应引擎重新编译 C++ 子类和子弹蓝图，检查参数与引脚；组件类名虽然保留了，部分 C++ 声明和蓝图引脚仍有变化。升级时主要检查下面几项。

**时间参数。** 2.0 用 `DefaultBulletTimeLimits` 保存每发使用的阶段时间，激活时再复制到运行中的 `BulletTimeLimits`。为兼容旧内容，BeginPlay 发现 `BulletTimeLimits` 非空时，会先用它覆盖默认数组；两处都填过值的蓝图，建议合并到 `DefaultBulletTimeLimits`。阶段时间按条目逐个消耗，空数组会使子弹在更新检查时停止。

**速度与单位。** `MaxSpeed = 0` 表示加速时不限速，`MaxMoveDistance = 0` 表示去程不限距离；`ReturnRange` 按 cm 填写完成半径。加速度默认按秒计算（`bScaleByDelta = true`），旧内容如果按“每次更新增加多少速度”调过参数，需要检查换算。返回会把当前速度设为 `MaxSpeed`，使用返回时请设置正值。如果项目自己的派生类另有单位或参数约定，升级时也要检查这些重写。

**加速度曲线。** 原有 `AccleCurve` 资产入口继续保留，设置资产后就会使用，以 `DefaultAccle` 作倍率。选择曲线以资产和 `AccleType` 设置为准：只打开旧 `bUseAccleCurve` 标记不会产生有效曲线，关闭它也不会停用已经设置的资产。需要暂停加速时调用 `StopAcceleration`；使用内嵌曲线时，选择 `AccleType = Curve`。

**碰撞根与忽略。** 检查真正的碰撞体是否为根，或是否满足手册中的自动提升条件。子组件跟随根移动时，不会各自执行完整扫掠。子弹自身会自动忽略，需要穿过发射者时，把发射者加入 `IgnoreActors`。

**事件与停止。** `StopBulletMovement` 直接停止，不触发停止原因事件；需要事件时调用 `BulletStop`。它会先触发新增的 `OnBulletMovementStop` 和原有的 `OnBulletStop`，再停止移动，同一个伤害或回收动作绑定一个事件即可。`OnBulletRetun` 表示开始返回，完成返回使用停止原因 `Return`。新增枚举值追加在原 `Time/Distance/Speed/Return` 之后，保留原值的顺序。

UE4.26–UE5.1 的部分事件输出引脚使用 `RefractVelocity`、`DeceleSpeed`、`ReturnTargetMesh` 和 `ReturnTargetSocket`；UE5.2 及以后分别使用 `Velocity`、`CurrentSpeed`、`ReturnMesh` 和 `ReturnSocket`。切换引擎分支后，请检查这些事件的连线。

**对象池。** 激活会重置速度、时间、距离及当前反弹目标等飞行状态。再次发射前，仍需重新填入消耗过的反弹次数、目标列表和命中减速，清理持久力，设置本发的游戏参数；具体顺序见[手册](README.md#尾迹停止与对象池)。尾迹会随 Actor 销毁而清理，想保留自然消退效果时，先停止子弹、延迟销毁，或生成独立结束效果。

**C++。** 加速度、减速度和枚举分别放在 `FBulletAcceleInfo.h`、`FBulletDeceleInfo.h`、`E_BulletAccleType.h`、`E_BulletRefractType.h` 和 `E_BulletStopReason.h`，按需要包含相应头文件。部分运行状态和内部标志不再作为可写蓝图参数公开，直接读写内部字段的代码需要调整为对应函数调用。`TrailVFX` 仍接收 Niagara System；引用它时使用资产对象，不依赖底层指针包装类型。

### 验证与性能说明

本次逐版编译使用的引擎版本是 4.26.2、4.27.2、5.0.3、5.1.1、5.2.1、5.3.2、5.4.4、5.5.4、5.6.1、5.7.4 和 5.8.3。各包已按引擎插件的要求，通过对应版本的 UBT、UHT（`EngineRuntime`）和非 Unity 插件 C++ 编译检查。检查目标为 Win64 Game Development，使用 `NoLink`，完成到 C++ 对象编译，没有链接成每版 DLL，也没有走完各版默认 `BuildPlugin` 的 Editor、Development 和 Shipping 构建；这些结果只覆盖上面列出的补丁版本。

UE5.8 另外完成了 PIE 运行检查，覆盖普通 Actor 的激活、停止、注销和销毁，以及游戏子弹的实际集成。其他版本的运行、各版打包和联网模式仍需在对应项目中验证。

本次主要检查兼容性和运行行为，尚未做跨版本性能对比。评估 Mass 的收益时，建议在实际场景中测量整帧耗时，并把碰撞、Actor、拖尾和游戏逻辑一起考虑。

## 1.0

初版提供用 C++ 编写、不依赖刚体模拟的子弹移动组件。给 Actor 添加 BulletMovement、设置参数即可使用，已有速度、射程、时间阶段、加速度曲线、命中减速、偏转、反弹、追踪、返回、尾迹及散射生成辅助函数。
