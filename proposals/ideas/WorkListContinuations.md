---
title: "NNNN - Work List Continuations and Multi-Signature Dispatch"
params:
  authors:
    - tbd: TBD
  sponsors:
    - tbd: TBD
  status: Draft
---

## Introduction

This proposal extends [D3D12 Work Lists](WorkLists.md) with GPU-driven
continuation chains and per-list signature selection. One CPU-side
`DispatchList1` call can execute a sequence of GPU-authored lists, and each list
can select a different work list signature and paired binding. This enables
multi-phase pipelines, including compute-to-graphics handoffs, without CPU
readback between phases.

## Motivation

Core Work Lists execute one list against one bound signature. A multi-phase
GPU-driven pipeline must therefore return to the CPU between phases when:

- A later phase uses a different executable class or record layout.
- The number or address of later phases is determined by GPU work.
- A producer shader decides whether another phase is needed.
- A later phase needs a different program table or optional raytracing binding.

The CPU can issue a worst-case sequence of calls and arrange for unused calls to
read zero records, but that recreates the empty-call problem Work Lists are
intended to solve. CPU readback provides exact control but adds latency and
breaks fully GPU-resident pipelines.

Applications need a bounded command that lets GPU memory select both the next
list and the signature/binding used to interpret it, with explicit ordering and
visibility controls between phases.

## Proposed solution

This proposal adds:

- `ID3D12WorkListSignatureArray`, an immutable array of work list signatures.
- A parallel array of per-signature bindings supplied at `SetProgram`.
- `D3D12_DISPATCH_LIST_INPUT1`, which adds `SignatureIndex` and
  `NextDispatchList`.
- `DispatchList1`, which executes a head list and follows GPU-authored
  continuations.
- Flags controlling continuation permission, completion waits, and UAV-memory
  visibility.
- CPU-provided upper bounds for graphics list count and records per graphics
  list.

`SignatureIndex` selects one signature-array slot and its paired binding.
`NextDispatchList` points to another `D3D12_DISPATCH_LIST_INPUT1`. The
implementation follows the chain while the current list permits continuation
and the pointer is non-zero.

Continuation-only callers may retain the core single-signature `SetProgram`
binding and use `DispatchList1` with `SignatureIndex == 0`. A signature array is
required only when lists select among multiple signature/binding pairs or when
another extension requires the array binding.

## Detailed design

### Relationship to Core Work Lists

This proposal depends on the signatures, records, program tables, bindings, and
execution semantics in [D3D12 Work Lists](WorkLists.md).

It does not change the meaning of a single primary or secondary record. It adds
a list-level selection and chaining layer around core dispatch.

The optional
[Work Lists Raytracing Integration](WorkListsRaytracing.md) proposal may add
raytracing-class signatures and bindings to an array. Continuations work
correctly without that proposal.

### Signature arrays

An `ID3D12WorkListSignatureArray` contains one or more WLS pointers:

```c++
typedef struct D3D12_WORK_LIST_SIGNATURE_ARRAY_DESC
{
    UINT                            NumSignatures;
    ID3D12WorkListSignature* const* pSignatures;
} D3D12_WORK_LIST_SIGNATURE_ARRAY_DESC;
```

The array is immutable. Duplicate WLS pointers are allowed. This lets an
application pair one WLS with multiple program tables or other bindings by
placing the same pointer in multiple array slots.

Every PCS reachable through the array must satisfy the global-root and
per-executable-class uniformity rules defined by core Work Lists:

- One shared global root signature.
- One global root parameter update set for graphics-class signatures.
- One global root parameter update set for compute-class signatures.
- One input-assembler update set across graphics-class signatures.

Different array entries may have different executable classes.

The signature array retains one reference to each WLS in the descriptor.
Bindings do not retain the state objects whose identifiers appear in program
tables; the application preserves those objects for GPU execution as required
by core Work Lists.

### Paired binding array

The array form of `SetProgram` binds a signature array and an equally sized
binding array:

```c++
typedef struct D3D12_SET_WORK_LIST_DESC1
{
    ID3D12WorkListSignatureArray*  pSignatureArray;
    UINT                           NumBindings;
    const D3D12_WORK_LIST_BINDING* pBindings;
} D3D12_SET_WORK_LIST_DESC1;
```

`NumBindings` must equal the signature count. `pBindings[i]` pairs with
`pSignatures[i]`, and `SignatureIndex == i` selects both.

Core graphics/compute Work Lists use program-table bindings. If the raytracing
proposal is present, raytracing WLS entries use raytracing bindings. Each
binding type must match its paired WLS executable class.

Bindings are fixed for the complete dispatch chain. A continuation cannot issue
`SetProgram`; it can only select among entries bound by the CPU before
`DispatchList1`.

