<!-- Trusted provenance: F-b3ffbf60eb0b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b3ffbf60eb0b

## Purpose

F-b3ffbf60eb0b performs an ordered transformation of memory using four explicit inputs and two additional execution-context values. On normal completion, it returns a 64-bit value combining the original first input with a word read near the end of the operation.

The established behavior is a sequence of address calculations, memory reads, and memory writes. No application-level purpose is established. Several 32-bit values are used both as numeric operands and as address bases; this does not establish object types, allocation boundaries, ownership, or pointer validity.

Access order is significant. Earlier writes can affect later reads when their storage overlaps. Repeated accesses must therefore remain distinct in the behavioral contract.

## Interface

### Explicit inputs

| Input | Represented width | Established role |
|---|---:|---|
| First input | 32 bits | Base for two word reads; supplies the low 32 bits of the return value. |
| Second input | 32 bits | Not explicitly consumed by the represented body. |
| Third input | 32 bits | Multiplication operand, address base, stored word, and component of the halfword-write destination. |
| Fourth input | 32 bits | Replacement value for a context-relative word near entry. |

The first and third inputs are represented as signed 32-bit integers. The second and fourth inputs are represented as 32-bit values without an established signed interpretation.

Use of the first and third inputs in address calculations does not establish that callers supply ordinary pointers. They may be address-like values, numeric values, or values serving both purposes.

### Additional execution-context inputs

Two further values are consumed without being established by the four explicit arguments:

- **Context base:** supplies the address of a word at byte offset `0x14`, which is read and then replaced.
- **Output base:** supplies destinations for byte writes at offsets `0x11` and `0x09`, and a word write at offset `0x64`.

Both bases are represented by 32-bit integer values. Their origin, initialization, calling convention, and validity are unresolved. They are required execution context, not established additional formal arguments.

### Return value

Normal completion produces a 64-bit bit pattern:

- Bits 31–0 are the original first input’s 32-bit pattern.
- Bits 63–32 are the word read from the doubled-product base at offset `0x44`, after the halfword write and before the final word write.

This specifies bit placement in the result, not the byte order of a stored 64-bit value. There is no represented status result or separate error indicator.

### Width and layout conventions

All offsets below are byte offsets.

- A **word** is 32 bits.
- A **halfword** is 16 bits.
- A **byte** is 8 bits.
- The return value is 64 bits.

Byte stores use the low 8 bits of their source values. The halfword store uses the low 16 bits of its source. These truncations do not modify the retained full-width values used elsewhere.

The representation does not establish address-space width, byte order, alignment requirements, or a complete arithmetic-overflow policy. A 32-bit value used to form an address does not, by itself, establish a 32-bit address space.

## Behavior

### Retained values and derived bases

The following names describe values, not objects or ownership relationships:

| Descriptive value | Meaning |
|---|---|
| Saved context word | Initial word read from context base plus `0x14`. |
| Multiplier word | Word read from saved context word plus `0x14`, after replacement of the context field. |
| First product | Third input multiplied by multiplier word. |
| Linked word | Word read from first product plus `0x74`. |
| Second product | Third input multiplied by first product. |
| Doubled product | Second product multiplied by two. |
| Third product | Third input multiplied by second product. |
| Late base | Word read from first input plus `0x14` during the later portion of execution. |

The products are represented as signed 32-bit integer computations. Their mathematical results must not silently be treated as unbounded integers or as guaranteed modulo-\(2^{32}\) arithmetic. Overflow behavior is unresolved.

The order below applies to executions in which the arithmetic and resulting accesses have defined meanings in the execution environment.

### Context capture and initial publication

The operation first reads the word at context base plus `0x14` and retains it as the saved context word. It then writes the fourth input’s full 32-bit pattern to that same address.

Only after this replacement does it read the multiplier word from saved context word plus `0x14`. Consequently, this read observes memory after the context-field write; it is not an entry-state snapshot.

The first product is then formed. Before reading the linked word, the operation:

1. Writes the third input’s full word to third input plus `0x54`.
2. Writes the saved context word’s low byte to output base plus `0x11`.

The linked word is subsequently read from first product plus `0x74`.

### Linked-value publication and repeated observation

