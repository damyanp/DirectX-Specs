---
title: "NNNN - D3D12 Work Lists"
params:
  authors:
    - tbd: TBD
  sponsors:
    - tbd: TBD
  status: Draft
---

## Introduction

This proposal adds D3D12 Work Lists, a GPU-driven dispatch model that can select
a different graphics or compute program for each record in GPU memory. A single
CPU command can execute records across many programs without CPU readback,
per-program command submission, or worst-case empty `ExecuteIndirect` calls.
Programs may use different indirect-argument layouts, and applications may
factor bindings shared by many executions into a primary record while keeping
per-execution values in secondary records.

This proposal defines the graphics and compute baseline. Optional raytracing,
multi-signature continuation, and GPU-timeline validation facilities are
described by separate proposals:

- [Work Lists Raytracing Integration](WorkListsRaytracing.md)
- [Work List Continuations and Multi-Signature Dispatch](WorkListContinuations.md)
- [Work Lists GPU Timeline Validation Hooks](WorkListsGpuTimelineValidation.md)

## Motivation

`ExecuteIndirect` cannot select a pipeline state object or generic program from
GPU memory. An application whose active programs are known only on the GPU must
issue one indirect call for every possible program and arrange for inactive
calls to execute zero records. This is functionally correct, but large program
sets produce many empty calls, repeated CPU work, and implementation overhead
that can starve the GPU.

The workaround also imposes one indirect-command layout on each call. Programs
that update different root parameters therefore require separate command
signatures and separate calls, even when their records are produced by the same
GPU pass. Values shared by many executions, such as material bindings, must
either be duplicated in every record or managed through application-specific
indirection.

GPU-driven rendering needs one dispatch primitive with the following
properties:

- The GPU selects a program for each batch of records.
- Different programs may use different per-execution argument layouts.
- Shared and per-execution bindings can be stored at different frequencies.
- All record counts, program indices, and argument bytes can be produced on the
  GPU without CPU readback.
- The driver sees each program's argument layout while compiling the program,
  allowing implementation-specific specialization rather than requiring a
  generic interpreter.

## Proposed solution

Work Lists introduce three creation-time objects and two command-list
operations:

1. An `ID3D12ProgramCommandSignature` describes one program's indirect argument
   layout, including where each value comes from and which global or local root
   parameter it updates.
2. An `ID3D12WorkListSignature` groups the program command signatures that may
   participate in one dispatch and declares which pipeline subobjects may vary
   among their programs.
3. Generic programs in state objects associate with one program command
   signature at state-object creation time.
4. `SetProgram` binds the work list signature and a GPU-resident program table.
5. `DispatchList` reads a GPU-resident list header and primary records. Each
   primary record selects a program-table entry and either executes once or
   points to a secondary list containing multiple executions.

A program table slot contains a `D3D12_PROGRAM_IDENTIFIER` and optional local
root arguments. A primary record contains the selected slot index, optional
secondary-list metadata, and primary-sourced argument bytes. Secondary records
contain the values that vary per execution. This produces a transparent,
application-authored memory model: CPU upload, copies, or shaders may populate
every input consumed by the dispatch.

For example, a compute culling pass can write:

```text
dispatch input
  NumProgramInputs = 2
  ProgramInputs -> [
    { ProgramTableIndex = 4, NumSecondaryRecords = 120, ... },
    { ProgramTableIndex = 9, NumSecondaryRecords = 37,  ... }
  ]
```

The first program may source a material descriptor from its primary record and
draw arguments from secondary records. The second may use a different local
root signature and a different secondary-record stride. `DispatchList` executes
both without the CPU knowing either count.

## Detailed design

### Terms

- **Program command signature (PCS):** The per-program indirect argument layout
  represented by `ID3D12ProgramCommandSignature`.
- **Work list signature (WLS):** The per-list container represented by
  `ID3D12WorkListSignature`. It references one or more compatible PCSes.
- **Program:** A graphics or compute generic program declared in an
  `ID3D12StateObject`. Free-standing `ID3D12PipelineState` objects are not
  eligible.
- **Program table:** A GPU-resident array mapping integer indices to program
  identifiers and optional program-table-record local root arguments.
- **Primary list:** The array selected by
  `D3D12_DISPATCH_LIST_INPUT::ProgramInputs`.
- **Primary record:** One entry in the primary list. It selects a program and
  either executes once or points to a secondary list.
- **Secondary list:** An optional array of per-execution records associated with
  one primary record.
