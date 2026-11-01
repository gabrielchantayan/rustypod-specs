<!-- Trusted provenance: F-d2bc4a2e390c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d2bc4a2e390c

## Purpose

F-d2bc4a2e390c reports whether numeric bit 15 is set in an unsigned integer obtained from an ambiently selected memory location. Bit numbering begins at zero at the least-significant position; the selected bit has numeric weight 32,768.

On normal completion, the result is exactly:

- **0** when the selected bit is clear.
- **1** when the selected bit is set.

Other bits do not affect the result. In particular, a nonzero observed value can produce zero, and a set selected bit produces one rather than its numeric weight.

**Established static evidence:** the result distinguishes these two classes of observed values.

**Inferred intent:** a Boolean-like status query over externally supplied storage.

**documented_only:** the application meaning of the selected bit. The evidence does not identify it as readiness, enablement, failure, acknowledgement, or any other named condition.

## Interface

### Inputs

The function has no explicit parameters. Callers cannot directly provide an address, bit index, or output destination.

Its effective inputs are:

- An ambient value used to select an address.
- The unsigned integer available at the selected location.
- The environmental definitions governing address arithmetic, address interpretation, and the integer representation.

The selection incorporates an additive displacement of four before the resulting location is accessed through an unsigned-integer type. The declaration of the ambient value is unavailable, so the displacement’s units are not established.

If the ambient value is an integer representing a byte address, the selection may identify an address four bytes later, subject to valid arithmetic and conversion. If it is a typed pointer, the displacement depends on the size of the pointed-to element and the permitted pointer bounds. These are conditional interpretations, not interchangeable guarantees.

The ambient value’s initialization, lifetime, storage class, and mutability are **documented_only**. Whether obtaining it entails an observable memory read is also unresolved.

### Output and widths

The function returns an unsigned integer. Every normally completed result is representable as zero or one. The observed integer and the return value have the same represented type.

The supplied material does not define that type’s exact width, size, alignment, or calling convention. The selected numeric bit requires at least 16 value bits, but does not establish a 16-bit or 32-bit access.

A normalized result does not establish a Boolean type or one-byte return storage. Values used in acceptance examples are conditional on their representability in the observed type.

### Preconditions and error boundary

The normal-result contract requires a valid selection and a valid access to the complete unsigned-integer object. Relevant requirements include:

- Permitted arithmetic for the ambient value’s actual type.
- A valid resulting address and any required address conversion.
- Sufficient readable storage with suitable alignment and lifetime.
- Permission to access that storage using the represented integer type.
- Any concurrency or special-storage conditions required by the environment.

No validation, separate error result, or recovery behavior is established.

Zero means that the selected bit was clear in the value supplying the result. It must not be interpreted as a documented indication of an invalid address, unavailable storage, or failed access.

## Behavior

### Selection and observation

The result is tied to the location selected from the ambient value using the displacement described above. There is no caller-supplied alternative and no established fallback location.

The target is observed as an unsigned integer, rather than as an independently selected byte containing the relevant bit. A numeric bit position alone does not establish a narrower memory-access width.

The static body contains one target-value observation. It contains no second target observation, consistency comparison, retry, or wait condition. The missing ambient declaration prevents a complete accounting of any additional access needed to obtain the address-selection value.

The necessary dependency order is that location selection determines the target observation, and the observed value determines the result. This does not establish a machine-instruction sequence or an ordering guarantee relative to unrelated memory activity.

### Result invariants

All valid observed values belong to one of two result classes:

| Observed-value property | Required normal result |
|---|---:|
| Numeric bit 15 is clear | 0 |
| Numeric bit 15 is set | 1 |

Within either class, changes to other representable bits leave the result unchanged. This applies to bits below the selected position and to any higher bits present in the type.

The function does not return the complete observed value, count set bits, or distinguish values within a result class.

No retained history is visible. There is no established accumulation, change detection, or comparison with an earlier invocation. The result describes the value supplying that invocation’s result; it does not promise that the storage remains unchanged afterward.

### Representation and layout

The selected bit is a numeric property, not a fixed byte offset within an object. Its physical placement depends on integer width, byte size, and byte order.

For example, if the observed integer occupies four eight-bit bytes, the selected bit lies in the most-significant position of the second byte in little-endian storage and of the third byte in big-endian storage. Neither layout is established as the actual target layout.

The displacement of four does not establish an enclosing structure, register map, neighboring field, allocation boundary, or object ownership relationship.

No universal address-wrapping rule is established. Integer arithmetic depends on width and signedness; pointer arithmetic depends on element size and valid bounds. Address conversion can impose further restrictions. The function provides no demonstrated overflow check or boundary check.

### Logical observation versus physical access

The available evidence does not establish:

- A fresh physical load on every invocation.
- Exactly one processor load or bus transaction.
- An indivisible observation during concurrent modification.
- A memory barrier or synchronization operation.
- Immediate visibility of another agent’s writes.
- Suitability for polling device-backed storage.

The single target observation in the static body is not a physical transaction-count guarantee. Missing type definitions and environmental rules must not be replaced with assumed volatile, atomic, or device-access semantics.

If the access preconditions fail, no normal zero-or-one outcome is specified. A fault, language-level invalid behavior, or another environmental consequence remains **documented_only**.

