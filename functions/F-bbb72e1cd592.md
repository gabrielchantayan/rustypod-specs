<!-- Trusted provenance: F-bbb72e1cd592; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bbb72e1cd592

## Purpose

F-bbb72e1cd592 coordinates construction-like dependencies, selects one of two main-object paths, performs two ordered callback passes, invokes a late construction-like operation, and processes temporary selection and lookup state. On normal completion, it returns the retained main-object address.

“Root,” “intermediate,” “main,” “descriptor,” and “selection” identify observable roles, not established application meanings. Allocation, construction, copying, and cleanup are inferred purposes where dependency bodies are unavailable.

The function provides no explicit status result, local allocation-failure handling, or rollback path. Normal completion requires dependencies to return and all subsequent memory accesses and indirect calls to be valid.

## Interface

### Inputs and result

The visible interface has four 32-bit inputs in a 32-bit address model.

| Input | Observed use |
|---|---|
| First | Saved locally; no subsequent semantic use is visible. |
| Second | Supplied to the root construction-like dependency. |
| Third | Repeatedly supplied as opaque construction or callback context. |
| Fourth | Address of an input record accessed throughout execution. |

The result is the retained 32-bit main-object address. It is not a separately computed success indicator.

The input record has these accessed fields:

| Byte offset | Width | Use |
|---|---:|---|
| +0x30 | 32 bits | Early construction-like argument and later lookup-initialization argument. |
| +0x3c | Signed 16 bits | Sign-extended into main +0x20. |
| +0x3e | Signed 16 bits | Sign-extended into main +0x24. |
| +0x5d | 8 bits | Zero/nonzero main-path selector. |
| +0x60 | 8 bits | Independently tested at two stages. |
| +0x6c | 32 bits | Zero/nonzero optional-property selector and, on the nonzero path, a reread argument. |
| +0x70 | Signed 32 bits | Operand of late signed negation. |

These are stage-specific reads, not an entry-time snapshot. Intervening dependencies may change values subsequently read.

### Accessed layouts

| Role | Required accesses |
|---|---|
| Main object | Dispatch-table pointer at +0x00; writable 32-bit fields at +0x20 and +0x24; readable 32-bit field at +0xe8; callable dispatch entries at table offsets +0x1c4 and +0x1c8. |
| Intermediate object | Readable/writable 32-bit flags at +0x48; writable bytes at +0x90 through +0x93. |
| Collection entry | A pointer at +0x08, whose target supplies a readable 32-bit value. |
| Callback object | Dispatch-table pointer in its first word; callable first table entry. |
| Callback table | Slots 0 through 33, with four-byte spacing. |
| Temporary header | 32-bit fields at +0x04, +0x08, and +0x0c. |
| Temporary chain node | 32-bit next pointer at +0x00 and words at +0x04 and +0x08. |

Callback selection also exposes a local 32-bit slot-index cell by address. Before each F-f30a0116f2cc call, that cell is assigned the current slot index. Its contents are not assumed preserved by the dependency.

### Preconditions and ownership

F-8157a180d901 receives size-like arguments in this order: 0xe8, 0xec, 0x98, either 0x1e0 or 0x1f0, then 0xf0 and 0xe8 in each of two passes, and finally 0x580. These values do not independently establish allocation semantics or complete object extents.

Results are forwarded without local null rejection. Every later dereference requires usable storage; every indirect call requires a valid target. No local validation establishes these conditions.

Passing an address does not establish ownership transfer. Dependency retention, destruction, release obligations, and lifetime extension remain documented_only.

## Behavior

### Evidence boundaries

One unmapped external dependency is designated **F-U1**, a document-local opaque ID. Its arguments suggest descriptor copying, but no verified declaration or body is supplied. Consequently, neither copying nor a byte-access extent is established by its calls. Apparent destination contents, source preservation, overlap behavior, and effects elsewhere remain unknown.

Global-sourced arguments are obtained at their respective call stages. No dependency is assumed to preserve globals, input records, objects, or exposed local storage.

### Initial preparation and root path

F-762936c3f077 receives the input-record address and an initial descriptor area.

F-U1 then receives a root descriptor area, a current global-sourced argument, and 0x5c. A second invocation receives root-descriptor +0x1c, another local area, and 0x30. No explicit initialization of that other local area precedes its use. Its contents and any transfer from it are therefore unresolved.

