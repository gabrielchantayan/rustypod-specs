<!-- Trusted provenance: F-1e09d19eb6dc; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1e09d19eb6dc

## Purpose

F-1e09d19eb6dc performs an ordered sequence of indirect calls, computed-address 32-bit stores, and byte reads followed by computed-address byte stores. If execution reaches its final repeating region and the operations continue completing, that region repeats indefinitely. No normal return or result value is represented.

No application-level purpose is established. The supplied material does not establish that the addresses identify ordinary objects, device registers, or a valid dispatch structure.

This contract separates established unsigned arithmetic and access widths from unresolved signed arithmetic and value conversions. In particular, an established eight-bit store does not necessarily establish which eight-bit pattern is stored.

## Interface

### Explicit inputs

| Input | Represented width and interpretation | Visible use |
|---|---|---|
| A | Unsigned 32-bit integer | Initial store address and value, first call argument, and retained selector-address calculation. |
| B | Uninterpreted 32-bit value | No explicit use. |
| C | Unsigned 32-bit integer | Logically shifted right by seven for an argument. |
| D | Unsigned 32-bit integer | Word-store value, direct byte-store address, and call argument. |

B remains part of the apparent interface. Its lack of explicit use does not establish that an indirect dependency cannot observe its incoming machine-level location.

### Implicit inputs and preconditions

Execution depends on incoming link-register state, memory at computed addresses, dependency results, and any additional machine state consumed by dependencies.

Each access requires an accessible address supporting the represented width. Each indirect transfer requires a usable target and compatible calling context. No visible validation establishes these conditions. Alignment, permissions, address interpretation, and the complete calling convention remain unresolved.

### Output and completion

There is no represented return value and no visible normal return. The final loop has no exit test.

An invalid access, invalid transfer, arithmetic or conversion failure, nonreturning dependency, or nonlocal transfer may prevent later operations. There is no represented error handler, rollback, or status-based recovery.

### Widths and conversion rules

The representation distinguishes unsigned 32-bit values, signed-looking 32-bit values, unsigned eight-bit selector reads, and eight-bit stores. Word stores access 32 bits. Byte order is unknown.

The following rules must remain distinct:

- Unsigned 32-bit arithmetic is reduced modulo \(2^{32}\). This includes the initial address calculation and the two unsigned source shifts described below.
- Right shifts of unsigned values are logical.
- Signed-looking multiplication, addition, and left shifts do not acquire wrapping semantics merely because they calculate an address or eventually feed an unsigned destination.
- Assigning an unsigned value above `0x7fffffff` to a signed-looking 32-bit temporary has unresolved conversion behavior. Neither a bit-preserving result nor continued execution is established.
- Three stores explicitly convert a value to a plain-character type. Its signedness and out-of-range conversion behavior are not established. The eight-bit access width alone does not prove low-eight-bit truncation.
- Values from 0 through 127 are representable under either signed or unsigned interpretation of an eight-bit plain-character type. Larger conversion inputs require additional semantics before a stored value can be specified unconditionally.

A **low-byte conversion assumption**, used only in conditional examples, means that conversion completes and the stored eight-bit pattern equals the conversion input modulo 256. For the two stores that first pass through a signed-looking temporary, identifying that byte with the earlier unsigned result also requires the intermediate conversion to complete and preserve the relevant bits.

## Behavior

F-U01 through F-U60 are opaque identifiers for indirect-call sites in encounter order. They do not establish distinct callee identities. “No explicit arguments” refers only to the supplied representation.

Every later operation is conditional on preceding operations completing. The sequence below specifies represented program order, not hardware memory-ordering guarantees.

### Entry and first indexed store

The first explicit memory operation stores A as a 32-bit word at:

\[
(4A + \texttt{0x14}) \bmod 2^{32}.
\]

F-U01 then receives A. Its target depends on incoming link-register state and a fixed displacement.

The next calls are:

| Calls, in order | Explicit data relationship |
|---|---|
| F-U02 → F-U03 | F-U02 has no explicit arguments; its uninterpreted 32-bit result is forwarded to F-U03. |
| F-U04 → F-U05 | F-U04 has no explicit arguments; its uninterpreted 32-bit result is forwarded to F-U05. |
| F-U06 → F-U07 → F-U08 → F-U09 | Each has no explicit arguments. |

F-U09 supplies a signed-looking 32-bit result. D is then stored as a 32-bit word at four times that result plus `0x14`. F-U10 receives four times the same result.

