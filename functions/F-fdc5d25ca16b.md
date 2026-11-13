<!-- Trusted provenance: F-fdc5d25ca16b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fdc5d25ca16b

## Purpose

F-fdc5d25ca16b scans a caller-supplied selection, requests recognition and descriptor retrieval for selected indices, aggregates successful descriptors, and conditionally requests result finalization and publication.

An optional auxiliary path constructs two objects and two associated wrappers, processes eligible descriptors, and requests termination handling. Under the supplied annotation that F-feee9156a257 does not return, successful auxiliary setup prevents a completed scan from reaching ordinary result finalization.

These are static behavioral findings. “Allocation,” “construction,” “selection,” “publication,” and “cleanup” describe inferred roles, not dependency guarantees. No runtime tests have been performed.

## Interface

The function accepts four inputs and returns a 32-bit integer.

| Input | Representation | Established use |
|---|---|---|
| Context | 32-bit address | Supplies backing-object access, state, an existing-result slot, and embedded dependency arguments. |
| Selection | 32-bit address | Its first two unsigned 32-bit words supply scan limits; dependencies receive its address. |
| Mode | 32-bit integer | Controls context handling and publication requests; its low eight bits are written into a requested result record. |
| Auxiliary flag | 32-bit integer | Zero bypasses auxiliary setup; any nonzero value attempts setup. |

The supplied representation uses 32-bit addresses and words. Endianness is not established.

### Accessible layouts

All offsets below are byte offsets.

| Context offset | Access | Meaning established by use |
|---|---|---|
| 0 | Read 32-bit address | Backing-object pointer, sampled repeatedly. |
| 4 | Pass embedded address | Argument to F-dfabedd4a3d1. |
| 8 | Read unsigned 32-bit word | Compared with 10 in modes 0 and 3. |
| 20 | Read/write 32-bit word | Existing-result argument and later result-handle slot. |
| 24 | Pass embedded address | Argument to F-4a19e3742733 and F-fd58bea73663. |

The backing object supplies a dispatch-table address in its first word, a signed byte at offset 77, and signed 16-bit values at offsets 80, 82, and 84.

The descriptor workspace occupies 20 bytes. Its first two 32-bit words contribute to extent arithmetic; its third word is compared as signed 32-bit data. All 20 bytes are exposed to an external operation during auxiliary processing.

### Preconditions and return values

No entry validation protects the context or selection. Requested storage is sometimes dereferenced without checking the request result. Defined operation therefore requires valid storage for every reached access and dependency-specific preconditions that are unavailable here.

Assuming execution reaches an ordinary return:

- A zero current masked successful-entry count returns the retrieval-status value: initially 1, permanently changed to 0 by any failed retrieval.
- A nonzero count followed by a zero result-record request returns 0.
- With a nonzero result-record request, the finalizer’s nonzero result is returned unchanged; a zero finalizer result returns 0.

A zero masked count does not prove that no retrieval succeeded: the count also becomes zero after 65,536 successes.

## Behavior

### Initial tracking object and mode handling

The first request is F-8157a180d901 with size 16. The function then samples a global 32-bit value and calls F-638beded7699 with size 8,192 and tag 50.

Using the first request result as an object address, it writes the tagged-request result at offset 8, the sampled global value at offset 0, and byte value 50 at offset 12, in that order. An unidentified external operation then receives the stored offset-8 handle and 8,192. After that call, the function writes zero at offset 4.

Neither request result is checked before use. The external operation is not established to clear storage or preserve previous writes. Bytes 13–15 receive no explicit write from this function. The resulting object is subsequently used as the failure-tracking handle.

F-acff1e21fa9a receives the descriptor-workspace address. Initialization is an inferred purpose, not an established postcondition.

F-4a19e3742733 next receives context offset 24. Mode handling follows:

| Mode or condition | Subsequent requests and direct changes |
|---|---|
| 0 or 3, context offset-8 word below 10 unsigned | Call F-fd58bea73663 with context offset 24. |
| 0 or 3, that word at least 10 | Call F-702e45e090f5 with context; set a local retained-state marker; omit the immediate F-fd58bea73663 call. |
| 1 | Call F-fd58bea73663 with context offset 24. |
| 2 | Call F-c2e1f1434f05 with context and its current offset-20 value; directly clear offset 20 afterward; call F-fd58bea73663. |
| Any other value | Make no immediate F-fd58bea73663 call; leave the retained-state marker clear. |

Unsupported modes are not rejected. The paired dependency calls suggest synchronization, but no exclusion or balancing guarantee is established.

### Saved samples and auxiliary setup

