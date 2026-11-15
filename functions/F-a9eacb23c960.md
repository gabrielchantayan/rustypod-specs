<!-- Trusted provenance: F-a9eacb23c960; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-a9eacb23c960

## Purpose

F-a9eacb23c960 has an address-dependent memory interface with opaque external dependencies. Its application-level purpose is unknown.

The established behavior comprises two ordinary-entry cases and several conditional continuations:

- A nonzero fourth input produces a 24-byte record and terminal memory publication.
- A zero fourth input performs a separate sequence that may reject execution, encounter an interrupt, or return.
- Other represented continuations acquire values, publish records, process scalars, perform conversion and selection, or repeat indefinitely.

“Publication” describes writes visible outside the function. It does not imply synchronization, successful delivery, valid storage, or ownership transfer. Independent accessibility of the additional continuations is **documented_only**.

## Interface

| Input | Represented width | Established role |
|---|---:|---|
| First | 8 bits | Initial byte on zero entry |
| Second | 32 bits | No explicit use established |
| Third | 32 bits | Nonzero-entry record field |
| Fourth | 32-bit address identity | Entry discriminator, record field, and carried selector |

No scalar return value is established. Ordinary return is represented only after the final zero-entry invocation completes.

Ambient execution state supplies a frame position, addresses, arithmetic operands, globals, and invocation targets. Their initialization and complete calling convention are unknown.

The following conventions apply:

- Byte, halfword, word, and doubleword mean 8, 16, 32, and 64 bits.
- Signed narrow values are sign-extended; unsigned narrow values are zero-extended. Narrow writes retain only the destination-width bits.
- A dependency **pair** comprises numerical low and high 32-bit components. This terminology makes no memory-byte-order assertion.
- A **secondary result** is a separately represented 32-bit result channel.
- Publication value, base, cursor, retained address, displacement, and work buffer are descriptive carried-state roles, not additional declared parameters.
- Numerical address examples use conditional 32-bit modular arithmetic. Signed-subtraction overflow is a separate predicate, not the sign of a wrapped difference.

No bounds, alignment, permissions, lifetime, or null-address validation is established. Every effect below is conditional on reaching it and on preceding operations completing.

## Behavior

### Entry obligations

#### Nonzero fourth input

The entry record occupies 24 bytes at an ambient output address:

| Offset | Width | Required content |
|---:|---:|---|
| 0 | 64 bits | Pair from F-call-02 |
| 8 | 32 bits | Third input |
| 0x0c | 32 bits | Fourth input |
| 0x10 | 32 bits | Extended unsigned halfword |
| 0x14 | 32 bits | Extended signed halfword |

F-call-01 supplies a secondary-result address. The unsigned halfword comes from that address plus 0x24 and is captured before F-call-02. The signed halfword comes from fourth-input-plus-ambient-displacement and is read after F-call-02. Record fields are written in ascending offset order.

The cursor is the record end. Publication value is twice the sum of pair-low and 0x48; base is publication value minus 0x3f. A word at publication-value-plus-0x54 supplies the retained address.

Terminal publication uses cursor-minus-base as displacement and pair-high as index. Its first destination consequently equals the record end under modular addressing.

#### Zero fourth input

The first input is written at ambient-destination-plus-one before F-call-03.

An entry global has two distinct observations after that call. The first determines working address by exclusive-or with an ambient word. The second contributes its low halfword to a write at F-call-03’s secondary-result address plus 0x0a. This is not a read through the address represented by the global.

Rejection means that F-call-03’s secondary result is signed-greater-than 0x38. Regardless of that decision, the following precede possible rejection:

- A signed-halfword read at first-global-observation-plus-working-address.
- A word write of 0x31 at working-address-plus-0x74.
- F-call-04 with the extended halfword, followed by F-call-05.

Rejection reaches an invalid instruction. Otherwise, an unsigned byte at absolute 0x3b is captured before F-call-05’s secondary result is written at working-address-plus-0x24. A payload global is captured after that write.

F-call-06 receives the byte at twice-the-captured-byte-plus-two, twice working address, the captured byte, and 0x26. F-call-07 precedes interrupt 0xde. Post-interrupt obligations require resumption: F-call-08 precedes F-call-09, followed by a zero-byte write at 0x1d plus 0x200 times the carried F-call-05 secondary result.

