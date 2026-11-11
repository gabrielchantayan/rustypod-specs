<!-- Trusted provenance: F-708b5aaef3f8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-708b5aaef3f8

## Purpose

F-708b5aaef3f8 conditionally reads a signed 16-bit field from an object-associated record, passes its value through three dependencies, and constructs a 32-bit result from the last dependency’s 64-bit result.

The ordinary result construction is consistent with reducing an IEEE 754 binary64 representation to binary32, with round-to-nearest, ties-to-even for values admitted to the retained normal range. It does not produce binary32 subnormals: sufficiently small nonzero values return positive zero, including negative inputs. This interpretation does not establish what arithmetic the dependencies perform.

A zero result from an initial object-dispatched dependency terminates processing before record access. An all-ones exponent with a nonzero fraction instead selects control-word-dependent indirect dispatch. That dispatch’s complete control-flow and calling-convention semantics remain unresolved.

## Interface

### Explicit input and object layout

The explicit input is an object address. The visible representation uses 32-bit addresses and 32-bit words.

| Location | Interpretation |
|---|---|
| Object offset 0 | Address of a dispatch table |
| Dispatch-table offset 8 | Callable target for the initial dependency |
| Object offset 8 | Address of an associated record |
| Associated-record offset 124 | Signed 16-bit field |

The initial dependency receives the original object address. Its 32-bit result is tested only for equality with zero; no Boolean, validity, or error interpretation is established.

If processing continues, the record field is sign-extended to 32 bits. Its numerical range is −32768 through 32767.

### Output

The function returns one 32-bit word.

Ordinary paths construct binary32-compatible encodings. Exceptional paths forward an indirect dependency’s returned word. The evidence does not establish that the external interface declares a floating-point return type.

### Implicit inputs

The function also depends on:

- Two global 32-bit data words supplied to F-596e18a33aec.
- Two further global 32-bit data words supplied to F-1b22d7e17791.
- A shared 32-bit control word used for exceptional dispatch.
- Three exceptional-dispatch global data words, described below as the **default first word**, **common second word**, and **alternative second word**.
- An inherited link-register value used to form an indirect destination.
- Bit 0 of an inherited processor-state quantity.

The global words’ values and immutability are unknown. They are not established as constants or entry-time snapshots. Unknown dependencies may modify them before their reads occur.

### Preconditions and ownership

Every address and target reached by execution must support the described access or transfer. No explicit null, bounds, alignment, or callable-target validation is visible.

On the early-return path, the associated-record pointer and record need not be usable because neither is read. The object’s dispatch-table access is still required.

The visible body neither allocates nor releases memory and establishes no ownership transfer. This does not establish ownership behavior for any dependency.

## Behavior

### Initial dispatch and field acquisition

The function reads the object’s dispatch-table address and invokes the target at table offset 8 with the original object address.

If the target returns zero, the function returns `0x00000000`. This path performs no visible associated-record pointer read, signed-field read, conversion-chain call, or exceptional-control-word read.

If the result is nonzero, the function reads the associated-record pointer from object offset 8, then reads the signed 16-bit field at record offset 124.

These record-related reads occur after the initial call returns. Mutations made by that dependency can therefore affect which record and field value are used. There is no visible pre-call snapshot of either.

### Dependency chain and global-data timing

The three dependencies are invoked in this order:

1. **F-b3a54d881220** receives the sign-extended field as one 32-bit argument and supplies a 64-bit result.
2. **F-596e18a33aec** receives four 32-bit argument words: the preceding result’s low word, its high word, and the first stage’s two global data words, in their designated argument positions.
3. **F-1b22d7e17791** receives the next result’s low word, its high word, and the final stage’s two global data words, again in their designated positions.

The first stage’s global pair is read **after F-b3a54d881220 returns** and before F-596e18a33aec is invoked. The final stage’s pair is read **after F-596e18a33aec returns** and before F-1b22d7e17791 is invoked.

