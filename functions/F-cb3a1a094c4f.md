<!-- Trusted provenance: F-cb3a1a094c4f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cb3a1a094c4f

## Purpose

F-cb3a1a094c4f conditionally retrieves a single byte through a dependent lookup. It first checks an adjusted selector against a 32-bit bound stored in caller-provided state. If the check passes, it invokes F-f3f7ca2d22ef. A nonzero result from that dependency supplies the address from which the byte is read.

The function returns zero when the bound check fails or when the dependency returns zero. A successful byte read can also return zero. Consequently, the return value alone cannot distinguish those three outcomes.

The apparent intent is a one-based lookup with a zero-based dependency interface. That interpretation is inferred, not an established description of the containing object or its data model.

## Interface

The function has two inputs and a one-byte output.

| Item | Contract |
|---|---|
| First input | An address-bearing value identifying the base of readable state. The supplied representation declares it as a 32-bit integer rather than a typed pointer. |
| Second input | A 32-bit integer selector. Its bit pattern participates in an unsigned range check after subtraction of one. |
| Return | One byte: either the byte read through a nonzero dependency result or zero on either explicit fallback path. |

The relevant relative layout is:

| Address relative to the first input | Use |
|---|---|
| Byte offset 40 | Address passed as the dependency’s first argument. |
| Byte offset 44 | Location of the unsigned 32-bit bound read before any dependency call. |
| Byte offset 40 from a nonzero dependency result | Location of the one-byte result read after the dependency returns. |

These offsets establish access locations, not complete object layouts. They do not establish allocation sizes, field names, element types, or ownership relationships. In particular, the address passed to the dependency is an interior address computed directly from the first input; it is not a pointer loaded from offset 40.

For normal completion, the bound location must support the required 32-bit read. If the dependency is called, its arguments must meet its own requirements. If it returns nonzero, the resulting address plus 40 must support a one-byte read.

The function does not explicitly validate any of those memory preconditions.

## Behavior

### Bound evaluation

The function first reads an unsigned 32-bit value at byte offset 44 from the first input. Call this value the bound.

For the range comparison, the selector is reduced by one using 32-bit unsigned arithmetic. Thus, the comparison candidate is the selector’s 32-bit bit pattern minus one, modulo \(2^{32}\). The comparison is strictly less than, not less than or equal to.

The consequences are:

- A zero bound rejects every selector.
- A selector of zero produces comparison candidate \(2^{32}-1\), which cannot be less than any unsigned 32-bit bound.
- A selector of one produces candidate zero and passes exactly when the bound is nonzero.
- For ordinary positive selectors, a selector equal to the bound passes; a selector one greater than the bound fails, where that increment is representable.
- Negative signed selectors are not rejected by a separate sign check. Their acceptance depends on the unsigned candidate and the stored bound.

If the comparison fails, the function returns zero immediately. It does not invoke F-f3f7ca2d22ef and does not perform the final byte read.

### Dependency invocation

If the comparison passes, the function calls F-f3f7ca2d22ef once with:

1. The first input’s address advanced by 40 bytes.
2. The selector reduced by one.

For selectors whose subtraction is representable as a signed 32-bit integer, the second argument is unambiguous. For example, selector one supplies zero, and selector three supplies two.

There is a representation-level uncertainty at the most negative signed 32-bit selector: its separate decrement expression is outside the signed 32-bit range. The unsigned range comparison remains determinate, but the supplied reconstruction alone does not establish the original instruction or language semantics for forming that dependency argument. This case is documented under Uncertainties rather than assigned an invented overflow policy.

The dependency is not called speculatively. Its invocation depends on the bound value already read. No second bound read or post-call bound check appears in the supplied body.

### Dependency result and return

If F-f3f7ca2d22ef returns zero, this function returns a zero byte without reading at an address derived from that zero result.

If the dependency returns nonzero, this function treats the result as an address-bearing value and reads exactly one byte at offset 40 from it. That byte is returned unchanged.

There is no explicit second lookup, retry, byte transformation, status encoding, or validation of the nonzero result’s address.

A nonzero dependency result does not guarantee a nonzero function result. A byte containing zero is returned as zero.

## Side effects

