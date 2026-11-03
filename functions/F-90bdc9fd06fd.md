<!-- Trusted provenance: F-90bdc9fd06fd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-90bdc9fd06fd

## Purpose

F-90bdc9fd06fd performs a fixed sequence of storage initialization, two operations on a supplied object or context, and a dependent operation using a value subsequently read from that context. An exact failure sentinel from the dependent operation selects an additional indirect dispatch.

The established behavior is the ordering and data flow of these operations. The semantic purpose of the allocated storage, the context, and the final dispatch is not established. In particular, the available evidence does not establish that the allocation is attached to the context, that the dependent operation creates a resource, or that the indirect dispatch performs cleanup.

This contract describes static evidence only. No runtime tests have been performed.

## Interface

### Explicit inputs

The recovered interface has four explicit inputs:

| Input | Representation | Established use |
|---|---|---|
| Context address | Address-sized value represented as a 32-bit integer in the recovered interface | Passed unchanged to two dependencies, then used as the base for a 32-bit read at byte offset 32 |
| Initialization value | Opaque 32-bit value | Written into allocated storage at byte offset 4 |
| First forwarded value | Opaque 32-bit value | Forwarded unchanged to F-9813035c7ce7 |
| Second forwarded value | Opaque 32-bit value | Forwarded unchanged to F-9813035c7ce7 |

The initialization and forwarded values have no established signedness or internal structure. Their complete 32-bit representations matter; no conversion, masking, or validation is visible.

The context address is not merely an opaque token: after the two context dependencies return, the function directly reads memory relative to it.

### Additional incoming machine state

The recovered invocation of F-9813035c7ce7 also includes two 32-bit values originating from incoming machine state, beyond the four explicit inputs. The available evidence does not establish these as ordinary caller-supplied arguments of F-90bdc9fd06fd, nor does it establish their semantic meaning.

Their appearance may reflect calling-convention details or imperfect recovery of the dependent call. They must remain documented dependencies rather than being replaced with invented values or promoted to a definitive public interface.

### Result

No explicit return value is established. The visible control flow returns after either:

- F-9813035c7ce7 produces a result other than the failure sentinel; or
- the failure sentinel is produced and the indirect dispatch returns.

No dependency result is explicitly returned to the caller. Incidental machine-register contents are not an established output contract.

### Preconditions for the visible accesses

For the sequence to proceed normally:

- F-8157a180d901 must return an address permitting the first two 32-bit writes described below.
- Both context dependencies must return before the subsequent field read can occur.
- The context address plus byte offset 32 must designate readable 32-bit storage at the time of that read.
- On the sentinel path, the saved field value plus byte offset 4 must designate readable 32-bit storage.
- The indirect dispatch target must be callable under the applicable machine-level convention.

The function does not visibly check these conditions. Alignment, address-space validity, and exception behavior are not established by the supplied evidence.

## Behavior

### 1. Request storage

The first visible dependency call is to F-8157a180d901 with the numeric argument 12.

The returned value is immediately used as a storage address. There is no visible null check, failure-sentinel check, or alternate allocation-failure path before the writes.

The use strongly suggests a request for 12 bytes of storage, but the dependency’s actual allocation contract is unknown. The established facts are the argument value and the subsequent accesses.

### 2. Initialize two words in order

Using the returned address as the base, the function performs these writes:

| Byte offset | Width | Value |
|---|---|---|
| 0 | 32 bits | Zero |
| 4 | 32 bits | The initialization input, unchanged |

The write at offset 0 precedes the write at offset 4. Both precede the first context dependency.

No direct write to bytes 8 through 11 is visible. If F-8157a180d901 provides a 12-byte region, this function explicitly initializes only its first eight bytes. The remaining four bytes are not established to be zero, preserved, or otherwise valid for interpretation.

No subsequent visible operation reads from the returned storage address. The address is not explicitly passed as an argument to either context dependency, F-9813035c7ce7, or the indirect dispatch.

This absence of explicit forwarding does not prove the storage is unreachable: F-8157a180d901 could publish it through state not shown here.

### 3. Invoke the context dependencies

After both storage writes, the function calls:

1. F-bf0b7433249b with the context address.
2. F-583c269a6d82 with the same context address.

The second call occurs only after the first returns. Neither result is used in the visible logic, and no visible status check occurs between them.

These dependencies may alter the context or other state. Consequently, the later field read must not be described as reading the context’s entry-time value.

### 4. Read and save the context field

After both context dependencies return, the function reads a 32-bit value from byte offset 32 of the context.

This value is saved and used in two ways:

- as an input to F-9813035c7ce7;
- as an address base on the conditional indirect-dispatch path.

Only one read of this context field is visible. The function does not visibly reload it after F-9813035c7ce7 returns.

Thus, a later change to the context field does not, by itself, replace the saved base used by this invocation. The validity and contents of memory reachable through that base may nevertheless change during the dependency call.

### 5. Invoke the dependent operation

The function calls F-9813035c7ce7 with the recovered argument sequence:

1. Numeric value 1.
2. The saved 32-bit context field.
3. The first forwarded input.
4. The second forwarded input.
5. The first additional incoming machine-state value.
6. The second additional incoming machine-state value.

The visible logic does not modify the two explicit forwarded inputs before this call.

The recovered result is compared for exact equality with −1. In the represented 32-bit model, that sentinel has all 32 bits set. This is not a general negative-result test.

### 6. Select the return or indirect-dispatch path

If the result is anything other than −1, the function returns without the final indirect dispatch. This includes zero and, under the recovered signed 32-bit interpretation, negative values other than −1.

If the result equals −1:

1. A 32-bit value is read at byte offset 4 from the saved context-field value, treating that saved value as an address base.
2. An indirectly selected target is invoked with that word as its sole recovered argument.
3. If the indirect invocation returns normally, F-90bdc9fd06fd returns.

The dispatch target comes from a separately referenced function-pointer location. It is not established to be a method pointer inside the saved-base object.

The offset-4 read occurs after F-9813035c7ce7 returns. Its value therefore reflects memory at that later point, not necessarily memory as it existed before the call.

No retry, explicit error return, or further visible operation follows the indirect dispatch.

## Side effects

### Direct memory effects

The directly established memory operations, in program order, are:

1. A 32-bit zero write at the storage address returned by F-8157a180d901.
2. A 32-bit write of the initialization input at that address plus 4.
3. After both context dependencies, a 32-bit read at the context address plus 32.
4. Only on the sentinel path, a 32-bit read at the saved base plus 4.

The indirect dispatch additionally depends on a function-pointer reference. Its precise load timing and low-level control-transfer form are not sufficiently established to promise more detailed memory ordering.

No direct write to the context or to the saved-base object is visible. Dependencies may perform such writes.

These are sequential control-flow observations, not concurrency guarantees. The evidence does not establish atomicity, synchronization, memory barriers, or behavior under concurrent mutation.

### Dependency effects

All four named dependencies and the indirect target may have additional effects not described here, including changes to shared state or memory accessible through the context.

The contract does not assume that any dependency:

- preserves globals;
- leaves the context unchanged;
- leaves the saved-base object allocated;
- owns, retains, or releases the newly obtained storage;
- returns normally on every input.

### Ownership and lifetime

There is no explicit release of the newly obtained storage in the visible function. There is also no explicit ownership transfer.

Calling this a definite leak would exceed the evidence because the storage-providing dependency may retain or publish the address. Calling it a successful ownership transfer would likewise be unsupported.

The saved context field is used as an address on the sentinel path without a visible retention or lifetime-protection operation. Whether it remains valid across F-9813035c7ce7 is an external requirement or an unknown dependency property.

### Errors and interruption

The only explicit status-dependent branch checks the exact −1 result from F-9813035c7ce7.

No visible handling is provided for invalid addresses, storage-acquisition failure, faults in dependencies, or an invalid indirect target. The evidence does not establish whether such conditions trap, terminate execution, unwind, or behave otherwise.

## Dependencies

