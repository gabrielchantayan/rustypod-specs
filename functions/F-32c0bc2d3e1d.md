<!-- Trusted provenance: F-32c0bc2d3e1d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-32c0bc2d3e1d

## Purpose

The established behavior is a single ordered path of memory reads and mutations, ending in an unresolved indirect invocation. Loaded values become both stored data and address bases. Access widths, intervening mutations, and the distinction between captured values and fresh reads are material to the contract.

No domain-level purpose is established. Self-referential stores and repeated constants suggest interconnected records, but do not establish a particular data structure, validity invariant, or ownership model.

This specification describes the supplied static representation. It does not claim independent instruction-level confirmation.

## Interface

### Explicit inputs and additional incoming state

| Input | Represented width | Use |
|---|---:|---|
| First argument | 32 bits | No explicit use in the visible body. |
| Primary base | 32 bits | Initial write destination and source of an address-like word. |
| Secondary base | 32 bits | Base for two early word reads. |
| Initial byte | 8 bits | Written at primary base plus `0x45`. |

Additional incoming state consists of:

- **I**, a represented signed 32-bit address-like value, used both as an offset from the secondary base and as a write base.
- An **initial halfword**, 16 bits wide.
- An unresolved indirect-call target.

The four explicit arguments alone therefore do not determine execution.

### Output

No explicit result value is established. If the indirect dependency returns normally, the routine returns without consuming its result.

There is no visible validation response, error code, recovery branch, or rollback mechanism.

### Widths, layouts, and notation

A byte, halfword, and word mean 8, 16, and 32 bits respectively. Offsets are measured in bytes. Decimal 100 is `0x64`; stored constants written as “100” below are decimal.

“Low byte” and “low halfword” mean the least-significant 8 or 16 bits of a captured value. They do not mean an additional memory read.

Capitalized labels below are specification-local snapshots, not source identifiers. Once captured, a label retains its value despite later writes. A fresh read is identified explicitly. Different labels need not designate different storage, and none establishes a valid pointer or object boundary.

The initial snapshots are:

| Label | Captured value |
|---|---|
| A | Word at secondary base plus I |
| B | Word at secondary base plus `0x14` |
| C | Word at primary base plus `0x74` |
| D | Word at A plus `0x14` |
| E | Word at C plus `0x64` |
| U | Unsigned word at E plus `0x14` |

Only the accessed offsets and widths constitute established layout information. Record sizes, field meanings, alignment, and disjointness are unknown.

### Arithmetic and execution preconditions

U and the later snapshot AT are unsigned 32-bit values. The following effective-address calculations use unsigned 32-bit arithmetic and wrap modulo \(2^{32}\):

- U plus `0x09`.
- U plus `0x64`.
- Twice U.
- AT plus `0x45`.
- AT plus `0x44`.

This establishes the numeric results before address interpretation; it does not establish that those addresses are mapped, accessible, aligned, or valid for the relevant access.

Other address calculations use represented signed values or pointer arithmetic. Their overflow, provenance, and address-mapping behavior remain unresolved. In particular, the other doubled-address expressions do not inherit U’s unsigned arithmetic merely because they also involve multiplication by two.

Normal completion requires support for every reached access and the final invocation. Potentially unaligned halfword and word accesses, sums of address-like values, and addresses derived from a signed byte occur without visible null, bounds, alignment, or ownership checks.

## Behavior

The following phases preserve the represented access order. Every observation is conditional on execution reaching it.

### Initial writes and link capture

The initial halfword is written at primary base plus `0x45`. A is then read, and word `0x45` is written at A plus `0x45`. Only afterward is the initial byte written at primary base plus `0x45`.

B is read and written as a word at I plus `0x34`. C and D are read in that order. The next word writes place A at C plus `0x54`, D at D plus `0x54`, and B at C plus `0x44`. A’s byte at `0x09` receives C’s low byte.

E and U are read in order. At the unsigned wrapped address U plus `0x09`, two separate byte writes place D’s low byte and then E’s low byte.

The global word at `0x000000db` is captured as **G1**. A word at B plus `0x14` is captured as **H**; **J** denotes H plus `0x20`. A word read at the unsigned wrapped address U plus `0x64` supplies a destination base, whose word at `0x14` receives C. B’s byte at `0x11` receives J’s low byte.

### Shift-derived value and an early captured destination

C plus `0x44` is read twice, producing distinct snapshots **K** and **L**, in that order.

