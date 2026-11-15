<!-- Trusted provenance: F-a4a67c2ba90e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a4a67c2ba90e

## Purpose

F-a4a67c2ba90e specifies a nonterminating sequence of indirect calls, byte accesses, and 32-bit memory accesses. Results from opaque dependencies determine subsequent arguments, addresses, stored values, and loop-carried state.

No application-level purpose is established. The accessed locations cannot be classified as ordinary objects, device registers, allocator metadata, or serialized data from the supplied evidence.

This contract describes the visible static representation, conditional on its recovered operations and control flow being accurate. It does not establish a safe standalone API or a complete executable reconstruction.

## Interface

### Explicit arguments and completion

The visible interface has four 32-bit argument positions.

| Position | Established use |
|---|---|
| First | No direct use in the visible body. |
| Second | No direct use in the visible body. |
| Third | Supplies shifted dependency arguments and a word-store value. Replaced by a dependency result near each iteration’s end. |
| Fourth | Passed to a dependency early in each iteration. Replaced by a scaled dependency result near that iteration’s end. |

The third and fourth positions are represented as signed 32-bit integers. Their arithmetic is described below as 32-bit bit-pattern transformations; portable signed-overflow behavior is not established.

The declared interface returns no value. The visible body has an unconditional loop and no normal return. A dependency that does not return, or an access that faults, can prevent subsequent operations.

### Additional entry requirements

Behavior also depends on:

- An inherited address-like value used to form the first indirect-call target.
- An inherited 32-bit current base.
- An inherited 32-bit carried word.
- A global 32-bit memory location read during each reached iteration.
- A secondary result channel whose provenance is unresolved.
- Dynamic call-target validity and the contents and accessibility of calculated memory locations.

These are additional prerequisites, not values supplied by the four explicit arguments. Their initialization is not established.

### Widths and address notation

The following notation expresses value relationships:

- \(W(x)\): the low 32 bits of \(x\).
- \(L_k(x)=W(x\times 2^k)\).
- \(E(x)\): the low eight bits of \((x\mathbin{\&}\texttt{0x3fffffff})\gg20\).
- \(H(x)\): the low eight bits of an unsigned 32-bit \(x\gg22\).

All byte stores retain eight bits. All word accesses are 32 bits. Wide dependency results are 64 bits, with low and high halves each 32 bits. Shifts in \(E\), \(H\), and the final word-read transformation are unsigned.

Calculated 32-bit arithmetic is described modulo \(2^{32}\). Whether that model exactly matches the executable’s address formation is an environmental unknown.

A **byte-selected store** comprises an unsigned byte read from a specified selector location, followed by a byte store at that byte’s numeric value plus 17. The destination therefore lies between 17 and 272 inclusive. This is not a pointer-sized field read.

A **direct store** uses its calculated address without that selector indirection. No structure layout beyond the stated widths and offsets is established.

## Behavior

### Execution and ordering

The following ledger describes one iteration. Initially, the current base and carried word are the inherited values; later iterations use the explicitly updated state.

Every listed direct access retains its position relative to the intervening calls. Dependencies may have additional effects, so this ledger is not a complete memory trace. In particular:

- Repeated selector expressions denote separate reads unless explicitly described as cached.
- Repeated stores remain distinct events, including stores to coincident addresses.
- A global snapshot is not interchangeable with a later global read.
- Explicit state assignments must not be moved across calls.

“Further calls” below denotes intervening opaque call occurrences in the representation, not permission to omit them. Distinct result labels identify particular occurrences; they do not imply distinct dependency identities.

### Opening operations

1. Three indirect calls occur. The third supplies a 32-bit opening address result. The carried word is stored at:
   \[
   W(4\,\text{opening address result}+20).
   \]
   A following dependency receives \(L_2(\text{opening address result})\).

2. Four further calls produce an unsigned 32-bit **opening value**. A selector byte is read at \(W(\text{current base}+1)\). The low byte of \(L_2(\text{opening value})\) is stored at the selected address. A dependency then receives \(L_2(\text{opening value})\).

