<!-- Trusted provenance: F-5abdae2e4fba; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5abdae2e4fba

## Purpose

F-5abdae2e4fba conditionally initializes two shared measurements, prepares temporary records, and dispatches those records to dependencies according to a signed mode and repeated observations of an input flag.

Three decisions are independent:

- Mode zero selects one dispatch sequence; every nonzero mode selects another.
- Mode exactly one selects alternate values for certain record fields.
- Bit 0 of an input byte is read at three separate points: twice for record-field selection and once for an optional final dispatch.

The established behavior consists of memory accesses, arithmetic, temporary-record preparation, and dependency calls. Allocation, object construction, measurement meaning, and visual positioning are inferred intent, not established contracts.

## Interface

### Arguments and result

| Argument | Representation | Established use |
|---|---|---|
| Input record | 32-bit address | Direct reads at offsets 4 and 8; forwarded to F-ed4b1fd8953e. |
| Mode | Signed 32-bit integer | Compared separately with zero and one. |
| Forwarded argument A | Opaque 32-bit value | Passed unchanged to every record consumer. |
| Forwarded argument B | Opaque 32-bit value | Passed unchanged to every record consumer. |

The directly accessed input fields are:

| Offset | Width | Interpretation |
|---|---:|---|
| 4 | 8 bits | Only bit 0 affects this function’s decisions. |
| 8 | 32 bits | Signed value used in the position calculation. |

The remainder of the input layout is unknown. Dependencies receiving its address may access additional fields.

There is no explicit return value. Consumer results are ignored. Results from F-8157a180d901 are forwarded to consumers, not returned to the caller.

### Preconditions

Normal completion requires readable input fields, global slots, and every subsequently dereferenced address. Conditional initialization additionally requires:

- A readable signed 32-bit guard at offset 40 through the independently reloaded cache-root address.
- Writable 32-bit fields at offsets 40 and 44 through the initially captured cache-root address.
- Lookup results readable at offset 8.
- Nested records readable at offsets 0, 4, and 12.
- Usable measurement addresses after clearing bit 0 of a dependency-produced word.
- Readable signed 16-bit fields at offsets 82 and 84 of those measurement addresses.

Dependency-specific requirements are unavailable. The function does not explicitly validate these conditions. Alignment, byte order, concurrency guarantees, and aliasing restrictions remain unspecified.

## Behavior

### Cache-root capture and guard

The function first reads a global slot containing a cache-root address and retains that address.

It then reloads the same global slot and reads the signed 32-bit field at offset 40 through the reloaded address:

- If the field is nonzero, both measurement sequences are skipped.
- If the field is zero, both measurement sequences execute.

Subsequent direct cache accesses use the initially retained address, not the guard’s reloaded address. If the global changes between reads, the guard can inspect one object while later cache accesses target another.

Offset 44 is not independently tested.

### Conditional measurement initialization

When the guard observes zero, the first measurement is prepared as follows:

1. F-9d0ecc5f6adf is called without arguments. Its opaque 32-bit result is retained as context for both measurements.
2. F-ed4b1fd8953e receives the input-record address and the value of the first lookup-key global.
3. After that call, a resource-domain global is read and its value retained for the second measurement.
4. A nested-record address is read from offset 8 of the lookup result.
5. F-5710a8628cba receives the retained context, a fresh read of the resource-domain global, and the nested record’s 32-bit word at offset 0.
6. After that call returns, F-59c1430f85c2 receives temporary output storage, the resolver result, the low 16 bits of the nested record’s word at offset 4, and the byte at offset 12.
7. The first 32-bit output word is treated as an address with bit 0 cleared.
8. Signed 16-bit fields at offsets 82 and 84 of that address are sign-extended and added. The signed 32-bit sum is stored at offset 40 of the retained cache root.

The captured resource-domain value and the fresh value supplied to the first resolver are separate observations. No dependency call separates those two global reads, but equality is not established.

The nested-record address is retained across the resolver call. Its selector fields are read afterward; the resolver is not assumed to preserve their contents.

The second measurement then proceeds:

1. F-ed4b1fd8953e receives the input-record address and the second lookup-key global’s value.
2. A nested-record address is read at offset 8 of that lookup result.
3. F-5710a8628cba receives the original context, the previously captured resource-domain value, and the nested record’s word at offset 0.
4. F-59c1430f85c2 receives temporary output storage, the resolver result, the low 16 bits of the nested word at offset 4, and the byte at offset 12.
5. The first output word is masked by clearing bit 0; signed 16-bit fields at offsets 82 and 84 are added.
6. The sum is stored at offset 44 of the retained cache root.

Each measurement sum is in the inclusive range −65,536 to 65,534 and therefore fits in signed 32-bit arithmetic.

The first cache store precedes the second lookup. Initialization is not atomic, and no rollback appears. Dependencies may subsequently change either cache field or the root global.

A zero first sum is stored as zero. This alone does not establish that a later invocation will initialize: that invocation makes its own guard observation.

### Seven base records

An external dependency, identified here as U-01, is invoked seven times with count 104 and different temporary destinations.

Each of these seven calls obtains its source argument from a different global slot. **Distinct source-global locations do not establish distinct source addresses or distinct contents.** The loaded values may alias.

The destination labels used below describe their roles in this function, not object types:

1. Primary-zero.
2. Primary-nonzero.
3. Center.
4. Companion.
5. Conditional.
6. Secondary-zero.
7. Secondary-nonzero.

U-01 appears to transfer data from source to destination. Statements about copied bytes are conditional on that interpretation; its implementation is unavailable.

#### Primary record selections

After the primary-zero transfer:

1. The table-base global is read and retained.
2. The input flag byte is read.
3. Index 1 is selected only when bit 0 is set and mode equals one; otherwise index 0 is selected.
4. The table-base global is read again.
5. The selected 32-bit entry, at four times the index from this freshly read base, is read and written at primary-zero offset 72.

The primary-nonzero transfer follows. After it:

1. The input flag byte is read again.
2. The same index rule is applied independently.
3. The selected 32-bit entry is read through the previously retained table base.
4. That value is written at primary-nonzero offset 72.

The second selection does not reload the table-base global and does not reuse the first entry value. Flag changes, table-content changes, or differing table-base observations can make the assigned values differ.

#### Captured arithmetic inputs and mode value

The function next reads, in order:

1. Signed 32-bit cache offset 40 through the retained cache root.
2. Signed 32-bit cache offset 44 through that root.
3. Signed 32-bit input-record offset 8.

Call these captured values \(L\), \(R\), and \(W\).

It then unconditionally reads the default mode-specific global. Only afterward does it test whether mode equals one. For mode one, it also reads the alternate global and uses that value instead.

Thus, mode one performs both global reads, default first; every other mode performs only the default read. The selected value is an opaque 32-bit field value.

#### Center and remaining preparation

After the center transfer, the function writes:

| Offset | Width | Value |
|---|---:|---|
| 60 | 32 bits | Calculated position \(P\). |
| 64 | 32 bits | Selected mode-specific value. |

The position calculation is:

\[
P=L+\operatorname{trunc}_0\left(\frac{(W-(L+R))-2}{2}\right)
\]

Division truncates toward zero. The expression uses signed 32-bit arithmetic. This formula describes established behavior when every intermediate addition and subtraction is representable; overflow behavior is unresolved.

The companion transfer follows, after which the selected mode-specific value is written at companion offset 64. The conditional, secondary-zero, and secondary-nonzero transfers follow in that order.

All seven base transfers occur before the mode-dependent dispatch branch. Later changes to globals or cache fields do not themselves change the already captured values used for these explicit assignments. This does not guarantee that dependencies preserve the temporary records.

### Mode-zero dispatch

Every record consumer receives these five arguments, in order:

1. The opaque 32-bit result of the immediately associated F-8157a180d901 call.
2. Literal zero.
3. Forwarded argument A.
4. Forwarded argument B.
5. The indicated temporary-record address.

For mode zero, the sequence is:

| Order | Argument to F-8157a180d901 | Consumer | Record |
|---|---:|---|---|
| 1 | 560 | F-60fe53f7ab39 | Primary-zero |
| 2 | 544 | F-7f6becd152e6 | Center |
| 3 | 164 | F-60d99bc349ed | Additional record |
| 4 | 560 | F-60fe53f7ab39 | Secondary-zero |
| 5 | 544 | F-7f6becd152e6 | Companion |

