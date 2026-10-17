# BulletMovement 2.0 User Guide

[简体中文](https://github.com/KamishiroRei/BulletMovementUseDoc/blob/main/README.md) · [Changes and migration](https://github.com/KamishiroRei/BulletMovementUseDoc/blob/main/CHANGELOG.en.md)

BulletMovement is a projectile movement component for ordinary Actors. Add **BulletMovement** to your bullet Actor, set its speed, range, and lifetime, and it can move as soon as it activates. From there, you can add acceleration curves, gravity, homing, return, reflection, hit deceleration, and Niagara trails through the same component.

Version 2.0 integrates Mass automatically on UE5. You still add the component, configure each shot, and use its functions and events from your bullet Actor. UE4 uses the same motion calculations from component Tick. Your project supplies damage, explosions, weapon rules, and pooling policy.

## Installation

Install the package for your engine version through Fab, or place the purchased `BulletMovement` folder in your project's `Plugins` directory. Enable BulletMovement under **Edit → Plugins**, enable its requested dependencies, and restart the editor. UE5 packages use Mass. UE4 packages do not require Mass. Trails use Niagara.

The release targets **Win64**, with source packages for UE 4.26, 4.27, and 5.0–5.8: 11 engine branches in total. Fab availability follows the product listing. Source packages require a C++ build environment suitable for the engine version you use. See [validation and performance scope](https://github.com/KamishiroRei/BulletMovementUseDoc/blob/main/CHANGELOG.en.md#validation-and-performance-scope) for what has been compiled and observed at runtime.

Keep only one installation of BulletMovement enabled in a project. In particular, avoid loading both an engine copy and a project copy. This repository contains the documentation; the commercial plugin source is provided through the purchase channel. Demo maps and example assets are not included, so the steps below use a bullet Actor and collision shape you create yourself.

## Five minutes: a moving bullet that stops

1. Create an Actor Blueprint such as `BP_Bullet`. Add a Sphere Collision or Box Collision and make it the root. Set **Mobility = Movable**, **Simulate Physics = false**, and **Collision Enabled = Query Only**. Set the bullet and a test wall to Block each other. Attach your visible mesh beneath the collision root and disable its collision if it is only for display.
2. Add **BulletMovement** and leave **Auto Activate** enabled. You do not need to call an activation function from BeginPlay.
3. Set `InitialSpeed = 3000`, `MaxSpeed = 0`, `MinSpeed = 0`, and `MaxMoveDistance = 6000`. Set `DefaultBulletTimeLimits` to `[3.0]`. Leave `GravityScale = 0` and `RefractType = None`. Speeds use cm/s, distances use cm, and time uses seconds.
4. Place the Actor in the level or create it with Spawn Actor. Point its +X axis along the firing direction, put the blocking wall ahead, and play.
5. Bind the component's `OnBulletStop` event. For this example, call Destroy Actor from that event. The bullet stops when it reaches its range or time limit, or reports `Hited` when it hits the wall. Stopping movement does not destroy the Actor by itself.

The default initial speed is 0, and the default lifetime is a single one-second stage. Set a nonzero speed for the first test. If the legacy `BulletTimeLimits` array has entries, BeginPlay copies them over `DefaultBulletTimeLimits`; use `DefaultBulletTimeLimits` consistently for new content.

If a shot needs its shooter, ignored Actors, target, or other data assigned before it moves, turn off Auto Activate. Configure it, then call `Activate` with Reset enabled or `ActiveBulletMovement()`. You can also use deferred Actor spawning to configure the component before finishing the spawn. `ActiveBulletMovement` accepts an optional Actor and an `Active` flag; omitting the Actor uses the component's owner, and passing `Active = false` stops movement.

## Collision and ignored actors

Movement sweeps `UpdatedComponent`, which is normally the Actor's collision root. Child components follow that movement, but each child does not receive a separate full movement sweep. Make the shape that should detect obstacles the root.

For a common Blueprint layout, the component can choose the collision root automatically. If the Actor still has an untouched `DefaultSceneRoot` and **exactly one descendant PrimitiveComponent with query collision enabled**, the component promotes that shape to the root and preserves world transforms. A Query Only shape qualifies. With several candidates, a custom root, or a different explicitly selected `UpdatedComponent`, it preserves your setup. In those cases, choose the collision root yourself.

The bullet adds itself to `IgnoreActors` automatically. Add the shooter and any other Actors the bullet should pass through; setting Owner or Instigator does not automatically add all of them. `IgnoreActors` applies to movement sweeps and automatic targeted-reflection queries. Changes made at runtime are synchronized with movement, while existing native movement-ignore entries are preserved.

Each ignore entry covers a whole Actor, rather than an individual component. It also applies only to the bullet's queries: it does not make another object ignore the bullet when that object sweeps toward it.

`SwitchCollision(false)` turns off sweeps for this component's movement. It does not change every component's Collision Enabled setting. Return movement has a separate `bSweepReturn` setting. Collision channels and both objects' responses still determine which interactions block or overlap. Connect your damage and explosion logic to the appropriate collision events; BulletMovement does not call Apply Damage for you.

## Speed, range, and time stages

| Setting | How it works |
| --- | --- |
| `InitialSpeed` / `CurrentSpeed` | Speed at activation and current scalar speed, in cm/s. `UpdateInitialSpeed(Speed)` changes both values and updates the velocity vector. |
| `MaxSpeed` | Caps speed during acceleration updates. Zero leaves acceleration uncapped. It does not cap every force input or a velocity you assign directly. |
| `MinSpeed` | Stops movement with reason `Speed` when scalar speed is at or below this value. |
| `MaxMoveDistance` | Limits travel before return, in cm. A positive value sets a limit; zero gives unlimited distance. `bCheckDistance = false` turns off distance checks. |
| `DefaultBulletTimeLimits` | Holds the duration of each time stage, in seconds. Activation copies it into the runtime stage array. Entries are individual durations, not cumulative timestamps. |
| `GravityScale` | Multiplies world gravity. Zero turns gravity off. It does not require rigid-body simulation. |

For example, `[0.5, 1.0, 2.0]` creates three stages lasting 0.5, 1, and 2 seconds. `OnBulletStepUp(Step)` fires when the bullet enters the next available stage. Stage indices start at zero, so the first transition reports 1. After the final stage, the bullet stops with reason `Time`.

Time checks run during movement updates, rather than through a precision timer. An empty stage array stops the bullet at the update check; it does not give the bullet an unlimited lifetime.

`MovedDistance` records the distance actually traveled, and `MoveableDistance` holds the shot's remaining range. During return, return-completion conditions take the place of the ordinary range checks.

## Acceleration, hit deceleration, and deflection

Configure acceleration in `AcceleInfo`, or call `AddAcceleration(Info)` to append an entry and enable acceleration at runtime. Several entries contribute together. With the default `bScaleByDelta = true`, a constant `DefaultAccle` is measured in cm/s². For example, starting at 1000 cm/s with an acceleration of 500 adds approximately 500 cm/s each second, until a positive `MaxSpeed` caps the speed.

There are two ways to use a curve:

- Assign a CurveFloat asset to `AccleCurve`. X is elapsed time for that acceleration entry, in seconds; Y is multiplied by `DefaultAccle`. The multiplier defaults to 0, so assigning a curve alone produces no acceleration. Set the multiplier as well.
- Set `AccleType = Curve` and edit `RuntimeAccleCurve` for an inline curve. Its Y values specify acceleration directly.

An assigned `AccleCurve` asset takes priority over the inline curve. `StartAcceleration()` resumes acceleration and `StopAcceleration()` pauses it; neither clears the entries or resets their curve time. If you set `bScaleByDelta = false`, the value becomes a speed increment per update. That makes the result depend on update frequency, so use it only when you want that behavior.

`HitDeceleration` removes speed on applicable blocking hits. In each entry, `Decele` is the amount removed in cm/s, and `MaxTime` is the number of hits for that entry, **not a duration in seconds**. You can append entries and enable hit deceleration with `AddDeceleration(Info)`.

Hit deceleration is most useful with reflection, which lets the bullet continue moving after a hit. A normal blocking hit without reflection stops the bullet immediately. Each applicable hit consumes the first entry. When that entry runs out, the last entry takes its place, so an array of three or more entries does not necessarily play in its original order. `OnBulletDecele(CurrentSpeed, DeceleStep)` reports the resulting speed and deceleration stage. If you need a strict custom sequence, supply the next entry from your game when the stage changes.

For deflection, call `AddForce(None, Vector)` to add a one-shot world-space velocity change on the next movement update. A named `AddForce` entry supplies persistent world-space acceleration. Calling it again with the same name replaces that entry. Use `RemoveForce(Name)` to remove one entry, or `ClearForce()` to clear persistent and pending one-shot forces. These inputs change the calculated trajectory; they are not mass-based rigid-body Add Force operations.

## Homing and initial aim

Set `bFollowAim = true` to point the bullet toward the world-space `AimPoint` when it activates. This sets its initial direction. To keep steering toward a moving target, set a positive `DefaultHoming` and call `SetupHoming(TargetComponent)`, or assign `HomingTarget` before activation. Any valid SceneComponent can be a target.

`DefaultHoming` controls how strongly each update blends the direction toward the target. Larger values turn faster. The public component multiplies the blend amount by DeltaTime; this is not a strict angular-speed limit or a deterministic network simulation.

Use `HomingCurve` to vary steering over time. X is elapsed homing time in seconds, and Y multiplies homing strength. `SetupHoming` resets the homing curve clock, which is separate from the bullet's time-stage clock. `SwitchHomingCurve` enables or disables the curve, or replaces it with another curve. `HomingEndRange` stops further steering once the bullet is within that distance of the target, in cm.

`SwitchHoming(false)` pauses homing. `HomingLost()` ends it explicitly and fires `OnBulletHomingLost`. If the target becomes invalid, for example because it is destroyed, the component also fires that event and turns homing off. The bullet continues under its other movement rules; it does not select a replacement target automatically.

## Return

Call `SetupReturn(ReturnComponent, SocketName)` to send the bullet toward a component's socket position. Use None for the socket name when no socket is needed. `OnBulletRetun(ReturnMesh, ReturnSocket)` fires when return **starts**. Its spelling is retained for compatibility.

`SetupReturn` sets `CurrentSpeed` to `MaxSpeed`. Set a positive value, such as 3000, when you use return. Zero leaves the return leg stationary, even though it means unlimited for ordinary acceleration.

`ReturnRange` is the completion radius in cm; 100 means one meter. `bSweepReturn` defaults to false, so the bullet can return through obstacles. Set it to true to sweep for collision during return.

The bullet stops with reason `Return` when it reaches the completion radius. An invalidated return target ends movement with the same reason, allowing both cases to use the same recycling logic. Ordinary distance and time-stage limits do not end a return leg, and return has no separate timeout. Your game can interrupt it when needed. If one movement step can carry the bullet much farther than the completion radius, reduce its speed, increase the radius, or shorten the update interval.

## Reflection and targeted reflection

For a normal bounce, set `RefractType = Normal`, clear the targeted-reflection list, and give `RefractTime` a positive count. The bullet reflects around the blocking hit's normal. With no explicit target list, zero disables reflection and -1 allows reflection without running out of counts. Reflection requires a blocking collision. `OnBulletRefracted(Hit, Velocity)` reports the hit and the new velocity.

In `Target` mode, a blocking hit redirects the bullet toward target components. `SetRefractTarget(Targets)` replaces the explicit list, while `AddRefractTarget(Targets)` appends to it. Keep those component references valid.

To search for targets automatically, leave the explicit list empty, choose the object channels in `RefractTargetType`, and set `RefractTargetMaxRange` as the search radius in cm. You can also set a forward cone angle with `RefractTargetLimitAngle`; zero disables that angle filter. The scene query does not guarantee candidates are ordered by distance, so automatic reflection should not be treated as nearest-enemy selection.

Both the target list and the remaining count affect reflection. If the list is nonempty, setting `RefractTime` to zero alone may not disable it. Use `RefractType = None` to turn reflection off. For pooled bullets, refill the list and count before each shot if you want to repeat the same chain. Your project supplies team filtering, target-selection rules, and damage cooldowns.

## Trails, stopping, and pooling

Assign a Niagara System to `TrailVFX` and set `TrailScale`. At runtime, `SetupTrailVFX(VFX, Scale)` replaces the trail. It follows the bullet root and deactivates when movement stops. The trail component is cleaned up when the Actor is destroyed, so it does not persist independently after destruction. Niagara settings control how particles fade. To let a trail finish naturally, stop the bullet and delay destruction, or create a separate finishing effect in your game.

There are two ways to stop:

- `StopBulletMovement()` stops movement and the trail without broadcasting a stop-reason event. `Deactivate()` also provides a way to stop when recycling.
- `BulletStop(Reason)` broadcasts `OnBulletMovementStop` first, then `OnBulletStop`, and then stops movement. Bind a particular damage or recycling action to one of those events to avoid running it twice.

Common automatic reasons are `Time`, `Distance`, `Speed`, `Return`, and `Hited`. Your game can also pass `Dameged` or `Direct` explicitly. The component does not implement damage handling. The public spellings `Hited`, `Dameged`, and `OnBulletRetun` remain unchanged for compatibility.

For a pool, prepare each shot explicitly:

1. Turn off Auto Activate when creating pool Actors. Recycle with `Deactivate()` or `StopBulletMovement()`, then manage the Actor's visibility and collision.
2. Before firing again, set the world transform, initial speed, range, `DefaultBulletTimeLimits`, targets, and per-shot `IgnoreActors`. Keep the bullet itself in the ignore array.
3. Refill consumed `HitDeceleration` entries, `RefractTime`, and reflection targets. Clear persistent forces that should not carry over. Reset acceleration-entry `AccleTime` and homing switches as needed. The internal deceleration-stage counter reported by `OnBulletDecele` does not reset automatically on activation; Blueprint users who need it to start at zero for every shot can keep their own per-shot counter.
4. Restore visibility and the required collision setup. Call `Activate` with Reset enabled, `ActiveBulletMovement()`, or `SwitchBulletMovement(true, true)`. Apply any homing or return behavior that needs to follow activation afterward.

Activation resets speed, remaining distance, time stages, and return state, clears `CurrentRefTarget`, and re-enables ordinary movement sweeps. It does not restore every property to its class default. `SwitchBulletMovement(true, false)` leaves an already active shot running without resetting it; pass Reset when you want to start a new shot.

On UE5, the component creates and releases its Mass entity and handles unregistration and destruction. Do not create another Mass movement entity for the same bullet. Use one system to control its position: another movement component or physics simulation that writes to it at the same time will conflict.

Some event output pin names differ between UE4.26–UE5.1 and later engines. Check the [upgrade notes](https://github.com/KamishiroRei/BulletMovementUseDoc/blob/main/CHANGELOG.en.md) when moving between branches; the event values have the same meaning.

## Execution on UE4 and UE5

The Actor setup, component functions, and numerical motion kernel are shared by both backends. On UE4.26/4.27, component Tick runs the motion calculations and scene movement without Mass. On UE5, the component gathers Actor and component input on the game thread, a Mass processor computes motion, and the game thread applies scene movement, collision, and events. The component handles registration; you do not need a special bullet base class, manager Actor, or Mass Trait asset.

UE5.0–5.3 process chunks sequentially within the Mass motion processor's worker task. UE5.4 and later use parallel chunk queries. UE5.8 also includes support for scene publication and batched queries through that version's Chaos interfaces. Other engine versions use ordinary component scene movement, so the UE5.8 implementation should not be assumed to apply unchanged to them.

The component's default update interval remains approximately 0.016 seconds. That interval, frame rate, collision-shape size, and bullet speed affect trajectories and event timing. Mass schedules motion work, but Actors, collision, events, Niagara, and your Blueprint logic still contribute to frame time. No universal throughput, overall speedup, or hitch-free result has been established across engine versions. Measure complete frame time on your target hardware using your actual projectile workload.

## Platform and networking scope

The release platform is **Win64**. BulletMovement does not provide projectile-state replication, RPCs, client prediction, replay, or server correction. A Mass processor being allowed to run in client and server worlds does not synchronize the projectiles between them. For a networked game, implement spawn and damage authority, replication, and visual presentation in your project. Check each released package's validation scope before relying on other platforms, dedicated servers, or packaged-runtime coverage.

The optional Blueprint helpers `GetShotAngle` and `SpawnBulletActor` can create spread transforms or spawn a specified Actor class. Both take a count, a cone, and a randomness setting; the cone's X and Y values control horizontal and vertical spread. `SpawnBulletActor` also accepts optional Owner and Instigator arguments. These helpers do not implement pooling, authoritative network spawning, or damage.

Before upgrading existing content, read [Changes and migration](https://github.com/KamishiroRei/BulletMovementUseDoc/blob/main/CHANGELOG.en.md), especially the notes on time arrays, consumed settings, collision roots, and C++ headers.
