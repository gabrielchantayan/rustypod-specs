<!-- Trusted provenance: F-241dff0e9fd7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-241dff0e9fd7

## Purpose

F-241dff0e9fd7 selects a nonzero starting value through an ordered sequence of dependency calls, optionally extends it across successive numeric values, finalizes the resulting endpoint, and publishes the starting value through an input/output word.

On success, the return value is the number of values accepted into the sequence, including its starting value. On failure, the function returns zero. Failure does not imply that context state or dependency-managed state remains unchanged.

An allocation or chain-extension role is inferred from the call structure. The supplied evidence does not establish what the numeric values identify, what resources they represent, or whether any dependency allocates, reserves, links, or releases memory.

## Interface

The interface consists of four arguments and a 32-bit unsigned return value:

| Argument | Representation | Established use |
|---|---|---|
| Context | 32-bit address value | Base address for context-field accesses; passed to every dependency. |
| Start word | Pointer to a readable and, on success, writable 32-bit unsigned word | Read once on entry. Its saved value controls validation, search order, and a possible final dependency call. On success, overwritten with the selected starting value. |
| Requested count | 32-bit unsigned integer | Upper bound on the number of values accepted by the extension loop, but not a requirement to return that many. |
| Fourth argument | 32-bit signed integer | Copied into temporary storage, then overwritten before that storage is passed to a dependency. No subsequent decision uses its incoming value. |

The following context fields are directly accessed. Offsets are byte offsets from the context base:

| Name used here | Offset | Width | Direct role |
|---|---:|---:|---|
| Limit | `0x7c` | 32 bits, unsigned | Entry validation, search arguments, extension bound, and final hint-update condition. |
| Earlier boundary | `0x1ec` | 32 bits, unsigned | Search argument and conditional-search boundary. |
| Search hint | `0x1f0` | 32 bits, unsigned | Search argument and ordering boundary; may be advanced after endpoint finalization. |
| Accounting word | `0x1f4` | 32 bits | Tested against zero and, if nonzero, reduced by the successful count. |

These names describe observed uses, not established domain meanings.

The start-word pointer must support the entry read. Any executed context access must reference valid storage, and any executed dependency must satisfy that dependency’s own preconditions. The function performs no explicit null-pointer, address-range, or alignment validation.

A return value of zero is the only failure indication provided by this interface. The function does not return a distinct error code for validation failure, search exhaustion, extension failure, endpoint-finalization failure, or final-link failure.

## Behavior

### Entry and validation

The function first reads and saves the start word. Later decisions use this saved value even if a dependency changes the memory addressed by the start-word pointer.

Entry behavior depends on the saved value:

- **Zero:** accepted as a request to select a starting value without an existing start.
- **One:** rejected immediately with return value zero. This path does not read any context field or call a dependency.
- **Two or greater:** the function reads Limit and rejects the request if the saved value exceeds it.

After successful validation, the function reads Limit again. For a zero saved start, this read supplies the upper argument for the first search call. For a nonzero saved start, the initial search obtains its upper argument through another Limit read.

Consequently, the entry-validation limit and a later search limit need not be the same if the underlying storage changes between reads. There is no single cached context snapshot governing the whole operation.

### Ordered candidate search

F-85c5eeab8703 is the candidate-search dependency. Each call receives the context and two 32-bit values. A zero result means that this function proceeds to another search attempt. A nonzero result becomes the selected starting value and immediately ends candidate searching.

The source establishes the order and arguments of these calls. It does not establish whether the two numeric arguments denote an inclusive range, an exclusive range, a directional search, or another domain-specific relationship.

#### Saved start is zero

The search order is:

1. Call F-85c5eeab8703 with Search hint and the Limit value read after validation.
2. If the result is zero, call it with Earlier boundary and a Search hint value read immediately after the first search call.
3. If the result is still zero, call it with `2` and a freshly read Earlier boundary.
4. If the result is still zero, call it with `2` and a freshly read Limit.

Search hint is read immediately after the first search call even when that call succeeds. Thus, a successful first search still has a subsequent context read before extension begins.

Earlier boundary is not subject to a comparison before the second attempt on this zero-start path.

#### Saved start is nonzero

The search begins with F-85c5eeab8703 called using the saved start and a freshly read Limit.

If this call fails:

1. Read Search hint and compare it unsigned with the saved start.
2. If Search hint is less than or equal to the saved start:
   - Attempt a search using a newly read Search hint as the first numeric argument and the saved start as the second.
   - Immediately after this call, read Search hint again, including when the call succeeds.
   - If the call fails, retain that post-call Search hint value as the upper argument for a possible Earlier boundary search.
3. If Search hint was greater than the saved start, omit that Search hint search. The upper argument for a possible Earlier boundary search remains the saved start.
4. If no candidate has yet been found, read Earlier boundary and compare it unsigned with the saved start.
   - Only if Earlier boundary is strictly less than the saved start, attempt a search using a newly read Earlier boundary and the upper argument selected above.
5. If there is still no candidate, search using `2` and a freshly read Earlier boundary.
6. If that fails, search using `2` and a freshly read Limit.

The comparison reads and the argument reads are separate accesses. In particular, a Search hint or Earlier boundary used as a search argument is not necessarily the value previously used to decide whether that search should run.

If every executed search returns zero, the function returns zero without entering extension.

### Candidate acceptance and extension

Once a search returns a nonzero value:

- That result is retained as the selected starting value.
- The current endpoint is set to that starting value.
- The accepted count is set to one.
- The temporary status word is set to zero.

The selected starting value is not independently checked against `2`, Limit, or either boundary. A nonzero dependency result is sufficient to enter this stage.

Extension continues only while both conditions hold:

1. The accepted count is less than Requested count.
2. The current endpoint is less than the currently read Limit.

The second condition is evaluated only when the first is true.

For each attempted extension:

1. Call F-c9f1dc723c52 with the context, the next numeric value, and a pointer to the temporary status word.
2. If that dependency returns zero, return zero immediately.
3. Otherwise, inspect the temporary status word.
   - If nonzero, stop extending without accepting the next value.
   - If zero, call F-2b2cf4a4fe41 with the current endpoint and the next numeric value.
4. If F-2b2cf4a4fe41 returns zero, return zero immediately.
5. Otherwise, increase the accepted count by one and advance the endpoint by one.

The temporary status word is initialized once before the loop, not before every dependency call. Its address refers to temporary storage within this invocation. Any retention of that address beyond its valid lifetime is not supported by this function’s interface.

A successful status check that reports a nonzero status is a normal stopping condition, not an immediate failure. Endpoint finalization still follows.

Requested count values of zero and one both skip extension. Either can nevertheless produce a successful return value of one.

### Endpoint finalization and publication

After extension stops, call F-3fb99b56c861 with the context and current endpoint.

If it returns zero, return zero immediately.

If it succeeds:

1. Read Limit.
2. If the endpoint is below Limit, read Search hint.
3. If Search hint is less than or equal to the endpoint, write endpoint plus one to Search hint.
4. If the saved entry start is nonzero, call F-2b2cf4a4fe41 with that saved entry start and the selected starting value.
   - A zero result returns failure immediately.
   - A zero saved entry start omits this call.
5. Write the selected starting value to the start word.
6. Read the Accounting word and test whether it is nonzero.
7. If nonzero, read it again and write back that second value minus the accepted count.
8. Return the accepted count.

There is no equality special case for the final F-2b2cf4a4fe41 call. If the saved entry start equals the selected starting value, the dependency is still called with equal numeric arguments.

Successful completion may return fewer values than requested because the endpoint reached Limit or the status dependency reported a nonzero status.

### Widths and arithmetic

Numeric comparisons involving starts, boundaries, endpoints, counts, and Limit are unsigned 32-bit comparisons.

The Accounting word is tested through a signed 32-bit view, but the predicate is only equality with zero. Its high bit therefore does not change whether the subtraction occurs. The resulting subtraction has unsigned 32-bit behavior: underflow wraps modulo \(2^{32}\). There is no saturation or insufficient-accounting check.

The extension loop computes endpoint plus one only after establishing that the endpoint is below a 32-bit Limit. Consequently, that addition cannot overflow during an executed extension. The Search hint update has the same protection.

The accepted-count increment also cannot overflow on an executed iteration: it occurs only after the count was strictly below the 32-bit Requested count.

## Side effects

The direct writes occur in this order, when their conditions are satisfied:

1. Possible Search hint advancement after successful endpoint finalization.
2. Start-word publication after any required final link succeeds.
3. Possible Accounting word reduction.

Writes performed by dependencies may occur earlier or between these direct writes. Their scope is not established.

There is no rollback path. For example:

- Earlier successful extension calls remain observable if a later extension fails.
- Endpoint finalization may have succeeded before the final link fails.
- Search hint may already have been advanced when the final link returns zero.
- A dependency may modify the start-word storage even when this function never reaches its own publication write.

