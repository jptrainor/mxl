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
| BUG | 5 |
| DOC | 14 |
| INT | 3 |
| DC-BUG | 10 |
| DC-DOC | 3 |
| DC-INT | 7 |
| **Total** | **42** |

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

## Already addressed

- [x] **REF-1: `validSlices` not reset when a new grain is opened.** Fixed.
  - Issue `dmf-mxl/mxl#717` (closed) · PR `dmf-mxl/mxl#718` (merged) · backports `dmf-mxl/mxl#725` (v1.1), `dmf-mxl/mxl#726` (v1.0)
  - Test: "Video Flow : validSlices on grain open" ([TF] L1263-L1337).

---

## 1. Looks like a real bug (most egregious first)

- [ ] **BUG-1: `mxlFlowWriterCommitGrain` can move `headIndex` backwards.**
  - Every commit sets `headIndex = _currentIndex` ([DW] L167). A partial commit doesn't update
    `_lastCommittedIndex`, so after cancelling a partly committed grain the writer can open and commit a lower
    index, and readers see the head go backwards.
  - The header says the head moves only "IF this grain is the new head".
  - Test evidence: "Video Flow : Slices" ([TF] L612-L618) asserts `headIndex == index` after each partial
    commit, so moving the head on a partial commit is intended. No test covers cancelling and then committing
    a lower index, and nothing suggests the head is meant to move backwards. Stays BUG.
  - Related: DC-BUG-7 (the sample path already guards against this).
  - Status: open

- [ ] **BUG-2: `mxlFlowWriterCommitGrain` copies the caller's whole `mxlGrainInfo` without checking it.**
  - The caller's struct is copied into shared memory as-is ([DW] L170).
  - Fields that should never change (`totalSlices`, `grainSize`, `version`, `size`) can be overwritten.
  - `validSlices <= totalSlices` isn't checked. If it's larger, the grain never finishes, yet readers using
    `>=` treat it as available.
  - The header only promises that `flags` is updated.
  - Test evidence: every writer test changes the struct returned by open (`validSlices`, `flags`) and passes
    it back ([TF] L97-L98, L613-L614, L1222-L1223). Passing the struct back is the intended way to commit.
    No test changes the fields that should be fixed or sets `validSlices > totalSlices`. Stays BUG, as
    missing validation.
  - Status: open

- [ ] **BUG-3: An invalid-flagged commit doesn't finish the grain.**
  - This contradicts the `MXL_GRAIN_FLAG_INVALID` documentation, which describes the flag as the proper way
    to move the ring buffer forward ([DW] L173-L178).
  - Test evidence: "Video Flow : Create/Destroy" ([TF] L96-L118) and "Data Flow : Create/Destroy"
    ([TF] L469-L490) commit an invalid-flagged grain with `validSlices == 0`. The reader gets
    `MXL_STATUS_OK` with the flag set, and `headIndex` moves to that index. The tests treat the grain as
    finished, which supports the fix in #728.
  - Status: issue filed `dmf-mxl/mxl#719` · PR open `dmf-mxl/mxl#728`

- [ ] **BUG-4: `mxlFlowWriterGetGrainInfo` can return a different grain's info.**
  - It reads slot `index % grainCount` and doesn't check that the stored index matches the requested one
    ([DW] L77-L82).
  - Test evidence: none found in the tests reviewed. No test calls `mxlFlowWriterGetGrainInfo`.
  - Status: open

- [ ] **BUG-5: `mxlCreateFlowSynchronizationGroup` writes through `group` without a null check.**
  - [FLOW] L666. Every other output pointer in the API is checked.
  - Test evidence: "Synchronization group : Repeated waits" ([TSG] L69-L70) only passes a valid pointer.
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
  - It only catches `std::exception` ([FLOW] L273-L281). Any other exception thrown during release goes
    through an `extern "C"` function and calls `std::terminate`.
  - Every other entry point has a `catch (...)`.
  - Test evidence: none found in the tests reviewed.
  - Status: open

- [ ] **DC-BUG-2: `mxlCreateInstance` doesn't check `in_mxlDomain` for null.** (B2)
  - The pointer is used directly as a path ([MXL] L45-L52). Building a `std::filesystem::path` or
    `std::string` from a null `char const*` is undefined behaviour.
  - The surrounding `try` can't catch undefined behaviour, so the documented "returns NULL" isn't guaranteed.
  - Test evidence: none found in the tests reviewed. Null arguments are tested for `mxlIsTmpFs` ([TI] L25-L34)
    and `mxlGetFlowDef` ([TF] L980-L990), but not for `mxlCreateInstance`.
  - Status: open

