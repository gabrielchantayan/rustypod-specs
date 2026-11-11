<!-- Trusted provenance: F-29ac7f46aa9f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-29ac7f46aa9f

## Purpose

F-29ac7f46aa9f is a stateful routine that performs computed-address memory accesses, assignments to undeclared destinations, dependency calls, conditional breakpoint handling, and repeated state transitions. The supplied static representation does not establish an application-level purpose.

Execution has two regions:

- A first region repeatedly selects between an initialization-like path and a byte-output path. Only a conditional exit from the byte-output path reaches the second region.
- A second region repeats without a represented normal exit.

“Initialization-like,” “seed,” and other descriptive labels summarize use, not application semantics. They do not establish allocation, initialization guarantees, device control, or ownership.

This contract describes static evidence and explicitly identifies unresolved machine behavior. No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width or type | Established use |
|---|---|---|
| First argument | 32-bit value | No explicit use is shown. |
| Byte destination | Pointer to an 8-bit location | Receives the selector’s low byte; later replaced by a result’s upper 32 bits. |
| Selector | Signed 32-bit integer | Controls first-region selection and supplies the byte-output value. |
| Address seed | Unsigned 32-bit integer | Supplies computed addresses and later becomes carried state. |

The explicit arguments are not a complete independent calling contract. Execution also depends on incoming register-like values, a stack-like base, a link value, an overflow condition, globals, and accessible memory.

The following descriptive state names are used:

| State | Represented role |
|---|---|
| Stack base | Address used for stack-adjacent stores and a pointer comparison. |
| Base | Unsigned 32-bit value used in address arithmetic. |
| Stored value | Unsigned 32-bit value used in stores and address formation. |
| Offset | Signed 32-bit value used in computed accesses. |
| Accumulator | Signed 32-bit carried value. |
| Increment | Signed 32-bit value added on the byte-output path. |
| Comparison pointer | Incoming pointer compared with stack base plus 20. |
| Held pointer | Pointer established on the byte-output path and replaced in the second region. |
| Working value | Unsigned 32-bit carried value. |

These names do not imply separate storage, nonaliasing, or ownership.

### Outputs and execution preconditions

No return value or normal return path is represented. Observable outcomes consist of completed accesses, assignments, calls, a possible breakpoint, and any resulting faults or dependency effects.

Explicitly typed accesses establish 8-, 16-, 32-, and 64-bit widths where stated below. Results from several dependencies are represented as 64-bit quantities split into lower and upper 32-bit halves. Address calculations use a represented 32-bit model; actual integer-to-pointer behavior remains unresolved.

An assignment’s right-hand-side width does not establish its destination’s width. In particular:

- Several global destinations have undeclared widths.
- The destination associated with F-422afe36d401 is undeclared. Its assigned value is the loaded value’s low 16 bits, but its access width and resulting memory footprint are not established.

No bounds, null, permission, or alignment checks are visible. Successful continuation requires executed accesses to be valid and reached dependencies to permit continuation. No ownership transfer, allocation, or release contract is established.

## Behavior

### First-region selection

At each entry to the first-region loop, the selector is shifted left by one. The initialization-like path is selected if the displayed shifted result is zero, or if its negative condition differs from the Boolean interpretation of the overflow condition. Otherwise, the byte-output path is selected.

This condition cannot safely be replaced by an ordinary signed comparison. Signed left-shift behavior and the separate overflow state require machine-specific interpretation.

### Initialization-like path

The routine selects a future stack base four bytes below the current stack base and writes the stored value there using a 32-bit store. The carried stack base is not updated until the shared continuation finishes.

Subsequent effects occur in this order:

1. Multiply the seed by eight and perform a 32-bit store of that product at the address numerically equal to the product.
2. Perform a 32-bit zero store at an address formed by shifting the seed left by 17, interpreting the result as signed, shifting right by 32, and adding 16. The oversized right shift leaves the actual address unresolved.
3. Perform a 16-bit store of `0xc470`. Its address is the stored value logically shifted right by four, minus sixteen times the base, minus one when bit three of the stored value is zero, plus 38.
4. Assign `0x8000` to the undeclared global at `0xfe238028`.
5. Set the seed to `0x0897f228`.
6. Assign the address `0x225fc470` to the undeclared global at that address.
7. Assign `0x225f` to the undeclared global at `0x0000229f`.
8. Read the undeclared global at `0x00000022`, shift its value right by 16, and store the result as 32 bits at the current stack base plus 84.

