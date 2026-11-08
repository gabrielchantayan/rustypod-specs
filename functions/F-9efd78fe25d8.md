<!-- Trusted provenance: F-9efd78fe25d8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9efd78fe25d8

## Purpose

F-9efd78fe25d8 synchronously processes an object and a signed adjustment. Its established behavior is to:

- Request a change based on a 32-bit object field, then save and later restore that field’s post-request value.
- Initialize temporary state, including a separate local object pointer.
- Prepare embedded storage and process a signed, inclusive index range.
- Invoke an embedded component’s virtual operation and set an object byte.
- Conditionally clear another byte, select an object, and apply an adjustment derived from shared storage.
- Issue five ordered virtual calls on the original object.

“Pending,” “refresh,” and “cached value” below are descriptive labels, not established application semantics. Rebuilding object-associated content and applying deferred state are plausible intent, but remain **inferred**.

No explicit result is returned. Observable behavior consists of memory accesses, writes, dependency calls, virtual dispatches, and whatever effects those dependencies produce.

## Interface

### Inputs and object layout

The inputs are an original object address and a signed 32-bit adjustment. The supplied representation uses 32-bit integer words and 32-bit address-bearing values.

| Object byte offset | Established access or use |
|---|---|
| `0x000` | Virtual-table address used for the five final calls. |
| `0x0BC` | 32-bit value read for each indexed lookup. |
| `0x0C0` | Embedded component; its first word supplies its virtual-table address. |
| `0x0EC` | One-byte field explicitly set to `1`. |
| `0x10C` | Embedded storage passed to F-b1304144a2df. |
| `0x288` | Signed 32-bit field used for the initial addition, then saved and restored. |
| `0x294` | Represented 32-bit read narrowed to a character for a zero/nonzero decision. |
| `0x295` | Separately accessed one-byte pending flag. |

The represented word access at `0x294` spans bytes `0x294` through `0x297`. Its narrowed value determines whether to subtract 18; the decision is not a full-word zero test or a test of byte `0x295`. The low eight bits determine zero versus nonzero. Actual instruction-level access width is unavailable.

These offsets establish accesses visible in this routine, not complete object sizes or dependency storage requirements.

### Temporary state

The routine has:

- A 16-byte scratch area.
- A three-word, 12-byte region.
- A separate local pointer holding the original object address.
- Two signed 32-bit locations supplied as range outputs.

After the initial request returns, the first word of the three-word region receives the object field’s post-call value, and its second word receives zero. Its third word has no explicit initialization.

The separate local pointer is initialized to the original object address **after those two word initializations and before F-c6cc9d37012f**. It is not a fourth word of the three-word region. No subsequent read, address-passing operation, or other use of this pointer is explicit in the supplied representation. Its purpose and any dependency-visible significance remain **documented_only**; stack proximity does not establish a larger aggregate or hidden dependency argument.

Scratch and range storage are passed to dependencies before this routine relies on their contents.

### Preconditions, outputs, and ownership

Normal completion requires valid storage for every reached field, virtual table, virtual entry, selected object, and shared record. There is no entry null check for the original object and no general validation of these addresses.

Two optional-path results are checked for zero before further object use. These checks do not validate other pointers or the objects’ layouts.

There is no explicit return status, allocation, release, ownership transfer, or destructor call. Dependencies’ ownership, retention, and memory-management behavior is unknown. Passing temporary addresses does not establish that dependencies refrain from retaining them.

## Behavior

### 1. Request an adjustment and initialize local state

The routine reads the signed 32-bit field at original-object offset `0x288`, adds the input adjustment, and calls F-45241221fad8 with the original object and the resulting value.

It does not directly store that sum into the field. The dependency’s behavior determines the field value on return.

After the call returns, the routine performs these explicit local initializations in order:

1. Reads the field at `0x288` into the first word of the three-word region.
2. Sets the region’s second word to zero.
3. Assigns the original object address to the separate local pointer.

Thus, the saved field value is not necessarily the entry value or the requested sum. The third region word remains without explicit initialization.

The initial addition has no explicit widening or overflow check. Representable signed results are established; behavior requiring signed overflow remains **documented_only**.

### 2. Update embedded storage and restore the saved field

