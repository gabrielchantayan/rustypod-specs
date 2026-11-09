<!-- Trusted provenance: F-1d9c46dffc3c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1d9c46dffc3c

## Purpose

F-1d9c46dffc3c repeatedly processes a polymorphic input object using a mutable context, an auxiliary argument, and an optional observer. Each iteration can notify the observer, determine processing eligibility, update flags, invoke processing operations, handle a secondary object, and advance the input.

Established normal-return conditions are:

- A nonzero context byte at offset 0x44.
- A nonzero result from the input callback at dispatch-table offset 0x10.
- A status with any low 12 bits set from primary processing, secondary processing, or either of the first two flag-controlled calls to F-8df67b693c5f.
- Flag bit 0x20 at its designated check.
- A low-12-bit failure from F-671465a4e6f7 **combined with** an odd result from the secondary object’s offset-0x18 callback. That combination invokes its offset-0x04 callback before returning.

A low-12-bit failure from F-671465a4e6f7 does **not** terminate processing by itself. An even offset-0x18 callback result allows execution to continue, with the failed processing status still carried. The input callback at offset 0x14 never terminates the loop through its return value; that value is ignored.

Every explicit normal return invokes a common finalization dependency. Application-level meanings of the objects, flags, status fields, and finalization remain unknown.

## Interface

### Arguments and storage layout

The supplied representation uses 32-bit addresses and four arguments:

| Argument | Established use |
|---|---|
| Context address | Direct byte read at offset 0x44; passed unchanged to context queries and primary processing; its address plus 0x04 is passed to other operations and finalization. |
| Input object address | First 32-bit word identifies a dispatch table. Callbacks are selected at table byte offsets 0x10 and 0x14. Also passed to several dependencies. |
| Auxiliary address | Passed unchanged to processing operations. Initially stored in a local 32-bit address slot. Its structure and nullability are unknown. |
| Optional observer address | Zero suppresses notification. Otherwise its first 32-bit word identifies a dispatch table, with a callback at byte offset 0x08. |

A secondary object, if available, also uses a first-word dispatch-table pointer. Its callbacks are selected at table offsets 0x18 and 0x04.

No explicit address validation, bounds checking, or null check exists for the context or input. Callers must provide storage and callback targets valid for every access reached. Whether a null auxiliary argument is valid depends on the invoked dependencies. Adding 0x04 identifies an address within or relative to the context; the source does not establish a separately owned object there.

### Result

The represented result is 64 bits, formed by concatenating two 32-bit components:

- **Low component:** the selected status word.
- **High component:** initially the input object address; changed to the address of a local address slot if secondary processing is entered.

This is word concatenation, not numeric addition. All 32 status bits are retained even when only the low 12 are tested.

The high component is changed before F-15a5a424befb is invoked. Once changed, it is not restored on later iterations. The returned local-slot address is not the slot’s contents and is not a durable pointer after return. The intended external return convention is documented_only.

### Persistent local state

State retained across iterations consists of:

- A 32-bit carried status, initially zero.
- A 32-bit flag word, initially zero.
- A 32-bit address slot, initially containing the auxiliary argument.
- The high-result component, initially containing the input address.

Primary and secondary processing receive the flag word’s address. No call explicitly receives the address slot’s address as an argument.

## Behavior

### Iteration entry and early exits

At the start of every iteration, the function reads a 32-bit value from a fixed data location, then reads the context byte at offset 0x44.

If the context byte is nonzero, the sampled fixed-data value becomes the low result. The input readiness callback is not called. Execution proceeds to common finalization.

If the byte is zero, the input callback at table offset 0x10 is invoked with the input object address as its sole explicit argument. A nonzero callback result selects the **carried status**, not the callback result, as the low result and proceeds to finalization. On the first iteration, that carried status is zero.

The fixed-data read occurs even when its value will not be used. A context-byte change made by the readiness callback does not trigger another context-byte test during that iteration.

### Observer notification and eligibility

After a zero readiness result, a nonzero observer argument causes its offset-0x08 callback to run with the observer and input addresses, in that order. Its return value is ignored.

The observer runs before eligibility queries. No guarantee is established that it preserves context state, input state, globals, or indirectly accessible storage.

Processing eligibility is then determined as follows:

1. F-10ae07e2dd5c receives the context address.
2. A nonzero result permits processing without invoking F-234222f778c7.
3. A zero result causes F-234222f778c7 to receive the input address.
4. Processing is skipped only when this latter result is exactly −1, represented as the 32-bit all-ones value.

Other negative results from F-234222f778c7 do not cause this skip.

When processing is skipped, the input-advance callback is reached directly. The processing block’s flag reset is not executed on that path.

### Flag-bit-0 update and processing bypass

For an eligible iteration, F-1bb99fef17b5 receives the input address.

If its result is zero, this stage leaves the flags unchanged. Otherwise, F-10ae07e2dd5c is called again with the context address. This second result independently determines bit 0:

- Zero sets bit 0.
- Any nonzero 32-bit value clears bit 0.
- All other flag bits are preserved.