The global read’s width and signedness are unavailable. The three global assignments’ widths are also unavailable.

If the comparison pointer differs from the current stack base plus 20, F-U01 is called through the current link plus `0x2e`. The represented link is set to `0x0897f193` before invocation. No explicit arguments are displayed.

After the optional call returns, the represented state is established as follows:

- Accumulator: zero.
- Result lower half: zero.
- Result upper half: `0x0897f1f8`.
- Offset: seed plus five, represented as `0x0897f22d`.
- Base: `0x0897f1f8`.

The selected future stack base remains four bytes below the current base. Execution proceeds to the shared continuation.

These represented assignments do not establish that F-U01 preserves globals, flags, resources, or unrelated memory.

### Byte-output path

The first memory effect is an 8-bit store of the selector’s low byte at the byte destination.

Before calling F-U02:

- The working value becomes 8.
- A comparison byte becomes zero.
- The call target is formed from the previous link plus `0xe56`.
- The represented link becomes `0x0897f1d7`.

F-U02 receives an explicit zero argument and produces a represented 64-bit result.

After a returning call, the routine:

1. Adds the increment to the accumulator.
2. Loads an unsigned 16-bit value at base plus 30 and converts its zero-extended value directly into the held pointer.
3. Loads an unsigned byte at the result’s lower-half address plus the base; that byte becomes the seed.
4. Selects the unchanged stack base as the future stack base.
5. Exits the first region if the zero comparison byte equals the represented overflow condition.

Otherwise, it enters the shared continuation. A direct exit carries working value 8 into the second region; taking the continuation replaces that value.

Signed accumulator overflow is not given portable semantics by the representation.

### First-region shared continuation

Both nonexiting paths perform these ordered operations:

1. Replace the byte destination with the result’s upper 32 bits.
2. Replace the working value with the result’s lower 32 bits.
3. Store the current offset as 32 bits at the lower-half address plus 12.
4. Interpret the lower half as signed and shift it right by a count taken from its low eight bits.
5. Replace the overflow condition with the represented signed-subtraction-overflow calculation for that shifted value minus one.
6. Load 32 bits from twice the shifted value.
7. Set the stored value to `0x0897f1ec` and the selector to `0x112fe3d8`.
8. Assign the loaded value’s low 16 bits to the undeclared destination associated with F-422afe36d401.
9. Store the current base as 32 bits at the loaded value’s address plus four.
10. Commit the selected future stack base.

Step 8 is an assignment, not a call. The right-hand-side truncation establishes the assigned low-16-bit value only. It does not establish a 16-bit destination access, a two-byte footprint, or preservation of adjacent bytes.

A completed initialization-like continuation decreases the carried stack base by four. A completed byte-output continuation leaves it unchanged. Subsequent initialization-like passes use `0x0897f1ec` as the stored value.

The right-shift count can exceed 31. Shift behavior and address arithmetic affected by overflow remain **documented_only** where machine semantics are unavailable.

### Transition into the second region

After the first-region exit, the routine loads an unsigned 16-bit value at the latest result’s lower-half address plus the current offset. It zero-extends that value into an address and stores the accumulator there as 32 bits.

The initial second-region iteration argument is zero. The held pointer, base, accumulator, seed, and working value carry forward.

### Second-region threshold operation

Each iteration retains the seed’s low byte and calls F-U03 with the iteration argument.

Call the result halves the **first lower address** and **first upper address**. The routine multiplies the first lower address by 32, then performs a 32-bit store of the product at the address numerically equal to that product. This store precedes the unsigned comparison of the product with 20.

If the product is below 20:

- The extended path is skipped.
- The shared-tail argument is zero.
- Base, accumulator, and working value retain their entry values.

Under wrapping 32-bit multiplication, the product is a multiple of 32, so a value below 20 must be zero. This does not make the preceding address-zero store valid.

### Extended second-region path

For a product at least 20, the routine performs these effects in order:

1. Store the current base as 32 bits at product address plus 88.
2. Store the held pointer’s low 16 bits at base plus 14.
3. Set the working value to the prior base shifted left by two.
4. Assign the retained seed byte to an undeclared destination called the **reset location**.
5. Call F-U04 with the held-pointer integer logically shifted right by two.
6. Call F-U05 without displayed arguments.

