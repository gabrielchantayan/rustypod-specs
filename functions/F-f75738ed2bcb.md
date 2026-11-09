<!-- Trusted provenance: F-f75738ed2bcb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f75738ed2bcb

## Purpose

F-f75738ed2bcb selects an operation from a descriptor and an effective type value, then either:

- delegates the entire request to a descriptor-supplied callback;
- delegates a selected payload to one of two dependencies;
- derives a source address and a 32-bit count, optionally invokes an external transfer operation, and returns that count; or
- returns −1 for specific rejected representations.

The apparent intent is to extract or transfer data from several value representations. That interpretation is inferred. The established behavior is the dispatch, memory access, mutation, and return-value contract below; the semantic meanings of the type values are not established.

## Interface

The function has four arguments and a signed 32-bit result.

| Argument | Contract |
|---|---|
| Input slot | Address of a 32-bit word. Depending on the selected path, that word is an immediate value or an address. Some paths follow that address to additional words. |
| Destination | A 32-bit argument interpreted as an address when nonzero on transfer paths. Zero suppresses the direct transfer operation and causes either delegated conversion dependency to receive a null second argument. |
| Type slot | Address of a signed 32-bit type value. Depending on descriptor selection, the function reads this value, replaces it, or does not access it. |
| Descriptor | Address of a record containing at least the fields needed by the selected path. |

The supplied static representation uses 32-bit integers and address-bearing words. Layouts below describe that representation, not a portable native-width interface. Bytes at unused offsets have no established meaning.

### Descriptor layout

| Byte offset | Width | Use |
|---|---:|---|
| 0 | 1 byte | Selector, compared with 0 and 5 |
| 4 | 32 bits | Mode, compared with 1 and −4 |
| 16 | 32-bit address | Optional callback-provider address |
| 20 | signed 32 bits | Constraint used only for effective type 1 |

When the callback-provider address is nonzero, the function reads a callback address at byte offset 24 within the referenced provider record.

### Other accessed layouts

The first word of the input slot may reference a record:

| Byte offset | Width | Conditional use |
|---|---:|---|
| 0 | signed 32 bits | Replacement type when descriptor mode is −4 and selector is not 5 |
| 4 | signed 32 bits | Replacement type when descriptor selector is 5 |

Other branches interpret a payload address as follows:

| Effective type | Fields read from referenced object |
|---|---|
| 6 | Signed 32-bit count at offset 12; 32-bit source address at offset 16 |
| Unhandled type | 32-bit source address at offset 8, followed by signed 32-bit count at offset 0 |

These layouts are path-specific. They are not evidence that all referenced objects share one structure.

### Preconditions

There is no general pointer validation.

Required readable or writable storage depends on the selected path. In particular:

- The descriptor must permit the initial read at offset 16.
- A nonzero callback-provider address must permit reading its callback field.
- Without a usable callback, the descriptor selector must be readable.
- A type-slot replacement requires a writable 32-bit type slot.
- Following an address requires the referenced storage to be readable at the relevant offsets.
- A nonzero destination passed to a dependency must satisfy that dependency’s requirements.

An early callback or early rejection can avoid accesses otherwise required. Therefore, valid storage for every argument is not an unconditional prerequisite for every path.

## Behavior

### 1. Callback override

The function first reads the descriptor’s callback-provider address.

If that address is nonzero, it reads the callback address from provider offset 24. If both addresses are nonzero, it invokes the callback with exactly:

1. the original input-slot address;
2. the original destination argument.

The callback’s result becomes the function result without further local processing.

On this path, the function does not locally inspect the selector, mode, type slot, or payload. It does not perform the direct transfer operation. The callback may independently access or modify memory; those effects are unknown.

A null provider or null callback does not itself produce an error. Either condition selects the built-in behavior.

### 2. Select the effective type and payload slot

Without a usable callback, the function reads the descriptor’s one-byte selector.

The initial payload-slot address is the original input-slot address. Selection then follows these rules, in order:

1. **Selector 0 and mode 1:**  
   Read the effective type from the type slot. Do not first follow the input slot’s word as an address.

2. **Every other selector/mode combination:**  
   Read the original input slot’s word as an address. If that address is zero, return −1 immediately, without replacing the type slot.

3. **After that nonzero-address check, selector 5:**  
   Read the signed 32-bit word at offset 4 from the referenced record. Write it to the type slot and use it as the effective type. The payload-slot address remains the original input-slot address. Descriptor mode does not select the −4 behavior in this case.

4. **Otherwise, mode −4:**  
   Read the referenced record’s first signed 32-bit word. Write it to the type slot and use it as the effective type. Change the payload-slot address to the referenced record’s word at offset 4.

