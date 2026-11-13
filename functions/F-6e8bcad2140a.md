<!-- Trusted provenance: F-6e8bcad2140a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-6e8bcad2140a

## Purpose

F-6e8bcad2140a conditionally modifies working state and transfers halfwords, then enters an unconditional cycle of two calls and a software interrupt. No application-level purpose is established.

The declared arguments do not fully determine execution. The routine also consumes ambient addresses, arithmetic state, and eight working words without visible initialization. A zero first argument selects additional work; it is not treated as a rejected null input.

Under the documented 32-bit word-arithmetic model, a contradiction in the second equality group prevents execution of the subsequent subtraction group and all later explicit memory operations before the terminal cycle. Those operations are syntactically present but unreachable from the supplied entry.

This contract distinguishes:

- **Established static behavior:** control flow, data dependencies, access widths, and ordering supported by the supplied representation.
- **Inferred intent:** interpretation not established by those operations.
- **documented_only:** unresolved environmental or instruction-level behavior.

No runtime tests have been performed.

## Interface

### Declared arguments and result

| Argument | Represented width | Established use |
|---|---:|---|
| First | 32-bit address associated with 16-bit accesses | Entry zero test and conditional address arithmetic |
| Second | 32 bits | No explicit use |
| Third | 32 bits | No explicit use |
| Fourth | Unsigned 32 bits | Arithmetic, equality gates, and a conditional increment by two |
| Fifth | 16 bits | Its address participates in arithmetic; its incoming value is not explicitly read |

There is no declared result and no visible normal-return path. A dependency returning does not make the enclosing routine return.

The second and third arguments have no explicit uses in the represented body. This does not establish that their physical argument locations are irrelevant to dependencies with unknown calling conventions.

### Ambient inputs

The following are specification labels, not additional declared parameters or source identifiers:

| Label | Meaning |
|---|---|
| Link address | Ambient 32-bit address used as the initial terminal cursor and a possible halfword source |
| Entry carry | One-bit arithmetic input |
| Auxiliary word | Ambient 32-bit arithmetic input |
| Stack address | Represented 32-bit address of the fifth argument |
| Slots 1–8 | Eight ambient 32-bit working values; slots 2 and 7 also have address interpretations |

A conventional invocation supplying only the five declared arguments is insufficient to determine the routine’s behavior.

### Arithmetic and memory model

Unless qualified otherwise, arithmetic relationships below use 32-bit words reduced modulo \(2^{32}\). Comparisons described as unsigned use that interpretation. This documents machine-word relationships; it does not assert portable semantics for every signed expression in the supplied representation.

Memory accesses explicitly represented in the body include:

- Unsigned 16-bit reads and writes.
- Signed 16-bit reads extended to 32 bits, exclusively in unreachable operations.
- 32-bit address loads and word stores, exclusively in unreachable operations.
- A byte-sized conversion of unresolved signedness, exclusively in an unreachable address calculation.

Arithmetic on a halfword-address interpretation scales an element displacement by two bytes. Explicit byte offsets do not receive that scaling.

Byte order, alignment restrictions, address-space mapping, and pointer provenance are not supplied. Each executed access requires an address supporting its access width. No validation establishes that prerequisite.

## Behavior

### Entry selection

With a nonzero first argument, the explicit conditional memory operations are skipped. The initial terminal cursor is the link address.

With a zero first argument, the routine computes an adjusted first address:

\[
A =
\bigl(\text{link address}+\text{entry carry}
+(\text{stack address}\gg1)\bigr)
(\text{slot 2}+1)
\pmod {2^{32}}.
\]

The stack-address shift is logical. This is numeric address arithmetic, not a validated allocation-relative calculation.

Before the initial transfer, the routine updates:

- Slot 1 by adding the fourth argument.
- Slot 3 by adding slot 2.
- Slot 5 by adding slot 4.

These updates occur in the listed order.

### Initial halfword transfer and intervening state update

The initial destination is the original slot-7 address plus slot 6, with byte-address arithmetic.

The represented ordering is significant:

1. The destination address is calculated.
2. An unsigned halfword is read from the original slot-7 address and retained.
3. Slot 7 becomes the destination address minus slot 8.
4. The retained halfword is written to the destination.

