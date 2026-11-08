<!-- Trusted provenance: F-8b52d3be0ca9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8b52d3be0ca9

## Purpose

F-8b52d3be0ca9 modifies a caller-supplied object, invokes dependencies on two embedded components, and selects between an alternate early-return path and a normal configuration-and-dispatch path.

The normal path classifies an object field, reads configuration data, configures one component, conditionally updates another field, publishes a one-byte mode, and performs an indirect method call. The alternate path invokes three dependencies and returns before those normal-path operations.

This contract describes established static behavior. “Component,” “configuration,” and “mode” are descriptive labels, not established application semantics. Dependency implementations, complete object layouts, and runtime behavior are unavailable.

## Interface

### Input and object layout

The sole argument is used as the base address of mutable object storage. Its supplied representation is a 32-bit integer used for address calculations; that representation does not establish a portable integer-based public interface.

The visible object-relative accesses are:

| Offset | Access width or form | Use |
|---|---|---|
| `0x08` | Embedded-region address | Passed to F-1216949dec94 on the alternate path |
| `0x34` | Embedded-region address | Passed to F-338d16f9ce72 and F-1333331b2da1 |
| `0x68` | 1 byte | Written with zero during common initialization |
| `0x69` | 1 byte | Written with zero during common initialization |
| `0x6A` | 1 byte | Written with one during common initialization |
| `0x6C` | Embedded-component address | Passed to initialization, configuration, and processing dependencies |
| `0x98` | Embedded-component address | Passed to an initialization dependency |
| `0xD0` | 4 bytes | Read and passed to F-342542431c39 on the normal path |
| `0xD8` | 4 bytes | Compared with one and conditionally written with one |

An embedded address establishes neither the extent of that component nor its permitted mutations. No direct access to offset `0x6B` is shown.

### External storage

The normal path also accesses:

- One four-byte default configuration source, read unconditionally after classification returns.
- Five separate four-byte override sources, designated A through E. Exactly one is read for classifications `0x6034` through `0x6038`, respectively.
- A global pointer through which a four-byte value is read during mode-4 preparation.
- A global destination pointer through which the selected mode is written as one byte.
- A returned object's first four-byte word and a four-byte callable entry at byte offset `0x20` in its dispatch table.

The configuration-source labels identify selection sources, not known numeric values. Their contents need not differ. No immutability or independence from dependency effects is established.

### Output and widths

There is no explicit return value. Observable outputs are memory accesses, direct writes, dependency calls, and final indirect dispatch.

The supplied representation uses eight-bit byte accesses and four-byte values for object fields, configuration arguments, classification, and dispatch arguments. The branch-selector and conditional-query results are represented as 32-bit integers. Pointer use in the visible dispatch is four bytes wide; the exact target ABI remains unresolved.

No signed ordering, multiplication, or size computation controls the behavior. Comparisons use equality or inequality with fixed values. Address additions identify fixed object or table offsets; the contract does not authorize wrapping or otherwise invalid address arithmetic.

### Preconditions

For the selected path to execute normally:

- The object base and accessed offsets must designate suitable live storage.
- The initial byte destinations must be writable.
- On the normal path, `0xD0` and `0xD8` must support four-byte reads, and `0xD8` must support a four-byte write when that branch is taken.
- Embedded addresses must satisfy their dependencies’ requirements.
- The default configuration source must be readable on every normal-path classification, including `0x6032` and `0x6033`, which discard its value.
- A selected override source must be readable when its classification applies.
- Global pointers and their referenced storage must be valid when used.
- The final returned object, its dispatch table, and the callable entry must support the described reads and invocation.

No visible null, bounds, alignment, lifetime, or dispatch-entry checks enforce these requirements. Consequences of invalid storage are not established.

## Behavior

### Common initialization and path selection

Three separate one-byte writes occur in this order:

1. Zero at object offset `0x69`.
2. One at object offset `0x6A`.
3. Zero at object offset `0x68`.

This ordering is not increasing-address order and is not an aggregate object clear.

F-936a8728139d is then called first with the address at offset `0x6C`, and next with the address at offset `0x98`. Both calls precede path selection. Their effects are unknown, so the initial byte values are not guaranteed to survive them.

F-c60499d3eebf is then called without visible arguments. A nonzero result selects the alternate path; zero selects the normal path.

### Alternate path

The alternate path invokes, in order:

1. F-338d16f9ce72 with the address at offset `0x34`.
2. F-1216949dec94 with the address at offset `0x08` and an address into local stack storage.
3. F-1333331b2da1 with the address at offset `0x34`.

The function then returns.

No initialization of the stack storage is visible before its address is passed. Its size, required incoming contents, output role, and pointer-retention rules are unknown.

The caller performs none of the subsequent normal-path reads, configuration calls, conditional field update, mode publication, or dispatch on this path. The alternate-path dependencies are not thereby guaranteed to leave those resources unchanged.

The first and third calls surround the middle call in execution order. Their meanings are not established as locking, unlocking, acquisition, release, or cleanup.

### Normal-path classification and configuration reads

