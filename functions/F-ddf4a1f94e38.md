<!-- Trusted provenance: F-ddf4a1f94e38; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ddf4a1f94e38

## Purpose

F-ddf4a1f94e38 processes a selector against a memory-mapped record and performs branch-dependent changes to software state and hardware-facing registers. Its visible operations include clearing flags, invoking dependencies, updating record bits, polling status, scanning context slots, and clearing entries in an apparent circular work structure.

The result is not uniformly Boolean. Established results include zero, one, 128, the supplied context address, and dependency-returned values.

“Record,” “recovery,” and “work structure” are descriptive terms. Their subsystem meanings are inferred; the access widths, addresses, conditions, ordering, and result selection below are established by the supplied static representation.

## Interface

### Arguments and result

The function accepts two unsigned 32-bit values:

- **Context address:** interpreted as the address of context storage.
- **Selector:** used both in full and through selected low bits.

The result is an unsigned 32-bit value.

The selector’s low seven bits produce a record index from 0 through 127. Bit 7 chooses the record class:

- **Lower class:** bit 7 clear.
- **Upper class:** bit 7 set.

Bits 8–31 do not affect initial record addressing. They remain significant when the complete selector is compared with a byte or passed to a dependency. The selector is not normalized to eight bits.

### Storage and access widths

Observed data accesses are one byte or four bytes. Comparison bytes and the scan-limit byte are interpreted as unsigned. The work cursor is interpreted as signed 32-bit.

| Object | Relevant layout |
|---|---|
| Context | Byte at `0x1A`: scan limit. Byte at `0x2B`: recovery branch control. Four-byte slots at `0x5C + 4 × index`. Four-byte fields at `0xA0` and `0xA4`. |
| Flag storage | Byte indexed by the record index for lower-class handling, or index plus 9 for upper-class handling. |
| Control state | Bytes at offsets 5 and 9; signed 32-bit cursor at `0x20`. |
| Recovery state | Byte at offset 4; four-byte selector field at `0x18`. |
| Work storage | First-array origin at offset zero, second-array origin at `0xB0`, and third-array origin at `0x160`; entries are four bytes. Each origin is obtained through a separate global-pointer read. |

These roles do not establish disjoint storage. For example, context slots 17 and 18 coincide with offsets `0xA0` and `0xA4`.

Let the register base be `0x38400000`:

| Register role | Address |
|---|---|
| Lower-class record | Base plus `0xB00 + 0x20 × index` |
| Upper-class record | Base plus `0x900 + 0x20 × index` |
| Polling status | Base plus `0x14` |
| Control word | Base plus `0x804` |
| Command word | Base plus `0x810` |
| Special status word | Base plus `0xB10` |

All listed register accesses are four bytes. The address formulas do not establish nonoverlapping banks or a hardware-supported record count. No smaller-range validation is visible.

### Preconditions and errors

There is no visible validation of addresses, capacities, hardware availability, or cursor invariants. Defined execution requires valid storage for every access reached on the selected path, including addresses obtained from separately observed global pointers.

In particular:

- Flags must accommodate the reached index or index-plus-9 access.
- Context storage must accommodate the selected slot and any scanned slots.
- The stored cursor must be safe when used directly to address the third work array.
- Register accesses must be valid for the target platform.
- Dependencies must receive inputs satisfying their own, unavailable contracts.

No explicit error return, exception conversion, rollback, or recovery from invalid memory is established. Dependency results propagated by this function have no supplied error interpretation.

## Behavior

### Entry observations and record selection

The control-state global pointer is captured first. The flag-storage global pointer is captured second. The function then reads the selected record word.

Bit 31 of that captured record determines whether it is treated as active. The initial branch decisions and the lower-class dependency argument use this captured word, not an automatic reread.

These entry pointer captures are not universal pointer snapshots. Some later accesses reload their corresponding globals, as specified below.

### Lower class, inactive record

When selector bit 7 and record bit 31 are both clear, the function reloads the flag-storage global, clears its byte at the record index, and returns zero.

