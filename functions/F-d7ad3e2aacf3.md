<!-- Trusted provenance: F-d7ad3e2aacf3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d7ad3e2aacf3

## Purpose

F-d7ad3e2aacf3 routes a block-oriented request through a general request dependency, an optional region-associated transfer, or both. Its local address calculations scale block quantities by **512 bytes**.

The established outcomes are:

- **Complete-request delegation:** when the optional region is disabled, its initially loaded descriptor pointer is null, or F-83b1e9e0a36b returns zero.
- **Immediate rejection:** when the optional region is enabled, its initially loaded descriptor pointer is nonnull, the direction selector is nonzero, and the rejection byte is nonzero. The function returns **0x15 without calling any dependency**.
- **Partitioned handling:** after eligibility and direction checks pass, a nonzero result from F-83b1e9e0a36b selects an optional leading general-request operation, a region-associated transfer, and an optional trailing general-request operation, in that order.

The partitioning and routing are established by static evidence. Calling the optional region a cache, identifying a storage technology, or interpreting the direction selector as read versus write would require additional dependency contracts.

## Interface

The function accepts six arguments and returns an unsigned 32-bit result.

| Input | Representation in the supplied model | Established role |
|---|---|---|
| Object | Address carried in a 32-bit integer | Provides eligibility fields and a descriptor pointer; forwarded to the general request dependency. |
| Request start | Unsigned 32-bit integer | Starting block position; participates in unsigned comparisons and endpoint calculations. |
| Request count | Signed 32-bit integer | Forwarded unchanged for complete delegation and selection; participates in request-end calculation. |
| Caller buffer | Address carried in a 32-bit integer | Passed to dependencies and advanced by block quantities scaled by 512. |
| Direction selector | Signed 32-bit integer | Zero and nonzero choose opposite transfer argument orders. Forwarded unchanged to the general request dependency. |
| Opaque context | 32-bit value | Forwarded unchanged to the general request dependency; not otherwise interpreted locally. |

### Accessed layout

| Location | Access width | Established use |
|---|---:|---|
| Object + 0x18 | 1 byte | Rejection condition for an eligible nonzero-direction request. |
| Object + 0x58 | 1 byte | Any nonzero value enables consideration of the optional region. |
| Object + 0x5c | 32-bit pointer | Descriptor address, loaded separately at several stages. |
| Descriptor + 0x00 | 32 bits | Region starting block, interpreted as unsigned. |
| Descriptor + 0x04 | 32 bits | Quantity added to the saved region start; loaded as a signed integer. |

The second descriptor word is consistent with a region block count, but no independent layout contract establishes that semantic name. Its addition to the unsigned region start forms an unsigned 32-bit endpoint.

### Result contract

On normal return, the result is one of:

- The result of complete-request delegation.
- The literal rejection value **0x15**.
- Zero, when the selected path has neither a leading nor a trailing general-request operation.
- The leading result, when no trailing operation is required.
- The trailing result, when there was no leading operation.
- The bitwise OR of leading and trailing results.

The region-associated transfer contributes no locally inspected result.

### Preconditions and ownership

The object must be readable at every field actually accessed. Each descriptor pointer used for a content read must identify readable storage at the required offset. The caller buffer and region-associated address must satisfy the dependencies’ access and lifetime requirements.

These conditions are not locally validated. In particular, there is no object-null check, caller-buffer-null check, or repeated null check after later descriptor-pointer loads.

No ownership transfer is established by the local body. Allocation, release, retention, and ownership effects inside dependencies remain unknown.

## Behavior

### Eligibility and complete delegation

The first relevant object access is the one-byte enable field at offset 0x58.

If it is zero, the function does not locally read the descriptor-pointer field or rejection byte before delegating. It calls F-4b651926be02 with all six original arguments and returns that dependency’s result.

If the enable byte is nonzero, the function loads the descriptor pointer at offset 0x5c. A null value also causes complete-request delegation with the original arguments. The rejection byte is not read on this path.

These are local-access guarantees only. The delegated dependency may perform additional accesses or effects under its own unknown contract.

### Direction-dependent rejection

When the enable byte is nonzero and the initially loaded descriptor pointer is nonnull:

- A zero direction selector bypasses the rejection-byte read.
- A nonzero direction selector causes a one-byte read at object offset 0x18.
- A nonzero rejection byte causes an immediate return of **0x15**.

The rejection precedes descriptor-content reads and every dependency call. It does not depend on request position, count, or overlap with the region. A disjoint request can therefore be rejected before selection is considered.

A zero direction selector is not rejected by this field, regardless of its stored value.

### Region observations and selection

After the preceding checks pass, the function obtains region bounds in this observable order:

1. It reads the region start through the initially loaded descriptor pointer.
2. It reloads the descriptor pointer from object offset 0x5c.
3. It reads the second descriptor word through that reloaded pointer.
4. It adds that quantity to the saved region start to obtain the region end.

The two descriptor words need not come from the same descriptor if the pointer changes between observations. No consistency check, locking operation, or snapshot mechanism is visible.

