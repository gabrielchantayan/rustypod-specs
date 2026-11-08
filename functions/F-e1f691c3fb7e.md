<!-- Trusted provenance: F-e1f691c3fb7e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e1f691c3fb7e

## Purpose

F-e1f691c3fb7e conditionally scans positions associated with a caller-supplied object. A query dependency determines which positions receive further processing.

For each selected position, an object dispatch supplies a 32-bit value:

- A nonzero value is submitted through a common handle. In mode 3, the function also performs two 16-bit record writes and updates an object-held count.
- A zero value permanently clears two working arguments and initiates an alternate lookup. A successful lookup supplies an object whose dispatch result becomes the handle for submitting the position.

After scanning, the current selection-structure field controls whether execution enters a statically marked nonreturning path or performs a final object dispatch. A second conditional nonreturning path follows that dispatch.

“Selection,” “record,” “lookup,” and “handle” describe observed usage, not established application semantics. In particular, record writes are indirect memory writes: no supplied precondition establishes that their destinations are disjoint from the object or other relevant state.

## Interface

### Arguments

The function has four visible arguments:

| Argument | Observed representation and use |
|---|---|
| Object address | Address of writable storage with 32-bit fields and a dispatch-table address |
| Mode | 32-bit value; compared with 3, passed to final dispatch, and initially used as the upper return half |
| Auxiliary argument | Opaque 32-bit value; passed to final dispatch unless a zero-result iteration clears it |
| Trailing argument | Opaque 32-bit value passed unchanged to final dispatch |

The observed object layout is:

| Byte offset | Use |
|---|---|
| `0x00` | Dispatch-table address |
| `0x48` | 32-bit argument for alternate lookup |
| `0x74` | 32-bit address used for record writes |
| `0x78` | 32-bit record count used in address calculations and subsequently incremented |
| `0x8C` | Selection-structure address, or zero |

The selection-like structure has these observed fields:

| Byte offset | Use |
|---|---|
| `0x00` | Unsigned 32-bit scan bound |
| `0x04` | 32-bit quantity used to form the mode-3 storage request |

The indirect operations are:

- Object dispatch-table offset `0x08`: returns a 32-bit value; receives the object address as its sole explicit argument.
- Object dispatch-table offset `0x18`: final dispatch; receives seven explicit arguments described below.
- Alternate-result dispatch-table offset `0x28`: returns a value subsequently used as a handle; no explicit arguments are shown.

The representation uses 32-bit address-bearing fields. Concrete types, calling conventions, and implicit receiver treatment are unresolved.

### Result

The visible result is a 64-bit composition:

- **Upper 32 bits:** the supplied mode, unless any selected-position object dispatch returns zero, in which case this half is zero.
- **Lower 32 bits:** zero on the entry bypass; otherwise the final-dispatch result on a path that reaches normal return.

This establishes the returned bit composition, not whether the original interface conceptually exposed an integer or a register pair.

### Preconditions and memory relationships

Every executed dereference and indirect call requires a valid address and suitable readable, writable, or executable storage. The function does not validate these requirements.

In particular:

- Nonzero selection addresses are not validated.
- The reloaded selection address used to capture the bound is dereferenced without another null check.
- Record addresses and capacity are not checked.
- Dispatch tables and targets are not checked.
- No allocation-failure branch protects later record writes.
- Dependencies must leave subsequently accessed state valid for execution to proceed safely; preservation is not established.

There is **no established non-aliasing precondition** for record storage. Record destinations may overlap object fields, selection storage, dispatch-related storage, or other accessible state. Such overlap can change later reads and control flow. A conventional, disjoint record array is an inferred intended arrangement, not a guaranteed input condition.

## Behavior

### Entry bypass

The function starts with a zero lower-result working value and an upper-result working value equal to the mode.

It reads object offset `0x8C`. If that field is zero, it returns immediately with the mode in the upper half and zero in the lower half. This path makes no dependency calls and performs no object writes, including no mode-3 table reset.

### Mode-3 preparation

For a nonzero entry selection field and mode equal to 3:

1. The function reads the 32-bit quantity at offset `0x04` of the referenced selection structure.
2. It forms a storage-request operand by shifting that quantity left by two.
3. F-638beded7699 receives that operand and `0x2D`.
4. The returned value is stored at object offset `0x74`.
5. Zero is stored at object offset `0x78`.

The visible calculation is 32-bit. No widened allocation-size calculation, overflow rejection, or signed-overflow policy is established. A mathematical size exceeding the operand range must not be treated as a guaranteed full-size request.

The address assignment replaces any previous value without an explicit preceding release. The count reset follows the address assignment. Neither operation proves that the returned storage is separate from the object.

Other modes skip this preparation.

### Common setup and captured bound

The function calls F-8157a180d901 with `0x10`. Its result, an opaque static-data operand, and `0x32` are passed to F-2c0adabb59c0. The latter result is retained as the common handle.

After these calls, the function reloads object offset `0x8C` and dereferences that address to obtain an unsigned 32-bit scan bound. There is no intervening null check. Setup code that clears or invalidates the field can therefore make this read invalid.

