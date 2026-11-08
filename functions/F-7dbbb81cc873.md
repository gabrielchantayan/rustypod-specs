<!-- Trusted provenance: F-7dbbb81cc873; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7dbbb81cc873

## Purpose

F-7dbbb81cc873 coordinates callback-based formatting of a three-word record. It may emit a prefix, then selects either an alternate formatter or a two-call formatting path. It returns a computed signed count or an explicit −1 on specified failure paths.

On the two-call path, the first call receives a writable indicator and zero context. Its return value contributes to the computed count. If the caller’s context is nonzero, a second call follows, possibly surrounded by one-unit delimiter callbacks.

“Measurement” and “actual output” describe the apparent roles of these calls. They do not establish that the first call is side-effect-free, that the second successfully produces output, or that dependencies preserve shared memory.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Arguments

1. **Output callback**

   A callable address passed to both formatting dependencies and invoked directly by this function with three arguments:

   - the original context;
   - an output-source address or address-like value;
   - a count.

   For direct invocations, exactly zero means failure. Every nonzero result, including a negative result, means success for this function’s control flow. Callback results are not accumulated into the returned count.

   Callback behavior inside dependencies is outside the established contract.

2. **Context**

   A signed 32-bit value passed unchanged to direct callbacks and the alternate formatter.

   On the two-call path, zero suppresses this function’s actual-output call and delimiter callbacks. It does not suppress optional prefix callbacks, nor does it establish that the measurement dependency performs no output.

3. **Flags**

   An unsigned 32-bit field. This function directly interprets:

   | Bits | Effect |
   |---|---|
   | `0x40` | Request the optional prefix stage. |
   | `0x80` | Select the alternate formatter after the prefix stage. |
   | `0x20` | Bypass classification-table selection and conditional alternate fallback. |
   | `0x100` | Select the alternate formatter when classification is unavailable, but only when `0x20` is clear. |
   | `0x10` | Transform the selected mode on the two-call path. |
   | `0x0f` | Supply a separate option value to both calls of the two-call formatter. |

   The alternate formatter receives the complete flag word. Bits not otherwise interpreted here are not removed.

4. **Record address**

   An address exposing three consecutive 32-bit words:

   | Byte offset | Established use |
   |---|---|
   | 0 | Read as the second argument to each two-call formatter invocation. |
   | 4 | Initially read as a signed classification value; also supplied to the prefix-source dependency. |
   | 8 | Read as the first argument to each two-call formatter invocation. |

   The payload meanings of words at offsets 0 and 8 are unknown. Their width does not establish that they are pointers.

### Result

The result is a signed 32-bit integer.

Explicit failure branches return −1. Other returns are arithmetic combinations of dependency-provided counts and fixed increments. Negative measurement results are not rejected, and arithmetic has no explicit overflow handling. Consequently, the result’s sign alone is not a general success criterion, and −1 can also arise from unchecked count arithmetic.

### Preconditions

The record must support the reads taken by the selected path. The word at offset 4 is read before any formatter-selection branch. Words at offsets 0 and 8 are read only when the two-call formatter is invoked.

The callback must be callable whenever directly invoked or used by a dependency. Output objects and dependency-provided sources must be usable for the operations that consume them.

There is no explicit null-record check, callback validation, record-size validation, source validation, or count-range validation. Behavior following an invalid memory access or a dependency that does not return normally is not established.

## Behavior

### Initialization and prefix stage

The function initializes a local unsigned 32-bit indicator to zero, reads and retains the record word at offset 4, and initializes the accumulated prefix contribution to zero.

If `0x40` is clear, no prefix dependency or prefix callback is invoked.

If `0x40` is set, operations occur in this order:

1. F-5e1a5190a137 receives the retained classification value and returns the source argument for the first prefix callback.
2. F-1cb585febd84 is called without explicit arguments. Its signed result becomes both the first callback’s count and the accumulated prefix contribution.
3. The callback receives the original context, the returned source, and that count.
4. A zero callback result immediately returns −1.
5. Otherwise, the callback receives the original context, a fixed prefix-separator object, and count 1.
6. A zero result immediately returns −1.
7. Otherwise, the accumulated prefix contribution is increased by one.