F-8157a180d901 receives 0xe8. F-da04303b789f receives its result, zero, and the second input, with the root descriptor address prepared as accompanying call state. The returned value is retained as the root object.

F-U1 next receives an intermediate descriptor area and a distinct global-sourced argument. No third argument is visible for this invocation.

After that call, the current initial-descriptor word at +0x08 is read, ORed with 0x80000000, and written to intermediate-descriptor +0x10.

If the current input byte at +0x60 is zero, three words from a separate global source are assigned, in source order, to intermediate-descriptor +0x34, +0x38, and +0x3c. These explicit assignments do not depend on presumed F-U1 behavior.

F-8157a180d901 receives 0xec. F-c47b7ab19be0 receives its result, zero, the root object, and the third input, with the intermediate descriptor address prepared as accompanying state. Its result is retained as the intermediate object.

### Optional intermediate property

Input +0x6c is tested for nonzero. On the nonzero path, that field is read again and supplied with a local output area to F-ccabec065f5e.

One output word goes to F-283eda65a14c. Its result goes to F-682adfd5a561 with selector 5. After that returns, F-5710a8628cba receives the result, the current global lookup-key argument, and another output word from F-ccabec065f5e.

A null final result skips the branch’s direct intermediate-object updates. It does not establish that earlier dependencies left the object unchanged.

For a nonnull result, the function reads the word at that address and retains its low eight bits. It then reads intermediate +0x48 and writes that value OR 0x00038000. The retained byte is written to +0x90, +0x91, and +0x92, in that order, followed by 0xff at +0x93. Conversion to a byte truncates; it neither saturates nor validates a range.

### Main-path preparation and initialization

F-8157a180d901 receives 0x98. F-6e66bed3f716 receives its result, zero, and the current input word at +0x30.

The local area used earlier as F-U1’s second argument is then explicitly populated from a six-word global source. Relative to the initial descriptor, four three-word groups begin at +0x1c, +0x28, +0x34, and +0x40. Their respective source-word orders are:

- 0, 1, 2.
- 0, 1, 2.
- 3, 4, 5.
- 0, 1, 2.

No dependency call intervenes among these assignments. They do not establish the area’s contents at its earlier use.

Input +0x60 is read again. If zero, initial-descriptor +0x18 is read and rewritten with bits 0 and 2 cleared and bit 24 set; other bits are retained.

The current byte at input +0x5d selects the main path:

| Observed byte | Size-like request | Selected dependency |
|---|---:|---|
| Zero | 0x1e0 | F-c98ed7b94040 |
| Nonzero | 0x1f0 | F-b7798624998f |

The selected dependency receives the size-request result, the F-6e66bed3f716 result, the intermediate object, and the third input. The initial descriptor address is prepared as accompanying state.

The returned main object’s dispatch entry at +0x1c4 executes. After it returns, input +0x3c is read as signed 16-bit and sign-extended into main +0x20. Input +0x3e is then read and similarly written to main +0x24.

### Temporary selection and fallback

F-ea223a7ae124 executes first. F-326b1fb00786 then receives a temporary lookup destination, that result, the current input word at +0x30, and a distinct current global argument. A null accompanying call-state pointer is prepared.

Several temporary container fields are zeroed. F-84e8874bcd0c receives the address of a container-pointer field and returns a header address. The function writes zero at header +0x04 and the header address itself at +0x08 and +0x0c.

Collection uses an unsigned 32-bit index starting at zero. Every condition evaluation calls F-d0b90560f1fd, reads the current global callback-table base, and compares the index with the unsigned count result.

For each accepted index, F-0ff4e60503e7 receives the lookup state and the index narrowed to signed 16-bit, then sign-extended. The returned entry’s +0x08 pointer is read, followed by the word at that pointer. That word is stored locally, and F-1f5270adb05d receives local output state, container state, and its address. The collection index then advances by one.

Counts are requeried rather than fixed initially. At index 32768, the accessor receives −32768. At 65536, the narrowed argument is zero. Intended validity of such indices is unresolved.

The callback-table base obtained during the terminating condition is retained for fallback and both callback passes. Later scans do not reload the global base.

If the observed selection-state word is zero, fallback examines slots 1 through 24 inclusive, ascending. Each currently nonnull slot causes its numeric index to be stored locally and F-1f5270adb05d to execute. This invocation exposes fewer explicit arguments than the collection invocation; its full calling contract is unresolved. Slots 0 and 25–33 are excluded from fallback.

