<!-- Trusted provenance: F-3dc3f1e45fdd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3dc3f1e45fdd

## Purpose

F-3dc3f1e45fdd is a non-returning memory-and-call sequence whose later effects depend on entry state, memory contents, and dependency results. Its observable contract includes narrow and wide stores, indirect calls, repeated processing, and a conditional software breakpoint.

The available static evidence does not establish an application-level purpose. In particular, initialization, device control, decoding, allocation, and reset handling are not established meanings of the operations.

This contract describes represented behavior, not demonstrated execution. Arithmetic, memory access, or dependency behavior may prevent completion of any described phase. No runtime tests have been performed.

## Interface

### Explicit arguments

There are nine represented arguments:

| Position | Width and interpretation | Observable relevance |
|---|---|---|
| 1 | Pointer to four-byte elements | Its contents provide the first store’s destination and data. Its address advanced by four bytes contributes to an early read and call argument. |
| 2, 3 | 32 bits each | No direct use is shown. |
| 4 | Unsigned 32 bits | Contributes to the initial dependency argument; replaced by a constant when the entry offset is nonzero. |
| 5, 6 | 32 bits each | No direct use is shown. |
| 7 | 32 bits | Its local value is replaced by zero without a later shown use. |
| 8 | 32 bits | No direct use is shown. |
| 9 | Unsigned 32 bits | Its local value is replaced by a byte-read result without a later shown use. |

Replacing a local argument value does not establish a write into caller-owned argument storage.

### Additional entry state

The function also consumes state not accounted for by those arguments:

- A signed 32-bit base used for a store at base plus `0x14`.
- A signed 32-bit offset controlling an early branch and read address.
- A 16-bit value used only on the nonzero-offset branch.
- A signed 32-bit accumulator increment.
- A comparison pointer.
- A cursor represented as a pointer to four-byte elements.
- An executable continuation used by F-U01.

The acquisition and calling convention of this state are **documented_only** unknowns.

### Results and memory requirements

No ordinary return or explicit error result is shown. Dependencies that provide results are represented as returning 64 bits, divided into low and high 32-bit halves; their actual return ABI is unknown.

The represented address domain is 32 bits. Pointer loads and stores explicitly typed as such occupy four bytes. Numeric values described as addresses do not establish mapped memory or permission to access it.

Every executed access requires suitable addressability, permissions, and applicable alignment. No validation or recovery is shown. The function provides no allocation, release, ownership-transfer, or cleanup guarantee.

## Behavior

### Interpretation rules

Unsigned 32-bit operations use modulo-\(2^{32}\) arithmetic. Converting a value to 16 or 8 bits retains its low bits under the represented integer conversions.

Signed multiplication, addition, and left shifts must not be silently treated as modular. Signed right shifts and out-of-range unsigned-to-signed conversions require applicable implementation semantics. Two expressions explicitly shift a signed 32-bit quantity right by 32; neither has a portable result established here.

Memory access width follows an explicit access type, not the width of the value assigned. The following named storage lacks declarations in the supplied evidence:

| Storage | Established value relationship | Unresolved property |
|---|---|---|
| `0x0897f07a` | Receives the entry offset shifted left by 12, converted to 16 bits | Write width |
| `0xbf836004` | Receives `0x6000` | Write width |
| `0xc1b00050` | Supplies a value converted to unsigned 32-bit data | Read width and source type |
| `0xbf836002` | Receives the entry 16-bit value | Write width |
| `0x0897f454` | Supplies retained signed 32-bit state | Read width and source type |
| `0x00000009` | Receives zero | Write width |
| `0x00000030` | Receives a value converted to 16-bit zero | Write width |
| `0xfe238028` | Receives `0x8000` | Write width |
| `0x225fc470` | Receives its own address | Write width |
| `0x0000229f` | Receives `0x225f` | Write width |
| `0x00000022` | Supplies a value subsequently shifted right by 16 | Read width, source type, and shift interpretation |
| Storage identified by F-422afe36d401 | Receives a loaded value converted to 16 bits | Write width |
| `0x0897f216` | Supplies a value converted to unsigned 32-bit cursor state | Read width and source type |
| Startup-associated cell | Receives a constructed or converted 16-bit value | Write width and independent placement |
| Associated high-byte storage | Supplies the upper component of that construction | Read width and independent placement |

These qualifications apply wherever those locations appear below. In particular, the cursor-source read is not established as a two-byte read.

### Entry effects and initial call