3. A subsequent call supplies a result that is passed to another dependency as its first explicit argument; its second is \(L_{30}(\text{opening value})\). Four further calls produce a fresh result. The selector at \(W(\text{current base}+1)\) is reread and cached. The function stores \(E(\text{opening value})\) at this selector’s value plus 17, then calls a dependency with the fresh result shifted left two bits.

4. Four intervening calls occur before the function stores \(L_2(\text{current third argument})\) as a word at **cached selector plus 20**. There is no new selector read for this word store.

5. The global 32-bit location is read and retained as a snapshot. The next dependency receives the current fourth argument. Thus the direct word store precedes the global read, which precedes this call.

### First wide-result segment

6. Four further calls produce a 64-bit result. Define the **first scaled result** as \(L_2\) of its low half. A fresh selector is read at \(W(\text{current base}+1)\), and \(H(\text{global snapshot})\) is stored at the selected byte address. A dependency then receives the wide result’s high half.

7. A subsequent call supplies a result forwarded to another dependency together with \(L_{28}(\text{first scaled result})\). Four further calls produce a fresh result. A fresh selector is read at \(W(\text{current base}+1)\); zero is stored at the selected byte address. A dependency receives \(L_2\) of the fresh result.

8. Four further calls produce a result whose left-two-bit transformation is passed twice to one dependency. Four more calls produce a result whose left-two-bit transformation is the **first payload**. That payload is stored as a word at:
   \[
   W(4096\,\text{first scaled result}+20).
   \]
   A dependency receives \(L_{14}(\text{first scaled result})\).

9. Four further calls produce a new address result. The first payload is stored at \(W(4\,\text{address result}+20)\). A dependency receives \(L_2(\text{address result})\).

### Second wide-result segment

10. Four further calls produce another 64-bit result. Define the **second scaled result** as \(L_2\) of its low half. A selector byte is read at:
    \[
    W(16384\,\text{first scaled result}+1).
    \]
    The function stores \(E(\text{first payload})\) at the selected address, then passes the wide result’s high half to a dependency.

11. A subsequent result is forwarded as one explicit argument. The next dependency produces a 64-bit result, whose low and high halves are forwarded, in that order, to another dependency. Two more calls produce a fresh 32-bit result. A selector is read at:
    \[
    W(2^{28}\,\text{second scaled result}+1).
    \]
    Zero is stored at its selected address. A dependency receives the fresh result shifted left two bits.

12. Four further calls produce the **middle result**. A selector is read at:
    \[
    W(4\,\text{second scaled result}+1).
    \]
    Zero is stored at its selected address. A dependency receives \(L_{10}(\text{second scaled result})\).

13. A subsequent result is forwarded together with \(L_{28}(\text{current third argument})\). Four further calls produce a fresh result. The selector location from operation 12 is reread, and zero is stored at the newly selected address. A dependency receives the fresh result shifted left two bits.

14. Four intervening calls occur before the **second payload**, \(L_{14}(\text{middle result})\), is stored as a word at:
    \[
    W(4096\,L_2(\text{middle result})+20).
    \]
    A dependency receives \(L_{14}(L_2(\text{middle result}))\).

15. After another call, a dependency receives two explicit arguments: \(L_{28}(\text{second payload})\), then the unresolved secondary result-channel value. Four further calls produce an address result. The second payload is stored at \(W(4\,\text{address result}+20)\), followed by a dependency receiving the scaled address result.

    Under the stated 32-bit model, \(L_{28}(\text{second payload})\) is always zero because the second payload already has its lowest 14 bits clear. This does not resolve the second argument’s provenance or value.

### Third payload segment

16. Four further calls produce the **third address result**. The selector location from operation 12 is reread. The function stores \(E(\text{second payload})\) at the selected address, then calls a dependency with \(L_{26}(\text{middle result})\).

17. A subsequent result is forwarded together with:
    \[
    L_{28}(L_2(\text{third address result})).
    \]
    The next dependency produces a 64-bit result; its low and high halves are passed, in that order, to another dependency. Two more calls produce a fresh result. The selector location from operation 12 is reread, zero is stored at the selected address, and the fresh result shifted left two bits is passed onward.

