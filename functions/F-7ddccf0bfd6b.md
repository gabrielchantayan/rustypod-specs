<!-- Trusted provenance: F-7ddccf0bfd6b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7ddccf0bfd6b

## Purpose

F-7ddccf0bfd6b performs an ordered sequence of external calls, mixed-width memory accesses, scaled-address stores, and shared-state assignments. Its last visible action is an attempted call to address zero.

No application-level purpose is established. Encoded-reference manipulation and link maintenance are possible interpretations of the repeated scaling and offset patterns, but remain **inferred intent**, not contractual object models.

This specification describes the supplied static representation. It does not establish executable calling conventions, dependency behavior, memory mappings, or successful completion. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Representation | Established use |
|---|---|---|
| First argument | Opaque 32-bit value | No explicit use in the supplied body. |
| Second argument | Unsigned 32-bit value | Supplied twice to the entry call; used for a byte read at offset 0x0E and selector reads based on its logical right shift by nine. |

The first argument’s apparent nonuse does not prove irrelevance to the actual calling convention or to dependencies observing incoming machine state.

### Additional environmental inputs

The body also consumes incoming execution state that provides:

- A word shifted logically right by nineteen for the entry call.
- A word supplying the first stored byte.
- An address base for the first selector read.
- A value from which the entry call target is derived.
- Secondary machine-register outputs after numerous dependencies.

These are effective inputs beyond the displayed arguments. Their origins and preservation rules are **documented_only**.

### Outputs and completion

No explicit result value is shown. Observable results are calls, reads, writes, and shared-state changes.

The terminal call targets address zero and receives three zero arguments. The displayed return is reachable only if that call returns. Whether the call traps, dispatches through mapped memory, or behaves otherwise is **documented_only**.

### Preconditions and errors

Completion requires every preceding dependency to return and every memory access to succeed. Derived addresses must support their access widths and alignment; derived call targets must support execution under the actual calling convention. This includes low-numbered addresses.

No explicit validation, error-status branch, rollback, or recovery is shown. A fault or nonreturning call can leave any already-completed prefix of mutations visible.

## Behavior

### Widths, arithmetic, and access forms

Direct data accesses are 8-bit bytes, 16-bit halfwords, and 32-bit words. Several call results are represented as 64-bit quantities split into low and high 32-bit portions. That split is established in the representation; the actual return convention is not.

Unsigned right shifts are logical. Byte stores retain only the low eight bits. For example, masking a word to thirty bits and shifting right by twenty produces at most ten bits, but a subsequent byte store retains only original bits twenty through twenty-seven.

Where explicitly described as **32-bit bitvector arithmetic**, addition, scaling, and left shifts discard bits above bit thirty-one. Signed multiplications and signed left shifts also occur in the representation. Their executable overflow and shift semantics remain **documented_only**; portable-language rules must not be substituted without machine evidence.

Recurring access forms are:

| Form | Observable access |
|---|---|
| Selector-mediated byte store | Read an unsigned byte at a derived base plus one; store a byte at the selector’s numeric value plus 0x11. |
| Four-scaled word slot | Store a word at four times an index plus 0x14. |
| Sixteen-scaled word slot | Store a word at sixteen times an index plus 0x14, frequently containing four times that index. |
| Other fields | Access offsets including 0x09, 0x0D, 0x0E, 0x15, 0x24, 0x34, 0x44, and 0x74 from separately derived bases. |

A selector-mediated destination ranges from 0x11 through 0x110. The selector is a byte-valued integer, not a pointer-sized field.

Repeated selector expressions normally require fresh represented reads. Dependencies between them may change memory. An explicitly captured selector retained across calls is different; the publication sequence below contains such a case.

No common structure layout, bounds, ownership, or nonaliasing relationship is established by these offsets.

### Entry and initial byte updates

The entry dependency receives the second argument twice and the incoming context word shifted right by nineteen. Four further calls precede the first direct mutation.

That mutation reads a selector at the incoming address base plus one, then stores through the selector at offset 0x11. Its value is the low byte of an incoming word masked to thirty bits and shifted right by twenty.

Later calls receive dependency results scaled by four. A byte read at the second argument plus 0x0E is captured and supplied in four argument positions.

