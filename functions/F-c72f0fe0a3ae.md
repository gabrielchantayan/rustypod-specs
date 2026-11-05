<!-- Trusted provenance: F-c72f0fe0a3ae; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c72f0fe0a3ae

## Purpose

F-c72f0fe0a3ae conditionally advances two 32-bit accumulators in a caller-supplied record, replaces a stored reference value, decreases a positive signed counter, and applies unsigned upper bounds to the accumulators.

The entire record-update path is gated by a byte obtained through F-18bc752d66cd. When that byte is zero, the function performs no explicit record access or upper-bound access.

The apparent intent is to maintain bounded progress or timing state from a newly obtained sample. That interpretation is inferred, not established: neither the units of the sample nor the transformation performed by F-d7be9cfb4000 is specified by the available evidence.

## Interface

### Inputs

The function accepts four arguments:

| Argument | Representation | Established use |
|---|---|---|
| Record address | Address carried in a 32-bit argument | Base address for the fields described below, accessed only on the enabled path |
| Initial sample word | 32-bit value, presented as signed | Copied into temporary storage; its resulting value is later used as the replacement reference |
| Auxiliary word A | Opaque 32-bit value | Copied into temporary storage, then its temporary slot is overwritten with the sample difference after F-c082dad756a2 returns |
| Auxiliary word B | Opaque 32-bit value | Copied into temporary storage; no later direct use is shown |

Before calling any dependency, the function makes local copies of the last three arguments. F-c082dad756a2 receives the address of the initial-sample copy, not the record address and not the address of the caller’s original argument storage.

The evidence shows three 32-bit temporaries, but does not establish a portable, externally valid 12-byte structure or a dependency contract permitting access beyond the addressed word. In particular, the auxiliary words must not be assigned meanings solely from their placement near the sample copy.

### Record layout

All offsets are byte offsets from the supplied record address.

| Offset | Width | Interpretation in this function |
|---|---:|---|
| `+0x04` | 32 bits | Stored reference, read for subtraction and subsequently replaced |
| `+0x08` | 32 bits | First accumulator; updated arithmetically, then compared and possibly replaced as unsigned |
| `+0x0c` | 32 bits | Second accumulator; updated arithmetically, then compared and possibly replaced as unsigned |
| `+0x20` | 32 bits | Signed counter; decreased by 120 only when strictly positive |

No direct access to the record’s offset-zero word or intervening fields is shown. The listed offsets do not establish the complete record size or ownership model.

### Output

There is no explicit return value. Results are conveyed through record writes and any effects of invoked dependencies.

### Preconditions

On an invocation whose gate byte is nonzero:

- The record address must support the listed 32-bit reads and writes.
- F-18bc752d66cd must return an address from which the byte at offset `+0x4c` can be read.
- F-c082dad756a2 must return in a manner that leaves the temporary sample usable by this function.
- Arithmetic must have defined behavior under the actual target’s execution rules.

The gate-result address must be readable even when the gate is zero. There is no null-address check before that byte access.

## Behavior

### 1. Gate evaluation

After preparing the temporary argument copies, the function invokes F-18bc752d66cd with no explicit arguments. It reads one byte at offset `+0x4c` from the returned address.

A zero byte terminates the function’s explicit work. A nonzero byte enables all subsequent steps. The test is for zero versus nonzero; it does not require the byte to equal one.

The gate is read once in the supplied body. The function does not recheck it after later dependencies execute.

### 2. Sample acquisition and difference

On the enabled path, the function invokes F-c082dad756a2 with the address of its temporary sample word.

After that dependency returns, the function subtracts the record’s current `+0x04` value from the temporary sample value. The difference occupies a 32-bit signed temporary, replacing the temporary copy of auxiliary word A.

This ordering matters:

- The reference is read after F-c082dad756a2 returns.
- The sample used is the temporary’s value after that call, whether or not it differs from the input.
- Auxiliary word A has no further independent direct use after its temporary slot is replaced.
- No assumption is justified that F-c082dad756a2 preserves globals or other reachable memory.

