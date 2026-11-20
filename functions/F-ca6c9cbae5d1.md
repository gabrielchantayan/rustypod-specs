<!-- Trusted provenance: F-ca6c9cbae5d1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ca6c9cbae5d1

## Purpose

F-ca6c9cbae5d1 executes an ordered sequence of dependency invocations and direct memory accesses. Its established output is mutation of memory and externally visible call activity, not a returned value.

The direct effects include selector-directed byte writes to low absolute addresses, byte and halfword reads, scaled-address 32-bit stores, field reads at repeated offsets, one reset-region clear, and two publication-region assignments. The sequence ends by attempting an invocation through absolute address zero.

No higher-level operation is established. Repeated offsets and scaling suggest structured state manipulation, but do not establish initialization, serialization, allocation, dispatch, or an object model.

This contract distinguishes:

- **Established static evidence:** behavior explicitly represented in the supplied material.
- **Inferred intent:** interpretations that are not behavioral requirements.
- **documented_only:** unresolved behavior requiring dependency implementations, ABI information, memory mappings, or instruction-level evidence.

No runtime tests have been performed.

## Interface

### Explicit inputs and output

The represented interface has four 32-bit input positions.

| Position | Established use |
|---|---|
| First | Unsigned value used for byte extraction, logical right shifts, address formation, and dependency arguments. |
| Second | Opaque 32-bit value with no explicit use represented. |
| Third | Opaque 32-bit value with no explicit use represented. |
| Fourth | Opaque 32-bit value supplied to the initial dependency invocation. |

No explicit return value is represented. A return follows the terminal zero-address invocation, but is reachable only if that invocation returns.

Absence of explicit uses for the second and third inputs does not establish irrelevance to actual machine execution. The represented dependency interfaces may omit live machine-register arguments.

### Additional entry state

Execution also depends on:

- An incoming base used to locate the entry selector byte.
- An incoming link-related value used to derive the first call target.
- Primary, secondary, and paired machine results from dependencies.
- Accessible memory at low and computed addresses.
- A **reset region**, a **publication region**, and a three-byte global value used in the final publication.
- Actual calling conventions and target behavior.

The incoming base and link-related value are not initialized within the represented body.

### Widths and arithmetic

Direct memory operations use 8-bit bytes, 16-bit halfwords, and 32-bit words. Some dependency results are represented as 64 bits and separated into low and high 32-bit portions. This representation does not establish a language-level 64-bit return convention.

Byte stores retain the source’s low eight bits. Explicit unsigned byte loads produce values from 0 through 255. Extension of other width-only byte and halfword values into dependency arguments is **documented_only**.

Unsigned right shifts are logical. For unsigned 32-bit expressions, arithmetic retains the low 32 bits. Some address and shift expressions instead have signed types; their exact machine behavior, particularly overflow and signed left shifting, is **documented_only**. Mathematical descriptions of those expressions are conditional on the corresponding 32-bit bit-vector interpretation.

Endianness, alignment requirements, memory permissions, and fault behavior are not established.

### Preconditions for progress

The function performs no represented validation. Reaching the end requires each read and write to be valid at its specified width, each invoked target to be callable, and all consumed machine results to be usable.

The stack-relative address passed externally must support whatever access the dependency performs. Address zero must be callable and return for the final return to be reached. These are conditions for progress, not checks or guarantees.

## Behavior

### Ordering and selector writes

The represented direct body contains no conditional branches, loops, validation exits, or recovery paths. It forms one ordered trace interrupted by dependency invocations. A dependency may branch, mutate memory, fail, transfer control, or not return.

A **selector write** means:

- An unsigned byte is read from the stated selector address.
- Seventeen is added to its numeric value.
- One byte is written at that resulting absolute address.

Every selector-write destination is therefore between absolute addresses 17 and 272 inclusive. The selector value is not a pointer to a structure.

Each selector occurrence performs a fresh read. Repeated expressions do not permit combining writes or reusing earlier selector values. Direct writes and dependency effects may change later selectors; address equality and aliasing must remain possible.

### Entry sequence and initial results

The first direct access reads the selector at the incoming base plus one and writes the low byte of the first input through it.

