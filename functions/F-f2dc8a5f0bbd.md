<!-- Trusted provenance: F-f2dc8a5f0bbd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f2dc8a5f0bbd

## Purpose

F-f2dc8a5f0bbd copies two fields from a source handle into a destination handle, increments the reference count associated with the copied backing record, and then attempts an operation on an object reached through the source handle.

The later operation has two routes:

- A conditional dispatch through F-621dae46a146 when the object's leading word matches a global discriminator.
- An ownership-sensitive path that can decrement a 16-bit counter, clear ownership state, and invoke F-259d2fe281b0 on an embedded region.

The ordering is important: the destination update and reference-count increment occur before any return from either route. A nonzero result does not imply that the destination remained unchanged, and no rollback is visible.

**Inferred intent:** the fields resemble a reference-counted handle, and the ownership-sensitive path resembles releasing a recursively held synchronization object. These interpretations are not established dependency contracts. The behavior below is expressed in terms of fields, comparisons, and calls rather than relying on those interpretations.

## Interface

### Inputs

The visible interface has four explicit inputs:

| Input | Visible role |
|---|---|
| Destination address | Address of a writable handle-like record. Passed to F-1c84e4d56090 before this function writes two of its fields. |
| Source address | Address of a readable handle-like record. Passed to F-ccc494c80b38 before its fields are used. |
| Auxiliary value A | Opaque 32-bit value forwarded to F-621dae46a146 if that dependency is called. |
| Auxiliary value B | Opaque 32-bit value forwarded to F-621dae46a146 if that dependency is called. |

The supplied representation uses 32-bit address-bearing values and 32-bit words. Two object fields are accessed as 16-bit quantities. Exact calling-convention details are not supplied.

The conditional call to F-621dae46a146 also includes four values represented as pre-existing machine state, beyond the explicit inputs. Their origins and meanings are unknown. They are not established as additional public parameters.

### Record layouts

Only the following accessed portions of the records are established. The layouts do not imply complete record sizes.

**Source and destination handle-like records**

| Byte offset | Width | Meaning established by use |
|---|---:|---|
| +4 | 32 bits | Address of a backing record |
| +8 | 32 bits | Opaque copied value |

The source and destination need enough accessible storage for these accesses. Any additional requirements imposed by the preliminary dependencies are unknown.

**Backing record**

| Byte offset | Width | Meaning established by use |
|---|---:|---|
| +0 | 32 bits | Counter incremented after copying the handle fields |
| +8 | 32 bits | Address of the object used by the subsequent operation |

**Object**

| Byte offset | Width | Meaning established by use |
|---|---:|---|
| +0 | 32 bits | Word compared with a global discriminator |
| +4 | 32 bits | Ownership-associated word compared with the result of F-196812b4b122 |
| +14 | 16 bits | Flags sampled before calling F-196812b4b122 |
| +18 | 16 bits | Counter conditionally decremented and explicitly cleared |
| +20 | Address only | Start of a region passed to F-259d2fe281b0 |

The object need not be interpreted as containing a separately allocated object at +20. The visible operation passes the address of that location, not a pointer loaded from it. Its required extent is unknown.

### Output

The function returns a 32-bit integer result. Its visible sources are:

- `0` on specified no-operation or retained-counter paths.
- `5` when the ownership-associated word does not match the result of F-196812b4b122.
- `26` when the backing record's object address is null.
- A nonzero result from F-621dae46a146.
- Any result from F-259d2fe281b0.

These values are not assigned named error meanings by the supplied evidence. In particular, a dependency result can numerically coincide with a locally produced result.

## Behavior

### 1. Preliminary dependency calls

The function first invokes F-ccc494c80b38 twice, in this order:

1. A call with no explicit argument shown.
2. A call with the source address.

It then invokes F-1c84e4d56090 with the destination address.

No return value from these three calls controls the visible continuation. Their side effects, validation behavior, allocation behavior, and termination behavior are unspecified. All descriptions of subsequent accesses are conditional on these calls returning normally.

The first call's argument interface is unresolved. It must not be assumed to be identical to the second call's interface merely because both calls identify the same dependency.

### 2. Destination update and counter increment

After the preliminary calls return, the function performs the following observable accesses in order:

1. Reads the backing-record address from source offset +4.
2. Stores that address at destination offset +4.
3. Reads the 32-bit value at source offset +8.
4. Stores that value at destination offset +8.
5. Reads the 32-bit counter at offset +0 of the backing record obtained in step 1.
6. Writes the incremented counter back to that same location.

