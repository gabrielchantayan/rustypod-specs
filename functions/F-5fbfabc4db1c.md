<!-- Trusted provenance: F-5fbfabc4db1c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5fbfabc4db1c

## Purpose

F-5fbfabc4db1c conditionally expands storage associated with a caller-supplied state object. It evaluates two independently tracked capacities against requirements formed from signed 16-bit state fields and two signed 32-bit input values.

The first capacity governs two unconditional storage operations and one flag-controlled operation. The second capacity governs one storage operation. Successful expansion rounds the first target to a multiple of eight and the second target to a multiple of four. If either capacity is successfully expanded, the function invokes F-eafa9af4344c after completing the required storage operations.

The function is not transactional. A dependency result is written into the state before its status is checked. Failure can therefore leave replacement storage values installed while the corresponding capacity remains unchanged.

The allocation, element-count, and refresh interpretations are inferred intent. The established behavior is the sequence of arithmetic, comparisons, dependency calls, state writes, status checks, and return-value construction described below.

## Interface

### Inputs

The function receives three arguments:

1. **State address:** an address used to read and modify a structured memory region.
2. **First adjustment:** a signed 32-bit value contributing to the first capacity requirement.
3. **Second adjustment:** a signed 32-bit value contributing to the second capacity requirement.

The visible implementation uses 32-bit address arithmetic and 32-bit storage words. The following byte offsets are relative to the state address:

| Offset | Access width | Observed role |
|---|---:|---|
| `0x00` | 32 bits | Opaque context value supplied to F-6dd003795db7 |
| `0x04` | 32 bits | First capacity, compared as unsigned |
| `0x08` | 32 bits | Second capacity, compared as unsigned |
| `0x10` | 8 bits | Enables the optional first-group operation when nonzero |
| `0x14` | 16 bits, signed | First contribution to the second requirement |
| `0x16` | 16 bits, signed | First contribution to the first requirement |
| `0x18` | 32 bits | First storage value in the first group |
| `0x1c` | 32 bits | Second storage value in the first group |
| `0x20` | 32 bits | Storage value for the second group |
| `0x28` | 32 bits | Optional storage value in the first group |
| `0x34` | 16 bits, signed | Second contribution to the second requirement |
| `0x36` | 16 bits, signed | Second contribution to the first requirement |

These names describe observed uses, not a recovered public structure definition. In particular, the 32-bit storage values are opaque. Their use as allocation addresses is plausible but not established independently.

### Preconditions

The state address must support all reads and writes reached by the selected execution path. A contiguous readable region through offset `0x37` covers the directly observed fields, but dependencies may require additional memory or invariants.

No null check, object-size check, capacity-consistency check, or validation of the adjustments is visible. Invalid addresses or incompatible state contents are outside the established safe-use contract.

No ownership transfer can be inferred solely from passing a storage value to F-6dd003795db7. Callers must obtain ownership and lifetime requirements from that dependency’s contract.

### Output

The visible return construction is 64 bits:

- Bits 0–31 contain the final 32-bit local status word.
- Bits 32–63 contain a retained 32-bit value described below.

The status begins at zero and is passed by address to every F-6dd003795db7 invocation. It is not explicitly reset between calls. A nonzero status after any such call stops further normal processing.

The upper return word is not established as a second status:

- If no storage operation is attempted, it is the original first adjustment’s bit pattern.
- Otherwise, it is the storage value most recently read from the state for an attempted operation.

Thus, the upper word can represent a signed input’s bit pattern or an opaque storage value. Treating the entire 64-bit result as a conventional scalar error code would require an external interface contract not present here.

## Behavior

### 1. Initial capture and first requirement

The function first captures the 32-bit context from offset `0x00`, initializes its local status to zero, and reads the first capacity from offset `0x04`.

It forms the first requirement from:

- The signed 16-bit field at `0x16`;
- The signed 16-bit field at `0x36`;
- The first adjustment.

The 16-bit fields are sign-extended before addition. The resulting 32-bit value is used as an unsigned quantity for comparison with the first capacity.

The visible expression does not establish the relative evaluation order of the two signed-field reads. Both contribute to the requirement before any first-group dependency call.

If the first capacity is at least the unsigned requirement, the entire first-group expansion sequence is skipped. The optional flag and first-group storage values need not be read on that path.

If the first capacity is smaller, the function computes a rounded target by adding seven to the requirement and clearing the lowest three bits.

### 2. First-group expansion

