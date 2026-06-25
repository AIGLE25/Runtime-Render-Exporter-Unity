# Runtime Render Exporter

Runtime Render Exporter is a lightweight Unity runtime exporter for deterministic Camera-to-MP4 and PNG sequence rendering.

It renders a selected Camera at an exact resolution and frame rate, reads each frame, then exports either directly to MP4 through ffmpeg or to a PNG sequence. It does not depend on any MIDI, piano, Timeline, or project-specific code.

## Requirements

- Unity 2022.3 or newer.
- A Camera in your scene.
- ffmpeg installed separately.

ffmpeg is not bundled in this package. Install it yourself and either:

- add `ffmpeg` to your system `PATH`, or
- select the exact `ffmpeg.exe` path from the plugin window.

## Compatibility

Runtime Render Exporter is designed around Unity's normal `Camera.Render()` path. In practice, that means it should work with most regular Unity projects where a Camera can render into a RenderTexture.

| Project type | Expected status | Notes |
| --- | --- | --- |
| Built-in Render Pipeline 3D | Supported | Standard Camera render path. |
| Built-in Render Pipeline 2D | Supported | Works if the scene is visible through the selected Camera. |
| URP 3D | Supported | This is the main target for V1. |
| URP 2D | Supported | Works with normal 2D Renderer camera output. |
| HDRP | Expected to work, needs validation | HDRP can have project-specific camera/post-process behavior, so test before claiming full support. |
| UI rendered by Camera | Supported | Canvas must be `Screen Space - Camera` or `World Space`. |
| UI rendered as Overlay | Not guaranteed | Overlay UI is not always captured by direct Camera rendering. |
| VR / XR / MR | Not V1 target | Use a normal mono export Camera for now. Stereo/XR mirror capture needs a dedicated path. |
| Cinemachine | Supported | Cinemachine drives the Camera; the exporter renders that Camera. |
| Timeline | Manual for now | Use `OnFrameRequested` and call `PlayableDirector.Evaluate()`. |

The safest V1 claim is:

> Works with Built-in and URP camera-based 2D/3D projects. HDRP should be tested per project. XR/MR/stereo capture is not a V1 feature.

## Installation

Import Runtime Render Exporter from the Unity Asset Store using Unity's Package Manager / My Assets workflow.

After import, the runtime code, editor tools, and demo scenes are already inside your project.

Open one of the included demo scenes:

```text
Assets/Export addon folder/RuntimeRenderSmokeTest.unity
Assets/Export addon folder/Demo/MidiPhysicsAddonDemo.unity
```

Then enter Play Mode and use the on-screen export controls.

## Quick Start

### Fastest working setup

1. Install the package.
2. Put a Camera in the scene.
3. Add `Runtime Camera Mp4 Exporter` to that Camera.
4. Set `Source Camera` to the same Camera.
5. Set `Output Path`, for example:

```text
Exports/render.mp4
```

6. Set `FfmpegPath` to either:

```text
ffmpeg
```

if ffmpeg is in PATH, or the full path to `ffmpeg.exe`.

7. Enter Play Mode and press `Start MP4 Export`.

### Which mode should I use?

| Goal | Recommended settings |
| --- | --- |
| Most compatible MP4 | `ExportMode = StreamMp4Only`, `ColorMode = Standard` |
| Best color fidelity for synthetic/game visuals | `ExportMode = StreamMp4Only`, `ColorMode = RGB Master` |
| Debug exact Unity frames before video encoding | `ExportMode = PNG Sequence Only` |
| What-you-see-is-what-you-export from the Game View | `CaptureMode = GameView`, `ResizeGameViewToExport = true` |
| Clean offscreen Camera export | `CaptureMode = DirectCamera` |
| Fallback comparison path | `ExportMode = StreamMp4Legacy` |

`DirectCamera` renders the selected Camera into an offscreen RenderTexture. It is good for clean camera-based exports and does not depend on the Game View window size.

