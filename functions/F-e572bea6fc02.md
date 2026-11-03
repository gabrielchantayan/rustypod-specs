<!-- Trusted provenance: F-e572bea6fc02; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e572bea6fc02

## Purpose

F-e572bea6fc02 performs an ordered search for a nonzero candidate through F-85c5eeab8703, then submits the first such candidate to F-3fb99b56c861. It returns the candidate only when that second dependency also returns nonzero.

After successful submission, the function conditionally decrements a 32-bit field in the supplied context. Every ordinary failure path returns zero without directly updating that field.

The interpretation of the candidate, the meaning of successful submission, and the role of the decremented field are unknown. Search, validation, allocation, and resource-accounting terminology would be inferred intent rather than established behavior.

## Interface

### Inputs

The function receives two values:

- **Context address:** represented as an integer in the supplied listing and used as the base address for field accesses and as an argument to dependencies. It must identify storage suitable for every access reached during execution.
- **Requested value:** an unsigned integer used to select the initial dependency argument. Its comparisons are unsigned.

The listing identifies the accessed fields as four-byte values. This contract describes the request, dependency results, and relevant field arithmetic using the listing’s apparent 32-bit integer model. The actual calling convention, address width, and binary-level function declarations are not independently established.

### Context layout

The following locations are directly relevant:

| Byte offset from context | Access width | Interpretation in this function | Direct accesses |
|---|---:|---|---|
| `0x07C` | 4 bytes | Uninterpreted argument value for the third candidate attempt | Read only if the first two candidate attempts return zero |
| `0x1EC` | 4 bytes | Initially an unsigned bound; later an uninterpreted argument value | Read once during normalization and read again if the third attempt is reached |
| `0x1F4` | 4 bytes | Signed integer tested against zero and conditionally decremented | Read only after successful submission; written only on the nonzero branch |

The offset `0x1F4` is the same location expressed by decimal byte offset 500.

No structure definition, alignment guarantee, byte order, field name, or relationship among these fields is supplied. Other context locations may be accessed or changed by dependencies.

### Output

The ordinary return value is:

- **Zero** when all candidate attempts return zero, or when the first nonzero candidate is rejected by F-3fb99b56c861.
- **The exact nonzero candidate value** returned by F-85c5eeab8703 when F-3fb99b56c861 subsequently returns nonzero.

Nonzero values are tested for inequality with zero, not positivity. Negative candidate values and negative submission results therefore satisfy the visible success predicates.

No distinct error code identifies which failure occurred.

### Preconditions

For defined execution of the visible accesses:

- The context’s `0x1EC` field must be readable at entry.
- Additional fields must remain accessible whenever their corresponding branches are reached.
- F-85c5eeab8703 and F-3fb99b56c861 must accept the argument forms used here.
- Context-relative address calculations and the conditional decrement must be valid under the actual execution model.

There is no visible null check, range validation for the context address, exception handler, or recovery from invalid memory.

## Behavior

### 1. Normalize the requested value

The function first replaces any unsigned requested value below 2 with 2.

It then reads the four-byte field at context offset `0x1EC` and compares that field, interpreted as unsigned, with the adjusted request. If the field is less than or equal to the adjusted request, the effective request becomes 2. Otherwise, it remains the adjusted request.

Consequently, the effective request is either:

- The original request, when that request is at least 2 and strictly below the initially observed bound; or
- 2 in every other case.

This is a reset to 2, not a clamp to the bound or to one less than the bound. Bounds of 0, 1, or 2 do not prevent an effective request of 2. There is no second comparison to establish that the replacement value lies below the bound.

The normalization-stage read of `0x1EC` occurs before any dependency call. The resulting effective request is retained for subsequent calls; later changes to the bound do not trigger normalization again.

### 2. Attempt to obtain a candidate

The function invokes F-85c5eeab8703 in the following order, stopping at the first nonzero result.

