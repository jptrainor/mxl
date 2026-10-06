# libmxl API consistency tracker

Findings from comparing the public API documentation (`lib/include/mxl/flow.h`, `lib/include/mxl/mxl.h`)
with the libmxl implementation, plus a review of consistency between the grain and sample paths and
across the C entry points.

- **Upstream repository:** [dmf-mxl/mxl](https://github.com/dmf-mxl/mxl)
- **Pinned commit:** [`c66150a`](https://github.com/dmf-mxl/mxl/tree/c66150ac387cf597f65c40acb1e908d6503674d7)
  (all line numbers refer to this commit)
- **Method:** read the code, then used the regression tests as evidence of maintainer intent.
  - A test that asserts current behaviour suggests it is intended.
  - A test that asserts documented behaviour suggests the code is wrong.
  - Findings have not been confirmed by running tests.
- **Tests reviewed:** `lib/tests/test_flows.cpp`, `test_flow_sync_groups.cpp`, `test_flows_timing.cpp`,
  `test_read_write_conflict.cpp`, `test_instance.cpp`, `lib/internal/tests/test_instance.cpp`,
  `lib/internal/tests/test_options.cpp`.
- **Tests not reviewed:** `lib/internal/tests/test_flowmanager.cpp`, `test_domainwatcher.cpp`,
  `test_sharedmem.cpp`, `lib/tests/test_time.cpp`, `lib/tests/fabrics/`.

## How to use this file

- Tick an item by changing `- [ ]` to `- [x]` and committing.
- Record upstream issues and PRs on the `Status:` line, e.g. `dmf-mxl/mxl#719`, `dmf-mxl/mxl#728`.
- For `INT-*` items, record the maintainers' decision on the `Decision:` line. It determines whether
  the item becomes a bug fix (`BUG`) or a documentation update (`DOC`).
- Status values: `open` · `issue filed` · `PR open` · `merged` · `backported` · `won't fix` · `obsolete`

## Categories

| Prefix | Category | Ranking |
|---|---|---|
| `BUG` | Looks like a real bug: the code doesn't do what the header promises, or does something unsafe | Most egregious first |
| `DOC` | Documentation needs updating: the behaviour is reasonable but the header is missing it, wrong, or contradicts itself | Most out of date first |
| `INT` | Needs maintainer intent: the header and code disagree, and which is right is a design decision | Most clearly out of sync first |
| `DC-*` | Discrete/Continuous Consistency: comparison of the grain and sample paths (A) and error handling in the C entry points (B) | Same rankings as above |

## Summary

| Section | Items |
|---|---|
| BUG | 7 |
| DOC | 14 |
| INT | 3 |
| DC-BUG | 10 |
| DC-DOC | 3 |
| DC-INT | 7 |
| **Total** | **44** |

## Changes from the test review

| Item | Before | After | Reason |
|---|---|---|---|
| Open invalidates skipped grains | INT-3 | DOC-7 | "Video Flow : Jump forward" asserts invalidation at open, with an explanatory comment |
| Grain commit wakes readers on partial commits | INT-4 | DOC-6 | "Video Flow : Slices" depends on partial reads after each partial commit |
| Writers shared within an instance | INT-5 | INT-3 | Renumbered only |
| Gap handling timing differs | DC-INT-1 | DC-DOC-1 | Both timings are asserted on purpose, with comments |
| Reader wake-ups differ | DC-INT-2 | DC-DOC-2 | Batched wake-up for samples and per-commit wake-up for grains are both tested |
| Batch-size hints | DC-DOC-1 | DC-DOC-3 | Renumbered; wording changed because tests use the hint for discrete flows |
| `MXL_ERR_UNKNOWN` for invalid input | DC-INT-5 | DC-INT-1 | Ranked first: tests assert the exact code that `mxl.h` says not to check for |
| Other DOC, DC-INT items | — | renumbered | To fill the gaps left by the moves |

## Changes from external review

| Item | Before | After | Reason |
|---|---|---|---|
| Sync group returns OK for incomplete grains | — | BUG-1 (new) | Found in an external review; verified against the code. Breaks the documented promise of the sync-group API in normal use |
| Sync group returns OK for overwritten data | — | BUG-2 (new) | Found in an external review; verified against the code. Bypasses the reader's too-late check |
| `mxlFlowWriterCommitGrain` can move `headIndex` backwards | BUG-1 | BUG-3 | Renumbered |
| `mxlFlowWriterCommitGrain` copies `mxlGrainInfo` unchecked | BUG-2 | BUG-4 | Renumbered |
| Invalid-flagged commit doesn't finish the grain | BUG-3 | BUG-5 | Renumbered |
| `mxlFlowWriterGetGrainInfo` can return another grain's info | BUG-4 | BUG-6 | Renumbered |
| `mxlCreateFlowSynchronizationGroup` null check | BUG-5 | BUG-7 | Renumbered |

## Already addressed

- [x] **REF-1: `validSlices` not reset when a new grain is opened.** Fixed.
  - Issue `dmf-mxl/mxl#717` (closed) · PR `dmf-mxl/mxl#718` (merged) · backports `dmf-mxl/mxl#725` (v1.1), `dmf-mxl/mxl#726` (v1.0)
  - Test: "Video Flow : validSlices on grain open" ([TF] L1263-L1337).

---

## 1. Looks like a real bug (most egregious first)

- [ ] **BUG-1: `mxlFlowSynchronizationGroupWaitForDataAt` can return `MXL_STATUS_OK` for a grain that isn't complete.** _(new)_
  - **What the API promises:**
    - `mxlFlowSynchronizationGroupAddReader` adds a grain reader so the group waits for the grain "to become
      fully available" ([FLOWH] L527-L550).
    - `mxlFlowSynchronizationGroupAddPartialGrainReader` makes the group wait for "at least
      `minValidSlices`" ([FLOWH] L552-L573).
    - `mxlFlowSynchronizationGroupWaitForDataAt` returns `MXL_STATUS_OK` only "if the data corresponding to
      the specified timestamp has become available" ([FLOWH] L589-L604).
  - **What the code does:**
    - For each reader, `waitForDataAt` converts the timestamp to an index. It calls the reader's wait method
      only if that index is greater than the flow's current `headIndex` ([SYNC] L84-L87).
    - If the index is less than or equal to `headIndex`, the reader is skipped and treated as ready. When
      every reader is skipped or ready, the function returns `MXL_STATUS_OK` ([SYNC] L131).
    - The slice requirement is only checked inside `DiscreteFlowReader::waitForGrain` ([SYNC] L91-L94,
      [DR] L183-L184), so a skipped reader's requirement is never checked.
  - **Why `headIndex` doesn't mean "complete":**
    - A grain commit moves `headIndex` to the grain's index on every commit, including partial ones
      ([DW] L167).
    - This is intended: "Video Flow : Slices" asserts `headIndex == index` after each partial commit
      ([TF] L612-L618).
    - So `headIndex == N` only means "at least one commit to grain N has happened", not "grain N is
      complete".
  - **Failure scenario:**
    1. A writer opens grain N and commits 1 of 1080 slices. `headIndex` becomes N.
    2. A consumer creates a group, adds a video reader with `mxlFlowSynchronizationGroupAddReader` (full
       grain required), and calls `WaitForDataAt` with the timestamp of grain N.
    3. The group sees `expectedIndex == headIndex`, skips the reader, and returns `MXL_STATUS_OK`.
    4. The consumer reads grain N and gets a frame with only 1 valid line.
    - The same happens with `AddPartialGrainReader` whenever the committed slice count is below
      `minValidSlices`.
  - **Why it matters:**
    - The sync group exists so a consumer can wait for video, audio and data for the same timestamp to be
      ready before processing them together.
    - This bug makes the group return early in exactly the case it is meant to handle: a writer that commits
      in slices (`maxCommitBatchSizeHint` < `totalSlices`).
    - It needs no unusual sequence of calls. It's a race between normal slice-by-slice writing and a normal
      wait, and it can't be detected from the return code.
    - Callers that then use `mxlFlowReaderGetGrainNonBlocking` (full grain) would get
      `MXL_ERR_OUT_OF_RANGE_TOO_EARLY` immediately after the group said the data was ready. Callers that read
      the payload directly would process half-written frames.
  - **Why the tests don't catch it:**
    - "Synchronization group : Repeated waits" ([TSG] L24-L36, L75-L107) only waits for indexes ahead of
      `headIndex`, and the writer commits complete grains (`validSlices = totalSlices`) in one step.
    - The reader's wait method is always called in that test, so the skip path is never taken. No test
      combines partial commits with a sync group.
  - **Possible fix:**
    - For discrete readers, always call `waitForGrain(expectedIndex, minValidSlices, deadline)`. It already
      returns immediately when the grain meets the requirement ([DR] L215-L237), so the fast path saves
      little.
    - Alternatively, keep the shortcut but also check the grain's `validSlices`, or its invalid flag, before
      skipping the reader.
  - **Suggested regression test:** open a grain, commit fewer slices than `totalSlices`, add a full-grain
    reader to a group, and call `WaitForDataAt` with a short timeout. Expect a timeout-type error, not
    `MXL_STATUS_OK`. Repeat with `AddPartialGrainReader` and a `minValidSlices` above the committed count.
  - Related: BUG-2 (same code path), DOC-4 (sync-group timeout code), DOC-6 (partial commits wake readers).
  - Status: open

- [ ] **BUG-2: `mxlFlowSynchronizationGroupWaitForDataAt` can return `MXL_STATUS_OK` for data that has already been overwritten.** _(new)_
  - **What the API promises:** `MXL_STATUS_OK` "if the data corresponding to the specified timestamp has
    become available" ([FLOWH] L589-L604). Data that has been overwritten in the ring buffer is no longer
    available.
  - **What the readers do on their own:**
    - Both readers reject indexes that are too old with `MXL_ERR_OUT_OF_RANGE_TOO_LATE`.
    - Discrete reader: the readable window is `headIndex − grainCount + 2` … `headIndex` ([DR] L176-L205).
      The tail slot is deliberately kept back for the writer, which "Video Flow : tail read during open
      write" asserts ([TRW] L84-L105).
    - Continuous reader: the window is the most recent half of the buffer ([CR] L125-L153). "Audio Flow :
      tail read during open write" asserts this boundary ([TRW] L283-L312).
  - **What the sync group does:** the skip described in BUG-1 ([SYNC] L86) applies to every index
    `<= headIndex`, however old. For an index older than the readable window, the reader is never asked,
    so its too-late check is bypassed and the group returns `MXL_STATUS_OK`.
  - **Failure scenario:**
    1. A consumer falls behind, for example because of a processing stall, by more than the flow's history
       (200 ms by default).
    2. It calls `WaitForDataAt` with the timestamp it wants next.
    3. That index is far below `headIndex`, so every reader is skipped and the call returns
       `MXL_STATUS_OK` immediately.
    4. The consumer then reads, and either gets `MXL_ERR_OUT_OF_RANGE_TOO_LATE` from the reader or, if it
       accesses slots directly, gets newer data that has replaced the old.
  - **Why it matters:**
    - The group's result is inconsistent with the readers it contains. The group says "ready"; each reader
      would say "too late".
    - A consumer that relies on the group to tell it whether to read or to skip ahead can't detect that it
      has fallen behind, so it can't recover cleanly by jumping to the current index.
    - It affects both discrete and continuous flows.
  - **Why ranked below BUG-1:**
    - It only happens when a consumer is already behind by more than the flow's history, which is a fault
      condition, whereas BUG-1 happens during normal slice-by-slice writing.
    - The consumer's next read usually reports too-late, so the error tends to surface one call later
      rather than being silently wrong.
    - It's still a bug: the documented contract says "available", and the group's result disagrees with
      the reader's.
  - **Note on intent:** the shortcut may have been meant as "data has arrived at some point". If the
    maintainers want that meaning, the header must say so and the result should not be `MXL_STATUS_OK` for
    data that can no longer be read. Either way, the code and the header disagree.
  - **Why the tests don't catch it:** "Synchronization group : Repeated waits" never waits for an index
    behind `headIndex` ([TSG] L75-L107).
  - **Possible fix:**
    - Always call the reader's wait method (`waitForGrain` / `waitForSamples`). Both already return
      `MXL_ERR_OUT_OF_RANGE_TOO_LATE` for indexes outside the window ([DR] L202-L205, [CR] L153).
    - If the shortcut is kept for performance, also check that the index is still inside the reader's
      readable window before skipping.
  - **Suggested regression test:** fill a flow beyond its history, then call `WaitForDataAt` with a timestamp
    older than the readable window. Expect `MXL_ERR_OUT_OF_RANGE_TOO_LATE`. Do this for a grain flow and a
    sample flow.
  - Related: BUG-1 (same code path).
  - Status: open

- [ ] **BUG-3: `mxlFlowWriterCommitGrain` can move `headIndex` backwards.** _(was BUG-1)_
  - **Finding:**
    - Every commit sets `headIndex = _currentIndex` ([DW] L167). A partial commit doesn't update
      `_lastCommittedIndex`, so after cancelling a partly committed grain the writer can open and commit a
      lower index, and readers see the head go backwards.
    - The header says the head moves only "IF this grain is the new head".
  - **Test evidence:** "Video Flow : Slices" ([TF] L612-L618) asserts `headIndex == index` after each partial
    commit, so moving the head on a partial commit is intended. No test covers cancelling and then committing
    a lower index, and nothing suggests the head is meant to move backwards. Stays BUG.
  - **Failure scenario:**
    1. The writer completes grain N. `_lastCommittedIndex` and `headIndex` are both N.
    2. The writer opens grain N+5 and commits some slices. `headIndex` becomes N+5, and readers can see the
       partial grain.
    3. The writer cancels N+5. `_lastCommittedIndex` is still N.
    4. The writer opens grain N+2. This is allowed because N+2 > N. Grain N+1 is marked invalid as a skipped
       grain.
    5. The writer completes N+2. `headIndex` is set to N+2, which is lower than N+5.
    - A reader that has seen `headIndex == N+5` now sees it move back to N+2. Indexes it was told about are
      now ahead of the head, and grain N+5's slot still holds the partial data from step 2.
  - **Suggested regression test:** run the scenario above with a reader that records `headIndex` after each
    commit. Assert that `headIndex` never decreases: either opening N+2 in step 4 is rejected, or `headIndex`
    stays at N+5 or higher after step 5.
  - Related: DC-BUG-7 (the sample path already guards against this).
  - Status: open

- [ ] **BUG-4: `mxlFlowWriterCommitGrain` copies the caller's whole `mxlGrainInfo` without checking it.** _(was BUG-2)_
  - **Finding:**
    - The caller's struct is copied into shared memory as-is ([DW] L170).
    - Fields that should never change (`totalSlices`, `grainSize`, `version`, `size`) can be overwritten.
    - `validSlices <= totalSlices` isn't checked. If it's larger, the grain never finishes, yet readers using
      `>=` treat it as available.
    - The header only promises that `flags` is updated.
  - **Test evidence:** every writer test changes the struct returned by open (`validSlices`, `flags`) and
    passes it back ([TF] L97-L98, L613-L614, L1222-L1223). Passing the struct back is the intended way to
    commit. No test changes the fields that should be fixed or sets `validSlices > totalSlices`. Stays BUG, as
    missing validation.
  - **Failure scenario:**
    - **`validSlices` too large:** a writer with an off-by-one error commits `validSlices = totalSlices + 1`.
      - The writer's `==` check fails, so the grain is never finished and `_lastCommittedIndex` doesn't
        advance.
      - Readers' `>=` check passes, so they return the grain as complete.
      - The writer and readers now disagree about the grain's state. The writer can keep committing to a
        grain that readers have already consumed.
    - **Fixed fields changed:** a writer reuses an `mxlGrainInfo` from another flow or clears it by mistake,
      and commits it with a different `totalSlices`, e.g. 1.
      - The writer treats the grain as complete after 1 slice.
      - Readers see `totalSlices == 1`, so a full-grain read returns `MXL_STATUS_OK` for a frame with 1 valid
        line.
      - The wrong `totalSlices` stays in that slot's header until the slot is reused.
  - **Suggested regression test:**
    - Commit `validSlices = totalSlices + 1`. Expect `MXL_ERR_INVALID_ARG`.
    - Commit with a changed `totalSlices`, `grainSize`, `version` or `size`. Expect `MXL_ERR_INVALID_ARG`, or
      check with `mxlFlowWriterGetGrainInfo` and a reader that the stored fields are unchanged.
  - Status: open

- [ ] **BUG-5: An invalid-flagged commit doesn't finish the grain.** _(was BUG-3)_
  - **Finding:** this contradicts the `MXL_GRAIN_FLAG_INVALID` documentation, which describes the flag as the
    proper way to move the ring buffer forward ([DW] L173-L178).
  - **Test evidence:** "Video Flow : Create/Destroy" ([TF] L96-L118) and "Data Flow : Create/Destroy"
    ([TF] L469-L490) commit an invalid-flagged grain with `validSlices == 0`. The reader gets
    `MXL_STATUS_OK` with the flag set, and `headIndex` moves to that index. The tests treat the grain as
    finished, which supports the fix in #728.
  - **Failure scenario:**
    1. An input times out, so the writer commits grain N with `MXL_GRAIN_FLAG_INVALID`, as the docs describe.
       Readers treat N as finished.
    2. The writer still treats N as open. `_lastCommittedIndex` is still N−1.
    3. Reopening N succeeds, so late data can be written into a grain that readers have already handled as
       invalid.
    4. Opening N+1 makes the writer treat N as a skipped grain. It rewrites N's header (`validSlices = 0`,
       flag set again), changing shared memory for a grain that has already been committed.
  - **Suggested regression test:**
    - The "Video Flow : Grain writer state" test in #728: after an invalid commit, reopening the same index
      returns `MXL_ERR_INVALID_ARG`, and opening the next index succeeds.
    - Add: after committing N as invalid, open N+1. Use `mxlFlowWriterGetGrainInfo` or a reader to check that
      N's header is unchanged.
  - Status: issue filed `dmf-mxl/mxl#719` · PR open `dmf-mxl/mxl#728`

- [ ] **BUG-6: `mxlFlowWriterGetGrainInfo` can return a different grain's info.** _(was BUG-4)_
  - **Finding:** it reads slot `index % grainCount` and doesn't check that the stored index matches the
    requested one ([DW] L77-L82).
  - **Test evidence:** none found in the tests reviewed. No test calls `mxlFlowWriterGetGrainInfo`.
  - **Failure scenario:**
    1. The writer completes grain N.
    2. It calls `mxlFlowWriterGetGrainInfo(N + grainCount)` to check whether that grain has been written
       yet.
    3. That index maps to the same slot as N, so the call returns `MXL_STATUS_OK` with grain N's info:
       `index == N`, all slices valid.
    4. A caller that doesn't check `info.index` concludes that grain N + `grainCount` is already complete.
  - **Suggested regression test:** complete grain N, then call
    `mxlFlowWriterGetGrainInfo(N + grainCount)`. Expect an error, e.g. `MXL_ERR_OUT_OF_RANGE_TOO_EARLY`, not
    `MXL_STATUS_OK` with `info.index == N`. Also check that `mxlFlowWriterGetGrainInfo(N)` still returns
    grain N.
  - Status: open

- [ ] **BUG-7: `mxlCreateFlowSynchronizationGroup` writes through `group` without a null check.** _(was BUG-5)_
  - **Finding:** [FLOW] L666. Every other output pointer in the API is checked.
  - **Test evidence:** "Synchronization group : Repeated waits" ([TSG] L69-L70) only passes a valid pointer.
  - **Failure scenario:** a caller passes `NULL` for `group`, for example from a binding that forwards an
    optional output. The library writes through the null pointer and the process crashes. Every other
    function returns `MXL_ERR_INVALID_ARG` for this mistake.
  - **Suggested regression test:** `mxlCreateFlowSynchronizationGroup(instance, nullptr)` returns
    `MXL_ERR_INVALID_ARG`. Before the fix this test crashes, so run it as a death test or in a subprocess.
  - Status: open

---

## 2. Documentation needs updating (most out of date first)

- [ ] **DOC-1: `mxlFlowWriterOpenSamples` docs contradict themselves.**
  - The summary says the range starts at `index`; `\param index` calls it the "head index".
  - The code treats `index` as the end of the range ([CW] L89-L102). The reader docs ("ending at a specific
    index") match the code.
  - Test evidence: "Audio Flow : Different writer / reader batch size" ([TF] L865-L880) writes the values
    `index − count + 1` … `index` and reads the same values back. "Audio Flow : tail read during open write"
    ([TRW] L254-L265, L294) writes ranges whose last sample is the given index. The tests treat `index` as
    the last sample and include it in the range. The docs should state this convention explicitly. Stays DOC.
  - Status: open

- [ ] **DOC-2: `mxlFlowReaderGetInfo` and `mxlFlowReaderGetRuntimeInfo` have copy-pasted text.**
  - Both say they return "the current flow config info value". `GetInfo` returns the full `mxlFlowInfo`;
    `GetRuntimeInfo` returns runtime info ([FLOWH] L245-L279).
  - Test evidence: "Video Flow : Slices" ([TF] L592-L599) reads both `.runtime` and `.config` from
    `mxlFlowReaderGetInfo`, which confirms it returns the full struct.
  - Status: open

- [ ] **DOC-3: `mxlCreateInstance` says `in_options` is "Currently not used."**
  - The options are parsed and parse errors are logged ([INST] L414-L427).
  - The history duration comes from `options.json` in the domain directory ([INST] L385-L412), which isn't
    documented.
  - Test evidence: `test_options.cpp` asserts the domain `options.json` sets the history duration
    ([TO] L32-L46). Instance options deliberately don't, with the comment "We don't want per-instance
    history durations" ([TO] L48-L74). Invalid options still produce an instance ([TO] L76-L90). The
    behaviour is intended; the header is out of date.
  - Status: open

- [ ] **DOC-4: The grain readers' timeout result is undocumented.**
  - `mxlFlowReaderGetGrain` and `mxlFlowReaderGetGrainSlice` never return `MXL_ERR_TIMEOUT`. On timeout they
    return `MXL_ERR_OUT_OF_RANGE_TOO_EARLY`, or `MXL_ERR_FLOW_INVALID` if the flow file was replaced
    ([DR] L110-L134).
  - The same applies to `mxlFlowSynchronizationGroupWaitForDataAt` ([SYNC] L124-L127).
  - The samples API documents this behaviour; the grain and sync-group APIs don't.
  - Test evidence: "Video Flow : Invalid flow (discrete)" ([TF] L289) asserts `MXL_ERR_FLOW_INVALID` from
    `mxlFlowReaderGetGrain`. The sync-group timeout test ([TSG] L103-L107) only asserts
    `!= MXL_STATUS_OK`, so no test fixes a timeout code. Stays DOC.
  - Related: BUG-1, BUG-2 (the fixes for these change which codes the sync group returns).
  - Status: open

- [ ] **DOC-5: `mxlFlowWriterCommitGrain` partial commits aren't documented.**
  - The docs say "done writing the grain", but the grain stays open until `validSlices == totalSlices`
    ([DW] L173-L178).
  - Committing an index other than the open one returns `MXL_ERR_INVALID_ARG` ([DW] L161-L164).
  - Test evidence: "Video Flow : Slices" ([TF] L601-L637) commits a grain in batches of
    `maxCommitBatchSizeHint` slices and reads each partial state back. "validSlices on grain open"
    ([TF] L1295-L1308) commits 3 slices and reopens the grain. Partial commits are a core, intended feature.
  - Status: open

- [ ] **DOC-6: Grain commit wakes readers on every partial commit.** _(moved from INT-4)_
  - [DW] L180-L182. The header says readers are woken when "a new grain is available".
  - Sample commit wakes readers only at batch boundaries.
  - Test evidence: "Video Flow : Slices" ([TF] L612-L629) reads each partial commit through
    `mxlFlowReaderGetGrainSlice` immediately after it is made. The slice API exists for this, so waking readers on each
    partial commit is intended. The header wording is out of date.
  - Related: DC-DOC-2.
  - Status: open

- [ ] **DOC-7: `mxlFlowWriterOpenGrain` changes shared memory before anything is committed.** _(moved from INT-3)_
  - Open marks skipped grains invalid and resets slot state ([DW] L98-L124). The sample writer does its gap
    handling at commit instead.
  - Test evidence: "Video Flow : Jump forward" ([TF] L1226-L1248) says "From the moment we open grain 100,
    the whole ring buffer should be invalidated. The head does not move until grain 100 is committed". It
    asserts the invalid flag on earlier grains before the commit. Invalidating at open is intended and should
    be documented.
  - Related: DC-DOC-1.
  - Status: open

- [ ] **DOC-8: `mxlReleaseFlowWriter` and `mxlDestroyInstance` delete flows without saying so.**
  - Releasing the last writer deletes the flow from the domain ([INST] L178-L182).
  - Destroying an instance deletes the flows of leaked writers that nobody else is using ([INST] L101-L116).
  - Test evidence: "Flow deletion on writer release" ([TI] L124-L155) and "Flow deletion on instance
    destruction" ([TI] L157-L182) assert both behaviours. They are intended.
  - Status: open

- [ ] **DOC-9: `mxlFlowReaderGetGrain` returns `MXL_STATUS_OK` for invalid-flagged grains.**
  - This happens however many slices are valid ([DR] L183-L184). It's only implied by the flag's comment.
  - Test evidence: [TF] L98-L107 and L471-L477 commit an invalid grain with 0 valid slices. The full-grain
    `mxlFlowReaderGetGrain` returns `MXL_STATUS_OK` with the flag set. Intended.
  - Status: open

- [ ] **DOC-10: `mxlFlowWriterOpenSamples` and `mxlFlowWriterCommitSamples` don't list their error cases or extra behaviour.**
  - Open returns `MXL_ERR_INVALID_ARG` when `count` exceeds the maximum write length, when `index < count`, or
    when the range overlaps committed data ([CW] L82-L96).
  - Commit wakes readers only at sync-batch boundaries ([CW] L169-L174) and zero-fills skipped gaps
    ([CW] L136-L161).
  - Commit with nothing open returns `MXL_ERR_INVALID_ARG`.
  - Test evidence: half-buffer writes (the maximum) succeed ([TRW] L260, L294). Gap zero-filling is asserted
    by "Audio Flow : Jump forward" ([TF] L1171-L1182). No test covers the error cases.
  - Status: open

- [ ] **DOC-11: Writers and readers for the same flow ID are shared within one instance.**
  - Each is a single reference-counted object per flow ID ([INST] L121-L143, L220-L231).
  - `mxlCreateFlowReader` ignores its `options` argument ([FLOW] L136).
  - Test evidence: "Flow readers / writers caching" ([TI] L58-L122) asserts that two readers for the same
    flow are the same handle (`audioReader == audioReader2`). Reader sharing is intended. Writers are only
    tested across different flows; see INT-3.
  - Related: INT-3.
  - Status: open

- [ ] **DOC-12: Smaller undocumented edge cases.**
  - `mxlFlowReaderGetSamples` with a `count` larger than the readable window returns
    `MXL_ERR_OUT_OF_RANGE_TOO_LATE` rather than an invalid-argument error ([CR] L123-L156).
  - `MXL_GRAIN_VALID_SLICES_ANY` returns grains with 0 valid slices ([DR] L183).
  - Test evidence: "Video Flow : Jump forward" ([TF] L1238-L1247) reads invalidated grains with
    `minValidSlices` 0 and expects `MXL_STATUS_OK`. Reading 0-slice grains with `ANY` is intended. No test
    covers the oversized sample read.
  - Status: open

- [ ] **DOC-13: Releasing a reader silently removes it from every sync group.**
  - [INST] L154-L161.
  - Test evidence: none found in the tests reviewed.
  - Status: open

- [ ] **DOC-14: `mxlGarbageCollectFlows` always returns `MXL_STATUS_OK`.**
  - Errors are only logged at debug level ([INST] L303-L371, [MXL] L115-L136).
  - If a flow file can't be opened, the flow is treated as active and left alone.
  - Test evidence: none found in the tests reviewed. `test_flowmanager.cpp` was not reviewed and may cover
    this.
  - Status: open

---

## 3. Needs maintainer intent (most clearly out of sync first)

- [ ] **INT-1: `mxlFlowWriterOpenGrain` doesn't enforce "must cancel or commit" first.**
  - The header says "must", but opening a different index silently drops the open grain ([DW] L84-L129).
  - Options: enforce it with an error code, or document that it's the caller's responsibility.
  - Test evidence: existing tests rely on the current behaviour.
    - "Video Flow : Create/Destroy" ([TF] L96-L98, L131) and the alpha variant ([TF] L205-L207, L240) commit
      an invalid-flagged grain and then open `++index`. At `c66150a` an invalid commit doesn't finish the
      grain, so this second open silently replaces an open grain.
    - The same tests, "Data Flow : Create/Destroy" ([TF] L503-L505) and "Video Flow : Slices" ([TF] L589),
      release the writer with a grain still open.
    - Enforcing the "must" would break these tests unless #728 lands first. Even then, the tests would still
      release writers with a grain open.
  - Related: DC-INT-2.
  - Decision: _pending_
  - Status: open

- [ ] **INT-2: `mxlFlowWriterCancelGrain` is undocumented and doesn't undo what open changed.**
  - Cancel only resets `_currentIndex` ([DW] L131-L135). The following changes made by open stay in shared
    memory:
    - the slot's `index`
    - the cleared invalid flag
    - the `validSlices` reset
    - skipped grains marked invalid
  - It also discards partial progress: reopening after cancel resets `validSlices` to 0, even if readers
    already saw a higher count.
  - Test evidence: none found in the tests reviewed. No test calls `mxlFlowWriterCancelGrain` or
    `mxlFlowWriterCancelSamples`.
  - Decision: _pending_
  - Status: open

- [ ] **INT-3: Two `mxlCreateFlowWriter` calls in one instance share one open/commit state.** _(was INT-5)_
  - [INST] L220-L231. Callers may expect independent writers.
  - Test evidence: the test "Flow readers / writers caching" ([TI] L58-L122) suggests caching was designed
    for both readers and writers. It only asserts reader sharing, though; no test creates two writers for the
    same flow in one instance. Writers in different instances are tested ([TI] L124-L155) and are
    independent.
  - Related: DOC-11.
  - Decision: _pending_
  - Status: open

---

## Discrete/Continuous Consistency

- **A:** comparison of the grain (discrete) path and the sample (continuous) path.
- **B:** error handling in the C entry points (`lib/src/flow.cpp`, `lib/src/mxl.cpp`).
- The source item (A1–A9, B1–B10) is shown in brackets after each title.

### Discrete/Continuous Consistency: Looks like a real bug (most egregious first)

- [ ] **DC-BUG-1: `mxlReleaseFlowWriter` can let an exception escape the C API.** (B1)
  - **Finding:**
    - It only catches `std::exception` ([FLOW] L273-L281). Any other exception thrown during release goes
      through an `extern "C"` function and calls `std::terminate`.
    - Every other entry point has a `catch (...)`.
  - **Test evidence:** none found in the tests reviewed.
  - **Failure scenario:**
    1. During release, the last-writer path calls `isExclusive()`/`makeExclusive()` and may delete the flow
       ([INST] L176-L184). Something in that path throws an exception not derived from `std::exception`,
       for example from a third-party library or a platform-specific unwind.
    2. The exception reaches the `extern "C"` boundary uncaught, and `std::terminate` aborts the whole host
       process.
    - The same failure in any other entry point would return `MXL_ERR_UNKNOWN`.
  - **Suggested regression test:**
    - Needs fault injection, because this can't be triggered through the public API with real flows.
    - Use an internal test with a test `FlowIoFactory` whose writer throws a non-`std::exception` value from
      `makeExclusive()`.
    - Call `mxlReleaseFlowWriter` and expect `MXL_ERR_UNKNOWN` instead of process termination.
  - Status: open

- [ ] **DC-BUG-2: `mxlCreateInstance` doesn't check `in_mxlDomain` for null.** (B2)
  - **Finding:**
    - The pointer is used directly as a path ([MXL] L45-L52). Building a `std::filesystem::path` or
      `std::string` from a null `char const*` is undefined behaviour.
    - The surrounding `try` can't catch undefined behaviour, so the documented "returns NULL" isn't
      guaranteed.
  - **Test evidence:** none found in the tests reviewed. Null arguments are tested for `mxlIsTmpFs`
    ([TI] L25-L34) and `mxlGetFlowDef` ([TF] L980-L990), but not for `mxlCreateInstance`.
  - **Failure scenario:** an application builds the domain path from an environment variable or config file
    and passes `NULL` when the setting is missing. It expects to get `NULL` back and report a configuration
    error. Instead, the library hits undefined behaviour, typically a crash inside the path or string
    constructor.
  - **Suggested regression test:** `mxlCreateInstance(nullptr, nullptr)` returns `nullptr`. Before the fix
    this test may crash the test process, so run it as a death test or in a subprocess.
  - Status: open

- [ ] **DC-BUG-3: The list of sync groups isn't thread-safe.** (B3)
  - **Finding:**
    - `createFlowSynchronizationGroup` and `releaseFlowSynchronizationGroup` change `_syncGroups` without
      taking `_mutex` ([INST] L439-L456).
    - `releaseReader` walks the same list while holding the lock ([INST] L151-L160).
    - Doing both at once on different threads is a data race on a `std::forward_list`.
  - **Test evidence:** none found in the tests reviewed. The sync-group test is single-threaded for group
    operations ([TSG] L69-L72, L109-L111).
  - **Failure scenario:** in a multi-threaded application sharing one instance, thread A creates or releases a
    sync group while thread B releases a reader. Both threads use the `_syncGroups` list at the same time. The
    list can be corrupted, giving crashes or a group that is never cleaned up. The failure is intermittent and
    timing-dependent.
  - **Suggested regression test:**
    - A stress test with two threads on one instance: one repeatedly creates and releases sync groups, the
      other repeatedly creates and releases readers.
    - Run it under ThreadSanitizer, which reports the race even when no crash happens. Without TSan the test
      may pass by luck.
  - Status: open

- [ ] **DC-BUG-4: Permission checks differ between the grain and sample paths.** (A7)
  - **Finding:**
    - The continuous writer and reader call `checkPermissions()` and throw if it fails ([CW] L26-L29,
      [CR] L19-L22).
    - The discrete writer and reader don't ([DW] L21-L30, [DR] L40-L51).
    - The same permission problem therefore fails at creation for one flow type and not at all for the
      other.
  - **Test evidence:** "mxlCreateFlow: unwritable domain" ([TF] L1008-L1027) tests a discrete (v210) writer
    only. That failure comes from flow creation, not from `checkPermissions()`. No continuous or reader
    permission test exists.
  - **Failure scenario:**
    1. A deployment sets file permissions that fail `checkPermissions()` for both an audio flow and a video
       flow.
    2. Creating a reader or writer for the audio flow fails immediately.
    3. Creating one for the video flow succeeds. The problem appears later, as read or write errors, or not
       at all.
    - Operators see different behaviour for the same misconfiguration depending on the flow type.
  - **Suggested regression test:**
    - Create a v210 flow and an audio flow, apply the same permission setup to both, and create a reader and
      a writer for each. Assert that both flow types give the same result.
    - The exact setup depends on what `checkPermissions()` checks; that function was not reviewed.
    - The test must not run as root, which bypasses permission checks.
  - Status: open

- [ ] **DC-BUG-5: The reader reports permission errors as `MXL_ERR_UNKNOWN`.** (B5)
  - **Finding:**
    - `mxlCreateFlowWriter` maps permission-type filesystem errors to `MXL_ERR_PERMISSION_DENIED`
      ([FLOW] L228-L240).
    - `mxlCreateFlowReader` maps only `ENOENT`; anything else becomes `MXL_ERR_UNKNOWN` ([FLOW] L154-L167).
    - The continuous reader's `checkPermissions()` failure is a `std::runtime_error`, which also becomes
      `MXL_ERR_UNKNOWN`.
  - **Test evidence:** "mxlCreateFlow: unwritable domain" ([TF] L1022) asserts `MXL_ERR_PERMISSION_DENIED` for
    the writer. Reporting permission errors as their own code is intended; the reader has no equivalent test.
    Strengthened.
  - **Failure scenario:** a consumer process runs as a user that can't read a flow's files. The writer side of
    the same setup would report `MXL_ERR_PERMISSION_DENIED`. `mxlCreateFlowReader` returns `MXL_ERR_UNKNOWN`,
    so the application can't tell the user it's a permissions problem. `mxl.h` also tells callers not to
    check for `MXL_ERR_UNKNOWN`.
  - **Suggested regression test:** create a flow, remove read permission from its files, and call
    `mxlCreateFlowReader`. Expect `MXL_ERR_PERMISSION_DENIED`. Do this for a grain flow and a sample flow. Must
    not run as root.
  - Status: open

- [ ] **DC-BUG-6: The sample path never updates `lastReadTime`.** (A9)
  - **Finding:**
    - The discrete reader touches the flow's access file on each successful read ([DR] L117-L121,
      L143-L147).
    - The continuous reader never opens or touches that file.
    - Reader activity on audio flows is therefore never reflected in their runtime info.
  - **Test evidence:** the discrete tests assert that `lastReadTime` increases after a read ([TF] L120-L121,
    L492-L493, L634-L636). "Audio Flow : Create/Destroy" checks `headIndex` after a read but not
    `lastReadTime` ([TF] L730-L735). The field is clearly intended to work; the audio gap is untested.
    Strengthened.
  - **Failure scenario:** a monitoring tool, such as `mxl-info`, or a controller uses `lastReadTime` to decide
    whether a flow has active consumers. Audio flows with readers that read continuously still show their
    original `lastReadTime`. The tool reports them as unused, and a controller might stop or reroute them.
  - **Suggested regression test:** copy the discrete check for an audio flow. Record `lastReadTime`, read
    samples with `mxlFlowReaderGetSamples`, wait briefly for the update to propagate (as the discrete tests do
    at [TF] L103-L104), and assert that `lastReadTime` has increased.
  - Status: open

- [ ] **DC-BUG-7: The grain path lets `headIndex` move backwards; the sample path doesn't.** (A1)
  - **Finding:**
    - This is the same issue as BUG-3, seen by comparing the two paths.
    - `openSamples` rejects `index <= _lastCommittedIndex` and any overlap with committed data
      ([CW] L90-L96). The grain path has no equivalent check against the current `headIndex`.
  - **Test evidence:** see BUG-3.
  - **Failure scenario:**
    - Grain path: as in BUG-3. A partial commit moves `headIndex` to N+5, then cancel, open N+2 and commit
      move it back to N+2.
    - Sample path, for comparison: commit samples ending at M, open a range ending at M+K, cancel, then open
      and commit a range ending at M+J where J < K.
      - `headIndex` goes from M to M+J and never decreases.
      - The cancelled range never moved the head, because sample commits are never partial.
  - **Suggested regression test:** use the BUG-3 test for the grain path. Add the sample-path sequence above,
    asserting that `headIndex` never decreases. That assertion already passes for samples, and should pass for
    grains once BUG-3 is fixed.
  - Fix together with BUG-3.
  - Status: open

- [ ] **DC-BUG-8: `mxlIsFlowActive` reports any open failure as "flow not found".** (B6)
  - **Finding:**
    - Every `open()` failure, including `EACCES`, returns `MXL_ERR_FLOW_NOT_FOUND` ([FLOW] L47-L58).
    - The code special-cases `ENOENT` and then returns the same code for everything else.
  - **Test evidence:** only the active case is tested ([TF] L55-L57, L272-L275).
  - **Failure scenario:** a monitoring process without read permission on a flow's data file calls
    `mxlIsFlowActive`. It gets `MXL_ERR_FLOW_NOT_FOUND` for a flow that exists and is being written. It
    reports the flow as missing, or a controller tries to recreate it, instead of reporting a permissions
    problem.
  - **Suggested regression test:** create a flow, remove read permission from its data file, and call
    `mxlIsFlowActive`. Expect `MXL_ERR_PERMISSION_DENIED`, or at least not `MXL_ERR_FLOW_NOT_FOUND`. Keep the
    existing check that a deleted flow returns `MXL_ERR_FLOW_NOT_FOUND`. Must not run as root.
  - Status: open

- [ ] **DC-BUG-9: Grain commit with no flow data leaves the writer's state set.** (A6)
  - **Finding:**
    - The discrete writer returns `MXL_ERR_UNKNOWN` and leaves `_currentIndex` set ([DW] L157-L187).
    - The continuous writer clears it first ([CW] L178-L182).
    - Minor: the writer is unusable at that point anyway.
  - **Test evidence:** none found in the tests reviewed.
  - **Failure scenario:** if a discrete writer ever loses its flow data, a commit returns `MXL_ERR_UNKNOWN`
    but the writer still records the grain as open. Any later call then behaves differently from a
    continuous writer in the same state, which has already cleared the open range.
  - **Suggested regression test:**
    - A writer with no flow data can't be created through the public C API. The discrete writer's
      constructor and destructor also use the flow data, so this needs an internal test hook.
    - With the hook, set up both writer types without flow data, call commit, and assert that both return
      `MXL_ERR_UNKNOWN` and leave no range or grain open.
  - Status: open

- [ ] **DC-BUG-10: `isExclusive()` and `makeExclusive()` behave differently when there's no flow data.** (A8)
  - **Finding:**
    - The discrete writer returns `false` ([DW] L137-L155); the continuous writer throws ([CW] L220-L238).
    - `~Instance` catches the exception, but `Instance::releaseWriter` doesn't ([INST] L101-L116,
      L167-L189).
    - Minor: the case shouldn't happen in practice.
  - **Test evidence:** none found in the tests reviewed.
  - **Failure scenario:** releasing the last writer with no flow data has different results by flow type:
    - **Discrete writer:** `releaseWriter` sees `false`, doesn't delete the flow, and removes the writer from
      the instance.
    - **Continuous writer:** the exception escapes `releaseWriter` before the writer is removed.
      `mxlReleaseFlowWriter` returns `MXL_ERR_UNKNOWN`, and the writer stays in the instance's table.
  - **Suggested regression test:**
    - As with DC-BUG-9, this needs an internal test hook, because the state can't be reached through the
      public API.
    - With the hook, call `isExclusive()` and `makeExclusive()` on both writer types without flow data.
      Assert that they behave the same way: both return `false`, or both throw.
    - Then release each through `Instance::releaseWriter` and assert that the outcome is the same for both.
  - Status: open

