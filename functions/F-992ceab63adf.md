<!-- Trusted provenance: F-992ceab63adf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-992ceab63adf

## Purpose

F-992ceab63adf has a recoverable **ordered invocation and memory-effect contract**. It makes indirect dependency calls, uses their results in address calculations, writes words and bytes, rereads byte selectors, and clears selected locations.

The supplied static evidence does not establish a higher-level service, object model, or resource lifecycle. Repeated clearing suggests initialization-like intent, but that is an inference, not an established purpose.

The visible path ends at an invalid-instruction boundary. Neither successful completion nor a normal return is established. All behavior below is conditional on the supplied static representation accurately describing executable behavior and execution reaching the relevant operation.

## Interface

### Explicit and ambient inputs

| Input | Width | Visible use |
|---|---:|---|
| First argument | 32 bits, opaque | No explicit use shown. |
| Second argument | 32 bits, opaque | Supplied to the first dependency invocation. |
| Third argument | 32 bits, opaque | No explicit use shown. |
| Fourth argument | Unsigned 32 bits | Logically shifted right by three to form the input quotient. |

Absence of explicit use does not prove ABI-level irrelevance.

Additional ambient inputs are visible:

- An incoming link-related value determines the first indirect target by addition of `0xfec`.
- An incoming 32-bit register value supplies the first direct word store.
- A secondary register result from a later invocation supplies arguments to another invocation.

Their initialization and ABI rules are unknown. The four explicit arguments alone do not define a complete execution precondition.

### Outputs

No return value is established. Observable outputs comprise:

- Dependency invocations, their order, and their explicit arguments.
- Direct memory reads and writes.
- Effects performed by dependencies.
- Arrival at the represented invalid-instruction endpoint, if reached.

### Widths and address notation

A word is 32 bits; a byte is eight bits. Read bytes are zero-extended for address arithmetic. Addresses below are byte addresses, not declarations of structures or arrays.

The contract uses these semantic quantities:

| Quantity | Meaning |
|---|---|
| \(Q\) | Fourth argument logically shifted right by three |
| \(S\) | Retained byte from the second read at \(Q+1\) |
| \(P\) | First payload supplied by a dependency |
| \(M\) | Main payload supplied later |
| \(H\) | \((M\mathbin{\&}0x3fffffff)\gg1\) |
| \(T_7\) | \((H\mathbin{\&}0x3fffff)\ll7\) |
| \(T_4\) | \((H\mathbin{\&}0x3fffff)\ll4\) |
| \(U(x)\) | Low byte of \((x\mathbin{\&}0x3fffffff)\gg20\) |
| \(L(x)\) | Low byte of \(4x\) |

An **indexed word location** for value \(x\) and offset \(d\) is \(4x+d\). A **record-scaled location** is \(16x+d\). Neither name establishes an object type.

A **selected byte write** reads a byte at a stated selector address and writes at the resulting byte value plus `0x11`. Its destination is therefore between `0x11` and `0x110`, inclusive.

Unsigned operations have 32-bit bit-vector semantics. Some other products and left shifts are represented with signed 32-bit intermediates. Portable signed-overflow behavior is not established. Address formulas involving those intermediates are conditional on the applicable machine arithmetic and address model. Byte order and alignment requirements are unknown.

## Behavior

### Ordering and ledger conventions

The visible path has no explicit branch, loop, validation, bounds check, or error-dependent continuation.

The following ledger covers every represented dependency occurrence and places it relative to direct memory effects. Rows execute in order; calls within a row execute in the order listed, followed by that row’s memory events unless a read is expressly placed before a call.

Dependency IDs have the form **F-992ceab63adf-Drr.k**, where `rr` is the ledger row and `k` is the invocation’s one-based position within that row. These are opaque occurrence IDs, not verified callee identities.

Two abbreviations keep the ledger readable:

- **Four-call block:** four consecutive invocations with no explicit arguments shown. If the block “supplies” a value, its fourth invocation supplies that value. Earlier results have no visible use.
- **Forwarding pair:** one invocation with no explicit arguments shown supplies a 32-bit word; the immediately following invocation receives that word as its sole explicit argument.

These abbreviations specify invocation counts and order; they do not imply shared targets or purity. “No explicit arguments shown” does **not** mean an empty effective ABI interface.

