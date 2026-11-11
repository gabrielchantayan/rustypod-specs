<!-- Trusted provenance: F-b4ec05c6cce7; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b4ec05c6cce7

## Purpose

F-b4ec05c6cce7 performs three initial 32-bit stores, then enters an unconditional loop containing indirect calls, memory transfers of several widths, a software breakpoint operation, and a change to an implicit frame-like address base.

The supplied static representation establishes no normal return, loop-exit condition, validation branch, or recovery path. The first explicit argument has no visible use. The second explicit argument affects the initial stores but has no visible subsequent use.

A higher-level purpose is not established. Descriptions such as initialization, device servicing, or record processing would be inferred intent, not behavioral requirements.

This contract distinguishes:

- **Established static evidence:** operations, widths, data dependencies, and sequencing visible in the supplied representation.
- **Inferred intent:** possible semantic interpretations that are not required by that evidence.
- **documented_only unknowns:** matters requiring unavailable calling-convention, dependency, memory-map, or execution evidence.

No runtime tests have been performed.

## Interface

### Explicit inputs

| Input | Represented width | Visible use |
|---|---:|---|
| First argument | 32 bits | No visible read or forwarding. |
| Second argument, called **n** below | Signed 32-bit integer | Multiplied by 256 and 512 for initial address calculations and one stored value. |

No range restriction is enforced visibly. The signed arithmetic in the representation does not, by itself, establish executable behavior for overflow or invalid address formation.

### Implicit inputs

Execution also depends on:

- A global 32-bit base value, read separately three times during initialization.
- Two other global 32-bit address values, each captured at a particular point in every iteration.
- An incoming link-like value used to select the first indirect call.
- An implicit frame-like address base.
- An incoming register value used in a late byte-store address.
- Secondary register results associated with three call sites.
- Memory reached through constants, calculated addresses, globals, and call results.

These inputs are not visibly initialized or validated by this routine.

### Outputs and termination

There is no explicit returned value. The visible control flow repeats indefinitely, subject to calls returning, the breakpoint resuming, and memory accesses completing.

“Unconditional loop” does not guarantee progress. A dependency can fail to return; the breakpoint can suspend or terminate execution; a memory access can fault. No corresponding status result or recovery action is shown.

### Widths and notation

A byte is 8 bits, a halfword is 16 bits, and a word is 32 bits. A pair-valued result is represented as 64 bits, split into low and high 32-bit components.

All offsets are byte offsets. Unsigned halfword loads produce values from 0 through 65,535. Narrow stores retain the source’s low 8 or 16 bits.

The address labels used below name represented values, not validated pointers or known structure types. No allocation extent, byte order, alignment guarantee, atomicity, or memory permission is established.

Unsigned word shifts have 32-bit results. For signed additions, multiplications, shifts, and address arithmetic, exact overflow and fault semantics remain documented_only; the contract does not silently impose universal modulo arithmetic.

## Behavior

### Initialization: separate global reads and three stores

Let **G** denote the global base-value source, rather than a single captured value.

The visible initialization proceeds as follows:

1. Read G and retain that value as **B**.
2. Calculate **D = 512 × n**.
3. Perform a word store whose destination is a separately read value of G plus 256 × n, and whose stored value is another separately read value of G.
4. Store D as a word at B + D.
5. Store B as a word at B + D.

The first store therefore does **not** necessarily use B for either its destination base or its stored value. There are three represented reads of G: the initial capture and the two reads associated with the first store.

The representation does not establish an evaluation order between the first store’s destination-base read and its stored-value read. Neither is specified here as occurring first. Both supply that store, after the initial capture.

If G remains stable across all three reads, the first store simplifies to storing B at B + 256 × n. That stability is an additional assumption, not established behavior.

The second and third stores explicitly use captured B and the same calculated D. The third overwrites the second at the same destination if both complete, but the intermediate write remains observable. A memory observer, device, or fault can distinguish the two writes.

### Dependency identifiers

