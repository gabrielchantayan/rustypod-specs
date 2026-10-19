<!-- Trusted provenance: F-78d399bb1bbd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-78d399bb1bbd

## Purpose

F-78d399bb1bbd conditionally initiates two actions after examining integer fields in two caller-supplied storage regions. Both regions must satisfy the eligibility condition. Rejection of either region prevents both actions.

The first action receives three integer values from the first region. The second action has no explicit arguments and is reached only after the first action returns normally. The function itself returns no value and provides no success or failure status.

This is an established control-flow role, not an established application-level purpose. The supplied evidence does not identify the fields as flags, addresses, handles, sizes, or ownership metadata. Allocation, cleanup, notification, and resource transfer must not be inferred from the conditional calls.

## Interface

The interface has two pointer inputs, described here as the **primary region** and **secondary region**. Both designate storage accessed as consecutive signed-integer elements.

| Field | Primary region | Secondary region |
|---|---|---|
| Element 0 | Participates in eligibility; also supplies the first argument to F-6218768696f9 on acceptance | Participates in eligibility |
| Element 1 | Participates in eligibility; also supplies the second argument to F-6218768696f9 on acceptance | Participates in eligibility |
| Element 2 | Supplies the third argument to F-6218768696f9 on acceptance | Not directly accessed |

There is no accompanying length, capacity, error destination, or ownership indicator. The return interface carries no value.

### Widths and layout

The supplied material identifies the elements as signed integers but does not independently establish their width, representation, alignment, or storage size. Let **W** denote their bit width and **S** their storage size.

The element positions have offsets of zero, S, and twice S from the respective region’s start. These are integer-element positions, not evidence of a packed byte format. Pointer width, byte order, and a concrete value of S remain unspecified.

Any 32-bit numerical interpretation in this contract is conditional. It must not be treated as proof of the target’s integer model.

### Access-dependent preconditions

The primary region must permit reading elements 0 and 1 whenever the function is entered. The secondary region must permit reading elements 0 and 1 only if the primary region is accepted. Primary element 2 must be readable only if both regions are accepted.

These requirements concern the locations actually accessed. A rejected primary region does not create a direct access requirement for the secondary region or primary element 2. A rejected secondary region does not create a direct access requirement for primary element 2.

Every accessed element must satisfy the execution environment’s object-validity and alignment requirements. Arithmetic used in eligibility evaluation must also have defined semantics under that environment.

There is no explicit validation of pointer values, readable extent, alignment, or arithmetic validity. Violations are not converted into a documented error result.

## Behavior

### Eligibility meaning

Eligibility depends on the first two elements of a region. The supplied expression uses a one-bit left shift of element 0, complements the shifted result, combines that mask with element 1, and tests whether the resulting signed integer is negative.

The complement applies to the result of the shift. It is not a complement of element 0 performed before shifting. The distinction matters for the resulting bit pattern.

Under a fixed-width, two’s-complement bit-vector interpretation, eligibility has this equivalent field-level meaning:

- Element 1 has its most significant bit set.
- Element 0 has its next-to-most-significant bit clear.

Both properties are required. With bit positions counted from zero at the least significant bit, the relevant positions are W − 1 in element 1 and W − 2 in element 0. For a conditional 32-bit interpretation, those positions are 31 and 30.

Under that interpretation, all other bits are irrelevant to the eligibility result. They are not necessarily irrelevant to the dependency receiving the primary region’s values.

This bit-vector characterization does not establish that every possible signed input has defined evaluation semantics. In particular, it does not turn an invalid signed left shift into an ordinary rejection.

Under strict signed-shift rules, nonnegative element-0 values below $2^{W-2}$ have a representable doubled value. For that subset, the shifted value’s sign bit is clear, and eligibility reduces to element 1 being negative. Negative element-0 values and larger nonnegative values require an independently established arithmetic model before their behavior can be promised.

### Acceptance and rejection

The primary region controls whether the secondary region is examined. If the primary region is rejected, neither dependency is called, and the secondary region is not directly read.

