<!-- Trusted provenance: F-1dbebb93bf31; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1dbebb93bf31

## Purpose

F-1dbebb93bf31 conditionally invokes a preparatory dependency, invokes a primary dependency unless preparation fails, updates a caller-supplied state record, and returns a 32-bit success or failure value.

The established contract is structural. The supplied evidence does not establish the domain operation, the meaning of the record fields, or whether the dependencies perform input, output, resource management, or another activity.

There are two failure routes:

- A negative signed 32-bit preparatory result skips the primary operation and the function’s direct state updates, then invokes the failure-path dependency.
- A nonzero 32-bit primary result invokes the failure-path dependency only after two direct state updates.

A zero primary result returns zero after those same state updates. Both failure routes return `0xffffffff` if the failure-path dependency returns normally.

All statements about completion assume that reached memory accesses succeed and invoked dependencies return normally. No exception handling, fault recovery, or rollback is established.

## Interface

### Arguments

| Argument | Established representation | Use |
|---|---|---|
| First | Opaque 32-bit value | Passed unchanged as the second argument to F-e0c58b8811df. |
| Second | Signed 32-bit integer | Passed as the third argument to F-e0c58b8811df; also used in the state adjustment. |
| Third | Signed 32-bit integer used as an address-bearing value | Identifies the state-record base and is passed unchanged to F-b66fa5cebc4a on failure. |

The function does not directly dereference the first argument. Its interpretation as an address, handle, count, or other value is unknown.

There is no explicit argument validation. The function does not reject a negative second argument or explicitly check the third argument for null, alignment, record extent, or access permissions.

### State-record layout

The function accesses these fields relative to the address represented by the third argument:

| Byte offset | Access width | Established role |
|---|---:|---|
| `0x0c` | 32 bits | Control word: initially captured, conditionally replaced, and later updated to set bit 18. |
| `0x14` | 32 bits | Opaque value captured once for dependency arguments. |
| `0x18` | 32 bits | Optional preparatory argument and subsequent arithmetic state field. |

No complete record definition, packing rule, or alignment requirement is supplied. Intervening bytes have no established role.

The accesses actually reached require valid readable or writable storage as applicable. In particular, reaching the primary operation’s return leads to read/write accesses at both `0x0c` and `0x18`. Dependencies may impose additional requirements that are not established here.

### External control value

A separate external value is combined with the retained 32-bit control word to decide whether preparation is required.

The external value’s declaration is absent. Its underlying storage width, type, numeric value, mutability, and intended meaning are therefore unknown. The 32-bit width established for the record’s control word does not establish a 32-bit storage width for this external value.

### Return value

Normal returns are limited to these 32-bit patterns:

| Return pattern | Condition |
|---|---|
| `0x00000000` | The primary dependency returned exactly zero. |
| `0xffffffff` | Preparation returned a negative signed result, or the primary dependency returned any nonzero result. |

The return interface’s signedness is not established. A signed 32-bit consumer may interpret `0xffffffff` as −1; the established contract is the returned bit pattern.

## Behavior

### Initial reads and preparation decision

The function first reads the 32-bit control word at offset `0x0c`, then captures the 32-bit opaque value at offset `0x14`.

It tests whether the bitwise intersection of the retained control word and the external control value is nonzero.

If the intersection is zero:

- F-e1c0335c1ea5 is not invoked.
- The function does not perform a preparatory write to the control field.
- The original retained control word becomes the primary dependency’s fourth argument.

If the intersection is nonzero, the function reads the 32-bit field at offset `0x18` and invokes F-e1c0335c1ea5 with:

1. The retained value from offset `0x14`.
2. The value read from offset `0x18`.

The preparatory result is interpreted as a signed 32-bit integer.

### Preparatory failure

A negative preparatory result selects the failure path immediately.

The function does not:

- Clear control bits or write the retained control word back.
- Invoke F-e0c58b8811df.
- Perform either post-primary state update.

It invokes F-b66fa5cebc4a with the original third argument and returns `0xffffffff` if that dependency returns normally.

This route has no direct record writes by F-1dbebb93bf31. It does not guarantee an unchanged record: preparation and failure handling may have side effects.

### Preparatory success

A zero or positive preparatory result allows execution to continue. The magnitude of that result has no further established use.

The function clears bits 4 and 17 of its retained control word, where bit zero is the least significant bit. The clearing mask is `0xfffdffef`, equivalently clearing `0x00020010`.

It writes the resulting 32-bit word to offset `0x0c`. That same retained value becomes the primary dependency’s fourth argument.

