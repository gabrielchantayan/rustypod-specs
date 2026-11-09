<!-- Trusted provenance: F-bbfb4fd1eaf0; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bbfb4fd1eaf0

## Purpose

The routine performs an entry-time mutation sequence followed by repeated memory writes and dependency calls. Its visible control flow has no normal return: processing either continues directly, restarts an earlier write sequence, or reaches a software breakpoint with a represented continuation.

No domain-level purpose is established. The supplied representation does not justify describing the routine as initialization, allocation, device control, or conversion.

This contract distinguishes:

- **Established static evidence:** represented operations, operand types, access widths where explicit, and control-flow ordering.
- **Conditional interpretation:** behavior requiring an identified machine arithmetic or address model.
- **documented_only:** unresolved behavior for which the supplied material provides insufficient evidence.

All execution descriptions are conditional on preceding accesses and calls completing. No runtime tests have been performed.

## Interface

### Explicit arguments

| Position | Represented type | Visible use |
|---|---|---|
| 1 | Opaque 32-bit value | No explicit use. |
| 2 | Pointer to an unsigned 32-bit word | Receives the original third argument during entry processing. |
| 3 | Unsigned 32-bit word | Supplies initial arithmetic, addresses, and the initial explicit output value. |
| 4 | Opaque 32-bit value | No explicit use. |
| 5 | Signed 32-bit integer | Participates in a signed, pointer-derived calculation during entry. |

The unused arguments remain part of the represented interface. Their absence from the visible body does not establish that they can be omitted from the calling convention.

### Additional entry state

Two values are consumed without being initialized through the explicit arguments:

- A pointer represented as pointing to unsigned 32-bit words.
- An unsigned 32-bit mask.

Their origins and calling-convention roles are unresolved. Supplying only the five explicit arguments is not an established sufficient invocation procedure.

For this contract, **input word** means the original third argument, **current pointer** means the evolving hidden pointer, and **entry mask** means the hidden unsigned value. These are descriptive roles, not additional declared parameters.

### Widths and arithmetic

Explicitly typed memory operations include 8-, 16-, 32-, and 64-bit accesses. Dependency results are represented as 64-bit quantities and are repeatedly separated into low and high 32-bit components.

Unsigned 32-bit arithmetic wraps modulo \(2^{32}\). That rule does **not** apply automatically to an expression evaluated as signed and only subsequently assigned to an unsigned destination.

In particular, candidate-address construction first converts a dependency result to signed 32-bit form and multiplies it by 32 as a signed operation. Overflow of that multiplication is unresolved. A wrapping result is only a conditional machine interpretation.

The locations at `0x28` and `0x2C`, and another named global destination, lack declarations in the supplied material. Their assignments establish supplied values and ordering, but **not destination widths**, signedness, or conversion behavior.

Pointer-derived numeric descriptions below use the represented 32-bit address interpretation. Actual pointer conversion rules, valid mappings, alignment, and endianness remain unresolved.

### Outputs and preconditions

There is no explicit return value or normal return path. Observable outputs are memory mutations, dependency calls, breakpoint behavior, and possible faults or nontermination.

No visible validation protects computed addresses. Reaching a particular event requires that all preceding accesses be valid under the actual memory model and that preceding dependencies return. Ownership, disjoint storage, and preservation of globals by dependencies are not preconditions established by the source.

## Behavior

### Entry mutations and initial state

The first two effects are assignments of `0x40` to the global location identified by `0x28`, then zero to the global location identified by `0x2C`. Their order is established; their access widths are not.

The initial accumulator is the bitwise AND of the entry mask and input word. The initial working word is eight times the input word minus two, using unsigned 32-bit arithmetic.

A separate entry calculation converts the current pointer to signed 32-bit form, adds the fifth argument, and adds eight. The resulting signed intermediate is stored as a 32-bit word at an address formed by signed multiplication of that intermediate by two. Both the additions and the doubling require defined signed arithmetic or an explicitly selected machine interpretation; unsigned wrapping cannot be inferred here.

After that store:

- The initial base address is the input word multiplied by \(2^{24}\), using unsigned arithmetic.
- The original input word is written through argument 2 as a 32-bit word.
- The working word’s low byte is written at base address plus 20.
- The initial increment is the input word multiplied by \(2^{25}\), using unsigned arithmetic.
- The replacement word is the input word shifted left by 28, using unsigned arithmetic.
- A separate write destination is initially equal to the base address.

Thus the two unresolved-width global assignments precede the signed-intermediate store, which precedes the explicit output write. Aliasing can make any of these effects interact.

### Restartable write sequence

This sequence runs once after entry and can run again after the extended processing path.

At the current base address, it performs three distinct writes in order: an 8-bit zero, the working word’s low 16 bits, and the 32-bit replacement word. It then adds the current increment to the accumulator with unsigned 32-bit wrapping and writes the updated accumulator as a 32-bit word through the separate write destination.

Initially, the base and separate destination coincide. After a restart they generally do not.

These operations must not be described as a single final assignment. Even where ordinary memory ends with the same bytes, individual accesses can differ in faults, device effects, and external observation.

Common processing follows with a prepared call argument of zero.

### Common processing pass

At pass entry, the low byte of the current increment is captured. Later replacement of the increment does not change this captured value.

The first call is to **F-000000000001**, with the prepared signed 32-bit argument. Its selected target depends on control-flow history.

The returned 64-bit quantity supplies two numeric components. Through its low 32-bit component, the routine writes a 16-bit value equal to that component’s low 16 bits. It then multiplies the working word by 64 using unsigned 32-bit arithmetic and writes that resulting 32-bit value at the address it denotes.

Next, the routine reads a 32-bit word through the current pointer and retains it as the new increment. It writes a 16-bit value at working word plus four; the value is formed from the current pointer converted to a signed halfword, plus four, then narrowed for storage. Under a low-bit-preserving conversion model, this is the low 16 bits of the pointer value plus four.

Only after that read and halfword write does the routine replace the current pointer. The replacement is read from a pointer-sized slot addressed by converting the first result’s high component to signed 32-bit form and adding 80. The represented pointer slot is 32 bits. Conversion and signed-addition behavior remain unresolved where not representable.

The dependency order is then:

1. **F-000000000002**, without explicit arguments.
2. **F-000000000003**, without explicit arguments.
3. **F-000000000004**, receiving the word read through the previous current pointer.
4. **F-000000000005**, receiving that word logically shifted right by three.

No result from the first three calls is visibly used. Their memory effects are not known.

### Candidate construction and branch

The result of F-000000000005 is represented as 64 bits. Its high 32-bit component is retained for later address and value uses.

The candidate is formed by converting the result to signed 32-bit form, multiplying by 32 as signed arithmetic, and assigning the product to an unsigned 32-bit word. Therefore:

- A representable signed product can be described using the ordinary signed result followed by unsigned conversion.
- An overflowing signed product has no established result in this contract.
- Modulo-\(2^{32}\) multiplication is valid only under an additional machine interpretation that supplies wrapping signed multiplication.

The routine writes the candidate as a 32-bit word at the address represented by the candidate **before** comparing it, unsigned, with 20.

If the candidate is below 20, the extended path is skipped and another common pass begins with a zero prepared argument. The next F-000000000001 target differs from the initial-entry target.

For defined signed multiplication, or under the explicitly assumed 32-bit wrapping interpretation, the candidate is a multiple of 32. Under either condition, “below 20” means zero. This does not avoid address zero: the zero-valued word write there precedes the comparison. No such arithmetic conclusion is promised for unresolved signed-overflow execution.

### Extended processing

For a candidate at least 20, the routine first writes the working word as a 32-bit word at candidate plus 88. It then writes a halfword derived from the replacement current pointer at working word plus 14.

The increment becomes the working word shifted left by two using unsigned arithmetic. The captured pass-entry byte is assigned to the named global destination. This supplies an 8-bit-derived value, but the global’s actual write width and assignment conversion are unresolved.

Next, **F-000000000006** receives the current pointer’s unsigned 32-bit representation logically shifted right by two. **F-000000000007** follows without explicit arguments and returns a represented 64-bit quantity.

