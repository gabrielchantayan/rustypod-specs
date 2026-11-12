<!-- Trusted provenance: F-bb1549df1bd4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bb1549df1bd4

## Purpose

F-bb1549df1bd4 performs an ordered, teardown-like transition on an input object. It replaces dispatch-related words, validates selected state, obtains an adjusted object address, processes optional related objects, and conditionally updates global registration state.

**Established static behavior:** five assignments obtain their replacement words from five distinct global operands. A sixth distinct global operand supplies an argument to a notification dependency. These operands are read at their respective use sites; they are not established constants.

**Inferred intent:** the dispatch transitions and dependent operations resemble destruction across object layers. This does not establish ownership, deallocation, complete destruction, or whether the returned address remains usable.

**Unknown:** concrete object types, dependency implementations, global operand values and storage properties, and indirect-call semantics. These remain documented_only.

## Interface

### Input, result, and widths

The function accepts one object address, designated **P**. The supplied representation uses 32-bit words and 32-bit address-bearing values. Offsets below are in bytes.

On normal completion, the function returns an address designated **B**:

- F-230c59ef0030 receives P + 0xA8.
- Its result is designated **A**.
- B is obtained by subtracting 0xA8 from A.

B is not established to equal P. Subsequent accesses through B must not be described as accesses through P without that additional condition.

There is no visible status-code result. Two failure paths invoke a dependency designated non-returning.

### Object and context layout

Except for the selector byte, all listed fields are 32-bit words.

| Location | Observed use |
|---|---|
| P + 0x00 | Destination of the first two dispatch-related assignments |
| P + 0xA8 | Address supplied to F-230c59ef0030 |
| P + 0xE8 | One-byte selector, conditionally read twice |
| P + 0xEC through +0x100 | Six consecutive validation words |
| P + 0x104 through +0x140 | Sixteen consecutive validation words |
| B + 0x00 | Destination of the remaining three dispatch-related assignments |
| B + 0x04 | Optional associated-object reference |
| B + 0x08 | Final invariant, checked only on the associated-object branch |
| B + 0x0C | Optional auxiliary reference |

For a nonzero auxiliary address, designated **Q**:

| Location | Observed use |
|---|---|
| Q + 0x04 | Word saved as **V** |
| Q + 0x08 | Optional collection reference, read at multiple points |

F-3fdd14ecd250 supplies context addresses with these observed fields:

| Context offset | Observed use |
|---|---|
| +0x1C | First argument to F-77781c8e0ff9 |
| +0x24 | Compared with B; conditionally cleared through a separately obtained context |
| +0x28 | Tested, used for indirect dispatch and dependency arguments, and compared with B |

These are partial access layouts, not complete type definitions.

### Global operands

The following labels identify distinct global words without assigning values or meanings to them.

| Global operand | Use site |
|---|---|
| **G1** | Initial assignment to P + 0x00 |
| **G2** | Assignment to P + 0x00 after validation or bypass |
| **G3** | Assignment to B + 0x00 immediately after deriving B |
| **G4** | Assignment to B + 0x00 after F-a6fdc7050c77 |
| **G5** | Final assignment to B + 0x00, after optional auxiliary processing |
| **G6** | Second argument to F-77781c8e0ff9 |

Each assignment reads its corresponding global operand at that assignment site. G6 is read while preparing the notification call, after the preceding context-provider call.

Distinct storage operands do not imply distinct contents. Their values, mutability, initialization, linkage, protection, and possible aliasing relationships are undocumented. No intervening dependency is assumed to preserve them.

### Preconditions

Defined normal execution requires every reached read, write, and indirect call to be valid. This includes the relevant global operands, selected object fields, context fields, dispatch tables, and temporary-state accesses performed by dependencies.

The address adjustment must yield a usable B. Optional references must remain suitable for every subsequent access through them, including accesses after dependencies that may modify state.

There is no visible null-input check, address-range validation, overflow guard, or dispatch-entry validation. Endianness and malformed-address behavior are not established.

## Behavior

### Initial replacement and validation

The function first reads G1 and writes the resulting word to P + 0x00. This occurs before the selector or validation fields are read.

