<!-- Trusted provenance: F-0ebbc37bef74; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0ebbc37bef74

## Purpose

F-0ebbc37bef74 is a parameterless procedure whose behavior depends on incoming machine state, global storage, memory contents, and unresolved indirect calls. Its directly represented effects include word propagation, extraction of high-order bits, nested byte-address selection, byte and halfword clearing, two updates of a global word, and a final attempted call to address zero.

No application-level purpose is established. Identifying it as initialization, serialization, allocation, dispatch, or hardware control would exceed the available evidence.

This contract describes the supplied static representation, conditional on execution reaching each operation. It does not establish that the represented control flow is executable or that any dependency returns normally.

## Interface

### Inputs and output

There are no explicit parameters and no declared return value. A normal return is represented only after the address-zero call; that call must itself return for the procedure to return normally.

Implicit inputs include:

- Incoming link state, used to derive the first call target.
- An incoming 32-bit payload, called **initial payload** below.
- An incoming unsigned 32-bit value, called **extraction input**.
- Primary and secondary results exposed by unresolved calls.
- Two global word locations, **global A** and **global B**.
- Storage at low absolute addresses, including address 1 and a three-byte operand beginning at address 9.
- Memory selected by returned values and derived addresses.

These names identify semantic roles in this contract, not recovered register assignments or an established ABI.

### Widths and address notation

A byte is 8 bits, a halfword 16 bits, a word 32 bits, and a paired result 64 bits. Paired results are divided into low and high 32-bit components. Address-valued calculations are represented using 32-bit quantities.

The following notation describes address selection:

- **word-slot(index)**: address `4 × index + 20`.
- **nested-target(source)**: read an unsigned byte at address **source**, then use that byte’s value plus 17 as the destination address.
- **low-byte(value)**: the least significant eight bits of **value**.

A nested target is therefore always between addresses 17 and 272, provided its source-byte read succeeds. It is not the source address plus 17.

Unsigned right shifts are logical. Unsigned 32-bit arithmetic and left shifts retain the low 32 bits. Some index products and left shifts are represented as signed operations; the evidence does not establish their behavior when signed intermediates overflow. Address examples below avoid that uncertainty.

### Preconditions and failure interface

Every reached memory operation requires accessible storage of the stated width and any alignment required by the actual execution environment. Every reached indirect call requires a callable target under the actual ABI.

No checks enforcing these conditions, error return, recovery path, retry, or rollback are represented. A fault or nonreturning dependency can leave earlier effects in place.

## Behavior

### Ordering and dependency notation

The phases below are ordered. Within each phase, the stated calls, reads, and writes retain their relative order.

**F-D01** through **F-D11** designate ordered groups of unresolved call sites. A group designation does not imply that its calls share a target or implementation. Each occurrence remains a separate observable call.

A call described without arguments has no explicit arguments exposed by the representation; this does not establish that it has no machine-level inputs. Similarly, an unused primary result does not establish that a call is effect-free.

Retained values below describe the representation’s dataflow. They do not impose undocumented register-preservation or global-preservation guarantees on dependencies.

### 1. Initial payload and high-bit extraction — F-D01

The first call target is derived from incoming link state plus `0xF38`. A subsequent call produces a word that is forwarded to the next call. Four further calls follow; the fourth produces the first index.

The procedure writes the initial payload to word-slot(first index), then reads global A into the **first global snapshot**. It calls a dependency with four times the first index, followed by four more calls. The last exposes a secondary result, the **secondary index**.

Two unsigned extractions then occur:

- **high-ten** is extraction input shifted right by 22, hence 0–1023.
- **high-nine** is first global snapshot shifted right by 23, hence 0–511.

The procedure writes high-ten to word-slot(secondary index), then calls with four visible arguments: secondary index twice, high-nine, and high-ten.

The global read is after the initial payload store and before the call receiving the scaled first index. Aliasing or dependency effects can therefore affect the snapshot.

### 2. Repeated extracted value and zero store — F-D02

A call produces a word. Before forwarding that word to the next call, the representation computes **derived-zero** as high-nine shifted right by 23. Because high-nine is at most 511, derived-zero is necessarily zero.

After the forwarding call, four calls follow; the fourth produces an index. High-ten is written to its word-slot.

A call with four times that index precedes four further calls. Their final secondary result is the **table selector**. A zero word is written to word-slot(table selector), followed by a call receiving the unscaled table selector.

### 3. Table capture and nested byte propagation — F-D03

A result-producing call, its forwarding call, and four further calls produce another index. A zero word, expressed as high-nine shifted right by 23, is written to its word-slot.

The next direct read captures a word from:

`((table selector & 0x1FFFFFFF) >> 19) + 20`.

This address is between 20 and 1043 and is not necessarily word-aligned. The preceding zero store remains ordered before this read, including if the accesses overlap.

A call receiving four times the index and four further calls follow. Their final secondary result is retained as the **triple argument**.

The captured table word shifted right by 22 becomes **table-high-ten**, in 0–1023. The procedure reads a byte at derived-zero plus 1—therefore absolute address 1—and writes low-byte(table-high-ten) to the corresponding nested target.

