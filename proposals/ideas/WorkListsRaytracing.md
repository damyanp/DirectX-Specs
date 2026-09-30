---
title: "NNNN - Work Lists Raytracing Integration"
params:
  authors:
    - tbd: TBD
  sponsors:
    - tbd: TBD
  status: Draft
---

## Introduction

This proposal extends [D3D12 Work Lists](WorkLists.md) so a GPU-authored list
can execute multiple `DispatchRays` operations. The command list binds one
raytracing pipeline state object (RTPSO) and its shader tables, while each work
list record supplies dispatch dimensions and optional global-root arguments.
The extension uses the same program command signature and primary/secondary
record model as graphics and compute Work Lists without introducing a
raytracing program table.

## Motivation

Raytracing workloads often determine dispatch count and dimensions on the GPU.
The ordinary `DispatchRays` API requires the CPU to issue each dispatch.
`ExecuteIndirect` does not provide a standard indirect ray-dispatch operation
that composes with Work Lists' per-record argument sourcing and cross-program
dispatch model.

Applications need to:

- Produce ray-dispatch dimensions and argument records on the GPU.
- Batch many ray dispatches behind one CPU command.
- Reuse existing RTPSOs, shader tables, shader identifiers, and local root
  argument storage.
- Use the same global root argument model as compute Work Lists.
- Avoid copying RTPSO or shader-table state into every record.

The raytracing design must also preserve ordinary `DispatchRays` use of the same
RTPSO and must not alter the layout of raytracing shader records.

## Proposed solution

A raytracing-class program command signature ends with a new
`DISPATCH_RAYS_DIMENSIONS` trigger. The selected work list signature contains
exactly one such PCS.

At `SetProgram` time, the command list binds:

- The raytracing-class work list signature.
- One RTPSO.
- The ray generation, miss, hit-group, and callable shader tables.

Each work list execution supplies only `Width`, `Height`, and `Depth`, plus any
global-root arguments declared by the PCS. Local root arguments remain in
ordinary shader-table records.

Raytracing records do not carry a program-table index because the RTPSO and
shader tables are already fixed by the command-list binding.

## Detailed design

### Relationship to Core Work Lists

This proposal depends on the object, record, argument-source, synchronization,
and dispatch model in [D3D12 Work Lists](WorkLists.md).

It adds:

- One raytracing executable class.
- One dispatch-trigger argument type.
- One tagged binding variant.
- Raytracing-specific primary record headers.
- Capability, validation, API, and DDI additions.

It does not change graphics or compute record interpretation.

### Raytracing-class signatures

A PCS is raytracing-class when its dispatch trigger is
`D3D12_INDIRECT_ARGUMENT_TYPE_DISPATCH_RAYS_DIMENSIONS`.

A raytracing-class WLS:

- Contains exactly one PCS.
- Uses no program table and ignores `SubobjectMask`.
- May source global-root arguments from primary or secondary records.
- May use system or static arguments where otherwise legal.
- Cannot declare PCS arguments targeting a local root signature.
- Cannot use implicit `INLINE_ROOT_PARAMETER` or `INLINE_STATIC_SAMPLER`
  arguments.

The exactly-one-PCS rule avoids a program-table-like selector in raytracing
records. A different PCS or RTPSO is selected by a different `SetProgram`
binding, or by the multi-signature extension when available.

### Dispatch dimensions

```c++
typedef struct D3D12_DISPATCH_RAYS_DIMENSIONS
{
    UINT Width;
    UINT Height;
    UINT Depth;
} D3D12_DISPATCH_RAYS_DIMENSIONS;
```

`DISPATCH_RAYS_DIMENSIONS` is a dispatch-trigger argument. Its record payload is
the structure above, matching the dimension fields in `D3D12_DISPATCH_RAYS_DESC`.

The trigger:

- Must be the PCS's only dispatch trigger.
- May appear at any position in the PCS argument array.
- May be sourced from a primary or secondary record.
- Requires the device's Work Lists raytracing capability.
- Does not contain shader-table addresses; those are bound through
  `SetProgram`.

### RTPSO and shader-table binding

