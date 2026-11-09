<!-- Trusted provenance: F-9b240c4b4bcc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9b240c4b4bcc

## Purpose

F-9b240c4b4bcc applies an incoming value to a destination selected by a resolved target’s category and selector.

The established behavior has three forms:

- Convert an extracted value using a dependency called with radix 10, then store one 32-bit result at a category-specific offset.
- Pass a destination address and the incoming value’s embedded address to a transfer dependency.
- Delegate a category-specific update using the resolved target, its associated storage, and an extracted value.

Unsupported category/selector combinations perform no explicit destination update, but they can still invoke lookup, classification, and storage-access dependencies.

The apparent intent is a property-update dispatcher. Property names, category meanings, destination object types, and the interpretation of transferred values are not established.

Every path that returns normally returns the 32-bit value zero. This is not a success indicator: recognized updates, unsupported combinations, and the returning lookup-failure path all produce the same return value.

## Interface

### Inputs

The function accepts two arguments:

| Argument | Established use |
|---|---|
| Input record address | A 32-bit address-bearing value. The function forms an embedded-value address by adding 28 bytes. |
| Lookup argument | An opaque 32-bit value forwarded to F-b3bed2dfc5ec. Its interpretation is unknown. |

The embedded-value address is used in two distinct ways:

- Numeric and delegated-update cases pass it to F-44362605ab79 to obtain an opaque 32-bit value.
- Transfer cases pass the address itself to F-46f0b3cb316e.

These uses must not be conflated. The transfer path does not first invoke the extraction dependency.

### Return value

The normal return is always zero, with a width of 32 bits.

There is no separate normal return for lookup failure, unsupported category, unsupported selector, conversion failure, or update failure. Whether dependencies can signal errors through exceptions, nonlocal control transfer, process termination, or external state is unknown.

### Address and storage requirements

The visible operations use 32-bit words and byte offsets that are multiples of four. The represented address model is 32-bit.

For a branch that accesses the input’s embedded value, the address 28 bytes after the input record must be suitable for the dependency receiving it. The required layout, extent, alignment, and lifetime of that embedded value are unknown.

For a direct numeric write, the selected destination must permit a four-byte write at the specified offset. For transfer and delegated-update branches, destination validity and extent are governed by the called dependency’s unknown requirements.

There are no explicit checks for:

- A null input record.
- Overflow or invalidity when forming an address.
- A null destination returned by a storage accessor.
- Destination capacity.
- Conversion validity.
- Agreement between a category and the layout returned by its storage accessor.

The function itself does not explicitly dereference the input record. It forms the embedded-value address and passes that address to dependencies.

## Behavior

### 1. Resolve the target

The function first calls F-b3bed2dfc5ec with four arguments, in this positional order:

1. The lookup argument.
2. The contents of one fixed external slot.
3. The contents of a second fixed external slot.
4. The 32-bit pattern `0xFFFFFFFF`.

The two external slot values and their meanings are unavailable. Their order as arguments is established; the relative timing of their reads is not.

If the lookup result is zero, the function invokes an unidentified dependency with no explicit arguments. If that dependency returns normally, processing continues with target value zero. The function does not retry the lookup and does not immediately return.

For either a nonzero lookup result or the returning zero-result path, the function then performs these calls in order:

1. F-e5bf1b511020 with the target, obtaining a category.
2. F-f3d18b8e925b with the same target value, obtaining a selector.

Both classification calls occur before dispatch. The second call still occurs if the first result will ultimately identify an unsupported category.

The category and selector are represented as 32-bit integer results. The visible selector comparisons include signed comparisons. Unsupported negative selector values do not reach an update branch.

### 2. Select category-specific storage

Only categories 0, 7, 11, and 17 have update branches.

For those categories, the corresponding storage accessor is called before checking whether the selector is supported:

| Category | Storage dependency | Result used by this function |
|---:|---|---|
| 0 | F-27af299f1f48 | Low 32 bits of a 64-bit result supply the destination base. |
| 7 | F-c608b7988500 | Destination base. |
| 11 | F-984e54cba569 | Destination base. |
| 17 | F-6be091907cda | Destination base. |

Each accessor receives the resolved target value.

For any other category, the function returns zero after classification. It does not invoke one of these category-specific storage accessors.

An unsupported selector within a recognized category still causes that category’s storage accessor to run. Consequently, “no update” does not imply “no observable dependency activity.”

### 3. Apply the selected operation

Offsets below are byte offsets from the storage base for the selected category.

