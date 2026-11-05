<!-- Trusted provenance: F-b8d82aad4dfd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b8d82aad4dfd

## Purpose

F-b8d82aad4dfd returns a binary decision about a NUL-terminated byte sequence, subject to an initial dependency-controlled gate and a per-position dependency-controlled rejection condition.

For an input that remains unchanged throughout the call, and when both dependencies permit acceptance, the structural rule is:

- Without a period: accept one through eight bytes.
- With exactly one period: accept one through eight bytes before the period and zero through three bytes after it.
- Reject an empty input, an input beginning with a literal space, an input beginning with a period, or an input containing two or more periods.

These structural rules do not establish which other characters are permitted. A dependency is invoked once for each scanned position, but its implementation and means of observing input are unavailable.

The shape is consistent with a short filename-like check. That is inferred intent, not an established interface requirement. In particular, the visible behavior does not establish filesystem semantics, character-set restrictions, or any requirement for a nonempty extension.

## Interface

**Input:** one pointer to character-addressable storage.

**Output:** a 32-bit result whose only explicit values are:

- `1`: the visible acceptance conditions were satisfied.
- `0`: the initial gate denied further processing, a per-position call returned nonzero, or the structural conditions failed.

The result is a Boolean decision, not a length, error category, or dependency status. Dependency results are reduced to zero versus nonzero; their original values are not returned.

**Input representation:** when the initial gate permits processing, the function treats the pointed-to storage as a NUL-terminated sequence of bytes. The NUL terminator is not included in the measured length.

**Widths:** scan positions, the period count, the position immediately after the most recently observed period, and the arithmetic used to evaluate the final conditions are represented by signed 32-bit integers in the supplied representation.

**Preconditions for a defined, normally completing scan:**

- Every input byte read must be accessible.
- A NUL terminator must be reached before advancing the signed scan position would overflow.
- Dependencies must return normally if a normal return from this function is expected.
- Any dependency-induced changes must leave subsequent accesses valid.

There is no explicit null-pointer check or buffer-size argument.

The initial gate runs before any direct input read. If it returns nonzero, this function itself does not dereference the input pointer. This is narrower than a guarantee that an invalid pointer is safe: the gate’s own behavior is unknown.

No ownership transfer is expressed by the visible interface. The function neither explicitly allocates nor explicitly releases storage.

## Behavior

### Initial gate

The first observable operation is a call to F-e5c47acd678a, with no explicit arguments.

If that dependency returns nonzero, F-b8d82aad4dfd returns `0` immediately. It does not scan the input or call F-e6dffb87a39f on that path.

If the gate returns zero, scanning begins at position zero.

### Scan and dependency ordering

The scan maintains:

- The current zero-based position.
- A count of periods observed by the post-dependency byte reads.
- The position immediately following the most recently observed period, initially zero.
- A sticky flag recording whether any per-position dependency call returned nonzero.

For each visited position, the observable ordering is:

1. Read the byte at that position to decide whether it is NUL.
2. If it is NUL, finish the scan without a per-position dependency call for that position.
3. Otherwise, call F-e6dffb87a39f with no explicit arguments.
4. Record any nonzero dependency result as a rejection condition.
5. Read the byte at the same position again and compare it with a period.
6. If that second read observes a period, increment the period count and record the following position.
7. Advance to the next position.

A nonzero per-position result does not stop scanning. All subsequent positions continue to be examined until a scan-entry read observes NUL.

Likewise, discovering multiple periods does not stop scanning. The period-count restriction is applied only after termination has been found.

There is no explicit length-based early exit. An input that will ultimately be rejected for excessive length is still scanned to its terminator, subject to memory validity, dependency behavior, and representable arithmetic.

### Two reads at each scanned position

The byte deciding whether a position is scanned and the byte deciding whether it contributes a period are read on opposite sides of a dependency call.

Consequently, they need not have the same value:

- A non-NUL byte changed to NUL by F-e6dffb87a39f does not terminate that iteration. The function still performs the period comparison and advances.
- A non-period changed to a period before the second read contributes a period.
- A period changed to another byte before the second read does not contribute a period.

A NUL observed at the start of an iteration causes immediate scan termination. There is no per-position call or period-classification read for that terminating position.

Therefore, for mutable storage, the outcome is defined by the ordered observations, not necessarily by the contents at entry or return.

### Post-scan checks

Once a scan-entry read finds NUL, the function reads the first input byte again. It then requires all of the following before considering the remaining shape conditions:

- That newly read first byte is not a literal space.
- The terminating scan position is nonzero.
- No per-position dependency call returned nonzero.
- Fewer than two periods were counted.

The first-byte read precedes the nonempty-length check. An empty input is therefore read once to terminate scanning and again during post-scan validation.

A leading space is not rejected before scanning. The per-position calls and period observations occur first.

For a defined scan, the counted periods are nonnegative. Thus the “fewer than two” requirement permits only zero or one counted period.

### Structural acceptance conditions

Let:

- \(L\) be the position at which the scan-entry read observed NUL.
- \(D\) be the number of periods observed by the post-dependency reads.
- \(P\) be the position immediately following the last such period, or zero if no period was counted.
- \(S = L - P\).

Under the stated representability preconditions, \(0 \le P \le L\), so \(S\) is nonnegative.

After the preliminary checks, the final conditions are equivalent to:

- Either \(S \le 3\), or there were no counted periods.
- If there were no counted periods, \(L < 9\).
- \(P < 10\).
- \(P \ne 1\).