The flag write does not use the entry-captured flag pointer. No dependency call or explicit context write occurs.

### Lower class, active record at index zero

For an active lower-class record whose index is zero:

1. Reload the flag-storage global and clear byte zero through that newly observed pointer.
2. Write four-byte zero to context offset `0xA0`.
3. Write four-byte zero to context offset `0xA4`.
4. Return zero.

The two context writes are separate and ordered. This branch calls none of the listed dependencies.

### Lower class, active nonzero index

The visible operation order is:

1. Call F-64f588d0d937 with the context address and integer 1.
2. Call F-181d23e9dd61 with the context address, complete selector, and initial record snapshot.
3. Clear the indexed flag through the entry-captured flag pointer.
4. Read register offset `0x804`, OR its value with `0x400`, and write the result back.
5. Call U1 without visible arguments and retain its result as a token.
6. Poll as specified below.
7. Return one.

The first two dependency results are ignored. Setting bit 10 at `0x804` is a read-modify-write; an atomic bit-set operation is not established.

#### Exact polling reads

Every polling pass performs two unconditional reads of register offset `0x14`, in this order:

- The **first** read supplies the bit-7 condition for that pass.
- The **second** read is retained provisionally.

If bit 7 of the first read was set, the function subsequently reads offset `0x804`, replacing the provisional value with that read’s result.

The continuation decision is then:

- First status read had bit 7 clear: stop, without reading `0x804` or calling U2.
- First status read had bit 7 set, but the subsequent `0x804` read had bit 3 clear: stop without calling U2.
- Both conditions hold: call U2 with the U1 token and integer 100. Repeat only when U2 returns zero.

Thus the second `0x14` read always occurs, although its value does not determine continuation. It must not be omitted: a read may have side effects or observe a different hardware value.

There is at least one polling pass after U1 returns. No independent iteration bound is established. A nonzero U2 result terminates polling, but the function still returns one; there is no separate timeout result.

### Upper class, inactive record

For an upper-class record with bit 31 clear, the function clears flag byte index plus 9 through the entry-captured flag pointer and returns 128.

No dependency is called. The return is the isolated class bit, not the complete selector.

### Upper class, active record taking recovery

Recovery is selected when the context byte at `0x2B` is zero and the initial record has no bits set under mask `0x03C00000`, covering bits 22–25. The context byte is read before that mask condition is evaluated.

The function calls F-fec396f49ccf with the context address and 1, then:

- Clears the selected context slot at `0x5C + 4 × index`.
- Clears flag byte index plus 9 through the entry-captured flag pointer.
- Scans context slots beginning with index zero.
- Returns the supplied context address.

The dependency result is ignored.

#### Scan bounds and pointer observations

At **every scan-condition evaluation**, including the final failing evaluation, the function:

1. Loads the recovery-state global pointer.
2. Reads the unsigned context scan-limit byte at `0x1A`.
3. Tests whether the current scan index is less than or equal to that byte.

The limit is inclusive and is reread, not captured once. A limit of zero allows slot zero to be examined. A limit of 255 allows indices zero through 255 if it remains unchanged.

The recovery pointer captured at a successful condition evaluation belongs to that iteration. Loading it is observable even if the corresponding slot is zero or the condition fails.

For each admitted index, the function reads its four-byte context slot. A zero slot causes no corresponding record access. A nonzero slot causes a fresh read of the upper-bank record at that scan index. The scan index is not reduced to seven bits.

#### Nonzero-slot processing

If the scanned record has both bits 18 and 19 set, the ordered effects are:

1. Reload the recovery-state global pointer and clear byte offset 4 through this newly observed pointer.
2. Write the scan index ORed with `0x80` to offset `0x18` through the pointer captured at the scan-condition evaluation.
3. Write `0x39` to register offset `0x810`.
4. Write the captured record value with bit 26 set back to the scanned record.
5. Clear the corresponding context slot.

The two recovery-state writes can therefore target different objects. Neither pointer observation may be substituted for the other.

