<!-- Trusted provenance: F-21712eedaa5c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-21712eedaa5c

## Purpose

F-21712eedaa5c selects between two observable behaviors:

- For selection −1, it performs three direct writes to caller-supplied destination storage, with a dependency invocation between the second and third writes, and returns the destination address.
- For every other selection, it obtains an object, selects an indirect-call target, invokes that target with temporary-storage and selection arguments, and performs three further dependency calls. It returns the final dependency’s result.

On the indexed path, a state byte determines whether the original selection is passed unchanged or replaced by a 32-bit table entry. The translated path also compares two dependency results and conditionally invokes another dependency before reading the table.

The sequence is consistent with obtaining a selected value and constructing or assigning a destination value. That interpretation is **inferred intent**, not an established effect. The supplied body establishes which addresses and values are passed to dependencies; it does not establish that those dependencies write temporary storage, construct the destination, or release resources.

## Interface

### Inputs and result

The function receives three inputs:

| Input | Visible representation | Use |
|---|---|---|
| Destination | Address represented as a pointer to 32-bit words | Receives direct writes on the −1 path; is passed to a dependency on either path |
| State | 32-bit address-valued input | Base for indexed-path accesses and dependency arguments |
| Selection | Signed 32-bit integer | −1 selects the special path; all other values select the indexed path |

Offsets below are byte offsets.

The result is address-valued and represented as a pointer to 32-bit words.

- On the −1 path, the result is the supplied destination address, assuming the dependency returns normally.
- On either indexed path, the result is the address-valued result of F-c1466ea05103. The function does not replace that result with the destination address.

There is no established success flag, error-code convention, or guarantee that an indexed-path result equals the destination.

### Directly accessed storage

On the −1 path, the function directly accesses the following destination locations:

| Offset | Access width | Direct action |
|---|---:|---|
| +0 | 32 bits | Stores a value read from a global 32-bit datum |
| +4 | 32 bits | Stores zero |
| +16 | 8 bits | Stores zero after F-87faa0f246f3 returns |

These writes span 17 bytes, but do not establish the destination’s complete size or the dependency’s access footprint.

The indexed paths use these state locations:

| Offset | Access or use |
|---|---|
| +0x18 | Its address is passed to F-e38e6836b640 |
| +0x840 | Its address is passed to F-8cb33c040d4d on the translated path; its 32-bit contents are subsequently read as a table-base address |
| +0x84c | One byte is read to choose direct or translated indexing |

Each object used for indirect dispatch supplies a dispatch-table address through its first 32-bit word. The relevant target-address entries are at dispatch-table byte offsets +0x1bc and +0x1c0.

Two distinct local temporary-storage regions have visible extents of 16 and 20 bytes. Their addresses are passed to dependencies. The enclosing function does not directly initialize, read, or write their contents in the supplied body. Whether dependencies access those contents, and how, remains unestablished.

### Preconditions

The function contains no visible checks for:

- Destination or state validity.
- Destination capacity.
- Object or dispatch-target validity.
- Table bounds or selection range.
- Validity of dependency results.
- Arithmetic overflow.

For a defined, normally returning execution, every address actually accessed must support the required access width, and every invoked target must satisfy its call-site requirements. Dependencies may impose additional storage, alignment, lifetime, or argument requirements that are not supplied.

The −1 path makes no direct state access. That does not establish that F-87faa0f246f3 cannot access related state indirectly.

## Behavior

### Distinguished selection: −1

The selection comparison occurs before any direct state access.

For selection −1, the observable local sequence is:

1. Read the global 32-bit datum and store that value at destination +0.
2. Store a 32-bit zero at destination +4.
3. Invoke F-87faa0f246f3 with the destination address and zero.
4. After that call returns normally, store an 8-bit zero at destination +16.
5. Return the destination address.

The mode byte, object dispatch, comparison operations, and translation table are bypassed.

The final byte write is not conditional on the dependency’s ordinary return value. If the dependency changes destination +16, that byte is subsequently overwritten with zero. The earlier stores at +0 and +4 are not repeated after the dependency; changes it makes there may remain.

No direct writes to destination +8 through +15 are visible. Their final contents cannot be determined because the dependency’s access behavior is unknown. The same limitation applies to destination storage beyond the directly written locations.

If the dependency does not return normally, the supplied body establishes no completion of the final byte write or ordinary return.

### Indexed selection: mode choice

For every selection other than −1, the function reads the byte at state +0x84c.

- Zero selects direct indexing.
- Any nonzero value selects translated indexing.

The mode byte is read once for this decision. There is no later reread, even if a dependency changes it. Nonzero values are not restricted to one.

### Direct indexing

With a zero mode byte:

1. F-e38e6836b640 receives the address state +0x18.
2. The returned object’s first 32-bit word supplies its dispatch-table address.
3. The target at dispatch-table offset +0x1c0 is obtained.
4. That target is invoked with the 16-byte temporary-storage address, the returned object address, and the original selection.
5. Execution continues to the common dependency sequence.

The selection is passed unchanged. Apart from routing −1 to the special path, the function does not locally reject negative selections or check an upper bound.

This path invokes F-e38e6836b640 once. It does not invoke the +0x1bc target, F-8cb33c040d4d, or F-814d4ee17e01.

### Translated indexing

With a nonzero mode byte, the sequence is:

1. Invoke F-e38e6836b640 with state +0x18.
2. Read the returned object’s dispatch-table address and obtain the target at offset +0x1bc.
3. Invoke that target and retain its 32-bit result.
4. Invoke F-8cb33c040d4d with state +0x840.
5. Compare its 32-bit result with the retained result.
6. If the results differ, invoke F-814d4ee17e01 with the state address.
7. Invoke F-e38e6836b640 again with state +0x18.
8. Read the 32-bit table-base address at state +0x840.
9. Read a 32-bit table entry at the address formed from that base and four times the original selection.
10. Read the second returned object’s dispatch-table address and obtain the target at offset +0x1c0.
11. Invoke that target with the 16-byte temporary-storage address, the second object address, and the table-entry value.
12. Continue to the common dependency sequence.

The +0x1bc call shows no explicit arguments. The supplied representation does not establish a receiver or other implicit arguments.

The comparison establishes equality or inequality of two 32-bit results. It does not establish that either result is a count, size, version, or capacity.

F-814d4ee17e01 is invoked once on inequality and not invoked on equality. There is no repeated comparison, retry, or local check of its result. Once it returns, execution proceeds regardless of whether it changed anything or made the compared properties agree.

The second object lookup always occurs on this path, including when the comparison results match. Its object may differ from the first object.

The table-base read occurs after the optional F-814d4ee17e01 call and after the second F-e38e6836b640 call. Changes made by either call can therefore affect the table base subsequently read. The table entry is read before the +0x1c0 dispatch information is obtained from the second object.

### Index arithmetic

The translation table has a visible four-byte stride, and each selected entry is read as a full 32-bit value. That entry replaces the original selection for the +0x1c0 call; it is not locally range-checked or otherwise transformed.

Negative selections other than −1 participate in the same address calculation. The source contains no separate rejection, saturation, or bounds adjustment.

The visible address and integer operations use 32-bit representations. Multiplication by four and addition to the table base must not be treated as unbounded mathematical arithmetic. The supplied representation does not independently settle precise machine-level overflow behavior for every input.

Safe use therefore requires a selection whose resulting entry address is valid under the actual target arithmetic. A negative or very large selection is neither guaranteed safe nor converted into a documented error by this function.

### Common indexed-path sequence

After the +0x1c0 target returns normally, both indexed paths perform these calls in order:

1. F-f814eaa01b27 receives the 20-byte temporary-storage address, the 16-byte temporary-storage address, and zero.
2. F-2b86bf938ef5 receives the destination address and the 20-byte temporary-storage address.
3. F-c1466ea05103 receives the 20-byte temporary-storage address.
4. The function returns F-c1466ea05103’s address-valued result.

The enclosing function does not inspect the ordinary return values of the indirect call, F-f814eaa01b27, or F-2b86bf938ef5.

Passing these storage addresses does not itself prove that any call writes, constructs, copies, consumes, transfers, or destroys a value. In particular, the F-2b86bf938ef5 invocation does not establish destination construction or assignment without that dependency’s contract.

The temporary storage is local to the invocation. The visible body establishes no right for a dependency to retain and later dereference those addresses. No separate call using only the 16-byte region as a cleanup argument is visible. Resource handling for either region remains unknown.

## Side effects

Established direct effects and interactions are:

- The three destination writes on the −1 path.
- The −1 path’s global-datum read.
- The indexed paths’ mode-byte and dispatch-information reads.
- The translated path’s table-base and table-entry reads.
- The specified dependency invocations, argument values, and ordering.

Writes to temporary storage through dependencies are **possible but unestablished**. The supplied body proves that their addresses are passed, not that writes occur.

Similarly, the indexed path passes the destination to F-2b86bf938ef5, but establishes neither that this call modifies the destination nor that it constructs or assigns any particular value. Its write footprint, if any, and its handling of previous destination contents are unknown.

No dependency is established to preserve the state object, globals, destination, temporary contents, or other reachable storage. Mutation by F-814d4ee17e01 is not guaranteed merely because it is conditionally invoked; its “update” role is inferred.

No allocation, deallocation, reference-count change, ownership transfer, or resource release is directly established. F-c1466ea05103’s position is consistent with finalization, but does not prove cleanup or ownership.