The signed subtraction and overflow-sensitive comparison in the supplied representation do not require modular arithmetic to describe valid inputs. Here, \(S\) lies between zero and the largest signed 32-bit integer, so subtracting three is representable. The resulting condition reduces to the suffix bound above.

For an input unchanged throughout execution:

**No period**

Here \(D=0\), \(P=0\), and \(S=L\). The accepted lengths are exactly one through eight bytes, provided the leading-space and dependency conditions also permit acceptance.

**Exactly one period**

Here \(P\) is the period’s one-based position. The conditions require \(2 \le P \le 9\) and \(0 \le S \le 3\). Equivalently:

- One through eight bytes precede the period.
- Zero through three bytes follow it.

A trailing period is therefore allowed. A leading period is not.

**Two or more periods**

The function returns `0`, regardless of segment lengths.

## Side effects

The visible body performs no explicit writes to the input or to global storage. Its direct operations consist of dependency calls, input reads, local bookkeeping, and a return.

This does not establish that the overall call is side-effect-free:

- F-e5c47acd678a may change externally visible state before the input is examined.
- F-e6dffb87a39f may change state once per scanned position.
- Neither dependency is established to preserve globals or input storage reachable through aliases.
- Neither dependency is established to preserve allocation lifetimes or to avoid ownership-related actions.

For a normally completing scan of length \(L\), the visible body performs:

- One initial-gate call.
- \(L\) per-position dependency calls.
- \(L+1\) scan-entry byte reads.
- \(L\) post-dependency period-comparison reads.
- One post-scan first-byte read.

These are logical accesses in the supplied representation, not a guarantee about physical load instructions or concurrent-memory behavior.

No diagnostic, exception mapping, recovery path, or error-specific return is visible. Invalid memory access and signed-position overflow are outside the established acceptance behavior.

## Dependencies

| Opaque function ID | Explicit arguments | Use of result | Established ordering |
|---|---|---|---|
| F-e5c47acd678a | None | Zero permits scanning; nonzero causes return `0`. | Called once, before any direct input access. |
| F-e6dffb87a39f | None | Any nonzero result permanently disqualifies acceptance for this invocation. | Called after a non-NUL scan-entry read and before the same position’s period-comparison read. |

The available evidence does not establish how either dependency reaches its result. In particular, F-e6dffb87a39f receives no explicit byte or position argument. Describing it as a character classifier would be an unsupported inference.

## Acceptance examples

The following examples are conditional, not runtime observations. Unless otherwise stated, assume the gate returns zero, every per-position call returns zero, the input remains unchanged, and all accesses and arithmetic satisfy the preconditions.

| Input | Result | Reason |
|---|---:|---|
| Empty sequence | `0` | Length is zero. |
| `A` | `1` | One byte, no period. |
| `ABCDEFGH` | `1` | Eight bytes, no period. |
| `ABCDEFGHI` | `0` | Nine bytes without a period exceed the bound. |
| `A.` | `1` | One-byte prefix and empty suffix are allowed. |
| `A.BCD` | `1` | One-byte prefix and three-byte suffix. |
| `A.BCDE` | `0` | Four-byte suffix. |
| `ABCDEFGH.XYZ` | `1` | Eight-byte prefix and three-byte suffix. |
| `ABCDEFGHI.X` | `0` | Nine-byte prefix places the post-period position outside the bound. |
| `.ABC` | `0` | The period has no preceding byte. |
| `A..B` | `0` | Two periods are counted. |
| ` A` | `0` | The first byte is a literal space. |
| `A B` | `1` | The visible structural checks do not reject an internal space under these dependency assumptions. |
| `A ` | `1` | The visible structural checks do not reject a trailing space under these dependency assumptions. |

Additional ordering examples:

- If the initial gate returns nonzero, the result is `0`; this function makes no direct input reads and no per-position calls.
- If an otherwise acceptable, stable three-byte input produces a nonzero result on the first per-position call, the final result is `0`, but all three per-position calls still occur.
- If a scanned byte changes to a period during its per-position call, it is counted as a period even if its scan-entry read observed another character.
- If a scanned byte changes to NUL during its per-position call, the scan advances rather than stopping at that position. Later readable storage is required.
- Bytes beyond the terminating NUL are not directly read on the stable-input path. Dependency accesses remain unspecified.

## Uncertainties

- **documented_only — Dependency semantics:** their purposes, global-state interactions, input aliases, memory actions, and failure behavior are unavailable.
- **documented_only — Character policy:** the body directly distinguishes NUL, literal space, and period. Any additional restrictions imposed by dependencies are unknown.
- **documented_only — Concurrent modification:** the visible ordering exposes distinct observations, but establishes no synchronization or memory model for concurrent writers.
- **documented_only — Invalid inputs:** null pointers, inaccessible memory, missing terminators, and scans requiring signed overflow have no established normal-return contract.
- **documented_only — Reentrancy and thread safety:** these cannot be established without dependency behavior.
- **Inferred intent:** the length and period rules resemble a short filename convention. No filesystem interpretation, normalization, case handling, or path handling is established.
- **Verification limit:** no runtime tests have been performed.

## Evidence

The contract is based solely on the supplied static function representation.

Established evidence includes the initial zero/nonzero gate, the two ordered byte reads surrounding each per-position call, the sticky rejection flag, deferred structural rejection, the final first-byte read, signed 32-bit bookkeeping, and explicit return values of zero and one.

The simplified length rules follow from the visible final conditions under the stated valid-scan arithmetic bounds. The examples are deductions from those conditions, not observed executions.

No dependency implementation or external interface specification was supplied. Unknown dependency effects are therefore left explicit rather than treated as absent.
