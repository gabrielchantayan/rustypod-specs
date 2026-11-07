<!-- Trusted provenance: F-c95faceeb5b7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c95faceeb5b7

## Purpose

F-c95faceeb5b7 saves four incoming register values into a selected memory region, conditionally invokes one dependency, publishes a global value, executes an ordered dependency sequence, performs two groups of calls with fixed scalar arguments, and enters an indefinite wait-for-interrupt loop.

The established contract concerns memory accesses, call order, argument values, retained values, and control flow. The scalars 768 and 2048 have no established unit. Their use does not establish byte counts, allocation requests, or resource sizes. Likewise, the subsequent four-argument calls are not established configuration submissions.

Startup, resource provisioning, and eventual idling are plausible interpretations, not dependency semantics established by the supplied representation.

## Interface

### Entry state and termination

The function has no declared parameters and no declared return value. Its behavior nevertheless depends on four incoming 32-bit register values:

- One incoming argument-register value.
- Two preserved-register values.
- The entry link-register value.

Each is saved as an uninterpreted four-byte word. The function does not validate these values or require them to represent pointers.

The represented data and address operations use a 32-bit model. Context fields participate in signed 32-bit integer arithmetic; the other saved and returned words have no established signed interpretation.

There is no ordinary return path. Once the final loop is reached, normal continuation from each wait operation leads to another wait operation. A dependency that fails to return, faults, or transfers control elsewhere may prevent later operations.

### Memory roles

These are descriptive roles, not source identifiers or additional parameters.

| Role | Established use |
|---|---|
| Preferred-endpoint global | Appears in an initial endpoint load and a separate null-test expression. |
| Fallback-endpoint global | Supplies the endpoint only when the null-test expression observes zero. |
| Selected save region | Receives four four-byte writes immediately before the selected endpoint; its lowest-address word is overwritten twice later. |
| Publication source and destination | A four-byte global value is read and written to another location after the conditional stage. |
| Static-data location | Its address is passed to F-56c0ec146ae2 with 31. |
| Context object | Returned by F-4ee7f1581707; four-byte fields at offsets 228 and 224 are read. |
| Result-pointer global | Appears in a retained-pointer load and a separate pointer expression used for the initial result write. |
| Retained result cell | The cell addressed by the retained pointer; read twice during the first call group. |
| Supplemental globals | Two four-byte values, each copied to the lowest-address word of the selected save region before its corresponding four-argument call. |

Distinct roles do not establish distinct storage. The supplied material warns of overlapping global symbols. Aliasing can affect subsequent reads, including reads otherwise described as obtaining values previously written.

### Access prerequisites

Reaching the final loop without an access failure requires, at each relevant operation:

- Writable four-byte locations at offsets −4, −8, −12, and −16 from the selected endpoint.
- Readable global sources and a writable publication destination.
- A context result supporting the field reads at offsets 224 and 228.
- A writable cell through the pointer expression used for the initial result store.
- A readable cell through the separately retained result pointer when its two reads occur.
- Dependencies and the wait operation to support the represented continuation.

The initial-store destination and retained result cell need not be the same. Exact alignment, address validity, and memory-access requirements are not established. These prerequisites are not checks implemented by this function.

## Behavior

### 1. Select an endpoint using two represented global accesses

The function first loads the preferred-endpoint global and retains that value as the prospective endpoint. It then separately uses the same global in a null-test expression.

- If the null-test expression observes zero, the fallback-endpoint global is read and replaces the retained endpoint.
- Otherwise, the initially loaded endpoint remains selected.

The nonzero branch does not select the value observed by the null test; it retains the earlier value. Under stable memory, these observations agree, yielding the familiar rule that a nonzero preferred endpoint is used and zero selects the fallback. Without that stability condition, the observations must not be collapsed.

For example, the first observation could be zero while the null-test observation is nonzero. In that case, the represented branch retains zero as the endpoint and does not read the fallback. Conversely, a nonzero initial observation is discarded if the null-test expression observes zero.

