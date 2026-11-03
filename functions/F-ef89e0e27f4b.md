<!-- Trusted provenance: F-ef89e0e27f4b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ef89e0e27f4b

## Purpose

F-ef89e0e27f4b performs a short initialization-and-worker-creation sequence against a caller-supplied object address. It clears two one-byte fields, passes an interior address and caller arguments to one dependency, then invokes a second dependency with a fixed worker name and configuration values.

The second dependency’s return value determines a status word and whether one of the cleared bytes is subsequently set to one. The function returns a 64-bit value containing that status in its low 32 bits and the fixed value 4096 in its high 32 bits.

The visible sequence is consistent with preparing an embedded object and attempting to start a worker. That interpretation is inferred intent, not an established contract for either dependency. In particular, the available evidence does not establish that a worker actually starts, what the worker does, or whether the returned high word denotes a stack size.

## Interface

### Inputs

The function accepts four arguments:

| Position | Width in the supplied representation | Visible use |
|---|---:|---|
| First | 32 bits | Base address for two byte stores and for deriving an interior address. Also forwarded unchanged to F-95f1655a5c87. |
| Second | 32 bits | Forwarded unchanged as the fifth argument to F-95f1655a5c87. |
| Third | 32 bits | Forwarded unchanged as the sixth argument to F-95f1655a5c87. |
| Fourth | 32 bits | Forwarded unchanged as both the fourth and seventh arguments to F-95f1655a5c87. |

The first argument is represented as a signed 32-bit integer but is used as an address. Its representation does not establish a signed numeric meaning for the addressed object.

No type, ownership, pointer, or numeric interpretation is established for the other three arguments. Their visible contract is preservation of their 32-bit values at the specified argument positions.

### Required addressable storage

Let the supplied object’s base address be B. The function directly writes one byte at each of these locations:

- B + 1.
- B + 3.

It also passes B + 80 to both dependencies. This is an address derived from the object base, not a value loaded from offset 80.

The caller must supply storage for which the direct stores and any dependency-required accesses are valid. The available evidence does not establish the complete object size, the alignment of the embedded region, or how many bytes either dependency accesses there.

Address calculations must produce the intended accessible locations. No null check, range check, overflow check, or storage validation appears in the visible function. Behavior for invalid addresses or address arithmetic outside the intended address space is not specified by this contract.

### Output

On a normal return from both dependency calls, the function returns a 64-bit value with this layout:

| Bit range | Meaning |
|---|---|
| 63–32 | Always 0x00001000, or 4096. |
| 31–0 | 0xFFFFFFFB if F-195f5d43c91b returned zero; otherwise 0x00000000. |

Consequently, the two normal return bit patterns are:

- Zero dependency result: 0x00001000FFFFFFFB.
- Nonzero dependency result: 0x0000100000000000.

The low word 0xFFFFFFFB represents −5 when interpreted as a signed 32-bit two’s-complement integer. The complete 64-bit return is not a sign-extended −5. Consumers must preserve the distinction between the packed return value and its low-word status.

No pointer or dependency-produced handle is returned directly.

## Behavior

### 1. Clear two bytes in a defined order

The function first stores zero to the byte at B + 1.

It next stores zero to the byte at B + 3.

These are separate one-byte stores. The function does not first read either byte and does not perform a visible wider store covering neighboring bytes. Bytes at other offsets are not directly initialized by this function.

Both stores precede the first dependency call. This ordering is established for the visible execution sequence; it is not evidence of atomic operations, memory barriers, or inter-thread synchronization.

### 2. Invoke the preparation dependency

The function calls F-95f1655a5c87 with seven arguments in the following positions:

| Position | Supplied value |
|---|---|
| 1 | B + 80. |
| 2 | A value obtained from a fixed global data location used for this call. |
| 3 | B, unchanged from the first input. |
| 4 | The fourth input. |
| 5 | The second input. |
| 6 | The third input. |
| 7 | The fourth input again. |

The repetition of the fourth input is part of the observable interface. It must not be replaced by a different input merely because the two positions appear redundant.

The global value’s contents and meaning are unavailable. The visible sequence obtains this call’s global argument after the two byte clears and before entering the dependency. There is no basis for treating that value as an invariant configuration constant across invocations.

No result from this call is tested or used in the visible continuation. If it returns normally, execution proceeds to the second dependency regardless of any result it may produce.

This does not mean the dependency cannot fail, terminate execution, modify the object, modify globals, or affect ownership. Those behaviors are outside the supplied evidence.

### 3. Invoke the worker-related dependency

After the first dependency returns normally, the function calls F-195f5d43c91b with seven arguments:

| Position | Supplied value |
|---|---|
| 1 | The fixed worker-name string AVSManager_WorkerThread. |
| 2 | A value obtained from a second fixed global data location. |
| 3 | B + 80. |
| 4 | 16, represented by 0x10. |
| 5 | 4096, represented by 0x1000. |
| 6 | Zero. |
| 7 | Zero. |

The global value for this second call is obtained in the continuation after F-95f1655a5c87 returns. The contract therefore must not assume it has the same value it had before that dependency ran. The first dependency may have changed global state.

The two numeric configuration arguments have established values but unestablished meanings. Neither “priority” for 16 nor “stack size” for 4096 is proven.

