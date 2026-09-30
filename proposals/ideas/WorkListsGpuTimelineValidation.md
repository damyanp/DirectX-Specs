---
title: "NNNN - Work Lists GPU Timeline Validation Hooks"
params:
  authors:
    - tbd: TBD
  sponsors:
    - tbd: TBD
  status: Draft
---

## Introduction

This proposal adds GPU-timeline validation hooks for
[D3D12 Work Lists](WorkLists.md). Work Lists consume GPU-authored pointers,
counts, indices, identifiers, and record payloads that may not exist when the
CPU records the command list. The hooks let a debug layer bind validation
programs that execute immediately before the corresponding Work Lists data is
consumed, report structural errors, and defensively neutralize selected invalid
inputs.

## Motivation

Traditional API and debug-layer validation can inspect object descriptors and
CPU parameters, but it cannot validate Work Lists bytes produced later by GPU
work. Important failures are therefore otherwise undefined behavior:

- An out-of-range signature index.
- An out-of-range program-table index.
- A primary or secondary list pointer outside an expected allocation.
- A record count exceeding an application allocation.
- A continuation pointer with an invalid shape.
- An identifier or local-root-argument footprint incompatible with the selected
  signature.

The debug layer could inject an independent compute pass before every
`DispatchList`, but a separate pass cannot naturally run at every continuation
boundary or at the point where the implementation has resolved a selected
program and its record addresses.

Validation needs implementation-supplied context and well-defined execution
points while preserving the application's transparent record format.

## Proposed solution

The command list optionally binds a validation program table containing generic
compute program identifiers. Non-zero indices in work list signature
descriptors select validators for three areas:

1. Signature selection, before `SignatureIndex` chooses a signature-array slot.
2. Primary-list validation, after signature selection and before record
   processing.
3. Secondary-list validation, after a primary record selects a PCS and before
   its executions.

The implementation invokes the selected validator with system-generated GPU
virtual address arguments identifying the relevant Work Lists data.

Validators report errors through debug-layer-owned UAVs and may neutralize
selected invalid data, such as clamping an index or setting a record count to
zero. Barriers guarantee that validator writes are visible before Work Lists
consumes the patched fields. A fourth, shader-validation area is implemented by
debug-layer shader instrumentation and requires no hook API.

## Detailed design

### Dependencies

This proposal depends on:

- [D3D12 Work Lists](WorkLists.md) for signatures, records, program tables, and
  dispatch.
- [Work List Continuations and Multi-Signature Dispatch](WorkListContinuations.md)
  for signature arrays and the `D3D12_SET_WORK_LIST_DESC1` binding path.

It may validate raytracing-class lists when
[Work Lists Raytracing Integration](WorkListsRaytracing.md) is present, but does
not require raytracing support.

Applications must not populate validation index fields or bind the validation
program table. The surface is reserved for the debug layer and tools, which may
need to substitute programs and root signatures transparently.

### Validation hook areas

#### Signature-selection validator

Runs once per dispatch-list header before `SignatureIndex` selects a WLS and
binding.

It receives a pointer to the `D3D12_DISPATCH_LIST_INPUT1` header and can:

- Check `SignatureIndex` against the bound signature/binding count.
- Check continuation-header invariants available before signature selection.
- Report an error.
- Neutralize a bad list by writing `SignatureIndex = 0` and
  `NumProgramInputs = 0`.

Neutralization intentionally does not clear `NextDispatchList`. A malformed list
can be skipped while validation continues through a producer-authored chain.

#### Primary-list validator

Runs once per list after signature selection and before any primary record is
processed.

It receives:

- The dispatch-list header address.
- The selected program table address when the selected class uses one.
- The primary-list address.

It can validate:

- Primary-list range, stride, and allocation-specific bounds.
- `NumProgramInputs`.
- Program-table range and slot-count assumptions known to the debug layer.
- Per-list application invariants.

It may set `NumProgramInputs = 0` to suppress the complete list.

#### Secondary-list validator

