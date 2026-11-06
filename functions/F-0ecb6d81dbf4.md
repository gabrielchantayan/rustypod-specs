<!-- Trusted provenance: F-0ecb6d81dbf4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ecb6d81dbf4

## Purpose

F-0ecb6d81dbf4 advances a state record, conditionally consumes a word through a stored cursor, invokes dependent operations, and may replace the record’s active four-word state.

The established behavior is structural: a four-byte advance, a one-unit decrement, a conditional callback sequence, and one of two state-replacement paths. The supplied evidence does not establish what the state represents, what the consumed words mean, or whether the callbacks perform allocation, release, synchronization, or error handling.

The function returns a 64-bit value assembled from two words of a temporary four-word block. That value is not established as a success code or pointer. In particular, returning without a state-replacement operation normally packages the first argument’s address together with the second argument’s original bits.

## Interface

### Inputs

The function receives four arguments:

| Argument | Established representation | Role visible in this function |
|---|---|---|
| State address | Address represented in a 32-bit word, pointing to a record of 32-bit words | Readable and writable state storage |
| Second argument | 32-bit word | Initial value of temporary word 1 |
| Third argument | 32-bit word | Initial value of temporary word 2 |
| Fourth argument | 32-bit word | Initial value of temporary word 3 |

The first argument must support the reads and writes required by the executed path. Across all paths, the function references eleven consecutive 32-bit state words, covering byte offsets 0 through 43.

The supplied representation uses 32-bit integer and address-bearing fields. It does not establish a portable interface for an environment with wider addresses.

### State layout

The following labels describe access behavior, not proven domain meaning:

| Byte offset | Width | Observed use |
|---|---:|---|
| 0 | 32 bits | Active word 0; initially increased by four; later compared with active word 2 |
| 4 | 32 bits | Active word 1 |
| 8 | 32 bits | Active word 2; participates in an equality test |
| 12 | 32 bits | Active word 3; interpreted as a word cursor on the conditional path |
| 16 | 32 bits | Staging word 0 |
| 20 | 32 bits | Staging word 1 |
| 24 | 32 bits | Staging word 2 |
| 28 | 32 bits | Staging word 3 |
| 32 | 32 bits | Counter-like word; initially decreased by one |
| 36 | 32 bits | First argument to the final conditional invocation of F-ff1bdc376dcd |
| 40 | 32 bits | Second argument to that invocation |

The term “staging” reflects the explicit sequence in which words at offsets 16–28 are written and then read back into offsets 0–12. It does not establish backup, recovery, or ownership semantics.

### Temporary block and return value

The function creates a temporary block of four 32-bit words. Before the first dependency call, it initializes that block as follows:

1. Word 0 receives the 32-bit representation of the state address.
2. Word 1 receives the second argument.
3. Word 2 receives the third argument.
4. Word 3 receives the fourth argument.

F-f1741872dc8d receives the address of this block on either state-replacement path. Its effects on the block are unknown.

On a normal return, the result is a 64-bit concatenation:

- Bits 0–31 come from temporary word 0.
- Bits 32–63 come from temporary word 1.

This is a bit-level construction, not a signed numeric addition. No signed interpretation of the combined result is established.

## Behavior

### Initial state changes

The function first reads the 32-bit word at state offset 0 and writes back that value increased by four.

It then reads the 32-bit word at offset 32 and writes back that value decreased by one.

These are separate, ordered updates: the offset-0 write precedes the offset-32 update. No lower-bound, upper-bound, or zero check is visible.

The established operand and storage widths are 32 bits. Behavior at arithmetic overflow boundaries is not independently established by the supplied representation. The contract therefore does not impose a portable signed-overflow rule or assume that underflow is rejected.

After these state changes, the temporary block is initialized from the four incoming arguments.

### Initial dependency calls and branch decision

The function calls F-8a5aadac202f with no explicit arguments, then calls F-83d59a117d49 with the state address.

The return from F-83d59a117d49 is tested for zero:

- A nonzero return enters the conditional processing path immediately.
- A zero return causes a comparison of the current 32-bit words at state offsets 0 and 8.
- If those words are equal, conditional processing is entered.
- If they differ, the function returns the concatenation of temporary words 1 and 0 without further explicit state writes.