### Discrete/Continuous Consistency: Documentation needs updating (most out of date first)

- [ ] **DC-DOC-1: Gap handling happens at different times in the two paths.** (A2) _(moved from DC-INT-1)_
  - The grain path marks skipped grains invalid at open, in shared memory, before anything is committed
    ([DW] L98-L115).
  - The sample path zero-fills skipped samples at commit ([CW] L136-L161).
  - In the grain path, open followed by cancel still invalidates grains; in the sample path it doesn't.
  - Test evidence: both timings are asserted on purpose.
    - "Audio Flow : Jump forward" ([TF] L1133-L1157) says "invalidation happens on commit" and asserts the
      old values are still readable after open.
    - "Video Flow : Jump forward" ([TF] L1230-L1248) asserts invalidation from the moment of open.
    - The difference is intended and should be documented. Making the two paths consistent would mean
      changing tests; that is an optional proposal for the maintainers, not a fix.
  - Related: DOC-7.
  - Status: open

- [ ] **DC-DOC-2: Reader wake-ups differ between the two paths.** (A5) _(moved from DC-INT-2)_
  - Grain commit wakes readers on every commit, including partial ones ([DW] L180-L182).
  - Sample commit wakes them only at sync-batch boundaries ([CW] L169-L174, L192-L218).
  - Test evidence:
    - "mxlFlowWriterCommitSamples should notify the reader" ([TF] L1029-L1080) commits blocks of 480
      samples, the default batch size ([TF] L957-L958), and expects a blocking reader to receive every block.
      Batched wake-up is intended.
    - "Video Flow : Slices" depends on per-commit wake-ups for partial grain reads (DOC-6).
    - Both behaviours are intended per path; document them.
  - Related: DOC-6, DC-DOC-3.
  - Status: open

