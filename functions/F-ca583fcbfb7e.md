<!-- Trusted provenance: F-ca583fcbfb7e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ca583fcbfb7e

## Purpose

F-ca583fcbfb7e checks one entry in a table when a numeric selector falls within an admitted range. An excluded selector causes normal return without a table access. An admitted selector causes a single entry to be read: zero permits normal return; any nonzero value triggers F-282b4cfec88b and, if that dependency returns normally, F-feee9156a257.

The apparent purpose is to enforce a zero-entry invariant. That purpose is **inferred**, not established. The available evidence does not identify the application domain, explain the entry values, or establish what either dependency does.

The selector’s pointer-shaped interface does not make this a null-pointer check. Its numeric interpretation selects an entry, and selector zero can require a table read.

## Interface

### Inputs

**Record address.** The first argument identifies storage containing two relevant fields:

| Byte offset | Width under the apparent target model | Interpretation |
|---|---:|---|
| 0 | 4 bytes | Integer value used to form the table address |
| 4 | 4 bytes | Signed exclusive upper bound for the selector |

No other record fields are accessed. The function does not establish the record’s total size or validate its storage.

**Selector.** The second argument has a pointer-shaped representation but is used numerically. Its signed integer interpretation determines range admission and the selected entry’s byte displacement. Storage designated by the selector is not directly dereferenced as an input object.

The selector is passed by value. The function provides no mechanism for returning a changed selector or record address.

### Widths and representation

The supplied representation supports an apparent 32-bit integer and address model:

- The relevant record fields are separated by four bytes.
- Each table entry occupies four bytes.
- Selector sign and ordering are evaluated through signed integer interpretations.
- Entry selection uses a four-byte stride.
- The selected entry is distinguished only as zero or nonzero.

This contract uses that apparent model for concrete offsets and numeric examples. It does not establish a portable interface for wider pointers, a different integer width, or different pointer-to-integer conversion rules. Byte order and alignment requirements are **documented_only**.

The field at offset zero is not independently validated as a pointer. Its integer contents participate in address formation before the selected entry is read.

### Outputs and completion

There is no explicit return value or output parameter. The established outcomes are:

- Normal return after excluding the selector.
- Normal return after reading a zero entry.
- Entry into the ordered dependency sequence after reading a nonzero entry.

Invalid memory or unsupported arithmetic can prevent those outcomes. There is no defined error code or recovery result for such conditions.

F-feee9156a257 is identified as non-returning in the supplied material. Its actual termination or control-transfer mechanism is unknown. Normal continuation after that call is not an established outcome.

### Conditional storage requirements

The required readable storage depends on the selector:

- **Negative signed selector:** neither record field nor any table entry is required by the established access path.
- **Nonnegative signed selector:** the bound field must be readable.
- **Admitted selector:** the base field and the selected four-byte entry must also be readable, and address formation must have supported semantics.

A pointer-shaped selector need not identify readable storage merely because of its interface type. Conversely, selector zero does not remove the record or table storage requirements.

## Behavior

### Range admission

The admitted range consists of selectors whose signed interpretation is nonnegative and strictly less than the signed bound.

A negative selector is excluded without a record read. For a nonnegative selector, the bound is read before admission is decided. Equality with the bound is excluded; the bound is exclusive.

Consequences:

- A bound of zero admits no selector.
- A negative bound admits no selector.
- A positive bound admits selector zero.
- The largest admitted selector for a positive bound is one less than that bound.
- An excluded selector does not require the table-base field or a table entry to be read.

There is no separate zero-selector exemption. With a positive bound, zero selects the first entry.

### Entry selection and result

For an admitted selector, the entry address is formed from the integer base field plus a byte displacement of four times the selector’s signed integer value. One four-byte integer is read at the resulting address.

A zero entry causes normal return without invoking either dependency.

Every nonzero entry has the same control-flow consequence. Positive and negative values are not treated differently, and the entry’s magnitude has no established significance.

The function does not scan the table, inspect neighboring entries, clear the selected entry, or repeat the check.

### Logical access and call order

For an admitted selector, the established order is:

1. Read the record’s bound field.
2. Read the record’s base field.
3. Read the selected table entry.
4. If that entry is nonzero, invoke F-282b4cfec88b.
5. If that invocation returns normally, invoke F-feee9156a257.

For an excluded nonnegative selector, only the bound read is required. For a negative selector, none of these memory reads occurs.

This is the logical ordering established by the supplied function representation, not a processor-level memory-ordering guarantee. No synchronization, atomic access, or coherent snapshot contract is established.

The function does not recheck the selector, bound, base, or entry between the two dependency calls. Therefore, if F-282b4cfec88b changes the selected entry to zero and returns normally, F-feee9156a257 is still invoked. If the first dependency does not return normally, the second call is not reached through the established sequence.

### Arithmetic limits

Range admission does not establish safe address formation.

Under signed 32-bit arithmetic, the largest nonnegative selector whose multiplication by four remains representable is 536,870,911. Its displacement is 2,147,483,644 bytes. Selector 536,870,912 has a mathematical displacement of 2,147,483,648 bytes, exceeding the positive signed 32-bit limit.

Both selectors can satisfy the range condition when the bound is sufficiently large. The function has no separate multiplication-overflow rejection.

Even when the displacement is representable, adding it to the integer base can exceed the signed integer range. A numerically representable result also does not prove that the converted address is aligned, mapped, or inside an accessible allocation.

The supplied evidence does not resolve machine wrapping versus source-level overflow semantics, nor all integer-to-pointer conversion behavior. Overflowing or otherwise unsupported address formation is **documented_only**: neither successful lookup nor graceful rejection is promised.

