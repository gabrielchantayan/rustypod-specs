<!-- Trusted provenance: F-de0abedbfa5b; snapshot 97be799c828e38f6392d8a35b84a1617ea63f9ce2d9553d7fd9e0b775b9b08db; static analysis only. -->
# F-de0abedbfa5b

## Purpose

F-de0abedbfa5b dispatches an event identifier to a media-interface action. It recognizes a fixed, ordered set of event labels, conditionally suppresses their actions using an accompanying 32-bit value, and returns a 32-bit recognition result.

The principal distinction is **recognized versus unrecognized**, not action success. A recognized event returns 1 even when its action is suppressed, its handler’s return value is ignored, or it has no action in this function. An unrecognized event returns 0 after exhausting the comparisons.

The labels suggest highlighting, selection, playback, locking, backlight, and sleep operations. Those meanings are inferred intent. The established behavior is the comparison order, argument forwarding, conditional calls, and explicit state writes described below.

## Interface

### Inputs and result

| Item | Contract |
|---|---|
| Receiver | A pointer to an object. It is forwarded unchanged to ordinary action dependencies. Certain paths access its dispatch table or a byte at offset 0xB0. |
| Event identifier | An opaque 32-bit value passed as the first argument to every event-comparison call. Label names suggest a string-like identifier, but its representation and validity rules are not established. |
| Event value | A 32-bit integer used for equality checks against shared state and forwarded to action dependencies. Its interpretation as an index, handle, event payload, or sentinel is unknown. |
| Result | A 32-bit value: 1 for a recognized event, 0 if every comparison reports nonmatching. This description presumes invoked dependencies return normally. |

No arithmetic is performed on the event value. Negative values, zero, and boundary bit patterns receive no special treatment beyond equality comparisons and forwarding. One recognized event substitutes zero for the value passed to its action.

### Receiver layout used directly

The supplied representation uses 32-bit integer and dispatch-table fields:

- At receiver offset 0, a dispatch-table pointer is read on the remote-video action path.
- At byte offset 0x124 within that table, a callable entry is read. This is an offset within the table, not within the receiver.
- At receiver byte offset 0xB0, one byte is written for two highlight events.

The byte offset 0xB0 follows from 0x2C units of a four-byte integer. The explicit write has width one byte; it does not assign a whole word.

No complete object size, alignment contract, table size, or ownership policy is supplied. At minimum, the particular locations accessed by an executed path must be valid. There is no explicit null-pointer check.

### Shared state

Two distinct 32-bit shared locations affect suppression:

- **Remote gate:** consulted only after recognition of `HandleRemotePlayPauseForVideo`.
- **Shared gate:** read after the `HandleMusicHilited` comparison, provided the remote-video comparison did not match.

Neither location’s initial value or application meaning is established. They must not be assumed equal or constant.

## Behavior

### Recognition and comparison order

Every comparison invokes the same external comparison dependency, identified here as **U-01**. U-01 receives the event identifier followed by the candidate event label. A result of zero selects that label; any nonzero result advances to the next comparison.

The function stops comparing labels at the first zero result. Consequently, the ordering below remains significant even if U-01 has stateful or nonstandard matching behavior.

The following table is in comparison order. Except where explicitly distinguished, the selected action is subject to the shared-gate rule.

