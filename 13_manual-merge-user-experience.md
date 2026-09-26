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

Color-coded, line-by-line two-way comparison

Explicit per-change actions for choosing Server or This device content

Read-only source views and an editable result view

Save progress and failure feedback
```

Out of scope:

```text
Automatic three-way merge or last-write-wins behavior

Syntax-aware merge, Markdown rendering, or executable content in the diff

Protocol, Server API, or persistence-format changes

Rendering note previews, embeds, or executable content inside the merge UI
```

## 3. Workspace layout

The modal title is `Resolve conflict` and shows the affected path beneath it.
It explains that the Server version remains authoritative until the merged
result is accepted as a new change.

### 3.1 Desktop

Desktop shows a two-way line diff of the immutable sources, followed by a
distinct result section. A changed hunk is a contiguous insert, delete, or
replacement between the two versions; unchanged lines provide context.

```text
Resolve conflict
notes/meeting.md

┌────────────────────────┬────────────────────────┐
│ Server version         │ This device's version  │
│ 12 - removed line      │ 12 + replacement line │
│ 13   unchanged line    │ 13   unchanged line   │
│                        │                        │
│ [Use Server]           │ [Use this device]     │
└────────────────────────┴────────────────────────┘

Merged result
editable text area

[Cancel]                                  [Save merged result]
```

Server-only and device-only lines use distinct colors and `-` / `+` markers;
the labels and markers remain the primary distinction, so the comparison does
not depend on color alone. Each source displays its own line numbers. The
source columns use equal visual weight and readable line spacing. Long source
lines scroll horizontally rather than reflowing into an ambiguous comparison.
They must not be editable. The result pane is visually separated and larger
than either source column.

### 3.2 Mobile

Mobile must not squeeze a two-column diff into a narrow viewport. It shows the
comparison and result through accessible tabs or a segmented control:

```text
[Changes] [Merged result]

Server version
- removed line

This device's version
+ replacement line

[Use Server as result] / [Use this device as result]

[Save merged result]
```

The comparison keeps both labels and all source text visible for each changed
hunk. Switching tabs never discards unsaved result text. The save action
remains reachable without requiring a desktop-only keyboard shortcut.

## 4. Line-by-line decisions

The initial result starts with this device's version. Equivalently, every
changed hunk initially selects `Use this device`; the workspace says so
explicitly: `Result starts with this device's version.`

For every changed hunk, a user can select either:

```text
Use Server

Use this device
```

Selecting a hunk replaces only that hunk in the generated result. It never
changes either immutable source or the Server Vault. A user can select any
combination of hunks, then continue editing the result as plain text.

This is a two-way decision aid, not an automatic merge. There is no hidden base
version, automatic choice, or last-write-wins rule. Where the same line was
changed differently, the user explicitly chooses one side or writes a combined
result.

The client bounds comparison work so a very large, completely divergent note
does not freeze a mobile device. If it cannot safely split that note into line
hunks, the workspace explains the limit and presents one whole-document choice
for each side plus the editable result. It never presents an incomplete line
diff as if it were complete.

When the user edits the result directly, it becomes a custom unsaved result.
Choosing another hunk after that requires confirmation because the result will
be rebuilt from the current hunk choices. The confirmation states that only the
unsaved result text changes; neither source nor the Server changes. Choosing an
entire source as the result follows the same confirmation rule and changes all
hunk choices to that source.

## 5. Result initialization and editing

The editor is plain text. It must preserve line endings and content exactly as
entered by the user. No Markdown preview is shown in this workflow, because a
preview could obscure source text or execute unwanted rendering behavior.

## 6. Save lifecycle and safety

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

## 7. Accessibility and privacy

- Source labels identify `Server version` and `This device's version` in text;
  color alone is never the distinction.
- Changed lines include `+` / `-` markers and source-specific line numbers.
- Tabs, hunk actions, editor, and save/cancel controls are keyboard and touch
  accessible.
- Long paths wrap or truncate safely with an accessible full value.
- The workspace shows note content only because the user explicitly opened a
  specific conflict. It never copies content into notices, logs, or diagnostic
  output.

## 8. Acceptance scenarios

| Scenario                                 | Expected result                                                                                              |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| Desktop Markdown conflict                | Color-coded Server and device lines are comparable side by side; the result is visibly editable and distinct. |
| Mobile Markdown conflict                 | A user can inspect both sides of each changed hunk, edit the result, and save without horizontal overflow.  |
| Select one changed hunk                  | Only that hunk changes in the generated result; neither source is changed.                                  |
| Select a hunk after manual result edits  | The user receives a replacement confirmation; neither source is changed.                                   |
| Select Server as result after editing    | The user receives the same replacement confirmation; neither source is changed.                             |
| Save succeeds                            | The result is durably queued, not falsely reported as already accepted by the Server.                        |
| Save fails or the app stops              | The durable resolution behavior from 0.3.0 preserves the user-created result for recovery.                   |
| Non-Markdown or binary conflict          | Manual Merge remains unavailable; only state-appropriate non-text resolution actions are offered.            |
