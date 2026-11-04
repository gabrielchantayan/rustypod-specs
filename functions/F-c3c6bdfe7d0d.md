<!-- Trusted provenance: F-c3c6bdfe7d0d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c3c6bdfe7d0d

## Purpose

F-c3c6bdfe7d0d dispatches an incoming record according to a 32-bit discriminator and, on two special paths, additional data obtained through F-682adfd5a561. Depending on the selected path, it invokes a receiver-associated indirect operation, forwards the receiver and record to one or both downstream functions, or returns without further dispatch.

Its normal return value is always the 32-bit value zero. That value does not distinguish an ignored record from one that caused an indirect operation or downstream processing.

The established behavior is a dispatch mechanism. Interpreting the record as an event, the receiver as an object, or particular numeric values as named protocol messages is inferred intent, not an established semantic contract.

## Interface

### Inputs

The function accepts two address-valued inputs:

- **Receiver:** An address whose first 32-bit word is used as an operation-table address on paths that invoke an indirect operation. The receiver is also passed unchanged to downstream dependencies.
- **Record:** An address from which a 32-bit discriminator is read at byte offset 4. This address is passed unchanged to F-682adfd5a561 and, where selected, to downstream dependencies.

The supplied representation uses 32-bit integers for addresses, discriminator values, and the return value. The layouts below therefore describe a 32-bit address model. The precise calling convention and native source-language types are not established.

### Relevant layouts

Only fields reached by a selected path need to be readable for this function’s own accesses.

| Referenced storage | Byte offset | Established use |
|---|---:|---|
| Record | 4 | Read a 32-bit discriminator |
| Receiver | 0 | Read an operation-table address |
| Receiver operation table | 0xE8 | Read an indirect operation address for the discriminator-0x13 matching path |
| Receiver operation table | 0x58 | Read an indirect operation address for the discriminator-0x16 matching path |
| Result of F-682adfd5a561 on the 0x13 path | 0x0C | Read a 32-bit comparison value |
| Result of F-682adfd5a561 on the 0x16 path | 8 | Read a nested address |
| Nested storage reached on the 0x16 path | 4 | Read a 32-bit comparison value |

The two operation-table offsets are byte offsets, not operation indices.

Two additional global data items participate in dispatch:

- A **dispatch adjustment**, used in arithmetic comparisons for discriminators other than 0x13 and 0x16.
- An **indirect-operation argument**, read and supplied to the operation at table offset 0x58 when its comparison succeeds.

Their values, mutability, initialization, and semantic meanings are unknown.

### Output

Every explicitly represented normal return produces zero in a four-byte return value.

There is no explicit output buffer. Any externally visible changes occur through dependencies or indirect operations, or through effects of the memory accesses themselves.

### Preconditions

The caller must supply storage and callable targets sufficient for the path actually taken. There are no explicit checks for null addresses, bounds, object identity, alignment, or the validity of an indirect target.

A successful lookup result is not checked for null before its fields are accessed. A nested address on the 0x16 path is likewise dereferenced without validation.

These are conditional memory-access requirements, not evidence that every malformed input is detected or converted into a defined failure result.

## Behavior

### Initial selection

The first explicit data access is a 32-bit read of the record discriminator at byte offset 4.

The discriminator is selected once for this invocation’s visible dispatch logic. The function does not explicitly reload it after a dependency or indirect operation runs.

The two literal discriminator values have priority over adjustment-based dispatch:

1. A discriminator of 0x13 selects the first special path.
2. Otherwise, a discriminator of 0x16 selects the second special path.
3. All other discriminator values use the dispatch adjustment.

Neither special path consults the dispatch adjustment.

### Discriminator 0x13

The function invokes F-682adfd5a561 with:

- The original record address.
- The numeric selector 0xC00.

After that dependency returns, the function reads the 32-bit field at byte offset 0x0C from the returned address.

If that field equals 0x80A9, the function:

1. Reads the operation-table address from the receiver’s first word.
2. Reads the operation address at table byte offset 0xE8.
3. Invokes that operation with the original receiver address and a zero-valued second argument.

If the comparison does not match, the indirect operation is skipped.

In either case, provided execution returns normally from all invoked operations, the function then invokes:

1. F-389be44b388f with the original receiver and record addresses.
2. F-81fc1937727a with those same addresses.

The second dependency is invoked after the first returns. Their results are not used in the visible dispatch logic.

This ordering matters: the receiver operation, when selected, runs before either downstream dependency. F-389be44b388f may change state subsequently observed by F-81fc1937727a. No preservation of receiver storage, record storage, or global state is established.

The function then returns zero.

### Discriminator 0x16

The function invokes F-682adfd5a561 with:

- The original record address.
- The numeric selector 0xF00.

After that dependency returns, the function:

1. Reads a nested address at byte offset 8 from the returned address.
2. Reads a 32-bit value at byte offset 4 from that nested address.
3. Compares that value with 0x6380002E.

If the comparison fails, the function returns zero immediately.

If the comparison succeeds, the function obtains the receiver-associated indirect operation at operation-table byte offset 0x58 and invokes it with:

- The original receiver address.
- The value supplied by the indirect-operation argument global.

The global value is an argument, not an address explicitly dereferenced by this function. Its interpretation by the indirect operation is unknown.

The function then returns zero. Neither F-389be44b388f nor F-81fc1937727a is invoked on this path, regardless of whether the nested comparison matches.

### Other discriminators

For any discriminator other than 0x13 or 0x16, the function adds the dispatch adjustment to the selected discriminator and compares the result with zero.

If that first comparison succeeds, it invokes:

1. F-389be44b388f with the original receiver and record addresses.
2. F-81fc1937727a with the same addresses.

It then returns zero.

If the first comparison fails, the represented expression reads the dispatch adjustment again and performs another addition, this time comparing the result with two.

- If the second result equals two, only F-81fc1937727a is invoked, with the original receiver and record addresses.
- Otherwise, no dependency or indirect operation is invoked after the discriminator and adjustment reads.

Both outcomes return zero.

There is no call between the two adjustment-based comparisons. Nevertheless, the supplied representation contains two adjustment references rather than an explicit saved adjustment value. A single immutable global snapshot is therefore not part of the established contract. Whether concurrent or asynchronous modification could be observed requires additional platform and memory-model evidence.

### Arithmetic boundaries

The discriminator and adjustment participate in a 32-bit integer expression. The comparisons distinguish a result equal to zero from one equal to two; they are not range tests.

For values whose sum is representable as a signed 32-bit integer, the comparison outcomes are unambiguous. For example, a discriminator of zero with an adjustment of two reaches the second comparison and selects F-81fc1937727a alone.

The supplied representation does not establish overflow semantics independently of the original machine instructions. Treating addition as modulo 2³² is plausible for a 32-bit machine operation, but overflow-dependent dispatch examples remain documented_only rather than unconditional acceptance requirements.

## Side effects

There are no explicit stores to receiver memory, record memory, returned storage, or globals in the visible function body.

The explicit effects are reads and calls. Called dependencies and indirect operations may modify memory, change globals, retain addresses, release resources, or fail to return. None of those possibilities is ruled out by the supplied material.

In particular:

- F-682adfd5a561 runs before its result fields are read.
- On the matching 0x13 path, receiver operation-table reads occur after F-682adfd5a561 returns.
- The matching 0x16 indirect operation receives the global argument after the lookup and nested comparison.
- On paths invoking both downstream dependencies, F-81fc1937727a begins only after F-389be44b388f returns.
- Calls receive the original address values, but this does not imply that the addressed contents remain unchanged.

No allocation, deallocation, ownership transfer, reference-count adjustment, or cleanup is explicitly performed by F-c3c6bdfe7d0d itself. The ownership policies of callees are unknown. Passing an address must not be interpreted as proof of either borrowing or transfer.

No explicit error reporting is present. Invalid storage, invalid indirect targets, or nonreturning dependencies are not converted into the normal zero result by any visible recovery mechanism.

## Dependencies

