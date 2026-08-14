# Annotating product screenshots and recordings

Use Remotion to mark newly added UI in product screenshots and short flow
recordings. Record the real interaction with Agent Browser, then add callouts in
a frame-accurate Remotion composition. Keep the source capture unchanged.

FFmpeg is a fallback for conversion, compression, frame extraction, and simple
emergency overlays. Do not build complex annotation timelines as FFmpeg filter
graphs when Remotion is available.

## Output location

Keep captures, composition projects, and rendering intermediates outside the
product repository unless the user explicitly wants them committed:

```sh
demo_tmp_dir=$(mktemp -d "${TMPDIR:-/tmp}/product-demo.XXXXXX")
```

Use a task-specific variable name. Report the final path because operating
system cleanup may eventually remove temporary directories.

## Reuse one Remotion template

A fresh Remotion installation is relatively expensive. In one measured local
run, installing the dependencies took about 73 seconds and occupied roughly
548 MB, while rendering a 14.6-second 1440×1000 composition took about 30
seconds. Reuse an installed template rather than scaffolding a project for each
video.

The template needs these packages at matching Remotion versions:

```json
{
  "dependencies": {
    "@remotion/cli": "<same-version-as-remotion>",
    "react": "<supported-version>",
    "react-dom": "<supported-version>",
    "remotion": "<same-version-as-cli>"
  }
}
```

For a first-time setup, follow the current Remotion project instructions:

```sh
npx create-video@latest --yes --blank product-demo-video
```

Do not copy `node_modules` into product repositories. Keep the reusable
template in a personal tools or system-temporary location.

## Capture the unannotated flow

Use Agent Browser for the real interaction and leave pauses long enough for a
human viewer:

```sh
agent-browser --session product-demo set viewport 1440 1000
agent-browser --session product-demo record start "$demo_tmp_dir/flow.webm"
agent-browser --session product-demo open http://app.localhost/example
agent-browser --session product-demo wait 800
# Fill and click the actual flow here.
agent-browser --session product-demo wait 1200
agent-browser --session product-demo record stop
agent-browser --session product-demo close
```

Capture important states as PNG files in the same session. Preserve all raw
captures; annotated outputs should use an `annotated-` prefix.

## Find timings and coordinates

Inspect the source metadata, then extract representative frames around each
interaction:

```sh
ffprobe -v error \
  -show_entries stream=width,height:format=duration \
  -of default=noprint_wrappers=1 \
  "$demo_tmp_dir/flow.webm"

ffmpeg -loglevel error -y \
  -ss 4.5 -i "$demo_tmp_dir/flow.webm" \
  -frames:v 1 "$demo_tmp_dir/frame-4.5.png"
```

Record each callout as `start`, `end`, `x`, `y`, `width`, and `height`. Sample
the beginning, middle, and end of scrolling segments because a fixed box may
drift away from its target.

## Define callouts as data

Keep project-specific work in a segment array. Reusing the composition and
changing only this data is the main token and maintenance advantage over
handwritten FFmpeg filters.

```tsx
type Segment = {
  start: number;
  end: number;
  number: number;
  label: string;
  box: { x: number; y: number; width: number; height: number };
};

const segments: Segment[] = [
  {
    start: 8,
    end: 50,
    number: 1,
    label: "New dish economics at a glance",
    box: { x: 340, y: 215, width: 710, height: 275 },
  },
];
```

Store timing in frames. Convert seconds with `Math.round(seconds * fps)` when
preparing the data.

## Composition pattern

Place the untouched recording beneath CSS annotations. Use
`useCurrentFrame()` to select the active segment and `spring()` or
`interpolate()` for deterministic entrances and exits.

