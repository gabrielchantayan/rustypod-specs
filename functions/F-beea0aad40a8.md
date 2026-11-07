<!-- Trusted provenance: F-beea0aad40a8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-beea0aad40a8

## Purpose

F-beea0aad40a8 conditionally performs a bulk structural reset or repeatedly processes individual positions. It writes a 32-bit word through a caller-supplied output address and returns a 64-bit value assembled from two 32-bit components.

**Established static behavior:** the function selects paths using dependency results, reads and writes specific fields, supplies temporary storage to dependencies, and constructs the return value from separately maintained words.

**Inferred intent:** the structure resembles a header-based container, and the two paths resemble whole-container clearing and individual range removal. The supplied body does not establish those semantic interpretations, element destruction, or memory release.

This contract describes the supplied function body without assuming that dependencies preserve global state, context fields, or storage accessible to them.

## Interface

The function receives four arguments:

| Argument | Representation and established use |
|---|---|
| Output address | Address of writable 32-bit storage. Receives an initial header-address word and subsequent path-dependent writes. Its address value initializes the low return component. |
| Context address | 32-bit address-like value. Used to access fields at byte offsets 16 and 20, passed directly to dependencies, and used to initialize the high return component. |
| Position-storage address | Address of a 32-bit word. Passed to dependencies and directly read before each individual-processing operation. |
| Boundary token | 32-bit value passed unchanged to F-73b7871d4f8c. Its interpretation is unknown. |

The observed context layout is partial:

| Byte offset | Width | Access |
|---|---:|---|
| 16 | 32 bits | Repeatedly read as the address of a header-like object. |
| 20 | 32 bits | Conditionally read during bulk eligibility; explicitly cleared on the bulk path. |

The observed header layout is also partial:

| Byte offset | Width | Access |
|---|---:|---|
| 4 | 32 bits | Read for a dependency argument; explicitly cleared during bulk reset. |
| 8 | 32 bits | Read into temporary storage before eligibility checks; assigned a header-address word during bulk reset. |
| 12 | 32 bits | Assigned a header-address word during bulk reset. |

No interpretation is established for other fields.

### Return representation

The return has 64 bits:

- The high 32 bits come from a path-dependent result word.
- The low 32 bits come from a saved address or a temporary subsequently exposed to F-d1df92322ed0.

Each component contributes its 32-bit bit pattern. This is concatenation, not arithmetic addition, sign extension, or evidence that the result is a single pointer.

On the bulk path, the high component is the header-address word loaded immediately before the final output write. On the individual path, it is the current context/result temporary. The low component initially contains the output address and is replaced during each individual-processing iteration.

### Preconditions and representation limits

Every reached memory access requires accessible storage of the indicated width. In particular:

- The output address must support a 32-bit write.
- The context must support its reached field reads and, on the bulk path, its offset-20 write.
- Each reloaded header address must support the particular field access reached.
- Position storage must support a 32-bit read whenever an individual-processing iteration executes.
- Dependency arguments must satisfy the dependencies’ own, otherwise unavailable contracts.

The body contains no explicit null, bounds, alignment, overflow, or alias checks. Defined behavior for invalid addresses or address arithmetic near the representation’s limits is not established.

The supplied representation uses 32-bit address-like words and integer results. It does not establish behavior in a wider-pointer environment or portable signed-overflow semantics for address calculations.

No non-aliasing guarantee is present. The output, context fields, header fields, and position storage may overlap unless callers impose stronger requirements. Such overlap can change later reads and final state.

## Behavior

### Initialization

The function performs these actions before its first dependency call:

1. Reads the context’s offset-16 word.
2. Writes that word through the output address.
3. Reloads the context’s offset-16 word.
4. Reads offset 8 of the header identified by that reload and copies the word into temporary storage.
5. Initializes separate temporary storage with the output address, the context address, and the position-storage address.

The initial output write therefore precedes the header-offset-8 read. If that write overlaps the context’s offset-16 field or another subsequently read location, later reads observe the resulting storage, not an immutable initial snapshot.

The temporary initialized with the output address becomes the low return component. The temporary initialized with the context address becomes the high return component on the individual path.

### Bulk eligibility

The first call to F-73b7871d4f8c receives:

- The original position-storage address.
- The address of the temporary containing the copied header-offset-8 word.

A result of zero bypasses the remaining eligibility checks and enters individual processing.

For a nonzero result, the function reloads the context’s offset-16 word into a separate temporary, replacing that temporary’s initial position-storage-address value. It then calls F-73b7871d4f8c with:

- The original boundary token.
- The address of this temporary header-address word.

If the second result is zero, the function enters individual processing without performing the explicit context-offset-20 eligibility read.

If the second result is nonzero, the function reads the context’s offset-20 word. A nonzero word selects bulk reset; zero selects individual processing.

Thus bulk reset requires, in order:

- A nonzero first eligibility result.
- A nonzero second eligibility result.
- A nonzero context-offset-20 word read after those calls.

