<!-- Trusted provenance: F-99aeb8eabebe; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-99aeb8eabebe

## Purpose

F-99aeb8eabebe selects the first table entry matching two unsigned 16-bit request keys, accesses an initial data region, obtains a secondary region’s location and length, and conditionally decodes metadata, requests destination storage, and invokes a converter.

The established behavior consists of field accesses, selection rules, dependency calls, arithmetic, destination writes, and status propagation. The dimensions, row-stride calculation, and scaled metrics suggest packed one-bit image or glyph processing. That interpretation is inferred intent, not an established format guarantee.

The operation is not transactional. It publishes destination fields before later operations can fail and performs no explicit rollback. Dependencies may change destination fields, exposed local storage, or shared state; their preservation and ownership guarantees are unknown.

## Interface

### Arguments and result

The routine accepts three 32-bit arguments:

| Argument | Established use |
|---|---|
| Destination address | Identifies writable storage; its field at `+0x04` supplies the context address |
| Request address | Supplies two selection keys and a value copied into the destination |
| Indexed-record selector | Used in computing an address within an array of 16-byte records |

The result is a 32-bit status:

- `6` if table traversal ends without a match.
- `6` if the secondary-length local is zero when tested after the first F-9787e15c337d call returns.
- Otherwise, the status from the first failing checked dependency, or the final converter’s status when conversion is reached.

F-7a0c4836ae1b and F-9787e15c337d do not contribute checked return statuses. F-8f41f510c86c supplies values, not checked statuses. Status `6` has no further established meaning.

### Referenced layouts

Offsets are in bytes. Addresses and scalar accesses are 32 bits except where explicitly identified otherwise.

The context is obtained from destination `+0x04`:

| Context offset | Use |
|---|---|
| `+0x68` | Handle passed to region-access dependencies |
| `+0xBC` | Base for secondary-region positioning |
| `+0xD0` | Flags; only mask `0x02` reaches the converter |
| `+0x12C` | Comparison operand |
| `+0x130` | Comparison operand and transformation argument |
| `+0x17C` | Unsigned table-entry count |
| `+0x184` | Table address |
| `+0x1A0` | Indexed-record array address |
| `+0x1B0` | Base for initial-region positioning |

Request fields:

| Offset | Width | Use |
|---|---:|---|
| `+0x0C` | Unsigned 16 bits | First selection key; later zero-extended and shifted left by eight |
| `+0x0E` | Unsigned 16 bits | Second selection key |
| `+0x20` | 32 bits | Read during metadata publication and copied to destination `+0x34` |

Each table entry occupies 36 bytes:

| Offset | Use |
|---|---|
| `+0x00` | First key, compared as a full unsigned 32-bit value |
| `+0x04` | Second key, compared as a full unsigned 32-bit value |
| `+0x08` | Flags |
| `+0x18` | Initial-region offset |
| `+0x1C` | Initial-region item count |

The indexed-record address is its array base plus 16 times the selector. The routine uses that record’s first and second 32-bit words. No selector bounds check is visible.

Handle fields `+0x20` and `+0x24` are passed to dependencies. Their precise types and ownership semantics are unknown.

### Preconditions

Every address reached on the chosen path must support the corresponding read or write. No null checks, object-size checks, selector validation, or region-bound checks are visible.

The caller and dependencies must provide usable values for consumed output locations. In particular, the initial interpreter’s result is unchecked even though its outputs are subsequently consumed.

No visible validation rejects negative dimensions, negative secondary lengths, overflowed sizes, or unsuitable allocation requests. Fault behavior and downstream handling of such inputs are documented_only.

## Behavior

### Selection

Before traversal, the routine obtains the context, the handle, the computed indexed-record address, and the table address.

Traversal starts at index zero. Before inspecting each candidate, the routine reads the current unsigned count at context `+0x17C`. If the index is at least that count, it returns `6`.

