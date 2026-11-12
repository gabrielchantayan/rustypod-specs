<!-- Trusted provenance: F-2d9b6ef1d9a1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-2d9b6ef1d9a1

## Purpose

F-2d9b6ef1d9a1 selects one of nine observable construction behaviors from a one-byte selector: explicit cases 0 through 7, plus a shared fallback for every other byte value.

The function obtains opaque 32-bit values, passes them to dependencies in selector-specific sequences, and publishes results through five caller-supplied output locations. Cases 5 through 7 also invoke a range-parameterized dependency and conditionally publish an intermediate value through the fifth output.

**Established:** the supplied static representation determines argument forwarding, direct memory accesses, dependency-call order, and output-store order.

**Inferred intent:** the call patterns resemble construction of operations over channel-like values, with some cases using a packed intermediate. Neither channel meaning nor operation semantics is established. Numeric operation codes and range arguments remain literal interface values in this contract.

## Interface

The function has eleven positional arguments and no explicit return value.

| Position | Contract name | Role |
|---|---|---|
| 1 | Context | Address-like 32-bit input; forwarded to F-e4de34edb462 and dereferenced only on cases 5–7. |
| 2 | Provider | Opaque 32-bit input passed to F-b35e541ebc78. |
| 3 | Sink | Opaque 32-bit input passed to the other four dependencies. |
| 4 | Descriptor | Address-like 32-bit input whose byte at offset `0x21` selects the behavior. |
| 5 | Input X | Opaque 32-bit value used in dependency arguments. |
| 6 | Input Y | Opaque 32-bit value used in dependency arguments. |
| 7 | Output A | Pointer to a writable 32-bit location. |
| 8 | Output B | Pointer to a writable 32-bit location. |
| 9 | Output C | Pointer to a writable 32-bit location. |
| 10 | Output D | Pointer to a writable 32-bit location. |
| 11 | Output E | Pointer to a writable 32-bit location; initially cleared. |

### Widths and layouts

- Output stores and output reloads are 32 bits.
- The selector is read as one byte at Descriptor offset `0x21`.
- Cases 5–7 read a 32-bit address-like field at Context offset `0x08`, then read one byte at offset `0xB2` from that referenced object.
- All obtained values and all results from F-e4de34edb462 are treated as opaque 32-bit quantities.
- Constants such as `0xFF` are supplied as values, not written as individual bytes by this function.
- The containing object sizes, alignment requirements, byte order, and portable pointer representation are not established beyond these observed accesses.

The caller must provide accessible storage for every direct access reached by the selected path. Output E must be writable before the selector is read. Cases 5–7 require the additional Context-based reads to remain valid after their dependency calls.

No null checks, bounds checks, allocation-failure checks, or explicit error return are present.

## Behavior

### Reading the behavioral notation

The following notation describes observable dependency interactions, not arithmetic implemented by this function:

| Event | Dependency and arguments |
|---|---|
| Obtain a token | F-b35e541ebc78(Provider, 0), returning one 32-bit value |
| Literal event | F-fbf0cbff8cb3(Sink, `0x28`, destination, literal) |
| Transfer event | F-cb523cc6fa7d(Sink, code, destination, source) |
| Binary event | F-915779408ccb(Sink, code, destination, first operand, second operand) |
| Range event | F-e4de34edb462(Context, Sink, source, lower argument, upper argument), returning one 32-bit value |

“Literal,” “transfer,” “binary,” and “range” name argument shapes only. They do not establish what a dependency does.

Tokens T1, T2, and so on denote results of successive obtain calls **within the selected case**. They need not be distinct, nonzero, pointers, or newly allocated objects.

In the event descriptions below, an output name used as an argument means a fresh 32-bit read of that output location at that point. It does not mean the token originally stored there. This distinction matters when outputs alias or dependencies modify reachable memory.

### Common entry behavior

The first direct action is a 32-bit zero store to Output E. The function then reads the selector byte and chooses one case.