It then calls with triple argument repeated three times and a fourth argument of zero.

### 4. Byte clearing and saved values — F-D04

One call is followed by a call explicitly receiving zero, then four further calls whose last result supplies an index.

The procedure reads a byte at `4 × table-high-ten + 1` and clears its nested target. The source-byte address is between 1 and 4093.

A call receiving the index shifted left by two precedes four further calls. The final result is shifted left by two and supplied twice to another call.

A subsequent call produces the **saved extraction word**, which is forwarded to the next call. One further call exposes a secondary result, the **saved shift value**.

The procedure then reads the byte at absolute address 1 and clears the address selected by that byte plus 17. A call receives the saved shift value three times.

### 5. First global B update and byte refresh — F-D05

A call produces a word. The next call receives that word and saved extraction word shifted right by 23. Four further calls produce an index.

The byte at absolute address 1 is read and retained as the **saved byte selector**. Its nested target is cleared. A call with the index shifted left by two and four further calls produce another index, whose left-shifted value is supplied twice to another call.

The next call produces the **global update word**. That full word is written to global B before a call receiving it and one further call.

The procedure then reads a word at saved byte selector plus 116 and writes its low byte at saved byte selector plus 1. These accesses use the retained selector, not a new read of address 1. Their respective address ranges are 116–371 and 1–256.

A call receives saved shift value shifted left by 11.

### 6. Paired result and two distinct refreshes — F-D06

A call returns a 64-bit value. The next call receives its low word, its high word, and global update word shifted right by 23, in that order.

Four further calls produce a result that is multiplied by four to form the **refresh base**. A word is read at refresh base plus 116, and its low byte is written at refresh base plus 1.

A call with refresh base precedes four further calls. Their final result is the **refresh index**. Shifting it left by two produces the unsigned 32-bit **scaled refresh value**.

The word at refresh base plus 116 is read again, and this second word’s low byte is written at refresh base plus 1. These are two separate reads and writes, separated by dependencies; the values need not match.

The phase ends with a call receiving saved shift value shifted left by 12, scaled refresh value twice, and refresh index shifted left by 18.

### 7. Full-word capture and 16-byte stride — F-D07

A result-producing call and its forwarding call are followed by another result-producing call and its forwarding call. Four further calls produce an index that is multiplied by four to form the **capture base**.

The full word at capture base plus 116 is retained as the **captured word**. Its low byte is written at capture base plus 1.

A call with capture base and four further calls produce another index. The full captured word is written at `16 × index + 20`. This destination has a 16-byte stride, not the four-byte stride of word-slot addressing.

A final call receives scaled refresh value logically shifted right by two and then left by 16. This calculation uses the already formed 32-bit scaled refresh value; it must not be replaced by an unbounded mathematical expression that restores previously discarded bits.

### 8. Captured-word propagation and scaled-value store — F-D08

Two result-producing-and-forwarding call pairs precede four further calls. Their last result supplies an index, and the full captured word is written to its word-slot.

A call with four times that index and four further calls produce another result. Its left shift by two becomes the **link value**.

The procedure writes link value as a word at:

`(scaled refresh value & 0xFFFFFFFC) + 20`.

A call then receives link value twice. Neither this store nor the argument duplication establishes that link value is a pointer.

### 9. Zero store, paired linkage, and fresh global snapshot — F-D09

Two result-producing-and-forwarding call pairs precede four further calls. Their final result supplies an index whose word-slot is cleared with a zero word.

A call with four times that index and four further calls yield a 64-bit result. Its low word shifted left by two becomes the **paired link**, and its high word becomes the **paired index**.

Paired link is written to word-slot(paired index). Global A is then read again into the **second global snapshot**. A call receives paired link, paired index, paired link, and paired link.

This global read is distinct from the first snapshot and follows the paired-link store.

### 10. Second linkage and captured source word — F-D10

**Exactly four calls occur after the phase-9 four-argument call and before the next direct store. The fourth produces the destination index.** There is no fifth call in this interval.

Paired link is written to that index’s word-slot. The procedure then reads and retains the **source word** at second global snapshot plus 20.

A call receiving four times the index precedes four further calls. The final call returns a 64-bit value divided into the **final low word** and **final high word**.

The procedure reads a byte at captured word shifted right by 23, plus 1, and writes low-byte(source word) to its nested target. The source-byte address is between 1 and 512.

The phase ends with a call receiving final high word three times and final low word shifted left by two as the fourth argument.

The paired-link store precedes the source-word read. The second global snapshot supplies the address value; it does not freeze the memory later read at that address.

### 11. High-bit propagation — F-D11

A result-producing call, its forwarding call, and four further calls produce an index.

The procedure reads a byte at:

`((final low word & 0x3FFFFFFF) >> 21) + 1`.

This source-byte address is between 1 and 512. At the corresponding nested target, it writes low-byte(source word shifted right by 22).

A call receiving the index shifted left by two precedes four further calls. Another call then receives the final result shifted left by two twice.

The nested targets in phases 10 and 11 can differ, even when their source-byte addresses happen to match, because dependencies intervene.

