<!-- Trusted provenance: F-e7864e8fcb8a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e7864e8fcb8a

## Purpose

F-e7864e8fcb8a performs preparation followed by up to three indirect invocations: a first optional invocation, a mandatory invocation, and a final optional invocation.

The dispatch sequence contains **two distinct reads of the global selector after preparation**:

1. The first read supplies an address retained for the mandatory and final optional stages.
2. The second read supplies the structure address used to locate the first optional target.

The two reads are ordered, but their results are not established as equal. If they differ, the first optional target comes from a different structure than the targets used by the remaining stages.

**Established static behavior:** the mandatory and final optional entries are read from the retained structure when their respective stages are reached. The global selector is not visibly read again after the first optional target has been obtained. Changes to the retained structure’s entries can affect subsequent invocations.

**Inferred intent:** the optional invocations may be notifications surrounding the mandatory operation. No notification protocol, matching-target requirement, or pairing guarantee is established. Either optional invocation can occur without the other.

This contract describes the supplied static representation. No runtime tests have been performed.

## Interface

### Inputs

The visible interface contains four argument positions and no defined return value.

| Position | Represented width | Visible use |
|---|---:|---|
| First | 32 bits | Used as an address for a four-byte read at byte offset 4; forwarded to the first optional invocation, if present, and the mandatory invocation. |
| Second | 32 bits | No visible use. |
| Third | 32 bits | No visible use. |
| Fourth | 32 bits | Forwarded in the recovered argument list of the first optional invocation, if present. |

The first argument is represented as a signed 32-bit integer. Its visible uses are address formation and argument forwarding, not signed comparison. No sign, range, alignment, or address-provenance validation is shown.

The absence of visible uses for the second and third arguments does not establish that the original source-level interface omitted them. Likewise, the recovered argument lists do not establish the formal prototypes of indirect targets.

### Accessed memory layout

The supplied representation uses a 32-bit address and word model.

| Location | Accessed content |
|---|---|
| Input address plus 4 bytes | Four-byte value supplied to preparation. |
| Separate stored datum | Four-byte preparation argument. |
| Global selector | Structure address, read twice after preparation. |
| Structure address from the second selector read, plus 48 bytes | First optional target. |
| Retained address from the first selector read, plus 24 bytes | Mandatory target. |
| Retained address from the first selector read, plus 48 bytes | Final optional target. |

These are accessed fields, not complete object definitions. The evidence does not establish total object sizes, other fields, type tags, allocation metadata, endianness, or alignment requirements.

Indirect-target slots are pointer-sized in the represented 32-bit model. The meaning of the input field and the separate preparation datum is unknown.

### Output and completion

No result value is defined for callers to consume. Dependency results are not visibly inspected or used to select later stages.

When the final optional target is null, the function explicitly returns without that invocation. When it is non-null, the representation shows an indirect invocation followed by return, but the exact final-transfer mechanics remain unresolved.

Normal continuation requires invoked dependencies to return. Faults, exceptions, nonlocal transfers, or nonreturning dependencies can prevent later stages.

### Preconditions for normal progression

The function does not enforce these requirements:

- The input field must be readable when preparation arguments are obtained.
- F-fb5bc2dc656c must accept its supplied arguments and return.
- Both reads of the global selector must be possible.
- The structure identified by the second selector read must permit the first optional-entry read.
- The retained structure must permit its mandatory and final optional-entry reads when those stages are reached.
- Every invoked target must be callable under the actual ABI.
- Dependencies must not invalidate memory needed by later accesses if normal progression is expected.

A null optional target is permitted and causes a skip. A null mandatory target has no corresponding guarded path.

## Behavior

### Preparation

The function obtains:

- The four-byte value at byte offset 4 from the input address.
- The separate stored four-byte datum.

It invokes F-fb5bc2dc656c with the input-field value first and the stored datum second.

The relative evaluation order of the two argument-producing reads is not established by the supplied representation. Both values are obtained for the preparation call.

There is no visible input-address null check or field-value validation. No result from preparation is visibly consumed.

Preparation may affect global state or referenced memory. In particular, neither post-preparation selector read is required to observe the selector’s value from before preparation.

### Dispatch selection: two ordered selector reads

After preparation returns, the function first reads the global selector and retains that address.

It then reads the global selector again. This second result supplies the base address for the first optional-target read at byte offset 48.

The distinction is behavioral:

- The **retained structure** is selected by the first read.
- The **first-stage structure** is selected by the second read.
- Only under an explicit equality assumption are these necessarily the same structure.

There is no visible dependency call between the two selector reads. Nevertheless, the supplied representation contains both reads; it does not justify replacing them with one read or asserting equal results.

