<!-- Trusted provenance: F-63acd5195fcc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-63acd5195fcc

## Purpose

F-63acd5195fcc conditionally resolves a 32-bit value and stores it through a caller-supplied output address.

Resolution requires both:

1. The supplied selector equals a shared selector word.
2. At least one of two ordered indirect queries returns a nonzero result.

The first query has priority. Its nonzero result selects one shared candidate word and suppresses the second query. Otherwise, the second query may select a different shared candidate word. A selected candidate is passed through two direct dependencies before the resulting 32-bit value is written to the output.

The function returns a 32-bit status: one after the output store, or zero when the selection conditions are not met. This status does not establish the semantic validity, nonzero value, or ownership of the stored result.

The structure suggests a prioritized capability or property lookup. That interpretation is inferred intent; the meanings of the selector, queries, candidate words, and resolved value are not established.

## Interface

### Inputs

The function has three inputs:

| Input | Representation | Role |
|---|---|---|
| Context address | 32-bit integer-shaped address in the supplied representation | Provides access to an object-pointer field at byte offset 0x888. |
| Selector | 32-bit integer | Compared for equality with a shared selector word. |
| Output address | Address of a 32-bit writable unit | Receives the result only on the successful resolution path. |

The interface does not establish whether any input is a higher-level object, handle, enumeration, or resource identifier.

Equality against the shared selector is a full-width comparison. No range check, masking, normalization, or conversion to a narrower selector is visible.

### Referenced layout

When the selector matches, the function uses this pointer chain:

- At context byte offset 0x888, a field supplies an object address.
- The first pointer-sized field at that object address supplies a dispatch-table address.
- A callable entry at dispatch-table byte offset 0x1ac supplies the first query.
- A callable entry at dispatch-table byte offset 0x1b0 supplies the second query.

The supplied representation is consistent with 32-bit addresses and four-byte dispatch entries. Under that layout, the two entries are adjacent. Their byte offsets, rather than a presumed class or table type, define the established layout.

No validation of the context address, object address, dispatch-table address, or callable entries is visible.

### Return and output

The return value is a 32-bit word with these normal outcomes:

- **One:** a candidate was selected, both direct dependencies returned normally, and the function performed its final 32-bit output store.
- **Zero:** the selector did not match, or both indirect query results were zero.

A zero return does not cause a direct write to the output destination by this function. Dependencies reached before that return may nevertheless have side effects, including changes to aliased memory.

On the successful path, the output is the 32-bit return value of F-ea9442e8ddbb. Zero is a permissible stored value: there is no result-value check before returning one.

### Preconditions

For defined normal execution along a matching-selector path, every pointer-chain access and invoked callable must be valid for its use. The output destination must be writable if execution reaches the final store.

The function does not directly dereference the context or output address on the selector-mismatch path. On a matching-selector path that ends with both queries returning zero, it does not directly dereference the output address.

These conditional access properties are not a general guarantee that invalid arguments are safe: indirect dependencies have undocumented behavior and calling conventions.

## Behavior

The ordering below describes the supplied static representation. It does not assert volatile, atomic, or instruction-level memory-access guarantees.

### 1. Compare the selector

The function compares the selector input with the shared selector word.

If they differ, it returns zero immediately. On this path, it performs no dispatch-chain access, invokes no dependency, and makes no output store.

If they match, it proceeds to the first indirect query. There is no later recheck of the selector word. Changes to that word during a dependency call do not introduce another comparison in this invocation.

### 2. Invoke the first query and capture its candidate

The function obtains the first query through the context’s object field and the dispatch entry at byte offset 0x1ac, then invokes it.

After that query returns normally, the function reads the first shared candidate word. This read precedes the decision based on the query’s result in the supplied representation. It therefore occurs even when the first query returns zero and the first candidate will not be used.

The query’s return is tested as a 32-bit integer:

- Any nonzero value selects the captured first candidate.
- Zero causes evaluation of the second query.

