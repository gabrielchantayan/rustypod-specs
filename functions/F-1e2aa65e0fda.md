<!-- Trusted provenance: F-1e2aa65e0fda; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-1e2aa65e0fda

## Purpose

The supplied static representation describes an ordered sequence of indirect calls, two identified dependency calls, coprocessor-transfer operations, and a final 16-bit memory write. Its terminal operation marks invalid instruction data rather than an ordinary return.

No application-level purpose is established. In particular, the evidence does not establish that this function initializes a device, processes a buffer, implements a protocol, or intentionally writes to low memory.

This contract records the represented execution path without assuming that the reconstruction is executable or complete. The input carries an explicit warning about invalid instruction data. All behavior below is conditional on execution reaching the relevant operation and on preceding dependencies returning.

## Interface

### Entry and exit

The apparent interface has no declared parameters and no declared return value. That is insufficient to establish a parameterless machine-level interface: the representation consumes incoming register state and coprocessor-related operands.

The observable entry-state dependencies are:

- An incoming link-register value, used to form the first indirect-call target by adding `0xDE6`.
- An incoming 32-bit fourth argument-register value, shifted left by 14 for a subsequent call.
- Incoming 32-bit values associated with saved registers 4 and 6. They supply, respectively, a logical right shift by 10 and a logical right shift by 3 followed by a left shift by 16.
- Coprocessor-related operands associated with register numbers 1, 3, 5, 6, 7, 9, 10, and 15.
- Two secondary register-result values whose producing operations are not established. They are consumed at separate calls later in the sequence.

Most later indirect-call targets are formed from fixed code-position-derived values plus constant offsets. Their validity, instruction-set state, calling convention, and function identities are unknown.

No normal return is represented. The final visible action after the memory write is an invalid-instruction termination marker. This does not establish a particular runtime exception, error code, or recovery mechanism.

### Widths and numerical notation

The representation uses 32-bit scalar values, 64-bit results split into two 32-bit words, and one 16-bit store.

For the numerical relationships below:

- A *low word* is bits 0–31 of a represented 64-bit result.
- A *high word* is bits 32–63.
- Left shifts describe retained 32-bit results: bits shifted beyond bit 31 are discarded.
- Logical right shifts insert zeros.
- Arithmetic right shifts propagate the sign bit.

These are bit-vector interpretations of the supplied representation, not a claim that its signed operations constitute well-defined portable language semantics. Target instruction decoding remains unverified.

Values described as words are opaque. Neither a 32-bit width nor subsequent forwarding establishes that a value is an integer rather than a pointer, handle, status, or packed field.

### Preconditions

There is no visible validation of entry state, call targets, returned values, coprocessor availability, or the final memory destination.

For the entire represented path to execute, every encountered call target must be executable, all preceding calls must return, each coprocessor operation must be permitted, and the final halfword store must be supported at its computed address. These are execution prerequisites, not checks performed by the function.

## Behavior

### Dependency notation and ordering

The opaque identifiers F-X001 through F-X092 below are document-local handles for unresolved call sites. Their numbers describe execution order, including positions occupied by the two identified dependencies. They do not establish distinct functions: different sites might resolve to the same target.

“No explicit arguments” means none appear in the representation. It does not prove that a callee receives no machine-register inputs. Likewise, discarding a represented result does not establish that the call has no other effects.

The listed order is the visible straight-line order. No represented branch, loop, retry, or error-result test changes it.

### 1. Entry-state consumption and initial coprocessor operations

1. F-X001 is called through the incoming link-register-derived target, with no explicit arguments.
2. F-X002 receives the incoming fourth argument-register word shifted left by 14.
3. F-X003 is called with no explicit arguments.
4. F-X004 receives four words, in order:
   - The incoming saved-register-6 word logically shifted right by 3, then left by 16.
   - Zero.
   - The incoming saved-register-4 word logically shifted right by 10.
   - Zero.
