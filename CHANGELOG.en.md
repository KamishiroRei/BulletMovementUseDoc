# BulletMovement Changes and Migration

[User guide](README.en.md) · [简体中文](CHANGELOG.zh-CN.md)

## 2.0

These notes correspond to `VersionName = 2.0` in the plugin descriptor, with Win64 source packages for 11 engine branches: UE 4.26, 4.27, 5.0, 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, and 5.8. Each package passed its corresponding UBT/UHT and plugin C++ object compilation. This does not establish linking, runtime, or packaged-build coverage on every version.

- The BulletMovement component manages its movement lifecycle on ordinary Actors, without a special Actor base class, manager, or Mass Trait asset.
- UE5 uses automatic Mass scheduling, separating game-thread input, numerical motion processing, and game-thread scene movement/events. UE4 runs the same numerical kernel through component Tick.
- UE5.0–5.3 traverse chunks sequentially on the Mass worker; UE5.4 and later use parallel chunk queries. UE5.8 additionally supplies scene-publication and query-batching support for the corresponding Chaos interfaces.
- Retains the public `UBulletMovementComponent` class name and common Blueprint entry points/events, including the compatibility spelling `OnBulletRetun`. Adds explicit stop reasons and the `OnBulletMovementStop` event entry point.
- Synchronizes `IgnoreActors` to movement sweeps while preserving pre-existing native ignore entries. Supports automatic promotion of one query-enabled collision shape beneath an untouched `DefaultSceneRoot`.
- Retains homing, return, reflection, acceleration, hit deceleration, time/range limits, and Niagara trail workflows, with pooling activation/stopping integrated into the UE5 entity lifecycle.
- Fixes a pooled shot retaining the previous shot's current reflection target. Each activation clears `CurrentRefTarget`, allowing a refilled reflection chain to start at its first target.

### Upgrading from 1.0

Keep a recoverable copy of your own project before upgrading. Rebuild with the matching engine, open existing bullet Blueprints, and review settings and pins. A retained class name does not guarantee that all C++ source or Blueprint pins are unchanged; recompile your subclasses and Blueprints.

**Time settings.** 2.0 adds `DefaultBulletTimeLimits` as the per-activation stage template. A nonempty legacy `BulletTimeLimits` array is copied into that template at BeginPlay; the runtime array is still consumed as stages advance. Consolidate conflicting old values. An empty stage array does not mean infinite lifetime.

**Speed and units.** In the public component, `MaxSpeed = 0` leaves acceleration uncapped, `MaxMoveDistance = 0` leaves outbound distance unlimited, and `ReturnRange` is a radius in cm. Acceleration defaults to per-second integration (`bScaleByDelta = true`), rather than a fixed increment per update. Return still takes its speed from `MaxSpeed`, so set a positive value when using it. Do not apply project-specific subclass numerical conventions to the public component.

**Acceleration curves.** The published `AccleCurve` asset input remains. Assigning an asset selects it, with `DefaultAccle` as its multiplier. The old `bUseAccleCurve` flag alone does not select a valid curve, and turning it off does not ignore an assigned asset. Use `StopAcceleration` to disable acceleration. Choose `AccleType = Curve` for the new inline curve.

**Collision roots and ignores.** Make the actual collision shape the root, or satisfy the automatic-promotion rule. The bullet itself is ignored automatically; add the shooter to `IgnoreActors` yourself. Do not rely on children following movement to provide full sweeps.

**Events and stopping.** `StopBulletMovement` stops silently; use `BulletStop` for a stop-reason callback. Existing `OnBulletStop` and additional `OnBulletMovementStop` both broadcast for a reasoned stop. Do not bind the same damage/recycling action to both. `OnBulletRetun` means return started; reason `Return` means it ended. New enum values follow the original `Time/Distance/Speed/Return` entries.

**Pooling.** Activation resets shot state such as speed, time, and distance. It does not restore consumed reflection counts, hit-deceleration arrays, persistent forces, or your game's per-shot settings. Refill them as described in the [guide](README.en.md#trails-stopping-and-pooling). Trails are cleaned up with Actor destruction; do not depend on them surviving it.

**C++.** Acceleration, deceleration, and enums now have separate headers: `FBulletAcceleInfo.h`, `FBulletDeceleInfo.h`, `E_BulletAccleType.h`, `E_BulletRefractType.h`, and `E_BulletStopReason.h`. Some runtime status and internal flags are no longer exposed as writable Blueprint settings. Use the supported function entry points and review code that accesses internal fields directly. `TrailVFX` still takes a Niagara System; do not depend on its underlying pointer-wrapper type.

### Validation and performance scope

The exact engine versions used for build checks are 4.26.2, 4.27.2, 5.0.3, 5.1.1, 5.2.1, 5.3.2, 5.4.4, 5.5.4, 5.6.1, 5.7.4, and 5.8.3. Checks use each version's UBT/UHT and non-Unity plugin C++ compilation with `NoLink`. They do not cover every historical patch or establish that DLL linking, packaging, and gameplay have passed on every branch.

Current runtime observations cover ordinary Actor activation, stopping, unregistration/destruction, and a projectile integration on UE5.8. They do not establish runtime coverage for every target version, packaged configuration, or network mode. Treat cross-version build checks separately from scene execution. No universal cross-version Mass net-speedup or hitch-free result has been established; measure your actual game workload.

## 1.0

The initial public workflow adds BulletMovement to an Actor and configures motion through parameters. Features include speed, range, time stages, acceleration curves, hit deceleration, deflection, reflection, homing, return, trails, and spread/spawning helpers.