The first memory modification is a two-byte store through an address obtained from argument 1’s four-byte contents. The stored value is derived from those contents through pointer and 16-bit conversions. With stable contents and the represented 32-bit pointer interpretation, it is the destination address’s low 16 bits.

Argument 1’s contents participate separately in the destination and value expressions. Their read count and relative evaluation order are not independently established. The advanced argument address is argument 1’s address plus four bytes.

A four-byte store of the entry base to base plus `0x14` follows. An unsigned two-byte read at advanced argument address plus entry offset then establishes initial outer data. This read occurs before testing the offset, including when the offset is zero.

A nonzero offset selects a replacement configuration. In observable order, the working fourth argument becomes `0x0897f06c`; storage at `0x0897f07a` receives the offset-derived 16-bit value; `0xbf836004` receives `0x6000`; the advanced address becomes `0xc1b00000`; outer data is replaced by the converted read from `0xc1b00050`; and `0xbf836002` receives the entry 16-bit value. The signed left shift used here has no unconditional wrapping interpretation.

The value read from `0x0897f454` is retained in signed 32-bit state before F-U01. F-U01 receives the signed interpretation of the 32-bit XOR of the working fourth argument and advanced address, shifted right by eight. The signed interpretation precedes that shift.

After F-U01, the retained scalar—not a fresh global read—controls a one-byte read at scalar times `0x20`, followed by a four-byte store of scalar times `0x10` to the same address. Both products are signed expressions. The byte does not determine subsequent control flow.

The remaining entry modifications occur in this order: zero to `0x00000009`, a 16-bit zero value to `0x00000030`, and an explicit two-byte store of the scalar’s low 16 bits at `0x0000000e`.

### Outer phase

Outer address state and address adjustment initially equal zero. Call-control state initially equals `0x0897f0c3`.

Each outer iteration uses a saved location four bytes below the current cursor. Its memory effects have the following ordering:

| Relative position | Effect |
|---|---|
| First | Four-byte outer-data store at cursor minus four |
| Next | Four-byte self-address store at eight times outer address state |
| Next | Four-byte zero store at the address derived by unsigned left shift of outer address state by 17, signed interpretation, signed right shift by 32, then addition of `0x10` |
| Next | Two-byte `0xc470` store at the calculated address below |
| Next | Assignment of `0x8000` to `0xfe238028` |
| Later | Self-address assignment at `0x225fc470`, then assignment of `0x225f` at `0x0000229f` |
| Last before the optional call | Read from `0x00000022`, right shift by 16, conversion, and four-byte store at cursor plus `0x54` |

The calculated two-byte-store address is the unsigned 32-bit result of

\[
(\text{outer data} \mathbin{\text{logical-right-shift}} 4)
-16(\text{address adjustment})
-[\text{outer-data bit 3 is clear}]
+0x26.
\]

The bracket contributes one when the condition holds and zero otherwise.

Before the self-address assignment at `0x225fc470`, outer address state becomes `0x0897f228`. The source read at `0x00000022` is not established to extract an unsigned 32-bit word’s upper half; its declaration is absent.

F-U02 is called only when the comparison pointer differs from cursor plus `0x14`. Its target is selected from the old call-control state plus `0x2e`. Call-control state becomes `0x0897f193` before invocation.

### Inner phase and entry to recurrence

Each inner phase starts with accumulator zero, persistent value and address adjustment `0x0897f1f8`, and a 64-bit processing value whose low half is zero and high half is `0x0897f1f8`. A retained address-like value is `0x0897f22d`.

An inner pass first stores that retained value as four bytes at processing-low plus `0x0c`. It then interprets the processing low half as signed and right-shifts it by the low eight bits of that same half. The branch predicate is whether subtracting one from this shifted signed value overflows. It is not a zero or general sign test.

Before acting on that predicate, the pass reads signed four-byte data at twice the shifted value, assigns its 16-bit conversion to storage identified by F-422afe36d401, and stores four-byte `0x0897f1f8` at the loaded value plus four. The cursor becomes the saved cursor-minus-four location, and outer data becomes `0x0897f1ec`.

An overflow selects another outer iteration carrying that decremented cursor, outer address state `0x0897f228`, and adjustment `0x0897f1f8`. Under conventional 32-bit subtraction, the predicate requires shifted value `−0x80000000`. The initialized low half is zero, so the first inner pass does not satisfy it.

On the non-overflow path, a one-byte `0xd8` store targets the processing high half. The working value becomes eight. F-U03’s target is selected using call-control state plus `0xe56`, and call-control state becomes `0x0897f1d7` before the call with zero.

