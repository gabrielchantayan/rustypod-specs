<!-- Trusted provenance: F-7e52a7ad2ba1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7e52a7ad2ba1

## Purpose

F-7e52a7ad2ba1 performs an ordered sequence of calls and memory accesses, including four consecutive 32-bit stores through a loaded pointer. The visible terminal path reaches a boundary marked as invalid instruction data; normal return is not established.

The supplied static representation does not establish an application-level purpose. The four-word sequence resembles record initialization, but allocation, object type, container membership, and ownership transfer remain unproven.

This contract separates:

- **Established static evidence:** explicit calculations, operand widths, call targets, arguments, memory accesses, and ordering.
- **Inferred intent:** interpreting integer-derived values as addresses and the four-word region as a record.
- **documented_only unknowns:** dependency behavior, ABI details, address validity, unresolved value provenance, and behavior beyond the terminal boundary.

Every later operation is conditional on earlier calls returning control and earlier accesses completing. No runtime tests have been performed.

## Interface

### Explicit inputs

| Specification label | Width and interpretation | Visible use |
|---|---|---|
| A | 32 bits; interpretation unspecified | Argument to the first indirect call. |
| B | 32 bits; interpretation unspecified | No explicit use in the visible body. |
| C | Unsigned 32 bits | Address calculations, logical shifts, stored values, and call arguments. |
| D | Unsigned 32 bits | Logical right shift by two; the result supplies addresses and stored values. |

B has no explicit direct use. This does not establish that unknown dependencies cannot observe its incoming machine representation.

### Additional state dependencies

| Specification label | Represented form | Visible use |
|---|---|---|
| Link state | Signed address-like integer | Forms the first indirect call target. |
| Register base | Signed 32-bit integer in the apparent machine model | Supplies a field address and is later shifted left by one. |
| Helper value | 32 bits | Second argument to F-7d50b94821c2. |
| Word base | Signed address-like integer | Supplies a word-store destination at offset 0x20. |
| Record base | Signed address-like integer | Supplies a byte-store address, a pointer-load address, and a stored address-like value. |
| Encoded base | Unsigned 32 bits | Shifted right by two for a stored word and a later byte-read address. |

Helper value, Word base, Record base, and Encoded base are represented as incoming stack-associated state. This does not establish additional public parameters or a complete calling convention.

These labels describe operands, not preservation guarantees. In particular, reuse of Encoded base on opposite sides of a call does not establish that its backing state remains unchanged.

The final byte store also depends on an **unresolved one-byte value**. The representation declares and uses this value but supplies no explicit definition identifying its producer. It is not established as an entry argument, a constant, or an output of any particular dependency.

### Outputs and completion

No normal return value is established. Observable effects include:

- The explicit memory reads and writes below.
- Four dependency calls and their explicit arguments.
- Effects performed by those dependencies.
- Any access failure, call failure, or terminal-instruction behavior.

A void-like signature does not establish successful completion. The terminal boundary does not establish a particular exception, process exit, or continuation.

### Widths and arithmetic

The explicit memory accesses use 8-bit bytes, 16-bit halfwords, and 32-bit words. The loaded pointer and the four fields written through it are represented as 32-bit values.

For this contract:

- **C-quarter** means unsigned C logically shifted right by two.
- **D-quarter** means unsigned D logically shifted right by two.
- **Encoded-quarter** means unsigned Encoded base logically shifted right by two at the relevant use.

Halfword stores retain the low 16 bits of their source. No saturation or range check is visible.

At the represented unsigned 32-bit expression level, multiplication of C by four and additions to unsigned expressions use arithmetic modulo \(2^{32}\). This includes the address expressions \(4C + 0x14\), \(4C + 0x3C\), and \(C + 6\).

Signed address-like additions and the left shift of Register base do not establish a portable overflow rule. Treating those results as addresses additionally depends on the execution environment. Endianness, pointer-conversion rules, and unaligned-access behavior remain documented_only.

## Behavior

### 1. First indirect call and initial stores

The first indirect call, F-I01, targets the address formed by:

**Link state + 0x1B0**

Its sole explicit argument is A. The calculated address is used directly as the call target; no intervening function-pointer load from that address is represented.

On return, the call’s 32-bit result is stored at **4 × C + 0x14**.

The following stores occur next, in order:

| Order after result store | Destination | Width | Value |
|---:|---|---:|---|
| 1 | D-quarter + 2 | 16 bits | Low 16 bits of D-quarter |
| 2 | C-quarter + 0x24 | 32 bits | D-quarter |
| 3 | C-quarter + 4 | 16 bits | Low 16 bits of C-quarter |

The function then calls F-I02 at the fixed computed target:

**0x08983287 + 0x1B0 = 0x08983437**

The two explicit arguments are C and C-quarter. No result from this call is explicitly consumed.

The numeric target is established. Whether the target architecture assigns execution-state meaning to particular address bits is not established; the target is not normalized here.