The first byte read at P + 0xE8 selects the following behavior:

- Zero selects six full-word checks, in ascending order, from P + 0xEC through P + 0x100.
- Nonzero causes a second read of the same selector byte. Only a second-read value of one selects the sixteen full-word checks from P + 0x104 through P + 0x140.
- Any second-read value other than one bypasses both ranges. In particular, a change from nonzero to zero does not select the six-word range.

Every selected word must be zero. The first nonzero word causes F-163ea045cd81, followed by F-feee9156a257, with no visible explicit arguments to either call. The latter is designated non-returning. Later validation words and normal continuation are not reached under that annotation.

The selector’s adjacent bytes do not participate in its comparisons. The two possible selector reads must not be collapsed if externally changing memory is relevant.

### Address adjustment and relationship processing

After successful validation or bypass, the function reads G2 and writes it to P + 0x00. It then calls F-7e3145ad059a with P, followed by F-230c59ef0030 with P + 0xA8.

After deriving B from the latter result, it reads G3 and writes it to B + 0x00. Thus G3 is sampled after both dependencies; its content is not assumed to match any earlier value.

F-d277584d672d then receives B. A zero result skips relationship dispatch. A nonzero result designates a dispatch-bearing object:

- Its dispatch entry at byte offset 0x80 is invoked with that object as the visible argument.
- If the returned address equals B, its dispatch entry at byte offset 0x7C is invoked with that same object and zero.
- A different returned address skips the second operation.

The second operation obtains its dispatch target after the first call; the dispatch-bearing object is not assumed unchanged.

F-a6fdc7050c77 subsequently receives B, zero, and one, regardless of whether relationship dispatch occurred.

### Context notification

The function reads G4 and writes it to B + 0x00 after F-a6fdc7050c77 returns. It then calls F-408b56b8b7c8 with B.

Next, F-3fdd14ecd250 supplies a context address. F-77781c8e0ff9 receives:

- The word at that context’s +0x1C field.
- The word read from G6 at this notification site.
- B.

G6 is not a fixed notification constant. Neither its initial value nor its preservation across earlier dependencies is established. The representation establishes that both argument-source reads occur before the notification call, but does not establish an independent ordering between those two reads.

Only after notification does the function read B + 0x0C and save it as Q. If Q is zero, all auxiliary, collection, and registration processing described below is skipped.

### Auxiliary predicate and context-dependent operation

For nonzero Q, the word at Q + 0x04 is saved as V before F-a1a48bd3fbde receives B.

A zero predicate result skips the remainder of this phase, but does not skip later collection or registration processing.

For a nonzero predicate result, a newly obtained context has its +0x28 field tested:

- If that field is zero, the context-dependent indirect operation and F-a5514b9f24f7 are skipped.
- If nonzero, another context-provider call supplies a context whose +0x28 reference is used to obtain an indirect operation at dispatch-table byte offset 0x48. No explicit arguments are visible for that indirect call.
- After that indirect call, B + 0x0C is reread, and offset +0x04 of that newly read auxiliary address is read.
- A further context-provider call supplies its +0x28 word as the first argument to F-a5514b9f24f7. The freshly read auxiliary word is its second argument.

The refreshed auxiliary word is not necessarily V, and the refreshed auxiliary address is not necessarily Q. The read occurs before the provider call supplying the first argument.

There is no second nonzero check between the first +0x28 test and dispatch through the next provider result. Likewise, the reread auxiliary address is dereferenced without a new null check.

On every nonzero-predicate path, including when the initial context +0x28 test fails, F-65528ded3450 receives B and zero, followed by F-a82ec4b43e2f receiving saved V.

### Collection traversal

Collection processing continues to use saved Q, not the possibly changed B + 0x0C.

Q + 0x08 is tested. If zero, traversal initialization, iteration, finalization, and the subsequent collection operation are all skipped.

If nonzero, Q + 0x08 is read again for F-538044b67754. That dependency receives:

- The address of a temporary state area represented as 20 bytes.
- The reread collection reference.
- The 32-bit value 0xFFFFFFFD.

