<!-- Trusted provenance: F-44cd36afc2b3; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-44cd36afc2b3

## Purpose

F-44cd36afc2b3 performs an ordered sequence of external calls, coprocessor operations, and narrow memory writes. The supplied representation ends at an invalid-instruction boundary rather than a normal return.

No application-level purpose is established. In particular, the evidence does not establish that this function is a numeric conversion routine, device initializer, allocator, or decoder. Its arithmetic-heavy call sequence and unusual memory accesses are insufficient to assign such a role.

This contract distinguishes:

- **Established static evidence:** operations and ordering explicitly represented in the supplied material.
- **Conditional interpretation:** consequences under a 32-bit, two’s-complement execution model with arithmetic signed right shifts and wrapping address arithmetic.
- **documented_only unknowns:** behavior that requires dependency implementations, machine instructions, or runtime evidence not supplied here.

The invalid-instruction warning limits confidence in treating the entire representation as a faithful executable path.

## Interface

### Explicit inputs

The represented interface accepts four positional arguments:

| Argument | Represented width | Observable use |
|---|---:|---|
| First | 32 bits, interpreted as signed for the displayed expression | Passed to the first dependency both unchanged and shifted left by four bits. |
| Second | 32 opaque bits | Passed unchanged to the first dependency. |
| Third | 32 opaque bits | No explicit use is represented. |
| Fourth | 32 bits, interpreted as signed in right-shift expressions | Used in a dependency argument and in the later address-selection expression. |

The unused third argument is not evidence that its incoming machine-register contents are irrelevant to every unknown dependency. It establishes only that the represented function body does not explicitly reference that argument.

### Implicit inputs

The representation additionally depends on:

- Two unresolved signed 32-bit register values, designated **R5** and **R7** here.
- An unresolved 32-bit secondary result associated with the initial call sequence.
- Another unresolved 32-bit secondary-result value used in the final call before the memory phase.
- Two unresolved coprocessor-state operands.
- A stack-provided pointer located at the represented entry-stack displacement `0x3c8`.
- Contents of several computed memory locations and one fixed 32-bit memory location.

R5 and R7 are architectural register designations, not established formal parameters. Their definitions and stability across dependencies are unknown.

### Results and termination

No ordinary return value is represented. The supplied interface is void-like, and the visible path ends in an invalid-instruction termination marker.

The contract does not establish:

- A successful return to the caller.
- A standardized error result.
- A particular exception, signal, trap number, or process exit status.
- That execution reaches the final marker rather than faulting or transferring control earlier.

### Preconditions

There is no explicit input validation.

Completion of the represented sequence requires all invoked dependencies and coprocessor operations to complete, and every reached memory access to be permitted by the execution environment. This includes accesses at very low addresses. Ordinary hosted-process null-pointer assumptions would make portions of this path inaccessible.

No ownership transfer, allocation requirement, or release obligation can be established from this interface.

## Behavior

### Width and arithmetic conventions

Words are 32 bits, halfwords are 16 bits, and bytes are 8 bits. Represented 64-bit call results are subsequently split into low and high 32-bit halves.

The call descriptions below use these terms:

- **Pair result:** a represented 64-bit result subsequently consumed as two 32-bit words.
- **Duplicate-low call:** three arguments consisting of the low word, the high word, and the low word again.
- **Signed shift:** the representation applies right shift to a signed 32-bit value.
- **Unsigned shift:** zeros are shifted into the high bits.

These terms describe visible dataflow, not a confirmed calling convention. A pair result might reflect two result registers rather than a single language-level 64-bit object.

Signed left shifts and address calculations require special caution. Where an expression discards high bits, derived addresses below are conditional on 32-bit machine-style wrapping. The supplied representation alone does not establish language-level behavior for signed overflow.

### Ordered dependency phase

The following opaque dependency IDs are contract-local identifiers for distinct visible call targets. They do not assert recovered names or semantic identities.

The calls occur in the following order, subject to earlier calls completing:

| Dependency | Inputs and visible result use |
|---|---|
| F-000000000001 | Receives the first explicit argument, the second explicit argument, and the first argument shifted left by four. Its ordinary result is not used explicitly. |
| F-000000000002 | Receives the unresolved secondary result shifted left by three. Produces a 32-bit word. |
| F-000000000003 | Receives that word and produces a pair result. |
| F-000000000004 | Receives a duplicate-low call from the preceding pair. Produces a word. |
| F-000000000005 | Receives that word and produces a pair. Its low word is retained. |
| F-000000000006 | Receives a duplicate-low call from that pair. Its result is not explicitly consumed. |
| F-000000000007 | Receives the retained low word shifted right unsigned by two, then right signed by ten. Produces a pair. |
| F-000000000008 | Receives a duplicate-low call from that pair. Produces another pair. The low word from F-000000000007 remains separately relevant. |
| F-000000000009 | Receives the pair from F-000000000008 and a third argument obtained by applying the unsigned-two-then-signed-ten shift to the low word from F-000000000007. Produces a pair. |
| F-000000000010 | Receives a duplicate-low call from that pair. Produces a word. |
| F-000000000011 | Receives that word and produces a pair. Its low word is retained for the last call before the memory phase. |
| F-000000000012 | Receives a duplicate-low call from that pair. Produces a word. |
| F-000000000013 | Receives that word and produces a pair. |
| F-000000000014 | Receives a duplicate-low call from that pair. Its result is not explicitly consumed. |
| F-000000000015 | Receives R5 shifted right signed by ten. Produces a pair. |
| F-000000000016 | Receives a duplicate-low call from that pair. Produces a pair. |
| F-000000000017 | Receives that pair and R5 shifted right signed by ten as the third argument. Its result is not explicitly consumed. |
| F-000000000018 | Receives R5 shifted right signed by ten. Produces a pair. |
| F-000000000019 | Receives that pair and R7 shifted right signed by ten as the third argument. Its result is not explicitly consumed. |
| F-000000000020 | Receives R7 shifted right signed by ten. Produces a pair. Its high word is interpreted as signed and shifted right by eleven. |
| F-000000000021 | Receives that pair and the shifted high word as its third argument. Its result is not explicitly consumed. |
| F-000000000022 | Receives the shifted high word and produces a word used by the first coprocessor operation. |
| F-000000000023 | After the first coprocessor operation, receives the fourth explicit argument shifted right signed by eleven. Produces a pair. |
| F-000000000024 | Receives that pair and R5 shifted right signed by eleven as the third argument. Its result is not explicitly consumed. |
| F-000000000025 | After two more coprocessor operations, receives three arguments: twice the unsigned-two-then-signed-ten reduction of the retained low word from F-000000000011; the unresolved secondary-result value; and R7 shifted right signed by eleven. Its result is not explicitly consumed. |

An unconsumed result does not imply an unnecessary call. Any of these dependencies may have observable side effects.

For the unsigned-two-then-signed-ten reduction, the intermediate value after the first shift cannot have its sign bit set. Under the stated 32-bit model, the combined reduction therefore equals an unsigned right shift by twelve. Its maximum is `0x000fffff`; doubling it yields at most `0x001ffffe`.

### Coprocessor operations

Three coprocessor operations are represented:

1. Immediately after F-000000000022, its 32-bit result supplies the data operand for the operation identified by coprocessor number 8, selector 1, and encoded operand 4.
2. Immediately after F-000000000024, R5 shifted right signed by eleven supplies the data operand for the operation using selector 4.
3. A further operation uses the same shifted value with selector 6.

The latter two operations also identify coprocessor number 8 and encoded operand 4. All three include the unresolved coprocessor-state operands.

The affected hardware state, privilege requirements, and exception behavior are documented_only.

### Address selection and initial memory effects

After F-000000000025, a 32-bit address-like selector is derived from the fourth explicit argument using signed and unsigned shifts.

Under the conditional 32-bit model:

- A nonnegative fourth argument produces selector `0x00000000`.
- A negative fourth argument produces selector `0xffffffff`.

A base address is then formed by multiplying this selector by four. It is consequently either `0x00000000` or `0xfffffffc`.

The next represented memory operations are ordered as follows:

1. Read an **unsigned 16-bit** value at the address formed from R7 shifted right signed by fourteen, multiplied by `0x04000000`, plus `0x20`.
2. Zero-extend that value and write it as a **32-bit word** at base plus `0x20`.
3. Test whether the base is zero.

Thus the 32-bit store precedes the zero-base branch. The branch is not a null-address guard for that store.

Under wrapping address arithmetic, the destination is `0x20` on the zero-base path and `0x1c` on the other path.

### Zero-base path

If the base is zero:

1. Read a 32-bit word from the selector address, which is address zero under the conditional model.
2. Treat that word as an address and write a zero halfword at that address plus two.
3. Select fixed address `0x0895b6c8` as the source of the subsequent 32-bit load.
4. Select the stack-provided pointer as the source of a later signed halfword argument, obtained from its sixth 32-bit element.

The word read at address zero also replaces the earlier unsigned-halfword value as the source of the later byte write.

### Nonzero-base path

If the base is nonzero:

- The base remains the source address for the subsequent 32-bit load.
- The base also remains the pointer used to obtain the sixth 32-bit element for the later signed halfword argument.
- The earlier unsigned-halfword value remains the source of the later byte write.

Under the conditional model, these word reads occur at `0xfffffffc` and `0x10`, respectively.

### Common memory tail

Both paths then perform the following sequence:

1. Read a signed 32-bit index from the selected source address.
2. Write the low eight bits of the selected byte-source value to index multiplied by `0x80`.
3. Form a halfword-address base from the index shifted left by ten.
4. Read the sixth 32-bit element through the selected pointer and narrow it to a signed 16-bit argument.
5. Invoke F-000000000026 with:
   - The branch-selected base shifted right signed by thirteen.
   - The original selector masked to its low 30 bits, shifted right unsigned by fourteen, then shifted left by nine.
   - The signed 16-bit value just described.