The equality comparison is short-circuited when the dependency return is nonzero. It is evaluated after the dependency calls, so its operands are not necessarily the values present immediately after the initial increment.

For the zero-and-unequal outcome, the function itself leaves active words 1–3, the staging words, and the final callback arguments untouched. This does not imply that dependencies left them unchanged.

### Conditional cursor consumption

Once conditional processing is selected, the function:

1. Reads the address-bearing word at state offset 12.
2. Writes an address advanced by one 32-bit word, or four bytes, back to offset 12.
3. Reads a 32-bit payload from the original address.

The cursor update occurs before the payload read. There is no explicit null check, bounds check, alignment check, or rollback.

The function then calls F-a2ee2ea15a51 with no explicit arguments. It passes the previously read payload, the 32-bit result from that call, and a literal zero to F-ff1bdc376dcd.

After F-ff1bdc376dcd returns normally, the function calls F-83d59a117d49 again with the state address. This second result selects the replacement path independently of the first result.

### Replacement path when the second predicate returns zero

When the second F-83d59a117d49 result is zero, the function reads the current word at state offset 12 as an address and reads a 32-bit word from that address.

This is a fresh cursor read after the intervening dependency calls. It is not necessarily the original cursor plus four: dependencies may have changed the state.

The function calls F-f1741872dc8d with:

- The address of the temporary four-word block.
- The newly dereferenced 32-bit word.

The supplied call site shows two explicit arguments for this invocation.

After the dependency returns, the function writes temporary words 0, 1, 2, and 3 to state offsets 0, 4, 8, and 12, respectively, in that order.

It then returns the 64-bit concatenation of temporary words 1 and 0. There is no explicit final invocation of F-ff1bdc376dcd on this path.

### Replacement path when the second predicate returns nonzero

When the second F-83d59a117d49 result is nonzero, the function calls F-f1741872dc8d with:

- The address of the temporary four-word block.
- A literal zero.
- A second literal zero.

The supplied call site shows three explicit arguments for this invocation. The difference from the two-argument invocation must remain visible in the contract; the evidence does not establish a single resolved dependency signature.

After this call returns, the function performs two ordered groups of writes:

1. Temporary words 0–3 are written to state offsets 16, 20, 24, and 28.
2. Those staging fields are read back and written to state offsets 0, 4, 8, and 12.

Within each group, the fields are processed in increasing offset order. The active-state update is not established as atomic.

The function then invokes F-ff1bdc376dcd with the current 32-bit values from state offsets 36 and 40, followed by a literal zero. The relative evaluation order of the two argument reads is not established by the supplied representation.

Finally, it returns the concatenation of temporary words 1 and 0 as they stand after the dependency returns.

## Side effects

### Direct memory effects

The function directly performs:

- An increase of four at state offset 0.
- A decrease of one at state offset 32.
- On the conditional path, an advance of four bytes at state offset 12.
- On the zero-result replacement path, four active-state writes.
- On the nonzero-result replacement path, four staging writes followed by four active-state writes.

Conditional cursor processing also directly reads a payload through an address obtained from offset 12. The zero-result replacement path performs another indirect word read through the then-current offset-12 address.

### Dependency effects and ownership

No allocation, deallocation, reference-count update, or ownership transfer is established directly by this function.

Passing a word to F-ff1bdc376dcd does not establish that the word identifies owned memory or that the dependency releases it. Likewise, passing the temporary block to F-f1741872dc8d establishes access to that block during the call, not a right to retain it after this function returns.

The dependencies’ access to global state, shared storage, and aliases is unknown. Their effects must not be treated as absent merely because those effects are not shown here.

If any dependency retains the temporary block’s address, that address refers to storage whose lifetime does not extend beyond this invocation. Whether retention occurs is unknown.

### Failure behavior

There is no explicit validation failure result, retry, rollback, or recovery path.

Invalid state storage, invalid cursor addresses, or dependency failures have no specified handling in the supplied evidence. A failure after the initial writes may leave the state partially modified. A failure during payload access occurs after the cursor has already been advanced.

Normal completion of each dependency is a precondition for describing later operations on that path.

## Dependencies