This replacement uses the control word captured before preparation. It does not reload or merge the control field after the preparatory call. Any intervening changes by preparation to that field can therefore be overwritten.

The value captured from offset `0x14` is likewise not refreshed. A preparatory change to that field does not change the corresponding primary argument.

### Primary invocation

F-e0c58b8811df receives four arguments in this order:

1. The originally retained 32-bit value from offset `0x14`.
2. The function’s first argument.
3. The function’s second argument.
4. The retained control word, with bits 4 and 17 cleared only when preparation was required and returned nonnegatively.

The primary result is interpreted as an unsigned 32-bit value.

Its complete value determines the eventual success or failure decision. Its low 31 bits separately contribute to the state adjustment.

### Post-primary control update

Whenever the primary dependency returns normally, the function freshly reads the control field at offset `0x0c`, sets bit 18 using `0x00040000`, and writes the result back.

This update does not use the initial retained control word. It operates on the value present after the primary call, including any changes that dependency made.

At this direct update, all bits other than bit 18 are preserved. No claim is made that a later dependency preserves the result.

### Post-primary arithmetic update

After the control update, the function reads the then-current 32-bit field at offset `0x18` using a signed interpretation and stores a 32-bit adjusted result back to that field.

Let:

- $P$ be the unsigned value of the field’s existing 32-bit pattern.
- $N$ be the mathematical signed value of the second argument.
- $R$ be the unsigned 32-bit primary result.
- $L = R \mathbin{\&} \texttt{0x7fffffff}$.

The stored bit pattern is:

$$
(P + N - L) \bmod 2^{32}.
$$

The modular result is established, not an overflow assumption. Masking the unsigned primary result produces an unsigned 32-bit operand. With the represented 32-bit signed and unsigned integer types:

- The second argument is converted to unsigned 32-bit for subtraction.
- The subtraction is unsigned 32-bit.
- The signed field read is converted to unsigned 32-bit for addition.
- The addition is unsigned 32-bit.
- The stored value is the resulting 32-bit pattern.

Thus, both underflow and overflow wrap modulo $2^{32}$ in this expression. Signed overflow is not an unresolved behavior of this field adjustment.

The primary result’s top bit does not contribute to the adjustment. This does not exclude it from the success/failure decision.

### Final decision

Only after both post-primary updates does the function examine the primary result for the final branch.

- If the complete primary result is zero, the function returns zero without failure handling.
- If any primary-result bit is nonzero, the function invokes F-b66fa5cebc4a with the original third argument, then returns `0xffffffff` if that dependency returns.

No direct rollback precedes failure handling. The failure-path dependency may nevertheless modify or invalidate the updated state.

## Side effects

### Direct access and call order

The established source-level sequence is:

1. Read the control field at `0x0c`.
2. Read the opaque field at `0x14`.
3. Evaluate the preparation condition using the retained control word and external value.
4. If preparation is required, read `0x18` and invoke F-e1c0335c1ea5.
5. If preparation returns negatively, invoke F-b66fa5cebc4a and return failure.
6. If preparation returns nonnegatively, write the cleared retained control word to `0x0c`.
7. Invoke F-e0c58b8811df.
8. Read and write `0x0c` to set bit 18.
9. Read and write `0x18` for the modular adjustment.
10. For a nonzero primary result, invoke F-b66fa5cebc4a.
11. Return the selected 32-bit pattern.

The function itself never writes offset `0x14`. Dependencies are not known to preserve that field.

This sequence describes the represented operations. It does not establish atomicity, synchronization, volatile-access guarantees, or a machine-level ordering contract for concurrent observers.

### Ownership and lifetime

No explicit allocation or deallocation appears in the function.

Passing the third argument to F-b66fa5cebc4a does not establish whether that dependency inspects, resets, retains, releases, or otherwise changes the record. Its role as a failure-path dependency is not evidence of ownership transfer.

No dependency is assumed to preserve globals, aliased storage, or resource lifetime. Every direct access requires the relevant storage to remain valid when that access is reached.

### Errors and partial progress

Failure returns do not identify which failure route occurred.

A preparatory failure prevents the primary call and all direct record writes by this function. A primary failure follows both post-primary updates.

The function does not return the preparatory status or primary result to its caller. The failure-path dependency’s result is unused.

## Dependencies

### F-e1c0335c1ea5

Conditionally invoked when the retained control word’s intersection with the external value is nonzero.

