```markdown
# AxiomProtocol: Zero-GC Combat & Enemy AI Framework

A high-performance, deterministic top-down combat engine and flocking AI framework developed in Unity (Universal Render Pipeline / C#). Built from the ground up to achieve strict **0 B managed heap allocations** during runtime gameplay loops, sustained 120+ FPS combat, and sub-millisecond physics ticks under heavy entity saturation.

---

## Technical Highlights & Engine Architecture

- **True Zero-GC Architecture:** No runtime calls to `Instantiate()`, `Destroy()`, Linq, or boxing delegates. Custom `ComponentPool<T>` recycling for projectiles, floating combat text, visual telegraphs, and enemy units.
- **Non-Allocating Physics:** Complete migration from standard allocation-heavy queries to `Physics.OverlapSphereNonAlloc` buffers across targeting, hitbox checks, swarm separation, and spawner clearance tests.
- **Deterministic State Graph:** Fully decoupled `IState` state-machine abstraction decoupled from `MonoBehaviour` lifecycles, enabling pre-cached state transitions with zero allocation footprints.
- **Flocking & Swarm Steering:** Vectorized neighbor-repulsion steering with inverse-distance falloff integrated into enemy locomotion to prevent capsule stacking and clump artifacts.
- **Data-Driven Waves & Payloads:** Modular gameplay behavior decoupled into scriptable payloads (`AbilityActionSO`) and composite wave definitions (`WaveSO`) supporting concurrent multi-group spawns.
- **Stress-Tested Scalability:** Benchmarked at sustained **128.8 FPS** during concurrent 1,000-projectile batch bursts with **0.000 ms** GC collection overhead.

---

## Architecture Overview

```text
Assets/_AxiomProtocol/
├── Data/
│   ├── Actions/                         # ScriptableObject ability payloads (Damage, Knockback, VFX)
│   └── Waves/                           # Data-driven wave compositions (WaveSO)
├── Prefabs/
│   ├── Characters/
│   │   ├── Enemy_Melee_Chaser.prefab    # Runner archetype (Fast chase, short-windup melee)
│   │   └── Enemy_Ranged_Sentinel.prefab # Sentinel archetype (Kiting, lateral strafe, ranged bolt)
│   ├── Combat/
│   │   ├── PR_EnergyBolt.prefab         # Recycled projectile entity
│   │   ├── UI_DamageCounter.prefab      # World-space pooled floating damage counter
│   │   └── VFX_ExpandingRing.prefab     # World-space telegraph / spawn ring effect
├── Scenes/
│   ├── Combat_MainArena.unity           # Primary encounter arena & wave loop
│   └── Combat_StressTest.unity          # 1,000-projectile profiler harness
└── Scripts/
    ├── Core/
    │   ├── Interfaces/
    │   │   ├── IPoolable.cs             # Contract for pooled object lifecycle
    │   │   └── IState.cs                # State machine interface (Enter, Update, FixedUpdate, Exit)
    │   └── Pooling/
    │       ├── ComponentPool.cs         # Generic pre-warmed stack pool
    │       └── PoolManager.cs           # Global centralized pool dispatcher
    └── Gameplay/
        ├── AI/
        │   ├── Core/
        │   │   ├── EnemyBrain.cs        # Central entity context & lifecycle controller
        │   │   └── EnemyStateMachine.cs # Reusable state controller
        │   ├── Interfaces/
        │   │   └── IEnemyContext.cs     # Interface isolating states from concrete enemy classes
        │   ├── Spawning/
        │   │   ├── WaveSO.cs            # Wave data structures & spawn group configurations
        │   │   └── WaveSpawner.cs       # Perimeter clearance spawner & pool consumer
        │   └── States/
        │       ├── EnemySpawnState.cs        # Invulnerable emergence & ground telegraph
        │       ├── EnemyChaseState.cs        # Pursuit with flocking repulsion steering
        │       ├── EnemyStrafeState.cs       # Ranged kiting & lateral orbital movement
        │       ├── EnemyAttackWindupState.cs # Melee windup, angle-lock & damage delivery
        │       ├── EnemyRangedAttackState.cs # Projectile launch with telegraph phase
        │       └── EnemyRecoveryState.cs     # Post-strike recovery & velocity clamp
        ├── Combat/
        │   ├── AbilityActionSO.cs       # Payload execution base
        │   ├── Action_Damage.cs         # Direct damage & UI number trigger
        │   └── Projectile.cs            # Linear raycast projectile with non-alloc impact
        └── Testing/
            └── CombatStressTester.cs    # Multi-projectile benchmark harness

```

---

## Core Systems & Implementation Details

### 1. Zero-GC Memory Management & Pooling

All dynamic runtime entities implement `IPoolable` and are managed through `PoolManager.Instance.Spawn<T>()` and `Despawn<T>()`.

* Pre-allocated internal stacks prevent pool array resizing during combat.
* `OnSpawnFromPool()` and `OnReturnToPool()` isolate object resets (clearing velocities, resetting state machines, wiping event delegates).
* Transforms and rigidbodies are initialized before activating renderers or visual effects to prevent single-frame origin flashes `(0, 0, 0)`.

```csharp
public interface IPoolable
{
    void OnSpawnFromPool();
    void OnReturnToPool();
    void SetPoolReleaseCallback(System.Action releaseAction);
}

