<!-- Trusted provenance: F-6dd1def7e7d8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6dd1def7e7d8

## Purpose

F-6dd1def7e7d8 performs an ordered sequence of indirect calls and direct memory accesses. It forwards one explicit argument, extracts high-order fields, copies and distributes word values, clears selected locations, assigns a global word twice, and finally attempts a call through numeric address zero.

No application-level purpose is established. Offsets, global labels, and repeated call patterns do not establish object initialization, service dispatch, allocation, or ownership management.

This contract covers the statically represented behavior. Each operation is conditional on all preceding calls returning and all preceding accesses completing. Dependency effects are additional to the direct effects described here.

## Interface

### Explicit inputs and result

The represented interface has three 32-bit inputs and no explicit return value.

| Input | Interpretation | Visible use |
|---|---|---|
| First argument | Unspecified 32-bit value | No explicit use. |
| Second argument | Unspecified 32-bit value | Forwarded unchanged to F-C001. |
| Third argument | Unsigned 32-bit value | Logically shifted right by 23 and stored as a four-byte word. |

An ordinary return follows the terminal indirect call, but it is reachable only if that call returns. There is no represented success code, failure code, or exception-handling path.

The first argument’s lack of explicit use does not prove that dependencies cannot observe incoming registers or other ambient execution state.

### Ambient inputs and storage

The sequence also depends on:

- An incoming link-derived value used to calculate the first call target.
- Two ambient unsigned 32-bit words, designated **ambient A** and **ambient B**.
- An auxiliary 32-bit value supplied through call-related state early in execution.
- A second auxiliary 32-bit value captured later and retained.
- A terminal byte pointer supplied through call-related state.
- A four-byte global designated the **control word**.
- A four-byte global designated the **anchor word**.
- A three-byte global view beginning at numeric address 9.
- Memory contents and dependency results encountered during execution.

These descriptive names do not establish data structures or application semantics. The auxiliary values and terminal pointer are not established as ordinary additional return values.

### Widths, addresses, and prerequisites

Direct accesses comprise unsigned byte reads, one-byte writes, four-byte reads and writes, a final two-byte write, and a three-byte view used in final word composition.

A word-to-byte copy reads all four source bytes and writes the resulting value’s low eight bits. It is not a byte-only source access. Low and high halves of a represented 64-bit result mean bits 0–31 and 32–63 respectively.

Addresses are numeric expressions. No enclosing allocation or common base is established. In particular, a byte-selected destination is the unsigned byte value plus 17, not that value added to an implicit object base.

Completion requires callable targets, valid accesses at the stated widths, usable ambient state, and an environment in which the final call through address zero returns. The function does not validate these requirements. Alignment, byte order, address width, and fault handling remain documented_only.

## Behavior

### Ordering and arithmetic conventions

F-C001 through F-C126 identify successive call occurrences, not verified distinct implementations. “No explicit arguments” means none are represented; it does not constrain what ambient state a dependency may inspect.

Unsigned right shifts are logical. Unsigned 32-bit left shifts discard bits beyond bit 31. Some address scaling and left shifts instead involve represented signed 32-bit intermediates; their overflow and negative-input interpretation remain qualified under **Uncertainties**. Examples involving these operations use values that avoid those ambiguities.

Retained values below describe represented data flow, not a general guarantee that unknown callees preserve registers, globals, or memory.

### Opening calls and initial field updates

F-C001 receives the second argument. Its target is the incoming link-derived value plus 3,896.

F-C002 returns a 32-bit value forwarded to F-C003. F-C004 through F-C006 then execute without explicit arguments.

F-C007 returns a 32-bit value. Before the next call, the function:

- Stores the third argument logically shifted right by 23 as a four-byte word at four times F-C007’s result plus 20.
- Reads a four-byte word at ambient A logically shifted right by 22, plus 20. This is the **initial word**.

F-C008 receives four times F-C007’s result. F-C009 through F-C012 execute without explicit arguments.

The function retains two unsigned fields: ambient B shifted right by 22, designated the **ambient field**, and the initial word shifted right by 22, designated the **initial field**. It reads a byte at ambient B plus 1 and writes the initial field’s low byte to the byte-selected address plus 17.

F-C013 receives the first auxiliary value three times, followed by the ambient field.