The write therefore uses the saved source value, not a new read through updated slot 7. Slot 7 is updated **before** the destination write, not after the transfer as a whole.

If the source read cannot complete, the slot-7 update and destination write are not reached. If the destination write cannot complete, the represented execution has already performed the source read and slot-7 update. How a concrete fault exposes or preserves working state remains **documented_only**.

After a successful destination write, carry is cleared and the first gate value becomes the link address.

An overlapping transfer remains a represented read followed by a write, with the slot-7 update between them. The contract does not permit treating it as having no effect merely because both addresses are equal.

### First zero-gated progression

Each gate is enabled only by a zero result from its predecessor. A nonzero result stops this progression but does not constitute a return or reported error; execution continues toward the terminal cycle.

The first group requires, in order:

1. Link address equals zero.
2. Adjusted first address \(A\) equals zero.
3. Fourth argument equals zero.
4. Updated slot 1 equals slot 2.
5. Updated slot 3 equals slot 4.
6. Updated slot 5 equals slot 6.
7. Updated slot 7 equals slot 8.

The represented XOR gates use a shift count derived from the preceding gate value. Whenever such a gate is enabled here, that preceding value is zero. Consequently, the relevant shift count is zero, the gates reduce to the equalities above, and they preserve the cleared carry.

After these conditions, an addition combines the auxiliary word, the fourth argument logically shifted right by one, and carry. Progression requires a zero 32-bit result; unsigned carry-out is retained.

On this reachable path, the fourth argument and incoming carry are already zero. The condition therefore requires the auxiliary word to be zero and produces no carry-out.

The next gate adds the stack address, twice slot 6, and the incoming carry. Its wrapped result must be zero. The represented outgoing carry combines carry-out from the stack address plus twice slot 6 with carry-out from adding the incoming carry to that wrapped sum.

These arithmetic conditions do not waive the earlier halfword-access requirements.

### Stack-derived state and halfword rewrite

If the stack-address gate succeeds, the routine establishes a stack-related state:

\[
S = 64\times\text{stack address}\pmod {2^{32}}.
\]

It also calculates an advanced link-derived address using the same byte displacement:

\[
L'=\text{link address}+64\times\text{stack address}
\pmod {2^{32}}.
\]

The rewrite uses a halfword at the previous link address in two expressions:

- Its unsigned value supplies the numeric destination address.
- Its value plus fifteen supplies the stored halfword, truncated to 16 bits.

The representation contains two source-value evaluations. Their relative evaluation order and correspondence to concrete memory transactions are not established. They must not be silently treated as one stable observation when describing device-backed or otherwise changing memory.

Reaching this rewrite requires the original link address to be zero. The source expressions therefore refer to address zero. An unmapped zero address can prevent continuation; no error-return or recovery path is shown.

After the rewrite completes, the fourth argument increases by two, carry is cleared, and the advanced address becomes the current link-derived address. The adjusted first address \(A\) is unchanged.

The assignment to stack-related state precedes the rewrite. Its concrete machine meaning and effect on execution are **documented_only**; it is not evidence of stack allocation, a valid stack switch, or ownership transfer.

### Established termination of the gated progression