After F-U03, the accumulator receives the signed entry increment. The function freshly reads `0x0897f216`, converts it to unsigned 32 bits, and installs it as cursor. It then reads one byte at result-low plus `0x0897f1f8`. An unsigned two-byte read at result-low plus `0x0897f22d` supplies a zero-extended destination for a four-byte accumulator store.

The represented byte comparison at this transition equates an explicit zero with the overflow predicate. They are equal on this path, so repeated processing begins with F-U04 argument zero and the byte just read as its byte source.

### Repeated processing and larger-result effects

Each cycle captures the byte source’s low eight bits before F-U04. Subsequent dependency memory changes do not refresh this captured byte.

F-U04’s high result half is retained as an output pointer. Its low half is interpreted as signed 32 bits and multiplied by `0x20` before assignment to unsigned address state. A four-byte self-address store at that address precedes its unsigned comparison with `0x14`.

An address below `0x14` skips the larger-result effects and supplies zero to F-U07. Otherwise, the following ordered groups apply:

| Group | Established effects |
|---|---|
| Before F-U05 | Four-byte persistent-value store at calculated address plus `0x58`; two-byte cursor truncation at persistent value plus `0x0e`; working value becomes persistent value shifted left by two using unsigned arithmetic |
| Startup-associated update | Fresh read of the associated high-byte storage; combination of its byte component with the captured low byte; assignment of the constructed 16-bit value to the startup-associated cell |
| Dependency interval | F-U05 receives cursor logically shifted right by two; F-U06 follows |
| F-U06-result writes | Captured byte at twice the signed result-low; high-half low byte at result-low plus eight; four-byte F-U04 output pointer at twice the accumulator; complete eight-byte result at result-low |
| Output record | Accumulator becomes four times the signed F-U04 output pointer; persistent value becomes signed cursor shifted left by eight; successive four-byte stores through that pointer contain working value, new persistent value, and cursor |

The eight-byte result store occurs after the byte writes and can overwrite them where addresses overlap.

Branch-control state receives the unsigned product of working value and `0x100`, converted to signed state. A selector is derived by signed left shift of cursor by 31, signed right shift by 32, conversion to unsigned 32 bits, and logical right shift by 20. No portable selector value is established.

A zero selector causes a converted 16-bit branch-control assignment to the startup-associated cell. Accumulator plus working value uses mixed signed/unsigned arithmetic: the addition is unsigned, followed by conversion back to signed state. The resulting four-byte value is stored through F-U06’s high-half pointer, and F-U07 receives zero.

A nonzero selector instead produces a four-byte selector store at branch-control state plus `0x20`, selects the breakpoint-path call base, and derives F-U07’s argument by signed right shift of branch-control state by 29. Software breakpoint 4 occurs before F-U07. Only upon resumption does persistent value become F-U06’s signed high half shifted right by 18.

### Tail and recurrence

After F-U07, its low half supplies both the destination and truncated value of a two-byte store. Next comes a four-byte self-address store at persistent value times `0x40`, using unsigned arithmetic.

The function reads four bytes through the current cursor before storing two bytes at persistent value plus four. That store’s value is the cursor converted to a signed 16-bit quantity, increased by four, then converted to the destination width; it is not a cursor dereference.

A four-byte pointer load from F-U07’s high half plus `0x50` replaces the cursor. F-U08 and F-U09 are then called in that order.

Before F-U10, the current working value becomes the next byte source. F-U10 receives the earlier read through the old cursor. That read value subsequently becomes both the new working value and, logically shifted right by three, the next F-U04 argument.

Consequently, the next captured byte comes from the previous working value, not the newly installed old-cursor read. No ordinary exit from this recurrence is shown.

## Side effects

Explicit accesses include one-, two-, four-, and eight-byte stores. Named-storage widths remain unresolved as catalogued above.

Writes can target low addresses, fixed high addresses, cursor-relative memory, dependency-derived pointers, and storage identified by an executable dependency ID. Neither executable-memory permissions nor successful instruction modification is established.

The supplied evidence warns that named storage may overlap smaller symbols. No non-aliasing, atomicity, barrier, device-register, or cache-coherence guarantee is established.

Dependencies may modify shared memory. The contract distinguishes retained values from fresh reads: the initial scalar survives as represented state across F-U01; cursor-source storage is reread after F-U03; the associated high byte is reread during each applicable update; and the cycle byte is captured before F-U04. These facts do not establish undocumented ABI preservation.

