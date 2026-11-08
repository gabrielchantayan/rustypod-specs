<!-- Trusted provenance: F-b4b3f9b8a15c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b4b3f9b8a15c

## Purpose

F-b4b3f9b8a15c coordinates a lookup, classification of supplied material, a record-dependent limit check, validation, and a sequence of processing calls. It returns a 32-bit status.

The visible branches distinguish lookup failure, parsing or first-classification rejection, unsupported second classification, an exceeded limit, a propagated validation status, and final processing failure. Success requires validation to return zero and the final processing dependency to return nonzero.

The apparent application-level purpose is to process supplied material in relation to a selected record. That purpose is inferred from argument flow, not established by dependency implementations. Terms such as “lookup,” “parsing,” “validation,” and “workspace” describe call-site roles; they do not establish the underlying data formats or transformations.

This contract describes the supplied static representation. It does not establish runtime behavior beyond what that representation supports.

## Interface

### Inputs

The function accepts five arguments represented as 32-bit quantities.

| Position | Established use |
|---|---|
| First | Passed to F-4006b4478b82. On the later processing path, also passed to F-e25ca5aeb3ea, then used to form another argument displaced by 32 bytes. |
| Second | Passed unchanged to F-780d6593c541. |
| Third | Passed unchanged to F-780d6593c541. |
| Fourth | Passed unchanged as the first argument to F-84370e32ba69. |
| Fifth | Passed unchanged as the second argument to F-84370e32ba69. |

The first argument is represented as a signed 32-bit integer; the other four are unspecified 32-bit values. The caller does not establish whether these values denote handles, addresses, lengths, or other application objects.

No explicit null, length, alignment, access, or ownership checks occur at this level. Acceptance of the first input by lookup does not prove that its later address-like uses are valid.

### Result

The returned status is a 32-bit integer.

| Result | Selecting condition |
|---|---|
| 0 | Validation returns zero, all subsequent dependencies return normally, and the final dependency returns nonzero. |
| 1 | Lookup returns zero. |
| 5 | Parsing returns zero, or parsing returns nonzero but the first classification does not match the required value. |
| 6 | Final processing returns zero. |
| 18 | The record’s unsigned 16-bit limit is less than the primary workspace’s unsigned 16-bit leading value. |
| 20 | Parsing and the first classification check pass, but the second classification is not 6. |
| Any nonzero status from F-780d6593c541 | Propagated unchanged when validation returns nonzero. |

These categories are not numerically disjoint. For example, status 6 can originate from validation or from final processing failure. The outward result alone does not distinguish those origins.

### Temporary storage and visible record fields

The invocation provides eight distinct 32-byte temporary regions, one 48-byte temporary region, and two classification-output regions of four bytes each.

The following descriptive names identify their roles:

- **Primary workspace:** 32 bytes, represented as sixteen unsigned 16-bit elements.
- **Parsed A, parsed B, parsed C:** three 32-byte regions passed to parsing.
- **Auxiliary D, auxiliary E, auxiliary F, auxiliary G:** four further 32-byte regions.
- **Wide workspace:** 48 bytes.
- **First classification output, second classification output:** each represented as two signed 16-bit elements.

These names do not imply read/write direction. In particular, passing a region as an earlier argument does not establish that it is a source or read-only input.

A nonzero lookup result is retained as the record reference. The function directly reads these fields relative to that reference:

| Byte offset | Width and interpretation established here | Use |
|---|---|---|
| 4 | Unsigned 16-bit | Limit for the primary workspace’s leading value. |
| 6 | 16-bit; signedness not established | First argument to an early F-5719df83e333 call. |
| 12 | 16-bit; signedness not established | First argument to a later F-5719df83e333 call. |

The remainder of the record layout is unknown. Dependencies receiving the reference may access additional state. Byte order, alignment requirements, and any argument-extension convention for the unspecified 16-bit fields are not established.

## Behavior

### Preparation precedes every lookup