When first-group expansion is required, operations occur in this order:

1. Read the storage value at `0x18`.
2. Invoke F-6dd003795db7 with:
   - The initially captured context;
   - Numeric argument `8`;
   - The initially read first capacity;
   - The rounded first target;
   - The storage value just read;
   - The address of the local status word.
3. Write the dependency’s returned 32-bit value to `0x18`.
4. Check the local status. If nonzero, return immediately.
5. Read the storage value at `0x1c`.
6. Invoke F-6dd003795db7 with the same captured context, old capacity, and target, but numeric argument `1` and this second storage value.
7. Write the returned value to `0x1c`.
8. Check the status. If nonzero, return immediately.
9. Read the byte at `0x10`.
10. If that byte is nonzero:
    - Read the storage value at `0x28`;
    - Invoke F-6dd003795db7 using numeric argument `8`, the same captured context, old first capacity, and target;
    - Write the returned value to `0x28`;
    - Check the status and return immediately if it is nonzero.
11. Record that first-group expansion completed.
12. Write the rounded target to the first-capacity field at `0x04`.

Every dependency result is stored before the corresponding status check. A failed operation’s returned value is therefore installed just as a successful operation’s result would be.

The first-capacity field is committed only after every required first-group operation succeeds. Failure during the second or optional third operation does not undo earlier storage writes.

The optional flag is read after the first two operations, not captured at entry. A dependency-induced change to that byte can therefore affect whether the optional operation occurs.

### 3. Second requirement and expansion

This phase is reached only if all attempted first-group operations succeeded.

The function computes the second requirement from:

- The signed 16-bit field at `0x14`;
- The signed 16-bit field at `0x34`;
- The second adjustment.

Again, the signed fields are sign-extended, and the 32-bit result is compared as unsigned against the capacity at `0x08`.

These requirement fields are read after the first-group processing. They are not entry snapshots.

If the second capacity is smaller than the unsigned requirement:

1. Read the storage value at `0x20`.
2. Compute the second target by adding three to the requirement and clearing the lowest two bits.
3. Invoke F-6dd003795db7 with:
   - The context captured at function entry;
   - Numeric argument `2`;
   - A read of the second-capacity field at `0x08`;
   - The rounded second target;
   - The storage value read from `0x20`;
   - The same local status address used by earlier operations.
4. Write the returned value to `0x20`.
5. Check the status. If nonzero, return immediately.
6. Write the rounded target to `0x08`.

The second capacity is read for the comparison and is also used as a call argument. The visible implementation does not provide a concurrency or synchronization contract that would make these reads atomic as a group.

A failure here does not undo a completed first-group expansion.

### 4. Final dependent action

F-eafa9af4344c receives the state address exactly once if normal processing reaches the end and either:

- First-group expansion completed; or
- Second-group expansion completed.

It is not called if neither capacity required expansion. It is also not called after any detected nonzero status, even when earlier operations already changed the state.

Its return value, if any, is not used by the visible function. Its effects on the state or other memory are unknown.

### 5. Arithmetic boundaries

Capacity comparisons are unsigned and strict. Equality does not trigger expansion.

The rounding operations are 32-bit mask operations:

- First target: add seven, then clear bits 0–2.
- Second target: add three, then clear bits 0–1.

They are not saturating operations. Under 32-bit modular arithmetic, sufficiently large requirements can produce a rounded target of zero. No check rejects such a target or verifies that it exceeds the old capacity.

Negative requirement sums are not clamped to zero. Their 32-bit representations can compare as very large unsigned requirements.

The supplied representation uses signed additions before conversion to an unsigned requirement. Exact signed-overflow semantics are not independently established by that representation. Applications requiring defined behavior should constrain inputs so both signed requirement sums fit within signed 32-bit range, unless separate evidence establishes the underlying machine arithmetic.

## Side effects

### Direct state writes

The function can directly write:

- Storage values at `0x18`, `0x1c`, `0x28`, and `0x20`;
- First capacity at `0x04`;
- Second capacity at `0x08`.

It does not directly clear, initialize, or copy the storage represented by those values.

Capacity writes are success commits for their respective groups. Storage writes precede success determination and are not rolled back.

### Dependency-mediated effects

F-6dd003795db7 can modify the local status through its supplied address. Its other effects are unknown. In particular, this contract does not assume it preserves the state object, preserves globals, frees old storage, retains old storage, or returns valid replacement storage on failure.

