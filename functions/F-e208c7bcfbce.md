<!-- Trusted provenance: F-e208c7bcfbce; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e208c7bcfbce

## Purpose

F-e208c7bcfbce selects between two lookup inputs using a 32-bit field in a context object, obtains a byte-addressed result through dependencies, and chooses one of two operations on a receiver:

- A null lookup result or a result whose first byte is zero selects the receiver’s fallback operation.
- A non-null result whose first byte is nonzero is transformed and passed to the receiver’s primary operation.
- A nonzero result from the primary operation triggers an additional indirect operation using that result and the original lookup pointer.

The established behavior distinguishes an absent or initially empty lookup result from a result that begins with a nonzero byte. It does not establish that the result is a string, that any operation displays text, or that either receiver operation reports an error.

This contract describes source-visible behavior. No runtime tests have been performed.

## Interface

**Inputs**

1. **Context address.** An address used to read a 32-bit field at byte offset `0x9c`.
2. **Receiver address.** An address passed as the first argument to either of two indirect operations. The receiver’s first 32-bit word supplies a dispatch-table address.

The function has no explicit length, capacity, ownership, allocator, or error-output argument.

**Output**

The function has no specified return value. Results produced by dependencies are consumed internally; none is returned to the caller.

**Widths and layouts**

The supplied representation uses a 32-bit address and word model:

| Location or value | Source-visible interpretation |
|---|---|
| Context at byte offset `0x9c` | One 32-bit integer, compared with zero |
| Receiver at byte offset `0` | A 32-bit dispatch-table address |
| Dispatch table at byte offset `0x08` | Primary-operation callable address |
| Dispatch table at byte offset `0x0c` | Fallback-operation callable address |
| First byte of a non-null lookup result | One unsigned 8-bit value |
| Result from F-9d0ecc5f6adf | One 32-bit value |
| Either selectable global lookup input | One 32-bit value |
| Result from F-8810f53aaa0b | One 32-bit value |
| Primary-operation result | One 32-bit integer, compared with zero |

These are byte offsets, not element indices. The two receiver dispatch entries are one 32-bit slot apart.

The unusual intermediate representation of the first byte as a pointer-shaped value does not introduce another memory access. Its behavioral significance is only whether the unsigned byte is zero.

**Preconditions**

For defined execution along a selected path:

- The context field must be readable.
- Each global value or indirect target accessed on that path must be readable.
- The receiver and dispatch table must support the accesses required by the operation actually selected.
- A non-null lookup result must permit at least a one-byte read.
- Every invoked target must be callable using the arguments described below.
- Any memory passed to a dependency must meet that dependency’s requirements at the time of the call.

There is no explicit validation of these conditions. In particular, a null receiver is not handled as a supported input. An ordinary path reaching a receiver operation dereferences the receiver.

## Behavior

### Selection and lookup

The function first reads the context’s 32-bit field at byte offset `0x9c` and tests it for equality with zero.

- Zero selects the **zero-case lookup input**.
- Any nonzero value selects the **nonzero-case lookup input**.

This is a binary test. Negative values, positive values, and distinct nonzero bit patterns do not select different behavior within this function.

After choosing the branch, the function calls F-9d0ecc5f6adf with no explicit arguments. Only after that call returns does it obtain the selected 32-bit global lookup input.

This ordering matters. The branch is based on the context-field value read before the dependency call, while the selected global input is obtained afterward. F-9d0ecc5f6adf is not assumed to preserve either the context or those globals.

The function then calls F-c0b020cd3882 with two arguments, in this order:

1. The 32-bit result of F-9d0ecc5f6adf.
2. The selected 32-bit global lookup input.

The result is treated as a byte-addressed pointer.

### Classification of the lookup result

If the lookup pointer is null, the function does not read a byte through it and selects the fallback operation.

If the pointer is non-null, the function reads its first byte as an unsigned 8-bit value:

- A zero byte selects the fallback operation.
- Any value from `1` through `255` selects the primary-operation path.

Only this first byte is directly examined by F-e208c7bcfbce. There is no established direct scan, terminator search, length calculation, or validation of later bytes.

The byte check precedes the transformation dependency. The function does not explicitly repeat that check after transformation.

### Fallback operation

For a null lookup pointer or a zero first byte, the function invokes the receiver operation at dispatch-table byte offset `0x0c`.

That operation receives one explicit argument: the receiver address.

The function then returns without invoking F-8810f53aaa0b, the primary receiver operation, or the additional indirect operation used after a nonzero primary result.

The fallback operation’s result, if any, is not consumed by this function. Its ownership effects and other side effects are not established.

### Primary-operation path

For a non-null lookup pointer with a nonzero first byte, the function calls F-8810f53aaa0b with the lookup pointer as its sole explicit argument.

The resulting 32-bit value becomes the second argument of the receiver’s primary operation. The primary operation is obtained through the receiver’s dispatch table at byte offset `0x08` and receives:

1. The receiver address.
2. The 32-bit result of F-8810f53aaa0b.
3. A zero-valued argument.

The dispatch-table access for this call occurs after F-8810f53aaa0b returns. Consequently, there is no basis for assuming that the receiver’s dispatch table still has its entry-time contents if that dependency can mutate shared or aliased state.

The primary operation returns a 32-bit integer:

- If the result is zero, F-e208c7bcfbce returns immediately.
- If the result is nonzero, the function invokes an additional indirect target obtained from a global target location.

That additional target receives:

1. The primary operation’s nonzero 32-bit result.
2. The original lookup pointer returned by F-c0b020cd3882.

The first argument is not normalized to a Boolean or converted to a positive value. The second argument is the original pointer value, not the transformation result.

After this indirect operation returns, F-e208c7bcfbce returns. There is no subsequent source-visible inspection of its result.

