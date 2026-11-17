---
title: StepAhead User Guide
---

# StepAhead: Predictive Foot Placement

StepAhead places a character's feet on the real ground *before* they land. Most foot IK only reacts to the ground under the foot. StepAhead knows where each foot will land next, because that spot is baked into your animations. It traces the ground there and blends the foot from the step it lifted off to the step it lands on as the step progresses. On stairs, slopes and ledges the swinging foot rises or drops toward the next step in time, instead of snapping when it touches down.

Supported engine versions: Unreal Engine 5.4 and 5.8. Platform: Windows (Win64).

## Contents

1. [How it works](#how-it-works)
2. [Quick start](#quick-start)
3. [Baking your animations](#baking-your-animations)
4. [The Next Step Predictor component](#the-next-step-predictor-component)
5. [Placing the feet: AnimBP, Control Rig or Blueprint](#placing-the-feet)
6. [Settings reference](#settings-reference)
7. [Runtime functions](#runtime-functions)
8. [Debugging](#debugging)
9. [Performance](#performance)
10. [Troubleshooting](#troubleshooting)
11. [Limitations](#limitations)

## How it works

StepAhead has two halves.

**The bake (editor).** For each foot, the bake finds where the foot is planted in every frame of an animation. It then writes three things into the animation:

| Data | Name (default) | Meaning |
|---|---|---|
| Bone | `nextstep_<foot>` | Where the foot is planted now, or where it will be planted next, relative to the character. |
| Curve | `Planted_<foot>` | 1 while the foot is planted, 0 while it is in the air. |
| Curve | `StepProgress_<foot>` | 0 while planted, rising to 1 at the moment the foot lands. |

The bone is added to your skeletal mesh as a new leaf bone. Because the data lives in the animations, blending, blend spaces, montages and motion matching all blend the predictions along with the pose.

**The runtime (game).** The *Next Step Predictor* component reads the baked bone every frame and traces the world under it. A *Predictive Foot Placement* node, in the AnimBP or a Control Rig, then moves each foot's IK target: from the ground it lifted off toward the ground it will land on, by its step progress. Your leg IK then solves the legs to those targets.

## Quick start

These steps use the UE5 mannequin (`SKM_Manny` / `SKM_Quinn`, feet `foot_l` / `foot_r`, IK targets `ik_foot_l` / `ik_foot_r`). Use your own names if they differ.

1. **Bake.** Open **Tools → StepAhead**. Add two rules, one per foot:
   - **Skeletal Mesh**: `SKM_Manny`.
   - **Foot Bone Name**: `foot_l` in one rule, `foot_r` in the other.
   - **Folder**: the folder that holds your locomotion animations.

   Click **Bake Next Steps**, then save the changed animations and meshes.
2. **Add the component.** Add a **Next Step Predictor** component to your character Blueprint. Under **Feet**, add two entries:
   - **Foot Bone** `foot_l`, **Target Bone** `ik_foot_l`.
   - **Foot Bone** `foot_r`, **Target Bone** `ik_foot_r`.

   Optionally set **Pelvis Bone** to `pelvis`.
3. **Place the feet.** In the AnimBP's AnimGraph, add **Predictive Foot Placement** after your locomotion pose. Follow it with your leg IK (for example **Two Bone IK** or **Leg IK**) that pulls each foot to its `ik_foot_*` bone.
4. **Test.** Play, and walk up stairs and slopes. Type `a.StepAhead.Debug.All 1` in the console to see what is traced.

To bake new animations automatically from then on, click **Save to Project Settings** in the panel. See [Bake on import](#bake-on-import).

## Baking your animations

### The StepAhead panel

**Tools → StepAhead** opens the bake panel. It holds a list of **rules**, usually one per foot. Each rule bakes one foot into every animation of one skeleton:

| Setting | Meaning |
|---|---|
| Enabled | Whether **Bake Next Steps** runs this rule. |
| Skeletal Mesh | The mesh the prediction bone is added to. Animations of its skeleton are baked. |
| Add To All Compatible Meshes | Also adds the prediction bone to every other mesh that can play these animations (for example Manny and Quinn). On by default. |
| Foot Bone Name | The foot whose plants are predicted. |
| Next Step Bone Name | The prediction bone's name. None: `nextstep_<foot>`. |
| Parent Bone Name | The prediction bone's parent. None: the skeleton's root. |
| Folder | Only animations in this folder are baked. Empty: any folder. |
| Plant Settings | How plants are detected (see below). |
| Planted Curve Name / Step Progress Curve Name | The curve names. None: `Planted_<foot>` / `StepProgress_<foot>`. |

**Bake Next Steps** runs every enabled rule. It first adds the prediction bone to the mesh(es) and merges it into the skeleton; any open editors for that skeleton close and reopen. Then it bakes every matching animation. If the bone already exists, you are asked before existing data is overwritten.

Save the animations and meshes afterwards. Adding a bone changes the skeleton, so the engine marks every loaded animation of that skeleton as changed. That is expected.

> **Tip:** Use source control, or back up the assets, before the first bake. The bake changes your skeletal meshes, skeleton and animations.

### Plant detection settings

| Setting | Default | Meaning |
|---|---|---|
| Height Tolerance | 3 cm | How far above its lowest point in a footstep the foot may be and still count as on the floor. |
| Max Ankle Height | 20 cm | A footstep whose lowest point is higher than this above the root is a foot in the air, not a plant. |
| Speed Threshold | 20 cm/s | The fastest horizontal foot speed that still counts as planted. |
| Min Contact Frames | 2 | Contacts shorter than this are ignored. |
| Step Progress Basis | Distance | What step progress rises with: the foot's horizontal travel (Distance), or time. |

Detection works on both in-place and root motion animations, on flat ground, slopes and stairs. It also handles loops: a plant that wraps around the end of the clip counts as one plant. In clips where the foot never lands (jumps, falls), the bone follows the foot and both curves stay at 0.

### Bake on import

**Save to Project Settings** stores the panel's rules as the project's import rules. They live under **Project Settings → Plugins → StepAhead** and are saved in `Config/DefaultEditor.ini`. While **Bake On Import** is on, every animation that is imported or reimported for a rule's skeleton (in the rule's folder) is baked automatically.

### Animations that suit the bake

- Locomotion with clear foot plants: walks, jogs, runs, starts, stops, pivots and strafes.
- Each foot needs at least one plant in the clip; contacts shorter than **Min Contact Frames** are ignored.

## The Next Step Predictor component

Add one per character. It finds the owner's first skeletal mesh (or the one set in **Mesh**). It ticks once per frame after the mesh has animated, and is the only place the ground is traced. The placement nodes read its results.

What it does each frame:

- **Traces.** A swinging foot's prediction bone is traced. A planted foot (planted curve 0.5 or more) is traced under the foot bone itself. A trace is reused until its point moves more than **Retrace Radius**, so a planted foot is usually traced once.
- **Follows the movement mode.** While the movement component isn't on the ground (jumping, falling, flying), the feet blend back to the animation (**Only On Ground**).
- **Checks the facing.** The predictions are baked relative to the character's facing, so they point the wrong way while the capsule is still turning. During that time they aren't used (**Facing Check**).
- **Checks reachability.** A step the Character Movement can't get onto (too high, too steep, too far down) is left to the animation. The foot doesn't reach for a ledge the character won't climb (**Check Reachable**).
- **Samples the terrain in between.** The ground between a swinging foot's plant and its next step is sampled, so the foot lifts over anything higher than where it lands (**Step Terrain**).
- **Follows moving ground.** Ground found on movable components (lifts, moving platforms) moves along with them.

## Placing the feet

All three ways run the same placement code, so they behave the same.

### AnimBP node

**Predictive Foot Placement** (search "StepAhead" or "Predictive" in the AnimGraph) is a skeletal control node. It finds the component on the owning actor; you can also set it on the hidden **Predictor** pin.

- Put it after the locomotion pose and before your leg IK.
- It moves each foot's **Target Bone** by the ground height and tilt, and the **Pelvis Bone** (if set) down by the lowest foot.
- Your leg IK must then pull the feet to those targets.
- The target bones must follow the feet in the animation. The UE5 mannequin's `ik_foot_*` bones do. If your skeleton's IK bones aren't animated, copy the foot bones onto them first (**Copy Bone**).
- The node's **Alpha** blends the whole effect.

### Control Rig unit

**Predictive Foot Placement** (category StepAhead) does the same in a Control Rig. Place it in **Forward Solve** before the leg IK.

- **The rig needs the curves.** A Control Rig only receives animation curves that already exist in its hierarchy. Import `Planted_*` and `StepProgress_*` into the rig's curves; the bake adds them to the skeleton's curve list, so they can be imported from there. Without them every foot counts as planted, and the unit logs a warning.
- **Propagate To Children** moves the target bones' children with them.
- **Outputs:**
  - **Valid**: the component was found and every foot was placed.
  - **Offsets**: the height and tilt added to each foot, in the order of **Feet**.
  - **Lowest Height**: the lowest height added to any foot, for your own pelvis handling.

### Blueprint

For custom setups, the **StepAhead** Blueprint library has the core function **Update Predictive Foot Placement**. Call it per foot with:

- a state variable you keep per foot;
- the component's **Get Next Step For Placement**;
- the foot's step progress and planted curves.

It returns a height and tilt offset. **Clamp Foot Offset**, **Weight Foot Offset** (with **Get Tracing Weight**) and **Apply Foot Offset** finish the job.

## Settings reference

### Next Step

| Setting | Default | Meaning |
|---|---|---|
| Feet | — | The feet to predict and place. **Foot Bone** (the baked foot) and **Target Bone** (the bone placement moves, e.g. `ik_foot_l`) are required. The prediction bone and curve names default to the bake's names (advanced). |
| Mesh | owner's first skeletal mesh | The mesh with the prediction bones. |
| Only On Ground | on | Only place the feet while the movement component's `IsMovingOnGround()` is true. For custom movement modes, override `IsMovingOnGround()` in your movement component. |

### Tracing

| Setting | Default | Meaning |
|---|---|---|
| Trace Settings → Trace Channel | Visibility | The channel the ground is traced on. |
| Trace Settings → Trace Up / Trace Down | 50 / 75 cm | How far above and below each prediction the trace runs. |
| Trace LODs | 3 entries | How to trace at each mesh LOD (see below). |
| Tracing Enabled | on | Off blends the feet back to the animation. Change it at runtime with **Set Tracing Enabled**. |
| Only When Rendered | on | Only trace while the mesh was rendered in the last 0.2 s. |
| Facing Check | on, 15° / 10° | Stops using the predictions while the capsule is more than **Max Angle** off the facing the Character Movement turns it to, until it is within **Resume Angle**. **Response**: **Hold Previous Step** (swinging feet blend to the animation height) or **Disable Tracing**. |
| Trace LOD Override | -1 | Always use this Trace LODs entry; -1 lets the mesh LOD decide. |
| Tracing Blend In / Out Time | 0.25 s | How fast the feet blend in and out when tracing turns on or off. |

**Trace LODs.** Each entry describes how to trace at one mesh LOD; LODs past the end use the last entry.

| Setting | Meaning |
|---|---|
| Shape | **Line** (cheapest; can drop into gaps and miss stair edges), **Sphere** (finds the highest ground around the point), **Box** (aligned with the character; finds a stair's level top at its edge), or **Disabled** (no tracing at this LOD; the feet blend to the animation). |
| Trace Radius | Sphere radius or box half size. |
| Retrace Radius | How far a prediction may move horizontally before it is traced again; 0 = every frame. |
| Async Traces | Run the traces asynchronously and read them a frame later, instead of blocking the game thread. Ignored while the component has a tick interval. |
| Step Terrain | How the ground between the steps is found: **None** (blend straight from step to step), **Other Foot** (the other foot's plant, if it is near the middle of the step; no extra traces), **Substeps** (**Substep Count** evenly spaced traces per new step), or **Under Foot** (one trace per frame just ahead of the foot, between **Under Foot Start** and **Under Foot End** of the step; the most exact). |
| Check Reachable | Leave steps the Character Movement can't get onto to the animation. |
| Check Path | Also sweep the capsule to each new step to find walls or railings in between (**Path Radius Scale** sets its radius). |

The defaults:
- **LOD 0:** a 5 cm box, retraced every 15 cm.
- **LOD 1:** a line, retraced every 30 cm.
- **LOD 2 and beyond:** no tracing.

LODs 0 and 1 both use **Other Foot** as the step terrain.

### Foot Placement

| Setting | Default | Meaning |
|---|---|---|
| Apply Height | on | Move the target bones to the ground height. |
| Apply Rotation | on | Tilt the target bones to the ground slope, up to **Max Align Angle** (30°). |
| Blend Time | 0.4 s | How long sudden changes (new ground, ground appearing or disappearing) take to blend in; 0 = instantly. |
| Foot Offset Limits | 50 / 50 cm | How far each foot may move down or up. |
| Pelvis Bone | None | A bone (e.g. `pelvis`) moved down by the lowest foot's height, so the legs can reach. |
| Hold Step Progress | on | Step progress never goes back down until the foot is planted, so blending animations doesn't pull the foot back mid-step. |

### Terrain and Reachability

| Setting | Default | Meaning |
|---|---|---|
| Terrain Settings → Flat Angle | 3° | Ground tilted less than this counts as level. |
| Terrain Settings → Stairs Angle Tolerance | 10° | A step steeper than the slope by more than this can be stairs. |
| Terrain Settings → Min Stairs Height | 5 cm | The least height a step needs to count as stairs. |
| Min Step Width | 60 cm | The narrowest tread the character can stand on, used to count how many steps fit on the way to a step; 0 = off. |
| Max Steps Look Ahead | 2 | The most steps counted on the way to a step. |

Reachability uses the Character Movement's **Max Step Height** and **Walkable Floor Angle**. A step is reachable when it can be walked onto over known ground (the feet's plants and the sampled terrain), or when its height fits within the steps that fit on the way. Ground steeper than the walkable angle keeps its height but counts as flat, so the foot isn't tilted onto a wall.

## Runtime functions

All on the **Next Step Predictor** component (category StepAhead):

| Function | Use |
|---|---|
| Set Tracing Enabled (Enabled, Blend) | Turn placement off, e.g. during a cutscene, a montage that shouldn't adapt, or while mounted. The feet blend back to the animation, and blend in again when it is turned back on. |
| Set Trace LOD Override / Clear Trace LOD Override | Force a Trace LODs entry, e.g. a cheaper one for background characters. |
| Reset Traces | Forget the reused traces after a teleport or a level geometry change. |
| Set Mesh | Use another mesh (e.g. one added at runtime). |
| Get Terrain / Get Step Terrain | The step the character is taking: flat, slope or stairs, with step height, distance and angles. Useful for driving animation selection, such as stair animations. |
| Get Next Step / Get Next Steps | The traced ground under the predictions. |
| Is Next Step Reachable / Get Next Step Reachability | Whether the Character Movement can get onto a foot's next step, and why not. |
| Get Placement Target Height / Get Lowest Placement Target Height | The height each foot blends toward, e.g. for your own pelvis or camera handling. |
| Get Tracing Weight | How much of the placement is applied (0..1) while tracing blends out or in. |
| Is Off Ground, Is Facing Aligned, Get Facing Angle, Is Strafing | State of the last update. |

## Debugging

Console commands (Debug and Development builds only):

| Command | Shows |
|---|---|
| `a.StepAhead.Debug.All 1` | Next and previous steps, placement, root and traces together. |
| `a.StepAhead.Debug.NextStep 1` | The step each foot will land on (green). |
| `a.StepAhead.Debug.PreviousStep 1` | The step each foot lifted off from (blue). |
| `a.StepAhead.Debug.Placement 1` | The target bones after placement (orange), with a ghost trail (`a.StepAhead.Debug.TrailLength`). |
| `a.StepAhead.Debug.Root 1` | The root bone (magenta) and the component location (white). |
| `a.StepAhead.Debug.Traces 1` | The ground traces (yellow; red without a hit) and their hits and normals (cyan). |
| `a.StepAhead.Debug.Terrain 1` | Each foot's step: flat (silver), slope (green) or stairs (pink), with its height and angles, and the sampled terrain in between. |
| `a.StepAhead.Debug.Reachability 1` | The reachability checks: capsule sweeps to each step (lavender, red where blocked). |
| `a.StepAhead.Debug.TraceStats 1` | A panel with the queries each component makes per step and per second. |
| `a.StepAhead.Debug.Curves 1` | A graph per foot of the planted (green) and step progress (orange) curves and the placed height (`a.StepAhead.Debug.CurveSeconds` sets its length). |
| `a.StepAhead.Debug.LogJumps <cm>` | Logs every placement update whose height jumps by more than this, with its inputs. |
| `a.StepAhead.Debug.TraceLODOverride <n>` | Force a Trace LODs entry on every character (-1 = off). |
| `a.StepAhead.Debug.HoldStepProgress <0/1>` | Override Hold Step Progress on every character. |

Combine with `slomo 0.2` to watch single steps. StepAhead logs to the `LogStepAhead` category.

## Performance

- A planted foot reuses its trace, and a swinging foot is retraced only after its prediction has moved **Retrace Radius**. A walking character usually needs a few traces per foot per step.
- Traces are asynchronous by default.
- Characters that weren't rendered recently trace nothing (**Only When Rendered**). Distant LODs trace nothing with the default Trace LODs.
- Dedicated servers trace nothing: the component doesn't tick there. Foot placement is cosmetic, so it isn't replicated.
- The placement runs on the animation worker threads and reads the component's results through a shared snapshot, without copying them.

## Troubleshooting

**The feet don't move.**
- Check that the prediction bones exist on the mesh the character uses (open the skeleton tree and look for `nextstep_*`).
- Check that the playing animations were baked (their curves list `Planted_*` and `StepProgress_*`).
- Check that the names in **Feet** match.
- Turn on `a.StepAhead.Debug.NextStep`: no markers means no prediction data.

**The feet move, but the legs don't follow.** Place the node before your leg IK, and make sure the IK uses the **Target Bones** as its targets.

**In a Control Rig, every foot stays planted.** Import the `Planted_*` and `StepProgress_*` curves into the rig (see [Control Rig unit](#control-rig-unit)).

**A foot doesn't step onto a ledge.** The step is probably unreachable for the Character Movement. Check `a.StepAhead.Debug.Reachability`, or the movement's **Max Step Height**.

**The feet go back to the animation while turning sharply.** That is the Facing Check. Raise **Max Angle**, or turn it off if your character turns instantly.

**The feet drop while jumping or on custom movement modes.** That is **Only On Ground**. For a custom ground mode, override `IsMovingOnGround()`.

**After a teleport the feet blend from the old spot.** Call **Reset Traces**.

## Limitations

- **Baked locomotion only.** Animations must be baked; un-baked animations play with no placement.
- **A mixed blend can look abrupt on stairs.** Stopping by blending a run straight into an idle (without a stop animation) can still look abrupt on stairs.
- **Moving ground is followed, not predicted.** Ground on a moving platform is traced where the platform is, and found ground moves with it. A platform that moves fast between traces can be a frame behind.
- **Windows only** for now.