- **Executable class:** Graphics-class for `DRAW`, `DRAW_INDEXED`, and
  `DISPATCH_MESH`; compute-class for `DISPATCH` and `FIXED_DISPATCH`. One WLS
  contains one executable class.

### Object and data model

Creation-time objects form this graph:

```text
ID3D12WorkListSignature
  +- SubobjectMask
  +- ID3D12ProgramCommandSignature[...]
       +- argument descriptors
       +- secondary-record stride
       +- optional shared global root signature
       +- optional per-PCS local root signature

ID3D12StateObject
  +- generic program A -> associated PCS A
  +- generic program B -> associated PCS B
```

At command-list execution time:

```text
SetProgram
  +- bound WLS
  +- bound program table

DispatchList
  +- D3D12_DISPATCH_LIST_INPUT
       +- NumProgramInputs
       +- Flags
       +- ProgramInputs { StartAddress, StrideInBytes }
            +- primary record 0 -> table slot -> optional secondary list
            +- primary record 1 -> table slot -> optional secondary list
```

The WLS and its PCSes are immutable device-child objects. Program-table
contents, dispatch inputs, primary records, and secondary records are
application-managed GPU memory.

### Workflow

An application:

1. Creates one PCS for each distinct program argument layout.
2. Creates a WLS containing the PCSes that can be selected together.
3. Creates state-object generic programs and associates each participating
   shader export with one of those PCSes.
4. Obtains each generic program's `D3D12_PROGRAM_IDENTIFIER`.
5. Populates a program table with identifiers and optional local root arguments.
6. Populates dispatch input, primary records, and optional secondary records.
7. Binds the WLS and program table through `SetProgram`.
8. Calls `DispatchList`.

The application may repeat steps 5 and 6 entirely on the GPU. Program table
memory is immutable while a referencing dispatch is in flight.

### Dispatch model

Each primary record selects one program-table slot.

- If the selected PCS has no secondary-sourced arguments, the primary record
  drives one execution.
- If the selected PCS has any secondary-sourced arguments, the primary record
  names a secondary list and count. Each secondary record drives one execution
  of the selected program.

Graphics records without
`D3D12_DISPATCH_LIST_FLAG_ALLOW_OUT_OF_ORDER_GRAPHICS` execute and retire in
record order. Setting the flag permits graphics records to launch and retire
out of order. Compute records have no record-order guarantee.

`MaxGraphicsProgramInputsPerPrimaryList` is a CPU-provided upper bound used for
implementation resource sizing. It must be at least the graphics primary-record
count read from GPU memory. Applications pass zero for compute-only dispatches.

### Program command signatures

A PCS declares:

- `pArgumentDescs`: ordered indirect argument descriptors.
- `SecondaryRecordByteStride`: the stride used for secondary records selected
  by this PCS.
- `pGlobalRootSignature`: an optional global root signature shared by every PCS
  in the parent WLS.

Each argument descriptor has three independent properties:

- `Type`: what value is supplied.
- `Source`: where its bytes come from.
- `Binding`: which root-signature namespace it updates.

The driver receives the PCS while compiling associated programs. It can
specialize fixed offsets and binding behavior into the compiled program rather
than interpreting a generic record format at execution time.

### Argument sources

`D3D12_INDIRECT_ARGUMENT_SOURCE` defines:

```c++
typedef enum D3D12_INDIRECT_ARGUMENT_SOURCE
{
    D3D12_INDIRECT_ARGUMENT_SOURCE_PRIMARY_RECORD       = 0,
    D3D12_INDIRECT_ARGUMENT_SOURCE_SECONDARY_RECORD     = 1,
    D3D12_INDIRECT_ARGUMENT_SOURCE_PROGRAM_TABLE_RECORD = 2,
    D3D12_INDIRECT_ARGUMENT_SOURCE_SYSTEM               = 3,
    D3D12_INDIRECT_ARGUMENT_SOURCE_STATIC               = 4,
} D3D12_INDIRECT_ARGUMENT_SOURCE;
```

- `_PRIMARY_RECORD` reads bytes from the primary record's inline payload.
- `_SECONDARY_RECORD` reads bytes from each secondary record.
- `_PROGRAM_TABLE_RECORD` reads local-root-argument bytes following the program
  identifier in the selected program-table slot.
- `_SYSTEM` supplies a value generated by the implementation.
- `_STATIC` stores the value in the argument descriptor and contributes no
  record bytes.

The source changes value frequency, not binding semantics. Every argument is
resolved for every execution.

### Argument bindings

