---
title: Use RefRW/RefRO instead of IAspect
section: burst
impact: HIGH
impactDescription: Direct access avoids abstraction overhead
---

# Use RefRW/RefRO instead of IAspect

Use `RefRW<T>` and `RefRO<T>` directly instead of creating `IAspect` implementations.

**Impact: HIGH (Direct access avoids abstraction overhead)**

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