18. Four further calls produce a result whose left-two-bit transformation becomes the **third payload**. It is stored at:
    \[
    W(4096\,L_2(\text{third address result})+20).
    \]
    The next dependency receives four explicit arguments, in order:

    1. \(W(1024\,L_2(\text{third address result}))\).
    2. The same value.
    3. \(L_{14}(\text{third address result})\).
    4. The third payload.

19. Four further calls produce an address result. The third payload is stored at \(W(4\,\text{address result}+20)\). A dependency receives the scaled address result.

### Closing stores and state transitions

20. Four further calls produce a result. Its left-two-bit transformation is stored at \(W(16\,\text{result}+20)\). A byte-valued working state is then set to zero, before a dependency is invoked with three copies of the stored value. That zero state supplies the later direct byte stores.

21. Four further calls produce an address result. The word value from operation 20 is stored at \(W(4\,\text{address result}+20)\). A dependency receives the scaled address result.

22. Four further calls produce another result. The following order is explicit and must be preserved:

    1. Form its left-two-bit value.
    2. Store that value as a word at \(W(16\,\text{result}+20)\).
    3. Replace the current base with \(W(2^{30}\,\text{result})\).
    4. Invoke a dependency with three copies of the stored value.

    **The base replacement precedes the three-argument invocation.** It is not an update performed after that dependency returns. Whether the dependency can observe or alter the corresponding ambient state is unresolved; no preservation or non-observability assumption permits reordering it.

23. Four further calls produce a new address result. The value stored in operation 22 is stored again at \(W(4\,\text{address result}+20)\). A dependency receives the scaled address result. This unscaled address result is also retained for the closing word value in operation 26.

24. Four further calls produce a value retained for a later word read. Two direct byte stores then occur, in order:

    - Zero at \(W(\text{current base}+17)\).
    - Zero at \(W(\text{current base}+1)\).

    Neither uses selector indirection. A dependency receives the retained value shifted left two bits.

25. A subsequent result is forwarded to another dependency. Four further calls produce a fresh result. The two direct zero stores from operation 24 are repeated in the same order, followed by a dependency receiving the fresh result shifted left two bits.

26. Four further calls produce the **closing address result**. The fourth argument is replaced by \(L_2(\text{closing address result})\). A word is then stored at:
    \[
    W(16\,\text{closing address result}+20).
    \]
    Its value is \(L_{31}\) of the address result retained from operation 23.

27. A 32-bit word is read at:
    \[
    ((\text{retained value from operation 24}
       \mathbin{\&}\texttt{0x3fffffff})\gg20)+20.
    \]
    This address calculation retains all ten bits remaining after the shift. Its range is 20 through 1043 inclusive; it is not an eight-bit selector calculation.

28. A dependency receives the updated fourth argument. The following dependency supplies the new third argument. The carried word is then set to zero. Another dependency receives:
    \[
    L_{28}(\text{word read in operation 27}\gg22).
    \]
    The address-like state used for the next iteration’s first call target is assigned after this invocation and before one final indirect call. The loop then repeats.

The next iteration uses the represented updated base, third argument, fourth argument, zero carried word, and reassigned first-target state. These explicit transitions do not establish general dependency clobber rules.

## Side effects

Direct effects include unsigned byte reads, byte stores, 32-bit stores, one global 32-bit read per reached iteration, one calculated-address word read near the tail, and changes to loop-carried working state. Numerous opaque calls occur between these effects.

There is no established direct allocation, release, ownership transfer, synchronization operation, error report, or successful-completion notification. Numeric dependency arguments are not necessarily pointers. Neither passing a value nor storing it establishes ownership or lifetime rules.

No visible bounds checks, null checks, alignment checks, call-result validation, or recovery branches protect the accesses or invocations. Valid execution therefore requires the reached addresses and call targets to be usable in the actual environment. Fault form, unwinding, and visibility of earlier effects are unresolved.

Dependencies must not be assumed to preserve global memory, selector contents, accessed words, or undocumented ambient state.