The next calls occur in this order:

1. F-c6cc9d37012f, whose 32-bit result is retained.
2. F-4cbf0127e2c0, whose 32-bit result is retained.
3. F-b1304144a2df, receiving the embedded address at `0x10C`, the F-4cbf0127e2c0 result, and the F-c6cc9d37012f result.
4. F-21694508aea9, with no explicit arguments.

The routine then explicitly writes the saved first word of the temporary region to object offset `0x288`.

Intervening changes to that object field are overwritten at this point. This is not a general rollback: other state is not restored, and later calls may change the field again. No subsequent explicit use of the separate local object pointer appears.

### 3. Prepare scratch storage and the embedded component

F-86442406562f receives the scratch-area address.

F-b0abdd43119c then receives the scratch-area address and the address of the three-word region. That region has received explicit initialization only for its first two words.

F-6d03eb593399 next receives the embedded component address at `0x0C0`.

The sequence suggests temporary-state preparation and component initialization or reset. Those semantic roles are inferred; dependency implementations are unavailable.

### 4. Process a signed inclusive range

F-80474946e6fd receives the original object and two signed 32-bit output locations. The first becomes the current index; the second is the inclusive upper bound.

While the signed current index is less than or equal to the upper bound, each iteration:

- Reads the original object’s 32-bit field at `0x0BC`.
- Calls F-3593620aad0a with that value and the current index.
- Calls F-01ef5e396f42 with the lookup result, scratch-area address, and embedded component address.
- Increments the signed current index by one before the next comparison.

The lookup field is read anew each iteration. A preceding dependency can therefore affect a later lookup argument. The lookup result is forwarded without an explicit zero or validity check.

An initially greater lower bound produces no iterations. Equal bounds produce one iteration when the subsequent increment is representable and no relevant mutation intervenes. Negative indices are allowed by the visible comparison.

An iteration at the maximum signed 32-bit index reaches an unrepresentable increment. No explicit overflow handling or alternate termination rule is present. The routine provides no fallback if the range dependency fails to establish usable bounds. Mutation through retained output addresses is not ruled out.

### 5. Invoke the component and set a byte

After the range finishes, including an initially empty range, the routine invokes the embedded component’s virtual entry at table byte offset `0x68`, passing the component address.

After that call returns, it writes byte value `1` at original-object offset `0x0EC`.

An earlier operation that does not return normally prevents this write from being reached.

### 6. Conditionally select an object

The routine reads the byte at original-object offset `0x295`.

If zero, it skips the optional path. Otherwise, it clears that byte before calling F-938c950e5046. The flag is not explicitly restored if selection fails.

If F-938c950e5046 returns zero, the optional path ends without the subsequent selector or shared-record reads.

For a nonzero result, the routine invokes that object’s virtual entry at table byte offset `0xE8`, passing the object and a **32-bit value read from global storage for this call**.

This selector-argument read occurs after F-938c950e5046 returns. Its numeric value and immutability are not established. Changes made by F-938c950e5046 or earlier dependencies can therefore affect the supplied value. The relative evaluation order of this global argument read and the virtual-target expression is not established.

After the virtual selector returns, the routine reads another global address-bearing location and retains its value as the saved shared-record address. This read occurs before testing the selector result for zero.

A zero selector result ends the optional path without dereferencing that saved record address.

### 7. Optionally refresh shared storage and apply an adjustment

For a nonzero selected object, the routine uses both the saved record address and fresh reads of the global address-bearing location:

| Operation | Address source |
|---|---|
| Retain a record address after selection | Global read immediately after the selector returns. |
| Test bit 0 of the word at record offset `0x14` | Separate fresh global read. |
| Form F-b255b26284e6’s argument, if reached | Another fresh global read, plus `0x14`. |
| Write the refreshed value at `0x18` | Saved record address. |
| Form F-d08d3ca17b81’s argument | Saved record address plus `0x14`. |
| Read the eventual adjustment value at `0x18` | Saved record address. |

These addresses coincide only if the relevant global reads agree.

If the tested bit is set, the routine skips F-b255b26284e6 and performs no explicit refresh.

