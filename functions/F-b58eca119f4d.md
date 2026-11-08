<!-- Trusted provenance: F-b58eca119f4d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b58eca119f4d

## Purpose

F-b58eca119f4d produces a 64-bit result from a caller-supplied, word-addressed object. It selects among three paths according to flags in that object:

1. Return a stored 64-bit value.
2. Convert a stored signed 64-bit integer into a binary64-compatible bit pattern.
3. Attempt a dependency-mediated conversion, otherwise return a default 64-bit value.

The flag tests have strict precedence. A stored-value flag overrides the integer flag, and the integer flag overrides the dependency-mediated conversion flags.

The numerical interpretation of the integer path is strongly supported by its sign handling, normalization, exponent construction, and rounding logic. The meanings of the object as a whole, the default value, and the dependency-mediated conversion are not established.

## Interface

### Inputs

The function accepts four arguments:

| Argument | Established use |
|---|---|
| Object address | Address of storage accessed as 32-bit words, with a 16-bit write on one path. |
| Second argument | A 32-bit argument with no visible use in this function. |
| Third argument | A 32-bit value forwarded unchanged to F-98085658bfd4 on the dependency-mediated path. |
| Fourth argument | A 32-bit value forwarded unchanged to F-98085658bfd4 on the dependency-mediated path. |

The second argument is neither inspected nor forwarded by the visible implementation.

### Object layout

The following offsets are relative to the object address. Each listed word is 32 bits wide.

| Byte offset | Role established by visible accesses |
|---|---|
| 0 | Low-order word of the integer-path input. |
| 4 | High-order word of the integer-path input; bit 31 supplies its sign. |
| 8 | Low-order word of the stored-value result. |
| 12 | High-order word of the stored-value result. |
| 20 | A value read after successful prerequisite calls and passed to F-d920a917d399. |
| 28 | Flags word, initially read for path selection. Also the address of a later 16-bit write. |

No visible access establishes a role for the other object bytes.

“Low-order” and “high-order” describe the significance assigned to the loaded words, not a claim about the target’s byte order within each word.

### Output

The function returns 64 bits. For any returned pair of 32-bit words, the word identified as high-order supplies bits 63–32, and the low-order word supplies bits 31–0.

The visible return type does not establish a language-level floating-point return convention. The integer conversion path constructs a bit pattern consistent with IEEE 754 binary64; callers must not infer an ABI-level floating-point return solely from that observation.

### Preconditions

The object address must support every access reached on the selected path:

- All paths read the 32-bit flags word at offset 28.
- The stored-value path reads offsets 8 and 12.
- The integer path reads offsets 0 and 4.
- The dependency-mediated path writes 16 bits at offset 28 and may later read offset 20.

Required alignment, fault behavior, and additional requirements imposed by dependencies are not established. There is no visible null check, bounds check, object validation, or synchronization.

## Behavior

### 1. Path selection

The function first reads the 32-bit flags word at offset 28.

It selects the first applicable case below:

| Condition on the initially read flags | Selected behavior |
|---|---|
| Bit mask `0x08` is present | Return the stored value. |
| Otherwise, bit mask `0x04` is present | Convert the stored signed integer. |
| Otherwise, at least one bit in mask `0x12` is present | Attempt dependency-mediated conversion. |
| Otherwise | Return the default value. |

Mask `0x12` tests bits `0x02` and `0x10`. Either bit suffices.

Subsequent changes to the flags do not visibly cause path selection to restart.

### 2. Stored-value return

On this path, the function reads:

1. The 32-bit word at offset 8.
2. The 32-bit word at offset 12.

It returns those words as the low and high halves, respectively.

No dependency is called, and no object write is visible on this path. The integer words and the field at offset 20 are not read by the visible function.

The returned bits are not checked, normalized, or interpreted. If the stored words encode a floating-point special value, an unusual payload, or arbitrary non-floating-point data, those bits are returned unchanged.

### 3. Signed integer conversion

The function reads the low-order input word at offset 0, followed by the high-order input word at offset 4. Together they form a signed, two’s-complement 64-bit integer.

#### Sign and magnitude

The high input word’s top bit determines the output sign.

For a negative input, the function obtains the unsigned magnitude through two-word two’s-complement negation. Arithmetic within each word is modulo \(2^{32}\). The carry relationship between the words is handled explicitly.

This permits the minimum signed 64-bit value, \(-2^{63}\), to have magnitude \(2^{63}\) without requiring a representable positive signed 64-bit intermediate.

For zero, the function returns 64 zero bits. There is no separate negative-zero integer representation.

#### Normalization and precision

