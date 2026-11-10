<!-- Trusted provenance: F-eea26ecf3091; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-eea26ecf3091

## Purpose

F-eea26ecf3091 is a state-driven operation that accepts an address, conditionally reads 32-bit fields through retained addresses, invokes two dependencies, may write computed bytes, and may return a pair of packed 32-bit words.

The supplied static representation establishes the effects of individual dispatch cases. The static-data contents needed to determine an actual dispatch sequence are absent. Reachability, execution counts, termination, and a complete memory-access range therefore remain unresolved.

No application-level purpose is established. In particular, the dependency calls and byte stores do not establish allocation, initialization, transformation, or destruction semantics.

## Interface

### Input and accessed layout

The sole argument is the **input address**. Two fields may be accessed through an active address:

| Byte offset | Width | Established use |
|---|---:|---|
| 0 | 32 bits | Arithmetic input in case 8 |
| 4 | 32 bits | Dependency argument in case 0; transition predicate in case 2 |

These accesses establish neither an object’s total size nor the fields’ semantic types. No explicit size, capacity, ownership flag, or type tag accompanies the address.

Validity requirements are path-dependent. An execution returning without field accesses need not read either field. Conversely, supplying a valid input address is not sufficient for safe execution: the initial active address is null, and the retained destination also begins null.

### Return value

A returning execution produces a 64-bit bit pattern containing high word \(H\) and low word \(L\):

\[
R=(\operatorname{zext}_{64}(H)\ll32)
 \mathbin{|}\operatorname{zext}_{64}(L).
\]

Both words initially contain zero. Packing does not sign-extend \(L\) into \(H\). No caller-level interpretation or error-status convention is supplied.

### Explanatory state

The following symbols are specification notation, not implementation identifiers.

| State | Width or kind | Initial value |
|---|---|---|
| \(A,B\) | Cached 32-bit arithmetic inputs | Separate static-cell observations at entry |
| \(H,L\) | 32-bit result words | Zero |
| \(U\) | Unsigned 32-bit working word | Zero |
| \(J,M\) | 32-bit arithmetic state | Zero |
| Carried address | Address | Null |
| Candidate address | Address | Null |
| Destination address | Address | Null |
| Transition base | 32-bit dispatch state | Static-cell observation at entry |

Arithmetic formulas below describe low-32-bit results where explicitly qualified. They do not establish portable behavior for overflowing signed expressions, signed shifts, or implementation-dependent conversions.

## Behavior

### Entry and dispatch

Entry observes the static cell supplying \(B\), then the cell supplying \(A\). These values are cached; dependency calls do not cause their source cells to be reloaded. A separate static cell supplies the initial transition base.

Each dispatch iteration has this order:

1. The carried address becomes the active address.
2. The transition base and a fresh dispatch-bias observation form the selector.
3. A reset-base cell is read and provisionally supplies the next transition base.
4. The original input address provisionally supplies the next carried address.
5. The selected case executes.

Cases 0 through 12 are explicit. Other selectors take the default path.

Every case except 6, 9, and the default path explicitly retains the active address for the next iteration. Case 6 returns. Case 9 and the default path retain the provisional reset base and original input address, then continue dispatching.

Consequently, the first iteration uses a null active address, not the input address. A reset path can make the original input address active on the following iteration.

### Case-local effects

A “transition cell” is the static cell associated with the specified case. Its value is not supplied.

| Case | Established effects |
|---|---|
| 0 | Reads the second field through the active address and passes its 32-bit value to F-f4ddf31692bb. After normal return, reads its transition cell. |
| 1 | Sets \(H=80\), then sets \(J\) to \(B\) multiplied by the represented 32-bit value \(-48\). Reads the transition cell shared with case 7. |
| 2 | Forms a field-dependent transition using two references to the second field and two references to its transition cell. |
| 3 | Sets \(L\) to the low 32 bits of \(A U\), then reads its transition cell. |
| 4 | Reads its transition cell without changing arithmetic state or result words. |
| 5 | Computes \(U\) using the transformation below, updates \(M\), then makes two ordered transition-cell references and selects according to \(U\)’s sign bit. |
| 6 | Returns the packed \(H,L\) result. This iteration’s dispatch-bias and reset-base reads have already occurred. |
| 7 | Updates \(H\), computes \(U\), stores one byte, updates \(J\), then reads its transition cell. |
| 8 | Computes \(U\) from two references to the first field, makes two ordered transition-cell references, records the active address as the candidate, and selects a transition according to \(U\)’s sign bit. |
| 9 | Continues using the provisional reset base and original input address. |
| 10 | Reads its transition cell, retains the active address, then copies the candidate address into the destination address. |
| 11 | Calls F-5e3309c9d083 with the candidate address. After normal return, sets \(U=0\), then reads its transition cell. |
| 12 | Reads the transition cell shared with case 11, then loads \(U\) from another static cell. |
| Other | Continues using the provisional reset base and original input address. |

