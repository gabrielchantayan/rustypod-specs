<!-- Trusted provenance: F-8163ddbc7a0d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8163ddbc7a0d

## Purpose

F-8163ddbc7a0d retains an address for a possible later update, separately obtains an initial word snapshot, waits until a polling expression observes bit 1 set, and invokes two dependencies in order. The second dependency’s unsigned result determines whether the function updates a word at the retained address.

On every shown normal-return path, the function returns the initial snapshot. That snapshot is not necessarily read from the eventual update target.

Three address roles must remain distinct:

- **Retained update address:** obtained from the shared pointer location first; used for the conditional post-call read and write.
- **Initial snapshot address:** obtained by separately evaluating the shared pointer location after the update address has been retained; used for the word value returned on normal completion.
- **Polling addresses:** obtained through the shared pointer expression during polling; used to read word index 7.

These addresses may coincide, but the represented behavior does not require them to coincide.

Interpreting this as a hardware-readiness wait followed by control-register configuration is inferred intent. The established behavior is pointer evaluation, word access, bit testing, ordered dependency calls, and a conditional masked update.

## Interface

### Inputs

There are no explicit parameters. Inputs come from shared state and a dependency result:

- The shared pointer value when the update address is retained.
- A separately evaluated shared pointer value that supplies the initial snapshot address.
- The first word at the initial snapshot address.
- Word index 7 at each address used by the polling expression.
- The unsigned result returned by F-2a818e25cf9e.
- The first word at the retained update address after both dependency calls, but only if the result selects an update.

The initial snapshot is not expressed as a read through the retained update address. A change to the shared pointer between those two evaluations can therefore make the snapshot source differ from the eventual update target, subject to the execution model’s visibility rules.

### Output

The normal return value is the word obtained by the initial snapshot read.

It is not:

- The retained update address.
- The original contents of the update target, unless those happen to equal the snapshot.
- The post-call contents of the update target.
- The replacement value written by this function.
- A success or timeout status.

Later shared-pointer changes, memory changes, dependency effects, and the function’s conditional write do not replace the retained return value.

### Width and layout model

The supplied representation uses unsigned word values. A 32-bit unsigned-word model is consistent with the masks and constants, but the supplied material does not independently establish the target’s exact integer widths or ABI.

The numeric acceptance examples use that 32-bit model.

| Address role | Accessed element | Byte offset under the 32-bit model |
|---|---|---:|
| Initial snapshot address | Word index 0, read once | 0 |
| Each polling address | Word index 7, read for the condition | 28 |
| Retained update address | Word index 0, conditionally read and written | 0 |

The polling displacement is seven word elements. Its expression as 28 bytes depends on four-byte words.

No byte-order conversion, byte-sized access, signed reinterpretation, or multiword transfer is expressed. The selector comparison is unsigned. The mask-and-combine operation is bitwise, not arithmetic addition.

### Preconditions and progress requirements

For a normal completion through the shown path:

- The shared pointer location must be readable whenever evaluated.
- The separately obtained initial snapshot address must permit the initial word read.
- Every address actually used by the polling expression must permit the word-index-7 read.
- At least one polling evaluation must observe bit 1 set.
- Both dependencies must return normally.
- If an update is selected, the retained update address must permit the post-call word read and write.

The retained update address need not permit an initial word read merely because it was retained: the initial snapshot read uses the separately obtained snapshot address. If the selector suppresses the update, this function does not dereference the retained update address.

These are access and progress requirements, not checked conditions. There is no explicit null, alignment, bounds, lifetime, or readiness-timeout check.

## Behavior

### Address retention and initial snapshot

First, the function evaluates the shared pointer location and retains its value as the update address.

Next, it evaluates the shared pointer location separately and reads the first word at the resulting snapshot address. It retains that word as the normal return value.

These operations establish two independent retained facts:

- An address for any later direct update.
- A word value for the eventual return.

They do not establish that the retained update address and initial snapshot address are equal. Equality follows only under an additional condition, such as the shared pointer remaining unchanged and its evaluations observing the same value.

After the snapshot has been read, changes to its source storage do not change the retained return value. Changes to the shared pointer do not retarget the retained update address.

### Polling condition

After taking the snapshot, the function tests bit 1 of word index 7 reached through the shared pointer expression. It repeats the test while that bit is clear.

At least one polling evaluation occurs. Readiness on entry does not eliminate the initial snapshot or the first polling evaluation.

The tested mask is `0x00000002`:

- `0x00000000` does not permit progress.
- `0x00000001` does not permit progress.
- `0x00000002` permits progress.
- Any other word with bit 1 set also permits progress.