`GameView` captures the presented Game View/player frame. Use it when you need the export to match exactly what the user sees on screen. Keep `ResizeGameViewToExport` enabled so the Game View/player is temporarily requested at the export resolution.

### Camera component flow

1. Open a scene with a Camera.
2. Select the Camera GameObject.
3. Click `Add Component`.
4. Add `Runtime Camera Mp4 Exporter`.
5. Set:
   - `Render Preset` if you want a quick starting point
   - `Width`
   - `Height`
   - `Frame Rate`
   - `Frame Count`
   - `Quality`
   - `Output Path`
6. Use `Use ffmpeg From PATH` or `Browse ffmpeg`.
7. Enter Play Mode.
8. Click `Start MP4 Export` in the component Inspector, or call `StartExport()` from your runtime UI.

The exported MP4 is written to `Output Path`.

Relative output paths are resolved automatically:

- in the Unity Editor, relative to the project root;
- in a built player, relative to `Application.persistentDataPath`.

Absolute paths are used unchanged.

The `Tools > Runtime Render > MP4 Exporter` window is optional and Editor-only. The main workflow is the Camera component.

### Built player usage

For a real build, call the exporter from your own runtime UI, button, keyboard shortcut, debug menu, or gameplay code.

Minimal runtime example:

```csharp
using Brice.RuntimeRender;
using UnityEngine;

public sealed class RuntimeExportButton : MonoBehaviour
{
    [SerializeField] private RuntimeCameraMp4Exporter exporter;

    private void OnGUI()
    {
        if (GUILayout.Button("Export MP4"))
        {
            exporter.StartExport();
        }
    }
}
```

In production, connect `StartExport()` to a normal Unity UI `Button`, UI Toolkit button, or your app's export panel.

## Component Usage

You can also add the component manually:

1. Add `RuntimeCameraMp4Exporter` to any GameObject.
2. Assign `Source Camera`.
3. Configure `Settings`.
4. Call `StartExport()`.

Example:

```csharp
using Brice.RuntimeRender;
using UnityEngine;

public sealed class ExportButton : MonoBehaviour
{
    [SerializeField] private RuntimeCameraMp4Exporter exporter;

    public void Export()
    {
        exporter.Settings.Width = 1920;
        exporter.Settings.Height = 1080;
        exporter.Settings.FrameRate = 60;
        exporter.Settings.FrameCount = 600;
        exporter.Settings.Quality = 90;
        exporter.Settings.OutputPath = "Exports/render.mp4";
        exporter.Settings.FfmpegPath = "ffmpeg";

        exporter.StartExport();
    }
}
```

## Deterministic Frame Control

Use `OnFrameRequested` if your scene must be advanced manually for each exported frame.

This is useful for custom animation systems, Timeline-like playback, simulation stepping, MIDI visualizers, replay viewers, or any scene where the exported frame should not depend on real-time playback.

```csharp
exporter.OnFrameRequested += (frameIndex, time) =>
{
    // Move your scene to the exact export time before the Camera renders.
    playableDirector.time = time;
    playableDirector.Evaluate();
};

exporter.StartExport();
```

The `time` value is calculated like this:

```text
time = frameIndex / frameRate
```

For content with a known duration, calculate `FrameCount` from that duration before starting the export. This lets each MIDI file, Timeline, replay, or simulation choose its own export length automatically:

```csharp
var durationSeconds = midiSong.DurationSeconds;
var frameRate = 60;

exporter.Settings.FrameRate = frameRate;
exporter.Settings.FrameCount = Mathf.CeilToInt(durationSeconds * frameRate);

exporter.OnFrameRequested += (frameIndex, time) =>
{
    midiVisualizer.SeekToTime(time);
};

exporter.StartExport();
```

The user does not need to enter a frame count manually if your app already knows the content duration.

## Progress, Completion, and Errors

