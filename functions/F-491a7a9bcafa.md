<!-- Trusted provenance: F-491a7a9bcafa; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-491a7a9bcafa

## Purpose

F-491a7a9bcafa conditionally operates on a zero-terminated byte sequence associated with an object. It identifies two positions by repeatedly invoking a cursor-advancement dependency, passes those positions and a computed count to a transfer dependency, and then invokes an object-dispatched operation with a second computed value and the constant `1`.

The apparent intent is removal of a range measured in cursor-advancement steps, followed by adjustment of object metadata. That interpretation is **inferred intent**, not a complete established contract: the advancement, transfer, initial query, and object-dispatched dependencies are not defined by the supplied evidence.

The established behavior is the gating, call order, byte checks, cursor-slot handling, and arithmetic described below. In particular, the function does not independently establish the units of the initial query result or the semantics of the transfer operation.

## Interface

### Inputs

The function accepts four arguments:

| Argument | Interpretation established by the supplied representation |
|---|---|
| Object | Address of an object whose first two fields are accessed as integer-sized words. The first word participates in indirect dispatch; the second is treated as a sequence address. |
| Start quantity | A pointer-shaped argument interpreted as a signed integer for comparisons and loop bounds. It is not dereferenced by this function. |
| Extent quantity | A pointer-shaped argument interpreted as a signed integer for comparisons and loop bounds. It is not dereferenced by this function. |
| Auxiliary value | A pointer-shaped value assigned to a temporary cursor slot before the initial dependency call. It is subsequently overwritten before traversal. No direct dereference of this argument is shown. |

The second and third arguments are therefore not ordinary byte-address inputs despite their pointer-shaped presentation. Their relevant behavior is determined by their signed integer interpretations.

### Widths and layout

The supplied representation supports the following target-level model:

- Object fields are accessed as 32-bit integer words.
- The dispatch-table reference occupies the object word at byte offset `0`.
- The sequence-address word occupies the object word at byte offset `4`.
- The dispatched operation is selected through a slot at byte offset `8` from the dispatch-table address.
- Cursor termination checks read one byte.
- Loop counters, the initial dependency result, and the shown subtraction expressions use signed integer-sized values, represented here as 32 bits.
- Sequence addresses are converted to that integer width for subtraction.

This is a target-specific contract. It must not be generalized to a platform with wider native pointers by preserving only the displayed argument types.

The exact signedness of a nonzero character does not affect termination: the test distinguishes a zero byte from every nonzero byte.

### Output

The function has no explicit result value. Its observable results consist of dependency calls and whatever memory or object changes those dependencies perform.

There is no returned success indicator, removed-count result, or error status.

### Preconditions

For a defined, normally completing execution:

- The object must be valid for every object-field read that actually occurs. There is no null-object check.
- If execution reaches traversal, each cursor must designate a readable byte whenever the function tests it for zero.
- Advancement dependencies must leave cursor slots in a state usable by subsequent operations.
- The transfer dependency must accept the supplied addresses and computed count.
- If execution reaches indirect dispatch, the object’s then-current dispatch reference and its selected slot must be valid.
- The arithmetic must be meaningful under the target’s actual integer and address rules.

These conditions are not validated comprehensively by F-491a7a9bcafa.

## Behavior

### 1. Initial eligibility checks

The function first reads the object’s sequence-address word.

If that word is zero, it returns without calling any dependency. The start quantity, extent quantity, and auxiliary value do not cause further work on this path.

If the word is nonzero, the function checks the signed interpretations of the two quantity arguments. The body is entered exactly when:

- the start quantity is nonnegative; and
- the extent quantity is strictly positive.

The supplied expression performs a selection and a nonzero test before completing the signed checks, but its accepted input set is equivalent to those two conditions. No argument-address dereference occurs during that selection.

Consequences include:

- A start quantity of zero is accepted.
- An extent quantity of zero causes an immediate return.
- A negative value for either quantity causes an immediate return.
- A positive extent does not guarantee that any cursor step will occur; the sequence may already begin with a zero byte.

On all rejected paths, the function makes no dependency calls and performs no direct writes to the object or sequence.

### 2. Initial dependency and sequence-address refresh

For an accepted request, two local cursor slots are initially populated with the extent argument and auxiliary argument. The function then calls F-8810f53aaa0b.

No explicit arguments are shown for this dependency call. Its integer result is retained for the two later count calculations.

The temporary slot initializations are established local assignments, but the supplied evidence does not establish whether the initial dependency can observe them through a calling convention, a saved alias, or any other mechanism. They do not by themselves establish an argument interface for that dependency.

After the call returns, the function reads the object’s sequence-address word again and uses that value to initialize the first traversal cursor.

