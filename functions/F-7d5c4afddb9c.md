<!-- Trusted provenance: F-7d5c4afddb9c; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-7d5c4afddb9c

## Purpose

F-7d5c4afddb9c recognizes two input forms locally. Either successful recognition returns the 32-bit result `1`. Every other input proceeds to fallback acquisition through an indirect operation associated with the receiver.

If acquisition returns null, the function returns the 32-bit result `0`. Otherwise, it delegates the original input-record address to an indirect operation associated with the acquired object and returns that operation’s 32-bit result unchanged.

The established behavior is recognition followed by conditional delegation. The application-level meanings of the recognized forms, the objects, and the results are unknown. In particular, the delegated result is not established to be Boolean.

## Interface

### Inputs and accessed layout

The function has two explicit inputs:

- **Receiver:** an object address used only if local recognition fails.
- **Input record:** an address used to read a discriminator and, conditionally, additional fields.

Offsets below are byte offsets. The representation establishes 32-bit integer field accesses and integer-carried addresses for the listed input fields. It does not establish complete object layouts or an independently documented pointer ABI.

| Object | Offset | Accessed content |
|---|---:|---|
| Input record | `+4` | 32-bit discriminator |
| Input record | `+8` | 32-bit unsigned comparison word for discriminator `0x21`; address-bearing 32-bit field for discriminator `0x16` |
| Input record | `+12` | 32-bit unsigned packed word, read conditionally for discriminator `0x21` |
| Nested object addressed through input `+8` | `+4` | 32-bit word used by the `0x16` recognition form |
| Receiver | `+0` | Value used as the receiver’s operation-table address |
| Receiver’s operation table | `+0xCC` | Acquisition operation entry |
| Acquired object | `+0` | Value used as the acquired object’s operation-table address |
| Acquired object’s operation table | `+0x30` | Delegation operation entry |

No direct access to input-record offset `+0` is shown.

### External comparison operands

Two external data expressions participate in recognition. Their declarations are absent.

- **Datum A** is evaluated and converted into a 32-bit unsigned saved value. This contract calls the result **effective A**. The function compares the input word at `+8` against effective A.
- **Datum B** is used directly as an equality operand against the unsigned upper-half extraction of the packed input word. No explicit conversion of datum B to a 32-bit or 16-bit saved value is shown.

The 32-bit width of effective A does **not** establish datum A’s underlying storage width or access width. Neither the storage width nor the access width of datum B is established. Datum B’s missing declaration also leaves its signedness and the precise comparison conversions unresolved.

The external expressions’ values, mutability, and semantic meanings are unknown.

### Output

The function returns one 32-bit value:

- `1` following either successful local recognition;
- `0` when fallback acquisition returns null;
- otherwise, the delegated operation’s result without normalization.

There is no shown output parameter.

### Preconditions

Successful execution requires readable storage only along the path actually taken:

- Input `+4` must be readable for every path.
- For discriminator `0x21`, input `+8` and the evaluation of datum A must be valid.
- Input `+12` and the evaluation of datum B are required only if the comparison against effective A succeeds.
- For discriminator `0x16`, input `+8` must supply an address through which nested offset `+4` can be read.
- Fallback requires a usable receiver, receiver-table address, and acquisition entry.
- A non-null acquisition result must support the subsequent object-table access and indirect delegation.

No explicit pointer, alignment, bounds, or callable-entry validation is shown.

The receiver is not directly accessed on an accepted local path. Its validity is therefore not a local prerequisite for those paths.

## Behavior

### Discriminator selection

The first established input access reads the 32-bit discriminator at input `+4`.

The function recognizes one form for discriminator `0x21` and another for discriminator `0x16`. Other discriminator values proceed directly to fallback.

The discriminator is retained for subsequent local decisions. The function does not reread input `+4` before selecting fallback.

A rejected `0x21` input does not undergo the nested-object access belonging to the `0x16` form.

### Packed recognition form: discriminator `0x21`

The function first reads the 32-bit unsigned word at input `+8`, then evaluates datum A and saves its converted 32-bit unsigned value as effective A.

Recognition can continue only when the input word equals effective A. This comparison concerns the converted value, not necessarily the complete underlying value of datum A. If datum A has a wider integral type, its conversion can discard information outside the 32-bit destination domain.

If the first comparison fails:

- The function does not read input `+12`.
- It does not evaluate datum B.
- It proceeds to fallback.

If the first comparison succeeds, the function reads the 32-bit unsigned packed word at input `+12`. It compares datum B directly against that word’s upper 16 bits, extracted by an unsigned right shift of 16 positions.

The extracted value is in the range `0x0000` through `0xFFFF`. Datum B is not shown being masked to 16 bits or explicitly converted to a 32-bit temporary. Equality must therefore be understood using datum B’s actual declaration and the applicable operand conversions. Those details are unavailable.