| Event label | Selected action when not suppressed |
|---|---|
| `HandleRemotePlayPauseForVideo` | Invoke U-02 through the receiver’s dispatch table; uses the remote gate instead of the shared gate |
| `HandleMusicHilited` | F-6f3e0e96361a with the original event value; special gate-read rule below |
| `HandleVideosHilited` | F-16f0e29b138e |
| `HandlePodcastsHilited` | F-6b95b88ebbda |
| `HandleiTunesUHilited` | F-f13086bc6f2a |
| `HandleGenericHilited` | Write byte value 0 at receiver offset 0xB0 |
| `HandlePhotosHilited` | F-4bb453fa7166 |
| `HandleNowPlayingHilited` | F-cd1d276c146c |
| `HandleNoBookMusicHilited` | F-f484c6443693 |
| `HandleAudiobooksHilited` | F-fdc37c1d9e13 |
| `HandleArtistsHilited` | F-7dede0a99c9d |
| `HandleGenresHilited` | F-b0219a0b9812 |
| `HandleAlbumsHilited` | F-18de7345c856 |
| `HandleCompilationsHilited` | F-6e438fed5eeb |
| `HandleComposersHilited` | F-a77b02a7bb76 |
| `HandleSongsHilited` | F-747af0926326 |
| `HandlePlaylistsHilited` | F-2f4cc473b82a |
| `HandleGeniusHilited` | Write byte value 1 at receiver offset 0xB0 |
| `HandleGeniusMixesHilited` | F-6f3e0e96361a with event value replaced by 0 |
| `HandleTVShowsHilited` | F-40fab6cd2c21 |
| `HandleMusicVideosHilited` | F-c85f277d4d26 |
| `HandleMoviesHilited` | F-d39c1d8c8f40 |
| `HandleVideoPlaylistsHilited` | F-0f25dcdf689a |
| `HandleRentalsHilited` | F-01fb4b07f44e |
| `HandleMusicSelected` | F-3c09594d1a82 |
| `HandleVideosSelected` | F-3abc7c71e3aa |
| `HandlePodcastsSelected` | F-84663833c9b3 |
| `HandleiTunesUSelected` | F-886ddf11afa2 |
| `HandlePhotosSelected` | F-0e3f9346656b |
| `HandleCoverFlowSelected` | F-d168f2772280 |
| `HandleSongsSelected` | F-7225193bb721 |
| `HandleAlbumsSelected` | F-4d36497e79b4 |
| `HandleCompilationsSelected` | F-1cd58634bc0f |
| `HandleArtistsSelected` | F-70460eed8c20 |
| `HandleGenresSelected` | F-b427a33253b9 |
| `HandleComposersSelected` | F-1eb22ff58e3c |
| `HandleAudiobooksSelected` | F-641996019781 |
| `HandleShuffleSongsSelected` | F-1b2d90b3a3de |
| `HandlePlaylistsSelected` | F-66c31f8db1bf |
| `HandleNowPlaying` | F-6e5e893ffc8b |
| `HandleGotoGeniusMixes` | F-4f2756ba2a9d |
| `HandleTVShowsSelected` | F-66c9e3357412 |
| `HandleMusicVideosSelected` | F-78f11f03944e |
| `HandleMoviesSelected` | F-362e7333e740 |
| `HandleVideoPlaylistsSelected` | F-0ba0a0089de6 |
| `HandleRentalsSelected` | F-f15870402da7 |
| `HandleMainMenuPlayPause` | F-e59e556d79ab |
| `HandleLock` | F-c9b00b68f2e1 |
| `HandleBacklightSelected` | F-6ac70711bbb5, then F-adf0a137c932 |
| `HandleSleepSelected` | F-6ac70711bbb5, then F-970e8a682a82 |
| `HandleNikePlusSelected` | No action and no gate comparison |

### Remote-video path

The first operation is comparison against `HandleRemotePlayPauseForVideo`.

If selected:

1. The event value is compared against the remote gate, read after U-01 returns.
2. Equality suppresses the action.
3. Inequality causes an indirect call through the receiver’s table entry at byte offset 0x124. The arguments are the receiver and original event value.
4. The function returns 1.

This path does not proceed to the music comparison or explicitly read the shared gate. Any changes U-01 makes to the remote gate before returning may affect the decision.

### Shared-gate sampling

If the remote-video comparison does not select its label, the next comparison is against `HandleMusicHilited`. After that comparison returns, the shared gate is read and retained for the later dispatch paths.

