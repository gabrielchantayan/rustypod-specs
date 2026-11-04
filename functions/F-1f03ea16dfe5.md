<!-- Trusted provenance: F-1f03ea16dfe5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1f03ea16dfe5

## Purpose

F-1f03ea16dfe5 produces a 32-bit result by consulting dynamically dispatched operations on two supplied objects.

Its behavior depends on an **ordered sequence of returned values**, not necessarily on a stable property of either object. The first object can be queried twice and the second object can be queried three times. Repeated queries must not be merged or cached: the supplied evidence does not establish that they return the same value or have no side effects.

The result is either zero or a 32-bit value sampled from global storage immediately after the first query. Zero therefore does not uniquely identify which path was taken; the sampled global value may itself be zero.

Any interpretation as a compatibility check, type relationship, or permission decision is inferred intent only. No such domain meaning is established.

## Interface

**Inputs**

- **First object:** an address identifying an object whose leading 32-bit field supplies a dispatch-table address in the represented execution model.
- **Second object:** another object address with the same required dispatch layout when a path reaches a query on that object.

For each query, the operation is selected through the dispatch-table entry at byte offset **28**, or hexadecimal **0x1c**, from the address obtained from the object's leading field. The selected operation receives the queried object address as its explicit argument.

**Output**

- A 32-bit return quantity.
- Zero is returned on the specific paths defined below.
- Otherwise, the return quantity is the unchanged 32-bit bit pattern sampled from global storage after the first query.
- No signed interpretation, Boolean normalization, or ownership meaning is established for that global value.

**Required accessible layout**

Within the represented 32-bit layout, each executed dispatch requires:

1. A readable leading 32-bit field at the queried object address.
2. A usable dispatch-table address obtained from that field.
3. A readable function-pointer entry at dispatch-table byte offset 28.
4. A callable target compatible with receiving the object address and returning a value compared as a 32-bit integer.

Only this leading field and dispatch entry are structurally exposed by the routine. No minimum complete object size, additional fields, inheritance arrangement, or object-construction protocol is established.

**Preconditions**

All object and dispatch accesses actually reached during execution must be valid. The selected targets must satisfy the calling convention and return normally for the normal-return behavior in this contract to apply.

The routine performs no explicit null checks, layout validation, target validation, or object-lifetime checks. A second object that is never queried is not explicitly dereferenced by this routine, although an invoked dependency could access it through aliases or other state.

The two arguments need not be distinct according to the visible routine. If they alias, both query roles operate on the same object, with the resulting sequence and side effects preserved.

## Behavior

### Initial query and global sampling

The routine first queries the first object through its dispatch entry at byte offset 28.

After that query returns, it reads a 32-bit global value. This sampled value is retained as the prospective nonzero-path result, although its actual bit pattern may be zero.

This ordering is essential:

- The initial query occurs before the global sample.
- A change to the global value made by that query can affect the sampled result.
- Later queries do not cause the routine to resample the global value.
- The global read occurs even when a subsequent decision returns literal zero.

The following branches use the return value of the initial query.

### Initial result equals 5

If the initial query on the first object returns 5:

- The first object is not queried again by this routine.
- The second object is queried once.
- If that query returns 6, the routine returns zero.
- For every other returned value, the routine returns the previously sampled global value.

There are exactly two indirect calls on a normally completing execution of this branch.

### Initial result does not equal 5

If the initial query returns anything other than 5, the routine queries the first object again. This is a new dispatch and a new call, not reuse of the initial result.

If the second query on the first object returns anything other than 6, the routine immediately returns zero. No query on the second object is made by this routine.

If the second query on the first object returns 6, the routine proceeds to query the second object:

1. On its first query, a result of 7 causes an immediate zero return.
2. Otherwise, a second query is made. A result of 8 causes an immediate zero return.
3. Otherwise, a third query is made. A result of 9 causes an immediate zero return.
4. If none of those three position-specific comparisons succeeds, the sampled global value is returned.

The comparisons are tied to query position. The first query on the second object is checked against 7, the second against 8, and the third against 9. A returned value matching a different position's constant does not by itself cause an early return.

This branch performs between two and five indirect calls in total.

### Dispatch and observation order

Each query independently obtains the dispatch information from the object being queried. The routine does not visibly retain a selected target for reuse across queries.

Consequently, a dependency that changes an object's leading dispatch field or a table entry can affect a later query. Such mutation must leave every subsequently reached access valid for normal execution to continue.

For normally returning dependencies and valid accesses, the possible call-role sequences are:

| Condition reached | Query sequence | Result |
|---|---|---|
| Initial first-object result is 5; second-object result is 6 | First, second | Zero |
| Initial first-object result is 5; second-object result is not 6 | First, second | Sampled global |
| Initial first-object result is not 5; repeated first-object result is not 6 | First, first | Zero |
| Repeated first-object result is 6; first second-object result is 7 | First, first, second | Zero |
| First second-object result is not 7; next result is 8 | First, first, second, second | Zero |
| Prior second-object checks fail; third result is 9 | First, first, second, second, second | Zero |
| All reached second-object checks fail | First, first, second, second, second | Sampled global |

In every row, the global sample occurs immediately after the first call and before the second call.

### Widths and arithmetic

