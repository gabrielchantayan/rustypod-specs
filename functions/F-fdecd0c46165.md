<!-- Trusted provenance: F-fdecd0c46165; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-fdecd0c46165

## Purpose

F-fdecd0c46165 conditionally invokes an indirect operation based on the first word of a caller-supplied record. If that word is nonzero, it invokes the operation and then increments a field accessed through a separate object address. On either normally completing branch, it clears the record’s first word and then its second word.

The returned value is zero on the zero branch and the indirect operation’s four-byte result on the nonzero branch. The operation’s result does not control the increment or the record clears.

The apparent intent is to consume or reset a small record while counting dispatched operations. This is inferred intent, not an established resource-management contract. Clearing the record does not establish successful processing, resource release, or ownership transfer.

Object and record storage are not checked for overlap. Consequently, record reads and clears can also access object fields. In particular, the counter need not retain its incremented value after the record clears.

## Interface

### Inputs and storage layout

The function takes two inputs:

| Input | Role |
|---|---|
| Object address | Base used to obtain an indirect target, a callback argument, and a counter field on the nonzero branch. |
| Record address | Address of two consecutive four-byte words; the first controls dispatch, and both are subsequently cleared. |

The supplied representation supports the following four-byte integer and scalar storage model:

| Base | Byte offset | Width | Use |
|---|---:|---:|---|
| Object | 4 | 4 bytes | First argument to the indirect operation |
| Object | 64 | Callable-address slot; exact width unresolved | Indirect operation target |
| Object | 80 | 4 bytes | Counter read and incremented after normal callback return |
| Record | 0 | 4 bytes | Initial discriminator, then cleared |
| Record | 4 | 4 bytes | Cleared after the first word |

The first record word and counter are represented as integer fields. The callback’s first argument and the function result are uninterpreted four-byte quantities.

The evidence supports a 32-bit interpretation of the displayed integer and scalar accesses. It does not independently establish the complete ABI, callable-address representation, endianness, or alignment requirements.

The two input addresses need not identify disjoint storage according to the visible body. Whether a particular overlap is valid under the original language and platform rules is unresolved.

### Preconditions

For normal completion on either branch:

- The first record word must be readable.
- Both record words must be writable when their respective clears occur.
- Address formation and memory access must satisfy the actual platform’s requirements.

For the nonzero branch:

- The callback target slot and four-byte argument field must be readable before invocation.
- The target must be callable with the observed argument and result structure.
- The counter must remain readable and writable after the callback returns.
- Record storage must remain writable through both final clears.

For the zero branch, no access is made **through the object argument**. The object address is not used to obtain fields or select a callback. This does not imply that object storage remains untouched: the record can overlap it, and the record accesses can then read or overwrite object fields.

There is no visible validation of null addresses, bounds, alignment, callable compatibility, storage lifetime, or overlap.

### Output

On normal completion, the function returns a four-byte value:

- Zero if the initial first record word is zero.
- The callback’s result if the initial first record word is nonzero.

The evidence does not identify the result as a status, address, resource identifier, or error code. All normal callback results follow the same subsequent increment-and-clear path.

## Behavior

### Initial decision

The first record word is read once for the branch decision and compared with zero. The second record word is not read directly by this function.

Any nonzero first-word value selects the callback branch. There is no positive-only condition or additional sentinel. A value interpreted as negative under the applicable integer representation also selects that branch.

The branch is not reconsidered after the callback. Callback changes to the first word therefore do not cancel the pending counter update or clears.

### Zero branch

If the initial first word is zero:

1. No callback is invoked.
2. No field access is performed through the object argument.
3. Zero is written to the first record word.
4. Zero is written to the second record word.
5. Zero is returned.

The initial record read and both record writes can access object storage if the record overlaps it. Thus, “no access through the object argument” is not a guarantee that object fields are unread or unchanged.

For example, if the record begins at object byte offset 76, its second word occupies the counter field at offset 80. An initially zero first word suppresses the callback and explicit counter increment, but the second record clear still writes zero to the counter location.

The first-word clear is present in the supplied representation even though the initial read established zero. This is a source-facing operation order, not proof of a separate physical memory transaction under every compilation or memory model. Volatile and device-memory semantics are not supplied.

