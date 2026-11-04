<!-- Trusted provenance: F-4290fdeada83; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4290fdeada83

## Purpose

F-4290fdeada83 selects a candidate object from a linked record structure and submits that candidate to F-dd82aab5ab38. It returns the selected candidate pointer if that dependency returns a nonzero result. Otherwise, it returns a null pointer and, on a dependency-reported failure, updates a context-associated failure count.

Traversal is deliberately narrow: the function skips only records whose tag is either `0x1F` or `0x56` and whose candidate pointer is null. A record containing a candidate stops traversal regardless of its tag. A record with any other tag also stops traversal, even when its candidate pointer is null.

**Inferred intent:** the diagnostic reference suggests that the candidate concerns a collation sequence. The supplied evidence does not establish the candidate’s complete type or the operation performed by F-dd82aab5ab38.

## Interface

### Inputs

The function receives two pointers:

1. **Context pointer.** A pointer to storage addressed in four-byte units. The function uses:
   - The four-byte word at byte offset `0`.
   - The four-byte word at byte offset `64`, corresponding to word index `16`.

2. **Starting-record pointer.** A nullable pointer to a record with the following accessed fields:

| Byte offset | Field | Access |
|---|---|---|
| `0` | One-byte tag | Read to determine whether an empty record permits traversal |
| `4` | Candidate pointer | Read from each visited non-null record |
| `8` | Next-record pointer | Read only when the tag permits traversal and the candidate pointer is null |

For a non-null candidate, the function also reads the candidate’s first four-byte word, at candidate-relative byte offset `0`.

The offsets above are established by the supplied representation. The pointer-bearing fields and spacing are consistent with a 32-bit target layout. The representation does not independently establish a portable pointer size, alignment requirement, endianness, or complete structure size. These offsets must not be reinterpreted as a host-native structure layout without establishing the target ABI.

### Output

The result is a pointer:

- The selected candidate pointer when F-dd82aab5ab38 returns nonzero.
- Null when traversal reaches a null record, selection stops without a candidate, or F-dd82aab5ab38 returns zero.

A successful result is the candidate pointer saved during selection. It is not a fresh read of the record’s candidate field after the dependency call.

### Preconditions

Required storage depends on the path taken:

- A null starting-record pointer is accepted without dereferencing the context.
- Every visited non-null record must support the candidate-pointer read at offset `4` and the tag read at offset `0`.
- A visited record needs a readable next-pointer field at offset `8` only if its tag permits traversal and its candidate is null.
- Once a candidate is selected, the context’s first four-byte word and the candidate’s first four-byte word must be readable.
- If F-dd82aab5ab38 returns zero, the context word at offset `64` must remain readable and writable through the subsequent failure handling.

The function performs no explicit null check on the context and no validation of record, candidate, or context storage. Invalid storage is outside its established normal-return behavior.

### Ownership

No allocation, release, copy of the candidate object, or ownership transfer is directly shown.

Returning a candidate pointer does not establish that the caller owns it, that it is newly created, or that it remains valid after the call. The dependencies’ effects on candidate lifetime and ownership are unknown.

## Behavior

### Selection and traversal

The function first tests the current record pointer for null. A null pointer causes an immediate null return.

For a non-null record, it reads and saves the candidate pointer from byte offset `4`. That read precedes the tag-based decision in the supplied representation.

Selection then follows these rules:

| Tag | Candidate pointer | Result |
|---|---|---|
| `0x1F` or `0x56` | Null | Read the next-record pointer at offset `8` and continue |
| `0x1F` or `0x56` | Non-null | Select this candidate |
| Any other byte value | Null | Return null |
| Any other byte value | Non-null | Select this candidate |

Thus, the two special tags permit traversal through an empty record; they do not cause traversal past a record that already has a candidate.

The tag is a single byte. The recognized values are unaffected by whether the target interprets plain character values as signed or unsigned.

There is no traversal bound or cycle detection. A stable cycle consisting entirely of special-tag records with null candidates does not reach a normal return. A non-special-tag record with a null candidate ends selection immediately; any record beyond it is ignored.

### Candidate processing

Once a non-null candidate is selected, the function invokes F-dd82aab5ab38 with four arguments, in this positional order:

1. The context’s four-byte word at offset `0`.
2. The selected candidate pointer.
3. The candidate’s first four-byte word.
4. The 32-bit all-ones value, `0xFFFFFFFF`.

The fourth argument’s bit pattern is established. Whether the dependency interprets it as signed `−1`, an unsigned maximum, a flag set, or another sentinel is unknown.

The context-word and candidate-word reads both supply arguments to this call. Their relative evaluation order is not established by the supplied call expression. No contract should require one of those two reads to precede the other.

The dependency’s result is tested only for zero versus nonzero:

- A nonzero result causes the function to return the saved candidate pointer immediately.
- A zero result enters failure handling.

There is at most one call to F-dd82aab5ab38 per invocation of this function. Failure does not cause traversal to resume at the next record, and no alternative candidate is tried.

### Failure handling

After F-dd82aab5ab38 returns zero, the function reads the context’s four-byte word at offset `64`.

- If that word is zero, it calls F-2022b0f73f3b with the context pointer and a fixed diagnostic-string address.
- If that word is nonzero, it skips that diagnostic call.

The function then increments the context word at offset `64` and returns null.

The diagnostic dependency is invoked before the increment. Consequently, the zero check and the increment need not operate on the same value: F-2022b0f73f3b may change the context word before the increment expression reads it. Its result, if any, is not used.

The counter update is a four-byte update. Under the target’s 32-bit word-arithmetic interpretation, it adds one modulo \(2^{32}\), so `0xFFFFFFFF` becomes zero. No saturation, overflow diagnostic, or wider stored result is shown. The supplied representation does not establish a separate source-language overflow guarantee beyond the recovered word operation.

The diagnostic is therefore gated by the counter’s value at this particular check, not by an independently maintained “already reported” flag. Counter wraparound, dependency mutation, or external changes can make a later invocation encounter zero again.

## Side effects

### Direct effects

The only explicit write by F-4290fdeada83 is to the context word at byte offset `64`, after F-dd82aab5ab38 returns zero and after any diagnostic call returns normally.

No direct writes to record tags, record links, candidate pointers, candidate contents, or the context’s first word are shown.

These early exits perform no direct writes and invoke neither dependency:

- The starting record is null.
- Traversal reaches a null next-record pointer.
- A non-special-tag record has a null candidate.

A nonzero result from F-dd82aab5ab38 bypasses this function’s diagnostic and counter-update path. It does **not** establish that the call had no side effects.

### Dependency-visible effects

F-dd82aab5ab38 receives the candidate pointer and two four-byte values read from caller-associated storage. F-2022b0f73f3b receives the context pointer and a diagnostic-string address.

Neither dependency’s implementation is supplied. They may change reachable memory, globals, error state, allocation state, or object lifetime. This function must not be specified as preserving such state merely because it contains no corresponding direct write.

In particular:

- The failure counter is checked after F-dd82aab5ab38 returns, so that check observes any changes the dependency made to the counter.
- The increment occurs after F-2022b0f73f3b returns, so it can incorporate changes made by that dependency.
- A successful return uses the previously saved candidate pointer even if F-dd82aab5ab38 changed the record’s candidate field.
- Returning that pointer is not proof that the dependency preserved the candidate allocation.

If a dependency does not return normally, subsequent operations and the function’s normal return are not established.

### Concurrency and aliasing

No locks, atomic operations, synchronization, or reentrancy protection are shown.

The evidence establishes no disjointness requirement between the context, records, candidate storage, and dependency-accessible memory. Aliasing can make writes visible through more than one role. In particular, a counter write could affect record or candidate bytes if the caller supplies overlapping storage.

Concurrent mutation requires an external synchronization contract. The supplied evidence does not establish coherent snapshots or race-safe counter updates.

## Dependencies

| Opaque ID | Invocation condition | Arguments | Result handling |
|---|---|---|---|
| F-dd82aab5ab38 | A non-null candidate has been selected | Context word at offset `0`; saved candidate pointer; candidate word at offset `0`; `0xFFFFFFFF` | Nonzero returns the saved candidate; zero triggers failure handling |
| F-2022b0f73f3b | F-dd82aab5ab38 returned zero and the subsequent counter read is zero | Context pointer; fixed diagnostic-string address | No result is used; counter increment follows normal return |