After an apparent two-word result, a selector-mediated zero store uses the second argument shifted right by nine as its selector base. The high result portion and a right-by-nineteen slice of the low portion are subsequently forwarded. Another zero store uses a fresh selector read from the same derived address after intervening calls.

A later unsigned result supplies a base formed by shifting right by eighteen and multiplying by 1024. The original result is supplied in four positions to another call. Two selector-mediated zero stores use the derived base, separated by calls.

The next fixed-location input requires a distinct rule: a call contains **four separate 32-bit global-read expressions at address 0x30**, one for each of four argument positions. There is no explicit capture followed by four uses. Argument evaluation order and the executable’s physical load count are unestablished. Equal argument values or a shared snapshot must not be required without additional environmental guarantees.

Further selector-mediated zero stores use saved secondary output shifted right by nine, with fresh selector reads.

### Initial word-slot mutations and bit slices

The next phase obtains a result, makes calls including four-zero-argument calls, and performs these ordered mutations:

- Store a secondary output into its own four-scaled slot.
- Read a word at the saved result’s right-by-eighteen, 1024-scaled base plus 0x14; supply its right-by-nineteen slice to a call.
- Store the saved secondary output into another dependency-selected four-scaled slot.
- Store four times a later result into that result’s sixteen-scaled slot.
- Read a word at the saved four-scaled value plus 0x14 and forward its right-by-nineteen slice.
- Store the saved four-scaled value into another four-scaled slot.

Dependencies intervene throughout. These accesses do not constitute an atomic update.

A later unsigned result supplies a four-scaled value and thirty-bit-masked slices. First, its four-scaled low byte is written through a selector based on saved secondary output shifted right by nine. After calls, its masked right-by-twenty low byte is written through a newly read selector at the same derived selector address.

Masked slices shifted right by seventeen and seven are also retained for a later call with seven displayed arguments. After another apparent two-word result, two selector-mediated zero stores use the saved right-by-seven slice as their base, again separated by calls.

### Mixed-width reads and high-bit address derivations

Subsequent calls consume saved result portions, scaled values, and the top four bits of a saved unsigned word. Selector-mediated zero stores use that word shifted right by nine.

A byte is read at twice the saved word’s numeric value and supplied in four positions. A separate selector-mediated zero store reads its selector from address one. A call later receives the captured byte shifted left by thirty-one. Under 32-bit bitvector arithmetic, only the byte’s least significant bit survives, in bit thirty-one.

A later saved word supplies a word-read address consisting of itself plus a displacement formed by shifting it right by nineteen, then left by thirty-one. Under that arithmetic model, the displacement is zero or 0x80000000 according to original bit nineteen.

Further operations include:

- A byte read at a secondary output plus 0x0E, forwarded in four positions.
- Additional apparent two-word results.
- Selector-mediated zero stores using unshifted bases, right-by-nine bases, and a saved low word multiplied by 4096.
- A call receiving the same stack-relative address in four positions.
- A selector read whose base is a saved right-by-nineteen slice multiplied by 0x80000000; under 32-bit bitvector arithmetic, its address is either one or 0x80000001.
- Selector reads using saved low words shifted right by nineteen or nine.

The stack-relative address exposes current stack storage to a dependency. Object size, initialization, and retention are not established.

Another byte read uses a saved low result word plus a secondary output. The captured byte is supplied in four positions. A later 16-bit read uses that byte plus another secondary output shifted right by nineteen. The halfword is supplied in four positions before another apparent two-word result.

### Field changes and larger-stride slots

The halfword-related result leads to two word stores of a saved right-by-nineteen value into separate dependency-selected four-scaled slots.

At a saved low-result base plus 0x0D, the function writes the low byte of twice that base. After calls, it writes constant four to the same saved address.

Two distinct word reads at offset 0x44 feed dependencies: one base is four times a saved result; the other is that result masked to thirty bits and shifted right by twenty.

Later mutations include:

- A word store at 256 times a result plus 0x14, containing four times a saved value.
- A four-scaled slot store containing a twenty-six-bit-masked, right-by-thirteen slice.
- Repeated sixteen-scaled slot stores containing four-scaled values.
- Subsequent four-scaled slot stores propagating those saved values.

Each occurrence remains an ordered access, including when addresses alias.

A later sixteen-scaled base receives zero at offset 0x24, followed by a word read at offset 0x74. After calls, a new result shifted right by twenty is stored at the loaded base plus 0x24. Another word at the loaded base plus 0x74 is read and forwarded.

