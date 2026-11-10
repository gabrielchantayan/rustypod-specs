<!-- Trusted provenance: F-af2efc6414cb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-af2efc6414cb

## Purpose

F-af2efc6414cb describes a sequence of memory reads and writes with two represented terminal paths, both ending at invalid instruction data. The supplied static evidence does not establish an application-level purpose or a normal return.

The opening accesses associate caller-relative storage with values obtained through hidden entry state. Later accesses use arithmetic-derived addresses, dependency entry addresses, low numerical addresses, and overlapping global storage. These operations do not establish a recognizable allocation, initialization, dispatch, or cleanup protocol.

This contract separates:

- **Established static evidence:** represented accesses, value widths, dependencies, and ordering.
- **Inferred intent:** possible interpretations not required by the evidence.
- **documented_only:** behavior that remains unresolved, including invalid full-width shifts and undeclared global access widths.

All execution claims are conditional on preceding operations completing. No runtime tests have been performed.

## Interface

### Explicit inputs and output

The represented interface has four positional arguments and no declared return value.

| Input | Represented width | Use |
|---|---:|---|
| A | 32 bits | Address base, subtraction, masking, and scaled-address calculations |
| B | 32 bits | No explicit use |
| C | 32 bits | Address base; its low byte is also written elsewhere |
| D | 32 bits | Address base; its full value is stored through A |

A, C, and D are treated as address-like integers. No object types, allocation extents, ownership rules, or valid ranges are established. B has no represented effect; its role in a complete interface remains unknown.

There is no represented success result, status code, or normal return.

### Hidden inputs and notation

The sequence additionally depends on:

- Three incoming register-held values, H₀, H₁, and H₂.
- A pointer value P supplied through incoming stack state.
- The address S of the stack location supplying P.
- Global storage read and written during execution.
- E, the entry address of F-e47d7b2e9c88.
- F, the entry address of F-8afdaab91693.

P and S are distinct. Writes through P modify memory, not the held value P. S is used in a separate scaled-address calculation.

The global-storage labels below identify roles, not disjoint allocations:

| Label | Role |
|---|---|
| Byte-constant global | Receives 0xb8 |
| Pointer-fragment global | Receives the low 16 bits of a derived address |
| First-clear global | Receives zero before the stack-derived writes |
| Mask-fragment global | Receives a value derived from A |
| Byte-composite region | Receives a widened byte, then a composite using a live byte read |
| R-value global | Receives a derived 32-bit value |
| J-source global | Supplies a value late in the sequence |
| Address-holder global | Receives a scaled P and is subsequently read for indirect stores |
| High-halfword global | Receives J’s high halfword and is read again near termination |
| Second-clear global | Receives zero near termination |
| Final-composite region | Receives a composite containing the live high-halfword value |

Some global symbols overlap. These labels do not establish separation from one another or from indirect destinations.

### Widths and arithmetic

Explicit indirect accesses establish 8-bit, 16-bit, or 32-bit memory operations as specified below. In contrast, the excerpt does not declare the types of directly named globals. A 16-bit, 24-bit, or 32-bit right-hand value does **not** independently establish a global store’s access width or footprint.

Offsets are in bytes. “Low byte” and “low halfword” mean truncation to 8 and 16 bits respectively.

For conditional address calculations, this contract uses 32-bit bit patterns and reduction modulo 2³² where appropriate. This does not resolve signed overflow, signed left shifts, target address rules, alignment, or exception behavior. Arithmetic right-shift consequences are explicitly conditional.

Required preconditions are operation-specific: every preceding access and arithmetic operation must complete, and each referenced location must permit the indicated access. No validation, bounds checking, alignment correction, or recoverable error handling is represented.

## Behavior

### 1. Initial reads and association writes

The represented memory-access order is:

1. Read a 32-bit value W at H₀ + H₂.
2. Read an unsigned byte T at H₁ + H₀.
3. Write W as a 32-bit word at D + 8.
4. Write D as a 32-bit word at A + 0x40.
5. Write W as a 32-bit word at C + 0x74.
6. Read a 32-bit value X at D + 0x28.
7. Read a 32-bit value Y at C + 0x5c.
8. Write the low byte of C at W + 3.
9. Write T as one byte at C + 0x10.
10. Write the low byte of Y at T + 0x1c.

The final destination uses T’s numerical byte value, not its original address. Before applying environment-specific address rules, it ranges from 0x1c through 0x11b.

These are sequential accesses, not an entry-time snapshot. Earlier stores can affect later reads through aliasing. Once read, W, T, X, and Y are held values; later memory writes do not implicitly reread their sources.

**Inferred intent:** the first three stores might associate related records. No object schema or conventional field interpretation is established.

### 2. Arithmetic-derived writes

Define the following values:

- U is the 32-bit value produced from A − 0x38, subject to unresolved signed-subtraction semantics.
- Q is twice the unsigned 32-bit result of subtracting 0xe0 from A, reduced to 32 bits.
- M is the unsigned subtraction result masked with 0x7fffffff.
- R is M shifted logically right by one.

Thus M ranges from zero through 0x7fffffff, and R ranges from zero through 0x3fffffff.

Another address, V, is represented by shifting signed X left by two positions and then right by 32 positions. **V is documented_only.** A right shift by the full 32-bit operand width does not establish a usable result from this evidence.

The accesses and assignments occur in this order:

1. Write the low byte of R at X + 0x0897e8b8.
2. Assign 0xb8 to the byte-constant global. Its destination access width is unresolved.
3. Write the low byte of U at A − 0x32.
4. Write U as a 32-bit word at V.
5. Write 0xffff as a halfword at Q + 4.
6. Write 0xffff as a halfword at Q + 6.
7. Write 0xb8 as one byte at V + 4.
8. Write 0xffff as a halfword at Q + 2.
9. Assign Q’s low 16 bits to the pointer-fragment global. The assigned value is 16 bits; the global access width is unresolved.
10. Read an unsigned halfword at Q.
11. Zero-extend that halfword to an address and write U masked with 0x897e8b80 there as a 32-bit word.

The three halfword stores do not directly initialize the halfword at Q. Its later read can nevertheless be affected by aliasing with other writes. The resulting indirect destination lies between zero and 0xffff.

### 3. Dependency-address writes and early termination

E is used as data. No invocation of F-e47d7b2e9c88 is represented.

Two derived values are formed:

- Z results from interpreting the full 32-bit E as signed and shifting it right by 32 positions.
- N results from interpreting the full 32-bit E as signed, shifting it right by 31 positions, and **then** narrowing the result to 16 bits.

Z is **documented_only** because of the full-width shift. N does not narrow E before shifting. Under arithmetic-right-shift semantics, N is 0x0000 when signed E is nonnegative and 0xffff when signed E is negative.

A halfword N is written at Z + 4 before testing signed E:

- If E is negative, the represented path stops at invalid instruction data.
- If E is nonnegative, execution continues.

No rollback is represented.

The continuing path performs these accesses:

1. Write N as a halfword at Z + 0x12.
2. Write E as a 32-bit word at address E.
3. Read a signed halfword at Z + R.
4. Repeat the halfword write at Z + 0x12.
5. Repeat the word write of E at E.
6. Sign-extend the previously read halfword, shift it right by 16 positions, and write the resulting 32-bit value at Z + 8.

Under arithmetic-right-shift semantics, the final stored value is zero for a nonnegative halfword and 0xffffffff for a negative halfword. The intervening writes do not cause that halfword to be reread.

### 4. Low-address, global, and stack-derived writes

On the continuing path:

1. Assign zero to the first-clear global. Its access width is unresolved.
2. Form K = (E masked with 4) + 0x58. K is either 0x58 or 0x5c.
3. Write Z as a 32-bit word at K + 4.
4. Assign the 16-bit value obtained from M shifted logically right by 17 to the mask-fragment global.
5. Assign the low byte of R, widened to a 16-bit value, to the byte-composite region.
6. Write R as a 32-bit word at 4K + 0x10.
7. Assign R to the R-value global.
8. Write K’s low halfword at P.
9. Write Z shifted left by eight as a 32-bit word at P.
10. Write that same shifted value as a 32-bit word at P + 20.
11. Write P as a 32-bit word at address P.
12. Read a 32-bit value L at address 2P.
13. Write L’s low halfword at P.
14. Write L as a 32-bit word at address L.
15. Write the signed result of shifting P right by 31 as a 32-bit word at 32S + 0x18.
16. Read the then-current byte-valued global view associated with the byte-composite region. Form a 24-bit composite with the 16-bit value 5 above that byte, and assign it to the overlapping region.
17. Write P as a 32-bit address value at 4P + 0x18.

The global access widths in steps 4, 5, 7, and 16 are not independently established. Step 16 forms the value 0x000500 combined with the live low byte; it does not establish a three-byte store.

The byte in step 16 is a **live global read at that point**, not necessarily the low byte assigned in step 5. Intervening indirect writes can overlap its storage.

Under the conditional 32-bit address model, K + 4 is 0x5c or 0x60, and 4K + 0x10 is 0x170 or 0x180. Signed shifts and scaled address calculations retain the arithmetic uncertainties stated above.

### 5. Live global indirection and final writes

The final portion begins by reading the J-source global into a held 32-bit value J and obtaining F. J is read at this late point, not captured at entry.

The represented sequence then:

1. Assigns 4P to the address-holder global. Its store width is unresolved.
2. Forms a value from F shifted left by two, subject to unresolved signed-shift semantics.
3. Reads the address-holder global for both the destination and the value of a 32-bit indirect store, then performs that store.
4. Reads the address-holder global again and writes R as a 32-bit word at that newly read address plus four.
5. Writes 4J as a 32-bit word at address 4J.
6. Writes the shifted F value as a 32-bit word at J plus the signed result of shifting J left by six and then right by 17.
7. Extracts J’s high 16 bits using a logical right shift and assigns that 16-bit value to the high-halfword global.
8. Assigns zero to the second-clear global.
9. Writes zero as a 32-bit word at 8F.
10. Writes zero as a 32-bit word at 8F + 4.
11. Writes one byte at 8F + 0x10, containing the low byte of 8F plus eight, truncated to eight bits.

The global access widths in steps 7 and 8 are unresolved. The global reads in steps 3 and 4 supply address-sized values, but their underlying access footprints are not independently declared.

Step 3 must not be replaced unconditionally with “write 4P at 4P.” Its destination and value come from live global reads. Their relative operand-evaluation order is not established. More importantly, that indirect store may overlap the address-holder itself. Step 4 therefore uses a fresh read **after** step 3, not necessarily the originally assigned 4P.

The terminal assignments follow in this order:

| Destination | Assigned value |
|---|---|
| Absolute location 0x86 | 0x897ea700 |
| Absolute location 0xea700000 | 0x66 |
| Absolute location 0x0897eb26 | 0xeaec |
| Absolute location 0x12fd4c34 | 0x1a |
| Absolute location 0x897ea65c | Its own address |
| Final-composite region | A 32-bit composite: live high-halfword value above low halfword 0xea70 |
| Absolute location 0x12fd4cb8 | 0x5c |

These directly named destinations have unresolved access widths.

For the final composite, the high-halfword global is **read again at that point**. Its value is not necessarily J’s originally extracted high halfword: the intervening clear, indirect writes, and terminal assignments may overlap it. The composite’s 32-bit value width does not establish a 32-bit destination store.

The represented continuation then stops at invalid instruction data.

## Side effects

The sequence writes to caller-relative fields, hidden-state-derived addresses, low numerical addresses, global storage, absolute locations, and dependency-address-derived destinations.

No allocation, deallocation, reference-count operation, ownership transfer, synchronization primitive, or resource-release protocol is represented. Writing an address does not establish ownership transfer.

