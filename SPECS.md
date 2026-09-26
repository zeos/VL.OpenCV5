# VL.OpenCV5 · Specification

A cross-platform computer vision pack for vvvv gamma 8, based on OpenCV 5 through OpenCvSharp5. It is a port of `vvvv/VL.OpenCV` (OpenCV 4.9, Windows only) with a platform-neutral core that runs in exports for Windows, Linux and macOS, and a separate Windows package for everything that needs WinForms, GDI+ or Media Foundation.

## 1. Context and verified facts

### 1.1 Current VL.OpenCV

These facts come from `vvvv/VL.OpenCV` at commit `75e8b9a` (8 Jan 2026).

- `src/VL.OpenCV.csproj` targets `net8.0-windows` with `UseWindowsForms`. It references `OpenCvSharp4.Windows` and `OpenCvSharp4.Extensions` 4.9.0.20240103, `Microsoft.Windows.CsWin32` and `VL.CoreLib` 2024.6.6.
- `VL.OpenCV.vl` (3.9 MB) holds most of the node set, around 230 patched nodes that call OpenCvSharp directly. It depends on `OpenCvSharp4`, `OpenCvSharp4.Extensions`, `OpenCvSharp4.runtime.win`, `VL.CoreLib`, `VL.CoreLib.Windows`, `System.Drawing` and `System.Windows.Forms`.
- `VL.OpenCV.HDE.vl` is an editor extension that depends on `VL.HDE` and `VL.Skia`.
- 24 C# files in `src/`, grouped by platform dependency

| File | Lines | Platform |
|---|---|---|
| `CVImage.cs`, `Converters.cs`, `Calibration.cs`, `Enums.cs`, `Utils.cs`, `HoldLatestCopy.Mat.cs`, `UnsupportedMatTypeException.cs`, `VideoSourceToCvImage.cs`, `YOLO*.cs` | about 1100 | Neutral. They use VL.Core, VL.Lib.Basics.Imaging, VL.Lib.Video and `Stride.Core.Mathematics` (managed, cross-platform) |
| `Renderer.cs`, `Renderer.Designer.cs`, `PictureBoxIpl.cs`, `DIPHelpers.cs` | about 570 | WinForms, System.Drawing, `BitmapConverter` |
| `VideoInInfo.cs`, `Dynamic Enums/VideoInput/*` | about 340 plus enums | Media Foundation and DirectShow through CsWin32, for camera enumeration only |

- Camera capture itself already goes through OpenCV's `VideoCapture` inside the patches (23 references), so capture is portable. Only device enumeration and the preview window are Windows-bound.
- 57 help patches under `help/` (Basics, Batch Processing, Helpers, Topics with Calibration, Detection, Drawing, Filter, Tracking, Utils).
- Content files: Haar cascades, calibration XMLs, fiducials, assets.

### 1.2 OpenCvSharp5 and OpenCV 5

Sources are the OpenCvSharp5 NuGet page and `shimat/opencvsharp/docs/migration-4-to-5.md`.

- OpenCvSharp5 targets `net8.0` and wraps OpenCV 5.0 with opencv_contrib. The namespace stays `OpenCvSharp`.
- Runtime packages include `OpenCvSharp5.runtime.win`, `OpenCvSharp5.runtime.win-arm64`, `OpenCvSharp5.official.runtime.linux-x64`, `OpenCvSharp5.runtime.linux-arm64`, `OpenCvSharp5.runtime.osx.x64`, `OpenCvSharp5.runtime.osx.arm64`. The full Linux x64 package links FFmpeg and uses GTK3 for highgui. The slim variants disable `videoio`, `dnn` and `highgui`, so they are not usable here.
- Breaking changes that hit this pack

