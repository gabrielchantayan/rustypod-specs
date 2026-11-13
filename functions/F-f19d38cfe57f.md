<!-- Trusted provenance: F-f19d38cfe57f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-f19d38cfe57f

## Purpose

The represented procedure performs an ordered sequence of indirect calls, calculated word stores, byte reads, and byte stores, followed by an unconditional loop.

No application-level purpose is established. The evidence does not justify describing it as an encoder, allocator, initializer, or protocol operation.

This contract distinguishes:

- **Established static evidence:** visible operations, dataflow, and ordering.
- **Conditional arithmetic deductions:** consequences under the stated 32-bit machine interpretation.
- **documented_only unknowns:** facts not recoverable from the supplied material.

The representation warns of invalid instruction data and removal of five unreachable regions. Accordingly, this contract describes the represented path, not proven behavior of an independently recovered executable. No runtime tests have been performed.

## Interface

### Inputs and entry state

| Input | Represented width | Visible role |
|---|---|---|
| First explicit argument, **A** | Signed 32-bit | Supplies stored words, address arithmetic, and the first call’s first argument. |
| Second explicit argument | Opaque 32-bit | No explicit use is visible. |
| Third explicit argument, **B** | Unsigned 32-bit | Its logical right shift by seven supplies one call argument. |
| Additional entry word, **E** | Unsigned 32-bit register state | Supplies scaled words, shifted fields, addresses, and call arguments. |
| Additional entry control value | Represented 32-bit control-register state | Selects the first indirect-call target through addition of 3810. |

The three explicit arguments alone do not determine the represented execution. Neither additional entry value has a visible initialization within the procedure.

A later write also uses a secondary register result associated with the call immediately before that write. Its meaning and calling-convention status are unknown.

The unused second explicit argument is not proven irrelevant to all dependencies: recovered explicit argument lists do not establish every register a callee might inspect.

### Outputs and completion

There is no represented return value and no normal return on the visible path. If dependencies return and memory operations complete, execution reaches an unbounded loop with no visible exit.

Observable outputs are direct memory effects, indirect-call events, their explicit arguments and ordering, and any externally visible dependency behavior.

No ownership transfer, output-buffer extent, completion notification, or status convention is established.

### Widths and notation

For the behavioral tables:

- **Rₙ** denotes the 32-bit result retained from call event **Cₙ**.
- Multiplication, address addition, and left shifts are interpreted conditionally as retaining the low 32 bits.
- Unsigned right shifts are logical.
- A word store writes four bytes; a byte store retains the low eight bits.
- **S** denotes the secondary register result after C24.

Some represented expressions use signed multiplication or signed left shift. Their overflowing behavior is not established by source-language rules; the modular interpretation makes the arithmetic deductions explicit without resolving that uncertainty.

Endianness, alignment requirements, memory mappings, and pointer validity are unknown.

## Behavior

### Ordering model

The finite prefix contains **98 indirect calls, 12 word stores, five byte reads, and six byte stores**. Each completed loop iteration adds **ten indirect calls, one byte read, and one byte store**.

The tables preserve call order, explicit argument counts, retained-return forwarding, and placement of direct memory accesses. An entry listing several calls means those calls occur individually, left to right, each with no explicit arguments. “Unused” means the represented caller does not subsequently consume that primary result; it does not mean the call has no effects.

**C labels identify call events, not dependency function identities.** Every intervening call remains observable. Dependencies may change memory, affect other state, fail, transfer control elsewhere, or never return.

### Initial word-write sequence

Before C01, store the word A at address \(4A+20\). There is no preceding visible validation.

