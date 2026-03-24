---
name: video_toolkit
description: Create professional videos autonomously using claude-code-video-toolkit — AI voiceovers, image generation, music, talking heads, and Remotion rendering.
metadata:
  openclaw:
    os: ["darwin", "linux"]
    requires:
      bins: ["node", "python3", "ffmpeg", "npm"]
---

# Video Toolkit — OpenClaw Skill

> **Status: Alpha.** This skill is designed for [OpenClaw](https://docs.openclaw.ai) and is experimental. It provides autonomous video creation using the [claude-code-video-toolkit](https://github.com/digitalsamba/claude-code-video-toolkit). Expect rough edges — contributions and feedback welcome.

Create professional explainer videos from a text brief. The toolkit uses open-source AI models running on cloud GPUs (Modal) for voiceover, image generation, music, and talking head animation. Remotion (React) handles composition and rendering.

## Setup (First Run)

Run the verification script to check current state:

```bash
python3 tools/verify_setup.py --json
```

If `summary.cloud_gpu` is false, you need to set up Modal. This is a one-time process.

### Install Modal CLI

```bash
pip3 install modal
python3 -m modal setup   # Opens browser for authentication
```

### Install Python Dependencies

```bash
pip3 install -r tools/requirements.txt
```

### Deploy Tools to Modal

Each tool is a separate Modal app. Deploy them and capture the endpoint URLs:

```bash
# Deploy each tool — the output contains the endpoint URL
# Look for the line: "Created web endpoint for ... => https://...modal.run"

modal deploy docker/modal-qwen3-tts/app.py    # Speech (most important)
modal deploy docker/modal-flux2/app.py         # Image generation
modal deploy docker/modal-music-gen/app.py     # Background music
modal deploy docker/modal-sadtalker/app.py     # Talking head narrator
modal deploy docker/modal-image-edit/app.py    # Image editing
modal deploy docker/modal-upscale/app.py       # Image upscaling
modal deploy docker/modal-propainter/app.py    # Watermark removal
```

After each deploy, Modal prints a URL like:
```
https://username--video-toolkit-flux2-flux2-generate.modal.run
```

### Save Endpoint URLs to .env

Create or update `.env` in the toolkit root with the URLs from each deploy:

```bash
# Create .env if it doesn't exist
touch .env

# Add each endpoint URL (replace with actual URLs from deploy output)
echo 'MODAL_QWEN3_TTS_ENDPOINT_URL=https://username--video-toolkit-qwen3-tts-ttsengine-generate.modal.run' >> .env
echo 'MODAL_FLUX2_ENDPOINT_URL=https://username--video-toolkit-flux2-flux2-generate.modal.run' >> .env
echo 'MODAL_MUSIC_GEN_ENDPOINT_URL=https://username--video-toolkit-music-gen-musicgen-generate.modal.run' >> .env
echo 'MODAL_SADTALKER_ENDPOINT_URL=https://username--video-toolkit-sadtalker-sadtalkergen-generate.modal.run' >> .env
echo 'MODAL_IMAGE_EDIT_ENDPOINT_URL=https://username--video-toolkit-image-edit-qwenedit-edit.modal.run' >> .env
echo 'MODAL_UPSCALE_ENDPOINT_URL=https://username--video-toolkit-upscale-upscaler-upscale.modal.run' >> .env
echo 'MODAL_DEWATERMARK_ENDPOINT_URL=https://username--video-toolkit-dewatermark-dewatermark-dewatermark.modal.run' >> .env
```

### Optional: Cloudflare R2 (Recommended)

R2 provides reliable file transfer between your machine and cloud GPUs. Free tier: 10GB storage, zero egress. Without it, tools fall back to free upload services (slower, less reliable).

If the user has R2 credentials, add to `.env`:
```
R2_ACCOUNT_ID=...
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
R2_BUCKET_NAME=video-toolkit
```

### Verify Setup

Run verification again to confirm everything is configured:

```bash
python3 tools/verify_setup.py
```

All tools should show as configured. Modal apps scale to zero when idle — no ongoing cost.

**Cost:** Modal Starter plan includes $30/month free compute. A typical 60s video costs $1-3 to produce.

## End-to-End Workflow

### Step 1: Create Project

```bash
cp -r templates/product-demo projects/{project-name}
cd projects/{project-name}
npm install
```

Templates available: `product-demo` (marketing/explainer), `sprint-review` (sprint demos), `sprint-review-v2` (composable scenes).

### Step 2: Write the Config

Edit `src/config/demo-config.ts`. The config drives everything:

```typescript
export const demoConfig: ProductDemoConfig = {
  product: {
    name: 'My Product',
    tagline: 'What it does in one line',
    website: 'example.com',
  },
  scenes: [
    { type: 'title', durationSeconds: 9, content: { headline: '...', subheadline: '...' } },
    { type: 'problem', durationSeconds: 14, content: { headline: '...', problems: [...] } },
    { type: 'solution', durationSeconds: 13, content: { headline: '...', highlights: [...] } },
    { type: 'stats', durationSeconds: 12, content: { stats: [...] } },
    { type: 'cta', durationSeconds: 10, content: { headline: '...', links: [...] } },
  ],
  audio: {
    backgroundMusicFile: 'audio/bg-music.mp3',
    backgroundMusicVolume: 0.12,
  },
};
```

Scene types: `title`, `problem`, `solution`, `demo`, `feature`, `stats`, `cta`.

**Duration rule:** Set `durationSeconds` to `ceil(voiceover_duration + 2)`. You won't know the exact voiceover duration until Step 4, so estimate at ~2.5 words/second, then adjust after generating audio.

### Step 3: Write the Voiceover Script

Create `VOICEOVER-SCRIPT.md` with one section per scene:

```markdown
## Scene 1: Title (9s, ~15 words)
Build videos with AI. This is the product name toolkit.

## Scene 2: Problem (14s, ~30 words)
The problem statement goes here. Keep it punchy and relatable.
```

**Word budget:** `(durationSeconds - 2) * 2.5` words per scene. The -2 accounts for the 1s audio delay and 1s padding.

### Step 4: Generate Assets

**All tool commands must be run from the toolkit root directory** (not from inside the project). This is critical.

```bash
cd /path/to/claude-code-video-toolkit
```

#### 4a. Background Music (ACE-Step)

```bash
python3 tools/music_gen.py --preset corporate-bg --duration 90 --output projects/{name}/public/audio/bg-music.mp3 --cloud modal
```

Presets: `corporate-bg`, `upbeat-tech`, `ambient`, `dramatic`, `tension`, `hopeful`, `cta`, `lofi`.

#### 4b. Voiceover (Qwen3-TTS)

Generate per-scene audio files:

```bash
# For each scene:
python3 tools/qwen3_tts.py \
  --text "The voiceover text for this scene" \
  --speaker Ryan \
  --tone warm \
  --output projects/{name}/public/audio/scenes/01.mp3 \
  --cloud modal
```

Speakers: `Ryan`, `Aiden`, `Vivian`, `Luna`, `Aurora`, `Aria`, `Nova`, `Stella`, `Orion`.
Tones: `neutral`, `warm`, `professional`, `excited`, `calm`, `serious`.

For voice cloning (if the user has a reference recording):
```bash
python3 tools/qwen3_tts.py \
  --text "..." \
  --ref-audio assets/voices/reference.m4a \
  --ref-text "Transcript of the reference audio" \
  --output projects/{name}/public/audio/scenes/01.mp3 \
  --cloud modal
```

#### 4c. Scene Images (FLUX.2)

Generate background images for key scenes:

```bash
python3 tools/flux2.py \
  --prompt "Dark tech background with blue geometric grid, cinematic lighting" \
  --width 1920 --height 1080 \
  --output projects/{name}/public/images/title-bg.png \
  --cloud modal
```

#### 4d. Presenter Portrait + Talking Head (FLUX.2 + SadTalker)

Generate a presenter portrait, then animate it with SadTalker for narrator PiP:

```bash
# Generate portrait
python3 tools/flux2.py \
  --prompt "Professional presenter portrait, clean style, dark background, facing camera, upper body" \
  --width 1024 --height 576 \
  --output /tmp/presenter.png \
  --cloud modal

# Generate per-scene narrator clips (NOT one big file — keep them short)
python3 tools/sadtalker.py \
  --image /tmp/presenter.png \
  --audio projects/{name}/public/audio/scenes/01.mp3 \
  --preprocess full --still --expression-scale 0.8 \
  --output projects/{name}/public/narrator-01.mp4 \
  --cloud modal
```

**Critical SadTalker notes:**
- Always use `--preprocess full` (preserves dimensions, default `crop` outputs square)
- Always use `--still` for professional look (reduces head movement)
- Generate per-scene clips (6-15s each), NOT one long video
- Processing time is ~3-4 min per 10s of audio on Modal A10G
- The `--expression-scale 0.8` keeps expressions subtle

#### 4e. Image Editing (Qwen-Edit) — optional

Create scene variants from existing images:

```bash
python3 tools/image_edit.py \
  --input projects/{name}/public/images/title-bg.png \
  --prompt "Make it darker with red tones, more ominous" \
  --output projects/{name}/public/images/problem-bg.png \
  --cloud modal
```

#### 4f. Upscaling (RealESRGAN) — optional

Upscale images for HD quality:

```bash
python3 tools/upscale.py \
  --input image.png --output image-4x.png --scale 4 --cloud modal
```

### Step 5: Sync Timing

After generating voiceover, check actual durations and update config:

```bash
# Check each audio file duration
for f in projects/{name}/public/audio/scenes/*.mp3; do
  echo "$(basename $f): $(ffprobe -v error -show_entries format=duration -of csv=p=0 "$f")s"
done
```

Update each scene's `durationSeconds` in `demo-config.ts` to: `ceil(audio_duration + 2)`.

### Step 6: Review

Render still frames at scene midpoints and inspect visually:

```bash
cd projects/{name}
npx remotion still src/index.ts ProductDemo --frame=100 --output=/tmp/review-title.png
npx remotion still src/index.ts ProductDemo --frame=400 --output=/tmp/review-problem.png
# ... etc for each scene
```

Check for:
- Text truncation or overflow
- Animation timing (are all elements visible?)
- Narrator PiP positioning
- Background image opacity/contrast

### Step 7: Preview & Render

```bash
cd projects/{name}

# Preview in browser
npm run studio

# Render final MP4
npm run render
```

Output lands in `out/ProductDemo.mp4`.

## Composition Patterns

### Per-Scene Audio

Don't use a single voiceover file. Use per-scene audio inside `TransitionSeries.Sequence`:

```tsx
<Sequence from={30}>
  <Audio src={staticFile('audio/scenes/01.mp3')} volume={1} />
</Sequence>
```

The `from={30}` adds a 1-second delay so scenes don't start talking immediately.

### Per-Scene Narrator PiP

```tsx
<Sequence from={30}>
  <OffthreadVideo
    src={staticFile('narrator-01.mp4')}
    style={{ width: 320, height: 180, objectFit: 'cover' }}
    muted
  />
</Sequence>
```

**Always use `<OffthreadVideo>`, never `<video>`.** Remotion requires its own video component for frame-accurate rendering.

### Transitions

Import custom transitions from `lib/transitions/presentations/` and Remotion's from `@remotion/transitions`:

```tsx
import { TransitionSeries, linearTiming } from '@remotion/transitions';
import { fade } from '@remotion/transitions/fade';
import { glitch } from '../../../lib/transitions/presentations/glitch';
import { lightLeak } from '../../../lib/transitions/presentations/light-leak';
```

Do NOT import `TransitionSeries` or `fade` from `lib/transitions` — it won't resolve.

## Cost Estimates (Modal)

| Tool | Typical Cost | Notes |
|------|-------------|-------|
| Qwen3-TTS | ~$0.01/scene | ~20s per scene on warm GPU |
| FLUX.2 | ~$0.01/image | ~3s warm, ~30s cold |
| ACE-Step | ~$0.02-0.05 | Depends on duration |
| SadTalker | ~$0.05-0.20/scene | ~3-4 min per 10s audio |
| Qwen-Edit | ~$0.03-0.15 | ~8 min cold start (25GB model) |
| RealESRGAN | ~$0.005/image | Very fast |

**Total for a 60s video:** ~$1-3 depending on number of scenes and narrator clips.

## Common Mistakes

1. **Running tools from project directory** — Always `cd` to toolkit root first
2. **Using `<video>` instead of `<OffthreadVideo>`** — Will not render correctly
3. **Importing transitions from `lib/transitions` barrel** — Import custom transitions from `lib/transitions/presentations/` directly
4. **One big SadTalker video** — Generate per-scene clips (6-15s), not one 60s file
5. **Not syncing timing after TTS** — Audio duration ≠ estimated duration. Always check and update config
6. **Missing 1s audio delay in duration math** — If audio starts at `from={30}`, add 1s to your duration calculation
