<!-- Trusted provenance: F-9a93975eefed; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9a93975eefed

## Purpose

F-9a93975eefed performs a state-dependent operation on a mutable context. It checks a marker and an immediate-block flag, optionally initializes a negative phase, chooses a processing dependency, conditionally invokes a notification dependency, and updates or substitutes a status.

The 64-bit result combines two independent 32-bit components:

- Low half: the selected status.
- High half: an initially supplied argument that an indirect dependency can modify through a local argument area.

“Owner,” “phase,” “status,” “capture,” and “notification” are descriptive labels, not established application semantics. The transformation chains operate on opaque two-word values; their numeric interpretation is unknown.

This contract describes static evidence. All dependency-dependent outcomes are conditional on those dependencies returning normally and leaving subsequent accesses valid.

## Interface

### Inputs and result

The function receives:

1. A context address.
2. Three opaque 32-bit values, designated A, B, and C.

Addresses stored in the supplied representation occupy 32-bit fields. The representation does not establish a portable interface for wider pointers.

The return value is 64 bits. Bits 0–31 contain the selected 32-bit status; bits 32–63 contain the current A from a local argument area. This is a bitwise composition, not an established signed or unsigned application-level number.

A is preserved locally before validation. After successful validation and the immediate-block check, B and C complete a contiguous three-word local area ordered A, B, C. An indirect dependency receives its address and may modify its contents. Subsequent uses of A and B take their then-current values; the returned upper half likewise uses the final local A. C has no separate direct use after entering that area.

### Context layout

Offsets are in bytes.

| Offset | Width | Observed use |
|---|---:|---|
| `0x00` | 32 bits | Owner address |
| `0x0C` | 32 bits | Signed quantity tested for notification eligibility |
| `0x14` | 32 bits | Auxiliary-record address |
| `0x44` | 32 bits | Marker compared with an external expected value |
| `0x70` | 32 bits | Signed phase |
| `0x74` | 32 bits | Stored status |
| `0xFC` | 8 bits | Processing-route selector |
| `0xFE` | 8 bits | Immediate-block flag |
| `0xFF` | 8 bits | Gate for a nonpositive phase |
| `0x108` | 64 bits | Saved opaque value, also accessed as two 32-bit words |
| `0x148` | 32 bits | Final-substitution enablement |

The function does not validate the context address, extent, alignment, or accessibility.

### Owner and auxiliary-record layouts

After successful marker validation, the owner address is captured from context offset `0x00`. Owner-relative operations use that captured address except where explicitly described as fresh context-owner reads.

| Owner offset | Width | Observed use |
|---|---:|---|
| `0x00` | 32 bits | Address used to resolve an indirect call and supplied as its first argument |
| `0x80` | 32 bits, low byte tested | Notification inhibition |
| `0x90` | 32 bits | Initialization counter |
| `0x9C` | 32 bits | Indirect notification target |
| `0xA0` | 32 bits | First notification argument |
| `0xD8` | 32 bits | Field cleared when the initial counter check finds zero |

The auxiliary record is accessed at:

- Offset `0x00`: one byte, compared with `0x14`.
- Offset `0x10`: one 32-bit word, tested for nonzero and later supplied to notification.

There is no explicit null check for the owner, auxiliary record, or virtual-table address. A path that reaches an access requires the relevant address to remain valid.

### External transformation operands

The three external operand pairs have distinct contract-local identities:

| Identity | Binding to supplied operand suffixes | Argument order |
|---|---|---|
| E-01 | `cdc/ce0` | Operand associated with `cdc`, then operand associated with `ce0` |
| E-02 | `cd4/cd8` | Operand associated with `cd4`, then operand associated with `cd8` |
| E-03 | `ccc/cd0` | Operand associated with `ccc`, then operand associated with `cd0` |

Each pair supplies two 32-bit operands to one transformation call. Their numeric values, storage properties, and mutability are not established. They are **not contractual constants**.

