<!-- Trusted provenance: F-d1ccc55a32d7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d1ccc55a32d7

## Purpose

The available static evidence describes an ordered sequence of indirect dependency calls, five coprocessor transfers, and a final 16-bit memory write. The represented control flow then stops at an invalid-instruction boundary.

No application-level purpose is established. The sequence cannot reliably be classified as initialization, decoding, device configuration, or deliberate fault generation. Its completeness and executable validity are also unresolved.

This contract describes observable ordering and value relationships **conditionally on the supplied static representation being accurate**. It does not establish that execution can reach every operation or that the represented entry is a valid complete routine.

## Interface

### Entry and completion

The displayed interface declares no parameters and no return value. Nevertheless, the sequence consumes implicit incoming machine state:

| Input label | Represented width | Visible role |
|---|---:|---|
| A | 32 bits | Supplies two shifted arguments to F-000000000005 |
| B | 32 bits | Supplies the third shifted argument to F-000000000005 |
| C | 32 bits | Supplies an unsigned right-shifted argument to F-000000000006 |
| P | 32 bits | First coprocessor-related operand in transfers 1, 3, and 5 |
| Q | 32 bits | First coprocessor-related operand in transfer 2 |
| R | 32 bits | Second coprocessor-related operand in every transfer; also the first in transfer 4 |
| Incoming link state | Machine address-related value | Determines the first indirect-call target |

These labels identify distinct incoming dataflow roles. Distinct labels do not require distinct numeric values.

The representation does not establish a source-language calling convention, initialization mechanism, or preservation convention for these inputs. Calls without displayed argument expressions may still consume machine state.

No normal return is visible. After the final store, the representation reports an invalid instruction and truncated control flow. This is not evidence of a specific hardware exception, intentional halt, or successful completion protocol.

### Width and arithmetic interpretation

The represented dataflow uses:

- 32-bit ordinary arguments, narrow results, and arithmetic intermediates;
- three 64-bit result containers;
- one 16-bit memory store.

Arithmetic descriptions below use fixed-width bit patterns. A left shift producing a 32-bit value discards bits above bit 31. Logical right shifts insert zeros. Bit positions are numbered from zero at the least significant bit.

This interpretation captures the represented machine-level relationships. It does not assert that signed-looking expressions would be valid for every input in a high-level language.

The 64-bit containers establish low-word and high-word relationships, but not a particular language-level return type or register-pair convention.

### Preconditions for reaching the store

Reaching the final write requires every preceding indirect call to resolve, execute, and return, and every preceding coprocessor transfer to complete. The derived destination must support the attempted 16-bit write under the target’s mapping, access, and alignment rules.

These are execution conditions, not checks performed by this function. No explicit validation of call targets, input ranges, coprocessor permissions, destination accessibility, or alignment appears.

## Behavior

### Dependency references and order

F-000000000001 through F-000000000022 are opaque, site-local dependency IDs assigned in call order. They identify call occurrences, not independently recovered implementations. Multiple occurrences may reach the same implementation; a runtime-dependent occurrence may reach different implementations on different executions.

The first target is derived from incoming link state plus 3,132 bytes. The other displayed targets are derived from embedded code-location anchors and fixed offsets. These target expressions do not establish dependency signatures or meanings.

Descriptions of later operations are conditional on all earlier operations completing.

### Initial calls and first coprocessor transfer

The sequence first invokes F-000000000001 and F-000000000002. Neither has displayed arguments or a consumed result.

F-000000000003 then supplies a 32-bit result. That result becomes the data operand of coprocessor transfer 1. F-000000000004 follows the transfer without displayed arguments or a consumed result.

The complete ordered transfer contract is:

| Transfer | Position in sequence | Numeric selectors | 32-bit data operand | Ordered coprocessor-related operands |
|---|---|---|---|---|
| 1 | After F-000000000003, before F-000000000004 | (0, 6, 7) | Result of F-000000000003 | (P, R) |
| 2 | After F-000000000005, before F-000000000006 | (0, 5, 7) | Primary result shifted left by 1 | (Q, R) |
| 3 | After F-000000000008, before transfer 4 | (0, 7, 7) | Zero | (P, R) |
| 4 | After transfer 3 and zero-intermediate derivation, before F-000000000009 | (0, 0, 7) | Zero | (R, R) |
| 5 | After F-000000000009, before F-000000000010 | (0, 2, 7) | Scaled result shifted left by 8 | (P, R) |

The operand routing is established static dataflow. Unknown hardware semantics do not make these identities or their ordering unknown. In particular, transfer 2 uses Q rather than P, and transfer 4 uses R in both operand positions.

### Primary result and input transformations

F-000000000005 receives three 32-bit arguments, in this order:

1. A shifted left by 22 bits;
2. A shifted left by 11 bits;
3. B shifted left by 22 bits.

Its 32-bit result is the **primary result**.

Transfer 2 uses the primary result shifted left by one bit. A **scaled result** is then formed by shifting the primary result left by four bits.

F-000000000006 receives:

1. the primary result shifted left by one bit;
2. C logically shifted right by five bits;
3. the scaled result;
4. zero.

The second argument ranges from zero through 134,217,727 inclusive. It is not an arithmetic, sign-extending shift.

F-000000000007 and F-000000000008 follow, with no displayed arguments or consumed results.

### Zero intermediate and secondary result

Transfer 3 sends zero using operands (P, R).

Next, a **zero intermediate** is derived by retaining the lowest two bits of the scaled result and shifting that two-bit value left by 16 bits. The intermediate is necessarily zero: forming the scaled result already cleared its lowest four bits.

Transfer 4 then sends zero using operands (R, R).

F-000000000009 receives a single explicit zero argument and supplies a 32-bit **secondary result**.

Transfer 5 uses the scaled result shifted left by eight bits and operands (P, R).

F-000000000010 then receives three 32-bit arguments:

1. the scaled result shifted left by eight bits;
2. the scaled result shifted left by 12 bits;
3. the secondary result shifted left by nine bits.

The first two arguments are equivalently the primary result shifted left by 12 and 16 bits, respectively, with 32-bit truncation.

### First wide result and narrow-result chain

F-000000000011 supplies the first 64-bit result. F-000000000012 receives, in order:

1. its low 32 bits;
2. its high 32 bits;
3. its low 32 bits shifted left by one bit, truncated to 32 bits.

No result of F-000000000012 is consumed visibly.

F-000000000013 receives the zero intermediate shifted left by eight bits, so its explicit argument is zero. Its 32-bit result becomes the sole displayed argument to F-000000000014. The 32-bit result of F-000000000014 becomes the sole displayed argument to F-000000000015.

F-000000000016 follows without displayed arguments or a consumed result.

No visible condition interprets any result in this chain as success, failure, a pointer, or a handle. Returned bit patterns are forwarded without a displayed status check.

### Final wide-result chain

F-000000000017 supplies the second 64-bit result. F-000000000018 receives:

1. that result’s low 32 bits;
2. that result’s high 32 bits;
3. that result’s high 32 bits shifted left by 11 bits, truncated to 32 bits.

F-000000000018 supplies the **final wide result**.

Its low 32 bits become the sole displayed argument to F-000000000019. Its bits 41 through 63 are extracted as an unsigned 23-bit quantity for the later destination calculation.

Before F-000000000019, the earlier zero intermediate is shifted left by 25 bits. It remains zero.

F-000000000020 then supplies a 32-bit result. F-000000000021 receives that result as its first argument and the shifted zero intermediate as its second argument.

F-000000000022 follows without displayed arguments or a consumed result.

Thus **four calls occur after production of the final wide result and before the store**: F-000000000019, F-000000000020, F-000000000021, and F-000000000022. Three of those calls precede the final call; all four intervene between production and the write.

