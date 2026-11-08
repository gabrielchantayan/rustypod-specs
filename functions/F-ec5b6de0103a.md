<!-- Trusted provenance: F-ec5b6de0103a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ec5b6de0103a

## Purpose

F-ec5b6de0103a processes an event against a mutable receiver. Its established behavior consists of:

1. An optional preliminary action for a particular event subtype.
2. State-dependent entry actions when the event kind is `0x70001`.
3. Primary virtual event dispatch.
4. Conditional secondary virtual dispatch if the primary result is zero.

The returned value comes exclusively from the final dispatch stages. Preliminary and entry actions can occur even when the function ultimately returns zero.

**Inferred intent:** The labels `EnterDefault` and `EnterVolume` suggest mode-entry actions. The application meaning of those modes, event kinds, and state values is not established.

## Interface

### Arguments and result

| Argument | Established role |
|---|---|
| Receiver | Address of a mutable object containing a virtual-table address and the fields listed below. |
| Event | Address of a record containing an event kind and a conditionally interpreted data field. |
| Auxiliary A | Opaque 32-bit value copied into addressable local storage. |
| Auxiliary B | Opaque 32-bit value copied into a separate addressable local working slot. |

The result is a 32-bit integer:

- A nonzero primary-handler result is returned unchanged.
- Otherwise, an eligible secondary handler can supply a nonzero result, also returned unchanged.
- In all other normally returning cases, the result is zero.

Only zero versus nonzero is tested. Negative results receive no separate treatment. Whether these results signify handling, success, failure, or another status is unknown.

### Receiver layout

All offsets are byte offsets from the original receiver address.

| Offset | Established access |
|---|---|
| `0x00` | Virtual-table address used to locate indirect call targets. |
| `0x50` | 32-bit field used in the secondary-handler comparison. |
| `0xB0` | 32-bit word narrowed to a character-sized value, then compared with `1`. |
| `0x140` | 32-bit word narrowed to a character-sized value for state selection; separately, the address of a one-byte write. |
| `0x184` | 32-bit word narrowed to a character-sized argument for F-26bcb12c385a. |

For the narrowed reads, selection depends on the numeric low eight bits of the 32-bit value, subject to the character representation expressed by the supplied static representation. Character signedness does not change equality with the small positive constants used here. Its effect on the argument passed to F-26bcb12c385a is unresolved for values with the high bit set.

The state write is different from those narrowed reads: it writes the byte value `5` at address receiver plus `0x140`. Without an established byte order, this does **not** establish that the numeric low eight bits of the 32-bit word become `5`.

### Event layout

| Offset | Established access |
|---|---|
| `0x04` | 32-bit event kind. |
| `0x08` | 32-bit field interpreted according to the current event kind. |

For kind `0x16`, the field at `0x08` is used as an address, and a 32-bit subtype is read at offset `0x04` from the associated record.

For kind `0x0F`, the field at `0x08` is compared directly with the receiver’s 32-bit field at `0x50`; it is not dereferenced for that comparison.

No access to event offset `0x00` is established.

### Preconditions and representation limits

Every address actually accessed must designate readable or writable storage of the required size. Every selected virtual-table entry must identify a callable target with a compatible calling convention.

There are no explicit null checks, object-size checks, bounds checks, or indirect-target checks. Kind `0x16` requires an accessible associated-record subtype even when that subtype triggers no preliminary action.

The supplied representation expresses 32-bit integer and address-bearing fields and eight-bit character operations. It does not establish a portable ABI, byte order, alignment requirements, or complete dependency signatures. No arithmetic transformation of the auxiliary values or returned statuses is established.

## Behavior

### Preliminary subtype action

The first classification reads the event kind at `0x04`.

If the kind is `0x16`, the function reads the address-bearing field at `0x08`, then the subtype at associated-record offset `0x04`.

Only subtype `0x2503` causes F-26bcb12c385a to run. Its arguments are:

- The original receiver address.
- The 32-bit receiver value at `0x184`, narrowed to a character-sized value.

Subtype `0x2502`, subtype `0x2A`, and all other subtypes cause no preliminary call.

The preliminary call does not terminate processing, and its result is not used to determine the function’s return value. Its effects on receiver storage, event storage, global storage, or other reachable state are unknown. Later classifications use later reads rather than a preserved initial classification.

### Addressable local storage

After the preliminary stage, the function initializes four local slots, in this order:

1. A copy of the receiver address.
2. A copy of the event address.
3. A copy of auxiliary A.
4. A copy of auxiliary B.

Entry dependencies receive addresses of these slots. They can therefore observe or modify slot contents according to their own behavior.

Subsequent receiver accesses and final event dispatch use the original receiver and event arguments, not replacement addresses read from those slots. Changing a local address slot alone does not redirect those accesses.

These slots belong to the current invocation. No extended lifetime or safe retention of their addresses is established.

### State-dependent entry handling

The function rereads the event kind after local initialization. Only a current kind of `0x70001` enters state-dependent handling.

