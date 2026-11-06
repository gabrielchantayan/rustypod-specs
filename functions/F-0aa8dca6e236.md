<!-- Trusted provenance: F-0aa8dca6e236; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0aa8dca6e236

## Purpose

F-0aa8dca6e236 conditionally repeats an indirect call and a working-state update, then performs four further indirect calls, reads data, and issues four memory writes. The visible path ends at a bad-instruction termination marker rather than a normal return.

No application-level purpose is established. The supplied representation does not establish object initialization, allocation, collection processing, or a valid standalone entry point.

This contract describes static evidence from the supplied representation. That representation explicitly warns about bad instruction data and truncated control flow. Actual instruction validity and execution beyond the represented boundary remain unresolved.

## Interface

### Explicit inputs

The displayed interface contains three parameters:

| Input | Represented width | Visible use |
|---|---:|---|
| First parameter | 32 bits | No explicit use |
| Second parameter | 32 bits | No explicit use |
| Third parameter | Signed 32-bit integer | Initial loop-control value, compared with −64 |

The first two parameters have no explicit use in the visible body. This does not establish that their incoming machine locations are irrelevant to indirect callees. No complete calling convention or hidden-argument contract is supplied.

### Additional entry state

Three descriptive labels identify incoming state used by the representation:

- **Accumulator:** a 32-bit working integer, later also used to derive read addresses.
- **Operand:** a 32-bit working integer used in loop arguments and updates.
- **Linkage value:** incoming state used to derive indirect call targets.

These labels are not established additional formal parameters. Their initialization and calling-convention roles are unknown.

The routine also depends on executable indirect targets, readable and writable memory at derived addresses, a global reference designated **G**, and results or effects of unresolved dependencies.

### Outputs and completion

No ordinary return value is established. The displayed interface indicates no returned value, and the visible body contains no normal return.

After the fourth final store, the representation terminates at a bad-instruction marker. Whether actual execution faults, traps, transfers elsewhere, or differs because of incorrectly represented instructions is **documented_only**.

### Widths and layout

The representation uses:

- 32-bit scalar inputs and working integers.
- Signed comparison against −64.
- A 32-bit unsigned mask for a one-bit adjustment.
- Three explicit 32-bit data loads before the final call.
- One 16-bit store followed by three 32-bit stores.
- A global reference supplying the second store.

Offsets below are byte offsets. Endianness, actual address-space width, alignment requirements, and arithmetic overflow behavior are not independently established.

The three final 32-bit destinations begin at offsets `0x97`, `0x9B`, and `0x9F` from the same captured value. They therefore occupy consecutive four-byte regions under an ordinary byte-addressed model. No larger object layout or ownership relationship is established.

## Behavior

### Conditional call-and-update phase

The initial loop guard tests whether the third parameter is strictly greater than −64.

- Values of −64 or less skip the loop.
- Values of −63 or greater enter the loop.

Each entered iteration first invokes **F-unknown-01**.

Its target is formed from the current linkage value plus `0xEB2`. After forming that target and before making the call, the represented linkage state is assigned `0x08570171`.

The call has four represented argument expressions:

| Position | Value |
|---:|---|
| 1 | Four times the operand, plus `0x53` |
| 2 | The operand |
| 3 | Twice the operand, minus `0x67` |
| 4 | The operand shifted left by three bit positions |

No return value from this call is explicitly consumed. The argument expressions contain no explicit memory loads; their relative evaluation order is not established.

The first iteration derives its target from the incoming linkage value. In the represented state flow, subsequent iterations derive their targets from `0x08570171`. This distinction does not establish a complete machine-level register-preservation contract.

After the call returns, the routine derives a one-bit adjustment by doubling the operand and testing the result with the 32-bit mask `0x80000000`. The adjustment is 1 when the tested bit is set and 0 otherwise.

The next working value is the accumulator plus `0x35`, plus that adjustment, plus the operand. This value replaces all three of:

- The loop-control value.
- The accumulator.
- The operand.

The signed loop guard is then evaluated again.

Under a 32-bit modular arithmetic interpretation, the update is the low 32 bits of accumulator + operand + 53 + adjustment. This interpretation is conditional: the representation does not independently settle signed multiplication overflow, signed left-shift behavior, or all conversions at arithmetic boundaries.

The adjustment tests the high bit of the doubled operand. It is not equivalent to testing whether the original operand is negative.

After a completed update, accumulator and operand are equal in the represented local state. No iteration bound or general termination guarantee is established.

### Post-loop calls

When the guard is false, three indirect calls occur in this order:

| Dependency | Target derivation | Explicit result use |
|---|---|---|
| F-unknown-02 | Current linkage value plus `0xFB8` | None |
| F-unknown-03 | Fixed anchor `0x08570199` plus `0x7FA` | None |
| F-unknown-04 | Fixed anchor `0x0857019B` plus `0xFFA` | Its 32-bit result is retained |