Other bits do not affect the condition.

There is no explicit delay, timeout, retry bound, yield, cancellation check, or error branch. If bit 1 is never observed set, the shown path remains in the polling loop; neither dependency is reached and no normal return occurs.

Polling uses the shared pointer expression, not the retained update address or a retained snapshot address. Under an execution model that observes shared-pointer changes, different polling evaluations may therefore access different underlying storage.

The representation establishes repeated evaluation at its behavioral level. It does not establish volatile qualification, synchronization, exact machine-level load counts, or visibility of asynchronous changes.

### Ordered dependency calls

Once a polling evaluation observes bit 1 set, the function invokes an external dependency whose opaque ID is unavailable in the supplied map. Its explicit argument is the integer value 1.

Only after that call returns does the function invoke F-2a818e25cf9e, with no explicit arguments shown.

The first call’s return value is not used by the visible decision logic. F-2a818e25cf9e supplies the unsigned selector.

Neither dependency is assumed to preserve the shared pointer, preserve either addressed word, maintain the retained address’s lifetime, or leave other state unchanged.

### Selector classification

The selector determines the following direct behavior:

| Unsigned selector | Subsequent direct behavior |
|---|---|
| 0 or 1 | Read the retained update target, combine its masked value with `0x00000C20`, and write the result |
| 2 or 3 | Read the retained update target, combine its masked value with `0x00000DA8`, and write the result |
| Any other value | Return the initial snapshot without a post-call target read or write |

Classification applies to the unsigned selector itself. There is no expressed truncation to a smaller selector field. Under a 32-bit model, `0xFFFFFFFF` is in the no-update class.

The no-update class does not mean that nothing has happened. The snapshot read, polling, and both calls have already occurred, and dependency effects remain unconstrained.

### Conditional target update

For selectors 0 through 3, the function reads the first word at the retained update address after both dependencies have returned.

It retains only the bits selected by `0x80000007` from that later word and combines them with the selected replacement pattern.

Under the 32-bit model:

- Bit 31 and bits 0–2 come from the post-call target read.
- Every other bit is determined by the replacement pattern.
- Pattern `0x00000C20` sets bits 5, 10, and 11.
- Pattern `0x00000DA8` sets bits 3, 5, 7, 8, 10, and 11.
- All remaining non-preserved bits are cleared.

Neither replacement pattern overlaps the preserved bits.

The preserved bits do not come from the initial snapshot. They come from the later read at the retained update address, even when the snapshot was taken from different storage.

The function performs one direct word write for the selected update. Bit masking and bitwise combination introduce no arithmetic carry or overflow. Exact-width interpretation beyond the conditional 32-bit model remains documented_only.

### Return

After a selected write, or after selecting the no-update path, the function returns the initial snapshot.

There is no post-write readback, verification, additional polling, or recovery operation in the supplied body.

## Side effects

The represented direct access and call order is:

1. Read the shared pointer location to retain the update address.
2. Separately evaluate the shared pointer location to obtain the snapshot address.
3. Read the first word at the snapshot address and retain that value.
4. Evaluate the polling expression one or more times, reading word index 7 through the shared pointer expression.
5. Invoke the unmapped external dependency with argument 1.
6. Invoke F-2a818e25cf9e.
7. For selectors 0 through 3, read the first word at the retained update address.
8. For those same selectors, write the replacement word at that address.
9. Return the retained snapshot.

This sequence distinguishes represented evaluations from any particular compiled load sequence.

The function does not explicitly write the shared pointer location or the polling word. It does not explicitly allocate, release, acquire ownership of, or transfer ownership of memory. Dependency behavior is not constrained by those observations.

The target update is a separate read followed by a write. No lock, atomic read-modify-write operation, interrupt exclusion, or other interference protection is shown. A change between the target read and write may be overwritten if the environment allows such interference.

The retained address has no visible lifetime protection. If a dependency invalidates it, the update path contains no shown validity check or repair mechanism.

If the storage is device-mapped, accesses may have effects beyond ordinary memory reads and writes. Read-to-clear behavior, write restrictions, and similar device semantics are not established here.

## Dependencies

### F-2a818e25cf9e

- Called once after the preceding external dependency returns.
- Receives no explicit arguments in the supplied representation.
- Returns the unsigned selector used by this function.
- Has no established state-preservation, ownership, or lifetime guarantees.
- Has an unknown purpose and full return-value domain.

### External dependency with unavailable opaque ID

The supplied dependency map does not identify this call with an opaque function ID; no ID is invented.

