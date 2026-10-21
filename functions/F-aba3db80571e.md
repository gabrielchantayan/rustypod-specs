<!-- Trusted provenance: F-aba3db80571e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aba3db80571e

## Purpose

F-aba3db80571e conditionally copies one four-byte value from a fixed offset within a caller-designated memory region to a caller-designated output location.

Two selector values are accepted: **0x3691** and **0x36DD**. For either accepted selector, the function reads the four-byte value at byte offset **0x288** from the supplied base, writes that value to the output location, and returns **1**. For other selectors, it returns **0** without reading that source location or writing the output.

This is a selector-gated transfer of an opaque four-byte value. The supplied evidence does not establish what that value represents. In particular, it does not establish that the value is an integer, pointer, handle, status word, or owned resource.

The interpretation as a property-query operation is plausible intent, not an established semantic requirement. The observable contract is the conditional transfer described below.

## Interface

The interface has three ordered inputs and one four-byte return value.

| Position | Input | Contract |
|---|---|---|
| First | Base value | Used to form the source address by adding the byte offset 0x288. It is not dereferenced on a rejected selector. |
| Second | Selector | An integer selector, represented as an `int` in the supplied recovered interface. The accepted values are 0x3691 and 0x36DD. |
| Third | Output location | Address of a writable four-byte destination. It is dereferenced only on an accepted selector. |

The return value is four bytes wide:

- **1:** The selector was accepted and the source-to-output transfer reached its normal return.
- **0:** The selector was rejected; no source read or output write was performed by this function.

A successful transfer is not conditioned on the contents of the source value. A source value of zero still produces a return value of 1.

### Widths and layout

The source access, destination access, and return value are explicitly four bytes wide in the supplied evidence. The base and selector are represented as integer inputs. A 32-bit integer/address model is consistent with that representation, but the supplied fragment alone does not establish the complete calling convention or executable address model.

The only required source layout is:

| Byte displacement from base | Extent | Use |
|---|---:|---|
| 0x288 | 4 bytes | Value copied for an accepted selector |

Offset 0x288 is **648 bytes**. The accessed field therefore occupies byte offsets 648 through 651 inclusive. No earlier fields are read, and no other part of the region is traversed.

The output is a single four-byte destination. There is no output length argument, capacity check, terminator, or variable-sized result.

### Preconditions for normal completion

For either accepted selector:

- Forming the source address must produce an address usable by the execution environment.
- The four-byte source access must be readable.
- The four-byte destination access must be writable.
- Both accesses must satisfy any applicable alignment and memory-access requirements.
- The caller must provide whatever synchronization is needed if other execution contexts can modify or observe these locations concurrently.

These are conditions for ordinary completion, not validations performed by the function.

For a rejected selector, the function does not require the source or destination to be accessible for its own memory operations, because it performs neither access. This does not establish a universal guarantee about constructing or passing arbitrary pointer representations under an unspecified calling environment.

## Behavior

### Selector decision

The selector is examined before the source read or output write.

The decision has two accepted cases:

1. Selector **0x3691** is accepted directly.
2. Otherwise, the selector is reduced by **0x3600**, and the result is compared with **0xDD**. This admits selector **0x36DD**.

The subtraction and comparison are part of the static evidence. They do not describe indexing into a table, iterating over a range, or retrieving a selector-specific offset. Both accepted selectors use exactly the same source location.

Under ordinary 32-bit modular arithmetic, the second comparison has exactly one matching selector bit pattern: 0x36DD. Under signed arithmetic, the subtraction for 0x36DD is representable and produces 0xDD. The accepted cases therefore do not depend on an overflow interpretation.

The recovered expression does not, by itself, specify language-level behavior for extreme signed inputs whose subtraction would overflow. This limitation is documented under Uncertainties; it is not evidence of an additional accepted selector.

### Accepted selectors

For either accepted selector, the observable memory-access order is:

1. Read the four-byte value at the supplied base plus 0x288.
2. Write that value to the supplied output location.
3. Return the four-byte value 1.

The source value is read before the destination is written. No second source read, post-write validation, or subsequent output adjustment is shown.

The function does not transform, mask, validate, or interpret the copied value. Both selectors produce the same transfer behavior.

The function performs the write even if the destination already contains the same four-byte value. The evidence does not show a comparison that would suppress an unchanged write.

### Rejected selectors

For a selector that fails both acceptance conditions, the function returns the four-byte value 0.

On that path:

- The four-byte source field is not read.
- The output location is not written.
- No default output value is produced.
- No error information is stored elsewhere by any explicit operation in the supplied body.

Consequently, an existing output value remains unchanged by this function on a normally completing rejection path. A caller must use the return value to distinguish a newly copied result from an older output value.

### Aliasing and overlap

No non-aliasing check or restriction is shown.

If the destination denotes exactly the same four-byte location as the source, the function still performs the source read followed by the destination write and returns 1, assuming valid accesses.

