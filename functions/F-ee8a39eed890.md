<!-- Trusted provenance: F-ee8a39eed890; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ee8a39eed890

## Purpose

F-ee8a39eed890 conditionally obtains and initializes a record associated with an object reference and a selector supplied through an input descriptor. It returns either that record’s address or a null pointer.

The established behavior has two successful initialization modes:

- **Zero-classification mode:** uses two separately read values from the object at byte offset 112 and marks the record with a mode value of one.
- **Nonzero-classification mode:** resolves a position, obtains an associated entity, performs a bracketed dependency operation on that entity, and marks the record with a mode value of zero.

These names describe branch behavior, not established application concepts. The meanings of the object, selector, position, associated entity, and record resource remain undocumented.

A non-null return establishes that the function completed one of these paths. It does not establish that every dependency operation succeeded: several dependency results are ignored, and their failure conventions are unknown.

## Interface

### Input descriptor

The function accepts one pointer to a descriptor containing at least two consecutive 32-bit words:

| Byte offset | Role |
|---|---|
| 0 | Object-reference word, captured once at entry |
| 4 | Selector word, read when classification is requested |

The first word is read before the initial gate dependency is called. Consequently, even a call that will fail the gate requires the descriptor’s first word to be readable.

The second word is not captured at entry. It is read for the initial classification and, on one branch, read again for a second classification. Changes to that word between those reads can affect later behavior.

The captured object-reference word is used both as a dependency argument and as an address base for object reads. This interpretation requires a compatible target address representation.

### Returned record

The result is a pointer to a record, or null. The visible accesses use the following 32-bit word layout:

| Byte offset | Role established by this function |
|---|---|
| 0 | Object-reference field |
| 4 | Resource field, consumed and sometimes replaced |
| 8 | First position or object-derived field |
| 12 | Second position or object-derived field |
| 16 | Auxiliary field, explicitly initialized to zero |
| 20 | Mode field: one for zero-classification mode, zero otherwise |
| 24 | Associated-entity field, written only in nonzero-classification mode |

These are descriptive roles, not a claim about a public type definition.

F-e77faabfc1bb supplies the record pointer. Its result must support the accesses reached on the selected path. The zero-classification path accesses through byte offset 23; the other path accesses through byte offset 27. The resource field at offset 4 can be read before this function assigns it, so the record provider must supply a usable value whenever the selected dependency path consumes that field.

### Widths and address requirements

The supplied representation uses 32-bit integers and four-byte record words. Object references, entity references, and several dependency results are carried in those words. The record result itself is represented as a pointer.

The established constant-offset accesses are:

- Object-reference base plus 112 bytes.
- Associated-entity reference plus 24 bytes.

The source does not establish signed-overflow handling, pointer provenance rules, alignment guarantees, or valid address ranges. Valid, suitably aligned storage and meaningful address arithmetic are preconditions for defined use; no local validation enforces them. The contract does not extend these 32-bit layouts to a 64-bit pointer ABI.

## Behavior

### Entry and gating

The function first reads and retains the descriptor’s object-reference word. It then calls F-e135feff4093 without explicit arguments.

If that dependency returns zero, the function returns null. It does not read the selector, obtain a record, or explicitly write record fields on this path. The gate dependency may nevertheless have side effects.

If the gate returns nonzero, the function reads the descriptor’s selector word and calls F-40e4fc419124 with the retained object reference and that selector. Only whether this first classification result is zero determines the main branch.

The retained object reference is not refreshed from the descriptor after entry.

### Zero-classification mode

If the first classification returns zero, the function calls F-e77faabfc1bb without explicit arguments.

A null result ends the function with a null return. No record initialization follows.

For a non-null record, F-b7aa346e402e is called with the retained object reference, zero, and zero. Its return value selects the resource operation:

- **Zero result:** the function reads the record’s resource field and calls F-eacf2e4e8b78 with that field, the retained object reference, zero, and zero. This function performs no explicit assignment to the resource field on this subpath.
- **Nonzero result:** the function calls F-25bf76bbe81a with no explicit arguments, then writes the retained nonzero result into the record’s resource field.

