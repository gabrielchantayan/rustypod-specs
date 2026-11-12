<!-- Trusted provenance: F-2ebaf1c41fd6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2ebaf1c41fd6

## Purpose

The supplied static representation establishes a sequence of indirect calls, address calculations, word stores, and byte-selected writes. It does not establish an application-level purpose.

The path:

- Forwards its explicit input to an entry-state-dependent call target.
- Passes selected dependency results to later dependencies.
- Combines implicit entry values and dependency results to form memory addresses and stored values.
- Performs fourteen 32-bit stores, including a zero store at absolute address `0x24`.
- Reads four unsigned bytes and uses each byte to select a low-address byte-write destination.
- Ends at an invalid-instruction termination marker, without a demonstrated normal return.

This contract describes the displayed path conditionally. It does not establish that the recovered instruction stream is valid, that the apparent function boundary is correct, or that execution can reach every operation.

## Interface

### Explicit input and output

The displayed interface accepts one opaque 32-bit input, **P**. Its interpretation and signedness are unspecified. The first dependency receives P unchanged.

No normal return value is established. Although the displayed interface declares no returned value, the represented path terminates at an invalid-instruction marker rather than a normal return. No success status, error result, or normal continuation is shown.

### Implicit inputs

The path also consumes entry state absent from the explicit interface:

| Label | Width or interpretation | Use |
|---|---|---|
| A | 32-bit address-forming value | Base for the first word store |
| B | Unsigned 32-bit value | Initial self-address store and logical shifts |
| C | Unsigned 32-bit value | Logical shifts and a later byte-read address calculation |
| V | Opaque 32-bit value | Data for the first word store |
| H | Entry-state address-forming value | Basis of the first indirect call target |

These labels are contract notation, not fields of an established object layout. Supplying only P does not supply all state consumed by this path.

A further 32-bit value, **E**, is consumed from auxiliary call-related register state after F-X15 and before F-X16. Its producer and calling-convention meaning are unresolved.

### Widths and arithmetic

Use these derived values:

| Label | Definition | Range |
|---|---|---|
| Q | B logically shifted right by 3 | `0`–`0x1fffffff` |
| R | B logically shifted right by 22 | `0`–`0x3ff` |
| S | C logically shifted right by 3 | `0`–`0x1fffffff` |
| T | C logically shifted right by 6 | `0`–`0x03ffffff` |

**low-byte(x)** denotes the least significant eight bits of x.

**fragment(x)** denotes the least significant eight bits of the result of masking x with `0x3fffffff` and logically shifting right by 20. Its final byte therefore contains original bits 20–27; the expression does not retain all ten bits remaining after that shift.

Word stores write 32-bit values. Byte stores write eight-bit patterns. Logical shifts of unsigned 32-bit values have the displayed unsigned interpretation.

Several scaled dependency results and address additions are represented using signed 32-bit arithmetic. Their behavior outside representable signed ranges is not independently established. Where examples require wrapping addresses or scaled signed values, they explicitly assume a modulo-2³² machine-arithmetic model.

The unsigned product S × `0x80000` produces a 32-bit bit pattern. Its subsequent use through a signed address-forming value introduces a separate interpretation issue; this contract does not infer a complete execution model from that representation.

### Preconditions for completing the displayed sequence

Every reached call target must be executable under the actual calling convention. Every reached memory access must be permitted, including low-address accesses and potentially unaligned word stores. Dependencies must return to the represented continuation.

The path performs no visible validation of these conditions. A fault or nonreturning dependency prevents later operations.

## Behavior

### Ordering conventions

Dependencies are labeled **F-X01** through **F-X84**, in encounter order. These opaque identifiers distinguish call sites, not proven distinct function identities.

“No explicit arguments” means none appear in the supplied representation. It does not exclude implicit register, memory, or global inputs.

The following stages preserve the displayed ordering. Dependencies may introduce additional effects between direct accesses. A retained value in this contract expresses a displayed data dependency, not an independently verified register-preservation rule.

### 1. Initial stores and forwarding calls

Before any dependency call:

1. Store V as 32 bits at A + `0x44`.
2. Store B as 32 bits at B + `0x24`.

Then perform:

| Call | Explicit arguments | Used result |
|---|---|---|
| F-X01 | P | None shown |
| F-X02 | None | Opaque 32-bit value |
| F-X03 | Result of F-X02 | None shown |
| F-X04 | None | Opaque 32-bit value |
| F-X05 | Result of F-X04 | None shown |
| F-X06–F-X08 | None, in order | None shown |
| F-X09 | None | 32-bit value J1 |

After F-X09:

1. Store Q as 32 bits at 4 × J1 + `0x44`.
2. Store R as 32 bits at R + `0x24`.
3. Invoke F-X10 with 4 × J1.

Q and R derive from B, not J1.

### 2. Split-result use

Invoke F-X11, F-X12, and F-X13 with no explicit arguments, in that order. Invoke F-X14 with no explicit arguments.

F-X14 supplies a displayed 64-bit result. Let **L** be its lower 32 bits and **U** its upper 32 bits.

Next:

1. Invoke F-X15 with L shifted left by 2.
2. Store Q as 32 bits at 2 × U + `0x44`.
3. Store R as 32 bits at R + `0x24`.
4. Invoke F-X16 with three arguments: 2 × R, E, and E shifted left by 2.

The upper-half-based store occurs after F-X15. F-X15 can therefore affect memory before that store. The representation does not establish the ABI mechanism supporting the split result or auxiliary value E.

The shifts of values represented as signed, including L and E, require the arithmetic qualification in the interface.

### 3. Fixed-address clearing and repeated arguments

Perform:

| Call | Explicit arguments | Used result |
|---|---|---|
| F-X17 | None | None shown |
| F-X18 | S | None shown |
| F-X19–F-X21 | None, in order | None shown |
| F-X22 | None | J2 |

Then:

1. Store Q as 32 bits at 4 × J2 + `0x44`.
2. Store the 32-bit value zero at absolute address `0x24`.
3. Invoke F-X23 with 4 × J2.

The zero write is an actual unconditional operation on this displayed path, not a null check.

Invoke F-X24, F-X25, and F-X26 with no explicit arguments. F-X27, also with no explicit arguments, supplies J3.

Invoke F-X28 with J3 shifted left by 2 and retain its result as J4. Then:

1. Store Q as 32 bits at 4 × J4 + `0x14`.
2. Invoke F-X29 with J4, J4, J4, and S.

The first three arguments to F-X29 are the same retained result.

### 4. Further result-indexed word stores

Invoke F-X30, F-X31, and F-X32 with no explicit arguments. F-X33 supplies J5 with no explicit arguments.

1. Store Q as 32 bits at 4 × J5 + `0x14`.
2. Invoke F-X34 with 4 × J5.

Invoke F-X35, F-X36, and F-X37 with no explicit arguments. F-X38 supplies J6.

Let **K** be 4 × J6.

1. Store K as 32 bits at 16 × J6 + `0x14`.
2. Invoke F-X39 with K.

Invoke F-X40, F-X41, and F-X42 with no explicit arguments. F-X43 supplies J7.

1. Store the earlier K as 32 bits at 4 × J7 + `0x14`.
2. Invoke F-X44 with 4 × J7.

The data written through J7 remains derived from J6.

Invoke F-X45, F-X46, and F-X47 with no explicit arguments. F-X48 supplies J8.

1. Store 4 × J8 as 32 bits at 16 × J8 + `0x14`.
2. Invoke F-X49 with 4 × J8 and 16 × J8.

No bounds or allocation-size checks accompany these stores.

### 5. C-derived word store

F-X50 supplies an opaque 32-bit result with no explicit arguments.

1. Invoke F-X51 with that result and T.
2. Invoke F-X52, F-X53, and F-X54 with no explicit arguments.
3. Invoke F-X55 with no explicit arguments to obtain J9.
4. Store S as 32 bits at 4 × J9 + `0x14`.
5. Invoke F-X56 with 4 × J9.

This is the final direct word store in the displayed path.

### 6. First pair of byte-selected writes

