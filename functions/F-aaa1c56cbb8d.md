<!-- Trusted provenance: F-aaa1c56cbb8d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-aaa1c56cbb8d

## Purpose

F-aaa1c56cbb8d performs a straight-line sequence of mixed-width memory mutations, repeatedly follows address-bearing values, and finishes with an unresolved indirect invocation carrying three explicit arguments.

**Established static evidence:** The supplied representation includes explicit byte, halfword, and word dereferences; unsigned 32-bit carry calculations; acquisitions from named fixed-address storage; writes relative to a function-associated address; repeated overwrites; and an indirect transfer represented as a call.

**Inferred intent:** Some locations may be linked records or execution state. Neither interpretation is established. Constants that correspond to printable characters do not establish string construction or text processing.

**Contract boundary:** This is a static, representation-level contract. Recovered types and control-transfer notation do not independently establish the executable calling convention. The widths of undeclared fixed-address globals are unresolved, even when their acquired values enter 32-bit destinations.

## Interface

### Explicit inputs and result

| Input | Represented type | Use |
|---|---|---|
| First argument | Uninterpreted 32-bit value | No explicit use in the body. |
| Second argument | Signed 32-bit integer | Initial doubled-address store and early address arithmetic. |
| Third argument | Unsigned 32-bit integer | Comparisons, arithmetic, and memory addressing. |
| Fourth argument | Character-sized value | Early byte calculation; character signedness is unresolved. |

The represented result type is void. There is no constructed return value. Observable effects are memory accesses, mutations, and the terminal indirect invocation. If that invocation returns normally, the represented function returns without a value.

### Implicit inputs

The body additionally depends on:

- An incoming signed 32-bit arithmetic value, called **S** here.
- An incoming byte used by the first store.
- An incoming unsigned 32-bit comparison value, called **R** here.
- An unresolved indirect target whose address is compared and later invoked.
- Values acquired from fixed-address storage.
- Memory reached through explicit inputs, retained values, and freshly loaded fields.

These names describe dependencies; they do not add formal arguments or establish a mechanism for supplying implicit state.

### Widths and memory requirements

Explicit byte, halfword, and word dereferences access 8, 16, and 32 bits respectively. Pointer-valued field accesses additionally depend on the recovered pointer model.

Named globals are different: their declarations are absent. Assignment to a 32-bit destination establishes a retained 32-bit value, not necessarily a 32-bit memory transaction. Likewise, conversion of a global to a signed halfword establishes the converted operand, not the global’s access width.

Every reached address must support its particular access. There are no represented bounds checks, null checks, alignment repairs, or recovery paths. Addresses include doubled values, sums of independently obtained values, fixed displacements, and locations preceding loaded addresses.

No allocation, release, or ownership transfer is explicitly identified. Accessibility, lifetime, and aliasing preconditions cannot be reduced to one contiguous input buffer.

## Behavior

### Arithmetic, acquisition, and ordering rules

Narrow stores retain the represented low 8 or 16 bits. A word store of `0x45` affects four bytes, unlike a byte store of the same numeric value.

Explicit carry operations test unsigned 32-bit addition. Their carry indicators are separate from low-32-bit arithmetic results. The representation does not establish a universal wrapping policy for signed arithmetic or address calculations.

Byte order is unspecified. Overlapping accesses therefore cannot generally be reduced to a unique final word value.

Separate occurrences of a fixed-global expression are separate logical acquisitions in this contract. They must not be replaced with one entry-time snapshot. This preserves dependencies and potential alias effects without asserting a physical bus-access count, volatility, or an exact machine instruction sequence.

For all fixed addresses below, source access width remains unresolved unless independently established. “Retained 32-bit value” describes the destination representation, not the underlying global declaration.

### Initial mutations and fixed-state acquisition

The opening effects have this order:

1. The incoming byte is stored at twice the second argument.
2. Byte `0x59` is stored at displacement `0x59` from the address associated with F-1b0ccebdf05e.
3. The low byte of S is stored at the sum of the third and second arguments.
4. The low halfword of the third argument is stored at the sum of the third argument and S.

