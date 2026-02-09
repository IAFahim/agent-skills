# Unity ECS Best Practices

Elite Unity ECS (Entities) architecture guidelines with strict 4-layer design patterns for optimal performance and maintainability.

## About

This skill provides comprehensive guidelines for Unity Entity Component System (DOTS) development, with rules prioritized by performance impact. Following IAFahim's personal coding standards, it emphasizes Data-Oriented Design, Burst compilation, and clean architecture.

## Categories

| Section | Priority | Focus |
|---------|----------|-------|
| `arch-` | CRITICAL | 4-layer architecture, ECR principle |
| `archetype-` | CRITICAL | Chunk iteration optimization |
| `structural-` | CRITICAL | Batching structural changes |
| `burst-` | CRITICAL/HIGH | Unity.Mathematics, RefRW/RefRO |
| `component-` | HIGH | IEnableableComponent usage |
| `system-` | HIGH | System ordering and dependencies |
| `job-` | HIGH | IJobEntity patterns |
| `memory-` | HIGH | Constants and allocation patterns |
| `debug-` | MEDIUM | Code style and conventions |

## Key Standards

- **No Documentation**: No `///` XML summaries or block comments
- **Side Comments Only**: Comments on same line as code
- **No Emojis**: Text-only indicators
- **[BurstCompile]**: All Logic and Systems must be Burst-compiled
- **Unity.Mathematics**: Use `float3`, `quaternion`, `float4x4` exclusively
- **No Magic Numbers**: Define `public const` in Logic classes
- **Granularity**: Prefer small, composable components
- **IEnableableComponent**: Use for state toggling, never `bool IsActive`
- **IAspect Ban**: Use `RefRW<T>`, `RefRO<T>`, and `IJobEntity` instead

## Installation

1. Copy the `unity-ecs` folder to your `skills/` directory
2. Build with: `cd packages/react-best-practices-build && npm run build-unity`
3. Use via Craft Agent or any LLM that supports skill loading

## Usage

Reference the guidelines when:
- Writing `ISystem` or `SystemBase` implementations
- Designing `IComponentData` and `ISharedComponentData`
- Optimizing archetype chunks and structural changes
- Working with `Unity.Jobs` and `Unity.Burst`
- Managing `EntityCommandBuffer` and structural changes

## Version

Version 1.0.0 - February 2026
