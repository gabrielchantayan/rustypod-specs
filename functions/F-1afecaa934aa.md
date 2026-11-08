<!-- Trusted provenance: F-1afecaa934aa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1afecaa934aa

## Purpose

F-1afecaa934aa initializes selected fields of a mutable context, processes a sequence of 16-bit table entries through two dependencies, and invokes a final dependency when the loop terminates normally.

The established result is a 32-bit scalar restricted to zero or one. Zero requires both a zero result from finalization and a strict unsigned comparison between two context fields. Every dependency-triggered early exit returns one. The available evidence does not establish that these values mean success and failure.

The field accesses suggest sequence processing with mutable output or linkage state. That is inferred intent, not an application-level contract.

## Interface

### Inputs and result

The function receives:

- A context address represented by a 32-bit address-bearing argument.
- An opaque 32-bit value forwarded unchanged to F-bae8961d34e4 on every admitted iteration.

It returns a 32-bit value of zero or one. Nonzero dependency results are reduced to one rather than propagated.

### Context layout

Offsets are byte offsets from the context address. Field descriptions identify observed roles, not domain types or ownership.

| Offset | Width | Observed role |
|---|---:|---|
| `0x04` | 32 bits | Parameter used to construct a mask by subtracting one |
| `0x24` | 32 bits | Source for initialization of `0x28` |
| `0x28` | 32 bits | Initialized to `0x24` minus 32; used in the final unsigned comparison |
| `0x2C` | 32 bits | Address-bearing value copied during initialization, conditionally decreased by four, and used in the final comparison |
| `0x34` | 32 bits | Cleared during initialization |
| `0x4C` | 32 bits | Captured per iteration for bitwise and signed-range tests |
| `0x50` | 32 bits | Signed lower bound |
| `0x54` | 32 bits | Signed upper bound |
| `0x58` | 16 bits | Cleared during initialization |
| `0x5A` | 8 bits | Cleared during initialization |
| `0x5B` | 8 bits | Cleared during initialization |
| `0x5C` | 32 bits | Address-bearing value initialized from `0x2C`, then read during processing |
| `0x60` | 32 bits | Cleared during initialization |
| `0x64` | 32 bits | Address-bearing value cleared before each processing call |
| `0x68` | 8 bits | Cleared before each processing call |
| `0x84` | 16 bits | Signed iteration limit, read at every loop test |
| `0x90` | 32 bits | Address of consecutive 16-bit table entries |

Indirect accesses are:

| Address source | Additional offset | Width | Access |
|---|---:|---:|---|
| Value captured from `0x2C` during initialization | `0x08` | 32 bits | Store the captured address value |
| Current value of `0x64` | `0x0C` | 32 bits | Conditional equality-test read |
| Current value of `0x5C` | `0x0C` | 32 bits | Conditional equality-test read |
| Value captured from `0x5C` before F-ecde43d0a2b1 | `0x1C` | 32 bits | Conditional post-call store |
| Current table address from `0x90` | Twice the iteration index | 16 bits | Entry read, then reread after F-bae8961d34e4 returns zero |

### Preconditions

All storage reached by the selected execution path must support the indicated accesses. In particular:

- Initialization always writes through the address captured from `0x2C`, without checking it.
- Each admitted iteration requires a readable table entry after the per-iteration clears.
- Reaching the indirect equality comparison requires readable storage at both object addresses plus `0x0C`.
- Reaching the link store requires writable storage at the saved address plus `0x1C`.

The function establishes no pointer validation, capacity validation, synchronization, or recovery from inaccessible memory.

No non-aliasing guarantee is present. Context fields, table storage, and indirectly addressed objects may overlap unless an unavailable external contract forbids it. Consequently, each read observes the state left by preceding writes and calls, not necessarily the entry-time state.

## Behavior

### Initialization and its ordering

Before the first iteration-limit read or dependency call, the function performs these memory operations in order:

1. Clears the 32-bit field at `0x60`.
2. Clears the byte at `0x5B`.
3. Clears the byte at `0x5A`.
4. Reads the 32-bit value at `0x24`, subtracts 32, and stores the result at `0x28`.
5. Clears the 32-bit field at `0x34`.
6. Reads the address-bearing value at `0x2C`.
7. Copies that captured value to `0x5C`.
8. Stores that same captured value at the address eight bytes beyond it.
9. Clears the 16-bit field at `0x58`.

The function also establishes an internal iteration index of zero and a carried 16-bit boundary value of zero.

The indirect initialization store uses the captured value even if copying it to `0x5C` changes aliased storage. It does not reread `0x2C` to select that store’s address or value.

The subtraction used to initialize `0x28` has 32-bit signed operands in the supplied representation. No underflow check or saturation is visible. Behavior outside the signed representable range remains documented_only; the evidence does not justify imposing a portable wrapping rule.

Initialization can change the iteration limit before its first read. For example, if the value captured from `0x2C` equals the context address plus `0x7C`, the indirect store targets context offsets `0x84` through `0x87`, overlapping the 16-bit limit. Thus, an entry-time zero or negative limit does not by itself guarantee that iteration is skipped.

