<!-- Trusted provenance: F-94aa0f9f3dc4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-94aa0f9f3dc4

## Purpose

F-94aa0f9f3dc4 conditionally processes entries associated with an input address, adjusts shared accounting, and, except for one recovered-address sentinel, validates allocation metadata and performs context-accounting updates followed by dependency calls.

The established contract is an ordered sequence of reads, writes, branches, and calls. Destruction, deallocation, and locking are plausible interpretations, but are **inferred intent**, not established ownership guarantees. In particular, the evidence does not establish that any dependency releases storage.

This specification is based on static evidence only. No runtime tests have been performed.

## Interface

### Input and result

The function accepts one 32-bit address-valued argument, designated **P**, represented as a signed integer.

- P equal to zero causes an immediate return.
- A nonzero P must permit every access reached on the selected execution path.
- There is no explicit return value.
- Invalid metadata causes a call to a dependency marked nonreturning.
- Other dependencies’ failure and return behavior are unknown.

Address calculations belong to the represented 32-bit environment. This contract does not establish behavior for wider pointers, invalid addresses, or signed arithmetic overflow.

### Input-associated layout

Offsets are in bytes.

| Location | Width | Use |
|---|---:|---|
| P + 0x18 | Signed 32 bits | Loop bound, read on every condition evaluation |
| P + 0x68 + 0x4c × entry index | 32 bits | Nonzero value triggers an entry-processing call |
| P − 4 | Signed 32 bits | Displacement subtracted from P |

Entry indices begin at zero. The stride does not establish the contents or size of a complete entry.

Let **R** be the address obtained by subtracting the displacement at P − 4 from P.

| Location | Width | Use |
|---|---:|---|
| R − 8 | Unsigned 32 bits | Shared-accounting adjustment amount, designated **A** |
| R − 0x10 | Unsigned 32 bits | Header validation and masked accounting amounts |
| R − 0x0c | Unsigned 16 bits | First context-table index |
| R − 0x0a | Unsigned 16 bits | Second context-table index |

The header and index fields are accessed only when R differs from 8. The amount at R − 8 is accessed even when R equals 8; that case therefore requires an access at address zero in the represented address space.

### External state

A shared-accounting pointer source is evaluated:

- Once on each loop-condition evaluation, including the terminating evaluation.
- Three separate times after A is read: once for the low-word source, once to capture the high-word source address, and once for the low-word destination.

These evaluations are not a single pointer snapshot. Their results need not be equal unless stability is an explicit precondition.

A separate external pointer source is dereferenced to obtain an allocator-context address, designated **C**, after F-2fd8826557a4 returns. C is retained for subsequent accesses and calls.

Initialization, lifetime, synchronization, and permissible mutation of these external sources are **documented_only** unknowns.

## Behavior

### 1. Null handling and entry traversal

For P equal to zero, return before any object-memory access, external-pointer read, write, or dependency call described here.

For nonzero P, begin with a signed entry index of zero. Each condition evaluation first reads the shared-accounting pointer, retaining that value, and then reads the signed count at P + 0x18. Enter the body only when the index is less than the freshly read count.

Within an entered iteration:

- Read the entry-associated word at P + 0x68 + 0x4c × index.
- If the word is nonzero, call F-94cb53877c15 with no explicit arguments.
- Advance the index by one.

The count and entry words are not snapshotted. A dependency’s changes can affect later iterations. A zero or negative count prevents entry-body accesses, but does not prevent the initial condition’s shared-pointer and count reads.

F-94cb53877c15 receives neither P nor the entry index explicitly. Its means of selecting or affecting an entry is unknown. The function itself does not clear an entry word.

The pointer retained from the final, terminating condition evaluation is designated **T**. T later selects the shared-accounting high-word destination.

There is no visible bounds validation. Every reached entry address must be valid, and signed index and address arithmetic must remain within the supported execution conditions.

### 2. Address recovery

After traversal, another check of P against zero appears. P has not been reassigned before that check, so it creates no additional reachable null case under ordinary sequential execution.

The function then reads the signed displacement at P − 4, subtracts it from P to obtain R, and reads A from R − 8.

The displacement’s broader encoding and allocation meaning are unknown. R equal to zero has no special handling.

### 3. Shared-accounting update

After reading A, the function performs three distinct evaluations of the shared-accounting pointer source. Their order and uses are:

1. Obtain a base designated **L**, then read the unsigned low word at L + 0x38. Retain this original low-word value.
2. Obtain a base designated **H**, then capture the address H + 0x3c. The high word is not yet read.
3. Obtain a base designated **W**, then write the original low word minus A to W + 0x38.

Only after that low-word write does the function read the 32-bit word through the captured address H + 0x3c. It adjusts that value and writes the result to T + 0x3c.

Thus four independently obtained bases participate:

| Base | When obtained | Role |
|---|---|---|
| T | Terminating loop-condition evaluation | High-word destination |
| L | First post-A pointer evaluation | Low-word source |
| H | Second post-A pointer evaluation | Captured high-word source address |
| W | Third post-A pointer evaluation | Low-word destination |

The high-word adjustment subtracts the sum of:

- The arithmetic sign extension of A interpreted as signed 32 bits: zero when its top bit is clear, or −1 when it is set.
- A borrow of one when the original unsigned low word read from L + 0x38 is less than A, otherwise zero.

The low-word subtraction and high-word result are 32-bit modular results. The borrow uses the original low-word source value, not a later read from the destination.

If T, L, H, and W identify the same stable accounting object, the pair implements subtraction of the sign-extended 32-bit amount A from a two-word quantity. It is not generally subtraction of a zero-extended unsigned A.

Without that stability condition, it is an ordered set of accesses potentially involving different objects. In particular:

- The low-word source and destination can differ.
- The high-word source and destination can differ.
- The high-word source address is captured before the low-word write, but its contents are read afterward.
- If the low-word write aliases the captured high-word source, the subsequent high-word read observes that write.
- No dependency call separates these shared-accounting accesses, but no synchronization guarantee establishes pointer stability.

The two writes are not shown as atomic.

### 4. Preparation and context capture

After both shared-accounting writes, call F-2fd8826557a4 with no explicit arguments.

Only after that call returns, dereference the separate external context-pointer source and retain C. The call may affect the value subsequently captured; preservation is not assumed.

If R equals 8, return after this context capture. This path has already:

- Read A.
- Performed shared accounting.
- Called F-2fd8826557a4.
- Read C.

It does not access the allocation header, validate metadata, update context counters, or perform the remaining dependencies.

### 5. Header validation

For R other than 8, examine the unsigned header at R − 0x10.

- If the first header read is zero, call F-a817d8d58c10.
- Otherwise, read the header again and call F-a817d8d58c10 if that read has its least significant bit set.

F-a817d8d58c10 is marked nonreturning. No rollback precedes this call.

The two validation reads are distinct. If the first read is nonzero and the second read becomes zero, the least-significant-bit test does not reject that second value. Validation must therefore not be described as testing one immutable header snapshot.

No other header bits are rejected. Validation is not repeated after subsequent dependencies.

### 6. Initial context-accounting updates

After validation, call F-c0fc6d033caa with C.

After that call returns, read the first unsigned 16-bit index from R − 0x0c, then read the header and mask it with 0x3ffffffc. Designate the resulting unsigned 32-bit amount **S**.

Perform these read-modify-write operations in order:

1. Subtract S from the 32-bit word at C + 0x224 + 4 × first index.
2. Subtract S from the 32-bit word at C + 0x30c.
3. Read the second unsigned 16-bit index from R − 0x0a.
4. Subtract S from the 32-bit word at C + 0xe0 + 4 × second index.
5. Subtract S from the 32-bit word at C + 0x220.

Both indices are zero-extended before multiplication by four. Their scaled range is zero through 0x3fffc; this arithmetic range does not establish that the corresponding table storage exists.

S excludes the header’s lowest two and highest two bits. It is a multiple of four between zero and 0x3ffffffc inclusive. Validation does not require S to be nonzero.

S is retained across these four subtractions. Changes to the header during these writes do not change that retained amount. Conversely, the second index is read after the first two writes, so aliasing can affect its value.

No index-range or underflow checks are visible.

### 7. Count updates and final amount

Call F-09dcb55e8e0a with no explicit arguments. Its result, designated **K**, is represented as signed 32 bits.

After it returns:

1. Decrement the signed 32-bit word at C + 0x310 + 4 × K by one.
2. Decrement the signed 32-bit word at C + 0x390 by one.
3. Read the header again, mask it with 0x3ffffffc, and subtract that newly obtained amount from the 32-bit word at C + 8.

There is no visible validation of K. It must select valid storage without unsupported signed multiplication or address overflow. Its sign is not independently checked.

The final masked amount can differ from S because the header is reread after F-09dcb55e8e0a and after the count decrements. The evidence does not require an order between the final subtraction’s counter-source read and header read.

### 8. Final calls and return

Call these dependencies in order:

1. F-cc6c9b4a0f68 with C and the address R − 0x10.
2. F-e586bff64fb7 with C.

Return if both calls return.

C is not reloaded from the external source. Whether F-cc6c9b4a0f68 preserves storage needed by F-e586bff64fb7 is an external requirement, not a guarantee established here.

## Side effects

Directly established effects comprise conditional entry-processing calls, two shared-accounting writes, ordered context-accounting writes, and additional dependency calls.

The function does not explicitly clear the caller’s pointer, erase the object, mark entries processed, validate table extents, or undo earlier effects after later failure.

Amount subtractions use 32-bit modular arithmetic. The two count decrements are represented as signed operations; portable signed-overflow behavior is not established. Valid execution must not rely on unsupported signed overflow in counts, index scaling, displacement subtraction, or address formation.