F-83b1e9e0a36b is then called with the original request start, original signed request count, and saved region start. The saved region end is **not** an argument.

A zero selection result causes complete-request delegation with all six original arguments. Bounds already read do not change those delegated arguments.

A nonzero selection result selects partitioned handling. The visible function does not establish what this result means. In particular, it does not independently verify a nonempty, nonwrapping intersection.

### Partitioned handling

The selected path retains a general-request status initialized to zero.

**Leading operation.** If the original request start is below the saved region start under an unsigned comparison, F-4b651926be02 receives:

- The original object.
- The original request start.
- The difference between region start and request start.
- The original caller buffer.
- The original direction selector and opaque context.

The returned status is retained. The working caller-buffer address advances by that difference multiplied by 512.

The leading count is not capped at the original request count. A nonzero returned status does not stop processing: buffer advancement and the region-associated operation still follow if the dependency returns normally.

If the request does not start below the region, there is no leading call, the retained status remains zero, and the working caller buffer remains the original buffer.

**Region-associated operation.** Using the original request values and saved region bounds, the function determines:

- Middle start: the unsigned maximum of request start and region start.
- Request end: the unsigned 32-bit sum of request start and request count.
- Middle end: the unsigned minimum of request end and region end.
- Middle count: middle end minus middle start.

These calculations occur after any leading call. The saved bounds are not refreshed after dependency side effects.

The function reloads the descriptor pointer from object offset 0x5c and passes that value to F-03b6e09474a7. Its returned value is used as a memory base. The region-side address is that base plus 512 times the difference between middle start and saved region start.

Consequently, the region-side address can combine earlier bounds with a base obtained for a later descriptor. No local check establishes their consistency.

The external transfer dependency receives:

| Direction selector | First argument | Second argument | Third argument |
|---|---|---|---|
| Zero | Working caller-buffer address | Region-side address | Middle count scaled by 512 |
| Nonzero | Region-side address | Working caller-buffer address | Middle count scaled by 512 |

This specifies argument order, not source/destination semantics. Those semantics depend on the external dependency’s unavailable contract.

The transfer call is made even when the middle count is zero. Its result, if any, is ignored.

**Trailing operation.** After the external transfer returns, the function compares request end with saved region end using an unsigned comparison.

If request end is no greater than region end, the function returns the retained leading status.

Otherwise, F-4b651926be02 receives:

- The original object.
- Saved region end as the starting block.
- Request end minus saved region end as the count.
- The working caller buffer advanced by middle count scaled by 512.
- The original direction selector and opaque context.

The returned status is bitwise OR-combined with the retained leading status. This is neither first-error selection nor last-error selection.

### Widths and boundary behavior

The visible model uses 32-bit integers and addresses. No wider arithmetic or explicit overflow checks are shown.

Established arithmetic distinctions matter:

- Request-end addition combines an unsigned start with a signed count, producing an unsigned 32-bit result. A negative count is not locally rejected.
- Region-end addition likewise produces an unsigned 32-bit result.
- Bound comparisons and the selection of middle endpoints are unsigned.
- Middle-end subtraction is unsigned before its result is held in a signed 32-bit temporary.
- Leading differences are also held in signed integer storage before their use in buffer advancement.
- Some byte-count and caller-buffer calculations use signed integer expressions. The supplied source view does not justify a portable signed-overflow guarantee.
- The region-side displacement is formed from unsigned block subtraction and scaling, then combined with the returned address in the visible 32-bit model.

Accordingly, the contract does not promise safe modulo-address behavior for every oversized signed product or address calculation. Nor do unsigned minimum and maximum operations make wrapped intervals behave like ordinary contiguous intervals.

For the ordinary contiguous-region interpretation, sufficient preconditions include nonnegative counts, nonwrapping endpoints, representable byte displacements, adequate accessible storage, and a selection result consistent with safe partitioning. The function does not visibly enforce those conditions.

## Side effects

### Locally established effects

The function performs the conditional object and descriptor reads described above and invokes dependencies in the stated sequence.

There is no visible direct write to object fields, descriptor fields, or global storage. The transfer dependency receives addresses through which memory may be modified, but its exact access semantics are not supplied.

No local allocation, deallocation, retain, or release operation is visible.

### Dependency-visible state and continuation

Dependencies must not be assumed to preserve globals, object fields, descriptor pointers, descriptor contents, or ownership.

Observable sequencing includes:

- Selection runs after bound acquisition and before any leading operation.
- A leading operation, when present, completes before the later descriptor reload used for base acquisition.
- Base acquisition completes before the external transfer.
- The external transfer completes before the trailing operation, when one is required.

A dependency may therefore change state consumed by a later dependency. The function nevertheless retains the previously acquired bounds and original request values.

There is no rollback. A nonzero leading status does not suppress subsequent operations. No local status handling detects failure of base acquisition or the external transfer.

Faults, exceptions, nonlocal exits, and dependency behavior after invalid arguments are not defined by the visible normal-return paths.

## Dependencies

