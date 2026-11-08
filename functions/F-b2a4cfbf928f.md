<!-- Trusted provenance: F-b2a4cfbf928f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b2a4cfbf928f

## Purpose

F-b2a4cfbf928f conditionally resolves an indexed entry from receiver-associated storage, updates receiver state, and invokes operations on a receiver-associated target. It first invokes a supplied callback. If an extent-like receiver field is nonzero, it obtains an auxiliary value and invokes F-d7be9cfb4000 before selecting a branch.

The established contract concerns memory accesses, signed comparisons, dependency calls, and their ordering. “Entry,” “boundary,” and “target” are descriptive labels. Selection, clipping, notification, or presentation semantics are **inferred intent**, not established domain behavior.

Normal execution either returns during branch selection or completes entry processing and a final call involving a stack-backed temporary. No explicit success value or error result is established.

## Interface

### Explicit inputs and result

| Input | Established properties |
|---|---|
| First argument | A 32-bit value with no direct use visible in the body. |
| Callback | A callable address invoked before any visible receiver-field access. No explicit arguments or consumed return value are shown. |

The function has no explicit return value. The first argument’s lack of direct use does not establish that its incoming machine-level value is irrelevant to the callback; the complete calling convention is unknown.

### Additional execution context

The visible behavior also depends on values not presented as ordinary arguments:

- A receiver address.
- A signed 32-bit candidate index.
- Two independently supplied 8-bit state values.
- A 32-bit value committed to the receiver during entry processing.
- A 32-bit condition controlling two optional target calls.

Their provenance and calling convention are **documented_only**. The supplied representation does not establish whether this is a conventional standalone function or depends on surrounding execution state.

The two state values are not proven to be Boolean encodings or to differ.

### Receiver layout

All offsets are byte offsets from the receiver address.

| Offset | Width | Observed use |
|---|---:|---|
| `+0x18` | 32 bits | Reference to an auxiliary polymorphic object. |
| `+0x1c` | 32 bits | Reference to the target polymorphic object. |
| `+0x20` | 32 bits | Base index, used in signed comparisons and index arithmetic. |
| `+0x24` | 32 bits | Destination for the boundary result associated with F-d7be9cfb4000. |
| `+0x28` | Address within receiver | Passed to F-5cd5fa633bae. |
| `+0x2c` | 32 bits | Extent-like value, tested against zero and used in arithmetic. |
| `+0x40` | 32 bits | Destination for the candidate index after entry resolution. |
| `+0x44` | 8 bits | Destination for one of the two supplied state values. |
| `+0x48` | 32 bits | Destination for the supplied commit value. |

The visible dispatch layout uses an object’s first word as a dispatch-table pointer and byte offsets within that table. The supplied representation supports a 32-bit pointer-layout interpretation; portability to other layouts is not established.

### Global data inputs

Two separate 32-bit global-data words are read on the entry-processing path:

| Descriptive label | Read position and use |
|---|---|
| Global word A | Read after F-b9acea4b06ad returns; its value is written through the address returned by that dependency. |
| Global word B | Read after the required and any optional target calls; its value is written into the temporary’s first four-byte word before F-22abefe7f288. |

These labels identify data locations, not dependency functions. Their definitions, values, initialization, and mutability are unknown. Neither is established to be a fixed constant. They are not established to have equal values.

### Execution preconditions

Every memory location, object reference, dispatch table, callable entry, and returned address used on the executed path must support the indicated access.

In particular:

- The callback must be callable.
- F-5cd5fa633bae must return an entry address permitting a four-byte read at offset `+8`.
- F-b9acea4b06ad must return an address permitting a four-byte write.
- Both global words must be readable when their respective reads are reached.
- The temporary must support the observed accesses and dependency uses.

No local checks enforce these conditions. Recovery from invalid accesses, arithmetic overflow, or exceptional dependency exits is not established.

## Behavior

### Callback and initial extent test

The callback runs first, with no visible receiver access preceding it. Its return value is not visibly consumed.

