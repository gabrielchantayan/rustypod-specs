<!-- Trusted provenance: F-cfe3d0bc6560; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cfe3d0bc6560

## Purpose

F-cfe3d0bc6560 executes a fixed forward sequence of externally dependent calls, address calculations, memory reads, and memory writes. Direct effects include byte updates through byte-selected absolute addresses, indexed 32-bit stores, assignments to two global storage locations, a final 16-bit clear, and an attempted call to address zero.

No application-level purpose is established. The evidence does not justify classifying the function as initialization, parsing, allocation, cleanup, dispatch, or a supervisor operation. Apparent storage names do not establish those meanings.

This contract describes represented behavior, conditional on the accesses and dependencies completing. It does not establish a useful return value, successful termination, or valid memory mappings.

Evidence classifications used below:

- **Established static evidence:** represented operations and their data dependencies.
- **Static deduction:** numerical consequences of established widths, masks, and unsigned arithmetic.
- **Inferred intent:** interpretations such as “pointer-like traversal”; these do not establish object types or application semantics.
- **documented_only:** unresolved behavior requiring dependency contracts, machine semantics, or execution evidence.

## Interface

### Explicit inputs and output

| Input | Width | Established use |
|---|---:|---|
| First input word | Unsigned 32 bits | Its upper 13 bits become the fourth represented argument of the first call. |
| Second input word | Uninterpreted 32 bits | Supplied as both the first and second represented arguments of the first call. |

The represented interface returns no value. No status, count, pointer, or error result is established.

### Implicit inputs

The first call additionally requires:

- An entry link-state value from which its target is calculated by adding 4094.
- An entry unsigned word whose upper 13 bits become its third argument.

An additional entry value supplies the base for the first explicit byte-selector read, at offset 1.

The two explicit inputs therefore do not describe the complete entry contract. The required machine state and calling convention remain documented_only.

Later execution consumes ordinary call results, secondary machine-state outputs, and represented 64-bit results split into low and high 32-bit halves. The listing establishes their subsequent uses, but not whether each is a conventional language-level return value.

### Widths, arithmetic, and memory prerequisites

The contract distinguishes 8-bit bytes, 16-bit halfwords, 32-bit words, and 64-bit aggregates.

- Unsigned selector bytes range from 0 through 255.
- Byte stores retain the low 8 bits.
- Word stores span 32 bits.
- Unsigned right shifts are logical.
- Unsigned 32-bit calculations discard bits above bit 31.
- Masking precedes shifting where a masked field is described.

Some address calculations instead involve represented signed multiplication or signed left shifts. Their overflow and negative-value semantics are documented_only; they must not be silently generalized into defined portable signed arithmetic. Signed-byte interpretation is likewise unresolved where it affects promoted arguments or shifts.

Execution requires all reached reads, writes, and call targets to be valid under the target memory and execution model. The function does not check these prerequisites. Endianness, alignment requirements, address translation, and fault handling are unavailable.

## Behavior

### Ordering and repeated observations

The represented body is a single forward sequence without explicit branches, loops, retries, validation, or recovery.

Standalone reads, writes, and calls retain their represented order. A dependency that faults, does not return, or transfers control elsewhere can prevent all later operations. No rollback is represented.

Repeated reads are distinct observations unless the function explicitly saves a value. Intervening calls cannot be assumed to preserve selectors, globals, or other memory. Conversely, an explicitly retained value must not be replaced by a fresh read of its originating storage.

Relative evaluation order is not established for multiple reads within one call’s argument list.

### Recurring address layouts

A recurring byte update has two separate address dependencies:

1. An unsigned byte is read from a computed selector address.
2. The destination is the selector’s numeric value plus 17.

Thus the destination is an absolute address from 17 through 272. It is not a field 17 bytes beyond the selector’s source object.

Many word destinations use four times an index plus 20. Other destinations use sixteen times an index plus 20, or 256 times an index plus 20. These scale factors are observably different. They do not establish a common record stride.

Additional accesses use offsets 1, 9, 13, 14, 21, 36, 52, 68, and 116. Width and address dependencies are established; complete structures, bounds, and ownership are not.

### Opening calls and selector clears

The first call receives, in order:

- The second explicit input.
- The second explicit input again.
- The upper 13 bits of the implicit entry word.
- The upper 13 bits of the first explicit input.

Several calls precede the first explicit memory update. That update reads a selector byte at the implicit entry base plus 1 and clears the destination selected by that byte.

Following additional calls and scaled-result arguments, an unsigned result determines an **opening clear base**: its value shifted right by 18, then multiplied by 1024. A call receives four copies of that result. A selector at the opening clear base plus 1 subsequently selects a byte to clear.

After intervening calls, the function rereads the selector at that same opening clear base plus 1 and performs another clear. The selector is not retained between these two operations; their destinations can differ.

A later call contains four represented 32-bit reads from absolute address 48. These are repeated reads, not an established single read replicated into four arguments.

Further clears use a selector address derived from a secondary output shifted right by 9, plus 1. Calls separate the repeated selector reads.

### Initial indexed word updates

A secondary signed output is stored as a 32-bit value at four times that same output plus 20.

The next call uses that output and an extracted field from a word read through a **separate, later-result base**. This base is calculated from a later unsigned call result by shifting it right by 18 and multiplying by 1024. The word is read at that base plus 20, and its upper 13 bits are passed onward.

The later unsigned result is not the result that established the opening clear base. No dependency contract establishes equality between the two results or bases. Substituting the opening clear base for this word-read address changes the represented dependency.

Subsequent calls provide indices for stores that:

- Copy the earlier secondary output to another four-times-index destination.
- Store a retained four-times-index value at a sixteen-times-index destination.
- Read a word at the retained scaled value plus 20 and pass its upper 13 bits onward.
- Copy the retained scaled value to another four-times-index destination.

These are indexed memory updates, not established allocation or insertion operations.

### Packed fields and mixed-width observations

A subsequent unsigned result supplies several retained quantities: four times the result; its low 30 bits shifted right by 20; its low 30 bits shifted right by 17; and its low 30 bits shifted right by 7.

The first two quantities supply byte-store values, with low-byte truncation, through selector addresses derived from a secondary output shifted right by 9. Calls separate the writes and selector observations.

A later call has seven represented arguments, including four copies of another result, the right-by-17 and right-by-7 fields, and a retained target-related word.

A 64-bit result then contributes a retained low word and a high word passed onward. Two later clears independently read selectors at the right-by-7 field plus 1.

The next region includes:

- Passing the top four bits of a retained unsigned word to a dependency.
- Selector clears using that word shifted right by 9.
- Reading an unsigned byte at twice that word and passing four copies of the saved byte.
- A clear selected by the byte at absolute address 1.

Another retained low word supplies a word-read base. The additional offset is formed by shifting its upper-13-bit field left by 31 in unsigned 32-bit arithmetic. Consequently, the offset is zero or `0x80000000`, determined by bit 19 of the original word. The base-plus-offset calculation remains 32-bit unsigned arithmetic. The loaded word is saved and passed four times.

A later byte read at a secondary output plus 14 likewise supplies four copies of one saved value; its ABI extension is unresolved.

One call receives the same stack-relative address four times. This exposes an address, but establishes neither buffer size nor initialized contents.

Subsequently, an unsigned byte is read at the sum of a retained low word and a secondary signed output. A later 16-bit read uses the sum of that saved byte value and an upper-13-bit field. Four copies of the halfword are passed onward. Argument extension and ABI interpretation remain documented_only.

### Offset-13 updates and larger scales

An upper-13-bit quantity is stored at two distinct four-times-index word destinations.

A retained low result supplies a byte-write base. At base plus 13, the function first writes the low byte of twice the base value. After intervening calls, it writes the literal byte 4 to the same address.

Between those writes, a word is read at four times another unsigned result plus 68. After the second write, another word is read at that result’s low 30 bits shifted right by 20, plus 68. These addresses are not interchangeable.

Later operations include:

- A word store at 256 times an unsigned index plus 20.
- Calls using 64 times that index.
- An indexed store containing the index’s low 26 bits shifted right by 13.
- Repeated storage and propagation of four-times-index values through sixteen-times-index and four-times-index destinations.

### Offset-36 state and offset-21 reads

A later stage writes a 32-bit zero at sixteen times a retained index plus 36, then reads a word at sixteen times that index plus 116. That loaded word becomes a new base.

