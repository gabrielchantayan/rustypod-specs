<!-- Trusted provenance: F-aa02b30efe51; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aa02b30efe51

## Purpose

F-aa02b30efe51 selects a non-null entry from an indexed object table, passes that entry and an opaque value to a dependency, and then performs conditional release-like processing on synchronization-related state embedded in the object.

The established behavior is narrower than a complete synchronization API:

- An entry dependency runs before the index is validated.
- Indices outside 0–28, and null entries within that range, lead to a designated non-returning dependency.
- After the selected-entry dependency returns, the function may return an early status, reject an ownership comparison, decrement a 16-bit field, or clear ownership-related fields and invoke a final dependency.

“Owner,” “recursion count,” and “release” are inferred interpretations of the field usage. They are not established meanings supplied by the dependencies.

## Interface

### Inputs

The function has three inputs:

| Input | Representation | Observable use |
|---|---|---|
| Object base | 32-bit address-like value | Locates an embedded state region and an indexed table. |
| Entry index | Unsigned 32-bit integer | Selects one of 29 permitted table positions. |
| Opaque argument | 32-bit value | Forwarded unchanged to F-925357ba24ce. |

No length, allocation descriptor, or explicit ownership token is supplied.

### Return value

The function returns a 32-bit integer if execution reaches a returning path:

| Condition | Result |
|---|---|
| Embedded state address is zero after the selected-entry call | 26 |
| Special-marker handling returns nonzero | That dependency’s result, unchanged |
| Stored identity differs from the identity dependency’s result | 5 |
| Stored identity matches, and the decrement-only path leaves a nonzero 16-bit result | 0 |
| Stored identity matches, and the clearing path runs | Result of F-259d2fe281b0 |

The invalid-index and null-entry paths call F-a817d8d58c10, which is designated as non-returning. No normal return value is established for those paths.

### Object-relative layout

The supplied representation establishes the following accesses. Offsets are bytes from the object base.

| Offset | Width | Access and role |
|---|---:|---|
| `0x114` | 32 bits | First word of the embedded state region; compared with a global marker value. The address of this region is passed to dependencies. |
| `0x118` | 32 bits | Stored identity; read for comparison and conditionally cleared. |
| `0x122` | 16 bits | Flags snapshot used to select decrement behavior. |
| `0x126` | 16 bits | Conditionally decremented and written back, or cleared. |
| `0x128` | Address only in this function | Passed to F-259d2fe281b0. The size and layout of the referenced region are unknown. |
| `0x154 + 8 × index` | 32 bits | Selected table entry, required to be nonzero. |

The table has an eight-byte stride, but this function directly reads only the first four bytes of each selected slot. It does not establish the purpose of the remaining four bytes.

The highest permitted entry starts at offset `0x234`; its four-byte read extends through offset `0x237`. This is a minimum extent for that table access, not a complete object-size requirement. Dependencies may access additional memory.

### Preconditions

For defined completion, every reached direct access must address valid storage with the access properties required by the execution environment. Dependencies may impose additional, undocumented preconditions.

A null object base is not explicitly rejected. The later test concerns the derived address at offset `0x114`, and it occurs only after earlier dependency calls and a successful table-entry read. It is not a general null-object guard.

## Behavior

### 1. Invoke the entry dependency

The function derives the address at object offset `0x114` and passes it to an entry dependency whose opaque function ID is unavailable in the supplied dependency map.

This invocation precedes both index-range validation and the table-entry read. Consequently, an invalid index does not prevent the entry dependency from running.

Its return value is not used. Its effects are unknown: the supplied material does not establish whether it changes the object, global state, ownership, or memory lifetime.

### 2. Validate and select the table entry

After the entry dependency returns, the function tests the unsigned index.

- If the index exceeds 28, it invokes F-a817d8d58c10 without reading the selected table entry.
- Otherwise, it reads the 32-bit entry at `0x154 + 8 × index`.
- If that entry is zero, it invokes F-a817d8d58c10.
- A nonzero entry proceeds to the selected-entry call.

Because the comparison is unsigned, all input bit patterns representing values from 29 through `0xffffffff` take the out-of-range path. They are not negative indices.

For accepted indices, multiplication by eight produces values from 0 through 224, so the index-scaling calculation does not overflow 32 bits. Validity of the subsequent object-relative address still depends on the base address and memory model.

### 3. Invoke the selected-entry dependency