| Change | Effect on VL.OpenCV |
|---|---|
| `InputArray`, `OutputArray`, `InputOutputArray` are now `readonly ref struct`s | The patches use `InputArray.FromMat`/`Create` nodes (about 30 references) and call `Cv2` methods whose parameters are these types. VL is not expected to handle ref structs in pins or node state (verify in M0), so these calls must move into C# |
| Fluent `Mat` instance methods removed (`mat.CvtColor(...)`, `mat.Threshold(...)` etc.) | Any patch calling them breaks. Replace with `Cv2.Xxx(src, dst, ...)` behind the C# facade |
| `CvDnn` and `CvAruco` became `Cv2.Dnn` and `Cv2.Aruco` | 17 `CvDnn` and 12 `CvAruco` references in `VL.OpenCV.vl`, plus C# in `YOLO*.cs` |
| Aruco `DetectMarkers` is an instance method on `ArucoDetector` | Marker nodes need a stateful detector |
| Darknet, Caffe and Torch DNN readers removed | The YOLO3 nodes (`ReadNetFromDarknet`) and a Caffe reference stop working. Models must be ONNX |
| `TrackerGOTURN` removed | Remove the node |
| `OpenCvSharp4.Extensions` split, `BitmapConverter` moved to `OpenCvSharp5.GdipExtensions` | Windows package only |
| New depth types `CV_16BF`, `CV_32U`, `CV_64U`, `CV_64S`, `CV_Bool` | Type switches in `Converters.cs` and `CvImage` need default handling |
| `VideoCapture.Get` returns -1 for unsupported properties | Property readouts must treat -1 as unknown |
| Nearest-neighbour resize and warp interpolation changed numerically | Pixel-exact help examples and tests need new baselines |
| `features2d` reshuffle (`BRISK`, `KAZE`, `AKAZE` moved to `OpenCvSharp.XFeatures2D`) | Check the feature detection nodes |

- OpenCvSharp4 and OpenCvSharp5 share the assembly name `OpenCvSharp`. VL.OpenCV and VL.OpenCV5 therefore cannot be loaded in the same patch or export.

## 2. Goals and non-goals

### Goals

- A platform-neutral core package, `VL.OpenCV5`, targeting `net10.0`, usable in vvvv 8 exports for `win-x64`, `win-arm64`, `linux-x64`, `linux-arm64`, `osx-arm64`.
- Keep the existing node names, categories and the `CvImage` concept wherever possible, so users can port patches by swapping the package.
- Cross-platform camera input with device enumeration on each OS.
- Real-time performance with no per-frame managed allocations in the hot path.
- ONNX-based object detection replacing Darknet YOLO3.

### Non-goals

- Running the vvvv editor on Linux or macOS.
- GPU acceleration (OpenCvSharp has no CUDA support).
- Binary compatibility with VL.OpenCV patches that reference removed APIs.
- A native preview window on Linux or macOS. Preview there goes through VL.ImGui.Web (`WebImage`) or through frames saved to disk.

## 3. Package layout

```
VL.OpenCV5/                               repository, forked from vvvv/VL.OpenCV with history
  VL.OpenCV5.vl                           core node set, no Windows dependencies
  VL.OpenCV5.Windows.vl                   Renderer, device enumeration via Media Foundation
  VL.OpenCV5.HDE.vl                       editor extension (Windows, uses VL.Skia)
  help/
  content/                                cascades, calibrations, fiducials, assets, ONNX model notes
  src/
    VL.OpenCV5/                           net10.0
      VL.OpenCV5.csproj
      Facade/                             Cv facade (section 5.1)
      Imaging/                            CvImage, converters, IImage bridge
      Video/                              VideoIn, VideoSourceToCvImage, DeviceEnumeration
        Devices.Linux.cs                  V4L2 via /sys/class/video4linux
        Devices.MacOS.cs                  AVFoundation through the Objective-C runtime
        Devices.Fallback.cs               index-based probing
      Detection/                          Aruco, ONNX YOLO, Haar
      Calibration/
    VL.OpenCV5.Windows/                   net10.0-windows
      Renderer.cs, PictureBoxIpl.cs, DIPHelpers.cs
      VideoInInfo.cs (Media Foundation, DirectShow)
    Tests/
      VL.OpenCV5.Tests/                   xUnit, runs on all three OSes without vvvv
  deployment/
    VL.OpenCV5.nuspec
    VL.OpenCV5.Windows.nuspec
  .github/workflows/
    test.yml                              matrix windows, ubuntu, ubuntu-arm, macos-arm
    release.yml
  README.md
  MIGRATION.md                            guide for users porting patches from VL.OpenCV
```

### Fork setup

- Create the repository as a GitHub fork of `vvvv/VL.OpenCV`, so history, issues context and the upstream link are kept.
- Keep `upstream` as a remote and work on a long-lived `opencv5` branch. Fixes that also apply to VL.OpenCV 4 go back upstream as separate PRs from `main`.
- Rename files, package ids and the `.vl` documents in one early commit (`VL.OpenCV` to `VL.OpenCV5`), so later diffs against upstream stay readable.
- Keep the BSD-3-Clause `LICENSE` with the original vvvv copyright notice, and add your own copyright line for new work.
- If the vvvv group later adopts the port upstream, the fork can be merged back as VL.OpenCV 4.0 without rewriting history.

