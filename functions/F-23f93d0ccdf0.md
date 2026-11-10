<!-- Trusted provenance: F-23f93d0ccdf0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-23f93d0ccdf0

## Purpose

F-23f93d0ccdf0 compares a value obtained from an associated object with a cached 32-bit value. A change triggers cache replacement and additional processing, with distinct paths for the all-ones sentinel and every other value. All normally completing invocations then perform two stages of condition-dependent processing.

The observable contract includes repeated reads, repeated queries, operand snapshots, and ordered calls. A replacement must not collapse these operations merely because their operands or results appear equivalent.

“Provider,” “cache,” and “update” describe structural roles. Their application-level meaning is inferred rather than established.

## Interface

### Input and result

The sole explicit input is a pointer to a mutable object. No return value is specified.

The representation uses 32-bit integer words and 32-bit pointer representations. Offsets below are byte offsets relative to the input object.

| Offset | Access | Established use |
|---|---|---|
| `0x000` | Read, 32-bit pointer | Dispatch-table pointer for object operations. |
| `0x030` | Read, 32 bits | First operand supplied to F-93e7ee291583. |
| `0x038` | Address passed | Embedded destination supplied to dependencies; its extent is unknown. |
| `0x050` | Read/write, 32 bits | Cached provider value. |
| `0x52C` | Write, 8 bits | Cleared when a value change is detected. |
| `0x700` | Write, 8 bits | Receives the low eight bits of a dependency result on a non-sentinel change. |
| `0x888` | Read, 32-bit pointer | Associated provider pointer. |
| `0x8B0` | Write, 32 bits | Receives the all-ones value on a non-sentinel change. |

The provider address plus `0x060` is passed to F-b7abcf60c990. This denotes an address within the provider, not a read of the word stored there.

### Indirect operations

The representation identifies indirect targets by dispatch-table byte offset, not by stable function identity:

| Receiver | Dispatch offset | Established use |
|---|---|---|
| Provider | `0x158` | Returns the 32-bit value compared with the cache. |
| Provider | `0x0EC` | Conditionally invoked on a sentinel change. |
| Provider | `0x170` | Receives a provider pointer, the new 32-bit value, and the embedded destination address. |
| Provider | `0x1B8` | Returns a 32-bit zero/nonzero condition. |
| Input object | `0x058` | Receives the input object and two 32-bit update operands. |
| Object returned by F-6a21f4f07f67 | `0x09C` | Receives that object and zero. |

Some provider calls have no explicit receiver argument in the supplied representation. Their complete calling conventions are not established. Results of the update operations and the operation at `0x09C` are not used here.

### Preconditions and failure boundary

Normal completion requires every reached memory access, dispatch target, dependency argument, and returned object used for dispatch to be valid.

There are no visible null checks, bounds checks, dependency-error translations, or recovery paths. Invalid addresses, invalid dispatch targets, and abnormal dependency termination have no established recovery behavior. Earlier writes are not rolled back by any visible mechanism.

## Behavior

### External operands and snapshots

Several operands come from external 32-bit words whose values are not supplied. This contract distinguishes:

- **Condition key:** a snapshot used by later condition-dependent updates.
- **Base snapshot:** the first read of a particular external condition word.
- **Reduced snapshot:** a separate read of that same external word, followed by subtraction of decimal 25.
- **Four-update key and four arguments:** operands for four consecutive non-sentinel updates.
- **Middle key and argument:** operands for the next update.
- **Two later keys and shared argument:** operands for two updates after an intermediate helper.
- **Common key and two arguments:** operands used after either change branch.
- **Two notification operands:** separate external operands used by the sentinel and non-sentinel notification dependencies.

These roles do not imply distinct numeric values.

The base and reduced snapshots are **not** established to differ by 25. That relationship requires the two reads of their shared backing word to return the same value, together with an applicable interpretation of the subtraction at integer boundaries.

### 1. Initial query and ordered captures

The routine first obtains the provider through the input object and invokes provider dispatch entry `0x158`.

After that call returns, the following reads occur in order:

1. Read the external condition key.
2. Read the external condition word into the base snapshot.
3. Read that same external condition word again, independently, and subtract 25 to form the reduced snapshot.

The initial provider call can therefore affect all three captured operands. No dependency call appears between these three reads, but the representation does not establish that the external condition word is immutable or that its two reads must agree.

