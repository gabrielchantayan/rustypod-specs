<!-- Trusted provenance: F-abb7a1ff7712; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-abb7a1ff7712

## Purpose

F-abb7a1ff7712 conditionally allocates a temporary sequence of 16-bit elements, populates it through a dependency, replaces each element encountered before a zero element, submits the temporary address together with the original object, and performs allocator accounting followed by release-related calls.

This describes the established operation sequence, not a particular text transformation. The evidence does not establish that the elements are characters, that replacement performs case conversion, or that submission copies the sequence.

The function has no explicit return value. Its observable behavior consists of memory accesses, dependency calls, temporary-sequence writes, allocator-accounting writes, and any additional effects of dependencies.

## Interface

### Input object

The function receives one address-like argument represented as a 32-bit integer. It initially reads a 32-bit field at byte offset `+4` from that address.

- If the field is zero, the function returns immediately.
- Otherwise, it proceeds to a size query and allocation.

There is no visible validation of the input address, alignment, object extent, or readability. The caller must provide storage suitable for the initial read and meet any additional requirements of dependencies receiving the object.

After allocation succeeds, the function reads the same field again. The population dependency receives this later value, which need not equal the value used by the initial gate. There is no second nonzero check.

The field’s semantic type is **documented_only**. Its use is consistent with a source reference, but that is inferred intent rather than an established contract.

### Allocation request and temporary storage

F-d8cbf9aa79a9 is called without explicit arguments and supplies a signed 32-bit integer. The allocation request is formed by adding one to this result and then multiplying by two.

For a nonnegative result \(n\) whose intermediate and final values are representable, the request is:

\[
2(n+1)\text{ bytes}.
\]

Under signed 32-bit arithmetic, the nonnegative range in which both operations remain representable is \(0 \le n \le 1{,}073{,}741{,}822\). No range check, negative-result check, or explicit overflow handling is visible. Exact behavior outside representable arithmetic is **documented_only**.

F-fab3799b3190 receives the computed request and returns an address or null. A null result ends processing.

The temporary sequence is accessed in 16-bit units, advancing two bytes per element. Traversal requires an accessible zero element. The allocation formula alone does not establish the size query’s units, population capacity, or termination guarantee.

### Result and failure surface

There is no explicit success value or recoverable error code.

Visible outcomes are:

- Return after the initial zero-field check.
- Return after a null allocation result.
- Completion of population, replacement, submission, accounting, and release-related calls.
- Invocation of the non-returning F-a817d8d58c10 when allocation-header validation fails.

No exception handling, rollback, or recovery mechanism is visible. Outcomes after invalid accesses, violated dependency requirements, unrepresentable arithmetic, or unexpected nonlocal control flow remain **documented_only**.

## Behavior

### 1. Initial gate and allocation

The first observable operation is a 32-bit read of the object field at `+4`. If that value is zero, no listed dependency is called.

Otherwise, the function calls F-d8cbf9aa79a9, forms the allocation request, and calls F-fab3799b3190.

Neither dependency visibly receives the object address or the initially read field. Any relationship between the queried size and the object therefore depends on ambient state or an unavailable dependency contract.

A null allocation result causes return without population, replacement, submission, or explicit cleanup.

### 2. Population and traversal

For a nonnull allocation result, the function rereads the object’s `+4` field and calls F-1e32f7e1be72 with:

1. The temporary address.
2. The newly read 32-bit field value.

After this call returns, traversal starts at the first temporary element.

For each position, the function reads the current 16-bit element. A zero value ends traversal. A nonzero value causes a call to F-5afffd4d8a24 with no explicit arguments, followed by a 16-bit write of its replacement result at that position. Traversal then advances two bytes.

The current element, element address, object address, and index are not explicitly supplied to the replacement dependency. An input-dependent mapping cannot be assumed from this interface.

The stopping decision uses the element read before its replacement call. Writing zero does not stop traversal at that position: the next element is still examined. Conversely, a dependency may change later elements, so the stopping position is determined by the values actually read, not by a retained original length.

The terminating zero receives no direct replacement write. There is no visible traversal bound or capacity check.

### 3. Submission

After traversal reaches zero, F-a6afbdf5375f receives:

1. The original object address.
2. The temporary sequence address.

Its result is not visibly inspected.

Whether submission copies data, retains the address, modifies the object, modifies the sequence, or changes allocator state is **documented_only**. The subsequent body still accesses the temporary allocation and its preceding metadata. Submission therefore cannot safely invalidate that storage unless an unavailable contract supplies some compatible arrangement.

### 4. Cleanup entry and header validation

After submission, the function checks the retained temporary address for null. It later performs another nonnull check around the accounting and release body.

The visible path already has a nonnull retained address, with no intervening assignment changing it. These checks do not establish detection of a dependency freeing or invalidating the allocation.