6. At the returned address plus `0x30`, write the low 16 bits of that returned address.
7. At the halfword-address base plus two, write the low 16 bits of that base.
8. At index multiplied by negative `0x40`, plus two, write a zero halfword.
9. At the halfword-address base itself, write a zero halfword.
10. Write the 32-bit value `0x0895bb34` to fixed address `0x0895bb8c`.
11. At index multiplied by negative `0x40`, plus eight, write byte `0x34`.
12. Write the 32-bit value `0x0895bb34` to fixed address `0x0895bb34`.
13. Reach the invalid-instruction termination marker.

The index is obtained before F-000000000026. Later address computations use that retained value, not a represented reload from its original memory location.

## Side effects

Directly represented side effects comprise:

- Twenty-six external calls.
- Three coprocessor operations.
- Path-dependent memory reads and writes.
- Two fixed-address word writes.
- Abnormal termination or an unresolved continuation at the invalid-instruction boundary.

The stated order is material. Computed addresses may alias each other, fixed storage, or memory accessed by dependencies. No disjointness guarantee is established.

The contract specifies access widths, not byte order or atomicity. Neither endianness nor synchronization semantics is established.

There is no visible allocation or release operation. That does not establish that dependencies allocate nothing, release nothing, or preserve caller-owned memory. No dependency may be assumed to preserve globals merely because its return value is ignored.

There is no visible rollback. If an operation faults, the preceding completed side effects may remain.

## Dependencies

F-000000000001 through F-000000000025 are the ordered call targets described above. Their internal behavior, errors, ownership conventions, and global-state effects are documented_only.

F-000000000026 accepts two represented 32-bit arguments and one signed 16-bit value. Its result is used directly as an address without a visible success check. No allocation or validity guarantee is established for that address.

Additional execution dependencies are:

- A compatible coprocessor interface.
- An environment permitting the reached memory accesses.
- An ABI explaining the unresolved register and secondary-result values.
- A valid interpretation of the instruction stream through the reported invalid-data boundary.

## Acceptance examples

These are conditional static acceptance criteria, not results of executed tests.

### Nonnegative fourth argument

Given a nonnegative fourth argument and the stated machine model, the selector and base are zero. If execution reaches the memory phase:

- The initial word store targets `0x20`.
- That store occurs before the read at address zero.
- The halfword clear through the word read at zero precedes the index load from `0x0895b6c8`.
- F-000000000026 receives zero for both its first and second arguments.

A reconstruction that rejects the zero base before the store would not preserve the represented order.

### Negative fourth argument

Given a negative fourth argument and wrapping 32-bit address arithmetic:

- The selector is `0xffffffff`.
- The initial word store targets `0x1c`.
- The zero-base path is not taken.
- The index is read from `0xfffffffc`.
- The sixth-element word is read from `0x10`.
- F-000000000026 receives first argument `-1` and second argument `0x01fffe00`.

These accesses remain conditional on the environment permitting them.

### Halfword reduction

If the retained low word from F-000000000011 is `0xffffffff`, the first argument to F-000000000025 is `0x001ffffe`, not a negative value. This follows from the unsigned shift occurring before the signed shift.

### Dependency-result write

If F-000000000026 returns address `0x12345678`, and the following access succeeds, the next represented store writes halfword `0x5678` at `0x123456a8`.

The result is not dereferenced before adding `0x30`.

### Partial completion

If the store through the result of F-000000000026 faults, the earlier byte store has already been attempted, while the subsequent halfword clears and fixed-address word writes have not yet been reached.

No recovery behavior is represented.

## Uncertainties

- **documented_only — Control-flow reliability:** The supplied material explicitly reports bad instruction data and a truncated ending. Reachability and completeness are not independently established.
- **documented_only — Dependency semantics:** All external-call implementations are absent.
- **documented_only — Implicit register provenance:** R5, R7, and both secondary-result values lack complete definitions.
- **documented_only — ABI details:** Argument extension, register preservation, pair-result transport, and stack interpretation are unconfirmed.
- **documented_only — Overflow semantics:** Derived wrapping addresses describe a conditional machine model, not guaranteed behavior of signed source-language arithmetic.
- **documented_only — Memory environment:** Accessibility, alignment tolerance, mapped-device behavior, endianness, and aliasing are unknown.
- **documented_only — Coprocessor effects:** Hardware meaning and fault behavior are unavailable.
- **documented_only — Terminal behavior:** The final marker does not establish a specific runtime trap or prove that no continuation exists in a corrected instruction interpretation.

Any application-level explanation beyond the described effects would be inferred intent rather than established behavior.

## Evidence

This contract is based exclusively on the supplied static representation: its explicit argument uses, widths, arithmetic expressions, call-result dataflow, memory-access order, branch, coprocessor operations, fixed writes, and invalid-instruction warnings.

The opaque dependency IDs are local references introduced for this document. They do not carry semantic claims.

No dependency bodies, original instruction bytes, runtime traces, or execution-environment descriptions were supplied. No runtime tests have been performed.