- [ ] **DC-DOC-3: The discrete writer doesn't use the batch-size hints.** (A5)
  - `maxSyncBatchSizeHint` and `maxCommitBatchSizeHint` are stored for grain flows ([INST] L254-L255), but the
    discrete writer never reads them.
  - Test evidence:
    - "Video Flow : Options" ([TF] L511-L557) validates and stores the hints for a video flow; both default
      to 1080, i.e. `totalSlices`.
    - "Video Flow : Slices" ([TF] L596-L613) uses `maxCommitBatchSizeHint` as the number of slices per
      partial commit.
    - For discrete flows the hints are guidance to callers, not "ignored". Document what they mean for each
      flow type.
  - Related: DC-DOC-2.
  - Status: open

### Discrete/Continuous Consistency: Needs maintainer intent (most clearly out of sync first)

- [ ] **DC-INT-1: Invalid caller input returns `MXL_ERR_UNKNOWN`.** (B7) _(was DC-INT-5)_
  - Parse failures in `mxlCreateFlowWriter` end up in the generic `std::exception` handler ([FLOW] L242-L246).
  - `mxl.h` warns callers not to rely on `MXL_ERR_UNKNOWN`, so a caller can't detect "bad flowDef" or "bad
    options".
  - `MXL_ERR_INVALID_ARG` looks like the natural code, but switching changes behaviour callers can see.
  - Test evidence:
    - "Video Flow : Options" ([TF] L526-L537) and "Audio Flow : Options" ([TF] L930-L941) assert exactly
      `MXL_ERR_UNKNOWN` for invalid batch-size options. The tests rely on the code that `mxl.h` tells callers
      not to rely on.
    - "Invalid flow definitions" ([TF] L319-L375) only asserts `!= MXL_STATUS_OK`.
    - Ranked first because the tests and the header guidance directly contradict each other.
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-2: Opening while something is already open is silently allowed in both paths.** (A3) _(was DC-INT-3)_
  - In both writers, a second open replaces the current one without an error ([DW] L117-L127,
    [CW] L116-L117).
  - Whatever is decided for INT-1 should apply to both paths.
  - Test evidence: see INT-1. The grain tests rely on it. No sample test opens twice without committing.
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-3: There's no single rule for which error code a bad handle returns.** (B4) _(was DC-INT-4)_
  - For a bad reader:
    - `mxlFlowSynchronizationGroupAddReader` and `AddPartialGrainReader` return `MXL_ERR_INVALID_FLOW_READER`
      ([FLOW] L702-L756).
    - `mxlFlowSynchronizationGroupRemoveReader` ([FLOW] L760-L778) and `mxlReleaseFlowReader`
      ([FLOW] L172-L189) return `MXL_ERR_INVALID_ARG`.
  - The writer functions have the same split between `MXL_ERR_INVALID_FLOW_WRITER` and
    `MXL_ERR_INVALID_ARG`.
  - Test evidence: no test passes a bad reader or writer handle. A null instance for `mxlGetFlowDef` returns
    `MXL_ERR_INVALID_ARG` ([TF] L980).
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-4: Commit with nothing open gives the same result by different routes.** (A4) _(was DC-INT-6)_
  - Both paths return `MXL_ERR_INVALID_ARG`.
  - The grain path gets there because the index doesn't match ([DW] L161-L164); the sample path checks
    explicitly ([CW] L131-L134).
  - The grain path relies on an accident: a commit whose `grainInfo.index` equals `MXL_UNDEFINED_INDEX` would
    pass the check.
  - Question: should both paths use an explicit "nothing open" check?
  - Test evidence: none found in the tests reviewed.
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-5: Exception logging varies.** (B9) _(was DC-INT-7)_
  - Some entry points log the exception before returning `MXL_ERR_UNKNOWN`, e.g. `mxlCreateFlowWriter`,
    `mxlIsFlowActive`, `mxlGetFlowDef`.
  - Most `catch (...)` blocks return silently, e.g. `mxlCreateFlowReader` and all the reader/writer data calls.
  - Question: should every `MXL_ERR_UNKNOWN` be logged?
  - Test evidence: not testable through the reviewed tests.
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-6: Argument checks happen in different places (style only).** (B8) _(was DC-INT-8)_
  - `mxlCreateFlowWriter`, `mxlReleaseFlowWriter`, `mxlGetFlowDef` and `mxlFlowWriterCommitGrain` check
    arguments before the `try`; the rest check inside it ([FLOW]).
  - Behaviour is the same either way, but one convention would make DC-INT-3 and DC-INT-5 easier to keep
    consistent.
  - Test evidence: not applicable.
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-7: `mxlDestroyInstance` checks for null after deleting (style only).** (B10) _(was DC-INT-9)_
  - It deletes first and then tests for null to pick the return code ([MXL] L104-L105).
  - Harmless, since deleting a null pointer is a no-op, but every other entry point checks first.
  - Test evidence: not applicable.
  - Decision: _pending_
  - Status: open

