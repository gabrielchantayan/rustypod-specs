<!-- Trusted provenance: F-5d3cf13b6b5f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5d3cf13b6b5f

## Purpose

F-5d3cf13b6b5f accepts an object and a signed integer position, conditionally invokes an object-specific preliminary operation, resolves or validates the position, and may dispatch two indirect operations followed by a final dependency call. On ordinary completion, it returns either the resolved position or −1.

The established behavior is positional and state-dependent. A special maximum-positive integer can select the last position of a nonempty object. A resolved position of −1 suppresses the later indirect operations and final dependency call.

The operation’s domain meaning is not established. The access pattern is consistent with an indexed container operation, but neither element removal nor any particular ownership policy is proven. The preliminary branch can enter a nonreturning dependency instead of completing the ordinary path.

## Interface

### Inputs

- **Object reference:** A reference to storage accessed as a sequence of 32-bit integer-sized words. The function reads an indirect-dispatch-table address from byte offset 0, a signed quantity from byte offset 4, and a byte-sized flag derived from the word at byte offset 16.
- **Position:** A signed 32-bit integer, interpreted by signed comparisons and by the dependencies that receive it.

The significant position constants are:

| Value | Established role |
|---|---|
| 2,147,483,647 | Special selector when the quantity at object offset 4 is positive |
| −1 | Suppresses the post-validation operation sequence |
| 0 | Ordinary position value; also the failure result tested from certain dependencies |

The representation is expressed using the supplied source’s 32-bit integer model. Pointer width, calling convention, byte order, and exact machine behavior for overflowing arithmetic are not independently established.

### Object layout used by this function

| Byte offset | Access | Meaning established here |
|---|---|---|
| 0 | Read as a dispatch-table address | Locates indirect operations |
| 4 | Read as a signed 32-bit integer | Controls special-position resolution, a range comparison, and a range endpoint |
| 16 | Read as a word, then narrowed to a character-sized value | Controls whether the preliminary dependency is called |

Only the narrowed character value is tested at offset 16. A nonzero full word does not necessarily enable the preliminary branch: narrowing may discard its nonzero bits. Under ordinary 8-bit character semantics, the test is whether the least significant eight bits of the word are nonzero. This describes numeric narrowing, not an independently established byte-order-dependent memory access.

Two indirect entries are used:

- The entry at dispatch-table byte offset 184 receives an object reference, a start position, an end position, and an all-ones 32-bit value.
- The entry at dispatch-table byte offset 188 receives an object reference and an all-ones 32-bit value.

Their semantic names and complete ABI-level signatures are unknown.

### Output and termination

On ordinary return:

- Validation failure returns −1 immediately.
- Otherwise, the function returns the effective position used for the later branch.
- If that effective position is −1, no later indirect operation or final dependency call occurs.

A separate preliminary path calls F-feee9156a257, which the supplied static evidence identifies as nonreturning. Its termination mechanism is unknown: it may transfer control or terminate execution, but no specific mechanism is established.

### Preconditions

For an execution with defined memory accesses:

- The object must be readable at every accessed field.
- Any indirect operation that is reached must have an accessible dispatch table and a callable entry at the required offset.
- The object must remain usable across dependency calls wherever this function subsequently accesses it.
- Integer and address arithmetic must be meaningful under the target execution model.

The function performs no visible null-object check, storage-bounds check, dispatch-entry check, or overflow check. Whether a dependency checks position validity is governed by that dependency’s unknown contract.

## Behavior

### 1. Preliminary flag-controlled operation

The first object access determines whether the narrowed value at offset 16 is nonzero.

If it is zero, the preliminary dependency is skipped.

If it is nonzero, F-a583137ba13d is called with:

1. The object reference.
2. The original input position, before any special-position resolution or validation.

A zero result permits execution to continue. A nonzero result selects the nonreturning path:

1. Add 8 to that result.
2. Pass the adjusted value to F-c26c62a103c0.
3. Subtract 8 from F-c26c62a103c0’s result.
4. Pass that adjusted result to F-feee9156a257.

The two adjustments surround a dependency call and cannot be canceled algebraically: they apply to different values. No null check or validity check is visible between these operations.

F-feee9156a257 is identified as nonreturning. Consequently, the ordinary validation and dispatch sequence is not reached on this path under that contract. Behavior if it unexpectedly returns is not established.