### Two callback passes

Pass 0 completes before pass 1 begins. Each pass prepares two descriptors and then scans callback slots:

- F-U1 receives the first per-pass descriptor area, its current global-sourced argument, and 0x5c.
- F-8157a180d901 receives 0xf0. F-af14f808666c receives its result, the main object’s then-current +0xe8 word, the main object, and the third input, with the first descriptor address prepared.
- F-U1 receives the second descriptor area, a distinct current global-sourced argument, and 0x5c.
- Second-descriptor +0x58 is read and rewritten OR 4.
- F-8157a180d901 receives 0xe8. F-da04303b789f receives its result, zero, the preceding construction-like result, and the third input, with the second descriptor address prepared.
- F-682adfd5a561 receives that result and selector 0x1100.

Both F-U1 calls recur in pass 1 after pass 0’s callbacks. This establishes fresh calls and argument evaluation, not descriptor reinitialization or equality with template bytes.

Slots 0 through 33 are examined in ascending order. Each slot’s callback-object pointer is read before selection-query dependencies execute. A null pointer skips that slot.

For a nonnull slot, the current slot index is assigned to a local 32-bit cell. F-f30a0116f2cc receives a local lookup destination, the address of container state, and **the address of that slot-index cell**, not the index by value. The dependency can observe the stored index through that address. Preservation of the addressed value is unknown; the cell is assigned the applicable index again before each subsequent call.

After F-f30a0116f2cc returns, a local output pointer and its adjacent word are zeroed. F-d49a210f073f receives the lookup state, another local destination, the output pointer’s address, and zero.

A nonnull output pointer enables dispatch through the previously retained callback object’s first table entry. Arguments are the callback object, the pass number masked to eight bits, the pass’s selector-0x1100 result, and the third input. Neither the gate pointer nor the addressed slot-index value is directly supplied as a callback argument.

The retained callback object is used even if a query changes the table slot. Selection is queried anew in pass 1. After both passes, main dispatch entry +0x1c8 executes.

### Late operation and temporary-state processing

F-U1 receives a late descriptor area, its current global-sourced argument, and 0x70. This does not establish descriptor contents or a 0x70-byte write.

Input +0x70 is read as signed 32-bit, negated, and assigned to late-descriptor +0x48. Representable negations have their ordinary signed result. For −2147483648, the supplied representation does not establish wrapping or another defined result.

The current initial-descriptor word at +0x08 is reread, ORed with 0x80000000, and written to late-descriptor +0x68.

F-8157a180d901 receives 0x580. F-ab1dbc69170e receives its result, zero, the root object, and the third input, with the late descriptor address prepared. Its result is not retained.

F-9c9f8fa3ba70 receives the main object. A nonzero result causes F-8635951b0989 to receive main/zero, then intermediate/one. Zero skips both calls.

If the container header pointer is nonzero, its value and its +0x08 word are placed in local cells. F-89131283692e receives local output and container addresses plus the addresses of those cells. Afterward, the current header’s +0x0c receives the local-chain head, and the header becomes that head.

Within this branch, a separate container-pointer chain is traversed. Each node’s next pointer is installed as the current chain pointer before F-ff1bdc376dcd executes: first with the node’s +0x08 word, +0x04 word, and zero; then with the node address, one, and zero. Chain equivalence and memory-release semantics are not established.

Finally, F-3a8de16627bf receives the temporary lookup structure, and the retained main-object address is returned.

## Side effects

Established direct effects include descriptor assignments, temporary-state initialization, conditional intermediate flags and bytes, main-object field writes, and header/chain updates.

The local slot-index cell is explicitly written before every F-f30a0116f2cc call. Its address is exposed to that dependency; dependency writes to the cell, retention of its address, and resulting effects are unresolved. No preservation guarantee is implied by its role as an input.

Other dependencies and indirect dispatches may have additional effects. No direct input-record or callback-table-slot write is shown, but dependency mutations remain possible.

There is no visible local recovery, rollback, exception handler, or constructor-result validation.

## Dependencies