The routine then acquires a value from `0x087FC710`, retaining it as unsigned 32-bit value **B**. A separate acquisition from that same global supplies the base for a halfword store at displacement `0x54`; the payload is the low halfword of S.

Next, acquisitions from `0x087FC6F8` and `0x087FC6E8` supply retained 32-bit values. The former later supplies a halfword payload; the latter later supplies a signed-halfword arithmetic operand. Their global access widths are not established by these destinations.

The two acquisitions from `0x087FC710` have different roles. An intervening change, or another permitted source of differing values, must not be hidden by treating them as one snapshot.

### Carry-dependent address construction

The routine forms a low unsigned 32-bit result from twice B plus the indicator for R being less than or equal to the third argument.

Its carry indicator is true if doubling B carries. Otherwise, it tests whether adding the comparison indicator to the low doubled result carries. The second carry test is short-circuited when the first succeeds.

The global at `0x087FC6E4` participates through distinct acquisitions:

- An initial value, increased by the carry indicator, supplies a retained address-related result.
- A later value is compared against `0xFFFFFFA6`.
- Only if that threshold condition is false does another acquisition participate in the carry test following addition of `0x59`.

The resulting adjustment is one when either the threshold condition or the conditionally evaluated carry test succeeds; otherwise it is zero. In the carry test, the earlier carry indicator is added to the low 32-bit result of the separately acquired value plus `0x59`. Overflow of that preliminary addition is not independently included in the condition.

An offset combines S, the doubled-base result, and this adjustment. A fresh acquisition from `0x087FC6F8` supplies the operand for retaining the indicator R ≤ that value. This comparison acquisition is distinct from the earlier payload acquisition.

The subsequent mutations are ordered:

- Store the offset’s low halfword at the third argument plus the offset.
- Increase the offset by R and store its new low halfword at the corresponding updated address.
- Store the low halfword of the earlier `0x087FC6F8` payload at twice the doubled-base result.
- Store byte `0x45` at displacement `0x9E` from the retained carry-related address.

### Transition using captured fixed state

After that byte store, the routine captures:

| Fixed address | Established retained use | Unresolved source property |
|---|---|---|
| `0x087FC73C` | Signed 32-bit destination-base value | Global declaration and memory-access width. |
| `0x087FC688` | Value converted to a signed 16-bit operand | Global declaration, original width, and memory-access width. |

In particular, the acquisition at `0x087FC688` is **not established as a word read followed by narrowing**, nor as a halfword memory read. Only the retained signed-halfword operand is established.

Both acquisitions precede the next potentially aliasing mutations. The routine explicitly reads a word from the third argument plus the updated offset, uses it as an address for a byte store of `0x45` at displacement `0x45`, and repeats the halfword store at the original source location.

A halfword is then stored at the captured destination base plus `0x45`. Its payload combines the signed-halfword interpretation of the earlier `0x087FC6E8` value, the captured signed-halfword operand from `0x087FC688`, and an indicator for the unresolved target address being below `0x46`.

The third-argument-relative word is loaded again. This newly acquired address receives **word** `0x45` at displacement `0x45`. Finally, the captured destination base plus `0x45` receives a byte derived from the fourth argument, `0x53`, and the retained comparison indicator from the later `0x087FC6F8` acquisition.

The halfword and byte destinations share a starting address. The repeated word load may observe modifications caused by intervening stores. Neither captured fixed-state value is reacquired for these uses.

### Central dependent mutations

The central body has no represented ordinary loop. Its addresses depend on earlier loads and mutations. Offsets are access locations, not evidence of a validated structure layout. Decimal displacement 100 means `0x64`, distinct from `0x74`.

**Initial linking and byte propagation.** A word from the third argument plus `0x14` is copied to the updated arithmetic offset plus `0x34`. Traversal through a captured base’s `0x74` field and another reached record’s `0x14` field leads to updates at `0x44` and `0x54`, including a self-address store.

One byte destination at `0x09` receives low bytes of two loaded addresses consecutively. An acquisition from `0x000000DB` supplies a retained 32-bit payload; the global’s memory-access width remains unknown.