F-call-10 supplies a pair. The captured payload is written at pair-high-plus-0x31 as a word; pair-low’s low halfword is then written at captured-byte-plus-0x32. F-call-11 receives twice pair-low. Ordinary return requires that invocation to complete.

### Publication contracts

#### Terminal publication

The required ordering is:

| Phase | Observable obligation |
|---|---|
| Initial write | Publication value, as a word, at base-plus-displacement |
| Destination capture | Capture the publication-destination global |
| Narrow write | Cursor’s low halfword at displacement-plus-four-times-index |
| Captured-destination writes | Publication value at offset 0x48, then twice cursor at offset 0x28 |
| Final narrow write | Base’s low halfword at offset 0x2c from the address-valued datum associated with absolute 0x3a |

An invalid instruction follows. The captured destination remains authoritative for both destination-relative writes even if an intervening write aliases its global.

#### Supplementary record and dispatch

A nonzero selector selects terminal publication.

For selector zero, unsigned base/cursor ordering, equality, and signed overflow of cursor-minus-base are captured before a six-word record is written at base:

| Offset | Content |
|---:|---|
| 0 | Cursor |
| 4 | Incoming high word |
| 8 | Zero |
| 0x0c | Base |
| 0x10 | Cursor minus base |
| 0x14 | Retained address |

The record uses ascending writes.

An incoming high word of zero bypasses F-call-12 and selects acquisition: current address is base, argument is cursor, numerical parameter is zero, and preliminary-byte source is cursor-plus-cursor-minus-base. Sign and overflow describe cursor-minus-base.

Otherwise, F-call-12 supplies a pair. A dispatch global is captured before an unsigned byte is read at retained-address-plus-0x11.

Strict unsigned base-below-cursor selects routing. It carries that pair and byte, the preceding publication value as routing value, cursor as selection address, and cursor-minus-base as arithmetic address.

When the ordering fails, a fresh dispatch-global observation controls the outcome:

- **Fresh zero:** acquisition has current address 0x26, argument 0x49, numerical parameter pair-high, and preliminary-byte source 0x26 plus the earlier dispatch capture. Sign is clear; overflow describes publication-value-minus-0x2a.
- **Fresh nonzero:** fixed publication writes words at 0x26000, 0x26008, 0x26004, and 0x2600c, in that order, containing 0x2a, 0x26, pair-high, and retained address. Selector becomes 0x26010 between the second and third writes; this does not write through 0x26010.

After fixed publication, F-call-13’s doubleword result is written at retained address. Ascending word fields at offsets 8, 0x0c, and 0x10 contain publication value, 0x26, and retained address. Common dispatch receives current address 0x26, the earlier dispatch capture as displacement, clear zero and sign predicates, and overflow from publication-value-minus-0x2a.

### Common dispatch

A retained zero predicate, or unequal retained sign and overflow, requires acquisition fallback. Fallback preserves overflow from old-retained-address-minus-0x2f before replacing retained address with 0x4f. Acquisition argument and numerical parameter are both 0x49; sign describes current address, and preliminary-byte source is current-address-plus-displacement.

Otherwise, handoff saves selector at frame offset 0x348 before F-call-14. Its returned byte is written at current-address-plus-0x0a. A subsequent invalid-instruction guard depends on the retained zero predicate, not on this byte. Common-dispatch admission to handoff has zero clear.

F-4bb678bae409 receives token 0x08571690 and the datum associated with absolute 0x09. F-call-15 then supplies a pair. Three ascending words at pair-high contain pair-low, current address, and 0x34. F-call-16 receives pair-low and that record’s 12-byte end.

A fresh handoff-global capture precedes a word write at absolute 0x68 and a halfword write at carried-displacement-minus-six. Their values are carried-displacement-minus-0x3a and 0x34.

Supplementary publication resumes with unchanged publication value, base equal to carried-displacement-minus-0x3a, selector and retained address equal to twice the fresh capture, and incoming high word equal to F-call-16’s secondary result minus 0x3e. Following the established fixed-publication route, cursor is twice publication value and carried displacement is the earlier dispatch capture.

