<!-- Trusted provenance: F-8f8d7acaa630; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8f8d7acaa630

## Purpose

The supplied static representation describes a sequential routine containing 53 dependency invocations, ten coprocessor-transfer operations, one explicit 16-bit memory store, and an invalid-instruction termination boundary.

No application-level purpose is established. The evidence does not identify this behavior as initialization, decoding, device configuration, allocation, or successful processing of the declared inputs.

The representation includes an instruction-decoding warning. This contract therefore distinguishes:

- **Established static evidence:** the represented operations, data dependencies, and ordering.
- **Conditional arithmetic deductions:** consequences of interpreting the operations as fixed-width machine arithmetic.
- **Unknown behavior:** execution validity, dependency semantics, environmental requirements, and effects not established by the supplied material.

The contract describes the represented behavior. It does not establish that the entire sequence executes successfully on any particular machine.

## Interface

### Declared inputs and output

| Position | Represented width | Explicit use |
|---|---:|---|
| First input | 32 bits | No explicit use in the represented body. |
| Second input | 32 bits | No explicit use in the represented body. |
| Third input | Signed 32 bits | Shifted left by 22 and passed to `F-U08`. |

No ordinary return value is declared. No normal return path is represented. If execution reaches the end, the explicit store precedes an invalid-instruction termination operation.

The lack of explicit references to the first two inputs does not establish that dependencies cannot observe entry registers, stack state, or other implicit inputs.

### Additional entry state

The representation also consumes:

- Two incoming general-purpose register values, designated here as **entry word A** and **entry word B**.
- Incoming link-register state used to derive the first call target.
- An additional register result exposed after the early calls.
- Seven incoming coprocessor-related selector values, distinguished below by selector numbers.
- An execution environment capable of resolving computed calls, performing the represented transfers, and interpreting the final store.

Entry word A supplies the left-8 expression. Entry word B supplies the left-26 and left-13 expressions.

The actual entry ABI, required register initialization, and origins of the additional register result are undocumented.

### Widths and layout conventions

For the conditional deductions in this contract:

- Ordinary words are 32 bits.
- Wide results are 64 bits.
- A wide result’s low word is bits 0–31; its high word is bits 32–63.
- A 32-bit left shift discards bits shifted beyond bit 31.
- Right shifts of signed quantities are interpreted as arithmetic where identified.
- Explicit unsigned right shifts are logical.

These are machine-word interpretations. Represented signed left shifts do not independently establish well-defined portable high-level arithmetic.

The final explicit memory write is 16 bits wide and stores the numeric value 205, hexadecimal `00CD`. Byte order is unspecified. No input buffer, object layout, allocation size, or ownership-bearing pointer interface is established.

## Behavior

### Ordering and dependency notation

`F-U01` through `F-U52` are editorial opaque identifiers for unresolved call sites. They identify positions in this contract, not proven distinct functions. Two sites might resolve to the same function, and a computed target is not established as valid executable code.

`F-3de949054bdd` is the one dependency with a supplied identity.

All operations follow the represented sequence. Reaching any operation requires the preceding calls to return and the preceding operations to complete. No local retry, exception handler, status branch, or recovery path is shown.

“None shown” in the tables means that the representation supplies no explicit arguments. It does not exclude implicit register, stack, memory, or global inputs. Results described as unused are not explicitly consumed by this routine; their production may still have effects.

### Initial sequence

| Position | Dependency | Explicit arguments and result use |
|---:|---|---|
| 1 | `F-U01` | None shown. Target derived from incoming link-register state plus displacement `0294` hexadecimal. |
| 2 | `F-U02` | None shown. |
| 3 | `F-U03` | None shown. |
| 4 | `F-U04` | Entry word A shifted left 8; the additional exposed register result; entry word B shifted left 26. |
| 5 | `F-U05` | None shown. |
| 6 | `F-U06` | None shown. |
| 7 | `F-U07` | None shown. Produces the **first scalar result**, a 32-bit word. |
| 8 | `F-U08` | First scalar result; third declared input shifted left 22; first scalar result shifted left 5. |
| 9 | `F-U09` | None shown. Produces a word passed to `F-U10`. |
| 10 | `F-U10` | Result of `F-U09`. |
| 11 | `F-3de949054bdd` | None shown. No result explicitly consumed. |
| 12 | `F-U11` | None shown. Produces a word passed to `F-U12`. |
| 13 | `F-U12` | Result of `F-U11`. |
| 14 | `F-U13` | None shown. Produces a word passed to `F-U14`. |
| 15 | `F-U14` | Result of `F-U13`. |
| 16 | `F-U15` | Entry word B shifted left 13; the retained left-5 result arithmetically shifted right 5. |
| 17 | `F-U16` | Word result of `F-U15`. |
| 18 | `F-U17` | None shown. Produces the **first wide result**. |

