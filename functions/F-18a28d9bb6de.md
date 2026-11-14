<!-- Trusted provenance: F-18a28d9bb6de; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-18a28d9bb6de

## Purpose

F-18a28d9bb6de is a globally controlled, state-dispatched operation that can inspect context and table data, invoke dependencies, copy individual bytes, and publish a result word, count, and flag.

**Established static behavior:** these effects belong to particular dispatcher states. The supplied source does not establish a normal state sequence, which states are reachable for particular arguments, or a universal success or failure path. The global values controlling dispatch and arithmetic are unavailable.

**Inferred intent:** some paths appear to obtain an address-like result and associated count using context-backed dependencies or table data, with optional copying. This does not establish a retrieval API, allocation contract, count unit, or ownership model.

Two distinctions govern the contract:

- Caller arguments are saved separately from active addresses. The working context address and output pointers begin at zero; a particular state installs the caller arguments.
- Copy setup takes its destination from the **current retained working word**. That word begins at zero and can receive a dependency return or values propagated through other states, including a retained table-source address. Dependency-result provenance is conditional on the preceding state sequence.

## Interface

The visible interface has five positional inputs and an unsigned 32-bit return value.

| Position | Visible representation | Established use |
|---|---|---|
| 1 | Pointer to a signed 32-bit integer | Saved, then potentially installed as the count-output pointer. |
| 2 | Pointer to a byte | Saved, then potentially installed as the flag-output pointer. A separate state tests the active pointer for null. |
| 3 | Signed 32-bit integer | Saved, then potentially installed as the working context address; used in address arithmetic and a nonzero test. |
| 4 | Pointer to an unsigned 32-bit integer | Saved, then potentially installed as the result-output pointer. |
| 5 | Signed 32-bit integer | Multiplied by a global value in the argument-installation state and retained for subsequent arithmetic and dependency arguments. |

Address-bearing integer words and table fields are 32 bits. Context-field reads and word-output writes are 32 bits. Copy accesses and flag publication are one byte. Flag construction also uses signed 16-bit conversions. The source does not establish a portable 64-bit interface.

The return value is a mutable unsigned 32-bit working word, initially zero. States can replace it with constants, scaled working values, transformed dependency returns, or arithmetic intermediates. It is not established as a conventional status code.

### Active addresses

Before argument installation:

- The working context address is zero.
- The working count-output pointer is null.
- The working result-output pointer is null.
- The working flag-output pointer is null.

The installation state activates the saved context and output addresses, records whether the supplied context value is nonzero, resets another working word to zero, and scales the fifth argument.

Initialization-before-use is not enforced independently of dispatch. Valid caller pointers therefore do not make an access safe if the corresponding working pointer has not yet been installed.

### Conditional storage requirements

Every reached access requires valid storage at its current calculated address:

- Context access requires a readable 32-bit word at the working context address plus `0x20`.
- Result and count publication require writable 32-bit destinations.
- Flag publication requires a writable byte destination.
- Table access requires readable fields at the calculated entry address.
- Each copy assignment requires a readable source byte and writable destination byte.

Before argument installation, a context-field access nominally targets address `0x20`; output publication uses null working pointers.

The word-publication state has no null checks. The flag-publication state likewise has no local null check. A different state tests the flag pointer, but missing transition values prevent establishing that this test guards every write.

No explicit destination capacity, table length, input-length validation, or address-overflow check is visible.

## Behavior

### Dispatch and termination

Mutable working values, including the count, return word, retained copy-destination candidate, active addresses, counters, and arithmetic intermediates, begin at zero. The initial dispatcher state comes from global storage.

Each iteration has these significant read boundaries:

1. An arithmetic global is captured.
2. An unsigned 32-bit difference between the dispatcher state and a global dispatch base is checked.
3. A difference greater than `42` returns the current working return word.
4. For an in-range iteration, another global is read, interpreted as signed 32-bit data, and shifted right by `27` before the selected state body executes.

The out-of-range path performs the first capture but not the in-range shift read.

