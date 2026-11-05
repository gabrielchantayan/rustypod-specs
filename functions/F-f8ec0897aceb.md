<!-- Trusted provenance: F-f8ec0897aceb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f8ec0897aceb

## Purpose

F-f8ec0897aceb conditionally retrieves a 32-bit word from a table associated with a caller-supplied object. Retrieval requires a nonzero preliminary gate result, a nonzero initial selector, an unsigned position below a metadata count, and a nonzero table address after any initialization attempt.

An adjustment dependency receives writable copies of the selector and control value. The resulting control value determines whether the position is reversed within the previously read count. The selector is evaluated separately for each table-pointer read; the function does not retain a single selected-slot address across initialization.

Zero is both the ordinary rejection result and a possible retrieved word. There is no separate success indicator.

The apparent purpose is indexed access to one of several tables, optionally in reverse order, with conditional initialization. This domain interpretation is inferred from the accesses, not independently established.

## Interface

The function accepts four inputs and returns one 32-bit value.

| Input or output | Width and interpretation | Established use |
|---|---|---|
| Object address | 32-bit address-bearing value | Base of object-field accesses; passed to adjustment and initialization dependencies |
| Initial selector | Signed 32-bit value | Tested for nonzero, copied into writable invocation-local storage, and subsequently used for table selection |
| Initial control value | Unsigned 32-bit value | Copied into writable invocation-local storage; its low byte is examined after adjustment |
| Position | Unsigned 32-bit value | Compared against a count, optionally reversed, then used as a four-byte element index |
| Return value | 32 raw bits | Zero on rejection or unavailable table; otherwise the loaded word |

No signedness, pointer interpretation, or ownership meaning is established for the returned bits.

### Referenced layout

| Location | Read width | Meaning established by use |
|---|---:|---|
| Object address + 0x40 | 32 bits | Metadata address |
| Metadata address + 0x2e | 16 bits | Unsigned count, extended to 32 bits |
| Object address + 0x3ac + four times the selector evaluated for that read | 32 bits | Table address |
| Final table address + 0x10 + four times the effective position | 32 bits | Returned word |

The first 16 bytes of a selected table precede the indexed words. Their contents and purpose are unknown.

The initial selector check is not a selector-range check. No upper bound, lower bound, or rejection of negative selectors is visible. Adjustment may change the selector to zero or a negative value without causing another selector-validation step.

For successful accesses, every address reached on the chosen path must refer to readable storage. A nonzero pointer value alone does not establish that condition. The representation does not establish byte order, alignment requirements, or the behavior of overflowing address calculations.

## Behavior

### Preliminary gate and selector check

The function initializes invocation-local selector and control storage from the corresponding inputs, then calls F-3b037d560480 with no explicit arguments.

If the gate returns zero, the function returns zero. It performs no object-field or table read on that path.

If the gate returns nonzero, the selector must also be nonzero to proceed. An initially zero selector is rejected before the adjustment dependency is called. The position and control value cannot override this rejection.

The gate is called even when the initial selector is zero or the object address is zero. Statements about omitted reads describe this function’s own accesses, not those performed internally by the gate.

### Count acquisition and unsigned bound

After the gate and selector checks pass:

- A zero object address supplies a count of zero without dereferencing the object.
- A nonzero object address causes a 32-bit metadata-address read at object offset 0x40.
- That metadata address is used for an unsigned 16-bit read at offset 0x2e.
- The count is extended to 32 bits and compared with the input position.

The position must be strictly less than the count under unsigned comparison. Equality fails. Failure returns zero before adjustment or initialization.

The count ranges from 0 through 65,535. Therefore:

- Count zero admits no position.
- A zero object address admits no position.
- At the maximum count, the largest admitted position is 65,534.
- Position 65,535 and all larger unsigned positions are always rejected.

There is no null check on the metadata address. An invalid metadata address does not take an established graceful-failure path.

The count is read once and retained for the remainder of the invocation. Later dependency effects on metadata do not trigger another count read or another position-bound check.

### Adjustment

After the position passes the bound, F-3dfc1640191e receives:

1. The object address.
2. The address of writable 32-bit invocation-local selector storage.
3. The address of writable 32-bit invocation-local control storage.

Subsequent operations use the contents of these storage locations when evaluated. The dependency’s own result is not used as a success or failure condition.

The function performs no selector validation after adjustment. An initially nonzero selector may become zero and still participate in table lookup. Negative adjusted values are likewise not explicitly rejected.

