<!-- Trusted provenance: F-6b019ff6e1cb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6b019ff6e1cb

## Purpose

F-6b019ff6e1cb conditionally obtains a text reference from a caller-supplied record, removes a classification-controlled prefix and at most one sign, and delegates conversion to a 64-bit conversion dependency. It then applies sign-dependent range handling. If a selection gate fails, it returns a caller-supplied 64-bit fallback.

The text-to-integer purpose is inferred from the operations. The established contract is the observable sequence of record reads, dependency calls, unsigned selection arithmetic, text scanning, result-bit processing, and conditional error-state writes. The application meanings of the record, selection, mode, limits, and error value are not established.

## Interface

### Arguments

| Position | Established interpretation |
|---|---|
| First | Reference to a caller record containing 32-bit fields and an overlapping byte access. |
| Second | Opaque 32-bit argument with no direct use in the visible body. |
| Third | Low 32 bits of the fallback result. |
| Fourth | High 32 bits of the fallback result. |

The fallback is the exact concatenation of the fourth argument above the third. The third argument is not sign-extended. All 64-bit fallback patterns are permitted by the visible construction and are returned without range adjustment.

The second argument’s lack of direct use does not establish its irrelevance to an incompletely represented calling convention.

### Referenced storage

The representation uses 32-bit stored references.

| Storage base | Byte offset | Visible access |
|---|---:|---|
| Caller record | 0 | Repeated 32-bit object-reference reads. |
| Caller record | 4 | Conditional unsigned byte read for the initial gate. |
| Caller record | 4 | Later 32-bit read, converted to a character-sized value and compared with 2. |
| Selected object | 20 | Read of a 32-bit entry-table reference. |
| Selected entry | 16 | Read of a 32-bit text reference. |
| Classification descriptor | 0 | Read of a 32-bit classification-table reference. |
| Error-state storage | 0 | Conditional 32-bit write of 2. |

Entry selection uses a stride of 40 bytes. No other entry fields are directly accessed.

The byte read and the later word-to-character conversion at caller-record offset 4 are separate operations. Their relationship depends on byte order, character-conversion semantics, and intervening changes to the record.

### Result and preconditions

The output is a 64-bit result. Converted results undergo signed-range handling; fallback bits do not.

Memory-safe execution requires every reached access to be valid:

- The caller record must support the reads actually performed, including the second object-reference read when the first returns zero.
- Reached object, table, entry, text, and classification references must support their respective reads.
- Prefix scanning must reach a stopping byte before encountering inaccessible storage.
- An error-state pointer returned on a range-failure path must support a 32-bit write.

The function does not visibly validate table allocation size, text references, classification storage, or error-state pointers. It provides no established recovery contract for invalid memory.

## Behavior

### Initial reads and selection gate

F-7405054f15fb is called before any visible caller-record inspection. Its result supplies an unsigned 32-bit selection value.

The initial gate does **not** use one cached object reference:

1. The first object-reference read determines whether the enabling byte at record offset 4 is read.
2. A second object-reference read supplies the object condition for the outer gate.

The gate’s other operand initially contains the selection value. Only a nonzero **first** object-reference read replaces that operand with the unsigned enabling byte.

Consequently:

| First object read | Second object read | Gate behavior |
|---|---|---|
| Zero | Zero | No enabling-byte read; fallback. |
| Zero | Nonzero | No enabling-byte read; proceed only if the initial selection is nonzero. |
| Nonzero | Zero | Enabling byte was read; fallback regardless of its value. |
| Nonzero | Nonzero | Proceed only if the previously read enabling byte is nonzero. |

Thus, “a zero object reference causes fallback” is a valid simplification only when these reads observe stable contents. A zero first read alone does not establish fallback.

If the outer gate fails, neither limit dependency call nor text conversion is reached.

### Limit checks and mode-dependent selection

After the outer gate passes, F-e84094e554a6 is called with no explicit arguments in the supplied representation. Its unsigned 32-bit result is the first limit.

The initial selection must be less than or equal to this limit. A greater selection returns the fallback without the mode-field read or second limit call. Equality passes.

After the first check succeeds, the function reads the 32-bit field at record offset 4 and converts it to a character-sized value. If the converted value equals 2, the selection becomes twice the initial selection plus one, evaluated modulo \(2^{32}\). Otherwise, selection is unchanged.

F-e84094e554a6 is then called again, this time with an explicit argument obtained from a fresh object-reference read. Its unsigned 32-bit result is the second limit.

The possibly transformed selection must be less than or equal to the second limit. A greater value returns the fallback; equality passes.

These calls occur at different times and have different represented inputs. Neither their results nor the object references may be treated as interchangeable or cached across the operation.

### Entry-address arithmetic

After the second check succeeds, the object reference is read again. The function obtains the entry-table reference from offset 20 of that object.

For table-base value \(B\) and selected unsigned value \(S\), the text-reference slot address is:

\[
A = (B + 40S + 16) \bmod 2^{32}.
\]

The multiplication and additions use 32-bit unsigned arithmetic. A large selection can therefore wrap both the product and the resulting slot address. The function reads the 32-bit text reference at this effective address.

The inclusive limit checks do not establish allocation bounds or prevent address wraparound. Even a permitted selection requires valid storage at the effective address actually reached.

The object passed to the second limit call need not be the object subsequently used for table access if the record changes between reads. No additional nonzero check protects the later object reads.

### Classification-controlled prefix

The selected text is examined byte by byte. Each byte is read as an unsigned value.

A zero byte immediately ends prefix scanning and does not cause a classification call.

For every nonzero byte examined, F-2a4c402ba617 is called. The returned address is used to read a 32-bit classification-table reference. The previously read text-byte value indexes a byte in that table.

- Classification bit 0 set: continue at the next text byte.
- Classification bit 0 clear: stop at the current text byte.

The dependency is called separately for every examined nonzero byte, including the byte that ends scanning through a clear classification bit. Descriptor and table-reference state is reacquired on each such iteration.

The behavior resembles whitespace skipping, but no particular character class is established. Classification bit 0, rather than a fixed character list, determines what is skipped. A plus or minus byte whose classification bit is set is skipped as part of the prefix rather than treated as a sign at that position.

The text byte is read before its classification dependency call. Classification indexing and subsequent sign recognition use that saved byte value; the visible body does not reread the stopping byte after the call.

### Sign removal and conversion

The byte that ends scanning determines the conversion pointer and sign condition:

- Byte 43, plus: advance past it; do not record a minus condition.
- Byte 45, minus: advance past it and record a minus condition.
- Any other byte, including zero: use the address of that stopping byte.

Only one sign is explicitly removed. The function does not perform another classification scan or independently remove a second sign afterward.

F-c4e0ec1bb29e is then called with the resulting text pointer, followed by two zero-valued arguments. It returns a 64-bit bit pattern.

The meanings of the zero arguments, accepted grammar, radix, treatment of trailing text, and converter-internal error behavior are unresolved. This wrapper does not visibly inspect an end pointer or independently determine whether any digit was consumed.

### Sign-dependent range handling

Let \(M\) be the converter’s returned bit pattern interpreted as an unsigned 64-bit value.

**Without a removed minus sign:**

- If \(M < 2^{63}\), return the bits unchanged.
- Otherwise, perform the error-state operation described below and return \(2^{63}-1\).

**With a removed minus sign:**

- Form the 64-bit two’s-complement negation of \(M\).
- If \(M \leq 2^{63}\), return that negated bit pattern.
- If \(M > 2^{63}\), perform the error-state operation and return \(-2^{63}\).

The negative-path threshold follows from the visible word arithmetic and signed-overflow predicates. It is not an assumption about what the converter intends to return.

In particular, \(M=2^{63}\) is accepted only on the removed-minus path, where it produces the signed minimum. A removed minus applied to zero produces zero.

### Error-state operation and represented inputs

On wrapper-detected range failure, F-4af36158f863 returns a pointer. The function writes the 32-bit value 2 through that pointer before returning the applicable saturated result.

The dependency’s represented inputs differ by path:

| Range-failure path | Represented arguments, in order |
|---|---|
| No removed minus | Original converter-result low 32-bit word; original high 32-bit word. |
| Removed minus | Original converter-result low word; original high word; two zero words; five opaque 32-bit values originating from unspecified incoming registers. |

On the minus path, the second argument is calculated through intermediate word arithmetic, but its 32-bit value equals the converter result’s original high word. The first two arguments are not the words of the negated result.

The semantic significance of these arguments, and the accuracy or completeness of the represented calling convention, remain unknown. They must not be omitted from the contract or assumed irrelevant.

## Side effects

The visible body performs no explicit allocation, deallocation, or writes to the caller record, object, entry table, or text.

Its only explicit memory write is the conditional 32-bit store of 2 through the error-state pointer. It does not explicitly clear that storage on successful conversion or fallback. Dependencies may independently modify it or other state.

Observable ordering includes:

- Initial selection dependency before record reads.
- First object-reference read before the conditional enabling-byte read, then a distinct object-reference read for the gate.
- First limit call before the mode-field read.
- Fresh object-reference read for the second limit call.
- A further object-reference read and table access only after the second check passes.
- Each text-byte read before its classification dependency call and classification-table access.
- Conversion after prefix and sign processing.
- Error-state dependency call before the explicit error-state store.

Repeated record reads and classification acquisitions are behaviorally significant. Unknown callees must not be assumed to preserve records, globals, text, classification tables, or pointer lifetimes.

No visible operation expressly transfers ownership, and the body does not explicitly release any referenced storage. Whether dependencies retain, allocate, release, or invalidate memory is unresolved.

## Dependencies

