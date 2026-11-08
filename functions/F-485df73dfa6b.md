<!-- Trusted provenance: F-485df73dfa6b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-485df73dfa6b

## Purpose

F-485df73dfa6b coordinates operations on an invocation-local 68-byte temporary region. It passes that region to dependencies, conditionally supplies resource values selected by an object’s mode, computes positioning arguments from geometry and measurements, and performs a six-argument submission through F-4a510ca8d13f. After submission returns, it passes the temporary region to F-282eebd7c413.

Measurement-based positioning and rendering are inferred intent. The established behavior is the sequence of calls, field reads, geometry writes, conditional resource selection, and argument construction described below. The temporary region’s concrete type and the submission’s visual or external meaning are unknown.

The function does not explicitly write the temporary region. Any initialization, mutation, or release associated with that region would be performed by dependencies whose implementations are not supplied.

This contract describes ordinary execution with valid accesses and normally returning dependencies. No runtime tests have been performed.

## Interface

### Inputs and result

| Input | Representation | Established use |
|---|---|---|
| Object address | 32-bit address-bearing value | Locates object fields and an embedded input region |
| Operation argument | Opaque 32-bit value | Forwarded unchanged to F-4a510ca8d13f |
| Bounds address | Address of signed 32-bit elements | Elements zero and two are read only in the two centering branches |

There is no explicit return value or error result.

The supplied representation uses 32-bit addresses and integers. One dependency supplies a 64-bit result. Required address validity, storage extent, and alignment are caller or dependency preconditions; the function does not check them.

### Object layout

| Byte offset | Access | Established role |
|---|---|---|
| `+0x48` | 32-bit read | Alignment flags snapshot; conditionally read again for later resource selection |
| `+0xA4` | Address passed onward | Embedded input supplied to F-0eb164d8526b |
| `+0xE9` | 8-bit read | Fifth submission argument |
| `+0xEB` | Plain-character read | Adjustment subtracted from the computed position |
| `+0x250` | 16-bit read | Initial resource mode and a separately read later mode |

The initial mode is matched by its 16-bit value. The later comparison uses a signed 16-bit interpretation, but the compared value, `0x4D80`, is positive and representable in that width.

The extent and format of the embedded input at `+0xA4` are not established.

### Invocation-local storage

The function provides three regions:

- A 68-byte temporary region whose address is passed to dependencies.
- An original geometry record with four consecutive signed 32-bit components.
- A separate working geometry record with the same layout.

Within either geometry record, components one through four occupy relative byte offsets `+0`, `+4`, `+8`, and `+12`. Their coordinate meanings are not established.

F-412e4635fdb0 receives the original record’s address. The function explicitly copies its four components into the working record, then passes the working record’s address to F-e84807520bd9. It later explicitly modifies the original record’s first component.

The geometry records are distinct storage, but neither is established to remain unchanged across later calls: dependencies have received their addresses and might retain them.

### Global storage

Five separate 32-bit global slots are designated **A**, **B**, **C**, **D**, and **E**. Their contents are opaque resource arguments. Separate slots may contain equal values.

Another global slot holds a pointer to a 32-bit word. The function captures this pointer before later calls and dereferences the captured pointer when constructing the submission arguments.

## Behavior

### Temporary preparation and initial selection

F-0eb164d8526b is called with:

1. The temporary region’s address.
2. The object address plus `0xA4`.

The function itself performs no explicit initialization of the temporary region before this call.

After the call returns, the function reads the mode at object offset `+0x250`.

| Initial mode | Selected global slot |
|---|---|
| `0x4D80` | A |
| `0x4D81` | B |
| `0x4D82` | C |
| `0x4D83` | D |
| `0x4D84` | E |

For a recognized mode, the selected slot’s current 32-bit contents are read and passed with the temporary region to F-ccb5d9751852.

For any other mode, the initial resource-slot read and that call are omitted. Execution continues without an explicit error.

Selection depends on the mode read after F-0eb164d8526b, not necessarily the mode present at entry. No stability guarantee applies to the selected global value after it is read.

### Geometry preparation

The following calls occur next, in order:

1. F-88dd67d704e5 receives the object address and temporary region.
2. F-412e4635fdb0 receives the object address and original geometry record’s address.