5. F-X005 returns a word. That word immediately supplies coprocessor-transfer event 1.
6. F-X006, F-X007, F-X008, and F-X009 are called in that order, without explicit arguments.
7. F-X010 returns a 64-bit value. Its low word supplies coprocessor-transfer event 2.
8. F-X011 receives that low word and that high word shifted left by 20. It returns another 64-bit value.
9. F-X012 receives F-X011’s low word, high word, and high word shifted left by 12.
10. F-X013, F-X014, and F-X015 are called in order without explicit arguments.
11. F-X016 returns a 32-bit word, called the *seed* below.
12. F-0e02e4c13579 receives the unshifted seed. Its represented result is unused.

The first argument to F-X004 retains only original bits 3–18 of the incoming saved-register-6 word. The third argument retains original bits 10–31 of the other incoming word.

### 2. Seed-derived forwarding and a first explicitly zeroed group

A seed-derived word is formed by shifting the seed left by 11. A separate word is explicitly set to zero.

1. F-X018 receives zero.
2. F-X019 through F-X022 are called in ascending order without explicit arguments.
3. F-X023 returns a 64-bit value.
4. F-X024 receives:
   - F-X023’s low word shifted left by 4.
   - Its unchanged high word.
   - The seed-derived word shifted left by 17.

   The third argument is therefore the original seed shifted left by 28, retaining only its lowest four bits.
5. F-X024’s word result passes through F-X025, whose word result is then passed to F-X026.
6. F-X027 is called without explicit arguments.
7. F-X028 receives two zeros. Its word result is passed to F-X029.
8. F-X030 receives four words: zero, an unresolved secondary register-result word, zero, and zero. Its first argument is zero because shifting the already left-shifted seed-derived word by another 31 discards every bit. Its other zero arguments derive from the explicitly zeroed word.
9. F-X030’s returned word supplies coprocessor-transfer event 3, then is passed to F-X031.
10. F-X032 returns a word, F-X033 transforms or forwards it into another returned word, and F-X034 receives that result.
11. F-X035, F-X036, and F-X037 are called without explicit arguments.
12. F-X038 returns a word, which F-X039 receives.
13. F-X040 receives zero.
14. F-X041 and F-X042 are called without explicit arguments.
15. F-X043 receives zero, a second unresolved secondary register-result word, and zero.

Neither secondary register-result word has a sufficiently established producer to specify its value or equate it to a preceding primary result.

### 3. Further forwarding and the second identified dependency

1. F-X044 and F-X045 are called without explicit arguments.
2. F-X046 returns a word, called the *sample*. A derived word is the sample shifted left by 5.
3. F-X047 receives the sample, zero, and that derived word.
4. F-X048 returns a word, which F-X049 receives.
5. F-3de949054bdd is called with no explicit arguments.
6. F-X051 returns a word, which F-X052 receives.
7. F-X053 returns a word, which F-X054 receives.
8. F-X055 receives zero and the arithmetic right shift by 5 of the sample-derived word. Its word result is passed to F-X056.
9. F-X057 returns a 64-bit value. Its low word supplies coprocessor-transfer event 4.
10. F-X058 receives F-X057’s high word shifted left by 1.
11. F-X059 returns a word. That same word supplies coprocessor-transfer events 5 and 6, in that order.

The second argument to F-X055 is not generally the original sample. The left shift first discards its top five bits; the subsequent arithmetic right shift sign-extends the retained bit 26.

### 4. Additional zero arguments and coprocessor effects

1. F-X060 returns a word. A *packed word* is formed by shifting it left by 27.
2. F-X061 receives zero, zero, the packed word, and zero.
3. F-X062 through F-X066 are called in ascending order without explicit arguments.
4. F-X067 returns a word, which supplies coprocessor-transfer event 7.
5. F-X068, F-X069, and F-X070 are called without explicit arguments.
6. Coprocessor-transfer event 8 receives zero. Its represented source is the packed word shifted left by 24, which necessarily discards all bits.
7. F-X071 receives zero and zero.
8. F-X072 is called without explicit arguments.
9. F-X073 returns a word, which supplies coprocessor-transfer event 9.
10. F-X074 is called without explicit arguments.
11. F-X075 receives three zeros and returns a word, called the *late word*.
12. The late word shifted left by 1 supplies coprocessor-transfer event 10.
13. F-X076 receives:
    - The late word shifted left by 1.
    - Zero.
    - The late word shifted left by 4.
    - Zero.

    The second argument is zero because masking the packed word with `0x7FFFFF` removes all potentially set bits.