F-925357ba24ce receives:

1. The nonzero 32-bit entry value just read.
2. The original opaque 32-bit argument.

Its return value is not used. The function does not directly write the table slot, copy an entry payload, or explicitly free the selected entry.

No preservation guarantee is established for this call. Later object and global reads occur after it returns and must not be replaced by assumptions about values observed before the call.

### 4. Test the derived state address

After F-925357ba24ce returns, the function checks whether the previously derived address at offset `0x114` is zero.

If it is zero, the function returns 26. It does not perform the later marker comparison, flags read, identity comparison, or final dependency call.

This is an address test, not a test of the 32-bit contents at offset `0x114`.

Whether a valid execution can reach this branch is unresolved. Under modular 32-bit address arithmetic, a particular nonzero base could produce a zero derived address. Under stricter pointer-validity requirements, the earlier accesses or dependency call may already be invalid. The branch is statically present; its safe reachability is not established.

### 5. Apply special-marker handling

For a nonzero embedded state address, the function compares its first 32-bit word with a global marker value.

If they differ, F-621dae46a146 is not called.

If they match, F-621dae46a146 receives the embedded state address and a zero second argument:

- A nonzero result is returned immediately and unchanged.
- A zero result allows processing to continue.

The relative evaluation order of the two comparison operands is not established by the supplied representation. The marker comparison as a whole occurs after the selected-entry dependency and before the flags snapshot.

### 6. Snapshot flags and obtain an identity

The function reads the 16-bit field at offset `0x122` and retains that value for the later mode decision.

It then calls F-196812b4b122 with no explicit arguments. After that call returns, it reads the 32-bit field at offset `0x118` and compares it with the returned value.

This ordering is significant:

- The flags decision uses a snapshot taken before F-196812b4b122.
- The stored identity is read after F-196812b4b122.
- If that dependency changes either field, the two decisions can reflect different moments.

If the stored identity differs from the returned identity, the function returns 5. On this branch, it performs no direct count decrement, count clear, identity clear, or call to F-259d2fe281b0.

### 7. Choose decrement-only or clearing behavior

If the identities match, the flags snapshot selects the next action.

The function masks the flags with `0x003f`, shifts the result right by four bits, and compares the result with 2. Equivalently, the selected mode requires bits 5–4 to be binary `10`. Bits 3–0 and bits 15–6 do not affect this decision.

If the mode is not selected, the function skips the decrement and proceeds directly to clearing.

If the mode is selected:

1. It reads the 16-bit field at offset `0x126`.
2. It subtracts one and stores the result back into that 16-bit field.
3. It tests the stored 16-bit result for zero.

A nonzero result causes return value 0, without clearing the identity and without calling F-259d2fe281b0.

A zero result proceeds to clearing.

For the 16-bit storage behavior represented here, subtraction retains the low 16 bits. An initial value of zero therefore becomes `0xffff`, which is nonzero. There is no visible underflow rejection or saturation. Only an initial bit pattern of `0x0001` decrements to zero.

### 8. Clear fields and invoke the final dependency

The clearing path performs these operations in order:

1. Store zero into the 16-bit field at offset `0x126`.
2. Store zero into the 32-bit field at offset `0x118`.
3. Call F-259d2fe281b0 with the address at offset `0x128`.
4. Return that dependency’s result.

When the decrement has already produced zero, the explicit zero store still follows.

There is no visible rollback if F-259d2fe281b0 returns an error-like result. Both fields have already been cleared before that result becomes available.

## Side effects

### Direct writes

This function directly writes only:

- The 16-bit field at `0x126`, on matching-identity decrement or clearing paths.
- The 32-bit field at `0x118`, on the clearing path.

It does not directly modify the flags field, table entry, first word of the embedded state region, or global marker.

### Dependency effects and ownership

All dependency effects remain separate from those direct writes. The supplied material does not establish that any dependency:

- Preserves object fields or globals.
- Acquires or releases a synchronization primitive.
- Retains or releases a referenced allocation.
- Preserves the lifetime of the object or selected entry.
- Returns normally in all reachable circumstances.

Passing an address or entry value is not, by itself, evidence of ownership transfer. No explicit allocation or deallocation is visible in this function.

### Concurrency and atomicity

No atomic-access, memory-ordering, or indivisibility guarantee is established.

