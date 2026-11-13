<!-- Trusted provenance: F-e35a7e01fccc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e35a7e01fccc

## Purpose

The supplied static representation describes an operation dominated by indirect calls and interleaved byte-sized and 32-bit memory writes. Its application-level purpose is not established.

The observable pattern includes:

- Transforming incoming machine state and dependency results into addresses and values.
- Writing to computed locations, including low numeric addresses.
- Passing shifted, masked, or duplicated values to unresolved dependencies.
- Reading selector bytes and using them to select subsequent write destinations.
- Reaching a terminal bad-instruction marker rather than an ordinary return.

**Evidence limitation:** the supplied representation explicitly reports invalid instruction data and truncated control flow. This contract specifies the represented operations conditionally; it does not establish that they are a valid, executable reconstruction of the underlying program.

No allocation, object construction, protocol operation, or ownership transfer can be identified reliably.

## Interface

### Explicit inputs

The represented interface has four positional inputs:

| Input | Represented width | Established direct use |
|---|---:|---|
| First input | 32 bits | Passed to the first indirect dependency. |
| Second input | 32 bits | No direct use appears. |
| Third input | 32 bits | Written once as a full word and twice as its low byte. |
| Fourth input | 32 bits, represented as signed | Shifted left by one; the low byte is written. |

The absence of a direct use of the second input does not establish that it is irrelevant to the real machine-level calling convention.

### Additional incoming state

The representation consumes incoming machine state not accounted for by those four inputs:

- A 32-bit address-like value, called the **root value** below.
- An unsigned 32-bit **selector input**.
- An unsigned 32-bit **base input**.
- An incoming return/control address used to derive the first indirect call target.
- An additional return-register value consumed after a later dependency call.

These are required inputs to the represented behavior. A caller supplying only the four explicit arguments is not sufficient to characterize execution.

### Output and completion

No ordinary return value or successful return path is shown. The visible path ends at an invalid-instruction termination marker.

The final marker does not establish a particular hardware exception, operating-system signal, error code, or nonreturning runtime API.

### Width and notation conventions

For the following conditional operational contract:

- A **byte** is 8 bits.
- A **word** is 32 bits.
- `low8(x)` denotes the least significant eight bits.
- `low32(x)` denotes the least significant 32 bits.
- Right shifts of values described as unsigned are logical.
- A paired dependency result is represented as 64 bits, separated into low and high 32-bit halves.
- Address additions and left shifts are described using 32-bit bit-vector arithmetic.

The last convention is a normalization of the apparent machine-width operations, not proof of the original language’s overflow rules. Signed multiplication and signed left shifts appear in the supplied representation; their portable language-level meaning is not established.

Memory endianness, alignment requirements, address-space type, and the validity of numeric addresses are unknown.

## Behavior

### Dependency naming and sequencing

The dependency identifiers **F-d001** through **F-d095** designate unresolved indirect-call occurrences in execution order. These are occurrence identifiers, not evidence of 95 distinct implementations. Different occurrences may resolve to the same target.

In the sequence below, a call has no explicit arguments unless arguments are stated. This means only that the supplied representation displays no explicit arguments; it does not establish an empty machine-level interface.

Every later step is conditional on all preceding calls returning and all preceding memory accesses completing.

### 1. Initial calls and root writes

Before the first call, derive the **initial base** by logically shifting the base input right by two.

Invoke F-d001 with the first explicit input. Invoke F-d002 through F-d005 in order. Retain the paired result of F-d005 as **initial low** and **initial high**.

Then, in order:

1. Write `low8(root value)` to initial base plus 1.
2. Write initial high as a word to root value plus `0x34`.

Invoke F-d006 with initial low shifted left by two. Invoke F-d007 through F-d010, retaining F-d010’s word result as the **first result**.

### 2. Doubled-root writes and an early shifted result

Derive the **doubled root** as twice the root value.

Write, in order:

1. `low8(doubled root)` to `(selector input >> 22) + 0x11`.
2. `low8(third input)` to doubled root plus `0x0D`.

