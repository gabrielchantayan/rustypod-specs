<!-- Trusted provenance: F-3e1cbf8608fb; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-3e1cbf8608fb

## Purpose

F-3e1cbf8608fb uses a signed index to select an address-bearing integer from storage associated with a global base. It adds **2,144** to the loaded signed integer, converts the resulting sum to a pointer, reads an unsigned word through that pointer, and returns the word’s lowest five bits.

Every successfully completed evaluation returns a value in the inclusive range **0 through 31**.

The order of arithmetic and conversion is significant: the displacement is added **before** the integer-to-pointer conversion. The operation is not established as pointer arithmetic on a pointer obtained directly from the loaded integer.

The supplied body does not establish what the index selects or what the returned bits mean. Interpretations such as an object category, status, or flags are inferred intent rather than established behavior.

## Interface

### Input

The function accepts one argument represented as a signed short integer.

Under the apparent model of a **16-bit signed short** and **32-bit signed integer**:

- The input range is −32,768 through 32,767.
- The input is promoted to a signed integer before multiplication by 396.
- Negative inputs remain negative during that multiplication.
- No explicit conversion to an unsigned index appears.
- No index-range check appears.

These widths are the model used for the numerical bounds below. The supplied body does not independently establish the target ABI.

### Output

The return value is represented as an unsigned integer. Under the apparent 32-bit model, it is a 32-bit unsigned scalar with bits 5 through 31 cleared.

The function returns neither a pointer nor a reference to the storage read. Zero is a normal result when the final word’s low five bits are all zero; it is not an established error indicator.

### Required environment

Successful execution requires:

- A usable base supplied by the referenced global datum.
- A readable signed-integer entry at the location selected by the input.
- Defined or otherwise target-established evaluation of the relevant arithmetic.
- An integer-to-pointer conversion that produces a usable pointer from the displaced integer sum.
- Readable unsigned-word storage at that converted pointer.
- Compatible alignment, representation, and storage lifetime.

Under the apparent 32-bit model, both the selected signed-integer entry and the final unsigned word occupy four bytes. Byte order and alignment are target properties, not independently specified by the body.

Allocation sizes, application-valid indices, ownership, and synchronization requirements are not supplied. The function does not check these conditions.

## Behavior

### Selecting the signed-integer entry

The input is multiplied by **396**, or **0x18c**, and the product participates in the expression selecting an entry relative to the global base.

Under the apparent byte-address interpretation, the selected location is the base plus a signed displacement of 396 bytes per index step. This spacing does not establish that complete 396-byte records exist or that every index in the argument’s representable range is valid.

For a 16-bit signed input promoted to a 32-bit signed integer:

- Input −32,768 produces a scaled displacement of **−12,976,128**.
- Input 32,767 produces a scaled displacement of **12,975,732**.
- Every possible input product fits in a signed 32-bit integer.

The multiplication therefore does not overflow under that model. This fact does not establish that adding the product to the base is valid or that the resulting entry is accessible.

The global datum’s declaration is absent. Consequently, the exact type and arithmetic rules of the base-selection expression are not independently established. Byte-displacement descriptions in this contract use the apparent address interpretation rather than asserting an independently verified global type.

No visible operation rejects, clamps, or normalizes a negative index. Under the byte-address interpretation, input −1 selects a location 396 bytes before the base, provided the target permits the calculation and the location is valid.

### Loading and displacing the integer

The selected entry is read as a signed integer. Under the apparent model, this is a signed 32-bit load.

The function then adds **2,144**, or **0x860**, to that loaded integer. This addition occurs in the integer domain, before conversion to the pointer used for the final read.

Under the apparent 32-bit model, the constant 2,144 is representable as a signed integer, so this is signed 32-bit addition. Its mathematical result is representable precisely when the loaded integer is at most **2,147,481,503**. The lower endpoint does not create underflow: adding 2,144 to −2,147,483,648 gives **−2,147,481,504**, which remains representable.

