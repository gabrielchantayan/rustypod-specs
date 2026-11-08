<!-- Trusted provenance: F-e86b88cccc7e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e86b88cccc7e

## Purpose

F-e86b88cccc7e dispatches an operation over a destination described by an element width, row length, row count, and row stride. It has three observable execution paths:

- One delegated operation for a contiguous region whose computed signed byte extent is at most 16,384.
- One delegated operation per row when row length and row stride differ.
- A direct 32-bit pattern store and an external block operation for a larger contiguous region, followed by a remainder operation only if the external operation returns zero.

**Inferred intent:** the function implements a repeated-value fill, with a specialized path for large contiguous regions. The construction of repeated-byte and repeated-halfword patterns supports this interpretation. Neither dependency’s implementation is available, so exact destination contents and complete fill coverage are not established.

The function returns no value or status. Some conditions cause silent rejection, but rejection does not universally imply an absence of earlier memory effects.

## Interface

The function accepts seven positional inputs:

| Position | Contract name | Represented type | Role |
|---|---|---|---|
| 1 | Context | Address represented by a signed 32-bit integer | Passed to F-be4c308983c8; provides a pointer field used on the large contiguous path. |
| 2 | Destination | Address represented by a signed 32-bit integer | Initial destination address. A zero value causes immediate return. |
| 3 | Pattern | Unsigned 32-bit integer | Passed unchanged to F-be4c308983c8; checked and expanded for the large contiguous path. |
| 4 | Element width | Signed 32-bit integer | Multiplier for byte extents and row advancement. |
| 5 | Row length | Signed 32-bit integer | Per-row element count and factor in the contiguous element total. |
| 6 | Row count | Signed 32-bit integer | Row-loop bound and factor in the contiguous element total. |
| 7 | Row stride | Signed 32-bit integer | Compared with row length; multiplied by element width for row advancement. |

These are the supplied representation widths, not an independently established calling convention. No authoritative ABI description is available.

### Context layout

The only context field directly accessed by this function is a 32-bit pointer field at byte offset **56**. The pointer is interpreted as addressing a writable 32-bit word, called the **scratch word** in this contract. That name describes its apparent role, not an established allocation, exclusivity, or ownership rule.

For an accepted large contiguous operation, the observable order is:

1. Read the pointer field.
2. Store a constructed 32-bit pattern through the pointer.
3. Read the pointer field again while preparing the external call.
4. Invoke the external dependency using the second pointer value.

The second read is significant. If the scratch-word store overlaps the pointer field, the pointer supplied to the external dependency can differ from the pointer used for the store.

The source establishes no restriction preventing the scratch word from overlapping the destination, other context storage, or other caller-visible memory.

### Arithmetic quantities

The contract uses these quantities:

- **Element total:** row count multiplied by row length, held in an unsigned 32-bit quantity.
- **Byte extent:** element width multiplied by element total, held in a signed 32-bit quantity for dispatch.
- **Row byte advance:** element width multiplied by row stride.
- **Block count:** byte extent shifted right by six on the large contiguous path.
- **Remainder bytes:** byte extent minus 64 times block count.

The large path requires a positive signed byte extent greater than 16,384. Consequently, within that path, the shift gives floor division by 64, and remainder bytes lie between 0 and 63.

The arithmetic mixes signed multiplication, unsigned multiplication, conversion to a signed representation, and integer-based address advancement. A portable overflow policy is not established. In particular, assigning the row-count–row-length product to an unsigned quantity does not independently establish defined unsigned arithmetic for the preceding signed multiplication.

Unless explicitly discussing represented values, the acceptance examples assume nonoverflowing arithmetic and representable address calculations.

## Behavior

### Zero destination

A zero destination causes immediate return before extent calculations, context-field accesses, scratch-word stores, or dependency calls.

No other input is validated in this case. A nonzero destination is not checked for accessibility, alignment, or available extent.

### Initial calculation and dispatch

For a nonzero destination, the function calculates element total and byte extent before selecting a path.

