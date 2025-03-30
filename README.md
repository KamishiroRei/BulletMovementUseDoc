# BulletMovement 组件使用手册

## 版本：1.0

### 1. 组件概述

`BulletMovement` 是一个轻量级的子弹移动组件，旨在为射击游戏、弹幕游戏等非物理驱动的子弹场景提供简单、灵活的移动控制。通过参数化配置，玩家可以快速实现多种弹道效果，支持从基本的子弹移动到复杂的反射、追踪、回归等效果。

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

### 2. 使用方式

1. **在 Actor 内添加 BulletMovement 组件**
   将 `BulletMovement` 组件添加到角色或物体的 Actor 中。

2. **调整组件参数**：
   - **初始速度**：设置子弹启动时的原始速度。
   - **最大速度**：设置子弹能够达到的最大速度。如果为 0 则不限制子弹的最大速度。
   - **停止速度**：当子弹的速度减至此值以下时，触发停止事件，原因是速度过低。
   - **最大移动距离**：设置子弹的最大移动距离，超过此距离触发停止事件。
   - **最大存在时间/阶段**：设置子弹的生存时间，每个阶段的时间限制，超过时间触发停止事件。
   - **初始速度指向瞄准位置**：启用后，子弹将在激活时指向目标位置。
   - **瞄准方向**：设置子弹的目标方向（世界位置）。

3. **物理与加速度设置**：
   - **加速度信息**：设置子弹的加速度，可以使用动态曲线调整加速度变化。
   - **命中后减速/阶段**：设置子弹命中后的减速效果，可以分阶段设置减速的强度。
   - **重力尺度**：设置子弹受重力影响的程度，与场景的默认重力相乘来影响子弹轨迹。
   
4. **特效设置**：
   - **拖尾特效**：为子弹设置拖尾特效，并控制特效的大小和消失时机。
   - **反射类型**：设置子弹命中后的反射类型，支持标准反弹和定向反射。 
   - **追踪效率**：控制子弹的追踪效率，支持通过曲线调整动态追踪表现。
   - **回归效果**：设置子弹在命中目标后是否回归，并指定回归目标和插槽。

---

### 组件详细功能介绍：
- **反弹效果**：
  - 标准反弹：沿命中法线进行镜像反弹。
  - 定向反弹：根据指定的反射目标进行反弹，支持自动检测反射目标。
  - 自动反射：检测指定距离内的目标并进行反射。

- **追踪与回归**：
  - 追踪效果：控制子弹自动追踪目标，可以手动启用追踪并通过曲线动态调整追踪强度。
  - 回归效果：当子弹停止时，可以根据回归目标进行回归，适用于回旋镖、火箭等类型。

---

### EndlishVer.README.md：

```markdown
# BulletMovement Component User Manual

## Version: 1.0

### 1. Component Overview

`BulletMovement` is a lightweight, performance-friendly movement component designed for shooting games, bullet-hell games, and other non-physics-driven projectile scenarios. With parameterized configuration, this component allows quick implementation of various projectile effects.

#### Supported Features:
1. **Basic Projectile Control**:
   - Persistent acceleration/deceleration based on fixed values or dynamic curves.
   - Stage-based hit velocity changes based on hit count and stage progression.
   - Maximum projectile range control.
   - Time-based control with multiple stages, allowing for complex effects at each stage.

2. **Special Trajectory Effects**:
   - Persistent or single-time deviation of the projectile's direction via force.
   - High-performance, numeric-based bounce effects.
   - Directional rebound effects, both manual and automatic.

3. **Tracking and Missile Trail**:
   - Percentage-based control of tracking performance.
   - Dynamic tracking performance changes controlled by curves.
   - Integrated missile trail effects that do not disappear when the projectile disappears.
   - Return effect where projectiles return to a specified target post-impact.

### 2. Usage

1. **Adding BulletMovement Component to Actor**
   Attach the `BulletMovement` component to an Actor in your game.

2. **Adjusting Component Parameters**:
   - **Initial Speed**: The speed at which the projectile starts.
   - **Max Speed**: The maximum speed the projectile can reach. If set to 0, there is no speed limit.
   - **Stop Speed**: The speed below which the projectile stops, triggering a stop event due to "Speed Below Minimum."
   - **Max Move Distance**: The maximum distance the projectile can travel before stopping.
   - **Max Time/Stages**: Defines the time limit for each stage of the projectile's existence.
   - **Initial Speed Aimed at Target**: If enabled, the projectile will adjust its movement direction toward a specific aim point.

3. **Physics and Acceleration**:
   - **Acceleration Info**: Defines the acceleration applied to the projectile over time, either through a constant value or a curve.
   - **Hit Deceleration/Stage**: Defines the deceleration effect applied to the projectile upon hitting a target.
   - **Gravity Scale**: The level of gravity affecting the projectile's motion.

4. **Special Effects**:
   - **Trail Effect**: Apply a visual effect that follows the projectile as it moves.
   - **Reflection Type**: Configure how the projectile behaves when it hits a surface—either standard bounce or directional reflection.
   - **Tracking Efficiency**: Control how efficiently the projectile tracks its target, with the option to enable dynamic adjustments via a curve.
   - **Return Effect**: The projectile will return to a designated target upon stopping if a valid return object is set.

---
