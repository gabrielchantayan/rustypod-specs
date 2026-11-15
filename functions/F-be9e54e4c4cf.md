<!-- Trusted provenance: F-be9e54e4c4cf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-be9e54e4c4cf

## Purpose

The supplied static representation establishes a stateful sequence of indirect calls, 32-bit memory transfers, byte updates, bit extraction, and a final attempted call through address zero. It does not establish a domain-level purpose.

The observable behavior depends on more than the explicit argument: incoming register state, a return-link-derived call target, values supplied through secondary result registers, two global storage regions, and numerous indirect dependencies all affect execution.

This contract describes the visible accesses and their ordering, conditional on execution reaching them. It does not characterize the operation as successful initialization, allocation, decoding, or cleanup. Those interpretations are unsupported.

## Interface

**Explicit input:** one uninterpreted 32-bit value. It is forwarded unchanged to the first indirect dependency. No subsequent direct use of that argument is visible.

**Additional effective inputs:**

- An incoming return-link value used to construct the first call target.
- An incoming 32-bit register value used in an unsigned right shift by 22, then in a memory address.
- Another incoming 32-bit register value used as the first stored word.
- Secondary result-register contents following several calls. These supply word indexes, an opaque argument, a saved value used in later shifts, and the final byte-store destination.
- Memory read through all derived addresses described below.

These are machine-state requirements, not established additional source-language parameters. Their calling convention is **documented_only**.

**Output:** the displayed interface has no explicit return value. A normal return appears after the attempted call through address zero, but reaching that return depends on whether that call completes and returns. No success indicator is established.

**Memory widths:**

- Word loads and stores are 32 bits.
- Byte loads and stores are 8 bits.
- The final clearing store is 16 bits.
- Two-word dependency results are represented as 64 bits and split into low and high 32-bit portions.
- Converting a loaded word to a stored byte retains its low eight bits.

For address descriptions, define two layout operations:

- **Word-slot address:** 20 plus four times an index.
- **Byte-selected address:** 17 plus the unsigned byte read from a specified address.

These describe arithmetic, not recognized object types. There is no evidence of an array bound, allocation header, tagged-pointer schema, or valid mapped range.

Address calculations appear 32-bit. Unsigned shifts are logical. Several index scalings and left shifts are displayed using signed intermediates; their behavior outside nonnegative, representable ranges cannot safely be specified as portable signed arithmetic. A low-32-bit machine interpretation is plausible but remains **documented_only** where signed overflow or shifting a negative value would matter.

**Execution preconditions:** to complete the visible sequence, every reached call target must be executable, every read address readable, and every write address writable at its stated width. Any required alignment must also be satisfied. The function performs no visible checks establishing these conditions.

## Behavior

### Dependency sequencing

The operation has no visible conditional branches or loops. It follows one displayed sequence unless an access or dependency interrupts it.

Many dependencies are invoked without explicit arguments, including between consecutive memory operations. They cannot be treated as inert preparation: they may modify globals, pointed-to storage, or other externally observable state. Values already captured in the visible sequence are subsequently reused, while later explicit memory reads obtain memory again.

The phase descriptions below preserve the order of the direct memory operations. Dependency families are identified in **Dependencies**; the labels do not assert that repeated-looking call sites reach the same function.

### Initial word construction

The initial dependency receives the explicit argument. Four further calls follow; the last supplies an index.

1. Store the incoming preserved word at that index’s word-slot address.
2. Read a word at address 20 plus the incoming address-selector word logically shifted right by 22.

A five-call group follows. Its first call receives four times the earlier index; its last supplies another index.

3. Logically shift the saved loaded word right by one. Use that result as a word-slot index and store four times the new dependency result there.
4. Read another word at address 20 plus the original saved loaded word—not at a scaled index.

A dependency receives the half-shifted value. Another returns an opaque word, which is passed to the following dependency. Four more calls follow; the last supplies a new index.

5. Store the previously computed four-times result at this new index’s word-slot address.
6. Read the first distinguished global word, here called **global A**.

The next five-call group begins with four times the latest index. After that group:

7. Store the second loaded word, logically shifted right by 22, at a word-slot address selected by a secondary result register.

Global A’s captured value is logically shifted right by 23 and retained. A dependency then receives four explicit words: the secondary index twice, the shifted global value, and the shifted loaded word.

### Shifted-word replication and clearing

A result-producing call and a forwarding call follow. Between them, the already shifted global value is shifted right by another 23 bits. Because the original value is only 32 bits, this twice-shifted value is necessarily zero.

Four further calls produce an index.

8. Store the previously saved 22-bit-shift result at that index’s word-slot address.

Another five-call group begins by receiving four times that index.

9. Store a zero word at a word-slot address selected by a newly observed secondary register result.

A dependency receives that secondary index. A result-producing call, a forwarding call, and four further calls follow.