Consequences:

- If Output E overlaps the selector byte, the selector is read after the zero store.
- Selection occurs once. Later changes to the descriptor do not cause redispatch.
- The initial zero store is not repeated after dependency calls.

### Token acquisition and publication

The following table specifies the order of obtain calls and direct output stores before the event sequences described afterward. A publication immediately follows the obtain call listed in that row, before the next obtain call.

| Selector | Obtain calls and immediate publications |
|---|---|
| 0 | T1 → C, then B, then A; T2 → D; T3 retained internally. |
| 1 | T1 → B, then C, then A; T2 → D; T3 retained internally. |
| 2 | T1 → B, then C, then A; T2 → D; T3 through T6 retained internally. |
| 3 | T1 → D; T2 → A; T3 → B; T4 → C; T5 through T12 retained internally. |
| 4 | T1 → D; T2 → A; T3 → B; T4 → C; T5 through T13 retained internally. |
| 5 | T1 retained internally; T2 → D; T3 through T5 retained internally. |
| 6 or 7 | T1 through T4 retained internally; no A–D publications during acquisition. |
| Other | T1 → D, then B, then C, then A. |

All listed obtain calls complete before the subsequent event sequence begins.

### Selector 0

After three obtain calls:

1. Binary event: code 1, destination T3, operands Input Y and Input X.
2. Transfer event: code `0x26`, destination read from D, source T3.
3. Read A and issue a literal event targeting that captured value with literal 0.

The final literal event does not reload A after entering the dependency.

### Selector 1

After three obtain calls:

1. Binary event: code 1, destination T3, operands Input Y and Input X.
2. Transfer event: code `0x26`, destination read from A, source T3.
3. Read D and issue a literal event targeting that captured value with literal `0xFF`.

Selectors 0 and 1 therefore differ both in which output supplies the transfer destination and which output supplies the final literal destination.

### Selector 2

After six obtain calls, the ordered events are:

| Order | Event |
|---|---|
| 1 | Literal to T6, value 1. |
| 2 | Binary code 6 to T3, operands Input Y and T6. |
| 3 | Binary code 1 to T4, operands T3 and Input X. |
| 4 | Transfer code `0x26` to A, source T4. |
| 5 | Binary code 1 to T5, operands T4 and T6. |
| 6 | Transfer code `0x26` to D, source T5. |

A is read for event 4; D is read only after event 5 completes.

### Selector 3

After twelve obtain calls:

| Order | Event |
|---|---|
| 1 | Literal to D, value `0xFF`. |
| 2 | Literal to T5, value 1. |
| 3 | Literal to T6, value 2. |
| 4 | Literal to T7, value 3. |
| 5 | Binary code 6 to T8, operands Input Y and T5. |
| 6 | Binary code 1 to T9, operands Input Y and T8. |
| 7 | Binary code 1 to T10, operands Input X and T9. |
| 8 | Transfer code `0x26` to A, source T10. |
| 9 | Binary code 1 to T11, operands T10 and T5. |
| 10 | Transfer code `0x26` to B, source T11. |
| 11 | Binary code 1 to T12, operands T10 and T6. |
| 12 | Transfer code `0x26` to C, source T12. |

T7 participates in a literal event but is not subsequently passed to another dependency by this function. That event remains part of the observable behavior.

### Selector 4

After thirteen obtain calls:

| Order | Event |
|---|---|
| 1 | Literal to T5, value 1. |
| 2 | Literal to T6, value 2. |
| 3 | Literal to T7, value 3. |
| 4 | Literal to T8, value 7. |
| 5 | Binary code 6 to T9, operands Input Y and T6. |
| 6 | Binary code 1 to T10, operands Input X and T9. |
| 7 | Transfer code `0x26` to A, source T10. |
| 8 | Binary code 1 to T11, operands T10 and T5. |
| 9 | Transfer code `0x26` to B, source T11. |
| 10 | Binary code 1 to T12, operands T10 and T6. |
| 11 | Transfer code `0x26` to C, source T12. |
| 12 | Binary code 1 to T13, operands T10 and T7. |
| 13 | Transfer code `0x26` to D, source T13. |

