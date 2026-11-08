<!-- Trusted provenance: F-e6b20755bfcb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e6b20755bfcb

## Purpose

F-e6b20755bfcb prepares destination-associated data from a mutable context, then repeatedly calls a dependency using a captured destination and a fixed step. After every such call, it advances a local progress value and a two-word position in the context, including when the dependency returns a nonzero result.

The established behavior consists of field accesses, dependency calls, signed comparisons, and 32-bit arithmetic. Describing the destination as a serialized record, the position as an offset, or the repeated calls as output submissions is inferred intent, not an established protocol.

The function returns zero when its signed progress condition is satisfied. Otherwise, it returns the first nonzero result from F-87fe98a22e06, after performing that iteration’s position update.

## Interface

### Input and result

The function receives one 32-bit address-like argument identifying a mutable context.

There is no visible validation of that address, nested handles, destination capacity, alignment, step, or limit. The context must permit the accesses described below, including a four-byte access beginning at offset `0xe4`. Dependencies may require additional valid storage.

The result is a 32-bit integer:

- Zero if the current signed limit is less than or equal to local signed progress.
- Otherwise, the exact nonzero result returned by F-87fe98a22e06.

A zero result does not imply that a repeated call occurred. Preparation precedes the first termination comparison, so a nonpositive limit can produce a zero result after preparation without any F-87fe98a22e06 call.

### Context layout

Offsets are byte offsets from the supplied context address. Descriptive field names do not establish application-level types.

| Offset | Width | Established use |
|---|---:|---|
| `0x0b` | 8 bits | Zero/nonzero flag controlling a status lookup. |
| `0x24` | 32 bits | Value passed to a destination-field dependency. |
| `0x34` | 32 bits | Storage passed by address to F-e219d7c81ff5, then read for a destination-field call. |
| `0x40` | 32 bits | Initial step candidate and conditional destination-field source. |
| `0x6c` | 32 bits | Argument to F-8497b1f74739. |
| `0x70` | 32 bits | First argument to each F-87fe98a22e06 call. |
| `0x98` | 32 bits | Low word of the position. |
| `0x9c` | 32 bits | High word of the position. |
| `0xa0` | 32 bits | Snapshot low word. |
| `0xa4` | 32 bits | Snapshot high word. |
| `0xa8` | 32 bits | Conditionally initialized saved low word. |
| `0xac` | 32 bits | Conditionally initialized saved high word. |
| `0xc0` | 32 bits | Initial step candidate, destination-field source, and repeatedly read signed limit. |
| `0xe4` | 32 bits | Destination base captured at entry. |

### Destination and global input

The captured destination base is passed to an unmapped external dependency and to every F-87fe98a22e06 call. Destination-relative addresses are passed to F-cfb711b78946 at offsets `0x08`, `0x0c`, `0x10`, `0x14`, and conditionally `0x18`.

The unmapped external dependency receives three arguments:

1. The destination base captured at entry.
2. A value read from a global data symbol at that call site.
3. The numeric value eight.

The second argument is **not established to be a fixed source address**. Its value, type, width, immutability, and interpretation are unresolved. The read occurs after F-436187425b01 and the snapshot writes; an entry-time value must not be substituted for it.

Each value supplied as the second argument to F-cfb711b78946 is 32 bits. That dependency’s actual write width, byte order, bounds requirements, and alignment requirements are unknown.

## Behavior

### Entry capture and step selection

The function first reads and retains the destination base from `0xe4`.

It then reads an initial step candidate from `0x40`. A signed 32-bit comparison of the fields at `0xc0` and `0x40` determines whether to replace that candidate with another read from `0xc0`.

With stable field contents, the selected step is the signed minimum of the two values. A high-bit-set value therefore compares as negative and can be selected instead of a small positive value.

The selected step remains fixed for the invocation. Later changes to `0x40` or `0xc0` do not recompute it.

The selection contains repeated field accesses. It does not establish a coherent snapshot under concurrent mutation. In particular, the initially read candidate need not match the value of `0x40` used in the comparison if another actor changes the field.

### Saved-position initialization

The function reads the saved high word at `0xac`. Only if that word is zero does it test the saved low word at `0xa8`.

When both are zero, it performs these copies in order:

1. Read `0x98` and write that value to `0xa8`.
2. Read `0x9c` and write that value to `0xac`.

If either tested saved word is nonzero, neither saved word is written by this initialization.

The all-zero pair is an initialization condition, not a separate validity flag. A legitimately saved zero position remains indistinguishable from the condition that triggers initialization on a later invocation.

### Preparation and snapshot

F-436187425b01 is called once with the context address.

After it returns, the function:

1. Reads the current low position word at `0x98` and writes it to `0xa0`.
2. Reads the current high position word at `0x9c` and writes it to `0xa4`.

Thus, saved-position initialization uses pre-preparation values, while these snapshot writes use post-preparation values.

