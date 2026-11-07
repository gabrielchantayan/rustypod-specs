<!-- Trusted provenance: F-ce5e3d6d9ab0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ce5e3d6d9ab0

## Purpose

F-ce5e3d6d9ab0 conditionally processes two descriptor entries using a shared context. It advances a context counter, stores derived identifiers in descriptor fields, and invokes dependencies with fixed numeric selectors.

First-entry processing precedes second-entry processing. During the second path, separate reads of the first-entry field control an additional counter increment and a later group of cross-entry operations. Those conditions can differ if intervening calls or overlapping storage change the field.

“Entry,” “identifier,” “object request,” and “cross-entry operation” describe apparent roles. Their domain meanings are inferred, not established dependency semantics. This contract describes static evidence only; no runtime tests have been performed.

## Interface

### Arguments and result

| Argument | Representation in the supplied material | Use |
|---|---|---|
| Context | Address carried in a 32-bit value | Supplies a mutable counter and is passed to dependencies. |
| Descriptor | Address carried in a 32-bit value | Supplies entry fields and receives identifier stores. |
| Auxiliary value | Opaque 32-bit value | Forwarded unchanged during first-entry processing. |

There is no explicit return value or returned success indicator.

The function performs no visible null, bounds, alignment, or storage-validity checks. Every reached memory access requires valid accessible storage. Dependency calls may impose additional requirements that cannot be established here.

### Accessed layout

Offsets are byte offsets. Each listed field is accessed as a 32-bit quantity.

| Base | Offset | Observable role |
|---|---:|---|
| Context | `0x48` | Counter read and updated during allocation paths. |
| Descriptor | `0x2c` | First-entry presence test and opaque payload source. Read independently at several stages. |
| Descriptor | `0x30` | Second-entry presence test and opaque payload source. |
| Descriptor | `0x34` | Destination of the first identifier store. |
| Descriptor | `0x38` | Destination of the second identifier store. |
| Nonzero result from F-9b3ea8470521 | `0x0c` | Value read for a subsequent F-a2b3e96577b2 call. |

Entry presence means that the relevant 32-bit field compares unequal to zero. No positive-value requirement is visible. Payload values are forwarded without establishing whether they represent pointers, handles, strings, or other data.

### Widths, arithmetic, and aliasing

Counter values, saved identifiers, the auxiliary argument, payloads, and the saved result later passed to F-a2b3e96577b2 are represented as 32-bit quantities. Counter calculations use additions of one; cross-entry arguments also use the saved second-path base plus two.

No overflow check is visible. Arithmetic relationships in this contract apply where the additions are representable. Executable behavior at overflow is **documented_only**; wrapping, trapping, and caller restrictions are not established.

No disjointness check is visible. Context, descriptor, and returned objects are not established to occupy separate storage. A store through one listed field can therefore affect another field if their addresses overlap.

The field descriptions identify direct accesses, not preservation guarantees. In particular, skipping an allocation path means that this function does not perform that path’s direct output-field store. It does not guarantee that the field’s contents remain unchanged: another direct store may alias it, or a dependency may modify it.

## Behavior

### Initial state and conditional output stores

The saved first identifier begins at zero. It is replaced only when the initial first-entry presence test succeeds. Later changes to the descriptor’s first-entry field do not independently update that saved identifier.

Neither descriptor output field is initialized unconditionally:

- A direct store through descriptor offset `0x34` occurs only on the initial first-entry allocation path.
- A direct store through descriptor offset `0x38` occurs only on the second-entry allocation path.

These statements concern stores through those offsets. They do not promise retention of previous contents when a path is skipped, nor persistence of a stored value after dependencies execute or aliased storage is written.

### First-entry path

The first relevant descriptor access reads offset `0x2c`.

If that value is zero, the function skips the first-entry allocation and its dependency calls, then proceeds to the second-entry presence test.

If it is nonzero, the function:

1. Reads the context counter at `0x48`.
2. Forms the counter value plus one and saves it as the first identifier.
3. Stores that identifier at context offset `0x48`.
4. Stores the same identifier at descriptor offset `0x34`.
5. Calls F-9b3ea8470521 with the context.

Both stores precede the dependency call. The first identifier remains the saved arithmetic result even if later writes or dependency effects change either stored field.

If F-9b3ea8470521 returns zero, the function returns immediately. It does not perform the second-entry presence test. There are no compensating stores to undo the preceding updates; however, the dependency itself is not established to preserve those locations.

For a nonzero result, the function uses that result as the first object and makes the following calls in order:

| Dependency | Arguments in order |
|---|---|
| F-c2e9c7166e37 | Context; fresh read of descriptor `0x2c`; saved first identifier |
| F-211638bb75da | First object; selector `0x25`; saved first identifier |
| F-2ea344827b50 | First object; selector `0x78`; saved first identifier; auxiliary value; zero |