These are distinct expressions in the supplied representation, not proof of two distinct physical machine loads or a particular concurrent-memory model.

### 2. Save four incoming words

Using the selected endpoint, four four-byte writes occur in this order:

1. Entry link-register value at offset −4.
2. One preserved-register value at offset −8.
3. The other preserved-register value at offset −12.
4. Incoming argument-register value at offset −16.

The endpoint denotes the end of the save region. The writes occupy the preceding 16 bytes.

The address of the lowest word, at offset −16, is retained for two later writes. Subsequent changes to either endpoint global do not trigger reselection. No restoration of the saved values appears, and the lowest word is later overwritten twice.

### 3. Execute the conditional stage and publication

F-a3aed6337812 is called once. Its integer result controls one branch:

- Zero omits F-40ce7199841c.
- Any nonzero result invokes F-40ce7199841c once.

If the applicable calls return, both paths continue by reading the publication-source global and writing its four-byte value to the publication destination.

The branch does not establish a success or failure convention. The publication read occurs after the conditional calls, so their effects on its source may affect the published value.

### 4. Execute the ordered dependency sequence

Following the publication write, these calls occur in order:

1. F-6e491bc8f77e.
2. F-8c9069b715c9.
3. F-55d62c5e6969.
4. F-e5b930f22dc7.
5. F-654cb948a147.
6. F-89cd9591c5d7.
7. F-676d14c54b98.
8. F-56c0ec146ae2, with explicit arguments 31 and the address of the static-data location.
9. F-bdff51d0df41.

No explicit result checks intervene. Passing the static-data address establishes neither its layout nor whether the dependency reads, writes, or retains it.

### 5. Read context fields and pass their sum

F-4ee7f1581707 is called. Its result supplies the base for two four-byte signed-integer field reads, at byte offsets:

- 228, hexadecimal 0xE4.
- 224, hexadecimal 0xE0.

Their sum is passed as the sole explicit argument to F-unmapped-01.

The two reads occur in the same addition expression. Their relative order is not established. Both precede F-unmapped-01 and follow the context-producing call.

The represented arithmetic is signed 32-bit addition. There is no explicit widening, overflow check, or range check. For sums within the signed 32-bit range, the mathematical sum determines the argument. For sums outside that range, the supplied representation does not establish portable signed-arithmetic behavior; target-level wrapping must not be assumed.

There is also no explicit context null check or address-overflow check before the field accesses.

Execution then continues through:

1. F-f1d4263e4fae.
2. F-ebf08b3004bd.
3. F-unmapped-02.
4. F-d1d7ff259363.

No explicit result checks are shown.

### 6. Execute the first call group

The first group uses the scalar 768, hexadecimal 0x300. Its represented sequence is:

1. Call F-d62ebd393b90 and retain its four-byte result.
2. Load the result-pointer global and retain that pointer.
3. Use a separate expression involving the result-pointer global to write the retained result through the pointer supplied by that expression.
4. Call F-unmapped-03 with 768 and retain its four-byte result.
5. Read the first supplemental global and write that value to the retained lowest-word address of the save region.
6. Read the four-byte value at the retained result pointer.
7. Call F-b50b901b67cb with that read value, zero, the result from F-unmapped-03, and 768.
8. Read the four-byte value at the same retained result pointer again.
9. Call F-7aad9d988c5b with this newly read value.

The pointer retained in step 2 and the pointer supplied by step 3 must not be assumed equal without a stability condition. The representation contains separate global expressions; it does not prove separate physical loads. If their values differ, the initial store targets a different cell from the later reads.

After step 2, changing the result-pointer global alone does not redirect either later read. Both use the retained pointer. Changes to the pointed-to memory remain relevant.

Neither later read is guaranteed to recover the initially stored result. F-unmapped-03 may affect the retained cell. The supplemental write may also affect it through aliasing. F-b50b901b67cb may change it before the second read.

