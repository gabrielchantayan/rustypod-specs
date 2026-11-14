<!-- Trusted provenance: F-6dcfaa589da3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6dcfaa589da3

## Purpose

F-6dcfaa589da3 repeatedly interleaves indirect dependency calls with byte and four-byte memory accesses. Dependency results supply addresses, publication values, selectors, and state for the next iteration.

The visible control flow has one unconditional repeated path and no normal return. Its application-level purpose is not established. Terms such as “publication,” “selector,” and “region” describe observable roles, not recovered object types, protocols, or hardware functions.

This contract describes the supplied static representation. It does not establish executable validity, the completeness of recovered control flow, or the actual calling conventions of indirect dependencies.

## Interface

### Explicit inputs

| Argument position | Presented width | Visible use |
|---|---:|---|
| First | 32 bits | Initial address base for a four-byte store at offset `0x14`; replaced at iteration end. |
| Second | 32 bits | No explicit use shown. |
| Third | 32 bits | No explicit use shown. |
| Fourth | Unsigned 32 bits | Supplies a field for the first selected byte store; replaced at iteration end. |

The second and third positions cannot be declared irrelevant to the actual machine interface: implicit dependency inputs and register conventions remain unresolved.

Three additional incoming machine-state values are consumed:

- A 32-bit **carried value**, initially written through the first argument.
- An **incoming selector base**, whose byte at offset `1` selects byte-store destinations.
- An **entry control value**, used to calculate the first indirect target.

Consequently, the four explicit arguments do not fully specify an invocation.

### Outputs and preconditions

There is no visible normal completion or returned value. Outputs are ordered direct memory accesses, dependency invocations, and any effects of those dependencies.

Progress requires each reached access and control transfer to be valid and each dependency to return. No checks establish these conditions. The representation supplies no buffer lengths, object extents, ownership requirements, or guarantees that numerically small addresses are accessible.

### Widths and layout

Direct accesses comprise unsigned byte reads, byte stores, and four-byte reads and stores. Byte stores retain only the low eight bits of the value being stored. Most values and address calculations are presented as 32-bit quantities. Selected dependency results are 64 bits and are explicitly separated into low and high 32-bit halves.

No object layout beyond the accessed offsets is established. Endianness, alignment requirements, atomicity, and memory mapping are unknown.

Arithmetic simplifications and numerical examples below are conditional on 32-bit wrapping arithmetic: high bits are discarded after shifts, multiplication, and addition. This is not a claim that overflowing signed expressions have defined source-language semantics. Unsigned masks and right shifts establish the stated bit extractions; signed left shifts and overflowing address calculations require machine-level confirmation.

## Behavior

### Ordering conventions

The phases below form one continuous sequence. Their boundaries do not imply synchronization or isolation.

An **acquisition group** means four consecutive indirect calls with no explicit arguments shown. The first three results are not explicitly consumed; the fourth supplies the named result. An **unused four-call group** has the same call count, with no explicitly consumed result.

These groups remain observable dependency interactions. “No explicit arguments” does not establish an argument-free machine interface, and unused results do not make calls removable.

Every selected store described as using a byte selector performs a byte read, treats its unsigned value directly as an address base, and stores at that value plus the stated offset. It does not read a pointer from a table.

### 1. Initial publication and selection

First, store the carried value as four bytes at the first argument plus `0x14`.

Next, invoke five dependencies in sequence. The first target depends on incoming control state and receives the first argument. The next four calls have no explicit arguments; the fifth call’s 32-bit result becomes the **initial index**.

Read the byte at incoming selector base plus `1`. Store bits 20–27 of the fourth argument at the selected byte value plus `0x11`. The expression masks to bits 0–29 before shifting right by 20; byte truncation discards the two remaining upper extracted bits.

The selected destination ranges from `0x11` through `0x110`.

### 2. Initial dependency exchange and selected clear

Invoke dependencies in this order:

1. One call receiving the carried value shifted left by 12.
2. One call with no explicit arguments, producing a 32-bit result.
3. One call receiving that result, then the initial index shifted left by 30.
4. One call producing a 64-bit result.
5. One call receiving that result’s low 32 bits, then its high 32 bits.
6. Two calls with no explicit arguments, the second producing a result retained for phase 3.