Except where stated otherwise, cases retain the active address. Both dependency return values are ignored.

### Case 2: field-dependent transition

With equal observations of the second field and equal observations of the transition cell, this case selects:

- Transition-cell value plus 10 when the field is zero.
- Unadjusted transition-cell value when the field is nonzero.

There are two field references and two transition-cell references within the expression. Their relative operand evaluation order is not established. Neither pair is established as one captured value.

If repeated references observe different values, the result need not equal either stable-observation alternative. The stable description is not a concurrency or special-memory guarantee.

### Case 5: arithmetic transformation and repeated observations

Case 5 uses cached \(A,B\), current \(H,J\), and seven arithmetic static-data roles:

| Static-data role | References in the arithmetic expression |
|---|---:|
| Product coefficient | One |
| First mask | One |
| Correction constant | Two |
| Second mask | One |
| Shift mask | Three |
| Third mask | One |
| Multiplier for \(M\) | One |

The correction constant’s references separately supply a subtraction operand and a predecessor operand. The shift mask’s references separately supply the right-shift operand, left-shift operand, and operand combined with a working word by inclusive OR.

No relative operand evaluation order is established among those repeated arithmetic references. Their role names identify shared cells, not guaranteed identical observations. In particular, the three shift-mask observations must not automatically be treated as one value.

The following mathematical characterization distinguishes these observations. It is conditional on 32-bit bit-pattern arithmetic; it is not a portable signed-language guarantee. All additions, subtractions, products, complements, and bitwise combinations in this characterization produce 32-bit words.

Define:

- \(P\): the low word of \(ABH\).
- \(T\): one plus the low word of \(BH\) times the product coefficient.
- \(D=P-1\).
- \(F,N,V\): the first, second, and third mask observations.
- \(C_{\mathrm{sub}}\): the correction-constant observation used as a subtraction operand.
- \(C_{\mathrm{pred}}\): the correction-constant observation whose predecessor is used.
- \(S_{\mathrm{right}},S_{\mathrm{left}},S_{\mathrm{or}}\): the three shift-mask observations, identified by use rather than chronological order.

Let \(Q\) combine the right-shifted \(S_{\mathrm{right}}\) and left-shifted \(S_{\mathrm{left}}\):

\[
Q=
\operatorname{rightShift}_{1}(S_{\mathrm{right}})
\mathbin{|}
(S_{\mathrm{left}}\ll31).
\]

The right shift here is the represented signed right shift. Its target-independent semantics are unresolved; it is not established as a logical shift or rotation.

Define an arithmetic residual:

\[
E=(T\mathbin{|}S_{\mathrm{or}})-P-Q-1.
\]

A **bit-completion word** \(K\) has every bit of \(D\) set. At positions where \(D\) is clear, it takes the parity of the corresponding bits of \(N,V,E\):

\[
K=D\mathbin{|}
\bigl(\neg D\mathbin{\&}(N\mathbin{\oplus}V\mathbin{\oplus}E)\bigr).
\]

Define a **correction word**:

\[
C=(C_{\mathrm{pred}}-1)
 \mathbin{\oplus}
 \bigl(C_{\mathrm{sub}}+(T\mathbin{|}F)-P-1\bigr).
\]

The resulting working word is:

\[
U=K-P-C\pmod{2^{32}}.
\]

This characterization does not invent an order for the arithmetic references. If both correction-constant references observe the same value, their two symbols may be replaced by one value. Likewise, the shift-mask symbols may be collapsed only if all three references observe the same value. A characterization using one value per role requires **both** the bit-pattern arithmetic interpretation and these equal-observation conditions.

