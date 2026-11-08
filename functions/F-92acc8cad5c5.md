<!-- Trusted provenance: F-92acc8cad5c5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-92acc8cad5c5

## Purpose

F-92acc8cad5c5 conditionally selects one of three text references and dispatches a converted value through a caller-supplied receiver. Selection depends on an initial gate, an object lookup and enable byte, four ordered queries against temporary state, and a final query using a previously acquired service value.

After a successful dispatch return, the function performs another lookup, conditionally invokes an additional dependency, and performs its final temporary-state operation.

**Established behavior:** every normal return is the 32-bit value 1. Gate rejection, an absent or disabled first lookup result, either alternative selection, and an absent second lookup result do not produce different normal return values.

**Inferred intent:** the three text references appear to represent a start action and two error-related actions. Those meanings are suggested by static labels, not established by verified text bytes or dependency behavior.

## Interface

### Inputs and result

The function receives:

- **Dispatch receiver:** a pointer to an object whose first 32-bit word supplies a virtual-table address. On the active path, the callable entry at table byte offset `0x11C` receives this object and the value returned by F-9b9c1c7483f4.
- **Gate input:** an opaque 32-bit value passed to F-1bdc33170811. No other direct interpretation of this input is visible.

The normal output is a 32-bit value equal to 1. The function does not expose the dispatch result or query results as its return value.

A zero result from the initial F-6da76fc58c80 call leads to F-a817d8d58c10, which is marked nonreturning in the supplied representation. No normal result is established for that path.

### Widths and required layouts

The supplied representation uses 32-bit integer and pointer values. Four query dependencies and the final selection query produce represented 64-bit results. Their low 32 bits govern selection; their high 32 bits can become arguments on the alternative path.

The following accesses must be valid on paths reaching them:

| Object or storage | Required access |
|---|---|
| Dispatch receiver | First-word virtual-table pointer; callable entry at table offset `0x11C` |
| Each object returned by F-938c950e5046 | First-word virtual-table pointer; callable entry at table offset `0xF8` |
| Object returned by F-87be4aa684d1 | First-word virtual-table pointer; callable entry at table offset `0x15C` |
| Object returned by the alternative-path F-6da76fc58c80 call | First-word virtual-table pointer; callable entry at table offset `0x90` |
| Nonzero first lookup result | Readable byte at object offset `0x150` |
| Main temporary storage | 24 addressable local bytes |
| Text-related temporary storage | 4 addressable local bytes |

Offsets are byte offsets. No bounds or null checks are visible for the virtual-table objects. The first lookup result is checked for zero before its enable byte is read.

The temporary sizes establish the storage supplied by this function, not the internal formats or ownership contracts accepted by dependencies. The dispatch receiver is not dereferenced on paths that terminate before dispatch.

## Behavior

### 1. Initialization, gate, and first global snapshot

F-44874cacba18 is called unconditionally, without explicit arguments.

Next, F-1bdc33170811 receives the gate input. Its 32-bit result is tested against zero.

After that call, the function reads a first 32-bit global value and saves it for lookup operations. This read occurs even if the gate result is nonzero.

A nonzero gate result ends processing with return value 1. No provider acquisition, object lookup, enable-byte read, or temporary-state dependency follows.

### 2. First provider acquisition and lookup

When the gate result is zero, the following sequence occurs:

1. F-938c950e5046 supplies a provider object.
2. A second 32-bit global value is read and saved.
3. That second global is referenced again, as a separate source-level read supplying the argument to the provider’s virtual entry at `0xF8`.
4. The virtual entry is invoked with the provider object and the value from this separate argument read.
5. F-682adfd5a561 receives the virtual operation’s result followed by the saved first global value.

The saved second-global snapshot is retained for the later provider call. It is not the expression supplying the first provider call’s global argument.

