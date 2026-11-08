<!-- Trusted provenance: F-afa700fbb3bf; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-afa700fbb3bf

## Purpose

F-afa700fbb3bf selects behavior from a record’s 32-bit type value and conditionally invokes operations associated with a mutable receiver.

Two type values are recognized:

- **0x0F:** performs gated processing and optional follow-up work. Every normal return from this path is **1**, including when processing is skipped.
- **0x13:** evaluates a primary match, then a secondary match if necessary. A successful match invokes its associated operation and returns **1**; otherwise the result is **0**.

All other type values return **0**, subject to the separate type reads described below.

These return and control-flow properties are established by the supplied static body. Interpreting **1** as “handled,” or assigning application-specific meanings to the types, remains inferred intent.

## Interface

### Arguments and result

The function receives two address-bearing arguments:

| Argument | Required storage |
|---|---|
| Receiver | A mutable object containing a dispatch-table address and the fields listed below. |
| Record | Storage containing a 32-bit type at byte offset **0x04** and, when reached, 32-bit comparison values at offsets **0x08** and **0x0C**. |

The represented interface uses **32-bit words and addresses**. All offsets in this contract are byte offsets.

The result is a **32-bit value**, explicitly **0** or **1** on every visible normal return. Dependency results affect branching or subsequent arguments but are not directly returned.

No null, bounds, alignment, dispatch-target, or address-overflow checks are present. Every reached memory access and indirect call requires valid storage or a callable target. These are caller/environment preconditions, not guarantees established by this function.

### Receiver storage

This is an accessed-field description, not a complete object layout.

| Offset | Represented width or interpretation | Use |
|---|---|---|
| **0x00** | 32-bit dispatch-table address | Resolves receiver operations. |
| **0x34** | 32-bit address | Supplies the base for a nested read at offset **0x34**. |
| **0x40** | 32-bit value | Primary comparison against record offset **0x08**. |
| **0x44** | 32-bit value | Primary comparison against record offset **0x0C**. |
| **0x80** | 32-bit value | Third argument to F-d156b0b83eb3. |
| **0x84** | 32-bit value | Second argument to F-d156b0b83eb3. |
| **0x98** | 32-bit value | Secondary comparison against record offset **0x0C**. |
| **0x9C** | 32-bit value | Enables secondary matching when nonzero. |
| **0x15C** | 32-bit value | Enables an optional receiver operation when nonzero. |
| **0x170** | 32-bit bit field | Mask **0x10** selects the type-0x0F processing branch. |
| **0x174** | Word-sized field converted to one byte | Zero/nonzero eligibility test. |
| **0x17C** | Address of embedded storage | Passed to two follow-up dependencies; extent unknown. |

The eligibility condition converts the field at **0x174** to a one-byte value before testing it. It is not a nonzero test of the entire 32-bit field. A word whose low eight bits are zero does not satisfy this condition merely because higher bits are nonzero. The source representation does not establish a particular machine-load width for this conversion.

Two distinct **16-byte local scratch regions** and one addressable **32-bit local scalar** are also used. The scratch regions have no explicit initialization or other explicit writes in the supplied body.

### Receiver operations

Indirect operations are identified by dispatch-table byte offset because no opaque function IDs are supplied for their targets.

| Table offset | Arguments supplied | Result consumed |
|---|---|---|
| **0x128** | Receiver | 32-bit zero/nonzero eligibility result. |
| **0x11C** | Receiver, secondary scratch address, primary scratch address, address of the local 32-bit scalar | None. |
| **0x12C** | Receiver | 32-bit value used for follow-up gating and as a later argument. |
| **0xD4** | Receiver | None. |
| **0xA0** | Receiver, one-byte result from F-5710a8628cba | None. |

The table is reached through the receiver’s dispatch-table field at each call site. No preservation guarantee permits treating all targets as belonging to an immutable table captured on entry.

## Behavior

### Type selection