## Side effects

No explicit store is visible. The function does not explicitly modify the ambient selection value, write the observed integer, acknowledge the selected bit, or retain the result.

There are no visible allocation, release, ownership-transfer, or lifetime-extension operations. Accessing the storage does not establish ownership. The storage’s validity and lifetime must be supplied externally.

The absence of explicit stores does not prove observational purity. Reading device-backed or specially managed storage may change external state. Read-to-clear, consuming, and other read-triggered effects are **documented_only**.

For valid, stable ordinary memory with no special read effects, the function returns the selected bit’s state without changing that memory. This conditional guarantee does not exclude modifications by other execution agents.

No called function is present, so there is no callee whose preservation of globals or treatment of memory ownership must be assumed.

## Dependencies

There are no called-function dependency IDs to enumerate.

The environmental dependencies are:

| Dependency | Contribution to the contract | Unresolved properties |
|---|---|---|
| Ambient selection value | Determines the basis of the target location | Type, initialization, storage, lifetime, mutability |
| Arithmetic and address interpretation | Gives meaning to the displacement and resulting location | Units, widths, bounds, overflow, conversion rules |
| Unsigned-integer representation | Defines the observed value and returned type | Exact width, size, alignment, ABI |
| Backing storage | Supplies the target observation | Extent, lifetime, ownership, access permissions, read effects |
| External coordination, if required | Makes shared access valid | Atomicity, synchronization, visibility |

These dependencies do not establish a fixed address, a particular hardware platform, or harmless read semantics.

## Acceptance examples

These are conditional behavioral acceptance criteria derived from static evidence. **No runtime tests have been performed.**

Unless otherwise stated, each example assumes valid address selection, valid readable storage, a representable observed value, and normal completion.

### Selected-bit discrimination

| Observed value, decimal | Required result | Distinction demonstrated |
|---:|---:|---|
| 0 | 0 | No set bits |
| 1 | 0 | A lower set bit is insufficient |
| 16,384 | 0 | Bit 14 is not selected |
| 32,767 | 0 | All lower bits may be set |
| 32,768 | 1 | The selected bit alone is sufficient |
| 32,769 | 1 | A lower set bit does not change the result |
| 65,535 | 1 | Other set bits do not suppress the result |

These cases distinguish the contract from a whole-value nonzero test and from returning the selected bit’s unnormalized numeric weight.

If the observed type is independently established to have 32 unsigned value bits, these additional cases apply:

| Observed value, decimal | Required result |
|---:|---:|
| 65,536 | 0 |
| 2,147,483,648 | 0 |
| 4,294,934,527 | 0 |
| 4,294,967,295 | 1 |

The third value has every one of those 32 bits set except bit 15. These examples do not establish the actual access width.

### Location discrimination

If the ambient value is independently established to be an integer byte address, and all relevant arithmetic and conversion are valid, the result follows the integer beginning four bytes beyond that address.

If the ambient value is instead a typed pointer, the result follows the location selected by that pointer type’s displacement rules. It need not be four bytes later.

When candidate locations contain different selected-bit states, acceptance depends on the location actually selected under the established type rules, not on an assumed structure layout.

### Repeated observations

With stable selection and stable, valid ordinary-memory contents, repeated normally completed invocations return the same result.

For two valid observations:

- Values differing only in bit 15 require different results.
- Values differing only in other bits require equal results.

These criteria relate results to observed values. They do not require a new hardware transaction on each invocation or immediate detection of concurrent updates.

### Invalid and special storage

An inaccessible, expired, misaligned, or otherwise invalid target has no specified normal result. Zero is not documented error handling.

For storage whose reads change external state, acceptance requires an independently supplied storage contract. The absence of an explicit store does not establish that the invocation leaves external state unchanged.

## Uncertainties

The following remain **documented_only**:

- Exact integer width, storage size, alignment, and return ABI.
- Ambient selection type, initialization, storage, lifetime, and mutability.
- Displacement units, arithmetic width, overflow behavior, and address-conversion rules.
- Whether obtaining the ambient value requires an observable read.
- Backing-object extent, ownership, lifetime, and storage kind.
- Byte size, byte order, and physical placement of the selected bit.
- Physical access granularity and transaction count.
- Atomicity, synchronization, and visibility of concurrent updates.
- Read-triggered external effects.
- Application meaning of the selected bit.
- Consequences of violated preconditions.

The status-query interpretation is inferred intent, not an established application-level meaning. Familiar platform conventions do not resolve the missing declarations or environmental guarantees.

## Evidence

The supplied static body establishes a parameterless interface, ambient location selection incorporating a displacement of four, an unsigned-integer observation, and a normalized return determined by numeric bit 15. The observation and return use the same represented type.

It shows no explicit stores, function calls, allocation, release, validation, retry, synchronization, or recovery.

The result classes, normalization, and independence from other representable bits follow from the supplied operation. Exact byte displacement, integer width, machine transaction behavior, and storage-specific side effects do not.

The prior draft and feedback are not independent behavioral evidence. Unavailable declarations and environmental contracts remain unresolved rather than being supplied by convention.

No runtime tests, target-memory observations, or independent execution validation have been performed.
