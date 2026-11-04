<!-- Trusted provenance: F-291cabdf1ebc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-291cabdf1ebc

## Purpose

F-291cabdf1ebc conditionally calls F-c8e9cf68c1b6 using metadata reached through a supplied object and calculations derived from a supplied signed position. It prepares a signed quantity and a pair of 32-bit words. If the dependency returns zero, it records the supplied position through a freshly loaded metadata chain and returns zero. Otherwise, it returns 3.

The interpretation of the metadata as a stream, buffer, file, or other resource is not established. “Object,” “descriptor,” “position,” and “address-like value” describe observable roles, not verified resource types or units.

The established behavior includes repeated metadata reads and a post-call pointer-chain reload. These must not be replaced in the contract by an assumption that all reads observe one stable descriptor.

## Interface

### Inputs

The supplied representation exposes two 32-bit arguments:

| Argument | Interpretation and use |
|---|---|
| Object address | Address-bearing value used to locate a metadata link at byte offset `0x1B8`. Passed unchanged as the first argument to F-c8e9cf68c1b6. |
| Position | Signed 32-bit value used in quantity selection, multiplied by four for address construction, and stored unchanged after dependency success. |

No visible validation checks the object address, position sign, position range, alignment, or accessibility of referenced storage.

The 32-bit representation of pointer-bearing fields is established. A complete machine-level pointer model is not.

### Return value

The explicit return values are 32-bit values:

- **0:** The dependency returned zero and the subsequent position store completed.
- **3:** The initial metadata link was zero, or the dependency returned a nonzero result.

The dependency’s particular nonzero result is not propagated.

Invalid memory accesses and arithmetic cases whose runtime semantics are unresolved are not represented by an additional explicit error result. No visible handler converts a failed memory access into 3.

### Referenced layout

Offsets are in bytes. Every listed field occupies 32 bits.

| Base | Offset | Role |
|---|---:|---|
| Object | `0x1B8` | Link to auxiliary metadata; zero selects the immediate failure path. |
| Auxiliary metadata | `0x10` | Link to a descriptor. |
| Auxiliary metadata | `0x48` | Low base word for address construction; read separately for the low calculation and carry calculation. |
| Auxiliary metadata | `0x4C` | High base word for address construction. |
| Descriptor | `0x00` | Signed endpoint used in quantity selection. |
| Descriptor | `0x04` | Position destination on dependency success. |
| Descriptor | `0x08` | Opaque value supplied as the dependency’s second argument. |

The descriptor used for the endpoint, the descriptor used for the opaque argument, and the descriptor used for the final store need not be identical if links change between accesses.

Normal completion requires each accessed location to be valid at the time it is accessed. On the success path, the final destination must be writable. Allocation extents, byte order, and alignment requirements beyond these accesses are **documented_only**.

## Behavior

### Initial gate

The first metadata access reads the object’s 32-bit link at offset `0x1B8`.

If that link is zero, the function returns 3 without calling F-c8e9cf68c1b6 and without performing an explicit write.

This check does not validate the object address itself. The link location must already be readable for the check to complete.

When the link is nonzero, its initially obtained value supplies the auxiliary-metadata base for the pre-call accesses. The object’s link is not reloaded for those accesses.

### Signed quantity selection

The function reads the descriptor link at auxiliary-metadata offset `0x10`, then reads the signed 32-bit endpoint at descriptor offset `0x00`.

The quantity is selected by a strict signed comparison:

- If position plus 16,384 is less than the endpoint, the quantity is 16,384.
- Otherwise, the quantity is endpoint minus position.

When both the threshold addition and the applicable subtraction are representable as signed 32-bit values, this selects the smaller of 16,384 and endpoint minus position.

Equality takes the subtraction branch. Thus, an endpoint exactly 16,384 greater than the position still produces quantity 16,384.

There is no lower-bound check or local rejection of negative positions:

- Position equal to endpoint produces quantity zero.
- Position greater than endpoint can produce a negative quantity.
- A negative position can produce a positive, zero, or negative quantity, depending on the endpoint.

The mathematical minimum description is conditional on non-overflowing signed arithmetic. Runtime treatment of overflow in the threshold addition or endpoint subtraction is **documented_only**. The contract does not replace those operations with saturation, wider arithmetic, or guaranteed wrapping.

### Two-word address-like value

The displacement expression multiplies the signed position by four. Its runtime behavior when that signed multiplication overflows is **documented_only**.