The function body contains no explicit stores, allocations, releases, or ownership transfers.

Its directly established memory-access sequence is:

1. Read the unsigned 32-bit bound from the first input’s base plus 44.
2. If the comparison passes, invoke the dependency.
3. If that invocation returns nonzero, read one byte from the returned base plus 40.

These are the accesses visible in the supplied representation. They are not a claim about additional reads or writes performed inside F-f3f7ca2d22ef, nor a hardware-level memory-ordering guarantee.

The dependency may change memory or global state; its behavior is not supplied. This function does not preserve or restore such state explicitly. If the dependency changes the byte later read through its returned address, the final read occurs after those changes. If it changes the bound, this function does not reread the bound before returning.

No release of the returned address is visible. That absence establishes only that this body contains no explicit release; it does not establish that the address is owned, borrowed, permanent, or safe to retain.

Invalid addresses are not converted into a documented error return. The source supplies no local exception handling, fault recovery, synchronization, or lifetime protection.

## Dependencies

### F-f3f7ca2d22ef

This is the only explicit function dependency.

Its established caller-facing role here is:

- It is invoked at most once.
- It is invoked only after the unsigned bound comparison succeeds.
- Its first argument is the first input advanced by 40 bytes.
- Its second argument represents the selector decremented by one, subject to the extreme signed-subtraction uncertainty.
- Its zero result selects the zero-return path.
- Its nonzero result supplies the base for a one-byte read at offset 40.

Nothing supplied establishes whether this dependency performs a container lookup, allocates storage, releases storage, mutates shared state, or returns an interior address. Those behaviors must remain unknown.

Inferred intent: the caller appears to use the dependency to resolve an indexed entry in a structure beginning at offset 40. This interpretation is not required to describe the caller’s established behavior.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reports of executed tests. Each assumes valid memory and normal dependency completion wherever applicable.

| Conditions | Required behavior |
|---|---|
| Bound is zero; selector is one | Return zero without calling the dependency. |
| Bound is five; selector is zero | Return zero without calling the dependency, because the unsigned candidate is \(2^{32}-1\). |
| Bound is three; selector is four | Return zero without calling the dependency, because candidate three is not less than bound three. |
| Bound is three; selector is three; dependency returns zero | Call the dependency with the first input advanced by 40 and adjusted selector two; return zero without a final byte read. |
| Bound is three; selector is one; dependency returns a valid nonzero base whose byte at offset 40 is 165 | Call with adjusted selector zero; return byte value 165. |
| Same successful lookup, but the byte is zero | Return zero. Do not infer lookup failure from this return value. |
| Bound is \(2^{32}-1\); selector is signed negative one | The unsigned candidate is \(2^{32}-2\), so invoke the dependency with adjusted selector negative two. |
| Dependency changes the bound before returning a valid nonzero base | Do not repeat the bound check; read the byte through the returned base. |

An invalid first input is not an accepted “not found” case. The function attempts the bound read before it can select either normal return path.

## Uncertainties

The following items are **documented_only**:

- **Complete ABI:** the representation shows 32-bit integer inputs and address-bearing results, but does not independently establish the platform’s full pointer ABI or calling convention.
- **Address arithmetic limits:** wraparound, invalid-address behavior, and alignment requirements are not established.
- **Extreme signed decrement:** dependency-argument formation for the most negative signed 32-bit selector requires additional instruction-level or language-semantics evidence.
- **Object identity and layout:** only the stated offsets and read widths are established.
- **Dependency behavior:** mutation, allocation, release, ownership, and global-state preservation are unknown.
- **Concurrency:** no synchronization or coherent-snapshot guarantee is established.
- **Failure outside explicit branches:** memory faults and abnormal dependency completion have no specified recovery behavior here.

## Evidence

The supplied static function body establishes one unsigned 32-bit bound read, a strict unsigned comparison against the selector decremented by one, one conditional call to F-f3f7ca2d22ef, a zero-result check, and one conditional byte read at offset 40 from a nonzero dependency result. Both explicit fallback paths return zero.

The one-based indexing interpretation is inferred from the decrement and comparison, not from supplied type or domain documentation.

No dependency body, runtime trace, ABI specification, or execution results were supplied. No runtime tests have been performed.
