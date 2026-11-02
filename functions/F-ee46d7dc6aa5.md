<!-- Trusted provenance: F-ee46d7dc6aa5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ee46d7dc6aa5

## Purpose

F-ee46d7dc6aa5 conditionally changes a referenced state object and returns the object’s current 32-bit value at byte offset 20.

A 16-bit field at object offset 28 gates the operation. If bit `0x0400` is already set, the function performs no dependency calls or explicit writes and returns the value at offset 20. Otherwise, a zero second argument selects one dependency and resets selected fields; a nonzero second argument selects another dependency, sets the gate bit, copies a word from the enclosing record, and conditionally invokes an external dependency.

This is an established static description, not a claim about the application-level meaning of the fields. Interpreting the two branches as deactivation and activation is plausible intent only. In particular, neither the returned word nor the word conditionally passed to the external dependency is established to be a pointer, handle, status, or resource identifier.

## Interface

### Inputs

The function has two inputs:

1. **An enclosing-record reference.** The supplied representation treats this as a pointer to 32-bit words. The function reads:
   - A 32-bit pointer representation at byte offset 48.
   - A 32-bit word at byte offset 0, but only on the initially ungated, nonzero-argument path, after the first dependency returns.

2. **A signed 32-bit integer.** The function distinguishes zero from nonzero. For a nonzero value, it forwards the original value to both applicable dependency calls without visible arithmetic or normalization. Negative values select the same branch as positive nonzero values.

The 32-bit value read from enclosing-record offset 48 identifies the state object. That object reference is captured before the gate is inspected. The function does not visibly reacquire it from the enclosing record after dependency calls.

### Relevant object layout

Offsets below are measured in bytes from the captured state-object reference.

| Offset | Width | Observed use |
|---|---:|---|
| 0 | 32 bits | Receives the enclosing record’s first word on the nonzero-argument path. |
| 20 | 32 bits | Returned on every normal path; cleared on the zero-argument path; inspected and possibly forwarded on the nonzero-argument path. |
| 28 | 16 bits | Supplies the initial gate bit and receives an exact branch-specific replacement value. |

The relevant enclosing-record fields are:

| Offset | Width | Observed use |
|---|---:|---|
| 0 | 32 bits | Source of the conditional copy into object offset 0. |
| 48 | 32 bits | State-object pointer representation. |

These are partial layouts, not complete structure definitions. No significance is established for intervening bytes.

### Output

On normal completion, the function returns the 32-bit word read from captured-object offset 20 at the end of the selected path.

No signedness, success convention, or ownership interpretation is established for the return value. In particular, it is not necessarily a Boolean.

### Preconditions

The caller must provide memory and execution conditions sufficient for the reads, writes, and calls reached on the selected path:

- The enclosing record must support the initial 32-bit read at offset 48.
- The captured object must support the 16-bit read at offset 28 and the final 32-bit read at offset 20.
- Ungated paths require the relevant object fields to be writable.
- The nonzero ungated path additionally requires the enclosing record’s first word to remain readable when the copy occurs.
- Dependency-specific preconditions must hold, but are not supplied.

There are no visible null checks, bounds checks, lifetime checks, or error conversions. Required alignment and the consequences of invalid accesses are not established by this evidence.

## Behavior

### Initial selection

The function first reads the object reference from enclosing-record offset 48. It then reads the object’s 16-bit field at offset 28 and tests bit `0x0400`.

Only this bit determines whether the body is entered. Other bits in that field do not independently inhibit execution.

If the bit is set, the function skips both argument-dependent branches and returns the current word at object offset 20. The second argument has no further effect on this path.

If the bit is clear, the function chooses a branch according to whether the second argument is zero. The gate is not visibly tested again after any dependency returns.

### Ungated object with a zero second argument

The following effects occur in order:

1. F-29b5e86301ff is invoked with the captured object reference.
2. After that dependency returns normally, the function writes the exact 16-bit value `0x0001` at object offset 28.
3. It writes the exact 32-bit value zero at object offset 20.
4. It reads object offset 20 for the return value.