The constant represents −3 under a signed 32-bit two’s-complement interpretation. Its semantic meaning and the dependency’s signedness expectations are unknown.

F-ebd94610a438 repeatedly receives the traversal state and an output-reference location. A zero result ends traversal. A nonzero result permits inspection of the output reference:

- A zero output reference produces no element operation and does not end traversal.
- A nonzero output reference supplies an indirect operation at dispatch-table byte offset 0x04. No explicit arguments are visible.

When advancement returns zero, F-944b595e25e9 receives the traversal state.

The function then tests Q + 0x08 again. If nonzero, it reads the reference again to obtain an indirect operation at dispatch-table byte offset 0x04. Consequently, this operation is not guaranteed to target the reference used for traversal initialization, nor even the value observed by its immediately preceding nonzero test.

No iteration bound is visible. A dependency that continually returns nonzero can prevent later phases from being reached. No memory-release meaning is established for either indirect operation.

### Registration checks and auxiliary finalization

For every Q-nonzero path reaching this phase:

- A fresh context’s +0x28 word is compared with B. Equality causes F-a82ec4b43e2f to receive V.
- Another fresh context’s +0x24 word is compared with B. Equality causes one more context-provider call, followed by a zero write to +0x24 of that newest context.
- F-65528ded3450 receives B and zero.
- F-8349c252c168 receives saved Q and zero.

The context receiving the zero write is not rechecked for equality. It need not be the context used for the comparison.

F-a82ec4b43e2f can receive V twice: once following a nonzero auxiliary predicate and once following the later +0x28 equality. F-65528ded3450 likewise occurs once on every normally progressing Q-nonzero path and an additional time for a nonzero predicate. No duplicate suppression is visible.

### Final associated-object phase

After auxiliary processing, or immediately after its bypass, G5 is read and written to B + 0x00. Its value is sampled only at this late site, after all preceding returning dependencies.

The function tests B + 0x04:

- If zero, it returns B without checking B + 0x08.
- If nonzero, F-c719314a16bb receives B. The associated reference is then reread and tested.
- If that reread is nonzero, the reference is read again to obtain an indirect operation at dispatch-table byte offset 0x04. No explicit arguments are visible.
- Regardless of whether the post-dependency reference remains nonzero, B + 0x08 is checked.

A nonzero final invariant invokes F-c8eefdf96f4b and then designated non-returning F-feee9156a257. Otherwise, B is returned.

The reference selecting the branch, the reference tested after F-c719314a16bb, and the reference used for dispatch are not guaranteed identical.

## Side effects

Directly established persistent writes are:

- Two word assignments to P + 0x00, sourced respectively from G1 and G2.
- Three word assignments to B + 0x00, sourced respectively from G3, G4, and G5.
- A conditional zero write to a freshly obtained context’s +0x24 field.

If P equals B, all five dispatch-related assignments affect the same address in that order. Their source words are still read separately at their respective sites.

The function directly reads six distinct global operands. It contains no explicit assignment to those global operands. This does not establish immutability: dependencies, external activity, or unestablished aliasing may affect their contents.

Traversal dependencies receive temporary-state and output-reference storage. Their writes and lifetime requirements are not fully specified.

No direct assignment clears the validation fields, B + 0x04, B + 0x08, B + 0x0C, or Q + 0x08. Dependencies may modify those fields or release referenced storage; preservation and ownership are not assumed.

No rollback is visible. Initial validation failure follows the G1-sourced assignment. Final invariant failure follows the G5-sourced assignment and preceding processing.

## Dependencies

