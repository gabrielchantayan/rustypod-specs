<!-- Trusted provenance: F-4e3d1a07efa8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4e3d1a07efa8

## Purpose

F-4e3d1a07efa8 initializes or updates a caller-supplied destination using one of three paths selected by comparisons against an embedded source object:

- A first recognized-marker path.
- A second recognized-marker path, considered only if the first comparison does not return zero.
- A fallback path taken when neither comparison returns zero.

The two recognized-marker paths obtain a value through dependencies, install a common destination tag, clear the destination’s second word, and pass the obtained value to another dependency. The fallback installs a different tag and, except for an exact source/destination alias, clears the destination’s second word and passes the embedded source’s second word to that same dependency.

This is an established structural description. Interpreting the tags as type metadata, the second words as owned pointers, or the recognized markers as semantic message categories would require evidence not supplied here.

## Interface

### Inputs

**Destination address**

The first argument identifies writable storage. Direct accesses use two consecutive 32-bit words:

| Relative byte offset | Direct use |
|---|---|
| `+0x00` | Receives one of two global 32-bit values. |
| `+0x04` | Receives zero on every recognized-marker path and on a non-aliasing fallback path. |

The direct accesses establish an eight-byte accessed region on paths that write both words. They do not establish the destination’s complete allocation size or the storage requirements of dependencies.

**Source base**

The second argument is represented as an integer-sized address value in the supplied listing. The function derives an embedded-object address by adding `0x4c`, or 76 bytes.

The visible source layout is:

| Offset from source base | Use |
|---|---|
| `+0x4c` | Address supplied as the first argument to marker comparisons; also the address used for the fallback alias check. |
| `+0x50` | A 32-bit value read and passed to F-151ab4bfe553 on the non-aliasing fallback path. |

The value at `+0x50` is therefore the second word relative to the embedded-object address.

The exact representation and total extent of the embedded object are unknown. In particular, the comparison calls do not establish that the embedded object is a raw character array.

### Output

On every visible path that returns normally, the function returns the original destination address.

It does not return a comparison result, a dependency result, or an explicit success/error status. The final dependency call’s return value, if any, is not used.

### Widths and preconditions

The explicit data loads and stores described here are 32 bits wide. Comparison results are tested only for equality or inequality with zero; their magnitude and sign otherwise have no visible effect.

Address formation uses an addition of 76 to the source-base representation. The supplied listing does not independently establish the machine-level overflow behavior of that address calculation. Valid use requires that the derived address identify the intended accessible storage; this contract does not promise address validation or safe wraparound.

The destination must be writable for the direct stores taken by the selected path and suitable for F-151ab4bfe553 when that dependency is called. The source must be suitable for the comparison dependency. The non-aliasing fallback additionally requires a readable 32-bit value at source offset `+0x50`.

No visible null, alignment, bounds, allocation, or object-validity checks precede these operations.

## Behavior

### Marker selection

The function first calls F-3805a25164a0 with:

1. The embedded-object address at source base plus `0x4c`.
2. A fixed first marker reference.

The supplied symbol annotation suggests a marker associated with `<<<note>>>`. The actual marker bytes and the comparison algorithm are not available. The established branch condition is a zero result, not independently verified textual equality.

If this first call returns zero, the function selects the first recognized-marker path. It does not perform the second comparison.

If the first result is nonzero, the function calls F-3805a25164a0 again with the same derived address and a different fixed marker reference. Its annotation suggests `<<<instructions_text>>>`.

A zero second result selects the second recognized-marker path. A nonzero second result selects the fallback.

Both comparisons occur before this function performs any direct destination store. This ordering does not imply that the comparisons are side-effect-free.

### First recognized-marker path

After the first comparison returns zero:

1. F-9d0ecc5f6adf is called without explicit arguments.
2. Its 32-bit result is retained.
3. A path-specific 32-bit global value is read.
4. F-c0b020cd3882 is called with the retained result and that global value.
5. Its 32-bit result is retained for the destination update.
6. A common recognized-path tag is read from global storage and written to the destination’s first word.
7. Zero is written to the destination’s second word.
8. F-151ab4bfe553 is called with the destination address and the result from F-c0b020cd3882.
9. The original destination address is returned if execution returns normally.