| Opaque IDs | Observed role |
|---|---|
| F-U1 | Unmapped external operation with pointer and sometimes size-like arguments. |
| F-762936c3f077 | Initial descriptor preparation. |
| F-8157a180d901 | Size-like requests. |
| F-da04303b789f, F-c47b7ab19be0, F-6e66bed3f716 | Early or per-pass construction-like operations. |
| F-c98ed7b94040, F-b7798624998f | Alternative main-object paths. |
| F-ccabec065f5e, F-283eda65a14c, F-5710a8628cba | Optional-property pipeline. |
| F-682adfd5a561 | Selector-based operation. |
| F-ea223a7ae124, F-326b1fb00786 | Temporary lookup preparation. |
| F-d0b90560f1fd, F-0ff4e60503e7 | Repeated count query and narrowed-index access. |
| F-84e8874bcd0c, F-1f5270adb05d | Header acquisition and selection-related operation. |
| F-af14f808666c | First construction-like operation in each pass. |
| F-f30a0116f2cc | Selection query receiving a pointer to an assigned local 32-bit slot index. |
| F-d49a210f073f | Gate query receiving an output-pointer address. |
| F-ab1dbc69170e | Late construction-like operation. |
| F-9c9f8fa3ba70, F-8635951b0989 | State query and conditional operations. |
| F-89131283692e, F-ff1bdc376dcd, F-3a8de16627bf | Temporary-state processing. |

Indirect dispatch targets have no supplied opaque IDs or independent contracts.

## Acceptance examples

These are conditional static criteria, not executed tests. They assume dependencies return and required accesses remain valid.

1. **Main selection:** A zero +0x5d read selects request 0x1e0 and F-c98ed7b94040. Any nonzero byte selects 0x1f0 and F-b7798624998f.

2. **Signed fields:** Halfword reads 0xffff and 0x8000 produce main writes 0xffffffff and 0xffff8000, respectively, after dispatch +0x1c4.

3. **Optional property:** A nonnull lookup addressing 0x123456ab yields intermediate bytes ab, ab, ab, ff. The flag write preserves the bits read and sets 0x00038000. A null lookup causes no direct property writes in that branch.

4. **Fresh flag reads:** Initial-descriptor +0x08 values 0x12 early and 0x34 late produce explicit writes 0x80000012 and 0x80000034.

5. **Fallback boundary:** With zero observed selection state and only slots 0, 1, 24, and 25 nonnull during fallback, only 1 and 24 trigger fallback calls. Later scans include all four.

6. **Callback-query indirection:** At a nonnull slot 7, F-f30a0116f2cc’s third argument addresses a 32-bit cell containing 7 on call entry. It is not the numeric argument 7.

7. **Index-cell mutation:** If that query changes the cell to 99, preservation is not required. At the next nonnull slot 8, the cell is assigned 8 before the next query. A successful gate for slot 7 still dispatches through the previously retained callback object; 99 is not directly passed to the callback.

8. **Callback ordering:** Stable entries and successful gates only at slots 2 and 7 produce pass-0/2, pass-0/7, pass-1/2, pass-1/7 order.

9. **Per-pass descriptors:** If second-descriptor +0x58 reads 0x20 after F-U1 returns, the explicit write is 0x24. No other descriptor bytes are specified.

10. **Late arithmetic and state:** Input +0x70 equal to 7 yields −7; −1 yields 1. The minimum signed value remains unresolved. A zero state-query result skips both F-8635951b0989 calls; nonzero orders main/zero before intermediate/one.

## Uncertainties

The following remain documented_only:

- F-U1’s declaration, memory-access behavior, return behavior, extents, and preservation properties.
- Template contents, unassigned descriptor regions, and early local-area initialization.
- The intermediate F-U1 invocation’s missing third argument, if any.
- Full descriptor schemas and accompanying calling conventions.
- Application meanings of objects, flags, selectors, and dispatch methods.
- Dependency mutation or retention of the addressed slot-index cell.
- Selection, gate, insertion, chain, and cleanup semantics.
- Dependency ownership, release, failure, and nonlocal-exit behavior.
- Global mutability, aliasing, synchronization, and concurrent interference.
- Validity of narrowed collection indices and minimum-signed-value negation.

## Evidence

The supplied static representation and opaque dependency mapping establish the described widths, addresses, explicit assignments, branch conditions, and call order. In particular, callback selection explicitly assigns a local 32-bit index and supplies its address as F-f30a0116f2cc’s third argument.

Allocation, copying, construction, and cleanup purposes are inferred intent only. Dependency bodies, verified external declarations, template contents, underlying instruction confirmation, and runtime traces were not supplied. No runtime tests have been performed.
