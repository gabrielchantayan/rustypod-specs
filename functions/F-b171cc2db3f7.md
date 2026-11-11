<!-- Trusted provenance: F-b171cc2db3f7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b171cc2db3f7

## Purpose

F-b171cc2db3f7 dispatches six request codes using a caller-supplied object address and selector. It returns a 32-bit handled indicator and writes a 32-bit result on specified paths.

“Handled” does not imply that a result was written. Some paths return 1 without directly writing the result slot. All paths execute a shared-state entry sequence before request dispatch, including requests that ultimately return 0.

**Established static evidence:** the supplied body establishes the branch conditions, access widths, object offsets, explicit stores, dependency-call ordering, and return values described below.

**Inferred intent:** the function appears to expose properties, derive shared values, return embedded-object addresses, and construct a three-word descriptor. These interpretations do not establish object types, text encodings, ownership, or dependency semantics.

## Interface

| Input | Representation | Use |
|---|---|---|
| Object base | 32-bit address-bearing value | Base for field accesses and embedded-object addresses. |
| Selector | 32-bit value | Compared against opaque global tokens. |
| Request code | 32-bit value | Recognized values are `0x9401` through `0x9406`. |
| Result slot | Address of a 32-bit word | Receives a result only on the specified paths. |

The return value is a 32-bit word:

- **0:** unrecognized request or explicit rejection.
- **1:** handled request, possibly without a direct result-slot write.

No explicit checks validate addresses, object extent, alignment, dispatch tables, or callable targets. Every reached memory access and call requires a valid corresponding location or target. The result slot need only be writable on paths that directly write it, although dependency behavior and aliasing may impose additional requirements.

### Object-relative layout

Offsets are in bytes.

| Offset | Established use |
|---|---|
| `0x3c` | 32-bit field, interpreted as signed for descriptor validation and as raw bits for another result. |
| `0x40` | 32-bit field passed to a dependency and used in descriptor arithmetic. |
| `0x44` | Embedded-object address passed to dependencies. |
| `0x80` | First descriptor word. |
| `0x84` | Second descriptor word. |
| `0x88` | Third descriptor word. |
| `0x8c` | Embedded address returned by one request. |
| `0x144` | Embedded address passed to an update dependency and subsequently returned. |
| `0x1fc` | Embedded address passed to an object-access dependency. |

The descriptor occupies 12 bytes. The source does not establish the complete object layout.

### Selector tokens and shared state

The following names identify global-token roles, not necessarily distinct numeric values:

- **A:** zero-result selector for `0x9401`.
- **B:** indirect-result selector for `0x9401`.
- **C:** shared selector for `0x9402` and `0x9403`.
- **D:** shared selector for `0x9404` and `0x9405`.
- **E:** descriptor selector for `0x9406`.
- **F:** raw-field selector for `0x9406`.

A global address identifies a shared working destination. Other globals supply a flags base, initialization word, dependency arguments, and an opaque data address.

C is captured once after the common entry calls. Subsequent uses of C in this invocation use that captured value, even if a dependency changes its backing global. Other token comparisons use the reads described below.

## Behavior

### Common entry sequence

The following occurs before dispatch:

1. Capture the global shared-destination address.
2. Read the 32-bit flags word at offset `0x0c` from a separate global base.
3. If flags bit 0 is clear:
   - Call F-b255b26284e6 with an address eight bytes before the shared destination obtained for this call.
   - Immediately after that call returns, read and save the global supplying the **third argument** of F-804adc29bbfb.
   - Only after saving that argument, test whether the predicate returned nonzero.
4. If the predicate returned nonzero:
   - Read the global initialization word and store it at the captured destination.
   - Store zero at captured-destination offset 4.
   - Read the global supplying the **second argument** of F-804adc29bbfb.
   - Call F-804adc29bbfb with the captured destination, that second argument, and the previously saved third argument.
   - Call F-d08d3ca17b81 with an address eight bytes before the captured destination.
5. Call F-438680ffebfe with the then-current global shared destination and an opaque global data address.
6. Read and capture C, then dispatch on the request code.