The result of F-b7aa346e402e is retained across the intervening call. It is not recomputed before being stored.

The function then performs these explicit operations in order:

1. Writes the retained object reference to record offset 0.
2. Reads a 32-bit word at object byte offset 112 and writes it to record offset 8.
3. Reads a 32-bit word at object byte offset 112 again and writes it to record offset 12.
4. Writes one to record offset 20.
5. Writes zero to record offset 16.
6. Returns the record pointer.

The two object reads are distinct. Equality of the fields at offsets 8 and 12 is not guaranteed if the underlying value can change between reads.

This branch makes no explicit write to record offset 24.

### Nonzero-classification mode

If the first classification returns nonzero, the function calls F-e77faabfc1bb without explicit arguments.

A null result causes a null return. In particular, the second classification is not performed when record acquisition fails.

For a non-null record, initialization proceeds in this order:

1. Write the retained object reference to record offset 0.
2. Read the descriptor’s selector again.
3. Call F-40e4fc419124 with the retained object reference and this newly read selector.
4. Pass the retained object reference and the second classification result to F-08426f0ea83d.
5. Retain the result of F-08426f0ea83d as the resolved position.
6. Write that position to record offset 8.
7. Write the retained object reference to record offset 0 again.
8. Write the resolved position to record offset 12.
9. Write zero to record offset 16.
10. Write zero to record offset 20.
11. Call F-04b7df4e1a09 with the retained object reference and the resolved position.
12. Write its result to record offset 24.

The second classification result is not required to equal the first, and it is not checked for zero. The first result selects the mode; the second supplies input to the resolver.

If the associated-entity result is zero, the function passes the record pointer to F-66819ba2201a and returns null. Fields have already been written before this cleanup call. There is no visible rollback.

For a nonzero associated entity, the function continues in the following order:

1. Call F-4c6d30ac2311 without explicit arguments.
2. Read the record’s resource field.
3. Call F-5c3b4734522b with that resource value and the associated-entity reference plus 24 bytes.
4. Call F-cc60a532ba6c without explicit arguments.
5. Call F-2b1ba6d8de9d with the retained associated-entity reference and zero.
6. Call F-b7aa346e402e with the retained object reference, retained resolved position, and zero.

The adjacent no-argument calls bracket F-5c3b4734522b, but their purpose is unknown. They are not established to acquire or release a lock.

If the final F-b7aa346e402e result is zero, the function reads record offsets 4, 0, 12, and 16 and passes those values, in that argument-position order, to F-eacf2e4e8b78. The exact evaluation order among these four reads is not established.

Those arguments come from the record’s then-current contents. They must not be substituted with earlier retained values: intervening dependencies may have changed the record.

If the result is nonzero, the function reads record offset 4, passes it to F-25bf76bbe81a, and then overwrites offset 4 with the retained nonzero result.

Both subpaths return the record pointer. No local status check follows either resource operation.

## Side effects

The explicit side effects are record-field writes and dependency calls. The object and input descriptor are not explicitly written by this function. That does not establish their immutability across the call: dependencies may alter accessible memory or global state.

Write ordering is part of the contract. In particular:

- Zero-classification mode performs resource handling before writing the object and position fields.
- Nonzero-classification mode writes several record fields before requesting an associated entity.
- A zero associated-entity result is stored before cleanup is invoked.
- Replacement resource values are written only after the corresponding F-25bf76bbe81a call.
- No explicit clearing of offset 24 occurs in zero-classification mode.

The function performs no visible direct allocation or deallocation operation. Record acquisition and cleanup are delegated. A non-null record provider result is not proven to be freshly allocated, uniquely owned, or initially zeroed.

The sole explicit record-cleanup call occurs after a zero associated-entity result in nonzero-classification mode. Null record-provider results do not trigger that cleanup. Successful returns do not establish how the caller must eventually dispose of the record or its resource.