After mode handling, the function samples context offset 0. From that backing object it reads signed shorts at offsets 82 and 84 and sums their promoted values to initialize the aggregate dimension. It also retains the signed byte at offset 77.

The initial dimension ranges from −65,536 through 65,534. Its value plus one fits signed 32-bit arithmetic.

The selection’s second word is saved before auxiliary setup. Its first word is saved only after setup or completed fallback. Thus the two limits have distinct sampling times.

With a nonzero auxiliary flag, setup requests occur in this order:

1. F-d7be9cfb4000 receives 65,536 and the saved initial dimension plus one. Its return becomes the auxiliary comparison bound.
2. F-8157a180d901 receives 184. F-cec144b59c0b receives that request result, a four-word structure containing zero, zero, saved initial dimension plus one, and the bound, followed by the saved signed byte and two zero arguments. Its return is the first auxiliary object.
3. F-638beded7699 receives 20,480 and tag 45, producing the descriptor-storage handle.
4. Context offset 0 is reread; a signed short at backing-object offset 80 is sampled through this new pointer.
5. Another 184-byte request is passed to F-cec144b59c0b. Its structure contains zero, zero, the saved initial dimension plus one, and the newly sampled short. Its remaining arguments use the earlier byte sample and two zeros. Its return is the second auxiliary object.
6. A 68-byte request is passed to F-1ade66241a82 with the second object, producing the second-object wrapper.
7. Another 68-byte request is passed to F-1ade66241a82 with the first object, producing the first-object wrapper.

The later offset-80 sample may therefore come from a different backing object. Earlier dependencies intervene and are not assumed to preserve context or backing data.

Readiness requires all of the following: nonzero first object, nonzero word at its offset 4, nonzero descriptor-storage handle, nonzero second object, and nonzero results for both wrappers. There is no independent readiness test of the second object’s offset-4 word.

### Failed-readiness handling

If readiness fails, the established order is:

1. If the first object is nonzero, invoke its dispatch-table entry at offset 4 with that object.
2. Call F-8349c252c168 with the descriptor-storage handle and tag 45, **including when the handle is zero**.
3. If the first-object wrapper is nonzero, call F-282eebd7c413 with it, immediately followed by F-feee9156a257.
4. Only if execution reaches this point, dispatch offset 4 on the second object when that object is nonzero.
5. If the second-object wrapper is nonzero, call F-282eebd7c413 with it, immediately followed by F-feee9156a257.

F-feee9156a257 is annotated nonreturning. Under that annotation, reaching either wrapper sequence prevents all later handling on that path.

Only if fallback reaches its end are both local object handles, both wrapper handles, and the descriptor-storage handle set to zero before scanning. These calls do not independently establish destruction or release.

### Scan and retrieval

The scan starts at index 1. Before each query it stops if the unsigned index is at least the saved first selection word, or the selected-encounter count is at least the saved second word.

Index advancement is a 32-bit increment followed by clearing bit 16. Starting from 1, the sequence is 1 through 65,535, then 0, then 1 again. The selected-encounter count instead increments as an ordinary unsigned 32-bit value.

Each candidate is queried through F-88658862e573 with selection, index divided by 32, and index modulo 32. Zero skips that candidate. Nonzero increments the encounter count before recognition or retrieval.

F-45681f0808e6 then receives context and index:

- A nonzero return bypasses retrieval and requests F-008d77a63c85 with selection, index, and 0.
- A zero return leads to backing-object dispatch offset 12, using the current backing object, index, and workspace address.

A zero retrieval return permanently changes retrieval status to 0. F-008d77a63c85 is then requested first with the failure-tracking handle, index, and 1, and then with selection, index, and 0.

These are update requests, not proof of actual bit changes. Successful retrieval has no corresponding direct selection-update request.

The saved limits do not guarantee termination for every dependency response pattern. In particular, the index cycles rather than exceeding 65,535.

### Auxiliary processing after successful retrieval

When the first auxiliary object is nonzero, the function samples a 32-bit descriptor extent as the workspace’s second word minus its first word.

While auxiliary eligibility remains enabled, it forms a prospective endpoint by adding that extent to the packed cursor. Eligibility becomes permanently disabled if:

- The endpoint is greater than or equal to the setup bound under signed 32-bit ordering; or
- The current masked successful-entry count, before increment, exceeds 1,023.

Equality rejects processing. A preceding count of 1,023 still permits an attempt. The count is successful retrievals, not successful auxiliary operations.

For an eligible attempt, F-970cd717d3d2 receives the second object. The backing pointer is then reread for dispatch offset 60. That dispatch receives the backing object, workspace address, index, and the second object’s words at offsets 4 and 8.

