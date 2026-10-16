<!-- Trusted provenance: F-9ed8931accde; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-9ed8931accde

## Purpose

F-9ed8931accde determines whether two caller-supplied integer objects contain equal values.

The established behavior is value equality, not address equality, identity testing, ordering, subtraction, or comparison of surrounding memory. Two different objects may compare equal. Two pointers to the same valid, stable integer object also compare equal.

The function produces a Boolean result. It does not return either input value, modify the compared objects, or transfer ownership.

This contract describes the supplied static evidence. No runtime tests have been performed. Target-specific integer widths, calling conventions, and generated memory-access instructions are not established by that evidence.

## Interface

### Inputs

The function accepts two pointer arguments. Each identifies an integer object whose value is used in the comparison.

For a defined invocation:

- Each pointer must designate an object that may legally be read as the declared integer type.
- Each object must remain alive and readable for the evaluation.
- Any alignment requirements of the target’s integer type must be satisfied.
- Each object must contain a value that is valid to read under the applicable language and platform rules.
- Concurrent access must not violate the applicable memory model.

There is no explicit null check, range check, lifetime check, alignment check, or other input validation. Supplying an invalid pointer is not a supported way to obtain a negative comparison result.

The arguments need not designate distinct objects. No non-aliasing restriction is present in the supplied declaration or body.

### Input width and layout

Each argument is a pointer to an integer object, not an integer passed by value, a byte sequence with an explicit length, or an array descriptor.

The source identifies the pointee type as the language’s ordinary signed integer type. Its exact bit width, storage size, representation, and alignment are not supplied. In particular, the evidence does not independently establish a 32-bit pointee width.

Let the target integer width be **W** bits. The comparison uses the integer values obtained from the two designated objects under that target’s rules. It does not explicitly narrow them to a smaller width, widen them to a separately declared fixed-width type, or perform arithmetic on them.

Each pointer designates the object to be compared directly. There is no field offset, element index, length field, terminator, or traversal. No second element or adjacent object is consulted.

Pointer width is likewise unspecified. Neither pointer width nor integer width should be inferred from the opaque function identifier.

### Output

The result is Boolean:

- **True** when the two read integer values are equal.
- **False** when they are unequal.

The source establishes the logical result, not the ABI-level size or register representation used to return it. A consumer must not assume that the returned Boolean occupies the same width as either compared integer.

## Behavior

### Value acquisition

The function’s expression uses the integer value designated by the first argument and the integer value designated by the second argument.

At the source-expression level, each operand contains one dereference of its corresponding argument. There is no explicit repeated sampling, preliminary validation read, or subsequent confirmation read.

This does not establish an exact number of physical memory transactions. Compilation may combine, eliminate, or otherwise transform accesses where the language rules permit. The source also does not request volatile or atomic access.

### Read ordering

The supplied expression does not impose a first-argument-before-second-argument read order. It likewise does not impose the reverse order.

A behavioral consumer must therefore not depend on either object being read first. The function does not establish a synchronized snapshot of two independently changing locations.

For ordinary valid objects that remain unchanged during evaluation, this ordering freedom does not affect the result. For externally changing storage, the supplied evidence does not establish a sampling protocol or consistency guarantee. Unsynchronized concurrent modification may also violate the applicable language memory model.

### Comparison

After obtaining the operand values, the function evaluates integer equality and returns that comparison’s Boolean result.

The following distinctions matter:

- The addresses themselves are not compared for equality.
- Equal values at different addresses produce true.
- Unequal values produce false regardless of any relationship between the addresses.
- Negative values are compared as integer values; they are not rejected.
- Zero has no special treatment.
- The function does not calculate a difference, so its comparison does not introduce subtraction overflow.
- It does not compare the byte representations of the objects. Padding or representation details are not an independently specified comparison criterion.

Because both operands have the same declared integer type, the supplied expression does not introduce a mixed-type signed-versus-unsigned comparison between them.

### Completion and invalid inputs

For valid inputs whose reads complete normally, the function returns the comparison result. Its body contains no loop, retry, allocation, dependency call, or explicit failure branch.

