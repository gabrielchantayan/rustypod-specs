<!-- Trusted provenance: F-3a3de8dc90f7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3a3de8dc90f7

## Purpose

F-3a3de8dc90f7 conditionally delegates an operation after two prerequisite calls and a set of byte-level checks on a dependency-populated record. When the record passes those checks, the function constructs a 32-bit value from four record bytes, applies wrapping alignment arithmetic, and supplies the result to the final dependency.

The established behavior is a validation-and-dispatch gate. The underlying record format, meaning of the calculated value, and application-level operation are not established. In particular, the evidence does not establish that the prerequisites perform seeking or reading, even though that interpretation may be plausible.

The function returns:

- The first prerequisite’s result if that result is nonzero.
- The second prerequisite’s result if that result is nonzero.
- The integer value 2 if both prerequisites return zero but record validation fails.
- The final dependency’s result if validation succeeds.

No runtime tests have been performed.

## Interface

The interface has four opaque 32-bit input values and a 32-bit integer result. This contract refers to the inputs by their positions: first, second, third, and fourth.

| Input | Established use |
|---|---|
| First | Passed unchanged to F-b829b770e451 if validation succeeds. |
| Second | Passed to both prerequisites and, on success, to F-b829b770e451. |
| Third | Passed unchanged to F-b829b770e451 if validation succeeds. |
| Fourth | Passed unchanged to F-b829b770e451 if validation succeeds. |

No input is directly dereferenced in the supplied function body. Any pointer, handle, descriptor, or other interpretation belongs to the dependencies’ undocumented contracts.

The function supplies F-0c21b5cad025 with the second input, a pointer into local scratch storage, and the integer value 128. The subsequent checks interpret byte positions relative to that pointer.

### Record layout used by this function

Offsets are zero-based. Each listed field occupies one byte unless otherwise stated.

| Offset | Interpretation within this function |
|---|---|
| 0 | Initial zero-valued gate; conditionally overwritten with the byte at offset 74. |
| 1 | Unsigned selector or length-like value. Successful validation requires a value from 1 through 33. |
| 2–62 | A 61-byte region containing the dynamically selected zero-byte position. |
| 63 | Separate byte that must be zero on every successful validation path. |
| 74 | Additional zero-valued gate, consulted only when the original byte at offset 0 is zero. |
| 82 | Additional zero-valued gate, consulted only when offsets 0 and 74 have both passed. |
| 83–86 | Four bytes combined in most-significant-byte-first order into an unsigned 32-bit value. |

For an accepted selector \(n\), the dynamically checked byte is at offset \(2+n\). Consequently, accepted selectors cause accesses only to offsets 3 through 35. The byte at offset 2 is never the dynamically checked byte.

No requirement is established for unlisted bytes.

### Preconditions and ownership

The second prerequisite must provide usable byte values at every position subsequently examined if it returns zero. A zero result alone does not independently prove that it initialized those positions; that guarantee would have to come from its own contract.

The supplied evidence shows a pointer and a size-like argument of 128. It does not independently establish the full writable extent of the underlying local storage or the dependency’s write bounds. Safe operation therefore depends on the actual storage layout and F-0c21b5cad025 honoring a compatible memory contract.

The scratch storage is local to this invocation. There is no explicit allocation, deallocation, or ownership transfer in the visible function body. Passing the scratch pointer does not establish that the dependency may retain it after returning.

## Behavior

### Prerequisite calls and early returns

The function first invokes F-e40a3ceddab2 with:

1. The second input.
2. The integer value zero.

If its result is nonzero, the function returns that exact result. It does not invoke either remaining dependency and does not inspect the scratch record.

If the result is zero, the function invokes F-0c21b5cad025 with:

1. The second input.
2. The local scratch pointer.
3. The integer value 128.

If this result is nonzero, the function returns that exact result. It does not validate the record or invoke F-b829b770e451.

Negative and positive nonzero prerequisite results are treated alike: both terminate processing and propagate unchanged. The function does not translate prerequisite failures into the value 2.

### Leading zero checks

Following two zero prerequisite results, the function examines the byte at offset 0.

