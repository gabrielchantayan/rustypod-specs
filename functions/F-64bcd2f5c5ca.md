<!-- Trusted provenance: F-64bcd2f5c5ca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-64bcd2f5c5ca

## Purpose

F-64bcd2f5c5ca conditionally performs receiver-dispatched operations associated with a requested position and returns either the effective position or the rejection value −1.

The visible behavior supports three principal outcomes:

- A preliminary receiver-dependent check enters a path identified as non-returning.
- A position-checking dependency returns zero, causing an immediate return of −1.
- The function selects an effective position, optionally invokes a range operation, invokes a second receiver operation and a final helper, then returns the effective position.

The value 2,147,483,647 has special treatment: when a receiver field at byte offset 4 is positive, this request selects one less than that field without invoking the ordinary position-checking dependency.

**Inferred intent:** the receiver field at offset 4 resembles an element count, and the dispatched operations resemble modification of an indexed collection. Neither the collection interpretation nor a particular operation such as deletion, truncation, or notification is established. The behavioral contract therefore uses “position,” “bound field,” and “range operation” without assigning undocumented semantics.

## Interface

### Inputs

The function receives:

1. **Receiver:** a pointer to an object whose visible layout supports the accesses below.
2. **Requested position:** a signed integer represented as 32 bits in the supplied representation.

The same representation shows a signed 32-bit return value and 32-bit object words. These widths are the basis of this contract; the underlying ABI is not independently supplied.

### Receiver layout

| Byte offset | Visible interpretation | Required use |
|---|---|---|
| 0 | Dispatch-table address | Used to resolve receiver-specific dependencies. |
| 4 | Signed 32-bit bound field | Used for special-position resolution and range decisions. |
| 16 | Word converted to a character-sized value | Its low character-sized portion controls the preliminary check. |

The conditional flag is not a test of the entire word at offset 16. On the apparent eight-bit-character target, only its low eight bits affect the zero/nonzero result. Character signedness does not change that result.

Three dispatch-table entries are used:

| Table byte offset | Explicit meaningful inputs | Visible result use |
|---|---|---|
| 64 | Receiver, original requested position | Returns an address whose first signed 32-bit word is inspected. |
| 184 | Receiver, effective position plus one, current bound minus one, all-one 32-bit value, four additional opaque values | No return value is consumed. |
| 188 | Receiver, all-one 32-bit value | No return value is consumed. |

The all-one 32-bit value is the bit pattern `0xFFFFFFFF`; interpreted as signed 32-bit, it is −1. Its semantic meaning to either dispatched dependency is unknown.

### Output

On a normal return, the result is:

- −1 if F-402eef64af83 returned zero; or
- The effective position selected by this function.

An accepted effective position of −1 also returns −1. Consequently, the return value alone does not distinguish dependency rejection from acceptance of that position.

### Preconditions

There are no visible null-pointer, alignment, object-size, dispatch-entry, or returned-pointer checks.

For any path that uses them, the receiver, its referenced dispatch table, and the selected dispatch entries must remain valid. If the preliminary dispatch occurs, its returned address must permit reading a 32-bit word.

Dependencies that return normally must leave the receiver usable for subsequent accesses on that path. This is a requirement for normal continuation, not an established ownership or preservation guarantee.

Arithmetic must remain within the supported signed-integer semantics. The supplied representation does not establish wrapping behavior for signed overflow.

## Behavior

### 1. Preliminary receiver-dependent check

The function first examines the character-sized conversion of the receiver word at offset 16.

If that value is zero, it skips this entire preliminary check.

If it is nonzero:

1. It resolves the receiver dispatch entry at table offset 64.
2. It calls that dependency with the receiver and the **original requested position**.
3. After that dependency returns, it reads the first 32-bit word at the returned address.
4. If the word is zero, processing continues.
5. If the word is nonzero, it calls F-8ad439e5daa6, followed—if the first call returns—by F-feee9156a257.

The supplied material identifies F-feee9156a257 as non-returning. Under that identified behavior, the nonzero-word path has no normal integer result and does not proceed to position resolution.

The mechanism of non-return is not established. It must not be described specifically as throwing, terminating the process, aborting, or transferring control to a particular handler.

This preliminary dispatch happens before special treatment of 2,147,483,647. The dispatched dependency therefore receives that sentinel unchanged when it is the original request.

### 2. Effective-position selection

If preliminary processing permits continuation, the function chooses between two paths.