The function first calls F-c41e7e373603 separately on the eight 32-byte regions, in this order:

1. Primary workspace.
2. Parsed A.
3. Parsed B.
4. Parsed C.
5. Auxiliary D.
6. Auxiliary E.
7. Auxiliary F.
8. Auxiliary G.

It then calls F-5649a8925096 on the wide workspace.

All nine calls precede lookup, including on a path that ultimately returns status 1. Their return values do not control a visible branch.

These calls appear to prepare temporary storage, but their actual writes and coverage are unknown. They are not established as zero-filling operations. Neither classification-output region receives a separate preparation call before parsing.

### Lookup gates parsing

F-4006b4478b82 receives the first input.

A zero result selects status 1. Parsing, classification reads, direct record-field reads, validation, and subsequent processing do not occur.

A nonzero result becomes the retained record reference. There is no second lookup or visible refresh of that reference.

### Parsing and classification have ordered rejection conditions

For a nonzero lookup result, F-84370e32ba69 receives these arguments, in order:

1. Fourth input.
2. Fifth input.
3. First classification output.
4. Second classification output.
5. Parsed A.
6. Parsed B.
7. Parsed C.

A zero parser result selects status 5 without directly inspecting either classification output.

After a nonzero parser result, the leading signed 16-bit element of the first classification output is compared with −4083, whose 16-bit bit pattern is 0xF00D. A mismatch selects status 5 without inspecting the second classification output.

Only after the first classification matches is the leading signed 16-bit element of the second classification output examined. A value other than 6 selects status 20.

The second 16-bit element of each classification-output region is not directly inspected by this function. Whether parsing writes those elements, or initializes all storage consumed later, remains unknown.

### Derivation and the unsigned limit check

After both classification checks pass, the following operations occur in order:

1. F-e25ca5aeb3ea receives the ordered pair: parsed B, wide workspace.
2. The 16-bit record field at byte offset 6 is read.
3. F-5719df83e333 receives that field value, then the primary workspace.
4. F-5d01188e4589 receives the primary workspace, then the wide workspace.
5. The unsigned 16-bit field at record offset 4 is compared with the unsigned 16-bit leading element of the primary workspace.

The workspace value used in the comparison is its value after F-5d01188e4589 returns. It is not necessarily the value left by F-5719df83e333.

If the record limit is strictly less than the primary value, status 18 is selected. Equality passes. Both compared values range from 0 through 65,535; a set high bit does not make either value negative.

The supplied representation does not establish which operand of this comparison is read first. It does establish that both are observed after the preceding dependency call. There is no entry-time snapshot or explicit stability check.

The record reads at offsets 6 and 4 occur at different stages. Earlier dependency effects may therefore influence later observed values. No assumption that a dependency preserves globals or reachable record state is part of this contract.

### Validation can terminate processing or alter later inputs

If the limit check passes, F-780d6593c541 receives:

1. Second input.
2. Third input.
3. Primary workspace.

A nonzero result is returned unchanged. None of the later processing calls occur on that path.

A zero result permits processing to continue. The primary workspace is passed by address, and preservation of its contents is not established. Later processing therefore receives its post-validation state, not necessarily the state that passed the limit check. The function performs no second limit check.

### Accepted processing follows a fixed call sequence

After validation returns zero, the function performs these operations in order:

1. F-52d507a33d28 receives the primary workspace and retained record reference.
2. The 16-bit record field at byte offset 12 is read.
3. F-5719df83e333 receives that field value and auxiliary D.
4. F-477fb87c1603 receives auxiliary D, auxiliary E, parsed A, and the record reference.
5. F-e25ca5aeb3ea receives parsed B and the wide workspace.
6. F-7913cbe7fa05 receives the primary workspace and the wide workspace.
7. F-ab130655f6ab receives the wide workspace, the record reference, and integer 1.
8. F-e25ca5aeb3ea receives the first input and auxiliary F.
9. F-e25ca5aeb3ea receives the first input displaced by 32 bytes and auxiliary G.
10. F-07cbcf070c14 receives auxiliary F, auxiliary G, the wide workspace, and the record reference.
11. F-4c970da72680 receives parsed B, parsed C, auxiliary F, auxiliary G, auxiliary D, auxiliary E, and the record reference.

