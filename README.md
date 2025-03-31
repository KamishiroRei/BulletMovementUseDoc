# BulletMovement Component User Manual

## Version: 1.0

### 1. Component Overview

BulletMovement is a pure C++, physics-free, high-performance, fully parameter-driven lightweight bullet movement component. It provides an all-in-one solution for projectile movement in shooting and bullet-hell games. With a highly game-oriented design, this component is easy to use and ready out of the box by simply configuring parameters.

This component is plug-and-play: simply add it to an Actor and configure the parameters to use it.

#### Supported Features:
1. **Basic Ballistic Control**:
   - Supports acceleration and deceleration, including both constant values and dynamic curves.
   - Enables staged speed variation based on hit count and phase control.
   - Allows setting a maximum range for bullets.
   - Supports stage-based time control, triggering specific effects at each phase.

2. **Special Trajectory Effects**:
   - Supports continuous or one-time trajectory deflection.
   - Provides high-performance numerical-based reflection, including standard and directional reflection.
   - Enables manual or automatic directional reflection.

3. **Homing and Missile Effects**:
   - Supports percentage-based homing effects.
   - Allows dynamic control of homing performance through curves.
   - Integrates missile trails that persist beyond bullet lifespan.
   - Supports return effects, enabling bullets to return to a specified target.

### How to Use:
1. **Add the BulletMovement component to an Actor**
2. **Adjust the component parameters (see detailed explanations below)**
3. **Done. No further calls required—it's ready to use.**
   - For special requirements, refer to the function documentation below or check the DemoContent.
   - If auto-activation is disabled, manually call the `ActiveBulletMovement` function to activate the component.

---

### Detailed Component Features:

#### Public Variables

- **InitialSpeed (float)**
  - **Description**: Sets the initial speed of the bullet upon activation.
  - **Access**: Component parameter

- **MaxSpeed (float)**
  - **Description**: Sets the maximum speed the bullet can reach. A value of 0 means no speed limit.
  - **Access**: Component parameter

- **MinSpeed (float)**
  - **Description**: When the bullet's speed drops below this value, it triggers a stop event due to insufficient velocity.
  - **Access**: Component parameter

- **MaxMoveDistance (float)**
  - **Description**: Defines the maximum distance the bullet can travel before triggering a stop event.
  - **Access**: Component parameter

- **BulletTimeLimits (Array<float>)**
  - **Description**: Defines the bullet's lifespan with staged time limits, triggering a stop event when exceeded.
  - **Access**: Component parameter

- **FollowAim (bool)**
  - **Description**: If enabled, the bullet will face the target direction upon activation.
  - **Access**: Component parameter

- **AimPoint (Vector)**
  - **Description**: Defines the bullet's target direction (world position).
  - **Access**: Component parameter

- **AcceleInfo (Array<BulletAcceleInfo>)**
  - **Description**: Configures bullet acceleration, either as a constant value or through dynamic curves.
  - **Access**: Component parameter

- **HitDeceleration (Array<BulletDeceleInfo>)**
  - **Description**: Configures deceleration effects upon impact, allowing stage-based deceleration strength.
  - **Access**: Component parameter

- **GravityScale (float)**
  - **Description**: Defines the bullet's gravity influence by multiplying the scene's default gravity.
  - **Access**: Component parameter

- **TrailVFX (NiagaraSystem)**
  - **Description**: Sets the bullet's trail effect, controlling size and fade-out timing.
  - **Access**: Component parameter

- **TrailScale (float)**
  - **Description**: Controls the scale of the trail effect.
  - **Access**: Component parameter

- **RefractType (BulletRefractType)**
  - **Description**: Defines the bullet's reflection type, supporting standard and directional reflection.
  - **Access**: Component parameter

- **RefractTime (int)**
  - **Description**: Defines the number of times the bullet can reflect.
  - **Access**: Component parameter

- **RefractTargetType (Array<CollisionChannel>)**
  - **Description**: Specifies the collision channels for automatic reflection detection.
  - **Access**: Component parameter

- **DefaultHoming (float)**
  - **Description**: Controls the homing efficiency of the bullet.
  - **Access**: Component parameter

- **HomingCurve (CurveFloat)**
  - **Description**: Dynamically adjusts homing intensity through curves.
  - **Access**: Component parameter

#### Public Functions

- **ActiveBulletMovement(Actor ActiveBullet, bool Active)**
  - **Description**: Activates or deactivates bullet movement.
  - **Parameters**:
    - **ActiveBullet (Actor)**: The bullet actor to control.
    - **Active (bool)**: `true` to enable movement, `false` to disable.

- **SwitchBulletMovement(bool Active, bool Reset)**
  - **Description**: Toggles bullet movement state, with optional reset.

- **UpdateInitialSpeed(float Speed)**
  - **Description**: Updates the bullet's initial speed.

- **AddAcceleration(BulletAcceleInfo Info)**
  - **Description**: Adds acceleration settings to the bullet.

- **AddDeceleration(Array<BulletDeceleInfo> Info)**
  - **Description**: Applies deceleration stages upon impact.

- **SwitchCollision(bool Begin)**
  - **Description**: Enables or disables collision detection.

- **AddForce(Name Code, Vector Force)**
  - **Description**: Applies an instant directional force.

- **RemoveForce(Name Code)**
  - **Description**: Removes persistent force by identifier.

- **ClearForce()**
  - **Description**: Clears all active forces.

- **AddRefractTarget(Array<SceneComponent*> Targets)**
  - **Description**: Configures reflection targets.

- **SetupHoming(SceneComponent Target)**
  - **Description**: Enables homing behavior towards a target.

- **SwitchHomingCurve(bool Begin, CurveFloat Curve = nullptr)**
  - **Description**: Controls homing curve application.

#### Public Events

- **OnBulletHomingLost**
  - **Description**: Triggered when the current homing target is lost (e.g., destroyed).

- **OnBulletRefracted**
  - **Description**: Triggered when the bullet reflects upon impact.

- **OnBulletStepUp**
  - **Description**: Triggered when the bullet progresses to the next time phase.

- **OnBulletDecele**
  - **Description**: Triggered when the bullet applies a deceleration effect.

- **OnBulletReturn**
  - **Description**: Triggered when the bullet begins returning to its original location or target.

- **OnBulletStop**
  - **Description**: Triggered when the bullet stops moving, providing the reason for stopping.