The initial dependency invocation receives the first input twice and the fourth input once. Its target is formed from the incoming link-related value plus 4094.

**Exactly four subsequent dependency invocations precede the second entry-selector write.** The fourth of those subsequent invocations supplies a retained result. The function then rereads the selector at the incoming base plus one and writes the low byte of the first input logically shifted right by 22. Only after that write is the retained result shifted left by two and supplied to the next invocation.

Later early calls consume left-shifted results, repeated result values, the first input shifted right by 19, the first input shifted right by nine, and an internal call-anchor value.

A paired result supplies a retained low 32-bit address source and a high 32-bit portion. Before the high portion is passed onward, a zero selector write uses the selector at one plus the first input shifted right by nine. Additional calls occur before another zero selector write from that selector address.

### Result-derived reads and clears

The retained low result supplies its upper four bits to a dependency. Subsequent direct operations, separated by calls, include:

- A zero selector write using one plus that result shifted right by nine.
- An unsigned byte read at twice that result; the loaded value is supplied in four argument positions.
- Another zero selector write using the shifted-result selector.
- A selector read at absolute address one followed by a zero selector write.

A secondary machine result shifted right by 18 is also supplied to a dependency.

A later paired result replaces the retained address source. A selector based on an earlier secondary result shifted right by nine is cleared before the paired high portion is passed onward. Another clear uses the selector at the new low result plus one.

The function reads a full 32-bit word at the new low result plus a high-bit displacement and supplies that word in four argument positions. Under 32-bit bit-vector interpretation, the displacement is zero or `0x80000000`, selected by bit 19 of the low result.

Further accesses, with intervening calls, are ordered as follows:

- Clear through the selector at one plus the retained low result shifted right by nine.
- Clear through the selector at a secondary result plus one.
- Read one byte at that secondary result plus 14.
- Clear through the selector at one plus that secondary result shifted right by nine.
- Clear through the selector at a subsequent paired low result plus one.
- Clear through the selector at one plus 4096 times that paired low result.

No bounds, null, or alignment checks are represented.

### Stack exposure and mixed-width reads

A secondary result is supplied in three argument positions, with its logical right shift by 19 in another. Later, one stack-relative address is supplied in all four argument positions to a dependency. The referenced area’s extent and initial contents are not established.

After another paired result, selector clears use a parity-derived selector address and one plus the paired low result shifted right by 19. Under unsigned bit-vector arithmetic, the parity-derived address is either one or `0x80000001`.

An unsigned byte is read at a retained low result plus a signed secondary result and supplied in four argument positions. Another selector clear uses the retained result shifted right by nine.

A subsequent unsigned secondary result is shifted right by 19. Following calls and a selector clear at the unshifted result plus one, a 16-bit quantity is read at the shifted value plus the earlier unsigned byte. That halfword is supplied in four argument positions. No two-byte alignment check is shown.

### Scaled stores and offset-13 writes

The next paired result begins the scaled word-store sequence. The earlier secondary result shifted right by 19 is stored as a 32-bit value at four times the paired high portion plus 20. Calls intervene before the same retained value is stored at four times another call result plus 20.

The paired low portion remains a base for two separate byte writes at offset 13:

- First, the low byte of twice that base is written.
- Later, after intervening calls and a word read, the constant byte four is written.

Between those writes, a dependency receives a 32-bit word read at four times a newly returned unsigned value plus 68. A later word read uses 68 plus that same returned value masked to 30 bits and shifted right by 20.

Subsequent stores include four times the retained returned value at 256 times another result plus 20, and a 26-bit-masked extraction shifted right by 13 at four times a further result plus 20.

The recurring store pattern alternates destinations based on four-times and sixteen-times scaling, both with offset 20. Stored values include retained scaled results. Independently produced destination values may coincide; the pattern establishes neither disjoint objects nor contiguous allocation.

### Offset-36, offset-116, and offset-21 accesses

After further scaled stores, a zero word is written at a sixteen-times-scaled base plus 36. A word is then read at that same scaled base plus 116, becoming a new retained base.

A later unsigned result shifted right by 20 is stored at the new base plus 36. A full word at the new base plus 116 is supplied to a dependency.

