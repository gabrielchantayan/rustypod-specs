<!-- Trusted provenance: F-ada7f3a62be3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ada7f3a62be3

## Purpose

The available static representation describes an ordered sequence of indirect calls interleaved with eight 32-bit memory stores and four byte stores. Several call results determine later arguments, store addresses, or stored values. Byte values loaded from memory also select destinations for the byte stores.

No application-level purpose is established. In particular, the evidence does not establish that this operation constructs an object, serializes a value, manipulates a device, or manages a collection.

The represented path ends at an invalid-instruction boundary rather than an ordinary return. The source explicitly warns about unreliable instruction decoding and truncated control flow. This contract therefore distinguishes the represented sequence from verified executable behavior. It does not establish that a real execution reaches every listed operation.

## Interface

### Explicit inputs

The represented interface has three explicit inputs:

| Input | Represented width and interpretation | Visible use |
|---|---|---|
| First input | Opaque 32-bit value | First argument of the first indirect call |
| Second input | Opaque 32-bit value | Second argument of the first indirect call |
| Third input | Signed 32-bit integer | Shifted left by two positions for the third argument of the first indirect call |

No pointer, length, ownership, or nullability interpretation is established for these inputs.

### Additional entry-state dependencies

The representation also uses four values not supplied through the explicit interface:

- **Address seed:** an incoming linkage-related value used to form the first indirect-call target by adding `0xFEC`.
- **Selector word:** an unsigned 32-bit value whose upper ten bits determine an early store address and value.
- **Bit-source word:** an unsigned 32-bit value used in right shifts, a word store, call arguments, and a later byte-load address calculation.
- **Payload word:** an opaque 32-bit value written to three computed locations.

These are descriptive contract names, not additional declared parameters. Their origin, initialization, and calling-convention status are unknown. The representation supplies no assignments establishing them locally.

### Outputs and completion

The declared result is void. There is no established successful return value, status code, or ordinary return path.

Observable outputs consist of:

- Indirect calls and their arguments.
- Eight represented 32-bit stores.
- Four byte loads and four byte stores.
- Whatever additional effects the unresolved dependencies perform.
- The represented terminal invalid-instruction condition.

### Preconditions

For the full represented sequence to proceed, every reached indirect target must be callable, every dependency must return to its continuation, and every accessed memory location must permit the required access.

No validation establishes these conditions. In particular, some destinations are small numeric addresses rather than offsets from a demonstrated object base. Ordinary process-memory accessibility must not be presumed.

## Behavior

### Interpretation and notation

The following names describe retained values and arithmetic:

- **B** is the entry bit-source word.
- **S** is the entry selector word.
- **P** is the entry payload word.
- **Q** is the unsigned value B shifted right by three positions.
- **T** is the unsigned value S shifted right by 22 positions.
- **Scale4(x)** denotes the represented multiplication by four or left shift by two, as specified below.
- **Scale16(x)** denotes the represented multiplication by sixteen.
- **Low8(x)** denotes the low eight bits selected for a byte store.

Unsigned right shifts are logical. Thus Q lies between zero and `0x1FFFFFFF`, and T lies between zero and `0x3FF`.

For unsigned 32-bit calculations, this contract uses modulo-\(2^{32}\) arithmetic. Several other calculations are represented using signed 32-bit integers. Overflow, negative signed left shifts, and conversion from arithmetic results to addresses are not fully established. Examples involving those cases are conditional on the actual instruction semantics.

Dependencies below have specification-local opaque IDs. The numeric ordering identifies call occurrences; it does not prove distinct callee identities.

### Ordered call and access sequence

The complete represented call order is:

| Dependency occurrence | Represented arguments | Retained result |
|---|---|---|
| F-000000000001 | First input, second input, third input shifted left by two | None |
| F-000000000002 | No explicit arguments | None |
| F-000000000003 | Q | None |
| F-000000000004 through F-000000000006 | No explicit arguments, in increasing ID order | None |
| F-000000000007 | No explicit arguments | A |
| F-000000000008 | Scale4(A) | None |
| F-000000000009 through F-000000000011 | No explicit arguments, in increasing ID order | None |
| F-000000000012 | No explicit arguments | C |
| F-000000000013 | C shifted left by two | D |
| F-000000000014 | D, D, D, Q | None |
| F-000000000015 through F-000000000017 | No explicit arguments, in increasing ID order | None |
| F-000000000018 | No explicit arguments | E |
| F-000000000019 | Scale4(E) | None |
| F-000000000020 through F-000000000022 | No explicit arguments, in increasing ID order | None |
| F-000000000023 | No explicit arguments | G |
| F-000000000024 | Scale4(G) | None |
| F-000000000025 through F-000000000027 | No explicit arguments, in increasing ID order | None |
| F-000000000028 | No explicit arguments | H |
| F-000000000029 | Scale4(H) | None |
| F-000000000030 through F-000000000032 | No explicit arguments, in increasing ID order | None |
| F-000000000033 | No explicit arguments | J |
| F-000000000034 | Scale4(J), Scale16(J) | None |
| F-000000000035 | No explicit arguments | Opaque 32-bit value K |
| F-000000000036 | K, B shifted right by six | None |
| F-000000000037 through F-000000000039 | No explicit arguments, in increasing ID order | None |
| F-000000000040 | No explicit arguments | L |
| F-000000000041 | Scale4(L) | None |
| F-000000000042 through F-000000000044 | No explicit arguments, in increasing ID order | None |
| F-000000000045 | No explicit arguments | Unsigned 32-bit value U |
| F-000000000046 | U shifted left by two | None |
| F-000000000047 | No explicit arguments | Opaque 32-bit value M |
| F-000000000048 | M | None |
| F-000000000049 through F-000000000051 | No explicit arguments, in increasing ID order | None |
| F-000000000052 | No explicit arguments | N |
| F-000000000053 | N shifted left by two | None |
| F-000000000054 through F-000000000056 | No explicit arguments, in increasing ID order | None |
| F-000000000057 | No explicit arguments | Unsigned 32-bit value V |
| F-000000000058 | Zero-extended byte from the immediately preceding byte load | None |
| F-000000000059 | No explicit arguments | Opaque 32-bit value R |
| F-000000000060 | R | None |
| F-000000000061 through F-000000000063 | No explicit arguments, in increasing ID order | None |
| F-000000000064 | No explicit arguments | W |
| F-000000000065 | W shifted left by two | None |
| F-000000000066 through F-000000000069 | No explicit arguments, in increasing ID order | None |

“No explicit arguments” describes the visible call expression, not a proven absence of implicit register inputs, shared-state inputs, or calling-convention obligations.

The memory operations below occur at the specified boundaries in that sequence.

### Word stores

1. **After F-000000000007, before F-000000000008:**
   - Store P as a 32-bit word at numeric address `Scale4(A) + 0x44`.
   - Then store T as a 32-bit word at numeric address `T + 0x24`.

   These are two ordered stores. T is not scaled before being used as the second address. Its possible destinations range from `0x24` through `0x423`, including addresses that need not be word-aligned.

2. **After F-000000000013, before F-000000000014:**
   - Store P as a 32-bit word at `Scale4(D) + 0x14`.

   D is the result of F-000000000013, not the earlier result C. F-000000000014 receives D itself three times, not its scaled form.

3. **After F-000000000018, before F-000000000019:**
   - Store P as a 32-bit word at `Scale4(E) + 0x14`.

4. **After F-000000000023, before F-000000000024:**
   - Retain Scale4(G).
   - Store that retained value as a 32-bit word at `Scale16(G) + 0x14`.

5. **After F-000000000028, before F-000000000029:**
   - Store the retained Scale4(G), not Scale4(H), at `Scale4(H) + 0x14`.

   This explicitly carries a value across intervening calls.

6. **After F-000000000033, before F-000000000034:**
   - Store Scale4(J) as a 32-bit word at `Scale16(J) + 0x14`.

7. **After F-000000000040, before F-000000000041:**
   - Store Q as a 32-bit word at `Scale4(L) + 0x14`.

There are eight stores because the first boundary contains two. No surrounding structure layout, allocation extent, or base pointer is established. The differing scale factors must not be normalized into one assumed array layout.

### Byte-load and byte-store sequence

After F-000000000045, form a retained address quantity:

\[
Z = (Q \times 0x80000) \bmod 2^{32}.
\]

Although this quantity is held in a signed-looking temporary in the representation, its originating multiplication uses an unsigned operand. The exact machine interpretation of subsequent signed-looking address arithmetic remains an uncertainty.

The byte operations proceed as follows:

1. **Before F-000000000046:**
   - Read one unsigned byte from address `Z + 1`.
   - Add `0x11` to that byte’s numeric value to obtain a destination address.
   - Store Low8(U shifted left by two) at that destination.

2. **After F-000000000052, before F-000000000053:**
   - Read one unsigned byte again from address `Z + 1`.
   - Add `0x11` to the newly read byte.
   - Store Low8((U AND `0x3FFFFFFF`) shifted right by 20) at that destination.

   This uses the retained U from F-000000000045. It does not use N for the stored byte. N supplies the subsequent call argument.

3. **After F-000000000057, before F-000000000058:**
   - Read one unsigned byte again from address `Z + 1`.
   - Retain that byte.
   - Store Low8(V multiplied by four) at address `retained byte + 0x11`.
   - Pass the retained byte, zero-extended, to F-000000000058.

   The call argument is the selector byte, not the stored byte and not V.

4. **After F-000000000064, before F-000000000065:**
   - Read one unsigned byte from address `(V × 8 modulo 2³²) + 1`.
   - Add `0x11` to that byte.
   - Store Low8((V AND `0x3FFFFFFF`) shifted right by 20) at that destination.

   This final read uses V, not Z. W affects only the subsequent shifted call argument.

Every byte-store destination is in the inclusive numeric range `0x11` through `0x110`. The destination calculation does not add an established allocation or object base.

