<!-- Trusted provenance: F-2ff2b3f2e1e7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2ff2b3f2e1e7

## Purpose

F-2ff2b3f2e1e7 repeatedly selects and processes nodes associated with a caller-supplied state object until a one-byte stop field becomes nonzero. Its processing decision depends on the low 32 bits of a node-selected indirect call’s result. Between iterations, it invokes synchronization-like dependencies and, when no continuation node is available, an additional dependency.

On exit, it clears the stop field and conditionally invokes a final indirect dependency.

The control flow and memory accesses below are established by the supplied static representation. Descriptions such as “node,” “continuation,” and “stop field” are behavioral labels. Interpreting the function as a worker loop protected by a lock and waiting for work is plausible, but the dependency implementations are unavailable and do not establish those semantics.

## Interface

### Entry and return

The function accepts one value used as the base address of a mutable state object. The supplied representation models this argument as a 32-bit integer used in address calculations.

There is no explicit return value. Normal completion occurs after the stop field has been observed as nonzero, cleared, and the conditional finalization path has completed. Completion is not guaranteed: the loop, any dependency, or either indirect call may fail to return.

No null check, bounds check, allocation check, or explicit error result is present.

### State-object layout

Offsets are byte offsets from the supplied base address.

| Offset | Width or interpretation | Observed use |
|---|---|---|
| `+0x04` | Address of an embedded region; extent unknown | First argument to F-f024b4153e2c |
| `+0x08` | 32-bit pointer-valued field | Initial node source and post-callback nonzero check |
| `+0x10` | 32-bit field | Cleared when restarting selection; may receive the processed node pointer |
| `+0x14` | Signed 32-bit integer | Processing occurs only when this value is greater than zero |
| `+0x18` | Address of a region; extent unknown | Argument to F-fceaba74ea88 |
| `+0x1c` | One byte | Loop termination test; cleared on exit |
| `+0x20` | Address of a region; first word later interpreted as a pointer | Argument to F-338d16f9ce72 and F-1333331b2da1; finalization gate |

The regions passed to dependencies need not be independent objects. Their full layouts, sizes, and possible overlap are unknown. In particular, a dependency receiving `+0x18` might affect adjacent state; its implementation is not available.

### Node layout

A selected node is used as follows:

| Offset | Width or interpretation | Observed use |
|---|---|---|
| `+0x00` | 32-bit pointer to a dispatch table | Locates the node-selected indirect target |
| `+0x04` | 32-bit pointer-valued word | Supplies the next continuation after one callback outcome |

The dispatch table contains a callable target at byte offset `+0x08`. That target is invoked with the selected node as its sole explicit argument.

The indirect result is represented as 64 bits. Only its low 32 bits affect reachable selection decisions for a valid, nonnull node. The high 32 bits do not establish a status code, pointer, or secondary output.

### Preconditions

For defined execution, every memory access and indirect call actually reached must be valid. This includes the state fields, any selected node, its dispatch table, and the finalization-gate pointee when nonnull.

When processing starts with no retained continuation, the node loaded from `+0x08` must support the accesses performed after the first F-f024b4153e2c call. There is no protective null check before dispatch through that node.

The caller must also satisfy any synchronization, lifetime, or calling-convention requirements of the dependencies. Those requirements are not recoverable here.

## Behavior

### Entry and loop admission

1. The function initializes its private continuation pointer to null.
2. It calls F-338d16f9ce72 with the address at state offset `+0x20`.
3. After that call returns, it reads the byte at `+0x1c`.
4. A zero byte admits one loop iteration. Any nonzero byte exits the loop.

Thus, an already-set stop field does not bypass the initial dependency call. The stop field is checked only at loop admission, not after each processing step.

### Processing eligibility and selection

At the start of an admitted iteration, the function reads the signed 32-bit field at `+0x14`.

If the value is zero or negative, it performs no node processing in that iteration. The private continuation remains unchanged.

If the value is positive:

- A nonnull private continuation is selected directly.
- If the private continuation is null, the function first writes a 32-bit zero to `+0x10`, then reads the node pointer from `+0x08` and selects that value.

The ordering of that write and read is observable: clearing `+0x10` precedes loading the initial node.

The function then calls F-f024b4153e2c with:

- the address at state offset `+0x04`; and
- the selected node.

The return value of this first call is ignored.

After the dependency returns, the function loads the selected node’s dispatch-table pointer, obtains the callable target at table offset `+0x08`, and invokes it with the selected node.

The original selected-node value remains the basis of dispatch. The function does not replace it with the first dependency’s return value or reload it from `+0x08`.

### Callback result: low 32 bits equal zero

