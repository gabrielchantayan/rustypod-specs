<!-- Trusted provenance: F-c8eefdf96f4b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c8eefdf96f4b

## Purpose

F-c8eefdf96f4b conditionally updates a caller-supplied state object, inserts an object into one link structure, and drains a separate structure through two dependency calls per removed node.

A 32-bit field at state offset +16 controls the operation. When its initially observed value is zero, the function makes no dependency calls, does not inspect either link structure, and returns the state address paired with the second argument. Otherwise, it calls F-43e0a1831120, performs an insertion using the resulting state, and processes the head at state offset +0 until a condition evaluation observes zero.

“Head,” “node,” and “insertion” describe the visible access pattern. Application-level types and ownership semantics are not established.

## Interface

The function accepts four arguments and returns a packed 64-bit value.

| Argument | Established role |
|---|---|
| First | State address, represented as 32 bits in the supplied operations. |
| Second | 32-bit value supplying the upper return word on the zero-gate path. |
| Third | Not referenced by the visible body. |
| Fourth | 32-bit value copied into mutable temporary storage and passed by address to F-43e0a1831120 on the nonzero path. |

The return layout is:

| Bits | Contents |
|---|---|
| 31–0 | The supplied state address’s 32-bit representation. |
| 63–32 | A path-dependent 32-bit word specified under Behavior. |

The second argument contributes its 32-bit bit pattern on the zero-gate path; the packing does not sign-extend it into the lower word. No Boolean, status, count, or signed interpretation of the complete result is established.

### Observable storage layout

Offsets are byte offsets. Every listed field is 32 bits wide.

| Storage | Offset | Observed role |
|---|---:|---|
| State | +0 | Drain head, reread at each loop condition. |
| State | +4 | Existing insertion head, subsequently replaced. |
| State | +16 | Initial gate and, on later reads, an object address. |
| Object selected before F-43e0a1831120 | +8 | Value copied into a dependency-visible temporary. |
| Object selected after F-43e0a1831120 | +12 | Destination for the prior insertion head. |
| Drain node | +0 | Next-node address. |
| Drain node | +4 | Second argument of the first per-node call. |
| Drain node | +8 | First argument of the first per-node call. |

The before-call and after-call objects need not be identical. No field meanings are established for unlisted offsets.

The contract depends on 32-bit address-bearing words. Replacing these fields with wider native pointers would change the layout and is not justified by the supplied evidence.

## Behavior

### Initial gate

The function first reads state +16 into a mutable temporary word, here called the **return temporary**. It selects the second argument as the provisional upper return word.

If the initial +16 value is zero:

- No referenced object is dereferenced.
- Neither dependency is called.
- State +0 and state +4 are not examined.
- No direct state-field write occurs.
- The upper return word is the second argument’s 32-bit value.
- The lower return word is the supplied state address.

A nonempty drain structure therefore has no effect on this path.

### Preparing the dependency call

If the initial +16 value is nonzero, the function reads state +16 again and uses this second value as an address. It reads the 32-bit word at that address +8.

The first +16 read controls the branch and initializes the return temporary. The second selects the object for the +8 read. No dependency call separates these reads, but the visible behavior contains both reads rather than a single reused value.

The function prepares two additional mutable temporary words:

- A payload temporary containing the object’s +8 word.
- An argument temporary containing the fourth argument.

It then calls F-43e0a1831120 with these four arguments, in order:

1. Address of the argument temporary.
2. The supplied state address.
3. Address of the payload temporary.
4. Address of the return temporary.

These are mutable temporary addresses, not merely by-value input copies. The dependency can change their contents and can access the state directly. The caller does not consume the dependency’s return value.

The visible body supplies no guarantee that F-43e0a1831120 preserves state fields, referenced storage, or globals.

### Insertion after F-43e0a1831120

After a normal dependency return, the function reloads state +16 and saves that 32-bit value as the insertion target address. It then performs the following ordered effects:

1. Reads the current state +4 value.
2. Writes that value to the saved target address +12.
3. Writes the saved target address to state +4.

Thus, in an ordinary non-aliasing layout, the target’s +12 field receives the former insertion head, and state +4 becomes the target address.

Target selection uses the post-call state +16 value, not the initial gate value or the return temporary. The saved target address remains the value written to state +4 even if the preceding target write aliases and changes state +16.

There is no null or validity check on the post-call target. A zero or otherwise unsuitable value still leads to an attempted write at target +12.

