# BulletMovement Changes and Migration

[User guide](README.en.md) · [简体中文](CHANGELOG.zh-CN.md)

## 2.0

BulletMovement 2.0 keeps the existing way of building a projectile: add the component to an ordinary Actor and configure its movement. On UE5, the component now manages Mass scheduling automatically. On UE4, it runs the shared motion kernel from component Tick.

### What's changed

- Ordinary Actors can use the component without a special base class, manager Actor, or Mass Trait asset. On UE5, activation, stopping, unregistration, and destruction are integrated with the Mass entity lifecycle.
- UE5 gathers Actor and component input on the game thread, computes numerical motion in Mass, and applies scene movement, collision, and events on the game thread. UE4 uses the same motion calculations without Mass.
- UE5.0–5.3 process chunks sequentially in the Mass motion processor's worker task. UE5.4 and later use parallel chunk queries. UE5.8 also supports scene publication and batched queries through the corresponding Chaos interfaces.
- The public class remains `UBulletMovementComponent`, along with common Blueprint functions and events, including the compatibility spelling `OnBulletRetun`. There are additional explicit stop reasons and an `OnBulletMovementStop` event.
- Changes to `IgnoreActors` are synchronized with movement sweeps, preserving native movement-ignore entries that were already present. The component can also promote a single query-enabled collision shape beneath an untouched `DefaultSceneRoot`.
- Homing, return, reflection, acceleration, hit deceleration, time and distance limits, and Niagara trails remain available through the component workflow.
- A pooled shot no longer retains the previous shot's current reflection target. Each activation clears `CurrentRefTarget`, so a refilled reflection chain can start at its first target.
- Fixed plugin compilation issues: added the required Blueprint categories, renamed conflicting UE5.1 delegate parameters, and explicitly included the Mass entity handle header on UE5.6 and UE5.7.

The plugin descriptor uses `VersionName = 2.0`. The release targets Win64 and provides source packages for UE 4.26, 4.27, and 5.0–5.8, covering 11 engine branches. Demo maps and example assets are not included. The [validation scope below](#validation-and-performance-scope) explains the compilation and runtime coverage.

### Upgrading from 1.0

Keep a recoverable copy of your project before upgrading. Install the package for your engine, rebuild, and open and recompile existing bullet Blueprints and C++ subclasses. Check their settings and pins. Keeping the class name does not mean every C++ reference or Blueprint pin is unchanged.

**Time settings.** Use `DefaultBulletTimeLimits` as the template for each activation. BeginPlay copies a nonempty legacy `BulletTimeLimits` array into that template, and activation then copies the template into the runtime array. The runtime entries are consumed as stages advance. If old content contains values in both arrays, consolidate them so the legacy array does not overwrite your intended settings. An empty stage array stops at the movement check; it does not mean unlimited lifetime.

**Speed and units.** In the public component, `MaxSpeed = 0` leaves acceleration uncapped, `MaxMoveDistance = 0` gives unlimited distance before return, and `ReturnRange` is a radius in cm. Acceleration now defaults to integration over elapsed time with `bScaleByDelta = true`, instead of a fixed speed increment per update. `SetupReturn` sets `CurrentSpeed` to `MaxSpeed`, so it needs a positive value; zero leaves the return leg stationary. Use these public-component conventions when updating your content; project-specific subclasses may use different numerical conventions.

**Acceleration curves.** The `AccleCurve` asset input is still supported. An assigned asset takes priority, and its Y value is multiplied by `DefaultAccle`. That multiplier defaults to 0, so assigning a curve alone produces no acceleration. The legacy `bUseAccleCurve` flag alone does not select a curve, and setting it to false does not bypass an assigned asset. Call `StopAcceleration()` to pause acceleration. To use the new inline curve, select `AccleType = Curve` and edit `RuntimeAccleCurve`.

**Collision roots and ignores.** Make the actual collision shape the root, or use the single-shape layout described in the [collision guide](README.en.md#collision-and-ignored-actors) so the component can promote it automatically. Child shapes following the root do not each receive a full movement sweep. The bullet ignores itself automatically; add the shooter to `IgnoreActors` yourself.

**Events and stopping.** `StopBulletMovement()` stops without broadcasting a stop reason. Use `BulletStop(Reason)` when you need a callback: it broadcasts `OnBulletMovementStop`, then `OnBulletStop`, and then stops movement. Bind a particular damage or recycling action to one of those events. `OnBulletRetun` means return has started; a stop with reason `Return` means it has ended. New stop-enum values follow the original `Time`, `Distance`, `Speed`, and `Return` entries. The compatibility spellings `Hited`, `Dameged`, and `OnBulletRetun` are retained.

On UE4.26–UE5.1, the affected event output pins are named `RefractVelocity`, `DeceleSpeed`, `ReturnTargetMesh`, and `ReturnTargetSocket`. On UE5.2 and later, they are named `Velocity`, `CurrentSpeed`, `ReturnMesh`, and `ReturnSocket`, respectively. Check event connections when switching engine branches.

**Pooling.** Activation resets shot state such as speed, remaining distance, time stages, and return state. It also clears `CurrentRefTarget`. It does not refill consumed reflection counts or hit-deceleration arrays, clear persistent forces, or restore your per-shot settings. Prepare these before reactivation using the [pooling sequence](README.en.md#trails-stopping-and-pooling). The deceleration-stage counter does not reset automatically, so use your own per-shot counter if needed. Trail components are cleaned up when the Actor is destroyed; they do not persist independently afterward.

**C++ headers and property access.** Acceleration, deceleration, and the enums now have separate headers: `FBulletAcceleInfo.h`, `FBulletDeceleInfo.h`, `E_BulletAccleType.h`, `E_BulletRefractType.h`, and `E_BulletStopReason.h`. Some runtime status fields and internal flags are no longer writable Blueprint settings. Review code that accesses internals directly and use the supported component functions. `TrailVFX` still takes a Niagara System; avoid depending on the pointer-wrapper type used internally.

### Validation and performance scope

Compilation checks used the following engine patch versions: 4.26.2, 4.27.2, 5.0.3, 5.1.1, 5.2.1, 5.3.2, 5.4.4, 5.5.4, 5.6.1, 5.7.4, and 5.8.3. Each corresponding source package passed that engine's UBT, UHT with `EngineRuntime` plugin classification, and non-Unity plugin C++ object compilation for Win64 Game Development using `NoLink`.

These checks exercise Unreal's build and reflected-header tooling and compile the plugin's C++ objects. They do not complete the default `BuildPlugin` builds for Editor, Game Development, and Game Shipping, or establish that plugin DLL linking succeeded on all 11 versions. They also do not establish packaging or gameplay coverage, or compatibility with every historical patch in an engine branch.

Runtime observations are limited to UE5.8: ordinary Actor activation and stopping, component unregistration and Actor destruction, and a projectile integration. This does not establish runtime coverage for every target engine, packaged configuration, or network mode.

Mass integration changes how motion work is scheduled. An overall speedup or hitch-free result has not been established across engine versions. Actors, collision, events, Niagara, and project logic still contribute to frame time; measure the complete frame with your actual projectile workload.

## 1.0

The initial release introduced the Actor-component workflow: add BulletMovement and configure projectile motion through parameters. It included speed and range limits, time stages, acceleration curves, hit deceleration, deflection, normal and targeted reflection, homing, return, Niagara trails, and spread and spawning helpers.
