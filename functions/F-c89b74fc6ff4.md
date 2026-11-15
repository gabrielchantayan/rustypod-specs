<!-- Trusted provenance: F-c89b74fc6ff4; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-c89b74fc6ff4

## Purpose

The visible routine performs an ordered series of indirect invocations, byte accesses, and 32-bit memory accesses. It derives addresses and arguments from incoming machine state, call results, loaded words, and zero-extended selector bytes. Some writes target absolute low numeric addresses, including `0x0D` and `0x11`.

No application-level purpose is established. Scaling and repeated offsets do not establish an object model, array, device interface, initialization operation, or ownership protocol.

This contract describes the supplied static representation conditionally. That representation encounters invalid instruction data and stops without an ordinary return. It does not establish that the entire displayed path is executable or reachable.

## Interface

### Inputs and notation

There are no displayed formal parameters. Observable behavior nevertheless depends on:

- **Entry address:** a 32-bit unsigned value used as an address and arithmetic operand.
- **Entry payload:** a 32-bit unsigned value supplying two publication bytes.
- **Entry call anchor:** the incoming value from which the first indirect target is constructed.
- **Secondary outputs:** additional machine-state values exposed after particular invocations, without a recovered ABI or complete definition.

These are descriptive roles, not established parameter locations.

The following notation records retained values without assigning application meanings:

| Notation | Meaning |
|---|---|
| $q$ | Entry address logically shifted right by two |
| $m$ | The 32-bit unsigned product $q \times 0x100000$ |
| $R_n$ | The consumed 32-bit result of invocation F-*n* |
| $L_n$, $H_n$ | Low and high 32-bit halves of a consumed 64-bit result |
| $X_{30}$, $X_{64}$, $X_{132}$ | Secondary machine-state values exposed following the indicated invocation |

F-001 through F-183 identify invocation sites in order, not proven distinct callees. The 64-bit results are those of F-020, F-093, F-153, and F-155. Other consumed primary results are 32 bits.

Additional retained quantities are defined where first needed below. They refer to snapshots, not implicit rereads.

### Access widths and arithmetic

Byte reads produce values from zero through `0xFF`. Byte writes retain the low eight bits. A displayed word read or write is 32 bits, including when a loaded word is subsequently truncated to a byte.

For a byte selector $s$, writing at $s+0x11$ means an absolute numeric address in `0x11`–`0x110`. It is not a displacement relative to the selector’s storage location.

Unsigned operations are 32-bit operations; unsigned right shifts are logical, and unsigned multiplication and left shifting discard bits beyond the 32-bit result. Expressions using signed-looking call results, signed-looking address bases, or a signed conversion of a wide-result half do not independently establish overflow behavior. Their formulas below are conditional on the represented machine operations being meaningful. They are not promises about portable signed arithmetic.

Endianness, alignment requirements, address-space mapping, permissions, and fault behavior are unresolved.

### Outputs and completion

There is no established return value, success indicator, error code, or normal-return postcondition. Outputs are the ordered memory effects and dependency invocations, together with any effects those dependencies produce.

## Behavior

### Ordering contract

The dependency ledger is part of this behavior specification. Every listed invocation occurs in its listed position if preceding operations complete. A grouped ledger row expands into individual invocations in left-to-right order; it does not authorize omission or coalescing.

The memory events below occur between the identified invocation boundaries. Reads must remain fresh where specified. Retained values must remain snapshots where specified. Dependencies have unknown effects, so neither repeated writes nor invocations with no displayed arguments are redundant.

### 1. Entry publication and first clear

Before F-001, read the byte at entry address plus one. Publish the low entry-payload byte at the selected address plus `0x11`.

After F-003 and before F-004, reread the same selector location. Publish the low byte of the entry payload logically shifted right by 22. F-004 consumes $4R_3$.

F-005, F-006, F-007, and F-008 are four separate subsequent invocations with no displayed arguments. Only after F-008 is a byte read at $q \times 0x80000+1$ and zero written at the resulting selector plus `0x11`.

F-009 receives $(8q,8q,q,0)$.

### 2. Retained selector and word-to-byte publications

After F-013, reread the byte at $q \times 0x80000+1$. Retain its zero-extended value as $s$ and clear $s+0x11$. F-014 receives $4R_{13}$.

After F-018, read a full word at $s+0x74$, then write its low byte at $s+1$. F-019 receives $m$, $q$, the low 13 bits of $q$ shifted left by 17, and $s$.

F-020 supplies a wide result. F-021 receives its low half, high half, low half again, and the low 12 bits of $q$ shifted left by 18.