10. Store the twice-shifted global value—necessarily zero—at the resulting index’s word-slot address.
11. Read a word at address 20 plus the zero-cleared secondary index masked with `0x1fffffff`, then logically shifted right by 19.

The mask-and-shift produces a value from zero through 1023. It is added directly to 20; it is not multiplied by four.

A five-call group follows, starting with four times the most recent ordinary index.

12. Read a byte at address 1, because the earlier twice-shifted global value is zero.
13. At 17 plus that byte, store the low eight bits of the word from step 11 logically shifted right by 22.

This byte conversion retains original word bits 22 through 29, not all ten bits remaining after the shift.

A dependency receives the same opaque secondary-register word three times, followed by an explicit zero. A no-argument call follows; the next dependency receives zero. Four further calls produce an index.

14. Read a byte at address 1 plus four times the shifted word used in step 13.
15. Store zero at 17 plus that byte.

Since the shifted word is at most 1023, the step-14 read address is between 1 and 4093.

### Low-address byte clearing and global publication

A five-call group begins with four times the latest result. Its final call supplies another result. A dependency receives four times that result twice. The next call returns a fresh word, which is passed onward, followed by another call.

16. Read the byte at address 1 and clear the byte at 17 plus its value.

The currently observed secondary-register value is retained for later shifts. A dependency receives that value three times. A result-producing call follows; the next dependency receives that result and the earlier fresh word logically shifted right by 23. Four more calls produce an index.

17. Read the byte at address 1 again. Retain its value and clear the byte at 17 plus that value.

The two address-1 reads are distinct. Their values may differ because dependencies intervene.

A five-call group begins with four times the latest index. Its result is scaled by four and passed twice to another dependency. A subsequent call supplies a fresh word.

18. Write that fresh word to the second distinguished global word, **global B**.

The fresh word is passed to a dependency, followed by another call.

19. Using the byte retained in step 17 as an unscaled address base, read a word at base plus 116.
20. Store that word’s low byte at base plus 1.

The address base is the saved byte, not a new read of address 1.

### Word-to-byte transfers and scaled-slot writes

A dependency receives the retained secondary value shifted left by 11. A later call supplies a 64-bit result. The next dependency receives its low word, its high word, and the word published in step 18 logically shifted right by 23.

Four calls produce a new index, which is scaled by four.

21. Read a word at that scaled base plus 116.
22. Store its low byte at that base plus 1.

A five-call group starts with that base and produces another index.

23. Read the word at the same base plus 116 again.
24. Again store its low byte at the same base plus 1.

The second read is not redundant: intervening calls can change the source word.

A dependency receives four arguments: the retained secondary value shifted left by 12; four times the latest index twice; and that index shifted left by 18. A result-producing call, a forwarding call, another result-producing call, and another forwarding call follow. Four further calls supply a new index, scaled by four.

25. Read and retain the word at this new base plus 116.
26. Store its low byte at this base plus 1.

A five-call group begins with this base and supplies another index.

27. Store the full retained word at address 20 plus sixteen times that index.

A dependency receives the earlier four-times index logically shifted right by two and then shifted left by 16. This is not necessarily equivalent to shifting the original unscaled index left by 16: the earlier scaling may already have discarded high bits.

Two result-producing/forwarding pairs and four calls follow.

28. Store the retained full word at the resulting index’s word-slot address.

A five-call group begins with four times this index and supplies another result.

29. Store four times that result at address 20 plus the earlier four-times index masked with `0xfffffffc`.

The destination mask is explicit. It is not evidence that any independently supplied index was validated.

### Final word propagation and byte updates

A dependency receives the step-29 stored value twice. Two result-producing/forwarding pairs and four calls follow.

30. Clear a word at the resulting index’s word-slot address.

A five-call group begins with four times this index. Its final call returns a 64-bit value.

31. Store four times its low word at the word-slot address selected by its high word.
32. Read global A again.

This is a new global read, not reuse of the earlier captured value.

A dependency receives the scaled low word, the high word, and the scaled low word twice more. Four calls produce another index.

33. Store the same scaled low word at this new index’s word-slot address.
34. Read a word at address 20 plus the newly captured global A value.

A five-call group follows, beginning with four times the most recent index and ending in another 64-bit result.

35. Read a byte at address 1 plus the step-25 retained word logically shifted right by 23.
36. At 17 plus that byte, store the low byte of the step-34 loaded word.

A dependency receives the latest 64-bit result’s high word three times and four times its low word. A result-producing call, a forwarding call, and four further calls follow.

37. Read a byte at address 1 plus the latest low word masked with `0x3fffffff` and logically shifted right by 21.
38. At 17 plus that byte, store the low byte of the step-34 word logically shifted right by 22.

The address selector in step 37 ranges from zero through 511 before adding 1.

### Terminal operations

