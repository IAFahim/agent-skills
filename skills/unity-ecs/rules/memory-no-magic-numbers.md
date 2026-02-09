---
title: Define Constants in Logic Classes
section: memory
impact: HIGH
impactDescription: Improves maintainability and Burst optimization
---

# Define Constants in Logic Classes

Define `public const` values in Logic classes instead of magic numbers.

**Impact: HIGH (Improves maintainability and Burst optimization)**

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
// GOOD: Constants in Logic class
public static class MovementConstants
{
    public const float FRICTION = 0.95f;
    public const float MAX_VELOCITY = 50.0f;
    public const float GROUND_HEIGHT = 0.1f;
}

[BurstCompile]
public partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (transform, velocity) in SystemAPI.Query<RefRW<LocalTransform>, RefRW<Velocity>>())
        {
            velocity.ValueRW.Value *= MovementConstants.FRICTION;
            velocity.ValueRW.Value = math.clamp(velocity.ValueRW.Value, -MovementConstants.MAX_VELOCITY, MovementConstants.MAX_VELOCITY);
            transform.ValueRW.Position.y = math.max(transform.ValueRW.Position.y, MovementConstants.GROUND_HEIGHT);
        }
    }
}
```

**Best: Constants per System**

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
