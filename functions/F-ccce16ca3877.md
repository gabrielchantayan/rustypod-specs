<!-- Trusted provenance: F-ccce16ca3877; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ccce16ca3877

## Purpose

F-ccce16ca3877 conditionally tears down an object whose second 32-bit word matches an externally stored discriminator. On the matching path, it optionally invokes a dependency using the object’s first word, clears the discriminator field, and performs header-based accounting and finalization through an externally selected context.

**Established static behavior:** a null argument and a discriminator mismatch return without invoking dependencies or writing memory. A matching object follows the teardown path even when its first word is zero. That path can reach a non-returning dependency if the preceding header fails a validity check.

**Inferred intent:** the object appears to be a managed allocation with an eight-byte preceding header. The surrounding operations resemble payload cleanup, accounting removal, and allocation release. These interpretations do not establish ownership or the semantics of any dependency.

The contract below describes the supplied static representation. No runtime tests have been performed.

## Interface

### Argument and return

The function accepts one pointer to an object. The represented interface has no return value.

A null pointer is explicitly accepted. For a non-null pointer, the function initially accesses the 32-bit word at object offset `+4`. It does not initially validate the pointer, allocation extent, alignment, or accessibility.

On the matching path, the function also accesses the first object word and an eight-byte region immediately preceding the object.

### Object and header layout

Offsets are byte offsets relative to the supplied object pointer:

| Offset | Width | Observed use |
|---|---:|---|
| `−8` | 32 bits | Header validity checks and masked accounting quantity |
| `−4` | 16 bits | First context-accounting index |
| `−2` | 16 bits | Second context-accounting index |
| `0` | 32 bits | Optional dependency argument |
| `+4` | 32 bits | Discriminator compared with an external value, then cleared on the matching path |

The two 16-bit fields are independently loaded as unsigned values. Their byte ordering is target-dependent; the supplied representation does not establish endianness.

The header address passed to a dependency is the object address minus eight bytes.

### External state

The function directly references:

- An external discriminator value used for the initial comparison.
- An external location containing a context pointer.

The context contains the following accessed 32-bit fields:

| Context-relative address | Operation |
|---|---|
| `0x224 + 4 × first header index` | Subtract a cached masked header quantity |
| `0x30c` | Subtract that same cached quantity |
| `0xe0 + 4 × second header index` | Subtract that same cached quantity |
| `0x220` | Subtract that same cached quantity |
| `0x310 + 4 × dependency-provided index` | Decrement by one |
| `0x390` | Decrement by one |
| `0x08` | Subtract a newly read masked header quantity |

These are address expressions, not evidence that the fields belong to separate, nonoverlapping arrays. No index bounds or non-aliasing checks are present.

### Width and validity preconditions

The supplied representation uses 32-bit object words, context fields, addresses, and integer results, with 16-bit unsigned header-index loads.

For a normal completion, every address actually accessed must be valid for the corresponding access. Dependencies must also satisfy whatever preconditions their own implementations require. Those requirements are not supplied.

No checks establish that the context is valid, header indices are in range, arithmetic remains within an allocation, or accounting fields are large enough for subtraction.

## Behavior

### 1. Initial eligibility check

If the argument is null, the function returns immediately. No discriminator, header, or context access occurs.

Otherwise, it reads the object’s second 32-bit word and compares it with the external discriminator. A mismatch returns immediately, without reading the object’s first word or preceding header and without calling dependencies.

A match enters the teardown path. There is no later discriminator recheck.

### 2. Optional first-word processing

The function tests whether the first 32-bit object word is nonzero.

If nonzero, it invokes **F-8349c252c168** with:

1. The first object word.
2. The integer value `4`.

The representation separately expresses the nonzero test and the value supplied as the argument. It does not establish an atomic snapshot across those uses.

If the first word is zero, this call is skipped. The remaining matching-path behavior is unchanged.

Nothing supplied establishes what F-8349c252c168 does with the value, whether it regards it as a pointer, whether it releases anything, or which memory it preserves.

### 3. Discriminator clearing and context selection

After the optional dependency returns, the function writes zero to the object’s second word.

It then calls **F-2fd8826557a4** with no explicit arguments.

Only after that call returns does the function load the context pointer from the external context-pointer location. Consequently:

- Effects of the earlier dependencies can affect which context is selected.
- The discriminator-clearing write precedes context selection.
- Subsequent direct accounting accesses use the selected context value; the external pointer is not explicitly reloaded later.