Aliasing is not excluded. Overlapping counters, metadata, external pointer storage, or shared-accounting fields can make earlier writes affect later reads. The ordered accesses govern such cases, not a model of independent counters.

Ownership transfer, locking, actual deallocation, and post-return object validity remain **documented_only**. Release-like accounting does not establish any of them.

## Dependencies

| Opaque ID | Established invocation | Unknown behavior |
|---|---|---|
| F-94cb53877c15 | No explicit arguments; on each encountered nonzero entry word | Entry selection, mutation, ownership |
| F-2fd8826557a4 | No explicit arguments; after shared accounting and before context capture | Synchronization, preparation, global changes |
| F-a817d8d58c10 | No explicit arguments; invalid-header path; nonreturning | Diagnostic or termination mechanism, further effects |
| F-c0fc6d033caa | C; after validation and before context metadata reads | Locking, mutation, ownership |
| F-09dcb55e8e0a | No explicit arguments; signed 32-bit result K | Result range, metadata and global changes |
| F-cc6c9b4a0f68 | C and R − 0x10 | Release, retention, mutation, context lifetime |
| F-e586bff64fb7 | C; after F-cc6c9b4a0f68 | Finalization, unlocking, ownership |

No dependency is assumed to preserve globals or metadata, or to acquire or release memory, without separate evidence.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Except where stated, assume valid addresses, supported arithmetic, returning dependencies, stable relevant memory, and no unintended aliasing.

1. **Null input.** P equal to zero returns without any described memory access or dependency call.

2. **No entries.** A nonzero P with a zero or negative count still causes one loop-condition shared-pointer read and one count read. No entry word is read; address recovery and accounting follow.

3. **Traversal and changing bounds.** A stable count of three with entry words zero, nonzero, nonzero causes two F-94cb53877c15 calls. If the first iteration instead triggers a call that changes the count to one, the next condition evaluation terminates traversal.

4. **Stable-base borrow.** If T, L, H, and W are equal, the low word is 3, the high word is 5, and A is 8, the low write stores 0xfffffffb, followed by a high write of 4.

5. **Sign-extension-sensitive amount.** With the same stable-base condition, low word zero, high word 5, and A equal to 0xffffffff, the writes produce low word 1 and high word 5. The −1 sign extension and borrow cancel in the high adjustment.

6. **Distinct accounting bases.** Suppose T, L, H, and W identify distinct, nonoverlapping objects; L’s low source is 3, H’s high source is 5, and A is 8. Write 0xfffffffb only to W + 0x38 and 4 only to T + 0x3c. Do not redirect either write to its source object.

7. **High-source aliasing.** Suppose the captured address H + 0x3c equals W + 0x38, the original low source is 3, and A is 8. The low write first stores 0xfffffffb at that address. The subsequent high read obtains 0xfffffffb, and the high destination receives 0xfffffffa after the borrow adjustment.

8. **Recovered address 8.** Given valid preceding accesses, including the amount read at address zero, R equal to 8 completes shared accounting, calls F-2fd8826557a4, reads C, and returns without header access.

9. **Invalid and changing headers.** A first header read of zero immediately selects F-a817d8d58c10. A nonzero first read followed by an odd second read also selects it. A nonzero first read followed by zero does not fail the second test.

10. **Zero masked amount.** A stable header of 2 passes validation and yields S equal to zero. Size subtractions still perform their accesses and writes; both count decrements and final calls still occur.

11. **Header change across a dependency.** If the initial masked amount is 16 and F-09dcb55e8e0a changes the header’s masked amount to 32, the final C + 8 subtraction uses 32, provided no intervening alias changes it again.

## Uncertainties

The following remain **documented_only**:

- Object type, entry meaning, displacement encoding, and header flag semantics.
- Valid index ranges, table extents, and permitted negative values of K.
- Whether shared accounting represents an intended signed 64-bit quantity.
- Pointer stability, synchronization, and concurrent-access rules.
- Dependency failure, ownership, locking, and deallocation behavior.
- Whether counter wraparound is acceptable to consumers.
- Byte order, alignment requirements, and behavior outside valid-address and supported-arithmetic preconditions.
- The target’s signed-right-shift rules; the described sign-extension effect follows the represented arithmetic-right-shift interpretation.

No unknown ownership or synchronization property is promoted to a guarantee.

## Evidence

The supplied static representation establishes the branches, represented widths, offsets, mask, dependency argument shapes, and access sequencing described above. In particular, it separately evaluates the shared-accounting pointer three times after reading A, captures the high-word source address before the low-word write, reads that high source afterward, and uses the pointer retained from the terminating loop condition for the high-word destination.

It also establishes separate short-circuit header reads, metadata rereads after dependencies, and the nonreturning designation of F-a817d8d58c10.

Dependency implementations, runtime observations, ownership documentation, and concurrency guarantees were not supplied. No runtime tests have been performed.
