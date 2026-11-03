<!-- Trusted provenance: F-daef4b52b13b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-daef4b52b13b

## Purpose

F-daef4b52b13b selects one of three result-producing paths for a supplied object reference, with a common setup dependency and a common finalization dependency.

The established behavior is:

- A nonzero result from the first predicate selects the fixed 32-bit result `2`.
- Otherwise, a zero result from the second predicate selects a 32-bit value stored at object offset `0x1c`.
- Otherwise, the result comes from an indirect operation on an object obtained through F-a6226d4de001.

Each normally completing path invokes F-fd58bea73663 after selecting its result and before returning.

The predicates’ meanings, the meaning of result `2`, and the semantic roles of the object fields are unknown. Terms such as “predicate,” “setup,” and “finalization” describe their positions in this function’s behavior, not verified domain semantics or ownership guarantees.

## Interface

### Inputs

The function has four input positions and a 32-bit result.

| Input | Established use |
|---|---|
| First input | A 32-bit object reference/address value. Passed to setup and both predicates; also used as the base for direct field accesses and as an argument to F-a6226d4de001. |
| Second input | A 32-bit value forwarded unchanged to the indirect operation, but only on the delegated path. |
| Third input | A 32-bit value with no direct use in the supplied function. It is not forwarded to any visible dependency. |
| Fourth input | A 32-bit value copied into a local writable slot before setup. The slot’s address is supplied to F-3794f888f18b. Its later contents supply the argument to F-fd58bea73663. |

The evidence supports a 32-bit address-and-word layout for the accesses shown. It does not establish a portable interface for wider pointers, an exact calling convention, signedness for opaque values, or source-level public types.

The second and fourth inputs are opaque values. They may encode addresses or handles, but that interpretation is not established. The function does not directly dereference either one.

### Result

On normal completion, the function returns one 32-bit value:

- `2` when the first predicate reports any nonzero value;
- the word read at first-input-relative offset `0x1c` when both predicates report zero;
- the indirect operation’s returned word when the first predicate reports zero and the second reports nonzero.

The stored-field and delegated results are not normalized. In particular, zero, `2`, and values with the high bit set are all passed through if produced by the selected path.

F-fd58bea73663 is called after the result is selected. Its own result, if any, does not replace the selected result.

### Layout requirements

For paths that access the first object directly:

| Byte offset | Access |
|---|---|
| `0x18` | Read one 32-bit word for an argument to the indirect operation. Used only on the delegated path. |
| `0x1c` | Read one 32-bit word as the function result. Used only on the stored-value path. |

The delegated path additionally requires the object returned by F-a6226d4de001 to support this layout:

- Its initial 32-bit word supplies a dispatch-table address.
- At byte offset `0x9c` from that address, a callable target is obtained.
- That target is invoked with the returned object as its first argument, the word from the original object at offset `0x18` as its second argument, and the function’s second input as its third argument.

No other object fields, table entries, or allocation extents are established. These offsets are byte displacements, not element indices.

## Behavior

### Setup and path selection

Before either predicate is called, the function copies the fourth input into a local 32-bit slot. It invokes F-3794f888f18b with the address of that slot and the first input.

Because setup receives the slot’s address, the later finalization argument must not be assumed to equal the original fourth input. It is the slot’s contents when finalization is invoked. The visible function performs no explicit assignment to that slot after setup, but an address has been exposed to a dependency; the supplied evidence does not establish whether that address escapes or is subsequently modified indirectly.

After setup returns, F-867a1fcaaaab is called with the first input. Its result is tested only for zero versus nonzero. Positive and negative nonzero values take the same branch.

If this first predicate is nonzero:

- The selected result is `2`.
- F-56e2874b0016 is not called.
- Neither object field at offset `0x18` or `0x1c` is directly read by this function.
- F-a6226d4de001 and the indirect operation are not called.
- Finalization is invoked before the function returns.

This does not imply that the object is untouched: setup, the first predicate, and finalization may have effects that are unknown here.

### Stored-value path

If the first predicate returns zero, F-56e2874b0016 is called with the first input. Its result is also tested solely for zero versus nonzero.

When that second result is zero, the function reads the 32-bit word at object offset `0x1c` and selects that word as its result. It then invokes finalization and returns the selected value.

The field read occurs after both predicates have returned. The value is therefore taken from the object’s state at that time, not necessarily its state at function entry. Nothing in the supplied evidence establishes that setup or either predicate preserves that field.

The word is captured before finalization. If finalization subsequently changes the object’s field, that change does not cause this function to reread it or replace the already selected result.

On this path, the function does not directly read offset `0x18`, obtain a delegated object, or invoke the indirect operation.

### Delegated path

When the first predicate returns zero and the second returns nonzero, the function calls F-a6226d4de001 with:

1. The first input.
2. The integer value `1`.

The returned value is used as an object address without a visible null check or other validity check.

The function resolves an indirect target through that object’s dispatch table and invokes it with:

1. The object returned by F-a6226d4de001.
2. The 32-bit word read from offset `0x18` of the original object.
3. The original second input.

The indirect operation’s 32-bit result becomes the selected result. Finalization is then invoked, and the selected result is returned.

The offset-`0x18` read and dispatch-table resolution occur after F-a6226d4de001 returns and before the indirect call. Their relative evaluation order is not established by the supplied representation. A contract must not require one of those independent reads to precede the other.

The function does not directly read offset `0x1c` on this path.

### Normal completion and failure boundaries

Every visible normal return follows one invocation of F-fd58bea73663. There is no retry, fallback after delegated failure, explicit status translation, or separate visible error channel.

