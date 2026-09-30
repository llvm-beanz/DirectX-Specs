---
title: "NNNN - Shader Debugging Intrinsics"
params:
  authors:
    - llvm-beanz: Chris Bieneman
    - joecitizen: Jack Elliott
    - gracezhang72: Grace Zhang
  sponsors:
    - austinkinross: Austin Kinross
  status: Draft
---

## Introduction

This proposal introduces a coordinated HLSL and Direct3D debugging facility for
shader code. Shader authors can detect whether live debugging is enabled and
request a breakpoint from a shader, while applications can select the behavior
of those breakpoint requests when creating a pipeline or state object. The
facility supports interactive debugging, postmortem diagnosis of rare failures,
and production configurations in which breakpoint requests are disabled without
requiring separate DXIL shader variants.

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
[numthreads(8, 1, 1)]
void main(uint GI : SV_GroupIndex) {
    if (dx::IsDebuggingEnabled()) {
        ValidateComplexInvariants();
    }

    if (someRareCondition) {
        DebugBreak();
    }
}
```

Direct3D provides per-pipeline and per-state-object policy for `DebugBreak()`.
The policy has three modes:

- **Default:** notify a registered live GPU debugger when the instruction is
  reached; otherwise continue execution.
- **Always halt:** halt even when no live debugger is registered, allowing the
  GPU fault and dump workflow to capture the failure for postmortem analysis.
- **Force no-op:** compile breakpoint requests as no-ops for pipelines where
  debugging must be disabled.

The runtime exposes device capability bits indicating support for live-debugger
notification, forced halting, and CPU-hosted debug breaks. HLSL lowers both
intrinsics to DXIL operations for DirectX targets. For SPIR-V targets,
`DebugBreak()` lowers to a non-semantic debugging instruction where available;
`dx::IsDebuggingEnabled()` remains DirectX-specific.

## Detailed design

### Terms

A **registered live debugger** is a live GPU debugger registered with the OS for
the current process, such as PIX's live debugger. Postmortem tools, including
DirectX Dump Files, and driver-internal monitoring or exception handlers are not
registered live debuggers for the behavior specified here.

The **backend compiler** is the IHV-supplied compiler that translates DXIL into
hardware-specific instructions.

### HLSL additions

Two intrinsic functions are added. They are available to all shader stages in
Shader Model 6.10 and later.

#### `DebugBreak()`

```hlsl
void DebugBreak();
```

- **Effects**: Requests a debugger breakpoint according to the selected [debug
  break policy](#debug-break-policy).
- **Remarks**: The call has observable runtime behavior and must not be removed,
  combined, speculated, or moved across other observable operations unless the
  selected pipeline policy explicitly compiles it as a no-op.

#### `dx::IsDebuggingEnabled()`

```hlsl
bool dx::IsDebuggingEnabled();
```

- **Returns**: `true` if a registered live debugger has enabled debugging for
  the currently executing thread at the point of the call, and `false`
  otherwise.
- **Remarks**: The result may change during shader execution as debugging state
  is mutable.

#### Example usage

```hlsl
if (dx::IsDebuggingEnabled()) {
    for (uint i = 0; i < arraySize; ++i) {
        if (data[i] < 0.0f || data[i] > 1.0f) {
            DebugBreak();
        }
    }
}
```

Debugging state is mutable. Results may differ between threads and between
calls if debugging state changes during shader execution. The compiler must not
assume the result is uniform or invariant for a draw, dispatch, or shader
invocation. This function is a DirectX extension and is not available when
targeting SPIR-V.

Using either intrinsic with a shader model earlier than 6.10 is a compilation
error. Using `dx::IsDebuggingEnabled()` when targeting SPIR-V or any other
non-DirectX target is also a compilation error.

### Interchange format additions

#### DXIL

Two DXIL operations are added:

```llvm
declare void @dx.op.debugBreak(
  immarg i32             ; opcode
)
```

`dx.op.debugBreak` implements the breakpoint request and is subject to the
runtime policy selected for the pipeline. It is a side-effecting operation.


```llvm
declare i1 @dx.op.isDebuggingEnabled(
  immarg i32             ; opcode
)
```

`dx.op.isDebuggingEnabled` returns one when debugging is enabled for the current
thread and zero otherwise. It queries mutable runtime state and must not be
marked `readonly` or `readnone`. Optimizers must not common separate calls or
hoist, sink, or otherwise move a call to a point at which it could observe
different runtime state.

##### Validation

The operations are valid only in Shader Model 6.10 or later. The DXIL validator
must reject them in older shader models, reject malformed signatures, and apply
the same shader-stage availability as the HLSL intrinsics. No new DXIL metadata
is required.

#### SPIR-V

For SPIR-V targets, `DebugBreak()` uses the existing
`NonSemantic.DebugBreak` extended instruction:

```spirv
%debugBreak = OpExtInstImport "NonSemantic.DebugBreak"
              OpExtInst %void %debugBreak DebugBreak