After F-025, form $4R_{25}$. Read a word at that base plus `0x74`, write its low byte at the base plus one, and pass the base to F-026.

After F-030 and before F-031, read a byte at $m+1$ and clear its selected address plus `0x11`. F-031 consumes $X_{30}$.

### 3. Two loaded words and another selector snapshot

Retain $a=R_{32}$; F-033 receives $a$.

After F-037, reread the byte at $m+1$ and publish the high byte of the 32-bit value $m$ at its selected address plus `0x11`. F-038 receives $4R_{37}$.

After F-042, retain $b=4R_{42}$. In order:

1. Read a word at $a+4$ and retain it as $w$.
2. Read a byte at $w+1$.
3. Clear that selector plus `0x11`.

F-043 receives $b$ three times.

After F-047, read a byte at $(w \mathbin{\mathrm{shr}} 2)+1$. Retain its zero-extended value as $t$ and clear $t+0x11$. F-048 receives $4R_{47}$.

After F-052, perform these accesses in order:

1. Read a word at $b+4$; retain it as $v$.
2. Read a word at $t+0x74$; retain it as $p$.
3. Write the low byte of $p$ at $t+1$.

F-053 receives $(p,b \ll 19,b \ll 19,4R_{52})$. F-055 receives $(R_{54},p \mathbin{\mathrm{shr}} 2)$. Retain $h=v \mathbin{\mathrm{shr}} 22$ for subsequent operations.

### 4. Word stores at scaled destinations

After F-059, read a word at $4R_{59}+0x74$, retain it as $d$, and write its low byte at $4R_{59}+1$. F-060 receives $4R_{59}$.

After F-064, write the full retained word $d$ at $4X_{64}+0x14$. F-065 receives $X_{64}$. F-067 receives $R_{66}$ and two copies of $(h \mathbin{\&} 0x1FF)\ll21$.

After F-071, write $h\ll23$ at $4R_{71}+0x14$. F-072 receives $4R_{71}$.

After F-076, first write the unsigned 32-bit product $4R_{76}$ at $16R_{76}+0x14$. Only then retain:

- $n=R_{76}\mathbin{\&}0x3FFFFFFF$;
- $k=n\mathbin{\mathrm{shr}}2$;
- $f=4R_{76}$, preserving the earlier product.

F-077 receives $n$.

After F-081, write $n$ at $4R_{81}+0x14$. F-082 receives $4R_{81}$.

### 5. Paired updates at offset `0x34`

After F-086, retain $c=8R_{81}$ and perform:

1. A byte write of the low byte of $c$ at $p+1$.
2. A word write of $R_{86}\ll2$ at $c+0x34$.

F-087 receives $R_{86}\ll2$. Retain $u=R_{86}\mathbin{\&}0x3FFFFFFF$ across the subsequent invocations. F-089 receives $R_{88}$.

After the wide result from F-093:

1. Write the low byte of $c$ at $u+1$.
2. Write $H_{93}$ at $c+0x34$.

F-094 receives $L_{93}\ll2$. The two writes to $c+0x34$ are separate observable events.

### 6. Byte fields and direct address `0x11`

After F-098, retain $e=16R_{81}$. Write the low byte of $e$ at $(u\mathbin{\mathrm{shr}}20)+0x11$, then write the low byte of $f$ at $e+0x0D$.

The first destination here is not selected by an 8-bit load: its range is `0x11`–`0x410`.

F-099 receives $R_{98}\ll2$ twice. F-101 receives $(R_{100},R_{100}\mathbin{\mathrm{shr}}3)$.

After F-105, write the low byte of $e$ directly at numeric address `0x11`, then repeat the low-byte write of $f$ at $e+0x0D$. F-106 receives $R_{105}\ll2$.

After F-110, write the full retained word $f$ at $u+0x14$, then write the low byte of $u\ll1$ at $u+0x0D$.

### 7. Larger scaling and direct selector publication

From $R_{110}$ retain:

- $g=32R_{110}$;
- $j=R_{110}\ll7$;
- $z=(R_{110}\mathbin{\&}0x07FFFFFF)\mathbin{\mathrm{shr}}17$, ranging from zero through `0x3FF`.

After F-113, write $j$ at $g+0x14$, then write the low byte of $R_{113}\ll2$ at $g+0x0D$. F-114 receives $R_{113}\ll2$.

After F-118:

1. Write the low byte of $k\ll1$ at $z+7$.
2. Read a byte at $g+1$.
3. Publish the low byte of $z$ at that selector plus `0x11`.

F-119 receives $R_{113}\ll21$ three times. Retain $r=R_{120}$; F-121 receives $r$.