Consequently:

- F-b3a54d881220 may affect the global values supplied to either later dependency.
- F-596e18a33aec may affect the global values supplied to F-1b22d7e17791.
- Neither pair is established as an immutable constant or a value captured at function entry.

The argument positions are established. A relative read order between the two words within either argument pair is not established by the supplied call expressions.

All subsequent local result construction uses the 64-bit result of F-1b22d7e17791. The dependencies’ numerical operations are unknown.

### Representation and arithmetic widths

For the final 64-bit result:

- The low word contains bits 0–31.
- The high word contains bits 32–63.
- The sign is high-word bit 31.
- The exponent-like field is high-word bits 20–30.
- The fraction-like field consists of high-word bits 0–19 together with the entire low word.

These positions match binary64 fields, but the bit-level behavior below does not require the result to have a declared floating-point type.

Unsigned 32-bit additions and left shifts retain only the low 32 bits. Right shifts of the unsigned words are logical. The lower- and upper-bound comparisons described below use signed 32-bit interpretations; replacing them with unsigned comparisons would change behavior.

### Finite representations with nonzero exponent

This path applies when the exponent-like field is neither zero nor 2047.

The function clears the high word’s sign bit and adds `0xC8000000`, retaining the low 32 bits. This produces the adjusted high word.

The adjusted high word is first interpreted as signed and compared with `0x00100000`. If it is smaller, the result is positive zero. The sign is discarded, and no rounding occurs.

Under a binary64 interpretation, every finite value with unbiased exponent below −126 takes this early zero path. Even a value immediately below the smallest binary32 normal value cannot round upward into that range.

For an adjusted high word that passes the lower bound, the original low word determines rounding:

- Bit 28 is the guard bit.
- Bits 0–27 indicate whether any lower discarded bit is nonzero.
- Bit 29 is the least significant retained bit.

An increment occurs exactly when the guard bit is set and either a lower discarded bit is nonzero or bit 29 is set. An exact halfway case therefore increments only when the retained least significant bit is odd.

The increment adds `0x20000000` to the low word modulo \(2^{32}\). If the original low word exceeds `0xDFFFFFFF`, the adjusted high word is also incremented by one.

After rounding, the adjusted high word is compared, as signed, with `0x0FEFFFFF`:

- If greater, the result is the original sign combined with `0x7F800000`.
- Otherwise, the result combines the original sign, the adjusted high word shifted left three bits with 32-bit truncation, and the rounded low word shifted right 29 bits.

The overflow decision follows rounding. A rounding carry can therefore change a retained finite result into signed infinity.

### Zero-exponent representations

When the exponent-like field is zero:

- If the fraction-like field is zero, the function returns the original sign bit alone.
- If the fraction-like field is nonzero, the function returns positive zero.

Thus positive and negative zero remain distinct. Under a binary64 interpretation, every nonzero binary64 subnormal is flushed to positive zero, including negative subnormals.

### All-ones-exponent representations

When the exponent-like field is 2047 and the fraction-like field is zero, the function returns the high word masked with `0xFF8FFFFF`. The two possible inputs produce `0x7F800000` and `0xFF800000`, respectively.

When the exponent-like field is 2047 and the fraction-like field is nonzero, execution enters exceptional dispatch. No local rule establishes NaN payload propagation, NaN sign preservation, or quiet/signaling conversion.

### Exceptional dispatch

The first control-word read selects a mode from its low four bits. Later flag decisions use additional reads, not an established immutable snapshot.

All described indirect destinations are formed by clearing the inherited link-register value’s low two bits. The supplied representation shows these transfers as calls whose returned 32-bit words are forwarded. A conventional callable target and normal return behavior are not independently established.

#### Mode 10

After mode selection, another control-word read tests bit 6.

- If clear, the transfer receives one explicit word: zero.
- If set, it receives one explicit word: `0x80000000`.