After computing \(U\), case 5 sets \(M\) to the current \(J\) multiplied by the separate static multiplier. It then performs two ordered references to its transition cell:

1. The first supplies an alternative adjusted by \(-3\).
2. The second supplies an unadjusted alternative.

Both references precede selection. Under the represented arithmetic-sign-shift interpretation, a clear bit 31 of \(U\) selects the first alternative; a set bit 31 selects the second. Calling these “one base minus 3” and “the same base” additionally requires equal transition-cell observations.

### Case 7: byte generation

The high word changes first:

\[
H\leftarrow A(BH+C_h),
\]

where \(C_h\) is a static-cell observation. Next:

\[
U\leftarrow ABH,
\]

using the updated \(H\).

The store address combines a represented 32-bit integer conversion of the destination address with unsigned 32-bit offset \(U\). The addition is unsigned 32-bit addition after that conversion. Conversion back to an address, truncation, and validity depend on the address model.

Under the low-bit interpretation, the stored byte is:

\[
\bigl((J-17)(-77)
 [B((M-1)\mathbin{|}139)+69+J]123\bigr)
 \bmod256.
\]

Narrow character conversions occur in the represented expression. This formula describes the stored low byte under the stated interpretation, not an unrestricted claim about wider intermediate values.

Only after the store does the case update:

\[
J\leftarrow B\bigl(A(J+B)\bigr).
\]

It then reads its transition cell.

A failed store does not establish completion of the later update or transition read. There is no shown bounds check, null check, capacity check, allocation, or terminator write. The value 80 assigned in case 1 is not evidence of destination capacity.

### Case 8: field computation and candidate retention

For equal first-field observations \(X\), the computation is:

\[
U=\neg\bigl(
(ABX+C_x)\mathbin{|}(C_yBX+C_z)
\bigr),
\]

with three separate static-data roles \(C_x,C_y,C_z\).

The first field appears twice. Their relative operand evaluation order is not established, and the single-\(X\) formula requires equal observations.

After computing \(U\), the case reads its transition cell twice in order:

1. The first observation supplies an alternative adjusted by \(+10\).
2. The second supplies an unadjusted alternative.

It then records the active address as the candidate. Under the represented sign-shift interpretation, a clear bit 31 selects the first alternative; a set bit 31 selects the second. This is a sign-bit selection, not a zero test.

## Side effects

### Direct memory effects

The function can read arithmetic and dispatch static cells, read 32-bit fields at offsets 0 and 4 through active addresses, and write one byte per case-7 execution.

No contiguous output range or maximum write count is established. Repeated stores may target the same address, different addresses, or invalid addresses.

Statement-level ordering is significant:

- Case 10 reads its transition cell before updating the destination.
- Case 12 reads its transition cell before loading \(U\).
- Cases 5 and 8 obtain both transition alternatives before selection.
- Case 5 updates \(M\) after computing \(U\), before obtaining those alternatives.

Repeated expression-level references are separately documented where relevant. This does not establish actual machine-load counts, volatile semantics, or synchronization.

### Dependency effects and ownership

F-f4ddf31692bb receives a field value, not the field’s address. F-5e3309c9d083 receives the candidate address, which may still be null.

Neither dependency’s contract is available. Global preservation, preservation of pointed-to storage, and ownership behavior cannot be assumed. Transition cells are read after normal dependency return, so dependency mutations may influence subsequent dispatch. Cached \(A,B\) remain the entry observations.

No direct allocation, release, or ownership transfer is established. Calling F-5e3309c9d083 does not clear the candidate or destination. If it invalidates storage also retained as the destination, a later byte store requires an external lifetime guarantee absent from this evidence.

## Dependencies

| Opaque function ID | Argument | Established interaction | Unknown contract |
|---|---|---|---|
| F-f4ddf31692bb | Active address’s second 32-bit field | Called in case 0; result ignored | Meaning, accepted values, side effects, ownership, failure behavior |
| F-5e3309c9d083 | Candidate address | Called in case 11; result ignored | Null handling, mutation, allocation or release, global effects, failure behavior |

