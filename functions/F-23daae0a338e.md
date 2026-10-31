<!-- Trusted provenance: F-23daae0a338e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-23daae0a338e

## Purpose

F-23daae0a338e combines an unconditional dependency interaction with a control-gated, conditional transfer of a 16-bit field. Its returned result comes from F-941f6e9b1993, independently of whether the field transfer occurs.

The transfer is eligible only when three observations permit it: the caller-designated source field is nonzero, the control argument is nonzero, and a dependency-selected destination field is zero. Eligibility does not preserve either the source value or the destination selection. The actual transfer uses a later source read and a separately obtained destination.

An application-level interpretation such as initialization or synchronization is **inferred intent**, not established behavior. No semantic meaning is established for either field.

## Interface

### Arguments and result

| Item | Representation | Established use |
|---|---|---|
| Source argument | 32-bit integer in the supplied representation | Forwarded to dependencies and used as the base address of a directly read field |
| Control argument | 32-bit integer in the supplied representation | Forwarded unchanged to F-941f6e9b1993; tested locally for zero |
| Return value | Opaque 32-bit value | The saved result of F-941f6e9b1993 |

Every nonzero control value has the same local enabling effect. This includes negative values. The original value, rather than a normalized Boolean, reaches F-941f6e9b1993; that dependency may distinguish values that this function treats alike.

The return value has no established status-code, numeric, or address interpretation. No local conversion or replacement of its 32-bit representation is visible.

### Accessible storage

The function directly accesses only the following fields:

| Role | Offset from designated base | Extent | Permitted local operations |
|---|---:|---:|---|
| Source field | 230 bytes (`0xE6`) | 16 bits | Initial read and conditional reread |
| Tested destination field | 128 bytes (`0x80`) | 16 bits | Conditional read |
| Actual destination field | 128 bytes (`0x80`) | 16 bits | Conditional write |

The source base is the first argument. The tested and actual destination bases come from different invocations of F-fdd8589eca2d. They need not identify the same object.

The predicates compare 16-bit fields with zero. Although the source representation uses a signed 16-bit interpretation for these tests, signedness does not change their outcome: every nonzero bit pattern satisfies the source predicate and fails the destination-zero predicate. The transfer itself preserves the 16-bit field representation.

No complete object layout, allocation size, byte order, or alignment guarantee is established.

### Validity requirements

The source field must be readable when its initial read occurs, including when control is zero. A path that reaches the transfer additionally requires a readable source field at the later read, readable storage for the tested destination, and writable storage for the actual destination.

These requirements cover source-relative bytes 230–231 and destination-relative bytes 128–129. They are requirements on the addresses at the time of access, not merely on their validity at entry. Dependency activity may affect lifetimes or accessibility.

The address expressions involve 32-bit integer representations and fixed byte offsets. No explicit widening, checked addition, or overflow recovery is established. Valid execution requires address formation to be supported by the execution environment; this contract does not promise modulo wrapping or permit an overflowing addition merely because the resulting bit pattern might resemble an address.

## Behavior

### Unconditional dependency interaction

Every normally completing invocation includes an initial call to F-fdd8589eca2d with no arguments. Its 32-bit result is supplied to F-3e5212464798 together with the original source argument and a zero third argument.

F-941f6e9b1993 receives the original source and control arguments after F-3e5212464798 completes. Its result becomes the function’s saved return value.

This establishes an ordering constraint among the three calls:

1. The initial F-fdd8589eca2d call completes before F-3e5212464798 begins.
2. F-3e5212464798 completes before F-941f6e9b1993 begins.
3. F-941f6e9b1993 completes before the control-dependent work and direct field accesses.

The initial result of F-fdd8589eca2d is forwarded, not directly dereferenced by this function. No direct source-field read precedes this dependency interaction. The locally unused result of F-3e5212464798 does not alter the saved return value.

### Control and eligibility

Nonzero control requires one no-argument call to F-5c5958cfd5fe after F-941f6e9b1993 and before the initial source-field read. Zero control omits that call.

The initial source-field read is unconditional once execution reaches the field-processing portion. In particular, zero control prevents destination processing but does not prevent this read.

Destination inspection requires both a nonzero value from that source read and nonzero control. The source is observed before control can exclude destination processing at this point. Thus an invalid source address is not made safe by passing zero control.

When destination inspection is eligible, another no-argument invocation of F-fdd8589eca2d selects the object whose field is tested. A nonzero tested field prevents the transfer. A zero tested field permits the function to obtain the actual destination.

All field observations occur after the preceding dependency calls. Values present at entry are not snapshots used later.

### Transfer semantics

An eligible transfer requires a further call to F-fdd8589eca2d. The object selected by this final call receives the write. After that call returns, the source field is read again, and the resulting 16-bit representation is written to the actual destination field.

The following distinctions are observable requirements:

- **Selection is repeated.** The destination chosen for testing is not retained as the required write target.
- **The source is reread.** The initial nonzero source observation is not the value committed to the destination.
- **The actual destination is not tested.** Its existing field value does not constrain the write once the earlier destination test permitted a transfer.
- **Eligibility is not reconsidered.** A zero value at the final source read is still copied.

Consequently, the transfer may write zero or replace a nonzero field. Describing it simply as copying a nonzero source into an empty destination would omit established behavior.

The final source read precedes the destination write. No direct destination read is associated with the final selection. Dependency-internal accesses remain outside this description.

### Normal-completion outcomes

| Observations governing the path | Calls to F-fdd8589eca2d | Direct field activity |
|---|---:|---|
| Control is zero | 1 | Initial source read only |
| Control is nonzero; initial source read is zero | 1 | Initial source read only |
| Control and initial source are nonzero; tested destination is nonzero | 2 | Initial source read and destination test |
| Control and initial source are nonzero; tested destination is zero | 3 | Initial source read, destination test, source reread, destination write |

