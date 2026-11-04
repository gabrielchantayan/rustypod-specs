<!-- Trusted provenance: F-d8da6d0658bc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d8da6d0658bc

## Purpose

F-d8da6d0658bc conditionally dispatches a supplied record through methods associated with a supplied receiver. One distinguished record form first receives an opportunity for special handling. If that handler returns a nonzero result, dispatch ends with that exact result. Otherwise, the function selects a general handler using the record’s current discriminator.

The externally visible result is either:

- The exact nonzero 32-bit result returned by the special handler.
- The value `1` after either general handler returns normally.
- The value `0` when neither general-handler discriminator matches.

**Established static evidence:** the implementation performs discriminator checks, indirect calls through fixed receiver-table offsets, and the return selection described below.

**Inferred intent:** the receiver is object-like, its first word identifies a method table, and the supplied record is event-like. These interpretations explain the layout but do not establish a class hierarchy, event taxonomy, or ownership protocol.

## Interface

### Inputs

The function accepts two arguments:

1. **Receiver address.** Identifies storage whose first pointer-sized entry is used to locate a table of callable entries.
2. **Record address.** Identifies storage containing the discriminator and payload words described below.

The supplied representation uses 32-bit integer words and four-byte payload fields. It also represents the record address through an integer-valued parameter. The behavioral contract therefore concerns the represented 32-bit execution model; it does not establish portability to wider-pointer environments.

The record layout relevant to this function is:

| Byte offset from record address | Width | Use |
|---|---:|---|
| `0x00` | Not established | Not directly accessed by this function |
| `0x04` | 32 bits | Discriminator |
| `0x08` | 32 bits | First general-handler payload |
| `0x0C` | 32 bits | Special-selector input and second general-handler payload |
| `0x10` | 32 bits | Third payload for one general-handler branch |

No record size, alignment guarantee, field names, or structure beyond these accesses is established.

The receiver’s table supplies callable entries at byte offsets `0x90`, `0x78`, and `0x5C`. These are byte offsets, not table indices. The receiver’s first entry and the selected table entry are obtained for the applicable indirect call; the contract does not require the table address to remain cached across calls.

### Output

The return value is a 32-bit integer:

- A special-handler result is propagated unchanged when nonzero, including a negative result.
- General-handler dispatch returns `1`, independently of any return value produced by that handler.
- Unrecognized dispatch returns `0`.

Consequently, the output is not restricted to a Boolean range, and `1` does not by itself establish that a general handler accepted or successfully processed the record.

### Preconditions

The caller must supply valid storage for every access actually reached:

- The record discriminator must be readable at offset `0x04`.
- A discriminator of `0x13` on the initial check requires a readable word at offset `0x0C`.
- General dispatch requires the payload words used by its selected branch.
- Any selected indirect call requires a readable receiver table reference, a readable callable entry at the applicable offset, and a target compatible with that call’s arguments.

There are no explicit null, bounds, alignment, or callable-target checks. Invalid storage or an incompatible target has no defined recovery behavior in the available evidence.

The receiver need not be dereferenced on a path that selects no indirect call.

## Behavior

### Initial special-handling eligibility

The function first reads the 32-bit discriminator at record offset `0x04` and compares it with `0x13`.

Only when that comparison succeeds does it read the word at offset `0x0C`. Special handling is eligible when subtracting `0x8000` from that word produces `0xAA`. In the represented 32-bit model, the matching selector value is `0x80AA`.

The subtraction is a selector calculation, not a write to the record. For the matching input, the arithmetic fits within a signed 32-bit integer. The recovered expression alone does not establish a portable source-language overflow rule for other possible inputs.

If the initial discriminator differs from `0x13`, the special-selector word is not read for this eligibility check.

### Special-handler call

When both eligibility conditions hold, the function obtains the receiver’s callable entry at table offset `0x90` and invokes dependency **F-unknown-1** with:

1. The receiver address.
2. The original record address.

The dependency therefore receives access to the record itself, rather than copies of selected payload words.

After normal return:

- A nonzero result immediately becomes this function’s result.
- A zero result permits general dispatch to continue.

A nonzero special result prevents all subsequent discriminator checks and general-handler calls in this invocation.

A zero result does not imply that the record, receiver, table, or other state remained unchanged. Subsequent behavior uses the accesses described below, rather than an assumed snapshot from before the special call.

### General dispatch for discriminator `0x14`

After special handling is skipped or returns zero, the function reads the discriminator at record offset `0x04` and compares it with `0x14`.

If it matches, the function invokes dependency **F-unknown-2**, obtained from the receiver’s current table entry at byte offset `0x78`, with these argument positions:

1. Receiver address.
2. 32-bit word from record offset `0x08`.
3. 32-bit word from record offset `0x0C`.
4. 32-bit word from record offset `0x10`.

These payloads are passed as word values. This function does not establish whether the dependency interprets any word as signed data, unsigned data, flags, an address, or another encoding.

After the dependency returns normally, F-d8da6d0658bc returns `1`. No dependency result is tested or propagated.

### General dispatch for discriminator `0x13`

If the preceding `0x14` comparison fails, the function performs another discriminator read at record offset `0x04` and compares that value with `0x13`.

If it matches, the function invokes dependency **F-unknown-3**, obtained from the receiver’s current table entry at byte offset `0x5C`, with:

1. Receiver address.
2. 32-bit word from record offset `0x08`.
3. 32-bit word from record offset `0x0C`.

After normal return, F-d8da6d0658bc returns `1`, without testing or propagating any dependency result.

