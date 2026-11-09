<!-- Trusted provenance: F-ae991b0f9dc5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ae991b0f9dc5

## Purpose

F-ae991b0f9dc5 performs a mode-dependent transition on a caller-supplied object, then conditionally notifies a relationship target and propagates flags through object callbacks.

The statically established behavior consists of:

1. For zero Mode, checking and clearing a state field, with an exceptional call sequence when the checked state is nonzero.
2. For nonzero Mode, configuring an embedded region, conditionally configuring an obtained value, submitting a temporary descriptor, writing three external destinations, and passing a derived value to the input object.
3. On continuing paths, performing a zero-mode callback sequence, relationship notification, and conditional flag propagation.

“Activation,” “deactivation,” “rendering configuration,” and similar semantic interpretations are **inferred intent**, not established behavior. The contract below specifies observable accesses, calls, ordering, and conditions without assigning those meanings.

## Interface

### Inputs and result

- **Object:** a pointer to mutable storage accessed through 32-bit object words and represented pointer fields.
- **Mode:** an unsigned 32-bit value.
- **Result:** no explicit return value.

Mode is not restricted to zero or one. Zero selects one entry path; every nonzero value selects the other. A later notification produces an argument of one for zero Mode and zero for every nonzero Mode.

Normal completion depends on invoked dependencies returning and on all accessed storage and dispatch entries being valid. The function provides no explicit error result.

### Directly accessed object layout

Offsets are byte offsets from Object. The table describes the supplied representation, not independently inspected machine instructions.

| Offset | Represented access | Role |
|---|---|---|
| `0x00` | 32-bit pointer-bearing word | Dispatch-table address |
| `0x34` | 32-bit pointer-bearing word | Related-object address |
| `0x3C` | 32-bit pointer-bearing word | Notification-target address |
| `0x48` | 32-bit word | Flags |
| `0x80` | Address formation | Embedded region passed to F-6d16652f2d03 |
| `0xA0` | 32-bit word-valued expression converted to `char` | Gate for the related-object callback |
| `0xC0` | 32-bit word | Argument to an unmapped dependency |
| `0xDC` | 32-bit word | Zero-mode state |
| `0xE0` | Address formation | Embedded region used by the nonzero path |
| `0x9A0` | 32-bit word-valued expression converted to `char` | Gate for querying the obtained value |

The accesses at `0xA0` and `0x9A0` are **not established as independent 8-bit memory loads**. Each represented expression accesses an object word and converts its value to `char` before comparison with zero. The tested value is that conversion result.

Consequently:

- These are not tests of the entire 32-bit word against zero.
- They are not specified as tests of the byte physically located at the listed address.
- No byte position within either word is selected by this contract.
- Actual generated load widths, target endianness, and target-specific character conversion rules are not independently established.

Additional layouts used directly are:

| Storage | Access |
|---|---|
| Notification target | Pointer-bearing field at `+0x78` |
| Related object | Dispatch-table pointer at `+0x00`; 32-bit flags at `+0x48` |
| Final callback result | Writable 32-bit flags at `+0x48` |
| Object dispatch table | 32-bit adjustment at `−0x0C`; callable entries at `+0x1C` and `+0x5C` |
| Related-object dispatch table | Callable entry at `+0x128` |

### Preconditions

The function does not explicitly validate nullness, alignment, storage extent, lifetime, or callable entries except where a particular null check is described below.

Normal execution requires suitable live storage for every access actually reached. In particular:

- The nonzero path requires readable adjustment metadata before Object’s dispatch table.
- Embedded addresses passed to dependencies must satisfy those dependencies’ requirements.
- The final object callback must return a value suitable for a flags read and write; this function does not check that result for null.
- A gate described as a converted object word must not be treated as requiring only one accessible byte on the strength of the character comparison.

Additional extent, ownership, and calling-convention requirements imposed by dependencies are **documented_only**.

## Behavior

### 1. Zero-mode entry path

For zero Mode, the function reads the 32-bit state at `Object + 0xDC`.

If the state is nonzero, it calls, in order:

1. F-ff3f21513a57, with no explicit arguments.
2. F-feee9156a257, with no explicit arguments.

