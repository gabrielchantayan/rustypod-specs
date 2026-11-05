<!-- Trusted provenance: F-a8a62ade417f; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a8a62ade417f

## Purpose

F-a8a62ade417f conditionally dispatches an operation selected by one byte from an externally supplied buffer. For selectors 0 through 16, it reads an entry from the current dispatch table, derives an adjusted object address, resolves a callable, and invokes that callable with the adjusted address and a pointer to a local control byte.

A nonzero dispatch result is returned unchanged. A zero result requires the control byte to be within an inclusive range before F-dece790ddc92 is called with a local buffer and fixed arguments. If that dependency returns normally, this function returns zero.

Selectors 17 through 255 return 4 after the initial eligibility checks succeed. Eligibility failures and rejected control-byte values instead enter F-a817d8d58c10, which the supplied static representation identifies as nonreturning.

The dispatch structure suggests object-oriented operation selection, and the final dependency call suggests submission of a response. These are inferred intentions, not established object-model or protocol semantics.

## Interface

### Inputs

The function receives two arguments:

1. **Object address value:** A signed 32-bit value used in the calculation of the adjusted object address.
2. **Descriptor address:** A pointer to externally supplied storage containing the fields below.

| Descriptor byte offset | Access width | Established use |
|---|---:|---|
| 0 | 32 bits | Read as the second argument to F-dece790ddc92, only when that call is reached. |
| 8 | 32 bits | Read at entry and interpreted as the external byte-buffer address. |
| 12 | 16 bits | Compared with 1 during eligibility checking, provided the captured buffer address is nonzero. |

The complete descriptor layout is not established. The comparison at offset 12 concerns only the accessed 16-bit field, not an entire 32-bit word.

The function also reads a global dispatch-table base when a supported selector is processed. This base is not an explicit argument and is not captured before the eligibility helper runs.

The supplied representation uses 32-bit words and address values for these operations. It does not independently establish all target ABI or memory-access rules.

### Ordinary return value

The return value is a signed 32-bit integer:

- **4** for an eligible invocation whose selector is 17 through 255.
- **The exact nonzero dispatch result** for a supported selector whose callable returns nonzero.
- **0** when dispatch returns zero, the control byte is accepted, and F-dece790ddc92 returns normally.

There is no established ordinary return value for paths entering F-a817d8d58c10. An ordinary return also depends on all reached dependencies returning where expected and all reached memory accesses being valid.

### Preconditions

No general pointer-validity checks are present. Defined, normally completing execution requires valid storage and callable addresses for the path taken, including:

- A readable descriptor field at offset 8 before any buffer-address rejection.
- A readable 16-bit descriptor field at offset 12 when the captured buffer address is nonzero.
- A readable byte at the captured buffer address after eligibility succeeds.
- A readable current global table-base value and selected table entry for selectors 0 through 16.
- A valid resolved callable and any object-related storage needed for indirect resolution.
- A readable descriptor word at offset 0 if F-dece790ddc92 is reached.

A null descriptor is not explicitly rejected before dereference. A nonzero buffer address is not evidence of readable storage, and the function does not check a buffer length before reading its first byte.

## Behavior

### 1. Capture the buffer address and check eligibility

The first relevant external read obtains the 32-bit buffer-address field at descriptor offset 8. The function retains this address and establishes 4 as its default ordinary result.

Eligibility uses short-circuit evaluation in this order:

1. The captured buffer address must be nonzero.
2. The 16-bit descriptor field at offset 12 must equal 1.
3. F-5f5f14cedda3 is called with the descriptor address.
4. Its signed 32-bit result must be less than 3.

Failure at any stage enters F-a817d8d58c10 rather than returning 4.

The helper-result comparison has no lower bound. Negative results qualify; the accepted results are not limited to 0, 1, and 2.

The buffer address and offset-12 field are not reread after the helper. Consequently:

- Changing the descriptor’s buffer-address field during the helper does not redirect the subsequent selector read.
- Changing the byte at the originally captured address can change the selector.
- Changing the offset-12 field does not trigger another eligibility check.
- Changing the global dispatch-table base can affect later dispatch, because that global is read only after the helper returns and a supported selector is recognized.

No preservation guarantee for the helper’s effects on descriptor storage, buffer storage, global state, or other memory is established.

### 2. Read and classify the selector

After eligibility succeeds, the function reads one byte through the captured buffer address and stores it in the first byte of a 23-byte local buffer.

The selector is unsigned:

- **0 through 16:** Continue to table lookup.
- **17 through 255:** Return 4.

The local-buffer write occurs for both supported and unsupported selectors. Unsupported selectors do not cause a dispatch-table-base read, selected-entry read, callable invocation, or call to F-dece790ddc92.

No additional external-buffer read is explicitly present in this function.

### 3. Read the current table base and selected entry

For a supported selector, the function reads the current global dispatch-table base. This occurs after the eligibility helper has returned and after the selector has been read and copied locally.

The selected entry is located at the table base plus eight times the selector. The selector-dependent byte offset therefore ranges from 0 through 128.

Each entry comprises two 32-bit words, read in this order:

1. A value used either directly as the callable address or as an encoded offset for indirect lookup.
2. A packed value containing an object adjustment and a dispatch-mode bit.

The function uses the table base available at this point. It does not use a base saved before the helper. Thus, if the helper replaces the table-base global, lookup uses the replacement value.

The entry offset multiplication is bounded by 128. The behavior of a table-base address calculation that exceeds the target address range is not independently established.

### 4. Initialize control storage and resolve the callable

After reading both entry words, the function initializes a separate local one-byte control object to 1.

The second entry word has two roles:

- Its least significant bit chooses direct or indirect callable resolution.
- Its signed 32-bit interpretation, arithmetically shifted right by one bit, supplies the byte displacement added to the object address value.

Under that displayed signed-shift interpretation, the displacement ranges from −1,073,741,824 through 1,073,741,823. Negative odd encodings round downward during the shift, rather than following signed division’s truncation toward zero.

The addition of this displacement to the signed object address value forms the adjusted object address. Behavior if that signed addition overflows is not independently resolved by the supplied representation.

**Direct mode:** If the packed word’s least significant bit is clear, the first entry word is used as the callable address without clearing any of its bits. This resolution path does not read an object word through the adjusted address.

**Indirect mode:** If that bit is set:

1. A 32-bit word is read at the adjusted object address.
2. The bottom two bits of the first entry word are cleared.
3. The masked first word and the object word, after unsigned conversion, are added using unsigned 32-bit arithmetic.
4. The numerical sum wraps modulo \(2^{32}\).
5. That wrapped value is converted to an address, and the callable address is read there.

The modulo-\(2^{32}\) sum is established behavior of this indirect lookup calculation. It is not part of the unresolved signed object-adjustment arithmetic. Whether the resulting address is valid, aligned, readable, or meaningful as a callable-address location remains unknown.

Neither mode performs explicit null-address, alignment, table-integrity, or callable-validity checks.

### 5. Invoke dispatch and interpret its result

The resolved callable receives:

1. The adjusted object address.
2. A pointer to the local control byte, initially 1.

The callable’s result is treated as a signed 32-bit integer.

If the result is nonzero, it is returned unchanged. The control byte is not range-checked, and F-dece790ddc92 is not called.

If the result is zero, the function reads the control byte as unsigned and accepts only **2 through 19**, inclusive. Values 0, 1, and 20 through 255 enter F-a817d8d58c10.

A callable that returns zero without changing the initial control value therefore takes the nonreturning rejection path.

The callable is explicitly passed the control-byte address, not the local-buffer address. The representation does not establish permission to reach neighboring local storage through that pointer.

### 6. Call the final dependency

For a zero dispatch result and an accepted control byte, F-dece790ddc92 receives seven arguments:

| Position | Value |
|---|---|
| 1 | Descriptor address |
| 2 | Current 32-bit descriptor word at byte offset 0 |
| 3 | 0 |
| 4 | 3 |
| 5 | 13 |
| 6 | Address of the 23-byte local buffer |
| 7 | Accepted control-byte value, from 2 through 19 |

The descriptor word at offset 0 is read after dispatch. It can reflect mutations made by either earlier dependency.

The meanings of the fixed arguments and the seventh argument are not established. In particular, the seventh argument is not established to denote a byte count or an amount of storage accessed.

The return value of F-dece790ddc92 is not inspected. If it returns normally, the function returns the saved dispatch result, which is zero on this path.

## Side effects

### Relevant access and call order

The static representation establishes this path-dependent sequence:

1. Read the descriptor’s buffer-address field.
2. If that address is nonzero, read the 16-bit eligibility field.
3. If that field equals 1, call F-5f5f14cedda3.
4. If its signed result is below 3, read the selector through the captured address.
5. Write the selector to the first local-buffer byte.
6. For a supported selector, read the current global table base.
7. Calculate the selected entry address and read its first word, then its second word.
8. Initialize the local control byte to 1.
9. Form the adjusted object address.
10. In indirect mode, read the object word and then the callable address at the unsigned-wrapped lookup address.
11. Invoke the resolved callable.
12. If dispatch returns zero, read and validate the control byte.
13. If accepted, read the descriptor word at offset 0 and call F-dece790ddc92.

This is the ordering expressed by the supplied static representation, not an independent claim about instruction scheduling or concurrent-memory semantics.

The function shows no direct writes to the external descriptor, external buffer, table, or global table base. Dependencies may modify those or other locations; their preservation contracts are unavailable.

### Local storage, lifetime, and ownership

The function provides a 23-byte local buffer and a separately identified one-byte control object. Only the first buffer byte is explicitly initialized. The control byte is explicitly initialized to 1 before dispatch.

The maximum accepted seventh-argument value is 19, numerically smaller than the local buffer capacity of 23 bytes. This relationship establishes neither the argument’s meaning nor how much storage F-dece790ddc92 accesses. It also does not establish initialization of the remaining buffer bytes.

No allocation, deallocation, release, or explicit ownership transfer appears in this function. Pointers to local storage cease to designate live invocation-local objects when the invocation ends. Whether dependencies copy, retain, modify, consume, or otherwise use that storage requires separate contracts.