The represented calculation uses unsigned 32-bit subtraction and an unsigned comparison. For results greater than one, any wrapped subtraction result is replaced with zero. Consequently, values such as 0x80000000 and 0xffffffff clear bit 0; they do not set other bits through this calculation.

The two calls to F-10ae07e2dd5c are separate observations. Their results need not agree.

If bit 0 is set after this stage, the function skips primary processing and everything nested under it. It clears the entire flag word and invokes the input-advance callback.

### Primary processing

When bit 0 is clear, the local address slot is set to zero before F-c5d5df1d385e is called. The dependency receives, in order:

1. Context address.
2. Input address.
3. Auxiliary address.
4. Address of the 32-bit flag word.

Its 32-bit result replaces the carried status. If the result masked with 0x00000fff is nonzero, the function immediately selects that full result and proceeds to finalization.

A nonzero status with all low 12 bits clear is accepted by this check. Acceptance does not imply application-level success.

### Secondary-processing eligibility and execution

After an accepted primary result, F-b4f0b3c858e6 receives the context address plus 0x04.

Its zero result suppresses the remaining secondary-processing eligibility checks. For a nonzero result, the function checks flag bit 0. Only if that bit is clear does F-2076b637e64f receive the context address. Secondary processing occurs only if this final query returns zero.

Immediately before secondary processing, the high-result component is changed to the address of the local address slot. F-15a5a424befb then receives:

1. Context address plus 0x04.
2. Input address.
3. Address of the flag word.
4. Auxiliary address.

Its result replaces the carried status. Any nonzero low-12-bit field causes immediate finalization and return of the full status word.

Changing the high-result component does not populate the address slot. The high component continues to identify that local slot even if later primary-processing iterations clear its contents again.

### Stop flag and checked follow-on operations

After primary processing and any secondary processing, the function checks bit 0x20. If set, it exits with the current carried status. Thus, this flag does not suppress the earlier secondary-processing eligibility sequence or secondary processing itself.

If bit 0x20 is clear, bit 0x02 controls a call to F-8df67b693c5f with the context address plus 0x04 and the auxiliary address. Its result replaces the carried status. A nonzero low-12-bit field causes immediate finalization.

Only after that call returns an accepted status is bit 0x08 tested. If set then, another call to F-8df67b693c5f is made with the same arguments. Its result again replaces the carried status and receives the same check.

Bit 0x08 alone does not cause either call: the second call is nested within the bit-0x02 path. The bit-0x08 test occurs after the first call rather than from a flags snapshot taken before both calls.

### Conditional secondary-object handling

If either bit 0x04 or bit 0x10 is set, the function reads the local address slot. When nonzero, its value is captured as the secondary object address, and F-671465a4e6f7 receives:

1. Context address plus 0x04.
2. Captured secondary object address.
3. Auxiliary address.
4. Integer value 1.

The result replaces the carried status.

If its low 12 bits are zero, neither secondary-object callback is invoked here.

If any low 12 bits are set, the secondary object’s table-offset-0x18 callback receives the captured object address. Its result is tested only for bit 0:

- **Odd result:** The object’s offset-0x04 callback is invoked. Its return value is ignored. The function then finalizes and returns the F-671465a4e6f7 status, not either callback result.
- **Even result:** Neither the offset-0x04 callback nor an immediate return occurs. Execution continues, with the failed F-671465a4e6f7 status still carried.

The offset-0x04 callback target is obtained after the offset-0x18 callback has returned; the earlier dispatch state is not established to remain unchanged.

The slot is explicitly zeroed before primary processing, with no explicit subsequent assignment in the supplied representation. A nonzero value therefore requires an effect not explained by the shown direct assignments, such as an indirect write. This path’s practical reachability remains documented_only.

### Final flag-controlled operation and advancement

If execution continues and bit 0x10 is set, F-8df67b693c5f receives the context address plus 0x04 and the auxiliary address once more.

Unlike its earlier uses:

- The return value is ignored.
- The carried status is unchanged.
- No low-12-bit test is performed.

The entire flag word is then reset to zero.

The input’s offset-0x14 callback is invoked with the input address. Its return value is ignored, regardless of whether it is zero or nonzero. The next iteration begins with the fixed-data read and context-byte test.

### Common finalization

Every explicit normal-return path invokes F-2b47ae8dfd37 with the context address plus 0x04. Its return value is ignored. The selected high and low components are then returned.

This is one finalization call per explicit normal return, not per iteration. Faults, exceptions, nonlocal transfers, and non-returning callees are not covered by this guarantee.

## Side effects

Established direct effects include:

- Repeated reads of the fixed 32-bit data location and context byte.
- Reads of dispatch-table pointers and callback targets.
- Local writes to the status, flags, address slot, and high-result component.
- Exposure of the flag word’s address to primary and secondary processing.
- Dependency and callback invocations in the order described above.

No explicit allocation, bulk copy, or direct write into caller-owned storage appears outside effects performed by dependencies and callbacks.

No general retain, release, destruction, or ownership-transfer protocol is established for any argument. The secondary object’s offset-0x04 callback has cleanup-like placement, but that placement does not establish that it frees or releases memory.

