<!-- Trusted provenance: F-72e2e88ac0bc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-72e2e88ac0bc

## Purpose

The supplied static representation establishes an ordered sequence of indirect calls, computed-address memory accesses, and bit-field transformations. It does not establish an application-level purpose such as initialization, serialization, allocation, or device control.

The observable contract is primarily an interaction sequence:

- Four explicit input slots are represented.
- Additional entry-machine state supplies the first stored word and the first indirect-call target.
- The displayed path contains 79 indirect call sites.
- Explicit data-memory accesses comprise **13 word stores, two byte stores, three word-read references, and two byte reads**.
- The displayed path ends at a reported invalid-instruction boundary, without an ordinary return.

These observations are conditional on the accuracy of the supplied representation and on execution reaching each operation. No runtime execution or testing has been performed.

## Interface

### Explicit inputs

| Specification label | Width and interpretation | Established use |
|---|---|---|
| P | Signed 32-bit integer | Address formation for the entry store; argument to the first call. |
| Second input | Opaque 32-bit quantity | No explicit reference in the displayed body. |
| Third input | Opaque 32-bit quantity | No explicit reference in the displayed body. |
| U | Unsigned 32-bit integer | Scaled addresses, stored values, extracted fields, and call arguments. |

The absence of explicit references to the second and third inputs does not establish that dependencies cannot observe their entry values through machine state. The actual calling convention is unknown.

### Implicit inputs

Two entry-state quantities are required to describe the first operations:

- An opaque 32-bit register value supplies the word stored at P + 20.
- A link-related machine value supplies the base for the first indirect-call target; the displayed target calculation adds 4094 bytes.

Neither quantity has an established application meaning.

### Outputs and termination

The recovered interface declares no return value, and the displayed path contains no ordinary return. Outputs consist of the memory effects and dependency interactions described below.

There is no established success result, status code, exception contract, or error object. At the terminal boundary, the supplied representation reports invalid instruction data. Whether actual execution traps, faults, transfers elsewhere, or differs from the recovered sequence remains `documented_only`.

### Widths and notation

This contract uses:

- **W(address)** for a 32-bit memory location.
- **B(address)** for an 8-bit memory location.
- **R10**, **R15**, and corresponding labels for results of numbered call sites.
- **Q** for the unsigned 32-bit value obtained by assigning the represented product 4 × R50.
- **H** for the unsigned word retained before call 46.

These are specification labels, not implementation identifiers. Offsets are in bytes.

Results R10, R15, R20, R25, R30, R35, R40, R45, R50, R55, R60, R72, and R77 are represented as signed 32-bit integers. R65 is unsigned 32-bit. The result of call 67 is an opaque 32-bit quantity.

Unsigned arithmetic and unsigned left shifts retain the low 32 bits. Unsigned right shifts introduce zero bits. Byte stores retain the low eight bits of their represented stored value.

Signed multiplication, addition, left shifts, and out-of-range conversion to signed values do not acquire an unconditional wrapping contract here. In particular:

- Assigning 4 × R50 to Q does not establish that the multiplication itself was unsigned.
- The selector address R50 × 0x10000000 + 1 involves represented signed arithmetic.
- The shifted arguments derived from R50, R72, and R77 involve signed left shifts.
- The unsigned result R65 shifted left by 2 is assigned to a signed 32-bit intermediate before its first two uses as call arguments.

Where these operations exceed their established signed range, exact behavior is `documented_only`. Address formulas below express the represented relationships, not a guarantee of valid or modulo-wrapped addresses.

## Behavior

### Ordering and dependency notation

F-00000001 through F-00000079 are opaque **call-site IDs**. They identify invocation positions, not proven distinct dependency implementations.

The displayed path contains no explicit branches or loops. Its ordering applies only while preceding accesses complete and preceding calls return normally. Calls described as having no explicit arguments have none shown in the supplied call expression; this does not establish that their implementations consume no machine-state inputs.

Results not subsequently referenced by the displayed body are omitted from the value-flow description. Their calls remain observable and cannot be removed on that basis.

### Entry and unsigned-input field

