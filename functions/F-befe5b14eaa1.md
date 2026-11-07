<!-- Trusted provenance: F-befe5b14eaa1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-befe5b14eaa1

## Purpose

F-befe5b14eaa1 conditionally invokes a two-argument operation and then processes an associated allocation through validation, accounting updates, and two dependency calls.

The established behavior has three principal outcomes:

- No action beyond an initial lookup when no object is returned.
- No further action when the returned object has a nonzero 32-bit field at byte offset `+4`.
- Invocation of the two-argument operation, followed by conditional allocation processing when that field is zero.

The accounting pattern is consistent with allocation-release bookkeeping, but release, allocator ownership, synchronization, and the meaning of the counters are **inferred intent**, not established dependency behavior.

## Interface

**Inputs**

Two opaque 32-bit arguments are accepted. They are forwarded, in their original order, only to F-b5a86af15404. This function does not directly interpret them as integers, addresses, flags, or handles.

**Output**

There is no defined return value. Normal completion communicates through dependency calls and memory side effects.

One validation-failure path invokes F-a817d8d58c10, which is identified as nonreturning in the supplied static representation. Consequently, that path has no established normal continuation.

**Address and integer model**

The supplied representation uses 32-bit addresses and 32-bit accounting words. Allocation metadata includes two unsigned 16-bit indices. Each index is zero-extended before multiplication by four.

Accounting additions and subtractions are 32-bit operations. Their stored bit patterns wrap modulo \(2^{32}\); the function shows no saturation, overflow detection, or underflow rejection. This statement describes the represented machine-width effects, not permission to access an invalid computed address.

**Required memory conditions**

For any reached access to complete normally:

- The initial nonzero object address must permit a 32-bit read at offset `+4`.
- A later nonzero allocation address must permit the metadata reads described below.
- The captured context address and all index-derived locations must permit their respective reads and writes.
- Dependencies must return normally where subsequent behavior is expected.

The function does not visibly establish these conditions with bounds checks. The two supplied arguments have no additional locally established validity requirements.

## Behavior

### Initial selection

The function first calls F-aaf1c55e4172 without explicit arguments.

If its result is zero, the function returns immediately. No object-field read, forwarding of the two input arguments, allocation processing, or accounting update occurs on this path.

For a nonzero result, the function reads the 32-bit field at byte offset `+4` from that object.

If the field is nonzero, the function returns immediately. The exact nonzero value does not otherwise affect this function.

If the field is zero, the function calls F-b5a86af15404 with the two original arguments. Its result is not used.

The function then passes the previously obtained object address to an unresolved dependency. This dependency’s identity is not supplied by the opaque dependency map. Its result is treated as an allocation address.

The object address is retained across the preceding call: the function does not repeat the initial lookup or reread the object’s `+4` field before invoking the unresolved dependency. Whether the preceding call modifies or invalidates that object is not established.

If the unresolved dependency returns zero, the function returns. F-b5a86af15404 has already been called, but the later helper, metadata validation, and accounting updates do not occur.

### Context capture and allocation validation

For a nonzero allocation address, the function calls F-2fd8826557a4 without explicit arguments.

After that call returns, it reads a globally referenced 32-bit cell and captures its value as the context address. Reading the context after the helper is significant: a context change made by that helper can affect the captured address.

The function then validates the allocation header, located eight bytes before the allocation address:

- A zero 32-bit header causes a call to F-a817d8d58c10.
- Otherwise, the header is read again, and a set least-significant bit also causes a call to F-a817d8d58c10.

The second header read is conditional on the first read being nonzero. The represented behavior therefore does not reduce these checks to a guaranteed single snapshot.

No accounting writes precede this validation. However, F-b5a86af15404, the unresolved dependency, and F-2fd8826557a4 have already been invoked. The globally referenced context cell has also already been read.

For either validation failure, F-a817d8d58c10 is the identified nonreturning dependency. Its diagnostic output, termination mechanism, and other effects are unknown.

### Metadata layout

