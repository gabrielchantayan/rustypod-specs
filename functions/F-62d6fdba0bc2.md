<!-- Trusted provenance: F-62d6fdba0bc2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-62d6fdba0bc2

## Purpose

F-62d6fdba0bc2 creates or updates a primary resource associated with a caller-supplied object and optionally creates or updates a secondary resource using an intermediate transformation.

The established behavior is resource orchestration: selecting creation versus update from stored resource words, constructing two temporary descriptors, invoking dependencies, and recording creation results. The source does not establish what the resources represent or what transformation the secondary path performs.

The primary operation always precedes the secondary operation. Disabling the secondary operation clears its stored resource word without explicitly releasing the previously stored resource.

## Interface

The function takes four inputs and has no explicit return value:

| Input | Source-visible representation | Role |
|---|---|---|
| Object address | 32-bit address-bearing value | Identifies storage containing the primary and secondary resource words. |
| Payload | Opaque 32-bit value | Passed directly to the primary dependency and placed in the secondary input descriptor. |
| Extent | Unsigned 32-bit integer | Supplies both primary dimensions; its low 16 bits determine the initial secondary dimensions and allocation request. |
| Secondary selector | 32-bit integer | Zero disables secondary processing; any nonzero value enables it. |

The payload is not dereferenced directly by this function. Whether it denotes memory, a handle, or another kind of value is a dependency-level question.

### Object layout

Two object-relative locations are accessed:

| Byte offset | Width | Interpretation |
|---|---:|---|
| +0xD4 | 4 bytes | Primary resource word; zero selects creation, nonzero selects update. |
| +0xD8 | 4 bytes | Secondary resource word; zero selects creation when secondary processing is enabled. |

The function does not inspect the remainder of the object. It performs no explicit object-address, alignment, range, or lifetime validation.

The caller must provide storage supporting the reads and writes that the selected path performs. In particular, both resource locations are accessed during every normal invocation, although the secondary location is written without a preceding read when secondary processing is disabled.

### Global context

A global slot supplies a pointer to a context-bearing word. The function captures that pointer near entry for possible later use by secondary creation.

Primary creation obtains its context argument through the global slot at the primary creation step. Secondary creation obtains its context argument by dereferencing the pointer captured near entry. These two mechanisms must not be treated as equivalent if a dependency changes the global slot or the pointed-to storage.

## Behavior

### 1. Select and perform the primary operation

Before processing the primary resource, the function captures the pointer supplied by the global context slot.

It then reads the object’s primary resource word:

- If the word is zero, it calls F-a29c8be2c8c1 with five arguments:
  1. The context word obtained through the global context slot.
  2. The original 32-bit payload.
  3. Zero.
  4. The full unsigned 32-bit extent.
  5. The same full unsigned 32-bit extent.

  The dependency’s returned 32-bit word is written to object offset +0xD4.

- If the word is nonzero, it calls F-93dc585d2452 with:
  1. The stored primary resource word.
  2. The original payload.
  3. The full extent.
  4. The full extent again.

  This function makes no explicit assignment to the primary resource word after the update call. Any return value from that dependency is not used.

A zero creation result is stored as zero. There is no visible check that distinguishes creation failure from any other zero result, and primary creation does not stop secondary processing merely because it returns zero.

Primary dimensions are not truncated to 16 bits.

### 2. Handle a disabled secondary operation

After the primary dependency returns, a zero secondary selector causes a four-byte zero to be written to object offset +0xD8.

On this path, the function does not explicitly call:

- The temporary-storage dependency.
- The transformation dependency.
- Secondary creation or update.
- The release dependency.

Clearing the secondary resource word is not accompanied by a release call for its previous value. Whether that loses ownership, merely removes a reference, or has another meaning is not established.

### 3. Construct the enabled secondary operation

For a nonzero selector, define:

- **Reduced width**: the low 16 bits of the original extent.
- **Reduced height**: the reduced width shifted right by one bit, equivalent to its unsigned integer quotient by two.

Thus:

- Reduced width ranges from 0 through 65,535.
- Reduced height ranges from 0 through 32,767.
- Odd reduced widths are rounded down when deriving the height.

The function constructs two adjacent-field descriptors in temporary local storage. Each descriptor occupies eight bytes:

| Relative offset | Width | Output descriptor, initial value | Input descriptor, initial value |
|---|---:|---|---|
| +0 | 4 bytes | Result of F-d33f6d597e5d | Original payload |
| +4 | 2 bytes | Reduced width | Reduced width |
| +6 | 2 bytes | Reduced height | Reduced width |