The first record-field access represented in the body reads the 32-bit type at **record + 0x04** and compares it with **0x0F**.

If equal, the type-0x0F path executes. Neither record comparison field is then used.

If unequal, the body reads **record + 0x04** again and compares that value with **0x13**. The two comparisons are therefore represented as separate reads, not as comparisons against one explicitly retained value.

If the second comparison fails, the result is **0**, with no dependency calls.

This describes the supplied static access structure. It does not establish synchronization, volatile-access guarantees, or defined behavior under concurrent mutation.

### Type 0x0F: eligibility

The function first tests the one-byte conversion of the receiver field at **0x174**.

- A zero value skips all processing and returns **1**.
- A nonzero value causes a call to operation **0x128**.
- A zero result from that operation also skips all remaining processing and returns **1**.
- Only a nonzero converted field and a nonzero operation result allow the flag-selected branch and common follow-up to execute.

The flag field at **0x170** is read after operation **0x128** returns. Changes made by that operation can therefore affect branch selection.

### Type 0x0F: mask 0x10 clear

When the subsequent flag test finds mask **0x10** clear, processing occurs in this order:

1. The addressable 32-bit local scalar is assigned **1**.
2. F-412e4635fdb0 receives the receiver and the primary 16-byte scratch address.
3. The receiver’s 32-bit enable field at **0x15C** is read.
4. If that field is nonzero, operation **0x11C** receives the receiver, the distinct secondary scratch address, the primary scratch address, and the scalar’s address, in that order.
5. F-d156b0b83eb3 receives the primary scratch address, the 32-bit receiver value at **0x84**, and the 32-bit receiver value at **0x80**, in that order.
6. F-6d16652f2d03 receives the receiver and the primary scratch address.

There are **no explicit writes to either scratch region in this body**. Passing an address to a dependency does not establish that the dependency writes it, initializes it completely, or treats all 16 bytes as meaningful.

The assignment of **1** to the separate scalar is explicit. Operation **0x11C** can potentially modify that scalar through its supplied address, but the caller does not subsequently read its value. Likewise, the caller does not directly read or write the secondary scratch region after supplying it to that operation.

The primary scratch address is passed through the three direct dependencies and, when enabled, operation **0x11C**. Its contents and any transformations remain dependency behavior.

The enable field is read only after F-412e4635fdb0 returns. The values at **0x84** and **0x80** are obtained for F-d156b0b83eb3 after the optional operation. Earlier calls cannot be assumed to preserve these fields. The argument order is established; the relative evaluation order of the two field reads is not.

### Type 0x0F: mask 0x10 set

When mask **0x10** is set, the entire scratch-processing sequence is skipped.

The function reads the 32-bit address at **receiver + 0x34**, reads a 32-bit value at **that address + 0x34**, and passes that value to F-92bea7c30bd6.

Neither the nested address nor the resulting access is validated.

### Type 0x0F: common follow-up

After either flag-selected branch returns normally, operation **0x12C** is called with the receiver.

- If its result is zero, no further processing calls occur.
- If its result is nonzero, F-3fac671648b8 is called with the receiver.
- If F-3fac671648b8 returns zero, the final two calls are skipped.
- Otherwise, F-6904f7699031 receives **receiver + 0x17C** and the retained result from operation **0x12C**. F-ec4cfa552cce then receives **receiver + 0x17C**.

The result from operation **0x12C** is retained across F-3fac671648b8; it is not reacquired afterward. Both embedded-storage arguments designate the same offset within the original receiver.

Every normally completed type-0x0F path returns **1**, regardless of either follow-up gate.

### Type 0x13: primary match

The function reads the 32-bit value at **record + 0x0C** and retains it for matching.

It first compares that value with **receiver + 0x44**. If unequal, the remaining primary-match checks are skipped.

If equal, it reads the 32-bit values at **record + 0x08** and **receiver + 0x40**. The primary match succeeds if the record value equals either:

- the receiver value at **0x40**; or
- a distinct 32-bit global value, called the **primary global comparison value** here.

The primary global is read only when the record value differs from the receiver value at **0x40**.

On a primary match, operation **0xD4** executes with the receiver, followed by return value **1**. No secondary-match enable check occurs on that path.

### Type 0x13: secondary match

Only after primary matching fails does the function read **receiver + 0x9C**.

If that field is zero, the function returns **0**.

If nonzero, it compares the retained record value from offset **0x0C** with **receiver + 0x98**. A mismatch returns **0** without reading the secondary global.

If those values match, the function reads **record + 0x08** and a second, distinct 32-bit global value, called the **secondary global comparison value**.

If they differ, the function returns **0**.

If equal:

1. F-5710a8628cba is called with no explicit arguments.
2. Its one-byte result is supplied, with the receiver, to operation **0xA0**.
3. The function returns **1**.

If the primary comparison already read record offset **0x08**, the secondary path reads that field again rather than explicitly reusing the earlier value. Record offset **0x0C**, by contrast, is retained for both match paths.

Primary matching has strict precedence. A successful primary match suppresses all secondary matching and its action.

## Side effects

The supplied body contains **no explicit writes to receiver fields, record fields, global storage, or either 16-byte scratch region**.

Its explicit local state changes include scalar assignments for loaded values, dependency results, comparison state, and the addressable 32-bit scalar initialized to **1**. These assignments must be distinguished from possible writes performed by dependencies through supplied addresses.

Externally visible effects remain possible through every call:

- Receiver operations and direct dependencies may modify receiver fields or global state.
- Dependencies receiving scratch addresses may write through them, but neither writes nor initialization coverage are established.
- Dependencies receiving **receiver + 0x17C** may modify embedded receiver storage.
- Earlier calls may replace the receiver’s dispatch-table address or modify table contents used by later calls.
- Operation **0xA0** is resolved after F-5710a8628cba returns; that dependency cannot be assumed to preserve the receiver’s dispatch state or globals.

No explicit allocation, deallocation, reference-count operation, or ownership transfer appears in the body. This does not establish the absence of those effects inside dependencies.

The local scratch regions and addressable scalar belong to this invocation. Their addresses are supplied during calls. Whether dependencies retain those addresses is unknown; the caller provides no post-return lifetime guarantee.

There is no separate error-reporting channel. A zero gate result suppresses work but is not independently reported as an error. Invalid accesses, invalid call targets, nonreturning dependencies, and exceptional transfers have no visible recovery path here.

## Dependencies

| Opaque function ID | Established call-site contract |
|---|---|
| **F-412e4635fdb0** | Receives the receiver and primary 16-byte scratch address. Result unused. Writes and initialization unknown. |
| **F-d156b0b83eb3** | Receives the primary scratch address, then the 32-bit receiver values at **0x84** and **0x80**. Result unused. |
| **F-6d16652f2d03** | Receives the receiver and primary scratch address after F-d156b0b83eb3. Result unused. |
| **F-92bea7c30bd6** | Receives the nested 32-bit value obtained through receiver offset **0x34**. Result unused. |
| **F-3fac671648b8** | Receives the receiver. Its represented 32-bit zero/nonzero result gates the final two follow-up calls. |
| **F-6904f7699031** | Receives the embedded-storage address at **0x17C** and the retained nonzero 32-bit result from operation **0x12C**. |
| **F-ec4cfa552cce** | Receives that same embedded-storage address after F-6904f7699031 returns. |
| **F-5710a8628cba** | Receives no explicit arguments and supplies the one-byte value passed to operation **0xA0**. |

These descriptions establish arguments, consumed results, and ordering only. They do not establish ownership, global preservation, complete initialization, success semantics, or internal failure handling.

## Acceptance examples

These are conditional static acceptance scenarios, **not executed tests**. Unless stated otherwise, storage is valid, relevant values remain stable, and dependencies return normally.