F-a583137ba13d may change object or global state. Later reads must not be interpreted as necessarily observing the object’s entry state.

### 2. Resolve the special position or validate the ordinary position

After the preliminary stage, the function chooses between two routes.

**Special-position route:** If the input position equals 2,147,483,647 and the signed quantity currently read from object offset 4 is greater than zero, the effective position becomes the quantity read for the subtraction minus one. F-402eef64af83 is not called.

For an object whose quantity remains a stable positive value \(n\) throughout these reads, the effective position is \(n-1\). This subtraction stays within the signed 32-bit range.

**Validation route:** In every other case, F-402eef64af83 receives the object and the unchanged input position. Its result is tested only for zero:

- Zero causes an immediate return of −1.
- Any nonzero result permits execution to continue.

In particular, the maximum-positive selector does not automatically resolve for a zero or negative quantity; it is passed to F-402eef64af83 instead.

The validator’s result is a success/failure gate, not a replacement position. The effective position remains the input position on this route. Its exact acceptance rules are unknown, including whether it accepts −1 or other negative values.

### 3. Skip later work for an effective position of −1

After successful resolution or validation, the effective position is compared with −1.

If equal, the function returns −1 without:

- Calling either indirect dispatch entry.
- Calling F-d6032bc0598f.

This return value is indistinguishable from validation failure by the return value alone. Earlier calls or side effects may distinguish the paths.

### 4. Conditionally dispatch a trailing range

For an effective position other than −1, the function compares:

- The effective position plus one.
- A signed quantity read from object offset 4.

If the former is strictly less than the latter, the entry at dispatch-table offset 184 is invoked with these established arguments:

1. The object reference.
2. Effective position plus one.
3. A quantity read from object offset 4, minus one.
4. The 32-bit all-ones value.

Under a stable quantity \(n\), the numeric endpoints are therefore position \(+1\) and \(n-1\). Although these resemble inclusive endpoints, the callee’s actual endpoint convention is unknown.

The supplied call representation also contains several incoming register-derived values whose origins and semantic roles are not established. They do not justify defining additional public inputs to F-5d3cf13b6b5f or assigning meanings to additional callback arguments. The complete signature is `documented_only`.

The quantity used to construct the endpoint is read separately from the quantity used for the comparison. No atomic snapshot or synchronization is visible. Relative evaluation order among independent argument expressions and dispatch-target evaluation is not established by the supplied evidence.

If the strict comparison is false, this indirect call is skipped.

### 5. Dispatch the unconditional post-position operation

For every effective position other than −1, provided earlier calls return, the function next invokes the entry at dispatch-table offset 188 with:

1. The object reference.
2. The 32-bit all-ones value.

This occurs whether or not the offset-184 operation ran.

The dispatch table is accessed for this operation after any earlier offset-184 call has completed. The earlier callee cannot be assumed to preserve the table pointer, object quantity, or other state.

### 6. Invoke the final dependency and return

After the offset-188 operation returns, F-d6032bc0598f is called with the object reference and effective position.

Its return value is not used. If it returns normally, F-5d3cf13b6b5f returns the effective position.

Neither indirect operation’s return value is visibly used.

## Side effects

No direct assignment to an object field or global variable is visible in this function. That does not make the operation read-only: every reached dependency or indirect operation may have side effects.

The established call order on an ordinary successful path is:

1. F-a583137ba13d, only when the narrowed flag is nonzero.
2. F-402eef64af83, unless special-position resolution succeeds.
3. Dispatch entry at offset 184, only when the range comparison succeeds and the effective position is not −1.
4. Dispatch entry at offset 188, whenever the effective position is not −1.
5. F-d6032bc0598f, whenever the effective position is not −1 and preceding calls return.

The nonreturning alternative follows F-a583137ba13d with F-c26c62a103c0 and then F-feee9156a257.

There is no visible rollback, retry, explicit allocation, explicit release, or ownership transfer. Calls may nevertheless perform those actions internally. In particular:

- Passing an object reference does not establish borrowing, retention, or destruction semantics.
- The arithmetic adjustments around F-c26c62a103c0 do not prove a header layout or allocation boundary.
- No dependency is assumed to preserve globals.
- No dependency is assumed to preserve object state unless required for subsequent accesses to remain valid.

The function returns no distinct error code for an invalid object, invalid indirect target, arithmetic overflow, or a dependency’s nonlocal exit.