## Side effects

The established direct effects are conditional memory reads and, for a nonzero admitted entry, dependency calls.

There are no explicit stores to the record, table, or global state. There is no explicit allocation, deallocation, ownership transfer, diagnostic output, or resource operation.

On excluded-selector and zero-entry normal-return paths, the function does not directly modify caller storage. This does not imply that concurrent agents cannot modify that storage.

On the nonzero-entry path, dependency effects remain unknown. Either dependency might modify shared state, produce external output, release memory, acquire resources, terminate execution, or transfer control nonlocally. None of those effects is established, and none is ruled out.

In particular:

- The lack of explicit call arguments does not establish the absence of hidden inputs or global-state access.
- No preservation guarantee is available for the record or table after a dependency call.
- No ownership or lifetime guarantee can be inferred for either dependency.
- The absence of further table reads does not imply that dependencies leave the table intact.

The function has no explicit record-null check, base-null check, allocation-size validation, or invalid-read recovery. If a required read fails, subsequent reads or calls need not occur. The exact failure mechanism is **documented_only**.

## Dependencies

### F-282b4cfec88b

**Invocation condition:** An admitted entry has been read and found nonzero.

**Position:** First in the dependency sequence.

**Call interface:** No explicit arguments are supplied, and no result is consumed by F-ca583fcbfb7e.

**Continuation:** Normal return leads to F-feee9156a257 without another table check. Failure to return normally prevents that second call through the established control flow.

Its purpose, implicit inputs, calling convention details, memory effects, ownership effects, resource behavior, and failure modes are **documented_only**.

### F-feee9156a257

**Invocation condition:** F-282b4cfec88b has returned normally after the nonzero-entry decision.

**Position:** Second in the dependency sequence.

**Call interface:** No explicit arguments are supplied, and no result is consumed.

**Completion:** The supplied material identifies it as non-returning. This does not establish whether it exits, aborts, throws, transfers control elsewhere, or uses another mechanism.

Its implementation and side effects are unavailable. Behavior following an unexpected normal return is **documented_only**.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Unless stated otherwise, they assume the apparent 32-bit representation, stable readable storage, supported address conversions, and representable address arithmetic.

| Conditions | Required behavior |
|---|---|
| Selector is −1 and the record address is null. | Return normally without reading the record or invoking either dependency. |
| Selector is zero and the bound is zero. | Read the bound and return; do not read the base field or a table entry. |
| Selector is zero and the bound is −1. | Read the bound and return without table access. |
| Selector is zero, the bound is one, and the first entry is zero. | Read the bound, base, and first entry in that order; return without dependency calls. |
| Selector is zero, the bound is one, and the first entry is nonzero. | Perform those reads, invoke F-282b4cfec88b, and invoke F-feee9156a257 if the first call returns normally. |
| Selector is three and the bound is three. | Read the bound and return without reading the base or entry three. |
| Selector is four and the bound is three. | Read the bound and return without table access. |
| Selector is two, the bound is three, and the selected entry is zero. | Read the entry eight bytes beyond the base and return normally. |
| Selector is two, the bound is three, and the selected entry is −1. | Enter the same dependency sequence as for any other nonzero entry. |

Additional conditional cases:

- **Unreadable record with a nonnegative selector:** a successful normal return is not guaranteed; the bound read is required.
- **Excluded selector with an unusable table-base value:** after a successful bound read, the base field and selected entry are not accessed.
- **Positive bound with a zero base and selector zero:** admission still occurs. The function does not reject the resulting address before attempting the required entry read.
- **First dependency changes the entry to zero and returns:** the second dependency is still invoked.
- **First dependency does not return normally:** the second dependency is not reached through this sequence.
- **Selector 536,870,912 with a larger signed bound:** range admission succeeds, but the four-byte displacement exceeds positive signed 32-bit range. The resulting execution behavior remains **documented_only**.
- **Representable displacement but unreadable selected address:** no defined error result or recovery is established.

## Uncertainties

**Inferred intent.** The bound resembles a table length, and a nonzero entry appears to trigger a failure or invariant-handling path. Neither interpretation establishes application meaning or dependency behavior.

**Target semantics — documented_only.** The exact ABI, pointer conversion rules, byte order, alignment requirements, overflow behavior, and invalid-access behavior are not independently established.

**Storage validity — documented_only.** There is no evidence that the bound corresponds to the actual allocation extent. Admission alone does not establish a valid entry.

**Ownership — documented_only.** The function performs no explicit ownership operation, but the dependencies’ ownership and lifetime effects are unknown.

**Concurrency — documented_only.** No synchronization or consistency guarantee connects the bound, base, and entry reads. Behavior under concurrent mutation is not established.

**Exceptional continuation — documented_only.** Dependency failures, nonlocal transfers, and unexpected return from the dependency marked non-returning have no established recovery contract.

These uncertainties do not imply additional validation, retries, cleanup, or preservation guarantees.

## Evidence

The established contract rests solely on the supplied static function representation:

- Negative signed selectors bypass record access.
- Nonnegative selectors require the bound read.
- Admission requires a nonnegative selector strictly below the signed bound.
- Admitted lookups use the base field and four-byte entry spacing.
- Zero entries return without dependency calls.
- Nonzero entries trigger the stated call order without an intervening recheck.
- No explicit stores or ownership operations are present.

Application meaning and unavailable dependency behavior are not established by that evidence. No implementation execution, runtime tests, dependency inspection, or external verification has been performed.