If bits 18 and 19 are not both set, the record is ORed with one of two masks:

- At scan index zero only, read register offset `0xB10`. If its bits 29–30 are not both set, use `0x88000000`.
- Otherwise use `0x84000000`.

The special status register is not read for nonzero scan indices, or for records taking the both-bits-set case. Its bit 31 does not affect this mask choice.

After selecting the mask, the modified record is written before the context slot is cleared. All record modifications use the per-slot captured record word rather than rereading immediately before writing.

The originally selected slot was cleared before scanning. It will be skipped if still zero when reached; aliasing or intervening mutation can change that outcome.

### Upper class, other active records

If recovery is not selected, the function reloads the control-state global pointer and reads its byte at offset 9. The complete 32-bit selector is compared against that unsigned byte.

A selector above 255 cannot match, even if its low byte equals the stored byte.

#### Nonmatching selector

The function calls F-5c256682e106 with the complete selector, clears flag byte index plus 9 through the entry-captured flag pointer, and returns the dependency’s result.

#### Matching selector

The function reloads the control-state global pointer again and clears byte offset 5 through that pointer. This is a separate pointer observation from the comparison-byte access.

It then calls F-5c256682e106 with the selector and F-9c06a5856a0f with arguments 1 and 1. Both results are ignored.

After those calls, setup occurs in this exact order:

1. Read the work-storage global pointer for the first-array origin.
2. Read the signed cursor at offset `0x20` through the entry-captured control pointer.
3. Independently read the work-storage global pointer and add `0x160` for the third-array origin.
4. Independently read the work-storage global pointer and add `0xB0` for the second-array origin.

These three array origins need not derive from one pointer value.

#### Work traversal

The locally retained cursor candidate is incremented before each attempted iteration. A negative incremented value ends traversal immediately. A nonnegative value greater than 43 is replaced with zero.

For each candidate admitted to the body:

1. Reread the stored cursor through the entry-captured control pointer.
2. Clear the third-array entry indexed by that stored cursor.
3. Read the first-array entry indexed by the candidate.
4. If that entry is zero, stop without advancing the stored cursor.
5. Otherwise clear the candidate’s first-array entry, clear its second-array entry, and store the candidate as the cursor, in that order.

The third-array clear precedes the first-array read and targets the stored cursor, not necessarily the candidate. The stored cursor receives no range correction before this addressing operation.

After traversal, the function calls F-9c06a5856a0f with arguments 1 and 2, clears flag byte index plus 9 through the entry-captured flag pointer, and returns that final call’s result.

## Side effects

Explicit effects include byte and word writes, register reads and writes, global-pointer observations, and dependency calls.

Pointer provenance is part of the contract:

- Control state is captured before flag storage at entry.
- Inactive lower-class and active lower-class index-zero flag writes reload the flag global.
- Active nonzero lower-class and all upper-class flag writes use the entry-captured flag pointer.
- Upper-class comparison and byte-offset-5 clearing use separate later control-pointer reads.
- Work cursor accesses use the entry-captured control pointer.
- Recovery condition captures and special-case byte-write reloads are distinct observations.
- Work-array origins come from three separately ordered global reads.

No dependency or register read is assumed to preserve globals or pointed-to contents. Equal pointer observations are possible, not guaranteed.

No allocation, release, ownership transfer, locking, memory barrier, or transaction is established by the visible operations. Dependency ownership effects remain unknown. Ordered individual writes do not establish atomic multi-field updates, and no rollback is visible.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-64f588d0d937 | Receives context and 1 before active nonzero lower-class handling; result ignored. |
| F-181d23e9dd61 | Receives context, full selector, and initial record snapshot; result ignored. |
| F-fec396f49ccf | Receives context and 1 before recovery clearing and scanning; result ignored. |
| F-5c256682e106 | Receives the full selector on non-recovery active upper-class paths. Its result is returned only for a mismatch. |
| F-9c06a5856a0f | Receives `(1, 1)` before traversal and `(1, 2)` afterward. Only the latter result is returned. |
| U1 | Called without visible arguments; supplies the token passed to U2. |
| U2 | Receives that token and 100. Zero permits another polling pass; nonzero stops polling. |