This branch does not directly read the payload at offset `0x10`.

### No matching general branch

If neither general comparison matches, the function returns `0`.

For a stable record with a discriminator other than `0x13` or `0x14`, no dependency is invoked.

### Read ordering and mutation visibility

The following sequencing is established:

- The initial discriminator check precedes any special-selector read.
- Special-handler execution, if selected, finishes before general dispatch begins.
- General dispatch checks `0x14` before checking `0x13`.
- The second general discriminator check is reached only after the first fails.
- Once a general handler returns normally, the function returns `1` without another record check.

The record discriminator is read again after a special handler returns zero. A special handler can therefore affect subsequent dispatch by changing that discriminator. Changes to payload words or the receiver’s table can likewise affect the subsequent general call.

Argument positions are established; the relative evaluation order of payload reads and indirect-target lookup within an individual call expression is not. This contract does not impose an unsupported order on those accesses.

The separate discriminator accesses must not be described as one immutable snapshot. Conversely, no concurrency, volatile-memory, atomicity, or synchronization guarantee is established. Behavior under concurrent unsynchronized mutation is **documented_only**.

## Side effects

The function contains no explicit writes to record storage, receiver storage, or global storage. It does not explicitly allocate, release, retain, or transfer memory.

Indirect dependencies may nevertheless have side effects. Their contracts are unavailable, so this function does not establish that they:

- Preserve the receiver, record, table, or global state.
- Leave payload values unchanged.
- Avoid allocation or memory release.
- Return normally.
- Preserve the validity of storage needed after a zero special result.

In particular, continued execution after a zero special result requires the subsequent record accesses and any selected receiver-table access to remain valid. No protection against dependency-induced invalidation is present.

At most two indirect calls are selected during one invocation: the special handler, followed by one general handler. If special handling is skipped, at most one indirect call is selected.

## Dependencies

The following opaque IDs are local references to unresolved indirect dependency roles. They do not claim globally resolved function identities. Different roles could resolve to the same target.

| Opaque function ID | Lookup | Supplied arguments | Result treatment |
|---|---|---|---|
| **F-unknown-1** | Receiver table, byte offset `0x90` | Receiver address; record address | Exact nonzero result propagated; zero continues dispatch |
| **F-unknown-2** | Receiver table, byte offset `0x78` | Receiver address; payloads at `0x08`, `0x0C`, `0x10` | Ignored; normal completion leads to `1` |
| **F-unknown-3** | Receiver table, byte offset `0x5C` | Receiver address; payloads at `0x08`, `0x0C` | Ignored; normal completion leads to `1` |

No direct dependency implementation or dependency-specific ownership contract is available.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not runtime observations. Unless otherwise stated, storage is valid, handlers return normally, and no actor changes relevant state.

1. **Ordinary `0x14` record.**  
   With discriminator `0x14` and payload words `A`, `B`, and `C`, F-unknown-2 receives the receiver followed by `A`, `B`, and `C`. Neither other dependency is called. The result is `1`.

2. **Ordinary `0x13` record without the special selector.**  
   With discriminator `0x13` and selector `0x80AB`, special handling is skipped. F-unknown-3 receives the receiver and the words at offsets `0x08` and `0x0C`. The result is `1`.

3. **Special result ends dispatch.**  
   With discriminator `0x13` and selector `0x80AA`, F-unknown-1 is called. If it returns `-7`, this function returns `-7`, with no general-handler call.

4. **Special decline followed by unchanged dispatch.**  
   With the same eligible record, if F-unknown-1 returns zero and leaves the discriminator at `0x13`, F-unknown-3 is subsequently called using the payload values available at that point. The final result is `1`.

5. **Special decline redirects dispatch.**  
   If F-unknown-1 changes the discriminator to `0x14`, supplies valid current payloads, and returns zero, F-unknown-2 is subsequently selected. F-unknown-3 is not selected. The result is `1`.

6. **Special decline removes the general match.**  
   If F-unknown-1 changes the discriminator to a stable value other than `0x13` or `0x14` and returns zero, no general handler is called and the result is `0`.

7. **Selector alone is insufficient.**  
   A stable discriminator of `0x12` does not invoke special handling even if offset `0x0C` contains `0x80AA`. The result is `0`; neither receiver-table access nor a special-selector read is required by this path.

8. **General-handler return does not control the result.**  
   If a selected general handler returns normally with a zero or negative machine result, F-d8da6d0658bc still returns `1`.

## Uncertainties

The following remain **documented_only**:

- The semantic meanings of the discriminator constants and payload words.
- Full receiver and record layouts, alignment requirements, and ABI details beyond the represented word accesses.
- Whether table entries represent methods, callbacks, or another dispatch convention.
- Dependency behavior, including ownership, global mutation, reentrancy, and exceptional control flow.
- Signedness interpretations of general-handler payloads.
- Portable overflow semantics for the recovered selector subtraction on arbitrary inputs.
- Hardware-visible access details under optimization, concurrency, or memory-mapped I/O.
- Behavior after invalid addresses, corrupted table entries, or dependency-induced storage invalidation.

No validation or error-recovery contract is inferred from their absence.

## Evidence

The supplied static representation establishes the field offsets, constants, indirect-call locations, argument positions, branch ordering, repeated discriminator checks, and return-value handling documented above.

Object-like and event-like terminology is inferred intent only. Dependency identities and internals are unresolved.

No runtime tests, execution traces, or implementation-access checks have been performed. The acceptance examples are conditional consequences of the static control flow, not claims of observed execution.