5. **Otherwise:**  
   Read the effective type from the existing type slot. The payload-slot address remains the original input-slot address.

The distinction between following the original input word and changing the payload-slot address is important. Only the mode −4 branch changes the payload slot to the referenced record’s second word. The selector-5 branch obtains a type from the referenced record but keeps the original input slot as the payload slot.

Any type-slot replacement occurs before type-specific processing. A subsequent −1 result does not roll back that replacement.

### 3. Dispatch by effective type

Type comparisons use signed 32-bit values. Recognized values are 1, 2, 3, 5, 6, 10, 258, and 266.

#### Effective type 1

Read the signed 32-bit word in the selected payload slot as an immediate value.

- If the value is −1, return −1.
- If the value is zero, read descriptor offset 20. Return −1 if that field is zero; otherwise accept.
- If the value is neither zero nor −1, read descriptor offset 20. Return −1 if that field is strictly positive; otherwise accept.

An accepted value produces a count of 1 and a source address referring to the function’s local 32-bit copy of the immediate value. Processing then reaches the common transfer stage.

The transfer count is **1**, not the four-byte width of the stored value. If the external dependency treats the count as bytes, only one byte is requested. Which byte represents the intended value depends on target byte order, which the supplied material does not establish.

Negative immediate values other than −1 are not categorically rejected. They use the same constraint rule as positive nonzero values.

#### Effective types 2, 10, 258, and 266

Read the selected payload slot’s word and invoke F-a194c5e9ba8c with that word as the first argument.

The second argument is:

- null when the original destination is zero;
- otherwise, the address of a local 32-bit word initially containing the destination argument.

Return the dependency’s result directly. Do not execute the common transfer stage.

The second argument is therefore not the destination address itself. Its interpretation and any writes through it belong to the dependency contract.

#### Effective type 3

Behave as for the preceding delegation group, but invoke F-be39f2326e96.

Return that dependency’s result directly, with no subsequent common transfer stage.

#### Effective type 5

Select a null source address and count zero.

Reach the common transfer stage, which skips the external transfer because the count is zero. Return zero.

Once this type has been selected, the type-specific branch does not read the payload. Earlier selection may nevertheless have required a nonzero referenced record and may have updated the type slot.

#### Effective type 6

Read the selected payload slot’s word as an object address.

From that object:

1. read the signed 32-bit count at byte offset 12;
2. read the source address at byte offset 16.

Reach the common transfer stage.

No local check rejects a negative count or null source address.

#### Every other effective type

Read the selected payload slot’s word as an object address.

The source expression first obtains the source address from byte offset 8. A subsequent expression obtains the signed 32-bit count from byte offset 0 through the payload slot again.

Reach the common transfer stage.

This fallback includes zero, negative types, and positive values not explicitly recognized. There is no unsupported-type error.

### 4. Common transfer and return

For paths reaching this stage:

- If the original destination is nonzero **and** the selected count is nonzero, invoke F-unknown-01 with the destination, source address, and count, in that order.
- Otherwise, do not invoke F-unknown-01.
- Return the selected signed 32-bit count.

The result of F-unknown-01 is ignored.

The local gate is “nonzero count,” not “positive count.” A negative count therefore reaches F-unknown-01 when the destination is nonzero. Any conversion of that argument to an unsigned size, or any resulting large transfer, belongs to the unknown dependency boundary; it must not be assumed safe.

With a zero destination, these paths still perform their preceding selection and payload reads. Zero destination suppresses the transfer call, not all memory access.

## Side effects

### Established local effects

The only direct caller-visible write established in the body is replacement of the type-slot word:

- from referenced-record offset 4 when selector is 5; or
- from referenced-record offset 0 when selector is not 5 and mode is −4.

This write precedes dispatch and can remain visible even when later processing returns −1.

For delegated conversion paths, the function supplies an address to a local destination-containing word when the destination is nonzero. A dependency may modify that local word, but the function returns immediately afterward and does not locally propagate its resulting value elsewhere.

### Dependency effects

The callback, the two mapped dependencies, and F-unknown-01 may have additional side effects. Their bodies and contracts are not supplied.

The function makes no explicit allocation or release call identifiable from the supplied body. This does not establish that dependencies allocate nothing, release nothing, or preserve global state.

The local value used as the source for type 1, and the local destination-containing word used for delegation, have only invocation-local lifetimes. Safe retention of either address after return is not established.

### Access ordering and aliasing

Established high-level ordering is:

1. callback-provider lookup;
2. callback lookup, if applicable;
3. either callback invocation or built-in selector processing;
4. optional input-address check;
5. optional type-slot replacement;
6. type-specific payload reads or delegation;
7. optional direct transfer.