There are 43 state bodies. One returns the working return word directly. Another replaces the dispatcher state with a global value. Other states form transitions using globals, predicates, and arithmetic results.

The dispatch base occurs separately in range selection and state selection. The representation does not justify treating all global accesses as one immutable configuration snapshot.

Termination, state reachability, and a normal execution order remain unresolved. Repeated entry into the argument-installation state repeats its assignments and fifth-argument scaling.

### Arithmetic and predicates

The operation mixes unsigned 32-bit arithmetic, signed 32-bit interpretations, signed right shifts, character-sized conversions, and signed 16-bit flag intermediates.

Unsigned word calculations retain 32-bit results. Signed expressions cannot safely be treated as unlimited mathematical arithmetic. Target behavior for signed overflow, signed conversion, character signedness, and address wrapping remains unresolved.

Several transitions narrow predicates through a character-sized conversion. Others inspect bit 31 or use a signed right shift. Missing globals prevent simplifying these operations into ordinary index increments, identity scaling, or familiar search comparisons.

One repeated unsigned predicate is identifiable independently of those globals: it is one exactly when its input word is zero. Input one yields zero, and an explicit override makes every input greater than one yield zero.

Another state compares saved destination-like and source-like words using unsigned less-than. That comparison influences dispatch; it does not establish overlap-safe copying.

### Context-backed operations

One state reads the word at working context address plus `0x20` and records whether it is nonzero.

Three dependency-call states also read that field while forming arguments. These are separate reads, not a shared field snapshot:

- F-b09db3398a5a receives the context field, a globally scaled working word, two zero arguments, and another global word.
- F-efda6d785d87 receives a retained working word, the address of the mutable count, and the context field.
- F-f0c4ba604fa0 receives a retained working word, the context field, the count address, another working word, a globally scaled count, and literal one.

Before argument installation, each context read nominally targets `0x20`. After installation, it targets the supplied context address plus `0x20`.

The signed 32-bit count begins at zero. A separate state resets it to zero; a table state can replace it; dependencies receiving its address may modify it. Their count constraints and mutation contracts are unavailable.

The scaled-count argument to F-f0c4ba604fa0 is formed before entering that dependency. Mutation through the count address during the call cannot retroactively change that by-value argument, but can affect subsequent count uses.

### Capture boundary across F-b09db3398a5a

The arithmetic global captured at iteration entry survives this call and participates in post-call calculations.

One retained result combines that capture, the dependency return, and a multiplier read after the call. A second return-derived calculation combines the same unshifted capture with its signed right shift by `28`, then contributes to the working return word and a sign-bit predicate.

The second dependency argument separately reads its scaling global during argument formation. This read is not specified as reuse of the iteration-entry capture.

Multiplier reads in the return-derived calculations and global reads forming the next transition occur after the call. The in-range shift-by-`27` read occurred before the state body and is distinct from the later shift-by-`28` use of the retained capture.

Consequently:

- Changing the captured global inside the dependency does not replace the captured value.
- Changing a subsequently read multiplier can affect the post-call calculation.
- Caching all globals across the call would change the represented read structure.
- Rereading all globals after the call would also change it.

These are statement-level capture boundaries, not a total ordering among all operand reads within compound expressions.

### Table-backed operations

Entries have a 16-byte stride, addressed from a global table base using a transformed working position.

| Entry offset | Width | Established use |
|---|---:|---|
| `0x00` | 32 bits | Arithmetic involving working selection or search values. |
| `0x04` | 32 bits | Transformed arithmetic feeding a sign-bit-based transition. |
| `0x08` | 32 bits | Copy-source address; can also enter retained-result propagation. |
| `0x0c` | 32 bits | Globally transformed into the mutable count. |

One state calls F-0e5a7929e73f without arguments, retains its signed 32-bit return for later arithmetic, and resets a table-position word to zero. Other states transform positions and selection predicates.

No table size, ordering, matching rule, or safe traversal bound is established.

A table state transforms the field at `0x0c` into the count, calls F-d33f6d597e5d with a scaled count, and stores its return in the retained working word later available to copy setup. A separate call to the same dependency stores its return elsewhere; the two return destinations must not be conflated.