### NuGet dependencies

`VL.OpenCV5`
- `OpenCvSharp5`
- `OpenCvSharp5.runtime.win`, `OpenCvSharp5.runtime.win-arm64`, `OpenCvSharp5.official.runtime.linux-x64`, `OpenCvSharp5.runtime.linux-arm64`, `OpenCvSharp5.runtime.osx.arm64` (optionally `osx.x64`)
- `VL.CoreLib` as a development dependency (`PrivateAssets="all"`), since vvvv ships it

`VL.OpenCV5.Windows`
- `VL.OpenCV5`, `OpenCvSharp5.GdipExtensions`, `Microsoft.Windows.CsWin32` (build only), `VL.CoreLib.Windows`

## 4. Architecture decisions

1. **A C# facade is the stable node surface.** Patches never touch `InputArray`, `OutputArray`, `InputOutputArray` or `MatExpr`. Every OpenCV call goes through static C# methods that take `Mat`, `CvImage` and plain values. This makes the ref struct change invisible to VL, shields patches from future OpenCvSharp changes, and puts the hot paths into C# where allocations can be controlled.
2. **Patched nodes keep their role for composition.** Node logic that is mostly plumbing (caching, change detection, spreads, CvImage handling) stays in patches but calls the facade.
3. **One core, thin platform parts.** Only device enumeration differs per OS inside the core, selected at runtime with `OperatingSystem.IsLinux()` and friends. WinForms, GDI+ and Media Foundation live only in `VL.OpenCV5.Windows`.
4. **Capture uses OpenCV's `VideoCapture` everywhere**, with an explicit backend per OS (`MSMF` or `DSHOW` on Windows, `V4L2` on Linux, `AVFOUNDATION` on macOS). The VL.Lib.Video bridge (`VideoSourceToCvImage`) stays in the core for sources provided by other packs.
5. **A new package id, `VL.OpenCV5`.** It cannot coexist with VL.OpenCV in one process (same `OpenCvSharp` assembly name), so the README and MIGRATION.md must say so plainly. Coordinate with the vvvv group early, since they may prefer to publish it as VL.OpenCV 4.0 upstream.

## 5. Components

### 5.1 Facade (`src/VL.OpenCV5/Facade`)

- Static classes grouped like the node categories (`Filter`, `Color`, `Geometry`, `Drawing`, `Arithmetic`, `Features`, `Calibration`, `Tracking`, `IO`).
- Signature pattern for image operations
  ```csharp
  public static void GaussianBlur(Mat src, Mat dst, Int2 kernelSize, float sigmaX, float sigmaY, BorderTypes border)
  ```
  The caller owns `dst`. Patched nodes keep one output `Mat` or `CvImage` in their state and reuse it every frame.
- Convenience overloads that return a new `CvImage` exist only for non-realtime use and are marked as allocating in their summary.
- Parameters use VL-friendly types (`Int2`, `Vector2`, `RectangleF` from `Stride.Core.Mathematics`, `Spread<T>`) with conversions in one place (`Converters.cs`).
- Every method has an XML summary, which vvvv shows as node help.
- First pass covers every `Cv2` call found in `VL.OpenCV.vl` (section 1.2 lists some of them, the M1 task produces the full list).

### 5.2 Imaging

- Port `CvImage`, `Converters`, `UnsupportedMatTypeException` and `HoldLatestCopy.Mat` unchanged in behaviour.
- `IImage` bridge to VL.Lib.Basics.Imaging for exchange with VL.Skia, VL.Stride and VL.ImGui.Web without copying where the layout allows.
- Handle the new depth types by mapping them to `UnsupportedMatTypeException` or to the nearest supported format, never by crashing.

### 5.3 Video

- `VideoIn` node. Inputs `Device` (dynamic enum), `Index` (int, for devices without names), `Resolution`, `FPS`, `Backend` (Auto by default), `Enabled`. Output `CvImage` plus `Actual Resolution`, `Actual FPS`, `Is Open`.
- Capture runs on a dedicated thread, one per device, publishing the latest frame into a triple buffer of preallocated `Mat`s. The mainloop reads the newest completed frame without locking or allocating.
- Device enumeration