The function reads four bytes at object offset `0xD0` and passes the value to F-342542431c39.

After that dependency returns, the function unconditionally reads the four-byte default configuration source. Classification-specific selection occurs after this read:

| Classification result | Reads after classification returns, in order | Configuration argument | Selected mode |
|---|---|---|---:|
| `0x6032` | Default source | Numeric value 2000; default value discarded | 1 |
| `0x6033` | Default source | Numeric value 2000; default value discarded | 4 |
| `0x6034` | Default source, then override A | Value read from A | 4 |
| `0x6035` | Default source, then override B | Value read from B | 4 |
| `0x6036` | Default source, then override C | Value read from C | 4 |
| `0x6037` | Default source, then override D | Value read from D | 4 |
| `0x6038` | Default source, then override E | Value read from E | 4 |
| Any other result | Default source | Value read from default source | 4 |

Thus, the default-source read is not exclusive to the default classification case. It also occurs when a literal or override subsequently replaces the value.

For classifications `0x6034` through `0x6038`, only the corresponding override is read; the representation does not read all five sources. The default read precedes that selected override read.

These are reads performed after F-342542431c39 returns, not snapshots from before the call. That dependency may affect their contents. No guarantee makes the reads an atomic snapshot or makes the source values stable across other activity.

Only classification `0x6032` selects mode 1. Classification `0x6033` shares its configuration argument, not its mode.

The selected four-byte configuration argument is passed to F-6904f7699031 together with the address at object offset `0x6C`. F-ec4cfa552cce is then called with that same component address. Neither call has a visibly checked result.

### Mode-4 external operations

After component configuration and processing, mode 4 invokes two unmapped dependencies:

1. F-U01 with numeric arguments `1` and `0x20`.
2. F-U02 without visible arguments.

Mode 1 skips both.

F-U01 and F-U02 are document-local opaque function IDs, not recovered dependency identities. Their results are not visibly consumed. Their memory effects, failure behavior, and purposes are unknown.

### Conditional update at offset `0xD8`

The function next reads four bytes at object offset `0xD8`. This read follows the two external operations when mode 4 applies.

If the field equals one, the function neither calls F-b6a73313bbe9 nor directly writes that field.

Otherwise, it calls F-b6a73313bbe9 with numeric argument `0x3D`:

- A zero result causes a four-byte write of one at `0xD8`.
- A nonzero result causes no direct write to that field.

There is no field reread between the dependency return and the conditional write. A zero result therefore causes the caller to write one even if the dependency changed the field. A nonzero result establishes only the absence of this caller's write, not preservation of the earlier field value.

### Mode-specific preparation

Mode 1 requires no additional dependency at this stage.

Mode 4 performs this ordered sequence:

1. Reads a four-byte value through a global pointer.
2. Passes that value to F-be4e5c73e101.
3. Calls F-6622e2509ce3 and retains its four-byte result.
4. Calls F-4ffd7bc0e22a and retains its four-byte result.
5. Calls F-b8522592e971 with the F-4ffd7bc0e22a result first and the F-6622e2509ce3 result second.

The retrieval order therefore differs from the eventual argument order. No ownership or global-preservation properties follow from these calls.

### Publication and indirect dispatch

The locally selected mode determines both the published byte and final four-byte dispatch argument:

| Mode byte | Dispatch argument |
|---:|---:|
| 1 | 0 |
| 4 | 1 |

The mode byte is written through the global destination pointer before F-378e2d0a4ff6 is called.

The result of F-378e2d0a4ff6 is used as an object pointer. Its first four-byte word supplies a dispatch-table address. The callable entry at table byte offset `0x20` is then invoked with:

1. The returned object pointer.
2. The mode-derived four-byte dispatch argument.

F-U03 is the document-local opaque ID for this unresolved indirect target. Offset `0x20` denotes 32 bytes from the table base, or eight four-byte slots, not an established application method number.

The dispatch argument is derived from the local selection, not from a reread of the published global byte. F-378e2d0a4ff6 may modify globals without changing that already selected argument.

The representation also contains a return before publication for a mode byte other than 1 or 4. No visible path produces such a byte; this does not establish a supported third mode.

## Side effects

Directly established writes are:

- The three initial one-byte object writes.
- A conditional four-byte write of one at `0xD8`.
- A one-byte global publication on reachable normal-path completion.

Observable normal-path reads include the object classification field, the unconditional default configuration source, any selected override, the conditional-update field, mode-4 global input where applicable, and the final dispatch-object and table entries.

In particular, discarding the default configuration value does not remove its read from this contract.

Dependencies receive embedded addresses and may modify object storage, globals, or other resources. Their effects cannot be bounded from this function alone. The caller has no visible allocation or deallocation operation; that is not a guarantee about dependencies.

The alternate-path stack address belongs to invocation-local storage. Whether its recipient retains it is unresolved. No ownership transfer or release obligation is established for other supplied or returned addresses.

There is no visible rollback, general error result, exception translation, or recovery path. If a dependency does not return normally, earlier operations have occurred and subsequent operations are not established. The surrounding alternate-path calls do not establish guaranteed cleanup after an interrupted middle call.

