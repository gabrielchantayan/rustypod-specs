<!-- Trusted provenance: F-7ec4b7b0ad1d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7ec4b7b0ad1d

## Purpose

F-7ec4b7b0ad1d conditionally dispatches an operation through an object-associated indirect target.

The established behavior has three stages:

1. Obtain a receiver from F-e6c2a72b1331.
2. If the receiver is non-null, pass the second input to F-31316fda04f5.
3. If that dependency returns a nonzero 32-bit value, select an indirect target using the receiver and dispatch with the receiver and that value as arguments.

The first input does not participate in the visible behavior. Either prerequisite can stop the operation before dispatch.

This describes the control flow and accesses visible in the supplied static material. It does not establish the application-level meaning of the receiver, the second input, the intermediate result, or the dispatched operation.

## Interface

### Inputs

The function accepts two values, each represented as a 32-bit quantity:

| Input | Established use |
|---|---|
| First input | Not read, forwarded, compared, or otherwise used by the visible body. |
| Second input | Passed by value to F-31316fda04f5, but only after F-e6c2a72b1331 returns a non-null receiver. |

No visible validation, masking, extension, arithmetic, or dereference is applied to either input.

The second input’s semantic type is unknown. It could represent a scalar, identifier, address, or another 32-bit encoding; none of those interpretations is established. In particular, the wrapper does not itself dereference that input.

The first input is behaviorally unused in the supplied body. This does not establish that it can be removed from the external interface: calling conventions and external callers are not provided.

### Output

The recovered interface has no return value. The visible paths either return without dispatch or reach the indirect operation and then show a return.

There is no explicit success indicator, error code, or output parameter. Any result produced by a dependency or the indirect target, beyond the intermediate value used for dispatch, is not exposed as a defined return value of this function.

The final transfer has a recovery ambiguity described under **Uncertainties**. Consequently, ordinary call-and-return behavior for that transfer is not independently established.

### Conditional memory requirements

There is no unconditional caller-buffer requirement visible in the body. Memory requirements arise only on the path that reaches dispatch:

- The receiver returned by F-e6c2a72b1331 must remain suitable for the later receiver read.
- The receiver’s initial 32-bit field must provide an address usable for the dispatch lookup.
- The target slot at byte offset 72 from that address must be readable and contain a usable transfer target.
- The selected target must accept the argument arrangement represented by the recovered invocation.

The function does not visibly check these conditions. They are requirements for a valid dispatch, not validations performed by the function.

## Behavior

### Receiver acquisition and first early return

The first visible external operation is a call to F-e6c2a72b1331 with no explicit arguments.

Its result is retained as the receiver and compared with null.

If it is null:

- F-31316fda04f5 is not called.
- No receiver field or dispatch slot is read by the visible body.
- No indirect dispatch occurs.
- The function returns without a defined result.

This path does not inspect the second input. It does not establish that the first dependency itself has no side effects.

### Intermediate-value acquisition and second early return

If the receiver is non-null, the function calls F-31316fda04f5 with the second input.

The dependency’s result is represented as a 32-bit integer. The function distinguishes exactly zero from all nonzero bit patterns.

If the result is zero:

- No receiver field or dispatch slot is read by the visible body.
- No indirect dispatch occurs.
- The function returns without a defined result.

There is no visible cleanup action before this return.

A nonzero result is not checked for positivity, alignment, range, address validity, or any other property. Although the recovered representation is an integer, the branch is a zero test rather than a signed ordering test. A value with its highest bit set therefore does not take the zero-result exit merely because it could be interpreted as negative.

### Indirect-target selection

When both prerequisites succeed, the function uses the receiver retained from the first dependency call. It does not visibly acquire a replacement receiver after the second dependency returns.

The visible lookup then:

1. Reads the initial 32-bit field at the receiver address.
2. Uses that field as an address-like base.
3. Locates a target slot 72 bytes, equivalently hexadecimal 0x48, beyond that base.
4. Reads the indirect target from that slot.
5. Transfers control to the selected target, presenting the retained receiver first and the nonzero intermediate value second.

The receiver-field read occurs after F-31316fda04f5 returns. The visible body does not snapshot this field before calling that dependency. Therefore, changes made by the dependency can affect the later target selection, provided the retained receiver remains valid.

The displacement is 72 bytes from the address represented by the initial receiver field. It is not a displacement of 72 integer elements and is not applied directly to the receiver address.

The supplied material represents the receiver’s initial field as 32 bits. The exact native pointer width, target-slot width, alignment rules, and machine-level address arithmetic are not independently specified. In particular, no bounds check or overflow check is visible, and no portable signed-overflow or address-wraparound behavior should be inferred from the recovered expression.

### Ordering guarantees and limits

For a completed dispatch path, the visible order is:

- F-e6c2a72b1331 completes.
- Its result is found non-null.
- F-31316fda04f5 is invoked with the second input and completes.
- Its result is found nonzero.
- The receiver’s initial field is read.
- The target slot is read.
- The indirect transfer occurs.

These are ordering statements about the wrapper’s visible operations. They do not describe accesses inside dependencies, establish atomicity, or provide synchronization guarantees.

## Side effects