The third-argument snapshot is read even when F-b255b26284e6 returns zero. It is not read through this sequence when flags bit 0 is set.

The snapshot timing is part of the contract. If either initialization store aliases the third-argument global, F-804adc29bbfb still receives its earlier saved value. Its second argument is read after both stores and therefore can reflect their effects.

The captured destination is also distinct from later reloads of the destination global. A dependency may change that global; initialization and F-d08d3ca17b81 still use the captured address, while subsequent operations explicitly using the current global may use a different address.

### Request `0x9401`: zero or indirect result

Compare the selector with A first.

- If equal, write zero to the result slot and return 1.
- Otherwise, compare it with B. If unequal, return 0 without a direct result-slot write.
- If equal to B:
  - Call F-7d964f24a044 with object base plus `0x1fc`.
  - Read the returned object’s first word as a dispatch-table address.
  - Invoke the callable entry at table byte offset `0x280`.
  - Write the returned 32-bit value to the result slot and return 1.

No returned-object or target validation is visible. The indirect call’s complete implicit interface is unknown.

If A and B coincide, A takes precedence.

### Request `0x9402`: derive a shared result

Call F-b7abcf60c990 with object base plus `0x44` before checking the selector against captured C.

If the dependency returns zero, return 1 without a direct result-slot write. If it returns nonzero but the selector differs from C, also return 1 without a direct result-slot write.

Otherwise:

- Call F-5fb92c16ae07 with an eight-byte temporary destination and object base plus `0x44`.
- Compare the captured shared-destination address with the temporary’s address.
- If unequal, call F-151ab4bfe553 with the then-current shared destination and the temporary’s second 32-bit word.
- Call F-289ec4311c64 with the temporary’s address.
- Call F-44362605ab79 with the then-current shared destination.
- Write its 32-bit result to the result slot and return 1.

Address equality skips only F-151ab4bfe553. It does not skip temporary processing or final extraction.

### Request `0x9403`: derive and transform a shared result

Call F-b7abcf60c990 with object base plus `0x44`.

- A zero return produces return value 1 without a direct result-slot write.
- A nonzero return with a selector unequal to captured C produces return value 0.
- A nonzero return with a matching selector continues as follows.

Construct two eight-byte temporaries in this order:

- Call F-9d0ecc5f6adf; pass its result, captured C, and a first opaque global argument to F-5710a8628cba. Pass that result to F-7afe554bcf62 with the first temporary destination.
- Call F-9d0ecc5f6adf again; pass its new result, captured C, and a second opaque global argument to F-5710a8628cba. Pass that result to F-7afe554bcf62 with the second temporary destination.

Then:

- Call F-4db1a5daf3db with a third eight-byte temporary destination and object base plus `0x44`.
- Compare the captured shared destination with the third temporary’s address. If unequal, call F-151ab4bfe553 with the then-current shared destination and that temporary’s second word.
- Call F-289ec4311c64 on the third temporary.
- Call F-44362605ab79 on the second temporary, saving its result.
- Call F-44362605ab79 on the first temporary, saving its result.
- Call F-1111096c956d with the then-current shared destination, the first temporary’s extracted value, and the second temporary’s extracted value, in that argument order.
- Call F-44362605ab79 on the then-current shared destination.
- Write its result to the result slot.
- Call F-289ec4311c64 on the second temporary, then on the first.
- Return 1.

Extraction order and argument order differ: the second temporary is extracted first but supplies the final argument. The result-slot store occurs before the final two temporary-processing calls.

### Request `0x9404`: conditional embedded address

Compare the selector with D. If unequal, return 0.

Otherwise, call F-52bea7bf3f16 with the object base. A zero result returns 0 without a direct result-slot write. A nonzero result writes object base plus `0x8c` to the result slot and returns 1.

The result is the address, not the word stored at that address.

### Request `0x9405`: update and return an embedded address

Compare the selector with D. If unequal, return 0.

Otherwise:

- Call F-87be4aa684d1.
- Read the returned object’s first word as a dispatch-table address and invoke the entry at table byte offset `0x1a4`.
- If the indirect call returns nonzero, return 0.
- Read the 32-bit field at object offset `0x40`.
- Call F-14e01b2d6d30 with the object base and that field value. If it returns zero, return 0.
- Call F-9d0ecc5f6adf.
- Pass its result and two opaque global arguments to F-5710a8628cba.
- Call F-4225f441f48b with object base plus `0x144`, the preceding result, zero, and zero.
- Write object base plus `0x144` to the result slot and return 1.

No status from F-4225f441f48b is checked.

### Request `0x9406`: descriptor or raw field

Compare the selector with E first.

For a match, the source establishes this evaluation and access order:

1. Read the signed 32-bit field at `0x40`.
2. Evaluate its 32-bit increment immediately, **before testing the −1 sentinel or reading `0x3c`**.
3. Determine whether the original field equals −1, then reread global token E.
4. If the original field is not −1, read the signed 32-bit field at `0x3c`.
5. Reject with return value 0 if the original field was −1, or if the field read at `0x3c` is zero or negative.
6. On acceptance:
   - Store the captured `0x3c` value at `0x88`.
   - Evaluate the increment of the captured `0x40` value again and store its 32-bit result at `0x80`.
   - Store 1 at `0x84`.
   - Store object base plus `0x80` in the result slot.
   - Return 1.

The reread of E occurs even on the −1 path; it does not prevent sentinel rejection. The −1 path does not read `0x3c`.

The early increment must not be moved to the successful-store path. In particular, it is evaluated when the original `0x40` value is the largest signed 32-bit integer even if a later nonpositive `0x3c` value would reject the descriptor. Target-specific consequences of signed overflow remain unresolved.

Apart from −1, no nonnegative-position check or comparison between the two fields is visible.

If the selector differs from E, compare it with F. A match reads the raw 32-bit word at `0x3c`, writes it to the result slot, and returns 1. No positivity restriction applies. Any other selector returns 0.

If E and F coincide, the descriptor branch takes precedence.

### Other request codes

Return 0 after the common entry sequence, without a direct result-slot write.

## Side effects

Direct stores are limited to:

- Two conditional initialization stores at the captured shared destination.
- Result-slot stores on the specified paths.
- Three descriptor stores on successful `0x9406`/E execution.

Store ordering is significant. In particular, descriptor writes occur at `0x88`, then `0x80`, then `0x84`, followed by the result-slot write. The successful-path increment for `0x80` occurs after the `0x88` store.

Aliasing can make these stores affect globals or other described storage. Saved addresses, argument snapshots, captured fields, and dependency results retain their already obtained values; later explicit memory reads may observe intervening writes.

No dependency contract establishes preservation of globals, object fields, temporary contents, result storage, or referents of previously returned values. “No direct result-slot write” therefore does not guarantee unchanged result memory.

Eight-byte temporaries are local storage passed to dependencies. Their use does not establish allocation, reference counting, destruction, or ownership transfer. No explicit allocation, rollback, synchronization, or exception handling is visible. Faults or non-returning dependencies may leave earlier effects in place.

## Dependencies

Roles below describe call-site use only.

| Opaque ID | Observed use |
|---|---|
| F-b255b26284e6 | Conditional initialization predicate. |
| F-804adc29bbfb | Initialization-associated call using a saved third argument. |
| F-d08d3ca17b81 | Receives captured destination minus eight bytes. |
| F-438680ffebfe | Unconditional pre-dispatch operation. |
| F-7d964f24a044 | Supplies object for indirect entry `0x280`. |
| F-b7abcf60c990 | Tests the embedded object at `0x44`. |
| F-5fb92c16ae07 | Populates the `0x9402` temporary. |
| F-151ab4bfe553 | Receives current shared destination and a temporary’s second word. |
| F-289ec4311c64 | Processes temporaries at specified points. |
| F-44362605ab79 | Produces a 32-bit value from an object-like argument. |
| F-9d0ecc5f6adf | Supplies an input for subsequent combination calls. |
| F-5710a8628cba | Receives that input and two additional arguments. |
| F-7afe554bcf62 | Populates the first two `0x9403` temporaries. |
| F-4db1a5daf3db | Populates the third `0x9403` temporary. |
| F-1111096c956d | Receives shared destination and two extracted values. |
| F-52bea7bf3f16 | Gates the `0x8c` address result. |
| F-87be4aa684d1 | Supplies object for indirect entry `0x1a4`. |
| F-14e01b2d6d30 | Gates the `0x9405` update. |
| F-4225f441f48b | Processes the embedded destination at `0x144`. |

