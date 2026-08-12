---
title: "AvalonDock v5 Is Released: The Numbers Behind It"
author: "sharpSteff"
pubDatetime: 2026-08-12T08:00:00Z
slug: avalondock-v5-released
featured: true
draft: false
tags:
  - open-source
  - dotnet
  - wpf
  - avalondock
  - oss
description: >
  AvalonDock v5.0.0 ships today. A look back at the numbers: 109 days, 223 commits, 69 merged
  pull requests, 7 to 14 NuGet packages, and a test suite that grew from 114 to 527 tests.
  Plus what comes next with AvalonDock on LibreWPF.
---

Back in June I wrote about [what AvalonDock v5 was changing under the hood](/posts/avalondock-v5) while it was still in early pre-alpha. Today it's not pre-alpha anymore: **AvalonDock v5.0.0 is released**.

This post is the other half of that story. Less "here's the architecture", more "here's what it actually took": the numbers, what landed after that first article, and where the project goes from here.

## Table of contents

## The release in numbers

Everything below is measured between the last v4 release, `v4.74.1` (25 April 2026), and the v5.0.0 tag.

What it took:

- **109 days** of development
- **223 commits**
- **69 merged pull requests**
- **492 files touched** (242 added, 231 modified, 19 deleted)
- **+60,938 / -9,675 lines**

What changed:

| | v4.74.1 | v5.0.0 |
| :-- | :-- | :-- |
| NuGet packages | 7 | **14** |
| Target frameworks | 6 | **3** |
| Test fixtures | 24 | **77** |
| Test methods | 114 | **527** |
| Test code | 4,554 lines | **16,521 lines** |

A few of those deserve a second look.

**223 commits in 109 days.** The repository has 1,245 commits in total, going back to the initial import from CodePlex in December 2017. That means roughly **18% of AvalonDock's entire recorded history happened in the last three and a half months**. For a library that has been in maintenance mode more often than not since 2007, that's the part I still find slightly hard to believe.

**+60,938 / -9,675 lines.** The additions are not just new features. A large share is test code, documentation, and the new component projects. The deletions are the interesting number in the other direction: nearly ten thousand lines removed from a codebase where almost nothing had been removed in years.

**Seven packages became fourteen.** v4 shipped one docking assembly plus six theme packages. v5 splits the framework into pieces you can take or leave:

```text
Dirkster.AvalonDock                        (WPF docking framework)
Dirkster.AvalonDock.Core                   (UI-agnostic interfaces, models, DTOs)
Dirkster.AvalonDock.Mvvm                   (MVVM base classes, no dependencies)
Dirkster.AvalonDock.Mvvm.CommunityToolkit  (CommunityToolkit.Mvvm integration)
Dirkster.AvalonDock.DependencyInjection    (IServiceCollection extensions)
Dirkster.AvalonDock.Serializer.Xml         (XML layout serialization)
Dirkster.AvalonDock.Serializer.Json        (JSON layout serialization)
+ 7 theme packages
```

**Six target frameworks became three.** The old list was:

```xml
<ComponentTargetFrameworks>
  netcoreapp3.0;net5.0-windows;net9.0-windows;net10.0-windows;net40;net48
</ComponentTargetFrameworks>
```

The new one:

```xml
<ComponentTargetFrameworks>net9.0-windows;net10.0-windows;net48</ComponentTargetFrameworks>
```

Every removed TFM was already end-of-life. `net40` in particular had been dragging conditional compilation and API workarounds through the codebase for over a decade.

**114 tests became 527.** This is the number I care about most, because it's the one that makes everything else sustainable. Of those, **92 are FlaUI-driven UI automation tests** across 19 files: real drag gestures, real floating windows, real layout round trips against a running application. The rest run without a WPF `Application` at all, which is exactly what `AvalonDock.Core` was extracted for: 41 files, 2,623 lines, and **zero** `using System.Windows` statements.

CI runs the two categories separately, which keeps the fast feedback fast:

```yaml
- name: Test (Unit Tests)
  run: >
    dotnet test source/AvalonDock.sln
    --configuration Release
    --filter "Category!=FlaUI"
```

---

## Who built it

223 commits from **nine human contributors**, plus dependabot keeping the dependency tree honest with 15 automated bumps.