If the upper-half comparison fails, recognition fails. If it succeeds, the function checks whether the packed word’s lower 16 bits equal `0x7A0D`.

The lower-half condition is implemented by subtraction in a 32-bit unsigned value followed by a zero comparison. Its acceptance condition is exact equality with `0x7A0D`; it is neither a range test nor a signed-order test.

This form returns `1` exactly when all of the following hold:

1. The discriminator is `0x21`.
2. Input `+8` equals effective A.
3. Datum B compares equal to the unsigned upper-half extraction under the actual operand types.
4. The packed word’s lower 16 bits equal `0x7A0D`.

No receiver-table access or indirect dependency invocation occurs on this accepted path.

### Nested recognition form: discriminator `0x16`

For discriminator `0x16`, the function reads the address-bearing field at input `+8`, then reads the 32-bit word at offset `+4` from the addressed object.

It adds `0xBFFFFFD2` in the represented 32-bit arithmetic domain and accepts when the resulting bit pattern equals `0x23800000`.

The addition is modulo \(2^{32}\). Its acceptance condition is exactly that the nested word have bit pattern `0x6380002E`:

\[
(0x6380002E + 0xBFFFFFD2) \bmod 2^{32}
= 0x23800000.
\]

This modular interpretation does not introduce a checked-overflow error or an unbounded-integer calculation.

Acceptance returns `1` without fallback. This form does not directly access input `+12` or evaluate either external comparison datum.

### Fallback acquisition

Fallback occurs for every input not accepted locally.

The function obtains the receiver’s operation-table address, selects its entry at byte offset `0xCC`, and invokes the selected operation, identified here as **F-indirect-01**.

The supplied representation shows no explicit arguments for this invocation. It does not establish whether a machine-level calling convention supplies an implicit receiver or other argument state.

The returned value is interpreted as an object pointer:

- If null, the function returns `0` immediately.
- If non-null, execution proceeds to delegation.

There is no acquired-object table access on the null-result path.

### Delegation

After a non-null acquisition result, the function reads the acquired object’s operation-table address and selects its entry at byte offset `0x30`, identified here as **F-indirect-02**.

The delegated operation receives these two explicit values, in order:

1. The acquired object pointer.
2. The original input-record address.

The wrapper does not copy, replace, or normalize the input record before passing that address.

F-indirect-02’s 32-bit result becomes the function’s result unchanged. There is no local Boolean conversion, second recognition pass, or result-dependent cleanup.

## Side effects

### Direct reads and their order

No direct memory stores are shown.

The established path-dependent access order is:

| Path | Ordered local accesses |
|---|---|
| Accepted `0x21` | Discriminator; input `+8`; datum A evaluation and conversion; input `+12`; datum B evaluation |
| `0x21` rejected at first comparison | Discriminator; input `+8`; datum A evaluation and conversion; fallback accesses |
| `0x21` rejected later | Discriminator; input `+8`; datum A evaluation and conversion; input `+12`; datum B evaluation; fallback accesses |
| Accepted `0x16` | Discriminator; address-bearing field at input `+8`; nested word at `+4` |
| Rejected `0x16` | The same nested-form accesses, followed by fallback accesses |
| Other discriminator | Discriminator, followed by fallback accesses |

“Datum evaluation” does not assert a particular physical load width, number of machine loads, or storage representation. The missing external declarations prevent those conclusions.

Within fallback, receiver-table lookup precedes acquisition. Acquired-object table lookup occurs only after acquisition returns non-null. Delegation follows that lookup.

These relationships describe the supplied representation. They do not establish hardware memory barriers, atomic accesses, synchronization, or a concurrency-safe snapshot of input and external state.

### Effects of dependencies

Neither indirect operation’s implementation is supplied. Either may affect globals, objects, resources, or control flow.

The contract does not assume that either dependency:

- preserves global data;
- leaves the receiver or input record unchanged;
- preserves operation-table contents;
- allocates or avoids allocating memory;
- retains, releases, or otherwise manages references;
- returns normally.

The acquired-object table is read after F-indirect-01 returns. It is therefore the table accessible through that returned object after acquisition-side effects, not a table captured beforehand.

### Ownership and lifetime

The wrapper shows no explicit allocation, deallocation, retain, or release operation.

Receiving the acquired pointer does not establish whether it is owned, borrowed, reference-counted, or valid beyond delegation. Passing the original input-record address does not establish transfer or retention of ownership.

Any such obligations belong to unknown dependency or caller contracts and remain documented_only.

### Error behavior

The only explicit locally generated failure-like result is `0` for null acquisition. Its application-level meaning is unknown.

No recovery path is shown for invalid addresses, unreadable external data, invalid table entries, or abnormal dependency termination. Delegated error values, if any, are returned without interpretation.

## Dependencies

