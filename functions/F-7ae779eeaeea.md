<!-- Trusted provenance: F-7ae779eeaeea; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7ae779eeaeea

## Purpose

F-7ae779eeaeea selects between two workflows using a 32-bit mode argument.

- **Mode zero:** perform four initial dependency calls, conditionally invoke operations on a caller-supplied subject, then construct and submit a value.
- **Mode nonzero:** invoke three dependencies, capture a shared pointer, access that shared pointer again for a byte write at offset `+1`, then write another byte at offset `+0` through the captured pointer.

The function has no explicit return value. Its observable behavior consists of dependency calls, shared-memory reads, and two direct byte writes in the nonzero branch.

Application-level meanings are not established. In particular, the material does not establish that these workflows represent initialization, cleanup, navigation, notification, or ownership transfer.

## Interface

### Explicit inputs

| Input | Representation | Use |
|---|---|---|
| Subject | Opaque 32-bit value | Passed unchanged to conditional dependencies in the zero branch. Not directly consumed in the nonzero branch. |
| Mode | Signed 32-bit integer | Compared with zero to select the workflow. |

Every nonzero mode selects the same branch, including negative values. The mode is not otherwise interpreted as a magnitude or bit field.

The function does not directly dereference the subject. Whether it represents an address, handle, identifier, or another value is **documented_only**. Requirements imposed by dependencies receiving it are also **documented_only**.

### Shared inputs

The following descriptive labels identify distinct shared locations without assigning application semantics:

| Shared location | Visible use |
|---|---|
| Construction slot | A 32-bit value read once before the zero-branch construction calls. |
| Submission-base slot | Supplies an address through which a 32-bit value is read at byte offset `+4`, late in the zero branch. |
| Nonzero initial slot | A 32-bit value passed to the first nonzero-branch dependency. |
| Nonzero companion slot | A 32-bit value passed to the third nonzero-branch dependency. |
| Flag-pointer slot | Supplies a pointer captured after the three nonzero-branch calls, and is accessed again to determine the target of the first byte write. |

Two additional incoming machine-context values, each displayed as a 32-bit quantity, are forwarded to zero-branch dependencies. They are not established as explicit parameters of this function. Their ABI significance and validity requirements are **documented_only**.

### Output and memory requirements

There is no explicit status, success, failure, or data return.

Normal completion of the visible accesses requires:

- Any nonzero address returned by F-3f15dcbf790f to support a one-byte read at offset `+0x30`.
- The submission address selected late in the zero branch to support a 32-bit read at offset `+4`.
- The pointer obtained from the second nonzero-branch access to the flag-pointer slot to support a one-byte write at offset `+1`.
- The pointer captured by the preceding access to that slot to support a one-byte write at offset `+0`.
- Every invoked dependency to meet its own input and execution requirements.

The two flag-pointer requirements can concern different addresses. A valid captured pointer does not, by itself, establish validity of the later pointer access.

Only the result of F-3f15dcbf790f receives a visible zero check before its derived memory access. No explicit null, bounds, alignment, or lifetime checks protect the submission read or either flag write.

## Behavior

### Branch selection and completion

The mode comparison precedes the branch-specific calls. Exactly one workflow is selected.

The zero branch returns after submission. It does not continue into the nonzero branch. The nonzero branch returns after the two byte writes.

All later actions described below are conditional on preceding calls returning normally and preceding memory accesses completing.

### Zero branch: initial calls and blocking condition

The initial dependency order is:

1. F-a6e77b5dfc20 returns a value retained as an unsigned 32-bit quantity.
2. F-c0c0b8633623 is invoked; no returned value is explicitly consumed.
3. F-bc4e8f603b51 returns a retained integer result.
4. F-3f15dcbf790f returns a value tested for zero and otherwise used as an address.

The fourth result determines a blocking condition:

- A zero result leaves the condition false and causes no offset-`+0x30` read.
- A nonzero result causes one byte to be read at offset `+0x30`.
- A zero byte leaves the condition false.
- Any nonzero byte makes the condition true.

All four calls occur before the subject-operation gate is decided. A blocking result from an earlier call does not skip a later initial call. Likewise, if F-3f15dcbf790f returns a nonzero address, the byte read occurs even when another gate condition already prevents subject operations.

### Zero branch: conditional subject operations

F-baabc64698a1 receives the original subject only when all three conditions hold:

- The retained result from F-bc4e8f603b51 is zero.
- The upper 16 bits of the retained unsigned 32-bit result from F-a6e77b5dfc20 are not equal to `4`.
- The blocking condition derived from F-3f15dcbf790f is false.

If F-baabc64698a1 is invoked and returns nonzero, F-09d363f7f1ec receives the same original subject. If it returns zero, F-09d363f7f1ec is skipped. If the gate is closed, neither subject dependency is invoked.

The upper-half test excludes the inclusive range `0x00040000`–`0x0004FFFF`. Lower-half differences within this range do not affect exclusion. The comparison uses an unsigned right shift by 16, not a signed shift or a comparison of the complete 32-bit value against `4`.

