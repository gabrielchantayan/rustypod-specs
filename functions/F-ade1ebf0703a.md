<!-- Trusted provenance: F-ade1ebf0703a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ade1ebf0703a

## Purpose

F-ade1ebf0703a conditionally updates accounting associated with a metadata-bearing address, then passes the metadata address and a captured context address to dependent operations.

The established sequence consists of two entry dependency calls, a context-address read, and, for nonzero input, header validation, a preparation call, **seven accounting decrements**, and two closing calls. The seven decrements comprise four subtractions using one cached amount, two unit decrements, and one subtraction using a freshly read header amount.

The layout and accounting pattern suggest allocation-release bookkeeping. That is inferred intent, not an established ownership contract. The supplied body does not establish whether any dependency releases storage, combines blocks, acquires synchronization, or releases synchronization.

## Interface

**Input:** One represented 32-bit integer value, used as an address when nonzero. Its signed presentation does not establish that only positive values are accepted. Zero selects the bypass path.

**Output:** No explicit return value. The ordinary path returns after its applicable dependency calls. Invalid header observations invoke F-a817d8d58c10, which the supplied representation marks as non-returning.

**Widths:** The supplied representation uses 32-bit integer values, addresses, headers, and accounting words. Both metadata indices are unsigned 16-bit values, widened before multiplication by four.

For nonzero input, the function directly accesses the following preceding metadata:

| Offset relative to input address | Width | Use |
|---|---:|---|
| −8 bytes | 32 bits | Header validation and masked accounting amount |
| −4 bytes | Unsigned 16 bits | First accounting-table index |
| −2 bytes | Unsigned 16 bits | Second accounting-table index |

The function does not directly access the payload beginning at the input address. This does not rule out payload access through a dependency or through overlapping storage.

A global slot supplies a context address. The function reads that address once, after both entry dependencies, and retains it for subsequent direct accesses and explicit dependency arguments. Later changes to the slot do not select a replacement context.

The ordinary nonzero path updates these context-relative 32-bit words:

| Location | Operation |
|---|---|
| 0x224 plus four times the first metadata index | Subtract cached amount |
| 0x30c | Subtract cached amount |
| 0xe0 plus four times the second metadata index | Subtract cached amount |
| 0x220 | Subtract cached amount |
| 0x310 plus four times the result of F-09dcb55e8e0a | Subtract one |
| 0x390 | Subtract one |
| 0x08 | Subtract freshly read, masked header amount |

**Memory preconditions:** Every reached read, write, and dependency operation must have valid storage and an execution environment that permits the access. No explicit check establishes context validity, metadata accessibility, alignment, table bounds, or sufficient counter values. Effects of invalid accesses are not established.

**Arithmetic limits:** The amount-based subtractions use an unsigned 32-bit masked amount and produce 32-bit results, including modular underflow. The unit decrements and some address calculations are presented as signed integer arithmetic. Their behavior outside the signed representable range is not independently established by the supplied body; a universal wraparound guarantee for those operations would require additional machine-level evidence.

## Behavior

### Entry and zero-input path

Every invocation begins with this sequence:

1. Call F-2de462625f09 without explicit arguments.
2. Call F-2fd8826557a4 without explicit arguments.
3. Read the context address from the global slot.
4. Test whether the input is zero.

If the input is zero, the function returns without reading metadata, writing accounting words, or calling any remaining dependency.

Zero input therefore does not mean “no effects.” Both entry dependencies still execute, and the context slot is still read. Their implementations are absent, so neither normal return nor preservation of global state is established independently.

If an entry dependency changes the context slot and returns, the subsequent context read observes the slot as it then exists. The context is not captured before those calls.

### Header validation

For nonzero input, the metadata-header address is eight bytes before the input address.

Validation uses separate, short-circuited observations:

- First, read the 32-bit header and compare it with zero.
- If that observation is zero, invoke F-a817d8d58c10 without performing the second validation read.
- Otherwise, read the header again and examine bit 0.
- If bit 0 of the second observation is set, invoke F-a817d8d58c10.

Thus, rejection occurs when the first observation is zero, or when the first is nonzero and the second has bit 0 set.

There is no single header snapshot governing validation. In particular, a nonzero first observation followed by a zero second observation passes these checks: the second check tests only bit 0, not whether the second value is nonzero.

F-a817d8d58c10 is represented as non-returning. Under that represented behavior, rejection prevents the preparation call and all accounting updates. The supplied body does not establish a diagnostic, exit status, exception, recovery mechanism, or cleanup action.

Passing validation does not establish:

- A positive accounting amount.
- Valid metadata indices.
- A valid context address.
- Sufficient counter values.
- Continued validity or stability of the header after validation.

