<!-- Trusted provenance: F-8becc2a0337c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8becc2a0337c

## Purpose

F-8becc2a0337c performs an index-related validity check followed by an ownership-sensitive state transition on an embedded object.

Its established behavior is to:

- Invoke an entry helper on an embedded control region.
- Reject an unsigned index outside 0–28, or an accepted-range index whose corresponding table word is zero, through a nonreturning dependency.
- Invoke another dependency before examining the embedded control state.
- Optionally invoke a special-state dependency and propagate its nonzero result.
- Compare a stored owner word with an identity obtained from a dependency.
- On an owner match, either decrement a 16-bit count without further action or clear ownership and invoke a final dependency.
- Return a 32-bit result indicating the branch outcome or forwarding a dependency’s result.

**Inferred intent:** The owner comparison, selected-mode count decrement, ownership clearing, and final call resemble release of an owned synchronization object, possibly with recursive ownership. That interpretation is not required by this contract. The supplied evidence does not establish a public object type, synchronization guarantees, or the precise purpose of any dependency.

## Interface

### Inputs

The recovered interface exposes four arguments:

| Argument | Width | Contract |
|---|---:|---|
| Object base | 32 bits | Address used to derive the embedded control address, owner field, mode field, count field, final-operation address, and indexed table entry. |
| Index | 32 bits, unsigned | Compared against 28 before the indexed table word is read. Negative-looking 32-bit bit patterns are therefore out of range. |
| Auxiliary value A | 32 bits | Forwarded to the special-state dependency if that dependency is invoked. No direct interpretation is visible. |
| Auxiliary value B | 32 bits | Forwarded to the special-state dependency if that dependency is invoked. No direct interpretation is visible. |

The recovered special-state call also exposes four additional machine-context values whose origins are not established. They are not established as ordinary caller-supplied arguments to F-8becc2a0337c.

### Return value

The result is a 32-bit integer:

- **26** if the embedded control address is null at the explicit null check.
- **5** if the owner word does not match the identity dependency’s result.
- **0** if the owner matches and the selected-mode count decrement produces a nonzero 16-bit result, without a later final-operation call.
- The exact nonzero result of the special-state dependency when that call returns nonzero.
- The exact result of the final-operation dependency when that call occurs.

These values describe return paths, not a complete error-code taxonomy. A dependency can return a value numerically equal to one of the locally selected results.

### Visible object layout

All offsets below are byte offsets from the object base. They are relative layout requirements, not a declaration of the complete object.

| Offset | Width | Visible use |
|---|---:|---|
| `0x114` | 32 bits | First word of the embedded control region; compared with an external discriminator. Its address is passed to two dependencies. |
| `0x118` | 32 bits | Owner word; read after the identity call and cleared on the final-operation path. |
| `0x122` | 16 bits | Mode-containing field; captured before the identity call. |
| `0x126` | 16 bits | Count-containing field; read, decremented, and written only on the owner-matched selected-mode path, or overwritten with zero on the final-operation path. |
| `0x128` | Unspecified extent | Address passed to the final-operation dependency. |
| `0x154 + 8 × index` | 32 bits | Indexed validity word; must be nonzero for an accepted-range index. |

The indexed record stride is eight bytes, but only its first four bytes are visibly examined. No interpretation of the remaining four bytes is established.

### Preconditions and validity

Ordinary returning execution requires that all reached memory accesses and dependency calls be valid in the execution environment. In particular:

- The entry helper is invoked before index rejection.
- An in-range index causes a four-byte read from its table entry before the explicit embedded-address null check.
- Later paths require access to the indicated control fields and to whatever storage the dependencies themselves access.

The function does not establish an allocation size, alignment policy, lifetime guarantee, or protection against concurrent mutation. It performs no explicit check that the object base itself is nonnull.

## Behavior

### 1. Entry helper and index validation

The function derives an embedded control address at object-base offset `0x114` and invokes an entry helper with that address.

After that helper returns, it evaluates the unsigned index:

- If the index exceeds 28, it invokes F-a817d8d58c10 without reading the indexed table word.
- Otherwise, it reads the 32-bit word at offset `0x154 + 8 × index`. If the word is zero, it invokes F-a817d8d58c10.
- A nonzero word permits execution to continue.

F-a817d8d58c10 is marked nonreturning in the supplied evidence. No normal return result from F-8becc2a0337c is established for these rejection paths.

