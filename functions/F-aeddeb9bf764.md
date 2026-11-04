<!-- Trusted provenance: F-aeddeb9bf764; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aeddeb9bf764

## Purpose

F-aeddeb9bf764 evaluates up to four candidates in a fixed priority order through F-a1e68ae27737. The first nonzero dependency result selects a classification, which this function writes into a 32-bit field at destination offset 0x18.

| Candidate byte offset | Classification selected by a nonzero result |
|---|---:|
| 0x48 | 4 |
| 0x4C | 5 |
| 0x50 | 6 |
| 0x54 | 7 |

If all four results are zero, the function returns without performing its classification store. It does not explicitly clear the destination or write a failure value.

**Established behavior:** selection depends on the zero/nonzero results of ordered dependency calls.

**Inferred intent:** the function appears to classify an input by first-success priority. Neither the meaning of the candidates nor the meaning of “success” is established. In particular, the dependency is not established to perform equality comparison.

## Interface

### Explicit inputs

**Destination address carrier**

The first argument supplies the address used for the final store. The destination field begins at byte offset 0x18 from that address and is 32 bits wide.

The supplied representation uses a 32-bit integer address carrier. This does not independently establish a complete calling convention, valid address range, alignment rule, or overflow model.

This argument is not forwarded to F-a1e68ae27737. That omission does not establish that the dependency cannot access destination memory: it may reach that memory through globals, aliases, or other state.

**Opaque 32-bit input**

The second argument is a 32-bit quantity passed unchanged as the first argument to every dependency invocation. This function does not directly interpret or dereference it.

Whether this quantity represents a scalar, address, handle, or other value is unknown.

### Implicit input

A global location supplies a candidate-record base address. The function retains a base before the first dependency call. Candidate accesses after the first call use that retained base.

The first candidate access is separately expressed through the global base. There is no dependency call between retaining the base and reading the first candidate. Ordinary execution with a stable global therefore uses the same record for all reached candidates. Behavior under asynchronous changes during those initial operations is not established.

### Return

There is no specified return value. Normal completion occurs either:

- after writing a classification selected by a nonzero dependency result; or
- after four zero results, without the classification store.

Any incidental machine return-register contents are outside this contract.

### Relevant layout

All offsets below are byte offsets.

| Address origin | Offset | Access width | Access condition |
|---|---:|---:|---|
| Global-resolved candidate base | 0x48 | 32-bit read | Reached before the first dependency call |
| Retained candidate base | 0x4C | 32-bit read | First result is zero |
| Retained candidate base | 0x50 | 32-bit read | First two results are zero |
| Retained candidate base | 0x54 | 32-bit read | First three results are zero |
| Destination address carrier | 0x18 | 32-bit write | Any reached result is nonzero |

These are access roles, not proof of separate allocations. Candidate storage, destination storage, and global storage are not established to be disjoint.

There is no explicit field load addressed through the first argument. Nevertheless, a candidate read can read destination storage when the addresses alias. The layout does not support a general claim that destination fields are never read.

### Preconditions for normal completion

Every memory access actually reached must be valid in the execution environment:

- The global location must be readable.
- Each reached candidate field must support its 32-bit read.
- The retained base must remain usable for later reads when earlier results are zero.
- On a selected path, destination offset 0x18 must support a 32-bit write.
- Every reached dependency call must accept its arguments and return normally.

The function performs no explicit null, bounds, alignment, lifetime, or ownership checks. It does not need a valid destination store location on the all-zero path for its own direct accesses, provided that location is not also reached through another alias or by the dependency.

## Behavior

### Ordered evaluation

The function retains a candidate base and reads the first 32-bit candidate at offset 0x48 through the global-referenced address. It calls F-a1e68ae27737 with the opaque input first and that candidate second.

A nonzero result selects classification 4. The function then performs its destination store and returns. No later candidate read or dependency call is reached through this invocation’s shown control flow.

A zero first result causes a read at retained-base offset 0x4C, followed by another call with the same opaque input and the newly read candidate. A nonzero second result selects classification 5.

If the first two results are zero, the function reads retained-base offset 0x50 and makes the third call. A nonzero third result selects classification 6.

If the first three results are zero, it reads retained-base offset 0x54 and makes the fourth call. A nonzero fourth result selects classification 7. A zero fourth result causes return without the classification store.