For each candidate, the first request key is read and zero-extended, then compared with the entry’s first 32-bit key. Only if that comparison succeeds are the second request key and second entry key read and compared. The first entry matching both keys is selected. A mismatch advances the entry address by 36 bytes and increments the index.

Consequently:

- Entries with nonzero upper 16 bits in either key cannot match the corresponding request key.
- Duplicate matches select the earliest entry.
- Failure while processing a selected entry does not restart traversal or try another match.
- The entry count and request keys are read during traversal, not established immutable snapshots.

### Initial-region access

The selected entry’s flags are read to derive an item size. The size is four bytes plus one byte for each set bit among `0x01`, `0x02`, and `0x04`; it therefore ranges from four through seven bytes. Higher bits do not affect this size.

F-e40a3ceddab2 receives the retained handle and the sum of context `+0x1B0` and entry `+0x18`. A nonzero status is returned immediately.

After successful positioning, F-d036abe1e536 receives the handle and the derived item size multiplied by the entry count read at this stage. A nonzero status is returned immediately. Neither failure path includes an explicit F-9787e15c337d call.

After successful acquisition, F-7a0c4836ae1b receives:

- Current handle fields `+0x20` and `+0x24`.
- The entry count and full entry flags, read for this call.
- The indexed record’s first word.
- Writable addresses for a secondary-offset local and secondary-length local.

Its return value is ignored. The full flags and count supplied here need not equal earlier reads if intervening dependency effects or aliasing changed them.

F-9787e15c337d is then called with the handle. Only after it returns is the secondary-length local tested. Zero causes return `6`; any nonzero 32-bit representation proceeds.

This decision concerns the local’s value at the test, not necessarily the value originally written by F-7a0c4836ae1b. That dependency received its address, and no preservation guarantee across finalization is supplied.

### Preliminary publication and transformation

For a nonzero length at that decision point, the routine reads the indexed record’s second word. It compares context `+0x130` with context `+0x12C`.

If equal, the record word is selected for publication. If unequal, F-8f41f510c86c is called, and its result replaces that value. This conditional call has no explicit arguments in the supplied representation; its actual argument convention and implicit inputs remain unresolved.

The selected value is written to destination `+0x38`.

F-8f41f510c86c is then called with three explicit arguments:

1. The current first request key, zero-extended and shifted left by eight.
2. A fresh read of the indexed record’s second word.
3. The current context field at `+0x130`.

Its result is retained as an input to metadata decoding, not checked as a status.

### Secondary-region access and decoding

F-e40a3ceddab2 receives the handle and context `+0xBC` plus the secondary-offset local read for this call. A nonzero status returns immediately.

F-d036abe1e536 then receives the handle and the secondary-length local read at this later point. A nonzero status also returns immediately. Neither failure path explicitly finalizes this secondary attempt.

The earlier nonzero test does not establish an immutable length for this acquisition. Calls intervene, and the initial interpreter was given the local’s address.

After successful secondary acquisition, the current handle field at `+0x20` is copied into a cursor local. F-bfb4440a667e receives:

- The cursor local’s writable address.
- The current handle field at `+0x24`.
- The transformation result.
- Six writable output addresses.

For describing subsequent uses, these outputs are called horizontal origin, vertical origin, width, height, auxiliary metric, and conversion parameter. These names do not establish external semantic types.

If decoding returns nonzero, the routine skips metadata publication, allocation, and conversion. It calls F-9787e15c337d and returns the saved decoding status.

### Metadata publication

On successful decoding, the routine performs the following destination stores in order. Output names denote values read from the decoder-exposed locals at the relevant use, not guaranteed immutable snapshots.

