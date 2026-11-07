<!-- Trusted provenance: F-edfd5e489b39; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-edfd5e489b39

## Purpose

F-edfd5e489b39 conditionally registers a caller-specified memory region in shared state, initializes an aligned block within that region, updates an accumulated size, and invokes a state-processing dependency. It can instead use a default region recorded in the shared state when both supplied arguments are zero.

A separate final path conditionally performs an indirect invocation. That path is reachable both after a successful registration and when registration is skipped because the region table is full.

The memory layout and initialization suggest allocator-region setup. That interpretation is **inferred intent**, not an established contract of the dependencies. Established behavior is the sequence of reads, writes, calls, and branches described below.

## Interface

### Inputs and return

The function accepts two 32-bit values:

| Input | Interpretation in the visible body |
|---|---|
| Region base | Address used for region initialization and end-address calculations. |
| Region extent | Byte quantity used to derive an aligned block size and nominal region end. |

Although the supplied signature represents both inputs as signed integers, the body uses them as address and size bit patterns. Alignment masks and the capacity comparison are explicitly unsigned operations. A replacement must preserve the relevant 32-bit arithmetic rather than silently widening the calculations.

The function has no defined return value. It may return normally, invoke an indirect target and then return if that target returns, or enter a dependency identified as non-returning.

### Shared-state layout

The body obtains one state pointer from a global pointer slot after its first dependency call. All subsequent explicit state accesses use that captured pointer, even if a dependency later changes the global slot.

The following offsets are relative to the captured state address:

| Offset | Width | Visible role |
|---|---:|---|
| `0x04` | 32 bits | Accumulated size; increased during registration. |
| `0x10 + 8 × index` | 32 bits | Registered aligned block address. |
| `0x14 + 8 × index` | 32 bits | Registered aligned block size. |
| `0xB0` | 32 bits | Registration count and table index. |
| `0xB4` | 8 bits | Final indirect-invocation enable field; must equal exactly 1. |
| `0xB8` | 32 bits | Pointer to an object whose first 32-bit word gates the indirect invocation. |
| `0xC4` | 32 bits | Default region base. |
| `0xC8` | 32 bits | Default region extent. |
| `0xCC` | 8 bits | Default-region enable field; any nonzero value enables substitution. |

The registration capacity is 20 entries. For indices 0 through 19, the two-word descriptors occupy offsets `0x10` through `0xAC`.

### Preconditions for defined, successful use

The function does not validate these conditions before accessing memory:

- The global state-pointer slot must be readable.
- The captured state must permit the field reads and writes actually reached.
- A selected region must permit every initialization write described below.
- Any non-null object pointer used by the final gate must permit a 32-bit read at its base.
- The indirect target must be valid if the final invocation is reached.
- Dependencies must satisfy whatever additional preconditions they require; those contracts are not supplied.

There is no independent null-address rejection, minimum-extent check, overlap check, or arithmetic-overflow rejection.

## Behavior

### 1. Preliminary calls and state capture

The initial ordering is fixed:

1. Invoke F-2fd8826557a4, with no explicit arguments shown.
2. Read the global state-pointer slot and capture its value.
3. Invoke F-c0fc6d033caa, with no explicit arguments shown.
4. Begin examining fields through the captured state pointer.

The first dependency can therefore affect which state pointer is captured. The second can affect the state contents before the function reads them, but changing the global pointer slot during that call does not redirect the explicit accesses that follow.

No dependency return value is used by the visible body.

### 2. Optional default-region selection

The function first reads the byte at state offset `0xCC`.

If that byte is nonzero and both supplied inputs are zero, it substitutes:

- the 32-bit value at `0xC4` for the region base;
- the 32-bit value at `0xC8` for the region extent.

The default base is read before the default extent.

Otherwise, the supplied inputs remain selected. In particular:

- A zero base with a nonzero extent does not select defaults.
- A nonzero base with a zero extent does not select defaults.
- Two zero inputs do not select defaults when the enable byte is zero.

The selected defaults are not validated.

### 3. Capacity decision

The function reads the count at `0xB0` and compares it as an unsigned 32-bit value against 20.

- Counts from 0 through 19 enter the registration path.
- Counts of 20 or greater skip the entire registration path.

A bit pattern representing a negative signed integer is a large unsigned count and therefore skips registration.

Skipping registration does not skip the final indirect-invocation gate.

### 4. Aligned address and block-size calculation

Let:

- \(B\) be the selected region base;
- \(L\) be the selected region extent;
- \(P\) be the aligned block address;
- \(S\) be the recorded block size;
- \(E\) be the trailing-marker address.

For the 32-bit address arithmetic represented by the supplied body:

\[
P = (B + 15)\mathbin{\&}\texttt{0xFFFFFFF8}
\]

