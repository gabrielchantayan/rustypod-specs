<!-- Trusted provenance: F-9dc23492acea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9dc23492acea

## Purpose

F-9dc23492acea conditionally updates one of two 32-bit fields in a caller-supplied context. Selection depends on a signed 16-bit value read through a caller-supplied indirect buffer reference, after a preliminary dependency reports a signed result of at least three.

The recognized selector values are:

- **0x004D:** operates on the context field at byte offset **0x30**.
- **0x0044:** operates on the context field at byte offset **0x38**.

Both recognized paths invoke a shared preparation dependency, request a resource using the value **0x14**, and invoke a branch-specific dependency with that resource and the address of context offset **0x54**. The returned 32-bit value is compared with the selected context field.

The function directly stores the returned value only when the selected field is zero and differs from that value. A different, nonzero existing value leads to two further dependency calls rather than a direct replacement store.

**Inferred intent:** the two fields may represent alternative state objects, with separate creation or initialization operations and a shared reconciliation operation. The supplied evidence does not establish object types, allocation semantics, reference counting, or destruction semantics.

## Interface

### Inputs

The function has two supplied arguments:

1. **Context reference.** The supplied representation is a 32-bit integer used as an address. The function accesses 32-bit fields at byte offsets **0x30** and **0x38**, depending on the selected path. It also computes the address at byte offset **0x54** and passes that address to a dependency.
2. **Indirect buffer reference.** A pointer to a 32-bit address-bearing word. The function passes this reference to dependencies and, when the preliminary check permits further processing, dereferences it to locate the selector.

The selector occupies a **16-bit signed field at byte offset 2 from the current buffer address**. It is not established to be a single character, despite the recognized values corresponding to common character encodings.

The source representation uses 32-bit integers and four-byte quantities for returned resources, saved configuration values, stored state, and reconciliation results. This contract describes those widths without asserting a portable language-level ABI.

### Output

There is no explicit return value.

Normal completion may leave the selected field unchanged or directly assign one 32-bit value to it. A conflict path reaches a dependency annotated as non-returning; normal completion of that path is therefore not established.

### Preconditions

The caller and surrounding environment must provide:

- An indirect buffer reference valid for the operations performed by F-d5d68dc4e17c.
- If that dependency returns at least three, a readable address word at the indirect buffer reference and a readable two-byte selector beginning at buffer offset 2.
- For a recognized selector, a context supporting the selected field access and the address passed at offset 0x54.
- Readable branch-specific global configuration words at offsets 0x104 and 0x100.
- Inputs satisfying any additional requirements of the invoked dependencies.

The visible function performs no null checks or independent memory-validity checks. The preliminary result does not, by itself, prove that the selector read is safe: the selector spans buffer offsets **2 and 3**, while the threshold is only **three**.

## Behavior

### Preliminary gate

The first observable operation is a call to **F-d5d68dc4e17c**, with the indirect buffer reference as its sole argument.

Its return is treated as a signed 32-bit integer:

- A result less than **3** causes immediate normal return.
- A result equal to or greater than **3** permits selector inspection.

The function does not visibly inspect the buffer before this call. Any buffer or reference changes made by the dependency therefore precede the selector reads.

A negative result satisfies the early-return condition. The function does not distinguish a negative error indication from any other result below three.

### Selector evaluation

After the preliminary gate, the function loads the current buffer address through the indirect reference and reads the signed 16-bit selector at buffer offset 2.

If that value is **0x004D**, the first recognized path executes.

If it is not 0x004D, the function evaluates a second selector read through the indirect reference and compares that value with **0x0044**. Thus, the non-0x004D route contains a second buffer-address load and a second selector load; it is not specified as one cached selector tested twice.

There is no intervening dependency call between those two comparisons on the non-0x004D route. The supplied evidence does not establish concurrency, volatile-memory semantics, or atomicity. Under stable memory, an unrecognized selector produces normal return after the second comparison.

The 0x004D route terminates before the second comparison, either by explicit return or by reaching the dependency annotated as non-returning.

### Recognized-path preparation

For either recognized selector, the operation order is:

1. Read a 32-bit value from offset **0x104** of that path’s global configuration block.
2. Read a 32-bit value from offset **0x100** of that block.
3. Invoke **F-c5267be1642a** with the context reference, the integer **7**, the integer **2**, and the indirect buffer reference.
4. Invoke **F-8157a180d901** with the integer **0x14**, decimal **20**.
5. Pass the returned 32-bit quantity and the address of context offset **0x54** to the branch-specific dependency.
6. Read the selected 32-bit context field.

The branch-specific dependency is:

| Selector | Dependency | Selected context field |
|---|---|---|
| 0x004D | F-4e35d69205e8 | Offset 0x30 |
| 0x0044 | F-93b74ed75390 | Offset 0x38 |

The two paths use separately referenced global configuration bases. Their equality, relationship, and contents are unknown.

The saved configuration values precede all three preparation calls. If those calls modify the configuration storage, this invocation nevertheless uses the values already captured when it later performs reconciliation.

Conversely, the selected context field is read **after** those calls. The function compares the branch-specific return with the field’s value at that later point, not with an entry-time snapshot. Dependencies may modify the context, indirect reference, buffer, or globals; preservation is not established.

There is no visible check of the result from F-8157a180d901 before passing it onward. In particular, a zero result is not handled as a local failure.

### Selected-field outcomes

Let the **candidate** denote the 32-bit return from the branch-specific dependency, and the **existing value** denote the subsequent read of the selected context field.

**Equal values:**  
If the existing value equals the candidate, the function returns normally without directly writing the field. This includes the case where both are zero.

**Zero existing value, different candidate:**  
If the existing value is zero and differs from the candidate, the function stores the candidate in the selected context field and returns normally. The candidate must therefore be nonzero on this direct-store path.

**Nonzero existing value, different candidate:**  
The function invokes **F-2f51066133e0** with these arguments, in this order:

1. The existing value plus **4**.
2. The candidate.
3. The configuration value captured from offset **0x100**.
4. The configuration value captured from offset **0x104**.

It then subtracts **4** from that dependency’s 32-bit return and passes the adjusted value to **F-feee9156a257**.

The supplied representation marks the latter call as non-returning. No ordinary return or subsequent local update is established for this path.

The function does not visibly store the candidate into the selected context field on this conflict path. Any replacement, release, or mutation performed by the dependencies is outside the locally established behavior.

### Arithmetic and representation boundaries

The visible address-like adjustments are:

- Context reference plus **0x54**.
- Existing nonzero field value plus **4**.
- Reconciliation return minus **4**.

The supplied representation places these operations in a 32-bit setting. No explicit overflow, underflow, alignment, or resulting-address validation appears. Exact behavior at representation boundaries is **documented_only**: the evidence is insufficient to choose between machine-address wrapping and language-level signed-arithmetic interpretations.

A candidate of zero with a nonzero existing value follows the conflict path. Zero is not universally rejected or treated as a request to return.

## Side effects

### Directly established accesses

Beyond effects internal to dependencies, the function performs:

- Reads of the indirect buffer-address word and selector, as required by selector evaluation.
- Two global configuration reads on a recognized path, in the order **0x104, then 0x100**.
- One read of the selected context field after preparation.
- At most one direct 32-bit store to that selected field, only for a zero existing value and a different candidate.

There is no visible direct write to the buffer, its indirect reference, the configuration blocks, or the context region at offset 0x54. Passing those addresses to dependencies may nevertheless expose them to mutation.

### Ownership and lifetime

The function forwards the resource returned by F-8157a180d901 to a branch-specific dependency. It does not visibly release that resource locally, including when the candidate equals an existing value.

That absence does not establish a leak. Allocation, consumption, transfer, retention, and release behavior are unknown.

Likewise, the adjusted addresses passed on the conflict path do not establish a specific object-header layout or destruction protocol. No ownership guarantees should be derived from the four-byte adjustments alone.

### Errors and termination

There is no explicit local error status, diagnostic, exception handler, or rollback.

The only locally established early rejection is the signed result below three from F-d5d68dc4e17c. Unrecognized selector values cause normal return without recognized-path preparation.

Dependency failures and invalid memory accesses have no visible recovery path. The conflict path reaches a call annotated as non-returning; its actual termination mechanism is unknown.

