<!-- Trusted provenance: F-1d2bdbbd2407; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1d2bdbbd2407

## Purpose

F-1d2bdbbd2407 obtains an address from a dependency, initializes two 32-bit fields relative to that address, invokes further setup and indirect-dispatch operations, and returns the originally obtained address.

The established contract is an ordered sequence of calls and memory accesses. Object construction and registration under the label “TRadioCntlr” are inferred purposes, not established type or lifecycle guarantees.

The function performs no visible validity checks, branches on no dependency result, and exposes no separate error result. Normal completion requires that its unchecked memory accesses and dependency invocations succeed.

## Interface

### Inputs

The function accepts two positional 32-bit values.

Both are forwarded unchanged to F-3c6991d61386. The function does not directly interpret or dereference either input. Their signedness, semantic meaning, valid ranges, and possible roles as addresses or handles are unknown.

### Output

On normal completion, the function returns the same address obtained from F-3c6991d61386.

The supplied representation treats that address as pointing to 32-bit integer storage. This supports the direct field accesses described below; it does not establish a complete object type.

Subsequent dependencies do not select a replacement return address. They may nevertheless affect the lifetime or contents of the storage at that address. Returned-address identity is established; continued validity is not.

### Directly accessed layouts

**Acquired storage**

| Byte offset from acquired address | Access width | Direct operation |
|---|---:|---|
| 0 | 32 bits | Write the second sample of the shared initialization word |
| 28 | 32 bits | Write the first sample increased by 220 |

The offset-28 field is seven 32-bit elements from the acquired address. These accesses require writable regions covering bytes 0–3 and 28–31 relative to that address. They do not establish a complete 32-byte object layout, the meanings of intervening bytes, or an allocation size.

**Dispatch receiver and intermediate storage**

| Location | Access | Meaning |
|---|---|---|
| Start of the provider result | Read one 32-bit word | Interpret that word as the address of callable-pointer storage |
| Storage addressed by that word | Read a callable pointer | Obtain the target for the indirect invocation |

These are two distinct memory reads. The first has an established width of 32 bits. The width of the subsequent callable-pointer read is not independently established.

The first word is not directly used as the callable target.

### Data roles

This contract uses the following descriptive roles:

- **Acquired address:** the result of F-3c6991d61386.
- **Shared initialization word:** the shared 32-bit source sampled for both direct writes.
- **Provider result:** the address returned by F-U01 and passed as the indirect call’s receiver.
- **Dispatch-storage word:** the 32-bit word read from the provider result and interpreted as an address.
- **Registration value A** and **registration value B:** the two shared 32-bit values passed as the third and fourth arguments of the indirect call, respectively.
- **Label reference:** the reference to data identified as “TRadioCntlr.”

These names identify roles, not additional interfaces or proven types.

### Preconditions for normal completion

The body does not check these requirements:

- F-3c6991d61386 must return an address supporting the two specified writes.
- The shared initialization and registration sources must be readable when accessed.
- F-U01 must return an address supporting the initial 32-bit receiver-word read.
- Interpreting that word as an address must produce a location supporting a callable-pointer read.
- The resulting callable pointer must be valid for invocation with the supplied arguments.
- Each dependency must accept its arguments and return normally.

Any alignment, address-representation, calling-convention, or lifetime requirements needed to satisfy these operations remain external prerequisites.

## Behavior

### 1. Obtain the acquired address

The function first calls F-3c6991d61386 with four arguments, in these positions:

1. First 32-bit input.
2. Second 32-bit input.
3. Constant 1.
4. Constant 0.

The returned address is retained for both direct field writes, the subsequent call to F-1855137c994c, and the eventual return.

No null check, bounds check, or alternative failure path is visible before that address is used.

The dependency’s role is unknown beyond its supplied interaction. It may allocate storage, return existing storage, or have another behavior. The body does not establish any such choice.

### 2. Sample the shared initialization word and write two fields

After acquisition returns, the function reads the shared initialization word and retains that first 32-bit sample.

It then reads the same shared source again. The second sample supplies the value written at offset 0 of the acquired storage.

After that write, the function writes the first sample plus hexadecimal DC, decimal 220, at offset 28.

The established source-level order is therefore:

- First initialization sample.
- Second initialization sample.
- Offset-0 write using the second sample.
- Offset-28 write using the first sample increased by 220.

No dependency call intervenes between the initialization samples or between the two direct writes.