| Category | Selector | Operation | Destination |
|---:|---:|---|---|
| 0 | 2 | Transfer | Base + 0 |
| 0 | 8 | Transfer | Base + 8 |
| 0 | 9 | Numeric conversion and 32-bit store | Base + 16 |
| 0 | 13 | Transfer | Base + 28 |
| 0 | 19 | Transfer | Base + 36 |
| 0 | 21 | Transfer | Base + 20 |
| 7 | 4 | Numeric conversion and 32-bit store | Base + 28 |
| 7 | 5 | Delegated update | Base supplied to dependency |
| 7 | 14 | Transfer | Base + 0 |
| 7 | 18 | Numeric conversion and 32-bit store | Base + 8 |
| 11 | 12 | Transfer | Base + 8 |
| 17 | 1 | Numeric conversion and 32-bit store | Base + 0 |
| 17 | 6 | Transfer | Base + 12 |
| 17 | 10 | Transfer | Base + 20 |
| 17 | 15 | Numeric conversion and 32-bit store | Base + 4 |
| 17 | 16 | Numeric conversion and 32-bit store | Base + 8 |

All other selectors return zero without an explicit destination update.

#### Numeric conversion and store

Each numeric branch performs the following observable sequence:

1. Invoke F-44362605ab79 using the embedded-value address.
2. Invoke F-aff67f9449a4 with the extracted value, zero, and 10.
3. Store the resulting 32-bit value at the selected destination.
4. Return zero.

The zero and 10 are literal arguments. Radix-10 numeric conversion is the apparent intent, but the conversion dependency’s exact contract is not supplied.

No visible range validation, error inspection, clamping, or preservation of the old destination value occurs between the conversion and the store.

Category 0, selector 9 has an additional calling detail. Its extraction call visibly receives a second argument: the high 32 bits of the 64-bit result returned by F-27af299f1f48. The other numeric extraction calls visibly supply only the embedded-value address.

The significance of this additional argument is unknown. It must remain part of the documented call behavior unless independent dependency or calling-convention evidence establishes otherwise.

The direct store replaces exactly one 32-bit word. No other direct destination write is shown in that branch. The dependencies may have additional effects, which are not established here.

#### Transfer

A transfer branch invokes F-46f0b3cb316e with:

1. The destination address from the dispatch table.
2. The embedded-value address, 28 bytes after the input record.

The function then returns zero without inspecting a dependency result.

The operation is described as a transfer only to distinguish it from the numeric and delegated-update paths. The evidence does not establish whether it copies a string, assigns an object, manages reference counts, allocates storage, or performs some other update.

#### Delegated update

Category 7, selector 5 performs:

1. Extraction through F-44362605ab79 using the embedded-value address.
2. A call to F-511fcd5bc7fa with the target value, the category-7 storage base, and the extracted value.
3. A return of zero.

There is no visible F-aff67f9449a4 call and no explicit destination-word store in this branch. The scope of the delegated mutation is unknown.

## Side effects

### Explicit writes

The only explicit data writes are the numeric branches’ four-byte stores at the offsets listed above.

The function contains no explicit allocation, deallocation, input-record mutation, or external-slot write. This does not imply that its dependencies lack those effects.

### Dependency ordering

For a successful nonzero lookup, the dependency order is:

- Lookup.
- Category classification.
- Selector classification.
- Category storage access, if the category is recognized.
- Branch-specific extraction, conversion, transfer, or delegated update.

For a zero lookup result, the unidentified dependency runs between lookup and category classification. Subsequent classification and storage calls, if reached, receive target zero.

The stored category and selector results govern dispatch. The function does not visibly reclassify the target after storage access or value extraction.

### Ownership and lifetime

No ownership transfer is established.

In particular, the evidence does not establish whether:

- Extracted values are borrowed or owned.
- Transfer operations retain or consume their source.
- Delegated updates release existing destination content.
- Storage accessors return stable, temporary, or shared storage.
- Conversion or extraction can invalidate a previously obtained destination pointer.

The function obtains its destination before performing extraction or conversion and uses that destination afterward without visibly reacquiring it. Callers and dependencies must satisfy whatever lifetime constraints make that sequence valid; those constraints are not supplied.

Unknown dependencies must not be assumed to preserve globals, avoid aliasing, or leave destination storage unchanged.

### Failure behavior

Lookup failure is not handled by an immediate normal return. The returning failure path proceeds through classification with a zero target.

