---
title: Use Unity.Mathematics Types Exclusively
section: burst
impact: CRITICAL
impactDescription: Required for Burst SIMD optimization
---

# Use Unity.Mathematics Types Exclusively

Use `Unity.Mathematics` types (`float3`, `quaternion`, `float4x4`) instead of `UnityEngine` types.

**Impact: CRITICAL (Required for Burst SIMD optimization)**

`Unity.Mathematics` types are Burst-compatible and use SIMD instructions. `UnityEngine.Vector3` causes Burst errors.

**Incorrect:**

```csharp
// BAD: UnityEngine types cause Burst errors
using UnityEngine;

public struct Velocity : IComponentData
{
    public Vector3 Value; // Not Burst-compatible!
}

[BurstCompile] // Will fail to compile!
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
```

**Correct:**

```csharp
// GOOD: Unity.Mathematics types
using Unity.Mathematics;

public struct Velocity : IComponentData
{
    public float3 Value;
}

[BurstCompile]
public partial struct MovementSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        float dt = SystemAPI.Time.DeltaTime;

        foreach (var (transform, velocity) in SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            transform.ValueRW.Position += velocity.ValueRO.Value * dt;
        }
    }
}
```

**Common Type Conversions:**

```csharp
// GOOD: Use math library for operations
using static Unity.Mathematics.math;

// Vector operations
float3 direction = normalize(target - origin);
float distance = length(target - origin);
float3 rotated = rotateY(forward, angle);

// Quaternion operations
quaternion rotation = quaternion.AxisAngle(up, angle);
float3 rotatedVector = rotate(rotation, forward);

// Interpolation
float3 lerpPosition = lerp(from, to, t);
quaternion slerpRotation = slerp(fromRotation, toRotation, t);

// Math functions
float clamped = clamp(value, min, max);
float eased = smoothstep(0, 1, t);
```

**Conversion Between Types:**

```csharp
// Convert Unity.Mathematics to UnityEngine (for non-Burst code)
float3 mathPos;
UnityEngine.Vector3 unityPos = mathPos;

// Convert UnityEngine to Unity.Mathematics
UnityEngine.Vector3 unityPos;
float3 mathPos = unityPos;
```