F-eafa9af4344c receives the state address and can have effects not visible in this body. No preservation guarantees are available.

The context value is captured once and reused. A dependency changing the context field does not cause later calls in this invocation to reload it. Other fields, including later storage values and second-group requirement fields, are read when their phase is reached.

There is no visible synchronization. Concurrent mutation and reentrant access require an external contract.

## Dependencies

### F-6dd003795db7

Observed interface: six arguments and a 32-bit result.

| Argument | Observed content |
|---|---|
| 1 | Captured 32-bit context |
| 2 | Numeric value `8`, `1`, or `2` |
| 3 | Old capacity for the relevant group |
| 4 | Rounded requested capacity |
| 5 | Existing opaque storage value |
| 6 | Address of a writable 32-bit status word |

The result replaces the relevant storage value unconditionally.

**Inferred intent:** resize storage with element widths of eight, one, or two bytes.

**Unknown:** whether the numeric argument actually denotes element width; allocation and release rules; content preservation; initialization of added storage; status meanings; failure-result validity; and effects on shared state.

### F-eafa9af4344c

Observed interface: the state address is passed after successful expansion processing.

**Inferred intent:** reconcile or refresh state derived from the changed capacities or storage.

**Unknown:** exact updates, global effects, failure behavior, ownership effects, and whether its changes can affect values subsequently observed by callers.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, assume dependencies do not modify unrelated fields, each successful storage operation leaves status zero, and arithmetic is within signed 32-bit range.

1. **Both capacities sufficient.**  
   First capacity is 16 and the first signed-field contributions are 2 and 3, with first adjustment 11. Second capacity is 12 and its contributions are 1 and 2, with second adjustment 9. Both requirements equal their capacities. No dependency is called. The low return word is zero; the upper word is 11.

2. **Only the first group grows.**  
   First capacity is 8 and its requirement is 9. The target is 16. With the optional flag zero, F-6dd003795db7 is called first with numeric argument `8`, then with `1`, both using old capacity 8 and target 16. Their results are stored at `0x18` and `0x1c`. The first capacity becomes 16. If the second capacity is sufficient, F-eafa9af4344c is still called once.

3. **Optional operation enabled.**  
   First-group expansion is needed and the flag read after the second successful operation is nonzero. A third operation occurs with numeric argument `8`. Only after its zero-status result is installed at `0x28` is the first capacity updated.

4. **First-group partial failure.**  
   The first operation succeeds and the second leaves status 7. Both returned storage values have been installed. The first capacity is not committed, the optional flag is not read by the subsequent branch, the second requirement is not evaluated, and F-eafa9af4344c is not called. The low return word is 7. The upper word is the storage value read from `0x1c` before the failed operation.

5. **Second-group failure after first-group success.**  
   The first group completes and commits its capacity. The second operation leaves a nonzero status. Its returned value is installed at `0x20`, but the second capacity is not committed. The first group remains changed, and F-eafa9af4344c is skipped.

6. **Only the second group grows.**  
   The first capacity is sufficient. Second capacity is 4 and its requirement is 5, producing target 8. One storage operation occurs with numeric argument `2`. On zero status, the second capacity becomes 8 and F-eafa9af4344c is called once.

7. **Negative requirement sum.**  
   First contributions and adjustment sum to −1 without signed overflow. Its unsigned representation is `0xffffffff`. A first capacity of zero therefore triggers expansion. The observed 32-bit add-and-mask rounding produces target zero, rather than rejecting the request or selecting a saturated capacity.

## Uncertainties

The following are **documented_only**:

- A public structure definition and externally supported calling convention.
- Whether callers consume both return words or only the low status word.
- The meaning of individual nonzero status values.
- Allocation ownership, release behavior, and storage validity following failure.
- Dependency effects on globals, aliased state, or retained addresses.
- Exact signed-overflow behavior beyond the supplied representation.
- Requirements governing concurrency, callbacks, and reentrancy.
- Memory byte order and any stronger alignment requirements.
- The semantic purpose of the signed fields and optional flag.

The storage-growth and final-refresh interpretations are inferred intent, not additional acceptance requirements.

## Evidence

Established static evidence comes solely from the supplied function representation: access widths and offsets, signed extension of 16-bit fields, unsigned capacity comparisons, rounding masks, dependency argument values, result-write ordering, status-controlled exits, capacity commits, final-call gating, and the two-word return construction.

No dependency implementation, public interface declaration, or runtime observation was supplied. No runtime tests have been performed.