The static representation includes another null comparison of the original object pointer here. On ordinary execution, it is redundant: the earlier null check already returned, and the local pointer value has not been reassigned.

### 4. Header validation

The function considers the 32-bit word at object offset `−8`.

It invokes **F-a817d8d58c10** if either:

- The header word is zero; or
- The header word’s least-significant bit is set.

The zero test occurs first. The bit test is evaluated only if the zero test does not trigger. The representation contains separate header reads for these tests, not a guaranteed single immutable snapshot.

F-a817d8d58c10 is marked non-returning. The supplied function therefore provides no returning error result for these conditions.

Failure occurs after the discriminator has already been cleared and F-2fd8826557a4 has already run. An optional F-8349c252c168 call may also have completed. There is no rollback.

Passing validation establishes only the tested conditions at the respective reads. It does not establish a nonzero masked quantity, valid indices, sufficient counter balances, or header immutability.

### 5. Accounting preparation and first four updates

After validation succeeds, the function calls **F-c0fc6d033caa** with the selected context.

After that call returns, the direct operations occur in this order:

1. Load the unsigned 16-bit index at object offset `−4`.
2. Form the context address `0x224 + 4 × index`.
3. Read the 32-bit header word at offset `−8` and retain its value masked with `0x3ffffffc`.
4. Read the first indexed context field and write back that value minus the retained quantity.
5. Read the context field at `0x30c` and write back that value minus the same retained quantity.
6. Load the unsigned 16-bit index at object offset `−2`.
7. Read the indexed context field at `0xe0 + 4 × index` and write back that value minus the same retained quantity.
8. Read the context field at `0x220` and write back that value minus the same retained quantity.

The first index and accounting quantity are thus sampled after F-c0fc6d033caa, not before it. The second index is sampled after the first two accounting writes.

The mask retains bits 2 through 29. It clears the two lowest and two highest bits. The resulting quantity ranges from zero through `0x3ffffffc` and is a multiple of four.

Each 16-bit index ranges from 0 through 65,535. Multiplication by four produces a displacement from 0 through 262,140 before adding the context-relative offset.

### 6. Additional accounting and final header read

The function next calls **F-09dcb55e8e0a** with no explicit arguments.

After it returns, the function:

1. Uses its 32-bit integer result as an index into context-relative address `0x310 + 4 × result`.
2. Decrements that 32-bit field by one.
3. Decrements the 32-bit field at context offset `0x390` by one.
4. Reads the header word at object offset `−8` again.
5. Masks this fresh value with `0x3ffffffc`.
6. Subtracts the fresh masked quantity from the 32-bit field at context offset `0x08`.

There is no range check on the dependency-provided index. A negative or excessive result is not rejected.

The last subtraction does not reuse the earlier retained quantity. If intervening writes or dependency effects change the header, the quantity subtracted from `0x08` can differ from the quantity subtracted from the first four accounting fields.

### 7. Final dependency sequence

After all direct accounting writes, the function calls:

1. **F-cc6c9b4a0f68**, with the selected context and the header address.
2. **F-e586bff64fb7**, with the same selected context.

If both return normally, the function returns without a value. There is no direct object or header access after the first of these final calls.

## Side effects

### Direct writes

On a matching path that reaches normal completion, the function directly performs:

- One zero write to the object discriminator.
- Four context read-modify-writes subtracting a cached masked quantity.
- Two context read-modify-writes decrementing fields by one.
- One context read-modify-write subtracting a freshly sampled masked quantity.

These are ordered accesses, not a transaction. If addresses overlap, later accesses observe earlier writes subject to intervening dependency effects. The static representation does not justify treating the accounting fields as independent.

### Arithmetic

The accounting updates operate on 32-bit stored values. They contain no saturation, underflow detection, or overflow reporting. Their stored results follow the represented 32-bit word arithmetic; for example, decrementing zero yields the all-ones 32-bit value.

Context address calculations also use the represented 32-bit address arithmetic. Wrapped or out-of-range addresses do not become safe merely because the arithmetic produces a value.

### Ownership and synchronization

The optional initial call and final header-processing call suggest resource cleanup, but no ownership transfer or release guarantee is established. In particular:

- Passing the first object word does not prove that its referent is released.
- Passing the header address does not prove that the allocation is freed.
- Clearing the discriminator does not prove that the object remains valid for reuse.
- No dependency is known to preserve globals, object storage, headers, or context contents.