### 2. Stores preceding the helper

After F-I02 returns, the visible sequence is:

1. Store the low 16 bits of D-quarter at **C + 6**.
2. Perform a field-based word store using the 32-bit field at **Register base + 0x3C**.
3. Store C-quarter as a 32-bit word at **4 × C + 0x3C**.
4. Store C-quarter as a 32-bit word at **Word base + 0x20**.
5. Store C-quarter as a 32-bit word at **C-quarter + 0x14**.
6. Capture a C-quarter value for the final indirect call.
7. Store Encoded-quarter as a 32-bit word at **C-quarter + 0x24**.
8. Call F-7d50b94821c2 with **C logically shifted right by four** and **Helper value**, in that argument order.

The helper’s first argument is C shifted right by four, not C-quarter shifted right by four.

The field-based operation in step 2 uses the field at Register base + 0x3C both as a destination base and as the word to store. The destination is that base plus 0x18. If both field evaluations produce X, the operation stores X at X + 0x18.

The representation contains the field read in both roles but does not establish the number or ordering of underlying machine loads. If the field can change between evaluations, the exact destination/value pairing remains documented_only.

Step 7 targets the same address as the earlier word store of D-quarter: **C-quarter + 0x24**. At that point, it replaces the earlier stored word with Encoded-quarter, subject to intervening aliasing and dependency effects. The helper may subsequently modify that location.

### 3. Post-helper read and four-word region

After F-7d50b94821c2 returns, the function reads one unsigned byte from:

**Encoded-quarter + 0x18**

This access uses Encoded base at its post-helper use. The helper’s contract does not establish preservation of that state or of the memory being read.

The function then forms:

- **V:** Record base + 0x31.
- **Doubled base:** Register base shifted left by one.

Doubled base is formed before the final indirect call and used after it. Its name describes the apparent arithmetic relationship, not a guarantee of portable signed-overflow semantics.

The subsequent accesses occur in this order:

1. Store the byte value **100** at **Record base + 0x45**.
2. Load a 32-bit pointer-like value **P** from **Record base + 0xA5**.
3. Store V as a 32-bit word at **the previously read unsigned byte + 0x38**.
4. Perform four 32-bit stores through P:

| Offset from P | Value |
|---|---|
| 0x00 | 100 |
| 0x04 | V |
| 0x08 | V |
| 0x0C | P |

The byte-derived destination is an absolute numeric address in the representation. It is not relative to Record base or Encoded-quarter. Its possible numeric range is **0x38 through 0x137**, inclusive.

The load of P precedes the byte-derived word store. Consequently, if that word store overlaps the location from which P was loaded, the subsequent four stores still use the already loaded pointer. No reload of P is represented.

The four field stores are ordered, separate operations. No atomic publication, transaction, or rollback is established. If they overlap earlier locations, their writes can replace earlier data.

### 4. Final indirect call and unresolved byte store

After all four stores through P, the function calls F-I03 at:

**0x0898338B + 0xF8 = 0x08983483**

Its three explicit arguments are:

1. 100.
2. V.
3. The C-quarter value captured before the helper, plus 0x25.

No output is explicitly established as being produced by this call.

After the call returns, the function stores the unresolved one-byte value at:

**Doubled base + 0x1E**

The destination uses the value formed before F-I03; it is not recomputed from a fresh Register base evaluation after that call.

The final byte’s provenance remains unresolved. Its representation suggests recovered register-related state, but no explicit definition associates it with F-I03, another dependency, or an entry-state value. Temporal placement after F-I03 does not establish a producer relationship. In particular, the contract must not treat it as F-I03’s ordinary return value or as an established secondary output.

The representation then reaches the boundary marked as invalid instruction data. No reliable subsequent behavior is supplied.

## Side effects

Memory is accessed through multiple independently derived addresses. There are no visible checks for:

- Null or invalid addresses.
- Sufficient readable or writable extent.
- Alignment.
- Executability of call targets.
- Dependency success or failure.
- Pointer ownership, lifetime, or retention.

For execution to reach later operations, each preceding call must return control and each preceding access must complete. These are execution preconditions, not validations performed by this function.

The observable phase boundaries are:

- F-I01 precedes every explicit store.
- Its result store and the next three stores precede F-I02.
- The replacement word at C-quarter + 0x24 precedes F-7d50b94821c2.
- The unsigned-byte read follows that helper.
- The four stores through P precede F-I03.
- The unresolved-byte store follows F-I03.

No disjointness is established among the memory regions. Overlapping writes, dependency effects, concurrent modification, or memory-mapped semantics may affect later observations.

No allocation or deallocation is explicit in the body. Unknown dependencies may allocate, release, retain, mutate, or invalidate memory; their contracts are unavailable. Passing an address-like value does not independently establish ownership transfer or preservation.

There is no visible rollback. A failure can leave earlier stores and dependency effects observable.

## Dependencies

F-I01, F-I02, and F-I03 are document-local opaque call-site identifiers, not established global function identities.

