<!-- Trusted provenance: F-bfe1e48bd9b5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bfe1e48bd9b5

## Purpose

F-bfe1e48bd9b5 prepares a bounded byte sequence, dispatches on its first prepared byte, and conditionally updates a caller-supplied record. Visible operations include numeric adjustments, component changes, conversion-like processing, and boundary-like assignments.

The return value is 32 bits. Every visible return supplies **0** or **1**, but neither value establishes transactional success:

- Return 1 can follow record writes and dependency calls.
- Return 0 does not consistently include final record postprocessing.
- No rollback or dependency-failure recovery is visible.
- The weekday-labeled path has no reachable successful continuation under the supplied control flow.

Calendar, comparison, and conversion terminology expresses inferred intent. It does not establish dependency semantics, accepted text, or numeric representation.

## Interface

### Inputs and storage

**Input address:** A 32-bit address-like value identifying readable bytes. Preparation examines successive original bytes until an observed zero or a limit of 29 nonzero observations. Required readability extends through the zero actually examined, or through all 29 examined positions if no zero is encountered.

Each original-byte check precedes its corresponding preparation call. Dependencies are not established to preserve unread input bytes, so initial contents alone do not determine the number of iterations.

**Record address:** Caller-supplied mutable storage. Direct address arithmetic is 32-bit. The greatest explicitly accessed byte offset is 43; paths using that offset require at least 44 accessible bytes. Dependencies may require additional storage, alignment, or initialization.

| Byte offset | Width | Explicit use |
|---|---:|---|
| 0 | 8 bytes | Read or replace a numeric pair |
| 8 | 4 bytes | Component read, replacement, or carry accumulation |
| 12 | 4 bytes | Component read, replacement, normalization, or assignment of 1 |
| 16 | 4 bytes | Assignment of 1 |
| 20 | 4 bytes | Assignment of zero |
| 24 | 4 bytes | Assignment of zero |
| 32 | 4 bytes | Assignment of an unavailable constant |
| 36 | 4 bytes | Assignment of another unavailable constant |
| 41 | 1 byte | Assignment of 1 |
| 42 | 1 byte | Conditional test or assignment of zero |
| 43 | 1 byte | Assignment of zero |

This is a partial access layout, not a complete record definition.

A **pair** consists of two 32-bit words. For record storage, the first word is at offset 0 and the second at offset 4. When a returned 64-bit value is split into arguments, the order is its low 32 bits followed by its high 32 bits. This does not establish floating-point representation, byte endianness, or atomic access.

The numeric parser receives separate local storage for two output words. References below to **parser-output words** mean the contents read for the particular call, not an immutable copy of the parser’s original output. Intervening dependencies are not established to preserve that storage.

### Outputs and preconditions

The function returns a status and updates storage in place. It does not return a replacement record.

Local storage includes a 32-byte preparation buffer, parser-output storage, and temporary storage used by the colon path. No explicit allocation, release, or retention operation appears in the visible body. Dependency ownership behavior remains unknown.

There are no visible null checks, comprehensive bounds checks, or rollback. The behavioral descriptions assume dependencies return normally and all required memory accesses are valid. Invalid addresses, malformed dependency results, and unmet dependency preconditions have no established safe outcome.

## Behavior

### Preparation and dispatch

Preparation checks one original byte at a time. Each nonzero observation causes a call to F-1aef7372e8d8, whose returned byte is stored at the corresponding preparation-buffer position. No explicit argument to that dependency is exposed.

Preparation stops after an observed original zero or 29 calls, then stores a zero terminator. A zero returned by F-1aef7372e8d8 does not itself stop preparation. Identity conversion, lowercasing, and dependence on the inspected original byte are unverified.

The first prepared byte selects the path:

| Prepared first byte | Selection |
|---|---|
| Plus, minus, or ASCII decimal digit | Numeric processing |
| Comma, period, or slash | Return 1 |
| Lowercase “l” | Local-conversion-labeled path |
| Lowercase “u” | Epoch/conversion-labeled paths |
| Lowercase “s” | Boundary-labeled path |
| Lowercase “w” | Weekday-labeled path |
| Anything else, including zero | Return 1 |

There is no visible leading-byte skipping before dispatch.

### Opaque target checks

Two dependencies lack mapped IDs and are designated here by contract-local opaque IDs:

- **F-unmapped-01:** Two pointer arguments; its result is tested for zero or nonzero.
- **F-unmapped-02:** Temporary-storage address and numeric argument 48.

F-unmapped-01 receives prepared text or a derived address and a target pointer. F-fca51b715561 receives two pointers and an additional argument of 8 or 9.

Neither dependency is established to perform equality, prefix matching, bounded reading, or side-effect-free comparison. Target descriptions distinguish call sites and inferred intent. A zero-result target call does not prove a particular accepted spelling.

### Numeric entry and colon selection

F-d920a917d399 receives the preparation-buffer address and parser-output storage. Its signed 32-bit return is immediately used as a byte displacement into the buffer, without a visible bounds check.

If the byte at that displacement is a colon, F-2a4c402ba617 supplies access to a 32-bit table-base object. A byte entry indexed by the unsigned first prepared byte is compared with value 32. An unequal entry selects buffer address plus one; an equal entry selects buffer start. This is not an explicit sign test.

F-unmapped-02 then receives temporary storage and 48. F-4c6e5de9d7fd receives the selected input address and that storage. Its nonzero result returns 1 before this path passes the main record to F-dff0a098ef8e.

On a zero result, temporary processing establishes an adjustment:

- F-dff0a098ef8e processes the temporary storage.
- F-8a693c103d1c receives its first pair followed by an unavailable constant pair; its result replaces the temporary pair.
- F-baa332e06e32 and F-b3a54d881220 run, with argument flow not fully exposed.
- F-387e686c467e receives the F-b3a54d881220 result first and the temporary pair second.
- If the first prepared byte is minus, bit 31 of the adjustment’s high word is toggled using `0x80000000`.

The bit operation does not establish arithmetic negation.

The main record passes to F-dff0a098ef8e and then F-681e65750b84. Its pair is read only afterward. F-1b22d7e17791 receives that freshly read record pair first and the adjustment second. Its result replaces the record pair, and the function returns 0 without another postprocessing call.

### Non-colon admission and unit selection

For a non-colon byte at the parser-returned displacement, the candidate address begins at that displacement. F-2a4c402ba617 supplies the table-base object. The candidate advances while bit 0 of the table entry indexed by its current unsigned byte is set. The displayed condition obtains the table base through that object on each evaluation. No independent buffer bound is visible.

This classification is not proven to mean whitespace.

F-1cb585febd84 receives the resulting candidate address. Its 32-bit result is used as a length and index, but is not established to be a string length.

The unsigned 32-bit subtraction-and-comparison admits exactly **3 through 10 inclusive**. Other results return 1. For an admitted result, the byte at candidate address plus result minus one is inspected. If it is lowercase “s”, it is replaced with zero and the result decreases by one.

F-dff0a098ef8e then processes the main record. Unit selection uses the reduced value:

| Reduced value | Ordered F-unmapped-01 calls |
|---:|---|
| 3 | Three-byte-labeled target; zero selects direct combination |
| 4 | Scaling-labeled target first; zero selects scaling. Only nonzero permits the calendar-labeled target call; zero there selects a component update |
| 5 | Month-labeled target; zero selects month processing |
| 6 | Minute-labeled target first; zero selects scaling. Only nonzero permits the second-labeled target call; zero there also selects scaling |
| Other | No unit-target call |

Nonselecting outcomes reject. These calls may have side effects; evaluating a short-circuited call or changing their order is not equivalent on the available evidence.

Every normally completing path after this record-preparation call finishes through F-681e65750b84. Selected operations return 0; rejected cases return 1 after that call.

