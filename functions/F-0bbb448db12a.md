<!-- Trusted provenance: F-0bbb448db12a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0bbb448db12a

## Purpose

The established behavior is a control-dependent sequence of memory accesses followed, if execution continues, by an unconditional dispatch-and-interrupt loop.

Under the supplied 32-bit arithmetic and control-flow model:

- A control word whose low six bits are nonzero causes no explicit pre-loop data-memory access.
- A control word whose low six bits are zero causes one ordered 16-bit transfer.
- Additional zero-result gates can permit an auxiliary-state update and an integer-addressed 16-bit store.
- Every subsequent pre-loop arithmetic and record-writing block is unreachable.
- The initial dispatch base is always the incoming continuation address.

No application-level purpose is established. Describing this function as a decoder, allocator, serializer, or error handler would add unsupported intent.

This contract separates established static behavior from syntactic content that cannot execute under the stated model. Target-specific semantics and unavailable dependency contracts remain **`documented_only`**. No runtime tests have been performed.

## Interface

### Explicit inputs

The displayed interface contains five positional inputs:

| Position | Displayed width | Established use |
|---|---:|---|
| First | 32 bits | No visible use. |
| Second | 32 bits | No visible use. |
| Third: **control word** | 32-bit unsigned | Arithmetic classification, low-six-bit arithmetic, and bit-4 inspection. |
| Fourth | 32 bits | No visible use. |
| Fifth | 16 bits | Its address is used; its incoming numeric value is not explicitly consumed. |

The absence of a visible use does not establish that an input is absent from the actual external calling convention.

### Implicit entry state

The body also consumes state not explained by the displayed positional interface:

| Role | Apparent width | Use |
|---|---:|---|
| **Context A** | 32 bits | Equality gate. |
| **Context B** | 32 bits | Initial destination offset and later arithmetic gate. |
| **Source address** | 32 bits | Initial 16-bit read address; subsequently replaced by an adjusted address. |
| **Context C** | 32 bits | Source-address adjustment and equality gate. |
| **Accumulator** | 32 bits | Addition gate. |
| **Continuation address** | 32 bits | Initial dispatch base and exceptional memory-read address. |
| **Argument-slot address** | 32 bits | Address of the fifth input’s represented storage. |
| **Auxiliary state component** | Not established | Assigned on the reachable exceptional path. |

These are descriptive roles, not a verified ABI. The auxiliary component’s width, architectural meaning, and relationship to actual storage are not established.

### Result and termination

No explicit result value or normal return is shown.

Execution that reaches the final loop repeatedly attempts two calls and a software interrupt. A fault, a nonreturning call, or an interrupt that does not return can stop that progression. There is no visible recovery path or error result.

### Arithmetic and memory model

The behavioral reductions below use the apparent 32-bit address model:

- Unsigned 32-bit arithmetic retains the low 32 bits.
- Unsigned comparisons and carry conditions are distinct from signed overflow.
- Logical right shifts insert zeros.
- Movement by one 16-bit element means two bytes.
- Integer-address offsets are byte offsets.
- A 16-bit store retains the low 16 bits of its value.
- A signed 16-bit load, where present in unreachable content, is sign-extended to 32 bits.

These rules describe the supplied representation. They do not establish that integer-derived addresses are valid, aligned, or portable.

## Behavior

### Control classification

The function forms a 32-bit product of the control word and `0x04000000`.

That product is zero exactly when the control word’s low six bits are zero. The remaining control bits cannot affect this classification.

A second product, formed only on the zero-product path, uses `0x08000000`. It is also zero on that path.

### Nonzero low six bits

If any low-six-bit control bit is set:

- The initial transfer is skipped.
- All subsequent pre-loop conditional effects are skipped.
- The function reaches the dispatch loop with the incoming continuation address as its base, assuming execution otherwise continues.

There is no explicit data-memory load or store before the first call on this path. This excludes instruction fetches and any effects of the called dependency.

