<!-- Trusted provenance: F-d6ed7b2dea17; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d6ed7b2dea17

## Purpose

F-d6ed7b2dea17 conditionally constructs a sequence of dependency operations and returns a 32-bit value representing either its original input or a dependency result.

Its decisions depend on two endpoint-like inputs. It computes an inclusive-span quantity, optionally performs an initial transformation when that quantity is not eight, and then selects between two further transformations according to an unsigned comparison.

**Inferred intent:** The structure is consistent with adjusting a selected bit range to an eight-unit width and positioning the result. The supplied evidence does not establish that interpretation. In particular, it does not establish the meanings of operation selectors 2, 6, 7, or 40, or the contents of the byte table used by the initial transformation. Those selectors are therefore specified as observable dependency arguments rather than named operations.

This is a static behavioral contract. No runtime tests have been performed.

## Interface

The function has five input slots and one 32-bit return value.

| Input | Representation in the supplied evidence | Established use |
|---|---|---|
| First input | Opaque 32-bit value | Not referenced by the visible body. |
| Context | 32-bit integer used as an address | A 32-bit field is read at byte offset 4. The context value is also passed to operation dependencies. |
| Source value | Opaque 32-bit value | Used directly as the initial candidate result and passed to dependencies on the initial-transformation path. |
| Lower endpoint | Unsigned 32-bit value | Participates in arithmetic, unsigned comparisons, and dependency arguments. |
| Upper endpoint | Signed 32-bit integer | Participates in the inclusive-span calculation. No subsequent signed ordering comparison uses it. |

The signed and unsigned labels reflect the supplied representation; they do not establish a public source-language declaration.

The return value is an opaque 32-bit quantity. Depending on the path, it is:

- the source value, unchanged;
- a value returned by F-b35e541ebc78 during the initial transformation; or
- a later value returned by F-b35e541ebc78 during the final transformation.

No separate success flag or error code is established.

### Arithmetic model

The branch-relevant arithmetic is 32-bit arithmetic. Define:

- **Span bits:** upper endpoint minus lower endpoint plus one, reduced modulo \(2^{32}\).
- **Span:** those bits interpreted as a signed 32-bit integer for the signed quantity and table index shown in the evidence.
- **Adjustment:** eight minus the span bits, reduced modulo \(2^{32}\), interpreted as an unsigned 32-bit value.

Thus:

\[
S = (U-L+1)\bmod 2^{32}
\]

\[
A = (8-S)\bmod 2^{32}
\]

The initial transformation is skipped exactly when \(S=8\). Subsequent comparisons of \(A\) and \(L\) are unsigned.

This fixed-width model describes the apparent machine-level calculation. It does not claim that equivalent signed arithmetic would be portable in an arbitrary source language.

### Preconditions and validity limits

For a normal execution of the visible body:

- The context must permit the initial 32-bit read at byte offset 4.
- If the span bits differ from eight, the table entry selected by the signed span must be readable.
- Each dependency must accept its actual arguments and return normally for execution to reach the next described event.

There is no visible validation of endpoint order, span range, context validity, table bounds, or dependency results. No restriction such as “span must be between one and eight” is enforced by this function.

## Behavior

### Initial state capture

The first visible memory access loads one 32-bit value from context byte offset 4. This value becomes the **captured handle**.

The function computes the span and adjustment, and initially selects the source value as its **candidate result**.

The captured handle is retained locally. Later calls use this captured value; the context field is not visibly reloaded, even if a dependency changes that field.

The first input has no visible effect on the calculation or dependency arguments.

### Optional initial transformation

When the span bits are not eight, the function performs these events in order:

1. Invoke F-b35e541ebc78 with five explicit argument values:
   - captured handle;
   - zero;
   - source value;
   - lower endpoint;
   - lower endpoint again.

   Retain its 32-bit result as the first intermediate value.

2. Read a one-byte table element selected by the signed span. The table base is obtained through a global pointer-valued location.

3. Invoke F-fbf0cbff8cb3 with:
   - context;
   - selector 40;
   - first intermediate value;
   - the selected byte value.

4. Invoke F-b35e541ebc78 with the two explicit argument values captured handle and zero. Replace the candidate result with its returned 32-bit value.

5. Invoke F-915779408ccb with:
   - context;
   - selector 2;
   - candidate result;
   - source value;
   - first intermediate value.

The visible body does not consume the return values of F-fbf0cbff8cb3 or F-915779408ccb.

The table access occurs after the first F-b35e541ebc78 call returns and before the F-fbf0cbff8cb3 invocation. Consequently, the first dependency call may affect which global table pointer or byte is subsequently observed. The function does not visibly capture the table pointer before that dependency call.

