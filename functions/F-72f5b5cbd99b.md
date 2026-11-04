<!-- Trusted provenance: F-72f5b5cbd99b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-72f5b5cbd99b

## Purpose

F-72f5b5cbd99b conditionally performs a setup sequence on a state object obtained from a dependency. A nonzero leading state byte prevents the sequence. Otherwise, the function requests a resource using a size-related field, stores the returned value, and, if that value is nonzero, invokes further dependencies before setting the leading byte to 1.

A zero return establishes that execution reached the end of the visible success path. It does not establish that every dependency successfully performed its intended operation: most dependency results are not inspected.

**Inferred intent:** The leading byte appears to be an initialization marker, and the resource request appears to concern storage sized in four-byte units. Neither interpretation establishes an allocation contract, element type, ownership policy, or concurrency guarantee.

## Interface

**Explicit inputs:** None.

**Implicit inputs:**

- The address returned by F-71400a9a0352.
- The byte at offset 0 from that address.
- A 32-bit size-related quantity at offset 12, read separately for two dependency calls on the success path.
- A global value, designated **G-01** in this contract, read after F-U02 returns and used as the second argument to F-U03. Its precise width, meaning, and indirection semantics are unresolved.

**Output:** A 32-bit result with these visible normal-return outcomes:

| Return bits | Signed 32-bit interpretation | Condition |
|---|---:|---|
| `0xFFFFFFFF` | −1 | The initial state byte is nonzero. |
| `0xFFFFFFFB` | −5 | The initial state byte is zero and F-fab3799b3190 returns zero. |
| `0x00000000` | 0 | The initial state byte is zero, F-fab3799b3190 returns nonzero, and the subsequent sequence completes normally. |

The signed values interpret the return bits. They do not establish the signedness of the public return ABI or a broader error-code convention.

### State layout

All offsets below are byte offsets from the address returned by F-71400a9a0352.

| Offset | Access established here | Use |
|---:|---|---|
| 0 | Read one byte; write one byte on the completed success path | Entry gate and final marker |
| 12 (`0x0C`) | Read a 32-bit quantity on two separate occasions on the success path | Size-related arguments |
| 56 (`0x38`) | Write a 32-bit value | Result of F-fab3799b3190 |
| 68 (`0x44`) | Pass its address | Sole invocation of F-b00260d1087a |
| 100 (`0x64`) | Pass its address | First invocation of F-046c66ae6de2 |
| 96 (`0x60`) | Pass its address | Second invocation of F-046c66ae6de2 |
| 108 (`0x6C`) | Pass its address | Third invocation of F-046c66ae6de2 |
| 104 (`0x68`) | Pass its address | Fourth invocation of F-046c66ae6de2 |

Passing an address does not establish the size, type, or initialization requirements of the object beginning there. The four nearby addresses passed to F-046c66ae6de2 are not proven to identify independent four-byte objects.

### Preconditions for normal completion

- F-71400a9a0352 must return an address readable at offset 0. There is no preceding null-result check.
- If the entry byte is zero, offset 12 must support the represented 32-bit reads and offset 56 the represented 32-bit write.
- The retained state address and all addresses passed to dependencies must remain valid for their subsequent uses.
- On the completed success path, offset 0 must remain writable for the final byte store.
- Each invoked dependency must accept its supplied arguments and return normally for execution to continue.
- G-01 must be readable when the success path reaches its global read.
- A target-specific contract must resolve arithmetic behavior outside the safe range described below.

These are requirements for the described completion paths, not validations performed by this function.

## Behavior

### Entry gate

The function invokes F-71400a9a0352 without explicit arguments and retains its returned address as the state base. It then reads the byte at offset 0.

If the byte is nonzero, the function returns `0xFFFFFFFF`. It does not directly read offset 12, write offset 56, read G-01, invoke any later setup dependency, or write the leading byte.

The test distinguishes zero from nonzero only. Values 1, 2, and 255 all reject setup; the function does not normalize them.

This branch is not necessarily free of side effects. F-71400a9a0352 has already executed, and its effects are unknown.

### Resource request and result storage

When the entry byte is zero, the function reads the 32-bit quantity at offset 12 and applies a left shift by two to form the argument to F-fab3799b3190.

After that dependency returns, its 32-bit result is written at offset 56. The store occurs before the result is tested for zero.

If the result is zero:

- Offset 56 has been overwritten with zero, regardless of its previous contents.
- The function returns `0xFFFFFFFB`.
- F-U01 and every later dependency are skipped.
- G-01 is not directly read.
- The function does not directly write the leading byte.