No intervening dependency call is visible between the second-global snapshot read and its separate argument read. The representation therefore establishes two source-level reads, but does not establish different values for them, concurrent mutation, or volatile access semantics.

If F-682adfd5a561 returns zero, processing ends with return value 1. No enable byte is read and neither temporary is supplied to its setup dependency.

If the lookup returns nonzero, the function reads one byte at offset `0x150` from that result:

- Zero ends processing with return value 1.
- Any nonzero byte admits the active path.

The condition does not require the byte to equal 1. Signedness does not change this zero/nonzero decision. This function performs no later direct dereference of the first lookup result.

### 3. Main temporary and required service

For an enabled first lookup result, F-ba5a4c1e0746 receives the address of the 24-byte temporary and two zero arguments.

F-6da76fc58c80 is then called without explicit arguments at this call site. Its represented 32-bit result is tested against zero:

- If zero, F-a817d8d58c10 is called. It is annotated as nonreturning. No subsequent dispatch, temporary cleanup, or normal return is established.
- If nonzero, the value is retained for the final selection query.

F-87be4aa684d1 then supplies another object. Its virtual entry at table offset `0x15C` receives that object and the address of the 24-byte temporary.

There is no visible null check before this virtual call. Which temporary bytes the operation reads or writes is unresolved.

### 4. Ordered, short-circuiting queries

The function queries the 24-byte temporary through these dependencies, in order:

1. F-d1b4663968a8.
2. F-15632527c6ab.
3. F-e8ce52ce5425.
4. F-6e482452607b.

Each receives the temporary’s address. Each produces a represented 64-bit result, but only the low 32 bits determine whether evaluation continues.

A nonzero low half stops this sequence and selects the alternative path. Remaining queries, including the final service-dependent query, are skipped.

A zero low half allows evaluation to continue even if the high half is nonzero. Earlier high halves are not accumulated into a combined condition.

Only when all four low halves are zero does F-e3cdfecc712a run. It receives:

- The nonzero service value retained from the initial F-6da76fc58c80 call.
- The address of the 24-byte temporary.

Its represented 64-bit result determines the remaining choice:

- Nonzero low half selects the start text.
- Zero low half selects the alternative path.

Before that final low-half decision, the retained 64-bit intermediate is replaced with the final result’s high half and a zero low half. This masking matters to the arguments used on the alternative path.

The service value is not reacquired or revalidated before F-e3cdfecc712a. Continued validity across intervening dependency calls is an external requirement, not a guarantee established here.

### 5. Alternative-path arguments and selection

The alternative path calls F-6da76fc58c80 again, this time with two explicit 32-bit arguments.

Their bit patterns depend on where selection stopped:

| Reason for alternative path | First argument | Second argument |
|---|---|---|
| One of the four ordered queries had a nonzero low half | That query’s low 32 bits | That query’s high 32 bits |
| All four low halves were zero and the final query’s low half was zero | Zero | Final query’s high 32 bits |

The high half is obtained by shifting the represented unsigned 64-bit value right by 32 bits and then representing the result as a 32-bit argument. This preserves the relevant high-half bit pattern; it does not introduce a signed comparison.

The returned object is immediately used for a virtual call at table offset `0x90`. No null check is visible.

The supplied representation shows no explicit receiver argument for that virtual invocation. Its actual calling convention and implicit receiver handling remain unresolved.

The virtual operation’s 32-bit result selects:

- **Alternative text A** when zero.
- **Alternative text B** when nonzero.

### 6. Text conversion and dispatch

The selected text references are:

- **Start text:** one byte beyond the beginning of a statically referenced text location.
- **Alternative text A:** a directly referenced text location whose label suggests a missing-feature condition.
- **Alternative text B:** a directly referenced text location whose label suggests missing information.

Verified bytes, encoding, lengths, and termination are not supplied. The start pointer’s one-byte displacement is part of the established behavior; it must not be replaced with an assumed complete label spelling.