The selected table value is an unsigned byte, with a numeric range of 0 through 255. Its semantic meaning is unknown.

When the span bits equal eight, this entire phase is skipped: no table access occurs, the candidate remains the source value, and none of these three dependency invocations occurs.

### Final branch selection

After the optional initial phase, the function compares the unsigned adjustment with the unsigned lower endpoint.

There are three mutually exclusive outcomes.

#### Adjustment equals lower endpoint

If \(A=L\), the function returns the current candidate result immediately.

There are no final-transformation calls on this path.

This equality does not by itself suppress the optional initial phase. If the span bits differ from eight, that phase has already executed before the equality return.

#### Adjustment is less than lower endpoint

If \(A<L\) under unsigned comparison:

1. Invoke F-b35e541ebc78 with captured handle and zero, retaining its result as the final operand value.
2. Invoke F-fbf0cbff8cb3 with:
   - context;
   - selector 40;
   - final operand value;
   - \(L-A\), as a 32-bit quantity.
3. Invoke F-b35e541ebc78 again with captured handle and zero, retaining its result as the final result value.
4. Invoke F-915779408ccb with:
   - context;
   - selector 6;
   - final result value;
   - current candidate result;
   - final operand value.
5. Return the final result value.

Because this branch requires \(L>A\), its subtraction produces a strictly positive mathematical difference representable in 32 unsigned bits.

#### Adjustment is greater than lower endpoint

If \(A>L\) under unsigned comparison, the sequence is the same except:

- F-fbf0cbff8cb3 receives \(A-L\) as its fourth argument.
- F-915779408ccb receives selector 7 rather than selector 6.

The difference is again strictly positive and representable in 32 unsigned bits.

### Return-value identity

The final result returned on either non-equality path is the value obtained from the second F-b35e541ebc78 invocation in that final phase. It is not the return value of the subsequent F-915779408ccb invocation.

Likewise, when the initial phase runs and the function later takes the equality return, the returned candidate is the second F-b35e541ebc78 result from the initial phase.

These identity rules do not imply that dependency results are distinct, newly allocated, valid handles, or different from the source value.

## Side effects

### Directly visible memory accesses

The body visibly performs:

- one 32-bit read at context byte offset 4, on every path;
- one byte-table read when the span bits differ from eight.

There are no explicit memory stores in the visible body. The dependencies may perform stores or other externally observable actions; their implementations are not supplied.

The representation of the context field establishes its offset and width, but not its ownership, endianness, alignment requirements, or the layout of the rest of the context.

### Dependency effects and ordering

Dependency invocations occur synchronously in the order described above, assuming each returns normally.

The function does not:

- check dependency return values for failure;
- retry an operation;
- visibly undo earlier operations;
- explicitly release any dependency result;
- restore context fields or globals that dependencies may change.

These absences do not prove a leak or establish an ownership rule. A dependency could allocate, reuse, retain, consume, or release resources. None of those behaviors is known here.

In particular, the final F-915779408ccb call receives the value that the enclosing function will return. The supplied evidence does not establish whether that call preserves the value’s validity or changes an object represented by it.

### Call counts

For normal completion:

| Path | F-b35e541ebc78 | F-fbf0cbff8cb3 | F-915779408ccb |
|---|---:|---:|---:|
| Span equals eight; adjustment equals lower endpoint | 0 | 0 | 0 |
| Span equals eight; adjustment differs from lower endpoint | 2 | 1 | 1 |
| Span differs from eight; adjustment equals lower endpoint | 2 | 1 | 1 |
| Span differs from eight; adjustment differs from lower endpoint | 4 | 2 | 2 |

The two middle rows have the same counts but different arguments and selector sequences.

## Dependencies

### F-b35e541ebc78

Provides opaque 32-bit results used as intermediate operands, candidates, or returned values.

The supplied evidence shows two call shapes:

- five explicit arguments during the first call of the optional initial phase;
- two explicit arguments at all other sites.

The true dependency signature and calling convention are unknown. The shorter call sites must not be interpreted as proving that additional argument positions are zero, absent at the ABI level, or semantically ignored.

Every invocation visibly uses the same initially captured handle as its first explicit argument and zero as its second.

### F-fbf0cbff8cb3

Receives four explicit arguments. Its second argument is always selector 40.

Its third argument is a preceding F-b35e541ebc78 result. Its fourth is either:

- the unsigned byte selected from the table; or
- a strictly positive unsigned endpoint/adjustment difference.

Its return value is not used by this function.

### F-915779408ccb