Copy setup performs these assignments in order:

1. Reads the current table entry’s field at `0x08` into the saved copy-source address.
2. Copies the **current retained working word** into the saved copy-destination address.
3. Derives an extent-like signed value from the count using a global multiplier.

The retained destination candidate starts at zero. In addition to receiving the table-associated dependency return, it can receive propagated working values. The source supports propagation of a saved table-source address through other states into this retained word.

Thus, reaching copy setup does not establish that any dependency has run, that the destination came from a dependency, or that it identifies allocated storage. Dependency-return provenance applies only when the preceding state sequence supplies it. A table-derived destination likewise requires the corresponding propagation sequence.

After setup, copy states use the saved source and destination addresses; they do not themselves reread the table source field.

### Byte copying

Two state bodies each perform one source-byte read and one destination-byte write at corresponding calculated displacements. Each displacement is a working counter multiplied by a global word.

Their counter updates differ:

- One derives the next counter from the calculated displacement.
- The other combines its counter, global factors, and the iteration-entry arithmetic capture.

Other states initialize counters or calculate copy-related transition predicates.

For a completed assignment, the source byte is obtained before storage to the destination. The subsequent counter update follows the assignment.

The source does not establish consecutive displacements, a conventional byte count, complete region coverage, destination capacity, or overlap safety. An extent-like value does not establish a checked bound on every access. No bulk-copy dependency or per-byte capacity check is visible.

### Output publication

The word-publication state writes:

1. The current result word through the working result-output pointer.
2. A globally scaled count through the working count-output pointer.

The second scaling expression occurs after the first write. If the first destination aliases its multiplier global, the first write can affect the second stored value.

After argument installation, these destinations correspond to the fourth and first caller arguments respectively. Before installation, both working pointers are null.

A separate state publishes the flag. Its constructions produce two candidate bytes:

- The zero construction produces `0`.
- The nonzero construction produces `1`.

The nonzero construction uses marker `175`, signed 16-bit conversion, multiplication, masking, and final byte multiplication. It does not publish marker `175`.

The separate flag-pointer test examines the active pointer. Before argument installation, it tests null even if the caller supplied a non-null second argument.

Ordering between flag publication and word publication is unresolved. No commit, rollback, or all-or-nothing output protocol is visible.

## Side effects

Potential direct effects include:

- Repeated reads of dispatch and arithmetic globals.
- Context-field and table-field reads.
- Dependency calls.
- Mutation of the local count through dependency arguments.
- Calculated byte writes.
- Two word-output writes and a separate byte-output write.

No direct assignment to a named global or explicit resource release appears. Pointer-based writes can nevertheless alias globals, context data, table data, or other outputs.

Dependency effects are unknown. Dependencies may change globals or reachable memory and may acquire or release resources. Their return values do not establish ownership, lifetime, or release responsibility. In particular, copy setup neither proves allocation nor transfers ownership.

No explicit diagnostic, exception handler, fault recovery, or transactional restoration is visible. An invalid access or dependency-side failure may follow completed reads, calls, or writes. Invalid-access behavior is not defined by the supplied evidence.

## Dependencies

| Opaque ID | Established interface and use | Unresolved contract |
|---|---|---|
| F-d33f6d597e5d | Two call sites, each accepting one transformed count word. Returns enter different working words and undergo zero testing. One return destination is a possible later copy-destination source. | Allocation, ownership, accessible size, side effects, failure meaning. |
| F-0e5a7929e73f | No arguments; returns a signed 32-bit value used in table-related arithmetic. | Whether the result represents a bound, count, identifier, or another quantity. |
| F-f0c4ba604fa0 | Six arguments, including context data, count address, working words, scaled count, and literal one. Return undergoes global scaling. | Count mutation, memory access, resource effects, return meaning. |
| F-b09db3398a5a | Five arguments. Return calculations combine a pre-call capture with post-call global reads. | Resource semantics, ownership, side effects, failure convention. |
| F-efda6d785d87 | Three arguments: retained working word, count address, and freshly read context data. Return undergoes arithmetic and predicate formation. | Count semantics, memory effects, return meaning. |