The second dependency’s return is examined only for equality with zero. Its magnitude and sign otherwise have no visible effect.

### 4. Select the low-word status and optionally write a byte

If F-195f5d43c91b returns zero:

- The low return word is selected as 0xFFFFFFFB.
- The function stores one to the byte at B + 3.
- The function returns that low word paired with the high word 0x00001000.

If F-195f5d43c91b returns any nonzero value:

- The low return word is zero.
- No further direct store is performed.
- The function returns that low word paired with the high word 0x00001000.

The zero case is visibly treated as the failure-like case. The exact error domain or external meaning of −5 is unknown.

In the nonzero case, the function does not restore B + 3 to zero after either dependency. Its initial clear cannot be used to infer that the byte is still zero at return.

## Side effects

The function’s own explicit memory writes are limited to:

1. Zero at B + 1.
2. Zero at B + 3.
3. One at B + 3, only after a zero result from F-195f5d43c91b.

The function also obtains two values from global data locations for its dependency arguments. It performs no visible explicit global write.

Both dependencies can have additional effects not described by the available evidence. These may include changes to the supplied object, other memory, global state, resource state, or scheduling. No preservation guarantee follows from their opaque status.

The visible function performs no explicit allocation, deallocation, reference-count adjustment, object destruction, or resource cleanup. It does not roll back the preparation call when the second dependency returns zero.

Ownership remains undocumented. Passing B and B + 80 to dependencies does not establish whether those dependencies retain either address, access them asynchronously, or release associated resources. Required object lifetime beyond the calls is therefore unknown.

Repeated calls repeat the same visible stores and dependency invocations. There is no visible guard against reinitialization or duplicate worker creation, but the dependencies may impose such behavior.

## Dependencies

### F-95f1655a5c87

Called once per invocation that reaches it, after both initial byte stores.

Established contract at this boundary:

- Receives the seven arguments listed above.
- Receives both the object base and its offset-80 interior address.
- Has no return value consumed by this function.
- Must return normally for the second call to be reached.

Its internal behavior, failure signaling, memory requirements, ownership rules, and global-state effects are undocumented.

### F-195f5d43c91b

Called once after a normal return from F-95f1655a5c87.

Established contract at this boundary:

- Receives the fixed worker name, a global-sourced value, the offset-80 address, and four fixed numeric arguments.
- Supplies a 32-bit result used solely for a zero-versus-nonzero decision.
- A zero result selects low-word status −5 and causes the final byte store.
- Every nonzero result selects low-word status zero.

Whether a nonzero result is a handle, identifier, boolean, pointer-like value, or another status representation is undocumented.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not reports of executed tests.

### Nonzero dependency result without dependency memory changes

Assume valid storage, normal dependency returns, no dependency changes to the two byte fields, and a second dependency result of one.

Required observations:

- B + 1 is cleared before B + 3.
- Both clears precede F-95f1655a5c87.
- The second dependency follows the first.
- Both bytes remain zero under the stated no-mutation assumption.
- The return value is 0x0000100000000000.

The same return is required for every other nonzero second-dependency result, including negative signed 32-bit values.

### Zero dependency result

Assume both calls return normally and the second returns zero.

Required observations:

- The final direct store writes one at B + 3 after the second call returns.
- The returned low word is 0xFFFFFFFB.
- The returned high word remains 0x00001000.
- No visible cleanup call is added between the zero result and the return.

### Dependency changes to the initially cleared bytes

Assume a dependency changes B + 1 and B + 3, with no later independent mutation.

If the second dependency returns nonzero, those changes are not explicitly overwritten by this function.

If it returns zero, B + 3 is overwritten with one, but B + 1 is not restored. An unconditional guarantee that both fields retain their initial cleared values would therefore be incorrect.

### Global change during preparation

Assume F-95f1655a5c87 changes the global value used as the second argument of F-195f5d43c91b and then returns normally.

The second call must use the value obtained after that return, not a value cached before preparation.

### Packed return interpretation

For a zero second-dependency result, a consumer splitting the result into 32-bit halves obtains high word 4096 and low word 0xFFFFFFFB. A consumer expecting the entire return to equal signed 64-bit −5 would reject the actual contract.

## Uncertainties

The following are documented_only:

- Full object layout and the meanings of offsets 1, 3, and 80.
- Meanings and valid ranges of the second, third, and fourth inputs.
- Contents and roles of both global-sourced arguments.
- Semantics of the numeric dependency arguments 16, 4096, zero, and zero.
- Resource ownership, retention, asynchronous access, and lifetime requirements.
- Whether the dependencies create a worker, and whether that worker can execute before this function returns.
- Dependency behavior on invalid input or internal failure.
- Synchronization guarantees and safety of concurrent or repeated invocation.
- Target-specific behavior for invalid addresses or overflowing address calculations.
- Any external symbolic error name associated with the low-word value −5.

The name and call shape support a worker-initialization interpretation, but do not resolve these unknowns.

## Evidence

Established static evidence consists of the supplied function body and the supplied opaque dependency mapping. It shows two initial byte stores, two ordered calls, a zero-result branch, one conditional byte store, and construction of a two-word return value.

The contract distinguishes those visible operations from inferred intent and undocumented dependency behavior. No implementation access beyond the supplied material was used. No runtime tests have been performed.
