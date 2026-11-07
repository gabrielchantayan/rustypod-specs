<!-- Trusted provenance: F-d0cc96f5a9d2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d0cc96f5a9d2

## Purpose

F-d0cc96f5a9d2 conditionally issues four ordered virtual operations through objects obtained from F-17fe47085641, then issues two ordered virtual operations on its input object. On normal completion, it returns the 32-bit value 1.

The conditional operations depend on a field of the input object and six operands derived from a global integer. The final two operations occur regardless of whether the conditional operations were selected.

The supplied static evidence establishes the call structure, argument relationships, and explicit reads. It does not establish the application-level meaning of the operations, the identities of the objects returned by the dependency, or the effects of the virtual methods.

## Interface

### Input object

The function accepts one object address.

The supplied representation uses 32-bit integer elements and a virtual-table layout consistent with a 32-bit target:

| Location | Interpretation visible to this function |
|---|---|
| Input object, byte offset 0 | Address of a virtual dispatch table |
| Input object, byte offset 148 (`0x94`) | 32-bit branch-selection field |
| Input object's virtual table, byte offset 88 (`0x58`) | Callable entry used for the final two operations |

The selector is compared with the integer values 0 and 1. Other values can bypass the conditional operations. These are exact comparisons, not a general false/true interpretation.

The function does not receive an object size or validate the address, selector location, table address, or callable entries. A successful execution therefore requires readable object storage and valid dispatch targets wherever the selected path dereferences them.

### Global inputs

Four distinct global locations are visible:

- **Base global:** a 32-bit integer used to form six operands for the conditional operations.
- **Primary final-argument global:** a 32-bit value both saved for later use and read separately for the first final operation.
- **First secondary-argument global:** a 32-bit value supplied to the first final operation.
- **Second secondary-argument global:** a 32-bit value supplied to the second final operation.

These names describe uses within this function, not application-level types or meanings. In particular, the final-argument globals could contain integers, handles, or address-like bit patterns; the evidence does not resolve that question.

### Dependency interface

F-17fe47085641 is called without explicit arguments. Its result is treated as an object address:

- The first word is interpreted as a virtual-table address.
- The entry at byte offset 244 (`0xf4`) is invoked.
- The invocation receives the returned object as receiver, followed by two 32-bit operands.

Each selected conditional path makes four separate calls to F-17fe47085641. The function does not reuse a previously returned object in place of any of those calls.

### Result

On normal completion, the function returns the 32-bit value 1.

There is no explicit alternative status value, validation failure return, or propagation of a dependency's return value. A constant success-like return must not be interpreted as evidence that the invoked operations succeeded.

## Behavior

### 1. Prepare the conditional operands

Before inspecting the selector, the function prepares six values from the base global. In the supplied representation, the reads and computations appear in this order:

1. Read the base global without adjustment.
2. Read the base global and subtract 3.
3. Read the base global and subtract 5.
4. Read the base global and subtract 1.
5. Read the base global and subtract 2.
6. Read the base global and subtract 4.

The resulting values are retained for use after dependency calls begin. In particular, the conditional operations do not explicitly reread the base global after calling F-17fe47085641 or one of its returned objects.

These six expressions reference the global separately. The evidence does not justify replacing them with an atomic snapshot or assuming they all observe the same value when external mutation is possible. Conversely, the supplied representation alone does not prove six distinct physical memory loads or establish concurrent-memory semantics.

When the base remains stable at a value \(B\), the prepared values correspond to \(B\), \(B-3\), \(B-5\), \(B-1\), \(B-2\), and \(B-4\).

### 2. Select a conditional path

The selector at input-object byte offset 148 is first compared with 0.

- If that comparison succeeds, the zero path is selected.
- Otherwise, the selector is compared with 1.
- If the second comparison succeeds, the one path is selected.
- If neither comparison succeeds, no conditional dependency operations occur.

The representation contains a second selector access for the comparison with 1. It does not establish that the selector is read once into a stable local snapshot.

Consequently, under a model in which the two accesses can observe different values, selection is governed by the observations at those comparisons: a nonzero first observation followed by 1 selects the one path. Whether such a change is supported by the actual execution environment is unknown.

### 3. Execute the zero path, if selected