It then reads the 32-bit word at receiver offset `0x140` and narrows it for selection:

| Selected state | Action before primary dispatch |
|---|---|
| `0` | No entry action. |
| `1` | Volume entry, addressed-byte state write, then F-979606fd565e. |
| `2` | Virtual operation at table offset `0x158`, then common entry handling. |
| `3`, `4`, or `6` | Common entry handling. |
| `5` | F-979606fd565e only. |
| Any other value | No entry action. |

The state is not reselected after the operation at `0x158`. That operation can, however, affect subsequently read fields or global storage.

All normally returning paths continue to primary event dispatch.

### Common entry handling

For selected states `2`, `3`, `4`, and `6`, the function reads the 32-bit word at receiver offset `0xB0` and narrows it to a character-sized value.

Only equality with exactly `1` enables default entry. A different nonzero value does not suffice.

Default entry has this observable order:

1. Call F-9b9c1c7483f4 with the address of the local event-address slot, the label `EnterDefault`, and the address of the local receiver-address slot.
2. Call the receiver’s virtual operation at table offset `0x11C`, supplying the original receiver and the dependency’s 32-bit result.
3. Call F-8a5aadac202f with the address of that same event-address slot.

Volume entry follows whether or not default entry ran.

For selected state `2`, the operation at `0x158` precedes the flag read. Its changes to the word at `0xB0` therefore affect default-entry eligibility. Selected state `1` bypasses both this operation and the flag read.

### Volume entry, state write, and global operand

Volume entry has this order:

1. Call F-9b9c1c7483f4 with the address of the local auxiliary-B slot, the label `EnterVolume`, and the address of the local auxiliary-A slot.
2. Call the receiver’s virtual operation at table offset `0x11C` with the original receiver and the returned 32-bit value.
3. Call F-8a5aadac202f with the address of the same auxiliary-B slot.
4. Write the byte value `5` at receiver offset `0x140`.
5. Read the designated global operand storage and pass its current value, together with the original receiver, to F-979606fd565e.

The global operand is read when this final call is reached. It is not a fixed literal or an invocation-wide snapshot. The preceding dependencies have unknown global-state effects, so their execution may affect the value supplied.

The direct state write affects only the byte at the addressed location. It does not directly replace the entire 32-bit state word. With byte order unspecified, neither a numeric low-byte value of `5` nor a subsequent state selection of `5` follows unconditionally from this write.

The helper results are not tested for success before proceeding. The result of F-9b9c1c7483f4 is used as an argument; the results of the other entry operations do not control final status propagation.

### Selected state `5`

For selected state `5`, the function skips default entry, volume entry, and the direct state-byte write.

It reads the designated global operand storage when the F-979606fd565e call is reached and supplies that current value as the second argument.

Use of the same storage location does not establish equality with the operand supplied on a volume-entry path, on another invocation, or before an intervening dependency call.

### Primary event dispatch

After entry handling, the function rereads the event kind.

If the current kind is `0x16`, it reads the current associated-record address and subtype. It selects:

- Table offset `0x10C` when the current subtype is `0x2A`.
- Table offset `0x110` otherwise.

For every other event kind, it selects table offset `0x110` without a nested subtype read.

The selected operation receives the original receiver and event addresses. A nonzero result returns immediately, unchanged. No secondary-condition reads occur on that path.

Dependencies executed earlier can change the event and thereby change this selection.

### Secondary event dispatch

After a zero primary result, the function rereads the event kind.

If it is not `0x0F`, the function returns zero without reading the identity fields for this condition.

For current kind `0x0F`, it reads:

1. The 32-bit event field at `0x08`.
2. The 32-bit receiver field at `0x50`.

If they differ, it returns zero. If they match, it calls the virtual operation at table offset `0xE0` with the original receiver and event addresses.

A nonzero secondary result is returned unchanged; a zero secondary result produces zero.

Because these reads follow the primary call, primary-handler mutations can enable or disable secondary dispatch.

## Side effects

The only established direct write to receiver storage is the one-byte write of `5` at offset `0x140` after volume entry. Other bytes of the containing word are not directly written by that operation.

The function also initializes local slots and exposes their addresses to entry dependencies. No direct write to event storage or global storage is established.

Dependencies may modify the receiver, event, associated records, local slots, globals, or other reachable storage. No preservation guarantee is supplied. In particular, the global operand read for F-979606fd565e must not be treated as invariant across earlier calls.

Separate virtual calls obtain their targets through receiver-table accesses at the relevant call sites. The evidence does not establish that dependencies preserve the receiver’s table address or its entries.

No allocation, ownership transfer, retain operation, or release operation is established by this function alone. F-8a5aadac202f follows each entry-value submission, which is consistent with cleanup or finalization, but that interpretation remains inferred.

There is no explicit rollback, exception recovery, or compensation. If a dependency does not return normally, subsequent operations—including the state-byte write or final dispatch—cannot be assumed to occur.

## Dependencies

### Identified functions