Bit 4 can affect internal Boolean state, but it does not enable a pre-loop memory operation on this path.

### Zero low six bits: initial transfer

If the control word’s low six bits are zero, the function computes:

- An address-related working value from the continuation address plus the argument-slot address logically shifted right by one. The represented expression also includes the original bit-4 condition, but that condition is necessarily false here.
- A destination equal to the incoming source address plus Context B, modulo the apparent address width.

The relevant sequence is:

1. Read an unsigned 16-bit value from the incoming source address.
2. Replace the working source address with the destination minus Context C.
3. Write the previously read 16-bit value to the destination.
4. Replace another working address value with the incoming continuation address.

The read precedes the write even when source and destination coincide. No bounds, alignment, or accessibility check is shown.

The source-address replacement occurs before the store. A faulting store therefore need not imply that all preceding internal state remained unchanged.

### Gates leading to the exceptional store

Further processing first requires the incoming continuation address to be zero.

With a zero continuation, the next zero-result gate requires the argument-slot address logically shifted right by one to be zero. Under the stated unsigned 32-bit model, the slot address must therefore be either zero or one.

If the slot address is at least two, the remaining pre-loop effects are skipped.

For slot addresses zero or one, the following address-comparison stages reduce to zero tests because their shift counts are zero. The chain then requires:

1. Context A equals Context B.
2. The updated source address equals Context C.
3. An accumulator addition produces zero.
4. The argument-slot address plus twice Context B produces zero modulo \(2^{32}\).

On this path the carry entering the accumulator addition is false, and the low-six-bit control contribution is zero. Consequently, the accumulator must be zero, and that addition produces no carry.

The final gate is therefore exactly a modular sum of the slot address and twice Context B. Twice a 32-bit quantity is even, including after wraparound. A slot address of one cannot make that sum zero.

Thus, reaching the auxiliary update requires all of the following:

- Zero low-six-bit control value.
- Zero incoming continuation address.
- Zero argument-slot address.
- Context A equal to Context B.
- Updated source address equal to Context C.
- Zero accumulator.
- Twice Context B equal to zero modulo \(2^{32}\).

The last condition allows Context B to be either `0x00000000` or `0x80000000`. The multiplication is a 32-bit operation; its mathematical overflow is not itself an additional carry input.

These are necessary arithmetic conditions, not evidence that such an entry environment is valid.

### Reachable exceptional effects

If all preceding gates succeed and the memory accesses complete, the function:

1. Assigns the auxiliary state component from the argument-slot address multiplied by 64. The assigned value is necessarily zero.
2. Forms an updated continuation address by advancing the old continuation by 64 times the slot address in bytes. This address is necessarily zero.
3. Uses unsigned 16-bit contents addressed through the old continuation to form a store destination and a stored value.
4. Advances a separate working address by one 16-bit element, making that working address equal to two.
5. Commits the updated continuation address, which remains zero.

The exceptional store expression references the 16-bit contents at the old continuation more than once:

- One reference supplies a zero-extended 16-bit destination address.
- Another supplies a value increased by 15 and narrowed to 16 bits.

The old continuation is necessarily zero here. If its referenced contents remain stable at a value \(v\), the effect is a 16-bit store of \((v+15)\bmod 2^{16}\) to byte address \(v\).

The representation does not establish a single shared snapshot for the repeated references, their physical read count, or their relative evaluation order. It does establish that the values required for the store must be obtained before that store completes.

There is no allocation or destination validation.

### Why the later blocks cannot execute

After the reachable exceptional store:

- The continuation-derived working address is zero.
- The working address formed from the original continuation and shifted slot address is zero.
- The separately advanced working address is two.

The next exclusive-or gate therefore produces zero and passes.

The following gate combines the advanced address, two, with the other zero working value using a zero shift count. Its result is two, not zero. That gate fails.

Later gates are disabled by their control-flow conditions. They cannot independently become enabled.