Each transformation obtains its pair at that stage, after preceding dependencies return. No pair is established as cached for a later stage or reused from initialization during notification. The representation does not establish an atomic pair snapshot or the relative evaluation order of the two reads within one call’s arguments.

## Behavior

### Validation and immediate rejection

The context marker at `0x44` is compared with an external expected 32-bit value.

- Mismatch: return status `0x15` with original A in the upper half.
- Match: capture the owner address from context offset `0x00`, then test the byte at `0xFE`.

A nonzero immediate-block byte returns status `4` with original A in the upper half. Capturing the owner address does not itself dereference the owner.

Neither early-return path reaches phase handling, processing dependencies, stored-status updates, or final substitution. No direct context or owner write occurs on either path.

After these checks, the local argument area contains A, B, and C.

### Phase and gate selection

The first signed 32-bit phase test separates positive values from zero and negative values.

- Positive phase: proceed to ordinary processing without testing the gate.
- Nonpositive phase: inspect the gate byte at `0xFF`.
- Nonpositive phase with nonzero gate: take the gated path.
- Nonpositive phase with zero gate: read the phase again; initialize if that signed value is negative, then proceed to ordinary processing.

For stable context contents, zero phase with zero gate proceeds without initialization.

The gated path selects status `1`. If the stored status at `0x74` is zero, it writes `0x11` there; otherwise, it retains that field. It skips initialization, both processing routes, the ordinary notification sequence, and both ordinary status-update calls. It still reaches final substitution.

### Negative-phase initialization

Initialization proceeds in this order:

1. Read the captured owner’s counter at `0x90`. If zero, clear its word at `0xD8`.
2. Test the captured owner’s notification target at `0x9C`.
3. If that target is nonzero, test whether the low byte of owner word `0x80` is zero.
4. If both tests pass, perform the initialization capture sequence.
5. Increment the captured owner’s counter at `0x90`.
6. Write zero to the context phase at `0x70`.
7. Call F-558af7087074 with the context.
8. Proceed to ordinary processing.

The counter increment uses the value present after the optional capture sequence. Dependencies in that sequence must not be assumed to preserve the counter.

The increment is represented using a signed 32-bit field. Behavior at signed overflow is not established; modulo wraparound is not guaranteed.

### Initialization capture sequence

The indirect dependency identified here as F-000000000001 is resolved through the captured owner’s first word, using a table entry at byte offset `0x44`. Its arguments are:

1. The owner’s first word itself, not the captured owner address.
2. The address of the local A, B, C area.

After that dependency returns:

1. Call F-baa332e06e32 with current A and B; its return value is unused.
2. Obtain a 64-bit value from F-b3a54d881220.
3. Call F-387e686c467e with that value’s low and high 32-bit halves, followed by current A and B.
4. Obtain E-01’s current operands and call F-c4b34006a13c with the preceding result’s low and high halves, followed by those operands.
5. After that call returns, obtain E-02’s current operands and apply F-c4b34006a13c to the new split result.
6. After that call returns, obtain E-03’s current operands and apply F-c4b34006a13c again.
7. Discard this third transformation’s result.
8. Obtain a separate 64-bit value from F-1b89e74f4780 and store it at context offset `0x108`.

Every transformation remains required even though the final transformed result is discarded. Its side effects may affect subsequent calls, operands, or state.

In particular, a dependency preceding a stage may change that stage’s external operands. Values observed for E-01 do not freeze E-02 or E-03.

### Ordinary processing and eligibility

The selector byte at context offset `0xFC` chooses one dependency:

- Zero: F-f3193c743e68, receiving the context.
- Nonzero: F-41cf4e67d41c, with no explicit arguments.

Its 32-bit result becomes the processing status. Subsequent notification eligibility is evaluated in this short-circuit order:

1. Processing status differs from decimal `100`.
2. Captured owner’s notification target at `0x9C` is nonzero.
3. Low byte of captured owner word `0x80` is zero.
4. Signed context word at `0x0C` is greater than zero.
5. Auxiliary-record byte at offset `0x00` equals `0x14`.
6. Auxiliary-record word at offset `0x10` is nonzero.

