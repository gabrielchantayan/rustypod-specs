<!-- Trusted provenance: F-799bc49cafe5; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-799bc49cafe5

## Purpose

F-799bc49cafe5 obtains a 64-bit result from F-870c4fcf7295 and writes a two-word representation to caller-supplied storage. The first output word is the low 32 bits of that result. The second is derived by multiplying its high 32 bits by an externally stored scale value.

The function’s distinguishing behavior is that the scale applies only to the high word. The visible computation does not multiply the complete 64-bit result, propagate a carry between output words, or normalize the resulting pair.

These statements describe the supplied static evidence. The meaning of the dependency’s result, the purpose of the scale, and the units of either output word are not established. In particular, the literal argument 1000 does not by itself establish a time conversion.

## Interface

### Inputs

The function accepts two arguments:

- **Destination:** a pointer to storage containing at least two consecutive 32-bit words.
- **Forwarded value:** one opaque 32-bit value passed to F-870c4fcf7295 without a visible transformation.

The destination’s pointer width is not established by its declaration alone. Its addressed element width is 32 bits, and the second output location is four bytes after the first.

No visible input specifies an output length, a scale value, an allocation policy, or a status destination.

### Output layout

On the ordinary completion path, the function writes:

| Byte offset from destination | Width | Content |
|---|---:|---|
| 0 | 32 bits | Low 32 bits of the dependency result |
| 4 | 32 bits | Product derived from the dependency result’s high 32 bits and the externally stored scale |

These are two adjacent word stores, not an explicitly shown single 64-bit store. The order of the words is established independently of byte endianness: the low word occupies the lower-addressed slot.

Byte ordering within each word is not established by the supplied material. Consumers must not infer a portable eight-byte serialization format from this layout.

### Return and ownership

The function has no declared return value. It exposes no separate success indicator.

There is no visible allocation, deallocation, retention of the destination pointer, or transfer of ownership by this function. The destination remains caller-supplied storage. That observation does not establish what F-870c4fcf7295 may allocate, release, retain, or modify internally.

### Preconditions

For the ordinary two-write path to complete:

- F-870c4fcf7295 must return normally.
- The destination must support both 32-bit writes at offsets 0 and 4.
- Any alignment requirements of the execution environment must be satisfied.
- The external scale must be readable when the second output expression is evaluated.

These are access requirements, not checks performed by the function. No visible null, bounds, alignment, or accessibility validation occurs.

## Behavior

### Dependency invocation

Before either visible destination write, the function invokes F-870c4fcf7295 exactly once.

The arguments, in order, are:

1. The supplied opaque 32-bit value.
2. The integer constant 1000.

The dependency yields a 64-bit result. Its bit pattern supplies both output calculations. There is no second call to obtain the high word and no visible retry.

The function does not inspect a status field, test a sentinel, or distinguish successful results from error-encoded results. If the dependency returns an error indication within its 64-bit result, the visible function still processes those bits through the same output operations.

### First output word

After the dependency returns, the function converts its result to a 32-bit integer and stores that value at destination offset 0.

At the bit-layout level, this selects the result’s low 32 bits. The high 32 bits do not contribute to this first output word.

The first store is a separate operation preceding the second store. No conditional branch suppresses it based on the returned value.

### Second output word

The function interprets the returned 64-bit bit pattern as unsigned for a right shift of 32 bit positions. This extracts the high 32 bits without sign-extending the original 64-bit value.

The extracted value is then converted to a 32-bit integer and multiplied by the external scale. The resulting value is assigned to the word at destination offset 4.

This distinction matters:

- The high-word extraction is a logical, unsigned shift.
- The extracted word is subsequently represented as an integer for multiplication.
- The first output word does not participate in that multiplication.
- No carry, borrow, saturation, rounding, or normalization operation is visible.

The supplied representation uses a 32-bit integer conversion, but does not supply the external scale’s declaration or the original language-level arithmetic contract. For representable products, the second output is directly established. For products outside the relevant arithmetic range, precise overflow semantics remain documented_only.

A likely fixed-width implementation would retain the low 32 bits of a product, but the supplied evidence is insufficient to elevate that expectation into an unconditional overflow contract.

### Observable ordering

The visible sequence is:

1. Invoke F-870c4fcf7295.
2. Receive its 64-bit result.
3. Write the low result word at destination offset 0.
4. Read the external scale for the high-word multiplication.
5. Write the derived second word at destination offset 4.
6. Return without a declared result.

This is the ordering represented by the supplied operations. It is not a claim about processor-level memory barriers, volatile accesses, or inter-thread synchronization.

The scale is not visibly captured before the dependency call. Consequently, the scale used by this function cannot be assumed to be its value at function entry. A dependency side effect may affect the value subsequently read.