The payload read for F-c2e9c7166e37 occurs after the object request. It need not equal the value that passed the initial presence test.

No return value from these three calls controls this path. After they return normally, processing proceeds to the second-entry presence test.

### Second-entry allocation

The function reads descriptor offset `0x30` after the first path has been skipped or completed.

If this read is zero, there is no second-entry allocation or further dependency call.

If it is nonzero, the function:

1. Reads the current context counter and saves it as the second-path base.
2. Forms the base plus one and saves it as the second identifier.
3. Stores the second identifier at context offset `0x48`.
4. Stores the second identifier at descriptor offset `0x38`.
5. Reads descriptor offset `0x2c` again.
6. If this read is nonzero, rereads the current context counter, adds one, and stores the result back at context offset `0x48`.
7. Calls F-9b3ea8470521 with the context.

The conditional extra increment does not directly store either descriptor output field. With overlapping storage, its counter store may nevertheless affect descriptor data.

The second-path base is whatever counter value is read when this path begins. It is not assumed to equal the value previously stored by the first path. Earlier dependency effects can change it.

If the second object request returns zero, the function makes no remaining calls. It performs no rollback. The values actually present afterward remain subject to dependency and aliasing effects.

### Second-entry operations

When the second request returns nonzero, that result becomes the second object. The following calls occur in order:

| Dependency | Arguments in order | Result use |
|---|---|---|
| F-c2e9c7166e37 | Context; fresh read of descriptor `0x30`; saved second identifier | Ignored |
| F-211638bb75da | Second object; selector `0x25`; saved second identifier | Ignored |
| F-211638bb75da | Second object; selector `0x7b`; saved second identifier | Saved as an opaque 32-bit value |
| F-2ea344827b50 | Second object; selector `0x2f`; zero; saved second identifier; zero | Ignored |
| F-a2b3e96577b2 | Second object; saved opaque result; fresh 32-bit read at second-object offset `0x0c` | Ignored |

The second-entry payload is reread after the object request. The object field at `0x0c` is read after the selector-`0x2f` call, not when the object is first obtained.

The selector-`0x7b` result is forwarded without a visible success or validity check.

### Conditional cross-entry operations

After the preceding F-a2b3e96577b2 call returns, the function rereads descriptor offset `0x2c`.

If the latest value is zero, processing ends. If it is nonzero, the following calls occur:

| Dependency | Arguments in order | Result use |
|---|---|---|
| F-2ea344827b50 | Second object; selector `0x4e`; saved first identifier; saved second identifier; second-path base plus two | Ignored |
| F-211638bb75da | Second object; selector `0x7b`; saved first identifier | Saved as an opaque 32-bit value |
| F-887ba2404a49 | Second object; selector `0x2f`; `0xffffffff`; second-path base plus two | Ignored |
| F-a2b3e96577b2 | Second object; saved opaque result; fresh 32-bit read at second-object offset `0x0c` | Ignored |

The all-ones argument is a 32-bit bit pattern. Its intended signed meaning is unknown.

Both occurrences of “second-path base plus two” derive from the saved base, not from a fresh counter read. The final object-field read follows F-887ba2404a49 and can observe its effects.

The presence test controlling the extra counter increment and the presence test controlling these calls are separate observations. Neither implies the other. Cross-entry calls can use a saved first identifier of zero if the first entry was initially absent but is present at the final test.

## Side effects

### Direct memory effects

The direct store destinations are context `0x48`, descriptor `0x34`, and descriptor `0x38`. No unconditional clearing of output fields is visible.

Stores commit before the corresponding object-request result is checked. There is no visible rollback mechanism. This establishes the ordering and absence of compensating stores, not that stored values survive dependency execution.

For example, with disjoint context and descriptor storage, a first-path dependency can modify descriptor `0x38` while leaving descriptor `0x30` zero. The second path is then skipped, yet the second output field has changed. That behavior is compatible with the function’s direct-store contract.

### Dependency effects and ownership

Dependency side effects are unknown. This contract does not assume that calls preserve counters, descriptor fields, returned-object contents, global state, or memory lifetimes.

F-9b3ea8470521 produces a value used as an object address when nonzero. This does not establish allocation, ownership transfer, or a release obligation. No explicit release or reference-count action is identifiable in this function.

Likewise, forwarding a descriptor payload does not establish borrowing, copying, retention, or consumption. Safe execution requires storage to remain valid for every later reached access, but the complete lifetime obligations require dependency contracts.

### Errors and termination

The only explicitly checked failure indication is a zero result from F-9b3ea8470521:

- On the first path, zero causes immediate return before the second-entry test.
- On the second path, zero suppresses all remaining calls.

No explicit error status is returned. Other dependency results are ignored or forwarded without validation.

Invalid memory, arithmetic overflow, abnormal dependency termination, and dependency-specific errors are **documented_only**. No recovery behavior for them is established.

