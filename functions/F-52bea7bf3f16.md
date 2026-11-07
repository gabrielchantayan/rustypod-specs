<!-- Trusted provenance: F-52bea7bf3f16; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-52bea7bf3f16

## Purpose

F-52bea7bf3f16 conditionally obtains a 32-bit candidate through a primary dependency sequence or a fallback provider, then passes a nonzero candidate to a destination associated with its input object.

On normal completion, it returns a 32-bit status:

- **Zero:** the initial availability check rejected the input, or the selected candidate was zero.
- **One:** the selected candidate was nonzero and the delivery and subsequent cleanup-position calls returned normally.

Selecting the primary path suppresses fallback even when the primary candidate is zero. The delivery dependency’s return value is not consulted.

“Workspace,” “context,” “handle,” “lookup,” “candidate,” and “cleanup” describe roles suggested by the call sequence. They do not establish concrete types, initialization guarantees, resource ownership, or successful resource release.

## Interface

### Input and derived addresses

The sole input is a represented 32-bit, address-like value identifying a containing object. Three derived addresses are used:

| Offset from input | Use |
|---|---|
| `0x44` | Availability check and repeated context-value retrieval. |
| `0x204` | Two primary-path lookup operations. |
| `0x8c` | Destination argument for delivery. |

These are address calculations, not direct reads of the corresponding object contents. The visible routine contains no direct load or store through these derived addresses; the dependencies may access them.

The function performs no visible null, alignment, extent, or bounds validation before the first dependency call. Callers must supply an input for which every reached dependency’s arguments are valid.

The representation uses 32-bit address arithmetic. Behavior on signed overflow, address wraparound, or invalid derived addresses is not established. The contract does not convert those cases into supported inputs.

### Return value

Every visible normal-return path produces the 32-bit integer zero or one. Nonzero candidates are tested by equality with zero, not by signed positivity. In particular, the bit pattern `0x80000000` qualifies for delivery.

There is no caller-supplied output buffer.

### Local storage

The representation exposes the following distinct local regions:

| Region | Visible extent | Use |
|---|---:|---|
| Workspace | 72 bytes | Passed to the initial workspace-role call, context preparation, and final workspace cleanup-position call. |
| Context | 12 bytes | Passed to context preparation, checking, handle preparation, and context cleanup. |
| Handle | 4 bytes | Passed through primary-path operations and handle cleanup. |
| Lookup state | 4 bytes | Passed to lookup preparation and status acquisition. |
| Key slot | 4 bytes | Stores a later low-order result word before lookup preparation. |
| Separate word slot | 4 bytes | Passed to status acquisition, then conditionally overwritten with another low-order result word. |
| Status region | 8 bytes | Two consecutive 32-bit words explicitly set to zero before status acquisition. |

These extents describe represented local storage, not proven dependency access footprints or complete object layouts. In particular, passing the 72-byte workspace to a dependency does not prove that the dependency writes or initializes any of it.

## Behavior

### 1. Availability gate

The first dependency call is F-b7abcf60c990 with the input-derived address at offset `0x44`.

If its 32-bit result is zero, the function immediately returns zero. No workspace-role call, context acquisition, primary attempt, fallback, delivery, or later cleanup-position call occurs.

Any nonzero result passes the gate; the result need not equal one.

### 2. Workspace-role call and context preparation

After a successful gate, F-fb045f7ae136 receives the workspace address and zero.

The established behavior is the call and its position before subsequent operations. Whether it initializes storage, allocates resources, leaves bytes untouched, or accesses more than the represented region is unknown. An initialization role is an inference only.

F-d3b5bb23607f then receives the address at input offset `0x44`. This invocation has a represented 64-bit result. F-e268b7be09a5 subsequently receives, in order:

1. The context address.
2. The workspace address.
3. The low-order 32 bits of that result.
4. The high-order 32 bits of that result.

The high half is obtained by a logical 32-bit right shift of the 64-bit bit pattern, followed by selection of the resulting 32-bit word. This specifies bit significance, not an in-memory byte order or semantic signedness.

F-e872853778bf then receives the context address. A zero result bypasses the primary attempt and proceeds to fallback. A nonzero result enables the primary attempt.

### 3. Primary preparation and first key

After the successful context check, the function reads the **handle-argument global**. Its contents are supplied as the third argument to F-7c17f91a10c9, whose arguments are:

1. Handle address.
2. Context address.
3. Value read from the handle-argument global.
4. Zero.
5. Zero.

This is a global read at this point, not a fixed constant or a value captured at function entry. Earlier dependencies may have changed the global. The read occurs after the successful context check and before the next F-d3b5bb23607f invocation.

F-d3b5bb23607f is then called again with the address at offset `0x44`. The low-order 32 bits of this result are stored in the key slot. The earlier 64-bit result is not reused.

F-30d34c0a8352 receives the lookup-state address, the address at input offset `0x204`, and the key-slot address.

After that call returns, the function explicitly writes zero to the first status word and then zero to the second status word. F-9a3e3c13ddff then receives:

1. Lookup-state address.
2. Separate word-slot address.
3. Address of the two-word status region.
4. Zero.

There is no visible direct initialization of the separate word slot before this call. Its role may be output-only, in/out, or dependent on indirect initialization; the supplied evidence does not resolve that interface requirement.

After the call, only the first status word controls the next visible branch. The second word is not directly tested.

### 4. Conditional key refresh and handle operations

If the first status word is zero, the function skips the remaining primary operations and proceeds to handle cleanup, then fallback.

If that word is nonzero:

1. F-d3b5bb23607f is called a third time with the address at offset `0x44`.
2. Its low-order 32 bits overwrite the separate word slot.
3. F-d04a003cf624 receives the address at offset `0x204` and the separate word-slot address.
4. Its 32-bit result is passed as the second argument to F-b304c40f7b39, with the handle address as the first argument.

The three context-value retrievals are distinct calls. There is no visible equality or consistency check among their results.

Next, F-fb37bbc180ad is invoked on the handle:

- If the first result is zero, this sequence stops after one invocation.
- If the first result is nonzero, a second invocation occurs.
- If the second result is zero, this sequence stops after two invocations.
- If both tested results are nonzero, a third invocation occurs. Its result is ignored.

After the third invocation returns, F-4d0908b456c1 is called with no explicit arguments. Its 32-bit result becomes the primary candidate, and primary selection is recorded.

The third handle operation is required even though its result is unused. Its effects cannot be replaced by an additional condition or omitted.

### 5. Handle cleanup and selection boundary

Every normally completing path that called F-7c17f91a10c9 subsequently calls F-3cdbb43c2646 with the handle address.

This cleanup-position call occurs before fallback or delivery. Its actual release, mutation, and ownership effects are unknown.

If primary selection was recorded, fallback is skipped regardless of the primary candidate’s value. A zero result from F-4d0908b456c1 therefore produces no fallback retry.

If primary selection was not recorded, fallback follows handle cleanup. If the context check rejected the primary attempt, fallback is reached without handle preparation or handle cleanup.

### 6. Fallback and its global reads

Fallback first calls F-9d0ecc5f6adf with no explicit arguments.

After that call returns, two distinct globals are read:

- The **fallback second-argument global** supplies the second argument to F-5710a8628cba.
- The **fallback third-argument global** supplies its third argument.

The first argument is the 32-bit result from F-9d0ecc5f6adf.

Both global reads occur after F-9d0ecc5f6adf returns and before F-5710a8628cba is invoked. The supplied representation does not establish an evaluation order between those two reads. Their argument positions are fixed even though their relative read order is not.

These globals are distinct from the handle-argument global. None is established to be constant or immutable. Any preceding dependency, including F-9d0ecc5f6adf, may have changed their contents.

The 32-bit result from F-5710a8628cba becomes the fallback candidate. A zero result does not cause another provider call or retry.

### 7. Delivery, final cleanup, and return

For either selected path:

- A zero candidate suppresses delivery and establishes final status zero.
- A nonzero candidate causes F-4225f441f48b to receive the address at input offset `0x8c`, the candidate, zero, and zero. Final status is one.

The delivery result is ignored. Status one is not proof of dependency-defined delivery success.

After the delivery decision, F-9beda7955a36 receives the context address. The final workspace cleanup-position dependency then receives the workspace address. Its opaque ID is unavailable.

The function returns after both final calls return normally.

## Side effects

Established direct memory effects include storing selected result words and explicitly zeroing the two consecutive status words. The function also directly reads three distinct globals on the paths described above. It contains no visible direct writes to those globals or the caller’s object.

Dependency-mediated effects remain unrestricted by the available evidence. Dependencies may mutate the input object, globals, reachable storage, or external state. They must not be assumed to preserve global operands for later reads.

Observable ordering includes:

- Availability checking before the workspace-role call.
- The workspace-role call before initial context-value retrieval.
- Context preparation before context checking.
- The handle-argument global read after successful context checking and before the second context-value retrieval.
- Lookup preparation before the two explicit status-word zero writes.
- The third context-value retrieval only after a nonzero first status word.
- Handle cleanup before fallback or delivery.
- F-9d0ecc5f6adf before both fallback global reads.
- Both fallback global reads before F-5710a8628cba, without a specified order between the reads.
- Delivery, when reached, before context cleanup, and context cleanup before workspace cleanup.

No visible operation explicitly transfers ownership of the input object. Ownership of the candidate and resources reachable through local storage is unknown. Cleanup-position calls do not by themselves establish release, reference-count changes, candidate validity, or preservation of other state.

Local storage exists only for this invocation. Whether dependencies retain its addresses, copy its contents, or consume referenced resources is undocumented.