```c++
typedef enum D3D12_INDIRECT_ARGUMENT_BINDING
{
    D3D12_INDIRECT_ARGUMENT_BINDING_GLOBAL_ROOT_SIGNATURE = 0,
    D3D12_INDIRECT_ARGUMENT_BINDING_LOCAL_ROOT_SIGNATURE  = 1,
} D3D12_INDIRECT_ARGUMENT_BINDING;
```

Global arguments update the shared global root signature. Local arguments
update the PCS-specific local root signature. Dispatch-trigger and
input-assembler arguments ignore `Binding`.

Within one WLS:

- Every PCS uses the same global root signature pointer when non-null.
- PCSes of the same executable class update the same set of global root
  parameter indices.
- Graphics PCSes touch the same set of vertex-buffer slots and agree on whether
  an index-buffer view is present.
- Local root signatures and local parameter sets may differ per PCS.

These constraints keep command-list global state well-defined while preserving
per-program specialization.

### Supported argument types

Work Lists support the existing `ExecuteIndirect` argument types where
applicable:

- `DRAW`
- `DRAW_INDEXED`
- `DISPATCH`
- `DISPATCH_MESH`
- `VERTEX_BUFFER_VIEW`
- `INDEX_BUFFER_VIEW`
- `CONSTANT`
- `CONSTANT_BUFFER_VIEW`
- `SHADER_RESOURCE_VIEW`
- `UNORDERED_ACCESS_VIEW`
- `INCREMENTING_CONSTANT`

They add:

- `DESCRIPTOR_TABLE`, which supplies a GPU descriptor handle.
- `FIXED_DISPATCH`, which stores fixed compute dispatch dimensions in the PCS
  and contributes no record bytes.
- `INLINE_ROOT_PARAMETER`, which declares an implicit local root parameter and
  its source.
- `INLINE_STATIC_SAMPLER`, which declares an implicit local static sampler.

The raytracing and GPU-validation proposals add further argument types.

Exactly one dispatch-trigger argument is required. Its position in the argument
array is unconstrained. Its type determines the executable class.

### Work-Lists-specific argument details

`INCREMENTING_CONSTANT` uses `Source == SYSTEM` and contributes no record
bytes:

```c++
typedef enum D3D12_INCREMENTING_CONSTANT_FLAGS
{
    D3D12_INCREMENTING_CONSTANT_FLAG_NONE                     = 0,
    D3D12_INCREMENTING_CONSTANT_FLAG_RESET_PER_SECONDARY_LIST = 0x1,
} D3D12_INCREMENTING_CONSTANT_FLAGS;

struct
{
    UINT                              RootParameterIndex;
    UINT                              DestOffsetIn32BitValues;
    D3D12_INCREMENTING_CONSTANT_FLAGS Flags;
} IncrementingConstant;
```

Without the flag, the value starts at zero for each list and increases once per
execution across primary-record boundaries. With
`RESET_PER_SECONDARY_LIST`, it restarts at zero for each primary record's
secondary list. The flag has no effect on an inline-only PCS. A WLS may contain
at most one incrementing constant. Numbering follows logical record order and
remains deterministic when graphics execution is allowed to run out of order.

`DESCRIPTOR_TABLE` stores one 8-byte `D3D12_GPU_DESCRIPTOR_HANDLE` in the
selected record source:

```c++
struct
{
    UINT RootParameterIndex;
} DescriptorTable;
```

The handle must address the currently bound heap of the root parameter's
descriptor-heap type. Local descriptor tables use the same representation.

`FIXED_DISPATCH` uses `Source == STATIC`, contributes no record bytes, and
declares:

```c++
struct
{
    UINT ThreadGroupCountX;
    UINT ThreadGroupCountY;
    UINT ThreadGroupCountZ;
} FixedDispatch;
```

Each dimension must be at least one and no greater than the device's applicable
dispatch dimension limit. The runtime validates the fixed grid at PCS creation.

`INLINE_ROOT_PARAMETER` wraps a complete `D3D12_ROOT_PARAMETER1`:

```c++
typedef struct D3D12_WORK_LIST_INLINE_ROOT_PARAMETER
{
    D3D12_ROOT_PARAMETER1 RootParameter;
    UINT                  DestOffsetIn32BitValues;
    UINT                  Num32BitValuesToSet;
} D3D12_WORK_LIST_INLINE_ROOT_PARAMETER;
```

Its binding must be `LOCAL_ROOT_SIGNATURE`. Primary, secondary, and system
sources override the corresponding local argument per execution.
`PROGRAM_TABLE_RECORD` stores the declared constants subrange or non-constant
slot in the program-table tail. Multiple inline arguments may compose
non-overlapping subranges of one 32-bit constants slot, but other parameter
types may appear only once. Local root signature shader visibility is `ALL`.

