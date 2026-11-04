<!-- Trusted provenance: F-cadadf0e8ea2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cadadf0e8ea2

## Purpose

F-cadadf0e8ea2 conditionally prepares an object-associated operation, resolves a requested signed position, and invokes an ordered sequence of indirect and direct dependencies. It returns the effective position, returns −1 when a prerequisite rejects the request, or enters a dependency path identified as non-returning.

**Established static behavior:** the function has a distinguished request value of 2,147,483,647, a distinguished effective position of −1, an optional preliminary check, and two indirect dispatch slots. The dependency calls and their order are visible; their application-level meanings are not.

**Inferred intent:** the distinguished maximum request may select the last position in a nonempty collection. The remaining calls may adjust or remove an indexed element and an associated trailing range. These interpretations are not contractual descriptions of the unknown dependencies.

## Interface

### Inputs and result

The visible interface accepts:

- An object address, represented as a pointer to 32-bit integer storage.
- A signed 32-bit requested position.

On an ordinary return, the result is a signed 32-bit integer:

- −1 if F-402eef64af83 returns zero.
- Otherwise, the effective position, which is either the original requested position or the result of the special maximum-request rule.

A return value of −1 is ambiguous: it can represent prerequisite rejection or successful passage through the prerequisite with an effective position of −1. The return value alone does not distinguish these cases.

No null-pointer check, object-size check, dispatch-table check, or explicit arithmetic-overflow check is visible.

### Object storage accessed directly

The supplied representation establishes these byte offsets from the object address:

| Offset | Width | Visible use |
|---|---:|---|
| 0 | 32 bits | Address of an indirect dispatch table |
| 4 | 32 bits | Signed value used in positivity checks, bounds comparisons, and position calculations |
| 40 | 32 bits, narrowed for the test | Converted to an 8-bit character value and compared with zero |

The field at offset 4 is called the **extent field** below. This is a behavioral label, not proof that the value is a valid size or count.

The offset-40 condition depends on the narrowed character value, not on whether the entire 32-bit word is nonzero. Character signedness does not affect a comparison against zero. The representation does not independently establish that this operation is a physical byte load at offset 40; narrowing a word and loading a particular byte should not be treated as interchangeable without target-layout evidence.

The dispatch table is accessed at byte offsets 0xB8 and 0xBC. Both slots must contain callable targets whenever their corresponding paths are reached.

### Preconditions and ownership

The caller must supply storage readable at every directly accessed field and a usable dispatch table on paths that invoke it. Additional requirements imposed by dependencies are unknown.

The function contains no explicit allocation, deallocation, ownership-transfer operation, or direct write to an object field. This does **not** establish that the dependencies preserve the object, retain its lifetime, or leave global state unchanged. Safe execution of later accesses requires that earlier dependencies leave the necessary storage and dispatch state usable.

## Behavior

### 1. Optional preliminary check

The function first reads the offset-40 value and tests its narrowed character value.

- If that value is zero, it skips F-7aa7337e68ee.
- If it is nonzero, it calls F-7aa7337e68ee with the object and the original requested position.

If F-7aa7337e68ee returns zero, processing continues.

If it returns nonzero, the function calls F-7dd55568ef00 with no explicit arguments. If that call returns, it then calls F-feee9156a257 with no explicit arguments. The supplied representation marks the latter call as non-returning.

On this marked non-returning path, no position resolution, prerequisite call, indirect dispatch, or ordinary result follows. If F-7dd55568ef00 itself does not return, the subsequent call is not reached. If F-feee9156a257 unexpectedly returns, behavior beyond that violated non-return assumption is **documented_only**, not a supported continuation of this contract.

The preliminary dependency receives the original request, including 2,147,483,647 or −1 when either was supplied. Position resolution has not yet occurred.

### 2. Resolve the effective position

After the preliminary stage, the function considers the original request and the then-current extent field.

If both conditions hold:

- The requested position is exactly 2,147,483,647.
- The extent field is strictly greater than zero under signed comparison.

then the effective position becomes the extent field minus one, and F-402eef64af83 is not called.

For this branch, a stable positive signed 32-bit extent lies from 1 through 2,147,483,647. Subtracting one therefore produces a representable effective position from 0 through 2,147,483,646.

In every other case, the function calls F-402eef64af83 with the object and the original requested position.

- A zero result causes an immediate return of −1.
- Any nonzero result permits processing to continue with the original requested position as the effective position.

The special rule does not apply to an empty or negative extent. A maximum request with such an extent is passed unchanged to F-402eef64af83.

There is no visible range validation after a nonzero prerequisite result. In particular, the function itself does not reject negative effective positions other than treating −1 specially.

### 3. Handle the effective −1 sentinel

If the effective position is −1, the function returns −1 without invoking either indirect dispatch slot or F-d6032bc0598f.

This does not bypass the earlier preliminary stage or F-402eef64af83. An original request of −1 reaches this sentinel handling only after the prerequisite returns nonzero.

### 4. Optionally dispatch a trailing range

For any effective position other than −1, the function compares the effective position plus one with the then-current extent field, using signed comparison.

If the incremented position is strictly less than that extent, the function invokes the target found in dispatch slot 0xB8. Its first four visible argument values are:

1. The object address.
2. The effective position plus one.
3. The extent field minus one.
4. The 32-bit all-ones value, 0xFFFFFFFF.

The third argument is based on an extent read for the call; it is not established as a snapshot shared with an earlier stage.

The supplied representation also shows four additional incoming machine-state words passed to this target. They are not initialized or assigned semantic roles within this function. Their values, types, and significance are **documented_only**. A behavioral substitute must not invent meaningful defaults for them, and the representation alone does not establish that they are four intentional application-level parameters.

If the signed comparison is false, the 0xB8 dispatch is omitted.

### 5. Dispatch the unconditional follow-on operation

For every effective position other than −1, the function next invokes the target found in dispatch slot 0xBC, whether or not slot 0xB8 was invoked.

The visible arguments are:

1. The object address.
2. The 32-bit all-ones value, 0xFFFFFFFF.

The dispatch-table address is read for this invocation. The function does not establish that a preceding dependency preserved the table pointer or its entries.

### 6. Invoke the final dependency and return

After the 0xBC invocation returns, the function calls F-d6032bc0598f with the object and the effective position.

If that call returns normally, the function returns the effective position. No visible result from either indirect invocation or F-d6032bc0598f is checked or used to change that result.

## Side effects

The visible body performs reads and dependency calls but no explicit object-field writes.

The observable call order on a continuing path is:

1. F-7aa7337e68ee, only when enabled by the narrowed offset-40 value.
2. F-402eef64af83, unless the positive-extent maximum-request rule applies.
3. Dispatch slot 0xB8, only when the trailing-range comparison succeeds.
4. Dispatch slot 0xBC, for an effective position other than −1.
5. F-d6032bc0598f, for an effective position other than −1.

The preliminary nonzero-result path instead invokes F-7dd55568ef00 and then, if control returns, F-feee9156a257.

Dependencies may change object fields, dispatch entries, globals, or memory ownership; no preservation guarantee is established. Later direct reads occur after earlier calls, so a specification must not replace the observed accesses with one entry-time snapshot of the object.

No rollback, exception handler, retry, or recovery protocol is visible. Effects completed by an earlier dependency are not visibly undone when a later dependency fails to return normally.

## Dependencies