The left-5 then arithmetic-right-5 expression is not generally an identity. Under the stated interpretation, it preserves the low 27 original bits and sign-extends original bit 26.

After `F-U17`, transfer 1 consumes its low word. `F-U18` then receives its high word shifted left 1.

`F-U19` follows with no explicit arguments. Transfers 2 and 3 consume its same word result. No dependency invocation intervenes between those two transfers.

### Upper-bit value and middle sequence

`F-U20` executes with no explicit arguments. Its word result is shifted left 27 and retained as the **upper-bit value**. Under 32-bit truncation, only the original low five bits survive, occupying bits 27–31.

A separate retained word is explicitly set to zero before `F-U21`. It remains zero in the representation until used to form the arguments of `F-U35`.

| Position in this phase | Dependency or operation | Explicit behavior |
|---:|---|---|
| 1 | `F-U21` | Receives zero, zero, the upper-bit value, zero. |
| 2 | `F-U22` | No explicit arguments. |
| 3 | `F-U23` | No explicit arguments. |
| 4 | `F-U24` | No explicit arguments. |
| 5 | `F-U25` | No explicit arguments. |
| 6 | `F-U26` | No explicit arguments. |
| 7 | `F-U27` | No explicit arguments. Its word result supplies transfer 4. |
| 8 | Transfer 4 | Occurs before `F-U28`. |
| 9 | `F-U28` | No explicit arguments. |
| 10 | `F-U29` | No explicit arguments. |
| 11 | `F-U30` | No explicit arguments. |
| 12 | Transfer 5 | Payload is the upper-bit value shifted left 24. |
| 13 | `F-U31` | Receives that same left-24 expression and zero. |
| 14 | `F-U32` | No explicit arguments. |
| 15 | `F-U33` | No explicit arguments. Its word result supplies transfer 6. |
| 16 | Transfer 6 | Occurs before `F-U34`. |
| 17 | `F-U34` | No explicit arguments. |
| 18 | `F-U35` | Receives the retained zero word shifted left 22, that word shifted left 11, and zero. Produces the **second scalar result**. |

Under the fixed-width model, transfer 5 has a zero payload and both arguments to `F-U31` are zero. The left-27 followed by left-24 operations discard every original bit. All three arguments to `F-U35` are also zero.

Transfer 7 consumes the second scalar result shifted left 1. The second scalar result shifted left 4 is retained as the **nibble-shifted result**.

`F-U36` receives four words:

1. Second scalar result shifted left 1.
2. The low 23 bits of the upper-bit value, shifted left 4.
3. Nibble-shifted result.
4. Zero.

Its second argument is zero under the fixed-width model because the upper-bit value has no set bits below bit 27.

`F-U37` and then `F-U38` execute with no explicit arguments.

### Later sequence

Transfer 8 carries zero. Between transfers 8 and 9, the representation derives a retained quantity by masking the nibble-shifted result with 3 and shifting the masked value left 16. Transfer 9 also carries zero.

Call the retained quantity the **zero-derived value**. Under the stated arithmetic model it is zero: a left-4 result has four zero low bits, so masking it with 3 yields zero.

| Position in this phase | Dependency or operation | Explicit behavior |
|---:|---|---|
| 1 | `F-U39` | Receives zero. Its word result is retained for `F-U40`. |
| 2 | Transfer 10 | Payload is the nibble-shifted result shifted left 8. |
| 3 | `F-U40` | Receives the nibble-shifted result shifted left 8; the nibble-shifted result shifted left 12; the result of `F-U39` shifted left 9. |
| 4 | `F-U41` | No explicit arguments. Produces the **second wide result**. |
| 5 | `F-U42` | Receives the second wide result’s low word, its high word, and its low word shifted left 1. |
| 6 | `F-U43` | Receives the zero-derived value shifted left 8. Produces a word. |
| 7 | `F-U44` | Receives the result of `F-U43`. Produces a word. |
| 8 | `F-U45` | Receives the result of `F-U44`. |
| 9 | `F-U46` | No explicit arguments. |
| 10 | `F-U47` | No explicit arguments. Produces the **third wide result**. |
| 11 | `F-U48` | Receives the third wide result’s low word, its high word, and its high word shifted left 11. Produces the **final wide result**. |
| 12 | `F-U49` | Receives the final wide result’s low word. |
| 13 | `F-U50` | No explicit arguments. Produces a word. |
| 14 | `F-U51` | Receives the result of `F-U50` and the zero-derived value shifted left 25. |
| 15 | `F-U52` | No explicit arguments. |