The diagnostic reference suggests a message about a missing collation sequence and a string-format component. The actual string bytes are not supplied. Exact punctuation, formatting, substitution requirements, and rendered output are therefore not established.

Only two arguments to F-2022b0f73f3b are shown. No candidate name or other substitution argument should be invented.

## Acceptance examples

These examples are conditional expectations derived from the supplied static evidence. They are not runtime test results. Unless stated otherwise, all accessed storage is valid, dependencies return normally, and there is no concurrent mutation.

### Empty input

Given a null starting-record pointer:

- The result is null.
- No context storage is read or written.
- Neither dependency is called.

The context pointer need not be dereferenceable on this path.

### Traversal through empty special-tag records

Given an empty `0x1F` record followed by an empty `0x56` record, followed by a record with candidate pointer **P**:

- The function follows the first two next-record pointers.
- It selects **P**, regardless of the third record’s tag.
- It invokes F-dd82aab5ab38 exactly once with **P**.

If that dependency returns nonzero, the result is **P**.

### A populated special-tag record stops traversal

Given a `0x56` record with candidate **P** and a following record with candidate **Q**:

- **P** is selected.
- The current record’s next-pointer field is not needed.
- **Q** is not considered.

The special tag does not override the presence of a candidate.

### An empty ordinary-tag record is a barrier

Given a record with tag `0x20`, a null candidate, and a next record containing **P**:

- The result is null.
- The next-pointer field is not read.
- **P** is not considered.
- Neither dependency is called, and the context counter is not updated.

### Dependency success

Given a selected candidate **P**, and F-dd82aab5ab38 returning any nonzero value:

- The result is the saved pointer **P**.
- F-2022b0f73f3b is not called.
- This function performs no counter increment.

Any counter mutation performed by F-dd82aab5ab38 itself remains possible.

### Failure with a zero counter

Given F-dd82aab5ab38 returning zero and the subsequent counter read yielding zero:

- F-2022b0f73f3b is called.
- The counter increment follows its return.
- The result is null.

If the diagnostic dependency leaves the counter unchanged, the stored value becomes one. If it changes the counter to seven, the stored value becomes eight under 32-bit word arithmetic.

### Failure with a nonzero counter

Given F-dd82aab5ab38 returning zero and the subsequent counter read yielding five:

- F-2022b0f73f3b is skipped.
- The counter becomes six, absent intervening mutation.
- The result is null.

For an initial value of `0xFFFFFFFF`, the corresponding 32-bit update produces zero.

### No fallback after candidate failure

Given a selected candidate **P**, a reachable later candidate **Q**, and F-dd82aab5ab38 returning zero for **P**:

- **Q** is not examined as a fallback.
- Failure handling occurs for **P**.
- The result is null.

## Uncertainties

The following are **documented_only** unknowns:

- The complete types, sizes, alignment, and validity rules of the context, records, and candidates.
- The independently confirmed target pointer width, ABI, and endianness.
- The semantic meaning of the context’s first word and the candidate’s first word.
- The meaning of `0xFFFFFFFF` to F-dd82aab5ab38.
- The operations, side effects, lifetime guarantees, and failure mechanisms of both dependencies.
- The diagnostic’s exact bytes and formatting behavior.
- Whether the counter is intended as an error count, diagnostic-suppression state, or another measure.
- Any external termination guarantee for record chains.
- Any external synchronization or aliasing restrictions.

“Candidate processing,” “failure counter,” and “collation sequence” are descriptive interpretations. The established behavior is the selection rule, dependency calls, zero/nonzero branching, and four-byte context update described above.

## Evidence

This contract is based solely on the supplied static representation and opaque dependency mapping.

Established evidence includes the byte offsets, recognized tag values, null checks, traversal conditions, dependency argument positions, zero/nonzero result handling, diagnostic gate, counter-update placement, and return paths.

No dependency implementation, complete type definition, independently confirmed ABI, exact diagnostic bytes, or runtime observations were supplied. No runtime tests have been performed.