If the primary region is accepted but the secondary region is rejected, neither dependency is called. Primary element 2 is not directly read on that path.

Acceptance of both regions enables the ordered pair of dependency calls. F-6218768696f9 receives primary elements 0, 1, and 2 as its first, second, and third explicit arguments, respectively. The values are not transformed for this call by the visible body.

F-d350614e3326 is called with no explicit arguments only after F-6218768696f9 returns normally. No returned value from the first dependency is inspected to decide whether to make the second call. Normal return from the second dependency permits this function to return without a value.

Rejection is therefore distinguishable from acceptance through dependency invocation, not through a returned status.

### Read ordering and value consistency

The primary eligibility evaluation precedes any secondary eligibility evaluation. Both eligibility evaluations precede argument evaluation for the accepted-path call.

No temporal order is established between the two element reads within an individual eligibility expression. Likewise, argument position does not establish the temporal order of the three argument reads. All three arguments must be evaluated before control enters F-6218768696f9.

The primary fields used as arguments are expressed as accesses to the input storage, rather than as explicit saved results from eligibility evaluation. Consequently, the visible function does not establish an atomic snapshot linking the accepted primary values to the later arguments.

With stable ordinary storage, the primary values used for eligibility and the corresponding argument values agree. Concurrent modification, asynchronous modification, and device-backed storage do not acquire defined behavior merely because the function reads the same locations in multiple expressions.

These are source-level ordering and value relationships. They do not specify an exact count of physical loads, prohibit reuse of a previously loaded value, or impose a machine-level memory-access protocol.

### Aliasing and completion

The visible body does not require distinct input regions. Identical pointers and overlapping regions are compatible with its accesses, provided the accessed locations and arithmetic are otherwise valid.

There are no direct writes between the eligibility evaluations. Thus, the function itself introduces no intervening storage mutation that would make overlapping inputs behave differently from stable nonoverlapping inputs with the same field values.

The second dependency is not a guaranteed finalization action. If F-6218768696f9 does not return normally, the visible function provides no independent route to F-d350614e3326. If either dependency fails to return normally, normal completion of this function is likewise not established.

## Side effects

The directly visible operations are integer reads, conditional calls, and return. The body contains no explicit input modification, global assignment, allocation, or release.

That limited observation does not establish a side-effect-free operation. Either dependency may modify global state, access other memory, acquire or release resources, or alter control flow. Their implementations and contracts are not supplied.

The primary and secondary pointers are not passed as explicit arguments to the dependencies. F-6218768696f9 instead receives three integer-valued arguments from the primary region. Those values must not be presumed incapable of identifying memory or resources: their semantic meanings are unknown.

No ownership transfer is explicitly expressed by this function. Equally, the absence of pointer arguments does not prove that either dependency preserves the input storage or any related resource. Ownership effects require a dependency contract that is unavailable here.

There are no direct input accesses after entry into F-6218768696f9 in the visible body. This does not establish what either dependency may access indirectly.

There is no explicit error-reporting or recovery mechanism. Rejection is a normal path without dependency calls. Invalid memory access, invalid arithmetic, and dependency failures are not specified as equivalent to rejection.

## Dependencies

| Opaque ID | Invocation condition | Explicit inputs | Established relationship |
|---|---|---|---|
| F-6218768696f9 | Both regions are accepted | Three integer values corresponding to primary elements 0, 1, and 2 | Its normal return is required before the second call |
| F-d350614e3326 | Both regions are accepted and F-6218768696f9 returns normally | None | Follows the first dependency; its normal return permits normal completion |

Neither dependency’s return value is used by the visible body.

The supplied call sites establish the explicit argument expressions and call ordering, not the dependencies’ complete interfaces. Their parameter declarations, return types, calling conventions, hidden state, global-state preservation, ownership rules, and failure behavior remain undocumented here.

In particular, the first dependency may change state that the second dependency subsequently observes. This function does not preserve or restore such state between the calls.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not results of execution.

