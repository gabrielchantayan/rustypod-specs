<!-- Trusted provenance: F-bebadae17ce5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-bebadae17ce5

## Purpose

F-bebadae17ce5 performs a fixed sequence of indirect calls, narrow and word-sized memory accesses, a software interrupt, and a final unresolved indirect transfer.

The visible data movement includes:

- Storing two incoming 16-bit values into separate memory locations.
- Loading an unsigned 16-bit index.
- Storing a 16-bit value obtained through an indirect call.
- Loading a 32-bit base value and using it to read an indexed byte.
- Recording the third explicit argument and the zero-extended byte in two fields relative to that base.
- Writing the low byte of the base value to an address formed by adding the loaded index and loaded byte.

No application-level purpose is established. In particular, the indexed byte access does not establish that the base denotes a table, and the field writes do not establish an object type or ownership relationship.

This contract describes the visible operations and their ordering. It does not assert that the supplied representation is a complete or reliable recovery of the original calling convention or control flow.

## Interface

### Explicit arguments

The supplied interface has four 32-bit argument slots:

| Argument | Visible use |
|---|---|
| First argument | No explicit use in the visible body. |
| Second argument | No explicit use in the visible body. |
| Third argument | Used as an address base for a 16-bit write at offset `0x3C` and a 32-bit read at offset `0x68`; subsequently stored as a 32-bit value. |
| Fourth argument | Used as an address base for an unsigned 16-bit read at offset `0x1C`. |

The absence of explicit uses of the first two arguments does not establish that indirect callees cannot observe them through an unresolved calling convention or shared machine state.

### Additional machine-state inputs and outputs

The visible behavior depends on state beyond the four explicit arguments:

- An incoming link-register value determines the first indirect call target.
- An incoming 16-bit register-associated value supplies the first two 16-bit stores.
- An incoming register-associated address supplies the base for another 16-bit store at offset `0x3C`.
- A register-associated result of the first indirect call supplies the base for a 16-bit store at offset `0x24`.
- A 16-bit register-associated result of a later indirect call supplies the value stored through the incoming address base.
- An unresolved indirect-transfer target is needed after the software interrupt.
- Two global operands and a stack-relative address participate in the final transfer.

These are observable dependencies, not a recovered portable function signature. Their initialization and preservation requirements remain undocumented.

### Widths and layout

The visible representation uses 8-bit byte accesses, 16-bit halfword accesses, and 32-bit word accesses and arguments.

The following locations are accessed directly:

| Address expression | Access |
|---|---|
| First call’s address result plus `0x24` | Write 16 bits |
| Third argument plus `0x3C` | Write 16 bits |
| Fourth argument plus `0x1C` | Read 16 bits, unsigned |
| Incoming address base plus `0x3C` | Write 16 bits |
| Third argument plus `0x68` | Read 32 bits |
| Loaded base plus loaded unsigned index | Read 8 bits, unsigned |
| Loaded base plus `0x4C` | Write 32 bits |
| Loaded base plus `0x3C` | Write 32 bits |
| Loaded unsigned byte plus loaded unsigned index | Write 8 bits |

No padding, total object size, allocation boundary, endianness, or alignment policy is established beyond these individual accesses.

### Return and preconditions

The recovered interface declares no return value and shows a return after the final indirect transfer. Normal completion is conditional on all calls, the interrupt, and the final transfer permitting execution to continue.

No explicit validation or error return is visible. For successful execution, each reached call target must be executable and each reached memory access must be valid under the actual machine’s access rules. Those are execution prerequisites, not checks performed by this function.

## Behavior

### Initial calls and halfword accesses

The function first invokes F-000000000001. Its target is derived from the incoming link-register value by adding `0xF10`.

After that call returns, the function writes the incoming 16-bit value to offset `0x24` from an address supplied through the call’s register-associated result.

It then invokes F-000000000002 and writes the same represented incoming 16-bit value at offset `0x3C` from the third explicit argument.

It invokes F-000000000003 and then reads an unsigned 16-bit value at offset `0x1C` from the fourth explicit argument. This value is the saved index used later. Its numeric range is 0 through 65,535.

It next invokes F-000000000004. A 16-bit register-associated result of this call is written at offset `0x3C` from the incoming address base.

These operations occur in this order. The intervening calls must not be treated as inert: they may affect memory that a later explicit access reads or writes.