The supplied evidence provides no preservation contract for F-436187425b01. It may affect context fields, destination-accessible storage, or globals. In particular, it must not be assumed to preserve the global later used as the unmapped external call’s second argument.

### External call using a global value

After both snapshot writes, the function reads the global data value used as the unmapped external dependency’s second argument and makes the three-argument call described in the interface.

This is a call-site global read, not an entry capture and not evidence of an immutable constant. The value supplied is the value obtained at this later point, subject to any earlier effects and applicable memory semantics.

Copying an eight-byte prefix is plausible intent because of the argument arrangement and the numeric value eight. Neither copying, a source-pointer interpretation, nor an eight-byte memory effect is established without the dependency’s contract and the global’s definition.

### Marker and destination-field calls

After the unmapped external call returns, the function tests the byte at `0x0b`.

- If nonzero, the marker is `0xffffffff`, and F-8497b1f74739 is skipped.
- If zero, F-8497b1f74739 receives the current word at `0x6c`.
  - A result with bit `0x200` clear produces marker zero.
  - A result with bit `0x200` set produces marker `0xffffffff`.

F-cfb711b78946 then receives the captured destination plus `0x08` and that marker.

The following calls occur in order:

1. F-e219d7c81ff5 receives the numeric value four and the address of context offset `0x34`.
2. F-cfb711b78946 receives destination plus `0x0c` and the then-current word at `0x34`.
3. F-cfb711b78946 receives destination plus `0x10` and the then-current word at `0x24`.
4. F-cfb711b78946 receives destination plus `0x14` and the then-current word at `0xc0`.

The function next tests `0xa4`, testing `0xa0` only if `0xa4` is zero. If both are zero, F-cfb711b78946 receives destination plus `0x18` and the then-current word at `0x40`.

Otherwise, this final destination-field call is omitted. No direct clearing or replacement of destination offset `0x18` occurs on the omitted-call path.

These field reads occur after preceding dependency calls. The zero-pair test therefore need not observe the values originally written by the snapshot operations, and the conditional value from `0x40` need not equal the selected step.

### Repeated call and progress comparison

Local 32-bit progress starts at zero.

Before each F-87fe98a22e06 call, the function rereads `0xc0`. Both that limit and local progress are compared as signed 32-bit values.

If the limit is less than or equal to progress, the function immediately returns zero.

Otherwise, F-87fe98a22e06 receives:

1. The current word at `0x70`.
2. The destination captured at entry.
3. The fixed selected step bit pattern.

The function does not reduce the step to a remaining amount. Positive progress can overshoot a positive limit.

### Updates before result handling

After F-87fe98a22e06 returns, the function reads the current low position word at `0x98`. It then:

- Adds the selected step to local progress, retaining the low 32 bits.
- Writes the low 32 bits of the low-position-plus-step sum to `0x98`.
- Reads the high position word at `0x9c` and writes its adjusted value back to `0x9c`.

The high-word adjustment consists of:

- Zero when the step’s high bit is clear, or minus one when it is set.
- Plus one if the unsigned low-word addition carries beyond 32 bits.

All position-word results retain their low 32 bits. For a stable position during these direct accesses, this is equivalent to adding the signed 32-bit step, sign-extended to 64 bits, to the combined position modulo \(2^{64}\).

The low-word write precedes the high-word read and write. No atomic two-word update is established. The position being advanced is the position left by the dependency, not necessarily the position passed into the iteration.

Only after these updates is the dependency result tested:

- A nonzero result is returned unchanged.
- A zero result leads to another signed limit comparison.

## Side effects

### Direct writes

The visible body can write these context fields, in the stated order within each group:

- `0xa8`, then `0xac`, when the saved pair passes the zero test.
- `0xa0`, then `0xa4`, after F-436187425b01 returns.
- `0x98`, then `0x9c`, after every F-87fe98a22e06 call, including calls returning nonzero.

There is no visible rollback.

### Dependency effects and ownership

Dependencies receive the context, an address inside it, destination-relative addresses, context-derived values, and a value obtained from global storage. Their memory effects, global effects, allocation behavior, release behavior, retention, and ownership rules are not established.

No direct allocation, release, or explicit ownership transfer appears in this body. That absence does not establish that dependencies preserve or retain any particular storage.

The destination and step are captured early. Other inputs remain live:

- The external-call global is read after preparation and snapshot writes.
- The limit at `0xc0` is reread for each loop comparison.
- The handle at `0x70` is reread for each repeated call.
- Destination-field values are read at their respective call sites.
- Position words are read after the repeated dependency returns.

Aliasing among the context, destination, global storage, and dependency-accessible storage is not characterized. No non-aliasing or synchronization guarantee should be added.

### Errors and termination

Only the result of F-87fe98a22e06 is explicitly propagated. No visible result-based error handling is performed for other dependencies.

There is no explicit retry limit, cancellation check, timeout, or rejection of zero or negative steps.