The zero path performs four rounds. Every round obtains a receiver by calling F-17fe47085641 and then invokes that receiver's virtual entry at byte offset 244.

For a stable base value \(B\), the two explicit operands are:

| Round | First operand | Second operand |
|---|---:|---:|
| 1 | \(B-4\) | 0 |
| 2 | \(B-1\) | \(B-2\) |
| 3 | \(B-3\) | \(B-5\) |
| 4 | \(B\) | 0 |

The zero in the fourth round is explicitly selected by this path. It does not come from the prepared base-minus-two value.

The order is significant: each virtual operation precedes the next acquisition call. The function does not acquire all four receivers first and then perform the operations.

### 4. Execute the one path, if selected

The one path uses the same four-round acquisition-and-invocation structure.

For a stable base value \(B\), its explicit operand pairs are:

| Round | First operand | Second operand |
|---|---:|---:|
| 1 | \(B-4\) | \(B-5\) |
| 2 | \(B-3\) | 0 |
| 3 | \(B-1\) | 0 |
| 4 | \(B\) | \(B-2\) |

The fourth round uses the prepared base-minus-two value. It is not recomputed after the preceding dependency calls.

### 5. Perform the two final operations

After the selected conditional path completes, or immediately after bypassing both paths, the function performs the final operations.

First, it reads the primary final-argument global and retains that value.

It then invokes the input object's virtual entry at byte offset 88, with:

1. The input object as receiver.
2. A separate read of the primary final-argument global.
3. A read of the first secondary-argument global.

After that invocation returns, the function again dispatches through the input object's virtual entry at byte offset 88, with:

1. The input object as receiver.
2. The retained primary value captured before the first final invocation.
3. A read of the second secondary-argument global.

The two uses of the primary global have different provenance. The first final operation uses a fresh argument read; the second uses the earlier retained value. They agree if the global remains stable across those reads, but equality is not unconditional.

The second final invocation follows the first. Its secondary argument is therefore obtained after the first invocation returns. The first invocation may have changed that global.

The second dispatch is also expressed as a fresh lookup through the input object's table. The function does not explicitly retain the first final operation's callable entry for reuse. If the first invocation changes the object's dispatch state, that change can affect the second lookup, subject to the validity of the resulting object.

The supplied representation does not establish a complete machine-level ordering among the callable-entry lookup and argument reads within a single invocation.

### 6. Return

After both final operations return normally, the function returns 1. Return values from the virtual operations are not used.

If a dependency fails to return normally, the evidence does not establish that subsequent operations or the final return occur.

## Side effects

### Direct effects

The function explicitly reads:

- The base global while preparing the six conditional operands.
- The input object's selector.
- Dispatch-table addresses and callable entries for selected operations.
- The three final-argument globals, with the primary global used in two separate pre-call reads.

No explicit write to the input object's fields, the named globals, or separately allocated storage appears in the function.

The retained operands and retained primary final argument are local execution state. Their retention does not imply that the underlying global locations remain unchanged.

### Effects through calls

For either selected conditional path, the function performs:

- Four calls to F-17fe47085641.
- Four virtual invocations through the returned objects.
- Two virtual invocations through the input object.

When neither path is selected, it performs only the two input-object virtual invocations.

Any of these calls may have externally visible effects not described by the supplied evidence. The contract does not assume that they preserve globals, object fields, dispatch tables, or other shared state.

### Ownership and lifetime

No explicit allocation, release, ownership transfer, or reference-count operation is visible.

That absence does not establish the ownership policy of F-17fe47085641's results. The function neither visibly releases those results nor establishes whether acquisition itself retains, allocates, borrows, or returns shared storage.

Each acquired object must remain valid through its immediately following virtual invocation. The input object must remain valid through the final dispatches. In particular, behavior is not established if an earlier invocation invalidates storage needed by a later one.

### Errors

There are no explicit checks for:

- A null or invalid input address.
- A null or invalid dependency result.
- An invalid dispatch table or callable entry.
- Rejected operands.
- Failure statuses returned by invoked operations.

The evidence specifies no recovery, rollback, retry, cleanup-on-failure, or exception-handling behavior.

## Dependencies

### F-17fe47085641

Established use:

- Called without explicit arguments.
- Called four times on each selected conditional path.
- Each result is immediately used as the receiver for one virtual invocation at table byte offset 244.