Unlike selector 3, this case does not issue a literal event directly targeting D. T8 receives literal 7 but has no later direct use here.

### Selector 5

After five obtain calls and the early publication of T2 to D:

1. Issue a literal event to T4 with value 1.
2. Issue a binary event with code 6, destination T3, operands Input Y and T4.
3. Issue a binary event with code 1, destination T5, operands T3 and Input X.
4. Issue a transfer event with code `0x27`, destination T1, source T5.
5. Issue a literal event targeting a fresh read of D, with value `0xFF`.
6. Invoke the range dependency three times with source T1, publishing each result immediately before the next invocation:

| Range arguments | Output |
|---|---|
| 11, 15 | A |
| 5, 10 | B |
| 0, 4 | C |

After the third result is stored, read the address-like field at Context offset `0x08` and then its byte at offset `0xB2`. If that byte equals 5, store T1 to E. Otherwise, perform no further direct store.

### Selectors 6 and 7

Both cases begin with four obtain calls and the same event sequence:

1. Literal to T2, value 1.
2. Binary code 6 to T1, operands Input Y and T2.
3. Binary code 1 to T3, operands T1 and Input X.
4. Transfer code `0x27` to T4, source T3.

Four range calls follow, all using T4 as their source. Each returned result is stored before the next call.

| Output | Selector 6 range arguments | Selector 7 range arguments |
|---|---|---|
| A | 12, 15 | 11, 15 |
| B | 8, 11 | 7, 10 |
| C | 4, 7 | 1, 5 |
| D | 0, 3 | 0, 0 |

After storing D, perform the Context-based byte read described for selector 5. Store T4 to E only when that byte equals the selected case value: 6 for selector 6 or 7 for selector 7.

Selector 7 passes no range containing position 6. This omission is observable in the arguments; whether it reflects a format rule is unknown.

### All other selector values

Obtain one token, publish it to D, B, C, and A in that order, then issue a literal event targeting that token with value `0xFF`.

There are no binary, transfer, or range events on this path.

## Side effects

### Direct memory effects

On a normally completing path, the function performs direct stores to all five output locations. E is always initially written with zero; cases 5–7 may write it again after the Context-based comparison.

Output values are not guaranteed to remain equal to their direct-store values:

- Output locations may alias one another.
- Outputs may overlap the descriptor or Context-related storage.
- Dependencies may modify memory reachable through their arguments, through aliases, or through otherwise accessible state.

For example, if A and D share storage in selector 0, the T2 publication to D replaces the earlier T1 publication to A before the event sequence begins.

The function imposes no explicit non-aliasing check. Its ordered accesses describe aliasing behavior only where the underlying memory accesses remain valid for the target execution environment.

### Dependency effects and ownership

Dependency calls are observable effects in their own right. Their global-state changes, resource effects, failure behavior, and memory effects are not established here.

This function contains no explicit release operation. It does not establish ownership transfer, reference retention, token freshness, or caller obligations to release returned values. In particular, an obtain call must not be described as allocating memory without a separate dependency contract.

No rollback is visible. Earlier stores and dependency effects precede any later failure or interruption; whether a dependency can fail or interrupt execution is unknown.

### Arithmetic and validation

The function performs no direct arithmetic on token values or Input X/Input Y. Codes 1 and 6 are dependency arguments, not evidence that this function directly adds, shifts, or otherwise computes on those values.

Range bounds are passed unchanged. Inclusive endpoints, extracted widths, scaling, sign extension, and normalization are not established.

## Dependencies