The inspected byte is reduced to a zero-or-one blocking indicator. Its individual bits have no separately established significance: `0x01`, `0x80`, and `0xFF` all block the gate. The displayed bitwise combination yields the three-condition gate because the blocking indicator is normalized; it does not establish an arbitrary-bit mask interface.

### Zero branch: construction and submission

Construction and submission follow whether the subject operations ran or were skipped.

First, the function samples the 32-bit construction slot. That retained value is used after two intervening calls:

- F-a66a9be680df produces a value.
- F-c5a232e4455b receives that value and the integer constant `12`.
- F-afb630a7107c receives the result from F-c5a232e4455b, the previously sampled construction-slot value, two zero-valued arguments, and one displayed incoming machine-context value.

After F-afb630a7107c returns, the function obtains the submission base from shared storage and reads a 32-bit value at byte offset `+4`. F-9a5c97dc580d then receives:

- The result from F-afb630a7107c.
- The value obtained by that late offset-`+4` read.
- The constants `1`, `0`, and `0`, in that order.
- The other displayed incoming machine-context value.

The function then returns without an explicit value.

The construction-slot sample and submission read have different timing guarantees. Changes to the construction slot during construction calls do not replace the retained sample. Conversely, the function does not establish an earlier snapshot of the submission base or its offset-`+4` contents: those are read after F-afb630a7107c.

No visible validation rejects a zero or unusual intermediate result from F-a66a9be680df, F-c5a232e4455b, or F-afb630a7107c before it is forwarded.

### Nonzero branch: calls and distinct pointer accesses

The nonzero branch performs three dependency calls:

1. Read the nonzero initial slot and pass its 32-bit value to F-416575e983b4.
2. Invoke F-6ac70711bbb5 and retain its result.
3. Read the nonzero companion slot and pass it, together with the retained result, to F-2926dbb9b751.

After the third call returns, the visible memory-access sequence is:

1. Read the flag-pointer slot and retain the pointer obtained.
2. Access the flag-pointer slot again and write byte value `1` at offset `+1` through the pointer obtained by that access.
3. Write byte value `0` at offset `+0` through the pointer retained by the first access.

These are distinct source-visible accesses to the shared pointer. The first write does **not** use the retained pointer merely because that pointer was captured immediately beforehand. The second write does use the retained pointer; it does not require another shared-pointer access.

No dependency call intervenes between the pointer capture and either write. Nevertheless, the supplied material does not establish that asynchronous or concurrent mutation is impossible, nor does it establish a synchronization contract that would make such mutation supported.

If the shared slot supplies the same pointer for both accesses, the writes target offsets `+1` and `+0` of the same buffer. If the accesses supply different pointers, the writes can target different buffers. The subject input is not directly forwarded to any dependency in this branch.

## Side effects

### Direct memory effects and ordering

The only explicit memory writes are the nonzero-branch byte stores:

- First, `1` is written at offset `+1` through the pointer supplied by the second flag-pointer access.
- Then, `0` is written at offset `+0` through the earlier captured pointer.

For a stable pointer and no other writes to the affected bytes, the buffer begins with `0x00, 0x01` after completion. Its source-visible intermediate state has byte `+1` changed to `1` while byte `+0` retains its preceding value.

That same-buffer description is conditional. If the shared pointer changes between capture and the first store, neither a single-buffer intermediate pair nor a single-buffer final pair is generally established. The two destination bytes can also alias; if they do, the later zero store follows the earlier one store.

These are source-visible sequencing statements, not guarantees of atomicity, hardware visibility, or inter-thread ordering. No locking, barriers, atomic operations, or supported concurrent-access rules are established.

### Dependency effects and retained state

No supplied dependency implementation establishes preservation of shared slots or referenced storage. A dependency may therefore affect what a later shared read observes, unless an external contract forbids it.

Retained scalar values are distinguished from later shared reads:

- Initial query results remain the values retained for the gate.
- The construction argument remains the value sampled before construction calls.
- The submission data is obtained after construction.
- The flag-pointer accesses occur after F-2926dbb9b751.

This does not imply that dependencies preserve the objects, handles, or storage represented by retained values.

### Ownership and failure

No direct allocation, deallocation, retain, release, or ownership-transfer operation is established. Passing a value to another dependency does not establish ownership transfer or preservation.

There is no visible retry, rollback, exception handling, or error-reporting path. Zero/nonzero results control specific gates; they are not established as general success/failure conventions.

Behavior after a dependency fault, nonlocal exit, or invalid memory access is **documented_only**. The function does not visibly undo effects already performed.

## Dependencies

Only the following local usage contracts are established:

| Opaque ID | Visible role |
|---|---|
| F-a6e77b5dfc20 | First zero-branch query; its unsigned 32-bit result supplies the upper-half exclusion. |
| F-c0c0b8633623 | Second zero-branch call; no result is explicitly consumed. |
| F-bc4e8f603b51 | Third zero-branch call; a zero result permits its part of the gate. |
| F-3f15dcbf790f | Fourth zero-branch call; a nonzero result is used for the byte read at `+0x30`. |
| F-baabc64698a1 | Conditional subject operation; its nonzero result permits the next operation. |
| F-09d363f7f1ec | Receives the subject only after the gate opens and the preceding operation returns nonzero. |
| F-a66a9be680df | Supplies the first construction-stage value. |
| F-c5a232e4455b | Receives that value and `12`; supplies the next construction-stage value. |
| F-afb630a7107c | Receives the transformed value, retained construction sample, two zeros, and displayed machine context. |
| F-9a5c97dc580d | Receives the constructed result, late-read submission value, three constants, and displayed machine context. |
| F-416575e983b4 | First nonzero-branch call; receives the initial shared value. |
| F-6ac70711bbb5 | Second nonzero-branch call; supplies a value for the third call. |
| F-2926dbb9b751 | Third nonzero-branch call; receives that result and the companion shared value. |

Complete signatures, domain meanings, ownership rules, and effects are **documented_only**.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Unless an example states otherwise, memory is valid, dependencies return normally, and no external agent changes the memory being described.

### Gate opens; both subject operations run

For mode zero, suppose F-a6e77b5dfc20 returns `0x00030000`, F-bc4e8f603b51 returns zero, and F-3f15dcbf790f returns zero.

No offset-`+0x30` read occurs. F-baabc64698a1 receives the subject. If it returns nonzero, F-09d363f7f1ec receives that same subject. Construction and submission follow.

### Gate opens; second subject operation is skipped

Under the preceding gate conditions, a zero result from F-baabc64698a1 skips F-09d363f7f1ec. Construction and submission still follow.

### Upper-half exclusion does not skip initial observations

A first query result of either `0x00040000` or `0x0004FFFF` blocks both subject operations, even with otherwise permissive inputs.

All four initial calls still occur. A nonzero result from F-3f15dcbf790f still causes the byte read at `+0x30`. Construction and submission still follow.

### Other gate exclusions

With the upper-half comparison permissive, a nonzero result from F-bc4e8f603b51 blocks both subject operations. Independently, a nonzero inspected byte such as `0x80` blocks them. Neither exclusion suppresses the subsequent construction sequence.

### Retained sample versus late read

If F-a66a9be680df changes the construction slot, F-afb630a7107c still receives the earlier sample.

If F-afb630a7107c changes the submission base or its offset-`+4` contents before returning, the subsequent submission read uses the later state, subject to validity.

### Stable nonzero-branch pointer

Modes `1` and `-1` both select the three-call nonzero workflow. No zero-branch dependency is invoked.

If both flag-pointer accesses supply the same buffer, initially containing `0x7E, 0x55`, the ordered writes produce the source-visible intermediate pair `0x7E, 0x01`, then `0x00, 0x01`.

### Distinct pointer observations

As a conditional source-access example, suppose the capture obtains buffer A, but the next flag-pointer access obtains a distinct, nonoverlapping buffer B.

The first store targets B at `+1` with `1`. The second targets A at `+0` with `0`. Those stores alone do not change A at `+1` or B at `+0`.

This describes the consequences of distinct observed pointer values; it does not establish that concurrent mutation producing them is supported or well-defined by the execution environment.

## Uncertainties

The following remain **documented_only**:

- Application meaning of the subject, mode, shared slots, inspected byte, and written byte values.
- Complete dependency contracts, including failures, side effects, and preservation of globals.
- Exact ABI treatment of the displayed incoming machine-context values and argument lists.
- Pointer widths, address-space rules, alignment requirements, and object-lifetime guarantees beyond the displayed accesses.
- Whether external contracts guarantee stable flag-pointer observations.
- Ownership, aliasing restrictions, allocation rules, and memory accessibility.
- Endianness of multi-byte stored values; byte-store offsets themselves are explicit.
- Atomicity, concurrency, asynchronous mutation, and hardware visibility.
- Address-overflow behavior or reachability of invalid address calculations.

The unsigned 32-bit right shift by 16 is established and does not require an arithmetic-overflow rule. Memory offsets identify required locations; they do not establish valid wraparound addressing.

Descriptive terms such as “construction,” “submission,” and “blocking” summarize visible relationships. They are not established application intent.

## Evidence

Established behavior comes from static inspection of the supplied function representation: the mode comparison, displayed widths, call order, conditional byte read, normalized gate, retained values, argument forwarding, shared-read timing, and ordered byte stores.

In particular, the nonzero branch shows three dependency calls followed by a pointer capture, a separate shared-pointer access for the offset-`+1` store, and use of the captured pointer for the offset-`+0` store. Treating both writes as necessarily using one pointer would erase a visible access.

No dependency bodies, external contracts, runtime traces, or test results establish additional semantics. Prior prose and reviewer comments are not independent proof of behavior.

No runtime tests have been performed. The acceptance examples are conditional predictions from static evidence, not observed executions.
