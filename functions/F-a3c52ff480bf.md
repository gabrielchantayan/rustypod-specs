<!-- Trusted provenance: F-a3c52ff480bf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a3c52ff480bf

## Purpose

F-a3c52ff480bf constructs a byte-oriented numeric-formatting fragment and updates a caller-supplied formatting record. Its result consists of the returned signed length, buffer writes, and record mutations. The returned length alone does not describe all bytes accessed or written.

Established behavior includes descriptor selection, four opaque special-result payloads, three selector-dependent construction modes, insertion of a context-derived separator, insertion of marker bytes `0x3c` and `0x3e`, and conditional shortening of the reported fragment. Some paths delegate completion and return the dependency’s result unchanged.

Floating-point formatting, exponent handling, and deferred expansion of zero runs are inferred intent. The supplied evidence does not establish the value’s numeric encoding, the meanings of the descriptors, or the downstream interpretation of marker bytes.

## Interface

### Arguments and return

| Argument | Observed representation | Role |
|---|---|---|
| Selector | Signed 32-bit integer | Selects construction mode, special payload, and finalizer selector. |
| Buffer | Writable byte pointer | Receives dependency output and subsequent construction writes. |
| Value | Opaque 32-bit quantity | Forwarded unchanged to classification and digit generation. |
| State | Address in the observed 32-bit address model | Supplies flags and precision; receives descriptors, counters, and conditional updates. |

The return is a signed 32-bit integer. Direct-return paths report a constructed length, sometimes reduced by one. Delegated paths propagate the finalizer’s return without validation or adjustment.

### State layout

| Byte offset | Width | Meaning used in this contract |
|---|---:|---|
| `+0x08` | 32 bits | Opaque descriptor pointer |
| `+0x0c` | 32 bits | Leading counter |
| `+0x10` | 32 bits | Trailing counter |
| `+0x18` | 32 bits | Flags |
| `+0x38` | 32 bits | Signed precision-like quantity |

“Precision” is a descriptive name, not an established numeric-format specification.

The flags are captured before any dependency call. Subsequent flag decisions use this snapshot. Precision and counters are instead read from the state at their respective use sites. Entry values therefore do not necessarily determine later behavior.

Neither counter is initialized on every path. An untouched counter retains its then-current contents, including possible dependency changes.

### Preconditions

There is no capacity argument or bounds check. Storage must cover dependency writes, shifts, sentinel-like reads, and accesses beyond the reported fragment. Ordinary paths do not guarantee zero termination.

State and context-derived addresses must be valid for their accesses. Digit generation must supply the metadata and buffer bytes subsequently consumed. Its exact initialization and capacity obligations are unavailable.

The descriptions below assume arithmetic is representable and referenced memory is valid. They do not establish safety for arbitrary negative precision, extreme metadata, overlapping buffer/state storage, or inconsistent dependency results. Where storage overlaps, earlier writes may affect later reads; no alias isolation is implemented.

## Behavior

### Shared setup and observation order

The function first captures the 32-bit flags, then calls F-187ab9ba4eb6. From the returned context it obtains the separator:

- Context offset `+0x2c` supplies an object pointer.
- The beginning of that object supplies a 32-bit displacement.
- The byte at object address plus displacement becomes the separator.

The supplied expression references the context’s pointer field more than once; it does not establish a single-load guarantee or a complete evaluation order within that expression. The separator is nevertheless captured before classification and is not refreshed later.

Next, F-5a77fe92b299 receives the value and an address for a 32-bit auxiliary result. The function selects and stores a descriptor at state offset `+0x08`:

| Condition, in precedence order | Selected descriptor |
|---|---|
| Auxiliary result nonzero | Dedicated nonzero-result descriptor |
| Auxiliary result zero; snapshot flag `0x02` set | Bit-`0x02` descriptor |
| Neither preceding condition; snapshot flag `0x04` set | Bit-`0x04` descriptor |
| Otherwise | Default descriptor |