After F-125, reread the byte at $g+1$, retain it as $s_2$, and clear $s_2+0x11$. F-126 receives $R_{125}\ll2$.

After F-130, write the low byte of $r\mathbin{\mathrm{shr}}3$ at $g+0x0D$, then write $s_2$ directly at numeric address `0x0D`. F-131 receives $(2s_2,0x98,0x260)$.

### 8. Second direct write and linked word updates

F-133 consumes $X_{132}\mathbin{\mathrm{shr}}3$.

F-138 receives $R_{137}\ll2$ and itself returns the consumed result $R_{138}$. Between F-138 and F-139:

1. Write the retained low byte of $r\mathbin{\mathrm{shr}}3$ at $2s_2+0x0D$.
2. Write `0xC0` directly at numeric address `0x0D`.

The immediately following invocation, F-139, receives $R_{138}\ll2$, not the earlier result of F-137.

After F-143, write $R_{143}\ll2$ at $2s_2+0x24$, then read a word at $2s_2+0x74$ and retain it as $y$. Retain $\ell=R_{143}\ll5$.

F-144 receives $(y,y\ll1,\ell,(y\mathbin{\&}0x7FFFFFFF)\mathbin{\mathrm{shr}}2)$.

After F-148, write $\ell$ at $y+0x24$. Next read a full word at $y+0x74$ and supply that loaded value to F-149.

### 9. Wide-result destinations and terminal clears

After F-153, retain $o=L_{153}\ll2$. Write the full 32-bit value $r\mathbin{\mathrm{shr}}3$ at $4H_{153}+0x14$.

F-154 receives $(o,o,o,H_{153}\mathbin{\mathrm{shr}}3)$. F-155 supplies another wide result; F-156 receives $(L_{155},H_{155},L_{155},H_{155}\mathbin{\mathrm{shr}}3)$.

After F-160, write $o$ at $4R_{160}+0x14$. F-161 receives $4R_{160}$.

After F-165, retain $x=R_{165}\ll2$. Read a byte at $y+1$ and publish the low byte of $x$ at its selected address plus `0x11`.

After F-168, reread the byte at $y+1$. Publish the low byte of $(R_{165}\mathbin{\&}0x3FFFFFFF)\mathbin{\mathrm{shr}}20$ at the newly selected address plus `0x11`. F-169 receives $R_{168}\ll2$.

After F-173, read a byte at $x\times0x400+1$ and clear its selected address plus `0x11`. F-174 receives $(x\ll3,x\ll3,x,R_{173}\ll2)$.

After F-178, reread that same selector location and clear the newly selected destination. F-179 receives $R_{178}\ll2$.

F-180, F-181, F-182, and F-183 then occur separately, with no displayed arguments. The representation subsequently stops at invalid instruction data; subsequent behavior is not established.

## Side effects

Direct effects include all specified byte and word accesses and all 183 indirect invocations.

Aliasing is unrestricted. Selector locations, low numeric destinations, loaded-word-derived addresses, and dependency-visible memory may overlap. Such overlap can affect later fresh reads.

No explicit allocation, release, ownership transfer, validation, bounds check, rollback, or error report is visible. This does not constrain dependency behavior: a dependency may allocate, release, mutate globals, alter memory, fail, or not return.

Each access requires whatever validity, alignment, and permissions the actual execution environment imposes. An access failure or nonreturning dependency can prevent every later operation. No recovery path is established.

## Dependencies

### Target construction and invocation ledger

Target construction is visible even though callee identity is unresolved.

In the table, a four-digit hexadecimal anchor suffix denotes the corresponding constant with prefix `0x0882`; for example, EA93 denotes `0x0882EA93`. The target is that anchor **plus `0xFFE` unless another offset is explicitly shown**.

K1 through K9 denote nine distinct displayed code-label anchors, ordered by their introduction in the representation. They are not interchangeable with the numeric anchors. Their absolute values are not independently established here.

A dash means **no displayed arguments**, not proof of no machine-level inputs. Parenthesized lists give displayed arguments in order. Multi-site rows pair sites and anchors positionally.

