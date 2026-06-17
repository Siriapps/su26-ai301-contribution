# su26-ai301-contribution

This is my contribution log for Codepath AI 301 course

---

# Contribution 1: Feature: Trash bin / soft delete for conversations, memories, and tasks

**Contribution Number:** 1  
**Student:** Lakshmi Siri Appalaneni  
**Issue:** [BasedHardware/omi#5092](https://github.com/BasedHardware/omi/issues/5092)  
**Status:** Stopped — upstream PR submitted by another contributor; work halted at Phase II

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

**Backend — delete means destroy.** All three content types follow the same pattern: the public DELETE route calls a database helper that invokes Firestore `.delete()` on the document, then immediately runs destructive side effects in the same request.

**Historical rollback blocks naive reintroduction.** Omi previously used a `deleted: bool` soft-delete flag, but commit `d3a2aa6cf` removed it. The migration script `backend/migration/remove_soft_deleted_documents.py` still actively purges any document where `deleted == True`. A new field — `deleted_at: Optional[datetime]` — is required.

**App — cosmetic undo, not real recovery.** The Flutter layer has partial UX hints but no server-backed Trash.

**Root cause summary:** Deletion was intentionally simplified to hard delete in PR #2515. Issue #5092 asks to reintroduce recoverable deletion, but the implementation must be greenfield (new field, new endpoints, new UI) while deferring Pinecone/GCS/FCM cleanup to a separate permanent-delete path.

### Proposed Solution

Reintroduce **soft delete via `deleted_at`** across conversations, memories, and action items, backed by new API routes and surfaced in the Flutter app as a Trash experience with undo toasts and configurable auto-purge. Full six-step plan saved in `implementation_plan.md`.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Deleting a conversation, memory, or task in Omi is instant and permanent.

**Match:** `invalid_at` filtering pattern, `discarded` + `include_discarded`, memory undo toast UX, Cloud Run job gating.

**Plan:** Six-step backend + Flutter implementation (models → DB layer → routers → purge job → Flutter clients → Trash UI).

**Implement:** https://github.com/Siriapps/omi/tree/fix-issue-5092

**Review / Evaluate:** Backend `test.sh` + emulator manual testing per content type.

---

## Why Work Stopped

While preparing for Phase III implementation, another contributor submitted a pull request addressing [BasedHardware/omi#5092](https://github.com/BasedHardware/omi/issues/5092). Continuing would duplicate upstream work and risk merge conflicts. I completed Phase II (environment setup, reproduction via code tracing, and a detailed implementation plan) but did not open an upstream PR or land feature commits. I pivoted to a new issue in the Lance project for Contribution 2.

**Phase summary:**

| Phase | Status |
|-------|--------|
| Phase I — Issue understanding | Complete |
| Phase II — Reproduction & plan | Complete |
| Phase III — Implementation | Not started (stopped) |
| Phase IV — Pull request | Not started (stopped) |

---

## Testing Strategy

Not executed — work stopped before Phase III. Planned tests documented in `implementation_plan.md`.

---

## Implementation Notes

### Week 1 Progress (Phase I)

- Selected [BasedHardware/omi#5092](https://github.com/BasedHardware/omi/issues/5092).
- Forked upstream to [Siriapps/omi](https://github.com/Siriapps/omi) and created branch [`fix-issue-5092`](https://github.com/Siriapps/omi/tree/fix-issue-5092).
- Traced all three backend delete paths and confirmed permanent deletion with no server-side Trash.
- Investigated historical `deleted: bool` rollback (PR #2515); decided `deleted_at` is the safe reintroduction path.
- Drafted full implementation plan (`implementation_plan.md`).

### Week 2 Progress (Phase II)

Environment setup complete on Windows (Pixel_7 emulator, whisper path override, OneDrive ReadOnly fix on l10n, build_runner envied warning non-blocking). No feature code pushed upstream. Branch [`fix-issue-5092`](https://github.com/Siriapps/omi/tree/fix-issue-5092) remains available locally for reference.

---

## Pull Request

**PR Link:** Not submitted — work stopped before PR

**Status:** Stopped — upstream PR by another contributor

---

## Learnings & Reflections

### Technical Skills Gained

- Tracing delete flows across Python backend routers, database layer, and Flutter providers.
- Understanding soft-delete design constraints (`deleted_at` vs legacy `deleted: bool` migration collision).
- Windows Flutter OSS setup (emulator, dependency overrides, codegen).

### Challenges Overcome

- Large Omi repo (~1+ GB) plus Android emulator made local builds slow and resource-heavy.
- Windows-specific blockers: whisper pubspec encoding, OneDrive ReadOnly on l10n files.

### What I'd Do Differently Next Time

- Check for existing upstream PRs on the issue earlier before investing in a full implementation plan.
- Clone large Flutter repos outside OneDrive-synced folders to avoid file-attribute issues.

---

## Resources Used

- [BasedHardware/omi#5092](https://github.com/BasedHardware/omi/issues/5092)
- [Siriapps/omi — fix-issue-5092](https://github.com/Siriapps/omi/tree/fix-issue-5092)
- Omi commit `d3a2aa6cf` — "No more soft-delete on database #2515"
- Local `implementation_plan.md` — full file-level implementation plan

---

# Contribution 2: Bug: index out of bounds panic in FTS inverted index builder (#7313)

**Contribution Number:** 2  
**Student:** Lakshmi Siri Appalaneni  
**Issue:** [lance-format/lance#7313](https://github.com/lance-format/lance/issues/7313)  
**Status:** Phase II Complete

---

## Why I Chose This Issue

After Contribution 1 stopped because another contributor submitted an upstream PR for the Omi Trash feature, I needed a new issue that was open, well-scoped, and technically clear. [lance-format/lance#7313](https://github.com/lance-format/lance/issues/7313) fits that profile: it is a concrete Rust bug in the FTS (full-text search) inverted index builder that panics during `table.optimize()` when position tracking is enabled (`with_position: true`, the default). The issue includes the exact panic message, production table examples, the file and function name, and a proposed fix direction. Lance is the core columnar format behind LanceDB, so fixing a crash during index optimization has real user impact. The fix is bounded to one function in `rust/lance-index`, which is a manageable scope for my first Rust OSS contribution.

---

## Understanding the Issue

### Problem Description

`IndexWorker::process_batch()` in `rust/lance-index/src/scalar/inverted/builder.rs` panics with `index out of bounds` when rebuilding an FTS inverted index with `with_position: true`. This was observed during `table.optimize()` / `optimize_indices()` on production LanceDB tables.

### Expected Behavior

When `tokens.add()` returns a `token_id`, the code should ensure `posting_lists` has a valid entry at that index **before** accessing `posting_lists[token_id]`. Indexing and optimization should complete without panicking.

### Current Behavior

In the `with_position` branch of `process_batch`, `posting_lists` is grown only when `token_id == posting_lists.len()` (exact equality). When `token_id > posting_lists.len()` — which happens when the token vocabulary and posting-list vector are out of sync (e.g. stale `next_id` from a legacy index partition) — the code skips growth and panics:

```
thread 'tokio-rt-worker' panicked at .../builder.rs:1344:66: index out of bounds: the len is 1731 but the index is 4456
```

### Affected Components

- **Primary:** `IndexWorker::process_batch()` — `with_position` token loop (~lines 1320–1353)
- **Secondary access:** `finish_open_doc` loop (~line 1405) — safe once the first loop resizes correctly
- **Not affected:** Non-position branch (lines 1375–1384) already uses `resize_with(tokens.len(), ...)`
- **Related prior fixes:** `merge_from` (line 979) and stale-`next_id` tests (lines 3890–3962) fixed similar mismatches on a different code path

---

## Reproduction Process

### Environment Setup

I forked [lance-format/lance](https://github.com/lance-format/lance) to [Siriapps/lance](https://github.com/Siriapps/lance) and cloned into `c:\Users\siria\OneDrive\Documents\code\lance` on Windows.

**Blockers encountered and resolved:**

1. **`protoc` / proto path failure** — Git symlink stubs at `rust/*/protos` (pointing to `../../protos/`) do not resolve on Windows; `cargo check` failed with `Could not make proto path relative: ./protos/encodings_v2_0.proto`.
   - **Fix:** Created Windows directory junctions from each `rust/*/protos` to the repo-root `protos/` folder. Re-run `git restore rust/*/protos` after builds to keep the working tree clean; recreate junctions before the next `cargo` command.
2. **First `cargo check` compile time** — Full workspace compile on Windows took ~6 minutes (expected for first build).

**Current status:** `cargo check -p lance-index` succeeds. Working branch: [`fix-issue-7313`](https://github.com/Siriapps/lance/tree/fix-issue-7313). Reproduction test committed as `240045ec`.

### Steps to Reproduce

1. Fork [lance-format/lance](https://github.com/lance-format/lance) and clone your fork.
2. On Windows, create proto junctions (see Environment Setup above).
3. Create branch: `git checkout -b fix-issue-7313` and push to your fork.
4. Verify build: `cargo check -p lance-index`
5. Run the reproduction test:

```bash
cargo test -p lance-index test_process_batch_with_position_panics_when_token_id_exceeds_posting_lists_len -- --nocapture
```

6. **Observed result (current main / pre-fix):** Test passes via `#[should_panic]` — worker panics with:

```
index out of bounds: the len is 1731 but the index is 4456
```

7. **How the test simulates production:** It pre-populates `worker.builder` with 1731 tokens, sets stale `tokens.next_id = 4456` (mimicking legacy FTS partitions), sizes `posting_lists` to 1731, then calls `process_batch` with a new unseen token. `tokens.add()` assigns id 4456; the `==` guard does not grow the vector; line 1344 panics.

### Reproduction Evidence

- **Branch link:** https://github.com/Siriapps/lance/tree/fix-issue-7313
- **Commit:** `240045ec` — `test: reproduce FTS posting_lists index panic for issue #7313`
- **Test file:** `rust/lance-index/src/scalar/inverted/builder.rs` — `test_process_batch_with_position_panics_when_token_id_exceeds_posting_lists_len`
- **Panic location:** `builder.rs:1344` — `&mut builder.posting_lists[token_id as usize]`

---

## Solution Approach

### Analysis

**Root cause:** The `with_position` and `!with_position` branches in `process_batch` use different strategies for keeping `posting_lists` in sync with `tokens`:

| Branch | Strategy | Handles `token_id > len`? |
|--------|----------|---------------------------|
| `with_position` (buggy) | Grow only when `token_id == len`, single push | No |
| `!with_position` (correct) | `resize_with(tokens.len(), ...)` after tokenizing | Yes |

**Why `token_id` can exceed `len`:** `TokenSet::add()` returns `next_id` for new tokens. If `next_id` is stale (higher than the number of posting lists built so far) — from legacy index partitions written before #7115, or from a vocabulary/posting-list desync during optimize — the equality check fails and the vector is never grown to the required size.

**Why the issue's suggested fix (`==` → `>=` + one push) is insufficient:** For `len=1731` and `token_id=4456`, one push yields `len=1732` — still out of bounds at index 4456. The correct approach is `resize_with(token_id + 1, ...)` to fill the entire gap.

**Prior art in the same file:**

- `merge_from` line 979: `self.posting_lists.resize_with(self.tokens.len(), ...)`
- `test_merge_from_after_remap_does_not_panic` (line 3853)
- `test_merge_with_stale_next_id_token_file_does_not_panic` (line 3925)

### Proposed Solution

Replace the `==` + single `push` block in the `with_position` token loop with `resize_with(token_idx + 1, ...)`, preserving the existing `memory_size` overhead tracking. No Python/Java binding changes — logic stays in Rust core per Lance AGENTS.md.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** FTS index rebuild panics when `with_position=true` because `posting_lists` is not grown before indexed access when `token_id > posting_lists.len()`.

**Match:** Non-position branch (`resize_with`), `merge_from` (`resize_with`), stale-`next_id` regression tests.

**Plan — Step by step** (single file: `rust/lance-index/src/scalar/inverted/builder.rs`):

**Step 1 — Fix resize guard (~lines 1325–1342)**

Replace:

```rust
if token_id as usize == builder.posting_lists.len() {
    let old_posting_lists_overhead_size = ...;
    builder.posting_lists.push(
        PostingListBuilder::new_with_posting_tail_codec(true, posting_tail_codec),
    );
    let new_posting_lists_overhead_size = ...;
    Self::adjust_tracked_value(memory_size, old, new);
}
```

With:

```rust
let token_idx = token_id as usize;
if token_idx >= builder.posting_lists.len() {
    let old_posting_lists_overhead_size = (builder.posting_lists.capacity()
        * std::mem::size_of::<PostingListBuilder>()) as u64;
    builder.posting_lists.resize_with(token_idx + 1, || {
        PostingListBuilder::new_with_posting_tail_codec(true, posting_tail_codec)
    });
    let new_posting_lists_overhead_size = (builder.posting_lists.capacity()
        * std::mem::size_of::<PostingListBuilder>()) as u64;
    Self::adjust_tracked_value(
        memory_size,
        old_posting_lists_overhead_size,
        new_posting_lists_overhead_size,
    );
}
```

**Step 2 — Flip reproduction test (~lines 3753–3797)**

- Rename: `test_process_batch_with_position_panics_...` → `test_process_batch_with_position_handles_token_id_gaps`
- Remove: `#[should_panic(expected = "index out of bounds")]`
- Add after `process_batch`:

```rust
let new_token_id = worker.builder.tokens.get("unseen_token_xyz").expect("new token indexed");
assert!((new_token_id as usize) < worker.builder.posting_lists.len());
assert_eq!(worker.builder.posting_lists.len(), 4457);
```

**Step 3 — Verify**

```bash
cargo test -p lance-index test_process_batch_with_position_handles_token_id_gaps -- --nocapture
cargo test -p lance-index test_worker_trims_position_temp_buffers
cargo test -p lance-index test_worker_flush_keeps_position_temp_memory_bounded
cargo test -p lance-index
cargo fmt --all
cargo clippy -p lance-index --tests -- -D warnings
```

**Step 4 — Commit and push**

```bash
git commit -m "fix: grow posting_lists before indexed access in FTS with_position builder"
git push origin fix-issue-7313
```

**Implement:** https://github.com/Siriapps/lance/tree/fix-issue-7313 (Phase III commits pending)

**Review:** Before PR — Conventional Commits title (`fix:`), no drive-by changes, tests required per AGENTS.md.

**Evaluate:**

- New regression test passes (no panic)
- Existing position worker tests pass
- Full `cargo test -p lance-index` passes
- `cargo fmt` and `cargo clippy` clean

---

## Testing Strategy

### Unit Tests

- [x] `test_process_batch_with_position_panics_when_token_id_exceeds_posting_lists_len` — reproduces #7313 (Phase II)
- [ ] `test_process_batch_with_position_handles_token_id_gaps` — confirms fix (Phase III)
- [ ] Existing: `test_worker_trims_position_temp_buffers`, `test_worker_flush_keeps_position_temp_memory_bounded`
- [ ] Existing: `test_merge_from_after_remap_does_not_panic`, `test_merge_with_stale_next_id_token_file_does_not_panic`

### Integration Tests

- Not required for this fix — unit test covers the exact panic path; production scenario (`optimize_indices` on legacy FTS tables) is impractical to replicate locally.

### Manual Testing

N/A — Rust unit test is sufficient for this bounded vector-index bug.

---

## Implementation Notes

### Week 2 Progress (Phase II)

- Selected issue #7313 after Contribution 1 stopped.
- Created branch `fix-issue-7313`, pushed to [Siriapps/lance](https://github.com/Siriapps/lance).
- Resolved Windows proto junction blocker.
- Added and ran reproduction test; confirmed exact panic: `the len is 1731 but the index is 4456`.
- Documented fix plan: `resize_with(token_idx + 1)` not `>=` + single push.

### Code Changes (Phase II only)

- **File:** `rust/lance-index/src/scalar/inverted/builder.rs`
- **Commit:** `240045ec` — reproduction test only (no fix yet)

---

## Pull Request

**PR Link:** Not yet submitted — awaiting review before opening upstream PR

**Draft PR title:** `fix: grow posting_lists before indexed access in FTS with_position builder`

**Draft PR body:**

> Fixes #7313  
> Replaces `token_id == posting_lists.len()` + single push with `resize_with(token_id + 1, ...)` in the `with_position` branch of `IndexWorker::process_batch`  
> Adds regression test for stale `next_id` / posting-list length mismatch

**Maintainer Feedback:**

- (none yet)

**Status:** Phase II Complete — fix implementation pending

---

## Learnings & Reflections

### Technical Skills Gained

- Reading Rust FTS index builder code (`IndexWorker`, `TokenSet`, `PostingListBuilder`).
- Writing `#[tokio::test]` reproduction tests with `#[should_panic]`.
- Windows-specific OSS setup (proto symlink → junction workaround).

### Challenges Overcome

- Git symlink stubs not resolving on Windows for `protoc` build.
- Understanding why `>=` + one push is insufficient for large token ID gaps.

### What I'd Do Differently Next Time

- Check for platform-specific build issues (symlinks) earlier in environment setup.
- Verify issue-suggested fix against worst-case production numbers before implementing.

---

## Resources Used

- [lance-format/lance#7313](https://github.com/lance-format/lance/issues/7313)
- [Siriapps/lance — fix-issue-7313](https://github.com/Siriapps/lance/tree/fix-issue-7313)
- Lance `rust/lance-index/src/scalar/inverted/builder.rs` — `process_batch`, `merge_from`, stale-`next_id` tests
- Lance AGENTS.md — testing and PR conventions