Descriptor contents and ownership are unknown.

Classification and context acquisition precede every precision normalization or generation request. No preservation of precision, counters, or other accessible state is assumed across these calls.

### Special-result construction

Any nonzero classifier return selects special construction. Four opaque three-byte payloads are distinguished by:

- Classifier return exactly `1`, versus another nonzero value.
- Signed selector below `0x61`, versus at least `0x61`.

The selected payload is copied in order to buffer offsets `0`, `1`, and `2`. Offset `3` receives zero. The function then writes the flags snapshot with bit `0x10` cleared to state offset `+0x18` and returns `3`.

This flags write can overwrite intervening dependency changes. Digit generation and finalization are not called. Payload meanings, including any association with infinity or NaN, are not established.

### Normal-path generation

Normal classification leads to one call to F-da69ec178db1. It receives:

- An address for local metadata.
- Buffer offset `1` as the output start.
- The unchanged value.
- A requested count.
- Mode `0` or `1`.

Construction subsequently consumes two signed 32-bit metadata quantities, called **position** and **generated count** here. Their apparent exponent-like and length-like roles are inferred. The dependency contract establishing both outputs, their layout, and their valid ranges is unavailable.

### Scientific-style construction: `E` and `e`

After classification, precision above `17` is replaced with `17`; the former precision minus `17` is stored in the trailing counter. Otherwise this step leaves both fields unchanged.

Generation receives mode `0` and the precision read for the request plus one.

After generation, the working length becomes generated count plus one, and buffer byte `1` is copied to byte `0`. The function then reads precision again:

- Nonzero precision, or snapshot flag `0x08` set: byte `1` receives the separator.
- Zero precision and that flag clear: working length becomes exactly `1`.

A positive trailing counter, read after these operations, causes `0x3e` to be written at the working end and increases length by one. This counter is consulted even when the function did not initialize it.

F-e8a83ee60e09 then receives the buffer, working length, position, and original selector. Its return is propagated unchanged.

A generation-time precision or counter mutation can therefore change construction despite the earlier cap or request.

### Fixed-style construction: `F` and `f`

Generation receives mode `1` and the precision read for its request, without a local cap. After generation, byte `0` receives ASCII zero, `0x30`. Subsequent precision-dependent decisions use state reads after generation.

#### Negative position

Let the negative position’s representable negation be its magnitude.

When magnitude exceeds the precision read for this comparison plus one, the function writes the separator at byte `1`, sets length to `2`, and copies the then-read precision into the trailing counter. No shifting or leading-marker insertion occurs in this short form.

Otherwise, provisional length is precision plus position plus two. If generated count plus one is smaller, the trailing counter receives the difference and length becomes generated count plus one.

The prefix is then constructed by backward copying:

- Magnitude `1`: source indices `1` through the working length move right by one. Byte `0` becomes ASCII zero, byte `1` becomes the separator, and length increases by one.
- Greater magnitude: that same source range moves right by two. Bytes `0`, `1`, and `2` become ASCII zero, separator, and `0x3c`; length increases by two. The leading counter becomes magnitude minus one.

These ranges include the byte at the pre-shift working end. Nonpositive loop bounds perform no corresponding copies.

After either negative-position construction, a positive trailing counter causes `0x3e` insertion at the working end and a one-byte length increase. A stale positive counter can therefore affect output where this path did not assign it.

#### Nonnegative position

Provisional length is precision plus position plus two.

If it is below `19`, source indices `1` through position plus one move left by one, in ascending order. The separator is written at position plus one. The function does not append a trailing marker on this branch.

Otherwise, initial working length becomes `18`:

- **Position below `17`:** the same leftward movement and separator insertion occur. The trailing counter becomes precision plus position minus `16`. Any nonzero result, not merely a positive result, causes `0x3e` insertion at index `18` and increases length to `19`.
- **Position at least `17`:** source indices `1` through `17` move left by one. Index `17` receives `0x3c`; the leading counter becomes position minus `16`. Length becomes `19`, and index `18` receives the separator. Nonzero precision causes that precision to be copied into the trailing counter and `0x3e` to be appended at index `19`, increasing length to `20`. Zero precision leaves the trailing counter untouched and appends no marker.

#### Fixed-style return

A final precision read controls the return together with snapshot flag `0x08`. Nonzero precision or a set flag returns the constructed length. Otherwise the function returns one less.

This shortening neither erases the excluded byte nor writes a terminator. Without an aliasing or external-write guarantee, the final precision read need not equal earlier reads.

### General-style construction: all other selectors

After classification, precision below `1` is replaced with `1`. Generation uses mode `0`, requesting the resulting precision when at most `17`, otherwise `17`. Precision above `17` is not reduced in the state by this request selection.

After generation, byte `0` becomes ASCII zero and working length becomes generated count plus one.

Fixed-style construction is selected exactly when position is less than the precision read **after generation**, and position is greater than `-5`. That route changes the effective selector to lowercase `f`.

#### General fixed construction

For negative position, its magnitude determines a backward shift of source indices `0` through the initial working length, inclusive. Length increases by the magnitude. Bytes `0` through the magnitude, inclusive, become ASCII zero; byte `1` then becomes the separator.

For nonnegative position below generated count, source indices `1` through position plus one move left by one, in ascending order. The separator is written at position plus one.

For nonnegative position at least generated count:

- Source indices `1` through initial working length plus one move left by one, in ascending order.
- The separator is written at generated count plus two.
- Index generated count receives `0x3c`.
- The leading counter becomes position minus initial working length plus two.

This last construction does not increase working length. Its accesses extend beyond the nominal fragment boundary.

#### General scientific construction

When the fixed-selection condition is false, byte `1` is copied to byte `0`, and byte `1` receives the separator. Position remains the generated position for finalization.

#### General trailing treatment

When snapshot flag `0x08` is clear, the trailing counter becomes signed `-1`. The function first compares the byte **at** the working end with the separator.

If that byte differs, immediately preceding ASCII zero bytes are scanned backward and excluded from the length. There is no explicit lower-bound check on this scan. If the last retained byte is the separator, length decreases by one more.

If the byte at the working end equals the separator, zero scanning is skipped; the preceding byte is still checked for separator removal. Consequently, memory outside the nominal fragment can affect the result.

When snapshot flag `0x08` is set, trimming is skipped. Precision read at this stage above `17` causes the trailing counter to become precision minus `17`, followed by `0x3e` insertion at the working end and a one-byte length increase. Otherwise the counter remains untouched.

Effective selector `f` returns this length directly. Other selectors are reduced by two and passed to F-e8a83ee60e09 with the buffer, length, and position. Its result is returned unchanged.

## Side effects

Direct writes comprise descriptor storage, conditional precision updates, conditional counter updates, the special-path flags rewrite, and buffer construction.

Only the special path explicitly appends a zero terminator. Normal construction can read or write beyond the reported length, and shortening does not clear excluded bytes.

No explicit allocation, release, ownership transfer, or error-recovery operation appears. This establishes only direct behavior. Dependencies may allocate, mutate accessible state, retain pointers, or release resources; their contracts are unavailable.

The routine combines captured flags and separator with later state reads. It does not refresh the captured values after dependency calls. No synchronization guarantee is established.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-187ab9ba4eb6 | Called once before classification; returns context used for separator access. | Context lifetime, validity, mutation, ownership, failure behavior. |
| F-5a77fe92b299 | Called once with value and auxiliary-output address; return selects normal or special construction. | Classification meanings, output initialization, accessible-state effects. |
| F-da69ec178db1 | Called once on normal paths with metadata address, buffer offset `1`, value, count, and mode. | Metadata layout, rounding, termination, write extent, state mutation, valid ranges. |
| F-e8a83ee60e09 | Called for scientific completion with buffer, length, position, and selector; return propagated unchanged. | Output syntax, capacity, errors, ownership, further state effects. |