Then reread the byte at incoming selector base plus `1` and store zero at its value plus `0x11`.

This is a fresh selector read. The clear need not address the byte written in phase 1.

### 3. First paired four-byte publication

Pass the retained phase-2 result shifted left by two to a dependency. An acquisition group then supplies a result whose left shift by two becomes the **first payload**.

Store the first payload as four bytes at initial index multiplied by `0x4000`, plus `0x14`.

The next call receives four arguments, in order:

- Initial index multiplied by `0x1000`.
- The same value again.
- Initial index shifted left by 14.
- First payload.

An acquisition group supplies a destination index. Store the same first payload at four times that destination index plus `0x14`. Then call a dependency with four times the destination index.

### 4. Two index-publication exchanges

An acquisition group supplies a source index. Store four times that index as four bytes at sixteen times the index plus `0x14`. Call a dependency with three identical arguments, each four times the source index.

Another acquisition group supplies a destination index. Store the same scaled source value at four times the destination index plus `0x14`, then call a dependency with four times that destination index.

Repeat this entire exchange with newly acquired source and destination indices.

The second exchange also establishes:

- A **region base** equal to its source index multiplied by `0x40000000`.
- A **second destination index**, retained for a later publication.

Under wrapping arithmetic, the region base depends only on the source index’s low two bits.

### 5. Two ordered pairs of direct region clears

Following the final destination-consuming call from phase 4, an acquisition group supplies a 32-bit **region-read source**.

Perform these events in order:

1. Store a zero byte at region base plus `0x11`.
2. Store a zero byte at region base plus `1`.
3. Call a dependency with the region-read source shifted left by two.
4. Call a dependency with no explicit arguments, obtaining a result.
5. Forward that result as the sole explicit argument to another dependency.
6. Execute an acquisition group, obtaining a fresh result.
7. Store a zero byte at region base plus `0x11`.
8. Store a zero byte at region base plus `1`.
9. Call a dependency with the fresh result shifted left by two.

In particular, the second pair of clears occurs **after obtaining the fresh result and before invoking its shifted-result consumer**. That consumer may observe or modify the cleared locations. Moving either clear after the call changes the observable sequence.

### 6. Region-derived publication and small-address read

An acquisition group supplies a new index. Store the second destination index shifted left by 31 as four bytes at sixteen times the new index plus `0x14`.

Retain four times the new index as the **retained address argument**. Under wrapping arithmetic, the stored value is either zero or `0x80000000`.

Next, read four bytes from:

- `0x14` plus bits 20–29 of the region-read source.

The starting address ranges from `0x14` through `0x413`. There is no multiplication by four or alignment adjustment.

Call a dependency with the retained address argument. One subsequent call with no explicit arguments supplies the **retained result**. Call another dependency with the four-byte read value shifted right by 22 and then left by 28.

Under 32-bit truncation, this last argument retains original bits 22–25 in positions 28–31.

### 7. Four-byte zero publication

An acquisition group supplies another index. Store four zero bytes at four times that index plus `0x14`, then call a dependency with four times the index.

This is one four-byte store, not a selected byte clear.

### 8. Region-selected byte and four-byte publications

An acquisition group supplies an unsigned **region payload**.

Read the selector byte at region base plus `1`, and store the low byte of region payload shifted left by two at selector plus `0x11`. Call a dependency with the full 32-bit shifted value.

A call with no explicit arguments supplies a result. The following call receives that result, then the region payload shifted left by 30. An acquisition group supplies another result.

Now reread region base plus `1` and retain that byte as the **saved selector**. Store bits 20–27 of the region payload at saved selector plus `0x11`.

Call a dependency with the latest acquired result shifted left by two. Execute an unused four-call group. Then store the retained result from phase 6 shifted left by two as four bytes at saved selector plus `0x14`.

This last store uses the saved selector, not another selector read.

### 9. Global-sourced selected exchange

Read one four-byte global value. This read occurs after the saved-selector four-byte publication and before the following dependency calls.