There is no requirement that an affirmative query return exactly one. Negative nonzero values also satisfy the test.

If the first query returns nonzero, the second query is not invoked, and the second shared candidate word is not read by this function.

### 3. If necessary, invoke the second query and capture its candidate

When the first query returns zero, the function resolves the dispatch chain again to obtain the callable at byte offset 0x1b0.

The second lookup is not specified as reuse of a previously captured object or dispatch-table pointer. Changes made by the first query to the context’s object field or to the relevant dispatch structure can therefore affect the second invocation.

After the second query returns normally, the function reads the second shared candidate word. As with the first candidate, this read occurs before the result test in the supplied representation, including when that result is zero.

The second query’s result is tested for nonzero:

- A nonzero result selects the captured second candidate.
- A zero result causes the function to return zero without calling either direct dependency or performing its own output store.

Thus, a first-query zero result does not itself terminate the search. A second-query zero result does.

### 4. Resolve the selected candidate

Once a candidate has been selected, the function invokes F-9d0ecc5f6adf with no explicit arguments shown in the supplied representation.

It then invokes F-ea9442e8ddbb with two 32-bit values, in this order:

1. The return value of F-9d0ecc5f6adf.
2. The selected candidate captured after the successful indirect query.

The candidate is not reread from shared storage between these calls. If F-9d0ecc5f6adf changes the shared candidate word, the already-captured value remains the argument to F-ea9442e8ddbb.

No condition is imposed on the return value of F-9d0ecc5f6adf before it is forwarded. In particular, a zero return is not treated as a local failure.

### 5. Store and report success

After F-ea9442e8ddbb returns normally, the function stores its 32-bit return value at the supplied output address. It then returns one.

There is no local validation, interpretation, or cleanup of that value. The store also occurs if the value is zero.

If a dependency fails to return normally, the supplied representation establishes no recovery path, fallback result, compensating action, or guaranteed final store.

### Widths and address arithmetic

The status, candidate words, dependency return values, and output unit are 32-bit quantities in the supplied representation.

Address traversal uses fixed byte offsets. There is no visible count-based calculation, narrowing conversion, or multiplication. The source does not establish portable address-overflow semantics for the context-plus-offset operation. Valid address computation and accessible storage are preconditions; arithmetic wrapping is not a supported fallback.

## Side effects

The function’s explicit write is one 32-bit store to the output destination on the successful path. It performs no explicit store on either normal zero-return path.

Its explicit reads depend on the path:

| Path | Reads after the selector comparison |
|---|---|
| Selector mismatch | None |
| First query nonzero | First dispatch traversal and first candidate |
| First query zero, second query nonzero | Both dispatch traversals and both candidates |
| Both queries zero | Both dispatch traversals and both candidates |

Dependency calls are additional observable effects. Their behavior is not constrained by this function’s body. They may mutate shared state, modify reachable objects, allocate or release resources, or affect memory that aliases the output destination. No preservation or ownership guarantee can be inferred merely because this function contains no corresponding operation.

The final output store occurs after all successful-path dependency calls. If those calls modify the destination, this function’s final store replaces the four bytes at that destination, provided execution reaches the store normally.

No output initialization occurs before the queries. Consequently, a zero return means “no direct output assignment by this function,” not necessarily “the caller’s output bytes are unchanged.”

No allocation, release, lock operation, atomic operation, or explicit error-reporting operation appears directly in this function. Equivalent effects inside dependencies remain unknown.

No non-aliasing restriction is established. An output destination that overlaps shared state or part of the context structure can be modified by the final store. Subsequent calls could observe that modification.

## Dependencies

### F-9d0ecc5f6adf

Called exactly once on a normally completing resolution path, after candidate selection and before F-ea9442e8ddbb.

No explicit arguments are shown. Its 32-bit return value becomes the first argument to F-ea9442e8ddbb.

Its semantic purpose, global-state effects, ownership behavior, failure modes, and requirements are documented_only.

