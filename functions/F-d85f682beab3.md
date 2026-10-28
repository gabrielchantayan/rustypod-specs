<!-- Trusted provenance: F-d85f682beab3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d85f682beab3

## Purpose

F-d85f682beab3 conditionally forwards a packed 32-bit value to a dependency.

The established operation is:

- Read a four-byte value through the first input and pass it, together with the address of initialized local storage, to F-cdf60417d657.
- Continue only if F-cdf60417d657 returns zero.
- Capture two shifted contributions from local quantities.
- Call F-93fd250aa9bf.
- Combine the previously captured contributions with bitwise OR while forming the third argument to F-b63ebee7ff72, and invoke that dependency.

The distinction between capturing the shifted contributions and combining them is part of the observable call ordering: both shifts precede F-93fd250aa9bf; their OR follows its normal return.

The application-level meanings of the packed fields, the constant argument 3, and the value obtained from F-93fd250aa9bf are not established. Descriptions such as notification, registration, or scheduling would be inferred intent rather than demonstrated behavior.

This contract is based on static evidence only. No runtime tests have been performed.

## Interface

### Inputs

The supplied representation has four positional inputs:

| Position | Represented width and kind | Established use |
|---|---|---|
| First | Signed 32-bit integer used as an address base | Locates a four-byte value at byte offset `0x40000`. |
| Second | Opaque four-byte value | Not explicitly referenced by the body. |
| Third | Unsigned 32-bit integer | Initializes an address-exposed local quantity whose post-call value supplies the lower packed field. |
| Fourth | Signed 32-bit integer | Initializes a separate saved local quantity whose post-call value supplies the upper packed field. |

These widths and kinds describe the supplied representation. They do not independently establish the complete binary calling convention, pointer representation, or original declarations.

The second input has no explicit effect on the body’s branches, arithmetic, memory accesses, or dependency arguments.

### Output

The function has no explicit return value.

Normal return does not indicate whether the zero-return continuation path was taken. The function also does not inspect or forward a result from F-b63ebee7ff72.

### Storage preconditions

The function reads four bytes from the address formed by adding `0x40000`, or 262,144 bytes, to the first input’s address base.

Ordinary successful execution requires that address formation and the resulting access be valid in the execution environment. There is no explicit check for a null base, overflow, alignment, inaccessible memory, or insufficient object extent.

The offset alone does not establish a particular object type or allocation size. The four-byte value read at that offset is passed by value, not by passing the address of the field.

## Behavior

### Initialization and initial read

Before the first dependency invocation, the function initializes:

- An unsigned 32-bit local quantity from the third input.
- A signed 32-bit saved local quantity from the fourth input.

It then reads the four-byte field at the first input’s base plus `0x40000`.

That field is read before F-cdf60417d657 is invoked. The body does not read it again. A subsequent change to that field therefore does not replace the value already supplied as the first dependency argument.

### First dependency and continuation condition

F-cdf60417d657 receives two explicit arguments:

1. The four-byte value obtained from the initial read.
2. The address of the unsigned local quantity initialized from the third input.

The address makes that local quantity accessible to the dependency. On the continuation path, the function uses its value after the dependency returns, rather than unconditionally using the original third input.

The dependency’s result is represented as a signed 32-bit integer and is compared only with zero:

- **Zero:** continue to the shifts and remaining dependency calls.
- **Any nonzero value:** skip those operations and return normally.

Positive and negative nonzero results have the same branch effect. There is no retry, alternate submission, rollback, or explicit error report.

Zero is an established continuation condition. The evidence does not establish that it means success, nor that every nonzero value denotes an error.

The saved quantity initialized from the fourth input is also consumed after this call. Although its address is not explicitly passed, the dependency’s implementation and access extent are unavailable. Its preservation is not an independently established dependency guarantee.

### Capturing the shifted contributions

On the zero-return path, the function evaluates and saves two 32-bit contributions, in this order:

1. The post-call unsigned local quantity shifted right by 10 bits.
2. The saved signed local quantity shifted left by 22 bits, with the result assigned to an unsigned 32-bit contribution.

Both shifted contributions are captured before F-93fd250aa9bf is invoked. The body does not yet combine them with OR.

The unsigned right shift discards bits 0 through 9. Bits 10 through 31 become contribution bits 0 through 21. This contribution cannot set bits 22 through 31.

There is no rounding adjustment. Consequently, unsigned local values that differ only in their lowest ten bits produce the same right-shifted contribution.