A selector write using the new base plus one stores the low byte of an earlier result masked to 30 bits and shifted right by 20. A later, fresh selector read at that address directs a zero byte write.

Two unsigned byte reads occur at offset 21 from **separately obtained secondary-result occurrences**. Each loaded value is supplied to dependencies. Their bases and accessed addresses may be equal; neither inequality nor non-aliasing is established.

Interspersed zero selector writes use one plus 1024 times the retained right-shift-by-20 value. A third unsigned byte read at offset 21 from another secondary-result occurrence is later zero-extended into a word store at four times a further secondary result plus 20.

Additional stores propagate secondary results, scaled values, and an extraction masked to 29 bits, shifted right by 18, then left by 13.

### Reset clear and low-address propagation

Further selector writes and scaled stores precede the explicit reset-region clear. The region’s declaration width is unavailable; its later captured values are 32 bits.

Calls and direct stores continue after the clear. The clear does not establish that later reads return zero: dependencies or aliased accesses may change the region.

The intervening propagation includes zero word stores, scaled offset-20 stores, and full-word reads whose extracted parts become dependency arguments. A retained unsigned value is explicitly set to zero. Its later copies and sufficiently shifted derivatives remain zero within the represented direct computation.

Consequently, some selectors are read at absolute address one and some dependency arguments are zero. This does not make memory at address one zero. One unsigned byte read produces a subsequent byte-read address from one through 256 inclusive. A later byte read at absolute address one is zero-extended into a scaled word store.

### Final field propagation and first reset capture

Later operations clear a byte at a scaled-result address plus nine. A retained base then receives its own low byte at offset nine. A full word at that base plus 52 is supplied to a dependency.

Offset-20 word reads and stores propagate values through several calculated addresses. These operations precede the **first explicit reset-region read after its clear**.

That first reset read captures a 32-bit value. Calls occur between the capture and its later use. Its logical right shift by 23 supplies dependency arguments; shifting that retained extraction right by another 23 produces zero for later direct uses. The captured reset value is **not used as an address source**.

Other retained word-derived values are stored during this interval. The reset capture must remain distinct from current reset-region contents: subsequent dependency changes do not alter the already captured value.

Selector writes, calls, and another capture of the byte at absolute address one follow. That byte is retained as a small numeric base. A dependency-produced 32-bit value is then assigned to the publication region and supplied unchanged as the explicit argument of the following invocation.

Thus the first reset capture and its shift-derived uses occur before the first publication assignment.

### Word-to-byte propagation and second reset capture

After the first publication assignment and intervening calls, a full word at the retained small base plus 116 is read, and its low byte is written at that base plus one.

Further calls produce another scaled base. A full-word read at its offset 116 is followed by a low-byte write at offset one. Calls intervene before the same address pair is used again, requiring another full-word read. A later separately obtained scaled base undergoes the same word-to-byte propagation; its loaded word is also retained for subsequent word stores.

Additional stores propagate that retained word, scaled links, and zero. A paired result supplies a low portion shifted left by two, which is stored at four times the high portion plus 20.

Only then does the **second explicit reset-region read after the clear** capture another 32-bit value. Several dependency invocations and a further word store intervene before a full word is read at this captured value plus 20.

That address uses the second captured value, not a new reset-region read. Dependencies may change the global in the meantime without changing the retained address source.

After more calls, portions of the word loaded at offset 20 are written through selectors. One write stores its low byte; another stores the low byte of its logical right shift by 22. Each uses its own selector read.

### Terminal publication and control transfer

The terminal sequence obtains a byte-width dependency result and then:

- Writes one zero byte through a secondary byte-address result.
- Assigns the publication region a 32-bit composite whose low eight bits come from the byte result and whose upper 24 bits come from a three-byte global value.
- Writes a zero halfword at the address formed by masking a retained unsigned result to 30 bits and shifting right by seven.
- Attempts an invocation through absolute address zero with three explicit zero arguments.

The three-byte value is consumed by the composite assignment after the preceding byte-zero store. Possible aliasing must not be excluded.

The halfword-zero destination ranges from zero through `0x007FFFFF`, inclusive. No alignment check or special handling of zero is represented. Normal completion depends on the terminal invocation returning.

## Side effects

