<!-- Trusted provenance: F-3717a8bcbcf7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3717a8bcbcf7

## Purpose

F-3717a8bcbcf7 conditionally performs a manager-like initialization sequence when a shared 32-bit state slot contains zero. The sequence obtains prerequisites, prepares a named resource, constructs an object, invokes an activation dependency, performs a registration-like operation, and finally writes a fixed value into the object currently referenced by that shared slot.

If the slot is already nonzero, the function skips initialization.

The apparent intent is lazy initialization of a block-manager thread. That interpretation is inferred from the fixed label **iMABlockManagerThread** and the dependency sequence. The supplied body does not establish the meanings of the created resource, the shared slot, or the registration operation.

The return value is a 64-bit composite, not simply a Boolean. Its upper 32 bits reflect a scratch word that changes during initialization; its lower 32 bits distinguish normal completion from one specific null-object outcome.

## Interface

### Inputs

No ordinary parameters are declared in the supplied representation. The function nevertheless depends on:

- A shared slot containing a 32-bit pointer or pointer-like value.
- A global 32-bit fallback value used during named-resource preparation.
- A global argument supplied to a registration-like dependency.
- The fixed label **iMABlockManagerThread**.
- An incoming machine-register value, captured into a local word on the initialization path.

The incoming register value is not established as a public argument. Its initial value can be presented through an output-address argument during name preparation. On the zero-length path, the function overwrites it before passing that address to another dependency.

### Output

The established return construction is:

| Path | Upper 32 bits | Lower 32 bits |
|---|---:|---:|
| Shared slot initially nonzero | 0x00080000 | 1 |
| Initialization reaches the final write | 0x00003000 | 1 |
| Derived object address is zero, label measure is zero | 0x00080000 | 0 |
| Derived object address is zero, label measure is nonzero | 0x00000020 | 0 |

These values describe the explicit local assignments in the supplied body. They are not a guarantee against undocumented dependency writes into caller-owned storage.

Two dependency paths are marked non-returning. They produce no ordinary return from this function.

### Widths and local layout

The supplied representation uses 32-bit integers and pointer-like values for addresses, dependency results, slot contents, and object fields. The composite result is 64 bits.

Three local words occupy consecutive four-byte positions:

| Position relative to the first word | Role |
|---|---|
| +0 | Scratch word, later used as the return value’s upper half |
| +4 | Companion scratch word |
| +8 | Named-resource word whose address is passed to dependencies |

This adjacency is established. A dependency’s interpretation of the three words as one descriptor is not established.

Pointer additions and subtractions are represented at 32-bit width. The body contains no explicit overflow, alignment, or address-validity checks. Validity of the resulting addresses is a caller/environment precondition rather than an enforced guarantee.

## Behavior

### 1. Initial state check

The function first retains the address of the shared slot. It initializes the first scratch word to 0x80000 and the companion word to 1, then reads the slot.

If the slot is nonzero, it immediately returns 0x0008000000000001. No initialization dependency is called, and no object field is written on this path.

If the slot is zero, initialization proceeds. There is no visible lock, atomic claim, second zero check, or rollback protocol.

### 2. Prerequisite acquisition

The function captures the incoming register value into the named-resource word and calls F-e1d80b9aa375 without explicit arguments.

It then calls F-fdb517b65f69.

- If that result equals signed 32-bit −1, it calls the non-returning F-a817d8d58c10.
- Otherwise, it calls F-cce4c42e04c6 with 0x31.
- If that result equals unsigned 32-bit 0xffffffff, it calls the same non-returning dependency.

These comparisons use the same all-ones bit pattern, with different signedness in the supplied representation.

The second prerequisite result is retained for use during activation. The first prerequisite result is used only for its failure comparison in the visible body.

The second prerequisite call is skipped when the first prerequisite reports failure.

### 3. Named-resource preparation

The function calls F-1cb585febd84 with the fixed label. Its result is treated as an unsigned 32-bit measure.

After that call, the function reads the global fallback value. This read occurs even when the measure is nonzero and the fallback will be replaced.

If the measure is zero:

- F-fe78e25e82ff is not called.
- The fallback value becomes the base used for the next step.
- The first scratch word remains 0x80000.
- The companion scratch word remains 1.

If the measure is nonzero:

- The first scratch word becomes 0x20.
- The companion scratch word becomes the measure.
- F-fe78e25e82ff receives the address of the named-resource word, the unsigned maximum of 0x20 and the measure, and the measure itself.
- Its return value becomes the base used for the next step.

