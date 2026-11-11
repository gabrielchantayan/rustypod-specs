<!-- Trusted provenance: F-ec22cb7b1c36; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-ec22cb7b1c36

## Purpose

The supplied static representation establishes a conditional sequence of indirect calls, memory accesses, arithmetic-controlled loops, and exceptional operations. It does not establish an application-level purpose or a normal completion protocol.

**Inferred intent:** the function appears to transform machine state using an incoming address-like value, register-derived storage, and computed records. That interpretation does not establish a public data format, safe operating range, or useful result.

**Established limitation:** the representation reports invalid instruction data and truncated control flow. This contract describes the displayed operations if execution reaches them and preceding operations permit continuation. It does not guarantee that the entire sequence is executable or reachable.

## Interface

### Entry state and result

The displayed interface declares no parameters and no return value. Nevertheless, behavior depends on hidden inputs:

- A return-address-related value used to construct the first indirect target.
- A 32-bit signed incoming value used both as an arithmetic operand and as an address base.
- An additional register result used after an indirect call.
- Register-derived storage interpreted as a cursor into 32-bit words.
- Stack-relative storage and stack-relative addresses.

Their calling convention, initialization, and validity requirements are **documented_only** unknowns.

No normal return is displayed. The final visible operation is an invalid-data halt. Dependency results are intermediate state, not an established result of this function.

### Widths and interpretation

The visible operations include 8-, 16-, 32-, and 64-bit memory accesses. Address calculations use 32-bit values in the displayed model.

Unsigned 32-bit arithmetic retains the low 32 bits. Narrow stores retain the low 8 or 16 bits. Signed right shifts are distinguished below from logical right shifts. Signed expression reconstruction, including overflowing signed arithmetic, is not an independently established execution-language guarantee; resulting 32-bit bit patterns are described where relevant.

The initial character read has unresolved signedness. Later byte reads are explicitly unsigned or byte-sized. Byte order, alignment restrictions, and the consequences of invalid or unaligned accesses are unknown.

### Preconditions for conditional continuation

Every reached call target must support the actual calling convention; every reached memory access must be permitted for its width. Exceptional operations must permit compatible resumption for later operations to occur. Loop predicates must eventually become false for execution to progress beyond their loops.

The function visibly checks none of these conditions. No bounds validation, allocation validation, error return, or rollback protocol is established.

## Behavior

### 1. Entry calls and initial loop targets

Execution starts with three indirect calls without explicit arguments. Their target expressions, in order, are:

| Call | Target expression |
|---|---|
| First entry call | Incoming return-address-related value plus `0xed0` |
| Second entry call | `0x08572109 + 0xf0c` |
| Third entry call | `0x0857210d + 0xf36` |

The initial loop nest then starts with an offset of `0x199`.

Indirect-target state changes are significant:

| Call position | First entry | Repeated entry |
|---|---|---|
| First call of the outer loop | `0x08572111 + 0xf46` | `0x0857215d + 0xf46` |
| First call of the inner loop | `0x08572115 + 0xf30` on the first inner iteration of each outer iteration | `0x08572125 + 0xf30` on subsequent inner iterations |

Thus, a new outer iteration resets the first inner-call target to its first-entry expression. Repetition within that inner loop uses the other expression. Different target expressions must not be treated as calls to the same dependency without further evidence.

After the first call of each inner iteration, the remaining three calls have target expressions:

1. `0x08572119 + 0xefc`, with no explicit arguments.
2. `0x0857211d + 0xeaa`, producing a 64-bit result.
3. `0x08572121 + 0xe6a`, receiving that result’s low 32 bits and its logical right shift by 57.

The second argument to the last call is at most `127`; it is not the upper 32-bit half. The last call’s result replaces a stack-associated 32-bit value.

Next, the function forms a derived address value by taking the low 32 bits of the incoming base minus the additional register result, then logically shifting right by 16. It reads a character at the incoming base plus the current offset.

The inner loop repeats exactly when both displayed conditions hold:

- The incoming base is nonzero.
- Its signed negativity predicate equals the signed-subtraction-overflow predicate for current offset minus `0x42`.

