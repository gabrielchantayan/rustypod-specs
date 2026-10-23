<!-- Trusted provenance: F-c21c7dbaa861; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c21c7dbaa861

## Purpose

F-c21c7dbaa861 performs an ordered, two-object classification check and returns either a 32-bit zero or a 32-bit value read from shared storage.

The first argument must produce classification value 17. The second argument must produce a value other than 18, 19, or 10. Only when both conditions hold does the function read and return the shared value.

The classification for the **second argument is obtained first**. The classification for the **first argument is obtained second**. Both calls occur on every normally completing invocation, regardless of the classification returned by the first call.

The returned shared value is not necessarily nonzero. Consequently, a zero result does not distinguish a rejected classification pair from an accepted pair whose shared value is zero.

Interpreting this operation as a compatibility check is plausible intent, not an established semantic meaning for the classification numbers.

## Interface

### Inputs

The function accepts two object references, referred to here as the **first object** and **second object**.

The supplied representation treats each reference as pointing to storage whose initial field supplies a dispatch-table address. A callable entry is obtained from byte offset 28, hexadecimal 0x1C, within that table. The corresponding object reference is passed to the selected operation as its sole explicit argument.

Under the apparent 32-bit representation:

| Location | Required role |
|---|---|
| Initial four bytes of each object | Dispatch-table address |
| Four bytes at dispatch-table byte offset 28 | Callable entry address |
| Selected operation’s result | 32-bit classification value |

The apparent pointer and entry widths follow from the supplied representation; the original ABI and complete object layout are not independently established. The byte displacement of 28 and the use of the selected entry are directly visible.

### Output

The function returns a 32-bit value:

- Zero when the first object’s captured classification is not 17.
- Zero when the second object’s captured classification is 18, 19, or 10.
- Otherwise, the 32-bit value read from a designated shared storage location after both classification calls complete.

There is no separate status output, output parameter, or visible error-result encoding.

### Preconditions

Normal completion requires that:

- Both object references support the accesses performed when their respective dispatch occurs.
- Each dispatch-table reference is usable at the time it is read.
- Each selected callable entry can be invoked with the corresponding object reference.
- Each invoked operation returns normally with a usable 32-bit result.
- On the accepted branch, the shared storage location can be read.

The function does not visibly validate these conditions. In particular, a classification that will eventually cause rejection does not remove the requirement to perform the other object’s call.

The references are not visibly required to identify different objects. Aliasing is therefore not excluded by this function, although the invoked operations may impose additional requirements that are unknown here.

## Behavior

### Ordered observations

The function first accesses the second object’s dispatch information, obtains the callable entry at byte offset 28, and invokes that entry with the second object reference.

After this invocation returns, its classification result is retained for the final decision.

The function then accesses the first object’s dispatch information, obtains the entry at the same byte offset, and invokes that entry with the first object reference. Its returned classification is also retained.

This ordering is part of the observable contract:

1. Dispatch and invoke using the second object.
2. Receive the second object’s classification.
3. Dispatch and invoke using the first object.
4. Receive the first object’s classification.
5. Decide whether to return zero or read the shared value.
6. If accepted, read and return the shared value.

There is no visible classification-based early return between the two invocations. For example, a second-object classification of 18 still leads to invocation of the first object’s operation.

The first object’s dispatch information is obtained after the second object’s operation returns. It must not be modeled as an entry captured before that call. If the earlier operation changes dispatch information subsequently used for the first object, the later dispatch observes the resulting state, subject to validity and execution-environment constraints.

### Selection rule

The accepted classification pairs are exactly those satisfying both requirements:

- The first object’s captured result equals 17.
- The second object’s captured result equals none of 18, 19, and 10.

Every other normally obtained pair returns zero.

No range check accompanies these comparisons. Values outside the listed constants, including values that would be negative under a signed interpretation, are not rejected merely for being outside an expected classification range.

The comparisons use equality or inequality with small positive constants. For these tests, signed versus unsigned interpretation of the same 32-bit result does not change the outcome.

### Shared-value timing

The shared storage is read only on the accepted branch, after both object operations have completed.

The function does not visibly snapshot that storage before either invocation. If an invoked operation changes the shared value and the classification pair is accepted, the final read occurs after that change. No assumption that either dependency preserves shared state is warranted.

On a rejected pair, the function itself does not read this designated shared value. The invoked operations may independently access it; their behavior is not available.

### Repeated and aliased objects

Each argument causes a separate dispatch and invocation. Passing the same object as both arguments does not visibly collapse the calls or reuse the first result.

For an aliased object with stateful classification behavior, the second-argument role receives the first call’s result, and the first-argument role receives the second call’s result. The final decision uses those two captured results in their respective roles.

The function does not visibly request either classification again during the decision.

### Arithmetic and representation