The core direct binding is also valid with `DispatchList1`. In that case every
list uses `SignatureIndex == 0` and the one bound WLS/binding pair. An array
binding is not valid with the core `DispatchList` entry point.

### Dispatch input

```c++
typedef struct D3D12_DISPATCH_LIST_INPUT1
{
    UINT                                  NumProgramInputs;
    D3D12_DISPATCH_LIST_FLAGS1            Flags;
    UINT                                  SignatureIndex;
    UINT                                  Reserved;
    D3D12_GPU_VIRTUAL_ADDRESS_AND_STRIDE  ProgramInputs;
    D3D12_GPU_VIRTUAL_ADDRESS             NextDispatchList;
} D3D12_DISPATCH_LIST_INPUT1;
```

The structure is GPU-resident and may be authored by shaders.

- `NumProgramInputs`, `ProgramInputs`, and base flags retain their core meaning.
- `SignatureIndex` selects the signature/binding pair.
- `Reserved` is zero.
- `NextDispatchList` is zero or points to another
  `D3D12_DISPATCH_LIST_INPUT1`.

Every list in a chain uses the `_INPUT1` layout. A continuation pointer may not
target a core `D3D12_DISPATCH_LIST_INPUT`.

A list with zero records may still continue to another list.
With a direct single-signature binding, `SignatureIndex` must be zero.

### Continuation flags

```c++
typedef enum D3D12_DISPATCH_LIST_FLAGS1
{
    D3D12_DISPATCH_LIST_FLAG1_NONE = 0,
    D3D12_DISPATCH_LIST_FLAG1_ALLOW_OUT_OF_ORDER_GRAPHICS =
        D3D12_DISPATCH_LIST_FLAG_ALLOW_OUT_OF_ORDER_GRAPHICS,
    D3D12_DISPATCH_LIST_FLAG1_ALLOW_NEXT_DISPATCH_LIST_CONTINUATION = 0x2,
    D3D12_DISPATCH_LIST_FLAG1_END_WITH_WAIT_FOR_COMPLETION          = 0x4,
    D3D12_DISPATCH_LIST_FLAG1_END_WITH_MEMORY_FLUSH                 = 0x8,
} D3D12_DISPATCH_LIST_FLAGS1;
```

- **`ALLOW_NEXT_DISPATCH_LIST_CONTINUATION`:** If set and `NextDispatchList` is
  non-zero, the implementation reads and launches the next list. Otherwise the
  chain ends.
- **`END_WITH_WAIT_FOR_COMPLETION`:** Prevents the next list or following
  command-list work from beginning until all work launched by the current list
  has completed. This is the synchronization half of a producer/consumer
  boundary.
- **`END_WITH_MEMORY_FLUSH`:** Makes shader UAV writes from the current list
  available to later work. This is the access/visibility half of a
  producer/consumer boundary.

An application commonly sets both wait and flush when the current list's
shaders write any part of the next list header, its records, or other data the
next list consumes.

### Next-list pointer semantics

The implementation may read `NextDispatchList` and begin preparing the next
list before all records in the current list retire. Record ordering does not
implicitly sequence continuation launch.

Therefore:

- A statically prepared chain with no data dependency may continue without a
  wait.
- A list whose shaders author the next list must set
  `END_WITH_WAIT_FOR_COMPLETION`.
- If the next list reads UAV data written by the current list, the current list
  must also establish memory visibility, normally using
  `END_WITH_MEMORY_FLUSH`.

The continuation address itself is consumed only when continuation is allowed.
A zero pointer ends the chain even when the flag is set.

The application is responsible for acyclic, terminating chains. Cycles or
unbounded chains are invalid.

### Launch order, retirement order, and visibility

Three concepts are independent:

- **Launch order:** When an implementation may begin processing a later list.
- **Retirement order:** The observable ordering of generated graphics work. The
  next list's `ALLOW_OUT_OF_ORDER_GRAPHICS` flag determines whether it may
  overtake work already in flight.
- **Memory visibility:** Whether writes from one list are available to shaders
  in a later list.

An ordered graphics list does not by itself make shader writes visible to a
continuation. Compute and raytracing lists have no record-retirement ordering
guarantee. Data dependencies always require explicit wait/visibility semantics.

Ordinary command-list operations recorded after `DispatchList1` form an ordered
trailing boundary, but they still require the appropriate memory visibility
when consuming UAV writes.

### State across continuations

The following remain fixed across the chain:

- The signature array.
- The parallel binding array.
- Command-list render targets, viewports, scissors, and other state not
  represented by record arguments.
- The shared global root signature contract.

Each list independently selects:

- One WLS and paired binding.
- Its primary list, record count, and stride.
- Its ordering and continuation flags.

Global root binding state remains separated into graphics and compute
namespaces. Switching executable class does not copy values between them.

