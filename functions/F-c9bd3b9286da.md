<!-- Trusted provenance: F-c9bd3b9286da; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c9bd3b9286da

## Purpose

F-c9bd3b9286da conditionally prepares resource-valued fields in a mutable object, optionally supplies configuration data to selected resources, and conditionally creates an associated resource.

The established contract consists of field accesses, local initialization writes, dependency calls, and explicit return paths. “Resource,” “configuration,” and “associated resource” are descriptive roles, not established external types. Allocation, locking, clearing, and asynchronous execution remain inferred intent.

The explicit 32-bit return values are:

| Value | Established condition |
|---|---|
| 0 | Processing reaches normal completion |
| 1 | The enable byte is zero after the entry dependency returns |
| 2 | The first resource-producing call returns zero |

A return of 0 does not establish that all dependency operations succeeded. Later resource-producing results and the associated-resource result are stored without a visible failure check.

## Interface

### Arguments and result

The function accepts two 32-bit arguments:

1. **Object address:** used as the base for byte-offset field accesses and interior addresses passed to dependencies.
2. **Configuration argument:** its low eight bits are stored in the object on enabled paths; its complete 32-bit value independently determines whether configuration calls occur.

These uses are distinct. For example, `0x00000100` stores a zero byte but still requests configuration.

The result is a 32-bit status word. The explicit return contract assumes that the dependencies return and that the relevant memory accesses complete. Faults, exceptions, blocking, and nonreturning dependencies are not specified.

### Directly used object layout

All offsets are byte offsets from the object address.

| Offset | Width or use | Role in this function |
|---|---|---|
| `0x0A` | 8-bit read | First argument of ordinary resource-producing calls |
| `0x0B` | 8-bit read | Mode, read at several separate decision points |
| `0x1C` | 32-bit read | Resource-producing argument |
| `0x2C` | 32-bit read | Resource-producing argument |
| `0x30` | 32-bit read | Resource-producing argument |
| `0x3C` | Interior address | Configuration data for the first three resource slots |
| `0x40` | 8-bit read | Enable flag |
| `0x46` | 8-bit read | Auxiliary-resource flag |
| `0x47` | 8-bit write | Low byte of the configuration argument |
| `0x4C` | 32-bit read/write | Primary resource slot |
| `0x50` | 32-bit read/write | Secondary resource slot |
| `0x54` | 32-bit read/write | Third resource slot |
| `0x58` | 32-bit read/write | First auxiliary resource slot |
| `0x5C` | 32-bit read/write | Second auxiliary resource slot |
| `0x78` | Interior address | Argument to entry and exit dependencies |
| `0x94` | 32-bit read/write | Associated-resource slot |

The highest direct object access extends through offset `0x97`. This does not establish the complete required object size: dependencies receive interior addresses and may require additional storage.

### Local storage

Two distinct local areas are visible:

- A **16-byte scratch array**, comprising four 32-bit elements. Its first element receives a 32-bit zero write before every F-798fb9f4fe09 call. The same array is later passed to F-95f1655a5c87 and F-195f5d43c91b if associated-resource creation occurs.
- A **four-byte auxiliary configuration buffer**, comprising two 16-bit elements. Only its first element is explicitly initialized, to `0xFFFF`, before auxiliary-slot configuration.

Neither area is established to be fully initialized by this body.

### Preconditions and arithmetic

The object fields used on the selected path must be accessible with appropriate permissions and target alignment. Dependency-specific argument validity requirements are unknown.

No explicit checks establish object validity, object extent, supported modes, or resource validity. Address formation uses byte displacements from the 32-bit address-like argument; behavior for overflowing address calculations or invalid resulting addresses is not established.

No resource-size arithmetic is visible. Truncating the configuration argument to eight bits is distinct from its full-width zero comparison. The literal `0x20` passed during associated-resource creation has no established unit or size meaning.

## Behavior

### Entry and disabled path

The first dependency call is F-338d16f9ce72, receiving the interior address at `0x78`.

