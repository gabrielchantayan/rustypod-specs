<!-- Trusted provenance: F-df6609bc8e2d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-df6609bc8e2d

## Purpose

F-df6609bc8e2d consumes a prefix of input items and accumulates numeric digit values in a caller-supplied buffer. Recognition depends on a signed radix field in a mutable context. In radix 16, two configurable byte values can additionally be consumed without producing output.

Output consists of individual digit-value bytes, not character encodings or a completed numeric value. Leading zero values can be suppressed. The function can request buffer expansion and adjust two context-held addresses after that request.

Processing stops when an initial dependency reports a stop condition or when the current input item fails the acceptance rules. The return value is the current output cursor. There is no separate success, failure, or consumed-count result.

This contract describes behavior established by the supplied static evidence. It does not establish the semantic meaning of the dependencies, their memory ownership rules, or their failure behavior.

## Interface

### Arguments and result

| Item | Contract |
|---|---|
| Context address | Identifies mutable storage containing the fields listed below. It is passed directly to F-8ff6310c8504. |
| Initial output cursor | Address at which the next retained digit value would be written. Its original value also participates in leading-zero suppression throughout the call. |
| Zero-suppression control | A 32-bit integer. Zero disables suppression; every nonzero value enables it. |
| Return value | Current output cursor when processing stops. It can differ from the initial cursor because of writes or the result of buffer expansion. |

The supplied representation uses 32-bit integer and word operations. Pointer arithmetic and address-bearing fields are represented with the same width. Signed comparisons must not be replaced with unsigned comparisons. Arithmetic involving represented addresses must not silently gain wider host-language semantics.

### Context layout

Offsets are byte offsets from the context address.

| Offset | Established use |
|---|---|
| `+0x04` | Signed 32-bit radix, read during candidate acceptance. |
| `+0x0c` | Address-bearing 32-bit field conditionally rebased after expansion. |
| `+0x10` | A second address-bearing 32-bit field conditionally rebased after expansion. |
| `+0x14` | 32-bit extent used with the buffer base to identify the expansion boundary. Its address is passed to the expansion dependency. |
| `+0x18` | Address passed to the expansion dependency. Its type and occupied extent are unknown. |
| `+0x58` | Address-bearing 32-bit buffer-base field. Its address is passed to the expansion dependency. |
| `+0x6c` | 32-bit input-related value loaded separately for the inspection and consumption dependency calls. |

A separate global slot supplies an address at function entry. The function retains that address for the duration of the invocation. Bytes at offsets `+4` and `+3` from that address are potential radix-16 markers.

Calling these fields a “buffer base,” “extent,” and “radix” describes their observed roles. It does not establish a complete context structure or allocation contract.

### Preconditions

For defined execution, all reads, writes, and dependency calls reached on the selected path must be valid. In particular:

- The context must expose the fields accessed on that path.
- A retained digit requires a writable byte at the current output cursor after any expansion.
- A marker comparison requires readable storage at the cached global address.
- F-5a03f1b3b280 must return a usable cursor if processing subsequently writes through it.
- Address computations and saved-address rebasing must be meaningful under the target’s 32-bit representation.

The function does not visibly enforce these conditions. It does not validate a null context, null cursor, allocation failure, cursor ordering, or a cursor already beyond the expansion boundary.

## Behavior

### Iteration and stopping order

At entry, the function reads the global address used for marker comparisons and initializes its current output cursor from the supplied cursor.

Each iteration has this observable order:

1. Call F-8ff6310c8504 with the context address.
2. If its result is nonzero, return the current output cursor immediately.
3. Load the context field at `+0x6c` and pass its value to F-a3abf174e781.
4. Interpret that dependency’s result as a 32-bit input value and derive a signed candidate digit.
5. Apply radix and marker acceptance rules.
6. For an accepted item, check the output boundary and possibly request expansion.
7. Write a digit byte or suppress output, according to the candidate and suppression rule.
8. Load `+0x6c` again and pass that value to F-1c4d822fa26c.
9. Retain the resulting output cursor and begin the next iteration.

A rejection in step 5 returns before the boundary check, any explicit output write, or the call to F-1c4d822fa26c for that iteration.

This ordering does not prove that a rejected input item remains untouched: F-8ff6310c8504 and F-a3abf174e781 may themselves have side effects. The narrower guarantee is that the explicit consumption dependency is not called on that rejection path.

There is no iteration limit or output-length limit independent of the context and dependencies.

### Candidate digit derivation

For ordinary byte-valued input:

- `0` through `9` produce values 0 through 9.
- `A` through `F` produce values 10 through 15.
- `a` through `f` produce values 10 through 15.
- Every other input produces a negative candidate.

