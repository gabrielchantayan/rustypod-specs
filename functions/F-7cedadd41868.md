<!-- Trusted provenance: F-7cedadd41868; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7cedadd41868

## Purpose

F-7cedadd41868 performs an ordered sequence of unchecked memory operations and nine dependency calls, then enters an unconditional infinite loop. The supplied representation contains **30 explicit four-byte stores**.

No application-level purpose is established. Initialization, object construction, device control, and failure handling are possible interpretations, not supported contract claims.

The terminal loop is reached only if all preceding operations complete and all dependencies return normally. A fault or a non-returning dependency can stop progress earlier.

## Interface

### Explicit inputs

| Position | Contract name | Representation and use |
|---|---|---|
| 1 | First offset | Signed 32-bit integer used in the first destination calculation |
| 2 | Value/base | Signed 32-bit integer used as stored data and as an address component |
| 3 | Unused input | Four-byte value with no explicit use in the body |
| 4 | Selector | Unsigned 32-bit integer used as data, an address component, and a call argument |

The unused input has no explicit dataflow into the described operations. Whether a dependency can observe incidental entry-state copies is unknown.

### Ambient inputs

The explicit arguments are insufficient to establish a valid invocation. Additional entry state supplies:

- A four-byte payload for the first store.
- Two signed 32-bit address bases.
- A retained pointer, called the **anchor**, with a four-byte representation.
- A signed 32-bit incoming link value used to derive the first call target.
- A four-byte stack-associated value at offset `0x168`, supplied to F-c512cbdbd236.

The origins, validity, and calling-convention requirements of these values are `documented_only`.

### Results and termination

There is no explicit return value and no normal return path. After the final store, the body enters a loop containing no further explicit memory operations or dependency calls.

Fault handling, exceptional exits, external termination, and dependency non-return behavior are not established.

### Widths and contract notation

These names describe contract values, not implementation variables:

| Name | Meaning |
|---|---|
| D | Unsigned 32-bit selector |
| Q | D logically shifted right by two bits |
| W | Eight-byte result of the second call |
| X | Result of converting W to signed 32 bits |
| Y | Result of converting the unsigned upper 32-bit portion of W to signed 32 bits |
| T | Unsigned 32-bit result of the fourth call |
| U | T logically shifted right by two bits |
| V | T logically shifted right by four bits |
| K | Four-byte result of the sixth call |
| R | Signed 32-bit result of the eighth call |
| S | Signed 32-bit result of the ninth call |
| A | Signed 32-bit conversion of the anchor pointer, used in integer address expressions |

All listed stores occupy four bytes, including pointer-valued stores. Address formulas denote byte addresses, without implicit element-size scaling.

Unsigned 32-bit addition and multiplication wrap modulo \(2^{32}\). Logical right shifts introduce zero bits. Combining a signed 32-bit operand with D uses its unsigned 32-bit conversion for the displayed mixed arithmetic.

The following require separate qualifications:

- **Signed narrowing:** X is a conversion result, not an unconditional promise of W’s low 32-bit pattern. Y is likewise a conversion result, not an unconditional promise that the extracted upper portion retains its bit pattern.
- **Pointer conversion:** A is the displayed pointer-to-signed-integer conversion. Its correspondence to the anchor’s address bits requires an established execution model.
- **Signed arithmetic:** The first destination addition, first call-target addition, X minus `0x16`, R plus `0x18`, and S plus `0x3c` are signed expressions. Portable wrapping behavior is not established.
- **Signed shift:** The ninth call uses R shifted left by two bits. Negative inputs or nonrepresentable results have no established portable interpretation here.
- **Final global expression:** The referenced global’s declaration is absent. Its type, read width, and destination-base addition type are unknown.

Under an additional conventional low-32-bit conversion model, X has W’s low 32-bit pattern and Y has its upper 32-bit pattern. Neither claim is unconditional in this contract.

## Behavior

The following sequence describes explicit operations in the supplied representation. Dependency-internal effects are additional and unknown. Repeated stores remain distinct events.

