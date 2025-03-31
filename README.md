# BulletMovement 组件使用手册

## 版本：1.0

### 1. 组件概述

BulletMovement 是一个纯C++，无物理，高性能，完全参数驱动的轻量级子弹移动组件，提供一站式的射击游戏-弹幕游戏的子弹移动解决方案，集成了众多游戏化的、直接上手可用的功能，只需要通过简单的参数配置即可使用。
该组件即插即用，只需要添加进actor里并设置参数，即可在运行后使用

#### 功能支持：
1. **基础弹道控制**：
   - 支持加速/减速，包括固定值和动态曲线方式。
   - 支持阶段式命中速度变化（基于命中次数和阶段控制）。
   - 支持子弹的最大射程控制。
   - 支持阶段化时间控制，每个阶段可以触发特定效果。

2. **特殊轨迹效果**：
   - 支持持续或单次偏转子弹轨迹的效果。
   - 提供高性能数值化反弹效果，支持标准反弹和定向反弹。
   - 支持手动或自动反射的方向性反弹效果。

3. **追踪与导弹尾迹**：
   - 支持基于百分比的诱导追踪效果。
   - 通过曲线动态控制追踪性能的变化。
   - 集成导弹尾迹特效，不会随着子弹消失而立刻消失。
   - 支持回归效果，子弹可以指定回归对象并返回。

### 使用方式：

1. **在 Actor 内添加 BulletMovement 组件**
2. **调整组件参数（详细见下方的说明）**
3. **完成。没有其他任何需要调用的，组件已经可以使用了**
   - 如果有特殊需求，可以参照下方的函数说明，或者查看DemoContent里的内容。
   - 如果关闭了组件的自动激活，则需要手动调用ActiveBulletMovement函数来启动组件。

---

### 组件详细功能介绍：

#### 公开变量

- **InitialSpeed (float)**
  - **描述**：设置子弹启动时的原始速度。
  - **参数**：无
  - **访问性**：组件参数

- **MaxSpeed (float)**
  - **描述**：设置子弹能够达到的最大速度。如果为 0 则不限制子弹的最大速度。
  - **参数**：无
  - **访问性**：组件参数

- **MinSpeed (float)**
  - **描述**：当子弹的速度减至此值以下时，触发停止事件，原因是速度过低。
  - **参数**：无
  - **访问性**：组件参数

- **MaxMoveDistance (float)**
  - **描述**：设置子弹的最大移动距离，超过此距离触发停止事件。
  - **参数**：无
  - **访问性**：组件参数

- **BulletTimeLimits (Array<float>)**
  - **描述**：设置子弹的生存时间，每个阶段的时间限制，超过时间触发停止事件。
  - **参数**：无
  - **访问性**：组件参数

- **bFollowAim (bool)**
  - **描述**：启用后，子弹将在激活时指向目标位置。
  - **参数**：无
  - **访问性**：组件参数

- **AimPoint (Vector)**
  - **描述**：设置子弹的目标方向（世界位置）。
  - **参数**：无
  - **访问性**：组件参数

- **AcceleInfo (Array<BulletAcceleInfo>)**
  - **描述**：设置子弹的加速度，以常量加速度或者动态曲线调整速度变化。
  - **参数**：无
  - **访问性**：组件参数

- **HitDeceleration (Array<BulletDeceleInfo>)**
  - **描述**：设置子弹命中后的减速效果，可以分阶段设置减速的强度。
  - **参数**：无
  - **访问性**：组件参数

- **GravityScale (float)**
  - **描述**：设置子弹受重力影响的程度，与场景的默认重力相乘来影响子弹轨迹。
  - **参数**：无
  - **访问性**：组件参数

- **TrailVFX (NiagaraSystem)**
  - **描述**：为子弹设置拖尾特效，并控制特效的大小和消失时机。
  - **参数**：无
  - **访问性**：组件参数

- **TrailScale (float)**
  - **描述**：控制拖尾特效的大小。
  - **参数**：无
  - **访问性**：组件参数

- **RefractType (BulletRefractType)**
  - **描述**：设置子弹命中后的反射类型，支持标准反弹和定向反射。
  - **参数**：无
  - **访问性**：组件参数

- **RefractTime (int)**
  - **描述**：设置子弹的反射次数。
  - **参数**：无
  - **访问性**：组件参数

- **RefractTargetType (Array<CollisionChannel>)** 
  - **描述**：设置自动反射的检测类型。
  - **参数**：无
  - **访问性**：组件参数

- **RefractTargetMaxRange (float)**
  - **描述**：设置自动反射的最大检测范围。
  - **参数**：无
  - **访问性**：组件参数

- **RefractTargetLimitAngle (float)**
  - **描述**：设置自动反射的最大检测角度。
  - **参数**：无
  - **访问性**：组件参数

- **DefaultHoming (float)**
  - **描述**：控制子弹的追踪效率。
  - **参数**：无
  - **访问性**：组件参数

- **HomingCurve (CurveFloat)**
  - **描述**：通过曲线动态调整追踪强度。
  - **参数**：无
  - **访问性**：组件参数

- **bSweepReturn (bool)**
  - **描述**：设置子弹在回归时是否进行碰撞检测。
  - **参数**：无
  - **访问性**：组件参数