## Dependencies

| Opaque ID | Locally established role |
|---|---|
| **F-d5d68dc4e17c** | Receives the indirect buffer reference; its signed return gates processing at three. |
| **F-c5267be1642a** | Receives the context, constants 7 and 2, and the indirect buffer reference before resource acquisition. No return value is used. |
| **F-8157a180d901** | Receives 20 and returns a 32-bit quantity forwarded without a local validity check. |
| **F-4e35d69205e8** | Handles selector 0x004D; receives the resource result and context address at offset 0x54; returns the candidate. |
| **F-93b74ed75390** | Handles selector 0x0044 with the corresponding argument shape and candidate role. |
| **F-2f51066133e0** | Handles unequal values when the existing field is nonzero; receives the adjusted existing value, candidate, and two saved configuration values. |
| **F-feee9156a257** | Receives the reconciliation result minus four; annotated as non-returning. |

These roles describe observed call relationships, not complete dependency contracts.

## Acceptance examples

The following are conditional contract examples, not runtime results. Unless stated otherwise, memory is valid and stable, dependencies return normally where expected, and their effects are limited to those explicitly specified.

1. **Gate rejects processing.**  
   F-d5d68dc4e17c returns 2, zero, or a negative signed value. The function returns without performing its own selector read or invoking recognized-path dependencies.

2. **Threshold permits inspection.**  
   The preliminary result is exactly 3 and selector storage is readable. Selector processing occurs. The result 3 does not establish that a three-byte allocation alone is sufficient for the two-byte read at offset 2.

3. **Unrecognized selector.**  
   Both selector reads observe 0x0045. The function returns without configuration reads, preparation, resource acquisition, or selected-field access.

4. **0x004D installs a candidate.**  
   The first selector read observes 0x004D. F-4e35d69205e8 returns a nonzero candidate, and the subsequent read at context offset 0x30 is zero. The function stores that candidate at offset 0x30 and returns. It performs no direct store at offset 0x38.

5. **0x0044 preserves equality.**  
   The first selector read is not 0x004D and the second observes 0x0044. F-93b74ed75390 returns the same value later read at context offset 0x38. The function returns without a direct field write or either conflict-path call.

6. **Zero equals zero.**  
   Either recognized path returns a zero candidate, and the selected field is zero when read. The equality case applies; the function does not perform a redundant zero store.

7. **Conflicting nonzero state.**  
   The selected field is read as 0x1000 and the candidate is 0x2000. F-2f51066133e0 receives 0x1004, 0x2000, the saved offset-0x100 value, and the saved offset-0x104 value. If it returns 0x3004, F-feee9156a257 receives 0x3000. Normal return is not established.

8. **Preparation changes the selected field.**  
   The selected field is initially zero, but a preparation dependency changes it to the eventual candidate before the field read. The function takes the equality path rather than performing a store.

9. **Configuration changes after capture.**  
   A preparation dependency changes the configuration words. A subsequent conflict call still receives the two values captured before preparation, in offset-0x100 then offset-0x104 argument order.

## Uncertainties

The following remain **documented_only**:

- The meaning of the preliminary result, including whether it measures bytes, elements, availability, or another property.
- Buffer format, byte order, alignment requirements, and synchronization.
- The meaning of selectors 0x004D and 0x0044.
- The identities and relationship of the two global configuration blocks.
- Whether the resource-producing dependency allocates memory.
- Ownership, lifetime, aliasing, and release obligations for all returned values.
- Complete dependency side effects, failure modes, and preservation guarantees.
- The interpretation of the context region beginning at offset 0x54.
- Arithmetic behavior at 32-bit address boundaries.
- Whether the non-returning annotation is an accurate runtime property and, if so, how termination or control transfer occurs.

## Evidence

This contract derives from static inspection of the supplied function representation. Established behavior consists of the visible accesses, widths, comparisons, constants, call arguments, ordering, direct stores, and annotated control-flow endpoints.

Object-management terminology is inferred intent only. The representation does not supply dependency implementations or a complete memory-layout specification.

No runtime tests have been performed. The acceptance examples specify conditional expected behavior; they are not claims of observed execution.
