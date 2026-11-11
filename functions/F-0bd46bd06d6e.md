<!-- Trusted provenance: F-0bd46bd06d6e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-0bd46bd06d6e

## Purpose

F-0bd46bd06d6e performs an unconditional, stateful repetition of indirect calls, memory reads, narrow and word-sized stores, address calculations, and a software breakpoint. The supplied representation has no ordinary return path.

No application-level purpose is established. In particular, the evidence does not identify initialization, device control, data conversion, or a named protocol. The contract therefore describes observable ordering and data dependencies without assigning those meanings.

All behavior below is established from static evidence unless explicitly marked as inferred or unknown. Continuing through the entire sequence is conditional on dependencies returning, memory accesses completing, and execution resuming after the breakpoint.

## Interface

### Inputs and completion

There are no explicit parameters and no explicit return value.

Execution nevertheless consumes implicit state:

| Contract name | Role |
|---|---|
| Entry link | Determines the first indirect-call target on initial entry. |
| Context base | Supplies the base for several word accesses and undergoes an explicit subtraction of 244. |
| Ambient base | Contributes to a late byte-store address. |
| Global A | Holds a 32-bit pointer read early in each iteration. |
| Global B | Holds a different 32-bit pointer read later in each iteration. |
| Secondary result channels | Supply values consumed after three particular calls. |

These are descriptive names, not established public interface names.

The function has no visible validation or initialization of these inputs. Successful execution requires accessible memory at every address actually used, callable dependency targets, and suitable behavior from the surrounding execution environment. These are conditions for completing the sequence, not checks performed by the function.

There is no represented success status, error status, normal termination, or cleanup path. A call that does not return, a memory fault, or breakpoint handling may prevent subsequent operations.

### Widths, layouts, and arithmetic

The represented accesses use 8-bit bytes, 16-bit halfwords, and 32-bit words and addresses.

**Three calls provide represented 64-bit paired results:** F-D05, F-D09, and F-D28. Each supplies a low 32-bit component and a high 32-bit component that the caller consumes separately. This does not establish the complete physical calling convention.

Unsigned halfword reads are zero-extended. Byte and halfword stores retain only the low 8 or 16 bits, respectively. Word stores retain 32 bits.

For the numerical descriptions below, address arithmetic, multiplication, and left shifts use the low 32 bits. Logical right shifts insert zeros. This expresses the apparent machine-width dataflow; it does not establish that overflowing signed operations would be defined in a corresponding high-level language. Exact architectural overflow, alignment, and fault behavior remain documented_only.

Offsets are byte offsets. No object sizes, field types beyond the individual accesses, byte order, or alignment guarantees are established.

## Behavior

### Call notation and state interpretation

F-D01 through F-D37 identify call occurrences in one iteration. These are opaque labels, not recovered public function identities.

The following occurrence pairs have equal target expressions after constant arithmetic:

- F-D05 and F-D06.
- F-D09 and F-D13.
- F-D17 and F-D24.
- F-D19 and F-D31.
- F-D25 and F-D32.

Equal targets do not establish equal inputs, outputs, or side effects.

A call with no listed arguments has no explicit arguments in the supplied representation. It is not thereby established to ignore incoming machine state.

“Captured” values below describe the represented caller dataflow. They do not assert that an unknown dependency preserves hardware registers, globals, or the context base. Context-relative accesses use the base available at the indicated point; preservation across intervening calls is not an additional guarantee.

### Initial calls and first stores

At iteration entry, a byte-sized working value is set to zero. F-D01 is then invoked with two explicit zero arguments at the address formed by adding `0xFFE` to the current entry link. A separate byte-sized operand is set to zero after F-D01 and before F-D02.

F-D02 and F-D03 execute in order. The function then reads Global A once, capturing pointer **A**, and invokes F-D04.

F-D05 supplies paired components **P** and **Q**, low word first. F-D06 receives P and the low 32 bits of Q + P. The separately initialized zero byte participates in the represented sum but contributes no numerical change.