The allocation address denotes the location after the following metadata:

| Relative byte offset | Width | Established use |
|---|---:|---|
| `−8` | 32 bits | Header; checked for zero and low-bit validity, and masked to derive accounting quantities |
| `−4` | 16 bits, unsigned | Index for a context word beginning at offset `0x224` |
| `−2` | 16 bits, unsigned | Index for a context word beginning at offset `0x0E0` |

The accounting mask is `0x3FFFFFFC`. It clears bits 31, 30, 1, and 0 and retains bits 2 through 29. The resulting quantity is a multiple of four between zero and `0x3FFFFFFC`, inclusive.

Passing validation does not establish that the masked quantity is positive. For example, a stable header of `0x00000002` passes the shown checks but produces a zero accounting quantity.

### Ordered accounting updates

After successful validation, the function calls F-c0fc6d033caa with the captured context address.

The subsequent operations use that captured address. They do not reread the global context cell, even if this dependency changes it. Conversely, metadata values are read after this call; they are not necessarily the values present during validation.

The following direct reads and writes occur in order:

1. Read the unsigned 16-bit index at allocation offset `−4` and form the context location at `0x224 + 4 × index`.
2. Read the 32-bit allocation header and mask it with `0x3FFFFFFC`. Retain this quantity for the next four subtractions.
3. Read, subtract from, and write the 32-bit word at the indexed `0x224` location.
4. Read, subtract from, and write the 32-bit word at context offset `0x30C`.
5. Read the unsigned 16-bit index at allocation offset `−2` and form the context location at `0x0E0 + 4 × index`.
6. Read, subtract from, and write the 32-bit word at the indexed `0x0E0` location.
7. Read, subtract from, and write the 32-bit word at context offset `0x220`.
8. Call F-09dcb55e8e0a without explicit arguments.
9. Use that dependency’s returned index to locate the 32-bit context word at `0x310 + 4 × index`, then decrement it by one.
10. Decrement the 32-bit context word at offset `0x390` by one.
11. Read the 32-bit allocation header again, apply `0x3FFFFFFC` again, and subtract this newly obtained quantity from the 32-bit context word at offset `0x008`.
12. Call F-cc6c9b4a0f68 with the captured context address and the address of the header at allocation offset `−8`.
13. Call F-e586bff64fb7 with the captured context address.
14. Return normally if these calls return.

Each listed update reads the destination’s then-current value before writing its result. If computed locations alias, later updates operate on values left by earlier updates; the function does not consolidate them.

The first four subtractions share one masked quantity. The subtraction at context offset `0x008` uses a later header read, after F-09dcb55e8e0a. Equality between these quantities is not guaranteed.

There is no visible range check on the index returned by F-09dcb55e8e0a. Its valid range and interpretation are unresolved.

## Side effects

The function directly changes the context words listed above. It also invokes dependencies that may have additional effects not established by the supplied material.

Several ordering properties are part of the contract:

- The two-argument operation occurs before the allocation-producing dependency.
- A zero allocation result does not undo that operation.
- Context capture occurs after F-2fd8826557a4.
- Header validation occurs before F-c0fc6d033caa and before direct accounting writes.
- Four quantity-based subtractions occur before F-09dcb55e8e0a.
- Both unit decrements and the final quantity subtraction occur before F-cc6c9b4a0f68.
- F-e586bff64fb7 is reached only after F-cc6c9b4a0f68 returns.

No rollback is shown. If a dependency fails to return, or a reached memory access cannot complete, earlier effects are not visibly reversed.

The function performs no direct payload clearing and no explicit direct deallocation. Passing the header address to F-cc6c9b4a0f68 does not by itself establish that the allocation is released. Neither ownership transfer nor continued caller access to the allocation is specified by the available evidence.

No lock, atomic update, or concurrency guarantee is independently established. Helper placement may suggest a protected accounting sequence, but dependency semantics are required to confirm that interpretation.

## Dependencies