14. F-X077 and F-X078 are called without explicit arguments.
15. Coprocessor-transfer events 11 and 12 each receive zero, in that order.

The packed word retains only the lowest five bits of F-X060’s result. Its use does not establish a field layout or intended packing scheme.

### 5. Final result processing and destination calculation

1. F-X079 receives zero and returns a word.
2. Coprocessor-transfer event 13 receives the late word shifted left by 12.
3. F-X080 receives the late word shifted left by 12, the late word shifted left by 16, and F-X079’s result shifted left by 9.
4. F-X081 returns a 64-bit value.
5. F-X082 receives its low word, high word, and low word shifted left by 1.
6. F-X083 receives zero. This zero derives from masking the late word’s left-shift-by-4 result with 3 before further shifts; that mask must produce zero.
7. F-X083’s result passes to F-X084, whose result passes to F-X085.
8. F-X086 is called without explicit arguments.
9. F-X087 returns a 64-bit value.
10. F-X088 receives F-X087’s low word, high word, and high word shifted left by 11. It returns another 64-bit value.
11. F-X089 receives F-X088’s low word.
12. F-X090 returns a word.
13. F-X091 receives F-X090’s result and zero.
14. F-X092 is called without explicit arguments.
15. The final 16-bit store is attempted.
16. The representation terminates at an invalid-instruction marker.

Let \(H\) be F-X088’s unsigned high word. The final destination is derived as follows:

\[
N = H \mathbin{\text{logical-right-shift}} 20
\]

\[
A = N + \left\lfloor N/4 \right\rfloor
\]

The represented intermediate first extracts 23 bits by shifting the 64-bit result right by 41. Its subsequent signed shift cannot sign-extend a negative value because that intermediate is at most `0x7FFFFF`. Consequently, the address formula above follows without a signedness ambiguity.

The store writes the 16-bit value `0x00CD` at address \(A\). There is no added buffer base.

## Side effects

### Calls and external state

Every dependency may have side effects not visible here. The evidence provides no basis for assuming that any dependency preserves globals, leaves memory unchanged, or allocates or releases particular objects.

No explicit ordinary-memory load or allocation appears in the represented body. The only explicit ordinary-memory write is the final halfword store. This does not exclude memory accesses within dependencies or architectural operations.

No ownership transfer, borrowing lifetime, object layout, or release obligation is established for any forwarded word.

### Coprocessor-transfer order

Thirteen transfer events are represented. Each specifies coprocessor number 0 and a further selector value of 7. The remaining operand triples, in execution order, are:

| Event | Selector | Coprocessor-related operand pair |
|---|---:|---|
| 1 | 5 | 15, 15 |
| 2 | 4 | 7, 15 |
| 3 | 6 | 3, 15 |
| 4 | 6 | 3, 15 |
| 5 | 3 | 6, 15 |
| 6 | 3 | 15, 15 |
| 7 | 1 | 1, 15 |
| 8 | 5 | 5, 15 |
| 9 | 6 | 9, 15 |
| 10 | 5 | 10, 15 |
| 11 | 7 | 9, 15 |
| 12 | 0 | 15, 15 |
| 13 | 2 | 9, 15 |

These are operand descriptions, not verified architectural register names or device semantics. No cache, interrupt, synchronization, or privilege meaning is established.

### Final memory write

The destination range is `0x0000` through `0x13FE`; the corresponding two-byte access may extend through `0x13FF`.

