<!-- Trusted provenance: F-882a3ccad27b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-882a3ccad27b

## Purpose

F-882a3ccad27b selects a replacement value for a three-bit field and writes a 32-bit word through an external reference. Supported selectors are 0 through 4; selectors 0 and 1 select the same field value. Other selectors return a rejection status without the explicit word write, but only after the initial word read.

For a supported selector, the written value combines bits 31–3 from the initially read word with a fixed replacement for bits 2–0.

The external reference is evaluated separately for the initial read and the supported-selector write in the supplied representation. The function does not explicitly retain the initial destination address for reuse. Consequently, describing the operation as an update to one location requires reference stability. The fragment does not establish whether a changing reference could redirect the write.

The field’s domain meaning is unknown. Interpreting it as a mode or hardware-control setting is inferred intent, not an established guarantee.

## Interface

### Selector input

The function accepts one 32-bit selector. Its complete value determines the selected behavior.

| Selector | Replacement for bits 2–0 | Normal status |
|---:|---|---:|
| 0 | `000` | 0 |
| 1 | `000` | 0 |
| 2 | `101` | 0 |
| 3 | `011` | 0 |
| 4 | `001` | 0 |
| Any other 32-bit value | No word write | 1 |

There is no demonstrated selector truncation, masking, normalization, or caller-supplied lookup storage. Inputs such as `0x00000102` do not select the behavior of selector 2. A bit pattern interpreted as negative by a caller is not among the five accepted values.

### External reference and accessed storage

No destination argument is supplied by the caller. Instead, the function accesses storage through an external reference.

The supplied representation contains two distinct uses of that reference:

- An evaluation identifying the location from which a 32-bit word is initially read.
- For a supported selector, a later evaluation identifying the location to which the replacement 32-bit word is written.

These are separate reference evaluations in the representation, not evidence of two physical pointer loads. The reference’s declaration, mutability, volatile semantics, and generated access mechanics are unavailable.

Normal completion requires the initial reference evaluation and word read to be valid, including for unsupported selectors. Supported selectors additionally require the later reference evaluation and word write to be valid.

Reference stability is a precondition for interpreting this as a read-modify-write of one location. It is not established as an unconditional property of the function or its environment.

### Return value

The normal return is a 32-bit status:

- **0:** a supported selector reached and completed the explicit word-write path.
- **1:** an unsupported selector was rejected after the initial read, without the explicit word write.

The status does not return either accessed address, the previous word, the replacement word, or an indication that any numerical bit changed. A zero status does not establish device acknowledgement or persistence of the written value.

No alternative normal return value is shown.

## Behavior

### Initial access precedes selector rejection

Before deciding whether the selector is supported, the function evaluates the external reference and reads a 32-bit word from the location it identifies.

The low three bits of that word are excluded from the portion retained for a possible write. Bits 31–3 supply the preserved portion of the eventual output.

An unsupported selector then returns 1. It does not reach the later reference evaluation used by the explicit write.

Rejection is therefore not a read-free operation. An unsupported selector cannot be used to bypass an invalid initial reference or avoid the effects of reading side-effecting storage.

### Supported-selector output construction

For a supported selector, the output word has:

- Bits 31–3 equal to those bits of the word obtained by the initial read.
- Bits 2–0 equal to the selected replacement pattern.

All three low bits are replaced. The operation is not an accumulation of newly set bits:

- Selectors 0 and 1 clear the entire low field.
- Selector 2 sets bits 2 and 0 and clears bit 1.
- Selector 3 sets bits 1 and 0 and clears bit 2.
- Selector 4 sets bit 0 and clears bits 2 and 1.

The preservation mask is `0xFFFFFFF8`. The selected low-field values are respectively 0, 0, 5, 3, and 1.

### Later destination evaluation

After selecting the replacement field, the function evaluates the external reference again for the write and writes the constructed 32-bit value through it. It then returns 0.

The representation does not explicitly save the address used for the initial read. This creates two distinct statements of the contract:

- **Unconditional value relationship:** the written word’s upper 29 bits come from the initial word read.
- **Conditional same-location update:** if both reference evaluations identify the same location, the function replaces that location’s low three bits using the initially read upper bits.

Without the stability condition, the contract does not promise preservation of the write destination’s previous upper bits. If the environment and access mechanics permit the later evaluation to identify a different location, the constructed value would carry upper bits from the read location rather than from the write location. Whether such redirection can actually occur is unresolved.

### Unchanged-value writes

Every supported selector reaches the write path after successful preceding accesses. There is no equality comparison that suppresses a write when the constructed value already matches the destination.

With a stable reference and unchanged ordinary storage, selector 4 applied to `0x00000001` writes `0x00000001` and returns 0. Numerical equality does not mean that the write is omitted.

### Width and numerical interpretation

The selector, word operation, and returned status are represented as 32-bit quantities. Bit 0 is the least significant bit; bits 2–0 form the selected field.

The field update uses masking and bitwise combination. No arithmetic addition, subtraction, multiplication, or shifting is needed, and no arithmetic overflow boundary arises from the demonstrated transformation.

The numerical bit positions do not establish byte order, physical bus width, access alignment, or the number of processor instructions.

## Side effects

### Represented access order

For a supported selector, the observable access sequence represented by the fragment is:

1. Evaluate the external reference for the read.
2. Read the identified 32-bit word.
3. Select the replacement low field.
4. Evaluate the external reference for the write.
5. Write the constructed 32-bit word.
6. Return 0.

For an unsupported selector, execution stops with status 1 after the initial reference evaluation and word read; the explicit write path is not reached.

This sequence describes the supplied representation. It does not establish physical pointer-load counts, volatile access guarantees, barriers, bus transactions, or atomicity.

### Reference stability and interference

Two independent environmental questions affect preservation:

- Do both reference evaluations identify the same location?
- Does the contents of that location change between the read and write?

Even with a stable reference, upper-bit preservation is relative to the initially read word, not to the contents immediately before the write. An intervening change to an upper bit can be overwritten by the later write.

If the reference itself can change in a way that redirects the later access, the write may instead concern another location. The fragment supplies neither a saved destination address nor an explicit mechanism establishing reference stability. It also does not establish that redirection is possible on the actual platform.

No lock, atomic read-modify-write primitive, retry, or conflict detection is shown. External coordination is required wherever callers need guarantees beyond the demonstrated read-then-write behavior.

### Read-sensitive and write-sensitive storage

An unsupported selector can have externally visible effects if the initial word read has side effects. “No explicit write” does not mean “no state change.”

A supported selector performs the explicit write even if its value is numerically unchanged. Device-specific effects of either access are unknown; ordinary-memory semantics must not be assumed for an unidentified target.

### Ownership and lifetime

No allocation, release, ownership transfer, or lifetime-management operation is shown. The function accepts no ownership-bearing object and returns none.

The environment must supply accessible storage for each access actually reached. The function does not establish continued accessibility after return.

### Errors outside selector rejection

The fragment shows no explicit recovery for an invalid reference evaluation, an unreadable location, an unwritable location, or a platform access fault.

Such failures must not be specified as returning 1. That status is established only for unsupported selectors after the initial read. Likewise, status 0 does not prove that a device accepted or acted on a write; no acknowledgement or readback is shown.

## Dependencies

No function calls are shown within F-882a3ccad27b. There are no demonstrated callee contracts or additional opaque function IDs to enumerate.

The environmental dependencies are:

- The externally supplied reference and its evaluation semantics.
- A readable 32-bit location for the initial access.
- A writable 32-bit location for the later access when the selector is supported.
- Reference stability when same-location update semantics are required.
- Coordination against interfering changes when preservation of intervening state is required.
- Platform-specific rules for alignment, access permissions, ordering, and side-effecting storage.

No helper, allocator, validator, or synchronization behavior can be attributed to an unseen callee.

## Acceptance examples