Aliasing is not prohibited by the visible body. In particular, the target +12 destination could overlap state +0 or a dependency-visible temporary. The ordered writes must therefore not be replaced by an abstract insertion that assumes unrelated storage. Subsequent draining and return selection use the values actually present after those writes.

### Drain-condition evaluation

After insertion, every drain-condition evaluation performs these actions in order:

1. Reads state +0 into a saved node address.
2. Samples the current return temporary into the working upper return word.
3. Tests whether the saved node address is zero.

This includes the final condition evaluation that observes a zero head. An initially empty post-insertion drain structure still causes the return temporary to be sampled.

A nonzero saved node address starts one drain iteration.

### Processing one node

For each nonzero saved node address, the function:

1. Reads the node’s +0 next address.
2. Writes that next address to state +0.
3. Reads the node’s +8 and +4 fields for the first dependency call.
4. Calls F-ff1bdc376dcd with the +8 value, the +4 value, and a 32-bit zero.
5. If that call returns normally, calls F-ff1bdc376dcd with the saved node address, a 32-bit one, and a 32-bit zero.

The relative evaluation order of the node’s +8 and +4 argument reads is not established. Both occur after the state-head advance and before the first per-node call.

The second call uses the saved node address. The visible body does not reread the node’s fields after the first call. It also does not inspect either dependency return value.

The next condition evaluation rereads state +0. Traversal is therefore not restricted to the original sequence of next pointers. Either per-node call can affect subsequent processing if it changes the state head.

### Return selection

When a condition evaluation observes a zero head, the function returns with the upper word equal to the return temporary’s value sampled during that final condition evaluation.

This value is not necessarily the value that F-43e0a1831120 left in the temporary. That dependency receives its address and could retain or expose it to F-ff1bdc376dcd. Either per-node call could then change the temporary while its lifetime is still valid. Aliasing with an insertion or other reached write could also change it.

Equality with F-43e0a1831120’s outgoing temporary value requires an explicit assumption that no intervening action changes the temporary before the final sample. Valid lifetime alone does not imply preservation.

The upper return word is not necessarily the current state +16 value, either object address, the second argument, or a processed-node count.

The final return preparation copies the selected upper word back into the return temporary and packs it with the original supplied state address. No further dependency call appears between the final condition sample and the return.

## Side effects

### Direct writes and indirect effects

On the nonzero path, the visible body initializes three temporary words, writes the insertion target’s +12 field, replaces state +4, and writes state +0 once per processed node. It also performs the final temporary update used in return preparation. The zero-gate path has temporary assignments but no direct state-field writes.

F-43e0a1831120 can write through the temporary addresses and state address it receives. Effects of both dependencies extend beyond what this caller establishes; neither is known to preserve unrelated storage or globals.

The insertion finishes before the first drain-condition evaluation. Each drain-head advance precedes both calls for that node. No rollback is visible if a later access or dependency call fails to complete.

### Ownership and lifetime

The two-call sequence per node suggests processing associated storage followed by processing the node itself. Deallocation is inferred intent, not an established contract.

The supplied body does not prove:

- That F-ff1bdc376dcd frees or destroys anything.
- That its second argument denotes a size, count, or ownership mode.
- That either call transfers ownership.
- That either dependency preserves any allocation.

A node must remain readable through the field reads needed for the first call. The visible body makes no subsequent node dereference before passing its saved address to the second call. Whether that second call accepts an address whose allocation has already been released is a dependency-contract unknown.

The temporary addresses refer to storage valid during this invocation. Retaining an address does not itself establish a lifetime violation: another dependency could use it before the invocation ends. No persistent lifetime beyond the invocation is established.

### Preconditions, arithmetic, and failure

Successful execution requires valid storage for every reached access:

- State +16 on every path.
- The pre-call object’s +8 field on the nonzero path.
- State +4 and the post-call target’s +12 destination.
- State +0 at each drain condition and head update.
- Each processed node’s +0, +4, and +8 fields.

There is no explicit validation, recoverable error result, cycle detection, or iteration bound. Normal termination requires normal dependency returns and eventual observation of a zero drain head. A cycle or dependency-driven reinsertion may prevent termination.

Address calculations use fixed byte offsets. The visible body establishes no length multiplication, checked arithmetic, or overflow-recovery policy. A wrapped or otherwise invalid address is not made acceptable by the absence of a check.

No synchronization or atomicity guarantee is established.