The upper-part expressions produce ten-bit intermediate values before byte conversion. Consequently, the stored byte selects original bits 20 through 27 of U or V; original bits 28 and 29 are discarded by conversion to eight bits. The evidence does not justify treating the two stores as a complete encoding of a 32-bit value.

### Termination

After F-000000000069, the representation reaches an invalid-instruction marker and stops decoding control flow.

No retry, cleanup, rollback, error return, or successful completion branch is represented. If a dependency fails to return, or a memory access faults, subsequent listed operations are not established to occur.

## Side effects

The direct memory effects have widths of either four bytes or one byte. Endianness, access atomicity, alignment behavior, and memory-mapped-device semantics are not established.

Earlier writes remain observable if execution stops later. No transactional behavior is shown.

Potential aliasing matters:

- Computed word-store destinations may coincide or overlap.
- Byte stores may overlap earlier word stores.
- A byte-store destination may overlap a later selector-byte source.
- Dependencies may change memory subsequently read by this function.

The three reads from `Z + 1` are separate observable reads. They must not be replaced by one cached value without an independently established stability guarantee.

No direct allocation, release, ownership transfer, or reference-count operation is identifiable. Passing a value to an unresolved dependency does not establish whether it is an address, whether it denotes owned storage, or whether the dependency retains or releases anything.

The representation directly assigns memory rather than named global variables. The accompanying warning about overlapping global symbols does not establish a usable global-state layout.

## Dependencies

F-ada7f3a62be3 contains 69 represented indirect-call occurrences. F-000000000001 through F-000000000069 are opaque, specification-local references for those occurrences, not resolved external function identities.

The first target depends on the incoming address seed. Later targets are formed from address-like constants or code labels and fixed offsets. Their implementation, signatures, error behavior, memory effects, and termination behavior are unavailable.

For each dependency:

- Only the displayed arguments and explicitly consumed result are established.
- An ignored result is not proof of success.
- Preservation of globals or pointed-to memory is unknown.
- Ownership retention, transfer, allocation, and release are unknown.
- Hidden state inputs and additional ABI effects are unknown.

The retained values in the behavior section describe dataflow in the supplied representation. They are not an independently verified register-preservation guarantee.

## Acceptance examples

These are conditional static acceptance examples, not runtime test results.

1. **Initial argument transformation.**  
   With third input equal to 3, and a valid first call target, F-000000000001 receives 12 as its third displayed argument. The first two displayed arguments remain the two original opaque inputs.

2. **Unscaled selector address.**  
   If S is `0x00400000`, T is 1. At the first store boundary, the second word store targets address `0x25` and writes the 32-bit value 1. A store to `0x28` would incorrectly scale T. Actual completion depends on whether an unaligned word access at `0x25` is supported and writable.

3. **Retained value across calls.**  
   If G is 3 and H is 5, the store following F-000000000023 writes 12 at `0x44`. The store following F-000000000028 writes the same retained value 12 at `0x28`, not 20.

4. **Distinct byte values from one result.**  
   If U is `0xABCDEF12`, the first byte store writes `0x48`. The second writes `0xBC`. If the selector reads return `0x20` and `0x21`, respectively, their destinations are `0x31` and `0x32`.

5. **Selector argument differs from stored data.**  
   If V is 3 and the third selector read returns `0x7F`, the corresponding byte store writes `0x0C` at `0x90`. F-000000000058 receives `0x7F`, not `0x0C`.

6. **Final selector source changes.**  
   With V equal to 3, the last selector read uses address `0x19`, independently of the earlier retained Z. Reusing `Z + 1` for this read would violate the represented dataflow.

7. **Repeated reads remain independent.**  
   If a dependency changes the byte at `Z + 1` between the first and second reads, the second store uses the new selector. The first selector is not retained for reuse.

All examples assume preceding calls return and required accesses succeed. None establishes ordinary return after the final call.

## Uncertainties

The following remain **documented_only**:

- Whether the invalid-instruction warnings reflect genuine executable instructions, misidentified code boundaries, embedded data, or another decoding problem.
- Whether the represented entry signature and additional entry-state values match the real ABI.
- Actual dependency identities, signatures, implicit inputs, and side effects.
- Whether any listed low numeric addresses are accessible.
- Signed-overflow, negative-shift, pointer-conversion, and address-wrap semantics.
- Alignment requirements and byte order for word stores.
- Allocation boundaries, object layouts, and ownership rules.
- The behavior of any continuation beyond the truncated boundary.
- A high-level interpretation of the scaling and bit extraction.

Repeated scaling and low/high byte extraction may suggest an encoding or representation conversion. That is inferred intent only; it does not establish a format, data structure, or consumer.

## Evidence

This contract derives solely from the supplied static representation: its explicit call order, displayed arguments, result uses, arithmetic, memory-access widths, and terminal decoding warning.

The representation supports the stated sequence of 69 indirect-call occurrences, eight word stores, four byte loads, and four byte stores. It does not supply dependency bodies, runtime observations, a validated memory map, or an independently verified calling convention.

No runtime tests have been performed. Acceptance examples are arithmetic and ordering consequences of the supplied representation, conditional on its fidelity to executable behavior.
