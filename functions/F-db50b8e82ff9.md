<!-- Trusted provenance: F-db50b8e82ff9; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-db50b8e82ff9

## Purpose

F-db50b8e82ff9 selects an indexed record, conditionally invokes a sequence of dependent operations involving that record and a globally referenced buffer, publishes the selected index, and returns a Boolean derived from one dependency’s result.

The established return rule is narrow:

- If selection returns exactly −1, return false.
- Otherwise, return true if and only if F-d1a3481e68e2 returns zero.

On a nonzero result from F-d1a3481e68e2, the function additionally saves three bytes, invokes an external three-argument operation, and restores those bytes to a previously captured buffer address.

An inferred intent is to compare or reconcile a selected record with a current buffer while preserving selected buffer fields. That interpretation is not an established contract for the dependencies. In particular, the supplied material does not establish that any dependency performs comparison, copying, locking, allocation, or notification.

## Interface

**Explicit inputs:** None.

**Output:** A Boolean value. Its external calling-convention representation is not established; its logical value follows the rules above.

**Implicit state:**

| State | Role |
|---|---|
| Record-base reference | Supplies the base address used to calculate a selected record address. |
| Buffer reference | Supplies addresses passed to dependencies and used for byte reads. It is read repeatedly rather than captured once for the whole operation. |
| Index-output reference | Supplies the destination for the final selected-index write. |

These names describe roles, not ownership or stability guarantees.

The selected index and the two inspected dependency results are represented as integer values in the supplied material. Their use is consistent with a 32-bit target integer model, but the material does not independently establish the ABI or signed-overflow semantics. The selected index is tested against −1; no other range check appears.

The relevant layout quantities are:

- Record stride: 20 bytes.
- External operation’s third argument: 12.
- Saved and restored byte offsets: 7, 6, and 11, in that order.
- Each saved or restored field: exactly one byte, with no established signed interpretation.

The stride does not establish a complete 20-byte record layout. The value 12 does not, by itself, establish a 12-byte copy.

**Preconditions for defined completion:**

- Every dependency actually reached must return normally.
- Address calculations and pointer uses must be valid under the target execution model.
- Each reached byte read and byte write must access valid storage.
- The final index-output reference must designate writable storage suitable for the selected integer value.
- Dependencies must satisfy their own unknown preconditions.

No ownership transfer, reference counting, allocation, or release protocol is established.

## Behavior

### Selection and the sentinel path

The function initializes its return value to false, then invokes F-222833e3624b with no arguments. The resulting integer becomes the selected index.

If that value is exactly −1, the function skips the entire record-processing sequence. It does not invoke the entry, preparation, evaluation, transfer-like, finalization, or exit dependencies described below.

It still resolves the index-output reference and writes −1 through it before returning false. Thus the sentinel path is not side-effect-free.

Other negative values are not rejected by the visible logic. A value such as −2 enters record processing, subject to valid address arithmetic and dependency preconditions.

### Entry and record-address calculation

For any selected index other than −1, the function invokes F-unmapped-01 with the integer argument 3.

Only after that call returns does it read the record-base reference and calculate the selected record address as the base plus 20 times the selected index.

This ordering matters: if F-unmapped-01 changes the record-base reference, the subsequent calculation uses the changed value.

The selected record address is retained for later calls. The function does not visibly recalculate it after subsequent dependencies return.

The function next invokes:

1. F-3c3879b3cad3 with arguments 1 and the selected record address.
2. F-d1a3481e68e2 with the selected record address and the buffer-reference value read for that invocation.

The record address is calculated before F-3c3879b3cad3, but the buffer address supplied to F-d1a3481e68e2 is obtained after F-3c3879b3cad3 returns. Changes to the buffer reference made by that dependency can therefore affect the evaluation call.

### Evaluation result and destination capture

After F-d1a3481e68e2 returns, the function reads the buffer reference again and captures that address as the restoration destination.

This capture occurs even when the evaluation result is zero and no restoration will follow. It is distinct from the buffer-reference read used for the preceding dependency call.

The function sets its eventual Boolean result according to whether the evaluation result equals zero:

- Zero produces true.
- Every nonzero value produces false.

There is no visible distinction among positive and negative nonzero results.

### Nonzero-result byte preservation

Only when the evaluation result is nonzero, the function performs the following actions in order:

1. Reads one byte at offset 7 from the buffer reference resolved for that read.
2. Reads one byte at offset 6 from the buffer reference resolved for that read.
3. Reads one byte at offset 11 from the buffer reference resolved for that read.
4. Invokes F-unmapped-02 with the then-resolved buffer reference, the retained selected record address, and the integer 12.
5. Writes the first saved byte to offset 7 of the captured restoration destination.
6. Writes the second saved byte to offset 6 of that same destination.
7. Writes the third saved byte to offset 11 of that same destination.

The byte-source reference is not visibly replaced by the earlier captured destination. Each read uses the buffer reference as expressed at that point. The external call also receives a separately resolved buffer-reference value.

Consequently, unconditional claims that all byte reads, the external operation, and the restoration target the same object would be too strong. They do so only under an additional stability condition.

The restoration destination, however, remains the address captured immediately after F-d1a3481e68e2. If F-unmapped-02 changes the global buffer reference, the three restoration writes still target that captured address.

The function itself restores only offsets 7, 6, and 11. It does not visibly restore other bytes, reverse other dependency effects, or reevaluate the Boolean result after the external call.

### Completion and publication

For every non-sentinel selection that reaches normal completion, the function next invokes:

1. F-2d55a2d7c401 with no arguments.
2. F-unmapped-03 with the integer argument 3.

These calls occur whether the evaluation result was zero or nonzero.