```c++
typedef struct D3D12_WORK_LIST_RAYTRACING_BINDING
{
    ID3D12StateObject*                          pRaytracingStateObject;
    D3D12_GPU_VIRTUAL_ADDRESS_RANGE             RayGenerationShaderRecord;
    D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE  MissShaderTable;
    D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE  HitGroupTable;
    D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE  CallableShaderTable;
} D3D12_WORK_LIST_RAYTRACING_BINDING;
```

`pRaytracingStateObject` must be a raytracing pipeline state object. Shader
table fields have the same meaning and alignment rules as the corresponding
fields of `D3D12_DISPATCH_RAYS_DESC`.

The binding is immutable from GPU execution of `SetProgram` until every
referencing `DispatchList` completes. This includes the RTPSO reference,
addresses, ranges, strides, and the referenced shader-table contents.

### Tagged Work Lists bindings

Core Work Lists define a tagged binding with the `PROGRAM_TABLE` variant. This
extension adds the `RAYTRACING` enum value and union member:

```c++
typedef enum D3D12_WORK_LIST_BINDING_TYPE
{
    D3D12_WORK_LIST_BINDING_TYPE_PROGRAM_TABLE = 0,
    D3D12_WORK_LIST_BINDING_TYPE_RAYTRACING    = 1,
} D3D12_WORK_LIST_BINDING_TYPE;

typedef struct D3D12_WORK_LIST_BINDING
{
    D3D12_WORK_LIST_BINDING_TYPE Type;
    union {
        D3D12_WORK_LIST_PROGRAM_TABLE_BINDING     ProgramTable;
        const D3D12_WORK_LIST_RAYTRACING_BINDING* pRaytracing;
    };
} D3D12_WORK_LIST_BINDING;
```

`D3D12_SET_WORK_LIST_DESC` is correspondingly revised:

```c++
typedef struct D3D12_SET_WORK_LIST_DESC
{
    ID3D12WorkListSignature* pSignature;
    D3D12_WORK_LIST_BINDING  Binding;
} D3D12_SET_WORK_LIST_DESC;
```

The binding type must match the WLS executable class:

- Graphics/compute WLS: `_PROGRAM_TABLE`.
- Raytracing WLS: `_RAYTRACING`.

The descriptor pointer in `pRaytracing` need remain valid only for the
synchronous `SetProgram` call. The referenced state object and GPU resources
retain normal execution-lifetime requirements.

### Raytracing record layouts

When the PCS contains any secondary-sourced argument, a primary record uses:

```c++
typedef struct D3D12_WORK_LIST_RAYTRACING_RECORD
{
    UINT                      NumSecondaryRecords;
    UINT                      ReservedPadding;
    D3D12_GPU_VIRTUAL_ADDRESS SecondaryRecords;
    // Followed by primary-sourced argument bytes.
} D3D12_WORK_LIST_RAYTRACING_RECORD;
```

`ReservedPadding` is zero and aligns the GPU virtual address. Each secondary
record executes one ray dispatch against the bound RTPSO and shader tables.

When there are no secondary-sourced arguments, the primary record has no fixed
header. Its inline argument payload starts at byte zero. There is no
`ProgramTableIndex` because raytracing-class signatures have no program table.

The dispatch input's primary stride still applies. A zero-sized inline payload
uses the normal minimum stride and alignment rules established by core Work
Lists.

### State-object associations

The PCS is represented by the same program command signature state subobject
used by core Work Lists and associated through
`D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION`.

Every raytracing shader actually invoked by a work list dispatch must have the
selected PCS associated. This applies to ray generation, miss, hit-group, and
callable shaders. Shaders present in an RTPSO or shader table but not invoked by
the dispatch are unconstrained.

The requirement is invocation-scoped because shader identifiers live in GPU
memory and the runtime cannot know at `SetProgram` which shaders will run.

### Global root signatures

Every invoked shader must use either:

- The PCS's `pGlobalRootSignature`, or
- No global root signature.

The command list uses compute root-binding state for raytracing-class Work
Lists, matching ordinary raytracing behavior. PCS global-root arguments update
that shared state per execution.

### Local root signatures

Raytracing local root signatures continue to use the standard state-object
subobject and shader-record mechanism.

