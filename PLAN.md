# VL.OpenCV5 · Plan

Execution plan for [SPECS.md](SPECS.md). SPECS.md says what and why; this file tracks order, ownership and status. Tick boxes as work lands.

Owners: **[CC]** Claude Code alone · **[M]** maintainer · **[CC+M]** Claude prepares, maintainer verifies.

## Status

| Item | State |
|---|---|
| Local repo | Initialized 2026-09-26 from `vvvv/VL.OpenCV` `main` at `75e8b9a`, full history and tags |
| Remotes | `upstream` = vvvv/VL.OpenCV, `origin` = [zeos/VL.OpenCV5](https://github.com/zeos/VL.OpenCV5) (GitHub fork) |
| Branches | `main` tracks `upstream/main` untouched. `opencv5` is the work branch, pushed to `origin` |
| Current phase | Phase 0, then M0 |

## Branching rules

- `main` mirrors upstream and is only used to open fixes back to VL.OpenCV 4.
- `opencv5` is long-lived. Commits go straight onto it and are pushed to `origin`, no PRs inside the fork. Make `opencv5` the fork's default branch on GitHub.
- Scripted `.vl` rewrites are the exception. They go on a local branch and are merged into `opencv5` only after the category was opened in the editor (SPECS §9).
- Tooling that is not part of the package (inventory, rewrite scripts, smoke app) lives in `tools/`.

## Phase 0 · Bootstrap

- [x] Init repo with upstream history, `upstream` remote, `opencv5` branch [CC]
- [x] Commit SPECS.md and PLAN.md [CC]
- [x] Create GitHub fork of `vvvv/VL.OpenCV` as `zeos/VL.OpenCV5` [M], add it as `origin` [CC]
- [x] Rename commit `d840fca` [CC]. Files, assembly (`VL.OpenCV5.dll`), package id, dependency references in all `.vl` documents and help patches. C# namespaces and node categories stay `VL.OpenCV`/`OpenCV` (`RootNamespace` pinned). Left alone: help prose, `.github/CONTRIBUTING.md`, `Changelog.md`, stale hints to `VL.OpenCV.Dev.vl` and `VL.OpenCVSharp.vl`
- [ ] Open `VL.OpenCV5.vl`, `VL.OpenCV5.HDE.vl` and one help patch in vvvv to confirm the renamed references resolve [M]
- [ ] Restructure commit with `git mv` into the SPECS §3 layout (`src/VL.OpenCV5`, `src/VL.OpenCV5.Windows`, `src/Tests`, `docs/`), so blame survives [CC]
- [ ] LICENSE keeps the vvvv notice, add own copyright line for new work [CC]
- [ ] Replace `.github/workflows/main.yml`. It publishes to nuget.org on push to `main` with `VVVV_ORG_NUGET_KEY`, which the fork doesn't have. Superseded by `test.yml` (M0) and `release.yml` (M6) [CC]
- [ ] `.gitignore` for new output paths (`lib/net10.0*/`, test results) [CC]
- [ ] Tell the vvvv group about the port and the naming question [M]

## Fact check (SPECS §10.2)

Verified 2026-09-26

- Upstream `main` HEAD is still `75e8b9a` ✔
- 24 `.cs` files in `src/`, 57 `.vl` help patches ✔
- `src/VL.OpenCV.csproj` as described (net8.0-windows, WinForms, OpenCvSharp4 4.9.0.20240103, CsWin32, VL.CoreLib 2024.6.6) ✔
- nuget.org has `OpenCvSharp5` 5.0.0.20260905 and every runtime package named in SPECS §1.2 and §3, plus `OpenCvSharp5.GdipExtensions`, all at the same version ✔
- Line matches in `VL.OpenCV.vl`: `CvDnn` 17 ✔, `CvAruco` 12 ✔, `InputArray` 66 lines (SPECS says about 30 references), `VideoCapture` 49 lines (SPECS says 23). These are raw line counts. The M1 inventory gives exact numbers
- License mismatch upstream: `LICENSE` is BSD-3-Clause, but the nuspec declares `LGPL-3.0-only`. Resolve before the first release (SPECS assumes BSD-3-Clause)
- `VL.CoreLib` on nuget.org stops at 2025.7.4 (vvvv 7). Version and feed for vvvv 8 are unknown. Upstream `NuGet.config` points at the vvvv TeamCity feed

Remaining

- [ ] Read OpenCvSharp5 `docs/migration-4-to-5.md` and the nuspecs. Confirm the target framework, every row of the SPECS §1.2 breaking-change table, and full vs slim runtime contents [CC]
- [ ] Inspect the `linux-arm64` native library (`ldd`, `Cv2.GetBuildInformation`) for videoio with FFmpeg and V4L2, dnn, and GTK3 dependencies [CC]
- [ ] Find the VL.CoreLib version and feed for vvvv 8 [CC+M]
- [ ] Update SPECS.md where facts differ [CC]

## M0 · Feasibility gate

Deliverables: `tools/M0.Smoke/`, `.github/workflows/test.yml`, `docs/m0-report.md`.

- [ ] `tools/M0.Smoke`, net10.0 console app. Prints `Cv2.GetVersionString()` and build info, reads a small bundled video, runs one dnn and one aruco call, opens camera 0 with `--camera` [CC]
- [ ] RID-conditional runtime package references, so `dotnet publish -r <rid>` carries one native runtime [CC]
- [ ] CI matrix `windows-latest`, `ubuntu-24.04`, `ubuntu-24.04-arm`, `macos-15` (arm64). Build, publish self-contained, run the smoke app, upload logs [CC]
- [ ] Record the apt packages the Linux runtimes need (`libgtk-3-0`, FFmpeg libs) from `ldd` output [CC]
- [ ] `docs/m0-report.md` with a table of RID × loads, version, video file read, videoio backends, dnn, aruco [CC]
- [ ] vvvv 8 ref struct test: a node calling a method with an `InputArray` parameter, and an `InputArray` in node state [M]
- [ ] Minimal vvvv 8 export referencing OpenCvSharp5 on `linux-x64` and `osx-arm64`, check that native runtimes are copied [M]
- [ ] Smoke app with a real camera on each OS [M]

Exit: go or no-go written into `docs/m0-report.md`. If `linux-arm64` lacks videoio, decide on building OpenCvSharpExtern for arm64 in CI.

## M1 · Core project and facade

- [ ] `src/VL.OpenCV5/VL.OpenCV5.csproj`, net10.0, Nullable, warnings as errors, RID-conditional runtimes, VL.CoreLib with `PrivateAssets="all"` [CC]
- [ ] Port neutral files (`CVImage`, `Converters`, `Calibration`, `Enums`, `Utils`, `HoldLatestCopy.Mat`, `UnsupportedMatTypeException`, `VideoSourceToCvImage`, `YOLO*`) into `Imaging/`, `Video/`, `Detection/`, `Calibration/` [CC]
- [ ] `src/VL.OpenCV5.Windows`, net10.0-windows, with Renderer, PictureBoxIpl, DIPHelpers, VideoInInfo and the VideoInput dynamic enums. Compiles now, behaviour checked in M5 [CC]
- [ ] CI guard that fails if the core project references `System.Windows.Forms`, `System.Drawing.Common` or `Windows.Win32` [CC]
- [ ] `tools/ApiInventory`. Parses `VL.OpenCV.vl`, extracts OpenCvSharp members, reflects over OpenCvSharp4 4.9 and OpenCvSharp5, classifies each as unchanged, renamed, removed or ref-struct-affected. Writes `docs/api-inventory.md` and a CSV for the M2 scripts [CC]
- [ ] The same tool lists nodes using `VL.CoreLib.Windows`, `System.Drawing` and `System.Windows.Forms`, which move to the Windows package [CC]
- [ ] Review the inventory [M]
- [ ] Facade, first one category end to end (Filter), then review signatures before covering the rest [CC+M]
- [ ] Facade for the full inventory: `Filter`, `Color`, `Geometry`, `Drawing`, `Arithmetic`, `Features`, `Calibration`, `Tracking`, `IO`, with XML summaries and caller-owned `dst` [CC]
- [ ] `src/Tests/VL.OpenCV5.Tests` (xUnit). Reference image per facade method, CvImage depth and channel matrix, rejection of new depth types, video file read, zero-allocation test over 1000 calls [CC]

Exit: core builds without Windows references, inventory reviewed, tests green on all four CI legs.

## M2 · Patch migration

- [ ] `tools/PatchRewrite`, idempotent. Swaps dependencies (OpenCvSharp4 packages to OpenCvSharp5, adds the facade), `CvDnn` to `Cv2.Dnn`, `CvAruco` to `Cv2.Aruco`, `InputArray.FromMat` patterns to facade calls. Writes a change log per category [CC]
- [ ] After each rewrite, check XML well-formedness and diff node counts against the source [CC]
- [ ] `docs/m2-checklist.md`, one branch per category in this order: Basics and IO, Filter, Color, Drawing, Geometry, Arithmetic, Tracking, Calibration, Detection [CC]
- [ ] Open each category in the vvvv 8 editor, fix what the script missed, hand error lists back [M]
- [ ] Fixes from error lists [CC]

Exit: all nodes compile without red in vvvv 8 on Windows.

## M3 · Cross-platform video

- [ ] `VideoIn` with a capture thread per device and a triple buffer of preallocated `Mat`s, lock-free handoff to the mainloop [CC]
- [ ] Explicit backend per OS, `-1` property readouts reported as unknown [CC]
- [ ] Enumeration: Linux via sysfs, macOS via AVFoundation and `objc_msgSend`, fallback index probe 0 to 9. Refresh on demand and on open failure [CC]
- [ ] Tests for the triple buffer under concurrency and for the sysfs parser against fixture trees [CC]
- [ ] Cameras on Windows, Raspberry Pi 5 and macOS arm64 in headless exports, 640 × 480 at 30 fps or more [M]

## M4 · Detection

- [ ] Stateful Aruco detector node, recreated only when its parameters change [CC]
- [ ] ONNX YOLO node (v8+ output layout): letterbox preprocessing, decode, NMS, `YOLODescriptor` output. Remove Darknet and Caffe paths [CC]
- [ ] Choose the YOLO model. Ultralytics YOLOv8 and later are AGPL-3.0, which matters for a bundled test model and for users. Apache-2.0 alternatives exist (YOLOX, RT-DETR variants) [M]
- [ ] Haar cascade test, remove `TrackerGOTURN` [CC]
- [ ] Results on real scenes [M]

## M5 · Windows package

- [ ] Renderer on `OpenCvSharp5.GdipExtensions`, Media Foundation and DirectShow enumeration overriding the core one, `VL.OpenCV5.Windows.vl` and `VL.OpenCV5.HDE.vl` [CC]
- [ ] Renderer and HDE checked in the editor [M]

## M6 · Help, migration guide, release

- [ ] `MIGRATION.md` (renamed, removed and behaviour-changed nodes, no coexistence with VL.OpenCV) and README (platforms, GTK3, macOS camera permission, FFmpeg LGPL) [CC]
- [ ] `deployment/VL.OpenCV5.nuspec`, `deployment/VL.OpenCV5.Windows.nuspec`, `.github/workflows/release.yml` [CC]
- [ ] Port and check the 57 help patches, rebaseline pixel-exact examples [M]
- [ ] Export tests on every target RID [M]

## Next actions

1. [M] Set `opencv5` as the fork's default branch, check the rename in the vvvv editor.
2. [CC] Restructure commit.
3. [CC] Finish the fact check and update SPECS.md.
4. [CC] M0 smoke app and CI matrix.
5. [CC] API inventory script and `docs/api-inventory.md`.

## Open decisions

- Package id `VL.OpenCV5`, or `VL.OpenCV` 4.0 upstream. Waiting on the vvvv group.
- VL.CoreLib version and feed for vvvv 8.
- Linux x64 highgui: document the GTK3 dependency, or build a runtime without highgui.
- `osx-x64` support, optional in SPECS. Decide before release.
- YOLO model and its licence.
- Package license, BSD-3-Clause (LICENSE file) or LGPL-3.0-only (upstream nuspec).
