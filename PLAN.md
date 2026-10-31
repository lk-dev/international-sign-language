# International Sign Language — Development Plan

Open source toolkit for **speech/text → sign language** (3D signing avatars) and **sign language → text** (camera and video recognition), usable as an app, an API, and a **broadcast/web overlay**.

Status: planning (2026-10-31). Actionable checklist: [TODO.md](TODO.md). Technical conventions: [docs/design/engine.md](docs/design/engine.md). Research notes: [docs/research/](docs/research/).

---

## 1. Vision and scope

| Product | What it does | Who uses it |
|---|---|---|
| **Playground app** | Every feature testable in one web app: translate, camera recognition, video tools, learning, overlay preview, API console | everyone, demos, QA |
| **Overlay** | Transparent signing avatar driven by live audio, live captions or an API channel | TV broadcasters, streamers (OBS), websites, video players |
| **API + SDKs** | Text → sign timeline / rendered video, live channels, recognition, lexicon | developers, media companies |
| **Learn** | Lessons, fingerspelling practice with camera feedback, quizzes | learners, hearing families of Deaf children |
| **Studio** | Authoring tool for signs: timeline editor, webcam capture, mocap import, review workflow | lexicon contributors, Deaf reviewers |
| **Model training** | Dataset collection (with consent), training pipelines, released recognition models | ML contributors |

### Non-goals (stated publicly)

- We do not replace human interpreters. The avatar adds accessibility where none exists, for example at 3 a.m., in niche content or in announcements. Docs and marketing must say this clearly.
- We do not claim accuracy for unreviewed content. Every sign carries a review status (`draft` / `community` / `verified`).

### Principles

1. **Deaf-led:** native signers author, review and test. A Deaf advisory board decides on each language's quality.
2. **Privacy by default:** camera and video processing run on the device. Nothing leaves the browser unless the user explicitly contributes data.
3. **One engine everywhere:** the same TypeScript/WebGL code runs in the app, the overlay, OBS, CasparCG, websites and mobile wrappers.
4. **Open formats:** sign notation, timelines and datasets are documented and versioned, with HamNoSys/SiGML import and export.

---

## 2. Languages

There is no universal sign language. Ethnologue lists ~150–160 sign languages and the WFD estimates ~300. Each language needs its own fingerspelling, lexicon, grammar rules, mouthing language and speech-recognition locale.

| Tier | Languages | Target |
|---|---|---|
| **1 (MVP)** | ASL (US), BSL (UK), LSF (France), DGS (Germany), LSE (Spain), Libras (Brazil), PJM (Poland), International Sign (IS) | fingerspelling + starter lexicon (~40 signs) + text→sign |
| **2** | LIS (Italy), NGT (Netherlands), Auslan (AU), LSM (Mexico), ISL (India, two-handed alphabet), JSL (Japan, kana fingerspelling), CSL (China), KSL (Korea), RSL (Russia), USL (Ukraine), TİD (Turkey), LSQ (Quebec) | fingerspelling + lexicon via community |
| **3** | everything else, through the Studio + community workflow | |

Per-language package (`languages/<code>/`):

- `meta.json`: name, ISO 639-3 code, region, spoken languages for mouthing/ASR (`en-US`…), writing system, reviewers;
- `alphabet.ts`: manual alphabet (one-handed, two-handed or kana), plus dynamic letters (PJM Ą, Ć… CH, CZ; ASL J, Z);
- `lexicon/*.json`: signs in the notation format (§4.2), each with sources, license, review status and reviewer;
- `grammar.ts`: ordering rules (time–topic–comment, WH position, negation), non-manual markers, number/date signing;
- `numbers.ts`: number system (many languages have special handshapes for 1–10, 11–19 and money/age/time incorporation);
- `tests/`: golden translations and recognition fixtures.

The name "International Sign" is an actual contact language (IS) **and** sounds like our project name. Docs must distinguish the **project** from the **IS language**. Decide on a product name before launch (see §17).

---

## 3. Architecture

### 3.1 Monorepo layout (npm workspaces, TypeScript, ESM, Node ≥ 26)