Invoke F-d011 with two identical arguments, each the first result shifted left by two.

Invoke F-d012 and retain its unsigned word result as the **early result**. Invoke F-d013 with the early result and the early result shifted right by three.

Invoke F-d014 through F-d017, retaining F-d017’s word result as the **second result**.

Then write:

1. `low8(doubled root)` to absolute address `0x11`.
2. `low8(third input)` again to doubled root plus `0x0D`.

These are distinct stores separated from the earlier stores by dependency calls.

### 3. Initial-base fields and a scaled working base

Invoke F-d018 with the second result shifted left by two. Invoke F-d019 through F-d022, retaining F-d022’s unsigned word result as the **scale result**.

Write:

1. The third input as a word to initial base plus `0x14`.
2. `low8(initial base << 1)` to initial base plus `0x0D`.

Derive and retain:

- **Working base:** scale result multiplied by 32.
- **Scaled payload:** scale result shifted left by seven.
- **Reduced selector:** `(scale result & 0x07FFFFFF) >> 17`.

The reduced selector is between 0 and 1023 inclusive.

Invoke F-d023 through F-d025. Retain F-d025’s word result as the **working result**.

Write:

1. Scaled payload as a word to working base plus `0x14`.
2. `low8(working result << 2)` to working base plus `0x0D`.

### 4. Selector-directed writes

Invoke F-d026 with the working result shifted left by two. Invoke F-d027 through F-d030.

Then perform this ordered sequence:

1. Write `low8(fourth input << 1)` to reduced selector plus 7.
2. Read one unsigned byte from working base plus 1.
3. Write `low8(reduced selector)` to the read byte plus `0x11`.

The byte read is performed after F-d030. It is not established to equal any earlier value stored in nearby memory.

Invoke F-d031 with three identical arguments, each the working result shifted left by 21.

Invoke F-d032 and retain its unsigned word result as the **persistent result**. Invoke F-d033 with that result. Invoke F-d034 through F-d037, retaining F-d037’s word result as the **clearing result**.

Now:

1. Read another unsigned byte from working base plus 1; retain it as the **saved selector byte**.
2. Write zero to saved selector byte plus `0x11`.

This is a fresh read, not reuse of the earlier selector read.

### 5. Byte payload propagation and low-address stores

Invoke F-d038 with the clearing result shifted left by two. Invoke F-d039 through F-d042.

Derive:

- **Byte base:** twice the saved selector byte.
- **Byte payload:** `low8(persistent result >> 3)`.

Write byte payload to working base plus `0x0D`. Then write the saved selector byte to absolute address `0x0D`.

Invoke F-d043 with:

1. Byte base.
2. `0x98`.
3. `0x260`.

Invoke F-d044. The next call, F-d045, receives an additional return-register value from this region shifted logically right by three. That register value is not represented as F-d044’s ordinary return value; its ABI meaning remains unresolved.

Invoke F-d046 through F-d049, retaining F-d049’s word result as the **intermediate result**.

Invoke F-d050 with the intermediate result shifted left by two, retaining its word result as the **transformed result**.

Then write:

1. Byte payload to byte base plus `0x0D`.
2. Byte value `0xC0` to absolute address `0x0D`.

Invoke F-d051 with the transformed result shifted left by two. Invoke F-d052 through F-d055, retaining F-d055’s word result as the **field result**.

### 6. Linked word operations

Write the field result shifted left by two as a word to byte base plus `0x24`.

Read a word from byte base plus `0x74`; retain it as the **linked base**.

Derive the **linked payload** as the field result shifted left by five.

Invoke F-d056 with four arguments, in order:

1. Linked base.
2. Linked base shifted left by one.
3. Linked payload.
4. `(linked base & 0x7FFFFFFF) >> 2`.

Invoke F-d057 through F-d060.

Write linked payload as a word to linked base plus `0x24`.