| Opaque ID or operation | Invocation condition | Supplied arguments | Result use |
|---|---|---|---|
| F-682adfd5a561 | Discriminator 0x13 | Record, 0xC00 | Returned address supplies a field at offset 0x0C |
| F-682adfd5a561 | Discriminator 0x16 | Record, 0xF00 | Returned address supplies a nested address at offset 8 |
| F-389be44b388f | Discriminator 0x13, or first adjustment comparison equals zero | Receiver, record | Result ignored |
| F-81fc1937727a | Discriminator 0x13, first adjustment comparison equals zero, or second adjustment comparison equals two | Receiver, record | Result ignored |
| Receiver operation at table offset 0xE8 | Discriminator 0x13 and returned field equals 0x80A9 | Receiver, zero | Result ignored |
| Receiver operation at table offset 0x58 | Discriminator 0x16 and nested field equals 0x6380002E | Receiver, global argument value | Result ignored |

The indirect operations cannot be assigned opaque function IDs from the available dependency map. Their identity is determined by receiver-associated table contents at invocation time.

No dependency contract establishes whether results are newly allocated, shared, borrowed, stable across calls, or subject to release. This function performs no explicit release of the lookup result.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Each assumes valid required storage and normal return from every selected call.

1. **0x13 with a matching field.**  
   F-682adfd5a561 receives the record and 0xC00. Its returned field at offset 0x0C is 0x80A9. The operation at receiver-table offset 0xE8 runs with receiver and zero, followed by F-389be44b388f, followed by F-81fc1937727a. The final return is zero.

2. **0x13 with a nonmatching field.**  
   The same lookup occurs, but the returned field differs from 0x80A9. No receiver-table operation is selected by this function. Both downstream dependencies still run, in their established order. The final return is zero.

3. **0x16 with a matching nested field.**  
   F-682adfd5a561 receives the record and 0xF00. The address read at result offset 8 leads to a field at offset 4 equal to 0x6380002E. The operation at receiver-table offset 0x58 runs with receiver and the global argument value. Neither downstream dependency runs. The final return is zero.

4. **0x16 with a nonmatching nested field.**  
   The lookup and nested reads occur. No receiver-table operation or downstream dependency follows. The final return is zero.

5. **Adjustment selects both downstream dependencies.**  
   The discriminator is zero and the dispatch adjustment is zero when first read. F-389be44b388f runs before F-81fc1937727a. No lookup or receiver-table operation is selected. The final return is zero.

6. **Adjustment selects only the final dependency.**  
   The discriminator is zero and the adjustment is two for both comparisons. The first comparison fails and the second succeeds. Only F-81fc1937727a runs. The final return is zero.

7. **Adjustment selects neither dependency.**  
   The discriminator is zero and the adjustment is one for both comparisons. No call follows the adjustment comparisons. The final return is zero.

8. **Special-path precedence.**  
   The discriminator is 0x16 and the adjustment would otherwise produce zero. The adjustment is not consulted: only the 0x16 lookup-and-compare path applies.

9. **Invalid lookup result.**  
   F-682adfd5a561 returns an address unsuitable for the required read. There is no established normal-return acceptance outcome. A zero result must not be promised.

## Uncertainties

The following remain documented_only:

- The meanings of the discriminator values, lookup selectors, comparison constants, and global data.
- The dispatch adjustment’s actual value and whether it can change.
- The exact width, representation, and ownership implications of the global indirect-operation argument.
- Overflow behavior for the adjustment addition.
- Byte order, alignment requirements, and the precise calling convention.
- The identities and contracts of the receiver-associated indirect operations.
- Dependency side effects, allocation policies, address retention, resource release, and failure behavior.
- Whether memory reads can interact with volatile storage, memory-mapped devices, concurrent mutation, or asynchronous handlers.
- Any intended higher-level meaning of the unconditional zero return.

No assumption that a dependency preserves globals or keeps passed storage alive is justified.

## Evidence

The supplied static function representation establishes the branch priorities, numeric comparisons, byte offsets, call arguments, downstream call order, lack of explicit stores, and zero-valued normal returns described above. The supplied dependency map establishes the opaque IDs used for direct dependencies.

The 32-bit address-layout interpretation follows the represented integer widths and address-valued field accesses. Higher-level object and event terminology is descriptive, not proof of an original type system or protocol.

No dependency bodies, global definitions, machine-instruction listing, or runtime observations were supplied. No runtime tests have been performed.
