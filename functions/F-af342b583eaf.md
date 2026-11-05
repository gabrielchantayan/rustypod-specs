<!-- Trusted provenance: F-af342b583eaf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-af342b583eaf

## Purpose

F-af342b583eaf performs an operation on an embedded component, compares a 32-bit observation of that component before and after the operation, and issues either one or three dynamically dispatched notifications.

It then evaluates a separate completion condition. The function returns 1 only when an embedded-component predicate is nonzero and a subsequent object-level check returns zero. Otherwise, it returns 0.

The interpretation of the component operation, notification values, and completion checks is unknown. In particular, a changed observation does not by itself imply success or failure, and the return value is not directly determined by whether that observation changed.

## Interface

### Inputs

The function accepts two inputs:

- **Object:** an address used both as an object with a dispatch-table reference at its beginning and as the base of an embedded component.
- **Request:** a 32-bit integer-valued argument used in address-like arithmetic and passed unchanged to F-2403187d906c.

No null checks, bounds checks, object-type checks, or request validation are visible.

### Relevant layout

The supplied representation establishes the following accesses:

| Location | Use |
|---|---|
| Beginning of Object | Read to obtain the current dispatch-table reference. |
| Object + 76 bytes | Address of the embedded component passed to three dependencies. |
| Dispatch-table reference + 88 bytes | Read to obtain a callable notification target. |
| Request + 100 | First argument to an external dependency. |

The embedded-component offset follows from advancing nineteen 4-byte integer elements. The dispatch-slot displacement is a byte offset, not a count of table entries.

The header and dispatch-slot accesses are used as pointer-bearing accesses. The supplied representation is consistent with a 32-bit target, but the complete pointer ABI, calling convention, and table layout are not independently available.

### Outputs

The function returns a 32-bit value that is always either:

- **1:** the final component predicate was nonzero, and the object-level check was called and returned zero.
- **0:** the final component predicate was zero, or the object-level check returned nonzero.

A return value is reached only if all preceding operations return normally.

### Preconditions

For a defined, normally completing invocation:

- Object must support the header read, access to its embedded component, and all operations performed by the dependencies.
- Every dispatch-table reference used during the invocation must support a read at byte offset 88.
- Every callback target actually loaded must be callable with Object and two 32-bit arguments.
- The computation and use of Request + 100 must be valid under the actual target’s arithmetic and address conventions.
- Storage referenced by the function must remain usable across dependency calls.

These are requirements for safe execution, not conditions checked by this function. Dependencies may mutate object state or external state; their preservation and lifetime guarantees are unknown.

## Behavior

### Initial external operation

Before reading the component observation or resolving any notification callback, the function calls an external dependency with:

1. Request plus decimal 100.
2. The 32-bit value 2,147,483,647, whose hexadecimal representation is 0x7fffffff.

The external function’s return value does not affect subsequent control flow.

Its meaning is unknown. The second argument must not be described as a timeout, capacity, mask, or sentinel without additional evidence. Likewise, the address-like first argument does not establish that the external dependency reads rather than writes the referenced storage.

### Before-and-after observation

The function next performs these operations in order:

1. Calls F-7a393ce8e214 with the embedded-component address.
2. Retains the low 32 bits of that result as the initial observation.
3. Calls F-2403187d906c with the embedded-component address and the unchanged Request argument.
4. Calls F-7a393ce8e214 again with the same embedded-component address.
5. Compares the low 32 bits of the second result with the initial observation.

The second observation is represented as a 64-bit result, but only its low 32 bits influence behavior. Its upper 32 bits do not reach any notification argument or the final return value.

Consequently, two observations that differ only in their upper 32 bits select the unchanged branch. Signed interpretation does not alter this equality test: the relevant condition is equality of the two 32-bit patterns.

There is no visible check of the return value from F-2403187d906c. A normal return from that dependency always leads to the second observation.

### Notification parameter slots

The notification arguments come from six global 32-bit slots. Their values are not supplied. For this contract, they are identified by role:

| Slot | Role |
|---|---|
| **Common** | Third argument of the unchanged notification and the first changed notification. |
| **Unchanged** | Second argument of the unchanged notification. |
| **Changed-first** | Second argument of the first changed notification. |
| **Changed-middle** | Third argument of the second changed notification. |
| **Changed-shared** | Source for the second arguments of the second and third changed notifications. |
| **Changed-final** | Third argument of the third changed notification. |

These labels do not assert that the slots are immutable constants. No dependency is assumed to preserve them.

### Unchanged observation

When the two low-32-bit observations are equal, the function performs one notification:

1. Reads the Unchanged slot and retains its value.
2. Reads Object’s current dispatch-table reference and obtains the callback at byte offset 88.
3. Reads the Common slot.
4. Calls that resolved callback with Object, the retained Unchanged value, and the Common value.

The callback is therefore selected after the observation comparison and after the Unchanged slot read.

No Changed-first, Changed-middle, Changed-shared, or Changed-final notification value is needed on this branch.

### Changed observation

When the low-32-bit observations differ, the function performs three notifications.

**First notification**

The function resolves Object’s current callback at dispatch-table byte offset 88, then reads Common. It calls the resolved target with:

- Object;
- the value read from Changed-first for this call;
- the previously read Common value.

The relative evaluation order of individual call-expression operands is not established by the supplied representation. The explicit callback resolution and Common capture precede the call.

**Second notification**

After the first callback returns, the function reads Changed-shared and retains that value for the third notification.

It then makes a second notification using a newly resolved dispatch target, with:

- Object;
- a fresh read of Changed-shared;
- a read of Changed-middle.

The retained value for the third notification is distinct from the Changed-shared read used as the second notification’s argument. This distinction matters if the storage can change between those reads. The exact relative order of dispatch resolution and the argument-slot reads within this second call is not established.

**Third notification**

After the second callback returns, the function:

1. Resolves Object’s dispatch target again.
2. Reads Changed-final.
3. Calls the resolved target with Object, the Changed-shared value retained before the second notification, and the newly read Changed-final value.

Thus, a mutation of Changed-shared by the second callback does not replace the third notification’s retained second argument. A mutation of Changed-final by that callback can affect the third notification because Changed-final is read afterward.

The three notifications need not use the same callback target. The dispatch table is consulted again after each preceding callback returns.

### Final completion condition

After the selected notification sequence finishes, the function calls F-441b6e398b00 with the embedded-component address.

- If its 32-bit result is zero, F-af342b583eaf returns 0 without calling F-a967c7d12f06.
- If its result is nonzero, the function calls F-a967c7d12f06 with Object.
  - A zero result from this second check produces return value 1.
  - A nonzero result produces return value 0.

This check occurs after notifications. Notification-induced changes to the object or component can therefore affect the final checks.

## Side effects

The established direct effects are dependency calls and reads of object storage, dispatch storage, and global notification slots. No direct assignment to caller-owned object fields or notification slots is visible.

That does not make the function read-only. The external operation, component operation, observation functions, predicates, and callbacks may have side effects whose scope is not available.

The visible call order is:

1. External dependency.
2. Initial component observation.
3. Component operation.
4. Second component observation.
5. One or three notifications.
6. Final component predicate.
7. Conditional object-level check.

No explicit allocation, deallocation, ownership transfer, reference-count operation, or cleanup is visible. Passing an address to a dependency does not establish whether that dependency retains, releases, or merely borrows it.

The function does not visibly catch failures, undo changes, retry operations, or translate dependency errors. If a dependency fails to return normally, later calls and the normal return value are not guaranteed.

There is no visible synchronization establishing a consistent snapshot of notification slots or dispatch state. Some values are captured and retained, while others are reread after callbacks. Concurrent access rules remain unknown.

## Dependencies

