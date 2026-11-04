<!-- Trusted provenance: F-9ad5e4fb511b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9ad5e4fb511b

## Purpose

F-9ad5e4fb511b conditionally dispatches operations on an object obtained through an input object's indirect dispatch table.

If the initial dispatch returns null, the function performs no further visible work. Otherwise, it obtains a related receiver from the returned object and tries one lookup, followed by a fallback lookup only if the first returns null. A lookup result that is non-null and distinct from the initially returned object receives an operation with argument zero. The initially returned object then receives an operation with argument one.

This sequence is established by the supplied static representation. The semantic meanings of the objects, lookup selector, and zero/one arguments are not established. In particular, the final operations must not be described as destruction, release, activation, or ownership transfer without additional evidence.

## Interface

**Identity:** F-9ad5e4fb511b.

**Input:** One pointer to a structure whose first machine-level field supplies a dispatch-table address. The recovered parameter type is a pointer to an integer, but its use is object-like rather than arithmetic.

**Output:** No defined return value. The recovered interface is void-returning. Results produced by the intermediate lookup calls determine control flow; results from the final operation calls are ignored.

**Relevant layout:** The supplied representation uses integer-sized object fields as addresses. Under its apparent 32-bit address model:

| Location | Required interpretation |
|---|---|
| Input object, byte offset 0 | Dispatch-table address |
| Input object's dispatch table, byte offset 252 | Initial acquisition callable |
| Initially returned object, byte offset 52 | Related receiver address |
| Related receiver, byte offset 0 | Dispatch-table address |
| Related receiver's dispatch table, byte offset 288 | Primary lookup callable |
| Related receiver's dispatch table, byte offset 292 | Fallback lookup callable |
| Selected object's dispatch table, byte offset 160 | Operation callable |
| Initially returned object's dispatch table, byte offset 160 | Operation callable |

The field at byte offset 52 corresponds to integer-array index 13 in the supplied representation, assuming four-byte integers. The dispatch-table displacements are byte offsets, not integer-array indices.

**Argument values:**

- Both lookups receive the captured related receiver and selector 6144, equivalently hexadecimal 0x1800.
- A distinct, non-null selected object receives an operation with that object as its first explicit argument and integer zero as its second.
- The initially returned object receives an operation with that object as its first explicit argument and integer one as its second.

The initial acquisition call has **no explicit arguments in the supplied call expression**. Its callable is obtained through the input object, but this does not establish that the input object is passed as a receiver through any particular calling convention.

**Preconditions:**

- The input pointer must support the initial dispatch-table read. There is no null-input guard.
- Every reached dispatch-table pointer and callable slot must be valid for the corresponding access and invocation.
- A non-null initial result must support the related-receiver field read.
- The captured related receiver must support dispatch. There is no null check on that receiver.
- Objects used after a dependency call must remain usable for the later accesses. The function does not independently establish their lifetimes.

These are access requirements, not evidence of validation or ownership.

## Behavior

### Initial acquisition and early return

The function first reads the input object's dispatch-table address, obtains the callable at byte displacement 252, and invokes it. The result is interpreted as an object pointer.

A null result causes an immediate return. In this case, the function does not read the related-receiver field, perform either lookup, or invoke either object operation.

The acquisition dependency may itself have side effects. Therefore, the null-result case does not imply that the invocation as a whole leaves all state unchanged.

### Related receiver and primary lookup

For a non-null initial result, the function reads the address stored at byte offset 52 of that object, under the apparent 32-bit layout. This address becomes the related receiver.

The function reads that receiver's dispatch-table address and invokes the callable at byte displacement 288. The explicit arguments are the captured receiver and selector 6144.

A non-null primary result becomes the selected object. It suppresses the fallback lookup even if it is pointer-equal to the initial object.

A null primary result triggers the fallback lookup.

### Conditional fallback

The fallback uses the **same captured related-receiver address** and the same selector, 6144. It dispatches through the callable at byte displacement 292.

The fallback dispatch expression reads the related receiver's dispatch-table address again. Consequently, if the primary lookup changes that dispatch-table field and returns null, the fallback uses the table available at the later read.

The function does not reread the related-receiver field from the initial object. Changing that field during the primary lookup does not, by itself, change the receiver address used for the fallback.

The fallback result becomes the selected object, including when that result is null.

### Conditional operation on the selected object

After lookup resolution, the function tests whether the selected object is non-null and pointer-distinct from the initial object.

If both conditions hold, it reads the selected object's dispatch-table address and invokes the callable at byte displacement 160 with explicit arguments consisting of the selected object and zero.

If the selected object is null, this call is skipped.

If the selected object equals the initial object, this call is also skipped. Equality is an address comparison; the function does not inspect object contents, compare identities through a dependency, or search for other forms of aliasing.

There is no subsequent fallback or retry after selecting a non-null object. In particular, failure or unusual behavior in this operation does not lead to the fallback lookup.

### Final operation on the initial object

After the lookup path and any selected-object operation, the function reads the initial object's dispatch-table address and invokes its callable at byte displacement 160 with explicit arguments consisting of the initial object and one.

This operation is reached on every non-null acquisition path **provided all preceding accesses and calls return normally**.

The initial object address is retained across the intervening calls. Its dispatch-table address is read for this final dispatch rather than being cached before those calls. A prior dependency can therefore affect the final callable by changing that object's dispatch-table field.

No later use of the initial object is visible in the supplied representation.

### Widths and arithmetic

The wrapper performs no visible integer arithmetic on the selector or the zero/one operation arguments. All three constants fit in a signed 32-bit integer; there is no visible truncation, sign-dependent branch, or overflow involving them.