This reread is significant. The sequence address used for traversal need not equal the nonzero value observed by the initial eligibility check. There is no second null-address check before the first traversal-byte read.

The function must not be specified as taking one immutable sequence-base snapshot at entry.

### 3. Locating the first position

The first traversal uses a counter beginning at zero.

While both of the following remain true, the function calls F-d3922c7fe0ec with the address of the first cursor slot:

- the counter is less than the signed start quantity; and
- the byte at the current cursor is nonzero.

After each normally returning advancement call, the counter increases by one.

The count comparison is evaluated before the byte check. Thus, when the start quantity is zero, this traversal does not read the sequence byte and makes no advancement call.

For a positive start quantity, encountering a zero byte ends this traversal early. The function does not require that the requested number of steps be available.

A step means one call to F-d3922c7fe0ec. The supplied evidence does not establish that a step advances by one byte, one encoded character, or any other fixed unit. The cursor slot’s resulting address is supplied by the dependency’s effects.

### 4. Locating the second position

After the first traversal, the function copies the current first cursor into a separate second cursor slot and resets the traversal counter to zero.

The second traversal uses the extent quantity as its bound. It repeatedly:

- checks whether the counter is below that bound;
- if so, reads the byte at the second cursor;
- if the byte is nonzero, calls F-d3922c7fe0ec with the address of the second cursor slot; and
- increments the counter after the call returns.

The first cursor slot is not the slot explicitly passed during this traversal.

The second traversal therefore starts at the position reached by the first traversal, rather than at the object’s sequence base. It stops at either its requested call count or a zero byte.

Because an accepted extent is positive, the second traversal attempts a byte check on its first condition evaluation. This remains true when the first traversal already stopped at a terminator.

The function does not validate that cursor advancement is monotonic, bounded, or terminating at a valid sequence position.

### 5. Transfer call

After both traversals, the function invokes D-01, the transfer dependency.

To state its arguments precisely, define:

- **Q**: the retained integer result from F-8810f53aaa0b;
- **S**: the first cursor value supplied to the transfer call;
- **E**: the second cursor value supplied to the transfer call;
- **B**: the object’s sequence-address word read for this transfer-count calculation.

The transfer receives, in order:

1. S;
2. E;
3. Q minus the integer address difference between E and B.

The third argument is therefore:

**Q − (E − B)**

The address difference is integer subtraction after address conversion, not established typed element-distance arithmetic.

The function does not directly check:

- that E is at or beyond B;
- that E is at or beyond S;
- that the computed count is nonnegative;
- that the referenced ranges are valid; or
- that any required terminator is included.

B is read from the object for this calculation after the advancement calls. It is not necessarily the sequence address used to initialize traversal.

### 6. Object-dispatched operation

After D-01 returns normally, the function performs an indirect object call through D-02.

The dispatch target is obtained from the object’s dispatch reference and the slot at byte offset `8`. The explicit arguments are:

1. the object;
2. Q minus the integer address difference between the second and first cursor values used in this calculation; and
3. the integer constant `1`.

The computed second argument is:

**Q − (E − S)**

These cursor values are read for the later call. The evidence does not establish that earlier dependencies cannot have retained aliases to the cursor slots.

The dispatch reference is accessed after the transfer operation returns. The function does not establish that D-01 preserves that reference or any other object field.

After the dispatched operation returns, F-491a7a9bcafa returns without inspecting a result.

## Side effects

### Direct reads and writes

The function directly:

- reads the object’s sequence-address word for initial eligibility;
- reads it again after F-8810f53aaa0b to initialize traversal;
- reads bytes at the active cursor positions while evaluating traversal conditions;
- reads the sequence-address word for the transfer-count calculation;
- reads the dispatch reference and selected dispatch slot; and
- writes its local cursor slots and counters.

No direct sequence-byte write or object-field write is shown.

That does not make the operation read-only. Every dependency may have side effects not specified here.

### Observable ordering

For an accepted, normally completing request, the established broad order is:

1. Initial sequence-address eligibility read.
2. Initial dependency call.
3. Refreshed sequence-address read.
4. First traversal’s byte checks and advancement calls.
5. Second traversal’s byte checks and advancement calls.
6. Transfer-count calculation and transfer call.
7. Indirect dispatch and object call.
8. Return.

Within each traversal condition, the counter bound is checked before reading the byte. A zero-byte observation prevents an advancement call for that iteration.

The evidence does not establish an ordering among individual argument-expression evaluations within a single call.

### Ownership and lifetime

No direct allocation or release operation is shown.

The function passes cursor values to D-01 and addresses of temporary cursor slots to F-d3922c7fe0ec. Passing those values does not establish ownership transfer.

