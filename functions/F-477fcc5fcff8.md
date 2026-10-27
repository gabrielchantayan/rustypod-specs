<!-- Trusted provenance: F-477fcc5fcff8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-477fcc5fcff8

## Purpose

F-477fcc5fcff8 performs an ordered initialization sequence using addresses supplied by two dependencies. It obtains an initial address, copies a 32-bit global word to that address, and makes two further dependency calls to obtain the address used for four final stores.

The final initialization consists of **one 8-bit store and three 32-bit stores**. These stores address 13 bytes within a region extending through byte offset 23; they do not clear that entire region. The separate initial-address store writes another 4 bytes. Across all five direct stores, the write widths total **17 bytes**, but the destinations may overlap, so this is not necessarily 17 distinct bytes.

An object-initialization role is inferred intent only. The visible operations do not establish object types, allocation boundaries, field meanings, or resource-management responsibilities.

## Interface

### Inputs

The function accepts two 32-bit arguments.

| Argument | Established use |
|---|---|
| First argument | Not read or forwarded by the visible body. It does not select a branch, address calculation, or stored value. |
| Second argument | Copied as a complete 32-bit value to **final-address byte offset 16**. |

Final-address byte offset **20 receives a 32-bit zero**, not the second argument.

The argument representation does not establish signedness, enumeration membership, pointer meaning, or ownership. The second argument undergoes no visible validation, masking, or conversion before its store.

### Output

There is no explicit return value. The function does not return a destination address, completion flag, or error code.

Observable outputs consist of the direct memory accesses and dependency invocations described below, together with any effects performed by the dependencies themselves.

### Address terminology

This contract uses these descriptive labels:

- **Initial address:** the result of F-c5da7fd8e586.
- **First derived address:** the result of the first F-5b24edfb22c0 invocation.
- **Final address:** the result of the second F-5b24edfb22c0 invocation.
- **Global source word:** the 32-bit global storage read for the initial copy.

These labels identify roles in the sequence. They do not imply distinct allocations, object bases, or non-overlapping storage.

### Widths and arithmetic

All offsets below are measured in bytes.

The initial address is used as a pointer to 4-byte elements. Advancing it by one element therefore produces **initial address plus 4 bytes**, not plus 1 byte.

The two derived results are represented as integer address values in the supplied static body. Their relevant additions are byte displacements: 8 bytes for the second dependency argument, followed by offsets 8, 12, 16, and 20 for the final stores.

The supplied representation uses 32-bit words and 32-bit integer address intermediates. This does not independently establish a complete machine ABI or valid behavior for an overflowing address calculation. No wraparound rule, widening rule, or overflow recovery is specified.

## Behavior

### Ordered normal sequence

The following sequence applies when the dependencies return normally and every required memory access succeeds.

1. **Obtain the initial address.**  
   Invoke F-c5da7fd8e586 with no visible arguments and use its result as the initial address.

2. **Read and copy the global source word.**  
   After that invocation returns, read the 32-bit global source word. Store the read value as a 32-bit word at the initial address.

   The read occurs after F-c5da7fd8e586 returns. The copied value is therefore not necessarily the value that existed at function entry. The dependency may have changed the global word.

3. **Obtain the first derived address.**  
   Invoke F-5b24edfb22c0 with initial address plus 4 bytes.

   The initial 32-bit store precedes this call. The dependency receives control only after that store in the visible sequence.

4. **Obtain the final address.**  
   Invoke F-5b24edfb22c0 again, with first derived address plus 8 bytes.

   This argument is based on the first invocation’s returned address. No fixed relationship between the final address and the initial address is established.

5. **Perform the final stores.**  
   Use the second invocation’s returned value as the final address and perform these stores in order:

   | Store order | Offset from final address | Width | Stored value |
   |---|---:|---:|---|
   | 1 | 8 | 8 bits | Zero |
   | 2 | 12 | 32 bits | Zero |
   | 3 | 16 | 32 bits | Second argument |
   | 4 | 20 | 32 bits | Zero |

6. **Return without an explicit result.**

There are no further visible dependency calls after the final stores.

### Final-region layout

Under valid, non-wrapping address arithmetic, the final stores directly address these byte ranges:

- Byte 8.
- Bytes 12–15.
- Bytes 16–19.
- Bytes 20–23.

These ranges contain 13 distinct bytes. The highest addressed byte is offset 23, but that does not establish a 24-byte allocation or a requirement to initialize every preceding byte.

The final stores do not directly write:

- Bytes 0–7.
- Bytes 9–11.
- Bytes 24 and later.

In particular, the first final-region store clears only byte 8. Expanding it into a 32-bit clear would introduce writes to bytes 9–11 and would not preserve the established behavior.

The initial-address store is separate. It can affect otherwise untouched final-region bytes if the regions overlap. Dependencies can also affect those bytes. Consequently, “not directly written by the final stores” is not an unconditional preservation guarantee.

The second argument is stored as all 32 bits at offset 16. Its individual byte representation depends on the execution environment’s byte order, which is not established here.

### Aliasing and interference

The evidence does not establish that any returned address is disjoint from another returned address or from the global source storage.

The contract therefore specifies an ordered sequence of accesses rather than unconditional end-of-function values for every destination:

- Either F-5b24edfb22c0 invocation may change memory previously written by the caller.
- Any dependency may change global storage.
- The final stores may overlap the initial 32-bit destination.
- The initial destination may overlap the global source word.
- Returned addresses may be interior addresses rather than allocation bases.

For the initial copy, the global value is read before its corresponding destination store. If the source and destination overlap, the value used is the value obtained by that read; no second global read is visible.

When all accesses succeed, address arithmetic is valid, and there is no interfering modification, the final fields contain zero at offset 8, a zero word at offset 12, the second argument at offset 16, and a zero word at offset 20. The earlier initial-address value remains only where it is not subsequently overwritten.

### Preconditions and incomplete execution

Normal completion requires:

- Every dependency invocation to return normally.
- The global source storage to support the 32-bit read.
- The initial destination to support the 32-bit store when performed.
- The final destination to support each listed store when performed.
- Each dependency argument to satisfy that dependency’s own requirements.
- Address calculations and alignment to be valid for the actual execution environment.

The visible body does not check for null results, failure sentinels, inaccessible storage, insufficient extent, expired storage, unsupported alignment, or address overflow. A dependency result is used directly, not interpreted through an explicit success/failure branch.

No error return, rollback, retry, or cleanup path is visible. A dependency that does not return normally prevents continuation through the subsequent normal sequence. If an access fails, earlier successful effects are not visibly reversed. The evidence does not specify fault delivery, partial effects of a failing access, exception handling, or process termination.

## Side effects

### Direct effects

On successful normal completion, the visible body performs:

- One call to F-c5da7fd8e586.
- One 32-bit global read after that call.
- One 32-bit store at the initial address.
- Two ordered calls to F-5b24edfb22c0.
- One 8-bit store and three 32-bit stores relative to the final address.

The five direct stores have an aggregate width of 17 bytes. This accounting counts store widths, not unique destination bytes. The final four stores address 13 bytes; overlap with the initial store may reduce the total distinct storage written.

### Ownership and lifetime

No direct allocation or release operation is visible. The dependency calls do not, by themselves, establish that storage is allocated, retained, transferred, or released.

The function does not establish:

- Ownership transfer for either argument.
- Ownership of any returned address.
- Whether returned storage is new, borrowed, shared, or pre-existing.
- Whether dependencies preserve previously returned storage.
- A destruction operation or release obligation.
- Any lifetime extending beyond the accesses performed here.

Each directly accessed location must remain valid when its access occurs. A caller cannot infer that a dependency preserves earlier storage merely because that storage will be used later.

### Concurrency

No explicit synchronization, atomic operation, or memory-ordering primitive is visible.

The listed order describes the static single-invocation sequence. It does not establish atomicity of individual accesses, indivisibility of the initialization, inter-thread visibility, or a publication protocol. Concurrent modification and observation require additional environmental or dependency guarantees.

## Dependencies

| Opaque function ID | Invocation | Result consumed by this function |
|---|---|---|
| F-c5da7fd8e586 | Once, with no visible arguments | Initial address for the 32-bit global-word copy and the first subsequent argument calculation |
| F-5b24edfb22c0 | First invocation: initial address plus 4 bytes | First derived address used to calculate the next invocation’s argument |
| F-5b24edfb22c0 | Second invocation: first derived address plus 8 bytes | Final address used for the four final stores |

