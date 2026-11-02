<!-- Trusted provenance: F-3c50d3f1f4fd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3c50d3f1f4fd

## Purpose

F-3c50d3f1f4fd conditionally obtains two object references, records each reference in a caller-supplied record, and invokes an indirect operation on each object. The first invocation receives the integer value 1; the second receives 0.

A single byte in the supplied record gates the entire operation. If that byte is zero, the function returns without obtaining either object or writing either reference field.

The established behavior is a sequence of reads, calls, and writes. Descriptions such as “activate the first object” or “deactivate the second object” are inferred intent only: the meanings of the indirect operation and its integer argument are not established.

## Interface

**Input:** one address-like argument identifying a writable record, called the **subject** below.

**Output:** no explicit result value is established. Results are observable through the subject’s reference fields and through effects of the invoked dependencies.

The source representation uses an integer-valued argument for the subject address and integer-pointer representations for the returned objects. Its apparent model is a 32-bit address space with 32-bit object references and table entries. This is the layout represented by the supplied material, not an independently verified ABI.

The subject has these relevant locations:

| Byte offset from subject | Represented width | Role |
|---|---:|---|
| +0x5A | 1 byte | Execution gate |
| +0x7C | 4 bytes under the represented 32-bit layout | First returned object reference |
| +0x80 | 4 bytes under the represented 32-bit layout | Second returned object reference |

The gate is tested for equality to zero. Every nonzero byte value selects the enabled path; no particular nonzero value is required. Signed versus unsigned interpretation does not affect this zero/nonzero decision.

Each returned object is used through this apparent layout:

| Location | Represented width | Role |
|---|---:|---|
| Object +0x00 | 4 bytes | Address of a dispatch table |
| Dispatch table +0xA0 | 4 bytes | Address of an indirectly invoked operation |

The indirect operation receives the returned object reference as its first argument. Its second argument is the integer constant 1 for the first object and 0 for the second. The supplied material represents these as integer arguments; a narrower Boolean parameter type is not established.

For defined completion, the following access requirements apply:

- The subject’s gate byte must be readable.
- On the enabled path, both subject reference fields must be writable.
- Each returned object must support the reads needed to obtain its dispatch target.
- Each indirect target must be callable with the supplied object reference and integer argument.
- Address calculations and accesses must be valid in the actual execution environment.

No validation of these conditions is shown. The disabled path does not access the reference fields or returned-object layouts.

## Behavior

### Gate decision

The first established data access is a one-byte read at subject offset +0x5A.

If the byte is zero, the function returns immediately. It does not directly write either reference field, read either dependency-selector value, or invoke a dependency.

If the byte is nonzero, execution proceeds through the two object-processing stages. There is no second gate check. A dependency changing the gate after the initial decision does not, in the shown control flow, prevent the second stage.

### First object-processing stage

The function reads the value held in a first global selector location and passes that value to F-cb982c8fd0d4. The selector’s value and meaning are not provided. It is an input to the dependency, not an established object address, name, or ownership token.

After F-cb982c8fd0d4 returns, its result is written to the subject’s field at +0x7C.

Only after that write does the function resolve an indirect operation through the returned object:

- It obtains the dispatch-table address from the object’s initial word.
- It obtains the operation address from byte offset +0xA0 in that table.
- It invokes that operation with the returned object reference and integer value 1.

The returned reference is therefore published into the subject before this indirect invocation begins. If the invoked operation can independently reach the subject, it can observe the preceding write, subject to the execution environment’s memory model. No synchronization or cross-thread visibility guarantee is established.

The object used for dispatch is the result returned by F-cb982c8fd0d4. The supplied behavior does not reload the object reference from the subject’s +0x7C field to choose the dispatch receiver.

No null check, status check, or type check intervenes between receiving the result and using it for dispatch.

### Second object-processing stage

After the first indirect invocation returns, the function reads the value held in a second global selector location and passes that value to F-cb982c8fd0d4.

This read occurs after the first dependency call and first indirect invocation. It must not be treated as a selector value captured at entry. Either earlier callee could affect the second selector if it has access to that state; no preservation guarantee is available.

After the second dependency call returns, the function writes its result to the subject’s field at +0x80.

It then resolves the indirect operation using that returned object’s initial word and the dispatch-table entry at byte offset +0xA0. The operation receives the second returned object reference and integer value 0.

The two stages use the same dispatch-table offset, but they need not select the same executable target. Their returned objects and table contents may differ.

The supplied representation ends with this second indirect operation and a return. Recovery of the terminal indirect transfer is explicitly uncertain. The representation supports the object-reference publication followed by a two-argument indirect invocation, but does not independently establish whether the machine-level transfer is an ordinary call or a tail transfer.

### Ordering, aliasing, and partial completion

The established order on the enabled path is:

1. Read the first selector and call F-cb982c8fd0d4.
2. Store the first result at subject +0x7C.
3. Resolve and invoke the first indirect operation with argument 1.
4. Read the second selector and call F-cb982c8fd0d4.
5. Store the second result at subject +0x80.
6. Resolve and invoke the second indirect operation with argument 0.

This order is significant if records, objects, tables, or selector storage alias one another. For example, publication into a subject field precedes the corresponding dispatch-table reads; an overlapping write can therefore affect those later reads. The supplied material provides no non-aliasing guarantee.