1. **Unrecognized type.** If both type reads produce **0x22**, the result is **0** and no dependency is called.

2. **One-byte eligibility.** With type **0x0F** and the field at **0x174** equal to **0x00000100**, its one-byte conversion is zero. The result is **1**, with operation **0x128** and all processing skipped.

3. **Eligibility rejection.** With a nonzero eligibility conversion but operation **0x128** returning zero, neither flag-selected branch nor operation **0x12C** executes. The result remains **1**.

4. **Clear-mask path without optional operation.** If eligibility succeeds, mask **0x10** is clear, and **0x15C** is zero after F-412e4635fdb0, the sequence is F-412e4635fdb0, F-d156b0b83eb3, F-6d16652f2d03, then operation **0x12C**, followed by any enabled follow-up.

5. **Optional operation enabled by an earlier call.** If F-412e4635fdb0 changes **0x15C** from zero to nonzero, operation **0x11C** executes. It receives the two distinct scratch addresses and the address of a scalar assigned **1** beforehand. This scenario does not require or imply initialized scratch contents.

6. **Set-mask branch.** If eligibility succeeds and mask **0x10** is set, F-92bea7c30bd6 receives the nested value. Neither scratch address is passed to any dependency on this branch.

7. **Follow-up gates.** A zero result from operation **0x12C** suppresses all three follow-up dependencies. A nonzero result followed by zero from F-3fac671648b8 suppresses only F-6904f7699031 and F-ec4cfa552cce. Both cases return **1**.

8. **Retained follow-up argument.** If operation **0x12C** returns **7** and F-3fac671648b8 returns nonzero, F-6904f7699031 receives **7**, regardless of unrelated receiver-state changes made by the intervening call.

9. **Primary exact match.** If record **0x0C** matches receiver **0x44**, and record **0x08** matches receiver **0x40**, operation **0xD4** executes and the result is **1**. Neither the primary global nor secondary enable field is read.

10. **Primary global alternative.** Under the same offset-0x0C match, a differing offset-0x08 value succeeds if it equals the primary global. Operation **0xD4** still takes precedence over any secondary match.

11. **Secondary success.** If primary matching fails, **0x9C** is nonzero, retained record **0x0C** matches receiver **0x98**, and the secondary read of record **0x08** matches the secondary global, F-5710a8628cba precedes operation **0xA0**. The operation receives its one-byte result, and the function returns **1**.

12. **Secondary disabled or mismatched.** A zero **0x9C**, an offset-0x0C mismatch, or a final secondary-global mismatch yields **0** without either type-0x13 action.

## Uncertainties

The following remain **documented_only**:

- Application meanings of types, fields, global values, operations, and return values.
- Endianness, exact machine-load scheduling, and concrete calling conventions.
- Full object size, alignment requirements, and embedded-storage extent.
- Scratch contents, initialization coverage, and dependency interpretation of those regions.
- Dependency side effects, ownership behavior, retained addresses, and failure modes.
- Global mutability and synchronization requirements.
- Defined behavior under concurrent mutation, invalid addresses, or invalid dispatch targets.
- Machine-level argument extension for the one-byte value passed to operation **0xA0**.

The visible comparisons use equality and zero/nonzero tests, not signed ordering. Mask **0x10** applies to a represented 32-bit field. Apart from fixed-offset address formation, no numeric arithmetic result is exposed. Portable semantics for address overflow are not established; valid reached addresses remain a precondition.

## Evidence

This contract derives solely from the supplied static function body and opaque dependency mapping.

Established evidence covers represented widths, field offsets, conditional reads, branch precedence, explicit scalar assignments, absence of explicit scratch writes, dependency arguments, call order, and normal return values.

“Eligibility,” “primary,” “secondary,” and “follow-up” are explanatory labels. They do not establish application intent or dependency semantics.

No implementation was executed, no runtime tests were performed, and no acceptance example is reported as an observed result.