The layout interpretation depends on the recovered integer and pointer widths. The supplied representation strongly suggests four-byte integer fields carrying 32-bit addresses, but it does not independently establish a complete ABI. A wider-pointer implementation must not simply retain the same integer-array indexing and assume the same field location.

## Side effects

The function contains no explicit assignment to object fields, global variables, or caller-owned memory. Its visible direct actions are memory reads, pointer comparisons, indirect calls, and control-flow decisions.

Dependency calls may write memory, change dispatch tables, alter global state, allocate, release resources, invalidate objects, or terminate control flow. None of those behaviors is established or excluded by this function alone.

The observable call ordering, when calls return normally, is:

1. Initial acquisition.
2. Primary lookup, only for a non-null acquisition result.
3. Fallback lookup, only for a null primary lookup result.
4. Selected-object operation with zero, only for a non-null selected object distinct from the initial object.
5. Initial-object operation with one, for the non-null acquisition path.

There is no explicit synchronization, exception handling, error translation, or cleanup branch. Invalid accesses and failures inside dependencies are not converted into a reported status.

**Ownership:** No ownership or borrowing contract is established. Pointer return values are used without visible reference-count adjustments. The operation arguments zero and one cannot safely be interpreted as deallocation flags from this evidence alone.

## Dependencies

The following opaque identifiers are local specification references for unresolved indirect call roles. They are not independently established identities of the runtime call targets.

| Opaque identifier | Dispatch source | Explicit arguments | Result use |
|---|---|---|---|
| F-9ad5e4fb511b-D01 | Input object's table, byte offset 252 | None shown | Interpreted as initial object pointer |
| F-9ad5e4fb511b-D02 | Related receiver's table, byte offset 288 | Receiver, 6144 | Primary selected-object candidate |
| F-9ad5e4fb511b-D03 | Related receiver's table, byte offset 292 | Receiver, 6144 | Fallback selected-object candidate |
| F-9ad5e4fb511b-D04 | Selected object's table, byte offset 160 | Selected object, zero | Return value ignored |
| F-9ad5e4fb511b-D05 | Initial object's table, byte offset 160 | Initial object, one | Return value ignored |

These roles need not correspond to five distinct implementations. Conversely, the two operations at displacement 160 need not resolve to the same implementation when their receiver objects differ.

No dependency is assumed to preserve global state, preserve object contents, retain references, or release memory.

## Acceptance examples

These examples specify conditional observations, not executed tests. Except where stated otherwise, assume valid layouts, valid call targets, and normally returning dependencies.

1. **Acquisition returns null.**  
   F-9ad5e4fb511b-D01 is invoked once. No related-receiver field is read and none of D02 through D05 is invoked.

2. **Primary lookup returns a distinct object.**  
   D01 returns object A; A supplies receiver R; D02 returns object B, with B non-null and unequal to A. D03 is skipped. B receives the operation with zero before A receives the operation with one.

3. **Primary lookup returns the initial object.**  
   D02 returns A. D03 is skipped because the primary result is non-null. The operation with zero is skipped because the selected pointer equals A. A receives only the operation with one.

4. **Fallback returns a distinct object.**  
   D02 returns null and D03 returns B, distinct from A. Both lookups receive R and selector 6144, in primary-then-fallback order. B receives zero before A receives one.

5. **Both lookups return null.**  
   D02 and D03 execute. No selected-object operation occurs. A still receives the operation with one.

6. **Fallback returns the initial object.**  
   D02 returns null and D03 returns A. No operation with zero occurs. A receives the operation with one.

7. **Primary lookup replaces the receiver's dispatch table.**  
   If D02 changes R's dispatch-table field and returns null, the fallback target is obtained through R's later dispatch-table value. The function retains R's address.

8. **Primary lookup replaces the initial object's related-receiver field.**  
   If D02 changes that field and returns null, the fallback still uses the previously captured R. The field is not reread.

9. **Selected-object operation replaces the initial object's dispatch table.**  
   If B's operation changes A's dispatch-table field and returns normally while leaving A usable, the final operation resolves through A's updated table.

10. **A preceding dependency does not return.**  
    If a lookup or selected-object operation terminates execution or otherwise fails to return normally, the final operation on A is not guaranteed to occur. The wrapper provides no visible recovery mechanism.

## Uncertainties

**documented_only — ABI and widths:** The representation suggests a 32-bit layout. Exact calling convention, pointer width, alignment requirements, and implicit argument passing are not independently established.

**documented_only — Initial call receiver:** The initial call shows no explicit arguments. Whether it consumes an implicit receiver or other machine state is unknown.

**documented_only — Semantic purpose:** Lookup, acquisition, and operation are descriptive roles inferred from data flow. The domain meaning of selector 6144 and arguments zero and one is unknown.

**documented_only — Lifetime and ownership:** The evidence does not establish whether returned addresses are borrowed, retained, newly allocated, or releasable. It does not establish that any dependency preserves objects needed later.

**documented_only — Exceptional control flow:** The final indirect transfer is represented as a call with a control-flow recovery warning. Ordinary call-versus-tail-transfer details and unusual continuation behavior are not established.

**documented_only — Concurrency:** No concurrency contract is supplied. The repeated dispatch-table reads describe the static access sequence, not a guarantee of atomicity or safe concurrent mutation.

## Evidence

The sole evidence is the supplied static representation of F-9ad5e4fb511b. It establishes the explicit null checks, dispatch displacements, field access, constants, short-circuit fallback condition, pointer-identity exclusion, and ordering of the represented calls.

Object-oriented terminology and the apparent 32-bit layout are interpretations of that representation, not independently verified runtime facts. Dependency semantics remain unresolved.

No implementation access, external dependency inspection, or runtime tests have been performed.