For **direct combination**, the record pair is read after the target call. F-1b22d7e17791 receives the record pair first and the parser-output words read for that call second. Its result replaces the record pair.

For **scaling**, F-596e18a33aec receives the parser-output words read for its call; further arguments are not exposed. After it returns, the record pair is read into the same local storage previously used for parser output. F-1b22d7e17791 receives the scaling result first and those newly read record words second. Its result replaces the record pair. Distinct minute-like and second-like factors are not established.

### Calendar-labeled component update

For the four-valued calendar selection, F-b37fcadfad8d and F-edfc768caf62 process the record in that order.

The 32-bit field at offset 8 is subsequently passed to F-b3a54d881220. F-1b22d7e17791 receives that returned pair first and the parser-output words read for its call second.

F-baa332e06e32 next produces a 32-bit result; its arguments at this call are not explicitly exposed. The result is stored at offset 8, byte 42 is cleared, and F-dff0a098ef8e processes the record. Common final postprocessing follows.

### Month-labeled processing

F-b37fcadfad8d and F-edfc768caf62 run in order. The field at offset 12 is passed to F-b3a54d881220. F-1b22d7e17791 receives that returned pair first and the parser-output words read for this call second.

The next F-baa332e06e32 call has no explicitly exposed arguments. Its signed 32-bit result becomes the component stored at offset 12. Whether that component equals zero is saved for a later branch.

Normalization uses this saved component:

- Below 1, the input to F-870c4fcf7295 is component minus 12.
- Otherwise, that input is component minus 1.
- The dependency’s second argument is 12.
- Offset 12 is replaced with component minus twelve times the returned quotient-like result.
- That result is added to the then-current 32-bit field at offset 8.

These are 32-bit expressions. Overflow behavior and quotient rounding are unresolved, so a mathematically normalized component range is not guaranteed.

Byte 42 is cleared, then F-dff0a098ef8e processes the record. The following numeric calls have explicitly exposed operand flow:

1. **F-baa332e06e32 receives the first parser-output word and then the second parser-output word, as read for this call.**
2. F-b3a54d881220 returns a pair; its arguments at this call are not explicitly exposed.
3. **F-a2d1320ee317 receives that returned pair’s low word, its high word, then the first and second parser-output words read for this later call.**

The parser-output arguments to the first and third calls are separate reads. Their equality is not guaranteed across intervening dependencies.

The branch after F-a2d1320ee317 uses the saved component-zero condition, not an established result of F-a2d1320ee317. A saved zero component skips correction but does not skip these three calls.

For a saved nonzero component, F-387e686c467e receives the retained F-b3a54d881220 pair first and the parser-output words read for that call second. F-c4b34006a13c receives its returned pair and an unavailable constant pair.

Only afterward is the record pair read, overwriting the parser-output storage. F-1b22d7e17791 receives the correction first and the newly read record pair second. Its result replaces the record pair. Both correction outcomes finish through F-681e65750b84 and return 0.

### Local-conversion-labeled path

F-unmapped-01 receives the buffer and a local-time-labeled target. Nonzero returns 1.

Zero selects F-dff0a098ef8e followed by F-05b741fcd138 on the record. The record pair is read after the latter call. F-1b22d7e17791 receives the dependency result first and the fresh record pair second. Its result is stored, F-681e65750b84 runs, and the function returns 0.

### Epoch/conversion-labeled paths

F-unmapped-01 first receives the buffer and an epoch-labeled target. Byte 42 is tested only after a zero result.

If that result is zero and byte 42 is nonzero, F-596e18a33aec receives the record pair first and an unavailable constant pair second. F-1b22d7e17791 then receives its result first and another unavailable constant pair second. The result is stored, F-681e65750b84 runs, and the function returns 0.

Otherwise, a second target call occurs. Nonzero returns 1. Zero selects a two-stage conversion:

- F-dff0a098ef8e processes the record.
- F-05b741fcd138 produces a retained first pair.
- F-8a693c103d1c receives a record pair read after that call, followed by the retained pair. Its result is stored.
- F-681e65750b84 processes the record.
- F-05b741fcd138 produces a second pair.
- F-387e686c467e receives the second pair first and the retained first pair second.
- After that call, the record pair is read again.
- F-1b22d7e17791 receives the F-387e686c467e result first and the fresh record pair second.

The final result is stored and return 0 follows without another postprocessing call.

### Boundary-labeled path

F-fca51b715561 receives the buffer, a boundary-labeled target, and 9. Nonzero returns 1. Zero selects buffer plus 9 as the candidate and calls F-b37fcadfad8d on the record.

The common direct accesses occur in this order:

1. Store byte 1 at offset 41.
2. Store 32-bit zero at offset 24, then at offset 20.
3. Read the unavailable constant destined for offset 36.
4. Read another unavailable constant and store it at offset 32.
5. Store the saved constant at offset 36.
6. Clear byte 43, then byte 42.

Three F-unmapped-01 target calls are short-circuited in order:

- A zero month-labeled result stores 32-bit 1 at offset 16 and returns 0.
- Only after nonzero is the second target tried. Zero calls F-b37fcadfad8d again, stores 32-bit 1 at offset 12 and then offset 16, and returns 0.
- Only after another nonzero is the third target tried. Zero returns 0 without additional explicit component stores; nonzero returns 1.

None of these outcomes calls F-681e65750b84.

### Weekday-labeled path

F-fca51b715561 receives the buffer, a weekday-labeled target, and 8. Nonzero returns 1.

On zero, F-d920a917d399 receives buffer plus 8 and parser-output storage. Its signed return below 1 returns 1. A condition recording whether that return was zero is saved independently of later numeric results.

For a return of at least 1, the exposed operand flow is:

1. **F-baa332e06e32 receives the first parser-output word and then the second, as read for this call.**
2. F-b3a54d881220 returns a pair; its arguments are not explicitly exposed.
3. **F-a2d1320ee317 receives that pair’s low word, its high word, then the first and second parser-output words read for this later call.**

The two sets of parser-output reads need not have identical values.

The next rejection tests the saved parser-return-zero condition. Reaching it required a parser return of at least 1, so the condition is false and return 1 follows.

Later range checks, F-11ed8242c40c, and record-update operations are unreachable under the supplied semantics. They do not establish successful weekday behavior.

## Side effects

Rejection does not imply an unchanged record or absence of dependency effects. No rollback is visible.

Direct stores affect only their stated widths. Record reads after dependencies observe then-current storage, not an entry snapshot. Parser-output reads likewise occur for individual calls; they are not guaranteed to preserve the parser’s original words.

F-681e65750b84 runs after final stores on some paths, before final stores on others, and not on boundary-labeled paths. Its role as normalization, validation, or invalidation is unverified.

Dependencies may affect globals, input, record storage, local storage whose address has escaped, allocation state, or ownership. The visible body does not establish preservation or prohibit pointer retention.

## Dependencies

Roles describe exposed use, not verified implementations.

| Opaque ID | Exposed role |
|---|---|
| F-1aef7372e8d8 | Produces prepared bytes |
| F-d920a917d399 | Receives parser-output storage and returns a displacement/count-like value |
| F-2a4c402ba617 | Supplies access to a classification-table base object |
| F-1cb585febd84 | Produces a 32-bit admission/index value |
| F-unmapped-01 | Two-pointer target check |
| F-fca51b715561 | Target check with additional argument 8 or 9 |
| F-unmapped-02 | Temporary-storage operation with argument 48 |
| F-4c6e5de9d7fd | Temporary parse-like operation |
| F-dff0a098ef8e | Record or temporary-storage processing |
| F-b37fcadfad8d, F-edfc768caf62 | Component-related preparation |
| F-681e65750b84 | Selected record postprocessing |
| F-05b741fcd138 | Record-dependent pair production |
| F-baa332e06e32 | 32-bit conversion-like result; explicit parser-word arguments at the identified month and weekday calls |
| F-b3a54d881220 | Pair conversion-like result |
| F-a2d1320ee317 | Receives converted low/high words followed by separately read parser-output words on the month and weekday paths |
| F-596e18a33aec, F-1b22d7e17791, F-8a693c103d1c, F-387e686c467e, F-c4b34006a13c | Numeric operations with operand order specified above |
| F-870c4fcf7295 | Quotient-like operation |
| F-11ed8242c40c | Call only in unreachable continuation |