### Acquisition and scalar processing

Acquisition establishes frame-plus-0x324 as work buffer. An auxiliary-address global is captured before the preliminary unsigned-byte read. F-call-17 receives the acquisition argument and numerical parameter.

Equal retained sign and overflow select scalar processing of pair-low. Unequal predicates select pair publication with the returned pair and preliminary byte.

Every scalar cycle captures the condition “work buffer unsigned-greater-than 0x2f and current address nonzero.” Before that decision takes effect:

- Incoming scalar is written at current-address-plus-0x54.
- Retained address’s low byte is written at current-address-plus-0x0d.
- F-call-18 receives the scalar and 0x27.
- F-call-19 supplies the processing scalar.

A true captured condition selects dependency-derived publication. Otherwise, a signed byte at current-address-plus-displacement becomes auxiliary address. Six ascending work-buffer words contain processing scalar, current address, work-buffer address, negated current address, extended signed byte, and retained address.

For zero or positive current address, conversion uses current address as bundle source and current-address-plus-processing-scalar as address-sum operand. Minimum-signed-word negation remains **documented_only**.

For negative current address, frame offset 0x17c receives that address. A word at current-address-plus-eight is captured before F-call-20. Twice that captured word becomes both current and retained address. The extended signed byte becomes work buffer and displacement while remaining auxiliary address. Scalar processing resumes with F-call-20’s result, without repeating acquisition.

### Dependency-derived publication

F-call-21 supplies a pair. Five ascending words at pair-high contain auxiliary address, work buffer, current address twice, and retained address. F-call-22 receives pair-low and the 20-byte record end. Work buffer is saved at frame offset 0x2f4 before F-call-23.

A nonzero F-call-23 secondary result selects repetition. Each iteration writes current address’s low byte at destination-plus-two. The first destination is retained address; later destinations are current address. The signed controller remains current address:

- A negative controller exits after one iteration.
- A nonnegative controller has no represented normal exit.

After negative exit, F-9dbe6a32552a supplies a pair. Pair-low is written at pair-high-plus-0x74 before a byte at pair-high supplies an offset for a signed-halfword read from that base. F-514d608fbe68 receives the extended halfword, the base, and frame storage at offset 0x398.

Its word result and a subsequent global observation form numerical low and high words. Publication value, base, and retained address are derived as on nonzero entry, without constructing its record. Cursor is controller-minus-0x64; selector remains the controller. Terminal publication uses the new high word as index.

A zero F-call-23 secondary result instead requires F-call-24. Current address’s low byte is written two bytes beyond an address read from frame offset 0x2bc. Two globals are captured in order. A fresh observation of the first is written at twice-work-buffer-plus-0x64 before an unsigned byte is read at twice the sum of work buffer and auxiliary address.

F-call-25 receives F-call-24’s result and the second global capture. Pair-high is saved at frame offset 0x370. Pair publication uses that pair, the byte, and the first global capture as retained address.

### Pair publication and feedback

The publication scalar initially comes from an unsigned byte; feedback may replace it with a full word.

| Order | Required access |
|---:|---|
| 1 | Word write of scalar at pair-low-plus-0x18 |
| 2 | Signed-byte read at scalar-plus-pair-high |
| 3 | Word write of twice pair-low at extended-byte-plus-0x18 |
| 4 | Halfword write of retained address at retained-address-plus-pair-high |
| 5 | Byte write of twice pair-low at pair-high-plus-six |
| 6 | Byte write of twice pair-high at retained-address-plus-0x15 |

Auxiliary address becomes frame-plus-0x3a0. F-be507c50aa84 receives twice pair-high. Derived current address is twice pair-high minus 0x0b. After the call, twice derived current address is written at twice-derived-current-address-plus-0x58; a word 0x2c follows at auxiliary-address-plus-secondary-result.

Frame publication has these constraints: token 0x083fbb72 is written at old-frame-minus-four before a fresh scalar-global observation is written at old-frame-minus-eight. Frame then decreases by 12, and derived current address is written at the new frame position.

The numerical datum associated with absolute 0x3e preserves its low halfword and receives the secondary result’s low halfword as its upper halfword. A later scalar-global observation supplies the feedback scalar.