The saved right-by-twenty result has at most twelve significant bits. Its later logical right shift by twenty-one is therefore zero, independently of intervening dependency side effects.

### Offset-0x15 reads and the first shared-state assignment

Three separate bytes are read at dependency-derived bases plus 0x15. Calls and selector stores separate these reads. One captured byte is zero-extended into a four-scaled word-slot store.

A secondary output is masked to twenty-nine bits and shifted right by eighteen. That saved slice is later shifted left by thirteen and stored in another word slot. It also supplies 1024-scaled selector bases.

Further stores propagate dependency results and scaled values. One saved secondary output is scaled by eight, then four, then four again before a later store: multiplication by 128 under 32-bit bitvector arithmetic.

The function explicitly clears **shared location A**. Later reads of A are separate accesses after dependencies; the clear does not justify replacing those reads with zero.

The following phase includes zero-valued word-slot stores, sixteen-scaled slot stores, further selector stores, and a distinct fixed-target call receiving the same value twice.

### Later byte chain and ordered link-like accesses

A byte read at a dependency-produced base plus one is captured. After intervening calls, that byte becomes the numeric base for another byte read at offset one. The second address is between one and 256.

One selector-store value formed from the captured byte masked to twenty-two bits and shifted right by twelve is necessarily zero. A separate similarly shaped expression uses a full dependency-produced word and is **not** thereby necessarily zero. Slices of explicitly zero saved words remain zero in the represented value flow.

More word-slot stores follow. Later byte mutations include zero at an address formed from a four-scaled result multiplied by 8192 plus 0x09, and the low byte of another saved base at that base plus 0x09. A word at the latter base plus 0x34 feeds a call.

Subsequent word reads at offset 0x14 produce further bases through right shifts by twenty-two and one. Stores precede subsequent reads in these sequences. Possible aliasing does not permit prefetching the reads or moving stores after them.

A later read of A is captured, shifted right by twenty-three, and then shifted right by twenty-three again. The second result is necessarily zero. The original read need not have been zero.

### Captured address-one base and first publication to shared location B

Near the first assignment to **shared location B**, the function explicitly captures the unsigned byte at address one. This is a retained numeric base, unlike selector expressions that are freshly reread.

The established order is:

1. Capture the byte at address one.
2. Store zero at the captured value plus 0x11.
3. Perform intervening calls, eventually obtaining another word result.
4. Assign that word result to B.
5. Perform two further calls.
6. Read a fresh 32-bit word at the **captured byte’s numeric value plus 0x74**.
7. Store that word’s low byte at the **same captured value plus one**.

The retained base remains between zero and 255. Its word-read address ranges from 0x74 through 0x173; its byte-store address ranges from one through 0x100.

Reloading address one after the calls would not preserve this represented behavior. Neither would reading the offset-0x74 word before publication: B’s assignment or intervening dependencies may affect that memory.

Later sequences use dependency-derived four-scaled bases to read words at offset 0x74 and write their low bytes at offset one. Two such reads from one saved base occur separately across calls and must not be merged. Another source word is explicitly captured and reused in subsequent sixteen-scaled and four-scaled word-slot stores.

Additional mutations include storing a scaled result at a saved four-aligned base plus 0x14, clearing another four-scaled slot, and propagating a low result portion scaled by four through two more slots.

### Final reads, composition, and termination

A is freshly read and retained before intervening calls. A later word read uses that retained value plus 0x14. Its low byte and right-by-twenty-two slice feed separate selector-mediated stores, with calls and fresh selector reads between them.

The terminal sequence is:

1. Obtain a byte-sized dependency result.
2. Store zero through a dependency-supplied byte pointer.
3. **After that store**, read the fixed-location 24-bit input at address 0x09.
4. Assign B a 32-bit composition with that 24-bit input in bits eight through thirty-one and the dependency byte in bits zero through seven.
5. Store a zero halfword at the address formed from a saved word masked to thirty bits and shifted right by seven.
6. Attempt the call to address zero with three zero arguments.

The preceding byte store may affect the 24-bit input through aliasing. An earlier cached value is not justified. The represented 24-bit read does not establish its physical machine-load decomposition or memory byte order.

## Side effects

