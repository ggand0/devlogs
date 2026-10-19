# Original sword clips: the AI options in 2026

Written by Claude Fable 5.1.

2026-10-17. Gota wants sword clips that are not reproductions of the M2TW references, because a full set of copies would read as a copy. Kimodo (devlog 0230) makes good bare-hand and leg motion but no sword work. This log records a web survey of what exists in October 2026, the four routes that produce original weapon motion, and Gota's pick. No code changed.

## Why text-to-motion fails at swords

Kimodo and its peers are trained on body-only capture (HumanML3D-style data, Bones Rigplay for Kimodo). The sword is not in the data, so the word "sword" in a prompt carries no motion. Nothing found in 2026 generates weapon-specific skeletal motion from text for a 3D engine. The nearest were a hobby Roblox sword animator trained on keyframes (https://github.com/AMohammed03/R6-Sword-ML-Animator/) and 2D sprite preset tools (https://ludo.ai/tools/animation-presets/weapons), neither useful for our skeleton.

## The four routes

### 1. Perform and capture with single-camera AI mocap

Film a performer holding a stick of sword weight, run video-to-motion. Our 19-joint skeleton carries the sword as a rigid child of the hand, so the capture never needs to know a weapon exists.

- Local and free: GEM, the renamed GENMO from NVIDIA (paper https://arxiv.org/pdf/2505.01425, code https://github.com/NVlabs/GENMO, checkpoint GEM-SMPL at https://huggingface.co/Tinghong-Ye22/code-GENMO). Also GVHMR and TRAM.
- Hosted: Move One from about $18 a month (https://www.strayspark.studio/blog/ai-mocap-indie-developers-2026-comparison), DeepMotion from $9 a month (https://www.mimem.ai/guides/best-mocap-software), Rokoko Vision free for short clips (https://filmthreat.com/features/10-best-ai-tools-for-turning-video-into-3d-character-motion/), Uthana (https://uthana.com/models/video-to-motion).
- Weak spot in all of them: hands and anything hidden behind the body, so the blade path needs a cleanup pass at the contact frames. A January 2026 paper on physics refinement of video capture addresses that gap (https://arxiv.org/pdf/2601.19036).

Rejected by Gota: he is not a trained swordsman, so this route means hiring a performer.

### 2. Kimodo with body prompts

Prompt the arm, not the tool: "steps forward with the right foot and swings the right arm in a wide diagonal arc from high right to low left", then attach the sword bone. Free to try with what is installed. Expected limit: no blade weight or follow-through, since the data has none. Not tried yet.

### 3. Cascadeur for moves nobody can perform

AI-assisted posing, not generation: place four or five key poses, AutoPhysics fills the arc with momentum. Version 2026.1 added point constraints that lock a prop in the hand during physics (https://cascadeur.com/blog/view/cascadeur-2026-1-new-renderer-ue-live-link; interview on props and retargeting https://digitalproduction.com/2026/02/18/cascadeur-on-physics-ai-and-control/; FAQ https://cascadeur.com/help/faq). Free tier and a Linux build exist. Fits parry, block and the fencing ideas parked for after Early Access, and the cleanup of clips from routes 1 and 4.

### 4. AI video, then mocap it (Gota's pick)

Generate a video of the swing with a video model, then run video-to-motion from route 1 on it. The only route that invents moves without a performer. Seedance 2 is reported to handle weapon motion, cloth and impacts (guide dated April 2026, https://autoweeb.com/blog/how-to-create-anime-sword-fight-videos-with-ai); a 2026 paper cascades text to skeleton to video for controllable complex motion (https://arxiv.org/pdf/2603.08028). Two lossy steps: the video model may change proportions or hand count mid-clip, and the mocap then estimates whatever is hidden. Expect to generate several videos per move and keep the cleanest.

## What it needs on our side

- A prompt and camera recipe for the video model: a single figure, side or three-quarter view, static camera, plain background, full body in frame, the sword visible through the whole swing. The hidden half of a swing is where mocap guesses.
- Video-to-motion running locally (GEM-SMPL on our GPU) or a hosted account.
- A SMPL-to-19-joint source mapping for the transfer tool (handoff 122). The tool maps Kimodo's SOMA skeleton today; mocap gives SMPL.
- A cleanup pass per clip at the contact frames, in Blender or Cascadeur.
- License reading before anything ships: GEM's code and checkpoint licenses are unread (NVlabs research repos are often non-commercial; devlog 0230 already excludes Kimodo's SMPL-X model for that reason), and the video model's terms on output use. Same table format as devlog 0230.

## Sources read

- https://www.strayspark.studio/blog/ai-mocap-indie-developers-2026-comparison
- https://www.mimem.ai/guides/best-mocap-software
- https://filmthreat.com/features/10-best-ai-tools-for-turning-video-into-3d-character-motion/
- https://sunstrikestudios.com/en/blog/motion_capture_for_games_and_film/
- https://themenonlab.blog/blog/mixamo-llm-mocap-video-to-3d-animation
- https://arxiv.org/pdf/2505.01425 (GENMO / GEM)
- https://arxiv.org/pdf/2603.15975 (UMO, motion foundation model priors)
- https://arxiv.org/pdf/2512.10730 (IRG-MotionLLM, text-to-motion with refinement)
- https://arxiv.org/pdf/2601.02107 (MagicFight, martial arts video generation)
- https://cascadeur.com/blog/view/cascadeur-2026-1-new-renderer-ue-live-link
- https://digitalproduction.com/2026/02/18/cascadeur-on-physics-ai-and-control/
- https://autoweeb.com/blog/how-to-create-anime-sword-fight-videos-with-ai

## The chosen workflow (2026-10-17, second pass)

Gota asked for one workflow, not candidates. Picks from a second search round:

1. Video: Kling v3 Pro, image-to-video. Reviews name it the model for human performance with martial arts listed as a strength, and it leads the blind-vote text-to-video arena (https://llm-stats.com/leaderboards/best-ai-for-video-creation, https://www.eachlabs.ai/blog/best-ai-video-generation-models-2026, https://pixo.video/blog/seedance-vs-veo-vs-kling). Start every clip from the same reference frame of a plain figure with a sword, so proportions hold across the set. Static camera, three-quarter view, plain background, full body and blade in frame, one move per 5 s clip, several generations per move, keep the cleanest.
2. Mocap: GVHMR, local on the shared GPU (https://dl.acm.org/doi/10.1145/3680528.3687565, code on GitHub under zju3dv/GVHMR). It is the baseline every 2026 paper measures against, it reconstructs world-grounded SMPL-X directly, and its known weakness (drift under camera motion) does not apply to a static-camera clip. If contact frames come out physically off, add a refinement pass: FlowHMR (https://arxiv.org/html/2610.03691v1) or HTD-Refine (https://arxiv.org/pdf/2605.26879), both of which report gains on top of GVHMR.
3. Transfer: a SMPL-X-to-19-joint source mapping in the transfer tool (handoff 122), the sword a rigid child of the right hand.
4. Cleanup: Cascadeur with the sword point-constrained to the hand, contact frames and follow-through only.

Licenses to read before anything ships: GVHMR's repo license and Kling's output terms on the paid plan.

## 2026-10-18: the first clips, the mocap step changes, the adapter

Kling 4.0 on Gota's paid plan, the swordsman reference image (`work/notes/kling/swordsman_ref.png`, made with ChatGPT: plain gambeson, no helmet, no shield, grey studio, 9:16). Two generations:

- Omni reference mode, long prompt: the model recomposed the frame to 16:9 with a small figure and filled 5 s with a flourish of four swings. Omni reference is a loose look reference; Keyframes mode makes the image the literal first frame.
- Keyframes mode, start frame plus an end frame, short prompt: one swing, clean body, full-frame figure, 1080x1916 at 24 fps. But a salute, not a strike: a slow lift for over a second, a drop with no step and no turn, then 2.5 s of relaxed stand. The end frame was a relaxed stand and nothing was being hit. Next generation: an end frame in a deep committed lunge and a prompt that attacks an enemy in front of him within the first second.

**GVHMR is out.** Its license (repo LICENSE, read 2026-10-18) permits educational, research and non-profit use only; commercial use needs permission by email from Zhejiang University. The SMPL and SMPL-X body models it needs are research-only as well. Meshcapade, the SMPL authors' commercial home, was bought by Epic Games in February 2026 and its platform's status is unclear. Move One is an iOS capture app, not an upload service.

**DeepMotion Animate 3D is the mocap step.** It takes an uploaded MP4, MOV or AVI, exports FBX, BVH, GLB and MP4, has hand and face tracking at half a credit per second extra, 1 credit per second of animation. Freemium: 60 s a month, "available for personal, non-commercial use". "In order to have licensed animation, you must have a premium subscription plan." So the pipeline test runs on freemium with the 5 s clip; shipping clips need a premium month. Prices at https://deepmotion.com/pricing.

**The adapter.** `assets_dev/_skel/bvhsource.py`: a BVH file (DeepMotion, Move One, Rokoko, Mixamo-style names, `--map our=theirs` for others) as a source for `transfer.py`. Parses the hierarchy and channels, forward kinematics per frame, joint names by a candidate table with sides, facing from the hips line (any rig facing), units from the leg length, the blade as a frame on the right hand (forward across the fingers at rest, flat up, turning with the hand), the lowest ankle over the clip as the ground, the pelvis's straight-line drift taken out and put in the clip's `travel`, nearest-frame resampling to 20 fps, `--start`/`--end` to trim idle. Tested end to end on a synthetic Mixamo-style T-pose BVH in centimetres (90 frames at 30 fps, arm swing, head turn, knee bend, 40 cm walk): 60 samples, travel 0.38 m, `validate_clip` ok. Untested on a real DeepMotion file until Gota uploads the clip.

**A latent bug in `author.py`, fixed.** `Pose.leg` took the knee hinge from thigh × shin and, for a dead-straight leg, fell back to thigh × Z, which points the other way from a forward-bending knee: the thigh yawed half a turn between a straight frame and a bent one (the validator caught it: 3600 deg/s on the synthetic clip). Mocap straightens knees all the time. Now the fallback is pelvis-forward × thigh, and a hinge pointing backward (a hair of hyperextension, mocap noise) or too short (under 0.03, about 2 degrees of bend) takes that fallback too. Authored clips with any knee bend are unchanged. Astra should know, since author.py is the shared pose builder.

Next: Gota uploads the Kling clip to DeepMotion (freemium), hand tracking on, foot locking on, BVH export; the adapter runs on it; the clip renders beside the video; then the better generation with the lunge end frame.

**The bar, restated by Gota 2026-10-18:** the generated motion is a reference for Astra to start from, not a finished clip; a clip good enough to use as it is remains the ideal. The tool output need not ship, so per-tool licenses matter less than motion quality, and the Kling video alone is already a reference sheet. The whole chain, routes 1 to 4 above included, gets re-evaluated from scratch against that bar once the first DeepMotion result is in.

**First real file, 2026-10-18.** DeepMotion's BVH of the salute clip (`work/notes/kling/salute.bvh`, 121 frames at 24 fps, 53 joints with fingers, names like `r_arm_JNT`, centimetres) went through the adapter after two small additions: a `_JNT`/`_joint`/`_bone` suffix strip in the name match, and the middle finger preferred as the hand's child. The body carried over as the video shows it: guard, lift over the head at 2.15 s, drop at 2.85 s, relaxed hold; feet planted, no flips, travel 0.13 m left and 0.09 m forward. The validator flags only the hand at the drop (1508 deg/s), the video's own two-frame fall. The blade was wrong at first (up at the start, out to his right at the end): the guessed rest grip is off by a roll in DeepMotion's rig. Fix: `--blade0 x,y,z`, the blade's direction in the first frame as the video shows it, calibrates the grip. With `--blade0=-0.5,0.15,0.85` the blade matches the video throughout. Sheet and GIF in `work/notes/kling/sheets/salute/`. The chain works end to end; the motion itself is the salute, not a strike, so the next Kling generation is the real test.