If the loop was skipped, F-unknown-02 uses the incoming linkage value. If an iteration completed, the represented state flow supplies the linkage value assigned in the loop.

No explicit arguments are shown for these three calls. That does not establish zero-argument machine-level interfaces. Registers, stack contents, shared memory, and other hidden inputs may matter.

### Reads and final call

After F-unknown-04 returns, the routine reads a 32-bit value from accumulator plus `0x45`. This contract calls the captured value **P**.

The routine then calls **F-unknown-05**, using the target formed from fixed anchor `0x0857019D` plus `0xF9A`. Its three represented arguments are:

1. The retained result from F-unknown-04.
2. A 32-bit value read from accumulator plus `0x35`.
3. A 32-bit value read from accumulator plus `0x39`.

The read producing P precedes both the final call and its argument loads. Both argument loads precede the call, but their order relative to each other is unspecified.

All three loads occur after the preceding post-loop calls. They are not entry-time snapshots. Earlier dependencies may modify the relevant memory before these reads, or may invalidate it.

The final call’s 32-bit return value is designated **R**. Neither P nor R is established as a pointer, allocation, handle, or index with any particular validity guarantee.

The final stores use the captured P. No reload from accumulator plus `0x45` is shown after F-unknown-05, even if that dependency modifies the original source memory.

### Final stores and intervening global reference

After F-unknown-05 returns, the following sequence is represented:

| Order | Operation | Width and value |
|---:|---|---|
| 1 | Store at four times R minus `0x4E` | 16 bits: P converted to 16 bits, plus `0x94`, with final 16-bit truncation |
| 2 | Evaluate G as the source for the next assignment | Underlying access semantics unresolved |
| 3 | Store at P plus `0x97` | 32 bits: the value supplied by G |
| 4 | Store at P plus `0x9B` | 32 bits: P plus `0xE6` |
| 5 | Store at P plus `0x9F` | 32 bits: twice the entire quantity P plus `0x97` |

The first stored bit pattern is the low 16 bits of P + `0x94`, under the represented truncation model.

The last stored expression doubles the entire sum P + `0x97`. It is not twice P followed by addition of `0x97`.

**G is evaluated after completion of the first, 16-bit store and before the second store.** It is not merely a value obtained sometime after the calls.

If G denotes memory, its read occurs at that point. Consequently:

- Earlier dependencies may affect the value read.
- The first store may itself overlap and modify G.
- The second store must then use the value resulting from that preceding modification, subject to the actual memory-access semantics.
- Moving the global reference before the first store is not equivalent without an established independence guarantee.

The first destination may also overlap any later destination. No disjointness checks are visible.

### Terminal action

After the fourth store, the representation reaches the bad-instruction termination marker. No subsequent behavior or normal completion is established.

## Side effects

### Direct effects

The visible body explicitly represents:

- Three 32-bit data loads used for P and the final call’s arguments.
- Evaluation of G between the first and second final stores.
- One 16-bit store.
- Three 32-bit stores.
- Changes to represented working state and linkage state.
- Indirect calls whose additional effects are unknown.

If G is a memory-backed 32-bit datum, its evaluation adds a 32-bit global read. Its storage class, volatility, and precise access semantics are not established.

### Observable ordering

The represented sequence is:

1. Zero or more loop calls, each followed by its working-state update.
2. F-unknown-02.
3. F-unknown-03.
4. F-unknown-04.
5. The read producing P.
6. The two final-call argument reads, in unspecified relative order.
7. F-unknown-05.
8. The 16-bit store.
9. Evaluation of G, including its read if memory-backed.
10. The 32-bit store supplied by G.
11. The 32-bit store of P + `0xE6`.
12. The 32-bit store of twice P + `0x97`.
13. The terminal marker.

Dependencies must not be assumed to preserve globals, memory contents, or memory lifetime. Captured values and later reads must remain distinct: P is captured before the final call, whereas G is evaluated after that call and after the first store.

### Aliasing and partial completion

Under ordinary sequential memory semantics, later stores determine the final bytes where destinations overlap. The G case additionally makes aliasing affect a later store’s *value*, not just the final contents of overlapping destinations.

If the 16-bit destination overlaps memory backing G, the intervening global read observes the updated bytes. If the destination supplied by G overlaps G itself, the source evaluation still precedes that destination write.

No transaction, rollback, recovery handler, or cleanup path is shown. If a dependency does not return, later operations are not reached. If an access faults, later operations are not required to occur, and earlier completed effects are not represented as undone.

### Ownership and preconditions

No allocation or release is explicitly visible. Ownership of any accessed storage is not established.