PCS arguments may not target a raytracing local root signature. The implicit
local root signature argument types are also prohibited. There is no per-work-
list-record override path for shader-record local arguments, and adding one
would change ordinary raytracing shader-table layout.

Applications needing per-invocation local variation use different shader-table
records. Applications needing values sourced from Work Lists records use the
global root signature.

### Reusing one RTPSO

Associating a PCS with raytracing shaders does not make the RTPSO exclusive to
Work Lists. The same RTPSO remains usable with ordinary `DispatchRays`.

This portability is possible because the PCS:

- Does not change shader-table local root argument layout.
- Does not provide local-root overrides.
- Adds only a Work Lists dispatch layout and global-root update contract.

Ordinary `DispatchRays` ignores the PCS association.

### Application flow

An application:

1. Creates a raytracing-class PCS ending in `DISPATCH_RAYS_DIMENSIONS`.
2. Creates a WLS containing that one PCS.
3. Associates the PCS with all RTPSO shaders that may be invoked by the work
   list dispatch.
4. Builds ordinary raytracing shader tables.
5. Populates Work Lists primary and optional secondary records with dimensions
   and global-root arguments.
6. Binds the WLS, RTPSO, and shader tables using the tagged raytracing binding.
7. Calls `DispatchList`.

The same RTPSO may be bound conventionally for unrelated `DispatchRays` calls.

### Interaction with continuation chains

If the separate
[Work List Continuations and Multi-Signature Dispatch](WorkListContinuations.md)
proposal is supported, each signature-array slot may pair a raytracing-class WLS
with an independent RTPSO and shader-table binding. `SignatureIndex` selects the
signature and binding together.

The signature array's global-root compatibility rules still apply across
graphics, compute, and raytracing entries.

### D3D API additions

#### Capability

The Work Lists feature data gains:

```c++
typedef struct D3D12_FEATURE_DATA_WORK_LISTS
{
    D3D12_WORK_LISTS_TIER Tier;
    BOOL                  DispatchRaysSupported;
} D3D12_FEATURE_DATA_WORK_LISTS;
```

`DispatchRaysSupported` is orthogonal to the core tier. A device may implement
core Work Lists without this proposal.

#### Argument type

`D3D12_INDIRECT_ARGUMENT_TYPE` gains
`D3D12_INDIRECT_ARGUMENT_TYPE_DISPATCH_RAYS_DIMENSIONS`.

The per-argument descriptor carries no additional static payload; the selected
record source contains `D3D12_DISPATCH_RAYS_DIMENSIONS`.

#### Binding types

The tagged binding types and the revised `D3D12_SET_WORK_LIST_DESC` are as
defined above. Existing graphics and compute applications use the
`PROGRAM_TABLE` variant.

#### Record types

`D3D12_WORK_LIST_RAYTRACING_RECORD` is added as the hybrid raytracing primary
header. The fully inline raytracing record is represented by its payload and
does not require a public C structure.

### Validation

Creation-time validation rejects:

- A raytracing PCS when `DispatchRaysSupported` is false.
- More or fewer than one PCS in a raytracing-class WLS.
- A mixture of executable classes.
- Any local-root-bound PCS argument.
- `INLINE_ROOT_PARAMETER` or `INLINE_STATIC_SAMPLER`.
- A duplicate ray dispatch trigger.
- Raytracing-incompatible argument types.

The debug layer reports:

- A non-RTPSO in the raytracing binding.
- Binding type and WLS class mismatch.
- Invalid shader-table ranges or strides detectable at recording time.
- Mutation of bound state during its in-flight lifetime where observable.

Execution is invalid if an invoked shader:

- Lacks the selected PCS association.
- Uses an incompatible PCS.
- Uses a different non-null global root signature.

These conditions require GPU-based validation because the deciding shader
identifiers and control flow are GPU-resident.

### DDI changes

The DDI mirrors:

- `DispatchRaysSupported`.
- The dispatch-dimensions argument type and 12-byte payload.
- `D3D12DDI_WORK_LIST_BINDING_TYPE_RAYTRACING`.
- `D3D12DDI_WORK_LIST_RAYTRACING_BINDING`.
- Raytracing primary-record byte layouts.

