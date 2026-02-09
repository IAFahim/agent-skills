---
title: Optimize Archetype for Chunk Iteration
section: archetype
impact: CRITICAL
impactDescription: 10-100× performance improvement
---

# Optimize Archetype for Chunk Iteration

Design archetypes so entities with the same components are stored together in chunks.

**Impact: CRITICAL (10-100× performance improvement)**

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
// GOOD: Use IEnableableComponent for optional features
public struct RenderEnabled : IComponentData, IEnableableComponent { }
public struct PhysicsEnabled : IComponentData, IEnableableComponent { }
public struct AIEnabled : IComponentData, IEnableableComponent { }

// All entities share the same archetype, features are toggled
// Result: Single archetype, chunk iteration is optimal

[BurstCompile]
public partial struct RenderSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Automatically iterates only enabled components
        foreach (var (transform, mesh) in SystemAPI.Query<RefRW<LocalTransform>, RefRW<MeshRenderer>>()
            .WithAll<RenderEnabled>())
        {
            // Render logic
        }
    }
}
```

**Correct: Archetype Design Pattern**

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
