<!-- Trusted provenance: F-11effff0926d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-11effff0926d

## Purpose

F-11effff0926d conditionally reconciles a caller-supplied two-word quantity with a signed product derived from a context object and a requested 32-bit value. It then stores the requested value in the context and invokes a final dependency.

The conditional reconciliation path is enabled only when a context byte exceeds three and a word reached through a context-held pointer is nonzero. On this path, one dependency may replace the two-word quantity. The function compares that resulting quantity with the product and, if they differ, selects one of two further dependencies according to their signed ordering.

A nonzero result from either reconciliation stage is returned immediately. Otherwise, the function performs the context update and final call, then returns zero.

This is the established behavior of the supplied static source. The domain meaning of the context, quantities, and reconciliation operations is unknown.

## Interface

The function accepts four arguments and returns a 32-bit integer:

| Argument | Representation | Role established by the source |
|---|---|---|
| Context address | Address represented by a 32-bit argument | Base address for fixed-offset field accesses and argument to dependencies. |
| Requested value | Signed 32-bit integer | Multiplier, argument to the first dependency, and value eventually stored in the context. |
| Input low word | Unsigned 32-bit integer | Initializes the low word of a mutable 64-bit quantity and is also passed separately to the first dependency. |
| Input high word | Signed 32-bit integer | Initializes the high word of that quantity and is also passed separately to the first dependency. |

The high and low words form a 64-bit bit pattern. Its signed interpretation is:

**quantity = signed high word × 2³² + unsigned low word.**

The initial quantity is copied into local storage before the conditional processing. The first dependency receives the address of that local storage, allowing it to alter the quantity used by the subsequent comparison. The separately passed input words remain the original argument values.

### Context layout used here

Offsets are byte offsets from the context address.

| Offset | Access width | Use |
|---|---:|---|
| `0x0e` | 8 bits, unsigned | Enables further checking when its value is greater than three. |
| `0x24` | 32 bits | Receives the requested value on the completion path. |
| `0x40` | 32 bits, signed | Supplies the other operand of the signed product. |
| `0x6c` | 32-bit pointer representation | Supplies a pointer used for the gate and as a dependency argument. |

The pointer obtained from offset `0x6c` is dereferenced to read its first 32-bit word when the unsigned byte gate passes. Only whether this word is zero matters.

No bounds, null-pointer, alignment, or object-lifetime checks are present. The caller must supply memory valid for every access actually reached. In particular, a byte greater than three requires the pointer field and its pointed-to word to be readable before any dependency can report an ordinary error.

The target byte order and exact calling convention are not established by this fragment. The two-word composition and explicit high/low roles are established.

## Behavior

### Conditional entry

The function first initializes its local two-word quantity from the input words.

It then reads the unsigned byte at context offset `0x0e`.

- Values zero through three bypass all reconciliation dependencies.
- Values four through 255 cause the function to read the pointer at offset `0x6c` and the first 32-bit word at that pointer.
- If the pointed-to word is zero, reconciliation is bypassed.
- If the pointed-to word is nonzero, the reconciliation path is entered.

The pointer field is not needed for the gate when the byte is at most three. The pointed-to word need not be positive: every nonzero 32-bit bit pattern passes that test.

### First reconciliation call and product read

When reconciliation is enabled, the function calls F-da5096269123 with six arguments, in this order:

1. The pointer read from context offset `0x6c`.
2. The address of the mutable local two-word quantity.
3. The original input low word.
4. The original input high word.
5. The context address.
6. The requested value.

After this call returns, the function reads the signed 32-bit field at context offset `0x40` and multiplies it by the signed 32-bit requested value. Both operands are widened to signed 64-bit values for this multiplication.

Every product of two signed 32-bit integers fits in a signed 64-bit integer. Consequently, the product itself requires no truncation or overflow policy.

The timing of this read is significant:

- It occurs after F-da5096269123 returns, so a change that dependency makes to the multiplier field can affect the product.
- It occurs before the function checks that dependency’s result.
- Thus, even a nonzero dependency result does not prevent the multiplier-field read and product computation.