```

### 2. Modular Enemy State Machine (`IState`)

All AI states are decoupled classes instantiated once inside `EnemyBrain.Awake()`. Transitions recycle the same class references without allocating memory on the heap.

```text
[EnemySpawnState] (Invulnerable, locks velocity, spawns telegraph ring)
       │
       ▼
[EnemyChaseState] / [EnemyStrafeState] (Flocking steering, checks attack range)
       │
       ▼
[EnemyAttackWindupState] / [EnemyRangedAttackState] (Locks velocity, telegraphs)
       │
       ▼
[EnemyRecoveryState] (End-lag phase, resets cooldowns)
       │
       └────────────────────────► Loops back to Chase / Strafe

```

### 3. Swarm Steering & Neighbor Repulsion

To prevent enemies from collapsing into single-point clusters during swarming, `EnemyChaseState` and `EnemyStrafeState` calculate dynamic repulsion forces using a pre-allocated non-allocating physics buffer:

$$\vec{R}_{\text{repulsion}} = \sum_{i=1}^{N} \frac{\vec{P}_{\text{self}} - \vec{P}_{\text{neighbor}, i}}{\text{dist}_i} \cdot \left(1 - \frac{\text{dist}_i}{R_{\text{separation}}}\right)$$

$$\vec{V}_{\text{steer}} = \text{Normalize}\left(\vec{D}_{\text{target}} + \vec{R}_{\text{repulsion}} \cdot W_{\text{separation}}\right)$$

### 4. Wave Spawner & Obstruction Clearance

The `WaveSpawner` dynamically monitors perimeter coordinates surrounding the target:

* Queries a circular spawn ring between `minSpawnRadius` and `maxSpawnRadius`.
* Employs an iterative clearance probe (`Physics.OverlapSphereNonAlloc`) validating whether candidate coordinates intersect colliders on the `ObstructionMask`.
* Forces rigidbodies and transforms to initialize ground elevations directly ($y = 1.0\text{m}$), eliminating vertical stacking artifacts on spawn.

---

## Enemy Archetypes

| Archetype | Class/Role | Primary Loop | Key Mechanics |
| --- | --- | --- | --- |
| **Runner** | Swarm Pressure | `Spawn` $\rightarrow$ `Chase` $\rightarrow$ `Windup` $\rightarrow$ `Recovery` | Fast speed (`4.5 m/s`), short telegraph (`0.65s`), tight swarm separation (`1.6m`). Forces player repositioning. |
| **Sentinel** | Zone Control / Sniper | `Spawn` $\rightarrow$ `Strafe` $\rightarrow$ `Ranged Attack` $\rightarrow$ `Recovery` | Kites at distance (`10.0m`), switches strafing directions every `2.5s`, fires pooled linear projectiles. |
| **Warden** *(Planned)* | Frontline Tank | `Spawn` $\rightarrow$ `Slow Advance` $\rightarrow$ `Slam Windup` $\rightarrow$ `Recovery` | Massive health pool, heavy knockback resistance, wide AoE damage radius with prolonged windup. |
| **Hunter** *(Planned)* | Flanker / Duelist | `Spawn` $\rightarrow$ `Pounce Dash` $\rightarrow$ `Combo Strike` $\rightarrow$ `Disengage` | Predictive targeting intercepting player dash vectors. |
| **Null Unit** *(Planned)* | Anti-Ability Support | `Spawn` $\rightarrow$ `Orbit Swarm` $\rightarrow$ `Pulse Aura` | Non-allocating suppression aura disabling active player combat abilities within range. |

---

## Profiling & Performance Metrics

Stress tests executed on `Combat_StressTest.unity` targeting 1,000 simultaneous active projectiles interacting with dummy targets:

* **Target Framerate:** 128.8 FPS (Sustained)
* **Managed GC Allocations:** `0 B` per frame during active spawn/flight/impact cycles
* **Garbage Collection Pauses:** `0.000 ms`
* **Physics Overhead:** Continuous query handling using zero-allocation arrays (`Collider[N]`)

```text
+-------------------------------------------------------------------+
| Metric                             | Measurement                  |
+-------------------------------------------------------------------+
| Batch Fire Size                    | 1,000 Projectiles / burst    |
| Frame Time                         | ~7.7 ms                      |
| GC Alloc (FireBatch)               | 0 B                          |
| GC Alloc (PoolManager.Spawn)       | 0 B                          |
| GC Alloc (Physics.OverlapSphere)   | 0 B                          |
+-------------------------------------------------------------------+

```

---

## Setup & Testing Instructions

1. **Clone the Repository:**
```bash
git clone [https://github.com/your-username/AxiomProtocol-CombatFramework.git](https://github.com/your-username/AxiomProtocol-CombatFramework.git)

```


2. **Open in Unity:** Open using Unity 2022.3 LTS or newer (Universal Render Pipeline configured).
3. **Verify Physics Layers:**
* Layer 8: `Enemy`
* Layer 9: `Player`
* Layer 10: `Projectile`


4. **Run Stress Test Scene:**
* Open `Assets/_AxiomProtocol/Scenes/Combat_StressTest.unity`.
* Open the **Profiler** (`Ctrl + 7`).
* Enter **Play Mode** and press **Space** to fire a 1,000-projectile radial burst. Observe the **GC Alloc** row in the CPU Usage profiler module.


5. **Run Wave Combat Scene:**
* Open `Assets/_AxiomProtocol/Scenes/Combat_MainArena.unity`.
* Enter **Play Mode** to test the `WaveSpawner` automatically pulling Runners and Sentinels from pre-warmed pools.



```

```