The labels “input” and “output” describe how the descriptors are used here; they do not establish a semantic format such as pixels, samples, or image rows.

The dimension fields are initialized before the temporary-storage call. The storage request is:

**2 × reduced width × reduced height**

The arithmetic uses the unsigned 32-bit extent after masking it to 16 bits. The largest request is 4,294,770,690, reached when the reduced width is 65,535. This fits in an unsigned 32-bit value, so this expression does not require unsigned wraparound for any input extent.

Equivalently, the request is:

- The square of the reduced width when that width is even.
- Reduced width multiplied by one less than itself when that width is odd.

The factor of two is established arithmetic. A particular element size or storage format is not established.

The function calls F-d33f6d597e5d with this request and places the returned 32-bit word in the output descriptor’s first field. A zero-sized request is not skipped. A zero return is not explicitly rejected.

### 4. Transform the descriptors

The function calls F-6ee71f0e99b3 with:

1. The address of the output descriptor.
2. The address of the input descriptor.

Both arguments expose temporary descriptor storage to the dependency. The function does not inspect a transformation status or branch on a return value.

Crucially, subsequent operations use the output descriptor’s fields after this call. The source does not establish that the transformation preserves its data word or either dimension.

Consequently:

- The secondary resource operation uses the post-transformation data word.
- It uses the post-transformation 16-bit width and height fields.
- The later release operation uses the output descriptor’s then-current data word, not an independently retained copy of the original storage result.

The allocation request is based on the initial dimensions; it is not recomputed if the transformation changes those dimensions.

### 5. Select and perform the secondary resource operation

Only after the transformation returns does the function inspect the secondary resource word at object offset +0xD8.

- If the word is zero, it calls F-a29c8be2c8c1 with:
  1. The context word obtained by dereferencing the pointer captured near entry.
  2. The output descriptor’s current data word.
  3. Zero.
  4. The output descriptor’s current width.
  5. The output descriptor’s current height.

  The returned 32-bit word is written to object offset +0xD8.

- If the word is nonzero, it calls F-93dc585d2452 with:
  1. The stored secondary resource word.
  2. The output descriptor’s current data word.
  3. The output descriptor’s current width.
  4. The output descriptor’s current height.

  This function makes no explicit assignment to the secondary resource word after that update call.

The secondary state read therefore observes any changes to object offset +0xD8 made by earlier dependencies during the invocation. It is not a snapshot taken at entry.

### 6. Release the temporary data word

After either secondary creation or secondary update returns, the function calls F-5e3309c9d083 with the output descriptor’s current data word.

There is no explicit test for a zero value, no separate success check, and no visible conditional suppression of this call based on the creation result.

This is the final dependency call on the normal enabled-secondary path.

## Side effects

### Explicit writes

The function itself performs these externally relevant object writes:

- Primary creation: writes the creation result to +0xD4.
- Secondary disabled: writes zero to +0xD8.
- Secondary creation: writes the creation result to +0xD8.
- Update paths: do not explicitly replace the corresponding stored resource word.

It also writes temporary descriptor fields. Those fields become observable to F-6ee71f0e99b3 through the descriptor addresses passed to it.

### Call ordering

Normal enabled-secondary execution has this dependency order:

1. Primary creation or update.
2. Temporary-storage request.
3. Descriptor transformation.
4. Secondary creation or update.
5. Temporary-data release.

Normal disabled-secondary execution performs the primary creation or update, then clears the secondary word.

These are call and object-operation ordering guarantees visible in the supplied source. They do not specify machine-instruction ordering between otherwise independent argument loads.

### Ownership and lifetime

The function follows a request–use–release pattern for the temporary data word. That pattern does not prove the ownership contracts of the participating dependencies.

In particular, the source does not establish:

- Whether the storage dependency returns allocated memory or another kind of token.
- Whether transformation replaces, consumes, or releases its original output data word.
- Whether resource creation or update copies the temporary data, retains it, or takes ownership.
- Whether the final release is safe for zero or for a transformed replacement value.
- Whether the stored primary or secondary resource words require eventual destruction.

The descriptor addresses refer to invocation-local storage. There is no visible mechanism to extend their lifetime after the function returns. A dependency that retains either address would require an external lifetime contract not supplied here.

### Errors and interruption

There is no explicit validation, error return, rollback, or cleanup branch.

If dependencies return normally, execution continues as described even when a creation or storage result is zero. If a dependency does not return normally, subsequent writes and release calls are not established as occurring.

