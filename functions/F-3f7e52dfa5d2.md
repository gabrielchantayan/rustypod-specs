<!-- Trusted provenance: F-3f7e52dfa5d2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3f7e52dfa5d2

## Purpose

F-3f7e52dfa5d2 acquires an object, invokes an indirect operation with a mutable three-word argument block, calls F-3a7a47316439 with a position in the original buffer and a 16-bit value, and then directly updates that buffer.

The direct buffer tail performs **five count-byte updates and four payload-byte stores**. The payload consists of the low byte of the block’s second word followed by bits 31–24, 23–16, and 15–8 of its first word. The fifth count update has no accompanying payload store.

A count-prefixed encoding operation is plausible intent, not established semantics. The visible operations do not establish a message format, the amount of data written by dependencies, or consistency between the final count and initialized buffer contents.

## Interface

### Arguments

The supplied representation uses 32-bit pointer values and 32-bit argument words.

| Argument | Established use |
|---|---|
| First argument | A 32-bit word with no direct use in the visible body. It is not explicitly forwarded to any dependency. |
| Buffer argument | The original byte-buffer address. Its pointer representation initializes the first word of the mutable block. All direct buffer accesses continue to use this original address. |
| Third argument | A 32-bit word initializing the second word of the mutable block. Selected portions are consumed after the indirect call. |
| Fourth argument | A 32-bit word initializing the third word of the mutable block. The routine has no subsequent direct read of this word, but exposes it to the indirect operation. |

The byte at buffer offset zero is read as an unsigned 8-bit count. It is also the destination of every direct count update.

### Mutable argument block

Before acquiring the object, the routine prepares a local, contiguous three-word block:

| Byte offset | Width | Initial contents |
|---|---:|---|
| 0 | 32 bits | Original buffer pointer representation |
| 4 | 32 bits | Third argument |
| 8 | 32 bits | Fourth argument |

The indirect operation receives the block’s address. Later field uses consume the contents remaining at their respective read points; the initial values are not guaranteed to survive dependency execution.

In particular, the first word can cease to contain the original buffer address. Such a change does not redirect this routine’s direct buffer accesses. That word is subsequently used as data for three payload bytes and the return value, not dereferenced by those operations.

### Result

On normal completion, the result is a 64-bit bit pattern:

- Bits 63–32 contain the first block word as read after the final count update.
- Bits 31–0 are zero.

Construction zero-extends the 32-bit word before shifting it left by 32 bits. It does not sign-extend the word or return it in the low half.

No dependency return value, explicit status, or final count is incorporated into this result.

### Preconditions for normal completion

The visible operations require:

- Valid original-buffer storage for every direct access actually selected.
- A non-null acquired object with a readable dispatch-table pointer and usable callable entry.
- An indirect target compatible with the supplied block address and object pointer.
- Dependencies that return normally on paths expected to continue.
- Storage used by later operations to remain valid despite dependency effects.

No buffer capacity argument, null-buffer check, capacity check, or count-range validation is visible.

For the routine’s own buffer accesses, a readable and writable region covering offsets 0 through 256 is sufficient, provided address arithmetic is valid and storage remains live. This 257-byte bound does **not** bound dependency accesses or guarantee successful completion.

## Behavior

### Preparation and object acquisition

The three block words are initialized before F-a4a34621f0a7 is called without explicit arguments.

If acquisition produces a null pointer, the routine calls F-a817d8d58c10. The supplied static representation marks that call as non-returning. Under that annotation, this branch does not reach indirect dispatch, F-3a7a47316439, the direct buffer tail, or a normal return.

The annotation does not establish how control leaves the routine. A particular exception, diagnostic, termination mechanism, or cleanup policy must not be assumed.

### Indirect object operation

For a non-null acquired object, the routine:

- Reads the object’s initial 32-bit word as a dispatch-table address.
- Reads the callable entry at byte offset 36 from that address.
- Invokes the selected target with the block address first and the object pointer second.

The offset is **36 bytes**, not entry number 36.

The indirect result is ignored. The operation may nevertheless affect subsequent behavior by changing the block, the original buffer, or other reachable state. No preservation of those locations or of global state is established.

### Buffer-position dependency

After the indirect operation returns, the routine calls F-3a7a47316439 with:

| Supplied value | Meaning established by the caller |
|---|---|
| Destination pointer | Original buffer base plus its current unsigned count byte plus one |
| 16-bit value | Bits 31–16 of the block’s second word at argument evaluation |

The destination calculation does not truncate the count-plus-one sum to eight bits. A count of zero selects offset 1; a count of 255 selects offset 256.