The function then invokes F-d7be9cfb4000 with the difference as its first argument and the integer constant **1000** as its second argument.

The result is used as a signed 32-bit increment. The constant does not, by itself, establish division, scaling, rounding, saturation, or time units.

### 3. Accumulator updates

After F-d7be9cfb4000 returns, the function updates the first accumulator at `+0x08` by adding the returned increment to that field’s then-current value.

It next updates the second accumulator at `+0x0c` in the same way, using the same returned increment.

These are separate, ordered updates, not an atomic paired operation. The first accumulator’s write precedes the second accumulator’s update.

The initial accumulator values are not saved before either dependency call. Consequently, changes made by dependencies to the record’s reachable storage can affect the values subsequently updated.

### 4. Reference replacement

After both accumulator additions, the function stores the temporary sample word into the record at `+0x04`.

Thus, the replacement reference is not necessarily the original second argument: F-c082dad756a2 has been given an address through which it may change that temporary.

This write occurs after F-d7be9cfb4000 returns. The available body does not prove that all dependencies leave the temporary sample unchanged except for an intended sample update.

### 5. Signed counter decrease

The function reads the counter at `+0x20` as signed.

- If it is zero or negative, there is no explicit counter write.
- If it is strictly positive, the function subtracts **120** and writes the result back.

This is not saturation at zero. An initial value between 1 and 119 becomes negative. A value of 120 becomes zero.

For an unchanged, valid positive signed 32-bit input, this subtraction itself cannot overflow the signed 32-bit range.

### 6. Unsigned accumulator limits

After the counter step, the function uses a shared 32-bit upper-bound value to limit the accumulators.

The supplied body represents the following distinct uses:

1. Capture the upper bound into a local unsigned 32-bit snapshot.
2. Compare an upper-bound read against the first accumulator, interpreted as unsigned.
3. If the upper bound is strictly less, assign an upper-bound read to the first accumulator.
4. Compare the saved snapshot against the second accumulator, interpreted as unsigned.
5. If the snapshot is strictly less, assign that snapshot to the second accumulator.

The condition is strict. Equality causes no limiting write.

If the upper bound stays unchanged throughout this sequence, both accumulators are limited to the same unsigned maximum. However, the body does not give both comparisons a guaranteed common snapshot: the second accumulator uses the saved value, whereas the first comparison and replacement refer directly to the shared bound.

These are logical accesses visible in the supplied representation, not a guarantee of distinct physical bus transactions. Volatility, compiler transformations, synchronization, and concurrent mutation rules are not supplied.

## Side effects

### Explicit writes

On the enabled path, the function explicitly writes:

- The first accumulator after addition.
- The second accumulator after addition.
- The stored reference.
- The counter, only if positive.
- Either accumulator again, only if its unsigned limiting condition is satisfied.

The direct sequence is therefore: first addition, second addition, reference replacement, optional counter decrease, optional first limit, optional second limit.

The temporary storage is also initialized and modified during execution. A pointer to the temporary sample is exposed to F-c082dad756a2.

### Dependency effects and ownership

No allocation, deallocation, ownership transfer, or resource-release operation is explicit in this body.

That absence does not establish that dependencies allocate nothing, release nothing, or preserve global state. Their effects remain subject to their own undocumented contracts.

The sample pointer refers to function-local storage. The caller receives no persistent sample buffer from this function. Whether F-c082dad756a2 retains that address is unknown; retaining it would require a lifetime contract not established here.

### Errors and partial updates

The function has no explicit error return, validation branch, rollback, or exception recovery.

An invalid gate-result address can fail before any record access. On the enabled path, invalid record storage or dependency failure can prevent completion.

The updates are incremental. If execution stops after the first accumulator write, the record may already be partially changed. No transactional guarantee is established.

## Dependencies

| Opaque function ID | Invocation | Established contribution |
|---|---|---|
| F-18bc752d66cd | No explicit arguments | Returns an address whose byte at `+0x4c` determines whether updates proceed |
| F-c082dad756a2 | Address of the local sample word | Receives access to that temporary before the difference is computed |
| F-d7be9cfb4000 | Signed difference and constant 1000 | Returns the increment applied to both accumulators |