## Dependencies

| Opaque ID | Established local use |
|---|---|
| F-9b3ea8470521 | Receives context. Its result is tested against zero and otherwise used as an object address and argument. |
| F-c2e9c7166e37 | Receives context, a fresh descriptor payload, and a saved identifier. |
| F-211638bb75da | Receives an object, selector, and identifier. Selector-`0x25` results are ignored; selector-`0x7b` results are saved and forwarded. |
| F-2ea344827b50 | Receives an object, selector, and three additional values. Selectors used here are `0x78`, `0x2f`, and `0x4e`. |
| F-887ba2404a49 | Receives the second object, selector `0x2f`, a 32-bit all-ones value, and second-path base plus two. |
| F-a2b3e96577b2 | Receives an object, a saved dependency result, and a freshly read object field. |

Descriptions such as “emission” or “patching” are inferred intent only. Encoding, allocation, mutation, ownership, and error semantics are not specified by the visible call sites.

## Acceptance examples

These are conditional static expectations, not executed tests.

Unless an example explicitly introduces mutation, assume valid disjoint storage, no concurrent modification, representable arithmetic, normally returning dependencies, and no dependency changes to the context counter or descriptor fields. Preservation and final-value expectations below depend on those assumptions.

### Neither entry present

With both presence fields zero:

- No dependencies are called.
- No direct stores occur.
- The counter and output fields retain their previous contents under the stated storage and concurrency assumptions.

### First entry only

With counter `10`, first entry nonzero, second entry zero, and a successful object request:

- Counter and descriptor `0x34` receive `11` before the request.
- First-path calls use saved identifier `11`.
- The auxiliary value is forwarded unchanged with selector `0x78`.
- No direct store through descriptor `0x38` occurs.
- Under the example assumptions, descriptor `0x38` is unchanged and the final counter is `11`.

### First request returns zero

With counter `10` and a nonzero first entry:

- Counter and descriptor `0x34` receive `11` before the request.
- A zero request result causes immediate return.
- No subsequent dependency calls or second-entry test occur.
- Under the example assumptions, the two stored values remain `11`; the function performs no rollback.

### Second entry only

With counter `10`, first entry zero, second entry nonzero, and a successful request:

- Counter and descriptor `0x38` receive `11`.
- No direct store through descriptor `0x34` occurs.
- Second-entry operations use identifier `11`.
- No extra increment or cross-entry calls occur.
- Under the example assumptions, descriptor `0x34` retains its prior value.

### Both entries present

With counter `10`, both entries nonzero, and both requests successful:

- The first identifier is `11`.
- The second-path base is `11`; the second identifier is `12`.
- The extra increment stores `13` before the second request.
- Descriptor outputs are `11` and `12`.
- Cross-entry calls use saved identifiers `11` and `12`, and derived value `13`.

If only the second request returns zero, the first-path calls have already occurred, but no second-entry operation calls occur. Under the example assumptions, the final counter is `13` and outputs remain `11` and `12`.

### Skipped output store does not imply preservation

Start with a nonzero first entry and zero second entry. Keep storage disjoint, but allow a first-path dependency to change descriptor `0x38` while leaving `0x30` zero.

The second path is skipped and no direct store through `0x38` occurs. Nevertheless, `0x38` contains the dependency-written value. A requirement that it retain its entry value would incorrectly impose a dependency-preservation guarantee.

### Independent later presence tests

Start with the first entry zero and the second entry nonzero. If the first-entry field becomes nonzero during the successful second request:

- The earlier presence test did not cause an extra increment.
- The saved first identifier remains zero.
- If the final presence test is nonzero, cross-entry calls occur using that zero identifier.

Conversely, an earlier nonzero test can cause the extra increment even if a dependency later clears the field and suppresses cross-entry calls.

## Uncertainties

The following remain **documented_only**:

- Actual calling convention, pointer model, alignment requirements, and executable overflow behavior.
- Domain meaning of payloads, selectors, identifiers, and the additional counter value.
- Dependency mutation, global-state, ownership, lifetime, and error contracts.
- Whether valid callers permit overlapping storage or concurrent access.
- Whether object offset `0x0c` denotes a position, size, address, or something else.

No explicit synchronization is visible. Thread safety and reentrancy are not established.

## Evidence

Established static evidence consists of the represented field widths and offsets, direct-store order, repeated presence and payload reads, saved arithmetic operands, dependency-call order and arguments, zero-result branches, and absence of visible rollback or a returned status.

The output-field contract is deliberately limited to direct stores. It does not infer preservation from a skipped path or persistence from a completed store.

Identifier allocation, object requests, operation emission, and cross-entry linking are inferred interpretations of usage. They do not extend the established behavior.

Only the supplied static representation was examined. No dependency implementations were examined, and no execution, runtime tracing, or tests were performed.