The relative evaluation order of the destination argument and the 16-bit argument is not established. Both are evaluated before the dependency executes.

The dependency’s result is ignored. The visible caller does not establish its write size, byte order, count handling, failure reporting, or resource behavior. In particular, the later addition of two to the count is not proof that this dependency writes exactly two bytes.

The direct tail rereads the count after this dependency returns. It does not reuse the count that selected the dependency’s destination.

### Direct buffer tail

After F-3a7a47316439 returns, the routine performs the following ordered count and payload effects:

| Stage | Count effect | Following payload effect |
|---|---|---|
| First | Read the count; write that value plus two, reduced modulo 256 | Store the second block word’s bits 7–0 at the wrapped updated count plus one |
| Second | Read the count again; write that value plus one, reduced modulo 256 | Store the first block word’s bits 31–24 at the wrapped updated count plus one |
| Third | Read the count again; write that value plus one, reduced modulo 256 | Store the first block word’s bits 23–16 at the wrapped updated count plus one |
| Fourth | Read the count again; write that value plus one, reduced modulo 256 | Store the first block word’s bits 15–8 at the wrapped updated count plus one |
| Fifth | Read the count again; write that value plus one, reduced modulo 256 | None |

Each of the first four count writes precedes its associated payload store. The fifth count write occurs after all four payload stores.

For each payload destination, the wrapped updated count is derived from that stage’s captured count read. Selecting the destination does not require another read of the count byte. The final addition of one produces an offset from 1 through 256 and is not reduced modulo 256.

The extracted payload values are numeric bit ranges. They are not a copy of four consecutive bytes from the first word: the first payload byte comes from the second word, and the first word contributes three bytes, excluding its low eight bits.

Let \(c\) be the count immediately after F-3a7a47316439 returns. Assuming valid storage, no aliasing between the buffer and the local block or other active control storage, and no intervening external mutation, the four payload offsets are:

\[
\begin{aligned}
o_1 &= ((c+2)\bmod256)+1,\\
o_2 &= ((c+3)\bmod256)+1,\\
o_3 &= ((c+4)\bmod256)+1,\\
o_4 &= ((c+5)\bmod256)+1.
\end{aligned}
\]

The final count is:

\[
(c+6)\bmod256.
\]

These formulas describe only the direct tail. They do not imply that the complete call increases the entry count by six: dependencies can alter the count before the tail begins.

### Return construction

After the fifth count write, the routine reads the first block word and constructs the 64-bit result.

The payload-byte reads and the final whole-word read are separate uses. Their values need not correspond to one immutable snapshot if storage is independently changed or aliases other writes. Under ordinary non-overlapping storage with no intervening mutation, they all reflect the same retained first-word value.

## Side effects

### Direct effects

The routine directly:

- Initializes the three-word local block.
- Reads the acquired object’s dispatch pointer and selected callable entry.
- Reads the original buffer’s count for F-3a7a47316439’s destination.
- Performs five further count reads and five count writes in the direct tail.
- Performs four payload-byte stores.
- Reads selected portions of the block after dependency execution.
- Reads the first block word for the return value.

Every direct payload offset lies between 1 and 256. Consequently, these payload stores do not directly overwrite the count at offset zero.

The direct tail is not an atomic append. Its first four count/payload pairs are separate writes, and the final count update is another separate write. Interrupted execution can therefore leave a count update visible without its associated payload store.

No rollback is visible.

### Dependency effects

F-a4a34621f0a7, F-a817d8d58c10, F-3a7a47316439, and the indirect dependency may have additional effects not established by this body.

The indirect dependency receives access to the mutable block, which initially contains the original buffer pointer. F-3a7a47316439 receives a pointer into the original buffer. Neither visible invocation supplies a buffer capacity.

No dependency is assumed to preserve global state, preserve the block, retain or release resources, or restrict writes to an inferred encoding extent.

### Ownership and lifetime

No explicit allocation, release, reference-count adjustment, or ownership transfer appears in the routine’s own operations.

The acquired object is not explicitly released here. This alone does not establish a leak; its ownership model and dependency behavior are unknown.

The block is local storage exposed to the indirect dependency. The routine supplies no post-return lifetime guarantee for its address and does not establish permission to retain it.

### Errors and partial completion

No recoverable error result is constructed. Dependency failures and invalid memory accesses have no established local recovery path.

If execution stops after earlier effects, those effects need not be undone. The exact behavior for invalid buffer addresses, invalid dispatch data, insufficient storage, or a dependency that does not return remains unspecified by the visible body.

## Dependencies