Any represented nonzero integer result satisfies an eligibility branch, including a negative result. Neither comparison is required to return exactly 1.

The dependencies’ effects remain relevant. The header reload and offset-20 read occur after preceding calls, so they need not observe the values present at entry.

### Bulk reset

Once bulk reset is selected, the function:

1. Reloads the context’s offset-16 word and reads offset 4 of the identified header.
2. Calls F-7594ea2817ed with the original context address and that offset-4 word.
3. Assigns a header-address word to offset 8 of a header object, obtaining both the destination header and assigned address through context-offset-16 reads.
4. Reloads the header address and writes zero to its offset 4.
5. Assigns a header-address word to offset 12 of a header object, again obtaining the destination and assigned address through context-offset-16 reads.
6. Writes zero to context offset 20.
7. Reloads context offset 16 into a result word.
8. Writes that result word through the original output address.
9. Returns with that result word in the high half and the saved output-address word in the low half.

The completed assignments occur in the stated order: header offset 8, header offset 4, header offset 12, context offset 20, then final output.

Within either self-reference assignment, the supplied expression does not establish a separate portable order between the reads that obtain the destination header and the assigned header-address word.

The function does not retain one immutable header address across the dependency call and reset assignments. If F-7594ea2817ed replaces context offset 16, subsequent accesses use the replacement. Aliasing can also affect subsequent reloads.

Under stable, non-aliasing storage, with no dependency effects that invalidate the relevant accesses:

- Header offsets 8 and 12 end with the header’s own address.
- Header offset 4 ends with zero.
- Context offset 20 ends with zero.
- The output contains the final header-address word.

These are conditional final-state assertions. Unconditionally, the body establishes the ordered writes, not that every written value survives until return. In particular, if the output overlaps context offset 20, the final output write follows and can overwrite the zero stored there.

The high return component is captured before the final output write. If that write changes a field from which the word was loaded, the return still uses the captured word.

No individual-processing comparison follows a completed bulk reset.

### Individual processing

All paths that do not select bulk reset enter a loop. Each loop-condition call to F-73b7871d4f8c receives:

- The original position-storage address.
- The original boundary token.

Only a result **exactly equal to 1** terminates the loop. Zero, 2, negative results, and every other represented value continue into an iteration. This differs from the eligibility checks’ nonzero conditions.

Each continuing iteration performs the following ordered actions:

1. Reads and saves the 32-bit word at the original position-storage address.
2. Calls F-ec352f140c12 with that address.
3. Replaces the low-component temporary with the saved pre-call position word.
4. Calls F-d1df92322ed0 with:
   - The address of the context/result temporary.
   - The original context address.
   - The address of the low-component temporary.
5. Reads the context/result temporary and writes its current word through the original output address.
6. Returns to the loop-condition call.

The third argument to F-d1df92322ed0 initially contains the position word read **before** F-ec352f140c12. Changes that F-ec352f140c12 makes to position storage do not substitute a later value for this captured word.

The context/result temporary is initialized once with the context address; it is not explicitly reinitialized at the start of each iteration. F-d1df92322ed0 can modify it through its first argument. Its value therefore carries across iterations unless changed through an accessible route.

The low-component temporary is explicitly replaced with each iteration’s captured position word before F-d1df92322ed0 receives its address. That dependency can then modify either component temporary. The function does not require it to do so.

When a loop comparison returns exactly 1, the function returns the current component temporaries without another explicit output write.

If no iteration executes, and dependencies have no relevant effects through other accessible storage:

- The output retains the initially written header-address word.
- The return high half remains the context address.
- The return low half remains the output address.

Consequently, the output word need not equal the return’s high half.

## Side effects

The function’s explicit caller-visible writes are:

- One initial 32-bit output write on every normally reached path.
- On bulk reset: three header-field writes, one context-field write, and one final output write.
- On individual processing: one output write after each normally completed F-d1df92322ed0 call.

These are write events, not unconditional persistent postconditions. Aliasing can make a later write replace an earlier one, and dependencies can introduce additional mutations.

The function also writes temporary storage, some of which is exposed to dependencies by address. The body establishes use of those addresses during calls, not permission for a dependency to retain or use them after the function returns.

Ownership is unresolved:

- No direct allocation or deallocation operation appears in the body.
- F-7594ea2817ed is not established to free anything.
- F-d1df92322ed0 is not established to destroy or release an element.
- F-ec352f140c12 is not established to advance a position, despite being consistent with that interpretation.
- No transfer of ownership is established merely by passing an address or word.

There is no explicit rollback or error-recovery path. If a dependency does not return normally, prior writes may already have occurred. The body does not interpret any result as a conventional success or error code: comparison results select control flow, while the other dependency results are unused.

Termination is not guaranteed. If loop comparisons never return exactly 1 and dependencies continue returning normally, individual processing continues.

## Dependencies

