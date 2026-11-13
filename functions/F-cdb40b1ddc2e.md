<!-- Trusted provenance: F-cdb40b1ddc2e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cdb40b1ddc2e

## Purpose

F-cdb40b1ddc2e performs an ordered sequence of dependency calls, scalar and bit-field forwarding, low-address memory reads, record writes, and count-dependent state changes.

No application-level purpose is established. The visible operations do not justify identifying the function as a formatter, decoder, validator, allocator, or object constructor.

Both represented terminal paths end at an instruction-interpretation boundary rather than an ordinary return. Normal completion is therefore not established.

This contract distinguishes three evidence classes:

- **Established static evidence:** operations and ordering directly represented in the supplied material, conditional on execution reaching them.
- **Inferred intent:** possible interpretations that are not behavioral requirements.
- **documented_only:** unresolved behavior for which the supplied material does not support an executable guarantee.

The supplied material includes a warning about invalid instruction data. The unusual calls and low-address accesses must be retained as static evidence, but their correspondence to executable machine behavior remains unresolved.

## Interface

### Explicit input

The represented interface accepts one opaque 32-bit argument. The first dependency call receives its bit pattern in two argument positions.

No pointer, length, handle, signed-integer, or ownership interpretation is established for this argument.

### Implicit inputs

The function also depends on machine state not declared as explicit parameters:

| Implicit input | Represented use |
|---|---|
| Control-flow-related entry value | Derives the initial indirect-call target |
| Unsigned 32-bit entry value | Supplies bit 31 to an early dependency |
| Signed 32-bit entry value | Multiplied by two to form the marker-store address |
| Another 32-bit entry value | Participates in unsigned subtraction and high-byte extraction for the source-read base |
| Additional call-result state | Supplies two later 32-bit writes and one dependency argument |
| Stack addresses | Supply a truncated, adjusted halfword value |

The initialization, provenance, ABI meaning, and validity of these inputs are **documented_only**.

Two distinct fixed-storage words are read late in execution:

- A 32-bit base-address value, captured before the record-consuming call.
- An unsigned 32-bit count, captured after that call returns.

Neither storage word has an established constant value.

### Widths and representation

| Quantity or operation | Width |
|---|---:|
| Explicit argument, ordinary captured results, record words | 32 bits |
| Selected wide results | 64 bits |
| Selected memory reads and count-byte write | 8 bits |
| Marker and stack-address-derived writes | 16 bits |
| Captured destination base and unsigned count | 32 bits |

Splitting a wide result into low and high 32-bit halves is a numerical operation. It does not establish memory endianness.

Pointer-shaped values are represented using 32-bit quantities. Conversion of negative signed values or numeric addresses to actual storage locations is not independently specified.

### Output, errors, and preconditions

No return value is declared, and no ordinary return is visible. Observable outputs are calls, memory accesses, stack-state changes, and the represented terminal boundaries.

Reaching any later operation requires earlier calls to return and earlier accesses to complete. No bounds checks, alignment checks, permission checks, error codes, recovery mechanism, or rollback are shown. In particular, successful access to low numeric addresses is a prerequisite, not a guarantee.

## Behavior

### Ordering model

All following requirements are conditional on execution reaching the relevant point along the represented path.

A dependency invocation shown without explicit arguments may still consume registers, stack state, or shared memory. “No explicit arguments” is not a purity or ABI guarantee. Likewise, “no captured result” means only that the representation does not assign that call’s primary result to a value subsequently used here.

Calls with uncaptured results remain observable events. They must not be removed, merged with neighboring calls, or treated as interchangeable.

### Opening calls

The opening sequence has these observable relationships:

- The explicit input is duplicated into the initial call’s two represented arguments.
- One no-explicit-argument call follows before bit 31 of the implicit unsigned entry value is passed to another call.
- A separate no-explicit-argument call then produces a captured 32-bit result, which is duplicated into two argument positions.
- Two calls with no explicit arguments and no captured results separate that duplication from the next 32-bit producer.
- That next result is duplicated. Three further no-explicit-argument calls with uncaptured results precede another producer.
- That producer’s result is duplicated. Two further no-explicit-argument calls with uncaptured results precede the signed-selector producer described below.

