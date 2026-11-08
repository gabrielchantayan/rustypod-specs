<!-- Trusted provenance: F-df15eb1d41a0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-df15eb1d41a0

## Purpose

F-df15eb1d41a0 constructs two compact byte-translation tables in storage obtained through an object reference. It publishes an inclusive input interval for each table.

The first table conditionally converts ASCII lowercase bytes to uppercase. The second conditionally converts ASCII uppercase bytes to lowercase. Conversion requires two nonzero classification results: one for the original byte and another for its proposed counterpart. Otherwise, the routine writes the original byte.

Classification also determines each table’s interval. Every byte within that interval receives an entry, including bytes rejected by classification.

A boundary case is significant: the ascending interval search queries values 0 through 254, never 255. If those queries all return zero, the selected interval is [255, 255], and the routine writes one identity entry, 0xFF, without classifying that byte.

These statements describe this routine’s direct behavior. Final object fields and table contents additionally depend on dependency side effects and storage aliasing. No dependency is established to preserve those locations.

## Interface

### Arguments

| Argument | Established representation | Use |
|---|---|---|
| First | 32-bit integer used as an address-like object reference | Supplies the object for classification and object-relative field accesses |
| Second | Opaque 32-bit value | Copied into temporary storage whose address is passed to preparation and finalization dependencies |
| Third | Opaque 32-bit value | Copied into separate temporary storage; no subsequent explicit read is visible |
| Fourth | Opaque 32-bit value | Copied into temporary storage, passed by address to two dependencies, and supplied by value to another |

The first argument’s application-level type is unknown. Its use requires addressable storage, but the supplied representation does not independently establish a native pointer declaration.

### Object-relative fields

| Byte offset | Direct access | Role |
|---|---|---|
| +0x1C | 32-bit unsigned write | First interval’s selected lower bound |
| +0x20 | 32-bit unsigned write | First interval’s selected upper bound |
| +0x24 | 32-bit unsigned write | Second interval’s selected lower bound |
| +0x28 | 32-bit unsigned write | Second interval’s selected upper bound |
| +0x2C | Pointer read | First table’s destination |
| +0x30 | Pointer read | Second table’s destination |

This is a partial layout, not a complete object definition. Pointer width, byte order, alignment requirements, and unrelated fields are not independently established.

Bounds are 32-bit values. Table entries are individual bytes.

### Results

There is no explicit return value or status result.

Direct outputs comprise four bound-field writes and two sequences of byte writes. Each sequence contains between 1 and 256 entries. There is no directly written table header, terminator, allocation result, or capacity field.

These outputs are not guaranteed to remain intact at return: later dependencies or overlapping writes may modify them.

### Preconditions for normal completion

- The object-relative accesses must refer to valid storage with the required readability or writability.
- Each destination pointer must be valid when captured, and the resulting buffer must remain writable for the selected number of entries.
- Dependencies must return normally and leave subsequent required accesses valid.
- Any dependency access through exposed temporary addresses must be valid under its own contract.

The routine performs no capacity or disjointness checks. Overlap between table buffers, object fields, or other live storage can affect subsequent observations. It provides no snapshot or overlap-safe semantics.

## Behavior

### Preparation and exposed temporaries

The second, third, and fourth argument values are copied into separate 32-bit temporaries before preparation begins.

The observable dependency order is:

1. F-77ea74ddecdb receives the address of the fourth-argument temporary.
2. F-23df8d149813 receives the address of the second-argument temporary, the then-current value of the fourth-argument temporary, and the integer 2.
3. F-8a5aadac202f receives the address of the fourth-argument temporary.

Thus, the value supplied to F-23df8d149813 can reflect modification by F-77ea74ddecdb. Neither address-taking call receives merely an immutable copy of the original fourth argument.

The third-argument temporary has no explicit later read in this routine. Whether a dependency accesses it through a larger representation is unresolved. No such access is required by the visible direct operations.

### Interval selection

Both tables use the same selection rule, with different classification selectors:

- First table: selector 0x10.
- Second table: selector 8.

For each interval, ascending classification queries start at 0 and proceed through consecutive values, stopping at the first nonzero result. The last ascending candidate is 254.

If a nonzero result occurs, that candidate becomes the locally selected lower bound. The upper-bound search starts at 255 and proceeds downward. A candidate is queried only while it is strictly greater than the selected lower bound. The first nonzero result stops the descending search. If all queried candidates return zero, the upper bound reaches the lower bound without querying that lower-bound value again.

If all ascending queries return zero, both locally selected bounds are 255. No descending search occurs, and byte 255 is not queried during selection.

Only zero versus nonzero matters. The magnitude or sign of a nonzero classification result does not otherwise affect selection.

Even when classification results vary over time, the selected bounds satisfy:

\[
0 \leq L \leq U \leq 255.
\]

The interval need not contain only positively classified values.

### First table: publication and construction

After selecting the first interval, the routine directly writes its lower bound to +0x1C, then its upper bound to +0x20. It next reads the destination pointer at +0x2C.

