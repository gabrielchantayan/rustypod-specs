<!-- Trusted provenance: F-e7b91dd3e7a5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7b91dd3e7a5

## Purpose

F-e7b91dd3e7a5 dispatches property requests against a caller-supplied object. Depending on the property identifier and representation selector, it publishes a stored 32-bit value, an object-relative address, a dependency-produced value, or a literal zero or five. One accepted property refreshes three words within the object before publishing their region’s address.

Two independent result channels are visible:

- A 32-bit function return value, either 0 or 1 on every established normal-return path.
- A caller-provided location receiving a 32-bit output on some paths.

A return value of 1 does not guarantee an output write. Several selector mismatches return 1 without directly modifying output. Conversely, publishing zero is a successful output-producing path returning 1.

Some paths pass an object-relative address, a resource address, and numeric arguments to a dependency before passing the same object-relative address to another dependency. Formatting is a plausible purpose, but neither text production nor a memory write by the first dependency is established here.

## Interface

### Arguments and widths

| Argument | Established contract |
|---|---|
| Object base | A 32-bit address-like value used for field accesses and address construction. Not dereferenced on every path. |
| Representation selector | A 32-bit value compared for equality against externally stored selector values. |
| Property identifier | A 32-bit dispatch value. Recognized identifiers are `0x6280` through `0x628b`, inclusive. |
| Output location | An address receiving one 32-bit direct store on output-producing paths. |

The function return is 32 bits. Object word accesses and published results are 32 bits; the gates at offsets `0x2c`, `0x2d`, and `0x68` read individual bytes.

One numeric dependency result is consumed as 64 bits and split into two 32-bit portions. The upper portion is passed onward as a 32-bit argument. Its high bit does not cause the argument to become wider.

All object offsets and dispatch-table offsets below are byte offsets. No portable address-overflow behavior, alignment rule, or fault behavior is established.

### Selectors and resources

The following labels describe roles, not recovered values:

| Label | Use |
|---|---|
| S-text | Selector consulted by properties `0x6280`–`0x6283`; also saved at entry for a fallback argument. |
| S-raw | Alternative selector for `0x6280`. |
| S-zero, S-five | Selectors tested in that order by `0x6285`. |
| S-action | Selector consulted by `0x6286`–`0x6289`. |
| S-refresh | Selector for `0x628a`. |
| S-records | Selector for `0x628b`. |

**The entry snapshot of S-text and its later comparison reads are distinct.** The function reads S-text before dispatch. Each reached property comparison in `0x6280`–`0x6283` then reads the global expression again. Only the fallback of `0x6282` uses the saved entry value as an argument. No global-stability assumption joins these reads.

Other referenced values are labeled R-fallback, R-mode-three, R-mode-other, R-flag-zero, R-flag-one, and R-enabled. Two additional resource addresses are labeled R-format-A and R-format-B. These names do not establish contents or semantics.

Selector values need not be distinct. Actual comparison order determines precedence.

### Object layout

| Offset | Established use |
|---|---|
| `0x20` | Read as a 32-bit value. |
| `0x24` | Address passed to F-1f62160dfd52, then F-44362605ab79. |
| `0x2c` | Byte gate for `0x6287` and `0x6288`. |
| `0x2d` | Byte gate for `0x6286`. |
| `0x30` | Address passed to a lookup dependency or published directly. |
| `0x5c` | Receives a 32-bit indirect-call result; its address is published by `0x628a`. |
| `0x60` | Receives a 32-bit zero during `0x628a`. |
| `0x64` | Receives a 32-bit indirect-call result during `0x628a`. |
| `0x68` | Byte gate for `0x6289`. |
| `0x8c` | Address passed to F-44362605ab79. |

This establishes neither the full object layout nor the extent of any region passed to a dependency. In particular, the address at `0x24` does not establish a buffer capacity, write bound, or separation from subsequent fields.

### Preconditions

Every reached direct read or write requires suitable accessible storage. Output-producing paths require a valid 32-bit output destination. Paths without a direct output write do not directly dereference that destination.

