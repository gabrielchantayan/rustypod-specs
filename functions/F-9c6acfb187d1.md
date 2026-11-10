<!-- Trusted provenance: F-9c6acfb187d1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9c6acfb187d1

## Purpose

F-9c6acfb187d1 processes an input value against a mutable context and returns a 32-bit status. A classification dependency selects among three processing modes:

- Mode 1 prepares temporary descriptors, then either resolves and dispatches an object or stores a constructed value in context-associated state.
- Mode 2 updates two context bytes, calls a dependency, and conditionally writes two caller-provided output words.
- Mode 3 calls a dependency and conditionally adjusts the default status.

The business meanings of the classifications, objects, flags, and status values are not established. Names such as “classification,” “descriptor,” and “object” below describe observable roles, not recovered domain semantics.

This contract describes the supplied static evidence. No runtime tests have been performed. External dependency behavior is not assumed.

## Interface

### Inputs and result

The visible interface has four arguments:

| Argument | Established representation | Role |
|---|---|---|
| Context | 32-bit address-bearing value | Forwarded to dependencies and used as the base of direct memory accesses. |
| Input | Opaque 32-bit value | Forwarded unchanged to F-e02d54ff56be. |
| First output destination | Address of a writable 32-bit word | Directly written only in mode 2 when the mode’s first byte is nonzero. |
| Second output destination | Address of a writable 32-bit word | Written immediately after the first output under the same condition. |

The result is a 32-bit integer. Its initial value is read from a global default-status word before the classification call. Some branches replace it with a dependency result, replace it with another global word, or add three.

Neither output destination is validated. Null or otherwise invalid destinations are harmless only along paths that do not dereference them, subject to unknown dependency effects and aliasing.

### Directly accessed context layout

Offsets are byte offsets from the context address:

| Offset | Width | Visible use |
|---|---:|---|
| `0xDC` | 32 bits | Address of a secondary state structure. |
| `0x540` | 8 bits | Receives the first byte produced for mode 2. |
| `0x541` | 8 bits | Receives the second byte produced for mode 2. |
| `0x548` | Address only | Passed as an embedded-region address to two dependencies. |

The secondary state structure has a 32-bit field at offset `0x18`. Two mode-1 paths store a dependency result there.

An object returned by F-7b777ffb67c1 is interpreted as beginning with a 32-bit dispatch-table address. Calls are made through entries at byte offsets `0x1C` and `0x18` in that table, with the object address as their argument. No concrete object type or ownership model is established.

### Temporary storage

The function provides dependency-visible addresses for:

- One 32-bit classification payload.
- Two four-byte regions whose first bytes are used as mode-2 values.
- Two four-byte word backing regions.
- Two four-byte flag backing regions.
- An eight-byte temporary.
- Two descriptors initially represented by pairs of pointers into the backing regions.

For clarity, the descriptor used for lookup is the **lookup descriptor**; its backing words are the **first word** and **second word**. The other is the **flag descriptor**, with a **primary flag** and **secondary flag** in the first bytes of its two backing regions.

These labels do not imply a documented external descriptor format. Dependency calls receive descriptor addresses and may mutate their contents.

## Behavior

### Entry and classification

The function first captures the global default status. It initializes the descriptor pointer slots to refer to the local backing regions, but the visible body does not initialize the backing contents themselves.

It then calls F-e02d54ff56be with:

1. The context.
2. The input.
3. The address of the 32-bit classification payload.
4. The address of the first mode-byte region.
5. The address of the second mode-byte region.

Only classification results exactly equal to 1, 2, or 3 receive special handling. Every other result returns the captured default status, without direct output writes or descriptor cleanup.

The initial global read precedes this call. A dependency’s later change to that global does not replace the captured default on paths that retain it.

### Mode 1: descriptor preparation and gate

For classification 1, the following calls occur in order:

1. F-b9acea4b06ad receives the lookup descriptor’s address and the classification payload. Its return value is ignored.
2. F-5b24edfb22c0 receives the flag descriptor’s address. Its return value is ignored.
3. F-b9acea4b06ad receives the eight-byte temporary’s address and the lookup descriptor’s address.
4. F-0882173e5b5b receives the result of the preceding call, the lookup descriptor’s address, and the flag descriptor’s address.
5. F-289ec4311c64 receives the eight-byte temporary’s address.

The result from F-0882173e5b5b controls a short-circuited gate:

- If it is zero, processing proceeds to object lookup. F-6d70e45e29c9 is not called.
- If it is nonzero, F-6d70e45e29c9 is called with the lookup descriptor’s address.
- If that second result is zero, processing proceeds to object lookup.
- Only when both results are nonzero does processing take the constructed-value path.

Cleanup of the eight-byte temporary occurs before this gate is evaluated and before any call to F-6d70e45e29c9.

### Mode 1: constructed-value path

When both gate results are nonzero:

1. The secondary-state address is read from context offset `0xDC` and retained.
2. F-892d80db0918 is called with the context.
3. F-b627bc4d550a is called with the context, the preceding result, and the flag descriptor’s address.
4. Its 32-bit result is stored at offset `0x18` of the retained secondary-state address.
5. F-df68e84bb5d6 is called with the context.

This path does not explicitly replace the returned status, so it retains the entry-captured default.

The store uses the state address captured before the two construction calls. If either call changes context offset `0xDC`, the store still targets the previously captured structure.

### Mode 1: object lookup and common preparation

The alternative gate outcome calls F-7b777ffb67c1 with the context and lookup descriptor’s address.

If it returns zero, no object dispatch occurs and the result remains the captured default.

For a nonzero object address:

1. The function calls its dispatch-table entry at offset `0x1C`. The return value is ignored.
2. It calls F-44362605ab79 with no visible arguments.
3. It calls F-e8e6f4e4cd16 with the context and the preceding result.
4. It reads the primary flag’s first byte.

The flag read therefore occurs after these calls. Its value is not snapshotted before them.

### Mode 1: nonzero primary flag

If the primary flag is nonzero:

1. F-3277c7b39899 is called with the context.
2. F-e2a5066d36a6 is called with the context and object; its return value is ignored.
3. The result is replaced by the current value of a second global status word.

This second global is read after both calls. It is distinct from the default-status word captured at entry.

No type-query dispatch or secondary-flag action occurs on this branch.

### Mode 1: zero primary flag

If the primary flag is zero, the secondary flag is examined. A nonzero secondary flag causes F-74367d9e9387 to be called with the context and object before type-query dispatch. Its result is ignored.

The dispatch-table entry at offset `0x18` is then queried as follows:

- If the first query returns 1, F-e2a5066d36a6 is called with the context and object; its result becomes the function result.
- Otherwise, a second query is made. If it returns 2, F-8fa855e66a61 is called with the context and object; its result becomes the function result.
- Otherwise, a third query is made. If it returns 3, F-8fa855e66a61 supplies the result.
- Otherwise, the fallback path is taken.

The queries are separate calls. The contract does not assume a stable result or a side-effect-free query. The dispatch-table pointer is read for each invocation rather than being guaranteed constant across them.

In the fallback path:

1. F-7b4c17780d18 is called with the context and object; its result becomes the eventual function result.
2. The secondary-state address is read from context offset `0xDC`.
3. F-892d80db0918 is called with the context.
4. F-b627bc4d550a is called with the context, the preceding result, and the flag descriptor’s address.
5. Its result is stored at offset `0x18` of the captured secondary-state address.
6. Context offset `0xDC` is read again, and the field at offset `0x18` of that currently referenced structure is tested.
7. If that field is nonzero, F-6904f7699031 is called with the address at context offset `0x548` and the integer 500, followed by F-ec4cfa552cce with the same address.

The post-store test can inspect a different structure from the one just written if a dependency changed context offset `0xDC`. Consequently, a nonzero stored value alone does not guarantee the final two calls.

### Mode 1: final cleanup

After either main gate path, including a null object lookup, cleanup calls occur in this order:

1. F-289ec4311c64 with the flag descriptor’s address.
2. F-289ec4311c64 with the lookup descriptor’s address.

The function returns the selected status only after these calls complete normally. Their return values are ignored.

### Mode 2

For classification 2:

1. The first mode byte is read and stored at context offset `0x540`.
2. The second mode byte is read and stored at context offset `0x541`.
3. F-0b3e930d3414 is called with the context, classification payload, first mode byte, first word, and second word.
4. The low eight bits of that dependency’s result are used as a byte result.
5. If the bitwise intersection of that result and the first mode byte is nonzero, three is added to the captured default status.
6. If the first mode byte is nonzero, the first word is written to the first output destination, followed by the second word to the second output destination.

The two conditions are independent. A nonzero mode byte permits output writes even if the status is not adjusted.

The first mode byte is used again after the dependency call; the visible body does not preserve a separate pre-call snapshot for those decisions.

Crucially, the visible mode-2 path does not initialize either word before passing its value to F-0b3e930d3414. It also supplies no direct pointer to either word in the classification call. Their meaningful initialization is therefore not established. Output values and behavior dependent on those values are documented_only unless an external mechanism or a corrected representation establishes their provenance.

There are no descriptor cleanup calls in mode 2.

### Mode 3 and other classifications

For classification 3, F-13c744134877 is called with the context and classification payload. A nonzero result causes three to be added to the captured default status; zero leaves it unchanged.

There are no direct output writes or descriptor cleanup calls on this path.

## Side effects

Direct writes are limited to:

- The two context bytes in mode 2.
- The secondary structure’s 32-bit field on the two described mode-1 paths.
- The two caller output words in mode 2 when the first mode byte is nonzero.
- Local descriptor initialization and dependency-mediated temporary changes.

All called dependencies may have additional effects. The evidence does not establish global preservation, memory ownership, allocation, reference counting, locking, or object lifetime.

