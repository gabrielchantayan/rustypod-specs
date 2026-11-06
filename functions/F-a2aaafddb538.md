<!-- Trusted provenance: F-a2aaafddb538; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a2aaafddb538

## Purpose

F-a2aaafddb538 conditionally replaces two adjacent bytes in caller-addressed storage, then invokes a final dependency with the original first and second arguments. Every visible normal return produces the 32-bit value 1.

The replacements are independent: each byte is written only when its corresponding predicate returns a nonzero value. A zero predicate result causes no explicit clearing or replacement write for that byte.

The function uses selectors from a shared record and context values from a separate shared location. The first predicate receives a separately read context value; later predicate and byte-producing calls receive an earlier saved context value. Equality between those two context reads is not established.

The apparent intent is to obtain two selected properties before a final operation. That interpretation is **inferred intent**, not an established meaning for the selectors, destination fields, or dependencies.

## Interface

### Arguments and result

The supplied representation exposes four 32-bit arguments:

| Argument | Established role |
|---|---|
| First | Original destination address-like value. Also initializes a mutable local slot and is passed unchanged to the final dependency. |
| Second | Zero/nonzero control for the conditional stages. Also initializes a mutable local slot and is passed unchanged to the final dependency. |
| Third | Opaque value initializing a mutable local slot whose address is supplied to F-9b9c1c7483f4. |
| Fourth | Opaque value initializing a mutable local slot whose address is supplied to F-9b9c1c7483f4 and F-8a5aadac202f. |

The first two arguments have 32-bit integer representations. The first is used for address calculations; the second is tested against zero. These uses do not establish complete semantic types, ownership, or validity constraints.

The return value is 32 bits. Normal completion returns exactly 1, regardless of either predicate outcome. No dependency result is propagated as a status.

### Accessed storage and widths

The function accesses:

| Storage | Width and use |
|---|---|
| Record global | A shared 32-bit record-address value, captured at entry and separately read for the first selector lookup. |
| Context global | A shared 32-bit value, read twice after the first construction call. |
| Record offset 12 | A 32-bit selector used in the first stage. |
| Record offset 16 | A 32-bit selector used in the second stage. |
| Destination offset 176 | A one-byte field conditionally written by the first stage. |
| Destination offset 177 | A one-byte field conditionally written by the second stage. |

Offsets are in bytes. These accesses establish field locations and widths, not complete record or destination layouts. No byte-order conversion or multi-byte assembly is shown.

Four distinct mutable 32-bit local slots initially contain the respective incoming arguments. Dependencies receive addresses of these slots, not addresses of the original argument storage. Their relative placement is not an interface guarantee.

Predicate results are 32-bit integers tested only for equality or inequality with zero. Negative values, if returned, satisfy the nonzero condition. Byte-producing results are one byte wide; no Boolean normalization is visible.

### Preconditions for normal completion

Every reached selector read must address readable four-byte storage. Every reached destination write must address writable one-byte storage. Accesses to the shared globals and all dependency-specific argument requirements must also be valid.

A zero second argument bypasses selector dereferences and the function’s explicit destination writes. It does not bypass the entry read of the record global or the final dependency, and therefore does not establish that an arbitrary first argument is safe.

There are no visible checks for null addresses, selector validity, handle validity, arithmetic overflow, or dependency-reported errors.

## Behavior

### Entry and conditional control

The function first reads the record global and saves that record address for later accesses. This occurs even when conditional processing will be skipped. Reading the address does not itself dereference a selector field.

It initializes the four local slots from the incoming arguments. The original second argument then determines control flow:

- Zero: skip both selector stages.
- Nonzero: execute the first stage, followed by the second stage.

The slots are initialized only once. Changes to the second-argument slot do not revise the original branch decision.

### First stage: predicate path

The first stage obtains its initial selector by reading the record global again and reading the 32-bit selector at offset 12 from the address obtained by that read. This access is expressed through the shared global, rather than through the entry-saved record address.

It invokes F-9b9c1c7483f4 with, in dependency-argument order:

1. The address of the fourth-argument slot.
2. The selector just read.
3. The address of the third-argument slot.

The result is an opaque 32-bit value.

After this dependency returns, the represented context-read sequence is:

1. Read the context global and save that value for later calls.
2. Read the context global separately as the context argument for the first F-8094c3d72052 invocation.
3. Invoke F-8094c3d72052 with that second context read and the opaque construction result.

No dependency call intervenes between the two context reads. Nevertheless, the supplied representation does not establish a stability guarantee that makes their values equal. The first predicate uses the second, directly read value—not the saved value.

The function retains the predicate result and calls F-8a5aadac202f with the address of the fourth-argument slot. This post-use call occurs for either predicate outcome and before the predicate-dependent branch.

### First stage: conditional byte production

If the retained predicate result is zero, the function performs no explicit write to destination offset 176.

