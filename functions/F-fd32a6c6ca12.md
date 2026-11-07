<!-- Trusted provenance: F-fd32a6c6ca12; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fd32a6c6ca12

## Purpose

F-fd32a6c6ca12 has two observable paths:

- When an input object's byte at offset `0xB50` is nonzero, set its byte at offset `0xB51` to `1` and return.
- Otherwise, obtain a candidate address from a dependency. If that address is nonzero, validate its preceding header, update accounting fields in a separately obtained state object, and invoke three state-related dependencies around those updates.

The accounting operations resemble removal of an allocated item from allocator bookkeeping. That interpretation is **inferred intent**, not an established ownership or deallocation contract. The dependency implementations are unavailable; the visible behavior does not establish that storage is actually released.

This contract describes normal control flow and observable access ordering supported by the supplied static representation. It does not establish behavior for invalid addresses, corrupted metadata, unexpected dependency effects, or execution outside the represented 32-bit memory model.

## Interface

**Input:** One value used as an object address, represented as a 32-bit integer in the supplied interface.

**Output:** No explicit return value. Normal completion is observable through memory changes and dependency calls.

**Directly accessed layouts:**

| Object | Offset | Width | Role |
|---|---:|---:|---|
| Input object | `0xB50` | 8 bits | Initial branch condition |
| Input object | `0xB51` | 8 bits | Flag written on the early-return path |
| Candidate | `-8` | 32 bits | Header used for validity checks and masked accounting amounts |
| Candidate | `-4` | 16 bits, unsigned | First accounting selector |
| Candidate | `-2` | 16 bits, unsigned | Second accounting selector |
| State object | `0x224 + 4 × first selector` | 32 bits | First selected accounting field |
| State object | `0x30C` | 32 bits | Fixed accounting field |
| State object | `0xE0 + 4 × second selector` | 32 bits | Second selected accounting field |
| State object | `0x220` | 32 bits | Fixed accounting field |
| State object | `0x310 + 4 × dependency result` | 32 bits | Selected count field |
| State object | `0x390` | 32 bits | Fixed count field |
| State object | `0x08` | 32 bits | Final accounting field |

“Candidate” means the nonzero address returned by F-0bcb279b43b1. “State object” means the address read from a global location after F-2fd8826557a4 returns.

These offsets identify accesses, not independently established structure declarations. In particular, the selected fields are not proven to lie in disjoint arrays.

**Preconditions for defined normal execution:**

- The initial byte at input address plus `0xB50` must be readable. The later null check does not protect this access.
- On the flag path, the byte at input address plus `0xB51` must be writable.
- On the accounting path, the candidate metadata and all computed state fields must support the indicated accesses.
- Dependencies must return normally where execution is expected to continue, except for F-a817d8d58c10, which is represented as non-returning.
- Metadata selectors and the index returned by F-09dcb55e8e0a must produce valid effective addresses. This function performs no visible bounds checks.

The access widths are established by the supplied representation. Endianness, actual alignment requirements, and portable behavior for invalid or overflowing address computations are not established.

## Behavior

### Initial flag handling

The first operation is an 8-bit read at input offset `0xB50`.

If the byte is nonzero:

1. Write the 8-bit value `1` at input offset `0xB51`.
2. Return without calling any dependency or performing the accounting path.

The original value at `0xB51` is not read or preserved. Any nonzero value at `0xB50` selects this path; the condition is not restricted to the value `1`.

If the byte is zero, the function then checks whether the input address is zero. A zero input returns at this point, but only after the earlier byte access. Consequently, this is not a safe null-input interface.

### Candidate acquisition

For a nonzero input whose initial flag is zero:

1. Call F-0bcb279b43b1 with no explicit arguments.
2. If its result is zero, return immediately.
3. Otherwise, call F-2fd8826557a4 with no explicit arguments.
4. Read the state-object address from a global location.

The state address is captured **after** F-2fd8826557a4. Changes that dependency makes to the global location therefore affect the captured address.

Subsequent visible state accesses use that captured address. The function does not visibly reload the global state pointer after later dependency calls.

The input object is not explicitly passed to any dependency. Beyond its early flag access and null check, it is not directly accessed on this path.

### Header validation

For a nonzero candidate, the function uses the 32-bit word at candidate offset `-8` as a header.

Validation rejects either of these conditions:

- The header is zero.
- Bit 0 of the header is set.

