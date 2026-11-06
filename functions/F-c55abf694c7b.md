<!-- Trusted provenance: F-c55abf694c7b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c55abf694c7b

## Purpose

F-c55abf694c7b performs a state update selected by a signed 32-bit field in a caller-supplied object. It invokes dependencies on two object subregions, then performs path-specific four-byte stores. The zero-selector path also passes a temporary value to a second dependency and stores the resulting value.

External operands are obtained from bindings evaluated at distinct points during execution. They are not established to be a single entry-time snapshot. In particular, an earlier dependency can change the binding subsequently used to obtain a later call’s operand.

The constants and arithmetic suggest regularly spaced quantities, possibly fixed-point values. Those interpretations are inferred intent, not established field semantics.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

The function accepts four inputs and has no explicit return value.

| Input | Established use |
|---|---|
| Object address | Base for the selector read, destination addresses passed to dependencies, and direct stores. |
| Second argument | A 32-bit value with no direct use in the supplied body. |
| Third argument | A 32-bit value with no direct use in the supplied body. |
| Fourth argument | A signed 32-bit value used to initialize temporary storage only on the zero-selector path. |

The absence of direct use of the second and third arguments establishes no dependency behavior or broader calling convention.

### Object layout used directly

Offsets are byte offsets from the supplied object address.

| Offset | Established role |
|---|---|
| `0x98` | First destination passed to F-470fed045f43 on every path. |
| `0x9C` | Four-byte store on both nonzero paths. |
| `0xA0` | Four-byte zero store on both nonzero paths. |
| `0xE0` | Second destination passed to F-470fed045f43 on every path. |
| `0xF8` | Base used to address the negative path’s additional pair of stores. |
| `0xFC` | Four-byte store on the negative path. |
| `0x100` | Four-byte zero store on the negative path. |
| `0x130` | Initial signed 32-bit selector read. |
| `0x26C` | Final four-byte store on every normally completing path. |

The complete object type, allocation model, and dependency-accessed region sizes are unknown. A contiguous allocation covering offsets through `0x26F` would cover the direct object accesses, but would not by itself establish validity for the dependency calls.

### External bindings

Six external bindings supply operands. The following labels identify their distinct uses without assigning domain semantics:

| Binding label | Use |
|---|---|
| **Zero-first** | Second operand of the first F-470fed045f43 call when the selector is zero. |
| **Zero-second** | Second operand of the second F-470fed045f43 call when the selector is zero. |
| **Adjustment** | Second operand of F-f79d2cf69ee3. |
| **Shared** | Base for a nonzero-path arithmetic read at relative offset `+0x04`; separately evaluated as the first F-470fed045f43 call’s second operand. |
| **Difference** | Separately referenced for two negative-path reads, at relative offsets `+0x04` and `−0x44`. |
| **Nonzero-second** | Second operand of the second F-470fed045f43 call on either nonzero path. |

A **binding read** obtains the operand currently supplied by that binding. A **relative read** obtains a signed 32-bit value from storage addressed using such an operand. These are distinct accesses: reading Shared does not itself read the signed value at its relative offset `+0x04`.

The width and physical representation of the external bindings are not independently established. The explicitly typed selector, relative data reads, temporary value, arithmetic values, and direct object stores are four bytes. The source representation suggests a 32-bit address environment; the actual ABI, byte order, and alignment requirements remain unknown.

### Preconditions

Normal completion requires:

- A valid initial selector read.
- Valid storage for the direct stores taken by the selected path.
- Accessible external bindings and valid relative-read addresses where used.
- Arguments satisfying each invoked dependency’s unavailable contract.
- Arithmetic and address calculations permitted by the actual execution environment.

There are no explicit null, bounds, alignment, overflow, or resource-validity checks.

## Behavior

### Initial selection

The function first reads the signed 32-bit selector at object offset `0x130`. That value selects one path:

- Zero: the zero path.
- Negative: the negative path.
- Positive: the positive path.

