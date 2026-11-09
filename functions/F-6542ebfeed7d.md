<!-- Trusted provenance: F-6542ebfeed7d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6542ebfeed7d

## Purpose

F-6542ebfeed7d coordinates a requested 32-bit value with two receiver-held related-object references. It inspects related objects, selects one of three transition alternatives, obtains a 64-bit comparison result, performs conditional updates and notifications, and clears specified receiver fields on normal completion.

The established contract concerns explicit accesses, argument flow, branch precedence, and ordering. Descriptions such as “current object,” “pending object,” “position,” “allocation,” or “lock” are possible interpretations, not established semantics.

There is no explicit return value or error result. Effects are observable through memory writes and dependency calls. This contract is based on static evidence only; no runtime tests have been performed.

## Interface

### Inputs

| Input | Representation and established use |
|---|---|
| Receiver | Pointer to mutable storage organized in 32-bit words. Provides dispatch tables, related-object references, embedded addresses, and writable fields. |
| Requested value | Signed 32-bit input. Forwarded to dependencies, recorded in the receiver, compared with the low component of a dispatch result, and used in subtraction. |
| Selection control | Signed 32-bit input. Tested only for equality with zero. A nonzero value prevents the primary-match alternative, but does not prevent secondary promotion. |

The supplied representation uses 32-bit stored references and dispatch-table fields. One receiver dispatch produces a 64-bit result.

No explicit validation checks receiver validity, reference validity, dispatch targets, or dependency results.

### Receiver layout

Offsets are byte offsets from the receiver.

| Offset | Established use |
|---|---|
| `0x000` | Main dispatch-table reference. |
| `0x014` | Notification-subobject address; its first 32-bit word supplies its dispatch-table reference. |
| `0x054` | Primary related-object reference. |
| `0x058` | Secondary related-object reference. |
| `0x05C` | Receives the requested value after the main comparison dispatch returns. |
| `0x088` | Embedded address passed to U-02 or U-03. |
| `0x990`–`0x9A0` | Five consecutive 32-bit fields cleared in ascending address order. |
| `0x9A4` | One byte cleared after those five fields. |
| `0xA4C` | Embedded address passed first to U-01 and later to U-04. |

A nonzero related-object reference leads to an interface at that reference plus four bytes. The interface’s first word supplies a dispatch-table reference.

Additional temporary storage comprises a 24-byte region, three separate 32-bit key-result locations, a retained 64-bit dispatch result, a retained Boolean condition, and retained 32-bit notification values.

Three global 32-bit values are designated **K-equal**, **K-different**, and **K-final**. Their values and meanings are unknown.

### Preconditions and result validity

Every reached memory access requires valid accessible storage; every reached dispatch requires a callable target.

The function does not explicitly initialize the three key-result locations before their producer calls:

- F-292bc74cc014 receives the temporary-key output address.
- A nonzero primary reference permits a dispatch receiving the primary-key output address.
- A nonzero secondary reference permits a dispatch receiving the secondary-key output address.

A zero slot skips its producer without supplying a replacement key. Defined comparisons therefore require the relevant output locations to have been established by their producers.

The primary-key comparison precedes the selection-control zero test. Nonzero selection control does not prevent that comparison. A reached comparison involving an unestablished key has no deterministic behavior specified here.

## Behavior

### Initial calls and inspection

The first call is F-ba5a4c1e0746 with the 24-byte temporary-region address and two zeros. This establishes an invocation, not an independently known initialization contract.

Next, the main receiver dispatch at table offset `0x170` receives the receiver and requested value. After it returns, the retained Boolean condition is set false. F-292bc74cc014 then receives the temporary-region address and temporary-key output address. U-01 follows, receiving the receiver’s embedded address at `0xA4C`.

Related-object inspection occurs after these calls. Their mutations can therefore affect the inspected references.

If the primary slot is nonzero:

1. Its embedded interface’s `0x168` dispatch receives the interface address and primary-key output address.
2. The primary slot and dispatch information are read again to invoke entry `0x138`.
3. Only if that result is nonzero, another lookup through the primary slot invokes entry `0x0E4`.
4. The retained condition becomes true exactly when both predicate results are nonzero.

The predicate invocations show no explicit arguments; their complete calling conventions remain unknown.