The supplied representation annotates the second dependency as nonreturning. Under that annotation, neither the state-clear write nor the common tail is reached. The annotation is not independent proof of the dependency’s runtime behavior; execution after an unexpected return is not established here.

If the state is zero, the function explicitly writes a 32-bit zero to the same field and continues. This write occurs even though the value just checked was already zero.

### 2. Nonzero-mode entry path

For every nonzero Mode, the following sequence occurs, subject to prior calls returning.

#### Embedded-region preparation

The function obtains Object’s dispatch-table address and the 32-bit adjustment stored at table offset `−0x0C`. It adds that adjustment to the represented address of Object and uses the result as an argument.

It then calls:

1. F-9a97db7d2605 with `Object + 0xE0` and the adjusted address.
2. F-cadf0b285940 with `Object + 0xE0` and `15`.
3. F-a3a247b2e8f6 with `Object + 0xE0`.
4. F-unmapped-01 with `Object + 0xE0`, `1`, and the then-read 32-bit value at `Object + 0xC0`.
5. F-unmapped-02 with `Object + 0xE0`.

The adjustment is expressed using the representation’s 32-bit integer/address model. No bounds or overflow check is visible. Exact overflow behavior, address validity, and pointer-provenance rules remain unresolved.

F-unmapped-01 and F-unmapped-02 are local opaque labels for dependencies lacking supplied mappings, not recovered identities.

#### Obtained-value configuration

The function calls F-bb50a072e902 with `0`, then calls F-5b4ae75440ab with that result and `1`. The second result is called the **obtained value** below. No ownership or concrete type is established.

Next it evaluates the 32-bit word-valued expression at `Object + 0x9A0`, converted to `char`.

- If the converted value is zero, configuration proceeds without calling F-1a3ec5a48643.
- Otherwise, F-1a3ec5a48643 receives the obtained value. A zero result permits configuration; a nonzero result skips it.

This short-circuit condition depends on the converted character value, not directly on whether the whole word is zero.

When configuration proceeds:

1. F-c51ae8bb81bd receives the obtained value.
2. F-abc658b0bf5e receives the address of the first temporary record.
3. The function overwrites selected record fields.
4. F-dcb90cca1e28 receives the obtained value and the record address.
5. F-c9bd3b9286da receives the obtained value and `0`.

The explicit record writes occur in the following order:

| Order | Offset | Width | Value |
|---:|---:|---:|---:|
| 1 | `0x00` | 8 bits | `2` |
| 2 | `0x01` | 8 bits | `3` |
| 3 | `0x04` | 32 bits | `0` |
| 4 | `0x18` | 32 bits | `240` |
| 5 | `0x28` | 32 bits | `0` |
| 6 | `0x08` | 32 bits | `0` |
| 7 | `0x24` | 32 bits | `0` |
| 8 | `0x10` | 32 bits | `240` |
| 9 | `0x30` | 16 bits | `0` |
| 10 | `0x14` | 32 bits | `320` |
| 11 | `0x2D` | 8 bits | `0` |
| 12 | `0x0C` | 32 bits | `320` |

These widths describe explicit record assignments, unlike the word-to-character gate expressions.

Unlisted bytes are not explicitly assigned by this function. Their contents depend on F-abc658b0bf5e and any other effects on the temporary storage. Neither complete initialization nor zero-filled padding is established.

#### Descriptor submission and external writes

Whether or not obtained-value configuration occurred, the function next:

1. Calls F-971745bb50d1 with a value read from an external data entry.
2. Writes another external value into the second temporary descriptor’s 32-bit field at offset `0x00`.
3. Calls F-c5a3e2a1a1d4 with the descriptor address at offset `0x04`, another external value, `4`, and `16`.
4. Writes the 32-bit value `0x1700` at descriptor offset `0x44`.
5. Calls F-16920fdc01be with the descriptor address.
6. Calls F-5e828d4b34e9 with that same address.
7. Writes `0xFFFFFFFF` to a first externally addressed 32-bit destination.
8. Writes `0xFFFFFFFF` to a second externally addressed 32-bit destination.
9. Writes `1` to a third externally addressed 32-bit destination.
10. Calls F-0ddd45153607 with four arguments, each `0x10000`.