There are no visible null checks for the original object, output destination, or object supplied for the indirect calls in `0x628a`. The lookup result in `0x6282` is explicitly checked against zero.

Dependency-specific memory and calling requirements remain documented_only. No non-aliasing requirement is enforced by the visible body.

## Behavior

### Entry and publication

The S-text entry read precedes dispatch, including unsupported properties and `0x6284`. Output is not initialized at entry.

On an output-producing normal path, the function completes the relevant dependency calls and field updates, performs a 32-bit output store, and returns 1. A path described as leaving output unchanged means **no direct output store**; dependencies may still affect aliased memory.

### Property `0x6280`: numeric-processing or raw-value form

The selector is first compared against a fresh read of S-text, not the entry snapshot.

A match selects the numeric-processing form. The 32-bit field at object offset `0x20` is passed to F-d7be9cfb4000 with 1000. Let L and H denote the lower and upper 32-bit portions of its 64-bit result. A second call to that dependency receives H as a 32-bit argument and 100. The 32-bit portion consumed from the second result is used alongside L.

F-1f62160dfd52 then receives, in argument order:

- Object base plus `0x24`.
- R-format-A’s address.
- L.
- The consumed 32-bit result of the second numeric call.

Its return is not used. After it returns, F-44362605ab79 receives object base plus `0x24`. That dependency’s result is published.

The supplied body does not establish that F-1f62160dfd52 writes memory, produces text, or terminates a string.

If the S-text comparison fails, the selector is compared with S-raw. A match publishes the exact 32-bit field read at `0x20`, without these numeric-processing calls. A second mismatch returns 1 without a direct output write. Thus S-text has precedence if the consulted values coincide.

### Property `0x6281`: alternative resource form

Admission compares the selector against a fresh read of S-text. A mismatch returns 1 without a direct output write.

An accepted request uses the same numeric calls, argument widths, object-relative address, and subsequent F-44362605ab79 call as the numeric-processing form of `0x6280`. The difference is that F-1f62160dfd52 receives R-format-B’s address instead of R-format-A’s address.

The final dependency result is published. Any text or memory effects of F-1f62160dfd52 remain unknown.

### Property `0x6282`: lookup-derived value or fallback

Admission compares against a fresh read of S-text. A mismatch returns 1 without a direct output write.

For an accepted request, F-6372f72ded3a receives object base plus `0x30` and the 32-bit value read at `0x20`.

A zero lookup result selects fallback without reading lookup-relative fields or calling the predicate. For a nonzero lookup result, the function reads its 32-bit field at offset `0x0c` and passes that value to F-d8cbf9aa79a9.

If the predicate returns nonzero, F-44362605ab79 receives the original lookup-result address plus `0x08`; its result is published. The address is retained across the predicate call, but continued validity of the underlying memory is not independently guaranteed.

A zero predicate result selects the same fallback as a zero lookup result. Fallback calls F-9d0ecc5f6adf with an explicit zero argument, then calls F-5710a8628cba with:

1. That dependency’s result.
2. The saved entry snapshot of S-text.
3. R-fallback, read for this call after F-9d0ecc5f6adf returns.

The latter result is published. In particular, fallback does not replace the saved selector argument with the later comparison value.

### Property `0x6283`: embedded-address dependency

The selector is compared against a fresh read of S-text. A mismatch returns 1 without a direct output write.

A match passes object base plus `0x8c` to F-44362605ab79 and publishes its result.

### Property `0x6284`: unconditional rejection

The function returns 0 without a direct output write, property-specific object access, or property-specific dependency call. The entry S-text read still occurs.

### Property `0x6285`: literal selection

The selector is compared with S-zero first. A match publishes 32-bit zero and returns 1.

Only after that comparison fails is S-five consulted. A match publishes 32-bit five and returns 1. Otherwise, the function returns 1 without a direct output write.

If the two consulted selector values are equal, the zero branch wins.

### Property `0x6286`: external gate and byte-selected resource

A selector unequal to S-action returns 0 without a direct output write.