For both dependencies, internal reads, writes, allocation, release, validation, error conventions, and nonlocal control flow remain unknown.

Neither dependency is assumed to preserve globals, earlier destination contents, or storage lifetime. The two invocations of F-5b24edfb22c0 have the same opaque identity, but their different arguments and execution positions do not establish equivalent results or side effects.

## Acceptance examples

These examples are conditional criteria derived from static evidence. They are not executed tests.

### Ordinary successful initialization

Assume valid non-overlapping storage, valid address arithmetic, normally returning dependencies, and no interfering writes. Let the global source word at the actual read be 0x11223344 and the second argument be 0xA1B2C3D4.

Acceptance requires:

- A 32-bit copy of 0x11223344 to the initial address before the first F-5b24edfb22c0 invocation.
- The first invocation’s argument to be initial address plus 4 bytes.
- The second invocation’s argument to be first derived address plus 8 bytes.
- A zero byte at final offset 8.
- A zero 32-bit word at final offset 12.
- The complete 32-bit value 0xA1B2C3D4 at final offset 16.
- A zero 32-bit word at final offset 20.

Storing the second argument at offset 20 instead of offset 16 does not satisfy this contract.

### Exact final-store footprint

Assume bytes 9–11 initially contain distinct nonzero values. Also assume no dependency modifies them, the initial store does not overlap them, and no external writer intervenes.

Those three bytes must retain their values after normal completion. Clearing bytes 8–11 together would violate the required single-byte store at offset 8.

The final-store footprint is 13 bytes, not a contiguous 13-byte interval and not a full 24-byte clear.

### Unused first argument

Under otherwise identical controlled conditions, changing only the first argument must not change the visible body’s selected addresses, stored values, dependency arguments, or control-flow decisions.

This criterion concerns the body’s use of that argument; it does not assert identical behavior under uncontrolled external state.

### Global modification before the copy

Suppose F-c5da7fd8e586 changes the global source word from one value to another before returning normally.

The subsequent initial-address copy must use the value read after that return. Reading and retaining the earlier value before the dependency call would not match the established order.

### Overlap with the initial destination

Suppose final address plus 16 bytes equals the initial address, all accesses are valid, and no other effect interferes.

The later 32-bit second-argument store overwrites the earlier global-word copy at that address. A requirement that the initial copied value always remain until return would be incorrect.

### Interrupted dependency sequence

Suppose the first F-5b24edfb22c0 invocation does not return normally.

The initial global-word copy precedes that invocation. The second invocation and final stores are not reached through the visible normal sequence. No rollback of the initial copy is required by the evidence; the dependency’s own effects remain unspecified.

## Uncertainties

The following are **documented_only**:

- The identity and semantic role of the global source word.
- Object types and meanings of the initialized fields.
- Allocation sizes and complete layouts.
- Whether returned addresses identify bases, interior locations, or unrelated storage.
- Dependency internals, failure conventions, and nonlocal control-flow effects.
- Ownership, release obligations, and long-term lifetime.
- Exact ABI, byte order, and alignment requirements.
- Behavior of invalid or overflowing address calculations.
- Fault behavior and partial effects of failing accesses.
- Concurrency, atomicity, and external visibility guarantees.

An initialization or construction interpretation is inferred intent, not an established object-model contract. No inheritance, reference-counting, allocation, or destruction semantics follow from the visible sequence alone.

## Evidence

The supplied static body establishes two 32-bit inputs, no visible use of the first input, and no explicit return value. It establishes one initial dependency call, a subsequent 32-bit global read and store, and two ordered calls to the second dependency.

The address expressions establish a 4-byte displacement for the first derived-address request and an 8-byte displacement for the second. The final accesses establish, in order, an 8-bit zero at offset 8, a 32-bit zero at offset 12, the 32-bit second argument at offset 16, and a 32-bit zero at offset 20.

Those widths support the 13-byte final footprint and 17-byte aggregate direct-store accounting, subject to possible overlap with the initial destination. No visible validation, rollback, or direct resource-release operation appears.

Dependency behavior, semantic field names, and ownership rules are not established by this evidence. No runtime tests have been performed.
