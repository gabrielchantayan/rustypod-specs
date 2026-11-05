<!-- Trusted provenance: F-5caaec0467a7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5caaec0467a7

## Purpose

F-5caaec0467a7 coordinates conditional processing of an input record through dynamically selected operations on a receiver. It may invoke a preliminary operation, always invokes one of two main operations if execution continues, and may invoke a final operation.

A nonzero result from any invoked operation terminates processing and becomes this function’s result. If all invoked operations return zero, this function returns zero.

The established behavior is dispatch and result propagation. Describing the input as an “event,” a nonzero result as “handled,” or the final operation as a notification would be inferred intent, not an established semantic contract.

## Interface

### Inputs

The function receives two values:

- **Receiver:** a reference to an object whose first word supplies a dispatch-table address. A further receiver field at byte offset 80 participates in one conditional comparison.
- **Record:** an address-like value identifying a record with a discriminator at byte offset 4 and a discriminator-dependent field at byte offset 8.

Both values are passed unchanged as the two arguments to every dependency invoked by this function. “Unchanged” concerns the argument values, not the contents of the referenced memory: dependencies may modify those contents.

### Result

The result is an integer with these rules:

- Return the first nonzero result produced by an invoked dependency.
- Return zero if processing reaches the end.

There is no conversion to a Boolean and no normalization to a particular nonzero value. Negative results, if produced by a dependency, are propagated just like positive nonzero results.

The supplied representation uses integer-valued results and fields with four-byte spacing. This contract treats those integer values as 32-bit under that representation. The actual executable’s integer width, pointer width, byte order, and calling convention are not independently established.

### Relevant memory layout

Only the following locations are used directly:

| Region | Byte offset | Interpretation |
|---|---:|---|
| Receiver | 0 | Dispatch-table address |
| Receiver | 80 | Integer value compared with the record’s field at offset 8 when the discriminator is 15 |
| Record | 4 | Integer discriminator |
| Record | 8 | Nested-record address when the discriminator is 22; integer comparison value when it is 15 |
| Nested record | 4 | Integer subtype |
| Dispatch table | 224 | Callable entry for F-dep-04 |
| Dispatch table | 268 | Callable entry for F-dep-02 |
| Dispatch table | 272 | Callable entry for F-dep-03 |
| Dispatch table | 292 | Callable entry for F-dep-01 |

The receiver offset of 80 follows from an indexed integer access with index 20 and the supplied four-byte integer model. The other offsets are explicit byte offsets.

These are partial layouts, not complete object definitions. No interpretation is established for other fields, including the first word of the record or nested record.

### Preconditions

For execution to complete normally:

- Each directly accessed address must be readable for the applicable value width.
- Each selected dispatch-table entry must designate an operation compatible with two arguments and an integer result.
- The receiver and record must remain usable at every access that execution reaches, including accesses after dependency calls.
- When a discriminator read yields 22, the nested address subsequently obtained from record offset 8 must support a read at offset 4.

There is no explicit validation of null references, address ranges, alignment, object type, table size, or callable-entry validity.

Preconditions are path-dependent. For example, the nested record is not accessed when the discriminator is not 22. The receiver field at offset 80 is not accessed unless the final discriminator read yields 15.

## Behavior

### 1. Preliminary eligibility check

The function first reads the record’s discriminator at byte offset 4.

If that value is not 22, it skips the preliminary operation without reading the nested record.

If the discriminator is 22, it:

1. Reads the nested-record address from record offset 8.
2. Reads the subtype from nested-record offset 4.
3. Tests whether that subtype is zero.

Only a discriminator of 22 combined with a subtype of zero triggers F-dep-01.

When triggered, F-dep-01 is resolved through the receiver’s dispatch-table address and the entry at byte offset 292. It is called with the receiver and record arguments.

- A nonzero result is returned immediately.
- A zero result permits processing to continue.

The preliminary eligibility values are not reused for main-operation selection. A zero result from F-dep-01 does not imply that the receiver, record, nested record, or dispatch table is unchanged.

### 2. Main-operation selection

If the preliminary stage did not terminate processing, the function reads the record discriminator again.

If this read yields 22, it reads the nested-record address again and then reads the nested subtype again. It selects:

- F-dep-02 when the current subtype is 42.
- F-dep-03 for every other subtype.

If the discriminator read does not yield 22, it selects F-dep-03 without accessing the nested record for this stage.

The selected operation is resolved through the receiver’s dispatch-table address:

- F-dep-02 uses the entry at byte offset 268.
- F-dep-03 uses the entry at byte offset 272.

The selected operation is then called with the receiver and record arguments.

Exactly one main operation is invoked on every path that reaches this stage and completes its required accesses. F-dep-02 and F-dep-03 are mutually exclusive within this invocation.

A nonzero main-operation result is returned immediately. A zero result leads to the final eligibility check.

### 3. Final eligibility check

After the main operation returns zero, the function reads the record discriminator again.

If it is not 15, processing ends with result zero. Neither the record field at offset 8 nor the receiver field at offset 80 is needed for this final comparison.

If it is 15, the function reads:

1. The record’s integer field at byte offset 8.
2. The receiver’s integer field at byte offset 80.

It compares these integer values for equality. This is not a comparison with the receiver’s address, the record’s address, or the nested subtype.

If the values differ, processing ends with result zero.

If the values are equal, the function resolves F-dep-04 through the receiver’s current dispatch-table address and the entry at byte offset 224, then calls it with the receiver and record arguments.

- A nonzero result is returned immediately.
- A zero result leads to the function returning zero.

### Read and call ordering

The direct access sequence matters because dependency calls occur between successive checks:

- Preliminary discriminator and possible nested-subtype reads precede F-dep-01.
- Main selection uses fresh discriminator and, where applicable, nested-record reads after F-dep-01 returns zero.
- Final eligibility uses fresh record and receiver reads after the main operation returns zero.
- Each invoked dependency is resolved through a receiver dispatch-table lookup at its own call site. A table address obtained for an earlier call is not retained for later calls.

Consequently, dependency mutations can change later selection and eligibility. An initially nonmatching final discriminator can become 15 during the main operation, and an initially matching one can cease to match.

Within each stage, selection is based on the values actually read for that stage. The function does not provide a transaction, lock, atomic snapshot, or explicit synchronization around those reads. Concurrent modification semantics are not established by the supplied evidence.

### Call-count boundaries

For an invocation that completes normally:

- At least one dependency is called.
- At most three dependencies are called.
- F-dep-01 is called at most once.
- Exactly one of F-dep-02 and F-dep-03 is called unless F-dep-01 returns nonzero first.
- F-dep-04 is called at most once and only after a main operation returns zero.

Three calls are possible when the preliminary operation is eligible and returns zero, the main operation returns zero, and the post-main record satisfies the final condition. Changes made by dependencies can make these conditions hold at different stages.

## Side effects

The function contains no direct writes to the receiver, record, nested record, or dispatch table. Its directly established effects are memory reads, indirect calls, and returning an integer.

The dependencies may have side effects. Their implementations are unavailable, so this contract does not assume that they:

- Preserve global state.
- Preserve receiver or record contents.
- Preserve dispatch-table contents.
- Retain, transfer, or release any particular allocation.
- Return normally.
- Leave earlier side effects reversible.

A zero result means only that this function continues. It is not evidence that the dependency made no changes.

There is no explicit allocation, release, ownership transfer, rollback, or cleanup operation in this function. Argument passing alone does not establish ownership transfer. Nevertheless, a dependency that invalidates memory needed by a later stage would prevent normal completion unless execution terminates before the later access.

The function performs no explicit error translation. It propagates nonzero dependency results without assigning them a documented meaning. Memory-access failures and invalid indirect calls have no explicit recovery path in the supplied behavior.

## Dependencies

The following opaque identifiers are local labels for indirect call sites. They do not establish the identity, name, or implementation of the callable stored in a dispatch table.

| Opaque ID | Table offset | Invocation condition | Treatment of result |
|---|---:|---|---|
| F-dep-01 | 292 | Preliminary discriminator is 22 and preliminary nested subtype is zero | Nonzero terminates; zero continues |
| F-dep-02 | 268 | Main-stage discriminator is 22 and main-stage nested subtype is 42 | Nonzero terminates; zero continues |
| F-dep-03 | 272 | Every other main-stage selection | Nonzero terminates; zero continues |
| F-dep-04 | 224 | Main result is zero, final discriminator is 15, and the two comparison fields are equal | Nonzero is propagated; zero yields final zero |