## Dependencies

| Opaque ID or dispatch entry | Established use | Unestablished properties |
|---|---|---|
| F-a583137ba13d | Receives object and original position when the narrowed flag is nonzero; zero continues, nonzero selects the special path | Domain meaning, side effects, result representation, ownership |
| F-c26c62a103c0 | Receives the preceding nonzero result plus 8; its result is subsequently reduced by 8 | Transformation semantics, validity requirements, ownership |
| F-feee9156a257 | Receives the adjusted result and is identified as nonreturning | Exit mechanism, cleanup, error payload meaning |
| F-402eef64af83 | Receives object and original position; zero rejects, nonzero permits continuation | Validity rules and state changes |
| Dispatch offset 184 | Conditionally receives object, two computed positions, and all-ones value | Range convention, operation meaning, complete ABI signature |
| Dispatch offset 188 | Receives object and all-ones value after the optional range operation | Operation meaning and side effects |
| F-d6032bc0598f | Receives object and effective position after both dispatch decisions | Operation meaning, ownership, return-value meaning |

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless stated otherwise, assume valid storage, stable object fields, arithmetic within range, and dependencies that return normally.

1. **Empty trailing range after special resolution.**  
   With a zero narrowed flag, quantity 5, and input position 2,147,483,647, the effective position is 4. Validation is skipped. The range comparison is \(5 < 5\), so offset 184 is skipped. Offset 188 receives the object and all-ones value, F-d6032bc0598f receives position 4, and the function returns 4.

2. **Ordinary position with trailing positions.**  
   With quantity 5, input position 1, and a nonzero validation result, offset 184 receives numeric endpoints 2 and 4 plus the all-ones value. Offset 188 then runs, followed by F-d6032bc0598f with position 1. The return is 1.

3. **Ordinary last position.**  
   With quantity 5, position 4, and successful validation, offset 184 is skipped because \(5 < 5\) is false. Offset 188 and F-d6032bc0598f still run. The return is 4.

4. **Validation failure.**  
   When validation is reached and returns zero, the function returns −1. Neither indirect entry nor F-d6032bc0598f is called afterward. Any earlier preliminary side effects remain possible.

5. **Accepted −1.**  
   If F-402eef64af83 accepts input −1, the function returns −1 without either indirect call or F-d6032bc0598f. This is a successful gate followed by the sentinel skip, not necessarily validation failure.

6. **Maximum-positive selector on an empty object.**  
   With quantity 0 and the maximum-positive input, special resolution does not occur. Validation receives that maximum-positive input. A zero validation result returns −1; a nonzero result reaches arithmetic whose overflow semantics are not established here.

7. **Preliminary nonreturning branch.**  
   With a nonzero narrowed flag, suppose F-a583137ba13d returns an address-like value \(a\), and F-c26c62a103c0 returns \(b\) when given \(a+8\). F-feee9156a257 is then called with \(b-8\). The ordinary path is not reached.

8. **Accepted negative position other than −1.**  
   With quantity 3, position −2, and successful validation, the visible arithmetic permits offset 184 to receive endpoints −1 and 2. This does not establish that such a position is semantically valid; no separate nonnegative-position check is visible here.

## Uncertainties

The following remain `documented_only`:

- Domain meaning of the object, its quantity, and every dependency.
- Whether the quantity is a count, bound, or another signed measure.
- Ownership, lifetime, allocation, release, and retention behavior.
- Complete indirect-call signatures and register-derived argument roles.
- Target pointer width, byte order, character width, and exact machine arithmetic.
- Synchronization requirements or permitted concurrent mutation.
- Nonreturning behavior’s precise mechanism.
- Callee behavior for invalid positions or malformed objects.

Signed arithmetic needs particular care. Position plus one can exceed the signed 32-bit range if validation accepts 2,147,483,647. Quantity minus one can exceed that range if evaluated for the minimum signed quantity. The visible function supplies no overflow handling. Likewise, adding or subtracting 8 on address-like dependency results has no established bounds guarantee. Wrapping, trapping, or other outcomes must not be invented.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Established observations are the field offsets, constants, branches, comparisons, call order, argument adjustments, unused return values, and ordinary return values.

Container semantics, range inclusivity, ownership, and cleanup intent are not established. The nonreturning designation is supplied static evidence, not an observed execution result.

No runtime tests have been performed.
