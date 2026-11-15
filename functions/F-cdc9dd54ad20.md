<!-- Trusted provenance: F-cdc9dd54ad20; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cdc9dd54ad20

## Purpose

The supplied static representation describes a routine that interleaves computed-address memory writes, a small number of memory reads, and many indirect calls, then enters an unconditional repeating call-and-write sequence.

No application-level purpose is established. The evidence does not justify identifying the routine as a serializer, allocator, device driver, decoder, or protocol handler. Its observable contract is therefore expressed in terms of memory accesses, call ordering, retained values, and nontermination.

The representation carries warnings about invalid instruction data, removed unreachable regions, and overlapping global symbols. Consequently, the behavior below describes the supplied representation, not a validated reconstruction of executable behavior.

## Interface

### Explicit inputs

| Input | Represented width | Established use |
|---|---:|---|
| First input | Unsigned 32 bits | Initial address generation, a stored right-shifted value, initial call arguments, and two byte-address selector reads |
| Second input | Opaque 32 bits | No explicit use appears |
| Third input | Unsigned 32 bits | Initial call argument and selection of a word destination using its upper 25 bits |

The routine has no represented return value. The displayed control flow has no normal return: after its finite prefix, it repeats indefinitely if every call and memory operation completes.

The second input must not be described as behaviorally irrelevant at the machine interface. Its lack of explicit use does not establish what indirect callees can observe through registers or other calling-context state.

### Additional execution context

The first indirect-call destination depends on an incoming link-related register rather than solely on the three explicit inputs. A later stored value depends on a secondary register output whose producing-call contract is unavailable.

These are material hidden dependencies. A contract based only on the three explicit arguments cannot fully determine execution.

### Memory notation and widths

For this document:

- A **word access** is a represented 32-bit load or store.
- A **byte access** is an 8-bit load or store.
- A **selector destination** derived from address \(a\) means: read one unsigned byte at \(a\), then add hexadecimal \(0x11\) to that byte’s value to obtain the byte-store address.
- A **four-scale slot** for value \(x\) is address \(4x+0x14\).
- A **sixteen-scale slot** for value \(x\) is address \(16x+0x14\).

These names describe address arithmetic only. They do not establish arrays, object boundaries, ownership, or valid address ranges.

Unsigned shifts are logical. A byte store retains only the low eight bits of its source value. Endianness is not established; individual bytes of word stores cannot be specified.

Arithmetic involving unsigned 32-bit values is interpreted at that width. Some scaling and shifting operations are represented through signed 32-bit intermediates. Their behavior outside nonoverflowing ranges is **documented_only**: the supplied material does not establish a portable overflow rule or a verified machine-level replacement.

## Behavior

### Execution model and ordering

The finite prefix contains a long sequence of indirect calls. Most explicitly argument-bearing calls are separated from the next visible memory operation by several further calls.

Those intervening calls are observable dependency boundaries, not removable padding. They may modify memory, terminate execution, fail, or return values used in later calculations. No preservation of global state, selector bytes, or destination contents is assumed.

The description below records the order of direct memory effects and the important explicit argument relationships. It does not assert that memory remains unchanged between these effects.

### 1. Initial word stores

The routine first stores the first input logically shifted right by seven into its four-scale slot.

Only after this store does it invoke F-indirect-entry, explicitly passing, in order:

1. The first input.
2. The first input again.
3. The third input.
4. The first input again.

After a sequence of indirect calls, a returned 32-bit quantity determines another four-scale slot. The routine stores the same retained first-input-right-shift-by-seven value there, then passes the returned quantity scaled by four to a further indirect call.

These two stores need not have distinct destinations. Neither store is conditional on a validity check.

### 2. First pair of selector-directed byte writes

After additional calls, the routine retains a returned unsigned word as the **first byte payload**.

It then:

1. Reads one byte at address \((\text{first input} \mathbin{\gg} 8)+1\).
2. Writes the low byte of the first byte payload shifted left by two to the resulting selector destination.
3. Calls an indirect dependency with the full shifted value.
4. Executes additional indirect calls.
5. Reads the selector byte again from the same selector-source address.
6. Writes the low byte of \((\text{first byte payload}\mathbin{\&}0x3fffffff)\mathbin{\gg}20\) to the newly selected destination.

The second selector read is not a cached reuse of the first. Intervening calls may redirect the second store.

The second byte retains original payload bits 20 through 27. Although the masked shift can produce ten bits, the byte store truncates the result.

A separate returned quantity, shifted left by two, is passed to the call following the second byte write.

### 3. Global-derived repeated word stores

The routine next reads a global value represented at absolute address \(0x3c\), converts it to an unsigned word, and retains that value.

It computes the destination:

\[
4(\text{third input}\mathbin{\gg}7)+0x14.
\]

It stores the retained global-derived value there, executes intervening calls, and stores that same retained value there again.