The initial nonzero check does not protect against a dependency subsequently replacing the primary slot with zero or an invalid reference. No renewed nonzero check precedes the later predicate lookups.

After primary inspection, a nonzero secondary slot causes its embedded interface’s `0x168` dispatch to receive that interface address and the secondary-key output address.

The retained condition is not recomputed after transition selection.

### Transition selection

Three alternatives have ordered precedence.

**Primary match.** If the temporary key equals the primary key and selection control is zero, call F-7fc6540cfe92 with the receiver. This alternative wins even if the secondary key also matches. The branch itself performs no direct related-slot assignment; dependency effects remain unknown.

**Secondary match.** Otherwise, if the temporary key equals the secondary key:

- Call F-b29453bea6f2 with the receiver.
- After it returns, read the secondary slot.
- Write that value into the primary slot.
- Write zero into the secondary slot.

The copied reference is the post-call secondary-slot value, not a retained reference from key inspection.

**Remaining alternative.** Otherwise:

- Call F-b29453bea6f2 with the receiver.
- Call F-7fc6540cfe92 with the receiver.
- Call F-8157a180d901 with numeric argument `0x24`, decimal 36.
- Forward its 32-bit result to F-9e37ec060d26 together with the temporary-region address and requested value.
- Store F-9e37ec060d26’s 32-bit result in the primary slot.

There is no explicit result check before forwarding or storing these results. The numeric argument 36 is not independently established as an allocation size. This alternative has no direct secondary-slot clear.

### Comparison and request recording

After the chosen transition returns, the main receiver dispatch at table offset `0x158` receives the receiver and produces a retained 64-bit result.

Only then does the function write the requested value to receiver offset `0x05C`.

Subsequent behavior compares the request with the result’s low 32-bit component. The high 32-bit component is used only on the unequal path. These are representation-level components, not established domain fields. Signed interpretation and conversion at representation boundaries require the qualifications in **Uncertainties**.

### Equal component

When the requested value equals the low component:

- Fetch the notification dispatch target at table offset `0x014`.
- Read K-equal into a retained 32-bit value.
- Invoke the captured target with the notification-subobject address and retained value.

The target lookup precedes the global read. The invocation has two explicit arguments.

This branch skips the temporary predicates, both embedded-state update alternatives, F-b582c09a05ae, and the three K-different helper calls. It proceeds to common completion.

### Different component

When the requested value differs from the low component, F-a2c64bafa8a6 receives the temporary-region address and high component. If its result is nonzero, F-a7ff1a6124bd receives the temporary-region address.

The first update alternative is selected if either:

- Both temporary predicates return nonzero; or
- The retained primary-object condition is true.

The temporary predicates are evaluated before the retained condition supplies the alternative reason for success. Thus a true retained condition does not skip F-a2c64bafa8a6, nor does it skip F-a7ff1a6124bd when the first predicate is nonzero.

The first update alternative calls U-02 with receiver address `0x088`, then F-b582c09a05ae with the receiver.

The other alternative calls U-03 with receiver address `0x088`, zero, and the 32-bit signed subtraction of the low component from the requested value. No range check, saturation, absolute-value conversion, or explicit overflow handling is present. A portable result for signed overflow is not established.

### K-different argument sequence

After either unequal-path update, the function performs two distinct K-different reads:

1. Read K-different and retain that 32-bit unsigned value.
2. Read K-different separately for the first F-069a1d444a2d call, which also receives the receiver.

The next two calls to F-069a1d444a2d also receive the receiver. Their other arguments are formed independently from the original retained value:

- Bitwise OR of the retained value with a signed-converted copy shifted right by 18.
- Bitwise OR of the retained value with a signed-converted copy shifted right by 17.

Neither calculation uses the preceding derived argument. Neither rereads the global.

**The signed conversion and shifts do not establish unconditional sign extension.** If the retained unsigned value is representable as signed 32-bit, its signed conversion is exact and nonnegative; both right shifts have the ordinary nonnegative result. If it exceeds the signed maximum, the conversion is implementation-defined in the supplied language representation. If conversion produces a negative value, its right-shift behavior is also implementation-defined. Target/compiler or instruction-level evidence would be needed to guarantee arithmetic right shift and sign extension. Exceptional conversion behavior is likewise not resolved by this body.