The function defines no separate error result. False means that the compared values differ; it does not mean that either pointer was invalid or that a read failed.

Behavior following an invalid dereference is outside this contract’s defined invocation domain. The evidence does not promise an exception, a particular fault, process termination, a diagnostic, or a recoverable false result.

## Side effects

For a defined invocation, the supplied body performs no explicit writes.

In particular, it does not:

- Modify either compared integer.
- Replace either caller-held pointer.
- Update a global or static object.
- Allocate or release memory.
- Retain an argument for later use.
- Emit output or report diagnostics.
- Acquire synchronization or publish a state change.

Ownership remains with the caller or whichever party already owns the objects. The function borrows read access only for its evaluation. There is no evidence of ownership transfer or deferred access.

The lack of explicit writes does not make arbitrary storage safe to read. Device-backed storage, volatile objects, atomic objects, and concurrently modified memory have access rules that this interface does not establish. No device-side effects or hardware transaction semantics can be inferred from this body.

## Dependencies

No dependency function is called in the supplied body. There are therefore no dependency function IDs to enumerate.

The operation relies on the target’s rules for:

- Pointer validity and object lifetime.
- Reading the declared integer type.
- Integer equality.
- Boolean return values.
- Calling and return conventions.

These are platform and language prerequisites, not evidence of additional callable dependencies. No unknown callee is involved whose ownership behavior or global-state effects would need to be assumed.

## Acceptance examples

These examples are conditional contract cases, not reported test results. Unless stated otherwise, both arguments designate valid, readable integer objects that remain unchanged during evaluation.

| Condition | Required result or boundary |
|---|---|
| Two distinct objects both contain 7. | Return true. Distinct addresses do not prevent equality. |
| One object contains 7 and the other contains 8. | Return false. |
| Both objects contain zero. | Return true; zero is an ordinary comparable value. |
| Both objects contain −1. | Return true. |
| One object contains −1 and the other contains 1. | Return false. |
| Both arguments designate the same stable, valid object. | Return true, regardless of that object’s valid integer value. |
| Both objects contain the minimum representable value of the target integer type. | Return true, without requiring arithmetic on that minimum. |
| One object contains the target type’s minimum and the other its maximum. | Return false, without computing their difference. |
| The designated integer values are equal, but adjacent objects contain different data. | Return true; adjacent objects are not comparison inputs. |
| Either argument is null or otherwise invalid to dereference. | No defined Boolean outcome is specified. |
| Another execution context changes an operand without sufficient synchronization. | No reliable comparison or snapshot guarantee is established. |

On a target independently established to use 32-bit signed integers, the same rules apply across that target’s valid integer range. That conditional example does not establish that this function’s actual target uses 32-bit integers.

An implementation satisfying this contract must preserve the value-equality result for valid, stable inputs. It need not preserve a particular instruction sequence, register assignment, or physical read count.

## Uncertainties

The following items are **documented_only** because the supplied evidence does not resolve them:

- Exact integer width, storage size, alignment, and representation.
- Pointer width and calling convention.
- Boolean return width and ABI encoding.
- Which operand is evaluated first.
- Generated instructions and exact physical memory-access count.
- Target-specific consequences of invalid pointers or unreadable storage.
- The surrounding caller’s synchronization, ownership, or lifetime discipline.
- Any intended use with special memory regions or externally changing storage.

**Inferred intent:** the function is a small predicate for equality of two referenced integers. This interpretation agrees with the body, but no caller context establishes a broader role, such as a container comparator or application-specific identity check.

Nothing in this contract requires guessing those broader uses or replacing unknown platform properties with customary defaults.

## Evidence

**Established static evidence:** the supplied function accepts two pointers to the same integer type and returns the equality comparison of their dereferenced values. Its body contains no additional statements, calls, writes, validation, or error handling.

**Derived contract consequences:** under valid-read preconditions, equality of the two values determines the Boolean result; pointer identity and neighboring storage do not determine it. No arithmetic operation introduces an overflow boundary.

**Evidence limits:** only the supplied source fragment was available. No implementation environment, caller behavior, generated binary, ABI description, or runtime observation was provided. No runtime tests have been performed.
