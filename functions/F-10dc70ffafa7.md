<!-- Trusted provenance: F-10dc70ffafa7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-10dc70ffafa7

## Purpose

F-10dc70ffafa7 obtains a destination from a dependency, inspects record metadata reached through its fourth argument, and selects one of two paths.

When the inspected byte at record offset 2 is zero and the unsigned byte at offset 3 exceeds 11, the function passes an addressable 32-bit temporary to a dependency, performs two ordered 16-bit destination writes, and passes the temporary to a final dependency. The first write uses the temporary’s low-order numeric 16 bits. The second uses the two-byte portion of the temporary’s representation beginning at byte offset 2. Calling that second portion the high-order numeric 16 bits requires a little-endian interpretation that the supplied evidence does not independently establish.

Otherwise, the function passes the acquired destination to a different dependency with fixed numeric arguments 2 and 5.

These are established static operations. The semantic meanings of the record, destination, and temporary are unknown. Allocation, conversion, publication, and resource release are possible interpretations of individual dependencies, not established contracts.

## Interface

The function accepts four arguments and has no explicit return value.

| Argument | Recovered representation | Established use |
|---|---|---|
| First | 32-bit value | Passed to F-603a53835f04, followed by three zero arguments. |
| Second | 32-bit value | Passed unchanged to F-5cfc9479b2e4 on the qualifying path. |
| Third | 32-bit value | Passed unchanged to F-5cfc9479b2e4 on the qualifying path. |
| Fourth | Pointer to a 32-bit integer cell | Its pointer representation initializes a 32-bit temporary; the referenced cell supplies the address used for record inspection. |

The fourth argument has two distinct uses. Its own address representation initializes the temporary. The contents of its referenced cell are subsequently used to locate the record. Those values are not interchangeable.

The recovered representation assigns the pointer value to a 32-bit temporary. This establishes the operation in the supplied representation, not a portable pointer-conversion contract for environments with wider pointers.

The result of F-603a53835f04 is treated as a destination pointer supporting 16-bit units. On the qualifying path, the function directly writes one unit at byte offset 0 and one at byte offset 2.

### Preconditions for defined normal execution

At each relevant access:

- The fourth argument must identify a readable 32-bit cell.
- The address obtained from that cell must permit reading the record byte at offset 2.
- If the first inspected byte is zero, the address obtained for the second condition must permit reading the record byte at offset 3.
- On the qualifying path, the acquired destination must permit the two 16-bit writes, including any applicable alignment requirements.
- Each invoked dependency must accept its supplied arguments and return normally for subsequent operations to occur.

No explicit null, bounds, alignment, or dependency-failure checks appear.

These conditions apply when the accesses occur, not merely on entry. F-603a53835f04 runs before record inspection and may change relevant memory or global state; no preservation guarantee is supplied.

## Behavior

### Initialization and destination acquisition

Before invoking a dependency, the function initializes an addressable 32-bit temporary from the fourth argument’s pointer representation.

This initialization does not read the record address from the referenced cell. If the temporary remains unchanged, it represents the cell’s address rather than the address contained in that cell.

The function then invokes F-603a53835f04 with the first argument and three numeric zeros. Its returned value is retained as the destination for whichever branch is selected.

This call precedes both predicate byte accesses. The supplied evidence does not establish that it allocates storage, returns a fresh object, preserves referenced memory, or reports failure through any particular result.

### Record inspection

After destination acquisition returns, the function reads the cell referenced by the fourth argument and uses the resulting value to access the byte at record offset 2.

If that byte is nonzero, the predicate fails. The caller does not evaluate the offset-3 byte access.

If the offset-2 byte is zero, the function evaluates a second condition using the byte at record offset 3 as an unsigned 8-bit value:

- Values 0 through 11 fail.
- Values 12 through 255 qualify.

The comparison with zero does not depend on whether the offset-2 byte is signed. The unsigned interpretation of the offset-3 byte is significant: values 128 through 255 satisfy the threshold.

The second condition contains another use of the cell referenced by the fourth argument. The supplied representation does not establish a single record-address snapshot reused by both conditions. With stable memory, both accesses address the same record. Exact machine-load behavior, concurrent mutation, and volatile semantics are not established.

### Qualifying path

When both conditions hold, the function invokes F-5cfc9479b2e4 with:

1. The address of the 32-bit temporary.
2. The second argument.
3. The third argument.