| Order | Destination offset | Value |
|---:|---|---|
| 1 | `+0x48` | Current 32-bit value read from global storage at this point |
| 2 | `+0x50` | Width |
| 3 | `+0x4C` | Height |
| 4 | `+0x54` | Arithmetic right shift by three of width plus seven |
| 5 | `+0x5E` | One-byte value `1` |
| 6 | `+0x18` | Width multiplied by 64 |
| 7 | `+0x1C` | Height shifted left by six |
| 8 | `+0x20` | Horizontal origin shifted left by six |
| 9 | `+0x24` | Vertical origin shifted left by six |
| 10 | `+0x2C` | Width multiplied by negative 64, then arithmetic-right-shifted by one |
| 11 | `+0x30` | Zero |
| 12 | `+0x28` | Auxiliary metric arithmetic-right-shifted by two, plus 32, then masked with `0xFFFFFFC0` |
| 13 | `+0x34` | Current request field at `+0x20` |
| 14 | `+0x68` | Vertical origin plus height |
| 15 | `+0x64` | Horizontal origin |

The row stride is calculated after the width store and before the height store; its destination store follows the height store.

The global-storage read occurs after successful decoding and immediately supplies the first listed store. Its value, constancy, and preservation across earlier dependencies are not established. It is not a known fixed constant.

The `+0x5E` store affects one byte only. Other listed destination stores are 32 bits. The request-value read occurs after the preceding destination writes, which matters if the objects alias.

### Allocation, conversion, and completion

F-a1215dd1ebde receives the destination address and height multiplied by the previously computed row stride.

A nonzero allocation status skips conversion. The routine performs no explicit metadata rollback, but the allocator itself may have modified the destination before returning.

If allocation succeeds, F-f6392b77a034 receives:

- The cursor local’s value read after allocation returns.
- A fresh read of handle `+0x24`.
- The conversion-parameter local’s value read after allocation returns.
- Current context `+0xD0` masked with `0x02`, producing zero or two.
- The address of destination `+0x4C`.

The cursor and conversion-parameter locals were exposed by writable address to the decoder. Their converter inputs are not guaranteed snapshots of the values present immediately after decoding; preservation across allocation is unknown.

After decoding, allocation, or conversion supplies the status for this completion path, F-9787e15c337d is called. Its return value is ignored, and the saved status is returned.

### Arithmetic boundaries

The visible scalar model is 32-bit. No wider intermediate or overflow check is established for address calculations, item-count multiplication, dimension scaling, stride calculation, allocation sizing, or origin-plus-height addition.

Arithmetic right shifts are used for the signed metric expressions. The auxiliary-metric addition uses an unsigned 32-bit constant before masking.

For ordinary nonnegative dimensions without overflow, stride is width rounded up to whole bytes, allocation size is height times stride, and destination `+0x2C` is negative 32 times width.

The evidence does not establish general overflow rejection, unbounded mathematical arithmetic, or a portable overflow policy for all signed expressions.

## Side effects

Direct destination writes are limited to the stores described above. Dependencies may additionally affect the destination, handle, context, referenced storage, exposed locals, and shared state.

The direct-write boundaries are:

- No direct destination store precedes the post-finalization nonzero-length decision.
- Destination `+0x38` is written before secondary positioning, acquisition, or decoding can fail.
- Metadata publication precedes allocation.
- No explicit rollback or destination-storage release follows allocation or conversion failure.

These boundaries describe this routine’s actions, not preservation guarantees. Dependency effects may change the same storage before return.

F-9787e15c337d is a finalization call event. Its presence does not prove that memory is freed, ownership is transferred, or a handle remains usable. No explicit finalization occurs after failed positioning or failed acquisition.

No synchronization is visible. Aliasing and retained output addresses prevent treating all input fields and decoder outputs as immutable.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-e40a3ceddab2 | Handle and computed position; nonzero status returned immediately |
| F-d036abe1e536 | Handle and requested size; nonzero status returned immediately |
| F-7a0c4836ae1b | Initial-region inputs and writable secondary-offset/length outputs; return ignored |
| F-9787e15c337d | Finalization at the specified call sites; return ignored |
| F-8f41f510c86c | Conditional publication value and later transformation value; not status-checked |
| F-bfb4440a667e | Writable cursor and six outputs; zero enables metadata publication |
| F-a1215dd1ebde | Destination and computed size; zero enables conversion |
| F-f6392b77a034 | Current conversion inputs and destination subobject; status propagated |

