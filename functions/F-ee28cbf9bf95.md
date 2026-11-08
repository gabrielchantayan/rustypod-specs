<!-- Trusted provenance: F-ee28cbf9bf95; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ee28cbf9bf95

## Purpose

F-ee28cbf9bf95 continuously processes a byte-sized selector obtained from a shared descriptor. Each iteration performs two opening calls, reads the selector, optionally constructs and submits a six-word request, and performs a closing call. A nonzero selector enables request processing; a zero selector bypasses it.

Request processing selects a token using byte-addressed state tables, optionally invokes a descriptor-selected callable, packs a 32-bit value, submits the request by address, and conditionally performs further state changes and calls.

The function has no normal return path. These are static control-flow properties. Interpreting the request as an event, the tables as device state, or the interval-related calls as delays would be inferred intent, not established behavior.

## Interface

### Entry and effective inputs

The function has no explicit arguments and supplies no explicit return value. Its effective inputs are:

- Shared configuration values.
- Contents of a descriptor reached through a captured address.
- Contents of byte-addressed state tables.
- Return values and side effects of dependencies.
- Any externally induced changes to storage it subsequently reads.

The represented widths are:

| Item | Width |
|---|---:|
| Selector, descriptor packing byte, table entries | 8 bits |
| Tokens, request words, configuration values, represented addresses | 32 bits |
| Descriptor-selected callable result | 32 bits |

The selector is zero-extended to the range 0–255 before use.

### Entry-time captures

Before starting the loop, the function performs five configuration reads in this order:

1. Baseline token.
2. Descriptor address.
3. Active token.
4. Primary-table address.
5. A second read of the same primary-table configuration location, followed by addition of 254 bytes to obtain the secondary-table address.

The two table-configuration reads are distinct. Their results need not be equal if the configuration changes between reads. Consequently, the captured secondary-table address is not unconditionally the captured primary-table address plus 254.

These five captured values remain in use across iterations. Subsequent changes to their originating configuration locations do not refresh the captures. Contents of storage reached through the captured addresses remain subject to later reads and writes.

### Descriptor layout

Offsets are byte offsets from the captured descriptor address.

| Offset | Width | Established use |
|---|---:|---|
| 1 | 8 bits | Contribution to bits 24–31 of request word 5 |
| 2 | 8 bits | Selector |
| 8 | 32 bits | Optional callable address |
| 16 | 32 bits | Second argument to the first interval-related call |
| 20 | 32 bits | Argument to the first opening call |

The body establishes no use for other descriptor bytes.

### State-table addressing

For a captured nonzero selector:

- Primary state is the byte at the captured primary-table address plus the selector.
- Secondary state is the byte at the captured secondary-table address plus the selector.
- A primary-state value of exactly 2 additionally selects a marking-table write, using a marking-table address read during that iteration.

Primary offsets range from 1 through 255. Secondary offsets, measured from the value obtained by the second entry-time table-configuration read, range from 255 through 509.

If the two table-configuration reads agree, primary state for selector 255 and secondary state for selector 1 occupy the same byte. Otherwise, that particular overlap is not established. Other overlaps depend on actual addresses.

### Request layout and lifetime

The represented request consists of six consecutive 32-bit words, totaling 24 bytes.

| Word | Assignment made by this function before submission |
|---|---|
| 0 | Baseline, active, or override token |
| 1 | None |
| 2 | None |
| 3 | Zero |
| 4 | Zero |
| 5 | Packed descriptor, selector, and callable-result contributions |

Words 1 and 2 are not initialized by this body. They must not be interpreted as zero, valid fields, or stable values. Prior contents or dependency writes are not converted into a defined initialization contract.

The request occupies function-local storage reused across iterations. F-1949aa589eec receives its address. The body does not establish copying, ownership transfer, permitted retention, asynchronous use, or a release obligation.

## Behavior

### Opening calls and selector capture

Each iteration begins with these operations:

1. Read the descriptor’s 32-bit field at offset 20 and pass it to F-d38ae664ee46.
2. After that call returns, read the current opening/closing configuration and pass it to F-338d16f9ce72.
3. After the second call returns, read the selector byte at descriptor offset 2.