Before F-00000001, the implicit entry word is stored at W(P + 20). No null, range, alignment, or permission check precedes this store.

F-00000001 receives P. F-00000002 through F-00000005 follow in numerical order, with no explicit arguments.

After F-00000005:

- W(U × 0x10000000 + 20) receives U × 0x04000000.
- U AND 0x1fffffff is retained for later field extraction.
- F-00000006 receives U × 0x04000000.
- F-00000007 through F-00000009 have no explicit arguments.
- F-00000010 produces R10.

The address and stored-value arithmetic involving U in this phase is unsigned 32-bit arithmetic.

After F-00000010, W(4 × R10 + 20) receives the retained masked value logically shifted right by 9. F-00000011 receives 4 × R10. F-00000012 through F-00000014 have no explicit arguments, and F-00000015 produces R15.

The extracted value therefore contains input bits 9 through 28, shifted into positions 0 through 19. Input bits 29 through 31 do not contribute.

### First word-derived arguments

After F-00000015, W(16 × R15 + 20) receives 4 × R15.

F-00000016 receives three explicit arguments:

1. An unsigned word read from W(4 × R15 + 20), logically shifted right by 12.
2. 4 × R15.
3. An unsigned word read from W(4 × R15 + 20), shifted left by 2 with 32-bit unsigned narrowing.

The store address and read address are different expressions: 16 × R15 + 20 versus 4 × R15 + 20. They must not be treated as one location.

The two word references occur within the same call expression. Their relative evaluation order is not established. Nor does the evidence establish that they observe identical values in volatile, concurrent, or externally changing memory.

### Repeated result propagation

The next interactions have the following order. Each row begins after the preceding row’s final call has returned.

| Ordered interval | Required memory effect and invocation arguments |
|---|---|
| F-00000017–F-00000021 | Calls 17–19 have no explicit arguments; call 20 produces R20. W(4 × R20 + 20) receives 4 × R15. Call 21 receives 4 × R20. |
| F-00000022–F-00000026 | Calls 22–24 have no explicit arguments; call 25 produces R25. W(16 × R25 + 20) receives 4 × R25. Call 26 receives 4 × R25, 4 × R25, and U shifted left by 26. |
| F-00000027–F-00000031 | Calls 27–29 have no explicit arguments; call 30 produces R30. W(4 × R30 + 20) receives 4 × R25. Call 31 receives 4 × R30. |
| F-00000032–F-00000036 | Calls 32–34 have no explicit arguments; call 35 produces R35. W(16 × R35 + 20) receives 4 × R35. Call 36 receives 4 × R35. |
| F-00000037–F-00000041 | Calls 37–39 have no explicit arguments; call 40 produces R40. W(4 × R40 + 20) receives 4 × R35. Call 41 receives 4 × R40. |
| F-00000042–F-00000046 | Calls 42–44 have no explicit arguments; call 45 produces R45. W(16 × R45 + 20) receives 4 × R45. The unsigned word at W(4 × R45 + 20) is then retained as H, before call 46 receives 4 × R45. |

Call 26’s third argument depends only on U’s low six bits. It is an unsigned 32-bit left shift.

The retention of H is material: later use depends on this earlier read, not on rereading its address after intervening dependencies.

### Retained field and scaled-result field

F-00000047 through F-00000049 have no explicit arguments. F-00000050 produces R50.

The represented product 4 × R50 is assigned to unsigned Q. W(Q + 20) then receives H logically shifted right by 12. The addition to Q is unsigned 32-bit arithmetic, subject to the earlier caveat concerning formation of Q.

F-00000051 receives Q. F-00000052 through F-00000054 have no explicit arguments. F-00000055 produces R55, after which W(16 × R55 + 20) receives 4 × R55.

F-00000056 receives:

1. (Q AND 0x3f) shifted left by 14.
2. 4 × R55.
3. R50 shifted left by 30.

The first argument uses Q, not H. Where the represented scaling successfully produces the expected fourfold value, Q’s lowest two bits are zero, so only output bits 16 through 19 can be set in this argument. The third argument’s signed-shift behavior outside its valid range remains unresolved.