`INLINE_STATIC_SAMPLER` wraps one `D3D12_STATIC_SAMPLER_DESC1`, uses
`Binding == LOCAL_ROOT_SIGNATURE` and `Source == STATIC`, and contributes no
record bytes.

### Record byte layout

Arguments appear in PCS order in each source's payload. Each argument starts at
its natural alignment:

- DWORD values and draw/dispatch structures: 4 bytes.
- GPU virtual addresses, descriptor handles, and root descriptors: 8 bytes.
- Input-assembler views: their D3D12 structure alignment.

Each payload's minimum size is the aligned sum of the arguments sourced from
that payload. Non-zero application-provided strides must be at least the
minimum size and satisfy the required alignment. A zero secondary-record stride
is a broadcast form in which every secondary index resolves to the same bytes.

Unlike `ExecuteIndirect`, the primary-record stride is a dispatch-time property
and the secondary-record stride is a PCS property. Different selected programs
may therefore interpret different secondary layouts in the same dispatch.

### Primary records

When a PCS has secondary-sourced arguments:

```c++
typedef struct D3D12_WORK_LIST_PRIMARY_RECORD
{
    UINT                      ProgramTableIndex;
    UINT                      NumSecondaryRecords;
    D3D12_GPU_VIRTUAL_ADDRESS SecondaryRecords;
    // Followed by primary-sourced argument bytes.
} D3D12_WORK_LIST_PRIMARY_RECORD;
```

When it has no secondary-sourced arguments:

```c++
typedef struct D3D12_WORK_LIST_INLINE_PRIMARY_RECORD
{
    UINT ProgramTableIndex;
    // Followed by primary-sourced argument bytes.
} D3D12_WORK_LIST_INLINE_PRIMARY_RECORD;
```

`ProgramTableIndex` must be less than the bound program table's `SlotCount`.
`SecondaryRecords` must be 8-byte aligned and shader-resource accessible when
`NumSecondaryRecords` is non-zero.

A single primary-list stride must accommodate the largest primary record that
may be selected by the bound WLS.

### Program tables

A program-table record is:

```text
D3D12_PROGRAM_IDENTIFIER (32 bytes)
optional local-root-argument bytes
```

The table is a transparent GPU buffer. There is no update API and no opaque
per-slot runtime data. Programs may come from different state objects if every
program:

- Has an associated PCS contained by the bound WLS.
- Has the correct executable class.
- Uses the shared global root signature.
- Differs from other reachable programs only in subobjects allowed by the WLS
  `SubobjectMask`.
- Fits its program-table-record local root arguments in the bound slot stride.

`SubobjectMask` may permit variation in shader stages and ordinary graphics
pipeline subobjects such as stream output, blend, sample mask, rasterizer,
depth/stencil, input layout, strip-cut value, primitive topology, sample
description, and view instancing. It does not permit variation in state that
must remain uniform, including the global root signature.

The table binding contains both a physical byte range and a logical slot count:

```c++
typedef struct D3D12_WORK_LIST_PROGRAM_TABLE_BINDING
{
    D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE Table;
    UINT                                       SlotCount;
} D3D12_WORK_LIST_PROGRAM_TABLE_BINDING;
```

`Table.StartAddress` is 8-byte aligned. A non-zero stride is 8-byte aligned, at
least 32 bytes, and no larger than
`D3D12_PROGRAM_TABLE_MAX_BYTE_STRIDE`. A zero stride broadcasts one physical
record to every logical index.

The table contents are immutable from GPU execution of `SetProgram` until every
referencing `DispatchList` completes.

### State-object integration

Participating graphics and compute programs are generic programs declared in
an `ID3D12StateObject`. A PCS is represented by a state subobject and associated
with shader exports through `D3D12_SUBOBJECT_TO_EXPORTS_ASSOCIATION`.

```c++
typedef struct D3D12_PROGRAM_COMMAND_SIGNATURE
{
    ID3D12ProgramCommandSignature* pProgramCommandSignature;
} D3D12_PROGRAM_COMMAND_SIGNATURE;
```

A generic program with a PCS association is exclusive to Work Lists. It cannot
also be bound as an ordinary generic pipeline or used as a Work Graph node. An
application needing both paths declares two generic programs, one with the PCS
association and one without.

For partial graphics programs, the PCS association is a shader compile-time
input and must be attached when the partial program is constructed. It cannot
be changed at the final link step.

