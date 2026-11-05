<!-- Trusted provenance: F-46a3b712fbbd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-46a3b712fbbd

## Purpose

F-46a3b712fbbd conditionally offers a record to a primary receiver’s specialized operation, then uses a fallback receiver unless that operation returns a nonzero result.

The specialized operation is eligible only when three record words satisfy a fixed recognition predicate involving two external comparison words. A failed recognition predicate does not reject the record: it routes the record to the fallback path. A recognized record also takes the fallback path if the specialized operation returns zero.

This is an established description of the visible dispatch behavior. Interpreting the recognition predicate as a format identifier, or interpreting a nonzero result as “consumed,” is inferred intent rather than an established ownership or processing guarantee.

## Interface

### Inputs

The routine has two inputs:

- **Primary receiver:** an address whose first word supplies the base address of an indirect-operation table.
- **Record address:** a 32-bit unsigned value used as an address and forwarded unchanged to downstream operations.

The supplied representation uses 32-bit integer words and byte-addressed offsets. The behavior below describes that apparent 32-bit execution model; the underlying binary interface has not been independently verified.

The primary receiver is not checked for null. The record address is not checked for null, alignment, readability, or minimum extent.

### Record layout

The routine observes the following 32-bit unsigned words relative to the record address:

| Byte offset | Role | Condition for reading |
|---|---|---|
| +4 | Initial discriminator | Always |
| +8 | Full-width comparison field | Only if the discriminator equals `0x21` |
| +12 | Combined upper/lower recognition field | Only if the first two comparisons succeed |

No read of the word at offset +0 is visible in this routine.

Two external 32-bit words participate in recognition:

- **Comparison word A:** compared with the entire word at record offset +8.
- **Comparison word B:** compared with the upper 16 bits of the word at record offset +12.

Their values, initialization, storage ownership, and mutability are not established by the supplied evidence.

### Indirect-operation layout

The primary receiver’s operation table supplies these entries:

| Byte offset within table | Role |
|---|---|
| +`0xDC` | Specialized operation |
| +`0xCC` | Fallback-receiver lookup |

A non-null fallback receiver supplies its own operation table through its first word. That table’s entry at byte offset +`0x30` is the fallback operation.

These are byte offsets, not operation indices. Under a four-byte table-entry model, they correspond to indices 55, 51, and 12 respectively. The behavioral contract depends on the byte offsets, not on a particular source-language object model.

### Output

There is no defined return value.

Normal completion occurs through one of three routes:

1. The specialized operation returns nonzero.
2. The fallback-receiver lookup returns null.
3. The fallback operation returns.

No status is reported to the caller to distinguish those outcomes.

## Behavior

### Recognition

The routine first reads the 32-bit word at record offset +4 and compares it with `0x00000021`.

If this comparison fails, neither the word at +8 nor the word at +12 is read by the recognition logic. The specialized operation is skipped, and fallback processing begins.

If the discriminator matches, the routine reads the word at +8 and obtains comparison word A. The two full-width unsigned values must be equal. If they differ, the word at +12 is not read by the recognition logic, and fallback processing begins.

If both preceding comparisons succeed, the routine reads the word at +12. This single sampled word is used for both remaining checks:

- Its upper 16 bits, obtained by an unsigned right shift of 16, must equal comparison word B.
- Its lower 16 bits must equal `0x7A89`.

Both checks must succeed before the specialized operation is invoked.

The upper-half comparison is a full-width equality against a value in the range `0x00000000` through `0x0000FFFF`. Comparison word B is not visibly masked or truncated. Consequently, if comparison word B exceeds `0xFFFF`, recognition cannot succeed.

The low-half comparison does not depend on the value of the upper half beyond the requirement that the upper-half comparison also succeed. The routine does not perform a second read of the record word at +12 to obtain the low half.

All recognition predicates are evaluated before the specialized operation is called. The routine does not repeat recognition after that call.

### Specialized operation

When recognition succeeds, the routine resolves the specialized operation through the primary receiver’s operation-table entry at +`0xDC`.

It invokes that target with two visible arguments:

- The original primary receiver.
- The original record address.

The result is treated as a 32-bit integer and tested only for equality with zero.

- **Any nonzero result:** the routine returns immediately. The fallback-receiver lookup and fallback operation are not invoked.
- **Zero result:** the routine proceeds to fallback processing.