### Base capture and first adjusted call argument

After invoking F-000000000005, the function reads a 32-bit value at offset `0x68` from the third argument. This captured value serves as the base for subsequent indexed access and two word-sized stores.

The function then invokes F-000000000006, followed by F-000000000007. The latter has a visible 32-bit result.

F-000000000008 is invoked with one explicit argument: the result of F-000000000007 increased by `0xAF`.

There is no visible overflow check for that addition. At the machine-value level, a 32-bit interpretation retains the low 32 bits. The supplied representation does not establish a source-language signed-overflow policy.

### Indexed byte read and word-sized stores

After invoking F-000000000009, the function reads one byte at:

**captured base + saved unsigned 16-bit index**

The byte is interpreted as unsigned and extended to a 32-bit value in the range 0 through 255.

The function then invokes F-000000000010 and writes the third explicit argument as a 32-bit value at offset `0x4C` from the captured base.

It next invokes F-000000000011 and writes the zero-extended byte as a full 32-bit value at offset `0x3C` from the same captured base.

The latter operation is not a byte store. All 32 bits of that field are replaced; the upper 24 bits of the stored value are zero.

The base, index, and byte are represented as captured values. There is no explicit reload of the third argument’s `0x68` field, the fourth argument’s `0x1C` field, or the indexed byte before these stores. Changes to those memory locations by later callees therefore do not, in the supplied representation, select replacement values for the pending explicit operations.

### Byte store through a small integer-derived address

After invoking F-000000000012, the function performs an 8-bit store at:

**saved unsigned byte + saved unsigned 16-bit index**

The stored value is the low eight bits of the captured base.

The captured base is not included in this destination address. This distinction is essential: the earlier read is base-relative, while this write uses only the two unsigned values.

The destination’s numeric range is 0 through 65,790. Their sum cannot overflow 32 bits. Whether addresses in that range are mapped, writable, meaningful, or prohibited is unknown.

No bounds check or low-address check precedes this write.

### Final calls, interrupt, and transfer

The function invokes F-000000000013 and obtains a visible 32-bit result.

It invokes F-000000000014 with that result decreased by `0xA6`, then invokes F-000000000015 with no explicit arguments shown.

The subtraction has no visible underflow check. A 32-bit machine-value interpretation retains the low 32 bits; no higher-level signed arithmetic contract is established.

The function then executes software interrupt `0x4A`.

If execution continues, it performs an unresolved indirect transfer, labeled F-000000000016, with four represented arguments:

1. A global operand interpreted as an address-bearing value.
2. One byte read at offset 9 from that address-bearing value.
3. A second global operand.
4. An address at offset `0x12C` within the current stack frame.

The supplied representation does not establish the evaluation order of those final arguments. In particular, it does not prove how many times the first global operand is fetched, whether it is stable between evaluations, or the signedness and widening convention of the byte argument.

A return is shown after the transfer. It is not established that the transfer actually behaves as an ordinary returning call.

## Side effects

### Explicit memory changes

On a path that reaches all operations, six explicit stores occur:

1. A 16-bit store through the first call’s address result.
2. A 16-bit store relative to the third argument.
3. A 16-bit store through the incoming address base.
4. A 32-bit store of the third argument relative to the captured base.
5. A 32-bit store of the unsigned byte relative to the captured base.
6. An 8-bit store through the index-plus-byte address.

There is no explicit rollback. If a later access or call fails, earlier completed effects are not visibly undone.

### Aliasing and ordering

No non-aliasing guarantee is present. The explicit arguments, incoming address base, call-provided address, captured base, globals, stack region, and final byte-store destination may overlap.

Consequently:

- An earlier store can affect a later read.
- A later store can partially or fully overwrite an earlier store.
- The 16-bit and 32-bit writes at offset `0x3C` can overlap if their bases coincide.
- A callee can change memory before the next explicit access.

The specified access order must be preserved when evaluating such cases. Final-memory expectations require explicit assumptions about aliasing and callee behavior.

### Ownership and lifetime

No allocation, deallocation, reference-count update, or ownership transfer is explicitly identified.

Storing the third argument does not establish ownership of the referenced storage. Passing a stack-relative address does not establish permission to retain it after the function returns. Unknown callees must not be assumed either to preserve globals or to acquire or release memory.