For a nonzero magnitude, the function identifies its leading nonzero word and counts leading zero bits. It then constructs:

- A normalized leading portion.
- A lower portion containing retained precision and rounding information.
- An exponent contribution derived from the position of the highest set bit.

The output construction uses a sign bit, an 11-bit exponent position, and a 52-bit fraction position consistent with binary64.

Under the shift behavior discussed in **Uncertainties**, this conversion rounds to the nearest representable binary64 value, resolving exact halfway cases toward the result with an even least-significant significand bit.

The rounding decision distinguishes:

- A discarded portion below halfway: do not increment.
- A discarded portion above halfway: increment.
- An exact halfway portion: increment only if the retained significand is odd.

Carry propagation from the low output word into the high output word is explicit. A rounding increment can therefore cross a word boundary and, when necessary, increase the encoded exponent.

#### Range and special values

The signed 64-bit input range does not require binary64 overflow or subnormal results. A nonzero integer magnitude is at least 1 and at most \(2^{63}\).

The visible integer path has no dependency calls, no floating-point exception check, and no object write. Its arithmetic is performed through integer operations rather than a visible floating-point conversion instruction.

### 4. Dependency-mediated conversion

This path is entered only if neither higher-priority flag applies and at least one bit in `0x12` is present.

The visible sequence is:

1. Read a default low-order word from one external data location into local result storage.
2. Read a default high-order word from a second external data location into local result storage.
3. Read the object’s flags word again.
4. At byte offset 28, perform a 16-bit write of the low 16 bits of that newly read flags value, with bit `0x02` set.
5. Call F-98085658bfd4 with the object address, the 32-bit value 1, the third argument, and the fourth argument.
6. If that call returns nonzero, stop the conversion attempt and return the default value through the fallback behavior below.
7. If it returns zero, call F-94f17d1d24a0 with the object address.
8. If that call returns nonzero, return the default value through the fallback behavior below.
9. If both calls return zero, read the field at offset 20 from the object’s current state.
10. Call F-d920a917d399 with that field value and the address of the local result storage.
11. After F-d920a917d399 returns normally, return the current contents of that local storage as a 64-bit result.

The first two dependency results are treated only as zero versus nonzero. No distinction among nonzero values is visible.

F-d920a917d399’s return value is not inspected. Its writes, if any, to the supplied result storage determine the returned words. The local storage is initialized before any dependency call, so bytes not changed by dependencies retain their initialized values, subject to any other dependency effects that cannot be resolved from the supplied body.

There is no visible rollback of the 16-bit flag write if either prerequisite returns nonzero.

#### Default-value fallback

The fallback return reads the two external default-value locations again and returns their then-current values.

Consequently, the defaults initially placed in local result storage are not necessarily the defaults returned after a failed prerequisite. A dependency could change the external locations before fallback reaches them.

Likewise, the offset-20 field is read only after both prerequisite calls have returned zero. Its value is not a snapshot taken at function entry.

### 5. Direct default return

If none of the path-selection masks matches, the function reads the external low-order default word and the external high-order default word and returns their concatenation.

No dependency call or object write is visible on this path.

The supplied material does not reveal the default words’ contents. They must not be labeled as zero, NaN, an error sentinel, or any other specific value without additional evidence.

## Side effects

### Directly visible effects

The only visible write to caller-provided object storage is the 16-bit write at offset 28 on the dependency-mediated path.

Its stored numeric value is the low 16 bits of a fresh flags-word read, with bit `0x02` set. Because the store is narrower than the flags read, it is not a 32-bit flags assignment.

The exact effect on the numeric value of a later 32-bit flags read depends on byte order. On a little-endian target, the write sets bit `0x02` while preserving the other bits of the flags word. Target byte order is not independently established by the supplied material.

Local result storage is initialized from the default words and supplied by address to F-d920a917d399.

### Dependency effects and ownership

The function contains no visible allocation or release operation. This does not establish that its dependencies allocate nothing, release nothing, or leave globals unchanged.

The object address is passed to two dependencies. Their possible mutations, ownership actions, and retained references are unknown.

The value at offset 20 is passed to F-d920a917d399 without a visible ownership annotation. Its interpretation as an address, handle, scalar, or owned resource is not established.

The result-storage address supplied to F-d920a917d399 refers to temporary local storage used during this invocation. No transfer of that storage’s ownership is established. Retaining and later accessing the address would require a lifetime contract not provided here.

### Error behavior

There is no separately identified error return channel. Nonzero prerequisite results lead to the default-value return, but the meaning of that default is unknown.