The exact negative value is not always −1. The first calculation subtracts hexadecimal `0x30` from the full 32-bit input result and interprets the wrapped result as signed. If that signed result is at most 9, it is retained directly, including when it is negative.

Consequently, byte values below `0x30` initially produce values from −48 through −1. Other ordinary non-digit bytes produce −1 after the alphabetic checks.

This distinction matters because the signed radix comparison occurs before negative candidates are handled. It is not correct to replace all invalid inputs with −1 before that comparison.

The input dependency is not visibly restricted to returning a byte. For wider results:

- A negative signed result of the initial wrapped subtraction remains a negative candidate.
- A nonnegative result from 0 through 9 remains that digit.
- Recognized alphabetic values produce 10 through 15.
- Other values produce −1.

Marker matching uses the original full 32-bit input value, not a truncation to its low byte.

### Radix and marker acceptance

The first acceptance gate reads the signed radix and compares it with the signed candidate.

If the radix is less than or equal to the candidate, the function returns the current cursor. Equality therefore rejects the candidate.

If the candidate is nonnegative and passes this gate, it is accepted. There is no separate radix-range validation. For example, a radix greater than 16 permits all recognized digit values, but does not add recognition of letters beyond `F` or `f`.

A negative candidate must pass an additional gate:

1. Read the radix again.
2. Require it to equal 16.
3. Read the byte at offset `+4` from the cached global address.
4. If that byte does not equal the full input value, read the byte at offset `+3`.
5. Require one of those comparisons to match.

Failure returns the current cursor.

The first marker byte is always examined before the second; the second is not read when the first matches. Both marker bytes are zero-extended for comparison with the full input value.

Marker recognition applies only to negative candidates. A configured marker whose byte also denotes a recognized digit does not override digit behavior.

An accepted negative candidate is consumed without producing output. The function does not visibly restrict markers to a prefix position, a single occurrence, or a position before any retained digits.

### Expansion and cursor repair

For each accepted item, including a zero that may be suppressed or a marker that produces no output, the function reads the buffer base at `+0x58` and compares the current cursor with:

- that base plus the extent at `+0x14`.

Expansion occurs only on equality. A cursor beyond that boundary does not trigger a separate visible check.

When equality holds:

1. Save the pre-call values at `+0x0c` and `+0x10`.
2. Call F-5a03f1b3b280 with addresses of the context locations `+0x58`, `+0x18`, and `+0x14`, in that order.
3. Replace the current output cursor with the dependency’s return value.
4. Read the post-call field at `+0x0c`. If nonzero, overwrite it with the then-current buffer base plus the saved first address’s displacement from the old buffer base.
5. Read the post-call field at `+0x10`. If nonzero, overwrite it using the analogous saved displacement and a newly read buffer base.

The two repairs occur in that order.

Eligibility for repair depends on each field’s value after the dependency returns. The displacement uses its saved pre-call value. A dependency that changes a field between zero and nonzero therefore affects whether the repair occurs, without changing the saved displacement.

There is no visible failure branch after expansion. A null or otherwise invalid returned cursor is not converted into an error result before the remaining processing.

### Output retention and zero suppression

After any expansion:

- A positive candidate is written as one byte at the current cursor; the cursor advances by one byte.
- A zero candidate is written and advances the cursor when suppression is disabled.
- With suppression enabled, a zero is written only when the current cursor differs from the original cursor argument.
- A negative accepted candidate produces no explicit byte write and does not advance the cursor.

Positive candidates are limited to 1 through 15, so their byte representation is the numeric digit value without loss.

The zero-suppression comparison uses the original cursor argument, not an output count and not a relocated version of that original address. If expansion returns a different cursor, a zero can be retained even when no earlier digit has been explicitly emitted.

The function does not append a terminator.

### Consumption

After acceptance and any output action, F-1c4d822fa26c is called with a fresh load of `+0x6c`. This call occurs for retained digits, suppressed zeros, and accepted markers.

The value passed to this dependency need not equal the value previously passed to F-a3abf174e781. Intervening dependencies may modify the context.

The consumption dependency’s result is not used for a visible branch or error result.

## Side effects

Established direct effects are:

- Calls to F-8ff6310c8504 and, when processing continues, F-a3abf174e781.
- Optional calls to F-5a03f1b3b280.
- Optional byte writes containing digit values.
- Conditional overwrites of context fields `+0x0c` and `+0x10`.
- Calls to F-1c4d822fa26c for accepted items.

No direct allocation or deallocation is visible. Allocation, relocation, release of old storage, and ownership transfer by dependencies are undocumented. Neither the original cursor nor the returned cursor is established to be an allocation base.

