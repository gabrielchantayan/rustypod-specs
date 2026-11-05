<!-- Trusted provenance: F-1af8cff970ad; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1af8cff970ad

## Purpose

F-1af8cff970ad initializes a caller-supplied memory region, obtains a backing address through F-871e64ddc6de, initializes three consecutive 32-bit words at that address, establishes address-valued metadata, invokes F-b00260d1087a on an embedded region, and then clears a range determined by the metadata present after that invocation.

The established behavior is a sequence of memory operations and dependency calls. A constructor-like initializer for a container is a plausible interpretation, but object type, ownership, and the meanings of the metadata are not established.

The function returns the original input address on normal completion. It has no explicit failure-status result and no established safe handling of a null backing address.

## Interface

### Input

One address identifying writable caller-provided storage.

The visible operations require access to the following layout:

| Offset from input | Width | Established use |
|---|---:|---|
| +0 | 8 bits | Written with zero at entry. |
| +1 through +3 | 3 bytes | Not directly accessed by the visible body. |
| +4 | 32 bits | Initially zeroed, then assigned the result of F-871e64ddc6de; subsequently used as a backing address. |
| +8 | 32 bits | Initially zeroed, then assigned the backing address plus 12; later used to determine the clearing bound. |
| +12 | 32 bits | Initially zeroed, then assigned the backing address plus 128. |
| +16 | Unknown extent | Address passed to F-b00260d1087a. |

The dependency invocation at offset +16 prevents establishing a complete minimum object size from this body alone. Sixteen writable bytes cover the direct accesses to the outer region, but do not establish adequate storage for the dependency.

The recovered representation uses 32-bit address values and 32-bit metadata fields. The three initial backing-store writes and every subsequent clearing write are also 32 bits wide.

### Output

On normal completion, the function returns exactly the address supplied by the caller, not the backing address and not the embedded-region address.

The output state includes all direct writes described below and any effects of the dependencies. The function does not return a separate success flag, allocation size, initialized-word count, or error code.

### Preconditions for ordinary completion

Ordinary successful use requires:

- The input address supports the direct accesses at offsets +0, +4, +8, and +12.
- F-b00260d1087a accepts the address at input +16 and has whatever additional storage or state it requires.
- The address returned by F-871e64ddc6de supports the initial three-word initialization.
- The metadata present after F-b00260d1087a returns describes a range that can safely be written by the final clearing loop.
- Address arithmetic, alignment, and accessibility are valid for the execution environment.

These are safety requirements inferred from the accesses, not validations performed by F-1af8cff970ad.

## Behavior

### 1. Initialize the outer metadata

The function performs these direct writes in order:

1. Write an 8-bit zero at input +0.
2. Write a 32-bit zero at input +4.
3. Write a 32-bit zero at input +8.
4. Write a 32-bit zero at input +12.

The direct body does not initialize the intervening bytes at offsets +1 through +3.

All four writes precede the first dependency call. This is not an atomic reset: a failure or externally observable interruption can expose a partially initialized state.

### 2. Obtain and publish a backing address

The function invokes F-871e64ddc6de with two arguments whose values are:

- First argument: 128.
- Second argument: zero.

The returned value is treated as an address and stored in the 32-bit field at input +4.

A request for 128 bytes is a plausible interpretation of the first argument, but the dependency’s contract is unavailable. The argument must not be promoted into a guarantee that 128 writable bytes were allocated, that the storage is fresh, or that ownership was transferred.

The backing address is published in the outer region before initialization of its first three words.

### 3. Initialize three consecutive backing words

Using the returned address as a local cursor, the function performs exactly three iterations. Each iteration:

- Checks whether the current cursor is zero.
- If it is nonzero, writes a 32-bit zero at that cursor.
- Advances the cursor by four bytes, regardless of whether a write occurred.

For an ordinary nonzero backing address, this writes zeros at displacements 0, 4, and 8 from the returned address.

The zero check applies separately to each cursor position. It is not a check that suppresses the entire three-word initialization when the original return value is zero.