F-9b9c1c7483f4 receives the address of the 4-byte temporary and the selected text pointer.

Its returned value is passed to the dispatch receiver’s virtual entry at `0x11C`, together with the receiver. The virtual operation’s return value is ignored.

After dispatch returns, F-8a5aadac202f receives the address of the 4-byte temporary. Its placement suggests a cleanup role, but memory release or ownership effects are not established.

### 7. Second lookup and final temporary operation

After F-8a5aadac202f returns:

1. F-938c950e5046 supplies a provider object again.
2. This object’s virtual entry at `0xF8` receives the object and the **saved second-global snapshot** from before the first provider call.
3. F-682adfd5a561 receives the virtual operation’s result followed by the **saved first-global snapshot**.
4. If this lookup result is nonzero, F-84de48694ae6 is called without explicit arguments.
5. F-4e292a3dd745 receives the address of the 24-byte temporary.
6. The function returns 1.

Neither global is reread for this lookup. In particular, the second provider call uses the saved second-global snapshot, not a new global read and not a separately saved copy of the first provider call’s argument read.

The provider itself is reacquired, so it need not be the original provider object. The second lookup result need not match the first.

No enable-byte read occurs for the second lookup. A nonzero result alone triggers F-84de48694ae6, and the lookup result is not explicitly passed to that dependency.

## Side effects

The significant observable ordering is:

- Initialization precedes the gate call.
- The first-global snapshot read follows the gate call, including on gate rejection.
- On gate acceptance, provider acquisition precedes both second-global reads.
- The saved second-global read precedes the separate global argument read for the first provider invocation.
- No dependency call is visible between those two second-global reads.
- The first lookup precedes any enable-byte access.
- Main-temporary setup precedes initial service acquisition.
- Text conversion precedes dispatch.
- Dispatch precedes the text-temporary operation.
- That operation precedes the second provider acquisition and lookup.
- The optional F-84de48694ae6 call precedes the final main-temporary operation.

The function saves values and supplies local storage to dependencies. No direct modification of caller-owned object fields or global storage is shown. Dependencies may modify reachable storage or globals; their effects are unknown.

The representation warns of overlapping global symbols. Distinct global roles in this contract do not establish independent backing storage or independence from statically referenced text data.

No direct heap allocation or deallocation is shown. Setup-like and cleanup-like call placement does not establish that dependencies allocate, retain, transfer, or release memory.

There is no visible rollback, exception handler, or cleanup guard. The stated cleanup sequence applies only when intervening calls return normally.

## Dependencies

| Dependency | Established call-site role |
|---|---|
| F-44874cacba18 | Unconditional initial operation |
| F-1bdc33170811 | Gate using the second input |
| F-938c950e5046 | Acquires a provider before each lookup |
| Provider virtual entry `0xF8` | Produces the first lookup argument; initially receives a separate global read, later the saved snapshot |
| F-682adfd5a561 | Produces each lookup result from the provider result and saved first-global value |
| F-ba5a4c1e0746 | Receives 24-byte temporary storage and two zeros |
| F-6da76fc58c80 | Supplies the initial service value and, conditionally, an alternative-selection object |
| F-a817d8d58c10 | Annotated nonreturning operation for a zero initial service value |
| F-87be4aa684d1 | Supplies the object used with the main temporary |
| Population virtual entry `0x15C` | Receives the object and main temporary |
| F-d1b4663968a8, F-15632527c6ab, F-e8ce52ce5425, F-6e482452607b | Ordered, short-circuiting 64-bit queries |
| F-e3cdfecc712a | Final query using the retained service value and main temporary |
| Alternative virtual entry `0x90` | Selects alternative A or B by zero/nonzero result |
| F-9b9c1c7483f4 | Converts selected text using 4-byte temporary storage |
| Receiver virtual entry `0x11C` | Dispatches the converted value |
| F-8a5aadac202f | Post-dispatch operation on the text temporary |
| F-84de48694ae6 | Additional operation when the second lookup is nonzero |
| F-4e292a3dd745 | Final operation on the main temporary |

