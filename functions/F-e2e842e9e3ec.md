<!-- Trusted provenance: F-e2e842e9e3ec; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-e2e842e9e3ec

## Purpose

F-e2e842e9e3ec selects a record using a value obtained from a dependency, rejects three selection conditions, and otherwise passes addresses within the selected record to a sequence of five calls. It then passes the record’s first 32-bit word to another dependency.

The record offsets and visible text fragments suggest a diagnostic operation involving saved processor registers and status. That interpretation is **inferred intent**, not an established output contract. The supplied material does not establish what the dependencies print, whether they print at all, or what the software interrupts do.

The established contract is the selection logic, address calculations, dependency-call sequence, software-interrupt sequence, and conditional 64-bit return values. Normal completion of any path is conditional on its interrupts and dependencies returning control.

## Interface

### Inputs

The recovered interface declares no ordinary parameters. It nevertheless consumes an incoming value from register `r3`, initially saving it in a local pointer slot.

That incoming value has no explicit influence on the visible branch decisions or dependency arguments: it is replaced by the selected record pointer before successful-path dependency calls. On early-return paths, the initial saved value is not explicitly dereferenced. This does not establish that the register is irrelevant to the original calling convention or to interrupt handling.

The function also relies on external state:

- A selector returned by F-459a7f325a37.
- A base reference used to locate a 32-bit exclusion value at byte offset 12.
- A table-base reference used to locate a record pointer.
- Five format-like references supplied to F-88996a971ccf.
- A label-like reference supplied to F-5d4d7ea4b919.

These references have no supplied initialization, validity, concurrency, or ownership contract.

### Widths and layouts

The supplied representation establishes these operation sizes:

| Item | Size or arithmetic |
|---|---|
| Selector | 32-bit signed integer representation |
| Exclusion value | 32-bit integer load |
| Record element | 4 bytes |
| Table entry stride | 52 bytes |
| Record-pointer field within an entry | Byte offset 16 |
| Return value | 64 bits |

The apparent execution model uses 32-bit pointers. That pointer width is supported by the register-oriented representation and four-byte pointer-related types, but the platform ABI is not independently provided.

The table lookup uses the selector directly as an index. Its effective location is the table base plus 52 times the selector, plus 16 bytes. There is no visible subtraction to convert a one-based selector to a zero-based index.

### Outputs

Four explicit 64-bit return values appear:

| Condition | Return value |
|---|---|
| Selector equals zero | `0x082bf8d400000004` |
| Nonzero selector equals the exclusion value | `0x082bf7ec00000004` |
| Selected record pointer is null | `0x082bf82000000004` |
| Successful sequence reaches its final return | `0x082bf85c00000004` |

All four have low 32 bits equal to 4. Their upper 32 bits differ. The meaning of either half is unknown; the function does not establish a conventional success/error interpretation.

The supplied 64-bit return representation does not establish how the result is transported by the actual ABI.

### Preconditions

For a path to complete normally:

- Every external reference actually used on that path must be valid for the access performed.
- Interrupt handling must return control to the next visible operation.
- Each invoked dependency must return normally.
- The selected table field must be readable whenever table lookup is reached.
- Addresses passed to dependencies must satisfy those dependencies’ unknown requirements.

There is no visible validation of table bounds, record size, alignment, permissions, or selector range beyond the zero and exclusion checks.

## Behavior

### 1. Selector acquisition

The function first saves the incoming `r3` value into its local pointer slot. It then calls F-459a7f325a37 without visible arguments and uses its returned 32-bit integer as the selector.

The selector is obtained once. There is no explicit retry or second selector query.

The external exclusion value and table information are used after this call. No evidence establishes that F-459a7f325a37 preserves them.

### 2. Zero-selector path

If the selector equals zero, the function requests software interrupt number `0x123456`.

If execution resumes and reaches the explicit return, it returns `0x082bf8d400000004`.

On this path, the visible body does not read the exclusion value, look up a record pointer, or invoke either of the other two dependencies.

### 3. Exclusion path

For a nonzero selector, the function reads the 32-bit exclusion value at byte offset 12 from its associated external base and compares it with the selector.

If they are equal, it requests software interrupt number `0x123456` and, if execution continues, returns `0x082bf7ec00000004`.

The exclusion comparison precedes table access. Consequently, an excluded selector does not require a valid corresponding table entry under the visible control flow.

### 4. Table lookup and null-record path

