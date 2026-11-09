<!-- Trusted provenance: F-f213fdc0eb88; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f213fdc0eb88

## Purpose

F-f213fdc0eb88 conditionally processes entries belonging to an input object. A processed entry receives a four-word region cache, compatibility handling for a retained subordinate object, configuration processing, and a final update.

Selection depends on a region span, an entry-specific requested extent, and the object’s current entry count. With a stable count greater than one, every traversed entry is selected. With a stable count of one, the entry is selected only when the span is strictly smaller than its requested extent.

“Region,” “extent,” “subordinate object,” and “configuration” describe data-flow roles. Their application-level meanings and units are not established. Allocation, initialization, binding, and replacement are inferred dependency roles, not verified lifecycle contracts.

This specification describes static evidence only. No runtime tests have been performed.

## Interface

### Call boundary

The function accepts one object address carried in a 32-bit quantity. It exposes no explicit return value or status result.

Normal return occurs when a signed 32-bit entry-count read is no greater than the current traversal index. The function provides no explicit validation, allocation-failure handling, rollback, or exception recovery.

Valid execution requires readable and writable storage for the accesses actually reached, usable address-bearing dependency results, and valid indirect-call targets. Dependency success, memory preservation, and ownership behavior are not established.

### Object layout

Offsets below are byte offsets from the input object.

| Offset | Width or extent | Role |
|---|---|---|
| `0xA4` | Extent unknown | Embedded address supplied during preparation. |
| `0xE9` | 8 bits | Optional-processing input, explicitly zero-extended there; separately read for the final update. |
| `0xEA` | 8 bits | Interpreted as a signed byte and widened for the final update. |
| `0xEB` | 8 bits | Interpreted as a signed byte and widened for the final update. |
| `0xF0` | 32 bits | Flags; bit mask `0x800` selects the configuration source. |
| `0xF4`–`0xF7` | Four bytes | Inline configuration source. |
| `0xFC` | 32 bits | Associated address. |
| `0x244` | Signed 32 bits | Entry count, consulted repeatedly. |

For entry index \(k\), each location in the following table is displaced by an additional \(108k\) bytes from the input object:

| Entry-zero offset | Width or extent | Role |
|---|---|---|
| `0x248` | Extent unknown | Address supplied to the final-update argument producer. |
| `0x250` | 32 bits | Retained subordinate-object address. |
| `0x254` | Extent unknown | Embedded entry state passed to dependencies. |
| `0x298` | 32 bits | First cached region word. |
| `0x29C` | Signed 32 bits | Cached lower endpoint. |
| `0x2A0` | 32 bits | Third cached region word. |
| `0x2A4` | Signed 32 bits | Cached upper endpoint. |
| `0x2A8` | Signed 32 bits | Requested extent. |

These locations establish neither complete object sizes nor alignment requirements.

### Temporary data and external input

The acquired region occupies four consecutive 32-bit words. The second and fourth words are treated as signed lower and upper endpoints. The first and third words are copied without arithmetic.

Separate working-region storage also contains four consecutive 32-bit words. Additional temporary storage includes a zero-extended byte in a 32-bit slot, a two-word output area, and a four-byte configuration copy.

An external datum supplies the second argument to F-d06d54bfd78a. Its storage width, type, value domain, and semantic role are not established. It is not established to be constant.

## Behavior

### Preparation and traversal boundary

F-88dd67d704e5 receives the input object and its embedded address at `0xA4`. F-412e4635fdb0 subsequently receives the object and storage for the acquired region. Both calls precede the first entry-count check, including when no entries will be processed.

Traversal begins at index zero. Each boundary check rereads the signed count at `0x244`; processing continues only while the index is smaller than that read. The index advances by one after each traversed entry, whether selected or skipped.

The acquired region is explicitly obtained only once per invocation. There is no reacquisition between entries. This does not guarantee that callees or aliases preserve its storage.

### Entry selection

An entry qualifies when the acquired upper endpoint minus the acquired lower endpoint is strictly less than the entry’s requested extent.