The caller does not consume this dependency’s return value. Because the temporary’s address is supplied, the dependency may modify its contents. No particular resulting value is established.

After that call returns normally, the function performs these direct writes in order:

| Order | Destination | Source interpretation |
|---|---|---|
| First | 16-bit unit at byte offset 0 | Numeric narrowing of the temporary to its low-order 16 bits. |
| Second | 16-bit unit at byte offset 2 | The two-byte portion of the temporary’s representation beginning at byte offset 2. |

The second source selection is representation-relative. Under an ordinary little-endian 32-bit layout, it supplies the temporary’s high-order numeric 16 bits. Without an established target byte order, that numeric interpretation must remain conditional.

The first source selection is different: its low-order numeric value follows from narrowing, independently of the destination’s byte order. The physical ordering of the two bytes written for that 16-bit value still depends on the execution environment.

These are separate 16-bit writes. The supplied representation does not establish an atomic 32-bit transfer. A permitted observer between the writes could encounter a partially updated destination.

After both writes, the function invokes F-9b718979506d with the address of the same temporary, then returns without a value. This call is not conditioned on whether the earlier dependency modified the temporary.

There is no direct destination write by the caller after F-9b718979506d. That does not establish the destination’s final contents: the dependency may affect aliased or otherwise accessible storage.

### Nonqualifying path

If either predicate condition fails, the function invokes F-7f008c1060db with:

1. The destination returned by F-603a53835f04.
2. Numeric value 2.
3. Numeric value 5.

It then returns without a value.

On this path, the caller does not invoke F-5cfc9479b2e4 or F-9b718979506d and does not perform the two direct 16-bit destination writes. The effects of F-7f008c1060db remain unknown; absence of direct caller writes does not imply an unchanged destination.

### Widths, layout, and arithmetic

The temporary is 32 bits. Each direct destination write is 16 bits. The destination positions are adjacent and cover four bytes, assuming ordinary byte-addressed storage with the recovered unit sizes.

The record accesses use fixed byte offsets 2 and 3. The second destination write uses byte offset 2. There is no input-dependent length calculation or allocation-size calculation.

Even with stable, nonoverlapping storage, the two writes should not unconditionally be described as reproducing the temporary’s complete 32-bit value. That interpretation follows under a compatible little-endian layout; it is not established independently of representation.

No checked address arithmetic or overflow handling is visible. Invalid addresses, arithmetic overflow in forming addresses, and inaccessible storage have no specified recovery behavior in the supplied evidence.

## Side effects

The following order is established at the level of the supplied representation, assuming normal completion of preceding operations:

| Phase | Caller-visible operation |
|---|---|
| Initialization | Initialize the 32-bit temporary from the fourth argument’s pointer representation. |
| Destination acquisition | Invoke F-603a53835f04. |
| First predicate condition | Obtain the record address through the referenced cell and inspect byte offset 2. |
| Second predicate condition | Only if the first byte is zero, evaluate the unsigned byte at offset 3 through another use of the referenced cell. |
| Qualifying path | Invoke F-5cfc9479b2e4; write the low-order numeric 16 bits; write the representation portion selected at temporary byte offset 2; invoke F-9b718979506d. |
| Nonqualifying path | Invoke F-7f008c1060db with the destination, 2, and 5. |

Dependency-internal effects may occur within each call. Nothing establishes that dependencies preserve globals, referenced cells, records, or destination storage.

### Ownership and lifetime

The caller uses the acquired destination without visibly establishing ownership. No explicit destination release appears.

The temporary is local addressable storage whose address is passed to two dependencies on the qualifying path. The supplied evidence does not establish whether those dependencies retain that address or interpret its stored value as a resource reference. Retention must not be assumed safe beyond the temporary’s lifetime.

The final dependency’s position after the writes does not establish cleanup, destruction, or ownership transfer.

### Aliasing

No nonaliasing guarantee is supplied.

The first destination write occurs before the second source portion is used. If destination storage overlaps the temporary, the first write may change the representation from which the second portion is selected. Consequently, interpreting both writes as derived from one unchanged post-dependency value requires a noninterference assumption.

Dependencies may also modify aliased storage. In particular, the final dependency may change the destination through an access path not visible in this caller.

### Errors and interrupted execution

The function exposes no explicit error result, retry, rollback, or failure translation.

The acquired destination is not checked for null before it is written or forwarded. No status result from the branch-specific dependencies is inspected.