### Initial stores and first two calls

Five stores precede the first call:

| Order | Destination | Four-byte data |
|---|---|---|
| A1 | First ambient base + first offset | Ambient payload |
| A2 | A + D | Value/base |
| A3 | 2 × D | Value/base |
| A4 | Second ambient base + D | Anchor pointer representation |
| A5 | Value/base + D | Q |

A1 uses signed addition. A2, A4, and A5 use unsigned 32-bit arithmetic after the applicable conversions. No validation, alignment adjustment, or bounds checking precedes these accesses.

The first dependency, **F-I01**, receives D. Its target is derived from the incoming link value plus `0xff8`; that displayed addition is signed. No result is explicitly consumed.

The second dependency, **F-I02**, has no explicit arguments. Its eight-byte result supplies W.

“No explicit arguments” does not mean that a dependency cannot observe other machine state.

### Stores derived from the second call

After F-I02 returns:

| Order | Destination | Four-byte data |
|---|---|---|
| B1 | D + Q | X |
| B2 | 2 × D | Signed result X − `0x16` |
| B3 | 2 × Q | Q |
| B4 | 2 × D | D |
| B5 | A + Q | Y |
| B6 | 2 × D | D |

B1 stores the signed 32-bit conversion of W. B5 stores the signed 32-bit conversion of the unsigned quantity obtained by shifting W’s unsigned 64-bit interpretation right by 32 bits. Neither store is specified unconditionally as a raw half of W.

B2 subtracts **after** conversion to X. The subtraction is representable when X is at least \(-2147483626\). Smaller values can overflow signed 32-bit arithmetic.

B4 and B6 are separate stores, with B5 between them. Aliasing or memory-mapped behavior may make their separation observable.

The third dependency, **F-I03**, receives D twice. No result is explicitly consumed.

The fourth dependency, **F-I04**, has no explicit arguments and supplies T.

### Stores derived from the fourth call

After F-I04 returns:

| Order | Destination | Four-byte data |
|---|---|---|
| C1 | 2 × D | D |
| C2 | Q + D | T |
| C3 | D + `0x40` | Q |
| C4 | 2 × D | D |
| C5 | Q + `0x4c` | Q |
| C6 | D + `0x4c` | Q |
| C7 | 2 × D | D |
| C8 | 4 × T + `0x14` | U |
| C9 | T + D | T |
| C10 | 4 × T + `0x14` | U |

These destination calculations use unsigned 32-bit arithmetic.

C2 has the same address expression as B1. Intervening operations can nevertheless affect the same bytes through aliases.

C8 and C10 are distinct stores separated by C9, even when C9 targets another location.

### Middle calls and anchor stores

The fifth dependency, **F-c512cbdbd236**, receives:

1. The four-byte value read through address D + `0x3c`.
2. The stack-associated four-byte value at offset `0x168`.

The first argument is read at this call stage, after C10 and all preceding calls. It is not an entry-time snapshot. The physical evaluation order of the two argument sources is not established.

After this call returns, the body stores the anchor pointer representation at **V + D**.

The sixth dependency, **F-I05**, receives T and U and supplies K.

After F-I05 returns, the body stores the anchor pointer representation at **U + D**.

The seventh dependency, **F-I06**, receives K and U. No result is explicitly consumed.

The eighth dependency, **F-I07**, receives U twice and supplies R.

No dependency is assumed to preserve memory affected by earlier stores.

### Final stores and ninth call

After F-I07 returns:

| Order | Destination | Four-byte data |
|---|---|---|
| D1 | A + D | D |
| D2 | R + `0x18` | R |
| D3 | V + D | Anchor pointer representation |
| D4 | V + `0x18` | V |

D2 uses signed addition. The other destination calculations use unsigned 32-bit arithmetic.

The ninth dependency, **F-I08**, receives, in order:

1. V.
2. R shifted left by two bits using a signed expression.
3. U.
4. V.

For nonnegative R no greater than `0x1fffffff`, the shifted result is representable as signed 32 bits. Outside that range, this contract does not promise the low 32 bits of four times R.