```
packages/
  notation/      sign format: types, JSON schema, validator, HamNoSys/SiGML import/export
  engine/        text → gloss → timeline; pose sampling; FK of hands; no rendering deps
  avatar-web/    three.js renderer: rigs, IK, face, materials, avatar loader (glTF/VRM)
  recognition/   MediaPipe adapters, features, templates, kNN/DTW, ONNX model runner
  asr/           Web Speech adapter, Whisper (transformers.js) worker, server ASR client
  translate/     rule-based + LLM translation, lexicon-constrained output
  sdk-js/        API client + <sign-avatar> web component + embed script
apps/
  web/           playground (all features)
  overlay/       overlay page (transparent/green/black bg, URL params)
  server/        HTTP API, SSE/WebSocket live channels, static hosting
  studio/        sign authoring + review
  mobile/        later: PWA first, then native shell (§12)
integrations/
  obs/  casparcg/  hbbtv/  videojs/  nginx-rtmp/  ndi/
ml/              Python: data prep, training, evaluation, ONNX export (PyTorch)
languages/       per-language packages (§2)
avatars/         avatar assets (glTF/VRM + textures + licenses)
docs/            design, API (OpenAPI), guides, research
```

### 3.2 Data flow

```
speech ─► ASR ─► text ─► normalize ─► translate (text→gloss) ─► plan (gloss→timeline) ─► sample(t) ─► avatar
captions/teletext/API text ─────────┘                                                     │
                                                                           overlay / app / video export
camera/video ─► MediaPipe landmarks ─► features ─► recognizer (templates/kNN/DTW/ONNX) ─► glosses ─► text
```

### 3.3 Technology decisions

| Area | Choice | Why |
|---|---|---|
| Language | TypeScript (strict, `erasableSyntaxOnly`), Node runs `.ts` natively | one language for engine, browser and server |
| Rendering | three.js (WebGL2; WebGPU renderer when stable on target browsers) | runs in OBS/CasparCG (Chromium), all browsers, mobile |
| Build | Vite | fast, static output |
| Server | plain `node:http` (no framework) + SSE; `ws` only if bidirectional needed | small surface, easy to self-host |
| Landmarks | MediaPipe Tasks Vision (Hand + Pose, later Holistic/Face blendshapes) | on-device, Apache-2.0, fast |
| ASR | Web Speech API (live, where available) + Whisper via transformers.js (files, offline) + optional server Whisper | no server cost by default |
| ML training | Python + PyTorch → ONNX → ONNX Runtime Web | standard tooling, runs in the browser |
| Mobile | PWA first; native shell later (Flutter or Capacitor wrapping the web engine) | Flutter cannot reuse WebGL engine natively; avoid two engines |
| Tests | `node --test` for engine/server; headless Chrome screenshots for visual regression | no Docker on dev machines |

**Why not Flutter for the core:** broadcast graphics (CasparCG HTML, OBS browser source), websites and players all need a web engine. A Flutter engine would be a second implementation. Flutter (or Capacitor) is fine as a mobile shell around the web engine.

---

## 4. Speech/text → sign

### 4.1 Pipeline stages