The following opaque IDs identify dependency roles, not proven unique implementations:

| Opaque ID | Role | Established interface |
|---|---|---|
| F-indirect-01 | Acquisition through receiver-table offset `0xCC` | No explicit arguments shown; returns a value interpreted as an object pointer |
| F-indirect-02 | Delegation through acquired-object-table offset `0x30` | Receives acquired object and original input address; supplies the forwarded 32-bit result |

Different operation tables may select different targets for either role.

External data dependencies are separate from these callable dependencies:

- **Datum A:** evaluated into effective A, a 32-bit unsigned saved comparison value. Its declaration, storage width, access width, and pre-conversion value domain are unknown.
- **Datum B:** used directly against an unsigned upper-half extraction. Its declaration, storage width, access width, signedness, and resulting comparison conversions are unknown.

Neither datum is established to be constant across calls or protected against concurrent changes.

## Acceptance examples

These are conditional expectations derived from static evidence, not runtime results. All examples require valid accesses and normally returning dependencies along the selected path.

### Matching packed form

Suppose:

- The discriminator is `0x21`.
- Input `+8` equals effective A.
- Datum B’s declared type can represent `0x1234`, and its value compares equal to the extracted unsigned value `0x1234`.
- Input `+12` contains `0x12347A0D`.

The function returns `1`. Neither indirect dependency is invoked, and the receiver is not directly accessed.

This example constrains effective comparison values. It does not assert that either external datum occupies four bytes.

### Datum A conversion

Suppose datum A has a wider integral representation whose conversion to the shown 32-bit unsigned destination produces `0x12345678`.

With discriminator `0x21`, input `+8` equal to `0x12345678` passes the first comparison, irrespective of information discarded by that conversion. Final acceptance still requires the upper-half and lower-half conditions.

Whether the actual datum A declaration permits this scenario is documented_only.

### Early packed-form rejection

Suppose discriminator `0x21` selects the packed form, but input `+8` differs from effective A.

The function proceeds to fallback without reading input `+12` or evaluating datum B. If F-indirect-01 returns null, the result is `0`.

Unreadable storage at input `+12` alone does not prevent this particular path from completing.

### Datum B is not shown truncated

Suppose the first packed-form comparison succeeds and input `+12` contains `0x12347A0D`.

If datum B’s declared type represents `0x00011234` and the applicable comparison conversions preserve its distinction from `0x1234`, the upper-half comparison fails and fallback occurs.

Equality of only the lower 16 bits of datum B does not establish acceptance. This example is conditional on representability and comparison semantics; the missing declaration prevents an unconditional claim about datum B’s width.

### Lower-half rejection

Suppose the first comparison succeeds, datum B compares equal to upper-half value `0x1234`, and the packed word is `0x12347A0E`.

The lower-half condition fails, so fallback occurs.

### Matching nested form

With discriminator `0x16`, a valid nested address, and nested word `0x6380002E`, the function returns `1` without evaluating the external data or invoking either indirect dependency.

Replacing the nested word with `0x6380002F` causes local rejection and fallback.

### Delegated result propagation

For an unrecognized input, suppose F-indirect-01 returns a usable non-null object and F-indirect-02 returns `0xDEADBEEF`.

F-indirect-02 receives that object and the original input-record address. The wrapper returns `0xDEADBEEF`, not a Boolean normalization.

## Uncertainties

The following remain **documented_only**:

- The application-level meaning of either recognized form or any result.
- The external datum declarations, storage widths, access widths, and actual values.
- Datum A’s value before conversion to effective A.
- Datum B’s signedness, value domain, and precise comparison conversions.
- Complete object layouts, alignment requirements, byte order, and pointer ABI.
- The acquisition calling convention and any implicit arguments.
- The indirect targets’ identities, effects, ownership rules, and error contracts.
- Pointer lifetimes and any obligations after delegation.
- Concurrency behavior and synchronization requirements.
- Outcomes after invalid accesses, invalid indirect calls, or abnormal dependency termination.

A recognition-and-delegation purpose is supported structurally. Interpreting `1` as “supported,” “valid,” or “equal” would be inferred intent, not established semantics.

## Evidence

The supplied static representation establishes the discriminator branches, conditional field accesses, 32-bit unsigned saved comparison value for datum A, direct use of datum B in the upper-half comparison, low-half equality condition, modular nested-form arithmetic, indirect-operation offsets, null-acquisition return, explicit delegation arguments, and result forwarding.

It does not include the declarations of the external data. Consequently, it does not establish their storage or load widths, nor does it justify treating datum B unconditionally as a 32-bit value.

The nested-word acceptance value `0x6380002E` follows algebraically from the represented 32-bit addition and equality comparison. It is not an observed execution result.

No dependency implementations, external-data definitions, runtime traces, or execution results were supplied. No runtime tests have been performed.