- **ReturnRange (float)**
  - **描述**：设置子弹的回归完成距离。
  - **参数**：无
  - **访问性**：组件参数

#### 公开函数

- **ActiveBulletMovement(AActor* ActiveBullet, bool Active)**
  - **描述**：激活或停用子弹移动。
  - **参数**：
    - **ActiveBullet (Actor)**：要控制的子弹Actor。
    - **Active (bool)**：是否启用（true）或停用（false）移动。
  - **详细说明**：
    - 初始化子弹属性（速度、旋转、拖尾、碰撞）。
    - 当Active为true时，启用组件的Tick功能。

- **SwitchBulletMovement(bool Active, bool Reset)**
  - **描述**：切换子弹移动状态，并可选择重置。
  - **参数**：
    - **Active (bool)**：期望的激活状态。
    - **Reset (bool)**：是否重置移动参数。
  - **详细说明**：
    - 当Reset为true时，内部调用ActiveBulletMovement()。

- **UpdateInitialSpeed(float Speed)**
  - **描述**：更新子弹的初始速度。
  - **参数**：
    - **Speed (float)**：新的初始速度值（单位：每秒）。
  - **详细说明**：
    - 重置CurrentSpeed以匹配新的InitialSpeed。
    - 使用缓存方向更新速度向量。

- **AddAcceleration(BulletAcceleInfo Info)**
  - **描述**：添加加速度配置到子弹。
  - **参数**：
    - **Info (BulletAcceleInfo)**：包含基础加速度值和可选曲线的加速度配置。
  - **详细说明**：
    - 自动启用加速度系统（bUseAccele）。

- **AddDeceleration(Array<BulletDeceleInfo> Info)**
  - **描述**：应用碰撞后的减速阶段。
  - **参数**：
    - **Info (Array<BulletDeceleInfo>)**：包含阶段持续时间和减速量的减速配置数组。
  - **详细说明**：
    - 自动启用减速系统（bUseDecele）。

- **SwitchCollision(bool Begin)**
  - **描述**：控制碰撞检测系统。
  - **参数**：
    - **Begin (bool)**：true = 启用碰撞检测，false = 禁用碰撞检测。
  - **详细说明**：
    - 禁用碰撞检测会阻止命中事件、反射逻辑和减速触发。

- **AddForce(FName Code, Vector Force)**
  - **描述**：应用即时方向力。
  - **参数**：
    - **Code (Name)**：力的分组标识符。
    - **Force (Vector)**：向量大小/方向（世界空间）。
  - **详细说明**：
    - Code=None应用直接力（非持久）。
    - 命名的力会持续存在直到被移除。

- **RemoveForce(FName Code)**
  - **描述**：通过标识符移除持久力。
  - **参数**：
    - **Code (Name)**：要移除的力的标识符。
  - **详细说明**：
    - 无效的标识符会被静默忽略。

- **ClearForce()**
  - **描述**：清除所有活动力。
  - **详细说明**：
    - 重置DirectForce向量。
    - 重置CurrentLoopForce向量。
    - 清空持久力映射。

- **AddRefractTarget(Array<SceneComponent*> Targets)**
  - **描述**：配置反射目标。
  - **参数**：
    - **Targets (Array<SceneComponent*>)**：用于定向反射的组件数组。
  - **详细说明**：
    - 适用于E_BulletRefractType::Target类型的反射。

- **SetupHoming(SceneComponent* Target)**
  - **描述**：启用追踪行为。
  - **参数**：
    - **Target (SceneComponent)**：要追踪的组件。
  - **详细说明**：
    - 启用追踪系统（bUseHoming）。
    - 重置曲线计时。

- **SwitchHomingCurve(bool Begin, CurveFloat* Curve = nullptr)**
  - **描述**：控制追踪曲线应用。
  - **参数**：
    - **Begin (bool)**：true = 应用曲线调制。
    - **Curve (CurveFloat)**：可选的替换曲线。
  - **详细说明**：
    - 维护与子弹阶段计时器分离的独立计时。

#### 公开事件

- **OnBulletHomingLost**
  - **描述**：当当前目标无法追踪时触发（例如目标销毁）。
  - **参数**：无

- **OnBulletRefracted**
  - **描述**：子弹触发反射时，通知命中信息和反射速度。
  - **参数**：
    - **Hit (HitResult)**：命中结果。
    - **Velocity (Vector)**：反射后的速度。

- **OnBulletStepUp**
  - **描述**：子弹达到下一个时间阶段时触发。
  - **参数**：
    - **Step (int)**：当前阶段。

- **OnBulletDecele**
  - **描述**：子弹触发减速时，通知当前速度和减速阶段。
  - **参数**：
    - **CurrentSpeed (float)**：当前速度。
    - **DeceleStep (int)**：减速阶段。

- **OnBulletRetun**
  - **描述**：子弹开始返回到其原始位置或目标时触发。
  - **参数**：
    - **ReturnMesh (SceneComponent)**：返回目标的网格组件。
    - **ReturnSocket (Name)**：返回目标的插槽名称。

- **OnBulletStop**
  - **描述**：子弹停止移动时，通知停止原因。
  - **参数**：
    - **Reason (BulletStopReason)**：停止原因。