The cleanup sequence then:

1. Calls F-2fd8826557a4 without explicit arguments.
2. Reads a 32-bit allocator-state address from a global location.
3. Identifies the header address eight bytes before the temporary address.
4. Reads the 32-bit header word and checks whether it is zero.
5. If that read was nonzero, reads the header word again and tests bit 0.

A zero first read or a set bit 0 in the second read invokes F-a817d8d58c10, identified as non-returning. No subsequent accounting operations are reached on that path.

These are validation reads, not a retained value for later accounting. If the header can change between reads, the checks need not inspect one stable word. Concurrent mutation and synchronization guarantees are **documented_only**.

Validation does not require a positive masked accounting quantity. A stable header word of `0x00000002`, for example, passes validation while producing a masked quantity of zero.

### 5. Allocator accounting and release

Let the captured allocator-state address be the accounting base. All offsets below are byte offsets. The header starts at temporary offset `−8`; its following two 16-bit selectors occupy temporary offsets `−4` and `−2`.

After validation, F-c0fc6d033caa receives the captured accounting base.

**After that call returns, the first selector read and its address computation precede acquisition of the masked header quantity.** The ordered operations are:

1. Read the unsigned 16-bit selector at temporary offset `−4`.
2. Multiply that selector by four and compute its allocator-state-relative address.
3. Read the current 32-bit header word and mask it with `0x3ffffffc`, obtaining \(Q_1\).
4. Read the 32-bit accounting field at base `+0x224 + 4 × first selector`, subtract \(Q_1\), and write the result there.
5. Read the field at base `+0x30c`, subtract the same \(Q_1\), and write the result there.
6. Read the unsigned 16-bit selector at temporary offset `−2` and compute its allocator-state-relative address.
7. Read the field at base `+0xe0 + 4 × second selector`, subtract \(Q_1\), and write the result there.
8. Read the field at base `+0x220`, subtract \(Q_1\), and write the result there.
9. Call F-09dcb55e8e0a without explicit arguments.
10. Use its signed 32-bit result as the index for the field at base `+0x310 + 4 × index`; read that field, decrement it by one, and write it back.
11. Read the field at base `+0x390`, decrement it by one, and write it back.
12. Read the field at base `+8` and read the header word again; subtract the newly masked header quantity from that field and write the result at base `+8`.
13. Call F-cc6c9b4a0f68 with the captured accounting base and header address.
14. Call F-e586bff64fb7 with the captured accounting base.

The mask retains header bits 2 through 29 and clears the other four bits.

The first four quantity subtractions reuse \(Q_1\). The final subtraction uses a fresh header read after F-09dcb55e8e0a and the two decrements. It may therefore use a different quantity. This difference can arise through dependency mutation or, where addresses overlap, through preceding direct writes.

The allocator-state address is captured before F-c0fc6d033caa. Changing the global address afterward does not by itself redirect subsequent accounting accesses or call arguments.

The source establishes the first selector read before the \(Q_1\) header read. It does not establish a total evaluation order between the two operand reads within the final subtraction.

## Side effects

### Direct writes and ordering

The function directly writes:

- A 16-bit replacement at each position whose traversal read was nonzero.
- Four 32-bit accounting fields through subtraction of \(Q_1\).
- Two 32-bit counters through decrement by one.
- One 32-bit accounting field through subtraction of a freshly masked header quantity.

Each accounting update is a separate read-modify-write operation. The sequence is not shown as one atomic transition.

The addressed fields are not proven distinct. Indexed fields may overlap fixed fields or other indexed fields. Later reads then observe earlier writes, absent intervening mutation. No non-aliasing guarantee between allocator state and temporary metadata is supplied.

There is no direct object write in the visible body. Dependencies receiving the object or accessing ambient state may nevertheless modify it.

### Widths and arithmetic

Sequence reads and writes are 16-bit. Both embedded selectors are unsigned 16-bit values, widened before multiplication by four; their scaled values range from zero through 262,140.

The header and masked quantities are unsigned 32-bit values. Quantity subtraction produces an unsigned 32-bit result, with no visible underflow check or saturation. For otherwise valid accesses, subtraction below zero wraps modulo \(2^{32}\).

The two decrement operations use signed 32-bit accesses. Signed-overflow behavior is not established by this representation. The signed dependency-supplied index, allocation-size arithmetic, and address calculations likewise have no explicit overflow protection.

No bounds checks constrain the selectors or dependency-supplied index. Required preconditions include accessible, suitable storage at every computed address and representable address calculations under the target’s rules.

### Ownership and lifetime

The function requests storage and later performs a metadata-based release sequence. This supports an inference of temporary ownership, but does not establish which dependency releases storage or whether any dependency retains an alias.