Runs once for each primary record after the implementation has selected its PCS
and before secondary executions begin.

It receives the current primary-record address and that record's secondary-list
address. Additional tracked context, such as program-table stride, slot count,
argument layout, and descriptor-heap bases, is supplied through
debug-layer-private bindings.

The selected PCS supplies the validator index. Different programs may therefore
use validators specialized for their record layouts.

It can validate:

- `ProgramTableIndex`.
- Program identifier and expected program metadata.
- Local-root-argument footprint.
- Secondary list pointer, count, stride, and allocation bounds.
- Program-specific primary and secondary payload invariants.

It may set a hybrid primary record's `NumSecondaryRecords = 0` to suppress its
executions. For inline primary records it may neutralize the dispatch-trigger
payload where that argument type has a defined zero-work form.

For graphics- and compute-class PCSes, the validator runs once per primary
record, including fully inline records. For a raytracing-class PCS it runs only
for hybrid records; a fully inline raytracing PCS must use validation index
zero.

#### Shader validation

The debug layer may rewrite application shaders to run validation inside each
record execution. This area has no proposal-level hook or descriptor field. The
instrumented shader may use the Tier 1 data-program forms of
`PROGRAM_TABLE_POINTER`, `PRIMARY_RECORD_POINTER`, and
`SECONDARY_RECORD_POINTER` to inspect the record that launched it.

### Validation program table

The validation program table uses the core
`D3D12_WORK_LIST_PROGRAM_TABLE_BINDING` layout:

```c++
typedef struct D3D12_WORK_LIST_PROGRAM_TABLE_BINDING
{
    D3D12_GPU_VIRTUAL_ADDRESS_RANGE_AND_STRIDE Table;
    UINT                                       SlotCount;
} D3D12_WORK_LIST_PROGRAM_TABLE_BINDING;
```

Each slot contains a generic compute program identifier and any local root
arguments used by that validation program. Slot zero is reserved as the
"no validator" sentinel and is never invoked.

Validation programs:

- Are ordinary generic compute programs in state objects.
- Have their own PCSes.
- Use `FIXED_DISPATCH` to declare the validator dispatch grid.
- May bind debug-layer-owned UAVs for reporting.
- May use the system-generated pointer arguments defined below.

The validation table is immutable from GPU execution of `SetProgram` until all
referencing chains complete.

### Descriptor indices

The core PCS descriptor gains:

```c++
UINT RecordValidationProgramTableIndex;
```

Zero disables secondary-list validation for that PCS. A non-zero value selects
one validation program table slot.

The core WLS descriptor gains:

```c++
UINT ListValidationProgramTableIndex;
```

Zero disables primary-list validation for that WLS. A non-zero value selects
one validation program table slot.

The signature-array `SetProgram` descriptor gains:

```c++
D3D12_WORK_LIST_PROGRAM_TABLE_BINDING ValidationProgramTable;
UINT SignatureSelectionValidationProgramTableIndex;
```

A zero `ValidationProgramTable.Table.StartAddress` disables all hooks and
requires the remaining table fields and signature-selection index to be zero.

Every non-zero validation index in every bound WLS and PCS must be less than
`ValidationProgramTable.SlotCount`.

### Why validation is bound only through the array path

Validation must run before signature selection and at every continuation link.
The array binding already provides:

- The set of signatures and bindings that may be selected.
- A stable command-list binding covering the complete chain.
- A place to bind one validation table shared by all selected signatures.

A size-one signature array supports validation for applications that otherwise
need only one signature.

Core Tier 1 implementations without this proposal may still use debug-layer
injection: the layer records a separate compute validation dispatch immediately
before the application's `DispatchList`. That strategy cannot provide the same
per-continuation integration and requires no public hook surface.

### System-generated pointer argument types

This proposal adds six `D3D12_INDIRECT_ARGUMENT_TYPE` values:

```text
DISPATCH_LIST_HEADER_POINTER
PROGRAM_TABLE_POINTER
PRIMARY_LIST_POINTER
PRIMARY_RECORD_POINTER
SECONDARY_LIST_POINTER
SECONDARY_RECORD_POINTER
```

