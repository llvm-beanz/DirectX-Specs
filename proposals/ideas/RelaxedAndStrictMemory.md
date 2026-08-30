---
title: "Relaxed and Strict Memory Modes"
params:
  authors:
    - llvm-beanz: Chris Bieneman
  sponsors:
    - tbd: TBD
  status: Idea
---

## Introduction

This proposal introduces the concept of relaxed and strict memory modes for
HLSL, allowing developers to opt-in to relaxed memory protections enabling
performance critical optimizations and unsafe memory operations.

## Motivation

HLSL is mostly strict in its memory model, enforcing strong memory protections
that can limit certain performance optimizations. A tension exists between the
need for strict memory safety and the desire for high performance.

Developers of performance-critical shaders often need more control over memory
operations to achieve optimal performance. Meanwhile, developers building
applications that handle untrusted inputs (such as user-generated content, or
web content) require strong memory safety guarantees to prevent security
vulnerabilities and ensure correct program behavior.

HLSL's current memory does not adaquately address the needs of either group.

This proposal aims to introduce explicit relaxed and strict memory modes in
addition to the current default mode. The new modes will provide developers with
clear semantics for memory operations, giving shader authors the ability to
choose between strict memory safety and the ability to relax memory protections
for performance optimizations and unsafe memory operations.

Along with the introduction of the new memory modes, this proposal introduces a
set of new unsafe memory operations that can only be used when the relaxed
memory mode is enabled.

## Proposed solution

This proposal defines three memory modes for shaders executing under Direct3D:

- **Default mode**: The current organically grown memory mode.
- **Strict mode**: Provides strong memory safety guarantees, preventing unsafe
  memory operations and ensuring correct program behavior. Disallows binding
  unsized resources (root descriptors).
- **Relaxed mode**: Allows developers to opt-in to relaxed memory protections,
  enabling performance-critical optimizations and new unsafe memory operations.

Enabling the **strict** or **relaxed** memory modes requires explict opting in
with a pipeline state flag. The default mode is used if no explicit memory mode
is specified.

Some shader and root signature capabilities may be disallowed when the
**strict** mode is enabled or may require that the **relaxed** mode be enabled.
These restrictions ensure that memory safety guarantees are maintained in strict
mode and that performance optimizations are only applied when explicitly allowed
in relaxed mode.

### Unsafe Memory Operations

This proposal also introduces a pair of unsafe memory operations:

```hlsl
ByteAddressBuffer ByteAddressBuffer::createFromAddress(uint64_t address, uint64_t size=MAX_UINT64);
RWByteAddressBuffer RWByteAddressBuffer::createFromAddress(uint64_t address, uint64_t size=MAX_UINT64);
```

These new unsafe memory operations allow developers to create byte address
buffers directly from a memory address, bypassing the usual safety checks. Use
of these APIs in a shader will set the shader feature flag indicating that the
shader relies on relaxed memory operations.

### PSO Creation Failure

During PSO creation, the runtime and driver must each validate that the memory
mode specified for the pipeline state is consistent with the capabilities and
requirements of the shaders and root signature. If a shader uses unsafe memory
operations, the **relaxed** memory mode must be enabled; if the **strict**
memory mode is enabled, any disallowed capabilities or unsafe operations must
trigger a PSO creation failure.