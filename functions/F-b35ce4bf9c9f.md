<!-- Trusted provenance: F-b35ce4bf9c9f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b35ce4bf9c9f

## Purpose

The supplied static representation describes a procedure that performs an ordered sequence of indirect calls, two 32-bit memory writes, four byte writes, and four byte reads used to select byte-write destinations. It ends at an explicitly reported invalid-instruction boundary rather than an established normal return.

No application-level purpose is established. In particular, the evidence does not justify interpreting the operations as allocation, serialization, device control, initialization, or cleanup.

This contract distinguishes:

- **Established static evidence:** operations and ordering explicitly represented in the supplied material.
- **Inferred interpretation:** a 32-bit machine-word interpretation used to describe arithmetic and addresses.
- **documented_only unknowns:** behavior that cannot be established from this representation, especially indirect-callee effects and execution beyond the invalid-instruction boundary.

All behavioral statements below are conditional on execution reaching the relevant operation. An earlier call, memory fault, or other interruption may prevent later operations.

## Interface

### Explicit inputs

The represented interface has two inputs:

| Input | Represented width and interpretation | Use |
|---|---|---|
| First input | 32-bit signed integer | Shifted left by two positions; the resulting word is passed to the first dependency and later stored. |
| Second input | 32-bit unsigned integer | Used to calculate the first word-write address and, after a logical right shift, the fourth argument to the first dependency. |

The signed interpretation of the first input does not establish signed-overflow behavior. Several represented operations involve signed left shifts or signed arithmetic. Their exact instruction-level behavior remains unknown.

### Implicit incoming state

Three additional incoming quantities are consumed without an assignment establishing their values inside the visible procedure:

- **Incoming word:** a represented 32-bit value stored by the first memory write.
- **Selector base:** an address-like value used for two byte reads at offset one.
- **Entry target anchor:** a value used to calculate the first indirect-call target by adding 4,076 bytes.

These are interface requirements even though they are absent from the explicit two-input signature. The available material does not establish how callers supply them or what calling convention preserves them.

### Result and completion

The represented return type is void. No explicit result value or successful return is shown.

The visible sequence terminates at an invalid-instruction marker. That marker does not establish whether the real execution traps, transfers control elsewhere, or follows instructions not recovered in the supplied representation. A consumer must not treat normal completion as an established postcondition.

### Notation and arithmetic model

The following names are explanatory labels, not additional parameters:

- **Scaled input:** the first input shifted left by two positions.
- **Pair low** and **pair high:** the lower and upper 32-bit halves of the 64-bit result from dependency D02.
- **Index word:** the represented 32-bit result from D07.
- **Source word:** the unsigned 32-bit result from D12.
- **Scaled source:** the source word shifted left by two positions.
- **First later result:** the represented 32-bit result from D15.
- **Second later result:** the represented 32-bit result from D20.
- **Final later result:** the represented 32-bit result from D25.

Where a concrete example needs an arithmetic interpretation, it assumes 32-bit bit-vector arithmetic: discarded high bits do not survive. This is an inferred machine-level interpretation, not a claim that every signed expression in the supplied representation has defined language-level semantics.

A byte read is unsigned and therefore produces a value from 0 through 255. A byte store retains only the low eight bits of its value. Memory endianness is not established.

## Behavior

### 1. Initial scaling and word write

The procedure first forms the scaled input.

Before any dependency call, it writes the incoming word as a 32-bit quantity to the address calculated as:

**four times the second input, plus 20 bytes.**

This is a direct calculated address. The evidence does not show a separate base pointer, object header, array bound, or validation step.

It then invokes D01 with four explicitly represented arguments, in this order:

1. Scaled input.
2. Scaled input.
3. Scaled input.
4. Second input logically shifted right by three positions.

The three equal arguments are values derived before the call; the procedure does not perform three separate reads of caller memory to obtain them.

### 2. Split-result forwarding and second word write

D02 is invoked with no explicitly represented arguments and supplies a 64-bit result. The procedure divides it into two 32-bit halves.

D03 receives, in order:

1. Pair low.
2. Pair high.
3. Pair low again.
4. Pair high logically shifted right by three positions.

D04, D05, and D06 are then invoked in that order, each with no explicitly represented arguments.

D07 is invoked next and supplies the index word. The procedure writes the scaled input as a 32-bit quantity to:

**four times the index word, plus 20 bytes.**

D08 then receives four times the index word as its single explicitly represented argument. The write therefore precedes D08.

D09, D10, and D11 follow, in that order, with no explicitly represented arguments.

### 3. Source-word acquisition and first selected byte write

D12 supplies the source word. The procedure forms the scaled source by shifting that word left by two positions.

It then:

1. Reads one unsigned byte at selector base plus one.
2. Adds 17 bytes to the numeric value of that byte.
3. Writes the low byte of the scaled source to the resulting address.

The byte that is read is used directly as an address selector, not as an offset added back to selector base. Consequently, the destination is numerically between 17 and 272 inclusive. Whether those addresses identify accessible memory, mapped hardware, or something else is unknown.