No visible recovery handles invalid memory, dependency failure to return, or other exceptional control transfer. The specified normal-return behavior assumes all reached accesses and calls complete normally.

## Dependencies

| Opaque function ID | Invocation and visible contract |
|---|---|
| F-98085658bfd4 | Called first on the dependency-mediated path, after local default initialization and the 16-bit object write. Receives the object address, 1, and the third and fourth input arguments. Zero permits the next call; nonzero selects fallback. |
| F-94f17d1d24a0 | Called only after F-98085658bfd4 returns zero. Receives the object address. Zero permits the final conversion call; nonzero selects fallback. |
| F-d920a917d399 | Called only after both prerequisite calls return zero. Receives the freshly read offset-20 field and a writable address for the initialized local result storage. Its return value is ignored. |

Two external 32-bit data locations supply the default low and high words. Their contents, mutability constraints, and consistency guarantees are unknown.

No dependency is assumed to preserve globals, preserve the object, or retain or relinquish ownership.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests. Memory accesses and dependencies are assumed to complete normally.

### Selection and stored values

- If the initial flags contain both `0x08` and `0x04`, the stored-value path wins. The integer input does not affect the result.
- If the stored low word is `0x89ABCDEF` and the stored high word is `0x01234567`, the result is `0x0123456789ABCDEF`.
- If the initial flags contain `0x04` and `0x10`, but not `0x08`, integer conversion wins. No prerequisite call is made.
- If all tested mask bits are absent, the result is the externally supplied default pair.

### Integer conversion

Under the target-shift interpretation supporting nearest-even binary64 conversion:

| Signed input | Expected result |
|---|---|
| 0 | Positive-zero bit pattern, `0x0000000000000000`. |
| 1 | Binary64 representation of 1, `0x3FF0000000000000`. |
| −1 | Binary64 representation of −1, `0xBFF0000000000000`. |
| \(2^{53}+1\) | Rounds to \(2^{53}\), the even-significand neighbor. |
| \(2^{53}+3\) | Rounds to \(2^{53}+4\), the even-significand neighbor. |
| \(2^{63}-1\) | Rounds to \(2^{63}\). |
| \(-2^{63}\) | Exactly represents \(-2^{63}\), with bit pattern `0xC3E0000000000000`. |

### Dependency ordering and mutable state

- If F-98085658bfd4 returns nonzero, neither later dependency is called. The 16-bit object write has already occurred, and fallback reads the external defaults again.
- If F-98085658bfd4 returns zero and F-94f17d1d24a0 returns nonzero, F-d920a917d399 is not called.
- If both prerequisites return zero and change the field at offset 20, F-d920a917d399 receives the field’s value after those changes.
- If a prerequisite changes the external default words and then selects fallback, the returned words come from the fallback reads, not the earlier local initialization.
- If both prerequisites return zero and F-d920a917d399 leaves the result storage untouched, the returned result is the pair initialized before the prerequisite calls.

## Uncertainties

### Established static evidence

The supplied body establishes branch precedence, accessed offsets, access widths, two-word result assembly, dependency order, zero/nonzero gating, default initialization, fallback rereads, and explicit carry-aware rounding construction.

### Inferred intent

The integer path is intended to convert a signed 64-bit integer into binary64 bits with round-to-nearest, ties-to-even behavior. The flag meanings “stored value,” “integer,” and “conversion eligible” are descriptive labels, not independently documented object semantics.

### documented_only

- **Target shift semantics:** Some displayed shift counts become negative or reach/exceed 32 before being masked to eight bits. A language-neutral reading cannot assign these the ordinary semantics of every implementation language. The binary64 interpretation relies on target logical-shift behavior that supplies zero for the relevant out-of-range right and left shifts. The instruction-level behavior is not independently available here.
- **Byte order and alignment:** Not independently established.
- **Default bits:** The values and semantic meaning of the two external default words are unavailable.
- **Dependency contracts:** Mutation, memory management, retention, failure modes, and internal work are unavailable.
- **Concurrency:** No atomic snapshot, lock, or consistent pairing of separately read words is established.
- **Return ABI:** The supplied body establishes 64 returned bits, not a higher-level floating-point interface.

## Evidence

This contract is based solely on static inspection of the supplied function body and the supplied opaque dependency mapping.

The main evidence is the ordered flag tests; fixed-offset word reads; the narrow flags write; sign extraction and two-word negation; leading-zero normalization; explicit rounding and carry propagation; short-circuited dependency calls; and distinct initialization versus fallback reads of the default words.

No runtime tests have been performed. No dependency implementation, external data contents, target instruction listing, or independent ABI documentation was supplied.
