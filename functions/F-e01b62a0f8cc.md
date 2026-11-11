<!-- Trusted provenance: F-e01b62a0f8cc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e01b62a0f8cc

## Purpose

F-e01b62a0f8cc updates an existing state object from configuration storage and additional incoming context. It performs ordered field copies, selector-dependent byte updates, partial validation, optional-region processing through a dependency, derived-value calculations, an external counter increment, and a final dependency call.

This is not an atomic initializer. Rejection leaves earlier writes in place. Some destination fields retain their previous contents on particular paths.

**Established static evidence:** access widths, offsets, branch conditions, explicit read/write ordering, dependency arguments, and return bit patterns.

**Inferred intent:** terms such as “configuration,” “mode,” “optional region,” and “counter” describe apparent roles, not established application semantics.

**Unknown:** the complete entry convention, dependency contracts, external-storage identities, and platform behavior for certain signed arithmetic operations.

## Interface

### Explicit arguments and result

| Item | Width | Behavior |
|---|---:|---|
| First explicit argument | 32 bits | No direct use is visible. |
| Selector | 32 bits | Selects an early group of byte writes. Values outside 0–7 skip that group without independently causing rejection. |
| Result | 32 bits | `0x00000000` on normal completion; `0xFFFFFFFF` on a visible rejection path. |

The failure result is an exact bit pattern. Its interpretation as signed −1 is a caller convention.

### Additional incoming context

The supplied representation requires four values beyond the explicit arguments:

- **Destination base**, identifying an existing writable object.
- **Configuration base**, identifying readable configuration storage.
- **Baseline word**, a 32-bit value used in word and byte assignments.
- **Marker word**, another 32-bit value used in word and byte assignments.

This contract uses:

- **D+offset** for destination-relative storage.
- **S+offset** for configuration-relative storage.
- **Baseline byte** for the low eight bits of the baseline word.
- **Marker byte** for the low eight bits of the marker word.

Neither incoming word is established to be a Boolean, zero, or one.

### External storage

Four additional storage resources are accessed:

- **K1 storage:** supplies a 32-bit value only when configuration kind 1 is selected.
- **K7 storage:** supplies a 32-bit value only when configuration kind 7 is selected.
- **Counter-pointer storage:** supplies the pointer used for the completion-stage increment.
- **T storage:** supplies the first argument of the final dependency call.

These are storage reads at the points described below, not immutable constants or values necessarily captured at entry. Their contents may reflect earlier writes, aliasing, or dependency effects.

### Storage and execution prerequisites

Direct destination accesses extend through offset `0x8D`. Direct configuration accesses extend through the word at `0x50`, ending at `0x53`. The dependency receiving S+`0x40` may access additional storage; its extent is unknown.

The caller must provide valid storage for all reached accesses and satisfy applicable alignment and mapping requirements. No bounds checks, null checks, synchronization, or rollback are visible.

Source, destination, and external storage are not established to be disjoint. Earlier writes can therefore affect later reads. A whole-input snapshot is not equivalent to the established access sequence.

## Behavior

### Initial updates

The following 32-bit copies occur in order, with each source read associated with its corresponding destination write:

| Source | Destination |
|---|---|
| S+`0x04` | D+`0x50` |
| S+`0x18` | D+`0x64` |
| S+`0x14` | D+`0x54` |
| S+`0x28` | D+`0x74` |

Next, the routine writes:

1. Baseline word to D+`0x88`.
2. Baseline word to D+`0x84`.
3. Marker byte to D+`0x8D`.

All these writes precede every visible rejection.

### Selector-dependent byte updates

| Selector | Ordered writes |
|---|---|
| 0 or 7 | Baseline byte to D+`0x42`, then D+`0x43`; four individual byte copies from S+`0x30`–`0x33` to D+`0x5D`–`0x60`, in ascending order. |
| 1, 2, or 5 | Baseline byte to D+`0x42`, then D+`0x43`. |
| 3 | Baseline byte to D+`0x42`, then D+`0x43`; four individual byte copies from S+`0x2C`–`0x2F` to D+`0x59`–`0x5C`, in ascending order. |
| 4 | Byte copy from S+`0x10` to D+`0x42`, then S+`0x11` to D+`0x43`. |
| 6 or any other value | No writes in this group. |

