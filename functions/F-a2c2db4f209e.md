<!-- Trusted provenance: F-a2c2db4f209e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a2c2db4f209e

## Purpose

F-a2c2db4f209e sequences indirect calls, calculated-address memory transfers, and named-global accesses. The represented body explicitly writes **shared word A twice** and reads **shared word B once**. It does not explicitly update shared word B. Calculated-address stores may alias either global or other storage, and dependency effects are unknown; therefore, the number of distinct modified shared-state objects is not established.

The final explicit call targets address zero and receives three zero-valued arguments. Normal return is represented only after that call returns.

The static evidence does not establish an application-level purpose. Labels such as “payload,” “index,” and “link word” below describe dataflow, not object types, allocation semantics, or a dispatch protocol.

## Interface

### Explicit arguments and completion

The represented interface has four 32-bit arguments and no explicit result.

| Argument | Established direct consumption |
|---|---|
| First | None in the represented body. |
| Second | Passed to F-d001. |
| Third | None in the represented body. |
| Fourth | Interpreted as unsigned, logically shifted right by 23, and passed to F-d003. |

Absence of direct use does not establish machine-level irrelevance. Unknown dependencies may observe ambient machine state.

There is no explicit success value, failure value, or exception protocol. Completion requires every preceding call and access to complete and the final address-zero call to return.

### Additional required state

The body consumes values outside its declared arguments:

- Incoming link-related state used to construct the first call target.
- An unsigned 32-bit **entry word**.
- A 32-bit **entry address**.
- An unsigned 32-bit **entry selector**.
- A **saved secondary word** obtained from a secondary-register value associated with F-d016.
- A late secondary-register value used as a byte-store destination.
- Two named 32-bit global views, **shared word A** and **shared word B**.
- A named 24-bit global view used in the final composition of shared word A.
- Memory at calculated numeric addresses.

These labels do not imply additional ordinary parameters or complete dependency return signatures.

### Widths and arithmetic

A byte is 8 bits, a halfword 16 bits, a word 32 bits, and a represented wide result 64 bits. Byte reads used in address calculations are zero-extended. Byte stores retain the low eight bits of the supplied value. Unsigned right shifts are logical.

A transfer described as reading a word and storing its low byte requires a 32-bit source read; it is not interchangeable with an 8-bit source read.

Under the **32-bit bitvector interpretation**, scaling, addition, and left shifts produce their low 32 bits. Several expressions are represented with signed intermediate types. Whether overflowing signed arithmetic or signed left shifts faithfully describe machine operations remains documented_only. Examples relying on wraparound are conditional on the bitvector interpretation.

Endianness, alignment requirements, and the physical overlap of global views are not established.

## Behavior

The following order applies only while preceding operations complete. The represented body has no explicit conditional branches, loops, retries, or validation paths.

F-d001 through F-d114 are opaque call-site IDs in execution order. They do not imply 114 distinct targets. “No explicit arguments” describes the representation, not an empty machine-level interface.

### Initial extraction and byte store

F-d001 receives the second argument. F-d002 has no explicit arguments. F-d003 receives the fourth argument logically shifted right by 23. F-d004 through F-d006 have no explicit arguments, and F-d007 produces the **first index**.

Before F-d008, the body reads a byte at entry address plus 1. Adding 17 to that byte gives a destination address. The body stores there the low byte of entry word logically shifted right by 22. This destination is between 17 and 272 inclusive; it is not entry address plus an offset.

F-d008 receives the first index shifted left by 2. F-d009 through F-d011 have no explicit arguments. F-d012 produces the **second index**, and F-d013 receives that index shifted left by 2 as two equal arguments.

F-d014 produces the **first selected word**. F-d015 receives it, followed by F-d016 with no explicit arguments.

### Selector-derived clearing and retained secondary state

The body reads a byte at:

\[
(\text{entry selector} \gg 9)+1
\]

It writes zero to the byte-derived address plus 17. After this store, the secondary-register value associated with F-d016 is retained as the saved secondary word and passed to F-d017. An ordinary declared return value does not establish its provenance.

F-d018 produces a word. F-d019 receives that word and the first selected word logically shifted right by 23. F-d020 through F-d022 have no explicit arguments, and F-d023 produces the **third index**.

The selector-derived byte location is then read again. This is a fresh read after intervening calls. Its value is retained as the **small base**, and zero is stored at small base plus 17.

### First named-global write and small-base transfer

F-d024 receives the third index shifted left by 2. F-d025 through F-d027 have no explicit arguments. F-d028 produces the **fourth index**, and F-d029 receives that index shifted left by 2 twice.

F-d030 produces the **second selected word**. The body writes it to shared word A before passing the retained value to F-d031. F-d032 has no explicit arguments.

The body then reads a full word at small base plus 116 and writes its low byte at small base plus 1. The small base is retained across F-d031 and F-d032, but those dependencies may change the source memory.

### Repeated transfers from a retained scaled base

F-d033 receives the saved secondary word shifted left by 11. F-d034 produces a represented 64-bit result. F-d035 receives, in order:

1. Its low 32 bits.
2. Its high 32 bits.
3. The second selected word logically shifted right by 23.

F-d036 through F-d038 have no explicit arguments. F-d039 produces a word whose fourfold value becomes **base C**.

The body reads a word at base C plus 116 and stores its low byte at base C plus 1. F-d040 receives base C. F-d041 through F-d043 have no explicit arguments. F-d044 produces the **retained index**; that index shifted left by 2 becomes the **retained scaled word**.

The body then performs a second full-word read at base C plus 116 and a second low-byte store at base C plus 1. The source is reread after the intervening dependencies.

F-d045 receives:

1. Saved secondary word shifted left by 12.
2. Retained scaled word.
3. Retained scaled word again.
4. Retained index shifted left by 18.

### Payload acquisition and differently scaled destinations

F-d046 produces a word passed to F-d047. F-d048 produces another word passed to F-d049. F-d050 through F-d052 have no explicit arguments. F-d053 produces an index whose fourfold value becomes **base D**.

The body reads a full word at base D plus 116 and retains it as the **payload**. Its low byte is stored at base D plus 1.

F-d054 receives base D. F-d055 through F-d057 have no explicit arguments. F-d058 produces an index. The full retained payload is written at sixteen times that index plus 20.

F-d059 receives the retained scaled word logically shifted right by 2 and then shifted left by 16. This preserves the represented operation order. Under 32-bit bitvector arithmetic, its numerical value equals the original retained index shifted left by 16:

\[
\left(\left((x\ll2)\bmod2^{32}\right)\gg2\right)\ll16
\equiv
(x\mathbin{\&}\texttt{0x3FFFFFFF})\ll16
\equiv x\ll16
\pmod{2^{32}}.
\]

The discarded top two bits of \(x\) cannot affect the low 32 bits after a left shift by 16. This equality does not resolve the separate signed-arithmetic uncertainty in the supplied representation.

F-d060 produces a word passed to F-d061. F-d062 produces a word passed to F-d063. F-d064 through F-d066 have no explicit arguments. F-d067 produces an index.

The body writes the full retained payload at four times the F-d067 index plus 20. It does not reread the original payload source.

### Linkage writes and full-word clearing

F-d068 receives four times the F-d067 index. F-d069 through F-d071 have no explicit arguments. F-d072 produces a word; shifting it left by 2 produces **link word E**.

The body stores link word E at:

\[
(\text{retained scaled word}\mathbin{\&}\texttt{0xFFFFFFFC})+20.
\]

The mask is explicit, although under the bitvector interpretation the retained scaled word already has two zero low bits.

F-d073 receives link word E twice. F-d074 produces a word passed to F-d075. F-d076 produces a word passed to F-d077. F-d078 through F-d080 have no explicit arguments. F-d081 produces an index.

A zero-valued 32-bit word is written at four times that index plus 20.

### Pair-derived links and delayed read through shared word B

F-d082 receives four times the F-d081 index. F-d083 through F-d085 have no explicit arguments. F-d086 produces a represented 64-bit pair.

The pair’s low word shifted left by 2 becomes **link word F**. Its high word becomes the **pair index**.

The body first writes link word F at four times the pair index plus 20. It then reads shared word B once and retains its value as the **saved base**.

F-d087 receives link word F, pair index, link word F, and link word F, in that order. F-d088 through F-d090 have no explicit arguments. F-d091 produces another index.

The body writes link word F at four times the F-d091 index plus 20. Only afterward does it read the word at saved base plus 20, retaining that result as the **captured word**.

The saved base is not reloaded from shared word B. Nevertheless, the contents at saved base plus 20 may change before the delayed read, including through the immediately preceding store if addresses alias.

### Payload-derived byte destination

F-d092 receives four times the F-d091 index. F-d093 through F-d095 have no explicit arguments. F-d096 produces a represented 64-bit pair, designated **final low word** and **final high word**.

The body reads a byte at:

\[
(\text{payload}\gg23)+1.
\]

It stores the captured word’s low byte at that byte-derived address plus 17.

F-d097 receives final high word three times, followed by final low word shifted left by 2. F-d098 produces a word passed to F-d099. F-d100 through F-d102 have no explicit arguments. F-d103 produces an index.

### Masked-selector byte destination

The body reads a byte at:

\[
((\text{final low word}\mathbin{\&}\texttt{0x3FFFFFFF})\gg21)+1.
\]

It stores the low byte of captured word logically shifted right by 22 at that byte-derived address plus 17.

F-d104 receives the F-d103 index shifted left by 2. F-d105 through F-d107 have no explicit arguments. F-d108 produces an index, and F-d109 receives that index shifted left by 2 twice.

F-d110 has no explicit arguments. F-d111 produces a word passed to F-d112. F-d113 produces an 8-bit result.

### Final accesses and dispatch

After F-d113, the final explicit operations occur in this order:

1. Write a zero byte through the late secondary-register-derived destination.
2. Read the 24-bit global view and combine it as the upper 24 bits with the F-d113 result as the low byte; write the resulting word to shared word A.
3. Write a zero halfword at:
   \[
   (\text{final low word}\mathbin{\&}\texttt{0x3FFFFFFF})\gg7.
   \]