The latter result’s high component becomes the separate write destination. Its low component supplies another address. The following memory effects occur in order:

| Effect | Width and qualification |
|---|---|
| Write the captured byte at twice the signed 32-bit conversion of the low-component address | 8-bit store; signed doubling can overflow. |
| Write the high component’s low byte at low-component address plus eight | 8-bit store. |
| Write F-000000000005’s retained high component at twice the current accumulator | 32-bit pointer-valued store in the represented address model; address doubling is unsigned. |
| Write F-000000000007’s complete result through its low-component address | 64-bit store; byte layout depends on endianness. |

The accumulator is then replaced by the retained high component from F-000000000005, converted to signed 32-bit form and shifted left by two. The working word is replaced by the current pointer converted to signed 32-bit form and shifted left by eight. Neither signed shift has an unconditional wrapping guarantee.

Through the address supplied by F-000000000005’s retained high component, three consecutive 32-bit words are written, at offsets zero, four, and eight:

- The current increment.
- The newly computed working word.
- The current pointer’s unsigned 32-bit representation.

These stores follow the earlier 64-bit store; no non-overlap is established.

The replacement word becomes the increment multiplied by 256 using unsigned arithmetic.

### Restart selection and breakpoint continuation

The control value comes from an unresolved expression with this represented structure: convert the current pointer to signed 32-bit form, shift left by 31, shift that signed result right by 32, convert to unsigned 32-bit form, then logically shift right by 20.

The signed left shift and the width-sized right shift prevent a portable interpretation. In particular, the apparent right-shift count of 32 cannot simply be treated as producing zero or as being reduced modulo 32.

The base address is set to zero before the control value is tested.

- **Control value zero:** execution returns to the restartable write sequence. Its first three writes target address zero. Its separate accumulator destination remains the high component from F-000000000007.
- **Control value nonzero:** the control value is written as a 32-bit word at replacement word plus 32. The prepared call argument becomes the signed right shift by 29 of the replacement word interpreted as signed. The routine invokes **F-000000000008**, the software breakpoint operation, with immediate value 4.

If execution resumes after the breakpoint, the working word is replaced by a signed right shift by 18 of the separate write destination interpreted as signed. Common processing resumes with the previously prepared argument, rather than resetting it to zero.

Signed right shifts of negative values require the actual machine interpretation; sign-extending arithmetic shifts are not established merely by the signed operand type. Whether breakpoint continuation is reachable is also unresolved.

## Side effects

### Memory footprint and ordering

The footprint includes argument-provided storage, fixed global locations, an additional global destination, and addresses constructed from inputs, hidden state, and dependency results.

Only explicitly typed accesses have established widths. In particular:

- Assignments at `0x28` and `0x2C` are not established as 32-bit stores.
- Assignment of the captured byte to the other global is not established as an 8-bit store.

Unknown global widths may also affect whether those assignments overlap other storage.

The supplied representation establishes sequential operation order, not hardware atomicity, barriers, device semantics, or cross-thread visibility. Distinct stores may overlap each other or subsequent reads.

### Ownership and lifetime

No visible operation establishes allocation, ownership acquisition, ownership transfer, or release.

Pointer-like results do not establish valid storage or ownership. Dependencies may mutate globals or pointed-to storage, and their ownership effects are unknown. Absence of an explicit release does not establish a leak.

### Errors and partial execution

There is no visible error return, recovery path, or rollback. Invalid addresses, alignment faults, unresolved arithmetic, nonreturning dependencies, and breakpoint handling can prevent later events.

An interrupted execution can leave any completed prefix of side effects. The routine does not perform all validation before mutation.

## Dependencies

The opaque IDs identify call roles, not proven external API identities.

| ID | Represented interface and use |
|---|---|
| **F-000000000001** | One signed 32-bit argument; 64-bit result used for a halfword destination and a pointer-slot address. |
| **F-000000000002** | No explicit arguments; result not visibly consumed. |
| **F-000000000003** | No explicit arguments; follows F-000000000002. |
| **F-000000000004** | Receives the unsigned 32-bit word read through the previous current pointer; result not visibly consumed. |
| **F-000000000005** | Receives that word logically shifted right by three; 64-bit result supplies candidate construction and retained high-component storage. |
| **F-000000000006** | Receives the current pointer’s unsigned representation logically shifted right by two; result not visibly consumed. |
| **F-000000000007** | No explicit arguments; 64-bit result supplies addresses, stored bytes, and a complete 64-bit store. |
| **F-000000000008** | Software breakpoint with immediate value 4. |

