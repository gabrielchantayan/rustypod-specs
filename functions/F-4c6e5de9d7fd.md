<!-- Trusted provenance: F-4c6e5de9d7fd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4c6e5de9d7fd

## Purpose

F-4c6e5de9d7fd processes two initial fields, conditionally processes a separate optional field and a dot-prefixed suffix, writes selected portions of an output record, and delegates processing of the remaining input.

The 32-bit return value is **0 for success** or **1 for failure**. Failure does not imply that the output record or other accessible state is unchanged.

Three distinct 4-byte field destinations participate:

- **Initial-first destination:** supplied to the initial parser and later read for output offset +0x14.
- **Initial-second destination:** supplied to both parsing calls and later read for output offset +0x18.
- **Optional destination:** supplied as the first destination of the optional parsing call, or explicitly assigned zero when that call is skipped; later read as the argument to a transformation dependency.

These destinations are not immutable snapshots of parser results. Dependencies receive their addresses, and whether those addresses are retained is unknown. Values used later are the values present at the respective reads, including any intervening effects.

The delimiters, parser controls, and output construction suggest a time-like representation with an optional fractional component. This is **inferred intent**, not an established grammar or numeric interpretation.

## Interface

### Inputs and widths

The function receives two 32-bit address-valued inputs:

| Input | Established role |
|---|---|
| Input address | Base for direct byte reads and addresses supplied to parsing and trailing-processing dependencies. |
| Output address | Base of the record receiving direct writes and trailing processing. |

There is no explicit input length, output capacity, ownership flag, or locale argument.

Field destinations hold 32-bit values. Accumulators and numeric dependency results hold 64-bit values. When passed as separate words, each 64-bit operand is supplied **low 32-bit word first, high 32-bit word second**. This establishes interface width and word order, not integer signedness, floating-point representation, or mathematical meaning.

The body contains no direct store through the input address. It does not establish input immutability: output may overlap input, and dependencies may modify accessible memory.

### Output layout

Offsets below are relative to the output address.

| Offset | Width | Direct behavior |
|---|---:|---|
| +0x14 | 4 bytes | Receives the value read from the initial-first destination. |
| +0x18 | 4 bytes | Receives the value read from the initial-second destination. |
| +0x1c | 4 bytes | Read after successful trailing processing; compared with zero. |
| +0x20 | 8 bytes | Receives the final dependency-produced composition. |
| +0x29 | 1 byte | Assigned 1 before trailing processing. |
| +0x2a | 1 byte | Assigned 0 before trailing processing. |
| +0x2b | 1 byte | Assigned 0 or 1 after successful trailing processing. |

The 8-byte field occupies +0x20 through +0x27. No direct access to +0x28 is shown.

The function does not directly initialize +0x1c. Finalization reads its contents after the trailing dependency returns successfully.

### Return conditions

On normally completing dependency calls, return 0 requires:

1. Initial parsing returns exactly 2.
2. Optional parsing, if invoked, returns exactly 1.
3. Trailing processing returns exactly 0.

Failure of any applicable return-value condition produces return 1. No diagnostic or distinct error category is returned.

These gates do not independently establish that dependencies initialized their destinations, preserved earlier values, or accepted all supplied memory.

### Preconditions

Defined execution on a selected path requires readable input bytes, writable output locations, valid dependency arguments, and initialized field values wherever consumed.

The dot path additionally requires the provider’s returned address to support a 4-byte read of a table address. Each classification lookup requires a readable byte at that captured table address plus an unsigned input-byte index.

No local null, capacity, alignment, address-overflow, or scan-length check is shown. Address operations use the supplied 32-bit address representation; behavior at address-space boundaries is not established.

A zero input byte does not intrinsically terminate scanning. Its table entry determines whether it is accepted. Normal scan completion requires eventually observing a classification byte other than 0x20.

## Behavior

### Initial state and parsing

Before the first dependency call, two fixed data words are loaded into a 64-bit **fractional accumulator**. Their contents and numeric encoding are unavailable.