For a nonzero, nonexcluded selector, the function locates the pointer field at:

- The table base,
- Plus 52 bytes per selector unit,
- Plus a fixed 16-byte field offset.

It reads the record pointer from that location and stores it in the local pointer slot, replacing the saved incoming value.

If the loaded pointer is null, the function requests software interrupt number `0x123456` and, if execution continues, returns `0x082bf82000000004`.

A null pointer is the only visible record-pointer validity check.

Negative selectors other than the excluded value are not explicitly rejected. Neither are large positive selectors. Their resulting effective addresses and possible faults depend on the target’s address arithmetic and memory mapping.

The supplied representation does not settle multiplication overflow, address wrapping, or other machine-level details sufficiently to define a portable arithmetic rule. A replacement must not invent a bounds check or mathematical unbounded-index interpretation and call it established behavior.

### 5. Successful-path dependency sequence

For a nonnull selected record pointer, the function first requests software interrupt number `0x123456`.

If execution continues, it makes five calls to F-88996a971ccf in the following order. The offsets below are byte offsets from the pointer held in the local slot when each call’s arguments are formed.

| Call | First argument | Remaining visible arguments |
|---|---|---|
| 1 | First format-like reference | Record addresses at offsets 4, 8, 12, and 16 |
| 2 | Second format-like reference | Record addresses at offsets 20, 24, 28, and 32 |
| 3 | Third format-like reference | Record addresses at offsets 36, 40, 44, and 48 |
| 4 | Fourth format-like reference | Record address at offset 52; address of the local pointer slot; record address at offset 56 |
| 5 | Fifth format-like reference | Record address at offset 60 |

The record-derived arguments are addresses, not explicit loads of the 32-bit words at those addresses. The supplied body therefore does not itself establish the values F-88996a971ccf consumes or emits.

The fourth call is distinct: its middle non-format argument is the address of the local pointer slot, not another address within the record. This exposes a pointer-valued local object to the dependency. No read-only contract is supplied, so the dependency cannot be assumed to leave that slot unchanged.

If that call changes the local pointer slot, the fifth call forms its record-relative address from the changed pointer. The final explicit record-word read also uses the pointer then held in that slot. There is no visible restoration of the original record pointer.

Arguments to an individual call do not have a documented left-to-right evaluation order. Their observable call positions are established; a finer ordering of argument evaluation is not.

### 6. Final record-word processing

After the fifth F-88996a971ccf call returns, the function requests software interrupt number `0x123456`.

It then explicitly reads the 32-bit word at offset zero from the pointer currently held in the local pointer slot and supplies that word as the first argument to F-5d4d7ea4b919. The second argument is the label-like reference whose visible text identifies `cpsr`.

After that dependency returns, the function requests software interrupt number `0x123456` once more.

If execution reaches the final return, the result is `0x082bf85c00000004`.

No return value from F-88996a971ccf or F-5d4d7ea4b919 is visibly inspected.

## Side effects

### Observable ordering

A normally completing successful path has this visible sequence:

1. Call F-459a7f325a37.
2. Read and compare the exclusion value.
3. Read and save the selected record pointer.
4. Request a software interrupt.
5. Make five ordered calls to F-88996a971ccf.
6. Request a software interrupt.
7. Read the current record’s first 32-bit word and call F-5d4d7ea4b919.
8. Request a software interrupt.
9. Return the successful-path constant.

Each early-return path requests one software interrupt before returning its path-specific constant. A normally completing successful path requests three.

Software interrupts are not established no-ops. They may transfer control, fail to return, or have externally visible effects. The interrupt number alone does not identify an operating-system service or diagnostic action.

### Memory access and mutation

The visible body writes its local pointer slot. It does not explicitly write record contents or table entries.

The dependencies receive record addresses, and one receives the address of the local pointer slot. Without their contracts, no guarantee can be made that the selected record remains unchanged, that external state is preserved, or that the local slot retains its original selected value.

The function does not snapshot the record’s words before calling dependencies. In particular, its explicit offset-zero word read occurs after all five F-88996a971ccf calls and the following interrupt.

There is no explicit allocation, deallocation, ownership transfer, locking, or cleanup operation in the supplied body. This establishes only the absence of such explicit operations here; it does not establish what dependencies do.

### Faults and abnormal completion

There is no visible local recovery from:

- An invalid external reference.
- An invalid table-field address.
- An invalid nonnull record pointer.
- A dependency that fails or does not return.
- An interrupt that does not resume normal execution.
- A local pointer slot changed to an invalid value before later use.