### Preparation and cached amount

After validation succeeds, the function calls F-c0fc6d033caa with the captured context address.

After that dependency returns:

1. Read the unsigned 16-bit index at input minus four bytes.
2. Form the first selected accounting location using context offset 0x224 plus four times that index.
3. Read the header again.
4. Retain the header value masked with **0x3ffffffc**.

The mask retains bits 2 through 29 and clears bits 0, 1, 30, and 31. The cached amount is a multiple of four, ranging from zero through **0x3ffffffc**.

This read is not a revalidation. A dependency-induced header change, including a change that would have failed the earlier validation, is not checked again before the amount is used.

### First four accounting decrements

The function performs four ordered read-modify-write operations using the same cached amount:

1. Read the word selected by the first metadata index, subtract the cached amount, and write the result.
2. Read the word at context offset 0x30c, subtract the cached amount, and write the result.
3. Read the unsigned 16-bit index at input minus two bytes. Use it to select the word at context offset 0xe0 plus four times the index; read that word, subtract the cached amount, and write the result.
4. Read the word at context offset 0x220, subtract the cached amount, and write the result.

The second index is read only after the first two accounting writes. It is not captured alongside the first index. Consequently, overlapping storage can affect the second index before it is read.

All four subtractions reuse the cached amount even if an earlier write changes the header through aliasing.

No underflow check or saturation is present. For example, subtracting an amount of four from an accounting word containing zero produces **0xfffffffc**.

### Selector-dependent and fixed unit decrements

After the first four accounting writes, the function calls F-09dcb55e8e0a without explicit arguments.

Its returned integer selects the location at context offset 0x310 plus four times that result. After the dependency returns, the function:

1. Reads the selected 32-bit word, subtracts one, and writes it back.
2. Reads the 32-bit word at context offset 0x390, subtracts one, and writes it back.

There is no visible bounds check or nonnegativity check on the selector result. The body does not establish an intended table extent or guarantee that the selected address lies within it.

These two decrements use signed integer arithmetic in the supplied representation. Ordinary representable cases are clear: zero becomes negative one, whose 32-bit representation is **0xffffffff**. Behavior at the minimum signed value requires information beyond the supplied representation; unchecked subtraction alone does not establish defined signed overflow.

### Fresh-header accounting decrement

After both unit decrements, the function obtains:

- The 32-bit accounting value at context offset 0x08.
- A fresh 32-bit header value, masked with **0x3ffffffc**.

It subtracts the masked header amount from the accounting value and stores the result at context offset 0x08.

**The relative order of these two operand reads is not established.** Both contribute to the subtraction before its result is stored; the supplied expression does not specify which value is read first.

This final subtraction does not reuse the earlier cached amount. A header change after that earlier capture can therefore make the final amount differ from the amount used by the first four subtractions.

There is no additional header validation before this operation.

### Closing calls

After all seven direct accounting writes, the function calls:

1. F-cc6c9b4a0f68 with the captured context address and the metadata-header address, eight bytes before the input.
2. F-e586bff64fb7 with the captured context address.

The function then returns if both dependencies return.

There is no visible status check, rollback, or alternate branch surrounding these calls. Earlier accounting writes are not conditionally delayed until the closing dependencies succeed.

## Side effects

**Direct memory effects:** The ordinary nonzero path performs seven ordered 32-bit accounting read-modify-write operations. No direct assignment targets the metadata fields or payload as such. Aliasing can nevertheless cause an accounting write to modify storage also serving as metadata, payload, or a global slot.

**Dependency effects:** Dependency bodies are unavailable. Their calls must not be treated as preserving globals, counters, metadata, or storage lifetime. A call with no explicit arguments can still access external state.

**Captured context:** Subsequent direct accesses and context arguments use the originally captured address. This is address retention, not a guarantee that the corresponding storage remains valid or unchanged.

**Aliasing:** No non-aliasing guarantee is supplied. Selected accounting locations may coincide with each other or with fixed locations. Updates occur sequentially rather than as simultaneous deductions. A later operation can therefore read a value already modified by an earlier operation.

The same qualification applies to metadata. Earlier writes can affect a later metadata read if their storage overlaps. The cached amount remains unchanged after capture, but the later index and fresh header observation are not protected from such overlap.

**Atomicity and concurrency:** The body describes separate reads, arithmetic, and writes, not a single atomic transaction. Synchronization roles for F-c0fc6d033caa and F-e586bff64fb7 are plausible but unproven. Thread safety is documented_only.

**Ownership:** Passing the metadata-header address to F-cc6c9b4a0f68 does not establish ownership transfer, deallocation, retention, or permission to reuse the input address after return.