| Opaque ID or selection | Established invocation | Unknown contract |
|---|---|---|
| F-a4a34621f0a7 | Called without explicit arguments; supplies the object pointer checked for null | Object identity, allocation, ownership, synchronization, and other effects |
| F-a817d8d58c10 | Called on null acquisition; annotated non-returning | Control-transfer mechanism, diagnostics, cleanup, and externally reported error |
| F-3a7a47316439 | Receives original-buffer current-count-plus-one address and a 16-bit value from the second block word | Encoding, write extent, count changes, failures, ownership, and other effects |
| Indirect dependency; no opaque ID supplied | Selected through byte offset 36 of the table addressed by the object’s first word; receives block address and object pointer | Full input/output contract, mutation extent, resource handling, and failure behavior |

The indirect dependency cannot be assigned a concrete function identity from the supplied evidence.

## Acceptance examples

These are conditional expectations derived from static evidence, not execution reports. Unless otherwise stated, assume valid non-overlapping storage, normal dependency returns, and no independent mutation during the direct tail.

### Ordinary direct tail

Suppose, when F-3a7a47316439 returns:

- The first block word is `0x12345678`.
- The second block word is `0xABCD00EF`.
- The count is 10.
- The block words remain unchanged through return.

The direct effects must be:

| Count written | Payload effect |
|---:|---|
| 12 | Write `0xEF` at offset 13 |
| 13 | Write `0x12` at offset 14 |
| 14 | Write `0x34` at offset 15 |
| 15 | Write `0x56` at offset 16 |
| 16 | No further payload store |

The result must be `0x1234567800000000`.

If the second word already had the stated value when F-3a7a47316439’s arguments were evaluated, that dependency must receive the 16-bit value `0xABCD`. Its argument is not determined by a later snapshot alone.

### Wraparound from 254

Suppose the count is 254 immediately after F-3a7a47316439 returns.

The four count/payload pairs must write counts 0, 1, 2, and 3, with payload stores at offsets 1, 2, 3, and 4 respectively. The fifth count update must leave the count equal to 4.

No rejection or preservation of overwritten payload bytes is visible.

### Offset 256 remains reachable

If the count is 255 when the destination argument for F-3a7a47316439 is evaluated, its destination must be the original buffer base plus 256, not the base itself.

Separately, if the direct tail begins with count 253, its first count update writes 255 and its first payload store targets offset 256. Its later payload offsets are 1, 2, and 3, and its final count is 3.

These cases distinguish byte-width count wraparound from the wider destination-offset addition.

### The dependency changes the count

Suppose the count used for F-3a7a47316439’s destination is 7, but the dependency leaves it equal to 20.

The dependency receives destination offset 8. The direct tail then uses payload offsets 23, 24, 25, and 26, and finishes with count 26.

The destination’s earlier count does not control the tail.

### The first block word changes

Suppose the indirect operation replaces the first block word with `0x89ABCDEF`, and it remains unchanged thereafter.

The routine must still use the original buffer address for direct buffer accesses. Its three payload bytes sourced from the first word must be `0x89`, `0xAB`, and `0xCD`, in that order. Its result must be `0x89ABCDEF00000000`.

The replacement word is not dereferenced by those later operations.

### Null acquired object

If F-a4a34621f0a7 returns null, F-a817d8d58c10 must be invoked.

Under the supplied non-returning annotation, no later indirect call, buffer-position dependency, direct buffer-tail write, or normal return is reached. The local block has already been initialized, and acquisition may already have had effects.

## Uncertainties

The following remain **documented_only**:

- Semantic meanings of the four arguments, block fields, count, and payload bytes.
- The full contract and concrete identity of the indirect dependency.
- Whether any dependency allocates, releases, retains, or transfers resources.
- F-3a7a47316439’s encoding, byte order, write extent, and treatment of the count.
- Caller-provided buffer capacity and any external validation.
- Synchronization, reentrancy, and behavior under concurrent mutation.
- Exact diagnostics, failure mechanisms, and cleanup behavior.
- Behavior if the annotated non-returning dependency unexpectedly returns.
- ABI details beyond the visible widths, argument uses, pointer operations, and result construction.

The count-prefixed encoding interpretation remains inferred intent. It does not strengthen any dependency contract or justify assuming a complete, valid encoded record.

## Evidence

This contract is based solely on the supplied static function representation and opaque dependency mapping. Prior prose and reviewer feedback are not independent behavioral evidence.

The visible operations establish the block initialization, null branch, dispatch-table byte offset, dependency arguments, five direct count updates, four payload stores, extracted bit ranges, and 64-bit return construction. The non-returning property is supplied as an annotation rather than demonstrated by a dependency body.

No dependency bodies, caller contracts, allocation bounds, or runtime observations were provided. No runtime tests have been performed.
