<!-- Trusted provenance: F-1cd9b34cabc0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1cd9b34cabc0

## Purpose

F-1cd9b34cabc0 selects operations from an event-like record and an object-like receiver, invokes direct or receiver-dispatched dependencies, and returns a 32-bit integer.

**Established static behavior:** initial discriminator values `0x0F` and `0x13` select dedicated paths that return `1` after normal completion. Initial value `0x16` selects a dependency whose nonzero result is returned unchanged; a zero result permits further dispatch using fresh discriminator reads. Other initial values enter that further dispatch directly.

**Inferred intent:** the result may indicate whether an event was handled. This is not an established semantic contract: some paths return `1` without invoking an action-specific dependency, while the initial-`0x16` path can propagate any nonzero 32-bit result.

## Interface

### Inputs and representation

The function receives two address-bearing inputs:

- **Receiver:** an address identifying an object-like region.
- **Event:** an address identifying an event-like record.

The supplied representation uses 32-bit integer values and 32-bit address-bearing words. Offsets in this contract are byte offsets. The exact ABI, alignment requirements, and arithmetic-overflow rules are not independently established.

The receiver regions used directly are:

| Offset | Width or use | Meaning established here |
|---|---|---|
| `0x00` | 32-bit address-bearing word | Dispatch-table address for indirect calls. |
| `0x18` | Address only | Receiver-relative address used in an identity comparison; its contents are not read for that comparison. |
| `0xB0` | 32-bit word | First argument to F-5710a8628cba on one conditional path. |

The event fields used directly are:

| Offset | Width | Use |
|---|---:|---|
| `0x04` | 32 bits | Discriminator, including subsequent fallback reads. |
| `0x08` | 32 bits | First payload argument in either fallback call. |
| `0x0C` | 32 bits | Second payload argument in either fallback call. |
| `0x10` | 32 bits | Third payload argument in the `0x14` fallback call only. |

The event’s leading word is not directly read by this function. Dependencies receiving the event may inspect other fields.

F-682adfd5a561 returns an address identifying a supplementary record. The required supplementary fields depend on the selector:

| Selector | Supplementary field | Use |
|---|---|---|
| `0x0D00` | 32-bit address-bearing word at `0x08` | Compared with receiver-plus-`0x18`. |
| `0x0C00` | 32-bit integer at `0x0C` | Selects an action. |
| `0x0F00` | 32-bit word at `0x08` | Passed to F-I00F0. |

The returned address is consumed, not ignored. It determines which memory supplies the identity-comparison value, action value, or F-I00F0 argument. This does not establish a common semantic type for all supplementary records.

### Shared state

Four external 32-bit storage locations are assigned neutral labels:

| Label | Uses |
|---|---|
| **G-A** | Saved value, fresh action-comparison value, and source of later arguments. |
| **G-B** | Argument to F-I0058. |
| **G-C** | Saved value and separately read argument to F-I0058. |
| **G-D** | Argument to F-5710a8628cba. |

Their initialization, ownership, meaning, legal ranges, and synchronization requirements are unknown.

### Result

On normal completion, the function returns:

- `1` after either dedicated initial-`0x0F` or initial-`0x13` path.
- The exact nonzero 32-bit result of F-I00F0 when reached from initial discriminator `0x16`.
- `1` after either recognized fallback call.
- `0` when fallback recognizes neither `0x14` nor `0x13`.

There is no explicit error return distinct from these dispatch results.

### Preconditions

Requirements are path-dependent. Every directly accessed memory region and invoked call target must be valid when used. In particular:

- The event discriminator must be readable.
- A supplementary-record result must permit the selected field read.
- Indirect-call paths require a readable receiver dispatch-table address and a valid target at the selected slot.
- Fallback payload fields must be readable when their arguments are prepared.
- The matching ordinary-action path requires receiver offset `0xB0` to be readable.

The function provides no explicit null checks, bounds checks, supplementary-record validation, or dispatch-target validation. A fallback path returning `0` need not directly access the receiver.

## Behavior

### Initial classification

The first operation relevant to dispatch is a 32-bit read of the discriminator at event offset `0x04`.

That value selects a dedicated `0x0F`, `0x13`, or `0x16` path, or enters fallback dispatch. An event mutation after selection does not redirect a dedicated initial-`0x0F` or initial-`0x13` path into another branch.

### Initial discriminator `0x0F`

F-682adfd5a561 is called with the event and selector `0x0D00`.

After it returns, the function reads the supplementary address-bearing word at offset `0x08` and compares it with the receiver-plus-`0x18` address.

- Equality causes F-583efa6014bd to be called with the receiver and integer `1`.
- Inequality causes no further call on this path.

The function then returns `1`.

This is an address-identity comparison. The supplementary value is not itself dereferenced, and the contents at receiver offset `0x18` are not compared.

### Initial discriminator `0x13`

F-682adfd5a561 is called with the event and selector `0x0C00`. After that call returns, the visible read sequence is:

1. Read and save G-C.
2. Read and save G-A.
3. Read the supplementary action at offset `0x0C`.

The saved values therefore reflect reads performed after F-682adfd5a561 completes. They are not snapshots from function entry.

