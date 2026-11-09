<!-- Trusted provenance: F-0ec3a2a1b51e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ec3a2a1b51e

## Purpose

F-0ec3a2a1b51e produces a 64-bit result from a primary 32-bit input, a secondary opaque 32-bit input, and, on some paths, ambient machine state and externally stored words.

**Established static behavior:** Most primary-input patterns are transformed directly into a 64-bit result without consulting external data. Patterns with an all-ones exponent field and a nonzero fraction instead select an indirect transfer using a control word, additional data words, and sometimes incoming processor state.

**Inferred intent:** The direct transformations match conversion of IEEE 754 binary32 encodings into binary64 encodings, except that nonzero binary32 subnormals are flushed to positive zero. The indirect paths appear to provide special handling for NaN encodings. The meaning of their modes and transferred arguments is not established.

This contract distinguishes the visible transformations from the unresolved indirect-transfer behavior. No runtime tests have been performed.

## Interface

### Explicit inputs

1. **Primary input:** An unsigned 32-bit word. All 32 bits participate in classification or result construction.
2. **Secondary input:** An opaque 32-bit word. Direct-result paths ignore it. Some indirect paths forward it unchanged; one path replaces it with an externally read word.

For describing the primary input, the following bit fields are useful:

| Field | Bits | Width |
|---|---:|---:|
| Sign | 31 | 1 bit |
| Exponent | 30–23 | 8 bits |
| Fraction | 22–0 | 23 bits |

These names describe an interpretation of the bits, not a requirement that the caller pass a language-level floating-point value.

### Ambient inputs

The indirect paths additionally depend on:

- A **control word**, treated by the visible expressions as a 32-bit bit field.
- A **default argument word**.
- An **alternate first argument word**.
- An **alternate second argument word**.
- An **incoming link-address word**, whose low two bits are cleared to obtain the indirect-transfer address.
- The low bit of an **incoming processor-state quantity**, used on one path.

The underlying storage declarations, volatility, synchronization guarantees, and precise machine calling convention are not established.

### Output

The represented output is a 64-bit unsigned bit pattern.

For direct results, this contract specifies the upper and lower 32-bit words numerically. This does not specify memory byte order or physical return-register placement.

For indirect paths, the visible representation returns the indirect operation’s 64-bit result unchanged, if that operation returns normally. Whether every represented indirect operation is an ordinary returning call is unresolved.

### Preconditions and validity boundaries

There is no visible rejection of any primary or secondary 32-bit pattern.

Direct-result paths require no visible external-memory access. Indirect paths require readable external state and an executable transfer destination under the actual machine ABI. No visible validation checks these conditions.

No explicit input is established to be a pointer, allocation handle, or owned resource.

## Behavior

### 1. Classify using 32-bit arithmetic

Classification first adds `0x00800000` to the primary input with 32-bit wraparound and examines mask `0x7F000000`.

If any masked bit is nonzero, the function takes the normal-value transformation described below. Equivalently, this selects primary inputs whose exponent field is between 1 and 254 inclusive.

For the remaining inputs, the function tests bit `0x00800000` of that same wrapped sum:

- A set bit selects the zero/subnormal family: exponent field 0.
- A clear bit selects the infinity/NaN family: exponent field 255.

The wraparound matters. In particular, exponent 255 must not be classified using an unbounded mathematical sum.

### 2. Transform normal encodings

For exponent fields 1 through 254:

- The lower result word is the primary input shifted left by 29 bits, retaining only the low 32 bits.
- The upper result word starts from an arithmetic right shift of the signed interpretation of the primary input by 3 bits.
- For a clear sign bit, add `0x38000000` to that shifted value.
- For a set sign bit, subtract `0x38000000` from that shifted value.

The upper and lower words are concatenated into the 64-bit result.

Under the floating-point interpretation:

- The sign is preserved.
- The binary64 exponent is the binary32 exponent plus 896.
- The 23 fraction bits are placed in the high 23 positions of the 52-bit binary64 fraction.
- The remaining 29 fraction positions are zero.

Consequently, this path represents every finite normal binary32 value exactly in binary64. It does not consult a rounding mode or perform visible floating-point arithmetic.

### 3. Handle exponent-zero encodings

For exponent field 0, the function examines all primary-input bits except the sign bit.

- If any of those bits are nonzero, the result is exactly `0x0000000000000000`.
- Otherwise, the primary input becomes the upper result word and the lower result word is zero.

Thus:

- Positive zero remains positive zero.
- Negative zero remains negative zero.
- Every nonzero subnormal becomes positive zero, including negative subnormals.

