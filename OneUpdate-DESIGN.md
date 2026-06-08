# OneUpdate — Implementation Design

This document defines a lightweight Unity library that centralizes update callbacks through `PlayerLoop`. It is self-contained and intended as implementation context for an coding agent.

Code identifiers are in English. Explanations are in Vietnamese. The name `OneUpdate` is temporary.

---

## 1. Goals And Scope

OneUpdate centralizes update logic into a static PlayerLoop-driven core so gameplay systems can be paused, slowed, resumed, and maintained by group instead of being scattered across many `MonoBehaviour.Update` methods.

The primary value is **time orchestration by group**, not raw performance. Non-fixed phases are independent from global `Time.timeScale` because OneUpdate passes unscaled delta time multiplied by each group's `TimeScale`. FixedUpdate has an important limitation: Unity still controls whether the fixed phase runs and how often it runs; see section 9.

Design goals:

- Lightweight and reusable.
- Interface-based, not tied to `MonoBehaviour`.
- Group-level pause and time scale.
- Static PlayerLoop manager, no hidden GameObject and no `DontDestroyOnLoad`.
- Mechanism only, not policy: no base class in the core, no ScriptableObject group system, no timeScale modifier stack.

---

## 2. Core Architecture

The library keeps two intentionally different state layers:

### Registry

The Registry is the synchronous source of truth for registration state:

```csharp
Dictionary<object, RegistrationInfo> _registry;
```

`RegistrationInfo` stores:

- `UpdateGroup Group`
- phase mask
- target order
- editor-only cached type name

`Register` and `Unregister` update the Registry immediately. All questions like "is this target registered?" or "which group owns this target?" are answered from the Registry.

### PhaseList Ops Log

Each group owns one `PhaseList` per update phase. A `PhaseList` stores real tick entries plus an ordered mutation log:

```csharp
List<Entry> _entries;
List<PendingOp> _pending;
```

`PendingOp` is drained at the beginning of each `PhaseList.Run`. This makes mutation during iteration safe while preserving operation order.

The two-layer split is required:

- If Registry were delayed, double-register checks and unregister lookup would be unreliable.
- If PhaseList mutated synchronously during iteration, callback traversal would be unsafe.

---

## 3. Public API

```csharp
namespace OneUpdate
{
    public interface IInitializationUpdatable { void OnInitializationUpdate(float dt); }
    public interface IEarlyUpdatable          { void OnEarlyUpdate(float dt); }
    public interface IFixedUpdatable          { void OnFixedUpdate(float dt); }
    public interface IPreUpdatable            { void OnPreUpdate(float dt); }
    public interface IUpdatable               { void OnUpdate(float dt); }
    public interface ILateUpdatable           { void OnLateUpdate(float dt); }
    public interface IPostLateUpdatable       { void OnPostLateUpdate(float dt); }

    public static class Updater
    {
        public static UpdateGroup Default { get; }

        // name is debug-only, not a key. Empty, whitespace, and duplicate names are allowed.
        // group order is ascending; tie-break is creation index.
        public static UpdateGroup CreateGroup(string name, int order = 0);

        // null group means Updater.Default.
        public static void Register(object target, UpdateGroup group = null, int order = 0);
        public static void Unregister(object target);
    }

    public sealed class UpdateGroup
    {
        public string Name { get; }
        public int Order { get; }

        // Default 1. Allows 0. Rejects negative, NaN, and Infinity.
        public float TimeScale { get; set; }

        public bool IsPaused { get; }

        public void Pause(object token);
        public void Resume(object token);
        public void ResumeAll();
    }
}
```

`IInitializationUpdatable` is intentionally verbose. Unity's `Initialization` PlayerLoop phase runs every loop; it is not a one-time initialization callback. Do not name this API `OnInit`.

`Updater.Default` must be created by the library during static initialization and must not depend on lazy group creation during a tick.

---

## 4. Expected Usage

```csharp
public static class Groups
{
    public static readonly UpdateGroup Gameplay = Updater.CreateGroup("Gameplay", order: 0);
    public static readonly UpdateGroup UI = Updater.CreateGroup("UI", order: 10);
    public static readonly UpdateGroup Cutscene = Updater.CreateGroup("Cutscene", order: 20);
}

public sealed class Player : MonoBehaviour, IUpdatable, ILateUpdatable
{
    private void OnEnable() => Updater.Register(this, Groups.Gameplay);
    private void OnDisable() => Updater.Unregister(this);

    public void OnUpdate(float dt) { }
    public void OnLateUpdate(float dt) { }
}

public sealed class FactionAI : IUpdatable, IDisposable
{
    public FactionAI() => Updater.Register(this, Groups.Gameplay);
    public void OnUpdate(float dt) { }
    public void Dispose() => Updater.Unregister(this);
}
```