The function does not stage a snapshot of all input storage. Aliasing the descriptor, type slot, input slot, or referenced records can therefore affect later reads. In particular, a type-slot write occurs before payload dispatch and may alter storage read during that dispatch.

No synchronization or atomicity guarantee is visible. Exact instruction-level ordering beyond the supplied expression and statement structure is documented_only.

## Dependencies

| Opaque ID | Invocation contract established here |
|---|---|
| F-a194c5e9ba8c | Called for effective types 2, 10, 258, and 266 with a selected payload word and either null or an address of a local destination-containing word. Its result is returned directly. |
| F-be39f2326e96 | Called for effective type 3 with the same argument arrangement. Its result is returned directly. |
| F-unknown-01 | Locally assigned opaque label for the unmapped external dependency. Receives destination, source address, and nonzero count when destination is also nonzero. Its result is ignored. |
| F-unknown-02 | Locally assigned opaque label for the dynamically selected callback. Receives the original input-slot address and original destination. Its result is returned directly. |

The two locally assigned labels identify call boundaries only. They do not assert known implementations or identities.

## Acceptance examples

These examples are conditional static expectations, not executed tests. Assume required storage is accessible, relevant fields do not unexpectedly alias, and dependencies return normally.

1. **Callback bypasses built-in decoding.**  
   A nonzero provider contains a nonzero callback address. F-unknown-02 receives the original input-slot address and destination. If it returns 37, the function returns 37 without locally reading the type slot.

2. **Missing callback falls through.**  
   A nonzero provider contains a null callback. Built-in selection proceeds; the null callback is not reported as an error.

3. **Direct type selection avoids the preliminary address check.**  
   Selector is 0, mode is 1, the type slot contains 5, and the input slot contains zero. The function returns zero without treating that zero input word as a rejected record address.

4. **Other selection rejects a null referenced record first.**  
   Selector is 5 and the input slot contains zero. The function returns −1 before replacing the type slot, even if its previous value was 5.

5. **Selector 5 takes precedence over mode −4.**  
   Selector is 5 and mode is −4. The replacement type comes from referenced-record offset 4, not offset 0. The original input slot remains the payload slot.

6. **Type replacement survives rejection.**  
   Selector is not 5, mode is −4, and the referenced record begins with type 1 followed by immediate −1. The type slot becomes 1, then the function returns −1.

7. **Type-1 zero constraint.**  
   Effective type is 1 and the selected immediate value is zero. Descriptor offset 20 equal to zero produces −1. Any nonzero signed value there permits count 1.

8. **Type-1 nonzero constraint.**  
   Effective type is 1 and the immediate value is −2. A positive constraint produces −1; a zero or negative constraint permits count 1.

9. **Delegation receives destination indirection.**  
   Effective type is 3 and destination is nonzero. F-be39f2326e96 receives an address to a local word containing that destination, rather than the destination directly. Its return value is propagated unchanged.

10. **Zero destination remains a read-only sizing-like path only conditionally.**  
    Effective type is 6 and destination is zero. The function still reads the object’s count and source fields, skips F-unknown-01, and returns the count. Dependency-free execution does not imply absence of memory reads.

11. **Negative count is not locally rejected.**  
    Effective type is 6, count is −3, and destination is nonzero. F-unknown-01 receives that 32-bit count representation. If it returns normally, the function returns −3.

12. **Unknown type uses the fallback layout.**  
    Effective type is 7. The function uses the source address at object offset 8 and count at offset 0 rather than returning an unsupported-type error.

## Uncertainties

The following are documented_only:

- Semantic names for the selector, mode, type codes, and type-1 constraint.
- Target byte order and the intended meaning of a one-unit transfer from a 32-bit local value.
- The exact external contract of F-unknown-01, including count units, signedness, overlap handling, and failure behavior.
- Internal effects, ownership rules, and accepted payloads of F-a194c5e9ba8c and F-be39f2326e96.
- Callback calling-convention details beyond the displayed two arguments and propagated result.
- Whether any dependency preserves globals, retains argument addresses, allocates, or releases memory.
- Behavior for invalid addresses, insufficient storage, concurrent mutation, or a dependency that does not return normally.
- Whether negative return values other than the explicit local −1 cases represent errors, counts, or dependency-specific results.
- Exact machine-level access behavior where the supplied static representation reloads an address-bearing word.

## Evidence

The supplied static function body establishes the field offsets, comparison values, branch precedence, explicit type-slot writes, dependency arguments, local rejection conditions, and return paths described above.

“Extraction,” “transfer,” and “sizing-like” are inferred intent, not independently established API semantics. The supplied body does not establish the behavior of any dependency.

No implementation access, runtime execution, or tests were performed. The acceptance examples are consequences of static analysis under their stated preconditions, not observed results.
