<!-- Trusted provenance: F-a7ff56c8db1a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a7ff56c8db1a

## Purpose

F-a7ff56c8db1a writes two caller-supplied, four-byte values into one of two fixed groups within globally addressed storage. It also clears the four-byte field immediately following each supplied value.

The selector distinguishes **zero from nonzero**, not two particular numeric values. Zero selects the lower-offset group; every nonzero value selects the higher-offset group.

The established behavior is field assignment. The supplied evidence does not establish what the fields represent, why the intervening fields are cleared, or whether either group corresponds to a particular device, channel, object, or operation.

## Interface

### Inputs

The function accepts three inputs:

| Input | Representation established by the supplied source | Role |
|---|---|---|
| Selector | Integer | Selects the lower group when equal to zero, otherwise the higher group. |
| First value | Four-byte value, 32 bits | Supplies the first nonzero-offset payload field in the selected group. |
| Second value | Four-byte value, 32 bits | Supplies the second payload field in the selected group. |

“Payload” here identifies an uninterpreted input value; it does not imply a buffer or message.

Both four-byte values are copied without an established signed interpretation, numeric conversion, validation, or dereference. Their bit patterns may therefore be described independently of whether a caller regards them as integers, addresses, flags, or some other representation. No such interpretation is established by this function.

The selector is compared only with zero. The supplied source does not independently specify its ABI width. A 32-bit integer interface is consistent with the presentation, but only the four-byte widths of the two values and destination stores are explicit.

### Implicit input

A global location supplies the base address of the destination storage.

The source first obtains and retains this base before choosing a branch. The first write in either branch uses a separately expressed access to the global base; the remaining three writes use the retained base. This distinction matters if the global can change between those accesses.

### Output

There is no returned value and no explicit success or failure result.

The observable result, for a normally completed invocation, consists of four four-byte stores. No allocation, newly returned object, or ownership transfer is established.

### Preconditions for normal completion

The selected destination addresses must permit four-byte writes. Any applicable alignment, address-range, mapping, and access-permission requirements must be satisfied.

For the ordinary single-base interpretation of the layout, the global base must remain stable across the initial capture and the base access used for the first store. Without that stability, the first store and the remaining three stores can address different storage regions.

No null-base check, bounds check, or destination validation is present.

## Behavior

### Destination layout

The two groups are disjoint when evaluated relative to the same base.

| Selector condition | Offset from base | Width | Stored content |
|---|---:|---:|---|
| Zero | `0x2C` | 4 bytes | First value |
| Zero | `0x30` | 4 bytes | All zero bits |
| Zero | `0x34` | 4 bytes | Second value |
| Zero | `0x38` | 4 bytes | All zero bits |
| Nonzero | `0x3C` | 4 bytes | First value |
| Nonzero | `0x40` | 4 bytes | All zero bits |
| Nonzero | `0x44` | 4 bytes | Second value |
| Nonzero | `0x48` | 4 bytes | All zero bits |

With a stable base, the lower group occupies bytes `0x2C` through `0x3B`, inclusive. The higher group occupies bytes `0x3C` through `0x4B`, inclusive. Each group is 16 bytes, consisting of four consecutive four-byte fields.

These offsets establish a storage layout, not a declared object type. In particular, adjacent fields must not be assumed to form a wider integer, pointer pair, descriptor, or other composite value.

### Read and write sequence

At the source-observable level, the operation has the following ordering:

1. The global base is read and retained.
2. The selector determines which group is selected.
3. The global base is accessed for the destination of the first store, which writes the first value.
4. The retained base is used to clear the next four-byte field.
5. The retained base is used to write the second value.
6. The retained base is used to clear the final four-byte field.

The destination fields are not read before being overwritten. The function does not inspect their existing contents, compare old and new values, merge bits, or conditionally preserve a field.

The source presents four separate stores. It does not establish a transactional update, a single 16-byte write, an atomic pair of payload updates, or synchronization with other observers. Exact machine instruction selection and visibility to concurrent observers are not established by the supplied material.

### Selection boundaries

The branch condition is equality with zero:

- Zero selects offsets `0x2C`, `0x30`, `0x34`, and `0x38`.
- Any nonzero selector selects offsets `0x3C`, `0x40`, `0x44`, and `0x48`.
- If negative selector values are representable by the interface, they select the higher group.
- The selector is not used as an index or multiplied into an address.

There is no distinct rejection behavior for selectors other than zero and one.

### Value handling

Each supplied value is written as a complete four-byte value. A zero payload remains a payload write; it does not suppress any later store.