| Call event, in order | Explicit arguments | Retained result or immediately following direct effect |
|---|---|---|
| C01 | A, \(8E\) | Primary result unused. Target depends on incoming control state. |
| C02 | None | Retain R₂. |
| C03 | R₂ | Primary result unused. |
| C04 | None | Retain R₄. |
| C05 | R₄, \(E\mathbin{\gg}8\) | Primary result unused. |
| C06, C07, C08 | None each | Primary results unused. |
| C09 | None | Retain R₉; then store A at \(8E+20\). |
| C10 | \(4R₉\) | Primary result unused. |
| C11, C12, C13 | None each | Primary results unused. |
| C14 | None | Store \(4R₁₄\) at \(16R₁₄+20\). |
| C15 | \(8E,\ 8E,\ E\mathbin{\gg}8,\ B\mathbin{\gg}7\) | Primary result unused. |
| C16, C17, C18 | None each | Primary results unused. |
| C19 | None | Retain R₁₉; then store \(8E\) at \(8E+20\). |
| C20 | \(4R₁₉\) | Primary result unused. |
| C21, C22, C23 | None each | Primary results unused. |
| C24 | None | Primary result unused; the subsequent write uses secondary result S. |

In particular, C06–C08 are three separate calls, followed by the fourth no-explicit-argument call C09. The store after C09 occurs **before** C10 receives the scaled retained result.

After C24, save \(H=S\cdot2^{24}\), and store H at \(S\cdot2^{26}+20\). Under modular arithmetic, the stored word depends on S’s low eight bits; the scaled address component depends on its low six bits.

### Remaining word-write sequence

The following saved values remain relevant across later represented calls:

- **K** is \(4R₃₄\).
- **Q** is the unsigned word \(4R₄₈\).
- **M** is \(16R₆₂\).
- **J** is the selector index \((Q\ \text{AND}\ 2147483647)\mathbin{\gg}7\).

Their continued use is visible dataflow, not a guarantee that unknown dependencies preserve arbitrary globals or memory.

| Call event, in order | Explicit arguments | Retained result or immediately following direct effect |
|---|---|---|
| C25 | H | Primary result unused. |
| C26, C27, C28 | None each | Primary results unused. |
| C29 | None | Store H at \(4R₂₉+20\). |
| C30 | \(4R₂₉\) | Primary result unused. |
| C31, C32, C33 | None each | Primary results unused. |
| C34 | None | Save K; store K at \(16R₃₄+20\). |
| C35 | K | Primary result unused. |
| C36 | None | Retain R₃₆. |
| C37 | R₃₆ | Primary result unused. |
| C38 | None | Retain R₃₈. |
| C39 | R₃₈ | Primary result unused. |
| C40, C41, C42 | None each | Primary results unused. |
| C43 | None | Store K at \(4R₄₃+20\). |
| C44 | \(4R₄₃\) | Primary result unused. |
| C45, C46, C47 | None each | Primary results unused. |
| C48 | None | Save Q; store Q at \(16R₄₈+20\). |
| C49 | Q | Primary result unused. |
| C50 | None | Retain R₅₀. |
| C51 | R₅₀ | Primary result unused. |
| C52 | None | Retain R₅₂. |
| C53 | R₅₂ | Primary result unused. |
| C54, C55, C56 | None each | Primary results unused. |
| C57 | None | Store \(E\mathbin{\gg}16\) at \(4R₅₇+20\). |
| C58 | \(4R₅₇\) | Primary result unused. |
| C59, C60, C61 | None each | Primary results unused. |
| C62 | None | Save M; store \(4R₆₂\) at \(M+20\). |
| C63 | M, M, \(E\mathbin{\gg}15\) | Exactly three explicit arguments. |
| C64, C65, C66 | None each | Primary results unused. |
| C67 | None | Retain R₆₇; then store M at \(M+20\). |
| C68 | \(4R₆₇\) | Primary result unused. |
| C69, C70, C71 | None each | Primary results unused. |
| C72 | None | Retain unsigned R₇₂ for the byte-write phase. |

The stores after C62 and C67 are separate operations to the same calculated address, with C63–C67 between them. They cannot be replaced by a single final-value description.

The stored \(E\mathbin{\gg}16\) is an unsigned quantity with up to 16 significant bits, not necessarily one byte.

### Selector-based byte access

A **J-selected write** consists of these ordered operations:

1. Read one unsigned byte at address \(J+1\).
2. Add 17 to the loaded byte’s value.
3. Store one byte at that resulting address.

