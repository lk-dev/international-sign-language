# Engine design notes (v0, draft)

Conventions shared by `notation`, `engine`, `avatar-web` and `recognition`. Anything that produces or consumes poses must follow them, because recognition templates are generated from the same hand model the renderer uses.

## 1. World frame

- Units: metres. Right-handed: **+Y up**, **+Z towards the viewer** (the avatar faces +Z), **+X = viewer's right**.
- The avatar's right side is therefore **−X**.
- Reference skeleton (all avatars are calibrated to it; `avatar.json` may override landmarks):

| Joint / landmark | Position (x, y, z) |
|---|---|
| pelvis | (0, 1.00, 0) |
| chest centre | (0, 1.30, 0) |
| shoulder joints | (±0.19, 1.43, 0.0) |
| neck base | (0, 1.45, 0) |
| head centre | (0, 1.62, 0) |
| upper arm / forearm length | 0.29 / 0.26 |

## 2. Signer frame (side-relative)

Signs are authored for the **dominant hand** in a side-relative frame so they mirror automatically:

- `x_s` > 0 = **out** (towards the hand's own side), `x_s` < 0 = **in** (towards the midline / other side);
- `y` up, `z` forward (towards the viewer);
- world `x = −x_s` for the right hand, `x = +x_s` for the left hand.

Named directions: `up`, `down`, `fwd`, `back`, `in`, `out`, combinable (`up+in`, `fwd+down`), normalised after combination.

Body landmarks (signer frame, reference skeleton):

| Name | (x_s, y, z) |
|---|---|
| `neutral` | (0.08, 1.20, 0.32) |
| `neutralHigh` | (0.08, 1.38, 0.32) |
| `fingerspell` | (0.14, 1.36, 0.30) |
| `chest` / `chestIpsi` / `chestContra` | (0 / 0.08 / −0.08, 1.30, 0.14) |
| `stomach` | (0, 1.12, 0.14) |
| `neck` | (0, 1.47, 0.07) |
| `chin` | (0, 1.53, 0.11) |
| `mouth` | (0, 1.565, 0.12) |
| `nose` | (0, 1.60, 0.13) |
| `eye` | (0.035, 1.64, 0.11) |
| `cheek` | (0.06, 1.58, 0.10) |
| `forehead` | (0, 1.70, 0.11) |
| `temple` | (0.08, 1.67, 0.06) |
| `ear` | (0.10, 1.62, 0.0) |
| `headTop` | (0, 1.78, 0) |
| `shoulderIpsi` / `shoulderContra` | (±0.19, 1.43, 0.06) |
| `side` | (0.30, 1.15, 0.15) |

Locations may also reference the **other hand**: `L.palm`, `L.back`, `L.index.tip`, `L.thumb.tip`, … (BSL fingerspelling, base-hand signs). The other hand is resolved first, then its FK gives the target.

## 3. Hand model

Hand-local frame (right hand): origin at the wrist, **+Y = towards the fingers**, **+Z = palm normal** (out of the palm), **+X = thumb side** (`x = y × z`). The left hand is the mirror image (geometry scaled −1 on X), so the same `(fingerDir, palmDir)` gives a proper rotation for both hands.

Hand orientation in a keyframe: `f` = finger direction (wrist → middle knuckle), `p` = palm facing direction; `p` is orthogonalised against `f`.

Finger bases (MCP), metres:

| Finger | MCP (x, y) | Phalanges (prox, mid, dist) |
|---|---|---|
| index | (+0.027, 0.082) | 0.043, 0.025, 0.020 |
| middle | (+0.008, 0.085) | 0.047, 0.029, 0.021 |
| ring | (−0.011, 0.080) | 0.044, 0.028, 0.020 |
| pinky | (−0.028, 0.072) | 0.034, 0.020, 0.018 |
| thumb | CMC (0.020, 0.025, 0.005) | metacarpal 0.045, proximal 0.032, distal 0.026 |

Joint rotations:

- Fingers: `R_mcp = Rz(spread) · Rx(mcp)`, then `Rx(pip)`, `Rx(dip)`. Positive flexion rotates +Y towards +Z (into the palm). Spread is positive away from the middle finger.
- Thumb: `R_base = Rz(−(40° + rad)) · Rx(palm) · Ry(−roll)` (radial abduction, palmar abduction, pronation/opposition; default roll ≈ 50°), then `Rx(mcp)`, `Rx(ip)`.

A **handshape** = per-finger `[mcp, pip, dip, spread]` + thumb `{rad, palm, roll, mcp, ip}`. Named shapes (`A`, `S`, `B`, `5`, `C`, `O`, `1`, `V`, …) are defined once in the engine and reused by every language.

## 4. Keyframes and sampling

- Per-hand key: `t`, `hs` (handshape), `at` (landmark), `off` (signer-frame offset), `by` (contact point of the hand: `palm`, `wrist`, `tips`, `index.tip`, …), `f`, `p`, `path` to next key (`line`, `arc{bulge}`, `circle{axis,turns}`, `zigzag`), `local` (`wiggle`, `twist`, `flick`). Missing fields inherit from the previous key.
- Non-manual track: `brows` (−1 furrow … +1 raise), `eyes` (−1 squint … +1 wide), `mouth {open, wide, round}`, `puff`, `head {pitch, yaw, roll}`, `torso {lean, turn}`, `gaze` (`viewer` / `hands` / locus).
- Interpolation: positions with minimal-jerk easing; hand orientation as quaternion slerp; handshape per joint (fingers lead the arm by ~40 ms).
- `sample(t)` returns a `Pose`. IK and rendering are the renderer's job.

## 5. Recognition features

MediaPipe hand landmark order (0 wrist; 1–4 thumb CMC→tip; 5–8 index; 9–12 middle; 13–16 ring; 17–20 pinky). The engine computes the same 21 points from a handshape via FK, so every named handshape and alphabet letter has a synthetic template.

- Canonical handshape vector: subtract wrist, rotate so wrist→middle-MCP = +Y and index-MCP→pinky-MCP lies in the XY plane, divide by palm length, mirror left hands (x → −x) → 60 floats.
- Orientation: palm normal and finger direction in camera space (6 floats), compared separately with a weight. Some letters differ only by orientation: ASL H/U, K/P, G/Q.
- MediaPipe world landmarks: x right, y down, z away from the camera. That is also right-handed, so chirality is preserved by proper rotations.
- Handedness: MediaPipe assumes mirrored (selfie) input. For raw camera frames and ordinary videos the label is inverted.