For accepted-range indices, multiplication by eight produces offsets from 0 through 224; the resulting table-word offsets range from `0x154` through `0x234`. The range check therefore prevents a large index from reaching that indexed read. It does not protect the preceding entry-helper call.

### 2. Post-validation dependency and null check

After successful index validation, the function invokes F-c11907680298.

Only after that call does it check whether the previously derived embedded control address is null. If it is null, the function returns 26.

This is an address check, not a test of the 32-bit word stored at that address. A zero control word does not itself select return value 26.

The check is also not an early object-validation guarantee: the entry helper and, for an in-range index, the table read precede it.

### 3. Optional special-state call

For a nonnull embedded control address, the function reads its first 32-bit word and compares it with an external 32-bit discriminator.

If the values differ, it proceeds without the special-state call.

If they match, it invokes F-621dae46a146. The recovered call exposes:

- The embedded control address.
- A zero-valued argument.
- Auxiliary values A and B.
- Four additional machine-context values whose origin and semantic role are unknown.

If this dependency returns a nonzero 32-bit integer, F-8becc2a0337c immediately returns that value. The later mode read, identity call, owner comparison, count transition, and final operation do not occur.

If the dependency returns zero, execution continues. Zero is therefore a continuation result here, not necessarily the final result of the function.

### 4. Mode capture and owner comparison

The function reads the 16-bit mode-containing field at offset `0x122` and retains that value locally.

It then invokes F-196812b4b122 to obtain a 32-bit identity value. After the call returns, it reads the owner word at offset `0x118` and compares it with that returned identity.

This ordering matters:

- The mode decision uses the field value captured **before** the identity call.
- The owner comparison uses the owner word read **after** the identity call.

No assumption that the identity dependency preserves object memory or globals is justified.

If the owner word differs from the returned identity, the function returns 5. On that path, it performs no direct count write, owner clear, or final-operation call. Earlier dependencies may nevertheless have produced side effects.

### 5. Owner-matched count handling

If the owner matches, the captured mode value determines whether a decrement is attempted.

The selected mode is exactly the case where bits 5–4 of the captured 16-bit value are binary `10`. Bits 3–0 and bits 15–6 do not affect this decision. Equivalently, the selected mode has bit 5 set and bit 4 clear.

For that mode:

1. The function reads the 16-bit count field at offset `0x126`.
2. It subtracts one.
3. It stores the narrowed 16-bit result back into the count field.
4. It tests that resulting 16-bit value for zero.

Under the 16-bit narrowing behavior represented by the supplied evidence, this is a modulo-65,536 decrement for stored bits. Thus an initial zero becomes `0xFFFF`; it does not remain zero.

If the stored decrement result is nonzero, the function returns 0 without clearing the owner or invoking the final-operation dependency.

If the stored decrement result is zero, it proceeds to the final-operation path.

For every other mode, the decrement is skipped entirely and execution proceeds directly to the final-operation path.

### 6. Final-operation path

On this path, direct writes and the final call occur in this order:

1. Write a 16-bit zero to the count field at offset `0x126`.
2. Write a 32-bit zero to the owner word at offset `0x118`.
3. Invoke F-259d2fe281b0 with the address at object-base offset `0x128`.
4. Return that dependency’s 32-bit result unchanged.

When a selected-mode count of one is decremented, the count is first written as zero by the decrement and then written as zero again by this path. These are two visible stores in the supplied sequence.

The function does not visibly restore either field if the final dependency returns a nonzero result. Conversely, the values of those fields after the dependency returns cannot be guaranteed without knowing that dependency’s effects.

## Side effects

### Direct memory effects

The visible body directly writes only the count and owner fields:

- A 16-bit decrement write on the owner-matched selected-mode path.
- A 16-bit zero write followed by a 32-bit owner-clear write on the final-operation path.

It visibly reads the indexed validity word, control discriminator word, mode field, owner word, and conditionally the count field.

No direct allocation, deallocation, table update, or auxiliary-value dereference is visible. This does not establish that dependencies avoid those operations.

### Ownership and lifetime

The function explicitly clears the owner word before invoking the final-operation dependency. It does not visibly transfer a heap allocation, free the object, or retain either auxiliary value itself.

Dependencies receive interior addresses. Their memory ownership, retention, destruction, and release behavior remain unknown. The final-operation call must not be equated with deallocation solely from its placement after the owner clear.

### Ordering limits

The sequence above describes the visible operation order. The supplied evidence does not establish atomic accesses, memory barriers, inter-thread ordering guarantees, or safe concurrent use.