Every dependency receives the same receiver and record argument values originally supplied to F-5caaec0467a7.

Different slots may contain the same callable. Conversely, a slot may resolve to different callables across invocations or after mutations. Neither uniqueness nor immutability is established.

## Acceptance examples

These are conditional acceptance scenarios derived from the static behavior. They are not reports of executed tests. Except where a mutation is explicitly described, assume referenced state remains stable, required memory is valid, and every invoked dependency returns normally.

### Preliminary short-circuit

Given a discriminator of 22 and nested subtype zero, if F-dep-01 returns −7:

- F-dep-01 is called.
- Neither main operation nor F-dep-04 is called.
- The function returns −7 exactly.

### Special main selection

Given a discriminator of 22 and nested subtype 42:

- F-dep-01 is skipped.
- F-dep-02 is called.
- F-dep-03 is not called.

If F-dep-02 returns 9, the function returns 9 without performing the final eligibility reads.

### Default main selection

Given a discriminator of 22 and nested subtype 7:

- F-dep-01 is skipped.
- F-dep-03 is called.

If that call returns zero and leaves the discriminator at 22, F-dep-04 is skipped and the function returns zero.

A subtype other than zero or 42 is not explicitly rejected.

### Preliminary mutation affects selection

Given an initial discriminator of 22 and subtype zero, suppose F-dep-01 changes the nested subtype to 42 and returns zero.

The main stage rereads the subtype and calls F-dep-02, not F-dep-03.

If instead F-dep-01 changes the discriminator to a value other than 22, the main stage calls F-dep-03 without reading the nested subtype for that stage.

### Matching final fields

Given a discriminator of 15, a record comparison value of 123, and a receiver comparison value of 123:

- F-dep-01 is skipped.
- F-dep-03 is called.
- If F-dep-03 returns zero and preserves those conditions, F-dep-04 is called.

If F-dep-04 returns 6, the function returns 6. If it returns zero, the function returns zero.

### Unequal final fields

Given a final discriminator of 15, a record comparison value of 123, and a receiver comparison value of 124, F-dep-04 is not called. After a zero main result, the function returns zero.

### Main mutation creates final eligibility

Suppose the main stage selects F-dep-02 using discriminator 22 and subtype 42. If that operation returns zero after changing the discriminator to 15 and making the comparison fields equal, F-dep-04 is subsequently called.

Final eligibility is therefore not restricted by the discriminator that selected the main operation.

### Unused nested address

Given a discriminator other than 22 at both the preliminary and main checks, neither stage dereferences a nested-record address.

If the main operation returns zero and the final discriminator is also not 15, the final comparison fields are not read either. This does not remove the requirement for valid receiver dispatch accesses.

## Uncertainties

The following remain **documented_only**:

- The application-level meanings of discriminators 22 and 15, subtypes zero and 42, and the comparison fields.
- Whether zero means “not handled,” “success,” or another condition outside this dispatch protocol.
- The identity, full signature, side effects, ownership behavior, and failure behavior of each dependency.
- Native pointer width, endianness, alignment requirements, and calling convention.
- Whether the 32-bit integer interpretation precisely matches the original executable.
- Object lifetime guarantees, aliasing restrictions, and concurrency rules.
- Whether dependencies can throw, terminate, transfer control nonlocally, or invalidate subsequent accesses.

A staged handling or dispatch pipeline is a plausible interpretation, but it is not required to describe the established behavior.

## Evidence

The supplied static representation establishes the conditional reads, dispatch-table offsets, call order, fresh checks after calls, equality comparison, result propagation, and absence of direct state writes.

The four-byte integer interpretation is supported by the supplied field representation and indexed access pattern; native executable details have not been independently verified.

No dependency implementations, complete type definitions, caller contracts, or runtime observations were supplied. No runtime tests have been performed. The acceptance examples specify expected behavior under stated conditions rather than observed outcomes.