- Called once after a polling evaluation observes bit 1 set.
- Receives the explicit integer argument 1.
- Must return before F-2a818e25cf9e is invoked.
- Has no return value used by the shown logic.
- Has unknown memory, global-state, ownership, and exceptional-control-flow effects.

If either dependency does not return normally, the supplied body establishes no subsequent normal completion or guaranteed target write.

## Acceptance examples

These are conditional static examples, not executed tests. Unless otherwise stated, they assume 32-bit unsigned words, valid required accesses, ordinary storage without additional access effects, readiness eventually observed, and normally returning dependencies.

### First replacement pattern

Suppose both initial pointer evaluations obtain the same address. The initial snapshot and the later target read both yield `0x12345678`, and the selector is 0.

The function writes `0x00000C20` and returns `0x12345678`. Selector 1 produces the same replacement pattern.

The dependencies occur in the stated order before the later target read.

### Second replacement pattern

Suppose the initial snapshot and later target read both yield `0xFFFFFFFF`, and the selector is 3.

The function writes `0x80000DAF` and returns `0xFFFFFFFF`. Selector 2 produces the same replacement pattern.

The written value preserves bit 31 and bits 0–2 from the later target read.

### Snapshot differs from the later target value

Suppose the initial snapshot is `0x00000001`, the post-call target read yields `0x80000006`, and the selector is 1.

The function writes `0x80000C26` and returns `0x00000001`.

Using the initial snapshot’s low bits to form the write would violate this behavior, regardless of whether the two reads used the same address.

### Three distinct address roles

Suppose the first shared-pointer evaluation yields address A, which is retained for updates. Before the separate snapshot-address evaluation, an observable change makes the shared pointer yield address B. The function reads `0x11223344` from B.

Before a later polling evaluation, the shared pointer changes to address C. A poll reads word index 7 through C and observes bit 1 set. The selector is 2, and the post-call first-word read at A yields `0x80000005`.

The function writes `0x80000DAD` at A and returns `0x11223344`.

Its snapshot comes from B, its successful poll uses C, and its update uses A. A need not have been readable at the snapshot step, but must support the later read and write.

This example is conditional on the execution model observing the stipulated shared-pointer changes.

### No-update selector

Suppose the snapshot read yields `0xA5A5A5A5` and the selector is 4.

After polling and both calls, the function returns `0xA5A5A5A5` without its own post-call read or write at the retained update address. That address need not support a target dereference on this path.

The same classification applies to `0xFFFFFFFF` under the 32-bit model. Dependency-originated changes are not excluded.

### Polling progression and nontermination

Suppose successive polling evaluations observe `0x00000000`, `0x00000001`, and `0x00000002`.

The first two observations do not permit the dependency calls. The third does.

If every polling evaluation instead observes bit 1 clear, the shown execution remains in the loop, after having already retained the update address and taken the initial snapshot.

## Uncertainties

The following remain **documented_only**:

- **Exact widths and ABI:** Integer width, pointer width, calling convention, and return-register details are not independently established. Numeric examples use the stated 32-bit model.
- **Machine-level accesses:** The represented separate pointer evaluations do not establish exact physical load counts or instruction ordering.
- **Asynchronous visibility:** Volatile qualification, barriers, cache effects, concurrency rules, and whether external pointer changes are observable are unknown.
- **Storage interpretation:** Device identity, readiness meaning, and replacement-pattern meanings are unknown.
- **Access effects:** Device-specific read and write semantics are not supplied.
- **Dependency contracts:** State preservation, ownership effects, lifetime effects, failure behavior, and complete interfaces are unknown.
- **Address lifetime:** Nothing guarantees that the retained update address remains usable across the calls.
- **Fault handling:** Behavior following invalid access, bus errors, or other exceptional control flow is not established.
- **Progress:** Nothing guarantees eventual observation of bit 1 set.
- **Unmapped call identity:** An opaque function ID for the first dependency is unavailable.

The inferred intent is a readiness wait followed by conditional configuration. It does not establish a particular device, operating environment, synchronization guarantee, or ownership policy.

## Evidence

Established static evidence consists of:

- An initial evaluation of the shared pointer to retain an update address.
- A separate shared-pointer evaluation supplying the address for the initial word snapshot.
- A loop testing bit 1 of word index 7 through the shared pointer expression.
- An external call with argument 1, followed by F-2a818e25cf9e.
- Unsigned selector classes 0–1, 2–3, and all other values.
- A conditional post-call read through the retained update address.
- Masking by `0x80000007`, combination with one of two replacement patterns, and a write through that same retained address.
- Return of the separately obtained initial snapshot on every shown normal-return path.

This contract is based solely on the supplied static representation. No runtime tests, hardware observations, or independent implementation inspection have been performed.