If the bit is clear, it calls F-b255b26284e6 with the freshly obtained address plus `0x14`. A zero result suppresses the explicit refresh. For a nonzero result, it:

- Reads signed 32-bit fields at selected-object offsets `0x8C` and `0x84`.
- Writes their difference, `0x8C` minus `0x84`, to saved-record offset `0x18`.
- Calls F-d08d3ca17b81 with saved-record offset `0x14`.

The relative evaluation order of the subtraction’s two operand reads is not established.

After this refresh-related sequence, the routine evaluates the represented word at original-object offset `0x294` and narrows it to a character. This decision precedes the chosen branch’s read from saved-record offset `0x18`.

F-bfa8d5524d41 receives the selected object and:

- The saved record’s value unchanged if the narrowed character is zero.
- That value minus 18 otherwise.

When F-d08d3ca17b81 was called, the final record read occurs afterward. Its preservation of the explicit refresh is not assumed.

Both subtractions are 32-bit signed operations without explicit overflow handling. Unrepresentable results remain **documented_only**.

### 8. Issue five ordered virtual calls

Finally, the routine invokes the original object’s virtual entry at table byte offset `0x58` five times.

Let **A** through **G** identify seven global 32-bit storage locations. These labels do not imply distinct values or immutable contents. A retained value is a prior local capture; an argument read is a separate read for that call.

| Call | First argument after the object | Second argument after the object |
|---|---|---|
| 1 | Argument read of B | Argument read of A |
| 2 | Argument read of C | Retained A |
| 3 | Retained C | Argument read of D |
| 4 | Argument read of F | Argument read of E |
| 5 | Retained F | Argument read of G |

A is retained before call 1. C is retained after call 1 returns and before call 2. F is retained after call 3 returns and before call 4.

Each retained read is separate from the corresponding argument read. Thus, calls 4 and 5 need not receive the same first argument: call 4 reads F again, while call 5 uses retained F. The analogous distinction applies to A in calls 1 and 2, and C in calls 2 and 3.

With stable storage, the pairs simplify to `(B, A)`, `(C, A)`, `(C, D)`, `(F, E)`, and `(F, G)`. Stability is a condition, not a guarantee.

Relative evaluation order among a call’s argument expressions and virtual-target expression is not established. The original object’s virtual table is consulted separately for each call; preceding calls can affect later dispatch.

## Side effects

Explicit nonlocal writes are:

- Restoring a saved 32-bit value at original-object offset `0x288`.
- Setting byte `0x0EC` to `1`.
- Clearing a nonzero byte at `0x295`.
- Conditionally writing a signed 32-bit difference at saved-record offset `0x18`.

Explicit local writes include the temporary-region initialization, the separate object-pointer initialization, and range-index increments.

Dependencies may additionally change object fields, globals, virtual tables, shared records, or temporary storage. They must not be assumed to preserve earlier writes or global values.

There is no explicit status reporting, retry, general rollback, synchronization, or cleanup path. Normal sequencing assumes dependencies return normally. Exception propagation, invalid-memory behavior, concurrency, and reentrancy remain unresolved.

## Dependencies

| Opaque function ID | Established interaction |
|---|---|
| F-45241221fad8 | Receives the original object and requested field value. |
| F-c6cc9d37012f | Produces the third argument later supplied to F-b1304144a2df. |
| F-4cbf0127e2c0 | Produces that call’s second argument. |
| F-b1304144a2df | Receives embedded storage at `0x10C` and the two results. |
| F-21694508aea9 | Runs before explicit field restoration. |
| F-86442406562f | Receives the scratch area first. |
| F-b0abdd43119c | Receives scratch storage and the partially initialized three-word region. |
| F-6d03eb593399 | Receives the embedded component before range processing. |
| F-80474946e6fd | Receives the object and two range-output addresses. |
| F-3593620aad0a | Receives the freshly read lookup field and current index. |
| F-01ef5e396f42 | Receives the lookup result, scratch address, and component address. |
| F-938c950e5046 | Produces the optional object; precedes the selector-argument global read. |
| F-b255b26284e6 | Receives freshly addressed shared storage; its result gates refresh. |
| F-d08d3ca17b81 | Receives saved-record offset `0x14` after the explicit refresh. |
| F-bfa8d5524d41 | Receives the selected object and computed adjustment. |