The two samples must not be conflated in the contract. If an execution model permits different observations, the offset-0 value follows the later sample, while the offset-28 value follows the earlier one.

The supplied representation does not establish volatility, atomicity, synchronization, or a hardware-level load count. The ordered accesses described here are static evidence, not a guarantee about compiler transformations or concurrent execution.

The addition is represented using a 32-bit integer value. Its result is established where increasing the signed 32-bit interpretation by 220 is representable. For values outside that range, overflow behavior is **documented_only**. The evidence does not establish a portable wrapping rule, a trap, or another specific outcome.

### 3. Invoke secondary setup

After both direct writes, the function calls F-1855137c994c with:

1. The acquired address.
2. Hexadecimal 5800, decimal 22528.

The meaning of 22528 is unknown. It is not established as a byte count, flag set, object identifier, or allocation request.

No result from this call controls subsequent execution or replaces the retained return address. Its effects on the acquired storage, shared sources, or memory ownership are unknown.

### 4. Obtain the receiver and resolve the indirect target

After F-1855137c994c returns, the function calls F-U01 without explicit arguments.

F-U01 is a contract-local opaque identifier for the provider dependency whose mapped identity was not supplied.

The provider result participates in a two-stage lookup:

1. Read a 32-bit word from the start of the provider result.
2. Interpret that word as the address of storage containing a callable pointer, then read the callable pointer from that storage.

The callable pointer obtained by the second read determines the indirect target, designated F-U02.

The intermediate address is essential to the contract. The provider’s first word does not itself supply the final callable pointer, and the representation does not justify collapsing the two reads into one.

The structure resembles dispatch through an intermediate table or slot. Its type, the existence of additional entries, and any broader table layout are inferred or unknown. Only the two accesses above are established.

There are no visible checks on the provider result, intermediate address, or callable pointer.

### 5. Invoke the resolved target

F-U02 receives four arguments in this order:

1. The provider result itself.
2. The label reference associated with “TRadioCntlr.”
3. Registration value A.
4. Registration value B.

The receiver argument remains the provider result. It is not replaced by the intermediate callable-pointer storage address.

The registration sources are read for this call after the preceding explicit dependencies have returned. They are not snapshots taken at entry. F-3c6991d61386, F-1855137c994c, and F-U01 may have changed shared state before these reads.

The supplied call expression does not establish the relative evaluation order between the dispatch-resolution reads and the registration-source reads. It establishes the two-read lookup dependency, the argument positions, and completion of argument and target evaluation before invocation.

The label suggests identification or registration intent. Its encoding, termination, storage extent, retention rules, and downstream meaning are not established.

The indirect call’s result is not visibly used.

### 6. Invoke the final dependency and return

After F-U02 returns normally, the function calls F-93eb550dc612 without explicit arguments.

After that call returns normally, the function returns the original acquired address.

The function performs no further direct writes to the acquired storage after the offset-28 write. This does not guarantee final field contents: any later dependency may mutate or release that storage.

## Side effects

### Directly established effects

The function performs:

- Two reads of the shared initialization word in the supplied representation.
- A 32-bit write at offset 0, followed by a 32-bit write at offset 28.
- Reads of the two shared registration values.
- A 32-bit word read from the provider result.
- A separate callable-pointer read through the address represented by that word.
- Five ordered invocation stages: acquisition, secondary setup, provider invocation, resolved indirect invocation, and the final dependency call.

The two dispatch reads have different established width information: the receiver-word read is 32-bit; the callable-pointer read has an unspecified pointer width.

### Delegated effects

Dependencies may modify shared state, mutate the acquired storage, allocate or release memory, retain references, synchronize, or perform external operations. None of those effects is established or excluded by the supplied body.

No dependency is assumed to preserve globals, preserve either dispatch location, or preserve the validity of the acquired address.

### Ownership and lifetime

The function establishes no allocation ownership, transfer rule, reference-counting obligation, or release requirement.

There is no explicit release operation in the visible body. That does not prove that dependencies refrain from releasing memory.

The provider result and intermediate callable-pointer storage are not returned. Their owners, required lifetimes, and possible retention by F-U02 are unknown.

### Errors and interrupted execution

No conditional error handling, retry, rollback, or cleanup branch is visible.

If any memory access fails, or a dependency throws, exits nonlocally, fails to return, or otherwise interrupts execution, subsequent operations are not guaranteed to occur. Earlier writes and dependency effects are not visibly undone.

A non-null return, a live returned allocation, and clean failure on invalid addresses are not established guarantees.