Further progression first requires \(L'=0\).

If \(L'\ne0\), the second equality group is skipped. If \(L'=0\), its first gate produces \(A\), which is zero because reaching the rewrite already required \(A=0\).

The next gate compares the fourth argument with \(A\) using an XOR with a zero shift count. Reaching the rewrite required the fourth argument to be zero, and the rewrite then incremented it by two. Consequently this gate produces:

\[
2\mathbin{\mathrm{XOR}}0=2.
\]

That result is nonzero. Thus the remaining equalities in the second group cannot execute, and the group cannot enable its successor.

This is a static contradiction under the stated word-arithmetic model, not an unresolved question about ABI or large-shift semantics. The subtraction group, conditional pointer load, paired stores, and final memory-update group are unreachable from the supplied entry.

### Syntactically present unreachable operations

The following describes dead operations to delimit the supplied representation. None is an executable requirement for an invocation entering through the documented entry.

The dead subtraction group first forms the adjusted first address plus carry plus `0x085740e8`, then shifts that word left by its own low byte. Counts exceeding 31 have unresolved instruction-level semantics, but this uncertainty does not affect reachable behavior.

Three subtraction gates then use:

\[
r=x-y-(1-c)\pmod {2^{32}},
\]

with separately represented outgoing flag:

\[
(y<x)\ \lor\
\bigl((x-y)\bmod2^{32}<c\bigr).
\]

The operand pairs are:

| Gate | \(x\) | \(y\) | Additional update |
|---|---|---|---|
| First | Slot 1 | Shifted word | None |
| Second | Slot 2 | Slot 7 | None |
| Third | Slot 4 | Slot 1 logically shifted right by one | Slot 3 increases by `0x24` |

Each successor requires a zero predecessor result and receives its outgoing flag. The flag expression must not be replaced by an assumed conventional borrow interpretation.

After those gates, the dead pointer-load block increases slot 6 by eight and slot 8 by ten. It derives a candidate cursor \(P\), thirteen bytes beyond the current link-derived address, and loads a 32-bit address through a location formed by converting a halfword at the old link-derived address to a byte-sized value and adding four. That byte conversion’s signedness is unresolved.

The subsequent gate uses the complemented incoming flag \(n\) in this expression:

\[
(S\gg1)-(P+n)+n.
\]

The two flag contributions cancel modulo \(2^{32}\). The equality required by this syntactic gate is therefore:

\[
\texttt{0x042ba0a0}=(S\gg1)-P\pmod {2^{32}}.
\]

There is no remaining subtraction of twice the complemented flag.

If that dead gate were considered in isolation, its successor would read a signed halfword at \(A\), extend it to 32 bits, derive an address one **byte** beyond \(A\), and select a cursor equal to \(P\) minus the fourth argument. It would store the extended value at \(P\), followed by the previously loaded address at \(P+4\). The read precedes both stores; the stores occupy adjacent four-byte fields and occur in that order.

A final dead equality sequence requires the extended value and loaded address to be zero, the fourth argument to equal the one-byte-advanced address, and the four slot pairs to match.

Its final memory group captures a source address thirteen bytes beyond the selected cursor, advances the cursor by twenty-seven bytes, and then reads a signed halfword through the captured address. After extension, that value determines a destination by subtracting `0x08574200`. The group stores the extended value there and the previously loaded address four bytes later, in that order.

### Terminal cycle

Every visible path that completes its selected prefix reaches an unconditional cycle.

For reachable execution, the initial terminal cursor is:

- The original link address if the stack-derived rewrite did not complete.
- The advanced link-derived address \(L'\) if that rewrite completed.

Failed arithmetic gates do not replace this cursor with their scratch results.

Each iteration performs, in order:

1. Invocation of F-unknown-01 at the cursor plus **3,766 bytes**.
2. Invocation of F-unknown-02 at numeric target `0x0857a94b`.
3. Invocation of F-unknown-03, a software interrupt with immediate `0x5b`.
4. Assignment of `0x083d4cc2` as the next cursor.

F-unknown-01 receives three explicit 32-bit words: zero, zero, and `0xffffffa5`. The last word is the signed −91 bit pattern.

No explicit arguments are represented for F-unknown-02. This is not a guarantee that it ignores ambient state.

If both calls and the interrupt return and execution reaches the cursor assignment, the next cursor-derived call target is `0x083d5b78`. A fault, dependency nontermination, or nonlocal control transfer can prevent that next invocation. No normal exit from the cycle is represented.

## Side effects

Reachable explicit memory effects before the terminal cycle are limited to:

- The initial halfword read and write selected by a zero first argument.
- The deeply gated halfword-source evaluations and halfword rewrite.

No explicit 32-bit memory load or store in the conditional body is reachable from the supplied entry. Such operations occur only in the dead groups described above.

Working-state effects include arithmetic slot updates, the intervening slot-7 update, a conditional stack-related state assignment, and the conditional increment of the fourth argument.

No explicit allocation, release, ownership transfer, synchronization, bounds enforcement, error result, or recovery action is established. The routine does not establish ownership of any numerically addressed memory.

Terminal dependencies may read or modify memory, alter global state, release resources, or fail to return. A nonzero first argument therefore promises only that the explicit prefix memory accesses are skipped, not that execution leaves memory unchanged.

## Dependencies

| Opaque ID | Invocation boundary | Established information |
|---|---|---|
| F-unknown-01 | Cursor-derived indirect call | Target is cursor plus 3,766 bytes; three explicit 32-bit argument words |
| F-unknown-02 | Fixed numeric call target | Target is `0x0857a94b`; no explicit arguments represented |
| F-unknown-03 | Software-interrupt boundary | Immediate is `0x5b`; handler contract unavailable |

These IDs designate unresolved dependencies, not established identities or semantic equivalences.

No dependency is assumed to preserve globals, ambient registers, referenced allocations, or calling-state invariants. Ownership and return behavior remain unknown.

## Acceptance examples

These are conditional static acceptance cases, not executed tests.

1. **Nonzero first argument.**  
   If execution can reach the call boundary, the initial F-unknown-01 target is the link address plus 3,766 bytes. No explicit prefix memory access occurs. Its explicit argument words are zero, zero, and `0xffffffa5`.

2. **Zero first argument and nonzero link address.**  
   Assuming the initial accesses succeed, the routine performs the initial halfword transfer with the intervening slot-7 update. The nonzero link address prevents deeper progression. The initial terminal cursor remains the link address.

3. **Destination-write failure.**  
   If the initial source read succeeds but the destination write fails, the represented slot-7 update has already occurred. A description placing that update after the failed write is inconsistent with the static ordering. Concrete fault-state visibility is unresolved.

4. **Self-copy.**  
   With slot 6 equal to zero, the initial source and destination coincide. The represented order remains source read, slot-7 update, destination write.

5. **Wrapped initial destination.**  
   If original slot 7 plus slot 6 overflows 32 bits, the destination is the wrapped address under this model. Mapping and write permission remain environmental prerequisites.

6. **Rewrite followed by nonzero advanced address.**  
   If all rewrite prerequisites hold and its accesses complete, but \(L'\ne0\), the second equality group is skipped. The terminal cursor is \(L'\).

7. **Rewrite followed by zero advanced address.**  
   If the rewrite completes with \(L'=0\), the second equality group first produces zero from \(A\), then produces two from the incremented fourth argument XOR \(A\). No subtraction-group operation or subsequent explicit 32-bit access can execute.

8. **Returning terminal dependencies.**  
   If the first iteration reaches its cursor assignment after both calls and the interrupt, the next cursor-derived target is `0x083d5b78`, independent of the entry link address.

## Uncertainties

The following remain **documented_only**:

- Application purpose and valid invocation context.
- ABI and complete ambient-state acquisition rules.
- Concrete meaning of the stack-related state assignment.
- Whether the environment supports the reachable numerical addresses, particularly address zero.
- Exact transaction ordering for the rewrite’s two source-value evaluations.
- Byte order, alignment requirements, memory permissions, and device behavior.
- Fault delivery and visibility of prior working-state updates.
- Dependency effects, ownership behavior, and return behavior.
- Large-shift semantics and byte-conversion signedness within the unreachable groups.

The subtraction group and its successors are not potentially reachable behavior awaiting clarification of these uncertainties. Their unreachability follows from the zero-to-two contradiction before them.

Dependence on ambient machine state suggests a machine-context-specific operation rather than a self-contained five-argument API. That is inferred intent, not an established purpose.

## Evidence

This contract derives solely from the supplied static representation: declared widths, address scaling, explicit access widths, data dependencies, assignment ordering, zero-gated control flow, numeric call targets, interrupt immediate, and unconditional terminal back edge.

The reachability conclusion follows from three established facts: entry to the rewrite requires \(A=0\) and fourth argument zero; the rewrite preserves \(A\) and increments the fourth argument to two; the second equality group then produces a nonzero XOR result.

The corrected pointer-load gate follows from cancellation of its complemented-flag terms modulo \(2^{32}\). The initial-transfer ordering follows from the slot-7 assignment lying between the source read and destination write.

No dependency bodies, instruction listing, runtime traces, or execution results were supplied. No runtime tests have been performed.