F-U01 through F-U37 are opaque call-site identifiers assigned in iteration order. They are not resolved dependency identities. Separate identifiers do not establish separate target functions.

A call without listed explicit arguments may still consume registers, globals, or other implicit state. Unspecified arguments, clobbers, ownership effects, and failure behavior remain unknown.

### Iteration opening and first byte stores

The ordering of the two byte initializations is significant:

1. Initialize the byte value destined for the later store at **P + 0x15** to zero.
2. Invoke F-U01 with two explicit zero arguments. Its target is selected using the current link-like value plus 0xFFE.
3. After F-U01 returns, initialize a different byte value—the addend used by F-U06—to zero.
4. Invoke F-U02.
5. Invoke F-U03.
6. Read a global address value and retain it as **P**.
7. Invoke F-U04.
8. Invoke F-U05, receiving pair **A**.
9. Invoke F-U06 with these ordered arguments:
   - A’s low word;
   - A’s high word plus A’s low word plus the byte initialized after F-U01.
10. Store the secondary byte result associated with F-U06 at P + 0x11.
11. Invoke F-U07.
12. Store the byte initialized before F-U01 at P + 0x15.
13. Invoke F-U08, receiving a halfword result and a secondary address.
14. Store that halfword at the secondary address + 0x0C.

Thus only the byte destined for P + 0x15 is initialized before F-U01. The F-U06 addend is initialized after that call returns and before F-U02.

The representation carries both initialized values to their uses. This does not establish a general register-preservation guarantee for the dependencies.

P is captured before F-U04. No reload of its global source is shown before either P-relative byte store. A dependency’s change to that global source does not, by itself, replace captured P; it can nevertheless affect the validity or contents of the addressed storage.

### Pair-derived stores and fixed-address load

F-U09 returns pair **B₂**. The following operations occur in order:

1. Store a zero halfword at B₂’s high word + 0x2C.
2. Store B₂’s low word logically shifted right by 26 as a word at the current frame base + 0x26C.
3. Establish the address constant 0x21C48960.
4. Invoke F-U10.
5. Load a word-sized address from the current frame base + 0x320.
6. Invoke F-U11.
7. Invoke F-U12.
8. Store B₂’s low word shifted left by two as a word at the captured address + 0x54.
9. Invoke F-U13, F-U14, F-U15, F-U16, and F-U17, in that order.
10. Load unsigned halfword **H** from address 0x21C48994.

The right-shifted result lies between 0 and 63. The left shift is explicitly an unsigned 32-bit operation; bits shifted beyond the word width are discarded.

The address loaded from the frame is captured before F-U11 and F-U12. No reload is shown before the subsequent store.

### Breakpoint and halfword-derived addresses

After loading H:

1. Invoke F-U18.
2. Load a word from the current frame base + 0x280.
3. Pass that word to F-U19 and retain its word result.
4. Execute a software breakpoint operation with immediate 0x92.

All following operations depend on execution continuing after the breakpoint.

5. Shift F-U19’s result left by two to obtain offset **O**.
6. Calculate **K** by logically shifting 0x21C48960 right by 18 and multiplying by four. K equals 0x21C4.
7. Load word-sized address **R** from H + 0x44.
8. Invoke F-U20.
9. Invoke F-U21 and retain its word result.
10. Load a word from H + 4.
11. Invoke F-U22 with F-U21’s result as its **first** argument and the word loaded from H + 4 as its **second** argument.
12. Read another global address value and retain it as **Q**.
13. Invoke F-U23.
14. Store Q’s low halfword at Q + O.
15. Invoke F-U24.
16. Load word-sized address **S** from K + 0x54, which is address 0x2218.
17. Invoke F-U25 and retain its returned address.
18. Store K’s low halfword, 0x21C4, at that returned address + 4.
19. Invoke F-U26.
20. Invoke F-U27.
21. Load unsigned halfword **J** from S + 0x34.

H is used directly as a zero-extended address component. No separately identified base is added to it. Consequently, H + 4 ranges from 4 through 0x10003, and H + 0x44 ranges from 0x44 through 0x10043.

