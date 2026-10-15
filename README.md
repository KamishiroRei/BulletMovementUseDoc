# BulletMovement 2.0 User Guide

[简体中文](https://github.com/KamishiroRei/BulletMovementUseDoc/blob/main/README.md) · [Changes and migration](https://github.com/KamishiroRei/BulletMovementUseDoc/blob/main/CHANGELOG.en.md)

BulletMovement adds parameter-driven projectile motion to an ordinary Actor: acceleration and curves, gravity, homing, return, reflection, hit deceleration, distance and time limits, and Niagara trails. Add the **BulletMovement** component to your bullet Actor, configure it, and movement starts automatically. No special bullet base class, manager Actor, or Mass Trait asset is required. Your game supplies damage, explosions, weapon rules, and pooling.

> This guide covers the 2.0 update for Win64, with source packages for 11 engine branches: UE 4.26, 4.27, and UE 5.0–5.8. Current availability on Fab follows the product listing. See the [validation scope](https://github.com/KamishiroRei/BulletMovementUseDoc/blob/main/CHANGELOG.en.md#validation-and-performance-scope) for build and runtime coverage.

## Installation

Install the package for your engine version through Fab, or put the purchased `BulletMovement` folder in your project's `Plugins` directory. Enable BulletMovement under **Edit → Plugins**, enable the requested dependencies, and restart the editor. UE5 packages use Mass; UE4 packages do not require Mass. Niagara supplies optional trails.

Source packages require a C++ build environment that matches your engine. Enable only one copy of BulletMovement per project; do not load both an engine installation and a project installation. This documentation repository contains instructions, while the commercial plugin source is supplied through the purchase channel.

## Five minutes: a moving bullet that stops

1. Create an Actor Blueprint such as `BP_Bullet`. Add a Sphere Collision or Box Collision and make it the root. Set **Mobility = Movable**, **Simulate Physics = false**, and **Collision Enabled = Query Only**. Set the bullet and test wall to Block each other. Attach the visible mesh beneath the collision root; disable mesh collision if it is only visual.
2. Add **BulletMovement** and leave **Auto Activate** enabled. No BeginPlay activation call is necessary.
3. Set `InitialSpeed = 3000`, `MaxSpeed = 0`, `MinSpeed = 0`, and `MaxMoveDistance = 6000`. Set `DefaultBulletTimeLimits` to `[3.0]`. Leave `GravityScale = 0` and `RefractType = None`. Speed is in cm/s, distances in cm, and time in seconds.
4. Place the Actor in the level or create it with Spawn Actor. Point its +X axis along the firing direction, put a blocking wall ahead, and play.
5. Bind the component's `OnBulletStop`. For this minimal example, call Destroy Actor in the event. The bullet stops at its distance/time limit or with reason `Hited` when it reaches the wall. Stopping movement does not itself destroy the Actor.

Initial speed defaults to 0 and the default single time stage lasts one second, so configure a nonzero speed. If the legacy `BulletTimeLimits` field also contains entries, BeginPlay copies that array over `DefaultBulletTimeLimits`; use the latter consistently for new content.

To supply the shooter, ignored actors, target, or other per-shot data before the first movement, turn off Auto Activate, configure the component, and call `Activate` with Reset enabled or `ActiveBulletMovement`. Deferred Actor spawning is another way to configure before completing the spawn.

## Collision and ignored actors

Movement sweeps `UpdatedComponent`, normally the Actor's collision root. Children do not each receive a full movement sweep just because they follow the root. Make the collision shape you need to test the root.

The component also handles a common Blueprint layout: if the root is the untouched `DefaultSceneRoot` and there is **exactly one descendant PrimitiveComponent with query collision enabled**, it promotes that collision component to the root while preserving world transforms. A Query Only shape qualifies. If there are multiple candidates, a custom root, or an explicitly selected different `UpdatedComponent`, your setup is preserved; choose the collision root yourself.

`IgnoreActors` includes the bullet itself automatically. **The shooter, Owner, and Instigator are not all added automatically.** Add the shooter and any other actors that the bullet should pass through. The array applies to movement sweeps and automatic targeted-reflection queries, and runtime edits are synchronized. It ignores entire Actors, not individual components, and does not create a reciprocal ignore rule for another object sweeping toward the bullet.

`SwitchCollision(false)` disables sweeps for this component's movement; it does not globally change every component's Collision Enabled setting. Return sweeps have their own `bSweepReturn` setting. UE collision channels and both objects' responses still determine blocking and overlaps. Connect your game's damage and explosion logic to collision events; the plugin does not call Apply Damage automatically.

## Speed, range, and time stages

| Setting | Current behavior |
| --- | --- |
| `InitialSpeed` / `CurrentSpeed` | Activation speed / current scalar speed in cm/s. `UpdateInitialSpeed` updates both and the current velocity vector. |
| `MaxSpeed` | Speed cap applied during acceleration updates. Zero means unlimited. It is not a global limiter for arbitrary forces or direct velocity assignments. |
| `MinSpeed` | Stops with reason `Speed` when scalar speed is at or below this threshold. |
| `MaxMoveDistance` | Positive values limit outbound travel in cm; zero means no distance limit. `bCheckDistance = false` disables distance checks. |
| `DefaultBulletTimeLimits` | Duration of each individual time stage in seconds, rather than cumulative timestamps. Copied into the runtime stage array on activation. |
| `GravityScale` | Multiplier for world gravity. Zero disables gravity; rigid-body simulation is unnecessary. |

For example, `[0.5, 1.0, 2.0]` gives three stages lasting 0.5, 1, and 2 seconds. `OnBulletStepUp(Step)` fires when entering a remaining stage. Stages are zero-based, so the first transition reports 1. Once all stages are consumed, the bullet stops with reason `Time`. Checks occur during movement updates, so this is not a precision timer; an empty array stops at the update check.

`MovedDistance` records actual movement and `MoveableDistance` records the shot's remaining range. Outbound range checks are replaced by return completion conditions during return.

## Acceleration, hit deceleration, and deflection

Add entries to `AcceleInfo`. With the default `bScaleByDelta = true`, constant `DefaultAccle` uses cm/s² and changes scalar speed according to elapsed time. An initial speed of 1000 and acceleration of 500 add approximately 500 cm/s each second, up to a positive `MaxSpeed`. Multiple entries add together.

For a CurveFloat asset, assign `AccleCurve`: X is elapsed seconds for that entry, and Y is multiplied by `DefaultAccle`. For an inline curve, select `AccleType = Curve` and edit `RuntimeAccleCurve`; its Y values are acceleration directly. An assigned `AccleCurve` takes priority. `StartAcceleration` and `StopAcceleration` enable or pause application; pausing does not clear entries or reset curve time. Setting `bScaleByDelta = false` makes the value a per-update speed increment whose result depends on update frequency; use it only when that behavior is intentional.

In `HitDeceleration`, `Decele` is the speed removed per applicable hit in cm/s. `MaxTime` is a **hit count, not seconds**. This is primarily useful with reflection, which lets movement continue; a normal non-reflecting blocking hit stops immediately. Each hit consumes the current first entry; when exhausted, it is replaced by the last entry. Do not rely on three or more entries retaining their original array order. `OnBulletDecele` reports the resulting speed and deceleration stage. For a strict custom sequence, your game can supply the next entry when the stage changes.

`AddForce(None, Vector)` applies a one-shot world-space velocity change on the next movement update. A named `AddForce` supplies persistent world-space acceleration; calling it with the same name replaces that entry. Remove one with `RemoveForce(Name)`, or all with `ClearForce()`. These are numerical trajectory inputs, not mass-based rigid-body Add Force operations.

## Homing and initial aim

With `bFollowAim = true`, activation points the bullet toward the world-space `AimPoint`. This sets the initial direction. For ongoing homing, set a positive `DefaultHoming` and call `SetupHoming(TargetComponent)`, or assign `HomingTarget` before activation. The target can be any valid SceneComponent.

`DefaultHoming` controls the per-update blend toward the target direction; larger values turn faster. This public homing mode multiplies its blend amount by DeltaTime and is not a strict angular-speed limit or a deterministic network simulation. For `HomingCurve`, X is elapsed homing time in seconds and Y multiplies homing strength. Use `SwitchHomingCurve` to enable, disable, or replace the curve. `HomingEndRange` stops further steering within the given distance in cm. `SwitchHoming(false)` pauses homing; `HomingLost()` explicitly ends it and raises the loss event.

If the target becomes invalid, such as when it is destroyed, `OnBulletHomingLost` fires and homing turns off. Other movement rules continue; the component does not automatically acquire a replacement target.

## Return

Call `SetupReturn(ReturnComponent, SocketName)` to head toward a component's socket position. Use None when no socket is needed. The compatibility-named `OnBulletRetun` event fires when return **starts**.

Return speed is taken from `MaxSpeed`, so give it a positive value such as 3000 when using return. Although `MaxSpeed = 0` means unlimited during ordinary acceleration, it cannot provide a return speed. `ReturnRange` is the completion radius in cm: 100 means one meter. `bSweepReturn` defaults to false, allowing return through obstacles; true enables swept collision.

Reaching that radius stops with reason `Return`. An invalidated return target also ends with the same reason so it can use the same recycling path. Outbound distance and time stages do not end a return leg. Return has no separate timeout; your game may interrupt it. If a single movement step is much longer than the completion radius, reduce speed, enlarge the radius, or shorten the update interval.

## Reflection and targeted reflection

Set `RefractType = Normal`, clear the targeted-reflection list, and set a positive `RefractTime` to reflect direction around the hit normal. With no explicit target list, zero disables reflection; -1 serves as a count that does not run out. Reflection requires a blocking collision. `OnBulletRefracted` supplies the hit result and reflected velocity.

In `Target` mode, a hit redirects the bullet toward target components. `SetRefractTarget` replaces the explicit list; `AddRefractTarget` appends to it. Keep target references valid. With no explicit targets, set the object channels in `RefractTargetType`, the search radius in `RefractTargetMaxRange`, and optionally the forward cone angle in `RefractTargetLimitAngle`. Zero angle disables the angle filter. Automatic candidates come from a scene query and are not guaranteed to be sorted by nearest distance; do not rely on automatic nearest-enemy selection.

Both the target list and remaining reflection count affect subsequent behavior. With a nonempty list, setting the count to zero alone may not disable reflection; use `RefractType = None` to turn reflection off. Refill the list and count for each shot if pooled bullets should repeat the same chain. Your game supplies team filtering, target selection rules, and damage cooldowns.

## Trails, stopping, and pooling

Assign a Niagara System to `TrailVFX` and set `TrailScale`, or replace it at runtime with `SetupTrailVFX`. The trail follows the bullet root, deactivates on stop, and is cleaned up when the Actor is destroyed. Niagara asset settings determine particle fade; the plugin does not guarantee that a trail survives Actor destruction. For a natural finish, stop the bullet before delayed destruction or spawn a separate finishing effect in your game.

`StopBulletMovement()` silently stops movement and the trail without a stop-reason event. `BulletStop(Reason)` broadcasts `OnBulletStop` and another stop event, `OnBulletMovementStop`, then stops. Usually bind only one for a particular gameplay action to avoid duplicate damage or recycling. Common automatic reasons are `Time`, `Distance`, `Speed`, `Return`, and `Hited`. Your game may pass `Dameged` or `Direct` explicitly; the component does not implement damage handling. Compatibility spellings `Hited`, `Dameged`, and `OnBulletRetun` are retained.

Recommended pooling sequence:

1. Turn off Auto Activate for pool creation. Recycle with `Deactivate()` or `StopBulletMovement()`, then manage visibility and collision.
2. Before another shot, set its world transform, initial speed, range, `DefaultBulletTimeLimits`, targets, and per-shot `IgnoreActors`. Keep the bullet itself in that array.
3. Refill consumed `HitDeceleration`, `RefractTime`, and reflection targets. Clear unwanted persistent forces. Reset acceleration entry `AccleTime` and homing switches as needed. The internal deceleration stage reported by `OnBulletDecele` is not automatically reset on activation; Blueprint users needing a count starting at zero for every shot can maintain their own per-shot counter. Activation reset does not restore every property to class defaults.
4. Restore visibility and the required collision setup. Call `Activate` with Reset enabled, `ActiveBulletMovement()`, or `SwitchBulletMovement(true, true)`, then apply any homing/return behavior that must follow activation. Activation resets speed, remaining distance, time stages, and return state, and re-enables outbound movement sweeps.

On UE5, the component manages entity creation, release, unregistration, and destruction automatically. Do not create a second Mass movement entity for the same bullet or combine it with another movement component or physics simulation that also writes its position.

## Execution on UE4 and UE5

Both backends share the numerical motion kernel and component workflow. UE4.26/4.27 run that kernel and scene movement from component Tick, without Mass. UE5 gathers Actor/component input on the game thread, computes numerical motion in a Mass processor, then performs scene movement, collision, and events on the game thread. No manual entity registration is required.

UE5.0–5.3 traverse chunks sequentially within the Mass motion processor's worker task; UE5.4 and later use parallel chunk queries. UE5.8 additionally includes scene-publication and query-batching support for its corresponding Chaos interfaces. Other versions use ordinary component scene movement. This 5.8 optimization is not an identical implementation across all engine versions.

The component retains its update interval, approximately 0.016 seconds by default. Interval, frame rate, collision shape size, and speed affect trajectory and event timing. Mass schedules motion work; Actors, collision, events, Niagara, and your Blueprint logic still have costs. No universal cross-version throughput, net speedup, or hitch-free guarantee has been established. Measure complete frame time on your target hardware with the actual projectile workload.

## Platform and networking scope

The release target platform is **Win64**. The plugin does not supply projectile state replication, RPCs, client prediction, replay, or server correction. Mass processors being eligible to run in client/server worlds does not automatically synchronize projectiles. Networked games must implement spawn/damage authority, replication, and visual presentation. Refer to each released package's actual validation scope for other platforms, dedicated servers, and packaged runtime coverage.

The optional Blueprint helpers `GetShotAngle` and `SpawnBulletActor` generate spread transforms or spawn a specified Actor class using count, cone, and randomness settings. They do not replace pooling, authoritative network spawning, or a damage system.

Before upgrading existing content, read [Changes and migration](https://github.com/KamishiroRei/BulletMovementUseDoc/blob/main/CHANGELOG.en.md), especially the notes on time arrays, consumed settings, collision roots, and C++ headers.