| Opaque ID or role | Established interaction | Unestablished semantics |
|---|---|---|
| F-8157a180d901 | Called first with 12; result used for two 32-bit writes | Allocation policy, initialization, failure convention, publication, ownership |
| F-bf0b7433249b | Called with the context after both storage writes | Purpose, state changes, failure behavior |
| F-583c269a6d82 | Called with the context after F-bf0b7433249b returns | Purpose, state changes, failure behavior |
| F-9813035c7ce7 | Called after the context-field read; receives the argument sequence above; result compared with −1 | Operation semantics, full calling convention, mutation, ownership effects |
| Indirect dispatch target; opaque ID unavailable | Invoked only for the −1 result, with the late-read offset-4 word | Target identity, purpose, calling convention, return behavior, resource effects |

No dependency is established as interchangeable with another. The order of the two context calls is part of the observed behavior.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Each assumes valid required memory and normally returning dependencies unless stated otherwise.

### Storage initialization preserves the input bits

Given a returned storage region and an initialization input of `0xA1B2C3D4`:

- The word at offset 0 is written as zero.
- The word at offset 4 is written with `0xA1B2C3D4`.
- The function performs no direct write to the word at offset 8.
- Both writes precede F-bf0b7433249b.

A byte-by-byte ordering for the nonzero word cannot be specified without an established byte order.

### Context mutation precedes field selection

Suppose the context field at offset 32 initially contains one address, but the two context dependencies leave a different address there.

F-9813035c7ce7 receives the address present after both dependencies return. Reading the field before either call would not satisfy the established sequence.

### A non-sentinel result skips dispatch

If F-9813035c7ce7 returns zero, the function returns without the offset-4 argument read or final indirect dispatch.

The same branch applies to any other represented result except −1. A signed result of −2 does not select the dispatch path.

### The sentinel selects a late argument read

Suppose the saved base is valid and F-9813035c7ce7 returns −1. If the word at saved-base offset 4 was changed during that call, the indirect target receives the value present when the subsequent read occurs.

Using a copy obtained before F-9813035c7ce7 would not match the visible behavior.

### Changing the context field does not replace the saved base

Suppose F-9813035c7ce7 changes the context’s offset-32 field to another address and returns −1.

The function still uses the previously saved base for its offset-4 read. This example requires that the old base remain readable; the function supplies no visible protection if the dependency invalidates it.

### Invalid storage is not converted into a status result

If F-8157a180d901 returns an address unsuitable for the first write, there is no visible guard that converts this condition into a clean failure return. The resulting machine behavior is outside the established contract.

## Uncertainties

The following items are **documented_only**:

- **Storage semantics:** Whether the request for 12 denotes allocation, arena acquisition, or another storage mechanism.
- **Ownership:** Who retains or releases that storage, and whether it is published through hidden state.
- **Object layouts:** The meanings of context offset 32 and saved-base offset 4.
- **Incoming machine state:** Whether the two extra recovered arguments are true semantic inputs or recovery artifacts.
- **Indirect dispatch:** The target’s identity and whether the low-level transfer is an ordinary call, a tail transfer, or another equivalent mechanism. The available recovery explicitly indicates uncertainty around this transfer.
- **Integer and address model:** The 32-bit model is supported by the recovered types and word accesses, not by an independently supplied ABI specification.
- **Address arithmetic:** No overflow check is visible. Wraparound, trapping, and validity consequences are not established.
- **Byte order and alignment:** Neither is independently specified.
- **Dependency failures and concurrency:** No complete exception, synchronization, or concurrent-access contract is available.

Inferred intent is limited to describing the first dependency as storage acquisition and the −1 branch as failure-related dispatch. Neither inference establishes allocation ownership or cleanup semantics.

## Evidence

The supplied static representation establishes:

- the four explicit recovered inputs and absence of an explicit result;
- the initial dependency argument of 12;
- the ordered 32-bit writes at offsets 0 and 4;
- the ordered context calls;
- the subsequent 32-bit context read at offset 32;
- the dependent call’s recovered argument sequence;
- an exact comparison against −1;
- the conditional offset-4 read and indirect dispatch.

The representation also flags uncertainty in recovery of the final indirect control transfer. This contract therefore preserves its visible condition and argument data flow without asserting a fully recovered dispatch ABI.

No runtime observations, dependency implementations, independent layout definitions, or ABI documentation were supplied. No runtime tests have been performed.