If that comparison is false, selection additionally depends on a fresh signed count read being greater than one. That additional count read is omitted when the span comparison succeeds.

Consequently, selection does not use a single invocation-wide count snapshot. A boundary check may admit an entry under one count value, while the selection decision observes another.

For a skipped entry, this function performs no explicit entry writes or entry-specific dependency calls. Preparation and selection reads have already occurred.

### Region contract and cache visibility

Every selected entry starts with a working copy of the four acquired words. A separate span-versus-request comparison determines whether its upper endpoint needs adjustment. If the span is smaller, the working upper endpoint becomes the acquired lower endpoint plus a further requested-extent read. Otherwise, it remains the acquired upper endpoint.

Requested extent is therefore not captured once for the whole entry. Selection, adjustment, and subsequent compatibility checking use distinct reads.

The direct cache writes occur in increasing cache-address order: first word, lower endpoint, third word, then working upper endpoint. They all precede entry-specific dependency calls.

The working region, rather than the cache, is supplied to later dependencies. Those dependencies may modify working storage; no subsequent direct cache refresh is visible. Cache contents and the region eventually presented to the final update are therefore not guaranteed to remain equal.

### Associated-state inputs

When a read of the associated address at `0xFC` is nonzero, F-6489b8f5efca receives the object, an associated-address argument, embedded entry state, and writable temporary storage containing a zero-extended read of byte `0xE9`.

The guard and argument use separate associated-address reads. There is no established atomic snapshot spanning them. Changes to the widened-byte temporary have no subsequent explicit use by this caller.

A zero associated-address guard skips only this optional call. It does not establish that later configuration access is safe.

Independently of that optional call, F-78beca4e4455 receives the embedded entry-state address and output storage comprising two 32-bit words. Only the first output word is consumed here. Its lowest bit is cleared, and the resulting address supplies a byte at offset `0x4D`.

The displayed interpretation of that byte is signed and widened: `0x80` corresponds to −128 and `0xFF` to −1. This value governs subordinate compatibility and is also supplied on creation. No output-initialization, null, bounds, or tag-validity check precedes the byte access.

### Subordinate-object compatibility

An absent retained address requires creation. A present address is reusable only if both of these conditions hold:

- F-19fde5f06ccc, called with the retained address plus `0x98`, returns a signed 32-bit value equal to a requested-extent read made after that call.
- The signed 32-bit field at offset `0x0C` of the subsequently read retained object equals the signed associated byte.

The second compatibility read is conditional: it is omitted when the first comparison fails.

The retained address is not frozen across F-19fde5f06ccc. If that dependency changes the entry’s address field, subsequent field access uses the newly read value. The function supplies no intervening validity check.

For an incompatible object, the retained address is checked again. If nonzero, an indirect operation is invoked through a dispatch table reached from that object, using the function slot at table byte offset `0x04`. The complete argument contract and whether this operation releases the object remain unknown.

Creation requests 184 from F-8157a180d901. Its result becomes an input to F-cec144b59c0b together with the working-region address, the signed associated byte, and two zero-valued arguments. The result of F-cec144b59c0b is stored as the retained address and passed with embedded entry state to F-22ebf78204e2.

There is no visible null-result check. The stored initialization result need not equal the allocation-like result. An old retained address is not explicitly cleared before the replacement operation or allocation-like call.

### Configuration and final update

After reuse or creation handling, each selected entry reaches F-d06d54bfd78a with embedded entry state and a read of the external datum. F-a7cb9e140c0e follows, receiving embedded entry state and the working-region address.

The external datum is read at this stage for each selected entry, not captured during preparation. Earlier dependencies in the same entry, or dependencies processing earlier entries, are not assumed to preserve its value.

Configuration-source selection occurs after F-a7cb9e140c0e:

- With mask `0x800` set in the flags read at `0xF0`, the source is the object’s inline four bytes beginning at `0xF4`.
- With that mask clear, the source is eight bytes beyond an address read from `0xFC`.

Four individual bytes are copied from the selected source in increasing address order. The copy is not established to be atomic.