No dependency is assumed to preserve globals or state, initialize unrelated memory, or avoid retaining supplied pointers.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless stated otherwise, examples require valid, nonoverlapping storage and representable arithmetic.

1. **Descriptor precedence.** With classifier auxiliary result zero and both `0x02` and `0x04` set in the captured flags, the descriptor write selects the bit-`0x02` descriptor, regardless of later flags changes.

2. **Special result.** Classifier return `1` with selector `E` selects the below-`0x61`, return-`1` payload. Bytes `0`–`2` receive that payload, byte `3` receives zero, and the return is `3`. The function’s flags write is the captured flags with `0x10` cleared.

3. **Scientific cap.** With selector `e`, precision `20` when the post-classification cap is evaluated, and no subsequent precision or counter mutation by generation, precision becomes `17`, trailing counter becomes `3`, and generation receives count `18`, mode `0`. A trailing marker is inserted before finalization. The example does not require entry precision to have been `20`.

4. **Scientific zero precision.** With selector `E`, precision zero at both the generation-request read and the post-generation separator decision, snapshot flag `0x08` clear, and trailing counter nonpositive at its later read, finalization receives length `1`. The separator-insertion step does not write a separator. Entry precision zero alone is insufficient.

5. **Fixed extreme fractional position.** With selector `f`, precision `2` at the request read and throughout post-generation construction, and generated position `-5`, generation receives count `2`, mode `1`. The short form writes zero and separator at indices `0` and `1`, sets trailing counter to `2`, appends `0x3e` at index `2`, and returns `3`. If generation instead changes precision to `3`, and it remains `3` during construction, the same position selects the short form but stores trailing counter `3`.

6. **General selection boundary.** With precision `6` at the **post-generation selection read**, position `-4` selects fixed construction; positions `-5` and `6` select scientific construction. Neither entry precision nor the generation request establishes this selection-time value.

7. **General normalization and request cap.** Precision zero after classification, at normalization, becomes `1` and yields request `1`. Precision `30` at request selection yields request `17` without a state cap. In the latter case, precision remains `30` after generation only if generation and intervening effects do not modify it.

8. **General trailing precision reread.** With snapshot flag `0x08` set and precision `20` at trailing treatment, the trailing counter becomes `3` and a trailing marker is appended. This criterion does not require a generation request of `17`: the later precision may differ from request-time precision.

9. **Length is not termination.** On a fixed-style path with precision zero at the final return decision and snapshot flag `0x08` clear, the return is constructed length minus one. Acceptance must not require zero at that shortened boundary.

## Uncertainties

The following remain **documented_only**:

- Numeric encoding of the opaque value.
- Descriptor and special-payload contents.
- Metadata layout, initialization, valid ranges, and digit-generation semantics.
- Finalizer syntax, buffer requirements, and failure conventions.
- Marker expansion and required initial counter values.
- Dependency state effects, ownership, release, aliasing, and pointer retention.
- Safe behavior under arithmetic overflow or invalid memory access.

Observed integers and address fields are 32 bits; bytes are 8 bits. Relevant signed arithmetic includes precision increments, precision plus position plus two, generated-count increments, position negation, counter differences, index updates, and selector minus two. Negating the minimum signed 32-bit position is not representable as a positive signed 32-bit integer. The evidence does not justify portable wraparound semantics.

No explicit validation failure or error return is established. Nonzero classification produces formatted special output rather than an error. A finalizer result that might denote failure is propagated without interpretation.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping. Established claims concern visible field widths, accesses, branch conditions, byte movements, dependency arguments, and return propagation.

Formatting terminology, exponent-like interpretation, and deferred-zero intent are inferences, not dependency guarantees. No constant-data contents, dependency bodies, or external callers were available.

No runtime tests have been performed.
