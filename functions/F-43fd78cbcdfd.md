<!-- Trusted provenance: F-43fd78cbcdfd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-43fd78cbcdfd

## Purpose

F-43fd78cbcdfd conditionally repeats a two-dependency operation for one selected record. It first checks the record’s one-byte gate, calls F-dd3335123e5a with a value obtained from a caller-supplied object and a forwarded argument, and, when that dependency returns nonzero, calls F-fceaba74ea88 with a value obtained from the selected record.

The function returns when either:

- The selected record’s gate is zero at the beginning of an iteration.
- F-dd3335123e5a returns zero.

The record selection does not advance between iterations. Continued operation therefore depends on repeated dependency results and repeated reads of the same record’s gate.

This is a static behavioral contract. The application-level meanings of the gate, the object field, and the record field are unknown. Descriptions such as “gate” and “record” identify observable roles, not established domain types.

## Interface

### Inputs

The recovered interface has three arguments:

| Argument | Representation | Observable use |
|---|---|---|
| Object address | Recovered as a 32-bit integer/address value | Supplies the address of a four-byte field at byte offset `0x34`. That field is read before each call to F-dd3335123e5a. |
| Forwarded value | Opaque four-byte value | Passed unchanged as the second argument to every invocation of F-dd3335123e5a. No interpretation or local dereference is shown. |
| Record selector | Recovered as a signed 32-bit integer | Multiplied by 24 and added to a base address read from shared storage to select one record. |

The recovered integer declarations support a 32-bit interpretation, but they do not by themselves establish the original source-language types or the machine’s complete address-arithmetic semantics.

### Return

The function has no explicit return value. Both normal exit conditions return without supplying a result or status code.

A zero result from F-dd3335123e5a is used only to stop processing. It is not propagated as a return value.

### Relevant memory layout

The function uses the following byte-oriented layout:

| Storage | Offset or extent | Use |
|---|---|---|
| Shared base-address storage | One address-sized value | Read to obtain the base used for initial record selection. |
| Selected record | 24-byte selection stride | The selector chooses a record address by scaling by 24. The complete record layout is not established. |
| Selected record | Offset `0x04`, four bytes | Read as the argument to F-fceaba74ea88 after a nonzero result from F-dd3335123e5a. |
| Selected record | Offset `0x13`, one byte | Read as a zero/nonzero gate at the start of each iteration. |
| Caller-supplied object | Offset `0x34`, four bytes | Read as the first argument to F-dd3335123e5a. |

The stride does not prove that the selected storage belongs to a bounded array, that records are independently allocated, or that every byte in a 24-byte span is meaningful.

### Preconditions and validity

For defined operation, the shared base-address storage must be readable. The selected record’s gate must be readable whenever an iteration begins.

Other memory requirements are conditional:

- The object’s four-byte field must be readable whenever the gate is nonzero.
- The record’s four-byte field must be readable whenever F-dd3335123e5a returns nonzero.
- Values passed to either dependency must satisfy that dependency’s own requirements, which are not established here.

There are no visible checks for null addresses, selector bounds, alignment, allocation lifetime, or arithmetic overflow. An initially zero gate avoids dereferencing the object, but it does not avoid record selection or the gate read.

## Behavior

### Record selection

Before the first gate check, the function obtains the shared base address and selects a record at a displacement of 24 times the third argument.

The selected address is retained for the duration of this invocation. The function does not reload the shared base address on later iterations and does not recompute record selection after dependency calls.

Consequently, a dependency changing the shared base-address storage does not redirect this invocation to a different record. Changes to the contents at the already-selected address can still affect later reads.

### Iteration and access order

Within the function’s own direct operations, each iteration has the following observable order:

1. Read the one-byte gate at selected-record offset `0x13`.
2. If the byte is zero, return normally.
3. Otherwise, read the four-byte field at object offset `0x34`.
4. Call F-dd3335123e5a with that freshly read field as its first argument and the original forwarded value as its second argument.
5. If the dependency returns zero, return normally.
6. Otherwise, read the four-byte field at selected-record offset `0x04`.
7. Call F-fceaba74ea88 with that freshly read value.
8. If that call returns normally, begin another iteration by reading the same gate again.

