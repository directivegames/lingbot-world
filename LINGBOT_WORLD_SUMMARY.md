# LingBot-World Summary (Conversation Notes)

## What this project is

`lingbot-world` is an open-source **interactive world model inference** project built on Wan2.2.

In this repo, a world model means a model that predicts future visual states (video frames) conditioned on:
- a starting observation (image),
- text prompt,
- optional control signals (camera trajectory and/or actions).

Primary open-sourced focus in this repository is **inference** and model architecture integration, not full end-to-end training code.

---

## Repo-level pipeline (inference)

High-level path:
1. `generate.py` parses CLI args and runtime settings.
2. `wan/image2video.py` (`WanI2V`) builds and runs the generation pipeline.
3. Text encoder + VAE + DiT experts run diffusion sampling.
4. Decoded output is saved as video (`.mp4`).

Key implementation files:
- `generate.py`
- `wan/image2video.py`
- `wan/modules/model.py`
- `wan/utils/cam_utils.py`
- `wan/configs/wan_i2v_A14B.py`
- `wan/configs/shared_config.py`

---

## Training data (from paper)

The paper describes a hybrid data engine:
- General videos (real-world/open-source/in-house)
- Game recordings (with synchronized action and camera signals)
- Synthetic Unreal Engine renders (with precise camera metadata)

Data is further processed with:
- Filtering/profiling (quality, motion, perspective, etc.)
- Camera pseudo-labeling when missing
- Hierarchical captioning:
  - narrative/global caption
  - scene-static caption
  - dense temporal captions

So a training sample is not only RGB frames; it includes controls + semantic metadata.

---

## Training pipeline (from paper)

Three-stage evolution strategy:

1. **Stage I: Pre-training**
   - Build strong general video prior and visual fidelity.

2. **Stage II: Middle-training**
   - Inject world knowledge + action/camera controllability.
   - Improve long-horizon consistency and interactive behavior.
   - Uses control-conditioned training and a two-expert denoising design.

3. **Stage III: Post-training**
   - Adapt toward low-latency interactive rollout (causal-style generation + distillation).

Note: these stages are described in the paper; this repo mainly exposes inference flow.

---

## Control signals: role and mechanism

Control signals are central to interactive behavior:
- Camera controls: intrinsics + poses
- Action controls: e.g., WASD-like signals (Act variant)

During inference, controls are transformed into geometric/control embeddings and injected into the DiT backbone (`c2ws_plucker_emb` path).
Conceptually, training with these signals teaches action-contingent dynamics: different controls should produce different future trajectories.

---

## Output capabilities

- Native inference output in this repo is **video**.
- No direct geometry output (mesh/point cloud/NeRF/depth) is produced by default code path.
- Geometry-related use in paper is a downstream use case (reconstruction from generated video), not the direct output type of this inference entrypoint.

---

## Inference length limits

There is no strict single hard-coded "global max duration" gate in the current CLI path, but practical limits exist:

- `frame_num` should follow `4n+1` convention.
- Default frame count is `81` frames.
- Longer outputs are mostly limited by GPU memory/time.
- If `action_path` is provided, effective frame count is constrained by available control sequence length (poses/actions).

---

## Physics correctness expectations

LingBot-World does **not** enforce hard physical laws like a deterministic game physics engine.

It learns plausible dynamics statistically from data and controls, which improves consistency, but does not guarantee strict collision or conservation behavior in all cases.

Implication: in long or difficult rollouts, physically implausible behavior can still occur (e.g., wall clipping/drift).

---

## Can we add other control signals (e.g., game logic)?

Likely yes, according to the paper's conditioning formulation and current code pattern.

Possible extension path:
- Add synchronized game-logic channels in training data.
- Extend control embedding/injection dimensions and modules.
- Train with objectives/curriculum that force utilization of those controls.
- Expose new controls in inference API.

Caveat: better controllability does not automatically yield perfect symbolic simulation; strict guarantees usually require hybrid engine constraints.

---

## Recommended next steps

1. Run baseline inference with provided examples.
2. Compare `cam` vs `act` behavior using same prompt/image.
3. Sweep `frame_num`, `sample_steps`, and guidance scale to observe stability/latency tradeoffs.
4. If desired, design a prototype schema for additional control channels (game logic) and where to inject them in `WanModel`.

