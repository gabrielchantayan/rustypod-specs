<!-- Trusted provenance: F-4ffd7bc0e22a; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-4ffd7bc0e22a

## Purpose

F-4ffd7bc0e22a conditionally performs two memory writes, calls dependencies around those writes, and returns an integer derived from a final evaluation of global selection state.

The modification path requires:

1. A clear least significant bit in the value initially read through a shared reference.
2. A nonzero value after the first dependency’s result has been assigned to a signed integer temporary.

On that path, the function writes one byte of zero, writes four bytes representing 100, and then calls the second dependency.

The two destination addresses are obtained differently:

- The one-byte destination uses a fresh selection value, adds the numeric constant 12, and then converts the addition result to the destination pointer type.
- The four-byte destination uses an earlier selection value already converted to a signed integer temporary, adds 8, and then converts that result to the destination pointer type.

**The first addition is not established to be a 12-byte displacement.** The selection’s declaration is unavailable. If it is pointer-typed, the addition is scaled according to its pointed-to type before conversion to the one-byte destination pointer. The width of the eventual write does not determine the units of the preceding addition.

A guarded initialization or synchronization pattern is plausible intent, but neither interpretation is established. The contract concerns the represented reads, conversions, branches, writes, calls, and return.

## Interface

### Explicit inputs and output

The function has no explicit arguments.

Its declared return type is a signed integer type. The supplied representation does not establish its concrete width, representation, or relationship to machine address width.

On normal completion, the function evaluates the selection state at the return site and converts that value as required by its return type. The result is not explicitly a success indicator, a pointer, or an owned handle.

The same return expression is reached whether the modification path is taken or skipped.

### Implicit inputs

Two global cells participate:

- **Shared-reference cell:** supplies the reference used for the initial indirect read and the argument expression at each dependency call.
- **Selection cell:** supplies the saved selection, the fresh selection used for the one-byte destination, and the final return expression.

Their declarations, types, widths, and qualifiers are unavailable. The initial indirectly read value’s complete access width is also unavailable. Testing its low bit does not establish a one-byte read.

The representation evaluates these global expressions at distinct points. That establishes source-level evaluation order, not necessarily distinct physical memory transactions or visibility to concurrent observers.

### Values subject to conversion

| Value | Established treatment |
|---|---|
| First dependency result | Assigned to a signed integer temporary before its zero/nonzero test |
| Selection captured after that call | Assigned to a separate temporary of the same signed integer type |
| Fresh selection for the one-byte write | Participates in addition of 12 before conversion to the destination pointer type |
| Saved selection for the four-byte write | Participates in integer addition of 8 before conversion to the destination pointer type |
| Final selection | Converted to the function’s declared signed integer return type |

The dependency result and selection values are not established to have the same original type as these integer destinations. Value preservation cannot be assumed.

### Modified storage

| Write | Address derivation | Width | Stored value |
|---|---|---:|---|
| First | Fresh selection, addition of 12 under the selection expression’s type rules, then destination-pointer conversion | 1 byte | Zero |
| Second | Saved signed integer selection, integer addition of 8, then destination-pointer conversion | 4 bytes | 100 |

For the first write, the numeric addend 12 is established; a displacement of 12 bytes is conditional on the missing type and address semantics.

For the second write, the arithmetic operand is established to be a signed integer temporary. This excludes pointer-scaled addition at that stage, but does not by itself establish that the converted destination is eight bytes beyond an address represented by the saved value.

The four-byte write begins at its resulting destination address. Its byte order is target-dependent. It is not four repetitions of the byte value 100.

These accesses do not establish a complete object layout, allocation size, field meaning, or common containing object.

## Behavior

### Initial gate

The function obtains the shared reference and reads the value it references. It tests whether the least significant bit is clear.

If that bit is set:

- Neither dependency is called.
- Neither explicit write occurs.
- No saved-selection assignment occurs.
- Execution proceeds to the final selection evaluation and return conversion.

If the bit is clear, the function obtains the shared reference for the first dependency’s argument and calls F-b255b26284e6.

The argument expression is evaluated separately from the initial gate expression. The representation does not establish that both evaluations necessarily obtain the same reference.

The gate is tested only once. A subsequent change to the referenced value does not trigger another gate test.

### First dependency and saved selection

After F-b255b26284e6 returns normally, its result is assigned to a signed integer temporary.

The function then evaluates the selection cell and assigns that value to a separate temporary of the same signed integer type.

