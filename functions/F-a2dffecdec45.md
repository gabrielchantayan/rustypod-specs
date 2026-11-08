<!-- Trusted provenance: F-a2dffecdec45; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a2dffecdec45

## Purpose

F-a2dffecdec45 conditionally reuses or initializes an object in a destination context. It can also process source children, including on a path that creates no object for the current source.

The established contract concerns descriptor selection, a reuse opportunity, writable-storage acquisition, field updates, dependency calls, and recursive traversal. It does not establish a deep-copy operation, a particular hierarchy model, or resource ownership.

“Source,” “destination,” “descriptor,” “object,” “anchor,” and “continuation” are descriptive labels for observed roles. Their application-level meanings remain inferred intent.

## Interface

### Inputs and result

The represented interface accepts four 32-bit quantities:

| Input | Contract |
|---|---|
| Source address | Identifies the structure whose owner, flags, descriptor or child link, and other fields are read. |
| Destination address | Identifies the context used for selection, storage acquisition, and subsequent processing. |
| Optional placement-state address | Zero or the address of a two-word record. This original address is passed to F-493743038bd6 when an object is initialized. |
| Opaque recursive value | Forwarded unchanged as the fourth argument of every direct recursive call. It also initially occupies the invocation’s local continuation word. |

The result is a 32-bit pointer-valued quantity. A successful reuse returns the exact nonzero value supplied by F-3931e02c2857. Otherwise, successful storage acquisition leads to return of the acquired object address, provided all subsequent processing returns normally.

Zero has several meanings: descriptor resolution failed, storage acquisition failed, or the current invocation intentionally created no object. In the last case, children may still have been processed successfully. Child results do not become the parent’s return value.

### Widths, validity, and arithmetic

Addresses and ordinary structure words use the represented 32-bit model. Flag accesses are one byte. Offsets in this contract are byte offsets.

The routine supplies no explicit bounds, alignment, pointer-validity, or address-overflow checks. Every executed access requires valid storage of the appropriate width. Address additions do not establish portable behavior when an address calculation overflows.

The routine does not perform size calculations, counters, or signed ordering comparisons. Its relevant scalar decisions are zero tests, word equality, and byte-mask tests.

No disjointness guarantee is established. Storage designated by different inputs or returned by a dependency may overlap. Later reads must therefore reflect preceding writes and dependency effects rather than an assumed immutable entry snapshot.

### Observed layouts

| Region | Offset | Use |
|---|---:|---|
| Source | `0x00` | Owner/context address captured at entry. |
| Source | `0x10` | Word copied to object offset `0x18`. |
| Source | `0x1D` | Flag byte; bit `0x01` selects behavior at several distinct times. |
| Source | `0x20` | Descriptor input during initially unmarked processing; first-child address during traversal. |
| Source | `0x28` | Word supplied to F-12d234fad301. |
| Captured owner/context | `0x0C` | Word compared with the destination’s corresponding word. |
| Captured owner/context | `0xCC` | Address supplied to F-12d234fad301. |
| Destination | `0x0C` | Context-comparison word. |
| Destination | `0x40` | Argument to reuse lookup or default placement anchor. |
| Destination | `0x48` | Address supplied to final processing. |
| Destination | `0xCC` | Address supplied to F-12d234fad301. |
| Destination | `0x18C` | Flag byte; bit `0x10` permits storage acquisition for an initially marked source. |
| Destination | `0x18D` | Flag byte; bit `0x10` permits a reuse lookup for an initially unmarked source. |
| Descriptor | `0x24` | Link tested for reuse, later read and replaced during ordinary initialization. |
| Child | `0x08` | Next-child address, read after that child’s recursive processing. |

Object storage has the following observed fields:

| Offset | Use |
|---:|---|
| `0x10` | Read during ordinary initialization. |
| `0x18` | Receives the source word at `0x10`. |
| `0x1D` | Bit `0x01` is set during marked initialization. |
| `0x20` | Receives the selected descriptor during ordinary initialization. |
| `0x24` | Receives the object’s `0x10` word during ordinary initialization; later supplies child continuation. |
| `0x28` | Receives the descriptor’s previous link, or is passed by address to F-12d234fad301. |

A placement record consists of a 32-bit anchor at `0x00`, followed by a 32-bit continuation at `0x04`. A default anchor obtained through destination offset `0x40` must support a word read at `0x24`.

## Behavior

### Entry state and independent flag observations

The source’s owner/context address is captured before the initial source-flag decision. Subsequent transfer processing uses this captured address even if the source’s owner field changes.

The invocation also has a local two-word placement record. Initially, its anchor contains the original optional placement-state address itself—not the contents of that address—and its continuation contains the fourth argument. Traversal, if reached, replaces these initial values.

The source flag is not treated as a stable mode. Its bit `0x01` is observed at entry, again after successful storage acquisition and the common field copy, and again after attachment. Dependencies and overlapping writes can make these observations disagree.

### Descriptor selection and reuse

An initially clear source flag selects descriptor processing.

