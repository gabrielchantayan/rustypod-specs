<!-- Trusted provenance: F-da2fe84eecde; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-da2fe84eecde

## Purpose

F-da2fe84eecde writes a supplied 16-bit value into a two-byte slot selected by a base address and a signed index. The slot lies at byte offset 28 within a 32-byte stride.

Before performing that write, the function compares the index with 2. An index greater than 2 invokes F-a817d8d58c10, which is identified by the supplied static evidence as non-returning. The function contains no explicit lower-bound check: negative indices take the direct-write path.

This is an address-calculation and store contract, not evidence of a complete container abstraction. A likely intent is to update one field among three indexed records, but neither the existence of such records nor the validity of their storage is established.

## Interface

The function has three inputs and no declared result value:

| Input | Recovered representation | Role |
|---|---|---|
| Base address | Integer-form address, presented as an `int` | Origin for destination-address calculation. |
| Index | Signed integer, presented as an `int` | Controls both the upper-bound branch and the 32-byte stride displacement. |
| Value | Two-byte quantity | Supplies the 16 bits written to the destination. |

The recovered interface indicates 32-bit integer inputs and a 16-bit stored quantity. The exact machine ABI, pointer representation, and argument-passing mechanism are not supplied independently. Accordingly, 32-bit arithmetic is the recovered model; its overflow behavior is not established by the text alone.

The value has no established signedness or semantic encoding. The function stores its 16-bit representation without an explicit transformation. It does not interpret the value as a flag, character, enumeration, or signed number.

For an execution that reaches the store, the destination is:

**base address + (index × 32) + 28 bytes**

The destination must support a two-byte write under the target machine’s access rules. This requirement includes address validity, write permission, and any applicable alignment restrictions. The function does not check these conditions.

There is no explicit allocation, ownership transfer, or release on the direct-write path. The base address does not become owned merely because it is supplied to this function.

## Behavior

### 1. Index classification

The first explicit operation affecting control flow is a signed comparison of the index against 2.

- An index of 2 or less bypasses F-a817d8d58c10 and proceeds toward the store.
- An index greater than 2 invokes F-a817d8d58c10 before any destination write performed explicitly by this function.

This comparison is not a complete bounds check. In particular, it admits all representable negative indices.

The function performs no explicit read from the base-addressed storage before making this decision. Its branch depends on the supplied index, not on a field or size value loaded through the base address.

### 2. Dependency invocation for indices above 2

F-a817d8d58c10 is invoked with no explicit arguments in the supplied representation. Static annotation identifies this dependency as non-returning.

Under that non-returning characterization, an invocation with an index greater than 2 does not reach this function’s explicit store or normal return. The dependency’s actual outcome is unknown: the evidence does not distinguish termination, a trap, an exception, a nonlocal transfer, an indefinite operation, or another non-returning mechanism.

The call does not establish that no memory changes occur. The dependency may have effects of its own, and its treatment of global state or memory ownership is not documented here.

If the non-returning characterization is inaccurate and the dependency returns normally, the visible continuation proceeds to the same destination store. There is no second comparison or explicit early return after the call. This is a conditional consequence of the visible control flow, not an assertion that the dependency returns in practice.

### 3. Destination calculation

For executions reaching the write, the index selects a displacement in 32-byte increments, followed by a fixed 28-byte offset.

For indices ordinarily suggestive of a three-entry layout, the destination starts at:

| Index | Destination offset from the base |
|---:|---:|
| 0 | 28 bytes |
| 1 | 60 bytes |
| 2 | 92 bytes |

A negative index moves the destination backward. For example, index −1 produces a mathematical displacement of −4 bytes. No explicit branch rejects it.

These offsets describe address selection, not storage validation. Even an index of 0 can fail if the resulting destination is invalid or unwritable.

The fixed positive-index examples do not require index-multiplication overflow, but adding them to a base near an address-space boundary may still exceed the representable address range. For sufficiently negative indices, multiplying by 32 may also exceed a signed 32-bit range. The supplied evidence does not resolve whether such cases wrap, fault, or fall outside the applicable execution semantics.