Each value is an implementation-generated GPU virtual address bound as a root
descriptor at the argument's declared `RootParameterIndex`.

- **`DISPATCH_LIST_HEADER_POINTER`:** Address of the current
  `D3D12_DISPATCH_LIST_INPUT1`. Available to the signature-selection and
  primary-list validators.
- **`PROGRAM_TABLE_POINTER`:** Address of the selected program table. Available
  to the primary-list validator and, for graphics/compute, to data PCSes.
- **`PRIMARY_LIST_POINTER`:** Address of the current list's primary-record
  array. Available only to the primary-list validator.
- **`PRIMARY_RECORD_POINTER`:** Address of the primary record currently being
  validated or executed. Available to the secondary-list validator and to data
  PCSes.
- **`SECONDARY_LIST_POINTER`:** Address of the selected primary record's
  secondary-record array, or zero when none exists. Available only to the
  secondary-list validator.
- **`SECONDARY_RECORD_POINTER`:** Address of the current secondary record.
  Available only to data PCSes for hybrid lists.

The program table, primary record, and secondary record forms are available to
data PCSes at Tier 1 so GPU-based validation can patch shaders with context
needed for instrumentation. Validator dispatches and the remaining forms are
Tier 2.

Pointer arguments:

- Use `Source == SYSTEM`.
- Target a root descriptor.
- Contribute no record bytes.
- Are supplied independently for each invocation.

### Validator dispatch dimensions

A validation PCS uses:

```text
FIXED_DISPATCH(x, y, z)
```

with `Source == STATIC`.

The signature-selection and primary-list validators normally use `(1, 1, 1)`.
A secondary-list validator may use a fixed grid selected for its validation
algorithm. Dynamic grid sizing is not provided by this proposal.

### Invocation order

For each list:

1. Invoke the signature-selection validator, if configured.
2. Establish visibility for its writes.
3. Read the possibly corrected `SignatureIndex` and select the WLS/binding.
4. Invoke the selected WLS's primary-list validator, if configured.
5. Establish visibility for its writes.
6. Resolve every primary record's selected PCS and invoke its secondary-list
   validator when configured. These invocations may run concurrently.
7. Wait for all secondary-list validators and establish visibility for their
   writes.
8. Re-read the possibly modified list and record fields.
9. Process the surviving records. Instrumented shader validation runs inside
   each execution.
10. Apply continuation semantics and repeat for the next list.

The implementation must not speculatively consume fields that a validator is
permitted to patch before the validator and required barrier complete.

### Barriers and memory visibility

Validators write application-owned Work Lists input fields and debug-layer-owned
reporting buffers through UAV access.

After each validator invocation, the implementation provides ordering
equivalent to:

- Completion of the validator dispatch.
- UAV write visibility for fields the Work Lists engine subsequently reads.

This ordering is internal to validation hooks and does not replace application
synchronization between ordinary producer shaders and Work Lists consumption.

Reporting UAV visibility to later CPU readback follows the debug layer's normal
GPU-based validation synchronization.

### Defensive neutralization

Validators should prefer changes that suppress unsafe work while retaining
enough control flow to continue diagnosis:

- Bad signature index: select slot zero and set `NumProgramInputs = 0`.
- Bad primary list: set `NumProgramInputs = 0`.
- Bad program-table index in a hybrid record: set
  `NumSecondaryRecords = 0`.
- Bad secondary pointer or count: set `NumSecondaryRecords = 0`.
- Bad inline dispatch trigger: write the trigger's defined zero-work form where
  possible.

Neutralization is best-effort. It does not make invalid application behavior
valid and cannot guarantee recovery from arbitrary corrupt pointers or races.

### Validation in continuation chains

Each continuation is validated independently. The signature-selection validator
runs before every `SignatureIndex` read, followed by the selected WLS and PCS
validators.

A validator may suppress one list while leaving its continuation intact. This
lets validation collect errors from later links when the chain structure itself
is still readable.

