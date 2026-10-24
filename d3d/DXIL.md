# DirectX Intermediate Language

This specification describes DXIL for compiler, validator, driver, and tool
implementers. It separates LLVM-derived instructions, DXIL operations, and
binary serialization.

## Scope and completion status

This draft establishes the structure for a complete DXIL specification.
It specifies eight fixed-width integer operations using the operation
definition format from the supplied linear-algebra proposal.

The completed operation families are:

- Integer multiplication and division with two results: `IMul`, `UMul`,
  and `UDiv`.
- Integer arithmetic with carry or borrow: `UAddc` and `USubb`.
- Bitfield extraction and insertion: `Ibfe`, `Ubfe`, and `Bfi`.

These operations have fixed `i32` operands, established DXBC counterparts,
and observable boundary behavior. They exercise shared declarations,
aggregate results, signedness, and operand normalization.

Other sections establish ownership, summarize verified rules, and identify
the remaining specification work. **TBD** identifies incomplete coverage,
not implementation permission or undefined behavior.

This draft does not claim complete operation coverage or replace published
shader-model specifications. Proposed linear-algebra behavior remains
separate from the completed operation definitions.

## Specification conventions and sources

### Requirement language

**Must** states a requirement. **Must not** states a prohibition.
**Can** states a capability or permission.

**Invalid** describes a violation of the static DXIL contract.
**Undefined behavior** describes execution without a specified outcome.
A defined exceptional result, such as division by zero below, is neither.

Operation signatures and their requirement bullets define the contract.
Examples and conformance vectors illustrate that contract.
Sections marked **TBD** are editorial work items.

### Source baseline

DXC references in this draft use commit
`194bb75bea2becb42810d72a16195089c1908e10`.
This pins the evidence, not a product release or supported deployment.

| Source | Responsibility |
| --- | --- |
| [DXIL reference][dxil] | DXIL restrictions, metadata, operations, validation |
| [LLVM language reference][langref] | LLVM-derived instruction semantics |
| [LLVM bitcode format][bitcode] | Bitstream and LLVM IR record encoding |
| [D3D11.3 specification][d3d11] | Explicitly inherited DXBC behavior |
| [DXC operation database][opdb] | Opcode, class, signature, and availability |
| [DXC operation implementation][opimpl] | Declaration and result-type ABI |
| [DXBC converter][converter] | DXBC-to-DXIL operand and result correspondence |
| [Container definitions][container-header] | Container and program layouts |
| [Reflection definitions][rdat-header] | RDAT headers and table layout |

The supplied D3D11.3 snapshot is version 1.17, dated June 20, 2025.
It is byte-identical to the archived copy linked here at the repository
revision used to prepare this draft.

The supplied linear-algebra snapshot provides the declaration-and-bullets
template. The [repository proposal][linear-algebra] provides a discoverable
related reference, not a claim that both snapshots are identical.