### Iteration admission

At each loop test, the function reads `0x84` as a signed 16-bit value and compares it with the internal index.

- A limit less than or equal to the index terminates the loop and begins finalization.
- A limit greater than the index admits the current iteration.

The first such read occurs after all initialization writes. Therefore, a zero or negative value observed by that first read causes no table-entry processing. This guarantee concerns the value actually read, not an earlier value present at function entry.

The limit is reread at every loop test. Dependencies and aliased stores can therefore affect subsequent admission decisions.

The index starts at zero and advances only after F-ecde43d0a2b1 returns zero. Since the limit is signed 16-bit, an admitted index is at most 32766. The index may advance to 32767, but no signed 16-bit limit can admit another iteration at that index. The visible index progression does not approach 32-bit signed overflow.

### Per-iteration processing

For every admitted iteration, the function:

1. Clears the byte at `0x68`.
2. Clears the 32-bit field at `0x64`.
3. Reads the table address from `0x90`.
4. Reads the 16-bit entry at twice the current index.
5. Calls F-bae8961d34e4 with the context, the carried boundary, that entry, and the opaque second input.

The table address and entry are obtained after the clears. Aliasing may make that ordering observable.

The boundary and entry have 16-bit source widths. The exact argument-extension convention is documented_only.

If F-bae8961d34e4 returns nonzero, the function immediately returns one. It does not update the carried boundary, evaluate the decrement conditions, invoke F-ecde43d0a2b1, or finalize.

If F-bae8961d34e4 returns zero, the function rereads `0x90` and the entry at the same index. It interprets this second entry as signed 16-bit, adds one, and assigns the result to the carried 16-bit boundary.

The reread may select different storage or a different value from the entry passed to the dependency. There is no table-preservation assumption.

For signed entry values from −32768 through 32766, the incremented boundary is representable as signed 16-bit. An entry of 32767 produces an intermediate value of 32768, requiring an out-of-range narrowing conversion. Retaining the low 16 bits, yielding −32768 under a two’s-complement interpretation, is plausible but not independently established by the supplied representation.

### Conditional decrement

After updating the carried boundary, the function captures the 32-bit value at `0x4C`. It decreases the current value of `0x2C` by four only if all of these conditions pass:

1. The captured bit pattern AND the mask formed from `0x04` minus one is zero.
2. The signed 32-bit value at `0x50` is less than or equal to the captured value interpreted as signed 32-bit.
3. That signed captured value is less than or equal to the signed 32-bit value at `0x54`.
4. The address-bearing value at `0x64` is nonzero.
5. The 32-bit values at offset `0x0C` from the objects addressed by `0x64` and `0x5C` are equal.

The mask subtraction is unsigned 32-bit arithmetic. In particular, a mask parameter of zero produces `0xFFFFFFFF`.

These conditions short-circuit in the listed order. A failed condition prevents evaluation of subsequent conditions and their associated reads. The evidence does not establish a relative evaluation order between the two operands of the final equality comparison.

The nonzero check of `0x64` precedes its use as an object address. There is no corresponding nonzero check for `0x5C`. The equality operation uses address-field reads in addition to those required by the preceding nonzero test; it does not establish a general pointer-stability guarantee.

When all conditions pass, the function reads the current `0x2C`, subtracts four, and writes the result back there. This is not a decrement of the initialization snapshot. No capacity check, lower-bound check, or saturation accompanies the operation. The supplied representation uses signed 32-bit arithmetic here; out-of-range behavior remains documented_only.

The mask condition is not generally a divisibility test. Such an interpretation would require additional restrictions on `0x04`, notably an appropriate power-of-two value.

### Post-processing and link store

After the decrement stage, whether or not a decrement occurred, the function captures the current address-bearing value at `0x5C` and calls F-ecde43d0a2b1 with the context.

A nonzero result causes an immediate return of one. The index is not advanced, the post-call link store is skipped, and finalization is not invoked.

On a zero result:

1. The internal index advances by one.
2. The function reads `0x64` and tests it against zero.
3. If nonzero, it reads `0x64` for the stored value and writes that 32-bit value at offset `0x1C` from the address captured before the call.
4. The function returns to the iteration-limit test.

The destination is a pre-call snapshot. The nonzero test and stored value use post-call reads. Changing `0x5C` during the dependency does not redirect this store.

The two post-call `0x64` accesses are distinct in the supplied representation. Their observation under concurrent or externally changing memory is not specified.

### Finalization and result

When a loop test rejects further iteration, F-d2ecc918a7b7 is called with the context.

- If it returns nonzero, the function returns one without evaluating the final field comparison.
- If it returns zero, the function compares the current values at `0x2C` and `0x28` as unsigned 32-bit quantities.
- The result is zero exactly when `0x2C` is strictly less than `0x28`.
- Equality or greater unsigned ordering produces one.

These are post-finalization values. No preservation of either field by the final dependency may be assumed. The relative evaluation order of the two comparison-field reads is not established.

## Side effects