There is no restoration of the previous offset-56 value or rollback of dependency effects.

The branch decision uses the returned result, not a fresh read from offset 56. Any nonzero result selects the remaining setup sequence; the visible condition does not reject negative signed interpretations or require a valid pointer.

### Successful continuation

For a nonzero resource result, the remaining observable sequence is:

1. Read the 32-bit quantity at offset 12 again and apply the same left shift by two.
2. Invoke F-U01 with the resource result as its first argument and the freshly computed size-related value as its second.
3. Invoke F-b00260d1087a with the retained state address plus 68.
4. Invoke F-046c66ae6de2 with the retained state address plus 100.
5. Invoke F-046c66ae6de2 with the retained state address plus 96.
6. Invoke F-046c66ae6de2 with the retained state address plus 108.
7. Invoke F-046c66ae6de2 with the retained state address plus 104.
8. Invoke F-U02 without explicit arguments and retain its 32-bit result.
9. Read G-01 after F-U02 has returned.
10. Invoke F-U03 with F-U02’s result, the value obtained by that global read, and zero, in that argument order.
11. After F-U03 returns, write byte value 1 at offset 0.
12. Return `0x00000000`.

F-U01, F-U02, and F-U03 are opaque dependency IDs used here because no mapped IDs were supplied for those dependencies.

The two offset-12 reads are distinct. The argument to F-U01 may differ from the argument to F-fab3799b3190 if the field changes during the resource request. The function retains the original state base rather than obtaining a new one.

The embedded-address call order is 68, 100, 96, 108, 104. In particular, the four F-046c66ae6de2 calls use offsets 100, 96, 108, and 104, in that order.

### Global-read timing

G-01 is an actual global input read on the successful continuation, not merely an unspecified value supplied at entry.

The read occurs after all embedded-address calls and after F-U02 returns, but before F-U03 is invoked. Earlier dependencies, including F-U02, may change that global. Capturing G-01 before F-U02 would not preserve the established ordering.

There is no visible validation of the global value before it is passed to F-U03. The evidence does not establish whether that value is a scalar, pointer, handle, or reference to another object.

### Arithmetic boundaries

The size field is represented as a signed 32-bit integer, and both size-related arguments are formed by shifting it left by two. The representation does not establish a wider intermediate, checked multiplication, saturation, or explicit unsigned conversion.

For nonnegative field values from 0 through 536,870,911, multiplication by four remains representable as a positive or zero signed 32-bit result. For example, 8 produces 32, and the largest value in that range produces 2,147,483,644.

For negative field values or larger nonnegative values, the supplied representation does not resolve a portable language-level result. Exact wraparound or target instruction behavior is **documented_only**, not an acceptance requirement.

There is no visible size validation before either dependency call. Zero is not rejected by this function. The absence of validation does not establish that a dependency accepts every possible size.

## Side effects

The function’s direct state writes are:

- A 32-bit write at offset 56 after F-fab3799b3190 returns, for both zero and nonzero results.
- A one-byte write of 1 at offset 0 after F-U03 returns on the successful continuation.

The successful continuation also directly reads G-01 at the specified late point. No direct write to G-01 is shown.

Dependencies may modify the state object, G-01, other globals, resources, or external state. Their effects are not bounded by this function’s visible direct accesses.

The final byte store does not depend on an inspected success result from F-U01, F-b00260d1087a, F-046c66ae6de2, or F-U03. F-U02’s result is forwarded to F-U03 without validation. If these calls return normally, their visible return values do not select an additional error branch.

There is no explicit rollback if execution fails to return normally after the resource-result store. The evidence does not establish exception handling, nonlocal-transfer behavior, or recovery from an invalid dependency argument.

No explicit release operation appears. Storing the resource result does not prove ownership acquisition, exclusive ownership, or an obligation to release it. Unknown dependencies may consume, transfer, retain, or release resources; none is assumed to preserve ownership.

No synchronization, atomic gate-and-set operation, or re-entry protection is established. The leading byte is not rechecked after the entry read. A dependency’s intervening change to that byte does not change the selected path; if execution reaches the final store, the function overwrites it with 1.

## Dependencies