The repeated captures are independent. Their equality is not established.

### Scalar, selector, and wide-result groups

The following table specifies the successive groups. Each group has a producer, an initial forwarding call, one intervening no-explicit-argument call with no captured result, and then a derived-value consumer.

The “separate duplicate-result stage” is important: where present, it consists of a **new subsequent call with no explicit arguments that produces a captured 32-bit result**, followed by a call receiving that result twice. The preceding derived-value consumer has **no captured result**.

The last column counts additional no-explicit-argument calls with uncaptured results after the listed stages and before the next producer.

| Group, in order | Initial forwarding | Argument to derived-value consumer | Separate duplicate-result stage | Following uncaptured calls |
|---|---|---|---|---:|
| Signed 32-bit selector | Captured value once | Byte selected by its sign | Absent | 2 |
| First bit-24 group | Captured 32-bit value once | Bit 24 | Present | 2 |
| Second bit-24 group | Captured 32-bit value once | Bit 24 | Present | 4 |
| First bit-29 group | Captured 32-bit value once | Bit 29 | Present | 4 |
| Second bit-29 group | Captured 32-bit value once | Byte selected by bit 29 | Absent | 2 |
| Unsigned high-half predicate | Wide result’s low half twice | Predicate from the high half | Present | 2 |
| Signed high-half selector | Wide result’s low half twice | Byte selected by a signed high-half expression | Absent | 2 |
| First wide-sign group | Wide result’s low half once | Bit 63 | Present | 2 |
| Second wide-sign group | Wide result’s low half once | Bit 63 | Present | 4 |
| Bit-22 group | Captured 32-bit value once | Bit 22 | Present | 2 |

Thus, in each bit-24 group, the first bit-29 group, the unsigned high-half-predicate group, both wide-sign groups, and the bit-22 group, the extraction consumer and the later producer of the duplicated value are distinct calls. Their results and effects cannot be conflated.

All byte selections occur at the derived-value consumption point, after the intervening call. An earlier forwarding call does not imply an earlier byte read.

### Extraction and selection rules

Bits 24, 29, and 22 are individual bits of their respective captured unsigned 32-bit values. Each derived bit argument is exactly zero or one. The masks used in their extraction do not produce multi-bit fields.

The signed 32-bit selector reads one byte from numeric address:

- `0x10` for a nonnegative captured value.
- `0x11` for a negative captured value.

This selection follows the represented arithmetic sign-shift interpretation.

The second bit-29 group reads one byte from `0x10` plus bit 29, again selecting `0x10` or `0x11`.

For the unsigned high-half predicate, let \(h\) be the wide result’s high 32 bits. The consumer receives:

\[
\operatorname{bit}_{31}\big((h-7)\bmod 2^{32}\big).
\]

The subtraction is unsigned 32-bit arithmetic and the following shift is logical. The predicate is one for:

- \(h=0\) through \(6\); or
- \(h=\texttt{0x80000007}\) through `0xFFFFFFFF`.

It is zero for the intervening range. It is not an unrestricted unsigned “less than seven” comparison.

The signed high-half selector instead interprets the high half as a signed 32-bit value and subtracts seven before an arithmetic sign shift. Where subtraction is representable:

- A signed high half below seven selects byte address `0x11`.
- A signed high half at least seven selects byte address `0x10`.

Subtraction is not representable for signed high halves from \(-2^{31}\) through \(-2^{31}+6\). Behavior for those inputs is **documented_only**; wrapping subtraction is not established.

Each wide-sign group forwards bit 63 of its own captured 64-bit result. This is zero or one regardless of the low half.

No contents are established for byte addresses `0x10` and `0x11`. They need not contain Boolean constants, and intervening dependencies may change them.

### Marker store and nibble forwarding

After the bit-22 group and its following calls, another no-explicit-argument call produces a captured 32-bit value.

Before forwarding that result twice, the function writes the 16-bit value `0x002F` to an address calculated by multiplying an implicit **signed 32-bit value by two**.

The multiplication is representable only when its input lies in:

\[
[-2^{30},\;2^{30}-1].
\]

