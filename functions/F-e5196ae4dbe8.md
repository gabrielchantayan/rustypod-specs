<!-- Trusted provenance: F-e5196ae4dbe8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e5196ae4dbe8

## Purpose

F-e5196ae4dbe8 conditionally processes an object addressed by a 32-bit input. On an eligible path, it updates a status byte, performs a dependency-controlled iteration sequence, optionally issues a notification-like call, selects a byte from a shared table, publishes that byte, and conditionally invokes a shared indirect target.

Eligibility requires a selector byte equal to 2 and an object byte at offset `0x264` equal to zero. The selector’s location is determined by a displacement read from separate shared storage; it is not an established fixed object offset.

“Notification,” “wait,” and “completion” describe inferred intent only. The established contract consists of the represented accesses, branch conditions, call arguments, and sequencing. It does not establish a synchronization protocol or the domain meanings of the stored values.

## Interface

### Input and result

The sole explicit input is a 32-bit unsigned value used as an object base address. Object offsets in this document are byte offsets from that address.

The function has no defined return value and no explicit error-output parameter. Its observable results are direct byte writes, dependency calls, shared-storage accesses, and a possible indirect invocation.

There is no visible validation of null addresses, object size, alignment, pointer lifetime, table extent, or dependency-provided state. Safe execution requires every address actually accessed on the chosen path to remain valid at the time of access.

### Object storage

| Offset | Access width or representation | Established use |
|---|---|---|
| Shared displacement, value unavailable | One byte at the resulting address | Selector compared with 2. |
| `0x100` | One byte | Compared with 5 and, conditionally, 6. |
| `0x22c` | One byte | Compared repeatedly with a selected table byte. |
| `0x22d` | One byte | Receives the selected result byte. |
| `0x230` | Address passed to a dependency; later a 32-bit pointer slot | Supplies the pointer used by the final guard. |
| `0x238` | Address passed to two dependencies | Storage extent and representation are unknown. |
| `0x244` | One byte | Zero-tested; optionally written with 2. |
| `0x245` | One byte | Written with 2 after entry eligibility succeeds. |
| `0x264` | One byte | Entry exclusion check and later table-selection condition. |

The known fixed-offset byte accesses extend through `0x264`, inclusive. That observation does not establish the complete allocation size: the selector displacement and dependency accesses may require other storage.

Passing addresses at `0x230` and `0x238` establishes neither ownership transfer nor the size of the structures located there.

### Shared data

The function uses four distinct shared inputs:

- **Selector displacement:** read to form the selector-byte address from the object base. Its value is unavailable. Its declaration and exact storage width are not supplied.
- **Notification argument:** a 32-bit value read after every predicate call, including the terminating one.
- **Table base:** used after F-338d16f9ce72 returns to locate the selected entry.
- **Indirect-call target:** obtained from separate shared storage for the final conditional invocation.

The table has a 16-byte entry stride. On every normally reached active path, the saved selector is 2, so the selected entry is at table base plus `0x20`. This function accesses entry-relative offsets `0x0c`, `0x0e`, and `0x0f`, each as one byte.

### Widths and address calculations

The supplied representation uses a 32-bit unsigned object address, a 32-bit saved selector, a 32-bit predicate result, a 32-bit notification value, and a 32-bit pointer and pointed-to word for the final guard.

The selector is read through a character interpretation before conversion to the unsigned selector value. Character signedness is not established, but it cannot change which byte passes the equality test against 2.

The table stride calculation is harmless on the reachable path: 2 multiplied by 16 is 32. Safe address interpretation still requires object-relative and table-relative additions to reach valid intended storage. The fragment does not establish a portable rule for invalid or overflowing address calculations, or independently specify the arithmetic type of every shared address-bearing value.

## Behavior

### 1. Locate the selector and determine eligibility

The initial selector access depends on a value read from shared selector-displacement storage. That value supplies the displacement used with the object base before the selector byte can be accessed. The selector-byte read is therefore not the first represented memory input.

The function reads the resulting selector byte and retains its converted 32-bit unsigned value.

If the saved selector is not 2, the function returns. It does not read object offset `0x264`, perform direct writes, or call a dependency on this path.

If the saved selector is 2, the function reads the unsigned byte at `0x264`. A nonzero value causes an immediate return without direct writes or dependency calls.

Only selector 2 together with a zero entry-time byte at `0x264` reaches the remaining behavior. Neither the selector byte nor its shared displacement is visibly reread. Later mutations of either location do not replace the saved selector.

### 2. Publish the initial status

The function writes the one-byte value 2 to `0x245`, then calls F-18d91b9c7191 with the original object address.

The write precedes the call. The dependency’s behavior is unknown: it is not established to preserve that byte, other object storage, globals, or object lifetime.