The evidence represents query results as 32-bit integers, compared for exact equality or inequality with 5, 6, 7, 8, and 9. No range check or conversion to a smaller classification field is shown.

The return value and global sample are 32-bit quantities. The sampled value is returned without visible arithmetic or transformation.

The dispatch access uses a fixed byte displacement of 28. There is no loop counter, variable index calculation, result accumulation, or arithmetic-overflow behavior in the visible decision logic.

## Side effects

The routine itself visibly performs:

- Reads needed to resolve each reached indirect call.
- One read of the global value after the initial call.
- Two to five indirect calls on normally completing paths.

It contains no explicit object-field writes, global writes, allocation, deallocation, reference-count adjustment, or resource-release operation.

These observations do **not** establish that the whole operation is read-only. Each indirect dependency may mutate objects, dispatch structures, the sampled global's backing storage, or other reachable state. Its effects are unknown.

There is no visible ownership transfer or explicit release of either input. Nevertheless, the routine's absence of a release operation does not prove that an invoked dependency preserves object lifetime or memory ownership. A dependency that invalidates an object or table needed by a later query can prevent normal completion.

Changes to the global storage after sampling do not alter the already sampled return quantity. Conversely, this does not imply that later calls preserve that storage.

No explicit error result, exception handler, retry mechanism, or recovery path is present. Invalid memory, an incompatible target, a nonreturning target, or abnormal control transfer has no defined normal result in this contract.

## Dependencies

**F-000000000001 — unresolved indirect-call dependencies**

This opaque ID labels the unresolved dispatch-target family used by the queries. It does not assert that all queries reach one function, or that two objects share a target.

The visible dependency contract is limited to:

- Target selection from the currently queried object's dispatch-table entry at byte offset 28.
- The queried object address passed as an explicit argument.
- A returned 32-bit integer consumed by the caller's position-specific comparison.

Target identity, internal behavior, side effects, ownership effects, and failure behavior are documented_only unknowns. No constancy or purity guarantee is established.

**G-1 — sampled global storage**

G-1 is a documentation label for the 32-bit global storage read after the initial query. Its domain meaning, initialization, synchronization rules, and permitted values are unknown.

G-1 is data, not an additional function dependency.

## Acceptance examples

These examples are conditional acceptance criteria derived from the supplied static evidence. They have not been executed. Each assumes valid accesses, compatible targets, and normal returns. “Sample” means the global value read immediately after the initial call.

| First-object results, in call order | Second-object results, in call order | Sample | Required return |
|---|---|---|---|
| 5 | 6 | Any 32-bit value | Zero |
| 5 | 7 | 23 | 23 |
| 6, 4 | Not queried | 23 | Zero |
| 4, 6 | 7 | 23 | Zero |
| 4, 6 | 0, 8 | 23 | Zero |
| 4, 6 | 0, 0, 9 | 23 | Zero |
| 4, 6 | 8, 9, 7 | 23 | 23 |
| 4, 6 | 0, 0, 0 | Zero | Zero |

Additional ordering and state criteria:

- **Initial-call mutation:** If G-1 initially contains 11 and the initial query changes it to 23 before returning, a path returning the sample must return 23.
- **Later-call mutation:** If the sample is 23 and a later query changes G-1 to 41, a path returning the sample must still return 23.
- **Repeated first-object query:** An initial result of 6 does not bypass the second first-object query. If that second result is 4, the required result is zero without a second-object query.
- **Position-sensitive second-object checks:** Results of 8, 9, and 7, in that order, fail all three respective comparisons against 7, 8, and 9. The routine must return the sample.
- **Early termination:** After any matching zero-return comparison, no remaining query is made by this routine.
- **Changed dispatch:** If an earlier dependency changes a dispatch entry needed later, the later query resolves through the updated structure, subject to valid memory and calling-convention requirements.
- **Aliased inputs:** If both arguments identify the same object, the call sequence remains unchanged. Returned values are interpreted by call position, not by distinct object identity.

## Uncertainties

**Established static evidence**

The call order, branch comparisons, dispatch offset, single global sample, and choice between zero and the sampled value are directly represented in the supplied routine.

**Inferred intent**

The small comparison constants may represent categories or states. The routine may express a relationship between two objects. Neither interpretation is required to explain its behavior and neither is adopted as a contract.

**documented_only unknowns**

- The meanings of the comparison constants and global value.
- The concrete identities and implementations of indirect targets.
- Whether any target returns stable results across repeated calls.
- Whether targets mutate or release either object or other resources.
- Whether the global value encodes a pointer, status, token, or other quantity.
- Synchronization, atomicity, volatile-access requirements, and concurrent-mutation behavior.
- Error or exception behavior originating inside dependencies.
- Full object layout and lifetime requirements beyond the reached accesses.
- Original source-level types and portability beyond the represented 32-bit layout.

No guarantee of thread safety, reentrancy, memory safety under invalid inputs, or dependency purity is established.

## Evidence

This contract is based solely on static inspection of the supplied representation of F-1f03ea16dfe5.

The evidence establishes one unconditional initial indirect call, one subsequent global read, position-sensitive conditional calls, and returns of either literal zero or the retained global sample. It provides no dependency bodies, domain definitions, concurrency contract, or runtime observations.

No tools, implementation access beyond the supplied material, runtime tests, or smoke executions were used. The acceptance examples are derived conditions, not observed test results.
