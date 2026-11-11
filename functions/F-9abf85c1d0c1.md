<!-- Trusted provenance: F-9abf85c1d0c1; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9abf85c1d0c1

## Purpose

F-9abf85c1d0c1 describes a machine-state-dependent routine that performs an ordered initialization sequence, optionally invokes an initialization dependency, and enters repeated indirect calls and computed-address memory accesses.

The supplied static representation establishes no application-level purpose. It does not establish allocation, decoding, device initialization, or a particular data format. Address-zero accesses, potentially unaligned accesses, and an unresolved shift expression remain part of the contract rather than being repaired or suppressed.

No normal return is represented. Continued execution is conditional on memory accesses succeeding, dependencies returning, and execution resuming after any breakpoint.

## Interface

### Effective inputs

There are no explicit parameters in the displayed interface. Effective inputs include:

- An entry 32-bit control value.
- An entry 32-bit address-base value.
- An entry link-register value.
- A stack-associated address.
- A stack-associated pointer-sized value, used in a 32-bit address calculation.
- A 32-bit value at `0x0897ede0`, read once unconditionally and again on the optional path.
- Conditionally, a 32-bit value at `0x0897eca4`.
- Conditionally, a value from global storage associated with `0x0ce00026`. Its read width is unspecified; its obtained value is converted to unsigned 32-bit form.
- Words read through computed pointers.
- Dependency results and additional represented post-call register state.

The calling convention, validity of entry registers, relationship between the two stack-associated inputs, and physical read timing of the stack-associated pointer-sized value are not established.

### Outputs and completion

There is no established return value, status code, completion notification, or normal exit. Observable behavior consists of memory accesses, global assignments, dependency calls, and a conditional software breakpoint.

No allocation, ownership transfer, release, lifetime extension, or valid backing storage is established for any address.

### Width and arithmetic conventions

Explicit accesses include 8-bit, 16-bit, 32-bit, and one 64-bit store; explicit ordinary memory loads are 32 bits. Represented pointer calculations use 32-bit addresses. Represented 64-bit dependency results are divided into low and high 32-bit words.

Unless qualified below, arithmetic and address results are described by their low 32 bits. Narrow stores retain the low 8 or 16 bits. This is a descriptive width model, not a claim that corresponding signed-language overflow or signed shifts have portable semantics.

Several directly assigned globals lack declarations. Their access widths cannot be inferred solely from assigned constants. Byte order, alignment support, negative-value right-shift behavior, and the disputed shift by 32 remain unresolved.

### Preconditions for progress

There is no represented validation layer. Progress requires permission for every executed memory access, indirect call, and special operation, including accesses to zero, low absolute addresses, dependency-selected addresses, and potentially unaligned locations.

## Behavior

### Initial accesses and retained state

Let the entry control, entry address base, and stack-associated address denote their respective input values. The initial observable accesses occur in this order:

1. Store halfword `0x0010` at the address obtained by clearing the control value’s top bit and logically shifting right by one.
2. Store a byte at entry address base plus seven. The byte is the low byte of the stack-associated address logically shifted right by twelve.
3. Store halfword `0xed38` at that shifted stack-associated address plus six.
4. Read and retain the 32-bit value at `0x0897ede0`.
5. Assign the pointer-valued address `0x97ed3800` to global storage associated with that address. The destination access width is not established.

The initial working pointer is the stack-associated address shifted left by four. The accumulator begins as entry address base minus `0xf9`.

Default setup state comprises a selector of zero, selected data address `0x97ed3800`, shifted control equal to entry control shifted left by fourteen, and setup write base `0xa1`.

These accesses precede optional initialization. Aliasing with later input storage can therefore affect later reads.

### Optional initialization

The optional path is taken exactly when the low fifteen entry-control bits are nonzero.

Its accesses and call occur in this order:

1. Assign `0x10` to global storage associated with `0x225fb880`.
2. Assign `0x18` to an interrupt-associated global symbol.
3. Read `0x0897ede0` again as a 32-bit value, then store byte `0xce` at `0x97ed3800` plus this fresh value.
4. Read the global associated with `0x0ce00026`, convert the obtained value to unsigned 32-bit form, and use it as the accumulator.
5. Assign zero to global storage associated with `0x0ce00000`.
6. Invoke F-D01.

F-D01 targets the entry link-register value with its low two bits cleared, plus `0x8a4`. Its three explicit arguments are, in order:

- The low fifteen entry-control bits logically shifted right by one.
- `0xc0`.
- Address `0x0ce00000`.

After the call, the working pointer becomes twice the retained **first** read from `0x0897ede0`. It does not use the fresh conditional read. The selected data address becomes `0x0897eca0`, and a 32-bit value is read at `0x0897eca4`.

Two halfword stores follow:

- `0xeca0` at four times a represented post-call register value.
- `0xeca0` at thirty-two times the value read from `0x0897eca4`, plus six.

The additional post-call register value is not established as a conventional return value.

This path selects setup write base zero, selector sixteen, and shifted control zero. When the condition is false, none of this path’s accesses or its call occurs.

### Common setup

Both paths perform these stores in order:

| Destination | Access | Value |
|---|---|---|
| Shifted control plus `0x30` | 16-bit | Low halfword of shifted control |
| Selector times `0x02000000`, plus `0x40` | 32-bit | Selector times `0x02000000` |
| Setup write base | 8-bit | Zero |
| Setup write base | 32-bit | Accumulator |
| Setup write base plus `0x28` | 32-bit | `0x40` |
| Setup write base plus `0x2c` | 32-bit | Setup write base |

The optional path consequently writes a byte and then a word at address zero. The default path uses `0xa1`, which is unaligned for an ordinary four-byte word access.

The accumulator is then masked with the selected data address. The offset becomes eight times that address minus two.

A 32-bit sum is formed from the working pointer, the stack-associated pointer-sized input, and eight. That sum is stored as a word at twice the sum.

Subsequently:

- The primary write pointer becomes selected data address times `0x01000000`.
- The selected data address is stored as a word at setup write base plus four.
- The offset’s low byte is stored at primary write pointer plus `0x14`.
- Shared arithmetic/data state becomes selected data address times `0x02000000`.
- The working value becomes zero.
- The secondary write pointer becomes equal to the primary write pointer.

The shared state is mutable data, not a permanent increment: subsequent reads and extended processing replace it.

### Reentry block

Initial execution enters this block. One later branch also returns here.

The block performs, in order:

1. A zero-byte store through the primary write pointer.
2. A halfword store through that pointer containing the offset’s low halfword.
3. A word store through that pointer containing the working value.
4. Addition of the shared arithmetic/data state to the accumulator.
5. A word store through the secondary write pointer containing the updated accumulator.

Initially the pointers are equal, producing four overlapping stores at one address. Earlier stores remain observable even if the final word overwrites their ordinary-memory contents.

After the block, the argument for the next F-D02 invocation is reset to zero.

### Repeated call sequence

Immediately before each F-D02 call, the routine saves the low byte of the current shared arithmetic/data state. It then performs this ordered sequence:

1. Invoke F-D02 with the current argument and obtain a represented 64-bit result.
2. Treat its low word as an address; store that low word’s low halfword there.
3. Store a word equal to offset times `0x40` at the address equal to that same value.
4. Read a word through the working pointer, replacing the shared arithmetic/data state.
5. Store a halfword at offset plus four containing the low sixteen bits of the old working-pointer value plus four.
6. Read a replacement working pointer from the address given by F-D02’s high result word plus `0x50`.
7. Invoke F-D03, then F-D04, without represented explicit arguments.
8. Invoke F-D05 with the word read in step 4.
9. Invoke F-D06 with that word logically shifted right by three.

The self-addressed store in step 3 is independent of F-D02’s result. The saved byte predates the word read in step 4.

F-D06 supplies another represented 64-bit result. Its high word becomes the candidate record pointer and replaces the primary write pointer. Its low word, multiplied by thirty-two with 32-bit truncation, becomes the working value. This product is stored as a word at the address equal to the product.

Only **after** that store is the product compared unsigned with `0x14`:

- A product below `0x14` restarts the call sequence with argument zero, without executing the reentry block.
- Otherwise execution enters extended processing.

A truncated 32-bit multiple of thirty-two is below `0x14` only when zero. The comparison therefore does not guard against a zero-address write; that write has already occurred.

On the restart, shared arithmetic/data state remains the fetched word. The next saved byte comes from that word.

### F-D02 target selection

F-D02’s target is the selection seed plus `0xa1e`, at the stated address width.