The reset location’s address and access width are unresolved.

Call F-U05’s result halves the **later lower address** and **later upper address**. The routine then:

1. Stores the retained byte at twice the later lower address.
2. Stores the later upper address’s low byte at later lower address plus eight.
3. Stores the first upper address as a represented 32-bit pointer at twice the current accumulator.
4. Stores the complete represented 64-bit F-U05 result at the later lower address.
5. Sets the accumulator to four times the first upper address.
6. Sets the base to the held-pointer integer shifted left by eight.
7. Performs three consecutive 32-bit stores through the first upper address:

| Byte offset | Value |
|---:|---|
| 0 | Working value: prior base shifted left by two |
| 4 | Held-pointer integer shifted left by eight |
| 8 | Held-pointer integer |

The accesses may overlap. Their order must not be collapsed into an unordered aggregate operation.

The branch expression is formed by interpreting the held pointer as signed, shifting it left by 31, shifting that result right by 32, converting to unsigned, and logically shifting right by 20. Its actual result is unresolved because of the signed and oversized shifts.

**When the expression is zero:**

- Assign zero to the reset location, with destination width still unresolved.
- Add the working value to the accumulator.
- Store the updated accumulator as 32 bits at the later upper address.
- Select zero as the shared-tail argument.
- Retain the base established from the held pointer shifted left by eight.

**When the expression is nonzero:**

- Store the expression as 32 bits at working value multiplied by 256, plus 32.
- Prepare the shared-tail argument by interpreting working value multiplied by 256 as signed and shifting right by 29.
- Execute the represented software breakpoint with immediate value 4.
- Only if execution resumes, set the base to the signed later upper address shifted right by 18 and proceed to the shared tail.

On the nonzero branch, the accumulator remains four times the first upper address.

### Path-dependent shared tail

The represented call target depends on the preceding path:

| Path | Target calculation | Argument |
|---|---|---|
| Below threshold | `0x0897edf1` plus `0xa1e` | Zero |
| Extended, zero expression | `0x0897ee1d` plus `0xa1e` | Zero |
| Extended, nonzero expression after breakpoint resumption | `0x08790d52` plus `0xa1e` | Signed-shift-derived argument |

After the selected call returns, the routine:

1. Stores the result’s low 16 bits at its lower-half address using an explicit 16-bit access.
2. Stores base multiplied by 64 as 32 bits at the address numerically equal to that product.
3. Loads 32 bits through the old held pointer.
4. Stores the low 16 bits of held-pointer integer plus four at base plus four.
5. Replaces the held pointer with the represented 32-bit pointer loaded at the tail result’s upper-half address plus 80.
6. Calls F-U07, then F-U08, without displayed arguments.
7. Assigns the current working value to the seed.
8. Calls F-U09 with the earlier value loaded through the old held pointer.
9. Sets the next iteration argument to that earlier value logically shifted right by three.
10. Replaces the working value with that earlier value.

The replacement pointer does not supply the next iteration argument. The seed assignment precedes F-U09; the working-value replacement follows it. Execution then repeats.

## Side effects

Established effects include explicit-width memory accesses, stack-adjacent stores, computed-address stores, global assignments, dependency calls, and a conditional breakpoint.

The assignment associated with F-422afe36d401 supplies a low-16-bit value to an undeclared destination. Its destination width and resulting memory footprint remain unresolved. It must not be classified as an established 16-bit memory write.

No validation, synchronization, rollback, allocation, or release is represented in this routine. These observations do not constrain its dependencies. No pointer ownership transfer is established.

Dependencies may affect accessible memory, globals, flags, resources, or control flow. Later reads concern then-current memory; preservation across calls is not guaranteed. Faults may interrupt execution after earlier effects have occurred. No recovery path is represented.

## Dependencies

The F-U identifiers are specification-local opaque call-site identifiers, not resolved function identities.