There is no subsequent direct selector read. A dependency changing the selector does not cause reselection.

All ordering below concerns the operations visible in the supplied body. It does not define internal dependency operations or impose an otherwise unspecified ordering between argument evaluations within an individual call.

### Zero path

The sequence is:

1. Initialize a four-byte automatic temporary from the fourth argument.
2. Read the Zero-first binding and invoke F-470fed045f43 with destination `object + 0x98` and the obtained operand.
3. After that invocation returns, read the Zero-second binding and invoke F-470fed045f43 with destination `object + 0xE0` and the newly obtained operand.
4. After the second invocation returns, read the Adjustment binding and invoke F-f79d2cf69ee3 with the temporary’s address and that operand.
5. After F-f79d2cf69ee3 returns, read the temporary’s resulting 32-bit value and store it at object offset `0x26C`.
6. Return.

The Zero-second binding is not captured before the first call. A change to it during the first call can therefore change the operand supplied to the second call. Similarly, the Adjustment binding is evaluated only after the second call returns; changes made by either preceding call can affect its eventual value.

The last store uses the temporary’s post-dependency value, not necessarily the fourth argument.

This path contains no direct stores to offsets `0x9C`, `0xA0`, `0xFC`, or `0x100`. That is not a promise that those locations remain unchanged: dependency write sets are unknown.

### Negative path

Let:

- \(m<0\) be the initially read selector.
- \(r\) be the signed 32-bit value obtained from a Shared-binding evaluation followed by a relative read at `+0x04`.
- \(u\) be the signed 32-bit value obtained using a Difference-binding reference and relative offset `+0x04`.
- \(v\) be the signed 32-bit value obtained using a separate Difference-binding reference and relative offset `−0x44`.

For representable arithmetic, define:

\[
d=(m+1)\times983{,}040,\qquad a=d-r,\qquad b=u-v.
\]

The established sequence is:

1. Compute \(d\).
2. Read the Shared binding, read the signed value at that operand’s relative offset `+0x04`, and compute \(a\).
3. Perform the two Difference-based relative reads and compute \(b\).
4. Evaluate the Shared binding again and call F-470fed045f43 with destination `object + 0x98` and this later operand.
5. After that call returns, read the Nonzero-second binding and call F-470fed045f43 with destination `object + 0xE0` and the obtained operand.
6. Store \(a\) at offset `0x9C`.
7. Store four-byte zero at offset `0xA0`.
8. Store \(b\) at offset `0xFC`.
9. Store four-byte zero at offset `0x100`.
10. Store `0x00010000`, decimal 65,536, at offset `0x26C`.
11. Return.

The Difference expression contains two separate binding references, not one captured base reused for both reads. The relative evaluation order of its two operands is unspecified by the supplied representation. Thus, neither “the `+0x04` operand is read first” nor “both relative reads necessarily use the same binding value” is a contract guarantee. Both operand evaluations finish before the first dependency call.

Likewise, the Shared binding used for arithmetic and its later evaluation as a call argument are distinct accesses. The body does not reuse a saved arithmetic base for the call. No dependency call intervenes between these two accesses, but equality of their results still requires binding stability over that interval.

The first dependency can change the Nonzero-second binding before its evaluation for the second call.

The arithmetic results \(a\) and \(b\) are computed before either dependency call. Later changes to their source bindings or source data do not cause recomputation.

### Positive path

Let \(m>0\) be the initial selector, and let \(r\) be the signed 32-bit value obtained by evaluating Shared and reading at relative offset `+0x04`.

For representable arithmetic:

\[
d=(m-1)\times983{,}040,\qquad c=d+r.
\]

The sequence is:

1. Compute \(d\).
2. Read the Shared binding, read the signed value at its relative offset `+0x04`, and compute \(c\).
3. Evaluate the Shared binding again and invoke F-470fed045f43 with destination `object + 0x98` and this later operand.
4. After that call returns, read the Nonzero-second binding and invoke F-470fed045f43 with destination `object + 0xE0` and the obtained operand.
5. Store \(c\) at offset `0x9C`.
6. Store four-byte zero at offset `0xA0`.
7. Store `0x00010000` at offset `0x26C`.
8. Return.