Two explicit word loads from one `0x44` field are retained separately. A character loaded from twice an unsigned address is converted to an integer, shifted right by that address’s low eight bits, narrowed, and written to two byte destinations. Character signedness, signed right-shift behavior, and excessive shift counts remain unresolved.

A `0x54` field first receives a computed address and is immediately overwritten by the retained payload associated with `0x000000DB`. Other stores retain effects of the computed address. Subsequent mutations replace `0x44` and `0x74` fields and overwrite one shifted-byte destination.

**Mixed-width payload stage.** A reached record receives word `0x73` at `0x10`, byte `0x73` at `0x14`, then word `0x6E` at `0x10` and word `0x6E` at `0x20`, with intervening accesses preserved by the representation. Its byte at `0x1C` receives `0x79` and later `0x68`.

Separate word loads of a working record’s `0x64` field supply retained payloads with distinct later uses. A reached `0x54` word is retained and later restored. A word from a reached record’s `0x50` field replaces its `0x10` word and supplies its first byte. Additional mutations install address-bearing values and self-addresses at `0x44` and `0x54`.

**Later constant and traversal stages.** Traversals through `0x04`, `0x14`, and `0x74` fields lead to byte and word stores involving `0x64`, `0x74`, and `0x66`. Word destinations include offsets `0x10`, `0x30`, `0x40`, `0x54`, and `0x60`. Some same-starting-address byte and word stores occur consecutively; they remain distinct operations.

A loaded address minus `0x74` is stored as a word at a location `0x20` before that address. Other operations read and write back `0x64` or `0x54` fields. A byte at `0x0D` receives the same retained value twice consecutively.

Later traversal continues to replace links, copy low address bytes into byte locations, and follow freshly acquired fields. A working record’s `0x14`, `0x44`, `0x54`, `0x64`, and `0x74` fields can all influence subsequent destinations. No list consistency, object identity, or ownership invariant is established.

This central description specifies significant dependencies and overwrite boundaries, not a complete normalized access ledger. Full central trace equivalence remains **documented_only**.

### Terminal mutations and invocation

A value acquired from `0x087FC9C0` is retained in a signed 32-bit destination. The working record’s current `0x74` field is explicitly loaded as a word.

The terminal sequence then:

- Stores that field’s low halfword at the sum of the field value and working-record address.
- Stores the low halfword of the retained fixed value at twice another retained traversal address.
- Loads a character from the sum of a different retained traversal address and the working-record address, converting it to unsigned 32 bits.
- Stores the retained field halfword at the sum of the fixed value and field value.
- Repeats the first halfword store.
- Stores word `0x45` at the converted character value plus `0x45`.

Next come, in order, acquisitions from `0x087FC9DC` and `0x087FC9C8`, followed by an explicit word load from the working record’s `0x44` field. The fixed acquisitions supply retained 32-bit values; their memory-access widths are unresolved.

The final stores are byte `0x44` at the converted character value plus `0x44`, the retained `0x087FC9C8` payload as a word at twice the retained `0x087FC9DC` value, and the latter value’s low halfword at its sum with another retained traversal address.

The unresolved indirect dependency receives:

1. The converted character value bitwise ORed with `0x44`.
2. The retained value associated with `0x087FC9DC`.
3. The working-record `0x44` word captured before the final three stores.

Later aliasing does not replace these captured arguments. The actual transfer convention and return behavior remain unresolved.

## Side effects

Mutations can reach caller-accessible storage, fixed-address storage through aliasing, and storage relative to F-1b0ccebdf05e’s associated address. Effects are not confined to argument locations or one allocation.

Mixed-width stores can change values subsequently treated as addresses. Intermediate writes and repeated acquisitions therefore matter independently of final memory contents.

No rollback, explicit error result, exception handler, synchronization operation, allocation, or release is shown. Invalid accesses or an unsuccessful indirect transfer may occur after substantial mutation; the source does not define their outcome.

No preservation, allocation, release, or ownership guarantee is imported from the unresolved dependency.

## Dependencies