4. Invoke F-d114 through address zero with three zero-valued arguments.
5. Return if F-d114 returns.

The 24-bit global value is not an earlier snapshot. The preceding byte store may affect it if the addresses overlap. The halfword destination is not rounded for alignment and can be zero or odd.

## Side effects

The named-global accesses are two explicit 32-bit writes to shared word A, one explicit 32-bit read of shared word B, and a 24-bit global read used for the final composition. No explicit named-global write to shared word B appears.

Other direct effects include calculated-address byte, halfword, and word stores and corresponding reads. These accesses may overlap the named globals or one another. No exact count of distinct modified storage objects follows.

Every dependency may have additional memory, global-state, allocation, release, or control-flow effects. No preservation or ownership contract is supplied. Numeric addresses are not proven to identify allocated objects, and passing an address-like value does not establish ownership transfer.

Repeated accesses remain separate observable operations. Intervening calls, aliasing, and memory attributes may distinguish them even when calculated addresses match.

No atomicity, synchronization, rollback, transaction, or reentrancy guarantee is established.

## Dependencies

F-d001 through F-d114 identify call sites only. Target identity and equivalence between sites remain documented_only.

Most targets are formed from fixed code-associated values plus offsets. F-d001 instead depends on incoming link-related state. F-d114 explicitly targets address zero. None of these constructions proves that the target is callable.

The caller consumes:

- 32-bit results from the word and index producers identified above.
- 64-bit results from F-d034, F-d086, and F-d096, split into low and high words.
- An 8-bit result from F-d113.
- Secondary-register state associated with F-d016 and the late byte-store destination.

These observations do not establish complete signatures, register-preservation rules, failure behavior, or memory ownership.

## Acceptance examples

These are conditional static expectations, not executed tests. Each assumes preceding operations complete and relevant accesses are supported.

1. **Argument extraction.** A fourth argument of `0xFFFFFFFF` causes F-d003 to receive `0x000001FF`: nine retained bits.

2. **Initial indirect byte destination.** If the byte at entry address plus 1 is `0x20` and entry word is `0xFFC00000`, the initial byte store writes `0xFF` at address `0x31`.

3. **Fresh selector reads.** If the two selector-derived byte reads yield 4 and 9 respectively, the zero-byte destinations are 21 and 26. The retained small base is 9.

4. **Full-width source access.** A word value of `0x123456AB` at small base plus 116 produces byte `0xAB` at small base plus 1. The required source access remains 32 bits.

5. **Repeated source observation.** If intervening dependencies change the low byte at base C plus 116 from `0x11` to `0x22`, the corresponding stores write `0x11` and then `0x22`.

6. **Retained payload and different strides.** A payload of `0x89ABCDEF` remains the value used by both later payload stores despite subsequent source-memory changes. If F-d058 and F-d067 each produce 3, their destinations are 68 and 32 respectively.

7. **Equivalent F-d059 value.** Under bitvector arithmetic, a retained index of `0xC0001234` gives retained scaled word `0x000048D0`. F-d059 receives `0x12340000`, also the low 32 bits of the original index shifted left by 16.

8. **Retained base, delayed contents.** If shared word B yields `0x1000`, changing shared word B afterward does not redirect the captured-word read from `0x1014`. A change to memory at `0x1014` before that read can change the captured word.

9. **Unaligned final destination.** A final low word of `0xC0000080` gives final halfword destination 1. Completion requires support for that access.

10. **Final composition timing.** If the 24-bit global read yields `0x123456` after the late zero-byte store and F-d113 yielded `0xAB`, the final shared-word-A write is `0x123456AB`. Earlier contents of that global view do not determine this write.

11. **Partial effects on failure.** If an access faults before final dispatch, earlier completed effects may remain. No rollback or later execution is established.

## Uncertainties

The following remain documented_only:

- Application-level purpose and validity of the represented function boundary.
- Complete calling convention and provenance of register-derived state.
- Dependency identities, complete contracts, target validity, and preservation behavior.
- Exact layouts and overlap of named-global views and calculated addresses.
- Address validity, memory attributes, endianness, and alignment requirements.
- Whether signed-looking arithmetic faithfully represents underlying word operations.
- Allocation, release, and ownership behavior of dependencies.
- Reliability and provenance of the final secondary-register destination.
- Whether address zero traps, invokes a platform mechanism, executes mapped code, or returns.
- Any externally meaningful error-reporting protocol.

There are no represented error checks or recovery paths. Unknown failures must not be replaced with a guessed status result or assumed successful completion.

## Evidence

The supplied static representation supports the explicit argument uses, call ordering, access widths, shifts, masks, retained-value relationships, two writes to shared word A, single read of shared word B, and final address-zero call.

Its global-overlap warning prevents an assumption that the named views are independent. The bitvector identity for F-d059 follows from the stated 32-bit widths; it does not require an assumed dependency contract.

Higher-level intent remains inferred at most, not established. No runtime tests have been performed. No execution trace, dependency implementation, memory map, or ABI specification was supplied.