If F-da5096269123 returned a nonzero 32-bit result, that result becomes the function’s return value. No comparison, completion-field write, or final dependency call follows.

A dependency that invalidates the context can therefore cause the subsequent multiplier read to fail before its nonzero result is returned. This function provides no protection against that behavior.

### Quantity comparison

If the first dependency returns zero, the product is compared with the current local two-word quantity. The quantity may differ from its initial value because the dependency received writable access to it.

Equality compares the full 64-bit bit patterns. If equal, neither of the two adjustment dependencies is called.

For unequal values, the selection uses signed 64-bit ordering, not unsigned ordering:

- **Product greater than quantity:** call F-87fe98a22e06.
- **Product less than quantity:** call F-1d494943708d.

The source implements the ordering through high-word subtraction, a low-word borrow, and signed-overflow handling. That handling matters when the high words cross the signed boundary; the sign of a wrapped subtraction alone is not the comparison rule.

### Product-greater branch

When the product is strictly greater than the signed quantity, F-87fe98a22e06 receives:

1. A freshly read value from context offset `0x6c`.
2. The address of a fixed static datum.
3. The 32-bit value one.

The contents, extent, mutability, and semantic meaning of the static datum are not established.

If the dependency returns nonzero, the function immediately returns that result. If it returns zero, execution proceeds to completion.

### Product-less branch

When the product is strictly less than the signed quantity, F-1d494943708d receives:

1. A freshly read value from context offset `0x6c`.
2. The upper 32 bits of the product-minus-quantity difference, computed with the low-word borrow.
3. The low 32 bits of the product.
4. The high 32 bits of the product.

For precision, the difference argument is a word-level result. Let the product have low and high words, and let the current quantity have low and high words. The low-word comparison supplies a borrow of one exactly when the product’s unsigned low word is smaller than the quantity’s unsigned low word. The second argument is then the product high word minus the quantity high word minus that borrow, reduced modulo 2³².

This definition remains valid even when the mathematical signed difference would not fit in 64 bits. It must not be replaced by an assumption that the difference is an ordinary in-range signed integer.

The last two arguments describe the product, not the difference.

If F-1d494943708d returns nonzero, that result is returned immediately. Otherwise, execution proceeds to completion.

### Completion

Completion is reached when:

- Reconciliation was bypassed;
- The first dependency returned zero and the product equaled the current quantity; or
- The selected adjustment dependency returned zero.

The function then:

1. Writes the requested value as a 32-bit value at context offset `0x24`.
2. Calls F-779292dbb800 with the context address.
3. Returns zero if that call returns normally.

The context write precedes the final call. The final dependency’s return value, if any, is not used to select this function’s result.

The function does not restore the stored value after the final call. Therefore, the direct write is guaranteed on this path, but the value still present at offset `0x24` upon return depends on the final dependency’s behavior.

## Side effects

The only direct write to caller-addressed memory shown in this function is the 32-bit completion write at context offset `0x24`. The local two-word quantity is also initialized and exposed as writable storage to F-da5096269123.

There is no direct allocation or release operation in this function. This does not establish that its dependencies allocate nothing, release nothing, preserve globals, or leave the context unchanged.

No ownership transfer is established for:

- The context;
- The object reached through offset `0x6c`;
- The fixed static datum;
- The local quantity address passed to the first dependency.

The local quantity has invocation-local storage lifetime. The fragment provides no evidence that retaining its address is permitted.

The context-held pointer is read again for an adjustment call rather than guaranteed to be the same pointer used by the first dependency. A mutation to that field by F-da5096269123 can therefore change the recipient of the adjustment call.

There is no rollback. On a nonzero return from a reconciliation dependency, this function skips its completion write and final call, but any changes already made by dependencies remain outside its rollback control.

No synchronization or atomic access semantics are shown. Concurrent mutation guarantees are not established.

## Dependencies

