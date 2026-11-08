<!-- Trusted provenance: F-cd18aed0fb4d; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-cd18aed0fb4d

## Purpose

F-cd18aed0fb4d performs a synchronous configuration-and-acquisition transaction. It prepares configuration storage, invokes two transfer operations separated by a delay, and polls a completion indication subject to a time-limit check. After completion, it acquires a result byte, assembles an unsigned value, optionally applies an adjustment for one exact selector value, and writes a 32-bit result to the caller’s output object.

This transaction structure is established by static evidence. Interpreting the transfers as device-register operations is inferred intent. Device identity, measurement units, selector meanings, and the physical effects of the operations are unknown.

A zero return follows the function’s explicit result store. Earlier exits either propagate a nonzero read status or return 31 after a nonzero time-limit result. Configuration-operation return values are not checked.

## Interface

### Arguments

| Argument | Width and interpretation | Use |
|---|---|---|
| Selector | Unsigned 32-bit integer | Its low four bits contribute to configuration. Its complete value is compared with 7 when deciding whether to adjust the result. |
| Options | Unsigned 32-bit integer | Bit 1 selects result assembly and a base delay. Bit 0 changes configuration and adds 10 to the delay. No other bits are directly used. |
| Output address | Pointer to a writable unsigned 32-bit object | Receives the final result. Its address representation also initializes a separate 32-bit scratch object. |
| Auxiliary value | Opaque 32-bit quantity | Its highest-order byte initializes the highest-order byte of configuration storage. Its other bytes have no direct use. |

This contract calls options bit 1 the **compact-format option** and options bit 0 the **additional-setting option**. These names describe observable behavior, not established device terminology.

The supplied representation uses a 32-bit address value when initializing the result scratch object. It does not establish behavior for a wider-address environment.

### Return

The function returns a 32-bit integer status:

- **0:** successful result acquisition, followed by the explicit output store.
- **31:** a time-limit check returned nonzero.
- **Any propagated nonzero read status:** returned without an explicit output store.

A read operation can itself return 31. The numeric return alone therefore does not always identify the exit stage. Propagation preserves the examined 32-bit status; no positive-only error convention is established.

### Preconditions and storage lifetime

For a defined successful store, the output address must designate an appropriately aligned, writable 32-bit unsigned object whose lifetime includes that store. No explicit null, alignment, or accessibility check is visible.

Two separate four-byte local scratch objects are used: configuration storage and result storage. They remain alive throughout this invocation, including between dependency calls. Their lifetime does **not** prohibit a dependency from retaining an address and using it later while this function is still executing. Whether any dependency actually retains or reuses such an address is unknown.

After this function returns, those local objects no longer exist. A retained address does not extend their lifetime. No ownership transfer, allocation, or release is established.

Dependency accesses must remain valid for the actual interfaces involved. Interpreting transfer arguments as byte counts is conditional; the dependency implementations were not supplied.

## Behavior

### Configuration preparation

Before invoking dependencies, the function captures whether the compact-format option is enabled.

It constructs a configuration byte whose:

- Upper nibble is the selector’s low four bits.
- Lower nibble is 15 in compact format and 13 otherwise.

Thus, selector bits above bit 3 do not affect this byte. They remain relevant to the later full-width equality comparison with 7.

The initialized configuration storage has the following layout:

| Byte offset | Additional setting disabled | Additional setting enabled |
|---|---:|---:|
| 0 | 0 | 1 |
| 1 | 0 | 2 |
| 2 | Configuration byte | Configuration byte |
| 3 | Auxiliary value’s highest-order byte | Auxiliary value’s highest-order byte |

Offsets correspond to the little-endian byte layout represented by the supplied static evidence.

The selected delay argument is:

| Compact format | Additional setting | Delay argument |
|---|---|---:|
| Disabled | Disabled | 25 |
| Enabled | Disabled | 20 |
| Disabled | Enabled | 35 |
| Enabled | Enabled | 30 |

The selector does not affect this delay. No time unit is established.

Before the first dependency call, the function also initializes the separate result scratch object from the output address representation. Passing addresses within that scratch object does not directly pass the output object as the acquisition buffer.

### Configuration call sequence

The function performs these operations in order:

1. Calls F-ed60e3f64839 with identifier 82, second argument 1, and the address of configuration offset 0.
2. Calls the delay dependency with the selected delay argument.
3. Calls F-ed60e3f64839 with identifier 83, second argument 2, and the address of configuration offset 1.

Neither configuration return value nor the delay return value is inspected. There is no explicit failure exit between these calls based on their returned values.

If the second argument is a byte count, the first configuration operation covers offset 0, while the second covers offsets 1 and 2. Offset 3 is outside both such ranges. That does not establish that offset 3 is irrelevant to every possible dependency implementation.