The first call after the first store explicitly receives the global-derived value. The first call after the second store receives a separately returned quantity shifted left by two.

The global is not explicitly reread for the second store. Its original access width and signedness are not established by the provided global declaration.

### 4. Word snapshots and scaled-address writes

The following table describes the next direct memory operations in order. Each named result is a fresh returned quantity at the relevant stage, not an input object or stable memory location.

| Stage | Direct memory behavior and retained data |
|---|---|
| First snapshot | For result \(A\), store \(4A\) at \(16A+0x14\), then read a word at \(4A+0x14\). Retain that read as the **first snapshot**. Also retain the unsigned quantity represented by \(A\) shifted left by 26. |
| First repeated store | After intervening calls, store \(4A\) at \(16A+0x14\) again. |
| Second snapshot | For fresh result \(B\), store \(4B\) at \(16B+0x14\), then read a word at \(4B+0x14\). Retain that read as the **second snapshot**. |
| Shifted second snapshot | After intervening calls, store the second snapshot logically shifted right by eight at \(4B+0x14\). |
| First self-value stage | For fresh result \(C\), store \(4C\) at \(16C+0x14\). After calls, store \(4C\) at \(4C+0x14\). |
| Second repeated scaled store | For fresh result \(D\), store \(4D\) at \(16D+0x14\), then repeat that same store after further calls. |
| Snapshot-derived address | For fresh result \(E\), store \(4E\) at \(16E+0x14\). Define the **snapshot-derived base** as eight times the first snapshot logically shifted right by eight. After calls, store \(4E\) at that base plus \(0x14\). |
| Snapshot-derived self-value | For fresh result \(F\), store \(4F\) at \(16F+0x14\). After calls, store the snapshot-derived base at that base plus \(0x14\). |

Each snapshot load follows its corresponding store without an intervening call in the supplied representation. Nevertheless, the store and load addresses can coincide, so the load is not necessarily a read of preexisting data.

Across these stages, explicit call arguments include:

- Scaled results used in the table.
- The first snapshot shifted right by 16.
- The snapshot-derived base, sometimes supplied twice in the same call.
- An unsigned value initially formed by shifting \(A\) left by 26 and subsequently shifted right by seven on three separate occasions.

The latter three right shifts act on an already width-limited value. They are not equivalent to shifting an unlimited-precision version of \(A\).

The first snapshot remains retained across the intervening calls; it is not explicitly reloaded.

### 5. Secondary-register-derived stores

After the snapshot-derived self-value stage and more calls, a secondary register output supplies a quantity used to form:

- A value scaled by \(0x01000000\).
- A destination scaled by \(0x04000000\), then increased by \(0x14\).

The routine stores the former at the latter.

After further calls, it stores that same retained scaled value into a four-scale slot selected by a new returned quantity.

The secondary register is not one of the three explicit inputs. Its exact producing dependency and signed-overflow behavior are unresolved.

### 6. Additional scaled stores and a narrowed snapshot value

For a new returned result, the routine writes its fourfold value to its sixteen-scale slot. After calls, that retained fourfold value is written to a four-scale slot selected by another result.

A subsequent result supplies another fourfold quantity, called the **retained scaled quantity**. The routine stores it to the corresponding sixteen-scale slot.

After additional calls, it writes the first snapshot shifted right by 24 to a four-scale slot selected by a fresh result. This is a 32-bit store of a value between zero and 255, not a byte store.

Next, for a fresh result \(G\), it stores \(4G\) at \(16G+0x14\). After calls, it replaces that word with \(16G\).

The explicit call between those two stores receives the sixteenfold value twice and the first snapshot shifted right by 23 as its third shown argument.

### 7. Later selector writes and clearing

The routine derives a **later selector base**:

\[
(\text{retained scaled quantity}\mathbin{\&}0x7fffffff)\mathbin{\gg}7.
\]

Using fresh call results, it performs the following ordered direct effects:

1. Read a selector byte at the later selector base plus one.
2. Write the low byte of a new returned payload shifted left by two to that selector destination.
3. Execute indirect calls, including one receiving that shifted payload in all three shown argument positions.
4. Reread the selector byte.
5. Write the low byte of the same payload masked by \(0x3fffffff\) and shifted right by 20.
6. Execute further calls and obtain another payload.
7. Store that new payload shifted left by two, truncated to a byte, at \((\text{first snapshot}\mathbin{\gg}31)+0x1c\).
8. Reread the later selector byte and store zero to its selected destination.
9. Execute more calls.
10. Reread that selector byte again and store zero to its selected destination.

The direct destination in step 7 is therefore either \(0x1c\) or \(0x1d\).

One intervening call receives the first snapshot shifted right by 31. Another receives the newest payload masked by \(0x3fffffff\), then shifted right by six. Neither argument is interchangeable with the low byte stored in step 7.