There is no visible error handler, rollback sequence, or explicit failure result. Invalid accesses and dependency failures are not locally translated into a documented outcome. Nonreturning calls and nonlocal transfers have no supplied completion contract.

## Dependencies

| Dependency | Established interaction | Unestablished behavior |
|---|---|---|
| F-e38e6836b640 | Receives state +0x18; supplies an object address used for dispatch | Lifetime, nullability, ownership, and mutation |
| Dispatch target at +0x1bc | Supplies the first 32-bit comparison result | Meaning, implicit arguments, and side effects |
| F-8cb33c040d4d | Receives state +0x840; supplies the second comparison result | Meaning and preservation of storage |
| F-814d4ee17e01 | Receives state on comparison inequality | Whether it updates anything; exact effects and failure behavior |
| Dispatch target at +0x1c0 | Receives 16-byte temporary-storage address, object address, and effective selection | Whether it writes storage; output representation and ownership |
| F-f814eaa01b27 | Receives both temporary-storage addresses and zero | Reads, writes, conversion, copying, or transfer |
| F-2b86bf938ef5 | Receives destination and 20-byte temporary-storage address | Destination modification, construction, replacement, and ownership |
| F-c1466ea05103 | Receives 20-byte temporary-storage address; supplies indexed-path result | Cleanup, mutation, and result validity or lifetime |
| F-87faa0f246f3 | Receives destination and zero on the −1 path | Access footprint and initialization behavior |

## Acceptance examples

These are conditional expectations from static evidence, not runtime-test results. Each assumes valid accesses and normally returning dependencies unless stated otherwise.

1. **Sentinel bypasses direct state access.** With selection −1, the function does not directly dereference the state input. It performs the first two destination stores, invokes F-87faa0f246f3, performs the trailing byte store, and returns the destination. An inaccessible state address alone does not invalidate this direct sequence, but the dependency must independently meet its own access requirements.

2. **Dependency modifies sentinel-path fields.** If F-87faa0f246f3 changes destination +0, +4, and +16, the enclosing function does not restore the first two fields. It does overwrite the byte at +16 with zero.

3. **Direct selection remains unchanged.** With mode zero and selection 7, the +0x1c0 target receives 7. No translation-table entry is read, and the comparison and conditional-call sequence is absent.

4. **Matching comparison results still require another lookup.** With nonzero mode and equal comparison results, F-814d4ee17e01 is skipped. A second object lookup still occurs. If selection 3 addresses a valid table entry containing 11, the later +0x1c0 call receives 11.

5. **Mismatch precedes fresh table access.** With unequal results, F-814d4ee17e01 is invoked before the second object lookup and table-base read. If it replaces the table, that replacement is used unless intervening behavior changes the base again. No equality recheck follows.

6. **Nonzero mode is not limited to one.** A mode byte of 2 takes the same translated branch as a mode byte of 1. A dependency changing the byte afterward does not cause a new mode decision.

7. **Negative nonsentinel input is not rejected locally.** Selection −2 reaches the indexed path. Direct mode passes −2 unchanged. Translated mode uses it in address calculation. Successful execution remains conditional on valid addressing and downstream argument use.

8. **Temporary writes are not an acceptance requirement established here.** The indexed sequence passes the two temporary-storage addresses in the specified order. This body alone does not require that either region change or that the destination be constructed. Such requirements would need dependency evidence.

9. **Indexed return is not normalized.** If F-c1466ea05103 returns an address different from the destination, that different address is returned unchanged.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of the destination, state, selections, and table entries.
- Complete object layouts and alignment requirements.
- Global-datum identity, value, mutability, and ownership implications.
- Implicit arguments of the +0x1bc target.
- Valid index ranges, table bounds, and exact overflowing-address behavior.
- Whether dependencies read or write either temporary-storage region.
- Whether F-2b86bf938ef5 modifies, constructs, or assigns the destination.
- Whether the conditional dependency mutates state.
- Allocation, ownership, copying, transfer, cleanup, retained references, and failure contracts.
- Validity, lifetime, and intended use of the indexed return value.
- Concurrency, synchronization, and reentrancy requirements.

“Initialization,” “translation,” “update,” “conversion,” and “finalization” are interpretive labels where they describe dependency purposes, not additional guarantees.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping.

Established evidence comprises the sentinel comparison, represented widths, storage offsets, branch conditions, direct reads and writes, temporary-region extents, dependency arguments and ordering, dispatch offsets, and path-specific return expressions.

Dependency bodies and contracts, runtime traces, independent type definitions, and machine-level overflow analysis were not supplied. In particular, passing a storage address is evidence of an invocation argument, not evidence that the callee writes that storage.

No runtime tests have been performed. Inferred purposes and unresolved dependency behavior are not promoted into established effects.
