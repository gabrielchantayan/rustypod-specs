<!-- Trusted provenance: F-b9b3cfdafea6; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-b9b3cfdafea6

## Purpose

F-b9b3cfdafea6 checks an encoded record length against a context-supplied address limit and a minimum length of 262 bytes. If a context flag is nonzero, it also checks 256 consecutive record bytes against an unsigned bound supplied by the context.

The function calls F-7b0b9523800e with reason 8 when the initial length/address check fails, and with reason 16 for each scanned byte that violates its bound. Every execution that reaches the normal return produces a 32-bit zero. That return value does not distinguish an invocation that reported violations from one that did not.

**Established static behavior:** the length decoding, comparisons, conditional calls, scan order, and normal return are visible in the supplied representation. Neither kind of dependency call is followed by an explicit termination of this function.

**Inferred intent:** the function appears to validate a record containing a two-byte length and a fixed-size byte table. The semantic record format and the meaning of the table entries are not established.

**Unknown:** F-7b0b9523800e may return, alter state, or transfer control elsewhere. Its placement does not establish that reporting a violation prevents subsequent accesses.

## Interface

### Inputs

The visible interface has two 32-bit address-valued arguments.

| Input | Role |
|---|---|
| Record address | Base for the encoded length and the optionally scanned bytes. |
| Context address | Base for the address limit, scan flag, and byte bound; also passed to F-7b0b9523800e. |

No independent allocation length or buffer capacity is supplied. The function does not visibly check either address for nullness, alignment, lifetime, or membership in an allocation.

### Record layout

All offsets are byte offsets from the record address.

| Offset | Access width | Established use |
|---|---:|---|
| 2 | 8 bits | High byte of the encoded length. |
| 3 | 8 bits | Low byte of the encoded length. |
| 6–261 inclusive | 8 bits per entry | The 256 values examined when scanning is enabled. |

The length is assembled explicitly in high-byte-first order, independently of the byte order used for context word accesses. It is an unsigned 16-bit value widened to 32 bits, with possible values from 0 through 65,535.

The function does not visibly inspect record offsets 0, 1, 4, or 5. The scan does not extend beyond offset 261 even when the encoded length is larger than 262.

### Context layout

All offsets are byte offsets from the context address.

| Offset | Access width | Established use |
|---|---:|---|
| 4 | 32 bits, unsigned | Address limit for the initial comparison. |
| 8 | 8 bits | Scan-enabling flag; zero disables scanning, any nonzero value enables it. |
| 400 | 32 bits, unsigned | Exclusive upper bound for each scanned byte. |

The supplied representation does not independently establish the storage byte order or alignment requirements of the 32-bit context accesses.

### Output

Normal completion returns a 32-bit zero.

A zero result is not evidence that all checks passed. If the dependency calls return, the function can report violations and still return zero. There is no visible accumulated error result.

### Ownership and access preconditions

Each memory access actually reached requires readable storage suitable for that access. These requirements apply independently of whether the initial validation succeeds: the length bytes are accessed before the check, and a returning reason-8 call can be followed by the full scan.

The visible body performs no allocation, deallocation, ownership transfer, or retention of either address. Passing the context address to F-7b0b9523800e does not establish that dependency’s ownership behavior. Continued valid storage after a dependency call is a condition for subsequent accesses, not a guarantee made by this function.

## Behavior

### Length capture

The function reads record offsets 2 and 3 and combines their unsigned byte values into the length. Offset 2 contributes the upper eight bits; offset 3 contributes the lower eight bits.

Both reads precede the initial validation decision. Their relative order is not established by the supplied expression.

The decoded length is captured once. A later dependency-induced change to those bytes does not cause the length to be decoded again or the initial check to be repeated.

### Initial length and address check

The function reports an initial violation when either of the following is true:

- The unsigned 32-bit address limit at context offset 4 is less than the computed record end.
- The decoded length is less than 262.

The computed record end is the record address plus the decoded length in the exhibited 32-bit unsigned arithmetic domain. An overflowing sum wraps modulo 2³². There is no separate carry or overflow rejection.

The address comparison is strict: equality between the stored limit and the computed end does not trigger a violation through this comparison. Acceptance of equality does not, by itself, establish whether the surrounding format treats the stored limit as an exclusive endpoint.

The minimum-length logic rejects lengths below 256 and lengths from 256 through 261. Together these reject exactly 0 through 261. The length of 262 is the first value that satisfies this portion of the check.

When either condition fails, the function makes one call to F-7b0b9523800e, passing:

1. The context address.
2. Reason value 8.

If both conditions fail, this stage still makes only one call. The address comparison is evaluated before the minimum-length alternatives; an already-failing address comparison can short-circuit those alternatives. The alternatives require no additional memory reads.

