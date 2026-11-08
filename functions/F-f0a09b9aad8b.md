<!-- Trusted provenance: F-f0a09b9aad8b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f0a09b9aad8b

## Purpose

F-f0a09b9aad8b implements an ordered dispatch procedure for an object and an event-like record. It attempts specialized handling, a shared handling stage, an object-local keyed handler, and a general fallback. If those stages decline the record, certain record types can reach a final virtual callback gated by an object-resident byte.

“Object,” “record,” “handler,” and “handled” describe inferred roles. The established behavior is the sequence of memory accesses, calls, zero/nonzero decisions, and return values below.

Most successful stages produce the normalized return value **1**. Two late stages preserve their callees’ return values instead. Consequently, the function is not strictly a Boolean-returning interface.

## Interface

### Inputs

The function takes two address-valued arguments:

- **Object address:** a pointer to mutable storage containing a virtual-dispatch table pointer, two embedded lookup structures, and a byte-sized gate.
- **Record address:** a 32-bit address-valued argument identifying a readable record. The same address is passed to downstream handlers.

The supplied representation uses 32-bit integers, address values, and four-byte words. All offsets below are byte offsets. The contract is specific to that representation; it does not establish an equivalent layout for a wider-pointer target.

### Relevant object layout

| Offset | Width or role | Use |
|---|---|---|
| `+0x00` | 32-bit address | Virtual-dispatch table address |
| `+0x74` | Embedded structure, extent unknown | Receiver supplied to the record-type lookup |
| `+0x9C` | Embedded structure, extent unknown | Receiver supplied to the specialized lookup |
| `+0xF9` | 1 byte | Late-callback gate; explicitly cleared after that callback returns |

The two embedded lookup receivers are addresses within the object, not pointers loaded from those offsets.

Relevant virtual-table slots are:

| Table offset | Role |
|---|---|
| `+0x200` | Early callback for initial record type `0x0F` |
| `+0xD4` | Final gated callback |

Each virtual call receives the object address followed by the record address.

### Relevant record and payload layout

| Location | Width | Use |
|---|---|---|
| Record `+0x04` | 32 bits | Record type and later lookup key |
| Record `+0x08` | 32-bit address | Payload address, read only for initial type `0x16` |
| Payload `+0x04` | 32 bits | Specialized lookup key |
| Payload `+0x08` | 32 bits | First eligibility field |
| Payload `+0x0C` | 32 bits | Second eligibility field |

The payload fields are examined only on the initial type-`0x16` branch. Neither zero eligibility field establishes that the payload pointer itself may be null.

### Output

The function returns a 32-bit integer:

- **1** when an early callback, a dispatched keyed handler, or the shared handling stage returns any nonzero value.
- The exact nonzero result of F-6e38c8351d29 when that dependency returns nonzero.
- **0** when all preceding stages decline and the final callback is unavailable because of the current record type or a zero gate byte.
- The exact result of the final virtual callback, including zero, when that callback is reached.

Negative values are nonzero for all dispatch decisions. Where normalization applies, a negative callee result becomes **1**.

### Preconditions and ownership

No null checks, bounds checks, table validation, or address-overflow checks are established. Every access reached by the selected path must refer to suitable storage, and every invoked table slot must identify a callable target.

The function explicitly writes only its local lookup-result storage and, conditionally, the object byte at `+0xF9`. This does not imply that dependencies leave other storage unchanged.

No allocation, release, reference-count operation, or ownership transfer is directly established. The object, record, payload, table, lookup structures, and resolved handler values are used without explicit retention. Their validity across calls is a caller/dependency requirement not resolved by this evidence.

## Behavior

### 1. Initialize lookup-result storage and read the initial type

A local 32-bit lookup-result slot is initialized to zero once, before the first record-type read.

The function then reads the 32-bit field at record offset `+0x04`. This initial value selects the specialized handling branch.

The local slot is shared by both possible lookup operations. It is not explicitly reset before the later lookup.

### 2. Attempt specialized handling

There are two mutually exclusive specialized branches.

#### Initial type `0x0F`

The function loads the object’s virtual-table address and calls the target at table offset `+0x200`, passing the object and record addresses.

- A nonzero callback result causes an immediate return of **1**.
- A zero result continues to the shared handling stage.

A zero result does not undo any callback side effects.

#### Initial type `0x16`

The function reads the payload address from record offset `+0x08`, then reads the 32-bit field at payload offset `+0x08`.

- If this field is nonzero, specialized lookup is skipped.
- If it is zero, the function reads the 32-bit field at payload offset `+0x0C`.
- Specialized lookup occurs only if this second field is also zero.

Thus, the read at payload `+0x0C` is conditional. A nonzero first eligibility field prevents that read.

