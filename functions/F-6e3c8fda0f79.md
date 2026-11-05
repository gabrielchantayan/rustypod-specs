<!-- Trusted provenance: F-6e3c8fda0f79; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6e3c8fda0f79

## Purpose

F-6e3c8fda0f79 selects between two delivery paths using an 8-bit mode value.

- **Zero mode:** derives a temporary representation from a nonzero preparation input, submits that representation to a dependency, and delivers a temporary wrapper only if submission returns a nonzero result.
- **Nonzero mode:** bypasses preparation and submission, delivering a temporary wrapper around a separately supplied nonzero direct input.

Every path begins with a one-byte write of 1 at context offset 13, including paths that subsequently return without calling any dependency.

“Preparation,” “submission,” “wrapper,” and “finalization” describe inferred roles. The established contract is the accesses, arguments, branch conditions, and call ordering described below. The function does not independently establish allocation, ownership transfer, copying, retention, or release semantics.

## Interface

The function accepts five arguments and has no explicit return value.

| Argument | Represented width | Established use |
|---|---|---|
| Context | Address in a 32-bit address model | Destination of the entry byte store; source of the receiver used for indirect delivery |
| Preparation input | 32 bits | Selected only in zero mode; tested for zero and passed to dependencies |
| Direct input | 32 bits | Selected only in nonzero mode; tested for zero and passed to wrapper construction |
| Mode | 8 bits | Zero selects preparation and submission; every nonzero value selects direct delivery |
| Submission argument | 32 bits | Forwarded unchanged to submission in zero mode |

The two inputs are represented as 32-bit integer values. Whether dependencies interpret them as pointers, handles, identifiers, or another kind of value is **documented_only**. Neither selected input is checked for positivity: every nonzero bit pattern passes its input check.

### Context and dispatch layout

The function directly accesses these context locations:

| Byte offset | Access width | Use |
|---|---|---|
| 0 | 32 bits | Receiver address for indirect delivery |
| 13 | 8 bits | Written with 1 at entry |

The receiver’s first 32-bit word supplies a dispatch-table address. A callable address is obtained from byte offset 288, hexadecimal 0x120, within that table. The callable receives the receiver address and the address of the selected temporary wrapper.

The dispatch lookup occurs only on a delivery path, after wrapper construction returns. The function does not establish a complete context, receiver, or dispatch-table layout.

For normal execution, the entry byte must be writable. Delivery additionally requires readable context and dispatch storage and a callable target compatible with the observed invocation. No explicit guards validate those requirements.

### Temporary storage

Three local byte-storage regions are represented:

- A 236-byte preparation region.
- A 24-byte wrapper region for zero-mode delivery.
- A separate 24-byte wrapper region for nonzero-mode delivery.

These sizes describe storage made available to dependencies, not established representation formats. The function performs no explicit initialization of these regions before passing them to dependencies. It does not establish which bytes dependencies initialize or later read.

All offsets above are byte offsets. Address calculations and pointer-sized values are represented in a 32-bit model. The supplied material does not establish byte order, alignment requirements, or defined behavior for invalid or overflowing address calculations.

## Behavior

### Common entry effect

Before testing the mode or either input, the function writes the byte value 1 at context offset 13.

Consequently, even an otherwise empty operation requires a valid destination for this write. A zero selected input does not bypass the store, and there is no explicit null-context check.

The store establishes an entry-time effect, not a guaranteed final value. Subsequent calls can modify shared or aliased storage. The function does not repeat the store before returning.

### Zero mode: preparation and submission

When the mode byte is zero, the preparation input controls the initial early return. The direct input does not affect this branch.

A zero preparation input causes return immediately after the entry store. No dependency is called, no submission state is read, and no receiver lookup occurs.

For a nonzero preparation input, F-7a47d707217e receives that input. Its 32-bit result is classified only as zero or nonzero. F-70d12dc217e6 then receives:

- A 32-bit selector: 0 for a zero classification result, otherwise 4.
- The original preparation input.
- The address of the 236-byte preparation region.

No other classification-result bits are forwarded as the selector. In particular, a result with its high bit set still selects 4.

After F-70d12dc217e6 returns, the function reads a shared pointer and then reads a 32-bit field at byte offset 48, hexadecimal 0x30, from the addressed storage. F-8b99cba237b2 receives that field value, the preparation-region address, a literal zero, and the original submission argument, in that argument order.

The shared-state lookup is after preparation, not an entry-time snapshot. If earlier dependencies change the shared pointer or its target field, the lookup uses the state present when it occurs. No preservation of that state by an unknown dependency is assumed.

F-8b99cba237b2 produces a 32-bit result. The only local decision based on that result is whether it equals zero.