The stored value is the source word shifted left by two positions and then narrowed to eight bits. Its bottom two bits are necessarily zero.

### 4. Second selected byte write

D13 and D14 are invoked in order. D15 is then invoked and supplies the first later result.

After D15 returns, the procedure performs a fresh unsigned byte read at selector base plus one. It adds 17 to that new byte value and stores one byte at the resulting address.

This second stored byte comes from:

1. Clearing the highest two bits of the source word.
2. Logically shifting the remaining value right by 20 positions.
3. Keeping only the low eight bits.

Thus the stored byte is source-word bits 20 through 27. Source-word bits 28 and 29 survive the intermediate shift but are discarded by the byte store; bits 30 and 31 are cleared before the shift.

The second selector read must not be replaced by the first selector value. Intervening dependency calls or the first store itself may have changed the memory observed by that read.

D16 then receives the first later result shifted left by two positions.

### 5. First zero-byte write and four-argument call

D17, D18, and D19 are invoked in order. D20 then supplies the second later result.

The procedure next:

1. Calculates an address as 1,024 times the scaled source, plus one.
2. Reads one unsigned byte from that address.
3. Adds 17 to the byte value.
4. Stores a zero byte at the resulting address.

Under the 32-bit bit-vector interpretation, the selector-read address is also the source word shifted left by twelve positions, plus one, with high bits discarded. This equivalence is conditional on that arithmetic model.

D21 follows the zero-byte write. Its four explicitly represented arguments are:

1. Scaled source shifted left by three positions.
2. The same value again.
3. Scaled source.
4. Second later result shifted left by two positions.

The first two arguments are shifts of the already scaled source. Under the bit-vector interpretation, each equals the original source word shifted left by five positions, with high bits discarded.

### 6. Second zero-byte write and final calls

D22, D23, and D24 are invoked in order. D25 then supplies the final later result.

The procedure repeats the selector read at:

**1,024 times the scaled source, plus one.**

It uses this freshly read unsigned byte, plus 17, as the destination of another zero-byte store.

This read is distinct from the preceding read at the same calculated address. D21 through D25 may have changed that memory. The first zero store may also alias it. The contract therefore does not require the two zero stores to target the same address.

D26 receives the final later result shifted left by two positions.

D27, D28, D29, and D30 are invoked in order. No further ordinary memory operation or normal return is established before the invalid-instruction boundary.

## Side effects

### Directly represented memory accesses

Excluding any memory activity inside dependencies, the procedure exposes this order:

| Order | Access |
|---|---|
| 1 | Write incoming word: 32 bits at four times the second input plus 20. |
| 2 | Write scaled input: 32 bits at four times the index word plus 20. |
| 3 | Read selector byte at selector base plus one. |
| 4 | Write low byte of scaled source at the selected address plus 17. |
| 5 | Read selector byte again at selector base plus one. |
| 6 | Write source-word bits 20–27 at the newly selected address plus 17. |
| 7 | Read selector byte at 1,024 times scaled source plus one. |
| 8 | Write zero at that selected address plus 17. |
| 9 | Read selector byte again at 1,024 times scaled source plus one. |
| 10 | Write zero at that newly selected address plus 17. |

Dependency calls occur between these accesses as specified in the behavior and dependency sections.

The two word writes may alias one another. Word writes may also overlap selector memory or byte-write destinations. Byte-write destinations may coincide with one another or with selector-read addresses. No non-aliasing condition is established.

A 32-bit write denotes its represented width; it does not establish atomicity, bus transaction count, or byte-write order.

### Ownership and lifetime

No direct allocation, release, ownership transfer, or reference-count operation is established. Dependencies may perform such operations, but their behavior is unknown.

Passing a numeric value that could be an address does not establish that the callee borrows or owns the referenced memory. No guarantee is made that a dependency preserves global memory, selector contents, input-related storage, or externally visible state.

### Errors and preconditions

There are no visible range checks, null checks, alignment checks, status checks, retries, or rollback operations.

Execution of the full visible sequence requires:

- Accessible destinations for both word writes.
- Accessible sources for all four selector-byte reads.
- Writable destinations selected by those byte reads.
- Callable indirect targets with compatible execution behavior.
- Dependency returns that permit the next represented operation to execute.

No error value is produced for an invalid address or failing dependency. If execution stops partway through, prior writes and dependency side effects are not visibly undone.

## Dependencies

The identifiers below are contract-local opaque labels for **call sites**, not claims about distinct function identities. The supplied dependency map provides no resolved identities for these callees. Different sites might ultimately identify the same function.

“No explicit arguments” means none are shown in the supplied representation. It does not prove that the callee receives no meaningful register, stack, or global state.

