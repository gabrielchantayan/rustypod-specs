<!-- Trusted provenance: F-ff29814bf4cc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ff29814bf4cc

## Purpose

F-ff29814bf4cc initializes selected fields in a derived memory region, constructs a 32-entry linked structure, and invokes dependencies using a result stored in that region.

The established contract concerns memory accesses, dependency arguments, call ordering, and return behavior. Interpreting the region as a pool, the entries as a free list, or the stored result as a registration handle is inferred intent, not established dependency semantics.

The function first reads a 32-bit global value and writes that value to the caller-supplied address. It then obtains an anchor from a dependency and derives the region base from that anchor. The derived region is not established to coincide with the caller-supplied address.

On ordinary completion, the function returns the derived region base as a 32-bit integer. A particular dependency result, exactly −1, selects a call annotated as nonreturning.

No runtime tests have been performed.

## Interface

### Input and initial global access

The function accepts one pointer to storage whose first four bytes must be writable.

Before any dependency call, the function:

1. Reads a 32-bit value from a global storage location.
2. Stores the value read into the first four bytes at the input address.

The supplied evidence does not establish the global’s numerical value, semantic purpose, immutability, or stability across invocations. The stored value is the value observed by this read, not an independently established constant.

The function also forms an address 684 bytes, or `0x2AC`, beyond the input address and passes it to F-72538fc0941d. Any further storage, alignment, or lifetime requirements imposed by that dependency are unknown.

There is no visible null, size, alignment, or address-validity check.

### Address terminology and arithmetic

This contract uses:

- **Input address:** the caller-supplied pointer.
- **Anchor:** the 32-bit address-shaped result returned by F-72538fc0941d.
- **Region base:** the anchor minus `0x2AC`.

The region base derives from the dependency’s result, not directly from the input address. It equals the input address only if the returned anchor equals the input address plus `0x2AC`.

The visible representation uses 32-bit integer/address values and four-byte words. No overflow or address-wrap check is present. The evidence does not establish portable language-level overflow semantics or behavior for invalid addresses. The ordinary behavioral contract requires the computed accesses to designate valid storage.

### Output

If the continuation dependencies return normally, the function returns the region base as a 32-bit integer.

This is an address-shaped result. There is no separate success status or explicit error-return branch. The −1 used to select the failure call is a dependency result, not a designated return value of this function.

### Region layout

Offsets are relative to the region base.

| Offset | Width or extent | Established use |
|---|---:|---|
| `0x004` | Nominal 680-byte range | Address passed to an external operation with size `0x2A8` |
| `0x018` | 4 bytes | Head word read and replaced during link construction |
| `0x02C + 20 × k`, for `k = 0…31` | 4 bytes at each address | Entry link word |
| `0x2C8` | 4 bytes | Provisional all-one value, then a dependency result |
| `0x2CC` | 4 bytes | Stored region base |

The nominal external-operation range extends from offset `0x004` through `0x2AB`, inclusive. It includes the head word and every link word. The fields at `0x2C8` and `0x2CC` lie outside that range.

Twenty-byte entry spacing does not establish initialization of twenty bytes per entry. The loop writes only the first four bytes at each entry address.

The relationship between the input address and these offsets is unknown. The initial input-word store can therefore overlap a region field, including the head word.

## Behavior

### Initial read and write

The first visible memory operation is the 32-bit global read. The function then writes the value read to the input address, before invoking F-72538fc0941d or any other dependency.

The function does not save or restore the input’s previous first word. A later failure does not visibly undo this write.

The global read is significant independently of the store: different invocations may observe different values. Nothing supplied establishes that dependencies, other callers, or other execution contexts cannot modify the global.

### Anchor acquisition and preliminary region writes

F-72538fc0941d receives the input address plus `0x2AC`.

After it returns, the function derives the region base by subtracting `0x2AC` from the returned anchor. No validation of the result is visible.

The next observable region writes occur in this order:

1. Store the 32-bit all-one bit pattern at offset `0x2C8`.
2. Store the 32-bit region base at offset `0x2CC`.

The all-one bit pattern represents −1 under the signed interpretation used later. At this point it is a provisional field value; no failure decision has yet occurred.

These stores target the derived region even when it differs from the input address.

### External operation

An external dependency whose opaque function ID is not supplied receives two arguments:

1. Region base plus `0x004`.
2. Integer 680, or `0x2A8`.

No return value from this operation is used.

Its semantics are unknown. The argument shape does not establish clearing, copying, allocation, release, or confinement of its effects to the nominal range.