- [ ] **DC-BUG-3: The list of sync groups isn't thread-safe.** (B3)
  - `createFlowSynchronizationGroup` and `releaseFlowSynchronizationGroup` change `_syncGroups` without taking
    `_mutex` ([INST] L439-L456).
  - `releaseReader` walks the same list while holding the lock ([INST] L151-L160).
  - Doing both at once on different threads is a data race on a `std::forward_list`.
  - Test evidence: none found in the tests reviewed. The sync-group test is single-threaded for group
    operations ([TSG] L69-L72, L109-L111).
  - Status: open

- [ ] **DC-BUG-4: Permission checks differ between the grain and sample paths.** (A7)
  - The continuous writer and reader call `checkPermissions()` and throw if it fails ([CW] L26-L29,
    [CR] L19-L22).
  - The discrete writer and reader don't ([DW] L21-L30, [DR] L40-L51).
  - The same permission problem therefore fails at creation for one flow type and not at all for the other.
  - Test evidence: "mxlCreateFlow: unwritable domain" ([TF] L1008-L1027) tests a discrete (v210) writer
    only. That failure comes from flow creation, not from `checkPermissions()`. No continuous or reader
    permission test exists.
  - Status: open

- [ ] **DC-BUG-5: The reader reports permission errors as `MXL_ERR_UNKNOWN`.** (B5)
  - `mxlCreateFlowWriter` maps permission-type filesystem errors to `MXL_ERR_PERMISSION_DENIED`
    ([FLOW] L228-L240).
  - `mxlCreateFlowReader` maps only `ENOENT`; anything else becomes `MXL_ERR_UNKNOWN` ([FLOW] L154-L167).
  - The continuous reader's `checkPermissions()` failure is a `std::runtime_error`, which also becomes
    `MXL_ERR_UNKNOWN`.
  - Test evidence: "mxlCreateFlow: unwritable domain" ([TF] L1022) asserts `MXL_ERR_PERMISSION_DENIED` for
    the writer. Reporting permission errors as their own code is intended; the reader has no equivalent test.
    Strengthened.
  - Status: open

- [ ] **DC-BUG-6: The sample path never updates `lastReadTime`.** (A9)
  - The discrete reader touches the flow's access file on each successful read ([DR] L117-L121, L143-L147).
  - The continuous reader never opens or touches that file.
  - Reader activity on audio flows is therefore never reflected in their runtime info.
  - Test evidence: the discrete tests assert that `lastReadTime` increases after a read ([TF] L120-L121,
    L492-L493, L634-L636). "Audio Flow : Create/Destroy" checks `headIndex` after a read but not
    `lastReadTime` ([TF] L730-L735). The field is clearly intended to work; the audio gap is untested.
    Strengthened.
  - Status: open

- [ ] **DC-BUG-7: The grain path lets `headIndex` move backwards; the sample path doesn't.** (A1)
  - This is the same issue as BUG-1, seen by comparing the two paths.
  - `openSamples` rejects `index <= _lastCommittedIndex` and any overlap with committed data ([CW] L90-L96).
    The grain path has no equivalent check against the current `headIndex`.
  - Test evidence: see BUG-1.
  - Fix together with BUG-1.
  - Status: open

- [ ] **DC-BUG-8: `mxlIsFlowActive` reports any open failure as "flow not found".** (B6)
  - Every `open()` failure, including `EACCES`, returns `MXL_ERR_FLOW_NOT_FOUND` ([FLOW] L47-L58).
  - The code special-cases `ENOENT` and then returns the same code for everything else.
  - Test evidence: only the active case is tested ([TF] L55-L57, L272-L275).
  - Status: open

- [ ] **DC-BUG-9: Grain commit with no flow data leaves the writer's state set.** (A6)
  - The discrete writer returns `MXL_ERR_UNKNOWN` and leaves `_currentIndex` set ([DW] L157-L187).
  - The continuous writer clears it first ([CW] L178-L182).
  - Minor: the writer is unusable at that point anyway.
  - Test evidence: none found in the tests reviewed.
  - Status: open

- [ ] **DC-BUG-10: `isExclusive()` and `makeExclusive()` behave differently when there's no flow data.** (A8)
  - The discrete writer returns `false` ([DW] L137-L155); the continuous writer throws ([CW] L220-L238).
  - `~Instance` catches the exception, but `Instance::releaseWriter` doesn't ([INST] L101-L116, L167-L189).
  - Minor: the case shouldn't happen in practice.
  - Test evidence: none found in the tests reviewed.
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