- If that byte is nonzero, validation fails.
- If it is zero, the function reads the byte at offset 74 and writes that byte into local offset 0.
- If the copied byte is nonzero, validation fails.
- If the copied byte is zero, the function examines offset 82. A nonzero value there causes validation failure.

Thus, successful validation requires the original bytes at offsets 0, 74, and 82 all to be zero.

The local overwrite is part of the established behavior. The implementation does not merely compare all three positions without mutation. The overwrite occurs whenever the original offset-0 byte is zero, including paths that later fail validation.

The checks short-circuit:

- Original offset 0 nonzero: offsets 74 and 82 are not examined by this logic.
- Original offset 0 zero but offset 74 nonzero: offset 82 is not examined.
- Any failure in these gates prevents examination of the selector at offset 1.

### Selector and selected-byte checks

If all three leading gates pass, the function reads offset 1 as an unsigned 8-bit value, \(n\), and promotes it to an unsigned 32-bit value.

Validation then requires all of the following:

- \(n\) is nonzero.
- \(n\) is no greater than 33.
- The byte at offset 63 is zero.
- The byte at offset \(2+n\) is zero.

The selector is not interpreted as signed. Values from 128 through 255 are rejected as values above 33, rather than being treated as negative.

The visible branch structure has these access consequences:

- For \(n=0\), validation fails without examining offset 63 or the selected byte.
- For \(n\geq34\), validation fails without examining offset 63 or the selected byte.
- For \(1\leq n\leq33\), offset 63 is examined.
- The selected byte is examined only if offset 63 is zero.

There is no requirement that preceding bytes in the 61-byte region be nonzero, printable, or valid text. The checked zero byte need not be the first zero byte in that region. Interpreting this as validation of a string of exactly \(n\) characters would impose requirements absent from the evidence.

Every validation failure returns the integer value 2 and prevents the final dependency call.

### Constructing the forwarded value

After validation succeeds, bytes at offsets 83 through 86 are interpreted as unsigned bytes and combined into an unsigned 32-bit quantity \(x\):

\[
x =
2^{24}b_{83}
+2^{16}b_{84}
+2^{8}b_{85}
+b_{86}.
\]

Offset 83 is the most significant byte; offset 86 is the least significant byte. This interpretation is independent of the host’s native byte order.

The function computes the forwarded quantity \(q\) using unsigned 32-bit arithmetic:

\[
q =
\left(
\left((x+127)\bmod 2^{32}\right)
\mathbin{\&}
\texttt{0xFFFFFF80}
+128
\right)\bmod 2^{32},
\]

where the addition of 128 applies after masking. Equivalently, first add 127 with 32-bit wraparound, clear the low seven bits, and then add 128 with 32-bit wraparound.

When neither addition overflows, this is the next 128-byte multiple at or above \(x\), plus another 128. In particular:

- \(x=0\) produces 128.
- \(x=1\) produces 256.
- \(x=128\) produces 256.
- \(x=129\) produces 384.

The wrapping arithmetic is material. The function does not check for overflow, saturate the value, or reject a zero result.

### Final dispatch

The function invokes F-b829b770e451 with five arguments, in this positional order:

1. The first input.
2. The second input.
3. The calculated 32-bit quantity \(q\).
4. The third input.
5. The fourth input.

Its result becomes the function’s result without translation. There is no subsequent record check or explicit cleanup call.

## Side effects

The established direct local side effect is the conditional overwrite of scratch offset 0 with the byte from offset 74.

All externally visible side effects are mediated by the dependencies. Their ordering is established:

1. F-e40a3ceddab2 runs first.
2. F-0c21b5cad025 runs only after a zero result from the first dependency.
3. Record inspection and the conditional local overwrite follow only after a zero result from the second dependency.
4. F-b829b770e451 runs only after successful record validation.

No rollback is visible. A validation failure does not undo state changes already made by either prerequisite. A nonzero prerequisite result likewise does not imply that the prerequisite left external state unchanged.

The supplied expression does not establish a portable relative evaluation order among the four reads used to construct \(x\). All four belong to the successful-validation path and precede the final call, but no more specific read order should be required.

