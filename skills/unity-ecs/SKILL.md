---
name: unity-ecs
description: Unity Entity Component System (DOTS) architecture and performance optimization guidelines. Use when building Unity ECS systems, writing Burst-compiled jobs, optimizing DOTS performance, or reviewing ECS code.
version: "1.0.0"
---

# Unity ECS Best Practices

Elite Unity ECS (Entities) architecture guidelines with strict 4-layer design patterns. Contains rules prioritized by performance impact, from critical archetype design to Burst optimization.

## How It Works

1. Agent detects Unity ECS/Entities task or DOTS implementation
2. Agent loads relevant rules based on context (System, Component, Entity, Memory)
3. Agent applies Data-Oriented Design patterns to code generation or review

## Usage

Reference the guidelines when:
- Writing `ISystem` or `SystemBase` implementations
- Designing `IComponentData` and `ISharedComponentData`
- Optimizing archetype chunks and structural changes
- Working with `Unity.Jobs` and `Unity.Burst`
- Managing `EntityCommandBuffer` and structural changes
- Using `NativeArray`, `NativeList`, and other native containers

## Categories Covered

### 1. Architecture & Design (CRITICAL)
- 4-layer architecture separation
- System organization and dependencies
- Entity-Component-Responsibility principle

### 2. Performance & Archetypes (CRITICAL)
- Chunk iteration optimization
- Archetype design for cache efficiency
- Structural change batching

### 3. Memory & Burst (HIGH)
- Burst compiler compatibility
- Native container allocation patterns
- GC avoidance strategies

### 4. System Scheduling (HIGH)
- Job dependency management
- System ordering attributes
- Sync point minimization

## Quick Reference

| Priority | Category | Prefix |
|----------|----------|--------|
| CRITICAL | Architecture | `arch-` |
| CRITICAL | Archetypes | `archetype-` |
| CRITICAL | Structural Changes | `structural-` |
| HIGH | Memory | `memory-` |
| HIGH | Burst | `burst-` |
| HIGH | Systems | `system-` |
| MEDIUM | Jobs | `job-` |
| MEDIUM | Components | `component-` |
| LOW | Debugging | `debug-` |

## Present Results to User

When reviewing Unity ECS code:
1. List architecture violations (missing layers, coupling)
2. Identify performance bottlenecks (chunk iteration, sync points)
3. Flag Burst incompatibilities
4. Suggest archetype optimizations
5. Provide corrected code with explanations

## Troubleshooting

- **Burst errors**: Check for managed types, virtual calls, or unsupported APIs
- **Structural change performance**: Use `EntityCommandBuffer` for batching
- **Chunk iteration issues**: Verify archetype design matches access patterns
- **System dependency cycles**: Check `[UpdateBefore]`/`[UpdateAfter]` attributes

## User Preferences (IAFahim)

This skill follows your personal coding standards:
- **No Documentation**: No `///` XML summaries or block comments
- **Side Comments Only**: Comments on same line as code (`code; // comment`)
- **No Emojis**: Text-only indicators
- **[BurstCompile]**: All Logic and Systems must be Burst-compiled
- **Unity.Mathematics**: Use `float3`, `quaternion`, `float4x4` exclusively
- **No Magic Numbers**: Define `public const` in Logic classes
- **Granularity**: Prefer small, composable components over God structs
- **IEnableableComponent**: Use for state toggling, never `bool IsActive`
- **IAspect Ban**: Use `RefRW<T>`, `RefRO<T>`, and `IJobEntity` instead