Q supplies both an address base and the narrowed stored value. The representation establishes neither a record identifier nor an ownership transfer.

### Final pair, frame change, and distributed stores

F-U28 returns pair **C**. Let **V** be its low word and **W** its high word.

The remaining iteration performs these operations in order:

1. Store V as a word at the current frame base + 0x258.
2. Load unsigned halfword **T** from W + 0x14.
3. Calculate V shifted left by six.
4. Calculate four times W.
5. Decrease the implicit frame-like base by 0xF4.
6. Calculate four times J.
7. Store V as a word at 4 × J + 0x74.
8. Invoke F-U29.
9. Store T as a word at R + 0x54.
10. Invoke F-U30.
11. Store T as a word at R + 0x64.
12. Invoke F-U31.
13. Store R’s low halfword at the sum of the calculated V-left-shifted-by-six value and F-U31’s secondary register result.
14. Invoke F-U32.
15. Invoke F-U33.
16. Load a byte from J + 4 × J, equivalently 5 × J.
17. Invoke F-U34.
18. Store the captured byte at T + the incoming implicit register value + 0x19.
19. Invoke F-U35.
20. Store the calculated four-times-W value as a word at R + 0x44.
21. Invoke F-U36.
22. Establish a fixed link-like value for the next iteration.
23. Load a word from the calculated four-times-W value + 0x44.
24. Pass that loaded word to F-U37.

The loop then repeats without a visible condition.

T is zero-extended: its stores at R + 0x54 and R + 0x64 are full word stores with their upper 16 bits zero. The late byte is captured before F-U34; no reread is shown afterward.

Because J is an unsigned halfword, 4 × J is at most 262,140 and 5 × J is at most 327,675. These particular calculations fit within 32 bits. That does not establish that the resulting addresses are mapped or correctly aligned.

The frame-base decrease has no visible matching increase. If interpreted literally, and absent changes from dependencies or other implicit effects, each completed iteration contributes a decrease of 244 bytes. Whether this is genuine repeated frame movement remains documented_only.

The first F-U01 target depends on incoming link-like state. Later iterations use the fixed value established near the preceding iteration’s end.

## Side effects

### Memory and ordering

The routine writes bytes, halfwords, and words through global-derived, frame-relative, constant-derived, and result-derived addresses. It also performs explicitly ordered reads between calls.

The regions may alias. An earlier store can affect a later read, including a global-value read, if their storage overlaps. No disjointness, synchronization, or concurrency model is established.

The described order is the order represented by the supplied material. It is not a guarantee of bus transactions, hardware atomicity, volatile-access semantics, or ordering against external agents.

No dependency is assumed to preserve globals, memory, ownership, or implicit machine state. Descriptions of captured values report visible dataflow, not an independently verified calling convention.

### Ownership and lifetime

No explicit allocation, release, retain, lifetime check, or ownership transfer appears in this routine. Dependencies may perform such actions; their behavior is unknown.

For an access to complete, its destination or source must be usable at the actual access point. A successful earlier read of an address value does not prove continued validity across subsequent calls.

### Errors and interruption

There are no visible null checks, bounds checks, result-status checks, retries, or rollback actions.

Invalid memory, alignment restrictions, arithmetic consequences, dependency behavior, or breakpoint handling can prevent progress. The representation establishes no error return, exception interface, or recovery policy.

## Dependencies