| Attempt | Arguments visible in the supplied representation | Condition for execution |
|---|---|---|
| First | Context, effective request | Always after normalization |
| Second | Context, literal 2, effective request | First attempt returned zero |
| Third | Context, current four-byte value at `0x1EC`, current four-byte value at `0x07C` | First and second attempts returned zero |

The first attempt visibly supplies two arguments; the later attempts visibly supply three. The actual dependency prototype is unknown. This contract preserves that distinction rather than inventing a missing argument or declaring an argument unused.

The third attempt does not reuse the bound value read during normalization. Its field arguments are read after both preceding attempts have returned zero. Changes made by either earlier call can therefore affect the third attempt’s arguments.

Both third-attempt field values are obtained before that call is entered. The supplied representation does not establish the relative evaluation order of the reads at `0x1EC` and `0x07C`.

If all three results are zero, the function returns zero. It does not invoke F-3fb99b56c861 or directly access `0x1F4` on this path.

### 3. Submit the first nonzero candidate

Once a nonzero candidate has been obtained, the function invokes F-3fb99b56c861 with:

1. The context.
2. That candidate.

This dependency is invoked at most once.

If it returns zero, the function returns zero immediately. It does **not** resume candidate acquisition, try a later fallback, directly decrement `0x1F4`, or visibly release the rejected candidate.

If it returns nonzero, execution proceeds to the conditional decrement. The eventual return value is the saved candidate, not the submission result.

Dependency-induced changes to context fields do not cause the function to reconsider an earlier candidate or rerun normalization.

### 4. Conditionally decrement the field at `0x1F4`

After a nonzero submission result, the function reads `0x1F4` as a signed integer and tests it against zero.

- If the observed value is zero, the function makes no direct write to that field.
- If the observed value is nonzero, the supplied expression reads the field again, subtracts one from that second value, and writes the result to `0x1F4`.

The second read matters to the source-level access description: the condition’s value is not explicitly reused as the subtraction operand. For ordinary, stable memory without intervening modification, a nonzero value becomes one less than its prior value.

This is not a positive-only decrement. A negative value also enters the decrement branch. Under a 32-bit signed model, decrementing −2,147,483,648 exceeds the signed range. The supplied evidence does not establish wrapping, trapping, or another defined outcome for that case.

After this step, the function returns the saved nonzero candidate. The dependency may already have modified `0x1F4`; the visible decrement uses the value present after that dependency returns.

## Side effects

### Direct memory effects

The only explicit write in this function is the conditional four-byte write at `0x1F4`.

Its explicit access ordering is:

1. Read `0x1EC` during normalization.
2. Execute candidate dependency calls as needed.
3. If the third attempt is reached, read `0x1EC` and `0x07C` for that call, with their relative order unspecified.
4. If a candidate exists, invoke the submission dependency.
5. Only after a nonzero submission result, read `0x1F4`.
6. If that read is nonzero, read `0x1F4` again and write its decremented value.

This source-level description does not establish machine instruction counts, atomicity, synchronization, or memory-mapped I/O behavior.

### Indirect effects

Both dependencies may have effects not described by the supplied evidence. In particular, this function does not establish that they preserve:

- Context fields, including the three fields listed above.
- Global state.
- The availability or contents of referenced memory.
- Resources associated with a candidate.

No rollback is visible after any dependency call. Returning zero therefore does not imply that the call had no effects.

### Ownership and lifetime

No explicit allocation, release, reference transfer, or ownership annotation appears.

The function forwards a candidate to F-3fb99b56c861 and may subsequently return the same value. This does not prove that the value is a pointer, that it remains dereferenceable, or that ownership transfers to the caller.

If a dependency invalidates the context and execution later accesses it, the supplied function provides no visible protection. Lifetime guarantees must come from external contracts.

## Dependencies

### F-85c5eeab8703

Established interaction:

- Called between one and three times.
- Receives the argument forms listed above.
- A zero result permits the next attempt, if any.
- The first nonzero result becomes the sole candidate.
- Its result is treated as a value, without visible dereferencing or interpretation beyond the zero test.