### Initial clearing and retained call result

F-C014 has no explicit arguments. F-C015 receives the ambient field logically shifted right by 23. This argument is necessarily zero: the earlier shift by 22 leaves at most ten significant bits.

F-C016 through F-C018 have no explicit arguments. F-C019 returns a 32-bit value. Before F-C020, a byte is read at four times the initial field plus 1, and a zero byte is written at that byte value plus 17.

F-C020 receives F-C019’s result shifted left by two. F-C021 through F-C023 have no explicit arguments. F-C024 returns a value whose left-shifted-by-two form is passed twice to F-C025.

F-C026 returns a retained unsigned 32-bit value. F-C027 receives it unchanged, followed by F-C028 with no explicit arguments.

### Sign-selected accesses and first control assignment

The sign bit of ambient B selects a byte-read address:

- Address 1 if its signed 32-bit interpretation is nonnegative.
- Address 2 if that interpretation is negative.

The selected byte is read, and zero is written at its value plus 17.

The second auxiliary value is then captured. F-C029 receives it three times. F-C030 returns a value; F-C031 receives that value followed by F-C026’s retained result logically shifted right by 23.

F-C032 through F-C034 have no explicit arguments. F-C035 returns a value.

The sign-selected address is now read again. This fresh byte becomes the **selected byte**. Zero is written at the selected byte plus 17. The earlier byte value cannot substitute for this read: intervening dependencies may have changed memory.

F-C036 receives F-C035’s result shifted left by two. F-C037 through F-C039 have no explicit arguments. F-C040 returns a value whose left-shifted-by-two form is passed twice to F-C041.

F-C042 returns a retained unsigned 32-bit value. That value is written to the control word before being passed to F-C043. F-C044 follows without explicit arguments.

A four-byte word is then read at the selected byte plus 116. Its low byte is written at the selected byte plus 1.

### Repeated word-to-byte copies

F-C045 receives the second auxiliary value shifted left by 11.

F-C046 produces a represented 64-bit result. F-C047 receives, in order, its low half, its high half, and F-C042’s retained result logically shifted right by 23.

F-C048 through F-C050 have no explicit arguments. F-C051 returns a value that is multiplied by four to form the **copy base**.

A four-byte word is read at the copy base plus 116, and its low byte is written at the copy base plus 1. F-C052 receives the copy base.

F-C053 through F-C055 have no explicit arguments. F-C056 returns a value whose left-shifted-by-two form becomes the retained unsigned **scaled value**.

The copy at the copy base is performed again, with a fresh four-byte read at offset 116 and a byte write at offset 1. This is not reuse of the earlier source value.

F-C057 receives four arguments:

1. The second auxiliary value shifted left by 12.
2. The scaled value.
3. The scaled value again.
4. F-C056’s result shifted left by 18.

### Payload capture and full-word distribution

F-C058 returns a value forwarded to F-C059. F-C060 returns another value forwarded to F-C061.

F-C062 through F-C064 have no explicit arguments. F-C065 returns a value multiplied by four to form another base.

A four-byte word is read at that base plus 116 and retained as the **saved payload**. Its low byte is written at the base plus 1. F-C066 receives the base.

F-C067 through F-C069 have no explicit arguments. F-C070 returns a value. The complete saved payload is written as four bytes at sixteen times that result plus 20.

F-C071 receives the scaled value logically shifted right by two, then shifted left by 16. The initial formation of the scaled value may already have discarded high bits; this expression must not be treated as unrestricted recovery of F-C056’s original result.

F-C072 returns a value forwarded to F-C073. F-C074 returns another value forwarded to F-C075. F-C076 through F-C078 have no explicit arguments.

F-C079 returns a value. The complete saved payload is written at four times that result plus 20. F-C080 receives four times the result.

### Linking, clearing, and anchor capture

F-C081 through F-C083 have no explicit arguments. F-C084 returns a value that is shifted left by two.

The shifted result is written as four bytes at the scaled value with its bottom two bits cleared, plus 20. Those bits are already zero under the represented 32-bit formation of the scaled value. F-C085 receives the shifted F-C084 result twice.

F-C086 returns a value forwarded to F-C087. F-C088 returns another value forwarded to F-C089. F-C090 through F-C092 have no explicit arguments.