There is no separate region-relative initialization of the head word before this call. However, the initial input-word store can have initialized or overwritten that word if the addresses alias. Its value at loop entry may reflect:

- Preexisting memory contents.
- The initial input-word store.
- Effects of F-72538fc0941d.
- Effects of the external operation.

For example, if the anchor equals the input address plus `0x294`, then the region base equals the input address minus `0x018`. The head at region offset `0x018` consequently coincides with the input address. In that case, the initial global-derived store targets the head, though later dependency effects may replace it before the loop reads it.

### Link construction

The function visits exactly 32 entry addresses in increasing order, beginning at region offset `0x02C` and advancing by 20 bytes each iteration.

Each iteration has these observable memory effects, in order:

1. Read the four-byte head word at offset `0x018`.
2. Store the value read into the current entry’s four-byte link word.
3. Store the current entry’s address into the head word.

The loop does not dereference the value obtained from the head. It copies that value into a link word.

Let the **initial head** mean the value read during the first iteration. With no interfering mutation during construction:

- The entry at offset `0x02C` receives the initial head.
- Each subsequent entry links to the immediately preceding entry.
- Immediately after construction, the head contains the address at offset `0x298`.
- Following the constructed links moves through entry addresses in descending 20-byte steps, ending at the initial head value after the first entry.

The last link-word store covers offsets `0x298` through `0x29B`.

A zero-terminated chain is established only if the initial head is zero. A nonzero initial head is neither validated nor normalized.

The counter is unsigned and 32 bits wide. Each increment is followed by a mask of `0xFFFEFFFF`, which clears bit 16. The reached counter values are 1 through 32, so the mask changes none of them. The loop executes 32 iterations, producing 32 head reads, 32 link-word stores, and 32 head stores.

These relationships describe the state immediately after the loop. Subsequent opaque dependencies are not established to preserve it.

### Result acquisition and failure selection

After link construction, F-64595c3343d2 is called without explicit arguments.

Its 32-bit result becomes the first argument to F-c930571cdede. The complete argument order is:

1. The result of that F-64595c3343d2 invocation.
2. Integer 1.
3. The region base.
4. Integer 1.

After F-c930571cdede returns, the function writes its 32-bit result to region offset `0x2C8`, replacing the provisional all-one value.

The function compares the returned result itself with signed −1. It does not reload the stored field for this comparison.

If the result is exactly −1, F-a817d8d58c10 is called without explicit arguments. The supplied annotation identifies this dependency as nonreturning. Under that annotation, execution does not reach the second F-64595c3343d2 invocation, the final dependency call, or the ordinary return.

No broader negative-value check is present. Zero, positive results, and negative results other than −1 select continuation, provided execution otherwise proceeds normally.

### Final call and return

On continuation, F-64595c3343d2 is called again. This is a distinct invocation; the earlier result is not reused.

After that invocation returns, the function reads the four-byte field at region offset `0x2C8`.

F-23aadb16b673 then receives:

1. The result of the second F-64595c3343d2 invocation.
2. The freshly read field value.

This ordering matters. The second no-argument call may affect memory, so the field value supplied to F-23aadb16b673 is not guaranteed to equal the result previously stored from F-c930571cdede. The function performs no second −1 check.

Any return value from F-23aadb16b673 is ignored. If it returns normally, F-ff29814bf4cc returns the previously derived region base.

## Side effects

The directly established memory effects are:

- One 32-bit global read.
- One four-byte store at the input address.
- One provisional store at region offset `0x2C8`.
- One region-base store at offset `0x2CC`.
- Thirty-two head reads, thirty-two link-word stores, and thirty-two head stores.
- One replacement store at offset `0x2C8`.
- On continuation, one field read after the second F-64595c3343d2 call.

The input store may alias a later region access. These are ordered operations, not a guarantee that all destinations occupy distinct storage.

Opaque dependencies may introduce additional effects. No dependency is established to preserve globals, region contents, ownership, or resource state.

No explicit allocation, deallocation, reference-count operation, or ownership transfer is established by the visible body. Passing the region base to a dependency does not prove transfer of ownership; returning it does not prove acquisition of ownership by the caller.

No rollback is visible. By the time F-c930571cdede can report −1, the initial store, preliminary region stores, external operation, and link construction have already occurred.

No synchronization or atomic-access guarantee is visible. Concurrent or reentrant mutation is outside the established guarantees.

## Dependencies