The condition key, base snapshot, and reduced snapshot are retained for later use. Subsequent updates to their backing data do not refresh these snapshots.

After these captures, the returned provider value is compared with the object’s 32-bit cache at `0x050`.

The subtraction is represented as a 32-bit operation. Its exact behavior at signed arithmetic boundaries remains unresolved; this contract does not silently choose wrapping, trapping, or another interpretation.

### 2. Unchanged value

When the provider result equals the cached value:

- This routine does not replace the cache or clear the byte at `0x52C`.
- Both change-specific branches are skipped.
- The operations common only to changed values are also skipped.
- Execution proceeds to unconditional final processing.

This path still calls dependencies and can perform updates and returned-object operations.

### 3. Change bookkeeping

When the provider result differs from the cache:

1. Write the returned 32-bit value to `0x050`.
2. Write zero to the byte at `0x52C`.
3. Select a branch according to whether the returned value is the all-ones 32-bit sentinel, interpreted as signed −1.

Every other value takes the non-sentinel branch, including zero and negative values other than −1.

These writes precede all branch-specific calls.

### 4. Sentinel change

The sentinel branch performs the following ordered operations:

1. Read the sentinel notification operand and pass it to F-3f44bd7f1069.
2. Invoke F-c0c0b8633623 with no explicit arguments.
3. Invoke F-bc4e8f603b51.
4. If its 32-bit result is zero, invoke the current provider’s dispatch entry `0x0EC`.
5. Invoke F-ba5a4c1e0746 with the address of a 24-byte invocation-local temporary and two zero operands.
6. Pass the embedded destination address at `0x038` and the preceding dependency’s 32-bit result to F-a449eda002e0.
7. Invoke F-4e292a3dd745 with the temporary’s address.

No direct initialization of the temporary before F-ba5a4c1e0746 is shown. The later call on that temporary is cleanup-shaped, but neither initialization semantics nor resource release is established.

This branch does not directly write `0x700` or `0x8B0`. Dependencies may nevertheless modify accessible state.

### 5. Non-sentinel change

First, invoke provider dispatch entry `0x170`, supplying a provider pointer, the new value, and the embedded destination address.

Next, invoke F-e8ce52ce5425 on the embedded destination. Its represented 64-bit result is used in three different widths:

- Store its low eight bits at `0x700`.
- Test its low 32 bits for zero.
- If that low half is nonzero, pass the low and high 32-bit halves separately to F-23e8d37d5f69.

The half ordering is numerical: low half first, high half second. This does not establish the result’s memory byte order.

The conditional processing is:

- **Low half zero:** read the current word at `0x030` and invoke F-93e7ee291583 with that word and zero.
- **Low half nonzero:** invoke F-23e8d37d5f69 with both halves. If its result is nonzero, invoke F-1f374213a813, then read the current word at `0x030` and invoke F-93e7ee291583 with that word and the result of F-1f374213a813.
- **Low half nonzero and F-23e8d37d5f69 returns zero:** neither F-1f374213a813 nor F-93e7ee291583 is invoked at this point.

A nonzero high half does not override a zero low half. Likewise, a zero stored byte does not imply that the low 32-bit half was zero.

The branch then continues in this order:

1. Four object updates through entry `0x058`.
2. One object update using the middle key and argument.
3. Write all ones to the 32-bit field at `0x8B0`.
4. Invoke F-13b43f762d0e on the input object.
5. Two object updates using the two later keys and their shared argument.
6. Invoke F-2ae3a22c0939 on the input object.
7. Read the non-sentinel notification operand and pass it to F-710d98905737.

The update operands have important capture distinctions:

- Before the four-update sequence, capture the four-update key. The first update separately reads the backing key word for its own key operand. The remaining three updates use the captured key. Each update uses its corresponding external argument.
- The middle update reads its own key and argument.
- After the intermediate F-13b43f762d0e call, capture the shared argument for the two later updates. The first later update separately reads that backing word for its argument; the second uses the captured argument. Each uses its respective external key.

The captured key or argument need not equal its separately read first-call counterpart. Nor may later external reads be moved before intervening dependencies under an assumption that those dependencies preserve globals.

### 6. Operations common to changed values

After either change branch:

1. Perform two object updates.
2. Query the current provider through entry `0x1B8`.
3. If that result is zero:
   - Update using the captured condition key and reduced snapshot.
   - Invoke F-6a21f4f07f67.
   - Invoke the returned object’s entry `0x09C` with that object and zero.
4. Otherwise, update using the captured condition key and base snapshot.
5. Invoke F-a7af65ac69dc on the input object.

For the first two updates, the common key is captured before the first call. That first call independently reads the backing key word for its own key operand; the second uses the captured key. The two calls use their respective external arguments.

The zero-condition path uses the reduced snapshot derived from the **second** initial read of the condition word. The nonzero path uses the base snapshot from the **first** read.

### 7. Unconditional final processing

This processing follows both changed and unchanged paths:

1. Invoke F-13b43f762d0e on the input object.
2. Invoke F-b7abcf60c990 with the current provider address plus `0x060`.
3. If that result is zero, update using the captured condition key and base snapshot. Do not query provider entry `0x1B8` for this decision.
4. If that result is nonzero, query the current provider through `0x1B8`. Perform the base-snapshot update only when this query returns nonzero.
5. Independently invoke F-b7abcf60c990 again, using the then-current provider address plus `0x060`.
6. If this second result is nonzero, query the current provider through `0x1B8`. If that query is also nonzero, return immediately.
7. In every other case:
   - Update using the captured condition key and reduced snapshot.
   - Invoke F-6a21f4f07f67.
   - Invoke the returned object’s entry `0x09C` with that object and zero.
   - Return.

The two subobject queries are independent. The first result is not reused for the second decision. Provider condition queries are also independent and occur only where the short-circuit conditions require them.

## Side effects

### Direct writes

The visible persistent direct writes are:

- Changed-value cache replacement at `0x050`.
- Changed-value byte clear at `0x52C`.
- Non-sentinel low-byte result storage at `0x700`.
- Non-sentinel all-ones word storage at `0x8B0`.

Only the specified byte is written at each byte-store location. The routine also supplies writable-looking addresses to dependencies, but their write extents and effects are unknown.

### Calls, mutation, and ordering

Dependencies must not be assumed to preserve external words, the provider pointer, object fields, or dispatch tables. Later invocation sites obtain provider and dispatch information again rather than uniformly pinning the initial targets.

Where an indirect-call expression contains multiple reads, the representation does not establish an atomic snapshot or a complete order among those reads.

Assuming all reached calls return normally, F-13b43f762d0e is invoked:

- Once on an unchanged-value path.
- Once on a sentinel-change path.
- Twice on a non-sentinel-change path.

Repeated update pairs remain separate observable calls. No deduplication is authorized.

### Ownership and lifetime

The temporary occupies 24 bytes of invocation-local storage. Its address is passed to F-ba5a4c1e0746 and later to F-4e292a3dd745.

The routine shows no direct allocation, deallocation, reference-count operation, declared ownership transfer, or persistent pointer retention. Helper placement alone does not prove any of these effects. Retaining the temporary’s address beyond its lifetime would require an external contract not supplied here.

## Dependencies

| Opaque function ID | Established role |
|---|---|
| F-3f44bd7f1069 | Receives the sentinel notification operand. |
| F-c0c0b8633623 | Invoked next on the sentinel path. |
| F-bc4e8f603b51 | Its zero result enables provider entry `0x0EC`. |
| F-ba5a4c1e0746 | Receives the temporary and two zeros; produces a 32-bit operand. |
| F-a449eda002e0 | Receives the embedded destination and that operand. |
| F-4e292a3dd745 | Receives the temporary afterward. |
| F-e8ce52ce5425 | Produces the represented 64-bit result from the destination. |
| F-23e8d37d5f69 | Receives the result halves; only zero/nonzero of its result is used. |
| F-1f374213a813 | Produces the conditional second operand for F-93e7ee291583. |
| F-93e7ee291583 | Receives the current `0x030` word and a selected 32-bit operand. |
| F-13b43f762d0e | Receives the input object at the specified intermediate and final stages. |
| F-2ae3a22c0939 | Receives the input object late in the non-sentinel branch. |
| F-710d98905737 | Receives the non-sentinel notification operand. |
| F-6a21f4f07f67 | Returns an object used for dispatch at `0x09C`. |
| F-a7af65ac69dc | Receives the input object after change-only condition processing. |
| F-b7abcf60c990 | Queries the provider subobject twice during final processing. |