Both opening dependencies can therefore affect storage read later in the current iteration.

If the selector is zero, the function skips request construction, main-branch state-table processing, callable processing, submission, and interval-related operations. It proceeds directly to the closing call.

Otherwise, the zero-extended selector is retained for the rest of that iteration. Later changes to descriptor offset 2 do not replace it.

### Initial token selection and state writes

The function first assigns the captured baseline token to request word 0, then reads primary state.

If primary state is zero:

- Selection leaves word 0 at the baseline token.
- Selection does not read secondary state.
- Selection does not read the override-token configuration.
- No marking-table write occurs.

This describes the selection phase only. A later active-token comparison can still cause a secondary-state write, including when the baseline and active tokens are equal.

If primary state is nonzero, the function assigns the captured active token to word 0. Primary state 2 introduces the following ordering:

1. Read the marking-table address configuration.
2. Assign the active token to request word 0.
3. Write byte value 1 at the marking-table address plus the captured selector.

For other nonzero primary-state values, it assigns the active token without reading the marking-table configuration or performing that write.

After the active-token assignment and any marking write, the function reads the current override-token configuration. This first override read occurs on every nonzero-primary path, before secondary state is tested.

It then reads secondary state:

- If secondary state is zero, selection leaves word 0 at the active token. There is no second override read and no secondary-state clear.
- If secondary state is nonzero, the function reads the override-token configuration again, assigns this second value to word 0, and then writes zero to secondary state.

The second override value, not the first, selects the token on the override path. The two reads need not agree. The first value does not contribute to request word 5.

A nonzero secondary byte alone does not activate override selection: primary state must first be nonzero.

### Descriptor-selected callable and request packing

The function next reads the callable address at descriptor offset 8 and tests it for null.

A null value suppresses invocation and supplies a zero callable contribution.

A non-null value causes another read of the callable address for invocation. The invoked target receives:

- Integer value 1.
- The zero-extended selector captured for this iteration.

Its complete 32-bit result becomes the callable contribution. The null-check read and invocation read are separate; equal target addresses are not guaranteed under intervening mutation. In particular, the initial non-null check does not establish that a subsequently observed address is valid.

After callable processing returns, the function writes zero to request words 3 and 4. It then reads the byte at descriptor offset 1 and assigns word 5 the bitwise OR of:

- That byte, zero-extended and shifted left 24 bits.
- The captured selector shifted left 16 bits.
- The entire 32-bit callable contribution.

There is no low-16-bit mask on the callable result. Its upper bits can set bits in both apparent descriptor and selector regions. These contributions are ORed, not added or independently protected.

A callable change to descriptor offset 1 can affect this iteration’s packing because that byte is read afterward. A change to descriptor offset 2 does not change the captured selector.

The callable has no established preservation contract for request word 0 or other accessible storage. Token selection therefore does not, by itself, guarantee the value present when submission begins.

### Submission and first interval-related call

The function passes the address of request word 0 to F-1949aa589eec.

After submission returns, it calls F-689e9e98c8f7 with:

- The current interval-target configuration.
- The current 32-bit descriptor field at offset 16.

Both reads occur after submission. Their relative evaluation order is not established by the supplied representation.

The request is passed by address, not as an immutable value. Neither submission nor the subsequent dependency has an established guarantee to preserve request word 0. Later decisions depend on the request storage actually observed after both calls, not necessarily on the token selected earlier.

### Post-submission comparisons

After the first F-689e9e98c8f7 call returns, the function reads and captures the current interval-target configuration. This read occurs even when the captured value will not be used.

It then compares request word 0 against the entry-captured active token.

If they match, the function:

1. Writes byte value 1 to secondary state for the captured selector.
2. Calls F-689e9e98c8f7 with the just-captured interval target and integer value 1000.
3. Proceeds to the conditional additional operation.

If the active comparison fails, it separately examines request word 0 against the entry-captured baseline token:

- A baseline match proceeds to the additional operation.
- A baseline mismatch skips it.

The active comparison takes precedence. If baseline and active tokens are equal, a matching request takes the active path, including the secondary-state write and extra call with 1000.