| Opaque ID | Invocation condition | Role and result handling |
|---|---|---|
| F-da5096269123 | Both gate conditions pass | Receives the context-associated pointer, writable two-word quantity, original words, context, and requested value. Nonzero result is propagated, but only after the multiplier read and product computation. |
| F-87fe98a22e06 | First dependency succeeds and product is strictly greater than the signed quantity | Receives the current context-associated pointer, fixed datum address, and one. Nonzero result is propagated. |
| F-1d494943708d | First dependency succeeds and product is strictly less than the signed quantity | Receives the current context-associated pointer, difference high word, and product words. Nonzero result is propagated. |
| F-779292dbb800 | Completion path | Runs after the requested value is stored. Its result is not used; normal return is followed by a zero result. |

No dependency implementation is supplied. Their internal validation, memory effects, global effects, ownership rules, and failure mechanisms are documented_only unknowns.

## Acceptance examples

These are conditional acceptance cases derived from static evidence, not executed tests. They assume valid reached memory accesses and normally returning dependencies unless stated otherwise.

1. **Byte gate bypass**

   With the context byte equal to three, the context-held pointer is not read for reconciliation. No reconciliation dependency runs. The requested value is stored, the final dependency is called, and the result is zero.

2. **Zero pointed-to word**

   With the byte equal to four and the first word at the context-held pointer equal to zero, reconciliation is bypassed. The multiplier field is not read by this function on that path.

3. **First dependency failure still precedes a multiplier read**

   Suppose the gate passes and F-da5096269123 returns seven. The function reads the multiplier field and computes the product before returning seven. It does not perform its completion write or final call.

4. **Equal positive values**

   Suppose the first dependency returns zero, leaves the quantity equal to twelve, and the multiplier field is three while the requested value is four. The product is twelve. Neither adjustment dependency runs; completion follows.

5. **Strictly greater product**

   Suppose the product is twelve and the current quantity is eleven. F-87fe98a22e06 runs with the fixed datum address and the value one. A zero result permits completion; a nonzero result is propagated.

6. **Strictly smaller product with low-word borrow**

   Suppose the product is twelve and the current quantity is thirteen. F-1d494943708d receives a difference high word of `0xffffffff`, a product low word of twelve, and a product high word of zero.

7. **Signed ordering differs from unsigned ordering**

   Suppose the product is zero and the current quantity has both words equal to `0xffffffff`, representing negative one. The product is greater, so F-87fe98a22e06 is selected. Treating the quantity as a large unsigned integer would select the wrong dependency.

8. **First dependency changes later inputs**

   Suppose F-da5096269123 returns zero after changing the multiplier field and the pointer at offset `0x6c`. The product uses the changed multiplier. If adjustment is required, its dependency receives the changed pointer.

9. **Adjustment error suppresses only this function’s completion actions**

   If the selected adjustment dependency returns a nonzero result, that result is returned without the completion write or final call. Changes already made by dependencies are not undone.

10. **Final dependency observes the stored value**

    On completion, the requested value is written before F-779292dbb800 runs. If that dependency subsequently changes the field and returns normally, this function still returns zero and does not restore the earlier value.

## Uncertainties

The following remain documented_only:

- The domain purpose of the requested value, multiplier, quantity, and gate.
- Whether reconciliation is intended as movement, positioning, accounting, or another operation.
- The complete context layout and pointed-to object layout.
- The contents and meaning of the fixed static datum.
- Dependency ownership contracts and whether they retain passed addresses.
- Dependency effects on globals, context fields, other memory, or resource lifetimes.
- Whether dependencies can fail through mechanisms other than an ordinary returned status.
- The validity constraints imposed externally on negative requested values or negative quantities.
- Platform byte order and any alignment guarantees beyond those required by the shown accesses.

The fragment contains no explicit rejection of negative requested values, negative multipliers, or any particular input-word pattern. Such restrictions cannot be added without another source of authority.

## Evidence

The supplied static function establishes the field offsets and access widths, short-circuit gate, dependency argument order, post-call multiplier read, signed widened multiplication, two-word equality and signed ordering, error propagation, completion write, and final call order.

The interpretation of the routine as “reconciliation” is descriptive inferred intent, not a domain guarantee. The behavioral contract above is defined by the concrete reads, comparisons, calls, writes, and return conditions.

No dependency bodies, runtime traces, or runtime tests were supplied or performed. No claims about their internal behavior are established.