The later mandatory and final optional stages use the retained address. They do not reselect their structure through the global selector.

Retaining an address does not acquire ownership, extend lifetime, or protect the referenced memory.

### First optional invocation

Using the structure address obtained by the **second** selector read, the function reads the indirect-target slot at byte offset 48.

If the target is null, the first optional invocation is skipped.

If the target is non-null, the recovered invocation presents five argument positions:

1. The original first argument.
2. Zero.
3. The target selected for this invocation.
4. The original fourth argument.
5. An unresolved register-derived value.

The origin, initialization, and application-level meaning of the fifth value are **documented_only**. It must not be specified as a particular caller argument, constant, or preserved value.

The third recovered argument is the selected target value itself. It is not an additional dispatch-entry read.

These recovered positions do not establish the target’s formal parameter list or a complete ABI contract. No visible result from this invocation controls subsequent behavior.

If this invocation changes the global selector, the change does not redirect the later stages: their structure address has already been retained. If it changes entries in the retained structure, those changes can affect later reads.

### Mandatory invocation

After the first optional invocation returns, or immediately after that stage is skipped, the function reads the mandatory-target slot at byte offset 24 from the **retained structure**.

It invokes that target with the original first argument.

The mandatory target is read at this stage, not cached before the first optional invocation. A modification to this entry before the read can therefore change the selected target.

There is no visible null check. A null or otherwise invalid mandatory target does not cause a defined graceful return, error result, or fallback.

No result from the mandatory invocation is visibly consumed.

### Final optional invocation

After the mandatory invocation returns, the function reads the optional-target slot at byte offset 48 from the **retained structure**.

This is a new entry read. It does not reuse the first optional target. It can differ because:

- The two stages may use different structures.
- The retained structure’s optional entry may have changed.
- Both conditions may apply.

If the newly read target is null, the function returns without the final optional invocation.

If it is non-null, the recovered final invocation presents two arguments:

1. Zero.
2. One.

The original first argument is not included in this recovered argument list. The lists for the two optional stages must not be normalized into one assumed signature.

The final operation is represented as an indirect call followed by return, but its exact transfer form and full ABI state are **documented_only**. The supported description is a non-null-dependent indirect transfer with visible argument expressions zero and one; an exact call-versus-tail-transfer contract is not established.

### Widths and arithmetic

All layout offsets are byte offsets: 4, 24, and 48.

The input field and separate preparation datum are four-byte values. The representation shows no length arithmetic, multiplication, bounds check, or explicit overflow check.

The first argument and retained selector address are represented through 32-bit integer values. Valid execution requires each offset access to reach its intended readable location. The evidence does not establish a portable wraparound rule for addresses near the address-space boundary or define behavior for signed-overflow cases in the recovered address expressions.

The constants zero and one have no established application meaning. They must not be described as Boolean flags, event identifiers, or status codes without additional evidence.

## Side effects

### Direct reads and their order

The visible memory-access sequence is:

1. Read the input field and stored preparation datum, in an unspecified relative order.
2. Invoke F-fb5bc2dc656c.
3. Read the global selector and retain its address.
4. Read the global selector again.
5. Read the first optional entry at offset 48 from the address supplied by the second selector read.
6. If that target is non-null, invoke it.
7. Read the mandatory entry at offset 24 from the retained address.
8. Invoke the mandatory target.
9. Read the final optional entry at offset 48 from the retained address.
10. If that target is non-null, perform the final indirect invocation.

Each stage following an invoked dependency is contingent on that dependency returning normally.

No direct memory write, allocation, release, reference-count adjustment, or explicit global update is visible in this body.

### Dependency-mediated effects and ownership

Dependencies may change globals, modify dispatch entries, invalidate referenced memory, acquire or release resources, or fail to return. Their implementations are not supplied.

No ownership transfer or lifetime guarantee is established for:

- The input object.
- Either selected structure.
- Any indirect target.
- The stored preparation datum.

Forwarding the input address does not visibly copy the object. Retaining a structure address does not establish a reference-count increment, lock, or other lifetime protection.

### Errors and concurrency

There is no visible error-reporting channel, retry, recovery branch, or cleanup path.

Null optional entries cause their respective stages to be skipped. The mandatory entry is invoked without a null guard.

Invalid memory or targets may fail according to the execution environment; the precise failure mode is unknown.

No locking, atomicity, volatility, memory-ordering, or synchronization guarantee is established. The two selector reads remain distinct in the supplied representation, but this alone does not define deterministic behavior under concurrent mutation or guarantee a particular machine-level observation model.