A zero return skips auxiliary processing for this descriptor without disabling future attempts. On a nonzero return, two four-word structures are populated. In increasing word order, **each contains zero, zero, the saved initial dimension plus one, and the previously sampled descriptor extent**.

F-d156b0b83eb3 receives the second structure’s address, the current packed cursor, and zero. Next, F-b774eb6cb163 receives, in order:

1. First-object wrapper.
2. Second-object wrapper.
3. First structure’s address.
4. Second structure’s address.
5. 255.
6. Zero.

The second structure’s contents at this later call are not guaranteed to remain their initialized values: F-d156b0b83eb3 intervenes with its address. Neither dependency is assumed to preserve other reachable memory.

After these calls, the workspace’s second word is directly written with the saved prospective endpoint. The destination displacement is formed as 20 times the previous packed-descriptor count, that count increments, and the workspace’s first word is written with the prior packed cursor. An unidentified external operation receives descriptor-storage handle plus that displacement, workspace address, and 20. The packed cursor then becomes the saved endpoint.

The external operation is not established to copy 20 bytes or preserve the workspace. The sampled extent and endpoint need not describe the workspace after intervening calls.

### Aggregation

Every nonzero retrieval return subsequently:

- Increments the successful-entry count and clears bit 16, wrapping 65,535 to zero.
- Assigns the current index to the first-index field if that field is zero.
- Adds the workspace’s then-current second-word-minus-first-word extent to the 32-bit aggregate extent.
- Assigns the current index to the last-index field.
- Replaces the aggregate dimension if the workspace’s third word is greater under signed comparison.

Zero is the first-index sentinel. A successful index zero leaves that field eligible for replacement.

There are no explicit overflow checks for extent subtraction, endpoint addition, accumulated extent, or destination arithmetic. These are represented 32-bit operations; valid same-object addressing and adequate storage capacity are not established.

### Scan termination and result handling

At scan termination, a **nonzero first-object guard encloses all three** of these operations, in order:

1. If the first-object wrapper is nonzero, call F-282eebd7c413 with it, then annotated-nonreturning F-feee9156a257.
2. If reached, dispatch offset 4 on the second object when that object is nonzero.
3. If reached and the second-object wrapper is nonzero, call F-282eebd7c413 with it, then F-feee9156a257.

This block does not itself dispatch offset 4 on the first object. Successful readiness requires a nonzero first-object wrapper, so under the nonreturning annotation a completed scan with successful auxiliary setup cannot proceed beyond its first wrapper sequence.

If execution reaches the count branch with a zero masked successful count, it conditionally dispatches offset 4 on the first object, conditionally requests F-8349c252c168 for a nonzero descriptor-storage handle with tag 45, calls F-663fcf71923c with the failure-tracking handle’s address, and returns retrieval status. No result-record request or F-5240c36738e7 request occurs.

For a nonzero count, F-638beded7699 receives 20 and tag 45. A nonzero result receives a direct byte write at offset 12 containing the mode’s low eight bits. No complete record initialization is established.

After staging additional temporary values, F-787c961b99fe receives context, selection, first index, last index, masked successful count, aggregate extent, and aggregate dimension. The staged values do not establish additional arguments.

Before testing that finalizer’s result:

- Mode 2 calls F-4a19e3742733 with context offset 24 and directly writes the record handle to context offset 20.
- Other modes call F-4a19e3742733 only when the retained-state marker is clear, then call F-dfabedd4a3d1 with context offset 4 and the record handle’s address.
- F-fd58bea73663 follows.
- If the first object is nonzero, its offset-4 dispatch follows, then F-8349c252c168 with descriptor-storage handle and tag 45.

A nonzero finalizer result is returned after F-663fcf71923c.

A zero record request or zero finalizer result instead leads to F-5240c36738e7 with selection and failure-tracking handle, conditional F-8349c252c168 on the current nonzero record handle with tag 45, F-663fcf71923c, and return 0. F-dfabedd4a3d1 may change the handle through its address. No explicit rollback reverses a preceding mode-2 context assignment.

## Side effects

Direct writes include the tracking-object fields, mode-2 context slot changes, result-record mode byte, auxiliary temporary structures, and descriptor-workspace endpoint fields.

Dependencies receive mutable addresses for context, selection, workspace, temporary structures, and handle slots. Their effects on globals, backing pointers, storage contents, ownership, or lifetime are not established.