A signed byte is read at the unsigned wrapped address twice U, promoted to signed 32 bits, and right-shifted by U’s low eight bits. The result’s low byte, the **shift byte**, is written at L plus `0x08`. K is written as a word at C plus `0x74`; the shift byte is then written at C plus `0x11`.

The count ranges from 0 through 255. Counts of at least 32, and right shifts of negative signed values, have unresolved semantics in the supplied representation.

B’s word at `0x54` receives J and then G1 as separate writes. B’s byte at `0x19` receives J’s low byte. L’s word at `0x44` receives C, followed by J at H plus `0x74`.

A word at L plus `0x14` is captured as **LD**. C’s byte at `0x11` then receives G1’s low byte. A subsequent word read at C plus `0x14` supplies the low byte written at LD plus `0x15`. That later read supplies the value, not a replacement destination.

### Mutations through P and Q

A fresh word at C plus `0x14` supplies **P**. C plus `0x64` is then read as **M**.

In order, P’s word at `0x10` receives `0x73`; C’s word at `0x44` receives C; P’s byte at `0x14` receives `0x73`; and C’s word at `0x54` receives G1. C plus `0x74` is captured as **N**. C’s byte at `0x09` receives C’s low byte, then P’s word at `0x54` receives N.

P’s word at `0x10` next receives `0x6e`; C’s word at `0x44` again receives C; P’s word at `0x20` receives `0x6e`.

A fresh word at C plus `0x64`, **O**, supplies the low byte written at P plus `0x09`. **Q** is read at P plus `0x74`, and Q’s word at `0x54` is saved.

The following byte writes remain distinct and ordered: `0x79` at P plus `0x1c`; O’s low byte at C plus `0x11`; M’s low byte at C plus `0x09`; `0x68` at P plus `0x1c`; O’s low byte again at C plus `0x11`; and C’s low byte at Q plus `0x11`.

P plus `0x50` is captured as **V**. C’s words at `0x44`, `0x64`, and `0x34` receive M, O, and O respectively. P’s byte at `0x11` receives O’s low byte. A fresh word at C plus `0x74` is captured as **W**.

P’s word at `0x10` receives V, Q’s word at `0x54` receives its saved value, and P’s bytes at `0x00` and `0x09` receive V’s low byte and C’s low byte respectively.

### Transition to R and S

A word at C plus `0x14` supplies **R**. Q’s byte at `0x11` receives P’s low address byte. P’s word at `0x44` receives R, and P’s byte at `0x00` again receives V’s low byte.

Q’s words at `0x74` and `0x14` are captured as **X** and **S** in that order. P’s word at `0x54` receives P.

A record address is read at S plus `0x04`. Its byte at `0x11` receives W’s low byte, then its byte at `0x10` receives 100. R’s word at `0x54` receives 100 before that record’s word at `0x60` receives 100.

The old word at R plus `0x74` is captured before that field receives S. The captured target’s word at `0x30` then receives 100.

**T** is read at S plus `0x14`. T’s byte at `0x09` receives W’s low byte. R’s word at `0x54` receives R. A word write at X minus `0x20` has value X minus `0x74`.

A word at T plus `0x14` supplies a destination base. S’s byte at `0x11` receives N’s low byte. A fresh word at S plus `0x14`, the **S14 snapshot**, supplies the low byte written at that captured destination plus `0x15`.

### Marker writes and replacement links

A fresh S-plus-`0x14` read supplies **Y**. R’s word at `0x64` is read as **Z** and immediately written back unchanged. Y’s byte at `0x0c` receives `0x73`; R’s byte at `0x15` receives Z’s low byte; S’s word at `0x34` receives the S14 snapshot.

Y’s word at `0x74` is captured as **AA**. Y’s word at `0x54` receives Y. S’s byte at `0x0d` receives R’s low byte, followed by the S14 snapshot at S plus `0x54` and Y at S plus `0x14`.

A fresh R-plus-`0x64` word is captured as **AB**. Byte `0x74` is written at Y plus `0x10`, word `0x74` at R plus `0x54`, and word `0x74` at Y plus `0x10`. Y plus `0x74` is then captured as **AC**.

S’s word at `0x54` receives S. Y’s word at `0x30` receives `0x74`. R’s byte at `0x01` receives Z’s low byte; Z’s byte at `0x09` receives AB’s low byte; R’s word at `0x74` receives AB.

R’s byte at `0x11` receives `0x74`. Y’s word at `0x60` is read, after which R’s byte at `0x11` receives Z’s low byte. R’s word at `0x34` receives AB. Y’s byte at `0x10` receives the low byte of the Y-plus-`0x60` snapshot.

