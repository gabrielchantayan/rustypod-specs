<!-- Trusted provenance: F-e87e2dd9e3e1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e87e2dd9e3e1

## Purpose

The available static representation describes a flag-dependent sequence of memory accesses, an optional coprocessor-related operation, and a terminal region whose instruction data could not be recovered reliably.

No application-level purpose is established. In particular, the evidence does not establish that this function implements a valid copy operation, a device transaction, or an intentional trap. Its strongest supported contract is the ordering and conditional selection of the operations visible before the recovery boundary.

This contract distinguishes:

- **Established static evidence:** operations and dependencies explicitly represented in the supplied material.
- **Inferred intent or machine model:** interpretations that require assumptions about address width, instruction semantics, or the correctness of recovered control flow.
- **documented_only unknowns:** behavior that cannot be established from the supplied material.

There is no established normal-return path. The terminal representation must not be treated as proof of a deliberately implemented halt.

## Interface

### Explicit inputs

The recovered interface has four argument positions:

| Position | Represented width | Visible use |
|---|---:|---|
| First | 32 bits | Anchor for reads at negative byte offsets. |
| Second | 32 bits | No visible use. |
| Third | 32 bits | No visible use. |
| Fourth | 32 bits | Basis for the address supplied to the optional coprocessor-related operation. |

The first and fourth arguments are represented as signed integers, but their visible uses are address calculations. This does not establish source-language signed-overflow behavior or a portable pointer model.

The second and third arguments have no visible effect within the recovered region. That is not a guarantee about unrecovered execution beyond its boundary.

### Implicit inputs

The visible behavior also depends on state not supplied through the four explicit argument positions:

- An entry byte value.
- An entry destination address for that byte.
- An entry 32-bit base value used for a later byte-store address.
- An entry unsigned 32-bit word used to derive that address.
- Entry overflow, zero, and negative condition indicators.
- An incoming coprocessor-related operand or selector.

These values are required to describe the represented behavior. An ordinary four-argument call does not, by itself, specify them.

The predicates in the recovered representation refer to the incoming condition indicators. No visible operation explicitly recomputes those indicators.

### Outputs

The recovered interface declares no return value. Observable outputs may instead include:

- A conditional one-byte store through the entry destination address.
- Conditional 32-bit memory reads relative to the first argument.
- An optional coprocessor-related operation.
- A conditional one-byte store through a computed address.
- An unresolved terminal instruction condition.

No output structure, status code, success indicator, or recoverable error result is established.

### Address and access preconditions

For execution to complete each represented memory access without a memory-access failure, the corresponding address must support that access:

- The entry destination must support a one-byte write when overflow is clear.
- The locations four, sixteen, and twenty bytes before the first argument must support 32-bit reads when zero is set.
- The final computed destination must support a one-byte write when negative is clear.

These are conditional access requirements, not visible validation rules. No bounds checking, null checking, alignment checking, or fault handling appears in the recovered region.

The access requirements of the coprocessor-related operation are **documented_only**.

## Behavior

### 1. Conditional initial byte store

If the incoming overflow indicator is clear, the function first writes the incoming byte to the incoming destination address.

If overflow is set, this store is omitted.

This store precedes every explicit read relative to the first argument. Consequently, aliasing matters: if its destination overlaps a later-read location, that later read observes memory after this store, subject to the actual memory system’s behavior.

The incoming destination is independent of the first and fourth explicit arguments in the supplied representation. No derivation connecting them is established.

### 2. Conditional first read group

If the incoming zero indicator is set, the following reads occur in this order:

1. A 32-bit word is read four bytes before the first argument. It becomes the word available for the later address calculation.
2. A 32-bit word is read sixteen bytes before the first argument. Only its low eight value bits become the selected byte for the later store.
3. A 32-bit value is read twenty bytes before the first argument. It becomes the base for the later store address.

The second access is represented as a full 32-bit read followed by narrowing to eight bits. It is not represented as a one-byte read. This distinction affects access validity, memory-mapped side effects, and possible faults.

