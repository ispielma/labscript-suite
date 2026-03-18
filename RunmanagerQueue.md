# RunmanagerQueue Multi-Repo Implementation Plan

## Branch Bases
- `blacs`: `RunmanagerQueue` created from current checked-out `master`
- `runmanager`: `RunmanagerQueue` created from current checked-out `master`
- `labscript-utils`: `RunmanagerQueue` created from current checked-out `master`
- `labscript-suite`: `RunmanagerQueue` created from current checked-out `master`

## Summary
- Move authoritative shot queue ownership from BLACS into runmanager.
- BLACS becomes a shot executor that requests the next shot from runmanager, validates the returned file, acknowledges receipt, and then executes it.
- Keep BLACS direct load, but reduce it to a single local override shot instead of a BLACS-side queue.
- Move lyse communication ownership from BLACS into runmanager.
- BLACS buffers completed-shot notifications to runmanager at runtime and retries delivery if runmanager is temporarily unavailable.
- Change canonical runmanager globals storage from HDF5 to TOML.
- Keep legacy HDF5 globals readable as import-only migration sources.
- Extract the reusable queue editor into `labscript_utils.qtwidgets` as a code-only widget.

## Delegation
- Main agent:
  - cross-repo integration
  - protocol decisions
  - migration rules
  - verification and conflict resolution
- Worker 1, `labscript-utils`:
  - `ShotQueueWidget`
  - reusable queue tree/editor extraction
- Worker 2, `runmanager` storage/UI:
  - TOML globals backend
  - HDF5 import layer
  - new per-global fields and group table redesign
- Worker 3, `runmanager` queue/protocol:
  - queue controller
  - queue tab
  - `Engage` behavior change
  - ZMQ queue API
  - empty-queue policies and standard shot
- Worker 4, `blacs`:
  - executor/pull client
  - single local direct-load override shot
  - fallback repeat
  - buffered shot-complete notifications back to runmanager
  - plugin compatibility surface and plugin updates

## Execution Order
1. Setup
- Create and switch to `RunmanagerQueue` in all modified repos.
- Save this file in `labscript-suite`.

2. Shared UI extraction
- Add `ShotQueueWidget` in `labscript_utils.qtwidgets`.
- Scope:
  - drag/drop
  - add/delete/clear/reorder
  - selection handling
  - queue serialization
- Do not include execution logic, BLACS validation, or networking.

3. Runmanager globals migration
- Replace HDF5-assuming globals CRUD with a storage abstraction.
- Canonical format becomes TOML.
- Legacy `.h5/.hdf5` globals remain readable as import sources only.
- Fixed migration mapping:
  - old value -> `default`
  - old units -> `units`
  - `scan_enabled = false`
  - `scan = ""`
  - `expansion = ""`
- Legacy HDF5 scan semantics are intentionally not preserved.
- On first edit of a legacy HDF5 globals file, prompt for TOML destination, write TOML, switch the open document to TOML, and apply the edit there.

4. Runmanager globals UI redesign
- Change group table columns to:
  - `Delete`
  - `Name`
  - `Default`
  - `Units`
  - `Scan?`
  - `Scan`
  - `Expansion`
- `Default` is required.
- `Scan?` is a checkbox.
- `Scan` is editable only when `Scan?` is checked.
- `Expansion` applies only to `Scan` and is disabled or blank when `Scan?` is unchecked.
- Standard shots use only `Default`.
- Queued scan shots use `Scan` when enabled, otherwise `Default`.

5. Runmanager queue ownership
- Add a runmanager `Queue` tab using `ShotQueueWidget`.
- Queue items are logical shot descriptors, not compiled file paths.
- `Engage` appends items to the runmanager queue instead of pushing to BLACS.
- Queue settings:
  - compile mode: `precompile` or `compile_on_request`
  - empty-queue policy: `stop`, `repeat_last`, `repeat_standard`
  - standard-shot labscript path
- `compile_on_request` semantics:
  - queue order and scan-point selection are frozen when queued
  - current `Default` values are used at compile time for unserved shots

6. Runmanager pull API
- Extend the existing ZMQ server/client with:
  - `queue_request_next`
  - `queue_ack_received`
  - `queue_get_state`
  - `queue_get_items`
  - `queue_clear`
  - `queue_delete`
  - `queue_move`
- `queue_request_next()` returns `{offer_id, agnostic_path, source_kind}`.
- `queue_ack_received(valid=True)` removes the shot immediately.
- `queue_ack_received(valid=False)` or ack timeout restores the shot to the queue head.
- No completion ack in v1.