Consequently, if the returned address is zero, the first iteration skips its write, but the next two iterations attempt writes at addresses 4 and 8. On an environment where those addresses are inaccessible, execution faults during this phase. There is no explicit recovery or early return.

Address wraparound can also affect an individual cursor check near the top of the address space. Whether such addresses are usable is environment-dependent; the visible body supplies no range validation.

### 4. Establish two derived addresses

After the three iterations, the function reads the backing-address field at input +4 and uses it to establish:

1. The 32-bit value at input +12 as the backing address plus 128.
2. The 32-bit value at input +8 as the backing address plus 12.

The write at +12 precedes the write at +8.

These calculations use the stored backing-address field, not the final advanced cursor. Under ordinary non-aliasing conditions, that field still contains the dependency’s returned address. If the initial backing writes overlap the outer metadata, however, they may have changed the value subsequently read.

The derived addresses suggest a capacity boundary and a boundary after three words, respectively. Those meanings remain inferred intent rather than an independently established type contract.

### 5. Invoke the embedded-region dependency

The function invokes F-b00260d1087a with the address input +16.

This occurs after the initial three backing writes and after both derived-address fields have been assigned.

The dependency’s return value is not used by the visible body. Its memory effects are not specified by the supplied evidence. In particular, the final phase must not be described as operating on guaranteed-preserved metadata: no available contract establishes that this dependency preserves the fields at +4 or +8, the backing storage, or other state.

### 6. Clear the metadata-selected range

The function starts an unsigned 32-bit element index at zero.

For each loop-condition evaluation, it reads the 32-bit fields at input +8 and input +4. The recovered operation subtracts the backing-address value from the boundary value as a signed 32-bit difference, shifts that difference right by two, converts the resulting bound to unsigned, and compares the index against that unsigned bound.

Whenever the comparison succeeds, the function uses the current backing-address field to select the destination and writes a 32-bit zero at:

- The backing address plus four times the current index.

It then increments the index and evaluates the condition again.

Important consequences:

- The bound is not established once before F-b00260d1087a.
- The bound is not cached once for the entire clearing phase.
- The backing-address field is used again for the write destination.
- The field at input +12 does not constrain this clearing phase.
- No check establishes that the boundary is at or above the backing address.
- No check establishes that the difference is divisible by four.
- No check establishes that the selected range lies within the storage associated with the earlier request for 128.

With unchanged metadata, ordinary non-overlapping addresses, and a difference of 12, the bound is three. The function then zeros the same three backing words that it initialized before the embedded-region call.

For a nonnegative signed difference, the shifted value corresponds to the number of complete four-byte units in that difference. A residual one to three bytes does not cause a partial-width write.

A negative signed difference does not provide a safe “empty range” condition. Under the recovered arithmetic-right-shift interpretation, converting the negative shifted result to unsigned produces a large bound.

Because the loop rereads metadata, writes that overlap those fields can change subsequent conditions or destinations. The ordinary three-word description therefore requires non-aliasing assumptions; it is not unconditional behavior for every possible address.

### 7. Return

After the clearing loop terminates, the function returns the original input address.

No additional direct cleanup, release, or rollback is visible.

## Side effects

### Direct memory effects

The direct body can write:

- One byte at input +0.
- Three 32-bit fields at input +4, +8, and +12.
- Up to three initially selected backing words, subject to the per-cursor zero check and successful execution.
- Every 32-bit word selected by the final clearing loop.

It does not directly initialize all 128 bytes of the nominal backing region. Under the ordinary unchanged-metadata case, only its first 12 bytes are directly cleared.

The direct body has no explicit global-memory access. That does not establish that the dependencies are free of global effects.

### Ownership and lifetime

The function stores an externally obtained address in caller-provided storage. No direct release operation appears.

The evidence does not establish:

- Who owns the returned backing storage.
- Whether either dependency retains an address.
- Whether a dependency releases or replaces storage.
- Whether repeated calls on the same input are supported.
- Whether previously referenced storage is reclaimed.