A record address is read at R plus `0x14`. S’s byte at `0x11` receives AA’s low byte. R’s word at `0x44` receives S, and R’s byte at `0x11` receives AB’s low byte. R plus `0x74` is captured as **AD**, followed by S’s low byte at R plus `0x0d`.

The captured R-plus-`0x14` target receives word `0x66` at `0x10`. Another word write of S at R plus `0x44` intervenes before byte `0x66` is written at the target’s `0x10`. R’s word at `0x54` and the target’s word at `0x60` then receive `0x66`.

A fresh R-plus-`0x74` read supplies **AE**, before that field receives S. AE’s word at `0x40` receives `0x66`. R’s byte at `0x0d` receives AD’s low byte. AE plus `0x44` is captured, then R’s word at `0x54` receives R and AC’s word at `0x54` receives AC.

### Later traversal and timed rereads

A record address is read at AE plus `0x14`. S’s byte at `0x11` receives the captured AE-plus-`0x44` low byte. A fresh S-plus-`0x14` word supplies the low byte written at the loaded record plus `0x15`.

Another record address is read at S plus `0x14`. R plus `0x64` is captured as **AF**; word `0x73` is written at the loaded record plus `0x20`; R plus `0x64` is reread as **AG**. AG’s low byte is written at that record plus `0x09`.

Its word at `0x74` supplies the next record. That record receives R’s low byte at `0x0d`, a read-and-write-back of its word at `0x54`, and byte `0x74` at `0x09`. AF is then written at R plus `0x44`, before R is written at the traversed record plus `0x34`.

The traversed record’s word at `0x04` supplies another record. Its word at `0x24` receives AG. R’s byte at `0x19` receives AF’s low byte. The new record’s words at `0x74` and `0x04` are captured as **AH** and **AI**.

A fresh R-plus-`0x74` read supplies **AJ**. A word read at AJ plus `0x14` supplies a target whose word at `0x54` receives AH. AJ plus `0x04` is then captured separately as the **first AJ target**.

R’s byte at `0x0d` receives AG’s low byte twice. R’s word at `0x64` receives AI. Word `0x74` is written at the first AJ target plus `0x30`. Only afterward is AJ plus `0x04` reread; this later target’s word at `0x74` supplies **AK**.

A fresh R-plus-`0x64` word, the **retained R64 snapshot**, supplies the low byte written at R plus `0x1d`. R’s word at `0x44` is read, and that target’s word at `0x14` is captured as the **replacement target**. AH is written at AK plus `0x34` before the replacement target is written at R plus `0x14`.

R’s word at `0x54` is then saved. The captured replacement target receives R’s low byte at `0x09`. Reading AK’s word at `0x04`, then that target’s word at `0x74`, supplies **AL**.

AL’s byte at `0x15` receives AH’s low byte. R’s word at `0x54` receives R; R’s byte at `0x11` receives AK’s low byte; AL’s byte at `0x11` receives the saved R-plus-`0x54` low byte. AL plus `0x64` is captured as **AM**.

### Final mutations and invocation

R’s byte at `0x09` receives the retained R64 snapshot’s low byte. A fresh AK-plus-`0x04` read supplies **AN**. AN’s byte at `0x09` receives R’s low byte; its word at `0x50` is captured as **AO**; its byte at `0x11` receives AM’s low byte; its byte at `0x09` again receives R’s low byte.

R’s word at `0x14` receives AM. R’s byte at `0x0d` receives AN’s low address byte. AN’s byte at `0x00` receives AO’s low byte.

AK’s words at `0x74` and `0x14` are captured as **AP** and **AQ**. AN’s word at `0x54` receives AN. AQ’s word at `0x04` supplies **AR**. AR’s byte at `0x10` receives 100; R’s word at `0x54` receives 100; AR’s word at `0x40` receives 100.

R’s byte at `0x0d` receives AM’s low byte. AQ’s byte at `0x09` receives AR’s low byte. R’s word at `0x24` receives AM. AQ’s byte at `0x11` receives R’s low byte.

The global word at `0x087fc9c0` is captured as **G2**, then R plus `0x74` is freshly read as **AS**. The terminal access sequence is:

1. AS’s low halfword is written at AS plus R.
2. G2’s low halfword is written at twice AR.
3. A signed byte at AP plus R is read and converted to unsigned 32-bit **AT**.
4. AS’s low halfword is written at G2 plus AS, then again at AS plus R.
5. Word `0x45` is written at the unsigned wrapped address AT plus `0x45`.
6. Global words at `0x087fc9dc` and `0x087fc9c8` are read as **G3** and **G4**, respectively.
7. R plus `0x44` is read as **AU**.
8. Byte `0x44` is written at the unsigned wrapped address AT plus `0x44`.
9. Word G4 is written at twice G3.
10. G3’s low halfword is written at AH plus G3.