| Opaque function ID | Established interaction |
|---|---|
| F-26bcb12c385a | Preliminary action for kind `0x16`, subtype `0x2503`; receives receiver and a narrowed character-sized value. |
| F-9b9c1c7483f4 | Receives two local-slot addresses and an entry label; produces a 32-bit value passed to virtual operation `0x11C`. |
| F-8a5aadac202f | Receives the corresponding working-slot address after the virtual operation at `0x11C`. |
| F-979606fd565e | Receives the original receiver and a value freshly read from designated global storage when the call is reached. |

The global operand’s value, meaning, access width, and preservation guarantees are not established by the supplied declaration-free use. Earlier dependencies cannot be assumed to leave that storage unchanged.

### Virtual dependencies

No opaque function IDs are supplied for runtime-selected targets. They are identified here by their established table slots, not by invented target identities.

| Table offset | Established interaction |
|---|---|
| `0x158` | Receiver-only call for selected state `2`. |
| `0x11C` | Receives receiver and the 32-bit entry value. |
| `0x10C` | Primary handler for current kind `0x16`, subtype `0x2A`. |
| `0x110` | Primary handler for all other current classifications. |
| `0xE0` | Secondary handler after a zero primary result and matching kind-`0x0F` identity fields. |

Dependency ownership rules, global preservation, failure semantics, and retained-pointer behavior remain unknown.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless an example says otherwise, dependencies return normally and do not modify storage relevant to later decisions.

1. **Preliminary action:** Kind `0x16` with subtype `0x2503` calls F-26bcb12c385a before final dispatch. With classification unchanged, the primary slot is `0x110`.

2. **Special dispatch:** Kind `0x16` with subtype `0x2A` causes no preliminary action and selects primary slot `0x10C`. Subtype `0x2502` causes no preliminary action and selects `0x110`.

3. **Direct volume entry:** Kind `0x70001` with selected state `1` performs volume entry without reading the default-entry flag. It then writes byte `5` at receiver offset `0x140`, reads the global operand, and calls F-979606fd565e. This does not unconditionally establish a numeric state value of `5`.

4. **Common entry order:** Kind `0x70001`, selected state `2`, and a flag narrowing to `1` produce: operation `0x158`; default entry; volume entry; addressed-byte write; global operand read and F-979606fd565e; primary dispatch.

5. **Flag mutation:** If operation `0x158` changes the flag so its narrowed value is `2`, default entry is skipped. Volume entry still occurs.

6. **Existing selected state:** Selected state `5` causes no entry-label sequence and no direct state write. F-979606fd565e receives the value read from global storage at that call.

7. **Addressed-byte preservation:** If four accessible bytes begin at receiver offset `0x140`, the direct assignment replaces only the first addressed byte with `5`. The other three are unchanged by that assignment. Under a separately established little-endian representation, this also changes the numeric low eight bits to `5`; without that condition, the equivalence is not established.

8. **Late global value:** If an entry dependency changes the designated global operand storage and no later operation changes it again, F-979606fd565e receives the changed value, not an earlier snapshot.

9. **Primary short circuit:** A primary result of `-7` is returned unchanged without evaluating the secondary condition.

10. **Secondary result and mutation:** A primary handler that returns zero after producing a matching kind-`0x0F` event enables secondary handling. A nonzero secondary result is returned unchanged; a zero result produces zero.

11. **No transition:** Kind `0x70001` with selected state `0` or an unlisted value skips entry actions and proceeds to primary dispatch.

## Uncertainties

The following remain **documented_only**:

- Application meanings of event kinds, subtypes, labels, states, and receiver fields.
- Byte order and the relationship between the addressed-byte write and later numeric state selection.
- Character signedness and argument-extension behavior for the preliminary narrowed argument.
- The value, representation, access width, meaning, and preservation guarantees of the designated global operand.
- Dependency effects on global storage and whether concurrent changes are possible.
- Representation and lifetime of values produced by F-9b9c1c7483f4.
- Whether F-8a5aadac202f performs cleanup, release, or another action.
- Ownership of receiver, event, associated storage, auxiliary values, and dependency-produced values.
- Whether dependencies retain local-slot addresses.
- Reentrancy, synchronization requirements, dependency failures, nonlocal exits, and invalid-storage fault behavior.
- ABI details beyond the widths and offsets expressed in the supplied representation.

## Evidence

The supplied static representation establishes the comparisons, field offsets, 32-bit word reads followed by character narrowing, addressed one-byte state write, local-slot exposure, dependency order, virtual-table offsets, repeated event classifications, and nonzero-result propagation.

It also establishes that F-979606fd565e receives a value obtained from global storage when its call is reached. On volume-entry paths, that read follows the entry dependencies and the one-byte state write. No immutability or cross-call equality of that value is established.

Mode-entry intent and a possible cleanup role are inferences, not behavioral guarantees. Dependency implementations, complete declarations, target byte order, and runtime observations were not provided. No runtime tests have been performed.
