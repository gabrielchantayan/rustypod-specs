<!-- Trusted provenance: F-b5915edc3f5a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b5915edc3f5a

## Purpose

F-b5915edc3f5a conditionally initializes a mutable context. A nonzero active-handle field bypasses initialization and returns zero. Otherwise, the function acquires access to a candidate slot, probes a context-associated value, optionally validates and imports header data, and computes derived values before deciding whether to publish the slot’s current value.

Zero does not necessarily mean publication occurred. A size-change path invokes several dependencies, updates size fields, invalidates another context field, and returns zero without explicitly publishing a candidate.

The candidate slot is mutable state, not an acquisition-time snapshot. Its address is supplied to the acquisition dependency. Later header access, cleanup, size-change handling, and publication each read the slot when needed. No supplied dependency contract prevents retention of that address and mutation during subsequent calls.

“Candidate,” “header,” “active handle,” and “release-shaped” describe observed roles. They do not establish a file format, resource type, or ownership model.

## Interface

### Arguments and result

The supplied representation uses four-byte integers and addresses.

| Argument | Established use |
|---|---|
| Context address | Addresses the mutable record described below. |
| Second argument | Four-byte value, neither inspected nor forwarded. |
| Third argument | Four-byte value, neither inspected nor forwarded. |
| Candidate seed | Four-byte value copied into a local slot whose address is passed to F-14f40f321d22. |

The return value is a signed 32-bit integer.

| Result | Condition |
|---|---|
| 0 | Existing active handle, accepted derived values followed by publication, or completion of the size-change path. |
| Acquisition status | Any nonzero result from F-14f40f321d22 is returned unchanged. |
| 10 | Negative probe result. |
| 26 | Rejected header, insufficient matching-size usable size, or rejected derived values. |

An acquisition status can itself equal 10 or 26; the numeric result alone does not identify the path.

No explicit null, alignment, bounds, or accessibility checks are present. Every reached read or write requires suitable storage. Successful acquisition does not establish that a later slot value, candidate address, or header address is valid.

### Context layout

Offsets are byte offsets from the context address. Field names describe use rather than verified domain semantics.

| Offset | Width | Established use |
|---|---:|---|
| 0x00 | 4 bytes | Argument to the probe and, on size change, the final dependency. |
| 0x0C | 4 bytes | Initial active-handle test, failure clearing, and successful publication. |
| 0x11 | 1 byte | Conditionally set to one from a header condition; never explicitly cleared here. |
| 0x12 | 1 byte | First scaling input. |
| 0x13 | 1 byte | Second scaling input. |
| 0x14 | 1 byte | Third scaling input. |
| 0x16 | 1 byte | Boolean derived from the first F-d1e995cfb2d3 result. |
| 0x17 | 1 byte | Boolean derived from the second F-d1e995cfb2d3 result. |
| 0x1C | 2 bytes | Stored size, read as unsigned. |
| 0x1E | 2 bytes | Stored usable size, read as unsigned. |
| 0x20 | 4 bytes | First adjusted result. |
| 0x24 | 4 bytes | Second adjusted result. |
| 0x28 | 4 bytes | Retained usable size minus 35. |
| 0x2C | 4 bytes | Third adjusted result. |
| 0x60 | 4 bytes | Argument to F-94cb53877c15; subsequently zeroed on size change. |

### Candidate slot and header

The local candidate slot occupies four bytes. Its address escapes to F-14f40f321d22. The function does not establish an immutable candidate value after that call.

On a positive probe, the slot’s then-current value is treated as an address. A four-byte read at candidate offset 0x44 supplies a header address. That header address is retained independently of later slot changes.

| Header offset | Established use |
|---|---|
| 0x00 | Base address passed to F-7f14f1ab6851 with length 16. |
| 0x10–0x11 | High-byte-first unsigned size encoding. |
| 0x12 | Value greater than one triggers the context flag write. |
| 0x13 | Selector read separately for two tests. |
| 0x14 | Unsigned one-byte subtraction amount. |
| 0x15–0x17 | Scaling bytes imported on the matching-size path. |
| 0x34 | Address passed to the first F-d1e995cfb2d3 call. |
| 0x40 | Address passed to the second F-d1e995cfb2d3 call. |