The unresolved dependency is then invoked with three represented 32-bit arguments: AT bitwise-OR `0x44`, G3, and AU.

## Side effects

Every described write belongs to the represented trace, including duplicate writes, overwritten values, mixed-width writes to the same offset, and unchanged-value write-backs. Equal final contents do not establish equivalence if accesses are removed or reordered.

There is no visible allocation, release, locking, explicit I/O operation, or ownership transfer before the invocation. This does not establish ownership of accessed storage or exclude memory-mapped I/O.

Aliasing can make any earlier write affect a later read, including a global read. Captured values remain snapshots; fresh reads are not interchangeable with them.

An access failure may occur after earlier mutations. No restoration path or transactional boundary is visible. The dependency’s additional effects are unknown.

## Dependencies

**F-000000000000** is a specification-local opaque identifier for the unresolved indirect dependency, not a recovered identity. Only the invocation position and three represented arguments are established. Its target, calling convention, return behavior, memory effects, global preservation, and memory ownership rules are **documented_only**.

The global storage references are established independently of their unknown contents:

| Snapshot | Storage address | Read width |
|---|---:|---:|
| G1 | `0x000000db` | 32 bits |
| G2 | `0x087fc9c0` | 32 bits |
| G3 | `0x087fc9dc` | 32 bits |
| G4 | `0x087fc9c8` | 32 bits |

The supplied representation warns that some global symbols overlap smaller symbols. It does not resolve their ownership, semantic types, or complete overlap relationships.

## Acceptance examples

These are conditional static deductions, not executed tests.

1. **Unsigned U offsets.** If U is `0xffffffff`, U plus `0x09` is `0x00000008`, U plus `0x64` is `0x00000063`, and twice U is `0xfffffffe`. Those numeric results are established; accessibility is not.

2. **Captured destination.** If LD differs from the subsequent word read at C plus `0x14`, the byte write still targets LD plus `0x15`. The subsequent value contributes only its low byte.

3. **Transition aliasing.** If R plus `0x54` equals the S-plus-`0x04` target plus `0x60`, the two word-100 writes occur separately in the specified order.

4. **AF ordering.** The AF write at R plus `0x44` follows the traversed record’s byte-`0x74` write and precedes its word write of R at `0x34`.

5. **Separate AJ targets.** If intervening writes change AJ plus `0x04`, the word-`0x74` write uses the first captured target. The traversal supplying AK uses the later reread.

6. **Early AK mutation.** If AK plus `0x34` overlaps R plus `0x54`, the saved R-plus-`0x54` word reflects the preceding AH write, subject to byte order and intervening overlapping writes.

7. **Width distinction.** The byte-`0x74` and word-`0x74` writes at Y plus `0x10` are separate accesses. The latter affects four bytes, not one.

8. **Negative terminal byte.** If the terminal signed byte is −1, AT is `0xffffffff`. The word-`0x45` destination is `0x00000044`, and the later byte-`0x44` destination is `0x00000043`, by unsigned 32-bit addition. The first dependency argument is `0xffffffff`. Address accessibility remains unknown.

9. **Partial completion.** If the halfword write at twice AR fails and execution stops there, preceding mutations may remain. The terminal byte read, subsequent global reads, and invocation have not been reached.

## Uncertainties

The following remain **documented_only**:

- Domain purpose, record types, object extents, valid input ranges, and permitted aliases.
- Independent instruction-level fidelity, endianness, alignment support, and atomicity.
- Signed overflow and pointer arithmetic outside the explicitly established unsigned calculations.
- Address mapping and accessibility, including for unsigned-wrapped addresses.
- Oversized shifts and right shifts of negative signed values.
- Concurrent modification, volatile storage, and device-memory semantics.
- Global contents, semantic types, ownership, and complete overlap relationships.
- The indirect target and behavior beyond its represented invocation.

Repeated constants and self-links do not establish stronger structural, lifetime, or ownership guarantees.

## Evidence

The supplied static function representation is the sole behavioral evidence. It establishes the represented access widths, constants, sequence, incoming-state dependencies, captured-versus-reread distinctions, unsigned U and AT calculations, global storage references, and terminal invocation arguments.

Structural purpose is inferred intent only. No runtime tests, execution traces, dependency implementation, or independent instruction-level evidence have been provided or used.