Fields omitted by the selected branch are not reset by this stage.

### Range validation

The routine reads S+`0x08` as an unsigned 32-bit value, then writes the baseline byte to D+`0x40`.

- If the captured value is at most 8, this stage does not write D+`0x3C`.
- If it exceeds 8, S+`0x08` is read again:
  - A second value greater than 16 causes immediate rejection.
  - Otherwise, the marker word is written to D+`0x3C`.

For stable storage, values 0–8 preserve D+`0x3C`, values 9–16 assign the marker word, and values above 16 reject.

The second read is not constrained to remain above 8. For example, if the first value exceeds 8 and the second is 0, the marker-word write still occurs. The intervening D+`0x40` write can matter if storage aliases.

### Mode validation and initialization

The routine examines the 32-bit field at S+`0x0C`.

- If its first read equals 1, it writes the marker byte to D+`0x41`.
- Otherwise, it writes the baseline byte to D+`0x41`, then reads S+`0x0C` again:
  - A value other than 2 causes immediate rejection.
  - A value of 2 causes the existing signed 32-bit value at D+`0x3C` to be incremented and stored there.

With stable inputs, mode 1 does not increment D+`0x3C`; mode 2 does. If the range was at most 8, mode 2 increments the pre-existing destination value. Signed-overflow behavior is not established.

After successful mode validation, these writes occur in order:

| Destination | Value |
|---|---|
| D+`0x14`, word | Fresh 32-bit read from S+`0x0C` |
| D+`0x08`, word | Baseline word |
| D+`0x0C`, word | Baseline word |
| D+`0x05`, byte | Baseline byte |
| D+`0x04`, byte | Baseline byte |
| D+`0x4C`, word | Baseline word |
| D+`0x45`, byte | Marker byte |
| D+`0x46`, byte | Baseline byte |

The value stored at D+`0x14` is not necessarily the value that passed validation: it comes from another read.

### Configuration-kind dispatch and external-value reads

The routine next reads the byte at S+`0x00`. This configuration-kind byte is independent of the explicit selector.

| Configuration kind | Action |
|---|---|
| 0, 3, or 5 | Write the baseline word to D+`0x78`. |
| 1 | Read 32 bits from K1 storage, then write that value to D+`0x78`. |
| 7 | Read 32 bits from K7 storage, then write that value to D+`0x78`. |
| Any other byte value | Reject immediately. |

K1 storage is read only in the kind-1 branch; K7 storage is read only in the kind-7 branch. Each selected read occurs after all preceding destination updates and before its D+`0x78` write and the first dependency call.

Neither value is established to be immutable, nonzero, or distinct from the other. If its storage aliases a previously modified location, the selected read observes the value present at this stage.

A rejection here retains preceding writes and calls neither dependency.

### Optional-region dependency

For every accepted configuration kind, the routine calls **F-6d8c85350b66** with S+`0x40`. Its result is distinguished only as zero or nonzero.

No preservation, ownership, validation, or memory-lifetime contract is supplied for this dependency.

#### Zero result

The routine writes, in order:

1. Baseline byte to D+`0x28`.
2. Baseline word to D+`0x2C`.
3. Baseline word to D+`0x30`.
4. Baseline word to D+`0x34`.
5. Baseline word to D+`0x38`.
6. Baseline word to D+`0x7C`.
7. Baseline word to D+`0x80`.

There is no direct D+`0x78` reset in this branch. A zero dependency result is not function-level rejection.

#### Nonzero result

The routine first writes the marker byte to D+`0x28`, then performs these ordered 32-bit copies:

| Source | Destination |
|---|---|
| S+`0x40` | D+`0x2C` |
| S+`0x44` | D+`0x30` |
| S+`0x48` | D+`0x34` |
| S+`0x50` | D+`0x38` |

All four source reads occur after the dependency returns.

The last three source values are retained for a subsequent conditional transformation. After the copies, the routine reads the byte at D+`0x58`:

- If it equals 7, each retained value is shifted left by one and stored, in order, at D+`0x30`, D+`0x34`, and D+`0x38`.
- Otherwise, those fields retain their copied values at this stage.

These shifts use signed 32-bit operands in the supplied representation. Behavior for negative operands or results outside the signed representable range remains unresolved. D+`0x2C` is not shifted.

The explicit selector does not directly select this transformation. This body does not directly initialize D+`0x58`.

### Derived values and arithmetic widths

Still within the nonzero-result branch:

1. D+`0x78` is read as an unsigned 32-bit value.
2. D+`0x30` is read as a signed 32-bit value.
3. Their sum is stored at D+`0x7C`.

**This addition is established unsigned 32-bit arithmetic.** The signed operand is converted to unsigned 32-bit form because the other operand is unsigned 32-bit. The stored sum is reduced modulo \(2^{32}\). Overflow does not make this particular addition unresolved, and no saturation or rejection is visible.

The routine then compares the captured unsigned D+`0x78` value against an unsigned 32-bit read of D+`0x34`.

- If D+`0x34` is larger, D+`0x34` is read again and the captured D+`0x78` value is subtracted using unsigned 32-bit arithmetic.
- Otherwise, the selected result is zero.

The result is stored at D+`0x80`.

For stable destination storage, this is the unsigned lower-clamped difference between D+`0x34` and the captured D+`0x78` value. The comparison and subtraction use separate D+`0x34` reads. If that field changes between them, the subtraction can wrap modulo \(2^{32}\); the comparison does not validate the second read.

The earlier D+`0x78` assignment does not guarantee the value used here: F-6d8c85350b66 executes between that assignment and this read.

### Completion

Both dependency-result branches converge on this ordered sequence:

1. Read the counter pointer from external counter-pointer storage.
2. Write the baseline byte to D+`0x8C`.
3. Read the signed 32-bit value through the captured counter pointer, increment it, and write it back.
4. Read T from external T storage while preparing the final call.
5. Call **F-266a29fc8dc5** with the read T value and a second argument of zero.
6. Return `0x00000000` if that call returns normally.

The pointer is acquired before the D+`0x8C` write, but the counter value is read afterward. T is read after the counter increment, not at entry or before the first dependency.

These timings matter if external storage aliases modified memory or is changed by F-6d8c85350b66. No counter overflow check is visible; signed-overflow behavior remains unresolved.

The final dependency’s result is ignored. Its effects may change destination storage, external storage, or memory ownership; no preservation guarantee is established.

## Side effects

Direct effects comprise destination writes and a read-modify-write of one externally addressed counter. Up to two dependencies are invoked.

No direct allocation, release, ownership transfer, rollback, or synchronization operation is visible. This does not constrain either dependency’s ownership behavior.

Configuration storage is directly read rather than directly written. Aliasing and dependency effects can nevertheless change its contents.

Rejection is a partial-update outcome:

- **Range rejection:** initial writes, selector writes, and the D+`0x40` write have occurred.
- **Mode rejection:** the preceding effects and baseline-byte write to D+`0x41` have occurred, including any range-stage D+`0x3C` assignment.
- **Kind rejection:** all successful mode-stage initialization has also occurred.

No visible rejection path calls either dependency, acquires the completion counter pointer, or increments the counter.

No atomicity guarantee is established for any access or for the counter increment.

## Dependencies

| Opaque ID or resource | Established use | Unresolved contract |
|---|---|---|
| F-6d8c85350b66 | Called once after accepted kind dispatch, with S+`0x40`; zero/nonzero selects subsequent writes. | Access extent, mutations, global effects, ownership, failure, and non-return behavior. |
| F-266a29fc8dc5 | Called after the counter increment, using a freshly read T value and zero. | Argument meaning, access extent, mutations, ownership, and return behavior. |
| K1 and K7 storage | Selected 32-bit read immediately before the corresponding D+`0x78` assignment. | Addresses, contents, aliasing, mutability, and semantic meaning. |
| Counter-pointer storage | Read after optional-region processing and before the D+`0x8C` write. | Pointer representation, target validity, ownership, aliasing, and synchronization. |
| T storage | Read after the counter increment for the final call. | Stored type and width beyond the visible argument transfer, address, contents, and aliasing. |

