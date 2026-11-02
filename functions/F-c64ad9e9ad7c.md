<!-- Trusted provenance: F-c64ad9e9ad7c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c64ad9e9ad7c

## Purpose

F-c64ad9e9ad7c conditionally copies a descriptor’s leading word and a derived number of four-byte payload elements into caller-provided storage.

The transfer decision depends on a multiplication expression whose result is assigned to a signed 32-bit local value. The multiplier’s declaration is absent. Its type, width, the multiplication’s arithmetic conversions, and any conversion into the signed local are therefore unresolved. The function must not be described as necessarily performing signed 32-bit multiplication.

Once the signed local value has been obtained, the visible body conditionally negates negative values and applies a signed upper-bound comparison against 28. When those negations are representable, values with magnitudes above 28 are rejected. Accepted calls with distinct descriptor addresses copy the leading word and then transfer payload elements in descending index order. Accepted calls with identical descriptor addresses return without descriptor or payload writes.

The function performs a shallow storage transfer. It does not explicitly copy a payload-address field, allocate or release storage, or establish ownership.

This contract is based on static evidence only. Numerical examples involving multiplication are conditional on the unresolved operand types and conversions producing the stated signed local value.

## Interface

### Arguments and descriptor layout

The function receives two descriptor addresses:

1. Destination descriptor address.
2. Source descriptor address.

The supplied representation supports the following descriptor prefix:

| Byte offset | Width | Observed use |
|---|---|---|
| 0 | 4 bytes | Signed leading word, read for the multiplication expression and copied on an accepted nonidentity call. |
| 4 | 4 bytes | Address-bearing word used to locate payload elements. |

The second field is accessed through the represented signed integer-word layout and then used in address expressions. It is not explicitly copied as a pointer object. Its interpretation as an address, including signedness effects and conversion to a usable address, depends on the execution environment.

Payload elements occupy four bytes each. The transfer preserves those four-byte values without establishing whether they represent numbers, pointers, or other data.

The descriptor layout does not establish endianness, alignment requirements, or the behavior of overflowing address calculations.

### Global input

A global object supplies the other operand of the multiplication expression.

Its declaration is not supplied. Consequently, the following are not established:

- Its declared type, width, or signedness.
- Whether its value is integral or belongs to another arithmetic type.
- The usual arithmetic conversions applied to the multiplication.
- The type and range of the multiplication result.
- The conversion, if any, when that result is assigned to the signed 32-bit local.
- Behavior when the multiplication or assignment conversion cannot produce an ordinary representable result.

The expression contains one visible use of this global operand. No later use appears during validation or copying. This does not establish the number or width of physical memory transactions.

There is no explicit assignment to the global object. Aliased writes could nevertheless affect its storage.

### Result

The function returns a four-byte status value:

| Value | Established meaning |
|---|---|
| 1 | The signed value used for validation compares greater than 28. |
| 0 | Validation passes and the identity or transfer path completes. |

These statuses are not transfer counts. A return of 0 may involve no writes, only a leading-word write, or a leading-word write followed by payload transfers.

No separate status is provided for invalid addresses, inadequate capacity, alignment faults, arithmetic problems, conversion problems, or interrupted transfers. Where those conditions prevent ordinary execution, a return value is not established.

### Path-dependent storage requirements

For ordinary completion:

- The source leading word must be readable for the initial expression.
- The global operand must be accessible according to its unresolved type and evaluation requirements.
- An accepted nonidentity call additionally requires a readable source leading word for the later copy and a writable destination leading word.
- A positive-length nonidentity transfer requires readable address-bearing fields and valid storage for every reached payload read and write.
- Rejection and accepted descriptor identity do not access payload-address fields or payload elements.
- An accepted zero-length nonidentity call does not access payload-address fields or payload elements.

The body contains no destination-capacity check. The caller must supply sufficient accessible storage for the addresses actually reached, including any redirection caused by aliasing.

## Behavior

### Initial value formation

The function evaluates a multiplication involving the source leading word and the global operand, assigning the result to a signed 32-bit local.

The relative evaluation order of those operands is not established by the multiplication expression.

The assigned signed value is the basis for the subsequent validation and transfer count. It must be distinguished from both:

- The mathematical product of two abstract numbers.
- The multiplication expression’s result before assignment conversion.

Those values need not coincide under the unresolved type and conversion rules. For example, the global operand might cause the source word to be converted before multiplication, and assignment might apply a further conversion afterward.

Accordingly, the unconditional contract begins with the signed local value that the expression and assignment actually produce. Describing that value as an exact mathematical product requires additional assumptions.

### Validation

The function makes a separate signed validation value from the assigned local. If the assigned local is negative, the validation value is its negation. Otherwise, the validation value remains unchanged.

It then compares the validation value with 28 using a signed comparison. A value greater than 28 produces status 1 immediately.

For an ordinarily obtained signed local whose negation is representable:

- Values from −28 through 28 pass.
- Values below −28 or above 28 are rejected.
- Equal positive and negative magnitudes receive the same validation result.