```

The instruction can be ignored safely by a runtime that does not support it.
Debugger support is not universal, so this lowering does not guarantee an
interactive break on every Vulkan implementation.

No SPIR-V lowering is defined for `dx::IsDebuggingEnabled()`. Compilation must
diagnose its use when targeting SPIR-V rather than silently substitute a
constant value.

### D3D API additions

#### Debug break policy

Two flags are added to `D3D12_PIPELINE_STATE_FLAGS`:

```c++
typedef enum D3D12_PIPELINE_STATE_FLAGS {
    // Existing values omitted.
    D3D12_PIPELINE_STATE_FLAG_DEBUG_BREAK_ALWAYS_HALT = 0x20,
    D3D12_PIPELINE_STATE_FLAG_DEBUG_BREAK_FORCE_NOP   = 0x40,
} D3D12_PIPELINE_STATE_FLAGS;
```

Equivalent flags are added to `D3D12_STATE_OBJECT_FLAGS`:

```c++
typedef enum D3D12_STATE_OBJECT_FLAGS {
    // Existing values omitted.
    D3D12_STATE_OBJECT_FLAG_DEBUG_BREAK_ALWAYS_HALT = 0x200,
    D3D12_STATE_OBJECT_FLAG_DEBUG_BREAK_FORCE_NOP   = 0x400,
} D3D12_STATE_OBJECT_FLAGS;
```

The state-object flags are supplied through the existing configuration
subobject:

```c++
typedef struct D3D12_STATE_OBJECT_CONFIG {
    D3D12_STATE_OBJECT_FLAGS Flags;
} D3D12_STATE_OBJECT_CONFIG;
```

The policies are:

| Flags | Behavior |
|-------|----------|
| Neither flag | At execution time, notify a registered live debugger if one exists; otherwise continue. |
| `*_DEBUG_BREAK_ALWAYS_HALT` | Halt shader execution whether or not a live debugger is registered. |
| `*_DEBUG_BREAK_FORCE_NOP` | Backend-compile every `DebugBreak()` in the object as a no-op. |

`*_DEBUG_BREAK_ALWAYS_HALT` and `*_DEBUG_BREAK_FORCE_NOP` are mutually
exclusive. The runtime and debug layer must reject an object that specifies
both. When neither flag is specified, the backend compiler must preserve an
execution-time check so a debugger can be attached or detached after pipeline
creation and before `DebugBreak()` executes.

The pipeline flags apply to graphics and compute pipelines created through
`CreateGraphicsPipelineState`, `CreateComputePipelineState`, and
`CreatePipelineState`.

For raytracing pipeline state objects, collections, and executable state
objects containing work graphs or generic programs, absence of a
`D3D12_STATE_OBJECT_CONFIG` subobject selects default behavior. When a config
subobject is present, every export must be associated with that subobject or a
subobject with the same debug-break policy. The same consistency requirement
applies when an existing collection is included in a larger state object.

Generic programs, pre-rasterization shader partial programs, and pixel shader
partial programs first use policy supplied through their
`D3D12_STATE_SUBOBJECT_TYPE_FLAGS` subobject. Program policy overrides
state-object policy. If no program policy is supplied, state-object policy or
the default behavior applies.

When Advanced Shader Delivery is used with either non-default policy, the
driver must just-in-time compile the shaders in the pipeline or state object, or
the shaders must have been compiled offline with matching policy.

#### Device capability

A feature query reports supported behavior:

```c++
typedef struct D3D12_FEATURE_DATA_DEBUG_BREAK {
    BOOL HaltSupported;
    BOOL LiveDebuggingSupported;
    BOOL CpuSupported;
} D3D12_FEATURE_DATA_DEBUG_BREAK;