Bindings updated by records follow the core Work Lists leakage/reset rules. The
end-of-call reset policy applies after the complete chain, not after each list.

### Cross-class pipelines

Signature arrays permit compute-to-graphics and graphics-to-compute chains. If
the raytracing extension is available, they also permit transitions to or from
raytracing-class lists.

For example:

```text
compute culling list
  -> compute compaction list
  -> graphics draw list
```

The compute producer can write the graphics list header and records, then
continue to it with wait and visibility flags. Render-target and rasterizer
state cannot change inside the chain, so phases requiring a different
command-list setup still require separate CPU-recorded calls.

### Graphics resource bounds

`DispatchList1` takes:

- `MaxGraphicsPrimaryLists`: maximum number of graphics-class lists in the
  chain.
- `MaxGraphicsProgramInputsPerPrimaryList`: maximum primary-record count in any
  graphics-class list in the chain.

A graphics list counts toward `MaxGraphicsPrimaryLists` even when it contains
zero records. The implementation may use these values to size scheduling or
translation resources before reading the GPU-authored chain.

Exceeding either bound is undefined behavior and may be diagnosed by GPU-based
validation.

Compute- and raytracing-class lists do not count toward
`MaxGraphicsPrimaryLists`.

### D3D API additions

#### Capability tier

```c++
typedef enum D3D12_WORK_LISTS_TIER
{
    D3D12_WORK_LISTS_TIER_NOT_SUPPORTED = 0,
    D3D12_WORK_LISTS_TIER_1             = 10,
    D3D12_WORK_LISTS_TIER_2             = 20,
} D3D12_WORK_LISTS_TIER;
```

Tier 2 includes core Work Lists plus the complete surface in this proposal.

#### Signature array creation

```c++
interface ID3D12WorkListSignatureArray : ID3D12DeviceChild
{
};

HRESULT ID3D12DeviceN::CreateWorkListSignatureArray(
    const D3D12_WORK_LIST_SIGNATURE_ARRAY_DESC* pDesc,
    REFIID                                      riid,
    void**                                      ppSignatureArray);
```

#### Array binding

`D3D12_PROGRAM_TYPE` gains `D3D12_PROGRAM_TYPE_WORK_LIST1`.
`SetProgram` interprets the payload as `D3D12_SET_WORK_LIST_DESC1`.

The separate GPU-validation proposal may add validation fields to this
descriptor.

Binding compatibility is:

| Bound program | `DispatchList` | `DispatchList1` |
|---|---|---|
| Core single WLS/binding | Valid | Valid; `SignatureIndex` is zero |
| WLS array and binding array | Invalid | Valid |

#### Dispatch

```c++
void ID3D12GraphicsCommandListN::DispatchList1(
    D3D12_GPU_VIRTUAL_ADDRESS DispatchListInput,
    UINT                      MaxGraphicsPrimaryLists,
    UINT                      MaxGraphicsProgramInputsPerPrimaryList);
```

`DispatchListInput` points to `D3D12_DISPATCH_LIST_INPUT1`.

### Validation

Creation-time validation checks:

- Non-zero signature count.
- Non-null WLS pointers.
- Shared global root signature compatibility.
- Per-class global root and input-assembler update-set compatibility.

`SetProgram` debug-layer validation checks:

- Tier 2 support.
- Signature-array and binding counts match.
- Every binding type matches its paired WLS.
- Binding descriptors and resources satisfy core and optional raytracing rules.

`DispatchList1` recording-time validation checks:

- A direct command list for any chain that may select graphics, or a compute
  command list when every selected signature is compute- or raytracing-class.
- Alignment of the head input address.
- Legal CPU-provided maxima.

Execution is invalid when:

- `SignatureIndex` is out of range.
- A continuation pointer is misaligned or inaccessible.
- A continuation targets the wrong input structure.
- A chain cycles or does not terminate.
- A graphics bound is exceeded.
- A producer omits required wait or visibility flags.

The separate GPU Timeline Validation Hooks proposal defines execution-time
diagnosis for selected structural errors.

### DDI changes

```c++
typedef HRESULT (APIENTRY* PFND3D12DDI_CREATEWORKLISTSIGNATUREARRAY)(
    D3D12DDI_HDEVICE                               hDevice,
    const D3D12DDI_WORK_LIST_SIGNATURE_ARRAY_DESC* pDesc,
    D3D12DDI_HWORKLISTSIGNATUREARRAY               hSignatureArray);

typedef void (APIENTRY* PFND3D12DDI_DISPATCHLIST_1)(
    D3D12DDI_HCOMMANDLIST         hCommandList,
    D3D12DDI_GPU_VIRTUAL_ADDRESS DispatchListInput,
    UINT                          MaxGraphicsPrimaryLists,
    UINT                          MaxGraphicsProgramInputsPerPrimaryList);
```