For a selected `HandleMusicHilited`, the supplied representation also shows a separate reference to the shared gate for its equality decision. If equal, the function returns 1 without invoking an action. Otherwise it invokes F-6f3e0e96361a with the receiver and original event value, then returns 1.

For every subsequent gated event, the decision uses the value retained immediately after the music comparison. It does not explicitly reload the shared gate after later comparison calls. Thus a later U-01 call that changes the shared location does not, in this representation, replace the already retained gate value.

The two shared-gate references on the music path are statically represented accesses. Whether they correspond to distinct physical loads, and what concurrent updates could be observed, is not established.

### Ordinary dispatch and suppression

For each ordinary gated event:

- Equality between the original event value and the applicable gate suppresses the selected action.
- Inequality causes exactly the action listed in the table.
- Direct action dependencies receive the receiver followed by the original event value, unless the table specifies the zero substitution or an argumentless sequence.
- After a normally returning action, the dispatcher returns 1.
- A dependency’s return value is not used to determine the dispatcher’s result.

Suppression applies after recognition. It does not skip earlier comparisons or turn a recognized event into an unrecognized one.

### Special state and argument behavior

`HandleGenericHilited` and `HandleGeniusHilited` perform their one-byte writes only when the original event value differs from the retained shared gate. The former writes 0; the latter writes 1. There is no explicit read of the old byte, toggle, or check for whether the byte already has the target value.

`HandleGeniusMixesHilited` first applies the gate comparison to the **original** event value. If not suppressed, it invokes F-6f3e0e96361a with zero as the second argument. This is distinct from testing zero against the gate.

`HandleBacklightSelected` and `HandleSleepSelected` invoke two argumentless dependencies in order when not suppressed. They do not pass the receiver or event value. There is no second gate check between the calls. The second call is reached only if the first returns normally.

`HandleNikePlusSelected` is recognized unconditionally once its comparison returns zero. It neither compares the event value to a gate nor performs a direct action.

### Unrecognized events and errors

If every comparison is nonzero, the function returns 0. It performs no explicit receiver access on that path, but all preceding comparison calls and the shared-gate read still occur.

There is no explicit error reporting, input validation, recovery, or rollback. Invalid input behavior depends on which paths are reached and on dependency requirements. An indirect-call fault or a dependency that does not return prevents the ordinary result described above; no handling for such conditions is shown.

## Side effects

The direct receiver mutations are limited to one-byte writes at offset 0xB0 for the two specified highlight labels. The dispatcher does not explicitly allocate, copy, free, retain, or release an object or identifier.

This is not a guarantee of global purity, receiver preservation, or ownership preservation:

- U-01 executes repeatedly before dispatch and may have effects not shown here.
- Direct action dependencies and U-02 may mutate receiver state, shared state, external devices, or memory.
- No evidence establishes whether any dependency takes ownership, releases storage, or retains its arguments.
- The dispatcher makes no explicit defensive copy of the receiver or identifier.
- No synchronization or reentrancy protection is shown.

Observable ordering includes comparisons before the selected action, the stated timing of shared reads, and the order of paired argumentless calls. There are no explicit dispatcher writes to either gate.

A suppressed action means only that the dispatcher omits that branch’s action. It does not mean the invocation has no side effects, because the comparisons have already executed.

## Dependencies

The dispatch table above gives every directly selected action dependency by its supplied opaque function ID. Unless marked otherwise, each is invoked with two arguments: receiver and 32-bit event value.

Two dependencies lack supplied mapped IDs. The following are document-local opaque identifiers, not claims about published identities:

| ID | Established interface and use | Unknowns |
|---|---|---|
| U-01 | Comparison dependency; receives the original event identifier and one candidate label; zero selects the label | Identifier representation, comparison semantics, memory access rules, side effects, ownership |
| U-02 | Indirect action obtained through the receiver’s dispatch table at offset 0x124; receives receiver and event value | Concrete target identity, effects, return contract, ownership |

F-6ac70711bbb5, F-adf0a137c932, and F-970e8a682a82 are called without explicit arguments in this function. No action dependency’s return value is consumed.

