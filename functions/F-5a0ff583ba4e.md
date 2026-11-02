<!-- Trusted provenance: F-5a0ff583ba4e; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-5a0ff583ba4e

## Purpose

F-5a0ff583ba4e prepares two fields in caller-designated storage, then transfers control to an unresolved indirect dependency. A 32-bit unsigned input determines both a stored value and the dependency’s fourth explicit argument.

The established behavior consists of two ordered 32-bit stores, a subsequent read from global storage at address 0x00000054, and an indirect invocation with four explicit arguments. The global read’s width and type are unknown.

The stores are consistent with initializing a self-referential record, while the derived argument may represent a category or index. Those interpretations are inferred intent, not established semantics. The dependency’s identity, effects, ownership behavior, and return behavior remain unresolved.

## Interface

### Inputs

The supplied representation exposes four input positions:

| Position | Represented input | Established use |
|---|---|---|
| First | Uninterpreted 32-bit value | No explicit use in the visible body. |
| Second | Uninterpreted 32-bit value | No explicit use in the visible body. |
| Third | 32-bit integer used as an address | Identifies the destination base and supplies the value stored at that base. |
| Fourth | Unsigned 32-bit integer | Supplies both arithmetic derivations. |

This contract calls the third input the **base address** and the fourth input the **unsigned input**. These are descriptive interface terms, not assertions about application-level meaning.

The first two inputs do not participate in the explicit arithmetic, stores, or dependency arguments. Their lack of explicit use does not establish that their incoming machine representations are inaccessible to the unresolved dependency.

### Memory layout and global access

| Location | Access | Width and interpretation |
|---|---|---|
| Base address + 0x38 | Write | 32-bit unsigned result of shifting the unsigned input right by 15 bits. |
| Base address + 0x00 | Write | 32-bit representation of the base-address input itself. |
| Global storage at 0x00000054 | Read | Width and type unknown; the resulting value supplies the dependency’s first explicit argument. |

The store at offset 0x38 occupies offsets 0x38 through 0x3B. The store at offset zero occupies offsets zero through three. If both belong to one ordinary contiguous object, that object must extend through offset 0x3B. No allocation size or object boundary is established.

The global location is established, but its declaration is absent. Neither the global-symbol overlap warning nor the destination-store widths determine the global read’s width. No four-byte extent, signedness, pointer interpretation, or argument-conversion rule is established for that read.

### Output and completion

No return value is exposed by the supplied representation. If the indirect dependency returns normally to the visible continuation, F-5a0ff583ba4e returns without explicitly consuming or exposing a dependency result.

There is no visible status output, validation result, or error conversion.

The two stored values are observations of the visible prefix, not unconditional return-time postconditions. The dependency may subsequently modify or release the affected storage.

### Preconditions for normal completion

Normal execution of the visible prefix requires:

- The destination at base offset 0x38 to support the represented 32-bit store.
- The destination at base offset zero to support the represented 32-bit store.
- Effective-address formation to be valid under the actual target’s rules.
- Global storage at 0x00000054 to support the read actually performed, whose width and type are unresolved.
- The indirect transfer to reach a destination compatible with the actual transfer mechanism and argument presentation.

No visible checks enforce these requirements. Exact alignment constraints, access permissions, arithmetic-overflow consequences for address formation, and hardware fault behavior are not established.

## Behavior

### Ordered destination writes

The first explicit memory operation stores the unsigned input logically shifted right by 15 bits at base offset 0x38.

Because the input is unsigned and 32 bits wide, the stored result lies between zero and 0x0001FFFF, inclusive. Existing contents of that field do not participate in the calculation, and no explicit read of those contents precedes the store.

The second explicit memory operation stores the 32-bit base-address value at base offset zero. It stores the address value supplied by the caller, not the previous contents at that address.

Under the represented address model, the second store creates a self-reference. Whether that field is intended to serve as a link, header, or another application-specific construct is unknown.

The offset-0x38 store precedes the offset-zero store. These are separate operations, not an established atomic update.

### Dependency argument preparation

After both stores, the function presents four explicit argument positions to the unresolved dependency **D-01**:

| Position | Established value |
|---|---|
| First | The value obtained by reading global storage at 0x00000054; read width and type unknown. |
| Second | Zero; the actual ABI-level argument representation is not established. |
| Third | Zero; the actual ABI-level argument representation is not established. |
| Fourth | A value derived from the unsigned input using 32-bit unsigned arithmetic. |

The fourth argument is formed by logically shifting the unsigned input right by 12 bits, subtracting 0x52 modulo \(2^{32}\), and logically shifting the subtraction result right by four bits.

The subtraction occurs before the final shift. It must not be replaced with a subtraction performed after that shift or with an independently shifted constant.

The initial shift yields a value in the inclusive range zero through 0x000FFFFF. Subtraction therefore wraps when that value is below 0x52. The final logical shift yields a value between zero and 0x0FFFFFFF; it does not reinterpret wrapped results as negative.

Important boundaries are:

- Inputs below 0x00052000 produce unsigned subtraction underflow and a fourth argument between 0x0FFFFFFA and 0x0FFFFFFF, inclusive.
- Input 0x00052000 produces a fourth argument of zero.
- Every input from 0x00052000 through 0x00061FFF, inclusive, produces a fourth argument of zero.
- Input 0x00062000 produces a fourth argument of one.
- The maximum unsigned input, 0xFFFFFFFF, produces a fourth argument of 0x0000FFFA.

No visible guard rejects low inputs, clamps the wrapped result, or selects a separate error path.

The stored value and the fourth argument are separate derivations of the original unsigned input. Neither uses the value written by the other.

### Global-read ordering and aliasing

The global read supplying D-01’s first argument follows both destination stores in the represented sequence. The contract therefore does not permit substituting a value captured at function entry.

If either destination store overlaps storage consumed by the global read, the earlier write can affect the first argument. Determining the exact resulting value requires information absent here: the global read’s width and type, byte order, and any relevant conversion rules.

In particular, the global read must not be modeled as an established four-byte access. Its accessed extent is unresolved.

The indirect target’s acquisition mechanism and timing are also unresolved. The known ordering of the data read does not establish how or when the transfer destination is obtained.

### Indirect transfer and continuation

D-01 is invoked after the visible stores and with the argument values described above. The supplied representation models the indirect transfer as a call, but its exact machine-level control-flow behavior is not established.

If D-01 returns to the visible continuation, no additional explicit data-memory update occurs before F-5a0ff583ba4e returns.

If D-01 does not return, faults, or transfers control elsewhere, the supplied evidence establishes no alternate completion path or recovery action.

## Side effects

### Directly established effects

The visible prefix:

1. Writes 32 bits at base offset 0x38.
2. Writes 32 bits at base offset zero.
3. Reads global storage at 0x00000054 with unresolved width and type.
4. Transfers control to D-01.

This list describes explicit data accesses and control flow in the supplied representation. It does not bound accesses performed by calling machinery, indirect-target resolution, or D-01 itself.

There is no visible rollback. If the first store completes but the second store fails, the first store is not visibly undone. Likewise, a failure during the subsequent global read or indirect invocation occurs after both stores in the represented sequence.

The store widths do not establish hardware atomicity, synchronization, memory barriers, or guarantees for concurrent observers.

### Ownership and lifetime

No allocation, deallocation, retention, or ownership transfer is explicitly performed before the indirect invocation. The function writes directly into caller-designated storage without visible bounds or lifetime validation.

The base address is not one of D-01’s four explicit arguments. That does not prove D-01 cannot access the destination: it may have another alias or access through globals or machine state.

Neither continued caller ownership nor preservation of the allocation after D-01 is established. The dependency’s memory-management behavior is unknown.

### Effects beyond the visible prefix

No preservation guarantee is available for the destination fields, the global storage, other memory, or resources once D-01 executes.

Consequently, the contract distinguishes values written by the prefix from values observable at function return. A returning dependency may have changed either field or invalidated the storage entirely.

## Dependencies

### D-01 — unresolved indirect dependency

D-01 is an opaque contract-local identifier, not a recovered implementation identity.

- **Invocation point:** After both destination stores and acquisition of the global-value argument.
- **First explicit argument:** Value read from global storage at 0x00000054; read width, type, and argument conversion are unknown.
- **Second and third explicit arguments:** Zero values.
- **Fourth explicit argument:** The unsigned 32-bit arithmetic result described in Behavior.
- **Return value:** No result is visibly consumed.
- **Target-selection mechanism:** Unknown.
- **Machine-level transfer convention:** Unresolved beyond the supplied call-like representation.
- **Memory and global preservation:** Unknown.
- **Ownership, allocation, and release behavior:** Unknown.
- **Error and non-return behavior:** Unknown.