Writing at E may modify executable storage if E denotes writable instruction memory. Whether such writes are permitted or whether the sequence is correctly represented remains **documented_only**.

Repeated and overwritten stores remain distinct operations. They can matter through aliasing, faults, device memory, or other environment-specific behavior. No atomicity or concurrency guarantees are established.

A fault or unresolved operation may prevent reaching either terminal stop. No recovery, rollback, or partial-write handling is established.

## Dependencies

| Opaque function ID | Established relationship |
|---|---|
| F-e47d7b2e9c88 | Entry address E is used for arithmetic, sign testing, and memory writes |
| F-8afdaab91693 | Entry address F is used for scaling and memory writes |

Neither dependency is invoked in the represented sequence. No assumptions about their global preservation, memory ownership, or release behavior are needed or justified.

## Acceptance examples

These are conditional static consequences, not executed tests.

1. **Byte truncation:** If C ends in byte 0xab, the write at W + 3 stores 0xab, not the full value C.

2. **Byte-derived address:** If T is 0xff and Y ends in 0x37, the final initial byte store writes 0x37 at 0x11b, provided that address is writable.

3. **Read-after-write aliasing:** If C + 0x74 and D + 0x28 identify the same word, the read producing X observes the preceding W store under ordinary coherent-memory semantics.

4. **Masked arithmetic boundary:** If unsigned A − 0xe0 yields 0xffffffff, M is 0x7fffffff and R is 0x3fffffff. The first R-derived byte write stores 0xff.

5. **Full-width sign extraction:** For E = 0x00008000, arithmetic right shift of the full signed 32-bit value by 31 gives zero, so N is 0x0000. Narrowing E to a signed halfword first would give a different result and does not match the represented operation.

6. **Early stop ordering:** If signed E is negative and execution reaches its test, the N store at Z + 4 has already occurred. Continuing-path writes must not be attributed to that branch.

7. **Live address-holder:** If the first address-holder-directed word store overlaps and changes that global, the R-store destination is the newly read holder value plus four—not unconditionally 4P + 4.

8. **Live composite byte:** If an intervening write changes the byte-composite region’s byte view to 0x9a, the later composite value is 0x00059a, regardless of the byte assigned earlier.

9. **Live final halfword:** If J initially contributes high halfword 0x1234, but an intervening write changes the high-halfword global to 0xabcd, the final composite value is 0xabcdea70. This does not determine its store footprint.

10. **No normal completion:** Even if every represented access completes, the represented outcome is an invalid-instruction stop, not a successful return.

## Uncertainties

The following remain **documented_only**:

- Validity of the entry boundary and recovered control flow.
- Results of both right shifts by 32.
- Signed-overflow and signed-shift behavior.
- Target endianness, alignment requirements, address wrapping, and fault semantics.
- Exact access widths of undeclared globals and directly named absolute destinations.
- The complete global overlap map.
- Relative evaluation order of the destination and value reads within the address-holder-directed store.
- ABI details supplying hidden registers, P, and S.
- Mapping and writability of low, absolute, and dependency-derived destinations.
- Whether the representation describes executable instructions, embedded data, or a mixture.
- Application purpose, object schemas, ownership, and resource lifetimes.
- Behavior beyond the represented invalid-instruction stops.

These limits prevent a complete executable replacement contract. They do not justify removing explicit accesses, caching live global reads, inferring destination widths from assigned values, or inventing fallback behavior.

## Evidence

The sole evidence is the supplied static representation and opaque dependency mapping. It supports the listed accesses, explicitly typed indirect widths, arithmetic dependencies, ordering, signed branch, live global reads, and terminal invalid-instruction stops.

The representation explicitly warns about invalid instruction data and overlapping globals. Direct-global declarations and a complete storage map are absent, so global access footprints remain unresolved even where assigned value widths are known.

No binary inspection, execution, memory-map validation, or runtime tests have been performed.