“Low eight value bits” does not independently identify the lowest-addressed byte of the read location. That mapping depends on byte order, which is not established here.

If zero is clear, all three reads are omitted. The later computation, if reached, instead uses the corresponding incoming implicit values.

The read group is controlled by zero, not by whether the later byte store will occur. Thus, these reads remain present when zero is set and negative is also set, even though negative suppresses the final store.

### 3. Conditional coprocessor-related operation

If the incoming negative indicator is set, the representation performs a coprocessor-related long-load operation with:

- Coprocessor number 7.
- The incoming coprocessor-related operand or selector.
- An address expression equal to the fourth argument minus 332 bytes.

This operation follows the first read group and precedes the repeated read described below.

The operation’s label does not establish its memory transfer size, direction, architectural validity, privilege requirements, or device effects. Those properties are **documented_only**.

If negative is clear, this operation is omitted.

### 4. Conditional repeated word read

If the incoming zero indicator is set, the function again reads a 32-bit word four bytes before the first argument.

This second read replaces the earlier word for purposes of the later address calculation. It does not reload either the selected byte or the base value.

The two reads from this location must not be collapsed into one observable access. They are distinct accesses, separated by two other reads and, when negative is set, the coprocessor-related operation. Even if ordinary memory would normally return the same value twice, that equality is not guaranteed by the supplied evidence.

No preservation guarantee is supplied for the coprocessor-related operation. In particular, this contract does not assume that it leaves accessible memory or external state unchanged.

### 5. Conditional computed byte store

If the incoming negative indicator is clear, the function writes one byte to a computed address.

The selected byte is:

- The low eight bits of the word read sixteen bytes before the first argument, when zero is set.
- The incoming implicit byte otherwise.

The base is:

- The value read twenty bytes before the first argument, when zero is set.
- The incoming implicit base otherwise.

The displacement is formed from the selected unsigned 32-bit word by combining a logical right shift of 10 bits with a left shift of 22 bits. Under a 32-bit word interpretation, this is a rotation right by 10 bits.

The selected word is:

- The value from the **second** read four bytes before the first argument, when zero is set.
- The incoming implicit unsigned word otherwise.

The destination address is the selected base minus that rotated value.

The rotation’s 32-bit interpretation is supported by the represented unsigned word width. Address subtraction and address wrapping require separate qualification: a modulo-\(2^{32}\) address calculation is consistent with a 32-bit machine model, but the supplied representation does not establish portable behavior for overflowing signed expressions or invalid addresses.

If negative is set, this final byte store is omitted.

### 6. Terminal recovery boundary

After the conditional operations, the representation ends at an invalid-instruction recovery boundary.

The established fact is that no normal continuation or return is recovered. It is not established whether actual execution would trap, halt, enter an exception handler, execute differently decoded instructions, or continue into code absent from the supplied material.

Any preceding operation may also prevent arrival at this boundary by faulting or otherwise failing to complete.

## Side effects

### Memory and ordering

The visible ordinary-memory writes are each one byte wide. The visible ordinary-memory reads are each 32 bits wide.

Subject to the condition indicators, their order is:

1. Initial byte store.
2. First word read four bytes before the first argument.
3. Word read sixteen bytes before the first argument.
4. Base read twenty bytes before the first argument.
5. Coprocessor-related operation.
6. Second word read four bytes before the first argument.
7. Computed byte store.
8. Terminal recovery boundary.

This is the order in the static representation, not a claim about hardware bus transactions, atomicity, memory barriers, or concurrency synchronization.

There is no visible rollback. If an operation fails, the evidence supplies no mechanism for undoing earlier completed stores or external effects.

### State and ownership

The visible region does not explicitly allocate, release, transfer, or retain ownership of memory. It supplies no object-lifetime protocol.

That observation is limited to the visible operations. It is not a guarantee that the unresolved coprocessor-related operation has no resource or global-state effects.