The specified order is the order in the supplied sequential representation. It does not establish atomic accesses, thread safety, synchronization, or hardware memory-ordering guarantees.

## Dependencies

| Opaque function ID | Established use |
|---|---|
| F-936a8728139d | Receives the two embedded-component addresses before path selection |
| F-c60499d3eebf | Supplies the zero/nonzero path selector |
| F-338d16f9ce72 | First alternate-path call on the region at `0x34` |
| F-1216949dec94 | Receives the region at `0x08` and a local stack address |
| F-1333331b2da1 | Last alternate-path call on the region at `0x34` |
| F-342542431c39 | Classifies the four-byte object-field value; returns before configuration-source reads |
| F-6904f7699031 | Receives the component address and selected configuration argument |
| F-ec4cfa552cce | Subsequently receives the same component address |
| F-U01 | Unmapped mode-4 call with arguments `1` and `0x20` |
| F-U02 | Unmapped mode-4 call without visible arguments |
| F-b6a73313bbe9 | Conditionally queried with `0x3D` to control the field write |
| F-be4e5c73e101 | Receives the globally obtained four-byte value in mode 4 |
| F-6622e2509ce3 | Supplies the eventual second argument of F-b8522592e971 |
| F-4ffd7bc0e22a | Supplies its eventual first argument |
| F-b8522592e971 | Receives those results in the specified argument order |
| F-378e2d0a4ff6 | Supplies the final dispatch object after mode publication |
| F-U03 | Unresolved indirect target receiving the object and dispatch argument |

Except for explicitly inspected results, dependency results are not visibly used. No dependency is assumed to preserve globals, retain object lifetime, or allocate or release memory in any particular way.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. They assume valid required storage and normal dependency returns.

1. **Nonzero path selector.**  
   The initial byte writes and both component calls precede the selector. The three alternate-path calls then occur in order. The caller does not read the default configuration source or perform normal-path publication.

2. **Classification `0x6032`, update field already one.**  
   After classification returns, the default source is read even though its value is discarded. The component receives configuration 2000. No override source is read. Both unmapped external calls, the conditional query, and mode-4 preparation are skipped. Byte 1 is published before dispatch with argument 0.

3. **Classification `0x6033`, update field not one, query returns zero.**  
   The default read occurs and is superseded by 2000. Mode 4 applies. F-U01 and F-U02 precede the field read and query. The caller writes four-byte one, performs mode-4 preparation, publishes byte 4, and dispatches with argument 1.

4. **Classification `0x6034` changes configuration storage during classification.**  
   The default source is read after the classification dependency returns; override A is read afterward and supplies the configuration argument. Neither value is defined by its contents before the dependency call. No other override is read.

5. **Classification `0x6038`, update field not one, query returns nonzero.**  
   The default source is read first, then override E. The caller omits its own update-field write. Dependency-mediated field changes remain unspecified. Mode-4 preparation, publication, and dispatch still follow.

6. **Unlisted classification.**  
   The post-classification default-source value is used without an override read. The function does not reject the classification solely because it is unlisted. Mode 4 applies.

7. **Conditional query changes the update field.**  
   If the earlier field read was not one and the query returns zero, the caller writes one without rereading, regardless of an intervening dependency mutation.

8. **Object retrieval changes the published byte.**  
   Publication precedes F-378e2d0a4ff6. Any mutation by that dependency does not cause the caller to recompute the dispatch argument from global storage. The final global byte is not guaranteed to equal the published value.

## Uncertainties

### Established static evidence

The supplied representation establishes the stated access widths, object offsets, constants, branch conditions, call ordering, argument relationships, and dispatch structure. It specifically establishes an unconditional default configuration read after classification returns, followed by any selected override read.

### Inferred intent

The first two component calls appear initialization-related. Classification-dependent values appear to configure one component. The published byte appears to express a mode.

These interpretations are not required for the operational behavior and establish no domain-specific meaning.

### documented_only unknowns

- Complete object layout, alignment requirements, and embedded-component extents.
- Exact ABI, endianness, and address semantics beyond the visible four-byte representation.
- Numeric contents, mutability, storage properties, and aliasing relationships of configuration sources.
- Identities and lifetimes of global destinations and returned objects.
- Dependency mutation, ownership, allocation, release, pointer retention, and failure behavior.
- Size and required initial contents of alternate-path stack storage.
- Semantics and failure behavior of F-U03.
- Whether unrecovered executable behavior differs from the visible representation.

These unknowns are not resolved by assuming constant configuration data, harmless callees, or preserved globals.

## Evidence

This contract is derived from the supplied static function representation and dependency mapping. The earlier draft and reviewer feedback are not independent proof; the configuration-read ordering described here is supported directly by the supplied representation.

The supplied material reports removed unreachable regions, an analysis restart, and incomplete recovery of the final indirect transfer. Those reports limit confidence in exhaustive reconstruction; they do not supply additional behavior.

No runtime tests, execution traces, memory observations, or independent implementation inspection have been performed. Acceptance examples describe conditional consequences of the visible operations, not runtime verification.
