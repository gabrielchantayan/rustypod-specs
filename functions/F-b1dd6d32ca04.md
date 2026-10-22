<!-- Trusted provenance: F-b1dd6d32ca04; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b1dd6d32ca04

## Purpose

F-b1dd6d32ca04 selects one of two value-producing paths, obtains an integer result, and normally stores its 16-bit representation in caller-provided storage. Its 32-bit return value reports whether that store occurred.

One path recognizes the integer value −1 as a failure sentinel. That sentinel causes an immediate return of 0 without a direct write to the output location. The other path does not perform this sentinel check: if it produces −1, the function stores its 16-bit representation and returns 1.

This is an established distinction in the supplied static evidence. The meaning of the produced value, the reason for choosing either path, and the wider application-level meaning of success are unknown.

## Interface

### Inputs

The function has two explicit inputs:

- **Context address:** represented by an integer-valued parameter in the supplied source. It is passed unchanged to F-7bcec1f9d3df on one path and used as the base address for indirect dispatch on the other.
- **Output address:** a pointer to a writable two-byte destination. On a successful path, the function writes one 16-bit result there.

The supplied representation uses integer intermediates for dependency results and a four-byte return type. This contract interprets those intermediates as 32-bit values and the destination element as 16 bits, consistent with the recovered representation. The source does not independently establish every ABI or pointer-width detail.

### Return value

For executions that complete normally, the return is one of two 32-bit values:

| Return | Meaning established by this function |
|---|---|
| 0 | The indirect dependency returned exactly −1; this function did not perform its output store. |
| 1 | A dependency result was converted to 16 bits and stored at the output address. |

The return value is not the produced value. The produced value is available only through the output destination.

A return of 1 does not establish that the produced value is nonnegative, fits in a signed 16-bit range, or is meaningful to the caller. In particular, a stored all-ones 16-bit value can accompany success.

### Preconditions

For a normally completing execution:

- F-e3c7fd56b220 must be callable under the applicable ABI.
- If F-e3c7fd56b220 returns zero, the context must support the dispatch reads described below, and the selected target must be callable.
- If F-e3c7fd56b220 returns nonzero, the context must satisfy F-7bcec1f9d3df’s requirements.
- If execution reaches the output store, the output address must permit a two-byte write with the alignment required by the target environment.

No explicit validation establishes these conditions. The wrapper does not check for null addresses, invalid dispatch structures, an absent target, or an unwritable output destination.

The indirect-failure path does not directly dereference the output address. This does not establish a general API promise that an invalid output address is accepted: the selected dependency may have its own effects or requirements.

## Behavior

### Path selection

The first explicit action is a call to F-e3c7fd56b220 with no explicit arguments.

Its integer result is used only to distinguish zero from nonzero:

- **Zero selects indirect dispatch.**
- **Any nonzero value selects F-7bcec1f9d3df.**

Positive and negative nonzero selector results take the same branch. The selector’s result is not itself stored in the output destination and does not directly become the wrapper’s return value.

The selector call completes before either value-producing dependency is invoked. Consequently, any changes it makes to memory can affect subsequent dispatch reads or the behavior of F-7bcec1f9d3df. The wrapper does not capture the dispatch structure before calling the selector.

### Zero-selector path: indirect dispatch

When the selector returns zero, the function obtains a callable target through three dependent reads:

1. It reads a pointer-valued field at byte offset 12 from the context address.
2. It dereferences that pointer to obtain another address, used as the dispatch-table base.
3. It reads the callable address from byte offset 32 within that table.

These offsets are byte offsets. The evidence establishes the address relationships, not the semantic identity of the structures. Calling either structure an object, interface, or virtual-method table would be an interpretation rather than an established requirement.

The selected target is then invoked. The supplied call expression contains no explicit arguments. It does not establish any implicit receiver convention or other ABI-specific argument arrangement.

After that call returns, its integer result is compared with exactly −1, before conversion to the output width:

- If the result is −1, the function returns 0 immediately.
- Otherwise, the function converts the result to the 16-bit destination representation, stores it, and returns 1.

No other value is explicitly rejected. Zero, positive values, and negative values other than −1 all reach the output store.

Because the comparison precedes narrowing, distinct integers that would produce the same 16-bit representation can receive different treatment. Under the recovered 32-bit integer interpretation, −1 is rejected here, whereas 65535 is not.

### Nonzero-selector path: direct dependency

When the selector returns nonzero, the function calls F-7bcec1f9d3df with the context value as its sole explicit argument.

The wrapper does not perform the dispatch-chain reads on this path. F-7bcec1f9d3df may independently access the context or other state; its implementation is not supplied.

Whatever integer result this dependency returns is converted to the 16-bit destination representation and stored. The function then returns 1.

There is no explicit error-sentinel test on this path. In particular, −1 does not cause a return of 0.

### Conversion and store

Both successful paths converge on one two-byte output assignment.

There is no explicit range check, clamping, overflow report, or preservation of the full integer value. Under the conventional recovered-machine interpretation, the output contains the low 16 bits of the selected dependency’s result.

