<!-- Trusted provenance: F-8c15b4ec33cd; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8c15b4ec33cd

## Purpose

F-8c15b4ec33cd performs a fixed, state-dependent sequence of indirect calls, memory transfers, indexed byte writes, global-word updates, and a terminal call through address zero.

The supplied representation establishes these operations, but not an application-level purpose. It does not establish a supervisor service, an object model, or an ownership-management protocol. Repeated call patterns may suggest generated or encoded execution; that is inferred intent, not a behavioral requirement.

This contract describes the explicit behavior conditional on each preceding operation completing. It does not supply missing dependency contracts or guarantee that execution returns.

## Interface

### Invocation and implicit inputs

The recovered interface has no explicit parameters and no explicit return value. It nevertheless consumes incoming execution state:

- A 32-bit unsigned word used as an argument after logical right shift by 23.
- A second 32-bit unsigned word whose bits 22–29 supply the first byte write.
- An address used for a byte read at offset 1.
- A third 32-bit unsigned word used to derive a byte-read address by logical right shift by 9, then addition of 1.
- Incoming link context used to construct the first indirect-call target.
- An auxiliary 32-bit register value captured during execution.
- An auxiliary address used for a late byte-zeroing operation.

The actual calling convention, entry-state obligations, and provenance of the auxiliary values are **documented_only**. Their appearance in the representation does not establish ordinary parameters or identify which dependency produces them.

### Outputs and completion

There is no explicit scalar result. Observable outputs consist of:

- Ordered dependency calls and their recovered arguments.
- Explicit memory reads and writes.
- Two writes to one designated global word.
- An attempted call through address zero with three zero arguments.

A return follows that last call in the representation. Normal completion therefore requires that the call through address zero return. Faulting, nonreturning, or environment-specific behavior remains possible.

### Widths and arithmetic

The recovered operations distinguish 8-bit bytes, 16-bit halfwords, 32-bit words, and three 64-bit call results. Each 64-bit result is split into low and high 32-bit halves. A terminal operation combines a 24-bit read with an 8-bit result.

Byte stores retain bits 0–7. Thus, a byte store of an unsigned word shifted right by 22 retains original bits 22–29.

Unsigned right shifts are logical. Descriptions of wrapping shifts, scaling, and addresses use a **32-bit bit-vector interpretation where explicitly stated**. The representation also contains signed-looking intermediate arithmetic; its portable signed-language semantics, overflow behavior, and correspondence to machine instructions are not established.

In particular, under 32-bit modular arithmetic, shifting a word left by 2, logically right by 2, and then left by 16 produces the same 32-bit value as shifting the original word left by 16. Both retain only original bits 0–15 in the final result. This identity does not resolve whether an earlier signed-looking shift has defined language-level semantics.

### Continuation preconditions

To execute the whole sequence, the environment must provide usable incoming state, callable targets, and accessible memory for every described access. No explicit validation enforces these requirements. Alignment, permissions, executable mappings, and fault handling are **documented_only**.

## Behavior

### Dependency notation and ordering

F-1001 through F-1112 identify successive call occurrences, not proven-distinct functions. Different IDs may resolve to the same target.

An “unparameterized” call has no explicit arguments in the recovered representation; it may still consume registers or other implicit state. Unless specified otherwise, ordinary recovered results are 32-bit words.

All ordering below is conditional on preceding operations completing. Unknown dependencies may modify memory between explicit accesses. A retained value denotes an already captured value, not a later reread of its source.

### 1. Initial calls and first byte write

F-1001 is invoked through a target formed from incoming link context plus 4,094 address units.

F-1002 receives the first incoming unsigned word logically shifted right by 23. Unparameterized calls F-1003 through F-1005 follow; F-1006 then supplies a word result.

Before forwarding that result, the routine reads an unsigned byte at the incoming address plus 1. It uses the byte’s zero-extended numeric value plus 17 as a destination address and stores the low byte of the second incoming unsigned word shifted right by 22.

The destination is therefore in the numeric range 17–272, inclusive, under ordinary byte addressing. It is not computed as an offset from the incoming address.

F-1007 receives the F-1006 result shifted left by 2.

### 2. First repeated sequence and auxiliary capture

Unparameterized F-1008 through F-1010 precede result-producing F-1011. F-1012 receives two copies of the F-1011 result shifted left by 2.

F-1013 supplies a word forwarded unchanged to F-1014. Unparameterized F-1015 follows.

The routine derives a read address from the third incoming unsigned word by logical right shift by 9 and addition of 1. It reads an unsigned byte there and writes zero at the address equal to that byte plus 17.

The auxiliary 32-bit register value is then captured. F-1016 receives three copies of that value. The representation does not establish which preceding call produced it.

### 3. Repeated byte read and first global write

F-1017 supplies a word. F-1018 receives that word followed by the F-1013 result logically shifted right by 23.

Unparameterized F-1019 through F-1021 precede result-producing F-1022.

The context-derived address from section 2 is read again. This is a fresh byte read. The resulting zero-extended byte is retained as a small numeric base, and zero is written at that base plus 17.