The overflow predicate is not an unsigned borrow test. The current offset remains unchanged within this inner loop.

### 2. Initial outer-loop writes and continuation

After the inner loop exits, a no-explicit-argument call uses target `0x08572125 + 0xdc2`.

The function then:

1. Replaces the offset with `0x0085723a`.
2. Multiplies the incoming base by the character just read.
3. Records whether the resulting 32-bit product is zero.
4. Writes the product’s low 16 bits at base plus `0x22`.
5. Writes the base’s low byte at base plus `1`.
6. Calls target `0x08572145 + 0x2e0` without explicit arguments.
7. Establishes a pointer to a stack-relative 32-bit slot.
8. Calls target `0x08572159 + 0xf56` without explicit arguments, retaining its 32-bit result.

At the displayed expression level, the outer loop repeats when the product is zero. Its other continuation flag is explicitly set true before the last two calls. The reliability of this reconstructed flag behavior remains uncertain.

A repeated outer iteration uses offset `0x0085723a` for its character reads and the repeated-entry outer-call target specified above. There is no pointer-validity check before the base-relative writes.

### 3. Breakpoint and identified dependencies

After the outer loop exits, the function writes `0x723a` as 16 bits at the latest derived address plus `0x34`. It then shifts the last indirect-call result signed-right by 13 and executes a software breakpoint with immediate `0xac`.

If execution resumes, **F-c5ddd881f572** receives:

- A 32-bit read at the stack-slot address plus the incoming base.
- A byte read at base plus `5`.

The relative evaluation order of these argument reads is not established.

Its result is consumed as 64 bits. The following operations occur in order:

1. Write the low byte of a nearby stack-relative address at the sum of the shifted signed result and the dependency result’s upper 32 bits.
2. Write the low 16 bits of the base shifted signed-right by four at the dependency result’s low 32-bit address plus `4`.
3. Store the incoming base into the stack-relative 32-bit slot.
4. Call **F-f28d5d0f667f** with the preceding result’s low 32 bits and a newly read byte from base plus `5`.
5. Execute a software interrupt with immediate `0x53`.

The second byte read is independent of the first. Intervening writes, aliasing, or dependency effects may change its value.

The second dependency result is also consumed as 64 bits. Neither result has established ownership or lifetime semantics.

### 4. Cursor-processing outer entry

After interrupt resumption, a 32-bit working value starts at zero. A storage cursor comes from register-derived state.

At each outer entry, the function:

1. Retains the current cursor for that iteration.
2. Shifts the working value logically right by 12.
3. Reads an unsigned byte at that value plus `0x14`.
4. Makes an indirect call with four arguments.
5. Makes another indirect call, at `0x085721c5 + 0xe7e`, obtaining a signed 32-bit result.

The first call’s target changes across iterations:

- First outer iteration: `0x085721ad + 0xe54`, using a fixed code-address-derived base.
- Subsequent outer iterations: `0x0894a20a + 0xe54`, using the fixed data-address-derived base selected after the preceding inner loop.

The latter expression is still used as an indirect call target in the representation. Its data-address origin does not establish executability or dependency identity.

The first call receives:

1. The low 32 bits of the current 64-bit result.
2. A signed 32-bit carried value, whose transition is specified below.
3. The newly read unsigned byte.
4. The low 32 bits of the current result shifted signed-right by 52, then shifted logically right by eight.

The second argument is **not** a repeatedly shifted call result:

- On the first outer iteration, it is the earlier indirect-call result shifted signed-right by 13, as prepared before the breakpoint.
- On subsequent outer iterations, it is the preceding iteration’s last preserved control value minus `0x21`, retained as a signed 32-bit bit pattern.

That carried value is established after the inner loop and is also used for stores later in the same iteration.

Because the initial working value is zero, the first byte read targets numeric address `0x14`. No fallback address is shown.

The fourth argument’s mixed shifts matter. For an all-ones 64-bit input it yields `0x00ffffff`, not `0x0000000f`.

### 5. Cursor-processing inner loop

The signed result from the second outer-entry call becomes the first inner-call argument. Its signed right shift by 22, retained as 32 bits, becomes the companion argument.

