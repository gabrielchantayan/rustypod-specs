<!-- Trusted provenance: F-6300bfdf2cc8; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6300bfdf2cc8

## Purpose

F-6300bfdf2cc8 coordinates initialization, repeated candidate retrieval, candidate evaluation, and finalization over a small mutable state area.

Its observable result combines:

- A low 32-bit outcome: either a count associated with a qualifying candidate or the all-ones failure sentinel.
- A high 32-bit value: the first state word as it exists **after finalization**.

The visible behavior resembles a filtered search or filtered-position calculation. That interpretation is inferred intent, not an established domain contract. The supplied evidence does not establish what candidates represent, what the state words encode, or whether any dependency manages resources.

The important distinction is that the counter does not count every retrieved candidate. It advances only when evaluation returns nonzero and the second state word does not equal the original second argument. A qualifying candidate at that equality condition terminates the search without incrementing the counter.

## Interface

### Inputs

The function accepts four arguments:

| Argument | Visible representation | Role established by the supplied evidence |
|---|---|---|
| First | Address-bearing 32-bit integer | Used to read one 32-bit word at byte offset 4. That word is passed to initialization. |
| Second | 32-bit integer | Initial first state word and the fixed comparison reference used during the search. |
| Third | 32-bit integer | Initial second state word. |
| Fourth | Opaque 32-bit value | Initial third state word; this word also receives candidate data and supplies the evaluator’s argument. |

The first argument is not itself forwarded to initialization. The function reads the word at its address plus four bytes and forwards the loaded value.

The visible operation therefore requires that this address calculation identify a readable four-byte region. There is no visible null check, bounds check, alignment check, or address-wrap check. Target-specific alignment and address-arithmetic requirements remain unknown.

The fourth argument is an untyped word for purposes of this contract. It must not be assumed to be a pointer, handle, owned allocation, or scalar of a particular domain.

### Local state layout

The supplied representation places three addressable 32-bit state words consecutively:

| Offset from the state base | Initial contents | Visible use |
|---|---|---|
| 0 bytes | Second argument | Passed by address to initialization, retrieval, and finalization; packed into the result after finalization. |
| 4 bytes | Third argument | Compared with the original second argument after a qualifying evaluation. |
| 8 bytes | Fourth argument | Passed separately by address to retrieval and subsequently read as the evaluator’s argument. |

These are the three visible words, not a proven complete layout for any external type.

Retrieval receives both the base address and an address into the same state area, at offset 8. These arguments alias the same local storage. A behavioral substitute must preserve that relationship rather than provide an unrelated output cell.

### Output

The visible return representation is 64 bits:

- Bits 63–32 contain the first state word read after finalization.
- Bits 31–0 contain the selected outcome word.

This is bitwise concatenation, not sign extension of a 32-bit result. A low word of all ones does not by itself make the complete 64-bit result all ones.

The semantic type of the aggregate is unknown. The evidence does not establish whether callers treat it as an integer, a packed pair, or a calling-convention-specific aggregate.

## Behavior

### Initialization and first retrieval

Before calling a dependency, the function:

1. Initializes the qualifying-candidate counter to zero.
2. Establishes the three state words from the second, third, and fourth arguments.
3. Reads the 32-bit word at byte offset 4 from the address represented by the first argument.
4. Calls F-8ca21d3a8cca with the state base address and that loaded word.

Any return value from initialization is unused.

The first retrieval follows initialization unconditionally on the visible normal-return path. There is no preliminary comparison between the second state word and the original second argument. Equality at entry therefore does not itself produce success.

The three initial state values are established before initialization. Their values after initialization are not constrained by this function: initialization receives the state address and may change the state within whatever contract actually governs that dependency.

### Retrieval and evaluation

Each retrieval calls F-b38a3aba0e18 with:

- The state base address.
- The address of the third state word.

Its return value is tested only for zero versus nonzero.

If retrieval returns zero:

- No evaluation occurs for that retrieval.
- The selected low result becomes the all-ones 32-bit sentinel.
- The function proceeds to finalization.

If retrieval returns nonzero:

- The function reads the third state word.
- It passes that 32-bit value to F-56e5aaaec5ab.
- The evaluator’s return is also tested only for zero versus nonzero.