If it is nonzero, the function:

1. Reads the 32-bit selector at offset 12 from the entry-saved record address.
2. Invokes F-9b9c1c7483f4 with the address of the second-argument slot, that selector, and the address of the first-argument slot.
3. Invokes F-cf946f91e683 with the saved context from the first of the two context reads and the new opaque result.
4. Writes the returned byte at offset 176 from the original first argument.
5. Invokes F-8a5aadac202f with the address of the second-argument slot.

The byte write precedes this post-use call.

The selector is read again; its earlier value is not reused. Both the record-address source and the record contents may therefore matter. The predicate-path selector comes through a fresh record-global read, while the byte-producing selector comes through the entry-saved record address.

### Second stage: predicate path

The second stage begins after the first stage’s reached calls and write have completed. It executes regardless of the first predicate’s outcome.

The function reads the 32-bit selector at offset 16 from the entry-saved record address. It invokes F-9b9c1c7483f4 with:

1. The address of the fourth-argument slot.
2. That selector.
3. The address of the third-argument slot.

It then invokes F-8094c3d72052 with the saved context and the returned opaque value. Unlike the first predicate, this predicate does not receive a fresh context-global read.

After retaining the predicate result, the function invokes F-8a5aadac202f with the address of the fourth-argument slot. This occurs before branching on the result, whether that result is zero or nonzero.

### Second stage: conditional byte production

If the second predicate result is zero, the function performs no explicit write to destination offset 177.

If it is nonzero, the function:

1. Rereads the 32-bit selector at offset 16 from the entry-saved record address.
2. Invokes F-9b9c1c7483f4 with the address of the second-argument slot, the reread selector, and the address of the first-argument slot.
3. Invokes F-cf946f91e683 with the saved context and the new opaque result.
4. Writes the returned byte at offset 177 from the original first argument.
5. Invokes F-8a5aadac202f with the address of the second-argument slot.

This byte write also precedes its post-use call. The second-stage selector is reread on success rather than carried forward from the predicate path.

### Persistence and address selection

The local slots are not visibly reset between uses. Consequently:

- The second predicate-path construction receives the current third- and fourth-argument slots, including any earlier mutations.
- If both predicates succeed, the second byte-producing construction receives the current first- and second-argument slots, including mutations from the first byte-producing path.
- A changed first-argument slot does not redirect either explicit destination write. Both use the original first argument.
- A changed second-argument slot does not alter the final dependency’s arguments.
- Later context-global changes do not cause a fresh context read for the second predicate or either byte-producing call.

The function visibly reuses the saved record and context values. This does not establish broader preservation guarantees for memory or globals across opaque dependencies.

### Final operation and result

After both stages, or immediately after bypassing them, the function invokes F-876f4b90d114 with the original first and second arguments.

That dependency’s result is ignored. If execution returns normally from it, the function returns the 32-bit value 1.

There is no visible rollback, alternate status result, or error-recovery branch.

## Side effects

The function’s explicit writes outside its local storage are the independently conditional one-byte writes at destination offsets 176 and 177.

When both occur, the offset-176 write precedes the offset-177 write. The first write is not deferred until the second predicate succeeds.

These are explicit-write guarantees, not guarantees about final memory contents. Dependencies may affect the destination, records, globals, or other storage. In particular, the final dependency runs after the conditional writes and may change their observable results.

The addresses supplied to F-9b9c1c7483f4 and F-8a5aadac202f expose mutable local slots. Slot preservation is not established. Nor is it established whether dependencies retain those addresses; the supplied function provides no externally usable lifetime guarantee for them beyond this invocation.

No direct allocation or deallocation operation is shown. F-8a5aadac202f follows each completed construction path on normal flow, but its position does not prove destruction, resource release, or ownership consumption.

If a dependency does not return normally, later operations are not shown to execute. In particular:

- The corresponding post-use call is not guaranteed after abnormal exit from construction or consumption.
- A completed destination write has no visible rollback.
- The final dependency and constant return are not guaranteed after an earlier abnormal exit.

## Dependencies

| Opaque ID | Established interface and use | Unresolved semantics |
|---|---|---|
| F-9b9c1c7483f4 | Receives two local-slot addresses surrounding a 32-bit selector argument; returns an opaque 32-bit value. | Construction meaning, mutation, allocation, ownership, retained addresses, and failure encoding. |
| F-8094c3d72052 | Receives a context value and an opaque result; returns a 32-bit zero/nonzero predicate. Its first invocation uses the second context-global read. Its second invocation uses the saved first read. | Meaning of the predicate, side effects, and validity requirements. |
| F-cf946f91e683 | Receives the saved first context read and an opaque result; returns one byte for an explicit destination write. | Meaning and permitted range of that byte beyond its width, and other effects. |
| F-8a5aadac202f | Receives the address used as the first addressed argument of the corresponding F-9b9c1c7483f4 call. | Cleanup, release, ownership consumption, mutation, and idempotence. |
| F-876f4b90d114 | Receives the original first and second arguments after conditional processing; its result is ignored. | Final-operation meaning, validation, ownership, and resulting state changes. |