After consumer 2 returns, U-01 receives another temporary destination, a source loaded from an additional global slot, and count 92. This occurs before the supplier call with argument 164.

The fourth supplier is called within the mode branch; its consumer is called at the shared continuation. No dependency call intervenes between them.

The supplier arguments resemble sizes, but byte units and allocation semantics are not established.

### Nonzero-mode dispatch

Every nonzero signed mode, including negative modes, selects:

| Order | Argument to F-8157a180d901 | Consumer | Record |
|---|---|---|---|
| 1 | 560 | F-60fe53f7ab39 | Primary-nonzero |
| 2 | Value read from a global slot | F-4a73b9be0f3a | Extended center |
| 3 | 560 | F-60fe53f7ab39 | Secondary-nonzero |
| 4 | 544 | F-7f6becd152e6 | Companion |

The consumer argument pattern is the same as in mode zero.

After the first consumer returns, extended-center preparation occurs in this order:

1. Write 32-bit extension fields at offsets 104, 108, 112, 116, and 120: respectively 30, a value loaded from another global slot, 2, 25, and 0.
2. Invoke U-01 with the extended-center destination, the prepared center record as source, and count 104.
3. Write the 32-bit value 34 at offset 48.

Under the apparent byte-transfer interpretation, the prepared record has this layout:

| Offset | Width | Contents |
|---|---:|---|
| 0–103 | 104 bytes | Center contents, except the replacement at offset 48. |
| 48 | 32 bits | 34. |
| 104 | 32 bits | 30. |
| 108 | 32 bits | Value read from the extension-value global. |
| 112 | 32 bits | 2. |
| 116 | 32 bits | 25. |
| 120 | 32 bits | 0. |

The count-104 transfer does not cover the extension fields under that interpretation. The consumer’s actual read extent is unknown.

### Optional final dispatch

After the companion consumer returns, the function reads the input flag byte a third time.

If bit 0 is clear, there are no further visible calls.

If bit 0 is set:

1. Read a global slot containing another data-object address.
2. Read the 32-bit field at offset 24 of that object.
3. Write that value at conditional-record offset 100.
4. Call F-8157a180d901 with argument 560.
5. Call F-60fe53f7ab39 with that result, zero, the two forwarded arguments, and the conditional-record address.

This decision depends only on the final flag observation, not on either earlier observation.

## Side effects

### Direct writes and shared state

Outside invocation-local storage, the explicit writes are the conditional stores at offsets 40 and 44 through the initially retained cache-root address.

There is no explicit store addressed through the input-record argument. Aliasing or dependency effects may nevertheless change input memory or other shared state.

No synchronization, reentrancy guard, transaction, or rollback is visible. Repeated global reads must not be collapsed into one assumed stable observation.

### Ownership and lifetime

F-8157a180d901 is called:

- Five times for mode zero, plus once if the final flag is set.
- Four times for nonzero mode, plus once if the final flag is set.

Its results are not checked or directly dereferenced here. Each is passed unchanged to a consumer.

Allocation, ownership transfer, registration, and release behavior are unknown. No explicit deallocation appears, but that does not establish a leak.

Record storage is invocation-local. Temporary storage is reused across preparation roles; dependencies must not be assumed to receive independently durable storage. Retention of a supplied address beyond its usable lifetime would require a dependency contract not provided here.

### Errors

There is no explicit error return, null-result check, recovery branch, or cleanup path.

An unusable lookup result can reach an indirect read. An unusable supplier result is forwarded unchanged. Dependency failure, termination, and state-mutation conventions are unknown.

## Dependencies

| Opaque ID | Established interaction |
|---|---|
| F-9d0ecc5f6adf | Supplies the context retained for both measurement sequences. |
| F-ed4b1fd8953e | Receives the input-record address and a lookup-key value; its result is dereferenced at offset 8. |
| F-5710a8628cba | Receives context, a resource-domain value, and a nested-record word. |
| F-59c1430f85c2 | Receives temporary output storage, a resolver result, a masked 16-bit selector, and a byte selector. |
| F-8157a180d901 | Receives one integer or global-loaded value; supplies an opaque 32-bit result. |
| F-60fe53f7ab39 | Consumes primary, secondary, or conditional records. |
| F-7f6becd152e6 | Consumes center or companion records. |
| F-60d99bc349ed | Consumes the additional mode-zero record. |
| F-4a73b9be0f3a | Consumes the extended-center record. |
| U-01 | Receives destination, source, and count; apparent transfer role. |