After the callback returns, the function reads the receiver’s four-byte extent-like field at `+0x2c` and tests it for nonzero. Callback mutations can therefore affect this test and later accesses.

This is not a positivity test. A negative signed value also enters boundary processing.

### Conditional boundary processing

When the initial extent read is nonzero:

1. The auxiliary reference is obtained from `+0x18`, and its dispatch-table entry at byte offset `+0x08` is invoked. No explicit arguments are shown. A 32-bit result is retained.
2. After that call returns, the base at `+0x20` is read and captured.
3. F-d7be9cfb4000 is invoked with an endpoint expression and the auxiliary result. The endpoint uses a fresh extent read and the captured base:

   **extent + captured base − 1**

4. A 32-bit value made available following F-d7be9cfb4000 is stored at receiver offset `+0x24`.

The representation does not expose that boundary value as an ordinary return value. Its result convention is **documented_only**.

The endpoint arithmetic is represented at 32-bit width. No source-language overflow rule or guarantee of signed representability is established.

The next comparison uses the captured base, not a base reloaded after F-d7be9cfb4000. That dependency is not assumed to preserve the receiver.

### Branch selection and state-byte write

If boundary processing occurred and the boundary result is signed-less-than the captured base:

- The first supplied state byte is written to `+0x44`.
- The function returns only when both signed comparisons hold: the candidate is below the captured base, and the boundary is below the candidate.
- Otherwise, execution proceeds directly to entry processing.

The early-return condition is the strict interval:

**boundary < candidate < captured base**

Equality with either endpoint does not cause this return. This branch bypasses the ordinary range checks.

In every other case, including an initially zero extent:

1. The second supplied state byte is written to `+0x44`.
2. The current base is read from `+0x20`. The function returns if the candidate is signed-less-than that base.
3. If execution continues, the extent and base are read to form their sum. The function returns if that sum is signed-less-than-or-equal-to the candidate.
4. Otherwise, execution proceeds to entry processing.

With unchanged fields and representable arithmetic, this ordinary path admits:

**base ≤ candidate < base + extent**

This interval describes the observed comparisons; it is not additional validation. There is no separate rejection of negative extents.

The state-byte write precedes either ordinary range return. An early return is not a no-op.

### Entry resolution and candidate commit

Entry processing invokes F-5cd5fa633bae with:

- The address at receiver offset `+0x28`.
- The candidate minus a fresh read of the base at `+0x20`.

The subtraction is represented at 32-bit width. Overflow behavior and the dependency’s rules for negative or otherwise invalid relative indices are unknown.

The returned address is retained as the entry address. After the dependency returns, the candidate is written to receiver offset `+0x40`.

Consequently, entry-resolution effects precede this write, and subsequent target calls occur after it.

### Payload delivery

The target reference is loaded from receiver offset `+0x1c`. Its dispatch-table entry at byte offset `+0x10` is invoked with two explicit arguments:

- The target reference.
- The four-byte payload read from the resolved entry at offset `+8`.

The payload’s meaning and ownership are unknown. The evidence does not establish a total ordering among all reads used to assemble this call.

### Temporary preparation and first global-data read

F-b9acea4b06ad is invoked with the address of a stack-backed temporary and the retained entry address.

After that dependency returns:

1. Global word A is read as a 32-bit value.
2. That value is written through the address returned by F-b9acea4b06ad.
3. The target reference is reloaded from receiver offset `+0x1c`.
4. The target’s dispatch-table entry at byte offset `+0x1c` is invoked with the target reference and temporary address.

The global read is not established to occur before F-b9acea4b06ad. In particular, if that dependency changes global word A and returns normally, the subsequent read obtains the value present after the call, subject to the execution environment’s memory semantics.

The returned writable address is not proven to equal the temporary’s base. This write must not be conflated with the later write to the temporary’s first word.

### Commit value and target actions

After temporary delivery returns:

1. The supplied 32-bit commit value is written to receiver offset `+0x48`.
2. The target reference is reloaded, and its dispatch-table entry at byte offset `+0x3c` is invoked. No explicit arguments are shown.
3. If the supplied 32-bit condition is nonzero:
   - The target is reloaded and its dispatch-table entry at `+0x20` is invoked, with no explicit arguments shown.
   - The target is reloaded again and its dispatch-table entry at `+0x34` is invoked with the target reference, zero, and one.

The condition is tested for nonzero, not for equality to one.

These separate reloads are observable distinctions. Earlier dependencies are not assumed to preserve the target reference, object contents, or dispatch table.

### Second global-data read and final temporary call

After the required target call and any optional target calls return:

1. Global word B is read as a 32-bit value.
2. That value is written into the first four-byte word of the stack-backed temporary.
3. F-22abefe7f288 is invoked with the temporary address.
4. The function returns without an explicit result.

The global read occurs after the target calls, not during earlier temporary preparation. Those calls, and preceding dependencies, are not assumed to preserve global word B.

The last dependency resembles temporary finalization by position, but its actual semantics are unknown. No visible exception handler establishes that it runs after an abnormal exit from an earlier operation.

## Side effects

### Direct receiver writes

| Destination | Condition and ordering |
|---|---|
| `+0x24`, four bytes | Written after F-d7be9cfb4000 on the nonzero-initial-extent path. |
| `+0x44`, one byte | Exactly one direct state-byte write when normal execution reaches branch selection. |
| `+0x40`, four bytes | Candidate written after F-5cd5fa633bae returns. |
| `+0x48`, four bytes | Supplied commit value written after temporary delivery returns. |

An initially zero extent causes no direct write to `+0x24`. A branch-selection return occurs before direct writes to `+0x40` and `+0x48`.

These are direct-write guarantees only. Dependencies may also modify these fields.

### Other memory effects and ownership

The function reads global word A after F-b9acea4b06ad and writes the observed value through that dependency’s returned address. It later reads global word B and writes the observed value into the temporary’s first word. The function does not explicitly assign either global location by name; possible aliasing through other addresses is not established or excluded.

Dependencies may modify globals, receiver fields, referenced objects, dispatch tables, entry storage, or memory ownership. Neither global read may be replaced in this contract by an assumed immutable or entry-time value.

The temporary is backed by the current invocation’s stack. Its full extent and layout are unknown. Passing its address does not establish permission to retain it beyond the invocation, nor prove that callees refrain from retaining it.

No allocation, release, ownership transfer, or reference-count policy is established directly. F-22abefe7f288 must not be assumed to free memory.

There is no visible error reporting, rollback, retry, or compensation. If a dependency does not return normally, subsequent operations are not guaranteed.

## Dependencies

| Dependency | Established call-site behavior |
|---|---|
| Supplied callback | Called first; no explicit arguments or consumed result shown. |
| Auxiliary dispatch at `+0x08` | Supplies a 32-bit value passed to F-d7be9cfb4000. |
| F-d7be9cfb4000 | Receives endpoint expression and auxiliary result; boundary-result convention remains incomplete. |
| F-5cd5fa633bae | Receives receiver storage address and relative index; returns the entry address. |
| Target dispatch at `+0x10` | Receives target reference and entry payload. |
| F-b9acea4b06ad | Receives temporary and entry addresses; returns the address for the write using subsequently read global word A. |
| Target dispatch at `+0x1c` | Receives target reference and temporary address after that write. |
| Target dispatch at `+0x3c` | Runs after the receiver commit-value write; explicit arguments are unknown. |
| Target dispatch at `+0x20` | Runs only for a nonzero condition; explicit arguments are unknown. |
| Target dispatch at `+0x34` | Follows optional `+0x20`; receives target reference, zero, and one. |
| F-22abefe7f288 | Receives the temporary after its first word is overwritten with the subsequently read global word B. |

Dispatch offsets identify call sites, not independently established function identities. No additional opaque function IDs are supplied for them.