| ID | Explicit arguments | Result consumed by this procedure |
|---|---|---|
| D01 | Scaled input, scaled input, scaled input, second input shifted right by three | None shown |
| D02 | No explicit arguments | 64-bit pair |
| D03 | Pair low, pair high, pair low, pair high shifted right by three | None shown |
| D04 | No explicit arguments | None shown |
| D05 | No explicit arguments | None shown |
| D06 | No explicit arguments | None shown |
| D07 | No explicit arguments | Index word |
| D08 | Four times index word | None shown |
| D09 | No explicit arguments | None shown |
| D10 | No explicit arguments | None shown |
| D11 | No explicit arguments | None shown |
| D12 | No explicit arguments | Source word |
| D13 | No explicit arguments | None shown |
| D14 | No explicit arguments | None shown |
| D15 | No explicit arguments | First later result |
| D16 | First later result shifted left by two | None shown |
| D17 | No explicit arguments | None shown |
| D18 | No explicit arguments | None shown |
| D19 | No explicit arguments | None shown |
| D20 | No explicit arguments | Second later result |
| D21 | Scaled source shifted left by three, same value, scaled source, second later result shifted left by two | None shown |
| D22 | No explicit arguments | None shown |
| D23 | No explicit arguments | None shown |
| D24 | No explicit arguments | None shown |
| D25 | No explicit arguments | Final later result |
| D26 | Final later result shifted left by two | None shown |
| D27 | No explicit arguments | None shown |
| D28 | No explicit arguments | None shown |
| D29 | No explicit arguments | None shown |
| D30 | No explicit arguments | None shown |

The established call order is D01 through D30, interleaved with the direct memory operations described above.

D01’s target is calculated from the incoming entry target anchor plus 4,076 bytes. The remaining sites use represented fixed-address calculations. Target addresses alone do not establish callee semantics, executable validity, or instruction-state conventions.

A result marked “none shown” is not checked or explicitly consumed as a return value. This does not make the call side-effect-free or removable.

## Acceptance examples

These are conditional static acceptance examples, not executed tests. Each assumes valid memory, valid callees, and that execution reaches the relevant point.

### Initial word write and first call

Given first input 3 and second input 8:

- The scaled input is 12.
- The first word-write address is 52.
- The word written there is the incoming word, not 12.
- D01 receives 12, 12, 12, and 1.

The first write must precede D01.

### Split 64-bit result

If D02 supplies a result whose lower half is 5 and upper half is 24, D03 receives 5, 24, 5, and 3.

The fourth argument is derived from the upper half, not from the entire 64-bit result.

### Indexed word write

If D07 returns 9, the second word-write address is 56, and D08 receives 36. The stored word is the previously formed scaled input.

D08 must not precede that write.

### Source-byte extraction

If the source word is hexadecimal `0x0AB0003F`:

- The low byte of the scaled source is `0xFC`.
- The second nonzero-derived byte value is `0xAB`.

If the first selector read returns 7 and the second returns 9, the respective stores target addresses 24 and 26. They need not target the same byte.

### Selector mutation between zero stores

Suppose the first read at the scaled-source-derived selector address returns 1, while the second returns 255.

The zero stores target addresses 18 and 272 respectively. Reusing the first selector value for both stores would violate the represented behavior.

### Unsigned right-shift boundary

If the second input is hexadecimal `0xFFFFFFFF`, D01’s fourth argument is `0x1FFFFFFF`.

Under the 32-bit bit-vector interpretation, the initial word-write address wraps to 16. This address calculation is an arithmetic example, not evidence that address 16 is valid.

### Interrupted execution

If D15 does not return, the second selector read at selector base plus one and all subsequent operations are not reached. Earlier writes remain potentially observable; no rollback is established.

## Uncertainties

- **documented_only — Control-flow reliability:** The supplied material reports bad instruction data and a truncated endpoint. The visible sequence may not describe complete executable control flow.
- **documented_only — Normal termination:** No successful return, termination status, or post-boundary continuation is established.
- **documented_only — Calling convention:** Argument recovery, implicit incoming values, result conventions, and state preservation are not independently verified.
- **documented_only — Callee behavior:** No dependency body or semantic identity is supplied. Calls may mutate memory, alter external state, fail, or not return.
- **documented_only — Signed arithmetic:** Several signed shifts and multiplications cannot safely be assigned ordinary signed-language semantics. Bit-vector wrapping is an explicit interpretive assumption where used.
- **documented_only — Address space:** Pointer width, address translation, accessible low addresses, alignment requirements, and memory-mapped side effects are not established.
- **documented_only — Memory representation:** Endianness, atomicity, volatility, concurrency rules, and hardware ordering are unknown.
- **documented_only — Ownership:** Allocation, release, borrowing, and lifetime obligations cannot be inferred from the direct operations or numeric arguments.
- **Inferred intent only:** Repeated scaling and selector-based stores might represent structured data manipulation, but no such application-level interpretation is adopted by this contract.

## Evidence

This contract is based solely on the supplied static representation of F-b35ce4bf9c9f.

The representation explicitly supports the two-input interface, implicit incoming state uses, arithmetic expressions, 30 ordered indirect-call sites, consumed return values, two word writes, four selector-byte reads, four byte writes, and the invalid-instruction endpoint.

It does not supply dependency implementations, a verified calling convention, a memory map, a caller contract, executable instruction validation, or a normal-return path.

No runtime tests have been performed. The acceptance examples are deductions from the represented operations under their stated conditions, not observations of execution.
