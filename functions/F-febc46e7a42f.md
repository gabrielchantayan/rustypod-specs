<!-- Trusted provenance: F-febc46e7a42f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-febc46e7a42f

## Purpose

This operation exposes a sequence of indirect dependency calls, byte lookups, byte writes, and 32-bit word transfers. Some stored values are scaled dependency results; others are selected bit fields from earlier results or memory reads.

No application-level purpose is established. The evidence does not identify a conventional object format, device protocol, serialization operation, or recognizable algorithm.

The supplied static representation reports invalid instruction data and ends at an invalid-instruction boundary. This contract describes the represented path, conditional on its faithful recovery. It does not promise that the operation returns normally or that the entire path is executable.

## Interface

### Explicit inputs

| Position | Represented type | Observable role |
|---|---|---|
| First | Unsigned 32-bit integer | Supplies two byte values, a lookup-address calculation, and an argument to F-d001 |
| Second | Opaque 32-bit quantity | No explicit consumption on the represented path |
| Third | Opaque 32-bit quantity | No explicit consumption on the represented path |
| Fourth | Signed 32-bit integer | Supplies two identical, left-shifted arguments to F-d013 |

Lack of explicit consumption does not establish that dependencies cannot observe the second or third input through machine state.

### Implicit inputs

The first lookup depends on an incoming address-bearing register. The destination of F-d001 depends on incoming link state.

An additional unsigned 32-bit register observation supplies arguments and byte-processing values later in the path. Its producer and lifetime are unresolved. This contract calls it the **unresolved register value**. Its observations across intervening calls must not be assumed equal.

Other inputs include memory contents, dependency results, and any hidden state dependencies consume.

### Results and completion

No return value is declared. Dependency results are consumed internally, including one represented 64-bit result.

After F-d112, the representation reaches an invalid-instruction termination boundary, not an ordinary return. Whether execution traps, faults, transfers elsewhere, or follows a different path is unresolved.

### Widths, layout, and arithmetic

A byte is eight bits; a word is 32 bits. Byte writes retain the lowest eight bits. Unsigned right shifts insert zeros.

Unsigned 32-bit multiplication and left shifts retain the low 32 bits. This applies, for example, to calculations based on the first input and the low half of the 64-bit result.

Other calculations use represented signed 32-bit values, including many dependency results, the fourth input, and the high half of the 64-bit result. Negative signed left shifts, signed overflow, and out-of-range signed conversions are not resolved. Where an example uses wrapping machine arithmetic for these values, that condition is explicit.

The constants 17 and 20 are address displacements. They do not establish fields within a common structure. Word byte order, memory mapping, and access-alignment requirements are unknown.

## Behavior

### Ordering and dependency notation

F-d001 through F-d112 designate indirect-call occurrences in execution order. They are opaque contract-local IDs, not assertions that 112 distinct functions exist. A call described as having no explicit arguments may still consume hidden machine state.

The following phases define the observable ordering. Each phase is reached only if preceding accesses and calls complete. Dependencies may modify memory and globals; no preservation guarantee is assumed.

### Entry and first-input byte processing

Before any dependency call, the operation reads the byte one address unit beyond the incoming address-bearing register. It adds 17 to that byte’s unsigned value and writes the low byte of the first input at the resulting address.

F-d001 receives the first input. F-d002–F-d004 follow without explicit arguments, and F-d005 produces a represented signed 32-bit result.

Between F-d005 and F-d006, a byte is read at **16 × first input + 1**, using unsigned 32-bit arithmetic. Its zero-extended value becomes the **first lookup byte**. At address **first lookup byte + 17**, the operation writes the low byte of the first input logically shifted right by 22.

F-d006 receives F-d005’s result shifted left by two. F-d007–F-d009 have no explicit arguments. F-d010 produces another signed 32-bit result, and F-d011 receives that result shifted left by two as its first argument and the retained first lookup byte as its second.

All byte-derived write destinations in this contract fall between 17 and 272 inclusive. That numerical range is not a validity guarantee.

### Fourth input and unresolved-register processing