| Opaque identifiers | Established contribution |
|---|---|
| F-U01 | Link-selected call with two zero arguments; occurs between the two byte initializations. |
| F-U02–F-U04 | Calls surrounding capture of P; capture occurs after F-U03 and before F-U04. |
| F-U05 | Produces pair A. |
| F-U06 | Receives the A-derived arguments; supplies the secondary byte stored through P. |
| F-U07–F-U08 | Separate the P-relative writes and supply a halfword plus secondary destination. |
| F-U09 | Produces pair B₂. |
| F-U10–F-U17 | Surround the frame-loaded address capture and B₂-derived store. |
| F-U18–F-U19 | Precede the breakpoint; F-U19 consumes a frame-loaded word and supplies O’s source. |
| F-U20–F-U22 | Include a word producer and an ordered two-argument call using a load through H. |
| F-U23–F-U25 | Surround Q-derived and K-derived halfword stores; F-U25 supplies a destination. |
| F-U26–F-U28 | Precede J’s load and produce pair C. |
| F-U29–F-U31 | Separate the T word stores; F-U31 supplies a secondary address component. |
| F-U32–F-U36 | Surround the late byte transfer and R + 0x44 update. |
| F-U37 | Consumes the word loaded through the four-times-W-derived address before repetition. |

Complete signatures, target identities, clobbers, non-returning behavior, and ownership effects are documented_only.

## Acceptance examples

These are conditional consequences of static evidence, not executed tests.

1. **Stable global, zero argument.** If all three initialization reads of G yield B, n is zero, and the stores complete, all three destinations are B. The stored words are B, zero, and B, in that order. Without global stability, the first destination and value need not equal B.

2. **Stable global, unit argument.** If all three reads yield B and n is one, the first store writes B at B + 0x100. The next stores write 0x200 and then B at B + 0x200, assuming valid arithmetic and addresses.

3. **Distinct initialization reads.** Suppose the initial capture yields 0x1000, the first store’s destination-base read yields 0x2000, its value read yields 0x3000, and n is one. The first store writes 0x3000 at 0x2100. The next stores write 0x200 and then 0x1000 at 0x1200. This example does not prescribe which of the two reads associated with the first store occurs first.

4. **F-U01 does not return.** The pre-F-U01 byte initialization and the call have been reached. The separate byte initialization used by F-U06 has not been reached; neither have F-U02 or the later iteration stores.

5. **Unsigned shifts.** If B₂’s low word is 0xFC000003, the frame-relative extraction writes 63. The later word store through the captured address writes 0xF000000C.

6. **F-U22 argument order.** If F-U21 returns 0x11223344 and the subsequent word load from H + 4 yields 0x55667788, those are F-U22’s first and second explicit arguments, respectively.

7. **Narrowed address value.** If Q is 0x12345678 and Q + O is writable, the halfword stored there is 0x5678, not the full Q value.

8. **Zero extension and captured byte.** If T is 0xFFFF, both T stores write 0x0000FFFF as words. Separately, if J is seven, the late byte is read from address 35 and retained across F-U34 for its subsequent store.

9. **Breakpoint does not resume.** Earlier stores and dependency effects may already have occurred. No post-breakpoint operation is required to occur.

## Uncertainties

The following remain documented_only:

- Fidelity of the static representation’s implicit register dependencies and reconstructed control flow.
- Actual calling convention, pair-result convention, and secondary-result interpretation.
- Whether the frame-like base change represents literal executable behavior.
- Exact signed-overflow, address-wrap, alignment, and fault semantics.
- Global stability during initialization and any externally observable meaning of separate reads.
- Memory mapping, permissions, aliasing, byte order, atomicity, and device-specific effects.
- Breakpoint handling and resumption.
- Dependency effects on globals, registers, memory, ownership, and control flow.
- Semantic meaning of the constants, records, and transferred values.

A repeating processing-loop interpretation is plausible intent only.

## Evidence

The supplied static representation shows:

- An initial global capture followed by two further uses that read that global for the first store.
- Three ordered initial word stores, with the latter two using the captured base.
- One byte initialization before F-U01 and the other after its return, before F-U02.
- An unconditional loop containing 37 indirect call sites.
- Ordered byte, halfword, and word accesses, including unsigned halfword loads.
- Three pair-valued results and several secondary register results.
- A software breakpoint with immediate 0x92.
- A frame-like base decrease of 0xF4.
- No visible normal return, validation branch, or recovery path.

This contract uses only that supplied static material. No dependency inspection, memory-map validation, implementation execution, or runtime testing was performed.