U-01 is a document-local opaque identifier because no mapped ID was supplied for this dependency. Its copying behavior and other side effects are not independently established.

No direct recursive call appears.

## Acceptance examples

These are conditional static examples, not runtime results. Unless stated otherwise, dependencies return normally, U-01 performs the apparent byte transfers, relevant memory remains stable, and arithmetic does not overflow.

### Mode zero with a clear flag

Given a nonzero guard:

- Measurement initialization is skipped.
- Both primary fields select table entry 0.
- Only the default mode-specific global is read.
- Supplier arguments are 560, 544, 164, 560, and 544, paired with the listed consumers.
- The optional final dispatch is absent.

### Mode one with a set flag

Both primary selections use entry 1. The default mode-specific global is read before the alternate, and the alternate value is assigned to center and companion.

The nonzero dispatch runs, including extended-center preparation. The final set flag enables one additional supplier/consumer pair.

### Other nonzero modes

Mode 2 or mode −1 selects the nonzero dispatch but uses primary entry 0 and the default mode-specific value. A set final flag still enables the optional dispatch.

### Aliasing source globals

If two or all seven base-source global slots contain the same address, their source arguments may be equal. This does not contradict the preparation sequence: the established distinction is between global slots, not loaded addresses or template contents.

### Different cache roots and resource domains

If the initial root read yields address A and the guard reload yields B, zero at B+40 triggers initialization whose explicit cache stores target A+40 and A+44.

If the captured resource-domain value differs from the subsequent fresh read, the first resolver receives the fresh value and the second receives the captured value.

### Different table bases

If the retained table base is A and the first selection’s reload yields B, primary-zero reads its selected entry through B. Primary-nonzero reads through A after its transfer, even if that transfer changes the table-base global.

### Signed rounding and measurement bounds

For \(L=10\), \(R=20\), and \(W=101\), the numerator is 69 and \(P=44\).

For \(L=10\), \(R=20\), and \(W=29\), the numerator is −3 and \(P=9\), not 8.

Two measurement fields of −32,768 produce −65,536. Two fields of 32,767 produce 65,534. Neither sum overflows signed 32-bit arithmetic.

### Zero measurement and changing flags

A zero first measurement explicitly stores zero at retained-root offset 40. A later invocation repeats initialization only if its own guard read observes zero.

If early flag reads are clear but a consumer sets bit 0 before the final read, entry-0 primary selections can coexist with the optional final dispatch.

## Uncertainties

The following remain **documented_only**:

- Global values, lookup-key meanings, source contents, and table entries.
- Equality or aliasing relationships among source addresses loaded from distinct global slots.
- U-01’s exact transfer, failure, and side-effect contract.
- Whether supplier arguments denote byte sizes or its results denote allocations.
- Whether consumers create objects, retain records, adopt supplier results, or release resources.
- Dependency preservation of globals, input fields, nested records, and temporary storage.
- Tagged-address meaning beyond clearing bit 0.
- Alignment, byte order, concurrency rules, and aliasing restrictions.
- Signed-overflow behavior in the position calculation.
- Relative ordering of operand reads not independently sequenced by the supplied body.
- Consumer read extents and any visual interpretation.

Unknown dependency effects must not be replaced with assumptions of global preservation, successful allocation, or memory ownership.

## Evidence

Established static evidence supports the access widths and offsets, separate global observations, retained addresses and values, branch conditions, arithmetic structure, explicit stores, dependency arguments, and call sequence described above.

For the seven base preparations, the evidence establishes seven distinct source-global locations. It does not establish distinct loaded source addresses, nonoverlapping source regions, or differing contents.

Allocation, object assembly, cached dimensions, and centering remain inferred intent rather than dependency guarantees.

This specification is based on the supplied static body and dependency mapping. No runtime execution or tests have been performed.