| Opaque ID | Observed interaction | Unestablished behavior |
|---|---|---|
| F-8a5aadac202f | Called once, after initial state changes and temporary initialization, with no explicit arguments | Purpose, hidden inputs, global effects, and failure behavior |
| F-83d59a117d49 | Called with the state address; its zero/nonzero result controls branching; called a second time only after cursor processing | Meaning of the result, state mutation, and whether repeated calls agree |
| F-a2ee2ea15a51 | Called once on the conditional path with no explicit arguments; supplies the second word passed to the first F-ff1bdc376dcd call | Meaning of the returned word and other side effects |
| F-ff1bdc376dcd | Called after payload consumption with three words; called again only on the nonzero-result replacement path | Ownership, release behavior, global effects, and failure behavior |
| F-f1741872dc8d | Receives the temporary block address; invoked with two explicit arguments on one path and three on the other | Complete signature, interpretation of zero arguments, and temporary-block transformation |

The function does not explicitly branch on the results of F-8a5aadac202f, F-ff1bdc376dcd, or F-f1741872dc8d.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume valid required memory and normal dependency returns. Any stated absence of dependency mutation is an explicit condition, not an inferred guarantee.

### Initial predicate is zero and active bounds differ

Suppose offset 0 initially contains 100, offset 8 contains 200, and offset 32 contains 7. Suppose the first two dependencies do not alter these fields and F-83d59a117d49 returns zero.

The function writes 104 to offset 0 and 6 to offset 32. Because 104 differs from 200, it does not consume a cursor word or invoke the replacement dependency.

The low return word is the state address’s 32-bit representation. The high return word is the original second argument.

### Equality enters processing despite a zero predicate

Suppose offset 0 initially contains 100 and offset 8 contains 104. If the dependencies preserve those values and the first predicate returns zero, the post-increment comparison is equal.

The function enters conditional processing. Comparing the original value 100 against 104 instead would not satisfy the established behavior.

### A nonzero initial predicate bypasses equality evaluation

If the first predicate returns any nonzero value, conditional processing is selected without evaluating the explicit equality comparison. The second predicate call still occurs after payload processing and may independently return zero.

### Cursor processing uses the original address for its first payload

Suppose offset 12 contains a valid address, and that address contains payload word 55.

The function first stores the address advanced by four bytes into offset 12, then reads 55 from the original address. It passes 55, the result of F-a2ee2ea15a51, and zero to F-ff1bdc376dcd.

### Zero second predicate uses a freshly read cursor

Suppose an intervening dependency changes offset 12 to another valid address, and the second predicate returns zero.

The argument read for F-f1741872dc8d comes from that new address. It must not be taken from a cached original cursor.

If F-f1741872dc8d leaves the temporary block containing four specified words, those words become active state in increasing offset order. The first two also determine the return.

### Nonzero second predicate stages the replacement

Suppose the second predicate returns a nonzero value and F-f1741872dc8d produces four specified temporary words.

The function writes all four staging fields before copying staging fields into active state. It then invokes F-ff1bdc376dcd using state offsets 36 and 40 and a zero third argument.

If that final dependency leaves the temporary block unchanged, the returned low and high words equal the first two replacement words.

## Uncertainties

The following remain **documented_only**:

- The domain meaning of the active fields, counter-like field, payloads, and dependency results.
- Arithmetic behavior at 32-bit overflow and underflow boundaries.
- Exact dependency signatures beyond the arguments visible at each call site.
- The reason F-f1741872dc8d has different visible argument counts.
- Whether dependencies mutate state, globals, indirect storage, or previously exposed temporary storage.
- Memory ownership and any release semantics.
- Required cursor alignment and bounds beyond the need for valid executed accesses.
- Error, exception, non-returning, reentrant, and concurrent behavior.
- Whether any multiword state transition has synchronization guarantees outside this function.
- Whether the 64-bit return represents a domain object, paired values, or another encoding.

## Evidence

The supplied static function body establishes field offsets and widths, direct access ordering, short-circuit branching, visible dependency calls, temporary-block initialization, state-copy sequences, and the two-word return construction.

Descriptive labels such as “active,” “staging,” “cursor,” and “counter-like” are inferred intent or access-role terminology. They do not establish a higher-level protocol.

Dependency implementations, runtime observations, and validated interface declarations were not supplied. No runtime tests have been performed.