For a position whose multiplication by four is representable, the resulting displacement participates in 32-bit unsigned low-word addition. A negative product therefore contributes its corresponding unsigned 32-bit value. No separate high product word or sign-extension word is visibly added to the high base.

The low-word calculation has two stages:

1. Add the displacement to a read of the low base at auxiliary-metadata offset `0x48`, retaining the low 32 bits as an intermediate value.
2. Add 16 to that intermediate value, retaining the low 32 bits as the dependency’s low-word argument.

The high-word argument combines:

- The high base read at auxiliary-metadata offset `0x4C`.
- The carry from adding the displacement to a separate read of the low base at offset `0x48`.
- The carry from adding 16 to the previously calculated intermediate low word.

Each carry contributes either zero or one. The second carry is one exactly when the intermediate low word is at least `0xFFFFFFF0`.

The low-base read used to detect the first carry is distinct from the read used to form the intermediate low word. If those reads observe different values, the low word and high-word carry need not describe a coherent addition to one base value.

With stable base fields and defined displacement multiplication, the result is equivalent to adding the unsigned 32-bit displacement and 16 to the two-word base, propagating both low-word carries. The high-word result is retained as a 32-bit word; overflow beyond that word is not reported.

This equivalence does **not** establish a signed 64-bit displacement calculation. In particular, a negative product contributes its unsigned low word without a corresponding negative high-word adjustment.

### Call preparation and invocation

The function calls F-c8e9cf68c1b6 once on the non-null path, provided all preceding accesses and calculations complete.

The five arguments, in interface order, are:

1. The original object address.
2. The opaque 32-bit value at descriptor offset `0x08`.
3. The constructed low word.
4. The constructed high word.
5. The selected signed quantity.

The descriptor for the second argument is obtained by another read of the initially selected auxiliary metadata’s link at offset `0x10`. It is not necessarily the descriptor previously used to obtain the endpoint.

The endpoint read and quantity selection precede the intermediate low-word calculation. That intermediate calculation precedes call-argument evaluation and invocation. All argument-producing accesses must complete before the dependency is invoked.

The supplied representation does not establish a total order among the remaining argument-preparation reads. In particular, it does not justify ordering the high-base read, the repeated low-base read for carry detection, and the descriptor-chain reads for the second argument relative to one another.

### Result handling and successful-position update

If F-c8e9cf68c1b6 returns any nonzero result, F-291cabdf1ebc returns 3. It performs no explicit position store.

If the dependency returns zero, the function reloads the metadata chain:

1. Read the object’s auxiliary-metadata link at offset `0x1B8`.
2. Read the descriptor link at offset `0x10` from that newly reached metadata.
3. Store the original position as a 32-bit value at descriptor offset `0x04`.
4. Return zero.

Neither post-call link receives a null check. A zero dependency result therefore does not by itself guarantee a normal zero return.

The final store follows the chain present after the dependency returns. If the dependency changes the object’s metadata link or its descriptor link, the store follows the replacement chain rather than necessarily updating a pre-call descriptor.

## Side effects

### Direct effects

The only explicit memory write performed by this function is the successful-path 32-bit position store at descriptor offset `0x04`.

There is no such store when the initial metadata link is zero or when the dependency returns nonzero.

No allocation, deallocation, reference-count adjustment, resource close, rollback, or synchronization operation is visible in this function.

### Dependency-mediated effects

F-c8e9cf68c1b6 may have additional effects that are not established by the supplied material. The contract does not assume it preserves:

- Object or metadata links.
- Descriptor contents.
- Address-construction fields.
- Global state.
- Allocation lifetimes.
- Resource ownership.

A nonzero dependency result does not establish that the dependency had no effects. This function performs no visible rollback before returning 3.

### Ownership and access validity

No ownership transfer, retention, borrowing duration, or release obligation is established for any supplied or derived address.

The function does not visibly keep storage alive. A normally completing execution requires every accessed object to remain accessible at its respective access, including after the dependency returns zero.

No atomic snapshot or synchronization guarantee is visible. Repeated-read behavior is established, but actual concurrency semantics and behavior under conflicting accesses are **documented_only**.

## Dependencies

### F-c8e9cf68c1b6

This is the only visible direct function dependency.

At this call site it receives an object address, an opaque 32-bit descriptor value, two 32-bit address-like words, and a signed quantity. Its result is tested only for equality with zero.