U1 and U2 are document-local opaque dependency IDs because published mappings were not supplied. Their apparent timing roles are inferred. Token interpretation, time units, failure meanings, and dependency side effects are unknown.

## Acceptance examples

These are conditional static examples, not executed tests. Unless stated otherwise, they assume valid, noninterfering storage and dependencies that return normally.

1. **Reloaded lower-class flag target:** Suppose entry captures flag pointer A, but the flag global holds B when an inactive lower-class record finishes its initial register read. For selector 3, byte 3 through B is cleared, and the result is zero. The flag write does not target A merely because A was captured.

2. **Active lower-class index zero:** With selector zero and an active record, byte zero through the reloaded flag pointer is cleared, followed by context words `0xA0` and `0xA4`. The result is zero.

3. **Two status reads despite early termination:** If the first polling read of `0x14` has bit 7 clear, a second read of `0x14` still occurs. Even if that second read has bit 7 set, no `0x804` polling read or U2 call follows on that pass. The result is one.

4. **First status read governs continuation:** If the first `0x14` read has bit 7 set and the second has it clear, `0x804` is nevertheless read. If its bit 3 is set, U2 is called. A nonzero U2 result ends polling with result one.

5. **Inactive upper class with high selector bits:** Selector `0x180` selects upper-class index zero. If inactive, flag byte 9 through the captured flag pointer is cleared, and the result is 128.

6. **Recovery terminating observation:** With a stable scan limit of zero and slot zero empty, the recovery global is loaded before checking index zero and again before the failing check for index one. Slot zero is read; slot one is not.

7. **Split recovery targets:** For a nonzero slot at index 2 with record bits 18 and 19 set, suppose the condition captures recovery pointer A and the special-case reload observes B. Byte 4 through B is cleared; word `0x18` through A receives `0x82`. Command `0x39` and the bit-26 record update precede slot clearing.

8. **Full-selector mismatch:** With recovery excluded, selector `0x181` cannot match comparison byte `0x81`. F-5c256682e106 receives `0x181`, and its result is returned after the flag clear.

9. **Independent work origins:** Suppose the three work-global reads observe A, B, and C respectively, with stored cursor 5 and first-array entry 6 at A equal to zero. The traversal clears the word at B plus `0x160 + 4 × 5`, then reads A plus `4 × 6`. It does not update the cursor or clear an entry through C. The final dependency call still occurs.

10. **Circular wrap:** With stored cursor 43, the next candidate becomes zero. Third-array entry 43 is cleared before first-array entry zero is read.

## Uncertainties

The following remain **documented_only**:

- Hardware meanings, supported record counts, and register-read side effects.
- Pointer-object aliasing, byte order, alignment guarantees, and synchronization.
- Dependency ownership, global mutation, reentrancy, and error semantics.
- U1/U2 identities, timing units, and termination guarantees.
- Caller-established cursor and scan-limit invariants.
- Results of invalid memory accesses or asynchronously changing state.
- Precise signed-overflow behavior when incrementing the maximum signed 32-bit cursor.

The unsigned-byte scan limit bounds admitted indices to 0–255 and allows a terminating index of 256. Work-cursor arithmetic has a separate signed-overflow uncertainty. Negative or oversized stored cursors are not made safe merely by normalizing the candidate.

The supplied representation identifies omitted unreachable regions. No behavior is assigned to those omitted regions.

## Evidence

This contract uses only the supplied static representation of F-ddf4a1f94e38 and its dependency-ID mapping.

Established evidence includes access widths, offsets, pointer-read provenance and order, branch predicates, register masks, dependency arguments, explicit write ordering, polling reads, traversal rules, and return selection.

Subsystem terminology and apparent timing intent are inferences. Dependency implementations, caller contracts, and hardware semantics were not supplied. No runtime tests have been performed.