Observable direct effects include byte, halfword, and word accesses; repeated selector reads; the reset clear and two later explicit reset captures; two publication assignments; and external exposure of a stack-relative address.

No explicit allocation, release, ownership transfer, file operation, synchronization, rollback, or cleanup is represented. Dependencies may perform such operations.

The stack address is locally derived. Its valid extent, dependency access rights, retention behavior, and lifetime after the call are **documented_only**. No ownership grant or retention prohibition is established.

The supplied global-overlap warning prevents assuming independent global storage. Direct calculated addresses may also alias globals, selectors, other fields, or stack storage. Atomicity, thread safety, reentrancy, and stable memory across calls are not established.

## Dependencies

These opaque IDs identify roles, not verified distinct implementations.

| ID | Role |
|---|---|
| F-000000000001 | Initial invocation through the incoming link-derived target. |
| F-000000000002 | Ordered family of internally derived targets consuming results, shifts, repeated arguments, and addresses. |
| F-000000000003 | Separately referenced invocation receiving the same retained word twice. |
| F-000000000004 | Terminal attempted invocation through address zero. |

F-000000000002 includes many call sites; it does not assert a common target or shared behavior.

Primary words, paired portions, byte-width results, and secondary machine results are consumed. Missing secondary-result declarations in a dependency interface do not justify replacing those values with zero.

Target identity, argument completeness, result conventions, global preservation, ownership, exceptions, and nonreturning behavior remain **documented_only**.

## Acceptance examples

These are conditional static obligations, not executed tests.

1. **Initial truncation.** With first input `0x12345678` and initial selector five, the first write stores `0x78` at absolute address 22.

2. **Entry ordering.** If the initial invocation and all four following invocations return, the second entry-selector write occurs before the retained return value is shifted left by two and passed onward.

3. **Fresh selector.** If those calls change the entry selector from five to nine, the second entry-selector write targets address 26, not 22. For first input `0xFFFFFFFF`, it stores `0xFF`: the shift produces `0x3FF`, followed by byte truncation.

4. **Potentially coincident offset-21 reads.** If the first two relevant secondary-result occurrences both equal 100, both unsigned byte reads access address 121. Intervening effects may make their loaded values differ.

5. **First reset capture.** If the first reset read captures `0xFFFFFFFF`, its right-shift-by-23 extraction is 511, and the further right shift by 23 is zero. Neither value establishes a reset-derived memory address.

6. **Second reset capture across mutation.** If the second reset read captures 4096, and a dependency then changes the region to 8192, the later word read still uses address 4116.

7. **Full-width read.** If a post-publication offset-116 read yields `0xA1B2C3D4`, the following offset-one byte write stores `0xD4`. A byte-only read does not satisfy the required access width.

8. **Terminal effects.** If the final retained masked value is `0x00000100`, the zero halfword is written at address two before the zero-address invocation. If that target faults, no represented recovery rolls back earlier completed writes.

## Uncertainties

The following remain **documented_only**:

- Intended operation and logical data structures.
- Hidden entry-state ABI and secondary-result conventions.
- Actual dependency identities, complete arguments, and contracts.
- Signed arithmetic and signed-shift machine semantics.
- Alignment, endianness, permissions, and fault behavior.
- Reset declaration width and global overlap relationships.
- Aliasing and dependency mutations between accesses.
- Stack-area extent, retention, and ownership behavior.
- Whether the terminal target traps, transfers control, returns, or is callable.

No semantic error codes, retries, validation policy, transactional guarantees, or cleanup guarantees are established.

## Evidence

The contract uses only the supplied static representation: its four-position interface, explicit access widths, shifts and masks, dependency-result uses, ordered memory effects, global assignments, and terminal invocation.

The entry ordering is supported by the initial invocation and four subsequent invocations preceding the second entry-selector write. The reset chronology is supported by two explicit post-clear captures: the first feeds shifts before publication; the second supplies a retained offset-20 address after publication. Separate secondary-result occurrences support separate offset-21 reads, not unequal addresses.

Repeated offsets support access-pattern descriptions, not structure names or ownership claims. The straight-line representation establishes direct ordering, not guaranteed completion. No runtime execution, dependency inspection, ABI validation, or independent instruction-level verification has been performed.