### 8. Entry into the repeating tail

A further returned word is masked with \(0x3fffffff\) and shifted right by 25. Call this the **initial loop quantity**. Its range is zero through 31.

Before entering the loop, the routine:

1. Reads a selector byte at one plus the step-7 shifted payload multiplied by \(0x400\).
2. Stores zero to that selector destination.
3. Calls F-indirect-tail-setup with twice the initial loop quantity.
4. Makes another indirect call whose returned quantity seeds the loop.

The multiplication used to form the selector-source address has unresolved signed-overflow semantics where the represented signed intermediate overflows. The source payload is retained; the selector byte is not.

### 9. Unconditional repeating tail

Every represented iteration:

- Invokes an indirect dependency with the current call-derived quantity and the current loop quantity shifted right by eight.
- Executes additional indirect calls.
- Rereads the selector byte from the same tail selector-source address.
- Stores zero to the selected destination.
- Calls another dependency with a fresh returned quantity shifted left by two.
- Executes additional indirect calls.
- Uses the final returned quantity to derive the next call-derived quantity by shifting left by two.
- Uses that same result shifted left by 26 to replace the loop quantity.

There is no represented exit test.

On the first iteration only, the second argument of the first call is necessarily zero, because the initial loop quantity is at most 31. Later iterations do not inherit that restriction.

## Side effects

Direct side effects comprise word stores, byte stores, selector-byte reads, two retained word snapshots, and the global-derived read. Indirect dependencies may add arbitrary effects; their contracts are unavailable.

The routine performs no represented validation, error reporting, rollback, allocation, or release. This does not establish that dependencies perform none of those actions.

No ownership transfer is established. Scaled values and loaded words must not be treated as owned pointers merely because they participate in address calculations.

Successful execution requires every reached memory access and indirect call to be valid in the execution environment. Fault behavior, alignment enforcement, memory-mapped I/O semantics, and exception propagation are **documented_only**.

## Dependencies

The following are document-local opaque dependency IDs, not resolved implementation identities:

| ID | Role |
|---|---|
| F-indirect-entry | First call; target depends on incoming execution context |
| F-indirect-chain | Ordered collection of remaining unresolved indirect call sites |
| F-indirect-tail-setup | Call receiving twice the initial loop quantity |
| F-indirect-tail-cycle | Repeated tail call sites |

F-indirect-chain does not imply that its call sites share one target or contract. Calls displayed without explicit arguments must not be assumed to receive no machine-level arguments.

The supplied dependency map provides no independently specified callee behavior.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

1. **Initial scaling.** With first input \(0x100\), the first direct store is a 32-bit value of 2 at address \(0x414\), before F-indirect-entry runs.

2. **Third-input boundary.** Third inputs 127 and 128 select global-derived word destinations \(0x14\) and \(0x18\), respectively. This does not imply equal stored values across separate executions.

3. **Selector reread.** If the first selector read yields 3 and intervening calls change it to 9, the two initial payload-byte stores target \(0x14\) and \(0x1a\), respectively.

4. **Byte truncation.** A first byte payload of \(0x12345678\) produces initial byte values \(0xe0\) and \(0x23\), provided both writes are reached.

5. **Snapshot aliasing.** If the first snapshot result is zero, its preceding store writes zero at \(0x14\), and its immediately following load reads \(0x14\). For ordinary stable memory, the snapshot is therefore zero.

6. **Narrowed word store.** A first snapshot of \(0xab123456\) leads to a later 32-bit store of \(0x000000ab\). It does not justify a one-byte-only update.

7. **Tail evolution.** An initial loop quantity of 31 gives first-iteration second argument zero. If a later tail result is 1, the next loop quantity is \(0x04000000\), whose right shift by eight is \(0x00040000\).

8. **Nontermination.** If all calls return and all accesses complete indefinitely, the routine continues issuing tail calls and zero-byte stores without normal return.

## Uncertainties

The following remain **documented_only**:

- Actual executable control flow where the representation reports invalid instruction data or removed regions.
- Concrete indirect targets, calling conventions, and complete argument sets.
- Secondary-register result provenance.
- Global read width, signedness, volatility, and overlapping-symbol interpretation.
- Overflow semantics of signed intermediate operations.
- Endianness, alignment requirements, memory validity, and hardware effects.
- Dependency failures, nonlocal exits, memory ownership, and concurrency behavior.

Inferred intent is limited to an apparent staging of scaled values and selected byte fields. No higher-level interpretation is required by the evidence.

## Evidence

Established static evidence consists of the supplied routine representation: three explicit inputs, direct memory accesses, visible arithmetic, indirect-call order, retained values, and an unconditional repeating tail.

The warnings reduce confidence in correspondence to executable behavior. They do not authorize inventing missing branches, safe memory regions, dependency contracts, or termination conditions.

No runtime tests, executable inspection, or dependency verification have been performed.