If the auxiliary-update block was not entered, its outgoing enable condition is false, so the same later region is also skipped.

Therefore, under this model, no execution reaches the subsequent pre-loop arithmetic, context increments, pointer lookup, record stores, or final write pair. Reaching those blocks would require an explicitly different arithmetic model, different control flow, or an intervening state effect not represented here.

This also establishes that none of those blocks changes the initial dispatch base. On every path reaching the loop, that base equals the incoming continuation address.

### Unreachable syntactic content

The representation contains the following additional operations, but they are **not possible side effects under the established model**:

- Arithmetic involving `0x085740E8`, whose decimal value is **139,935,976**.
- A left shift using a low-byte count, followed by carry-like or borrow-like subtraction conditions.
- A conditional 36-byte address advance and a logical right shift.
- Context B and Context C increments of 8 and 10.
- A continuation-derived 13-byte address advance.
- A pointer-sized lookup through an address formed by a character-sized conversion of a 16-bit value plus four.
- An apparent two-field record containing a sign-extended 16-bit value at byte offset zero and another 32-bit value at byte offset four.
- A one-byte source-state advance associated with that record.

The final unreachable write pair has this represented order:

1. Form a read address at the old working base plus 13 bytes.
2. Advance the working base by 27 bytes.
3. Read a signed 16-bit value through the previously formed address and sign-extend it.
4. Store that 32-bit value at an address obtained by subtracting `0x08574200`, or **139,936,256**, from it.
5. Store another apparent 32-bit value four bytes later.

The base update therefore precedes the signed read, although the read address uses the old base.

Oversized shift behavior and character signedness in this unreachable region do not create uncertainty about the established reachable paths.

### Dispatch loop

Let the initial dispatch base be the incoming continuation address.

Each iteration attempts, in order:

1. A call to the executable address at the current dispatch base plus `0x0EB6`, or 3,766 bytes.
2. A call to executable address `0x0857A94B`.
3. Software interrupt `0x5B`.
4. Assignment of `0x083D4CC2` as the dispatch base for the next iteration.

The first call has three explicit 32-bit argument patterns: zero, zero, and `0xFFFFFFA5`. Treating the last pattern as signed −91 requires a dependency signature that is not supplied.

No explicit arguments are shown for the second call. That does not establish the absence of implicit inputs.

After a completed iteration, the next dispatch-relative target is `0x083D5B78`.

The second call occurs only if the first returns. The interrupt occurs only if both calls return. The base replacement and next iteration require the interrupt also to return control to the represented continuation.

## Side effects

Established potential effects are limited to:

- One ordered 16-bit transfer on the zero-low-six-bits path.
- Under the exceptional gates, an auxiliary-state assignment and a further integer-addressed 16-bit store.
- Updates to working address and arithmetic state.
- Calls to computed executable addresses.
- Repeated requests for software interrupt `0x5B`.

The unreachable record writes and later context updates are not established potential effects.

No explicit allocation, release, reference counting, synchronization, or ownership transfer is shown. A writable address is not thereby owned or exclusively accessible.

The call targets and interrupt may have additional effects. Nothing supplied establishes that they preserve globals, retain memory, release memory, or leave implicit machine state unchanged.

There is no explicit error reporting. Invalid reads, writes, executable targets, or interrupt conditions can prevent progress after earlier effects have occurred.

Safe execution of any reached operation requires the relevant access or control transfer to be supported by the actual environment. Required alignment, mappings, privileges, and fault delivery are **`documented_only`**.

## Dependencies

The only supplied opaque function identity is **F-0bbb448db12a**, identifying this function. No recursive call to that identity is established.

The two external call sites have no supplied opaque function identities. They cannot be assigned known dependency identities from address arithmetic alone.