| Opaque ID | Visible interaction | Unresolved contract |
|---|---|---|
| F-71400a9a0352 | Called once without explicit arguments; returns the retained state address | Address validity, lifetime, ownership, global effects |
| F-fab3799b3190 | Receives the first shifted size; its 32-bit result is stored and tested | Resource semantics, valid nonzero results, state and global effects |
| F-U01 | Receives the nonzero resource result and a size computed from the second field read | Whether it initializes, clears, copies, validates, consumes, or releases anything |
| F-b00260d1087a | Receives state base plus 68 | Required object extent and initial state, modifications, failure behavior |
| F-046c66ae6de2 | Receives offsets 100, 96, 108, 104 in four ordered invocations | Object boundaries, initialization requirements, ownership and failure behavior |
| F-U02 | Called without explicit arguments; returns a 32-bit value used by F-U03 | Result meaning, ownership, effects on G-01 and other state |
| F-U03 | Receives F-U02’s result, the subsequent G-01 read, and zero | Argument meanings, resource effects, error reporting |

No dependency is assumed to preserve globals, preserve the state object, or leave memory ownership unchanged.

## Acceptance examples

These are conditional obligations derived from static evidence, not executed tests. Except where stated otherwise, they assume valid accesses, arithmetic-safe sizes, and normal dependency returns.

### Nonzero entry byte

If F-71400a9a0352 returns an object whose leading byte is 7, the result must be `0xFFFFFFFF`. No later setup dependency may be invoked, and the function must not directly read G-01 or write offset 56 or the leading byte.

Effects of F-71400a9a0352 remain permitted.

### Zero resource result

With an entry byte of zero and an initial size field of 8, F-fab3799b3190 receives 32. If it returns zero, offset 56 must be written with zero before the function returns `0xFFFFFFFB`.

F-U01 and all subsequent dependencies must be skipped. The function must not directly read G-01 or set the leading byte to 1.

### Stable successful setup

With an entry byte of zero, a size field remaining 8, and a nonzero resource result, both size-related arguments are 32.

The resource-result store precedes F-U01. The embedded-address calls then receive offsets 68, 100, 96, 108, and 104 from the original base. After F-U02 returns, the function reads G-01 and supplies that read’s value as F-U03’s second argument. Only after F-U03 returns does the function write the leading byte to 1 and return zero.

### Size changed by the resource request

If the initial size field is 8 and F-fab3799b3190 changes it to 12 before returning nonzero, that dependency receives 32, but F-U01 receives 48 as its second argument.

Reusing the initial size calculation for F-U01 would violate the separate-read behavior.

### Global changed by F-U02

Suppose G-01 contains one value before F-U02 is called, and F-U02 changes it to a different value before returning. If no intervening actor changes it again, F-U03 must receive the later value as its second argument.

Passing a value captured before F-U02 would violate the global-read ordering.

### Zero size

With a zero entry byte and a stable size field of zero, F-fab3799b3190 receives zero. A zero result selects `0xFFFFFFFB`; a nonzero result continues through F-U01 with a second argument of zero.

The function itself does not reject the size.

### Intervening marker change and repeated invocation

If a success-path dependency changes the leading byte, the function does not re-evaluate the gate. Reaching the final store overwrites that byte with 1.

If a completed invocation leaves that byte unchanged thereafter, a later invocation obtaining the same valid object returns `0xFFFFFFFF`. This is a conditional sequential observation, not a concurrent single-initializer guarantee.

## Uncertainties

**documented_only:**

- State-object type, full extent, alignment, and lifetime.
- Pointer representation and signedness of the external return ABI.
- Exact size-shift behavior outside the nonnegative signed-32-bit-safe range.
- Whether the resource result denotes memory, a handle, an identifier, or another token.
- Extents and required initial contents of the embedded objects.
- G-01’s precise width, type, value interpretation, indirection, and access properties.
- Dependency error channels, nonlocal transfers, and ignored-result meanings.
- Dependency changes to globals, object validity, or resource ownership.
- Contracts and mapped identities of F-U01, F-U02, and F-U03.
- Thread safety, re-entrancy, signal safety, and memory-ordering guarantees.

These unknowns do not weaken the established requirement to read G-01 after F-U02 returns or alter the four F-046c66ae6de2 invocation positions.

**Inferred intent:** Conditional initialization and resource sizing in four-byte units are plausible interpretations only. They do not establish successful initialization, allocation semantics, or a release policy.

## Evidence

Established static evidence is limited to the supplied function representation and dependency-ID mapping: the state lookup, byte gate, two separate size-field reads, stored resource result, two early-return bit patterns, ordered setup calls, late global read, final byte store, and zero return.

No dependency implementations, executable target, or runtime traces were provided. Arithmetic and ABI details not resolved by the supplied representation remain documented unknowns.

No runtime tests have been performed. The acceptance examples specify conditional behavior supported by static evidence; they are not reported execution results.
