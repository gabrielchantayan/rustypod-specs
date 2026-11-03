<!-- Trusted provenance: F-b7db1b4e13da; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b7db1b4e13da

## Purpose

F-b7db1b4e13da conditionally processes a record associated with a mutable receiver. It first asks a dependency whether processing should proceed. If that dependency returns zero, this function returns zero immediately. Otherwise, it invokes a preparation dependency and an external operation, clears one bit in a receiver field, and conditionally sets another bit and invokes a final dependency.

The established behavior is the ordering of calls, memory accesses, bit changes, and return-value selection. The external operation receives arguments consistent with a 36-byte record transfer, but its implementation is not supplied. Actual copying, overlap handling, and ownership behavior are therefore not established.

A zero return is not, by itself, evidence that the receiver remained unchanged or that processing was rejected. Both early rejection and an accepted path without the final call return zero. The final dependency can also return zero.

## Interface

The function takes three arguments and returns a 32-bit value.

| Argument | Role established by the supplied source |
|---|---|
| Receiver address | Base address used for a 32-bit read at offset 0x38, an external-operation argument at offset 0x08, and 16-bit accesses at offset 0x24. |
| Unused 32-bit argument | Not read, forwarded, or otherwise used directly by this function. |
| Descriptor address | Address of two consecutive 32-bit words. Both are read for the initial dependency call. The first is read again if that call returns nonzero. |

The descriptor layout is:

| Offset | Width | Use |
|---|---:|---|
| 0x00 | 32 bits | First argument to F-cbcf1db3a55e; reread after acceptance and subsequently used as a source address. |
| 0x04 | 32 bits | Second argument to F-cbcf1db3a55e; not directly read again. |

Relevant receiver locations are:

| Offset | Width or extent | Use |
|---|---:|---|
| 0x08 | Address | First argument to the external operation; later passed to F-69bd2f35c00e if the conditional branch is taken. |
| 0x24 | 16 bits | Read and written as a flag field after the external operation returns. |
| 0x38 | 32 bits | Third argument to F-cbcf1db3a55e. |

After acceptance, the first descriptor word supplies an address whose offset 0x1C may be read as a 16-bit flag field.

The source representation uses 32-bit integer-typed address values. A 32-bit address model is consistent with this representation, but the complete ABI and address-space rules are not provided. The contract requires valid effective addresses for all accesses actually reached; it does not promise useful behavior for overflowing address calculations, inaccessible storage, or invalid alignment.

The return value is treated here as a raw 32-bit result. No status enumeration or error-code interpretation is established.

## Behavior

### 1. Initial dependency call

The function obtains:

- The descriptor’s first 32-bit word.
- The descriptor’s second 32-bit word.
- The receiver’s 32-bit word at offset 0x38.

It passes those values, in that argument order, to F-cbcf1db3a55e.

All three values are obtained before that dependency begins executing. Their relative read order is not established by the supplied expression and must not be relied upon.

If F-cbcf1db3a55e returns zero:

- This function returns the 32-bit value zero.
- No subsequent dependency is called.
- No direct receiver flag read or write is performed by this function.
- The descriptor’s first word is not reread by this function.

This is only a statement about direct behavior after the dependency returns. F-cbcf1db3a55e may itself have changed memory or other state.

Any nonzero result proceeds to the next stage. No distinction between positive and negative nonzero values is made.

### 2. Capture the source address

After acceptance, the function rereads the descriptor’s first 32-bit word and retains that value as the source address for subsequent operations.

This is a fresh read, not reuse of the value passed to F-cbcf1db3a55e. If that dependency changes the descriptor’s first word, the changed value is used.

The descriptor’s second word and the receiver’s word at offset 0x38 are not directly reread.

### 3. Invoke preparation and external operation

The function establishes a default return value of zero and calls F-29b5e86301ff with no explicit arguments.

After that dependency returns, it invokes the external operation with these arguments:

1. Receiver address plus 0x08.
2. The source address captured before F-29b5e86301ff was called.
3. The numeric value 0x24, equivalent to 36 decimal.

The external operation’s return value is not used.

Changes made by F-29b5e86301ff to the descriptor’s first word do not cause a new source address to be selected. Changes it makes to memory at the captured source address, the receiver, or other relevant state may still affect subsequent behavior.