The second argument to `F-U40` is a left-12 shift of the nibble-shifted result itself, not a further left-12 shift of its first argument.

Immediately after `F-U48`, and before `F-U49`, the representation logically shifts the final wide result right by 41 and retains the resulting unsigned quantity in a 32-bit word. That quantity has at most 23 significant bits. The left-25 expression used later by `F-U51` is also formed before `F-U49`.

The argument to `F-U43` and the second argument to `F-U51` are zero under the fixed-width model.

### Final address, store, and termination

After `F-U52` returns, the retained 23-bit quantity is used to calculate the store address. Its represented signed right shift by 11 cannot encounter a negative value under the stated model.

Let the **address index** be bits 52–63 of the final wide result returned by `F-U48`. The destination address is:

**address index + floor(address index / 4).**

This follows from logically shifting the wide result right by 41, shifting the retained quantity right by 11, and adding its unsigned right-2 result.

Consequences:

- The address index ranges from 0 through 4095.
- The destination ranges from 0 through 5118, hexadecimal `13FE`.
- No memory base is added.
- Not every address in that range is generated.
- Odd destinations are possible.
- The third wide result returned by `F-U47` is not the direct basis of this address; the result returned by `F-U48` is.

The routine then performs its explicit 16-bit store of `00CD`. A complete store affects the destination byte and the following byte, subject to memory and alignment semantics.

The invalid-instruction termination operation follows the store. There is no represented normal return, success status, retry, or cleanup path.

## Side effects

### Explicit coprocessor operations

Each represented operation has **six operands total: one payload and five non-payload operands**. Each selector tuple below contains those five non-payload operands in their represented order, omitting the payload.

“Incoming selector” identifies entry-state data. The selector numbers distinguish the supplied inputs; they do not independently establish hardware register meanings.

| Transfer | Payload | Five non-payload operands |
|---:|---|---|
| 1 | First wide result’s low word | `(0, 6, 7, incoming selector 3, incoming selector 15)` |
| 2 | Result of `F-U19` | `(0, 3, 7, incoming selector 6, incoming selector 15)` |
| 3 | Same result of `F-U19` | `(0, 3, 7, incoming selector 15, incoming selector 15)` |
| 4 | Result of `F-U27` | `(0, 1, 7, incoming selector 1, incoming selector 15)` |
| 5 | Upper-bit value shifted left 24; conditionally zero | `(0, 5, 7, incoming selector 5, incoming selector 15)` |
| 6 | Result of `F-U33` | `(0, 6, 7, incoming selector 9, incoming selector 15)` |
| 7 | Second scalar result shifted left 1 | `(0, 5, 7, incoming selector 10, incoming selector 15)` |
| 8 | Zero | `(0, 7, 7, incoming selector 9, incoming selector 15)` |
| 9 | Zero | `(0, 0, 7, incoming selector 15, incoming selector 15)` |
| 10 | Nibble-shifted result shifted left 8 | `(0, 2, 7, incoming selector 9, incoming selector 15)` |

The hardware effects, privilege requirements, fault behavior, and synchronization properties of these transfer-like operations are unresolved. Their presence does not establish a specific device protocol or memory barrier.

### Memory and state

The final halfword store is the only explicit ordinary-memory write shown. No explicit ordinary-memory load through a data pointer is represented.

This observation does not bound dependency effects. Calls may read or change memory, globals, registers, devices, or resources; none of those effects is established or excluded here.

The routine retains intermediate values across represented calls. That dataflow is not a general guarantee that unknown callees preserve global state or arbitrary registers.

The final destination may denote ordinary memory, mapped device state, or inaccessible space. The representation supplies no address validation or alignment correction.

### Ownership and failure

No declared input is established as an allocation, borrowed pointer, buffer, object, or resource handle. No ownership transfer, release obligation, or allocation behavior can be specified.

Returned values are used as data without explicit status checks. There is no established error-code convention.

If a call does not return, or an earlier operation faults without returning control to this sequence, later represented effects are not reached. If the final store fails, this contract does not guarantee either completion of that write or arrival at the terminal boundary. Partial-write behavior is environment-dependent.