The bound is captured once. Subsequent changes to the source bound do not alter the scheduled range.

### Position queries

The loop visits positions from zero up to, but excluding, the captured unsigned bound, in ascending order. A zero bound produces no queries. Even the maximum 32-bit bound does not require an iteration at that bound or a wraparound iteration.

For each position, F-88658862e573 receives:

- The selection address currently read from object offset `0x8C`;
- The unsigned position divided by 32;
- The position’s low five bits.

A zero query result skips selected-position processing. Any nonzero result enters it.

The function does not directly interpret a bitmap. Word and bit indices are established argument calculations; the query’s internal representation is unknown.

The selection address is reloaded for every query, unlike the bound. Callee mutations or preceding indirect record writes can therefore affect subsequent query arguments. A zero address is not checked before being passed to this dependency; whether the dependency accepts it is unresolved.

### Selected position: nonzero object result

For a selected position, the function reloads the object’s dispatch table and invokes its entry at `0x08`.

If the returned 32-bit value is nonzero, F-008d77a63c85 receives the common handle, that full value, and `1`. Its return value is ignored.

If the original mode is 3, record processing follows that call:

- A 16-bit store writes the low 16 bits of the position at an address calculated from the current table-address field plus four times the current count.
- A second 16-bit store writes the low 16 bits of the object result at an address calculated from the then-current table-address field plus four times the then-current count, plus two bytes.
- The current 32-bit count is then read, incremented by one, and written back.

The table and count reads occur after the submission dependency returns. Their pre-call values are not preserved for these operations.

The two destination expressions are separately evaluated. No disjointness guarantee permits treating the sequence unconditionally as one append to a stable four-byte slot:

- The first store can overlap the table-address or count field and thereby change the second store’s destination.
- Either store can overlap the count field and change the value subsequently incremented.
- Either store can overlap other object fields, including the selection address.
- The stores can affect other aliased storage that later calls or reads use.

When the table and count fields remain unchanged during the two stores, and the addressed slot is valid, the conventional result is a four-byte record containing the position halfword followed by the value halfword, with a subsequent count increment. Byte order within each halfword is target-dependent. The stores are separate, not a demonstrated atomic four-byte update.

No capacity, address-overflow, or count-overflow check is present. The source representation does not establish a safe mathematical result for overflowing signed or address calculations.

Recording depends on the original mode. Clearing the upper-result working value during an earlier zero-result iteration does not disable later mode-3 stores.

### Selected position: zero object result

If the dispatch at offset `0x08` returns zero, the function first clears:

- The upper-result working value;
- The auxiliary argument retained for final dispatch.

It then calls F-b5164379c09e with:

1. The position’s low 16 bits;
2. The current object field at `0x48`;
3. The first 32-bit value read through a global-data reference;
4. The next 32-bit value through that reference;
5. Zero;
6. Zero.

These operands establish the sources of the arguments, not a reliable left-to-right order for their memory reads.

A zero lookup result ends processing for that position. A nonzero result is treated as an object address: its dispatch-table entry at `0x28` is invoked with no explicit arguments shown. F-008d77a63c85 then receives that dispatch result, the full 32-bit position, and `1`.

This path performs no record-store sequence. The cleared working values remain zero through later iterations and final dispatch; lookup failure does not restore them. The lookup receives only 16 position bits, while alternate submission receives all 32.

### Post-scan paths

After the loop, the function rereads object offset `0x8C`.

If nonzero, it calls F-141ce3c91b04 with no explicit arguments shown, followed by F-feee9156a257. The supplied static representation marks the latter nonreturning. Under that annotation, this path does not reach the subsequent field clear or final dispatch.

If the field is zero, the function writes zero to it and invokes the object’s dispatch-table entry at `0x18` with:

1. The object address;
2. The common handle;
3. The original mode;
4. A separately read value of the dispatch-table entry at `0x18`;
5. The current upper-result working value;
6. The current auxiliary argument;
7. The original trailing argument.

The invocation target and the dispatch-entry argument are separate reads. Their ordering relative to other argument reads is not established.

The dispatch result becomes the lower return half. If the retained common handle is nonzero, F-141ce3c91b04 receives that handle, followed by F-feee9156a257. Under the nonreturn annotation, no normal return follows.

Normal post-scan return therefore requires a zero selection field at the post-scan check and a zero common handle at the later handle check.

Reaching final dispatch after a nonzero entry selection field does **not** necessarily require a callee to clear that field. The function’s own indirect record writes can clear it through aliasing. Only under an additional, unestablished assumption that those writes cannot affect the selection field or other relevant state would its pre-dispatch transition to zero require effects outside the function’s explicit assignments.

## Side effects

Established explicit writes include:

- Replacement of the table-address field and resetting its count in mode 3;
- The two ordered 16-bit indirect writes for each recorded nonzero result;
- The subsequent count update;
- The zero write to the selection field on the final-dispatch path.

The indirect writes are not confined by evidence to dedicated record storage. They may modify the object or other aliased storage. Their effects can include changes to subsequent destinations, query addresses, dispatch state, and post-scan branching. With aliasing, the resulting count need not equal the preceding count plus one in the ordinary append sense, because a halfword store may first alter it.