For a valid nonnull selected node, the function reads the current pointer-valued field at state offset `+0x08` after the indirect call returns. This is a fresh read, not a reuse of the initial selection value.

The private continuation is reset to null.

If that freshly read field is nonzero:

1. The function writes the selected node pointer to `+0x10`.
2. It reads the pointer-valued word at selected-node offset `+0x04`.
3. It retains that word as the private continuation.

If the freshly read field is zero, neither of those two accesses occurs: `+0x10` is not written by this branch, and the selected node’s second word is not read.

The state field’s nonzero value serves only as a gate here. It is not itself used as the next node.

The represented logic also tests whether the selected node is null. That test occurs after an unconditional dispatch-table dereference through the same node, so it is not a general null-safety mechanism. No valid null-node execution is established.

### Callback result: low 32 bits nonzero

The function calls F-f024b4153e2c a second time, with the same two explicit argument values used for the first call:

- state base plus `0x04`; and
- the selected node.

This second call’s result becomes the private continuation.

The function does not read the selected node’s `+0x04` word on this path. It does not directly update `+0x10` on this path, apart from any earlier clear during initial selection.

The second dependency may observe state changed by its first invocation or by the indirect callback. Equal arguments do not imply equal behavior or results.

### End-of-iteration dependency ordering

After processing—or after skipping processing because the signed count was not positive—the function branches on the private continuation:

- **Null continuation:** call F-1333331b2da1 with state base plus `0x20`, then call F-fceaba74ea88 with state base plus `0x18`.
- **Nonnull continuation:** call F-1333331b2da1 with state base plus `0x20`; do not call F-fceaba74ea88.

Both paths then call F-338d16f9ce72 with state base plus `0x20`.

Only after this final call returns does the next stop-field check occur.

Consequently, setting the stop field during a callback does not immediately stop the current iteration. The applicable end-of-iteration calls still occur before the function observes that change.

If the count becomes nonpositive while a nonnull continuation is retained, the function retains it, skips processing, and follows the nonnull end-of-iteration path. A nonpositive count alone does not force a call to F-fceaba74ea88.

### Exit and conditional finalization

After observing a nonzero stop byte, the function:

1. Writes zero to the single byte at `+0x1c`.
2. Reads a pointer-valued word from `+0x20`.
3. If that pointer is nonnull, reads its first 32-bit word.
4. If both the pointer and its first word are nonzero, invokes a separate indirect target with no explicit arguments.
5. Returns if the applicable indirect call returns, or immediately if the gate is false.

A null pointer at `+0x20` is not dereferenced. A nonnull pointer whose first word is zero is dereferenced but does not trigger the indirect call.

The first word is a gate, not the call target. The target comes from a separate unresolved location.

The stop-byte clear precedes all finalization-gate reads and any final indirect invocation. A later dependency could change the byte again; the function does not clear it a second time.

## Side effects

### Direct writes

The function directly performs only these state writes:

- A 32-bit zero to `+0x10` when positive-count processing begins without a retained continuation.
- A 32-bit selected-node pointer to `+0x10` on the zero-result branch when the post-callback `+0x08` field is nonzero.
- A one-byte zero to `+0x1c` after loop exit.

It directly reads node memory but does not directly write it.

### Indirect effects and ownership

All dependencies and indirect calls may have additional effects. Their contracts are unavailable. In particular, this function does not establish that they:

- preserve state fields or global memory;
- retain or release node ownership;
- leave a selected node allocated;
- acquire or release a particular synchronization primitive;
- return promptly, return at all, or report failures.

No explicit allocation or deallocation appears in this function. Retaining a pointer as the private continuation does not establish ownership transfer or a lifetime guarantee.

The zero-result path can read the selected node after the callback. That path therefore requires the node to remain readable until those accesses complete. The nonzero-result path passes the selected node to another dependency after the callback; its validity requirements depend on that dependency.

### Errors and concurrency

There is no local error-handling path, retry limit, timeout, or explicit dependency-status check. Invalid pointers or indirect targets are not converted into a documented error result.

Repeated calls involving `+0x20` suggest synchronization, but static call order does not prove mutual exclusion, memory ordering, or visibility of writes by other execution agents. Those properties remain dependency-level prerequisites.

## Dependencies

Known dependencies use the supplied opaque IDs. The two unresolved indirect call sites are labeled F-unknown-01 and F-unknown-02 here solely for reference; these labels do not identify particular implementations.