| Opaque function ID | Observed use | Unresolved behavior |
|---|---|---|
| F-73b7871d4f8c | Two-argument integer-result operation used for eligibility and loop termination. Some invocations receive addresses of temporary words. | Argument semantics, result meaning, purity, mutations, and error signaling. |
| F-7594ea2817ed | Receives the context address and the current header-offset-4 word before explicit reset writes. | Traversal, destruction, allocation, ownership, and context mutation. |
| F-ec352f140c12 | Receives position storage after its current word has been captured. | Whether it advances or otherwise changes that storage, and any additional effects. |
| F-d1df92322ed0 | Receives addresses of both return-component temporaries, with the original context address between them. | Resulting temporary values, structural changes, allocation, and ownership effects. |

A replacement claiming equivalent behavior must preserve the relevant call order, argument values, read timing, and temporary-storage exposure. Assuming stable context fields or pure comparisons requires evidence outside this body.

## Acceptance examples

These are conditional static acceptance examples, not runtime test results. Except where aliasing is explicitly described, assume valid, stable, non-aliasing storage and no dependency effects beyond those stated.

### Immediate individual-path termination

Let the context address be 0x00001000, output address 0x00003000, and header address 0x00002000.

If the first eligibility result is zero and the first loop comparison returns 1:

- The output contains 0x00002000.
- Neither individual-processing dependency is called.
- The return is 0x0000100000003000.
- No explicit reset write occurs.

### Successful bulk reset with non-aliasing storage

Let the header address remain 0x00002000 and the output address be 0x00003000. Both eligibility results are nonzero, the subsequently read context-offset-20 word is nonzero, and F-7594ea2817ed returns without changing relevant storage.

Under the stated stable, non-aliasing assumptions:

- F-7594ea2817ed precedes every explicit reset write.
- Header offsets 8 and 12 finish as 0x00002000.
- Header offset 4 and context offset 20 finish as zero.
- The output finishes as 0x00002000.
- The return is 0x0000200000003000.
- No loop-condition call follows.

### Output aliases context offset 20

Let the context address be 0x00002fec, the output address be 0x00003000, and context offset 16 identify stable header 0x00002000. The output therefore aliases context offset 20. Assume both eligibility results are nonzero and dependencies make no relevant changes.

The initial output write stores 0x00002000 into context offset 20, making its later eligibility read nonzero. During bulk reset:

- The explicit context-offset-20 reset writes zero.
- The later final output write overwrites that same word with 0x00002000.
- Context offset 20 therefore finishes as 0x00002000, not zero.
- The return is 0x0000200000003000.

A contract requiring context offset 20 to remain zero for this aliasing case would be incompatible with the write order.

### Nonzero does not imply loop termination

If a loop comparison returns 2, an iteration executes. If the next comparison returns 1, the function then returns. Treating the initial 2 as a terminating result changes behavior.

### Position capture precedes dependency changes

Suppose position storage contains word A and F-ec352f140c12 changes it to word B.

The third-argument temporary supplied to F-d1df92322ed0 initially contains A, not B. F-d1df92322ed0 may subsequently replace that temporary.

### Dependency-provided return components

Suppose F-d1df92322ed0 writes X into its first-argument temporary and Y into its third-argument temporary. If the next comparison returns 1 without relevant intervening effects:

- The iteration’s output write stores X.
- The return has X in its high half and Y in its low half.

### Header replacement during bulk processing

Suppose F-7594ea2817ed changes context offset 16 from header H to header J. With otherwise stable, non-aliasing storage, subsequent explicit reset writes target J; the final output and return high half contain J. Retaining H throughout the reset would change behavior.

## Uncertainties

The following remain **documented_only**:

- Public types and semantic names for the context, header, position, and boundary.
- Whether context offset 20 is a count, flag, or another quantity.
- Whether header offsets 8 and 12 are links or endpoints.
- Why eligibility accepts any nonzero result while loop termination requires exactly 1.
- Dependency effects on globals, caller storage, temporaries, allocation, and ownership.
- Whether the 64-bit return is an application-level pair or an implementation-specific convention.
- Caller-imposed non-aliasing, synchronization, alignment, or lifetime requirements.
- Behavior for invalid accesses, concurrent mutation, non-normal dependency exits, or retention of temporary addresses.
- Intra-expression ordering not established by the supplied representation.

These unknowns do not justify assuming pure comparisons, stable headers, automatic memory release, or eventual position progression.

## Evidence

The evidence is the supplied static function body and opaque dependency mapping.

Established observations include the 32-bit accesses and field offsets, initial output write, header reloads, short-circuited eligibility checks, exact-1 loop termination, pre-call position capture, exposure of return-component temporaries, ordered bulk writes, and 64-bit return concatenation.

The aliasing example follows from the explicit context-offset-20 zero write occurring before the final output write. The container-clearing and range-removal interpretations remain inferred intent, not verified dependency semantics.

No runtime tests have been performed.