F-000000000001 has history-dependent target selection: initial entry, continuation after a below-threshold candidate, extended-path restart, and breakpoint continuation use different represented target constructions.

Actual ABI details, executable target validity, storage validity, ownership effects, and global preservation are unresolved for all ordinary dependencies.

## Acceptance examples

These are conditional static expectations, not executed tests.

### Entry and overlapping writes

For input word one, unsigned arithmetic produces working word six, base address `0x01000000`, increment `0x02000000`, and replacement word `0x10000000`.

After the earlier entry effects complete, the restartable sequence attempts an 8-bit zero write, a 16-bit write of six, and a 32-bit write of `0x10000000` at that base. It then writes the updated accumulator there as a separate 32-bit operation.

The preceding assignments at `0x28` and `0x2C` supply `0x40` and zero, respectively; this example does not assign them access widths.

### Overflow-dependent zero candidate

Suppose F-000000000005 has low component `0x08000000`. Its signed multiplication by 32 exceeds the signed 32-bit range.

**Only if the selected machine interpretation supplies 32-bit wrapping signed multiplication**, the candidate becomes zero. Under that interpretation, the routine attempts a 32-bit zero write at address zero before selecting another common pass. F-000000000006 and F-000000000007 are skipped for that pass.

Without that interpretation, this input has no established candidate or branch outcome.

### Representable nonzero candidate

For a result whose signed 32-bit conversion is one, multiplication by 32 is representable. The candidate is 32.

The self-address write occurs at address 32 before the extended-path write at address 120. A fault at the first write prevents the later effect.

### Captured value versus global width

If the increment’s low byte is `0xA5` at pass entry and the subsequent pointer read yields a word ending in `0x3C`, the extended path assigns the captured value `0xA5` to the global destination, subject to its unknown destination conversion and width.

The later explicit 8-bit store at the doubled F-000000000007-derived address also uses `0xA5`, not `0x3C`, if its signed address calculation is resolved and the store completes.

### Zero-control restart

If the unresolved control expression evaluates to zero under the actual machine semantics, the next restartable sequence targets address zero with its byte, halfword, and word writes, in that order. The accumulator write instead uses the separate destination retained from F-000000000007.

## Uncertainties

- **documented_only — entry state:** Origins and required initialization of the hidden pointer and mask.
- **documented_only — global declarations:** Access widths, destination types, conversions, and overlap for the three global assignments.
- **documented_only — signed arithmetic:** Pointer conversions, signed additions, candidate multiplication by 32, other signed doubling, and signed left shifts where not defined by ordinary integer rules.
- **documented_only — control expression:** Meaning of the signed left shift by 31 followed by the signed right shift by 32.
- **documented_only — signed right shifts:** Treatment of negative operands in breakpoint argument preparation and continuation.
- **documented_only — memory model:** Address validity, pointer reconstruction, alignment, endianness, atomicity, and device behavior.
- **documented_only — dependencies:** Actual targets, ABI, result semantics, memory effects, ownership, and return behavior.
- **documented_only — breakpoint:** Trap disposition and reachability of the represented continuation.
- **Inferred intent:** The routine appears dependent on low-level execution state. No particular subsystem or valid memory map is established.

## Evidence

This contract uses only the supplied static representation: argument types, explicit casts, typed dereferences, assignments, calls, arithmetic, comparisons, and control-flow relationships.

Explicit dereference types support the stated access widths. Missing global declarations do not support equivalent width claims for global assignments. Likewise, the signed operation preceding assignment to the unsigned candidate prevents an unconditional modular-multiplication claim.

No implementation access, runtime execution, or tests were used. Successful execution, normal completion, dependency contracts, and a high-level purpose remain unestablished.
