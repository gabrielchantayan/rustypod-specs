<!-- Trusted provenance: F-0bf63dced320; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0bf63dced320

## Purpose

F-0bf63dced320 is represented as a machine-state-dependent routine that conditionally reads and modifies memory, invokes unresolved dependencies, constructs a descriptor on one route, and enters a potentially unbounded loop on another.

No reliable application-level purpose is established. Describing it as a parser, allocator, or dispatcher would exceed the evidence.

**Established static evidence:** two explicit input slots, a nominal 64-bit result, dependencies on incoming registers and carry state, 8-/16-/32-bit memory accesses, stack-slot writes, indirect invocations, a software interrupt, and incomplete control-flow recovery.

**Inferred intent:** some operations appear to manipulate records and prepare a descriptor for another component. These labels describe access patterns, not established object types.

**Unknown:** whether every represented region belongs to one valid callable routine, whether the entry boundary is correct, and whether the nominal result matches the executable ABI.

## Interface

### Inputs and arithmetic

The explicit inputs are:

- An uninterpreted 32-bit first input, with no visible explicit use.
- A signed 32-bit second input, reduced by 12 before the initial memory gate.

Implicit inputs include the incoming carry flag, return-link value, additional registers, stack contents, and global storage. Additional return-register values from dependencies also affect subsequent addresses and data.

The descriptions below use these width conventions:

- Arithmetic on words retains the low 32 bits unless stated otherwise.
- Signed comparisons and arithmetic right shifts interpret those bits as signed 32-bit values.
- Logical right shifts insert zeros.
- Byte and unsigned-halfword loads zero-extend.
- Byte and halfword stores retain the low 8 or 16 bits.
- “Overflow” means signed subtraction overflow; “carry” means unsigned addition carry.
- Address arithmetic is represented as 32-bit arithmetic. Valid address mapping is a separate precondition.

Semantic labels such as **selector**, **record address**, and **working value** are descriptive roles, not recovered interface names.

### Result

The nominal result is 64 bits. Returning routes either forward an unresolved invocation’s result or combine two 32-bit words:

- High word: a global word read at the return region, minus 84.
- Low word: a route-specific value shifted left once.

The early loop return uses the low word of a particular dependency result. The global-zero loop exit instead uses an additional return-register value arithmetically shifted right by 9 before the left shift.

Whether the result is an integer, a register pair, or another ABI-specific object remains unknown.

### Preconditions and ownership

Every reached memory access requires a valid mapping for its width. No bounds checking, alignment repair, access-failure handling, or rollback is established.

No allocation or ownership contract is visible. Dependency-returned addresses must not be presumed allocated, owned, retained, or released by this routine. Dependencies must not be presumed to preserve globals, stack storage, or other memory without separate contracts.

## Behavior

### Entry and initial halfword gate

With incoming carry clear, the routine invokes an unresolved target derived from the incoming return link plus `0xD58`. Recovery then ends at invalid instruction data. No normal return or defined exception is established.

With carry set:

1. Reduce the second input by 12.
2. Multiply an incoming register by 256, with 32-bit wrapping.
3. Read a halfword at the sum of that product and the adjusted input.

If the halfword is nonzero:

1. Arithmetically shift the wrapped product right by 29; call this the **gate component**.
2. Read a destination-base word at gate component minus 57.
3. Read another word at twice the gate component, plus another incoming register, plus 138.
4. Invoke an unresolved dependency with gate component minus 62, the second loaded word, a stack address at represented offset `0x2A4`, and 67.
5. Invoke another dependency with the preceding result’s low and high words and fixed address-valued word `0x08572BC4`.
6. Store an additional return-register word at destination-base offset 4.

Recovery then ends at invalid instruction data.

### Selector classification

For a zero initial halfword, form:

- A **signed selector**: adjusted input arithmetically shifted right by 25, minus 62.
- A **doubled selector**: twice another incoming register, interpreted unsigned.

The signed selector lies between −126 and 1. The doubled selector is necessarily even.

For doubled selectors below 113, the represented subtraction-overflow test against signed selector minus 116 cannot succeed under these width rules. Consequently, its nested special invocation and alternate descriptor entry are not established reachable behavior.

For doubled selectors at least 113:

- No overflow in signed subtraction of 113 selects extended record processing.
- Overflow selects common byte processing.

The latter condition holds for bit patterns `0x80000000` through `0x80000070`; only even members are attainable.

### Common byte processing

The byte-address calculation uses a displacement component of 104 on the below-113 route and −4 on the overflow route. Both calculations also subtract 70. Thus the actual byte addresses are:

- Doubled selector plus 34 on the small-selector route.
- Doubled selector minus 74 on the overflow route.

The represented sequence is:

1. Read and capture the unsigned byte at that address.
2. Invoke an unresolved dependency returning a nominal 64-bit value.
3. Save its high word into the represented stack slot at offset `0x124`.
4. Multiply its low word by doubled selector plus `0x08572BD3`; capture the wrapped product.
5. Invoke another dependency.
6. Using an additional return-register value, read a word at twice that value and double the word. Call this the **first doubled word**.
7. Form the destination from the first doubled word arithmetically shifted right by 21, plus the captured byte, plus 53.
8. Read the source word again and store its doubled value at that destination.

The destination and continuation depend on the first source read; the stored value depends on the separately represented second read. Stability between reads is not guaranteed.

Bit 20 of the first doubled word selects the continuation.

If set, invoke one dependency with a halfword read at an incoming-register-derived address plus 4, then another with a byte read at the captured product minus 60. Read a word at captured byte plus 89, add the second invocation’s additional return-register value, and store that register value’s low halfword at the resulting address. Recovery then ends.

If clear, descriptor construction follows with:

- Current record-like address equal to captured product minus 61.
- Current working value equal to captured byte plus 53.

### Descriptor construction

Before its opening invocation, this route saves the current working value into represented stack slot `0x128`. It then invokes an unresolved dependency with fixed address-valued argument `0x08572B58`.

The following order is significant:

1. Read and capture a global base word.
2. Read a byte at current record-like address plus 18.
3. Read a byte at offset 13 from a pointer held in incoming stack slot `0x1E4`.
4. Read the global base again.
5. Store a zero byte at the second global value plus the captured record byte.
6. Invoke F-e87e2dd9e3e1 with:
   - Stack-supplied pointer arithmetically shifted right by 20, minus 56;
   - Captured record byte;
   - First, captured global base;
   - Captured stack-derived byte.
7. Invoke another unresolved dependency.
8. Read an unsigned halfword at offset 44 from its additional return-register value.
9. Zero-extend that halfword to obtain the descriptor address.
10. Invoke another unresolved dependency.
11. Populate the descriptor in the following order.

| Offset | Width | Value |
|---|---:|---|
| 100 | 32 bits | Fixed address-valued word `0x0857266C` |
| 0 | 32 bits | Additional return-register value from the latest invocation |
| 4 | 32 bits | First, captured global base |
| 8 | 32 bits | Current record-like address minus 62 |
| 12 | 32 bits | `0x0857266C` |

The zero-byte store does not necessarily use the first captured global base. The direct dependency and descriptor offset 4 do use that captured value.

Finally, an unresolved target receives descriptor address plus 15, current record-like address minus 62, and two zero arguments. Its nominal 64-bit result is returned unchanged.

The descriptor address is between 0 and 65,535. This does not establish that its storage is mapped or suitably aligned.

### Extended record processing

Dependency boundaries are part of the represented behavior:

1. Invoke two unresolved dependencies.
2. Read the record address from the second invocation’s additional return-register value plus the signed selector.
3. Invoke another dependency, then write the signed selector’s low byte at record offset 11.
4. Invoke another dependency, then capture the byte at record offset 15.
5. Invoke another dependency, then write its additional return-register byte at record offset 19.
6. Invoke another dependency to obtain a base.
7. Read a word at that base plus the captured byte, used as an unscaled displacement.
8. Invoke another dependency.
9. Invoke another dependency and capture its returned **working value**.
10. Test signed subtraction overflow of the loaded word minus the record address.

These invocations cannot be collapsed across the accesses: their effects and preservation contracts are unknown.

### Extended subtraction-overflow route

Double the working value with wrapping and retain it for a later store. Let the **intermediate** be the captured record byte plus 18, ranging from 18 through 273.