The supplied evidence includes conversion through a signed short representation. For values outside the signed 16-bit range, the exact portability rules depend on the original language implementation and target. The intended recovered-machine behavior is truncation to 16 bits; universal language-level portability is not established.

The output store occurs after the selected dependency returns and, on the indirect path, after the −1 comparison. There is no explicit read of the previous output value.

## Side effects

### Directly visible effects

The function directly performs:

- One call to F-e3c7fd56b220.
- One subsequent call to either the indirect target or F-7bcec1f9d3df, provided execution reaches and completes that call.
- Dispatch-related memory reads only when the selector returns zero.
- One two-byte output write on a successful path.
- No output write of its own when the indirect target returns −1.

No explicit allocation, deallocation, resource acquisition, release, or other memory write appears in the wrapper.

### Dependency effects

The dependencies’ effects are unknown. They may modify the context, output-adjacent storage, globals, external resources, or other reachable state.

Accordingly, “no output write” on the indirect-failure path means only that **this wrapper does not execute its final assignment**. It does not guarantee that the destination retains its entry value if a dependency can access and modify it.

No supplied evidence establishes that any dependency preserves globals, transfers ownership, or releases memory.

### Aliasing and ordering

No non-aliasing condition is checked.

If the output location overlaps the context or dispatch-related storage, the final store occurs after the wrapper has completed its dispatch reads and the selected call. Such a store may therefore modify state used by later operations, even though this invocation has already selected and called its target.

Whether such overlap is valid under the original program’s object and language rules is undocumented.

## Dependencies

| Dependency | Explicit inputs | Result used by this function | Established role |
|---|---|---|---|
| F-e3c7fd56b220 | None shown | Integer tested for zero versus nonzero | Selects the value-producing path |
| F-7bcec1f9d3df | Context value | Integer converted to the output width | Produces the value on the nonzero-selector path |
| Unidentified indirect dependency | No explicit arguments shown | Integer first compared with −1, then converted if accepted | Produces the value on the zero-selector path |

No opaque function ID was supplied for the indirect target. Its identity is determined by the dispatch reads, and this contract does not assign it a fabricated identity.

No dependency implementation or independent contract is available. Names or presumed application roles are not evidence of additional behavior.

## Acceptance examples

These are conditional contract examples derived from static evidence, not executed tests. Except where stated otherwise, assume valid memory, normally returning dependencies, and no dependency-side modification of the output.

1. **Indirect success with a small value.**  
   If F-e3c7fd56b220 returns zero and the selected indirect target returns 42, the function stores the 16-bit representation of 42 and returns 1. F-7bcec1f9d3df is not called.

2. **Indirect failure sentinel.**  
   If the selector returns zero and the indirect target returns −1, the function returns 0 without its final output assignment. Under the stated no-dependency-write condition, the destination remains unchanged.

3. **Direct-path all-ones result.**  
   If the selector returns nonzero and F-7bcec1f9d3df returns −1, the function stores the all-ones 16-bit representation and returns 1. It does not apply the indirect path’s failure rule.

4. **Negative selector result.**  
   If the selector returns −1, the function selects F-7bcec1f9d3df. The selector value −1 is not itself a failure sentinel for this wrapper.

5. **Comparison before narrowing.**  
   If the selector returns zero and the indirect target returns 65535, the result is not equal to −1 as a 32-bit integer. Under the recovered truncation interpretation, the function stores all ones in the 16-bit destination and returns 1.

6. **Successful zero result.**  
   If either selected value-producing dependency returns zero, the function stores a 16-bit zero and returns 1. A produced zero is not treated as failure.

7. **Selector changes dispatch state.**  
   If the selector returns zero after changing the pointer field at context offset 12, the subsequent dispatch read uses the post-call field value. There is no earlier wrapper-side snapshot.

8. **Dependency modifies output before failure.**  
   If the indirect target modifies the output storage through another reference and then returns −1, the wrapper returns 0 without restoring the previous contents.

## Uncertainties

The following remain **documented_only**:

- The application-level meaning of the context and produced value.
- The selector’s purpose and the conditions that cause it to return zero.
- The semantic identities and ownership of the dispatch structures.
- The indirect target’s identity and full calling convention.
- The complete ABI, including pointer width and alignment requirements.
- Exact out-of-range signed-conversion rules beyond the recovered truncation interpretation.
- Dependency effects on globals, memory, ownership, and external resources.
- Thread safety, synchronization, reentrancy, and concurrent mutation behavior.
- Behavior after invalid memory access, a non-returning dependency, or another abnormal dependency outcome.

A likely intent is to expose a common 16-bit output interface over two alternative producers. That interpretation is not needed to establish the branch behavior and does not justify making the two paths’ error handling equivalent.

## Evidence

The established behavior comes solely from the supplied static function representation:

- The selector call precedes the branch.
- Zero selects the dispatch chain at byte offsets 12 and 32.
- Nonzero selects F-7bcec1f9d3df with the context argument.
- Only the indirect result is compared with −1.
- The indirect sentinel branch returns 0 before the output assignment.
- Both remaining paths share a 16-bit output assignment followed by return of 1.

No dependency bodies, runtime traces, ABI documentation, or execution results were supplied. No runtime tests have been performed.