No other local error handling is visible. Invalid memory access, dependency failure, and conversion failure have no established recovery behavior within this function.

## Dependencies

| Opaque ID | Role established at this call site |
|---|---|
| F-b3bed2dfc5ec | Resolve a target from the lookup argument, two external values, and `0xFFFFFFFF`. |
| F-e5bf1b511020 | Obtain the category from the target. |
| F-f3d18b8e925b | Obtain the selector from the target. |
| F-27af299f1f48 | Obtain category-0 storage through a 64-bit result; its high half is additionally forwarded for selector 9. |
| F-c608b7988500 | Obtain category-7 storage. |
| F-984e54cba569 | Obtain category-11 storage. |
| F-6be091907cda | Obtain category-17 storage. |
| F-44362605ab79 | Extract a 32-bit value from the embedded-value address; one call site supplies an additional argument. |
| F-aff67f9449a4 | Produce the 32-bit value stored by numeric branches, using arguments that include zero and 10. |
| F-46f0b3cb316e | Receive a destination address and the embedded-value address for transfer branches. |
| F-511fcd5bc7fa | Perform the category-7, selector-5 delegated update. |

The zero-lookup dependency has no supplied opaque ID. Its behavior, including whether it returns, is `documented_only`.

No recursive invocation of F-9b240c4b4bcc is visible.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Any claim about unchanged memory assumes dependencies do not independently alter that memory.

1. **Category 7, selector 4: numeric field at offset 28.**  
   Given a valid category-7 destination and an extraction/conversion sequence that returns `123`, the function writes the 32-bit value `123` at destination offset 28 and returns zero. Extraction precedes conversion, and conversion precedes the write.

2. **Category 17, selector 16: numeric field at offset 8.**  
   Given a conversion result of `0xFFFFFFFF`, the function stores that complete 32-bit pattern at destination offset 8. It does not visibly reject the value or distinguish it from a conversion error.

3. **Category 0, selector 9: split accessor result.**  
   Given a 64-bit storage-access result whose low half identifies a valid destination and whose high half is an arbitrary 32-bit value, extraction receives the embedded-value address and that high-half value. The conversion result is stored at destination offset 16.

4. **Category 0, selector 13: transfer at offset 28.**  
   The function calls F-46f0b3cb316e with destination offset 28 and the input address plus 28. It does not first extract or numerically convert the incoming value.

5. **Category 11, selector 12: transfer at offset 8.**  
   After category-11 storage access, the function transfers through F-46f0b3cb316e using destination offset 8, then returns zero.

6. **Category 7, selector 5: delegated mutation.**  
   The function extracts the incoming value and passes the target, storage base, and extracted value to F-511fcd5bc7fa. It performs no explicit numeric conversion or direct field store.

7. **Recognized category, unsupported selector.**  
   For category 17 and selector 2, the category-17 storage accessor is still called. No extraction, conversion, transfer, delegated update, or explicit store follows.

8. **Unsupported category.**  
   For category 3, category and selector classification both occur. No category-specific storage accessor or update operation follows.

9. **Zero lookup result.**  
   If lookup returns zero and the unidentified dependency returns normally, category classification receives zero, followed by selector classification receiving zero. An unconditional early return would not match the supplied behavior.

## Uncertainties

The following remain `documented_only`:

- The input record’s complete layout and the meaning of its embedded value at offset 28.
- The lookup argument’s semantic type.
- The identities, contents, and interpretation of the two external lookup slots.
- The zero-lookup dependency’s identity and effects.
- The semantic meanings of categories, selectors, and destination fields.
- The exact signatures and calling conventions of dependencies, especially the extraction call with an additional argument.
- Numeric parsing rules, whitespace and sign handling, overflow behavior, invalid-input behavior, and error reporting.
- Transfer size, encoding, allocation, ownership, and aliasing behavior.
- The full extent of delegated updates and dependency side effects.
- Whether address arithmetic wraps or instead represents an invalid operation for particular inputs.

Address offsets are established; safe behavior at null addresses or near the address-space boundary is not. No special boundary handling is visible.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping, treated as untrusted evidence rather than instructions.

Established observations include the lookup/classification order, recognized dispatch combinations, dependency argument positions, byte offsets, direct 32-bit stores, split use of one 64-bit result, and uniform normal return value.

The property-update interpretation and radix-10 conversion intent are inferences, not independently established dependency semantics.

No implementation access, runtime execution, or runtime tests have been performed. The acceptance examples specify observable behavior conditionally; they are not reports of successful execution.