F-1023 receives the F-1022 result shifted left by 2. Unparameterized F-1024 through F-1026 precede result-producing F-1027. F-1028 receives two copies of its result shifted left by 2.

F-1029 supplies a word. The routine writes it to the designated global word before passing the same captured value to F-1030. Unparameterized F-1031 follows.

Using the retained small base, the routine reads a full 32-bit word at offset 116, then stores that word’s low byte at offset 1. The source access is word-wide even though the destination is byte-wide.

F-1032 receives the captured auxiliary value shifted left by 11.

### 4. First wide result and repeated word-to-byte transfers

F-1033 supplies a 64-bit result. F-1034 receives:

1. Its low 32-bit half.
2. Its high 32-bit half.
3. The F-1029 result logically shifted right by 23.

Unparameterized F-1035 through F-1037 precede result-producing F-1038. Four times its result becomes a retained scaled base.

The routine reads a full word at this base plus 116, stores its low byte at this base plus 1, and passes the base to F-1039.

Unparameterized F-1040 through F-1042 precede result-producing F-1043. That result is retained both unshifted and shifted left by 2.

The routine then performs another full-word read at the **earlier F-1038-derived base** plus 116 and another low-byte store at that base plus 1. It neither substitutes the F-1043 result as the base nor reuses the previously read word.

F-1044 receives, in order:

1. The captured auxiliary value shifted left by 12.
2. The F-1043 result shifted left by 2.
3. A second copy of that shifted result.
4. The F-1043 result shifted left by 18.

### 5. Capturing and distributing a full word

F-1045 supplies a word forwarded to F-1046. F-1047 supplies a word forwarded to F-1048.

Unparameterized F-1049 through F-1051 precede result-producing F-1052. Four times its result forms a new base.

The routine reads a full word at this base plus 116 and retains the complete value for later use. It stores the word’s low byte at this base plus 1, then passes the base to F-1053.

Unparameterized F-1054 through F-1056 precede result-producing F-1057. The retained full word is stored at sixteen times that result plus 20.

F-1058 receives the retained shifted F-1043 result logically shifted right by 2, then shifted left by 16. The intermediate right shift is unsigned. Under 32-bit modular arithmetic, this argument equals the original F-1043 result shifted left by 16: the final shift discards the original top two bits as well as original bits 16–29. Their loss in the earlier shift therefore does not prevent simplification. Unresolved signed-language behavior of the original left shift remains a separate issue.

F-1059 supplies a word forwarded to F-1060. F-1061 supplies a word forwarded to F-1062.

Unparameterized F-1063 through F-1065 precede result-producing F-1066. The same retained full word is stored at four times this result plus 20. F-1067 then receives four times the result.

### 6. Further word writes, clearing, and global-derived reads

Unparameterized F-1068 through F-1070 precede result-producing F-1071.

The F-1071 result shifted left by 2 is stored as a full word at an address formed by clearing the low two bits of the retained, already-shifted F-1043 result and adding 20. F-1072 receives two copies of the stored value.

F-1073 supplies a word forwarded to F-1074. F-1075 supplies a word forwarded to F-1076.

Unparameterized F-1077 through F-1079 precede result-producing F-1080. A full 32-bit zero is stored at four times its result plus 20. F-1081 receives four times the result.

Unparameterized F-1082 through F-1084 precede F-1085, which supplies a 64-bit result.

The routine shifts its low half left by 2 and stores that word at four times its high half plus 20. It then reads a separate global word and retains its value as an address-like base.

F-1086 receives four arguments: the shifted low half, the high half, and two further copies of the shifted low half.

Unparameterized F-1087 through F-1089 precede result-producing F-1090. The same retained shifted low half is stored at four times this new result plus 20.

Only after that store does the routine read a full word at the retained global-derived base plus 20. The base is the value captured before F-1086; the global storage is not reread here.

F-1091 receives four times the F-1090 result.

### 7. Final indexed byte updates

Unparameterized F-1092 through F-1094 precede F-1095, which supplies a 64-bit result.

The full word retained in section 5 determines a byte-read address: logically shift it right by 23 and add 1. The routine reads an unsigned byte at that address. At the address equal to that byte plus 17, it stores the low byte of the word read through the global-derived base in section 6.

F-1096 receives three copies of the high half of the F-1095 result, followed by its low half shifted left by 2.

F-1097 supplies a word forwarded to F-1098. Unparameterized F-1099 through F-1101 precede result-producing F-1102.

A second indexed update uses the retained low half of F-1095. Its top two bits are cleared, the remaining value is logically shifted right by 21, and 1 is added to form a byte-read address. After reading the byte, the routine writes at that byte’s value plus 17. The stored byte comprises bits 22–29 of the retained global-derived word.

F-1103 receives the F-1102 result shifted left by 2.

Unparameterized F-1104 through F-1106 precede result-producing F-1107. F-1108 receives two copies of its result shifted left by 2. Unparameterized F-1109 follows.

### 8. Terminal operations