| Invocation site(s) | Target anchor(s), with exceptional offsets | Displayed arguments |
|---|---|---|
| F-001 | Entry call anchor | — |
| F-002–003 | EA93, EA95 | — each |
| F-004 | EA97 | $(4R_3)$ |
| F-005–008 | EAA3, EAA5, EAA7, EAA9 | — each |
| F-009 | EAAB + `0xFEC` | $(8q,8q,q,0)$ |
| F-010–013 | EB0B, EB0D, EB27, EB29 | — each |
| F-014 | EB2B | $(4R_{13})$ |
| F-015–018 | EB37, EB39, EB3B, EB3D | — each |
| F-019 | EB3F + `0xFEC` | $(m,q,(q\mathbin{\&}0x1FFF)\ll17,s)$ |
| F-020 | EB9F | — |
| F-021 | EBA1 | $(L_{20},H_{20},L_{20},(q\mathbin{\&}0xFFF)\ll18)$ |
| F-022–025 | EBB7, EBB9, EBBB, EBBD | — each |
| F-026 | EBBF | $(4R_{25})$ |
| F-027–030 | EBCB, EBCD, EBCF, EBD1 | — each |
| F-031 | EBD3 + `0xFEC` | $(X_{30})$ |
| F-032 | EC17 | — |
| F-033 | EC19 | $(a)$ |
| F-034–037 | EC2F, EC31, EC33, EC35 | — each |
| F-038 | EC37 | $(4R_{37})$ |
| F-039–042 | EC43, EC45, EC47, EC49 | — each |
| F-043 | EC4B | $(b,b,b)$ |
| F-044–047 | ECC3, ECC5, ECC7, ECC9 | — each |
| F-048 | ECCB | $(4R_{47})$ |
| F-049–052 | ECD7, ECD9, ECDB, ECDD | — each |
| F-053 | ECDF + `0xF90` | $(p,b\ll19,b\ll19,4R_{52})$ |
| F-054 | ED43 | — |
| F-055 | ED45 | $(R_{54},p\mathbin{\mathrm{shr}}2)$ |
| F-056–059 | ED5B, ED5D, ED5F, ED61 | — each |
| F-060 | ED63 | $(4R_{59})$ |
| F-061–064 | ED6F, ED71, ED73, ED75 | — each |
| F-065 | ED77 + `0xFEC` | $(X_{64})$ |
| F-066 | EDBB | — |
| F-067 | K1 | $(R_{66},(h\mathbin{\&}0x1FF)\ll21,(h\mathbin{\&}0x1FF)\ll21)$ |
| F-068–071 | EDD3, EDD5, EDD7, EDD9 | — each |
| F-072 | EDDB | $(4R_{71})$ |
| F-073–076 | EDE7, EDE9, EDEB, EDED | — each |
| F-077 | EDEF | $(n)$ |
| F-078–081 | EE67, EE69, EE6B, EE6D | — each |
| F-082 | EE6F | $(4R_{81})$ |
| F-083–086 | EE7B, EE7D, EE7F, EE81 | — each |
| F-087 | EE83 + `0xFEC` | $(R_{86}\ll2)$ |
| F-088 | K2 | — |
| F-089 | K3 | $(R_{88})$ |
| F-090–093 | EEFF, EF01, EF03, EF05 | — each |
| F-094 | EF07 | $(L_{93}\ll2)$ |
| F-095–098 | EF13, EF15, EF17, EF19 | — each |
| F-099 | EF1B + `0xFEC` | $(R_{98}\ll2,R_{98}\ll2)$ |
| F-100 | EF57 | — |
| F-101 | EF59 | $(R_{100},R_{100}\mathbin{\mathrm{shr}}3)$ |
| F-102–105 | EF6F, K4, EF73, EF75 | — each |
| F-106 | EF77 | $(R_{105}\ll2)$ |
| F-107–110 | K5, EF85, EF87, EF89 | — each |
| F-111–113 | EF8B, EFE3, EFE5 | — each |
| F-114 | EFE7 | $(R_{113}\ll2)$ |
| F-115–118 | EFF3, EFF5, EFF7, EFF9 | — each |
| F-119 | EFFB + `0xEE8` | $(R_{113}\ll21,R_{113}\ll21,R_{113}\ll21)$ |
| F-120 | K6 | — |
| F-121 | F049 | $(r)$ |
| F-122–125 | F05F, F061, F063, F065 | — each |
| F-126 | F067 | $(R_{125}\ll2)$ |
| F-127–130 | F073, F075, F077, F079 | — each |
| F-131 | F07B + `0xF60` | $(2s_2,0x98,0x260)$ |
| F-132 | F0DB | — |
| F-133 | F0DD | $(X_{132}\mathbin{\mathrm{shr}}3)$ |
| F-134–137 | F0F3, F0F5, F0F7, F0F9 | — each |
| F-138 | F0FB + `0xE08` | $(R_{137}\ll2)$ |
| F-139 | K7 | $(R_{138}\ll2)$ |
| F-140–143 | F107, F109, F10B, F10D | — each |
| F-144 | F10F | $(y,y\ll1,\ell,(y\mathbin{\&}0x7FFFFFFF)\mathbin{\mathrm{shr}}2)$ |
| F-145–148 | F15F, F161, F163, F165 | — each |
| F-149 | F167 | Loaded word at $y+0x74$ |
| F-150–153 | F173, F175, F177, K8 | — each |
| F-154 | K9 + `0xFEC` | $(o,o,o,H_{153}\mathbin{\mathrm{shr}}3)$ |
| F-155 | F1B7 | — |
| F-156 | F1B9 | $(L_{155},H_{155},L_{155},H_{155}\mathbin{\mathrm{shr}}3)$ |
| F-157–160 | F1CF, F1D1, F1D3, F1D5 | — each |
| F-161 | F1D7 | $(4R_{160})$ |
| F-162–165 | F1E3, F1E5, F1E7, F1E9 | — each |
| F-166–168 | F1EB, F26B, F26D | — each |
| F-169 | F26F | $(R_{168}\ll2)$ |
| F-170–173 | F27B, F27D, F27F, F281 | — each |
| F-174 | F283 + `0xFEC` | $(x\ll3,x\ll3,x,R_{173}\ll2)$ |
| F-175–178 | F2E3, F2E5, F2FF, F301 | — each |
| F-179 | F303 | $(R_{178}\ll2)$ |
| F-180–183 | F30F, F311, F313, F315 | — each |