A fault, nonreturning dependency, or breakpoint can expose a completed prefix of effects. No rollback or ownership recovery is shown.

## Dependencies

IDs identify interaction roles, not reviewed implementations.

| ID | Call or storage relationship |
|---|---|
| F-U01 | Entry continuation; transformed initial argument; no consumed result |
| F-U02 | Optional target at old call-control state plus `0x2e`; no explicit argument |
| F-U03 | Target at old call-control state plus `0xe56`; argument zero; 64-bit result |
| F-U04 | Target `0x07f755f6`; recurring argument; 64-bit result |
| F-U05 | Target `0x0897ee9f`; cursor logically shifted right by two |
| F-U06 | Executable anchor plus `0xffe`; 64-bit result |
| F-U07 | Target `0x0897f80f` on the small-result path, `0x0897f83b` on the larger-result zero-selector path, or `0x08791770` after breakpoint resumption; 64-bit result |
| F-U08 | Target `0x0897f191`; no explicit argument or consumed result |
| F-U09 | Target `0x0897fddd`; no explicit argument or consumed result |
| F-U10 | Target `0x0897f5e1`; earlier old-cursor read |
| F-422afe36d401 | Storage modification only; no call is established |

Initially, F-U02’s target is `0x0897f0f1`. F-U03 targets `0x0897ffe9` after F-U02’s control-state update, or `0x0897ff19` without it.

The executable anchor’s numeric address is not independently established by its label. Dependency implementations, hidden inputs, ownership effects, clobbers, and return conventions remain unknown.

## Acceptance examples

These are conditional static obligations, not executed tests. Each assumes preceding effects complete and the relevant arithmetic and conversions are defined.

1. **Initial truncation:** Stable argument-1 contents of `0x12345678` require a two-byte `0x5678` store at that address under the represented pointer interpretation.

2. **Zero offset:** The unsigned two-byte read at argument-1 address plus four still occurs. Only the offset-dependent replacement block is omitted.

3. **Unknown destination width:** Assignment of `0x6000` at `0xbf836004` does not require a four-byte store. Assignment of a 16-bit zero at `0x00000030` does not prove a two-byte store.

4. **Fresh cursor:** If F-U03 changes cursor-source storage so its later unsigned conversion yields `0x12345678`, the cursor becomes that value, not `0x5678`.

5. **Initial predicate:** Processing-low zero gives shift count zero and no subtraction overflow. Subject to preceding accesses, the non-overflow path reaches F-U03.

6. **F-U04 boundary:** Result-low zero gives address zero and a self-store before selecting the small-result path. Result-low one gives address `0x20` and selects the larger-result path.

7. **Signed product limit:** Result-low `0x08000000` cannot be assigned a wrapping address-zero interpretation merely because destination state is unsigned; its preceding signed multiplication exceeds 32-bit signed range.

8. **Overlapping result writes:** If F-U06’s low half is zero, its full result store covers addresses zero through seven after the captured-byte store at zero. The later store determines the overlapping bytes if all accesses complete.

9. **Byte recurrence:** The first larger-result cycle makes working value `0x225fc7e0`. If its tail completes, the next captured byte is `0xe0`, regardless of the old-cursor read installed as the new working value.

10. **Breakpoint:** Reaching breakpoint 4 does not establish that F-U07 executes. Its invocation requires resumption.

## Uncertainties

- **documented_only — declarations:** Missing global types prevent recovery of the catalogued access widths and conversion effects.
- **documented_only — arithmetic:** Signed overflow, signed shifts, pointer conversions, and unsigned-to-signed conversions require semantics not supplied here.
- **documented_only — shift counts:** Right shift by 32 and variable counts above 31 have no established portable interpretation. Count masking is not assumed.
- **documented_only — memory:** Endianness, alignment requirements, mappings, overlap, permissions, and executable-storage behavior are unresolved.
- **documented_only — reachability:** Represented backedges and breakpoint continuation do not prove feasible execution paths.
- **documented_only — dependencies:** Actual implementations, executable-anchor address, ABI obligations, and ownership behavior are unavailable.
- **Inferred intent only:** Fixed-address accesses may involve devices, and startup-associated storage may involve reset handling. Neither interpretation is contractual.

## Evidence

Established facts come from explicit access types, expression dependencies, represented control flow, retained-versus-fresh value use, call-target calculations, and store ordering in the supplied static material.

Names, comments, and the prior draft do not establish missing types, executable addresses, higher-level purpose, or runtime reachability. No instruction-level validation, dependency implementation review, implementation access, or runtime testing was performed.