These callbacks occur even with zero context. Negative prefix counts are forwarded without validation.

The source-producing dependency runs before the count-producing dependency. No guarantee establishes that the latter preserves the returned source or its backing storage.

### Alternate-formatter selection

After successful completion or omission of the prefix stage, `0x80` selects F-281ad1282806. It receives, in order:

- the complete flags;
- the callback;
- the original context;
- the original record address.

A negative dependency result causes an explicit −1 return. Otherwise, the result is added to the accumulated prefix contribution and returned.

This path does not reject a negative accumulated prefix contribution. It also does not locally suppress alternate-formatter activity when context is zero.

Without `0x80`, selection proceeds as follows:

- If `0x20` is set, classification-table selection is bypassed. The initial mode is 1, and `0x100` has no fallback effect.
- Otherwise, the retained classification value is eligible for lookup only if its unsigned 32-bit value, after subtracting one, is less than 30. This admits exactly classifications 1 through 30.
- An eligible value causes one byte to be read at the then-current classification-table base plus that value.
- The byte is converted through the supplied character interpretation to an unsigned 32-bit mode. A converted value of `0xffffffff` means unavailable.
- An ineligible classification or unavailable entry selects the alternate formatter if `0x100` is set.
- Without that fallback, unavailable classification selects initial mode 1.
- An available table entry becomes the initial mode and prevents the `0x100` fallback.

A valid zero table entry is an available mode, not an unavailable-entry marker. Character signedness affects high-bit entries and remains unresolved below.

### Mode transformation

The initial mode remains unchanged when `0x10` is clear.

When `0x10` is set:

- zero becomes 1;
- every nonzero mode has bit `0x08` set.

Therefore, default mode 1 becomes 9 when `0x10` is present. Bypassing classification does not bypass this transformation.

The transformed mode is retained for both formatter calls.

### Measurement call and count calculation

F-04fc1fe50d53 receives seven arguments:

1. the current record word at offset 8;
2. the current record word at offset 0;
3. the transformed mode;
4. the low four flag bits;
5. the address of the initialized 32-bit indicator;
6. the callback;
7. zero context.

This call occurs before checking whether the accumulated prefix contribution is negative.

After the call:

- A negative accumulated prefix contribution causes an explicit −1 return, without this function’s delimiter callbacks or actual-output call.
- Otherwise, the dependency’s signed return value is added to that contribution. A negative dependency result is not independently rejected.
- If the indicator’s low byte is nonzero, two is added to the computed count. Higher indicator bits do not affect this decision.

If the original context is zero, the computed count is returned at this point.

The count is not a verified total of output accepted by callbacks. In particular, the two-unit adjustment reflects the indicator at count calculation, not necessarily the later delimiter actions.

### Actual-output call and delimiter decisions

For nonzero context, the indicator’s low byte controls an opening-delimiter decision. If nonzero, the callback receives:

- the original context;
- a fixed delimiter object;
- count 1.

Zero from this callback immediately returns −1 and prevents the actual-output formatter call. Nonzero permits continuation. If the low byte was zero, no opening callback is made.

F-04fc1fe50d53 is then called with:

1. the then-current record word at offset 8;
2. the then-current record word at offset 0;
3. the retained transformed mode;
4. the low four flag bits;
5. a null indicator address;
6. the callback;
7. the original nonzero context.

Its return value is ignored.

After it returns, the indicator’s low byte is read again:

- If zero, the previously computed count is returned.
- If nonzero, the callback receives the original context, the same fixed delimiter object, and count 1.
- Zero from that closing callback returns −1; nonzero returns the previously computed count.