Accordingly:

- Loaded values from −2,147,483,648 through 2,147,481,503 admit a representable sum.
- Loaded values from 2,147,481,504 through 2,147,483,647 exceed the signed 32-bit maximum when increased by 2,144.
- Signed overflow can arise independently of whether any corresponding machine address would be mapped or readable.

For the overflowing range, the supplied material does not establish a normal result. It does not establish modulo wraparound, saturation, a particular fault, or a particular pointer value. Under ordinary signed-integer language semantics, overflow does not define a wrapped result; whether the actual target supplies different execution semantics remains unresolved.

This distinction prevents replacing the operation with “convert the loaded integer to a pointer, then advance that pointer by 2,144 bytes” without additional proof. Such a replacement changes the domain and potentially the behavior of the addition.

### Converting the sum and reading the final word

After the integer addition, the resulting sum is converted to a pointer to an unsigned integer. The function reads the unsigned integer through that pointer.

The source does not independently establish the mapping from signed integer values to pointers. In particular:

- A negative sum is not visibly rejected.
- No high-bit validation or address normalization appears.
- A representable sum is not necessarily a valid address.
- A readable address does not resolve an earlier signed-overflow condition.

Under a target model where representable integer sums map directly to byte addresses, the final read occurs at the address represented by the loaded integer plus 2,144. That interpretation is conditional on the arithmetic and conversion rules; it is not a guarantee of pointer arithmetic or address wraparound.

The indirection remains essential. The final read is not generally at the original base plus the indexed displacement plus 2,144. Its location depends on the integer read from the selected entry.

### Extracting the result

The final unsigned word is combined with **31**, or **0x1f**, using bitwise AND.

Therefore:

- Bits 0 through 4 determine the return value.
- All higher bits are excluded from the result.
- The retained bits are not shifted.
- The operation is not a Boolean conversion.
- The masked value is not written back.

Two completed evaluations that read final words with identical low five bits return the same value, regardless of differences in the upper bits.

### Logical read and evaluation order

The expression establishes this dependency order:

1. Obtain the base value used for selection.
2. Calculate the indexed location using the signed input and scale of 396.
3. Read the selected signed integer.
4. Add 2,144 to that integer.
5. Convert the integer sum to the pointer for the final read.
6. Read the unsigned word.
7. Mask the word with 31 and return the result.

The supplied material does not establish whether obtaining the base requires a distinct runtime memory read or how that global datum is represented.

This ordering describes value dependencies. It does not specify processor bus transactions, atomic accesses, barriers, or a snapshot across concurrent changes. No volatile or atomic qualification is supplied.

### Errors and invalid conditions

There is no visible error return, null check, bounds check, overflow check, recovery branch, or exception translation.

If entry selection is invalid, the selected read fails, the signed addition overflows, the integer-to-pointer conversion is unsuitable, or the final pointer is unusable, no successful result is established. These conditions need not have identical target behavior.

The contract does not assign a particular fault class or promise that execution reaches the final read. It also does not reinterpret such conditions as a return value of zero.

## Side effects

The visible computation performs memory reads and contains no explicit stores.

It does not visibly:

- Change the global base.
- Modify the selected entry.
- Modify the final word.
- Retain a pointer for later use.
- Allocate or release memory.
- Transfer ownership.
- Call another function.

The source therefore establishes no ownership operation. The surrounding environment must provide suitable storage lifetimes, but the responsible owner is unknown.

For ordinary readable memory, there is no visible state update beyond producing the return value. If the selected locations are read-sensitive device memory, reads could have external effects; that memory model is not established here.

No synchronization protects the base, selected entry, or final word. A coherent snapshot spanning them is not guaranteed.

## Dependencies

No function-call dependencies appear in the supplied body. There are therefore no additional opaque function IDs to enumerate.

Environmental dependencies are:

- The global datum used to obtain the base.
- The selected signed-integer storage.
- The unsigned-word storage reached after integer addition and pointer conversion.
- Integer widths and promotion rules.
- The type and arithmetic behavior of the base-selection expression.
- Signed-overflow semantics applicable to the target.
- Integer-to-pointer conversion rules.
- Byte order, alignment, readable mappings, and storage lifetime.

Byte order affects how stored bytes become the loaded signed integer and final unsigned word. The mask operates on the decoded numeric word, not on a separately specified byte encoding.

## Acceptance examples

These are conditional acceptance examples, not runtime observations. Unless stated otherwise, they assume the apparent 16/32-bit model, byte-address base selection, valid base arithmetic, readable selected entries, representable signed addition, and suitable integer-to-pointer conversion.

### Zero index and ordinary extraction

For input **0**, the selected signed-integer entry is at the base.

If that entry contains **1,048,576**, the integer addition produces **1,050,720**. If conversion of that sum yields a valid pointer and the unsigned word there is **0x1234567b**, the function returns **27**.

This example requires addition to the integer before conversion; it does not assume a pointer is first formed from 1,048,576.

### Positive index and a zero result

For input **2**, the selected entry is 792 bytes beyond the base under the byte-address interpretation.

If the entry contains **2,097,152**, the integer sum is **2,099,296**. If its converted pointer identifies a word containing **0x00000020**, the function returns **0**.

The set bit immediately above the retained field does not contribute to the result.

### Negative index

For input **−1**, the selected entry is 396 bytes before the base under the same interpretation.

If that entry is valid, contains **1,048,576**, and the word reached by converting the representable sum **1,050,720** contains **0xffffffff**, the return value is **31**.

This does not establish that negative indices are valid application inputs.

### Upper-bit independence

If two otherwise successful evaluations read final words **0x00000005** and **0xffffffe5**, both return **5**.

Both evaluations must independently satisfy the selection, signed-addition, conversion, and readability prerequisites.

### Signed-addition boundary

If the selected signed integer is **2,147,481,503**, adding 2,144 produces **2,147,483,647**, which is representable under the apparent model. A successful return still requires a suitable pointer conversion and readable final word.

If the selected integer is **2,147,481,504**, the mathematical sum is **2,147,483,648**, outside that signed range. No normal result or wrapped address is specified, even if a machine address with a corresponding bit pattern would be readable.

### Invalid selected storage

If the selected entry is unreadable, no normal return value is established. The body does not convert that failure into zero or another sentinel.

## Uncertainties

The following remain **documented_only**:

- Independently confirmed target ABI and scalar widths.
- The global datum’s declaration, value, and means of supplying the base.
- Exact arithmetic rules for combining that base with the scaled index.
- Application-valid index bounds and allocation boundaries.
- Required alignment and storage lifetime.
- Actual target behavior for signed overflow.
- Integer-to-pointer conversion, including negative sums.
- Whether memory is ordinary, shared, or read-sensitive.
- Concurrent-update behavior and consistency guarantees.
- Failure behavior for invalid calculations or inaccessible storage.
- Semantic meaning of the returned five bits.

An indexed collection containing address-bearing entries is a plausible interpretation. A field associated with referenced storage is also plausible. Neither interpretation establishes a collection type, record layout, ownership model, or application meaning.

The integer-addition-before-conversion ordering is established by the supplied expression; the target’s behavior when that addition overflows is not.

## Evidence

Established static evidence consists of the supplied single-function body:

- One signed-short argument.
- Index multiplication by 396.
- Selection relative to a global datum.
- A signed-integer read from the selected location.
- Addition of 2,144 to the loaded integer.
- Conversion of that sum to a pointer to an unsigned integer.
- An unsigned-word read through that pointer.
- Masking with 31 and an unsigned return value.
- No visible branches, calls, checks, or stores.

The numerical ranges and overflow boundary are arithmetic consequences of the explicitly stated apparent width model, not independent confirmation of that model.

No runtime tests, memory inspection, caller analysis, or implementation verification have been performed.