There is no local exception handler, retry, error-code translation, or rollback. If a dependency does not return normally, subsequent behavior is not established here.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-e135feff4093 | Initial gate; zero causes immediate null return after the first descriptor read |
| F-40e4fc419124 | Classification using object and selector; called once or twice |
| F-e77faabfc1bb | Supplies the candidate record pointer; null ends either initialization path |
| F-b7aa346e402e | Produces a zero/nonzero resource-selection result from object, position-like value, and zero |
| F-eacf2e4e8b78 | Receives the existing resource and three additional values when resource selection returns zero |
| F-25bf76bbe81a | Called before replacing the resource field; explicit argument shape differs between branches |
| F-08426f0ea83d | Resolves the second classification result with the object reference |
| F-04b7df4e1a09 | Obtains the associated entity from object and resolved position |
| F-66819ba2201a | Receives the partially initialized record when the associated entity is zero |
| F-4c6d30ac2311 | Called immediately before the resource/entity-offset operation |
| F-5c3b4734522b | Receives the resource field and associated-entity reference plus 24 |
| F-cc60a532ba6c | Called immediately after that operation |
| F-2b1ba6d8de9d | Receives the associated entity and zero before final resource selection |

The supplied call representations do not establish complete dependency prototypes. In particular, the no-explicit-argument and one-explicit-argument uses of F-25bf76bbe81a must remain distinct observations; neither proves what that dependency releases or otherwise changes.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests.

### Gate refusal

Given a readable first descriptor word and a gate result of zero, the result is null. No selector read or record-provider call occurs. Gate side effects remain possible.

### Record acquisition failure

Given a nonzero gate result and either first-classification outcome, a null record-provider result produces a null return. For a nonzero first classification, no second classification or resolver call occurs.

### Zero-classification initialization without replacement

Assume the first classification and resource-selection results are zero, all dependencies return normally, and dependencies do not modify the record unexpectedly. If the two successive reads at object offset 112 yield 30 and 31, record offsets 8 and 12 receive 30 and 31 respectively. Offset 20 receives one, offset 16 receives zero, and the record is returned.

There is no requirement that offset 24 be cleared.

### Selector changes between classifications

Assume the first classification is nonzero. If the selector changes before its second read, the second classification receives the changed selector. The second result is passed to F-08426f0ea83d even if it is zero. The function does not switch to zero-classification mode.

### Associated entity unavailable

Given successful record acquisition and resolution, if F-04b7df4e1a09 returns zero, offset 24 is written with zero, F-66819ba2201a receives the record, and the function returns null. None of the subsequent bracketing or resource-selection calls occurs.

### Final resource handling observes changed record fields

Assume a nonzero associated entity and that an intervening dependency changes offsets 0, 12, or 16. If final resource selection returns zero, F-eacf2e4e8b78 receives the current record fields, not necessarily the initially written object, resolved position, and zero.

If selection instead returns a nonzero replacement, F-25bf76bbe81a receives the current resource field before the replacement is stored.

## Uncertainties

**Documented only:**

- Dependency internals, failure conventions, global-state effects, and ownership rules.
- Whether the record provider allocates, reuses, or exposes shared storage.
- The semantic meaning of the selector, resolved position, modes, and object offset 112.
- Whether the bracketing calls implement synchronization.
- Whether the cleanup dependency frees the record, releases contained resources, or performs another operation.
- Whether any dependency can invalidate the descriptor, object, record, or associated entity.
- Exact target ABI, full dependency signatures, and behavior of invalid or overflowing address calculations.
- Concurrency guarantees and ordering beyond the visible sequential operations.

**Inferred intent:** the function appears to construct or prepare a stateful record with alternate positioning modes and resource handling. This interpretation is not required to explain the established reads, writes, calls, and return conditions.

## Evidence

The contract is based solely on the supplied static function representation and the opaque dependency mapping. Branch predicates, explicit field accesses, call order, retained values, and return paths provide the established evidence.

No dependency implementations, authoritative type declarations, allocator contracts, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples state conditional consequences of the visible behavior, not measured outcomes.