F-3e5212464798 and F-941f6e9b1993 each occur exactly once on every normally completing path. F-5c5958cfd5fe occurs exactly once for nonzero control and not at all for zero control.

Every normally completing path returns the saved result from F-941f6e9b1993. Field-transfer eligibility and completion introduce no separate local return status.

## Side effects

### Direct memory effects

There is at most one direct 16-bit write. It targets the field selected by the final F-fdd8589eca2d call.

The source argument is not explicitly used as a write destination. Nevertheless, the actual destination may alias the source object or overlap the source field. No non-aliasing guarantee exists. A direct write may therefore affect caller-designated source storage through aliasing.

The stated access widths do not establish atomicity, synchronization, memory barriers, or freedom from torn access. Behavior under concurrent unsynchronized mutation is **documented_only**.

### Dependency effects and ownership

The dependencies may change object contents, global state, object selection, or resource lifetimes. Nothing in this function establishes that any dependency preserves those properties.

No explicit allocation, release, reference acquisition, or reference release is performed locally. That observation does not establish that dependencies perform none, nor does it establish borrowed or owned status for returned addresses.

The local behavior does not explicitly transfer ownership of either argument or any selected object. Any broader ownership agreement is **documented_only**.

### Errors and interrupted execution

There is no visible local validation branch for null addresses, inaccessible storage, invalid lifetimes, or address arithmetic failure. No local error value is generated.

Results from F-3e5212464798 and F-5c5958cfd5fe are not inspected. The result of F-941f6e9b1993 is saved but not interpreted as success or failure. A normally returned value from that dependency does not suppress later operations.

Faults, dependency failures that do not return normally, and nonlocal control transfers have no visible local recovery. The ordering and call-count guarantees above apply only to operations actually reached and, where stated, to normal completion.

## Dependencies

| Opaque ID | Interaction | Limits of established knowledge |
|---|---|---|
| F-fdd8589eca2d | No arguments; initial result is forwarded, later results select field-access bases | Selection stability, validity, ownership, and side effects are unknown |
| F-3e5212464798 | Receives the initial dependency result, source argument, and zero | Local result is unused; other behavior is unknown |
| F-941f6e9b1993 | Receives both original arguments; supplies the saved 32-bit return value | Result meaning and side effects are unknown |
| F-5c5958cfd5fe | No arguments; invoked only for nonzero control | Local result is unused; other behavior is unknown |

Repeated calls to F-fdd8589eca2d are part of the observable contract. Caching a result, merging the calls, or assuming their objects are interchangeable requires guarantees not supplied here. Such changes could alter call effects, accessed storage, or the value ultimately written.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests. Unless otherwise stated, they assume valid address formation, accessible storage, and normal dependency returns.

1. **Disabled control does not eliminate source access.**  
   With control zero, the unconditional dependency interaction still occurs. F-5c5958cfd5fe is omitted. One source-field read occurs, with no destination lookup or direct write, and the saved result is returned.

2. **The enabling call can affect eligibility.**  
   With nonzero control, suppose the source field is nonzero at entry but F-5c5958cfd5fe changes it to zero. The later source test observes zero, so destination processing does not occur.

3. **An occupied tested destination blocks the transfer.**  
   With nonzero control and a nonzero initial source observation, suppose the destination test observes `0xFFFF`. This is nonzero regardless of signed interpretation. There is no final selection, source reread, or direct write.

4. **Stable values permit a straightforward transfer.**  
   Suppose both destination-related selections designate the same object, its tested field is zero, and the source remains `0x1234`. The actual destination receives the 16-bit representation `0x1234`. Its byte ordering is not specified here.

5. **A changed destination selection can replace an occupied field.**  
   Suppose the tested object has a zero field, but the final selection designates a different object whose field contains `0x5678`. The latter field is replaced by the final source value without another zero test. The tested object receives no direct write unless aliasing makes it overlap the actual destination.

6. **The copied value can become zero.**  
   Suppose the initial source test is nonzero and the destination test is zero. If the final selection call changes the source field to zero, zero is copied. The earlier source predicate is not reapplied.

7. **Negative control remains unnormalized.**  
   Control equal to −1 enables the optional call and local destination eligibility just as control equal to 1 does. F-941f6e9b1993 nevertheless receives −1, not 1.

8. **Return bits do not gate the transfer.**  
   If F-941f6e9b1993 returns any particular 32-bit pattern, later eligibility still depends on control and field observations. On normal completion, that same pattern is returned even if a field write occurs.

## Uncertainties

The following remain **documented_only**:

- Application meaning of the arguments, fields, dependency operations, and return value.
- Complete ABI details beyond the represented argument and result widths.
- Target byte order, alignment requirements, and exact integer-to-address semantics.
- Whether dependency selections are shared, thread-local, transient, or stable.
- Dependency behavior concerning globals, object mutation, ownership, and lifetime.
- Whether selected objects alias one another or the caller-designated object.
- Error conventions, fault behavior, and nonlocal exits.
- Reentrancy, thread safety, and concurrency guarantees.

No stable-selection, preservation, ownership, or wrapping-arithmetic assumption is added to resolve these unknowns.

## Evidence

The supplied function body and opaque dependency mapping establish the argument flow, relative call order, control predicates, 16-bit field accesses, byte offsets, repeated selection, source reread, and saved-result return described here.

The contract distinguishes those static observations from application-level interpretation and unspecified dependency behavior. No runtime tests have been performed. The acceptance examples describe conditional behavior supported by the supplied evidence, not observed executions.