The source's two copied fields are not captured together before writing the destination. Consequently, partial overlap between the source and destination can affect the value subsequently read at source offset +8. Likewise, aliasing between a handle and a backing record can affect later reads. The function does not visibly detect or reject such overlap.

For ordinary, nonoverlapping records, this phase makes the destination's two fields equal to the corresponding source fields and adds one to the backing record's counter.

No explicit overflow check accompanies the 32-bit increment. A precise portable result for signed overflow is not established by the supplied representation.

### 3. Reloading the source backing record

Only after the counter increment does the function reread source offset +4.

- If this reread produces zero, the function returns `0`.
- Otherwise, it reads the object address at offset +8 of the backing record identified by this reread.

The null check is not protection against an initially null copied backing-record address: that address has already been dereferenced for the increment. An initial null value therefore does not establish a safe zero-return case.

The reread also means that the backing record incremented earlier is not guaranteed to be the backing record used to locate the object. They are the same under stable, nonaliasing conditions, but that equivalence is not unconditional.

If the object address read here is zero, the function returns `26`. The preceding destination writes and counter increment remain in effect.

### 4. Conditional dispatch

For a nonnull object address, the function reads the object's leading 32-bit word and compares it with a global discriminator.

If they are equal, it calls F-621dae46a146 with:

- The object address.
- A zero-valued second argument.
- Auxiliary value A.
- Auxiliary value B.
- Four additional values represented as incoming machine state.

If the dependency returns a nonzero result, this function immediately returns that result.

If the words do not match, or if the dependency returns zero, execution continues to the ownership-sensitive path.

A zero result from F-621dae46a146 therefore means “continue” in this caller; it is not by itself the final result. The dependency's effects are not undone before continuation.

The object address selected earlier remains the address used by the continuation. The function does not visibly reload it from the backing record after this call. Since the dependency's lifetime and mutation guarantees are unknown, safe continuation requires that subsequent accesses through this address remain valid.

### 5. Ownership-sensitive path

The function reads and retains the 16-bit flags at object offset +14. It then calls F-196812b4b122.

After that call returns, the function reads the object's 32-bit ownership-associated word at offset +4 and compares it with the dependency's result.

This ordering has two consequences:

- The flags used for the decision are sampled before F-196812b4b122.
- The ownership-associated word is sampled after that call.

If the values do not match, the function returns `5`. On this branch, it makes no visible write to the object's ownership-associated word or its 16-bit counter. Earlier handle updates and dependency effects are unaffected.

If they match, the function examines the captured flags. The special counter-controlled mode is selected exactly when bits 4 and 5 form binary `10`. Equivalently, masking the flags with `0x0030` must produce `0x0020`. Other flag bits do not affect this decision.

#### Counter-controlled mode

In this mode, the function reads the 16-bit counter at object offset +18, subtracts one, and writes a 16-bit result to the same field. The resulting value is treated as a signed 16-bit quantity for comparison with zero.

- If the resulting value is nonzero, the function returns `0`. It does not visibly clear ownership or call F-259d2fe281b0.
- If the resulting value is zero, execution proceeds to the clearing path.

There is no visible check that the old counter is positive. In particular, an old value of zero is not treated as an already-cleared state: it is decremented rather than directly released.

The field access and store widths are established. Exact out-of-range conversion semantics are not fully established by the supplied representation; boundary cases requiring them are documented-only.

#### Other flag modes

When bits 4 and 5 do not select the counter-controlled mode, the function proceeds directly to the clearing path without first decrementing the counter.

### 6. Clearing path

The function performs these actions in order:

1. Writes a 16-bit zero at object offset +18.
2. Writes a 32-bit zero at object offset +4.
3. Calls F-259d2fe281b0 with the address of object offset +20.
4. Returns that dependency's result.

The clearing writes occur before the dependency call. There is no visible restoration of either field if the dependency reports an error.

In counter-controlled mode, reaching this path means the decrement already wrote zero to the counter; the explicit clearing write still follows.

## Side effects

The function's own visible writes are limited to:

- Destination offset +4.
- Destination offset +8.
- The copied backing record's 32-bit counter.
- Conditionally, the object's 16-bit counter.
- Conditionally, the object's ownership-associated word.

The function also reads the global discriminator. No direct write to that global is visible. This does not establish that called dependencies preserve it or other global state.

The function contains no explicit allocation or deallocation operation. That observation does not constrain the dependencies: notably, the effects of F-1c84e4d56090 on the destination's previous contents are unknown.

The increment suggests acquisition of an additional reference, but the full ownership protocol is not established. There is no visible balancing decrement on later local error paths.