Unless stated otherwise, the examples assume 32-bit signed integers, two’s-complement representation, stable ordinary storage, valid accessed locations, and normal dependency return. The ordinary numerical examples use nonnegative element-0 values whose one-bit left shifts remain representable.

### Accepted inputs

The primary region begins with 0, −1, and 7; the secondary region begins with 0 and −1.

Both regions are accepted. F-6218768696f9 receives the three values 0, −1, and 7 in those argument positions. F-d350614e3326 is then called with no explicit arguments.

This example establishes neither the purpose of the value 7 nor any state change performed by the dependencies.

### Primary rejection and an unexamined secondary input

The primary region begins with 0 and 0.

The primary region is rejected. Neither dependency is called. No secondary element or primary third element is directly accessed.

On this path, the body does not require readable secondary storage. That is an access property, not a general promise that arbitrary invalid pointer values are portable interface inputs.

### Secondary rejection

The primary region begins with 0 and −1. The secondary region begins with 0 and 1.

The primary region is accepted, but the secondary region is rejected. Neither dependency is called, and primary element 2 is not directly accessed. Two readable primary elements are sufficient for the direct reads on this path.

### Upper boundary of the unambiguous signed-shift subset

The primary region begins with 1,073,741,823, −1, and 12. The secondary region begins with 0 and −1.

The primary element-0 value doubles to 2,147,483,646, which is representable as a signed 32-bit integer. Both regions are accepted. F-6218768696f9 receives 1,073,741,823, −1, and 12, followed by the second dependency.

This demonstrates that primary element 0 need not be zero for acceptance.

### A bit-vector rejection that is not an unconditional signed-arithmetic case

Suppose the primary region begins with 1,073,741,824 and −1.

Under fixed-width 32-bit bit-vector shift semantics, primary element 0 has bit 30 set, so the primary region is rejected and neither dependency is called.

Under strict signed-shift rules, the doubled positive value is not representable. Rejection is therefore not an unconditional acceptance criterion for this input. The arithmetic model must first be established.

### Identical input regions

Both pointers designate the same three-element region containing 0, −1, and 9.

Both eligibility evaluations succeed. F-6218768696f9 receives 0, −1, and 9, and the second dependency follows. The visible body imposes no distinct-storage requirement.

### Nonreturning first dependency

Both regions satisfy eligibility, but F-6218768696f9 does not return normally.

The contract does not guarantee invocation of F-d350614e3326 or normal return from F-78d399bb1bbd. The second call is conditional on normal completion of the first, not merely on eligibility.

## Uncertainties

**documented_only — Concrete integer and pointer model.**  
Integer width, storage size, representation, alignment, pointer width, and byte order are not independently established. Numerical 32-bit examples remain conditional.

**documented_only — Signed shifting outside the defined subset.**  
The supplied material does not resolve how negative element-0 values or nonrepresentable signed left shifts behave in the actual execution environment. Fixed-width wrapping behavior must not be assumed, and such inputs must not automatically be classified as rejected.

**documented_only — Dependency behavior.**  
No implementation or independent contract establishes either dependency’s state changes, ownership effects, termination, error handling, or preservation of globals.

**documented_only — Mutation and physical accesses.**  
No synchronization, atomicity, volatile-access, or memory-mapped-I/O contract is established. Exact physical read counts and timing are unknown.

**Inferred intent.**  
The structure is consistent with a two-region eligibility gate followed by ordered actions. The field meanings and relationship between those actions remain unknown.

## Evidence

The established static evidence is limited to the supplied function body. It supports conditional examination of two integer regions, short-circuit rejection, three primary-region argument expressions, ordered dependency calls, and absence of a returned value.

The bit-position characterization is a mathematical consequence under the stated fixed-width, two’s-complement interpretation. It is not independent evidence of the target’s arithmetic model.

No dependency implementation, independent target-type definition, calling-convention specification, or runtime trace was supplied. No runtime tests have been performed.