Indirect targets are identified only by the dispatch sites listed under Interface. Their stable opaque function IDs are unavailable. Dependency internals, ownership rules, and failure conventions are documented_only.

## Acceptance examples

These are conditional static predictions, not executed tests. Unless a scenario specifies otherwise, assume valid accesses, normal returns, and no unmentioned mutations.

1. **Separate initial reads produce different values.**  
   Suppose the first condition-word read returns 100 and the second returns 200. The base snapshot is 100 and the reduced snapshot is 175. A base update uses 100; a reduced update uses 175. Deriving 75 from the base snapshot would violate the represented read behavior.

2. **Stable initial condition word.**  
   If both reads return 100, the base snapshot is 100 and the reduced snapshot is 75. Their difference of 25 is justified here by the explicit stability assumption and representable arithmetic, not by snapshot construction alone.

3. **Unchanged cache; both final subobject queries return zero.**  
   No change-only direct writes occur. F-13b43f762d0e runs once. Neither final provider condition query occurs. A base-snapshot update precedes a reduced-snapshot update, followed by the returned-object operation with zero.

4. **Unchanged cache; both final subobject queries and both reached provider queries return nonzero.**  
   The first stage performs a base-snapshot update. The second stage returns without a reduced-snapshot update or returned-object operation.

5. **Final results change between queries.**  
   If the first subobject query returns zero, a base-snapshot update occurs. If the second returns nonzero and its provider query returns nonzero, final reduced processing is skipped. Reusing the first result would be incorrect.

6. **Sentinel change with a zero result from F-bc4e8f603b51.**  
   The cache becomes all ones and `0x52C` becomes zero before sentinel dependencies run. Provider entry `0x0EC` is reached. The temporary-related calls occur in preparation, destination-processing, then cleanup-shaped order.

7. **Non-sentinel result `0x0000000100000000`.**  
   The byte at `0x700` becomes zero. The low-half test is zero despite the nonzero 64-bit result. F-93e7ee291583 receives zero as its second operand; F-23e8d37d5f69 is skipped.

8. **Low half `0x00000100`; F-23e8d37d5f69 returns zero.**  
   The stored byte is zero, but the low-half test is nonzero. F-23e8d37d5f69 runs. F-1f374213a813 and the conditional F-93e7ee291583 call do not. Remaining non-sentinel processing continues.

9. **Changed value −2.**  
   This takes the non-sentinel branch. The sentinel comparison does not classify all negative values together.

10. **Zero change-only condition and zero final subobject conditions.**  
    The condition-update sequence is reduced snapshot, base snapshot, reduced snapshot. The returned-object operation occurs twice. This sequence does not imply a numeric difference of 25 between the snapshots unless their initial backing-word reads were stable.

## Uncertainties

- **documented_only:** Concrete external operand values and their application meaning.
- **documented_only:** Stability, synchronization, and atomicity of external data. Separate reads of the condition word are established; equality of their results is not.
- **documented_only:** Exact boundary behavior of the 32-bit subtraction used for the reduced snapshot.
- **documented_only:** Complete calling conventions for indirect calls without explicit receiver arguments.
- **documented_only:** Embedded destination layout, temporary format, and dependency write extents.
- **documented_only:** Dependency allocation, release, retention, ownership, and failure behavior.
- **documented_only:** Recovery from invalid memory, invalid dispatch targets, or nonlocal exits.
- **Inferred intent:** The cache comparison avoids some refresh work, while final processing reconciles condition-dependent state. No particular UI, device, or business interpretation is established.

## Evidence

This contract is based on static analysis of the supplied restricted representation and opaque dependency mapping. Embedded commentary and the prior draft are not treated as independent proof.

Established evidence includes access widths and offsets, branch structure, argument flow, direct-write order, repeated dispatches, and the initial ordered sequence of one condition-key read followed by two separate reads of the condition word. The reduced snapshot is formed from the second condition-word read, not from the stored base snapshot.

Dependency internals, application semantics, subtraction boundary behavior, and runtime stability are not established.

No runtime tests, execution traces, or independent implementation checks have been performed. The acceptance examples are conditional predictions, not observed results.
