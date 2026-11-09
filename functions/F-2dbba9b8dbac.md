<!-- Trusted provenance: F-2dbba9b8dbac; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2dbba9b8dbac

## Purpose

F-2dbba9b8dbac selects between a context-free allocation path and a context-backed allocation path, returning a 32-bit result.

For a non-null context, the visible behavior combines:

- Eligibility checks against a backing handle and a configured size.
- Preferential selection of a numbered entry when the request is close to the configured size.
- A backing allocation attempt.
- Additional allocation attempts after making previously unmarked numbered entries available to the backing dependency.

These allocation-oriented descriptions are **inferred intent**. The established behavior is the sequence of checks, context accesses, bitmap changes, dependency calls, and returned values described below. The supplied material does not establish the dependencies’ allocation, ownership, or error contracts.

## Interface

### Inputs

The function accepts two explicit inputs:

| Input | Representation | Role |
|---|---|---|
| Context | Nullable pointer to 32-bit words | Selects the context-free path when null; otherwise supplies mutable state and numbered entries. |
| Requested size | Unsigned 32-bit integer | Used in unsigned eligibility comparisons and passed unchanged to allocation dependencies. |

The visible return type is a signed 32-bit integer. Its contents should be treated as an opaque 32-bit result: zero is used for several failure or rejection outcomes, while nonzero dependency results are returned without reinterpretation.

A context entry can also be returned directly. That return is not checked for zero, so the function can update state and then return zero without attempting a backing allocation.

The context-free path additionally forwards an implicit machine-state value to a dependency. This value is not an explicit caller argument in the supplied interface. Its portable meaning is unknown.

### Context layout

Offsets below are byte offsets from the context base. Each word is 32 bits wide.

| Offset | Visible use |
|---|---|
| 0 | Backing handle; tested for zero and passed to backing dependencies. |
| 4 | No explicit access shown. |
| 8 | Configured size; interpreted as unsigned for comparisons and passed as a 32-bit value to an entry-processing dependency. |
| 12 onward | Numbered entry words, with entry zero at offset 12 and entry index \(i\) at offset \(12 + 4i\). |
| 20 | Bitmap used to select and mark numbered entries. |
| 24 | Entry count, interpreted as unsigned. |

**The entry region overlaps the control fields.** Entry index 2 occupies the bitmap word at offset 20. Entry index 3 occupies the count word at offset 24. This contract does not replace that overlapping representation with an inferred separate array.

Consequently, processing entry 2 can expose a bitmap value after its selection bit has already been set. Processing entry 3 accesses the same word used for the entry count. Whether valid callers restrict the count to avoid these overlaps is unknown.

### Preconditions

For the null-context path, the global state accessed after initialization must support the indicated read, and the dependencies must accept the arguments described below.

For the non-null path:

- Every context word actually accessed must be readable.
- The bitmap word must be writable whenever an unmarked entry is encountered.
- Every selected entry location must be readable.
- Backing handles and other arguments must satisfy the relevant dependency contracts.
- Entry indices must remain within a range for which entry addressing and shift behavior are defined by the actual execution environment.

The function performs no explicit validation of the context’s extent, entry storage, alignment, count plausibility, or dependency-specific handle validity.

## Behavior

### 1. Context-free path

When the context pointer is null:

1. F-2fd8826557a4 is called without explicit arguments.
2. After that call returns, a 32-bit value is read through the global reference shown in the supplied material.
3. F-5e43693288bd is called with these ordered arguments:
   - The value obtained through that global reference.
   - The requested size.
   - Zero.
   - The constant `0x38`.
   - Zero.
   - Zero.
   - Zero.
   - The implicit machine-state value associated with the function’s entry.
4. The dependency’s result is returned directly.

The global read occurs **after** F-2fd8826557a4. It must not be modeled as a value necessarily captured before that call.

No local check of the requested size occurs on this path. Zero and the largest unsigned 32-bit request are both forwarded if execution reaches the allocation dependency. There is no visible local retry or translation of its result.

### 2. Initial context-backed eligibility checks

For a non-null context, the function first reads the backing-handle word.

- If it is zero, the function returns zero immediately.
- Otherwise, it compares the configured size with the requested size as unsigned 32-bit values.
- If the configured size is smaller than the requested size, the function returns zero immediately.

The configured-size comparison is bypassed when the backing handle is zero. Neither rejection performs an explicit context write or dependency call.

Equality is permitted: a request equal to the configured size is not rejected.