| Opaque ID | Established interaction | Unresolved semantics |
|---|---|---|
| F-72538fc0941d | Receives input address plus `0x2AC`; returns the anchor | Relationship to input, validation, failure representation, ownership, memory effects |
| F-64595c3343d2 | Called once before F-c930571cdede and again on continuation; each call has no explicit arguments | Result meaning, result stability, global and memory effects |
| F-c930571cdede | Receives a 32-bit result, 1, region base, and 1; returns the value stored and compared with −1 | Meaning of arguments and result, ownership, registration or resource effects |
| F-a817d8d58c10 | Called for exactly −1; annotated nonreturning | Termination or transfer mechanism, diagnostics, cleanup |
| F-23aadb16b673 | Receives the second no-argument result and a subsequently loaded field value | Whether it commits, releases, registers, or otherwise changes state |

One external operation lacks an opaque ID in the supplied dependency map. Its established interaction is the call with region base plus `0x004` and size 680. Its identity and semantics remain documented_only; no invented function ID or clearing behavior is assigned.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests.

### Global values differ across invocations

Assume two invocations read different 32-bit values from the global.

Each invocation must store its own observed value into its input’s first word before its first dependency call. The contract does not require the two stored values to match.

Later effects may overwrite those words; this criterion concerns the initial ordered store.

### Initially zero head

Assume valid addresses, returning dependencies, a zero initial head, and no mutation of the constructed links during or after construction.

Immediately after the loop:

- The head points to region offset `0x298`.
- The 32 entries form a descending-address chain with 20-byte steps.
- The entry at offset `0x02C` contains zero.

If F-c930571cdede returns 7, the function stores 7, skips the failure call, performs the continuation calls, and returns the region base if those calls return.

### Initial store aliases the head

Assume the anchor equals the input address plus `0x294`. Also assume the global read yields `0x12345678` and intervening dependencies do not change the head.

The input address and region-relative head address coincide. The first loop iteration must therefore store `0x12345678` into the entry at offset `0x02C`.

If an intervening dependency changes the head instead, the first entry receives that changed value. The initial store alone does not determine loop-entry contents.

### Nonzero initial head

Assume the first head read yields a nonzero value.

The first entry receives that exact 32-bit value. The remaining entries link backward through the newly constructed entries. No check rejects or dereferences the initial value, and no zero termination is guaranteed.

### Exact failure versus another negative result

If F-c930571cdede returns −1, the function stores the all-one bit pattern and invokes F-a817d8d58c10. Under the nonreturn annotation, no continuation calls or ordinary return follow. Earlier effects are not visibly undone.

If it returns −2 instead, the failure call is skipped. Continuation occurs if subsequent dependencies return normally.

### Mutation before the final field read

Assume F-c930571cdede returns 7, then the second F-64595c3343d2 invocation changes the field at `0x2C8` to 9.

F-23aadb16b673 must receive 9 as its second argument. If the replacement were the all-one bit pattern instead, that value would still be forwarded without another failure check.

### Region differs from input

Assume F-72538fc0941d returns an anchor different from input plus `0x2AC`, with all resulting accesses valid.

Subsequent region-relative accesses and the return value follow that anchor minus `0x2AC`. The initial global-derived store still occurred at the original input address.

## Uncertainties

The following remain documented_only:

- The global’s value, purpose, mutability constraints, and stability between reads or invocations.
- The external operation’s identity and semantics, including whether it establishes a zero head.
- The relationship between input and derived region, including overlap and aliasing.
- The meanings of the literal arguments equal to 1.
- The semantic role of the stored result and the final dependency call.
- Dependency allocation, release, ownership, lifetime, and shared-state effects.
- Whether the nonreturn annotation holds in every environment. If that dependency returns normally, the visible continuation remains after its call; no additional guard or rollback is shown.
- Required alignment, complete object extent, and behavior for invalid or overflowing address calculations.
- Contents of entry bytes beyond the four-byte link words.
- Portable ABI, byte order, and language-level pointer-arithmetic rules beyond the visible 32-bit representation.

## Evidence

This contract uses only the supplied function body, dependency-ID map, and nonreturn annotation. No dependency implementations, runtime traces, or test results were supplied.

Established static evidence supports the global read before the input store, the store before dependency calls, address calculations, field widths and offsets, ordered accesses, 32-iteration construction, exact −1 comparison, final field reload, and returned region base.

The head-alias example follows arithmetically: an anchor at input plus `0x294` places the region base at input minus `0x018`, making region offset `0x018` equal the input address.

Pool initialization, free-list intent, registration, clearing, ownership transfer, and resource release remain interpretations or unknowns, not established behavior.