Following calls, an unsigned dependency result shifted right by 20 is stored at the new base plus 36. A word at the new base plus 116 is supplied to another dependency.

A selector at the new base plus 1 first selects a packed-field byte write and later selects a clear. The selector is reread after intervening calls.

Some calls receive an additional right shift by 21 of a quantity already obtained by shifting an unsigned 32-bit result right by 20. This argument is necessarily zero.

Three later unsigned-byte reads use offset 21 from dependency-produced bases. Their values are passed onward; the third saved byte is additionally stored, zero-extended, at a four-times-index word destination.

A secondary unsigned output contributes its low 29 bits shifted right by 18. That extracted quantity is shifted left by 13 for an indexed word store and multiplied by 1024 for selector addressing.

### Global reset and narrowed retained values

After further calls and stores, the function assigns a 32-bit zero to **global A**. This is a neutral storage designation, not an assertion that the location is unused.

Calls intervene before later reads of global A. Those reads observe the storage value at their respective read times; the explicit reset does not prove they return zero.

Separately retained quantities do support static deductions:

- A 30-bit field shifted right by 19 is at most 2047.
- Shifting that retained result right by 22 yields zero.
- Applying another 30-bit mask and right shift by 19 to that result also yields zero.
- An unsigned word shifted right by 23 is at most 511; another right shift by 23 yields zero.
- A saved unsigned byte masked to 22 bits and shifted right by 12 yields zero.

Explicitly zeroed retained quantities also remain zero through their later uses unless reassigned. These deductions do not permit replacing later mutable-memory reads with constants.

This region includes an unresolved direct call receiving the same unsigned result twice.

### Late pointer-like dependencies

A later byte clear uses a heavily scaled address plus 9. Another byte store writes a retained result’s low byte at that result plus 9. A word at the retained result plus 52 contributes a dependency argument.

Further indexed stores interleave with word reads at:

- A secondary output plus 20.
- A previously loaded unsigned word shifted right by 22, plus 20.
- Another retained loaded word plus 20.
- An address derived from a masked and shifted secondary output, plus 20.

Global A is also read and narrowed during this region. The loaded values influence subsequent stores and selectors.

“Pointer-like” describes these address dependencies only. Valid objects, container representation, bounds, acyclicity, and ownership are not established.

### Final publication and retained selector snapshot

Near the first assignment to **global B**, the function reads the unsigned byte at absolute address 1 and **saves that value**. It immediately clears the destination at the saved value plus 17.

The saved selector remains available across further calls and the first assignment to global B. That assignment stores a later unsigned result as a full 32-bit word, followed by additional calls.

The subsequent update uses the saved selector twice: it reads a 32-bit word at the saved value plus 116, then writes that word’s low byte at the saved value plus 1.

This update does not reread absolute address 1 to choose its base. Changes to address 1 by intervening dependencies do not change either address in this particular update. The word contents, however, are read after those calls and are not an earlier snapshot.

Other following updates use separately calculated bases. At one retained scaled base, the word-at-offset-116 to byte-at-offset-1 operation repeats with calls between occurrences. Each occurrence performs its own word read.

Further stores propagate a saved 32-bit word, store calculated four-times results, and write zero to indexed destinations. Another read of global A saves a base before intervening calls; a later word read uses that saved base plus 20 rather than rereading global A at that later point.

### Final writes and control transfer

After additional calls and selector writes:

- A dependency supplies a byte-sized result.
- One byte is cleared through a secondary output interpreted as an address.
- Global B is overwritten with a 32-bit packed value: the returned byte occupies the low 8 bits, and the represented 24-bit quantity at absolute address 9 supplies the upper 24 bits.
- A 16-bit zero is written at the retained low aggregate word’s low 30 bits shifted right by 7.
- Address zero is called with three zero arguments.

The represented void return is reached only if the final call returns. Calling address zero is not established to be a no-op, trap, reset, or successful termination mechanism.

## Side effects

Direct writes affect:

- Byte-selected absolute addresses 17–272.
- Computed byte locations at offsets 1, 9, and 13.
- Numerous computed 32-bit destinations.
- Global A and global B.
- A final computed 16-bit destination.