Only after both assignments does it test whether the converted dependency result is nonzero. Therefore, the saved-selection evaluation and conversion occur even when the converted dependency result is zero.

If the converted result is zero, the function skips both writes and the second dependency, then proceeds to the return expression.

If it is nonzero, the function enters the modification path.

The original dependency result type is unavailable. Consequently, a nonzero original result is not sufficient, by itself, to establish entry to that path. The branch depends on the value produced by assignment to the integer temporary. A negative value in that temporary is nonzero and enables the path.

### First write: fresh selection

On the modification path, the function evaluates the selection cell again.

It adds 12 to this fresh selection value before converting the result to a pointer designating a one-byte destination. It then writes zero to that destination.

The order of addition and conversion matters:

- If the selection expression is integer-valued, the addition follows the applicable integer arithmetic rules. Interpreting the result as a byte-address displacement additionally requires appropriate integer-to-address semantics.
- If the selection expression is pointer-valued, the addition follows pointer arithmetic rules, including scaling by the pointed-to type and applicable validity constraints. Conversion to the one-byte destination pointer occurs afterward.
- The supplied evidence does not select between these cases.

Thus, “one-byte write after adding 12” is established. “One-byte write at byte offset 12” is established only under additional type and address assumptions.

### Second write: saved integer selection

After the one-byte write, the function uses the previously saved integer selection. It adds 8 to that signed integer value, converts the result to a pointer designating a four-byte destination, and writes 100 there.

This write does not obtain another selection value from the global cell.

The addition must not be modeled as arbitrary-precision arithmetic or assumed modular address wrapping. Values near the signed integer range boundary require target-specific treatment. The supplied representation contains no explicit overflow handling.

Even if the selection cell remains stable, the two destinations are not established to share an effective base. Conversion of the saved selection, differing arithmetic rules, and destination-pointer conversions can affect their relationship.

### Second dependency

After both writes complete, the function obtains the shared reference for F-d08d3ca17b81 and calls it.

No result from this call controls subsequent behavior.

Its argument is evaluated after the writes. Neither the earlier calls nor the writes are established to preserve the shared-reference cell.

### Return

On every normally completing path, the function evaluates the selection cell at the return site and applies the conversion required by its declared signed integer return type.

It does not return the saved selection merely because that value was captured earlier.

If F-d08d3ca17b81 changes the selection and returns normally, the return-site evaluation uses the resulting selection state. Writes that alias selection-related storage can likewise affect the eventual value.

Equality between the selection value and the returned integer requires a value-preserving return conversion. That equality is not established for arbitrary selection values.

## Side effects

The function’s explicit writes are limited to:

1. One byte containing zero.
2. Four bytes representing 100.

Both occur only on the enabled modification path, and the one-byte write precedes the four-byte write.

The first dependency can execute without either write occurring. The second dependency executes only after both writes complete.

The represented sequence is:

1. Initial indirect read and low-bit test.
2. First dependency argument evaluation and call, if enabled.
3. Assignment of its result to the integer temporary.
4. Selection evaluation and assignment to the saved integer temporary.
5. Zero/nonzero branch test.
6. Fresh selection evaluation, address calculation, and one-byte write, if enabled.
7. Saved-integer address calculation and four-byte write.
8. Second dependency argument evaluation and call.
9. Final selection evaluation and return conversion.

This sequence does not establish atomicity, memory barriers, physical transaction widths, or inter-thread visibility.

No explicit allocation, deallocation, ownership transfer, reference-count operation, rollback, retry, or error-reporting operation appears. The dependencies may have such effects, but their behavior is unknown. Neither dependency may be assumed to preserve globals, retain storage, release storage, or provide synchronization.

The write destinations are not proven disjoint from one another, the global cells, or the initially referenced memory. If they overlap, the later write can alter bytes written by the earlier one. No general adjacency or nonoverlap guarantee follows from the numeric constants alone.

Normal execution requires all reached reads, conversions, arithmetic operations, writes, and calls to be supported by the target environment. There are no explicit null, bounds, alignment, lifetime, or arithmetic checks.

If an access fails, a dependency does not return, or an operation otherwise prevents normal completion, no recovery behavior or particular return value is specified. Failure after the first write does not establish restoration of the previous contents.

## Dependencies

### F-b255b26284e6

Called once when the initial low bit is clear; otherwise not called.