### Local root signatures

Each PCS may use one local root signature. Two mutually exclusive authoring
paths are available:

1. **Implicit:** graphics and compute PCSes may use
   `INLINE_ROOT_PARAMETER` and `INLINE_STATIC_SAMPLER` arguments. The runtime
   synthesizes a local root signature and injects it into the state object for
   the shaders associated with the PCS.
2. **Explicit:** the application supplies a
   `D3D12_LOCAL_ROOT_SIGNATURE` state subobject and associates it with shader
   exports. Conventional root-binding arguments identify its parameter slots.

For an implicit local root signature, the runtime exposes the synthesized object
through:

```c++
interface ID3D12ProgramCommandSignature : ID3D12DeviceChild
{
    HRESULT GetSynthesizedLocalRootSignature(
        REFIID  riid,
        void**  ppLocalRootSignature);

    const D3D12_PROGRAM_COMMAND_SIGNATURE_DESC* GetDesc();
};
```

Local root argument values not overridden by primary or secondary arguments are
stored after the program identifier in the program-table record. Root
descriptors and descriptor handles are 8-byte aligned and 8 bytes wide. Root
constants are packed DWORD arrays. Per-record-overridden fields consume no
program-table tail space.

### State leakage and reset

Bindings updated by one execution are not inputs to another execution unless
the PCS explicitly sources them for that execution. Implementations must not
make correctness depend on record scheduling or stale state.

At the end of `DispatchList`, touched vertex and index buffer bindings reset to
null, touched root constants reset to zero, and touched root views and
descriptor tables reset to null in the applicable graphics or compute binding
namespace. This matches `ExecuteIndirect`. State is not reset between lists in
a continuation chain. Whether a future revision should restore pre-dispatch
state instead remains an open question.

### Execution order and command-list state

Work Lists may update only state represented by their argument descriptors.
Render targets, viewports, scissors, and other command-list state remain fixed
for the dispatch.

Graphics root bindings and compute root bindings remain separate. Graphics
records consume graphics root state; compute records consume compute root
state. Input-assembler arguments apply only to graphics-class signatures.

Predication is sampled once and applies to the complete `DispatchList` call.
Bundles cannot record
`SetProgram` or `DispatchList`. Direct command lists support every executable
class. Compute command lists support calls whose selected signatures are all
compute-class or, when the raytracing proposal is present, raytracing-class.
Copy and video command lists do not support Work Lists. Work Lists may execute
inside a render pass only when every selected signature is graphics-class and
every argument is legal in that render pass. Queries and counters observe the
complete call; Work Lists do not add per-record query granularity.

### Resource states and synchronization

Dispatch headers, primary lists, secondary lists, and program tables must be
accessible through `D3D12_BARRIER_ACCESS_COMMON` or
`D3D12_BARRIER_ACCESS_SHADER_RESOURCE`, or the equivalent legacy
`D3D12_RESOURCE_STATE_COMMON` or
`D3D12_RESOURCE_STATE_NON_PIXEL_SHADER_RESOURCE`. Producer shaders must
complete writes and establish the appropriate UAV or enhanced-barrier ordering
before `DispatchList` consumes those buffers.

Program-table contents are additionally covered by the binding immutability
window. Other record buffers may be reused after the dispatch that reads them
has completed.

### Object lifetime

A WLS retains references to its PCSes, and each PCS retains its global root
signature and synthesized local root signature. State objects that own program
identifiers are not retained by a WLS or by a program-table binding.
Applications must keep those state objects alive until every dispatch that may
read their identifiers has completed.

The command-list binding does not extend the CPU lifetime of application
descriptor memory passed to `SetProgram`; implementations consume that
descriptor synchronously. Normal GPU execution lifetimes still apply to the
bound WLS, program-table resource, and state objects.

### D3D API additions

#### Capability query

```c++
typedef enum D3D12_WORK_LISTS_TIER
{
    D3D12_WORK_LISTS_TIER_NOT_SUPPORTED = 0,
    D3D12_WORK_LISTS_TIER_1             = 10,
} D3D12_WORK_LISTS_TIER;

typedef struct D3D12_FEATURE_DATA_WORK_LISTS
{
    D3D12_WORK_LISTS_TIER Tier;
} D3D12_FEATURE_DATA_WORK_LISTS;
```

The extension proposals may add orthogonal capability fields or higher tiers.

#### Program command signature