On an ordinarily completed enabled invocation, the call order is F-18bc752d66cd, F-c082dad756a2, then F-d7be9cfb4000, once each. On the disabled path, only F-18bc752d66cd is called directly.

No meaning, purity, memory-preservation guarantee, or failure convention is inferred from these IDs.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not executed tests. Except where stated otherwise, they assume valid storage, ordinary dependency returns, no intervening mutation, and arithmetic within signed 32-bit range.

### Disabled gate

If F-18bc752d66cd returns a readable object whose `+0x4c` byte is zero:

- Neither later dependency is called directly.
- No record field is explicitly read or written.
- The shared upper bound is not explicitly accessed.

This does not require F-18bc752d66cd itself to be side-effect-free.

### Ordinary enabled update

Suppose:

- The gate is nonzero.
- The temporary sample after F-c082dad756a2 returns is 1500.
- The stored reference is 1200.
- F-d7be9cfb4000 returns 7 for arguments 300 and 1000.
- The accumulators are 20 and 40.
- The counter is 121.
- The upper bound remains 100.

The final directly maintained fields are:

| Field | Final value |
|---|---:|
| Reference | 1500 |
| First accumulator | 27 |
| Second accumulator | 47 |
| Counter | 1 |

The increment of 7 is a stipulated dependency result, not a derived interpretation of its arguments.

### Counter crossing zero

With a positive counter of 1, the counter write produces −119. With 119 it produces −1; with 120 it produces 0. A counter initially equal to zero or −1 receives no explicit counter write.

### Limit equality and excess

If the post-addition accumulators are 100 and 101 and the shared bound remains 100:

- The first accumulator receives no limiting write.
- The second accumulator is replaced with 100.

The first field has still received its earlier addition write.

### Negative value under an unsigned limit

Suppose the first accumulator is 2, the dependency increment is −3, and the bound remains 100.

The addition produces signed −1 without signed overflow. At the limiting comparison, its 32-bit representation is interpreted as unsigned 4,294,967,295. The first accumulator is therefore replaced with 100, not left at −1 and not clamped to zero.

### Dependency mutation of the reference

If F-c082dad756a2 changes the record’s stored reference through some independently reachable alias, the difference uses that changed reference. The function does not preserve a pre-call reference snapshot.

## Uncertainties

The following are **documented_only**:

- **Arithmetic overflow:** The represented subtraction and accumulator additions use signed 32-bit values. The supplied body does not independently establish defined wraparound, trapping, or other overflow behavior. Results outside the signed range must not be specified as modulo arithmetic without additional target evidence.
- **Dependency mathematics:** The transformation applied by F-d7be9cfb4000 is unknown, including treatment of negative inputs and rounding.
- **Temporary layout:** A pointer to one local word does not establish a public multiword input structure or authorized extent of dependency access.
- **Shared-bound consistency:** Atomicity, volatility, concurrency, and synchronization are unknown. A single-bound outcome is conditional on a stable bound.
- **Aliasing:** No non-aliasing contract is provided for the record, dependency-reachable storage, gate storage, or shared bound. Aliasing can make earlier writes affect later reads.
- **Gate meaning:** Nonzero enables updates, but the domain meaning of the byte is unknown.
- **Failure behavior:** Dependency errors, non-returning calls, faults, and exceptional exits are not specified.
- **Ownership:** Dependency allocation, release, retention, and lifetime requirements are unknown.

## Evidence

The established behavior comes solely from the supplied function body: the gate-byte access, conditional dependency calls, field offsets, 32-bit operations, constants 1000 and 120, signed positivity check, unsigned limiting comparisons, and explicit write order.

“Sample,” “reference,” “accumulator,” “counter,” and “upper bound” are descriptive roles assigned from those operations, not recovered domain definitions. Timing or progress semantics are inferred intent only.

No implementation beyond the supplied body was inspected. No runtime tests have been performed.