| OS | Method | Names | Formats |
|---|---|---|---|
| Windows | Media Foundation (existing `VideoInInfo.cs`, moved to the Windows package) and a fallback in the core | yes | yes |
| Linux | `/sys/class/video4linux/video*/name` and `index`, keeping only capture nodes (index 0) | yes | optional, via `VIDIOC_ENUM_FMT` ioctl in a later milestone |
| macOS | `AVCaptureDeviceDiscoverySession` through `objc_msgSend` P/Invoke | yes | optional |
| Fallback | Probe indices 0 to 9 with `VideoCapture.Open` | no | no |

- The dynamic enum refreshes on demand and when a device fails to open, never per frame.
- macOS camera permission is requested by the OS on the first open. The README explains that the permission attaches to the terminal or app that launches the export.

### 5.4 Detection

- Aruco. A stateful `ArucoDetector` node built from dictionary, detector parameters and refine parameters, recreated only when those inputs change.
- Object detection. Replace Darknet YOLO3 with an ONNX YOLO node (YOLOv8 or later output layout, with the class list and confidence and NMS thresholds as inputs). Keep `YOLODescriptor` as the output record so downstream patches keep working. Remove the Darknet and Caffe code paths.
- Haar cascades keep working (the C# `CascadeClassifier` stays in OpenCvSharp5). Ship the existing XML files.
- Remove `TrackerGOTURN`, list its removal in MIGRATION.md.

### 5.5 Windows package

- `Renderer` (WinForms preview window) with `OpenCvSharp5.GdipExtensions`.
- Media Foundation and DirectShow device enumeration with format lists, overriding the core enumeration when the package is referenced.
- The HDE extension (`VL.OpenCV5.HDE.vl`).

## 6. Code constraints

- Hot paths (facade image operations, capture loop, conversions) do not allocate in steady state. No LINQ, no closures, no boxing, no string formatting per frame. Reuse `Mat`s, use `ArrayPool<T>` and `Span<T>`.
- Every native object is disposed deterministically. Patched nodes dispose their state `Mat`s in `Dispose`, and live patching must not leak native memory (check with a soak test that recompiles repeatedly).
- `default` instead of `null` for optional OpenCvSharp5 array parameters.
- Nullable enabled, warnings as errors.
- No Windows API usage in `src/VL.OpenCV5`. A CI step fails the build if that project references `System.Windows.Forms`, `System.Drawing.Common` or anything in `Windows.Win32`.

## 7. Testing

- `VL.OpenCV5.Tests` runs in CI on Windows, Linux x64, Linux arm64 and macOS arm64 without vvvv. It covers
  - every facade method against a reference image with a tolerance, and against OpenCV 4 baselines where the behaviour changed intentionally (documented),
  - `CvImage` conversions for every supported depth and channel count, and graceful rejection of the new depth types,
  - a video file read through `VideoCapture` on each OS (no camera in CI),
  - an allocation test using `GC.GetAllocatedBytesForCurrentThread` around 1000 facade calls with reused `Mat`s, expecting zero,
  - Aruco detection on the bundled fiducial images, and ONNX YOLO on a small bundled model and image.
- Manual tests with real cameras on each OS and headless exports on Linux and macOS (section 8).

## 8. Milestones and acceptance criteria

M0 · Feasibility gate
- Confirm in vvvv 8 whether a node can call a method with a `ref struct` parameter or hold one in state. If it can, the facade is still preferred but less urgent.
- A plain .NET 10 console app on Windows, Linux x64, Linux arm64 and macOS arm64 loads OpenCvSharp5, prints `Cv2.GetVersionString()`, reads a video file and opens a camera.
- A minimal vvvv 8 export that references OpenCvSharp5 runs on `linux-x64` and `osx-arm64`, and the exporter copies the native runtimes.
- Confirm whether `OpenCvSharp5.runtime.linux-arm64` includes `videoio` with FFmpeg and V4L2.

M1 · Core project and facade
- New repository structure, `VL.OpenCV5` builds for `net10.0` with no Windows references.
- A script lists every OpenCvSharp member referenced in `VL.OpenCV.vl` (from the XML `OperationCallFlag` and `AssemblyCategory` references) and classifies it as unchanged, renamed, removed or ref-struct-affected. The result goes into `docs/api-inventory.md`.
- The facade covers the full inventory, with tests passing on all CI platforms.

M2 · Patch migration
- `VL.OpenCV.vl` becomes `VL.OpenCV5.vl`, rebound to the facade category by category, in this order: Basics and IO, Filter, Color, Drawing, Geometry, Arithmetic, Tracking, Calibration, Detection.
- Mechanical XML rewrites (dependency lists, the `InputArray.FromMat` pattern, `CvDnn` and `CvAruco` prefixes) may be scripted, but every category is opened and checked in the vvvv editor before it counts as done.
- All nodes compile without red in vvvv 8 on Windows.

M3 · Cross-platform video
- `VideoIn` with the threaded triple buffer and per-OS enumeration.
- A camera works on Windows, Linux (including a Raspberry Pi 5) and macOS arm64 in a headless export, at 640 × 480 and 30 fps or more.

M4 · Detection
- Stateful Aruco node, ONNX YOLO node, Haar cascades verified, GOTURN removed.

M5 · Windows package
- Renderer, Media Foundation enumeration and HDE extension split into `VL.OpenCV5.Windows`, behaving as in VL.OpenCV.

M6 · Help, migration guide, release
- All 57 help patches ported and opened without errors, pixel-exact examples rebaselined.
- `MIGRATION.md` lists renamed, removed and behaviour-changed nodes.
- NuGet release of `VL.OpenCV5` and `VL.OpenCV5.Windows`, exports tested on every target RID.

### 8.1 Work split with Claude Code

Estimated total for the maintainer is 3 to 5 weeks. The largest share is M2, because patch migration has to be checked in the vvvv editor.

| Milestone | My time | What Claude Code does alone | What needs from me |
|---|---|---|---|
| M0 Feasibility gate | 1 to 2 days | Console test app, CI matrix, runtime package checks | The ref struct test in vvvv 8, exports run on a Mac and a Linux machine |
| M1 Core and facade | 2 to 3 days | Repository split, API inventory script, facade, tests on all platforms | Reviewing the inventory and the facade signatures for VL friendliness |
| M2 Patch migration | 6 to 10 days | Scripted XML rewrites, a per-category checklist, fixes suggested from error lists | Opening every category in the editor, rebinding what the script cannot, judging node behaviour |
| M3 Cross-platform video | 2 to 4 days | Capture thread, triple buffer, V4L2 and AVFoundation enumeration | Camera tests on Windows, Raspberry Pi 5 and macOS |
| M4 Detection | 2 to 3 days | Aruco detector, ONNX YOLO pre and post processing, tests | Choosing the YOLO model, checking results on real scenes |
| M5 Windows package | 1 to 2 days | Moving Renderer, enumeration and HDE code, GdipExtensions port | Checking the Renderer and HDE in the editor |
| M6 Help and release | 3 to 5 days | MIGRATION.md, README, release workflow | Porting and checking 57 help patches, final exports |

## 9. Risks and open questions

- VL may not handle `ref struct` types at all, which makes the facade mandatory and M2 larger. M0 settles this.
- The patches are large (3.9 MB XML). Scripted rewrites can corrupt them silently, so every rewrite runs on a branch and is verified in the editor before merging.
- The `linux-arm64` runtime may lack FFmpeg or V4L2 support. Fallback is building OpenCvSharpExtern for arm64 in CI.
- The full Linux runtime needs GTK3 (`libgtk-3-0`) even for headless use, because highgui is linked. Document it, or build a custom runtime with videoio and dnn but without highgui.
- Licensing. VL.OpenCV is BSD-3-Clause and OpenCvSharp and OpenCV are Apache-2.0, all compatible. The full runtimes link FFmpeg (LGPL-2.1), which matters when shipping exports commercially.
- `VL.CoreLib.Windows` is referenced by the current `VL.OpenCV.vl`. Find which nodes use it during M1 and move them to the Windows package.
- vvvv 8 is still in preview, so VL.CoreLib APIs used by `VideoSourceToCvImage` may change.
- Coordination with the vvvv group on naming and on whether this should land upstream.

## 10. First task for Claude Code

1. Fork `vvvv/VL.OpenCV` into the new repository structure from section 3, keeping history.
2. Verify the facts in section 1 against the current commits of VL.OpenCV and OpenCvSharp5, and update this file where they differ.
3. Implement M0, except the vvvv editor test, which I do. Report which runtime packages load on which platform, and whether videoio works on each.
4. Write the API inventory script from M1 and produce `docs/api-inventory.md`.
