<!-- Trusted provenance: F-e3c577ff9bca; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e3c577ff9bca

## Purpose

F-e3c577ff9bca repeatedly obtains candidate pointers from a dependency and compares the signed 16-bit value at each candidate with an original signed 32-bit search value. It stops when either:

- The candidate-producing dependency returns zero.
- A candidate’s value equals the search value.

Both normal stopping paths invoke a cleanup dependency before returning a packed 64-bit result.

The result is not simply a Boolean or a single pointer. Its low 32 bits distinguish the two stopping paths under ordinary execution: zero on candidate-production termination, or the matching candidate’s pointer bits on a match. Its high 32 bits contain the pointer-slot value observed **after cleanup**, on either path.

**Inferred intent:** this resembles an iterator-based search for a signed 16-bit value. Iterator semantics, exhaustion semantics, and resource ownership are not established by the supplied evidence.

## Interface

### Inputs

The function has four input positions:

| Position | Represented width and type | Established use |
|---|---|---|
| First | Opaque 32-bit value | No direct use in the visible body. It is not explicitly passed to a dependency. |
| Second | Pointer represented as 32 bits, targeting a 16-bit signed value | Initially copied into a local pointer slot. That slot is reset to null after initialization and before candidate production. |
| Third | Signed 32-bit integer | Preserved as the comparison target and separately copied into dependency-facing mutable storage. |
| Fourth | Opaque 32-bit value | Copied into storage adjacent to the dependency-facing integer. No direct interpretation is visible. |

The third input has two distinct roles:

1. Its original value remains the target for every comparison.
2. A mutable copy is supplied indirectly to dependencies.

Changes to the mutable copy do not, in the visible behavior, redefine the search target.

### Dependency-facing storage

The visible local arrangement consists of three consecutive 32-bit slots, in this order:

| Relative byte offset | Contents before initialization |
|---|---|
| −4 | Copy of the second input |
| 0 | Copy of the third input |
| +4 | Copy of the fourth input |

The address passed as the state argument points to the slot at relative offset 0. The candidate-producing dependency additionally receives the address of the pointer slot at relative offset −4.

This is an observed arrangement of local storage, not evidence of a public structure definition. The extent of memory that any dependency may validly access is undocumented. In particular, adjacency does not itself establish that every dependency reads or writes every slot.

After initialization returns, the pointer slot is explicitly overwritten with null. The integer and fourth-input slots are not explicitly reset.

### Output

The function returns a 64-bit bit pattern assembled from two 32-bit components:

| Stopping condition | High 32 bits | Low 32 bits |
|---|---|---|
| Candidate-producing dependency returns zero | Pointer slot after cleanup | Zero |
| Candidate value matches the original third input | Pointer slot after cleanup | Matching candidate pointer captured before cleanup |

The pointer components are packed as raw 32-bit values. The high component occupies bits 63–32; the low component occupies bits 31–0. This describes bit positions, not byte order in memory.

No separate error object, status field, or ownership annotation is returned.

## Behavior

### Initialization and ordering

Before the first dependency call, the function copies the second, third, and fourth inputs into the local storage described above.

It then invokes F-8ca21d3a8cca with the address of the mutable third-input copy.

Only after that call returns does the function reset the candidate pointer slot to null. Consequently:

- The second input is not directly dereferenced by this function.
- The second input does not survive as the initial candidate slot value for the first candidate-producing call.
- Initialization precedes the null reset.
- Initialization could observe or affect adjacent storage if its actual contract permits such access; that contract is not supplied.
- Any pointer-slot value established during initialization is overwritten before candidate production.

There is no visible pre-loop test. Assuming initialization returns normally, F-b38a3aba0e18 is called at least once.

### Candidate production

Each iteration invokes F-b38a3aba0e18 with:

- The same state address used for initialization.
- The address of the mutable candidate pointer slot.

The dependency’s return is interpreted as a 32-bit integer. Exactly zero selects termination. Every nonzero value, including a negative value, selects the candidate-inspection path.

After the dependency returns, the function reads the pointer slot and captures its value before examining the dependency’s zero/nonzero outcome.