- Inputs: captured offset-`0x14` value and the offset-`0x18` value read for this call.
- Result interpretation: signed 32-bit; negative selects failure, nonnegative permits continuation.
- Unknowns: substantive operation, memory and global effects, ownership effects, additional failures, and whether it returns normally.

### F-e0c58b8811df

Invoked unless required preparation returns negatively.

- Inputs: captured offset-`0x14` value, first argument, second argument, and retained possibly-cleared control word.
- Result interpretation: unsigned 32-bit.
- Zero selects success after updates; nonzero selects failure handling after updates.
- Low 31 bits participate in the modular state adjustment.
- Unknowns: operation, result units or meaning, indirect side effects, ownership effects, and termination behavior.

### F-b66fa5cebc4a

Invoked on either failure route.

- Input: the original third argument.
- Result: unused.
- Unknowns: substantive effects, including whether it modifies or releases the record, changes globals, or returns normally.

The terms “preparatory,” “primary,” and “failure-path” identify call positions, not established domain semantics.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Each assumes valid reached accesses and normally returning dependencies. Final record contents are asserted only when subsequent dependencies are specified to preserve them.

### Preparation bypassed; primary success

Suppose the initial control word has no intersection with the external value, the second argument is 6, and the primary dependency returns zero while leaving control at `0x00000008` and the field at `0x18` at 10.

Then:

- Preparation is skipped.
- The primary dependency receives the initial retained control word.
- The direct control update writes `0x00040008`.
- The arithmetic update writes 16.
- Failure handling is skipped.
- The function returns zero.

### Successful preparation overwrites a changed control field

Suppose the initial control word is `0x00020030`, preparation is required, and preparation returns nonnegatively after changing the control field.

The function writes `0x00000020` to `0x0c` before invoking the primary dependency and passes `0x00000020` as its fourth argument. The preparatory control-field change is not merged.

If preparation also replaces the field at `0x14`, the primary dependency still receives its originally captured value.

### Preparatory failure

Suppose preparation is required and returns −1.

The primary dependency is not invoked. The clearing write and both post-primary updates are skipped. F-b66fa5cebc4a receives the original third argument. If it returns, the function returns `0xffffffff`.

Record preservation is not implied.

### Ordinary primary failure

Suppose the second argument is 12, the primary result is 5, and the field at `0x18` contains 100 immediately after the primary call.

The function first sets bit 18 of the current control field, then writes 107 to `0x18`, then invokes F-b66fa5cebc4a. It returns `0xffffffff` if that dependency returns.

### Top-bit-only primary result

Suppose the primary result is `0x80000000`.

Its low 31 bits are zero, so the arithmetic update adds the second argument modulo $2^{32}$. The complete result is nonzero, so failure handling still occurs and the normal return is `0xffffffff`.

### Crossing the signed boundary

Suppose the primary result is zero, the second argument is 1, and the post-primary field at `0x18` contains `0x7fffffff`.

The arithmetic update stores `0x80000000`. This is a defined unsigned 32-bit result, not unresolved signed overflow. The function returns zero after both updates.

### Modular subtraction with a negative argument

Suppose the second argument is −1, the primary result is 1, and the post-primary field at `0x18` contains zero.

The adjustment stores `0xfffffffe`, representing the low 32 bits of $0-1-1$. Failure handling follows because the primary result is nonzero.

## Uncertainties

The following remain **documented_only**:

- The external control value’s declaration, storage width, type, numeric value, and intended meaning.
- Domain meanings of the arguments and record fields.
- Whether the second argument represents a size, count, displacement, or another quantity.
- Whether the primary result’s low 31 bits represent a residual quantity.
- Address-formation overflow, invalid-address, alignment, and memory-fault behavior.
- Concurrency, atomicity, and synchronization requirements.
- Dependency effects on globals, aliased memory, ownership, lifetime, and control flow.
- Any standardized caller interpretation of the failure pattern.

The offset-`0x18` adjustment is not an arithmetic uncertainty: its stored result is modulo $2^{32}$.

The arithmetic shape is compatible with updating a position or count by a requested amount minus a residual amount. That is inferred intent only; it does not establish units or dependency semantics.

## Evidence

This contract uses only the supplied static function representation and opaque dependency mapping.

Established evidence includes argument representations, 32-bit field accesses, field offsets, retained versus fresh reads, conditional call paths, fixed bit masks, update order, unsigned arithmetic conversions, result comparisons, and returned patterns.

The external value’s declaration, dependency internals, callers, full record definition, runtime environment, and execution traces were not supplied. Their absence is not treated as evidence of preservation, ownership, or domain meaning.

No runtime tests have been performed.
