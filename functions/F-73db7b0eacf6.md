<!-- Trusted provenance: F-73db7b0eacf6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-73db7b0eacf6

## Purpose

F-73db7b0eacf6 prepares a four-word invocation record, calls an unconditional dependency, and conditionally passes that record to a second dependency. When the conditional call occurs, the function stores the difference between two words of the resulting record into a field addressed relative to its original first input. It returns a 64-bit value composed from the record’s first two words.

The established behavior is a conditional call-and-update operation. The supplied evidence does not establish an application-level meaning for the object, the selected dependency argument, or the computed difference. Interpretations such as position adjustment, range calculation, or callback dispatch are not part of this contract.

This specification describes behavior supported by the supplied static representation. It does not assert that either dependency preserves global state, leaves the invocation record unchanged, or follows any particular ownership convention.

## Interface

### Inputs

The function receives four 32-bit input words:

| Input | Established use |
|---|---|
| First | Copied into the first record word and independently used as the original base address for two object fields. |
| Second | Copied into the second record word, which supplies the upper 32 bits of the result. |
| Third | Copied into the third record word, which participates in the conditional difference calculation. |
| Fourth | Copied into the fourth record word. The enclosing function does not otherwise directly consume it. |

The supplied representation treats the first and third inputs as signed 32-bit integers and the second and fourth as untyped 32-bit quantities. The first input also participates in pointer formation. Its role as an address must therefore be distinguished from its role as a record word.

The original first input remains the basis for object-field accesses even if the conditional dependency changes the record’s first word.

### Invocation record

Before either dependency is called, the function creates a local record containing the four input words in input order:

| Byte offset from record start | Width | Initial contents |
|---|---:|---|
| 0 | 32 bits | First input |
| 4 | 32 bits | Second input |
| 8 | 32 bits | Third input |
| 12 | 32 bits | Fourth input |

The supplied representation places these words consecutively and passes the address of the first word to the conditional dependency. This supports a 16-byte record interpretation. It does not establish that the dependency reads or writes every word, nor that it is restricted to this extent.

No byte order is needed to describe the word-level behavior. A byte-level interchange format is not established.

### Object fields

Two 32-bit fields are addressed relative to the original first input:

| Offset | Access by this function |
|---|---|
| `0x15C` | Read to decide whether to make the conditional call; read again to obtain its first argument when the branch is taken. |
| `0x16C` | Written after the conditional dependency returns. |

The field at `0x15C` is tested for zero. No stronger type is established for its nonzero contents.

### Output

The return value is a 64-bit bit composition:

- Bits 31–0 contain the record’s first word at return time.
- Bits 63–32 contain the record’s second word at return time.

This is a concatenation of two 32-bit words, not sign extension of the first word. The third and fourth words are not returned directly.

### Preconditions

For ordinary completion:

- Address formation from the original first input must be valid for the required accesses.
- The 32-bit field at offset `0x15C` must be readable after the unconditional dependency returns.
- If the branch is taken and its dependency returns normally, the 32-bit field at offset `0x16C` must be writable when the subsequent store occurs.
- Both dependencies must satisfy whatever additional requirements their implementations impose.

No explicit null check, bounds check, alignment check, or access-permission check is established. A zero first input is not a special no-operation case.

## Behavior

### Preparation and unconditional call

The four record words are initialized from the four inputs before F-7c6efef9c7e7 is invoked.

F-7c6efef9c7e7 is called without explicit arguments in the supplied representation. Its return value, if any, is not consumed.

The object’s condition field is read only after this call returns. Consequently, branch selection uses the field value observed after the dependency’s effects, not a value captured on entry.

The absence of explicit arguments does not establish absence of effects. In particular, this contract does not assume that F-7c6efef9c7e7 preserves global state or memory accessible through other means.

### Conditional field reads and dispatch

After the unconditional dependency returns, the function reads the 32-bit field at original base plus `0x15C`.

If that read produces zero:

- F-59fa3fd3efc5 is not called.
- The enclosing function performs no difference calculation or store to offset `0x16C`.
- Execution proceeds to return-value construction.

If that read produces a nonzero value:

- The same field is read again.
- The value obtained by this second read becomes the first argument to F-59fa3fd3efc5.
- The address of the invocation record’s first word becomes its second argument.

The branch-controlling value and the value passed to the dependency are separate observations in the supplied representation. No explicit operation by this function occurs between them that updates the field, but stability cannot be guaranteed in the presence of external modification or special memory semantics.

In particular, the contract does not add a second nonzero check: if the first observation is nonzero and the second observation is zero, the dependency is still called with zero as its first argument.

### Post-call difference and object write

After F-59fa3fd3efc5 returns normally, the function takes the record’s third word minus its first word and stores the result into the 32-bit field at original base plus `0x16C`.

Both operands are the record values used after the conditional dependency returns. They are not necessarily the original inputs.

The destination remains relative to the original first input. Changing the record’s first word affects the subtraction and the return value; it does not redirect this object-field store.

The dependency’s own return value is ignored. There is no established status check or error-dependent branch between its normal return and the store.

### Arithmetic width

The subtraction is represented as an operation on two signed 32-bit values with a 32-bit destination.

When the mathematical difference is within the signed 32-bit range, the resulting stored value is established. For example, a third word of 150 and a first word of 100 produce 50.

When the mathematical difference lies outside the signed 32-bit range, the supplied representation alone does not establish a portable overflow rule. Machine-width wraparound is plausible, but it is not adopted here as an independently proven guarantee. Overflow behavior is documented_only.

