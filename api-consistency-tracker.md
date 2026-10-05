# libmxl API consistency tracker

Findings from comparing the public API documentation (`lib/include/mxl/flow.h`, `lib/include/mxl/mxl.h`)
with the libmxl implementation, plus a review of consistency between the grain and sample paths and
across the C entry points.

- **Upstream repository:** [dmf-mxl/mxl](https://github.com/dmf-mxl/mxl)
- **Pinned commit:** [`c66150a`](https://github.com/dmf-mxl/mxl/tree/c66150ac387cf597f65c40acb1e908d6503674d7)
  (all line numbers refer to this commit)
- **Method:** reading the code only. Findings have not been confirmed by tests.

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
| DOC | 12 |
| INT | 5 |
| DC-BUG | 10 |
| DC-DOC | 1 |
| DC-INT | 9 |
| **Total** | **42** |

## Already addressed

- [x] **REF-1: `validSlices` not reset when a new grain is opened.** Fixed.
  - Issue `dmf-mxl/mxl#717` (closed) · PR `dmf-mxl/mxl#718` (merged) · backports `dmf-mxl/mxl#725` (v1.1), `dmf-mxl/mxl#726` (v1.0)

---

## 1. Looks like a real bug (most egregious first)

- [ ] **BUG-1: `mxlFlowWriterCommitGrain` can move `headIndex` backwards.**
  - Every commit sets `headIndex = _currentIndex` ([DW] L167). A partial commit doesn't update
    `_lastCommittedIndex`, so after cancelling a partly committed grain the writer can open and commit a lower
    index, and readers see the head go backwards.
  - The header says the head moves only "IF this grain is the new head".
  - Related: DC-BUG-7 (the sample path already guards against this).
  - Status: open

- [ ] **BUG-2: `mxlFlowWriterCommitGrain` copies the caller's whole `mxlGrainInfo` without checking it.**
  - The caller's struct is copied into shared memory as-is ([DW] L170).
  - Fields that should never change (`totalSlices`, `grainSize`, `version`, `size`) can be overwritten.
  - `validSlices <= totalSlices` isn't checked. If it's larger, the grain never finishes, yet readers using
    `>=` treat it as available.
  - The header only promises that `flags` is updated.
  - Status: open

- [ ] **BUG-3: An invalid-flagged commit doesn't finish the grain.**
  - This contradicts the `MXL_GRAIN_FLAG_INVALID` documentation, which describes the flag as the proper way
    to move the ring buffer forward ([DW] L173-L178).
  - Status: issue filed `dmf-mxl/mxl#719` · PR open `dmf-mxl/mxl#728`

- [ ] **BUG-4: `mxlFlowWriterGetGrainInfo` can return a different grain's info.**
  - It reads slot `index % grainCount` and doesn't check that the stored index matches the requested one
    ([DW] L77-L82).
  - Status: open

- [ ] **BUG-5: `mxlCreateFlowSynchronizationGroup` writes through `group` without a null check.**
  - [FLOW] L666. Every other output pointer in the API is checked.
  - Status: open

---

## 2. Documentation needs updating (most out of date first)

- [ ] **DOC-1: `mxlFlowWriterOpenSamples` docs contradict themselves.**
  - The summary says the range starts at `index`; `\param index` calls it the "head index".
  - The code treats `index` as the end and writes `[index − count, index)` ([CW] L89-L102). The reader docs
    ("ending at a specific index") match the code.
  - Status: open

- [ ] **DOC-2: `mxlFlowReaderGetInfo` and `mxlFlowReaderGetRuntimeInfo` have copy-pasted text.**
  - Both say they return "the current flow config info value". `GetInfo` returns the full `mxlFlowInfo`;
    `GetRuntimeInfo` returns runtime info ([FLOWH] L245-L279).
  - Status: open

- [ ] **DOC-3: `mxlCreateInstance` says `in_options` is "Currently not used."**
  - The options are parsed and parse errors are logged ([INST] L414-L427).
  - The history duration comes from `options.json` in the domain directory ([INST] L385-L412), which isn't
    documented.
  - Status: open

- [ ] **DOC-4: The grain readers' timeout result is undocumented.**
  - `mxlFlowReaderGetGrain` and `mxlFlowReaderGetGrainSlice` never return `MXL_ERR_TIMEOUT`. On timeout they
    return `MXL_ERR_OUT_OF_RANGE_TOO_EARLY`, or `MXL_ERR_FLOW_INVALID` if the flow file was replaced
    ([DR] L110-L134).
  - The same applies to `mxlFlowSynchronizationGroupWaitForDataAt` ([SYNC] L124-L127).
  - The samples API documents this behaviour; the grain and sync-group APIs don't.
  - Status: open

- [ ] **DOC-5: `mxlFlowWriterCommitGrain` partial commits aren't documented.**
  - The docs say "done writing the grain", but the grain stays open until `validSlices == totalSlices`
    ([DW] L173-L178).
  - Committing an index other than the open one returns `MXL_ERR_INVALID_ARG` ([DW] L161-L164).
  - Status: open

- [ ] **DOC-6: `mxlReleaseFlowWriter` and `mxlDestroyInstance` delete flows without saying so.**
  - Releasing the last writer deletes the flow from the domain ([INST] L178-L182).
  - Destroying an instance deletes the flows of leaked writers that nobody else is using ([INST] L101-L116).
  - Status: open

- [ ] **DOC-7: `mxlFlowReaderGetGrain` returns `MXL_STATUS_OK` for invalid-flagged grains.**
  - This happens however many slices are valid ([DR] L183-L184). It's only implied by the flag's comment.
  - Status: open

- [ ] **DOC-8: `mxlFlowWriterOpenSamples` and `mxlFlowWriterCommitSamples` don't list their error cases or extra behaviour.**
  - Open returns `MXL_ERR_INVALID_ARG` when `count` exceeds the maximum write length, when `index < count`, or
    when the range overlaps committed data ([CW] L82-L96).
  - Commit wakes readers only at sync-batch boundaries ([CW] L169-L174) and zero-fills skipped gaps
    ([CW] L136-L161).
  - Commit with nothing open returns `MXL_ERR_INVALID_ARG`.
  - Status: open

- [ ] **DOC-9: Writers and readers for the same flow ID are shared within one instance.**
  - Each is a single reference-counted object per flow ID ([INST] L121-L143, L220-L231).
  - `mxlCreateFlowReader` ignores its `options` argument ([FLOW] L136).
  - Related: INT-5.
  - Status: open

- [ ] **DOC-10: Smaller undocumented edge cases.**
  - `mxlFlowReaderGetSamples` with a `count` larger than the readable window returns
    `MXL_ERR_OUT_OF_RANGE_TOO_LATE` rather than an invalid-argument error ([CR] L123-L156).
  - `MXL_GRAIN_VALID_SLICES_ANY` returns grains with 0 valid slices ([DR] L183).
  - Status: open

- [ ] **DOC-11: Releasing a reader silently removes it from every sync group.**
  - [INST] L154-L161.
  - Status: open

- [ ] **DOC-12: `mxlGarbageCollectFlows` always returns `MXL_STATUS_OK`.**
  - Errors are only logged at debug level ([INST] L303-L371, [MXL] L115-L136).
  - If a flow file can't be opened, the flow is treated as active and left alone.
  - Status: open

---

## 3. Needs maintainer intent (most clearly out of sync first)

- [ ] **INT-1: `mxlFlowWriterOpenGrain` doesn't enforce "must cancel or commit" first.**
  - The header says "must", but opening a different index silently drops the open grain ([DW] L84-L129).
  - Options: enforce it with an error code, or document that it's the caller's responsibility.
  - Related: DC-INT-3.
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
  - Decision: _pending_
  - Status: open

- [ ] **INT-3: `mxlFlowWriterOpenGrain` changes shared memory before anything is committed.**
  - Open marks skipped grains invalid and resets slot state ([DW] L98-L124). The sample writer does its gap
    handling at commit instead.
  - Related: DC-INT-1.
  - Decision: _pending_
  - Status: open

- [ ] **INT-4: Grain commit wakes readers on every partial commit.**
  - [DW] L180-L182. The header says readers are woken when "a new grain is available".
  - Sample commit wakes readers only at batch boundaries.
  - Related: DC-INT-2.
  - Decision: _pending_
  - Status: open

- [ ] **INT-5: Two `mxlCreateFlowWriter` calls in one instance share one open/commit state.**
  - [INST] L220-L231. Callers may expect independent writers.
  - Related: DOC-9.
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
  - Status: open

- [ ] **DC-BUG-2: `mxlCreateInstance` doesn't check `in_mxlDomain` for null.** (B2)
  - The pointer is used directly as a path ([MXL] L45-L52). Building a `std::filesystem::path` or
    `std::string` from a null `char const*` is undefined behaviour.
  - The surrounding `try` can't catch undefined behaviour, so the documented "returns NULL" isn't guaranteed.
  - Status: open

- [ ] **DC-BUG-3: The list of sync groups isn't thread-safe.** (B3)
  - `createFlowSynchronizationGroup` and `releaseFlowSynchronizationGroup` change `_syncGroups` without taking
    `_mutex` ([INST] L439-L456).
  - `releaseReader` walks the same list while holding the lock ([INST] L151-L160).
  - Doing both at once on different threads is a data race on a `std::forward_list`.
  - Status: open

- [ ] **DC-BUG-4: Permission checks differ between the grain and sample paths.** (A7)
  - The continuous writer and reader call `checkPermissions()` and throw if it fails ([CW] L26-L29,
    [CR] L19-L22).
  - The discrete writer and reader don't ([DW] L21-L30, [DR] L40-L51).
  - The same permission problem therefore fails at creation for one flow type and not at all for the other.
  - Status: open

- [ ] **DC-BUG-5: The reader reports permission errors as `MXL_ERR_UNKNOWN`.** (B5)
  - `mxlCreateFlowWriter` maps permission-type filesystem errors to `MXL_ERR_PERMISSION_DENIED`
    ([FLOW] L228-L240).
  - `mxlCreateFlowReader` maps only `ENOENT`; anything else becomes `MXL_ERR_UNKNOWN` ([FLOW] L154-L167).
  - The continuous reader's `checkPermissions()` failure is a `std::runtime_error`, which also becomes
    `MXL_ERR_UNKNOWN`.
  - Status: open

- [ ] **DC-BUG-6: The sample path never updates `lastReadTime`.** (A9)
  - The discrete reader touches the flow's access file on each successful read ([DR] L117-L121, L143-L147).
  - The continuous reader never opens or touches that file.
  - Reader activity on audio flows is therefore never reflected in their runtime info.
  - Status: open

- [ ] **DC-BUG-7: The grain path lets `headIndex` move backwards; the sample path doesn't.** (A1)
  - This is the same issue as BUG-1, seen by comparing the two paths.
  - `openSamples` rejects `index <= _lastCommittedIndex` and any overlap with committed data ([CW] L90-L96).
    The grain path has no equivalent check against the current `headIndex`.
  - Fix together with BUG-1.
  - Status: open

- [ ] **DC-BUG-8: `mxlIsFlowActive` reports any open failure as "flow not found".** (B6)
  - Every `open()` failure, including `EACCES`, returns `MXL_ERR_FLOW_NOT_FOUND` ([FLOW] L47-L58).
  - The code special-cases `ENOENT` and then returns the same code for everything else.
  - Status: open

- [ ] **DC-BUG-9: Grain commit with no flow data leaves the writer's state set.** (A6)
  - The discrete writer returns `MXL_ERR_UNKNOWN` and leaves `_currentIndex` set ([DW] L157-L187).
  - The continuous writer clears it first ([CW] L178-L182).
  - Minor: the writer is unusable at that point anyway.
  - Status: open

- [ ] **DC-BUG-10: `isExclusive()` and `makeExclusive()` behave differently when there's no flow data.** (A8)
  - The discrete writer returns `false` ([DW] L137-L155); the continuous writer throws ([CW] L220-L238).
  - `~Instance` catches the exception, but `Instance::releaseWriter` doesn't ([INST] L101-L116, L167-L189).
  - Minor: the case shouldn't happen in practice.
  - Status: open

### Discrete/Continuous Consistency: Documentation needs updating

- [ ] **DC-DOC-1: Discrete flows ignore the batch-size hints.** (A5)
  - `maxSyncBatchSizeHint` and `maxCommitBatchSizeHint` are stored for grain flows ([INST] L254-L255), but the
    discrete writer never reads them.
  - At minimum, document that the hints only affect continuous flows.
  - If the hints are meant to apply to grains too, see DC-INT-2.
  - Status: open

### Discrete/Continuous Consistency: Needs maintainer intent (most clearly out of sync first)

- [ ] **DC-INT-1: Gap handling happens at different times in the two paths.** (A2)
  - The grain path marks skipped grains invalid at open, in shared memory, before anything is committed
    ([DW] L98-L115).
  - The sample path zero-fills skipped samples at commit ([CW] L136-L161).
  - In the grain path, open followed by cancel still invalidates grains; in the sample path it doesn't.
    One timing should become the standard.
  - Related: INT-3.
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-2: Reader wake-ups differ between the two paths.** (A5)
  - Grain commit wakes readers on every commit, including partial ones ([DW] L180-L182).
  - Sample commit wakes them only at sync-batch boundaries ([CW] L169-L174, L192-L218), using the hints that
    the grain path ignores.
  - Question: should grain flows batch their wake-ups too, or is per-commit signalling deliberate?
  - Related: INT-4, DC-DOC-1.
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-3: Opening while something is already open is silently allowed in both paths.** (A3)
  - In both writers, a second open replaces the current one without an error ([DW] L117-L127,
    [CW] L116-L117).
  - Whatever is decided for INT-1 should apply to both paths.
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-4: There's no single rule for which error code a bad handle returns.** (B4)
  - For a bad reader:
    - `mxlFlowSynchronizationGroupAddReader` and `AddPartialGrainReader` return `MXL_ERR_INVALID_FLOW_READER`
      ([FLOW] L702-L756).
    - `mxlFlowSynchronizationGroupRemoveReader` ([FLOW] L760-L778) and `mxlReleaseFlowReader`
      ([FLOW] L172-L189) return `MXL_ERR_INVALID_ARG`.
  - The writer functions have the same split between `MXL_ERR_INVALID_FLOW_WRITER` and
    `MXL_ERR_INVALID_ARG`.
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-5: A malformed flow definition returns `MXL_ERR_UNKNOWN`.** (B7)
  - Parse failures in `mxlCreateFlowWriter` end up in the generic `std::exception` handler ([FLOW] L242-L246).
  - `mxl.h` warns callers not to rely on `MXL_ERR_UNKNOWN`, so a caller can't detect "bad flowDef".
  - `MXL_ERR_INVALID_ARG` looks like the natural code, but switching changes behaviour callers can see.
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-6: Commit with nothing open gives the same result by different routes.** (A4)
  - Both paths return `MXL_ERR_INVALID_ARG`.
  - The grain path gets there because the index doesn't match ([DW] L161-L164); the sample path checks
    explicitly ([CW] L131-L134).
  - The grain path relies on an accident: a commit whose `grainInfo.index` equals `MXL_UNDEFINED_INDEX` would
    pass the check.
  - Question: should both paths use an explicit "nothing open" check?
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-7: Exception logging varies.** (B9)
  - Some entry points log the exception before returning `MXL_ERR_UNKNOWN`, e.g. `mxlCreateFlowWriter`,
    `mxlIsFlowActive`, `mxlGetFlowDef`.
  - Most `catch (...)` blocks return silently, e.g. `mxlCreateFlowReader` and all the reader/writer data calls.
  - Question: should every `MXL_ERR_UNKNOWN` be logged?
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-8: Argument checks happen in different places (style only).** (B8)
  - `mxlCreateFlowWriter`, `mxlReleaseFlowWriter`, `mxlGetFlowDef` and `mxlFlowWriterCommitGrain` check
    arguments before the `try`; the rest check inside it ([FLOW]).
  - Behaviour is the same either way, but one convention would make DC-INT-4 and DC-INT-7 easier to keep
    consistent.
  - Decision: _pending_
  - Status: open

- [ ] **DC-INT-9: `mxlDestroyInstance` checks for null after deleting (style only).** (B10)
  - It deletes first and then tests for null to pick the return code ([MXL] L104-L105).
  - Harmless, since deleting a null pointer is a no-op, but every other entry point checks first.
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