A zero step with a persistently positive limit and zero-returning repeated calls makes no local progress. Negative steps and 32-bit wraparound also invalidate ordinary monotonic-progress reasoning. Termination is not guaranteed for arbitrary inputs and dependency effects.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-436187425b01 | Receives the context before snapshot writes and the later global read. | Context/global mutation, ownership, failure, and nonlocal effects. |
| F-8497b1f74739 | Conditionally receives `0x6c`; result bit `0x200` controls the marker. | Handle meaning, other result bits, and side effects. |
| F-cfb711b78946 | Receives destination-relative addresses and 32-bit values in the specified order. | Encoding, write width, bounds, ownership, and failure behavior. |
| F-e219d7c81ff5 | Receives four and the address of `0x34`, which is read afterward. | Transformation, meaning of four, and other effects. |
| F-87fe98a22e06 | Receives the current handle, captured destination, and fixed step. | Submission semantics, partial completion, mutation, retention, and result meanings. |

One additional external dependency has no supplied opaque ID. Its established interaction is the ordered call with captured destination, a call-site global value, and eight. Its identity and implementation remain documented_only; no familiar library operation is substituted for it.

## Acceptance examples

These are conditional consequences of the static behavior, not observed executions. Unless otherwise stated, dependencies return normally and do not modify relevant storage; direct accesses are not concurrently disturbed.

### Positive progression with overshoot

With `0x40 = 4`, stable `0xc0 = 10`, and all repeated calls returning zero:

- The step is four.
- Calls occur at local progress zero, four, and eight.
- Progress becomes twelve after the third call.
- The next comparison returns zero.
- The position advances by twelve overall.

There is no final call with step two.

### Preparation changes the global argument

Suppose the external-call global initially contains one value, and F-436187425b01 replaces it with another. If it remains unchanged thereafter, the unmapped external dependency receives the replacement value.

The global read follows the writes to `0xa0` and `0xa4`. The function does not pass an entry-captured global value. This example does not assign a type or pointer meaning to either value.

### Nonpositive limit

With stable `0xc0 = 0`, preparation, snapshot writes, the global-argument external call, and applicable destination-field calls occur before the function returns zero without calling F-87fe98a22e06.

A stable negative signed limit likewise prevents the first repeated call.

### Nonzero repeated-call result

With a positive limit, step four, and the first repeated call returning a nonzero result:

- Exactly one repeated call occurs.
- The position left by that call advances by four.
- The nonzero result is returned.
- No next limit comparison occurs.

This does not establish how much external work the dependency completed.

### Saved pair and snapshot timing

If both saved words are zero and preparation changes the position from five to nine, the saved pair receives five and the snapshot receives nine.

If the saved high word is zero but its low word is seven, neither saved word is initialized.

### Carry and signed extension

For post-call position `0x00000007:0xfffffffe` and step three, the updated position is `0x00000008:0x00000001`.

For post-call position zero and step `0xffffffff`, the updated position is `0xffffffff:0xffffffff`.

With stable `0x40 = 0xffffffff` and `0xc0 = 5`, signed selection chooses that minus-one step. A nonnegative, increasing-progress interpretation is invalid.

### Marker and conditional field

A nonzero flag at `0x0b` selects marker `0xffffffff` without a status lookup. With a zero flag, lookup results zero and `0x200` select markers zero and `0xffffffff`, respectively.

If either snapshot word is nonzero when the later zero-pair test occurs, the destination-plus-`0x18` call is omitted. This does not establish the resulting contents of that destination location.

## Uncertainties

### Inferred intent

The two-word position may represent an offset or count. The saved and snapshot pairs may identify processing stages. The destination-relative calls may construct a serialized record, and repeated calls may emit or process it.

The unmapped external call may copy an eight-byte prefix. That interpretation requires unprovided facts about both the callee and the global-derived argument. It is not an established memory operation.

### documented_only

The supplied evidence does not establish:

- The external-call global’s value, width, type, immutability, or interpretation.
- Whether any dependency modifies that global.
- The unmapped external dependency’s identity or memory effects.
- Destination capacity, encoding, alignment, or byte order.
- Dependency preservation, ownership, retention, allocation, or release rules.
- Application-valid ranges or units for steps, limits, handles, and positions.
- Caller restrictions on aliasing or concurrent mutation.
- Exception, fault, nonlocal-exit, or invalid-memory behavior.
- Whether a nonzero repeated-call result denotes failure, cancellation, or another condition.

## Evidence

This contract is based solely on the supplied static function body and opaque dependency mapping.

Established evidence includes context access widths and offsets, short-circuit tests, dependency order, the global read after preparation and snapshot writes, signed comparisons, captured versus reread inputs, carry-based position arithmetic, and updates preceding repeated-call result handling.

The global’s definition, dependency implementations, and external memory contents were not supplied. No runtime tests, executions, or memory observations have been performed.
