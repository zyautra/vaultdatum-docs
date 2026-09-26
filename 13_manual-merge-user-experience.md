# Manual Merge User Experience

> Status: design for `0.3.1`. This document replaces only the Manual Merge
> workspace experience; conflict detection, durable resolution, and Server
> commit rules remain defined by [05 Conflict Resolution](./05_conflict-resolution.md).

## 1. Goal

The `0.3.0` Manual Merge modal safely preserves both versions and the user’s
merged result, but it presents three plain text areas in sequence. It is safe,
not easy to review.

`0.3.1` provides a clear workspace in which a user can answer three questions
without interpreting implementation details:

```text
What did the Server version contain?

What did this device contain?

What content will be saved as the new result?
```

The workspace does not perform an automatic three-way merge and does not alter
the synchronization protocol.

## 2. Scope and non-goals

In scope:

```text
Clear Server / This device / Merged result hierarchy

Responsive Desktop and Mobile layouts

Explicit actions for choosing the initial result content

Read-only source views and an editable result view

Save progress and failure feedback
```

Out of scope:

```text
Automatic merge or last-write-wins behavior

IDE-grade inline conflict markers or a syntax-aware merge engine

Protocol, Server API, or persistence-format changes

Rendering note previews, embeds, or executable content inside the merge UI
```

## 3. Workspace layout

The modal title is `Resolve conflict` and shows the affected path beneath it.
It explains that the Server version remains authoritative until the merged
result is accepted as a new change.

### 3.1 Desktop

Desktop shows the two immutable sources side by side, followed by a distinct
result section.

```text
Resolve conflict
notes/meeting.md

┌────────────────────────┬────────────────────────┐
│ Server version         │ This device's version  │
│ read-only source       │ read-only source       │
│                        │                        │
│ [Use as result]        │ [Use as result]        │
└────────────────────────┴────────────────────────┘

Merged result
editable text area

[Cancel]                                  [Save merged result]
```

The source panes use equal visual weight, readable line spacing, a bounded
height, and independent scrolling. They must not be editable. The result pane
is visually separated and larger than either source pane.

### 3.2 Mobile

Mobile must not squeeze three text areas into a narrow vertical stack. It shows
one source at a time using accessible tabs or a segmented control:

```text
[Server] [This device] [Merged result]

active source or editable result

[Use Server as result] / [Use this device as result]

[Save merged result]
```

Switching sources never discards unsaved result text. The save action remains
reachable without requiring a desktop-only keyboard shortcut.

## 4. Result initialization and editing

The initial result starts with this device’s version, but the UI says so
explicitly: `Result starts with this device's version.` A user may replace it
with the Server or device source using an explicit `Use as result` action.

Replacing the result requires confirmation only when the result has been edited
since its last source selection. The confirmation makes clear that it replaces
only the unsaved result text, not either source or the Server Vault.

The editor is plain text. It must preserve line endings and content exactly as
entered by the user. No Markdown preview is shown in this workflow, because a
preview could obscure source text or execute unwanted rendering behavior.

## 5. Save lifecycle and safety

`Save merged result` means: record the user’s result durably, apply it to the
local replica, and queue a normal base-validated change. It does not claim that
the Server has accepted the change yet.

While saving:

```text
Save merged result → Saving… (disabled)
```

On success, show that the merged result is queued and return to the remaining
conflict list when the workspace was opened from Conflict Center. On failure,
keep the modal and the edited result open. Explain that neither source version
was discarded.

The existing durable manual-merge artifact remains the source of restart
recovery. Closing the modal before saving does not resolve the conflict and
does not alter either source.

## 6. Accessibility and privacy

- Source labels identify `Server version` and `This device's version` in text;
  color alone is never the distinction.
- Tabs, source actions, editor, and save/cancel controls are keyboard and touch
  accessible.
- Long paths wrap or truncate safely with an accessible full value.
- The workspace shows note content only because the user explicitly opened a
  specific conflict. It never copies content into notices, logs, or diagnostic
  output.

## 7. Acceptance scenarios

| Scenario                              | Expected result                                                                                     |
| ------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Desktop Markdown conflict             | Server and device sources are comparable side by side; the result is visibly editable and distinct. |
| Mobile Markdown conflict              | A user can inspect both sources, edit the result, and save without horizontal overflow.             |
| Select Server as result after editing | The user receives an unsaved-result replacement confirmation; neither source is changed.            |
| Save succeeds                         | The result is durably queued, not falsely reported as already accepted by the Server.               |
| Save fails or the app stops           | The durable resolution behavior from 0.3.0 preserves the user-created result for recovery.          |
| Non-Markdown or binary conflict       | Manual Merge remains unavailable; only state-appropriate non-text resolution actions are offered.   |