For inputs outside that range, signed multiplication overflows. The resulting address and execution behavior are **documented_only**. The contract does not assign wrapping, saturation, or unrestricted mathematical-address semantics to the calculation. Even for representable products, address validity and alignment remain unresolved.

After the marker write:

- A dependency receives the previously captured 32-bit result twice.
- A no-explicit-argument producer supplies a 32-bit value to a single-argument consumer.
- That consumer produces a further captured 32-bit value whose high nibble is passed to F-2cc1255261f3.

The nibble argument is in the inclusive range zero through fifteen. No explicit use of F-2cc1255261f3’s return value is shown.

Before that call, the function calculates the later source-read base as:

\[
b=((s-51)\bmod 2^{32})\mathbin{\text{logical-right-shift}}24,
\]

where \(s\) is the bit pattern of another implicit entry value. The unsigned subtraction makes \(b\) a numeric address from zero through 255, not a retained pointer into the original value.

After F-2cc1255261f3 returns, three producer/consumer pairs occur in order. Each producer has no explicit arguments and supplies its captured 32-bit result to the immediately following single-argument consumer. No primary result is captured from those consumers.

### Source reads and record construction

After the third pair, four 32-bit source words are read in this order:

1. Base \(b\), offset 0.
2. Base \(b\), offset 4.
3. Base \(b\), offset 8.
4. Base \(b\), offset 16.

There is no explicit source read at offset 12.

All four reads precede the following four ordered 32-bit writes:

| Destination address | Record offset | Value |
|---|---:|---|
| `0xC3` | 0 | Additional 32-bit call-result state |
| `0xC7` | 4 | Record address `0xC3` |
| `0xCB` | 8 | Captured source word from offset 8 |
| `0xCF` | 12 | Captured source word from offset 16 |

Source/destination overlap therefore does not change the already captured source words. These word accesses are not naturally aligned at the record destination; support for them is unresolved.

The function next captures the 32-bit destination base from fixed storage, then invokes F-U004 with four represented arguments:

1. Source word from offset 0.
2. The additional call-result-state value written at record offset 0.
3. Record address `0xC3`.
4. Source word from offset 4.

The destination base is captured **before** F-U004. Later direct writes use that captured value even if the dependency changes the fixed storage from which it came.

### Count-controlled terminal phase

After F-U004 returns, the function reads the unsigned 32-bit count into stack-resident state. It then writes a second additional 32-bit call-result-state value at captured destination base plus `0x68`.

Only after that word write does the count comparison control the terminal path.

**Count greater than eight:** execution reaches the first invalid-instruction boundary. No subsequent byte write, halfword write, wide producer, or final consumer is represented on this path.

**Count zero through eight inclusive:** F-U005 produces a 64-bit result. The subsequent represented effects occur in this order:

- The count’s low eight bits are written at captured base plus `0x0F`.
- A 16-bit stack-address-derived value is written at numeric address count plus `0x28`.
- A 32-bit stack slot receives record address `0xC3` shifted logically right by fourteen, therefore zero.
- Another 32-bit stack slot receives the captured source word from offset 16.
- F-U006 receives the wide result’s low 32 bits, high 32 bits, and `0x74`, in that order.
- Execution reaches the second invalid-instruction boundary.

The halfword value uses the address of the stack slot later assigned zero, not that slot’s contents. Under the represented truncation, it is the low 16 bits of that address plus `0x398`, reduced to the stored 16-bit result. Actual stack addressing and conversion semantics remain ABI-dependent.

## Side effects

Established direct effects comprise:

- Three selected byte reads, subject to the signed-overflow qualification.
- One 16-bit marker write, subject to the signed-multiplication qualification.
- Four ordered source-word reads.
- Four ordered record-word writes.
- Reads of the fixed-storage destination base and count.
- A destination-base-relative 32-bit write on both count paths.
- On the count-at-most-eight path, a byte write, a halfword write, and stack-state assignments.
- The dependency calls and represented terminal boundaries.

No allocation, deallocation, ownership transfer, locking, or transactional behavior is explicitly established.

Passing the record address does not establish whether a dependency borrows, modifies, retains, or releases the storage. Dependencies are not assumed to preserve globals, pointed-to memory, or undeclared machine state. Directly captured values and explicit read timing must be distinguished from unknown effects inside dependencies.

