<!-- Trusted provenance: F-afac6f2e1734; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-afac6f2e1734

## Purpose

F-afac6f2e1734 conditionally invokes two dependencies and copies two 32-bit words from a caller-supplied object into caller-supplied output locations.

A signed 32-bit field at byte offset `0xB8` controls whether the operation proceeds. If that field is negative, the function returns 0 without invoking either dependency or writing either output. If it is nonnegative, the function invokes the dependencies in order, copies the two words, and returns 1, provided execution reaches normal completion.

This is the established branch behavior. The meanings of the object, its fields, the dependency operations, and the copied words are not established. In particular, return value 1 indicates completion of the visible eligible path, not independently verified success of either dependency.

## Interface

The function has four inputs:

| Input | Role | Established requirements |
|---|---|---|
| Object address | Locates the controlling field and two source words | Must support the accesses actually reached during execution |
| Forwarded value | Passed as the first argument to F-f5c1d5368335 | Its semantic type and interpretation are unknown |
| First output address | Receives the word read at object offset `0xBC` | Must permit a 32-bit write on the eligible path |
| Second output address | Receives the word read at object offset `0xC0` | Must permit a 32-bit write on the eligible path |

The visible return value is a 32-bit value with two normal outcomes:

- **0:** the initially read controlling field was negative.
- **1:** the initially read controlling field was nonnegative, both dependency calls returned, and both output transfers completed.

No other normal return value is visible.

### Object layout

Only these object-relative locations are accessed directly:

| Byte offset | Width | Treatment |
|---|---:|---|
| `0xB8` | 32 bits | Interpreted as a signed integer for the eligibility decision |
| `0xBC` | 32 bits | Copied without an established signedness or semantic interpretation |
| `0xC0` | 32 bits | Copied without an established signedness or semantic interpretation |

The highest directly accessed byte is at offset `0xC3`. This does not establish the full object size or make earlier bytes part of the function’s direct read set.

The supplied representation uses a 32-bit integer-shaped carrier for the object address. That is not sufficient to establish a portable pointer ABI. The contract requires the identified address-relative accesses to resolve correctly in the execution environment. Address overflow, invalid mappings, alignment restrictions, and byte order are not defined by the available evidence.

## Behavior

### Eligibility decision

The first visible operation reads the signed 32-bit field at object offset `0xB8`.

The function proceeds exactly when that value is greater than or equal to zero. This includes zero and every positive signed 32-bit value. All negative signed 32-bit values take the ineligible path.

On the ineligible path, the function returns 0 immediately. It does not directly read the words at offsets `0xBC` and `0xC0`, read the dependency argument from global storage, call either dependency, or write either output.

The controlling field is read only once by the visible function. Changes made later by a dependency do not cause the function to repeat the eligibility decision.

### Eligible-path dependency calls

After a nonnegative controlling value is observed:

1. A value from a particular global storage location is supplied as the sole argument to F-6495483bd2c4.
2. The 32-bit result returned by F-6495483bd2c4 is retained for the next call.
3. F-f5c1d5368335 is invoked with the function’s forwarded input as its first argument and that result as its second argument.

The global argument is obtained before the first dependency is invoked. Its identity, meaning, and independently established storage width are unavailable.

The second dependency is called only after the first returns. Its result, if any, is not inspected by this function. There is no visible dependency-status check, retry, or alternate branch after either call.

Neither dependency is known to preserve the object, output locations, or global state. Consequently, the copied words are not snapshots taken before the calls.

### Ordered output transfers

After the second dependency returns:

1. The function reads the 32-bit word at object offset `0xBC`.
2. It writes that word to the first output location.
3. It reads the 32-bit word at object offset `0xC0`.
4. It writes that word to the second output location.
5. It returns 1.

This ordering is part of the visible behavior. In particular, the second source read follows the first output write. The function does not visibly prefetch both source words before writing either output.

### Aliasing

No distinctness check is visible for the object or output addresses.

If both output addresses designate the same writable 32-bit location, the second write replaces the first. The final value there is the word obtained by the second source read.

If the first output address designates the object word at offset `0xC0`, the first output write changes the value that the later second source read obtains. Thus both transfers can yield the original word from offset `0xBC`, assuming the accesses are otherwise valid and no other modification intervenes.

These are consequences of the visible access order, not guarantees that arbitrary overlapping or misaligned accesses are supported. Platform rules for such accesses remain unspecified.