No locking, atomic operation, or memory-order guarantee is established for the visible counter and ownership accesses. Concurrent mutation cannot be assumed safe. Validity and lifetime must cover every access, including accesses after dependency calls.

## Dependencies

| Opaque function ID | Caller-visible role | Unknown contract |
|---|---|---|
| F-ccc494c80b38 | Called twice before field access; the second call receives the source address | Exact interface, meaning of the first call, mutations, validation, ownership effects |
| F-1c84e4d56090 | Receives the destination before replacement fields are written | Whether it clears, releases, validates, or otherwise changes the destination |
| F-621dae46a146 | Conditional object dispatch; nonzero result terminates this function, zero continues | Result meanings, full calling convention, object lifetime, mutations, global effects |
| F-196812b4b122 | Supplies the value compared with the object's ownership-associated word | Whether it identifies a thread or another entity; mutations and global effects |
| F-259d2fe281b0 | Receives the embedded-region address after ownership state is cleared | Region layout, release semantics, result meanings, lifetime and global effects |

None of these dependencies is assumed to preserve memory, globals, or ownership merely because this caller does not inspect their effects.

## Acceptance examples

These examples are conditional contracts, not reported executions. Unless otherwise stated, preliminary calls return normally, addresses are valid, records do not overlap, relevant memory is stable except for described writes, and arithmetic stays within representable ranges.

1. **Null object after successful handle update.**  
   The source names a valid backing record whose counter is `7` and whose object address is zero. The destination receives the source's two fields, the counter becomes `8`, and the function returns `26`. It does not call F-196812b4b122 or F-259d2fe281b0.

2. **Conditional dispatch terminates early.**  
   The object discriminator matches the global discriminator, and F-621dae46a146 returns `9`. The function returns `9` after the handle update and increment. It does not enter the ownership-sensitive path.

3. **Zero dispatch result continues.**  
   The discriminator matches and F-621dae46a146 returns zero while leaving the object valid. If the later ownership comparison fails, the final result is `5`, not zero.

4. **Ownership mismatch.**  
   Conditional dispatch is bypassed. F-196812b4b122 returns a value different from the ownership-associated word read after that call. The function returns `5` without directly changing the object's counter or ownership-associated word.

5. **Counter-controlled mode retains ownership.**  
   Ownership matches, the relevant flag bits equal `0x0020`, and the counter starts at `3`. The function stores `2` and returns `0`, without calling F-259d2fe281b0 or clearing the ownership-associated word.

6. **Counter-controlled mode reaches zero.**  
   Under the same conditions, the counter starts at `1`. The function writes the decremented zero, explicitly writes zero again, clears the ownership-associated word, and calls F-259d2fe281b0 at object offset +20. Its return value becomes the final result.

7. **Other mode clears regardless of the old counter.**  
   Ownership matches, the relevant flag bits are not `0x0020`, and the counter starts at `7`. The function writes zero rather than decrementing to `6`, clears ownership, and returns F-259d2fe281b0's result.

8. **Zero counter is not a release trigger in counter-controlled mode.**  
   Ownership matches, the relevant flag bits select the counter-controlled mode, and the counter starts at zero. The decrement produces `−1`, which is nonzero, so the function returns `0` without clearing ownership or calling F-259d2fe281b0.

9. **Initially null backing address is not a supported null case.**  
   If source offset +4 supplies zero for the initial copy, the counter access occurs before the later null check. No normal result is established for this input.

## Uncertainties

The following are **documented_only**, not guessed contracts:

- Exact ABI, argument recovery, and meanings of the additional machine-state values passed to F-621dae46a146.
- Complete record sizes, alignment requirements, byte order, and the embedded region's layout.
- Dependency validation, mutation, allocation, release, and failure behavior.
- Meanings of the discriminator, flags, ownership word, and result codes.
- Validity guarantees across dependency calls, especially after a zero result from F-621dae46a146.
- Signed 32-bit increment overflow and out-of-range signed 16-bit conversion behavior.
- Supported aliasing relationships and any concurrency protocol.

The labels “handle,” “reference count,” and “ownership” describe inferred roles. They do not establish a complete lifecycle or synchronization API.

## Evidence

The supplied static representation establishes the access widths and offsets, dependency-call order, branch conditions, explicit stores, and return-value sources described above.

It also establishes several important limits: the backing-record dereference precedes the null check; source offset +4 is reread; flags are sampled before F-196812b4b122 while ownership is read afterward; and clearing precedes F-259d2fe281b0.

No dependency bodies, runtime traces, or authoritative data-type definitions were supplied. No runtime tests have been performed. All acceptance examples are conditional predictions from the visible static behavior.
