---
name: voiceover
description: Create Algerian Darija narration and integrate Natiq audio into Remotion, motion graphics, or ffmpeg video projects. Use when a video needs Algerian voiceover or existing narration needs revision.
---

Use the connected Natiq MCP tools and follow the server's instructions for
catalogue discovery, billing, asynchronous jobs and download URLs. If the server
is unavailable, explain the connection problem; never pretend audio was generated.

Keep the narration conversational Algerian Darija, with short spoken sentences.
Preserve product names and intentional French words. For high-stakes copy or
uncertain regional phrasing, ask for a native-speaker review. Do not claim perfect
Darija. Keep each scene independently editable, and keep one voice and delivery
style unless a character change is intentional.

Before generation, create a scene manifest in the user's video project containing
scene ID, narration, voice, purpose, country, speed, job ID, local audio path and
measured duration. Save the job ID as soon as a submit returns. Compare narration
AND voice/performance settings before reusing audio. If only visuals change, keep
the audio. If the requested runtime is too short after generation, explain the
tradeoff and shorten the script or extend the scene; do not silently cut speech.

Download successful clips into `public/audio/` for Remotion (or the project's
existing asset directory). Use stable names such as `scene-01.wav`. Signed URLs
are temporary: store local paths and job IDs in the manifest, not signed URLs.
Do not forward the API key to the audio host. Example, substituting the returned URL:

```bash
mkdir -p public/audio
curl --fail --location --output public/audio/scene-01.wav 'RETURNED_AUDIO_URL'
```

Use the existing project's frame rate. Round each scene up to whole frames and
add any deliberate pause separately. For example, with `scenes` from the manifest:

```tsx
import {Html5Audio, Series, staticFile, useVideoConfig} from 'remotion';

export const Narration = ({scenes}) => {
  const {fps} = useVideoConfig();
  return <Series>{scenes.map(scene => (
    <Series.Sequence key={scene.id}
      durationInFrames={Math.ceil(scene.duration_seconds * fps) + scene.pauseFrames}>
      <Html5Audio src={staticFile(scene.audio)} />
      {/* Put the scene's visuals here, inside the same Sequence. */}
    </Series.Sequence>
  ))}</Series>;
};
```

Set the composition duration to the sum of those exact frame counts. `scene.audio`
is relative to `public/`, for example `audio/scene-01.wav`. For ffmpeg, likewise
derive each visual segment's length from the successful audio duration and any
explicit pause. Listen to transitions and check the final frame for clipped words.

Natiq has no word timestamps. Prefer scene-level captions, or label proportional
caption timings as **estimated** and manually review them. Use a separate forced
alignment tool if the user needs accurate word highlighting. Render a preview,
check Arabic shaping/RTL text, voice/music balance, pacing and pronunciation, then
render the requested output. Refer to the README example for a minimal Remotion
composition and render command.