The initial call to **F-d770f78fa23f** receives, in order:

- The input address.
- Four 32-bit controls: 2, 0, 24, 58.
- The initial-first destination address.
- Four 32-bit controls: 2, 0, 59, 0.
- The initial-second destination address.

The controls are literal values. Their interpretation as widths, bounds, delimiters, or other parser settings is not established by this body.

If the result is not exactly 2, the function returns 1. It does not directly inspect input +5 or perform its direct output-record stores on this path. Effects of the parser remain possible.

### Optional-field selection

After successful initial parsing, the function reads the byte at input +5.

If that byte is not 0x3a:

- The optional destination is assigned the 32-bit value zero.
- The initial-second destination is not directly zeroed.
- The remaining-input cursor is input +5.
- No optional parser, dot inspection, classifier provider, or scan arithmetic is invoked.

If the byte is 0x3a, a second call to **F-d770f78fa23f** receives:

- Input +6.
- Controls 2, 0, 59, 0.
- The optional destination address.
- Controls 2, 0, 59, 0.
- The initial-second destination address.

The two destinations in this call are distinct. Its second destination is the same initial-second destination used by the initial call.

Only a result of exactly 1 permits continuation. Any other result causes return 1 before the function’s direct record stores.

The return value 1 does not establish that only one destination was modified. In particular, the second call may change the initial-second destination.

After successful optional parsing, the cursor becomes input +8, and the function reads that byte.

### Dot selection and first classification

If input +8 is not 0x2e, the cursor remains input +8. No classifier-provider or scan-arithmetic call occurs.

If it is 0x2e, the observable sequence is:

1. **F-2a4c402ba617** is called without explicit arguments.
2. Two fixed data words are loaded into a 64-bit **auxiliary accumulator**.
3. A 32-bit table address is read through the provider’s returned address and captured.
4. Input +9 is read as an unsigned byte, yielding an index from 0 through 255.
5. One byte at the captured table address plus that index is read and compared with 0x20.

The comparison is exact equality with 0x20. It is not a bitmask test or a generic nonzero test.

If the first classification does not equal 0x20, no scan or post-scan arithmetic occurs. The fractional accumulator retains its earlier value, and the cursor remains at input +8, pointing at the dot.

The provider call occurs before either field destination is copied to the output or the optional destination is read for transformation. Preservation of those destinations across this call is not established.

### Classification scan

If the first classification equals 0x20, scanning begins.

Each iteration retains the current high 32-bit word of both accumulators, advances the cursor by one byte, and reads the byte at the new cursor and its classification-table entry.

Because the cursor starts at input +8, the first iteration reads and classifies input +9 again. The preliminary eligibility lookup does not consume the byte. These are separate reads; mutable input or table contents can make their results differ.

A classification other than 0x20 terminates the scan before accepted-byte arithmetic for that iteration.

For each accepted iteration, dependencies execute in this order:

| Order | Dependency and operands |
|---|---|
| 1 | **F-9e5e664c8ae7**, without explicit arguments, returns a 64-bit value. |
| 2 | **F-c4b34006a13c** receives the fractional accumulator and a fixed-data 64-bit operand. |
| 3 | **F-1b22d7e17791** receives the result of order 2 followed by the result of order 1. |
| 4 | **F-8a693c103d1c** receives the result of order 3 and another fixed-data 64-bit operand. Its result replaces the fractional accumulator. |
| 5 | **F-c4b34006a13c** receives the auxiliary accumulator and the fixed-data operand referenced at order 2. Its result replaces the auxiliary accumulator. |

Each paired-word operand is passed low word first, high word second.

The accepted byte is not an explicit argument to any arithmetic dependency. A direct conversion from that byte to a numeric digit is therefore not established.

Fixed-data operands are read at their respective call sites. The two references to the same operand source need not produce equal values if intervening dependencies can modify that data.

When scanning terminates, **F-596e18a33aec** receives the fractional accumulator followed by the auxiliary accumulator. Their high words are those retained at the start of the terminating iteration; no accepted-byte arithmetic has intervened in that iteration. Its 64-bit result replaces the fractional accumulator.