Feedback reuses F-call-17 with 0x2c and the secondary result, without acquisition’s auxiliary-global capture or preliminary-byte read. Work buffer is twice the preceding pair-low plus 0x55; displacement is twice the preceding pair-high; retained address persists. Sign is clear and overflow describes twice-the-preceding-pair-high-minus-0x2c.

### Conversion and routing

F-call-26 receives processing scalar and address-sum operand; its consumed result is a halfword. Retained address is saved at old-frame-minus-four. Frame decreases by eight, and auxiliary address is written at the new frame position.

Only afterward are bundle words at offsets 0, 4, and 8 captured, in order. Auxiliary address’s low halfword is written at current-address-plus-four before the returned halfword is written at auxiliary-address-plus-first-bundle-word.

A conversion global is captured before a fresh observation whose doubled value supplies another halfword destination for auxiliary address. Twice the second bundle word is written at twice-that-word-plus-0x24. The earlier global capture is written at twice-its-value-plus-four.

Routing receives:

| Role | Value |
|---|---|
| Pair-low / pair-high | Twice third / twice second bundle word |
| Retained and arithmetic addresses | Twice captured conversion global |
| Routing scalar | Third bundle word |
| Selector | Second bundle word |
| Routing value | Current address minus 0x8a |
| Selection address | Unresolved address identity also used as F-call-11’s target |

Adjusted value is routing-value-plus-0x46 and becomes publication value. Under the modular interpretation:

| Unsigned routing value | Initial outcome |
|---|---|
| 0–0x7fffffff | Direct selection |
| 0x80000000–0xffffffb9 | Arithmetic publication |
| 0xffffffba–0xffffffff | Exceptional operation first |

The exceptional operation carries immediate 0xd3 and token 0x085714da; invocation of its supplied target is represented. Ordinary continuation is not established. Hypothetical continuation would require nonzero adjusted value and signed routing value greater than −0x47 for direct selection.

### Arithmetic publication

Adjusted value is written at adjusted-value-plus-arithmetic-address. A frame word at offset 0x2c4 is captured and copied to offset 0x294. Five ascending words at pair-low contain pair-high-minus-0x52, that capture, arithmetic address, its signed-right-shift-by-16 result, and arithmetic address again.

F-call-27 receives the 20-byte record end. Arithmetic address’s low halfword is written at arithmetic-address-plus-0x24 before F-call-28 and F-call-29.

Current-address and destination globals are captured in order before an unsigned-byte read at arithmetic-address-plus-0x0d. A word write uses fresh destination-global observations for both destination and contents; their relative evaluation order is unresolved.

Six ascending words starting four bytes beyond the earlier destination capture contain arithmetic address, captured current address, shifted arithmetic address, routing scalar, arithmetic address again, and extended byte.

Conversion repeats with arithmetic address as bundle source and address-sum operand, captured current address as current address, auxiliary address 0x2a, captured byte as retained address, and captured destination as processing scalar.

### Selection

Selection retains overflow from adjusted-value-minus-0x37 and sign/zero from selection address. Before success is decided, byte 0x2c is written at retained-address-plus-nine, an unsigned halfword is read at selection-address-plus-0x3a, and F-2c498e355d5f is invoked.

Success requires nonzero selection address and equal retained sign and overflow. F-call-30 precedes three ascending selector-relative word writes containing its secondary result, selection address, and retained address. F-call-31 precedes a retained-address word write at selection-address-plus-0x54.

Terminal publication then uses the observed halfword as displacement, F-call-31’s secondary result as index, F-c4570aea96e0’s address identity as cursor, selection address as base, and adjusted value as publication value. F-c4570aea96e0 is not invoked.

Failure carries the predicates and halfword displacement into common dispatch. Either failure condition requires acquisition fallback rather than handoff.

## Side effects

Observable effects include the specified memory accesses, dependency invocations, frame changes, repetition, interrupt, and invalid or exceptional instructions.

Captured and fresh observations of a global are distinct. Intervening writes and dependencies may alter later observations. No dependency is assumed to preserve globals.