#### Special actions

Action `0x60A4` selects F-817e3c69adf5 directly.

For an action other than `0x60A4`, the visible selection also tests whether subtracting `0x6000` from the action yields `0xA6`. For values where this signed 32-bit arithmetic is representable, that condition selects action `0x60A6`.

Either special condition causes one call to F-817e3c69adf5 with the receiver, followed by result `1`.

The special conditions take precedence over comparison with G-A. When either holds, the fresh G-A comparison and the ordinary-action call sequence are skipped. The earlier saved reads of G-C and G-A still occur.

#### Ordinary action matching G-A

If neither special condition holds, the action is compared with a fresh read of G-A. The saved G-A value is not used for this comparison.

On equality, the following sequence occurs:

1. **F-I0058** receives:
   - Receiver.
   - A fresh read of G-C.
   - A read of G-B.

2. After F-I0058 returns, **F-I0074** receives:
   - Receiver.
   - Saved G-C.
   - Saved G-A minus `0x220`.
   - Integer `0`.

3. After F-I0074 returns, **F-5710a8628cba** receives:
   - The 32-bit value at receiver offset `0xB0`.
   - A read of G-D.
   - Saved G-A.

4. If F-5710a8628cba returns zero, three calls follow in order:
   - **F-I00A0**, receiving the receiver and integer `0`.
   - **F-6ac70711bbb5**, with no explicit arguments.
   - **F-970e8a682a82**, with no explicit arguments.

5. The function returns `1`.

A nonzero F-5710a8628cba result suppresses all three calls in step 4. That result is not returned to the caller.

Saved and fresh values have distinct roles. A dependency may change G-C or G-A, but such a change does not replace the already saved values used by F-I0074 and F-5710a8628cba. Conversely, the argument reads for F-I0058 and the later read of G-D must not be treated as entry-time snapshots.

The exact evaluation order among independent arguments and dispatch-target reads within a single invocation is not established.

#### Unmatched ordinary action

If neither special condition holds and the action differs from the fresh G-A value, the function makes no further dependency call and returns `1`.

Thus, result `1` does not prove that an action-specific operation occurred.

### Initial discriminator `0x16`

F-682adfd5a561 is called with the event and selector `0x0F00`.

The function reads the 32-bit word at supplementary offset `0x08` and calls F-I00F0 with the receiver and that word.

- A nonzero F-I00F0 result is returned unchanged.
- A zero F-I00F0 result enters fallback dispatch.

The F-682adfd5a561 result determines the address of the field supplying F-I00F0’s second argument. F-I00F0’s own result separately determines whether dispatch continues.

### Fallback dispatch

Fallback is entered for initial discriminators other than `0x0F`, `0x13`, and `0x16`, or after F-I00F0 returns zero.

The function reads the discriminator again and compares it with `0x14`.

- If equal, F-I0078 receives the receiver and the 32-bit event fields at `0x08`, `0x0C`, and `0x10`, in that argument order. The function returns `1`.
- If unequal, the function performs another discriminator read and compares that read with `0x13`.
  - Equality invokes F-I005C with the receiver and the 32-bit event fields at `0x08` and `0x0C`, in that argument order, then returns `1`.
  - Inequality returns `0`.

The argument order does not establish the temporal order of independent payload reads.

The fallback comparisons use distinct discriminator reads in the supplied representation. They must not be replaced in this contract by a single assumed-stable value. Whether external mutation between those reads is valid or observable under the original execution model remains unknown.

Fallback selection of `0x13` invokes F-I005C; it does not execute the dedicated initial-`0x13` supplementary-record path.

## Side effects

The visible function body performs memory reads and dependency calls. It contains no explicit write to the receiver, event, supplementary records, or shared storage.

Dependencies may nevertheless mutate accessible state or produce other externally visible effects. No dependency is established to preserve globals, receiver contents, the event, or the dispatch table.

Each indirect call uses the receiver’s dispatch table at that call site. An earlier dependency may therefore affect a later target if such mutation is permitted by the surrounding program. No single dispatch-table snapshot is established across the call sequence.

No allocation, deallocation, reference-count adjustment, or ownership transfer is explicit in this body. Supplementary addresses are used without an explicit release here. This does not establish whether those records are borrowed, owned, retained elsewhere, or released by a dependency.

There is no explicit recovery for invalid access, invalid call targets, dependency failure, or a dependency that does not return normally. Normal return claims require completion of every selected call and access.

## Dependencies

Direct dependency IDs identify supplied dependencies. The F-I identifiers are contract-local opaque names for indirect operations. Their associated byte slots describe dispatch layout, not verified global function identities.