After the second call returns, the function copies the original record into the working record in first-to-fourth component order. Each component is read from the original record and written to its corresponding working component.

F-e84807520bd9 then receives:

1. The working record’s address.
2. Integer `4`.
3. Integer `0`.

No returned status is checked. Ordinary execution requires every subsequently consumed geometry component to contain a usable value.

The copy creates separate storage, not a continuing linkage between records. Nevertheless, later dependencies may access either record through retained addresses; preservation is not established.

### Alignment snapshot and measurements

After F-e84807520bd9 returns, the function reads the 32-bit flags at object offset `+0x48`. This saved value controls alignment selection.

It then calls:

1. F-263ec605ff37 with the temporary region, obtaining a signed 32-bit measurement \(M\).
2. F-b38e44b3c7fb with the temporary region, obtaining a 64-bit result.

Let \(L\) be the low 32 bits of the second result, interpreted as a signed 32-bit positioning value. The high 32 bits do not affect the position, submission arguments, or subsequent branch selection.

Alignment uses the saved flags even if either measurement dependency changes the object’s flags. Geometry values used for positioning are read after these measurement calls; they are not immutable snapshots of the values originally supplied by F-412e4635fdb0.

### Branch selection and bounds-read order

The first applicable alignment bit wins:

1. `0x01000000`
2. `0x02000000`
3. `0x04000000`
4. None of these bits

Define:

- \(P\): original first component when consumed for positioning.
- \(R\): original third component when consumed by the third branch.
- \(B_0\): bounds element zero, at byte offset `+0`.
- \(B_2\): bounds element two, at byte offset `+8`.
- \(S=L+M\).

The sum \(S\) is evaluated on every ordinary path, including branches that do not use its result.

Bounds access depends on the selected branch:

- **First branch:** \(B_2\) is read first, then \(B_0\), and both reads precede evaluation of \(S\).
- **Second branch:** \(S\) is evaluated before the bounds subtraction. The relative order of the two bounds reads within that subtraction is not established.
- **Third or default branch:** the function performs no explicit bounds read.

Bounds element one is not explicitly read.

### Position calculation and original-record writes

Let \(X\) be the last value explicitly written to the original record’s first component.

| Selected branch | Calculation and writes |
|---|---|
| `0x01000000` | Compute \(T=(B_2-B_0)-S\). Write \(P+\operatorname{trunc}_0(T/2)\), then add \(M\) to that component and write it again. |
| Otherwise `0x02000000` | Compute \(T=(B_2-B_0)-M\). Write \(P+\operatorname{trunc}_0(T/2)\), then add \(M\) to that component and write it again. |
| Otherwise `0x04000000` | Write \(R-S\). Do not subsequently add \(M\). |
| No selected bit | Add \(M\) to the original first component and write the result. |

Here, \(\operatorname{trunc}_0\) denotes signed integer division with truncation toward zero.

Equivalently, where all evaluated intermediates are representable:

\[
\begin{aligned}
X_{\text{first}} &= \left(P+\operatorname{trunc}_0\left(\frac{(B_2-B_0)-S}{2}\right)\right)+M,\\
X_{\text{second}} &= \left(P+\operatorname{trunc}_0\left(\frac{(B_2-B_0)-M}{2}\right)\right)+M,\\
X_{\text{third}} &= R-S,\\
X_{\text{default}} &= P+M.
\end{aligned}
\]

These are updates to the original record itself, not merely calculations held elsewhere. In each centering branch, there is an intermediate first-component write before the final addition. No dependency call occurs between those two writes.

Positioning arithmetic uses signed 32-bit values. Defined signed-overflow behavior is not established. The formulas specify unambiguous results only when all evaluated intermediates are representable, including the always-evaluated \(S\), bounds differences, position additions, and later subtractions. No saturation or overflow error handling is shown.

### Position delivery

The function next calls F-e5e685741bac with:

1. The temporary region.
2. The working record’s second component.
3. The original record’s updated first component minus the promoted character read at object offset `+0xEB`.

Let \(D\) denote that promoted character value. Without relevant intervening mutation, the third argument is \(X-D\).

The working second component is read for this call rather than frozen after F-e84807520bd9. The relative order of the argument-expression reads is not established.

