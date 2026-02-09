# Unity ECS Best Practices

**Version 1.0.0**  
IAFahim  
February 2026

> **Note:**  
> This document is mainly for agents and LLMs to follow when maintaining,  
> generating, or refactoring Unity Entity Component System (DOTS) codebases. Humans  
> may also find it useful, but guidance here is optimized for automation  
> and consistency by AI-assisted workflows.

---

## Abstract

Elite Unity ECS (Entities) architecture with strict 4-layer design, prioritizing archetype optimization, Burst compilation, and Data-Oriented Design patterns.

---

## Table of Contents

1. [Section 1](#1-section-1) — **CRITICAL**
   - 1.1 [Follow 4-Layer Architecture](#11-follow-4-layer-architecture)
   - 1.2 [Follow Entity-Component-Responsibility Principle](#12-follow-entity-component-responsibility-principle)
2. [Section 2](#2-section-2) — **CRITICAL**
   - 2.1 [Optimize Archetype for Chunk Iteration](#21-optimize-archetype-for-chunk-iteration)
3. [Section 3](#3-section-3) — **CRITICAL**
   - 3.1 [Batch Structural Changes with EntityCommandBuffer](#31-batch-structural-changes-with-entitycommandbuffer)
4. [Section 4](#4-section-4) — **HIGH**
   - 4.1 [Define Constants in Logic Classes](#41-define-constants-in-logic-classes)
5. [Section 5](#5-section-5) — **CRITICAL**
   - 5.1 [Use RefRW/RefRO instead of IAspect](#51-use-refrwrefro-instead-of-iaspect)
   - 5.2 [Use Unity.Mathematics Types Exclusively](#52-use-unitymathematics-types-exclusively)
6. [Section 6](#6-section-6) — **HIGH**
   - 6.1 [Use System Ordering Attributes](#61-use-system-ordering-attributes)
7. [Section 7](#7-section-7) — **HIGH**
   - 7.1 [Use IJobEntity over Entities.ForEach](#71-use-ijobentity-over-entitiesforeach)
8. [Section 8](#8-section-8) — **HIGH**
   - 8.1 [Use IEnableableComponent for State Toggling](#81-use-ienableablecomponent-for-state-toggling)
9. [Section 9](#9-section-9) — **MEDIUM**
   - 9.1 [Use Side Comments Only, No Block Comments](#91-use-side-comments-only-no-block-comments)

---

## 1. Section 1

**Impact: CRITICAL**

### 1.1 Follow 4-Layer Architecture

**Impact: CRITICAL (Essential for maintainable DOTS codebases)**

1. **Data Layer**: `IComponentData` structs - pure data, no logic

2. **Logic Layer**: Static utility methods or `IJobEntity` jobs - reusable algorithms

3. **System Layer**: `ISystem` implementations - orchestration and scheduling

4. **Presentation Layer**: MonoBehaviour wrappers - bridge to Unity GameObjects

**Incorrect:**

```csharp
// BAD: Logic mixed into component
public struct Health : IComponentData
{
    public float Value;
    public float Max;

    public void TakeDamage(float damage) // Logic in data!
    {
        Value = math.max(0, Value - damage);
    }
}

// BAD: System doing logic that should be in Logic layer
[BurstCompile]
public partial struct HealthSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (health, damage) in SystemAPI.Query<RefRW<Health>, RefRO<Damage>>())
        {
            health.ValueRW.Value = math.max(0, health.ValueRO.Value - damage.ValueRO.Amount); // Logic here!
        }
    }
}
```

**Correct:**

```csharp
// GOOD: Pure data component
public struct Health : IComponentData
{
    public float Value;
    public float Max;
}

// GOOD: Logic layer - static utility
public static class HealthLogic
{
    [BurstCompile]
    public static float ApplyDamage(float current, float damage)
    {
        return math.max(0, current - damage);
    }

    [BurstCompile]
    public static float GetHealthPercent(float current, float max)
    {
        return current / max;
    }
}

// GOOD: System layer - orchestration only
[BurstCompile]
public partial struct HealthSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var damageJob = new DamageJob
        {
            Damages = SystemAPI.GetComponentTypeHandle<Damage>(true),
            Healths = SystemAPI.GetComponentTypeHandle<Health>(false)
        };
        state.Dependency = damageJob.ScheduleParallel(state.Dependency);
    }
}

[BurstCompile]
partial struct DamageJob : IJobEntity
{
    [ReadOnly] public ComponentTypeHandle<Damage> Damages;
    public ComponentTypeHandle<Health> Healths;

    void Execute(in Damage damage, ref Health health)
    {
        health.Value = HealthLogic.ApplyDamage(health.Value, damage.Amount); // Calls Logic layer
    }
}
```

**Presentation Layer Example:**

```csharp
// GOOD: Presentation layer - MonoBehaviour bridge
public class HealthBarPresenter : MonoBehaviour
{
    public EntityManager EntityManager;
    private Entity _entity;

    void Update()
    {
        if (EntityManager.HasComponent<Health>(_entity))
        {
            var health = EntityManager.GetComponentData<Health>(_entity);
            UpdateUI(health.Value, health.Max);
        }
    }

    void UpdateUI(float current, float max)
    {
        fillAmount = current / max; // UI code in MonoBehaviour
    }
}
```

### 1.2 Follow Entity-Component-Responsibility Principle

**Impact: CRITICAL (Prevents bloated components and systems)**

Each Entity should have a single responsibility. Components should be small and focused.

ECS naturally encourages single responsibility through composition. Don't create "God Components" that do too much.

**Incorrect:**

```csharp
// BAD: God component - too many responsibilities
public struct Character : IComponentData
{
    public float3 Position;
    public float3 Velocity;
    public float Health;
    public float MaxHealth;
    public float Mana;
    public float MaxMana;
    public float Stamina;
    public float MaxStamina;
    public int Level;
    public float Experience;
    public float AttackPower;
    public float DefensePower;
    public float Speed;
    public bool IsDead;
    public bool IsBlocking;
    public bool IsAttacking;
    public int Gold;
    public float InventoryWeight;
}
```

**Correct:**

```csharp
// GOOD: Entity is composed of focused components
// A "Player" entity might have:
// - LocalTransform
// - Velocity
// - Health
// - Mana
// - CharacterStats
// - MovementState
// - PlayerInput (unique to players)
// - Inventory

// A "Enemy" entity might have:
// - LocalTransform
// - Velocity
// - Health
// - CharacterStats
// - MovementState
// - AIController (unique to enemies)
```

**Correct: Entity Composition**

---

## 2. Section 2

**Impact: CRITICAL**

### 2.1 Optimize Archetype for Chunk Iteration

**Impact: CRITICAL (10-100× performance improvement)**

Design archetypes so entities with the same components are stored together in chunks.

ECS stores entities in chunks of 16KB. Chunk iteration is fastest when all entities in a chunk have the same component types.

**Incorrect:**

```csharp
// BAD: Random component combinations create many archetype variations
public struct RenderFlag : IComponentData { public bool ShouldRender; }
public struct PhysicsFlag : IComponentData { public bool HasPhysics; }
public struct AIFlag : IComponentData { public bool HasAI; }

// Each unique combination creates a new archetype
// Entity A: RenderFlag, PhysicsFlag
// Entity B: RenderFlag, AIFlag
// Entity C: PhysicsFlag, AIFlag
// Entity D: RenderFlag, PhysicsFlag, AIFlag
// Result: 4 different archetypes, poor iteration
```

**Correct:**

```csharp
// GOOD: Group related components together
public struct Moveable : IComponentData { }
public struct Static : IComponentData { }

// Movement components only on Moveable entities
public struct Velocity : IComponentData { public float3 Value; }
public struct Acceleration : IComponentData { public float3 Value; }

// Physics components only on entities that need physics
public struct PhysicsBody : IComponentData { public float Mass; }
public struct PhysicsMaterial : IComponentData { public float Friction; }

// Clean archetype separation:
// Moveable entities: LocalTransform, Moveable, Velocity, Acceleration
// Static entities: LocalTransform, Static
// Physics entities: LocalTransform, Moveable, Velocity, PhysicsBody, PhysicsMaterial
```

**Correct: Archetype Design Pattern**

---

## 3. Section 3

**Impact: CRITICAL**

### 3.1 Batch Structural Changes with EntityCommandBuffer

**Impact: CRITICAL (100× faster than per-entity changes)**

Use `EntityCommandBuffer` to batch structural changes (add/remove components, create/destroy entities).

Each structural change synchronizes the EntityManager. Batching changes reduces sync points from O(n) to O(1).

**Incorrect:**

```csharp
// BAD: Structural change per entity
[BurstCompile]
public partial struct SpawnSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var prefab = SystemAPI.GetSingleton<EnemyPrefab>();

        for (int i = 0; i < 1000; i++)
        {
            var entity = state.EntityManager.Instantiate(prefab); // Sync per iteration!
            state.EntityManager.SetComponentData(entity, new SpawnPosition { Value = GetSpawnPoint(i) });
        }
    }
}
```

**Correct:**

```csharp
// GOOD: Batch multiple operations on same entity
[BurstCompile]
public partial struct DeathSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.TempJob);

        foreach (var (entity, health) in SystemAPI.Query<EntityRef, RefRO<Health>>())
        {
            if (health.ValueRO.Value <= 0)
            {
                ecb.RemoveComponent<Health>(entity);
                ecb.RemoveComponent<Velocity>(entity);
                ecb.AddComponent<Dead>(entity);
                ecb.SetComponent(entity, new DeathTime { Value = (float)SystemAPI.Time.ElapsedTime });
            }
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}
```

**Best: EntityCommandBufferSystem**

**Correct: Multiple Component Operations**

---

## 4. Section 4

**Impact: HIGH**

### 4.1 Define Constants in Logic Classes

**Impact: HIGH (Improves maintainability and Burst optimization)**

Define `public const` values in Logic classes instead of magic numbers.

Magic numbers are hard to maintain. Constants in Logic classes are Burst-compileable and centralized.

**Incorrect:**

```csharp
// BAD: Magic numbers scattered everywhere
[BurstCompile]
public partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (transform, velocity) in SystemAPI.Query<RefRW<LocalTransform>, RefRW<Velocity>>())
        {
            // Magic numbers!
            velocity.ValueRW.Value *= 0.95f;
            velocity.ValueRW.Value = math.clamp(velocity.ValueRW.Value, -50.0f, 50.0f);
            transform.ValueRW.Position.y = math.max(transform.ValueRW.Position.y, 0.1f);
        }
    }
}
```

**Correct:**

```csharp
// BETTER: Constants grouped by system
public static class GameConstants
{
    public static class Movement
    {
        public const float FRICTION = 0.95f;
        public const float MAX_VELOCITY = 50.0f;
        public const float ACCELERATION = 100.0f;
    }

    public static class Combat
    {
        public const float MELEE_RANGE = 2.0f;
        public const float MELEE_DAMAGE = 25.0f;
        public const float ATTACK_COOLDOWN = 0.5f;
    }

    public static class Physics
    {
        public const float GRAVITY = -9.81f;
        public const float GROUND_SNAP_DISTANCE = 0.1f;
        public const int COLLISION_LAYERS = 32;
    }
}
```

**Best: Constants per System**

---

## 5. Section 5

**Impact: CRITICAL**

### 5.1 Use RefRW/RefRO instead of IAspect

**Impact: HIGH (Direct access avoids abstraction overhead)**

Use `RefRW<T>` and `RefRO<T>` directly instead of creating `IAspect` implementations.

`IAspect` adds unnecessary abstraction. `RefRW`/`RefRO` provide direct component access with explicit read/write semantics.

**Incorrect:**

```csharp
// BAD: IAspect abstraction
public readonly partial struct CharacterAspect : IAspect
{
    public readonly RefRW<Health> Health;
    public readonly RefRW<Mana> Mana;
    public readonly RefRO<CharacterStats> Stats;

    public void TakeDamage(float damage)
    {
        Health.ValueRW.Value -= damage;
    }

    public void CastMana(float cost)
    {
        Mana.ValueRW.Value -= cost;
    }
}

[BurstCompile]
public partial struct CombatSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var character in SystemAPI.Query<CharacterAspect>())
        {
            character.TakeDamage(10);
            character.CastMana(5);
        }
    }
}
```

**Correct:**

```csharp
// GOOD: Direct RefRW/RefRO access
[BurstCompile]
public partial struct CombatSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (health, mana, stats) in SystemAPI.Query<RefRW<Health>, RefRW<Mana>, RefRO<CharacterStats>>())
        {
            float damage = CombatLogic.CalculateDamage(stats.ValueRO);
            health.ValueRW.Value = HealthLogic.ApplyDamage(health.ValueRO.Value, damage);

            float manaCost = 5.0f;
            mana.ValueRW.Value = ManaLogic.ConsumeMana(mana.ValueRO.Value, manaCost);
        }
    }
}

// GOOD: Logic layer for reusable operations
public static class HealthLogic
{
    [BurstCompile]
    public static float ApplyDamage(float current, float damage)
    {
        return math.max(0, current - damage);
    }
}

public static class ManaLogic
{
    [BurstCompile]
    public static float ConsumeMana(float current, float cost)
    {
        return math.max(0, current - cost);
    }
}
```

### 5.2 Use Unity.Mathematics Types Exclusively

**Impact: CRITICAL (Required for Burst SIMD optimization)**

Use `Unity.Mathematics` types (`float3`, `quaternion`, `float4x4`) instead of `UnityEngine` types.

`Unity.Mathematics` types are Burst-compatible and use SIMD instructions. `UnityEngine.Vector3` causes Burst errors.

**Incorrect:**

```csharp
// BAD: UnityEngine types cause Burst errors
using UnityEngine;

public struct Velocity : IComponentData
{
    public Vector3 Value; // Not Burst-compatible!
}

[BurstCompile] // Will fail to compile!
public partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, velocity) in SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * SystemAPI.Time.DeltaTime;
        }
    }
}
```

**Correct:**

```csharp
// GOOD: Unity.Mathematics types
using Unity.Mathematics;

public struct Velocity : IComponentData
{
    public float3 Value;
}

[BurstCompile]
public partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (transform, velocity) in SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * dt;
        }
    }
}
```

**Common Type Conversions:**

```csharp
// GOOD: Use math library for operations
using static Unity.Mathematics.math;

// Vector operations
float3 direction = normalize(target - origin);
float distance = length(target - origin);
float3 rotated = rotateY(forward, angle);

// Quaternion operations
quaternion rotation = quaternion.AxisAngle(up, angle);
float3 rotatedVector = rotate(rotation, forward);

// Interpolation
float3 lerpPosition = lerp(from, to, t);
quaternion slerpRotation = slerp(fromRotation, toRotation, t);

// Math functions
float clamped = clamp(value, min, max);
float eased = smoothstep(0, 1, t);
```

**Conversion Between Types:**

```csharp
// Convert Unity.Mathematics to UnityEngine (for non-Burst code)
float3 mathPos;
UnityEngine.Vector3 unityPos = mathPos;

// Convert UnityEngine to Unity.Mathematics
UnityEngine.Vector3 unityPos;
float3 mathPos = unityPos;
```

---

## 6. Section 6

**Impact: HIGH**

### 6.1 Use System Ordering Attributes

**Impact: HIGH (Prevents race conditions and sync points)**

Use `[UpdateBefore]`, `[UpdateAfter]`, and `[UpdateInGroup]` to control system execution order.

Systems run in parallel by default. Without explicit ordering, systems reading/writing the same data cause race conditions or unnecessary sync points.

**Incorrect:**

```csharp
// BAD: No ordering specified - race condition!
[BurstCompile]
public partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, velocity) in SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * SystemAPI.Time.DeltaTime;
        }
    }
}

[BurstCompile]
public partial struct CollisionSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, collision) in SystemAPI.Query<RefRW<LocalTransform>, RefRW<CollisionResponse>>())
        {
            transform.ValueRW.Position += collision.ValueRO.Pushback;
        }
    }
}
// Both systems write to LocalTransform - who runs first?
```

**Correct:**

```csharp
// GOOD: Explicit ordering
[UpdateBefore(typeof(CollisionSystem))]
[BurstCompile]
public partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, velocity) in SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * SystemAPI.Time.DeltaTime;
        }
    }
}

[UpdateAfter(typeof(MovementSystem))]
[BurstCompile]
public partial struct CollisionSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (transform, collision) in SystemAPI.Query<RefRW<LocalTransform>, RefRW<CollisionResponse>>())
        {
            transform.ValueRW.Position += collision.ValueRO.Pushback;
        }
    }
}
```

**System Groups:**

```csharp
// GOOD: Use groups for organization
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
[UpdateBefore(typeof(BuildPhysicsWorld))]
public partial struct PrePhysicsSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Runs before physics at fixed timestep
    }
}

[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
[UpdateAfter(typeof(PhysicsSystemGroup))]
public partial struct PostPhysicsSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Runs after physics at fixed timestep
    }
}

[UpdateInGroup(typeof(PresentationSystemGroup))]
public partial struct RenderUpdateSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Runs during presentation phase
    }
}
```

**Common System Groups:**

```csharp
// InitializationSystemGroup - Runs once at startup
// SimulationSystemGroup - Default variable timestep systems
// FixedStepSimulationSystemGroup - Fixed timestep (physics, networking)
// PresentationSystemGroup - UI, rendering updates
```

---

## 7. Section 7

**Impact: HIGH**

### 7.1 Use IJobEntity over Entities.ForEach

**Impact: HIGH (Modern pattern with better Burst compatibility)**

Use `IJobEntity` instead of `Entities.ForEach` for job-based iteration.

`IJobEntity` is the modern, type-safe job pattern. `Entities.ForEach` is legacy and less flexible.

**Incorrect:**

```csharp
// BAD: Legacy Entities.ForEach
[BurstCompile]
public partial struct MovementSystem : SystemBase
{
    protected override void OnUpdate()
    {
        float dt = Time.DeltaTime;

        Entities
            .ForEach((ref Translation translation, in Velocity velocity) =>
            {
               .Value += velocity.Value * dt;
            })
            .ScheduleParallel();
    }
}
```

**Correct:**

```csharp
// GOOD: Explicit read/write specification
[BurstCompile]
partial struct PhysicsJob : IJobEntity
{
    public float DeltaTime;
    public float Gravity;

    void Execute(
        RefRW<LocalTransform> transform,
        RefRW<Velocity> velocity,
        RefRO<Mass> mass,
        RefRO<GravityEnabled> gravityEnabled)
    {
        if (!gravityEnabled.ValueRO.Value) return;

        velocity.ValueRW.Value.y += Gravity * DeltaTime / mass.ValueRO.Value;
        transform.ValueRW.Position.xyz += velocity.ValueRO.Value * DeltaTime;
    }
}
```

**Best: IJobEntity with RefRW/RefRO**

---

## 8. Section 8

**Impact: HIGH**

### 8.1 Use IEnableableComponent for State Toggling

**Impact: HIGH (Avoids archetype changes, 50× faster)**

Use `IEnableableComponent` for toggling features instead of adding/removing components.

Adding/removing components causes structural changes that move entities between archetypes. `IEnableableComponent` toggles state without moving entities.

**Incorrect:**

```csharp
// BAD: Adding/removing components for state
public struct Stunned : IComponentData { }

[BurstCompile]
public partial struct StunSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.TempJob);

        foreach (var (entity, stunRequest) in SystemAPI.Query<EntityRef, RefRO<StunRequest>>())
        {
            ecb.AddComponent(entity, new Stunned()); // Structural change
        }

        foreach (var entity in SystemAPI.Query<EntityRef>().WithAll<Stunned>())
        {
            if (stunExpired)
            {
                ecb.RemoveComponent<Stunned>(entity); // Structural change
            }
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}
```

**Correct:**

```csharp
// GOOD: Use IEnableableComponent
public struct Stunned : IComponentData, IEnableableComponent { }

[BurstCompile]
public partial struct StunSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (stunned, stunRequest) in SystemAPI.Query<EnabledRefRW<Stunned>, RefRO<StunRequest>>())
        {
            stunned.Value = true; // No structural change
        }

        foreach (var (stunned, stunTimer) in SystemAPI.Query<EnabledRefRW<Stunned>, RefRW<StunTimer>>())
        {
            stunTimer.ValueRW.TimeLeft -= SystemAPI.Time.DeltaTime;

            if (stunTimer.ValueRO.TimeLeft <= 0)
            {
                stunned.Value = false; // No structural change
            }
        }
    }
}
```

**Querying Enabled Components:**

```csharp
// GOOD: Query only enabled components
foreach (var (transform, velocity) in SystemAPI.Query<RefRW<LocalTransform>, RefRW<Velocity>>()
    .WithAll<MovementEnabled>())
{
    // Only processes entities with MovementEnabled enabled
}

// GOOD: Query both enabled and disabled
foreach (var (transform, movementEnabled) in SystemAPI.Query<RefRW<LocalTransform>, EnabledRefRO<MovementEnabled>>())
{
    if (movementEnabled.Value)
    {
        // Movement logic
    }
}
```

---

## 9. Section 9

**Impact: MEDIUM**

### 9.1 Use Side Comments Only, No Block Comments

**Impact: MEDIUM (Follows user preference for cleaner code)**

Place comments on the same line as code. No `///` XML documentation or block comments.

Block comments interrupt code flow. Side comments keep code compact and readable.

**Incorrect:**

```csharp
// BAD: Block comments
/// <summary>
/// Represents the health component for entities.
/// Contains current and maximum health values.
/// </summary>
public struct Health : IComponentData
{
    public float Value;
    public float Max;
}

/*
 * BAD: Multi-line block comment
 * This system handles damage application
 * and health reduction for all entities.
 */
[BurstCompile]
public partial struct DamageSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Apply damage to all entities
        foreach (var (health, damage) in SystemAPI.Query<RefRW<Health>, RefRO<Damage>>())
        {
            // Reduce health by damage amount
            health.ValueRW.Value -= damage.ValueRO.Amount;
        }
    }
}
```

**Correct:**

```csharp
// GOOD: Code is clear without comments
public static class HealthLogic
{
    [BurstCompile]
    public static float ApplyDamage(float current, float damage)
    {
        return math.max(0, current - damage); // Clamp to zero
    }

    [BurstCompile]
    public static bool IsDead(float health)
    {
        return health <= 0;
    }
}
```

**Best: Self-Documenting Code**

---

## References

1. [https://docs.unity3d.com/Packages/com.unity.entities@latest](https://docs.unity3d.com/Packages/com.unity.entities@latest)
2. [https://docs.unity3d.com/Packages/com.unity.burst@latest](https://docs.unity3d.com/Packages/com.unity.burst@latest)
3. [https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity.html](https://docs.unity3d.com/Manual/BestPracticeUnderstandingPerformanceInUnity.html)