After all three helper calls return, the notification target is fetched and invoked with the notification-subobject address and the original retained K-different value. Later global mutations do not change these retained-derived arguments or that notification argument.

The separate reads have distinct dataflow origins. No stability or concurrency contract establishes that they must return equal values.

### Common completion

Both comparison paths reach this ordered suffix:

1. Read K-final into a retained 32-bit value.
2. Read K-final separately for a call to F-069a1d444a2d, also passing the receiver.
3. After that call returns, reload the notification dispatch target and invoke it with the notification-subobject address, the earlier retained K-final value, and explicit zero.
4. Clear the 32-bit receiver fields at `0x990`, `0x994`, `0x998`, `0x99C`, and `0x9A0`, in that order.
5. Clear the byte at `0x9A4`.
6. Call U-04 with receiver address `0xA4C`.
7. Call F-4e292a3dd745 with the temporary-region address.
8. Return without an explicit value.

The common notification has three explicit arguments. The preceding notification has only two; its missing third argument must not be treated as an explicit zero.

The helper’s K-final argument and the common notification’s K-final argument come from different reads. Their equality requires an additional stability assumption.

## Side effects

### Direct writes

The function directly writes:

- The primary slot on secondary promotion and on the remaining transition alternative.
- Zero to the secondary slot on promotion.
- The requested value to `0x05C`.
- Five zero words and one zero byte during common completion.

The explicit completion clears cover 21 bytes, not 24. They do not clear bytes `0x9A5` through `0x9A7`.

These are guarantees about writes at particular points, not unconditional return-state guarantees. Later dependencies can modify the same storage. In particular, U-04 and F-4e292a3dd745 execute after the clears.

### Ownership and dependency effects

Dependencies may modify receiver storage, globals, dispatch tables, related objects, or other state. Their argument lists do not establish preservation guarantees.

Forwarding F-8157a180d901’s result into F-9e37ec060d26 and storing the latter result does not establish allocation, construction, ownership transfer, release of an old object, or a failure representation. A zero result is not explicitly rejected.

Likewise, the first and last temporary-region calls do not independently establish initialization, destruction, or resource release.

### Abnormal completion

No explicit error status, rollback path, exception handler, or guaranteed cleanup mechanism is visible.

If a dependency fails to return normally, later notifications, writes, U-04, and the final temporary-region call are not established to execute. Earlier effects are not visibly undone.

U-01 and U-04 use the same embedded address and bracket much of the work. A synchronization role is inferred intent only; mutual exclusion, atomicity, recursive use, and exceptional release are unknown.

## Dependencies

Identifiers are opaque. U-01 through U-04 are document-local opaque identifiers without supplied published mappings.

| ID | Established use |
|---|---|
| F-ba5a4c1e0746 | Temporary-region address and two zeros before initial receiver dispatch. |
| F-292bc74cc014 | Temporary-region address and temporary-key output address. |
| F-b29453bea6f2 | Receiver before promotion or the remaining transition. |
| F-7fc6540cfe92 | Receiver on primary match and within the remaining transition. |
| F-8157a180d901 | Numeric argument 36; produces a forwarded 32-bit result. |
| F-9e37ec060d26 | Forwarded result, temporary-region address, and request; produces the stored primary-slot value. |
| F-a2c64bafa8a6 | Temporary-region address and high result component; first unequal-path predicate. |
| F-a7ff1a6124bd | Temporary-region address; conditionally evaluated second predicate. |
| F-b582c09a05ae | Receiver after U-02. |
| F-069a1d444a2d | Receiver and each specified global-derived argument. |
| F-4e292a3dd745 | Temporary-region address after U-04. |
| U-01 | Receiver address `0xA4C` before inspection. |
| U-02 | Receiver address `0x088` on the first update alternative. |
| U-03 | Receiver address `0x088`, zero, and signed difference on the other alternative. |
| U-04 | Receiver address `0xA4C` after the clears. |

Dispatch dependencies are identified by interface and table byte offset: main entries `0x170` and `0x158`; related-interface entries `0x168`, `0x138`, and `0x0E4`; notification entry `0x014`.

## Acceptance examples