## Dependencies

Only F-2cc1255261f3 has a supplied independent opaque identity. Other labels below identify unresolved roles, not proven unique functions.

| Opaque ID | Role |
|---|---|
| F-U001 | Initial entry-state-derived indirect target |
| F-U002 | Early indirect producers, forwarding calls, intervening calls, and extraction consumers |
| F-U003 | Later alignment-derived indirect producers and consumers |
| F-2cc1255261f3 | Receives the extracted high nibble |
| F-U004 | Four-argument record consumer |
| F-U005 | Count-at-most-eight wide-result producer |
| F-U006 | Final three-argument consumer |

Sites within one role may resolve to different functions; different roles may alias. No complete dependency ABI, ownership contract, error behavior, preservation guarantee, or return guarantee is supplied.

## Acceptance examples

These are conditional static expectations, not executed tests.

- **Input duplication:** An explicit input of `0x89ABCDEF` appears unchanged in both represented argument positions of the initial call.

- **Distinct extraction and producer calls:** Suppose a bit-24 producer returns `0x01000000`, and the separate no-explicit-argument producer after the bit consumer returns `0x12345678`. The bit consumer receives one. The later duplication call receives `0x12345678` twice, not a presumed result of the bit consumer.

- **Individual-bit behavior:** A bit-24 producer returning `0x02000000` supplies zero to its bit consumer. Neighboring high bits do not make the result a multi-bit field.

- **Signed byte selection:** A captured signed value of negative one selects byte address `0x11`; zero selects `0x10`. The byte is read after the intervening dependency call.

- **Unsigned high-half boundaries:** High halves six and seven produce predicates one and zero. `0x80000007` produces one, disproving replacement by a general unsigned less-than-seven test.

- **Signed high-half underflow:** A high half representing \(-2^{31}\) falls outside the established subtraction domain. No byte-address outcome is required for that case.

- **Marker arithmetic:** A marker input of three gives a representable product of six. If address six supports the access, it receives halfword `0x002F`. An input of `0x40000000` exceeds the representable multiplication range; a wrapped address is not specified.

- **Nibble forwarding:** A nibble-producing result of `0xA1234567` causes F-2cc1255261f3 to receive ten.

- **Read-base wraparound:** An implicit subtraction input of 50 produces unsigned difference `0xFFFFFFFF` and source base 255.

- **Overlapping storage:** If source words overlap the record destination, all four source reads still occur before the first record write.

- **Base mutation:** If F-U004 changes the fixed-storage base, subsequent direct state writes still use the base captured before that call.

- **Count nine:** The word write at base plus `0x68` precedes the terminal boundary. The later byte and halfword writes do not occur on this represented path.

- **Counts eight and zero:** Their halfword destinations are `0x30` and `0x28`, respectively. Both take the wide-producer/final-consumer path; zero has no special bypass.

## Uncertainties

The following remain **documented_only**:

- Whether the supplied instruction interpretation matches executable behavior.
- Application-level purpose and ordinary completion behavior.
- Complete dependency identities, calling conventions, and hidden inputs.
- Provenance of implicit entry state and additional call-result state.
- Out-of-range signed multiplication in the marker address.
- Signed underflow in the high-half selector.
- Actual address mapping, pointer conversion, endianness, alignment support, and permissions.
- Dependency effects on storage, globals, ownership, and control flow.
- Precise stack layout and external observability of stack assignments.
- Behavior beyond either invalid-instruction boundary.

These unknowns are not permission to remove represented operations or substitute safer-looking arithmetic.

## Evidence

This contract uses only the supplied static representation and opaque dependency mapping. No runtime tests have been performed.

Static evidence establishes the represented widths, extraction rules, distinct call captures, intervening-call order, memory-access order, record layout, unsigned count comparison, and signed arithmetic expressions.

In particular, the extraction consumers in the duplicated-result groups have no captured result; separate subsequent no-explicit-argument producers supply the duplicated values. The marker-address multiplication uses a signed 32-bit operand, so out-of-range products are not established wrapping addresses.

No higher-level intent or dependency preservation behavior is promoted to a requirement.