### Ordering boundaries

The established call and access order is:

1. Read and classify the context field.
2. Call F-9d0ecc5f6adf.
3. Obtain the selected global lookup input.
4. Call F-c0b020cd3882.
5. If its result is non-null, read the first byte.
6. Invoke the fallback operation, or call F-8810f53aaa0b and then the primary operation.
7. Only for a nonzero primary result, invoke the additional indirect target.

This does not establish a finer machine-level order among independent operand preparations within a single call expression.

If a dependency does not return normally, subsequent steps are not established to occur. No local recovery mechanism is shown.

## Side effects

The function contains no explicit direct memory store to the context, receiver, lookup result, or global state. Its direct memory effects are reads.

That does **not** make it side-effect-free. Every invoked dependency or indirect operation may have externally visible effects. The supplied evidence does not establish which memory, resources, devices, or other state they may affect.

Ownership and lifetime are particularly important:

- The lookup pointer is not explicitly copied into owned storage.
- No explicit allocation, deallocation, retain, or release operation is identifiable in this function.
- F-8810f53aaa0b cannot be assumed to preserve or release the lookup storage.
- The primary operation cannot be assumed to preserve that storage either.
- On the nonzero-primary-result path, the original pointer is forwarded after both calls.

Thus, a meaningful final indirect call requires a dependency-level lifetime arrangement that makes forwarding the original pointer valid. Whether that arrangement exists, and who owns the storage, remain unknown.

There is no explicit rollback. If a later call fails or does not return, earlier dependency side effects are not shown being undone.

## Dependencies

| Dependency | Explicit inputs | Result used here | Established role |
|---|---|---|---|
| F-9d0ecc5f6adf | None | 32-bit value | Supplies the first lookup argument |
| F-c0b020cd3882 | Provider result; selected global input | Byte-addressed pointer | Supplies the value classified by nullness and first byte |
| F-8810f53aaa0b | Non-null lookup pointer with an observed nonzero first byte | 32-bit value | Supplies the primary operation’s second argument |

Three additional call targets have no supplied opaque function IDs:

- Receiver dispatch slot at byte offset `0x08`: primary operation.
- Receiver dispatch slot at byte offset `0x0c`: fallback operation.
- Globally stored indirect target: invoked only after a nonzero primary result.

No semantic names, ownership guarantees, global-preservation guarantees, or error conventions are established for these targets.

The two selectable global lookup inputs are dependencies as data. Their values, types beyond their 32-bit representation, and mutability are not supplied.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. They assume valid addresses and normally returning dependencies unless stated otherwise.

1. **Zero selector with an absent result.**  
   Given a zero context field and a null lookup result, the call sequence is F-9d0ecc5f6adf, F-c0b020cd3882 using the zero-case input, then the receiver fallback operation. No lookup-byte read or transformation call occurs.

2. **Nonzero selector with an initially empty result.**  
   Given any nonzero context field and a non-null lookup pointer whose first byte is zero, the nonzero-case input is used. The first byte is read, and the fallback operation is invoked. Later bytes do not affect this function’s decision.

3. **Nonzero byte with a zero primary result.**  
   Given a lookup result beginning with byte `0x80`, F-8810f53aaa0b is invoked. The high bit does not make the byte fail the nonzero test. If the primary operation returns zero, the additional indirect operation is not invoked.

4. **Nonzero primary result.**  
   Given a valid nonempty lookup result and a primary result of `-1` in the 32-bit integer interpretation, the additional indirect operation receives that same result and the original lookup pointer. The fallback operation is not invoked.

5. **Context mutation during provider execution.**  
   Given an initially zero context field, changing that field to nonzero inside F-9d0ecc5f6adf does not switch the already selected branch. The zero-case global input remains selected.

6. **Selected-global mutation during provider execution.**  
   Given that F-9d0ecc5f6adf changes the selected global lookup input before returning, the subsequent lookup receives the value obtained after that call, not an assumed entry-time snapshot.

7. **Mutation after the byte test.**  
   If F-8810f53aaa0b changes the first byte to zero but otherwise returns under valid dependency contracts, this function still proceeds to the primary operation. It does not perform another emptiness test.

8. **Invalid receiver.**  
   A null or unreadable receiver is outside the documented preconditions. The function provides no supported no-op or fallback guarantee for that input.

## Uncertainties

The following remain **documented_only**:

- The semantic meaning of the context field and the two selected global values.
- Whether the lookup pointer denotes text, binary data, an object, borrowed storage, or owned storage.
- The meaning and valid range of the transformation result.
- Whether a zero primary result denotes success, completion, rejection, or another condition.
- The meaning of the nonzero primary result and the purpose of the additional indirect operation.
- Allocation, release, aliasing, mutation, and lifetime rules of all dependencies.
- Thread-safety, synchronization, and behavior under concurrent mutation.
- Exact calling conventions and return behavior of the indirect targets.

The supplied representation marks both terminal indirect operations as having unresolved control-flow reconstruction. Their argument relationships and path conditions are visible, but exact call-versus-tail-transfer mechanics are not established. This contract does not assign them cleanup, exception, or error-handler semantics.

## Evidence

**Established static evidence:** the context offset and zero test; provider-before-selected-global ordering; lookup argument order; null guard before the unsigned byte read; the two receiver dispatch offsets; transformation-before-primary sequencing; the primary operation’s zero third argument; and the conditional forwarding of its nonzero result with the original lookup pointer.

**Inferred intent:** the structure resembles selection of a value followed by either a fallback action or a transformed-value action. That interpretation is descriptive only; it does not establish application semantics.

**Verification status:** static analysis of the supplied restricted representation only. No implementation execution, runtime tests, dependency inspection, or ownership validation has been performed.