```c++
typedef struct D3D12_PROGRAM_COMMAND_SIGNATURE_DESC
{
    UINT                                  NumArgumentDescs;
    const D3D12_WORK_LIST_ARGUMENT_DESC*  pArgumentDescs;
    UINT                                  SecondaryRecordByteStride;
    ID3D12RootSignature*                  pGlobalRootSignature;
} D3D12_PROGRAM_COMMAND_SIGNATURE_DESC;

HRESULT ID3D12DeviceN::CreateProgramCommandSignature(
    const D3D12_PROGRAM_COMMAND_SIGNATURE_DESC* pDesc,
    REFIID                                       riid,
    void**                                       ppCommandSignature);
```

#### Work list signature

```c++
typedef struct D3D12_WORK_LIST_SIGNATURE_DESC
{
    UINT                                  NumProgramCommandSignatures;
    ID3D12ProgramCommandSignature* const* pProgramCommandSignatures;
    D3D12_PIPELINE_STATE_SUBOBJECT_MASK   SubobjectMask;
} D3D12_WORK_LIST_SIGNATURE_DESC;

HRESULT ID3D12DeviceN::CreateWorkListSignature(
    const D3D12_WORK_LIST_SIGNATURE_DESC* pDesc,
    REFIID                                riid,
    void**                                ppSignature);
```

#### Argument descriptor

```c++
typedef struct D3D12_WORK_LIST_ARGUMENT_DESC
{
    D3D12_INDIRECT_ARGUMENT_TYPE    Type;
    D3D12_INDIRECT_ARGUMENT_SOURCE  Source;
    D3D12_INDIRECT_ARGUMENT_BINDING Binding;
    union {
        // Existing ExecuteIndirect payloads and Work Lists additions.
    };
} D3D12_WORK_LIST_ARGUMENT_DESC;
```

#### Binding and dispatch

```c++
typedef enum D3D12_WORK_LIST_BINDING_TYPE
{
    D3D12_WORK_LIST_BINDING_TYPE_PROGRAM_TABLE = 0,
} D3D12_WORK_LIST_BINDING_TYPE;

typedef struct D3D12_WORK_LIST_BINDING
{
    D3D12_WORK_LIST_BINDING_TYPE Type;
    union {
        D3D12_WORK_LIST_PROGRAM_TABLE_BINDING ProgramTable;
    };
} D3D12_WORK_LIST_BINDING;

typedef struct D3D12_SET_WORK_LIST_DESC
{
    ID3D12WorkListSignature* pSignature;
    D3D12_WORK_LIST_BINDING  Binding;
} D3D12_SET_WORK_LIST_DESC;

typedef struct D3D12_DISPATCH_LIST_INPUT
{
    UINT                                  NumProgramInputs;
    D3D12_DISPATCH_LIST_FLAGS             Flags;
    D3D12_GPU_VIRTUAL_ADDRESS_AND_STRIDE  ProgramInputs;
} D3D12_DISPATCH_LIST_INPUT;

void ID3D12GraphicsCommandListN::DispatchList(
    D3D12_GPU_VIRTUAL_ADDRESS DispatchListInput,
    UINT                      MaxGraphicsProgramInputsPerPrimaryList);
```

`D3D12_PROGRAM_TYPE` gains `D3D12_PROGRAM_TYPE_WORK_LIST`. Passing that type to
`SetProgram` selects `D3D12_SET_WORK_LIST_DESC`. Core Work Lists requires
`Binding.Type == D3D12_WORK_LIST_BINDING_TYPE_PROGRAM_TABLE`; extension
proposals may add binding variants.

```c++
typedef enum D3D12_DISPATCH_LIST_FLAGS
{
    D3D12_DISPATCH_LIST_FLAG_NONE                         = 0,
    D3D12_DISPATCH_LIST_FLAG_ALLOW_OUT_OF_ORDER_GRAPHICS = 0x1,
} D3D12_DISPATCH_LIST_FLAGS;
```

#### Interfaces and constants

```c++
interface ID3D12ProgramCommandSignature : ID3D12DeviceChild
{
    HRESULT GetSynthesizedLocalRootSignature(
        REFIID  riid,
        void**  ppLocalRootSignature);

    const D3D12_PROGRAM_COMMAND_SIGNATURE_DESC* GetDesc();
};

interface ID3D12WorkListSignature : ID3D12DeviceChild
{
};

#define D3D12_PROGRAM_TABLE_MAX_BYTE_STRIDE 4096
```

`GetSynthesizedLocalRootSignature` returns an AddRef'd interface for the
identity-stable synthesized local root signature. It returns `S_FALSE` and a
null output when the PCS uses the explicit path or has no local root signature.
`GetDesc` returns immutable creation data valid for the PCS lifetime.