This ordering matters when dependencies modify accessible memory:

- The object field is read again for each invocation of F-dd3335123e5a.
- The record field passed to F-fceaba74ea88 is read **after** F-dd3335123e5a returns.
- The gate is not rechecked between a nonzero result from F-dd3335123e5a and the call to F-fceaba74ea88.
- The next gate read occurs only after F-fceaba74ea88 returns normally.

Dependency-internal reads and writes may occur between these direct operations. Their order and effects are outside the available evidence.

### Branch interpretation

Only zero versus nonzero is significant for both decisions.

For the one-byte gate, every nonzero byte permits the iteration to proceed. The signedness of the recovered character type does not change that zero/nonzero distinction.

For the recovered 32-bit result of F-dd3335123e5a, any nonzero value permits the subsequent call to F-fceaba74ea88. There is no separate positive-value requirement, negative-result handling, retry category, or error-code classification.

No result from F-fceaba74ea88 is inspected by this function.

### Repetition and termination

There is no visible iteration limit, selector increment, local progress counter, delay, or record-address update.

If the gate remains nonzero, F-dd3335123e5a keeps returning nonzero, and both dependencies keep returning normally, the function continues indefinitely. Each completed iteration then contains one call to each dependency.

The function can also fail to return because a dependency does not return. The available evidence does not establish whether either dependency can block, terminate execution, transfer control nonlocally, or raise an exception.

### Arithmetic boundaries

The record displacement uses a stride of `0x18`, or 24 bytes. Field offsets are byte offsets; the four-byte values are not used as additional indices by this function.

The recovered signed selector permits negative values syntactically. For ordinary representable arithmetic, a negative selector selects storage before the base address. No rejection of such a selector is visible.

The exact result of overflowing the selector multiplication or the subsequent address addition is **documented_only**. The supplied representation does not establish whether the original implementation relies on machine-width wrapping, a source-language overflow rule, or a restricted caller contract. Validity of a wrapped address must not be assumed.

## Side effects

### Direct effects

No explicit write to caller memory, record memory, or shared base-address storage is present.

Directly observable actions are memory reads, dependency invocations, repetition, and return. The function itself does not visibly allocate memory, release memory, update the gate, clear a record, or advance a queue.

The absence of local writes does not make the invocation side-effect-free: both dependencies may have effects that are not described by this function’s body.

### Ownership and lifetime

No local ownership transfer is established. Passing either four-byte field to a dependency does not prove that it is a pointer, a handle, an owned resource, or a borrowed reference.

In particular, F-fceaba74ea88 must not be characterized as a destructor, release operation, or deallocator solely from its use here.

The function retains the selected address across dependency calls. If a dependency invalidates the selected storage, a later gate or field read may access invalid memory. Likewise, the object address is reused for later object-field reads. No lifetime protection, reference acquisition, or revalidation is visible.

### Shared state and concurrency

Dependencies may change the gate, object field, record field, or shared base-address storage. This contract does not assume that unknown callees preserve globals or reachable memory.

No synchronization, atomic operation, memory barrier, or concurrent-access guarantee is established. The repeated reads described here are the accesses represented by the supplied body, not a guarantee that unsynchronized changes from another thread will be observed safely.

### Errors

No explicit error-reporting mechanism is visible.

A zero result from F-dd3335123e5a causes ordinary return, without an additional call to F-fceaba74ea88 for that iteration. Whether zero means failure, exhaustion, cancellation, or normal completion is unknown.

Invalid memory accesses and dependency failures are not caught or translated by any visible local mechanism.

## Dependencies

### F-dd3335123e5a

- Called only after a nonzero gate read.
- Receives two four-byte arguments:
  - The current value of the object field at offset `0x34`.
  - The unchanged second argument of F-43fd78cbcdfd.