The field write is a replacement, not a bitwise update. Bits previously present in the low 16-bit field are discarded, including any changes made there by the dependency before it returned.

The explicit write at offset 28 affects two bytes. It is not evidence of a four-byte store or of clearing the adjacent two bytes.

Under valid memory conditions and without an intervening external modification, this path returns zero. The dependency’s own result, if any, is not used as the function’s return value.

### Ungated object with a nonzero second argument

The following effects occur in order:

1. F-05077d384bfe is invoked with:
   - The captured object reference.
   - The original signed 32-bit second argument.
   - A zero third argument.
2. After that dependency returns normally, the function writes the exact 16-bit value `0x0400` at object offset 28.
3. It reads the first 32-bit word of the enclosing record and copies that value to object offset 0.
4. It inspects the 32-bit word at object offset 20.
5. If that word is nonzero, it invokes an unmapped external dependency with the value from object offset 20 and the original second argument.
6. After any such call returns, it reads object offset 20 for the return value.

The width and semantic type of the zero third argument are not independently established beyond the supplied call representation.

The write of `0x0400` replaces the entire 16-bit field; it does not preserve other flag bits. The copy into object offset 0 occurs after this replacement and before the conditional external call.

F-05077d384bfe may change the value at offset 20 before the conditional test. Consequently, the decision to invoke the external dependency is based on the value present after that dependency and the subsequent explicit writes, not necessarily the value present on entry.

The final return value is read after the external dependency returns. It is therefore not guaranteed to equal the nonzero value that triggered that call.

### Timing and aliasing

The object reference is captured once. If a dependency changes enclosing-record offset 48, the subsequent explicit object operations still use the captured reference.

Conversely, the enclosing record’s first word is read later, after F-05077d384bfe returns and after the flag replacement. It is not an entry-time snapshot. Changes to that word before the copy can affect what is copied.

No non-aliasing guarantee is supplied. If the enclosing record and object overlap, the ordered operations must be interpreted against that shared storage. For example, a preceding write may affect the later source read if those locations overlap. The evidence does not justify treating the enclosing record and object as independent allocations.

These are logical source-level operation orders. They do not establish instruction-level memory transactions, synchronization, atomicity, or safe behavior under concurrent mutation.

## Side effects

The function’s explicit writes are limited to these branch-dependent operations:

| Initial gate | Second argument | Explicit writes |
|---|---|---|
| Set | Any | None |
| Clear | Zero | Object offset 28 receives 16-bit `0x0001`; offset 20 receives 32-bit zero. |
| Clear | Nonzero | Object offset 28 receives 16-bit `0x0400`; offset 0 receives the enclosing record’s first 32-bit word. |

The function contains no explicit allocation, deallocation, ownership transfer, or reference-count operation. That observation does not establish that the dependencies lack such effects.

Dependency effects on globals, object fields, other reachable memory, or external state are unknown. They must not be assumed absent or rolled back.

The gate does not provide a general postcondition that remains true regardless of dependency behavior:

- On the zero branch, the explicit flag replacement follows the only dependency call.
- On the nonzero branch, the conditional external call follows the flag replacement and may have undocumented effects.
- No post-call flag repair is visible after that external call.

If a dependency fails to return normally, later operations on that path are not established to occur. The function supplies no visible recovery or rollback mechanism.

## Dependencies

### F-29b5e86301ff

Called exactly once on a normally progressing path when the initial gate is clear and the second argument is zero.

Its sole visible argument is the captured object reference. Its implementation, return contract, memory effects, lifetime effects, and error behavior are unknown. Any returned value is ignored.

The caller performs the flag replacement and offset-20 clearing only after this dependency returns normally.

### F-05077d384bfe

Called exactly once on a normally progressing path when the initial gate is clear and the second argument is nonzero.

Its visible arguments are the captured object reference, the unchanged second argument, and zero. Any returned value is ignored.