The placement of context-oriented dependencies may suggest synchronization. Their actual locking, unlocking, or other state effects are **documented_only**.

The direct operations contain no demonstrated atomic-update protocol. Concurrent mutation semantics are not established.

## Dependencies

| Opaque ID | Invocation | Established role |
|---|---|---|
| F-8349c252c168 | First object word and `4` | Optional call when the first-word test is nonzero |
| F-2fd8826557a4 | No explicit arguments | Runs after discriminator clearing and before context selection |
| F-a817d8d58c10 | No explicit arguments | Non-returning path for a zero header or set least-significant header bit |
| F-c0fc6d033caa | Selected context | Runs after header validation and before direct accounting updates |
| F-09dcb55e8e0a | No explicit arguments | Supplies the index used for one decrement |
| F-cc6c9b4a0f68 | Selected context and object-minus-eight address | Runs after direct accounting updates |
| F-e586bff64fb7 | Selected context | Last dependency on the normal path |

No dependency implementation is supplied. Return behavior, mutations, ownership effects, synchronization, and additional failures remain unknown except where explicitly stated above.

## Acceptance examples

These are conditional static expectations, not executed tests.

### Null argument

Given a null argument, the function returns without invoking any dependency or accessing the external discriminator or context.

### Discriminator mismatch

Given an accessible non-null object whose second word differs from the external discriminator, the function returns without changing that word. A missing or invalid preceding header is irrelevant because this path does not access it.

### Matching discriminator and zero first word

Given a discriminator match and a zero first object word, F-8349c252c168 is skipped. The discriminator is still cleared, and the remaining validation and accounting path proceeds.

### Matching discriminator and nonzero first word

Given a stable nonzero first word, F-8349c252c168 receives that value and `4`. If it returns normally, the discriminator-clearing write occurs afterward. This expectation does not assert what the dependency does.

### Invalid header after earlier side effects

Given a matching object and a header word that remains zero until validation, the function clears the discriminator, calls F-2fd8826557a4, selects the context, and invokes F-a817d8d58c10. It does not reach F-c0fc6d033caa or the direct accounting writes.

A stable header with its least-significant bit set follows the same failure destination.

### Valid header with zero accounting quantity

A stable header value of `2` passes the shown validation but masks to zero. Assuming valid distinct context addresses and normally returning dependencies, the quantity-based writes preserve their numeric values, while the two decrement operations still subtract one.

### Stable nonzero accounting quantity

Assume a stable header value of `0x1010`, first and second indices of `2` and `3`, and a F-09dcb55e8e0a result of `5`. Also assume valid, distinct target fields and no relevant dependency mutations.

The function subtracts `0x1010` from context offsets `0x22c`, `0x30c`, `0xec`, `0x220`, and `0x08`. It decrements fields at `0x324` and `0x390`.

### Header changed by the index-producing dependency

Assume the initially cached masked quantity is `0x1000`, and F-09dcb55e8e0a changes the header so its later masked value is `0x2000`. With otherwise valid, distinct addresses, the first four quantity-based updates subtract `0x1000`; the final update at context offset `0x08` subtracts `0x2000`.

### No counter-underflow rejection

Assume valid distinct fields, a subtraction quantity of `4`, and a target counter initially equal to `1`. Its stored 32-bit subtraction result is `0xfffffffd`; no shown branch rejects the insufficient balance.

## Uncertainties

The following remain **documented_only**:

- The discriminator’s value and meaning.
- The object’s semantic type and first-word interpretation.
- The allocation’s ownership and lifetime.
- The context’s complete layout and valid index ranges.
- Whether any accessed regions can alias in supported use.
- Dependency-side mutation, synchronization, release, and failure behavior.
- The valid range of the index returned by F-09dcb55e8e0a.
- Target endianness, alignment requirements, and fault behavior.
- Whether the non-returning error dependency terminates execution, transfers control, or uses another mechanism.

Names such as “header,” “context,” and “accounting” describe observed structural roles; they do not establish undocumented subsystem semantics.

## Evidence

The supplied static representation establishes the branch conditions, access widths and offsets, dependency order, mask, arithmetic updates, and non-returning error edge described above.

It does not include dependency bodies, external-state definitions, a complete memory model, or execution observations. Behavioral examples therefore state their environmental assumptions explicitly.

No runtime tests have been performed. No snapshot hash or independent provenance claim is asserted here.