The visible body contains no direct access to global state. This does not establish that dependencies preserve globals, preserve caller-owned resources, or refrain from allocating or releasing memory.

## Dependencies

| Opaque ID | Invocation condition | Arguments | Result handling |
|---|---|---|---|
| F-e40a3ceddab2 | Every invocation | Second input; zero | Nonzero propagates immediately; zero permits continuation. |
| F-0c21b5cad025 | First dependency returned zero | Second input; scratch pointer; 128 | Nonzero propagates immediately; zero permits record inspection. |
| F-b829b770e451 | Both prerequisites returned zero and all byte checks passed | First input; second input; calculated quantity; third input; fourth input | Returned unchanged. |

Dependency names do not establish their operational roles. Their memory ownership rules, exceptional behavior, global-state effects, resource consumption, and detailed result meanings remain undocumented.

The value 2 is the locally selected validation-failure result. It is not necessarily unique to that condition: a dependency can also return 2, and the visible interface does not distinguish those cases.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Record examples assume that both prerequisites return zero and provide stable, valid scratch storage.

| Condition | Required behavior |
|---|---|
| F-e40a3ceddab2 returns −7 | Return −7; invoke no later dependency. |
| First prerequisite returns zero; F-0c21b5cad025 returns 9 | Return 9; perform no record validation or final dispatch. |
| Original offset 0 is nonzero | Return 2; do not read offsets 74 or 82 through the validation logic. |
| Offset 0 is zero and offset 74 is nonzero | Copy offset 74 into local offset 0, then return 2 without examining offset 82. |
| Offsets 0 and 74 are zero; offset 82 is nonzero | Return 2 without examining offset 1. |
| All leading gates are zero; selector is 0 | Return 2 without examining offset 63 or a selected byte. |
| All leading gates are zero; selector is 34 or 255 | Return 2 without a dynamically selected access. |
| Selector is 1; offset 63 and offset 3 are zero | Selector-related validation passes. |
| Selector is 33; offset 63 and offset 35 are zero | Selector-related validation passes. |
| Selector is valid but offset 63 is nonzero | Return 2 without examining offset \(2+n\). |
| Selector is valid and offset 63 is zero, but offset \(2+n\) is nonzero | Return 2. |

For the following examples, assume every validation condition passes:

| Bytes at offsets 83–86 | \(x\) | Forwarded \(q\) |
|---|---:|---:|
| 00, 00, 00, 00 | 0 | 128 |
| 00, 00, 00, 01 | 1 | 256 |
| 00, 00, 00, 80 | 128 | 256 |
| 00, 00, 00, 81 | 129 | 384 |
| FF, FF, FF, 80 | 4,294,967,168 | 0 |
| FF, FF, FF, 81 | 4,294,967,169 | 128 |

A calculated value of zero still reaches F-b829b770e451. If that dependency returns 17, the enclosing function returns 17.

## Uncertainties

The following are **documented_only** unknowns, not inferred requirements:

- The application meaning of the four inputs and the record.
- Whether 128 denotes a requested transfer length, capacity, or something else to F-0c21b5cad025.
- The actual writable extent of the scratch allocation and the dependency’s initialization guarantees.
- Whether the checked bytes represent flags, terminators, reserved fields, or another format.
- Whether \(x\) represents a size, position, count, or unrelated quantity.
- Whether the calculated alignment arithmetic is intentional for values near the 32-bit limit.
- Dependency ownership, pointer-retention, global-state, and resource-management behavior.
- Behavior if a dependency fails to return normally or violates its memory contract.

Alignment or size adjustment is an inferred intent, supported by the low-seven-bit mask and addition of 128. It is not an established semantic interpretation of the value.

## Evidence

The supplied static function body establishes the dependency call sequence, early-return conditions, byte checks, conditional scratch overwrite, unsigned byte assembly, wrapping arithmetic, final argument order, and result propagation.

The field offsets above are relative byte displacements derived from the supplied local-storage references. They describe the accesses used by this function, not an independently verified external record schema.

No dependency bodies, external format documentation, runtime observations, or test results were supplied. Accordingly, the acceptance examples specify expected behavior under their stated preconditions; they do not claim observed execution.
