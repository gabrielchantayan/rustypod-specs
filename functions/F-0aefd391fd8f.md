<!-- Trusted provenance: F-0aefd391fd8f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0aefd391fd8f

## Purpose

F-0aefd391fd8f validates a 32-bit options word, initializes caller-supplied storage using a value read from a global datum, obtains a candidate resource through dependencies, and accepts that candidate only when two ordered checks satisfy distinct zero/nonzero conditions.

The established success path records the candidate, sets two one-byte fields, stores the original options word, optionally copies that word to another output location, and returns the 32-bit value zero.

Invalid options and a zero acquisition result lead to F-a817d8d58c10. Rejection of a nonzero candidate leads to F-5e7f4157bc69, followed by F-feee9156a257 if the preceding call returns. Both error dependencies are annotated as nonreturning. No normal error return is established.

“Resource,” “acquisition,” and “rejection” describe the observed call structure. The candidate’s actual type, the checks’ semantic purposes, and any ownership transfer remain unknown.

## Interface

### Arguments

The interface has five arguments, in this order:

| Argument | Representation | Established use |
|---|---|---|
| Unused input word | 32 bits | Neither read nor forwarded by the visible body |
| Resource-request input | 32 bits | Forwarded unchanged as the second argument to F-f777b9e7aa85 |
| Destination address | Address in the supplied 32-bit representation | Receives fixed-offset writes after options pass validation |
| Options | Unsigned 32-bit word | Validated, used to derive control values, and retained without normalization |
| Optional options-output address | Address or null | Receives one 32-bit options write on success when nonnull |

The resource-request input’s interpretation is unknown. It must not be assumed to be a pointer, handle, identifier, or size.

The body performs no visible destination null check, bounds check, alignment check, or writability check. On a valid-options path, the destination must support the described accesses. On success, a nonnull optional output address must support its 32-bit write.

The supplied representation uses 32-bit address arithmetic. Portable behavior at other address widths, address wraparound, and behavior for inaccessible memory are not established.

### Global input

After validation succeeds and before the first destination write, the function reads one 32-bit global datum. It stores the value obtained by that read at destination offset 0.

This datum is a memory input, not an established embedded constant. Neither its numeric value nor its immutability is shown. The visible function does not explicitly read it again.

The global datum must be readable when this access is reached. No evidence excludes overlap between its storage and either output region.

### Destination layout

Offsets are measured in bytes from the destination address.

| Offset | Access width | Initialization write | Later success write |
|---|---:|---|---|
| 0 | 32 bits | Value obtained from the global datum | None directly |
| 4 | 32 bits | Zero | Candidate resource value |
| 8 | 8 bits | Zero | One |
| 9 | 8 bits | Zero | One if options bit `0x08` is set; otherwise zero |
| 10–11 | No direct access | None | None |
| 12 | 32 bits | Zero | Original options word |

The ordinary nonoverlapping destination requires accessible storage spanning offsets 0 through 15. This does not establish that the complete logical record is exactly 16 bytes.

The table describes direct writes, not an unconditional final memory image. Aliasing and dependency effects can change the resulting contents.

### Result

The only established normal return is the 32-bit value zero.

No normal error-code return is shown. No completed-output guarantee is established for a path invoking either nonreturning error dependency.

## Behavior

### Validation

Validation precedes the global read, all direct output writes, and all dependency calls.

The selector is the options word masked with `0xffffff00`. The low-byte requirements are:

- Exactly one of bits `0x01` and `0x02` is set.
- Bit `0x04`, if set, requires bit `0x02`.
- Bit `0x10`, if set, requires bit `0x04`.
- Bit `0x08`, if set, requires bit `0x04`.

Ignoring bits `0x20`, `0x40`, and `0x80`, the permitted low-byte patterns are exactly:

`0x01`, `0x02`, `0x06`, `0x0e`, `0x16`, and `0x1e`.

Bits `0x20`, `0x40`, and `0x80` do not participate in the visible validation. They may accompany an otherwise valid pattern and are retained in subsequent options writes.

The selector must equal exactly one of these 32-bit values:

`0x00000100`, `0x00000200`, `0x00000400`, `0x00000800`, `0x00001000`, `0x00002000`, or `0x00004000`.

These are alternatives, not independently combinable selector bits. Other upper-byte combinations fail, including combinations of listed alternatives and values containing additional high bits.

For selectors `0x00000100`, `0x00000800`, and `0x00004000`, bit `0x08` must additionally be clear. The other permitted selectors impose no additional visible restriction on that bit.

Failure of any validation requirement leads directly to F-a817d8d58c10. Before that call, this function has not read the global datum, written either output region, or invoked another dependency.

### Global read and initialization

After successful validation, the function reads the 32-bit global datum. It then performs these direct writes in order:

1. The value just read, as a 32-bit write at destination offset 0.
2. A 32-bit zero at offset 4.
3. An 8-bit zero at offset 8.
4. An 8-bit zero at offset 9.
5. A 32-bit zero at offset 12.

The global read therefore precedes every initialization write. Its position matters if the global datum overlaps the destination.

No dependency call intervenes between the global read and these initialization writes. No visible rollback restores prior destination contents if a later operation fails.

The numeric value and semantic role of the global datum are unknown. It is not established to be a type identifier, dispatch-table address, ownership marker, or fixed value shared across invocations.

### Acquisition

After initialization, F-8157a180d901 is called with the numeric argument four.

Its returned 32-bit value becomes the first argument to F-f777b9e7aa85. That dependency receives six arguments, in order:

1. The result of F-8157a180d901.
2. The resource-request input.
3. A normalized zero-or-one value indicating whether options bit `0x01` is set.
4. Zero.
5. `0x00080000`, numerically 524,288.
6. Four.

The values and order are established; their units and semantic roles are not.

A zero result from F-f777b9e7aa85 leads to F-a817d8d58c10. Destination initialization has already occurred. Neither candidate check nor F-5e7f4157bc69 is called on this path, and the optional output has not been directly written.

A nonzero result becomes the candidate passed to the checks. No particular nonzero value or positive signed interpretation is required by the visible comparisons.

### Ordered checks and rejection

For a nonzero candidate, F-f18b588c61d8 is called first with that candidate.

- A nonzero result rejects the candidate and skips F-df646e63d1e6.
- A zero result permits F-df646e63d1e6 to be called with the same candidate.
- A nonzero result from F-df646e63d1e6 accepts the candidate.
- A zero result from F-df646e63d1e6 rejects it.

Acceptance therefore requires a zero first-check result followed by a nonzero second-check result. Both results are used only for zero/nonzero decisions.

On rejection, the function calls F-5e7f4157bc69 with the candidate. If that call returns, F-feee9156a257 is invoked.

No success-publication writes occur before this rejection sequence. Calling F-5e7f4157bc69 does not establish successful release, destruction, ownership transfer, or preservation of any caller-visible state.

### Success publication

After the checks accept the candidate, direct writes occur in this order:

1. The 32-bit candidate at destination offset 4.
2. The byte value one at offset 8.
3. A byte at offset 9: one if options bit `0x08` is set, otherwise zero.
4. The original 32-bit options word at offset 12.
5. If the optional output address is nonnull, the original 32-bit options word at that address.

The function then returns zero.

The offset-9 value is normalized to zero or one; it is not the mask value eight. The full options word retains all original bits, including low-byte bits not used in validation.

The global datum is not explicitly reread during publication. Changes to that datum after the initial read do not themselves cause this function to refresh destination offset 0.

### Aliasing and observable access order

No overlap checks are shown. Direct access order governs overlaps; the layout table does not override later writes.

If the optional output address equals the destination address, its final write replaces offset 0 with the options word. If it equals destination offset 4, it replaces the candidate field with the options word.

There are no reads through the destination argument and no reads through the optional output argument in the visible body. This does **not** establish that existing destination bytes are never read: the separate global read may access overlapping storage.

If the global datum occupies destination offset 4, its prior 32-bit value is read before initialization clears offset 4. That value is first copied to offset 0; successful publication later writes the candidate at offset 4. If the global datum occupies destination offset 0, its value is read before being written back there.

Writes through either output may also modify the global datum when storage overlaps. Dependencies may independently modify it or either output region. No preservation guarantee is inferred for those calls.

No atomicity, synchronization, or concurrent-observer guarantee is established. In particular, setting offset 8 precedes the success writes to offsets 9 and 12; it is not evidence of an atomic publication protocol.

## Side effects

Direct effects consist of one conditional global read, the destination writes, and the conditional optional-output write. The global read and destination initialization occur only after valid options.

The principal path distinctions are:

| Path | Direct actions before error or return |
|---|---|
| Invalid options | No global read or output writes; invokes F-a817d8d58c10 |
| Zero acquisition result | Global read and destination initialization; invokes F-a817d8d58c10 |
| Rejected nonzero candidate | Global read, initialization, applicable checks, then F-5e7f4157bc69 and, if it returns, F-feee9156a257 |
| Success | Global read, initialization, both checks, success publication, optional output write, zero return |

Dependency effects are additional and unspecified. Thus, “no direct optional-output write” on a failure path does not mean that dependencies cannot alter that memory.

The body contains no direct allocation or deallocation operation. It does not invoke F-5e7f4157bc69 for an accepted candidate before returning.

