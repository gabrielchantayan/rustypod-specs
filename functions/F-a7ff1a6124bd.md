<!-- Trusted provenance: F-a7ff1a6124bd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a7ff1a6124bd

## Purpose

F-a7ff1a6124bd conditionally reports whether bit 11 is set in a 32-bit word associated with an input object.

The operation first invokes a dynamically selected dependency. A zero result from that invocation prevents any subsequent candidate selection and produces zero. A nonzero result permits selection of a candidate: either a value already present in the input object or, when that value is zero, the result of F-fc6b91b5a714. A nonzero candidate supplies the address of the word to inspect.

The normal return value is always either zero or one. Zero is not a distinct error indication: it also represents a closed gate, an unavailable candidate, or a clear inspected bit.

This describes the behavior visible in the supplied static evidence. Interpreting the objects as particular application entities, or interpreting bit 11 as a named capability or status, would require information not provided.

## Interface

**Input:** one address identifying an object with the following relevant layout.

| Byte offset | Width | Role |
|---|---:|---|
| 0 | 32 bits | Address of a dispatch structure |
| 4 | 32 bits | Argument supplied to F-fc6b91b5a714 if fallback selection is needed |
| 8 | 32 bits | Preferred candidate address; zero selects the fallback path |

The dispatch structure contains a callable address at byte offset 8. That callable receives the original input-object address as its sole visible argument.

A selected nonzero candidate must support reading a 32-bit word at byte offset 28 from its address. Bit 11 of that word determines the returned flag.

**Output:** an unsigned result, represented as 32 bits in the supplied evidence, with the following normal-return range:

- **0:** no affirmative bit result was obtained.
- **1:** a candidate was selected and bit 11 of its inspected word was set.

The integer and address-bearing layouts above follow the apparent 32-bit representation in the supplied evidence. They are not an independently verified platform ABI. In particular, the widths of object fields must not silently become host-native pointer widths when applying this contract elsewhere.

### Preconditions

The routine performs no visible validation before its first indirect invocation. For normal execution:

- The input-object address must permit reading the dispatch-structure address.
- The dispatch structure must permit reading the callable address at byte offset 8.
- That address must identify an invocable dependency under the applicable calling convention.
- After a nonzero gate result, the input object must remain valid for the subsequent field reads required by the selected path.
- A selected nonzero candidate must permit the final 32-bit read at byte offset 28.

A zero candidate is handled explicitly. A zero input-object address, invalid dispatch structure, invalid callable address, or invalid nonzero candidate is not handled explicitly.

These are access-validity requirements, not proof that the routine owns any of the referenced storage.

## Behavior

### Gate invocation

Before considering either candidate source, the routine obtains the dispatch-structure address from the input object, obtains the callable address at dispatch-structure byte offset 8, and invokes that callable with the input-object address.

The invocation occurs even if the input object's preferred candidate is already nonzero.

The callable's result is tested only for zero versus nonzero. There is no visible restriction to positive values or to a Boolean encoding. Under the apparent 32-bit signed result representation, a negative result opens the gate just as any other nonzero result does.

If the result is zero, the routine returns zero immediately. On this path it does not subsequently read the preferred-candidate field, read the fallback-argument field, invoke F-fc6b91b5a714, or inspect a candidate word.

The callable may modify the input object or other state. No preservation guarantee is established. Candidate selection uses fields read **after** this invocation returns, rather than a snapshot taken before it.

### Candidate selection

After a nonzero gate result, the routine reads the preferred candidate from input-object byte offset 8.

If that value is nonzero, it becomes the selected candidate. F-fc6b91b5a714 is not invoked, and the input field at byte offset 4 is not needed by this routine on that path.

If the preferred candidate is zero, the routine reads the value at input-object byte offset 4 and invokes F-fc6b91b5a714 with:

1. The original input-object address.
2. The value just read from byte offset 4.

A zero result from F-fc6b91b5a714 produces a zero return from this routine without a candidate-word read. A nonzero result becomes the selected candidate.

There is no visible retry, second fallback, or attempt to recover from an invalid nonzero candidate. Selection of a nonzero preferred candidate excludes fallback selection even if a later read through that candidate cannot succeed.

The fallback result is used directly. The routine does not visibly store it into the preferred-candidate field or reread that field after the fallback invocation. If F-fc6b91b5a714 changes the object, those changes do not replace its returned value as the selected candidate for this invocation.

### Flag extraction

For a selected nonzero candidate, the routine reads the 32-bit word at candidate byte offset 28. It retains only the bit selected by hexadecimal mask `0x00000800` and shifts that retained bit right by 11 positions.

Consequently:

- A clear bit 11 produces zero.
- A set bit 11 produces one.
- Every other bit in the word is irrelevant to the returned value.

The extraction is unsigned in the supplied evidence. A set top bit does not cause sign extension or alter the result.

### Access and call ordering

The relevant sequence is:

1. Read the input object's dispatch-structure address.
2. Read the callable address from that structure.
3. Invoke the dynamically selected dependency.
4. If its result is nonzero, read the preferred candidate.
5. Only if that candidate is zero, read the fallback argument and invoke F-fc6b91b5a714.
6. Only if the chosen candidate is nonzero, read its word at byte offset 28.
7. Return the extracted bit, or zero from an earlier terminating condition.

