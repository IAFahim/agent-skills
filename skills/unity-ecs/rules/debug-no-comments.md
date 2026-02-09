---
title: Use Side Comments Only, No Block Comments
section: debug
impact: MEDIUM
impactDescription: Follows user preference for cleaner code
---

# Use Side Comments Only, No Block Comments

Place comments on the same line as code. No `///` XML documentation or block comments.

**Impact: MEDIUM (Follows user preference for cleaner code)**

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
// GOOD: Side comments only
public struct Health : IComponentData
{
    public float Value; // Current health value
    public float Max; // Maximum possible health
}

[BurstCompile]
public partial struct DamageSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (health, damage) in SystemAPI.Query<RefRW<Health>, RefRO<Damage>>())
        {
            health.ValueRW.Value = math.max(0, health.ValueRO.Value - damage.ValueRO.Amount); // Apply damage
        }
    }
}
```

**Best: Self-Documenting Code**

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
