<!-- Trusted provenance: F-49ef92ddd2b2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-49ef92ddd2b2

## Purpose

F-49ef92ddd2b2 attempts to claim one available entry from either of two eight-entry banks of byte-sized state.

A caller-supplied request byte selects the eligible banks. Bit 0 selects the first bank; bit 1 selects the second. An entry is available only when its state byte is exactly zero. Claiming an entry changes that byte to 1, reports the entry’s bank-relative index, and replaces the request byte with the selected bank’s identifier.

The first bank has priority when both selection bits are set. The second bank is considered only if the first bank is not requested or has no available entry.

Established behavior is a scan-and-mark operation. An interpretation as resource allocation is inferred intent: the supplied evidence does not identify what entries represent, how they become available again, or how callers use a claimed entry.

## Interface

### Inputs and outputs

The function accepts two pointer arguments and returns a 32-bit result.

| Interface element | Contract |
|---|---|
| Request pointer | Addresses one readable byte. Its low two bits select banks. On success, this byte is also written. |
| Index-output pointer | Addresses one byte that is written on success with an index from 0 through 7, subject to overlapping-storage effects described below. Its previous value is not read. |
| Return value | A 32-bit value: 0 for a successful claim, or hexadecimal 0x15, decimal 21, when no claim is made. |

The request byte is both an input selector and a success output:

- Input bit 0 enables the first bank.
- Input bit 1 enables the second bank.
- Other input bits do not select additional behavior.
- A successful first-bank claim replaces the entire byte with 1.
- A successful second-bank claim replaces the entire byte with 2.

Consequently, success does not preserve unrelated request bits.

The index is relative to the selected bank. A returned index of 3 denotes the fourth entry in either bank; it is not an absolute offset within the combined state region.

### Shared state layout

A shared location supplies a base address for byte-addressed state:

| Offset from base | Meaning |
|---|---|
| 0 through 7 | Eight entries in the first bank |
| 8 through 15 | Eight entries in the second bank |

Each entry occupies one byte. Zero means available for this operation. Every nonzero byte is treated as unavailable, regardless of its particular value or any possible signed interpretation.

The function writes the value 1 when claiming an entry. It does not normalize or otherwise modify entries it skips.

### Access preconditions

There is no explicit pointer validation.

For normal, non-overlapping use, the caller must provide readable request storage, writable request and output storage if a claim succeeds, and readable state storage for every entry actually inspected. A selected state entry must also be writable when claimed.

The function does not necessarily access all sixteen state bytes. It stops at the first available entry in the first bank that yields a claim. If neither selection bit is set, no state entry is inspected, although the shared base-address location is still read.

No alignment requirement beyond that needed by the underlying address representation is established for the byte arguments or state entries.

## Behavior

### Initial observation

Before testing either selection bit, the function reads the shared base address and retains that address for later accesses.

Its initial result is 0x15.

It then reads the request byte and tests bit 0. The two bank decisions are not based on a single saved copy of the request byte: the request byte is read again before deciding whether to scan the second bank.

### First-bank search

If bit 0 is set, the function inspects first-bank entries in ascending index order, beginning at index 0 and ending at index 7 if necessary.

For each candidate, it reads the state byte and compares it with zero. The first zero byte ends the search successfully. Nonzero bytes are skipped without writes.

An important address distinction is visible in the supplied representation:

- First-bank candidate reads obtain their base address from the shared base-address location.
- The eventual first-bank claim write uses the base address retained at function entry.

If the shared base remains stable, these refer to the same region. Stability is not checked. If the base changes during execution, the function can inspect a byte in one region and mark the corresponding offset in another.

For a successful first-bank search, externally relevant writes occur in this order:

1. Write the selected index, 0 through 7, to the index-output byte.
2. Replace the request byte with 1.
3. Write 1 to the selected first-bank state byte, using the retained base.

The function records a successful result and leaves this search.

If all eight first-bank entries are nonzero, this search makes no writes and leaves the result at 0x15.

### Second-bank decision and search

After the first-bank phase, the function reads the request byte again and tests bit 1.

Under stable, ordinary memory semantics, a successful first-bank claim has already replaced the request byte with 1. Bit 1 is therefore clear, and the second bank is not searched. This implements first-bank priority when both banks were initially requested.

If bit 1 is set at this later read, the function searches the second bank in ascending bank-relative index order, 0 through 7. Each candidate is located at the retained base plus 8 plus the candidate index.

Unlike first-bank candidate reads, these reads use the retained base rather than obtaining a fresh base from the shared location.

On the first zero state byte, writes occur in this order:

1. Write the bank-relative index to the index-output byte.
2. Replace the request byte with 2.
3. Write 1 to the selected second-bank state byte.

The function then returns 0 immediately.

If all eight second-bank entries are nonzero, that search makes no writes. The function returns the result already held: ordinarily 0x15 in this path.

### Boundaries and arithmetic

Each bank has exactly eight candidates. Index 8 is never inspected as a bank-relative candidate.

The index range is 0 through 7, so storing the index in one byte loses no information. The largest state offset used is 15. No multiplication or caller-supplied count controls the search.

The supplied representation establishes byte-sized state and output accesses and a 32-bit return value. It does not independently establish the machine pointer width or the behavior of invalid or overflowing address calculations.

### Failure behavior

With stable shared state and non-overlapping storage, the function returns 0x15 without modifying either argument or any state entry when:

- Neither bank-selection bit is set.
- Only the first bank is selected and all its entries are nonzero.
- Only the second bank is selected and all its entries are nonzero.
- Both banks are selected and all sixteen entries are nonzero.

There is no separate result for an invalid request value, invalid address, corrupt shared base, or concurrent interference. No recovery path for such conditions is shown.

## Side effects

A normal successful call performs three byte writes: index output, request replacement, and state marking, in that order. These writes do not form an atomic transaction in the supplied behavior.

No explicit allocation, deallocation, ownership transfer, callback, logging, or synchronization appears. The function marks shared state; that alone does not establish ownership or lifetime rules for any resource represented by an entry.

### Overlapping storage

The supplied behavior does not reject aliases among the argument bytes, state bytes, or shared address storage. Ordinary output guarantees therefore require care when these locations overlap.

For example, if both arguments address the same byte and the state region is separate, the index write is immediately overwritten by the request write. The shared argument byte ends as 1 for a first-bank claim or 2 for a second-bank claim, not necessarily as the selected index.

If the selected state byte aliases an argument byte, the final state-marker write can overwrite an earlier output. Writes through argument pointers may also affect other state entries if those pointers overlap the state region.

Accordingly, “the output contains the selected index” and “the request contains the selected bank identifier” are simultaneous postconditions only when relevant storage does not overlap in a conflicting way. The write sequence remains the established behavior.

### Concurrent access

No lock, compare-and-exchange, reservation primitive, or revalidation is visible. Two unsynchronized callers could observe the same zero entry before either marks it and could both report success.

The repeated request read and the differing base-address access patterns also make external mutation potentially significant. Exact outcomes under concurrent access are not established without the platform memory model and surrounding synchronization rules.

## Dependencies

No dependency function calls appear in the supplied evidence. There are therefore no opaque callee IDs to enumerate.

The direct dependencies are:

- The caller-provided request byte.
- The caller-provided index-output byte.
- The shared base-address location.
- The state bytes reached through that address.

No assumption about an unknown callee preserving globals, allocating storage, or releasing memory is needed or justified.

## Acceptance examples

The following examples are conditional acceptance criteria, not runtime observations. Unless stated otherwise, assume stable shared state, valid memory, no concurrent writers, and distinct request, output, and state storage.

1. **Neither bank selected.**  
   Request is 0x80, and the output initially contains 0xA5. The function returns 21. Request remains 0x80, output remains 0xA5, and no state entry is read or written. The shared base-address location is still read.

2. **First available first-bank entry wins.**  
   Request is 1. First-bank entries 0 and 1 are nonzero, and entry 2 is zero. The function returns 0, writes output index 2, leaves request as 1, and marks first-bank entry 2 as 1. Later first-bank entries and all second-bank entries are not inspected.

3. **Both banks selected, first bank succeeds.**  
   Request is 3, and both banks have an available entry at index 0. The first-bank entry is claimed, output is 0, and request becomes 1. The second bank is not searched or modified.

4. **First bank exhausted, second bank succeeds.**  
   Request is 3. All first-bank entries are nonzero. Second-bank entries 0 through 5 are nonzero, and entry 6 is zero. The function returns 0, output becomes 6, request becomes 2, and the state byte at combined offset 14 becomes 1.

5. **Second-bank-only request.**  
   Request is 2, and second-bank entry 7 is its first zero entry. The first bank is not inspected. Success reports index 7, not combined offset 15.

6. **Nonzero does not mean available.**  
   A state byte containing 0xFF is skipped just like a byte containing 1. Only an exact zero qualifies.

7. **All requested entries unavailable.**  
   Request is 3 and all sixteen entries are nonzero. The function returns 21 without changing request, output, or state.

8. **Unrelated request bits are discarded on success.**  
   Request is 0x82 and the second bank has an available entry. Success replaces the request byte with 2, rather than preserving bit 7.

9. **Arguments alias.**  
   Both argument pointers address one byte initially containing 2. The second bank’s first available entry is index 5, and state storage is separate. The function first writes 5 to the shared argument byte, then overwrites it with 2, marks the entry, and returns 0.

## Uncertainties

The following are **documented_only** because the supplied evidence does not resolve them:

- The meaning and names of the two banks.
- The system-level meaning of result 21 beyond “no claim made” in ordinary execution.
- Initialization, lifetime, and validity guarantees for the shared base and state region.
- Release behavior or any ownership protocol outside this function.
- Whether the wider calling contract prohibits argument or state aliasing.
- Whether shared storage is volatile, device-backed, or concurrently mutable.
- The platform’s pointer width, fault handling, memory ordering, and synchronization requirements.
- Any intended guarantee that first-bank inspection and marking use the same state region.

The allocation-like purpose is inferred intent. The byte tests, scan bounds, write ordering, address-source distinction, and result values are established static behavior.

## Evidence

This contract is based solely on the supplied static representation of F-49ef92ddd2b2.

The representation supports two eight-entry searches, exact-zero availability tests, ascending scan order, byte-sized writes, bank-relative indices, request replacement, a 32-bit success value of 0, and a default result of 0x15. It also exposes the initial retained base address, fresh shared-base references during first-bank inspection, retained-base accesses elsewhere, and the second request-byte read.

No runtime tests have been performed. No external documentation, callee implementation, platform definition, or ownership specification was supplied. The acceptance examples describe expected behavior under their stated conditions; they are not reported test results.
