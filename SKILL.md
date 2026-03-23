---
name: remotion
description: >
  Make sure to use this skill whenever the user wants to create videos programmatically
  with React and Remotion. Trigger for: Remotion setup, compositions, animations,
  useCurrentFrame, interpolate, spring, Sequence, Audio, Video, rendering MP4/GIF/WebM,
  Remotion Studio, Remotion Player, Remotion Lambda, SSR rendering, or any task involving
  "video with code", "animate React components", "render video from React", "programmatic video".
---

# Remotion Skill

Remotion lets developers build videos as React components. Each frame is a React render — you
use hooks to get the current frame number and compute what to display. Remotion renders all
frames and encodes them into a video file.

## Project Setup

```bash
npx create-video@latest   # scaffold new project (choose Hello World template)
npm run dev               # open Remotion Studio at localhost:3000
```

## Fundamentals

### useCurrentFrame
Returns the current frame (0-indexed). This is the foundation of all animation:

```tsx
import { useCurrentFrame } from 'remotion';

export const MyComp = () => {
  const frame = useCurrentFrame(); // 0 → durationInFrames-1
  return <div style={{ opacity: frame / 30 }}>Hello</div>;
};
```

### useVideoConfig
Access composition metadata anywhere:

```tsx
import { useVideoConfig } from 'remotion';

const { width, height, fps, durationInFrames } = useVideoConfig();
// fps * seconds = total frames  e.g. 30fps × 5s = 150 frames
```

### Composition
Registered in `src/Root.tsx` — defines the video dimensions, framerate, duration:

```tsx
import { Composition } from 'remotion';
import { MyComp } from './MyComp';

export const RemotionRoot = () => (
  <Composition
    id="MyComp"           // used in CLI: npx remotion render MyComp
    component={MyComp}
    durationInFrames={150} // 5 seconds at 30fps
    fps={30}
    width={1920}
    height={1080}
    defaultProps={{ title: 'Hello' }}
  />
);
```

### AbsoluteFill
Full-canvas div (position: absolute, inset: 0). Use as the root of every composition:

```tsx
import { AbsoluteFill } from 'remotion';

<AbsoluteFill style={{ backgroundColor: '#000' }}>
  <h1>Content</h1>
</AbsoluteFill>
```

## Animation

### interpolate
Map any input range to any output range:

```tsx
import { interpolate, useCurrentFrame } from 'remotion';

const frame = useCurrentFrame();

// Fade in over first 30 frames
const opacity = interpolate(frame, [0, 30], [0, 1], {
  extrapolateLeft: 'clamp',
  extrapolateRight: 'clamp',
});

// Slide in from left
const x = interpolate(frame, [0, 45], [-300, 0], {
  extrapolateRight: 'clamp',
});

// With easing
import { Easing } from 'remotion';
const eased = interpolate(frame, [0, 60], [0, 1], {
  easing: Easing.bezier(0.8, 0, 0.2, 1),
  extrapolateRight: 'clamp',
});
```

Extrapolation options: `'clamp'` | `'extend'` | `'wrap'` | `'identity'`

### spring
Physics-based animation — natural bouncy feel:

```tsx
import { spring, useCurrentFrame, useVideoConfig } from 'remotion';

const frame = useCurrentFrame();
const { fps } = useVideoConfig();

const scale = spring({
  frame,
  fps,
  from: 0,
  to: 1,
  config: {
    damping: 10,      // higher = less bounce
    stiffness: 100,   // higher = faster/bouncier
    mass: 1,          // lower = faster
    overshootClamping: false,
  },
  delay: 10,                // start after 10 frames
  durationInFrames: 40,     // finish within 40 frames
});

<div style={{ transform: `scale(${scale})` }}>Hello</div>
```

## Layout & Timing

### Sequence
Offset children in the timeline. `useCurrentFrame()` inside resets to 0 at the sequence start:

```tsx
import { Sequence } from 'remotion';

<AbsoluteFill>
  <Sequence from={0} durationInFrames={60}>
    <Intro />          {/* plays frames 0-59 */}
  </Sequence>
  <Sequence from={60} durationInFrames={90}>
    <MainContent />    {/* plays frames 60-149 */}
  </Sequence>
  <Sequence from={150}>
    <Outro />          {/* plays from frame 150 to end */}
  </Sequence>
</AbsoluteFill>
```

## Media

### Audio
```tsx
import { Audio, staticFile } from 'remotion';

<Audio src={staticFile('music.mp3')} volume={0.5} />
<Audio src={staticFile('voice.mp3')} startFrom={30} endAt={120} />
<Audio src="https://example.com/sound.mp3" />
```

### Video
```tsx
import { Video, staticFile } from 'remotion';

<Video src={staticFile('clip.mp4')} style={{ width: '100%' }} />
```

### Img (use instead of `<img>`)
Remotion's `<Img>` waits for the image to load before rendering the frame:

```tsx
import { Img, staticFile } from 'remotion';

<Img src={staticFile('logo.png')} style={{ width: 200 }} />
```

### staticFile
References files in the `public/` folder:

```tsx
import { staticFile } from 'remotion';
const src = staticFile('photo.jpg'); // → /public/photo.jpg
```

## Async Data

Use `delayRender` / `continueRender` for any async operation before a frame renders:

