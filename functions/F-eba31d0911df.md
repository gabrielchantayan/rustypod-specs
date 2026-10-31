<!-- Trusted provenance: F-eba31d0911df; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eba31d0911df

## Purpose

F-eba31d0911df conditionally obtains a record through two dependencies and, when both return nonzero values, writes two differences derived from four signed 16-bit record fields.

The first output is the field at byte offset `0x16` minus the field at byte offset `0x12`. The second output is the field at byte offset `0x14` minus the field at byte offset `0x10`. Each field is interpreted as signed 16-bit and widened to signed 32-bit before subtraction.

The function returns a 32-bit status value: zero if either dependency returns zero, or one after both output assignments complete. It does not return either difference directly.

A plausible intent is to obtain two extents from paired record coordinates. That interpretation is inferred, not established: the evidence does not identify coordinate axes, units, record semantics, or whether negative differences have application-specific meaning.

## Interface

The interface has five positional arguments and a 32-bit return value.

| Argument | Established use |
|---|---|
| First | An address-bearing 32-bit value identifying an object. The function reads a 32-bit field at byte offset `0x40` and passes that field to F-3a548392a5a0. |
| Second | A 32-bit value that is not used by the shown function body. |
| Third | A 32-bit value forwarded as the second argument to F-69facf68ac4d, but only if F-3a548392a5a0 returns nonzero. Its meaning is unknown. |
| Fourth | Address of the first writable 32-bit output location. Written only on the successful path. |
| Fifth | Address of the second writable 32-bit output location. Written only on the successful path, after the first output write. |

### Relevant layouts

The first argument exposes the following field:

| Byte offset | Width | Treatment |
|---|---:|---|
| `0x40` | 32 bits | Read once by this function before the first dependency call; forwarded without local interpretation. |

The nonzero result from F-69facf68ac4d is treated as a record address with these fields:

| Byte offset | Width | Interpretation | Role |
|---|---:|---|---|
| `0x10` | 16 bits | Signed | Subtrahend for the second output |
| `0x12` | 16 bits | Signed | Subtrahend for the first output |
| `0x14` | 16 bits | Signed | Minuend for the second output |
| `0x16` | 16 bits | Signed | Minuend for the first output |

Offsets are byte offsets from the corresponding base address. No other record fields are read directly by the shown function.

### Preconditions

For ordinary completion, the first argument must support a readable 32-bit access at offset `0x40`. This requirement applies even when the function ultimately returns zero.

If the second dependency returns a nonzero address, that address must support the four signed 16-bit reads described above. Both output addresses must then support 32-bit writes.

The function contains no explicit validation of these memory conditions. Alignment requirements, address-wrap behavior, and fault handling are not established by the supplied evidence.

Output addresses are not directly dereferenced when either dependency returns zero. This observation does not establish that arbitrary output addresses are valid under a broader caller contract; it describes only this function’s direct accesses on those paths.

The dependencies must return normally for the corresponding subsequent behavior to occur. The evidence does not establish their ownership rules, side effects, or validity guarantees.

## Behavior

### Lookup sequence

The function first reads the 32-bit field at offset `0x40` from the object identified by its first argument. It calls F-3a548392a5a0 with that field value.

If F-3a548392a5a0 returns zero:

- F-69facf68ac4d is not called.
- Neither output location is directly written by this function.
- No result-record fields are read.
- The function returns zero.

If F-3a548392a5a0 returns nonzero, the function calls F-69facf68ac4d with two arguments, in this order:

1. The nonzero value returned by F-3a548392a5a0.
2. The original third argument to F-eba31d0911df.

If F-69facf68ac4d returns zero:

- Neither output location is directly written by this function.
- No result-record fields are read.
- The function returns zero.

A zero value in the initial object field is not itself a local early-exit condition. The field is passed to F-3a548392a5a0, and that dependency’s return value controls whether execution proceeds.

### Successful computation

A nonzero result from F-69facf68ac4d is used as the record base address.

The function first reads the signed 16-bit fields at offsets `0x16` and `0x12`, widens both to signed 32-bit, and computes:

**First output = signed field at `0x16` − signed field at `0x12`.**

It stores that result as a 32-bit value through the fourth argument.

Only after this first output assignment does it evaluate and assign the second difference:

**Second output = signed field at `0x14` − signed field at `0x10`.**

The second result is stored as a 32-bit value through the fifth argument.

After both assignments complete, the function returns one.

The supplied evidence establishes the ordering between these two assignments. It does not establish which of the two input fields within an individual subtraction is loaded first.

### Integer behavior

All four record fields are signed 16-bit quantities. Their possible values range from −32,768 through 32,767. They are sign-extended before subtraction; the subtraction is not performed as a wrapping 16-bit operation.

Each difference therefore lies between −65,535 and 65,535, inclusive. Every possible difference fits in signed 32-bit storage.

Consequences include:

- A negative difference is retained.
- Equal paired fields produce zero.
- A difference can exceed the signed 16-bit range without overflowing the 32-bit output.
- The function performs no absolute-value conversion, clamping, normalization, or ordering check.
- A zero or negative difference does not change the success status.

A successful return indicates that both lookups produced nonzero results and both assignments completed. It does not certify that either difference is positive or meaningful to a consumer.

### Aliasing and sequential visibility

The function does not take a four-field snapshot before writing outputs. The first output write precedes the reads used for the second difference.

If the fourth argument points into storage containing either second-difference field, the first output write can change what the second computation reads. The resulting second output must then be understood from the memory contents after that write, subject to the target’s memory-access rules.