An accepted request calls F-93eb550dc612, then F-864a15812acb. Neither call shows explicit arguments. A zero result from the second call causes zero to be published, without reading the byte at `0x2d`.

For a nonzero result, the function reads that byte. Exactly 3 chooses the R-mode-three branch; every other byte chooses R-mode-other. This branch choice precedes a call to F-9d0ecc5f6adf with no explicit arguments shown.

The selected resource value is loaded **after** that call returns. F-ea9442e8ddbb receives the F-9d0ecc5f6adf result first and the selected resource value second. Its result is published.

The byte is therefore observed after the first two calls, while the selected resource’s contents are observed after the third. Later mutations do not cause the byte-based branch to be reevaluated.

### Properties `0x6287`, `0x6288`, and `0x6289`: byte-gated resources

Each compares the selector with S-action. A mismatch returns 1 without a direct output write.

| Property | Gate permitting the resource path | Resource |
|---|---|---|
| `0x6287` | Byte at `0x2c` equals 0 | R-flag-zero |
| `0x6288` | Byte at `0x2c` equals 1 | R-flag-one |
| `0x6289` | Byte at `0x68` is nonzero | R-enabled |

A failed gate publishes zero.

A successful gate calls F-9d0ecc5f6adf with no explicit arguments shown, then reads the corresponding resource value. F-ea9442e8ddbb receives that call result followed by the resource value, and its result is published.

These paths return 1. Values other than 0 and 1 fail both respective `0x2c` gates. Every nonzero byte satisfies the `0x68` gate, regardless of signed-byte interpretation.

### Property `0x628a`: refresh and publish an embedded region

A selector unequal to S-refresh returns 1 without a direct output write.

An accepted request obtains an object from F-17fab3ca59b5. Before dereferencing that returned object, it writes 32-bit zero to the original object at `0x60`.

The returned object’s first 32-bit word supplies a dispatch-table address. The callable at table byte offset `0x64` is invoked with that object as its argument. Its 32-bit result is stored at original-object offset `0x64`.

The dispatch-table pointer is then read again from the returned object. The callable at table byte offset `0x5c` is invoked with the same object argument. Its 32-bit result is stored at original-object offset `0x5c`.

Finally, the function publishes original-object base plus `0x5c` and returns 1.

The three explicit object stores occur in this order: `0x60`, `0x64`, `0x5c`. The published address identifies their contiguous three-word region, but no aggregate copy occurs. The first indirect call may change the table used by the second; reusing a cached table would not preserve the established accesses.

### Property `0x628b` and unsupported identifiers

A `0x628b` request compares the selector with S-records. A match publishes object base plus `0x30`; a mismatch returns 1 without a direct output write.

Every unsupported property returns 0 without a direct output write.

## Side effects

The only explicit object-field stores in the supplied body are the three 32-bit stores in accepted `0x628a` requests.

F-1f62160dfd52 is given object base plus `0x24`, but a write through that address is a **possible dependency-mediated effect**, not an established direct effect. The same distinction applies to other calls: their bodies and memory contracts are unavailable.

Output publication follows the path’s calls and direct field stores. If output aliases an object field, global, or dependency-visible storage, publication may overwrite earlier state. In particular, publishing the address from `0x628a` may overwrite one of the refreshed words.

No allocation, release, retain, or lifetime extension is directly visible. Dependency-produced outputs have unknown ownership. Directly constructed object-relative addresses identify storage within the supplied object; publication does not establish a longer lifetime.

There is no visible rollback or explicit recovery for access faults or abnormal dependency termination. Earlier stores and dependency effects may already have occurred.

## Dependencies