## Dependencies

| Opaque ID | Established interaction | Unknown contract |
|---|---|---|
| F-3c6991d61386 | Receives both 32-bit inputs, then 1 and 0; returns the acquired address | Allocation, failure representation, ownership, initialization, shared-state effects |
| F-1855137c994c | Receives the acquired address and 22528 after both direct writes | Meaning of the constant, storage mutations, ownership and lifetime effects |
| F-U01 | Called without explicit arguments; returns the receiver address | Identity, validity guarantees, ownership, shared-state effects |
| F-U02 | Target obtained by the two-stage lookup; receives receiver, label reference, registration value A, registration value B | Runtime identity, argument interpretation, retention, mutation, failure behavior |
| F-93eb550dc612 | Called without explicit arguments after F-U02 returns | Purpose, cleanup, synchronization, ownership effects |

F-U01 and F-U02 are contract-local opaque identifiers, not externally verified identity mappings.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence. They are not executed tests.

### Stable initialization samples

If both initialization reads produce 100, the acquired storage supports the writes, and all subsequent operations complete normally:

- The direct offset-0 write stores 100.
- The direct offset-28 write stores 320.
- F-1855137c994c is called after both writes.
- The returned address equals the acquisition result.

Final field contents equal those written only if subsequent dependencies leave the fields unchanged.

### Distinct initialization samples

If the first initialization sample is 100 and the second is 200:

- The offset-0 write stores 200.
- The offset-28 write stores 320.

This criterion is conditional on an environment permitting distinct observations. It does not establish concurrent-access safety.

### Intermediate dispatch storage

Suppose the provider returns a valid receiver, the receiver’s first 32-bit word represents address Q, and a callable-pointer read at Q yields target T.

The required behavior is to invoke T, passing the provider result as the first argument. Invoking Q directly would not satisfy the established lookup behavior.

This example assumes that the word-to-address interpretation and callable-pointer representation are valid in the execution environment.

### Invalid intermediate storage

If the receiver supports its 32-bit read but the resulting word does not designate readable callable-pointer storage, the second lookup cannot be assumed to succeed.

The function contains no visible validation or fallback for this case. The precise failure outcome is **documented_only**.

### Registration values changed by earlier dependencies

If F-1855137c994c or F-U01 changes a registration source before that source is evaluated, F-U02 receives the subsequently read value in its established argument position.

An implementation that unconditionally snapshots these sources at function entry would not preserve this behavior.

### Indirect-call interruption

If F-U02 does not return normally, neither invocation of F-93eb550dc612 nor normal return from F-1d2bdbbd2407 is established. Earlier effects may remain.

### Arithmetic boundary

For a first sample interpreted as 2,147,483,427, adding 220 produces 2,147,483,647, within signed 32-bit range.

For a first sample interpreted as 2,147,483,428, the addition exceeds that range. No particular wrapped value or failure mechanism is required without further arithmetic evidence.

## Uncertainties

The following remain **documented_only**:

- Machine ABI, general pointer width, and callable-pointer width.
- How the 32-bit dispatch-storage word is converted to an address, including any representation or addressability constraints.
- Input semantics and valid domains.
- Acquired-storage type, complete layout, alignment, field meanings, and lifetime.
- Meaning of the shared initialization word and the constant 22528.
- Overflow behavior outside the representable signed-addition range.
- Volatility, atomicity, synchronization, aliasing, and concurrent-access rules.
- Relative evaluation order among dispatch-resolution and argument-source reads within the indirect call expression.
- Label representation, lifetime, and interpretation.
- Dependency failure, ownership, retention, and state-preservation contracts.
- Whether the returned address still identifies live storage.

Construction, registration, and finalization are inferred roles only. The established 32-bit receiver-word read does not establish that all pointers are 32-bit.

## Evidence

Established static evidence consists of the supplied function representation and dependency identity map. It supports the input widths, mapped dependency interactions, constants, two initialization samples, ordered field writes, registration argument positions, and retained return-address identity.

For indirect dispatch, the evidence establishes a 32-bit word read from the provider result, interpretation of that word as the address of callable-pointer storage, and a subsequent callable-pointer read before invocation. It does not establish a direct callable-target read from the receiver or an independently known callable-pointer width.

Object-construction and registration intent are inferred from the operation sequence and label. No dependency bodies, ABI documentation, ownership contracts, or runtime observations were supplied.

No runtime tests have been performed. Acceptance examples are conditional obligations derived from static evidence, not observed results.