Its effects may influence the later enclosing-record copy and the later conditional external invocation. No guarantee is supplied that it preserves either record, globals, or allocation lifetimes.

### Unmapped external dependency

No opaque function ID for this dependency is supplied. Its identity remains documented_only rather than receiving a fabricated mapping.

It is invoked only on the initially ungated, nonzero-argument path, and only if object offset 20 is nonzero when inspected at that stage.

Its visible arguments are the offset-20 value and the unchanged second argument. Any returned value is ignored. The outer function instead reads object offset 20 again after normal return.

The dependency’s name, semantics, ownership behavior, and failure behavior are unknown.

## Acceptance examples

These examples are conditional static acceptance criteria, not results of runtime tests. Unless stated otherwise, they assume valid memory, normal dependency returns, and no unrelated concurrent modifications.

1. **An already-set gate suppresses work.**  
   If the initial 16-bit field is `0x0401` and object offset 20 contains `0x12345678`, either zero or nonzero input returns `0x12345678`. None of the dependencies is called, and there are no explicit writes.

2. **Other flag bits do not suppress the zero branch.**  
   If the initial field is `0x0002` and the second argument is zero, F-29b5e86301ff is called. Following normal return, the field becomes exactly `0x0001`, offset 20 becomes zero, and the function returns zero.

3. **The zero branch overwrites dependency changes to its two target fields.**  
   If F-29b5e86301ff changes the flag field to `0xFFFF` and offset 20 to a nonzero value before returning, the subsequent explicit writes replace those values with `0x0001` and zero.

4. **A nonzero argument is forwarded unchanged.**  
   With an initially clear gate and a second argument of −1, F-05077d384bfe receives −1 as its second argument and zero as its third. If the external dependency is subsequently invoked, it also receives −1 as its second argument.

5. **A zero value after preparation suppresses the external call.**  
   If object offset 20 is zero after F-05077d384bfe and the subsequent explicit writes, the external dependency is skipped. Without another modification, the return value is zero.

6. **A dependency can enable the external call.**  
   If offset 20 is zero on entry but F-05077d384bfe changes it to `0x80`, the later test invokes the external dependency, provided the value remains `0x80` until that test and argument read.

7. **The returned value can differ from the external-call argument.**  
   If the external dependency is invoked with `0x80` and changes captured-object offset 20 to `0x90` before returning, the function returns `0x90`.

8. **The copied enclosing-record word is read late.**  
   If F-05077d384bfe changes the enclosing record’s first word from `0x11` to `0x22`, and no intervening aliasing write changes it again, object offset 0 receives `0x22`.

9. **Changing the enclosing object reference does not retarget this invocation.**  
   If F-05077d384bfe replaces enclosing-record offset 48 with another valid object reference while leaving the captured object valid, the remaining object writes and return read still target the originally captured object.

## Uncertainties

The following remain documented_only:

- Application-level meanings of the gate, copied word, returned word, and second argument.
- Full record layouts, byte order, alignment requirements, and valid value ranges.
- Whether the offset-20 word is an address, handle, scalar, or another representation.
- Dependency return types and all dependency-side ownership, lifetime, global-state, and failure contracts.
- The identity and opaque function ID of the external dependency.
- Whether callers guarantee disjoint records or particular aliasing arrangements.
- Whether concurrent access is possible or supported.
- Whether invalid memory accesses trap, terminate execution, or produce other platform-specific behavior.

No arithmetic beyond fixed-offset addressing and a bit test is visible. There is no basis for introducing overflow handling, argument validation, retries, or additional error states.

## Evidence

Established behavior derives solely from the supplied static function representation: 32-bit word accesses, a 16-bit gate access, fixed field offsets, two argument-dependent branches, three possible dependency calls, and a final read of object offset 20.

The supplied dependency mapping establishes F-29b5e86301ff and F-05077d384bfe as the two mapped callees. It supplies no mapping or behavioral contract for the external dependency.

Activation-like and reset-like interpretations are inferred intent, not established semantics. No implementation beyond the supplied representation was inspected, and no runtime tests have been performed.
