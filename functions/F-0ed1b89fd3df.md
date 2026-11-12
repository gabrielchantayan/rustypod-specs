<!-- Trusted provenance: F-0ed1b89fd3df; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ed1b89fd3df

## Purpose

The supplied static representation describes a routine that performs calculated memory stores, invokes unresolved indirect dependencies, reads bytes to select further write destinations, and enters an unconditional loop.

No application-level purpose is established. The evidence does not justify identifying this routine as a decoder, allocator, initializer, serializer, or device driver.

This contract describes the explicit operations and data dependencies in the supplied representation. Execution is conditional on valid arithmetic, accessible memory, callable targets, and returning dependencies. Warnings about invalid instruction data and removed unreachable regions limit confidence that the represented flow corresponds to executable behavior.

## Interface

### Explicit inputs

| Input | Represented width and interpretation | Visible use |
|---|---|---|
| First argument | 32-bit signed integer | Initial address calculation, two word-store values, and the first indirect call’s argument |
| Second argument | Opaque 32-bit value | No explicit use |
| Third argument | 32-bit unsigned integer | Logical right shifts, a word-store value, a byte-store address, and dependency arguments |

The absence of an explicit use of the second argument does not establish its irrelevance under the actual calling convention.

### Implicit inputs

The routine depends on:

- Incoming return-link state, which contributes to the first indirect-call target.
- Target-base values established within the represented flow.
- Results of unresolved dependencies.
- Bytes read from calculated addresses.
- Memory accessibility, alignment rules, and the effects of dependencies or external agents.

No explicit buffer length, capacity, ownership token, or error-output parameter is identifiable.

### Output and termination

The represented interface returns no value. No normal return is visible.

If execution reaches the final loop, the represented control flow contains no exit condition. A dependency may fail to return, fault, trap, or transfer control elsewhere; those behaviors are unresolved rather than normal termination guaranteed by this routine.

There is no visible error code, validation failure branch, exception translation, or recovery path.

### Widths and arithmetic

Word stores are 32 bits. Byte reads yield unsigned 8-bit values. Byte stores retain only the low eight bits of their value.

Unsigned shifts and masks operate on 32-bit quantities. Several scaled addresses and left shifts instead originate from signed 32-bit expressions. The supplied material does not establish a signed-overflow policy or the executable’s address-arithmetic rules.

Accordingly:

- Unsigned masks and logical right shifts have the stated 32-bit interpretation.
- Descriptions involving wrapping multiplication or left shifts are conditional on ordinary 32-bit target arithmetic.
- Such descriptions do not assert that overflowing or negative signed shifts have defined source-language semantics.
- Byte order is unknown. A word store specifies a 32-bit value, not a particular ordering of its four constituent bytes.

To execute the complete represented path, every accessed address must permit the specified operation, word accesses must satisfy applicable alignment requirements, each indirect target must be callable, and every dependency preceding later behavior must return. The routine visibly checks none of these conditions.

## Behavior

### Contract notation

The following names describe calculations, not established objects or private implementation identifiers:

- **Word location W(x):** address 20 plus four times x.
- **Record location R(x):** address 20 plus sixteen times x.
- **Selected destination S(q):** read one unsigned byte at address q plus 1, then add 17 to that byte’s value.

Each use of S entails a fresh memory read. Its destination lies between addresses 17 and 272 inclusive. The selector is a byte, not a pointer-sized value.

W and R do not establish array bounds, a common allocation, or a record layout beyond the displayed address calculations.

Letters A through N below are contract labels for particular dependency results. They do not identify executable functions or imply ownership. The source represents H, J, and L as unsigned 32-bit results; A through G, I, K, M, and N have signed 32-bit result uses. Some later transformations explicitly convert or retain values as unsigned quantities.

All ordering below is conditional on control reaching the relevant operation.

### Initial word stores

The first explicit memory operation stores the first argument at W(first argument).

The routine obtains the third argument logically shifted right by 8, called **upper input**.

Five calls, F-U001 through F-U005, follow. Only F-U001 has a displayed argument: the first argument. F-U005 produces A.

The routine then:

1. Stores the first argument at W(A).
2. Executes F-U006 through F-U010. F-U006 receives four times A; the other four calls have no displayed arguments.
3. Retains F-U010’s result as B.
4. Retains four times B as the **first scaled result**.
5. Stores the first scaled result at R(B).

There is no bounds test, negative-index rejection, or validation before these stores.

### Further word stores and forwarding calls

F-U011 through F-U019 form a nine-call sequence:

- F-U011 receives the first scaled result.
- F-U012 produces a 32-bit opaque value explicitly forwarded to F-U013.
- F-U014 produces another such value explicitly forwarded to F-U015.
- F-U016 through F-U019 have no displayed arguments.
- F-U019 produces C.

The routine stores the retained first scaled result at W(C).

F-U020 through F-U024 then execute. F-U020 receives four times C; the others have no displayed arguments. F-U024 produces D.

Four times D is retained as an unsigned 32-bit **second scaled result**, subject to the signed-arithmetic qualification above. That value is stored at R(D).

F-U025 through F-U033 repeat the nine-call argument pattern:

- The first receives the second scaled result.
- The second result is forwarded to the third.
- The fourth result is forwarded to the fifth.
- The remaining four calls have no displayed arguments.
- The ninth produces E.

Upper input is then stored at W(E).

F-U034 through F-U038 follow, with four times E supplied to F-U034 and no displayed arguments for the remainder. F-U038 produces F.

The routine retains sixteen times F as the **record base**, then stores four times F at record base plus 20.

It also derives the **selector offset** by clearing bit 31 of the retained second scaled result and logically shifting right by 7. This derives from the retained value, not a reload of the earlier store. The selector offset lies between 0 and 16,777,215 inclusive; its selector-read address is therefore between 1 and 16,777,216 inclusive.

### Record overwrite and H-derived byte writes

F-U039 through F-U043 execute between two writes to the same record location:

- F-U039 receives three displayed arguments: record base, record base, and the third argument logically shifted right by 7.
- F-U040 through F-U043 have no displayed arguments.
- F-U043 produces G.

After those calls return, the routine overwrites the word at record base plus 20 with record base itself.

The first write of four times F and the later write of sixteen times F are distinct observable operations. The intervening dependencies may observe or alter memory. Collapsing them into a single final-state assignment would not preserve the represented sequence.

F-U044 through F-U048 then execute. F-U044 receives G shifted left by 2; the others have no displayed arguments. F-U048 produces unsigned result H.

The routine reads the byte at selector offset plus 1 and writes the low byte of H shifted left by 2 to the resulting selected destination. This exposes H’s bits 0 through 5 in destination bits 2 through 7 and writes zero in destination bits 0 and 1.

F-U049 through F-U055 follow:

- F-U049 receives H shifted left by 2 in all three displayed argument positions.
- F-U050’s result is forwarded to F-U051.
- F-U052 through F-U055 have no displayed arguments.
- F-U055 produces I.

The routine reads the byte at selector offset plus 1 again. At the newly selected destination, it writes the low byte of H after retaining its low 30 bits and logically shifting right by 20.

The stored value consists of H’s bits 20 through 27. Bits 28 and 29 survive the mask but not the final byte narrowing.

The two selector reads are independent. An intervening dependency or the first write itself may change the selector, so the destinations need not match.

### Input-derived byte write and zero writes

F-U056 through F-U060 execute next. F-U056 receives I shifted left by 2; the remaining calls have no displayed arguments. F-U060 produces unsigned result J.

The routine retains J shifted left by 2, converted to a signed 32-bit value in the supplied representation. Under ordinary bit-preserving 32-bit behavior, this is the corresponding low-order 32-bit pattern.

Two byte writes then occur in this order:

1. Write the low byte of J shifted left by 2 at address 28 plus upper input logically shifted right by 7.
2. Read the selector byte at selector offset plus 1 and write zero at that byte’s value plus 17.

The first address is equivalently 28 plus the third argument logically shifted right by 15. Its range is 28 through 131,099 inclusive.

F-U061 through F-U067 follow:

- F-U061 receives upper input logically shifted right by 7.
- F-U062 has no displayed arguments.
- F-U063 receives J masked to its low 30 bits and logically shifted right by 6.
- F-U064 through F-U067 have no displayed arguments.
- F-U067 produces K.