### 3. Run the predicate-controlled sequence

At least one iteration occurs. Each iteration has the following represented order:

1. F-38c4e07bf57b receives the object address.
2. F-ac18df10cd76 receives the object address and supplies a 32-bit result.
3. The shared 32-bit notification argument is loaded.
4. The predicate result determines whether iteration ends.

A zero result ends this sequence. Any nonzero result causes F-cf4ff229c311 to receive the object address before the next iteration begins. Negative results, where represented by the signed result type, are nonzero and therefore continue the sequence.

The notification argument is loaded even on iterations that continue. Only the value loaded after the terminating predicate call remains relevant to the optional notification.

There is no visible iteration limit, timeout, cancellation condition, or error translation. If the predicate never produces zero, later stages are not reached. If any dependency does not return, subsequent behavior is likewise not reached.

### 4. Optionally issue the notification-like call

After iteration ends, the function reads `0x244`.

If that byte is nonzero, it skips the notification path without reading `0x100` for this decision.

If `0x244` is zero, it reads `0x100` and compares that byte with 5. Only if this comparison fails does it read `0x100` again and compare the second reading with 6.

If either comparison succeeds, the function writes 2 to `0x244`, then calls F-d073470bb2d6 with the notification argument saved after the terminating predicate call. The argument is not the object address.

With stable storage, the state condition is equivalent to a byte value of 5 or 6. Without stability, the two potential reads are distinct observations. For example, a first reading other than 5 followed by a second reading of 6 satisfies the condition. This does not establish that concurrent unsynchronized mutation is safe.

The notification call may change shared storage before later table processing. The saved notification argument has already been loaded and is not refreshed for that call.

### 5. Apply saved-selector range checks

The function contains checks rejecting a saved selector above the signed nonnegative range or above 3.

These checks cannot reject the ordinary active path because entry eligibility already fixed the retained selector at 2. They do not reread the selector byte or its displacement.

### 6. Select a table entry and result byte

The function calls F-338d16f9ce72 with the address at object offset `0x230`.

After that call returns, it obtains the shared table base and computes the selected entry address. For this path, the address is table base plus `0x20`. The computed address is retained through the subsequent comparison sequence; the function does not visibly reload the table base on each pass.

Each comparison reads the object byte at `0x22c` and the selected entry byte at `0x0c`, then tests whether they differ.

**The relative evaluation order of these two operand reads is not established by the supplied representation.** Neither object-first nor table-first ordering is part of this contract. Establishing such an order would require additional instruction-level evidence.

When the compared bytes differ, the function reads `0x264`:

- Nonzero ends the comparison sequence and selects the entry byte at `0x0f`.
- Zero causes a call to F-162a41bac6a7 with the address at `0x238`, followed by another comparison.

When the compared bytes are equal, the function separately reads `0x264`:

- Zero selects the entry byte at `0x0e`.
- Nonzero selects the entry byte at `0x0f`.

Thus equality alone does not guarantee selection of `0x0e`. It must be followed by a zero reading of `0x264`.

The selected result byte is read when its branch is taken. The table entry is not captured as a single snapshot. Across iterations, comparison-byte contents may change while the retained entry address remains fixed.

Continued mismatch together with repeated zero readings at `0x264` produces repeated F-162a41bac6a7 calls without a visible bound.

### 7. Publish the result and evaluate the final guard

The function writes the selected table byte to `0x22d`, then calls F-b8a325ddd21b with the address at `0x238`.

Only after that dependency returns does it read the pointer slot at `0x230`.

- A null pointer ends processing without reading pointed-to storage.
- A nonnull pointer causes a read of its first 32-bit word.
- A zero word ends processing without an indirect invocation.
- A nonzero word permits invocation of the shared indirect-call target.

The pointed-to word is a nonzero guard, not the established invocation address. The invocation target comes from separate shared storage.

No explicit arguments are supplied in the representation of the indirect invocation. Its identity, actual calling convention, implicit context, and return behavior remain unknown.

## Side effects

When their paths are reached, direct object writes occur in this order:

1. `0x245` receives 2.
2. Optionally, `0x244` receives 2.
3. `0x22d` receives the selected table byte.

There is no visible rollback or restoration. Dependencies may overwrite these locations or introduce other effects; the direct-write list is not a guarantee about final object contents.

No direct allocation or deallocation appears. This function does not visibly release the object or the pointer obtained from `0x230`, but dependency ownership and release behavior are unknown. Continued access after each dependency requires storage to remain valid.

There is no explicit error status, exception handling, or recovery path. Invalid storage and dependency failures have no established handling contract here.