F-44362605ab79 then receives the entry address identified by entry-zero offset `0x248`. Its result becomes one argument to F-452653fc0ce2.

The final update also receives the input object, embedded entry state, working-region address, a fresh byte from `0xE9`, fresh signed-and-widened bytes from `0xEA` and `0xEB`, and the configuration-copy address.

The configuration copy precedes F-44362605ab79; the three object-byte reads follow it. Their relative argument-read order is not established. Unlike the optional-processing input, the final `0xE9` input is shown as a byte-valued argument without an explicit 32-bit zero-extension temporary.

## Side effects

### Direct writes and partial progress

For each selected entry, direct persistent writes comprise the four cache words and, when creation occurs, the retained subordinate address. There are no visible direct writes to the count, requested extent, flags, associated address, inline configuration, or external datum.

This does not imply preservation by dependencies. Several callees receive addresses into the object, entry state, or temporary storage; indirect mutation and reentrancy are not excluded.

Processing is incremental rather than transactional. Cache writes are already visible before compatibility checks, allocation-like processing, configuration access, and final update. No rollback is visible if a later operation fails or does not return.

### Ownership and lifetime

The request of 184 and subsequent initialization-like call do not establish allocation units, ownership transfer, or release obligations. The indirect operation is not proven to destroy or free the prior object. F-22ebf78204e2 is not proven to take ownership.

No explicit cleanup of the two-word associated-state output appears. Whether cleanup is required remains unknown.

Temporary addresses supplied to dependencies have no established lifetime beyond this invocation. The caller does not establish that dependencies copy rather than retain their contents.

### Access and arithmetic requirements

Safe execution requires valid storage at every access actually performed. In particular:

- Clearing the associated output’s lowest bit must yield an address permitting the byte read at `0x4D`.
- Retained-object storage and its dispatch table must remain usable at the time of each access.
- With mask `0x800` clear, the associated address plus eight must permit four byte reads. A previously observed zero associated address does not select an alternative source.
- Count growth must not expose entries beyond available storage.

Endpoints, requested extents, count comparisons, and traversal index are represented as signed 32-bit operations. Address-bearing quantities are 32-bit. Span subtraction, endpoint addition, index increment, stride multiplication, and address formation can exceed their representable domains.

Portable overflow behavior is not established. The examples below require representable arithmetic and valid resulting addresses; they do not impose wrapping, saturation, or rejection.

Negative requested extents and negative spans are not explicitly rejected. They participate in the displayed signed comparisons without an additional nonnegative-size rule.

## Dependencies

All identities below are opaque. Roles describe only this call site.

| Dependency | Established interaction |
|---|---|
| F-88dd67d704e5 | Receives the object and its embedded `0xA4` address before traversal. |
| F-412e4635fdb0 | Receives storage for the acquired four-word region. |
| F-6489b8f5efca | Optional processing with associated state and a writable zero-extended-byte temporary. |
| F-78beca4e4455 | Receives two-word output storage; its first word supplies the masked address-bearing value. |
| F-19fde5f06ccc | Returns a signed value compared with requested extent. |
| F-8157a180d901 | Receives the allocation-like request of 184. |
| F-cec144b59c0b | Receives the preceding result, working region, signed byte, and two zero arguments; returns the value retained by the entry. |
| F-22ebf78204e2 | Receives embedded entry state and the newly retained value. |
| F-d06d54bfd78a | Receives embedded entry state and the external datum read for this selected entry. |
| F-a7cb9e140c0e | Receives embedded entry state and working region before configuration capture. |
| F-44362605ab79 | Produces an argument for the final update after configuration capture. |
| F-452653fc0ce2 | Performs the final selected-entry call using state, region, object bytes, and copied configuration. |

An additional indirect dependency has unresolved identity and calling contract. No dependency is assumed to preserve globals, retain no pointers, own memory, release memory, or return successfully.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, assume valid storage, representable arithmetic, initialized outputs, returning dependencies, and no intervening mutation affecting the example.

### Empty or negative count