Pause tokens:

```csharp
Groups.Gameplay.Pause(cutscene);
Groups.Gameplay.Pause(menu);
Groups.Gameplay.Resume(menu);      // still paused by cutscene
Groups.Gameplay.Resume(cutscene);  // now active
```

Time scale:

```csharp
Groups.Gameplay.TimeScale = 0.3f;
Groups.UI.TimeScale = 1f;
```

---

## 5. Design Decisions

### 5.1 Register/Unregister Instead Of Handles

Registration is interface-based and does not require `MonoBehaviour`. Users call:

```csharp
Updater.Register(this, group);
Updater.Unregister(this);
```

The core does not return an `IDisposable` registration handle. This removes handle fields that users must store and dispose correctly. The cost is an internal Registry that maps target reference to registration info. This cost is acceptable because register/unregister is not a per-frame operation.

### 5.2 Seven Phase Interfaces

Each phase has its own interface. A target implements only the phases it needs. Interface introspection happens once during `Register`; no type checks are performed per frame.

Advanced or niche phases are:

- `Initialization`
- `EarlyUpdate`
- `PreUpdate`
- `PostLateUpdate`

Common phases are:

- `FixedUpdate`
- `Update`
- `LateUpdate` via Unity's `PreLateUpdate` phase

### 5.3 Groups

Groups are code-side handles created with `Updater.CreateGroup(name, order)`. The core does not define group enums or ScriptableObject assets.

Group execution order is:

1. ascending `Order`
2. ascending creation index when order ties

Implementation should keep a sorted group list or dirty sorted snapshot. Do not sort groups every phase every frame.

`CreateGroup` is valid during bootstrap, lazy initialization, and `Awake`, but not while OneUpdate is ticking. It is intended for static or bootstrap-level group definitions, not gameplay-instance churn.

### 5.4 Pause And TimeScale

Pause is composable. Each group owns a `HashSet<object>` of pause tokens using reference equality:

- `Pause(token)` adds a token.
- `Pause(sameToken)` is idempotent, not ref-counted.
- `Resume(token)` removes that token.
- `ResumeAll()` clears all tokens.
- `IsPaused` is true when token count is greater than zero.

TimeScale is intentionally not composable in the core. It is a scalar `float`, last-writer-wins. Composition policy for multiple sources of slow/boost is caller-owned.

`TimeScale` validation:

- `0` is allowed.
- negative values throw `ArgumentOutOfRangeException`.
- `NaN` throws `ArgumentOutOfRangeException`.
- `Infinity` throws `ArgumentOutOfRangeException`.

### 5.5 Single Active Registration Per Target

One target can have only one active registration:

- one group
- one order
- all implemented phases under that same group/order

To change group or order, call `Unregister(target)` first, then `Register(target, newGroup, newOrder)`.

Double-register is an Editor warning and runtime no-op. It must not mutate the existing registration.

### 5.6 Reference Equality

All identity-sensitive collections must use reference equality:

- Registry
- pause tokens
- PhaseList skip sets

Do not rely on `Equals` or `GetHashCode` from user objects. Targets may be records or business objects with value equality.

Use this internal comparer:

```csharp
internal sealed class ReferenceComparer : IEqualityComparer<object>
{
    public static readonly ReferenceComparer Instance = new ReferenceComparer();

    private ReferenceComparer() { }

    public bool Equals(object x, object y) => ReferenceEquals(x, y);

    public int GetHashCode(object obj) =>
        System.Runtime.CompilerServices.RuntimeHelpers.GetHashCode(obj);
}
```

### 5.7 No Core Base Class

The core exposes only interfaces, `Updater`, and `UpdateGroup`. A `ManagedBehaviour` helper can exist in samples or a separate optional assembly, but not in the core.

---

## 6. PhaseList Algorithm

### 6.1 Data Structures

```csharp
internal enum PendingOpType
{
    Add,
    Remove
}

internal readonly struct PendingOp
{
    public readonly PendingOpType Type;
    public readonly object Target;
    public readonly int Order;
    public readonly Action<float> Tick;
}

internal sealed class PhaseList
{
    private readonly List<Entry> _entries = new();
    private readonly List<PendingOp> _pending = new();
    private readonly HashSet<object> _skipSet = new(ReferenceComparer.Instance);

    private bool _isIterating;
    private bool _needsSort;
}
```

`Entry` stores target, callback, and order.

### 6.2 Run