After the applicable processing path finishes, the function resolves the index-output reference and writes the original selected index through it. This write occurs after F-unmapped-03 on the non-sentinel path.

The function then returns the already determined Boolean value. The finalization and exit calls do not supply a return value that the function uses to revise that Boolean.

If any dependency does not return normally, the supplied material establishes no recovery, guaranteed later call, guaranteed index publication, or Boolean return.

## Side effects

The function’s directly visible writes are:

- Three one-byte restoration writes on the nonzero-evaluation path.
- One selected-index write on every normally completing path.

Its directly visible memory reads include the role-specific references, the three conditional byte reads, and the reference resolutions needed for address calculation and publication.

Every reached dependency may have additional effects. No dependency is established to preserve global state, preserve the referenced storage, release resources, or refrain from releasing resources.

In particular:

- F-222833e3624b can affect state before either branch is taken.
- F-unmapped-01 can affect the record base before address calculation.
- F-3c3879b3cad3 can affect the buffer reference before evaluation.
- F-d1a3481e68e2 can affect the buffer reference before the restoration destination is captured.
- F-unmapped-02 can affect the global reference without changing the already captured restoration address.
- The finalization and exit dependencies can affect the index-output reference before the final write.

The equal integer arguments supplied to the entry and exit dependencies do not prove that they form a lock, transaction, or balanced resource-management protocol.

## Dependencies

The following identifiers are opaque. The unmapped identifiers are document-local labels for external dependencies whose identities were not supplied.

| Function ID | Invocation and established use |
|---|---|
| F-222833e3624b | No arguments. Its integer result selects the path, participates in record-address calculation, and is ultimately published. |
| F-unmapped-01 | Receives 3, only for a selection other than −1. Returns before the record-base reference is read. |
| F-3c3879b3cad3 | Receives 1 and the selected record address. No returned value is used visibly. |
| F-d1a3481e68e2 | Receives the selected record address and a resolved buffer reference. Its zero/nonzero result determines the Boolean and conditional preservation path. |
| F-unmapped-02 | Receives a resolved buffer reference, the selected record address, and 12. Reached only for a nonzero evaluation result. Its return value is not visibly used. |
| F-2d55a2d7c401 | No arguments. Reached after either evaluation outcome on a normally progressing non-sentinel path. |
| F-unmapped-03 | Receives 3 after F-2d55a2d7c401 and before index publication. |

F-unmapped-02 has a transfer-like argument shape. Treating it as a byte-copy operation remains an inference, not established behavior.

## Acceptance examples

These are conditional contract examples, not executed tests. Each assumes valid storage, valid arithmetic, and normal dependency returns unless stated otherwise.

### Sentinel selection

If F-222833e3624b returns −1:

- No processing dependency is invoked.
- The final index-output destination receives −1.
- The function returns false.

### Zero evaluation result

If selection returns 2 and evaluation returns zero:

- The selected address uses a displacement of 40 bytes from the record base read after F-unmapped-01.
- F-3c3879b3cad3 and F-d1a3481e68e2 are invoked.
- The restoration destination is captured.
- No preservation-byte reads, F-unmapped-02 call, or restoration writes occur.
- Finalization and exit occur before publication of 2.
- The function returns true.

### Nonzero evaluation result with a stable buffer reference

If the buffer reference remains stable and evaluation returns nonzero, let the bytes read at offsets 7, 6, and 11 be A, B, and C.

After F-unmapped-02 returns, the function writes A, B, and C back to those offsets, respectively. This remains true even if that dependency altered those bytes, provided the captured destination remains writable.

The return value is false. No guarantee follows for any other byte.

### Buffer-reference change during evaluation

If evaluation changes the buffer reference from one address to another, the restoration destination is captured from the new value after evaluation returns. It is not necessarily the address passed into evaluation.

### Buffer-reference change during the external operation

If F-unmapped-02 changes the buffer reference, restoration still uses the previously captured destination. The function does not redirect those writes to the new reference.

### Other negative selection

If selection returns −2, the function does not take the sentinel path. Processing uses a mathematical displacement of −40 bytes, subject to the target’s arithmetic rules and address validity. Safe acceptance is not established merely because −2 differs from −1.

## Uncertainties

The following remain **documented_only**:

- Exact ABI widths, Boolean representation, and machine-level arithmetic behavior.
- Overflow behavior for multiplying the selected index by 20 or adding that displacement to the base.
- Valid index bounds and record count.
- Full record and buffer layouts.
- Meaning of the integer arguments 1 and 3.
- Semantics, ownership effects, and failure behavior of every dependency.
- Whether F-unmapped-02 copies bytes, permits overlap, or interprets 12 as a byte count.
- Synchronization, atomicity, concurrent mutation, and volatile-access properties.
- Aliasing among records, buffer storage, index-output storage, and state holding their references.
- Behavior after invalid memory access or abnormal dependency termination.

No concurrent or asynchronous stability guarantee should be inferred from the visible sequence. Likewise, no non-aliasing guarantee permits disregarding the final index write or dependency effects when reasoning about final memory contents.

## Evidence

**Established static evidence:** The supplied function representation supports the branch conditions, dependency-call order, argument values, 20-byte stride, post-evaluation destination capture, ordered byte accesses, final index publication, and Boolean derivation described above.

**Inferred intent:** The transfer-shaped external invocation and selective restoration suggest record reconciliation with preservation of three fields. Entry and exit calls with matching arguments suggest paired control operations. Neither interpretation establishes dependency semantics.

**Unknown:** Dependency implementations, complete data layouts, ABI details, runtime validity, and environmental state are unavailable.

No runtime tests have been performed. The acceptance examples state conditional consequences of the supplied static evidence, not observed execution results.