Effects described after either call require normal return. No handling is established for exceptions, nonlocal transfers, termination, or dependency-internal errors.

## Acceptance examples

These are conditional static criteria, not runtime tests. Each requires the specified cases to be selected and necessary accesses to complete. Arithmetic examples additionally require the stated bit-pattern interpretation.

1. **Immediate return.** If the first case is 6, the result is zero. Dispatch-bias and reset-base reads still occur. No input-field access is required.

2. **High-word initialization.** Cases 1 then 6 return \(80\times2^{32}\), with low word zero. The accompanying \(J\) update does not alter the packed result.

3. **Loaded low-word input.** If case 12 loads \(W\), followed by cases 3 and 6 without intervening changes, the low result word is \(AW\bmod2^{32}\). The case-12 transition read precedes its load of \(W\).

4. **Destination copying.** If case 8 records address \(P_0\), then case 10 executes, the destination becomes \(P_0\). Case 10 first reads its transition cell; it does not load a replacement pointer from storage at \(P_0\).

5. **Computed byte.** With \(A=B=1\), \(H=J=M=0\), \(C_h=0\), and a valid destination, case 7 uses offset zero and stores byte 124:
   \[
   (-17\times-77\times68\times123)\bmod256=124.
   \]
   After the successful store, \(J=1\).

6. **Sign selection rather than zero selection.** If case 8 produces \(U=1\), it selects its first transition observation plus 10. If it produces \(U=\text{0x80000000}\), it selects its second observation unadjusted.

7. **Distinct transition observations.** If case 5 observes 20 first and 40 second, a clear sign bit selects 17; a set sign bit selects 40. The corresponding case-8 alternatives are 30 and 40.

8. **Distinct correction-constant observations.** In case 5, suppose \(A=B=H=0\), all mask observations are zero, \(C_{\mathrm{sub}}=0\), and \(C_{\mathrm{pred}}=1\). Then \(P=0\), \(T=1\), \(K=\text{0xffffffff}\), and \(C=0\), giving \(U=\text{0xffffffff}\). Replacing both correction observations by zero instead gives \(U=0\); replacing both by one gives \(U=\text{0xfffffffe}\). Thus collapsing the repeated cell to either observation changes the result in this example.

9. **No implicit null protection.** An initial selection of case 0, 2, or 8 attempts a field access through the initially null active address. No guarded success or defined error result is established.

10. **Reset behavior.** Case 9 or the default path carries the original input address into the next iteration. Neither path returns.

## Uncertainties

The following remain **documented_only**:

- **Reachability and termination:** missing static contents prevent a concrete sequence, iteration bound, or guaranteed return.
- **Arithmetic:** signed overflow, signed shifts, and out-of-range conversions lack a supplied portable contract. Low-word characterizations are conditional.
- **Repeated arithmetic observations:** case 5 has two correction-constant references and three shift-mask references. Their relative evaluation order and equality are not established.
- **Other repeated references:** case 2 repeats field and transition references; case 8 repeats its first-field reference. Stable-observation descriptions require the stated equalities.
- **Sign selection:** the sign-bit interpretation follows the represented arithmetic right shift, not a supplied target-independent guarantee.
- **Address model:** exact pointer conversions, truncation, alignment, readable ranges, and writable ranges are absent.
- **Memory execution model:** reference occurrences do not establish machine-load counts, volatile behavior, atomicity, or concurrency guarantees.
- **Dependencies and lifetime:** preservation, ownership, allocation, release, and failure behavior are unknown.
- **Errors and intent:** no explicit validation protocol, recovery behavior, or application-level meaning is established.

## Evidence

Established evidence is limited to the supplied static function representation and opaque dependency mapping. It supports initialization, dispatch structure, field widths and offsets, case-local effects, dependency arguments, statement-level ordering, repeated expression references, and packed return layout.

The case-5 bit-completion characterization is a conditional mathematical interpretation that distinguishes repeated arithmetic-cell observations. Its single-value-per-role simplification additionally requires equal observations for each repeated role.

No runtime tests have been performed. No dependency implementations, caller contracts, static-data contents, or instruction-level execution evidence were supplied. All acceptance examples are conditional; unresolved behavior remains documented rather than assumed.