| Opaque ID | Established use | Unresolved aspects |
|---|---|---|
| F-7405054f15fb | Unconditional initial call returning the 32-bit selection. | Inputs, state dependence, effects, and calling convention. |
| F-e84094e554a6 | Two conditional calls returning unsigned 32-bit limits; only the second has an explicit object-reference argument. | Limit meanings, omitted inputs, and effects. |
| F-2a4c402ba617 | Per-nonzero-byte call returning an address used to obtain the classification-table reference. | Classification meaning, storage lifetime, mutability, and effects. |
| F-c4e0ec1bb29e | Receives adjusted text pointer and two zero arguments; returns 64 bits. | Parsing rules, errors, memory access, and ownership behavior. |
| F-4af36158f863 | Receives path-dependent represented arguments and returns a pointer for a 32-bit store. | Argument meanings, storage identity, thread locality, lifetime, and other effects. |

No dependency is established as pure.

## Acceptance examples

These are conditional static acceptance cases, not runtime observations. Assume valid reached memory and normally returning dependencies. Except where changing reads are expressly specified, assume stable referenced state.

1. **Stable absent object.** Both initial object-reference reads return zero. The initial selection dependency still runs. No enabling byte, limit, text, or converter access follows; the exact fallback is returned.

2. **Changing object reference, retained selection.** The first object read returns zero and the second returns nonzero. No enabling byte is read. Initial selection 3 permits the outer gate to reach the first limit call; initial selection zero instead produces fallback.

3. **Object disappears at the gate.** The first object read is nonzero, so the enabling byte is read. If the second object read is zero, fallback follows even when that byte was nonzero.

4. **Stable disabled record.** Both object reads are nonzero and the enabling byte is zero. Neither limit call occurs.

5. **Inclusive first limit.** Initial selection 7 and first limit 7 pass. First limit 6 causes fallback before the mode read and second limit call.

6. **Mode transformation.** Initial selection 3 and character-sized mode value 2 produce selection 7. Second limit 7 permits table access; second limit 6 does not.

7. **Selection wraparound.** Initial selection \(2^{31}\), an adequate first limit, and mode value 2 produce selection 1, not \(2^{32}+1\).

8. **Entry-address wraparound.** Selection \(2^{29}\) passes both limits without transformation, and the table base is 4096. Since \(40 \times 2^{29}\) is zero modulo \(2^{32}\), the text-reference slot is address 4112. This case requires that effective address to be readable; the function supplies no allocation protection.

9. **Classification-controlled prefix.** Two nonzero bytes have classification bit 0 set; the next byte has it clear. Three classification calls occur. If the stopping byte is not a sign, conversion receives its address.

10. **Empty text.** The first text byte is zero. No classification call occurs, and conversion receives that zero byte’s address.

11. **Single minus removal.** A minus byte stops scanning. Conversion receives the following address. Any following space or second sign is left to the converter.

12. **Positive boundary.** Without a removed minus, converter bits \(2^{63}-1\) are returned unchanged. Bits \(2^{63}\) instead cause an error-state call with low word zero and high word \(2^{31}\), a 32-bit store of 2, and return of \(2^{63}-1\).

13. **Exact negative boundary.** With a removed minus, converter bits \(2^{63}\) return \(-2^{63}\) without the wrapper’s error-state call or store.

14. **Negative range failure and call inputs.** With a removed minus, converter bits \(2^{63}+1\) cause the error-state call with first words 1 and \(2^{31}\), then two zero words and the five incoming-register-derived values. After storing 2, the function returns \(-2^{63}\).

## Uncertainties

The following remain **documented_only**:

- Application meanings of the record, mode, selection, limits, and stored error value.
- Target byte order and exact word-to-character conversion semantics.
- Completeness and accuracy of represented dependency calling conventions.
- Meaning of the error-state dependency’s explicit arguments and incoming-register-derived values.
- Conversion grammar, radix, digit requirements, trailing-input treatment, and internal range or error behavior.
- Meaning of classification bit 0.
- Whether limits represent maximum indices, counts, or another bound.
- Dependency effects on storage, ownership, globals, and pointer lifetimes.
- Fault behavior for invalid addresses or scans that do not reach accessible stopping storage.
- Concurrency guarantees and the conditions under which repeated reads may differ.

## Evidence

Established behavior comes solely from static inspection of the supplied representation: branches, distinct reads, call ordering and represented arguments, field offsets, 32-bit selection and entry-address arithmetic, fallback construction, and 64-bit result processing.

The negative-range threshold and restoration of the original high word in the negative-overflow call are arithmetic consequences of the visible word operations. Selection, classification, and conversion terminology describes inferred roles, not established application semantics.

The supplied material reports removed unreachable regions whose contents are unavailable. No behavior is reconstructed from them.

No runtime tests have been performed. Dependency implementations, external calling-convention documentation, and target execution behavior were not available.