The path-specific global is read after F-9d0ecc5f6adf returns. The common tag is read after F-c0b020cd3882 returns. Consequently, this contract does not treat either global as a snapshot taken before those calls.

There is no alias exemption on this path.

### Second recognized-marker path

This path requires a nonzero first comparison result and a zero second result.

Its operation is the same as the first recognized-marker path, except that it reads a different path-specific global value after F-9d0ecc5f6adf returns. That value becomes the second argument to F-c0b020cd3882.

Both recognized-marker paths use the same common tag for the destination’s first word. Their observable differences include the extra comparison and the different global supplied to F-c0b020cd3882.

### Fallback path

This path requires both comparison results to be nonzero.

The function first reads the fallback tag from global storage and writes it to the destination’s first word. It then tests whether the destination address exactly equals the embedded-object address.

**Exact alias**

If the destination equals source base plus `0x4c`:

- The first-word tag store has already occurred.
- This function does not clear the second word.
- This function does not read the source word at `+0x50`.
- F-151ab4bfe553 is not called.
- The destination address is returned.

This is not a complete no-op or an unqualified self-assignment fast path: the first word is still overwritten.

**Different addresses**

If the addresses differ:

1. The destination’s second word is set to zero.
2. The 32-bit value at source base plus `0x50` is read.
3. F-151ab4bfe553 is called with the destination and that value.
4. The destination address is returned if execution returns normally.

The source-word read occurs after both direct destination stores. Its value is therefore the value present at that later time, not necessarily its value on entry.

### Overlapping storage

The fallback alias check excludes only exact equality with the embedded-object address. It does not establish disjointness.

For example, if the destination starts at source base plus `0x50`, the fallback first-word store targets the very source word that will subsequently be read. Under a valid ordinary-memory interpretation, the later read observes the fallback tag written there, absent another intervening change.

Other overlaps may similarly affect subsequently accessed storage. The visible implementation provides no general overlap protection.

On recognized-marker paths, even exact aliasing does not suppress the two destination stores or the call to F-151ab4bfe553.

## Side effects

### Direct writes

The function directly performs these destination writes:

- **Recognized-marker paths:** common tag at offset zero, followed by zero at offset four.
- **Non-aliasing fallback:** fallback tag at offset zero, followed by zero at offset four.
- **Exact-alias fallback:** fallback tag at offset zero only.

These describe direct writes and their ordering, not guaranteed final contents. F-151ab4bfe553 executes after the two-word update on paths that call it and may modify the destination further.

### Dependency effects

The supplied evidence does not establish which dependencies allocate, release storage, mutate globals, modify arguments, or access external state. Their effects must not be silently excluded from the contract.

In particular:

- Comparison calls may affect state observed by subsequent comparisons or global reads.
- F-9d0ecc5f6adf may affect the global read immediately following it.
- F-c0b020cd3882 may affect the common tag read immediately following it.
- F-151ab4bfe553 may change the destination after its direct initialization.

### Ownership and failures

No explicit allocation, deallocation, retain, release, or ownership-transfer operation is identifiable in the function body alone. Passing a 32-bit value to a dependency does not establish whether that value denotes memory or transfers ownership.

There is no visible cleanup of the destination’s previous second-word value before it is overwritten. Whether cleanup is necessary or occurs elsewhere is unknown.

There is no explicit error-handling branch. Zero dependency results are not visibly rejected: a zero result from F-c0b020cd3882, or a zero fallback source word, is passed onward to F-151ab4bfe553.

The function offers no demonstrated rollback behavior if a dependency fails, faults, or does not return. Direct writes already performed before such an event are not visibly undone.

## Dependencies