The addresses passed to adjustment remain relevant to later behavior: the evidence does not establish whether the dependency retains them or makes them accessible to another dependency. Consequently, the selector cannot be assumed immutable merely because the adjustment call has returned.

### Effective position

After adjustment returns, the function examines the low eight bits of the control value.

- If the low byte is zero, the effective position is the original position.
- If the low byte is nonzero, the effective position is the saved count minus the original position minus one.

Higher control bits alone do not request reversal. For example, 0x00000100 does not reverse; 0x00000101 does.

The earlier bound guarantees that reversal does not underflow and produces a value from zero through the saved count minus one. The effective position is therefore at most 65,534. Its multiplication by four is at most 262,136; including the 16-byte table prefix gives an offset of at most 262,152.

Those offset calculations do not establish that adding the table address is safe. Table extent and address validity remain separate requirements.

The reversal decision occurs before the first table-pointer read and before any initialization call. The control value is not examined again afterward. Changing control storage during initialization does not itself cause the effective position to be recomputed.

### Table availability and initialization

The function evaluates the current selector, forms the corresponding object-relative table-pointer address, and reads its 32-bit contents.

If this first availability read is nonzero, initialization is skipped.

If this first read is zero, the function:

1. Calls F-60950fd9e511 with the object address.
2. Ignores that dependency’s result.
3. Evaluates the selector again.
4. Computes a table-pointer address from that newly evaluated selector and performs a second availability read.

A zero second availability result causes a zero return. A nonzero result permits the final load sequence.

**The second read is not guaranteed to address the first read’s slot.** The selector’s storage was exposed to F-3dfc1640191e. If that dependency retained the address and F-60950fd9e511 can use it, initialization could change the still-live selector even though it receives no selector argument explicitly.

For example, the first read may inspect selector 2’s slot and find zero; initialization may change the selector to 3; the second availability read may then inspect selector 3’s slot. No comparison requires these selections to match.

If the selector remains unchanged, the two availability reads address the same slot. That is a conditional result of selector stability, not an unconditional contract.

Initialization is attempted at most once. A zero second availability read does not cause another initialization attempt, another selector search, or a return to the original slot.

### Final base and word reads

After either availability read produces a nonzero value, the function evaluates the selector again and performs another table-pointer read. This newly read pointer supplies the base for the final 32-bit word access:

**final table address + 0x10 + four times the effective position**

Neither an earlier slot address nor the earlier nonzero pointer value is reused as the final base.

The function’s table-pointer read sequence is therefore:

| Path | Table-pointer reads |
|---|---|
| First availability read is nonzero | First availability read, then final-base read |
| First read is zero; second read is zero | First availability read, initialization, second availability read |
| First read is zero; second read is nonzero | First availability read, initialization, second availability read, final-base read |

Each read uses the selector evaluated for that read. Repeated access to one slot requires selector stability. Repeated observation of one pointer value additionally requires stability of that slot’s contents.

The final-base read is not followed by another null check. If it differs from the pointer that passed the availability check, the final access uses the newly read value. No synchronization or pointer-stability guarantee is established.

The final word is returned unchanged, including when it is zero.

## Side effects

The function has no explicit store into the object, metadata, selected table, or caller-owned argument storage. Its explicit writable selector and control storage is invocation-local.

Dependency calls are observable and ordered:

- F-3b037d560480 is always called.
- F-3dfc1640191e is called only after the gate, initial-selector, and position checks pass.
- F-60950fd9e511 is called only after a zero first table-pointer read.

Dependency effects are unknown. They may affect object contents, globals, indirectly referenced memory, ownership, or the validity of later accesses. No preservation guarantee is inferred.

The function itself shows no allocation, release, ownership transfer, exception handling, retry loop, or cleanup operation. The returned word is not established to be a pointer or an owned resource.

The addresses exposed to adjustment identify storage live during this invocation. Their retention is not ruled out; their continued validity after the invocation is not established. Possible use during initialization must not be confused with permission to use them after return.

Invalid accesses and abnormal dependency termination have no established conversion into ordinary zero results.

## Dependencies

| Opaque ID | Visible interaction | Unknown behavior |
|---|---|---|
| F-3b037d560480 | Called once without explicit arguments; zero rejects lookup, nonzero permits the selector check | Meaning of the gate, global accesses, ownership effects, and failure behavior |
| F-3dfc1640191e | Receives the object address and addresses of writable selector and control storage | Adjustment rules; pointer retention or sharing; object, global, allocation, and ownership effects |
| F-60950fd9e511 | Receives the object address after the first table-pointer read returns zero; its result is ignored | Whether initialization occurs; which storage changes; whether retained selector/control addresses are accessible; failure and ownership effects |