| Dependency | Established call contract |
|---|---|
| F-338d16f9ce72 | Receives state base plus `0x20`; called once before the first stop check and once at the end of every completed iteration |
| F-1333331b2da1 | Receives state base plus `0x20`; called once per completed iteration before any optional F-fceaba74ea88 call |
| F-fceaba74ea88 | Receives state base plus `0x18`; called only when the private continuation is null at the end-of-iteration branch |
| F-f024b4153e2c | Receives state base plus `0x04` and the selected node; called before every node callback, and again when the callback’s low 32 bits are nonzero |
| F-unknown-01 | Selected through the node’s dispatch table at byte offset `+0x08`; receives the node; returns a represented 64-bit result whose low 32 bits control the branch |
| F-unknown-02 | Separate unresolved indirect target; invoked without explicit arguments only when the finalization pointer and its first word are both nonzero |

No relationship between the two indirect targets is established.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests. Dependency side effects must be constrained as stated for each expected outcome.

### Stop already requested

Given a nonzero stop byte at entry, and an initial F-338d16f9ce72 call that returns without changing it:

- No iteration or node processing occurs.
- The stop byte is cleared.
- The finalization gate is evaluated.
- Neither F-1333331b2da1 nor F-fceaba74ea88 is called by the loop.

### No eligible work on the first iteration

Given a zero stop byte and a signed count of zero or a negative value:

- The initial null continuation remains null.
- No node is selected.
- No direct write to `+0x10` occurs.
- The end-of-iteration sequence is F-1333331b2da1, F-fceaba74ea88, then F-338d16f9ce72.

A dependency must eventually change the stop state, prevent return, or otherwise alter execution for this scenario not to repeat indefinitely.

### Zero callback result with a successor

Given a positive count, no retained continuation, a valid initial node, a callback whose low 32 bits are zero, and a nonzero post-callback `+0x08` field:

- `+0x10` is first cleared before initial node selection.
- F-f024b4153e2c runs before the callback.
- After the callback, `+0x10` receives the selected node.
- The node’s `+0x04` word becomes the continuation.

If that word is nonnull, F-fceaba74ea88 is omitted. If it is null, F-fceaba74ea88 is invoked after F-1333331b2da1.

### Callback clears the state’s node-source field

Given a valid selected node and a zero low-32-bit callback result, if the callback leaves `+0x08` equal to zero:

- The continuation becomes null.
- The function does not read the node’s `+0x04` word.
- This branch does not write the selected node into `+0x10`.
- The null-continuation end-of-iteration sequence applies.

### Nonzero callback result

Given any nonzero low 32 bits:

- F-f024b4153e2c is invoked a second time.
- Its returned pointer determines the continuation.
- The node’s `+0x04` word is not consulted by this path.

A 64-bit result with only high-half bits set takes the zero-result path instead.

### Stop requested during processing

If the callback sets the stop byte and subsequent dependencies preserve that value, the current iteration still completes its applicable dependency sequence. The next loop admission observes the stop request, clears it, and evaluates finalization.

### Finalization gate

After loop exit:

- Null pointer at `+0x20`: no pointee read and no F-unknown-02 call.
- Nonnull pointer with zero first word: pointee read, no F-unknown-02 call.
- Nonnull pointer with nonzero first word: F-unknown-02 is invoked.

## Uncertainties

- **documented_only — Width provenance:** The layout follows the supplied 32-bit integer and pointer representation. The original ABI, pointer provenance rules, endianness, and signed-character convention are not independently established. Endianness does not change the numerical low-half result test described here.
- **documented_only — Address arithmetic:** Constant byte offsets are visible, but overflow behavior for invalid base addresses is not established. Valid, addressable storage is a precondition.
- **documented_only — Dependency semantics:** Synchronization, waiting, list manipulation, and cleanup interpretations are inferred intent, not established contracts.
- **documented_only — Count meaning:** The signed field controls eligibility only. The function does not directly increment, decrement, or reconcile it with node availability.
- **documented_only — Lifetime and ownership:** No allocation, reference-counting, or release contract is available.
- **documented_only — Indirect calls:** Concrete targets, full calling conventions, exception behavior, and nonreturning behavior are unknown.
- **documented_only — High result half:** Its extraction appears in the supplied representation, but it has no established observable effect on valid nonnull-node execution.
- **documented_only — Concurrent access:** Atomicity, visibility, race freedom, and permitted reentrancy are not established.
- **documented_only — Termination:** Neither eventual observation of a stop request nor eventual return from dependencies is guaranteed.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence includes field offsets and represented widths, signed positive-count comparison, call ordering, node dispatch layout, low-32-bit result branching, fresh post-callback reads, direct writes, and finalization gating.

Worker-loop, synchronization, and container-management interpretations are explicitly separated from those observations. No dependency implementation, original type definition, runtime trace, or external behavioral document was available. No runtime tests have been performed.