### Destination calculation and final write

Let the **address index** be bits 52 through 63 of the final wide result, interpreted as an unsigned 12-bit integer.

The destination byte address is:

**address index + floor(address index / 4)**.

This follows from the represented two-stage extraction. Bits 41 through 63 first form an unsigned value of at most 23 bits. Shifting that value right by another 11 bits selects bits 52 through 63. Although the latter shift is displayed with signed interpretation, its operand cannot have bit 31 set, so sign extension cannot affect this calculation.

The address index ranges from zero through 4,095. The destination therefore ranges from zero through 5,118, without arithmetic overflow at the represented 32-bit width.

After F-000000000022 returns, the function attempts to write the 16-bit value **0x00CD** at that destination.

Consequences of the explicit calculation:

- No base pointer is added.
- Address zero is possible.
- Odd destinations are possible.
- Some addresses within the overall range are unreachable.
- No alignment adjustment is visible.
- The write is a halfword write, not a single-byte write.
- The byte ordering of the halfword is unresolved.

The representation ends at the invalid-instruction boundary after this write. It supplies no subsequent executable behavior.

## Side effects

### Established visible effects

The explicit sequence contains:

- 22 indirect calls, in the order identified above;
- five ordered coprocessor transfers with the exact routing shown in the transfer table;
- one 16-bit ordinary data-memory store;
- a truncated continuation marked as invalid instruction data.

The store occurs after all 22 calls. Its destination depends on the result of F-000000000018, with four later calls before the write.

No other ordinary data-memory load or store is explicit in the representation. This does not exclude instruction fetches, omitted machine-level stack activity, dependency memory accesses, or coprocessor effects.

### Unknown effects of dependencies

No dependency is established to preserve global state, arbitrary registers, memory, or external resources. A call may have effects beyond its displayed arguments and consumed result.

The represented value flow across calls does not independently prove an executable register-preservation convention. Likewise, the visible P, Q, and R operand relationships describe the supplied representation; they do not establish how actual hardware state is maintained across unknown callees.

There is no visible local error recovery. If a call fails to return, a transfer faults, or the store cannot complete, later operations are not established to occur. The resulting exception, trap, or external failure behavior is unknown.

### Ownership

No explicit allocation, release, or ownership transfer is shown. Arguments may be interpreted by unknown dependencies as integers, addresses, handles, or other tokens. The contract assigns no retention, release, lifetime, or ownership guarantee to those dependencies.

Coprocessor operations are not assigned cache-maintenance, barrier, synchronization, or device-control semantics merely from their presence.

## Dependencies

| Opaque ID | Visible role |
|---|---|
| F-000000000001 | Initial call through a link-state-relative target |
| F-000000000002 | Initial call through an anchor-relative target |
| F-000000000003 | Supplies the data for transfer 1 |
| F-000000000004 | Call between transfer 1 and primary-result production |
| F-000000000005 | Receives three shifted inputs; supplies the primary result |
| F-000000000006 | Receives shifted primary result, shifted C, scaled result, and zero |
| F-000000000007 | Intervening call before transfer 3 |
| F-000000000008 | Last call before transfer 3 |
| F-000000000009 | Receives zero; supplies the secondary result |
| F-000000000010 | Receives three shifted values after transfer 5 |
| F-000000000011 | Supplies the first 64-bit result |
| F-000000000012 | Receives its low word, high word, and shifted low word |
| F-000000000013 | Receives zero; starts the narrow-result chain |
| F-000000000014 | Receives and transforms the preceding narrow result |
| F-000000000015 | Consumes the transformed narrow result |
| F-000000000016 | Call before production of the second wide result |
| F-000000000017 | Supplies the second 64-bit result |
| F-000000000018 | Receives split and shifted components; supplies the final wide result |
| F-000000000019 | Receives the final wide result’s low word |
| F-000000000020 | Supplies the last consumed narrow result |
| F-000000000021 | Receives that narrow result and zero |
| F-000000000022 | Final call before the store |