This is the order supported by the supplied static representation. It is not a claim about processor-level memory transactions, synchronization, or compiler guarantees for externally changing memory.

## Side effects

The routine contains no visible direct writes to the input object, dispatch structure, selected candidate, or global storage. It contains no visible allocation, release, reference-count adjustment, registration, or explicit error-reporting operation.

Those absences do not make the operation pure:

- The dynamically selected dependency is always invoked on normal entry.
- F-fc6b91b5a714 is invoked conditionally.
- Either dependency may have side effects not described by the available evidence.
- Either dependency may alter data subsequently read by this routine.
- Either dependency may fail to return normally.

No ownership transfer is established for the input object, fallback argument, or candidate. No assumption may be made that dependencies preserve global state, retain arguments, release memory, or return owned memory.

The routine itself does not release the selected candidate after reading it. That observation does not establish a borrowed-reference convention; the lifetime and ownership contract remains undocumented.

There is no visible exception handler or fault-recovery mechanism. Access failures, invalid invocation targets, and dependency failures are not converted into a defined zero result by the shown behavior.

## Dependencies

### F-opaque-01

This is a document-local opaque identifier for the dynamically selected callable; its actual identity is unavailable.

- **Selection:** callable address at dispatch-structure byte offset 8.
- **Input:** the original input-object address.
- **Consumed output:** an integer tested for zero versus nonzero.
- **Invocation condition:** every normally entered execution that reaches the indirect call.
- **Effect on control flow:** zero terminates the routine with zero; nonzero permits candidate selection.
- **Unknowns:** semantic purpose, side effects, ownership behavior, failure behavior, and preservation guarantees.

The identifier does not assert that every input object selects the same callable.

### F-fc6b91b5a714

- **Inputs:** the original input-object address and the 32-bit value read from input-object byte offset 4.
- **Consumed output:** zero or a candidate address.
- **Invocation condition:** the gate returned nonzero and the subsequently read preferred candidate was zero.
- **Effect on control flow:** zero terminates the routine with zero; nonzero selects the address used for the final word read.
- **Unknowns:** how the result is obtained, whether state is mutated, ownership and lifetime of the result, and failure behavior.

A cache lookup, lazy creation, or resolution role is possible inferred intent, not established behavior.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Each assumes all required accesses are valid and invoked dependencies return normally.

| Conditions | Required result and behavior |
|---|---|
| F-opaque-01 returns zero; preferred candidate would otherwise be nonzero | Return zero. Do not subsequently inspect the preferred candidate or invoke F-fc6b91b5a714. |
| F-opaque-01 returns a negative nonzero value; preferred candidate is nonzero; candidate word is `0x00000800` | Return one. Do not invoke F-fc6b91b5a714. |
| Gate is nonzero; preferred candidate is nonzero; candidate word is `0xFFFFF7FF` | Return zero because bit 11 is clear, despite all other bits being set. |
| Gate is nonzero; preferred candidate is zero; F-fc6b91b5a714 returns zero | Return zero without reading a candidate word. |
| Gate is nonzero; preferred candidate is zero; fallback returns a valid candidate whose word is `0xFFFFFFFF` | Return one. |
| Gate is nonzero; selected candidate word is `0x80000000` | Return zero. The top bit does not substitute for bit 11. |

Additional state-sensitive cases establish the timing requirements:

- If F-opaque-01 changes the preferred candidate from zero to a valid nonzero address and returns nonzero, the changed candidate is used without fallback.
- If F-opaque-01 clears the preferred candidate and changes the fallback argument before returning nonzero, fallback receives the changed argument.
- If F-fc6b91b5a714 stores one candidate into the input object but returns a different valid candidate, the returned candidate supplies the inspected word.
- If the preferred candidate is nonzero but invalid, this contract does not permit silently switching to fallback or reporting a defined zero. That failure case has no established normal-return behavior.

## Uncertainties

The following remain **documented_only**:

- The concrete ABI, alignment requirements, byte order, and independently confirmed widths. The layout above reflects the apparent 32-bit source representation.
- The concrete identities and semantics of dynamically selected dependencies.
- The meaning of the input object, fallback argument, candidate, and inspected bit.
- Ownership, retention, lifetime, and release obligations across dependency calls.
- Whether dependencies can mutate globals, invalidate objects, recurse, or fail through nonlocal control flow.
- Thread safety, concurrent mutation behavior, atomicity, and synchronization.
- Validity of address arithmetic near an address-space boundary. No wraparound-based behavior is promised.
- Hardware or runtime behavior for malformed addresses, misalignment, inaccessible storage, or invalid invocation targets.
- Whether all normally occurring objects satisfy the stated access preconditions.

The apparent intent is a gated, optionally resolved flag query. That interpretation is narrower than claiming validation, caching, authorization, or resource management, none of which is established.

## Evidence

The supplied static material establishes the gate call, its zero/nonzero test, the conditional preferred-candidate read, fallback invocation, final 32-bit mask-and-shift operation, and absence of visible direct writes.

The contract distinguishes those observations from inferred purpose and unresolved dependency behavior. No runtime tests, dependency inspection, ABI validation, or observed failure experiments have been performed.
