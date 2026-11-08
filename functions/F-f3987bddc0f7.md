<!-- Trusted provenance: F-f3987bddc0f7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f3987bddc0f7

## Purpose

F-f3987bddc0f7 conditionally obtains a candidate record and either delivers it through a receiver-selected callback or performs header validation, accounting updates, and a release-like dependency sequence.

Two ordered receiver observations gate all dependency calls. The first is an explicit character read. The second is a 32-bit word read followed by conversion to a character; it is not an explicit byte read. After acquisition, a candidate discriminator and the requested category select delivery or release.

“Record,” “accounting context,” and “release” describe inferred roles. The supplied static representation establishes accesses and calls, but not the surrounding allocation, synchronization, or ownership contracts.

## Interface

**Return:** no result value. Ordinary return does not distinguish closed gates, unsuccessful acquisition, delivery, or completion of the release path. One dependency is explicitly marked non-returning.

### Inputs

| Input | Representation | Observed use |
|---|---|---|
| Receiver | Pointer in the represented 32-bit address model | Supplies gating storage and the callback-table pointer. |
| Category | 32-bit integer | Tested using unsigned 32-bit arithmetic; stored unchanged on delivery. |
| Payload | 32-bit integer | Stored unchanged on delivery. |
| Auxiliary pointer | Pointer-sized value | Initially copied into a local pointer slot after both gates pass. No explicit dereference or outward use of that initial value is shown. |

Four-byte integer words and the stated pointer offsets are established by the representation. Exact calling conventions and portable behavior on other data models are not established.

### Receiver layout

| Byte offset | Access width | Observed use |
|---|---:|---|
| `0x00` | 32 bits | Callback-table pointer, read on delivery after the candidate writes. |
| `0x18` | 32 bits | Second-gate source word, conditionally read and then converted to a character. |
| `0x19` | 8 bits | First gate, explicitly read as a character. |

The second-gate word spans offsets `0x18` through `0x1B`, including the address used for the first gate. Nevertheless, these are separate, differently sized accesses at different points in the execution.

The callback entry is at byte offset `0x1C` within the table selected through receiver offset `0x00`.

The first gate passes when its character value is nonzero. The second passes when the character obtained by converting the loaded 32-bit word is nonzero.

The second condition must not be replaced unconditionally with “the byte at receiver offset `0x18` is nonzero.” That equivalence requires both an appropriate byte order and a conversion rule. For example, little-endian storage combined with conversion retaining the low eight bits would establish that equivalence. Neither assumption is independently established here.

### Candidate layout

For a nonnull acquired candidate:

| Relative byte offset | Width | Observed use |
|---|---:|---|
| `-8` | 32 bits | Header validation and masked-size reads. |
| `-4` | 16 bits, unsigned | Index for the first indexed accounting update. |
| `-2` | 16 bits, unsigned | Index for the second indexed accounting update. |
| `0` | 32 bits | Category destination on delivery. |
| `4` | 32 bits | Payload destination on delivery. |
| `8` | 32 bits | Discriminator selecting delivery or release. |

The two indices are native unsigned 16-bit accesses at the stated addresses. Their byte order is not separately established.

### Preconditions

No general pointer, bounds, alignment, or callback validation is performed.

- Every invocation requires a receiver supporting the explicit character read at `0x19`.
- If the first gate passes, the receiver must also support the represented **32-bit read** beginning at `0x18`. Readability of only the byte at `0x18` is insufficient.
- A nonnull candidate must support the discriminator read at offset `8`.
- Delivery requires writable candidate words at offsets `0` and `4`, readable callback-selection storage, and a callable selected entry.
- Release requires readable header storage, a usable captured context pointer, and valid accounting addresses selected by the indices and dependency result.
- Memory must remain valid when accessed despite any preceding dependency effects.

A null receiver is not treated as a benign no-op.

## Behavior

### Receiver gating

The first observation is an explicit character read at receiver offset `0x19`.

If its value is zero, the function returns. The second-gate word is not read, and no dependency is called.