The visible body contains no explicit write to caller memory, receiver memory, global state, or an output parameter. Its explicit work consists of dependency invocation, comparisons, lookup reads, and indirect dispatch.

That does not make the complete operation read-only. Either dependency, and the indirect target, may change memory or external state. Their effects are not supplied.

No explicit allocation, release, reference-count adjustment, ownership transfer, rollback, or cleanup is visible. Accordingly:

- Acquiring a receiver does not establish that ownership was acquired.
- Returning early does not establish a leak.
- Passing the intermediate result does not establish that ownership was transferred.
- Absence of explicit release does not establish that callees perform no release.

The receiver is retained across the second dependency call without a visible lifetime guard or revalidation. If that call invalidates it, the subsequent read can be invalid. The supplied evidence does not establish whether such invalidation is possible in the intended environment.

There is no visible retry or local error-recovery mechanism. Faults, nonlocal exits, and non-returning dependency behavior are not translated into a documented status by this body.

## Dependencies

### F-e6c2a72b1331

- **Arguments:** None explicitly supplied.
- **Consumed result:** A receiver address, with null selecting immediate return.
- **Invocation count:** Once per invocation that reaches the visible body.
- **Unknowns:** Receiver provenance, lifetime, ownership, object layout beyond the observed lookup, and dependency side effects.

Only null is rejected. A non-null result is not otherwise validated before the second dependency is called.

### F-31316fda04f5

- **Argument:** The second 32-bit input.
- **Consumed result:** A 32-bit value; zero suppresses dispatch and every nonzero value proceeds toward dispatch.
- **Invocation count:** Once when the first dependency returns non-null; otherwise zero.
- **Unknowns:** Input interpretation, result interpretation, failure meaning, ownership effects, and changes to the receiver or other state.

No assumption that this dependency preserves globals or receiver storage is justified.

### Runtime-selected indirect target

There is no established opaque function ID for the selected target.

- **Selection:** Read through the receiver-associated base at byte displacement 72.
- **Arguments represented:** Retained receiver, followed by the nonzero intermediate result.
- **Result handling:** No target result is explicitly consumed.
- **Unknowns:** Target identity, calling convention details, semantic operation, effects, ownership rules, and whether the transfer returns normally.

## Acceptance examples

These examples are conditional static acceptance criteria, not reported execution results. They assume dependency calls return normally unless stated otherwise.

### Null receiver

Given that F-e6c2a72b1331 returns null, the function returns without calling F-31316fda04f5 or reading a dispatch slot. This remains true for every second-input bit pattern.

The first dependency may still have produced effects of its own.

### Non-null receiver and zero intermediate result

Given a non-null receiver and a zero result from F-31316fda04f5, the second dependency receives the original second input. The function then returns without reading the receiver’s initial field or selecting an indirect target.

This path does not require the receiver’s dispatch layout to be usable, because the visible lookup is skipped.

### Nonzero intermediate result

Given a valid receiver, a nonzero intermediate result, and a valid lookup layout, the selected target receives the retained receiver and that intermediate value in that order.

For example, a result with the 32-bit pattern 0x80000000 passes the zero test. Whether the target accepts that value is outside the established contract.

### Receiver-field mutation during the second dependency

Given that F-31316fda04f5 changes the retained receiver’s initial field while leaving the receiver valid, subsequent lookup uses the field read after that dependency returns. It does not use an earlier snapshot.

If the dependency instead invalidates the receiver, successful dispatch is not established.

### Changing only the first input

For two invocations with identical relevant environmental behavior and identical second inputs, changing only the first input does not change the decisions or arguments represented in the visible body.

This comparison requires the environmental behavior to be held constant; dependencies may otherwise produce different results.

### Invalid lookup state

Given a non-null receiver and nonzero intermediate result, the function proceeds to lookup without visible layout validation. If the receiver, base, or slot is invalid, the contract does not promise a graceful return or a specific error.

## Uncertainties

The following are **documented_only**:

- **Application intent:** The lookup resembles dispatch through an object-associated function table. That is inferred intent, not proof of a particular object system.
- **Receiver layout:** Only the initial 32-bit field and the offset-72 lookup are observed. Other fields are unknown.
- **Address representation:** Native pointer width, target-slot width, endianness, alignment, and address-overflow behavior are not independently established.
- **Lifetime and ownership:** No ownership policy is supplied for either dependency result.
- **Dependency effects:** Neither dependency is established to preserve globals, receiver contents, or memory ownership.
- **Final transfer:** The supplied recovery annotation identifies an indirect jump treated as a call. Exact transfer and return semantics remain unresolved.
- **Failure model:** Zero is a suppression condition, but its meaning as absence, failure, or another state is unknown. No specific fault or exception behavior is established.
- **Concurrency:** No synchronization contract is supplied.

## Evidence

Established static evidence consists of two 32-bit inputs, the unused first input, the ordered dependency calls, the null and zero exits, the receiver-based offset-72 lookup, and the represented two-argument indirect transfer.

The absence of explicit writes, cleanup, allocation, and result propagation describes only the visible body, not its callees.

The supplied recovery annotation limits confidence in the final transfer’s exact call-and-return semantics. No runtime tests have been performed.