For every dependency, the actual ABI, complete inputs, global-state effects, memory effects, ownership behavior, failures, and termination remain unknown. Matching target expressions would not alone justify treating invocations as effect-free or interchangeable.

## Acceptance examples

These are conditional static examples, not executed tests.

1. **Initial truncation and fresh selection.** With entry payload `0x12345678` and first selector `0x20`, the initial write stores `0x78` at `0x31`. If the selector becomes `0x40` before the post-F-003 reread, that later write stores `0x48` at `0x51`.

2. **Required invocation sequence.** If F-004 returns, F-005 through F-008 must occur individually before the first scaled-address selector clear. A behavior that jumps directly from F-004 to the clear omits observable operations.

3. **Known target construction.** F-001 uses entry call anchor plus `0xFFE`. F-002 uses `0x0882EA93 + 0xFFE`; F-009 instead uses `0x0882EAAB + 0xFEC`. Unresolved callee identity does not erase these relationships.

4. **Full-width source access.** If a specified word load yields `0xA1B2C3D4`, its subsequent byte publication stores `0xD4`. Acceptance requires the full word access, not an assumed equivalent byte load.

5. **Selector bounds.** An 8-bit selector of `0xFF` chooses address `0x110`. By contrast, the phase-6 masked-result-derived destination can reach `0x410`.

6. **Wide-result split.** A wide result `0x1122334455667788` has low half `0x55667788` and high half `0x11223344`. Repeated low-half arguments remain repeated arguments.

7. **Snapshot preservation.** If a dependency changes memory at $t+0x74$ after the load defining $p$, later uses of $p$ still consume the retained word, not a fresh load.

8. **Result forwarding across byte writes.** If F-138 returns `3`, F-139 receives `12`, subject to the represented arithmetic. Both intervening byte writes precede F-139. Substituting the result of F-137 violates the dependency.

9. **Aliased direct writes.** If $s_2=0$, the two writes between F-138 and F-139 both target `0x0D`: the retained shifted byte first, then `0xC0`. They remain distinct ordered writes.

10. **Failure boundary.** If the first low-address publication faults, no subsequent invocation or write is required. No rollback or recovery is established.

## Uncertainties

The following remain **documented_only**:

- Validity of the entry point, instruction interpretation, and complete displayed path.
- Actual calling convention and intended meaning of incoming machine state.
- Definitions and provenance of secondary outputs beyond their visible consumption.
- Absolute values of the opaque code-label anchors.
- Overflow behavior for signed-looking shifts, products, and address calculations.
- Endianness, alignment, memory permissions, and low-address mapping.
- Actual dependency identities, effects, ownership rules, and termination.
- Application-level meaning of offsets, selectors, and scaled values.
- Runtime consequences of the invalid-instruction boundary and any behavior beyond it.

Address or index manipulation is an inferred interpretation of the scaling patterns, not an established data-structure contract.

## Evidence

Established static evidence consists of the supplied representation’s access widths, arithmetic operands, target-anchor constructions and offsets, all 183 displayed indirect invocations, consumed-result dependencies, memory-effect ordering, and invalid-instruction truncation.

The representation is untrusted evidence rather than an authoritative executable specification. The obligations above are conditional on it accurately representing the routine.

No runtime tests, execution traces, dependency inspection, independent instruction validation, or memory-layout validation have been performed.
