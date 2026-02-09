---
title: Batch Structural Changes with EntityCommandBuffer
section: structural
impact: CRITICAL
impactDescription: 100× faster than per-entity changes
---

# Batch Structural Changes with EntityCommandBuffer

Use `EntityCommandBuffer` to batch structural changes (add/remove components, create/destroy entities).

**Impact: CRITICAL (100× faster than per-entity changes)**

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
// GOOD: Batch with EntityCommandBuffer
[BurstCompile]
public partial struct SpawnSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.TempJob);
        var prefab = SystemAPI.GetSingleton<EnemyPrefab>();

        for (int i = 0; i < 1000; i++)
        {
            var entity = ecb.Instantiate(prefab);
            ecb.SetComponent(entity, new SpawnPosition { Value = GetSpawnPoint(i) });
        }

        ecb.Playback(state.EntityManager); // Single sync point
        ecb.Dispose();
    }
}
```

**Best: EntityCommandBufferSystem**

```csharp
// BETTER: Use ECBSystem for automatic playback
[UpdateInGroup(typeof(SimulationSystemGroup))]
public partial struct SpawnSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        var ecbSystem = SystemAPI.GetSingleton<BeginSimulationECBSystem>();
        var ecb = ecbSystem.CreateCommandBuffer();
        var prefab = SystemAPI.GetSingleton<EnemyPrefab>();

        for (int i = 0; i < 1000; i++)
        {
            var entity = ecb.Instantiate(prefab);
            ecb.SetComponent(entity, new SpawnPosition { Value = GetSpawnPoint(i) });
        }
        // ECBSystem handles playback automatically
    }
}
```

**Correct: Multiple Component Operations**

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