| Opaque function ID | Established use |
|---|---|
| F-d7be9cfb4000 | Numeric calls with 1000 and 100; first result consumed as 64 bits, second as 32 bits. Arithmetic meaning unknown. |
| F-1f62160dfd52 | Receives an object-relative address, one of two resource addresses, and two 32-bit numeric arguments. Return ignored; memory effects unknown. |
| F-44362605ab79 | Produces a published result from an object-relative or lookup-relative address. |
| F-6372f72ded3a | Receives an embedded address and a stored 32-bit value; result checked against zero. |
| F-d8cbf9aa79a9 | Supplies a zero/nonzero decision from a lookup-relative 32-bit field. |
| F-9d0ecc5f6adf | Supplies an argument for subsequent calls. Explicit zero is shown only on the `0x6282` fallback. |
| F-5710a8628cba | Produces fallback output from three arguments. |
| F-93eb550dc612 | First call on accepted `0x6286` requests. |
| F-864a15812acb | Supplies the external zero/nonzero gate for `0x6286`. |
| F-ea9442e8ddbb | Receives a dependency result and a subsequently loaded resource value. |
| F-17fab3ca59b5 | Supplies the object used for two indirect calls. |

The indirect targets have no supplied opaque identities. Their table offsets, invocation order, object argument, and consumed result widths are established; their wider contracts are documented_only. Calls without explicit arguments do not establish a complete zero-argument signature.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

1. **Separate selector observations.** If entry observes S-text as A and the later `0x6283` comparison observes B, a selector equal to B admits the request even when A differs from B.

2. **Saved fallback argument.** Under the same differing observations, an admitted `0x6282` request reaching fallback passes A—not B—to F-5710a8628cba.

3. **Mismatch status distinction.** A selector mismatch for `0x6286` returns 0; a mismatch for `0x6287` returns 1. Neither path directly writes output.

4. **Raw precedence.** If a `0x6280` selector fails the fresh S-text comparison but matches S-raw, output receives the exact word at `0x20`. If it matches S-text instead, the numeric-processing branch wins.

5. **Numeric portions without arithmetic assumptions.** If the first numeric result has portions L and H, the second call receives H and 100. F-1f62160dfd52 receives L and the consumed 32-bit second result. Acceptance does not require quotient/remainder semantics or formatted text.

6. **Lookup routes.** A zero lookup result skips the predicate. A nonzero lookup result with nonzero predicate reaches F-44362605ab79 at lookup address plus `0x08`, without fallback calls.

7. **Exact gates.** With accepted selectors and byte `0x2c` equal to 2, both `0x6287` and `0x6288` publish zero. Byte `0x68` equal to 2 permits the `0x6289` resource path.

8. **Refresh before publication.** If the indirect calls return A then B, and neither otherwise changes the relevant original-object fields, immediately before publication the words at `0x5c`, `0x60`, and `0x64` are B, zero, and A. They remain so after publication only if the output store does not overlap them.

9. **No-op dependency possibility.** If F-1f62160dfd52 leaves memory unchanged, the established caller behavior still proceeds to F-44362605ab79 with object base plus `0x24`. The caller contract alone does not require text to have been produced.

10. **Early rejection still reads entry state.** `0x6284` and unsupported properties return 0 without directly writing output, after the entry S-text read.

## Uncertainties

The following remain documented_only:

- Selector values, equality relationships, resource contents, and global synchronization.
- Dependency signatures beyond visible arguments and consumed results.
- Whether numeric processing represents division or another operation.
- Whether F-1f62160dfd52 formats, writes, enforces bounds, or terminates text.
- Ownership, allocation, release, and lifetime behavior of dependency results.
- Mutations of globals, objects, dispatch tables, or aliased output by callees.
- Memory validity across calls, alignment, address overflow, faults, and abnormal returns.

“Text,” “format,” “lookup,” “resource,” and “refresh” are descriptive role labels or inferred intent, not additional dependency guarantees.

## Evidence

The supplied static body establishes dispatch cases, separate entry and comparison reads of S-text, comparison precedence, access widths and offsets, visible arguments, call order, explicit stores, and normal return values.

The supplied provenance envelope associates the opaque function identifier with a pinned snapshot identifier. That provenance is distinct from behavioral evidence and does not supply dependency contracts. The snapshot identifier is not reproduced here.

No runtime tests have been performed. No dependency implementation or external dependency contract was supplied.