The descriptor region beginning at `0x04` is represented as 64 bytes. The arguments `4` and `16` are established; interpreting them as dimensions, element count, or element size is not.

External values and destination addresses are used at their respective sequence points. Dependencies must not be assumed to preserve them. Destination identities and possible aliasing are unknown, so the three writes are ordered operations, not necessarily writes to distinct storage.

The all-ones writes are data writes, not an established error indication.

#### Derived-value chain

The nonzero path ends with:

1. F-a66a9be680df, with no explicit arguments.
2. F-c5a232e4455b, with the preceding result and `12`.
3. F-4e95b66bff0e, with the preceding result and `0x7901`.
4. F-8e50f265e4fa, with Object, the final result, and `0`.

No ownership transfer or resource release is established by this chain.

### 3. Zero-mode conditional callback

After the selected entry path, zero Mode causes F-a1a48bd3fbde to receive Object.

If it returns nonzero:

1. The current Object dispatch-table entry at `+0x1C` is invoked with Object.
2. F-20460d1f31d3 is called without explicit arguments.

A zero result skips both calls. Nonzero Mode skips this query entirely.

### 4. Relationship notification

The function reads the notification-target pointer at `Object + 0x3C`, then the related-object pointer at `Object + 0x34`. These are post-entry-path reads, not entry-time snapshots.

If the notification target is nonnull and its field at `+0x78` equals Object, F-df6da5bd408b receives the target and:

| Mode | Notification argument |
|---:|---:|
| `0` | `1` |
| `1` | `0` |
| `2` through `0xFFFFFFFF` | `0` |

Although the represented calculation includes subtraction from one, its greater-than-one branch forces zero. Large unsigned Mode values therefore do not become large notification arguments.

The related-object pointer captured before notification is retained for the following fallback flags check. F-df6da5bd408b must not be assumed to preserve Object’s fields or the captured object’s storage.

### 5. Conditional flag propagation

The function checks the current 32-bit flags at `Object + 0x48`.

Propagation proceeds if:

- Object has flag `0x10`; or
- Object lacks that flag, but the previously captured related-object pointer is nonnull and its 32-bit flags at `+0x48` contain `0x10`.

If neither condition holds, the function returns. The related object’s flags are not read for this fallback when Object already has `0x10`.

When propagation proceeds:

1. A 32-bit read/modify/write sets Object’s flag `0x20`, preserving the other bits read for that operation.
2. F-6d16652f2d03 receives Object and `Object + 0x80`.
3. The related-object pointer at `Object + 0x34` is read again.
4. If that pointer is nonnull, the function evaluates the word-valued expression at `Object + 0xA0`, converted to `char`.
5. If that converted value is zero, the related object’s dispatch-table entry at `+0x128` is invoked.
6. Object’s current dispatch-table entry at `+0x5C` is invoked with Object.
7. The returned value is treated as an object address, and a 32-bit read/modify/write sets its flag `0x40` at `+0x48`.

The gate at `0xA0` is evaluated only after the nonnull related-pointer check. A nonzero converted value suppresses only the related-object callback, not the final Object callback.

No explicit argument is represented for the `+0x128` callback. An implicit receiver convention is unresolved. The final callback result is not checked for null.

## Side effects

Direct persistent writes are:

- The zero-mode state-clear write at `Object + 0xDC`.
- Three ordered external 32-bit writes on the nonzero path.
- Setting Object’s flag `0x20` when propagation qualifies.
- Setting flag `0x40` on the final callback result.

Aliasing may cause these writes to affect overlapping storage. No distinct-object guarantee is visible.

Dependencies and indirect callbacks may mutate objects or globals, allocate or release resources, perform I/O, or prevent normal completion. Their effects must not be inferred from opaque identifiers.

Both temporary records use local storage passed by address. No explicit allocation or release for that storage is shown. The function does not establish that recipients copy the records or that retaining their addresses is safe after return. Retention and ownership obligations remain **documented_only**.

There is no visible rollback sequence or explicit handling of dependency-reported errors. The flag operations are not established as atomic or synchronized.

## Dependencies

Identifiers describe observed use only.