Unknowns include receiver identity, ownership, allocation behavior, global mutation, failure behavior, and whether successive calls return the same object.

### Unidentified virtual dependencies

No opaque function IDs are supplied for the dynamically selected entries.

The observable dispatch interfaces are:

| Receiver | Table byte offset | Explicit operands after receiver |
|---|---:|---|
| Each F-17fe47085641 result | 244 | Two 32-bit values |
| Input object | 88 | Two 32-bit values |

These locations identify dispatch mechanisms, not fixed function identities. Different receivers or changed tables may select different implementations. Assigning application semantics or invented dependency IDs would exceed the evidence.

## Acceptance examples

These are conditional acceptance criteria derived from static inspection. They are not executed test results.

### Stable selector 0 and base 100

Assume valid objects and dispatch entries, normal returns, a stable selector of 0, and a stable base of 100.

The conditional virtual operations must receive these operand pairs, in order:

1. (96, 0)
2. (99, 98)
3. (97, 95)
4. (100, 0)

Each operation must be preceded by its own call to F-17fe47085641. Both final input-object operations must follow the fourth conditional operation. Normal completion returns 1.

### Stable selector 1 and base 100

Under the same validity and stability assumptions, with selector 1, the conditional pairs must be:

1. (96, 95)
2. (97, 0)
3. (99, 0)
4. (100, 98)

The fourth operation must use the previously prepared value 98 even if an earlier dependency call changes the base global.

### Stable selector outside the recognized values

With a stable selector of 2, or any other stable value unequal to both 0 and 1:

- F-17fe47085641 must not be called by this function.
- No conditional receiver operation must occur.
- Both final input-object operations must still occur.
- Normal completion must return 1.

The base preparation still precedes selector evaluation in the supplied representation.

### Retained primary argument

Assume the primary final-argument global is 700 for both reads preceding the first final invocation. Assume that invocation changes the global to 900.

The second final invocation must still receive 700 as its first explicit operand. It uses the retained value, not a post-invocation reread.

### Secondary argument changed by the first final operation

Assume the first final invocation changes the second secondary-argument global from 20 to 30.

Provided execution continues normally and no later change intervenes, the second final invocation must receive 30 as its second explicit operand.

### Boundary arithmetic

With a stable base of 5, all six prepared mathematical values are representable as signed 32-bit integers. The zero-path pairs are (1, 0), (4, 3), (2, 0), and (5, 0).

A base whose subtraction results fall outside the signed 32-bit range requires the unresolved arithmetic semantics described below; a wrapping-result assertion is not established by this example.

## Uncertainties

The following are **documented_only**, not guessed implementation guarantees:

- **Arithmetic overflow:** The representation uses integer subtraction, but does not independently establish instruction-level overflow semantics. Two's-complement wraparound may be intended; a portable signed-arithmetic wrapping contract is not established. Bases at or above \(-2^{31}+5\) keep all five subtractions within signed 32-bit range.
- **Physical read counts:** Repeated global and selector expressions establish distinct source-level accesses, not exact load counts, atomicity, volatile behavior, or synchronization.
- **Calling convention and semantic types:** The exact ABI and whether opaque 32-bit final arguments represent numbers, handles, or addresses are unknown.
- **Dependency behavior:** Side effects, errors, ownership, lifetime guarantees, and global preservation are unspecified.
- **Application intent:** The conditional pairs may express relationships among identifiers or resources, but neither those meanings nor an interpretation of zero as a null resource is established.
- **Failure behavior:** Invalid dereferences and non-returning or exceptional callees have no specified local handling.
- **Runtime validation:** No runtime tests have been performed.

## Evidence

The supplied static function representation supports:

- Six base-derived operand preparations before selector evaluation.
- Selector access at byte offset 148.
- Two recognized conditional paths, each containing four acquisition calls and four virtual invocations at table byte offset 244.
- The ordered argument pairs stated above.
- A bypass path for selectors matching neither comparison.
- Two subsequent input-object virtual invocations at table byte offset 88.
- A retained primary global value used by the second final invocation.
- An unconditional 32-bit return value of 1 after normal completion.

Resource meanings, ownership policies, overflow behavior, and callee effects remain outside the established evidence.