That pointer is captured once. The routine advances its working destination by one byte per entry; it does not reread +0x2C during construction.

Entries correspond to every input byte in the locally selected inclusive interval, in ascending order:

- Outside 0x61–0x7A, the entry is the input byte. No construction-time classification query is made for that entry.
- Inside 0x61–0x7A, selector 0x10 is queried for the input byte.
- A zero result produces an identity entry.
- A nonzero result causes a selector-8 query for the proposed uppercase counterpart, obtained by subtracting 0x20.
- Only a nonzero counterpart result produces the uppercase byte. Otherwise, the original byte is written.

Required construction-time queries occur before the corresponding byte write. An earlier positive interval-selection result is not reused instead of the construction-time query.

For entry index \(j\), starting at zero, the represented input is \(L+j\), where \(L\) is the **locally selected lower bound**. It is not determined by rereading +0x1C. Indexing relative to the published field is equivalent only while that field retains the selected lower bound.

In particular, a dependency’s modification of +0x1C does not itself change the active entry sequence. It can instead make the published metadata disagree with that sequence.

### Second table: publication and construction

The entire first table’s direct byte-write sequence completes before selection of the second interval begins.

After selecting the second interval, the routine directly writes its lower bound to +0x24, then its upper bound to +0x28. It then captures the destination pointer at +0x30 once.

The second table also contains one entry for every input in its locally selected interval, in ascending order:

- Outside 0x41–0x5A, the entry is the input byte, without a construction-time classification query.
- Inside 0x41–0x5A, selector 8 is queried for the input byte.
- A zero result produces an identity entry.
- A nonzero result causes a selector-0x10 query for the proposed lowercase counterpart, obtained by adding 0x20.
- Only a nonzero counterpart result produces the lowercase byte. Otherwise, the original byte is written.

Its entry index \(j\) represents \(L+j\), using the second interval’s locally selected lower bound. Equivalence to indexing from the value stored at +0x24 requires preservation of that field.

After the second table’s last direct entry write, F-7c87d6325a96 receives the address of the second-argument temporary. The routine then returns on the visible normal-completion path.

### Integer widths and boundaries

Selection and iteration use 32-bit unsigned values. Bound fields also receive 32-bit unsigned values.

The ASCII range checks use unsigned subtraction and accept exactly the stated 26-byte ranges. Inputs below a range’s start do not become accepted through negative arithmetic.

Classification input values are reduced to their low eight bits. Guarded conversion arithmetic stays within byte range:

- Lowercase-to-uppercase candidates are 0x41–0x5A.
- Uppercase-to-lowercase candidates are 0x61–0x7A.

Each written entry occupies one byte. The interval length is \(U-L+1\), ranging from 1 to 256. Iteration can advance to 256 for termination, but 256 is neither emitted as an additional entry nor queried as an additional table input.

## Side effects

### Direct writes versus final memory

The routine directly writes each bound field once and writes the selected number of entries through each captured destination.

Those direct operations do not imply unchanged final contents. For example:

- A construction-time classification call can overwrite a bound already published.
- A later classification call can overwrite an earlier table entry.
- A dependency can change a destination-pointer field before that field is captured.
- F-7c87d6325a96 can potentially modify either completed table or any published bound.

Such effects are not established dependency behavior, but neither are they excluded by the supplied evidence. Stable classification return values alone do not establish memory preservation.

A modification to an already captured pointer field does not redirect that table’s remaining writes. A modification to the second destination-pointer field before its capture can affect the second destination.

### Observable order

The direct phase order is:

1. Preparation dependencies.
2. First interval queries.
3. First lower-bound write, first upper-bound write, first destination-pointer read.
4. First table’s queries and byte writes.
5. Second interval queries.
6. Second lower-bound write, second upper-bound write, second destination-pointer read.
7. Second table’s queries and byte writes.
8. Final dependency.

Within a phase, classification calls can occur between successive entry writes. Bounds are published before table construction, not after it.

No synchronization, atomic publication, rollback, or coherent-snapshot mechanism is visible. A concurrent observer is not promised matching bounds and completed table contents.

### Ownership and failure

The visible body performs no direct allocation or deallocation. Obtaining pointers from the object does not establish their owner or lifetime.

Temporary addresses are exposed to dependencies. Whether dependencies retain those addresses, access a larger representation, acquire resources, or release resources is unresolved.

Preparation and finalization results are not inspected as status values. A zero classification result is treated as predicate rejection, not as a distinguished error.

No visible recovery path handles invalid storage, insufficient capacity, dependency failure, or interrupted execution. Partial updates may remain. Finalization is not visibly guaranteed if execution fails before reaching its normal call site.

## Dependencies