The routine freshly reads the selector byte at selector offset plus 1 and writes zero at the newly selected destination.

F-U068 through F-U072 then execute. F-U068 receives K shifted left by 2; the remaining four calls have no displayed arguments. F-U072 produces unsigned result L.

The **initial loop quantity** is L masked to its low 30 bits and logically shifted right by 25. It is between 0 and 31 inclusive.

The routine next performs a different selector-dependent zero write:

- Its selector-read address is the retained J-shifted-left-by-2 value, multiplied by 1,024, plus 1.
- It reads one unsigned byte there.
- It writes zero at that byte’s value plus 17.

Under 32-bit wrapping arithmetic, the selector-read address is equivalently J shifted left by 12, plus 1. This equivalence is conditional because the represented multiplication uses a signed value.

F-U073 receives twice the initial loop quantity. F-U074 has no displayed arguments and produces M.

### Loop behavior and target transition

The loop executes at least once if reached. Each iteration contains ten indirect calls and one selector-dependent zero write.

F-U075 through F-U079 begin the iteration:

- F-U075 receives the current **carried result** and the current loop quantity logically shifted right by 8.
- F-U076 through F-U079 have no displayed arguments.
- F-U079’s result supplies the next group’s first argument.

On the first iteration, the carried result is M and the loop quantity is the initial loop quantity. Therefore F-U075’s second displayed argument is zero on that iteration.

**F-U075 has an explicitly represented target transition.** Let F-T001 identify its first-entry target and F-T002 identify its recurring target:

- The target-base value established immediately before F-U074 selects F-T001 for the first execution of F-U075.
- A different target-base value is established near the end of each iteration, immediately before F-U084.
- That later base selects F-T002 for F-U075 on the next iteration.
- The same additive target displacement is applied to these distinct bases. Thus the represented change is in target selection, not merely in call arguments.

These labels describe two static target selections; their executable validity and actual dependency identities remain unresolved.

After F-U079, the routine freshly reads the J-derived selector described above and writes zero at its selected destination.

F-U080 through F-U084 follow:

- F-U080 receives F-U079’s result shifted left by 2.
- F-U081 through F-U084 have no displayed arguments.
- F-U084 produces N.

The next carried result becomes N shifted left by 2. The next loop quantity becomes N shifted left by 26, retained as an unsigned 32-bit value.

Under 32-bit arithmetic, that loop quantity preserves only N’s low six bits, moved into bits 26 through 31. Its logical right shift by 8 may therefore be nonzero on subsequent iterations.

There is no displayed condition, break, return, or iteration bound.

## Side effects

### Direct memory operations

Before loop entry, the explicit body performs:

| Operation | Count |
|---|---:|
| 32-bit word stores | 8 |
| Byte stores | 6 |
| Byte reads selecting write destinations | 5 |

The five selector-dependent byte writes are:

1. First H-derived write.
2. Second H-derived write.
3. First selector-offset-derived zero write.
4. Second selector-offset-derived zero write.
5. J-derived selector zero write.

Each has its own preceding byte read. The separate input-derived byte write has no selector read and brings the total to six pre-loop byte stores.

Each completed loop iteration adds one fresh J-derived selector read and one zero-valued byte store.

These counts exclude dependency effects. Address aliasing is unrestricted: stores may overlap earlier word stores, selector bytes, or other selected destinations. In particular, a write may affect a later selector read without any dependency intervening.

### Ownership and shared state

No direct allocation, release, ownership transfer, locking, or cleanup operation is established.

A dependency argument is not proven to be a pointer merely because it is address-like. Passing it does not establish ownership transfer or permission to retain or release memory.

No guarantee is available that dependencies preserve globals, prior stores, selector bytes, executable mappings, or resources. The represented local data flow does not establish a general callee-preservation contract.

No atomicity, synchronization, or concurrency protocol is visible.

## Dependencies

All calls are unresolved indirect calls. The following opaque IDs identify call sites, not necessarily distinct functions:

| Call sites | Executions before loop | Final result used here |
|---|---:|---|
| F-U001–F-U005 | 5 | A |
| F-U006–F-U010 | 5 | B |
| F-U011–F-U019 | 9 | C |
| F-U020–F-U024 | 5 | D |
| F-U025–F-U033 | 9 | E |
| F-U034–F-U038 | 5 | F |
| F-U039–F-U043 | 5 | G |
| F-U044–F-U048 | 5 | H |
| F-U049–F-U055 | 7 | I |
| F-U056–F-U060 | 5 | J |
| F-U061–F-U067 | 7 | K |
| F-U068–F-U072 | 5 | L |
| F-U073–F-U074 | 2 | M |
| F-U075–F-U079 | Five per iteration | Intermediate loop result |
| F-U080–F-U084 | Five per iteration | N |

There are 74 calls before loop entry and ten per completed iteration.

F-U001’s target depends on incoming return-link state. Later target expressions use established address-like bases and fixed displacements. F-U075 specifically selects F-T001 on first entry and F-T002 on subsequent iterations as described above.

“No displayed arguments” does not prove that the actual dependency consumes no register or stack inputs. Return conventions, memory effects, resource behavior, error behavior, and target validity remain unresolved.

## Acceptance examples

These are conditional static expectations, not executed tests.

1. **Initial ordering.** If the first argument is 3 and the initial access is permitted, the first explicit memory operation stores the word value 3 at address 32, before F-U001.

2. **Observable overwrite.** If F is 2, address 52 first receives the word value 8. After F-U039 through F-U043 return, it receives 32. Both writes must be retained.

3. **Fresh selector reads.** If a selector read yields 255, that write targets address 272. If the selector becomes 0 before the next read, the next selected write targets address 17.

4. **First H byte.** H equal to 63 produces byte value 252. H equal to 64 produces byte value 0.

5. **Second H byte narrowing.** H equal to 268,435,456 produces byte value 0 for the second H-derived write because bit 28 does not survive byte narrowing.

6. **Input-derived address.** A third argument of 32,768 places the J-derived byte at address 29. A third argument of zero places it at address 28.

7. **Pre-loop byte accounting.** If all pre-loop accesses and calls complete, there are exactly five explicit selector reads and six explicit byte stores. Only five stores use byte-selected destinations.

8. **First-loop target and argument.** If loop entry is reached, the first F-U075 selects F-T001 and receives zero as its second displayed argument.

9. **Subsequent-loop transition.** Under ordinary 32-bit arithmetic, if an iteration obtains N equal to 1 and reaches the next iteration, F-U075 selects F-T002, receives carried result 4, and receives 262,144 as its second displayed argument.

10. **Interrupted execution.** If a dependency does not return, subsequent operations are not established. No visible branch converts that event into a reported error result.

## Uncertainties

### Established static evidence

The supplied representation explicitly supports the operation counts, widths, masks, shifts, argument forwarding, repeated selector reads, record overwrite, unconditional loop, and first-entry versus recurring target selection at F-U075.

### Inferred intent

The repeated call patterns may represent a common underlying mechanism. The scaled word addresses may represent indexed storage. Neither inference establishes an application-level abstraction or additional guarantees.

### documented_only unknowns

- Whether the represented entry point and control flow match executable instructions.
- The impact of the invalid-instruction and removed-region warnings.
- Architecture, byte order, calling convention, signed-overflow behavior, and address conversion rules.
- Validity and accessibility of calculated memory and executable targets.
- Whether accesses denote ordinary memory, device registers, protected regions, or invalid locations.
- Actual identities and behavior of all dependencies, including F-T001 and F-T002.
- Hidden inputs, stack effects, nonlocal transfers, ownership, and dependency preservation guarantees.
- Whether the loop is reachable or any dependency intentionally terminates execution.
- Whether the unused explicit argument matters outside the represented body.

No runtime tests have been performed.

## Evidence

This contract is based solely on the supplied static representation and its warnings. The arithmetic, operation counts, call-site grouping, and target-transition description are static deductions from that material.

No runtime observation, executable validation, resolved dependency contract, or memory-map evidence establishes successful execution or an application-level result. The contract therefore preserves explicit represented behavior while leaving unsupported executable and semantic claims unresolved.