**Special-position path**

When both conditions hold:

- The requested position is 2,147,483,647.
- The receiver’s signed bound field at offset 4 is greater than zero.

The effective position becomes the bound field minus one.

F-402eef64af83 is not called on this path.

**Ordinary checking path**

For every other case, the function calls F-402eef64af83 with the receiver and the requested position.

- A zero result causes an immediate return of −1.
- Any nonzero result permits continuation with the requested position unchanged.

The dependency’s result is tested only for zero versus nonzero. It does not replace the position. A negative nonzero result therefore permits continuation just as a positive result does.

A request of 2,147,483,647 with a zero or negative bound takes this ordinary checking path. The sentinel is not unconditionally translated.

### 3. Effective position −1

If the effective position is −1, the function returns −1 without invoking:

- The dispatch entry at offset 184.
- The dispatch entry at offset 188.
- F-d6032bc0598f.

The preliminary check may already have run. F-402eef64af83 has also run if this position reached the ordinary checking path.

This is not a universal no-side-effect return: earlier dependencies may already have changed state.

### 4. Optional range operation

For an effective position other than −1, the function compares:

- Effective position plus one.
- The receiver’s bound field as read for this comparison.

If the former is strictly less than the latter, it invokes the dispatch entry at offset 184 with:

- The receiver.
- Effective position plus one.
- The bound field minus one.
- The all-one 32-bit value.
- Four additional opaque values shown in the supplied call representation.

The two calculated position arguments are consistent with a candidate inclusive interval from the following position through the last position. Inclusive-range semantics, however, belong to the unknown dispatched dependency and are not established here.

If effective position plus one equals the bound, the range operation is skipped. If it exceeds the bound, the range operation is also skipped.

The four opaque trailing values are not initialized or assigned by the visible function body. Their meaning, provenance, and status as genuine interface arguments are unknown. They must not be treated as zero, stable constants, owned pointers, or documented options.

### 5. Mandatory completion calls for positions other than −1

After the optional range operation returns, or immediately when it is skipped, the function:

1. Resolves and calls the receiver dispatch entry at offset 188 with the receiver and the all-one 32-bit value.
2. If that call returns, calls F-d6032bc0598f with the receiver and effective position.
3. If that call returns, returns the effective position.

The range operation’s result is not inspected. Neither is the result of the offset-188 dispatch or F-d6032bc0598f.

There is no visible success-result check, compensating action, rollback, or conversion of dependency failures into a status value.

## Side effects

### Direct effects

The visible body performs receiver-field reads, a read through the preliminary dependency’s returned address when applicable, dispatch resolution, and dependency calls.

It contains no explicit write to receiver fields, global storage, or the preliminary returned object. It performs no explicit allocation or release.

These observations do not establish that the operation as a whole is read-only or allocation-free. Dependencies may mutate state, allocate, release resources, or fail to return.

### Ordering and state visibility

The established call order is:

1. Preliminary offset-64 dispatch, when enabled.
2. F-8ad439e5daa6 and F-feee9156a257, only for a nonzero returned word.
3. F-402eef64af83, only on the ordinary checking path.
4. Offset-184 dispatch, only when the range condition holds.
5. Offset-188 dispatch, for every continuing effective position other than −1.
6. F-d6032bc0598f, after the offset-188 dispatch returns.

Earlier dependencies can affect later behavior:

- The bound used for special-position selection is accessed after the preliminary dispatch returns.
- The bound used for the range decision is accessed after F-402eef64af83 returns, if that helper was called.
- Dispatch entries are resolved at their respective call sites. The function does not visibly cache one table address for the whole operation.
- The offset-188 dispatch occurs after the optional range dependency returns and may therefore resolve against changed receiver state.

No dependency is assumed to preserve globals, receiver fields, or dispatch-table contents.

The supplied representation establishes these phase boundaries, but not every machine-level load count or the evaluation order among arguments to a single call. Concurrent mutation and reentrant mutation within an expression have no independently established synchronization contract.

### Ownership

The preliminary returned address is dereferenced once at the visible level. There is no visible release or ownership-transfer operation for it.

That does not establish whether it is borrowed, newly allocated, shared, or released indirectly elsewhere. Likewise, the receiver’s ownership is not transferred explicitly, but dependencies’ ownership behavior remains unknown.

## Dependencies

Only the following named dependencies have supplied opaque IDs:

| Dependency | Visible contract at this call site |
|---|---|
| F-402eef64af83 | Receives receiver and requested position. Zero causes immediate rejection; nonzero permits continuation without changing the effective position. Other effects are unknown. |
| F-8ad439e5daa6 | Called after a nonzero preliminary result word. No explicit arguments are shown. Its result is not consumed. It may itself fail to return. |
| F-feee9156a257 | Called after F-8ad439e5daa6 returns on that same path. No explicit arguments are shown. Identified by the supplied material as non-returning; mechanism unknown. |
| F-d6032bc0598f | Receives receiver and effective position after the offset-188 dispatch returns. Its result is not consumed. |

The receiver-dispatched dependencies have no supplied opaque function IDs. They are identified solely by table offsets 64, 184, and 188. Their internal behavior is documented_only.

## Acceptance examples

These examples are conditional static acceptance criteria, not results of executed tests. Unless stated otherwise, all invoked dependencies return normally, required memory is valid, state remains stable between relevant reads, and arithmetic is representable.

### Positive-bound sentinel

Given a bound of 5 and a request of 2,147,483,647:

- The effective position is 4.
- F-402eef64af83 is not called.
- The range operation is skipped because 5 is not less than 5.
- The offset-188 dispatch runs before F-d6032bc0598f.
- The normal result is 4.

If the preliminary flag is enabled, its dispatch receives 2,147,483,647, not 4.

### Interior accepted position

Given bound 5, request 1, and a nonzero result from F-402eef64af83:

- The range operation receives calculated positions 2 and 4.
- It also receives the all-one value and four opaque trailing values.
- The offset-188 dispatch follows it.
- F-d6032bc0598f then receives position 1.
- The normal result is 1.

### Last accepted position

Given bound 5, request 4, and a nonzero checking result:

- The range operation is skipped.
- The offset-188 dispatch and final helper still run.
- The normal result is 4.

### Dependency rejection

Given any request routed through F-402eef64af83, if that dependency returns zero:

- The function returns −1 immediately.
- Neither completion dispatch nor F-d6032bc0598f runs.
- Any earlier preliminary effects remain possible.

### Accepted −1

Given request −1 and a nonzero checking result:

- The normal result is −1.
- Neither range nor completion dispatch runs.
- F-d6032bc0598f does not run.

This result is indistinguishable by return value from rejection.

### Negative accepted position other than −1

Given bound 5, request −2, and a nonzero checking result:

- The function does not independently reject the negative position.
- The range condition compares −1 with 5 and succeeds.
- The range arguments include −1 and 4.
- Completion proceeds and the normal result is −2.

Whether dependencies permit such an input is unknown.

### Preliminary non-returning path

Given an enabled preliminary flag and a nonzero first word at the preliminary returned address:

- F-8ad439e5daa6 is called.
- If it returns, F-feee9156a257 is called.
- Under the identified non-returning behavior, there is no normal result and no later position processing.

### Boundary arithmetic limitation

Given request 2,147,483,647 and a nonpositive bound:

- F-402eef64af83 is called.
- If it rejects, the result is safely determined as −1.
- If it accepts, later evaluation requires adding one to the maximum signed 32-bit value.

No normal result is specified for that overflowing case without additional machine-level arithmetic evidence.

## Uncertainties

The following are documented_only:

- The receiver’s concrete type and the semantic meaning of its bound field.
- The precise effects and failure behavior of every dependency.
- The semantics of the two all-one arguments.
- The offset-184 dependency’s range interpretation and trailing argument interface.
- Ownership and lifetime of the preliminary returned address.
- Whether the non-returning identification reflects exception transfer, termination, or another mechanism.
- Machine-level signed-overflow behavior and any wider intermediate arithmetic.
- Synchronization, concurrent access, and reentrancy guarantees.
- Behavior after invalid memory access or a dependency violating continuation preconditions.

The apparent collection-modification interpretation is inferred intent, not an acceptance requirement.

## Evidence

This contract derives from the supplied static function representation and dependency-ID mapping.

Established static evidence includes the field offsets, branch conditions, sentinel handling, zero/nonzero result tests, calculated arguments, dependency-call ordering, visible lack of direct stores, and normal return expressions.

Representation-dependent evidence includes the apparent 32-bit integer model, character-sized flag conversion, the four opaque trailing call values, and the non-returning identification.

No implementation was executed. No runtime tests, ABI inspection, dependency-body review, or memory-ownership verification have been performed.
