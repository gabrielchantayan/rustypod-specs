<!-- Trusted provenance: F-8cdd2aa6bed3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-8cdd2aa6bed3

## Purpose

The established behavior is an ordered prefix of direct memory accesses and indirect dependency calls. An explicit integer input, two ambient register-derived values, dependency results, and two separately read selector bytes determine memory destinations, stored values, and later call arguments.

No application-level purpose is established. The available evidence does not justify interpreting this behavior as allocation, initialization, serialization, device control, or resource management.

The supplied representation reports invalid instruction data and ends at a control-flow truncation boundary. This contract therefore describes only the represented prefix. It does not establish that the instruction stream is executable, that dependencies return, or that the function completes normally.

## Interface

### Inputs

The explicit input is represented as a signed 32-bit integer. No range check or validation is visible. It contributes to:

- The initial word-write address.
- Arguments to **F-D01** and **F-D06**.
- The address used for both selector reads.

Two ambient inputs are also relevant:

- An unspecified incoming 32-bit register value supplies the initial word-write contents.
- An incoming link-register value contributes to the target of **F-D01**.

These are not established additional formal parameters. Their origins, entry-state requirements, and relationship to any calling convention are unknown.

Memory contents and dependency behavior supply further inputs. In particular, the selector location is read twice, with calls and a byte write between those reads. The two reads need not return equal values.

### Outputs and completion

No explicit return value or normal return is represented. Observable effects of the visible prefix consist of:

- Three direct 32-bit writes.
- Two direct unsigned 8-bit reads.
- Two direct 8-bit writes.
- Twenty-nine indirect dependency invocations.
- Any effects performed by those dependencies.

No success indicator, error result, checked failure convention, or recovery protocol is visible.

### Widths and arithmetic

The representation uses 32-bit integer quantities and word writes, unsigned 8-bit selector reads, and 8-bit writes. Each selector is therefore a number from 0 through 255, not a full-width pointer.

The following arithmetic distinctions are material:

- The result retained from **F-D15** is unsigned 32-bit. Its shifts, mask, and right shift have unsigned 32-bit semantics in the representation.
- The initial input multiplication occurs before its result is retained as unsigned. An unsigned destination does not establish that this preceding signed multiplication wraps.
- Other input-dependent and result-dependent signed multiplications, additions, and left shifts may encounter signed-overflow or signed-shift restrictions.
- Conversions between unsigned and signed 32-bit representations do not independently establish a portable interpretation of every high-bit-set value.

Where this contract uses a **conditional 32-bit wrapping model**, arithmetic is interpreted as 32-bit bit-pattern arithmetic reduced modulo \(2^{32}\). That model supports the address relationships and examples explicitly labeled with it; it is not an unconditional claim about executable behavior.

Pointer width, integer-to-address conversion, alignment requirements, byte order, and access-fault semantics remain unknown.

## Behavior

### Retained values and observation order

**F-D01** through **F-D29** identify dependency-call occurrences in order. They do not establish distinct implementations: multiple IDs may resolve to the same implementation.

For describing value relationships:

- **Input scale** means four times the explicit input, retained as a 32-bit quantity.
- **Retained word** means four times the result of **F-D05**.
- **Byte source** means the unsigned 32-bit result of **F-D15**.
- **First selector** and **second selector** mean the values returned by the two distinct byte reads.

These terms describe data dependencies, not additional memory accesses or an inferred storage layout.

All ordering obligations are conditional on execution reaching the relevant operation. Invalid execution, a fault, or a dependency that does not return may prevent the remaining prefix.

### Direct memory accesses

The direct accesses occur in this relative order, with calls interleaved as described below.

| Access | Numerical address relationship | Width and value |
|---|---|---|
| Initial write | Input scale plus 20 | 32-bit incoming ambient value |
| First result-addressed write | Sixteen times the **F-D05** result, plus 20 | 32-bit retained word |
| Second result-addressed write | Four times the **F-D10** result, plus 20 | The same retained word |
| First selector read | Explicit input multiplied by \(2^{28}\), plus 1 | Unsigned 8-bit read |
| First selector-directed write | First selector plus 17 | Low eight bits of the byte source shifted left two positions |
| Second selector read | The same input-derived address formula | A new unsigned 8-bit read |
| Second selector-directed write | Second selector plus 17 | Bits 20–27 of the byte source |

The first result-addressed destination uses a sixteenfold scale; the second uses a fourfold scale. Both writes use the value retained from **F-D05**. Neither rereads that value from the earlier destination.

Each selector-directed destination is numerically between 17 and 272 inclusive. The selector is used as a small integer address component; the read does not retrieve a pointer-sized destination.

The displacements 20 and 17 do not establish named fields, enclosing objects, allocated regions, or valid mappings.

### Call boundaries

The initial word write occurs before **F-D01**.

Calls **F-D01–F-D05** occur in order before the first result-addressed write. After that write, **F-D06–F-D10** occur in order. The second result-addressed write follows **F-D10** and precedes **F-D11**.