After obtaining the linked word, the operation forms the second product. It writes the third input’s full word to third input plus `0x54` again, then replaces the byte at output base plus `0x11` with the linked word’s low byte.

The next two reads, in order, are:

- A word from second product plus `0x44`.
- A word from first input plus `0x74`.

Both readings are retained independently.

The operation then writes the linked word’s full 32-bit pattern to output base plus `0x64`, followed by the low byte of the first reading from second product plus `0x44` to output base plus `0x09`.

It next reads second product plus `0x44` again. The low byte of this new reading is written to second product plus `0x11`.

The two reads from second product plus `0x44` are distinct observations. The intervening word and byte writes can change the second reading if their destinations overlap that word. The contract does not permit replacing the second read with the retained first reading.

### Doubled-product publication and late address selection

The doubled product is formed after the byte write to second product plus `0x11`.

The operation then writes the third input’s full word to third input plus `0x54` for a third time, followed by the linked word’s low byte to doubled product plus `0x11`.

It forms the third product for a word read at third product plus `0x44`. After retaining that word, it reads first input plus `0x14` and retains the result as the late base.

This late-base read occurs after all writes described above. Its value must not be replaced by a presumed entry-time value.

### Final writes and return capture

The remaining accesses occur in this order:

- The low byte of the earlier reading from first input plus `0x74` is written to linked word plus `0x01`.
- The low byte of the reading from third product plus `0x44` is written to late base plus `0x11`.
- The doubled product’s low byte is written to late base plus `0x09`.
- The late base’s low 16 bits are written to the address formed by third input plus doubled product.
- A word is read from doubled product plus `0x44` and retained as the return high word.
- The linked word’s full 32-bit pattern is written to linked word plus `0x54`.

Normal completion then returns the retained high word combined with the original first input as specified in the interface.

The halfword write precedes the return-high-word read and can therefore affect that reading through overlap. The final word write follows the reading; changing its source storage at that point does not replace the already retained return high word.

### Aliasing and value retention

No non-overlap precondition is established. Destinations can coincide with other destinations, overlap later read locations, or modify storage from which a later address is obtained. Partial overlap also matters because the operation mixes word, halfword, and byte accesses.

Once a value has been read and retained, later modification of its source storage does not imply a fresh read. In particular:

- The saved context word survives replacement of the context field.
- The linked word remains the source for later stores and address calculations.
- The reading from first input plus `0x74` remains the source for its later byte store.
- The late base remains the source for its subsequent address calculations and halfword truncation.
- The return high word remains fixed after its final read.

These are retained-value properties, not guarantees that their original storage remains unchanged.

### Preconditions and failures

Normal completion requires all nine word reads and all fifteen writes to be valid when reached. Every derived address must be meaningful under the actual address model, and every access must satisfy applicable permissions and alignment requirements.

There are no represented checks for null values, bounds, overlap, arithmetic overflow, or invalid addresses. No conditional branch, retry, recovery path, or rollback is shown.

If an access or arithmetic operation faults, the representation does not establish a function-specific error result or restoration of earlier state. Earlier successful writes may remain effective. Fault delivery and partial-access effects are environment-dependent.

## Side effects

The body contains **fifteen memory writes**:

| Width | Count | Destinations and sources |
|---|---:|---|
| 32 bits | **6** | Context base plus `0x14` receives the fourth input; third input plus `0x54` receives the third input three times; output base plus `0x64` receives the linked word; linked word plus `0x54` receives the linked word. |
| 8 bits | **8** | Output base plus `0x11` is written twice; output base plus `0x09`, second product plus `0x11`, doubled product plus `0x11`, linked word plus `0x01`, late base plus `0x11`, and late base plus `0x09` are each written once. |
| 16 bits | **1** | Third input plus doubled product receives the late base’s low 16 bits. |

The counts describe represented accesses, not distinct destinations or guaranteed hardware transactions.

If no other access overlaps the relevant storage, the context field ends with the fourth input, the output byte at offset `0x11` ends with the linked word’s low byte, and the third-input-relative word at offset `0x54` ends with the third input. Without those non-overlap conditions, later writes may change these locations again.