No direct release of caller-owned context or selection is visible. Passing a handle to a publication-like or cleanup-like dependency does not establish ownership transfer or release. Invalid addresses and unusable unchecked request results have no explicit recovery path.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-8157a180d901 | Storage-like requests of 16, 184, and 68 bytes. |
| F-638beded7699 | Size/tag requests: 8,192/50, 20,480/45, and 20/45. |
| F-8349c252c168 | Handle and tag-45 request. |
| F-acff1e21fa9a | Receives workspace address. |
| F-4a19e3742733, F-fd58bea73663 | Receive context offset 24 in mode-dependent order. |
| F-702e45e090f5 | Receives context in retained-state branches. |
| F-c2e1f1434f05 | Receives context and existing offset-20 value. |
| F-d7be9cfb4000 | Supplies auxiliary comparison bound. |
| F-cec144b59c0b, F-1ade66241a82 | Produce auxiliary objects and wrappers. |
| F-282eebd7c413, F-feee9156a257 | Ordered wrapper termination sequence; the latter is annotated nonreturning. |
| F-88658862e573, F-45681f0808e6 | Control selection processing and retrieval bypass. |
| F-008d77a63c85 | Receives handle, index, and update value. |
| F-970cd717d3d2 | Receives second auxiliary object before dispatch offset 60. |
| F-d156b0b83eb3, F-b774eb6cb163 | Receive the auxiliary structures and ordered arguments specified above. |
| F-787c961b99fe | Finalizer-like request returning status. |
| F-dfabedd4a3d1 | Receives embedded context address and record-handle address. |
| F-5240c36738e7 | Failure-path reconciliation-like request. |
| F-663fcf71923c | Receives failure-tracking handle address. |

Indirect dispatch offsets are 4, 12, and 60. Opaque IDs were not supplied for the two external operations; their semantics remain unidentified.

## Acceptance examples

These are conditional static expectations, not executed tests. Accessed storage must be valid; dependencies return normally except where the nonreturning annotation applies.

1. **Empty scan:** Auxiliary processing disabled and saved first selection word equal to 1 cause no query. Cleanup-like F-663fcf71923c occurs and return is 1.

2. **Recognition consumes the encounter limit:** With limit 1, a nonzero query and nonzero recognition result cause selection update value 0, no retrieval, and no later query.

3. **Failed retrieval:** A selected, unrecognized entry whose retrieval returns zero causes failure-tracking update value 1 before selection update value 0. With no successes, return is 0.

4. **Counter wrap:** With auxiliary processing disabled, saved upper bound 65,536, encounter limit 65,536, every query and retrieval nonzero, and every recognition result zero, indices 1–65,535 followed by 0 produce a zero masked count. No result record is requested; return is 1.

5. **Auxiliary structure inputs:** If saved initial dimension is 4, sampled extent is 7, cursor is 10, and the endpoint passes the bound test, both structures initially contain zero, zero, 5, and 7. F-d156b0b83eb3 receives the second structure, 10, and zero. The subsequent call receives wrappers in first-object then second-object order; preservation of the second structure is not assumed.

6. **Failed readiness with zero storage:** If readiness fails and descriptor storage is zero, F-8349c252c168 still receives zero and tag 45. A nonzero first wrapper then causes its termination sequence; under the annotation, second-object handling is not reached.

7. **Successful auxiliary setup:** At scan termination, the first wrapper’s termination sequence occurs before second-object dispatch. Its annotated nonreturn prevents finalization and ordinary return.

8. **Finalizer failure in mode 2:** When finalization is reached with a nonzero record, the context slot is assigned before the zero finalizer result is acted upon. Reconciliation and conditional record-release requests follow; no explicit slot rollback occurs.

9. **Endpoint equality:** An endpoint equal to the signed bound disables auxiliary attempts. Successful retrieval still contributes the workspace’s then-current values to aggregation.

## Uncertainties

The following remain **documented_only**:

- Dependency contracts for allocation, initialization, mutation, synchronization, ownership, publication, and release.
- Both unidentified external operations’ effects and preservation guarantees.
- The finalizer’s complete interface and the meaning of additionally staged values.
- Descriptor and backing-field domain meanings.
- Independent validity of the nonreturning annotation.
- Behavior after invalid accesses or unsafe arithmetic-derived addresses.

No runtime tests have been performed.

## Evidence

Established findings come from the supplied static representation: access widths, offsets, constants, explicit writes, sampling boundaries, call arguments, guards, call order, arithmetic, comparisons, counter masks, and return paths.

Role descriptions are inferred intent. The prior draft and reviewer feedback are not independent evidence; the specified auxiliary inputs and cleanup guards are supported by the supplied source representation. No runtime behavior or unavailable dependency postcondition is asserted.