No direct heap allocation, memory release, or ownership transfer is shown. Such behavior inside dependencies is unknown.

No non-aliasing requirement is enforced. If the start word aliases a context field, publication can affect subsequent behavior. In particular, if it aliases the Accounting word, the published starting value becomes the value tested and potentially reduced during accounting. If it aliases Search hint, publication can overwrite an earlier Search hint advancement.

The function provides no synchronization or atomic transaction guarantee.

## Dependencies

| Opaque ID | Established call contract within this function |
|---|---|
| F-85c5eeab8703 | Receives context and two 32-bit search arguments. Zero continues searching; nonzero selects the starting value. |
| F-c9f1dc723c52 | Receives context, the next numeric value, and a temporary status-word address. Zero causes failure. A nonzero result permits the function to inspect the status word. |
| F-2b2cf4a4fe41 | Receives context and two numeric values. Used both between adjacent extension values and, conditionally, between the saved entry start and selected start. Zero causes failure. |
| F-3fb99b56c861 | Receives context and the final endpoint. Zero causes failure; nonzero permits hint advancement and publication. |

These dependencies must not be assumed to preserve context fields, globals, aliased storage, or external resources. Their names here do not establish resource ownership or hidden side effects.

## Acceptance examples

The following are conditional acceptance criteria derived from the supplied static evidence, not results of executed tests. Unless stated otherwise, context fields remain stable during dependency calls and the start word does not alias them.

1. **Immediate rejection of one.**  
   Given an entry start of one, return zero without reading context fields, calling dependencies, or directly rewriting the start word.

2. **Out-of-range existing start.**  
   Given an entry start of twelve and an entry-validation Limit of eleven, return zero before candidate searching.

3. **Zero requested count still permits one accepted value.**  
   Given an entry start of zero, Requested count zero, and a first search result of seven, perform no extension checks. If endpoint finalization succeeds, publish seven and return one. With Limit twenty, Search hint four, and Accounting word ten, the direct final state is Search hint eight and Accounting word nine.

4. **Nonzero status stops extension successfully.**  
   Given a selected start of seven, Requested count four, and Limit twenty, suppose the first status call for eight returns nonzero and writes a nonzero status. Do not call the adjacent-value dependency for seven and eight. If finalization and any required final link succeed, publish seven and return one.

5. **Extension failure prevents direct publication.**  
   Given a selected start of seven, suppose the status call for eight succeeds with zero status but the adjacent-value call returns zero. Return zero without endpoint finalization, without this function’s start-word publication, and without its Accounting word subtraction. Dependency effects are not rolled back.

6. **Final-link failure can follow a hint write.**  
   Given an entry start of four, selected start six, endpoint seven, Limit ten, and Search hint five, successful endpoint finalization advances Search hint to eight. If the final call using four and six then returns zero, return zero without publishing six or subtracting accounting. The hint advancement remains.

7. **Accounting underflow wraps.**  
   Given a successful accepted count of three and a nonzero Accounting word of one, the final Accounting word is `0xfffffffe`, provided no aliasing or intervening modification changes the accounting reads.

8. **A candidate above Limit is not locally rejected.**  
   If a search returns nine while the later Limit is five, extension is skipped and hint advancement is omitted. If endpoint finalization and any required final link succeed, the function can still publish nine and return one.

## Uncertainties

**Inferred intent:** The structure resembles selecting and extending a chain of resource identifiers, with a search hint and an available-resource count. Those meanings are not established contracts.

**Documented only:** The dependency implementations, numeric search semantics, resource representation, ownership rules, and hidden effects are unavailable.

**Documented only:** The supplied evidence does not establish which context states callers normally maintain, whether start-word aliasing is intended, or whether external synchronization is required.

**Documented only:** A dependency’s success result does not establish that it preserves unrelated fields, and a failure result does not establish that it made no changes.

**Documented only:** Concurrent modifications and invalid memory accesses have no defined handling in the supplied material. This contract describes the explicit access order, not a concurrency guarantee.

## Evidence

The contract is based solely on the supplied static function body and opaque dependency mapping. Established observations include branch conditions, dependency-call ordering, field offsets, direct reads and writes, short-circuit behavior, return paths, and 32-bit arithmetic.

Domain terminology is explicitly separated from those observations. No dependency implementation or runtime behavior was available. No runtime tests have been performed.