Calls **F-D11–F-D15** then occur in order. The first selector read and first selector-directed write occur after **F-D15** and before **F-D16**.

Calls **F-D16–F-D22** occur in order. The second selector read and second selector-directed write occur after **F-D22** and before **F-D23**.

Calls **F-D23–F-D29** complete the represented call sequence. The invalid-instruction truncation boundary follows **F-D29**; a normal return is not shown.

These boundaries constrain caller-side observations only. They do not imply that dependencies preserve memory, globals, resources, or external state.

### Argument relationships

**F-D01** receives the input scale.

**F-D06** receives three explicit arguments:

1. The low six bits of the input scale, shifted left fourteen positions.
2. The retained word.
3. The explicit input shifted left thirty positions.

Under the conditional wrapping model, its first argument depends only on the input’s low four bits and places them in bit positions 16–19. Its third argument depends only on the input’s low two bits and places them in positions 30–31.

**F-D11** receives four times the result of **F-D10**. This is the scaled basis of the preceding destination, not necessarily the retained word stored there.

The first selector-directed byte write uses the low eight bits of the byte source shifted left two positions. Thus its byte value is determined by the byte source’s low six bits and is a multiple of four.

**F-D16** receives the byte source shifted left two positions as each of its first two arguments, followed by the byte source shifted left four positions. The first two 32-bit argument patterns are equal.

The opaque 32-bit result of **F-D17** is forwarded unchanged as the sole explicit argument to **F-D18**. No pointer, resource, handle, acquisition, or release meaning is established.

The result of **F-D22**, shifted left two positions, supplies the argument to **F-D23**. It does not replace the retained byte source.

For the second byte write, the byte source is masked to exclude bits 30–31 and then shifted right twenty positions. The byte-width write discards the remaining bits above the low eight. The stored byte is therefore bits 20–27 of the original byte source; bits 28–29 do not contribute.

**F-D28** receives:

1. The result of **F-D27** shifted left two positions.
2. The retained second selector, widened without sign extension.

There is no third selector read before this call.

### Repeated reads and constrained aliasing

The second selector must be read independently after **F-D22**. Substituting the first selector would change the represented behavior. Dependency writes may change the selector location between reads.

Under the conditional 32-bit wrapping model, however, the first selector-directed write cannot directly overwrite the later selector-read location by numerical address equality:

- The lookup address always has its low 28 bits equal to 1.
- Its possible numerical addresses are \(1+k\cdot2^{28}\), for \(k\) from 0 through 15.
- Each selector-directed write targets an address from 17 through 272.
- No address in that destination range equals any of those lookup addresses.

This exclusion applies to both selector-directed byte writes. It follows from the represented address arithmetic, not from an assumed object boundary or memory-protection rule.

Other overlaps are not generally excluded. The two selector-directed writes can target the same numerical address when their selectors are equal. Word writes can overlap other accesses where their addresses and four-byte footprints permit; address equality alone is not sufficient to analyze word-versus-byte overlap. There is no general assertion that all direct addresses can coincide.

Whether distinct numerical addresses alias through an unknown address-space mapping is unresolved. Such mapping-based aliasing is not established by the arithmetic.

## Side effects

### Memory and external state

The direct access counts and ordering above describe only the caller’s visible operations. Dependencies may perform additional reads, writes, I/O, allocation, release, or other effects.

A call with no explicit arguments is not guaranteed to consume no ambient inputs. No global-preservation guarantee follows from an unused result or an empty explicit argument list.

No synchronization, atomic transaction, rollback, or explicit external-state preservation is represented. Concurrency and volatile-access semantics are unknown.

### Ownership

The caller establishes no allocation, deallocation, reference-count change, or resource ownership transfer.

This is an evidence limitation, not a promise that dependencies lack such effects. In particular, forwarding the result of **F-D17** to **F-D18** does not establish an acquire-and-release pair.

### Preconditions and errors

Successful traversal requires every reached access and indirect call to be executable in the actual environment. The caller does not visibly enforce that requirement.

There are no visible checks for inaccessible addresses, invalid call targets, failure-indicating results, arithmetic overflow, or unsupported alignment. No recovery or rollback follows a failed operation in the represented prefix.

An interrupted execution may leave earlier effects in place. Whether a faulting access can have partial effects is unknown.

## Dependencies

All dependency identities below are opaque occurrence IDs.