The two comparisons are separate examinations. A single stable word value across both is not guaranteed against external mutation.

The additional operation receives a fresh read of the interval-target configuration. It does not necessarily receive the earlier captured target. The extra call with 1000 can change that configuration before the additional operation obtains its argument.

There is no separate post-submission branch for an override token. An override-selected value follows whichever comparisons it satisfies when examined.

### Closing and repetition

After the selected path completes, the function reads the current opening/closing configuration and passes it to F-1333331b2da1. It then begins another iteration.

The opening and closing configuration reads are independent and need not agree.

The closing call is reached only if preceding operations on that path return normally. A dependency that does not return, a fault, or a nonlocal transfer can prevent it. No normal loop termination is present.

## Side effects

### Direct writes and their ordering

The body directly writes:

- Request word 0 during token selection.
- Request words 3 and 4 as zero.
- Request word 5 as the packed value.
- A marking-table byte as 1 when primary state equals 2.
- A secondary-state byte as 0 during nonzero-primary, nonzero-secondary override selection.
- A secondary-state byte as 1 when the later active-token comparison succeeds.

A secondary clear precedes callable execution and submission. A later secondary set follows submission and the first interval-related call. Both writes can occur in one iteration.

No write is directly selected through the primary-state role. Nevertheless, overlap can make a marking or secondary write alter storage also used as primary state.

Ordering matters under aliasing. In particular, the marking write precedes the secondary read and can change its result when the addresses coincide.

### Dependency effects and ownership

No invoked dependency has an established general guarantee to preserve globals, descriptor contents, table storage, or accessible request storage.

The body performs no explicit allocation or deallocation. This does not establish that dependencies allocate nothing, release nothing, or accept no ownership. No request retention or release rule is supplied.

### Preconditions and errors

There is no explicit validation, error result, retry policy, or recovery path.

Execution requires suitable storage for every reached access and a valid target for every reached invocation. Marking-table storage is required only on the primary-state-2 path. Other storage requirements likewise depend on which accesses are reached.

Address values and calculations are represented at 32-bit width. The body contains no bounds or overflow checks. A portable wrapping rule, valid address range, alignment requirement, or defined behavior after invalid address arithmetic is not established.

No synchronization, atomicity, or memory-ordering guarantees are supplied. Distinct reads must not be collapsed into a promise that their values agree.

## Dependencies

| Opaque ID or unresolved role | Established interaction | Unresolved behavior |
|---|---|---|
| F-d38ae664ee46 | First call each iteration; receives descriptor field at offset 20 | Meaning, side effects, return behavior |
| F-338d16f9ce72 | Second opening call; receives current opening/closing configuration | No established acquisition or locking semantics |
| F-1949aa589eec | Receives the local request address | Read/write extent, copying, retention, ownership |
| F-689e9e98c8f7 | Called after submission; conditionally called again with second argument 1000 | Units, timing, effects, return meaning |
| F-1333331b2da1 | Closing call; receives freshly read opening/closing configuration | No established release or synchronization semantics |
| Descriptor-selected callable; opaque ID unavailable | Conditionally receives 1 and the captured selector; supplies a 32-bit contribution | Identity, detailed calling convention, storage effects |
| Conditional additional callable; opaque ID unavailable | Receives a fresh interval-target configuration value on active or baseline match | Identity, meaning, storage effects |

The supplied mapping does not identify the two unresolved callable roles. No opaque identity is invented for them.

## Acceptance examples

These are conditional static deductions, not executed tests. Unless stated otherwise, dependencies return normally, accesses are valid, relevant storage does not alias unexpectedly, and external agents do not mutate relevant values. These shared conditions do **not** imply dependency preservation; preservation needed by an example is stated explicitly.

1. **Zero selector**

   If the selector read yields zero, the iteration calls F-d38ae664ee46, F-338d16f9ce72, and F-1333331b2da1 in that order. The function performs no main-branch table access, request submission, or interval-related call.

2. **Independent table captures**

   If the first table-configuration read yields hexadecimal 1000 and the second yields hexadecimal 2000, the secondary base is hexadecimal 20FE. Primary state for selector 255 is at hexadecimal 10FF; secondary state for selector 1 is at hexadecimal 20FF.

   If both reads yield hexadecimal 1000, those two selector-specific addresses instead coincide at hexadecimal 10FF.