After F-D06, the low byte of its secondary result is stored at A + `0x11`. F-D07 executes next, followed by a zero-byte store at A + `0x15`.

F-D08 then supplies a halfword result and a secondary 32-bit address component. The halfword is stored at that secondary component plus `0x0C`.

Both stores through A use the captured pointer rather than reloading Global A. Thus a later change to Global A does not, in the represented dataflow, redirect these stores. F-D07 occurs between them and may observe or alter memory during that interval.

### First context updates and breakpoint

F-D09 supplies paired components **R** and **S**. The function first stores a zero halfword at S + `0x2C`, then stores R logically shifted right by 26 bits as a word at context base + `0x26C`. The latter value is between 0 and 63.

The fixed value `0x21C48960` is established before F-D10. After F-D10, the function reads a 32-bit pointer **B** from context base + `0x320`, equivalent to decimal offset 800.

F-D11 and F-D12 execute in order. The low 32 bits of R shifted left by two are then stored at B + `0x54`. This uses the previously captured R and B, not new reads of their origins.

F-D13, F-D14, F-D15, F-D16, and F-D17 execute in that order. The function next reads an unsigned halfword from fixed address `0x21C48994`, obtaining **H**.

After F-D18, it reads a word from context base + `0x280` and passes that word to F-D19. The 32-bit result of F-D19 is **J**.

A software breakpoint with immediate `0x92` follows F-D19. All remaining behavior in the iteration requires execution to continue beyond this breakpoint. No handler, recovery path, or bypass is supplied.

H is captured before F-D18 and F-D19. Modifying its source location during those calls does not itself change the captured value.

### Indexed accesses and halfword stores

After the breakpoint, the function derives:

- **Index offset:** the low 32 bits of J shifted left by two.
- **Fixed index:** `0x21C48960` logically shifted right by 18 bits, then multiplied by four. Its value is `0x21C4`.

A 32-bit pointer **C** is read from H + `0x44`. H is the zero-extended halfword itself; there is no intervening pointer-sized load that replaces H as the address base.

F-D20 executes, followed by F-D21, whose 32-bit result is **T**. A word is then read from H + 4. F-D22 receives T and that word.

Global B is read once to capture pointer **D**, after F-D22 and before F-D23. Following F-D23, the low halfword of D is stored at D plus the index offset. The stored value is derived from the pointer bits, not from a separate object field.

F-D24 follows that store. A pointer **E** is then read from fixed index + `0x54`, which is fixed address `0x2218`.

F-D25 supplies a 32-bit result used as pointer **F**. The halfword `0x21C4` is stored at F + 4. F-D26 and F-D27 execute in order, after which an unsigned halfword **U** is read from E + `0x34`.

Neither the uses of H nor those of U imply a conventional pointer or index abstraction. Their narrow, zero-extended values are used literally in the stated calculations.

### Second context update and base transition

F-D28 supplies paired components **V** and **W**. The function stores V as a word at context base + 600, equivalent to offset `0x258`.

It next reads an unsigned halfword **X** from W + `0x14`. Two 32-bit values are derived:

- **Shifted value:** V shifted left by six, retaining 32 bits.
- **Scaled high word:** W multiplied by four, retaining 32 bits.

The context base is explicitly decreased by `0xF4`, or 244 bytes. This is a state adjustment, **not an additional explicit memory store**.

After the adjustment, the function forms **Scaled halfword = 4 × U** and stores V as a word at Scaled halfword + `0x74`. This destination is not context-relative.

Because U is unsigned and 16-bit, Scaled halfword ranges from 0 to 262,140. Adding `0x74` yields addresses from 116 to 262,256 without 32-bit overflow in this calculation.

The context-relative store of V precedes the base adjustment. No restoration of the context base appears before repetition.

### Final updates and repetition

F-D29 executes before X is stored as a 32-bit word at C + `0x54`. F-D30 then executes before a second word store of X at C + 100, equivalent to C + `0x64`. Both stores zero-extend the original halfword X.