Call a dependency with the retained address argument. An acquisition group then supplies a 64-bit result. Its low half shifted left by two becomes the **secondary scaled value**.

Reread region base plus `1`. At selector plus `0x11`, store the low byte of the previously read global value shifted right by 22: original bits 22–29.

Call a dependency with the 64-bit result’s high half. A call with no explicit arguments supplies a 32-bit value; the following call receives that value, then the secondary scaled value shifted left by 28.

An acquisition group supplies a result. Reread region base plus `1` and store zero at selector plus `0x11`.

The global is sampled before the intervening calls. No preservation of the underlying global storage is assumed.

### 10. Separate duplicate-argument exchange and secondary publications

Call a dependency with the result preceding the phase-9 selector clear shifted left by two.

An acquisition group obtains a distinct **intermediate result**. The next dependency receives **two identical arguments**, each the intermediate result shifted left by two.

Only after this two-argument interaction does another acquisition group obtain the result whose left shift by two forms the **secondary payload**. The duplicated arguments belong to their own call; they are neither an extra publication nor arguments derived from the eventual secondary payload.

Store the secondary payload as four bytes at secondary scaled value multiplied by `0x1000`, plus `0x14`.

Call a dependency with the secondary scaled value shifted left by 14. An acquisition group supplies a destination index. Store the same secondary payload at four times that index plus `0x14`, then call a dependency with four times the index.

### 11. Final selector exchanges

An acquisition group supplies a 64-bit result. Its low half shifted left by two becomes the **final scaled value**.

Read a selector byte at secondary scaled value multiplied by `0x4000`, plus `1`. Store bits 20–27 of the secondary payload at selector plus `0x11`.

Continue with these calls:

- Pass the 64-bit result’s high half to a dependency.
- Obtain a 32-bit result from one call with no explicit arguments.
- Forward it to a dependency.
- Obtain another 64-bit result from one call with no explicit arguments.
- Pass its low and high halves, in that order, to a dependency.
- Execute two calls with no explicit arguments, retaining the second result.

Read a selector at final scaled value multiplied by `0x10000000`, plus `1`, and store zero at selector plus `0x11`.

Call a dependency with the retained latest result shifted left by two. An acquisition group supplies the **final publication source**.

Set the **next selector base** to four times the final scaled value. Read its byte at offset `1` and clear selector plus `0x11`.

Call a dependency with the final scaled value shifted left by 10. One call with no explicit arguments supplies a result; the next call receives that result, then the retained phase-6 result shifted left by 28.

An acquisition group supplies another result. Reread next selector base plus `1` and again clear selector plus `0x11`.

### 12. Final publication and restart

Call a dependency with the latest phase-11 result shifted left by two, then execute an unused four-call group.

The final publication source shifted left by 14 becomes the **next carried value**. Store it as four bytes at:

- Final publication source shifted left by two, then multiplied by `0x1000`, plus `0x14`.

Replace the fourth argument with the next carried value.

Call a dependency with the final publication source shifted left by two and then left by 14. After one call with no explicit arguments, call another dependency with:

1. The replacement fourth argument shifted left by 28.
2. An additional machine-level output whose producer and meaning are unresolved.

Under wrapping arithmetic, the first of these arguments is zero because the replacement fourth argument already has its low 14 bits cleared.

An acquisition group supplies a result whose left shift by two becomes the next first argument. The entry control state is replaced by a fixed continuation-derived value immediately before that group’s final call.

Restart phase 1 with the new first argument, carried value, selector base, fourth argument, and control state. No normal exit is shown.

## Side effects

Direct writes are immediately part of the visible sequence; no transactional grouping, rollback, or compensating action is shown.

Calculated locations may overlap selector storage, previous publications, global storage, or dependency-accessed memory. No non-aliasing guarantee exists. Repeated reads cannot generally be replaced with cached values, especially across dependency calls.

No allocation, release, ownership transfer, or lifetime extension is established by the direct operations. Dependencies may allocate, release, retain, or mutate storage; their behavior is unknown.