No resolved dependency implementation is supplied. The dependency mapping identifies the subject function but does not identify D-01.

## Acceptance examples

These are conditional static acceptance criteria, not execution results. They assume valid destination accesses, a successful global read under its actual unresolved access requirements, and an observation point at entry to D-01.

| Unsigned input | Value written at offset 0x38 | Fourth argument to D-01 |
|---|---:|---:|
| 0x00000000 | 0x00000000 | 0x0FFFFFFA |
| 0x00051000 | 0x0000000A | 0x0FFFFFFF |
| 0x00052000 | 0x0000000A | 0x00000000 |
| 0x00061FFF | 0x0000000C | 0x00000000 |
| 0x00062000 | 0x0000000C | 0x00000001 |
| 0xFFFFFFFF | 0x0001FFFF | 0x0000FFFA |

For every row:

- The offset-0x38 store occurs first.
- The offset-zero store then writes the 32-bit base-address representation.
- D-01 receives zero in its second and third explicit argument positions.
- Its first argument comes from the global read after both stores, without an assumed read width or type.

Additional conditional cases:

**Non-overlapping ordinary storage.** If the base address is 0x00100000 and the unsigned input is 0x00062000, the prefix writes 0x0000000C at 0x00100038, then 0x00100000 at 0x00100000. D-01’s fourth argument is one. Its first argument remains unspecified numerically without the global’s value and interpretation.

**Direct global-location alias.** If the base address is 0x00000054 and both stores are valid, the second store writes the base-address representation starting at the established global location before the global read. The first argument must reflect the subsequent read according to its actual width and type. This evidence does not justify asserting that the argument equals 0x00000054.

**Different first and second inputs.** Holding other relevant state fixed, changing those inputs does not alter the explicit arithmetic, stores, or argument expressions. Equivalent downstream execution through D-01 is not guaranteed.

**Non-returning dependency.** The two stores precede entry to D-01. If D-01 never returns, normal completion of F-5a0ff583ba4e is not required.

**Dependency modifies the destination.** A return-time field value different from the prefix’s written value does not by itself violate this contract. D-01 has no established preservation obligation.

**Invalid destination or global access.** No particular error code, recovery action, or rollback is required by the supplied evidence. Such execution falls outside the stated normal-completion preconditions.

## Uncertainties

The following remain **documented_only**:

- The global read’s width, type, accessed extent, and argument conversion.
- D-01’s identity, complete interface, and implementation.
- The indirect target’s acquisition mechanism and timing.
- The actual ABI, register assignments, and visibility of incoming machine state.
- Whether the indirect transfer behaves exactly like an ordinary call.
- The semantic meaning of the destination fields and global value.
- The valid input domain beyond the represented widths.
- Whether low-input unsigned wraparound is intended application behavior.
- Object size, allocation provenance, byte order, alignment, and lifetime.
- Target-specific address arithmetic and fault behavior.
- Whether aliasing or concurrent access occurs in valid use.
- Hardware atomicity and synchronization properties.
- D-01’s preservation, release, error, and return behavior.

The representation supports 32-bit destination stores and unsigned arithmetic. Those facts do not establish a 32-bit type for every referenced global or every argument position.

## Evidence

**Established static evidence:** The supplied body exposes four inputs, two ordered 32-bit destination stores, two unsigned shift-based derivations, a global-value read associated with address 0x00000054, two zero arguments, an unresolved indirect invocation, and a return continuation.

The global reference has no supplied declaration or explicit read-width conversion. The overlap warning does not resolve its width or type. Its location and placement after the two stores are established; its accessed extent is not.

**Inferred intent:** The destination may be a record containing a self-reference, and the fourth dependency argument may select a category or index. Neither interpretation is required for the observable contract.

**Unknown:** The dependency’s implementation, actual transfer convention, and execution effects are unavailable. No preservation or ownership guarantees are inferred from that absence.

No runtime tests have been performed. The acceptance values are arithmetic consequences of the supplied static representation, not observed runtime output.