The null indicator argument on the second call does not prove that the indicator cannot change. The first call received its address, and the supplied evidence does not prohibit retained aliases or later alias-mediated writes. Thus opening and closing decisions need not agree, and neither decision causes the count to be recomputed.

## Side effects

### Read, call, and write ordering

Established ordering includes:

- Indicator initialization precedes the initial classification read.
- The retained classification is read before prefix dependencies or callbacks.
- Both successful prefix callbacks precede formatter selection and any classification-table lookup.
- Table selection uses the table reference observed after the prefix stage, not a guaranteed pre-prefix snapshot.
- Measurement precedes the negative-prefix rejection and count calculation.
- The opening-delimiter decision follows count calculation.
- A required opening callback must succeed before the actual-output call.
- The actual-output call precedes the final indicator read and possible closing callback.

The relative evaluation order of the two record-word reads used as arguments to a single formatter call is not established.

### Record and indicator state

No direct stores to the caller’s record are visible in this function.

The classification value is retained from its initial read. Later record mutation does not change this function’s classification decision.

The payload words at offsets 0 and 8 are read separately for measurement and actual output. Dependencies or callbacks may change them between calls; the actual-output call can therefore receive different payloads from measurement. No snapshot or consistency check is imposed.

The indicator is a local 32-bit object initialized to zero and exposed by address to measurement. Only its low byte controls this function’s delimiter-related decisions. If measurement leaves it unchanged, it remains zero, but no general preservation guarantee applies after its address has been exposed.

### Ownership and partial completion

No direct allocation, deallocation, ownership transfer, or cleanup operation is visible.

This absence does not establish dependency ownership rules. Source lifetime, retention of addresses, release obligations, and storage managed inside dependencies remain undocumented.

Output is not rolled back. A prefix, opening delimiter, or body may already have been produced when a later direct callback returns zero.

## Dependencies

| Opaque ID | Established interaction | Unresolved contract |
|---|---|---|
| F-5e1a5190a137 | Receives the retained classification; returns the first prefix callback’s source. | Contents, validity, lifetime, ownership, failure representation, and side effects. |
| F-1cb585febd84 | Called after the source dependency, without explicit arguments; returns the signed prefix count. | Count units, range, derivation, and effects on shared state or the returned source. |
| F-281ad1282806 | Receives complete flags, callback, context, and record. Negative results are normalized to −1. | Formatting, record access, zero-context behavior, callback handling, ownership, and mutations. |
| F-04fc1fe50d53 | First receives a writable indicator and zero context; second, when selected, receives a null indicator argument and nonzero context. | Payload semantics, output syntax, count units, error conventions, retained aliases, and other effects. |

The caller-provided callback has no supplied opaque ID. Its established direct-call contract is limited to the argument tuples and zero-versus-nonzero result handling described above.

Visible data dependencies are the classification-table reference, fixed prefix-separator object, and fixed delimiter object. Their contents are unknown. A count of 1 establishes a one-unit request, not a particular character or encoding.

## Acceptance examples

These are conditional static consequences, not executed tests. Unless stated otherwise, arithmetic remains within signed 32-bit range and dependencies return normally.

1. **Default count-only path.**  
   With flags exactly `0x20`, zero context, and measurement returning 7 while leaving the indicator low byte zero, the result is 7. Measurement receives mode 1. This function makes no direct callback and no actual-output call.

2. **Count-only delimiter adjustment.**  
   Under the same conditions, a nonzero indicator low byte after measurement produces result 9. No delimiter callbacks occur because context is zero.

3. **Prefix with zero context.**  
   With `0x40` set, a prefix count of 4, and both prefix callbacks returning nonzero, the accumulated prefix contribution is 5. Both callbacks receive zero context if that is the caller’s context.

4. **Unconditional alternate selection.**  
   With flags exactly `0x80`, an alternate result of 12 produces 12; an alternate result of −2 produces −1. There is no table lookup or two-call formatter invocation.

5. **Conditional alternate fallback.**  
   With flags exactly `0x100`, classification 0 or 31 selects the alternate formatter. Classification 30 selects it only if the converted table entry is unavailable. An available zero entry instead selects the two-call path with mode zero.