The dispatch rules are:

| Condition | Selected path |
|---|---|
| Row length differs from row stride | Row loop, regardless of byte extent |
| Row length equals row stride and signed byte extent is at most 16,384 | Single delegated operation |
| Row length equals row stride and signed byte extent exceeds 16,384 | Large contiguous operation |

The threshold is strict: exactly 16,384 takes the single-call path.

The comparison uses the signed byte extent. A negative represented extent does not enter the large path, even if the same bits would denote a large unsigned value.

### Noncontiguous rows

When row length differs from row stride, a positive row count causes one call to F-be4c308983c8 per row.

Each call receives, in order:

1. Original context.
2. Current destination.
3. Original pattern.
4. Original element width.
5. Original row length.

The first call uses the supplied destination. After each normally returning call, including the final call, the function advances its working destination by row byte advance.

For row index \(k\), starting at zero, the destination is the initial address advanced \(k\) times by element width multiplied by row stride, subject to unresolved overflow and address-representation behavior.

Calls are sequential. The next call is not made until the preceding call returns. Any dependency changes to shared memory or global state can therefore affect subsequent calls.

A zero or negative row count produces no calls. This does not bypass the initial extent calculations for a nonzero destination.

The wrapper does not reject unsupported element widths, oversized patterns, negative row lengths, negative strides, overlapping rows, or zero row advancement on this path. Such inputs may be forwarded to the dependency; their ultimate effects are unknown.

### Small or nonpositive contiguous extent

When row length equals row stride and the signed byte extent is at most 16,384, the function invokes F-be4c308983c8 once with:

1. Original context.
2. Original destination.
3. Original pattern.
4. Original element width.
5. Unsigned 32-bit element total.

It then returns.

A zero count is still passed in a call. A negative represented byte extent also selects this path. Neither condition causes independent rejection.

The function performs no direct context-field access or scratch-word store on this path. The width and pattern restrictions described below apply only to the large contiguous path.

### Large contiguous operation: preparation

This path requires equal row length and row stride, together with a signed byte extent greater than 16,384.

The function derives block count and remainder bytes before handling element width. Supported widths are 1, 2, and 4.

#### Element width 1

The pattern must be at most 255: all bits above bit 7 must be zero.

If it is too large, the function returns without accessing the context pointer field, storing the scratch word, or calling a dependency.

Otherwise, the function stores a 32-bit value consisting of four repetitions of the pattern byte. For example, pattern `0xAB` produces the numeric word `0xABABABAB`.

Remainder bytes are used directly as the remainder element count.

#### Element width 2

The pattern must be at most 65,535.

If it exceeds that range, the function returns without a scratch-word store or dependency call.

For an accepted pattern, the function first stores a 32-bit value containing two repetitions of the 16-bit pattern. For example, `0x1234` produces `0x12341234`.

Only after that store does it test whether remainder bytes are even. An odd remainder causes return without a dependency call and without wrapper-issued restoration of the stored word.

An accepted remainder is divided by two to obtain the remainder element count.

Under ordinary nonoverflowing arithmetic, multiplying an element total by two produces an even byte extent, and subtracting a multiple of 64 preserves evenness. The odd-remainder rejection is therefore not reachable under those assumptions, but its position after the store remains part of the static behavior.

#### Element width 4

The function stores the original 32-bit pattern without restricting its value.

After the store, it checks whether remainder bytes are divisible by four. A failed check causes return without a dependency call or wrapper-issued restoration.

For an accepted remainder, the remainder element count is remainder bytes divided by four.

As with width 2, this alignment rejection is unreachable under ordinary nonoverflowing size arithmetic. The ordering of store before check is nevertheless established.

#### Other element widths

Any other width causes return without a context-pointer read, scratch-word store, or dependency call.

This is not a function-wide width restriction. Other widths can reach F-be4c308983c8 through the row loop or single-call contiguous path.

### Large contiguous operation: external call

After accepted preparation, the function calls **F-unmapped-01** with:

| Position | Supplied value |
|---|---|
| 1 | Original destination |
| 2 | Pointer freshly read from context offset 56 |
| 3 | Block count multiplied by 16 |
| 4 | Original unsigned 32-bit element total |
| 5 | Original element width |

The third argument is numerically the count of 32-bit words contained in the complete 64-byte blocks. Its meaning to the dependency is not established.

The return value is tested only for equality to zero:

- **Nonzero:** return without calling F-be4c308983c8.
- **Zero:** advance the working destination by 64 times block count, then call F-be4c308983c8 with the original context, advanced destination, original pattern, original element width, and remainder element count.

The remainder call occurs even when the remainder element count is zero.

The direct scratch-word store precedes the external call. The external dependency may subsequently overwrite that word, the context, or destination memory. Neither return branch restores the earlier word or establishes its continued preservation.

A zero result means only that this wrapper proceeds to the remainder call. It does not establish successful block filling. A nonzero result means only that the wrapper omits that call.

## Side effects

### Direct memory effects

The wrapper can issue one direct 32-bit store through the pointer loaded from context offset 56. It also reads that field once to obtain the store address and again before the external call.

No such direct accesses occur for a zero destination, a row-loop operation, a single-call contiguous operation, or a large-path rejection before pattern storage.

The scratch-word store is not necessarily separate from destination writes. If its address overlaps the destination region, the wrapper itself directly modifies that region before invoking a dependency. Partial overlap can also affect adjacent bytes or other structures according to the supplied address and actual memory layout.

Destination advancement updates only the wrapper’s working address value. It does not write an updated address back to a caller-owned address variable.

### Delegated effects and interference

The intended destination operations are delegated, **except for any overlap with the wrapper’s direct scratch-word store**. Exact dependency reads and writes, including their widths, ordering, coverage, and alignment requirements, remain unknown.

No preservation guarantee applies to context fields, scratch contents, destination contents, or globals across dependency calls. The function does not isolate overlapping storage or restore values afterward.

In particular:

- A scratch store overlapping the context pointer field can change the pointer subsequently supplied to F-unmapped-01.
- An external call can overwrite the scratch word before returning either zero or nonzero.
- A row helper can modify memory that a later row call accesses.
- A remainder helper receives the original context address, not a saved copy of context contents.

### Ownership and errors

The wrapper has no explicit allocation or release operation. Pointer forwarding alone does not establish ownership transfer, retention permission, borrowing duration, or whether dependencies release memory.

The function returns no status. Unsupported widths and oversized patterns are silently rejected only where the large-path checks apply. The external result is consumed locally and is not returned to the caller.

There are no explicit checks for null context, null scratch pointer, valid bounds, or appropriate alignment. The source establishes no recovery from invalid memory access or dependencies that fail to return normally.

## Dependencies

### F-be4c308983c8

This dependency receives five arguments: context, destination, pattern, element width, and count.

It is called:

- Once for each executed strided row.
- Once for a small or nonpositive contiguous extent.
- Once for the remainder after a zero result from F-unmapped-01.

The wrapper does not inspect a return value.

The dependency’s count signedness, actual memory behavior, validation, ownership rules, zero-count handling, and global-state effects are **documented_only**.

### F-unmapped-01

This document-local opaque ID denotes the external dependency called only after accepted large-path preparation. No authoritative mapped ID was supplied.

Its five arguments are specified above. Its returned integer controls only whether the remainder call occurs.

Its operation, memory effects, pointer lifetime requirements, ownership behavior, and error meanings are **documented_only**. No preservation of context or scratch storage is assumed.

## Acceptance examples

Unless otherwise stated, these examples assume valid accessed memory, nonoverflowing arithmetic, representable addresses, and normally returning dependencies. They specify wrapper behavior rather than final fill contents.