Event names do not establish the behaviors of their target dependencies. In particular, names suggesting sleep or backlight changes are not proof of the corresponding hardware effects.

## Acceptance examples

These are conditional contract examples derived from the supplied static representation, not executed tests. Assume all invoked dependencies return normally.

1. **Remote action permitted.** U-01 first returns zero for `HandleRemotePlayPauseForVideo`; the event value differs from the remote gate observed afterward. U-02 receives the receiver and unchanged value. The result is 1, with no subsequent label comparisons.

2. **Remote action suppressed.** The same label is selected, but the event value equals the observed remote gate. U-02 is not invoked. The result remains 1.

3. **Ordinary selected event.** Every comparison preceding `HandleAlbumsSelected` is nonzero, and its comparison returns zero. If the event value differs from the retained shared gate, F-4d36497e79b4 receives the receiver and original value. Later labels are not compared.

4. **Recognized but suppressed.** Under the preceding selection conditions, equality with the retained shared gate omits F-4d36497e79b4 and still yields 1.

5. **Gate changes during later comparisons.** The retained shared gate is 7 after the music comparison. A later comparison changes the shared location to 9 before selecting `HandleSongsSelected`. With event value 7, the action is suppressed; with event value 9, F-7225193bb721 is invoked. These outcomes depend on the retained value, not the later contents.

6. **Byte-state assignment.** `HandleGeniusHilited` is selected and not suppressed. The dispatcher writes one byte of value 1 at receiver offset 0xB0. It does not explicitly overwrite the following three bytes. A selected, nonsuppressed `HandleGenericHilited` instead writes byte value 0.

7. **Zero substitution after gating.** `HandleGeniusMixesHilited` is selected with original event value 12 and retained shared gate 0. F-6f3e0e96361a receives zero, despite the gate also being zero. If the original value were 0, the action would instead be suppressed.

8. **Ordered paired action.** `HandleSleepSelected` is selected and not suppressed. F-6ac70711bbb5 executes before F-970e8a682a82. A shared-gate change made by the first call does not trigger another gate check.

9. **Recognized actionless label.** Only the final comparison selects `HandleNikePlusSelected`. The result is 1 regardless of the event value’s equality with either gate, with no direct action afterward.

10. **No match.** All comparisons return nonzero. The result is 0. No selected handler or explicit receiver-byte write occurs; comparison side effects remain possible.

11. **First-match precedence.** If U-01 would select both a highlight label and a later selection label, only the earlier label is reached and handled.

## Uncertainties

The following remain **documented_only**:

- U-01’s concrete identity and whether it implements ordinary string equality, case folding, pointer identity, or another matching relation.
- Exact label storage contents beyond their supplied symbolic names, including encoding and termination.
- Both gate values, their initialization, mutability, and semantic meanings.
- Event-value interpretation and any restrictions imposed by callers or action dependencies.
- Full receiver layout, platform ABI, pointer validity rules, byte order, and concurrency memory model.
- Whether the two represented shared-gate references on the music path are separate machine-level reads.
- All transitive state changes, failure modes, and ownership decisions inside dependencies.
- The concrete target of U-02.
- Application-level meaning of a returned 1 beyond recognition by this dispatcher.

The apparent roles of highlighting, selection, playback control, and power-related actions are inferred from labels, not independently established behavior.

## Evidence

The supplied static function representation establishes the ordered zero-result comparisons, conditional gates, direct dependency selections, one-byte assignments, argument substitution, paired-call ordering, and explicit 0/1 returns described here. The supplied dependency map establishes the opaque identities used for direct action calls.

This contract does not treat embedded labels as instructions or as proof of downstream behavior. It separates visible dispatcher operations from inferred application intent and unknown dependency semantics.

No implementation access, independent binary inspection, or runtime tests have been performed. The acceptance examples are conditional predictions for later verification, not observed execution results.