These rules do not depend on identifying the multiplier’s type once the assigned signed local is known.

The minimum signed 32-bit value, −2,147,483,648, cannot be negated to a positive signed 32-bit value. The supplied body does not establish an ordinary outcome for that negation. It therefore does not justify promising rejection, acceptance, or a bounded transfer for that case.

Validation occurs before descriptor-address comparison. Descriptor identity does not bypass the multiplication expression, assignment conversion, conditional negation, or upper-bound check.

An ordinarily rejected call performs no explicit writes and does not access either payload-address field or any payload element.

### Descriptor identity

After validation passes, the function compares the two descriptor addresses.

If they are equal, it returns 0 without:

- Reading the source leading word a second time for copying.
- Writing the leading word.
- Reading either payload-address field.
- Reading or writing payload elements.

Identity is address identity, not equality of descriptor contents or payload addresses.

Thus, a positive accepted count does not require usable payload storage when both arguments identify the same descriptor.

### Count preparation and leading-word copy

For an accepted call with distinct descriptor addresses, the original signed local is conditionally negated if negative. When that operation is representable, its resulting value is the nonnegative transfer count.

The function then reads the source leading word again and writes that value to the destination leading word.

The copied word is not the normalized count. It is the value obtained from the second source-word read. The transfer count remains derived from the earlier expression and assignment; it is not recalculated after the copy.

The leading-word write occurs before all payload transfers, including any payload-address-field reads used by those transfers. It also occurs when the count is zero.

If storage changes between the two source-word reads, the copied value can differ from the value used in the initial expression. The body supplies no locking, consistency check, or retry. This observation does not establish valid or deterministic behavior for unsynchronized concurrent modification.

### Payload transfer

For an ordinary accepted count \(n\) and distinct descriptor addresses, payload indices are processed in descending order:

\[
n-1,\ n-2,\ \ldots,\ 1,\ 0.
\]

Each index selects a four-byte element beginning at byte offset \(4k\) from the corresponding payload base.

Each iteration obtains the source element’s four-byte value and stores it at the same destination index. The source value must be obtained before its corresponding destination store. The expression does not establish a complete ordering between all address-field reads involved in that iteration.

The payload bases appear as descriptor-field accesses in the transfer expression. They are not visibly captured as an independent, preserved pair before the loop. If earlier writes change those fields, later accesses cannot be assumed to use their original values.

There is no complete temporary payload snapshot.

For a count of zero, the transfer body is never entered. The loop condition still decrements its internal count from zero to −1, but no payload-address read or element access follows from that decrement.

For an ordinary accepted count of 28:

- The first transferred element is index 27, at byte offset 108.
- The last transferred element is index 0.
- Each payload’s accessed element range spans 112 bytes, assuming a stable base and valid address arithmetic.

Normal completion returns 0.

### Overlap and aliasing

Descending transfer order affects results. The function is not a general overlap-safe copy.

With stable descriptor fields, valid storage, and no descriptor or global aliasing:

- A destination payload beginning one element above the source preserves the original sequence during descending transfer.
- A destination beginning one element below the source can overwrite a source element before its later read.

Distinct descriptors that name the same payload do not take the identity shortcut. Their leading-word copy and positive-length payload accesses still occur.

Aliasing with descriptor or global storage can have additional effects:

- The leading-word store can change a source field needed later.
- Payload writes can modify descriptor leading words or address-bearing fields.
- Modified address-bearing fields can redirect later transfers.
- Writes can affect the global operand’s storage even though the body contains no explicit global assignment.

No field-preservation or overlap-detection mechanism is visible. Exact outcomes for such layouts depend on reached addresses and any expression ordering not fixed by the supplied body.

## Side effects

For ordinary arithmetic and stable, nonaliasing descriptor fields:

| Path | Explicit writes | Payload-address-field access |
|---|---|---|
| Validation rejects | None | None |
| Validation passes; descriptor addresses equal | None | None |
| Validation passes; distinct addresses; zero count | Destination leading word | None |
| Validation passes; distinct addresses; positive count | Destination leading word, then descending four-byte elements | Used during transfers |

Only the leading descriptor word is explicitly assigned. The destination’s address-bearing field remains unchanged when no other reached write aliases it.

The body contains no direct function call, allocation, release, reference-count operation, I/O operation, or callback. This describes the visible body, not any hidden runtime mechanism that an implementation might use.

No ownership transfer is established. Existing storage-sharing and lifetime obligations remain external.

No synchronization, transactional boundary, or rollback appears. If execution fails after writes have begun, the function contains no explicit restoration mechanism. Exact fault delivery and externally observable partial state remain unresolved.

## Dependencies

No called-function dependency appears in the supplied body. There are therefore no opaque callee IDs to list and no callee behavior to assume.

Direct dependencies are:

- The source descriptor’s leading word.
- The undeclared global arithmetic operand.
- The environment’s multiplication and assignment-conversion rules.
- Signed 32-bit comparison, negation, and decrement behavior.
- Descriptor-address equality.
- Address-bearing descriptor fields on positive-length nonidentity transfers.
- Storage reached through the computed addresses.
- Address arithmetic, alignment, and memory-access rules.

No conclusion about global preservation or memory ownership is imported from an unknown callee.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Unless stated otherwise, they require stable inputs, valid reached storage, representable subsequent arithmetic, and no aliasing between payloads, descriptors, or global storage.

### Positive assigned value

If the initial expression and assignment produce the signed local value 6, and the descriptors are distinct:

- Validation passes.
- The current source leading word is copied to the destination.
- Six elements are transferred in order 5, 4, 3, 2, 1, 0.
- The function returns 0.

For a source leading word of 3 and a global operand interpreted as 2, this example applies only if the unresolved multiplication and assignment rules actually produce signed local value 6.

### Negative assigned value

If the initial expression and assignment produce −8, the descriptors are distinct, and the source leading word remains −4:

- Validation uses magnitude 8 and passes.
- The destination leading word becomes −4, not 8.
- Eight elements are transferred in descending order.
- The function returns 0.

Interpreting this as “−4 multiplied by 2” requires types and conversions that produce the stated assigned value.

### Exact bound

If the assigned signed value is −28 and the descriptors are distinct:

- Validation passes.
- The count becomes 28.
- The leading word is copied.
- Exactly 28 elements are transferred, beginning at index 27.
- The payload extent is 112 bytes per stable base.
- The function returns 0.

A source word of 7 and a global operand interpreted as −4 instantiate this example only when evaluation and assignment produce −28.

### Rejection before identity

If the assigned signed value is 29:

- The function returns 1.
- It performs no descriptor or payload writes.
- It reads no payload-address field.
- The result is the same whether descriptor addresses are equal or distinct.

This example does not independently establish that a source word of 29 and a global operand interpreted as 1 produce that assigned value.

### Accepted identity

If the assigned signed value is 12 and both arguments identify the same descriptor:

- The function returns 0.
- It performs no writes.
- It does not read payload-address fields or payload elements.

The initial source-word and global-operand evaluation still occurs.

### Zero count

If the expression and assignment produce zero and the descriptors are distinct:

- The current source leading word is copied.
- No payload-address field is read.
- No payload element is transferred.
- The function returns 0.

A global operand described numerically as zero is not, by itself, sufficient to establish this path without the relevant type, evaluation, and conversion assumptions.

### Direction-sensitive overlap

For count 3 and an original source sequence \([A,B,C]\), with stable address-bearing fields:

- A destination beginning one element above the source receives \([A,B,C]\).
- A destination beginning one element below the source receives \([C,C,C]\).

The latter result follows because descending stores overwrite source elements before their later reads. Both examples require valid shifted storage and no descriptor or global aliasing.

### Unresolved arithmetic and conversion

If the multiplication or assignment cannot be shown to produce a particular signed local value, no status or transfer count is specified by a numerical-product example.

If the assigned value is −2,147,483,648, no ordinary validation or transfer outcome is asserted because its required negation is not representable.

## Uncertainties

The following remain **documented_only**:

- **Global declaration:** type, width, signedness, qualifiers, initialization, and permitted values.
- **Multiplication semantics:** operand conversions, expression result type, overflow behavior, and any nonintegral arithmetic behavior.
- **Assignment conversion:** the mapping into the signed 32-bit local, including out-of-range or otherwise exceptional cases.
- **Signed negation:** execution behavior for the minimum signed 32-bit value.
- **Address interpretation:** conversion of address-bearing words, address arithmetic overflow, and resulting validity.
- **Machine details:** endianness, alignment, and physical access granularity.
- **Exceptional accesses:** fault delivery, recovery, and precise externally visible partial state.
- **Concurrent modification:** legality and results when relevant storage changes during execution.
- **Complex aliasing:** exact results when writes alter fields used later, especially where expression evaluation order is unresolved.
- **Domain meaning and ownership:** what the leading word represents and which external component owns or releases storage.

The likely intent is a small, scaled payload copy. That is an inference, not evidence that every possible input produces a safely bounded operation.

## Evidence

Established static evidence supports:

- Multiplication of the source leading word by an undeclared global operand.
- Assignment of that expression into a signed 32-bit local.
- Signed conditional negations and a signed comparison against 28.
- Rejection status 1 and normal completion status 0.
- Validation before descriptor-address comparison.
- No payload access on ordinary rejection or accepted identity.
- A second source leading-word read on the nonidentity copy path.
- A leading-word store before payload transfers.
- Four-byte transfers in descending index order.
- No payload-address-field access for an ordinary zero count.
- No visible capacity check, direct call, allocation, release, synchronization, or rollback.

The missing global declaration does **not** establish a signed multiplier, signed 32-bit multiplication, or an identity conversion into the signed local. All examples that relate numeric operands to an assigned value are conditional on those unresolved semantics.

No runtime tests have been performed. The acceptance examples describe conditional expected behavior, not observed executions.
