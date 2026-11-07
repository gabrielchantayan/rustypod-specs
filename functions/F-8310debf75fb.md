<!-- Trusted provenance: F-8310debf75fb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8310debf75fb

## Purpose

F-8310debf75fb operates on an object, a mutable position representation, and an opaque boundary value. It first publishes an object-held header address to an output slot. It then either:

- Calls a cleanup-like dependency and resets selected object and header fields; or
- Repeatedly captures a position value, invokes an advancement dependency, invokes a processing dependency, and publishes a processing result until a comparison returns exactly one.

The reset pattern suggests an empty-state representation for a sentinel-based container. That is inferred intent, not an established type or ownership contract.

The return is a 64-bit packing of two 32-bit values. It is not established as a Boolean, status code, or single pointer.

This contract distinguishes direct behavior from effects that opaque dependencies may introduce. In particular, values in address-exposed temporary slots need not survive intervening dependency calls unchanged.

## Interface

### Arguments

The represented address model uses 32-bit addresses and 32-bit words.

| Argument | Meaning established by direct use |
|---|---|
| Output address | Address of a writable 32-bit output slot |
| Object address | Address used to access the header-reference and state fields |
| Position address | Address passed to comparison and advancement dependencies; its first 32-bit word is read for each entered iteration |
| Boundary value | Opaque 32-bit value passed to the comparison dependency |

The boundary value is not directly dereferenced by F-8310debf75fb. Its interpretation belongs to F-eac694802fc1.

### Directly accessed storage

| Storage | Byte offset | Width | Use |
|---|---:|---:|---|
| Output | 0 | 32 bits | Initial header publication and later path-dependent publication |
| Object | 16 | 32 bits | Header address |
| Object | 20 | 32 bits | Nonzero state test and reset to zero |
| Header | 4 | 32 bits | Cleanup argument and reset to zero |
| Header | 8 | 32 bits | Initial comparison operand and reset destination |
| Header | 12 | 32 bits | Reset destination |
| Position | 0 | 32 bits | Capture before advancement |

“Header” means the address obtained from object offset 16 at the relevant access. The function repeatedly reads that field; it does not establish one immutable header address for the entire invocation.

### Temporary storage and return value

Two temporary 32-bit slots determine the ordinary-path return:

- A **result slot**, initialized to the object address.
- A **captured-value slot**, initialized to the output address.

Each processing call receives both slots by address. Their contents can therefore be modified by that call. Retention of those addresses can also allow later dependencies to modify them during this invocation.

The 64-bit result is composed by bit position:

| Path | Upper 32 bits | Lower 32 bits |
|---|---|---|
| Fast reset | Header value captured immediately before the final output write | Output address |
| Ordinary path | Final result-slot contents | Final captured-value-slot contents |

The fast-path upper half is the value used for the final output write, not a new header lookup after that write.

If the ordinary path executes no iterations, the two return slots have not been exposed through the processing call: the upper half is the object address and the lower half is the output address.

### Preconditions

Every executed memory access must have accessible storage of the required width. Dependencies may impose additional requirements not supplied here.

The function contains no explicit null, alignment, bounds, or address-overflow checks. Fixed-offset address calculations are present, but no checked-arithmetic behavior is established.

No non-aliasing requirement is enforced. Overlap can change branch selection, later addresses, output, and final field contents. Examples using independent storage explicitly require it.

## Behavior

### Initial publication

Before any dependency call, the function:

1. Reads the header address from object offset 16.
2. Writes that value to the output slot.
3. Reads object offset 16 again.
4. Reads the word at offset 8 of that newly obtained header.
5. Saves that word in a temporary comparison-operand slot.

The second header lookup follows the output write. Consequently, the header used for the offset-8 read need not be the header value initially published if the output overlaps relevant storage.

The result and captured-value slots receive their initial values before the first comparison call.

### Initial comparisons and fast-path selection

The first call to F-eac694802fc1 receives:

- The position address.
- The address of the temporary slot containing the header’s offset-8 value.

Its 32-bit return is tested against zero. Any nonzero value permits the second initial comparison. A zero return bypasses the remaining fast-path selection and proceeds to the loop comparison.

For the second initial comparison, the function freshly reads object offset 16, saves that header address in another temporary slot, and calls F-eac694802fc1 with:

- The boundary value.
- The address of that second temporary slot.

This return is also tested against zero, not against one.

If the second return is zero, the function proceeds to the loop comparison without reading object offset 20 for fast-path selection. If it is nonzero, the function reads object offset 20. A nonzero word selects fast reset; zero selects the ordinary path.

The state word is neither decremented nor used as an iteration count. Every nonzero bit pattern satisfies its test.

Both initial comparison operands are passed by address. The dependencies may modify those temporary operands. The function does not subsequently use their contents to decide the branch; branch selection uses the comparison returns and, where reached, the object state read.