| Dependency | Established target | Explicit arguments | Established consumed output |
|---|---|---|---|
| F-I01 | Link state + 0x1B0 | A | 32-bit call result, stored at 4 × C + 0x14 |
| F-I02 | 0x08983437 | C; C-quarter | None explicitly |
| F-7d50b94821c2 | Supplied opaque dependency identity | C logically shifted right by four; Helper value | None explicitly |
| F-I03 | 0x08983483 | 100; V; captured C-quarter + 0x25 | None explicitly established |

The unresolved final byte is not attributed to any dependency.

All four dependency contracts are documented_only. Their complete argument conventions, memory effects, ownership effects, register-preservation rules, failure behavior, and normal-return guarantees are unavailable.

Unknown dependency behavior does not make the two fixed numeric target calculations unknown.

## Acceptance examples

These are conditional static deductions, not executed tests. Unless otherwise stated, they assume returning dependencies, successful accesses, and no interfering mutation or aliasing.

### Halfword truncation

For D = 0x0004000C:

- D-quarter is 0x00010003.
- The halfword store at address 0x00010005 writes 0x0003.
- The initial word store at C-quarter + 0x24 writes 0x00010003.

Writing only 0x00000003 to the latter destination would not match the represented word-store value.

### Replacement before the helper

For D = 0x1000 and Encoded base = 0x2000 at its pre-helper use:

- The initial word at C-quarter + 0x24 is 0x400.
- The later store to that destination writes 0x800.
- The replacement occurs before F-7d50b94821c2.

This does not guarantee that 0x800 remains there after the helper returns.

### Unsigned address arithmetic

For C = 0x40000000, unsigned 32-bit multiplication by four produces zero modulo \(2^{32}\). The two destinations based on that product are therefore 0x14 and 0x3C.

This arithmetic deduction does not establish that those addresses are writable.

### Fixed targets and helper argument

If reached, F-I02 targets 0x08983437 and F-I03 targets 0x08983483.

For C = 0x12345678:

- C-quarter is 0x048D159E.
- The helper’s first argument is 0x01234567.
- The final call’s third argument is 0x048D15C3.

A helper argument of 0x0048D159 would apply an additional shift not represented by the source.

### Unsigned byte-derived destination

If the post-helper byte read yields 0xFF, the word store of V targets 0x137.

If the read yields zero, it targets 0x38.

Neither destination includes an additional base address. Alignment and accessibility remain environmental conditions.

### Four-word layout and pointer capture

If the loaded pointer is P and the prepared address-like value is V, the four stores write 100, V, V, and P at offsets 0, 4, 8, and 12, respectively.

The separate store at Record base + 0x45 writes only one byte.

If the intervening byte-derived word store overlaps the saved-pointer location, the four stores still use the pointer value loaded before that overlap. They do not follow a newly overwritten pointer.

### Final byte with unresolved provenance

If the unresolved value used by the final store is 0xAB, and execution reaches that store, it writes 0xAB at Doubled base + 0x1E.

A value observed as an output of F-I03 alone is insufficient to predict this byte without additional evidence connecting that output to the unresolved operand. This example establishes neither such a connection nor normal completion afterward.

## Uncertainties

The following remain documented_only:

- Reliability of the recovered entry point, signature, and terminal control flow.
- Complete ABI, pointer interpretation, byte order, and unaligned-access behavior.
- Platform treatment of address conversions, signed additions, and signed left shifts.
- Provenance and validity of the additional register-associated and stack-associated state.
- Dependency effects on memory, globals, operand backing storage, and ownership.
- Whether repeated operand uses across calls observe preserved or changed state.
- Exact machine-load count and evaluation order for the field-based store.
- Effects of aliasing, concurrency, and memory-mapped accesses.
- Validity and application meaning of the low-address byte-derived word store.
- The producer and meaning of the unresolved final byte.
- Whether any dependency produces that byte; specifically, its association with F-I03 is not established.
- Behavior at or beyond the invalid-instruction boundary.
- Application-level meaning and lifetime of the four-word region.

These uncertainties do not weaken the established access widths, explicit shift amounts, fixed numeric call targets, argument expressions, or ordered stores.

## Evidence

The supplied static representation establishes the four explicit inputs, additional state operands, byte/halfword/word access widths, unsigned shifts, constants, call arguments, repeated destination, four-word layout, and visible operation ordering.

It explicitly supplies the fixed target calculations **0x08983287 + 0x1B0** and **0x0898338B + 0xF8**, yielding **0x08983437** and **0x08983483**.

It also supplies a declared one-byte operand used by the final store without an explicit producing definition. That evidence establishes the store’s width and position after F-I03, but not attribution of the stored value to F-I03.

The terminal region is marked as invalid instruction data with truncated control flow. A reliable normal return and behavior beyond that boundary are not established.

No runtime tests have been performed. Acceptance examples are static deductions under their stated conditions.