Direct side effects include the described calls, mixed-width accesses, the explicit clear of A, both explicit assignments to B, and exposure of a stack-relative address.

There is no visible direct allocation, release, ownership transfer, synchronization, or rollback operation. Dependencies may nevertheless allocate, release, retain addresses, modify globals, or change reachable memory. No such behavior may be assumed absent.

No ownership relationship is established for explicit inputs, computed addresses, or dependency-produced pointers. Address validity and lifetime must hold when each access occurs, not merely when a value was captured.

The representation warns of overlapping global symbols. Distinct labels here do not establish disjoint storage from each other or from computed accesses.

## Dependencies

Only F-7ddccf0bfd6b has a supplied opaque function identity. All other dependency identities and contracts are **documented_only**; no authoritative opaque IDs are available for them.

The visible dependency surface comprises an entry target derived from incoming execution context, many targets derived from fixed control-flow anchors with differing displacements, a distinct fixed-target call, and the terminal target at address zero.

Calls have zero through seven displayed arguments. Arguments include repeated captured values, separate global-read expressions, stack-relative addresses, scaled words, and portions of apparent two-word results. Secondary register outputs also feed later operations.

No-argument calls are still observable state-mutation boundaries. Similar-looking call sites must not be treated as one dependency, as pure operations, or as preserving globals and memory without additional evidence.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Selector width.** If a selector read yields 0xFF, its selector-mediated store targets 0x110. No pointer-sized selector load is required.

2. **Fresh selectors.** If intervening calls change a selector from 0x03 to 0x07, successive fresh-selector stores target 0x14 and 0x18.

3. **Byte truncation.** For first-store source 0x3AB00000, the masked right-by-twenty value is 0x3AB and the stored byte is 0xAB, provided execution reaches the store.

4. **Four global expressions.** If address 0x30 remains 0x11223344 throughout argument evaluation, all four displayed argument expressions yield that value. If it can change, a common snapshot, argument ordering, and physical load count remain unestablished.

5. **Retained publication base.** Suppose the prepublication address-one byte is 0x03. The immediate zero store targets 0x14. Even if calls later change address one to 0x20, the postpublication word read still uses 0x77 and its low-byte store targets 0x04.

6. **Late publication read.** If the word at 0x77 changes during the calls surrounding publication, the preceding example uses the word present at the later read, not an earlier prefetched word. If that word is 0x123456A5, the byte written at 0x04 is 0xA5.

7. **Final aliasing.** If the terminal byte-pointer store aliases the 24-bit input, composition uses the input after the zero-byte store. If the resulting numeric 24-bit value is 0x123456 and the dependency byte is 0x5A, B receives 0x1234565A. An earlier 24-bit snapshot is not acceptable.

8. **Width and partial execution.** A halfword read of 0xABCD consumes sixteen bits, not merely 0xCD. If that access faults, subsequent stores and the terminal call are not guaranteed; no rollback is established.

## Uncertainties

The following remain **documented_only**:

- Actual calling convention, complete incoming state, dependency identities, signatures, and preservation rules.
- Whether apparent 64-bit results correspond to actual two-word returns.
- Executable semantics of signed shifts, signed overflow, and address arithmetic.
- Argument evaluation order and physical loads for the four address-0x30 expressions.
- Physical implementation of the final 24-bit read.
- Endianness, alignment, memory mapping, volatile or device behavior, and atomicity.
- Exact global layout, overlap, and architectural significance.
- Dependency ownership, retention, allocation, and release behavior.
- Stack-storage initialization, extent, and lifetime after exposure.
- Concurrency, reentrancy, exceptions, and terminal-call behavior.

These unknowns do not erase explicit capture boundaries or represented ordering: the publication base is retained, its offset-0x74 word is read later, and the final 24-bit input is read after the zero-byte pointer store.

## Evidence

Established evidence consists of the supplied static body’s explicit access widths, value captures, arithmetic derivations, call boundaries, stores, shared-state accesses, and terminal attempted call.

In particular, the representation distinguishes four separate address-0x30 argument expressions from explicitly saved values; explicitly retains the address-one byte across publication and calls; and places the final 24-bit read after the dependency-supplied pointer store.

Arithmetic deductions are conditional on the widths and arithmetic models stated above. Application intent, dependency behavior, and successful execution are not established.

No implementation access, dependency inspection, runtime execution, or tests were used.
