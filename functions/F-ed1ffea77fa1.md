<!-- Trusted provenance: F-ed1ffea77fa1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ed1ffea77fa1

## Purpose

F-ed1ffea77fa1 conditionally processes a two-entry table under a byte-valued admission gate. A nonzero admission byte causes an immediate return. Otherwise, zero input selects only the first entry; nonzero input selects both entries and adds two preliminary dependency calls.

Each selected entry must be zero when read for processing to continue normally. A zero observation is followed by an explicit four-byte zero write. A nonzero observation invokes a first failure-path dependency and then, if that dependency returns, a dependency represented as non-returning. After successful processing of all selected entries, the function writes one to a previously saved gate destination.

The gate reference is evaluated twice before admission: once to save the eventual completion destination, and again to obtain the destination of the admission-byte read. These destinations coincide only if the two evaluations produce the same reference.

One-time initialization or consistency enforcement is a possible intent, not an established semantic role. The evidence does not establish resource cleanup, ownership of the table entries, or diagnostic semantics for the failure-path dependencies.

## Interface

### Input and result

The function accepts one integer argument represented as 32 bits. Only equality to zero affects behavior:

- **Zero mode:** select the first table entry only.
- **Nonzero mode:** make two additional preliminary calls and select both entries.

Negative and positive nonzero values select the same mode. No caller-supplied count or index exists.

The function has no return value. Its visible outcomes are:

1. Immediate normal return when the admission byte is nonzero.
2. Normal completion after all selected entries are observed as zero, explicitly zero-written, and followed by the completion-byte write.
3. A failure path through two dependencies, the second represented as non-returning.
4. Earlier loss of continuation if any invoked dependency does not return.

No returned error code, exception contract, or particular termination mechanism is established.

### Shared state and layout

| State | Relevant representation | Use |
|---|---|---|
| Global gate reference | Reference to byte-addressable storage | Evaluated twice at entry |
| Saved gate destination | Result of the first gate-reference evaluation | Receives the final one-byte write |
| Admission destination | Result of the second gate-reference evaluation | Supplies the admission byte |
| Global table base | Address-valued storage represented with a 32-bit integer | Captured after the preliminary dependency sequence |

The saved gate destination and admission destination are distinct roles, not necessarily distinct objects. Equality of their references is not enforced.

The table has two directly addressed entries:

| Entry | Byte offset from captured table base | Read width | Successful write |
|---|---:|---:|---|
| First | 0 | 4 bytes | Four-byte zero |
| Second | 4 | 4 bytes | Four-byte zero |

Entry values are tested only for zero versus nonzero. Their application-level interpretation is unknown. The gate test reads one byte; any nonzero byte suppresses processing. Completion writes exactly the byte value one.

### Widths and address calculations

The supplied representation uses 32-bit integer values for the input, entry tests, table-base representation, and unsigned position arithmetic. Entry writes cover four bytes.

Processing visits positions zero and one, then ends when the position advances to two. Multiplication by four therefore produces only the offsets zero and four; neither the position nor the offset calculation overflows in this traversal.

The represented combination of the 32-bit base and unsigned offset can wrap at the 32-bit boundary. The representation does not establish that a wrapped result denotes valid storage. Normal behavior requires both selected effective addresses to be valid under the actual execution environment.

Zero-mode selection includes an eight-bit reduction of the position. Because the only positions are zero and one, this reduction does not change selection.

### Preconditions

For a particular execution to follow the normal paths described here:

- The second gate-reference evaluation must identify readable byte storage.
- If completion is reached, the first evaluation’s saved destination must still identify writable byte storage.
- The table base captured after the dependency sequence must provide valid four-byte accesses for each selected entry.
- Applicable alignment and address-validity requirements must hold.
- Each dependency must return where subsequent behavior is expected.
- Dependency effects must leave every later-required access valid.

The saved gate destination need not be read by this function. Conversely, the admission destination need not receive the completion write.

Zero mode does not directly access the second table entry. This does not constrain dependency accesses or eliminate possible aliasing with other accessed storage.

## Behavior

### Gate-reference evaluations and admission

The first observable state access is an evaluation of the global gate reference. Its result is saved for possible completion.

The function then evaluates the global gate reference again and reads one byte through that second result.

If the byte is nonzero, the function returns immediately. On this path it makes no dependency calls, does not capture the table base, performs no direct table-entry accesses, and performs no direct writes.

The first reference evaluation still occurs on this early-return path. Describing admission as a read through the saved reference is valid only when both reference evaluations yield the same destination.

If the admission byte is zero, processing continues. There is no later admission-byte check and no later reload of the saved completion destination. In particular:

- A later change to the global gate reference does not redirect completion.
- A later change to either gate byte does not itself cause this invocation to reconsider admission.
- The saved destination can receive one even if it was not the destination whose byte admitted the operation.

These are access-order properties, not synchronization guarantees.

### Preliminary dependency sequence

With zero input, the function invokes F-4e9faa1368ca directly after admission.

With nonzero input, it invokes:

1. F-fdd8589eca2d.
2. F-a89b37b93e7d.
3. F-4e9faa1368ca.

Each later call requires the preceding call to return. No explicit arguments are shown for these calls.

F-4e9faa1368ca supplies a represented return value, but that value does not affect entry selection:

- Zero mode derives selection from the current position before each selection decision.
- Nonzero mode selects both entries independently of that value.

Its invocation remains behaviorally significant. Its side effects and return behavior are unknown.

### Table-base capture

After F-4e9faa1368ca returns, the function captures the global table base.

The value present at this capture determines both entry addresses. A replacement installed by an earlier dependency is used only if it remains the table-base value when capture occurs. Later preliminary calls may replace it again.

There is no table-base reload between the two entry positions. Subsequent changes to the global table base therefore do not redirect this invocation’s direct table accesses, although changes to the captured storage itself can affect observed values.

### Selection and entry processing

Processing order is always first entry, then second position.

In zero mode, the first entry is selected and the second position is skipped. The skipped entry receives neither a direct read nor a direct write.

In nonzero mode, both entries are selected.

For a selected entry, the function reads four bytes and tests the resulting value:

- **Zero:** write four-byte zero to the same entry, then continue.
- **Nonzero:** invoke F-da23f49bf314. If it returns, invoke F-feee9156a257, which is represented as non-returning.

The zero write is explicit even though the immediately preceding read observed zero. It is not merely a description of the entry’s expected state.

On a nonzero observation, no direct zero write to that entry precedes the failure-path calls. A first-entry failure therefore occurs before any direct table zero write. A second-entry failure occurs after the first entry’s zero write.

Dependency effects are separate from these direct actions. F-da23f49bf314 might modify the failing entry or other state, but such modification is not a visible recovery condition: after it returns, the next represented action is the call to F-feee9156a257, not a renewed entry test.

### Completion

After every selected entry follows its zero-valued path, the function writes the byte value one through the saved gate destination and returns normally.

Completion in zero mode does not certify the second entry’s value.

Completion in nonzero mode establishes an ordered history: the first entry was observed as zero and zero-written, followed by the second entry being observed as zero and zero-written. It does not establish an atomic two-entry observation or a lasting invariant under concurrent modification.

## Side effects

### Direct access order

On an admitted execution, the direct state-access sequence is:

1. Evaluate the global gate reference and save its result.
2. Evaluate the global gate reference again.
3. Read one byte through the second result.
4. Perform the mode-dependent dependency sequence.
5. Capture the global table base.
6. Read the first four-byte entry.
7. If zero, write four-byte zero to that entry.
8. In nonzero mode, read the second four-byte entry and, if zero, write four-byte zero to it.
9. Write one byte with value one through the saved gate destination.

A nonzero admission byte ends the sequence after step 3. A nonzero selected entry diverts execution into the failure-path dependencies before its normal zero write.

This ordering describes the supplied representation. It does not independently establish machine-level atomicity, memory-ordering guarantees, or compiler treatment of shared accesses.

### Ownership and dependency effects

No direct allocation, release, or ownership transfer is visible. The entry values are not passed as explicit arguments to a release operation. Their being four-byte values does not establish that they are pointers or handles.

Dependencies may read or modify global state, replace references, alter table contents, invalidate storage, or fail to return. Their preservation of any relevant state must be an explicit condition of an example or supplied by separate evidence.

The differing capture times matter:

- Completion uses the first gate-reference evaluation.
- Admission uses the second gate-reference evaluation.
- Table addressing uses the table-base value captured after all preliminary calls.

### Partial progress, aliasing, and reentrancy

There is no visible rollback. A second-entry failure leaves the first entry’s direct zero write in the execution history, without the completion write.

Disjointness of the gate destinations, table, and global-reference storage is not established. Aliasing may cause a write through one role to affect storage described under another role. Direct access descriptions do not imply physically separate objects.

The completion marker is written only at the end. No in-progress marker or reentrancy guard is visible. A dependency that reenters may encounter a zero admission byte, depending on reference values and intervening effects. Safe concurrent or reentrant use is not established.

## Dependencies

| Opaque function ID | Invocation condition | Established call-site role |
|---|---|---|
| F-fdd8589eca2d | Admitted execution with nonzero input | First preliminary call |
| F-a89b37b93e7d | Nonzero input, after F-fdd8589eca2d returns | Second preliminary call |
| F-4e9faa1368ca | Admitted execution after applicable earlier calls return | Final call before table-base capture |
| F-da23f49bf314 | A selected entry is read as nonzero | First failure-path call |
| F-feee9156a257 | F-da23f49bf314 returns | Second failure-path call, represented as non-returning |

No dependency’s internal memory effects, ownership rules, or semantic purpose are established. “Non-returning” does not identify whether the mechanism is process termination, an exception, a transfer of control, or something else.

## Acceptance examples

These are conditional static expectations, not executed tests.

Unless a case explicitly states otherwise, assume that required accesses are valid, the described objects do not overlap, and there is no concurrent or reentrant interference. All invoked dependencies return where continuation is described, except F-feee9156a257, whose represented non-returning behavior is assumed.