Calling this function on an already initialized region overwrites its visible address fields without any visible preliminary release. A leak is possible if those fields represent owned resources, but that ownership premise is not established here.

### Errors and partial state

There is no explicit error branch for an invalid input address, a zero backing address, an inaccessible backing region, or malformed post-dependency metadata.

Memory faults or dependency failures can leave partial writes visible. There is no visible transaction boundary or rollback.

## Dependencies

### F-871e64ddc6de

Called once, after the four initial outer-region writes, with arguments 128 and zero.

Its returned address is stored at input +4 and used for the three initial backing-word operations. The meanings of its arguments, allocation guarantees, failure convention, ownership rules, alignment guarantees, and global effects are unknown.

### F-b00260d1087a

Called once with input +16, after the backing words and derived-address fields have been initialized.

Its result is ignored. Its required region extent, memory effects, ownership behavior, failure behavior, and global effects are unknown. The final clearing phase uses metadata read after this call rather than relying on saved pre-call values.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

### Ordinary initialization

Given writable non-overlapping outer and backing regions, a nonzero backing address returned by F-871e64ddc6de, ordinary address arithmetic, and F-b00260d1087a preserving the relevant metadata:

- The byte at input +0 becomes zero.
- The fields at +4, +8, and +12 contain the backing address, backing address plus 12, and backing address plus 128, respectively.
- The first three backing words are zeroed before F-b00260d1087a is called.
- Those same three words are zeroed again after it returns.
- The return value equals the original input address.
- No direct clearing of the remaining 116 backing bytes is required.

### Embedded dependency writes into the initial range

If F-b00260d1087a writes nonzero values into the first three backing words while preserving the address metadata and keeping the storage valid, those values are overwritten with zeros by the final phase.

### Embedded dependency selects an empty range

If F-b00260d1087a leaves equal values at input +4 and input +8, the final clearing loop performs no backing-word writes.

The three earlier initialization iterations have already occurred.

### Embedded dependency extends the range

If the dependency leaves a valid backing address at +4 and a boundary exactly 16 bytes above it at +8, the final phase clears four consecutive 32-bit words, assuming those writes do not alter the metadata or fault.

The earlier initialization still affected only three words.

### Non-word-aligned boundary

If the post-dependency signed difference is 14, the final phase clears three complete 32-bit words. It does not clear the remaining two bytes as a separate operation.

### Zero returned address

If F-871e64ddc6de returns zero, the function stores zero at input +4, skips the first initial backing write, and then attempts the next write at address 4.

Safe failure, a null return, or successful continuation is not an acceptance requirement. On a system where address 4 is inaccessible, a fault at that write is consistent with the visible behavior.

### Boundary below the backing address

If the post-dependency signed difference is negative, the function does not explicitly reject it or treat it as an empty range. Under the recovered signed-shift interpretation, a large unsigned clearing bound is expected; ordinary bounded initialization is not guaranteed.

## Uncertainties

The following remain documented_only:

- The concrete object type and semantic names of its fields.
- The complete size and alignment requirements of the outer region.
- The contracts and implementation effects of both dependencies.
- Allocation, release, retention, and ownership rules.
- Whether overlap between outer and backing storage is permitted.
- Whether concurrent or reentrant observation is supported.
- The target’s precise behavior for invalid addresses and exceptional dependency exits.
- Exact machine-level overflow and shift semantics beyond the recovered 32-bit signed-difference and right-shift representation.

The ordinary examples assume valid addresses, no harmful overlap, no concurrent mutation, and arithmetic that does not cross the signed-difference or address-wrap boundaries. They do not establish validation of those assumptions.

## Evidence

The supplied static body establishes the access widths, direct offsets, dependency arguments, operation order, three-iteration initialization, post-dependency metadata reads, clearing-loop structure, and identity return.

The constructor-like purpose, allocation interpretation, and possible size/capacity interpretation are inferences, not established dependency contracts.

The supplied material also contains recovery warnings concerning unreachable-block removal and analysis reconstruction. No missing behavior has been invented from those warnings.

No runtime tests have been performed.