After it returns, the function reads the enable byte at `0x40`. An entry value observed before that dependency call would not determine this branch.

If the byte is zero, the function calls F-1333331b2da1 with the interior address at `0x78` and returns 1. This body performs no configuration-byte or resource-slot writes on that path. The dependencies may nevertheless change object or external state.

### Enabled-path configuration byte

If the enable byte is nonzero, the function stores the configuration argument’s low eight bits at `0x47`.

This write occurs before resource preparation and is not rolled back if the first resource-producing call fails.

### Resource preparation

The function reads the mode byte at `0x0B`. If it equals 3, this entire preparation phase is skipped: no preparation-phase slot clearing, scratch initialization, or F-798fb9f4fe09 call occurs.

For every other mode value, the function writes 32-bit zero values to `0x50`, `0x54`, `0x58`, and `0x5C`, in that order. It does not first clear `0x4C`, and no release operation for overwritten values is visible.

#### Common resource-producing call contract

An ordinary F-798fb9f4fe09 call receives eight arguments, in positional order:

1. The 8-bit field at `0x0A`.
2. The 32-bit field at `0x2C`.
3. The 32-bit field at `0x1C`.
4. The 32-bit field at `0x30`.
5. The integer 4.
6. Zero.
7. Zero.
8. Zero.

Auxiliary calls use the constant 4 in the first position instead of reading `0x0A`; the other positions follow the same pattern.

**Before every such call that actually occurs, the body writes a 32-bit zero to the first element of the 16-byte scratch array.** This includes the primary, secondary, third, and both auxiliary calls. These are repeated initialization writes, not a single initialization on entry.

The scratch address is not among this call’s eight explicit arguments. The writes do not establish initialization of the other three scratch elements or imply a scratch-buffer contract for F-798fb9f4fe09.

Object-derived arguments are obtained anew for each call. No ordering among separate argument-expression reads within one call is established.

#### Primary result and early failure

After the four slot-clearing writes, the function performs the scratch first-element zero write and makes the first ordinary resource-producing call.

Its 32-bit result is stored at `0x4C`. If that returned result is zero, the function immediately returns 2.

On this path:

- The configuration byte has already been written.
- The four other resource slots have already received zero writes.
- The primary slot receives zero.
- Configuration and associated-resource processing are skipped.
- F-1333331b2da1 is not called.

Whether the omitted exit call represents an intentional protocol or a defect is unknown.

#### Secondary and third results

After a nonzero primary result, the mode byte is read again.

If this read yields 3, no primary-result copy or further resource-producing call occurs in this phase.

Otherwise, the saved primary return value is written to `0x50`. This is a copy of the returned value, not a fresh read from `0x4C`.

If that same mode read is neither 1 nor 2, preparation ends after the copy. If it is 1 or 2, the function resets the scratch array’s first 32-bit element to zero, makes a second ordinary resource-producing call, and stores its result at `0x50`.

A zero second result does not trigger status 2. At this step, the body stores the result only at `0x50`; it does not itself overwrite `0x4C`. F-798fb9f4fe09 may independently mutate the primary slot or affect the validity or ownership of its value. Those effects are unknown.

After the second call and result store, the mode byte is read again. Only if this read yields 2 does the function reset the scratch first element, make a third ordinary resource-producing call, and store its result at `0x54`.

#### Auxiliary results

After storing the third result, the function reads the auxiliary-enable byte at `0x46`.

If it is nonzero, two auxiliary resource-producing calls occur in sequence:

- Reset the scratch first element, call with first argument 4, and store the result at `0x58`.
- Reset the scratch first element again, call with first argument 4, and store the result at `0x5C`.

Neither result is checked. Once this branch is entered, the second auxiliary call does not depend on the first result or a further mode or auxiliary-flag read.

### Optional configuration

Following preparation, including the mode-3 skip, the function tests the original full-width configuration argument. If it is zero, configuration is skipped.

Otherwise, slots are considered in this order:

1. `0x4C`
2. `0x50`
3. `0x54`
4. `0x58`
5. `0x5C`

For each of the first three slots, a nonzero test leads to an F-8df7565e01e6 call with a value loaded from that slot and the interior address at `0x3C`.

After handling the first three slots and before testing the auxiliary slots, the function writes `0xFFFF` to the first 16-bit element of the four-byte auxiliary configuration buffer. Its second element has no visible initialization.

Each auxiliary slot that passes its nonzero test leads to an F-8df7565e01e6 call with its loaded value and the address of that same local buffer.

Consequences:

- Equal slot values are not deduplicated.
- Later slot inspections occur after earlier configuration calls return.
- Dependencies may change later slots or configuration data.
- The auxiliary buffer is initialized once, not again between its two possible calls.
- Configuration return values do not control the visible flow.
- Each slot’s condition and call argument involve separate reads. Under concurrent mutation, the nonzero test is not an atomic guarantee that the value supplied to the call remains nonzero or unchanged.

### Associated-resource creation

After optional configuration, the function reads the mode byte again. Only if that read yields 2 does it read the 32-bit associated-resource slot at `0x94`.

If the mode is not 2, or the slot is nonzero, associated-resource creation is skipped.

Otherwise, F-95f1655a5c87 receives:

1. The address of the same 16-byte scratch array used by the earlier explicit zero writes.
2. An opaque static descriptor address.
3. The object address.

There is no additional direct scratch initialization immediately before this call. If preparation ran, the established earlier writes initialized its first element before each resource-producing call; they did not initialize the whole array. If preparation was skipped but a later mode change permits associated-resource creation, no such earlier scratch write is established.

After F-95f1655a5c87 returns, the function calls F-195f5d43c91b with seven arguments:

1. The static label `GraniteLayerClear`.
2. A value read from an opaque static global cell.
3. The scratch-array address.
4. The integer `0x20`.
5. A value obtained by reading an opaque static pointer cell and dereferencing the resulting pointer.
6. Zero.
7. Zero.

Both the direct global-cell read and the pointer-cell-plus-dereference read occur after F-95f1655a5c87 returns. Its effects may therefore influence their values. No evaluation order is established among separate argument expressions, including these global reads.

The width of the indirect dereference is unresolved. The visible data also does not establish a precise declared type for the directly loaded global value.

The 32-bit result of F-195f5d43c91b is stored at `0x94`, including if it is zero. No failure branch follows that result. The constant `0x20` does not establish that the scratch array supplies 32 readable bytes.

### Normal exit

F-1333331b2da1 receives the interior address at `0x78`, after which the function returns 0.

## Side effects

Direct object writes are limited to the configuration byte, preparation-phase slot clearing, resource-result storage, the conditional primary-result copy, and conditional associated-result storage described above.

Direct local writes include every per-resource-call 32-bit scratch reset and the single 16-bit auxiliary-configuration initialization when configuration is enabled.

No rollback or explicit release of overwritten resource values is visible. Equal values may occupy multiple slots. That does not establish shared ownership, retained references, or any later release obligation.

All dependencies may have additional effects on object fields, globals, resources, or supplied memory. In particular:

- Entry and exit calls are not assumed to implement synchronization.
- Resource-producing calls are not assumed to preserve existing slots or resource validity.
- F-95f1655a5c87 is not assumed to preserve the global cells read afterward.
- Neither scratch-consuming dependency is assumed to initialize a particular extent, copy its input, or avoid retaining its address.
- Configuration dependencies are not assumed to preserve the auxiliary buffer between calls.

Both local buffers are stack-backed storage. Any retention beyond the function’s execution requires a lifetime contract absent from the evidence.

## Dependencies