| Opaque dependency | Established local use |
|---|---|
| F-aaf1c55e4172 | Called first; supplies the nullable object address. |
| F-b5a86af15404 | Receives both original 32-bit arguments when the object’s `+4` field is zero. Its result is unused. |
| F-2fd8826557a4 | Called for a nonzero allocation result, before context capture. |
| F-a817d8d58c10 | Invoked for a zero header or a header with bit 0 set; identified as nonreturning. |
| F-c0fc6d033caa | Receives the captured context after validation and before accounting reads and writes. |
| F-09dcb55e8e0a | Supplies the index used for the unit decrement relative to context offset `0x310`. |
| F-cc6c9b4a0f68 | Receives the captured context and allocation-header address after direct accounting updates. |
| F-e586bff64fb7 | Receives the captured context after F-cc6c9b4a0f68 returns. |

One additional dependency is unresolved because the supplied map gives it no opaque ID. It receives the selected object address and returns the nullable allocation address. Its ownership and lifetime effects remain **documented_only**.

No dependency is assumed to preserve global state, metadata, or context contents unless established by additional evidence.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests.

1. **No selected object.** If F-aaf1c55e4172 returns zero, the function returns without forwarding either input argument or accessing object metadata.

2. **Nonzero object field.** If the lookup returns a readable object whose `+4` field is nonzero, the function returns after that field read. The unresolved allocation-producing dependency is not called.

3. **No allocation after the operation.** If the object field is zero and the unresolved dependency returns zero, F-b5a86af15404 must already have received the two original arguments. F-2fd8826557a4 and the accounting dependencies are not reached.

4. **Invalid stable header.** Given a nonzero allocation and a stable header of zero or `0x00000005`, F-a817d8d58c10 is invoked before any direct context-accounting write. Earlier dependency effects remain possible.

5. **Ordinary accounting.** Assume distinct destination words, valid indices, normally returning dependencies, no dependency mutations, and a stable header of `0x00000024`. The four early quantity counters and the counter at context offset `0x008` each decrease by 36. The selected `0x310`-relative counter and the counter at `0x390` each decrease by one. The final two dependency calls occur in the stated order.

6. **Different final quantity.** If the early masked header quantity is 36 and F-09dcb55e8e0a changes the header to `0x00000044`, the four earlier subtractions use 36, while the subtraction at `0x008` uses 68, assuming no intervening aliasing effects.

7. **Underflow.** If an otherwise unaffected quantity counter contains 8 and its applicable subtraction quantity is 12, its resulting 32-bit pattern is `0xFFFFFFFC`. A unit-decrement counter initially zero becomes `0xFFFFFFFF`. No local error branch rejects either transition.

8. **Aliased destinations.** If valid indices cause two accounting destinations to coincide, both updates apply in sequence. Acceptance must not expect only one update merely because the addresses match.

## Uncertainties

The following remain **documented_only**:

- The identity and contract of the unresolved dependency.
- The semantic meaning of the object’s `+4` field.
- The object, allocation, and context ownership models.
- Whether any dependency acquires or releases synchronization.
- Whether the header denotes size, capacity, or another accounting measure.
- The valid range and signedness contract of the index returned by F-09dcb55e8e0a.
- Whether allocator invariants prevent overlap among metadata and accounting destinations.
- Diagnostic and termination behavior of F-a817d8d58c10 beyond its nonreturning designation.
- Behavior when preconditions fail, including inaccessible memory or invalid computed addresses.

The release-accounting interpretation is inferred intent only. It does not establish deallocation, reference-counting semantics, or a particular allocator design.

## Evidence

This contract is based solely on the supplied static representation and opaque dependency mapping. That material establishes the represented branches, access widths, offsets, masks, dependency arguments, and operation ordering described above.

Embedded commentary and naming are not treated as independent proof of semantics. In particular, allocator roles, synchronization roles, and ownership effects are not inferred as guaranteed behavior from the call sequence.

No runtime tests have been performed. The acceptance examples are static, conditional predictions rather than observed results.