| Opaque function ID | Established use | Not established |
|---|---|---|
| F-3805a25164a0 | Receives the embedded-object address and a marker reference. Zero selects the associated recognized path. Called once or twice. | Comparison algorithm, encoding, exact marker bytes, mutation behavior, error behavior. |
| F-9d0ecc5f6adf | Called once without explicit arguments on either recognized-marker path. Supplies a 32-bit value to F-c0b020cd3882. | Meaning of its result, global effects, allocation or ownership behavior. |
| F-c0b020cd3882 | Receives the preceding result and a path-specific global value. Supplies a 32-bit value to F-151ab4bfe553. | Whether it transforms, resolves, constructs, or looks up anything; result lifetime and ownership. |
| F-151ab4bfe553 | Receives the destination and a selected 32-bit value after the direct two-word update. Skipped only on exact-alias fallback. | Complete destination layout, final contents, ownership effects, failure behavior. |

The function also reads two path-specific globals and two tag globals. Their values, mutability, and semantic types are not established. Different storage locations do not prove different stored values.

## Acceptance examples

These examples are conditional contract scenarios, not executed tests. They assume valid storage and normally returning dependencies.

### First comparison returns zero

Given a zero result from the first comparison:

- Exactly one comparison is made.
- F-9d0ecc5f6adf and F-c0b020cd3882 are each called once.
- The first path-specific global is read after F-9d0ecc5f6adf returns.
- The common tag is stored, followed by a zero second word.
- F-151ab4bfe553 receives the destination and F-c0b020cd3882’s result.
- The original destination address is returned.

This remains the selected behavior if the destination exactly aliases the embedded-object address.

### Only the second comparison returns zero

Given a nonzero first result and a zero second result:

- Both comparisons occur in order.
- The second path-specific global, rather than the first, is passed to F-c0b020cd3882.
- The destination update and final dependency call follow the recognized-marker sequence.

### Neither comparison returns zero, with disjoint storage

Given two nonzero results and a destination disjoint from the source:

- Neither F-9d0ecc5f6adf nor F-c0b020cd3882 is called.
- The fallback tag and zero are stored in that order.
- The source word at `+0x50` is then read and passed to F-151ab4bfe553.

### Neither comparison returns zero, with exact alias

Given two nonzero results and destination equal to source base plus `0x4c`:

- The embedded object’s first word receives the fallback tag.
- No direct second-word store or source-word read follows.
- F-151ab4bfe553 is skipped.
- The aliasing destination address is returned.

Any claim that the second word remains unchanged must separately account for possible effects of the earlier comparison calls.

### Neither comparison returns zero, with a specific overlap

Given two nonzero results and destination equal to source base plus `0x50`, with sufficient valid writable storage:

- Exact alias comparison is false.
- The fallback tag overwrites the source word at `+0x50`.
- Zero is written at destination offset four.
- The subsequent source-word read supplies the fallback tag to F-151ab4bfe553, absent intervening mutation.

This example distinguishes the visible behavior from a copy that snapshots the source before writing.

## Uncertainties

The following are **documented_only**:

- Exact marker contents and comparison semantics.
- Complete source and destination object layouts.
- Meaning and actual values of the global words.
- Dependency mutation, allocation, ownership, and failure contracts.
- Whether overlapping objects are valid for the surrounding application.
- Exact machine address-overflow and calling-convention details.
- Final destination contents after F-151ab4bfe553 returns.

**Inferred intent:** the two marker comparisons likely distinguish specially handled content, and the two tag globals likely distinguish destination forms. Neither interpretation is required for the established control flow or memory-access contract.

## Evidence

The supplied static listing establishes the branch conditions, explicit call order, offsets, 32-bit data accesses, direct write order, exact-alias exception, and returned destination address.

The listing includes an analysis warning concerning restarted processing and stack-related elimination. No independent machine-code validation or dependency implementation was supplied. This contract therefore records the visible static behavior without asserting a fully recovered type system or application meaning.

No runtime tests have been performed. The acceptance examples specify conditional expected observations; they are not execution evidence.