If the count is zero or negative at the first boundary check, both preparation calls have already occurred. No entry-specific calls, cache writes, or per-entry external-datum reads occur.

### Strict single-entry selection

With a stable count of one, lower endpoint 10, upper endpoint 30, and requested extent 20, the entry is skipped. A requested extent of 19 also skips it.

For requested extent 25, the entry is selected. Its cached endpoints become 10 and 35; the first and third acquired words are copied unchanged.

### Multiple entries without expansion

With a stable count of two, endpoints 10 and 30, and requested extent 15 for each entry, both entries are selected. Each retains an upper endpoint of 30 because selection by count does not itself require expansion.

### Signed, rather than nonnegative, extent handling

With count one, lower endpoint 10, upper endpoint 0, and requested extent −5, the signed span is −10. The entry is selected because −10 is smaller than −5, and its working upper endpoint becomes 5.

No validation failure is implied by either the negative span or negative request.

### Reuse and incompatible replacement

For a selected entry requesting 25, reuse occurs if F-19fde5f06ccc returns 25 and the subsequently accessed retained object has a `0x0C` field equal to the acquired signed byte. No replacement, allocation-like, initialization-like, or binding call is then made.

If the query instead returns 24, the `0x0C` compatibility read is omitted. If the retained address remains nonzero, the indirect operation precedes creation.

An associated byte of `0xFF` compares equal to a signed field value of −1, not 255.

### Mutable retained address

Suppose the extent query uses retained object A but changes the retained address to B before returning a matching extent. The subsequent compatibility field is read from B, not necessarily A. This expectation requires B to be valid for that access.

### Per-entry external value

If the external datum is A at the first selected entry’s read and B at the second’s, F-d06d54bfd78a receives A and B respectively. A creation or binding dependency can likewise affect the current entry’s value if it changes the datum before that entry’s read.

This example does not assign a width or type to the datum.

### Configuration capture and later mutation

With mask `0x800` set, configuration comes from inline bytes even when the associated address is zero. With the mask clear, zero does not trigger fallback; valid storage at the resulting address remains a precondition.

If F-44362605ab79 changes the original configuration source, the final update still receives the earlier copy. Its changes to `0xE9`, `0xEA`, or `0xEB` can affect the subsequent final-call inputs.

### Count changes and cached-region divergence

If the count becomes one while index zero is processed, the next boundary check terminates traversal. Increasing the count can admit more entries, subject to storage and arithmetic validity.

If a later dependency modifies the working region after cache writes, the final update can receive modified working values while the cache retains its earlier values. No automatic reconciliation is established.

## Uncertainties

The following remain **documented_only**:

- Application semantics, measurement units, coordinate conventions, and complete structure layouts.
- External-datum width, type, mutability, value domain, and purpose.
- Callee effects on globals, acquired storage, object fields, temporary storage, or other entries.
- Whether the displayed signed-byte interpretation matches original target declarations.
- The significance of the cleared address bit and any associated-output cleanup obligation.
- Indirect-call arguments, lifecycle effects, and ownership.
- Allocation success, allocation units, initialization failure behavior, and ownership transfer.
- Integer-overflow behavior and out-of-range address formation.
- Behavior after invalid access, abnormal dependency exit, or retained temporary pointers.
- Synchronization, atomicity, and behavior under concurrent mutation.

No preservation guarantee, fallback, error policy, or ownership rule is inferred to close these gaps.

## Evidence

**Established static evidence:** the supplied representation supports the listed widths and offsets, 108-byte entry stride, signed comparisons, conditional reads, repeated count and requested-extent reads, cache-write order, dependency phase order, configuration-byte capture, and absence of explicit validation or rollback. It places each external-datum read after subordinate processing and before F-a7cb9e140c0e, without establishing that datum’s width or invariance.

**Inferred intent:** region expansion, subordinate compatibility, allocation, initialization, binding, and lifecycle replacement are interpretations of observed relationships. They are not independent dependency guarantees.

**Runtime evidence:** none. No execution, tests, memory traces, or dependency inspection has been performed.