F-00000057 through F-00000059 have no explicit arguments. F-00000060 produces R60. W(4 × R60 + 20) receives 4 × R55, and F-00000061 receives 4 × R60.

### First byte store

F-00000062 through F-00000064 have no explicit arguments. F-00000065 produces unsigned R65.

A byte is read from B(R50 × 0x10000000 + 1) and interpreted as an unsigned value from 0 through 255. That **byte value**, plus 17, selects the destination of the first byte store. The destination’s numeric range is therefore 17 through 272; it is not the selector’s address plus 17.

The stored byte has the low eight bits of R65 shifted left by 2. Its lowest two bits are zero.

F-00000066 then receives:

1. The signed intermediate assigned from R65 shifted left by 2.
2. The same intermediate again.
3. R65 shifted left by 4, as an unsigned 32-bit result.

For values of the unsigned shifted result not representable as signed 32-bit, the conversion governing the first two arguments remains `documented_only`. The represented byte-store effect retains the low eight bits.

### Second byte store and final interactions

F-00000067 produces an opaque 32-bit result. F-00000068 receives that result. F-00000069 through F-00000071 have no explicit arguments, and F-00000072 produces R72.

The byte at B(R50 × 0x10000000 + 1) is read **again**. This second unsigned byte is retained independently of the first read.

At the address selected by the second byte value plus 17, the second byte store writes the low eight bits of:

(R65 AND 0x3fffffff), logically shifted right by 20.

The pre-narrowing result can contain ten bits. Only original R65 bits 20 through 27 survive the byte store. Bits 28 and 29 are discarded by narrowing; bits 30 and 31 are removed by the mask.

The remaining calls occur in this order:

- F-00000073 receives R72 shifted left by 2.
- F-00000074 through F-00000076 have no explicit arguments.
- F-00000077 produces R77.
- F-00000078 receives R77 shifted left by 2 and the retained second selector byte.
- F-00000079 has no explicit arguments.

The shifts of R72 and R77 are represented signed left shifts, with unresolved out-of-range behavior. After F-00000079, the displayed path reaches the invalid-instruction boundary.

## Side effects

### Memory writes

There are **15 explicit stores: 13 word stores and two byte stores**.

The word-store positions are exhaustively:

| Position | Destination | Stored word |
|---|---|---|
| Before call 1 | P + 20 | Implicit entry word |
| After call 5 | U × 0x10000000 + 20 | U × 0x04000000 |
| After call 10 | 4 × R10 + 20 | (U AND 0x1fffffff) logically shifted right by 9 |
| After call 15 | 16 × R15 + 20 | 4 × R15 |
| After call 20 | 4 × R20 + 20 | 4 × R15 |
| After call 25 | 16 × R25 + 20 | 4 × R25 |
| After call 30 | 4 × R30 + 20 | 4 × R25 |
| After call 35 | 16 × R35 + 20 | 4 × R35 |
| After call 40 | 4 × R40 + 20 | 4 × R35 |
| After call 45 | 16 × R45 + 20 | 4 × R45 |
| After call 50 | Q + 20 | H logically shifted right by 12 |
| After call 55 | 16 × R55 + 20 | 4 × R55 |
| After call 60 | 4 × R60 + 20 | 4 × R55 |

The byte stores occur after calls 65 and 72, respectively, each after its associated selector read.

These totals exclude any writes performed inside dependencies. Repeated offset 20 does not establish an object layout beyond the individual 32-bit accesses. No allocation extent, field meaning, or disjointness is established.

### Reads and aliasing

The explicit read references are:

- Two word references preparing call 16.
- One retained word read before call 46.
- Two separate selector-byte reads.

The static representation does not establish instruction-level load counts for the two word references within call 16.

The byte-store destinations may coincide or differ. Any destination may alias another accessed location, including a selector location. Consequently, later reads must not be replaced with assumed earlier values. No atomicity, synchronization, volatility, or memory-stability guarantee is established.

### Ownership and partial progress

No explicit allocation, release, ownership transfer, or cleanup is shown. Passing a computed value to a dependency does not establish that it denotes valid or owned storage.