| Route to the next call | Selection seed |
|---|---|
| Initial call after setup and reentry | `0x08bfed9a` |
| Restart after zero product | `0x0897edf1` |
| Extended processing followed by zero-derived-value reentry | `0x0897ee1d` |
| Extended processing followed by the nonzero-derived-value path, if execution resumes | `0x08790d52` |

These targets need not identify a common implementation. The intermediate constant `0x0897eddf` is not supplied as an argument to a subsequent F-D02 call.

### Extended processing

For a nonzero product, the routine:

1. Stores the offset as a word at product plus `0x58`.
2. Stores the replacement working pointer’s low halfword at offset plus `0x0e`.
3. Replaces shared arithmetic/data state with offset shifted left by two.
4. Selects a fixed function-symbol address, designated **F-B01**, as the base for F-D08.
5. Assigns the previously saved byte to a reset-associated global symbol.
6. Invokes F-D07 with the working-pointer value logically shifted right by two.
7. Invokes F-D08, without represented explicit arguments, at F-B01 plus `0xffe`.

F-B01 denotes the fixed symbol address established before F-D07. Its numerical address is not independently established here; no address is inferred from the symbol’s spelling. F-D08’s target expression remains specified even though its numeric target is unresolved.

F-D08 returns a represented 64-bit result. Its low word supplies a write address; its high word becomes the secondary write pointer. Stores occur in this order:

| Destination | Access | Value |
|---|---|---|
| Twice the low-word address | 8-bit | Saved byte |
| Low-word address plus eight | 8-bit | Low byte of the high word |
| Twice the current accumulator | 32-bit | Candidate record pointer |
| Low-word address | 64-bit | Complete F-D08 result |

The accumulator then becomes four times the candidate record pointer. The offset becomes working-pointer value shifted left by eight.

Three consecutive words are stored through the candidate record pointer, at displacements zero, four, and eight: shared arithmetic/data state, the new offset, and working-pointer value, respectively. These accesses establish neither a record schema nor an allocation guarantee.

The working value becomes shared arithmetic/data state times `0x100`. The primary write pointer becomes zero.

### Unresolved decision and breakpoint

The next derived value depends on a signed interpretation of the working-pointer value, a left shift by thirty-one, a signed right shift by thirty-two, conversion to unsigned 32-bit form, and a logical right shift by twenty.

The static representation does not establish a portable interpretation of the shift by thirty-two. Neither outcome may be discarded.

If the derived value is zero, execution returns through the reentry block with primary write pointer zero and F-D02 selection seed `0x0897ee1d`. That block resets the next argument to zero.

If nonzero, the routine:

1. Stores the derived value as a word at working value plus `0x20`.
2. Selects seed `0x08790d52`.
3. Sets the next F-D02 argument to the working value interpreted as signed and right-shifted by twenty-nine.
4. Performs a software breakpoint with immediate value four.

If execution resumes, the offset becomes the secondary pointer interpreted as signed and right-shifted by eighteen. The call sequence resumes without reentry writes or argument reset. Negative-value signed right shifts remain target-dependent.

## Side effects

The routine accesses absolute globals, stack-derived and entry-register-derived addresses, zero and low addresses, and locations selected by memory contents or dependency results.

Access order and width are contractual where established. Overlapping writes cannot be replaced by only their final contents: separate accesses may affect devices or fault independently. A 64-bit store does not establish atomicity.

Interrupt- and reset-associated symbol names do not establish hardware effects. Effects depend on declarations and memory mapping.

Dependency behavior is unknown. Dependencies may modify globals or reachable memory; preservation, allocation, and release are not assumed. Retained values are distinguished from later reads, which may observe intervening changes.

There is no represented bounds check, null check, alignment check, rollback, recovery, cleanup, or ownership operation.

## Dependencies

Opaque IDs identify represented call sites, not necessarily distinct implementations.

| ID | Established destination | Established interface |
|---|---|---|
| F-D01 | Entry link-register value with low two bits cleared, plus `0x8a4` | Three explicit arguments; additional post-call register state consumed |
| F-D02 | Selection seed plus `0xa1e`, using the documented transitions | One 32-bit argument; represented 64-bit result |
| F-D03 | `0x0897f191` | No explicit arguments represented |
| F-D04 | `0x0897fddd` | No explicit arguments represented |
| F-D05 | `0x0897f5e1` | One 32-bit argument |
| F-D06 | `0x07f755f6` | One logically shifted 32-bit argument; represented 64-bit result |
| F-D07 | `0x0897ee9f` | Working-pointer value logically shifted right by two |
| F-D08 | Fixed function-symbol address F-B01 plus `0xffe` | No explicit arguments represented; represented 64-bit result |