```tsx
import { delayRender, continueRender, useEffect, useState } from 'remotion';

const [data, setData] = useState(null);
const handle = delayRender(); // pause rendering

useEffect(() => {
  fetch('/api/data')
    .then(r => r.json())
    .then(d => {
      setData(d);
      continueRender(handle); // resume
    });
}, []);
```

## Rendering

### CLI
```bash
npx remotion render <compositionId> [output]

# Examples
npx remotion render MyComp out/video.mp4
npx remotion render MyComp out/video.gif    # animated GIF
npx remotion render MyComp out/video.webm   # WebM

# Flags
--codec=h264          # default MP4
--codec=gif
--codec=vp8           # WebM
--quality=80          # 0-100
--scale=0.5           # resize output
--frames=0-90         # render subset
--concurrency=4       # parallel threads
--props='{"name":"Alice"}'  # pass input props
```

### Single frame (still image)
```bash
npx remotion still <compositionId> --frame=30 out/frame.png
```

### SSR API
```ts
import { bundle } from '@remotion/bundler';
import { renderMedia, selectComposition } from '@remotion/renderer';

const bundleLocation = await bundle({ entryPoint: './src/index.ts' });

const composition = await selectComposition({
  serveUrl: bundleLocation,
  id: 'MyComp',
  inputProps: { name: 'World' },
});

await renderMedia({
  composition,
  serveUrl: bundleLocation,
  codec: 'h264',
  outputLocation: 'out/video.mp4',
});
```

### AWS Lambda
```bash
npm i @remotion/lambda
npx remotion lambda functions deploy
npx remotion lambda render <serve-url> MyComp
```

## Remotion Player (embed in web app)

```tsx
import { Player } from '@remotion/player';
import { MyComp } from './MyComp';

<Player
  component={MyComp}
  inputProps={{ title: 'Hello' }}
  durationInFrames={150}
  fps={30}
  compositionWidth={1920}
  compositionHeight={1080}
  controls
  autoPlay
  loop
  style={{ width: '100%' }}
/>
```

## Props / Parameterized Videos

```tsx
type Props = { name: string; color: string };

const MyComp: React.FC<Props> = ({ name, color }) => {
  const frame = useCurrentFrame();
  const opacity = interpolate(frame, [0, 30], [0, 1], { extrapolateRight: 'clamp' });
  return <div style={{ color, opacity }}>{name}</div>;
};

// In Root.tsx
<Composition
  id="MyComp"
  component={MyComp}
  defaultProps={{ name: 'World', color: 'white' }}
  durationInFrames={90}
  fps={30}
  width={1280}
  height={720}
/>
```

## Common Patterns

**Fade in:**
```tsx
const opacity = interpolate(frame, [0, 20], [0, 1], { extrapolateRight: 'clamp' });
```

**Slide in from left:**
```tsx
const x = interpolate(frame, [0, 30], [-200, 0], { extrapolateRight: 'clamp' });
<div style={{ transform: `translateX(${x}px)` }} />
```

**Spring pop-in:**
```tsx
const scale = spring({ frame, fps, config: { damping: 12 } });
<div style={{ transform: `scale(${scale})` }} />
```

**Looping animation:**
```tsx
const loop = frame % 60;
const x = interpolate(loop, [0, 60], [0, 500]);
```

**Typewriter text:**
```tsx
const chars = Math.floor(interpolate(frame, [0, 60], [0, text.length], { extrapolateRight: 'clamp' }));
<div>{text.slice(0, chars)}</div>
```

**Staggered children:**
```tsx
{items.map((item, i) => {
  const delay = i * 10;
  const s = spring({ frame, fps, delay, config: { damping: 14 } });
  return <div key={i} style={{ transform: `scale(${s})` }}>{item}</div>;
})}
```

## Key Packages

| Package | Purpose |
|---------|---------|
| `remotion` | Core — hooks, components, CLI |
| `@remotion/renderer` | SSR rendering API |
| `@remotion/player` | Embed player in React apps |
| `@remotion/lambda` | Render on AWS Lambda |
| `@remotion/bundler` | Bundle for SSR |
| `@remotion/three` | React Three Fiber integration |
| `@remotion/lottie` | Render Lottie animations |
| `@remotion/gif` | Export animated GIFs |
| `@remotion/noise` | Perlin noise for organic motion |
| `@remotion/paths` | SVG path animation |
| `@remotion/shapes` | Primitive shape components |
| `@remotion/fonts` | Google Fonts loader |
| `@remotion/captions` | Auto captions |
| `@remotion/tailwind` | Tailwind CSS support |
| `@remotion/motion-blur` | Motion blur effect |
| `@remotion/rive` | Rive animation support |

## Rules

- Always use `<Img>` not `<img>` — it waits for image load before rendering
- Always use `extrapolateLeft/Right: 'clamp'` on `interpolate` unless you want values to extend
- Frames per second × seconds = `durationInFrames` (30fps × 5s = 150)
- Everything in render must be **deterministic per frame** — no random values, no Date.now()
- For async data (fonts, API calls), always use `delayRender` / `continueRender`
- Use `<Sequence>` to organize scenes — inside a Sequence, `useCurrentFrame()` resets to 0
- Use `spring()` for UI feel animations; use `interpolate()` for linear/eased transitions