[microsoft/DirectXShaderCompiler#8631][format-pr] supplies additional
[container][container-draft] and [RDAT][rdat-draft] descriptions.
Those references are pinned to PR head
`ee43ad5b0068bf6677e2804fce1a3b906bb8dc20`.
The PR was open when consulted.
Its description identifies substantial LLM-generated content.
This draft uses it as an explanatory resource, not an independent
normative authority. Layouts included below were checked against DXC headers.

### Inheritance and conflict handling

DXIL uses a restricted LLVM-derived IR. A well-formed LLVM module is not
necessarily valid DXIL.

- The LLVM reference defines instruction behavior.
- DXIL defines the permitted subset and explicit DXIL-specific restrictions
  or semantic refinements.
- A DXIL operation definition specifies the behavior of a `dx.op` call.
- A cited DXBC counterpart supplies only the behavior explicitly inherited
  by that operation definition.
- The bitcode reference defines serialization, not execution semantics.
- The container format wraps bitcode and other parts. It is not the LLVM
  bitstream format.

DXBC instruction names do not establish a one-to-one mapping to LLVM
instructions. For example, DXBC `udiv` maps to DXIL operation `UDiv`.
That operation has two results and a defined zero-divisor result.
LLVM instruction `udiv` has one result and a different zero-divisor contract.

DXBC register syntax, swizzles, destination masks, null destinations,
and source modifiers are not implicit DXIL operation operands.
Producers express required computations and result uses explicitly in IR.

Conflicting sources require a recorded resolution.
This draft does not silently select a rule based on publication recency.

## Version and capability model

The [DXIL versioning section][dxil-versioning] distinguishes shader model,
DXIL version, and LLVM bitcode version. These are separate contracts.

### Shader model and shader kind

`!dx.shaderModel` identifies the shader kind and shader-model version.
Shader kind determines the execution environment and stage restrictions.

The complete specification must define:

- Accepted profiles and shader kinds for every supported shader model.
- Entry-point and exported-function restrictions.
- Operation, type, and metadata availability.
- Feature requirements beyond the shader-model version.
- Library target compatibility and linked-function requirements.
- Preview and experimental capability gates.

**TBD:** Publish the complete availability matrix, including stage-specific
minimum versions. Do not derive support solely from opcode presence.

### DXIL version and validator version

`!dx.version` identifies the DXIL version.
`!dx.valver` identifies validator-version metadata.
The DXIL version applies to the module.

**TBD:** Define accepted combinations of shader model, DXIL version,
validator version, and feature flags. Include compatibility behavior for
older consumers and newer metadata.

### LLVM representation version

The DXIL reference fixes the baseline bitcode representation at LLVM 3.7.
The DXC copies of the LLVM references are the sources used here.
Current upstream LLVM documentation is not a substitute for this baseline.

**TBD:** Enumerate DXIL extensions and deviations from that baseline.
Include newer vector forms without implying general acceptance of modern
LLVM IR, attributes, or opaque pointers.

## Module and execution model

### Module structure

A DXIL module contains functions, global values, types, declarations,
attributes, and metadata. The [DXIL reference][dxil] supplies the existing
module restrictions.

**TBD:** Specify required module records, target information, symbol rules,
linkage, visibility, global initialization, and accepted attributes.
Distinguish executable shader modules from libraries.

### Functions and control flow

LLVM-derived instructions express control flow and SSA value flow.
The DXIL reference restricts recursive calls and irreducible control flow.

**TBD:** Define function signatures, call restrictions, entry-point
reachability, return behavior, loop metadata, and control-flow validation.
Specify library exceptions independently.

### Invocations, lanes, waves, and groups

**TBD:** Define invocation identity, active lanes, helper lanes, quad formation,
wave membership, and thread-group membership.
Specify divergence, reconvergence, participation requirements, and wave-size
constraints before defining collective operations.

### Shader-stage execution

**TBD:** Define the execution contract for vertex, hull, domain, geometry,
pixel, compute, amplification, mesh, raytracing, and node shaders.
For libraries, define shader-kind propagation through the call graph.
Separate stage requirements from device capabilities.

## Type system and data representation

### Scalar types

LLVM integer types do not encode signedness.
An operation or instruction determines signed interpretation.
For example, the definitions below interpret identical `i32` types
differently for `IMul` and `UMul`.

**TBD:** Specify accepted integer and floating-point widths, Boolean values,
minimum precision, native low precision, and their feature gates.

### Aggregate, vector, and matrix types

Aggregate results use named LLVM structure types.
`extractvalue` accesses a structure field.
It is distinct from `extractelement`, which accesses a vector element.

**TBD:** Specify arrays, structures, vector lengths, matrix representations,
component types, opaque handles, payloads, node records, and layout metadata.
Separate legacy scalarization restrictions from version-gated vector support.

### Pointer types and address spaces

The DXIL reference defines pointer widths and address-space assignments.
Resource handles are not ordinary pointers.

**TBD:** Define every address space, pointer provenance restrictions,
legal casts, alignment, object bounds, and address calculations.
Include groupshared memory, thread-local memory, and node-record memory.

### Operation result types

The following types are used by the completed operation definitions.
Their field order and field types are part of the ABI.
They match [DXC's type construction][result-types].

```llvm
%dx.types.twoi32 = type { i32, i32 }
%dx.types.i32c = type { i32, i1 }
```

`%dx.types.twoi32` does not assign a universal meaning to either field.
The operation defines each field's meaning.

`%dx.types.i32c` contains an arithmetic result followed by a Boolean
carry or borrow. The second field is `i1`, not `i32`.

## Numerical behavior

### Integer behavior

Each operation defines signedness, result width, and exceptional inputs.
The completed definitions use two's-complement signed integers and
unsigned bit patterns as stated individually.

**TBD:** Define shared rules for wrapping, conversions, shifts, packed
components, and integer precision. Preserve differences between LLVM
instruction behavior and DXIL operation behavior.

### Floating-point behavior

**TBD:** Define representations, signed zero, infinities, NaNs, rounding,
denormals, accuracy bounds, and conversion behavior for each supported width.
Reconcile [D3D floating-point rules][d3d-float] with version-specific DXIL
denormal modes and operation exceptions.

### Optimization permissions and precise behavior

**TBD:** Define permitted fast-math flags, reassociation, contraction,
`precise` propagation, and denormal-mode attributes.
Distinguish `FMad` from `Fma`.
Do not transfer an LLVM optimization permission to a DXIL operation
without an explicit rule.

## Memory and synchronization model

### Storage and access

**TBD:** Specify thread-local, groupshared, resource, payload, and node-record
storage. Define initialization, lifetime, aliasing, alignment, and access
granularity for each category.

### Resource bounds and residency

The DXIL reference describes inherited resource bounds behavior.
This does not establish one universal bounds rule for every memory access.

**TBD:** Specify bounds checks per resource kind and operation.
Cover partial accesses, structured-buffer offsets, texture coordinates,
unmapped tiles, status returns, and descriptor indexing.
Do not apply resource-load zero results to arbitrary LLVM pointer loads.

### Atomic operations

**TBD:** Specify atomic locations, supported widths and formats,
read-modify-write behavior, comparison behavior, and return values.
Separate atomicity from synchronization and memory visibility.

### Barriers and ordering

**TBD:** Specify memory categories, synchronization scopes, execution
participation, divergent barrier behavior, and visibility.
Cover legacy barriers and memory-type, resource-handle, and node-record
barriers independently.

## LLVM-derived instructions

### Instruction semantics and permitted subset

The [LLVM language reference][langref] defines instruction semantics.
The [DXIL instruction list][instruction-list] defines the DXIL subset.
These lists do not define `dx.op` operation behavior.

| Instruction family | Representative LLVM instructions |
| --- | --- |
| Terminators | `ret`, `br`, `switch` |
| Integer arithmetic | `add`, `sub`, `mul`, `udiv`, `sdiv`, `urem`, `srem` |
| Floating arithmetic | `fadd`, `fsub`, `fmul`, `fdiv`, `frem` |
| Bitwise operations | `shl`, `lshr`, `ashr`, `and`, `or`, `xor` |
| Memory access | `alloca`, `load`, `store`, `getelementptr` |
| Pointer atomics | `cmpxchg`, `atomicrmw` |
| Conversion | `trunc`, `zext`, `sext`, `fptrunc`, `fpext`, `bitcast` |
| Numeric conversion | `fptoui`, `fptosi`, `uitofp`, `sitofp` |
| Comparison and SSA | `icmp`, `fcmp`, `phi`, `select` |
| Calls and aggregates | `call`, `extractvalue`, `insertvalue` |
| Vectors | `extractelement`, `insertelement`, `shufflevector` |

This table is an architectural grouping, not a complete legality table.

**TBD:** Specify each instruction's accepted operand types, attributes,
address spaces, and version gates.
Enumerate prohibited instructions and accepted LLVM intrinsics.
Preserve DXIL-specific floating-point refinements documented in DXC.

### Undefined and unconstrained values

**TBD:** Define the accepted uses of `undef`, uninitialized storage,
out-of-range conversion results, and LLVM undefined behavior.
Do not introduce modern LLVM value categories without baseline support.

## Metadata, signatures, and resources

### Module and function metadata

**TBD:** Specify required and optional named metadata, tag encodings,
ordering requirements, duplicate handling, unknown-tag handling, and
version-specific extensions.
Cover `!dx.version`, `!dx.valver`, `!dx.shaderModel`, `!dx.entryPoints`,
`!dx.resources`, `!dx.typeAnnotations`, and stage-specific properties.

### Shader signatures

The [signature sections][signatures] define existing signature metadata,
semantic interpretation, packing, and access operations.

**TBD:** Consolidate signature-element layouts, interpolation modes,
system values, packing constraints, and cross-stage matching.
Include patch, primitive, vertex, and mesh outputs.

### Resource metadata and handles

The [resource sections][resources] define metadata records and handle-based
resource access.

**TBD:** Specify resource classes, kinds, component types, formats, binding
ranges, register spaces, properties, and nonuniform indexing.
Cover binding-based handles, heap-based handles, annotations, and library
resource references with their version gates.

### Feature and stage properties

**TBD:** Define feature flags, thread-group dimensions, wave-size properties,
tessellation properties, geometry streams, mesh limits, raytracing payloads,
and node properties.
State consistency requirements between metadata and container parts.

## DXIL operations

### Operation definition format

A DXIL operation is identified by its opcode and operation table.
The completed entries below use `CoreOps`, table ID `0`.
Several opcodes share one external function declaration.

For example, `IMul`, `UMul`, and `UDiv` share
`@dx.op.binaryWithTwoOuts.i32`.
The first argument selects the operation.
The function name alone does not identify its behavior.

Each operation definition contains:

- A call signature with exact operand and result types and an immediate opcode.
- **Validation:** Static operand, version, stage, and capability constraints.
- **Effects:** Observable execution behavior.
- **Memory:** Memory access, ordering, or an explicit absence of memory access.
- **Returns:** Result values, including exceptional inputs and field order.
- **Remarks:** Additional interpretation or DXBC mapping details.
- **Sources:** Opcode, declaration, and inherited behavioral evidence.

Repeated **Validation** bullets express independent constraints.
An omitted category inherits an explicit family rule, not an unstated default.

Operation signatures use LLVM call syntax with the required immediate opcode.
Other operands are named placeholders.
The signatures do not use `immarg`, which is not a baseline LLVM 3.7 attribute.

### Common rules for completed integer operations

These rules apply to all eight completed definitions below.

- **Validation:** The shader model is at least 6.0 and the DXIL version
  is at least 1.0. Module version combinations must also be compatible.
- **Validation:** The opcode operand is a constant `i32` with the value
  shown in the call signature.
- **Validation:** Arguments and result types match the call signature.
  Only the `i32` overload is defined for these operations.
- **Validation:** These operations impose no additional shader-kind
  restriction. Library functions remain subject to their target rules.
- **Effects:** Each invocation computes its own result.
  No participation by other invocations is required.
- **Memory:** No memory is read or written. The operation provides no
  synchronization or memory-ordering effect.
- **Remarks:** Every non-opcode `i32` input bit pattern is accepted.
  Width and offset normalization is execution behavior, not a reason
  to reject an operand.

The [operation database][opdb] defines availability and overloads.
The [operation implementation][opimpl] defines the shared declaration names.

### Integer multiplication and division with two results

The [DXBC conversion implementation][two-result-mapping] maps the first
DXBC destination to field `0` and the second destination to field `1`.
For multiplication, this means high bits precede low bits.
For division, the quotient precedes the remainder.

Unused aggregate fields require no null destination operand.

#### `IMul`

```llvm
%result = call %dx.types.twoi32 @dx.op.binaryWithTwoOuts.i32(
  i32 41,
  i32 %a,
  i32 %b
)
```

- **Effects:** Interprets `a` and `b` as signed two's-complement integers
  and computes their exact 64-bit product.
- **Returns:** Field `0` contains bits `63:32` of the product.
  Field `1` contains bits `31:0`.
- **Remarks:** The product is representable in signed 64-bit arithmetic
  for every pair of signed 32-bit operands.
  An `i64` operation overload or an `i64` shader operand is not required.
- **Sources:** DXIL `IMul`, DXC `BinaryWithTwoOuts`, and
  [DXBC `imul`, section 22.12.6][dxbc-imul].

#### `UMul`

```llvm
%result = call %dx.types.twoi32 @dx.op.binaryWithTwoOuts.i32(
  i32 42,
  i32 %a,
  i32 %b
)
```

- **Effects:** Interprets `a` and `b` as unsigned integers and computes
  their exact 64-bit product.
- **Returns:** Field `0` contains bits `63:32` of the product.
  Field `1` contains bits `31:0`.
- **Remarks:** Signed interpretation of either input would change the
  high result. It does not change the low 32 result bits.
- **Sources:** DXIL `UMul`, DXC `BinaryWithTwoOuts`, and
  [DXBC `umul`, section 22.12.13][dxbc-umul].

#### `UDiv`

```llvm
%result = call %dx.types.twoi32 @dx.op.binaryWithTwoOuts.i32(
  i32 43,
  i32 %a,
  i32 %b
)
```

- **Effects:** Interprets `a` and `b` as unsigned integers.
  For nonzero `b`, computes the integer quotient and remainder.
- **Returns:** For nonzero `b`, field `0` is `floor(a / b)`.
  Field `1` is `a - field0 * b` and is less than `b`.
  For zero `b`, both fields are `0xffffffff`.
- **Remarks:** A zero divisor is not an invalid operand.
  LLVM `udiv` and `urem` cannot replace this operation without preserving
  its defined zero-divisor behavior.
- **Sources:** DXIL `UDiv`, DXC `BinaryWithTwoOuts`, and
  [DXBC `udiv`, section 22.12.9][dxbc-udiv].

### Integer arithmetic with carry or borrow

The [DXBC conversion implementation][carry-mapping] preserves the arithmetic
result in field `0`. It zero-extends field `1` when producing a DXBC
32-bit carry or borrow destination.

#### `UAddc`

```llvm
%result = call %dx.types.i32c @dx.op.binaryWithCarryOrBorrow.i32(
  i32 44,
  i32 %a,
  i32 %b
)
```

- **Effects:** Interprets `a` and `b` as unsigned integers and computes
  their mathematical sum.
- **Returns:** Field `0` contains the sum modulo `2^32`.
  Field `1` is `true` exactly when the sum is at least `2^32`.
- **Remarks:** There is no carry-in operand and no implicit status register.
- **Sources:** DXIL `UAddc`, DXC `BinaryWithCarryOrBorrow`, and
  [DXBC `uaddc`, section 22.12.8][dxbc-uaddc].

#### `USubb`

```llvm
%result = call %dx.types.i32c @dx.op.binaryWithCarryOrBorrow.i32(
  i32 45,
  i32 %a,
  i32 %b
)
```

- **Effects:** Interprets `a` and `b` as unsigned integers and subtracts
  `b` from `a`.
- **Returns:** Field `0` contains the difference modulo `2^32`.
  Field `1` is `true` exactly when `a < b`.
- **Remarks:** There is no borrow-in operand.
  The borrow Boolean is not the inverse of the specified condition.
- **Sources:** DXIL `USubb`, DXC `BinaryWithCarryOrBorrow`, and
  [DXBC `usubb`, section 22.12.14][dxbc-usubb].

### Bitfield extraction and insertion

Bits are numbered from `0`, the least significant bit, through `31`.
For these operations, define `w = width & 31` and `o = offset & 31`.
Only these normalized values participate in the computation.

#### `Ibfe`

```llvm
%result = call i32 @dx.op.tertiary.i32(
  i32 51,
  i32 %width,
  i32 %offset,
  i32 %value
)
```

- **Effects:** Extracts a bitfield from `value`, beginning at bit `o`,
  and sign-extends the extracted field.
- **Returns:** For `w == 0`, returns `0`.
  Otherwise, let `n = min(w, 32 - o)`.
  Extracts bits `o + n - 1` through `o` into bits `n - 1` through `0`.
  Fills the remaining high bits with extracted bit `n - 1`.
- **Remarks:** A field extending past bit `31` uses the available high
  bits and sign-extends from bit `31` of `value`.
  It does not wrap to bit `0`.
- **Sources:** DXIL `Ibfe`, DXC `Tertiary`, and
  [DXBC `ibfe`, section 22.11.6][dxbc-ibfe].

#### `Ubfe`

```llvm
%result = call i32 @dx.op.tertiary.i32(
  i32 52,
  i32 %width,
  i32 %offset,
  i32 %value
)
```

- **Effects:** Extracts a bitfield from `value`, beginning at bit `o`,
  and zero-extends the extracted field.
- **Returns:** For `w == 0`, returns `0`.
  Otherwise, let `n = min(w, 32 - o)`.
  Extracts bits `o + n - 1` through `o` into bits `n - 1` through `0`.
  Sets the remaining high bits to `0`.
- **Remarks:** A field extending past bit `31` uses the available high
  bits. It does not wrap to bit `0`.
- **Sources:** DXIL `Ubfe`, DXC `Tertiary`, and
  [DXBC `ubfe`, section 22.11.11][dxbc-ubfe].

#### `Bfi`

```llvm
%result = call i32 @dx.op.quaternary.i32(
  i32 53,
  i32 %width,
  i32 %offset,
  i32 %value,
  i32 %replacedValue
)
```

- **Effects:** Replaces a bitfield in `replacedValue` with low bits
  from `value`.
- **Returns:** Let `mask = (((2^w) - 1) << o) & 0xffffffff`.
  Returns `((value << o) & mask) | (replacedValue & ~mask)`,
  restricted to 32 bits.
- **Remarks:** The expression uses mathematical integers before the
  explicit 32-bit restriction. It does not introduce LLVM shift
  undefined behavior.
  For `w == 0`, returns `replacedValue`.
  For `w + o > 32`, replaces only bits through bit `31`.
- **Sources:** DXIL `Bfi`, DXC `Quaternary`, and
  [DXBC `bfi`, section 22.11.2][dxbc-bfi].

### Floating-point arithmetic, classification, and rounding

**TBD:** Define `FAbs`, `Saturate`, classification, min/max, rounding,
transcendentals, reciprocal square root, multiply-add, and dot products.
Specify overloads, NaN handling, signed-zero handling, denormal modes,
accuracy, and optimization permissions individually.

Source ownership: DXIL operation entries and D3D11.3 sections 3.1,
22.10, and 22.14. Resolve the rounding conflict recorded below first.

### Remaining integer, bit, and packed arithmetic

**TBD:** Define integer min/max, multiply-add, masked SAD, bit reversal,
population count, bit scans, packed dot products, and packing operations.
Specify bit-scan result numbering independently of HLSL intrinsic behavior.

Source ownership: DXIL operation entries and D3D11.3 sections 22.11
and 22.12. Do not inherit a 32-bit rule for other widths implicitly.

### Conversions and bit reinterpretation

**TBD:** Define legacy floating-point conversions, double construction
and splitting, bitcast operations, and packed-component conversions.
Distinguish operations from LLVM conversion instructions.

### Shader inputs, outputs, and system values

**TBD:** Define signature loads and stores, control-point access,
patch constants, barycentrics, vertex attributes, and system-value queries.
Specify valid signature IDs, row and column operands, and stage restrictions.

### Resource handles and constant buffers

**TBD:** Define handle creation, annotation, heap access, constant-buffer
loads, legacy loads, and library handle operations.
Specify uniformity, binding, range, alignment, and type constraints.

### Buffer and texture access

**TBD:** Define typed, raw, structured, and vector loads and stores.
Specify masks, coordinates, offsets, alignment, formats, bounds, and
status fields for every resource kind.

### Sampling, gathering, and sampler feedback

**TBD:** Define sampling, comparison, explicit level, gradients, bias,
gathering, raw gathering, LOD queries, sample positions, and feedback writes.
Specify sampler compatibility, derivative requirements, coordinate rules,
LOD clamping, offsets, filtering precision, and residency.

### Resource atomics and counters

**TBD:** Define atomic binary operations, compare-exchange, and UAV counters.
Specify operation selectors, accepted formats, address rules, result values,
and interaction with the memory model.

### Synchronization operations

**TBD:** Define `Barrier`, `BarrierByMemoryType`, `BarrierByMemoryHandle`,
and `BarrierByNodeRecordHandle`.
Specify legal flags, storage categories, execution scope, and uniformity.

### Pixel derivatives, evaluation, and coverage

**TBD:** Define coarse and fine derivatives, attribute evaluation, discard,
coverage, helper-lane queries, and sample-frequency queries.
State quad participation and helper-lane behavior per operation.

### Wave and quad operations

**TBD:** Define votes, ballots, shuffles, reductions, scans, matching,
multi-prefix operations, quad reads, quad votes, and wave/group queries.
Specify active-lane sets, helper-lane participation, result ordering,
undefined source lanes, wave-size dependencies, and convergence.

### Tessellation and geometry operations

**TBD:** Define control-point and patch access, domain location,
geometry instance IDs, stream emission, and stream cuts.
Specify stage limits and output-state transitions.

### Compute, amplification, and mesh operations

**TBD:** Define dispatch and group IDs, mesh payload access, mesh output
counts, primitive indices, vertex outputs, primitive outputs, and dispatch.
Specify group participation and output-allocation constraints.

### Raytracing pipeline operations

**TBD:** Define tracing, callable shaders, hit reporting, hit acceptance,
termination, and ray/instance/geometry queries.
Specify payload, attribute, shader-table, and shader-kind constraints.

### Inline ray queries

**TBD:** Define allocation, initialization, traversal, candidate inspection,
commitment, abortion, and committed-result queries.
Specify the state machine and legal query states for each operation.

### Hit objects and execution reordering

**TBD:** Define hit-object construction, tracing, inspection, attribute
access, invocation, and thread reordering.
Specify object state, payload lifetime, capability gates, and convergence.

### Work graphs and node records

**TBD:** Define node handles, record allocation, record pointers, output
completion, counts, sharing completion, and recursion-level queries.
Specify record lifetime, ownership, launch mode, and synchronization.

### Vector and linear-algebra operations

**TBD:** Define vector reductions, generalized dot products, and matrix
and vector operations in a separately gated extension subsection.

Use the supplied linear-algebra template for matrix and vector operations.
Cover fill, conversion, loads, stores, element access, layout queries,
multiplication, accumulation, outer products, and vector conversion.

The complete extension needs matrix types, mangling, metadata, dimensions,
scope, layout, alignment, conversion, bounds, participation, and capability
rules before its operation definitions are usable.

The pinned DXC table includes `LinAlgMatrixGetCoordinate`.
Its summary describes column and row ordering, while the supplied snapshot
defines row in `x` and column in `y`.
Resolve that discrepancy before importing either description as a contract.
Do not infer draft stability from allocated opcodes.

### Debugging, legacy helpers, and extensions

**TBD:** Define debugging queries and break operations, legacy register
helpers, and custom extension mechanisms.
Separate compiler-internal forms from accepted final DXIL.
Specify operation-table identification and reserved opcode handling.

## Libraries and linking

**TBD:** Define exports, imports, unresolved references, function properties,
resource linkage, shader-kind propagation, and linked-module validation.
Distinguish offline linking from runtime library consumption.

The DXIL reference contains historical experimental-library text alongside
newer library profiles. That text is not a complete current library contract.

## Binary file format

### Container, program, and bitstream boundaries

A DXIL container is identified by the bytes `DXBC`.
A DXIL program resides in a part identified by `DXIL`.
That program contains a header and a bounded LLVM bitcode payload.

```text
DXIL container
  Container header
  Part offset table
  Parts
    DXIL part
      Part header
      Program header
        Shader version and program size
        DXIL bitcode header
      LLVM bitcode payload
    Signature, validation, reflection, and optional parts
```

The `DXBC` container signature does not mean the program uses DXBC
instruction encoding.

### Container and part headers

The following layouts match the [DXC header][container-header] and the
[format PR description][container-draft].
Integer fields use little-endian encoding.
Offsets and sizes below are byte counts unless stated otherwise.

| Offset | Size | Container header field |
| --- | --- | --- |
| 0 | 4 | FourCC: `DXBC` |
| 4 | 16 | Container hash |
| 20 | 2 | Container major version |
| 22 | 2 | Container minor version |
| 24 | 4 | Total container size |
| 28 | 4 | Part count |
| 32 | `4 * count` | Part offsets |

Each part offset is relative to the container start and identifies a
part header.

| Offset | Size | Part header field |
| --- | --- | --- |
| 0 | 4 | Part FourCC |
| 4 | 4 | Payload size, excluding this header |
| 8 | Payload size | Payload |

**TBD:** Specify accepted container versions, size limits, alignment,
padding, part ordering, duplicate parts, unknown parts, hashing, and
required parts per target.
Distinguish writer conventions from consumer requirements.
Do not require contiguous parts solely because a writer emits them.

### DXIL program header and payload location

Offsets in this table are relative to the start of the `DXIL` part payload.
The fixed program header occupies 24 bytes.

| Offset | Size | Program header field |
| --- | --- | --- |
| 0 | 4 | `ProgramVersion` |
| 4 | 4 | `SizeInUint32` |
| 8 | 4 | `DxilMagic`: bytes `DXIL` |
| 12 | 4 | `DxilVersion` |
| 16 | 4 | `BitcodeOffset` |
| 20 | 4 | `BitcodeSize` |

`SizeInUint32` counts 32-bit words, including the program header.
`BitcodeSize` counts bytes.
`BitcodeOffset` is relative to `DxilMagic`, not the container, part
header, or program-header start.

For a program payload beginning at byte address `P`:

```text
bitcode_start = P + 8 + BitcodeOffset
bitcode_end   = bitcode_start + BitcodeSize
```

[DXC's payload helper][bitcode-helper] uses exactly this offset base.
Consumers must interpret the offset field rather than assume the bitcode
immediately follows the fixed header.

**TBD:** Specify version-field bit allocations, padding, allowed gaps,
and every cross-check between declared sizes and enclosing part bounds.

### LLVM bitstream encoding

The [bitcode reference][bitcode] owns:

- Bit ordering and fixed-width integer encoding.
- Variable-bit-rate integer encoding.
- Blocks, records, abbreviations, arrays, and blobs.
- Module, type, constant, function, attribute, and metadata records.
- Value numbering, references, symbol tables, and function-body encoding.

**TBD:** Enumerate accepted records and DXIL-specific restrictions.
Specify the relationship between serialized record legality and the
permitted LLVM instruction subset.

The generic bitcode reference also discusses wrappers and native objects.
That discussion does not make those wrappers legal inside a `DXIL` part.

### Container part schemas

| Part | Schema responsibility |
| --- | --- |
| `DXIL` | Executable program header and bitcode |
| `ISG1`, `OSG1`, `PSG1` | Input, output, and patch/primitive signatures |
| `SFI0` | Shader feature flags |
| `PSV0` | Pipeline-state validation data |
| `RDAT` | Runtime data and reflection |
| `RTS0` | Serialized root signature |
| `STAT` | Reflection/statistics program data |
| `ILDB`, `ILDN` | Debug program and debug name |
| `HASH` | Shader hash |
| `SRCI`, `PDBI` | Source and PDB information |
| `VERS` | Compiler version information |
| `PRIV` | Private data |

These identifiers match the pinned container header.
The table does not require every part in every container.

**TBD:** Define every payload schema, presence rule, version gate, and
cross-part consistency rule. Document obsolete identifiers separately.
Keep shader hashes distinct from the container hash.

### Runtime data

RDAT has its own header, offset table, and typed subparts.
It is not an LLVM metadata block.
The [runtime reflection header][rdat-header] defines:

- `RuntimeDataHeader`: 32-bit version and 32-bit part count.
- A following array of 32-bit offsets relative to the RDAT header.
- `RuntimeDataPartHeader`: 32-bit type and 32-bit payload size.
- `RuntimeDataTableHeader`: 32-bit record count and 32-bit record stride.

RDAT part offsets, part sizes, and record strides are four-byte aligned.
Record tables use the declared stride.

**TBD:** Specify supported versions, subpart types, record schemas,
strings, index arrays, byte buffers, references, null sentinels,
record extension, and unknown-record behavior.
Use the [RDAT PR description][rdat-draft] as a navigation aid, then
verify each requirement against readers, writers, and schema definitions.

### Binary validation boundaries

**TBD:** Define malformed-input handling at each nesting level.
Cover truncated headers, integer overflow, invalid offsets, overlapping
regions, invalid record strides, and inconsistent program metadata.
Specify validation without relying on unchecked host pointer arithmetic.

## Validation and conformance

### Validation layers

The complete specification must distinguish:

- LLVM well-formedness: types, dominance, instruction structure, and SSA.
- DXIL module validity: permitted forms, metadata, capabilities, and versions.
- Operation validity: opcode, declaration, operands, and shader kind.
- Container validity: binary bounds and cross-part consistency.
- Pipeline validity: signatures, resource bindings, and stage compatibility.
- Execution requirements that static validation cannot establish.

**TBD:** Map the [existing validation rules][validation] into these layers.
State the diagnostic point for each invalid condition.
Do not classify defined exceptional arithmetic results as invalid input.

### Completed-operation conformance vectors

All hexadecimal inputs and outputs below are 32-bit bit patterns.
Tuples follow aggregate field order.
Carry and borrow tuple elements use Boolean notation.

| Operation | Inputs | Expected result |
| --- | --- | --- |
| `IMul` | `0xffffffff, 2` | `(0xffffffff, 0xfffffffe)` |
| `IMul` | `0x80000000, 0x80000000` | `(0x40000000, 0)` |
| `UMul` | `0xffffffff, 2` | `(1, 0xfffffffe)` |
| `UMul` | `0xffffffff, 0xffffffff` | `(0xfffffffe, 1)` |
| `UDiv` | `7, 3` | `(2, 1)` |
| `UDiv` | `0, 0` | `(0xffffffff, 0xffffffff)` |
| `UAddc` | `0xffffffff, 1` | `(0, true)` |
| `UAddc` | `1, 2` | `(3, false)` |
| `USubb` | `0, 1` | `(0xffffffff, true)` |
| `USubb` | `1, 1` | `(0, false)` |
| `Ibfe` | `4, 4, 0x000000f0` | `0xffffffff` |
| `Ubfe` | `4, 4, 0x000000f0` | `15` |
| `Ibfe` | `8, 28, 0x80000000` | `0xfffffff8` |
| `Ubfe` | `8, 28, 0x80000000` | `8` |
| `Ibfe` | `32, 0, 0xffffffff` | `0` |
| `Ubfe` | `33, 32, 3` | `1` |
| `Bfi` | `4, 4, 0xa, 0xffff0000` | `0xffff00a0` |
| `Bfi` | `8, 28, 0xff, 0` | `0xf0000000` |
| `Bfi` | `32, 7, 0xffffffff, 0x12345678` | `0x12345678` |

For bitfield operations, test all normalized width/offset pairs.
Include zero widths, fields ending at bit `31`, fields extending beyond
bit `31`, and ignored high operand bits.

Static negative tests must cover nonconstant opcodes, unsupported overloads,
incorrect result structures, and incorrect argument counts or types.

These vectors define expected results. They are not a claim of GPU
conformance testing.

### Aggregate-result example

This function fragment illustrates `UAddc` result extraction.
It is not a complete shader module.

```llvm
define i32 @add_low(i32 %a, i32 %b) {
entry:
  %pair = call %dx.types.i32c @dx.op.binaryWithCarryOrBorrow.i32(
    i32 44, i32 %a, i32 %b)
  %sum = extractvalue %dx.types.i32c %pair, 0
  ret i32 %sum
}
```

## DXBC behavioral correspondence

The table maps DXBC instructions to DXIL operations, not LLVM instructions.
It covers only the completed definitions.

| DXBC instruction | DXIL operation | CoreOps opcode |
| --- | --- | --- |
| `imul` | `IMul` | 41 |
| `umul` | `UMul` | 42 |
| `udiv` | `UDiv` | 43 |
| `uaddc` | `UAddc` | 44 |
| `usubb` | `USubb` | 45 |
| `ibfe` | `Ibfe` | 51 |
| `ubfe` | `Ubfe` | 52 |
| `bfi` | `Bfi` | 53 |

**TBD:** Build the remaining correspondence by behavioral category.
Record split operations, combined operations, unsupported forms, and
behavior implemented through LLVM instruction sequences.
Do not copy DXBC token layouts or register operands into DXIL declarations.

## Unresolved source discrepancies and completion work

### Rounding and floating-point tables

DXC's `Round_ne` prose states nearest-even rounding, then describes
halfway rounding away from zero.
The [DXBC definition][dxbc-round-ne] states nearest-even.
The contradictory halfway sentence cannot define a contract.

The inherited rounding tables also require review against denormal modes.
A blanket copy would obscure mode-dependent results.
Complete the rounding family only after resolving both issues.

### Aggregate terminology and omitted descriptions

The DXIL resource-operation overview describes structure-result extraction
using `extractelement`. LLVM structure results use `extractvalue`.
The completed definitions follow the actual result ABI.

The DXIL reference places borrow-description text under `UDiv`, leaving
the `USubb` entry without its full behavior.
The completed `USubb` definition uses the DXBC counterpart and DXC ABI.

### Newer forms and proposal alignment

The pinned opcode table contains reserved entries and proposed operations.
Opcode allocation is not evidence of released support.
Availability must be recorded per operation, overload, and capability.

Linear-algebra coordinate ordering requires the resolution described in its
operation-family subsection.
Other proposal-to-implementation differences remain outside this first pass.

### Completion criteria

The specification is complete only after:

- Every accepted LLVM instruction and intrinsic has explicit restrictions.
- Every accepted operation has a declaration and complete behavior.
- Every opcode has an availability or reserved-status disposition.
- Every type, metadata record, and binary part has a defined schema.
- Every memory category has bounds, lifetime, and synchronization rules.
- Every shader kind has an execution and interface contract.
- Every source conflict has an explicit resolution.
- Conformance tests cover required results and invalid forms.

The [pinned opcode table][opcode-table] is the enumeration baseline for
remaining operation coverage. It is not replaced by representative examples.

[dxil]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/docs/DXIL.rst
[langref]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/docs/LangRef.rst
[bitcode]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/docs/BitCodeFormat.rst
[d3d11]: archive/D3D11_3_FunctionalSpec.htm
[d3d-float]: archive/D3D11_3_FunctionalSpec.htm#FloatingPointRules
[linear-algebra]: ../proposals/ideas/LinearAlgebra.md
[opdb]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/utils/hct/hctdb.py
[opimpl]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/lib/DXIL/DxilOperations.cpp
[converter]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/projects/dxilconv/lib/DxbcConverter/DxbcConverter.cpp
[container-header]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/include/dxc/DxilContainer/DxilContainer.h
[rdat-header]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/include/dxc/DxilContainer/DxilRuntimeReflection.h
[format-pr]: https://github.com/microsoft/DirectXShaderCompiler/pull/8631
[container-draft]: https://github.com/microsoft/DirectXShaderCompiler/blob/ee43ad5b0068bf6677e2804fce1a3b906bb8dc20/docs/DxilContainer_Format.md
[rdat-draft]: https://github.com/microsoft/DirectXShaderCompiler/blob/ee43ad5b0068bf6677e2804fce1a3b906bb8dc20/docs/RDAT_Format.md
[result-types]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/lib/DXIL/DxilOperations.cpp#L4426-L4432
[dxil-versioning]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/docs/DXIL.rst#versioning
[instruction-list]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/docs/DXIL.rst#operations-via-instructions
[signatures]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/docs/DXIL.rst#shader-parameters-and-signatures
[resources]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/docs/DXIL.rst#shader-resources
[two-result-mapping]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/projects/dxilconv/lib/DxbcConverter/DxbcConverter.cpp#L5588-L5628
[carry-mapping]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/projects/dxilconv/lib/DxbcConverter/DxbcConverter.cpp#L5630-L5661
[bitcode-helper]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/include/dxc/DxilContainer/DxilContainer.h#L558-L564
[validation]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/docs/DXIL.rst#validation-rules
[opcode-table]: https://github.com/microsoft/DirectXShaderCompiler/blob/194bb75bea2becb42810d72a16195089c1908e10/docs/DXIL.rst#operations-via-external-functions
[dxbc-imul]: archive/D3D11_3_FunctionalSpec.htm#inst_IMUL
[dxbc-umul]: archive/D3D11_3_FunctionalSpec.htm#inst_UMUL
[dxbc-udiv]: archive/D3D11_3_FunctionalSpec.htm#inst_UDIV
[dxbc-uaddc]: archive/D3D11_3_FunctionalSpec.htm#inst_UADDC
[dxbc-usubb]: archive/D3D11_3_FunctionalSpec.htm#inst_USUBB
[dxbc-ibfe]: archive/D3D11_3_FunctionalSpec.htm#inst_IBFE
[dxbc-ubfe]: archive/D3D11_3_FunctionalSpec.htm#inst_UBFE
[dxbc-bfi]: archive/D3D11_3_FunctionalSpec.htm#inst_BFI
[dxbc-round-ne]: archive/D3D11_3_FunctionalSpec.htm#inst_ROUND_NE