## Dependencies

The following opaque IDs identify unresolved invocation sites for this contract. They do not establish recovered function identities, distinct implementations, names, or semantic roles.

| Opaque ID | Established interaction |
|---|---|
| F-000000000001 | Incoming-link-relative target; register-associated address result used afterward. |
| F-000000000002 | Called before the third-argument-relative halfword store. |
| F-000000000003 | Called before reading the unsigned index. |
| F-000000000004 | Supplies the represented 16-bit value for the incoming-base-relative store. |
| F-000000000005 | Called before capturing the base from the third argument. |
| F-000000000006 | Called after base capture. |
| F-000000000007 | Supplies the result adjusted upward by `0xAF`. |
| F-000000000008 | Receives that adjusted value. |
| F-000000000009 | Called before the indexed byte read. |
| F-000000000010 | Called before storing the third argument. |
| F-000000000011 | Called before storing the zero-extended byte. |
| F-000000000012 | Called before the index-plus-byte destination write. |
| F-000000000013 | Supplies the result adjusted downward by `0xA6`. |
| F-000000000014 | Receives that adjusted value. |
| F-000000000015 | Last represented call before the software interrupt. |
| F-000000000016 | Unresolved post-interrupt transfer with four represented arguments. |

Except for F-000000000001 and F-000000000016, targets are expressed through fixed code-location-derived arithmetic in the supplied representation. Their exact addresses provide no established behavioral meaning.

Calls shown without explicit arguments must not be interpreted as proven zero-argument interfaces. Register inputs, clobbers, shared-state effects, failure behavior, and memory ownership are undocumented.

## Acceptance examples

These are conditional static expectations, not executed tests.

### Unsigned index and byte expansion

Assume the index read yields `0xFFFF`, the indexed byte yields `0xFE`, and all intervening operations return without altering relevant captured state.

The word written at captured-base offset `0x3C` is `0x000000FE`. The later byte-store destination is `0x100FD`, not the captured base plus `0x100FD`.

### Distinct source and destination address formulas

Assume the captured base is `0x20000134`, the index is 5, and the byte read at address `0x20000139` is 7.

The function later writes byte `0x34` to address 12. It does not write that byte to address `0x20000140`.

Reaching and completing this operation requires address 12 to be writable under the execution environment.

### Repeated offset with different widths

Assume the captured base equals the third argument and no callee changes the relevant fields.

The early write at offset `0x3C` replaces two bytes. The later write at the same offset replaces four bytes with the zero-extended indexed byte. The later store therefore supersedes the earlier halfword in the overlapping region.

### Captured field changes after the read

Assume the base has been captured and a subsequent callee changes the third argument’s field at offset `0x68`.

The later explicit indexed read and word stores still use the previously captured base in the supplied representation. No second field read is shown.

### Failure before the final transfer

Assume the final byte-store destination is not writable and the machine faults on that access.

The visible sequence provides no recovery path or rollback. F-000000000013 and subsequent operations are not reached unless the environment resumes execution past the fault.

## Uncertainties

The following remain **documented_only**:

- The true calling convention and hidden input/output register assignments.
- The reliability of the recovered control flow and indirect target expressions.
- Whether any represented call fails, does not return, or changes execution context.
- Callee effects on memory, globals, ownership, and machine state.
- Global operand widths, volatility, and final argument evaluation details.
- Stack-region extent, initialization, layout, and lifetime requirements.
- Alignment enforcement, endianness, address-space mapping, and access-fault behavior.
- The semantics and return behavior of software interrupt `0x4A`.
- Whether the final transfer returns normally.
- Source-language arithmetic rules for overflow in the two adjusted call arguments.
- The meaning, if any, of the low-address byte store.

A state-recording or table-processing interpretation is possible but unestablished. It must not replace the explicit access formulas or justify additional validation, retries, allocation, cleanup, or error handling.

## Evidence

Established static evidence consists solely of the supplied function representation: its explicit widths, offsets, ordering, arithmetic, indirect invocations, interrupt, and unresolved final transfer.

The role names used here—such as “captured base,” “saved index,” and “unsigned byte”—describe local dataflow, not recovered application types or intent.

No runtime tests have been performed. The acceptance examples are consequences of the visible operations under their stated assumptions, not observations of execution.