The arithmetic-related Shared evaluation and the call-argument evaluation are separate, just as on the negative path. The Nonzero-second evaluation occurs after the first dependency returns and can observe a rebinding performed during that call.

This path performs no Difference-binding references or Difference-relative reads. It contains no direct stores to offsets `0xFC` or `0x100`.

### Widths and arithmetic limits

The multiplier is `0x000F0000`, decimal 983,040. Arithmetic values are represented as signed 32-bit integers; the signed range is \(-2{,}147{,}483{,}648\) through \(2{,}147{,}483{,}647\).

- For every negative 32-bit selector, \(m+1\) is representable.
- For every positive 32-bit selector, \(m-1\) is representable.
- The negative-path product is representable exactly for selectors from `−2185` through `−1`.
- The positive-path product is representable exactly for selectors from `1` through `2185`.
- A representable product does not guarantee that the subsequent subtraction or addition is representable.
- The independent difference \(u-v\) can overflow.

For example, \(2184\times983{,}040=2{,}146{,}959{,}360\), whereas \(2185\times983{,}040=2{,}147{,}942{,}400\), above the signed maximum.

The evidence does not establish executable overflow semantics. No modular result, trap, error return, or successful continuation is guaranteed for out-of-range arithmetic.

## Side effects

### Direct stores

All dependency calls on a normally completing path finish before that path’s direct object stores begin.

| Path | Direct stores, in order |
|---|---|
| Zero | `0x26C`: resulting temporary value |
| Negative | `0x9C`: \(a\); `0xA0`: zero; `0xFC`: \(b\); `0x100`: zero; `0x26C`: 65,536 |
| Positive | `0x9C`: \(c\); `0xA0`: zero; `0x26C`: 65,536 |

Each store is four bytes. If a dependency writes one of these fields and returns normally, the applicable later direct store overwrites that value. No preservation claim follows for other fields.

### External reads and dependency effects

On the zero path, the body evaluates three external bindings in call order, with each later evaluation occurring after the preceding call returns.

On the positive path, it evaluates Shared twice and Nonzero-second once. On the negative path, it additionally contains two separate Difference-binding references. These counts describe body-level evaluations, not physical bus transactions or reads performed inside dependencies.

Dependencies are not assumed to preserve globals, object contents, resource contents, or bindings. They are not assumed to avoid callbacks or nonlocal exits. An earlier call can affect both the operand and the data subsequently observed through a later binding evaluation.

No disjointness is established between the object, binding storage, or relatively addressed data. Pre-call arithmetic results nevertheless remain the results used by the later direct stores; the body does not reread their inputs after the calls.

### Ownership and lifetime

There is no explicit allocation, deallocation, reference-count operation, or ownership-transfer operation in this body. This does not establish the absence of those effects inside dependencies.

The temporary passed to F-f79d2cf69ee3 has automatic lifetime ending when this invocation returns. The dependency’s retention behavior is unknown. Safe deferred use of that address is not established.

### Errors and incomplete execution

There is no explicit status result, validation failure branch, retry, or rollback. Dependency return values are not consumed.

If an access, calculation, or call prevents continuation, later operations are not guaranteed. This function provides no explicit rollback of effects already performed. The actual fault, exception, or nonlocal-exit mechanism is unknown.

## Dependencies

### F-470fed045f43

Invoked twice on every path reaching both calls:

1. Destination at object offset `0x98`.
2. Destination at object offset `0xE0`.

The second operands come from Zero-first and Zero-second on the zero path, or Shared and Nonzero-second on either nonzero path. Each is obtained at its specified evaluation point, not collectively captured at entry.

The call shape suggests subregion initialization or assignment. That is inferred intent only. Operand types, accessed extents, mutation of source storage, allocation or release behavior, retention, return semantics, and failure behavior are unknown.