In particular, the count and identity clears are separate writes. The function does not provide evidence that another observer cannot see intermediate state. Any exclusion supplied by the entry dependency is undocumented here.

## Dependencies

| Opaque function ID | Invocation | Established use |
|---|---|---|
| Unavailable in supplied map | Embedded state address | Runs first; return value unused. |
| F-a817d8d58c10 | No explicit arguments | Designated non-returning target for invalid index or zero selected entry. |
| F-925357ba24ce | Selected entry, opaque argument | Runs after successful selection; return value unused. |
| F-621dae46a146 | Embedded state address, zero | Runs only on marker equality; nonzero result propagates immediately. |
| F-196812b4b122 | No explicit arguments | Supplies the value compared with the stored identity. |
| F-259d2fe281b0 | Address at object offset `0x128` | Runs after both clearing writes; result becomes the function result. |

The global marker is an external data dependency. Its value, mutability, initialization, and meaning are not supplied.

## Acceptance examples

These are conditional static acceptance cases, not runtime test results. Unless stated otherwise, assume all reached accesses are valid, invoked dependencies return normally, and they do not introduce changes beyond those specified in the example.

1. **Out-of-range index**

   With index 29, the entry dependency runs first, followed by F-a817d8d58c10. No selected-slot read or F-925357ba24ce call occurs.

2. **Null entry at the upper boundary**

   With index 28 and a zero word at offset `0x234`, the entry dependency runs, that word is read, and F-a817d8d58c10 is invoked.

3. **Selected-entry forwarding**

   With index 0 and a nonzero entry at `0x154`, F-925357ba24ce receives that entry and the original opaque argument. This call occurs before marker handling or identity processing.

4. **Special-marker early result**

   With matching marker values and F-621dae46a146 returning 17, the function returns 17. It does not read the flags for the later mode decision or call either identity or final dependency.

5. **Identity mismatch**

   If marker handling permits continuation and the stored identity differs from F-196812b4b122’s result, the function returns 5 without directly changing the count or identity fields.

6. **Non-final decrement**

   With matching identities, flags `0x0020`, and count 2, the count becomes 1 and the function returns 0. The stored identity is not directly cleared; F-259d2fe281b0 is not called.

7. **Final decrement**

   Under the same conditions but count 1, the decrement stores zero. The clearing path then stores zero to the count, clears the identity, and calls F-259d2fe281b0. Its result is returned unchanged.

8. **Zero-count boundary**

   With matching identities, selected mode, and count zero, the count becomes `0xffff`. The function returns 0 without the clearing path.

9. **Mode bypass**

   With matching identities and flags `0x0010`, the count is cleared without first being decremented. The identity is then cleared and F-259d2fe281b0 is invoked.

10. **Flags changed by the identity dependency**

    If the flags snapshot selects decrement behavior, but F-196812b4b122 subsequently changes the stored flags to another mode, the original snapshot still controls the branch. The identity comparison uses the field read after that dependency returns.

11. **Final dependency reports failure**

    If the clearing path reaches F-259d2fe281b0 and it returns a negative or other error-like value, that value is returned unchanged. The function performs no visible restoration of the cleared fields.

## Uncertainties

The following remain **documented_only**:

- The entry dependency’s opaque ID and behavioral contract.
- The meanings of the global marker, flags mode, stored identity, and 16-bit field.
- Whether return values 5 and 26 correspond to named errors.
- Whether F-196812b4b122 returns a thread identifier or another identity.
- Whether dependencies preserve memory, globals, ownership, or object lifetime.
- The exact mechanism by which F-a817d8d58c10 does not return.
- The safe reachability of the zero-derived-address branch.
- Target alignment, byte order, and address-validity rules beyond the represented widths.
- Synchronization, atomicity, and behavior under concurrent mutation.

A recursive ownership-release interpretation is plausible from the identity comparison, mode-controlled decrement, and clearing sequence. It remains inferred intent, not an additional guarantee.

## Evidence

Established static evidence consists of the supplied function representation and dependency map: access widths and offsets, branch conditions, dependency arguments, direct write ordering, and return propagation.

The representation supports 32-bit address-like and integer values, unsigned 32-bit index comparison, and 16-bit flags/count accesses. It does not independently establish complete dependency contracts or the original source-language semantics of every recovered operation.

No runtime tests have been performed. The acceptance examples describe consequences of the visible control flow under their stated conditions, not observed executions.
