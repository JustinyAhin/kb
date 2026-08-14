# Annotating product screenshots and recordings

Capture the real product UI first and keep that source unchanged. Choose the
lightest annotation path that preserves fidelity:

- **Static screenshots:** create a transparent SVG overlay, rasterize it with
  `rsvg-convert`, and composite it over the raw PNG with FFmpeg.
- **Timed or animated recordings:** use Remotion for frame-accurate callouts.
- **Conversion and compression:** use FFmpeg directly.

Do not use generative image editing for product captures. It can redraw text
and controls instead of preserving the real interface.

## Output location

Keep captures, overlays, composition projects, and rendering intermediates
outside the product repository unless the user explicitly wants them
committed:

```sh
demo_tmp_dir=$(mktemp -d "${TMPDIR:-/tmp}/product-demo.XXXXXX")
```

Use a task-specific variable name. Report the final path because operating
system cleanup may eventually remove temporary directories.

Use predictable names that keep raw and annotated files paired:

```text
authority-hero-raw.png
annotated-authority-hero.png
flow-raw.webm
annotated-flow.mp4
```

## Capture the untouched source

Use the browser-control surface available in the current environment. Prefer
the in-app Browser when its skill is available; otherwise use Agent Browser or
another supported browser automation tool. Always capture the real rendered
page rather than recreating it.

Do not resize the browser only to make a screenshot look nicer. Preserve the
current viewport unless the deliverable specifies dimensions or the task is
testing a responsive breakpoint. When temporarily overriding a viewport, reset
it after capture.

For Agent Browser CLI, a recording flow looks like this:

```sh
agent-browser --session product-demo set viewport 1440 1000
agent-browser --session product-demo record start "$demo_tmp_dir/flow-raw.webm"
agent-browser --session product-demo open http://app.localhost/example
agent-browser --session product-demo wait 800
# Fill and click the actual flow here.
agent-browser --session product-demo wait 1200
agent-browser --session product-demo record stop
agent-browser --session product-demo close
```

Use an explicit viewport only when `1440×1000` is part of the intended output.
Capture important states as PNG files in the same session.

## Fast path for annotated screenshots

First inspect the raw screenshot dimensions and FFmpeg capabilities:

```sh
ffprobe -v error -select_streams v:0 \
  -show_entries stream=width,height \
  -of csv=s=x:p=0 \
  "$demo_tmp_dir/authority-hero-raw.png"

ffmpeg -hide_banner -filters | rg 'drawtext|overlay'
```

Some FFmpeg builds provide `overlay` but omit `drawtext`. In that case, do not
fight a long filter graph. Author labels, badges, connector lines, and arrows
in a transparent SVG with the exact source dimensions:

```svg
<svg xmlns="http://www.w3.org/2000/svg"
  width="1880" height="1071" viewBox="0 0 1880 1071">
  <rect x="80" y="300" width="360" height="96" rx="14"
    fill="#2a201b" fill-opacity="0.94" />
  <circle cx="122" cy="348" r="23" fill="#8e3b32" />
  <text x="122" y="356" text-anchor="middle"
    font-family="Arial, sans-serif" font-size="23" font-weight="700"
    fill="#fffaf0">1</text>
  <text x="160" y="340" font-family="Arial, sans-serif"
    font-size="21" font-weight="700" fill="#fffaf0">Amazon leads</text>
</svg>
```

Rasterize and composite the overlay while leaving the raw capture untouched:

```sh
rsvg-convert -w 1880 -h 1071 \
  -o "$demo_tmp_dir/authority-hero-overlay.png" \
  "$demo_tmp_dir/authority-hero-overlay.svg"

ffmpeg -hide_banner -loglevel error -y \
  -i "$demo_tmp_dir/authority-hero-raw.png" \
  -i "$demo_tmp_dir/authority-hero-overlay.png" \
  -filter_complex '[0:v][1:v]overlay=0:0:format=auto' \
  -frames:v 1 \
  "$demo_tmp_dir/annotated-authority-hero.png"
```

Use the actual screenshot dimensions in the SVG and `rsvg-convert` command.
This path is appropriate for a small set of stills with simple callouts. Switch
to Remotion when annotations need timing, animation, or repeated behavior.

## Visual language

Use one restrained annotation system throughout a deliverable:

- numbered badge;
- short label describing the new behavior;
- 3–5 px rounded outline or connector pointing to the relevant UI;
- canvas-colored outer stroke for separation;
- subtle focus mask when the surrounding interface competes for attention;
- quick spring entrance and short fade for animated callouts.

Do not cover the value or control being explained. Keep labels to one line when
possible, and use at least 20 px text for a 1440 px-wide capture.

## Animated recordings with Remotion

### Reuse one template

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

### Find timings and coordinates

Inspect the source metadata, then extract representative frames around each
interaction:

```sh
ffprobe -v error \
  -show_entries stream=width,height:format=duration \
  -of default=noprint_wrappers=1 \
  "$demo_tmp_dir/flow-raw.webm"

ffmpeg -loglevel error -y \
  -ss 4.5 -i "$demo_tmp_dir/flow-raw.webm" \
  -frames:v 1 "$demo_tmp_dir/frame-4.5.png"
```

Record each callout as `start`, `end`, `x`, `y`, `width`, and `height`. Sample
the beginning, middle, and end of scrolling segments because a fixed box may
drift away from its target.

### Define callouts as data

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

### Composition pattern

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

### Render video and screenshots

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

For a standalone screenshot that genuinely needs the reusable Remotion
components, use an image-backed composition at the source PNG's exact
dimensions. Do not resize the underlying screenshot.

### Why not record a composed video player

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

## Verification, delivery, and cleanup

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
- Embed requested screenshots inline in the final response and link the
  artifact directory.
- Report which files are raw and which are annotated.
- Close Agent Browser sessions and stop temporary development servers.
- Keep raw captures and final outputs; remove disposable extracted frames.
- Check the product repository's Git status to ensure media was not added.

## Keep FFmpeg filters simple

FFmpeg remains appropriate for:

- compositing rasterized SVG overlays onto static screenshots;
- extracting inspection frames;
- converting WebM to MP4;
- removing unused audio tracks;
- final compression and fast-start metadata;
- simple one-off boxes when `drawtext` is not needed.

Do not build animation timelines as long FFmpeg filter graphs. For repeated or
timed annotations, a declarative Remotion segment array is easier to review,
adjust, and reuse.