---

## Code references (pinned to `c66150a`)

[FLOWH]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/include/mxl/flow.h
[MXLH]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/include/mxl/mxl.h
[FLOW]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/src/flow.cpp
[MXL]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/src/mxl.cpp
[INST]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/internal/src/Instance.cpp
[DW]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/internal/src/PosixDiscreteFlowWriter.cpp
[DR]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/internal/src/PosixDiscreteFlowReader.cpp
[CW]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/internal/src/PosixContinuousFlowWriter.cpp
[CR]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/internal/src/PosixContinuousFlowReader.cpp
[SYNC]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/internal/src/FlowSynchronizationGroup.cpp
[TF]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/tests/test_flows.cpp
[TSG]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/tests/test_flow_sync_groups.cpp
[TRW]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/tests/test_read_write_conflict.cpp
[TI]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/tests/test_instance.cpp
[TO]: https://github.com/dmf-mxl/mxl/blob/c66150ac387cf597f65c40acb1e908d6503674d7/lib/internal/tests/test_options.cpp

| Key | File |
|---|---|
| [FLOWH] | `lib/include/mxl/flow.h` |
| [MXLH] | `lib/include/mxl/mxl.h` |
| [FLOW] | `lib/src/flow.cpp` |
| [MXL] | `lib/src/mxl.cpp` |
| [INST] | `lib/internal/src/Instance.cpp` |
| [DW] | `lib/internal/src/PosixDiscreteFlowWriter.cpp` |
| [DR] | `lib/internal/src/PosixDiscreteFlowReader.cpp` |
| [CW] | `lib/internal/src/PosixContinuousFlowWriter.cpp` |
| [CR] | `lib/internal/src/PosixContinuousFlowReader.cpp` |
| [SYNC] | `lib/internal/src/FlowSynchronizationGroup.cpp` |
| [TF] | `lib/tests/test_flows.cpp` |
| [TSG] | `lib/tests/test_flow_sync_groups.cpp` |
| [TRW] | `lib/tests/test_read_write_conflict.cpp` |
| [TI] | `lib/tests/test_instance.cpp` |
| [TO] | `lib/internal/tests/test_options.cpp` |