No exception handler, cancellation handling, or abnormal-exit cleanup guarantee is visible. The described cleanup order applies to normal control flow.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-b7abcf60c990 | Receives offset `0x44`; its zero/nonzero result gates all later work. |
| F-fb045f7ae136 | Receives workspace address and zero before context acquisition; initialization effects are unknown. |
| F-d3b5bb23607f | Supplies one represented 64-bit context value and up to two later low-32-bit keys. |
| F-e268b7be09a5 | Receives context, workspace, and the initial result’s low and high halves. |
| F-e872853778bf | Gates the primary attempt using the context. |
| F-7c17f91a10c9 | Receives handle, context, a freshly read global operand, zero, and zero. |
| F-30d34c0a8352 | Receives lookup state, offset `0x204`, and a key address. |
| F-9a3e3c13ddff | Receives lookup state, separate word slot, two-word status region, and zero. |
| F-d04a003cf624 | Receives offset `0x204` and the refreshed key address; supplies a 32-bit value. |
| F-b304c40f7b39 | Receives the handle and that supplied value. |
| F-fb37bbc180ad | Performs one to three handle operations; only the first two results are tested. |
| F-4d0908b456c1 | Supplies the primary candidate without explicit arguments. |
| F-3cdbb43c2646 | Receives the handle at the primary cleanup position. |
| F-9d0ecc5f6adf | Supplies fallback’s first argument before its two global operands are read. |
| F-5710a8628cba | Receives the preceding result and two distinct global-read values in their respective second and third argument positions; supplies the fallback candidate. |
| F-4225f441f48b | Receives offset `0x8c`, a nonzero candidate, zero, and zero. |
| F-9beda7955a36 | Receives the context at the final context-cleanup position. |

The workspace cleanup dependency’s opaque ID is **documented_only**. Its argument and call position are established, but its internal behavior is not.

## Acceptance examples

These are conditional specifications, not execution results. Each assumes valid reached arguments and normal dependency returns.

1. **Availability rejection.** If F-b7abcf60c990 returns zero, the function returns zero without calling F-fb045f7ae136 or any later dependency.

2. **Context rejection with successful fallback.** If the gate passes, the context check returns zero, and fallback supplies `0x12345678`, delivery receives that exact candidate. No handle preparation occurs. Context cleanup precedes workspace cleanup, and the return is one.

3. **Zero fallback candidate.** If fallback supplies zero, delivery is absent, both final cleanup-position calls occur, and the return is zero.

4. **Status rejection.** If the first status word is zero after F-9a3e3c13ddff, there is no third context-value retrieval and no F-fb37bbc180ad call. Handle cleanup precedes fallback.

5. **First or second handle result rejects selection.** A zero first result permits exactly one handle operation; a nonzero first and zero second permit exactly two. Neither case reaches primary candidate extraction. Both proceed through handle cleanup to fallback.

6. **Ignored third result.** If the first two handle results are nonzero, a zero third result still permits F-4d0908b456c1 to run.

7. **Zero primary candidate.** If F-4d0908b456c1 returns zero, fallback remains suppressed. Handle cleanup and final cleanup occur, delivery is absent, and the return is zero.

8. **High-bit candidate.** A candidate of `0x80000000` causes delivery and status one; it is not rejected as nonpositive.

9. **Different retrieval results.** Distinct values from successive F-d3b5bb23607f calls are permitted. The first supplies both context halves; later calls supply their own low-order words.

10. **Changed handle global.** If an earlier dependency changes the handle-argument global before its read, handle preparation receives the value read afterward, not an entry-time snapshot.

11. **Changed fallback globals.** If F-9d0ecc5f6adf changes either fallback global before returning, the subsequent reads occur after that change. The second and third argument positions remain distinct; no relative read order is required.

12. **Workspace contents remain unspecified.** The call to F-fb045f7ae136 alone provides no acceptance requirement that the workspace become zeroed or otherwise initialized. Such a requirement would need independent dependency evidence.

## Uncertainties

**Established static evidence:** visible branches, call ordering, global-read evaluation boundaries, explicit local stores, represented storage extents, 32-bit candidate handling, initial 64-bit splitting, and normalized return behavior.

**Inferred intent:** workspace preparation, context construction, lookup, delivery, and cleanup. These role labels do not establish successful initialization, acquisition, delivery, or release.

**Documented_only unknowns:**

- Dependency signatures, calling conventions, signedness, access footprints, and error conventions.
- Exact contents, declared types, load widths, mutability, synchronization, and meanings of the three globals.
- Relative evaluation order of the two fallback global reads.
- Writes and initialization guarantees, if any, of F-fb045f7ae136.
- Complete layouts and alignment requirements of local and embedded regions.
- Whether the represented 64-bit result is a native scalar or a calling-convention artifact.
- The separate word slot’s initialization requirements at status acquisition.
- Allocation, ownership, aliasing, pointer retention, release, and reference-count behavior.
- Implicit state used by argument-free dependencies.
- Thread safety, reentrancy, concurrent mutation, and abnormal-exit behavior.
- Identity and effects of the workspace cleanup dependency.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Prior prose, comments, and apparent domain names are not independent evidence.

No dependency implementations were available. No runtime execution, smoke checks, or tests have been performed. The acceptance examples describe conditional obligations, not observed outcomes.