typedef enum D3D12_FEATURE {
    // Existing values omitted.
    D3D12_FEATURE_DEBUG_BREAK = 72,
} D3D12_FEATURE;
```

`HaltSupported` indicates that `*_DEBUG_BREAK_ALWAYS_HALT` can halt GPU shader
execution. `LiveDebuggingSupported` indicates that a registered live GPU
debugger can be notified by a shader breakpoint. `CpuSupported` indicates that
a CPU-hosted implementation can issue a CPU debug break.

Shader Model 6.10 support is required for shaders containing the new DXIL
operations. `DebugBreak()` itself has no optional shader capability because it
can be implemented as a no-op. If neither `HaltSupported` nor
`LiveDebuggingSupported` is reported, all breakpoint requests execute as no-ops.
An application must query `HaltSupported` before selecting always-halt policy.

The debug layer validates mutually exclusive flags, use of always-halt policy
on unsupported devices, and consistent policy associations and collection
composition for state objects.

### DDI changes

The DDI mirrors the API policy flags:

```c++
typedef enum D3D12DDI_PIPELINE_STATE_FLAGS {
    // Existing values omitted.
    D3D12DDI_PIPELINE_STATE_FLAG_DEBUG_BREAK_ALWAYS_HALT = 0x20,
    D3D12DDI_PIPELINE_STATE_FLAG_DEBUG_BREAK_FORCE_NOP   = 0x40,
} D3D12DDI_PIPELINE_STATE_FLAGS;