Under a 32-bit bit-vector interpretation, the left shift places the saved quantity’s low ten bits into contribution bits 22 through 31. Bits shifted beyond the 32-bit width do not appear in that interpretation’s result.

However, the saved quantity is represented as signed. Its left shift must not be treated as an established full-domain unsigned operation merely because the contribution is subsequently stored as unsigned. Negative operands and signed-range overflow remain subject to the execution-model uncertainty described below.

### Retrieval, combination, and final invocation

After both shifted contributions have been captured, F-93fd250aa9bf is called with no explicit arguments.

If it returns normally, the function forms the third argument to F-b63ebee7ff72 by applying bitwise OR to the two captured contributions. This combination occurs after F-93fd250aa9bf returns, not before its invocation.

F-b63ebee7ff72 then receives three explicit arguments, in this order:

1. The four-byte value returned by F-93fd250aa9bf.
2. The integer constant 3.
3. The 32-bit OR of the previously captured shifted contributions.

The returned four-byte value is forwarded without a visible validity check. Zero does not suppress the final call.

There is no intervening reread of the quantities used for the shifts, and neither shift is repeated after F-93fd250aa9bf. Thus, later changes to those source quantities do not cause this body to recompute the contributions. This observation does not establish that the dependency preserves global state or has no other effects.

Subject to the signed-shift qualification, the resulting fields are:

| Result bits | Source |
|---|---|
| 0–21 | Bits 10–31 of the unsigned local quantity as read after F-cdf60417d657. |
| 22–31 | Low ten bits of the saved signed quantity as read after F-cdf60417d657, under the 32-bit bit-vector interpretation. |

The combination is bitwise OR, not addition. Under the stated interpretation, the two contributions occupy disjoint fields.

After F-b63ebee7ff72 returns normally, F-d85f682beab3 returns without examining a dependency result.

## Side effects

### Direct memory activity and ordering

The body establishes the following sequence on the zero-return path:

1. Initialize the two local quantities from the third and fourth inputs.
2. Read the four-byte field through the first input.
3. Invoke F-cdf60417d657 with that value and the address of the unsigned local quantity.
4. Read and right-shift the post-call unsigned quantity.
5. Read and left-shift the saved signed quantity.
6. Invoke F-93fd250aa9bf.
7. OR the captured contributions while forming the final call’s third argument.
8. Invoke F-b63ebee7ff72.

On a nonzero result from F-cdf60417d657, steps 4 through 8 are not reached.

No direct write through the first input and no explicit write to global storage appear in the body. This does not establish preservation of caller-visible state: the dependencies may perform additional reads, writes, or other effects.

### Ownership and lifetime

No allocation, deallocation, retain, release, or explicit ownership-transfer operation is shown.

The four-byte values read and exchanged with dependencies are opaque. They are not established to be pointers, handles, owned resources, or borrowed references. Ownership behavior cannot be inferred from their width or their passage between functions.

The address supplied to F-cdf60417d657 refers to function-local storage. That storage is available during the synchronous call. The evidence does not establish:

- Whether the dependency retains the address.
- Whether retention is permitted.
- Whether the dependency accesses exactly four bytes or a larger region.
- Whether it affects other local storage.

The function provides no mechanism for extending its local-storage lifetime beyond its return.

### Failure and interruption

There is no explicit fault recovery, exception handling, timeout, or rollback.

The described normal sequence assumes valid memory access and normal completion of each reached dependency. If a dependency does not return normally, subsequent steps are not established to occur.

In particular, if F-93fd250aa9bf does not return normally, both shifts have already been evaluated, but the later OR and invocation of F-b63ebee7ff72 are not reached on the represented path.

## Dependencies

| Opaque ID | Established interaction | Unestablished behavior |
|---|---|---|
| F-cdf60417d657 | Receives a four-byte value and an address to initialized local storage. Its represented signed 32-bit result controls continuation by comparison with zero. | Argument meanings, mutation rules, access extent, retained addresses, ownership, global effects, and failure mechanisms. |
| F-93fd250aa9bf | Called without explicit arguments after both shifts and before their OR. Supplies the four-byte first argument of the final call. | Meaning and ownership of its result, global effects, retained state, and failure mechanisms. |
| F-b63ebee7ff72 | Receives the retrieved value, constant 3, and the OR of the captured contributions. Its result is not inspected. | Operation meaning, interpretation of 3 and the packed fields, ownership, state changes, and failure mechanisms. |