### F-f79d2cf69ee3

Invoked only on the zero path, after both F-470fed045f43 calls. It receives the temporary’s address and the operand obtained from the then-current Adjustment binding.

Its transformation is unknown. The caller uses the temporary’s value after it returns. No restriction on its other effects or ownership behavior is established.

## Acceptance examples

These are conditional specifications, not executed tests. Unless stated otherwise, storage is valid, dependencies return normally, and arithmetic is representable.

### Zero selector and transformed temporary

With initial selector zero and fourth argument 25, the temporary is initialized to 25 before the first dependency call. If it contains 40 when F-f79d2cf69ee3 returns, the final direct store at `0x26C` is 40. There is no direct store of 65,536 on this path.

### Rebinding between zero-path calls

Suppose Zero-second initially supplies operand A. If the first F-470fed045f43 call changes that binding to supply B and it remains B until evaluation, the second call receives B.

If the second call changes Adjustment to supply C and it remains C until evaluation, F-f79d2cf69ee3 receives C. Neither operand is fixed by its entry-time value.

### Positive arithmetic

For selector 1 and arithmetic relative-read value 300, the direct stores after both calls are 300 at `0x9C`, zero at `0xA0`, and 65,536 at `0x26C`.

For selector 2 and the same relative-read value, `0x9C` instead receives 983,340.

### Separate Shared evaluations

Suppose a positive-path arithmetic evaluation obtains Shared operand A and reads 300 from A at `+0x04`. If the later call-argument evaluation obtains B, the first dependency receives B, while the arithmetic still uses 300. This example is conditional on such a binding change occurring; the function itself performs no intervening rebinding operation.

### Rebinding on a nonzero path

If the first dependency changes Nonzero-second from A to B and it remains B until the subsequent evaluation, the second dependency receives B. This applies to both nonzero paths.

### Negative unit selector

For selector `−1`, arithmetic Shared value 300, and Difference-relative values \(u=900\), \(v=200\), the ordered stores are:

- `0x9C`: −300.
- `0xA0`: zero.
- `0xFC`: 700.
- `0x100`: zero.
- `0x26C`: 65,536.

All follow both dependency calls.

### Separate Difference references

If the binding evaluation associated with `+0x04` obtains operand A, while the evaluation associated with `−0x44` obtains operand B, the difference uses the value at A’s `+0x04` minus the value at B’s `−0x44`. This does not establish which operand evaluation occurred first.

### Changes after arithmetic

If a dependency changes the selector or previously read arithmetic storage, execution remains on the initially selected path and uses the already-computed arithmetic results.

### Out-of-range product

For selector 2186, the mathematical scaled product exceeds the signed 32-bit maximum. No required wrapped value, failure mechanism, or completed object state is specified.

## Uncertainties

The following remain **documented_only**:

- Exact ABI, byte order, alignment, address width, and address-arithmetic rules.
- External binding widths and physical representations.
- Overflow behavior outside representable signed arithmetic.
- Complete object layout, allocation boundaries, and field meanings.
- Binding stability, resource contents, aliasing, and concurrent modification.
- Dependency read/write extents, global effects, ownership, retention, and failures.
- Reentrancy, synchronization, callbacks, and device-driven changes.
- Relative evaluation order of the two Difference operands.
- Domain meaning of the constants and arithmetic.

The positions of binding evaluations relative to dependency calls, the repeated Shared evaluations, and the separate Difference references are established behavior, not uncertainties.

## Evidence

The supplied body establishes the initial signed selector read, three paths, four-byte data accesses, arithmetic constants, dependency order, binding evaluation points, and ordered direct stores.

The overflow bounds follow from the displayed signed 32-bit operations and numerical constants. Interpretation as fixed-point spacing or subregion initialization remains inferred intent.

No dependency implementations, independent ABI documentation, complete object definitions, or runtime observations were supplied. No runtime tests have been performed.