F-d012 has no explicit arguments. F-d013 receives two copies of the fourth input shifted left by ten. F-d014 follows without explicit arguments.

The next lookup reads the byte at address 1. The operation writes zero to that byte’s unsigned value plus 17. The represented index for this lookup is zero; there is no established traversal or loop.

F-d015 receives three arguments: two copies of the unresolved register value available there, followed by that value multiplied by 16. F-d016–F-d018 have no explicit arguments, and F-d019 produces a signed 32-bit result.

Between F-d019 and F-d020, the operation reads a byte at **16 × unresolved register value + 1**. It writes the low byte of that same represented register value logically shifted right by 22 to the lookup byte plus 17. No dependency call separates this lookup from its associated write. Equality with the register observation used by F-d015 is not established.

F-d020 receives F-d019’s result shifted left by two. F-d021–F-d023 have no explicit arguments. F-d024 produces a signed 32-bit result, and F-d025 receives the retained lookup byte first and that result shifted left by two second. This argument order differs from F-d011.

### Split-result processing

F-d026 has no explicit arguments. F-d027 receives two zero arguments. F-d028 has no explicit arguments and supplies a represented 64-bit result.

The result is separated numerically into **split-low**, its unsigned low 32 bits, and **split-high**, its high 32-bit pattern represented as a signed integer. This separation does not depend on memory endianness.

Before F-d029, a zero word is written at **4 × split-high + 20**. F-d029 receives split-high. F-d030–F-d032 have no explicit arguments; F-d033 supplies a signed 32-bit destination selector.

Before F-d034, split-high is written as a word at **4 × destination selector + 20**. F-d034 receives four times that selector. F-d035–F-d038 then occur without explicit arguments.

Before F-d039, the operation writes:

- Value: **split-low × 2²⁶**, truncated to 32 bits.
- Address: **split-low × 2²⁸ + 20**, using unsigned 32-bit arithmetic.

Only the low six bits of split-low affect that value; only its low four bits affect that address.

F-d039 receives the same scaled value. F-d040–F-d042 have no explicit arguments, and F-d043 supplies another signed 32-bit selector. Before F-d044, the operation writes **the low 29 bits of split-low, logically shifted right by nine**, at four times that selector plus 20. F-d044 receives four times the selector.

### Scaled-word transfer phases

The next portion repeatedly obtains a selector, writes its fourfold-scaled value at a differently scaled address, and later propagates that value. For a selector called a **producer selector**, its **scaled word** is four times the selector, while its immediate write destination is **16 × producer selector + 20**. These are not the same scaling.

The table records the complete ordering for this portion. All calls in the “preceding calls” column have no explicit arguments. Selector results are represented as signed 32-bit values.

| Preceding calls | Result-producing call | Memory effect before the following call | Following call and explicit arguments |
|---|---|---|---|
| F-d045–F-d047 | F-d048 | Write its scaled word at 16 times its producer selector plus 20 | F-d049 receives two memory-derived fields with the scaled word between them, as detailed below |
| F-d050–F-d052 | F-d053 | Write F-d048’s scaled word at four times this new selector plus 20 | F-d054 receives four times this selector |
| F-d055–F-d057 | F-d058 | Write its scaled word at 16 times its producer selector plus 20 | F-d059 receives that scaled word twice, then split-low shifted left by 26 |
| F-d060–F-d062 | F-d063 | Write F-d058’s scaled word at four times this new selector plus 20 | F-d064 receives four times this selector |
| F-d065–F-d067 | F-d068 | Write its scaled word at 16 times its producer selector plus 20 | F-d069 receives that scaled word |
| F-d070–F-d072 | F-d073 | Write F-d068’s scaled word at four times this new selector plus 20 | F-d074 receives four times this selector |
| F-d075–F-d077 | F-d078 | Write its scaled word at 16 times its producer selector plus 20; then capture a word at its scaled word plus 20 | F-d079 receives that scaled word |

F-d049 has three arguments:

- First: a word read at **F-d048’s scaled word + 20**, logically shifted right by twelve.
- Second: F-d048’s scaled word.
- Third: another word read at **F-d048’s scaled word + 20**, shifted left by two as an unsigned value.

