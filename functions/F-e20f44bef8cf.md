<!-- Trusted provenance: F-e20f44bef8cf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e20f44bef8cf

## Purpose

F-e20f44bef8cf selects one of five 32-bit fields from a dependency-provided record, resolves a stored reference from that field, or formats all five fields into shared writable storage.

The input is interpreted as a 32-bit selector:

- Exactly one of bits 0–4 set: resolve the corresponding field.
- Two or more bits set anywhere in the 32-bit input: format all five fields.
- Zero, or a single set bit outside bits 0–4: return a null pointer.

The formatting result is a pointer to shared storage containing an asterisk followed by forty lowercase hexadecimal digits. This function does not write a terminating zero byte.

These behaviors are established by the supplied static representation. The semantic meaning of the five fields, the reference encoding, and the shared storage remains unknown.

## Interface

### Input

One unsigned 32-bit value serves as the selector. Its full width matters: bits above bit 4 participate in the population test even though they do not individually select a field.

The initial selection test uses 32-bit arithmetic. Negation is interpreted modulo \(2^{32}\), followed by a bitwise complement and bitwise intersection with the input. The result is nonzero exactly when the input has at least two set bits. This test takes precedence over all individual-selector cases.

### Output

The result is a pointer, with four possible origins:

| Input or field condition | Result |
|---|---|
| Input contains at least two set bits | Current value of the shared output-buffer pointer at return |
| Input is an accepted single-bit selector and its selected field is zero | Address of a fixed fallback object |
| Input is an accepted single-bit selector and its selected field is nonzero | Address computed from that field and one memory-resident displacement |
| Input is zero or an unsupported single-bit selector | Null pointer |

A null pointer is therefore an explicit result for unsupported selectors. It is not established as an error indicator for dependency failures or invalid memory.

### Record layout

F-187ab9ba4eb6 is invoked without explicit arguments. Its result is used as the base address of a record containing these fields:

| Selector | Byte offset from record base | Read width |
|---|---:|---:|
| 0x00000001 | 0x20 | 32 bits |
| 0x00000002 | 0x24 | 32 bits |
| 0x00000004 | 0x28 | 32 bits |
| 0x00000008 | 0x2c | 32 bits |
| 0x00000010 | 0x30 | 32 bits |

The formatting path reads all five fields in this order. Individual-selection paths read only the selected field.

These are native 32-bit reads. The supplied representation does not independently establish byte order, alignment requirements, or the record’s other contents.

### Preconditions

For normal completion:

- Any invoked dependency must return normally.
- Its returned record base must permit each field read required by the chosen path.
- The formatting path requires writable storage for the prefix byte and forty digit bytes.
- A nonzero selected field requires a readable 32-bit displacement at the address described under reference resolution.
- Address calculations and memory accesses must be valid under the actual target’s execution rules.

The function contains no checks that establish these conditions. No allocation, capacity check, record-size check, or validation of a computed result is visible.

## Behavior

### 1. Classify the selector

The function first distinguishes inputs with at least two set bits from all remaining inputs.

This is a population test, not a test restricted to the five supported selectors. For example, 0x80000001 enters formatting even though its highest set bit has no individual field mapping.

If the population test does not select formatting, the only accepted inputs are 1, 2, 4, 8, and 16. Every other remaining input returns null without calling F-187ab9ba4eb6.

One intermediate comparison treats the input as signed 32-bit. This affects the route taken by high-bit-only inputs but does not expand the accepted set: 0x80000000 still returns null.

### 2. Format all five fields

For an input containing at least two set bits, the visible sequence is:

1. Obtain the digit destination from the shared buffer pointer, advancing it by one byte.
2. Write the byte 0x2a, the asterisk character, through the shared buffer pointer.
3. Invoke F-187ab9ba4eb6 once.
4. Read the first 32-bit field at record offset 0x20.
5. Write its eight hexadecimal digits to the retained digit destination.
6. Repeat the field read and eight digit writes for offsets 0x24, 0x28, 0x2c, and 0x30.
7. Return the shared buffer pointer as evaluated at return time.

Each field produces exactly eight digits, including leading zeroes. Digits are emitted from the most significant nibble to the least significant nibble. Nibbles 0–9 produce bytes 0x30–0x39; nibbles 10–15 produce bytes 0x61–0x66.

If the shared pointer remains stable and the memory regions do not interfere, the resulting byte layout is:

| Buffer offsets | Contents |
|---|---|
| 0 | Asterisk |
| 1–8 | Field at record offset 0x20 |
| 9–16 | Field at record offset 0x24 |
| 17–24 | Field at record offset 0x28 |
| 25–32 | Field at record offset 0x2c |
| 33–40 | Field at record offset 0x30 |

There are no separators, labels, spaces, line breaks, or prefixes between fields.

No terminating zero byte is written. Byte 41 is outside the explicitly written output region. Treating the result as a terminated string requires an independently established terminator and adequate accessible storage.

The five fields are not captured as a single snapshot. Each field is read only after all eight digits of the preceding field have been written. Consequently, aliasing between output storage and the record, or intervening external mutation, can affect later reads. A replacement must not silently change this into five reads followed by all output writes if such memory interactions are observable.

The digit destination is retained across the dependency call. The returned pointer, however, comes from a later evaluation of the shared buffer pointer. If F-187ab9ba4eb6 changes that global pointer, the function may write digits through the previously retained destination and return a different pointer. Preservation of that global by the dependency is not established.

### 3. Select one field

For selectors 1, 2, 4, 8, and 16, the function:

1. Invokes F-187ab9ba4eb6 exactly once.
2. Reads the corresponding 32-bit field from the returned record.
3. Resolves that field as described below.
4. Returns the resulting pointer.

This path performs no explicit output-buffer writes.

The selected field is examined in its original 32-bit form before any low-bit masking.

### 4. Resolve the selected field

A selected field equal to zero returns the address of a fixed fallback object. The function does not inspect that object’s contents. Its size, mutability, representation, and lifetime guarantees are not established by the supplied body.

For a nonzero selected field:

- Clear its lowest two bits to obtain an aligned address value.
- Read a 32-bit quantity from four bytes before that aligned address.
- Add that quantity to the aligned address.
- Return the resulting address.

The displacement read is represented as a signed integer. For a valid 32-bit machine-address calculation, its bit pattern contributes to the returned address; a negative displacement can designate an address below the aligned address.

The low two bits are ignored only for this address calculation. Their meaning is unknown. They must not be interpreted as independently validated tags.

The zero test occurs before masking. Therefore, nonzero values 1, 2, and 3 do not select the fallback object. They produce a masked address of zero and attempt the displacement read four bytes below it. Under a wrapping 32-bit address model, that read address is 0xfffffffc. Whether such an access is valid is an environmental question.

There is no explicit rejection of an unexpected masked address, no validation of the displacement, and no dereference of the final computed result. A computed null pointer is not explicitly excluded.

## Side effects

### Explicit writes

The formatting path performs:

- One byte write for the asterisk.
- Forty byte writes for the hexadecimal digits.

No explicit write changes the shared buffer-pointer variable itself. The function does not allocate or free memory.

Individual selection and unsupported-selector handling contain no explicit memory writes.

### Reads and ordering

The significant visible ordering constraints are:

- On formatting paths, the prefix write occurs before the dependency call.
- Record reads occur after the dependency returns.
- Formatting alternates one field read with eight digit writes.
- Individual selection reads its field before deciding between fallback and reference resolution.
- A reference displacement is read only when the original selected field is nonzero.
- Unsupported selectors return without dependency calls or record reads.

These are static ordering observations, not guarantees about atomicity, hardware memory barriers, or concurrent visibility.

### Dependency effects

F-187ab9ba4eb6 may have effects not visible here. The supplied material does not establish that it preserves globals, avoids output-buffer mutation, retains memory ownership, or returns the same record on successive calls.

If it fails to return normally during formatting, the asterisk write has already occurred. This function provides no visible rollback mechanism.

### Ownership and lifetime

The function returns existing or computed addresses. It does not visibly transfer ownership, create a caller-owned copy, release a record, or retain an ownership token.

Callers must not infer permission to free the returned pointer. They also must not infer that formatted output remains unchanged after another operation using the same shared storage.

## Dependencies

### F-187ab9ba4eb6

**Invocation:** No explicit arguments; exactly one call on either a formatting path or an accepted individual-selection path.

**Consumed result:** A record base used for native 32-bit reads at offsets 0x20 through 0x30.

**Not established:** Return-value validity, error conventions, allocation behavior, ownership, lifetime, global preservation, concurrency properties, or side effects.