Invoke another dependency with the working value and that value arithmetically shifted right by 25. Its result supplies a **new base**.

Then:

1. Store loaded word minus record address at new-base offset 100.
2. Read an unsigned word at new-base offset `0x07000000`.
3. Compute a threshold indicator: one if the new base exceeds `0xF8FFFFFF`, otherwise zero.
4. Test carry from intermediate plus the read word, followed by carry from adding the threshold indicator to that wrapped sum.

If either addition carries:

1. Invoke a dependency, then capture a byte at new base plus `0x07000008` plus intermediate.
2. Invoke another dependency, then store the retained doubled working value at new-base offset `0x07000054`.
3. Invoke another dependency, then store the captured byte as a zero-extended word at new-base offset `0x07000044`.
4. Invoke another dependency, then store the low byte of new base plus `0x07000008` at captured byte plus intermediate.
5. Invoke another dependency; pass its returned value minus 166 to the next dependency.
6. Invoke another dependency, then issue software interrupt 74.
7. If execution continues, invoke an unresolved dispatch target with a global first argument, a byte read at offset 9 from a global-derived address, a separate global argument, and the stack address at represented offset `0x12C`.

The dispatch result is returned if the invocation returns. The byte-address global must not be assumed to reuse a captured first-argument value.

If neither addition carries, store the low byte of intermediate plus the read word plus the threshold indicator at offset 17 from another global-derived base. Recovery then ends at invalid instruction data.

### Non-overflow entry and loop

Before invoking the returning dependency, compute this predicate:

- The unsigned interpretation of working value arithmetically shifted right by 9 is greater than `0xFFFFFFBE`.

The invocation receives:

1. Working value.
2. Its arithmetic-right-shift-by-9 result plus 65.
3. Loaded word minus 60, wrapped first and then arithmetically shifted right by 20.

Call its returned words the **current low word** and **current high word**.

If the predicate is false, proceed directly to the shared tail.

If true, store the record address’s low halfword at current-low-word offset 42. Then compare the record address’s sign predicate with signed subtraction overflow of current high word minus 47:

- Equality proceeds to the shared tail.
- Inequality enters the loop body.

Initial loop state uses another incoming register minus the record address as the **store value**, and the returned low word logically shifted right by 25 as the **call seed**.

Each loop body performs:

1. Invoke a dependency with call seed, current high word, and a stack-base address.
2. Invoke another dependency returning a new word pair.
3. Store the store value at current-high-word offset 62.
4. Add 10 to the current high word, wrapping; call this the **incremented value**.
5. Multiply the incremented value by 65,536, wrapping, and test the signed result.

If that product is negative, return through the constructed-result region. Its low-word source is the latest invocation’s low word, shifted left once.

Otherwise:

1. Store that invocation’s low halfword at its returned-high-word offset 60.
2. Invoke another dependency.
3. Invoke another dependency to obtain an address, then store halfword 38 at offset 2.
4. Multiply the incremented value by `0x10000000`, wrap, and arithmetically shift right by 9; call this the **shifted product**.
5. Add shifted product and incremented value, wrapping.
6. Capture an XOR-mask global.
7. Invoke another dependency with twice the latest address and the wrapped value incremented value times 65,536 minus 81.
8. XOR the previously formed sum with the captured mask to obtain the **derived address**.
9. Save the derived address into represented stack slot `0x130`.
10. Read a byte at derived address plus 1 before the next invocation.
11. Invoke a dependency with `0x085734E8`, that byte, shifted product plus 47, and an additional return-register value from the preceding invocation.
12. Store shifted product plus 47 at derived-address offset 36.
13. Invoke another dependency.
14. Form the pair used by the shared tail: low word is derived address; high word is the latest additional return-register value plus 48.

The mask remains the captured pre-invocation value even if the intervening dependency changes its global storage.

### Shared tail and returns

For the pair reaching the tail:

1. Set current high word from the pair.
2. Set the next call seed to twice the pair’s low word.
3. Compute the wrapped square of twice the current high word.
4. Invoke a dependency with the pair’s low word plus another incoming register.
5. Read another global and pass it to a second dependency.
6. Invoke a third dependency.
7. Read and test the exit global.