The sign loss for negative nonzero subnormals is part of the established transformation. It must not be replaced with signed-zero preservation or exact subnormal widening.

### 4. Handle infinity encodings

For exponent field 255 and fraction field zero, the result is formed by placing the primary input in the upper 32 bits and OR-ing the 64-bit value with `0x0070000000000000`.

This yields:

- `0x7F800000` → `0x7FF0000000000000`.
- `0xFF800000` → `0xFFF0000000000000`.

No control-word read or indirect transfer appears on this path.

### 5. Dispatch nonzero-fraction, exponent-255 encodings

For exponent field 255 and a nonzero fraction, the function reads the control word and retains its low four bits as the dispatch selector.

After entering this family, the visible logic does not distinguish primary-input sign, payload, or quiet/signaling encoding. Those bits are not explicitly forwarded to the indirect operation.

All paths below use the same opaque dynamic dependency, **F-000000000001**. This ID names the unresolved indirect operation, not a statically identified implementation. Its destination is the incoming link-address word with its low two bits cleared.

The argument counts below describe only the arguments explicitly represented in the supplied static material. They do not establish that other machine registers are cleared or unobservable.

#### Selector 10

The visible access order is:

1. Read the default argument word.
2. Read the control word to test bit `0x00000040`.

If that control bit is set, the first argument is `0x80000000`. Otherwise, the first argument is the previously read default argument word.

Transfer to F-000000000001 with that one explicit argument. Return its represented 64-bit result unchanged.

The default argument read precedes the overriding control-bit test in the supplied representation, even when the override is selected.

#### Selector 8

First, read the control word to test bit `0x00000040`.

If the bit is set:

- Transfer with two explicit arguments, both zero.

If the bit is clear:

1. Read the alternate first argument word.
2. Read the control word to test bit `0x00000010`.
3. If this second bit is clear, transfer with:
   - First argument: the previously read alternate first argument word.
   - Second argument: the original secondary input.
4. If this second bit is set:
   - Replace the first argument with zero.
   - Read the alternate second argument word.
   - Transfer with zero and that alternate second word.

The alternate first argument read precedes the test that may discard its value.

Return the indirect operation’s represented result unchanged.

#### Selector 9

Read the control word to test bit `0x00100000`.

If that bit is clear:

- Transfer with one explicit argument, zero.

If that bit is set, read the control word again and test mask `0x00070000`.

If all three masked bits are clear, transfer with three explicit arguments:

1. `0x30000000`.
2. The original secondary input.
3. `0x30000000`, with bit 27 additionally set if the low bit of the incoming processor-state quantity is one.

The third argument is therefore either `0x30000000` or `0x38000000`.

If any bit in mask `0x00070000` is set:

- Read the control word again.
- Transfer with first argument `8`.
- Construct the second argument using 32-bit operations: shift that control value left by 15, complement all 32 bits, then shift left by 1, discarding overflow.

For that second argument, bits 31–16 are the complement of the sampled control word’s bits 15–0. Its low 16 bits are always `0xFFFE`. Original control bits above bit 15 do not survive this argument construction.

Return the indirect operation’s represented result unchanged.

#### All other selectors

Selectors 0–7 and 11–15 transfer to F-000000000001 with no explicit arguments shown.

“No explicit arguments” does not mean that argument registers are zero or that the destination cannot observe ambient state.

Return the indirect operation’s represented result unchanged.

## Side effects

### Visible reads

The normal, zero/subnormal, and infinity paths contain no visible external-data reads.

The nonzero-fraction, exponent-255 paths read the control word and, conditionally, the three argument-data words described above.

The ordering in this contract follows the supplied static representation. It establishes neither physical bus transactions nor volatile-load behavior. Whether a compiler or processor combines, reorders, or elides accesses cannot be determined here.

Repeated control-word references must not automatically be treated as one coherent snapshot. If the underlying state can change between accesses, the representation allows the selector and subsequent decisions to reflect different sampled values. No synchronization or stability guarantee is shown.

### Visible writes and state changes

No direct external-memory write, allocation, deallocation, reference-count operation, or ownership transfer is shown.

This absence does not constrain F-000000000001. That dependency may read or modify globals, alter processor state, manage memory, fail, or not return; none of those behaviors is specified by the supplied material.

No explicit diagnostic, error code, exception construction, or recovery path appears in F-0ec3a2a1b51e itself.

### Ownership

The two explicit inputs are passed by value. There is no established ownership transfer associated with them.

Externally read words are used as argument bit patterns. Their interpretations at the indirect destination are unknown. They must not be labeled as owned pointers, borrowed pointers, or release obligations without additional evidence.