Dependencies can affect later observations. The saved mode value is not refreshed after the identity call, while the owner and any subsequently used count are read later.

## Dependencies

| Dependency | Established use | Unestablished aspects |
|---|---|---|
| Entry helper, opaque ID unavailable | Called first with the embedded control address. | Exact interface, result significance, side effects, and synchronization behavior. |
| F-a817d8d58c10 | Called for an out-of-range index or a zero indexed validity word; marked nonreturning. | Whether it terminates, traps, throws, transfers control, or performs other effects. |
| F-c11907680298 | Called after successful index validation and before the embedded-address null check. | Arguments, result significance, and side effects. |
| F-621dae46a146 | Conditionally called when the control word matches the external discriminator; nonzero result is propagated. | Full calling convention, meaning of auxiliary context, and memory/global effects. |
| F-196812b4b122 | Supplies the value compared with the owner word. | Whether the value denotes a thread, task, process, or another identity; side effects. |
| F-259d2fe281b0 | Called after count and owner are cleared, with the interior address at offset `0x128`; result is propagated. | Operation semantics, ownership effects, and mutations during the call. |

The external discriminator is a data dependency. Its value, mutability, and initialization rules are not supplied.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, required memory accesses are valid, reached dependencies return normally, the embedded address is nonnull, and the special-state path either is not selected or returns zero.

1. **Unsigned rejection:** An index with bit pattern `0xFFFFFFFF` exceeds 28. The entry helper occurs first, followed by F-a817d8d58c10. No indexed table read occurs.

2. **Upper accepted boundary:** Index 28 selects the 32-bit word at offset `0x234`. A nonzero word passes validation; zero selects the nonreturning rejection dependency.

3. **Lower rejected boundary above the range:** Index 29 invokes the rejection dependency without an indexed table read.

4. **Special-state early result:** If the control word matches the discriminator and F-621dae46a146 returns 9, the function returns 9 without reading the mode field or invoking the identity dependency.

5. **Owner mismatch:** If F-196812b4b122 returns 7 and the subsequently read owner word is 8, the function returns 5. It performs no direct count write or owner clear.

6. **Selected mode with remaining count:** With captured mode `0x0020`, matching ownership, and count 3 at the reached count read, the function stores 2 and returns 0. It does not invoke F-259d2fe281b0.

7. **Selected mode reaching zero:** With captured mode `0x002F`, matching ownership, and count 1, the function writes count zero, writes count zero again, clears the owner, and invokes F-259d2fe281b0. The dependency’s result is the function’s result.

8. **Zero count wraps:** With captured mode `0x0020`, matching ownership, and count zero, the decrement stores `0xFFFF`. The function returns 0 without the final-operation call.

9. **Other mode bypasses decrement:** With captured mode `0x0030` and matching ownership, the function clears count and owner and invokes the final dependency regardless of the initial count.

10. **Mutation during identity lookup:** If the mode was captured as `0x0020` and the identity dependency changes the stored mode to another value, the captured selected mode still governs this invocation. An owner change during that call affects the later owner comparison.

11. **Final dependency reports failure:** If the final dependency returns a nonzero result, that result is propagated. No local rollback follows the earlier count and owner clears.

## Uncertainties

The following are **documented_only**:

- Exact calling conventions and complete signatures of dependencies, particularly the additional context exposed at the special-state call.
- The entry helper’s opaque identity.
- Object allocation bounds, alignment, byte order, lifetime, and concurrency model.
- The discriminator’s value and provenance.
- Semantic names for the mode, owner, count, and final-operation region.
- Whether dependencies mutate object fields, globals, or referenced memory.
- The mechanism used by the nonreturning rejection dependency.
- Whether the embedded-address null branch is reachable under valid object construction.
- Overflow and fault behavior for invalid object-base address arithmetic in the actual execution environment.

The stored decrement and zero comparison establish the stated 16-bit transition under the represented narrowing behavior. They do not establish a signed-count validity constraint or a separate underflow error.

## Evidence

The contract is based solely on the supplied static function representation and dependency-ID mapping.

Established evidence includes field offsets and widths, the unsigned index bound, conditional table access, dependency ordering, special-result propagation, mode capture timing, owner comparison, 16-bit count transition, ordered clearing writes, and final-result propagation.

Synchronization and recursive-release terminology is inferred intent only. Dependency semantics and unshown object invariants remain unknown.

No runtime tests have been performed.