Invalid-instruction boundaries have no represented ordinary continuation. Interrupt resumption is conditional. Faults do not undo preceding effects, and wide writes are not established as indivisible.

No allocation, release, ownership transfer, retention permission, lifetime extension, cleanup, or transactional guarantee is established. Passing record or frame storage to a dependency does not establish what that dependency may retain or release.

## Dependencies

F-call-01 through F-call-31 are opaque invocation-role IDs. Their explicit arguments, consumed results, and ordering are specified above. Numbering does not establish distinct targets; target identity may depend on carried invocation state. Unlisted arguments or result channels are not established.

Additional invoked dependencies are F-4bb678bae409, F-be507c50aa84, F-2c498e355d5f, F-9dbe6a32552a, and F-514d608fbe68. F-c4570aea96e0 supplies address identity only.

Dependency memory effects, global preservation, ownership behavior, exceptions, and termination remain **documented_only**.

## Acceptance examples

These are conditional static obligations, not executed tests. Required preceding operations must complete.

| Condition | Required consequence |
|---|---|
| Entry halfwords are unsigned 0xffff and signed 0x8000 | Record words contain 0x0000ffff and 0xffff8000 |
| F-call-02 alters the earlier halfword source | The record retains the pre-call unsigned-halfword observation |
| Nonzero-entry pair-low is zero | Publication value is 0x90, base is 0x51, and retained-address source is 0xe4 |
| Zero-entry global observations differ | Working address uses the first; the narrow write uses the second |
| F-call-03 secondary result is 0x39 | Preliminary read, 0x31 write, F-call-04, and F-call-05 precede rejection |
| That result is 0x38 or 0xffffffff | Signed rejection is false |
| Captured zero-entry byte is 0xff | F-call-06’s byte source is 0x200; the later halfword destination is 0x131 |
| Terminal narrow publication aliases its destination global | Both subsequent destination-relative writes use the earlier capture |
| Fixed publication faults at 0x26004 | Writes at 0x26000 and 0x26008 precede the fault; F-call-13 has not run |
| Pair-publication signed byte is 0x80 | The following word destination is 0xffffff98 under modular addressing |
| Preserved low halfword is 0xabcd; secondary low halfword is 0x5678 | The combined numerical datum is 0x5678abcd |
| Routing value is 0xffffffb9; arithmetic address is 0x100 | Arithmetic publication writes 0xffffffff at 0xff under modular interpretation |
| Routing value is 0xffffffba | Exceptional processing occurs first; hypothetical adjusted value is zero |
| Routing value is zero; selection address is 0x80000000 | Preliminary selection effects precede sign/overflow-mismatch fallback |
| Selection succeeds with selector zero | Record destinations are 0, 4, and 8; no null rejection is established |
| Repetition controller is 0xffffffff | One iteration precedes cursor 0xffffff9b under modular interpretation |
| Interrupt does not resume | No post-interrupt effects are required |

## Uncertainties

The following remain **documented_only**:

- Application intent, ambient initialization, complete calling convention, and independent continuation accessibility.
- Unresolved invocation targets and the selection-address identity.
- Dependency preservation, ownership, memory effects, exceptions, and termination.
- Executability and interpretation of overflowing arithmetic, signed shifts, pointer operations, and minimum-signed-word negation.
- Memory byte order, alignment, access decomposition, atomicity, permissions, and lifetime.
- Fault handling, interrupt resumption, and continuation after exceptional instructions.
- Observation width used to preserve the low halfword associated with absolute 0x3e, and datum width associated with absolute 0x09.
- Relative evaluation order of arithmetic publication’s fresh destination and contents observations.

## Evidence

**Established static evidence:** the supplied function representation supports the specified widths, layouts, explicit ordering constraints, captured-versus-fresh observations, dependency arguments and consumed results, retained predicates, frame changes, repetition condition, and exceptional operands.

**Inferred interpretation:** descriptive state roles, conditional modular arithmetic, and routing intervals derived under that arithmetic model. These do not establish application intent, valid storage, or successful execution.

**Unknown:** runtime reachability, dependency behavior, usable memory, and completion. No runtime tests, dependency executions, or memory traces have been performed.

The prior draft, embedded comments, and reviewer feedback are not independent behavioral evidence.