The pointer slot is not explicitly reset between iterations. If a dependency invocation leaves it unchanged, the previous slot contents remain available for use.

### Zero-result termination

When F-b38a3aba0e18 returns zero:

1. This function does not read a 16-bit value through the candidate pointer.
2. It invokes F-d0d3f100effc with the state address.
3. After cleanup returns, it reads the pointer slot for the result’s high 32 bits.
4. It returns zero in the low 32 bits.

The candidate pointer captured before the zero/nonzero decision is not used as the low component on this path.

A zero dependency result therefore does not guarantee an all-zero 64-bit result. The high component may be nonzero, including because the slot retained an earlier pointer or because cleanup affected it.

The evidence establishes a zero-result termination condition, not whether that condition means exhaustion, error, cancellation, or another dependency-defined event.

### Nonzero-result inspection

When F-b38a3aba0e18 returns nonzero, the function reads one signed 16-bit value through the captured candidate pointer.

The value is sign-extended to the comparison width and compared with the original signed 32-bit third input. The target is not truncated to 16 bits.

Consequences include:

- A stored 16-bit pattern of all ones represents −1 for this comparison.
- That value matches a target of −1, not a target of 65535.
- A target outside −32768 through 32767 cannot equal a valid signed 16-bit candidate value.
- An out-of-range target is not rejected before iteration; candidate production and reads still proceed.

No null check, bounds check, alignment check, or other pointer validation is visible before this read.

If the values differ, the function proceeds to another candidate-producing call. It does not invoke cleanup between nonmatching candidates and does not itself advance the pointer or modify the pointed-to value.

If the values match:

1. The candidate pointer captured before the read is retained for the result’s low component.
2. F-d0d3f100effc is invoked with the state address.
3. After cleanup returns, the current pointer-slot value is used for the high component.
4. The packed result is returned.

Thus, the two halves of a matching result can differ. Equality requires the pointer slot still to contain the captured matching pointer after cleanup; that preservation is not established.

### Termination and invalid execution

The function has no visible iteration limit. If candidate production always returns nonzero and no candidate matches, it continues indefinitely, provided the dependencies return and each candidate read remains valid.

The visible normal-return paths each invoke cleanup exactly once. Cleanup is not shown as a guaranteed action after an invalid candidate read, a nonreturning dependency, or another abnormal transfer of control.

For normal nonzero-result processing, each candidate pointer must permit a valid 16-bit read under the execution environment’s memory and alignment rules. What happens when that requirement is violated is undocumented; there is no visible conversion of such a failure into a defined result.

## Side effects

### Directly established effects

The function directly:

- Initializes its local state and pointer slots.
- Replaces the pointer slot with null after initialization.
- Allows dependencies to operate through supplied local-storage addresses.
- Reads a signed 16-bit value for every nonzero candidate-producing result.
- Calls cleanup before each visible normal return.

No direct write to candidate memory is visible. No direct heap allocation, deallocation, global write, or I/O operation is visible.

### Dependency-mediated effects

The dependencies may have additional effects not described by this evidence. Their names and call placement do not establish that they preserve globals, avoid I/O, or leave candidate storage alive.

In particular, the cleanup call is not evidence that:

- It releases memory.
- It does not release memory.
- It preserves the pointer slot.
- It preserves the candidate’s lifetime.
- It returns the state to its initial contents.

The low component of a matching result retains the pre-cleanup pointer bits even if cleanup changes the slot or invalidates the referenced storage. Returning those bits does not establish that the caller may dereference them.

### Ownership and storage lifetime

The function’s dependency-facing state resides in local storage. The visible function does not allocate a persistent state object.

Dependencies receive addresses into that local storage. No supported lifetime beyond the invocation is established for those addresses.

Ownership of the second input, produced candidates, and any resources associated with the state is unknown. No transfer of ownership can be inferred merely from passing or returning pointer bits.

## Dependencies