```text
Run(dt, active):
    DrainPending()

    if _needsSort:
        entries.Sort(orderComparer)
        _needsSort = false

    if active:
        _isIterating = true
        try:
            for entry in entries:
                if _skipSet.Contains(entry.Target):
                    continue

                try:
                    entry.Tick(dt)
                catch Exception ex:
                    Debug.LogException(ex)
        finally:
            _isIterating = false

    _skipSet.Clear()
```

Structure maintenance must run even when the group is paused. A paused group drains pending ops and compacts removals, but does not tick callbacks.

### 6.3 DrainPending

```text
DrainPending():
    for op in _pending:
        if op.Type == Add:
            entries.Add(new Entry(op.Target, op.Tick, op.Order))
            _needsSort = true
        else:
            for i = entries.Count - 1 downto 0:
                if ReferenceEquals(entries[i].Target, op.Target):
                    entries.RemoveAt(i)
                    break

    _pending.Clear()
```

Remove scans from the end and breaks after the first match. The one-registration-per-target invariant guarantees at most one match per PhaseList. Do not replace this with `RemoveAll(lambda)`: it captures, can allocate, and does unnecessary work.

### 6.4 Enqueue

```text
EnqueueAdd(target, tick, order):
    _pending.Add(Add(target, tick, order))

EnqueueRemove(target):
    _pending.Add(Remove(target))
    if _isIterating:
        _skipSet.Add(target)
```

`EnqueueAdd` must not remove the target from `_skipSet`. If a target is unregistered then registered again during the same iteration, the old entry must still be skipped for the remainder of the current run. The new entry becomes visible when pending ops drain on the next run of that PhaseList.

### 6.5 Same-Frame Mutation Semantics

Pending ops are replayed in call order:

| Operation sequence | Pending log | Result on next run |
|---|---|---|
| `Register(A)` then `Unregister(A)` | `Add(A), Remove(A)` | A is absent |
| `Unregister(A)` then `Register(A, new)` | `Remove(A), Add(A,new)` | A is present with new attributes |
| `Unregister(A)` during iteration | `Remove(A)` plus skip set | A is skipped immediately, removed on next run |
| `Unregister(A)` then `Register(A,new)` during iteration | `Remove(A), Add(A,new)` plus skip set | old A skipped now, new A ticks next run |

Pending ops drain at the beginning of each specific `PhaseList.Run`, not at the beginning of the frame. A target registered into a group whose phase has not run yet can tick later in the same frame.

---

## 7. Updater Behavior

### 7.1 Register

```text
Register(target, group, order):
    if target == null:
        throw ArgumentNullException

    if target.GetType().IsValueType:
        throw ArgumentException

    if group == null:
        group = Default

    if registry.ContainsKey(target):
        editor warning with old group/phase info
        return

    introspect target interfaces once

    if phaseMask == 0:
        editor warning
        return

    registry.Add(target, info)

    for each implemented phase:
        group.Phase(phase).EnqueueAdd(target, callback, order)
```

When building callbacks, prefer typed casts stored at registration time so tick execution does not perform interface checks.

### 7.2 Unregister

```text
Unregister(target):
    if target == null:
        throw ArgumentNullException

    if target.GetType().IsValueType:
        throw ArgumentException

    if !registry.TryGetValue(target, out info):
        return

    for each phase in info.phaseMask:
        info.Group.Phase(phase).EnqueueRemove(target)

    registry.Remove(target)
```

Enqueue removes before removing from the Registry. This allows immediate `Unregister(A); Register(A, newGroup)` to pass the double-register guard and enqueue the correct ordered operations.

---

## 8. PlayerLoop Lifecycle

### 8.1 RuntimeInitializeOnLoadMethod Hooks

Use two hooks for two different responsibilities:

```csharp
[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
private static void ResetStateForPlaySession()
{
    // Clear registrations, pending ops, pause tokens, and per-session state.
    // Preserve UpdateGroup identity for domain reload off.
}

[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
private static void InstallPlayerLoop()
{
    // Idempotent install.
}
```

`SubsystemRegistration` resets play-session state. `BeforeSceneLoad` installs PlayerLoop entries.

### 8.2 Idempotent Install

```text
Install():
    loop = PlayerLoop.GetCurrentPlayerLoop()
    RemoveOurMarkers(ref loop)
    InsertOurMarkers(ref loop)
    PlayerLoop.SetPlayerLoop(loop)
```

Never restore a stored default PlayerLoop snapshot. Other packages may have inserted their own entries. Removing only OneUpdate markers avoids deleting UniTask, Entities, or other PlayerLoop users.