The initial check does not itself stop processing. If the reason-8 call returns normally, execution continues to the flag read.

### Scan decision

After the initial check, including any returning reason-8 call, the function reads the byte at context offset 8.

- If the byte is zero, the function skips the scan and proceeds to its normal return.
- If the byte is nonzero, the function starts the fixed-size scan.

Consequently, the scan decision uses the flag value read after any reason-8 call, not necessarily its value at function entry. A returning dependency call that sets the flag can enable scanning; one that clears it can disable scanning.

The flag is not visibly read again once scanning begins. Changing it during a later reason-16 call does not, by itself, end the scan.

### Fixed-size byte scan

An enabled scan visits record offsets 6 through 261 in increasing order. It contains 256 iterations if every required access and dependency call permits continued normal execution.

For each position, the function obtains:

- The unsigned byte value at the current record position, in the range 0–255.
- The unsigned 32-bit bound at context offset 400.

The byte violates the bound when its value is greater than or equal to the bound. Equivalently, a byte passes only when its value is strictly less than the bound.

Each violating byte causes one call to F-7b0b9523800e with the context address and reason value 16. If the call returns normally, scanning continues at the next position. The current byte is not checked again.

The bound is accessed within each iteration rather than captured once for the entire scan. A change made during one dependency call can therefore affect later comparisons. The function likewise provides no snapshot of unvisited record bytes; changes to those bytes can affect later iterations.

The relative order of the bound read and the record-byte read within one comparison is not established. Both values are obtained before the corresponding reporting decision.

The scan size is independent of the decoded length:

- A short encoded length does not shorten the scan if the reason-8 call returns and the subsequently read flag enables scanning.
- A larger encoded length does not extend the scan beyond offset 261.

### Normal completion

After either skipping or completing the scan, the function returns zero.

There is no final recheck of the length, limit, flag, bound, or previously visited bytes. No visible result records how many violations occurred.

## Side effects

### Direct effects

The visible body reads memory and conditionally calls F-7b0b9523800e. It contains no explicit write to record or context storage.

For an execution that reaches normal completion, the visible access structure includes:

- Two byte reads for the encoded length.
- One unsigned 32-bit read of the address limit.
- One byte read of the scan flag.
- If scanning is enabled, 256 record-byte reads and 256 unsigned 32-bit bound reads.

A disabled scan performs neither the record-byte comparisons nor the bound reads. The encoded length and initial address limit are still read.

These describe the accesses in the supplied static representation, not an independently observed machine-level memory trace.

### Dependency-mediated effects

The contract does not assume that F-7b0b9523800e preserves the record, context, unrelated globals, allocation state, or control flow. Effects may also occur through aliases or external state not visible in this function.

If all calls return normally and the required accesses remain valid:

- The initial stage makes at most one reason-8 call.
- The scan makes at most 256 reason-16 calls.
- The maximum total is 257 calls.

Any reason-8 call precedes every reason-16 call. Reason-16 calls follow increasing record-offset order, though their arguments do not include the offset or offending byte value.

If a dependency call does not return, later reads, later calls, and the normal zero return are not established.

### Safety limitations

The initial validation is not an established memory-safety barrier:

- The length bytes are read before validation.
- The function does not verify the allocation containing either address.
- A returning reason-8 call does not prevent the scan.
- Scan length remains fixed even when the encoded length is too short.
- Address-sum overflow is not rejected.
- A dependency may alter or invalidate storage needed afterward.

Concrete behavior following an invalid access is not established.

## Dependencies

### F-7b0b9523800e

The observed arguments are the context address followed by a numeric reason.

| Reason | Trigger |
|---|---|
| 8 | The stored unsigned address limit is below the wrapped 32-bit record end, or the decoded length is below 262. |
| 16 | Scanning is enabled and the current unsigned byte is greater than or equal to the bound read for that iteration. |

F-b9b3cfdafea6 consumes no return value from this dependency.

**Inferred intent:** the dependency appears to report a validation failure or transfer control because of one.

**documented_only:** its implementation and behavioral contract are unavailable. Termination, exceptions, logging, memory mutation, global-state preservation, allocation, release, and ownership effects remain unknown. Neither reason value establishes a particular error-reporting mechanism.

## Acceptance examples

These are conditional criteria derived from static evidence, not executed tests. Unless an example states otherwise, assume valid readable storage, no address-sum wrap, stable relevant data, and dependency calls that return normally without modifying relevant state. Each scan example explicitly requires a nonzero scan-enabling flag.

### Minimum length and equal address limit

Given encoded length 262, an address limit equal to the record address plus 262, and a zero scan flag:

- No reason-8 call occurs.
- No table bytes or byte bound are read.
- No reason-16 call occurs.
- Normal completion returns zero.

### Length below the minimum

Given encoded length 261, an adequate address limit, and a zero scan flag:

- Exactly one reason-8 call occurs.
- The scan is skipped.
- Normal completion returns zero.

Lengths 0, 255, and 256 also trigger reason 8 under these conditions.

### Insufficient address limit

Given encoded length 262 and an address limit one less than the computed end:

- Exactly one reason-8 call occurs.
- After that call returns, the subsequently read flag determines whether scanning occurs.

If the encoded length is also below 262, the initial stage still reports only once.

### Exclusive byte boundary with scanning enabled

Given passing initial checks, a nonzero scan-enabling flag, and a stable bound of 10:

- A scanned byte of 9 passes.
- A scanned byte of 10 triggers reason 16.
- A scanned byte of 255 triggers reason 16.
- If exactly one of the 256 scanned bytes is at least 10, exactly one reason-16 call occurs.
- Normal completion returns zero.

The nonzero flag is essential. With a zero flag, no byte comparisons or reason-16 calls occur, even if exactly one record byte is at least 10.

### Bound of 256 or greater

Given passing initial checks, a nonzero scan-enabling flag, and a stable unsigned bound of 256:

- All 256 byte positions are visited.
- Every possible byte value passes.
- No reason-16 call occurs.
- Normal completion returns zero.

Any larger unsigned 32-bit bound produces the same comparison outcome for byte values.

### Zero bound

Given passing initial checks, a nonzero scan-enabling flag, and a stable bound of zero:

- Every scanned byte violates the bound, including a byte equal to zero.
- Exactly 256 reason-16 calls occur if all return normally.
- Normal completion still returns zero.

### Returning initial reporter changes the flag

Suppose the initial check triggers reason 8. If the dependency sets an initially zero flag to a nonzero value before returning, the function enters the scan. If it instead clears the flag before returning, the function skips the scan.

These outcomes depend on the post-call flag read, not the flag’s entry value.

### Returning byte reporter changes later checks

Given a nonzero scan-enabling flag, suppose the first scanned byte violates the bound. If the resulting reason-16 call changes the bound to 256 and returns, and that bound remains stable, all remaining 255 bytes pass.

The first byte is not revisited. Clearing the scan flag during that call does not stop the already-started scan.

### Larger encoded record

Given encoded length 1,000, an adequate address limit, and a nonzero scan-enabling flag:

- The initial address comparison uses length 1,000.
- The scan still visits only offsets 6 through 261.
- This function does not check bytes beyond offset 261.

### Maximum reporting count

Given an encoded length below 262, a nonzero scan flag that remains enabled after the reason-8 call, and a stable bound of zero:

- One reason-8 call occurs.
- If it returns and readable storage exists through offset 261, the scan makes 256 reason-16 calls.
- If all calls return, the total is 257 and normal completion returns zero.

### Wrapped endpoint comparison

For the initial comparison alone, a record address of 4,294,967,280 and length 262 produce a wrapped 32-bit end value of 246. A stored limit of 246 therefore does not trigger reason 8 through the address comparison, and the length satisfies the minimum.

This arithmetic criterion does not establish valid storage or successful execution at those addresses.

## Uncertainties

The following remain **documented_only**:

- The semantic record format and interpretation of the 256 byte values.
- The intended endpoint convention for the stored address limit.
- Target byte order and alignment requirements for context word accesses.
- Caller guarantees concerning storage size, lifetime, alignment, nonwrapping addresses, or prior validation.
- Whether F-7b0b9523800e normally returns.
- Dependency-induced mutation, allocation, release, logging, or nonlocal control transfer.
- Behavior under concurrent modification or special memory semantics.
- Concrete fault behavior for invalid addresses.
- Operand-read ordering where the supplied representation does not specify it.

These unknowns do not justify assuming early termination, immutable inputs, dependency-preserved globals, valid allocations, or a success-indicating return value.

## Evidence

The supplied static function representation establishes:

- Explicit high-byte-first decoding of two record bytes into an unsigned length.
- A 32-bit unsigned record-end comparison with no separate overflow rejection.
- Rejection of encoded lengths below 262.
- One conditional reason-8 call for the combined initial failure condition.
- A later flag read controlling whether the scan occurs.
- A fixed scan of 256 consecutive bytes in increasing offset order.
- A per-iteration unsigned bound comparison with conditional reason-16 reporting.
- Continuation after dependency calls when they return normally.
- No explicit writes to input storage and no accumulated error return.
- A 32-bit zero on normal completion.

The supplied dependency mapping identifies F-7b0b9523800e as the only visible callee. Its implementation is not supplied. No runtime tests have been performed.