F-D31 executes next. The low halfword of C is stored at the address formed by adding the shifted value to F-D31’s secondary 32-bit result.

After F-D32 and F-D33, the function reads a byte **Y** from U + Scaled halfword, equivalently 5 × U. This address ranges from 0 to 327,675.

F-D34 executes between that byte read and the byte store of Y at X + ambient base + `0x19`.

After F-D35, the scaled high word is stored as a word at C + `0x44`. F-D36 follows.

The entry-link working state is then assigned the fixed continuation value `0x08712031`. A word is read from scaled high word + `0x44` and passed to F-D37.

The function repeats unconditionally. Under the represented assignment, the following iteration’s F-D01 target is `0x0871302F`, rather than a target derived from the original incoming link. Whether the final dependency or execution environment alters that state is not established.

The explicit context-base subtraction is carried forward in the represented loop; no compensating restoration is shown. Additional dependency effects remain unknown.

## Side effects

### Direct-store inventory

A complete iteration contains **15 explicit memory stores: three byte stores, five halfword stores, and seven word stores**.

The following table lists them in execution order. Intervening calls and reads remain ordered as described above.

| Order | Width | Destination | Stored value |
|---|---:|---|---|
| 1 | 8 bits | A + `0x11` | Low byte of F-D06 secondary result |
| 2 | 8 bits | A + `0x15` | Zero |
| 3 | 16 bits | F-D08 secondary result + `0x0C` | F-D08 halfword result |
| 4 | 16 bits | S + `0x2C` | Zero |
| 5 | 32 bits | Context base + `0x26C` | R logically shifted right by 26 |
| 6 | 32 bits | B + `0x54` | R shifted left by two |
| 7 | 16 bits | D + index offset | Low halfword of D |
| 8 | 16 bits | F + 4 | `0x21C4` |
| 9 | 32 bits | Context base + 600 | V, before the base adjustment |
| 10 | 32 bits | Scaled halfword + `0x74` | V |
| 11 | 32 bits | C + `0x54` | Zero-extended X |
| 12 | 32 bits | C + 100 | Zero-extended X |
| 13 | 16 bits | Shifted value + F-D31 secondary result | Low halfword of C |
| 14 | 8 bits | X + ambient base + `0x19` | Y |
| 15 | 32 bits | C + `0x44` | Scaled high word |

Destinations may alias one another, globals, or locations subsequently read. No disjointness guarantee is established. A complete dependency-free memory summary cannot replace the sequence because dependencies may observe or modify memory between direct accesses.

### Other state and ownership effects

A completed iteration contains 37 call occurrences, one breakpoint operation, the explicit context-base adjustment, and the entry-link assignment. Effects inside dependencies are additional and unrestricted by this evidence.

There is no visible allocation, deallocation, reference-count adjustment, ownership transfer, or capacity check. Passing a pointer-like value does not establish borrowing, retention, ownership, or release by a dependency.

No explicit null check, bounds check, error-result branch, retry bound, or cleanup sequence appears. A zero or otherwise unusable result is still consumed by subsequent represented operations without a visible validity test.

## Dependencies

The caller consumes the following explicit inputs and results:

| Opaque occurrence | Established interaction |
|---|---|
| F-D01 | Two zero arguments; target depends on entry link |
| F-D05 | Supplies paired 32-bit components P and Q |
| F-D06 | Receives P and Q + P; secondary result supplies a byte |
| F-D08 | Supplies a halfword result and a secondary address component |
| F-D09 | Supplies paired 32-bit components R and S |
| F-D19 | Receives a context-loaded word; supplies J |
| F-D21 | Supplies T |
| F-D22 | Receives T and a word read from H + 4 |
| F-D25 | Supplies pointer F |
| F-D28 | Supplies paired 32-bit components V and W |
| F-D31 | Secondary result contributes to a halfword-store address |
| F-D37 | Receives a word read from scaled high word + `0x44` |