Opaque IDs are unavailable for virtual targets at table offsets `0x68`, `0xE8`, and `0x58`. Their implementations and contracts remain unresolved.

## Acceptance examples

These are conditional static criteria, not executed tests. Unless stated otherwise, dependencies return normally and do not introduce relevant mutations.

1. **Post-call restoration and local initialization.** If field `0x288` starts at 40 and the adjustment is 7, F-45241221fad8 receives 47. If it leaves the field at 52, the first temporary word receives 52, the second receives zero, and the separate local pointer receives the original object address before F-c6cc9d37012f. If the next four calls leave the field at 99, the explicit restoration writes 52. No value is prescribed for the third temporary word.

2. **Inclusive and empty ranges.** Bounds −1 and 1 produce lookup indices −1, 0, and 1. Bounds 5 and 4 produce no lookups, but the component virtual call and byte `0x0EC` write still follow. This does not establish termination for an upper bound at the maximum signed index.

3. **Consumed pending flag.** A nonzero pending byte is cleared before F-938c950e5046. Its zero result skips the selector and shared-record reads. A nonzero result followed by a zero selector result still causes the shared-record address to be read and retained before the optional path ends.

4. **Selector argument is read after the dependency.** If the selector-argument global initially contains 11 and F-938c950e5046 changes it to 23 before returning a nonzero object, the subsequent selector receives 23, provided it remains unchanged until the argument read. No ordering between that read and evaluation of the virtual target is required.

5. **Distinct shared addresses.** If the saved address denotes record R, the bit-test read denotes S, and the gate-argument read denotes T, a clear bit in S leads to F-b255b26284e6 receiving T plus `0x14`. A nonzero result causes the explicit write to R and F-d08d3ca17b81 to receive R plus `0x14`.

6. **Denied refresh.** A clear bit followed by a zero gate result causes no explicit refresh and no F-d08d3ca17b81 call. The adjustment path still reads the saved record after the character decision.

7. **Refresh and correction.** Selected fields 160 and 40 produce an explicit write of 120. If the later record read returns 120 and the narrowed character is nonzero, the adjustment is 102. If F-d08d3ca17b81 changes the record value to 130, the adjustment is instead 112.

8. **Word narrowing is not the pending test.** A represented word at `0x294` whose low eight bits are zero takes the unchanged-value branch even if its other bits are nonzero. The separately accessed pending byte does not replace this decision.

9. **Separate final reads.** If retained F is 9 but call 4’s argument read yields 12, call 4 receives 12 as its first argument after the object; call 5 receives 9. Equal arguments require agreeing reads, not merely retention.

## Uncertainties

The following remain **documented_only**:

- Application meanings of fields, flags, shared records, and virtual operations.
- Numeric values and mutability constraints of the selector-argument and final-call global storage.
- Purpose, later relevance, or dependency-visible significance of the separately initialized local object pointer.
- Complete dependency contracts, ownership, pointer retention, and memory management.
- Whether the third temporary word is ignored, initialized indirectly, or read.
- Signed-overflow behavior, including range termination at the maximum signed index.
- Actual instruction-level width of the narrowed word access.
- Unspecified expression-evaluation ordering, including selector argument versus virtual-target evaluation.
- Exception behavior, asynchronous effects, thread safety, and reentrancy.
- The shared-record protocol’s semantic role.
- Calling-convention details beyond the supplied representation.

## Evidence

Established static evidence consists of represented widths and offsets, explicit local and nonlocal assignments, dependency order, branch structure, signed inclusive comparison, virtual-table offsets, arithmetic expressions, and separate retained versus fresh reads.

In particular, the representation explicitly places the separate local object-pointer assignment after the first two temporary-word initializations and before F-c6cc9d37012f. It also places the selector-argument global read in the virtual call reached after F-938c950e5046 returns; it does not establish immutable selector storage or expression-evaluation order.

The supplied material reports omitted unreachable regions. Their unavailable contents support no additional behavior claims.

Higher-level intent is inferred separately from these structural observations. No dependency implementations or instruction-level execution evidence are supplied. No runtime tests have been performed.