6. **Bypass and transformation are separate.**  
   With flags exactly `0x120`, the initial and passed mode are both 1; there is no table lookup or conditional alternate fallback. With flags exactly `0x130`, the initial mode is still 1, but the passed mode is 9 because `0x10` transforms it.

7. **Available-mode transformation.**  
   For an available table mode of zero, `0x10` changes the passed mode to 1. For mode 2, it changes the passed mode to 10. Without `0x10`, these modes remain zero and 2.

8. **Direct callback results are Boolean-like.**  
   An opening-delimiter callback result of −7 permits the actual-output call. A result of zero prevents that call and returns −1.

9. **Ignored actual-output result, with final decision specified.**  
   With flags exactly `0x20`, nonzero context, measurement returning 6, and a zero indicator low byte at count calculation and the opening decision, the computed count is 6 and no opening callback occurs. If actual output returns a negative value, that value is ignored. The function returns 6 only if the final indicator low byte is zero, or if it is nonzero and the required closing callback returns nonzero. If the final low byte is nonzero and the closing callback returns zero, the result is −1. There is no prefix contribution in this example.

10. **Measurement precedes negative-prefix rejection.**  
    If the prefix count is −2 and both prefix callbacks succeed, the accumulated prefix contribution becomes −1. On a selected two-call path, measurement still occurs. The function then returns −1 without requesting actual output or making delimiter callbacks.

11. **Later indicator mutation does not revise the count.**  
    With flags exactly `0x20`, nonzero context, measurement returning 6, and the indicator low byte zero at count calculation and the opening decision, suppose a retained alias allows the indicator to become nonzero during actual output. A closing callback is then required. If it succeeds, the returned count remains 6, not 7 or 8.

12. **Negative measurement is not normalized.**  
    With flags exactly `0x20`, zero context, measurement returning −3, and the indicator low byte zero, the function returns −3.

## Uncertainties

- **documented_only — arithmetic overflow.** Counts and results are represented as signed 32-bit values. Prefix increments and count additions have no explicit overflow checks. The evidence does not establish portable overflow behavior; wrapping must not be assumed. The classification range test explicitly uses unsigned 32-bit arithmetic.

- **documented_only — character signedness.** Table entries are interpreted through a plain character before unsigned conversion. Under signed 8-bit character interpretation, byte `0xff` becomes `0xffffffff` and is unavailable; other high-bit bytes become large unsigned modes. Under unsigned character interpretation, byte `0xff` becomes 255 and does not match the unavailable marker. The supplied evidence does not independently establish which interpretation applies.

- **documented_only — alias preservation.** Measurement receives the indicator address. Later calls may affect it through a retained alias; the second call’s null indicator argument is not a preservation guarantee. Whether any dependency actually retains or uses such an alias is unknown.

- **documented_only — external mutation.** Dependencies and callbacks are not established to preserve the record, classification-table reference, table contents, output objects, or other shared memory. Concurrent access and reentrancy guarantees are not supplied.

- **documented_only — output semantics.** Separator and delimiter identities, body syntax, encoding, and count units are unknown.

- **documented_only — dependency failures and ownership.** Internal error conventions, memory management, address retention, and exceptional or non-returning behavior are not established beyond this function’s explicit result checks.

- **Inferred intent.** The first two-call formatter invocation appears intended to measure a representation and indicate surrounding delimiters; the second appears intended to emit the body. These interpretations do not strengthen the established dependency contracts.

## Evidence

Established behavior derives solely from the supplied static function body and opaque dependency mapping: argument widths, record offsets, initialization, branch conditions, explicit call arguments, reads, count arithmetic, result checks, and returns.

No dependency bodies, fixed-data contents, caller contracts, platform arithmetic specification, or runtime observations were supplied. No runtime tests have been performed. Acceptance examples state conditional consequences of the visible control flow, not verified executions.