The supplied material does not establish:

- The dependency’s operational purpose.
- The units or interpretation of its arguments.
- Whether it accepts zero or negative quantities.
- Its validation rules or error taxonomy.
- Its memory, global-state, or resource effects.
- Whether it retains addresses, transfers ownership, or releases storage.

Describing it as a read, write, transfer, mapping, or allocation operation would be inferred intent, not an established contract requirement.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless a case says otherwise, storage is valid at each access, pre-call metadata remains stable, all relevant signed arithmetic is representable, and the dependency returns normally.

### Missing metadata

If the object’s link at `0x1B8` is zero, the function returns 3. It does not call the dependency or write a position field.

### Quantity below the cap

With endpoint 20,000 and position 5,000, the selected quantity is 15,000.

If the dependency returns zero and the post-call chain is valid, the destination field at offset `0x04` receives 5,000 and the function returns zero.

### Quantity above the cap

With endpoint 50,000 and position 1,000, the selected quantity is 16,384.

The address-like calculation still uses position 1,000. Capping the quantity does not cap or replace the position.

### Exact cap boundary

With endpoint 17,384 and position 1,000, the threshold equals the endpoint. The strict comparison is false, and subtraction produces quantity 16,384.

### Zero and negative quantities

With endpoint 700 and position 700, the dependency receives quantity zero.

With endpoint 700 and position 701, it receives quantity −1. No local rejection occurs. Dependency acceptance remains unknown.

### Both low-word additions carry without signed multiplication overflow

Assume:

- Position is −1.
- Endpoint is 0.
- The stable low base is `0xFFFFFFF8`.
- The high base is 5.

Multiplying −1 by four produces −4, which is representable as a signed 32-bit value. Its unsigned 32-bit contribution is `0xFFFFFFFC`.

Adding that contribution to the low base produces intermediate low word `0xFFFFFFF4` and a first carry of one. Adding 16 produces final low word `0x00000004` and a second carry of one. The dependency therefore receives low word `0x00000004` and high word 7.

Quantity selection is also non-overflowing: position plus 16,384 is 16,383, which is not less than endpoint 0, so endpoint minus position produces quantity 1.

This case requires no assumption about wrapping signed multiplication.

### Dependency failure

If the dependency returns 9, the function returns 3, not 9. It does not perform the explicit post-call position store. Any dependency effects are not undone by this function.

### Replacement metadata after dependency success

If the dependency returns zero after installing a different valid metadata chain, the supplied position is stored through that replacement chain.

The pre-call descriptor is updated only if the reloaded chain ultimately identifies it.

If the post-call chain is null or invalid, the function has no visible guarded fallback. Normal completion is not guaranteed.

### Different pre-call descriptors

If the endpoint read reaches one descriptor but the later link read for the opaque argument reaches another, the quantity derives from the first descriptor and the dependency’s second argument derives from the second.

This describes the separate accesses; it does not establish that unsynchronized concurrent mutation is supported.

## Uncertainties

The following remain **documented_only**:

- Runtime treatment of signed overflow in threshold addition, endpoint subtraction, and multiplication by four.
- Full machine-level pointer interpretation and memory-fault behavior.
- Byte order, alignment guarantees, and allocation extents.
- Semantic units of the position, endpoint, quantity, and two-word value.
- Whether the two-word value represents an address, offset, or another encoding.
- Dependency behavior, accepted inputs, global effects, ownership rules, and lifetime effects.
- A total order among argument-preparation reads not ordered by the supplied representation.
- Concurrency guarantees and the semantics of mutation during accesses.

The inferred intent is a position-related operation with an upper quantity cap, followed by recording the position on success. That interpretation adds no requirements beyond the established accesses, calculations, call, result handling, and store.

## Evidence

Established static evidence is the supplied representation of F-291cabdf1ebc and the opaque dependency mapping to F-c8e9cf68c1b6.

That evidence supports the 32-bit fields, byte offsets, initial null-link gate, signed quantity selection, low-word calculations, separate carry-related low-base read, single dependency call, zero-versus-nonzero result handling, post-call chain reload, and successful-path position store.

The two-carry acceptance case uses position −1 so that multiplication by four remains within signed 32-bit range. Its quantity calculation is also non-overflowing.

No dependency implementation, authoritative resource schema, or runtime observations were supplied. No runtime tests have been performed.