The representation contains two reads for F-d049, not one captured read. Their relative evaluation order is not established. They need not produce equal values for volatile or externally changing memory.

The immediate producer write does not generally initialize the location read for F-d049. Under wrapping 32-bit address arithmetic, the two addresses coincide only when twelve times the producer selector is zero modulo 2³².

The word captured after F-d078’s write is called the **captured word**. Its read precedes F-d079. Later dependency changes to its source location do not change the already captured value.

### Captured-field processing

F-d080–F-d082 have no explicit arguments. F-d083 supplies a signed 32-bit result called the **retained selector**. Its fourfold-scaled value is represented as unsigned and called the **retained scaled value**. The signed multiplication producing it remains subject to the arithmetic uncertainties above.

Before F-d084, the captured word logically shifted right by twelve is written at **retained scaled value + 20**. F-d084 receives the retained scaled value.

F-d085–F-d087 have no explicit arguments. F-d088 supplies a producer selector whose scaled word is written at sixteen times that selector plus 20. F-d089 then receives:

| Argument | Value |
|---|---|
| First | Low six bits of the retained scaled value, shifted left by fourteen |
| Second | F-d088’s scaled word |
| Third | Retained selector shifted left by thirty |

F-d090–F-d092 have no explicit arguments. F-d093 supplies a new selector. F-d088’s scaled word is written at four times this new selector plus 20, and F-d094 receives four times the new selector.

F-d095–F-d097 have no explicit arguments. F-d098 produces an unsigned 32-bit **final result**.

The retained selector still comes from F-d083. F-d093 does not replace it for the final lookups.

### Final byte processing and boundary

The final lookup address is **retained selector × 2²⁸ + 1**. This multiplication is represented as signed arithmetic; unconditional wrapping semantics are not established.

Between F-d098 and F-d099, the operation reads a byte at that address and writes the low byte of **final result shifted left by two** to the lookup byte plus 17.

F-d099 receives two copies of that left-shifted result, followed by the final result shifted left by four. The first two arguments pass through a represented signed 32-bit quantity; out-of-range conversion behavior is unresolved.

F-d100 has no explicit arguments and supplies an opaque 32-bit result passed directly to F-d101. F-d102–F-d104 have no explicit arguments. F-d105 supplies a signed 32-bit result.

Between F-d105 and F-d106, the operation reads the final lookup address again. It writes the low byte of **the final result masked to 30 bits and logically shifted right by twenty** to this second lookup byte plus 17. At byte width, this selects bits 20–27 of the final result; bits 28–29 do not survive the store.

F-d106 receives F-d105’s result shifted left by two. F-d107–F-d109 have no explicit arguments. F-d110 supplies a signed 32-bit result, and F-d111 receives that result shifted left by two followed by the retained second lookup byte.

F-d112 has no explicit arguments. The next represented event is the invalid-instruction boundary.

The two final lookups are distinct observations separated by calls. Neither equal lookup bytes nor equal write destinations are guaranteed.

## Side effects

An uninterrupted traversal contains **20 explicit stores: six byte stores and fourteen word stores**. It also contains six explicit byte reads and three explicit word reads, counting the two reads associated with F-d049 separately. These counts exclude dependency behavior.

The operation has no established disjointness guarantees. Lookup memory, write destinations, dependency-accessed memory, and indirect-call destinations may alias. A later store may overwrite an earlier store. Dependencies may alter memory before subsequent reads.

No explicit allocation, release, ownership transfer, synchronization, rollback, or error-reporting mechanism is established. An address-like argument does not itself transfer ownership. Dependency retention or release of memory remains unknown.

No recovery path is represented for an unsuccessful access or call. Earlier effects are not shown being undone. These are execution requirements, not validated input checks.

## Dependencies

F-d001–F-d112 describe ordered call occurrences only. Their explicit arguments and consumed results are specified above.

F-d001’s destination depends on incoming link state plus 4094. Other destinations use represented address bases plus 4094, except F-d099, which uses a displacement of 4076. These relationships do not establish alignment, executable mappings, calling convention, or successful dispatch.