### Rejection and memory failures

F-a817d8d58c10 is reached for eligibility failures and invalid control-byte values following zero-result dispatch. It is marked nonreturning in the supplied representation.

Its concrete behavior is unknown. Termination, exceptions, diagnostics, cleanup, and resource release must not be assumed.

Invalid memory accesses or callable addresses can fail before this dependency is reached. No conversion of such failures to an ordinary status code is shown.

## Dependencies

| Opaque function ID | Established interaction | Unresolved contract |
|---|---|---|
| F-5f5f14cedda3 | Receives the descriptor after the initial field checks; a signed result below 3 permits continuation. | Result meaning, memory effects, global-state preservation, ownership, and exceptional behavior. |
| F-dece790ddc92 | Receives the seven arguments specified above after zero-result dispatch and control validation; its result is ignored. | Argument meanings, storage access extent, pointer retention, ownership, failure reporting, and other effects. |
| F-a817d8d58c10 | Entered for the specified rejections; annotated as nonreturning. | Concrete nonreturning mechanism and side effects. |

Selected dispatch targets are additional dependencies whose opaque identities are unavailable. Their established interface is an adjusted object address and a pointer to the initialized control byte, with a signed 32-bit result. No broader preservation or storage-access contract is available.

## Acceptance examples

These are conditional static criteria, not executed tests. Each assumes valid memory and suitable dependency behavior for every reached operation.

1. **Null captured buffer:** If the descriptor’s offset-8 field is zero, the offset-12 field is not read, the eligibility helper is not called, and execution enters F-a817d8d58c10.

2. **Wrong eligibility field:** With a nonzero captured address and an offset-12 field of 2, execution enters F-a817d8d58c10 without calling the helper or reading the selector.

3. **Signed helper boundary:** Helper results −1 and 2 permit selector processing. Result 3 rejects before the selector read.

4. **Unsupported selector:** An eligible selector of 17 or 255 is copied to the local buffer, then causes return value 4 without reading the table-base global.

5. **Captured buffer address:** If the helper changes the descriptor’s buffer-address field, the selector still comes from the originally captured address. If the helper changes the byte at that original address, the changed byte is used.

6. **Current table base:** If the helper replaces the table-base global and the selector is supported, the selected entry is read from the replacement table. For selector 16, the entry begins 128 bytes beyond that base.

7. **Indirect unsigned wrap:** Suppose the packed word selects indirect mode with zero displacement, the first entry word is hexadecimal 1013, and the object word is hexadecimal FFFFFFF0. Masking produces hexadecimal 1010; the unsigned sum wraps to hexadecimal 00001000. The callable address is read at that resulting address, assuming it is readable.

8. **Direct resolution:** With direct mode selected, the first entry word is used without the indirect-path mask. Whether that value denotes a valid callable remains a precondition.

9. **Nonzero result:** A callable returning −7 causes return value −7, regardless of the control-byte value it leaves. No final dependency call occurs.

10. **Unchanged control:** A callable returning zero while leaving the control byte at 1 causes entry into F-a817d8d58c10.

11. **Control boundaries:** After zero-result dispatch, values 2 and 19 permit the final call; values 0, 1, 20, and 255 reject. Accepted values are passed as the seventh argument without establishing count semantics.

12. **Late descriptor read:** If dispatch changes the descriptor word at offset 0, the final call receives the changed word. If that dependency returns normally, the function returns zero regardless of its returned value.

## Uncertainties

The following remain **documented_only**:

- Table contents, selected callable identities, and selector meanings.
- Full descriptor layout and protocol semantics.
- Meanings of the final dependency’s fixed arguments and seventh argument.
- Initialization or provenance of local-buffer bytes beyond the first.
- Physical adjacency of local objects and any dependency permission to access neighboring storage.
- Dependency mutation, preservation, ownership, retention, and release behavior.
- Behavior of overflowing signed object-address adjustment or unresolved table-base address arithmetic.
- Validity and consequences of accesses through invalid, misaligned, or otherwise unsuitable addresses, including an address produced by the established unsigned wrap.
- Target endianness, independent ABI details, and instruction-level realization of the displayed signed shift.
- The concrete effect of the nonreturning dependency.

The indirect lookup’s unsigned 32-bit addition and modulo-\(2^{32}\) wrap are established and are not included in the arithmetic uncertainties above.

## Evidence

**Established static evidence:** The supplied representation supports the descriptor access widths, short-circuit checks, signed helper comparison, captured buffer address, unsigned selector range, late global table-base read, ordered entry-word reads, packed adjustment and mode interpretation, unsigned-wrapped indirect lookup, control initialization and validation, dependency arguments, return paths, and nonreturning annotation described here.

**Inferred intent:** Adjusted-object dispatch and response submission are plausible interpretations of the structure. They do not establish a wider object model, protocol, storage-access extent, or ownership convention.

**Verification status:** Static analysis only. No runtime tests have been performed. Dependency implementations, table contents, and independent target-ABI evidence were not supplied. Acceptance examples are conditional consequences of the supplied representation, not observed executions.