**Failure after partial progress:** No rollback is visible if a later memory access or dependency fails after accounting writes have occurred. The concrete failure behavior is unknown.

## Dependencies

| Opaque function ID | Invocation and established use |
|---|---|
| F-2de462625f09 | First operation on every invocation; no explicit arguments |
| F-2fd8826557a4 | Second operation on every invocation; no explicit arguments |
| F-a817d8d58c10 | Invoked on rejected header observations; represented as non-returning |
| F-c0fc6d033caa | Receives captured context after validation and before accounting-index reads |
| F-09dcb55e8e0a | Called after four accounting writes; result selects the first unit-decrement location |
| F-cc6c9b4a0f68 | Receives captured context and metadata-header address after all seven writes |
| F-e586bff64fb7 | Receives captured context as the final ordinary-path dependency |

Only the result of F-09dcb55e8e0a visibly affects subsequent computation. Unused return values do not imply infallibility or absence of side effects.

## Acceptance examples

These are conditional static acceptance cases, not runtime observations. Unless stated otherwise, assume valid address calculations and storage, no relevant concurrent mutation, and dependencies that return without modifying the illustrated storage.

### Zero input

Given zero input and readable context-slot storage:

- F-2de462625f09 precedes F-2fd8826557a4.
- The context slot is then read.
- No metadata read or direct accounting write follows.
- No other listed dependency is called.

### Rejected validation

If the first header observation is zero, F-a817d8d58c10 is invoked without a second validation read.

If the first observation is nonzero and the second has bit 0 set, the same dependency is invoked after the second read.

Under its represented non-returning behavior, neither case reaches preparation or accounting.

### Changing validation observations

If the first header observation is **0x20** and the second is zero, validation passes: the first observation satisfies the nonzero check, and the second has bit 0 clear.

The later accounting amount depends on a subsequent header read, not either validation observation.

### Stable ordinary accounting

Assume non-overlapping illustrated storage, a stable header of **0x00000020**, metadata indices one and two, and selector result two.

The cached and fresh amounts are both 32. In order:

- Context offset 0x228 loses 32.
- Context offset 0x30c loses 32.
- Context offset 0xe8 loses 32.
- Context offset 0x220 loses 32.
- Context offset 0x318 loses one.
- Context offset 0x390 loses one.
- Context offset 0x08 loses 32.

Both closing dependencies follow these seven writes.

### Accepted zero amount

A stable header of **0x00000002** passes validation but yields a masked amount of zero.

The five amount-based operations still perform their reads and writes, although they subtract zero. The two unit decrements still subtract one.

### Different cached and fresh amounts

If the header yields **0x20** when the cached amount is captured, the first four subtractions use 32.

If F-09dcb55e8e0a subsequently changes the header to **0x40**, the final subtraction at context offset 0x08 uses 64. Its context-word read and header read have no established relative order.

### Overlapping locations

If the first metadata index is 58, its selected offset is 0x30c. The first two accounting writes therefore target the same word.

With cached amount four and an initial value of 100, those sequential writes produce 96 and then 92, assuming no intervening external mutation.

### Underflow and unit decrement

An amount-based subtraction of four from zero stores **0xfffffffc**.

A unit decrement from zero stores the representation of negative one, **0xffffffff**. This example does not establish behavior for decrementing the minimum signed value.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the header bits, metadata indices, counters, and selector result.
- Whether the input identifies a live allocation or another object.
- Dependency implementations, failure modes, mutations, and storage-lifetime effects.
- Actual synchronization, if any.
- Ownership and release behavior of F-cc6c9b4a0f68.
- Backing storage sizes and permitted index ranges.
- Alignment, byte order, and invalid-access behavior.
- Signed-overflow behavior and out-of-range address arithmetic in the underlying execution environment.
- Concrete effects of the dependency represented as non-returning.
- Caller-imposed restrictions on aliasing or concurrent modification.

The final subtraction’s operand-read order is unspecified by the supplied representation, not an additional guaranteed sequencing rule.

## Evidence

The supplied static body establishes the conditional branch structure, dependency order and explicit arguments, repeated header observations, metadata access widths, offsets, masking constant, and seven accounting writes. It also establishes the absence of explicit range, capacity, and underflow checks in this body.

The final context-word read and fresh header read are operands of one subtraction expression; their relative evaluation order is not established.

The invalid-header dependency carries a non-returning annotation. Its implementation and concrete termination behavior are not supplied.

Allocation-release bookkeeping is inferred intent. Ownership, synchronization, and dependency-preservation guarantees are not established.

No dependency bodies, runtime traces, complete layout declarations, or platform arithmetic contract were supplied. No runtime tests have been performed.