### 4. Store and return

The function performs one explicit two-byte store at the calculated destination. It replaces those two bytes with the representation of the supplied value.

No explicit read-modify-write operation appears. The previous destination contents do not influence the stored value. There is no explicit second field update, counter change, status assignment, or result-value construction.

After the store completes, the function returns normally without a declared result value.

The two-byte store does not by itself establish atomicity, synchronization, durability, or visibility guarantees to concurrent observers. Those properties depend on the target architecture and surrounding execution environment.

## Side effects

On the direct-write path, the sole explicit external memory effect is the two-byte destination write. Bytes outside that destination are not explicitly written by this function.

The address-shaped input is used as a numerical origin; the function does not explicitly load a pointer, length, or metadata through it. The destination is not validated against an allocation boundary or record count.

For indices above 2, the dependency call occurs before this function’s explicit store. Its effects must be accounted for separately. In particular, callers must not infer that it preserves globals, retains allocations, leaves the destination unchanged, or returns control.

No local error status is produced. Invalid destination accesses are not caught or translated into a documented return value. Any resulting fault or other abnormal behavior depends on the execution environment.

## Dependencies

### F-a817d8d58c10

- **Invocation condition:** the signed index is greater than 2.
- **Explicit arguments:** none shown.
- **Return characterization:** identified as non-returning by the supplied static annotation.
- **Known ordering:** invoked before the destination store.
- **Unknown effects:** memory access, global-state changes, allocation and release behavior, diagnostics, and the mechanism by which control fails to return.

The dependency’s name and placement do not establish that it is an assertion handler or bounds-error reporter. Such interpretations remain inferred intent, not part of the established contract.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. No runtime tests have been performed.

1. **Index zero.** Given a base whose bytes at offsets 28 and 29 are writable, and arithmetic valid for that base, index 0 writes the supplied 16-bit representation there and returns normally. F-a817d8d58c10 is not invoked.

2. **Largest directly admitted nonnegative index.** With equivalent storage and arithmetic preconditions, index 2 writes at offsets 92 and 93. It does not invoke the dependency.

3. **First index above the threshold.** Index 3 invokes F-a817d8d58c10 before this function performs a destination store. If the dependency is non-returning as annotated, this function does not reach its store or normal return.

4. **Negative index.** Given valid arithmetic and writable storage four bytes before the base, index −1 writes two bytes beginning there. Rejecting this index solely because it is negative would differ from the supplied behavior.

5. **Replacement rather than combination.** For a valid destination containing arbitrary prior bytes, the resulting two-byte representation is the supplied value, not a combination with the prior contents. Neighboring bytes are not explicitly modified by this function.

6. **Byte ordering.** If the supplied value has distinct high and low bytes, their order in memory follows the target’s two-byte representation. Neither little-endian nor big-endian ordering can be selected from this evidence alone.

7. **Returning dependency, conditionally.** If F-a817d8d58c10 returns despite its annotation, index 3 continues toward a write beginning 124 bytes beyond the base, subject to address validity and any effects of the call. No explicit guard prevents that continuation.

## Uncertainties

The following remain **documented_only**:

- Exact target ABI, pointer width, and integer-to-address conversion semantics.
- Overflow behavior for stride multiplication and address addition.
- Endianness and alignment requirements for the two-byte store.
- Whether the storage consists of records, and what the offset-28 field means.
- Whether callers guarantee nonnegative indices or valid writable storage.
- Whether F-a817d8d58c10 is genuinely non-returning and what effects it has.
- Atomicity, concurrency, and memory-ordering properties.
- Runtime behavior for invalid addresses or unsupported alignment.

The three-entry-record interpretation is inferred intent only. It must not replace the established signed comparison or introduce a lower-bound rejection absent from the supplied behavior.

## Evidence

Established static evidence consists of a signed upper-threshold comparison, a conditional dependency call, one two-byte store using a 32-byte stride and 28-byte fixed offset, and a normal return after that store. The dependency is statically marked non-returning.

The supplied material includes no dependency implementation, caller contracts, allocation layout, target ABI specification, or execution observations. No runtime tests have been performed.