Directly visible mutations comprise initialization, per-iteration clears, the conditional decrement, and the conditional indirect link store.

No rollback occurs on a dependency-triggered early return. Earlier writes, including dependency mutations, remain subject to whatever state the dependency actually leaves behind.

The visible body contains no direct allocation, deallocation, global-variable access, or I/O operation. Dependencies may perform any of these; their behavior is unknown.

Address copies and stores do not establish ownership acquisition or transfer. Reference counting, resource release, and object-lifetime guarantees are documented_only. Valid storage must remain available for every subsequent access, including stores through addresses saved before a dependency call.

## Dependencies

| Opaque ID | Invocation and inputs | Result handling |
|---|---|---|
| F-bae8961d34e4 | Once per admitted iteration, after clears and the first table-entry read; receives context, carried 16-bit boundary, selected 16-bit entry, and opaque 32-bit input | Nonzero causes immediate return of one; zero permits the entry reread |
| F-ecde43d0a2b1 | After a zero result from F-bae8961d34e4 and the decrement stage; receives context | Nonzero causes immediate return of one; zero permits index advancement and the conditional link store |
| F-d2ecc918a7b7 | Once when a loop test terminates processing; receives context | Nonzero yields one; zero enables the final unsigned comparison |

Dependency internals, ownership effects, global-state preservation, and context-preservation guarantees are documented_only. Subsequent reads must use the state actually left by each call.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Each assumes valid storage, completed dependency calls, and the stated values at the specified observation points.

1. **Zero at the first loop test.**  
   If the signed limit read after initialization is zero, neither per-iteration dependency runs. Finalization still runs. If it returns zero and leaves `0x2C = 0x1000` and `0x28 = 0x1004`, the result is zero.

2. **Negative at the first loop test.**  
   If that first signed limit read yields −1, processing proceeds directly to finalization. An entry-time value of −1 establishes this outcome only if initialization leaves the limit negative or zero.

3. **Initialization overlaps the limit.**  
   If the captured initialization address equals context plus `0x7C`, the indirect initialization store writes four bytes beginning at `0x84`. Even if the entry-time limit was zero, iteration occurs if the signed 16-bit limit subsequently read is positive. The exact resulting limit depends on the address bits and storage representation.

4. **Final equality is insufficient.**  
   If finalization returns zero and leaves `0x2C` and `0x28` equal, the result is one.

5. **Unsigned final ordering.**  
   If finalization returns zero and leaves `0x2C = 0xFFFFFFFC` and `0x28 = 0x00001000`, the result is one. Treating the first value as signed −4 would incorrectly reverse the ordering.

6. **Immediate processing exit.**  
   If the first F-bae8961d34e4 call returns nonzero, the result is one. Initialization and that iteration’s clears have occurred; neither other dependency runs.

7. **Inclusive signed bounds.**  
   With mask parameter four, captured value eight, bounds eight and twelve, nonzero `0x64`, and matching indirect values, the decrement occurs. Captured value twelve also passes. Value nine fails the mask condition and prevents the later conditional reads.

8. **Masking is not general divisibility.**  
   With mask parameter six, the mask is five. Captured value two passes the mask test despite not being a multiple of six. The remaining conditions still determine whether a decrement occurs.

9. **Changed table after processing.**  
   If the entry passed to F-bae8961d34e4 is ten, but the dependency changes the entry selected by the subsequent reread to twenty and returns zero, the carried boundary becomes twenty-one.

10. **Saved destination and updated source.**  
    If `0x5C` addresses object A before F-ecde43d0a2b1, and the dependency changes `0x5C` to B and `0x64` to C before returning zero, stable post-call reads of C cause C to be stored at A plus `0x1C`, not B plus `0x1C`.

11. **Post-processing early exit.**  
    If F-ecde43d0a2b1 returns nonzero, the result is one regardless of whether the current final-comparison fields would permit zero. The link store and finalization are skipped.

## Uncertainties

The following remain documented_only:

- Domain meanings of fields, table entries, indirect objects, and return values.
- Exact calling-convention extension of the 16-bit arguments.
- Out-of-range signed arithmetic and signed narrowing behavior.
- Byte order where overlapping stores determine narrower field values.
- Dependency side effects, global-state preservation, ownership, and object lifetime.
- External restrictions on aliasing among context, table, and indirect storage.
- Alignment requirements, concurrency, reentrancy, and memory-fault behavior.
- Any invariant restricting the mask parameter or guaranteeing available capacity.

These unknowns do not establish validation, cleanup, retries, or additional error reporting.

## Evidence

Established static evidence consists of the supplied function body’s access widths, offsets, ordered stores, conditional reads, dependency arguments, branches, and signed or unsigned comparisons.

The initialization alias case follows from the observed indirect store at captured address plus eight: selecting context plus `0x7C` makes that store begin at context plus `0x84`, before the first limit read.

Sequence-processing terminology and the suggested two’s-complement narrowing outcome are inferences, not established domain or runtime guarantees. No dependency implementation or external caller contract was supplied.

No runtime tests, fault experiments, or execution traces have been performed.