The first destination store also precedes the scale read. If the destination legally aliases the scale’s storage, that store may change the value used in the multiplication. No non-aliasing restriction is supplied.

### Completion and failure

No local error handling or rollback is visible.

If execution does not progress past the dependency call, neither explicit destination store has yet occurred. That does not exclude writes performed internally by the dependency.

If execution stops after the first store but before the second completes, the destination may contain a newly written first word and an unchanged second word. The function provides no transactional guarantee for the pair.

The precise behavior of invalid memory accesses, arithmetic overflow, or abnormal dependency termination is not established. Such conditions must not be described as returning a particular status, zeroing the output, or preserving both original destination words.

## Side effects

The function’s explicit ordinary-path effects are:

- One invocation of F-870c4fcf7295.
- One 32-bit write at the destination.
- One read of an external scale value.
- One 32-bit write four bytes after the destination.

No additional explicit destination writes are shown.

The dependency’s effects are unknown. It must not be assumed to preserve global state, avoid I/O, avoid allocation, or leave unrelated memory unchanged.

There is no visible locking, atomic update of the output pair, memory barrier, or synchronization protocol. A concurrent observer is not promised a coherent snapshot of both output words. Concurrent modification of the scale is likewise not given defined synchronization semantics by this function.

## Dependencies

### F-870c4fcf7295

**Established use:** called once with the forwarded 32-bit value and the constant 1000; supplies the 64-bit result processed by this function.

**Not established:** argument meaning, units, valid value ranges, failure encoding, global side effects, memory ownership behavior, reentrancy, or normal-return guarantees.

The dependency receives no explicit destination argument from this function. This does not prove that it cannot reach the destination through other state or aliases.

### External scale state

**Established use:** read for the multiplication that forms the second output word, after the first destination store.

**Not established:** initialization, mutability, exact declaration, signedness, permitted range, synchronization, or semantic units.

This state is a data dependency rather than an additional function call. No initializer or setter is included in the supplied evidence.

## Acceptance examples

The examples below are conditional static acceptance cases, not executed tests. Unless otherwise stated, assume normal dependency return, valid non-overlapping storage, a readable stable scale, and arithmetic values whose products are representable.

1. **Independent low and high words**

   If F-870c4fcf7295 returns `0x0000000300000005` and the scale read is 7, destination offset 0 receives 5 and offset 4 receives 21.

   This distinguishes the function from one that scales the complete 64-bit value.

2. **No carry from the low word**

   If the result is `0x00000001FFFFFFFF` and the scale is 2, the first word is `0xFFFFFFFF` and the second is 2.

   The low word is neither multiplied nor used to generate a carry into the second word.

3. **Zero high word**

   If the result is `0x0000000012345678` and the scale is 9, the first word is `0x12345678` and the second is zero.

4. **Zero scale**

   If the result is `0x0000000500000009` and the scale is zero, the first word is 9 and the second is zero. Zeroing the second word does not clear the first.

5. **Dependency changes the scale**

   If the scale is initially 2, the dependency changes it to 7 before returning `0x0000000300000005`, and nothing changes it afterward, the outputs are 5 and 21.

   Using the entry-time scale to produce 6 would not match the visible ordering.

6. **First output aliases the scale**

   If legal aliasing makes the first destination word the scale storage, and the dependency returns `0x0000000300000005`, the first store replaces that storage with 5 before the scale read. Provided the scale’s actual representation interprets the replacement as 5, the second output is 15.

7. **Overflow boundary remains conditional**

   If the extracted high word is `0x40000000` and the scale is 4, the mathematical product is \(2^{32}\). A truncating 32-bit implementation would write zero to the second slot. That output is not an unconditional acceptance requirement without additional arithmetic evidence.

## Uncertainties

The following remain **documented_only**:

- The semantic identity of the 32-bit forwarded value.
- The meaning and units of the dependency’s 64-bit result.
- The significance of the constant 1000.
- The external scale’s declaration, initialization, signedness, and valid range.
- Exact multiplication-overflow and out-of-range integer-conversion behavior.
- Pointer width, byte endianness, and required alignment.
- Whether destination overlap with external state is permitted by the original execution contract.
- Dependency effects, error conventions, and ownership behavior.
- Behavior under concurrent access or abnormal termination.

An inferred intent is to produce a two-component representation with only the second component scaled. No stronger interpretation, including a particular time format or physical unit, is justified.

## Evidence

The established contract comes solely from the supplied static function representation: a two-argument function with no declared return value, one dependency call returning 64 bits, a low-word store, an unsigned 32-bit shift followed by an integer conversion, an external-scale multiplication, and a second adjacent word store.

No dependency body, external-state declaration, caller contract, platform arithmetic specification, or runtime observation was supplied. No runtime tests have been performed. The acceptance examples express consequences and explicit conditions of the static evidence; they are not reports of observed execution.