## Dependencies

**F-opaque-unresolved** denotes the set of unresolved indirect dependency occurrences. It does not assert that all occurrences invoke one function.

Established properties are:

- Targets are formed from inherited or embedded address-like state plus offsets.
- Calls have zero through four explicit argument positions.
- Results are consumed as 32-bit values or as 64-bit values split into two halves.
- One invocation consumes an unresolved secondary result-channel value.
- Many calls have no explicit arguments or no visibly consumed result.

An invocation with no explicit arguments may still depend on registers or other ambient state. Exact target identities, calling convention, preservation rules, ownership behavior, memory effects, and nonreturning behavior remain documented_only unknowns.

## Acceptance examples

These are conditional static consequences, not executed tests.

1. **Selector width.** A selector byte of 255 selects byte-store address 272. It does not request a word-sized pointer fetch.

2. **Opening truncation.** An opening value of `0xffffffff` produces byte `0xfc` in operation 2 and byte `0xff` in operation 3. The masked right-shift expression in operation 3 is `0x3ff` before byte truncation.

3. **Cached selector.** If operation 3 reads selector 9, operation 4 stores its word at address 29. Changing the selector’s original memory location between these operations does not change the explicitly cached value.

4. **Fresh reads.** If intervening dependencies change the selector location between operations 12 and 13, their zero stores can target different addresses. The repeated location expression does not authorize caching.

5. **Global snapshot.** A dependency called after operation 5 may modify the global location. Operation 6 nevertheless uses the snapshot already read, rather than a second global read.

6. **High-shift narrowing.** A third-argument bit pattern of `0x00000010` gives zero for its left-28-bit argument under the 32-bit model.

7. **Base-update boundary.** If operation 22’s result is 3, the word value is 12 and its destination is 68. After that store, the current base is assigned `0xc0000000`; only then is the dependency invoked with arguments 12, 12, 12. A trace placing the assignment after the invocation violates the represented order, irrespective of whether dependency visibility of that state is known.

8. **Closing stored value.** Operation 26 stores zero if the operation-23 address result is even, and `0x80000000` if it is odd.

9. **Tail-read boundary.** A retained value of `0x3fffffff` produces word-read address 1043 in operation 27. Truncating the shifted intermediate to eight bits would incorrectly produce address 275.

10. **No normal completion.** If all calls return and all accesses succeed, reaching the loop tail starts another iteration instead of returning.

## Uncertainties

### Established static evidence

The supplied representation supports the stated access widths, explicit data relationships, direct-access ordering, unconditional loop, and inherited-state requirements. In particular, operation 22 places the current-base assignment between the word store and the three-argument dependency call.

### Inferred intent

Repeated scaling, selector accesses, and paired payload stores suggest structured state manipulation. No particular data structure, protocol, or application purpose is established or required by this contract.

### documented_only unknowns

- Whether the recovered function boundary and control flow match executable behavior.
- The significance of the reported removal of an unreachable block.
- The consequences of overlapping global symbols.
- Dependency identities and target aliasing.
- Calling convention, ambient-state visibility, and clobber rules.
- The provenance of the secondary result channel.
- Executable address arithmetic, endianness, alignment policy, memory mapping, and fault semantics.
- Ownership, lifetime, and synchronization requirements.
- Whether low numeric addresses are usable memory or special hardware locations.
- Whether dependencies terminate, unwind, or redirect execution.
- Whether the apparently unused first two arguments have ABI-level significance.

The explicit base-assignment order is established; its visibility to a dependency is not. These are separate claims.

## Evidence

This contract is based only on the supplied static representation: represented access types, arithmetic expressions, indirect invocations, state assignments, loop structure, and accompanying warnings.

The closing-state ordering follows the represented sequence of a word store, current-base assignment, and three-argument invocation. No assumption about dependency behavior is needed to establish that sequence.

No dependency inspection, implementation access beyond the embedded representation, runtime execution, smoke run, or tests have been performed. Acceptance examples are arithmetic and ordering consequences conditional on the representation’s accuracy.