The visible computation involves dispatch-address calculation using byte displacement 28, comparisons against 17, 18, 19, and 10, and selection of a 32-bit result.

There is no visible classification arithmetic, scaling, masking, narrowing, or aggregation. No overflow-handling policy for malformed dispatch addresses is established.

## Side effects

The function’s own visible operations consist of reading dispatch information, making two indirect calls, conditionally reading shared storage, and returning a value.

There is no visible direct write to either object or to shared storage. There is also no visible direct allocation, deallocation, reference-count operation, or ownership-transfer operation.

These observations do **not** establish that the complete invocation is side-effect free. Either indirect operation may mutate its object, another object, dispatch tables, shared storage, or other externally observable state. Either may also allocate or release memory. Their implementations are unavailable.

No ownership transfer can be inferred merely from passing an object reference. Conversely, the caller cannot infer that the references remain valid after an indirect call without an external dependency contract. If the first invocation invalidates storage needed for the second dispatch, this function has no visible mechanism to recover.

The supplied body contains no explicit synchronization. It establishes an order between its own calls and subsequent reads, but does not establish thread safety, atomic access guarantees, or coordination with concurrent writers.

## Dependencies

The following opaque IDs are documentary identifiers for unresolved **call sites**, not recovered identities of concrete implementations. The runtime entries may select the same implementation or different implementations.

| Opaque ID | Role | Known contract |
|---|---|---|
| F-indirect-01 | Second object’s dispatch-table entry at byte offset 28 | Invoked first, with the second object reference; supplies the second object’s captured 32-bit classification |
| F-indirect-02 | First object’s dispatch-table entry at byte offset 28 | Invoked second, with the first object reference; supplies the first object’s captured 32-bit classification |

For both dependencies, ownership behavior, mutation behavior, failure behavior, termination, and classification meaning are unknown.

The designated shared storage is an additional data dependency, not a function dependency. Its 32-bit contents supply the result on the accepted branch. Its initializer, writers, semantic type, and intended lifetime are not established.

## Acceptance examples

These are conditional specification examples derived from the supplied static evidence. They are not reports of executed tests. Unless stated otherwise, assume valid dispatches, normally returning dependencies, and valid shared storage.

| Second object’s captured result, obtained first | First object’s captured result, obtained second | Required result |
|---:|---:|---|
| 0 | 17 | Shared value read after both calls |
| 17 | 17 | Shared value read after both calls |
| 18 | 17 | Zero |
| 19 | 17 | Zero |
| 10 | 17 | Zero |
| 20 | 17 | Shared value read after both calls |
| 0 | 16 | Zero |
| 18 | 0 | Zero |

Additional acceptance conditions:

- **Zero-valued accepted result:** With classifications 0 for the second object and 17 for the first object, shared storage containing zero produces a zero return. The accepted branch still performs the shared read.
- **Unlisted bit pattern:** With second-object result 0xFFFFFFFF and first-object result 17, the pair is accepted. The second result matches none of the three exclusions.
- **No early rejection:** With second-object result 10, the first object’s operation must still be invoked. If it fails to return normally, no normal zero return is established.
- **Shared mutation:** If the first object’s operation changes shared storage to a particular 32-bit pattern before returning 17, and the previously captured second result is allowed, the function returns the value obtained by the subsequent shared read. In the absence of intervening writes, that is the newly stored pattern.
- **Aliased stateful object:** If both arguments identify one object, and its successive calls return 0 and then 17, the pair is accepted. Reversing those successive results yields rejection because the first-object role then has classification 0.
- **Rejected-branch read avoidance:** A first-object classification other than 17 causes a zero return without this function reading the designated shared storage, provided both calls completed normally.

## Uncertainties

The following remain **documented_only**:

- Original source-language types, calling convention, and independently verified pointer width.
- Concrete identities and complete contracts of the indirect dependencies.
- Whether the apparent dispatch layout represents a particular object system.
- Meanings of classifications 10, 17, 18, and 19.
- Whether the returned shared value denotes a flag, address-like token, handle, or another 32-bit quantity.
- Shared-storage initialization, ownership, mutability, and concurrency rules.
- Behavior after invalid references, invalid entries, dependency failures, or nonlocal control transfers.
- Whether externally imposed contracts forbid aliasing or require additional object state.

The supplied body has no visible recovery path for invalid accesses or failed dependency execution. That absence does not justify assigning a particular exception, signal, diagnostic, or fallback return.

## Evidence

Established static evidence consists of two ordered indirect invocations through dispatch-table byte offset 28, retention of both results, the exact classification predicate, a zero fallback, and a conditional 32-bit shared-storage read.

The apparent 32-bit dispatch layout is a representation-based inference. Classification semantics and the interpretation of the shared value remain unknown.

No runtime tests have been performed. No dependency implementation, caller behavior, shared-storage writer, or broader object definition was supplied.