A nonzero retrieval result is not itself forwarded to evaluation. Evaluation receives the contents of the third state word after retrieval.

If the evaluator returns zero:

- The counter does not advance.
- The equality condition is not tested on that path.
- Another retrieval is attempted.

If the evaluator returns nonzero, the function compares the current second state word with the original second argument.

- If they are equal, the selected low result is the current counter, and the search ends.
- If they differ, the counter advances by one, and another retrieval is attempted.

The evaluator’s nonzero return value is not used as a count, score, or result. Negative nonzero values take the same branch as positive nonzero values.

### Meaning of the counter

On a successful termination, the low result counts earlier iterations that met both conditions:

- Retrieval returned nonzero and evaluation returned nonzero.
- The second state word differed from the original second argument when checked.

The terminating qualifying candidate is excluded from that count. Candidates whose evaluation returns zero are also excluded.

Consequently:

- The first qualifying candidate can yield zero.
- Rejected candidates before success do not increase the returned count.
- Earlier qualifying candidates do not cause a partial count to be returned if retrieval eventually returns zero. That termination selects the sentinel instead.

The comparison reference remains the original second argument. It is not replaced by an updated first state word, even though the first state word initially has the same value and may subsequently be mutated.

### Ordering around evaluation

The second state word is inspected only after the evaluator returns nonzero.

The visible function does not take an earlier snapshot of that word for the comparison. The contract therefore must not assume that the comparison necessarily sees its value immediately after retrieval. Evaluation receives no explicit state pointer, but the supplied evidence does not establish that it is free of indirect state access or other side effects.

### Finalization and result assembly

After either visible loop exit, the function calls F-d0d3f100effc with the state base address.

On normal return from finalization:

- The selected low result remains the outcome chosen before finalization.
- The first state word is read for the high half of the return value.
- The two 32-bit words are packed into the 64-bit result.

Finalization is reached after successful termination and after retrieval returns zero. There is no separate finalization path for evaluator rejection; rejection continues the loop.

There is no visible iteration limit, cancellation condition, or timeout. If retrieval repeatedly returns nonzero without a qualifying equality condition, the function need not terminate.

## Side effects

### Directly established effects

The function visibly:

- Reads one 32-bit word through the first argument at offset 4.
- Writes the initial three-word local state.
- Supplies mutable local storage to three dependencies.
- Calls the evaluator once for each retrieval that returns nonzero.
- Calls finalization once after a normal loop exit.
- Returns a value incorporating state observed after finalization.

The function does not visibly write through the first argument. That does not establish that the memory reachable from it remains unchanged: dependency behavior is unknown.

### Ownership and lifetime

The state area is local to this invocation. Its addresses are provided to dependencies during the invocation. The visible function does not allocate a separate heap state object or transfer ownership of its local storage.

Nothing establishes whether dependencies retain these addresses, allocate resources, release resources, or treat values within the state as owned references. In particular, the finalization call is not proof that memory is freed.

The fourth argument is copied into local storage. No ownership transfer or reference-count operation on that value is directly visible.

### Failure and abnormal completion

The all-ones low word is selected specifically when retrieval returns zero. It is not an explicit error report from initialization, evaluation, or finalization.

There is no visible error recovery for:

- An invalid initial memory read.
- A dependency that fails to return normally.
- An invalid dependency access to the supplied state.
- Counter overflow.

Finalization is established only on paths that reach it normally. Unconditional cleanup after faults, exceptions, or other nonlocal exits is not established.

## Dependencies

| Opaque ID | Visible interaction | Established use of return value |
|---|---|---|
| F-8ca21d3a8cca | Receives the mutable state base and the 32-bit value loaded through the first argument. Called once before retrieval. | Ignored. |
| F-b38a3aba0e18 | Receives the mutable state base and the address of its third word. Called repeatedly. | Zero ends the search with the sentinel; every nonzero value permits evaluation. |
| F-56e5aaaec5ab | Receives the third state word by value after a nonzero retrieval result. | Zero rejects the candidate; every nonzero value enables the equality check. |
| F-d0d3f100effc | Receives the mutable state base after a normal loop exit. | Ignored; the first state word is read afterward. |

These descriptions establish invocation order and argument relationships, not the dependencies’ complete contracts.