Read a word from linked base plus `0x74` and pass that freshly read value to F-d061. Invoke F-d062 through F-d065, retaining F-d065’s paired result as **pair-one low** and **pair-one high**.

The word read for F-d061 occurs after the preceding write and intervening calls. No cached substitute is specified.

### 7. Paired results and indexed word stores

Derive the **retained shifted value** as pair-one low shifted left by two.

Write `persistent result >> 3` as a word to `(pair-one high × 4) + 0x14`.

This is a full-word write. It is not limited to the earlier byte payload.

Invoke F-d066 with:

1. Retained shifted value.
2. Retained shifted value.
3. Retained shifted value.
4. Pair-one high shifted right by three.

Invoke F-d067 and retain its paired result as **pair-two low** and **pair-two high**.

Invoke F-d068 with:

1. Pair-two low.
2. Pair-two high.
3. Pair-two low.
4. Pair-two high shifted right by three.

Invoke F-d069 through F-d072, retaining F-d072’s word result as the **indexed result**.

Write retained shifted value as a word to `(indexed result × 4) + 0x14`.

Invoke F-d073 with indexed result multiplied by four. Invoke F-d074 through F-d077, retaining F-d077’s unsigned word result as the **late result**.

### 8. Late selector updates and repeated clearing

Derive the **late shifted value** as late result shifted left by two.

Read one unsigned byte from linked base plus 1. Write `low8(late shifted value)` to that byte plus `0x11`.

Invoke F-d078 through F-d080, retaining F-d080’s word result as the **late intermediate result**.

Read linked base plus 1 again. Write `low8((late result & 0x3FFFFFFF) >> 20)` to the newly read byte plus `0x11`.

Invoke F-d081 with the late intermediate result shifted left by two. Invoke F-d082 through F-d085, retaining F-d085’s word result as the **late argument result**.

Next:

1. Read one unsigned byte from `(late shifted value × 0x400) + 1`.
2. Write zero to that byte plus `0x11`.

Invoke F-d086 with:

1. Late shifted value shifted left by three.
2. The same value again.
3. Late shifted value.
4. Late argument result shifted left by two.

Invoke F-d087 through F-d090, retaining F-d090’s word result as the **final result**.

Repeat the selector read from `(late shifted value × 0x400) + 1`, then write zero to the newly selected byte plus `0x11`.

Invoke F-d091 with the final result shifted left by two. Invoke F-d092 through F-d095.

The represented path then reaches the terminal invalid-instruction marker.

## Side effects

### Memory layout

The following offsets are observable access locations, not established structure fields:

| Address basis | Offsets and access widths |
|---|---|
| Initial base | Byte writes at `+1`, `+0x0D`; word write at `+0x14`. |
| Root value | Word write at `+0x34`. |
| Doubled root | Repeated byte writes at `+0x0D`. |
| Working base | Byte reads at `+1`; byte writes at `+0x0D`; word write at `+0x14`. |
| Byte base | Byte write at `+0x0D`; word write at `+0x24`; word read at `+0x74`. |
| Linked base | Byte reads at `+1`; word write at `+0x24`; word read at `+0x74`. |
| Other computed bases | Word writes at `+0x14`, selector-directed byte reads and writes. |
| Absolute addresses | Byte writes at `0x0D` and `0x11`. |

No nonaliasing guarantee exists. A store may overlap an earlier store, a later read, or memory accessed by a dependency. Word stores may therefore change subsequent selector bytes, depending on addresses and endianness.

Selector-byte-directed destinations range from `0x11` through `0x110`. The byte base ranges from 0 through 510. The reduced-selector write destination ranges from 7 through 1030. These are numeric bounds, not evidence that these addresses are mapped or writable.

### Ownership and dependency effects

No explicit allocation or release is visible. Ownership of all accessed memory is unknown.

Dependency calls may alter memory or other machine state. Their behavior is not constrained by this representation. In particular:

- Repeated reads must not be assumed equal.
- A later store must not be assumed to overwrite the same destination as an earlier selector-directed store.
- Earlier explicit writes do not establish final memory contents after subsequent calls.
- Calls may fail, trap, or not return.