The represented transfer result becomes the function result.

#### Mode 8

After mode selection, another control-word read tests bit 6.

If bit 6 is set, the transfer receives two zero words. This branch does not visibly read the three exceptional-dispatch data words.

If bit 6 is clear, the visible sequence is:

1. Read the global default first word.
2. Read the global common second word.
3. Reread the control word and test bit 4.
4. If bit 4 is clear, transfer using the two words already read.
5. If bit 4 is set, read the global alternative second word, then transfer using zero as the first word and that newly read value as the second word.

The default-first and common-second reads occur even when bit 4 selects the alternative arguments. Their read occurrence and ordering remain part of the behavior although their values are then discarded.

The represented transfer result is returned unchanged.

#### Mode 9

After mode selection, another control-word read tests bit 20.

If bit 20 is clear, the function reads the common second word and transfers with two explicit arguments: zero and that read value.

If bit 20 is set, another control-word read tests bits 16–18.

- If all three bits are zero, the function reads the common second word and transfers with three explicit words: `0x30000000`, the common second value, and a processor-state-derived word. The third word is `0x30000000` when inherited processor-state bit 0 is zero, or `0x38000000` when it is one.
- If any of bits 16–18 is set, another control-word read supplies the value used to construct the second argument. The first argument is 8. The second is formed by shifting that newly read 32-bit value left 15 bits, complementing the 32-bit result, and shifting left once more, with 32-bit truncation.

The represented transfer result is returned unchanged.

#### Other modes

The representation supplies no explicit arguments to the indirect transfer and forwards its returned word. This does not establish absence of dependence on inherited registers or other machine state.

## Side effects

No explicit stores to the object, record, control word, or global data words appear in the visible body.

The function nevertheless cannot be classified as side-effect-free:

- The initial object-dispatched dependency may modify the object, record, globals, or external state.
- Each of the three named dependencies may modify globals, memory, ownership, or external state.
- Exceptional indirect destinations have unknown effects.
- The complete processor-status consequences are not established.

The visible read and call ordering includes:

1. Initial object dispatch before associated-record access.
2. Signed-field acquisition before F-b3a54d881220.
3. First-stage global argument reads after F-b3a54d881220 and before F-596e18a33aec.
4. Final-stage global argument reads after F-596e18a33aec and before F-1b22d7e17791.
5. Exceptional control and data reads only after classification of the final 64-bit result.
6. In mode 8 with bit 6 clear, default-first read before common-second read before the bit-4 control reread, followed conditionally by the alternative-second read.

Repeated global reads do not establish atomicity, synchronization, volatility, or stable values.

No explicit error-reporting mechanism is visible. Zero is an ordinary result on several paths, not an established error code. Invalid memory or incompatible targets are not visibly converted into a local failure result.

## Dependencies

| Dependency | Established contract | Unresolved behavior |
|---|---|---|
| F-b3a54d881220 | Receives the sign-extended field as a 32-bit argument; supplies a 64-bit result | Arithmetic, global mutations, ownership, failures |
| F-596e18a33aec | Receives low/high result words followed by two global words read after the preceding call; supplies a 64-bit result | Operation, data meanings, side effects |
| F-1b22d7e17791 | Receives the next low/high words followed by two global words read after the preceding call; supplies the final 64-bit result | Operation, data meanings, side effects |
| Unresolved object-dispatched target | Receives the object address; zero suppresses further processing | Identity, complete signature, mutations, ownership |
| Unresolved exceptional destination | Receives mode-dependent explicit words; represented return word is forwarded | Identity, transfer semantics, calling convention, side effects |

No opaque IDs are supplied for the unresolved indirect targets. They are not equated with any named dependency.

## Acceptance examples

These are conditional consequences of the static representation, not runtime observations. Unless stated otherwise, the initial dependency returns nonzero and the dependency chain produces the specified final 64-bit pattern.