Such outcomes must not be silently converted into one of the four explicit return values.

## Dependencies

### F-459a7f325a37

Called once, with no visible arguments, before branch selection.

Its 32-bit result drives the zero test, exclusion comparison, and table index. Its purpose, possible result range, global effects, and failure behavior are unknown.

### F-88996a971ccf

Called five times on the successful path only.

Each call receives a format-like reference followed by the address arguments listed above. Visible fragments associated with the later references suggest register-oriented formatting, including stack-pointer, program-counter, and link-register roles. They do not establish a complete output grammar.

It is unknown whether this dependency reads, writes, retains, or otherwise interprets the supplied addresses. Exact output, formatting syntax, ownership effects, and error reporting remain undocumented.

### F-5d4d7ea4b919

Called once on the successful path, after the five preceding dependency calls and an intervening interrupt.

It receives a 32-bit value loaded from the current record pointer and a label-like reference associated with `cpsr`. A processor-status presentation role is inferred, not proven. Its output and side effects are unknown.

### Software-interrupt mechanism

Every explicit software interrupt uses number `0x123456`.

Its service semantics and register, memory, and control-flow effects are not supplied. The visible continuation describes behavior only if execution resumes normally.

## Acceptance examples

These are conditional acceptance scenarios derived from static evidence, not executed tests.

1. **Zero selector takes precedence.**  
   Given that F-459a7f325a37 returns zero and the interrupt resumes normally, the function returns `0x082bf8d400000004`. The visible path does not access the exclusion field or record table and does not call either remaining dependency.

2. **Excluded nonzero selector avoids table access.**  
   Given selector 7 and an exclusion value of 7, the function requests one interrupt and returns `0x082bf7ec00000004`, provided execution resumes. A valid table entry for selector 7 is not needed by this path.

3. **Table indexing uses the selector directly.**  
   Given nonzero, nonexcluded selector 2, the pointer field is read 120 bytes from the table base: two 52-byte strides plus 16 bytes. It is not read from the selector-1 entry.

4. **Null selected pointer takes the third early exit.**  
   Given a valid table-field read yielding null, the function requests one interrupt and returns `0x082bf82000000004` if execution continues. No F-88996a971ccf or F-5d4d7ea4b919 call occurs.

5. **Stable-pointer successful path preserves call order.**  
   Given valid state, normally returning dependencies and interrupts, and an unchanged local pointer slot, the five F-88996a971ccf calls receive the listed record-relative addresses in order. F-5d4d7ea4b919 subsequently receives the record’s offset-zero word as it exists at the later read. The final result is `0x082bf85c00000004`.

6. **Changing the exposed pointer affects later use.**  
   Given that the fourth F-88996a971ccf call replaces the local pointer slot with another valid record pointer, the fifth call receives that replacement pointer plus 60 bytes. The final offset-zero read also uses the replacement pointer, assuming no subsequent change.

7. **A negative selector has no dedicated rejection.**  
   Given a negative selector unequal to the exclusion value, the visible path proceeds toward table lookup. Safe access, a fault, or address wrapping cannot be selected as the required outcome without additional platform evidence.

## Uncertainties

The following are **documented_only**:

- The original function signature and meaning of incoming `r3`.
- Exact pointer width, ABI return convention, endianness, and alignment rules.
- Machine-level overflow and effective-address behavior for table indexing.
- External reference initialization, table bounds, and valid selector range.
- Record ownership, allocation size, field meanings, and lifetime.
- Complete contents and semantics of the format-like references.
- Dependency reads, writes, retention, allocation, release, and global effects.
- Software-interrupt semantics and whether each interrupt returns.
- Whether pointer-slot mutation is intended behavior or merely possible through the supplied address.
- The meaning of the four return constants.
- Concurrency and synchronization requirements.

The inferred diagnostic purpose does not resolve these uncertainties.

## Evidence

Established static evidence consists of the supplied recovered function representation: its 32-bit selector and word operations, branch order, exclusion-field offset, table stride and pointer-field offset, null check, ordered dependency arguments, exposure of the local pointer slot, software-interrupt sites, final word load, and four 64-bit return constants.

Register-related text fragments and the status-oriented label support the inferred diagnostic interpretation, but not exact output behavior.

No dependency implementations, runtime traces, ABI specification, memory-map documentation, or executed tests were supplied. No runtime tests have been performed.