These multiplications and the address addition are signed-looking operations. Their overflow behavior is not established by the unsigned behavior of the initial A-based store.

### Shared word destination

F-U11, F-U12, F-U13, and F-U14 run in order without explicit arguments.

A retained **word base** is formed as sixteen times the signed-looking F-U14 result. A 32-bit word containing four times that result is stored at word base plus `0x14`.

F-U15 then receives, in order:

1. Word base.
2. Word base again.
3. C logically shifted right by seven.

Between the store and F-U15, the unsigned quantity `(A AND 0x7fffffff)` logically shifted right by seven is formed and retained. This calculation does not read memory. Adding one gives the **A-selector address** used later.

F-U16, F-U17, F-U18, and F-U19 follow without explicit arguments. The same word destination is then explicitly overwritten with word base. F-U20 receives the signed-looking F-U19 result shifted left by two.

The base calculation, word values, displaced address, and F-U20 shift involve signed-looking arithmetic. Intervening dependencies may affect memory; the second explicit store still uses the retained base.

### First unsigned byte source

F-U21, F-U22, F-U23, and F-U24 run without explicit arguments. The unsigned 32-bit F-U24 result is retained as the **first byte source**.

That source is shifted left by two using unsigned 32-bit arithmetic. Bits shifted beyond bit 31 are discarded. The result is then assigned to a signed-looking 32-bit temporary.

This assignment precedes the next selector operation. If its input exceeds `0x7fffffff`, the temporary’s value and whether execution continues are unresolved.

Before F-U25:

1. An unsigned byte is read at the A-selector address.
2. An eight-bit store targets the loaded byte’s value plus `0x11`.

The stored value is obtained by converting the signed-looking temporary to plain character. Consequently, it is not unconditionally the unsigned shift result’s low eight bits. That identity requires suitable semantics for both conversions.

The selector read precedes the destination store. The relative evaluation order of the plain-character conversion and the selector read is not established; if conversion prevents completion, the read must not be assumed to have occurred.

F-U25 receives the same signed-looking temporary in all three explicit argument positions. These arguments are not narrowed to eight bits.

Next:

| Calls, in order | Explicit data relationship |
|---|---|
| F-U26 → F-U27 | F-U26 has no explicit arguments; its uninterpreted 32-bit result is forwarded to F-U27. |
| F-U28 → F-U29 → F-U30 → F-U31 | Each has no explicit arguments. |

Before F-U32, another A-selector read selects another byte-store destination using loaded byte plus `0x11`.

For this store, the first byte source is masked with `0x3fffffff` and logically shifted right by twenty. The established result is an unsigned value from 0 through 1023, not necessarily an eight-bit value. It is then converted to plain character for storage.

Under the low-byte conversion assumption, the stored pattern selects source bits 20 through 27. Without that assumption, this identity is not established for out-of-range conversion inputs. Conversion failure and its ordering relative to the selector read remain unresolved.

F-U32 receives the signed-looking F-U31 result shifted left by two.

Every A-selector operation performs a fresh read. Neither the selector value nor its derived destination is cached.

### D-derived store and selector clearing

F-U33, F-U34, F-U35, and F-U36 run without explicit arguments. The unsigned 32-bit F-U36 result becomes the **second byte source**.

Its unsigned left shift by two has established 32-bit truncation. The shifted value is assigned to a signed-looking temporary retained as the **scale**. Above `0x7fffffff`, the conversion result and continued execution remain unresolved.

Before F-U37, the represented memory operations are:

1. An eight-bit store at `(D logically shifted right by 7) + 0x1c`.
2. A fresh A-selector byte read.
3. A zero-byte store at that loaded byte plus `0x11`.

The direct store converts the retained scale to plain character. Its byte value is conversion-dependent, not unconditionally the unsigned shift result’s low byte. If either conversion prevents completion, the direct store and subsequent operations are not guaranteed to occur.

F-U37 receives D logically shifted right by seven. F-U38 follows without explicit arguments. F-U39 receives the second byte source masked with `0x3fffffff` and logically shifted right by six. This unsigned argument ranges from zero through `0x00ffffff`.

F-U40, F-U41, F-U42, and F-U43 follow without explicit arguments. Another A-selector read and zero-byte store occur before F-U44. F-U44 receives the signed-looking F-U43 result shifted left by two.

These calls do not explicitly replace the retained scale.