Whether character bytes `0x80` through `0xFF` promote to negative or nonnegative values is unresolved. The subtraction also requires a representable signed 32-bit result for the arithmetic guarantee above.

### Pointer capture and later resource selection

After F-e5e685741bac returns, the function reads the global pointer slot and saves its pointer value. It does not yet read the pointed-to word.

The function then reads the mode at `+0x250` again:

- If the mode is not `0x4D80`, it skips the later flags read and resource call.
- If the mode is `0x4D80`, it reads the current flags at `+0x48`.
- If bit `0x00000008` is set in that fresh flags value, it reads the current contents of slot E and calls F-ccb5d9751852 with the temporary region and that value.

This decision does not reuse the initial mode or alignment snapshot. Earlier dependency effects may therefore change the outcome.

The later slot-E read is independent of any initial slot-E read. When both occur, the two calls may receive different values.

### Submission and final temporary operation

F-33ca0546dba1 receives the temporary region and integer `0x21`.

After it returns, F-4a510ca8d13f receives:

1. The temporary region.
2. The caller’s unchanged opaque 32-bit operation argument.
3. The 32-bit word read through the previously captured pointer.
4. The working record’s fourth component minus its second component.
5. The byte read at object offset `+0xE9`.
6. Integer `4`.

The working components and pointed-to word are read after F-33ca0546dba1. Their values need not match earlier values.

Changing the global pointer slot after capture does not itself redirect this invocation’s dereference. Changing the word at the captured address can affect the submitted value.

The relative order of the individual reads used to construct these arguments is not established. The source establishes an 8-bit read for the fifth argument, but not a complete dependency-level signedness or argument-extension contract.

After submission returns, F-282eebd7c413 receives the temporary region. The function then returns.

## Side effects

### Established direct effects

The function explicitly:

- Copies four components from original geometry storage into working geometry storage.
- Updates the original geometry record’s first component according to the selected branch.
- Reads the specified object fields, conditionally selected resource slots, conditionally required bounds elements, the global pointer slot, and its captured referent.

It contains **no explicit write into the 68-byte temporary region**. Passing that region to dependencies does not establish that any dependency writes it.

There are no explicit writes to the caller’s object fields, bounds, resource slots, pointer slot, or captured referent.

Geometry writes may be observable during later calls because the record addresses have already been supplied to dependencies. The function does not establish that invocation-local geometry remains private.

### Possible dependency effects and ownership

Dependencies receive the temporary region’s address and may initialize, read, modify, or otherwise act on it. Which actions actually occur is unknown. Temporary-region writes are possible dependency effects, not established direct effects.

Dependencies may also affect object fields, geometry through retained addresses, globals, or other state. No preservation guarantee is established merely because a particular address is absent from a later call’s arguments.

No ownership transfer is established for any supplied address, selected resource value, or opaque argument. The final placement of F-282eebd7c413 does not prove that it frees memory or releases resources.

The temporary and geometry regions have invocation-local lifetimes. Continued validity after return is not promised, and dependency retention behavior is unknown.

### Preconditions and failures

There are no explicit null checks, bounds checks, status checks, recovery paths, or error returns.

Ordinary execution requires:

- Accessible object fields and dependency-valid embedded input.
- Usable geometry values at each consumption point.
- Readable bounds elements zero and two when either centering branch is selected.
- A captured pointer whose referent remains readable until submission argument construction.
- Representable evaluated signed-arithmetic intermediates.

Invalid access or non-returning dependency behavior has no demonstrated recovery contract. If a dependency fails to return normally, subsequent calls—including F-282eebd7c413—are not guaranteed.

## Dependencies

| Opaque function ID | Established call-site role |
|---|---|
| F-0eb164d8526b | Receives the temporary region and embedded input address |
| F-ccb5d9751852 | Receives the temporary region and a selected 32-bit resource value; called zero, one, or two times |
| F-88dd67d704e5 | Receives the object and temporary region |
| F-412e4635fdb0 | Receives the object and original geometry storage |
| F-e84807520bd9 | Receives working geometry storage with integers `4` and `0` |
| F-263ec605ff37 | Supplies the 32-bit measurement |
| F-b38e44b3c7fb | Supplies the 64-bit result whose low half contributes to positioning |
| F-e5e685741bac | Receives the temporary region and positioning arguments |
| F-33ca0546dba1 | Receives the temporary region and `0x21` before submission |
| F-4a510ca8d13f | Receives the six submission arguments |
| F-282eebd7c413 | Receives the temporary region after submission returns |

