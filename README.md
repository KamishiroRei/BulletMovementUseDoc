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