The external operation’s name, implementation, and opaque function ID are not supplied. Its argument arrangement suggests a destination, source, and byte count. That interpretation is inferred intent, not an established copying guarantee.

### 4. Read the receiver flags and clear bit 0x0040

After the external operation returns, the function reads the receiver’s 16-bit field at offset 0x24.

Let the value read at this point be the receiver flags. The function computes a 16-bit value equal to those flags with bit 0x0040 cleared, then writes that value to the same field.

This write occurs on every accepted path that reaches this stage, even if bit 0x0040 was already clear.

No other bit is changed by this particular write. In particular, it does not clear an existing 0x0100 bit.

The receiver flags used for the next condition are the value read before this clearing write, not a fresh read of the receiver field.

### 5. Test receiver eligibility and, conditionally, source flags

The first condition is whether either bit 0x0002 or bit 0x0010 was set in the receiver flags read in the preceding stage. Both bits need not be set.

If neither bit was set:

- No source flag read is performed by this function.
- No second receiver flag write is performed.
- F-69bd2f35c00e is not called.
- The function returns zero.

If either bit was set, the function reads a 16-bit field at offset 0x1C from the captured source address. This read occurs after the receiver’s bit-0x0040 clearing write.

If source bit 0x0080 is set:

- No second receiver flag write is performed.
- F-69bd2f35c00e is not called.
- The function returns zero.

If source bit 0x0080 is clear, processing continues to the final stage.

The timing matters when memory aliases. If the receiver flag write overlaps the source flag field, the source flag read observes memory after that write, subject to the platform’s ordinary memory semantics.

### 6. Set bit 0x0100 and invoke the final dependency

When the receiver eligibility condition holds and source bit 0x0080 is clear, the function writes the receiver flag field again.

The written value is based on the receiver flags captured in stage 4:

- Bit 0x0040 is cleared.
- Bit 0x0100 is set.
- All other bits are retained.

This is not a read-modify-write based on a fresh receiver flag read.

The function then calls F-69bd2f35c00e with the receiver address plus 0x08. Its 32-bit return value becomes this function’s return value.

There is no direct receiver write after that call. If F-69bd2f35c00e changes the receiver or other state, this function does not restore the earlier field value.

## Side effects

### Direct memory accesses

The function directly reads:

- Two 32-bit descriptor words before the initial call.
- One 32-bit receiver word before the initial call.
- The descriptor’s first 32-bit word again after acceptance.
- One 16-bit receiver flag field after the external operation.
- One 16-bit source flag field only when the receiver eligibility condition holds.

Its only explicit direct writes are to the receiver’s 16-bit field at offset 0x24:

1. An unconditional accepted-path write clearing bit 0x0040.
2. A conditional second write additionally setting bit 0x0100.

Dependencies may perform additional reads and writes. Their effects cannot be bounded from the supplied body.

### Ownership and lifetime

No direct allocation, deallocation, ownership transfer, or reference-count adjustment is visible in this function.

That does not establish that the dependencies avoid such operations. In particular, the body does not prove that the captured source address remains valid after either preceding dependency call. Validity across those calls is a precondition for any later access that reaches it.

No non-aliasing requirement is explicitly checked. The receiver, descriptor, and source may affect one another if their storage overlaps. Whether the external operation supports overlapping ranges is unknown.

### Errors and interruption

The function performs no explicit null checks, bounds checks, alignment checks, or recovery around dependency calls.

Zero from F-cbcf1db3a55e selects the early-return path; it is not established to mean a particular error.

If an access faults or a dependency fails to return normally, the supplied body provides no rollback mechanism. Earlier dependency effects and completed receiver writes must not be assumed undone.

## Dependencies

| Opaque ID | Invocation and established use | Unknown behavior |
|---|---|---|
| F-cbcf1db3a55e | Receives the two initial descriptor words and the receiver word at offset 0x38. Zero stops processing; nonzero permits it. | Validation meaning, memory effects, global effects, ownership behavior, and failure modes. |
| F-29b5e86301ff | Called without explicit arguments after the source address is captured and before the external operation. Its return is unused. | Purpose, synchronization behavior, memory effects, global effects, and ownership behavior. |
| F-69bd2f35c00e | Conditionally receives receiver plus 0x08. Its 32-bit result is returned directly. | Meaning of the result, receiver mutations, global effects, ownership behavior, and failure modes. |