No dependency is assumed to preserve globals or to acquire, transfer, or release ownership.

## Acceptance examples

These are conditional static consequences, not executed tests. Unless stated otherwise, arithmetic is representable, required accesses are valid, dependencies return normally, and relevant fields remain unchanged.

### Zero extent

For base 10 and extent zero:

- The callback runs.
- Auxiliary dispatch and F-d7be9cfb4000 are skipped.
- The second state byte is written.
- Candidate 9 returns at the lower check.
- Candidate 10 returns at the upper check because the sum is 10.
- Neither candidate reaches entry resolution or either explicit global-data read.
- No direct write changes `+0x24`.

### Ordinary half-open range

For base 10, extent 3, and boundary result at least 10:

- The boundary is stored and the second state byte is written.
- Candidate 10 reaches entry resolution with relative index zero.
- Candidate 12 reaches it with relative index two.
- Candidate 13 returns before entry resolution.

### Strict-interval branch

For captured base 10 and boundary result 5:

- The first state byte is written.
- Candidate 7 returns because it lies strictly between 5 and 10.
- Candidates 5, 10, and 4 proceed to entry processing.

With unchanged base, candidate 4 supplies relative index −6. Whether F-5cd5fa633bae accepts that index is unknown.

### Base mutation during boundary computation

Suppose the captured base is 10, F-d7be9cfb4000 supplies boundary 5, and that dependency changes the receiver base to 20.

Candidate 10 bypasses the strict-interval return because comparison uses captured base 10. Entry resolution then uses the fresh base 20 and receives relative index −10.

### Global word A changes during preparation

Suppose global word A initially contains `0x11111111`, and F-b9acea4b06ad changes it to `0x22222222` before returning a valid writable address.

Absent an intervening change, the function writes `0x22222222` through that returned address before temporary delivery. It does not use the earlier value merely because it existed before the dependency call.

### Global word B changes during target actions

On an admitted path, suppose the final executed target call leaves global word B equal to `0x33333333`.

Absent an intervening change, the function writes that value into the temporary’s first word before invoking F-22abefe7f288. This is independent of the value previously read from global word A.

### Optional actions and update visibility

A zero condition skips target dispatches `+0x20` and `+0x34`. Any nonzero condition executes both, in that order, after `+0x3c` and before the global word B read.

The payload-delivery call occurs after the candidate write at `+0x40`. Temporary delivery occurs before the commit-value write at `+0x48`. These ordering claims do not exclude intervening dependency mutations.

## Uncertainties

The following remain **documented_only**:

- Complete calling convention and provenance of implicit context values.
- Whether the callable unit relies on surrounding execution state.
- Actual argument conventions for calls shown without explicit arguments.
- The boundary-result convention associated with F-d7be9cfb4000.
- Semantic meanings of state bytes, payloads, and target operations.
- Definitions, values, initialization, and mutability of both global words.
- Whether dependencies alter either global before its respective read.
- Temporary size, complete layout, construction, retention, and finalization obligations.
- Aliasing among returned addresses, temporary storage, receiver storage, and globals.
- Dependency effects on memory and ownership.
- Arithmetic semantics outside signed 32-bit representability.
- Validity rules for unusual extents or negative relative indices.
- Fault, exception, nonlocal-exit, and abnormal-cleanup behavior.

No validation or preservation guarantee is added to resolve these gaps.

## Evidence

This contract is based solely on the supplied static representation and opaque dependency mapping.

Established evidence includes access widths, receiver and dispatch offsets, signed comparisons, branch structure, explicit arguments, direct writes, the two global-data reads, and ordering between calls and accesses. In particular, global word A is read after F-b9acea4b06ad returns, and global word B is read after the required and optional target calls. Their immutability is not established.

Domain labels and possible finalization intent are interpretations rather than proven semantics.

No runtime tests have been performed. Acceptance examples express conditional consequences of static evidence, not observed executions.