#### Zero submission result

A zero result causes F-58ba61416cc9 to receive the original preparation input, followed by return.

This path performs no wrapper construction, receiver lookup, indirect delivery, or wrapper finalization. The function supplies no diagnostic or separate error result. Whether zero denotes failure, absence, rejection, or another outcome is **documented_only**.

#### Nonzero submission result

For every nonzero submission result, F-ba5a4c1e0746 receives the zero-mode 24-byte wrapper region, the submission result, and a literal zero.

After that call returns, the function obtains the receiver and dispatch information from the context and invokes the selected callable with the receiver and wrapper address.

On normal return from delivery, F-58ba61416cc9 receives the original preparation input. Only after that call returns does F-4e292a3dd745 receive the zero-mode wrapper address.

Thus the established normal-return ordering is:

- Wrapper construction before indirect delivery.
- Indirect delivery before the preparation-input operation.
- The preparation-input operation before wrapper finalization.

No positivity or range restriction applies to the submission result beyond its nonzero test. A result of 0x80000000 takes this delivery path.

### Nonzero mode: direct delivery

Every nonzero mode byte selects the same direct-delivery branch. The function does not distinguish mode 1 from mode 255.

A zero direct input causes return immediately after the entry store. The preparation input and submission argument cannot prevent that return.

For a nonzero direct input, F-ba5a4c1e0746 receives the direct-mode 24-byte wrapper region, the original direct input, and a literal zero. After construction returns, the function obtains the receiver and dispatch information and invokes the selected callable with the receiver and wrapper address.

After normal return from delivery, F-4e292a3dd745 receives the direct-mode wrapper address.

This branch does not call F-7a47d707217e, F-70d12dc217e6, F-8b99cba237b2, or F-58ba61416cc9. It does not perform the submission shared-state lookup.

### Receiver lookup timing

Neither branch captures the receiver at entry. Receiver and dispatch accesses used for delivery occur after F-ba5a4c1e0746 returns. If wrapper construction changes the context’s receiver field, and no later change intervenes, delivery uses the changed state.

The lookup has a logical dependency chain: context receiver, receiver dispatch-table address, then callable slot. The supplied expression does not establish an atomic snapshot across these accesses or a complete evaluation order between the callable expression and its receiver argument. No consistency guarantee under concurrent or asynchronous mutation is established.

### Errors and abnormal control flow

There is no explicit error return, retry, fallback, or diagnostic.

The explicit checks distinguish only:

- Zero from nonzero mode.
- Zero from nonzero selected input.
- Zero from nonzero classification result.
- Zero from nonzero submission result.

There are no explicit checks for an invalid receiver, invalid dispatch table, null callable, unsuccessful wrapper construction, or unsuccessful finalization.

All subsequent-call guarantees are conditional on earlier calls returning normally. If a dependency or indirect callable faults, terminates execution, or otherwise does not return, the supplied body establishes no exceptional cleanup guarantee.

## Side effects

### Direct accesses and delegated effects

Directly established effects comprise:

- The entry byte store at context offset 13.
- Conditional reads of the shared pointer and its target’s offset-48 field.
- Conditional receiver, dispatch-table, and callable-slot reads.
- Passing local preparation and wrapper addresses to dependencies.
- Passing the selected wrapper address to the indirect callable.

The function contains no explicit heap allocation or deallocation and no explicit write to the context’s receiver field. Such effects may occur inside dependencies; their absence from this body does not exclude them.

### Ownership and lifetime

The preparation and wrapper regions are local storage belonging to this invocation. Their addresses are not established as valid beyond its lifetime. The source does not prove that callees copy the contents rather than retain an address, nor does it establish that retaining such an address would be safe.

For zero mode with a nonzero preparation input, F-58ba61416cc9 is called exactly once if all preceding calls return normally:

- After submission when its result is zero.
- After delivery when its result is nonzero.

This pattern is consistent with input cleanup, but release or consumption semantics remain inferred.

Each delivery path calls F-4e292a3dd745 once after its preceding operations return normally. Its placement is consistent with wrapper finalization, but the resources it affects are unknown.

There is no corresponding F-58ba61416cc9 call for the direct input. This asymmetry does not establish that the direct input is borrowed or that the preparation input is owned.

### Aliasing and visibility

The entry store can affect storage aliased by an input, shared state, or another object. Its ordering alone does not establish that any dependency reads that byte, or that 1 remains there until a dependency reads it.

For example, F-7a47d707217e may overwrite aliased storage before F-70d12dc217e6 runs. The later dependency’s reads are themselves unknown. The contract therefore establishes the initial write and its position in the sequence, not guaranteed observation of its value by another operation.