### Validation

Object creation validates all information available from descriptors and object
identity:

- A PCS has exactly one dispatch trigger; its position is unconstrained.
- Argument types, sources, bindings, and alignments form a legal layout.
- Secondary stride is zero when no secondary-sourced arguments exist and is
  large enough otherwise.
- Implicit and explicit local root signature paths are not mixed.
- Every PCS in a WLS has one executable class, and the WLS does not mix classes.
- PCSes in a WLS share one global root signature and compatible global/IA
  update sets.
- The subobject mask contains only permitted varying subobjects.

State-object creation validates PCS associations, shader class, root signature
compatibility, and local root signature associations.

The debug layer validates `SetProgram` and `DispatchList` recording-time
requirements, including binding type, alignment, resource ranges, and
CPU-provided upper bounds where possible.

The runtime cannot validate bytes produced later on the GPU. Out-of-range
program indices, invalid identifiers, incompatible program-table contents, and
undersized local-root-argument storage are undefined behavior without GPU-based
validation. The separate
[GPU Timeline Validation Hooks](WorkListsGpuTimelineValidation.md) proposal
provides a mechanism for execution-time diagnosis.

### DDI changes

The DDI mirrors creation and dispatch:

```c++
typedef HRESULT (APIENTRY* PFND3D12DDI_CREATEPROGRAMCOMMANDSIGNATURE)(
    D3D12DDI_HDEVICE                               hDevice,
    const D3D12DDI_PROGRAM_COMMAND_SIGNATURE_DESC* pDesc,
    D3D12DDI_HPROGRAMCOMMANDSIGNATURE              hSignature);

typedef HRESULT (APIENTRY* PFND3D12DDI_CREATEWORKLISTSIGNATURE)(
    D3D12DDI_HDEVICE                         hDevice,
    const D3D12DDI_WORK_LIST_SIGNATURE_DESC* pDesc,
    D3D12DDI_HWORKLISTSIGNATURE              hSignature);

typedef void (APIENTRY* PFND3D12DDI_DISPATCHLIST)(
    D3D12DDI_HCOMMANDLIST         hCommandList,
    D3D12DDI_GPU_VIRTUAL_ADDRESS DispatchListInput,
    UINT                          MaxGraphicsProgramInputsPerPrimaryList);
```

The runtime forwards the bound WLS handle and program-table binding through
`SetProgram`. API and DDI GPU-resident structures have identical byte layouts.

When the implicit local root signature path is used, the runtime creates the
root signature and injects it through existing state-object DDI mechanisms.
The driver receives the `_INLINE_*` argument descriptors to describe byte
sourcing, not to request root signature construction.

The Work Lists DDI capability reports Tier 1 support.

### WARP support

WARP should implement the complete baseline, including mixed graphics programs,
mixed compute programs, primary/secondary records, local root signatures,
program-table broadcast stride, and out-of-order flag acceptance. WARP may
execute records serially; the flag permits reordering but does not require it.

### PIX support

PIX must:

- Decode WLS and PCS objects and their argument layouts.
- Show the bound program table and selected identifier for each primary record.
- Decode primary and secondary payloads using the selected PCS.
- Attribute generated draws and dispatches to their source records.
- Report malformed or unavailable GPU-authored records without assuming CPU
  visibility at capture time.

### Other tooling impact

The debug layer requires object-creation and recording-time validation.
GPU-based validation is described separately. DRED and GPU dump tooling should
identify the active WLS, PCS, program-table index, and record addresses when
that data is available. Header and metadata generators must expose the new API
and DDI types with matching GPU layouts.

### Dependencies and risks

The proposal depends on generic programs, state objects, program identifiers,
local root signatures, and enhanced or legacy synchronization.

Principal risks are:

- Hardware cost or limitations when switching among many programs.
- Driver complexity for multiple per-program record layouts.
- Ambiguous ordering if applications treat GPU-produced data as implicitly
  synchronized.
- Difficult diagnosis of invalid GPU-authored pointers or identifiers.
- Program-table and local-root-argument alignment requirements that are too
  weak or unnecessarily strict.
- Excessive state tracking if the final reset policy restores prior bindings.

The transparent record model, creation-time specialization, explicit upper
bounds, and GPU-validation extension mitigate these risks.

## Testing

### Runtime and API tests

- Create graphics and compute PCSes for every supported argument type.
- Reject missing, duplicate, and class-incompatible dispatch triggers, while
  accepting the trigger at any argument-array position.
