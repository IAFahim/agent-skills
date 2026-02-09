---
title: Use IEnableableComponent for State Toggling
section: component
impact: HIGH
impactDescription: Avoids archetype changes, 50× faster
---

# Use IEnableableComponent for State Toggling

Use `IEnableableComponent` for toggling features instead of adding/removing components.

**Impact: HIGH (Avoids archetype changes, 50× faster)**

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