The remaining occurrences have no explicit arguments or directly consumed results beyond their place in the sequence. That absence does not constrain their implicit inputs or side effects.

No dependency is established to preserve globals, preserve the context base, return normally, validate addresses, or manage memory. The three paired-result interfaces and three secondary-result channels describe caller consumption, not complete ABI contracts.

## Acceptance examples

These are conditional static examples, not executed tests. Each assumes completion of the required calls and accesses, suitable dependency effects, and breakpoint resumption where necessary.

1. **Byte-store ordering.** If F-D06’s secondary result has low byte `0xA7`, A + `0x11` receives `0xA7` before F-D07. The zero store at A + `0x15` occurs after F-D07. F-D07 may therefore observe the first store before the second occurs.

2. **First paired-result arithmetic.** If F-D05 supplies P = `0xFFFFFFFF` and Q = 2, F-D06 receives first argument `0xFFFFFFFF` and second argument 1 under the stated 32-bit arithmetic model.

3. **Shift truncation.** If F-D09 supplies R = `0xFC000001`, context base + `0x26C` receives 63. The later store at B + `0x54` receives 4.

4. **Captured global pointer.** If Global A supplies A₁ and F-D04 subsequently replaces Global A with A₂, the represented direct stores through A still address A₁ + `0x11` and A₁ + `0x15`. This does not constrain other effects of F-D04.

5. **Pointer-bit halfword store.** If D is `0x12345678` and J is 3, the halfword at D + 12 receives `0x5678` after F-D23. The caller performs no separate lookup to obtain that stored value.

6. **Fixed-index calculation.** The fixed index is `0x21C4`; consequently, E is loaded from `0x2218`, and the halfword stored at F + 4 is `0x21C4`.

7. **Unsigned addressing boundary.** If U is `0xFFFF`, Scaled halfword is 262,140. The V-store address is 262,256, and the Y-read address is 327,675. U is not treated as −1.

8. **Context transition.** If the context base remains B₀ from the V store until the explicit adjustment, V is stored at B₀ + 600 and the adjustment produces B₀ − 244. The adjustment itself adds no memory store. Without further dependency changes, the next iteration uses the decreased base.

9. **Breakpoint interruption.** If breakpoint handling does not resume this execution, no operations described after the breakpoint are required to occur. In particular, the later global load and indexed stores need not happen.

## Uncertainties

### Established static evidence

The representation establishes the explicit access widths and offsets, three paired results, direct-store inventory, caller-visible dataflow, call order, breakpoint immediate, context-base subtraction, entry-link assignment, and unconditional repetition.

### Inferred intent

The repeated calls and structured offsets may belong to a larger stateful protocol. No particular protocol, object model, or device interpretation is sufficiently supported to become a requirement.

### documented_only

The following remain unresolved:

- The public calling convention and meanings of implicit machine-state inputs.
- The architectural identity and validity of the context base.
- The physical ABI, validity, and lifetime of paired and secondary results.
- Dependency behavior, including register preservation, global mutation, ownership, and nonlocal control transfer.
- Memory mapping, permissions, byte order, and alignment requirements.
- Whether accesses involve volatile memory, devices, atomicity, or concurrent mutation.
- Exact underlying overflow and fault behavior.
- Breakpoint handling and whether normal execution resumes.
- The meanings of fixed addresses and low-address calculations.
- Whether the represented loop and continuing context-base movement correspond to intended executable behavior.

These unknowns must not be replaced by assumed preservation, success, validation, or ownership rules.

## Evidence

This contract uses only the supplied static representation of F-0bd46bd06d6e. The operation sequence supports fifteen direct stores, including seven word stores; the context-base adjustment is separate state arithmetic. Three call results are represented as 64-bit values with separately consumed low and high words.

No implementation access, runtime execution, smoke checks, or tests were performed. The acceptance examples are conditional consequences of static dataflow, not observations of a running system.