A negative result, if represented as a negative signed 32-bit integer, is still nonzero and therefore suppresses fallback. No particular positive success code is required.

There is no visible restoration of receiver state, record contents, external comparison words, or other state after the specialized operation. Any such changes made by that operation remain subject to the operation’s own unknown contract.

### Fallback processing

Fallback processing occurs if recognition fails at any stage or if the specialized operation returns zero.

The routine resolves the fallback-receiver lookup through the primary receiver’s operation-table entry at +`0xCC`. No explicit arguments are visible at this call site. In particular, the supplied evidence does not show the primary receiver being passed as an explicit argument to this lookup.

The lookup result is interpreted as a receiver address:

- **Null:** return immediately.
- **Non-null:** use that receiver’s first word as an operation-table address and resolve the entry at +`0x30`.

The fallback operation is invoked with:

- The receiver returned by the lookup.
- The original record address.

Its result, if it has one, is not used. After the fallback operation returns, this routine returns.

The fallback dispatch uses the returned receiver’s table, not the primary receiver’s table. Neither the returned receiver’s first word nor the selected operation target is checked for null or validity.

### Ordering and state visibility

The established order is:

1. Read the discriminator at record offset +4.
2. Conditionally obtain and compare the +8 field and comparison word A.
3. Conditionally obtain and compare the +12 field and comparison word B, and check the low half.
4. Conditionally invoke the specialized operation.
5. If fallback is needed, resolve and invoke the fallback-receiver lookup.
6. If its result is non-null, resolve and invoke the fallback operation.

The exact relative machine-level ordering of the two operands within an individual comparison is not established by the supplied representation. No synchronization or atomic snapshot across record fields and external comparison words is shown.

If the specialized operation runs and returns zero, fallback lookup is resolved afterward through the primary receiver. There is no visible saved operation-table snapshot that forces fallback to use the primary receiver’s pre-call table.

Accordingly, the specialized operation must not be assumed to preserve the primary receiver’s table, the record contents, or global state. Fallback dispatch and downstream record processing can observe changes made by that operation.

## Side effects

### Directly established effects

The routine performs memory reads and indirect calls. It contains no visible explicit memory stores, allocation, deallocation, reference-count adjustment, record copying, or record-address adjustment.

Its own visible reads include:

- Conditional record-field reads described above.
- Conditional reads of the two external comparison words.
- Receiver operation-table pointers and selected operation entries.

The two receiver-table paths are only accessed when their respective calls are reached.

### Delegated effects

The indirect operations may modify memory, change external state, retain arguments, allocate or release resources, fail, or not return. None of those behaviors is defined by this routine’s visible body.

In particular:

- Passing the record address does not establish ownership transfer.
- A nonzero specialized result does not prove that the record was freed, retained, or fully processed.
- A zero specialized result does not prove that the record or receiver remained unchanged.
- Returning a fallback receiver does not establish whether that receiver is borrowed, newly allocated, shared, or reference-counted.
- The absence of a visible release after fallback does not establish that no release is required elsewhere.

### Preconditions and failures

For normal completion, every memory access and indirect call actually reached must be valid in the execution environment.

A path may avoid later reads, but that does not make an invalid earlier address safe. For example, an unmatched discriminator avoids reads at +8 and +12, yet still requires a valid read at +4.

There is no visible error recovery for:

- Invalid record or receiver addresses.
- Invalid operation-table addresses.
- Null or invalid selected operation targets.
- A non-null but invalid fallback receiver.
- Failures, exceptions, or nonlocal exits originating inside dependencies.

No bounds checking or overflow checking accompanies record-address offset calculation. Behavior at address-space boundaries, including any wraparound or fault behavior, requires the actual machine and runtime rules and is not established here.

## Dependencies

The following opaque IDs are specification-local labels for unresolved indirect call targets. They are not recovered global identities, and they do not establish that the three targets are distinct functions.

| Opaque function ID | Resolution | Visible call contract |
|---|---|---|
| F-000000000001 | Primary receiver’s table, byte offset +`0xDC` | Receives primary receiver and record address; supplies a 32-bit result tested for zero |
| F-000000000002 | Primary receiver’s table, byte offset +`0xCC` | No explicit arguments shown; supplies a nullable receiver address |
| F-000000000003 | Returned receiver’s table, byte offset +`0x30` | Receives returned receiver and original record address; result is ignored |

