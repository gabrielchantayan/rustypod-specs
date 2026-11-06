<!-- Trusted provenance: F-f3b1ef9aa1bb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f3b1ef9aa1bb

## Purpose

F-f3b1ef9aa1bb selects a destination-related value, updates a captured position target, invokes a transfer and two preparation dependencies, and requests storage for a four-word record. If that request returns non-null, it performs payload writes, obtains a context, and performs a head-linking sequence.

The Boolean result distinguishes only a null from a non-null record-storage result, provided execution reaches the return. It does not report whether earlier operations succeeded, whether the record remains intact, or whether the resulting structure is a valid list.

The apparent intent is to prepare data and prepend an associated record. That interpretation is inferred. The supplied body does not establish the meanings of the data or payload, fresh allocation, disjoint storage, ownership transfer, or preservation of payload values across dependencies.

## Interface

### Arguments

The function accepts four 32-bit arguments.

| Position | Established use |
|---|---|
| First | Not read by the supplied body. |
| Second | Not read by the supplied body. |
| Third | Initially placed in temporary storage; replaced before any subsequent use on the nonzero-measurement path and unused on the zero-measurement path. |
| Fourth | Initially placed in adjacent temporary storage; likewise replaced or unused. |

No original argument value reaches an explicit dependency argument or persistent write. In particular, the third and fourth arguments do not control the reservation request.

Changing the four arguments alone therefore does not change the body’s established explicit behavior, given the same relevant state and dependency behavior.

### Result

On normal return:

- **False** means F-d33f6d597e5d returned null.
- **True** means F-d33f6d597e5d returned non-null and execution completed the subsequent accesses.

There is no separate failure result for measurement, destination selection, transfer, preparation, resource production, or context lookup. Invalid accesses or non-returning dependencies are not converted into false.

A true result does not certify record freshness, payload preservation, a valid list, or ownership acquisition.

### State roles

These names describe observed roles, not public identifiers:

- **Position target:** a pointer captured from shared state at entry. The function later writes one 32-bit word through it.
- **Source object:** a fixed shared address passed to measurement and transfer dependencies. Its extent and contents are unknown.
- **Default base:** a shared 32-bit address-like value read after measurement.
- **Preparation configuration:** two shared 32-bit values passed to a preparation dependency.
- **Resource selector:** a shared 32-bit value passed to a resource-producing dependency.
- **First metadata value:** a shared 32-bit value read after resource production.
- **Second metadata value:** a shared 32-bit value read immediately after the first. Its location is also used for the third argument of the preparation call.
- **Record storage:** the pointer returned by the 16-byte request.
- **Context head location:** the 32-bit field at byte offset `0x38` from the returned context.

These role names do not establish separation between their storage regions.

### Record-relative access layout

For a non-null record-storage result, the body uses these offsets:

| Byte offset | Explicit write |
|---|---|
| `0x00` | Head value read after context lookup |
| `0x04` | Retained resource result |
| `0x08` | Retained first metadata value |
| `0x0C` | Retained second metadata value |

The request size is 16 bytes. The body accesses four consecutive 32-bit words; it does not establish the allocator’s actual extent, initialization policy, freshness, or alignment guarantees.

This table describes writes, not unconditional final contents. The context dependency runs after the payload writes and may modify the storage. The final head publication may itself overlap a record word.

## Behavior

### Measurement and destination selection

The function captures the position-target pointer before calling F-1cb585febd84 with the source-object address.

The measurement result is treated as an unsigned 32-bit quantity. It controls the reservation branch and is retained as the transfer count.

After measurement returns, the function reads the default base. This read occurs even if the measurement is nonzero and a subsequent reservation result replaces the default base.

Consequently, changes made by the measurement dependency can affect the observed default base, but changing the shared location from which the position-target pointer was obtained does not replace the already captured pointer.

For a zero measurement:

- The default base remains selected.
- F-fe78e25e82ff is not called.

For a nonzero measurement, F-fe78e25e82ff receives, in order:

1. The captured position-target pointer.
2. The greater of 32 and the unsigned measurement.
3. The measurement.

Its return value becomes the selected base without validation.

The second argument is 32 for measurements from 1 through 32 inclusive. For measurements above 32, including those with the high bit set, it equals the measurement. The body establishes a minimum argument value, not that this argument represents allocation capacity or usable bytes.

### Position update and transfer

The function writes the selected base plus 12 through the captured position-target pointer as a 32-bit value.

It then invokes the external transfer dependency with:

1. Selected base plus 12 as destination.
2. Source-object address as source.
3. The retained unsigned measurement as count.

The position write precedes the transfer call. The call occurs even when the count is zero.