This final dependency is called whenever the scan was entered and then terminated, including when the repeated classification of +9 rejects immediately.

The cursor remains at the first scanned byte whose classification is not 0x20. There is no local bound on the number of accepted bytes.

### Field reads and output composition

After the applicable parsing and suffix paths converge, direct writes occur in this order:

1. Store byte 0 at output +0x2a.
2. Store byte 1 at output +0x29.
3. Read the initial-first destination and store its 4-byte value at output +0x14.
4. Read the initial-second destination and store its 4-byte value at output +0x18.

The copied values are those present at these reads, not necessarily those originally produced by either parser. On the dot path, the provider and potentially multiple arithmetic calls have already occurred. Their effects through any retained destination addresses are unknown.

Next, the optional destination is read and supplied as a 32-bit argument to **F-b3a54d881220**. On the no-colon branch this destination was explicitly assigned zero. On the optional-parser branch its value is whatever is present at this later read, including any intervening effects.

The returned 64-bit value is supplied as the first operand to **F-1b22d7e17791**, with the fractional accumulator as the second operand. The 64-bit result is stored at output +0x20.

The body establishes dependency-mediated composition, not addition, scaling, units, rounding, or numeric range behavior.

### Trailing processing and finalization

The function calls **F-389c41c1cbea** with the current cursor followed by the output address.

| Reached path | Cursor supplied |
|---|---|
| No colon at +5 | Input +5 |
| Successful optional parsing, no dot at +8 | Input +8 |
| Dot present, preliminary +9 classification rejected | Input +8 |
| Scan entered and terminated | First scanned byte classified other than 0x20 |

A nonzero result causes overall return 1. Earlier writes are not rolled back, and there is no direct write to +0x2b on this failure path.

A zero result causes a 4-byte read at output +0x1c. The function writes byte 0 at +0x2b if that value is zero, otherwise byte 1, then returns 0.

This decision uses post-dependency record contents. Earlier field and flag writes are not repeated after trailing processing.

## Side effects

### Partial updates and preservation

Parser rejection occurs before direct record stores. Trailing rejection occurs after direct stores to +0x2a, +0x29, +0x14, +0x18, and +0x20.

These statements describe this function’s accesses only. Dependencies can have additional effects, including modifying input, field destinations, globals, or the output record. In particular, later dependencies may overwrite earlier direct stores. No rollback or final-record preservation guarantee is established.

The 8-byte store width does not establish atomicity.

### Retained addresses and aliasing

The parser receives addresses of all three field destinations across its calls. Whether it retains or exposes those addresses is unknown. Consequently, later dependencies cannot be assumed to preserve the parser-produced values merely because those destinations are not explicit arguments to those later calls.

Input/output disjointness is not established. Reached output stores can modify overlapping input bytes before trailing processing consumes them.

The classification-table address is captured once and reused. Later changes to the provider’s pointer slot do not cause a refresh. Changes to entries at the captured address can affect subsequent lookups.

### Ownership and resources

No explicit allocation, release, retention request, or ownership transfer appears in the body. The function does not explicitly release the provider’s returned address or the captured table.

Dependency ownership, retention, allocation, and release behavior remain unspecified. Absence of explicit arguments does not imply absence of memory or global-state effects.

## Dependencies

| Opaque ID | Established interface use |
|---|---|
| F-d770f78fa23f | Input address, eight 32-bit controls in two groups, and two writable 4-byte destinations. Initial result must equal 2; optional result must equal 1. |
| F-2a4c402ba617 | No explicit arguments; returns an address through which a 32-bit table address is read. |
| F-9e5e664c8ae7 | No explicit arguments; returns a 64-bit value once per accepted scan iteration. |
| F-c4b34006a13c | Two paired-word operands; returns a 64-bit result. Called twice per accepted iteration. |
| F-1b22d7e17791 | Two paired-word operands; used during scanning and final composition. |
| F-8a693c103d1c | A paired-word intermediate and a paired-word fixed-data operand. |
| F-596e18a33aec | Both accumulators as paired-word operands after scan termination. |
| F-b3a54d881220 | One 32-bit field value; returns a 64-bit result. |
| F-389c41c1cbea | Remaining-input cursor and output address; only result 0 permits finalization. |