These common conditions do **not** imply that dependencies preserve table values or global references. Entry conditions below apply at the actual reads; reference conditions apply at their stated evaluations or capture.

### Nonzero admission byte

**Conditions:** The second gate-reference evaluation identifies a byte containing 2. The first evaluation may identify the same or a different valid destination.

**Expected:** Immediate normal return after the admission read. No dependency calls, table-base capture, table-entry accesses, or completion write occur. The saved destination is not written.

### Zero mode with a nonzero unselected entry

**Conditions:** The admission byte is zero and input is zero. When the table base is captured after F-4e9faa1368ca, it identifies a table whose first entry is zero at its actual read. F-4e9faa1368ca leaves the saved gate destination writable. The second entry may be nonzero.

**Expected:** Only F-4e9faa1368ca is called before table processing. The first entry is read and explicitly zero-written. The second entry is not directly accessed. The saved gate destination receives one, followed by normal return.

There is no claim about the second entry’s final value unless dependency effects on it are additionally constrained.

### Nonzero mode with successful processing

**Conditions:** The admission byte is zero and input is any nonzero 32-bit value. After the three preliminary calls, the captured table base identifies entries that each contain zero at their respective reads. The calls leave the saved gate destination writable.

**Expected:** F-fdd8589eca2d, F-a89b37b93e7d, and F-4e9faa1368ca occur in that order. The first entry is read and zero-written, then the second is read and zero-written. Finally, the saved gate destination receives one.

Initially zero entries alone are insufficient: the read-time conditions must survive, or result from, the dependency effects.

### First-entry failure after dependency modification

**Conditions:** The admission byte is zero. The first entry may initially be zero, but the preliminary dependency effects produce a nonzero value at the first selected-entry read.

**Expected:** F-da23f49bf314 is invoked, followed by F-feee9156a257 if the former returns. No direct table-entry zero write or completion write precedes those calls.

Even if F-da23f49bf314 clears the entry, its return still leads to F-feee9156a257.

### Second-entry failure

**Conditions:** The admission byte is zero and input is nonzero. After the preliminary calls and table-base capture, the first entry reads as zero and the second reads as nonzero.

**Expected:** The first entry receives its direct zero write before the second entry triggers the failure-path calls. On the represented non-returning path, no completion write occurs. Effects of the failure-path dependencies on either entry remain unconstrained.

### Different saved and admission destinations

**Conditions:** The two gate-reference evaluations produce different destinations. The second destination contains zero. All selected entries read as zero after the preliminary calls, and dependencies preserve writability of the first destination.

**Expected:** Processing is admitted by the second destination, but completion writes one through the first. The function does not directly perform a completion write through the second destination.

### Replacement table base

**Conditions:** A preliminary dependency installs a replacement table base. At capture after F-4e9faa1368ca returns, the global still contains that replacement. Selected entries in the replacement table read as zero, and the saved gate destination remains writable.

**Expected:** Direct table processing uses the replacement table. Completion uses the originally saved gate destination, regardless of later changes to the global gate reference.

If a later preliminary call installs another table base before capture, that later value is used instead.

### Different returned values from F-4e9faa1368ca

**Conditions:** Two executions differ only in this dependency’s returned value. Its side effects, return behavior, reference values at capture, and entry values at reads are otherwise identical.

**Expected:** Entry selection is identical: first entry only for zero input, both entries for nonzero input.

## Uncertainties

The following remain **documented_only**:

- **Dependency semantics:** Internal reads, writes, ownership, preservation guarantees, and failure mechanisms are unavailable.
- **Unexpected return:** F-feee9156a257 is represented as non-returning. Its unexpectedly returning is not an accepted continuation guarantee.
- **Storage meaning:** The application-level meaning of the table and its entries is unknown.
- **Reference stability:** No synchronization or immutability guarantee establishes equality of the two initial gate-reference evaluations.
- **Aliasing:** Disjointness among gate storage, table storage, and global-reference storage is unproven.
- **Concurrency:** Atomicity, visibility, and safety under concurrent or reentrant execution are unspecified.
- **Invalid addresses:** Outcomes for inaccessible, misaligned, invalidated, or wrapped effective addresses are not established.
- **Platform details:** Exact ABI, pointer representation, and endianness are not independently confirmed.
- **Intent:** Initialization or consistency enforcement is inferred, not established.

## Evidence

Established static evidence consists of the two gate-reference evaluations, the admission-byte check through the second result, the saved completion destination, mode-dependent preliminary calls, table-base capture after F-4e9faa1368ca, two-position traversal, four-byte entry accesses, failure-path call order, and final one-byte write.

Entry selection is independent of F-4e9faa1368ca’s returned value because zero mode replaces the selection value with the bounded position, while nonzero mode selects each entry without consulting that value.

The distinction between initial state and read-time state follows from the intervening dependency calls. No dependency-preservation guarantees or dependency implementations were supplied.

No runtime tests have been performed.