For every dependency, the following remain unknown unless directly captured by the visible data flow:

- Whether it returns or terminates execution.
- Which memory, registers, or globals it reads or changes.
- Whether it allocates, retains, transfers, or releases memory.
- Whether its actual result convention matches the represented width.
- Whether another call occurrence reaches the same underlying function.

Explicitly retained values describe the recovered data flow. They do not supply a general register- or global-preservation guarantee.

## Acceptance examples

These are conditional arithmetic and ordering examples, not executed tests.

1. **Initial byte truncation.** With first input **0x12345678** and entry lookup byte **5**, the first store writes **0x78** at address **22**, provided the lookup and destination access complete.

2. **Unsigned lookup arithmetic.** With first input **0xFFC00000**, the first-input lookup address is **0xFC000001** under unsigned 32-bit arithmetic. If its byte is **9**, the associated store writes **0xFF** at address **26**.

3. **Zero-byte destination boundary.** If address **1** contains **255** when observed after F-d014, the next explicit store writes zero at address **272**. Earlier calls may have changed that lookup byte.

4. **Split result.** If F-d028 supplies high half **3** and low half **1**, the first split-result store writes zero at address **32**. The later unsigned low-half store writes **0x04000000** at **0x10000014**, assuming intervening execution reaches it.

5. **Mask boundary.** For split-low **0xFFFFFFFF**, masking to 29 bits and shifting right by nine yields **0x000FFFFF**. That is the value placed at F-d043’s selected destination.

6. **Different word locations.** If F-d048’s selector is **1**, its producer store writes **4** at address **36**, but F-d049’s two word reads use address **24**. The store does not establish either read’s value.

7. **Two independent word reads.** If F-d049’s first argument read yields **0x00012000** and its third argument read yields **3**, its arguments are **0x12**, the scaled word, and **12**. This example does not specify which read occurs first.

8. **Captured value.** If the capture after F-d078 yields **0xABCDEFFF**, the store before F-d084 writes **0x000ABCDE**, even if intervening dependencies change the original source location.

9. **Signed-shift qualification.** If the fourth input is **1**, both F-d013 arguments are **1024**. For a negative fourth input, the contract does not assign a definite shift result without an additional machine-arithmetic interpretation.

10. **Final byte extraction.** If the final result is **0x3AB00000**, the second final byte store writes **0xAB**. Its intermediate shifted field is **0x3AB**, but the store retains only eight bits.

11. **Repeated lookup and termination.** If the final lookups return **2** and **7**, their destinations are **19** and **24**. Completion of F-d112 still does not establish a normal return.

## Uncertainties

**Established static evidence:** the represented input widths, explicit data flow, ordered call occurrences, access widths, distinct repeated reads, shifts and masks, and abnormal terminal boundary.

**Inferred intent:** repeated scaling and field extraction suggest manipulation of structured state. No specific format or domain is established.

**documented_only:**

- Whether the recovered path accurately represents executable control flow.
- Actual entry convention and provenance of implicit register inputs.
- Producer and stability of the unresolved register value across calls.
- Signed overflow, negative signed shifts, and out-of-range signed conversions.
- Address validity, alignment, byte order, volatility, and concurrent modification.
- Dependency identities, hidden inputs, side effects, ownership, and failure behavior.
- Relative order of the two word reads used by F-d049.
- Actual execution behavior at the invalid-instruction boundary.

For traversal to reach the final boundary, all earlier accesses must succeed and all dependencies must complete sufficiently to permit continuation. No such conditions have been validated.

## Evidence

This contract is based solely on the supplied restricted static representation. That representation explicitly reports invalid instruction data and truncated control flow.

The access counts are static counts of explicit memory operations, excluding dependency bodies. Width distinctions follow the represented signed, unsigned, byte, word, and 64-bit quantities.

No independent implementation inspection, dependency-body inspection, runtime execution, smoke run, or tests have been performed. Acceptance examples are conditional consequences of the represented arithmetic and ordering, not observed execution results.