```csharp
exporter.OnProgress += progress =>
{
    Debug.Log($"{progress.FrameIndex + 1}/{progress.FrameCount}: {progress.Message} ETA {progress.EstimatedRemainingSeconds:0}s");
};

exporter.OnCompleted += path =>
{
    Debug.Log($"Export complete: {path}");
};

exporter.OnFailed += error =>
{
    Debug.LogError($"Export failed: {error}");
};
```

To cancel an export:

```csharp
exporter.CancelExport();
```

## Export Handler

`Export Handler` is optional. Leave it empty for the normal export workflow.

Use it when you want a custom `MonoBehaviour` to run before the exporter starts. The handler must implement `IRuntimeRenderExportHandler`:

```csharp
using Brice.RuntimeRender;
using UnityEngine;

public sealed class MyExportHandler : MonoBehaviour, IRuntimeRenderExportHandler
{
    public bool TryStartExport(RuntimeCameraMp4Exporter exporter)
    {
        Debug.Log("Prepare custom export state here.");

        // Return false to let RuntimeCameraMp4Exporter continue with its normal export.
        // Return true if this handler fully takes over and starts/controls the export itself.
        return false;
    }
}
```

This is useful for preparing Timeline, MIDI/replay state, custom UI, metadata files, or project-specific export logic.

## Settings

| Setting | Description |
| --- | --- |
| `ExportMode` | `StreamMp4Only`, `StreamMp4Legacy`, `PngSequenceOnly`, `PngSequenceAndMp4Keep`, or `PngSequenceAndMp4Delete`. `StreamMp4Legacy` uses the older stream/readback path for comparison, while codec and pixel format still come from `ColorMode` or the advanced ffmpeg fields. |
| `CaptureMode` | `DirectCamera` renders the selected Camera to a RenderTexture. `GameView` captures the current game view/screen frame. |
| `Width` | Output video width in pixels. |
| `Height` | Output video height in pixels. |
| `FrameRate` | Output video FPS. |
| `FrameCount` | Number of frames to export. |
| `Quality` | MP4 quality from `0` to `100`. Higher is better/larger. |
| `SpeedPreset` | ffmpeg x264 preset. Slower presets compress better. |
| `ColorMode` | `Standard` for compatible `yuv420p`, `StandardFullRange` for `yuv420p` with full-range tags, `YUV444 Sharp` (`Yuv444Master`) for sharper synthetic/color detail using BT.709 limited range, or `RgbMaster` for RGB-oriented master output. |
| `Mp4Codec` | Optional ffmpeg codec override. Empty means the selected `ColorMode` decides. |
| `Mp4PixelFormat` | Optional output pixel format override, for example `yuv420p`, `yuv444p`, or `rgb24`. Empty means the selected `ColorMode` decides. |
| `Mp4ColorTags` | Optional ffmpeg color/tag arguments override. Empty means the selected `ColorMode` decides. |
| `FfmpegOutputArguments` | Extra ffmpeg arguments inserted before the output path. |
| `FfmpegArgumentsOverride` | Full ffmpeg argument override. In the Editor UI this is shown as the editable `ffmpeg arguments` field under `Advanced ffmpeg`. Supports tokens listed below. |
| `OutputPath` | Final `.mp4` path. Absolute paths are used unchanged. Relative paths are resolved to the project root in Editor and to `Application.persistentDataPath` in builds. Default: `Exports/export.mp4`. |
| `FfmpegPath` | `ffmpeg` from PATH or a direct executable path. |
| `PngDirectory` | Output folder for PNG sequence modes. Absolute paths are used unchanged. Relative paths are resolved next to the resolved `OutputPath`. |
| `PngPrefix` | PNG filename prefix. |
| `RestoreCameraTarget` | Restores the Camera target texture after export. |
| `KeepGameViewRendering` | Restores the Camera target after each exported frame so the Game View/player keeps rendering during export. Disable for a tiny performance gain when using a dedicated export Camera. |
| `UseUnityCaptureFramerate` | Sets `Time.captureFramerate` during export so Unity time, particles, and regular runtime animation advance at the requested FPS. |
| `ResizeGameViewToExport` | In `GameView` capture mode, temporarily requests the Game View/player backbuffer to match export `Width` and `Height`, then restores the previous resolution after export. |
| `ExportHandler` | Optional `MonoBehaviour` implementing `IRuntimeRenderExportHandler`. Leave empty unless you need custom pre-export or takeover logic. |