This is not an exception-safety or fault-recovery guarantee. The evidence shows no mechanism ensuring finalization if an earlier dependency fails to return normally or if an object or dispatch-table access faults. Whether dependencies can throw, terminate execution, transfer control elsewhere, or report failure through ordinary values is unknown.

## Side effects

The only explicit write shown is initialization of the local slot from the fourth input. No direct write to the original object, delegated object, dispatch table, or global state is shown.

Nevertheless, every called dependency may have additional effects. In particular:

- Setup receives a writable slot address.
- Both predicates receive the original object reference.
- F-a6226d4de001 receives the original object reference.
- The indirect operation receives the delegated object and two opaque values.
- Finalization receives the slot’s current 32-bit contents.

No ownership transfer, reference-count change, lock acquisition, lock release, allocation, or deallocation can be established from those calls alone. The names “setup” and “finalization” must not be treated as evidence for any such action.

The relative order of the visible calls is fixed:

- Setup precedes the first predicate.
- The first predicate precedes the second predicate, when the second is reached.
- The second predicate precedes delegated-object acquisition, when that path is reached.
- Delegated-object acquisition precedes the indirect operation.
- Result selection precedes finalization.
- Finalization precedes normal return.

Dependencies must not be assumed to preserve global state or object contents. There is also no visible synchronization establishing safe concurrent access to any object or field.

## Dependencies

| Opaque ID | Visible interface and role |
|---|---|
| F-3794f888f18b | Called once before path selection with the address of the local 32-bit slot and the first input. Any returned value is unused. Mutation of the slot is possible through the supplied address. |
| F-867a1fcaaaab | Called once after setup with the first input. Zero permits further selection; any nonzero value selects result `2`. |
| F-56e2874b0016 | Called only if the first predicate returns zero. Receives the first input. Zero selects the stored-value path; nonzero selects delegation. |
| F-a6226d4de001 | Called only on the delegated path, with the first input and `1`. Its result supplies the delegated object address. |
| F-fd58bea73663 | Called once on each normally completing path with the current contents of the local slot. Its returned value is unused. |

The indirect dependency has no supplied opaque ID. Its identity is determined dynamically by the returned object’s initial word and the dispatch-table entry at byte offset `0x9c`. Its visible invocation has three arguments and supplies a 32-bit result. Its semantic contract remains unknown.

## Acceptance examples

These are conditional static acceptance examples, not results of executed tests. Each assumes that required accesses are valid and all reached dependencies return normally.

1. **First predicate reports a positive nonzero value.**  
   If F-867a1fcaaaab returns `1`, the result is `2`. The second predicate, delegated-object acquisition, and indirect operation are skipped. Finalization still occurs.

2. **First predicate reports a negative nonzero value.**  
   If its 32-bit result is interpreted as `-1`, the behavior is the same as for any other nonzero result. There is no positive-only condition.

3. **Both predicates report zero.**  
   If the word at offset `0x1c` is `0x89abcdef` when read, the returned word is `0x89abcdef`. No delegated operation is invoked.

4. **A predicate changes the stored value.**  
   If the object initially holds `7` at offset `0x1c`, but a predicate changes it to `11` before returning zero, the stored-value path returns `11`, provided no intervening change occurs before the read.

5. **Finalization changes the stored value.**  
   If the stored-value path captures `11` and finalization later changes that field to `13`, the function still returns `11`.

6. **Delegation is selected.**  
   If the predicates return zero and nonzero respectively, F-a6226d4de001 receives the first input and `1`. The selected indirect operation receives the returned object, the word at original-object offset `0x18`, and the second input. If it returns zero, this function returns zero after finalization.

7. **Setup replaces the slot contents.**  
   If the fourth input is value A and setup changes the local slot to value B, finalization receives B, assuming no subsequent indirect modification of that slot.

8. **The third input changes.**  
   Holding other inputs and relevant external state fixed, changing only the third input does not change any explicit argument, branch condition, or memory access shown in this function. This does not assert behavior for callees that inspect undocumented ambient calling state.

9. **Delegated-object acquisition returns an invalid address.**  
   No defined fallback result is established. The function attempts dispatch-related accesses without a visible validation branch; a safe return of `2` or the stored field must not be assumed.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of the predicates, fields, fixed result `2`, and second input.
- Whether the fourth input represents a resource and whether finalization releases anything.
- Whether setup retains the local slot’s address.
- Exact callable identity and validity constraints for the indirect target.
- Object lifetimes, alignment requirements, aliasing constraints, and concurrency guarantees.
- Dependency effects on globals, object fields, ownership, and memory.
- Dependency failure, exception, and nonlocal-control-flow behavior.
- Exact machine calling convention and signedness of opaque words.

Valid address calculations and readable storage are preconditions for the direct accesses on the chosen path. No bounds checks or overflow checks are visible. Behavior must not be generalized to out-of-range address addition, address wraparound, or inaccessible storage.

## Evidence

This contract is based solely on the supplied static function representation and dependency-ID mapping. Established facts are limited to visible argument use, 32-bit word accesses, byte offsets, zero/nonzero conditions, call ordering, result selection, and the absence of visible validation or recovery branches.

Interpreting the initial word as a dispatch-table address is supported by its use to obtain a callable target. Higher-level interpretations—such as a particular object model, resource guard, or ownership protocol—remain inferred intent, not established requirements.

No implementation dependencies were inspected. No runtime tests, smoke runs, memory observations, or behavioral experiments have been performed.
