---
title: Follow Entity-Component-Responsibility Principle
section: arch
impact: CRITICAL
impactDescription: Prevents bloated components and systems
---

# Follow Entity-Component-Responsibility Principle

Each Entity should have a single responsibility. Components should be small and focused.

**Impact: CRITICAL (Prevents bloated components and systems)**

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
// GOOD: Small, focused components
public struct LocalTransform : IComponentData
{
    public float3 Position;
    public quaternion Rotation;
    public float Scale;
}

public struct Velocity : IComponentData
{
    public float3 Value;
}

public struct Health : IComponentData
{
    public float Value;
    public float Max;
}

public struct Mana : IComponentData
{
    public float Value;
    public float Max;
}

public struct CharacterStats : IComponentData
{
    public int Level;
    public float Experience;
    public float AttackPower;
    public float DefensePower;
}

public struct MovementState : IComponentData
{
    public float Speed;
    public bool IsBlocking;
    public bool IsAttacking;
}

public struct Inventory : IComponentData
{
    public int Gold;
    public float Weight;
}
```

**Correct: Entity Composition**

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