```tsx
import {
  AbsoluteFill,
  OffthreadVideo,
  interpolate,
  spring,
  staticFile,
  useCurrentFrame,
  useVideoConfig,
} from "remotion";

const AnnotatedFlow = () => {
  const frame = useCurrentFrame();
  const { fps } = useVideoConfig();
  const segment = segments.find(
    ({ start, end }) => frame >= start && frame < end,
  );

  if (!segment) {
    return <OffthreadVideo src={staticFile("source.webm")} />;
  }

  const entrance = spring({
    frame: frame - segment.start,
    fps,
    config: { damping: 18, stiffness: 180, mass: 0.7 },
  });
  const exit = interpolate(segment.end - frame, [0, 6], [0, 1], {
    extrapolateLeft: "clamp",
    extrapolateRight: "clamp",
  });
  const visibility = entrance * exit;
  const { x, y, width, height } = segment.box;

  return (
    <AbsoluteFill>
      <OffthreadVideo src={staticFile("source.webm")} />
      <div
        style={{
          position: "absolute",
          left: x,
          top: y,
          width,
          height,
          border: "5px solid #b93427",
          borderRadius: 14,
          boxShadow:
            "0 0 0 3px rgba(255,255,255,.92), 0 0 0 2000px rgba(36,21,15,.08)",
          opacity: visibility,
          transform: `scale(${0.985 + visibility * 0.015})`,
        }}
      />
      {/* Render the reusable numbered label component here. */}
    </AbsoluteFill>
  );
};
```

Use the product's actual font and design tokens. Copy required local font files
into the Remotion `public/` directory and load them with `@font-face`. Avoid
network-dependent assets during rendering.

## Visual language

Use one restrained annotation system throughout a video:

- numbered badge;
- short label describing the new behavior;
- 3–5 px rounded outline around the relevant UI;
- canvas-colored outer stroke for separation;
- subtle focus mask outside the highlighted region;
- quick spring entrance and a short fade before the next callout.

Do not cover the value or control being explained. Keep labels to one line when
possible, and use at least 20 px text for a 1440 px-wide recording.

## Render video and screenshots

Register a composition with the source video's dimensions, frame rate, and
duration, then render it through the Remotion CLI:

```sh
npx remotion render \
  src/index.ts ProductDemo \
  "$demo_tmp_dir/annotated-flow.mp4" \
  --codec=h264 --crf=18
```

Use `remotion still` for an annotated screenshot or representative video frame:

```sh
npx remotion still \
  src/index.ts ProductDemo \
  "$demo_tmp_dir/annotated-result.png" \
  --frame=150
```

For a standalone screenshot, use an image-backed composition at the source
PNG's exact dimensions and render the same callout components over it. Do not
resize the underlying screenshot. Generative image editing can redraw text and
UI, so it is unsuitable when exact product fidelity matters.

## Why not record a composed video player

Do not ask Agent Browser to record a page that merely plays an existing video
with HTML overlays. Depending on the browser compositor and codec setup,
stopping the recording may fail with `No frames captured` or an FFmpeg encoding
error.

Remotion avoids this failure by rendering every frame directly. The reliable
workflow is:

1. Agent Browser records the real interaction once.
2. Remotion reads that recording as an asset.
3. Remotion renders frame-accurate callouts and animation.
4. FFmpeg optionally compresses or converts the final file.

## Verification and cleanup

Check the deliverables before sharing them:

```sh
ffprobe -v error \
  -show_entries stream=codec_name,width,height:format=duration \
  -of default=noprint_wrappers=1 \
  "$demo_tmp_dir/annotated-flow.mp4"
```

- Inspect a frame from every callout interval.
- Check the first and last six frames of transitions for flicker.
- Confirm labels do not cover highlighted values or controls.
- Confirm the output keeps the intended dimensions and duration.
- Close Agent Browser sessions and stop temporary development servers.
- Keep raw captures and final outputs; remove disposable extracted frames.
- Check the product repository's Git status to ensure media was not added.

## FFmpeg fallback

Use FFmpeg directly only when Remotion cannot run or the task is a single
static box with no reusable animation. FFmpeg remains appropriate for:

- extracting inspection frames;
- converting WebM to MP4;
- removing unused audio tracks;
- final compression and fast-start metadata;
- emergency one-off overlays.

For repeated annotations, a declarative Remotion segment array is easier to
review, adjust, and reuse than a long filter graph.