A five-call group begins with four times the latest ordinary result. Its result is scaled by four and supplied twice to another dependency. Three further calls form a result-and-forwarding sequence, followed by a final byte-producing call.

39. Store zero through the byte pointer supplied in a secondary result register.
40. Write global B as a 32-bit concatenation: the final dependency’s byte is the low byte, and a separately named 24-bit storage region associated with address 9 supplies the upper 24 bits.
41. Store a zero halfword at the address obtained by masking the latest low word with `0x3fffffff` and logically shifting right by seven.
42. Attempt to call address zero with three explicit zero arguments.

The final halfword address can be odd. No alignment correction is visible. A normal return is shown only after the address-zero call.

## Side effects

All listed writes are direct side effects. Indirect dependencies may introduce additional effects between them.

There is no visible transaction, rollback, bounds check, null check, allocation, deallocation, reference-count update, or ownership transfer. The absence of such operations in the visible function does not establish their absence inside dependencies.

The low-address byte writes may alias one another. In particular, writes based on an 8-bit selector target addresses 17 through 272. Word-slot destinations, global regions, and terminal stores may also overlap if their input values permit it. No non-aliasing guarantee is established.

Global B is written twice: first with a complete dependency-produced word, later with a composite word. The later write does not undo intervening side effects.

A dependency that fails to return prevents all later visible operations. Likewise, a faulting access can leave earlier writes committed. No recovery path is visible.

## Dependencies

Only the enclosing function has a trusted supplied identity. The following opaque IDs are contract-local references to unresolved dependency families, not recovered symbol identities:

| Opaque ID | Role |
|---|---|
| F-000000000001 | Initial indirect call, targeted using incoming return-link state, receiving the explicit argument. |
| F-000000000002 | Ordered families of no-explicit-argument calls and index-producing calls. |
| F-000000000003 | Calls receiving scaled indexes, saved words, repeated arguments, or shifted fields. |
| F-000000000004 | Calls whose ordinary results are forwarded to subsequent dependencies. |
| F-000000000005 | Calls represented as producing 64-bit results split into two 32-bit words. |
| F-000000000006 | Final byte-producing dependency. |
| F-000000000007 | Terminal target at address zero. |

A family includes multiple call sites. Membership does not imply common implementation, equivalent effects, or preserved state.

The displayed empty argument lists establish only that no explicit arguments are shown. They do not prove that machine registers or memory are irrelevant inputs. Secondary-register outputs likewise have no established typed interface.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests.

- **Initial argument:** if the explicit argument is `0x89abcdef`, the first dependency must receive that exact 32-bit pattern. No byte extraction or sign extension precedes forwarding.
- **Double shift:** for any captured 32-bit global A value, shifting it right by 23 twice produces zero. The corresponding later word store must therefore write zero, and the associated byte-selector read must use address 1.
- **Byte truncation:** if the step-11 loaded word is `0xffffffff`, shifting by 22 yields 1023, but step 13 writes byte `0xff`, not a wider value.
- **Fresh read:** if a dependency changes the word at the step-21 base plus 116 before step 23, step 24 must reflect the second read. Reusing the first loaded word is not equivalent.
- **Preserved full word:** if step 25 reads `0x12345678`, step 26 writes byte `0x78`, while steps 27 and 28 write the full word `0x12345678`, provided execution reaches them without an applicable arithmetic ambiguity.
- **Masked selector:** if the later low word is `0xffffffff`, step 37 reads its selector byte at address 512.
- **Composite global write:** if the final dependency returns byte `0xab` and the terminal 24-bit field is `0x123456`, step 40 writes numeric word `0x123456ab`. This example does not specify byte order in memory.
- **Terminal failure:** if address zero is not executable, the function may fail after step 41. A conforming description must not promise normal return or rollback.

## Uncertainties

The following remain **documented_only**:

- The domain purpose and meaning of the repeated call patterns.
- Exact dependency identities, calling conventions, clobber rules, and ownership effects.
- Whether the displayed secondary-register values are intentional outputs, hidden inputs, or representation artifacts.
- Signed scaling and left-shift semantics for negative or overflowing operands.
- Processor alignment requirements, memory mapping, endianness, and fault delivery.
- Global A and global B’s exact storage relationships with narrower overlapping symbols.
- The physical byte layout and read timing of the terminal 24-bit field.
- Whether address zero is a meaningful platform entry point or a faulting target.
- Concurrency, atomicity, reentrancy, and external observation guarantees.

No error-code contract is established. No assumption that unknown callees preserve globals or release memory is justified.

## Evidence

The supplied static representation supports the access widths, visible call ordering, explicit arguments, masks, shifts, global assignments, and terminal address-zero call described above.

Unsigned range reductions and the twice-shifted zero are mathematical consequences of the displayed 32-bit widths. Domain interpretations are not established evidence.

This contract was prepared from the embedded representation only. No implementation inspection beyond that representation, runtime execution, or runtime tests have been performed.