### Nonzero branch

If the initial first word is nonzero, the function invokes the runtime-selected dependency identified here as F-indirect-01.

Its arguments, in order, are:

1. The four-byte value obtained at object byte offset 4.
2. The record address itself.
3. The integer literal 4.

The callback target is obtained from the slot at object byte offset 64. The relative evaluation order of the target lookup and the first-argument field read is not established by the supplied expression. Both supply the invocation.

The callback receives the original record address, not a copied record value. Its access to that storage, and any mutation or retention of the address, depend on its unknown contract.

The literal 4 is established. Its interpretation as a byte count, element count, operation selector, or other parameter is not established.

After the callback returns normally:

1. Its four-byte result is retained for the eventual return.
2. The counter at object byte offset 80 is read.
3. One is added to that value, and the result is written to the counter.
4. Zero is written to the first record word.
5. Zero is written to the second record word.
6. The retained callback result is returned.

The counter is read after the callback, not saved before it. If the callback changes the counter, the increment uses the value present at the subsequent read.

The callback result is not tested. A result classified as failure by an external convention does not suppress any of the subsequent operations.

### Counter arithmetic

The counter update is represented as integer addition by one followed by a four-byte store. Where the addition is representable under the applicable integer model, the stored result is the value read plus one.

The supplied evidence does not establish a portable boundary rule. It does not justify a promise of modulo-\(2^{32}\) wrapping, saturation, trapping, or defined signed overflow. Counter-boundary behavior remains **documented_only** pending authoritative arithmetic or instruction evidence.

### Overlapping storage

There is no visible overlap check or preservation copy. Subject to valid accesses and defined arithmetic, the ordered operations apply to the same storage when addresses overlap.

Two counter overlaps are particularly consequential:

- **First record word overlaps the counter:** on a nonzero branch, the counter is incremented and then cleared by the first record store.
- **Second record word overlaps the counter:** on a nonzero branch, the counter is incremented and then cleared by the second record store. On the zero branch, that same store clears the counter without any callback or explicit counter increment.

Accordingly, the incremented counter value can be an intermediate state rather than a final postcondition.

Record clears can likewise overwrite other object fields when their addresses coincide. The function does not preserve such fields. These observations describe operation ordering under supported aliasing; they do not establish that every possible overlapping typed layout is legal.

## Side effects

The direct visible memory effects are:

- A read of the first record word on entry.
- On the nonzero branch, reads supplying the callback target and first argument.
- After normal callback return, a counter read and write.
- On either normally completing branch, a first-word clear followed by a second-word clear.

These access descriptions identify how addresses are obtained, not exclusive ownership of the affected bytes. A record access may also be an object-field access when storage overlaps. In particular, the zero branch can modify the counter through a record clear.

The second record word is not directly read by this function. The callback may read or modify it.

No allocation, deallocation, explicit locking, logging, or named global access is visible in the supplied body. This places no restriction on the callback’s behavior. Its effects on globals, memory, synchronization, or resource ownership are unknown.

Passing the record address does not establish ownership transfer. The visible body neither copies the record for later use nor explicitly releases its storage. No conclusion about destruction of a referenced resource follows from writing zeros.

The callback must leave subsequently accessed storage usable for normal completion. If it invalidates the counter or record storage, no recovery path is established.

There is no visible rollback or guaranteed cleanup mechanism:

- If the callback does not return normally, the subsequent increment and clears are not established to occur.
- If a later access fails, earlier callback effects or completed writes may already have occurred.
- Fault delivery, exception propagation, and nonlocal control transfer are not specified.

No atomic increment or synchronization mechanism is established. Concurrent access requires an external contract; the displayed read-add-write must not be treated as an atomic operation.

## Dependencies

### F-indirect-01

F-indirect-01 is a document-local opaque identifier for the callback selected at runtime. It does not imply a single fixed implementation.

Established interface and timing:

| Property | Contract |
|---|---|
| Selection | Target obtained at object byte offset 64 |
| Invocation condition | Initial first record word is nonzero |
| First argument | Four-byte field obtained at object byte offset 4 |
| Second argument | Original record address |
| Third argument | Integer literal 4 |
| Result | Four-byte value retained for this function’s return |
| Subsequent behavior | Counter increment and ordered record clears after normal return |