No explicit error status, bounds validation, null rejection, alignment check, exception handler, timeout, or retry policy appears. An invalid access, invalid target, or nonreturning dependency may stop progress after earlier effects have occurred. Exact fault behavior is not established.

## Dependencies

Only F-6dcfaa589da3 has a supplied resolved identity.

| Opaque reference | Interaction | Unresolved properties |
|---|---|---|
| F-000000000001 | Entry-selected indirect dependency; receives the current first argument. | Target validity, actual signature, implicit inputs, effects, termination. |
| F-000000000002 | Documentation reference for the remaining unresolved indirect dependency set. Calls have zero through four explicit arguments; consumed outputs include 32-bit values, 64-bit values, and an unresolved additional machine-level output. | Individual identities, calling conventions, register and global preservation, ownership effects, errors, termination. |

F-000000000002 is a set reference, not a claim that those calls share a function. Targets are calculated from continuation-like values and fixed displacements. Their executable interpretation remains unresolved.

## Acceptance examples

These are conditional static criteria, not executed tests. They assume valid accesses, returning dependencies, and the stated wrapping-arithmetic interpretation.

1. **Initial extraction:** Fourth argument `0x3AB00000` and selector `0x20` produce byte `0xAB` at address `0x31`.

2. **Fresh selector:** If the initial selector is `0x20` and an intervening dependency changes it to `0x40`, the initial publication targets `0x31`, while the subsequent clear targets `0x51`.

3. **Second clear-pair ordering:** If the fresh phase-5 result is `3`, the second writes to region base plus `0x11` and plus `1` both precede the dependency receiving `12`. That call must not be moved before either write.

4. **Distinct duplicate-argument call:** If phase 10’s intermediate result is `5`, its next dependency receives `(20, 20)`. If the later payload-producing result is `7`, the secondary payload is `28`; `(28, 28)` is not an equivalent replacement.

5. **Saved selector:** If phase 8 saves `0x30` and a later dependency changes selector storage to `0x70`, the four-byte publication still targets `0x44`, not `0x84`.

6. **Wrapping region base:** Source indices zero and four both yield region base zero. Their direct region clears target `0x11` and `1`.

7. **Unaligned read:** If the region-read source’s extracted bits 20–29 have numerical value one, the four-byte read begins at `0x15`, without rounding.

8. **Byte versus full-width argument:** Region payload `0x000000FF` produces byte `0xFC` for its shifted selected store, while the corresponding dependency receives `0x000003FC`.

9. **Partial progress:** A nonreturning dependency after a publication prevents subsequent visible events but does not imply reversal of the earlier publication.

10. **Continuation:** Completion of every event in an iteration leads to the next iteration, not a normal return.

## Uncertainties

The following remain **documented_only**:

- Executable validity, instruction boundaries, and completeness of the representation.
- Entry calling convention and the meaning of additional incoming state.
- Indirect target identities, signatures, and implicit inputs or outputs.
- Machine semantics of signed shifts and overflowing arithmetic.
- Endianness, alignment, address mapping, fault handling, and access atomicity.
- Volatile, device-backed, shared, or concurrently modified storage.
- Global layout and possible overlapping storage descriptions.
- Dependency preservation of registers, globals, or memory.
- Dependency ownership, lifetime, synchronization, and error effects.
- Whether the environment permits a full iteration to complete.

Structured-field manipulation or device-register handling are possible interpretations only. Neither is required by this contract.

## Evidence

Established static evidence consists of the unconditional loop, explicit access widths and offsets, masks and scaling operations, ordered indirect calls, result splitting, repeated selector reads, saved-selector use, the global read, and loop-carried replacements.

The second phase-5 clear pair visibly separates acquisition of a fresh result from the call consuming its shifted value. Phase 10 visibly contains a separate two-argument call using identical shifted intermediate results before secondary-payload acquisition.

Warnings accompanying the representation report omitted unreachable material and overlapping global descriptions. They limit completeness and storage interpretation without establishing additional behavior.

No implementation access, runtime execution, smoke run, or tests were performed. Acceptance examples are conditional arithmetic and ordering consequences, not observed outcomes.