## Side effects

The function directly performs:

- One controlling-field read on every normally entered path that can access the object.
- One global argument read on the eligible path.
- Two dependency calls on the eligible path, in the specified order.
- Two object-word reads and two output writes after those calls return.

The direct implementation does not allocate or release memory, clear an output on rejection, or explicitly update a global storage location. Dependencies may perform additional reads, writes, allocation, release, or other effects; none are established here.

The supplied addresses are used for memory access. There is no visible ownership-transfer operation. However, dependency behavior is unknown, so this function alone cannot establish that related storage remains allocated or unchanged across the calls. Successful completion requires the later source and destination accesses still to be valid when reached.

The operation is not visibly transactional. No rollback is shown if execution stops after a dependency effect or after the first output write. No validation or local recovery is shown for invalid addresses or dependency failures.

## Dependencies

### F-6495483bd2c4

- Invoked once on the eligible path.
- Receives a value obtained from global storage.
- Produces a 32-bit value passed to F-f5c1d5368335.
- Completes before F-f5c1d5368335 is invoked.
- Has no established contract here for global preservation, object preservation, ownership, errors, or nonlocal termination.

### F-f5c1d5368335

- Invoked once after F-6495483bd2c4 returns on the eligible path.
- Receives the forwarded input first and the preceding dependency’s result second.
- Completes before either source word is copied.
- Has no result visibly used by this function.
- Has no established contract here for memory effects, ownership, errors, or nonlocal termination.

Neither dependency is invoked on the negative-field path.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Successful-path cases assume valid accesses, dependency return, and no additional modifications beyond those stated.

### Negative controlling value

Given a controlling value of −1 and output locations containing arbitrary existing words:

- The function returns 0.
- Neither dependency is called.
- Both output locations retain their existing contents.
- Neither source word is directly read by this function.

The same branch applies to the minimum signed 32-bit value, −2,147,483,648.

### Zero controlling value

Given a controlling value of zero, source words `0x11223344` and `0xAABBCCDD`, distinct output locations, and dependencies that leave those sources unchanged:

- Both dependencies are called in order.
- The first output receives `0x11223344`.
- The second output receives `0xAABBCCDD`.
- The function returns 1.

Zero is eligible; the condition is not a strict-positive test.

### Positive boundary

A controlling value of 2,147,483,647 takes the eligible path under the same access and dependency assumptions. No narrower upper bound is visible.

### Dependency changes a source word

Suppose a dependency changes the word at offset `0xBC` before the second dependency returns. The first output receives the subsequently read value, not necessarily the value present at function entry.

If a dependency makes the controlling field negative after the initial eligible read, the function nevertheless continues to the transfers when the dependency calls return.

### Identical output addresses

Suppose both outputs designate the same location outside the source words. The dependencies leave source words A and B unchanged.

The location is first written with A and then with B. Its final value is B, and the function returns 1.

### First output aliases the second source

Suppose the first output designates offset `0xC0`, the second output is elsewhere, and the source words initially contain distinct values A and B.

The first transfer writes A over B. The second source read then obtains A, so the second output also receives A.

## Uncertainties

**documented_only — Domain meaning.** The object type, field meanings, forwarded value, and purpose of the global argument are unknown. The controlling field is established as an eligibility discriminator, not as a particular status, handle, count, or resource identifier.

**documented_only — Dependency contracts.** Preservation, ownership, failure reporting, reentrancy, and termination behavior of both dependencies are unknown. Normal continuation must not be assumed when a dependency does not return.

**documented_only — Memory model.** The evidence does not establish pointer width, alignment requirements, atomicity, synchronization, volatile semantics, or valid behavior under concurrent modification.

**documented_only — Error behavior.** The visible 0 result covers the negative-field branch. No defined error return is established for invalid memory, failed dependencies, or interrupted execution.

**Inferred intent.** The structure is consistent with an eligibility-gated dependency action followed by extraction of two words. This interpretation does not establish any further semantic guarantee.

## Evidence

The contract is based solely on the supplied static function representation. It establishes the signed branch condition, field offsets, 32-bit transfers, dependency order, output-access order, and normal return constants.

Dependency implementations, an authoritative ABI, callers, object definitions, and global-storage definitions were not supplied. No runtime tests have been performed. Acceptance examples describe conditional consequences of the visible behavior rather than observed executions.