None is established as a standard allocator, reader, releaser, or copying primitive.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes the stated state sequence is reached and the relevant arithmetic and accesses are valid.

1. **Immediate out-of-range return.**  
   If the initial unsigned dispatcher difference exceeds `42`, the operation captures the iteration arithmetic global and returns zero. It performs no in-range shift-by-`27` read, state-body dependency call, or output write.

2. **Context use before argument installation.**  
   A context-reading state reached before installation nominally accesses `0x20`, regardless of a valid nonzero third argument.

3. **Publication before argument installation.**  
   The word-publication state first attempts a write through the null working result pointer. Valid caller output pointers do not redirect that access.

4. **Initialized, aliased word outputs.**  
   If both active word-output pointers designate the same writable word and both writes complete, the result is written first and the scaled count second. The second value remains, subject to any additional aliasing that affects its calculation.

5. **Flag construction.**  
   With a valid active flag destination, the zero construction publishes `0`; the marker-based construction publishes `1`, not `175`. Before installation, the flag-pointer test examines null.

6. **Dependency changes a captured global.**  
   If F-b09db3398a5a changes the iteration-captured global, later uses of the capture retain the earlier value, including its signed shift by `28`. A multiplier read after the call can observe a dependency modification.

7. **Dependency changes the count.**  
   A count update through its address affects later count-consuming states unless overwritten. The scaled-count argument already passed by value to F-f0c4ba604fa0 remains unchanged.

8. **Copy setup without an assigned destination candidate.**  
   If setup is reached while the retained destination candidate still has its entry value, the saved copy destination becomes zero. Reading a valid table source does not make that destination valid or imply prior allocation.

9. **Conditional dependency-result destination.**  
   If a preceding table-count state stores a return from F-d33f6d597e5d in the retained destination candidate, and no intervening state replaces it, setup uses that return as its destination. This establishes provenance, not allocation or capacity.

10. **Conditional table-source destination.**  
    If a saved table-source address is propagated into the retained destination candidate before setup, setup uses that propagated address. It separately reads the current table source, so source and destination need not derive from the same entry or table snapshot.

11. **Single copy assignment.**  
    At a valid calculated displacement, one source byte is written to the corresponding destination displacement. This proves nothing about neighboring bytes or overall overlap handling.

12. **Zero-predicate boundaries.**  
    Inputs zero, one, two, and `0xffffffff` yield predicate values one, zero, zero, and zero. The ultimate transition still depends on globals.

## Uncertainties

The following remain **documented_only**:

- Global values, initial dispatch selection, transition targets, state reachability, and termination.
- Whether argument installation precedes any particular access.
- Which assignments precede a given copy-setup entry and therefore determine destination provenance.
- Signed-overflow behavior, character signedness, alignment requirements, and address-wrap semantics.
- Table bounds, ordering, matching rules, count units, and valid fifth-argument range.
- Return meanings, failure conventions, and output behavior on failure-like paths.
- Permitted aliasing, resource ownership, lifetime, and release responsibility.
- Dependency mutations of globals and reachable memory.
- Reentrancy, thread safety, and concurrent-mutation guarantees.

These uncertainties do not erase established assignment order or capture boundaries. They prevent assigning a unique high-level API or a universal dependency-backed destination provenance.

## Evidence

The supplied source establishes the five-input interface, unsigned 32-bit return, 43-way dispatcher, zero-initialized active addresses, separate argument-installation state, context offset, table layout, dependency call shapes, byte-copy assignments, and word-publication order.

It also establishes the iteration-entry capture and its reuse across F-b09db3398a5a, distinct pre-call and post-call global reads, and the copy-setup destination’s origin in a retained working word. That retained word begins at zero and has both dependency-return and propagated-value assignment paths, including a path carrying a saved table-source address.

Allocation, retrieval, and lookup semantics are inferred intent only. Missing global values and dependency implementations leave the documented uncertainties unresolved.

No runtime tests have been performed.