| Opaque ID | Observed interface and use | Unestablished aspects |
|---|---|---|
| F-77ea74ddecdb | Address of fourth-argument temporary; first dependency call | Access extent, mutations, ownership, and other side effects |
| F-23df8d149813 | Address of second-argument temporary, current fourth-argument temporary value, integer 2 | Meaning of 2, representation consumed, preparation effects |
| F-8a5aadac202f | Address of fourth-argument temporary after preceding preparation | Whether it releases anything or changes state |
| F-3e0e547bb25a | Object reference, selector 0x10 or 8, byte-range value | Classification semantics and all side effects; only its 32-bit zero/nonzero result is consumed |
| F-7c87d6325a96 | Address of second-argument temporary after both tables’ direct writes | Finalization, restoration, release, and memory-preservation behavior |

No dependency is established to preserve globals, object fields, buffers, or resource ownership. Preparation and finalization are descriptive phase names, not guarantees that state is installed or restored.

No direct recursive call is visible.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests.

Unless an example explicitly permits a mutation, assume:

- Valid, sufficiently large table buffers, disjoint from one another, the object, and exposed temporary storage.
- Normal dependency returns without invalidating live storage or execution state.
- Classification results stable for each selector/input pair.
- **Every dependency, including preparation and finalization, preserves the relevant object fields, destination-pointer fields, and both table-storage regions throughout the call.**

Under these assumptions, this routine’s direct writes determine the final bounds and table contents.

### ASCII classifications

Suppose selector 0x10 accepts exactly 0x61–0x7A and selector 8 accepts exactly 0x41–0x5A.

- First final bounds: [0x61, 0x7A].
- First table: 26 bytes, 0x41 through 0x5A.
- Second final bounds: [0x41, 0x5A].
- Second table: 26 bytes, 0x61 through 0x7A.

Each buffer begins with the result for its locally selected lower bound, which also remains published under the preservation assumptions.

### All queries reject

Suppose every classification query returns zero.

Both final intervals are [255, 255], and each buffer contains one written byte, 0xFF. Each ascending search queries 0 through 254. Neither selection nor construction queries byte 255 on this path.

### Only byte 255 is accepted

Suppose each selector accepts only byte 255.

Final bounds and table bytes are identical to the all-reject example. No query discovers that positive classification. The fallback interval therefore does not prove acceptance of byte 255.

### Missing counterpart

Suppose selector 0x10 accepts only 0x61, while selector 8 accepts only 0x42.

The first interval is [0x61, 0x61], with entry 0x61 because its proposed counterpart 0x41 is rejected. The second interval is [0x42, 0x42], with entry 0x42 because its proposed counterpart 0x62 is rejected.

Source-class membership alone is insufficient.

### Gaps within an interval

Suppose selector 0x10 accepts 0x61 and 0x63, while selector 8 accepts 0x41 and 0x43.

The first table’s three entries are 0x41, 0x62, 0x43. The second table’s three entries are 0x61, 0x42, 0x63. Rejected interior values receive identity entries rather than being omitted.

### Full coverage

Suppose both selectors accept every byte.

Both final intervals are [0, 255], and each table receives 256 entries. Only the appropriate ASCII letter range is converted in each table; other bytes remain unchanged.

### Classification changes

Relax result stability: suppose a lowercase byte accepted during first-interval selection is rejected during its construction-time query. Retain all memory-preservation assumptions.

That byte receives an identity entry. Its earlier positive result does not force conversion. Other bounds and entries depend on the complete ordered results.

### Stable results with an explicit memory mutation

Suppose every classification query returns zero, but permit the first query of the second interval search to overwrite +0x1C with 7 and the first table’s already-written byte with 0xA5. Assume every other dependency effect preserves the relevant storage.

The routine directly writes first bounds [255, 255] and first byte 0xFF before that permitted mutation. At return, however, the first published lower bound is 7 and its table byte is 0xA5. Its upper bound remains 255.

The second final interval is [255, 255], with entry 0xFF. Stable zero results and normal returns do not prevent the first table’s final metadata and contents from differing from this routine’s direct writes.

## Uncertainties

The following remain **documented_only**:

- Application-level meanings of the object and arguments.
- Formal meanings of selectors 0x10 and 8.
- Complete object layout, pointer width, alignment, and byte order.
- Dependency access extents through temporary addresses.
- Whether preparation changes classification state or finalization restores it.
- Dependency ownership, allocation, release, global-state effects, reentrancy, and exceptional behavior.
- Supported aliasing and concurrency arrangements.
- Whether the [255, 255] fallback is intentional, a sentinel convention, or a defect.
- Consumer indexing rules and any external mechanism distinguishing fallback from a genuine singleton interval.

Constructing bounded ASCII case-conversion tables is an **inferred intent** supported by the guarded transformations. General Unicode, multibyte, or locale-complete conversion is not established.

## Evidence

Established behavior derives solely from the supplied static function representation: argument-sized temporaries, object-relative accesses, dependency-call order, unsigned searches and range checks, guarded conversions, and sequential byte writes.

The distinction between locally selected bounds and mutable published fields follows from selection and iteration using local values rather than rereading those fields.

No dependency implementation, caller contract, runtime trace, allocation record, or external layout definition was supplied. No runtime tests have been performed. Acceptance examples state conditional consequences, not observed execution results.