Configuration storage is **not reconstructed** between the calls. The second operation receives a pointer into its then-current state, not a separately preserved copy of the initialized bytes. Its contents can therefore differ from initialization if an intervening dependency changes them, including through an address retained from the first call. Buffer preservation is not established merely because the apparent first transfer range contains one byte.

### Timing inputs and polling order

After the second configuration call, the function:

1. Invokes the time-reference dependency and captures its 32-bit result.
2. Reads and captures a global 32-bit timing value.
3. Begins polling.

The global read occurs after all preceding dependency calls, including time-reference acquisition. Its captured value cannot be assumed to equal its entry-time value.

Each iteration begins by calling the time-limit dependency with the captured reference and captured global value. The function does not explicitly refresh either argument during polling.

If the time-limit dependency returns nonzero, the function immediately returns 31. No completion read occurs on that iteration.

The meaning of the timing arguments, elapsed-time arithmetic, clock units, wraparound rules, and timeout boundary are delegated to that dependency.

### Completion acquisition

When the time-limit dependency returns zero, the function calls F-b3e2a11b7207 with:

- Identifier 87.
- Second argument 1.
- The address at byte offset 2 within result scratch storage.

It examines the returned 32-bit status. A nonzero status is returned immediately, with no explicit output store.

On zero status, the function examines bit 23 of the result scratch word: the high bit of the byte at offset 2 in the represented layout.

- If clear, the function begins another iteration with another time-limit check.
- If set, the function proceeds to result acquisition.

There is no explicit delay, yield, or fixed retry-count limit between incomplete polls. Polling can continue indefinitely if time-limit checks remain zero, completion reads remain successful, and the completion bit remains clear.

### Result acquisition and assembly

After observing completion, the function calls F-b3e2a11b7207 with:

- Identifier 85.
- Second argument 1.
- The address of result scratch offset 0.

The low 32 bits of the returned value are treated as status. If nonzero, they are returned immediately without the explicit output store.

On zero status, assembly uses result scratch storage as it exists after that call:

| Format | Unadjusted result | Range |
|---|---|---:|
| Compact | Byte at offset 0 | 0–255 |
| Noncompact | Four times the byte at offset 0, combined with the low two bits of the byte at offset 2 | 0–1023 |

In noncompact format, the two contributions occupy disjoint bit positions: the offset-0 byte supplies bits 2–9, and the offset-2 byte supplies bits 0–1. Their combination has no carry or overflow.

Under a one-byte-write interpretation of the reads, the offset-2 contribution comes from the prior completion read. However, that byte is not separately preserved. If the result dependency changes it, directly or through another permitted access, assembly uses the changed value.

The result dependency exposes an upper 32-bit return component as well. It does not affect the published result in either format: compact assembly does not use it, and noncompact assembly takes the relevant bits from scratch storage instead.

### Exact-selector adjustment

Only a full 32-bit selector equal to 7 activates adjustment:

| Unadjusted value | Final value |
|---|---|
| 0–63 | Unchanged |
| 64–159 | Decreased by 1 |
| 160–255 | Decreased by 2 |
| 256–1023 | Unchanged |

Compact format cannot produce the final row’s unadjusted range.

The range predicates use unsigned 32-bit subtraction. In particular, an intermediate subtraction of 2 from 0 or 1 wraps modulo 2³². Neither input reaches a subtracting adjustment branch, so both remain unchanged. This behavior is neither signed range checking nor saturation.

A selector such as 23 has the same low configuration nibble as 7 but does not receive this adjustment.

### Output publication

After successful result acquisition and any applicable adjustment, the function performs one explicit 32-bit store to the output object and returns zero.

Every earlier return bypasses that store. This is a statement about this function’s direct writes, not a guarantee that unknown dependencies leave the output object or other reachable memory unchanged.

## Side effects

Established direct effects and their order are:

- Initialization of two local four-byte scratch objects.
- Two configuration dependency calls, separated by one delay call.
- One time-reference call, followed by one global timing-value read.
- One time-limit call per entered polling iteration.
- Zero or more completion reads.
- At most one result read.
- Exactly one explicit output store on the successful acquisition path.

The function contains no visible allocation, deallocation, ownership release, rollback, cancellation, or final reset operation.

If the transfer dependencies access hardware, the configuration calls may alter external state and reads may observe or alter that state. Read-to-clear behavior and other physical effects are not established.

No synchronization is visible. Thread safety, reentrancy, shared-device coordination, and atomicity of the complete transaction are unknown.

## Dependencies

The following identifiers name supplied dependencies:

| Opaque ID | Observed use | Return handling |
|---|---|---|
| F-ed60e3f64839 | Calls with identifiers 82 and 83 and pointers into configuration storage | Ignored |
| F-b3e2a11b7207 | Calls with identifiers 87 and 85 and pointers into result storage | Nonzero 32-bit status propagated |

Three external services lack supplied opaque IDs. The following are contract-local opaque labels, not independently established identities:

| Contract-local opaque ID | Observed use | Return handling |
|---|---|---|
| F-opaque-01 | Receives the selected delay argument between configuration calls | Ignored |
| F-opaque-02 | Called once after configuration | Captured as a 32-bit time reference |
| F-opaque-03 | Receives captured timing inputs before each possible completion read | Zero permits progress; nonzero causes return 31 |

The global 32-bit timing value is another external input. Its initialization and interpretation are unknown.

No dependency implementation is available. This contract does not assume that any dependency preserves globals, preserves supplied buffers, refrains from retaining addresses, acquires ownership, or releases memory.

## Acceptance examples

These are conditional expectations derived from static evidence, not executed tests.

For exact transfer-byte examples below, assume the second transfer argument is a byte count. For acquisition examples, also assume successful reads write exactly their requested byte and that no other dependency or retained alias changes the relevant scratch bytes before they are consumed. These assumptions do not establish configuration-buffer preservation; examples 1 and 2 state that separately.

1. **Configuration with neither option enabled.**  
   Selector 3 and options 0 initialize configuration offsets 0–2 to 0, 0, and 61. The first operation receives the address of offset 0, and the delay argument is 25. **If the first configuration call and delay leave offsets 1 and 2 unchanged, including through retained aliases**, the second operation transfers 0 followed by 61. Without that preservation condition, it receives their then-current values.

2. **Configuration with both options enabled.**  
   Selector 3 and options 3 initialize offsets 0–2 to 1, 2, and 63. The delay argument is 30. **If the first configuration call and delay leave offsets 1 and 2 unchanged, including through retained aliases**, the second operation transfers 2 followed by 63. The function does not restore those values if they were changed.

3. **Immediate time-limit exit.**  
   If the first time-limit check returns nonzero, the function returns 31 after both configuration calls, the delay, time-reference acquisition, and the global read. It performs no completion read or explicit output store.

4. **Incomplete poll followed by another check.**  
   A successful completion read supplying byte 1 leaves the completion bit clear. The next observable dependency call is another time-limit check, not an immediate completion read.

5. **Noncompact assembly.**  
   With compact format disabled, completion byte 131 and result byte 18 produce an unadjusted value of 75: 72 from the result byte and 3 from the completion byte. Selector 3 stores 75; selector 7 stores 74.

6. **Compact assembly.**  
   With compact format enabled, result byte 18 produces 18 regardless of the completion byte’s low two bits. Completion must still have been observed before the result read.

7. **Adjustment boundaries.**  
   For selector 7, unadjusted values 0, 1, 63, 64, 159, 160, 255, and 256 yield 0, 1, 63, 63, 158, 158, 253, and 256 respectively.

8. **Full selector comparison.**  
   Selectors 7 and 23 produce the same selector-derived configuration nibble. Given unadjusted value 64, selector 7 stores 63 while selector 23 stores 64.

9. **Read failure.**  
   A nonzero status 9 from either read stage causes return 9 without the explicit output store. A result-read failure necessarily follows an observed completion indication. A read status of 31 is propagated as 31, indistinguishable numerically from the time-limit exit.

## Uncertainties

The following remain **documented_only**:

- Device identity, selector meanings, measurement units, and adjustment rationale.
- Transfer-count semantics and actual buffer-access bounds.
- Dependency mutation of configuration storage before its second use.
- Retention and later reuse of scratch addresses during the invocation.
- Dependency effects on globals, output memory, or shared external state.
- Delay units and guarantees.
- Timing-value meaning, clock arithmetic, and timeout boundaries.
- Configuration failure reporting and consequences of ignored returns.
- Ownership behavior within dependencies.
- Thread safety, reentrancy, and invalid-pointer fault behavior.

The scratch objects’ invocation-wide lifetime is established; actual retained-address use is not. Access after return is not supported by their lifetime.

The purpose of initializing result scratch from the output address is unknown. Under the stated successful one-byte-read assumptions, address-derived bytes do not contribute to the assembled result.

## Evidence

**Established static evidence:** argument widths, configuration layout, option selection, delay values, scratch initialization, dependency call order, timing-input capture, completion-bit position, status propagation, result assembly, unsigned adjustment boundaries, and the final output store.

**Inferred intent:** the transfer identifiers designate configuration, completion, and result locations, and the polling loop waits for an external acquisition. These interpretations do not establish a specific protocol or device.

**Runtime evidence:** none. No runtime tests, hardware observations, or dependency implementations were available.