Except where identified as 64-bit, consumed primary dependency results are 32 bits. Each index mentioned below is a new result unless expressly retained.

### Opening and first payload

| Row | Dependency invocations, in order | Direct memory events |
|---|---|---|
| 01 | First invocation receives the second argument. Second supplies a 64-bit result. Third receives its low word, high word, and low word. A four-call block supplies an index. | Store the ambient incoming word at the index’s word location with offset `0x14`. |
| 02 | One invocation receives four times the preceding index; a four-call block follows. | Store \(0x80000Q\) at \(0x200000Q+0x14\). |
| 03 | One invocation receives \(0x80000Q\) three times; a four-call block supplies an index. | Store \(Q\) at that index’s word location with offset `0x14`. |
| 04 | One invocation receives four times that index; a four-call block supplies \(P\). | Read a selector byte at \(Q+1\); write \(L(P)\) at the selected destination. |
| 05 | One invocation receives the preceding selector shifted right by three, \(4P\), and \(4Q\); a four-call block supplies an index. | Reread the byte at \(Q+1\), retaining it as \(S\); write \(U(P)\) at \(S+0x11\). |
| 06 | One invocation receives four times the row-05 index; a four-call block follows. | Store retained \(4P\) at \(4S+0x14\). |
| 07 | One invocation receives \(S,S,S,S\gg1\); a four-call block supplies an index. | Store retained \(4P\) at that index’s word location with offset `0x14`. |

The two reads at \(Q+1\) are distinct. Calls, aliasing writes, or external activity may change the second result. Later uses of \(S\) use the retained second result, not a further read.

### Intermediate word construction

The result names \(R_1,R_2,R_3\) below identify three successive dependency-produced values.

| Row | Dependency invocations, in order | Direct memory events |
|---|---|---|
| 08 | One invocation receives four times the row-07 index; a four-call block supplies \(R_1\). | Store \(4R_1\) at \(16R_1+0x14\). |
| 09 | One invocation receives \(4R_1\); a four-call block supplies an index. | Store retained \(4R_1\) at that index’s word location with offset `0x14`. |
| 10 | One invocation receives four times that index; a four-call block supplies \(R_2\). | Store \(4R_2\) at \(16R_2+0x14\). |
| 11 | One invocation receives \(4R_2\); a forwarding pair follows; a four-call block supplies an index. | Store \(((4R_2)\mathbin{\&}0x3fffffff)\gg1\) at that index’s word location with offset `0x14`. |
| 12 | One invocation receives four times that index; a four-call block supplies \(R_3\). | Store \(4R_3\) at \(16R_3+0x14\). |
| 13 | First read a word at \(4R_3+0x14\). One invocation receives that word logically shifted right by three; a four-call block supplies an index. | Store retained \(4R_3\) at that index’s word location with offset `0x14`. |
| 14 | One invocation receives four times that index; a four-call block supplies \(M\). | Store \(S\) at \(0x80000(S\gg3)+0x24\). |

Row 11 includes both the result-producing invocation and its forwarding invocation. They intervene between the record-scaled write and the later masked indexed write.

The row-13 read is not generally a readback of row 12’s store: the respective addresses are \(4R_3+0x14\) and \(16R_3+0x14\). Their equality requires particular values or address wrapping.

### Main payload and byte clearing

Let \(Z\) denote the retained shift source supplied in row 16.

| Row | Dependency invocations, in order | Direct memory events |
|---|---|---|
| 15 | One invocation receives \(16M,4M\); a four-call block follows. | Store \(S\) at \(16M+0x24\). |
| 16 | First read a word at \(16M+0x74\). One invocation receives that word; a four-call block supplies \(Z\). | Read a selector byte at \(16M+1\); write \(U(M)\) at its selected destination. |
| 17 | One invocation receives \(4Z\); a forwarding pair follows; a four-call block supplies an index. | Reread the selector byte at \(16M+1\); write zero at its selected destination. |
| 18 | One invocation receives four times the row-17 index; a four-call block follows. | Clear the byte at \(H+0x0f\). Then read the selector byte at \(0x400H+1\) and clear its selected destination. |
| 19 | One invocation receives \(Z\ll23\) three times and \(Z\ll25\) as its fourth argument; a forwarding pair follows; a four-call block supplies an index. | Reread the byte at \(0x400H+1\), retaining it as the clearing selector; clear its selected destination. |
| 20 | One invocation receives four times the row-19 index; a four-call block follows. | Store a zero word at clearing selector plus `0x44`; immediately store a zero word at absolute address `0x24`. |