3. **Baseline selection and baseline continuation**

   Suppose selector 7 is captured, primary state reads as zero, the callable check reads null, the baseline token is 16, the active token is 32, and descriptor offset 1 reads as 3.

   The function constructs word 0 as 16, words 3 and 4 as zero, and word 5 as hexadecimal 03070000. Selection reads neither secondary state nor override configuration.

   If submission and the first interval-related call preserve word 0, and it remains 16 through the comparisons, the baseline comparison succeeds. The additional operation follows, without an extra call with 1000. Without that preservation condition, the constructed baseline token does not determine the later branch.

4. **Active selection, submission, and active continuation**

   Suppose primary state is nonzero but not 2, secondary state reads as zero, and active and baseline tokens differ. Selection reads the override configuration once and assigns the active token.

   If word 0 remains unchanged through callable processing and submission entry, the submitted token is the active token. If submission and the first interval-related call also preserve it through the active comparison, secondary state is set to 1, the extra call receives 1000, and the additional operation follows.

   Callable mutation can invalidate the submitted-token conclusion. Mutation during submission or the first interval-related call can invalidate the post-submission branch conclusion.

5. **Primary-state-2 ordering**

   If primary state reads as 2, the function reads the marking-table address, assigns the active token, and writes the marking byte as 1, in that order. The first override read and secondary-state test follow the marking write.

6. **Different override reads**

   With nonzero primary and secondary states, suppose the first override read yields hexadecimal 40 and the second yields hexadecimal 50. Selection assigns hexadecimal 50 and then clears secondary state.

   If word 0 remains hexadecimal 50 through subsequent callable and dependency effects, and that value differs from both comparison tokens, neither conditional operation occurs.

7. **Unmasked callable contribution**

   If the captured selector is 1, descriptor offset 1 reads as zero, and the callable returns hexadecimal 80020004, the function assigns word 5 hexadecimal 80030004.

8. **Submission changes the decision**

   If selection chooses the active token but submission replaces word 0 with a value distinct from both comparison tokens, and that replacement remains unchanged through both comparisons, the function performs no later secondary-state set, extra call with 1000, or additional operation.

9. **Equal comparison tokens**

   If baseline and active tokens are equal and the active comparison observes that value in word 0, the active branch wins, even if primary state was zero. Secondary state is set to 1 and the extra call with 1000 occurs.

10. **Fresh additional-operation argument**

    If the post-interval target capture yields one value, but the extra call with 1000 changes the target configuration to another value that remains until the fresh read, the additional operation receives the latter value.

## Uncertainties

The following remain **documented_only**:

- Concrete configuration values, storage sizes, and address relationships.
- Equality of repeated configuration or descriptor reads.
- Dependency preservation, resource effects, and return guarantees.
- Whether interval-related arguments represent time, and any units.
- Descriptor-field meanings beyond their observed uses.
- Detailed callable conventions beyond the represented arguments and result.
- Whether submission reads request words 1 and 2.
- Request mutation, copying, retention, ownership, and release behavior.
- Identities of the two unmapped callable roles.
- Synchronization, atomicity, and concurrent-access rules.
- Exact overflow, invalid-address, alignment, and fault behavior.

Stateful event handling is a possible interpretation, not an acceptance requirement.

## Evidence

The supplied static body establishes the entry-read order, separate table captures, descriptor offsets, represented widths, token-selection paths, repeated override reads, marking-write ordering, callable processing, request packing, submission by address, post-submission comparison precedence, conditional state changes, fresh target reads, and indefinite repetition.

The supplied dependency mapping establishes only the five mapped dependency IDs listed above. It supplies no dependency preservation or ownership contracts and no opaque identity for the two unresolved callable roles.

The acceptance examples explicitly distinguish selected tokens, tokens present at submission, and values observed by later comparisons. Their conditional preservation assumptions are hypothetical premises, not inferred dependency guarantees.

No runtime tests have been performed. All examples are deductions from static evidence under their stated conditions.