The runtime forwards the signature-array handle and parallel binding array
through `SetProgram`. API and DDI `_INPUT1` structures have identical GPU byte
layouts.

A driver reporting Tier 2 implements the Tier 1 DDI and both additions above.

### WARP support

WARP should implement signature arrays, all list classes it otherwise supports,
continuation walking, wait/visibility flags, and graphics bound validation.
Serial execution is conformant.

### PIX support

PIX must visualize a continuation chain, including:

- Each list's GPU address.
- `SignatureIndex` and paired binding.
- Executable class.
- Primary record count and stride.
- Flags and continuation address.
- Wait/visibility boundaries.

Capture and replay must preserve GPU-authored chains and detect loops or
unreadable links without hanging tooling.

### Other tooling impact

GPU dump and DRED data should include the current chain link and signature index.
Debug tooling needs bounded chain traversal and cycle detection. Header and DDI
generators must preserve the `_INPUT1` GPU layout.

### Dependencies and risks

This proposal depends on core Work Lists. Raytracing entries additionally
depend on the raytracing proposal.

Risks include:

- GPU-authored cycles or corrupt pointers hanging hardware or tooling.
- Applications confusing graphics ordering with memory visibility.
- CPU maxima that are difficult to choose without defeating GPU-driven
  flexibility.
- Implementations needing to reserve excessive resources for the reported
  maxima.
- Command-list state fixed across a chain limiting some multi-phase pipelines.

## Testing

### API and validation tests

- Create arrays with one, many, and duplicate WLS pointers.
- Reject empty arrays, null entries, incompatible global root signatures, and
  incompatible per-class update sets.
- Validate binding-array count and per-slot class matching.
- Validate Tier 1 rejection of Tier 2 entry points.
- Validate head input alignment and CPU upper-bound parameters.

### Functional tests

- Execute one-list chains through `DispatchList1`.
- Execute multiple lists using one repeated signature and different bindings.
- Execute compute-to-graphics and graphics-to-compute chains.
- Exercise zero-record lists that continue.
- Exercise static chains without waits.
- Have shaders author the next header and records, using wait and visibility
  flags.
- Verify ordered/unordered graphics behavior at every boundary combination.
- Verify fixed command-list state across a chain.
- Exercise raytracing entries when that proposal is supported.

### Negative and stress tests

- Out-of-range signature indices.
- Null, misaligned, inaccessible, and cyclic continuation pointers.
- Exceeded graphics-list and per-list record bounds.
- Missing waits or visibility for producer/consumer chains.
- Very long but valid chains within implementation limits.
- PIX capture/replay and GPU dump handling of malformed chains.

## Alternatives considered

### Issue one `DispatchList` per phase

This requires the CPU to know every phase and its signature. It cannot follow a
GPU-decided number of phases without readback or worst-case empty calls.

### Put all signatures in one WLS

A WLS intentionally contains one executable class and one compatible global
binding contract. Combining unrelated classes and list-level layouts would make
creation-time specialization and validation substantially more complex.

### Store a signature pointer in GPU memory

GPU virtual addresses cannot safely identify CPU device-child objects. A
bounded integer index into a CPU-created immutable array gives the driver a
known set to validate and precompute.

### Allow continuations to rebind arbitrary state

Letting GPU memory perform `SetProgram` or general command-list state changes
would require a much broader GPU command processor and security model. The
fixed binding array keeps the selected state bounded and prevalidated.

### Implicit synchronization at every continuation

Always waiting and flushing would make the model simple but would serialize
independent phases and eliminate implementation scheduling freedom. Explicit
flags let applications pay only for real dependencies.

## Stakeholders

- Direct3D Work Lists, command-list, synchronization, and debug-layer owners.
- Hardware vendors and scheduler/command-processor owners.
- WARP owners.
- PIX, DRED, GPU dump, and GPU-based validation owners.
- Engine developers building GPU-driven multi-phase pipelines.
- Raytracing owners when raytracing entries are used.

## Prior work

- [D3D12 Work Lists](WorkLists.md)
- [Work Lists Raytracing Integration](WorkListsRaytracing.md)
- [Work Graphs](../../d3d/WorkGraphs.md)
- [Enhanced Barriers](../../d3d/D3D12EnhancedBarriers.md)

## Open questions

- What implementation or API limit, if any, should bound continuation-chain
  length?
- Should the feature guarantee minimum values for graphics list and per-list
  record bounds?
- Can graphics and compute global root binding namespaces optionally be shared
  for applications that use identical bindings?
- Should wait-only and flush-only flags have a direct formal mapping to legal
  enhanced barriers, or remain semantic operations specific to Work Lists?
- What exact write classes are covered by `END_WITH_MEMORY_FLUSH`?
- Should a future extension permit selected command-list state changes between
  continuation phases?