## Dependencies

- **`F-U01`–`F-U52`:** Unresolved call sites. Most targets are constructed from fixed location-like values and displacements; the first uses incoming link-register state. Identity, ABI, validity, effects, ownership, and failure behavior remain undocumented.
- **`F-3de949054bdd`:** Invoked once, after `F-U10` and before `F-U11`. No explicit arguments or consumed result are shown. Its supplied identity establishes no further semantics.
- **Coprocessor-transfer facility:** Required for the ten represented transfers. Operand structure is visible; hardware meaning is not.
- **Memory subsystem:** Determines whether the calculated low-address halfword write is permitted and completes.
- **Invalid-instruction handling:** Determines the external outcome if execution reaches the terminal operation.

Successful traversal of the entire sequence requires valid call targets, returning dependencies, completing transfers, and a completing final write. These are execution conditions, not established guarantees.

## Acceptance examples

These examples are conditional arithmetic and ordering checks, not executed tests.

1. **Third-input truncation.**  
   Under 32-bit left shifts, a third input of 1 contributes `00400000` hexadecimal to `F-U08`. An input of 1024 contributes zero. Only the input’s low ten bits survive.

2. **Signed shift pair.**  
   If `F-U07` returns `04000000` hexadecimal, its left-5 result is `80000000`. Arithmetic right shift by 5 produces `FC000000` for the second argument to `F-U15`, not the original word.

3. **Upper-bit elimination.**  
   If `F-U20` returns 31, the upper-bit value is `F8000000`. Its left-24 expression is zero, as is its low-23-bit masked expression. Transfer 5, `F-U31`, and the second argument to `F-U36` must reflect those deductions under the stated model.

4. **Zero-derived arguments.**  
   Whatever `F-U35` returns, its left-4 result has zero low four bits. Therefore the argument to `F-U43` and the second argument to `F-U51` are zero under fixed-width arithmetic.

5. **Independent shifts at `F-U40`.**  
   If the nibble-shifted result is `00000010` hexadecimal, the first two arguments to `F-U40` are `00001000` and `00010000`, respectively. The second is not a left-12 shift of the first.

6. **Operand accounting.**  
   Transfer 2 carries the result of `F-U19` as one payload plus the five operands in its listed tuple. Transfer 3 reuses that payload but has a different tuple. Neither tuple has six non-payload operands.

7. **Lowest destination.**  
   If bits 52–63 of the final wide result are zero, the destination is zero even when lower bits are nonzero. This remains the represented destination regardless of the third wide result’s corresponding bits.

8. **Address gaps and alignment.**  
   Address indices 3 and 4 produce destinations 3 and 5. No rounding or buffer-base substitution is represented. Whether either halfword write completes depends on the environment.

9. **Largest destination.**  
   An address index of 4095 produces destination 5118. A complete store covers addresses 5118 and 5119. Their byte values depend on byte order.

10. **Ordering on non-return.**  
    If `F-U49` does not return, the retained address-related quantity has already been derived, but `F-U50`, `F-U51`, `F-U52`, and the final store are not reached.

## Uncertainties

The following remain **documented_only**:

- Accuracy of the instruction decoding and whether the sequence represents executable code.
- Correspondence between the declared inputs and the actual entry ABI.
- Actual machine semantics of the represented signed shifts.
- Origins and validity of additional incoming register and selector state.
- Identity, implicit inputs, outputs, and side effects of unresolved dependencies.
- Whether wide results use the represented 64-bit return convention.
- Coprocessor meaning, privilege, fault, and ordering requirements.
- Memory mapping, byte order, alignment handling, and store atomicity.
- Whether the calculated low addresses are writable.
- Runtime behavior at the invalid-instruction boundary.
- Allocation, ownership, cleanup, global-state preservation, and error reporting.

No application intent is established. Any description such as initialization or device configuration would be inference rather than a supported behavioral requirement.

## Evidence

The supplied static representation supports the declared input widths, ordered dependency invocations, explicit argument expressions, retained result uses, ten coprocessor-transfer operations, final 16-bit store, and invalid-instruction terminal marker.

Each transfer visibly contains one payload and five non-payload operands. The five-element selector tuples above preserve that structure.

The supplied dependency mapping establishes this routine’s identity and the identity `F-3de949054bdd`; it does not resolve the other call sites.

Zero simplifications, shift consequences, address bounds, and numerical acceptance examples are deductions conditional on the fixed-width interpretation stated in this contract.

No runtime tests have been performed.