### 3. Preferential numbered-entry selection

After eligibility succeeds, the function evaluates whether the requested size is at least the configured size minus 16. The subtraction is unsigned 32-bit arithmetic.

For configured sizes of at least 16, this selects requests within the final inclusive 16-byte interval up to the configured size, subject to the earlier size limit.

For configured sizes below 16, subtraction wraps modulo \(2^{32}\). In combination with the earlier requirement that the request not exceed the configured size, this prevents entry into the preferential-selection scan.

When the condition holds, entries are considered in ascending index order, beginning at zero. At each iteration:

1. The current unsigned entry count is consulted to determine whether the index remains in range.
2. A mask is derived by shifting a one-bit value by the low eight bits of the index.
3. The bitmap is tested with that mask.
4. If the tested bits are already set, scanning continues.
5. Otherwise, the bitmap is updated by OR-ing in the mask.
6. Only after that write, the corresponding entry word is read and returned.

A selected entry is returned immediately. There is no backing allocation attempt, entry-processing call, or check that the returned entry is nonzero.

If the scan finds no unmarked entry, execution continues to the backing allocation path.

The shift’s low-eight-bit input is established by the supplied representation. The result for shift amounts of 32 or more is not established; see **Uncertainties**. Ordinary one-bit bitmap reasoning is therefore conditional on an index range with defined shift behavior.

### 4. Initial backing allocation attempt

If execution has not already returned, F-bc7ecdbf25ab is called with:

1. The backing handle read from the context.
2. The original requested size.
3. The constant `0x38`.

A nonzero result is returned immediately.

A zero result starts the recovery scan described next. The initial dependency call can change state; the function does not establish that context fields or global values remain unchanged across it.

### 5. Entry processing and allocation retries

After a zero result from the initial backing attempt, a second scan begins at entry index zero. It does not resume at the end of the preferential scan.

For each index:

1. The current entry count is compared with the index as unsigned values. If the index is no longer below the count, the most recent allocation result is returned. On this exhaustion path, that result is zero.
2. The mask for the index is formed using the same low-eight-bit shift input as above.
3. The bitmap is tested.
4. If the entry is already marked, no dependency is called for it.
5. If the entry is unmarked:
   - Its mask is OR-ed into the bitmap.
   - F-ef3e50369ccb is called with the backing handle, the entry word, and the configured size.
   - After that call returns normally, F-bc7ecdbf25ab is called again with the backing handle, the original requested size, and `0x38`.
6. A nonzero retry result ends the scan and is returned.
7. Otherwise, scanning continues with the next index.

The bitmap write precedes both the entry-processing call and the allocation retry. The function does not undo that write when a retry returns zero.

The result of F-ef3e50369ccb is not visibly tested or used to gate the retry. Its normal return is followed by a new allocation attempt.

The backing handle used for the retry is obtained after F-ef3e50369ccb returns. It must not be assumed equal to the handle supplied to that dependency. Subsequent count and bitmap checks likewise use the state available when those checks occur.

There is no additional allocation attempt merely because the scan reaches its end. Retries are associated only with entries that the recovery scan finds unmarked.

## Side effects

### Explicit context writes

The only explicit context writes shown are bitmap updates that set the mask for a selected or processed entry.

The function does not explicitly:

- Clear any bitmap bit.
- Zero an entry after selecting or processing it.
- Decrement the entry count.
- Reset the backing handle.
- Restore state after an unsuccessful retry.

Because the entry representation overlaps the bitmap, a bitmap update also changes the contents of entry index 2.

### Dependency effects and ownership

All four dependencies may have additional effects not established here. In particular, the supplied material does not prove that they preserve context memory, preserve globals, avoid reentrancy, allocate memory, or release memory.

F-ef3e50369ccb receives a backing handle, an entry value, and a size. Its apparent role is to make that entry available to the backing allocator. Whether it frees, transfers, registers, or otherwise processes the entry is **inferred intent**, not an established ownership contract.

Likewise, returning an entry does not by itself establish ownership transfer or a caller obligation to release it.

No rollback, synchronization operation, or exception-handling mechanism is visible. Concurrency guarantees are unknown. If a dependency fails to return normally, this function’s subsequent actions and final result are not established.

## Dependencies

