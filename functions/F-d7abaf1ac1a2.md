<!-- Trusted provenance: F-d7abaf1ac1a2; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-d7abaf1ac1a2

## Purpose

F-d7abaf1ac1a2 produces a Boolean result from a dynamically obtained 64-bit value, two helper operations, and a conditional unsigned comparison against memory.

The established decision is:

- If the two 32-bit comparison words associated with the helper-output storage are equal, return false.
- If they differ, return whether a 32-bit field in the third argument’s storage is strictly greater, as an unsigned value, than the 32-bit value read eight bytes before the address represented by the first helper’s comparison word.

An inferred interpretation is that the helpers construct position-like or iterator-like results and that the function compares a threshold with metadata preceding one position. That interpretation is not established. The contract does not require a container, iterator, timestamp, or other particular domain model.

This is a source-facing contract. It describes the supplied static evidence and its limits; no runtime tests have been performed.

## Interface

The function accepts four arguments and returns a Boolean.

| Argument | Established use |
|---|---|
| First argument | Pointer to receiver storage. Its leading word supplies the base used to locate a dynamically dispatched callable at byte offset `0x28`. The receiver is also passed to that callable. |
| Second argument | A 32-bit value with no explicit use in the supplied body. |
| Third argument | A 32-bit address-bearing argument used to access fields at byte offsets `0x14` and, conditionally, `0x10`. |
| Fourth argument | A 32-bit value forwarded to the dynamically dispatched callable. |

The supplied representation supports a 32-bit address model: pointer-bearing words and explicitly read fields are four bytes wide. The callable’s result is stored in an eight-byte local object.

### Receiver requirements

For an execution that reaches and returns from the dynamic call:

- The receiver must support the leading-word read.
- The resulting dispatch base must support reading a callable entry at byte offset `0x28`, or 40 bytes.
- That entry must identify a callable compatible with the observed invocation: receiver first, fourth argument second, and an eight-byte result.

These are execution prerequisites, not validations performed by this function.

### Third-argument storage requirements

The storage addressed by the third argument must support a four-byte read at offset `0x14`, or 20 bytes, whenever each helper argument is prepared.

If the final comparison words differ, the storage must additionally support a four-byte read at offset `0x10`, or 16 bytes.

The field at offset `0x14` is read separately for each helper call. It is not established that the values supplied to the two helpers are equal.

### Helper-output requirements

Each helper receives the address of a four-byte local region. The supplied local layout places a separate four-byte comparison word immediately after each such region.

The caller subsequently reads those comparison words. Their initialization is not explicit in this function. Consequently, a defined result depends on dependency behavior or additional layout information establishing that these words acquire valid values before use. An eight-byte helper result is a plausible explanation, but it is not proven by the supplied function alone.

## Behavior

### 1. Dynamic invocation

The function obtains a callable through the receiver’s leading dispatch reference, using byte offset `0x28` within that dispatch storage.

It invokes the selected callable with:

1. The receiver.
2. The fourth argument.

The eight-byte result is stored locally. That local result remains available while the subsequent helpers run, and its address is passed to F-8c2906987f41.

The dynamic dependency’s implementation is unavailable. It may have side effects. In particular, this contract does not assume that it preserves globals, receiver state, or storage reachable through other arguments.

### 2. First helper operation

After the dynamic call returns, the function reads the four-byte field at offset `0x14` from the storage addressed by the third argument.

It invokes F-8c2906987f41 with:

1. The address of the first helper-output region.
2. The just-read four-byte field value.
3. The address of the local eight-byte dynamic-call result.

For the final decision, the relevant word is the four-byte comparison word immediately following this helper-output region. Below, this is called the **selected word**.

The supplied body does not establish the helper’s write extent, interpretation of the input field, or treatment of the eight-byte result. It also does not establish whether the helper mutates that result through the supplied address.

### 3. Second helper operation

After F-8c2906987f41 returns, the function reads the field at offset `0x14` again.

It invokes F-4249bb465234 with:

1. The address of the second helper-output region.
2. This newly read four-byte field value.

For the final decision, the relevant word is the four-byte comparison word immediately following this second region. Below, this is called the **reference word**.

The second read is behaviorally significant. For example, if the first helper changes the field through an alias, the second helper receives the changed value. The caller does not cache the first field read for reuse.

### 4. Comparison and return

The body first forms an unsigned less-than-or-equal comparison between the reference word and the selected word. It then checks whether the words differ.

If the words are equal, the initial comparison is true, and its negation is returned. Therefore:

**Equal comparison words always produce false on a normally completing, defined execution.**

In that case, the function does not perform the final field read at offset `0x10` and does not read memory eight bytes before the selected address.

If the words differ, the initial ordering result is replaced. The function:

- Reads a four-byte unsigned value from offset `0x10` in the third-argument storage.
- Interprets the selected word as an address and reads a four-byte unsigned value at that address minus eight bytes.
- Returns the negation of the field-value-less-than-or-equal comparison.

Equivalently, with unequal comparison words, it returns true exactly when the field at offset `0x10` is strictly greater than the indirectly read value.

The numeric ordering of the selected and reference words does not determine the final answer in the unequal case. Their equality matters; their relative order does not survive the replacement comparison.

### Width and address arithmetic

Both final comparison operands are unsigned 32-bit values. Their ordering is not signed ordering.