Read-modify-address dependencies and aliasing can make earlier writes affect later observations. No disjointness between selectors, globals, computed destinations, or stack storage is established.

Dependencies may additionally change memory, globals, machine state, devices, or control flow. No preservation guarantee is available.

No allocation, release, reference-count change, or ownership transfer is established. Passing an address does not prove that a dependency owns, retains, or releases it. In particular, exposing a stack address establishes no valid post-return lifetime.

There is no represented synchronization, transaction boundary, error report, or rollback.

## Dependencies

The supplied identity map resolves only **F-cfe3d0bc6560** itself. No opaque function IDs are supplied for callees. Callee identities therefore remain documented_only rather than being assigned invented IDs or exposed under source labels.

| Dependency class | Established interaction | Unresolved contract |
|---|---|---|
| Entry-state-derived target | First call with four arguments | Identity, ABI, validity, effects |
| Anchor-derived computed targets | Ordered calls with varied arguments | Identity, target aliasing, semantics |
| Secondary-output producers | Outputs feed addresses and values | Defined outputs and preservation |
| Aggregate-result producers | Low and high 32-bit halves consumed separately | ABI and language-level meaning |
| Unresolved direct target | Receives one unsigned result twice | Identity and effects |
| Address-zero target | Receives three zero arguments | Mapping, execution, return behavior |

Similar argument shapes do not prove target identity or equivalent behavior.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes prior operations complete and stated values are available.

1. **Input extraction.** A first input of `0xFFF80000` supplies 8191 as the first call’s fourth argument. A first input of `0x0007FFFF` supplies zero.

2. **Distinct early bases.** If the opening unsigned result is `0x00040000`, its clear base is 1024. If the later unsigned result is `0x00080000`, the word read following the self-indexed store uses address 2068, not 1044. Its loaded word’s upper 13 bits are passed onward.

3. **Selector range.** A selector of 255 targets absolute address 272; a selector of zero targets address 17.

4. **Repeated selector read.** If an intervening dependency changes a selector, a later represented reread uses the changed byte rather than the earlier destination.

5. **Retained final selector.** If the final saved selector is 7, the immediate clear targets address 24. Even if a subsequent dependency changes address 1 to 200 before the first global-B assignment, the snapshot-dependent update reads its word at address 123 and writes its low byte at address 8—not at addresses 316 and 201.

6. **Byte truncation.** Where four times an unsigned result is written as a byte, a result of 65 produces byte value 4.

7. **High-bit offset.** Original bit 19 selects between offset zero and `0x80000000` in the unsigned upper-field-left-shift calculation.

8. **Mutable global.** A dependency changing global A after its reset can affect a later read. A value already saved from global A is a separate snapshot and is not refreshed automatically.

9. **Packed final publication.** A final byte of `0xAB` and represented 24-bit quantity `0x123456` produce the final global-B word `0x123456AB`.

10. **Late failure.** If the final halfword store faults, the address-zero call is not reached. No visible mechanism reverses earlier writes.

## Uncertainties

The following remain documented_only:

- Application purpose and intended valid input domain.
- Exact calling convention, implicit entry-state requirements, and secondary-output validity.
- Reliability of represented function boundaries and call signatures.
- Signed overflow, signed shifts, and byte or halfword argument extension.
- Endianness, alignment, low-address mappings, and device-memory semantics.
- Aliasing among computed destinations, globals, selector storage, and stack storage.
- Dependency effects, including global modification and nonlocal exits.
- Ownership, retention, allocation, and release behavior.
- Fault handling and address-zero execution.

Annotations about removed unreachable material and overlapping global symbols do not establish additional behavior or a reliable object layout.

## Evidence

This contract is based solely on static inspection of the supplied representation for F-cfe3d0bc6560. The prior draft and review feedback are not treated as independent evidence.

Established evidence includes access widths, represented call order, standalone memory-operation order, argument dependencies, global assignments, and aggregate-result splitting. In particular, the initial indexed word read uses the later unsigned result, while the final snapshot-dependent update uses a byte retained from absolute address 1 across calls and the first global-B assignment.

Range and zero-value conclusions are static deductions from unsigned widths and masks. Application intent and unavailable dependency contracts remain unresolved.

No runtime tests, smoke runs, dependency inspection, or observed executions have been performed.