Reaching each operation successfully requires its indirect target or memory access to be valid at that time. In particular:

- Call targets must be executable under the actual calling convention.
- The three source locations must remain readable until their respective loads.
- Each destination must be writable when its store occurs.
- G must be evaluable when reached.
- Any required alignment must hold, or the execution environment must support the access.

The routine shows no checks establishing these conditions. Several offsets can produce unaligned accesses.

## Dependencies

These opaque IDs label unresolved call roles, not verified distinct function identities.

| ID | Role | Known result use |
|---|---|---|
| F-unknown-01 | Repeated call with four operand-derived arguments | Return value not explicitly consumed |
| F-unknown-02 | First post-loop call | Return value not explicitly consumed |
| F-unknown-03 | Second post-loop call | Return value not explicitly consumed |
| F-unknown-04 | Third post-loop call | Supplies the first argument to F-unknown-05 |
| F-unknown-05 | Final call with one retained result and two loaded values | Supplies R for the 16-bit-store address |

Dynamic targets may coincide. No dependency definitions establish argument validation, register preservation, global preservation, allocation, release, error signaling, or returned-value validity.

A dependency may modify memory later read by this routine. It may also invalidate that memory or prevent further execution. None is assumed to own or release any referenced storage.

## Acceptance examples

These are conditional consequences of the static representation, not runtime test results.

### Loop boundary

With valid incoming state:

- An initial third parameter of −65 skips F-unknown-01.
- An initial third parameter of −64 also skips it.
- An initial third parameter of −63 enters it at least once.

These examples establish only the initial guard outcome.

### Exactly one iteration without arithmetic overflow

Assume an initial third parameter of −63, accumulator −200, operand 0, and a returning F-unknown-01 with working state usable as represented.

The four call arguments are 83, 0, −103, and 0. The adjustment is zero. The updated loop-control value is −147, so the loop exits after that iteration.

### Adjustment is not the original sign bit

Under a 32-bit modular doubling interpretation:

- Operand `0x40000000` doubles to `0x80000000`, producing adjustment 1 despite being nonnegative as a signed value.
- Operand `0x80000000` doubles to zero, producing adjustment 0.

These are conditional arithmetic examples, not proof that overflowing signed operations execute that way.

### Nonoverlapping final stores

Assume P is `0x1000`, R is `0x2000`, G supplies `0x12345678` when evaluated, and all relevant accesses succeed.

The stores are:

1. 16-bit `0x1094` at `0x7FB2`.
2. 32-bit `0x12345678` at `0x1097`.
3. 32-bit `0x10E6` at `0x109B`.
4. 32-bit `0x212E` at `0x109F`.

G is evaluated between the first and second stores. Byte order within stores remains unspecified unless supplied by the execution environment.

### First store modifies G

Additionally assume ordinary little-endian memory, valid accesses at the stated addresses, and that G denotes a 32-bit word beginning at `0x7FB2`. Immediately before the first store, that word contains `0x12345678`.

The 16-bit store of `0x1094` changes the word to `0x12341094`. The subsequent evaluation of G therefore supplies `0x12341094`, and that value is stored at `0x1097`.

An implementation that captures `0x12345678` before the 16-bit store and uses it for the second store does not match this conditional case. Big-endian overlap produces a different numeric result; the required ordering is unchanged.

### Failure at the first store

If the first final store faults before completing, no subsequent evaluation of G or later store is required by the represented path. Earlier dependency effects and completed reads are not shown as rolled back.

## Uncertainties

The following remain **documented_only**:

- Actual instruction validity, entry-point correctness, and control-flow completeness.
- Initialization and calling-convention roles of the additional incoming state.
- Actual address width, endianness, and alignment behavior.
- Overflow, shift, and conversion behavior at arithmetic boundaries.
- Identities and full contracts of all indirect dependencies.
- Dependency effects on globals, registers, memory contents, ownership, and lifetime.
- Whether G is ordinary memory, volatile storage, or a recovered reference with other semantics.
- Validity and application meaning of derived addresses.
- Loop termination for unrestricted entry state.
- Actual execution after the final visible store.

An initialization or record-linking interpretation is possible but is **inferred intent only**, not an acceptance requirement.

## Evidence

**Established static evidence:** The supplied representation shows the signed loop boundary, indirect-call sequence, argument arithmetic, working-state updates, three explicit data loads, one 16-bit store, evaluation of G in the following assignment, three 32-bit stores, and a terminal bad-instruction marker. The assignment sequence places G after the first store and before the second.

**Evidence limitations:** The representation warns of bad instruction data and truncated control flow. No dependency definitions, verified calling convention, memory map, or confirmed arithmetic model are supplied.

**Verification status:** No runtime tests have been performed. Acceptance examples are static control-flow and arithmetic consequences subject to their stated assumptions.
