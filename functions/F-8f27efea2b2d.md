<!-- Trusted provenance: F-8f27efea2b2d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8f27efea2b2d

## Purpose

F-8f27efea2b2d conditionally decrements a 32-bit object field. A decrement producing zero initiates recursive processing of selected subordinate records, a sequence of cleanup-related dependency calls, and allocation-accounting updates.

**Established behavior:** the function performs the accesses, calls, branches, and mutations specified below.

**Inferred intent:** the field resembles a reference count, the recursive calls resemble release of subordinate references, and the final operations resemble allocation disposal. These interpretations do not establish ownership or memory-release contracts for the dependencies.

A nonzero result from the entry dependency prevents all direct object accesses by this invocation. Otherwise, a field already equal to zero produces no decrement and no cleanup.

## Interface

### Argument and result

The function accepts one object-address argument, represented as a 32-bit integer. Address-like values loaded from object records are also 32 bits.

There is no explicit return value. Normal return occurs after:

- A nonzero entry-gate result.
- Finding the reference-like field already zero.
- Decrementing that field to a nonzero result.
- Completing the zero-transition path.

Invalid allocation metadata invokes F-a817d8d58c10, represented as nonreturning. There is no local error result, rollback, or recovery mechanism for dependency failures, invalid memory, or arithmetic outside the established execution domain.

### Directly accessed object layout

Offsets are byte offsets from the object address.

| Offset | Width or interpretation | Use |
|---|---|---|
| +4 | 32-bit signed representation | Tested against zero, then conditionally decremented |
| +16 | Unsigned 32-bit value | Repeatedly read as the subordinate-record count |
| +24 | Embedded address | Passed directly to F-1dc7fdb390e6 |
| +112 | 32-bit address-like value | Subordinate-array base and later dependency argument |

The embedded address at +24 is computed from the object address; this function does not load a pointer from that location.

F-c67cb20988a1 receives the object address and the numeric extent 116 bytes. That argument does not establish whether the dependency reads, clears, destroys, or releases any part of the object.

### Subordinate-record layout

Records are traversed with a stride of 20 bytes.

| Record offset | Width | Use |
|---|---:|---|
| +8 | 8 bits | Any nonzero value enables recursion |
| +16 | 32 bits | Address-like argument for the recursive invocation |

The other record bytes are not directly inspected.

### Allocation metadata

Metadata is accessed before the object address:

| Object-relative offset | Width | Use |
|---|---:|---|
| −8 | 32 bits | Header validation and accounting-amount extraction |
| −4 | Unsigned 16 bits | First accounting index |
| −2 | Unsigned 16 bits | Second accounting index |

The accounting amount is the header word masked with `0x3ffffffc`. The mask excludes the lowest two and highest two bits. Possible amounts are multiples of four from zero through `0x3ffffffc`.

### Memory and execution preconditions

A nonzero entry-gate result requires no direct access to object storage by this function. For every other path, all reached reads and writes must address suitable storage.

The zero-transition path additionally requires:

- Valid record storage for every record actually reached.
- Valid recursive arguments whenever their gates permit object access.
- Continued validity of object fields and preceding metadata across dependencies that precede later accesses.
- Valid captured allocator state and every reached accounting destination.
- Valid address arithmetic for the object, records, metadata, and accounting destinations.

These conditions are not checked locally. In particular, there is no early null-object check, bounds validation, cycle check, or independently enforced recursion limit.

## Behavior

### Entry gate

F-cf176702439d is called without explicit arguments before any direct object access.

A nonzero result ends this invocation immediately. A zero result permits object processing.

This guarantee concerns only this function’s direct operations. The entry dependency may itself access memory, alter the object or globals, or have other effects. Values present before that call are not necessarily the values subsequently read.

### Reference-like field

After a zero gate result, the function reads the 32-bit field at +4.

If the value is zero, it returns without writing that field.

If the value is nonzero, the field is read for a decrement, one is subtracted, and the result is written back. The decision to continue uses that computed result:

- A nonzero result returns immediately.
- A zero result enters cleanup.

For ordinary stable values, one therefore becomes zero and initiates cleanup, while three becomes two and returns. Negative values are not rejected.

The field has a signed 32-bit representation in the supplied evidence. Decrementing the minimum signed value is outside the established portable arithmetic contract; neither saturation nor defined wraparound is promised. The function does not reread the field to decide whether cleanup should continue after cleanup dependencies begin.

### Subordinate traversal

On the zero transition, the function reads the array address at +112.

A zero address skips both traversal and F-1137792fba7d.

A nonzero address establishes the initial record cursor. Traversal uses a 32-bit unsigned ordinal beginning at zero. Before processing each record, that ordinal is compared against a fresh unsigned 32-bit read of the count at object offset +16.

For each reached record:

- The byte at record offset +8 is read.
- If it is nonzero, the 32-bit argument at record offset +16 is read and passed to F-8f27efea2b2d.
- After any recursive invocation returns, the cursor advances by 20 bytes and the ordinal advances by one.

When the count comparison terminates traversal, the function rereads the object’s +112 field and passes that value to F-1137792fba7d.

Observable consequences include:

- The count is not a snapshot. A recursive invocation can change later loop decisions by changing the count.
- The cursor remains based on the initially captured array address. Changing the object’s array field does not redirect traversal.
- F-1137792fba7d receives the freshly read array field, which may differ from the traversed base and may even be zero.
- An initially nonzero array field with a count of zero still leads to F-1137792fba7d, without reading a record flag.
- Every nonzero flag value enables recursion; the flag need not equal one.
- A flagged record whose argument is zero still causes a recursive invocation. That invocation’s own gate determines whether it proceeds to object access.
- There is no deduplication or cycle detection. Repeated references can cause repeated calls.

Changing counts and invalid cursor arithmetic have no local safety fallback. Termination and memory safety depend on the reached values, dependency effects, and accessible storage.

### Cleanup calls and state selection

After the optional array path, the function makes these calls in order:

1. F-1dc7fdb390e6 with the embedded address at object offset +24.
2. F-c67cb20988a1 with the object address and 116.
3. F-2fd8826557a4 without explicit arguments.

It then reads the shared allocator-state address and retains that value for subsequent bookkeeping and final calls.

State selection therefore occurs after F-2fd8826557a4. Earlier cleanup dependencies may affect which state is selected. Once captured, the state address is not refreshed from the shared location, even if a later dependency changes that location.

The allocation metadata has not been saved before these calls. Later accesses require that metadata to remain readable despite the dependencies’ effects.

### Late object-address check and header validation

After capturing allocator state, the function tests the original object argument against zero.

A zero argument skips header validation and accounting. This late check does not make zero an ordinarily safe input: the reference-like field was already accessed on any path reaching this point.

For a nonzero argument, validation occurs as follows:

- Read the 32-bit header at −8 and test it against zero.
- If it is nonzero, read the header again and test its least significant bit.
- Invoke F-a817d8d58c10 if the first test finds zero or the second test finds the low bit set.
- Otherwise call F-c0fc6d033caa with the captured allocator-state address.

The invalid-header dependency is represented as nonreturning. No successful recovery path is established.

Validation occurs after recursive processing and the preceding cleanup calls. It cannot prevent their effects. The validated header is also not retained as the accounting snapshot: the function reads it again after F-c0fc6d033caa, without repeating validation.

### Accounting sequence

All state-relative offsets below use the captured allocator-state address. All accounting destinations are 32-bit slots.

| Order | Operation |
|---:|---|
| 1 | Read the unsigned 16-bit first index at object offset −4. |
| 2 | Read the header at −8 and capture its value masked with `0x3ffffffc`. |
| 3 | Subtract the captured amount from state offset `0x224 + 4 × first index`. |
| 4 | Subtract the same amount from state offset `0x30c`. |
| 5 | Read the unsigned 16-bit second index at object offset −2. |
| 6 | Subtract the same amount from state offset `0xe0 + 4 × second index`. |
| 7 | Subtract the same amount from state offset `0x220`. |
| 8 | Call F-09dcb55e8e0a without explicit arguments. |
| 9 | Decrement the slot at state offset `0x310 + 4 × returned index`. |
| 10 | Decrement the slot at state offset `0x390`. |
| 11 | Read the header again, mask it with `0x3ffffffc`, and subtract that amount from state offset +8. |
| 12 | Call F-cc6c9b4a0f68 with captured state and the address eight bytes before the object. |
| 13 | Call F-e586bff64fb7 with captured state. |

This is **five amount-based subtractions and two count decrements: seven direct accounting updates** on a completed accounting path.

The first four amount-based updates share one captured amount. The fifth uses a later header read, after F-09dcb55e8e0a and the count decrements. A dependency mutation or an aliased intervening write can therefore affect the later amount without changing the earlier captured amount.

The amount-based operations use unsigned 32-bit arithmetic, so their results are reduced modulo \(2^{32}\). No underflow check is present. The two count decrements use signed 32-bit representations; behavior at signed overflow boundaries is not established as portable wraparound.

The first two indices are zero-extended from 16 bits. The index returned by F-09dcb55e8e0a has a signed 32-bit representation, with no local rejection of negative or out-of-range values. Index multiplication and address addition do not establish valid or portable behavior for arbitrary inputs.

Destinations are not proven distinct. When accounting locations alias, the sequential updates accumulate at the same storage. Aliasing with subsequently read metadata or indices can also affect later operations. The table specifies operation order, not an independently established machine-level ordering of operand reads within an individual subtraction.

## Side effects

Depending on the path, direct effects include:

- A single write of the decremented reference-like field.
- Recursive invocations for reached records with nonzero flags.
- Five amount-based accounting writes and two count-decrement writes.
- Calls receiving subordinate-array, embedded-storage, object, allocator-state, and allocation-header addresses.

The seven accounting updates remain seven operations even if some destinations alias or an amount is zero. An earlier failure or nonreturning dependency may prevent later operations.

The function does not directly clear record flags, rewrite subordinate arguments, null the array field, or reset the count.

No dependency is established to release memory, preserve globals, preserve object fields, or retain ownership. In particular, later accesses must not be justified by assuming that a cleanup dependency leaves storage valid.

