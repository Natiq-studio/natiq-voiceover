# Natiq voiceover

Algerian Darija voiceovers inside Claude Code for Remotion, motion graphics and
ffmpeg projects. The plugin adds a practical video workflow skill and connects to
[Natiq Studio](https://natiq.studio)'s remote MCP server. No local speech model or
server is needed. The plugin is MIT licensed; Natiq generation is a paid service
using your account's tokens and [API terms and pricing](https://natiq.studio/api/docs/).

## Install

1. Sign in and create a **write-scoped** API key at
   [natiq.studio/api/keys](https://natiq.studio/api/keys/). A read-only key can browse
   voices and balance, but cannot generate audio.
2. Set `NATIQ_API_KEY` in the shell that launches Claude Code. Avoid putting the
   key in a repository, a prompt, or a committed shell script. Bash example that
   does not put the key in shell history:

   ```bash
   read -rsp 'Natiq API key: ' NATIQ_API_KEY; echo
   export NATIQ_API_KEY
   claude
   ```

   PowerShell 7: `$env:NATIQ_API_KEY = Read-Host -MaskInput 'Natiq API key'`, then
   launch `claude` from that shell. Restart Claude after changing the variable.
3. From a local clone, run these commands inside Claude Code:

   ```text
   /plugin marketplace add /absolute/path/to/natiq-voiceover
   /plugin install natiq-voiceover@natiq-tools
   /reload-plugins
   /mcp
   ```

   Once published, replace the local path with the GitHub `OWNER/natiq-voiceover`
   shown on this repository's page. `OWNER` means the actual repository owner,
   not the Natiq domain. Approve the MCP connection if Claude prompts you.
   For development you can also launch `claude --plugin-dir /path/to/natiq-voiceover`.

Use `/natiq-voiceover:voiceover` or ask for an Algerian voiceover in a video task.
The environment variable is expanded in `.mcp.json`; never replace it with a key
in that file. If `/mcp` reports 401, check the launching shell's variable and key
expiry/revocation. A 403 may mean the key is read-only. Respect rate-limit delays.

Without the plugin, direct MCP setup is also supported:

```bash
claude mcp add --transport http natiq https://natiq.studio/mcp --header "Authorization: Bearer $NATIQ_API_KEY"
```

That command stores the resolved key in Claude's local MCP configuration. Prefer
the plugin's environment-variable configuration if you do not want a saved key.
Install either connection to avoid duplicate Natiq tools.

## Tools and behavior

| Tool | Purpose |
| --- | --- |
| `list_voices` | Discover current voice IDs and previews |
| `list_purposes` | Discover delivery styles |
| `check_balance` | Read remaining tokens and pricing |
| `generate_speech` | Submit one scene; spends tokens once per call |
| `get_speech` | Poll an owned job and refresh its audio URL |
| `list_speech` | Recover recent jobs after an interrupted response |

Generation is asynchronous. Save the returned `speech_...` ID and poll until
`success` or `failed`, waiting at least the returned `poll_after_seconds`.
Never submit again just because the job is pending. A lost submit response is
ambiguous: inspect recent jobs before deciding whether another paid call is needed.
There is no automatic deduplication or idempotency key. Reuse downloaded clips
when narration and performance settings have not changed.

## Example: request to rendered video

In a Remotion project, ask:

> /natiq-voiceover:voiceover Make a two-scene vertical promo for a local delivery
> service in conversational Algerian Darija. Use a consistent voice, check my
> balance, save the clips in public/audio, and time the video to the actual audio.
> Show scene captions; label their timing as estimated. Render out/promo.mp4.

Claude discovers the catalogue and balance, writes the narration, then calls
`generate_speech` once per scene with the returned voice ID. For example:

```json
{"text":"راك حاب توصّل طلباتك بلا ما تتقلّق؟","voice":"VOICE_ID_FROM_LIST_VOICES"}
```

For each returned ID it calls `get_speech` with `{"speech_id":"RETURNED_JOB_ID"}`.
After success, it downloads the returned URL without any authorization header:

```bash
mkdir -p public/audio
curl --fail --location --output public/audio/scene-01.wav 'RETURNED_AUDIO_URL'
```

Download scene 2 the same way. Save `scenes.json` at the project root with the
**actual** durations and text from the results. This is an illustrative shape;
the numbers below are examples, not measured Natiq output:

```json
[
  {"id":"scene-01","text":"راك حاب توصّل طلباتك بلا ما تتقلّق؟","audio":"audio/scene-01.wav","duration_seconds":3.42,"pauseFrames":6},
  {"id":"scene-02","text":"خلّي التوصيل علينا، وركّز على خدمتك.","audio":"audio/scene-02.wav","duration_seconds":3.81,"pauseFrames":6}
]
```

Also preserve the job IDs and generation settings in your working scene manifest.
Keep keys and temporary signed URLs out of both files. In an existing Remotion 4
project, this minimal `src/index.tsx` demonstrates the complete timing chain:

```tsx
import React from 'react';
import {AbsoluteFill, Composition, Html5Audio, Series, registerRoot, staticFile} from 'remotion';
import scenes from '../scenes.json';

const fps = 30;
const frames = (s: typeof scenes[number]) => Math.ceil(s.duration_seconds * fps) + s.pauseFrames;
const Promo: React.FC = () => <Series>{scenes.map(s => (
  <Series.Sequence key={s.id} durationInFrames={frames(s)}>
    <AbsoluteFill style={{backgroundColor: '#162b25', color: 'white',
      justifyContent: 'center', alignItems: 'center', padding: 80, fontSize: 60,
      direction: 'rtl', textAlign: 'center'}}>{s.text}</AbsoluteFill>
    <Html5Audio src={staticFile(s.audio)} />
  </Series.Sequence>
))}</Series>;

const Root: React.FC = () => <Composition id="Promo" component={Promo}
  width={1080} height={1920} fps={fps}
  durationInFrames={scenes.reduce((sum, s) => sum + frames(s), 0)} />;
registerRoot(Root);
```

Keep your existing entry point and compositions when integrating this example.
Preview, listen for cut-off words, check Arabic shaping and music balance, then:

```bash
npx remotion render src/index.tsx Promo out/promo.mp4
```

For a new project, first follow [Remotion's setup guide](https://www.remotion.dev/docs/).
The sample assumes the existing project's Remotion/React dependencies and an
Arabic-capable font. Caption display is scene-level and **estimated**, not word
alignment. Natiq provides total clip duration, not word timestamps. Accurate
karaoke highlighting needs a separate alignment step.

## Privacy and contributions

Your narration and generation settings are sent to Natiq. Audio URLs may grant
temporary access to private audio; download locally before rendering and do not
commit those URLs. The remote server cannot write files on your machine.

New video workflow skills can live in `skills/<name>/SKILL.md`; the `natiq-tools`
marketplace can list additional plugins later. Keep language coaching separate
from this voiceover integration. No additional Darija coaching skill is included.

Validate changes with `claude plugin validate .` and
`claude plugin validate .claude-plugin/plugin.json`. Format follows the official
[plugin reference](https://code.claude.com/docs/en/plugins-reference),
[marketplace guide](https://code.claude.com/docs/en/plugin-marketplaces), and
[MCP configuration docs](https://code.claude.com/docs/en/mcp).