The function is not shown to be atomic or synchronized. Concurrent changes to the object or global context are outside the established contract.

## Dependencies

| Opaque function ID | Established use | Not established |
|---|---|---|
| F-a29c8be2c8c1 | Five-argument creation operation; returned word is stored in the selected object field. | Resource type, failure convention, data ownership, context mutation, retained references. |
| F-93dc585d2452 | Four-argument update operation for a nonzero resource word. | Failure reporting, mutation of resource state or globals, payload lifetime requirements. |
| F-d33f6d597e5d | Receives the computed unsigned 32-bit storage request; its result initializes the output data word. | Allocation policy, size limits, alignment, zero-size behavior, failure behavior. |
| F-6ee71f0e99b3 | Receives output and input descriptor addresses, in that order. | Transformation semantics, fields modified, bounds guarantees, ownership changes, retained addresses. |
| F-5e3309c9d083 | Receives the final output data word after the secondary resource call. | Release semantics, accepted value domain, ownership requirements. |

No dependency is assumed to preserve globals, object fields, or externally reachable storage merely because this function does not explicitly request those changes.

## Acceptance examples

These are conditional static acceptance cases, not reports of executed tests.

1. **Primary creation with secondary disabled.**  
   Given a zero primary word, extent 12, and selector zero, expect primary creation with dimensions 12 and 12. Its returned word is stored at +0xD4. Then +0xD8 is set to zero. No temporary-storage, transformation, or release call is made by this function.

2. **Primary update with secondary disabled.**  
   Given a nonzero primary word and a nonzero secondary word, selector zero selects primary update, followed by overwriting the secondary word with zero. No explicit destruction of the former secondary word occurs.

3. **Even reduced width.**  
   With extent 10 and a nonzero selector, the initial output dimensions are 10 by 5, the input dimensions are 10 by 10, and the storage request is 100. If transformation preserves the output fields, the secondary operation receives dimensions 10 by 5.

4. **Odd reduced width.**  
   With extent 5, the initial secondary dimensions are 5 by 2, and the request is 20. The height is not rounded up.

5. **Full-width primary, truncated secondary.**  
   With extent 65,537, primary creation or update receives 65,537 twice. Enabled secondary processing initializes width 1 and height 0 and requests zero storage. The subsequent transformation and secondary operation are still invoked if all dependencies return normally.

6. **Zero extent.**  
   Extent zero is passed unchanged to the primary operation. Enabled secondary processing builds zero dimensions, calls the storage dependency with zero, then continues without an explicit empty-case shortcut.

7. **Largest reduced dimensions.**  
   With extent 65,535, enabled secondary processing initializes width 65,535 and height 32,767 and requests 4,294,770,690. No source-level overflow reduction is expected for that expression. Whether the dependency accepts such a request is unknown.

8. **Transformation changes the output descriptor.**  
   If transformation replaces the data word and changes the dimensions, the secondary operation uses those replacement values. The final release receives the current data word rather than a saved original allocation result.

9. **Earlier dependency changes secondary state.**  
   If an earlier dependency changes +0xD8 from zero to a nonzero resource word, the later secondary decision selects update. Conversely, changing it to zero selects creation.

10. **Creation returns zero.**  
    A zero primary creation result is stored and does not itself prevent secondary processing. A zero secondary creation result is stored, and the final release still follows on normal return.

## Uncertainties

The following are **documented_only**:

- Semantic meanings of the payload, resources, dimensions, and transformation.
- Dependency signatures beyond the argument positions and value widths exposed here, including ABI extension details for the 16-bit dimension fields.
- Descriptor byte order and dependency-required alignment.
- Whether zero indicates failure, an empty resource, or another valid state.
- Ownership transfer, copying, retention, and release obligations.
- Dependency side effects on object fields, the global slot, or its pointed-to context word.
- Validity of the captured context pointer after intervening dependency calls.
- Behavior under invalid addresses, insufficient storage, failed allocation, dependency interruption, reentrancy, or concurrency.

**Inferred intent:** the temporary-storage request, descriptor transformation, and final release suggest a transient derived-data workflow. The differing descriptor dimensions suggest a shape-changing transformation. Neither inference establishes a specific data format or algorithm.

## Evidence

The established contract comes solely from the supplied function body: its accesses to two four-byte object fields, full-width primary arguments, low-16-bit secondary arithmetic, two eight-byte descriptors, dependency argument order, conditional creation/update branches, and final release call.

No dependency implementations, external ownership contracts, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples are consequences of the visible control flow and arithmetic, subject to their stated conditions.
