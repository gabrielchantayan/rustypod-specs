<!-- Trusted provenance: F-ad1296a5f5cd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ad1296a5f5cd

## Purpose

F-ad1296a5f5cd specifies a non-returning sequence of conditional calls, memory accesses, persistent state updates, and repeated processing.

No application-level purpose is established. The supplied representation does not justify classifying the routine as initialization, allocation, parsing, reset, or data-structure maintenance.

This contract separates explicit access widths from assignments to globals whose declarations are absent. It also distinguishes persistent state changes from temporary calculations. Unknown global types, dependency behavior, and machine arithmetic remain **documented_only** where they prevent a complete interpretation.

## Interface

### Explicit inputs

| Input | Represented form | Established use |
|---|---|---|
| First argument | 32-bit pointer | Compared with a hidden incoming pointer; replaced with a stack-relative address on outer-cycle continuation. |
| Second argument | 32-bit value | No explicit use. |
| Third argument | Signed 32-bit integer | Supplies the initial low address word and contributes to the initial offset; replaced with zero on outer-cycle continuation. |
| Fourth argument | Unsigned 32-bit integer | Contributes to the initial offset and continuation addresses; later becomes persistent processing state. |

The absence of an explicit use for the second argument does not establish whether callees can observe its physical argument location.

Additional effective inputs include:

- A hidden pointer used in the entry comparison.
- A hidden return-location-like value used to select indirect call targets.
- A hidden signed 32-bit contribution consumed after F-U02.
- A stack-base-like execution value.
- Memory at all reached addresses.
- Any additional state observed by dependencies.

### Outputs, termination, and ownership

There is no explicit result or normal return in the supplied control flow. Execution repeats, remains inside a dependency, encounters a breakpoint-like operation, or fails during a reached operation.

Several dependencies produce represented 64-bit results. Their low and high 32-bit halves are subsequently used separately. This does not establish a source-level aggregate return type or calling convention.

No allocation, ownership transfer, release, lifetime extension, or pointer validation is established. Reached memory must support the indicated accesses for execution to continue. Such validity is a precondition, not a check performed by this routine.

### Persistent processing state

The following descriptive names identify distinct state fields, not source identifiers:

| State | Initialization before processing | Subsequent updates |
|---|---|---|
| **Accumulator** | Zero, then increased by the hidden contribution after F-U02 | Replaced with four times F-U03’s high-word pointer on the extended path; further increased on the zero-selector path. |
| **Address word** | `0x0897F1F8` | Replaced with the cursor shifted left by eight on the extended path; replaced again after a resumed nonzero-selector breakpoint. |
| **Secondary word** | Eight, before F-U02 | Replaced with the address word shifted left by two on the extended path; replaced with a fetched word at every common-tail completion. |
| **Cursor** | Global-derived value converted to unsigned 32 bits and then to a pointer | Replaced by a pointer fetched during every common tail. |
| **Fourth-value state** | Fourth argument, then an unsigned byte read after F-U02 | Receives the secondary word during every common tail. |
| **Next-call argument** | Zero | Receives a fetched word logically shifted right by three at common-tail completion. |

The secondary word and address word are independent. The short processing path does not update either before entering the common tail.

## Behavior

The sequence below specifies represented ordering, not hardware synchronization, atomicity, or memory barriers. Later operations require earlier calls to return and earlier accesses to succeed.

### Outer-cycle entry

Compare the hidden incoming pointer with the current first argument.

If unequal, select F-U01 using the current hidden return-location-like value plus `0x2E`, replace that hidden value with `0x0897F193`, and invoke the selected target. No explicit arguments are shown. If equal, skip this call.

Then establish:

- Accumulator zero.
- Address word `0x0897F1F8`.
- An initial 64-bit pair with high word `0x0897F1F8` and low word equal to the third argument’s bits.
- An offset obtained from the fourth argument, third argument, and five.

The offset mixes unsigned and signed operands before assignment to a signed 32-bit slot. Its out-of-range interpretation is not independently established.

### Initial memory accesses and branch

Using the initial pair’s low word:

1. Store the offset as 32 bits at that word plus `0x0C`.
2. Interpret the low word as signed 32 bits and right-shift it by the low eight bits of that word.
3. Determine whether subtracting one from the shifted signed 32-bit value would overflow.
4. Read a signed 32-bit word at twice the shifted value.
5. Assign that loaded value, explicitly converted to 16 bits, to storage associated with F-422afe36d401.
6. Store `0x0897F1F8` as 32 bits at the loaded value plus four.

The subtraction supplies the branch condition; its numerical result is not the load address. The assignment to F-422afe36d401 is not a call. Its explicit 16-bit conversion establishes the assigned value’s narrowing, not an otherwise absent declaration of the destination’s storage type.

If overflow is indicated, take the outer-cycle continuation below.

Otherwise:

1. Store byte `0xD8` at the initial pair’s high-word address.
2. Set the secondary word to eight.
3. Select F-U02 using the current hidden return-location-like value plus `0xE56`.
4. Replace that hidden value with `0x0897F1D7`, then call F-U02 with zero.
5. Increase the accumulator by the hidden signed 32-bit contribution.
6. Read the global associated with `0x0897F216`, convert its value to unsigned 32 bits, and establish the cursor from that value.
7. Read an unsigned byte at F-U02’s low result word plus `0x0897F1F8`; replace the fourth-value state with that byte’s zero-extended value.

The global read in step 6 has no established storage width or signedness. The explicit conversion does not make it an unsigned 16-bit read.

The next comparison tests the earlier overflow indicator against zero. On this reached path, that indicator is already false, so the represented comparison succeeds. Read an unsigned 16-bit value at F-U02’s low result word plus the offset. Zero-extend it to an address, store the accumulator there as 32 bits, set the next-call argument to zero, and enter processing.

Thus, in the explicit control flow, a returning non-overflow path enters processing rather than performing another initial memory iteration.

### Outer-cycle continuation

The overflow-selected continuation performs these operations in order:

1. Store `0x0897F1EC` as 32 bits at the stack-base-like value minus four.
2. Multiply the fourth-value state by eight; store the 32-bit product at the address equal to that product.
3. Store 32-bit zero at an address formed by shifting the fourth-value state left by 17, interpreting the result as signed, right-shifting it by 32, and adding `0x10`.
4. Assign `0xC470` to the global associated with `0x770A5FC4`.
5. Assign `0x8000` to the global associated with `0xFE238028`.
6. Replace the fourth-value state with `0x0897F228`.
7. Assign the address value `0x225FC470` to the global associated with that address.
8. Replace the third argument with zero.
9. Assign `0x225F` to the global associated with `0x0000229F`.
10. Read the global associated with `0x00000022`, right-shift its value by 16 under that global expression’s unresolved type rules, and store the resulting conversion as signed 32 bits at the stack-base-like value plus `0x54`.
11. Replace the first argument with the stack-base-like value plus `0x14`.
12. Decrease the stack-base-like value by four.
13. Restart the outer cycle.

The assignments in steps 4, 5, 7, and 9 do not establish destination widths. Step 10 does not establish a signed 32-bit source read or arithmetic right shift. The explicit destination access in that step is 32 bits.

The shift by 32 in step 3 remains unresolved; subsequent effects are conditional on execution reaching them.

### Processing entry and short path

At each processing entry:

1. Preserve the low byte of the fourth-value state.
2. Set the call base to `0x0897EDF1`.
3. Call F-U03 with the next-call argument.
4. Retain its high result word as the **entry high pointer**.
5. Multiply its low result word by 32 and store that 32-bit product at the address equal to the product.
6. Compare the product with `0x14` as unsigned.

Under modulo-32-bit multiplication, the product is a multiple of 32. Therefore, its only possible value below `0x14` is zero. The store at address zero precedes the comparison in that case; there is no preceding null check.

For a product below `0x14`, select zero as F-U06’s argument and enter the common tail. The accumulator, address word, and secondary word remain unchanged on this short path.

### Extended processing path

For a product at least `0x14`:

1. Store the address word as 32 bits at the product plus `0x58`.
2. Store the cursor’s low 16 bits at the address word plus `0x0E`.
3. Replace the secondary word with the address word shifted left by two.
4. Assign the preserved byte value to a separate global whose declaration is absent.
5. Call F-U04 with the cursor logically shifted right by two.
6. Replace the call base with `0x0897EE1D`.
7. Call F-U05.

The global in step 4 is not established to occupy one byte merely because the assigned value is byte-sized.

Treat F-U05’s result as an **extended low address** and an **extended high address**. Perform these stores in order:

1. Preserved byte at twice the extended low address.
2. Low byte of the extended high address at the extended low address plus eight.
3. Entry high pointer, as 32 pointer bits, at twice the current accumulator.
4. Complete 64-bit F-U05 result at the extended low address.

Only after these stores:

- **Replace the accumulator** with four times the entry high pointer.
- **Replace the address word** with the cursor shifted left by eight.
- Store three consecutive 32-bit words through the entry high pointer:
  - Secondary word.
  - Newly updated address word.
  - Unshifted cursor bits.

These replacements persist into the common tail and later processing iterations. In particular, a later extended iteration’s store through twice the accumulator uses the accumulated state left by earlier iterations, until that iteration replaces it again.

Form a selector from the cursor by a signed left shift by 31, a signed right shift by 32, conversion to unsigned, and logical right shift by 20. The selector’s value is unresolved without the applicable shift and arithmetic semantics.

If the selector is zero:

- Assign zero to the separate global.
- Increase the accumulator by the secondary word.
- Store that updated accumulator as signed 32 bits through the extended high address.
- Select zero as F-U06’s argument.

The updated address word remains the cursor shifted left by eight.

If the selector is nonzero:

- Store the selector as 32 bits at the secondary word multiplied by `0x100`, plus `0x20`.
- Replace the call base with `0x08790D52`.
- Select an argument obtained by interpreting the secondary word multiplied by `0x100` as signed 32 bits and right-shifting it by 29.
- Execute a breakpoint-like operation with immediate four.
- If execution resumes, **replace the address word** with the extended high address interpreted as signed 32 bits and right-shifted by 18.

The accumulator remains four times the entry high pointer on this nonzero-selector path.

### Common tail and repetition

The common tail uses whichever address word, accumulator, secondary word, cursor, and call base the selected path leaves behind.

1. Call F-U06 with the selected argument.
2. Store its low result word’s low 16 bits at the address equal to that low word.
3. Multiply the address word by `0x40`; store that 32-bit product at the address equal to the product.
4. Read a 32-bit unsigned word through the current cursor; retain it as the **fetched word**.
5. Store the low 16 bits of the cursor-address-plus-four value at the address word plus four.
6. Replace the cursor with a 32-bit pointer read at F-U06’s high result word plus `0x50`.
7. Call F-U07, then F-U08, with no explicit arguments shown.
8. Replace the fourth-value state with the current secondary word.
9. Call F-U09 with the fetched word.
10. Replace the next-call argument with the fetched word logically shifted right by three.
11. Replace the secondary word with the fetched word.
12. Repeat processing.

Step 8 does not necessarily copy a shifted value. On the first short path it copies eight. On an extended path it copies the address word’s earlier value shifted left by two. On a later short path it copies the fetched word retained from the preceding tail.

## Side effects

Explicit pointer accesses include byte, 16-bit, 32-bit, and 64-bit stores, as well as byte, 16-bit, 32-bit, and pointer reads. Assignments to undeclared globals have unresolved storage widths.

Addresses may be fixed, computed, stack-relative, low-valued, unaligned, or associated with function storage. No disjointness, alignment, bounds, memory-map classification, or endianness guarantee is supplied. The overlap warning further prevents treating named globals as independent storage.

There is no rollback. Earlier writes may remain observable after an invalid access, nonreturning call, or breakpoint. No explicit validation, error result, handler, cleanup, or memory release is shown.

## Dependencies

These document-local opaque IDs identify call sites, not necessarily distinct runtime functions.