| Opaque ID | Established use | Unknown semantics |
|---|---|---|
| F-338d16f9ce72 | Entry call with address at `0x78` | Synchronization, acquisition, validation, mutations |
| F-1333331b2da1 | Disabled-path and normal-exit call with address at `0x78` | Release or synchronization behavior |
| F-798fb9f4fe09 | Eight-argument call; 32-bit result stored in a resource slot | Allocation, ownership, validity, mutations, later failure conventions |
| F-8df7565e01e6 | Slot-derived argument after a nonzero test, plus configuration-data address | Data extent, mutation, retention, resource effects |
| F-95f1655a5c87 | Scratch address, static descriptor address, object address | Initialization extent, global effects, descriptor meaning, retention |
| F-195f5d43c91b | Label, global-derived values, scratch address, constants; stored 32-bit result | Scheduling, ownership, input extent, retention, failure semantics |

## Acceptance examples

These are conditional consequences of the static evidence, not executed tests. Stable-state examples assume dependencies and concurrent activity do not alter the fields on which the example depends.

1. **Disabled object:** If the enable byte is zero after F-338d16f9ce72 returns, the body calls F-1333331b2da1 and returns 1 without its own configuration-byte, resource-slot, or scratch-initialization writes.

2. **First resource failure:** For enabled processing with an initial mode other than 3, a zero first result produces status 2. The four secondary/auxiliary slots have received zero writes, the scratch first element has received one 32-bit zero write, and `0x4C` receives zero. No exit dependency call occurs.

3. **Stable mode 0:** With a nonzero first result, one resource-producing call and one scratch first-element reset occur. The saved result is written to both `0x4C` and `0x50`. With stable slots and nonzero configuration, that equal value is supplied in two separate configuration calls.

4. **Stable mode 1, zero second result:** Two resource-producing calls and two scratch resets occur. The second result is stored at `0x50` without an early failure return. The body does not overwrite `0x4C` at that store; the second dependency’s effects on the primary resource remain unknown.

5. **Stable mode 2, auxiliaries enabled:** With a nonzero first result and a nonzero auxiliary flag at its read point, five resource-producing calls occur, each preceded by its own scratch first-element reset. The final two calls use first argument 4. Later zero results do not themselves cause status 2.

6. **Stable mode 3:** Preparation performs no clearing, resource-producing calls, or scratch resets. Existing slots can still be configured if the full configuration argument is nonzero.

7. **Byte truncation:** `0x00000100` and zero both store zero at `0x47`; only the former enables configuration.

8. **Mode changes after the first call:** If the initial mode permits preparation but the next mode read yields 3, the first result is still stored and checked. The primary-to-secondary copy is skipped.

9. **Globals changed during scratch preparation:** If associated-resource creation is reached and F-95f1655a5c87 changes the relevant global cells, F-195f5d43c91b’s arguments are obtained through subsequent reads, not preserved pre-call snapshots. Their mutual evaluation order remains unspecified.

10. **Zero associated result:** When creation is attempted, a zero F-195f5d43c91b result is stored at `0x94`; normal exit still returns 0 if the exit dependency returns.

## Uncertainties

The following remain **documented_only**:

- Complete object type, extent, alignment, and external field meanings.
- Supported mode values beyond the visible comparisons.
- Synchronization semantics and the reason status 2 omits the exit call.
- Resource ownership, validity, aliasing, and release requirements.
- Dependency mutations of object fields, globals, resources, or local buffers.
- Exact dependency parameter types and argument-extension conventions.
- Width and interpretation of the indirect global load and precise type of the direct global value.
- Descriptor meaning and the meaning of `0x20`.
- F-95f1655a5c87’s scratch initialization extent and subsequent scratch contents.
- Buffer retention, asynchronous use, faults, exceptions, and concurrency behavior.

Repeated mode reads and explicit partial local initialization are established. An immutable mode snapshot, fully initialized buffers, or global preservation would be unsupported assumptions.

## Evidence

The supplied body establishes access widths, constants, direct writes, dependency-call boundaries, conditional branches, repeated mode reads, partial local initialization, post-F-95f1655a5c87 global reads, and three explicit return values.

Resource allocation, locking, and background clearing are inferred intent only. Dependency implementations and external type contracts are unavailable.

No runtime tests have been performed.