## Acceptance examples

These are conditional static criteria, not executed tests.

1. **Preparation bound:** Exactly 29 preparation calls occur if each successive original-byte check observes nonzero. No thirtieth original-byte check is required.

2. **Prepared zero:** A zero preparation result is stored but does not itself stop preparation.

3. **Admission boundary:** An admission result of 2 rejects before numeric record preparation; 3 passes the gate. These are string-length examples only if F-1cb585febd84 actually computes length.

4. **Trailing removal:** An admitted value of 3 with lowercase “s” at candidate displacement 2 becomes reduced value 2. Record preparation and postprocessing occur, but no unit-target call occurs.

5. **Short circuit:** A zero scaling-target result for reduced value 4 suppresses the calendar-target call. A zero minute-labeled result for reduced value 6 suppresses the second-labeled call.

6. **Month operands:** Suppose the parser-output reads for the post-normalization F-baa332e06e32 call yield words A and B. That call receives A then B. If F-b3a54d881220 returns low/high words L and H, and the later parser-output reads yield C and D, F-a2d1320ee317 receives L, H, C, D—not necessarily L, H, A, B.

7. **Saved month condition:** A saved zero component skips correction only after the three post-normalization numeric calls. A numeric dependency result does not replace that saved condition.

8. **Weekday operands and rejection:** With a zero initial target result and parser return at least 1, the same explicit argument ordering applies: parser words to F-baa332e06e32, then converted low/high words followed by newly read parser words to F-a2d1320ee317. Return 1 follows the saved-condition check.

9. **Fresh record operand:** On the second conversion-labeled path, if F-387e686c467e returns pair A and the subsequent record read yields B, F-1b22d7e17791 receives A first and B second.

10. **Boundary rejection after writes:** A zero initial boundary target result followed by three nonzero target results returns 1 after the common record stores.

11. **Epoch gate:** A zero first epoch target result with byte 42 zero causes the second target call, not the epoch conversion operation.

## Uncertainties

The following remain **documented_only**:

- Comparison semantics, target contents, accepted spellings, and read extents.
- Whether F-1cb585febd84 computes string length.
- Numeric representation, unavailable constants, and unexposed arguments or implicit result flow.
- Parser initialization, output guarantees, displacement bounds, and consumption rules.
- Classification-table contents and traversal termination.
- Temporary-storage extent and the meaning of argument 48.
- Signed overflow, address overflow, quotient rounding, and exceptional arithmetic.
- Dependency effects on memory, globals, allocation, ownership, and retained pointers.
- Fidelity of displayed saved conditions to underlying execution.
- Safety with overlapping storage or malformed dependency outputs.

The explicit month and weekday arguments described above are not unresolved implicit argument flow. What remains unknown is the values read at each call and what the dependencies do with them.

## Evidence

Established static evidence comprises visible byte tests, argument placement, word extraction, access widths and offsets, ordered calls, short-circuit branches, stores, arithmetic expressions, and returns.

In particular, both the post-normalization month sequence and the reachable weekday sequence explicitly pass two parser-output words to F-baa332e06e32. Both explicitly pass the low and high words returned by F-b3a54d881220, followed by separately read parser-output words, to F-a2d1320ee317.

Calendar terminology and conversion-like or comparison-like roles are inferred intent, not established dependency behavior.

No runtime tests, dependency inspections, or implementation executions have been performed.