| ID | Target selection and explicit input |
|---|---|
| F-U01 | Hidden return-location-like value plus `0x2E`; no explicit input. |
| F-U02 | Current return-location-like value plus `0xE56`; zero. |
| F-U03 | Represented target `0x07F755F6`; next-call argument. |
| F-U04 | Current call base plus `0xAE`; cursor logically shifted right by two. |
| F-U05 | Internal anchor plus `0xFFE`; no explicit input. |
| F-U06 | Current call base plus `0xA1E`; selected argument. |
| F-U07 | `0x0897EDCD + 0x3C4`; no explicit input. |
| F-U08 | `0x0897EDDF + 0xFFE`; no explicit input. |
| F-U09 | `0x0897EDE1 + 0x800`; fetched word. |

F-422afe36d401 identifies storage receiving an explicitly narrowed value; no invocation is shown.

No dependency has an established purity, global-preservation, ownership, release, termination, or error contract. Calls may affect memory later read by this routine. Explicit local sequencing does not establish preservation of external or hidden state.

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Entry decision:** Equal entry pointers skip F-U01. Unequal pointers cause target selection before replacement of the hidden return-location-like value and invocation before accumulator initialization.

2. **Overflow ordering:** If the shifted value is signed 32-bit minimum, subtracting one indicates overflow. The offset store, signed word read, narrowed assignment, and following 32-bit store nevertheless precede the continuation.

3. **Destination zero-extension:** A 16-bit destination read of `0xFFFE` selects address `0x0000FFFE`, not a sign-extended address.

4. **Unknown global type:** A value read from the global at `0x0897F216` is converted to unsigned 32 bits. Acceptance must not require a two-byte read: that width is not established.

5. **Initial short path:** If F-U03’s low result word is zero and address zero is writable, zero is stored there before the short-path decision. The secondary word remains eight, and the tail copies eight into the fourth-value state.

6. **Persistent zero-selector state:** Suppose the address word is `0x100`, cursor bits are `0x20`, entry high pointer is `0x300`, and resolved selector semantics produce zero. After the extended updates, the secondary word is `0x400`, address word is `0x2000`, and accumulator is `0x1000`. The common tail uses `0x2000` for address-word-based accesses and copies `0x400` into the fourth-value state. This assumes all intervening calls and accesses succeed.

7. **Persistent nonzero-selector state:** If a nonzero selector reaches a breakpoint that resumes, the common tail uses the post-breakpoint replacement address word, not the earlier cursor-shifted address word.

8. **Later short iteration:** If one tail fetches `0x12345678`, the next secondary word is that value. A subsequent short path leaves it unchanged until its tail copies it into the fourth-value state.

9. **No rollback:** If F-U02 does not return, the preceding `0xD8` byte store has occurred, but the later global read, byte read, destination read, and accumulator store have not.

## Uncertainties

The following remain **documented_only**:

- Application purpose and validity of the represented function boundary.
- Global storage types, widths, signedness, conversions, and overlap.
- Consequently, the exact global read at `0x00000022` and its right-shift semantics.
- Shift counts at or beyond operand width, especially the two shifts by 32.
- Signed arithmetic overflow, out-of-range conversions, and address arithmetic.
- Exact calling conventions and dependency effects on external state.
- Memory mappings, alignment requirements, endianness, and access atomicity.
- Breakpoint handling and whether execution resumes.
- Pointer provenance, ownership, validity, and lifetime.
- External conditions capable of ending the cycles.

No portable behavior may be selected silently for these unknowns.

## Evidence

Established static evidence consists of the supplied interface, explicit pointer access types, constants, result-half extraction, branches, calls, and state assignments.

Undeclared global assignments establish values or expressions being assigned, not destination storage widths. Explicit persistent assignments establish that the accumulator, address word, and secondary word change as specified; these are not merely temporary derived values.

The ordering and acceptance criteria follow from the supplied representation, subject to the documented arithmetic and dependency uncertainties. No application-level intent is established. No runtime tests, execution traces, memory-map inspection, or independent implementation verification have been performed.