These are conditional specifications, not executed tests. Unless stated otherwise, they assume valid storage, callable targets, established compared keys, and normal dependency returns.

1. **Primary precedence.** If both keys match the temporary key and selection control is zero, select F-7fc6540cfe92 without taking promotion.

2. **Nonzero control and promotion.** With nonzero control and a matching secondary key, call F-b29453bea6f2, then copy the post-call secondary value to the primary slot and clear the secondary slot.

3. **Mutation during promotion.** If that call changes the secondary reference, copy the changed reference, not the one previously used for key extraction.

4. **Remaining transition.** With no eligible match, call F-b29453bea6f2, F-7fc6540cfe92, F-8157a180d901 with 36, and F-9e37ec060d26 in order. Store the final result without inferring allocation semantics.

5. **Equal comparison.** With equal request and low component, invoke neither temporary predicate. Fetch the notification target before reading K-equal, notify, then perform common completion.

6. **Retained inspection result.** Successful initial primary predicates retain a true condition even if a transition replaces the primary object. On the unequal path, this condition selects U-02 followed by F-b582c09a05ae.

7. **Predicate ordering despite retained truth.** With retained truth and a nonzero first temporary predicate, the second predicate still runs before U-02.

8. **Ordinary subtraction.** With retained false, a zero first temporary predicate, request 120, and low component 100, skip the second predicate and pass zero and difference 20 to U-03.

9. **Distinct K-different reads.** If the first read yields 1 and the second yields 2, the first helper receives 2, both derived helpers receive 1, and the notification receives 1. Whether such differing observations are feasible is documented_only.

10. **Established nonnegative shifts.** If retained K-different is `0x00040000`, the 18-bit derived argument is `0x00040001` and the 17-bit derived argument is `0x00040002`. Both derive independently from the retained value.

11. **High-bit qualification.** If retained K-different is `0x80000000`, no unconditional derived arguments are specified. Under the additional assumptions of conversion to the corresponding negative signed value and arithmetic right shift, the derived arguments would be `0xFFFFE000` and `0xFFFFC000`, respectively. Those assumptions are not established here.

12. **Distinct K-final reads.** If the retained read yields 7 and the helper-argument read yields 9, the helper receives 9 and the final notification receives 7 and explicit zero.

13. **Exact clearing.** Common completion explicitly writes five zero words and one zero byte. A claim that they remain zero at return additionally requires preservation by subsequent dependencies.

14. **Missing key.** A zero primary slot does not establish a primary key, even with nonzero control. A reached comparison without an established key has documented_only behavior.

## Uncertainties

- **documented_only — Domain semantics:** Object roles, request meaning, notification meanings, and global values are unknown.
- **documented_only — Dependency contracts:** Allocation, construction, initialization, destruction, release, failure representations, and preservation guarantees are unavailable.
- **documented_only — Key validity:** A skipped or ineffective producer leaves reached key comparisons without an established deterministic result.
- **documented_only — Dispatch ABI:** Complete signatures of argument-free predicates and differently arity notification calls are unavailable.
- **documented_only — Signed conversions and shifts:** Out-of-range unsigned-to-signed conversion and negative signed-right-shift behavior are not established. Sign extension is not guaranteed by the supplied representation alone.
- **documented_only — Other arithmetic boundaries:** Portable signed-overflow behavior for subtraction and target-specific signed interpretation at result-component conversion boundaries remain unresolved.
- **documented_only — Synchronization:** No concurrency or global-stability model is supplied.
- **documented_only — Abnormal completion:** Propagation, external cleanup, and exceptional resource handling are unknown.

## Evidence

Established static evidence comprises explicit access widths and offsets, call arguments, result flow, short-circuit evaluation, branch precedence, repeated reference reads, separate global reads, dispatch-lookup placement, and ordered normal-path writes.

For the K-different calculations, the supplied representation establishes a retained unsigned 32-bit value, conversion of a copy to signed integer, right-shift counts of 18 and 17, and independent OR combinations with the retained value. It does not independently establish the implementation-defined conversion result or negative-shift behavior.

Interpretations based on apparent dependency roles remain separate from established behavior. No target/compiler contract or instruction-level evidence resolving those arithmetic uncertainties is supplied. No runtime tests have been performed.