- Validate source/binding combinations, natural alignment, minimum strides,
  and zero-stride broadcast.
- Validate shared global root signature and global/IA update-set uniformity.
- Validate implicit and explicit local root signatures and reject mixed paths.
- Validate state-object PCS associations and Work-Lists-exclusive programs.
- Validate program table address, range, stride, slot count, and immutability.
- Validate `SetProgram` and `DispatchList` command-list restrictions.

### Functional tests

- Execute one and many programs selected from GPU-authored tables.
- Mix PCSes with different primary and secondary payload layouts.
- Exercise inline-only records, secondary lists, zero counts, and broadcast
  secondary strides.
- Exercise local root arguments sourced from table, primary, and secondary
  records, including split root-constant ranges.
- Compare ordered and out-of-order graphics results where ordering is and is
  not observable.
- Produce every dispatch input and record buffer from a shader with no CPU
  readback.
- Verify program-table identifiers sourced from multiple state objects.

### Conformance and tooling tests

- Run the functional matrix on at least two vendor implementations and WARP.
- Confirm PIX decodes captures and attributes generated work correctly.
- Confirm debug-layer messages identify creation-time and recording-time
  violations.
- Confirm DDI/API structure layouts and capability reporting agree.
- Stress maximum supported table stride and large program/record counts.

## Alternatives considered

### Extend `ExecuteIndirect` with a pipeline argument

A pipeline argument alone would still impose one command layout on the whole
call and would not expose the per-program layout during program compilation.
It also does not naturally provide program-table local arguments or
primary/secondary value frequencies.

### Issue one indirect call per possible program

This is the existing workaround. It creates worst-case CPU and implementation
overhead proportional to the number of possible programs rather than the number
of active programs.

### Require one uniform record layout

A uniform layout simplifies dispatch but forces every record to reserve the
union of all bindings used by all programs. It increases memory bandwidth and
prevents compile-time specialization around each program's actual inputs.

### Use opaque implementation-managed records

Opaque records would require a preprocessing or update API and make GPU-side
record generation harder. Transparent records let applications use ordinary
buffer production and synchronization mechanisms.

### Put all values in per-execution records

This duplicates material or batch values across many executions. The
primary/secondary model stores shared values once while retaining contiguous
per-execution data.

### Split primary/secondary records or local root arguments into later proposals

Both mechanisms affect PCS creation, record packing, program compilation, table
layout, and validation. Removing them from the baseline would not produce an
orthogonal extension; it would require a second version of nearly every core
object and structure.

## Stakeholders

- Direct3D runtime, debug layer, DDI, and state-object owners.
- Hardware vendors and user-mode driver compiler owners.
- WARP owners.
- PIX, DRED, GPU dump, and GPU-based validation owners.
- Engine developers implementing GPU-driven rendering.
- Work Graphs, generic-program, partial-graphics-program, and root-signature
  feature owners.

## Prior work

- [`ExecuteIndirect`](../../d3d/IndirectDrawing.md)
- [Work Graphs](../../d3d/WorkGraphs.md)
- [Raytracing shader records](../../d3d/Raytracing.md)
- [Partial Graphics Programs](../../d3d/PartialGraphicsPrograms.md)
- [Enhanced Barriers](../../d3d/D3D12EnhancedBarriers.md)

## Open questions

- Which additional graphics state should records be able to update, and which
  additions should be deferred? Candidates include primitive topology, stencil
  reference, depth bounds, shading rate, render targets, scissors, and
  viewports.
- Are device-specific per-call, per-table, or per-list limits required, and
  what guaranteed minimums should applications receive?
- Is 8-byte program-table base/slot alignment sufficient for every
  implementation?
- Is 4096 bytes the correct maximum program-table stride?
- Should HLSL add source syntax for local root signatures on non-library
  generic programs?
- Should dispatch completion reset touched bindings or restore prior
  command-list state?
- Is an explicit preprocess operation needed by implementations that translate
  records into native packets?
- What preemption granularity applies: record, list, or complete call?
- Should `ALLOW_OUT_OF_ORDER_GRAPHICS` also relax ordering within one draw or
  mesh dispatch, and how would that interact with rasterizer-ordered views?
- Can graphics and compute generic programs with PCS associations be made
  portable to non-Work-Lists execution paths under a restricted feature set?
- Should a WLS optionally identify a payload subrange as a sort-key hint when
  out-of-order graphics is enabled?
- Should D3D provide a convenience path that converts an existing
  `ID3D12PipelineState` into an equivalent state-object generic program with a
  PCS association?