A fourth, external dependency has no opaque ID in the supplied map. It receives receiver plus 0x08, the captured source address, and 0x24. Its result is unused. Its identity and semantics are **documented_only** rather than replaced with an assumed memory-copy contract.

No dependency is assumed to preserve globals, preserve relevant pointed-to storage, or retain or release ownership.

## Acceptance examples

These examples are conditional behavioral checks derived from static evidence. They are not reports of executed tests. Except where stated otherwise, assume valid accessed storage, normal dependency returns, and no dependency mutations beyond those specified.

### Initial rejection

Given F-cbcf1db3a55e returns zero:

- The result is zero.
- None of the later three dependencies is called.
- This function performs no direct receiver flag write.

Any changes made inside F-cbcf1db3a55e remain outside that last guarantee.

### Source address changed during the initial call

Given the initial descriptor word identifies address A, and F-cbcf1db3a55e changes that word to address B before returning nonzero:

- The initial dependency received A.
- The external operation receives B as its second argument.
- Any conditional source flag read uses B plus 0x1C.

If F-29b5e86301ff later changes the descriptor word to C, the captured address remains B.

### Receiver has no eligibility bits

Given the receiver flag read after the external operation yields 0x0040:

- The first direct flag write stores 0x0000.
- No source flag read occurs.
- F-69bd2f35c00e is not called.
- The result is zero.

### Eligible receiver, blocked source

Given the receiver flag read yields 0x0052 and the subsequent source flag read has bit 0x0080 set:

- The first flag write stores 0x0012.
- There is no second direct flag write.
- F-69bd2f35c00e is not called.
- The result is zero.

### Eligible receiver, permitted source

Given the receiver flag read yields 0x0042 and source bit 0x0080 is clear:

- The first flag write stores 0x0002.
- The second flag write stores 0x0102.
- F-69bd2f35c00e receives receiver plus 0x08.
- If that dependency returns 0x89ABCDEF, this function returns exactly that 32-bit value.

The receiver’s final contents depend on any changes made by the final dependency.

### Existing bit 0x0100 survives a skipped final call

Given the receiver flag read yields 0x0140:

- The first flag write stores 0x0100.
- No source flag read occurs because neither eligibility bit is set.
- The function returns zero.

Thus, skipping the final call does not imply that bit 0x0100 is clear.

### Conditional copy interpretation

If the external operation is independently established to copy exactly 36 bytes from its second argument to its first, without additional mutations:

- Its destination range is receiver offsets 0x08 through 0x2B inclusive.
- The receiver flag field at offset 0x24 receives bytes corresponding to source offsets 0x1C and 0x1D.
- The receiver word at offset 0x38 lies outside that destination range.

These consequences are conditional on that independently established external-operation contract.

## Uncertainties

- **documented_only — External operation:** Copy semantics, range validity requirements, overlap handling, and side effects are unavailable.
- **documented_only — Flag meanings:** Only masks and branch effects are established. Semantic names such as “enabled,” “dirty,” or “owned” would be guesses.
- **documented_only — Dependency effects:** No preservation, allocation, release, synchronization, or global-state guarantees are supplied.
- **documented_only — ABI details:** Endianness, alignment requirements, pointer representation, and address-overflow behavior are not independently established.
- **documented_only — Concurrency:** No atomicity or synchronization guarantee is established by this body.
- **Inferred intent:** The accepted path appears to prepare a receiver record, perform a transfer-like operation, normalize flags, and optionally invoke further processing. That interpretation does not strengthen the operational guarantees above.

## Evidence

The supplied static function body establishes the call order, zero/nonzero gate, fresh descriptor read, argument values, 16-bit masks, conditional source access, direct writes, and return-value selection.

The supplied dependency map establishes the three opaque dependency IDs listed above. It does not identify the external operation or provide any callee implementation.

No runtime tests have been performed. The acceptance examples are static consequences under explicit conditions, not observed executions.