### Alternate selector and loop initialization

F-U45, F-U46, F-U47, and F-U48 run without explicit arguments.

The unsigned F-U48 result is masked with `0x3fffffff` and logically shifted right by twenty-five. This produces the initial **loop state**, from 0 through 31.

Before F-U49, an unsigned byte is read at the **alternate-selector address**:

\[
1024 \times \text{retained scale} + 1.
\]

A zero byte is then stored at the loaded byte plus `0x11`.

This address uses the retained scale originating at F-U36, not the F-U48 result. Its multiplication and addition are signed-looking. Their overflow behavior is unresolved, independently of the earlier unsigned shift.

F-U49 receives initial loop state shifted left by one. This unsigned operation produces an even value from 0 through 62.

F-U50 then runs without explicit arguments. Its signed-looking result becomes the initial **loop payload**.

### Repeating region

Each iteration has this order:

| Region | Behavior |
|---|---|
| Entry | F-U51 receives loop payload, then loop state logically shifted right by eight. |
| First call group | F-U52, F-U53, F-U54, and F-U55 run without explicit arguments. |
| Selector effect | The alternate-selector byte is freshly read; zero is stored at loaded byte plus `0x11`. |
| Scaled-result call | F-U56 receives the signed-looking F-U55 result shifted left by two. |
| Final call group | F-U57, F-U58, F-U59, and F-U60 run without explicit arguments. |
| Transition | The F-U60 result shifted left by two becomes the next payload. The same result shifted left by twenty-six is assigned to unsigned loop state. |

Both transition shifts have signed-looking operands. Assignment of the second result to unsigned state does not retroactively establish unsigned shifting semantics.

For the first F-U51 invocation, the second argument is necessarily zero because initial state is at most 31.

If signed shifts are separately assumed to have 32-bit wrapping bit-vector behavior, subsequent second arguments equal the preceding F-U60 result’s low six bits shifted left by eighteen. This is conditional, not an established arithmetic rule.

F-U51 identifies a call site, not a guaranteed invariant target. Its target-forming state differs between initial entry and subsequent iterations.

There is no visible termination test.

## Side effects

### Explicit accesses and layout

A completed pre-loop path contains:

- Four 32-bit word stores.
- One direct eight-bit store at the D-derived address.
- Four A-selector byte reads, each followed by an eight-bit store.
- One alternate-selector byte read followed by a zero-byte store.

Each completed loop iteration adds one alternate-selector byte read and one zero-byte store.

These are access counts, not guarantees that execution reaches every access.

Every completed selector read yields an unsigned value from 0 through 255. Its derived destination therefore lies from `0x11` through `0x110`, inclusive.

The A-selector address lies from 1 through `0x01000000`. The D-derived store address lies from `0x1c` through `0x0200001b`. Neither calculation overflows unsigned 32-bit arithmetic.

The initial word-store address wraps modulo \(2^{32}\) and is always numerically divisible by four. No object layout, buffer extent, or byte order is established.

### Aliasing and ordering

No disjointness checks appear. Explicit stores may alter later selector reads, other destinations, or state visible to dependencies. A selector-derived store may overwrite a byte subsequently used as a selector.

Dependencies may also change memory. Repeated reads cannot be replaced by an assumed stable selector value, and repeated zero stores cannot be dismissed as redundant.

Retained values describe explicit data flow. They do not imply that unknown dependencies preserve arbitrary registers, globals, memory, or resource state.

### Ownership and failures

No explicit allocation, release, ownership transfer, or lifetime boundary appears. Computed addresses do not establish ownership or access rights.

Dependency ownership effects are unknown. A failure after earlier writes does not undo those writes through any represented local mechanism.

## Dependencies

All F-U01 through F-U60 are unresolved indirect-call sites.

F-U01 depends on incoming link-register state. Later targets are formed from fixed address expressions, represented code-address values, or retained target-forming state. Target validity and semantic identity are not established. Multiple site IDs may reach one implementation.

For every dependency, the following are unresolved:

- Complete calling convention and implicit arguments.
- Register, stack, global, and memory effects.
- Whether it returns normally.
- Error signaling, exceptions, and nonlocal transfers.
- Ownership, retention, and release behavior.
- Application meaning of returned values.

Only explicit argument order and represented result uses are contracted here. Arithmetic use of a result does not establish that the callee intentionally returns an index, address, status, or packed field.