The cached marker address is not reloaded after dependency calls. Marker contents are read when needed, however; they are not snapshotted at entry. Dependencies that change the global slot do not redirect this invocation’s marker reads, while changes to storage at the cached address can affect subsequent comparisons.

There is no established purity, global-preservation, or memory-preservation guarantee for any dependency.

## Dependencies

| Opaque ID | Established call contract | Not established |
|---|---|---|
| F-8ff6310c8504 | Receives the context address at the start of every iteration. Any nonzero result ends processing. | Meaning of the stop condition; whether it changes input, context, or globals. |
| F-a3abf174e781 | Receives the current 32-bit field at `+0x6c`; returns the value classified by this function. | Result range, end-of-input representation, inspection purity, or whether it advances input. |
| F-5a03f1b3b280 | Receives addresses of `+0x58`, `+0x18`, and `+0x14`; its return becomes the output cursor. | Allocation strategy, capacity guarantee, failure signaling, ownership, or preservation of other context fields. |
| F-1c4d822fa26c | Receives a newly loaded value from `+0x6c` after an item is accepted. | Exact consumption semantics, failure behavior, or preservation of output and context state. |

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Except where stated otherwise, assume the initial stop dependency returns zero, the input dependency supplies the described values, storage is adequate, and dependencies do not alter relevant state unexpectedly.

### Ordinary digits

With radix 10 and suppression disabled, input values corresponding to `0`, `7`, and `9` cause explicit output writes of byte values 0, 7, and 9. Each is followed by a consumption call.

An ensuing `A` produces candidate 10. Because radix 10 is less than or equal to 10, that iteration returns without a digit write or consumption call.

### Hexadecimal digits and out-of-range digits

With radix 16, `F` and `f` each produce byte value 15.

With radix 15, either is rejected before boundary checking or consumption because equality with the radix is rejecting.

### Leading zeros

With suppression enabled, radix 10, no expansion, and input `0`, `0`, `5`, `0`:

- Both initial zeros are accepted and consumed without explicit output.
- The `5` writes byte value 5 and advances the cursor.
- The final zero writes byte value 0 because the cursor now differs from the original argument.

The explicit output is two bytes: 5 followed by 0.

### Marker without output

Suppose the first configured marker byte is `.` and radix is 16. Input `.` derives a negative candidate, passes the marker comparison, reaches the boundary check, and is followed by consumption without an explicit output byte.

If the radix is 10 instead, it is rejected before marker-byte access.

### Marker does not override digit recognition

Suppose a configured marker byte is `A`. With radix 16, input `A` still produces byte value 10. With radix 10, it is rejected by the radix gate; it is not reconsidered as a marker.

### Expansion before suppression

Suppose suppression is enabled, the first input is `0`, and the cursor equals the expansion boundary. Expansion occurs before the zero-suppression decision.

If the returned cursor differs from the original cursor argument, the zero is explicitly written. If the returned cursor equals that original argument, the zero is suppressed. Both paths call the consumption dependency.

### Rejection avoids explicit expansion

If radix 2 receives `2`, candidate 2 fails the radix gate. Even if the cursor equals the buffer boundary, this function does not call the expansion dependency for that item.

### Immediate stop

If the first F-8ff6310c8504 call returns nonzero, the result is the original output cursor. The entry-time global-address read has occurred, but there is no input-inspection, expansion, explicit output, or consumption call.

## Uncertainties

The following remain **documented_only**:

- The semantic identity of the context, stream-like value, marker storage, and address-bearing fields.
- Whether F-a3abf174e781 is a non-consuming inspection operation.
- Whether F-1c4d822fa26c consumes exactly one item or can fail.
- Whether F-8ff6310c8504 distinguishes exhaustion, errors, or other stop conditions.
- Whether expansion allocates, reallocates, frees, or transfers ownership.
- Whether the extent is always nonnegative and consistent with the current cursor.
- Which fields or globals dependencies preserve.
- Whether relocated-cursor zero retention is intentional.
- Concurrency, reentrancy, and behavior after invalid memory access or dependency failure.

Inferred intent is radix-dependent digit accumulation with leading-zero suppression and special radix-16 syntax handling. That interpretation does not strengthen the established guarantees.

## Evidence

The supplied static function representation establishes the field offsets, 32-bit arithmetic and comparisons, dependency call order, marker-read order, expansion condition, address-repair rules, explicit byte writes, and return paths described above.

No dependency implementations or runtime observations were supplied. No runtime tests have been performed. The acceptance examples are conditional consequences of the visible control flow, not claims of observed execution.