The two calls with the ordered pair “parsed B, wide workspace” are separate invocations. Neither argument is established as read-only or write-only. The later call cannot be replaced contractually with reuse of an earlier result: either region or other dependency-visible state may have changed.

The offset-12 read occurs after F-52d507a33d28 returns. If that dependency changes the field, the later read can observe the changed value. No earlier snapshot is used.

The displacement applied to the first input is 32 bytes, not 32 elements of an established object type. The visible representation contains no explicit overflow guard. Behavior at a signed-arithmetic boundary or an invalid address is unresolved.

Returns from operations 1 through 10 are not used as explicit success or failure gates. If those dependencies return normally, execution continues.

### Final processing is reduced to a Boolean outcome

A zero result from F-4c970da72680 selects status 6. Any nonzero result selects status 0.

The final dependency’s nonzero numerical value is not propagated. No visible operation follows this decision other than returning the selected status.

## Side effects

### Observable ordering and writes

Preparation calls always precede lookup. Parsing requires lookup success. Validation requires parser success, both classification matches, and a passing unsigned limit comparison. The accepted-processing sequence requires validation to return zero.

The function performs no explicit direct store into the located record or caller-provided memory. Writes to such state, if any, occur through dependencies. The order and extent of reads or writes inside those dependencies are unknown.

Passing two arguments to F-e25ca5aeb3ea establishes their order only. It does not establish copying, conversion, transfer direction, or nonmutation of either argument. This restriction applies to all four invocations, including those involving the first input.

### Failure is not transactional

No rollback, retry, or compensating operation is visible.

A parser or validation rejection is not established as side-effect-free. Status 18 follows several preparation and processing-related calls. A final status 6 occurs only after the entire accepted-processing sequence has reached its final call.

Consequently, failure status does not imply unchanged record contents, unchanged caller-visible storage, or unchanged global state.

### Ownership, lifetime, and safe-use preconditions

The temporary regions are invocation-local storage available during the calls made by this function. No explicit allocation, release, or ownership-transfer operation appears at this level.

That absence does not establish that dependencies allocate nothing, release nothing, retain no addresses, or transfer no resources. Those behaviors remain unknown.

For defined ordinary completion, every reached access must be valid, and each dependency must respect the actual capacities and usable lifetimes of storage it receives. The function does not enforce these requirements. No non-aliasing guarantee is established for caller-supplied values or dependency-accessible state.

## Dependencies

The following are call-site contracts only, not complete dependency specifications.

| Opaque ID | Established role |
|---|---|
| F-c41e7e373603 | Receives each of the eight 32-byte regions once before lookup. |
| F-5649a8925096 | Receives the 48-byte wide workspace before lookup. |
| F-4006b4478b82 | Receives the first input; zero selects status 1, nonzero supplies the retained record reference. |
| F-84370e32ba69 | Receives the fourth and fifth inputs, two classification outputs, and three parsed regions. Zero selects status 5. |
| F-e25ca5aeb3ea | Called four times on the path reaching final processing, with the ordered argument pairs described above. No source/destination direction is established. |
| F-5719df83e333 | Receives a 16-bit record-field value and a 32-byte region, twice on the path reaching final processing. |
| F-5d01188e4589 | Receives primary and wide workspaces before the unsigned limit check. |
| F-780d6593c541 | Receives the second input, third input, and primary workspace; a nonzero result is propagated. |
| F-52d507a33d28 | Receives primary workspace and record before the offset-12 read. |
| F-477fb87c1603 | Receives auxiliary D, auxiliary E, parsed A, and record. |
| F-7913cbe7fa05 | Receives primary and wide workspaces during accepted processing. |
| F-ab130655f6ab | Receives wide workspace, record, and integer 1. |
| F-07cbcf070c14 | Receives auxiliary F, auxiliary G, wide workspace, and record. |
| F-4c970da72680 | Receives seven arguments in the stated order; zero selects status 6, nonzero selects status 0. |