### 8.3 Marker Types

Use seven internal marker structs as `PlayerLoopSystem.type`:

```csharp
internal static class PlayerLoopMarkers
{
    public struct Initialization { }
    public struct EarlyUpdate { }
    public struct FixedUpdate { }
    public struct PreUpdate { }
    public struct Update { }
    public struct PreLateUpdate { }
    public struct PostLateUpdate { }
}
```

`RemoveOurMarkers` removes only systems whose type is one of these markers.

### 8.4 Editor Stop Cleanup

In the Editor, subscribe to `EditorApplication.playModeStateChanged`. On `ExitingPlayMode`, remove OneUpdate marker entries only. This is defense in depth; the next install is already idempotent.

### 8.5 Domain Reload Off

With domain reload disabled:

- Static group holders may run only once.
- `UpdateGroup` identity must survive Play/Stop.
- Registrations, pending ops, pause tokens, and per-session state must be cleared on each play session.
- PlayerLoop must never accumulate duplicate marker entries.

This is a required test gate.

### 8.6 Insert Position

Default insert policy is append to each target PlayerLoop phase:

- `Initialization`
- `EarlyUpdate`
- `FixedUpdate`
- `PreUpdate`
- `Update`
- `PreLateUpdate`
- `PostLateUpdate`

Append means after all existing subsystems in that phase. It does not mean "right after `MonoBehaviour.Update`." `MonoBehaviour.Update` is one subsystem inside the Update phase. User-facing documentation must state this clearly.

Future extension may support insertion before a specific subsystem.

### 8.7 Ticking Guard

`Updater` keeps:

```csharp
private static bool _isAnyPhaseIterating;
```

`RunPhase` sets it with `try/finally`:

```text
RunPhase(phase):
    _isAnyPhaseIterating = true
    try:
        for group in sorted groups:
            group.Run(phase, baseDt)
    finally:
        _isAnyPhaseIterating = false
```

`CreateGroup` throws `InvalidOperationException` when this flag is true.

---

## 9. Delta Time Semantics

### 9.1 Non-Fixed Phases

For non-fixed phases:

```csharp
dt = Time.unscaledDeltaTime * group.TimeScale;
```

Callbacks must use the provided `dt`. Reading `Time.deltaTime` bypasses group timeScale and breaks group-level orchestration.

### 9.2 FixedUpdate

For FixedUpdate:

```csharp
dt = Time.fixedUnscaledDeltaTime * group.TimeScale;
```

This controls only the `dt` passed to `IFixedUpdatable.OnFixedUpdate`.

Unity still controls whether the FixedUpdate phase runs and how many times it runs. When global `Time.timeScale == 0`, Unity does not advance the physics accumulator, so fixed callbacks do not run even if `group.TimeScale > 0`.

OneUpdate does not provide an independent unscaled fixed accumulator. If that is needed, it is a future feature outside the core.

### 9.3 Physics

OneUpdate controls callbacks, not Unity physics simulation. Pausing a group stops that group's fixed callbacks. It does not pause physics globally.

---

## 10. API Contract

### 10.1 Throw

These are API misuse and must fail fast.

`ArgumentNullException`:

- `Register(null, ...)`
- `Unregister(null)`
- `Pause(null)`
- `Resume(null)`
- `CreateGroup(null, ...)`

`ArgumentException`:

- `Register(valueTypeTarget, ...)`
- `Unregister(valueTypeTarget)`
- `Pause(valueTypeToken)`
- `Resume(valueTypeToken)`

Reason: value types box into new references and break reference identity. Message should clearly say reference identity is required and callers must pass a class instance.

`ArgumentOutOfRangeException`:

- `group.TimeScale < 0`
- `float.IsNaN(group.TimeScale)`
- `float.IsInfinity(group.TimeScale)`

`InvalidOperationException`:

- `CreateGroup(...)` while OneUpdate is ticking.

### 10.2 Editor Warning And Runtime No-Op

These are development-time mistakes. In builds, they are no-ops without diagnostics.

- `Register(target)` when target implements no OneUpdate phase interface.
- Double-register when target is already in the Registry.

Double-register must not mutate the old registration.

### 10.3 Silent No-Op

These are defensive cleanup operations and must not log:

- `Unregister(target)` when target is not registered.
- `Resume(token)` when token is not held.

### 10.4 Caller Responsibilities

These are part of the usage contract.

Stable reference identity:

- Targets and pause tokens must be stable references.
- Prefer `this`, a dedicated object field, or a static readonly sentinel.
- Avoid strings as tokens. String literals may appear stable because of interning, but constructed strings may not match.
- Value types are rejected by the API.