Receives five explicit arguments.

Its second argument distinguishes the phase:

- selector 2 for the optional initial phase;
- selector 6 when adjustment is below the lower endpoint;
- selector 7 when adjustment is above the lower endpoint.

The remaining arguments connect the produced values as described in the behavior section. Its return value is not used by this function.

## Acceptance examples

These examples specify conditional observable traces, not runtime test results. They assume valid required memory, normally returning dependencies, and the stated arithmetic model. Symbolic dependency results identify particular call returns, not necessarily distinct objects.

### Eight-unit span with no dependency calls

Given lower endpoint 0 and upper endpoint 7:

- Span bits are 8.
- Adjustment is 0.
- The initial phase is skipped.
- Adjustment equals lower endpoint.
- The returned value is exactly the source value.

The context field is still read. The byte table is not accessed.

### Eight-unit span with selector 6

Given lower endpoint 3 and upper endpoint 10:

- Span bits are 8.
- Adjustment is 0.
- The initial phase is skipped.
- The final F-fbf0cbff8cb3 call receives difference 3.
- The final F-915779408ccb call receives selector 6 and the source value as its candidate operand.
- The return is the second F-b35e541ebc78 result from this final phase.

### Initial phase followed by equality return

Given lower endpoint 3 and upper endpoint 7:

- Span bits are 5.
- Adjustment is 3.
- The initial phase runs using table index 5.
- Its F-fbf0cbff8cb3 invocation receives the byte observed at that index.
- Its F-915779408ccb invocation uses selector 2.
- The final phase is skipped because adjustment equals lower endpoint.
- The return is the candidate produced during the initial phase.

### Both phases with selector 7

Given lower endpoint 1 and upper endpoint 3:

- Span bits are 3.
- Adjustment is 5.
- The initial phase uses table index 3 and selector 2.
- The final phase uses difference 4 and selector 7.
- The final combining call receives the candidate produced by the initial phase.
- The return is the result produced during the final phase, not the earlier candidate.

### Both phases with selector 6

Given lower endpoint 5 and upper endpoint 7:

- Span bits are 3.
- Adjustment is 5.
- This is an equality case, not a selector-6 case.

By contrast, given lower endpoint 6 and upper endpoint 8:

- Span bits remain 3.
- Adjustment remains 5.
- The initial phase runs.
- The final phase receives difference 1 and selector 6.

This distinguishes comparison of adjustment against the lower endpoint from comparison against the upper endpoint.

### Span greater than eight

Given lower endpoint 0 and upper endpoint 8:

- Span bits are 9.
- Adjustment is 4,294,967,295.
- The initial phase attempts table index 9.
- If that access is valid and the dependencies return normally, the final phase uses selector 7 and difference 4,294,967,295.

There is no visible clamp to eight and no rejection of this input.

### Reversed endpoints

Given lower endpoint 5 and upper endpoint 3:

- Span bits are 4,294,967,295, interpreted as signed span −1.
- Adjustment is 9.
- The initial phase attempts table index −1.

Normal completion cannot be asserted without knowing whether that table access is valid. The body contains no explicit reversed-range rejection.

## Uncertainties

The following are **documented_only**:

- The semantic purpose of all three dependencies and their selectors.
- The byte table’s contents, extent, initialization, mutability, and valid index domain.
- The true dependency signatures, especially the differing visible call shapes for F-b35e541ebc78.
- Whether returned values represent objects, identifiers, addresses, or another 32-bit representation.
- Ownership, lifetime, aliasing, allocation, retention, and release rules.
- Effects of dependencies on the context, globals, table storage, or previously produced values.
- Fault, exception, nonlocal-exit, and dependency-failure behavior.
- Concurrency, synchronization, and reentrancy guarantees.
- Any intended restriction on endpoint values beyond what the visible arithmetic and accesses require.

The bit-range interpretation remains inferred intent, not a requirement. No semantic labels such as masking, shifting, extraction, or allocation are necessary to satisfy the established trace contract.

## Evidence

Established static evidence comprises the supplied function body and the supplied mapping from dependency references to opaque function IDs.

That evidence supports:

- the five input slots and 32-bit return representation;
- the context read at byte offset 4;
- the fixed-width arithmetic and branch structure;
- the conditional one-byte table read;
- dependency invocation order, explicit argument values, selectors, and result flow;
- the absence of visible validation, explicit stores, cleanup calls, and dependency-result checks.

No dependency bodies, table contents, complete context layout, platform ABI specification, or runtime observations were supplied. The acceptance examples are arithmetic and trace consequences of the static evidence, conditional on the stated preconditions.