No dependency is assumed to preserve globals, preserve unrelated memory, own candidate values, release resources, or avoid callbacks. Such properties require separate evidence.

## Acceptance examples

The following are conditional acceptance criteria, not executed tests. Each assumes valid memory access, normally returning dependencies, and no counter overflow.

### Immediate retrieval termination

Suppose initialization returns normally and the first retrieval returns zero.

Expected behavior:

- Evaluation is not called.
- Finalization is called once.
- The low result is `0xFFFFFFFF`.
- The high result is the first state word after finalization.

The initial third word, or any word written there by the terminating retrieval, is not evaluated.

### First candidate qualifies at the reference condition

Suppose the first retrieval returns nonzero, evaluation returns nonzero, and the second state word equals the original second argument when checked.

Expected behavior:

- The search ends after one retrieval and one evaluation.
- The low result is zero.
- Finalization occurs before the high result is read.

### Rejected candidates do not advance the count

Suppose two retrievals return nonzero and their evaluations return zero. A third retrieval returns nonzero, its evaluation returns nonzero, and the comparison then finds equality.

Expected behavior:

- Three evaluations occur.
- The low result is zero.
- Equality during either rejected iteration, if present, does not terminate the search.

### Only qualifying nonterminal candidates advance the count

Suppose three retrieved candidates evaluate as nonzero, zero, and nonzero. The first qualifying candidate has unequal state and reference values. The third candidate has equal values when checked.

Expected behavior:

- The low result is one.
- The rejected middle candidate contributes nothing.
- The successful final candidate contributes nothing.

### Retrieval termination discards an accumulated count

Suppose four candidates qualify but each has unequal state and reference values. The next retrieval returns zero.

Expected behavior:

- The low result is `0xFFFFFFFF`, not four.
- No evaluator call occurs for the zero-returning retrieval.
- Finalization still occurs.

### State mutation does not move the reference

Suppose the original second argument is 7. A dependency changes the first state word to 12. A qualifying evaluation is followed by a second state word of 12.

Expected behavior:

- Equality is checked against 7, not 12.
- This iteration advances the counter rather than ending the search.

If a later qualifying evaluation is followed by a second state word of 7, that later iteration can terminate successfully.

### Finalization determines the high word

Suppose a successful search selects low result 2, and finalization leaves the first state word as `0x12345678`.

Expected return representation: `0x1234567800000002`.

If retrieval instead ended the search with the sentinel, the same final first word would produce `0x12345678FFFFFFFF`.

### Initially equal state is insufficient

Suppose the initial second state word equals the original second argument, but retrieval returns zero immediately.

Expected behavior: sentinel termination, not success with count zero.

Equality matters only after a nonzero retrieval and a nonzero evaluation.

## Uncertainties

The following are **documented_only**:

- **Domain meaning.** The roles of the three words and the candidates are not established.
- **Dependency memory contract.** The visible three-word layout does not establish every access dependencies may make, or whether they require a larger enclosing object.
- **Ownership.** Allocation, retention, destruction, and resource-release responsibilities remain unknown.
- **Return interpretation.** The 64-bit packing is established; its public semantic type is not.
- **Retrieval zero.** It may represent exhaustion, failure, or another terminal condition. Only its control-flow consequence is established.
- **Counter overflow.** The supplied representation uses a signed 32-bit counter and increments it without a visible guard. Behavior beyond the signed maximum is not established. Wraparound, trapping, and defined sentinel collision must not be asserted from this evidence.
- **Address arithmetic and byte order.** A four-byte-offset, 32-bit load is visible. Target rules for invalid addresses, overflow, alignment, and byte interpretation are not fully established.
- **Indirect mutation.** The absence of an explicit state pointer in the evaluator call does not prove that the evaluator cannot affect state through another route.
- **Abnormal-exit guarantees.** Finalization after faults or nonlocal control transfer is not established.

## Evidence

This contract derives from the supplied static function representation and the supplied opaque dependency mapping.

Established static evidence includes the three initial state writes, the offset-four input read, dependency call ordering, zero/nonzero branching, conditional counter advancement, comparison against the original second argument, finalization placement, and post-finalization packing of the first state word.

The filtered-search interpretation is inferred intent only. No runtime tests have been performed, and the acceptance examples have not been executed.