Arithmetic semantics, preservation rules, exceptional behavior, and resource contracts are not established by these call sites.

## Acceptance examples

These are **conditional static acceptance criteria**, not executed tests. They assume valid memory, normally returning dependencies, and initialized values wherever required.

1. **Initial rejection.** If the initial parser returns anything other than 2, overall return is 1. No direct +5 input inspection or output-record store occurs.

2. **No optional field.** Suppose initial parsing produces A and B and returns 2, input +5 is not 0x3a, and no intervening effects alter either destination. The direct field stores write A at +0x14 and B at +0x18. The transformation receives zero from the optional destination, and trailing processing receives input +5.

3. **Optional parsing uses distinct destinations.** Suppose optional parsing returns 1 after writing C to the optional destination and D to the initial-second destination. **Provided no intervening effects change either destination before its respective read**, the transformation receives C and the store at +0x18 receives D. C is not substituted for D.

4. **Dot-path mutation is respected.** Suppose optional parsing writes C and D, but an intervening dot-path dependency changes the optional destination to E and the initial-second destination to G. If those values remain until their respective reads, the transformation receives E and +0x18 receives G. The function does not restore C or D.

5. **Optional rejection.** With 0x3a at +5, any optional-parser result other than 1 causes return 1 before direct record stores or trailing processing.

6. **Rejected preliminary classification.** With successful optional parsing, a dot at +8, and a nonmatching preliminary +9 lookup, the provider is called, but scan arithmetic and F-596e18a33aec are not. Trailing processing receives input +8.

7. **Two stable accepted bytes.** If +9 and +10 consistently classify as 0x20 and +11 does not, accepted-byte arithmetic executes twice. F-596e18a33aec executes once, and trailing processing receives input +11.

8. **Eligibility changes before scanning.** If the preliminary +9 lookup matches but the repeated lookup rejects, accepted-byte arithmetic executes zero times. F-596e18a33aec still executes, and trailing processing receives input +9.

9. **Trailing failure.** A nonzero trailing result yields overall return 1 after the earlier direct stores. There is no rollback or direct +0x2b update.

10. **Post-call finalization.** After a zero trailing result, a zero 32-bit value at +0x1c produces byte 0 at +0x2b; any nonzero value produces byte 1. Overall return is 0.

## Uncertainties

The following remain **documented_only**:

- Parser grammar, range enforcement, width enforcement, and destination-write guarantees.
- Whether parser results establish initialization of every subsequently read destination.
- Retention of destination addresses and modifications through them by intervening dependencies.
- The proposed time-like interpretation.
- Fixed-data values and 64-bit numeric encoding.
- Arithmetic meanings, signedness, rounding, overflow, and exceptional values.
- How F-9e5e664c8ae7 obtains its result without an explicit byte argument.
- Classification-table meaning, mutability, validity, and lifetime.
- Trailing grammar and record modifications.
- Dependency preservation of globals, input, destinations, and output.
- Dependency ownership and resource effects.
- Multibyte storage byte order, alignment requirements, and atomicity.
- Behavior for invalid addresses, unsupported overlap, address arithmetic at boundaries, or scans that never encounter a rejecting classification.

Unknown behavior is not replaced with assumed library semantics.

## Evidence

Established facts derive from static inspection of the supplied body: access widths, offsets, literal controls, return gates, destination reuse, dependency argument order, repeated classification reads, cursor transitions, and direct-store order.

The field-preservation qualification follows from the parser receiving destination addresses and the dot path invoking other dependencies before later destination reads. No dependency implementations or preservation contracts are supplied.

The time-like and fractional interpretations are inferred intent only. Fixed-data contents, caller contracts, and record declarations are unavailable. No runtime tests have been performed.