| Dependency | Visible invocation | Treatment of result |
|---|---|---|
| F-682adfd5a561 | Event plus selector `0x0D00`, `0x0C00`, or `0x0F00`. | Consumed as the supplementary-record address; subsequently dereferenced at the selected field offset. |
| F-583efa6014bd | Receiver and `1` after an identity match. | Ignored. |
| F-817e3c69adf5 | Receiver for either special action. | Ignored. |
| F-5710a8628cba | Receiver field `0xB0`, G-D, saved G-A. | Explicitly tested against zero; zero enables three further calls. |
| F-6ac70711bbb5 | No explicit arguments, after F-I00A0. | Ignored. |
| F-970e8a682a82 | No explicit arguments, after F-6ac70711bbb5. | Ignored. |
| F-I0058, slot `0x58` | Receiver, fresh G-C, G-B. | Ignored. |
| F-I005C, slot `0x5C` | Receiver and two fallback payload fields. | Ignored. |
| F-I0074, slot `0x74` | Receiver, saved G-C, saved G-A minus `0x220`, zero. | Ignored. |
| F-I0078, slot `0x78` | Receiver and three fallback payload fields. | Ignored. |
| F-I00A0, slot `0xA0` | Receiver and zero. | Ignored. |
| F-I00F0, slot `0xF0` | Receiver and supplementary field `0x08`. | Explicitly tested against zero; nonzero is returned unchanged. |

Three dependency return values are consumed: the address from F-682adfd5a561 and the integers from F-5710a8628cba and F-I00F0. Only the latter two are explicitly tested against zero.

“Ignored” concerns return-value consumption, not side effects or the importance of normal completion.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, memory is valid, state is stable, dependencies return normally, and arithmetic is representable.

1. **Identity match.** With initial discriminator `0x0F`, F-682adfd5a561 receives selector `0x0D00`. If its returned record contains receiver-plus-`0x18` at offset `0x08`, F-583efa6014bd receives the receiver and `1`. Final result: `1`.

2. **Identity mismatch.** Under the same initial discriminator, a different supplementary field value suppresses F-583efa6014bd. Final result remains `1`. The returned supplementary address still matters because it selects the field being compared.

3. **Special-action precedence.** With initial discriminator `0x13` and supplementary action `0x60A4`, F-817e3c69adf5 is called even if G-A also equals `0x60A4`. The fresh G-A comparison and ordinary-action calls are skipped. Final result: `1`.

4. **Second special action.** Action `0x60A6` produces a representable difference of `0xA6` after subtracting `0x6000`, selecting F-817e3c69adf5. Final result: `1`.

5. **Ordinary match with zero check.** Let saved G-A be `0x7000`, saved G-C be `9`, and action equal the fresh G-A read. F-I0074 receives receiver, `9`, `0x6DE0`, and `0`. If F-5710a8628cba returns zero, F-I00A0, F-6ac70711bbb5, and F-970e8a682a82 follow in that order. Final result: `1`.

6. **Saved values survive dependency mutation.** Under the preceding selection, suppose F-I0058 changes G-A and G-C. F-I0074 still receives saved G-C `9` and adjusted saved G-A `0x6DE0`; F-5710a8628cba still receives saved G-A `0x7000`. Later reads are not assumed preserved.

7. **Nonzero check result is not propagated.** If F-5710a8628cba returns `5`, the three zero-enabled calls are skipped. Final result is `1`, not `5`.

8. **Supplementary argument and exact propagation.** With initial discriminator `0x16`, the word at offset `0x08` of the record returned by F-682adfd5a561 becomes F-I00F0’s second argument. If F-I00F0 returns `-7`, the function returns `-7` without fallback.

9. **Mutation before fallback.** If F-I00F0 changes the discriminator to `0x14` and returns zero, fallback invokes F-I0078 with the three payload fields and returns `1`.

10. **Fallback `0x13` is distinct.** If the initial-`0x16` dependencies leave discriminator `0x13` and F-I00F0 returns zero, fallback invokes F-I005C. There is no additional F-682adfd5a561 call with selector `0x0C00`.

11. **Ordinary action mismatch.** With initial discriminator `0x13`, a non-special action differing from fresh G-A causes no action-specific call after supplementary acquisition. Final result: `1`.

12. **Unrecognized stable discriminator.** With discriminator `0x12` throughout, no dependency is called and the result is `0`.

## Uncertainties

The following remain **documented_only**:

- Domain meanings of discriminators, actions, shared state, and receiver operations.
- Dependency ownership, lifetime, allocation, release, mutation, and failure contracts.
- Exact ABI, alignment requirements, and independent platform confirmation of the displayed 32-bit representation.
- Signed-overflow behavior for action minus `0x6000` and saved G-A minus `0x220`. Overflow cases are not assigned wraparound behavior here.
- Exact evaluation order of independent arguments and target-address reads within an invocation.
- Whether repeated ordinary reads correspond to distinct observable hardware transactions.
- Permitted concurrency, synchronization, and external mutation.
- Behavior following invalid memory access, invalid indirect targets, or nonlocal dependency exits.

## Evidence

This contract is based solely on static inspection of the supplied function representation and dependency-ID mapping. That evidence establishes field widths as represented, offsets, selectors, branch precedence, saved versus fresh reads, call order, consumed return values, and return selection.

In particular, the supplementary-acquisition return is visibly used as an address for subsequent field access. It is distinct from the two dependency results explicitly compared with zero and from the remaining ignored return values.

Receiver, event, action, and supplementary-record terminology describes inferred roles rather than verified domain semantics. Dependency implementations, caller contracts, and executable traces were unavailable. No runtime tests have been performed.