1. **ASR.** Live: Web Speech API (`continuous`, `interimResults`) with the spoken locale of the chosen sign language. Files/video: Whisper (`whisper-base`/`small`, word timestamps) in a Web Worker on WebGPU/WASM. Server option: faster-whisper behind a queue (heavy, only on owner's decision; see §13).
2. **Normalization.** Numbers, dates, times, currency, abbreviations and URLs become signable tokens. Remove disfluencies. Split into sentences.
3. **Translation text → gloss** (language-specific grammar, not word-for-word):
   - v1 rule-based: lexicon lookup (lemma + synonyms per spoken language), unknown word → fingerspell, question detection → non-manual markers;
   - v2 LLM: few-shot, **constrained output** (JSON schema; glosses must exist in the lexicon or be `FS:word`), grammar notes per language in the prompt, cached per sentence. Pluggable provider (hosted LLM or an open model), off by default for self-hosters;
   - v3 trained text→gloss models where parallel corpora exist (PHOENIX-2014T DGS, How2Sign ASL, BOBSL BSL; mind licenses, §8.3).
4. **Planning gloss → timeline.** Look up each sign. Time-scale it to the target speed. Insert transitions (coarticulation: the end of sign A blends into the start of sign B with minimal-jerk paths). Add sentence-level non-manuals: yes/no question = brows up + head forward, WH question = brows down, negation = head shake, topic = brows up + pause. Add mouthing visemes from the spoken word. Add holds at sentence ends.
5. **Sampling.** `sample(t) → Pose` (both arms, both hands, face, head, torso, gaze) at render rate.
6. **Live latency budget:** ASR partials ~0.5–1 s; we wait for a stable phrase, then translate, so expect 2–6 s total. Overlay shows a "catching up" state and drops or condenses content if it falls more than N seconds behind (configurable: `maxLag`).

### 4.2 Sign notation (open format, `packages/notation`)

Inspired by HamNoSys/SiGML but keyframe-based and machine-friendly. Defined formally in [docs/design/engine.md](docs/design/engine.md).

- **Sign** = id, gloss, language, sources, license, review status, `hands` (`1` / `2-sym` / `2-base` / `2-asym`), keyframes, non-manual track, mouthing (word or visemes), duration hint.
- **Keyframe (per hand)** = time, handshape (named or explicit joint angles), location (body landmark or other-hand landmark + offset), contact point of the hand, finger direction, palm direction, path type to the next key (`line`/`arc`/`circle`/`zigzag`), repetition, local movement (`wiggle`, `twist`, `flick`).
- **Coordinates:** signer-relative, dominant-side mirrored. A sign authored for a right-handed signer plays correctly left-handed with a flag.
- **HamNoSys/SiGML import:** reuse existing corpora (DGS-Korpus, DictaSign) where licenses allow.

### 4.3 Grammar features roadmap

| Phase | Features |
|---|---|
| A | lexical signs, fingerspelling, question/negation non-manuals, mouthing |
| B | numbers and incorporation (age, time, money), time-first ordering, WH placement per language |
| C | spatial referencing (place referents at loci, point/look back), agreement verbs (GIVE moving between loci) |
| D | classifiers (vehicle, person, flat object) driven by an LLM "scene" description, role shift (body + gaze turn) |
| E | prosody: speed, emphasis, emotion matching speech tone |

---

## 5. Avatar and animation

### 5.1 Rig

- Humanoid skeleton (VRM/glTF humanoid naming): hips, spine ×3, neck, head, shoulders/clavicles, upper/lower arms, hands, **15 finger joints per hand** (thumb CMC/MCP/IP plus 3 per finger) and forearm twist bones.
- Face: **ARKit 52 blendshapes** standard (brows, eyes, jaw, mouth, cheeks, tongue). Extra visemes as blendshapes. Signing needs: brow raise/furrow, eye widen/squint, mouth morphemes ("mm", "th", "cha", "pah", "puff cheeks"), mouthing visemes.
- Procedural fallback avatar (built from primitives in code) for tests and low-end devices.

### 5.2 Motion

| Problem | Solution |
|---|---|
| Hand placement | two-bone analytic IK (shoulder–elbow–wrist) with elbow pole + clavicle assist for high/contralateral targets |
| Wrist | target orientation from finger + palm direction; clamp to joint limits; distribute twist into forearm twist bones |
| Contacts | contact point (fingertip/palm/side) placed at body landmark with skin offset; per-avatar landmark calibration |
| Collisions | capsule collision of hands vs. torso/head/other hand; push-out pass after IK |
| Transitions | minimal-jerk interpolation, quaternion slerp for orientations, handshape blend with per-joint timing (fingers lead the arm) |
| Liveliness | breathing, blinks (Poisson), micro head/torso sway, gaze to hands during fingerspelling, to viewer otherwise |
| Speed | global speed control 0.5–1.5×; fingerspelling rate per language norm |
| Left-handed | mirror all tracks |

### 5.3 Avatars ("several realistic characters")

| Track | How | Notes |
|---|---|---|
| Stylized | procedural or low-poly glTF, flat/toon shading | MVP, fast on phones |
| Realistic 3D | CC0/CC-BY characters (MPFB/MakeHuman CC0 export, custom sculpted), PBR skin with subsurface approximation, hair cards, 4–6 diverse characters (age, gender, skin tone) | MetaHuman license forbids non-Unreal use, so we can't use it in an open web engine |
| Photoreal neural (research) | pose → video model (diffusion/GAN) trained on consenting signers; server GPU; batch only | actors' explicit consent and revenue share; watermark "AI avatar" |

Avatar package format: `avatars/<id>/avatar.glb` + `avatar.json` (landmark calibration, license, author, body metrics) + thumbnail. Skeleton + blendshape validation script.

### 5.4 Rendering targets

- App/web: 30–60 fps, DPR cap, shadows optional.
- Overlay: alpha channel (`premultipliedAlpha`), green/black key backgrounds, safe-area positioning (`pos=br`, `scale`, `margin`), optional rounded "interpreter box".
- Export: WebM VP9 with alpha (MediaRecorder / WebCodecs), PNG sequence, ProRes 4444 via server ffmpeg (batch).

---

## 6. Sign → text (recognition)

### 6.1 Input

- **Camera:** MediaPipe HandLandmarker (2 hands, world landmarks) + PoseLandmarker (shoulders/face reference). Later HolisticLandmarker + face blendshapes for non-manuals.
- **Video file:** same models in VIDEO mode, frame stepping, results → timed captions (SRT/VTT).
- Handedness: MediaPipe labels assume a mirrored image; raw camera/video frames have them inverted. Detect and expose a toggle.

### 6.2 Features

- Handshape: 21 landmarks → translate to the wrist, rotate into the hand frame (wrist→middle-MCP = y, index-MCP→pinky-MCP = x), scale by palm length, mirror left hands. This gives a 60-dim, orientation-invariant vector.
- Orientation: palm normal + finger direction in camera space.
- Location/movement: wrist positions relative to the shoulder midpoint, scaled by shoulder width; velocities; sequences resampled to fixed length.

### 6.3 Recognizer stages

| Stage | Method | Data |
|---|---|---|
| R1 static fingerspelling | nearest-neighbour vs. **synthetic templates generated by the engine's hand FK** (the avatar "teaches" the recognizer) + user samples | zero-shot + personal |
| R2 dynamic letters & isolated signs | DTW-kNN over feature sequences; engine-synthesized trajectories for every lexicon sign + user samples | zero-shot + personal |
| R3 trained isolated-sign model | landmark transformer / ST-GCN → ONNX | WLASL, MS-ASL, AUTSL, community data (§8) |
| R4 continuous recognition | CTC over landmark sequences, gloss LM | corpora + community |
| R5 sign → spoken-language text | gloss-free translation (landmark encoder + LLM decoder) | research |

Text output: glosses → readable sentence (rule-based joins for R1–R2, LLM rewrite as an option, clearly marked).

Metrics: top-1/top-5 accuracy (isolated), WER (continuous), BLEU/chrF + human eval (translation), latency p95, signer-independent splits.

---

## 7. Video tools

| Mode | Flow | Output |
|---|---|---|
| Speech in video → sign | decode audio (WebAudio), Whisper with timestamps, translate per segment, timeline aligned to video time | live preview overlay on the player; "sign track" JSON sidecar; rendered overlay video (alpha) |
| Sign in video → text | MediaPipe VIDEO mode + recognizer | SRT/VTT captions, transcript |
| Batch (server) | upload/URL → queue → render | for media companies via API; heavy, see §13 |

Sign track sidecar format (`.signtrack.json`): versioned, references language + lexicon version, contains timed glosses and optionally baked keyframes, so any player can render it with any avatar. This is the "subtitle file for sign language".

---

## 8. Learning, data and model training ("model nauki")

### 8.1 Learn mode (for people)

- Courses per language: alphabet, numbers, top-100 signs, phrases.
- Avatar demonstrates; user repeats on camera; recognizer scores handshape/orientation/movement with per-component feedback.
- Spaced repetition, quizzes (avatar signs → choose word; word → you sign), progress stored locally (IndexedDB), optional account sync later.

### 8.2 Train mode (for the model)

- Record samples per label (letter/sign) from the camera. Only landmarks are stored, never video. Train a personal kNN instantly in the browser. Export/import datasets as JSON.
- **Contribute** (opt-in): consent screen (purpose, license CC BY 4.0, withdrawal), demographics optional. Samples go to the server as landmark sequences with consent ID. A review queue lets Deaf moderators validate labels.
- Server stores contributions in the app data volume. Periodic dataset releases (versioned, with datasheet). Deletion on request via contributor token.

### 8.3 External datasets: license matrix (must check before use)

| Dataset | Language | Typical license | Usable for released weights? |
|---|---|---|---|
| WLASL | ASL | C-UDA (computational use) | check terms; likely research only |
| MS-ASL | ASL | research | no |
| How2Sign | ASL | CC BY-NC 4.0 | non-commercial only |
| AUTSL | TİD | research | no |
| PHOENIX-2014T | DGS | non-commercial research | no |
| BOBSL | BSL | research licence | no |
| DGS-Korpus | DGS | CC BY-NC (public part) | non-commercial only |
| Corpus PJM (UW) | PJM | ask the lab | ask |

Policy: models released under the project license are trained **only** on data compatible with it (own/community CC BY data). Research models on NC data are published separately with NC terms or not at all.

### 8.4 Training pipeline (`ml/`)

- Python 3.13+, PyTorch, reproducible configs, data versioning (manifest + hashes).
- Synthetic augmentation from the engine: render lexicon signs with random avatars, speeds and noise into landmark space.
- Export ONNX (fp16/int8), ship via CDN with SHA-256 pinning; model cards.
- GPU: the Coolify VM has no GPU. Train on rented GPUs on demand, only on the owner's decision. Inference runs in the browser.

---

## 9. Broadcast, streaming and web overlay

| Target | Integration | Status target |
|---|---|---|
| OBS | Browser Source with `overlay?bg=transparent&channel=…` | M2 |
| CasparCG | HTML template (`integrations/casparcg/`), key+fill SDI out | M7 |
| vMix / Wirecast | browser input / NDI | M7 |
| NDI with alpha | headless renderer → NDI SDK (native helper) | M7 |
| Live captions feed | ingest EBU-TT-D / WebVTT / SRT stream / teletext (via existing subtitle system) into a channel | M7 |
| HbbTV | HbbTV app toggles the sign overlay on the broadcast (red button) | M7 |
| Web players | video.js / hls.js / `<video>` plugin; reads `.signtrack.json` or live channel | M4 |
| YouTube/VOD | rendered alternate version with burned-in avatar | M4 |

Live operator console (in the app): choose language, audio input, speech recognition → publishes to a channel; shows lag, allows manual text injection and emergency stop.

Synchronisation: for VOD align exactly. For live, show the honest delay. Broadcasters can optionally delay the programme feed.

Standards and law to track: WCAG 2.2 SC 1.2.6 (sign language for prerecorded content), European Accessibility Act (applies from June 2025), EBU accessibility guidelines, national rules (e.g., Poland: Act on sign language 2011, KRRiT accessibility quotas). Regulators generally expect human interpreters, so position the avatar as an addition.

---

## 10. API and SDKs

- REST `/v1`, JSON, OpenAPI 3.1 spec in `docs/api/openapi.yaml`, generated reference page.
- Endpoints (v1):
  - `GET /api/health`
  - `GET /v1/languages`, `GET /v1/languages/{code}/lexicon`
  - `POST /v1/translate` `{text, lang, speed?}` → glosses + timeline (render with any avatar)
  - `POST /v1/live/{channel}` publish text; `GET /v1/live/{channel}/events` SSE subscribe
  - `POST /v1/samples` contribute landmarks (consent required)
  - later: `POST /v1/jobs/render` (video → overlay video), `POST /v1/jobs/recognize`, webhooks
- Auth: anonymous tier with per-IP limits; API keys for higher quotas (hosted instance); live channels are unguessable capability IDs (≥128-bit) + optional publish key.
- Rate limits per real client IP (Cloudflare-aware XFF parsing, ranges in one file) and per key.
- SDKs: `@isl/sdk` (TS) with `<sign-avatar lang="pjm" text="…">` web component and `<script>` embed; Python client; examples (Next.js, plain HTML, OBS, CasparCG).
- Versioning: semver for packages; API breaking changes only in `/v2`; lexicon versions pinned in timelines.

---

## 11. Playground app ("everything testable")

Tabs:

1. **Translate:** language, avatar, speed, handedness; text box + mic; gloss strip with per-sign review status; camera orbit; replay a single sign.
2. **Camera:** live recognition (letters → text), landmark debug view, confidence, handedness toggle.
3. **Video:** upload; mode A speech → sign overlay on the player; mode B sign → captions; export.
4. **Learn:** lessons and practice with camera feedback.
5. **Train:** record samples, personal model, export/import, contribute.
6. **Overlay:** URL builder + live preview (transparent/green), operator console.
7. **Lexicon:** browse/search signs per language, play, status, sources.
8. **API:** interactive console with examples (curl/JS).

UI languages: English first, then Polish, then community translations (i18n JSON). WCAG 2.2 AA, keyboard-first, high contrast, no audio-only cues.

---

## 12. Mobile and desktop

- PWA (installable, offline language packs: lexicon + models cached).
- Native shell later: Capacitor or Flutter + WebView with the web engine; native camera for performance; app stores.
- Desktop: the overlay also packaged as a small Electron/Tauri app for operators without OBS (later).

---

## 13. Infrastructure and operations

Hosting on the owner's Coolify server, following the workspace checklist (`GitHub/CLAUDE.md`):

| Item | Value |
|---|---|
| App name / uuid | `international-sign-language`, own Coolify project |
| Repo | `lk-dev/international-sign-language`, branch `main`, GitHub App, autodeploy |
| Domains | `international-sign-language.178.63.23.182.nip.io` with `X-Robots-Tag: noindex, nofollow` (`nip-noindex.py`); real domain later behind Cloudflare (proxy, Full strict), `www` → non-www |
| Resources | 4 GB RAM (`limits_memory=4g`, swap 4g); no background jobs at first; batch render/training jobs later with hard CPU limit 0.5–1 core and `cpu_shares` 256–512 |
| Data | contributions in volume `international-sign-language-data`; no per-app backups (VM backed up daily) |
| Image | multi-stage Dockerfile, `USER node`, healthcheck `/api/health`, logs on stdout, only Traefik 80/443 |
| ENV | complete before first deploy; secrets locked (`is_shown_once`); temporary values marked `TODO_…` |
| Mongo/Mail | not needed initially; if needed, own Mongo account (`mongo-konto.py`), mail `noreply@<domain>` via Mailgun |
| Heavy work | ASR/recognition in the browser; server Whisper/render/training only on owner's decision |

Static model files (MediaPipe `.task`, ONNX, Whisper) are served from our origin/CDN with pinned hashes, not hot-linked at runtime.

Self-hosting for the open source community: `docker compose` example, `.env.example`, docs. The owner's own Mac never runs Docker or dev servers; verification happens on the server.

---

## 14. Security and privacy

- Camera/video never uploaded by default. Contributions contain landmarks only, with consent records; GDPR: landmarks of hands/face can be personal data → DPIA, retention policy, deletion endpoint, privacy notice.
- Real client IP per workspace rule 14 (XFF from the right, skip Cloudflare + private ranges). Limits per IP and per key; failed auth logged to stdout for CrowdSec.
- Input limits (text length, payload size, channel count, SSE subscribers), CSP, no inline scripts, `Permissions-Policy` for camera/mic only where needed.
- Supply chain: lockfile, `npm audit`, Dependabot/Renovate, model hash pinning, SBOM on release.
- LLM translation: prompt-injection-safe (output must validate against lexicon schema), no user data retained by providers (configure), opt-in.
- SECURITY.md with disclosure process.

---

## 15. Quality and testing

| Layer | Tests |
|---|---|
| notation | schema validation of every lexicon file; lint (missing sources/status) |
| engine | unit tests (planner, timing, mirroring, FK); golden timelines per language |
| animation | IK math tests; **visual regression**: headless Chrome screenshots of key poses (every letter of every alphabet) compared with approved baselines |
| recognition | synthetic round-trip (engine renders letter → landmarks → recognizer must return the same letter); benchmark on held-out real samples |
| server | HTTP tests with `node --test` (no Docker), rate-limit and SSE tests |
| e2e | on the deployed instance: smoke checks via headless browser |
| human | **Deaf comprehension tests** per language each milestone (target ≥ 80% sign comprehension for verified lexicon), naturalness ratings |
| performance | 60 fps desktop / 30 fps mid-range phone; initial JS < 1.5 MB gz (models lazy) |

CI: GitHub Actions on the public repo (public repos get free standard runners, which solves the exhausted private budget). Checks: typecheck, tests, lexicon lint, build.

---

## 16. Open source governance

- **License proposal:** code Apache-2.0 (patent grant, business-friendly); avatars/animations/lexicon CC BY 4.0; community datasets CC BY 4.0 with consent; model weights licensed per training data (§8.3).
- Files: LICENSE, NOTICE, CONTRIBUTING.md, CODE_OF_CONDUCT.md (Contributor Covenant), SECURITY.md, GOVERNANCE.md, issue/PR templates, DCO sign-off (lighter than a CLA). DCO sign-off lines in commits are the contributors' own; maintainers' commits follow repo conventions.
- Lexicon contribution flow: PR with sign JSON + source + video reference (link, not upload) → automatic lint + rendered preview GIF → review by a native signer of that language → status `verified` with reviewer handle.
- Language maintainers per language (Deaf signers preferred), Deaf advisory board with veto on quality/marketing claims.
- Public roadmap (GitHub Projects), releases with changelog, semver.
- Partnerships: WFD, EUD, national associations (e.g., PZG in Poland), universities (UW Sign Linguistics Lab for PJM, Hamburg DGS-Korpus), broadcasters (accessibility departments).
- Funding: NLnet / NGI Zero (open source grants), EU Horizon/Digital Europe accessibility calls, national disability funds (e.g., PFRON), sponsors (GitHub Sponsors/Open Collective). Hosted API for media as sustainability option.

---

## 17. Milestones

| ID | Name | Deliverable |
|---|---|---|
| **M0** | Foundation | public repo, licenses, governance files, monorepo skeleton, notation spec v0, CI, Coolify app with health page |
| **M1** | Avatar + fingerspelling | procedural avatar with full hands + face, IK, 8 manual alphabets, Translate tab (text → fingerspelling), visual regression baselines |
| **M2** | Text/speech → sign MVP | starter lexicons (~40 signs × 8 languages, draft), rule-based translation, non-manuals, mouthing, mic (Web Speech), overlay page + live channels (SSE), API v1 + OpenAPI, deploy |
| **M3** | Recognition + learning | camera fingerspelling recognition (synthetic templates + kNN), Learn tab, Train tab (personal model, export), contribution endpoint with consent |
| **M4** | Video tools | Whisper in browser, speech-in-video → sign overlay, sign-in-video → captions, `.signtrack.json`, player plugin, alpha video export |
| **M5** | Studio | sign editor (timeline, gizmos), webcam capture → keyframes, BVH/FBX mocap import, review workflow, HamNoSys/SiGML import |
| **M6** | Realistic avatars | 4–6 glTF characters with ARKit blendshapes, PBR skin, avatar validator, mocap session with Deaf signers |
| **M7** | Broadcast | operator console, CasparCG template, OBS guide, captions feed ingest, NDI helper, HbbTV demo, broadcaster pilot |
| **M8** | Trained recognition | isolated sign model (ONNX) on licensed/community data, benchmarks, model cards |
| **M9** | Better translation | LLM constrained translation, grammar phases B–C, Deaf panel evaluation per language |
| **M10** | Research | continuous recognition, sign → text translation, photoreal neural avatar (consent-based) |
| **M11** | Apps | PWA offline packs, native mobile shell, desktop overlay app |

---

## 18. Risks

| Risk | Mitigation |
|---|---|
| Low acceptance by Deaf community ("avatars are bad") | Deaf-led review, honest positioning, quality gates per language, show review status in UI |
| Wrong/inaccurate signs | `draft` status by default, native review required for `verified`, easy issue reporting from the app ("report this sign") |
| Dataset licenses block open models | own consented data collection from day one (Train → Contribute) |
| Live latency too high for TV | captions feed ingest (text available earlier), condensation, honest delay indicator |
| Realism cost (art, mocap) | stylized first; grants for mocap sessions; community avatars |
| Browser support (WebGPU, Web Speech in CEF) | WebGL2 baseline; operator console in Chrome publishes text, overlays only render |
| GDPR (biometric-like data) | on-device processing, landmarks only, consent, DPIA |
| Project name confusion with IS language | choose product name before public launch |

---

## 19. Decisions needed from the owner

1. License set (proposal in §16).
2. Product name/domain (project vs. International Sign language confusion).
3. LLM provider for v2 translation (hosted vs. open model, cost).
4. Budget for mocap sessions with Deaf signers and for GPU training.
5. First broadcaster/partner for a pilot.