\[
S = \left(L - ((P + 4) - B) - 4\right)
    \mathbin{\&}\texttt{0xFFFFFFF8}
\]

\[
E = P + S
\]

The address additions and subtractions must not be replaced by wider, overflow-free arithmetic without establishing equivalent behavior. In ordinary non-wrapping cases, the size expression simplifies to:

\[
S = (L - (P-B) - 8)\mathbin{\&}\texttt{0xFFFFFFF8}
\]

Both \(P\) and \(S\) have their low three bits cleared. For a non-wrapping base calculation, \(P-B\) is between 8 and 15 bytes inclusive. Thus \(P\) is not merely the first 8-byte-aligned address at or above the base: an already 8-byte-aligned base produces \(P=B+8\).

A small extent can underflow the size calculation and produce a large masked size. The function does not reject that case before writing.

### 5. Region initialization and descriptor publication

After calculating \(P\), the function performs the following observable operations in order:

1. Write a 32-bit zero at address \(P-4\).
2. Calculate \(S\).
3. Write a 32-bit zero at address \(E\).
4. Read the count again to select the descriptor-address slot, then write \(P\) to that slot.
5. Read the count again to select the descriptor-size slot, then write \(S\) to that slot.
6. Write the 32-bit value \(S\) at address \(P\).
7. Write a 32-bit zero at address \(P+4\).
8. Write a 32-bit zero at address \(P+8\).

The count is not visibly cached for both descriptor writes. A faithful contract therefore does not replace those count reads with a promise that both writes always use one immutable index. Under ordinary stable-state use they select the same entry; concurrent or aliased mutation is not otherwise specified.

The initialized layout is:

| Address | Width | Value written |
|---|---:|---|
| \(P-4\) | 32 bits | Zero |
| \(P\) | 32 bits | \(S\) |
| \(P+4\) | 32 bits | Zero |
| \(P+8\) | 32 bits | Zero |
| \(E=P+S\) | 32 bits | Zero, written before the three writes starting at \(P\) |

These addresses can overlap. The write order matters: for example, when \(S=0\), the trailing marker and the size word occupy the same address.

The descriptor is published before validation and before the count is incremented.

### 6. Post-write validation and failure

After the writes, the function reads the 32-bit word at \(E\).

It invokes F-a817d8d58c10 if either:

- that word is nonzero; or
- the nominal last-word address \(B+L-4\) compares below \(E\).

The marker condition is evaluated first. If it fails, the bound comparison is not needed to select the failure path.

The bound comparison is an address comparison in the supplied representation. Exact behavior for wrapped or otherwise invalid addresses requires the underlying machine’s pointer/address semantics; it must not be restated as an unrestricted mathematical interval check.

F-a817d8d58c10 is identified as non-returning. On this path:

- Earlier region and descriptor writes have already occurred.
- The explicit accumulated-size update has not occurred.
- F-cc6c9b4a0f68 has not been called.
- The explicit count increment has not occurred.
- The final indirect-invocation gate has not been reached.

There is no visible rollback.

This check is not a pre-write safety barrier. It does not establish that the region was large enough for every earlier write.

### 7. Accumulated-size update, processing call, and count increment

If validation passes:

1. Read the 32-bit field at state offset `0x04`.
2. Write that value plus \(S\) back to `0x04`, with 32-bit result semantics.
3. Invoke F-cc6c9b4a0f68 with the captured state pointer as its explicit argument.
4. Read the count at `0xB0`.
5. Write that count plus one back to `0xB0`.

The increment occurs after F-cc6c9b4a0f68 returns. It therefore uses the count present after that call, not necessarily the count that selected the descriptor.

Likewise, the final value of the accumulated-size field cannot be attributed solely to this function’s addition if the dependency also modifies it. No preservation guarantee for that dependency is available.

### 8. Final indirect-invocation gate

After either successful registration or a capacity-based skip:

1. Read the byte at state offset `0xB4`.
2. If it is not exactly 1, return.
3. Read the pointer at `0xB8`.
4. If the pointer is null, return without dereferencing it.
5. Otherwise, read its first 32-bit word.
6. If that word is zero, return.
7. Otherwise, invoke a target obtained through a separate global indirect-target slot.

Any nonzero first word passes the gate. The word is not visibly used as the target address.

No explicit arguments to the indirect invocation are shown. Its full ABI, effects, and return behavior are unknown. If it returns normally, this function then returns.

## Side effects

Established direct side effects are:

- Region writes at the five locations described above, subject to overlap.
- Descriptor-address and descriptor-size writes when registration is entered.
- An accumulated-size update after validation succeeds.
- A count increment after F-cc6c9b4a0f68 returns.
- Invocation of preliminary dependencies on every entry.
- Conditional invocation of the non-returning failure dependency.
- Conditional invocation of a global indirect target.