At a measure of exactly 0x20, the first scratch word supplies the maximum argument. Above 0x20, the companion word supplies it. The argument’s numeric value is the maximum in either case.

The function then adds 0x0c to the selected base and stores the resulting 32-bit address in the named-resource word.

An unidentified external dependency receives, in order:

1. That adjusted address.
2. The fixed label.
3. The unsigned measure.

This call also occurs when the measure is zero. No result from it is visibly checked. Copying or resource-name installation is plausible intent, but its actual behavior is unknown.

### 4. Allocation and construction

The function calls F-8157a180d901 with 0x44, requesting a result subsequently treated as the base of a 68-byte object.

Before checking that result for any failure indication, the function performs these writes:

| Offset from returned base | Write |
|---|---|
| +0x00 | Four-byte zero |
| +0x04 | One-byte zero |
| +0x08 | Four-byte zero |

The write at +0x04 does not establish the values of bytes +0x05 through +0x07. No clearing of the remaining allocation is visible.

Construction then proceeds through two dependencies:

1. F-72538fc0941d receives the allocation base plus 0x0c.
2. F-fe1f2d17b425 receives the first constructor result plus 0x1c.

The function subtracts 0x28 from the second constructor result to derive an object address.

The body does not establish that either constructor returns its input. Consequently, the derived object address must not be assumed to equal the original allocation base.

The subsequent null check applies to this derived address, not to the allocator result. In particular, a second constructor result of 0x28 produces the null-object branch.

### 5. Activation and failure handling

If the derived object address is nonzero, the function replaces the first scratch word with 0x3000 and the companion word with the retained result from F-cce4c42e04c6.

It then calls F-2284e043d377 with:

1. The derived object address.
2. Zero.
3. The address of the named-resource word.
4. Zero.

At this call boundary, the two preceding adjacent local words contain 0x3000 and the retained prerequisite result. Whether the dependency accesses those preceding words is unknown.

A zero activation result triggers this ordered sequence:

1. F-985b9dd6c5ae receives the second constructor result.
2. F-fd7c65715716 receives the preceding result minus 0x1c.
3. F-feee9156a257 receives the preceding result minus 0x0c and does not return.

These calls resemble reverse construction and disposal, but their actual ownership effects are not established. The body does not call F-9088f1254fb8 before this non-returning path.

A nonzero activation result continues to registration.

### 6. Null-object completion

If the derived object address is zero, activation is skipped.

The function calls F-9088f1254fb8 with the address of the named-resource word, then returns a 64-bit value whose lower half is zero and whose upper half is the current first scratch word.

On this path:

- F-c80ddf4b1ba5 is not called.
- The final shared-object field write is not performed.
- No explicit allocation-release call is visible.
- The first scratch word has not been changed to 0x3000.

The absence of a visible release does not prove a leak: constructor ownership and dependency behavior are unknown.

### 7. Registration and successful completion

Following a nonzero activation result, the function:

1. Calls F-c80ddf4b1ba5 with the global registration argument.
2. Calls F-9088f1254fb8 with the address of the named-resource word.
3. Reloads the shared slot through its retained address.
4. Writes the four-byte value 0x80000 at offset +0x30 from the slot’s current contents.
5. Returns 0x0000300000000001 under the explicit local assignments.

The final store does not use the derived object address directly. It uses the shared slot’s value after both intervening dependencies have returned.

No explicit assignment to the shared slot appears in this body. Successful completion therefore requires some external state transition to make that slot reference writable storage before the final store.

## Side effects

Directly established effects are:

- Reads of the shared slot, fallback global, and registration argument.
- Three partial initialization writes into the allocation result.
- Changes to local scratch and named-resource words.
- A final four-byte store at offset +0x30 from the shared slot’s reloaded value on successful initialization.
- Dependency calls in the order specified above.

Dependency effects may include allocation, construction, activation, registration, and resource release, but those meanings are not independently established.

The named-resource word is passed by address to several dependencies. They may modify it according to contracts not supplied here. F-9088f1254fb8 receives its current value indirectly, not a guaranteed copy of its earlier value.

No assumption is made that dependencies preserve globals. In particular, the final shared-slot contents need not equal the value observed at entry.

The body establishes neither thread safety nor idempotence under concurrent calls. Two callers could both observe zero unless an external mechanism prevents that condition.

## Dependencies