| ID | Established interaction |
|---|---|
| F-422afe36d401 | Associated undeclared destination receives the loaded value’s low 16 bits; destination width and footprint unresolved; no invocation shown. |
| F-U01 | Optional call through previous link plus `0x2e`; no displayed arguments. |
| F-U02 | Previous link plus `0xe56`; zero argument; represented 64-bit result. |
| F-U03 | Target `0x07f755f6`; iteration argument; represented 64-bit result. |
| F-U04 | Target `0x0897edf1` plus `0xae`; held-pointer integer logically shifted right by two. |
| F-U05 | Opaque fixed code-address anchor plus `0xffe`; no displayed arguments; represented 64-bit result. |
| F-U06A/B/C | Three path-dependent shared-tail targets listed above; represented 64-bit results. |
| F-U07 | Target `0x0897edcd` plus `0x3c4`; no displayed arguments. |
| F-U08 | Target `0x0897eddf` plus `0xffe`; no displayed arguments. |
| F-U09 | Target `0x0897ede1` plus `0x800`; earlier old-held-pointer load as argument. |

Calling conventions, hidden inputs, clobbers, ownership effects, termination, and result validity are unresolved.

## Acceptance examples

These are conditional static obligations, not executed tests. Each assumes the specified path is reached and earlier operations permit continuation.

1. **Byte-output ordering.** With selector `0x12345678`, the byte-output path stores `0x78` before F-U02. A direct first-region exit carries working value 8.

2. **Stack commitment.** With stack base `0x1000`, a completed initialization-like continuation commits `0x0ffc`; a completed byte-output continuation commits `0x1000`. A fault before commitment does not establish the later state.

3. **Initialization address arithmetic.** With stored value `0x80` and base 3, the 16-bit destination is 8 minus 48 minus 1 plus 38, or `0xfffffffd` under wrapping 32-bit arithmetic. The attempted value is `0xc470`.

4. **Known initialization continuation.** The represented lower result half is zero, so the continuation attempts a 32-bit store of `0x0897f22d` at address 12. A later loaded-address-plus-four store contains `0x0897f1f8`.

5. **Truncation without destination-width inference.** If the continuation’s 32-bit load yields `0x1234abcd`, the assignment associated with F-422afe36d401 supplies `0xabcd`. Acceptance must not require exactly two affected bytes or unchanged neighboring bytes: the destination declaration and footprint are unavailable.

6. **Threshold-store ordering.** A first lower address of zero causes an attempted 32-bit zero store at address zero before the below-threshold decision. A fault there prevents an obligation to reach the tail.

7. **Extended layout.** With prior base 3, held pointer `0x40`, and valid noninterfering destinations, the three stores through the first upper address contain 12, `0x4000`, and `0x40`.

8. **Old versus replacement pointer.** If the old held pointer yields `0x40` and the offset-80 load supplies replacement pointer `0x9000`, the next iteration argument is 8, not `0x1200`. The working value becomes `0x40` after F-U09.

9. **Seed recurrence.** A below-threshold iteration entered with working value 8 assigns seed 8 before F-U09. If execution continues as represented, the next iteration retains byte `0x08`.

10. **Breakpoint boundary.** On the nonzero extended branch, the computed store precedes breakpoint 4. The subsequent base assignment and shared-tail call require breakpoint resumption.

## Uncertainties

The following remain **documented_only**:

- Architecture, instruction mode, ABI, and complete entry-state requirements.
- Oversized shifts, signed overflow, address wrapping, and integer-to-pointer conversions.
- Overflow-condition fidelity and dependency effects on machine state.
- Undeclared global widths, source signedness, and reset-location identity.
- The destination type, access width, and memory footprint of the assignment associated with F-422afe36d401.
- Whether that assignment affects executable storage.
- Dependency identities, contracts, hidden arguments, ownership effects, and termination.
- Fidelity of represented 64-bit results and accesses.
- Memory validity, aliasing, alignment, permissions, byte order, and atomicity.
- Hardware-specific effects, concurrent visibility, and instruction-cache requirements.
- Breakpoint disposition and executable fidelity of the represented nontermination.

These unknowns do not erase explicit ordering or state transitions. Conversely, explicit right-hand-side truncation does not resolve an undeclared destination’s access width.

## Evidence

**Established static evidence:** the supplied representation shows argument widths, explicit access types, constants, offsets, assignments, call-target calculations, the first-region exit, breakpoint, and second-region back edge. The assignment associated with F-422afe36d401 explicitly truncates its supplied value to 16 bits but does not declare its destination type.

**Inferred intent:** descriptive state names and the “initialization-like” label summarize usage only. No application purpose, allocation policy, or ownership model is established.

**Unknown:** complete machine semantics, undeclared destination layouts, dependency behavior, memory validity, and runtime outcomes.

No runtime tests have been performed. Acceptance examples are conditional consequences of static evidence.