No dependency implementation is supplied. In particular, absence of an explicit selector argument to F-60950fd9e511 does not prove that it cannot change the selector through retained state.

## Acceptance examples

These are conditional criteria derived from static evidence, not runtime results. Unless an example states otherwise, all reached memory is readable, dependencies return normally, and selector and relevant memory contents remain stable after adjustment.

1. **Gate rejection.** If F-3b037d560480 returns zero, the result is zero. Adjustment and initialization are not called, and this function performs no object-field read.

2. **Initially zero selector.** If the gate returns nonzero but the initial selector is zero, the result is zero without adjustment, irrespective of position or control.

3. **Zero object address.** With a nonzero gate result and nonzero initial selector, a zero object address supplies count zero. Every unsigned position fails the bound without an object dereference by this function.

4. **Exclusive bound.** With count 4, position 4 returns zero before adjustment. Position 3 passes the bound.

5. **Stable direct lookup.** With count 4, position 1, selector 2 after adjustment, and a zero adjusted control low byte, both table-pointer reads use object offset 0x3b4 if the selector remains 2. If the first pointer is nonzero, initialization is skipped. A stable table pointer yields a word read at table offset 0x14.

6. **Reversed lookup.** Under the preceding count and position, a nonzero adjusted control low byte produces effective position 2 and table-word offset 0x18.

7. **High control bits alone.** Adjusted control 0x00000100 leaves position 1 unchanged. Adjusted control 0x00000101 requests reversal.

8. **Adjustment to selector zero.** If adjustment changes a nonzero initial selector to zero, the first table-pointer read uses object offset 0x3ac. There is no repeated nonzero-selector check.

9. **Initialization with a stable selector.** If the first pointer read for selector 2 is zero and initialization leaves the selector at 2, the second availability read again uses offset 0x3b4. If nonzero, a further selector evaluation and pointer read supply the final base.

10. **Initialization changes selection.** Suppose adjustment exposes selector storage that initialization can later modify. The first read uses selector 2’s slot at offset 0x3b4 and finds zero. Initialization changes the selector to 3. The second availability read then uses offset 0x3b8. If it is nonzero and the selector remains 3, the final-base read also uses offset 0x3b8. Selector 2’s slot need not become nonzero.

11. **Changed selection is unavailable.** In the preceding changed-selector scenario, if selector 3’s second availability read is zero, the function returns zero—even if selector 2’s slot has become nonzero. There is no fallback to selector 2.

12. **Saved count and completed reversal.** If the saved count is 6 and original position is 1, reversal produces position 4 even if adjustment changes the metadata count. Later initialization does not trigger a new count read or reversal decision.

13. **Final reread differs.** If an availability read is nonzero but the final-base read observes a different pointer, the final word access uses the latter. A zero final-base value is not caught by another explicit null check.

14. **Zero table element.** A successfully loaded zero word is returned as zero and cannot be distinguished from rejection through the return value alone.

## Uncertainties

**Inferred intent:** The count appears to describe a table length, the selector appears to choose a table, and the low control byte appears to request reverse indexing. Their domain meanings are not established.

**documented_only — dependency behavior:** Implementations, global-state effects, pointer retention, ownership changes, and abnormal-return behavior are unavailable. A selector change during initialization is permitted by the visible interface and unknown retention behavior, not proven to occur.

**documented_only — selector range and address arithmetic:** No valid selector range is established. Four-byte selector scaling and object-relative address additions may encounter limits not specified by the supplied representation. No portable overflow or wraparound contract is asserted.

**documented_only — memory stability:** There is no established locking, atomicity, volatile-access contract, or exclusion of concurrent modification. Separate selector evaluations and pointer reads are visible; the conditions under which their values can change are not fully specified.

**documented_only — storage extent and failures:** The saved count does not independently prove table capacity. Behavior for inaccessible storage, invalidated allocations, or dependencies that do not return normally remains unspecified.

## Evidence

Established evidence consists of the supplied static representation of F-f8ec0897aceb and its opaque dependency mapping.

It establishes the gate order, selector rejection, unsigned 16-bit count acquisition, exclusive unsigned position bound, exposed writable selector and control storage, low-byte reversal decision, conditional single initialization call, separate selector evaluations for table-pointer reads, and final 32-bit word access.

It does not establish dependency implementations, selector preservation during initialization, ownership rules, platform memory semantics, or runtime outcomes.

No runtime tests have been performed.
