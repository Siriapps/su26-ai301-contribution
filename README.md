# su26-ai301-contribution
This is my contribution log for Codepath AI 301 course

# Contribution 1: Feature: Trash bin / soft delete for conversations, memories, and tasks

**Contribution Number:** 1  
**Student:** Lakshmi Siri Appalaneni  
**Issue:** [BasedHardware/omi#5092](https://github.com/BasedHardware/omi/issues/5092)  
**Status:** Phase II Complete

---

## Why I Chose This Issue

I chose this issue because I understand what the Omi team is asking for and believe I can help implement it. The feature: moving deleted conversations, memories, and tasks to a Trash bin with restore and permanent-delete options is a clear, well-defined problem with a practical user impact. Accidental deletion of irreplaceable life recordings is a real data-safety gap, and a soft-delete system with configurable auto-purge (e.g., 30 days) is a pattern I am familiar with from apps like Gmail and Google Photos.

The technologies involved also align well with my skills. Omi uses Python for the backend and Dart for the mobile app, both of which I am comfortable working in. I hope to learn how soft-delete is implemented end-to-end in a production open-source project — from schema changes and backend API design to UI flows like undo toasts and bulk trash actions.

---

## Understanding the Issue

### Problem Description

Deleting a conversation, memory, or task in Omi is currently permanent and instant. There is no undo, no recovery, and no Trash folder. For a device that records a user's life, one accidental tap can permanently erase meaningful content.

### Expected Behavior

Deleted conversations, memories, and tasks should move to a Trash bin instead of being permanently removed. Users should be able to restore items from Trash or permanently delete them through a separate icon. Trash should auto-purge after a configurable retention period (default: 30 days). An undo toast should appear immediately after deletion for quick recovery.

### Current Behavior

Deletion is instant and permanent. Conversations may use a `discarded` flag on the backend, but there is no user-facing Trash experience. Memories and tasks lack soft-delete support entirely. Vector embeddings and audio recordings are purged on delete rather than on permanent removal from Trash.

### Affected Components

Backend (Python): conversation, memory, and task schemas; soft-delete fields (`deleted_at` or extended `discarded` flag); scheduled auto-purge job; API endpoints for trash, restore, and permanent delete. Mobile app (Dart): Trash UI (tab or section per content type), undo snackbar, bulk actions, and auto-purge settings. Vector embeddings and audio storage should only be purged on permanent delete, not on soft delete.

---

## Reproduction Process

### Environment Setup

I forked and cloned the [BasedHardware/omi](https://github.com/BasedHardware/omi) repository into my fork ([Siriapps/omi](https://github.com/Siriapps/omi)) on a Windows machine with the repo under OneDrive. Reproduction combined **static code tracing** (backend routers + Flutter providers) with **local app setup** on an Android emulator.

**Blockers encountered and resolved:**

1. **Android emulator** — Booted a Pixel_7 AVD after installing the Flutter/Android toolchain on Windows.
2. **`flutter pub get` failure on `whisper_flutter_new`** — Dart-on-Windows has an ANSI-encoding bug that mangles Japanese commas in that git dependency's `pubspec.yaml`. **Fix:** local `path` override pointing at a vendored copy with an ASCII-only description.
3. **`gen-l10n` failure on `lib/l10n`** — OneDrive had set a **ReadOnly** attribute on localization files, blocking code generation. **Fix:** cleared the ReadOnly flag on the affected paths.
4. **`build_runner` reported failure** — Only the `envied` generator tripped on an integration-test file; all 6 required `.g.dart` files generated successfully. Treated as a non-blocking warning for local dev.

**Current status:** Environment setup complete. `flutter run --flavor dev` builds and runs on the Pixel_7 emulator. Working branch: [`fix-issue-5092`](https://github.com/Siriapps/omi/tree/fix-issue-5092).

See **Week 2 Progress** under Implementation Notes for full detail on each blocker, error messages, root causes, and fixes.

### Steps to Reproduce

1. Fork [BasedHardware/omi](https://github.com/BasedHardware/omi) and clone your fork locally (e.g. `git clone https://github.com/Siriapps/omi.git`).
2. Create a working branch: `git checkout -b fix-issue-5092` and push to your fork.
3. **Backend — conversations:** Open `backend/routers/conversations.py` and locate `DELETE /v1/conversations/{id}` (~line 382). Follow the call chain into `database/conversations.py` — confirm it calls Firestore `.delete()` and triggers Pinecone vector + GCS audio purge inline.
4. **Backend — memories:** Open `backend/routers/memories.py` and locate `DELETE /v3/memories/{id}` (~line 172). Confirm it hard-deletes the Firestore doc and calls `delete_memory_vector`.
5. **Backend — action items:** Open `backend/routers/action_items.py` and locate `DELETE /v1/action-items/{id}` (~line 448). Confirm hard delete with immediate vector/FCM cleanup.
6. **Historical context:** Check commit `d3a2aa6cf` ("No more soft-delete on database #2515") and `backend/migration/remove_soft_deleted_documents.py` — a prior `deleted: true` soft-delete was removed; reusing that field would collide with the existing purge migration.
7. **App — undo/trash UI:** In the Flutter app, inspect `app/lib/pages/memories/page.dart` (~line 53) for the 4-second client-side Undo toast; `conversation_provider.dart` (~line 799) for the unused `undoDeletedConversation()`; and action-items pages for any undo — confirm there is no Trash tab, restore screen, or auto-purge setting anywhere.
8. **Observed result:** Deletion is permanent at the database level across all three content types. The app offers at most a brief local undo window (memories/conversations only) with no server-side recovery — confirming [BasedHardware/omi#5092](https://github.com/BasedHardware/omi/issues/5092) is still valid.

### Reproduction Evidence

- **Branch link:** https://github.com/Siriapps/omi/tree/fix-issue-5092
- **Screenshots/logs:** Not required — issue confirmed via backend and Flutter code inspection (Steps 3–7 above). Local dev environment verified by successful `flutter run --flavor dev` on Pixel_7 emulator.
- **My findings:** The Trash/Recycle Bin feature is entirely missing. All three delete paths hard-delete Firestore documents and immediately purge side effects (Pinecone vectors, GCS audio, FCM reminders). A previous `deleted: bool` soft-delete existed but was removed in PR #2515; the leftover migration would conflict with reintroducing that pattern, so a new `deleted_at` timestamp field is required. The Flutter app has partial client-side undo toasts but no Trash UI, restore endpoints, or auto-purge.

---

## Solution Approach

### Analysis

The missing Trash feature is not a single bug but a **systemic hard-delete architecture** across backend and app, compounded by a prior soft-delete rollback.

**Backend — delete means destroy.** All three content types follow the same pattern: the public DELETE route calls a database helper that invokes Firestore `.delete()` on the document, then immediately runs destructive side effects in the same request:
- **Conversations** (`routers/conversations.py:382` → `database/conversations.py:540`): hard delete + Pinecone vector purge + cascade deletion of related audio, memory vectors, and action items.
- **Memories** (`routers/memories.py:172` → `database/memories.py:290`): hard delete + `delete_memory_vector`.
- **Action items** (`routers/action_items.py:448` → `database/action_items.py:491`): hard delete + vector purge + FCM reminder cancellation.

There is no intermediate "trashed" state, no restore endpoint, and no trash-list query. Once the DELETE handler completes, the data and its embeddings are gone.

**Historical rollback blocks naive reintroduction.** Omi previously used a `deleted: bool` soft-delete flag, but commit `d3a2aa6cf` ("No more soft-delete on database #2515") removed it. The migration script `backend/migration/remove_soft_deleted_documents.py` still actively purges any document where `deleted == True`. Reusing that field would risk existing docs being destroyed by the migration. A new field — `deleted_at: Optional[datetime]` — is required and is safe: the migration's `FieldFilter('deleted', '==', True)` will never match it, and legacy documents without `deleted_at` naturally read as active (no data migration needed).

**App — cosmetic undo, not real recovery.** The Flutter layer has partial UX hints but no server-backed Trash:
- **Memories:** a 4-second client-side "Undo" toast (`memories/page.dart:53-119`) that only works if the delete request has not yet completed.
- **Conversations:** a 3-second delay plus an `undoDeletedConversation()` method (`conversation_provider.dart:799`) that is wired but unused in the main flow.
- **Action items:** no undo at all.
- **Nowhere:** no Trash tab/screen, no restore API call, no permanent-delete confirmation flow, no auto-purge setting, no bulk empty-trash action.

**Root cause summary:** Deletion was intentionally simplified to hard delete in PR #2515. Issue #5092 asks to reintroduce recoverable deletion, but the implementation must be greenfield (new field, new endpoints, new UI) while deferring Pinecone/GCS/FCM cleanup to a separate permanent-delete path — otherwise soft delete would still destroy recoverability.

### Proposed Solution

Reintroduce **soft delete via `deleted_at`** across conversations, memories, and action items, backed by new API routes and surfaced in the Flutter app as a Trash experience with undo toasts and configurable auto-purge.

**Backend changes:**
1. Add `deleted_at: Optional[datetime] = None` to all three schemas. Leave `discarded` on conversations untouched (it means "failed processing", not user deletion).
2. Refactor DB layer: rename existing `delete_*` → `permanently_delete_*`; add `soft_delete_*`, `restore_*`, `get_trashed_*`; filter `deleted_at is None` in all list/get queries (mirroring the existing `invalid_at` pattern on memories).
3. Flip existing DELETE handlers to soft-delete only (stamp `deleted_at`, no vector/audio purge). Add per-type routes: `POST .../restore`, `DELETE .../permanent`, `GET .../trash` (plus `POST .../trash/empty` for action items).
4. Move all destructive side effects (Pinecone, GCS audio, FCM, cascade deletes) into the permanent-delete path only — triggered by explicit user action or the auto-purge job.
5. Add a daily auto-purge job (`trash_purge.py`) wired into the existing Cloud Run job infrastructure, default 30-day retention (`TRASH_RETENTION_DAYS`), with optional per-user override via `users/{uid}.trash_retention_days`.

**Flutter changes:**
1. Extend API clients and models (`DateTime? deletedAt`) for restore, permanent delete, trash list, and empty trash.
2. Wire providers with trashed-item lists and undo flows; extract the memory undo toast into a shared `showUndoToast()` used by all three content types.
3. Add a reusable `TrashListView` (restore + delete-forever per row, empty-trash in app bar, "Auto-deletes in N days" label), reachable from each content type's overflow menu.
4. Add a retention setting (15 / 30 / 60 days) persisted locally and synced to Firestore.
5. Add l10n keys for all trash-related strings across locales.

**Design constraints:**
- Filter trashed items in Python (`doc.get('deleted_at') is None`), not via new Firestore composite indexes — same approach as `invalid_at`.
- For conversations, add `include_trashed=False` query param analogous to existing `include_discarded`.
- On soft-delete of action items, still cancel FCM reminders immediately (user expectation), but defer vector deletion to permanent delete.
- Land changes in staged PRs (backend models → endpoint flip → purge job → Flutter) to keep reviews manageable.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Deleting a conversation, memory, or task in Omi is instant and permanent. All three backend DELETE endpoints call Firestore `.delete()` and immediately purge Pinecone vectors, GCS audio, and FCM reminders in the same request. The app has no Trash folder, no restore API, and no auto-purge — only brief client-side undo toasts that cannot recover data once the server request completes. A prior `deleted: bool` soft-delete was removed and must not be reused.

**Match:** Existing codebase patterns to reuse:
- `invalid_at` on memories (`memories.py:112-116`) — Python-side filtering without new Firestore indexes; `deleted_at` follows this model.
- `discarded` + `include_discarded` on conversations — template for `include_trashed` (separate semantics: discarded = failed processing).
- `invalidate_memory()` (`memories.py:269`) — stamp-a-timestamp instead of delete; model for `soft_delete_memory()`.
- Memory undo toast overlay (`memories/page.dart:53-119`) — reference UX for shared `showUndoToast()`.
- Hourly Cloud Run notification job (`jobs.py`, `notifications.should_run_job`) — reuse job gating pattern for daily trash auto-purge.

**Plan:** Six-step implementation (full file-level detail in `implementation_plan.md`):

| Step | Scope | Key files |
|------|-------|-----------|
| 1 | Backend models — add `deleted_at` | `conversation.py`, `memories.py` (MemoryDB), action-item response model |
| 2 | Backend DB layer — soft/restore/trash-list + query filters | `database/conversations.py`, `database/memories.py`, `database/action_items.py` |
| 3 | Backend routers — flip DELETE to soft; add restore/permanent/trash routes | `routers/conversations.py`, `routers/memories.py`, `routers/action_items.py` |
| 4 | Auto-purge job — 30-day retention | `utils/other/trash_purge.py`, `utils/other/jobs.py` |
| 5 | Flutter — API clients, schema, providers, undo toasts | `conversations.dart`, `memories.dart`, action-items client, providers |
| 6 | Flutter — Trash UI, empty-trash, retention setting, l10n | `TrashListView`, overflow menus, SharedPreferences + Firestore sync |

Suggested PR sequence: (1) backend models + DB layer + unit tests, no behavior change; (2) endpoint flip + new routes; (3) auto-purge job; (4) Flutter schema + clients + providers; (5) Flutter Trash views + settings + l10n.

**Implement:** https://github.com/Siriapps/omi/tree/fix-issue-5092 — feature implementation commits will land on this branch during Phase III.

**Review:** Before opening upstream PRs, verify against Omi contribution guidelines:
- [ ] No reuse of `deleted: bool` (migration collision).
- [ ] `discarded` and `deleted_at` kept semantically separate on conversations.
- [ ] Soft-delete does not call Pinecone/GCS/FCM purge; permanent-delete does.
- [ ] List queries exclude trashed items; trash-list endpoint returns only trashed items ordered by `deleted_at` desc.
- [ ] Legacy documents without `deleted_at` behave as active.
- [ ] Backend formatted with `black --line-length 120 --skip-string-normalization`.
- [ ] Flutter `*.g.dart` regenerated via `build_runner`, not hand-edited.
- [ ] l10n keys added for all supported locales.
- [ ] New backend tests added to `test.sh`.

**Evaluate:** Verification plan per content type:

*Backend (`bash test.sh`):*
- Soft-delete stamps `deleted_at`; item disappears from list queries but remains fetchable by ID.
- Restore clears `deleted_at` via `firestore.DELETE_FIELD`; item reappears in normal lists.
- Permanent delete removes the Firestore doc and invokes vector/audio/FCM cleanup (mock-asserted); soft-delete does not.
- Trash-list returns only soft-deleted docs, sorted by `deleted_at` descending.
- Auto-purge removes only items older than retention; respects per-user `trash_retention_days` override.
- Legacy docs without `deleted_at` validate as active; migration filter `deleted == True` never matches `deleted_at`.

*App (Pixel_7 emulator / simulator):*
- Swipe-delete conversation, memory, and task → undo toast appears → undo restores item.
- Open Trash from overflow menu → restore one item, permanently delete another → confirm server state.
- Empty trash → all trashed items permanently removed.
- Change retention setting (15 / 30 / 60 days) → persists locally and syncs to Firestore.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes


### Week 2 Progress

**Focus:** Local development environment setup on Windows — getting the Flutter app building and running on an Android emulator. No feature code yet; this week was entirely infrastructure and toolchain work.

Omi's docs assume macOS for the desktop app and do not document Windows-specific pitfalls. The repo also lives under OneDrive, which introduced file-sync-related failures on top of platform issues. Getting from `git clone` to a running app on Pixel_7 took resolving **four separate blockers** in sequence:

**1. Android emulator setup**
- Installed Flutter SDK, Android Studio, and platform tools on Windows.
- Created and booted a **Pixel_7** AVD — first time running a Flutter app target on this machine.
- Confirmed `flutter doctor` passed Android toolchain checks before attempting `flutter run`.

**2. `flutter pub get` — `whisper_flutter_new` git dependency failure**
- **Error:** `pub get` failed when resolving the `whisper_flutter_new` package pulled from git.
- **Root cause:** Dart-on-Windows has an ANSI-encoding bug that corrupts non-ASCII characters when parsing remote pubspec files. The dependency's pubspec contains **Japanese commas** in its description field, which get mangled on Windows and break resolution.
- **Fix:** Vendored the package locally and added a **`path` override** in `pubspec.yaml` pointing to the local copy with an **ASCII-only description**. After the override, `flutter pub get` completed successfully.

**3. `gen-l10n` — localization generation blocked**
- **Error:** `flutter gen-l10n` failed when writing to `lib/l10n/`.
- **Root cause:** OneDrive sync had silently set a **ReadOnly** attribute on localization files/directories, preventing the code generator from overwriting `.arb` outputs or generated Dart files.
- **Fix:** Cleared the ReadOnly flag on affected paths under `lib/l10n/`. Re-ran `gen-l10n` successfully.
- **Lesson:** Cloning Flutter projects into OneDrive-synced folders can cause subtle permission issues — worth checking file attributes when codegen fails with write errors.

**4. `build_runner` — envied generator failure (non-blocking)**
- **Error:** `dart run build_runner build` exited with failure after running all generators.
- **Root cause:** The `envied` generator tripped on an **integration-test file** it was not meant to process. This is unrelated to the trash feature.
- **Outcome:** All **6 required `.g.dart` files** for app development were generated successfully. Treated the envied failure as a non-blocking warning and proceeded.
- **Decision:** Did not patch upstream integration-test layout this week — out of scope for #5092 and does not block local dev.

**5. First Gradle build and app launch**
- Ran `flutter run --flavor dev` targeting the Pixel_7 emulator.
- First cold **Gradle build** took several minutes (expected for a large Flutter project on first compile).
- App installed and launched successfully on the emulator — local dev environment is now fully operational for Phase III implementation and manual testing.

**Week 2 outcome:** Environment setup complete. Can now implement and manually verify Trash UI flows on-device during Phase III. The effort this week was disproportionately spent on toolchain/platform issues rather than feature work, which is common when onboarding to a large cross-platform OSS project on a non-default OS.

### Code Changes

- **Files modified (Week 2 — local dev only, not yet pushed as feature PR):** `pubspec.yaml` (path override for `whisper_flutter_new`), vendored whisper package copy, ReadOnly attribute fixes on `lib/l10n/` (filesystem, not a code commit).
- **Key commits:** Contribution log commits on `su26-ai301-contribution` (`first log - chose issue`, `Phase 1 complete`); omi fork branch [`fix-issue-5092`](https://github.com/Siriapps/omi/tree/fix-issue-5092) ready for Phase III implementation commits.
- **Approach decisions:** Used code tracing to confirm reproduction before finishing app setup (validates issue even if emulator failed); chose `deleted_at` over `deleted: bool` after finding migration collision; staged PR plan to land backend before Flutter; path override for whisper rather than waiting on upstream Dart-on-Windows fix.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [BasedHardware/omi#5092 — Trash bin / soft delete feature request](https://github.com/BasedHardware/omi/issues/5092)
- [Siriapps/omi fork — fix-issue-5092 branch](https://github.com/Siriapps/omi/tree/fix-issue-5092)
- Omi commit `d3a2aa6cf` — "No more soft-delete on database #2515" (historical context for `deleted_at` decision)
- `backend/migration/remove_soft_deleted_documents.py` — existing purge migration that blocks reuse of `deleted: bool`
- Local `implementation_plan.md` — full file-level implementation plan for Phase III