No other function dependency is visible in the supplied body.

### Non-function dependencies

The function also relies on:

- A shared pointer used for formatted output.
- Writable storage reached through that pointer.
- A fixed fallback object.
- Memory containing encoded-reference displacement words.
- The target’s 32-bit address arithmetic and native memory-access behavior.

These dependencies are externally supplied state, not objects initialized or validated by this function.

## Acceptance examples

The following are conditional acceptance examples derived from static evidence. No runtime tests have been performed.

### Unsupported selectors

For input zero, the result is null. F-187ab9ba4eb6 is not called, and no output-buffer write is performed.

The same applies to a single unsupported bit, such as 0x00000020 or 0x80000000.

### Multiple bits take precedence

Input 0x00000003 selects formatting rather than resolving either of its two supported bits.

Input 0x80000001 also selects formatting. The unsupported high bit does not cause rejection once the input contains at least two set bits.

### Fixed-width formatting

Assume:

- The dependency returns a readable record.
- Its five fields are, in order, 0x00000000, 0x00000001, 0x89abcdef, 0xffffffff, and 0x01234567.
- The shared buffer pointer remains stable.
- Output storage does not overlap the fields or other relevant state.

The output consists of an asterisk followed by these adjacent eight-byte segments:

| Segment | Bytes interpreted as text |
|---|---|
| First | `00000000` |
| Second | `00000001` |
| Third | `89abcdef` |
| Fourth | `ffffffff` |
| Fifth | `01234567` |

Exactly 41 output bytes are written. A preexisting byte immediately after them is not explicitly overwritten by this function.

### Zero selected field

For input 0x00000008, the function calls the dependency and reads the field at offset 0x2c. If that field is zero, it returns the fallback-object address without performing a displacement read.

### Low bits removed during resolution

For input 0x00000002, suppose the selected field is 0x00001003 and address 0x00000ffc contains a displacement of positive 0x20.

The masked address is 0x00001000, and the returned address is 0x00001020. The contents at 0x00001020 are not read by this function.

### Negative displacement

Suppose an accepted selector yields 0x00002000 and the displacement word at 0x00001ffc represents negative 0x10.

Provided the target permits the accesses and arithmetic, the returned address is 0x00001ff0.

### Nonzero field masking to zero

If a selected field is 1, the function does not return the fallback object. It attempts the displacement read associated with a masked address of zero. Successful completion is conditional on the target making that access and subsequent calculation valid.

### Dependency changes the shared pointer

If the formatting destination is retained from one buffer and the dependency changes the shared pointer to a second buffer, digit writes continue through the retained destination. The returned pointer is the shared pointer’s later value. Acceptance must not assume it identifies the bytes just formatted.

## Uncertainties

The following are **documented_only**:

- The semantic meaning of the five fields and selectors.
- The intended meaning of the low two reference bits.
- The type and contents of the fallback object.
- Whether output storage normally has a preexisting terminator.
- Output-buffer capacity and initialization.
- Whether record storage, output storage, and pointer state can alias.
- The dependency’s ownership and global-preservation contracts.
- Native byte order, alignment restrictions, and fault behavior.
- Thread safety, reentrancy, and asynchronous mutation rules.
- Exact address-overflow behavior outside valid target-address calculations.
- Any validity or usability guarantees for the returned computed pointer.

The presentation as text suggests diagnostic formatting, and clearing two low bits suggests a tagged reference encoding. These are inferred intents only; neither interpretation adds a behavioral guarantee.

## Evidence

The supplied static function representation establishes:

- The population-based dispatch preceding individual selection.
- The five accepted selectors and their record offsets.
- One dependency call per supported operation.
- The prefix write before the formatting-path dependency call.
- Five ordered 32-bit reads, each followed by eight digit writes.
- Lowercase, most-significant-nibble-first, fixed-width hexadecimal output.
- The absence of an explicit terminator write.
- The original-field zero check before low-bit masking.
- The displacement read four bytes before the masked address.
- Return-time evaluation of the shared output pointer.
- Null returns for zero and unsupported single-bit selectors.

The 32-bit model is supported by the supplied unsigned-word operations, 32-bit mask, shifts through bit 31, four-byte field spacing, and 32-bit reference representation. Machine-level overflow and ABI details beyond that representation remain unverified.

No implementation execution, runtime tests, independent dependency inspection, or memory-layout validation has been performed.