The described sequence is source-level local ordering, subject to the explicitly unspecified operand-read order. It does not establish atomic operations, barriers, inter-thread visibility, or a valid concurrent-access protocol.

## Dependencies

| Opaque function ID | Invocation and explicit argument | Established contribution |
|---|---|---|
| F-18d91b9c7191 | Once after the `0x245` write; object address | Initial external operation. |
| F-38c4e07bf57b | Every predicate iteration; object address | Operation preceding the predicate. |
| F-ac18df10cd76 | Every predicate iteration; object address | Zero ends iteration; nonzero continues. |
| F-cf4ff229c311 | After every nonzero predicate result; object address | Operation before the next iteration. |
| F-d073470bb2d6 | Optional notification path; saved 32-bit shared value | External operation after the `0x244` write. |
| F-338d16f9ce72 | Before obtaining the table entry; address at `0x230` | External operation associated with that storage. |
| F-162a41bac6a7 | Each mismatch with `0x264` zero; address at `0x238` | Repeated comparison-path operation. |
| F-b8a325ddd21b | After the `0x22d` write; address at `0x238` | External operation before the pointer-slot read. |

The final indirect dependency has no supplied opaque ID. Its identity and behavior are documented_only.

No dependency is established to preserve globals, keep storage alive, acquire or release a lock, retain or release ownership, or return normally.

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests. Unless a scenario states otherwise, dependencies return normally, required storage remains valid, and relevant values remain stable.

### Selector located through shared storage

Given a shared displacement that identifies a selector byte other than 2, processing reads the displacement and the selector byte, then returns. It does not read `0x264`, write object bytes, or call dependencies.

This scenario must not be described as returning after a single initial selector-byte read: obtaining the displacement is also required.

### Entry exclusion

Given selector 2 and an entry-time nonzero byte at `0x264`, processing returns without writing `0x245` or calling dependencies.

### Predicate termination

Given predicate results nonzero, nonzero, then zero, F-38c4e07bf57b and F-ac18df10cd76 each run three times, and F-cf4ff229c311 runs twice. A shared notification value is loaded after each predicate call. Any notification receives the third loaded value.

A first result of zero instead produces one iteration and no F-cf4ff229c311 call.

### Notification conditions

Given `0x244` zero and a stable byte of 5 or 6 at `0x100`, processing writes 2 to `0x244` before calling F-d073470bb2d6.

Given nonzero `0x244`, that notification and its state-byte reads are skipped.

Given `0x244` zero, a first state reading of 4 and a second reading of 6 also satisfy the represented notification condition, provided such changing observations are otherwise valid.

### Matching comparison

Given equal bytes at object offset `0x22c` and entry offset `0x0c`, followed by a zero reading at `0x264`, entry byte `0x0e` is copied to `0x22d`. No F-162a41bac6a7 call is needed for that comparison.

This scenario permits either operand-read order. Equality followed by a nonzero `0x264` reading instead selects `0x0f`.

### Mismatch and continuation

Given unequal comparison bytes and nonzero `0x264`, entry byte `0x0f` is selected immediately without requiring a later match.

Given unequal bytes and zero `0x264`, F-162a41bac6a7 runs. If a later comparison matches and its following `0x264` reading is zero, entry byte `0x0e` is selected.

### Final post-call guard

If F-b8a325ddd21b leaves the pointer slot null, no pointed-to word is read and no indirect invocation occurs.

If it leaves a valid nonnull pointer to a zero first word, the word is read but no invocation occurs.

If it leaves a valid nonnull pointer to a nonzero first word, the separate shared indirect target is invoked. Neither an earlier pointer-slot value nor the guarded word alone identifies that target.

## Uncertainties

The following remain documented_only:

- The selector-displacement value, declared width, and complete object layout.
- Domain meanings of the state bytes, selector, and table fields.
- Relative instruction-level order of the two comparison-operand reads.
- Dependency mutation, ownership, lifetime, blocking, and failure contracts.
- Required structure extents and alignment for dependency arguments.
- Validity and mutability of shared storage, including the concurrency model.
- Exact arithmetic types of shared address-bearing values.
- Identity, calling convention, implicit arguments, and behavior of the indirect target.
- Behavior for invalid addresses or unavailable storage.

The apparent waiting and notification roles remain inferred intent, not additional guarantees.

## Evidence

Established static evidence is limited to the supplied function representation and opaque dependency mapping. It supports the shared-displacement-based selector access, access widths where represented, branch conditions, direct writes, explicit call arguments, and stated source-level sequencing.

The comparison expression establishes two operand reads and a comparison, but not their relative evaluation order. No instruction-level evidence has been supplied to resolve that order.

No runtime tests have been performed. Acceptance examples are conditional obligations derived from static evidence, not observed execution results.