Dependency validation, retained-reference behavior, global preservation, memory ownership, and failure-side effects are documented_only.

## Acceptance examples

These are conditional static examples, not executed tests. Unless otherwise stated, addresses are valid, relevant values remain stable between their reads, dependencies provide usable outputs, and arithmetic does not overflow.

1. **Empty table.** If the first count read is zero, return `6` without dependency calls or direct destination writes. Context, handle, indexed-record address, and table-address resolution have already occurred.

2. **Duplicate entries.** If two entries match both keys, select the earlier entry. A later failure does not cause fallback to the second match.

3. **Unsigned keys.** Request key `65535` matches entry key `0x0000FFFF`, not `0xFFFFFFFF`.

4. **Flag-derived size.** Stable flags `0x05` and count `3` request 18 bytes for the initial region. Flags `0x85` yield the same size, but the interpreter receives the full flags read for its call.

5. **Initial positioning failure.** If the first F-e40a3ceddab2 returns `9`, return `9` without initial acquisition, interpretation, or explicit finalization.

6. **Zero length after finalization.** If the secondary-length local is zero when tested after F-9787e15c337d returns, return `6` without directly writing destination `+0x38`. An interpreter output of zero alone is insufficient to establish this outcome.

7. **Secondary positioning failure.** If the secondary-length local is nonzero at that post-finalization decision and secondary positioning returns `11`, return `11`. This routine has already written `+0x38` but has not published decoded metadata. Dependency effects may independently alter destination storage.

8. **Ordinary metadata.** With width `9`, height `3`, horizontal origin `2`, vertical origin `−1`, and auxiliary metric `128`, publish stride `2`, scaled dimensions `576` and `192`, scaled origins `128` and `−64`, negative half-width field `−288`, aligned auxiliary field `64`, and origin-plus-height field `2`. Request allocation of six bytes.

9. **Allocation failure.** With the preceding values available during publication, if allocation returns `12`, skip conversion, call finalization, and return `12`. Metadata was published before allocation and no explicit rollback occurs. Its final contents are not guaranteed: allocation and finalization may modify them.

10. **Current global value.** If the global contains a particular 32-bit value at the publication read, that value supplies the write to destination `+0x48`, regardless of its contents before decoding.

11. **Converter input timing.** If allocation succeeds, converter inputs use the cursor and conversion-parameter values present when read after allocation. Equality to their immediate post-decoder values requires an additional preservation premise.

12. **Conversion failure.** If conversion returns `13`, finalization is called and the saved status `13` is returned. No explicit destination rollback is established.

## Uncertainties

- Application-level meanings of keys, records, dimensions, and metrics remain inferred.
- The global value copied to destination `+0x48` has unknown contents and mutability.
- The conditional F-8f41f510c86c call’s argument convention is unresolved.
- Dependency ownership, preservation, retained references, and failure effects are documented_only.
- Invalid selectors, negative values, address overflow, and arithmetic overflow lack visible validation and an established rejection policy.
- Output validity after the unchecked initial interpreter is not independently established.
- No total ordering between separate argument reads within a single dependency call is established beyond the shown call boundaries.
- No runtime tests have been performed.

## Evidence

Established static evidence is the supplied routine’s access widths, offsets, traversal, branches, dependency calls, status checks, arithmetic expressions, ordered stores, and return paths.

Packed-image processing and descriptive metric names are inferred intent only. No dependency implementation, external format contract, runtime trace, or independent calling-convention evidence was supplied. Unresolved behavior remains documented_only rather than being filled with assumed preservation, ownership, or validation guarantees.