No atomic decrement or explicit synchronization operation is directly established. Dependencies might provide synchronization, but their contracts are unknown. Concurrent mutation is not given a safe execution model by this evidence.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-cf176702439d | Entry gate; nonzero result ends direct processing |
| F-8f27efea2b2d | Recursive processing of each reached flagged record |
| F-1137792fba7d | Receives the freshly read array field after traversal |
| F-1dc7fdb390e6 | Receives the embedded address at +24 |
| F-c67cb20988a1 | Receives the object address and 116 |
| F-2fd8826557a4 | Runs immediately before shared allocator-state selection |
| F-a817d8d58c10 | Represented as nonreturning for either invalid-header condition |
| F-c0fc6d033caa | Receives captured allocator state before accounting |
| F-09dcb55e8e0a | Returns the index for one count decrement |
| F-cc6c9b4a0f68 | Receives captured state and the allocation-header address |
| F-e586bff64fb7 | Final call with captured state |

These interactions do not independently establish the dependencies’ internal behavior or ownership obligations.

## Acceptance examples

These are conditional contract examples, not observed executions. Each requires valid memory and arithmetic for its reached operations. Any dependency preceding a stated later event must return normally, except the explicitly nonreturning invalid-header case. Field assumptions apply at the specified reads, not merely before entry.

### Entry suppression

If F-cf176702439d returns nonzero, this invocation performs no direct object access, decrement, traversal, or accounting. Effects inside the gate remain unconstrained.

### Already-zero field

If the gate returns zero and the subsequent +4 test reads zero, the function returns without writing that field or entering cleanup.

### Nonterminal decrement

If the gate returns zero and both the +4 test and decrement read obtain three, the function writes two and returns without reading the array field.

### Terminal decrement without an array

If the gate returns zero, both reference-field reads obtain one, and the subsequent +112 read obtains zero, the function writes zero, skips traversal and F-1137792fba7d, and calls F-1dc7fdb390e6 with the embedded address at +24.

This example promises no later completion unless the additional reached dependencies, metadata, and accounting destinations satisfy their requirements.

### Empty nonzero array

If the gate returns zero, both reference-field reads obtain one, the initial array read is nonzero, and the first count comparison reads zero, no record flag is read.

The function nevertheless calls F-1137792fba7d with the array value read after that comparison. That argument equals the original base only if the field retains that value until the argument read.

### Mixed flags

If the gate returns zero, the decrement produces zero, and the initial array read is nonzero, traversal occurs subject to each count comparison.

If those comparisons read three throughout, the three reached flag reads obtain zero, two, and zero, and recursive processing does not invalidate the remaining record accesses, only the second record causes recursion. Its argument is whatever its +16 field contains when read.

### Changed loop bound

If the gate returns zero, the decrement produces zero, a nonzero array is captured, and the first count comparison reads three, the first record is reached.

If that record triggers a returning recursive call that changes the count to one, and the next comparison reads one, traversal stops after the first record.

### Invalid header

If the gate returns zero, the decrement produces zero, and all preceding traversal and cleanup operations complete, a first validation read of zero invokes F-a817d8d58c10 before F-c0fc6d033caa or direct accounting.

A nonzero first validation read followed by a second read with the least significant bit set produces the same call. A header consistently equal to `0x105` is one such case.

### Stable accounting amount

If the accounting path is reached, validation reads obtain `0x106`, and both subsequent amount reads also obtain `0x106`, each amount is `0x104`.

With valid, nonaliasing destinations and returning dependencies, the function performs five subtractions of `0x104` and two decrements by one. Dependency-internal writes remain additional effects, not part of these seven direct updates.

### Changed later amount

If the accounting path is reached and the first captured amount is `0x104`, the first four amount-based updates use `0x104`.

If F-09dcb55e8e0a changes the header and the final header read obtains `0x20a`, the fifth amount-based update subtracts `0x208`. There is no repeated header validation before that subtraction.

## Uncertainties

The following remain **documented_only**:

- The meaning and side effects of the entry gate.
- Whether the +4 field is formally a reference count.
- Ownership, release, and memory-lifetime guarantees of every dependency.
- The source and lifetime of shared allocator state.
- Valid index ranges and intended accounting-table boundaries.
- Whether accounting locations, object storage, or metadata can alias.
- Header-bit meanings beyond the explicit validation and mask.
- Whether any dependencies form a synchronization protocol.
- Signed-overflow and address-overflow behavior outside ordinary valid executions.
- Whether cycles, repeated references, or concurrent mutation are prohibited externally.
- Exceptional or nonreturning behavior of dependencies other than the represented invalid-header path.
- Behavior if the nonreturning characterization of F-a817d8d58c10 is incorrect.

## Evidence

The supplied function representation establishes the direct access widths, offsets, branch conditions, recursive structure, dependency arguments, repeated reads, accounting mask, and operation sequence described here.

Reference counting, destruction, allocation release, and synchronization are inferred intent, not established dependency contracts. The nonreturning characterization of F-a817d8d58c10 is supplied by the representation; its implementation is unavailable.

No runtime tests have been performed. Acceptance examples express conditional behavior supported by static evidence, not execution results.