| Opaque function ID | Visible call interface | Established role | Undocumented aspects |
|---|---|---|---|
| F-8ca21d3a8cca | Receives the state address | Called once before the pointer-slot reset and iteration | State interpretation, accessible storage extent, side effects, resource acquisition, failure behavior |
| F-b38a3aba0e18 | Receives the state address and pointer-slot address; returns a 32-bit integer | Repeatedly supplies or updates candidate state; zero terminates, nonzero causes inspection | Meaning of return values, whether the slot is always written, candidate validity and lifetime, progression guarantees |
| F-d0d3f100effc | Receives the state address | Called once on either visible normal exit | Resource release, pointer-slot mutation, global effects, candidate invalidation, failure behavior |

No assumption that one dependency preserves state written by another is part of this contract.

## Acceptance examples

These are conditional acceptance examples derived from static evidence. They are not reports of executed tests.

### Immediate zero result

Suppose initialization returns, the first candidate-producing call returns zero, and the pointer slot is null after cleanup.

Expected result: all 64 bits are zero. No candidate value is read.

If the same sequence instead leaves pointer bits P in the slot after cleanup, the expected result has P in its high 32 bits and zero in its low 32 bits. It is not an all-zero result.

### Immediate match with a preserved slot

Suppose the first candidate-producing call returns nonzero, supplies a readable pointer P to signed 16-bit value 23, and the original third input is 23. Suppose cleanup leaves the pointer slot equal to P.

Expected result: P appears in both 32-bit halves. Cleanup occurs after the matching read and before return.

### Match with cleanup changing the slot

Suppose a nonzero candidate-producing call supplies pointer P to the target value, but cleanup leaves pointer bits Q in the slot.

Expected result: high component Q, low component P.

If cleanup also invalidates the storage referenced by P, the returned low component still contains P’s bits; its subsequent dereferenceability is not guaranteed.

### Signed comparison

Suppose a valid candidate contains the 16-bit pattern 0xFFFF.

- With original target −1, it matches.
- With original target 65535, it does not match.

The second case continues candidate production rather than treating the low 16 bits of the target as sufficient.

### Nonmatch followed by termination

Suppose a first nonzero result supplies a valid nonmatching candidate. The next candidate-producing call returns zero.

Expected behavior: one candidate-value read, a second candidate-producing call, then cleanup. The returned low component is zero, regardless of the earlier candidate pointer. The high component reflects the slot after cleanup.

### Nonzero result without a valid pointer

Suppose candidate production returns nonzero but leaves the pointer slot null.

Expected visible action: the function attempts a signed 16-bit read through that null pointer. No graceful null-result path is established, and cleanup is not guaranteed after the invalid access.

### Out-of-range target

Suppose the original target is 32768 and all candidate reads are valid.

No signed 16-bit candidate can match. Normal termination requires a zero candidate-producing result; absent that event, the loop has no visible bound.

## Uncertainties

The following remain **documented_only**:

- The semantic purpose of the first input.
- Whether initialization accesses the neighboring pointer or fourth-input slots.
- The meaning and permitted mutations of dependency-facing state.
- Whether zero from candidate production denotes exhaustion or an error.
- Whether each nonzero result guarantees a fresh, valid candidate pointer.
- Candidate ordering, duplication, and progression.
- Candidate ownership and lifetime, particularly across cleanup.
- Dependency effects on globals, external resources, and retained addresses.
- Behavior when dependencies fail to return or perform abnormal control transfer.
- Pointer alignment requirements and invalid-read behavior.
- Byte order of any external serialization of the 64-bit result.

The search interpretation is inferred intent. The comparison width, call ordering, pointer-slot reset, and two-component return construction are established static behavior.

## Evidence

The supplied function body supports this contract through its visible dependency calls, local assignments, zero/nonzero branch, signed 16-bit dereference and comparison, and explicit 64-bit result construction.

The evidence supports a 32-bit pointer representation for this function’s result packing, a signed 32-bit target, and a signed 16-bit candidate value. It also distinguishes the candidate pointer captured before cleanup from the pointer slot read after cleanup.

Dependency implementations were not supplied. Their semantics, ownership policies, and preservation guarantees are therefore not established.

No runtime tests have been performed.