No dependency contract is supplied for preservation of globals, receiver validity, record lifetime, memory ownership, reentrancy, or error handling.

The two external comparison words are additional data dependencies. Their actual values are unknown. No particular byte signature can therefore be specified beyond the comparisons stated above.

## Acceptance examples

These examples are conditional expectations derived from static evidence, not results of runtime tests. Each assumes valid reached memory accesses, returning dependencies, and no concurrent interference unless stated otherwise.

1. **Discriminator mismatch**

   The word at +4 is `0x20`.

   Expected: recognition reads neither +8 nor +12; F-000000000001 is not called. F-000000000002 is called. Its result determines whether F-000000000003 is called.

2. **Full-width comparison mismatch**

   The word at +4 is `0x21`, but the word at +8 differs from comparison word A.

   Expected: the word at +12 is not read by recognition. Processing goes directly to fallback lookup without invoking F-000000000001.

3. **Upper-half mismatch**

   The first two checks pass. The word at +12 is `0x12347A89`, while comparison word B is `0x1235`.

   Expected: recognition fails despite the matching low half. The specialized operation is skipped.

4. **Comparison word B exceeds 16 bits**

   The first two checks pass, and comparison word B is `0x00010000`.

   Expected: recognition fails for every possible 32-bit value at +12. Its unsigned upper 16 bits cannot equal `0x00010000`.

5. **Low-half mismatch**

   The first two checks pass, comparison word B is `0x1234`, and the word at +12 is `0x12347A88`.

   Expected: recognition fails. A matching upper half alone does not enable the specialized operation.

6. **Recognized record, nonzero specialized result**

   The first two checks pass, comparison word B is `0x1234`, and the word at +12 is `0x12347A89`. F-000000000001 returns `-1`.

   Expected: the routine returns without calling F-000000000002 or F-000000000003.

7. **Recognized record, zero specialized result**

   All recognition checks pass. F-000000000001 returns zero, and F-000000000002 returns a valid receiver.

   Expected call order: F-000000000001, F-000000000002, F-000000000003. The fallback operation receives that returned receiver and the unchanged original record address.

8. **No fallback receiver**

   Fallback processing is reached, and F-000000000002 returns null.

   Expected: immediate normal return without accessing a fallback receiver’s table or invoking F-000000000003.

9. **Specialized operation changes dispatch state**

   Recognition succeeds. F-000000000001 changes the primary receiver’s table to another valid table and returns zero.

   Expected: subsequent fallback lookup uses the primary receiver’s table as read after the specialized call, rather than a visibly cached pre-call table.

10. **Specialized operation changes record contents**

    Recognition succeeds. F-000000000001 modifies the record and returns zero; fallback lookup returns a valid receiver.

    Expected: recognition is not repeated. F-000000000003 receives the original address and may observe the modified contents.

## Uncertainties

The following remain **documented_only**:

- Exact calling convention, register usage, argument cleanup, and pointer representation.
- Whether the no-explicit-argument lookup relies on implicit machine state.
- Actual identities and full contracts of the indirect targets.
- Values, lifetime, mutability, and synchronization rules of the external comparison words.
- Record meaning, total size, byte order, and any format-level validity rules beyond the visible checks.
- Whether the receivers implement a particular object system.
- Ownership and lifetime rules for both receivers and the record.
- Validity of addresses after dependency calls.
- Runtime behavior for invalid accesses, invalid targets, concurrent mutation, or boundary-crossing address arithmetic.
- Whether any dependency can reenter this routine, raise an exception, or terminate execution.

“Specialized handling” and “fallback handling” are descriptive role names. They do not establish semantic success, error recovery, or resource consumption.

## Evidence

The supplied static representation establishes the conditional field reads, the unsigned shift and mask, the two external comparisons, the three indirect-operation offsets, the visible call arguments, the specialized result’s zero/nonzero branch, and the null fallback-receiver branch.

The representation supports a 32-bit word/address interpretation, but that interpretation has not been independently checked against executable behavior or platform documentation.

The final indirect dispatch is represented with a recovery warning. The visible representation describes an indirect call followed by return; exact low-level transfer mechanics are not independently established.

No implementation execution, runtime inspection, or runtime tests have been performed. Acceptance examples are derived obligations, not observed outcomes. No provenance hash is asserted.