Row 18 has no dependency invocation between its two byte-clearing operations. Nevertheless, the first write precedes the second selector read and may affect it through aliasing.

The first two main-payload selector reads are also separate reads. Clearing need not target the earlier nonzero write.

### Remaining word clears

| Row | Dependency invocations, in order | Direct memory events |
|---|---|---|
| 21 | One invocation receives \(0,0x98,0x400H,0x98\); two consecutive forwarding pairs follow; a four-call block supplies an index. | Clear a word at that index’s word location with offset `0x44`; immediately clear the word at absolute address `0x24`. |
| 22 | One invocation receives four times the row-21 index; a four-call block supplies a 64-bit result. One further invocation receives that result’s low word shifted left by two. | Clear a word at twice the retained high word plus `0x44`; immediately clear the word at absolute address `0x24`. |
| 23 | One invocation receives zero, the secondary register result from row 22’s final invocation, and that secondary result shifted left by two. One invocation with no explicit arguments follows. One receives \(T_7\). A four-call block supplies an index. | Clear a word at that index’s word location with offset `0x44`; immediately clear the word at absolute address `0x24`. |
| 24 | One invocation receives four times the row-23 index; a four-call block supplies a value. Another invocation receives four times that value and supplies a new index. | Clear a word at the new index’s word location with offset `0x14`. |
| 25 | One invocation receives the row-24 index three times and \(T_7\) as its fourth argument; a four-call block supplies an index. | Clear a word at that index’s word location with offset `0x14`. |

The four absolute-address clears occur in rows 20–23. They are distinct operations, not one logical initialization. Within each pair, the selected-location clear comes first.

The constant `0x98` has no established semantic interpretation such as size or length.

### Final word construction and byte writes

Let \(R_4,R_5\) be the next record-producing values; let \(A,B\) be the final two byte payloads.

| Row | Dependency invocations, in order | Direct memory events |
|---|---|---|
| 26 | One invocation receives four times the row-25 index; a four-call block supplies \(R_4\). | Store \(4R_4\) at \(16R_4+0x14\). |
| 27 | One invocation receives \(4R_4\); a four-call block supplies an index. | Store retained \(4R_4\) at that index’s word location with offset `0x14`. |
| 28 | One invocation receives four times that index; a four-call block supplies \(R_5\). | Store \(4R_5\) at \(16R_5+0x14\). |
| 29 | One invocation receives \(4R_5,16R_5\). One with no explicit arguments supplies a word. The next receives that word and \(T_4\). A four-call block supplies an index. | Store \(T_7\) at that index’s word location with offset `0x14`. |
| 30 | One invocation receives four times the row-29 index; a four-call block supplies \(A\). | Read a selector byte at \(0x4000000H+1\); write \(L(A)\) at its selected destination. |
| 31 | One invocation receives \(4A\); a forwarding pair follows; a four-call block supplies an index. | Reread the selector byte at \(0x4000000H+1\); write \(U(A)\) at its selected destination. |
| 32 | One invocation receives four times the row-31 index; a four-call block supplies \(B\). | Reread the selector byte at \(0x4000000H+1\), retaining its value; write \(L(B)\) at its selected destination. |
| 33 | One invocation receives the selector retained in row 32; a forwarding pair follows; a four-call block supplies an index. | Read a selector byte at \(8B+1\); write \(U(B)\) at its selected destination. |
| 34 | One invocation receives four times the row-33 index; a four-call block follows. | No further direct memory effect is represented before the invalid-instruction endpoint. |

Row 29’s stored quantity is also represented as \((H\mathbin{\&}0x3fffff)\times0x80\), which equals \(T_7\).

Rows 31 and 32 preserve the distinction between forwarding \(4A\) and forwarding four times an intervening dependency result. Row 33’s upper-fragment byte and selector address use retained \(B\), not its intervening index result.

## Side effects

### Memory, aliasing, and state

Direct accesses comprise:

- Word writes at indexed and record-scaled locations.
- Word reads at \(4R_3+0x14\) and \(16M+0x74\).
- Byte selector reads at the addresses specified above.
- Selected byte writes, including zero writes.
- Four separate word clears at absolute address `0x24`.