An earlier failure prevents later checks. Status `100`, for example, avoids all notification-specific owner and record checks.

The eligibility check does not retain the auxiliary word or establish the validity of later accesses after intervening calls.

### Notification sequence

When all eligibility conditions pass:

1. Invoke F-000000000001 with the same argument forms used during initialization.
2. Read the two words at context offsets `0x108` and `0x10C`, and pass them to F-8d7acd353fe6. Retain its 64-bit result.
3. Call F-baa332e06e32 with current A and B.
4. Obtain a 64-bit value from F-b3a54d881220.
5. Transform it through F-387e686c467e using its low and high halves and current A and B.
6. Apply F-c4b34006a13c three times, using E-01, then E-02, then E-03. Each stage obtains its external pair only after preceding dependencies return.
7. Call F-8a693c103d1c with the final transformed value’s low and high halves, followed by the retained F-8d7acd353fe6 result’s low and high halves.
8. Obtain a 64-bit value from F-c16e7206cae2.
9. Invoke the notification target currently stored at captured-owner offset `0x9C`.

The notification call, designated F-000000000002, receives:

- The current captured-owner word at `0xA0`.
- The current auxiliary-record word at `0x10`, using the record address then read from the context.
- The low half of the F-c16e7206cae2 result.
- Its high half.

The earlier eligibility target and record contents are not frozen. There is no second validity or nonzero check before invocation. Dependencies may have changed the target, argument fields, or record address.

The saved value at `0x108` is read after F-000000000001 returns. Likewise, notification’s external operands are not cached copies from initialization.

### Ordinary status updates

Whether notification runs or is skipped, ordinary processing continues with:

1. F-a061acf353ed, receiving a freshly read context owner address, the retained processing status, and zero.
2. F-15a13295fe3c, receiving another freshly read context owner address and the then-current stored status.
3. A write of F-15a13295fe3c’s 32-bit result to context offset `0x74`.

These owner addresses may differ from the captured owner and from each other.

The selected return status remains the processing status unless final substitution replaces it. Updating the stored status alone does not replace it.

### Final substitution and return

The gated and ordinary paths converge here.

First, read context word `0x148`:

- Nonzero: form a comparison candidate from the selected status’s low eight bits.
- Zero: use candidate zero.

Read `0x148` again. If this second value is nonzero and the candidate is less than decimal `100`:

1. Call F-a061acf353ed with the current context owner address, the current stored status, and zero.
2. Reload the stored status after the call.
3. Use that reloaded value as the selected return status.

The two enablement reads are distinct. No consistency mechanism is shown. If the first reads zero and the second reads nonzero, candidate zero satisfies the comparison.

The comparison concerns only the candidate, not the full 32-bit status. For example, status `0x100` has candidate zero.

Finally, compose the selected status as the low half and current local A as the high half.

## Side effects

Visible direct writes are branch-dependent:

- Clear owner word `0xD8`.
- Increment owner counter `0x90`.
- Clear context phase `0x70`.
- Store a 64-bit value at context `0x108`.
- Initialize or replace stored status `0x74`.
- Initialize local argument storage.

Indirect access to the local area allows dependency-originated changes to A, B, or C. No bounds or write restrictions for that dependency are supplied.

There is no explicit allocation, release, ownership transfer, locking, rollback, or exception recovery. Dependencies may nevertheless allocate, release memory, mutate globals—including external operands—or invalidate objects. None is assumed to preserve globals or own or release passed memory.

The local argument area expires when this function returns. Retaining its address does not establish permission for later use.

The 64-bit store’s atomicity and physical word-write order are unknown. No coherent snapshot across separately read fields or external operands is guaranteed.

## Dependencies