## Dependencies

Only the preparation dependency has a supplied stable opaque function ID. The indirect targets are dynamically selected; assigning them fixed function identities would exceed the evidence.

| Dependency | Selection and call-site contract |
|---|---|
| F-fb5bc2dc656c | Receives the four-byte input-field value and stored datum. Its result is not visibly used. Both selector reads follow its return. |
| Dynamically selected first optional target | Read at offset 48 from the structure identified by the second selector read. Invoked only if non-null, with the five recovered argument positions described above. |
| Dynamically selected mandatory target | Read at offset 24 from the address retained from the first selector read. Invoked with the original first argument, without a null check. |
| Dynamically selected final optional target | Read at offset 48 from the retained address after the mandatory invocation. Invoked only if non-null, with visible arguments zero and one; final-transfer mechanics remain unresolved. |

The indirect dependencies need not identify the same function or share a prototype, even when their entries occupy the same offset in different structures.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests. Unless stated otherwise, required memory is accessible, targets are callable, and invoked dependencies return normally.

### Both optional stages are absent

If the first-stage structure’s optional entry is null when read and the retained structure’s optional entry is null at the final read, only preparation and the mandatory invocation occur.

This does not require the two selector reads to return equal addresses.

### Equal selector results and a stable optional target

If both selector reads return the same address and its optional entry remains a particular non-null target, that target is selected twice.

The first invocation uses the recovered five-position argument list. The final invocation uses the recovered zero-and-one list. Equality of target addresses does not establish a common formal signature.

### Different selector results

Suppose the first selector read identifies one structure and the second identifies another.

The first optional target is read from the second structure’s offset 48. The mandatory target is read from the first structure’s offset 24, and the final optional target from the first structure’s offset 48.

No target is substituted from the other structure merely to make the sequence appear paired. This example is conditional on the two reads observing different addresses; it does not establish how such a change is synchronized.

### Preparation changes the selector

If preparation changes the selector before returning, both selector reads occur afterward.

If the selector then remains stable across those reads, both observe that post-preparation structure. The function does not visibly retain a pre-preparation selector value.

### The first optional invocation changes only the selector

If the first optional invocation changes the global selector but leaves the retained structure intact, the mandatory and final optional reads still use the retained structure.

There is no later selector read through which that change redirects this invocation.

### The first optional invocation replaces the retained mandatory entry

If the first optional invocation modifies the retained structure’s mandatory entry, the replacement is read and invoked afterward.

This remains true even when the first optional target came from a different structure.

### The mandatory invocation installs a final optional target

If the first optional target read is null, but the mandatory invocation installs a non-null optional entry in the retained structure before the final read, only the final optional invocation occurs.

The two optional stages are not coupled.

### The final optional entry is null

If the first optional target is non-null but the retained structure’s optional entry is null at the final read, the first invocation occurs and the final invocation is skipped.

The null may result from a modification, or the stages may simply use different structures.

### The mandatory target is null

If the retained structure’s mandatory entry is null when read, no graceful-return branch is selected. The represented operation attempts invocation through that value.

The resulting runtime outcome is unknown.

## Uncertainties

The following remain **documented_only**:

- The complete original prototype and calling convention.
- The application meaning of the input field and stored preparation datum.
- The origin and meaning of the unresolved fifth argument.
- Which recovered argument positions are formal target parameters.
- The final transfer’s exact control-flow form, full ABI state, and return mechanics.
- Dependency effects, ownership rules, and return behavior.
- Object lifetime and synchronization requirements.
- Endianness, alignment restrictions, and invalid-address behavior.
- The storage qualifiers and machine-level observation semantics of the global selector.
- Whether the stored preparation datum can change.

The ordered selector reads are established in the supplied representation. Their equality is not. Conversely, the existence of two reads does not establish that they differ in any actual execution.

The paired-notification interpretation remains inferred intent only. No cleanup, destructor, ownership-transfer, or error protocol is established.

## Evidence

The supplied static representation supports:

- Preparation using an input-field value and a separately stored datum.
- A first post-preparation selector read whose address is retained.
- A second selector read supplying the first optional-entry base.
- A guarded first optional invocation.
- A later mandatory-entry read from the retained address, without a null guard.
- A final optional-entry read from that same retained address.
- Different recovered argument lists for the optional stages.
- No visible direct memory writes or consumption of dependency results.

The unresolved register-derived argument and uncertain final indirect-transfer recovery limit the ABI contract.

No implementation execution, runtime tests, independent instruction-level verification, dependency inspection, or ownership analysis was performed. The acceptance examples express conditional consequences of the supplied static representation, not observed runtime results.