| Inputs or condition | Required observation |
|---|---|
| Destination is zero | No dependency call, context-field access, or scratch-word store, regardless of the other inputs. |
| Width 1; row length 8; row count 3; stride 12 | Three F-be4c308983c8 calls with count 8 at destination offsets 0, 12, and 24. |
| Width 4; row length 3; row count 2; stride 5 | Two helper calls with count 3 at offsets 0 and 20. |
| Row length differs from stride; row count is zero or negative | No helper calls; initial extent calculations still precede loop selection. |
| Width 4; row length and stride 64; row count 64 | Byte extent is exactly 16,384. One helper call with count 4,096; no direct scratch store. |
| Equal row length and stride; row count zero | One helper call with count zero. |
| Width 3; row length and stride 10; row count 1 | One helper call with width 3 and count 10. The large-path width restriction does not apply. |
| Width 1; row length and stride 16,385; row count 1; pattern `0xAB` | Store `0xABABABAB`, then call F-unmapped-01 with third argument 4,096 and fourth argument 16,385. If it returns zero, call the helper at offset 16,384 with count 1. |
| Same dimensions; pattern 256 | Return without a scratch store or either dependency call. |
| Width 2; row length and stride 8,193; row count 1; pattern `0x1234` | Store `0x12341234`, then invoke the external dependency with third argument 4,096 and fourth argument 8,193. On zero, call the helper at offset 16,384 with count 1. |
| Width 4; row length and stride 4,112; row count 1 | Byte extent is 16,448. After the external call returns zero, call the helper at offset 16,448 with count zero. |
| Accepted large operation; external result is nonzero | The scratch store occurs before the external call. After its return, issue no remainder call or restoration. The stored value need not survive the external call. |
| Width 3; row length and stride 6,000; row count 1 | Byte extent is 18,000. Return without a scratch store or dependency call. |

Two alias-sensitive examples further constrain the contract:

- **Scratch word at the destination:** an accepted large operation directly writes its constructed 32-bit word into destination memory before the external call. Final destination contents remain dependent on subsequent calls.
- **Scratch word at context offset 56 itself:** the store replaces the pointer field’s bits. The external dependency receives the newly read value, interpreted as its second argument, rather than necessarily receiving the original scratch address. Whether that argument is valid for the dependency is unresolved.

## Uncertainties

The following remain **documented_only**:

- ABI details beyond the supplied 32-bit representations.
- Overflow semantics for signed dimension products, row advancement, and address calculations.
- Conversion behavior when an unsigned byte product is not representable as a signed 32-bit value.
- The helper’s interpretation of the count argument.
- Valid destination bounds, alignment, accessibility, and overlap constraints.
- Scratch-pointer validity, alignment, lifetime, and any caller-required alias restrictions.
- Byte order of the stored 32-bit numeric word.
- Actual destination contents or complete write coverage from either dependency.
- Whether dependencies retain, release, or otherwise take ownership of addresses.
- Preservation or mutation of context, scratch storage, and globals by dependencies.
- External return-value meanings beyond zero versus nonzero dispatch.
- Behavior after access faults or non-normal dependency termination.

For predictable intended-region behavior, valid memory, appropriate alignment, nonnegative dimensions, and nonoverflowing calculations are reasonable inferred caller requirements. They are not uniformly enforced by this wrapper.

No disjointness between destination, scratch storage, and context may be assumed from the supplied source.

## Evidence

**Established static evidence:** the supplied function body determines the zero-destination guard, arithmetic representations, signed threshold, row-loop order, width and pattern checks, direct store order, pointer reload, dependency arguments, and zero-result remainder dispatch.

**Dependency identification:** the supplied map identifies F-e86b88cccc7e and F-be4c308983c8. F-unmapped-01 is a document-local opaque label for the external dependency whose authoritative ID was not supplied.

**Inferred intent:** repeated-value filling and a large-region optimization based on 64-byte blocks. These interpretations do not establish dependency behavior.

**Unresolved behavior:** exact memory effects, ownership, overflow policy, and dependency preservation guarantees remain explicitly documented rather than assumed.

No runtime tests have been performed.
