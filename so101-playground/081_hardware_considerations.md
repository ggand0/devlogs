# Hardware Considerations for HIL-SERL Training

## 1. Side/Overhead Camera for Recording

For experiment documentation and eval footage (not policy input).

### Budget Setup (~$50-80)

| Item | Price | Notes |
|------|-------|-------|
| Logitech C920/C922 | ~$50-70 | 1080p, standard robotics lab choice |
| Desk clamp arm mount | ~$15-25 | Flexible positioning |

### Better Quality (~$100-150)

| Item | Price | Notes |
|------|-------|-------|
| Logitech Brio | ~$130 | 4K, better low-light |
| GoPro Hero (used) | ~$100-150 | Wide FOV |

### Placement
- 60-80cm above table
- 30-45° angle looking down
- Captures full arm workspace + cube area

---

## 2. Side Camera for Policy Input

If wrist camera FOV proves insufficient, adding a side camera for policy observation.

### Problem with Wrist-Only Camera
- Limited FOV - cube can leave frame when arm moves
- HIL-SERL paper: "if wrist cameras alone cannot provide a full view of the environment, we also place several side cameras"

### Candidates
Same cameras as recording setup (C920/C922 or Brio), but would need:
- Config changes to add second image observation
- Reward classifier retraining with new camera view
- Policy retraining from scratch

---

## 3. Lighting for Consistent Training

### Problem
- Training started at 1:45pm (daylight from window)
- Nighttime training has different lighting
- Visual observations shift → policy/reward classifier affected

### Solution: Desk Lamp

| Option | Price | Features |
|--------|-------|----------|
| Generic LED desk lamp | ~$25-40 | Adjustable brightness, color temp |
| BenQ ScreenBar | ~$100 | Monitor-mounted, no desk space |
| LED panel light | ~$30 | Clip/tripod mount |

### Key Requirements
- **Daylight color temp (5000-6500K)** - matches window light
- **Dimmable** - adjust to match original brightness
- **Stable position** - consistent between sessions

### Recommendation
Wait for daytime if no lamp available. Inconsistent lighting during training is worse than waiting.

---

## 4. Future Considerations

### If Adding Side Camera for Policy
1. Mount camera with clear view of workspace
2. Collect new reward classifier dataset with both views
3. Modify config to include second image observation
4. Retrain policy from scratch with dual-camera input

### If Switching to DrQ-v2
- Frame stacking (n_obs_steps=3) may help track cube motion
- Could reduce need for side camera if temporal info helps