The following criteria assume ordinary readable and writable memory, successful accesses, a reference that identifies the same location at both evaluations, and no concurrent modification. They are static-derived examples, not executed tests.

| Initial word | Selector | Required final word | Return | Explicit word write |
|---|---:|---|---:|---|
| `0xA5A5A5AF` | 0 | `0xA5A5A5A8` | 0 | Yes |
| `0xA5A5A5AF` | 1 | `0xA5A5A5A8` | 0 | Yes |
| `0xA5A5A5AF` | 2 | `0xA5A5A5AD` | 0 | Yes |
| `0xA5A5A5AF` | 3 | `0xA5A5A5AB` | 0 | Yes |
| `0xA5A5A5AF` | 4 | `0xA5A5A5A9` | 0 | Yes |
| `0xFFFFFFFF` | 2 | `0xFFFFFFFD` | 0 | Yes |
| `0x00000000` | 3 | `0x00000003` | 0 | Yes |
| `0x00000001` | 4 | `0x00000001` | 0 | Yes |
| `0x1234567F` | 5 | `0x1234567F` | 1 | No |
| `0x1234567F` | `0x00000102` | `0x1234567F` | 1 | No |
| `0x1234567F` | `0xFFFFFFFF` | `0x1234567F` | 1 | No |

Additional conditional criteria:

- **Rejected input still reads:** with an unsupported selector and a read-sensitive initial location, the read occurs before status 1. Its storage-specific effects remain outside this contract.
- **No unchanged-value optimization:** with a stable reference to a word whose low field is already zero, selector 0 still performs the explicit write.
- **Intervening content change:** if a stable location initially contains `0x00000000`, another actor changes it to `0x80000000` after the read, and selector 4 proceeds without further interference, the function writes `0x00000001`. The intervening high-bit change is not preserved.
- **Conditional destination redirection:** if the environment permits the read evaluation to identify one location and the later write evaluation to identify another, an initial read of `0x1234567F` with selector 2 produces a write value of `0x1234567D` at the later destination. Its previous upper bits do not contribute. This criterion does not assert that changing-reference behavior is possible in the actual environment.
- **Access failure:** an unreadable initial location does not have a specified status-1 outcome, even for an unsupported selector.

## Uncertainties

**Inferred intent — encoded setting:** The mapping resembles a small encoded setting. Its domain meaning and relationship to other system state are not established.

**documented_only — reference properties:** The external reference’s declaration, initialization, mutability, storage class, and volatile semantics are unavailable.

**documented_only — reference stability:** The fragment does not establish whether the two evaluations must identify the same location or whether an intervening reference change could redirect the write.

**documented_only — physical access mechanics:** Separate reference evaluations do not prove separate physical pointer loads. Instruction selection, access atomicity, alignment, ordering, barriers, byte order, and bus behavior are unknown.

**documented_only — target identity and lifetime:** The public identity, permitted addresses, ownership, accessibility period, and ordinary-memory or device nature of the accessed storage are unknown.

**documented_only — external interference:** The ability of threads, interrupt handlers, devices, or other actors to alter the reference or accessed contents is not established.

**documented_only — abnormal execution:** Fault handling and caller-visible consequences of invalid accesses are unspecified.

**documented_only — surrounding caller contract:** Caller-side validation, synchronization requirements imposed elsewhere, and calling-convention details beyond the represented input and return widths are not supplied.

## Evidence

The supplied static fragment establishes:

- A 32-bit selector and 32-bit normal status.
- An initial referenced word read before selector rejection.
- Retention of bits 31–3 using `0xFFFFFFF8`.
- The selector-to-field mapping stated above.
- A later, separate use of the external reference for the supported-selector write.
- No explicit retention of the initial destination address.
- Status 1 without that write for unsupported selectors.
- Status 0 following the supported-selector write.
- No explicit function calls, equality-based write suppression, synchronization, or access-fault recovery.

The fragment does not establish reference stability or generated physical access mechanics. Same-location preservation claims therefore require the stated stability condition.

No runtime tests, hardware observations, or independent implementation verification have been performed.