If a dependency does not return normally, later operations on that path are not established to occur. A failure after the first destination write is not accompanied by visible restoration of the previous contents. Invalid memory accesses have no visible recovery path.

## Dependencies

| Opaque function ID | Invocation condition | Supplied arguments | Established relationship |
|---|---|---|---|
| F-603a53835f04 | Always, after temporary initialization | First argument, 0, 0, 0 | Its result supplies the destination. |
| F-5cfc9479b2e4 | Both predicate conditions hold | Temporary address, second argument, third argument | Runs before the two direct destination writes. |
| F-7f008c1060db | Either predicate condition fails | Destination, 2, 5 | Runs instead of the qualifying-path calls and writes. |
| F-9b718979506d | Qualifying path, after both direct writes | Temporary address | Last dependency invoked before normal return on this path. |

Dependency bodies and independent contracts are unavailable. Their argument meanings, failure modes, memory ownership, retention behavior, and effects on shared state remain **documented_only**.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Unless otherwise stated, they assume valid memory, stable predicate inputs, noninterfering storage, and normally returning dependencies.

1. **Nonzero first inspected byte.**  
   If record offset 2 contains 1, F-603a53835f04 runs before F-7f008c1060db. The latter receives the acquired destination, 2, and 5. The caller does not inspect offset 3 for its predicate and performs neither direct destination write.

2. **Last failing threshold value.**  
   If offset 2 contains zero and offset 3 contains 11, the nonqualifying path is selected.

3. **First qualifying threshold value.**  
   If offset 2 contains zero and offset 3 contains 12, F-5cfc9479b2e4 runs, followed by both ordered destination writes and F-9b718979506d. F-7f008c1060db is not invoked.

4. **Unsigned upper range.**  
   If offset 2 contains zero and offset 3 contains 255, the qualifying path is selected. Interpreting that second byte as a negative signed value would not match the represented comparison.

5. **Known prepared value with explicit layout assumptions.**  
   If F-5cfc9479b2e4 leaves the temporary with numeric value hexadecimal 12345678, the first direct write has numeric value hexadecimal 5678. The second write uses the two-byte portion at temporary byte offset 2. Under a little-endian 32-bit layout, that portion has numeric value hexadecimal 1234. Without that layout assumption, hexadecimal 1234 is not an established second-write value. These criteria describe the direct writes, not necessarily the destination after the final dependency.

6. **Preparation leaves the temporary unchanged.**  
   If F-5cfc9479b2e4 leaves the temporary unchanged, its contents still derive from the fourth argument’s pointer representation, not the record address stored in the referenced cell. The first write narrows that representation numerically; the second selects its bytes at offset 2.

7. **Destination acquisition changes predicate inputs.**  
   If F-603a53835f04 changes the referenced cell or record before returning, branch selection follows the subsequently evaluated accesses. The caller captures no predicate byte before that dependency.

8. **Short-circuited second access.**  
   If offset 2 is readable and nonzero, inability to read offset 3 does not itself obstruct the visible predicate: the caller does not evaluate that access. Other accesses and dependency requirements still apply.

## Uncertainties

The following remain **documented_only**:

- The record’s semantic meaning and layout beyond the inspected offsets.
- The semantic meanings of the arguments and constants 2 and 5.
- Destination ownership, capacity, alignment, and lifetime.
- Dependency contracts, failure behavior, and effects on shared state.
- Whether either temporary-receiving dependency retains its address.
- Whether the final dependency releases or finalizes anything.
- Target endianness and the numeric significance of the temporary portion selected at byte offset 2.
- Exact machine instructions, load counts, access atomicity, and concurrency semantics.
- Permitted aliasing and dependency-induced invalidation.
- Behavior for invalid pointers, address overflow, or wider-pointer environments.

An inferred intent is to prepare a value, place derived portions into destination storage, and perform a post-write operation, with a separate path for unsuitable record metadata. That interpretation does not establish dependency semantics or a byte-order-independent full-value transfer.

## Evidence

The supplied static representation and opaque dependency mapping establish the argument uses, 32-bit temporary initialization, dependency call order, short-circuit predicate, unsigned threshold, and two ordered 16-bit destination writes.

The evidence distinguishes numeric narrowing for the first write from representation-based selection at temporary byte offset 2 for the second. It does not independently establish target endianness.

No dependency bodies, surrounding caller contracts, runtime traces, or execution results were supplied. No runtime tests have been performed. The acceptance examples are conditional deductions from static evidence, not observed outcomes.