F-1110 supplies a word forwarded to F-1111. F-1112 supplies an 8-bit result.

The routine then performs these operations in order:

1. Stores one zero byte through the recovered auxiliary address.
2. Reads the designated 24-bit source.
3. Constructs a 32-bit word with that read as bits 8–31 and the F-1112 result as bits 0–7.
4. Stores the constructed word to the same designated global word written in section 3.
5. Stores a 16-bit zero at an address obtained by clearing the top two bits of the retained F-1095 low half, then logically shifting right by 7.
6. Attempts a call through address zero with three zero arguments.

The auxiliary byte clear precedes the 24-bit read. If those locations overlap, the read may reflect that clear.

## Side effects

Explicit memory effects include low-numeric-address byte writes, full-word reads at offset 116, full-word writes at offset 20 with different scale factors, a separate global read, two designated global-word writes, and a final halfword clear.

These accesses do not establish one common structure layout:

- Scaling a result by 4 differs from scaling it by 16.
- A loaded byte may become a numeric address base.
- A full-word access remains observable even when only its low byte is used.
- Fresh reads may observe changes made by dependencies or earlier stores.
- Global storage, indirect storage, and differently sized accesses may overlap.

There is no explicit allocation, release, ownership transfer, locking, rollback, or error-status protocol. None may be inferred from the indirect calls.

The representation contains no conditional branches, loops, null checks, bounds checks, alignment checks, or error-return branches. Failure can leave earlier effects visible; no transactional guarantee is established.

## Dependencies

F-1001 through F-1112 denote the 112 ordered call occurrences above. Their identities and contracts are unknown. The terminal address-zero call is additional.

Most targets are formed from fixed code-location anchors and displacements. F-1001 instead uses incoming link context. This establishes target-construction forms, not verified function identities.

Continuation requires dependencies to return and provide the recovered result widths where used. It does not establish that they preserve globals, preserve referenced memory, own or release storage, avoid reentrancy, or lack hidden arguments.

A call described as forwarding a value receives the captured result specified above, even if intervening memory changes affect its original source.

## Acceptance examples

These are conditional static expectations, not executed tests. Calls and accesses must complete; dependencies must not alter the illustrated inputs unless an example explicitly allows that.

- **Early byte addressing:** If the byte at the incoming address plus 1 is 5, the first store targets address 22. If the supplying word is 4,194,304, the stored byte is 1.

- **Fresh repeated read:** If the first context-derived byte read yields 3 and the second yields 7, the corresponding zero stores target addresses 20 and 24. Reusing the earlier byte would violate the contract.

- **Word-width source access:** If a word read at a retained base plus 116 yields 0x123456AB, the byte stored at base plus 1 is 0xAB. This example specifies a numeric word result, not a byte-order assumption.

- **Different destination scales:** If F-1057 returns 3, its distribution store targets address 68. If F-1066 returns 3, its distribution store targets address 32.

- **Retained full word:** If a dependency changes the section 5 source location after capture, subsequent distribution stores still use the captured word, provided execution continues according to the recovered model.

- **F-1058 bit-vector equivalence:** If F-1043 returns 0xC0001234, its 32-bit left shift by 2 yields 0x000048D0. Logical right shift by 2 followed by a 32-bit left shift by 16 yields 0x12340000. Directly shifting the original result left by 16 yields the same word. This is a modular-arithmetic example, not a guarantee of defined signed-language evaluation.

- **Split wide result:** If F-1085 returns low half 5 and high half 7, the routine stores 20 at address 48 before reading the separate global base and invoking F-1086.

- **Masked terminal address:** If the F-1095 low half is 0xC0000080, the final halfword-zero address is 1. Successful access depends on the environment’s permissions and alignment behavior.

- **Terminal global construction:** If F-1112 yields 0x5A and the subsequent 24-bit read yields 0x123456, the designated global receives 0x1234565A. The auxiliary byte clear has already occurred.

## Uncertainties

The following remain **documented_only**:

- Accuracy of the recovered boundary, indirect-call interpretation, and execution model.
- Actual ABI, implicit inputs, register liveness, and auxiliary-value provenance.
- Dependency identities, hidden arguments, preservation rules, and side effects.
- Actual storage locations and overlap relationships of the globals.
- Memory-map validity, permissions, alignment, byte order, and device-memory semantics.
- Signed-language interpretation of shifts and arithmetic, address overflow, and fault behavior.
- Whether address zero faults, contains executable content, triggers environment-specific handling, or returns.

The 32-bit identity described for F-1058 is established algebraically under its stated bit-vector model. It does not resolve these separate execution and signed-semantics uncertainties.

## Evidence

Established static evidence consists solely of the supplied recovered representation: explicit access widths, data dependencies, arithmetic, call arguments, call ordering, and terminal operations.

Opaque dependency IDs and descriptive storage roles are specification notation, not additional symbol or application evidence. Inferred intent is explicitly separated from the contract; unresolved behavior remains documented_only.

No runtime tests or execution observations have been performed. The acceptance examples are conditional consequences of the static representation, not observed outcomes.
