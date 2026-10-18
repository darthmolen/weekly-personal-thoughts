---
kind: plan
status: queued
track: main
date: 2026-10-18
---

# Personal Thoughts — .NET MAUI Android App (MVP)

**Author:** Claude (planned with `plan-workflow` + `csharp-quality-developer` from `darthmolen/ai-plugins-and-skills`)

## Objective

Ship an Android .NET MAUI app that captures one-line personal thoughts (spoken or typed), shows them on a
week-at-a-time timeline that rolls over to the new day and week automatically, lets each thought open into a
details view where text, pictures, and audio clips can be added, and exports thoughts as plain text.

## Requirements (as stated)

1. **Thoughts are one-liners.** Tapping one opens its details.
2. **Front page = timeline by day/date** for one week, which **turns over automatically** (new day and new week
   show up without a restart).
3. **Header reads "Week of …"** with the week's start and end dates.
4. **Previous button** goes back one week at a time.
5. **Two add buttons:** an **audio icon** (speech-to-text) and a **pencil** (manual typing).
6. **Once a thought is created**, tapping it opens a **details view** with:
   - **pencil**: edit the details text by typing;
   - **wav icon**: dictate into the details (speech-to-text);
   - **paper clip** (a slightly different glyph, so it isn't mistaken for the add buttons): opens a menu with
     - **picture icon**: embed a picture;
     - **wav clip icon**: embed an audio clip.
7. **More than one thought per day** is allowed.
8. **New thoughts can only be added for today.**
9. **Export**: for now, only text is exported. *(Your note said "for map"; I read it as "for now"/MVP. Please
   correct me if you meant something else.)*

## Assumptions and open questions

These are my defaults. Each one is a small change if you'd like it different.

| # | Question | Default in this plan |
|---|---|---|
| Q1 | Which day does the week start on? | **Sunday.** `WeekCalculator` takes `firstDayOfWeek`, so switching to Monday is a single setting (see backlog stub). |
| Q2 | Can thoughts from earlier days still be edited (details, attachments)? | **Yes.** Only *creating* a thought is limited to today, and past thoughts stay editable. |
| Q3 | You asked for a Previous button. How do you get back to this week? | Add a **Next** button (disabled on the current week) and a **Today** shortcut on the header. Without them you can't return. |
| Q4 | What is the one-liner vs. the details? | `Summary` (one line, max 140 chars, newlines removed) is what the timeline shows. `Details` is free-form text in the details view. |
| Q5 | Which days does the timeline list? | The current week shows Sunday through **today**, newest first. A past week shows all 7 days. Empty days show a small "No thoughts" row so the dates stay continuous. |
| Q6 | Audio clip: record or pick a file? | **Record in-app** first. Picking an existing audio file is a second menu entry in the same flow. |
| Q7 | Export format and scope? | A **plain-text (`.txt`) file** for the **week on screen** or **all thoughts**, handed to the Android share sheet. Attachments appear as text placeholders (`[picture attached]`, `[audio clip 0:42]`). |
| Q8 | Sync or backup to the cloud? | **No.** Everything stays on the device for the MVP (see backlog stub). |

## Success Criteria

- [ ] `dotnet build -f net10.0-android` succeeds with **zero warnings** (warnings are errors, StyleCop is on).
- [ ] `dotnet test` passes for `PersonalThoughts.Core.Tests` and `PersonalThoughts.Infrastructure.Tests`.
- [ ] The app installs and runs on an Android emulator (API 34+) and on a physical phone (min API 26).
- [ ] The header shows `Week of Oct 18 – Oct 24, 2026`, built from a computed week range.
- [ ] If the app is left open across midnight, today's group appears without a restart. Crossing Saturday→Sunday
      moves to the new week. Resuming the app after it was backgrounded overnight does the same.
- [ ] Previous/Next/Today navigate between weeks. Next is disabled on the current week.
- [ ] The mic button creates a thought from speech-to-text. The pencil button creates a typed thought. Both are
      stamped with today's date and are hidden when viewing a past week.
- [ ] More than one thought per day is supported and shown in creation-time order.
- [ ] Tapping a thought opens the details view. Pencil edits the details text, wav dictates into the details,
      and the paper clip opens a menu to attach a picture or record/pick an audio clip. Attachments show inline:
      picture thumbnails, plus audio rows with play/pause and duration.
- [ ] Export produces a UTF-8 `.txt` file with day headings, times, summaries, and details, and opens the share sheet.
- [ ] Every `.cs` file follows `csharp-quality-developer` (see *Coding Standards* below).

## Approach

### Tech stack

| Concern | Choice | Why |
|---|---|---|
| Framework | **.NET 10 MAUI**, single target `net10.0-android` | Current LTS. Android is the only target the user asked for. |
| MVVM | `CommunityToolkit.Mvvm` (source-generated `[ObservableProperty]`, `[RelayCommand]`) | Less boilerplate, testable view models. |
| UI helpers | `CommunityToolkit.Maui` | `SpeechToText` (on-device Android recognizer), `FileSaver`, `Popup` (attach menu). |
| Audio record/playback | `Plugin.Maui.Audio` | Records to a file (AAC/`.m4a` on Android) and plays it back. |
| Pictures | MAUI Essentials `MediaPicker` (`PickPhotoAsync` / `CapturePhotoAsync`) | Uses the Android 13+ photo picker, so no storage permission is needed. |
| Persistence | `sqlite-net-pcl` + `SQLitePCLRaw.bundle_green` | Small local DB. Media files live in `FileSystem.AppDataDirectory/media/` and the DB stores relative paths. |
| Time | `System.TimeProvider` everywhere | Makes midnight and week rollover unit-testable with `FakeTimeProvider`. |
| Icons | Material Symbols Rounded font (Apache-2.0), embedded via `ConfigureFonts` | Provides `mic`, `edit`, `graphic_eq` (wav), `attach_file_add` (the "slightly different" clip), `image`, `audio_file`, `chevron_left/right`, `ios_share`. |
| Tests | xUnit + `Microsoft.Extensions.TimeProvider.Testing` | Covers core logic and the repository off-device. |

### Solution layout

```
weekly-personal-thoughts/
├── PersonalThoughts.slnx
├── Directory.Build.props          # Nullable, TreatWarningsAsErrors, StyleCop, AnalysisLevel=latest-recommended
├── Directory.Packages.props       # Central package versions
├── .editorconfig                  # CRLF for *.cs, 4-space indent, IDE/SA severities
├── .gitattributes                 # *.cs eol=crlf, *.sh eol=lf
├── stylecop.json                  # Header text, using-directive placement
├── src/
│   ├── PersonalThoughts.Core/             (net10.0 — no MAUI references)
│   ├── PersonalThoughts.Infrastructure/   (net10.0 — SQLite + file store)
│   └── PersonalThoughts.App/              (net10.0-android — MAUI UI)
└── tests/
    ├── PersonalThoughts.Core.Tests/
    └── PersonalThoughts.Infrastructure.Tests/
```

Core and Infrastructure contain no MAUI code, so they build and test on any CI runner. Device features (speech,
mic, photo picker, share sheet) sit behind interfaces declared in Core and implemented in App.

### Domain model (Core)

```
Thought
  Id            Guid
  Day           DateOnly          // local calendar day at creation; never changes
  CreatedAt     DateTimeOffset    // local time with offset
  UpdatedAt     DateTimeOffset
  Summary       string            // the one-liner, trimmed, single line, ≤ 140 chars
  Details       string            // free text, may be empty
  Origin        ThoughtOrigin     // Typed | Spoken

Attachment
  Id            Guid
  ThoughtId     Guid
  Kind          AttachmentKind    // Picture | AudioClip
  RelativePath  string            // under AppDataDirectory/media/
  CreatedAt     DateTimeOffset
  Duration      TimeSpan?         // audio only
  SortOrder     int

WeekRange(DateOnly Start, DateOnly End)   // End = Start + 6
DayGroup(DateOnly Day, IReadOnlyList<Thought> Thoughts)
```

Key Core services:

- `WeekCalculator.GetWeek(DateOnly day, DayOfWeek firstDay) → WeekRange`, plus `Previous`/`Next`.
- `IDayClock` (wraps `TimeProvider`): `Today`, `Now`, `TimeUntilNextMidnight()`, and a `DayChanged` event.
  It re-arms a one-shot timer for the next local midnight and also re-checks on app resume, because Android
  timers do not fire while the app is asleep.
- `ThoughtService`:
  - `AddAsync(summary, origin)` always stamps `clock.Today`, so a thought cannot be created for any other day.
  - `UpdateDetailsAsync`, `AddAttachmentAsync`, `RemoveAttachmentAsync`, `GetWeekAsync(WeekRange) → IReadOnlyList<DayGroup>`.
  - `SummaryNormalizer` collapses whitespace and newlines and truncates to 140 chars.
- `ITextExporter` / `PlainTextExporter`: `Export(IEnumerable<DayGroup>) → string`.
- Ports implemented in App: `ISpeechToTextService`, `IAudioClipRecorder`, `IPicturePicker`, `IExportSink`.

### Screens (App)

1. **TimelinePage** (`TimelineViewModel`)
   - Header: `‹` Previous · **"Week of Oct 18 – Oct 24, 2026"** · `›` Next (disabled on the current week). Tapping
     the title jumps to Today.
   - Body: a `CollectionView` grouped by `DayGroup`. Group header: `Sunday, Oct 18`, with a "Today" badge.
     Items: the time and the one-line summary (`LineBreakMode=TailTruncation`, `MaxLines=1`). Tap → details.
   - Bottom bar, visible only on the current week: **mic** button (speech-to-text) and **pencil** button (typed).
     - Mic: request `RECORD_AUDIO` → listen with a partial-results overlay → on final result, `AddAsync(text, Spoken)`
       → navigate to details.
     - Pencil: a single-line entry sheet → `AddAsync(text, Typed)` → navigate to details.
   - Toolbar overflow: **Export…** (This week / All thoughts).
   - Subscribes to `IDayClock.DayChanged`. If you were looking at the current week, it moves to the new current week
     and reloads. If you were browsing a past week, it stays there.
2. **ThoughtDetailsPage** (`ThoughtDetailsViewModel`, Shell route `thought?id=`)
   - Shows the summary (editable on tap), the date/time, the details text, and attachments.
   - Action row: **pencil** (focuses the `Editor` for typing) · **wav / `graphic_eq`** (dictates and appends to
     Details) · **paper clip / `attach_file_add`** (opens `AttachMenuPopup`).
   - `AttachMenuPopup`: **picture icon** (pick or take a photo, copy it into `media/`, add an attachment) and
     **wav clip icon** (record a clip with a timer and stop button, or pick an audio file, then add an attachment).
   - Attachments: picture thumbnails (tap to view full screen) and audio rows (play/pause, duration). Long-press
     to remove.
   - Details text autosaves when focus leaves the editor and when you navigate away.

### Android specifics

- `AndroidManifest.xml`: `RECORD_AUDIO`. `<queries>` for `android.speech.RecognitionService`, so speech works on
  Android 11+ package visibility. `CAMERA` only if "take photo" is kept.
- Min SDK 26, target the latest SDK that .NET 10 supports.
- `App.Window.Resumed` → `IDayClock.Recheck()`, so rollover still happens after the phone has slept.

## Coding Standards (from `csharp-quality-developer`)

These are applied from the first file, enforced by analyzers, and checked before every commit:

- **CRLF** line endings for `.cs` (set in `.gitattributes` and `.editorconfig`), exactly one trailing newline,
  no trailing whitespace, and 4-space indentation.
- File-scoped namespaces. Usings ordered System → Microsoft → third-party → project. One type per file, and the
  file name matches the type (SA1402/SA1649).
- `this.` qualification (SA1101). No `_` field prefix (SA1309). `readonly` where possible. `sealed` by default.
- XML docs on every public/internal member. Docs end with a period. Bool properties use the "Gets or sets a value
  indicating whether…" wording.
- Brace rules SA1500/SA1505/SA1513. Multi-line arguments start on the next line (SA1116).
- Logging uses `[LoggerMessage]` partial methods for repeated paths (repository and audio).
- `CA1812` suppressions for DI-only types go in `GlobalSuppressions.cs`, never inline.
  `InternalsVisibleTo` is used for test projects.
- Do **not** run `dotnet format`. Fix violations by hand. After three fix iterations, stop and report the
  remaining rules.
- `ConfigureAwait(false)` in Core and Infrastructure. Not in view models, which must resume on the UI thread.
- Copyright header on every file: `// Copyright (c) 2026, Steven Molen. All rights reserved.` /
  `// Licensed under the BSD 2-Clause License.` (matches `LICENSE`).
- The generated MAUI template files (`Platforms/**`, `MauiProgram.cs`, `App.xaml.cs`) are brought up to the same
  standard, not excluded.

## Phases

Each phase ends with a green build and tests, then a commit. Tests are written before the code they cover.

### Phase 0 — Repository scaffolding

- `.gitignore` (VisualStudio + MAUI), `.gitattributes`, `.editorconfig`, `Directory.Build.props`,
  `Directory.Packages.props`, `stylecop.json`, `PersonalThoughts.slnx`.
- Empty Core, Infrastructure, App, and test projects wired into the solution. `dotnet build` is clean.
- [ASYNC] GitHub Actions workflow `.github/workflows/ci.yml`: `ubuntu-latest`, `dotnet workload install maui-android`,
  build all, test Core and Infrastructure, upload the debug APK as an artifact.

### Phase 1 — Core domain and time

- Tests first: `WeekCalculatorTests` (Sunday start, Monday start, year boundary, DST weeks),
  `DayClockTests` (fires at local midnight with `FakeTimeProvider`, re-arms, `Recheck` after a jump),
  `SummaryNormalizerTests`, `ThoughtServiceTests` (always stamps today, multiple per day, ordering, attachments),
  `PlainTextExporterTests` (exact snapshot of the output format).
- Implement the models, `WeekCalculator`, `DayClock`, `ThoughtService`, `SummaryNormalizer`, `PlainTextExporter`,
  and the port interfaces.

### Phase 2 — Infrastructure (persistence)

- `SqliteThoughtRepository` (tables `Thoughts`, `Attachments`, index on `Day`). Days are stored as ISO
  `yyyy-MM-dd` text. Times are stored as UTC ticks plus the offset in minutes.
- `MediaFileStore` copies incoming picture/audio streams into `media/{thoughtId}/{attachmentId}.{ext}` and deletes
  them when an attachment is removed.
- Tests run against a temp-file SQLite DB and a temp directory: round-trip, week query, cascade delete of attachments.

### Phase 3 — App shell and timeline

- `MauiProgram`: DI registrations, fonts, CommunityToolkit, and the audio plugin.
- `AppShell` with a `thought` route.
- `TimelinePage` and `TimelineViewModel`: the week header, Previous/Next/Today, the grouped list, the empty-day
  rows, and the `DayChanged` subscription.
- The mic and pencil add buttons (the pencil flow first, the mic flow in Phase 4). Both are hidden on past weeks.
- View-model tests for header text, Next being disabled, buttons hidden on past weeks, and rollover moving
  to the new week. These live in Core.Tests by keeping view-model logic in a plain `TimelineState` class in Core.

### Phase 4 — Speech-to-text

- `CommunitySpeechToTextService : ISpeechToTextService` (`SpeechToText.Default`, the device's culture, partial
  results).
- Permission request flow and a listening overlay with cancel. Graceful message if no recognizer is installed.
- Hooked up to the timeline mic button (create a thought) and the details wav button (append to Details).

### Phase 5 — Details view and attachments

- `ThoughtDetailsPage` and view model: editable summary and details with autosave.
- The `AttachMenuPopup` with picture and wav-clip options.
- `MediaPickerPicturePicker : IPicturePicker` and `PluginAudioClipRecorder : IAudioClipRecorder` (record UI with an
  elapsed timer, stop, save, and duration).
- Attachment list: thumbnails, full-screen viewer, audio play/pause, long-press to remove.

### Phase 6 — Export (text only)

- Toolbar "Export…" → choose This week / All → `PlainTextExporter` →
  `personal-thoughts_{start}_{end}.txt` → `Share.RequestAsync(new ShareFileRequest …)`, with `FileSaver` as a
  "Save to device" option.
- Output format, locked by a snapshot test:

  ```
  Personal Thoughts — Week of Oct 18 – Oct 24, 2026

  Sunday, Oct 18, 2026
    08:14  Call mom about the weekend
           Details: She mentioned the cabin trip…
           [picture attached] [audio clip 0:42]
    21:02  Finish the garden plan
  ```

### Phase 7 — Hardening and release

- Test on a device: speech offline and online, permission denial paths, rotation, a long week (50+ thoughts),
  and rollover with the device clock moved forward.
- App icon and splash screen. `ApplicationId` `com.stevenmolen.personalthoughts`. Release signing notes go in a
  reminder (a person has to create the keystore).
- Update the README with build, sideload, and feature instructions.

## Dependencies / Prerequisites

- .NET 10 SDK with the `maui-android` workload. Android SDK/emulator (Visual Studio 2026, or the CLI with
  `android` command-line tools).
- A physical Android phone for the speech and microphone checks. The emulator's speech recognizer is unreliable.
- No dependency on any other plan. This is the first plan on the board.

## Files Expected to Change

- `.gitignore`, `.gitattributes`, `.editorconfig` — repository hygiene; CRLF for C#
- `Directory.Build.props`, `Directory.Packages.props`, `stylecop.json` — analyzers and central package versions
- `PersonalThoughts.slnx` — solution
- `.github/workflows/ci.yml` — build, test, and APK artifact
- `src/PersonalThoughts.Core/**` — models, `WeekCalculator`, `DayClock`, `ThoughtService`, `SummaryNormalizer`,
  `PlainTextExporter`, `TimelineState`, port interfaces, `GlobalSuppressions.cs`
- `src/PersonalThoughts.Infrastructure/**` — `SqliteThoughtRepository`, `MediaFileStore`, records
- `src/PersonalThoughts.App/**` — `MauiProgram.cs`, `App.xaml(.cs)`, `AppShell.xaml(.cs)`, `Views/TimelinePage`,
  `Views/ThoughtDetailsPage`, `Views/AttachMenuPopup`, `ViewModels/*`, `Services/*` (speech, audio, picker,
  export), `Resources/Fonts/MaterialSymbolsRounded.ttf`, `Platforms/Android/AndroidManifest.xml`
- `tests/PersonalThoughts.Core.Tests/**`, `tests/PersonalThoughts.Infrastructure.Tests/**`
- `README.md` — build and usage

## Anticipated Backlog

These are filed as stubs in `planning/backlog/`:

- `media-export_2026-10-18` — export pictures and audio (zip or Markdown bundle) when text-only is no longer enough.
- `week-start-setting_2026-10-18` — a user setting for the first day of the week (Q1).
- `backup-and-sync_2026-10-18` — off-device backup or restore (Q8).
- `release-signing-and-store_2026-10-18` — keystore, signed AAB, and Play Store listing.