For normal execution, the dependency order is fixed: F-cdf60417d657 first, then—only on a zero result—F-93fd250aa9bf followed by F-b63ebee7ff72.

No dependency is assumed to preserve globals or to acquire, retain, or release memory.

## Acceptance examples

These are conditional examples derived from static evidence, not executed tests. Unless otherwise stated, they assume valid memory access, normal dependency returns, and preservation of the saved quantity initialized from the fourth input.

### Nonzero continuation result

If F-cdf60417d657 returns 1, −1, or another nonzero represented value:

- Neither shift is evaluated.
- F-93fd250aa9bf is not called.
- F-b63ebee7ff72 is not called.
- The function returns without an explicit result.

This remains the branch outcome if F-cdf60417d657 modifies the addressed unsigned quantity before returning.

### Zero contributions

If F-cdf60417d657 returns zero and both local quantities subsequently read for the shifts are zero:

- Both captured contributions are zero.
- F-93fd250aa9bf is called.
- After its normal return, the OR produces `0x00000000`.
- F-b63ebee7ff72 receives the retrieved value, 3, and `0x00000000`.

The zero result is not formed by OR before the retrieval call.

### Unsigned truncation boundary

With the saved signed quantity equal to zero and a zero continuation result:

| Post-call unsigned quantity | Right-shifted contribution | Final packed value |
|---|---|---|
| 1,023 | `0x00000000` | `0x00000000` |
| 1,024 | `0x00000001` | `0x00000001` |

The final packed values are formed after the retrieval call returns normally. This boundary demonstrates truncation, not rounding.

### Combined fields

If the post-call unsigned quantity is 1,024 and the saved signed quantity is 1, the captured contributions are `0x00000001` and `0x00400000`.

After F-93fd250aa9bf returns normally, their OR produces `0x00400001`, which is supplied as the third argument to F-b63ebee7ff72.

### First-dependency mutation

If the third input is zero but F-cdf60417d657 writes 2,048 to the addressed unsigned quantity and returns zero, the right-shifted contribution is 2.

With the saved signed quantity equal to zero, the eventual packed value is `0x00000002`. Packing the original third input instead would not match the post-call read.

### No recomputation after retrieval

Suppose the contributions have been captured as `0x00000001` and `0x00400000`. A subsequent change during F-93fd250aa9bf to the original quantities used for those shifts does not trigger a reread or repeated shift in this body.

Conditional on the dependency returning normally without altering the captured contribution storage, the later OR remains `0x00400001`. This example does not assume general memory or global-state preservation.

### Unchecked retrieved value

If the continuation result is zero and F-93fd250aa9bf returns zero normally, F-b63ebee7ff72 is still invoked with zero as its first argument. The body contains no guard rejecting that returned value.

## Uncertainties

The following matters are **documented_only**:

- **Local layout:** The two initialized quantities are distinct in the representation, but physical adjacency and a wider aggregate layout are not independently established. The address argument does not prove an eight-byte input/output contract.
- **Dependency access extent:** F-cdf60417d657’s permitted or actual access beyond the addressed quantity is unknown. Preservation of the saved signed quantity is not guaranteed by the supplied evidence.
- **Signed left shift:** Negative operands and operands whose shifted mathematical value exceeds the signed range require additional execution-model evidence. The bit-vector field explanation is conditional, not a definitive full-domain guarantee.
- **Address formation:** Overflow behavior, pointer conversion, alignment requirements, and invalid-access outcomes are not established.
- **Representation:** Byte order and the internal interpretation of opaque four-byte values are unknown.
- **External effects and ownership:** Dependency implementations and resource contracts are unavailable. No global-preservation or memory-ownership guarantee is inferred.
- **Application intent:** The meanings of both fields, the retrieved value, and constant 3 remain unknown.

## Evidence

The supplied static body establishes the four input uses, local initialization, four-byte read at offset `0x40000`, first dependency arguments, zero-versus-nonzero branch, shift widths, and final dependency arguments.

It specifically establishes that the unsigned right-shift contribution and signed left-shift contribution are captured before F-93fd250aa9bf. Their bitwise OR appears afterward in formation of the third argument to F-b63ebee7ff72. There is no intervening reread or repeated shift.

The packed-field explanation follows from the represented widths and shift counts, subject to the signed-shift qualification. Application intent, dependency contracts, and a complete execution model are not supplied.

No runtime observations or tests support this contract; no runtime tests have been performed.