## Acceptance examples

These are conditional static consequences, not executed tests. Each requires reaching the relevant operation and completing preceding operations.

### Initial unsigned wrapping

- With A = 3, the initial store writes 3 at `0x20`; F-U01 subsequently receives 3.
- With A = `0x40000000`, multiplication by four wraps to zero; the store writes `0x40000000` at `0x14`.
- With A = `0xffffffff`, the final initial-store address is `0x10`.

These numeric addresses do not establish accessibility.

### Unsigned extraction and selector destinations

With C = `0xffffffff`, F-U15’s third argument is `0x01ffffff`.

If F-U48 supplies `0x3fffffff`, initial loop state is 31, F-U49 receives 62, and the first F-U51 second argument is zero.

If a completed selector read returns `0x7f`, the following completed store targets `0x90`. A later read returning `0x80` selects `0x91`, even with unchanged A.

### Two distinct plain-character conversion boundaries

If F-U24 returns `0x12345678`:

- Its unsigned shifted result is `0x48d159e0`, within the signed range.
- All three F-U25 arguments are therefore `0x48d159e0`, if reached.
- The first selector store contains `0xe0` only under the low-byte conversion assumption.
- The later extraction produces `0x123`, or 291, before plain-character conversion. A stored `0x23` likewise requires the low-byte conversion assumption.

Neither byte identity follows merely from an eight-bit store width.

If F-U24 returns `0xc0000001`, its unsigned shift produces exactly 4. The signed-looking assignment and plain-character conversion both receive a representable value. The first selector store writes 4, and F-U25 receives 4 in all three positions.

If F-U24 returns `0x20000000`, the unsigned shift produces exactly `0x80000000`. The next signed-looking conversion is unresolved, including whether execution continues. A byte value of zero is only conditional on a completing, bit-preserving intermediate conversion and suitable plain-character conversion.

### Retained scale

With D = `0x100` and F-U36 returning `0x40000001`, the unsigned shift produces 4:

- Retained scale is 4.
- The direct byte store writes 4 at `0x1e`.
- The alternate-selector address is `0x1001`.

Both conversions receive representable values.

If F-U36 instead returns `0x20000000`, only its unsigned shifted result, `0x80000000`, is established. Continued execution, retained scale, direct-store byte value, and subsequent alternate-selector address are not established without additional conversion and arithmetic semantics.

### Representable loop transition

If F-U60 returns 3, both transition shifts produce positive representable values. The next payload is 12, the next unsigned state is `0x0c000000`, and the next F-U51 second argument is `0x000c0000`.

No general signed-wrap assumption is needed for this example.

## Uncertainties

### Established static evidence

The representation establishes the displayed call sequence, access widths, unsigned masks and shifts, unsigned initial-address wrapping, unsigned F-U24 and F-U36 shift truncation, explicit conversion stages, retained data relationships, and unconditional final loop.

### Inferred intent

Repeated call groups and selector-derived stores could suggest state-machine or device-oriented behavior. Neither interpretation is established or needed.

Bit-preserving signed conversions, low-byte plain-character conversions, and wrapping signed arithmetic may describe an underlying implementation. They remain assumptions, not consequences of the access widths.

### documented_only

Unresolved matters include:

- Whether recovered control flow matches executable instructions.
- Architecture, byte order, alignment, and permissions.
- Out-of-range unsigned-to-signed conversion results and continuation.
- Plain-character signedness, out-of-range conversion results, stored representations, and continuation.
- Conversion-versus-selector-read evaluation order where not established.
- Signed overflow and invalid signed-shift behavior.
- Alternate-selector addresses when signed arithmetic is not representable.
- Ordinary, volatile, or device-memory access semantics.
- Dependency identities, hidden effects, ownership, and return behavior.
- Outcomes after invalid accesses or transfers.
- Effects of omitted or unrecovered regions.
- Application-level success criteria.

## Evidence

The sole behavioral evidence is the supplied static representation. It includes warnings about problematic instruction recovery and removal of unreachable regions. Those warnings limit confidence in executable fidelity; they do not justify inventing replacement behavior.

The arithmetic and ordering conclusions above are derived independently from the represented operands, widths, conversions, accesses, and control flow. Prior prose and reviewer feedback are not independent evidence.

No runtime tests or execution have been performed. Acceptance examples are conditional arithmetic and ordering consequences, not claims of reachability, memory safety, conversion semantics, or successful execution.