Invoke F-X57, F-X58, and F-X59 with no explicit arguments. F-X60 supplies an unsigned 32-bit value **X**.

Let **M** denote the address-forming value obtained from the 32-bit product S × `0x80000`, subject to the interpretation qualification above.

Then:

1. Read an unsigned byte **B1** at M + 1.
2. Store low-byte(4 × X) at B1 + `0x11`.
3. Invoke F-X61 with X shifted left by 2.
4. Invoke F-X62 with no explicit arguments.
5. Invoke F-X63 with the result of F-X62.
6. Invoke F-X64, F-X65, and F-X66 with no explicit arguments.
7. Invoke F-X67 with no explicit arguments to obtain J10.
8. Read an unsigned byte **B2** at M + 1.
9. Store fragment(X) at B2 + `0x11`.
10. Invoke F-X68 with J10 shifted left by 2.

B2 is a fresh read, not a reused B1. Intervening dependencies or an aliasing byte write can change the selector location.

J10 supplies the argument to F-X68; it does not supply the preceding byte’s data.

### 7. Second pair of byte-selected writes

Invoke F-X69, F-X70, and F-X71 with no explicit arguments. F-X72 supplies an unsigned 32-bit value **Y**.

Then:

1. Read an unsigned byte **B3** at M + 1.
2. Store low-byte(4 × Y) at B3 + `0x11`.
3. Invoke F-X73 with B3 zero-extended to 32 bits.
4. Invoke F-X74 with no explicit arguments.
5. Invoke F-X75 with the result of F-X74.
6. Invoke F-X76, F-X77, and F-X78 with no explicit arguments.
7. Invoke F-X79 with no explicit arguments to obtain J11.
8. Read an unsigned byte **B4** at 8 × Y + 1.
9. Store fragment(Y) at B4 + `0x11`.
10. Invoke F-X80 with J11 shifted left by 2.

F-X73 receives the saved selector B3 even if the preceding write changes the memory from which B3 was read.

The fourth selector-read address derives from Y, not M. The unsigned expression 8 × Y + 1 uses 32-bit arithmetic.

Each selector is in `0`–`255`, so each selected byte-write destination lies in `0x11`–`0x110`, inclusive. The selector is one byte used directly as an address component, not a fetched pointer.

### 8. Terminal sequence

Invoke F-X81, F-X82, F-X83, and F-X84 with no explicit arguments, in order.

The representation then ends at an invalid-instruction termination marker. It establishes no normal epilogue, return, or subsequent recovery action.

## Side effects

### Direct memory effects

The displayed path contains **fourteen 32-bit stores**, partitioned as follows:

| Address form | Count | Written values |
|---|---:|---|
| Base or scaled result plus `0x44` | 4 | V once; Q three times |
| Value plus `0x24` | 3 | B once; R twice |
| Absolute address `0x24` | 1 | Zero |
| Scaled result plus `0x14` | 6 | Q twice; K twice; 4 × J8 once; S once |
| **Total** | **14** | |

There are also **four unsigned byte reads** and **four byte stores**. These counts exclude all dependency effects.

Offsets do not establish structure types. In particular, low-address accesses must not be reinterpreted as omitted operations merely because they would fault in a conventional process address space.

Destinations may overlap each other, selector locations, or dependency-accessed memory. Access order therefore matters. Endianness determines which individual bytes a word store changes; the supplied evidence does not independently establish endianness.

### Calls, ownership, and failure

All 84 dependency calls remain potentially observable, including calls whose results are unused.

No direct allocation, deallocation, ownership transfer, reference-count operation, or resource acquisition is established. Argument passing alone establishes none of those properties. Dependencies are not assumed to preserve globals, leave memory unchanged, or release supplied resources.

No visible error handling, rollback, cleanup, or atomicity guarantee exists. If execution stops early, preceding writes and dependency effects may remain.

## Dependencies

F-X01 has an entry-state-dependent target formed from H plus `0xf1c`. P does not determine that target.

F-X02 through F-X84 use targets formed from fixed code-region anchors and offsets in the supplied representation. No external semantic identity is resolved for these call sites.

Accordingly:

- Distinct call-site IDs do not prove distinct callees.
- Explicit argument counts are not verified prototypes.
- A displayed 64-bit result does not establish an ABI return convention.
- Calls without explicit arguments may consume implicit state.
- Return behavior, global preservation, memory effects, and ownership behavior remain documented_only unknowns.

The terminal marker is not treated as an ordinary callable dependency.

## Acceptance examples

These are conditional static examples, not runtime tests. Each assumes the relevant operations are reached and accesses succeed.

1. **Initial ordering.** With A = `0x1000`, B = `0x2000`, and V = `0x11223344`, the first stores write V at `0x1044`, then B at `0x2024`. F-X01 subsequently receives P unchanged.

2. **Distinct B-derived values.** With B = `0x00800018`, Q = `0x00100003` and R = 2. The first J1-indexed store writes Q, followed by a word store of 2 at `0x26`. Whether that unaligned access succeeds is unresolved.

3. **Word-store accounting.** If the complete displayed path executes, it performs fourteen direct word stores: four with offset `0x44`, three with offset `0x24`, one absolute zero store at `0x24`, and six with offset `0x14`. Dependency stores are outside this count.

4. **Mandatory clearing.** After F-X22 returns, the J2-indexed Q store precedes the zero store at `0x24`, which precedes F-X23. Omitting the zero store changes the specified path.

5. **Earlier result retained as data.** With J6 = 3 and J7 = 10, K = 12. The J7-associated store writes 12 at `0x3c`, not 40.

6. **Byte extraction.** With X = `0x12345678`, the first X-derived byte is `0xe0`; the later fragment byte is `0x23`.

7. **Fresh selector reads.** If the first selector at M + 1 is 5 and the second is 9, the corresponding X-derived stores target `0x16` and `0x1a`. Reusing the first selector would not match the path.

8. **Saved selector argument.** If B3 is 7, F-X73 receives 32-bit value 7, regardless of whether the immediately preceding byte store changes the original selector location.

9. **Different final read address.** With Y = 3, B4 is read at `0x19`. That is not M + 1 unless the independently formed addresses coincide.

10. **Unsigned product boundary.** With C = `0x00010000`, S = `0x2000`, and S × `0x80000` has low 32 bits zero. Under the stated 32-bit address interpretation, the first three selector reads use address 1.

11. **Early failure.** If the first A-based store faults, neither the B-based store nor F-X01 is reached. No recovery action is established.

## Uncertainties

### Established static evidence

The representation supports the fourteen-store count, four byte reads, four byte writes, literal offsets, displayed access widths, call ordering, result forwarding, and data relationships described above.

It also explicitly marks invalid instruction data and truncated control-flow recovery.

### Inferred intent

Repeated scaled addresses may suggest indexed data manipulation. The byte operations may suggest extraction of portions of wider values. Neither observation establishes a data format, object model, or application purpose.

These interpretations are not acceptance requirements.

### Documented_only unknowns

- Validity of the recovered instruction stream and apparent function boundary.
- Reachability of the complete path.
- Actual dependency identities, prototypes, and calling conventions.
- Provenance and meaning of E.
- Required entry setup beyond P.
- Exact signed-overflow, signed-shift, and signed address-conversion behavior.
- Endianness, alignment requirements, access permissions, and fault delivery.
- Whether accessed memory is ordinary storage or another mapped resource.
- Aliasing and dependency-induced memory or global changes.
- Dependency ownership and resource-management behavior.
- Exact runtime meaning of the terminal marker.
- Any normal completion path outside the supplied representation.

## Evidence

This contract is based solely on the supplied static representation. The memory-effect inventory counts fourteen direct word stores independently of dependency behavior: four offset-`0x44` stores, three offset-`0x24` stores, one absolute zero store, and six offset-`0x14` stores.

The representation includes warnings about invalid instruction data and overlapping global symbols, and its control-flow recovery ends at the terminal marker. Those limitations prevent treating the apparent interface, dependency prototypes, or termination behavior as independently verified runtime facts.

No runtime tests have been performed. No dependency bodies, independent instruction-level verification, or external ABI documentation were supplied.
