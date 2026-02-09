---
title: Use System Ordering Attributes
section: system
impact: HIGH
impactDescription: Prevents race conditions and sync points
---

# Use System Ordering Attributes

Use `[UpdateBefore]`, `[UpdateAfter]`, and `[UpdateInGroup]` to control system execution order.

**Impact: HIGH (Prevents race conditions and sync points)**

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