No dependency is assumed to preserve globals, preserve object validity, or implement a particular ownership policy.

## Acceptance examples

These are conditional static acceptance cases, not executed tests. Unless stated otherwise, they assume valid accessed storage and normally returning dependencies.

1. **Gate rejection.** A nonzero gate result permits initialization, the gate call, and the first-global snapshot read. It prevents provider acquisition, second-global reads, lookup, and dispatch. The normal result is 1.

2. **Separate initial global reads.** With an accepted gate, the first provider acquisition is followed by a saved second-global read and a separate second-global argument read. The latter supplies the initial `0xF8` invocation. No dependency call occurs between these reads, and this case does not require different values.

3. **Missing or disabled first object.** A zero first lookup result prevents the enable-byte read. A nonzero result with enable byte zero prevents temporary setup. An enable byte of `0x80` admits the active path just as a byte of 1 does.

4. **Unavailable required service.** If the initial service result is zero, the main temporary has already been supplied to F-ba5a4c1e0746. F-a817d8d58c10 is called next. Normal cleanup and return are not established.

5. **Start selection with nonzero high halves.** If all four ordered query results have zero low halves, nonzero high halves do not stop evaluation. A final-query low half of 1 selects the start pointer with its one-byte displacement.

6. **Early alternative selection.** If the first ordered query has low half 2 and high half 7, the other three ordered queries and F-e3cdfecc712a are skipped. The alternative F-6da76fc58c80 call receives 2 and 7. A zero `0x90` result selects alternative A; any nonzero result selects B.

7. **Final-query alternative.** If all four ordered low halves are zero and the final query has low half zero and high half 9, the alternative service call receives zero and 9.

8. **Globals changed during dispatch.** If dispatch changes either global, the second lookup still uses the earlier snapshots. Its provider is newly acquired, and its `0xF8` argument is the saved second-global value rather than a fresh read.

9. **Second lookup succeeds without an enable check.** A nonzero second lookup result triggers F-84de48694ae6 even if an enable byte would be zero. This function does not read that byte on the second lookup. The final main-temporary operation follows, then return value 1.

## Uncertainties

The following remain **documented_only**:

- Exact text contents, encoding, termination, and semantic effect of the start pointer’s one-byte displacement.
- Meanings of the gate, global values, queries, and virtual operations.
- Global volatility, concurrent access rules, and the physical read behavior corresponding to the separate source-level references.
- Consequences of overlapping global symbols.
- Whether represented 64-bit returns are native scalar results, paired values, or incomplete type reconstruction.
- The actual prototype of F-6da76fc58c80, given its different explicit argument lists.
- Receiver handling and calling convention for virtual entry `0x90`.
- Internal layouts and initialization requirements of both temporaries.
- Ownership, allocation, retention, and release behavior of dependencies.
- Continued validity of retained values and objects across intervening calls.
- Behavior after invalid accesses, dependency failure, exceptions, or other abnormal control transfer.
- Accuracy of the nonreturning annotation on F-a817d8d58c10.

Visible query arithmetic consists of low-half interpretation, high-half extraction, and masking away the final query’s low half. No arithmetic overflow condition governs selection.

## Evidence

This contract is based on the supplied static function representation and opaque dependency mapping. Established claims cover visible branches, call ordering, source-level global reads, saved-value reuse, byte offsets, temporary sizes, integer-width handling, and the constant normal return.

The second global is visibly read into a saved value and separately referenced for the initial provider-call argument. The later provider call uses the saved value. These are distinct data-flow facts; neither proves that the initial reads yield different values.

Dependency implementations and verified text bytes were not supplied. Semantic interpretations are identified as inferred, and unresolved behavior remains documented rather than assumed. No runtime execution or tests have been performed.