Callees may affect state subsequently observed by this function; preservation of globals or indirectly reachable memory must not be assumed. The loop has no fixed iteration bound.

## Dependencies

| Opaque ID | Established role |
|---|---|
| F-10ae07e2dd5c | Context query for initial eligibility and, conditionally, a separate bit-0 update. |
| F-234222f778c7 | Input eligibility query; exactly −1 skips processing when the first context query returned zero. |
| F-1bb99fef17b5 | Input query controlling whether bit 0 is recomputed. |
| F-c5d5df1d385e | Primary operation; receives writable flags and returns a checked status. |
| F-b4f0b3c858e6 | Query using context plus 0x04, preceding secondary-processing eligibility checks. |
| F-2076b637e64f | Context query whose zero result permits secondary processing. |
| F-15a5a424befb | Secondary operation; receives writable flags and returns a checked status. |
| F-8df67b693c5f | Follow-on operation; first two possible results are checked, final bit-0x10 result is ignored. |
| F-671465a4e6f7 | Secondary-object operation; failure requires a further callback decision before immediate return. |
| F-2b47ae8dfd37 | Finalization on every explicit normal return. |

Dynamic callback targets have no supplied opaque IDs and are identified by dispatch-table byte offsets. Their internal behavior and ownership effects are unknown.

## Acceptance examples

These are conditional static consequences, not executed tests. Each assumes valid accessed storage, normally returning callees, and no intervening effects beyond those stated.

1. **Context stop:** A nonzero context byte on entry causes finalization without input or observer callbacks. The low result is the sampled fixed-data word; the high result is the input address.

2. **Readiness stop:** With a zero context byte, an initial offset-0x10 callback result of 7 causes finalization with low result zero, not 7.

3. **Advance is not a stop result:** Eligibility is skipped because the first context query returns zero and F-234222f778c7 returns −1. The offset-0x14 callback runs. Even a nonzero result from it does not prevent the next iteration.

4. **Bit-0 bypass:** The bit-update stage is requested and its context query returns zero. Primary processing is skipped, flags are cleared, and the input advances. A result of 0xffffffff instead clears bit 0.

5. **Status-mask boundary:** A primary result of 0x00001000 passes its check. A result of 0x00001001 immediately finalizes and returns that entire status.

6. **Stop-flag precedence:** After secondary-processing eligibility and any resulting processing, flags retain both 0x20 and 0x02. The function exits without either checked F-8df67b693c5f call.

7. **Nested second call:** With 0x20 clear, 0x08 set, and 0x02 clear, neither checked follow-on call occurs. If 0x02 is also set, the second call occurs only if the first result is accepted and 0x08 is set when subsequently tested.

8. **Ignored final result:** Bit 0x10 reaches the final follow-on operation. A return value of 1 from that operation alone neither causes exit nor replaces the carried status.

9. **High-result transition:** Secondary processing is entered and returns 1. Finalization follows, with the local slot’s address in the high result and 1 in the low result.

10. **Odd object-policy result:** Assuming a nonzero secondary object is available, F-671465a4e6f7 returns 0x1001 and the offset-0x18 callback returns 3. The offset-0x04 callback runs before finalization. The low result is 0x1001.

11. **Even object-policy result continues:** Under the same object-reachability assumption, F-671465a4e6f7 returns 0x1001 and the offset-0x18 callback returns 2. No offset-0x04 callback or immediate return occurs. If execution advances, the next context byte is zero, and readiness returns nonzero, finalization returns the carried 0x1001.

12. **Context stop supersedes carried failure:** After the continuing object-failure case, a nonzero context byte on the next iteration instead selects that iteration’s sampled fixed-data word, not the carried failed status.

## Uncertainties

- **documented_only — Semantic intent:** Domain meanings of objects, flags, statuses, advancement, and finalization are unknown.
- **documented_only — External return convention:** The representation assembles 64 bits, but its intended ABI interpretation, especially the local address in the high component, is unresolved.
- **documented_only — Fixed data:** Meaning, initialization, mutability, and synchronization are unknown.
- **documented_only — Secondary-object production:** No explicit assignment explains a nonzero address slot after its reset.
- **documented_only — Callee effects and ownership:** Dependencies may mutate accessible state; retention, release, allocation, and destruction are not established.
- **documented_only — Concurrency and reentrancy:** No locking, atomic-access, or preservation guarantees are shown.
- **documented_only — Exceptional execution:** Invalid storage, nonlocal exits, exceptions, and non-returning calls have no established recovery contract.

## Evidence

The supplied static representation establishes the argument uses, access widths, dispatch offsets, short-circuit ordering, unsigned bit-update arithmetic, status masks, state assignments, callback-result handling, common finalization, and result concatenation described here.

“Readiness,” “processing,” “advancement,” and “finalization” are role descriptions inferred from call placement, not verified domain semantics. Cleanup-like placement does not prove memory ownership. Secondary-object reachability and the intended external return interpretation remain unresolved.

No runtime tests have been performed.