### Errors and preconditions

No explicit validation, bounds checking, null checking, status-code handling, rollback, or recovery path appears.

Following the entire represented path requires valid call targets, returning dependencies, and accessible memory for every read and write. If an access faults, preceding effects need not be undone; later effects are not established.

## Dependencies

F-d001 through F-d095 are the only dependency names used by this contract. No resolved dependency implementation is supplied.

The visible target expressions generally derive from numeric control locations plus offsets. F-d001 instead depends on the incoming control address. Neither pattern establishes callable signatures or intended services.

Paired results are explicitly consumed from F-d005, F-d065, and F-d067. Other retained results and argument order are specified in the behavior section.

No dependency may be assumed to preserve globals, preserve pointed-to memory, own an argument, release an argument, or return normally. The representation’s surviving intermediate values describe its apparent dataflow, not an independently verified calling convention.

## Acceptance examples

These are conditional review examples, not executed tests.

1. **Initial base and truncation.**  
   If the base input is `0x00000400`, the initial base is `0x100`. If the root value ends in byte `0xAB`, the first explicit byte write stores `0xAB` at `0x101`, after F-d005 returns.

2. **Distinct full-word and byte uses.**  
   If the third input is `0x12345678`, its doubled-root stores write byte `0x78`; its initial-base `+0x14` store writes the full word `0x12345678`.

3. **Reduced-selector boundary.**  
   If the scale result is `0xFFFFFFFF`, the reduced selector is `0x3FF`. The fourth-input-derived byte is therefore written to numeric address `0x406`, provided execution reaches that store.

4. **Fourth-input truncation.**  
   If the fourth input’s bit pattern is `0x00000080`, shifting left by one produces low byte zero. The selector-relative store writes zero, not `0x80`.

5. **Fresh selector reads.**  
   If the first working-base selector read yields `0x20`, the corresponding write targets `0x31`. If the later read yields `0x40`, the later clearing write targets `0x51`. Clearing `0x31` instead would not match the represented order.

6. **Byte versus word payload.**  
   If the persistent result is `0x12345678`, its logical right shift by three is `0x02468ACF`. Byte-payload writes use `0xCF`; the later indexed word store uses `0x02468ACF`.

7. **Repeated zero stores remain distinct.**  
   If intervening dependencies change the late selector byte, the two final zero stores can target different addresses. Combining them into one store would discard observable behavior.

8. **Partial execution on failure.**  
   If the absolute-address `0x11` write faults, the earlier calls and writes may already have occurred. Subsequent operations are not required to occur, and no rollback is shown.

## Uncertainties

The following are **documented_only**:

- The actual function boundary and executable validity.
- Application purpose and semantic names for the accessed locations.
- Calling convention, hidden input delivery, and register preservation.
- Dependency target identities and complete signatures.
- Meaning and provenance of the additional return-register value used by F-d045.
- Whether paired results represent true 64-bit returns or multiple machine-register outputs.
- Memory mapping, endianness, alignment rules, and access volatility.
- Source-language signed arithmetic semantics.
- Ownership and lifetime of accessed memory.
- Hardware or runtime behavior at the terminal invalid-instruction marker.

Possible intent—such as field initialization, packed-address manipulation, or device control—is not established and imposes no acceptance requirement.

## Evidence

**Established static evidence:** the supplied representation contains 95 indirect-call occurrences, interleaved byte and word accesses, logical right shifts of unsigned values, masks, duplicated call arguments, paired-result extraction, and no ordinary return before the terminal invalid-instruction marker.

**Conditional interpretation:** the ordered behavior above treats that representation as a straight-line operational description and normalizes width-sensitive operations as 32-bit bit-vector operations. Invalid-instruction warnings prevent treating that interpretation as verified executable behavior.

**Runtime evidence:** none. No runtime tests, dependency executions, memory observations, or fault observations have been performed.