| Opaque function ID | Invocation and established use |
|---|---|
| **F-7a393ce8e214** | Called twice with the embedded-component address. Low 32 bits of the results determine the notification branch. |
| **F-2403187d906c** | Called between the observations with the embedded-component address and Request. Its return value is not used. |
| **F-441b6e398b00** | Called after notifications with the embedded-component address. A zero result short-circuits the final condition. |
| **F-a967c7d12f06** | Called with Object only when F-441b6e398b00 returns nonzero. A zero result causes return value 1. |
| **F-000000000001** | Local opaque identifier for the unmapped external dependency called first with Request + 100 and 0x7fffffff. |
| **F-000000000002** | Local opaque identifier for the dynamically selected notification operation at dispatch-table byte offset 88. The actual target may differ between calls. |

The two locally assigned identifiers are descriptive handles, not resolved identities from the supplied dependency map.

No purity, global preservation, ownership behavior, or error semantics are assumed for these dependencies.

## Acceptance examples

These examples describe conditional expected behavior. They are not reports of executed tests.

### Equal observations, failed final component predicate

Given two observation results with identical low 32 bits, and a zero result from F-441b6e398b00:

- Exactly one notification occurs.
- Its two parameter values come from Unchanged and Common.
- F-a967c7d12f06 is not called.
- The function returns 0.

### Different upper halves only

Given an initial observation with low word 7 and upper word 1, and a second observation with low word 7 and upper word 9:

- The unchanged branch is selected.
- Only one notification occurs.
- The upper-word difference has no effect on the return condition.

### Changed observation with successful final condition

Given different low-32-bit observations, a nonzero result from F-441b6e398b00, and zero from F-a967c7d12f06:

- Three notifications occur in the specified order.
- Both final checks occur after the third notification.
- The function returns 1.

### Unchanged observation can still return 1

Given equal low-32-bit observations, a nonzero result from F-441b6e398b00, and zero from F-a967c7d12f06:

- One notification occurs.
- The function returns 1.

This demonstrates that observation change is not a prerequisite for return value 1.

### Callback changes the dispatch target

If the first changed-branch callback installs a different valid dispatch target:

- The second notification uses the subsequently resolved target, not the saved first target.
- The third target is resolved after the second callback and can differ again.

### Second callback changes global parameters

Suppose Changed-shared is read as 20 immediately after the first callback, and the second callback later changes it to 30 and changes Changed-final to 40.

Provided no other relevant changes intervene:

- The third notification’s second argument remains 20.
- Its third argument is 40.
- The second notification’s own second argument comes from its separate Changed-shared read.

### Nonzero object-level check

If the component predicate is nonzero but F-a967c7d12f06 returns any nonzero 32-bit value, the function returns 0. Neither result is required to be exactly 1.

## Uncertainties

The following remain **documented_only**:

- The semantic purpose of the external dependency and its two arguments.
- Whether Request denotes an address, handle, or another encoded value.
- The arithmetic behavior when Request + 100 exceeds the signed 32-bit range. Defined wrapping must not be assumed from this representation alone.
- The meaning and mutability rules of the six notification slots.
- The interpretation of either half of the component observation result.
- Dependency side effects, ownership rules, retained references, error signaling, and lifetime guarantees.
- The complete object layout, pointer ABI, and callback calling convention.
- Evaluation order among operands contained within an individual indirect-call expression.
- Concurrency requirements and whether external mutation of shared storage is permitted.
- Behavior for invalid addresses, invalid dispatch targets, dependency nonreturn, or broken lifetime assumptions.

The before-and-after comparison suggests change detection as an intended role. That is inferred intent; only the low-32-bit comparison and its notification consequences are established.

## Evidence

This contract is based solely on static inspection of the supplied function representation and dependency map.

Established evidence includes the embedded-component and dispatch-slot displacements, dependency call order, two-way notification branch, repeated dispatch resolution, global-value capture points, low-32-bit comparison, short-circuited final check, and binary return values.

No dependency implementations, global values, surrounding type definitions, or runtime traces were supplied. No runtime tests have been performed.