A normally completing invocation therefore makes one to four dependency calls. Four calls occur both when the fourth candidate is selected and when none is selected. The result’s magnitude does not affect the classification; only zero versus nonzero matters.

These call counts describe calls made directly by this invocation. They do not constrain additional activity initiated internally by the dependency.

### Final destination store

After a selected dependency call returns, the function writes the corresponding value, 4 through 7, as one full 32-bit field at destination offset 0x18.

The store does not explicitly load or merge the field’s previous contents. It replaces the whole field rather than performing an increment, mask, or partial-field update.

This does **not** establish that the destination’s prior contents are irrelevant to selection:

- The destination field may alias a reached candidate field, making its contents a candidate argument.
- The dependency may independently read destination memory and use it in deciding its result.
- Earlier dependency calls may change memory that influences later candidates or results.

The established local property is therefore an unconditional replacement of the complete 32-bit field **after selection**, without an explicit prior-value load for that replacement. Independence of selection from destination state is not established.

There is at most one direct classification store in a normally completing invocation. It occurs after the selected call returns. Earlier dependency calls may have performed other writes, including writes to the same field.

### Read timing and mutable state

Candidates are read on demand rather than copied into a four-element snapshot.

Each later candidate read follows the preceding zero-returning dependency call. If that call changes the later candidate field, the next call receives the value read after that change, assuming no other interference. Previously consumed candidates are not reread.

A dependency change to the global candidate pointer does not redirect the later reads: those reads use the retained base. However, retaining an address does not preserve the object’s lifetime or contents. The dependency may modify or release the original storage; safe later access remains a precondition.

The initial base retention and first candidate resolution are separately expressed through global state. The evidence does not establish an exact machine-level load count, volatile semantics, or coherent behavior under concurrent mutation.

### Widths and arithmetic

Each candidate read and the opaque explicit input carry 32 bits. The final classification store is also 32 bits. Classifications 4, 5, 6, and 7 fit without truncation.

The dependency result is represented as an integer and is tested against zero. Its full ABI representation and semantic range are not independently supplied. Any representable nonzero result follows the selection branch, including a negative result if supported by that interface.

Address calculations add fixed byte offsets. The largest candidate offset is 0x54; its four-byte field occupies offsets 0x54 through 0x57. The destination field occupies offsets 0x18 through 0x1B.

No explicit address-overflow checks or byte-order conversions are present. Valid wrapping behavior, signed address-arithmetic semantics, pointer provenance, unaligned access behavior, and native byte order remain environment-dependent and unresolved.

## Side effects

The direct effects visible in this function are:

- reading global base information;
- reading each reached candidate field;
- invoking F-a1e68ae27737 sequentially; and
- writing one selected 32-bit classification, if any result is nonzero.

There is no explicit allocation, release, reference-count operation, or ownership transfer in this function. This does not constrain the dependency’s ownership or memory effects.

Aliasing can make the access roles overlap. For example, a candidate read may access the same four bytes later used for the classification store. Once selection occurs, the shown control flow performs no further candidate reads before returning, so the final store cannot influence a later candidate read by this invocation. This is a local ordering property, not a general alias-safety guarantee.

On the all-zero path, this function performs no classification store. The destination’s contents need not equal their entry value: dependency calls or other agents may have changed them.

The function provides no explicit rollback. Persistence of earlier effects is also not guaranteed. Later dependency calls may overwrite, reverse, or otherwise change earlier effects, regardless of whether those later calls return zero or nonzero. The caller’s lack of rollback says nothing about which dependency effects remain at completion.

There is no explicit synchronization or transaction boundary. Cross-thread visibility and atomicity are not established by the 32-bit access width alone.

There is no explicit error reporting, retry policy, or local recovery handler. A zero dependency result advances evaluation or ends the all-zero path; it is not separately reported as an error.

If a dependency call fails to return normally, the subsequent reads, calls, and classification store are not reached through the shown control flow. Memory faults and other abnormal outcomes are governed by the execution environment and unknown dependency behavior.

## Dependencies

### F-a1e68ae27737

The visible invocation contract is:

- Two arguments are supplied.
- The first is the unchanged opaque 32-bit input to F-aeddeb9bf764.
- The second is the 32-bit candidate read immediately for that invocation.
- The returned integer is tested only for zero versus nonzero.
- Calls are sequential; each reached call returns before this function advances.
- A nonzero result prevents this invocation from evaluating lower-priority candidates.

The following properties are **documented_only**:

- Whether the dependency compares, validates, searches, or performs another operation.
- Whether either argument denotes memory.
- Whether zero or nonzero represents an error in the dependency’s own interface.
- Whether it reads or writes destination memory, candidates, the global pointer, or unrelated state.
- Whether it retains arguments, acquires ownership, allocates, or releases storage.
- Whether it reverses earlier effects.
- Whether it is deterministic, reentrant, or thread-safe.
- How it reports or handles abnormal conditions.

No preservation of globals, candidate contents, destination contents, or object lifetimes is assumed.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Unless stated otherwise, assume valid reached accesses, normal dependency returns, stable relevant storage, and no dependency side effects affecting the example.

1. **First result is nonzero.**  
   Exactly one direct dependency call occurs, using the candidate at offset 0x48. The function writes 4 at destination offset 0x18. It does not directly read the remaining candidate fields.

2. **Second result is the first nonzero result.**  
   The first two results are zero and nonzero. Calls use offsets 0x48 and 0x4C in that order. The function writes 5 and does not reach offsets 0x50 or 0x54.

3. **Third result is negative and nonzero.**  
   If the interface supports −1, results of zero, zero, and −1 select classification 6. The negative sign does not create a separate branch.

4. **Fourth result is nonzero.**  
   Results of zero, zero, zero, and nonzero produce four calls followed by a store of 7.

5. **All results are zero.**  
   Four calls occur and this function performs no classification store. If the destination field starts as 99 and no other operation changes it, it remains 99.

6. **An earlier call changes a later candidate.**  
   The first call changes retained-base offset 0x4C and returns zero. The second call receives the subsequently read value, not an entry-time snapshot.

7. **The global pointer changes.**  
   The first call changes the global candidate pointer and returns zero. If the original record remains valid, the second candidate comes from offset 0x4C of the retained original base.

8. **The destination aliases the second candidate.**  
   Suppose destination offset 0x18 and retained-base offset 0x4C identify the same four bytes. The first call returns zero. The second candidate read therefore reads the destination field. If the dependency returns nonzero for that candidate, this function subsequently replaces those bytes with 5. The destination’s pre-store contents can participate in selection.

9. **The dependency consults destination memory independently.**  
   Suppose the dependency can reach the destination through external state and bases its first result on that field’s contents. A nonzero result selects 4. This behavior is compatible with the visible caller even though the destination address is not explicitly forwarded.

10. **Later calls reverse earlier effects.**  
    Suppose the first call changes an external field and returns zero, and the second restores that field and returns zero. If all remaining results are zero, this function performs no classification store. The first call’s change need not persist.

11. **The all-zero path includes a destination modification.**  
    A dependency call changes the destination, and all four results are zero. This function neither restores the entry value nor adds a classification store. The eventual contents depend on all intervening effects.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the candidates, opaque input, and classifications.
- Complete record layouts, allocation sizes, alignment, and ownership.
- Global initialization and synchronization rules.
- Exact ABI, address-space rules, endianness, and arithmetic semantics.
- Dependency access paths, memory effects, lifetime effects, and failure behavior.
- Which storage regions alias in any particular invocation.
- Whether destination state influences dependency results.
- Persistence or reversal of dependency effects.
- Behavior under invalid memory, concurrent mutation, or abnormal termination.

First-success classification is inferred intent, not an established domain meaning. No runtime tests have been performed.

## Evidence

The supplied static body establishes the candidate offsets and priority order, the zero/nonzero branch condition, classifications 4 through 7, and the final 32-bit store at destination offset 0x18. It also establishes the no-store return after four zero results.

It shows a retained candidate base for later reads and places each later read after the preceding zero-returning call. It contains no explicit load of the destination field through the first argument before the final store.

That absence does not exclude reads of destination storage through aliases or within F-a1e68ae27737. Likewise, the absence of explicit rollback does not establish that earlier effects persist.

The dependency implementation, surrounding type definitions, execution environment, and runtime observations were not supplied. Their behavior is left unresolved rather than supplied by convention or presumed intent.