Group lifecycle:

- Create groups during bootstrap, lazy initialization, or `Awake`.
- Do not create groups as gameplay-instance churn.
- Runtime-created groups survive Play/Stop when domain reload is off because group identity is preserved.

TimeScale composition:

- The core stores one scalar value.
- If multiple systems influence slow/boost, caller owns modifier composition and writes the final scalar into `group.TimeScale`.

Use provided delta time:

- Callbacks must use the `dt` parameter.
- Do not use `Time.deltaTime` or `Time.fixedDeltaTime` inside OneUpdate callbacks unless intentionally bypassing group time.

Main thread only:

- All APIs must be called from Unity main thread.
- The library has no locks and is intentionally not thread-safe.
- Background-thread calls are undefined behavior.

Single active registration:

- One target has at most one active group/order.
- To change group/order, call `Unregister` then `Register`.
- Double-register is an Editor warning and no-op.

---

## 11. Editor Diagnostics

All diagnostics in this section are `#if UNITY_EDITOR` only.

`RegistrationInfo` should cache:

```csharp
string TypeName;
```

only in editor builds. Set it during `Register` using `target.GetType().FullName`.

Periodic registry scan in Editor:

- Run infrequently, for example through `EditorApplication.update` with throttling.
- For each registered target that is a destroyed `UnityEngine.Object`, log a leak warning.
- Detect destroyed Unity objects via lifted equality: `((UnityEngine.Object)target) == null`.

Log format:

```text
[OneUpdate] Leak: target was destroyed but not unregistered.
  Type: <typeName>, Group: <group.Name>, Phases: <phase list>, Order: <order>
```

Also log Editor warnings for:

- double-register
- target implements no phase interface

No diagnostics from this section should exist in player builds.

---

## 12. Exception Handling

Each callback invocation is isolated:

```text
try:
    entry.Tick(dt)
catch Exception ex:
    Debug.LogException(ex)
```

One throwing target must not stop later targets from ticking.

Advanced policies like throttled logs or auto-removing a repeatedly throwing target are outside the initial core.

---

## 13. Tests Required Before Core Is Considered Stable

### 13.1 API Contract Tests

Throw tests: 13 cases.

- null arguments: 5 cases
- value-type arguments: 4 cases
- invalid TimeScale: 3 cases
- CreateGroup while ticking: 1 case

Editor warning + no-op tests: 2 cases.

- target implements no phase interface
- double-register does not mutate state

Silent no-op tests: 2 cases.

- unregister unknown target
- resume unknown token

### 13.2 PhaseList Mutation Tests

Cover these four traces:

- `Register(A)` then `Unregister(A)` in the same frame: A does not tick.
- `Unregister(A)` then `Register(A,new)` in the same frame: A ticks with new registration.
- `Unregister(A)` during iteration: A is skipped immediately and removed next run.
- `Unregister(A)` then `Register(A,new)` during iteration: old A skipped now, new A ticks next run.

### 13.3 Pause And TimeScale Tests

- `Pause(sameToken)` is idempotent.
- `Resume(unknownToken)` is silent no-op.
- multiple pause tokens require all tokens to be resumed.
- `ResumeAll` clears all tokens.
- `TimeScale = 0` is valid.
- invalid TimeScale values throw.

### 13.4 PlayerLoop And Domain Reload Tests

- Install is idempotent: no duplicate markers.
- Removing OneUpdate markers does not remove foreign PlayerLoop systems.
- Domain reload off Play/Stop repeatedly:
  - group identity persists
  - registrations clear
  - pause tokens clear
  - pending ops clear
  - PlayerLoop markers do not accumulate

### 13.5 Coexistence Test

Install OneUpdate alongside another PlayerLoop-using package or a test marker system. Both systems must remain installed and run.

---

## 14. Out Of Scope

- Replacing coroutines or async systems.
- Physics collision callbacks.
- Thread safety.
- Multi-registration: one target in multiple groups at once.
- Custom phases beyond the seven public phases.
- DOTS/SoA optimization.
- Tokenized or stacked TimeScale modifiers.
- Independent unscaled fixed accumulator.

---

## 15. Packaging Notes

Recommended UPM layout:

```text
Packages/com.yourname.oneupdate/
  package.json
  Runtime/
    OneUpdate.asmdef
    ...
  Editor/
    OneUpdate.Editor.asmdef
    ...
  Tests/
    Runtime/
    Editor/
  Samples~/
```

Editor diagnostics and play-mode cleanup belong in an Editor assembly or `#if UNITY_EDITOR` guarded code.