| Opaque ID | Established use | Unknown contract |
|---|---|---|
| F-2fd8826557a4 | Called first on the null-context path. | Initialization effects, global changes, failure behavior, and ownership effects. |
| F-5e43693288bd | Receives the post-initialization global value, requested size, fixed arguments, and implicit machine-state value; its result is returned directly. | Meaning of the global value, `0x38`, implicit value, result, and memory ownership. |
| F-bc7ecdbf25ab | Receives a context backing handle, requested size, and `0x38`; zero causes recovery processing, while nonzero is returned. | Allocation semantics, failure causes, side effects, and result ownership. |
| F-ef3e50369ccb | Called after marking an entry, with the backing handle, selected entry value, and configured size. | Whether entry storage is consumed or released; whether context or global state changes; failure signaling. |

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence. They are not executed tests. Unless stated otherwise, the examples assume valid memory, defined shifts, normal dependency returns, and no dependency mutation of context state.

### Null context uses the post-call global value

Given a null context, suppose F-2fd8826557a4 changes the value later obtained through the global reference from one handle to another.

Expected behavior:

- F-2fd8826557a4 runs first.
- F-5e43693288bd receives the later value, the unchanged request, and the specified fixed and implicit arguments.
- Its result is returned without a context-backed attempt.

### Zero backing handle rejects immediately

Given a non-null context whose backing handle is zero:

- The result is zero.
- No configured-size comparison is needed.
- No dependency is called.
- No bitmap update occurs.

### Oversized request rejects without recovery

Given a nonzero backing handle, configured size 64, and requested size 65:

- The result is zero.
- Neither entry scan runs.
- No backing dependency is called.

### Inclusive preferential threshold

Given configured size 64 and an unmarked entry zero:

- Request 48 qualifies for preferential selection because \(64 - 16 = 48\).
- The entry-zero bit is set before entry zero is read.
- Entry zero is returned without calling F-bc7ecdbf25ab.

Request 47 does not qualify for that preferential selection. It instead proceeds to the initial backing attempt.

### Small configured size wraps the subtraction

Given configured size 8 and request 8:

- The initial size check permits the request.
- The unsigned subtraction produces `0xfffffff8`.
- Preferential selection is skipped.
- F-bc7ecdbf25ab is attempted.

### Direct selection can return zero

Given a qualifying preferential request, an unmarked entry zero, and an entry-zero value of zero:

- The bit is set.
- Zero is returned immediately.
- No allocation dependency is called.

Zero therefore does not prove that the context was unchanged.

### Recovery marks before retrying

Given a request that skips preferential selection, an initial backing result of zero, and an unmarked entry zero:

- Entry zero is marked.
- F-ef3e50369ccb receives its value and the configured size.
- F-bc7ecdbf25ab is retried after that call returns.
- If the retry succeeds, its nonzero result is returned and the mark remains set.

### Marked entries do not trigger retries

Given an initial backing result of zero and a bitmap marking every entry within the count:

- The recovery scan processes no entry through F-ef3e50369ccb.
- No allocation retry occurs.
- The function returns zero.

### Overlapping entry observes the bitmap write

Given a qualifying preferential request, count 3, bitmap value 3, and no intervening mutation:

- Indices 0 and 1 are already marked.
- Index 2 is unmarked.
- The bitmap becomes 7.
- Entry index 2 aliases that bitmap word, so the returned value is 7, not its prior value of 3.

## Uncertainties

The following are **documented_only**:

- The real semantic meaning of the context, backing handle, numbered entries, bitmap, configured size, and `0x38`.
- The valid context extent and permitted entry count.
- Whether the overlapping entry/control layout is intentional or reflects limitations of the supplied representation.
- Exact shift behavior when the low-eight-bit shift amount is at least 32.
- Address arithmetic behavior for very large entry indices.
- The meaning and calling-convention treatment of the implicit machine-state argument.
- Dependency ownership, mutation, error, concurrency, and reentrancy contracts.
- Whether zero has meanings beyond the local rejection and retry decisions shown here.
- Behavior omitted from the supplied representation.

These unknowns must not be resolved by assuming conventional allocator behavior or independent entry storage.

## Evidence

The established claims come from the supplied static representation: explicit branch conditions, 32-bit word accesses, unsigned comparisons and subtraction, overlapping offsets, bitmap updates, dependency argument lists, and return paths.

The supplied material also reports removed unreachable regions. Those reports provide no behavioral specification for the omitted regions and do not establish whole-program reachability.

Allocation terminology and the proposed role of entry processing are inferred intent. No runtime tests, dependency implementations, authoritative external contracts, or execution traces were available.