| Interaction | Established interface | Unresolved contract |
|---|---|---|
| Dispatch-relative call | Executable target at base plus `0x0EB6`; three explicit argument patterns | ABI, implicit inputs, return behavior, memory effects, ownership effects, and preserved state |
| Fixed-target call | Executable target `0x0857A94B`; no explicit arguments shown | Signature, implicit inputs, return behavior, and state effects |
| Software interrupt | Interrupt number `0x5B` | Service meaning, privilege rules, effects, and return behavior |

The computed calls are not displayed as loads from a function-pointer table.

## Acceptance examples

These are conditional static expectations, not executed tests.

### Nonzero low-six-bit control

Given control word 1 and continuation `0x1000`:

- No explicit pre-loop data-memory access occurs.
- The first attempted call target is `0x1EB6`.
- Its explicit arguments are zero, zero, and `0xFFFFFFA5`.

Executability of that target is not asserted.

### High control bits alone

Given control word `0x40`, source address `0x2000`, Context B equal to 6, and source contents `0x1234`:

- The primary product is zero.
- The function reads `0x1234` at `0x2000`.
- The working source becomes `0x2006` minus Context C.
- It writes `0x1234` at `0x2006`.

A nonzero continuation prevents entry into the exceptional chain.

### Aliased initial transfer

With zero low control bits and Context B equal to zero, the initial source and destination coincide.

The 16-bit read still precedes the 16-bit write. This does not authorize removing accesses to externally observable memory.

### Ordinary slot, zero continuation

With zero low control bits, continuation zero, and slot address `0x1000`:

- The initial transfer occurs.
- The shifted-slot gate fails.
- No auxiliary update or exceptional store occurs.
- The first attempted call target is `0x00000EB6`.

No null-continuation rejection is shown.

### Slot address one

With continuation zero and slot address one, the shifted-slot gate can pass.

Nevertheless, the slot-plus-twice-Context-B gate cannot produce zero under 32-bit arithmetic. The auxiliary update and exceptional store cannot occur.

### Exceptional store followed by a failed gate

Assume:

- Control word, continuation, slot address, accumulator, Context A, and Context B are zero.
- Source is `0x2000` and Context C is `0x1000`.
- All required accesses are supported.
- The repeated 16-bit references at address zero observe stable value `0x3000`.

Then:

- The initial transfer reads and writes at `0x2000`.
- The updated source is `0x1000`, satisfying its equality gate.
- The auxiliary component receives zero.
- The exceptional store writes `0x300F` at address `0x3000`.
- The continuation remains zero.
- The next two gate results are zero and two.
- No later record write occurs.
- The first attempted call target remains `0x00000EB6`.

### Repeated dispatch

If both calls and the interrupt return control, the next iteration calls `0x083D5B78`, then `0x0857A94B`, then requests interrupt `0x5B`.

Nonreturn of any interaction removes the requirement for later events.

## Uncertainties

The following remain **`documented_only`**:

- The actual calling convention and complete implicit input state.
- Whether zero or one can be valid argument-slot addresses.
- Whether the exceptional arithmetic conditions can coexist with a valid execution environment.
- Address validity, alignment requirements, endianness, and fault behavior.
- Physical read count and evaluation order for repeated exceptional-store references.
- The auxiliary component’s representation and architectural effects.
- Dependency and interrupt semantics, including state preservation and ownership.
- Whether executable-address low bits encode execution state.

Character signedness and oversized shifts remain unresolved properties of unreachable syntactic content. They do not weaken the proof of unreachability under the stated model.

No supported application-level intent is promoted to a requirement.

## Evidence

This contract uses only the supplied static representation.

The reachable-path conclusions follow from 32-bit modular multiplication, zero-result gates, evenness of twice a 32-bit value, and the two gate results following the auxiliary update. In particular, the slot must be zero, the continuation remains zero, and the advanced working address becomes two, disabling every later pre-loop effect.

The constant conversions are `0x085740E8` = 139,935,976 and `0x08574200` = 139,936,256. The final unreachable read occurs after its working-base update but uses an address formed beforehand.

No runtime execution, dependency inspection, or external ABI verification has been performed.