| Opaque dependency or dispatch location | Established role |
|---|---|
| F-7aa7337e68ee | Optional preliminary decision using the object and original request; zero continues, nonzero selects the non-returning path |
| F-7dd55568ef00 | First no-explicit-argument call on that path |
| F-feee9156a257 | Subsequent no-explicit-argument call, marked non-returning |
| F-402eef64af83 | Prerequisite using the object and original request; zero returns −1, nonzero continues |
| Dispatch slot 0xB8 | Conditional invocation using the object, trailing-range endpoints, an all-ones word, and additional unresolved machine-state words |
| Dispatch slot 0xBC | Follow-on invocation using the object and an all-ones word |
| F-d6032bc0598f | Final invocation using the object and effective position |

The concrete identities of the indirect targets are unknown. No opaque function IDs for those targets are supplied.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, assume readable storage, stable stated field values, a preliminary stage that permits continuation, usable dispatch targets, and normal return from continuing dependencies.

1. **Maximum request with extent 5.**  
   A request of 2,147,483,647 resolves to effective position 4 without calling F-402eef64af83. The comparison of 5 against extent 5 is false, so slot 0xB8 is omitted. Slot 0xBC receives the object and 0xFFFFFFFF; F-d6032bc0598f receives the object and 4. The result is 4.

2. **Interior position with extent 5.**  
   A request of 1, accepted by F-402eef64af83, invokes slot 0xB8 with visible leading arguments comprising the object, 2, 4, and 0xFFFFFFFF. Slot 0xBC and F-d6032bc0598f follow in that order. The result is 1.

3. **Prerequisite rejection.**  
   For any request that reaches F-402eef64af83, its zero result causes an immediate return of −1. Neither indirect slot nor F-d6032bc0598f is invoked.

4. **Accepted −1 request.**  
   With original request −1 and a nonzero prerequisite result, the function returns −1 without either indirect invocation or the final dependency. This has the same result value as rejection but a different prerequisite outcome.

5. **Maximum request with zero extent.**  
   F-402eef64af83 receives 2,147,483,647 unchanged. If it rejects, the function returns −1. If it accepts, the subsequent increment is outside the signed 32-bit range; a defined ordinary outcome is not established by this representation.

6. **Accepted negative position other than −1.**  
   With request −2, extent 5, and prerequisite acceptance, the trailing comparison succeeds. Slot 0xB8 receives leading argument values comprising the object, −1, 4, and 0xFFFFFFFF. The function adds no independent rejection of this request.

7. **Preliminary nonzero result.**  
   If the narrowed offset-40 value is nonzero and F-7aa7337e68ee returns nonzero, the function enters the two-call non-returning sequence. The extent-resolution and ordinary dispatch stages are not reached under the stated non-return contract.

## Uncertainties

- **Arithmetic:** incrementing effective position 2,147,483,647 is not representable as signed 32-bit arithmetic. Subtracting one from an extent of −2,147,483,648 is likewise not representable if that expression is reached. Wrapping, trapping, and other target-specific outcomes are not established.
- **Repeated reads:** extent reads are not a transactional snapshot. Concurrent modification, volatile access semantics, and synchronization requirements are unknown.
- **Indirect-call interface:** the significance of the additional machine-state words shown at slot 0xB8 is unresolved.
- **Dependency effects:** allocation, release, lifetime changes, global mutation, exceptions, and nonlocal control transfer are not established.
- **Domain meaning:** collection removal, range adjustment, validation, and error reporting are possible interpretations, not proven dependency semantics.
- **Error reporting:** no ordinary error channel beyond the ambiguous −1 result is visible.

These items remain **documented_only** rather than being resolved by assumptions.

## Evidence

This contract is derived solely from the supplied static function representation and dependency-ID mapping. That evidence establishes visible branches, field offsets, argument values, call ordering, signed comparisons, and return expressions.

The non-returning status of F-feee9156a257 is an annotation in the supplied representation, not runtime confirmation. The representation is not independent proof of every reconstructed machine-level type or indirect-call signature.

No implementation access, dependency inspection, or runtime testing was performed. The acceptance examples describe consequences conditional on the stated inputs and dependency behavior; they are not observed execution results.