If the first value is nonzero, the function reads the 32-bit word beginning at receiver offset `0x18`, converts that value to a character, and checks the converted value against zero. A zero converted value causes return without dependency calls.

Thus, a nonzero source word alone does not establish that the second gate passes. Under a low-eight-bit conversion rule, a word such as `0x00000100` converts to zero; without an established conversion rule, that particular result remains conditional.

The two observations are not an atomic receiver snapshot. Their overlapping storage does not merge them into one access, and no gate is checked again after a dependency call.

### Candidate acquisition

After both gates pass:

1. The auxiliary pointer is copied into a local pointer slot.
2. F-8157a180d901 is called with the integer argument `12`.
3. F-2573f37ac5e3 is called without explicit arguments.
4. Its result is treated as the candidate pointer.

A null result causes immediate ordinary return, with no candidate discriminator access.

The argument `12` is established; interpreting it as an allocation size is inferred. No explicit argument or returned-value flow connects the first dependency to the second dependency’s result.

The auxiliary pointer’s initial local copy is not explicitly passed onward or dereferenced. The slot is overwritten before its address is supplied to the delivery callback.

### Path selection

For a nonnull candidate, its 32-bit discriminator at offset `8` is read first.

Release is selected exactly when:

- The discriminator is zero; and
- The category is outside the unsigned interval `22` through `33`, inclusive.

The range test subtracts `22` in 32-bit unsigned arithmetic and compares the result with `11`. Unsigned subtraction wraps modulo \(2^{32}\).

Consequently:

- Categories `22` and `33` are inside the accepted interval.
- Categories `21` and `34` are outside it.
- Negative signed categories are outside it after their unsigned conversion.
- A nonzero discriminator selects delivery regardless of category.

There is no subsequent discriminator recheck.

### Delivery

Delivery performs these externally relevant operations in order:

1. Store the category’s 32-bit representation at candidate offset `0`.
2. Store the payload’s 32-bit representation at candidate offset `4`.
3. Place the candidate pointer in the local pointer slot.
4. Read the receiver’s callback-table pointer and resolve the entry at table offset `0x1C`.
5. Invoke the selected callback with the receiver and the address of the local pointer slot.
6. Return if the callback returns normally.

The callback receives an indirect candidate pointer. It can inspect, replace, or clear the slot. This function does not inspect the slot after the call and performs no explicit post-callback release.

Any callback result is ignored. Ownership transfer, retention rights, and callback error conventions are unknown.

Both candidate writes precede callback resolution. No non-aliasing guarantee is established. If the candidate overlaps receiver or callback-selection storage, these writes can affect the later callback lookup.

### Release preparation and validation

The release path does not perform the delivery stores. It:

1. Calls F-2fd8826557a4 without explicit arguments.
2. Loads the accounting-context pointer through a global pointer source.
3. Uses the address eight bytes before the candidate as the header address.
4. Validates the header.

The context is captured after F-2fd8826557a4 returns. Changes that dependency makes to the global source can therefore affect which context is captured.

Validation first reads the 32-bit header word and checks whether it is zero. If it is nonzero, the represented expression reads the header word for the low-bit check. A zero first observation or a set low bit in the subsequent observation invokes F-a817d8d58c10, which is marked non-returning.

For stable header storage, validation accepts precisely nonzero words with bit zero clear. The representation does not establish that the two validation observations form one snapshot.

Validation does not establish that:

- The candidate belongs to the captured context.
- Either header index is within an allocated table.
- The masked size is nonzero.
- Later header reads will reproduce the validated value.

### Accounting and release sequence

After validation, F-c0fc6d033caa is called with the captured context pointer.

Only after that call returns is the header read for the first masked size. The mask is `0x3FFFFFFC`: it excludes the two lowest and two highest bits. The resulting unsigned size ranges from zero through `0x3FFFFFFC` and is a multiple of four.

Using that one size value, the function performs these 32-bit accounting updates in order:

| Order | Context-relative byte address | Direct operation |
|---:|---|---|
| 1 | `0x224 + 4 × first header index` | Subtract the first masked size. |
| 2 | `0x30C` | Subtract the first masked size. |
| 3 | `0x0E0 + 4 × second header index` | Subtract the first masked size. |
| 4 | `0x220` | Subtract the first masked size. |

The first index comes from candidate offset `-4`; the second comes from offset `-2`. They are not both captured before the accounting sequence. Their source expressions occur at their respective indexed updates.

The representation repeats an index expression within each indexed assignment. It does not establish a single index read or a complete ordering of address calculation and right-hand-side reads within that assignment. With stable index storage, each update addresses the corresponding location in the table.

The function then:

1. Calls F-09dcb55e8e0a without explicit arguments.
2. Decrements the 32-bit context word at word index `0xC4 + selector result` by one.
3. Decrements the context word at byte offset `0x390` by one.
4. Reads the header again, applies `0x3FFFFFFC`, and subtracts this newly obtained size from the context word at byte offset `0x08`.
5. Calls F-cc6c9b4a0f68 with the captured context and header pointers.
6. Calls F-e586bff64fb7 with the captured context pointer.
7. Returns if those calls return normally.

The selector is not locally range-checked. Neither accounting underflow nor pointer-address overflow is checked.

The first four size updates use the same captured size. The final size update uses a later header read and can therefore use a different size. The captured context pointer is not reloaded from the global source, even if later dependencies change that source.

Accounting addresses are not guaranteed distinct. Indexed entries may coincide with fixed fields or one another. Updates to coincident addresses occur sequentially rather than as one simultaneous balance calculation.

The masked-size operands are unsigned 32-bit values, so the represented subtraction expressions use unsigned 32-bit arithmetic. Storing an out-of-range unsigned result into a signed accounting word has target-dependent interpretation. The two count decrements use signed integer expressions; portable signed-overflow behavior is not established.

## Side effects

Established direct effects are:

- An explicit first-gate character read.
- Conditional reading of a complete 32-bit second-gate word, followed by character conversion.
- Acquisition-related dependency calls when both gates pass.
- A discriminator read for a nonnull candidate.
- Ordered category and payload stores on delivery.
- Callback-selection reads and invocation on delivery.
- Global context capture, header reads, accounting reads and writes, and release-path dependency calls.

Closed gates cause no direct writes or dependency calls. Null acquisition prevents later candidate accesses.

Invalid release headers reach the non-returning dependency only after release preparation and context capture. No ordinary accounting updates follow that dependency.

There is no explicit rollback. Direct updates already performed precede later dependency calls, but final balances cannot be inferred without assumptions about those dependencies’ effects.

No status code, error object, or diagnostic text is produced directly.

## Dependencies

| Opaque ID | Explicit interface used here | Established position or role |
|---|---|---|
| F-8157a180d901 | Integer argument `12`; result unused | Before acquisition. |
| F-2573f37ac5e3 | No explicit arguments; pointer result | Supplies candidate or null. |
| F-2fd8826557a4 | No explicit arguments; result unused | Before context capture. |
| F-a817d8d58c10 | No explicit arguments; non-returning | Rejected-header path. |
| F-c0fc6d033caa | Captured context pointer | After validation, before size accounting. |
| F-09dcb55e8e0a | No explicit arguments; integer result | Selects a counter to decrement. |
| F-cc6c9b4a0f68 | Captured context pointer and header pointer | After all direct accounting updates. |
| F-e586bff64fb7 | Captured context pointer | Final ordinary release-path call. |

The dynamic receiver-selected callback has no supplied opaque identity. Its observed arguments are the receiver and the address of a local pointer slot.

No dependency is established to preserve globals, headers, accounting balances, pointer validity, or candidate lifetime. Allocation, deallocation, locking, and ownership semantics remain documented_only.

## Acceptance examples

These are conditional deductions from static evidence, not executed tests. All examples require valid accessed storage and applicable arithmetic assumptions.

1. **First gate closed.** If the explicit character read at receiver offset `0x19` yields zero, the function returns without the 32-bit read at `0x18` and without any dependency call.

