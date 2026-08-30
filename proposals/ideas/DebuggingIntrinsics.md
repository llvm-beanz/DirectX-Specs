---
title: "NNNN - Shader Debugging Intrinsics"
params:
  authors:
    - llvm-beanz: Chris Bieneman
    - joecitizen: Jack Elliott
    - gracezhang72: Grace Zhang
  sponsors:
    - tbd: TBD
  status: Idea
---

## Introduction

This proposal introduces a coordinated HLSL and Direct3D debugging facility for
shader code. Shader authors can detect whether live debugging is enabled and
request a breakpoint from a shader, while applications can select the behavior
of those breakpoint requests when creating a pipeline or state object. The
facility supports interactive debugging, postmortem diagnosis of rare failures,
and production configurations in which breakpoint requests are disabled.

## Motivation

Shader programs execute many invocations in parallel, so a defect may occur in
only one invocation among millions. Existing shader debugging tools do not give
shader authors a portable way to stop precisely when a rare, data-dependent
condition occurs and inspect that invocation's state. For example, an author may
need to break only when a computed address is outside an expected range:

```hlsl
if (computedAddress >= allocationEnd) {
    // No HLSL operation currently requests a debugger breakpoint here.
}
```

Applications also lack a shader-visible way to determine whether live debugging
is enabled. Expensive invariant checks must therefore run in every build, be
controlled through application-specific bindings, or be removed from production
shaders. Removing the checks requires maintaining or compiling separate shader
variants and prevents a debugger attached later from enabling them dynamically.

Postmortem diagnosis has a related problem. Some applications intentionally
enter an infinite shader loop to provoke a Timeout Detection and Recovery (TDR)
event and obtain a GPU crash dump when a critical invariant fails. This is an
unreliable signaling mechanism: compiler optimization, driver behavior, and
machine-specific TDR settings can prevent or delay the expected result, and an
infinite loop cannot communicate that the halt was intentional.

Finally, interactive, postmortem, and retail use cases need different behavior
from the same shader bytecode. A breakpoint request should normally notify a
registered live debugger when one exists, but selected pipelines may need to
halt without a debugger so that a postmortem dump is produced, while retail
pipelines may need to eliminate all breakpoint overhead. Recompiling HLSL for
each policy complicates shader distribution and pipeline caching.

## Proposed solution

Add two HLSL intrinsics:

```hlsl
void DebugBreak();
bool dx::IsDebuggingEnabled();
```

`DebugBreak()` requests a breakpoint at the point where it executes.
`dx::IsDebuggingEnabled()` queries mutable debugging state so shader code can
conditionally execute diagnostics. A typical use is:

```hlsl
if (dx::IsDebuggingEnabled()) {
    ValidateComplexInvariants();
}

if (someRareCondition) {
    DebugBreak();
}
```

Direct3D will provide per-pipeline and per-state-object policy for
`DebugBreak()`. The policies will cover three application scenarios:

- **Default:** notify a registered live GPU debugger when the instruction is
  reached; otherwise continue execution.
- **Always halt:** halt even when no live debugger is registered, allowing the
  system's GPU fault and dump workflow to capture the failure for postmortem
  analysis.
- **Force no-op:** compile breakpoint requests as no-ops for pipelines where
  debugging must be disabled.

The runtime will expose device capability bits indicating support for
live-debugger notification and forced halting, including on CPU-hosted
implementations such as WARP. HLSL will lower both intrinsics to DXIL
operations for DirectX targets, preserving their runtime-observable behavior.
For SPIR-V targets, only `DebugBreak()` has a mapping, using an appropriate
non-semantic debugging instruction where available, since
`dx::IsDebuggingEnabled()` is DirectX-specific. The shader model, exact API and DDI surface,
validation rules, state-object composition behavior, WARP behavior, and tooling
integration will be developed during the Draft stage.

## Stakeholders

Provisional stakeholders include:

- HLSL language, compiler, DXIL, and validator owners.
- Direct3D runtime and debug-layer owners.
- Windows graphics kernel and WDDM owners.
- Hardware vendors and driver compiler owners.
- PIX and other live GPU debugger owners.
- DirectX Dump File and postmortem tooling owners.
- WARP owners.
- Shader authors and engine developers.
- SPIR-V tooling and Vulkan debugger implementers for the portable
  `DebugBreak()` behavior.

## Prior work

- The existing
  [HLSL debugging intrinsics proposal](https://github.com/microsoft/hlsl-specs/blob/main/proposals/0039-debugbreak.md)
  specifies `DebugBreak()`, `dx::IsDebuggingEnabled()`, and their DXIL and SPIR-V
  lowering.
- The existing [Direct3D Debug Break specification](../../d3d/D3D12DebugBreak.md)
  defines pipeline and state-object policy, capability discovery, DDI behavior,
  WARP behavior, and conformance testing.
- The [Direct3D GPU Dumps specification](../../d3d/D3D12GpuDumps.md) describes
  the live and postmortem GPU debugging workflows with which this proposal must
  integrate.

## Open questions

- What precisely constitutes enabled debugging for
  `dx::IsDebuggingEnabled()`, and can its result vary by thread or during one
  shader invocation?
- How should a deliberate shader halt be reported to the graphics kernel when
  no live debugger is registered: through an explicit driver interrupt, the TDR
  path, or both?
- Which behaviors must every Shader Model implementation support, and which
  require device capability bits?
- How should policy be inherited and validated across collections, executable
  state objects, generic programs, and partial programs?
- What guarantees are required for continuing execution after a live debugger
  handles `DebugBreak()`?
- Which HLSL behavior can be represented consistently in SPIR-V, and how should
  unsupported debugging queries be diagnosed for non-DirectX targets?