The runtime forwards the tagged binding through existing `SetProgram` plumbing.
No new dispatch DDI entry point is required.

### WARP support

WARP should support raytracing-class Work Lists whenever it reports the
underlying raytracing capabilities needed by the bound RTPSO. It may execute
dispatches serially and may report `DispatchRaysSupported == FALSE` until the
implementation is complete.

### PIX support

PIX must display the bound RTPSO and shader tables, decode raytracing primary and
secondary records, show each dispatch's dimensions, and identify the shader
records actually invoked where capture data permits.

### Other tooling impact

GPU-based validation must observe invoked shader identifiers rather than merely
scan shader tables. DRED and GPU dumps should record the active RTPSO, PCS,
shader-table addresses, and Work Lists record addresses. Header and DDI tooling
must preserve the 12-byte dimensions payload and record layouts.

### Dependencies and risks

This proposal depends on core Work Lists and D3D12 raytracing.

Risks include:

- Driver specialization requiring an explicit PCS selector on ordinary
  `DispatchRays`, which the current design does not provide.
- Shader-table mutation or lifetime errors hidden behind GPU-authored control
  flow.
- Divergent global-root-signature associations among invoked shaders.
- Demand for higher-frequency shader-table selection than the command-list
  binding provides.

## Testing

### API and validation tests

- Query supported and unsupported capability combinations.
- Create legal and illegal raytracing PCSes.
- Reject local-root-bound and implicit-local-root arguments.
- Validate tagged binding class matching and RTPSO type.
- Validate shader-table address, range, and stride requirements.
- Validate hybrid and inline raytracing record layouts.

### Functional tests

- Execute dimensions from primary records.
- Execute dimensions from secondary records with zero, one, and many entries.
- Mix dimensions with global root constants, descriptors, and tables.
- Invoke ray generation, miss, hit-group, and callable shaders carrying the PCS
  association.
- Use the same RTPSO successfully with ordinary `DispatchRays`.
- Exercise multiple independent raytracing bindings through continuation arrays
  when that proposal is supported.

### Tooling and conformance tests

- Verify PIX capture and replay of Work Lists ray dispatches.
- Verify WARP behavior when capability is reported.
- Verify GPU-based validation catches missing PCS and root-signature
  associations.
- Verify API and DDI structure layouts match.

## Alternatives considered

### Put RTPSO and shader tables in every record

This would greatly increase record size and bandwidth, complicate lifetime
tracking, and duplicate data shared across many dispatches.

### Use a raytracing program table

Raytracing already selects shaders through shader-table identifiers. A second
program table would duplicate that selection mechanism and create unclear local
root argument ownership.

### Allow per-record local root overrides

This would require rewriting or supplementing shader-table records and would
make the same RTPSO behave differently under Work Lists. Keeping local
arguments in shader records preserves existing raytracing semantics.

### Make PCS-associated RTPSOs exclusive to Work Lists

No structural requirement forces exclusivity because the PCS does not alter
shader-table layout. Allowing ordinary `DispatchRays` avoids duplicate RTPSOs.

### Require raytracing support in core Work Lists

Raytracing requires distinct hardware and driver capability. An orthogonal cap
allows graphics/compute Work Lists to ship independently.

## Stakeholders

- Direct3D Work Lists, raytracing, state-object, and debug-layer owners.
- Hardware vendors and raytracing compiler owners.
- WARP raytracing owners.
- PIX, DRED, GPU dump, and GPU-based validation owners.
- Engine developers using GPU-driven ray dispatch.

## Prior work

- [D3D12 Work Lists](WorkLists.md)
- [DirectX Raytracing](../../d3d/Raytracing.md)
- [`ExecuteIndirect`](../../d3d/IndirectDrawing.md)

## Open questions

- Do any implementations require the ordinary `DispatchRays` path to identify
  a PCS explicitly for compilation or execution?
- Should selected shader-table fields vary per primary record in a future tier?
- Should raytracing support remain an orthogonal capability or become mandatory
  for a future Work Lists baseline?
- What additional GPU-validation information is required to diagnose the exact
  invoked shader association that failed?