Its argument expression obtains the shared reference at that call site. Its result is assigned to a signed integer temporary before the branch tests zero versus nonzero.

Its declaration, parameter conversion rules, original result type, side effects, ownership behavior, synchronization properties, and failure modes are unavailable.

Interpreting it as an acquisition or entry operation is inferred intent only.

### F-d08d3ca17b81

Called once only when the converted first-dependency result is nonzero and both writes complete.

Its argument expression obtains the shared reference after the writes. No result from this call is used.

It is not established to receive the same reference as the first dependency, to release anything, or to preserve selection state or memory lifetime.

## Acceptance examples

These are conditional deductions from static evidence, not executed tests. Each assumes normal completion of every reached operation.

### Initial low bit set

Given a readable gating value whose low bit is set:

- Neither dependency executes.
- Neither explicit write occurs.
- The saved-selection assignment is skipped.
- The final selection evaluation supplies the value converted for return.

If that return conversion preserves the value, the returned integer equals the final selection.

### Clear bit and converted dependency result zero

Given a clear initial low bit and a first-dependency result that becomes zero on assignment to the integer temporary:

- F-b255b26284e6 executes.
- The selection is evaluated and converted into the saved integer temporary.
- Both writes and F-d08d3ca17b81 are skipped.
- A subsequent selection evaluation supplies the return value.

This applies even to an originally nonzero dependency result if the actual conversion rules produce zero. It does not assert that any particular unknown result type permits that conversion.

### Clear bit and converted dependency result nonzero

Given valid operations and a first-dependency result that becomes a nonzero integer:

- The fresh selection participates in addition of 12, followed by destination-pointer conversion and a one-byte zero write.
- The saved integer selection participates in addition of 8, followed by destination-pointer conversion and a four-byte write of 100.
- F-d08d3ca17b81 executes after both writes.

A converted result of negative one satisfies the branch condition.

### Conditional byte-addressed layout

Assume all of the following:

- The selection remains stable.
- Its conversion to the saved integer preserves its value.
- The relevant arithmetic and pointer conversions interpret the numeric additions as byte-address displacements from the same effective base.
- Both writes are valid.

Then the four-byte write occupies byte positions 8 through 11 relative to that base, and the one-byte write occupies byte position 12. The one-byte write still occurs first.

Without these assumptions, this layout is not required.

### Conditional pointer-scaled fresh selection

Assume the fresh selection is a pointer to elements of size four bytes, pointer addition is valid, and conversion preserves the resulting address.

Then adding 12 advances the fresh-selection address by 48 bytes before the one-byte destination conversion. The zero write occurs at that resulting address, not necessarily 12 bytes from the original selection address.

This example illustrates a permitted conditional interpretation, not an established declaration of the selection cell. It does not determine the second destination.

### Second dependency changes selection

If both writes complete and F-d08d3ca17b81 changes the selection before returning, the final evaluation uses that changed selection, followed by the return conversion.

The returned integer need not identify either written destination.

## Uncertainties

The following remain **documented_only**:

- Concrete signed integer width and representation, address width, and byte order.
- Declarations and qualifiers of both global cells.
- Whether the selection expression is integer-typed or pointer-typed.
- Any pointed-to type and scaling applicable to fresh-selection addition.
- Width and signedness of the initial indirect read.
- Dependency parameter types and the first dependency’s original result type.
- Value preservation and out-of-range behavior of assignments and return conversion.
- Integer arithmetic limits, pointer arithmetic validity, and address-conversion semantics.
- Alignment, object bounds, lifetime, aliasing, and concurrency requirements.
- Dependency side effects, ownership behavior, synchronization, and failure behavior.
- Application-level meaning of the written locations.

Neither a 12-byte displacement for the first destination nor a common object containing both destinations is established without additional assumptions.

## Evidence

The contract derives solely from the supplied static function representation and opaque dependency mapping.

Established evidence includes the signed integer return type and two signed integer temporaries; the low-bit gate; the first dependency result assignment; the saved-selection assignment before the branch test; addition of 12 before conversion to the one-byte destination pointer; addition of 8 to the saved integer before conversion to the four-byte destination pointer; write order; second dependency placement; and final selection evaluation.

The missing selection declaration prevents an unconditional byte-displacement interpretation of the fresh-selection addition. The missing ABI, global declarations, and dependency interfaces also prevent stronger claims about conversions, ownership, synchronization, and address validity.

No runtime tests have been performed. All acceptance examples are conditional static deductions.