## Dependencies

### F-43e0a1831120

Called once on the nonzero-gate path and never on the zero-gate path.

Its state changes can affect insertion target selection, the previous insertion head, and the later drain head. Its changes to the return temporary can affect the upper return word, subject to later mutation. It can also expose that temporary’s address for use while the temporary remains alive.

Its implementation, failure behavior, ownership effects, and global effects are **documented_only** unknowns.

### F-ff1bdc376dcd

Called twice for every node whose iteration reaches both calls:

| Call | First argument | Second argument | Third argument |
|---|---|---|---|
| First | Node +8 word | Node +4 word | Zero |
| Second | Saved node address | One | Zero |

The second call follows the first if the first returns normally; no returned value selects whether it occurs.

Effects on the state, exposed temporaries, other memory, and globals are not constrained by the supplied body. Accepted argument domains, release semantics, and failure behavior are **documented_only** unknowns.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reported executions.

### Zero gate despite a nonempty drain head

Given readable state +16 equal to zero and a second argument with bit pattern `89ABCDEF`:

- Neither dependency is called.
- State +0 is not read, even if it contains a nonzero node address.
- No direct state-field write occurs.
- The upper return word is `89ABCDEF`.
- The lower return word is the supplied state address’s 32-bit representation.

### Changed target with an empty preserved drain head

Suppose the initial gate is nonzero and F-43e0a1831120 returns with:

- A valid target address in state +16.
- A value H in state +4.
- A value R in the return temporary.
- Zero in state +0.

Additionally, suppose the insertion writes are valid and preserve both state +0’s zero value and the return temporary’s value R, and no other mutation changes them before the condition sample.

Then the target’s +12 destination receives H, state +4 receives the saved target address, no per-node calls occur, and the upper return word is R.

The preservation assumptions are necessary. The dependency’s outgoing values alone do not establish an empty drain or an unchanged return temporary after insertion.

### Insertion destination aliases the drain head

Suppose the target +12 destination is state +0, the saved state +4 value is a valid nonzero node address H, and the remaining accesses are valid.

The first insertion write changes the drain head to H. An outgoing zero drain head from F-43e0a1831120 does not prevent processing: the next condition observes the post-insertion head.

### Two-node drain without intervening mutation

Suppose the post-insertion head selects two valid nodes followed by zero. Assume the head updates do not alter the fields about to be read, and the dependencies do not change the head or return temporary.

The first node is unlinked before its two calls. The second node is then unlinked before its two calls. Four per-node calls occur, state +0 is zero afterward, and the upper word is the unchanged return temporary sampled by the final condition.

### Per-node mutation of the return temporary

Suppose F-43e0a1831120 leaves R in the return temporary and exposes its address to F-ff1bdc376dcd. During processing, a per-node call writes T through that address while the temporary is alive. Assume no later mutation changes it and the drain eventually ends.

The upper return word is T, not R.

### Dependency replaces the drain head

Suppose an iteration advances the head to one address, but either per-node call replaces state +0 with another valid node address.

The next condition uses the replacement address. It does not continue exclusively along the original links.

### Invalid post-call target

Suppose the initial gate is nonzero but F-43e0a1831120 leaves state +16 zero.

The function attempts the target +12 write without a guard. No successful return or recoverable error result is established for an invalid destination.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the state fields and payloads.
- Actual ownership, release, and allocation semantics.
- Caller guarantees about aliasing and storage separation.
- Whether dependencies retain or share temporary addresses.
- Reentrancy, concurrency, and synchronization rules.
- Dependency behavior on invalid inputs or resource failures.
- Validity of tagged or unusual address encodings.
- The reason for the unused third argument.
- The application interpretation of the packed result.

Link insertion and release-like processing are inferred intent. The read/write ordering, call arguments, gate behavior, and return-word selection described above are established by the supplied static representation.

## Evidence

The supplied function body establishes two pre-call reads of state +16 on the nonzero path, three mutable temporary arguments to F-43e0a1831120, post-call target selection, ordered insertion writes, head-first unlinking, and two ordered F-ff1bdc376dcd calls per completed iteration.

It also establishes that every drain condition samples the return temporary after reading the head, including the final zero-head condition. It does not establish preservation of that temporary between the first dependency’s return and the final sample.

Only the supplied body and opaque dependency mapping support this contract. No dependency implementation or runtime observations were supplied. No runtime tests have been performed.