Whether dependencies allocate, release, retain pointers, replace storage, mutate globals, or invalidate object data is **documented_only**. No preservation or ownership guarantee may be assumed from the caller alone.

### Errors

There is no explicit error-handling branch after execution enters the body.

The function does not inspect success results from advancement, transfer, or dispatch. Invalid addresses, invalid dispatch state, arithmetic problems, or dependency failures are not converted into a defined error return.

If a dependency does not return normally, later operations described here are not reached.

## Dependencies

| Opaque ID | Established use | Not established |
|---|---|---|
| F-8810f53aaa0b | Called once for an accepted request, before traversal. Its integer result participates in both later count calculations. No explicit arguments are shown. | Meaning and units of the result; hidden inputs; global or object effects; memory ownership effects. |
| F-d3922c7fe0ec | Called repeatedly with the address of one cursor slot. Each completed call counts as one traversal step. | Encoding rules; byte distance per step; validation; bounds behavior; side effects beyond the passed slot. |
| D-01 | Local opaque designation for the unmapped transfer dependency. Receives two cursor values and a computed count. | Whether it copies or moves bytes; overlap safety; count width conversion; zero-count behavior; return semantics. |
| D-02 | Local opaque designation for the runtime-selected object operation at dispatch-table byte offset `8`. Receives object, computed value, and `1`. | Meaning of the computed value and flag; target identity; storage-management behavior; invariants restored or required. |

D-01 and D-02 are document-local opaque identifiers, not additional identities supplied by the dependency map.

## Acceptance examples

These examples are conditional static expectations, not executed tests.

### Ineligible requests

For a valid object whose sequence-address word is zero, the function returns without invoking any dependency.

For a nonzero sequence-address word:

- start `−1`, extent `2`: no dependency calls;
- start `0`, extent `0`: no dependency calls;
- start `0`, extent `−1`: no dependency calls.

The auxiliary value does not change these outcomes.

### Zero start, positive extent

With start `0` and extent `2`, the function calls F-8810f53aaa0b and refreshes the sequence address.

The first traversal performs no byte read and no advancement call. The second traversal begins at that refreshed address and makes up to two advancement calls, stopping earlier if its current byte is zero.

Both subsequent calls still occur if all preceding operations return normally.

### Terminator reached before the requested start

Assume advancement moves to the next byte, preserves relevant object state, and encounters a zero byte after two calls. With start `5` and extent `3`:

- the first traversal makes two advancement calls;
- the second traversal observes the same terminator and makes none;
- the transfer receives equal cursor addresses; and
- the dispatched computed value equals Q, provided the cursor slots remain unchanged.

The function does not suppress the transfer or dispatch merely because no interval was traversed during the second phase.

### Conditional range-removal arithmetic

Assume:

- Q is `7`;
- the relevant sequence-base reads all produce the same address;
- each advancement moves exactly one byte;
- no terminator interrupts the requested traversal;
- dependencies preserve relevant cursor and object values;
- start is `2`; and
- extent is `3`.

Then S is base plus `2`, and E is base plus `5`.

D-01 receives a count of `2`. D-02 receives a computed value of `4`, followed by the constant `1`.

These observations support range-removal intent, but do not prove that the transferred count includes a terminator.

### Initial dependency changes the sequence address

If F-8810f53aaa0b replaces the object’s sequence address, traversal uses the replacement address.

If it replaces the address with zero, an accepted positive extent reaches a zero-address byte access unless earlier execution fails. The initial nonzero eligibility check does not protect that later access.

## Uncertainties

The following remain **documented_only**:

- The exact external ABI behind the pointer-shaped quantity arguments.
- Whether temporary-slot initialization before F-8810f53aaa0b reflects an interface detail absent from the shown call.
- The meaning, units, and terminator convention of Q.
- The advancement unit and encoding.
- Transfer overlap behavior and the formal type of its count argument.
- The meaning of the final constant `1`.
- Dependency effects on globals, object fields, storage lifetime, and escaped cursor-slot addresses.
- Behavior when address differences or subsequent subtraction exceed the signed integer range.

The shown arithmetic supplies no explicit overflow checks or saturation. Mathematical formulas in this document describe the expressions, not a promise of arbitrary-precision evaluation or portable wraparound.

## Evidence

**Established static evidence:** the supplied representation shows the eligibility checks, object-word accesses, two bounded traversal phases, one-byte termination checks, dependency-call order, indirect dispatch offset, and both subtraction expressions.

**Inferred intent:** range removal from an object-associated zero-terminated sequence, with a subsequent metadata or size update.

**Unknown:** dependency implementations, runtime invariants, complete ABI details, and fault behavior.

No runtime tests have been performed. The acceptance examples are conditional deductions from static evidence, not observed execution results.