**Inferred intent:** the byte at offset 8 may indicate an active or populated record, and the candidate may become associated with that record. Neither inference establishes ownership. Who must eventually release a successful candidate, and who manages destination storage, remain unresolved.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-8157a180d901 | Called after initialization with four; its result feeds acquisition | Meaning of result, side effects, failure behavior |
| F-f777b9e7aa85 | Receives six ordered arguments; zero selects failure, nonzero supplies the candidate | Resource type, input interpretation, allocation and ownership |
| F-f18b588c61d8 | First check; zero permits the second check | Check meaning and side effects |
| F-df646e63d1e6 | Second check; nonzero permits publication | Check meaning and side effects |
| F-5e7f4157bc69 | Receives a rejected nonzero candidate | Whether release occurs, whether it returns, other effects |
| F-feee9156a257 | Invoked after the rejection dependency returns; annotated nonreturning | Diagnostics and control-transfer mechanism |
| F-a817d8d58c10 | Invoked for invalid options or zero acquisition result; annotated nonreturning | Diagnostics and control-transfer mechanism |

No dependency is assumed to preserve globals, preserve output storage, own memory, release memory, or return normally unless required to describe the continuation conditional on its return.

## Acceptance examples

These are conditional deductions from static evidence, not executed tests. Success examples assume accessible storage, returning acquisition/check dependencies, a nonzero candidate, first-check result zero, and second-check result nonzero. Unless an example specifies overlap, output regions and the global datum are assumed disjoint, with no interfering dependency writes.

1. **First control alternative:** `0x00000101` passes validation. Acquisition receives one for its third argument. On success, offset 9 is zero, offset 12 contains `0x00000101`, and the function returns zero.

2. **Permitted bit `0x08`:** `0x0000020e` passes validation. Acquisition receives zero for its third argument. On success, offsets 8 and 9 both contain one.

3. **Forbidden selector/control pairing:** `0x0000010e` fails because selector `0x00000100` forbids bit `0x08`. F-a817d8d58c10 is invoked without a preceding global read or direct output write.

4. **Exclusive low bits:** `0x00000203` fails because both `0x01` and `0x02` are set. `0x00000200` fails because neither is set.

5. **Missing prerequisite:** `0x00000212` fails because bit `0x10` requires bit `0x04`.

6. **Combined selectors:** `0x00000302` fails despite its permitted low-byte pattern. Its selector is not one of the exact alternatives.

7. **Uninterpreted low-byte bits:** `0x000004e2` passes validation. Successful options writes retain `0x000004e2`; bits `0x20`, `0x40`, and `0x80` are not stripped.

8. **Changing global input across calls:** if two valid invocations read different global values, their initial offset-0 writes contain those respective values. The function does not establish a fixed initialization constant.

9. **Global overlapping offset 4:** if the global datum occupies destination offset 4 and initially contains `0x12345678`, initialization first writes `0x12345678` at offset 0, then clears offset 4. On success, offset 4 later receives the candidate.

10. **Acquisition failure:** with valid options and a zero acquisition result, the global read and initialization occur before F-a817d8d58c10. Neither check nor F-5e7f4157bc69 is called.

11. **First-check rejection:** a nonzero first-check result skips the second check and invokes F-5e7f4157bc69 with the candidate, followed by F-feee9156a257 if that call returns.

12. **Second-check rejection:** first-check result zero and second-check result zero cause the same rejection sequence, without direct success-publication writes.

13. **Null optional output:** a null fifth argument suppresses only the final optional write; it does not prevent success.

14. **Optional output overlapping offset 4:** success writes the candidate there, then the final optional write replaces it with the original options word.

## Uncertainties

The following remain **documented_only**:

- The global datum’s numeric value, semantic role, mutability, initialization, and synchronization.
- Whether its storage overlaps the destination or optional output in actual callers.
- Selector and control-bit meanings.
- Resource-request representation and validity requirements.
- Candidate type, lifetime, ownership, and eventual release obligation.
- Dependency effects on globals, outputs, and other caller-visible state.
- Required alignment, target byte order, and behavior for inaccessible memory or address wraparound.
- The wider record layout and meanings of offsets 10 and 11.
- Error diagnostics and the mechanism by which annotated nonreturning dependencies transfer control.
- Behavior if an annotated nonreturning dependency unexpectedly returns.
- Thread safety and observations during partial initialization or publication.

Numeric arguments alone do not establish sizes, counts, flags, or units.

## Evidence

Established static evidence comprises the supplied body’s 32-bit masks and comparisons, the global 32-bit read after validation and before the first destination write, fixed-width destination accesses, ordered dependency calls and arguments, zero/nonzero decisions, conditional optional write, and explicit zero success return.

The body establishes a read from global storage, not the datum’s numeric value or immutability. No read through the destination argument is shown, but overlap with the global storage is not excluded.

Validation rules and acceptance examples are deductions from those operations. The nonreturning classifications derive from supplied annotations, not independent examination of the dependencies. Resource-management terminology and possible field meanings are inferred intent, not established ownership or lifecycle guarantees.

No runtime tests have been performed. No dependency implementation has been examined.