A global reference-value slot supplies the second argument to F-7f14f1ab6851. It is read after probing and after loading the header address. Its declaration, contents, and pointed-to storage are not supplied.

Native byte order for context fields is not established. Only the header size’s high-byte-first construction is explicit.

## Behavior

### Existing active handle

The first operation reads context offset 0x0C.

If nonzero, the function returns zero without dependency calls or explicit writes. It does not initialize or expose the candidate slot on this path.

### Acquisition and probe

With an initially zero active-handle field, the function initializes the local slot from the candidate seed and calls F-14f40f321d22 with the context address, one, the slot’s address, and zero.

A nonzero acquisition result is returned immediately. There is no subsequent explicit slot read, release-shaped call, or active-handle reset. Dependency changes are not rolled back.

After a zero acquisition result, the function reads context offset 0x00 and calls F-f3192f39e4f6 with that value. It does not snapshot the candidate slot between acquisition and probing.

The signed probe result selects:

- **Negative:** common cleanup, returning 10.
- **Zero:** derived-value calculation without header processing.
- **Positive:** header processing.

The active-handle field is not rechecked. Acquisition or probing may have changed the context, globals, or candidate slot; no supplied preservation contract excludes such changes.

### Candidate-slot read timing

Each reached candidate use consumes a fresh slot read:

| Use | Read timing |
|---|---|
| Header lookup | After a positive probe, immediately for the candidate-relative read at 0x44. |
| Common cleanup | After the condition selecting failure, when preparing the argument to F-6cc778185718. |
| Size-change handling | After header validation and the stored-size comparison, when preparing the argument to F-6cc778185718. |
| Publication | After all three scaling calls, derived-field writes, and successful final validation. |

There is no requirement that these values equal the seed, the value left by acquisition, or a value previously used for header lookup.

In particular, header validation can use a header obtained through one slot value while cleanup or publication uses a later slot value. A slot change does not cause the retained header address to be reloaded or revalidated.

Once a value has been supplied as a call argument, mutation of the slot during that call does not retroactively change that argument.

### Positive-probe validation

The function reads the slot and loads the header address from offset 0x44 of the resulting candidate address. It then reads the global reference-value slot and calls F-7f14f1ab6851 with the retained header address, the global value, and 16.

A nonzero result selects common cleanup with result 26. The cleanup argument is read from the candidate slot after this call; it need not identify the candidate used for header lookup.

After a zero comparison result, header byte 0x12 is read. If greater than one, context byte 0x11 is set to one. Otherwise, that context byte is not written. This conditional write precedes subsequent validation and is not explicitly rolled back.

Header byte 0x13 is read once for equality with one and separately for a value below two. Under stable contents, selectors zero and one reach size validation; values two through 255 reject.

For stable accepted selectors, the size is:

\[
S=256H_{10}+H_{11},
\]

with unsigned header bytes. The constructed value is zero-extended to 32 bits.

The size must satisfy the unsigned power-of-two-or-zero bit condition and a signed lower bound of 512. The accepted sizes are exactly:

\[
512,\ 1024,\ 2048,\ 4096,\ 8192,\ 16384,\ 32768.
\]

Zero satisfies the bit condition but fails the lower bound.

The function then reads the unsigned subtraction byte at header offset 0x14 and obtains:

\[
U=S-H_{14}.
\]

For these admitted sizes, \(257\le U\le32768\). Both size values fit unsigned 16-bit storage, and this subtraction is safe in signed 32-bit arithmetic.

The stable-selector qualification is material. If the first selector read yields one but the second yields at least two, validation can consult a size value not constructed on that path. No deterministic size or outcome is established for that case.

### Size-change path

The function compares the validated size with the unsigned 16-bit field at context offset 0x1C.

If different, the explicit sequence is:

1. Read the current candidate slot and pass that value to F-6cc778185718.
2. Store the retained usable size at context offset 0x1E.
3. Store the retained size at context offset 0x1C.
4. Read context offset 0x60 and pass its value to F-94cb53877c15.
5. Store zero at context offset 0x60.
6. Read the then-current context field at 0x00 and call F-3452f1df7e41 with that value and the address of context offset 0x1C.
7. Return zero.

Dependency results are ignored. Slot mutation during the first call does not cause another slot read on this path.

This path does not require usable size to be at least 500. It does not import scaling bytes, compute derived values, explicitly publish a candidate, or explicitly clear the active-handle field.

The size stores use values retained before the first dependency call. Later dependencies may change those fields again. Likewise, the explicit zero at offset 0x60 does not guarantee its value after the final dependency returns.

### Matching-size path

If the stored size equals the validated size, usable size must be at least 500. A smaller value selects common cleanup with result 26.

Otherwise, the explicit sequence is:

1. Store size at context offset 0x1C, then usable size at 0x1E.
2. Import header bytes 0x15, 0x16, and 0x17 into context bytes 0x12, 0x13, and 0x14, respectively.
3. Call F-d1e995cfb2d3 with the retained header address plus 0x34.
4. Store its result as a one-byte boolean at context offset 0x16: zero for zero, one otherwise.
5. Call F-d1e995cfb2d3 with the same retained header address plus 0x40.
6. Store the analogous boolean at context offset 0x17.

The header address is not reloaded from the candidate slot between calls. Either dependency may affect later context reads or mutate an accessible candidate slot; no preservation contract is supplied.

### Derived values and final decision

The zero-probe path and completed matching-size path both reach this stage.

The function reads context offset 0x1E once as unsigned 16-bit \(V\), retaining \(B=V-12\). Each scaling input is read separately immediately before its corresponding call.

The product argument is unsigned 32-bit arithmetic:

\[
P_k=\bigl(\operatorname{uint32}(B)\,C_k\bigr)\bmod 2^{32}.
\]

Each F-870c4fcf7295 call receives its product and 255. Its arithmetic meaning is unknown.

Observable order:

1. Read context byte 0x12, make the first call, subtract 23 from its signed result, and store at offset 0x20.
2. Read context byte 0x13 and make the second call.
3. Store retained \(V-35\) at offset 0x28.
4. Store the second result minus 23 at offset 0x24.
5. Read context byte 0x14 and make the third call.
6. Store the third result minus 23 at offset 0x2C.

The same retained usable size drives all products and the offset-0x28 store, even if dependencies change context offset 0x1E. Later scaling bytes can reflect mutations by preceding calls.

Let the first and second adjusted results be \(A_1\) and \(A_2\). Validation rejects when:

\[
A_1<0\quad\text{or}\quad A_1<A_2.
\]

These comparisons use retained results, not reloaded context fields. The third adjusted result has no explicit acceptance constraint.

On rejection, common cleanup returns 26. Otherwise, the function reads the candidate slot at that point, writes its current four-byte value to context offset 0x0C, and returns zero. No nonzero or validity check accompanies publication.

Subtracting 23 from sufficiently negative dependency results can overflow the represented signed type. Exact behavior for such results is unresolved.

### Common cleanup

Cleanup reads the then-current candidate slot and passes that value to F-6cc778185718. After the call, it explicitly clears context offset 0x0C and returns the selected error.

No other field is explicitly restored. Changes to the slot during cleanup do not cause another slot read before return.

## Side effects

Initialization is not transactional. Before failure, the function may have written the flag, sizes, scaling bytes, boolean fields, and all derived fields. It performs no general rollback. Their final values nevertheless depend on undocumented dependency effects.

The local candidate slot’s address is exposed to acquisition. Subsequent dependencies may mutate it if acquisition retained or otherwise exposed that address. Such mutation can redirect cleanup or publication without changing the retained header address.

The function does not establish ownership transfer, reference counting, release semantics, or memory lifetime for candidate values. “Release-shaped” identifies call placement only. It is not evidence that F-6cc778185718 or F-94cb53877c15 frees anything.

No valid use of the local slot’s address after this invocation returns is established.

## Dependencies