| Opaque function ID | Established call-site role |
|---|---|
| F-558af7087074 | Called after phase clearing during initialization |
| F-f3193c743e68 | Context-taking processing route |
| F-41cf4e67d41c | Processing route with no explicit arguments |
| F-a061acf353ed | Receives owner address, status, zero |
| F-15a13295fe3c | Returns replacement stored status |
| F-baa332e06e32 | Receives current A and B; result unused |
| F-b3a54d881220 | Supplies an opaque 64-bit value |
| F-387e686c467e | Transforms split value using current A and B |
| F-c4b34006a13c | Transforms split value using a stage-specific external operand pair |
| F-1b89e74f4780 | Supplies the value saved during initialization |
| F-8d7acd353fe6 | Receives the saved two-word value |
| F-8a693c103d1c | Receives two split values; result unused |
| F-c16e7206cae2 | Supplies the value passed to notification |
| F-000000000001 | Indirect table-entry call receiving the local argument area |
| F-000000000002 | Indirect owner-held notification target |

The last two IDs are contract-local labels for unresolved indirect dependencies, not recovered implementation identities.

## Acceptance examples

These are conditional static cases, not executed tests. Unless stated otherwise, dependencies return normally, relevant addresses remain valid, and fields not explicitly changed remain stable.

1. **Marker mismatch:** return low half `0x15` and original A above it. No owner capture or dependency call is reached.

2. **Immediate block:** matching marker and nonzero `0xFE` produce status `4` and original A. Stored status is not directly changed.

3. **Gated initialization:** phase zero, nonzero gate, zero stored status, and disabled substitution cause a write of `0x11` but return status `1`. No ordinary processing occurs.

4. **Gated substitution:** with the preceding state and stable enabled substitution, F-a061acf353ed receives stored status `0x11`. If it changes that field to `9`, the returned low half is `9`.

5. **Positive phase:** a positive phase reaches the selected processing dependency even with a nonzero gate.

6. **Negative phase without capture:** negative phase, zero gate, zero counter, and zero notification target cause `0xD8` to be cleared, counter to become one, phase to become zero, and F-558af7087074 to run before processing.

7. **Status 100:** processing status `100` suppresses notification-specific checks but not ordinary status updates. With stable enabled substitution, candidate `100` does not qualify for substitution.

8. **Low-byte substitution:** processing status `0x100`, stable enabled substitution, and stored status `7` after ordinary updating cause a final substitution call with `7`. If that call preserves the field, return status is `7`.

9. **Changed external operands between stages:** if the E-01 transformation changes E-02’s storage, the E-02 stage receives the changed pair. It does not receive values cached before E-01 ran. The same rule applies from E-02 to E-03.

10. **Changed operands between chains:** if initialization and notification both occur and a dependency changes E-01 between them, notification’s first transformation receives E-01’s then-current pair, not initialization’s pair.

11. **Mutable upper half:** if a reached indirect local-area call changes A and no later operation changes it again, the returned upper half contains that changed value.

## Uncertainties

The following remain **documented_only**:

- External expected-marker value and its storage properties.
- Numeric values and mutability of E-01, E-02, and E-03.
- Read ordering within individual multi-argument expressions.
- Numeric interpretation of opaque 64-bit values.
- Application meanings of statuses, flags, record type, and phase.
- Initialization-counter overflow behavior.
- Dependency effects on globals, memory ownership, object lifetime, and subsequent address validity.
- Indirect-call conventions beyond the displayed argument structure.
- Concurrency behavior and cross-field consistency.
- Dependency failures, non-returning behavior, and error propagation.
- Whether an indirect dependency retains the local argument-area address.

Interpreting the chains as timing, measurement, or progress reporting is inferred intent, not established behavior. No such interpretation is required by this contract.

## Evidence

Established static evidence comprises the supplied access widths and offsets, signed comparisons, short-circuit conditions, branch structure, ordered dependency calls, visible writes, stage-specific external operand expressions, and return-half composition.

The operand order is E-01 (`cdc/ce0`), E-02 (`cd4/cd8`), then E-03 (`ccc/cd0`) in both chains. The representation supplies fresh external operand expressions at each stage; it does not establish immutable constants or cross-stage caching.

Comments, prior prose, and embedded semantic names are not treated as authoritative evidence. No runtime tests have been performed. Acceptance examples express conditional consequences of the static representation, not observed execution.