On normal completion with a nonzero second argument, the function makes:

- Exactly two predicate calls.
- Two to four construction calls.
- One corresponding post-use call for each construction.
- Zero to two byte-producing calls.
- Exactly one final-operation call.

With a zero second argument, only the final dependency is called.

No dependency is assumed to preserve globals, preserve destination memory, allocate storage, release storage, or transfer ownership.

## Acceptance examples

These are conditional criteria derived from static evidence. They have not been executed.

### Conditional processing disabled

Given a zero second argument and a normally returning final dependency:

- The entry record-global read still occurs.
- Neither selector field is dereferenced by these stages.
- Neither context-global read occurs.
- No explicit destination-byte write occurs.
- F-876f4b90d114 receives the original first argument and zero.
- Normal completion returns 1.

### Both predicates fail

Given a nonzero second argument and two zero predicate results:

- The first predicate receives the separately read second context value.
- The second predicate receives the saved first context value.
- Both predicate paths receive their post-use calls.
- No byte-producing call or explicit destination-byte write occurs.
- The final dependency still executes; normal completion returns 1.

### Only the first predicate succeeds

Given a nonzero first predicate, a zero second predicate, and a first byte-producing result of hexadecimal A6:

- The first byte-producing call receives the saved context, even if the first predicate received a different context value.
- A6 is explicitly written at destination offset 176 before that path’s post-use call.
- No explicit write occurs at offset 177.
- The second predicate stage and final dependency still execute.

### Only the second predicate succeeds

Given a zero first predicate, a nonzero second predicate, and a second byte-producing result of hexadecimal 03:

- No explicit write occurs at offset 176.
- The second predicate and its byte-producing call both receive the saved context.
- The function explicitly writes 03 at offset 177 before the corresponding post-use call.
- The byte is not normalized to 1.

### Different values at the two context reads

Suppose the first context-global read yields one value and the immediately following read yields another, under an execution environment that permits such a difference.

The first predicate receives the latter value. The second predicate and every reached byte-producing call receive the former, saved value. No dependency invocation occurs between those two reads.

If the context global remains stable across the two reads, their values coincide. That equality is conditional on stability, not an unconditional property of this function.

### Changes during dependencies

If the first construction changes the context global before returning, the first context capture reads the value present afterward. The first predicate still uses the subsequent separate read.

If a later dependency changes the context global, the function does not reload it for later context-bearing calls.

If a dependency changes a selector in the saved record, a later reached reread uses the selector then present. If it changes the record global, later selector accesses through the saved record address are not redirected merely by that global change.

### Mutated argument slots

If a dependency changes the first-argument slot, a later construction receiving that slot’s address encounters its current contents. The explicit byte writes and final dependency still use the original first argument.

These criteria do not imply that opaque dependencies leave the destination unchanged afterward.

## Uncertainties

The following remain **documented_only**:

- Semantic types, ownership rules, and validity domains of the arguments.
- Meanings and lifecycles of the record, context, selectors, and destination fields.
- Dependency side effects, resource behavior, and failure conventions.
- Whether dependencies retain local-slot addresses.
- Whether opaque 32-bit results represent addresses, handles, immediate values, or temporary objects.
- Stability of shared storage, including between the two consecutive context reads.
- Concurrency, synchronization, and hardware-level visibility guarantees.
- Behavior for invalid storage, abnormal exits, and address-space-boundary arithmetic.

The representation uses 32-bit address-like operands and fixed byte offsets. It shows no checked arithmetic and does not establish a portable overflow rule. The contract therefore requires valid reached addresses rather than asserting wraparound, saturation, or defined behavior for overflow.

The distinct context reads are established by the supplied representation. Actual machine-load behavior and the mechanism by which their values could differ are not established.

## Evidence

This contract is based on the supplied static function representation and opaque dependency mapping. The prior draft and reviewer feedback are not independent behavioral evidence.

Established static evidence includes:

- Four 32-bit inputs and a constant 32-bit normal result.
- Entry capture of the record global and a separate record-global read for the first selector.
- Two consecutive context-global reads after the first construction, with no dependency call between them.
- Use of the second context read for the first predicate.
- Use of the saved first context read for the second predicate and both possible byte-producing calls.
- Selector and destination widths, offsets, rereads, call ordering, local-slot reuse, and original-argument final dispatch.

Property extraction, construction, cleanup, and finalization are descriptive or inferred interpretations only; their semantic meanings are not established.

No dependency implementations, runtime traces, executable observations, or runtime tests were supplied or performed.