### 12. Final writes and terminal call

Four further unresolved calls occur in order:

- **F-D12:** no explicit argument.
- **F-D13:** produces a word.
- **F-D14:** receives that word.
- **F-D15:** produces a byte result and exposes a secondary pointer.

A zero byte is written through the secondary pointer.

Next, global B receives a 32-bit concatenation: the byte result occupies bits 0–7, and a three-byte operand represented by storage beginning at absolute address 9 supplies bits 8–31. That operand is consumed after the pointer-directed byte clear and before the global B write. Its physical byte-to-bit mapping is not established.

A zero halfword is then written at:

`(final low word & 0x3FFFFFFF) >> 7`.

The address is between 0 and 8,388,607 and is not explicitly aligned.

Finally, **F-D16** denotes the attempted call to address zero with three explicit zero arguments. Normal return is represented only if this call returns.

## Side effects

Direct effects comprise word stores, nested byte reads and writes, word-to-byte truncations, two reads of global A, two writes of global B, the final pointer-directed byte clear, and the halfword clear.

Dependencies can introduce additional effects. Their identities, memory behavior, global preservation, reentrancy, failure modes, and termination behavior are unresolved. No call may be omitted merely because its primary result is unused.

No allocation, release, ownership transfer, retention permission, reference counting, or lifetime guarantee is established. Receiving an address-like value does not establish that a dependency owns or releases the referenced storage.

No disjointness is established between globals, low-address storage, derived destinations, or dependency-accessed memory. Earlier writes can affect later reads through aliasing. There is no visible atomic grouping or rollback.

## Dependencies

F-0ebbc37bef74 is the only supplied resolved opaque function identity.

F-D01–F-D11 are contract-local call-group designations; F-D12–F-D16 identify the final individual calls. They do not establish external function identities or equivalence between call sites.

Most targets are formed from fixed anchors and offsets. The first instead uses incoming link state, and the last is explicitly zero. The target expressions do not establish valid entry points or a conventional calling sequence.

For every unresolved dependency, the complete ABI, hidden inputs, secondary-result semantics, preservation rules, ownership effects, and normal-return behavior remain unknown.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Extraction and forced zero.** If extraction input and the first global snapshot are both `0xFFFFFFFF`, high-ten is 1023 and high-nine is 511. Shifting high-nine right by 23 produces zero.

2. **Truncated extraction.** If the captured table word is `0xFFC00000`, table-high-ten is 1023, but its byte write stores 255.

3. **Different strides.** An index of 3 selects address 32 through word-slot addressing and address 68 through the phase-7 stride.

4. **Nested addressing.** A source byte containing 200 selects destination address 217, irrespective of the source-byte address.

5. **Retained selector.** If saved byte selector is 40, phase 5 reads at address 156 and writes at address 41, even if a dependency changes the byte at absolute address 1.

6. **Repeated refresh.** If an intervening dependency changes the word at refresh base plus 116, the second phase-6 byte write uses the second read, not the first.

7. **Full captured value.** A captured word of `0x12345678` is subsequently stored as that full word, not merely as `0x78`, and is not reread from its original location.

8. **Four-call boundary.** After the phase-9 call receiving paired link, paired index, paired link, and paired link, exactly four calls precede the phase-10 paired-link store. Adding a fifth call changes the contract because it can introduce effects, failure, or nontermination.

9. **Fresh global read.** If global A changes before its second explicit read, the second snapshot reflects that later read rather than the first snapshot.

10. **Final alias ordering.** If the secondary pointer selects a byte contributing to the three-byte operand, its zeroing precedes consumption of that operand. The global B update precedes the halfword clear, even if their storage overlaps.

11. **Terminal failure.** If address zero is not callable, normal return is not required. Replacing the attempted call with an ordinary return omits a represented action.

## Uncertainties

**Established static evidence:** explicit operation order, access widths, masks, shifts, truncations, duplicated arguments, the four-call phase-10 boundary, and the terminal address-zero call.

**Inferred intent:** repeated scaled stores may manipulate structured state or encoded references. This interpretation is not an acceptance requirement.

**documented_only:**

- Whether the representation corresponds to executable control flow.
- Actual ABI, instruction semantics, incoming-state provenance, and secondary-result conventions.
- Behavior of signed overflow and overflowing signed shifts.
- Alignment requirements, memory permissions, and low-address interpretation.
- Endianness and the physical layout of the three-byte concatenation operand.
- Storage overlap between globals and other referenced memory.
- Dependency preservation of retained machine state.
- Whether any dependency or the address-zero call returns normally.

These unknowns must not be replaced with assumed validation, cleanup, ownership, or fallback behavior.

## Evidence

The evidence consists solely of the supplied static function representation. No dependency implementations, callers, machine instructions, memory map, or execution trace were supplied.

The representation supports the conditional dataflow and read/write ordering described above. In particular, between the phase-9 four-argument call and the next paired-link store, it exposes three calls without consumed primary results followed by one index-producing call: four calls total.

The representation also warns of overlapping global symbols, preventing an assumption that named storage locations are independent.

No runtime tests have been performed.