These roles do not establish complete dependency contracts, memory effects, ownership, or state preservation.

## Acceptance examples

These are conditional static deductions, not executed tests. All assume valid storage, normally returning dependencies, and representable arithmetic intermediates.

### Alignment and positioning

Assume the values at their actual consumption times are:

- Original first component: `100`.
- Original third component: `500`.
- Working second component: `20`.
- \(M=10\), \(L=30\).
- Bounds elements zero and two: `5` and `205`.
- Character adjustment: `3`.

Assume no relevant mutation between the positioning writes and position-argument reads.

| Alignment snapshot | Final original first component | Position arguments after the temporary |
|---|---:|---|
| No alignment bits | `110` | `20`, `107` |
| `0x04000000` | `460` | `20`, `457` |
| `0x02000000` | `205` | `20`, `202` |
| `0x01000000` | `190` | `20`, `187` |
| All three bits | `190` | `20`, `187` |

For the first branch, the original first component is written as `180`, then `190`. For the second branch, it is written as `195`, then `205`. The last row demonstrates precedence rather than combined alignment.

### Negative odd centering difference

With the first branch selected, \(P=100\), \(M=10\), and \((B_2-B_0)-S=-5\), division yields `-2`. The original first component is written as `98`, then `108`. With adjustment `3`, position delivery uses `105`, absent relevant mutation.

### Geometry consumption times

If the working second and fourth components remain `25` and `85` through submission argument construction, the submitted difference is `60`.

If position delivery consumes second component `25`, but a later dependency changes it to `30` while the fourth remains `85`, the submitted difference is `55`.

If a measurement dependency changes the original first component from `100` to `120` through a retained address, a later default-branch calculation with \(M=10\) writes `130`.

### Resource rereads

If the initial mode is `0x4D84`, the initial resource call receives slot E’s value at that read. If an intervening dependency replaces that value, and the later mode is `0x4D80` with fresh flags bit `0x8` set, the second resource call receives the replacement.

An unrecognized initial mode does not prevent a later resource call if the later conditions become true.

### Deferred referent read

If the captured pointer addresses a word containing `7`, and F-33ca0546dba1 changes that word to `9`, submission receives `9`, assuming no subsequent relevant change.

Changing only the global pointer slot does not redirect the already captured pointer.

### Temporary-region effects

The body establishes no particular temporary-region byte value before or after a call. Acceptance must not require a direct initialization or direct cleanup write by F-485df73dfa6b. Any expected temporary contents require an independently established dependency contract.

## Uncertainties

The following remain **documented_only**:

- Temporary type, initialization requirements, contents, and actual dependency writes.
- Resource meanings, equality relationships, and stability.
- Geometry semantics beyond component layout and demonstrated use.
- Dependency access through retained addresses and preservation of object or global state.
- Allocation, ownership transfer, retention, and release behavior.
- Plain-character signedness and dependency-level argument-extension details.
- Signed-overflow behavior and extreme-value results.
- Synchronization, concurrent mutation, and volatile-access semantics.
- Dependency failure behavior and complete externally visible effects.

Rendering and measurement-based alignment remain inferred intent, not established output semantics.

## Evidence

The supplied body establishes access widths and offsets, resource-selection conditions, dependency-call order, alignment precedence, geometry copies, original-record positioning writes, and submission argument construction.

It specifically establishes the first centering branch’s bounds-element-two read before element zero and before the measurement sum. It also establishes that geometry addresses are passed to dependencies before later component reads, and that final working-geometry and captured-referent reads occur after F-33ca0546dba1.

For the temporary region, the body establishes address passing only: it contains no explicit store into that region. Dependency implementations are unavailable, so their writes cannot be promoted into established direct effects.

No dependency implementations, runtime behavior, visual output, or executed tests support this contract. Acceptance examples are conditional deductions from static evidence.