The subtraction of eight is performed on the unsigned selected word before dereferencing the resulting address. Under the represented 32-bit model, this subtraction wraps modulo \(2^{32}\). Wrapping does not establish that the resulting address is readable.

The function contains no explicit check for:

- A null receiver or invalid dispatch reference.
- An invalid third-argument address.
- An uninitialized helper comparison word.
- An invalid selected address.
- Address underflow before the indirect read.
- Alignment or memory accessibility.

There is no explicit error result distinct from false. A failure to satisfy an access or call prerequisite is not specified as a normal Boolean outcome.

## Side effects

### Direct effects

The body directly stores the dynamic call’s eight-byte result in local storage and uses local state for its Boolean decision.

It explicitly reads caller-reachable memory for:

- Receiver dispatch resolution.
- The offset-`0x14` field before each helper call.
- The offset-`0x10` field when the comparison words differ.
- The four-byte value eight bytes before the selected address when the comparison words differ.

The supplied body contains no explicit write to caller-owned storage, allocation, or release operation.

### Dependency effects and ownership

Absence of explicit external writes does not make the function pure. All three invoked dependencies may have effects not visible in this body.

The local eight-byte result and the helper-output addresses are supplied for use during calls. The caller does not explicitly transfer ownership, retain them beyond its return, or release them through a visible cleanup call. Whether a dependency retains an address, allocates resources, or releases memory is unknown.

The function has no visible synchronization or snapshot mechanism. Its separated reads must not be described as an atomic observation of receiver or third-argument state.

### Ordering limits

The established call order is:

1. Dynamic dependency.
2. F-8c2906987f41.
3. F-4249bb465234.
4. Final decision.

The offset-`0x14` reads occur when preparing their respective helper calls. The final indirect read is conditional on unequal comparison words.

The source does not establish a machine-level ordering between the two memory operands of the final unsigned comparison. Neither operand should be specified as necessarily read first.

## Dependencies

| Dependency | Observed interface and role | Not established |
|---|---|---|
| Dynamic dependency; opaque ID unavailable | Selected through receiver dispatch storage at offset `0x28`; receives receiver and fourth argument; supplies an eight-byte result. | Concrete identity, semantic meaning, memory effects, failure behavior, ownership behavior. |
| F-8c2906987f41 | Receives a local output address, the current offset-`0x14` field value, and the address of the eight-byte result. | Output size, guaranteed initialization of the selected word, input mutation, retained references, external effects. |
| F-4249bb465234 | Receives another local output address and a fresh offset-`0x14` field value. | Output size, guaranteed initialization of the reference word, external effects, ownership behavior. |

No preservation guarantees are imported from these dependencies.

## Acceptance examples

These are conditional acceptance criteria derived from static evidence, not observed executions. Each assumes the dependencies return normally, the comparison words are validly initialized, and all executed accesses are valid.

1. **Equal comparison words**  
   If both words have the same 32-bit value, the result is false. No final access at offset `0x10` or at the selected address minus eight is required.

2. **Unequal words, threshold exceeds indirect value**  
   If the words differ, the offset-`0x10` field is 11, and the indirectly read value is 10, the result is true.

3. **Unequal words, equal final operands**  
   If the words differ and both final comparison operands are 10, the result is false. The test is strict greater-than after negation, not greater-than-or-equal.

4. **Unequal words, threshold below indirect value**  
   If the field is 9 and the indirectly read value is 10, the result is false.

5. **Unsigned boundary**  
   With unequal words, a field value of `0xFFFFFFFF` and an indirectly read value of zero produce true. Reversing those values produces false.

6. **Word ordering does not select the answer**  
   Whether the selected word is numerically above or below the reference word, unequal words with final operands 11 and 10 produce true.

7. **Mutation between helper calls**  
   If the offset-`0x14` field is 100 when the first helper’s arguments are prepared and becomes 200 before the second helper’s arguments are prepared, the helpers receive 100 and 200 respectively.

8. **Address wrap is not validation**  
   A selected word of four makes the subtraction produce address `0xFFFFFFFC` under the 32-bit model. An unequal-word execution requires that address to support the four-byte read; false is not a specified fallback for an invalid access.

## Uncertainties

The following remain **documented_only**:

- The concrete dynamic dependency and its calling-convention details.
- Whether each helper writes an eight-byte object spanning its named output region and the adjacent comparison word.
- Any guarantee that the comparison words are initialized.
- The semantic types of the third-argument fields and the selected address.
- The significance of the eight-byte prefix before the selected address.
- Dependency effects on globals, aliased memory, resource ownership, or retained addresses.
- Alignment, address-space, and exceptional-control-flow behavior.
- Whether the apparent result-construction and position-comparison interpretation matches intended use.

The second argument has no explicit use in this body. That establishes only its absence from the visible computation, not a broader ABI or caller-level guarantee.

## Evidence

Established evidence consists solely of the supplied function representation: four arguments, receiver-based dynamic dispatch, one eight-byte result, two ordered helper calls, two separate reads at offset `0x14`, unsigned comparison words, and the conditional final comparison using offset `0x10` and an address reduced by eight.

Helper implementations, runtime traces, and independent layout declarations were not supplied. No runtime tests have been performed. The contract therefore distinguishes the visible decision rule from the dependency and storage prerequisites needed to execute it safely.