On rejection, call F-a817d8d58c10 with no explicit arguments. The supplied representation marks this dependency as non-returning. No continuation after that call is specified here.

The zero check and bit check are separate header expressions. With ordinary stable memory, their combined acceptance condition is “nonzero and even.” The representation does not establish an atomic snapshot or synchronization rule for concurrent modification between those expressions.

A redundant nonzero-candidate condition surrounds this processing, but there is no intervening visible reassignment of the candidate after its earlier zero-result return.

### Accounting update sequence

After successful validation, the following order is established. Dependencies may change memory; their effects must not be assumed absent.

1. **Call F-c0fc6d033caa**, passing the captured state address.

2. **Read the first selector**, an unsigned 16-bit value at candidate offset `-4`. Compute its selected state address using a four-byte stride and base offset `0x224`.

3. **Read the header again** and retain its value masked with `0x3FFFFFFC` as the initial accounting amount. This mask removes bits 0, 1, 30, and 31. The amount is a multiple of four in the range `0` through `0x3FFFFFFC`.

4. **Update the first selected field** by reading its 32-bit value, subtracting the initial amount, and storing the 32-bit result.

5. **Update the fixed field at `0x30C`** by the same read-subtract-write operation using the retained initial amount.

6. **Read the second selector**, an unsigned 16-bit value at candidate offset `-2`. Compute its selected state address using a four-byte stride and base offset `0xE0`.

7. **Update the second selected field** by subtracting the retained initial amount.

8. **Update the fixed field at `0x220`** by subtracting the retained initial amount.

9. **Call F-09dcb55e8e0a** with no explicit arguments.

10. **Update the count field selected by that result**, at state offset `0x310 + 4 × result`, by subtracting one from its current 32-bit value.

11. **Update the fixed count field at `0x390`** by subtracting one from its current 32-bit value.

12. **Read the header again**, mask this new value with `0x3FFFFFFC`, and subtract that amount from the current 32-bit field at state offset `0x08`.

13. **Call F-cc6c9b4a0f68**, passing the captured state address and the address of the candidate header at offset `-8`.

14. **Call F-e586bff64fb7**, passing the captured state address.

15. Return.

The first four amount-based updates use one retained masked amount. The update at state offset `0x08` uses a later header read. These amounts need not match if intervening writes or dependencies change the header.

Validation occurs before F-c0fc6d033caa. There is no visible validation repeated afterward. If that dependency changes the header, the subsequent accounting uses the changed masked value without rechecking whether the header is nonzero or has bit 0 clear.

### Arithmetic and aliasing

The accounting fields and count fields are accessed as 32-bit words. No visible underflow, overflow, or saturation checks precede the writes. Under the represented 32-bit machine-word arithmetic, subtraction wraps modulo `2^32`; an accounting field smaller than its deduction is not rejected.

The two metadata selectors are zero-extended from 16 bits. Each selector can range from `0` to `65,535`, yielding a stride contribution from `0` to `262,140` bytes. Those widths do not establish that the resulting state address is valid.

F-09dcb55e8e0a returns an integer used directly in address calculation. No local restriction to a nonnegative or bounded index is visible.

Access order matters if selected fields overlap:

- A selected amount field can coincide with a fixed field.
- The selected count field can coincide with `0x390`.
- Candidate metadata can, in principle, overlap state storage.

No disjointness guarantee is established. Each later read observes memory after earlier writes and dependency effects, rather than a snapshot of all initial field values.

## Side effects

Direct side effects are limited to:

- The single-byte flag write on the early-return path.
- Four 32-bit accounting deductions using the retained initial amount.
- Two 32-bit count decrements.
- One final 32-bit accounting deduction using a fresh masked header read.

The accounting path also invokes dependencies that may have additional effects.

No direct write to the candidate header or its selector fields is visible. That does not imply those fields remain unchanged: dependencies and overlapping state writes may alter them.

The function has no visible local rollback. Once an accounting write occurs, no restoration is shown if a later dependency fails to return or execution encounters an invalid access.

No ownership transfer, allocation lifetime, reference-count discipline, or actual memory release is established. Passing the header address to F-cc6c9b4a0f68 proves only that the address is supplied to that dependency.

There is no visible local locking or synchronization primitive. Whether the dependencies establish synchronization is unknown.

## Dependencies