7. BLACS executor conversion
- Remove BLACS queue ownership.
- Replace it with an executor that keeps:
  - current shot state
  - last completed shot
  - pause/abort/status state
  - one local direct-load override shot
  - fallback repeat state
- Keep a runtime-only buffer of completed-shot notifications waiting to be delivered to runmanager.
- BLACS direct load becomes single-shot local override only.
- When idle, BLACS requests the next shot from runmanager, validates the file, acknowledges receipt, then executes it.
- If runmanager is empty or unreachable, BLACS may rerun the last completed shot when fallback repeat is enabled.
- When a shot finishes, BLACS does not talk to lyse directly. Instead it enqueues a completion notification to runmanager and retries delivery in the background until successful or BLACS exits.
- BLACS shows a GUI counter for the number of completed-shot notifications still buffered for runmanager.
- The completed-shot notification buffer is runtime only and is explicitly not restored across BLACS restarts.

8. BLACS plugin compatibility
- Keep `BLACS['experiment_queue']` as a compatibility alias in v1.
- Preserve:
  - `get_status()`
  - `set_status()`
  - `master_pseudoclock`
- Update:
  - `cycle_time`
  - `progress_bar`
  - `delete_repeated_shots`
- Repeated-shot detection should use explicit HDF5 repeat metadata instead of filename heuristics.

9. Runmanager lyse ownership
- Move the lyse submission widget and submission loop into runmanager.
- Place the lyse widget under `View shot(s)` in the runmanager top controls.
- Add a runmanager remote API method `notify_shot_complete(filepath)` so BLACS can report completed shots to runmanager.
- Runmanager converts the agnostic/shared-drive filepath back to a local path before queueing it for lyse submission.
- Runmanager configuration persists only tracked lyse widget settings:
  - lyse hostname
  - whether analysis submission is enabled
- Runmanager does not treat transient lyse retry state as saved configuration state.
- `ports.lyse` remains a required runmanager labconfig entry; there is no silent fallback port.

10. Docs
- Update BLACS docs for pull-based execution.
- Update BLACS docs to remove BLACS-owned lyse submission and queue descriptions that no longer apply.
- Update runmanager docs for queue ownership, TOML globals, migration behavior, compile modes, and empty-queue policies.
- Update runmanager docs for runmanager-owned lyse submission.
- Keep this file aligned with actual implementation.

## Public Data Shape
```toml
format = "runmanager_globals"
version = 1

[groups."Group Name".globals.my_var]
default = "780e-9"
units = "s"
scan_enabled = true
scan = "linspace(700e-9, 900e-9, 11)"
expansion = "outer"
```

## Standard Shot Rules
- `repeat_standard` uses:
  - the dedicated standard-shot labscript path
  - current active groups
  - current `Default` values only
  - no scan fields
- The standard shot must compile to exactly one shot.

## Verification Targets
- Branch setup stays on `RunmanagerQueue` in all touched repos.
- `labscript-utils`:
  - widget add/delete/clear/reorder/drag-drop/state restore
- `runmanager`:
  - TOML create/open/edit/save
  - HDF5 import and convert-on-save
  - new globals table behavior
  - queue append/reorder/delete/clear
  - `precompile` vs `compile_on_request`
  - frozen scan overrides plus dynamic defaults
  - standard-shot generation
  - empty-queue policies
  - queue ZMQ API behavior
  - `notify_shot_complete` remote API behavior
  - lyse widget startup and shutdown behavior
  - tracked analysis settings persist without transient retry state affecting dirty-config detection
- `blacs`:
  - pull next shot, validate, ack
  - single direct-load override shot
  - fallback repeat on empty queue and communication failure
  - completed-shot notifications buffer and retry correctly when runmanager is unavailable
  - notification-buffer counter updates correctly
  - notification buffer is not restored after BLACS restart
  - plugin compatibility surface
- Cross-repo:
  - runmanager queue and BLACS executor interoperate end-to-end
  - all repeated or fallback shots get fresh HDF5 files
  - completed shots flow BLACS -> runmanager -> lyse without BLACS talking to lyse directly

## Implemented Follow-Ups
- BLACS local override UI was reduced to a compact file field plus browse button, and the old queue-pane remnants were removed from the BLACS layout.
- Runmanager queue-tab layout was cleaned up so the queue widget fills the page cleanly and the queue actions use the widget context menu rather than inline move buttons.
- BLACS no longer contains the old `analysis_submission` module or UI; those stale files were removed after lyse ownership moved to runmanager.
- Runmanager `AnalysisSubmission` startup was fixed so backing attributes are initialized before property setters run during widget construction.