### Fast cleanup and reset

When fast reset is selected, the function performs this sequence:

1. Reads the current header address from object offset 16 and reads that header’s offset-4 word.
2. Calls F-67251854b4c9 with the object address and that word.
3. Stores a freshly obtained header address at offset 8 of a freshly addressed header.
4. Stores zero at offset 4 of the then-addressed header.
5. Stores a freshly obtained header address at offset 12 of a freshly addressed header.
6. Stores zero at object offset 20.
7. Reads object offset 16 and captures that header value.
8. Writes the captured value to the output slot.
9. Returns the captured header value in the upper half and the output address in the lower half.

F-67251854b4c9’s return value is not inspected. Its completion does not imply preservation of object fields, global state, or ownership.

All reset operations follow that call. A header change made by the dependency therefore affects subsequent header lookups. The two self-reference stores each involve header-reference reads for the destination and stored value; the supplied representation does not establish a finer ordering between those reads within an individual store.

With a stable header address and mutually nonoverlapping relevant fields and output storage, the direct final state is:

- Header offset 4 is zero.
- Header offsets 8 and 12 contain the header address.
- Object offset 20 is zero.
- The output contains the header address.

**A stable header address alone is insufficient for that final-state guarantee.** Later writes take precedence over earlier writes to overlapping storage. In particular, the final output publication can overwrite a reset field. Overlap between object and header fields can likewise change which reset value remains.

No advancement or processing call occurs on the fast path.

### Ordinary iteration

The ordinary path begins with F-eac694802fc1 receiving the position address and boundary value.

A return of **exactly one** exits the loop. Every other 32-bit return value enters the body, including zero, two, and bit patterns interpreted as negative signed values. This predicate differs from both initial comparisons.

Each entered iteration performs these operations in order:

1. Reads the first 32-bit word at the position address and preserves its value.
2. Calls F-357c4c54c6ea with the position address.
3. Assigns the preserved pre-advancement value to the captured-value slot.
4. Calls F-096f3a3bf6a9 with the result-slot address, object address, and captured-value-slot address.
5. Reads the result slot and writes that word to the output slot.
6. Calls F-eac694802fc1 again to decide whether another iteration is required.

The processing dependency thus receives a captured-value slot initialized from the position read made before advancement. Its initial contents are not taken from a position word reread after advancement.

Neither the advancement return nor the processing return is inspected.

The result slot is initialized only once. F-8310debf75fb does not reset it at the beginning of each iteration. Its value at any later point nevertheless depends on all intervening actors able to write that slot.

### Address retention and final-slot contents

After F-096f3a3bf6a9 receives the two slot addresses, those addresses may remain available to dependencies during the invocation. No supplied contract forbids retention or subsequent use while the temporary storage remains live.

Therefore:

- A later comparison may modify either slot, including the comparison that returns one and terminates the loop.
- A later advancement call may modify previously exposed slots if it has access to retained addresses.
- The next iteration’s captured-value assignment overwrites that slot after advancement, but the subsequent processing call may modify it again.
- A processing-produced result is published immediately after that processing call, but it is not necessarily the upper half eventually returned.
- Captured-value contents do not unconditionally persist until a later iteration. They may change through retained-address access between iterations or at termination.

On loop exit, the function returns the slots’ **final contents**. No additional output write accompanies that return. The output and returned upper half can consequently differ.

### Failure and termination

There is no explicit error result, rollback, progress check, or iteration limit.

If every loop comparison returns something other than one and dependencies keep returning normally, iteration continues. Any failure to return, invalid access, or exceptional behavior inside a dependency has no specified recovery path here. Earlier writes and calls are not undone.

## Side effects

The initial output write always precedes the first dependency call.

Additional direct writes are:

- On fast reset: three header-field writes, one object-state write, and a final output write.
- On each ordinary iteration: a captured-value-slot assignment and an output write following processing, in addition to initialization and temporary comparison storage.

The ordinary path does not directly advance the position or directly reset object fields. Such effects, if present, are dependency effects.

Aliasing makes publication operationally significant. An output write can alter an object field, a header field, or the position representation if those locations overlap. Dependencies called afterward observe the resulting storage, subject to their own behavior.

No allocation or release is directly established. The cleanup-like and processing roles do not prove destruction, freeing, unlinking, ownership transfer, or retention of object data.

Temporary addresses passed to dependencies designate invocation-local storage. Retention during the invocation must be considered possible; permission to use those addresses after the storage lifetime ends is not established. No dependency is assumed to preserve global state or previously exposed writable storage.

## Dependencies

| Opaque ID | Observed role | Handling by F-8310debf75fb |
|---|---|---|
| F-eac694802fc1 | Initial comparisons and repeated termination comparison | Initial returns use nonzero tests; loop exit requires exactly one |
| F-67251854b4c9 | Fast-path operation using the object and current header offset-4 word | Return ignored; reset sequence follows |
| F-357c4c54c6ea | Operation on the position before processing | Return ignored; captured input comes from before this call |
| F-096f3a3bf6a9 | Processing with writable result and captured-value slots | Return ignored; post-call result slot is published |