The representation updates working values used by later operations. It does not establish an externally observable register-preservation contract or a calling convention for normal return.

### Errors

No explicit error checking or error reporting is visible. Possible access failures, unsupported coprocessor behavior, privilege failures, and the unresolved terminal instruction condition have no recovered handling path.

Their exact exception types and resulting machine state are **documented_only**.

## Dependencies

No independently identified dependency with a supplied opaque function ID is available.

Two represented operations require semantic support beyond ordinary reads, writes, and integer transformations:

- The coprocessor-related operation.
- The terminal invalid-instruction marker.

These are not established as calls to separately identifiable functions. Assigning them invented function identities would imply provenance that the evidence does not provide.

Their unresolved behavior remains part of this function’s uncertainty boundary. No assumption is made that either operation preserves globals, preserves memory, returns normally, or owns or releases resources.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Each assumes that the relevant operations complete and that the recovered control flow correctly represents execution.

### No zero-conditioned reads; computed store enabled

Given overflow set, zero clear, and negative clear:

- The initial byte store is omitted.
- All four zero-conditioned 32-bit reads are omitted.
- The coprocessor-related operation is omitted.
- One byte is written using the incoming implicit byte, base, and unsigned word.
- Execution then reaches the unresolved terminal boundary if that store completes.

If the incoming word is zero, its rotated value is zero, so the computed destination equals the incoming base.

### Initial store followed by fresh loaded values

Given overflow clear, zero set, and negative clear:

- The incoming byte is first stored through the incoming destination.
- The three-read group occurs in the specified order.
- The word four bytes before the first argument is read again.
- The final byte store uses the previously loaded base and byte, but the word from the second read.

If that second word is 1, its 32-bit rotation right by 10 is `0x00400000`. The final destination is the selected base minus that displacement, subject to the address-arithmetic qualification above.

### Coprocessor path with no computed store

Given zero set and negative set:

- The three-read group occurs.
- The coprocessor-related operation follows.
- The repeated word read occurs if the operation completes.
- No computed byte store occurs.

Overflow independently determines whether the initial byte store precedes this sequence.

### Full-width read before narrowing

Given zero set, if the word read sixteen bytes before the first argument produces the numeric value `0x123456AB`, the selected byte is `0xAB`.

This does not permit replacing the read with a byte access. If a 32-bit access at that location is invalid, the evidence provides no byte-read fallback.

### Aliasing changes later observations

Given overflow clear and zero set, suppose the initial destination overlaps a location in the first read group.

The initial byte write must precede that read. An implementation that reads all values first and performs the initial store afterward would not preserve the represented behavior.

### Distinct values from repeated reads

Given zero set and negative clear, suppose the first read four bytes before the first argument returns zero and the second returns 1.

The final address calculation uses 1. Reusing the first result would not satisfy the represented behavior.

## Uncertainties

The following remain **documented_only**:

- Whether the supplied entry point and recovered control flow correspond to valid executable instructions.
- The calling convention and method of supplying the implicit inputs.
- Byte order, alignment rules, memory attributes, and fault behavior.
- Definitive address width and address-overflow semantics.
- Coprocessor-operation transfer semantics, exceptions, and state effects.
- Whether the terminal region intentionally traps or merely reflects failed instruction recovery.
- Any continuation, cleanup, ownership action, or return behavior outside the recovered region.
- An application-level interpretation of the accessed locations.

A pointer-based helper or device routine might be an inferred purpose, but neither is established sufficiently to constrain callers.

## Evidence

This contract is based solely on the supplied static representation, which includes an explicit warning that instruction recovery encountered invalid data.

The strongest evidence consists of the represented argument widths, incoming condition predicates, access widths and offsets, ordering, unsigned shift combination, byte narrowing, and absence of a recovered normal-return path.

No runtime tests, execution traces, architecture validation, or independent implementation inspection have been performed. Acceptance examples express consequences of the static representation under stated conditions; they are not observed outcomes.