| Condition | Required result or behavior |
|---|---|
| Initial dependency returns zero | Return `0x00000000`; no associated-record or conversion-chain access |
| Signed field contains `0xFFFF` | F-b3a54d881220 receives −1, not 65535 |
| Final pattern `0x0000000000000000` | Return `0x00000000` |
| Final pattern `0x8000000000000000` | Return `0x80000000` |
| Final pattern `0x0000000000000001` | Return `0x00000000` |
| Final pattern `0x8000000000000001` | Return `0x00000000`, losing the sign |
| Final pattern `0x3FF0000000000000` | Return `0x3F800000` |
| Final pattern `0xBFF0000000000000` | Return `0xBF800000` |
| Final pattern `0x3810000000000000` | Return `0x00800000` |
| Final pattern `0x380FFFFFFFFFFFFF` | Return `0x00000000`; lower-bound rejection precedes rounding |
| Final pattern `0x3FF0000010000000` | Return `0x3F800000`; an even halfway case does not increment |
| Final pattern `0x3FF0000030000000` | Return `0x3F800002`; an odd halfway case increments |
| Final pattern `0x47EFFFFFF0000000` | Return `0x7F800000`; rounding causes overflow |
| Final pattern `0x7FF0000000000000` | Return `0x7F800000` |
| Final pattern `0xFFF0000000000000` | Return `0xFF800000` |

Additional conditional cases constrain timing and exceptional behavior:

- If F-b3a54d881220 changes a first-stage global word and no intervening change occurs, F-596e18a33aec receives the changed value, not an entry-time value.
- If F-596e18a33aec changes a final-stage global word and no intervening change occurs, F-1b22d7e17791 receives that changed value.
- For an exceptional representation, mode 10 with bit 6 clear transfers zero; with bit 6 set it transfers `0x80000000`.
- In mode 8 with bit 6 clear and the later bit-4 read set, both default-first and common-second reads still occur before that control reread. The alternative-second read follows it; the transfer receives zero and the alternative value.
- In mode 8 with bit 6 set, the transfer receives two zero words without those exceptional data reads.
- In mode 9 with bit 20 set and bits 16–18 clear, inherited processor-state bit 0 selects the third argument between `0x30000000` and `0x38000000`.

If the control word changes between reads, each decision uses its own observed value rather than the initial mode-selection value.

## Uncertainties

**documented_only — dependency semantics.** The numerical operations, failure behavior, side effects, and ownership behavior of the dependencies are unavailable. The full relationship between the signed input field and returned word cannot be established.

**documented_only — global data.** The global argument and exceptional-dispatch words have unknown values and mutability. Their observed read timing is established where described; no immutability, entry-time capture, or preservation across calls is assumed.

**documented_only — exceptional control flow.** The inherited destination, actual calling convention, normal return behavior, and dependence on other machine state remain unresolved.

**documented_only — execution environment.** Memory endianness, formal floating-point types, complete ABI details, synchronization, and processor-status effects are not established.

**Inferred intent.** Ordinary result construction strongly suggests binary64-to-binary32 reduction with ties-to-even rounding, signed infinities, and suppression of subnormal outputs. That interpretation does not replace the explicit positive-zero rules or resolve exceptional dispatch.

## Evidence

Established static evidence is the supplied function representation and opaque dependency mapping. It supports the access widths and offsets, sign extension, call order, post-call global argument reads, classification, masks, signed comparisons, rounding predicate, immediate results, and exceptional read sequences described here.

In particular, the mode-8 assignments establish default-first and common-second reads before the bit-4 control reread, including when their values are subsequently discarded. The conversion call expressions place each global argument pair after the preceding dependency has returned; they do not establish immutable constants.

Source comments and incomplete indirect-transfer annotations are not treated as authoritative explanations of target behavior.

No runtime tests have been performed. Acceptance examples are derived static consequences, not observed executions.