Dependencies may alter memory, globals, resources, or other machine state. They are not assumed pure, non-throwing, memory-preserving, or ownership-neutral.

There is no displayed rollback. Earlier writes may remain observable if a later operation fails or a dependency does not return.

## Dependencies

F-00000001 through F-00000079 identify invocation sites only. No dependency implementation or semantic contract is supplied.

Established target-construction distinctions are:

- F-00000001 uses entry link-related state plus 4094 bytes.
- Subsequent sites use displayed fixed code-reference bases.
- F-00000066 uses a 4076-byte displacement.
- Other fixed-base sites use a 4094-byte displacement.

These constructions do not establish executable validity, compatible signatures, instruction-set mode, callable boundaries, or target independence.

Results explicitly consumed later come from sites 10, 15, 20, 25, 30, 35, 40, 45, 50, 55, 60, 65, 67, 72, and 77.

## Acceptance examples

These are conditional consequences of the static representation, not executed tests. They require normal continuation, valid accesses, and defined arithmetic wherever applicable.

### Unsigned input behavior

For U = 0x20000200:

- The word store after call 5 targets numeric address 20 and stores zero.
- The extracted field stored after call 10 is 1.
- Call 26’s third argument is zero.

These results require 32-bit unsigned arithmetic rather than unbounded multiplication.

### Distinct word addresses and independent reads

For R15 = 4, the store after call 15 writes 16 at address 84. Call 16’s word references instead address 36.

If both references observe 0x12345678, the first argument is 0x00012345 and the third is 0x48d159e0.

If the reference supplying the first argument observes 0x12345000 while the reference supplying the third observes 3, those arguments are 0x00012345 and 12. The contract does not impose an evaluation order between those references.

### Retained data versus scaled-result data

If H is 0x89abcdef, the store after call 50 writes 0x00089abc, even if intervening dependencies change H’s original memory location.

If R50 is 3 and Q is consequently 12, call 56’s first argument is 0x00030000, independently of H. This example does not establish the behavior of the separate signed expression R50 shifted left by 30.

### Byte narrowing and changing selectors

For R65 = 0x12345678:

- The first stored byte is 0xe0.
- The second stored byte is 0x23.

If the first selector read yields 5 and the second yields 9, the stores target addresses 22 and 26. Call 78 receives 9 as its second explicit argument, even if the selector location changes again before that call.

### Store accounting and interrupted progress

If the entire displayed sequence reaches its terminal boundary, the body contributes exactly 13 word stores and two byte stores, excluding dependency effects.

If the initial store cannot complete, no subsequent invocation is established. If call 46 does not return, H has already been read, but the store after call 50 is not required to occur. No recovery or cleanup sequence is shown.

## Uncertainties

The following remain `documented_only`:

- Whether the recovered sequence accurately represents executable behavior, given the reported invalid instruction data.
- Application-level intent.
- Exact ABI, argument passing, register preservation, and dependency signatures.
- Provenance and meaning of implicit entry state.
- Signed overflow, signed shifts, out-of-range signed conversion, pointer conversion, and address-wrap behavior.
- Endianness, required alignment, accessible address ranges, and permissions.
- Whether accesses involve ordinary memory, volatile state, concurrency, or memory-mapped devices.
- Evaluation order and actual load behavior for call 16’s two word references.
- Dependency ownership, lifetime, allocation, release, and failure behavior.
- Actual behavior at or beyond the terminal invalid-instruction boundary.

An implementation promising ordinary return, validated addresses, stable external state, unconditional signed wrapping, or automatic cleanup would add unsupported guarantees.

## Evidence

The evidence is exclusively the supplied static function representation, including its warnings about invalid instruction data.

Direct static observations support the represented input widths, call ordering, access widths, address relationships, masks, shifts, retained values, and absence of a displayed normal return. Store accounting follows the 13 individually enumerated word-store positions and the two selector-directed byte stores.

Descriptions such as “field,” “propagation,” and “selector” identify observable value relationships only. They do not establish domain intent.

No runtime tests, instruction-level execution, dependency inspection, or independent implementation verification have been performed.