| Opaque ID | Established use |
|---|---|
| F-b35e541ebc78 | Produces 32-bit values from Provider and constant 0. Called 1–13 times according to selector. |
| F-cb523cc6fa7d | Receives Sink, code `0x26` or `0x27`, and two opaque values. Its result, if any, is ignored. |
| F-fbf0cbff8cb3 | Receives Sink, code `0x28`, a target value, and a literal. Its result, if any, is ignored. |
| F-915779408ccb | Receives Sink, code 1 or 6, a destination value, and two ordered operands. Its result, if any, is ignored. |
| F-e4de34edb462 | Receives Context, Sink, a source value, and two range arguments. Its 32-bit result is immediately published to an output. |

No dependency is assumed to preserve globals, output storage, or Context-related fields. Consequently, the final Context-based read in cases 5–7 observes state after all preceding calls and output publications, not entry state.

## Acceptance examples

These are conditional static acceptance scenarios, not executed tests. Unless stated otherwise, assume valid disjoint output storage, stable descriptor and Context data, normally returning dependencies, and no dependency modification of the caller’s output locations.

1. **Selector 0, distinguishable tokens.**  
   If the obtain calls return T1, T2, and T3, A/B/C finish with T1, D with T2, and E with zero. The call trace contains one binary event, one `0x26` transfer to T2, and a final literal-0 event to T1.

2. **Selector 1 preserves a different destination choice.**  
   With the same obtain results, A/B/C again finish with T1 and D with T2. The transfer targets T1, while the final literal-`0xFF` event targets T2. Treating this as selector 0 with only a changed literal is incorrect.

3. **Selector 4 retains the apparently unused literal event.**  
   Thirteen obtain calls occur. A/B/C/D receive T2/T3/T4/T1 respectively. The literal-7 event to T8 occurs despite T8 having no later direct use.

4. **Selector 5 with a nonmatching Context byte.**  
   D receives T2. A, B, and C receive the three range results in order. If the final observed Context-related byte is not 5, there is no final direct store to E; under the stated assumptions it remains zero.

5. **Selector 6 with a matching Context byte.**  
   The four range results are published to A through D. If the byte read afterward is 6, E receives T4, not T1 and not any range result.

6. **Selector 7 range boundaries.**  
   Acceptance requires exactly the argument pairs `(11,15)`, `(7,10)`, `(1,5)`, and `(0,0)` in that order. Replacing them with four uniform four-position ranges changes the contract.

7. **Unknown selector.**  
   A selector byte of `0xFF` follows the fallback path: one obtain call, publications to D/B/C/A, and one literal-`0xFF` event. It is not rejected.

8. **Output aliasing in selector 0.**  
   If A and D share one valid 32-bit location and dependencies leave it unchanged, that location contains T2 before event processing. Both the transfer destination read from D and the final literal destination read from A therefore yield T2.

9. **Late Context mutation.**  
   If a range dependency changes the Context-related byte before returning, the subsequent comparison uses that changed byte. Entry-state equality alone does not determine whether E is overwritten.

## Uncertainties

The following are **documented_only**, not inferred guarantees:

- Semantic meaning of the selector values and output channels.
- Meaning of operation codes `0x26`, `0x27`, `0x28`, 1, and 6.
- Whether obtained values are handles, indices, addresses, or another representation.
- Whether range arguments denote inclusive bit positions or any other extraction rule.
- Dependency error conventions, exception behavior, reentrancy, synchronization, and global-state effects.
- Ownership, allocation, retention, and release requirements.
- Valid input domains beyond the directly observed access requirements.
- Alignment, byte order, concurrency rules, and target-level behavior for invalid or partially overlapping accesses.
- Any reason for selector 3’s otherwise unused literal-3 target or selector 4’s otherwise unused literal-7 target.

## Evidence

This contract is derived solely from the supplied static representation of F-2d9b6ef1d9a1 and the supplied opaque dependency mapping.

Established evidence covers direct loads and stores, access widths and offsets, branch selection, literal argument values, dependency-call order, output reload timing, and conditional final publication.

Dependency implementations were not supplied. No runtime tests have been performed. The acceptance examples describe conditional consequences of the static behavior, not observed execution results.