Unknown dependency properties include its semantic purpose, accepted values, detailed calling convention, error conventions, memory effects, ownership rules, address retention, reentrancy, synchronization, and termination behavior.

This contract does not assume that F-indirect-01 preserves globals, leaves the object or record unchanged, preserves the counter, or owns or releases any memory.

## Acceptance examples

These are conditional predictions from static evidence, not executed tests. Unless otherwise stated, they assume valid accesses, compatible callback invocation, normal completion, no concurrent interference, and representable arithmetic. Overlap examples additionally require an aliasing arrangement supported by the actual implementation environment.

### Initially clear, disjoint record

Given a record disjoint from object storage, with first word zero and second word 27:

- No callback is invoked.
- No access is performed through the object argument.
- Neither record access overlaps an object field.
- Both record words finish as zero.
- The result is zero.

The nonzero second word does not select the callback branch.

### Initially clear record overlaps the counter

Given a record beginning at object byte offset 76, with its first word zero and the counter at offset 80 initially 12:

- No callback is invoked.
- No explicit counter read-add-write occurs.
- The first record clear writes zero at offset 76.
- The second record clear writes zero at offset 80.
- The counter’s final value is zero.
- The result is zero.

The counter changes through the record address despite the absence of an access through the object argument.

### Ordinary nonzero dispatch

Given a record disjoint from the counter field, first record word 9, counter 12, and a callback that returns a four-byte value \(R\) without changing relevant storage:

- The callback receives the field at object offset 4, the record address, and literal 4.
- The counter becomes 13 and is not overwritten by the record clears.
- Both record words finish as zero.
- The function returns \(R\), including when \(R\) is zero.

### Nonzero record’s second word overlaps the counter

Given a record beginning at object byte offset 76, first word 9, counter 12, and a callback that leaves relevant storage unchanged:

- The callback runs.
- The counter is read as 12 and written as 13.
- The first record word is cleared.
- The second record clear overwrites the counter with zero.
- The final counter value is zero, not 13.
- The callback result is returned.

### Callback changes the counter and record

Given a record disjoint from the counter field and a nonzero initial first word, suppose the callback sets the counter to 40 and writes nonzero values into both record words.

After normal completion:

- The post-callback counter read obtains 40.
- The counter’s final value is 41 because neither record clear overlaps it.
- Both callback-written record values are replaced with zero.
- The callback result is returned unchanged.

### Callback returns an externally defined failure value

If the callback returns normally with a value that an external convention calls failure, the function still performs the counter update and both clears. It returns that value without visible translation. Any final counter claim remains subject to overlap.

### Callback does not return normally

If the callback transfers control away without normal return, no subsequent counter update or record clear is guaranteed by the supplied sequence. No rollback of callback effects is established.

## Uncertainties

The following remain **documented_only**:

- Complete ABI, pointer width, callable-address width, endianness, and alignment rules.
- Original declarations and integer-overflow semantics.
- Supported object/record aliasing arrangements.
- Callback purpose, side effects, ownership rules, and normal-return guarantees.
- Meaning of the literal 4.
- Semantic meaning of either record word or the counter.
- Behavior under concurrent access, invalid storage, or callback-induced lifetime violations.
- Fault, exception, and external error conventions.
- Whether displayed stores correspond to distinct hardware transactions.

The inferred consume-and-clear purpose does not resolve these uncertainties.

## Evidence

Established static evidence consists of the supplied function representation: one zero/nonzero decision on the first record word, one conditional indirect invocation with three arguments, a post-invocation counter increment, two ordered record clears, and a four-byte result.

The overlap consequences follow from the displayed byte offsets and access order, conditional on supported aliasing. They establish why neither “object storage is untouched on the zero branch” nor “the incremented counter is always the final counter” is a valid unconditional claim.

No callback implementation, caller contract, authoritative ABI description, or runtime trace was supplied. No runtime tests have been performed. The acceptance examples are conditional static predictions, not observed outcomes.