The indirect targets have no supplied opaque IDs. Their identities and complete calling conventions remain documented_only.

## Acceptance examples

These are conditional static criteria, not executed tests. Unless stated otherwise, assume valid memory, returning dependencies, and no unmentioned mutation or aliasing.

1. **Unknown request:** returns 0 but still performs the common entry sequence.
2. **Zero initialization predicate:** when flags bit 0 is clear and F-b255b26284e6 returns zero, the third-argument global is nevertheless read. The initialization stores and F-804adc29bbfb call are skipped.
3. **Initialization snapshot under aliasing:** suppose the third-argument global contains a value when the predicate returns, and the subsequent initialization stores overwrite its storage. F-804adc29bbfb receives the earlier value as its third argument. Its second argument comes from the read after both stores.
4. **Explicit zero:** `0x9401` with selector A writes zero and returns 1.
5. **Handled without output:** `0x9402` with a zero embedded-object predicate returns 1 and preserves a sentinel result word under the no-aliasing and no-mutation assumptions.
6. **Selector asymmetry:** with a nonzero embedded-object predicate and selector unequal to captured C, `0x9402` returns 1 while `0x9403` returns 0; neither directly writes the result slot.
7. **Descriptor success:** `0x9406`/E with `0x40` equal to 4 and `0x3c` equal to 9 evaluates the increment before validation, then stores 9, 5, and 1 at `0x88`, `0x80`, and `0x84`, respectively. It returns the address at `0x80` and return value 1.
8. **Sentinel rejection:** a `0x40` value of −1 still undergoes the early increment and E reread, but causes return 0 without reading `0x3c` or directly writing the descriptor.
9. **Other negative position:** values −2 at `0x40` and 9 at `0x3c` pass validation and store the 32-bit representation of −1 at `0x80`.
10. **Overflow before rejection:** with the largest signed 32-bit value at `0x40` and zero at `0x3c`, increment evaluation precedes rejection. If that evaluation continues normally under target semantics, the branch returns 0 without descriptor stores; the source alone does not resolve overflow consequences.
11. **Raw result:** selector F, distinct from E, returns the raw `0x3c` word even when its signed interpretation is negative.
12. **Final processing order:** successful `0x9403` writes the result before calling F-289ec4311c64 on the second and first temporaries.

## Uncertainties

- **documented_only:** numeric global values, token equality relationships, and opaque argument contents.
- **documented_only:** concrete object types, encodings, temporary representations, result validity periods, and ownership or release obligations.
- **documented_only:** dependency side effects, reentrancy, concurrency, exceptions, and retention of temporary addresses.
- **documented_only:** complete indirect-call interfaces.
- **documented_only:** target-specific signed-overflow behavior for both increment evaluations, including the early evaluation on paths subsequently rejected. No explicit overflow-rejection branch is established.
- **Inferred intent only:** initialization, lookup, transformation, and lifetime-management interpretations. They do not authorize extra validation or memory-management behavior.

## Evidence

This contract uses the supplied static function body and opaque dependency mapping. The body establishes the third-argument read immediately after the initialization predicate, including its zero-return path, and the second-argument read after the initialization stores.

It also establishes an increment evaluation before descriptor rejection checks and a separate increment evaluation for the successful descriptor store. Neither evaluation has been relocated past validation in this contract.

No runtime tests have been performed. Acceptance examples are conditional consequences of static evidence, not observed execution results.