Validation cannot safely follow a continuation pointer that is unreadable,
misaligned, or outside an accessible allocation. Such errors may terminate
validation and execution according to implementation safety requirements.

### Limitations

The hooks do not guarantee diagnosis of:

- Arbitrary inaccessible GPU virtual addresses before hardware access.
- Data races that continue while validation executes.
- Cyclic or unbounded continuation chains without an implementation traversal
  limit.
- Semantic incompatibility that requires reconstructing complete shader
  behavior.
- Every invalid program identifier or state-object association on hardware that
  cannot expose the required metadata.
- Errors introduced after a validator runs.

Applications must not rely on hooks for correctness. Validation programs and
neutralization are debug facilities.

### D3D API additions

#### Descriptor fields

```c++
typedef struct D3D12_PROGRAM_COMMAND_SIGNATURE_DESC
{
    // Core fields...
    UINT RecordValidationProgramTableIndex;
} D3D12_PROGRAM_COMMAND_SIGNATURE_DESC;

typedef struct D3D12_WORK_LIST_SIGNATURE_DESC
{
    // Core fields...
    UINT ListValidationProgramTableIndex;
} D3D12_WORK_LIST_SIGNATURE_DESC;

typedef struct D3D12_SET_WORK_LIST_DESC1
{
    ID3D12WorkListSignatureArray*          pSignatureArray;
    UINT                                   NumBindings;
    const D3D12_WORK_LIST_BINDING*         pBindings;
    D3D12_WORK_LIST_PROGRAM_TABLE_BINDING  ValidationProgramTable;
    UINT SignatureSelectionValidationProgramTableIndex;
} D3D12_SET_WORK_LIST_DESC1;
```

The fields must be zero when hooks are unsupported or disabled.

#### Argument types

The six pointer forms above are added to `D3D12_INDIRECT_ARGUMENT_TYPE`.
Their descriptor payload identifies the root parameter index. They require
`Source == SYSTEM`.

#### Capability

GPU timeline validation hooks are mandatory for Work Lists Tier 2. No separate
application-facing capability bit is required.

### Validation of the validation configuration

Creation-time checks include:

- Pointer argument types use `SYSTEM` source and a compatible root descriptor.
- Validation PCSes are compute-class and use `FIXED_DISPATCH`.
- Data PCSes use only pointer forms meaningful to their execution context.
- Raytracing-class restrictions remain satisfied.

`SetProgram` checks include:

- Validation table fields are consistently all-zero when disabled.
- Slot zero is reserved.
- Every non-zero descriptor index is within `SlotCount`.
- Every referenced validation program has a compatible identifier, PCS, and
  root signature where CPU-visible metadata permits.

GPU execution validation checks table contents that are not CPU-visible.

### DDI changes

The API additions mirror one-to-one into:

- `D3D12DDI_PROGRAM_COMMAND_SIGNATURE_DESC`.
- `D3D12DDI_WORK_LIST_SIGNATURE_DESC`.
- The DDI counterpart of `D3D12_SET_WORK_LIST_DESC1`.
- `D3D12DDI_INDIRECT_ARGUMENT_TYPE`.

No new DDI entry points are required. Validation programs are generic compute
programs, identifiers use existing state-object mechanisms, and the validation
table is forwarded through existing `SetProgram` plumbing.

Drivers invoke validators at the defined execution points and synthesize pointer
arguments from the active list and record context.

### WARP support

WARP Tier 2 must implement the hooks. A serial implementation is sufficient and
provides a useful reference for validator ordering, pointer generation,
barriers, and neutralization.

### PIX support

PIX should display:

- Whether validation hooks were bound.
- Validation table slots and selected validator programs.
- Validator invocations associated with each chain link and record.
- Reported errors and any fields neutralized before execution.

Capture/replay should preserve debug-layer substitutions without presenting
them as application-authored state.

### Other tooling impact