Calling F-289ec4311c64 establishes a cleanup action, not proof of deallocation. Likewise, the object method at offset `0x1C` must not be assumed to retain, release, or destroy the object.

Output stores occur in argument order. If both output addresses are the same, the second store overwrites the first. If they alias context fields or other reachable storage, those writes may change that storage. The function provides no alias checks, rollback, or transactional guarantees.

## Dependencies

| Opaque ID | Observed role |
|---|---|
| F-e02d54ff56be | Selects processing mode and receives three output-region addresses. |
| F-b9acea4b06ad | Participates in descriptor and temporary preparation through two different visible call shapes. |
| F-5b24edfb22c0 | Receives the flag descriptor for preparation. |
| F-0882173e5b5b | Produces the first gate result. |
| F-289ec4311c64 | Receives temporary or descriptor addresses at cleanup points. |
| F-6d70e45e29c9 | Produces the second gate result when the first is nonzero. |
| F-7b777ffb67c1 | Resolves an object address or zero. |
| F-44362605ab79 | Produces a value forwarded to F-e8e6f4e4cd16. |
| F-e8e6f4e4cd16 | Receives context and that value before flag dispatch. |
| F-74367d9e9387 | Runs conditionally on the secondary flag. |
| F-e2a5066d36a6 | Handles the first query category or the primary-flag branch. |
| F-8fa855e66a61 | Handles the second or third query category. |
| F-7b4c17780d18 | Supplies the fallback result. |
| F-892d80db0918 | Supplies an intermediate construction value. |
| F-b627bc4d550a | Supplies the value stored in secondary state. |
| F-df68e84bb5d6 | Runs after the constructed-value gate path’s store. |
| F-3277c7b39899 | Runs before the primary-flag branch’s handler. |
| F-6904f7699031 | Receives an embedded address and 500. |
| F-ec4cfa552cce | Receives the same embedded address afterward. |
| F-0b3e930d3414 | Supplies the mode-2 byte result. |
| F-13c744134877 | Supplies the mode-3 nonzero/zero decision. |

The two indirect object methods have no supplied opaque IDs. Their behavior remains documented_only.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume valid required memory and normally returning dependencies.

1. **Unrecognized classification:** With entry default 40 and classification 7, the result is 40. Neither output destination is directly written.

2. **Short-circuited lookup:** With classification 1 and a zero first gate result, F-6d70e45e29c9 is not called. If object lookup returns zero, cleanup still receives the flag descriptor and then the lookup descriptor, and the default status is returned.

3. **Constructed-value store:** With both gate results nonzero, a captured secondary-state address A, and construction result 9, A’s field at `0x18` receives 9. F-df68e84bb5d6 follows. Changing the context’s state pointer during construction does not redirect that store.

4. **Changing query results:** With zero primary flag and successive query results 9, 9, and 1, the fallback path is taken. The third result is compared with 3, not reconsidered as a first-category result.

5. **Primary-flag override:** With nonzero primary flag, F-e2a5066d36a6 returning 17, and the override global equal to 62 when read afterward, the function returns 62, not 17.

6. **Mode-2 independent conditions:** If the first mode byte is 4 and the dependency’s byte result is 2, the status remains the captured default, but both output stores occur. Their contents remain subject to the unresolved word initialization.

7. **Mode-2 matching bits:** If the first mode byte is 4 and the dependency’s byte result is 12, the status is default plus three. This example assumes that addition is representable.

8. **Mode-3 decision:** With default 40, classification 3, and a nonzero dependency result, the returned value is 43, regardless of the dependency result’s exact nonzero magnitude.

## Uncertainties

- **documented_only — word initialization:** Mode 2 consumes two words without an established initialization path. Deterministic numeric outputs cannot be promised.
- **documented_only — arithmetic overflow:** Addition is displayed as signed 32-bit integer arithmetic. Behavior at the signed upper boundary is not established; this contract does not impose wrapping or saturation.
- **documented_only — dependency effects:** Dependencies may modify globals, reachable context, descriptors, or aliased storage. Preservation is not inferred.
- **documented_only — ownership:** No allocation, release, ownership transfer, or lifetime guarantee is established.
- **documented_only — failure behavior:** There are no visible validation or recovery branches for invalid addresses, invalid dispatch tables, failing cleanup, or abnormal dependency termination.
- **documented_only — storage representation:** Endianness, alignment requirements, complete descriptor layouts, and complete object layouts are not established.
- **Inferred intent:** The query-driven paths appear to select different object handlers, and the two final embedded-region calls appear related. Their domain meaning, including the unit or significance of 500, is unknown.

## Evidence

The established behavior comes solely from the supplied static function body and opaque dependency mapping. It supports the call order, branch conditions, widths, direct memory offsets, short-circuiting, status selection, cleanup order, repeated indirect queries, and captured-versus-reloaded state addresses described above.

No dependency implementation, runtime trace, external ABI documentation, or test result was supplied. Acceptance examples express consequences of the visible control flow under stated conditions; they do not claim observed execution.
