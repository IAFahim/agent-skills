---
title: Follow 4-Layer Architecture
section: arch
impact: CRITICAL
impactDescription: Essential for maintainable DOTS codebases
---

# Follow 4-Layer Architecture

ECS systems must follow the 4-layer architecture: **Data → Logic → System → Presentation**.

**Impact: CRITICAL (Essential for maintainable DOTS codebases)**

The 4-layer architecture separates concerns and prevents coupling between systems.

## The 4 Layers

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
