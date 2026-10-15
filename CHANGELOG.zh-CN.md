# BulletMovement 更新与升级

[使用手册](README.md) · [English](CHANGELOG.en.md)

## 2.0

以下说明对应插件描述文件中的 `VersionName = 2.0`，包含 UE 4.26、4.27、5.0、5.1、5.2、5.3、5.4、5.5、5.6、5.7、5.8 共 11 个引擎分支的 Win64 源码包。各包分别通过对应版本的 UBT/UHT 与插件 C++ 对象编译；这不代表全部版本均已完成链接、运行或打包验证。

- 普通 Actor 上的 BulletMovement 组件自动管理运动生命周期；不要求专用 Actor 基类、管理器或 Mass Trait 资产。
- UE5 使用自动 Mass 调度，游戏线程采集输入、数值运动处理、游戏线程场景移动与事件分离；UE4 使用同一数值内核的组件 Tick 后端。
- UE5.0–5.3 在 Mass worker 内顺序遍历 chunks，UE5.4 及以后使用并行 chunk 查询。UE5.8 另有对应 Chaos 接口的场景发布与查询批处理支持。
- 保留公开组件类名 `UBulletMovementComponent`、原有常用蓝图入口与事件，包括兼容拼写 `OnBulletRetun`。补充显式停止原因及 `OnBulletMovementStop` 事件入口。
- 将 `IgnoreActors` 同步到移动扫掠，保留原本已存在的原生忽略项；支持未替换 `DefaultSceneRoot` 下单个 Query 碰撞体自动提升为碰撞根。
- 保留追踪、返回、反弹、加速度、命中减速、时间/射程与 Niagara 尾迹工作流，并让对象池激活/停止与 UE5 实体生命周期衔接。
- 修复对象池重新发射时沿用上一发当前反弹目标的问题：每次激活都会清空 `CurrentRefTarget`，重新填入的反弹链可从首目标开始。

### 从 1.0 升级

升级前保留项目自己的可恢复副本，替换插件后使用对应引擎重新编译，并打开现有子弹蓝图检查参数与引脚。类名保持并不等于所有 C++ 源码或蓝图引脚完全不变；请重新编译你的子类和蓝图。

**时间参数。** 2.0 增加 `DefaultBulletTimeLimits` 作为每次激活的阶段模板。兼容字段 `BulletTimeLimits` 非空时，会在 BeginPlay 复制到模板；运行时它仍会随阶段消耗。若两处都有旧值，请合并到一处。空阶段数组不是无限寿命。

**速度与单位。** 公共组件的 `MaxSpeed = 0` 表示不限制加速上限，`MaxMoveDistance = 0` 表示不限制去程距离，`ReturnRange` 按厘米半径填写。加速度默认按秒计算（`bScaleByDelta = true`），并非每次更新固定加一次。返回速度仍取 `MaxSpeed`，需要返回时设置正值。不要把其他项目子类的特殊数值语义移植到公共组件。

**加速度曲线。** 旧 `AccleCurve` 资产入口继续保留；资产存在即使用它，并以 `DefaultAccle` 作倍率。仅设置旧 `bUseAccleCurve` 不会选择有效曲线，关闭这个标记也不会忽略已赋值资产。需要禁用加速时调用 `StopAcceleration`；新内嵌曲线通过 `AccleType = Curve` 选择。

**碰撞根与忽略。** 确认真正的碰撞体为根，或满足自动提升规则。默认子弹自身会忽略，发射者需由你加入 `IgnoreActors`。不再依赖子组件跟随移动来实现完整扫掠。

**事件与停止。** `StopBulletMovement` 静默停止；需要停止原因回调时用 `BulletStop`。原 `OnBulletStop` 与新增 `OnBulletMovementStop` 会在同一次原因停止中广播，不要为相同伤害/回收同时绑定两次。`OnBulletRetun` 表示返回开始，完成由停止原因 `Return` 表示。新增枚举值追加在原 `Time/Distance/Speed/Return` 之后。

**对象池。** 激活重置速度、时间和距离等单次状态，但不会自动恢复已消耗反弹次数、命中减速数组、持久力和游戏逐发参数。按[手册](README.md#尾迹停止与对象池)重新填充。尾迹随 Actor 销毁清理，不依赖它在销毁后自行留存。

**C++。** 加速度、减速度和枚举现在有各自头文件：`FBulletAcceleInfo.h`、`FBulletDeceleInfo.h`、`E_BulletAccleType.h`、`E_BulletRefractType.h`、`E_BulletStopReason.h`。部分只读运行状态或内部标志不再作为可写蓝图参数公开；使用正式函数入口，检查直接引用内部字段的代码。`TrailVFX` 仍接收 Niagara System；不要依赖其底层指针包装类型。

### 验证与性能说明

逐版构建使用的具体引擎版本为 4.26.2、4.27.2、5.0.3、5.1.1、5.2.1、5.3.2、5.4.4、5.5.4、5.6.1、5.7.4、5.8.3。检查采用本版 UBT/UHT 和非 Unity 插件 C++ 编译，使用 `NoLink`；没有覆盖每个分支的全部历史补丁，也没有据此宣称每版 DLL 链接、打包或实机已通过。

本次运行观察覆盖 UE5.8 的普通 Actor 激活、停止、注销/销毁与实际子弹集成；这不代表所有目标版本、打包配置或联网模式已经运行通过。跨版本构建验证与场景运行结果应分别理解。没有跨版本统一的 Mass 净性能收益或无卡顿结论；请按游戏实际负载测量。

## 1.0

初版公开使用方式为：给 Actor 添加 BulletMovement，通过参数配置弹道。提供基础速度、射程、时间阶段、加速度曲线、命中减速、偏转、反弹、追踪、返回、尾迹与散射生成辅助函数。