| Dependency | Established relationship | Unknowns |
|---|---|---|
| F-1b0ccebdf05e | Its associated address anchors the early byte store at `+0x59`; no direct invocation is shown. | Writability, executable-storage involvement, and relationship to its behavior. |
| Unresolved indirect dependency; opaque ID unavailable | Its address participates in a comparison and receives the terminal three-argument transfer. | Identity, signature, calling convention, state preservation, ownership effects, and return behavior. |
| Fixed-address storage | Supplies separately acquired arithmetic values, addresses, comparisons, and payloads. | Declarations, access widths, initialization, stability, volatility, lifetime, and aliasing. |
| Traversed storage | Supplies fresh links and receives explicit mixed-width accesses. | Object boundaries, pointer validity, invariants, and ownership. |

## Acceptance examples

These are conditional static criteria, not executed tests.

### Separate base acquisitions

If the first acquisition at `0x087FC710` supplies retained value `0x1000` and the second supplies `0x2000`, the early halfword destination is `0x2054`, while doubling uses `0x1000`. This example does not establish either acquisition’s memory-access width.

### Conditional acquisition and carry

If the threshold acquisition at `0x087FC6E4` exceeds `0xFFFFFFA6`, the adjustment is one and the carry-test acquisition is skipped.

If the threshold acquisition yields zero, the earlier carry is one, and the carry-test acquisition supplies `0xFFFFFFA6`, adding `0x59` produces low result `0xFFFFFFFF`; adding the carry overflows. The adjustment is one despite the false threshold condition.

### Captured transition values

If the captured base from `0x087FC73C` is `0x3000`, a later aliasing mutation of that global does not redirect the transition stores from `0x3045`.

If the retained signed-halfword operand from `0x087FC688` is −1, that operand remains −1 despite later changes to its source. This criterion requires neither a 16-bit nor a 32-bit memory acquisition at the fixed address.

### Distinct comparison and payload

If the two acquisitions from `0x087FC6F8` differ, the halfword payload uses the first retained value; the later byte adjustment uses R compared with the second. One shared snapshot is not equivalent.

### Equality and width boundaries

When R equals the third argument, the doubling adjustment includes one. For B equal to `0x80000000`, doubling has low result zero and carries.

Word `0x73` at `0x10` affects four bytes; byte `0x73` at `0x14` affects one. Both `0x1C` byte writes, first `0x79` and later `0x68`, belong to the sequence even when only the latter survives.

### Character conversion and argument capture

A terminal byte with bit pattern `0x80` can yield `0xFFFFFF80` under signed-character interpretation or `0x00000080` under unsigned-character interpretation. The evidence does not select one.

The third terminal argument remains the captured `0x44` word even if a final store subsequently changes that field.

## Uncertainties

The following remain **documented_only**:

- Actual purpose, valid calling convention, and delivery of implicit inputs.
- Declarations and memory-access widths of every referenced fixed-address global, including `0x087FC688` and `0x000000DB`.
- Physical read counts, volatility, and external observability beyond logical acquisition order.
- Signed overflow, narrowing details, address wraparound, alignment, byte order, and fault behavior.
- Character signedness, signed right shift, and out-of-range shift counts.
- Meaning and validity of the unresolved target-address comparison.
- Pointer representation, object boundaries, alias restrictions, lifetimes, and ownership.
- Whether function-associated writes modify executable storage.
- Indirect dependency identity, effects, transfer convention, and return behavior.
- Complete normalization of the central access trace.

## Evidence

The sole evidence is the supplied static representation. Explicit typed dereferences support the stated byte, halfword, and word access widths. Destination types and conversions support retained-value widths, including the signed-halfword operand associated with `0x087FC688`; they do **not** establish the access widths of undeclared globals.

The representation supports distinct global-expression acquisitions, conditional evaluation of the carry-test acquisition, captured transition operands, dependent mutations, repeated overwrites, and terminal argument capture.

Warnings concerning overlapping global symbols and unresolved control transfer limit interpretation. They do not establish another algorithm or supply missing declarations.

No implementation access, runtime execution, memory trace, or tests were available. All acceptance examples are conditional consequences of static evidence, not observed results.