F-I08 supplies S. After it returns:

| Order | Destination | Four-byte data |
|---|---|---|
| E1 | Anchor, used directly as a pointer | U |
| E2 | S + `0x3c` | V |
| E3 | Final global-derived destination | Final global-derived value |

E2 uses signed addition before conversion to the store destination.

### Final global-derived operation

E3 references a global associated with absolute address `0x0000003c`. Its declaration is not supplied.

Two source-level uses are established:

- One supplies the base operand of a destination expression involving addition of `0x18`.
- The other supplies the value assigned through a signed 32-bit destination.

The **destination store is four bytes**. That does not establish the global’s own type, signedness, or read width. Nor does it establish that the base addition is signed 32-bit arithmetic.

E3 evaluates after E1 and E2. Earlier stores and dependency effects can therefore influence the referenced global before this stage; no entry-time snapshot is present.

The expression does not explicitly retain one global value for both uses. It also does not prove two physical loads. Physical load count, evaluation order, conversion details, and whether the uses observe the same value remain unknown.

After E3 completes, the terminal loop begins.

## Side effects

### Memory and ordering

The body contains 30 explicit four-byte stores:

- Five initial stores.
- Six stores after the second call.
- Ten stores after the fourth call.
- Two anchor stores interleaved with middle calls.
- Four stores before the ninth call.
- Three stores after the ninth call.

Read-side evidence establishes:

- A four-byte indirect read supplying F-c512cbdbd236’s first argument.
- A four-byte stack-associated argument value.
- Two source-level uses of the final global, with unspecified type and read width.

These facts do not determine a total physical load count. Likewise, the explicit store sequence does not independently establish compiler, processor, volatile-access, or device-ordering guarantees.

Aliasing and partial overlap are not excluded. A store can affect a later argument read, another store’s bytes, or the final global-derived operation. No final-memory map can replace the ordered operations and intervening calls.

### Preconditions and errors

The body performs no explicit checks for:

- Null, unmapped, low, read-only, or unsuitable addresses.
- Alignment, bounds, or lifetime.
- Signed arithmetic overflow.
- Invalid pointer-to-integer conversions.
- Dependency results unsuitable for address use.
- Overlapping destinations.

Completing the sequence requires supported accesses and conversions, supported semantics for the executed arithmetic, callable targets, and normal return from every dependency.

There is no explicit error result, recovery branch, retry, rollback, or cleanup. Exact fault and exception behavior is `documented_only`.

### Ownership and retained state

The body contains no explicit allocation or release. Storing anchor pointer bits does not establish ownership transfer, retention, or lifetime extension.

Writing through the anchor changes the addressed memory; it does not explicitly replace the retained pointer value.

Dependencies may allocate, release, retain, invalidate, or modify resources. Their contracts are unavailable. The body does not restore modified memory before entering its loop.

## Dependencies

Opaque IDs identify call sites. Distinct IDs do not establish distinct implementations.

| Call | Opaque ID | Explicit arguments | Consumed result |
|---|---|---|---|
| 1 | F-I01 | D | None |
| 2 | F-I02 | None | Eight-byte W |
| 3 | F-I03 | D, D | None |
| 4 | F-I04 | None | Unsigned 32-bit T |
| 5 | F-c512cbdbd236 | Four-byte value through D + `0x3c`; stack-associated four-byte value | None |
| 6 | F-I05 | T, U | Four-byte K |
| 7 | F-I06 | K, U | None |
| 8 | F-I07 | U, U | Signed 32-bit R |
| 9 | F-I08 | V, signed shifted R, U, V | Signed 32-bit S |

F-I01’s target depends on incoming link state. The other unresolved indirect targets are derived from embedded code-location values and fixed offsets.

No dependency is assumed to preserve globals, memory, ownership, or resource validity.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless stated otherwise, relevant preceding operations complete, dependencies return, and unrelated destinations do not alias the locations discussed.

### Selector shifts do not align addresses

For D = `0x103`, Q = `0x40`.