| Dependencies | Observed role |
|---|---|
| F-ff3f21513a57; F-feee9156a257 | Nonzero-state zero-mode sequence |
| F-9a97db7d2605; F-cadf0b285940; F-a3a247b2e8f6; F-unmapped-01; F-unmapped-02 | Embedded-region preparation |
| F-bb50a072e902; F-5b4ae75440ab | Obtained-value chain |
| F-1a3ec5a48643 | Conditional configuration query |
| F-c51ae8bb81bd; F-abc658b0bf5e; F-dcb90cca1e28; F-c9bd3b9286da | First-record configuration |
| F-971745bb50d1; F-c5a3e2a1a1d4; F-16920fdc01be; F-5e828d4b34e9 | Descriptor preparation and submission |
| F-0ddd45153607 | Four-constant call |
| F-a66a9be680df; F-c5a232e4455b; F-4e95b66bff0e; F-8e50f265e4fa | Derived-value chain |
| F-a1a48bd3fbde; F-20460d1f31d3 | Zero-mode conditional sequence |
| F-df6da5bd408b | Relationship notification |
| F-6d16652f2d03 | Propagation operation |

Three further dependencies are indirect dispatch calls: Object entries `+0x1C` and `+0x5C`, and the related-object entry `+0x128`. Their implementations and stable identities are unavailable.

## Acceptance examples

These are conditional acceptance scenarios, not executed tests. Unless stated otherwise, dependencies return normally and introduce no interfering mutations.

1. **Zero Mode, nonzero state.**  
   F-ff3f21513a57 precedes F-feee9156a257. Under the nonreturning annotation, no subsequent state-clear write, notification, or propagation occurs.

2. **Zero Mode, no propagation condition.**  
   With state zero, no notification target, no related object, Object’s `0x10` flag clear, and F-a1a48bd3fbde returning zero, the function writes zero to the state and returns without setting `0x20`.

3. **Zero converted configuration gate.**  
   With nonzero Mode and the word expression at `0x9A0` converting to zero, first-record configuration occurs without F-1a3ec5a48643. This expectation does not require the whole word to equal zero.

4. **Nonzero converted configuration gate.**  
   If that conversion is nonzero, F-1a3ec5a48643 is called. Its zero result permits configuration; its nonzero result skips only that configuration sequence. Descriptor submission and external writes still follow.

5. **No endian-dependent gate expectation.**  
   A memory-byte description alone is insufficient to predict either character gate without establishing word interpretation and conversion rules. For a nonzero word known under the target rules to convert to zero, acceptance follows the zero-conversion branch, not the whole-word-nonzero branch.

6. **Largest Mode.**  
   Mode `0xFFFFFFFF` selects the nonzero entry path, skips the zero-mode query, and produces notification argument zero when the relationship test succeeds.

7. **Related flags enable propagation.**  
   If Object lacks `0x10` but the captured related object has it, Object receives `0x20`. After F-6d16652f2d03, a nonnull current related pointer and nonzero converted gate at `0xA0` suppress the related callback, but not the final Object callback or returned-object flag update.

8. **Notification changes the relationship.**  
   If notification replaces Object’s related pointer, the fallback flag test still uses the previously captured pointer, provided its storage remains valid. The later related callback decision uses the pointer reread after F-6d16652f2d03.

## Uncertainties

The following remain **documented_only**:

- Semantic object types, flag meanings, and valid-state invariants.
- Actual machine load widths for the word-to-character expressions, endianness, character signedness, and conversion rules for values outside the character range.
- Exact address-adjustment overflow and pointer-validity behavior.
- Dependency error channels, nonlocal control flow, state preservation, and ownership effects.
- Accuracy of the nonreturning annotation.
- External entry identities and aliasing relationships.
- Complete temporary-record schemas and unassigned-byte contents.
- Record-retention requirements.
- The related callback’s implicit calling convention.
- Whether the final callback can validly return null.
- Concurrency and synchronization guarantees.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Established evidence comprises represented access expressions, offsets, assignment widths, constants, branch conditions, explicit call arguments, and sequencing.

The word-to-character gates are distinguished from physical byte loads; no endianness-based byte selection is asserted. Semantic interpretations, unknown callee effects, and ownership obligations are not promoted to established behavior.

No runtime tests, traces, machine-instruction inspection, or independent implementation inspection have been performed.