Each inner iteration preserves the current control value before replacing it. Initially this control value is the outer-entry byte.

Nine indirect calls then occur:

| Position | Target expression | Explicit interaction |
|---|---|---|
| 1 | First iteration: `0x085721c9 + 0xeb0`; repeated inner iterations: `0x085721ed + 0xeb0` | Receives current signed result and companion |
| 2 | `0x085721cd + 0xee0` | No explicit arguments |
| 3 | `0x085721d1 + 0xf02` | Supplies a 32-bit result |
| 4 | `0x085721d5 + 0xf0e` | No explicit arguments |
| 5 | `0x085721d9 + 0xf08` | No explicit arguments |
| 6 | `0x085721dd + 0xef0` | No explicit arguments |
| 7 | `0x085721e1 + 0xec6` | No explicit arguments |
| 8 | `0x085721e5 + 0xe8c` | Supplies a 32-bit result |
| 9 | `0x085721e9 + 0xe5a` | Receives the eighth result shifted logically right by 13; supplies 64 bits |

Each new cursor-processing outer iteration restores the first-entry target for call 1.

The third call’s result minus three forms an unsigned 32-bit threshold. Between calls 5 and 6, the function forms an intermediate value equal to the current working value plus six.

After call 9, let its low and high 32-bit words be the low and high result words. By default, the retained intermediate is working value plus six, and a retained derivative is that intermediate shifted logically right by 12.

If the preserved control value is nonzero:

- The threshold becomes the intermediate shifted logically right by eight, then signed-right by 12, minus `0x55`.
- The retained intermediate becomes twice the low result word.
- The retained derivative becomes the previous working value.

The function then:

1. Sets the working value to twice the low result word, modulo \(2^{32}\).
2. Stores the low result word at cursor plus `0x28c`.
3. Forms the next signed-result bit pattern as low result word minus threshold, minus one if bit 31 of the low result word is clear.
4. Sets the next control value to preserved control minus `0x53`, modulo \(2^{32}\).

The high result word becomes the companion argument on another inner iteration.

Repetition occurs if either unsigned condition holds:

- Threshold is less than the low result word.
- Low result word minus threshold, modulo \(2^{32}\), is less than the bit-31 indicator.

The indicator is one when bit 31 is set and zero otherwise. It comes from the low word, not the high word.

### 6. Cursor stores, record reads, and outer continuation

After inner-loop exit, the function selects fixed address `0x0894a20a` for subsequent target construction, reads 32 bits at cursor plus `0x2cc`, and executes a software halt with immediate `0x26`.

If execution resumes:

- A signed displacement bit pattern becomes the last preserved control value minus `0x21`.
- A record address becomes threshold plus `0x70`, modulo \(2^{32}\).
- The record address’s low 16 bits are stored at displacement plus current working value.

Five 32-bit stores follow, in order:

| Address | Value |
|---|---|
| Cursor minus `4` | `0x0894a20a` |
| Cursor minus `8` | Retained intermediate |
| Cursor minus `12` | Word read at cursor plus `0x2cc` |
| Cursor minus `16` | High result word |
| Cursor minus `20` | Retained derivative |

The record reads then occur in this order:

| Offset | Width | Use |
|---|---:|---|
| `0` | 32 bits | Retained final-store value |
| `4` | 32 bits | Signed address-related operand |
| `0` | 64 bits | Next outer iteration’s 64-bit result |
| `12` | 32 bits | Next working value |

The overlapping reads at offset zero are separate observable reads. Their results need not agree if memory changes.

The next cursor is the current cursor minus 20 bytes. Repetition occurs if the record address is zero or threshold plus `0x3e`, retained as signed 32 bits, is less than `-0x32`.

The record reads precede this test. A zero record address does not prevent them.

On repetition, the next first-call target uses the fixed data-address-derived base, and its second argument is the displacement just established—not the earlier result shifted by 13.

### 7. Final visible operations

After outer-loop exit:

1. Read 32 bits at record offset `0x10`; store its low byte at the working address obtained from record offset `12`, plus `0x1d`.
2. Store the earlier 32-bit record-offset-zero value at the displacement plus the signed operand from record offset `4`.
3. Call **F-a041a7d81f49** with a 32-bit read at final-iteration cursor plus `0x24c`, and the signed record operand plus `0x50`.
4. Reach an invalid-data halt.

The cursor-relative argument uses the cursor retained for the final iteration, not the already decremented next cursor. No success status or normal return follows visibly.

## Side effects

Visible effects include stack-relative writes, incoming-base-relative writes, dependency-result-derived writes, repeated cursor-relative writes, descending cursor stores, record-derived stores, calls, and exceptional operations.

Aliasing is unrestricted. Earlier writes may alter later arguments, record fields, cursor storage, or other reachable state. No atomicity, synchronization, disjointness, or transactional behavior is established.

No allocation or release is directly identified. This does not constrain what dependencies do. No address-like result may be classified as owned, borrowed, or releasable from this evidence alone.

## Dependencies

| Opaque function ID | Established interaction | Unresolved contract |
|---|---|---|
| **F-c5ddd881f572** | Receives a 32-bit memory value and a byte; result consumed as 64 bits | Semantics, errors, ownership, memory effects, hidden-state effects |
| **F-f28d5d0f667f** | Receives a low result word and a newly read byte; result consumed as 64 bits | Same unknowns; no established relationship to the preceding dependency |
| **F-a041a7d81f49** | Receives a cursor-relative word and a record-derived value plus `0x50` | Semantics, return behavior, errors, side effects |

Other dependencies remain unresolved indirect targets. Their visible target calculations do not establish opaque identities. Distinct targets must remain distinguishable; common behavior must not be assumed. No callee is assumed to preserve globals or to acquire or release memory.

## Acceptance examples

These are conditional static acceptance criteria, not executed tests.

- **Initial target changes:** A second initial outer iteration uses `0x0857215d + 0xf46`. Its first inner iteration uses `0x08572115 + 0xf30`; another inner iteration uses `0x08572125 + 0xf30`.
- **Seven-bit argument:** An all-ones initial 64-bit result produces `127` after logical right shift by 57.
- **Narrow product store:** A product with low bits `0x12345678` writes `0x5678` at base plus `0x22`.
- **Repeated byte access:** If base plus `5` changes between the identified dependency calls, the second call receives the newly read byte.
- **First cursor access:** Reaching cursor processing attempts the first byte read at `0x14`; inaccessible memory does not select an alternative.
- **Outer argument transition:** If the first iteration’s last preserved control value is `0x10`, a subsequent outer first call receives `0xffffffef` as its second argument, interpreted as signed `-17`. Its target uses `0x0894a20a + 0xe54`.
- **Inner target transition:** Repeating a cursor inner loop changes its first-call target from `0x085721c9 + 0xeb0` to `0x085721ed + 0xeb0`.
- **Equality boundary:** Equal threshold and low result word terminate the inner loop when bit 31 is clear, but repeat it when bit 31 is set.
- **Zero record address:** Record reads are attempted before the zero-address continuation condition.
- **Nonresuming exception:** If the breakpoint does not resume, later dependency calls are not required to occur.

## Uncertainties

The following remain **documented_only**:

- Effects of malformed instruction regions on reconstructed calls, flags, and control flow.
- Actual ABI, entry-state origins, and register-state conventions.
- Initial character signedness and signed-overflow interpretation.
- Byte order, alignment, mappings, and invalid-access behavior.
- Indirect dependency identities and all undocumented callee effects.
- Exceptional-operation handlers and resumption behavior.
- Loop termination, final-path reachability, and normal-return possibility.
- Application purpose, ownership policy, and error policy.

No runtime tests have been performed. These unknowns do not imply safe access, deterministic completion, or successful return.

## Evidence

The sole evidence is the supplied static representation, including its invalid-instruction warning and truncated terminal control flow.

Established static evidence comprises displayed widths, access order, arithmetic, branch predicates, target-state transitions, identified dependencies, and exceptional operations. Application-level purpose is inferred intent. Runtime behavior beyond those conditional observations remains unresolved.