F-C093 returns a value. A four-byte zero is written at four times that result plus 20. F-C094 receives four times the result.

F-C095 through F-C097 have no explicit arguments. F-C098 produces a represented 64-bit result. Its low half shifted left by two becomes the **link value**. The link value is written as four bytes at four times the high half plus 20.

After this write, the anchor word is read and retained as an address. F-C099 receives the link value, F-C098’s high half, the link value again, and the link value once more.

F-C100 through F-C102 have no explicit arguments. F-C103 returns a value. The link value is written at four times that result plus 20.

Only after that store, a four-byte word is read at the captured anchor address plus 20. This is the **anchor payload**. The anchor global is not refreshed following the intervening calls. F-C104 receives four times F-C103’s result.

### Final field updates and terminal operations

F-C105 through F-C107 have no explicit arguments. F-C108 produces a represented 64-bit result. Its low half becomes the unsigned **terminal field**.

A byte is read at the saved payload logically shifted right by 23, plus 1. The anchor payload’s low byte is written at that byte value plus 17.

F-C109 receives F-C108’s high half three times, followed by the terminal field shifted left by two.

F-C110 returns a value forwarded to F-C111. F-C112 through F-C114 have no explicit arguments. F-C115 returns a value.

The terminal field is masked to its low 30 bits and logically shifted right by 21. A byte is read at this result plus 1. The low byte of the anchor payload logically shifted right by 22 is written at the byte-selected address plus 17.

F-C116 receives F-C115’s result shifted left by two. F-C117 through F-C119 have no explicit arguments. F-C120 returns a value whose left-shifted-by-two form is passed twice to F-C121.

F-C122 has no explicit arguments. F-C123 returns a value forwarded to F-C124. F-C125 returns a represented byte.

The remaining operations occur in this order:

1. Write a zero byte through the terminal side-channel pointer.
2. Obtain the three-byte view beginning at address 9 and compose a 32-bit value with that view in bits 8–31 and F-C125’s byte in bits 0–7.
3. Write the composed value to the control word.
4. Write a two-byte zero at the terminal field masked to 30 bits, then logically shifted right by 7.
5. Attempt F-C126 through address zero with three zero arguments.
6. Return if F-C126 returns.

The pointer clear precedes the three-byte read. Aliasing between them can therefore affect the composed control value.

## Side effects

The complete represented path contains **21 direct stores: eleven one-byte stores, nine four-byte stores, and one two-byte store**. These counts include both control-word assignments and exclude dependency effects.

| Width | Direct-store accounting |
|---|---|
| One byte: 11 | Initial field update; initial-field-selected clear; two sign-selected clears; selected-byte-based word-to-byte copy; two copy-base word-to-byte copies; saved-payload byte copy; two final byte-selected updates; terminal pointer clear. |
| Four bytes: 9 | Initial extracted-field store; first control assignment; two saved-payload stores; shifted F-C084 result store; explicit word clear; two link-value stores; final control assignment. |
| Two bytes: 1 | Final terminal-field-derived clear. |

These are static operation counts, not a guarantee that execution reaches every store. Stores may overlap, and a dependency or access failure may stop the sequence partway through.

Every unsigned-byte-selected destination lies between addresses 17 and 272. Sign selection reads address 1 or 2. The final two-byte destination lies between 0 and 8,388,607 and need not be aligned. The three-byte composition view begins at address 9.

No global memory snapshot is taken. Retained values, captured addresses, and fresh reads have different behavior when memory changes between calls.

There is no explicit allocation, deallocation, ownership transfer, reference counting, bounds checking, null checking, rollback, or error reporting. Dependency behavior may include any of these, but none is established. Passing or receiving a pointer does not establish ownership, borrowing, validity, or lifetime.

## Dependencies

F-C001 through F-C126 are opaque occurrence IDs used only for order and data flow.

- F-C001 uses an incoming-link-relative target.
- F-C002 through F-C125 use fixed location-derived target expressions.
- F-C126 uses numeric address zero.

Different occurrences may resolve to the same implementation. No target identity, purity, preservation rule, memory-management policy, error convention, or normal-return guarantee is established.