F-D03, F-D04, F-D05, and F-D07 have established numerical destinations despite being expressed through address arithmetic. F-D08 has an established symbolic base and displacement, not an independently established numerical base.

Dependency semantics beyond these interfaces are **documented_only**. No dependency is established to return normally, validate addresses, preserve globals, allocate storage, or release memory.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes the relevant operation is reached and preceding accesses succeed.

1. **Default setup:** If the low fifteen control bits are zero, F-D01 is skipped. Setup uses base `0xa1`, including stores at `0xa1`, `0xa5`, `0xc9`, and `0xcd`.

2. **Optional setup:** If those bits equal three, F-D01 receives one, `0xc0`, and `0x0ce00000`. Common setup uses base zero. Conversion of the value from `0x0ce00026` to unsigned 32 bits does not establish its read width.

3. **Distinct reads:** If the first read at `0x0897ede0` yields five and the fresh optional read yields nine, the optional byte store targets `0x97ed3809`; after F-D01 returns, the working pointer becomes ten.

4. **Overlapping stores:** Initial reentry performs a byte store, a halfword store, and two word stores at the same address, in that order. Checking only the final word is insufficient.

5. **Zero product:** An F-D06 low word of zero causes a word store at zero before restarting with F-D02 argument zero and seed `0x0897edf1`. Reentry writes are not repeated.

6. **Wrapped product:** An F-D06 low word of `0x08000000` likewise produces zero after multiplication by thirty-two with 32-bit truncation.

7. **Extended entry:** An F-D06 low word of one produces `0x20`, stored at address `0x20`. Extended processing then stores the offset at `0x78`.

8. **Call destinations:** When the repeated sequence reaches F-D03 through F-D06, their destinations are respectively `0x0897f191`, `0x0897fddd`, `0x0897f5e1`, and `0x07f755f6`. Extended processing calls F-D07 at `0x0897ee9f`, then F-D08 at F-B01 plus `0xffe`.

9. **Zero derived value:** Conditional on that unresolved result being zero, reentry writes use primary pointer zero; the next F-D02 argument is zero and its seed is `0x0897ee1d`.

10. **Nonzero derived value:** Conditional on a nonzero result and breakpoint resumption, F-D02 uses seed `0x08790d52` and the signed-right-shift-by-twenty-nine argument. The offset is separately replaced using the secondary pointer’s signed right shift by eighteen.

11. **Fault:** If a required access faults, later calls, comparisons, or restarts are not guaranteed. No represented rollback applies.

## Uncertainties

- **documented_only — global widths:** Several global declarations are absent, including the read width at `0x0ce00026`.
- **documented_only — disputed shift:** The shift by thirty-two has no established portable interpretation; its result and branch outcome remain unresolved.
- **documented_only — signed arithmetic:** Signed overflow, signed left shifts, and negative-value right shifts require platform-specific interpretation.
- **documented_only — platform:** Byte order, alignment behavior, memory mapping, fault handling, breakpoint handling, and special-symbol effects are unknown.
- **documented_only — calling convention:** Entry-state validity, stack-input placement, and the additional post-F-D01 register state are unresolved.
- **documented_only — F-B01:** The symbolic target base is established, but its numerical address is not independently established.
- **documented_only — completeness:** The supplied representation alone does not establish fidelity to a complete original instruction stream.
- **documented_only — termination and ownership:** No normal exit or memory-management protocol is established.
- **Inferred intent only:** Consecutive record-shaped stores suggest structured-state manipulation, but establish neither a type nor an application purpose.

## Evidence

Established behavior derives from explicit accesses, widths, expressions, represented call interfaces, and control-flow connections in the supplied static representation.

The F-D03, F-D04, F-D05, and F-D07 destinations follow their displayed address arithmetic. F-D06 has an explicit fixed target. F-D08 uses the fixed function-symbol base selected before F-D07, plus `0xffe`; its numerical base is not inferred from symbol spelling.

Control-flow connections establish the distinct restart routes and F-D02 arguments. Width arithmetic establishes that a truncated multiple of thirty-two below `0x14` must be zero. Neither deduction resolves dependency semantics or the disputed signed shift.

No runtime tests have been performed.