| Opaque ID | Established interaction | Contract not supplied |
|---|---|---|
| F-83b1e9e0a36b | Receives request start, request count, and saved region start. Zero selects complete delegation; nonzero selects partitioning. | Predicate, boundary guarantees, side effects, and ownership effects. |
| F-03b6e09474a7 | Receives a freshly loaded descriptor pointer; its return is used as a region-memory base. | Address validity, accessible extent, failure representation, lifetime, allocation, and ownership effects. |
| F-4b651926be02 | Receives a complete request or a leading/trailing portion. Direction and context are unchanged. Its result is returned directly or OR-combined. | Operation semantics, status meanings, partial completion, validation, and state or ownership effects. |

One further external dependency receives two ordered addresses and the scaled middle count. **No opaque function ID for it is supplied.** Its identity and contract remain **documented_only**, including argument roles, overlap support, alignment requirements, zero-length behavior, result convention, and failure behavior.

## Acceptance examples

These are conditional static expectations, not executed tests. Unless stated otherwise, assume stable descriptors, valid memory, representable arithmetic, and normally returning dependencies.

### Disabled region

With a zero enable byte, the function delegates exactly once to F-4b651926be02 using the original six arguments and returns its result.

Before that call, this function does not read the descriptor-pointer field or rejection byte. It does not call selection, base acquisition, or the external transfer.

### Null initial descriptor

With a nonzero enable byte and null initially loaded descriptor pointer, complete delegation occurs even if the direction selector and rejection byte are nonzero.

The rejection byte is not locally read; this case does not return 0x15 merely because that byte is set.

### Direction rejection before selection

With a nonzero enable byte, nonnull initial descriptor, nonzero direction selector, and nonzero rejection byte, the result is 0x15.

No descriptor-content read or dependency call occurs. This remains the result even when the requested interval would be disjoint from the region.

### Selection declines partitioning

After eligibility passes and bounds are acquired, a zero result from F-83b1e9e0a36b causes one complete-request call with the original arguments. There is no base-acquisition or external-transfer call.

### Request wholly inside the region

For region [100, 110), request [102, 105), and nonzero selection:

- No leading or trailing call occurs.
- The region-side address is the returned base plus 1,024 bytes.
- The external transfer’s third argument is 1,536.
- Zero direction places the caller buffer first; nonzero direction places the region-side address first.
- Normal completion returns zero, regardless of any ignored transfer result.

A nonzero-direction example additionally requires the rejection check to pass.

### Request spanning both boundaries

For region [100, 110), request [98, 113), and nonzero selection:

1. The leading call uses start 98, count 2, and the original buffer.
2. The external transfer uses the buffer advanced by 1,024 bytes, the region base, and a byte count of 5,120.
3. The trailing call uses start 110, count 3, and the original buffer advanced by 6,144 bytes.

Leading and trailing results of 0x02 and 0x04 produce 0x06. The nonzero leading result does not suppress either later operation.

### Zero-count selected request

For region [100, 110), request start 102, count zero, and nonzero selection, the middle count is zero. Base acquisition and the external transfer still occur, with a zero third argument to the transfer. No leading or trailing call occurs.

Whether the external dependency accepts the supplied addresses at zero length is unknown.

### Inconsistent selection

For region [100, 110), request [90, 95), and nonzero selection, the function attempts a leading count of 10 despite an original count of 5. The unsigned middle-count subtraction underflows before conversion to signed storage.

This demonstrates absent local validation, not a supported safe transfer or a portable oversized-byte-count result.

### Descriptor replacement during a dependency

If selection or the leading dependency replaces the object’s descriptor pointer, subsequent base acquisition receives the replacement pointer. Region bounds remain those saved earlier.

This expectation does not imply that combining those bounds with the replacement’s returned base is safe.

## Uncertainties

The following remain **documented_only**:

- Semantic object and descriptor types, ownership roles, and lifetimes.
- Whether the optional region is a cache or another mapping mechanism.
- The selection predicate and its guarantees for zero, negative, disjoint, wrapped, or oversized requests.
- The external transfer dependency’s opaque identity and contract.
- Whether zero direction means reading and nonzero direction means writing.
- The meaning of 0x15 and all general-request status values.
- Whether OR-combined statuses have an intended structured interpretation.
- Synchronization, reentrancy, and supported concurrent mutation.
- Behavior for invalid pointers, unsupported counts, signed overflow, and inaccessible transfer extents.
- Dependency mutation of globals, memory, descriptors, or ownership.

The transfer argument reversal and 512-byte scaling support an inferred block-transfer intent. They do not resolve these missing contracts.

## Evidence

This contract is based on static inspection of the supplied function and opaque dependency mapping.

Established evidence includes field offsets and access widths, conditional read order, separate descriptor-pointer loads, exact delegation conditions, rejection before dependency calls, dependency argument order, unsigned endpoint comparisons, 512-byte scaling, continuation after a nonzero leading result, and bitwise-OR status aggregation.

The unconditional “otherwise delegate” interpretation is not supported: the eligible nonzero-direction rejection path returns 0x15 without delegation.

No runtime tests have been performed. No dependency implementations, external transfer contract, independent layout documentation, or synchronization evidence were supplied.