The supplemental write is not one of the four explicit arguments to F-b50b901b67cb. Whether it supplies additional call data through an ABI or another mechanism remains unknown. No result from this first F-7aad9d988c5b call is subsequently consumed.

### 7. Execute the second call group

The second group uses the scalar 2048, hexadecimal 0x800:

1. Call F-d62ebd393b90 again and retain its four-byte result.
2. Call F-unmapped-03 with 2048 and retain its four-byte result.
3. Read the second supplemental global and write that value to the retained lowest-word address of the save region.
4. Call F-b50b901b67cb with the retained F-d62ebd393b90 result, 64, the result from F-unmapped-03, and 2048.
5. Call F-7aad9d988c5b with the retained F-d62ebd393b90 result.
6. Retain this F-7aad9d988c5b return value for the final loop.

Unlike the first group, there is no explicit publication of the second F-d62ebd393b90 result through the result-pointer global and no read through the retained result pointer to obtain either call argument.

The supplemental write can still affect aliased storage. The absence of an explicit second result-cell store does not guarantee that the cell remains unchanged.

Neither F-unmapped-03 result is checked before being passed onward. The scalars 768 and 2048 are arguments, not established byte measurements.

### 8. Wait indefinitely

The final F-7aad9d988c5b result is used as the represented operand of a repeated wait-for-interrupt operation.

There is no exit condition or operand refresh. If an operation completes and execution continues normally, another wait follows with the same retained operand. Earlier calls and memory operations are not repeated.

The representation does not establish wake-up conditions, interrupt eligibility, scheduling behavior, or the architectural meaning of this operand.

## Side effects

The direct writes are:

- Four ordered incoming-register saves.
- One publication write.
- One write through the separate result-pointer-global expression.
- Two supplemental writes to the retained lowest-word address.

Aliasing may cause these writes to affect other named roles. Consequently, neither global independence nor preservation of earlier stored values is guaranteed.

Dependencies may introduce further memory effects, resource changes, or control transfers. No dependency is assumed to preserve globals, allocate memory, retain ownership, or release resources.

There is no explicit release, rollback, restoration, cleanup, or ordinary return. This does not establish a resource leak because allocation and ownership semantics are unknown.

There are no local branches handling invalid pointers, context arithmetic overflow, zero F-unmapped-03 results, or dependency-reported failures. If a dependency returns an error-shaped value, the function proceeds according to the ordinary dataflow unless that value is the initial zero/nonzero branch result.

No atomicity, synchronization, or memory-ordering guarantee is established.

## Dependencies

All identities below are opaque. “No explicit arguments” does not establish independence from machine state or global memory.

| Dependency | Established interaction |
|---|---|
| F-a3aed6337812 | Called once; zero/nonzero result controls F-40ce7199841c. |
| F-40ce7199841c | Called only on the nonzero branch. |
| F-6e491bc8f77e, F-8c9069b715c9, F-55d62c5e6969, F-e5b930f22dc7, F-654cb948a147, F-89cd9591c5d7, F-676d14c54b98 | Ordered calls after publication. |
| F-56c0ec146ae2 | Receives 31 and a static-data address. |
| F-bdff51d0df41 | Follows F-56c0ec146ae2. |
| F-4ee7f1581707 | Supplies the context base. |
| F-f1d4263e4fae, F-ebf08b3004bd, F-d1d7ff259363 | Ordered calls surrounding F-unmapped-02. |
| F-d62ebd393b90 | Called twice; produces the values used differently by the two call groups. |
| F-b50b901b67cb | Receives two four-argument calls; no return value is consumed. |
| F-7aad9d988c5b | Called after each four-argument call; only its second result is retained for the loop. |

Three targets lack mapped identities. These document-local IDs express no additional semantics:

| Document-local ID | Established interaction |
|---|---|
| F-unmapped-01 | Receives the context-field sum. |
| F-unmapped-02 | Called between F-ebf08b3004bd and F-d1d7ff259363. |
| F-unmapped-03 | Called with 768 and then 2048; each result becomes the third explicit argument of the corresponding F-b50b901b67cb call. |

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless stated otherwise, they assume valid accessed storage and returning dependencies.

1. **Stable preferred endpoint.** If both preferred-endpoint expressions observe the same nonzero address, that address selects the save region. The fallback is not read for selection.

2. **Different endpoint observations.** If the initial endpoint load obtains address A but the null-test expression observes nonzero B, A remains selected. If the null-test expression instead observes zero, the fallback value replaces A.

3. **Unchecked fallback.** If the null-test expression observes zero and the fallback supplies zero, there is no further null rejection before the preceding-address writes. Successful accesses are not promised.

4. **Four saved values.** The first four writes store the link-register value, the two preserved-register values, and the incoming argument-register value at offsets −4, −8, −12, and −16 respectively.

5. **Conditional stage.** A zero F-a3aed6337812 result omits F-40ce7199841c. Any nonzero result includes it before publication.

6. **In-range arithmetic.** If the context-field reads obtain 16 and 32, F-unmapped-01 receives 48. This does not require a particular field-read order.

7. **Different result-pointer observations.** If the retained-pointer load obtains address A and the initial-store expression supplies address B, the first producer result is written at B, while both later result-cell reads use A. No automatic copy from B to A occurs.

8. **Value changed before the first cell read.** If F-unmapped-03 changes the retained cell to B, and the supplemental write and all other intervening activity leave that cell unchanged, the first F-b50b901b67cb call receives B as its first argument. Without that preservation condition, it receives the value present at the actual read.

9. **Supplemental alias.** If the retained cell coincides with the lowest save-region word, the supplemental write can replace B before that read. The argument reflects the resulting cell contents, not necessarily B or the original producer result.

10. **Value changed during the four-argument call.** If the first F-b50b901b67cb call leaves C in the retained cell and nothing changes it before the following read, F-7aad9d988c5b receives C. This places no requirement on the cell value at entry to F-b50b901b67cb.

11. **Zero scalar-call result.** If either F-unmapped-03 call returns zero, zero is passed as the corresponding third explicit argument. There is no zero-result branch.

12. **Second group and repeated waiting.** If the second producer result is D, the second four-argument call receives D, 64, the second F-unmapped-03 result, and 2048. The following F-7aad9d988c5b receives D. If it returns W and each wait continues normally, waits repeat with W without rerunning either call group.

## Uncertainties

The following remain **documented_only**:

- Exact addresses, alignment, byte order, and alias relationships.
- Whether repeated global expressions produce distinct physical loads, and how concurrent changes would be observed.
- Whether the save region is architectural stack storage.
- The meanings and layouts of the static data, publication state, supplemental values, and context fields.
- Valid-use ranges and overflow behavior for the signed context-field addition.
- Dependency failure conventions, global effects, ownership, and resource lifetimes.
- Whether either scalar passed to F-unmapped-03 denotes a size, and any associated unit.
- Whether F-unmapped-03 allocates anything or F-b50b901b67cb configures anything.
- Whether supplemental writes carry additional ABI-level arguments.
- Hardware interpretation and wake-up behavior of the final wait operation.

## Evidence

**Established static evidence:** the supplied representation supports the separate endpoint expressions, four register saves, conditional call, publication, ordered dependency calls, context-field reads, separate result-pointer expressions, differing call-group dataflows, supplemental writes, and indefinite final loop.

**Inferred intent:** initialization followed by idling; possible resource-related meanings for the two call groups. These interpretations do not strengthen the observable contract.

**Evidence limits:** this document uses only the supplied static representation and dependency mapping. No independent implementation inspection, runtime tests, hardware observations, or physical-load verification have been performed.