No source-content snapshot is visible. If the position write aliases the source object, it can change what the transfer dependency subsequently observes. Earlier dependencies may also modify the source.

The selected base is represented by a signed 32-bit intermediate in the supplied representation. The addition of 12 has no explicit overflow check. The body does not establish a portable overflow or wraparound contract. Normal address use requires the displacement and resulting accesses to be valid under the actual execution environment.

The external operation’s argument pattern suggests copying, but its precise contract is unavailable. Copy semantics, overlap handling, zero-count pointer requirements, and actual memory effects remain unresolved.

### Preparation and retained payload values

After the transfer returns, F-804adc29bbfb receives:

1. The captured position-target pointer.
2. The first preparation-configuration value.
3. The second preparation-configuration value.

Its result is not consumed.

Next, F-9956cc416ca8 receives the resource selector. Its 32-bit result is retained.

After that call returns, the body reads the first metadata value and then the second metadata value. The second comes from the shared location previously used for the third preparation argument. These are separate reads separated by dependency calls and may produce different values.

The resource result and both metadata values are retained before requesting record storage. Later changes to the original shared metadata locations do not cause the body to reload them.

This retention does not guarantee that the written payload survives later operations. It establishes only which values the body supplies to its three payload writes.

### Record-storage failure

F-d33f6d597e5d is called with 16.

If it returns null:

- The body performs no record-relative writes.
- F-187ab9ba4eb6 is not called.
- The body performs no context-head access.
- The function returns false.

The position update and all preceding calls have already occurred. No rollback or release is visible. The allocation dependency itself may have effects even when it returns null.

### Non-null storage and publication sequence

For a non-null result, the body writes:

1. The retained resource result at record offset `0x04`.
2. The retained first metadata value at offset `0x08`.
3. The retained second metadata value at offset `0x0C`.

It then calls F-187ab9ba4eb6 with no explicit arguments.

That dependency’s effects are unknown. The body provides no guarantee that it preserves the record payload, including when the storage is accessible through shared state or previously retained pointers. Conversely, if the payload stores overlap state used by that dependency, those stores may affect its behavior.

After the context dependency returns, the body:

1. Reads the 32-bit context field at offset `0x38`.
2. Writes that observed value at record offset `0x00`.
3. Writes the record pointer into the context field at offset `0x38`.
4. Returns true.

There is no context-null check, head validation, traversal, or explicit write through the old head value.

These accesses are established regardless of whether their regions overlap. Ordinary prepend topology requires additional, unestablished assumptions:

- The record occupies suitable writable storage.
- Its 16-byte region is separate from the context head field.
- It is separate from any existing record whose preservation is expected.
- Relevant state is not concurrently modified.
- The context dependency preserves the payload if final payload equality is required.

Without separation, the final publication write can overwrite a record word. If the returned record storage aliases the previous head record, the payload writes can modify that previous record before the head is read. The body therefore does not unconditionally preserve an existing record or create a distinct new list element.

## Side effects

### Direct accesses and ordering

The body directly performs:

- A 32-bit write through the captured position-target pointer.
- On a non-null storage result, three 32-bit payload writes before context lookup.
- After context lookup, a 32-bit head read, a 32-bit record-link write, and a 32-bit context-head write.

The last three accesses occur in that order. Their addresses are not proven disjoint.

The dependencies may add reads, writes, resource operations, or pointer retention. Their effects cannot be bounded from this body.

No explicit synchronization or memory-ordering mechanism is visible. Source-level access order does not establish thread-safe publication.

### Ownership and lifetime

The function contains no explicit free or release operation.

Under ordinary disjoint-storage assumptions, the final head write makes the record pointer available through the context head. This is evidence of intended publication, not proof of ownership transfer, fresh allocation, valid reachability, or a teardown contract.

Without those assumptions, even ordinary list reachability is not established: overlapping stores can corrupt a payload or link, and the returned storage may already belong to an existing object.

The ownership of the resource result, source object, selected destination, record storage, and context remains unknown. Lack of an explicit release after failure does not by itself prove a leak.

### Conditions for normal execution

Normal execution requires, at minimum:

- A captured position target valid for the explicit 32-bit write.
- Destination arithmetic and transfer arguments acceptable to the execution environment and transfer dependency.
- For a non-null record result, storage permitting the record-relative accesses.
- A context result permitting the read and write at offset `0x38`.
- Dependencies that return normally.

These requirements concern access validity. Freshness and disjointness are additional conditions needed for conventional list semantics, not checks performed by the function.

## Dependencies