typedef enum D3D12DDI_STATE_OBJECT_FLAGS {
    // Existing values omitted.
    D3D12DDI_STATE_OBJECT_FLAG_DEBUG_BREAK_ALWAYS_HALT = 0x200,
    D3D12DDI_STATE_OBJECT_FLAG_DEBUG_BREAK_FORCE_NOP   = 0x400,
} D3D12DDI_STATE_OBJECT_FLAGS;
```

The feature data is also mirrored:

```c++
typedef struct D3D12DDI_FEATURE_DATA_DEBUG_BREAK {
    BOOL HaltSupported;
    BOOL LiveDebuggingSupported;
    BOOL CpuSupported;
} D3D12DDI_FEATURE_DATA_DEBUG_BREAK;
```

The runtime passes the selected policy to the driver at pipeline or state-object
creation. The backend compiler lowers `dx.op.debugBreak` according to that
policy. Default policy must retain the ability to distinguish a registered live
debugger at instruction execution time; always-halt emits a halt independent of
debugger registration; force-no-op removes the breakpoint operation.

### Postmortem behavior

When always-halt policy is selected and `DebugBreak()` executes without a
registered live debugger, shader execution halts. An IHV KMD may notify dxgkrnl
through the designated interrupt, or dxgkrnl detects an engine timeout through
the normal TDR mechanism. The system then invokes the process debug-blob DDI and
generates a DirectX Dump File through the normal timeout workflow.

Postmortem tooling does not count as a registered live debugger and does not
suppress this halt. Under default policy without a registered live debugger, or
under force-no-op policy, no halt occurs and this proposal does not initiate a
dump.

### WARP support

WARP behavior depends on its execution backend:

- With BasicRender and `CpuSupported`, `DebugBreak()` issues a CPU debug break.
  The default debugger check uses `IsDebuggerPresent()` rather than GPU debugger
  registration. BasicRender does not generate a DirectX Dump File.
- With SoftGPU and `LiveDebuggingSupported`, default policy checks for a
  registered GPU debugger and notifies dxgkrnl when a breakpoint is reached.
- With SoftGPU, `HaltSupported`, and always-halt policy, a breakpoint reached
  without a debugger halts execution and can generate a DirectX Dump File.
- Force-no-op policy removes breakpoint behavior for either backend.

### PIX support

PIX live debugging registers through `ID3D12Tools3::RegisterLiveGpuDebugger`.
When a breakpoint is reached under default or always-halt policy, PIX must be
able to identify the stopped shader invocation, display its state, and resume
execution. Attaching or detaching PIX before a breakpoint executes must affect
the execution-time default-policy check without recreating the pipeline.

PIX pipeline inspection should display the effective debug-break policy and the
device capability bits. GPU captures and replay must preserve policy or report
clearly when the replay device cannot support it.

### Other tooling impact

DXC must expose the intrinsics, produce the new DXIL operations, preserve their
side-effect and mutable-state semantics through optimization, and emit the
SPIR-V behavior described above. The DXIL validator and disassembler must
recognize both operations.

DirectX Dump File tooling consumes deliberate always-halt failures through the
existing dump workflow. Debuggers other than PIX can participate by using the
registered live-debugger mechanism. Shader analysis, instrumentation, and
translation tools must preserve `dx.op.debugBreak` unless they knowingly apply
force-no-op policy, and must treat `dx.op.isDebuggingEnabled` as a mutable query.

### Dependencies and risks

This feature depends on Shader Model 6.10, corresponding DXIL validator and
compiler support, D3D runtime and debug-layer changes, driver compiler support,
live-debugger registration, and the DirectX Dump File workflow. Reliable early
postmortem notification may additionally depend on KMD and dxgkrnl interrupt
support.

The principal implementation risks are divergent debugger behavior across
vendors, a halted wave preventing useful forward progress or recovery, high
latency when a deliberate halt is detected only through TDR, and incorrect
optimization of the mutable debugging query. State-object composition and
offline shader delivery also risk applying a policy different from the one used
to compile an included shader. Capability queries, consistency validation, and
conformance tests mitigate these risks.

## Testing

### Compiler and validator testing

- Verify HLSL overload resolution and correct DXIL generation for both
  intrinsics in every shader stage.
- Verify `DebugBreak()` produces `NonSemantic.DebugBreak` for SPIR-V targets.
- Verify `NonSemantic.DebugBreak` under at least one debugger that implements
  the instruction, and verify that an implementation which ignores the
  non-semantic instruction can execute the shader normally.
- Verify `dx::IsDebuggingEnabled()` is diagnosed for SPIR-V targets.
- Verify optimizers preserve `DebugBreak()` and do not common or improperly move
  `dx::IsDebuggingEnabled()` calls.
- Verify the validator accepts both operations in Shader Model 6.10 and later,
  and rejects them in earlier shader models.
- Verify malformed signatures and invalid operation declarations are rejected.

### Runtime and functional testing

- Verify default policy notifies a registered live debugger and permits
  execution to resume.
- Verify default policy without a registered live debugger continues execution.
- Verify attaching or detaching a debugger can change subsequent default-policy
  behavior without pipeline recreation.
- Verify `dx::IsDebuggingEnabled()` reflects runtime state and that subsequent
  calls can observe state changes.
- Verify always-halt policy causes device removal and produces a valid DirectX
  Dump File when no live debugger is registered.
- Verify force-no-op policy allows execution beyond `DebugBreak()` and preserves
  expected UAV side effects.
- Verify BasicRender and SoftGPU behavior for every reported WARP capability.

### API, debug-layer, and conformance testing

The HLK conformance suite exercises compute pipelines and state objects. It
must:

- Verify `CheckFeatureSupport(D3D12_FEATURE_DEBUG_BREAK)` reports capability
  bits consistent with observed behavior and skips unsupported scenarios.
- Verify mutually exclusive flags and unsupported always-halt use are rejected.
- Verify graphics and compute pipeline creation propagates policy to the driver.
- Verify state-object config, per-program overrides, and collection-to-executable
  composition produce the specified effective policy.
- Verify inconsistent policy associations and cross-collection composition are
  rejected.
- Verify Advanced Shader Delivery uses matching offline compilation or JIT
  compilation for non-default policy.

## Transition strategy for breaking changes

The HLSL names are new, and existing shaders do not contain the new DXIL
operations, so the proposal does not intentionally change the meaning or
codegen of existing source. Older compiler or validator versions will reject
the new intrinsics or operations rather than miscompile them. Applications must
feature-query before requesting always-halt behavior and should retain a path
that does not use Shader Model 6.10 for older systems.

Tools that process DXIL must be updated to recognize the new operations before
accepting Shader Model 6.10 modules. No source migration or staged diagnostic is
otherwise required.

## Alternatives considered

### Application-managed debug state

Applications can bind a constant indicating that debugging is enabled. This
requires application coordination and resource bindings, may be stale when a
debugger attaches during execution, and does not provide a standardized
breakpoint operation.

### Separate debug shader variants

Applications can compile debug checks into a separate shader variant. This
removes retail overhead but increases compilation, storage, pipeline cache, and
deployment complexity. It also cannot enable diagnostics dynamically in an
already distributed shader. Per-pipeline policy retains one DXIL representation
while allowing the backend compiler to select appropriate behavior.

### Infinite loops for postmortem capture

An infinite shader loop can attempt to provoke TDR. It depends on optimization,
driver scheduling, and machine TDR configuration, has no defined relationship
to a live debugger, and does not identify an intentional breakpoint. A dedicated
operation has explicit semantics and can use a purpose-built notification path.

### Define only `DebugBreak()`

A breakpoint intrinsic alone addresses conditional stopping but does not let a
shader avoid expensive checks when debugging is unavailable. The mutable query
enables those checks without application-specific state.

### Use only a compile-time switch

A compile-time switch can remove debugging code but cannot respond to a debugger
attached after compilation and requires multiple shader variants. Runtime state
and object policy support both interactive and deployed scenarios.

## Stakeholders

Stakeholders include:

- HLSL language, compiler, DXIL, and validator owners.
- Direct3D runtime and debug-layer owners.
- Windows graphics kernel and WDDM owners.
- Hardware vendors, KMD owners, and driver compiler owners.
- PIX and other live GPU debugger owners.
- DirectX Dump File and postmortem tooling owners.
- WARP owners.
- Advanced Shader Delivery owners.
- Shader authors and engine developers.
- SPIR-V tooling and Vulkan debugger implementers for portable `DebugBreak()`
  behavior.

## Prior work

- The [Direct3D GPU Dumps specification](../../d3d/D3D12GpuDumps.md) describes
  the live and postmortem GPU debugging workflows with which this proposal
  integrates.
- The HLSL proposal was motivated in part by
  [HLSL specification issue 33](https://github.com/microsoft/hlsl-specs/issues/33).

## Open questions

- Can every participating KMD identify an intentional shader halt and notify
  dxgkrnl immediately, or must some implementations wait for TDR?
- Can every participating IHV support single-stepping past an always-halt
  breakpoint without aborting the shader? Resuming execution after the debugger
  handles a breakpoint is required by the HLSL semantics; the unresolved point
  is whether single-step behavior can be supported consistently by hardware and
  drivers.
- What exact replay behavior should PIX use when a capture's policy is not
  supported on the replay device?

## Acknowledgments

This combined proposal is derived from the HLSL Debugging Intrinsics proposal
and the Direct3D Debug Break specification, including design and implementation
input from the HLSL, PIX, Direct3D, Windows graphics, WARP, and hardware vendor
teams.