J lies between zero and \(2^{24}-1\). Each resulting destination lies between 17 and 272 inclusive. No table base or object base is visibly added.

Each selected write performs a fresh read. The byte is not cached across calls or earlier writes.

A later **alternate-selected write** uses the same read-then-write mechanism, but reads at address \(1024D+1\), where \(D=4R₈₄\). Under modular arithmetic, that read address is also \(4096R₈₄+1\).

### Finite byte-write sequence

Immediately after C72, perform a J-selected write of the low byte of \(4R₇₂\).

| Call event, in order | Explicit arguments | Retained result or immediately following direct effect |
|---|---|---|
| C73 | \(4R₇₂,\ 4R₇₂,\ 4R₇₂\) | Primary result unused. |
| C74 | None | Retain R₇₄. |
| C75 | R₇₄ | Primary result unused. |
| C76, C77, C78 | None each | Primary results unused. |
| C79 | None | Retain R₇₉; then perform a J-selected write described below. |
| C80 | \(4R₇₉\) | Primary result unused. |
| C81, C82, C83 | None each | Primary results unused. |
| C84 | None | Save unsigned R₈₄ and scaled word D; then perform two ordered byte writes described below. |
| C85 | \(E\mathbin{\gg}23\) | Primary result unused. |
| C86 | None | Primary result unused. |
| C87 | \((R₈₄\ \text{AND}\ 1073741823)\mathbin{\gg}6\) | Argument is not byte-truncated. |
| C88, C89, C90 | None each | Primary results unused. |
| C91 | None | Retain R₉₁; then perform a J-selected zero write. |
| C92 | \(4R₉₁\) | Primary result unused. |
| C93, C94, C95 | None each | Primary results unused. |
| C96 | None | Derive five-bit quantity P; then perform an alternate-selected zero write. |
| C97 | \(2P\) | Primary result unused. |
| C98 | None | Retain R₉₈ for loop entry. |

The write after C79 uses the low byte of
\((R₇₂\ \text{AND}\ 1073741823)\mathbin{\gg}20\).
Consequently, the stored byte consists of original R₇₂ bits 20 through 27. The intermediate shift can contain ten significant bits; the store retains eight.

After C84, the direct accesses occur in this order, with no intervening call:

1. Store the low byte of D at \((E\mathbin{\gg}23)+28\).
2. Read the J-selector byte.
3. Store zero at the destination selected by that read.

The first destination lies between 28 and 539 inclusive. Its write may change the selector byte read immediately afterward.

After C96, derive
\(P=(R₉₆\ \text{AND}\ 1073741823)\mathbin{\gg}25\).
P lies between zero and 31. The alternate-selected zero write occurs before C97, not after the loop begins.

### Unconditional loop and target transition

Every iteration has the following ten-call structure. **L1–L10** identify recurring call positions, not resolved dependency functions.

| Position | Explicit arguments | Retained result or direct effect |
|---|---|---|
| L1 | Carried first argument, carried second argument | Primary result unused. |
| L2, L3, L4 | None each | Primary results unused. |
| L5 | None | Retain its result; then perform an alternate-selected zero write. |
| L6 | Four times the L5 result | Primary result unused. |
| L7, L8, L9 | None each | Primary results unused. |
| L10 | None | Retain result W to determine the next iteration’s arguments. |

The first iteration’s L1 arguments are R₉₈ and zero. The second value is obtained by shifting P right by eight, which is zero for every permitted P.

For each subsequent iteration:

- The first L1 argument is \(4W\), retaining 32 bits.
- The second is W shifted left by 26, truncated to 32 bits, then logically shifted right by eight.
- Equivalently, the second argument is W’s low six bits multiplied by \(2^{18}\).

**Target-selecting state also changes.** The first iteration’s L1 uses the code-location base carried into the loop from the finite prefix, plus displacement 4094. At the end of the iteration, the carried base has been replaced by another represented code-location base. Later iterations’ L1 use that replacement base plus the same displacement. The replacement base is numerically 62 bytes higher in the representation.

Thus the first and subsequent L1 target expressions differ. This does not prove that they identify distinct underlying functions. The replacement expression remains the one used on later iterations; it is not restored to the entry expression.