| Opaque ID | Established interface use |
|---|---|
| F-14f40f321d22 | Receives context, one, local candidate-slot address, and zero. Nonzero status is propagated. Retention of the slot address is not excluded. |
| F-f3192f39e4f6 | Receives context offset 0x00; signed result selects negative, zero, or positive processing. |
| F-7f14f1ab6851 | Receives retained header address, later-read global value, and 16. Zero permits continuation. |
| F-6cc778185718 | Receives the candidate slot’s value read at cleanup or size-change call preparation. Result ignored. |
| F-94cb53877c15 | Receives context offset 0x60 on size change. Result ignored. |
| F-3452f1df7e41 | Receives the then-current context offset 0x00 value and address of offset 0x1C. Result ignored. |
| F-d1e995cfb2d3 | Receives two retained-header-relative addresses in separate calls. Results become booleans. |
| F-870c4fcf7295 | Receives three unsigned 32-bit products, each paired with 255. Signed results feed adjustments. |

No dependency implementation, preservation guarantee, or ownership contract is supplied.

## Acceptance examples

These are conditional static consequences, not executed tests. Arithmetic examples exclude signed adjustment overflow.

1. **Existing active handle.** A nonzero field at context offset 0x0C returns zero without calls or explicit writes.

2. **Acquisition error.** Acquisition returning 7 causes immediate return of 7, without subsequent cleanup or explicit active-handle clearing.

3. **Mutation during probing.** Suppose acquisition leaves the slot holding candidate A and retains its address. If probing changes the slot to B and returns positive, header lookup uses B’s field at offset 0x44, not A’s.

4. **Different header and cleanup candidates.** Suppose header lookup uses B, then F-7f14f1ab6851 changes the slot to C and returns nonzero. Cleanup receives C, then the active-handle field is cleared, and the function returns 26.

5. **Late publication.** Suppose header validation uses B’s header and later scaling calls leave the slot holding D. If adjusted results pass validation, D is published. The function does not revalidate D’s header or require D to equal B.

6. **Negative probe.** If probing changes the slot to C and returns negative, cleanup receives C, the active-handle field is cleared afterward, and the result is 10.

7. **Late global read.** If acquisition or probing changes the global reference-value slot, comparison receives the value read after header lookup, not an entry-time snapshot.

8. **Flag write before rejection.** With zero comparison result, header byte 0x12 above one, and stable selector two, the function writes the flag before cleanup with result 26. Cleanup may subsequently change it.

9. **Size-change asymmetry.** Size 512 and subtraction byte 255 give usable size 257. A differing stored size selects the size-change path and return zero. A matching stored size instead selects cleanup and return 26.

10. **Final boundary.** Adjusted results 10, 10, and −100 permit publication of the slot’s final value. Results 9 and 10 for the first two calls reject regardless of the third adjusted result.

11. **Unsigned product.** A zero probe, later usable-size read of 5, and first scaling byte one produce first product argument \(2^{32}-7\), not a signed argument of −7.

## Uncertainties

The following remain **documented_only**:

- Dependency behavior, memory effects, ownership, and preservation guarantees.
- Whether acquisition retains the candidate-slot address and which later dependencies can access it.
- Validity and lifetime of each independently read candidate value.
- Whether a published zero value is meaningful.
- Global reference contents and comparison semantics beyond zero permitting continuation.
- Access widths inside header-relative dependencies.
- Scaling dependency arithmetic and return ranges.
- Signed adjustment overflow behavior.
- Inconsistent selector reads that bypass size construction.
- Effects of invalid addresses, aliasing, concurrent mutation, or reentrancy.

No synchronization, coherent snapshot, or atomic publication protocol is established.

## Evidence

Established static evidence comprises access widths and offsets, explicit arguments, candidate-slot address exposure, later slot reads, retained header use, branch conditions, arithmetic conversions, write ordering, and return paths in the supplied representation.

Possible retained-address mutation is an unresolved dependency capability, not an observed event. Domain terminology and release-shaped interpretations are inferred intent, not ownership guarantees.

No dependency implementations were inspected. No runtime tests or dynamic observations were performed.