F-C046, F-C098, and F-C108 have represented 64-bit results. F-C125 has a represented one-byte result. Other consumed ordinary results are 32 bits. Auxiliary call-related state remains separate from these ordinary result descriptions.

No conditional control-flow branch or loop is visible in the represented function body. Sign-dependent address calculation does not skip the associated access. Dependencies may branch, loop, fault, mutate globals, or fail to return.

## Acceptance examples

The following are conditional static expectations, not executed tests. Each assumes earlier operations complete and the relevant accesses are valid.

1. **Nine-bit extraction.** With third argument `0xFFFFFFFF` and F-C007 result 3, the initial four-byte store writes `0x000001FF` at address 32.

2. **Forced-zero argument.** For every unsigned 32-bit ambient B, F-C015 receives zero. The successive logical right shifts by 22 and 23 remove all significant bits.

3. **Unsigned selection and narrowing.** If the initial word is `0xFFFFFFFF` and the byte at ambient B plus 1 is 5, the initial byte update writes `0xFF` at address 22. The intermediate initial field is `0x3FF`.

4. **Sign-dependent fresh read.** Ambient B of `0x80000000` selects address 2; `0x7FFFFFFF` selects address 1. If the selected entry changes between the two reads, the second clear uses the later value.

5. **Four-byte source requirement.** A word-to-byte source containing `0x123456AB` produces byte `0xAB`. Permission to read only its lowest-addressed byte does not satisfy the represented four-byte access.

6. **Full payload retention.** If the saved payload is `0x89ABCDEF` and F-C070 returns 2, the corresponding four-byte store writes `0x89ABCDEF` at address 52, not merely `0xEF`.

7. **Captured address, later contents.** If the anchor word is captured as 100 and later changed to 200, the anchor-payload read still accesses address 120. Its contents are obtained after the intervening calls and link-value store.

8. **Final composition.** If F-C125 returns `0x7E` and the three-byte view has numeric value `0x123456` when read, the final control value is `0x1234567E`. A preceding aliased pointer clear must affect the view before composition.

9. **Unaligned terminal destination.** Terminal field `0xFFFFFFFF` yields final two-byte destination `0x007FFFFF`. The calculation supplies no alignment guarantee.

10. **Store accounting.** If execution reaches every direct store, there are 21 store operations: eleven byte, nine word, and one two-byte. Aliasing does not reduce the operation count.

11. **Terminal failure is not success.** If address zero is not callable, its attempted invocation cannot be treated as a successful no-op. No local recovery or error result is represented.

## Uncertainties

The following remain **documented_only**:

- Dependency identities, actual signatures, target validity, and effects.
- Origins and lifetimes of ambient values, auxiliary state, and the terminal pointer.
- Actual calling convention and preservation of call-related state.
- Address width, byte order, alignment requirements, and access-fault behavior.
- Whether low numeric addresses identify ordinary memory, device mappings, special runtime storage, or invalid locations.
- Global addresses and overlaps; the representation warns that some global views overlap smaller symbols.
- Whether any dependency returns normally, especially F-C126.
- Whether the terminal pointer aliases the three-byte composition view or other affected storage.

Several scaling and left-shift operations use represented signed 32-bit intermediates. Negative operands, signed overflow, and signed left shifts must not be assigned ordinary mathematical or host-language behavior without executable evidence. A modulo-\(2^{32}\) machine-bitvector interpretation is conditional where signed operations are involved. Sign selection specifically uses an arithmetic right shift of a signed 32-bit interpretation.

The three-byte view establishes a numeric composition relationship, not a resolved byte-order mapping. Offsets 1, 17, 20, and 116 do not establish a data structure or ownership model.

## Evidence

**Established static evidence:** the supplied body establishes explicit argument uses, call order, represented result widths, direct access widths, fresh versus retained reads, two control assignments, anchor-address capture, terminal clearing, and the final call through address zero. Direct enumeration establishes 21 stores: eleven one-byte, nine four-byte, and one two-byte.

**Inferred intent:** none is required. “Control,” “anchor,” “payload,” and “link” name observable roles only.

**Unknowns:** dependency contracts, runtime memory validity, calling-convention details, ownership, overlap resolution, and runtime outcomes remain documented_only.

No runtime tests have been performed. Acceptance examples describe consequences of the static operations under their stated conditions, not observed execution.