The alternate-selector source remains based on saved D. Each iteration rereads its byte after L5 and before L6.

## Side effects

The direct operations include word stores, byte reads, and byte stores. No direct word load is visible.

There is no established object layout. Additions of 20, 17, 28, or one are address calculations, not proven fields within allocated objects.

Aliasing is unrestricted:

- Word destinations may coincide or overlap byte locations.
- Different selected writes may reach different destinations.
- A selected write may modify its own future selector source.
- The entry-derived byte write may alter a selector source or selected destination.

Intervening dependencies can also change memory. Earlier stored values are therefore not guaranteed to persist until a later direct overwrite.

No allocation, release, reference-count operation, or ownership transfer is established. Returned words are not proven to be pointers or owned objects.

No validation, bounds check, error translation, rollback, or cleanup path is visible. Invalid accesses or call targets may interrupt execution; the exact fault mechanism is unknown. Already performed writes are not visibly undone.

## Dependencies

All represented calls are indirect. The supplied opaque function ID identifies this procedure only; no dependency function IDs are supplied. Dependency identities therefore remain unresolved rather than receiving invented identities.

C01’s target depends on incoming control-register state. Later finite targets use represented code-location bases and fixed displacements; C78 uses a symbol-derived code base. The loop additionally carries target-selecting state as specified above.

Distinct call events or target expressions do not prove distinct functions. No dependency is assumed to preserve globals, avoid aliasing, own or release memory, return normally, or obey a completely recovered signature.

## Acceptance examples

These are conditional static deductions, not observed tests.

1. **Initial store:** If A is three and the destination is writable, the procedure stores the word three at address 32 before C01.

2. **Intervening calls matter:** After C05 returns, C06, C07, C08, and C09 execute in that order, each without explicit arguments. If C08 never returns, C09, the following store, and C10 do not occur.

3. **Store before scaled forwarding:** If E is 256, the store after C09 writes A at address 2068. Only afterward does C10 receive \(4R₉\). If execution reaches the store after C19, that store writes 2048 at address 2068.

4. **Unsigned third argument:** If B is \(2^{32}-1\), C15’s fourth explicit argument is \(2^{25}-1\), without sign extension.

5. **Fresh selector reads:** If successive selector reads obtain nine and ten, the corresponding stores target addresses 26 and 27. The first result does not determine the second destination.

6. **Selector self-modification:** Suppose J is 25 and address 26 contains nine immediately before the zero write after C84. That write reads address 26 and writes zero back to address 26. If intervening calls leave it unchanged, the read after C91 obtains zero and selects address 17.

7. **Loop-entry state:** Whether P is zero or 31, the first L1 receives zero as its second argument and uses the entry target expression. Later L1 events use the replacement target expression, regardless of whether their argument values happen to match the first iteration.

8. **Later argument boundary:** If W is 63, the next L1 arguments are 252 and 16,515,072. If W is 64, they are 256 and zero, under modular arithmetic.

## Uncertainties

The following remain **documented_only**:

- Fidelity of control flow given invalid-instruction and unreachable-region warnings.
- Actual calling convention and completeness of explicit argument lists.
- Provenance and validity of additional entry state and secondary result S.
- Dependency identities, signatures, side effects, and termination behavior.
- Original-machine semantics of overflowing signed operations.
- Endianness, alignment, memory permissions, and address-space interpretation.
- Whether numerical low addresses denote ordinary memory, mapped hardware, or invalid locations.
- Behavior of omitted regions and any alternative paths they might represent.
- Application-level purpose, intended data format, and ownership conventions.

## Evidence

Evidence consists exclusively of the supplied static representation and its recovery warnings.

The call tables account for the visible finite call sequence, retained-result forwarding, intervening memory accesses, and ten-call loop body. The loop description separately preserves argument-producing state and the entry-to-recurring target-expression transition.

Widths, address ranges, masks, call counts, access counts, and acceptance examples are static observations or arithmetic deductions under the declared interpretation. No implementation execution, dependency inspection, memory trace, or runtime test has been performed.