If the destination and captured owner/context words at `0x0C` are equal, the source word at `0x20` is used directly as the descriptor address. If they differ, that source word is passed to F-53a23d4b077c. A zero result from this dependency ends the invocation immediately with zero.

The direct-selection path has no corresponding null rejection. A zero descriptor on that path does not produce an orderly early return; the subsequent descriptor-field access still occurs.

Reuse depends first on the descriptor word at `0x24`. When this word is zero, destination flag byte `0x18D` is not read for the reuse decision. When it is nonzero, destination bit `0x10` at that offset determines whether F-3931e02c2857 is called with the destination word at `0x40` and the selected descriptor address.

A nonzero lookup result is returned immediately. Storage acquisition, object initialization, attachment, traversal, and final processing are all bypassed. A zero lookup result instead permits storage acquisition.

The descriptor link observed for this decision is not retained as the link later copied into an object. Ordinary initialization performs a separate read.

### Initially marked processing

An initially set source flag bypasses descriptor selection and reuse. The selected descriptor remains zero.

Destination bit `0x10` at `0x18C` controls whether storage is requested. If clear, no object is obtained for this invocation; placement initialization and child traversal follow directly. There is no intervening source-flag recheck governing entry to traversal on this path.

If the permission bit is set, storage acquisition proceeds as it does after unsuccessful or inapplicable reuse.

### Storage acquisition and object initialization

F-704d91f94e04 receives the destination address. A zero result ends the invocation immediately, without attachment, traversal, or final processing.

For a nonzero result, the source word at `0x10` is copied to object offset `0x18` before the setup-time source-flag observation. The returned storage is not established to be fresh, disjoint, or initialized. In particular, this common write can affect the next flag observation if addresses overlap.

A clear setup-time flag selects ordinary initialization. The object’s word at `0x10` is copied to its `0x24` field before the selected descriptor address is written to object offset `0x20`. The descriptor’s then-current `0x24` word is subsequently copied to object offset `0x28`; only afterward is the descriptor’s `0x24` field replaced with the current object address.

These ordering constraints are observable under aliasing. The descriptor value copied into the object can reflect earlier initialization writes and need not equal the value tested for reuse. Ordinary initialization does not explicitly clear the object’s flag bit.

If the source was initially marked but is clear at setup time, the selected descriptor is still zero. No delayed descriptor resolution occurs. Ordinary initialization therefore attempts the descriptor-field access at address `0x24`; there is no protective check for this transition.

A set setup-time flag selects marked initialization. The object flag byte at `0x1D` is read and rewritten with bit `0x01` set, preserving its other bits. F-12d234fad301 then receives the captured owner/context address plus `0xCC`, a current read of source offset `0x28`, the destination address plus `0xCC`, and the address of object offset `0x28`.

Marked initialization contains no explicit field-targeted initialization of object offsets `0x20` or `0x24`. Their later values depend on existing contents, overlapping writes, and dependency effects.

### Attachment and traversal eligibility

After either initialization branch, F-493743038bd6 receives the current object address, the original optional placement-state address, and the value `1`. It does not receive the address of the invocation’s local placement record.

The source flag is then read again. A clear bit skips traversal; a set bit selects traversal. Neither the entry flag nor the setup-time flag substitutes for this observation.

Attachment is not assumed to preserve source flags, links, placement storage, descriptors, or globals.

### Placement state and child traversal

Traversal uses the invocation’s local adjacent anchor and continuation words.

When an object exists, its word at `0x24` is read into continuation before the local anchor is assigned the object address.

Without an object, a nonzero placement-state input supplies the continuation from offset `0x04` before supplying the anchor from offset `0x00`. With a zero placement-state input, the destination word at `0x40` supplies the anchor, and that anchor’s word at `0x24` supplies continuation. There is no explicit null-anchor rejection.

These initialization operations do not directly write through the caller’s placement-state pointer.

Only after placement initialization is the source word at `0x20` read as the first-child address. Each nonzero child is processed recursively by F-a2dffecdec45 with the same destination, the address of this local placement record, and the original fourth argument.

The recursive result replaces local continuation, including when the result is zero. The processed child’s next-child word at `0x08` is read only after the recursive call returns and continuation has been updated. Consequently, recursion can change which sibling is visited next. A zero child result does not itself terminate sibling traversal.

The parent makes no explicit per-child replacement of its local anchor. Nevertheless, the placement record is exposed to recursive processing, and reachable dependencies are not assumed to preserve it.

No cycle detection, visited set, or depth limit is visible. Finite traversal and sufficient stack capacity are external requirements.

### Final processing

If a current object exists, the local anchor is cleared before F-ebd0ee53699f is called. Its arguments are the destination address plus `0x48`, the value obtained from the final-processing global datum at this stage, and the current object address.

The global value is read after attachment and any traversal. It must not be replaced by an assumed constant or by a value captured before those operations. Its declaration and storage width are not established by the supplied representation.

The anchor clear targets local storage, not the original placement-state input. This does not prove it is unobservable: the local record’s address may previously have been exposed during recursion.