### F-ea9442e8ddbb

Called exactly once after F-9d0ecc5f6adf returns normally.

It receives the preceding dependency’s return value and the captured candidate. Its 32-bit return value becomes the output word without further examination.

Whether it performs lookup, conversion, allocation, retention, or another operation is documented_only.

### Indirect query dependencies

The first and second queries are selected through dispatch-table byte offsets 0x1ac and 0x1b0, respectively. Their identities are not supplied as opaque function IDs.

The call expressions show no explicit arguments. This does not establish the absence of an implicit receiver, register-provided context, or other calling-convention inputs.

Only the zero/nonzero interpretation of each returned 32-bit result is established. Their other semantics and side effects are documented_only.

## Acceptance examples

These examples are conditional static expectations, not results of executed tests. They assume valid accesses and normally returning dependencies unless stated otherwise.

1. **Selector mismatch:** The input selector differs from the shared selector word. The function returns zero without dispatching either query, reading either candidate, invoking either direct dependency, or storing an output.

2. **First query wins:** The selector matches and the first query returns seven. The function captures the first candidate after that query returns, skips the second query, calls F-9d0ecc5f6adf, then calls F-ea9442e8ddbb with that return and the captured candidate. It stores the latter call’s return and returns one.

3. **Negative query result:** The first query returns a negative 32-bit value. Because the value is nonzero, it selects the first candidate exactly as a positive nonzero result would.

4. **Second query wins:** The first query returns zero and the second returns one. Both candidate reads occur in their respective post-query positions, but only the second captured candidate is forwarded for resolution.

5. **Neither query succeeds:** Both queries return zero. The function returns zero without invoking the direct dependencies or performing its own output store. This expectation does not assert that the indirect queries left the output destination unchanged.

6. **Zero resolved value:** A query succeeds and F-ea9442e8ddbb returns zero. The function writes a zero word and returns one.

7. **Candidate changed during a query:** The first query changes the first shared candidate before returning nonzero. The subsequently read candidate is the value selected. There is no pre-query candidate snapshot.

8. **Candidate changed during resolution:** After selection, F-9d0ecc5f6adf changes the selected candidate’s shared storage. F-ea9442e8ddbb still receives the previously captured word.

9. **Dispatch target changed by the first query:** The first query returns zero after changing the context’s object field. The second dispatch lookup uses the field as subsequently evaluated, rather than requiring the original object.

10. **Unusable output address on a zero-return path:** If the selector mismatches, or both validly invoked queries return zero, this function performs no direct access through the output address. If resolution succeeds, the final store requires a valid writable destination; no guarded error return is provided.

## Uncertainties

The following are documented_only:

- The semantic names and valid domains of the selector and candidate words.
- Whether the context and dispatch structure are formal objects under a particular ABI.
- Hidden arguments, receiver handling, and calling conventions of the indirect queries.
- The identities and internal behavior of those queries.
- The meaning, lifetime, and ownership of the final 32-bit value.
- Global preservation, allocation, release, and other effects of all dependencies.
- Behavior following invalid memory accesses, invalid call targets, exceptions, traps, or nonreturning calls.
- Synchronization and concurrent mutation semantics.
- Physical memory-read counts and optimizer effects where the representation does not establish volatile or atomic accesses.

The function’s structure supports describing ordered fallback selection. It does not support assigning domain-specific meanings to that selection or promising thread safety.

## Evidence

Established static evidence consists of the supplied function body and opaque dependency mapping: the selector gate, fixed-offset dispatch chains, ordered zero/nonzero query tests, candidate reads after query returns, two-call resolution sequence, final 32-bit store, and constant status returns.

The contract distinguishes those visible operations from inferred intent and undocumented dependency behavior. It does not assume that callees preserve globals or own, retain, or release memory.

No runtime tests have been performed. The acceptance examples are conditional consequences of the supplied static representation, not observed executions.