The function does not visibly allocate or release memory. It records a region address, but the source does not establish ownership transfer, borrowing duration, reclamation responsibility, or whether any dependency frees or retains memory.

The byte flags and default-region fields are read but not directly written by this body. Dependencies may modify them; their preservation is unknown.

No explicit synchronization operation is visible outside the dependency calls. Locking, exclusion, memory-order guarantees, and reentrancy are not established.

## Dependencies

| Opaque ID or dependency | Visible invocation | Established role and limits |
|---|---|---|
| F-2fd8826557a4 | First operation; no explicit arguments shown | Runs before state-pointer capture. Other effects unknown. |
| F-c0fc6d033caa | After state-pointer capture; no explicit arguments shown | Runs before default selection and capacity inspection. Other effects unknown. |
| F-a817d8d58c10 | Conditional call after initialization writes | Identified as non-returning. Failure mechanism and diagnostics unknown. |
| F-cc6c9b4a0f68 | Receives captured state after the size update | Runs before the explicit count increment. Mutation and ownership behavior unknown. |
| Global indirect target | Invoked only after all final gates pass | Target identity, ABI, effects, and lifetime behavior are documented_only unknowns. |

No dependency is assumed to preserve globals, state fields, region contents, or memory ownership merely because the visible body does not inspect its return value.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. Unless stated otherwise, assume valid writable addresses, non-wrapping arithmetic, stable non-aliased state, and returning dependencies that do not alter the relevant values.

### Ordinary registration

Given base `0x1000`, extent `0x100`, count 0, and final-enable byte 0:

- \(P=\texttt{0x1008}\).
- \(S=\texttt{0xF0}\).
- \(E=\texttt{0x10F8}\).
- The nominal last-word address is `0x10FC`, so the bound condition passes.
- The first descriptor receives address `0x1008` and size `0xF0`.
- The accumulated size increases by `0xF0`.
- F-cc6c9b4a0f68 receives the captured state.
- The count becomes 1.
- No final indirect invocation occurs.

### Unaligned base

Given base `0x1001` and extent `0x100`:

- \(P=\texttt{0x1010}\).
- The base-to-block displacement is 15 bytes.
- \(S=\texttt{0xE8}\).
- \(E=\texttt{0x10F8}\).

The alignment behavior must not be replaced with ordinary round-up-to-8 of the base itself.

### Default substitution

With both inputs zero and the default-enable byte nonzero, calculations use the two default fields. With only one input zero, neither default field is selected by this branch.

### Full table

With unsigned count 20:

- No direct region initialization, descriptor write, accumulated-size update, processing call, or count increment occurs.
- Both preliminary dependencies still run.
- Default selection is still evaluated before the capacity decision.
- The final indirect invocation can still occur if its gates pass.

### Small extent with overlapping markers

Given base `0x1000` and extent `0x10`:

- \(P=\texttt{0x1008}\) and \(S=0\).
- The trailing marker shares the size-word address.
- The post-write marker remains zero.
- The bound comparison passes.
- The write at \(P+8=\texttt{0x1010}\) lies outside the nominal 16-byte region.

If that additional location is writable, the visible checks do not reject the case. This demonstrates that successful validation is not proof of complete write containment.

### Exact final flag value

A final-enable byte of 2 suppresses the indirect invocation. A byte of 1 permits further evaluation, but a null object pointer or zero first word still suppresses it.

### Dependency mutation before increment

If F-cc6c9b4a0f68 returns with count 7, the subsequent explicit increment writes 8, regardless of which count was used earlier for registration.

## Uncertainties

The following are **documented_only** unknowns:

- The provenance and initialization of the shared state and indirect-target slots.
- Contracts, synchronization effects, and memory-management behavior of all dependencies.
- Whether region/state overlap or concurrent mutation is permitted.
- The intended minimum extent and valid address range.
- Fault behavior for invalid, inaccessible, misaligned, or wrapped addresses.
- Exact pointer-comparison semantics outside ordinary valid-address cases.
- The indirect invocation’s complete calling convention and identity.
- The semantic meaning of the accumulated size, descriptor entries, object gate, and default-region flag beyond their visible use.

Allocator registration and notification are plausible interpretations, not additional requirements.

## Evidence

Established behavior comes solely from the supplied static function body: its field offsets, access widths, arithmetic masks, branch conditions, explicit write order, and dependency-call positions.

The non-returning status of F-a817d8d58c10 comes from the supplied call-site annotation. No independent implementation evidence for that dependency is available.

No runtime tests have been performed. The acceptance examples are calculated consequences under their stated assumptions, not observed executions.
