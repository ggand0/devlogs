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