There is also no guarantee that the two returned references differ. If both dependency calls return the same object, the function still performs both field writes and both dispatch resolutions, with argument 1 first and 0 second. The first invocation may change the object’s dispatch table before the second resolution.

No rollback is shown. If an invocation does not return, or execution fails during a later access, writes and dependency effects that already occurred are not explicitly undone.

## Side effects

The function’s own established writes are limited to the two subject reference fields on the enabled path:

- +0x7C receives the first dependency result.
- +0x80 receives the second dependency result.

Existing field contents are overwritten without being read, compared, or explicitly released.

The function does not directly modify the gate byte. This does not imply that the gate remains unchanged: called operations may modify it or other accessible state.

Ownership is **documented_only**. Storing a reference does not establish whether it is borrowed, retained, newly allocated, shared, or transferred. No explicit allocation, release, or reference-count operation appears in this function, but any called operation may perform such work.

The dependency calls and indirect operations may have additional effects. Nothing supplied establishes that they preserve globals, preserve subject fields, avoid reentrancy, leave dispatch tables unchanged, or return normally.

Consequently, the writes establish what this function publishes at each stage, not unconditional final field values. A later callee may overwrite either field.

No locking, atomic publication protocol, explicit error reporting, or exception handling is shown.

## Dependencies

### F-cb982c8fd0d4

Called twice on the enabled path, once with each selector value.

Its return is consumed as an object reference without validation. The function depends on that reference supporting dispatch-table resolution.

The following dependency properties are **documented_only**:

- Meaning and width of the selector parameter beyond its represented machine-word use.
- Selector values and whether the two values differ.
- Allocation, lookup, caching, or other production mechanism.
- Nullability and failure conventions.
- Ownership and lifetime of returned references.
- Global preservation, reentrancy, and normal-return guarantees.

The second call occurs only after the first indirect invocation returns.

### Dynamically resolved operations

Both operations are selected through dispatch-table byte offset +0xA0. No opaque function ID for either target is supplied, so neither target is equated with a known dependency.

The established invocation contract is the object reference followed by the integer constant 1 or 0. The semantic meaning of those constants is unknown. Their use is compatible with a binary state-setting operation, but that interpretation is not required by the static evidence.

No return value from either indirect operation is explicitly consumed in the supplied representation. The uncertain terminal transfer prevents a stronger statement about incidental machine-register values at exit.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not reported runtime tests.

### Zero gate

Given a readable subject gate containing zero, the function returns without directly reading the selectors, calling F-cb982c8fd0d4, or writing either reference field.

This criterion does not require the two output-field locations to be accessed or validated.

### Ordinary enabled execution

Given a nonzero gate, valid writable reference fields, valid returned objects, and normally returning dependencies:

- F-cb982c8fd0d4 receives the first selector value.
- Its first result is stored at +0x7C before the first indirect invocation.
- The first indirect invocation receives that result and 1.
- F-cb982c8fd0d4 then receives the second selector value.
- Its second result is stored at +0x80 before the second indirect invocation.
- The second indirect invocation receives that result and 0.

If the callees do not subsequently alter the reference fields, the fields retain the corresponding results.

### Arbitrary nonzero gate

Given a gate byte of 0x80 or 0xFF, the enabled path is selected just as it is for 0x01. No Boolean normalization write occurs.

### Selector changed by an earlier invocation

Given that the first indirect invocation changes the second selector location and returns, the second dependency call uses the value read afterward, not necessarily its entry-time value.

### Same object returned twice

Given that both calls return the same valid object, both subject fields receive that reference. The indirect operation is resolved twice and invoked with 1, then 0. Neither stage is eliminated because the references match.

### Invalid first result

Given that the first call returns an unusable object reference, that reference is stored at +0x7C before dispatch resolution attempts to use it. A graceful error return is not an acceptance requirement: no such handling is established. The precise failure outcome is unknown.

### First indirect invocation does not return

Given that the first indirect invocation does not return, the second selector read, second dependency call, and second field write are not reached through the shown sequence. The first field write has already occurred.

## Uncertainties

The following remain **documented_only**, rather than implementation requirements inferred from naming or convention:

- Exact ABI, pointer width, alignment rules, and calling conventions beyond the apparent 32-bit representation.
- Full subject layout and the semantic purpose of its gate and reference fields.
- Selector contents and meanings.
- Concrete identities and contracts of the indirect targets.
- Ownership, allocation, destruction, and reference-lifetime rules.
- Whether aliasing or concurrent access is permitted.
- Behavior on invalid addresses, unusable results, dependency failures, or nonlocal exits.
- Whether callees mutate globals, subject fields, object layouts, or dispatch tables.
- Exact machine-level form of the terminal indirect transfer.

The apparent binary-control intent is an inference. The established requirement is narrower: invoke the selected operations in order with integer values 1 and 0.

## Evidence

The supplied static representation establishes the initial byte gate, two sequential calls to F-cb982c8fd0d4, publication of their results at subject offsets +0x7C and +0x80, and indirect operations selected through table offset +0xA0 with integer arguments 1 and 0.

It also contains an explicit recovery limitation for the terminal indirect transfer. That limitation is retained rather than converted into a stronger control-flow claim.

No dependency bodies, runtime traces, ABI documentation, selector values, or ownership contracts were supplied. No runtime tests have been performed.