| Opaque ID | Visible contract at this call site |
|---|---|
| F-e1d80b9aa375 | Called once before prerequisite checks; no explicit arguments or inspected result. |
| F-fdb517b65f69 | Supplies the first prerequisite result; −1 selects the fatal path. |
| F-cce4c42e04c6 | Receives 0x31; all-ones selects the fatal path; otherwise its result is retained. |
| F-a817d8d58c10 | Non-returning prerequisite-failure operation. |
| F-1cb585febd84 | Receives the fixed label; supplies an unsigned measure. |
| F-fe78e25e82ff | Called only for a nonzero measure; receives an output address, a maximum, and the measure; returns a base. |
| Unmapped external dependency | Receives the adjusted base, fixed label, and measure; no inspected result. No opaque ID was supplied. |
| F-8157a180d901 | Receives 0x44; returns storage immediately written by this function. |
| F-72538fc0941d | First construction stage, called at allocation offset +0x0c. |
| F-fe1f2d17b425 | Second construction stage, called at first-result offset +0x1c. |
| F-2284e043d377 | Activation-like operation; zero selects non-returning failure handling. |
| F-985b9dd6c5ae | First reverse-construction-like operation on activation failure. |
| F-fd7c65715716 | Second reverse-construction-like operation on activation failure. |
| F-feee9156a257 | Final activation-failure operation; non-returning. |
| F-c80ddf4b1ba5 | Receives the global registration argument after successful activation. |
| F-9088f1254fb8 | Receives the named-resource word by address on null-object and successful paths. |

## Acceptance examples

These are conditional static acceptance cases, not executed tests. They assume dependencies return as stipulated, supplied addresses are valid where dereferenced, and dependencies do not make undocumented writes to the scratch words.

1. **Already initialized.** Given a nonzero shared slot at entry, the result is 0x0008000000000001. No dependency is called.

2. **First prerequisite fails.** Given a zero shared slot and a first prerequisite result of −1, F-a817d8d58c10 is called. F-cce4c42e04c6 and all later initialization operations are skipped.

3. **Second prerequisite fails.** Given a successful first prerequisite and an all-ones second result, the second dependency receives 0x31, followed by F-a817d8d58c10. Label processing does not occur.

4. **Nonzero measure below the boundary.** Given a measure of 7, F-fe78e25e82ff receives a maximum argument of 32 and a measure argument of 7.

5. **Measure above the boundary.** Given a measure of 33, both numeric size arguments to F-fe78e25e82ff are 33. The scratch word used for the return’s upper half remains 32 until activation is attempted.

6. **Zero measure and null derived object.** F-fe78e25e82ff is skipped; the adjusted fallback address is passed to the unmapped dependency with measure zero. After construction produces a zero derived address, F-9088f1254fb8 runs and the result is 0x0008000000000000.

7. **Nonzero measure and null derived object.** After name preparation, a zero derived address leads to F-9088f1254fb8 and result 0x0000002000000000. Registration and the final field store are skipped.

8. **Activation fails.** A nonzero derived address and zero activation result cause the three failure-handling dependencies in their specified order. There is no ordinary return.

9. **Successful initialization with slot replacement.** If registration or named-resource finalization changes the shared slot to another valid object, the final +0x30 store targets that replacement object. The body does not force the store to target the allocation or derived object.

## Uncertainties

The following remain **documented_only**:

- The public calling convention and significance of the incoming register value.
- The units and semantic meaning of the label measure.
- The unmapped external dependency’s identity and behavior.
- The meaning of the fallback base and its +0x0c adjustment.
- Whether the adjacent local words form a descriptor consumed by any dependency.
- Allocation failure conventions and constructor ownership transfers.
- Whether reverse-construction-like calls release storage.
- Whether F-9088f1254fb8 releases, resets, or otherwise modifies the named resource.
- Which dependency populates the shared slot.
- The meaning of the final +0x30 field and its value 0x80000.
- Synchronization supplied outside this body.
- Actual behavior when a required address is null, invalid, misaligned, or produced by overflowing address arithmetic.

No recoverable error protocol is visible for invalid storage or a still-zero shared slot at the final write.

## Evidence

This contract is based solely on the supplied static function body and dependency mapping. Established claims concern explicit branches, argument values, arithmetic widths, direct memory accesses, call ordering, and return construction.

Names such as “manager,” “activation,” “registration,” and “reverse construction” express inferred intent, not verified dependency semantics. Embedded comments do not establish additional reachable behavior.

No dependency implementations, external documentation, or runtime observations were supplied. No runtime tests have been performed.