Required storage must remain valid through traversal, submission, header validation, accounting, and any dependency access that precedes actual release. The evidence does not establish ownership transfer, reference counting, allocator locking, or preservation of memory by unknown callees.

## Dependencies

All behavior beyond these observed interfaces is **documented_only**.

| Function ID | Observed interface and position |
|---|---|
| F-d8cbf9aa79a9 | No explicit arguments; signed 32-bit result used in allocation-size arithmetic. |
| F-fab3799b3190 | Receives the computed size; returns the temporary address or null. |
| F-1e32f7e1be72 | Receives the temporary address and the object field reread after successful allocation. |
| F-5afffd4d8a24 | No explicit arguments; called for each element read as nonzero; replacement is stored as 16 bits. |
| F-a6afbdf5375f | Receives the original object address and temporary address after traversal. |
| F-2fd8826557a4 | No explicit arguments; called before global allocator-state capture. |
| F-a817d8d58c10 | Non-returning validation-failure call. |
| F-c0fc6d033caa | Receives captured allocator state before the first selector read and accounting operations. |
| F-09dcb55e8e0a | No explicit arguments; signed 32-bit result selects a counter to decrement. |
| F-cc6c9b4a0f68 | Receives captured allocator state and the header address after accounting. |
| F-e586bff64fb7 | Receives captured allocator state after F-cc6c9b4a0f68 returns. |

No explicit arguments does not mean no inputs. Dependencies may consult or modify ambient state. No preservation of globals, sequence contents, metadata, or accounting fields is assumed.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests.

### Initial zero field

Given a readable object whose initial `+4` field is zero, the function returns after that read. None of the listed dependencies is called.

### Allocation failure

Given an initially nonzero field and a null allocation result, the size query precedes allocation. Population, replacement, submission, and explicit cleanup do not occur.

### Field changes before population

If the initial field is nonzero but becomes zero before successful allocation returns, F-1e32f7e1be72 receives zero as its second argument. Processing is not stopped by a repeated gate.

### Empty sequence

If population leaves zero at the first element, no replacement call occurs. Submission and the normal cleanup sequence still follow, provided dependencies return normally and required storage remains valid.

### Replacement writes zero

If traversal begins with elements `[7, 9, 0]` and dependencies otherwise leave the sequence unchanged, replacement results zero and 12 produce `[0, 12, 0]` at submission. Both nonzero input positions are visited.

### Invalid and zero-quantity headers

If the first validation read is zero, F-a817d8d58c10 is called without the second validation read. If the first read is nonzero and the second has bit 0 set, the same failure call occurs.

A stable header of `0x00000002` passes validation. With no subsequent header mutation, both masked quantities are zero; the two counter decrements still occur.

### First selector precedes quantity acquisition

If F-c0fc6d033caa returns normally, the next metadata read is the unsigned selector at temporary offset `−4`. Its scaled allocator-state address is computed before the header read supplying \(Q_1\). A behavior that acquires \(Q_1\) first does not match the established order.

### Header changes during accounting

Suppose the header read following that first selector yields a masked quantity of 32. If F-09dcb55e8e0a later changes the header so its masked quantity becomes 64, the first four subtractions use 32 and the final `+8` subtraction uses 64, provided no other relevant changes intervene.

### Unsigned accounting underflow

If a quantity-subtraction field contains 8 and the applicable masked quantity is 12, its unsigned 32-bit result is `0xfffffffc`, assuming valid access and no intervening mutation. There is no saturation or failure branch for this underflow.

## Uncertainties

The following remain **documented_only**:

- Object identity, layout beyond the accessed field, and field semantics.
- The origin and valid range of the size-query result.
- Sequence meaning, encoding, capacity, and termination guarantees.
- Replacement semantics and the dependency’s full return representation.
- Submission’s mutation, copying, retention, and ownership behavior.
- Actual release responsibility and allocator synchronization.
- Valid selector and counter-index ranges.
- Aliasing constraints among temporary storage, metadata, and allocator fields.
- Exact target handling of signed overflow and invalid address arithmetic.
- Dependency failures, global mutations, and nonlocal control flow.
- Concurrent access guarantees and operand-read ordering not explicitly established above.

Inferred intent is limited to temporary sequence processing followed by allocator-managed cleanup. Stronger text-processing or allocator semantics are not required for acceptance.

## Evidence

Established behavior derives from the supplied function body: explicit accesses, widths, branches, dependency arguments, traversal stride, mask, address calculations, and ordered calls and writes.

In particular, the first unsigned 16-bit selector read and allocator-state address computation occur after F-c0fc6d033caa returns and before the header read used to obtain \(Q_1\).

Dependency implementations and runtime observations were not provided. No runtime tests have been performed. The acceptance examples state conditional static consequences, not measured outcomes.