| Opaque ID | Explicit interface used here | Established role |
|---|---|---|
| F-0bcb279b43b1 | No arguments; returns an address-like integer | Supplies the candidate; zero causes immediate return |
| F-2fd8826557a4 | No arguments; result unused | Runs before the global state address is captured |
| F-a817d8d58c10 | No arguments; represented as non-returning | Receives control when header validation fails |
| F-c0fc6d033caa | Captured state address | Runs after validation and before selector and accounting-amount reads |
| F-09dcb55e8e0a | No arguments; returns an integer | Supplies the selected count-field index |
| F-cc6c9b4a0f68 | Captured state address; candidate header address | Runs after all visible accounting writes |
| F-e586bff64fb7 | Captured state address | Runs after F-cc6c9b4a0f68 and before normal return |

The function also depends on a global location containing the state address. Its initialization, ownership, and validity rules are not supplied.

No dependency is assumed to preserve globals, candidate metadata, state fields, or storage lifetime.

## Acceptance examples

These are conditional contract examples, not executed tests. Unless stated otherwise, they assume valid, non-overlapping storage, stable metadata, returning dependencies, and no dependency-induced changes to the fields discussed.

### Nonzero initial flag

Given an input whose byte at `0xB50` is `7`, the function writes `1` to byte `0xB51` and returns. No dependency is called. Existing accounting state is not directly accessed.

### No candidate

Given a nonzero input with byte `0xB50` equal to zero, and F-0bcb279b43b1 returning zero, the function returns after that dependency call. F-2fd8826557a4 is not called, and the global state address is not read by this function.

### Rejected header

Given a nonzero candidate whose stable header is either zero or `0x21`, execution calls F-a817d8d58c10 after F-2fd8826557a4 and the global state-address read. No local accounting update precedes that rejection.

### Ordinary accounting

Given:

- Header `0x00000022`;
- First selector `2`;
- Second selector `3`;
- F-09dcb55e8e0a returning `4`;

the masked amount is `0x20`. The four initial deductions target state offsets `0x22C`, `0x30C`, `0xEC`, and `0x220`, in that order. The selected count decrement targets `0x320`, followed by the decrement at `0x390`. The final deduction at `0x08` is also `0x20` if the header remains unchanged.

### Accepted header with zero accounting amount

A stable header of `2` passes validation but produces a masked amount of zero. The amount-based read-modify-writes still occur. Both count decrements and both final dependency calls still occur.

### Fresh final amount

Suppose the retained initial amount is `0x20`, but F-09dcb55e8e0a changes the candidate header so that its later masked value is `0x40`. The first four amount deductions use `0x20`; the deduction at state offset `0x08` uses `0x40`.

### Overlapping count fields

If F-09dcb55e8e0a returns `32`, the selected count address is state offset `0x390`. That word is decremented twice sequentially, not once.

### Underflow

If a targeted field contains `0x10` and its deduction is `0x20`, the represented 32-bit result is `0xFFFFFFF0`. No visible error branch handles that underflow.

## Uncertainties

The following remain **documented_only**:

- The application-level meanings of the input flags, candidate header bits, selectors, and accounting fields.
- Whether the accounting path actually frees, unlinks, caches, or otherwise changes ownership of the candidate.
- Whether dependency calls synchronize access or mutate metadata.
- Valid selector ranges and the valid range of the index from F-09dcb55e8e0a.
- Whether candidate storage and state storage are guaranteed disjoint.
- Failure behavior for invalid addresses, invalid indices, corrupted state pointers, and non-returning dependencies other than the designated rejection dependency.
- The behavior if F-a817d8d58c10 unexpectedly returns.
- Concurrency semantics and whether repeated reads can observe different values.
- Target byte order, alignment constraints, and portable-language treatment of overflowing address calculations.

The allocator-bookkeeping interpretation is inferred intent only. It must not replace the explicit access and call contract.

## Evidence

**Established static evidence:** The supplied representation shows the initial byte access before the null check, early-return conditions, dependency-call order, post-dependency global state capture, metadata widths, field offsets, mask `0x3FFFFFFC`, sequential accounting writes, and final fresh header read.

**Inferred intent:** The header-before-candidate layout, masked amount, selected accounting fields, and paired count decrements suggest allocation-related bookkeeping. No dependency implementation establishes that interpretation conclusively.

**Verification status:** No runtime tests have been performed. Acceptance examples are derived conditional expectations, not observed executions.