The debug layer owns validator program creation, reporting buffers, message
formatting, and table substitution. DRED and GPU dump tooling should distinguish
application programs from validators and retain validator reports when device
removal occurs. Driver diagnostics should identify which validation area was
active.

### Dependencies and risks

Risks include:

- Validation changing scheduling or hiding timing-sensitive races.
- A validator fault obscuring the original application error.
- Neutralization allowing execution to continue with misleading secondary
  symptoms.
- Driver complexity around internal barriers and pointer synthesis.
- Validation table substitutions conflicting with profilers or other command
  consumers.

Validators must be minimal, deterministic, and isolated from application
correctness.

## Testing

### Configuration tests

- Accept zeroed validation fields with no table.
- Reject partially initialized table bindings.
- Reject out-of-range indices.
- Reject illegal pointer argument source/binding combinations.
- Validate fixed-dispatch validator PCSes.
- Validate size-one and multi-entry signature arrays.

### Invocation tests

- Verify all three validator areas execute in the required order.
- Verify barriers make validator writes visible before consumption.
- Verify validator arguments contain the exact header, table, list, and record
  addresses expected.
- Verify zero indices skip invocation.
- Verify per-PCS validators change as program-table selections change.
- Verify every continuation link is validated.

### Error and neutralization tests

- Out-of-range signature index.
- Invalid primary list range or count.
- Out-of-range program-table index.
- Invalid secondary pointer, count, and stride.
- Undersized program-table local-root-argument storage.
- Invalid continuation pointer.
- Missing or incompatible program associations where observable.
- Confirm each supported neutralization suppresses unsafe work and reports the
  original error.

### Tooling and conformance tests

- Run identical malformed workloads on WARP and vendor Tier 2 implementations.
- Verify debug-layer messages are stable and actionable.
- Verify PIX distinguishes validation work from application work.
- Verify capture/replay with hooks enabled and disabled.
- Stress large tables, many PCSes, and long continuation chains.

## Alternatives considered

### CPU-only validation

CPU validation cannot inspect bytes authored after command-list recording and
cannot follow GPU-decided continuation chains.

### Inject one compute dispatch before `DispatchList`

This remains a useful Tier 1 fallback, but it cannot naturally validate each
continuation after signature selection or receive implementation-resolved
per-record context.

### Make record buffers opaque

Opaque implementation-managed records could be validated during construction,
but would sacrifice transparent GPU authoring, one of the core Work Lists
goals.

### Require applications to provide validators

Validation is a debugging responsibility. Requiring application participation
would make correctness tooling inconsistent and prevent transparent debug-layer
substitution.

### Stop execution on the first error

Immediate termination is safest but often produces too little diagnostic data.
Defensive neutralization permits continued validation where the remaining chain
is readable. Implementations may still terminate when safe recovery is
impossible.

### Generalize hooks into a D3D-wide validation framework

The execution points and pointer contexts are specific to Work Lists. A broader
framework may be considered later, but generalization would expand scope and
delay validation for this feature.

## Stakeholders

- Direct3D Work Lists, runtime, debug-layer, and DDI owners.
- GPU-based validation owners.
- Hardware vendors and driver validation/compiler owners.
- WARP owners.
- PIX, DRED, and GPU dump owners.
- Engine developers producing Work Lists entirely on the GPU.

## Prior work

- [D3D12 Work Lists](WorkLists.md)
- [Work List Continuations and Multi-Signature Dispatch](WorkListContinuations.md)
- [Work Lists Raytracing Integration](WorkListsRaytracing.md)
- [D3D12 GPU Dumps](../../d3d/D3D12GpuDumps.md)

## Open questions

- What maximum continuation traversal should validation enforce before
  reporting a probable cycle?
- Which invalid identifier and state-object association checks can every vendor
  expose reliably to validator programs?
- Should neutralization behavior be normative per error class or remain
  debug-layer policy?
- Should pointer argument types available to data programs remain part of the
  Tier 1 GPU-based validation fallback even though hook invocation is Tier 2?
- How should multiple command consumers, such as a debug layer and profiler,
  compose program-table substitutions?