Dependencies must not be assumed to preserve globals or to retain, acquire, or release memory in any particular way.

## Acceptance examples

These are conditional static examples, not executed tests. Unless an example states otherwise, storage is valid, stable, and disjoint; dependencies return normally without additional relevant mutations; signed operations remain representable. Observations refer to the indicated stage, before any unconstrained later dependency effects.

1. **Early rejection retains writes.** With S+`0x08` equal to 17, the result is `0xFFFFFFFF`. Initial copies, baseline/marker assignments, selected byte writes, and the D+`0x40` write have occurred. Mode processing and dependency calls have not.

2. **Unknown selector can complete.** Selector 8 performs no selector-group writes. Accepted range, mode, and kind can still reach normal completion.

3. **Low range preserves existing state before incrementing.** With range 8, initial D+`0x3C` equal to 20, and mode 2, that field becomes 21. Mode 1 instead leaves it at 20.

4. **Upper range initializes before incrementing.** With range 16, marker word 5, and mode 2, D+`0x3C` becomes 6.

5. **Mode rejection follows a byte write.** With accepted range and stable mode 3, D+`0x41` receives the baseline byte before rejection. The subsequent D+`0x14` assignment does not occur.

6. **Kind rejection occurs late.** Accepted range and mode followed by kind 2 cause rejection after mode-stage initialization. Neither dependency is called.

7. **Zero helper result uses the full baseline value.** With baseline word `0x12345678` and a zero result from F-6d8c85350b66, D+`0x28` receives `0x78`; D+`0x2C`, `0x30`, `0x34`, `0x38`, `0x7C`, and `0x80` each receive `0x12345678`.

8. **Transformation precedes derived arithmetic.** With a nonzero helper result, post-helper D+`0x78` equal to 10, copied values 3, 8, and 5, and D+`0x58` equal to 7, the transformed values are 6, 16, and 10. D+`0x7C` becomes 16 and D+`0x80` becomes 6.

9. **The derived sum wraps.** With no shifting, D+`0x30` containing `0xFFFFFFFF` and captured D+`0x78` equal to 2, D+`0x7C` receives `0x00000001`. The signed destination read converts to unsigned before addition.

10. **Only the selected external kind value is read.** Kind 1 reads K1 storage after preceding initialization; it does not read K7 storage. If K1 storage aliases D+`0x08`, that read observes the baseline word previously written there.

11. **Counter-pointer acquisition precedes the byte write.** If counter-pointer storage aliases bytes changed by D+`0x8C`, the increment still uses the pointer acquired before that byte write. Its target value is read afterward.

12. **T observes the post-increment state.** If T storage is the incremented counter word and its value changes from 4 to 5, the final dependency receives the value read as 5. Likewise, changes to T storage by the first dependency are not excluded by an entry-time snapshot.

## Uncertainties

The following remain **documented_only**:

- The true entry convention supplying the four additional incoming values.
- Application-level object types and field meanings.
- External-storage addresses, contents, aliasing, and mutability.
- Dependency access, ownership, preservation, and failure contracts.
- Platform byte order, alignment, fault handling, and concurrency guarantees.
- Signed-overflow behavior for the D+`0x3C` increment and counter increment.
- Signed-left-shift behavior for negative operands or unrepresentable results.

These arithmetic uncertainties do **not** apply to the established unsigned 32-bit addition at D+`0x7C`, its modulo-\(2^{32}\) result, the unsigned comparisons, or the conditional unsigned subtraction.

## Evidence

This contract is based solely on the supplied static representation and opaque dependency mapping.

Established evidence includes direct access widths and offsets, separate range and mode reads, the repeated subtraction-operand read, partial-update rejection paths, external-storage read timing, dependency-call order, unsigned arithmetic at D+`0x7C`, and exact return bit patterns.

Descriptive field roles are inferred intent rather than recovered application semantics. Unavailable contracts and platform details remain explicitly unresolved.

No runtime tests have been performed. No execution traces, dependency implementations, or independent platform definitions were supplied.