No dependency is assumed to preserve globals, preserve pointed-to storage, or implement ownership semantics absent additional evidence.

## Acceptance examples

These are conditional deductions from static control flow, not executed tests. Each assumes valid reached accesses and normal return from every reached dependency.

1. **Lookup miss.** Lookup returns zero. The result is 1. All nine preparation calls have occurred; parsing has not.

2. **Parser rejection.** Lookup succeeds and parsing returns zero. The result is 5, with no direct classification-output reads.

3. **First classification mismatch.** Parsing returns nonzero and the first leading classification value is −4082. The result is 5. The second classification is not inspected.

4. **Second classification mismatch.** Parsing returns nonzero, the first classification has bit pattern 0xF00D, and the second is 5. The result is 20. The derivation calls and validation are skipped.

5. **Limit exceeded.** Both classifications pass. At comparison time, the record limit is 9 and the primary leading value is 10. The result is 18; validation is not called.

6. **Equality accepted, validation propagated.** Both compared values are 10. Validation is reached. If it returns 23, the function returns 23 without entering accepted processing.

7. **Unsigned boundary.** The limit is 32,767 and the primary value is 32,768. The result is 18. A signed interpretation of the workspace value would contradict the established comparison.

8. **Final failure.** Validation returns zero and final processing returns zero. The result is 6. All preceding accepted-processing calls have already occurred; no visible rollback follows.

9. **Nonzero final result.** Under the same prerequisites, final processing returns −1 as a 32-bit value. The function returns 0 because the final decision distinguishes only zero from nonzero.

10. **Later record state is observed.** F-52d507a33d28 changes the offset-12 field, and nothing changes it before the following read. The second F-5719df83e333 invocation receives the changed field value.

11. **Neither paired argument is protected.** Suppose the first F-e25ca5aeb3ea invocation changes parsed B as well as the wide workspace. Such mutation is not excluded by the caller. The repeated invocation later receives the then-current regions; preservation of parsed B cannot be an acceptance requirement derived from this function.

## Uncertainties

The following remain **documented_only**:

- Application meanings of the five inputs, classifications, statuses, and record fields.
- Dependency read/write extents, transformations, initialization coverage, and internal operation order.
- Whether either argument of F-e25ca5aeb3ea is read-only, write-only, or both; copying semantics are not established.
- Byte order, alignment, valid address ranges, and aliasing restrictions.
- Required readable or writable extent associated with the first input. A 32-byte displacement does not establish a 32-byte access.
- Overflow, address-wrap, and invalid-pointer behavior when forming the displaced first-input argument.
- Whether parsing supplies every value later consumed.
- Ownership, allocation, release, pointer retention, and global-state behavior of dependencies.
- Fault, exception, nonreturning-call, and invalid-memory behavior.
- Concurrency guarantees or synchronization between record reads.
- Whether ignored dependency returns encode errors.
- Exact ABI treatment of the unspecified 16-bit record-field arguments.

Preparation, parsing, validation, and processing are inferred role descriptions. They do not resolve these unknowns.

## Evidence

Established static evidence consists of the supplied representation’s argument flow, temporary-storage sizes, call ordering, short-circuit conditions, direct field offsets and widths, unsigned limit comparison, constants, and return selection.

The representation establishes ordered argument pairs for F-e25ca5aeb3ea, not source/destination roles or copying behavior. No dependency implementation was supplied to strengthen that conclusion.

No external interface documentation, runtime traces, or execution results were provided. No runtime tests have been performed. The acceptance examples are conditional consequences of the visible control flow, not proof of memory safety, dependency correctness, transactional behavior, or execution on a real system.