No return value from final processing is inspected. If it returns normally, the current object address is returned. Without a current object, both final processing and this global read are skipped, and the invocation returns zero.

## Side effects

Direct nonlocal writes are limited to object fields and, during ordinary initialization, descriptor offset `0x24`. Local writes initialize placement state, replace continuation with recursive results, and clear the anchor before final processing.

There is no explicitly source-addressed store and no explicitly input-placement-addressed store. Neither source immutability nor placement-record immutability follows: another written region may overlap either input.

Descriptor replacement occurs before attachment. Attachment occurs before selected child traversal. Final processing follows traversal. No visible rollback reverses earlier writes if later dependencies fail to return normally.

No explicit release operation appears. A nonzero return does not establish ownership transfer, freshness, lifetime, or an obligation to release through any particular mechanism. Dependency retention of passed addresses is unknown; passing a local placement record does not establish that retaining it beyond this invocation is valid.

## Dependencies

| Opaque ID | Established interaction | Outside this contract’s established evidence |
|---|---|---|
| F-53a23d4b077c | Receives source offset `0x20` when context words differ; zero ends processing. | Resolution policy, allocation, ownership, and mutations. |
| F-3931e02c2857 | Receives destination offset `0x40` and the descriptor; a nonzero result is returned directly. | Matching policy, result lifetime, and mutations. |
| F-704d91f94e04 | Receives destination; a nonzero result is used as writable object storage. | Freshness, initialization, disjointness, and ownership. |
| F-12d234fad301 | Receives two context-relative addresses, a source word, and an object-field address. | Transfer semantics, resource handling, and other effects. |
| F-493743038bd6 | Receives object, original placement-state address, and `1`. | Attachment semantics, retention, and mutations. |
| F-ebd0ee53699f | Receives destination plus `0x48`, a late global value, and object. | Processing semantics, retention, ownership, and other effects. |
| F-a2dffecdec45 | Recursively processes children; each result replaces continuation. | Application-level meaning of the resulting structure. |

No dependency is assumed to preserve globals, flags, links, placement state, or other reachable storage.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Resolution failure:** For an initially unmarked source with unequal context words, a zero result from F-53a23d4b077c produces an immediate zero return. No reuse lookup or storage request follows.

2. **Successful reuse:** With a nonzero descriptor link, enabled reuse permission, and a nonzero F-3931e02c2857 result, that exact result is returned without initialization, attachment, traversal, or final processing.

3. **Short-circuited permission read:** A zero descriptor link makes the reuse decision independent of destination byte `0x18D`; that byte is not read for this decision.

4. **Stable ordinary initialization:** Given valid storage, consistently clear source-flag observations, no interfering aliasing, and normally returning dependencies, the object receives the ordinary field updates, attachment occurs, traversal is skipped, and final processing precedes return of the object.

5. **No parent object:** For an initially marked source with storage permission clear, children are traversed using initialized placement state. Even if children return nonzero objects, this invocation returns zero and performs no final-processing call.

6. **Storage failure:** A zero F-704d91f94e04 result ends processing immediately. The routine does not continue with children as a fallback.

7. **Null child result:** If a child returns zero but its subsequently read next-child field is nonzero, continuation becomes zero and the next child is still processed.

8. **Attachment changes traversal:** If attachment leaves the source flag clear, traversal is skipped. If it leaves the flag set, traversal is selected, independently of earlier observations.

9. **Late global observation:** If child processing changes the final-processing global and that value remains unchanged until the final read, F-ebd0ee53699f receives the changed value.

10. **Aliased placement storage:** If the caller’s placement-state address equals descriptor address plus `0x24`, ordinary descriptor replacement overwrites the placement anchor before attachment receives the original placement-state address.

11. **Mode transition without descriptor resolution:** An initially marked source that becomes clear before setup does not gain a resolved descriptor. Its ordinary-setup accesses require validity even at the zero-descriptor-derived address.

## Uncertainties

The following remain **documented_only**:

- Application-level types, flag meanings, and context-word semantics.
- Ownership, lifetime, retention, release, and supported alias configurations.
- Initial contents and provenance of acquired object storage.
- Dependency mutations, hidden failures, exceptions, and nonlocal exits.
- The final-processing global’s declaration, storage width, meaning, and synchronization.
- Supported concurrent mutation, reentrancy, and source-mode transitions.
- Traversal acyclicity, termination, and stack requirements.
- Behavior when an executed access is invalid or address arithmetic overflows.

Fresh object storage, stable source flags, and an acyclic child structure are plausible intended conditions, not established guarantees.

## Evidence

This contract derives from static inspection of the supplied function representation. No dependency bodies or global declaration were supplied. No runtime tests have been performed.

Established static evidence supports the listed access widths and offsets, conditional reads, repeated flag observations, ordered field updates, dependency arguments, local placement-state handling, recursive continuation updates, late next-child reads, final global observation, and return paths.

The behavioral labels are inferred intent only. The prior draft and reviewer feedback do not independently establish behavior or resolve the documented unknowns.