These descriptions establish call placement and data flow, not equality semantics, advancement distance, cleanup scope, memory ownership, or cross-call preservation guarantees.

## Acceptance examples

These are conditional static acceptance criteria. No runtime tests have been performed.

### Fast reset with independent storage

Assume valid, mutually nonoverlapping object fields, header fields, position storage, and output storage. Assume both initial comparisons return nonzero and leave the selected state nonzero. Assume the header address remains stable.

Expected behavior:

- The header address is published before comparison.
- F-67251854b4c9 receives the object and the current header offset-4 word.
- Reset writes occur in the stated order.
- Header offsets 8 and 12 finish as self-references; header offset 4 and object offset 20 finish as zero.
- The output receives the header address again.
- No ordinary iteration occurs.
- The return packs the final published header above the output address.

### Output aliases the state field

Assume the output address equals object offset 20, the header address is nonzero and stable, and header fields otherwise occupy separate valid storage. Assume both initial comparisons return nonzero and intervening dependencies preserve the state relevant to selection.

Expected behavior:

- Initial publication writes the header address into the state field, making it nonzero.
- Fast reset is selected.
- The reset sequence writes zero to the state field.
- Final output publication overwrites that zero with the header address.
- The final state field is the header address, despite header stability.
- The returned upper half is the published header value.

### Zero ordinary iterations

Assume the first initial comparison returns zero and the first loop comparison returns one.

Expected behavior:

- Only the initial output publication occurs directly.
- No advancement or processing call occurs.
- The return packs the object address above the output address.

### Nonzero does not uniformly mean termination

Assume the first initial comparison returns zero and the first loop comparison returns two.

Expected behavior: the loop body begins. Two satisfies an initial nonzero predicate but does not satisfy the loop’s exact-one exit predicate.

### Pre-advancement capture with preserved return slots

Assume the position initially contains A and advancement changes it to B. Processing receives a captured-value slot initially containing A. Suppose processing writes R to the result slot and C to the captured-value slot.

Expected behavior:

- The immediate output publication writes R.
- If the next comparison returns one **and preserves both slots**, the return packs R above C.
- Without that preservation condition, those return halves are not guaranteed.

### Terminating comparison changes retained slots

Assume processing writes R and C and makes the slot addresses available for later use. The following comparison changes the slots to S and T, then returns one.

Expected behavior:

- The preceding output publication remains R, unless another actor modifies the output.
- There is no final publication of S.
- The return packs S above T.

### Matching comparisons with zero state

Assume both initial comparisons return nonzero, but the subsequently read state word is zero.

Expected behavior: no fast cleanup occurs; execution proceeds to the ordinary loop comparison.

### Cleanup replaces the header

Assume fast reset is selected and F-67251854b4c9 replaces the object’s header reference.

Expected behavior: subsequent reset accesses use fresh post-call header lookups. They need not target the header from which the cleanup argument was obtained.

## Uncertainties

- **documented_only — Dependency semantics:** Exact comparisons, advancement effects, processing rules, cleanup effects, error signaling, and ownership behavior are unavailable.
- **documented_only — Retained-address effects:** The source exposes temporary addresses but does not establish whether dependencies retain them or which later calls use them.
- **documented_only — Object interpretation:** Sentinel-container and size-like interpretations are plausible, not required.
- **documented_only — Return interpretation:** The bit packing is established; a higher-level aggregate meaning is not.
- **documented_only — Aliasing and concurrency:** Permitted overlap, synchronization, and external mutation constraints are unspecified.
- **documented_only — Machine details:** Byte order, alignment requirements, fault behavior, and external calling conventions are not established.

The visible arithmetic consists of fixed byte offsets and 32-bit value handling. No element-count multiplication, state decrement, or checked address arithmetic is present. Architectural overflow behavior is not specified by this contract.

## Evidence

**Established static evidence:** The supplied function body establishes field offsets, word widths, initial publication, short-circuited selection, comparison predicates, fresh header lookups, reset order, capture before advancement, writable temporary arguments, publication after processing, and 64-bit return composition.

**Consequences of established accesses:** Later overlapping writes can supersede resets. Address-exposed return slots can be changed by intervening dependencies if those dependencies retain access. Final return values must therefore be distinguished from values produced by the last processing call.

**Inferred intent:** The fast path resembles whole-object emptying; the ordinary path resembles range processing with advancement before removal or processing. Neither interpretation establishes an undocumented dependency contract.

**Verification limit:** Static analysis only. No runtime tests, dependency implementations, external type definitions, or caller contracts were supplied. Acceptance examples remain conditional on their stated storage and dependency assumptions.