2. **Second gate closed.** If the first character is nonzero, the complete 32-bit word beginning at `0x18` is read. If conversion of that word to a character yields zero, the function returns without dependency calls. Readable storage must cover the whole word.

3. **Conditional byte interpretation.** Assume stable little-endian receiver storage and conversion retaining the low eight bits. Bytes `00 01 00 00` at offsets `0x18` through `0x1B` make the first gate nonzero, but the second read obtains `0x00000100`, whose converted character is zero. The operation stops. This conclusion depends on the stated byte-order and conversion assumptions.

4. **No candidate.** With both gates passing, F-8157a180d901 receives `12`, followed by F-2573f37ac5e3. A null result prevents discriminator reads, delivery, and release accounting.

5. **Inclusive boundaries.** With a zero discriminator, category `22` or `33` selects delivery. Category is stored before payload, and both stores precede callback resolution.

6. **Outside boundaries and override.** With a zero discriminator, category `21` or `34` selects release. With a nonzero discriminator, category `-1` instead selects delivery and is stored using its unchanged 32-bit representation.

7. **Rejected stable header.** On release, a stable zero header or stable header with bit zero set invokes F-a817d8d58c10 after preparation and context capture. The ordinary accounting sequence is not reached.

8. **Valid zero-size header.** A stable header of `2` passes validation and masks to zero. If dependencies preserve that header and execution reaches the updates, the function directly subtracts zero at each size-update location and performs both count decrements.

9. **Stable accounting balances.** Assume a header remaining `0x40`, valid and distinct accounting locations, representable arithmetic, and dependencies that return normally without otherwise modifying the relevant accounting locations. Then, relative to entry balances, the four initial size locations and the location at context offset `0x08` each decrease by `64`; the selected count and the fixed count each decrease by one. Without the dependency-preservation assumption, only the direct arithmetic operations—not those final balance changes—are established.

10. **Changed later size.** Suppose the first masked size is `64`, and F-09dcb55e8e0a changes the header so its later masked size is `128`. If subsequent execution and storage remain valid, the first four direct size subtractions use `64`; the final one uses `128`.

11. **Callback clears the slot.** If the callback clears the supplied pointer slot and returns normally, this function returns without inspecting the cleared slot or explicitly releasing the original candidate.

## Uncertainties

- **documented_only — Character conversion and byte order:** The second gate’s word-to-character conversion is explicit, but its complete out-of-range conversion behavior and the target byte order are not independently established.
- **documented_only — Ownership:** Acquisition, callback delivery, and the release-like calls do not establish ownership transfer or retention rights.
- **documented_only — Dependency mutation:** Dependencies may change global sources, headers, accounting locations, or memory lifetime.
- **documented_only — Synchronization:** No lock, atomicity, or concurrency-safety contract is established.
- **documented_only — Arithmetic boundaries:** Signed decrement overflow, out-of-range unsigned-to-signed storage, selector arithmetic, and address arithmetic lack a portable boundary contract.
- **documented_only — Index validity:** Unsigned 16-bit header indices and the unbounded selector result do not establish valid table extents.
- **documented_only — Error recovery:** Invalid pointers, invalid callbacks, dependency failures, and nonlocal exits have no established recovery behavior.
- **documented_only — Auxiliary input:** Its initial local copy is visible, but no explicit externally observable use of that value is established.
- **Inferred intent:** Acquisition, accounting, release, and dispatch terminology describe likely subsystem roles rather than proven callee semantics.

## Evidence

The supplied static representation establishes the distinct gate access widths, conditional second read, character conversion, dependency sequence, null-result handling, unsigned category test, delivery writes, indirect callback arguments, header checks, masked-size reads, accounting addresses, and release-path call order.

The supplied dependency map establishes opaque identities, not dependency behavior. Comments and the earlier draft do not independently establish runtime semantics.

No runtime tests have been performed. Acceptance examples are static deductions subject to their stated assumptions and documented unknowns.