When both fields are zero, the function reads the 32-bit key at payload offset `+0x04` and calls F-d6f1e679078e with:

1. The address of the object’s embedded structure at `+0x9C`.
2. The payload key.
3. The address of the local lookup-result slot.

The lookup’s return status and the slot value are separate conditions:

- A zero status prevents handler invocation, regardless of the slot’s contents.
- A nonzero status permits use of the slot.
- A zero slot value still prevents handler invocation.
- Only a nonzero status paired with a nonzero slot value leads to F-f111da7737c5.

F-f111da7737c5 receives the resolved slot value and the record address.

- A nonzero result causes an immediate return of **1**.
- A zero result continues to the shared handling stage.

#### Other initial types

No specialized callback, payload access, or specialized lookup occurs. Processing continues directly to the shared handling stage.

### 3. Attempt shared handling

After specialized handling is skipped or returns zero, the function calls F-0ae1825b406f with no explicit arguments.

Its 32-bit result is passed unchanged as the first argument to F-ae22d23cc97e. The record address is the second argument.

No local validation of the intermediate result is established. In particular, a zero result from F-0ae1825b406f does not suppress the call to F-ae22d23cc97e.

- If F-ae22d23cc97e returns nonzero, the function returns **1**.
- Otherwise, processing continues to the record-type lookup.

No guarantee is established that these calls preserve globals, object state, record contents, or previously resolved values.

### 4. Attempt the record-type lookup

The function rereads the 32-bit field at record offset `+0x04`. It does not reuse the initial type value.

That newly read value is passed as the key to F-d6f1e679078e, with the embedded structure at object offset `+0x74` as receiver and the same local lookup-result slot as output storage.

The lookup and invocation conditions match those of the specialized lookup:

- Zero lookup status: skip handler invocation.
- Nonzero lookup status but zero slot value: skip handler invocation.
- Nonzero lookup status and nonzero slot value: call F-f111da7737c5 with that value and the record address.

If the handler returns nonzero, the function returns **1**. If lookup does not yield an invocable value, or the handler returns zero, processing continues to the general fallback.

Because the output slot is not reset, a lookup that reports success without writing it exposes the slot’s prior contents to this decision. Whether F-d6f1e679078e permits such behavior is unknown. The wrapper itself does not enforce a fresh output value.

### 5. Attempt the general fallback

The function calls F-6e38c8351d29 with the object and record addresses.

- Any nonzero result is returned exactly, without normalization.
- A zero result permits consideration of the final callback.

This stage precedes both the final type restriction and the gate-byte read.

### 6. Consider the gated final callback

After F-6e38c8351d29 returns zero, the function rereads the 32-bit record type at offset `+0x04`.

Only these current values are eligible:

- `0x06`
- `0x07`
- `0x08`
- `0x0D`

For any other value, the function returns **0** without reading the gate byte.

For an eligible type, the function reads the byte at object offset `+0xF9`.

- If the byte is zero, the function returns **0**.
- If the byte is nonzero, the function loads the object’s virtual-table address and calls its target at offset `+0xD4`.

After that callback returns, the function writes zero to the byte at object offset `+0xF9`, then returns the callback’s exact 32-bit result.

The clear occurs **after** the callback, not before it. It occurs even when the callback returns zero. If the callback changes the gate byte, the subsequent explicit zero write overwrites that change.

## Side effects

### Explicit memory accesses

Subject to branch selection, the significant access order is:

1. Initialize the local lookup-result slot to zero.
2. Read the initial record type.
3. Perform the selected early virtual call or payload eligibility reads and specialized lookup.
4. Call the shared handling dependencies.
5. Reread the record type for the object-local lookup.
6. Call the general fallback if the preceding lookup path does not handle the record.
7. Reread the record type for final-callback eligibility.
8. Read the gate byte only for an eligible type.
9. Invoke the final virtual callback only for a nonzero gate.
10. Clear the gate byte after that callback returns.

An earlier successful stage prevents all later stages and their accesses.

The three possible record-type reads are distinct. Mutations by intervening dependencies can therefore change the key used by the second lookup or the eligibility decision for the final callback.

The final virtual-table address is also loaded when the final callback is reached; it is not established as a cached copy of the table used by an earlier callback.

### Gate state and reentrancy

This function does not explicitly clear the gate before invoking the final callback. Consequently, the supplied evidence does not establish a recursion guard.

If the callback synchronously reenters this function, the nested call may observe a nonzero gate unless some other participant has changed it. This is a consequence of the write ordering, not a claim that reentry occurs.

The post-callback clear is established only when control returns normally from the callback. Behavior after a fault, nonlocal exit, or other failure to return is not specified.

### Errors

There is no distinct local error channel. Zero is used as a decline/not-handled result, while nonzero results control successful dispatch.