| Opaque function ID | Visible role |
|---|---|
| F-163ea045cd81 | Initial validation-failure precursor |
| F-feee9156a257 | Designated non-returning failure transfer |
| F-7e3145ad059a | Receives P before address adjustment |
| F-230c59ef0030 | Supplies A from P + 0xA8 |
| F-d277584d672d | Supplies optional relationship object |
| F-a6fdc7050c77 | Receives B, zero, one |
| F-408b56b8b7c8 | Receives B after the G4 assignment |
| F-3fdd14ecd250 | Repeated context provider |
| F-77781c8e0ff9 | Receives context field, G6-sourced word, B |
| F-a1a48bd3fbde | Supplies auxiliary predicate |
| F-a5514b9f24f7 | Receives context reference and refreshed auxiliary word |
| F-65528ded3450 | Receives B and zero, potentially twice |
| F-a82ec4b43e2f | Receives saved V, potentially twice |
| F-538044b67754 | Initializes temporary traversal |
| F-ebd94610a438 | Advances traversal and supplies output reference |
| F-944b595e25e9 | Finalizes traversal state |
| F-8349c252c168 | Receives saved Q and zero |
| F-c719314a16bb | Precedes associated-reference rereads |
| F-c8eefdf96f4b | Final invariant-failure precursor |

Indirect dependencies are identified by dispatch-table byte offsets 0x80, 0x7C, 0x48, and 0x04. Complete calling conventions and ownership contracts are documented_only.

F-4455d83b1b92 identifies a shared continuation in the supplied representation; no separate call is established.

## Acceptance examples

These are conditional static consequences, not executed tests. Unless stated otherwise, accesses are valid and dependencies return.

1. **First validation failure.** With selector zero and P + 0xEC nonzero, G1 is read and assigned before F-163ea045cd81 and F-feee9156a257. G2 and later global operands are not reached.

2. **Last one-mode failure.** With a stable selector of one, fifteen zero words through P + 0x13C and a nonzero word at +0x140 cause failure after all sixteen selected checks. The six-word range is not validated.

3. **Selector change.** First read one and second read zero bypass both validation ranges. A stable selector of two also bypasses them.

4. **Use-site global sampling.** If F-7e3145ad059a changes G3, the subsequent B assignment uses the value read after that dependency, not a value cached at entry. If F-408b56b8b7c8 changes G6, the notification uses its later-read value.

5. **Late dispatch operand.** If an auxiliary dependency changes G5, the final B + 0x00 assignment uses G5 as read after auxiliary processing. Earlier dispatch assignments impose no equality requirement on that value.

6. **No auxiliary or associated object.** Zero Q skips auxiliary processing. A zero final B + 0x04 skips the invariant check even when B + 0x08 is nonzero.

7. **Nonzero predicate, zero context reference.** F-65528ded3450 and F-a82ec4b43e2f still occur, but the +0x48 operation, refreshed auxiliary read, and F-a5514b9f24f7 do not.

8. **Null traversal element.** A nonzero advancement result with a zero output reference skips only that element operation. A zero advancement result ends traversal and leads to finalization.

9. **Changed context for clearing.** If the comparison context has +0x24 equal to B, a different context returned next receives the zero write without another equality check.

10. **Associated reference cleared.** If F-c719314a16bb clears B + 0x04, associated dispatch is skipped, but B + 0x08 is still checked. A nonzero invariant invokes the final failure sequence.

## Uncertainties

- **documented_only:** concrete object types, field meanings, dispatch semantics, and whether B equals P.
- **documented_only:** values, mutability, initialization, storage properties, and aliasing of G1–G6.
- **documented_only:** dependency effects on globals, references, allocation lifetimes, and reentrancy.
- **documented_only:** ownership, release behavior, and post-return usability.
- **documented_only:** exceptional exits, unwinding, and the mechanism implementing designated non-returning behavior.
- **documented_only:** concurrency guarantees and stability between separate reads or provider calls.
- **documented_only:** malformed-address arithmetic semantics. The represented adjustment is 32-bit subtraction by 0xA8 without an explicit guard; no checked-overflow or portable signed-overflow contract is established.
- **Inferred intent only:** layered destruction, detachment, deregistration, and collection teardown.

## Evidence

The contract derives from the supplied static function representation, using the dependency mapping only to assign opaque function IDs. The prior draft and feedback are not independent evidence.

The representation supports the widths, offsets, branch conditions, global-source reads, direct writes, dependency ordering, rereads, and normal return expression documented above. Non-returning annotations are reported as supplied static evidence, not verified runtime properties.

No runtime tests have been performed. Acceptance examples describe conditional behavior supported by the representation, not observed execution.