All dependencies and indirect dispatches may have additional effects. No preservation of globals, object fields, or pointed-to storage is established.

Ownership remains unresolved:

- Retaining the allocation-like result does not establish ownership transfer.
- No explicit release of the record storage appears.
- No independent release of the first setup result appears.
- Alternate lookup results are not explicitly released.
- F-141ce3c91b04 cannot be classified as deallocation from its use here.
- The common handle’s lifetime rules are unknown.

No rollback is shown. Earlier calls and writes precede any later invalid access or nonreturning exit.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-638beded7699 | Receives the shifted size operand and `0x2D`; result is stored as the record address. |
| F-8157a180d901 | Receives `0x10`; result is passed to common-handle setup. |
| F-2c0adabb59c0 | Receives the preceding result, an opaque static-data operand, and `0x32`; returns the common handle. |
| F-88658862e573 | Receives the current selection address, word index, and bit index; zero skips processing. |
| F-008d77a63c85 | Receives a handle, a full 32-bit value or position, and `1`; result is ignored. |
| F-b5164379c09e | Receives the truncated position, object context, two global values, and two zeros; returns an optional object address. |
| F-141ce3c91b04 | Precedes each explicit nonreturning call; visible argument lists differ between the two sites. Signature and effect are unresolved. |
| F-feee9156a257 | Invoked on two conditional exit paths and statically marked nonreturning. |

The indirect dependencies have no supplied opaque IDs and are distinguished by dispatch-table offsets `0x08`, `0x18`, and `0x28`. Their bodies, memory effects, and ownership contracts are unavailable.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Entry bypass.**  
   With a zero entry selection field and mode 3, the result has upper half 3 and lower half zero. No allocation, reset, query, or dispatch occurs.

2. **Empty scan with a retained selection address.**  
   With a nonzero entry field and captured bound zero, setup still occurs. No position queries occur. If the field remains nonzero, F-141ce3c91b04 and then F-feee9156a257 are invoked instead of final dispatch.

3. **Ordinary disjoint record.**  
   In mode 3, at selected position `0x10002`, let the object result be `0x12345678`. With valid, sufficiently large storage and stable table/count fields, submission receives the full result. The record receives halfwords `0x0002` and `0x5678`, followed by a count increment.

4. **Direct clearing through record aliasing.**  
   Let mode be 3, the captured bound be one, position zero be selected, and the current record destination be object address plus `0x8C`, with count zero. Let the object result be `0x00010000`, and let relevant calls leave the selection field unchanged. The first store writes zero to the first half of that field; the second writes zero to the other half. The count update affects offset `0x78`, not `0x8C`. Thus the post-scan selection check observes zero without a callee clearing it. With a valid final-dispatch target, final dispatch is reached. A zero common handle then permits normal return.

5. **Record metadata overlap.**  
   If the first record store overlaps and changes the count or table-address field, the second destination must reflect that changed field. A fixed adjacent second halfword is not guaranteed. If the recomputed destination is invalid, no safe completion is established.

6. **Failed alternate lookup.**  
   A selected-position object result of zero clears the upper-result working value and auxiliary argument before lookup. A zero lookup result causes no alternate dispatch or submission and does not restore either cleared value.

7. **Recording after a zero result.**  
   In mode 3, a later selected position with a nonzero result still performs record processing after an earlier zero-result position, provided preceding mutations leave the necessary accesses valid.

8. **Alternate-path width difference.**  
   At position `0x10002`, lookup receives `0x0002`. If lookup succeeds, alternate submission receives `0x10002`.

9. **Conditional completion and nonreturn.**  
   With a zero selection field at the post-scan check, final dispatch occurs. A zero common handle permits return of its result in the lower half. A nonzero common handle instead triggers the subsequent cleanup-like call and statically nonreturning call.

## Uncertainties

The following remain **documented_only**:

- Application meanings of positions, submitted values, mode 3, and setup constants.
- Allocation guarantees, returned capacity, and any intended disjointness from object or global storage.
- Exact behavior of overflowing size, count, and address calculations.
- Concrete calling conventions, dependency signatures, memory preservation, and ownership.
- Target byte order and any stronger atomicity guarantees.
- Behavior when aliasing corrupts a later address or dispatch target.
- Whether F-feee9156a257 actually never returns at runtime; reachability statements rely on its static annotation.
- The conceptual interpretation of the 64-bit result representation.
- Error reporting beyond visible zero branches and nonreturning paths.
- Concurrency and reentrancy guarantees.

A disjoint record array and cleanup semantics are plausible intent, not established guarantees.

## Evidence

Established behavior comes from the supplied static representation: field offsets, operand widths, explicit call arguments, branch conditions, separately expressed record destinations, ordered halfword stores, count update, and return-half composition.

The aliasing possibility follows from those indirect writes and the absence of a supplied disjointness precondition. The selection-field clearing example uses zero-valued halfwords, so that example does not depend on target byte order.

Dependency meanings, allocation behavior, and ownership are not established by dependency names or usage. Nonreturn behavior is a static annotation, not a runtime observation.

No dependency bodies, implementation access, runtime tests, or runtime observations were available.
