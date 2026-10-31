# TODO

Checklist derived from [PLAN.md](PLAN.md). Milestones in order; each item is small enough for one PR.

## Resume here

- [ ] Owner decisions: license set, product name, LLM provider, mocap/GPU budget (PLAN §19)
- [ ] Research in `docs/research/` was stopped early (partial lexicons only, no alphabets yet): finish it, then merge into the language packages
- [ ] Start M0

## M0 Foundation

- [ ] LICENSE (code), LICENSE-CONTENT (CC BY 4.0), NOTICE
- [ ] CONTRIBUTING.md (DCO, lexicon PR flow), CODE_OF_CONDUCT.md, SECURITY.md, GOVERNANCE.md
- [ ] Issue/PR templates: bug, feature, "wrong sign" (language, gloss, what's wrong, reference link)
- [ ] npm workspaces skeleton (`packages/*`, `apps/*`), strict tsconfig (`erasableSyntaxOnly`, `.ts` imports), latest stable deps
- [ ] `packages/notation`: types + JSON Schema v0 + validator CLI (`npm run lint:lexicon`)
- [ ] `node --test` setup, typecheck script, build script
- [ ] GitHub Actions (public repo): typecheck, tests, lexicon lint, build
- [ ] `apps/server` minimal: `/api/health`, static hosting, stdout logs, client IP helper (Cloudflare ranges in one file)
- [ ] Dockerfile (multi-stage, `USER node`, `HEALTHCHECK`), `.env.example`, compose example for self-hosters
- [ ] Coolify: project + app `international-sign-language` (uuid = name), autodeploy from `main`, 4 GB limits, full ENV, nip.io + `nip-noindex.py`, volume `international-sign-language-data`

## M1 Avatar + fingerspelling

- [ ] `engine`: vector/quaternion math, signer frame ↔ world, direction parser
- [ ] `engine`: hand FK (21 landmarks) per [engine.md](docs/design/engine.md) + named handshape library (A…Y, 5, C, O, claw, bent, baby-O, 8, 3, ILY…)
- [ ] `avatar-web`: procedural avatar (torso, head, arms, hands with 15 joints each, face: brows, eyelids, eyes, mouth, cheeks)
- [ ] `avatar-web`: two-bone IK with pole + clavicle assist, wrist orientation from (f, p), joint limits
- [ ] Idle layer: breathing, blinks, sway, gaze
- [ ] 4 avatar looks (skin tone, hair, clothing, body shape) on the same skeleton
- [ ] Alphabets: ASL, LSF, DGS, LSE, Libras, IS (one-handed), BSL (two-handed, other-hand landmarks), PJM incl. dynamic letters
- [ ] `apps/web` Translate tab: language + avatar selectors, text → fingerspelling, speed, replay, orbit camera
- [ ] Visual regression: headless Chrome screenshot of every letter → baselines (run on server/CI)
- [ ] Deploy + smoke check on nip.io

## M2 Text/speech → sign MVP

- [ ] Sign keyframe DSL helpers (repeat, circle, arc, wiggle, twist) + planner (time scaling, transitions, holds)
- [ ] Starter lexicons (~40 concepts × 8 languages), all `draft` with sources
- [ ] Rule-based translator: tokenize, normalize numbers/dates, lemma + synonyms lookup, fallback fingerspelling
- [ ] Sentence non-manuals: yes/no, WH, negation, topic
- [ ] Mouthing visemes from words (per spoken language)
- [ ] Mic input via Web Speech API (locale per sign language), interim text display
- [ ] `apps/overlay`: URL params (`lang`, `avatar`, `bg`, `pos`, `scale`, `channel`, `maxLag`), alpha rendering
- [ ] Live channels: `POST /v1/live/{channel}`, `GET /v1/live/{channel}/events` (SSE), limits (size, rate, channels, subscribers)
- [ ] API v1: `/v1/languages`, `/v1/languages/{code}/lexicon`, `/v1/translate`
- [ ] OpenAPI 3.1 spec + API tab in the app
- [ ] Lexicon tab: browse/search/play, review status, sources
- [ ] "Report this sign" link → prefilled GitHub issue

## M3 Recognition + learning

- [ ] `recognition`: MediaPipe Hand + Pose loaders (self-hosted wasm + `.task` with pinned hashes)
- [ ] Feature extraction (canonical handshape, orientation, wrist relative to shoulders)
- [ ] Synthetic templates from engine FK for every alphabet letter
- [ ] kNN static recognizer + stability filter (N frames) → letters → words
- [ ] Camera tab: live text, landmark debug overlay, handedness toggle, confidence
- [ ] Round-trip test: engine letter → landmarks → recognizer returns the letter
- [ ] DTW recognizer for dynamic letters and lexicon signs (synthetic trajectories)
- [ ] Learn tab: alphabet and top signs lessons, practice with camera scoring, quizzes, spaced repetition (IndexedDB)
- [ ] Train tab: record samples per label, personal model, export/import JSON
- [ ] Contribute: consent screen, `POST /v1/samples` (landmarks only), storage in data volume, deletion token, per-IP limits
- [ ] Privacy notice + DPIA draft

## M4 Video tools

- [ ] Whisper worker (transformers.js, WebGPU/WASM, word timestamps), model choice per device
- [ ] Video tab mode A: speech in video → timeline synced to playback, overlay on the player
- [ ] Video tab mode B: sign in video → SRT/VTT captions
- [ ] `.signtrack.json` format spec + reader/writer
- [ ] Export overlay as WebM with alpha (WebCodecs/MediaRecorder), PNG sequence
- [ ] Player plugin (`<video>`, video.js, hls.js) reading signtrack or live channel
- [ ] `<sign-avatar>` web component + embed script (`sdk-js`)

## M5 Studio

- [ ] Timeline editor (keys per hand, non-manual track), 3D gizmos for location/orientation, handshape picker
- [ ] Webcam capture → keyframes (MediaPipe → retarget → simplify)
- [ ] BVH/FBX mocap import + retarget to reference skeleton
- [ ] HamNoSys/SiGML import (and export)
- [ ] Review workflow: draft → community → verified (reviewer handle), PR generator
- [ ] Rendered preview GIF in lexicon PRs (CI)

## M6 Realistic avatars

- [ ] glTF/VRM loader with humanoid mapping + ARKit 52 blendshapes
- [ ] Avatar validator (skeleton, finger joints, blendshapes, license fields)
- [ ] 4–6 diverse characters (CC0/CC BY sources or commissioned), PBR skin, hair cards
- [ ] Collision pass (hands vs body, hand vs hand)
- [ ] Mocap session plan with Deaf signers (consent, payment, CC BY license)

## M7 Broadcast

- [ ] Operator console (language, audio input, lag meter, manual text, emergency stop)
- [ ] OBS guide + browser source preset
- [ ] CasparCG HTML template (key + fill)
- [ ] Captions feed ingest (WebVTT/SRT stream, EBU-TT-D; teletext via existing subtitle systems)
- [ ] NDI with alpha helper
- [ ] HbbTV demo app
- [ ] Broadcaster pilot + accessibility/legal review (WCAG 1.2.6, EAA, national rules)

## M8 Trained recognition

- [ ] Dataset license matrix verified (PLAN §8.3)
- [ ] `ml/` pipeline: data manifest, augmentation from engine, landmark transformer / ST-GCN
- [ ] ONNX export (fp16/int8), ONNX Runtime Web runner, model cards, hash pinning
- [ ] Benchmarks (signer-independent split), published results

## M9 Better translation

- [ ] LLM translator with lexicon-constrained JSON output, per-language grammar notes, cache
- [ ] Grammar phase B (numbers, time-first, WH placement) and C (loci, agreement verbs)
- [ ] Evaluation set per language + Deaf panel comprehension tests

## M10 Research

- [ ] Continuous recognition (CTC)
- [ ] Sign → text translation (landmarks → LLM)
- [ ] Photoreal neural avatar (consent-based, watermark)

## M11 Apps

- [ ] PWA with offline language packs
- [ ] Native mobile shell (Capacitor or Flutter + WebView)
- [ ] Desktop overlay app (Tauri/Electron)

## Community and funding (ongoing)

- [ ] Contact Deaf associations (WFD, EUD, PZG) and sign linguistics labs (UW PJM, Hamburg DGS)
- [ ] Language maintainers per Tier 1 language
- [ ] Grant applications (NLnet/NGI Zero, EU accessibility calls, PFRON)