Likewise, this contract requires valid object-field addressing rather than assigning a wraparound meaning to overflowing address arithmetic.

### Return construction

After the optional store, the function returns the composition of the record’s second and first words, in that high-to-low order.

No explicit error value is defined. If a dependency does not return normally, or a required memory access fails, this contract does not promise that later operations or the final return occur.

## Side effects

The enclosing function establishes these effects:

- Initialization of a local four-word invocation record.
- One unconditional invocation of F-7c6efef9c7e7.
- One read of the condition field, followed by a second read only on the nonzero branch.
- Zero or one invocation of F-59fa3fd3efc5.
- One 32-bit write to offset `0x16C` after a normally returning conditional call.
- A return value reflecting the record’s first two words at return construction.

The store to offset `0x16C` follows the conditional dependency’s effects. If that dependency also writes the destination, this function subsequently overwrites it, provided the destination remains valid and execution reaches the store.

A zero condition suppresses only the conditional call and this function’s explicit difference store. It does not imply that the object or any other memory remains unchanged: the unconditional dependency has already executed.

### Ownership and lifetime

The record is local to this invocation. Its address is exposed to F-59fa3fd3efc5 during the conditional call.

No allocation, explicit release, or ownership transfer is shown in the enclosing function. This does not establish what either dependency allocates, releases, retains, or invalidates.

The record pointer must not be treated as having a guaranteed usable lifetime beyond the invocation. Whether F-59fa3fd3efc5 retains it is unknown.

The first argument supplied to that dependency is an opaque 32-bit field value. Calling it a handle, pointer, owned resource, or borrowed resource would exceed the evidence.

## Dependencies

| Opaque ID | Established interaction | Unestablished properties |
|---|---|---|
| F-7c6efef9c7e7 | Called once before reading the condition field, with no explicit arguments shown. Its result is unused. | Purpose, hidden inputs, state changes, memory effects, ownership behavior, and failure modes. |
| F-59fa3fd3efc5 | Called only after a nonzero condition observation. Receives the separately reread field value and the record address. Its result is unused. | Meaning of the field value, record interpretation, mutation extent, pointer retention, ownership behavior, and failure modes. |

A likely intent is that F-59fa3fd3efc5 may transform the record before the enclosing function computes and returns derived values. That is an inference, not a requirement that the dependency actually mutate anything.

## Acceptance examples

These are conditional acceptance examples derived from static evidence, not executed tests. Each assumes valid memory, normal dependency returns, and only the dependency effects explicitly stated.

### Zero condition

Given:

- Original base: `0x00001000`.
- Second input: `0xAABBCCDD`.
- The unconditional dependency leaves the record unchanged.
- The condition read at address `0x0000115C` yields zero.

Expected behavior:

- F-59fa3fd3efc5 is not called.
- This function does not write address `0x0000116C`.
- The return value is `0xAABBCCDD00001000`.

This does not imply that F-7c6efef9c7e7 made no other changes.

### Conditional call without record mutation

Given:

- Original base and first record word: `0x00001000`.
- Second record word: `0x11223344`.
- Third record word: `0x00001060`.
- Both condition-field reads yield the same nonzero opaque value.
- Neither dependency changes the record.

Expected behavior:

- F-59fa3fd3efc5 receives that opaque value and the record address.
- The function stores `0x00000060` at address `0x0000116C`.
- The return value is `0x1122334400001000`.

### Conditional dependency changes the record

Given original base `0x00001000`, suppose F-59fa3fd3efc5 leaves:

- First record word: `0x00002000`.
- Second record word: `0x55667788`.
- Third record word: `0x00002030`.

Expected behavior:

- The function stores `0x00000030` at address `0x0000116C`.
- It does not redirect that store to address `0x0000216C`.
- The return value is `0x5566778800002000`.

### Unconditional dependency changes branch selection

Given a zero condition field on entry, suppose F-7c6efef9c7e7 changes it to a nonzero value before returning.

Expected behavior:

- The function follows the nonzero branch because it reads the condition after that call.
- The entry-time zero does not suppress the conditional call.

### Different values on the two condition reads

If an execution environment permits the first condition read to yield a nonzero value and the second to yield zero:

- F-59fa3fd3efc5 is called with zero as its first argument.
- There is no additional zero check before that call.

This example does not assert thread safety or defined data-race behavior.

## Uncertainties

The following remain documented_only:

- The application-level purpose and full layout of the object.
- The meaning and ownership of the field at `0x15C`.
- Dependency contracts, including hidden state, record mutations, retained pointers, resource release, and failure behavior.
- Signed subtraction behavior outside the representable 32-bit range.
- Byte order, alignment requirements, and details of the calling convention.
- Whether repeated field reads correspond to distinct physical accesses under the actual execution model.
- Concurrency guarantees, atomicity, synchronization, and memory ordering.
- Behavior after invalid addressing, dependency failure, or invalidation of the original object.

The consecutive four-word record layout is supported by the supplied local-storage representation. An independently verified binary interface for that record is not available.

## Evidence

Established static evidence consists of the supplied function representation and the provided dependency-ID mapping. It supports the four input-word initializations, unconditional dependency call, conditional field reads, record-address argument, post-call subtraction and store, and 64-bit return composition.

The behavioral order above describes operations explicit in the enclosing function. It does not prescribe internal dependency operations or the evaluation order of otherwise independent operand reads.

Inferred intent is limited to the possibility that the conditional dependency transforms the record for subsequent calculation. No domain interpretation is asserted.

No implementation was executed. No runtime tests, memory traces, dependency implementations, or independent machine-level verification were available. The acceptance examples are specification conditions, not reports of observed results.