### ffmpeg override tokens

`FfmpegArgumentsOverride` can use these tokens:

```text
{input}, {output}, {width}, {height}, {fps}, {crf}, {preset}, {codec}, {pix_fmt}, {color_tags}, {extra_args}, {vf}
```

For stream exports, `{input}` resolves to `-`. For PNG sequence exports, it resolves to the generated PNG pattern.

Templates in the Inspector and `Tools > Runtime Render > MP4 Exporter` only fill the normal fields. After applying a template, every field remains editable.

The `Advanced ffmpeg` command field shows the generated ffmpeg arguments by default. If you edit it manually and start an export immediately, the export uses that text. Changing a normal export field regenerates the command from the visible settings.

## What V1 Does

- Direct Camera rendering.
- Exact resolution.
- Exact frame count.
- Exact frame rate metadata.
- Direct rawvideo streaming to ffmpeg for MP4 exports.
- Optional PNG sequence export, with optional MP4 encoding from the generated sequence.
- Editor window for setup and export.
- Runtime API for custom UI/buttons.

## What V1 Does Not Do

- It does not bundle ffmpeg.
- It does not export audio.
- It does not capture Unity UI automatically unless that UI is rendered by the selected Camera.
- It does not use AsyncGPUReadback.
- It does not use URP RenderGraph readback.
- It does not provide batch export yet.
- It does not automatically drive Timeline yet. Use `OnFrameRequested` for now.

## Troubleshooting

### ffmpeg does not start

Set `FfmpegPath` to a direct executable path using `Browse ffmpeg`, or make sure `ffmpeg` is available from your system `PATH`.

### The output file is missing or empty

Check:

- `OutputPath` points to a writable folder.
- The filename ends in `.mp4`.
- ffmpeg can write to the selected location.
- No other program is locking the output file.

### The video is black

Check:

- `Source Camera` is assigned.
- The Camera renders the objects you expect.
- The Camera culling mask includes your scene layers.
- The export is started in Play Mode.

### Game View looks softer than Direct Camera

`DirectCamera` renders directly at the export `Width` and `Height`, so it is usually the sharpest path for clean offscreen camera exports.

`GameView` captures the current Game View/screen frame. It is the best match for what the user sees on screen, but if the Game View is smaller than the export size, the captured frame must be upscaled and will look softer or more pixelated.

Keep `ResizeGameViewToExport` enabled for the cleanest Game View export. The exporter temporarily requests the Game View/player resolution to match the export size, captures at that size, then restores the previous resolution after export.

### PNG looks correct but MP4 colors differ

This usually comes from video encoding, not from Unity capture.

`Standard` uses H.264 `yuv420p` for compatibility. This can slightly change saturated colors.

For better color fidelity, try:

```text
ColorMode = RGB Master
```

For a sharp YUV export, try:

```text
ColorMode = YUV444 Sharp
```

### The video has the wrong duration

Duration is controlled by:

```text
FrameCount / FrameRate
```

For example:

```text
600 frames / 60 FPS = 10 seconds
```

### My animation plays in real time instead of exact frames

Use `OnFrameRequested` and manually evaluate your scene at the provided `time`.

## Product Notes

The stable V1 path is direct Camera render plus ffmpeg streaming.

AsyncGPUReadback and URP RenderGraph capture are intentionally not included yet. They are promising for performance, but they need isolated reliability work before they should become a sellable feature.



https://github.com/user-attachments/assets/0eea27b3-12a6-46cb-9251-f683f9e81557