A5 carries `0x40`. B1 targets `0x143`, and B3 targets `0x80`. No implicit alignment adjustment is applied.

### Repeated stores are separate events

For D = `0x100` and W representing `0x50`, X is `0x50`.

Through B6, the stores targeting `0x200` carry, in order:

1. Value/base.
2. `0x3a`.
3. `0x100`.
4. `0x100`.

Observing only the final value would omit required explicit store events.

### Both half-result stores require conversion qualifications

Suppose W has upper portion `0x80000000` and lower portion `0xffffffff`.

Without an established signed-conversion model, this contract does not require B1 to store `0xffffffff` or B5 to store `0x80000000`.

Under an additional model that converts by retaining the low 32 bits and interpreting them as signed two’s-complement values:

- X is −1, so B1 stores pattern `0xffffffff`.
- Y is \(-2147483648\), so B5 stores pattern `0x80000000`.
- B2 stores −23, pattern `0xffffffe9`.

The bit-pattern conclusions depend on that model for **both** B1 and B5.

### Signed subtraction is not automatically wrapping

If conversion yields X = \(-2147483648\), subtracting `0x16` exceeds the signed 32-bit range.

B2 is not unconditionally required to store `0x7fffffea`. That pattern additionally requires wrapping signed subtraction and an appropriate representation model.

### Result-derived scales remain distinct

For T = `0x80`, U = `0x20` and V = `0x08`.

C8 and C10 each store `0x20` at `0x214`, with C9 between them. Subsequent anchor stores target D + `0x08`, D + `0x20`, and again D + `0x08`.

For T = `0x40000000`, unsigned arithmetic instead makes the C8 and C10 destination `0x14`, with stored value `0x10000000`. This does not establish access validity.

### Final-global examples require an explicit type model

Assume, additionally, that the final global is read as signed 32 bits, the base addition is signed 32-bit arithmetic, and both uses supply `0x1000`. E3 then stores `0x1000` at `0x1018`.

Under the same type assumptions, if the execution model permits different observations and the base use supplies `0x1000` while the data use supplies `0x2000`, E3 stores `0x2000` at `0x1018`.

Neither example establishes the missing declaration, separate physical loads, or an evaluation order. Both concern evaluation after E1 and E2.

### Dependency non-return truncates the sequence

If F-I07 does not return, D1 and every later operation are not reached. Non-return need not mean execution reached the body’s own terminal loop.

## Uncertainties

The following remain `documented_only`:

- Application-level purpose and intended valid-input domain.
- Complete calling convention and origins of ambient values.
- Concrete dependency identities, shared targets, effects, and ownership contracts.
- Pointer-to-integer conversion semantics.
- Out-of-range signed narrowing behavior for X and Y.
- Signed overflow and out-of-range signed-shift behavior.
- Endianness, alignment requirements, memory mappings, and fault handling.
- The final global’s declaration, type, signedness, read width, and assignment conversion.
- The arithmetic type and overflow behavior of the final destination-base addition.
- Physical access counts, evaluation details, and ordering guarantees beyond the displayed operations.
- Concurrency, interrupt, and memory-mapped-device semantics.
- Whether the terminal loop represents a deliberate halt or another intended state.

Four-byte pointer-related operations support a 32-bit address interpretation in this representation. They do not establish a complete architecture or ABI contract.

## Evidence

**Established static evidence:** four explicit input positions; additional ambient values; 30 explicit four-byte stores; nine calls; an eight-byte result converted to signed 32 bits for B1 and B2; unsigned upper-portion extraction followed by signed conversion for B5; unsigned shifts and address arithmetic; several signed expressions; unchecked result-derived destinations; two source-level uses of an undeclared global in the final expression; and a terminal infinite loop.

**Inferred intent:** none is required. Low-bit signed conversion, signed wrapping, and a signed four-byte declaration for the final global are additional models, not established facts.

**Verification status:** static analysis only. No runtime tests, execution traces, dependency implementations, or independent ABI documentation were used. Acceptance examples specify conditional outcomes, not observed executions.