If both output arguments identify the same writable 32-bit location and neither output write changes the relevant record fields, the second assignment overwrites the first. The final shared value is the second difference.

More complicated overlaps depend on byte layout, endianness, and the permitted memory accesses. The evidence does not justify treating overlapping outputs and record storage as independent.

## Side effects

The direct side effects on the successful path are two 32-bit stores:

1. A store through the fourth argument.
2. A subsequent store through the fifth argument.

On either zero-return lookup path, this function performs no direct output stores. That is narrower than a guarantee that all reachable memory remains unchanged: both dependencies may have side effects, and their behavior is unknown.

The function makes one call to F-3a548392a5a0 and at most one call to F-69facf68ac4d. It performs no visible retry, fallback, diagnostic emission, or explicit resource release.

No allocation or deallocation operation appears directly in this function. This does not establish that its dependencies allocate nothing, nor that returned values are borrowed or owned. The function uses the second dependency’s nonzero result without visibly retaining or releasing it.

The two output stores are not an atomic transaction. If the first store completes and a later access fails, the supplied evidence shows no rollback mechanism. No particular exception, signal, or recovery behavior is established.

No direct global-variable access is shown. Global changes by dependencies remain unknown.

## Dependencies

### F-3a548392a5a0

**Input:** The 32-bit field read from the first argument’s object at offset `0x40`.

**Observed result handling:**

- Zero causes an immediate zero-return path.
- A nonzero result is forwarded as the first argument to F-69facf68ac4d.

The function does not otherwise inspect or dereference this result.

**Unknowns:** Argument meaning, return-value representation, memory ownership, global effects, error reporting, and any mutation of caller-visible storage.

### F-69facf68ac4d

**Inputs:** The nonzero result of F-3a548392a5a0, followed by the original third argument.

**Observed result handling:**

- Zero causes a zero return without direct output writes.
- A nonzero result is treated as a record address supplying four signed 16-bit fields.

**Unknowns:** Lookup semantics, selector interpretation, record lifetime, ownership, validity guarantees, global effects, and error reporting beyond the caller’s zero/nonzero branch.

Neither dependency may be assumed to preserve globals, preserve all externally reachable memory, or transfer memory ownership merely because this caller does not explicitly manage those concerns.

## Acceptance examples

These are conditional examples of the contract, not reports of executed tests. Unless stated otherwise, they assume valid memory, normally returning dependencies, nonoverlapping outputs, and stable record fields.

### First lookup returns zero

Suppose F-3a548392a5a0 returns zero.

The expected result is status zero. F-69facf68ac4d is not called, and this function makes no direct writes through either output argument.

Any stronger assertion that output values remain unchanged also requires that the dependency itself does not alter them through another reachable reference.

### Second lookup returns zero

Suppose F-3a548392a5a0 returns a nonzero value and F-69facf68ac4d returns zero.

The expected result is status zero. The second dependency receives the first dependency’s result and the original third argument. No record fields are read, and this function makes no direct output writes.

### Ordinary positive differences

Suppose both lookups return nonzero and the record fields are:

| Offset | Signed value |
|---|---:|
| `0x10` | 10 |
| `0x12` | 20 |
| `0x14` | 70 |
| `0x16` | 95 |

The first output becomes 75. The second becomes 60. The function returns one.

These values establish the offset pairing without assigning names such as width or height.

### Negative and zero differences

Suppose the fields at `0x16` and `0x12` are −100 and 200, while the fields at `0x14` and `0x10` are both 7.

The first output becomes −300. The second becomes zero. The return value remains one.

### Full signed-16-bit difference range

Suppose the first pair is 32,767 minus −32,768, and the second pair is −32,768 minus 32,767.

The outputs are 65,535 and −65,535, respectively. Neither calculation wraps to a signed 16-bit result.

### Shared output location

Suppose both output arguments identify the same 32-bit location, separate from the record. If the first difference is 75 and the second is 60, that location is written with 75 and subsequently overwritten with 60. The function returns one.

### Unused second argument

Changing only the second argument does not change the shown body’s argument forwarding, branch conditions, field accesses, or output calculations. This is a statement about the supplied function body, not a general guarantee about surrounding execution environments.

## Uncertainties

The following are **documented_only**; they are not resolved by assumptions:

- The semantic identity of the input object and returned record.
- Whether the differences represent dimensions, coordinate deltas, or another quantity.
- The third argument’s meaning.
- The purpose of the unused second argument.
- Endianness, exact ABI details, alignment requirements, and fault behavior.
- Dependency side effects, ownership rules, and returned-record lifetime.
- Whether callers are permitted to overlap outputs with each other or with record storage.
- Concurrency guarantees and behavior under external mutation.
- The load order of the two fields within each individual subtraction.

The supplied representation uses 32-bit arguments and results for the relevant values and signed 16-bit record reads. It does not support silently generalizing this contract to a wider pointer model.

## Evidence

**Established static evidence:** The supplied body shows one initial 32-bit object-field read, a required first dependency call, a conditional second dependency call, zero/nonzero branching on both results, two signed-field differences, and ordered 32-bit output assignments. The second argument is unused.

**Derived arithmetic fact:** Subtracting two sign-extended signed 16-bit values yields a result in the inclusive range −65,535 to 65,535, which fits in signed 32-bit storage.

**Inferred intent:** The paired field differences resemble extent calculations. No semantic labels or units are established.

**Verification limit:** No runtime tests have been performed. The acceptance examples describe expected behavior under stated conditions; they are not observed executions.