No non-aliasing guarantee is present. Byte writes may modify later selectors or parts of words. Word writes may modify later-read bytes. Calls may change memory between accesses. Cross-width consequences depend on byte order and mapping.

Retained values remain retained in the representation; this does not establish a general ABI preservation guarantee for arbitrary registers or globals.

### Ownership and errors

No allocation, release, reference counting, ownership transfer, or lifetime rule is established. A dependency argument is not necessarily a pointer, size, or owned resource.

Every reached access requires an accessible address and every reached invocation requires a valid executable target with compatible effective ABI requirements. No checks establish these conditions.

No failure status, rollback, cleanup path, exception translation, or recovery mechanism is visible. Faults or nonreturning dependencies may leave earlier effects observable.

## Dependencies

All dependencies are unresolved. The ledger’s occurrence IDs specify known invocation ordering, explicit argument lists, consumed primary results, and the one consumed secondary-register result.

Different IDs do not prove different callees. Similar invocation constructions do not prove a shared callee.

Dependency internals, hidden arguments, global preservation, memory effects, ownership behavior, errors, and termination properties remain **documented_only**. Unused primary results are not evidence that an invocation has no effects.

## Acceptance examples

These are conditional analytical criteria, not executed tests.

1. **Quotient boundary.** Fourth arguments zero through seven yield \(Q=0\): row 02 writes zero at `0x14`. Arguments eight through fifteen yield \(Q=1\): it writes `0x80000` at `0x200014`.

2. **Logical shift.** A fourth argument of `0xffffffff` yields \(Q=0x1fffffff\), not a sign-extended negative result.

3. **Fresh selectors.** If row 04 reads five and row 05 reads nine, their byte destinations are `0x16` and `0x1a`. Subsequent uses of \(S\) use nine.

4. **Byte truncation.** For payload `0x12345678`, \(L(x)=0xe0\) and \(U(x)=0x23\). The upper-fragment write does not store the full ten-bit shifted intermediate.

5. **Intermediate forwarding.** After row 10’s store, row 11 must invoke a dependency with \(4R_2\), obtain a word from another invocation, and forward that word before the four-call block and masked indexed write.

6. **Final-payload call order.** After row 30’s byte write, row 31 first passes \(4A\), then executes the result-producing and forwarding invocations, then its four-call block. After row 31’s byte write, row 32 passes four times row 31’s index—not \(4A\).

7. **Aliasing-sensitive clearing.** If row 18’s first byte clear changes its subsequent selector location, the following selected clear uses the changed byte.

8. **Repeated absolute clears.** A trace reaching rows 20–23 contains four separate word-zero writes to `0x24`, each after its paired selected-location write.

9. **Arithmetic bounds.** \(H\leq0x1fffffff\), \(T_7\leq0x1fffff80\), and \(T_4\leq0x03fffff0\). Under 32-bit unsigned arithmetic, \(0x4000000H\) depends only on \(H\)’s low six bits.

10. **Endpoint.** Reaching row 33’s byte write does not establish success: row 34’s five invocations and the invalid-instruction endpoint still follow.

## Uncertainties

The following remain **documented_only**:

- Reliability of instruction recovery and function boundaries.
- Exact ABI, target execution state, endianness, alignment, and address mapping.
- Machine behavior for signed arithmetic and shifts outside portable ranges.
- Meaning of low absolute addresses and overlapping memory symbols.
- Initialization of ambient register inputs and additional hidden ABI state.
- Dependency identities, contracts, nonlocal control effects, and memory effects.
- Whether the terminal invalid-instruction boundary is executable or reflects incomplete recovery.
- Semantic object layouts, concurrency guarantees, resource ownership, and higher-level purpose.

These unknowns do not erase the explicitly represented invocation arguments or their ordering.

## Evidence

Established static evidence consists of the supplied representation’s operand widths, explicit invocation arguments, result forwarding, address calculations, direct memory accesses, sequencing, and truncated endpoint.

The supplied representation warns of invalid instruction data and overlapping global symbols. The recovered contract is therefore conditional on representation accuracy.

No implementation access, independent instruction analysis, runtime execution, smoke run, or test was performed. Acceptance examples are static deductions under their stated conditions, not observed results.