Unknowns include its true signature, result domain, effects, error conventions, ownership rules, and whether the different argument forms represent intentional invocation modes.

### F-3fb99b56c861

Established interaction:

- Called exactly once if a nonzero candidate is obtained.
- Receives the context and that candidate.
- A zero result causes an ordinary zero return.
- A nonzero result permits the conditional decrement and return of the candidate.

Unknowns include what it accepts or rejects, whether it mutates the context, whether it consumes the candidate, and whether its result has meanings beyond zero versus nonzero.

## Acceptance examples

These are conditional contract examples derived from the supplied source. They are not runtime observations. Except where stated otherwise, assume dependencies return normally, leave the relevant fields stable, and do not invalidate storage.

### Request below the minimum

Given request 1 and an initial bound of 20:

- The effective request is 2.
- The first candidate attempt receives the context and 2.

The same effective request results from request 0.

### Request equal to the bound

Given request 8 and an initial bound of 8:

- Equality triggers replacement with 2.
- The first attempt receives 2, not 8 or 7.

Given request 7 with bound 8, the effective request remains 7.

### Small bound does not suppress attempts

Given request 9 and an initial bound of 0:

- The effective request becomes 2.
- The first candidate attempt still occurs.

No direct failure is produced merely because the bound is below 2.

### First candidate succeeds

Given first-attempt result 37, submission result 1, and `0x1F4` equal to 4 after submission:

- No second or third candidate attempt occurs.
- F-3fb99b56c861 receives candidate 37.
- `0x1F4` becomes 3.
- The function returns 37.

### Submission rejection does not cause fallback

Given first-attempt result 37 and submission result zero:

- Neither later candidate attempt occurs.
- The function returns zero.
- It performs no direct access to `0x1F4`.

Effects already performed by either dependency remain outside any visible rollback mechanism.

### Third attempt observes changed fields

Given zero results from the first two attempts, and suppose those calls leave `0x1EC` equal to 12 and `0x07C` equal to 99:

- The third attempt receives the context, 12, and 99.
- Its second argument need not equal the bound observed during normalization.
- Its third argument is not the retained effective request merely because the earlier calls used that request.

### Zero and negative counter values

Given a nonzero candidate and nonzero submission result:

- If `0x1F4` is zero, the function returns the candidate without a direct counter write.
- If `0x1F4` is −3 and remains stable, it becomes −4.
- If it is the minimum signed 32-bit value, no portable decrement outcome is established here.

### Nonzero does not mean positive

Given candidate −7, submission result −1, and `0x1F4` equal to 1:

- Both nonzero predicates succeed.
- The field becomes zero.
- The function returns −7.

### Exhausted candidate attempts

Given three zero candidate results:

- All three candidate attempts occur in order.
- F-3fb99b56c861 is not called.
- The function returns zero without directly reading or writing `0x1F4`.

## Uncertainties

The following remain **documented_only**:

- The actual binary calling convention, integer declarations, and address width.
- The reason for the differing visible argument counts to F-85c5eeab8703.
- The meanings of all three context fields and of candidate values.
- Dependency effects on memory, globals, ownership, and lifetime.
- Whether external synchronization prevents concurrent field changes.
- Machine-level behavior for signed overflow or invalid address arithmetic.
- Relative ordering of the two field reads used as arguments to the third attempt.
- Whether exceptional dependency behavior can prevent an ordinary return.

Inferred intent is limited to the visible shape: ordered candidate acquisition, a success gate, and a conditional accounting-like adjustment. Those labels are descriptive, not additional guarantees.

## Evidence

Established static evidence consists solely of the supplied function body and the supplied mapping to opaque dependency IDs. It supports the branch conditions, dependency-call ordering, field offsets and widths, direct memory accesses, and ordinary return rules described above.

No dependency implementation, external interface declaration, structure definition, execution trace, or runtime test result was supplied. No runtime tests have been performed.