If the exit global is zero, arithmetically shift the third invocation’s additional return-register value right by 9, then shift left once for the result’s low word.

Otherwise, store the computed square at twice current high word plus 52. Set the next store value to that square and the next current high word to the third invocation’s additional return-register value. Repeat the loop body.

Both constructed-result exits read the result-construction global upon reaching the return region and subtract 84 for the high result word. No iteration bound or termination guarantee is established.

## Side effects

Visible effects include memory reads and writes of all three documented widths, indirect control transfers, exposed stack addresses, and the conditional software interrupt.

The represented stack writes occur:

- After the first common-processing invocation and before the next invocation: save its high result word at `0x124`.
- Before descriptor construction’s opening invocation: save the working value at `0x128`.
- Before the loop’s byte-reading invocation: save the XOR-derived address at `0x130`.

These offsets describe the supplied stack representation, not a proven allocation layout or lifetime. Exposed stack addresses make the writes potentially observable to dependencies.

No atomicity, synchronization, cleanup, or resource-release guarantee is established. Earlier writes may remain visible after a failed access, nonreturning invocation, interrupt, or recovery boundary.

## Dependencies

- **F-e87e2dd9e3e1:** directly invoked during descriptor construction with the four arguments specified above. Its result is not explicitly consumed. Memory effects, ownership, and preservation guarantees are unknown.
- **F-882dff43a182:** associated with the shared constructed-result region. A separate invocation of this function is not established.
- **Unresolved indirect dependencies:** identities and complete interfaces are unavailable. Their represented arguments, results, additional return-register values, and invocation boundaries are specified where known. No invented function identities are assigned.

The interrupt mechanism and unresolved dispatch target have no supplied contracts.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Entry gate:** carry clear selects the initial indirect invocation before any halfword-gate access. No normal return is established.

2. **Selector boundaries:** doubled selector 112 takes common processing; 114 takes extended processing without subtraction overflow. The unattainable value 113 is not a valid doubled-selector example.

3. **Overflow interval:** `0x80000000` selects common processing with displacement component −4 and final byte offset −74. `0x80000072` does not select that overflow case.

4. **Repeated source read:** if the two common-processing source reads differ, the destination and bit-20 decision use the first doubled word, while the stored data uses the second read’s doubled value.

5. **Descriptor global changes:** if the first base read yields one address and the second yields another, the zero-byte store uses the second. F-e87e2dd9e3e1 and descriptor offset 4 still receive the first.

6. **Descriptor width and order:** halfword `0xFFFF` means address 65,535, not −1. Offset 100 is written before offsets 0, 4, 8, and 12, subject to valid memory.

7. **Loop entry:** a false unsigned-shift predicate skips both the preliminary halfword store and loop body. A true predicate performs that store, but equality of the sign and overflow predicates still selects the tail.

8. **Global timing:** changing the XOR-mask global during the intervening invocation does not replace the captured mask. Changing the tail argument global during its first invocation affects the subsequent read. The exit global is tested only after the third invocation.

9. **Return distinction:** the early product-sign return doubles the latest pair’s low word. The global-zero return instead doubles the arithmetic-right-shift-by-9 result of an additional return register.

## Uncertainties

The following remain **documented_only**:

- Actual routine boundaries, calling convention, and implicit entry-state requirements.
- Reachability and fidelity of damaged or unusually reconstructed regions.
- Dependency identities, memory effects, global preservation, and ownership.
- Memory validity, alignment, endianness, lifetimes, and concurrency behavior.
- Whether distinct represented accesses correspond to distinct executable reads.
- Behavior at invalid-instruction recovery boundaries.
- Whether the software interrupt returns or changes subsequent dispatch state.
- Whether the loop terminates for any intended invocation.
- Semantic meaning of the descriptor, records, globals, fixed addresses, and nominal result.

## Evidence

This contract uses only the supplied restricted static representation and opaque dependency mapping. The representation establishes the listed access widths, represented ordering, constants, branch expressions, stack writes, and result composition.

Selector parity, selector ranges, overflow intervals, and the unreachable small-selector overflow branch are arithmetic deductions under the stated 32-bit interpretation.

Recovery warnings materially limit executable interpretation. No runtime tests have been performed; no application-level behavior, memory safety, or successful end-to-end execution is established.