No allocation, release, function call, or explicit ownership transfer is represented. The operation requires access to storage but does not establish ownership of it. Address-like values are not established allocation bases.

There is no represented synchronization or explicit atomicity mechanism. The described order is the order of the supplied static representation, not an independently established machine-level visibility guarantee for concurrent observers or memory-mapped devices.

## Dependencies

No function-call dependency appears in the supplied body. No additional opaque function ID is needed.

Behavior depends on:

- The four explicit inputs, subject to the second input’s lack of direct use.
- The two execution-context bases.
- Memory contents at each read, including changes caused by earlier overlapping writes.
- The execution environment’s arithmetic, address-conversion, alignment, and fault semantics.

There is no unknown callee whose preservation of globals, allocation behavior, or release behavior can be assumed. Environmental effects outside the represented body remain unresolved rather than being assigned to an invented dependency.

## Acceptance examples

These are conditional consequences of static evidence, not runtime observations.

### Return composition

Assume all operations complete, the first input’s original pattern is `0x12345678`, and the late read at doubled product plus `0x44` yields `0xA1B2C3D4`.

The result is `0xA1B2C3D412345678`. Even if the final word write changes the memory used for that late read, the retained return high word remains `0xA1B2C3D4`.

### Distinct repeated readings

Assume the first read at second product plus `0x44` yields `0x11223344`. An intervening overlapping write changes that word, and its second read yields `0x55667788`.

The byte written to output base plus `0x09` is `0x44`. The subsequent byte written to second product plus `0x11` is `0x88`. Using `0x44` for both stores would violate the required observation order.

### Full-width and truncated linked values

Assume the linked word is `0x1234ABCD`, and all resulting addresses and arithmetic are valid.

Its byte stores to output base plus `0x11` and doubled product plus `0x11` use `0xCD`. Its word stores to output base plus `0x64` and linked word plus `0x54` use the full pattern `0x1234ABCD`.

The byte truncations do not narrow the retained linked word.

### Late-base truncation

Assume the late-base read yields `0x12345678`, and the subsequent accesses are valid.

The halfword written at third input plus doubled product has pattern `0x5678`. Its two-byte memory layout depends on the environment’s byte order.

### Late observation after overlap

Assume an earlier write changes the word at first input plus `0x14` before the late-base read.

The value observed by that read determines the subsequent late-base-relative destinations and the halfword’s source bits. An entry-time value cannot be substituted.

### Context replacement before multiplier observation

Assume saved context word plus `0x14` coincides exactly with context base plus `0x14`, with compatible word accesses and no intervening external modification.

The multiplier read observes the fourth input’s stored word, because the replacement occurs before that read.

### Unused second input

For executions differing only in the second explicit input, with identical relevant memory and execution context, the represented body has no direct dependence on that input. This does not resolve behavior outside the body or establish the complete calling convention.

## Uncertainties

The following remain **documented_only**:

- **Application purpose:** no named high-level operation is established.
- **Context provenance:** the source and calling convention of the additional bases are unknown.
- **Arithmetic overflow:** signed overflow, wrapping, trapping, and other machine interpretations are unresolved.
- **Address formation:** pointer width, conversion from integer values, and address-addition overflow are unresolved.
- **Physical layout:** byte order and alignment requirements are not established.
- **Exceptional behavior:** exact fault delivery, partial writes, and recoverability are unknown.
- **External mutation:** concurrency, device behavior, atomicity, and visibility are unspecified.
- **Representation fidelity:** correspondence between each represented access and separate machine-level transactions has not been independently verified.

Recurring offsets suggest structured storage, but that is inferred intent only. It does not justify object names, field meanings, ownership rules, or validity guarantees.

## Evidence

The supplied static body establishes:

- Four explicit inputs and a composite 64-bit return.
- Two additional consumed execution-context values.
- Nine 32-bit memory reads.
- Fifteen writes: **six 32-bit writes, eight byte writes, and one 16-bit write**.
- Three full-word stores of the third input to third input plus `0x54`.
- Two distinct reads from second product plus `0x44`.
- A late return-high-word read after the halfword store and before the final word store.
- No represented conditional branches or function calls.

No runtime tests have been performed. The acceptance examples specify behavior under stated conditions; they are not reports of execution.