| IDs | Explicit caller-side role |
|---|---|
| **F-D01** | Receives the input scale; its target depends on the incoming link-register value. |
| **F-D02–F-D04** | Ordered calls with no explicit arguments and no consumed results. |
| **F-D05** | Supplies the basis of the retained word and first result-addressed destination. |
| **F-D06** | Receives two input-derived bit patterns and the retained word. |
| **F-D07–F-D09** | Ordered calls with no explicit arguments and no consumed results. |
| **F-D10** | Supplies the second result-addressed destination’s basis. |
| **F-D11** | Receives four times the **F-D10** result. |
| **F-D12–F-D14** | Ordered calls with no explicit arguments and no consumed results. |
| **F-D15** | Supplies the unsigned byte source retained across both byte writes. |
| **F-D16** | Receives three shifted forms of the byte source. |
| **F-D17** | Supplies an opaque 32-bit value. |
| **F-D18** | Receives that value unchanged. |
| **F-D19–F-D21** | Ordered calls with no explicit arguments and no consumed results. |
| **F-D22** | Supplies the argument basis for **F-D23**. |
| **F-D23** | Receives the **F-D22** result shifted left two positions. |
| **F-D24–F-D26** | Ordered calls with no explicit arguments and no consumed results. |
| **F-D27** | Supplies the first argument basis for **F-D28**. |
| **F-D28** | Receives a shifted result and the retained second selector. |
| **F-D29** | Final visible call, with no explicit arguments or consumed result. |

Only **F-D01** has a target derived from the incoming link-register value. The remaining targets use fixed address-like bases and fixed displacements. **F-D16** uses a displacement of 4076; the others use 4094. These constructions establish neither valid executable targets nor function boundaries.

No dependency has an established error convention, ownership contract, global-preservation guarantee, or normal-return guarantee.

## Acceptance examples

These are conditional contract examples, not test results. Unless stated otherwise, they assume the conditional 32-bit wrapping model, valid accesses and call targets, and returning dependencies.

### Input-derived values

For explicit input 5:

- The initial word-write address is `0x28`.
- **F-D01** receives 20.
- **F-D06** receives `0x00050000` as its first argument and `0x40000000` as its third.
- The selector-read address is `0x50000001`.

The initial stored word still comes from the ambient register-derived input.

### Different word-destination scales

If **F-D05** returns 3 and **F-D10** returns 7:

- The first result-addressed write stores 12 at `0x44`.
- The second stores the same 12 at `0x30`.
- **F-D11** receives 28.

Using a fourfold rather than sixteenfold scale for the first destination would violate this relationship.

### Retained byte source

If **F-D15** supplies `0x12345678`:

- The first selector-directed write stores `0xE0`.
- **F-D16** receives `0x48D159E0`, `0x48D159E0`, and `0x23456780`.
- The second selector-directed write stores `0x23`.

Changing **F-D22**’s result does not change these byte-source relationships.

### Independent selectors

If the first selector is `0x20` and the second is `0x40`:

- The first byte-write destination is `0x31`.
- The second is `0x51`.
- **F-D28** receives 64 as its second argument.

A later dependency write to the lookup location does not replace the already retained second selector.

### No numerical self-overwrite of the lookup

For every explicit input under the conditional wrapping model, the lookup address has low 28 bits equal to 1. Neither selector-directed destination, restricted to 17 through 272, can equal it.

Consequently, the first selector-directed write cannot itself change the second selector through direct numerical overlap. Different selector values remain possible through intervening dependency writes or other externally supplied changes. Aliasing through an unknown mapping is documented as unresolved, not accepted as an established explanation.

### Interrupted execution

If the second result-addressed write faults and execution stops there, preceding calls and writes may already have taken effect. **F-D11** and later operations have not been reached. No rollback or normal return is established for that path.

## Uncertainties

### documented_only

The following remain unresolved:

- Validity and executability of the represented instruction stream.
- Actual entry conditions and calling convention.
- Origins and required relationships of ambient register inputs.
- Concrete dependency identities, behavior, and return guarantees.
- Constructed target validity.
- Actual treatment of signed arithmetic and signed/unsigned conversions.
- Pointer width, address conversion, mapping, alignment, endianness, and fault semantics.
- Whether distinct numerical addresses alias through the environment’s mapping.
- Concurrency, synchronization, and volatile-access rules.
- Resource meanings and ownership effects inside dependencies.
- Behavior beyond the truncation boundary.
- Any complete-function success criterion.

The numerical non-overlap between selector reads and selector-directed writes is established only within the stated wrapping address model. It does not resolve unknown physical or logical address mappings.

### Inferred intent

Scaling and byte extraction are consistent with indexing or bit-field manipulation, but neither interpretation is established. No application-level intent, ownership rule, or global-preservation guarantee is required by this contract.

## Evidence

The supplied static representation supports:

- One explicit signed 32-bit input and two relevant ambient register-derived inputs.
- Twenty-nine indirect-call occurrences.
- Three direct word writes, two unsigned byte reads, and two byte writes.
- The retained-value dependencies, masks, shifts, and observation order specified above.
- A bad-instruction warning and terminal control-flow truncation rather than a demonstrated return.

The selector-address exclusion is a mathematical consequence of the address formulas under the explicitly conditional 32-bit wrapping model. It is not runtime evidence or an assumption that arbitrary numerical addresses map to disjoint storage.

No runtime tests, execution traces, dependency inspection, or instruction-level validation have been performed.