Some computed destinations are odd. There is no alignment adjustment. Whether an odd-address store succeeds, splits into accesses, or faults is unknown.

Byte order is unknown. The established operation is a halfword-valued write of `0x00CD`, not a verified ordering of two byte writes.

The final address is calculated from the captured F-X088 result, but the store occurs after F-X089 through F-X092. Those calls may change memory mappings, access permissions, or other relevant state. No preservation guarantee is assumed.

## Dependencies

- **F-0e02e4c13579:** receives the unshifted F-X016 result. No visible result is consumed. Its purpose, memory effects, ownership behavior, and failure modes are unknown.
- **F-3de949054bdd:** called between F-X049 and F-X051 without explicit arguments. Its effects and calling contract are unknown.
- **F-X001 through F-X092, excluding positions 017 and 050:** unresolved indirect-call sites described above. Their handles identify positions, not independently verified function identities.
- **Coprocessor-transfer operations:** architecture-dependent effects with thirteen represented occurrences.
- **Invalid-instruction termination:** a terminal representation artifact whose concrete runtime manifestation is unverified.

There is no visible error-status interpretation, exception handling, cleanup path, or compensating action.

## Acceptance examples

These examples are conditional static acceptance criteria, not executed tests.

1. **Seed truncation.** If the F-X016 seed is `0x00000001`, F-X024’s third explicit argument is `0x10000000`. If the seed is `0x00000010`, that argument is zero. F-0e02e4c13579 still receives the original seed in each case.

2. **Arithmetic shift boundary.** If F-X046 returns `0x04000000`, F-X055’s second argument is `0x04000000`. If it returns `0x08000000`, that argument is `0xFC000000`, under the represented arithmetic-right-shift interpretation.

3. **Forced zeros.** Regardless of F-X060’s result, coprocessor-transfer event 8 receives zero, both explicit F-X071 arguments are zero, and F-X076’s second argument is zero.

4. **Lowest destination.** If F-X088’s high word is below `0x00100000`, the final store destination is address zero. No visible null-address check intervenes.

5. **Alignment exposure.** If that high word is `0x00100000`, the destination is address 1. The contract does not predict success on a machine that rejects unaligned halfword stores.

6. **Address progression.** High words `0x00300000` and `0x00400000` produce destinations 3 and 5, respectively. The mapping is not a contiguous sequence of halfword-aligned slots.

7. **Upper boundary.** A high word of `0xFFFFFFFF` yields \(N=4095\), destination `0x13FE`, and a represented write of `0x00CD`.

8. **Interruption before completion.** If a dependency does not return, or an earlier architectural operation faults, no later call, transfer, or final store is guaranteed. No visible recovery path should be inferred.

## Uncertainties

### Established static evidence

The represented order, explicit argument relationships, two identified dependencies, thirteen coprocessor-transfer events, final store width and value, and terminal invalid-instruction marker are directly supported by the supplied material.

### Inferred intent

No application-level intent is sufficiently supported. The call density, code-position-derived targets, and invalid-instruction warning do not by themselves establish an initialization sequence, intentional obfuscation, or meaningful executable code.

### documented_only

The following remain unresolved:

- Whether the represented instruction stream is valid executable code.
- The real calling convention and completeness of the apparent signature.
- The identities and contracts of indirect callees.
- Which operations produce the two secondary register-result inputs.
- Whether the represented 64-bit results are semantic 64-bit values or paired machine-register outputs.
- Actual coprocessor operand interpretation and privilege requirements.
- Endianness, address-space mapping, alignment behavior, and store atomicity.
- Callee register and global preservation.
- Ownership, allocation, release, concurrency, and reentrancy behavior.
- Concrete terminal fault behavior and whether the final store is reachable.

## Evidence

This contract uses only the supplied static representation and dependency map. The representation explicitly warns of invalid instruction data and truncates control flow at its endpoint.

No implementation access, instruction verification, execution, runtime tests, or observed state transitions were available. Numerical examples are deductions under the stated 32-bit bit-vector interpretation, not runtime observations.