I wrote 172 of them, about 83% of the non-bot commits. That says something about how much of v5 was one sustained refactor, and it's also a reminder of why the test suite mattered so much. When one person is moving the load-bearing walls of a twenty-year-old codebase, the tests are the only thing standing between "refactor" and "regression".

The other contributors carried real weight too: floating-window fixes, logical-children ordering, the VS2026 theme parser, navigator interaction fixes, and a steady stream of issue reports precise enough to act on. Several of the bugs closed in this window came with a reproduction project attached, which is the single most useful thing anyone can put in an issue tracker.

---

## What landed after the pre-alpha post

The June article covered `AvalonDock.Core`, dependency injection, MVVM, the `ToggleDockingManager`, DTO-based serialization, the VS2022 theme, and drop-zone geometry in Core. Here's what came after it.

### VS2026 themes

The `.vstheme` infrastructure paid off faster than expected. [PR #597](https://github.com/Dirkster99/AvalonDock/pull/597) added **VS2026 theme support in Blue, Dark and Light**, this time parsed from Visual Studio's newer JSON theme format rather than the XML `.vstheme` files, via a `VsJsonThemeParser` sitting next to the existing `VsThemeParser`.

`Dirkster.AvalonDock.Themes.VS` now ships nine variants from a single package:

```csharp
dockManager.Theme = new VS2015DarkTheme();
dockManager.Theme = new VS2022DarkTheme();
dockManager.Theme = new VS2026DarkTheme();   // new
```

Adding a theme generation ahead of v5's release is the clearest evidence that moving themes from hand-written brush dictionaries to parsed palettes was the right call.

### Real standalone windows

[PR #621](https://github.com/Dirkster99/AvalonDock/pull/621) added the option to undock a document into a **real top-level window** rather than an AvalonDock floating window, with its own taskbar entry and its own window chrome. [PR #637](https://github.com/Dirkster99/AvalonDock/pull/637) then added `DockingManager` switches to turn floating windows and standalone windows off entirely, for applications that want a fixed layout.

### JSON serialization on System.Text.Json

The JSON serializer that shipped with the DTO refactor was moved onto `System.Text.Json` in [PR #624](https://github.com/Dirkster99/AvalonDock/pull/624), with full round-trip coverage. And because the DTO layer is format-agnostic, [PR #607](https://github.com/Dirkster99/AvalonDock/pull/607) could make XML deserialization **tolerant of legacy v4 layout files**, so your users' saved layouts survive the upgrade.

### Keyboard shortcuts for toolboxes

[PR #592](https://github.com/Dirkster99/AvalonDock/pull/592) wired keyboard shortcuts into the `ToggleDockingManager`, so toggling a panel doesn't require the mouse. Small feature, disproportionate daily impact.

![AvalonDock Dark Toolbox](../../assets/images/AvalonDockCodeApp_Dark_Toolbox.png)

### A documentation site

v5 ships with **38 pages of documentation**: getting started, concepts, tutorials for MVVM, DI, styling and layout persistence, theme docs, an API reference, and a migration guide. It's published at [dirkster99.github.io/AvalonDock](https://dirkster99.github.io/AvalonDock/) and built by its own workflow on every push.

### Long-standing bugs, finally closed

Twelve issues were closed in this window, and some of them had been open a long time:

- [#483](https://github.com/Dirkster99/AvalonDock/issues/483): `IconSource` not working when bound from a style. **Opened May 2024.**
- [#494](https://github.com/Dirkster99/AvalonDock/issues/494): docking hints invisible in the Expression themes. **Opened November 2024.**
- [#505](https://github.com/Dirkster99/AvalonDock/issues/505): `CanClose` style setter losing against the model value.
- [#587](https://github.com/Dirkster99/AvalonDock/issues/587): drag overlay windows piling up.

These are the fixes that don't make a release headline but do make a difference in the applications people actually ship.

---

## Where AvalonDock sits today

For context on why any of this matters beyond the repository itself:

- **1.7k stars, 352 forks** on GitHub
- **~1.1 million downloads** of `Dirkster.AvalonDock` on NuGet
- **33 dependent NuGet packages**
- Used by Stride, RoslynPad, DaxStudio, Optick, Macad3D, the SharpDevelop WPF Designer, and Microsoft's Profile Explorer

A docking library is infrastructure. Nobody chooses your application because of it, but plenty of people abandon an application because the layout engine loses their windows. v5 is, more than anything, an attempt to make that infrastructure boring again, in the best sense of the word.

---

## What's next: AvalonDock on LibreWPF

The architectural point of `AvalonDock.Core` was never just faster tests. It was to make it structurally possible for the docking engine to run somewhere other than Windows-only WPF. That work has already started, on the [`librewpf` branch](https://github.com/Dirkster99/AvalonDock/tree/librewpf).

[LibreWPF](https://github.com/wieslawsoltes/wpf) is a cross-platform port of WPF that renders through ProGPU/Silk.NET instead of Win32, which means AvalonDock running on **Linux and macOS**. The branch currently carries about 5,200 lines beyond master, and the shape of the work is already clear: a `Platform` abstraction layer with `ICursorService`, `IDpiService` and `INativeWindowService`, implemented three times over for Windows, Linux and macOS, plus a `PlatformManager` that picks the right one at runtime.

Everything AvalonDock does that is *silently* Win32 has to be found and replaced. The floating-window drag engine is the clearest example: upstream it sends `WM_NCLBUTTONDOWN`/`HT_CAPTION` to enter the OS move loop, drives `DragService.UpdateMouseLocation` from `WM_MOVING`, and drops on `WM_EXITSIZEMOVE`. On a portable backend none of those messages exist, so the branch replaces them with a managed caption drag gated on `!OSPlatform.Windows`: capture the mouse, follow the pointer, feed `DragService`, drop on mouse-up.

Where it stands right now:

| Scenario | Status |
| :-- | :-- |
| Splitter resize drag | **Passing** |
| Tear a tool pane out into a floating window | **Passing** |
| Drag a floating window back onto a drop target | Drag engine ported, blocked in LibreWPF |

The remaining blocker is instructive, because it is *not* an AvalonDock bug. The main window reports coordinates in logical units while floating and overlay windows report theirs in device pixels, so `DropTarget.HitTest` compares a point in one coordinate space against detection rectangles in another and never intersects. The docking engine is internally consistent; the platform underneath is handing different windows different DPI scales. The fix belongs in LibreWPF, and once screen coordinates are uniform across windows, re-docking should pass with no AvalonDock change at all.

Three bugs found along the way have already been fixed *in LibreWPF itself*: a phantom `MouseUp` on window show, a `(0,0)` mouse teleport from `MouseDevice.Synchronize()` during a captured drag, and mouse capture being released by a transient window move. Debugging AvalonDock on a young WPF implementation turns out to be an excellent way to harden that implementation, and there's a full write-up of the investigation in [`docs/librewpf.md`](https://github.com/Dirkster99/AvalonDock/blob/librewpf/docs/librewpf.md) on the branch.

I won't promise a date. Cross-platform docking is a genuinely hard problem, and the honest status is "two of three core drag scenarios work, and we know exactly why the third doesn't". But the question is now about coordinate spaces rather than about whether the architecture permits it at all, and that's the difference v5 made.

---

## A personal note

In the pre-alpha post I wrote that v5 already felt like a milestone. Shipping it feels different: quieter, and somehow larger.

The work was demanding in a way that's hard to convey in a diffstat. Most of those 109 days were spent making careful judgment calls about a codebase older than some of the people who now depend on it: what to break, what to preserve, which twenty-year-old workaround is load-bearing and which one is scar tissue. The numbers at the top of this post are the visible residue of a lot of decisions that could have gone either way.

Thank you to [Dirkster99](https://github.com/Dirkster99) for the trust to make those calls, to everyone who filed an issue with a reproduction project, and to everyone who ran the pre-release packages in production and told us what broke. AvalonDock has survived CodePlex, Xceed, and a decade of maintenance-mode drift. Today it's on modern .NET, modular, tested, documented, and looking at Linux and macOS.

Go update your `PackageReference`.

→ [AvalonDock on GitHub](https://github.com/Dirkster99/AvalonDock)
→ [Documentation](https://dirkster99.github.io/AvalonDock/)
→ [Dirkster.AvalonDock on NuGet](https://www.nuget.org/packages/Dirkster.AvalonDock)
→ [The pre-alpha post: what v5 changes under the hood](/posts/avalondock-v5)