The two explicitly cleared fields receive 32 zero bits each. Their previous contents do not affect the operation.

There is no payload arithmetic. The only address computations shown are additions of fixed offsets to a base. The source does not establish safe wraparound behavior if a base-plus-offset computation exceeds the representable address range. Normal-operation acceptance therefore requires valid address computations rather than assuming modulo-address semantics.

### Completion and interruption

On normal completion, all four selected stores have occurred in the source-presented order.

The function contains no explicit error branch or recovery path. If an access cannot complete, the supplied source does not define the surrounding platform’s response. It also provides no rollback mechanism for stores already performed. A caller must not treat the absence of a returned status as evidence of transactional success.

## Side effects

With a stable base and ordinary writable storage, the function overwrites exactly the selected 16-byte group. The other group is not explicitly accessed.

This preservation statement concerns the function’s own accesses. It does not guarantee that unrelated agents leave the other group unchanged, nor does it cover special memory whose writes have additional effects.

The function does not explicitly assign to the global base location. However, destination storage could alias that location or other program state; the supplied evidence does not establish non-aliasing. Consequently, an unconditional claim that the global base’s contents remain unchanged would be stronger than the evidence permits.

No allocation, deallocation, reference-count operation, callback, lock, or external function call is shown. The function establishes neither ownership acquisition nor ownership release. Writing a pointer-shaped bit pattern does not, by itself, transfer ownership or release any previously stored resource.

Repeated calls overwrite the selected fields again. There is no accumulated state, counter increment, or preservation of the previous payloads within this function.

## Dependencies

No function dependency is present in the supplied body.

The established dependencies are environmental:

- A readable global base location.
- Destination addresses supporting the required four-byte writes.
- An execution environment with defined behavior for those accesses.
- External coordination, if callers require stable-base behavior or consistent concurrent observation.

No additional opaque function IDs are needed because no callees are shown. Initialization of the global base, allocation of the storage, and interpretation of the written fields remain outside this function’s established contract.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reports of executed tests.

### Zero selector

Given a stable, valid base, selector zero, first value `0x11223344`, and second value `0xAABBCCDD`, normal completion produces:

| Offset | Four-byte field value |
|---:|---:|
| `0x2C` | `0x11223344` |
| `0x30` | `0x00000000` |
| `0x34` | `0xAABBCCDD` |
| `0x38` | `0x00000000` |

The higher group receives no explicit write from this invocation. These are field values, not a specification of byte order.

### Nonzero selector

Under the same storage assumptions, selector seven directs the two values to `0x3C` and `0x44`, and clears `0x40` and `0x48`.

Selector one has the same selection behavior. A representable negative selector also selects this higher group.

### Existing contents are discarded

If the selected group initially contains nonzero values in all four fields, normal completion replaces both payload fields and clears both intervening fields. No initial field content is retained by the operation.

If both supplied values are zero, all four selected fields end with zero bits.

### Base changes between accesses

If an external change occurs after the initial base capture but before the base access used for the first store, the first payload store uses the later base. The remaining three stores use the initially retained base.

Thus, a contract requiring every write to target one storage object must impose base stability externally. The source does not enforce it.

### Invalid destination

If a selected address is not writable, there is no function-defined failure value or rollback guarantee. The exact fault, termination, or other environmental response is documented_only.

## Uncertainties

### Established static evidence

The source establishes the branch condition, fixed offsets, four-byte store widths, payload placement, zero writes, retained-base usage, absence of callees, and absence of an explicit returned result.

### Inferred intent

The adjacent groups appear intended to hold two supplied values with two associated fields reset to zero. Describing them as “groups” is structural; a semantic interpretation of the cleared fields is not established.

A single stable backing object is a plausible intended operating condition, not an enforced invariant.

### documented_only

The following remain unresolved:

- Exact calling convention, selector width, and address width.
- Concrete storage type, allocation extent, initialization, and lifetime.
- Byte order and alignment requirements.
- Atomicity, memory ordering, and concurrent-access guarantees.
- Whether the destination is ordinary memory or special-purpose mapped storage.
- Whether destinations alias the global base or other sensitive state.
- Address-overflow and invalid-access behavior.
- Semantic meaning and ownership implications of the stored values.
- Whether machine execution performs distinct global-base loads corresponding to both source expressions.

## Evidence

This contract is based solely on the supplied function body. Its evidence is static: one initial global-base capture, a zero-versus-nonzero branch, and four four-byte assignments in each branch at the offsets listed above.

No implementation execution, runtime inspection, or tests have been performed. Acceptance examples describe required outcomes under stated preconditions; they do not assert observed results.