## Dependencies

| Opaque function ID | Role | Established contract |
|---|---|---|
| F-000000000001 | Dynamic indirect operation reached only for exponent 255 with nonzero fraction | Destination is derived by clearing the low two bits of the incoming link-address word. The selected branch supplies the explicit arguments described above. Its represented 64-bit return value is propagated unchanged. |

F-000000000001 is a local opaque dependency designation because no static identity was supplied for the dynamic target. Different incoming link-address values may identify different destinations.

Its calling convention, valid destinations, argument meanings, memory effects, error behavior, and return guarantees are **documented_only**.

The control word and three argument-data words are data dependencies, not function dependencies. Their initialization, storage lifetime, access permissions, and mutation policy are **documented_only**.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed test results.

### Direct transformations

| Primary input | Required 64-bit result | Observable property |
|---|---|---|
| `0x00000000` | `0x0000000000000000` | Positive zero preserved |
| `0x80000000` | `0x8000000000000000` | Negative zero preserved |
| `0x00000001` | `0x0000000000000000` | Smallest positive subnormal flushed |
| `0x807FFFFF` | `0x0000000000000000` | Negative subnormal loses its sign |
| `0x00800000` | `0x3810000000000000` | Smallest positive normal widened |
| `0x3F800000` | `0x3FF0000000000000` | Positive one widened |
| `0xBF800000` | `0xBFF0000000000000` | Negative one widened |
| `0x3F800001` | `0x3FF0000020000000` | Lowest input fraction bit reaches result bit 29 |
| `0x7F7FFFFF` | `0x47EFFFFFE0000000` | Largest finite positive input remains finite |
| `0x7F800000` | `0x7FF0000000000000` | Positive infinity preserved |
| `0xFF800000` | `0xFFF0000000000000` | Negative infinity preserved |

For each row, changing the secondary input must not change the result. No indirect dependency is reached.

### Conditional dispatch observations

For primary input `0x7FC00001`:

- With a stable control word of `0x0000004A`, selector 10 applies. The default argument word is read, but the explicit transferred argument is `0x80000000`.
- With a stable control word of `0x00000008`, selector 8 transfers the alternate first argument word and the original secondary input.
- With a stable control word of `0x00000018`, selector 8 reads the alternate first argument word, discards it, then transfers zero and the alternate second argument word.
- With a stable control word of `0x00000048`, selector 8 transfers two zero arguments without a visible read of either alternate argument word.
- With a stable control word of `0x00000009`, selector 9 transfers one zero argument.
- With a stable control word of `0x00100009`, selector 9 transfers `0x30000000`, the secondary input, and either `0x30000000` or `0x38000000`, according to the incoming processor-state low bit.
- With a stable control word of `0x00110009`, selector 9 transfers `8` and `0xFFF6FFFE`.
- With selector 0, the transfer has no explicit arguments shown.

For every such case, if the dynamic operation returns a particular 64-bit value normally, that same value is the represented result of F-0ec3a2a1b51e.

These examples require readable external state and a valid indirect destination. They do not establish successful execution when either prerequisite is absent.

## Uncertainties

The following remain **documented_only**:

- Whether the indirect transfers are ordinary calls, continuation transfers, exception machinery, or another ABI mechanism.
- The significance of the selector values, control bits, and constructed arguments.
- The origin and validity rules for the incoming link-address word.
- The precise architectural meaning and availability of the incoming processor-state quantity.
- The actual storage widths and access qualifiers of externally stored data.
- Whether repeated control-word accesses observe stable state.
- The physical calling convention, return-register layout, and memory endianness.
- Any indirect dependency side effects, ownership behavior, errors, or non-returning behavior.
- Whether the apparent floating-point policy is intentional, particularly positive-zero flushing of negative subnormals.

No requirement to preserve global state, preserve processor flags, release memory, or normalize a NaN payload can be inferred for the unknown dependency.

## Evidence

**Established static evidence:** The supplied material exposes two explicit 32-bit inputs, a represented 64-bit return, fixed-width masks and shifts, direct-result branches, conditional external-data references, and unresolved indirect transfers. The classification, direct result patterns, argument selections, and represented read order above follow those operations.

**Inferred intent:** Binary32-to-binary64 conversion and special-value handling are interpretations supported by the bit layouts and constants. They are not independent evidence of an ABI, runtime exception policy, or full floating-point conformance.

**Verification status:** No implementation access, disassembly validation, runtime execution, or tests were performed. Acceptance examples are static consequences and conditional expectations, not observed outputs.