No recovery is shown for invalid memory, invalid virtual targets, or dependency failures that do not return normally. A lookup’s zero status causes fallback progression; the evidence does not establish whether that status means “not found,” “error,” or another condition.

## Dependencies

| Opaque dependency | Established interaction | Unknowns |
|---|---|---|
| F-d6f1e679078e | Called with an embedded receiver address, a 32-bit key, and writable 32-bit output storage; return value tested for zero | Lookup semantics, output-write guarantees, ownership, and other side effects |
| F-f111da7737c5 | Called with a nonzero resolved value and the record address; result tested for zero | Nature of the resolved value, invocation mechanism, mutations, and lifetime rules |
| F-0ae1825b406f | Called without explicit arguments; 32-bit result forwarded to the next dependency | Meaning, validity constraints, ownership, and global-state effects |
| F-ae22d23cc97e | Receives that forwarded result and the record address; nonzero means an immediate normalized return | Internal handling rules and side effects |
| F-6e38c8351d29 | Receives object and record; nonzero result returned unchanged | Interpretation of returned values and mutation behavior |

The two indirect targets have no supplied opaque function IDs. They are identified here only by virtual-table slot offsets. Their implementations and ownership behavior are unknown.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests. Unless stated otherwise, dependencies return normally and preserve the relevant input fields.

1. **Early callback normalization.**  
   With initial type `0x0F`, if the `+0x200` callback returns `-3`, the function returns **1**. No shared handling call or later lookup occurs.

2. **Conditional payload read.**  
   With initial type `0x16` and a nonzero field at payload `+0x08`, the field at payload `+0x0C` is not read, and the specialized lookup is not called. Shared handling is attempted.

3. **Eligible specialized lookup.**  
   With initial type `0x16` and both eligibility fields zero, the specialized lookup receives the payload’s `+0x04` key and the object’s `+0x9C` receiver address. If it reports nonzero status, writes a nonzero result, and the resulting handler returns `12`, the function returns **1**.

4. **Lookup status takes precedence over output.**  
   If a lookup writes a nonzero output but returns zero, that lookup does not cause F-f111da7737c5 to be invoked. Processing continues to the next stage.

5. **Shared handling short-circuit.**  
   If specialized handling declines and F-ae22d23cc97e returns any nonzero value, the function returns **1**, without the `+0x74` lookup.

6. **Mutation affects the later lookup key.**  
   If an early callback returns zero after changing the record type, and shared handling also declines, the `+0x74` lookup uses the changed type, not the initial type.

7. **General fallback preserves its result.**  
   If the earlier stages decline and F-6e38c8351d29 returns `-7`, the function returns **-7**. It does not perform final type eligibility or gate-byte checks.

8. **Ineligible final type avoids gate access.**  
   If the general fallback returns zero and the current type is `0x09`, the function returns **0** without reading object `+0xF9`.

9. **Eligible type with a zero gate.**  
   If the current type is `0x07` and the gate byte is zero, the final virtual callback is not called and the function returns **0**.

10. **Final callback result and clear.**  
    If the current type is `0x0D`, the gate is nonzero, and the final callback returns `-2`, the function clears the gate and returns **-2**. If that callback instead returns zero, the gate is still cleared and the return is **0**.

11. **Persistent lookup-output storage.**  
    If the specialized lookup leaves a nonzero value in the shared output slot, its handler declines, shared handling declines, and the later lookup reports success without modifying that slot, the later handler invocation uses the retained value. This example describes the wrapper’s behavior under that dependency behavior; it does not assert that the dependency supports such a success case.

## Uncertainties

- **documented_only — Semantic identities:** The actual meanings of record types, payload fields, embedded structures, and the gate byte are not established.
- **documented_only — Representation:** The supplied representation supports the stated 32-bit widths and offsets. No wider-address variant, byte order, alignment guarantee, or complete structure extent is established.
- **documented_only — Dependency contracts:** Callee output guarantees, global preservation, allocation, release, ownership, and exceptional behavior are unknown.
- **documented_only — Concurrency:** No synchronization or atomic-access semantics are established. The ordered accesses described here do not imply thread safety.
- **documented_only — Reentrancy:** Callback reentry is not excluded, and the gate is cleared only after the final callback returns.
- **Inferred intent:** The chain appears intended to route records through increasingly general handlers, with a late gated path for selected types. This interpretation does not override the exact ordering or return-value rules.

## Evidence

The supplied static function body establishes the access offsets, conditional reads, dependency call order, short-circuit behavior, lookup-output reuse, repeated record-type reads, gate-clear timing, and distinction between normalized and preserved return values.

The supplied dependency mapping establishes the opaque IDs used above. No dependency implementation, external caller contract, or runtime observation was supplied.

No runtime tests have been performed. The acceptance examples are static, conditional consequences rather than measured results.