| Opaque ID | Observed interface and use | Unestablished contract |
|---|---|---|
| F-1cb585febd84 | Receives the source address; returns an unsigned 32-bit measurement. | Meaning of measurement, terminator policy, source changes, other effects. |
| F-fe78e25e82ff | For nonzero measurement, receives the position target, unsigned maximum of 32 and measurement, and measurement; returns the selected base. | Capacity, allocation behavior, failure values, ownership, state preservation. |
| F-804adc29bbfb | Receives the captured position target and two configuration values after transfer. | Preparation meaning, errors, mutations, retention, resource handling. |
| F-9956cc416ca8 | Receives the resource selector; supplies the retained word written at record offset `0x04`. | Result meaning, failure signaling, ownership, shared-state effects. |
| F-d33f6d597e5d | Receives 16; null/non-null determines the subsequent branch. | Freshness, disjointness, actual extent, alignment, initialization, ownership, side effects. |
| F-187ab9ba4eb6 | Called after the payload writes; supplies the context for head access. | Context validity, lifetime, separation from record storage, payload preservation, other effects. |

The external transfer dependency has no supplied opaque ID. Its observed destination, source, and count arguments are documented above; no identity or stronger contract is invented.

No dependency is assumed to preserve globals, preserve record storage, retain no pointers, or acquire or release ownership.

## Acceptance examples

These are conditional static examples, not executed tests.

### Zero measurement

Given a zero measurement and otherwise valid execution:

- F-fe78e25e82ff is skipped.
- The position write uses the default base plus 12.
- Transfer is invoked with count zero.
- Preparation, resource production, and the 16-byte storage request still occur.

Zero measurement does not itself select false.

### Reservation thresholds

Given measurement 1, F-fe78e25e82ff receives the position target, 32, and 1. Transfer later receives count 1, not 32.

At measurement 32, the reservation’s last two arguments are both 32. At 33, both are 33. Because the comparison is unsigned, a measurement of `0x80000000` also becomes both of those arguments; no successful storage outcome is implied.

### Final storage request returns null

If all preceding calls return normally and F-d33f6d597e5d returns null, the function returns false without context lookup or record writes.

An implementation that skips the earlier position update or preparation calls merely because this request fails would not match the observed behavior.

### Conventional successful prepend

Assume valid, fresh record storage disjoint from the context head and existing records; a valid existing head; no interfering mutation; and preservation of the three payload words by F-187ab9ba4eb6.

Then, on normal return:

- The payload words contain the retained resource and metadata values.
- The record’s link contains the head value observed after context lookup.
- The context head contains the record pointer.
- The function returns true.
- The body’s record-relative stores do not modify a previous record because the stipulated regions are separate.

Without payload preservation, only the three earlier payload writes are established, not those final payload contents.

### Context lookup modifies payload

Suppose the context dependency can access the returned storage and changes its word at offset `0x08`, while leaving all subsequent accesses valid.

The body does not restore that payload word. A contract requiring the original metadata value at return would be too strong. The function can still return true.

### Head field overlaps the record

Suppose the context head field occupies the record word at offset `0x0C`, with otherwise valid accesses.

The final publication write places the record pointer into that word, overwriting its then-current contents. The earlier metadata write still occurred, but the ordinary four-word final-layout claim does not hold.

### Returned storage aliases the previous record

If F-d33f6d597e5d returns storage already used by the current head record, the three payload writes target that existing record. No freshness check prevents this.

If context lookup leaves the head equal to that same pointer, the subsequent link write makes the record link to itself. True does not reject this topology.

## Uncertainties

The following remain **documented_only**:

- Source encoding, extent, and measurement meaning.
- Purpose of the 12-byte displacement.
- Transfer semantics and memory-access requirements.
- Overflow behavior and validity at address-space boundaries.
- Reservation and allocation failure conventions.
- Record freshness, storage separation, and actual allocated extent.
- Context validity, lifetime, and separation from record storage.
- Preservation of payload and other state by dependencies.
- Payload meaning, resource ownership, and destruction responsibilities.
- Concurrency, reentrancy, and synchronization requirements.
- Exact relative ordering of separate shared-value reads used as arguments to the same call.

Preparation and list publication are inferred intent. The calls, branch conditions, argument values, explicit access order, and allocation-based Boolean decision are established static behavior.

## Evidence

This contract is based on the supplied function body and opaque dependency mapping.

Static evidence establishes the 32-bit argument and word widths, unsigned measurement comparison, zero/nonzero reservation branch, minimum reservation argument of 32, displacement of 12, storage request of 16, record-relative offsets, context offset `0x38`, and Boolean decision.

The context dependency occurs between payload initialization and head linking. No supplied evidence establishes payload preservation or disjoint record and context storage. Accordingly, final payload and conventional list-topology claims are conditional.

No dependency bodies, external transfer contract, or runtime observations were supplied. No runtime tests have been performed.
