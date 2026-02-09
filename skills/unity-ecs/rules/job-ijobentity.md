---
title: Use IJobEntity over Entities.ForEach
section: job
impact: HIGH
impactDescription: Modern pattern with better Burst compatibility
---

# Use IJobEntity over Entities.ForEach

Use `IJobEntity` instead of `Entities.ForEach` for job-based iteration.

**Impact: HIGH (Modern pattern with better Burst compatibility)**

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
// GOOD: Modern IJobEntity
[BurstCompile]
public partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        new MovementJob
        {
            DeltaTime = dt
        }.ScheduleParallel(state.Dependency);
    }
}

[BurstCompile]
partial struct MovementJob : IJobEntity
{
    public float DeltaTime;

    void Execute(ref LocalTransform transform, in Velocity velocity)
    {
        transform.Position += velocity.Value * DeltaTime;
    }
}
```

**Best: IJobEntity with RefRW/RefRO**

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
