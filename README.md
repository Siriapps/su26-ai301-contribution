# su26-ai301-contribution
This is my contribution log for Codepath AI 301 course

# Contribution 1: Feature: Trash bin / soft delete for conversations, memories, and tasks

**Contribution Number:** 1  
**Student:** Lakshmi Siri Appalaneni  
**Issue:** https://github.com/BasedHardware/omi/issues/5092  
**Status:** Phase I Complete

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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

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

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

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

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