Likewise, dependencies may change the receiver or submission state before the corresponding later lookup. No dependency is assumed to preserve globals, context fields, or aliased storage.

## Dependencies

| Opaque function ID | Established interaction | Unresolved semantics |
|---|---|---|
| F-7a47d707217e | Receives nonzero preparation input; 32-bit result reduced to zero/nonzero | Classification meaning and side effects |
| F-70d12dc217e6 | Receives selector 0 or 4, preparation input, and 236-byte region address | Representation layout, initialization, retention, ownership |
| F-8b99cba237b2 | Receives shared-state field, preparation-region address, zero, and submission argument; returns a 32-bit result | Submission meaning, result ownership, error interpretation |
| F-ba5a4c1e0746 | Receives a 24-byte wrapper-region address, a nonzero value, and zero | Wrapper format, writes, retention, failure behavior |
| F-58ba61416cc9 | Receives preparation input on both normally completed submission-result paths | Whether it releases, consumes, or otherwise changes anything |
| F-4e292a3dd745 | Receives the selected wrapper address after delivery and any zero-mode input operation | Finalization behavior and affected resources |

The runtime-selected indirect callable has no supplied opaque function ID. Its established interface here consists of the receiver and wrapper-address arguments. Its implementation, retention behavior, global effects, and memory ownership rules remain unknown.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless stated otherwise, they assume valid accessed storage and normal return from every invoked operation.

1. **Zero mode with zero preparation input.**  
   Context offset 13 receives the byte 1, followed by return. No dependency or receiver lookup occurs, even if the direct input is nonzero.

2. **Nonzero mode with zero direct input.**  
   The entry store occurs, followed by return. A nonzero preparation input does not trigger preparation, submission, or F-58ba61416cc9.

3. **Zero classification and zero submission result.**  
   With nonzero preparation input, classification precedes preparation. F-70d12dc217e6 receives selector 0. Submission uses the shared-state field read after preparation. F-58ba61416cc9 follows the zero submission result; no wrapper or dispatch operation occurs.

4. **Nonzero classification and nonzero submission result.**  
   Preparation receives selector 4 regardless of the particular nonzero classification value. The submission result is wrapped and delivered. F-58ba61416cc9 follows delivery and precedes F-4e292a3dd745.

5. **Mode byte 255 with nonzero direct input.**  
   The direct input is wrapped, delivered, and followed by wrapper finalization. Preparation, submission, shared-state lookup, and F-58ba61416cc9 are absent.

6. **Submission result 0x80000000.**  
   The nonzero result is delivered. Its signed interpretation does not create a rejection branch.

7. **Preparation replaces the shared pointer.**  
   If F-70d12dc217e6 replaces the shared pointer and that replacement remains current until lookup, submission receives the offset-48 field from the replacement target, not an entry-time target.

8. **Wrapper construction changes the receiver.**  
   If F-ba5a4c1e0746 changes the receiver field and it remains stable during subsequent dispatch accesses, delivery uses the changed receiver.

9. **Aliased entry byte is overwritten.**  
   If classification overwrites storage containing the entry byte, no requirement says preparation must observe 1. What remains established is that the entry store occurred before classification; preparation’s actual reads and the surviving byte value are unknown.

10. **Delivery does not return normally.**  
    The function establishes no guarantee that the later zero-mode input operation or wrapper finalization occurs.

## Uncertainties

The following remain **documented_only**:

- Semantic types of the two inputs and submission argument.
- Full object layouts, valid address ranges, alignment, and byte order.
- Meaning and persistence of the entry byte value.
- Preparation and wrapper formats and initialization requirements.
- Dependency reads, writes, global preservation, copying, retention, allocation, release, and ownership transfer.
- Meaning of zero submission results and handling of dependency-internal failures.
- Permitted aliasing, reentrancy, synchronization, thread safety, and dispatch consistency during mutation.
- Behavior for invalid addresses, address-calculation overflow, or abnormal control flow.
- Precise ABI details beyond the represented widths and observed arguments.

These gaps do not weaken the established local branch conditions or normal-return call ordering. They prevent stronger claims about resource management, successful delivery, or final shared-state contents.

## Evidence

Established static evidence consists of the supplied function body’s access widths, byte offsets, local storage extents, branch tests, dependency arguments, indirect-call structure, and normal-return ordering. Dependency IDs are taken from the supplied opaque mapping.

Role descriptions such as preparation, submission, wrapper construction, input cleanup, and finalization are inferred intent, not independently verified dependency contracts. No dependency implementation was supplied.

No runtime tests have been performed. The acceptance examples describe consequences of the supplied static representation under their stated conditions; they do not report observed runtime execution.