“No displayed arguments” and “no consumed result” describe only visible dataflow. Neither establishes a complete dependency signature or an absence of side effects.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. They assume accurate representation, the stated widths, returning dependencies, and successful preceding operations.

### Incoming operand routing

If P, Q, and R have distinguishable values, the five transfers must use these ordered operand pairs:

1. (P, R);
2. (Q, R);
3. (P, R);
4. (R, R);
5. (P, R).

Using P in transfer 2 or using P as the first operand in transfer 4 would violate the represented relationships, even if the hardware meanings remain unknown.

### Input truncation

If A has only bit 10 set, its 22-bit left shift produces zero at 32-bit width, while its 11-bit left shift produces bit 21.

The first two arguments to F-000000000005 therefore differ. Retaining an extra bit above bit 31 would not match the contract.

### Unsigned input shift

If C is 0xFFFFFFFF, the second argument to F-000000000006 is 0x07FFFFFF, not a sign-extended value.

### Primary-result scaling

If the primary result is 0x10000000, its one-bit left shift is 0x20000000, while its four-bit left shift is zero.

F-000000000006 therefore receives 0x20000000 as its first argument and zero as its third. The derived zero intermediate remains zero.

### Wide-result splitting

If F-000000000011 supplies 0x1122334480000001, F-000000000012 receives:

1. 0x80000001;
2. 0x11223344;
3. 0x00000002.

This example concerns bit extraction and truncation, not memory endianness.

### Calls before the store

If F-000000000018 supplies the final wide result, the next four call occurrences are F-000000000019 through F-000000000022, in that order. The store follows only after F-000000000022 returns.

A trace that stores after F-000000000021 but before F-000000000022 would not satisfy the contract.

### Destination boundaries and alignment

Address-index values 0, 1, 2, 3, and 4 produce destination addresses 0, 1, 2, 3, and 5 respectively. Address 4 is skipped at this boundary.

An address index of 4,095 produces destination 5,118.

If the destination is inaccessible or violates target alignment rules, successful completion of the store is not established. No explicit null check or alignment correction precedes it.

### Store width and byte order

Where the store succeeds, the written value is the halfword 0x00CD. On a little-endian target, its two bytes are CD followed by 00; on a big-endian target, they are 00 followed by CD.

The evidence does not choose between those byte orders or establish the physical transaction structure of the write.

## Uncertainties

The following remain **documented_only**:

- Whether the represented instruction stream is valid throughout.
- The actual executable boundary and entry convention.
- The intended application-level purpose.
- Dependency identities, complete signatures, preservation rules, and internal effects.
- Additional machine-state inputs consumed by calls.
- Whether the wide result containers correspond to scalar returns, register pairs, or another recovery artifact.
- Hardware interpretation, legality, and privilege requirements of the coprocessor operations.
- Actual memory mapping, alignment behavior, and endianness.
- Fault behavior for invalid targets, nonreturning dependencies, unsupported transfers, or failed memory access.
- The continuation beyond the invalid-instruction boundary.
- Whether the low destination range is intentional or reflects inaccurate recovery.

These uncertainties do not erase established operand routing, call order, store width, or arithmetic consequences.

## Evidence

**Established static evidence:** The supplied representation shows 22 ordered indirect calls, five coprocessor transfers, incoming operand routing, narrow and wide result propagation, shift and mask relationships, a final halfword store, and an invalid-instruction warning.

**Derived static consequences:** Fixed-width arithmetic establishes the zero intermediate, the unsigned input-shift range, the 12-bit address index, the destination range of zero through 5,118, and possible odd destinations. Counting the displayed calls establishes four call occurrences between production of the final wide result and the store.

**Inferred intent:** None asserted. In particular, neither the low-address write nor the invalid-instruction boundary establishes deliberate crash behavior.

**Runtime evidence:** None. No execution, runtime tests, target inspection, or dependency verification has been performed.