If the locations partially overlap, the visible order remains a complete four-byte read followed by a four-byte write of the value read. This is not a multi-element forward or backward copy. Exact behavior for misaligned or otherwise unsupported overlapping accesses remains dependent on the execution environment.

A destination may also point into some other part of the region designated by the base. In that case, the destination write modifies that region through the supplied output address. There is no preservation guarantee for memory that the caller explicitly designates as the destination.

## Side effects

The only explicit external-memory modification is the four-byte output write on an accepted selector.

The source access is a read. There is no separate write through the base address, except where the destination aliases memory in the source region.

No allocation, deallocation, resource acquisition, resource release, reference-count change, or ownership transfer is shown. The function neither follows nor releases a resource that the copied bits might happen to represent. Ownership and lifetime semantics of that value are **documented_only**.

There are no explicit global-memory accesses and no dependency calls in the supplied body. No logging, callback, lock, or I/O operation is shown.

The function provides no explicit null checks, bounds checks, alignment checks, or fault recovery. An invalid accepted-path access is therefore not represented as a normal return of 0. The supplied evidence does not specify the resulting trap, exception, signal, or other platform response.

No atomicity, memory-ordering, or thread-safety guarantee is established. The stated read-before-write order describes the operations visible in the supplied function, not a synchronization protocol for concurrent callers or observers.

If either location is memory-mapped hardware or another side-effectful memory region, the indicated read or write may itself have external effects. The function does not distinguish such memory from ordinary storage.

## Dependencies

No direct function dependencies are present in the supplied body.

In particular, no opaque dependency ID is required for the source load, destination store, or selector arithmetic: these are operations of F-aba3db80571e itself.

The following environmental requirements are not identified callees:

- A calling convention capable of supplying the three inputs and receiving the four-byte result.
- An address model in which the source address can be formed from the base and byte offset.
- Memory-access semantics supporting the source and destination operations.

No guarantees about unknown callees are needed, because no callee is shown. The absence of calls does not supply missing information about the surrounding program’s ownership model or concurrency rules.

## Acceptance examples

These examples are conditional behavioral requirements derived from static evidence. **No runtime tests have been performed.**

| Conditions | Required outcome |
|---|---|
| Selector is 0x3691; source and destination accesses are valid; source contains four-byte value V. | Read V from base + 0x288, write V to the output, return 1. |
| Selector is 0x36DD under the same valid-memory conditions. | Perform the same transfer from the same offset and return 1. |
| Selector is 0x3691; the source value is zero; output initially contains a nonzero value. | Write four zero bytes to the output and return 1. |
| Selector is 0x36DD; output already contains the source value. | Still perform the source read and output write; return 1. |
| Selector is 0x3690, 0x3692, 0x36DC, or 0x36DE. | Return 0 without reading the source field or writing the output. |
| Selector is 0; output initially contains value W. | Return 0 and leave W unchanged by this function. |
| Selector is rejected and the supplied source location is inaccessible. | Perform no source access; source inaccessibility alone does not cause a fault through this function’s explicit operations. |
| Selector is accepted and destination exactly equals base + 0x288. | Read the four-byte value, write it back to the same location, and return 1 if both accesses complete. |
| Selector is accepted but the source or destination access is invalid. | No normal error-return behavior is established. Returning 0 is not the specified response to such a fault. |

The two accepted selectors are alternatives, not stages. Neither requires a prior call with the other selector, and no persistent selector state is shown.

## Uncertainties

The following matters are **documented_only** because the supplied evidence does not resolve them:

- **Domain meaning:** The meaning of the selectors and copied field is unknown.
- **Complete ABI:** Register assignment, stack layout, pointer width, and calling-convention details are not supplied.
- **Extreme signed arithmetic:** The recovered integer notation does not establish the execution model for overflowing subtraction on rejected extreme inputs. No additional accepted case is evidenced.
- **Address overflow:** The behavior of base-plus-offset arithmetic outside the valid address range is unspecified.
- **Alignment and faults:** Required alignment, unaligned-access support, and fault delivery are not established.
- **Ownership:** The copied value may encode a resource, but no ownership or lifetime rule can be derived.
- **Concurrency:** There is no established synchronization, atomic access guarantee, or stable-snapshot guarantee under concurrent modification.
- **Side-effectful memory:** The intended use of ordinary memory versus device memory is unknown.

No uncertainty above changes the established behavior for the two accepted selectors when address formation and memory accesses are valid.

## Evidence

The contract is based solely on the supplied static function body.

Established static evidence includes:

- Three inputs and a four-byte return representation.
- A direct selector comparison with 0x3691.
- A secondary subtraction by 0x3600 and comparison with 0xDD.
- One four-byte source read at byte offset 0x288.
- One four-byte destination write using the value read.
- Return value 1 after the transfer and 0 on rejection.
- No explicit calls, allocation operations, or other external-memory accesses.

The characterization as a selector-gated field query is inferred intent. Domain semantics, ownership, and platform behavior beyond the visible operations remain unknown.

No runtime execution, instrumentation, or empirical validation has been performed.