- Supplies a recovered 32-bit result used solely for a zero/nonzero decision.
- A zero result ends the current invocation.
- A nonzero result leads to a fresh record-field read and a call to F-fceaba74ea88.

Its interpretation of arguments, memory effects, ownership behavior, global-state effects, and termination properties are unknown.

### F-fceaba74ea88

- Called only after F-dd3335123e5a returns nonzero.
- Receives the four-byte value read from selected-record offset `0x04` after that return.
- Has no result consumed by this function.
- On normal return, control proceeds to another gate check.

Its argument semantics and effects are unknown. Nothing here establishes that it changes the gate or guarantees progress.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Unless stated otherwise, referenced memory is valid and each invoked dependency returns normally.

### Initially zero gate

Given a selected record whose gate is zero at the first check:

- The function returns.
- Neither dependency is called.
- The object field and selected record’s four-byte argument field are not read by this function.
- Initial base acquisition and the gate read still occur.

### Nonzero gate followed by zero dependency result

Given a nonzero gate and an object field containing value A, if F-dd3335123e5a returns zero:

- It is called once with A and the forwarded value.
- F-fceaba74ea88 is not called.
- The record field at offset `0x04` is not read by this function.
- There is no second gate check.

### Gate cleared by the first dependency

Given an initially nonzero gate, suppose F-dd3335123e5a clears the gate but returns nonzero:

- F-fceaba74ea88 is still called after a fresh record-field read.
- The cleared gate is observed only on the next iteration.
- If it remains zero, that next check returns without another dependency call.

### Record field changed by the first dependency

Suppose the record field initially contains B, and F-dd3335123e5a changes it to C before returning nonzero:

- F-fceaba74ea88 receives C, provided the field remains valid and contains C when read.
- The function does not use a snapshot of B taken before the first dependency call.

### Object field changed between iterations

Suppose the first call to F-dd3335123e5a receives A, and a dependency changes the object field to D before the next iteration:

- If the next gate check is nonzero, the next call receives D.
- The forwarded second argument remains unchanged.

### Nonzero result other than one

If F-dd3335123e5a returns a negative recovered 32-bit value, or another nonzero value:

- F-fceaba74ea88 is called.
- No special error branch is taken solely because of that result’s sign or magnitude.

### Shared base changed during a call

If a dependency changes the shared base-address storage:

- Subsequent iterations still use the originally selected record address.
- The changed base may affect another invocation, but does not trigger reselection here.

### No progress condition

If the gate always remains nonzero and F-dd3335123e5a always returns nonzero:

- The two dependency calls repeat in the established order.
- There is no normal local exit merely because many iterations have completed.

## Uncertainties

The following are **documented_only**:

- Original source-language parameter types, calling convention, and precise overflow semantics.
- Valid selector range, record count, and allocation boundaries.
- Whether the four-byte dependency arguments represent integers, pointers, handles, or another encoding.
- Application-level meanings of the gate and dependency results.
- Ownership, allocation, release, and global-state behavior inside either dependency.
- Thread-safety and asynchronous mutation rules.
- Whether dependencies always return, or can invalidate memory retained by this function.
- Complete object and record layouts, including unused fields.
- Alignment and byte-order requirements of the target environment.

An inferred intent is to repeat a gated operation on one selected record until the gate clears or the first dependency returns zero. That interpretation does not establish queue processing, resource release, or eventual termination.

## Evidence

Established static evidence consists of the supplied recovered function body and opaque dependency mapping. It supports:

- One initial base-address acquisition and record selection using a 24-byte stride.
- Repeated one-byte gate reads at offset `0x13`.
- Conditional four-byte object-field reads at offset `0x34`.
- A zero/nonzero decision on F-dd3335123e5a’s result.
- Post-result four-byte record-field reads at offset `0x04`.
- Calls to F-fceaba74ea88 only on the nonzero-result path.
- No visible local memory writes, record advancement, or iteration bound.

No runtime tests have been performed. Dependency implementations and external callers were not available for establishing stronger guarantees.
