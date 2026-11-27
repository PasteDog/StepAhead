---
title: StepAhead User Guide
---

<!-- Interactive comparison (docs/compare/). Keep this block above the guide when copying UserGuide.md in. -->
<div style="position: relative; width: 100%; padding-bottom: calc(56.25% + 52px); margin-bottom: 0.5rem;">
<iframe src="compare/embed.html" title="Default foot IK vs StepAhead: drag the line to compare" allow="autoplay" style="position: absolute; inset: 0; width: 100%; height: 100%; border: 0;"></iframe>
</div>

*Drag the line to compare default foot IK with StepAhead. [Open full page](compare/).*
<!-- End of comparison -->

# StepAhead: Predictive Foot Placement

StepAhead places a character's feet on the real ground *before* they land. Most foot IK reacts to the ground under the foot. StepAhead knows where each foot will land next, because that spot is baked into your animations, and moves the swinging foot toward the ground there as the step progresses. On stairs, slopes and ledges the foot rises or drops in time, instead of snapping at contact.

Supported engine versions: Unreal Engine 5.4 and 5.8. Platform: Windows (Win64).

## At a glance

- **Set up in three steps:** bake your animations, add one component, add one node before your leg IK. No curves to author, no markers to place, no new IK rig. The defaults work for typical locomotion. See [Quick start](#quick-start).
- **Few traces:** each foot's landing spot is traced once and reused, instead of every foot every frame. See [Few traces](#few-traces).
- **Blends with everything:** the predictions are baked into the animations, so blend spaces, montages, layered blends and motion matching blend them with the pose.

## Contents

1. [At a glance](#at-a-glance)
2. [How it works](#how-it-works)
3. [Quick start](#quick-start)
4. [Baking your animations](#baking-your-animations)
5. [The Predictive Foot Placement component](#the-predictive-foot-placement-component)
6. [Placing the feet: AnimBP, Control Rig or Blueprint](#placing-the-feet)
7. [Settings reference](#settings-reference)
8. [Runtime functions](#runtime-functions)
9. [Debugging](#debugging)
10. [Performance](#performance) ([Few traces](#few-traces))
11. [Troubleshooting](#troubleshooting)
12. [Limitations](#limitations)

## How it works

**The bake (editor)** finds where each foot is planted in every frame of an animation and writes three things into it:

| Data | Name (default) | Meaning |
|---|---|---|
| Bone | `nextstep_<foot>` | Where the foot is planted now, or will be planted next, relative to the character. |
| Curve | `Planted_<foot>` | 1 while the foot is planted, 0 in the air. |
| Curve | `StepProgress_<foot>` | 0 while planted, rising to 1 when the foot lands. |

The bone is added to your skeletal mesh as a leaf bone.

**The runtime (game):** the *Predictive Foot Placement* component traces the ground under the baked bones. The *Predictive Foot Placement* node (AnimBP or Control Rig) moves each foot's IK target from the ground it lifted off toward the ground it will land on, by its step progress. Your leg IK solves the legs to those targets.

## Quick start

The bake runs once per skeleton; each character then needs steps 2 and 3. These steps use the UE5 mannequin (`SKM_Manny` / `SKM_Quinn`, feet `foot_l` / `foot_r`, IK targets `ik_foot_l` / `ik_foot_r`); use your own names if they differ.

1. **Bake.** Make sure your looping cycles have [Loop](#looping-animations) on. Open **Tools → StepAhead** and add one rule per foot:
   - **Skeletal Mesh**: `SKM_Manny`.
   - **Foot Bone Name**: `foot_l` in one rule, `foot_r` in the other.
   - **Folder**: the folder with your locomotion animations.

   Click **Bake Next Steps**, then save the changed animations and meshes. **Save to Project Settings** bakes new animations on import from then on.
2. **Add the component.** Add a **Predictive Foot Placement** component to your character Blueprint. Under **Feet**, add **Foot Bone** `foot_l` / **Target Bone** `ik_foot_l` and `foot_r` / `ik_foot_r`. Optionally set **Pelvis Bone** to `pelvis`.

   ![The Add Component menu with Predictive Foot Placement under StepAhead](https://pastedog.github.io/StepAhead/images/predictivefootplacement-component-add.jpg)

   ![The Feet list with foot_l / ik_foot_l and foot_r / ik_foot_r](https://pastedog.github.io/StepAhead/images/predictivefootplacement-component-feet-details.jpg)
3. **Place the feet.** In the AnimGraph, add **Predictive Foot Placement** after your locomotion pose, followed by your leg IK (e.g. **Two Bone IK** or **Leg IK**) to the `ik_foot_*` bones.
4. **Test.** Walk up stairs and slopes. `a.StepAhead.Debug.All 1` shows what is traced.

Every other setting is optional tuning.

## Baking your animations

### The StepAhead panel

**Tools → StepAhead** holds a list of **rules**, usually one per foot. Each rule bakes one foot into the animations of one skeleton:

| Setting | Meaning |
|---|---|
| Enabled | Whether **Bake Next Steps** runs this rule. |
| Skeletal Mesh | The mesh the prediction bone is added to; animations of its skeleton are baked. |
| Add To All Compatible Meshes | Also add the bone to every other mesh of the skeleton (e.g. Manny and Quinn). On by default. |
| Foot Bone Name | The foot to predict. |
| Next Step Bone Name | The prediction bone's name. None: `nextstep_<foot>`. |
| Parent Bone Name | The prediction bone's parent. None: the root. |
| Folder | Only bake animations in this folder. Empty: all folders. |
| Plant Settings | How plants are detected (see below). |
| Planted Curve Name / Step Progress Curve Name | None: `Planted_<foot>` / `StepProgress_<foot>`. |

![The StepAhead panel with one rule per foot on SKM_Manny](https://pastedog.github.io/StepAhead/images/stepahead-bake-animations-editor.jpg)

*One rule per foot. The grey text after an empty name shows the name used.*

**Bake Next Steps** adds the prediction bone to the mesh(es) and skeleton (open editors of that skeleton close and reopen), then bakes every matching animation. You are asked before existing data is overwritten.

Save the animations and meshes afterwards. Adding a bone marks every loaded animation of the skeleton as changed.

> **Tip:** Use source control or a backup before the first bake: it changes your meshes, skeleton and animations.

### Plant detection settings

| Setting | Default | Meaning |
|---|---|---|
| Height Tolerance | 3 cm | How far above its lowest point the foot still counts as on the floor. |
| Max Ankle Height | 20 cm | A footstep whose lowest point is higher than this above the root is in the air, not a plant. |
| Speed Threshold | 20 cm/s | The fastest horizontal foot speed that still counts as planted. |
| Min Contact Frames | 2 | Shorter contacts are ignored. |
| Step Progress Basis | Distance | Step progress rises with the foot's horizontal travel (Distance) or with time. |

Detection works on in-place and root motion animations, on flat ground, slopes and stairs. In clips where the foot never lands (jumps, falls), the bone follows the foot and both curves stay at 0.

### Looping animations

The bake reads each animation's **Loop** setting (Asset Details → Animation) to know the clip continues from its start. Without it, the step across the loop point gets no next plant and is predicted wrong. Turn **Loop** on for every looping cycle before baking, and bake again after changing it.

![Asset Details of a run cycle with Loop turned on](https://pastedog.github.io/StepAhead/images/animation-asset-setup.jpg)

*A run cycle with **Loop** on. The timeline shows the baked curves: planted (rectangles) and step progress (ramps) per foot.*

### Bake on import

**Save to Project Settings** stores the panel's rules under **Project Settings → Plugins → StepAhead** (`Config/DefaultEditor.ini`). While **Bake On Import** is on, animations imported or reimported for a rule's skeleton and folder are baked automatically.

![Project Settings, Plugins, StepAhead: Bake On Import and the bake rules](https://pastedog.github.io/StepAhead/images/stepahead-project-settings.jpg)

### Animations that suit the bake

Locomotion with clear foot plants: walks, jogs, runs, starts, stops, pivots and strafes. Each foot needs at least one plant in the clip.

StepAhead works best when the animation moves at the character's in-game speed: root motion, distance matching, or animations matched to the movement speed. Otherwise the predictions slide away from where the feet really land (see [Troubleshooting](#troubleshooting)).

## The Predictive Foot Placement component

Add one per character. It uses the owner's first skeletal mesh (or **Mesh**), ticks after the mesh has animated, and is the only place the ground is traced. It:

- **Traces only when a step moves:** a swinging foot's landing spot and the ground under a planted foot are traced once and reused until they move, so most frames trace nothing. See [Few traces](#few-traces).
- **Blends to the animation off the ground** (jumping, falling, flying): **Only On Ground**.
- **Ignores the predictions while the character is still turning**, since they point the wrong way: **Facing Check**.
- **Leaves unreachable steps to the animation**, so a foot doesn't reach for a ledge the character won't climb: **Check Reachable**.
- **Lifts the foot over obstacles between steps**, up to the Character Movement's **Max Step Height**, adding only what the animation's own swing doesn't clear: **Step Terrain**.
- **Moves with moving ground** (lifts, moving platforms).

## Placing the feet

All three ways run the same placement code.

### AnimBP node

**Predictive Foot Placement** (search "StepAhead" or "Predictive" in the AnimGraph) finds the component on the owning actor, or takes it from the hidden **Predictor** pin.

- Put it after the locomotion pose and before your leg IK, which must pull the feet to the target bones.
- It moves each **Target Bone** by the ground height and tilt, and the **Pelvis Bone** (if set) down by the lowest foot.
- The target bones must follow the feet in the animation, as the mannequin's `ik_foot_*` bones do. Otherwise, copy the foot bones onto them first (**Copy Bone**).
- **Alpha** blends the whole effect.

![AnimGraph: Local To Component, Predictive Foot Placement, Leg IK, Component To Local](https://pastedog.github.io/StepAhead/images/predictivefootplacement-animbp-setup.jpg)

*Predictive Foot Placement in component space, followed by Leg IK.*

### Control Rig unit

**Predictive Foot Placement** (category StepAhead) does the same in a Control Rig, in **Forward Solve** before the leg IK.

- **Import the curves.** A Control Rig only receives curves that exist in its hierarchy: import `Planted_*` and `StepProgress_*` from the skeleton's curve list (the bake adds them there). Without them every foot counts as planted, and the unit logs a warning.
- **Propagate To Children** moves the target bones' children with them.
- **Outputs:** **Valid** (component found, every foot placed), **Offsets** (height and tilt per foot, in the order of **Feet**), **Lowest Height** (for your own pelvis handling).

![Control Rig: Predictive Foot Placement followed by Full Body IK with the ik_foot bones as effector targets](https://pastedog.github.io/StepAhead/images/predictivefootplacement-controlrig-setup.jpg)

*Predictive Foot Placement followed by Full Body IK, with the IK foot bones as effector targets.*

### Blueprint

For custom setups, the **StepAhead** Blueprint library's **Update Predictive Foot Placement** takes, per foot, a state variable you keep, the component's **Get Next Step For Placement**, and the foot's step progress and planted curves. It returns a height and tilt offset; finish with **Clamp Foot Offset**, **Weight Foot Offset** (with **Get Tracing Weight**) and **Apply Foot Offset**.

## Settings reference

The tables list the defaults; the screenshots show a configured character, so a few values differ.

### Next Step

![Next Step settings: Feet, Only On Ground and Mesh](https://pastedog.github.io/StepAhead/images/predictivefootplacement-component-feet-details.jpg)

| Setting | Default | Meaning |
|---|---|---|
| Feet | — | The feet to place: **Foot Bone** (the baked foot) and **Target Bone** (the bone moved, e.g. `ik_foot_l`). Bone and curve names default to the bake's; **Ankle Height** (advanced) to the reference pose. |
| Mesh | owner's first skeletal mesh | The mesh with the prediction bones. |
| Only On Ground | on | Only place the feet while `IsMovingOnGround()` is true; override it for custom movement modes. |

### Tracing

![Tracing settings of the Predictive Foot Placement component](https://pastedog.github.io/StepAhead/images/predictivefootplacement-component-tracing-details.jpg)

| Setting | Default | Meaning |
|---|---|---|
| Trace Settings → Trace Channel | Visibility | The channel the ground is traced on. |
| Trace Settings → Trace Up / Trace Down | 50 / 75 cm | How far above and below each prediction to trace. |
| Trace LODs | 3 entries | How to trace at each mesh LOD (below). |
| Tracing Enabled | on | Off blends the feet back to the animation (**Set Tracing Enabled** at runtime). |
| Only When Rendered | on | Only trace while the mesh was rendered recently. |
| Facing Check | on, 15° / 10° | Ignores the predictions while the character is turning (off by more than **Max Angle**, back within **Resume Angle**). **Response**: **Hold Previous Step** or **Disable Tracing**. |
| Trace LOD Override | -1 | Always use this Trace LODs entry; -1 = by mesh LOD. At runtime: **Set Trace LOD Override**. |
| Tracing Blend In / Out Time | 0.25 s | How fast the feet blend in and out when tracing turns on or off. |

![Facing Check settings: Enabled, Max Angle, Resume Angle, Response](https://pastedog.github.io/StepAhead/images/predictivefootplacement-component-facingcheck-details.jpg)

**Trace LODs:** one entry per mesh LOD; later LODs use the last entry.

![Three Trace LODs entries: Box, Line and Disabled](https://pastedog.github.io/StepAhead/images/predictivefootplacement-component-tracing-lod-details.jpg)

| Setting | Meaning |
|---|---|
| Shape | **Box** (finds a stair's top at its edge), **Sphere** (highest ground around the point), **Line** (cheapest; can miss stair edges) or **Disabled** (no tracing; the feet blend to the animation). |
| Trace Radius | Sphere radius or box half size. |
| Retrace Radius | How far a prediction may move before it is traced again; 0 = every frame. |
| Async Traces | Read the traces a frame later instead of blocking the game thread. |
| Step Terrain | How the ground between steps is found: **Obstacle Search** (finds each obstacle and its top, up to **Max Searches** points; **Search Clearance**: how far above the step's line it searches), **Other Foot** (the other foot's plant; no traces), **Under Foot** (one trace per frame ahead of the foot, between **Under Foot Start** and **End** of the step), **Substeps** (**Substep Count** evenly spaced traces) or **None**. |
| Check Reachable | Leave steps the Character Movement can't get onto to the animation. |
| Check Path | Also sweep the capsule to each new step to find impassable walls in between (**Path Radius Scale**). |

Defaults: **LOD 0** a 5 cm box, retraced every 15 cm, with Obstacle Search; **LOD 1** a line, retraced every 30 cm, with Other Foot; **LOD 2+** no tracing.

### Foot Placement

![Foot Placement settings of the Predictive Foot Placement component](https://pastedog.github.io/StepAhead/images/predictivefootplacement-component-footplacement-details.jpg)

| Setting | Default | Meaning |
|---|---|---|
| Apply Height | on | Move the target bones to the ground height. |
| Apply Rotation | on | Tilt the target bones to the ground, up to **Max Align Angle** (30°). |
| Blend Time | 0.4 s | How long sudden ground changes take to blend in; 0 = instantly. |
| Foot Offset Limits | 50 / 50 cm | How far each foot may move down or up. |
| Step Over Apex Range | 20 / 80 % | Where in the swing the step-over arc may peak. |
| Step Over Max Raise | 2 | How much an obstacle near lift-off or landing may raise the arc, times its height. |
| Pelvis Bone | None | A bone (e.g. `pelvis`) moved down by the lowest foot, so the legs can reach. |
| Hold Step Progress | on | Step progress doesn't go back down until the foot is planted, so blends don't pull the foot back. |

### Terrain and Reachability

![Terrain settings: Flat Angle, Stairs Angle Tolerance, Min Stairs Height](https://pastedog.github.io/StepAhead/images/predictivefootplacement-component-terrain-details.jpg)

![Reachability settings: Path Radius Scale, Min Stair Tread Width, Max Stair Steps Look Ahead](https://pastedog.github.io/StepAhead/images/predictivefootplacement-component-reachability-details.jpg)

| Setting | Default | Meaning |
|---|---|---|
| Terrain Settings → Flat Angle | 3° | Ground tilted less than this counts as level. |
| Terrain Settings → Stairs Angle Tolerance | 10° | A step this much steeper than the slope can be stairs. |
| Terrain Settings → Min Stairs Height | 5 cm | The least height a step needs to count as stairs. |
| Min Stair Tread Width | 60 cm | The narrowest tread the character can stand on, to count the steps on the way; 0 = off. |
| Max Stair Steps Look Ahead | 2 | The most steps counted on the way to a step. |

Reachability uses the Character Movement's **Max Step Height** and **Walkable Floor Angle**. Ground steeper than walkable keeps its height but counts as flat, so the foot isn't tilted onto a wall.

## Runtime functions

On the **Predictive Foot Placement** component (category StepAhead):

| Function | Use |
|---|---|
| Set Tracing Enabled (Enabled, Blend) | Turn placement off and on, e.g. for cutscenes or while mounted; the feet blend to and from the animation. |
| Set Trace LOD Override / Clear Trace LOD Override | Force a Trace LODs entry, e.g. a cheaper one for background characters. |
| Reset Traces | Forget the reused traces after a teleport or level change. |
| Set Mesh | Use another mesh. |
| Get Terrain / Get Step Terrain | The current step: flat, slope or stairs, with height, distance and angles, e.g. to pick stair animations. |
| Get Next Step / Get Next Steps | The traced ground under the predictions. |
| Is Next Step Reachable / Get Next Step Reachability | Whether a foot's next step is reachable, and why not. |
| Get Placement Target Height / Get Lowest Placement Target Height | The height each foot blends toward, e.g. for your own pelvis or camera. |
| Get Tracing Weight | How much placement is applied (0..1). |
| Is Off Ground, Is Facing Aligned, Get Facing Angle, Is Strafing | State of the last update. |

## Debugging

![In-game debug view of seven characters walking down slopes: traces, steps, terrain labels, the Foot curves graph and the shared trace stats panel](https://pastedog.github.io/StepAhead/images/stepahead-ingame-debug-example.jpg)

*Seven characters walking down slopes: the traces (yellow; red: the previous trace, or no hit), terrain labels, the Curves graph for one character (left) and the TraceStats panel for all of them (top right): 2.5 ground traces per step.*

Console commands (Debug and Development builds only):

| Command | Shows |
|---|---|
| `a.StepAhead.Debug.All 1` | Next and previous steps, placement, root and traces. |
| `a.StepAhead.Debug.NextStep 1` | The step each foot will land on (green). |
| `a.StepAhead.Debug.PreviousStep 1` | The step each foot lifted off from (blue). |
| `a.StepAhead.Debug.Placement 1` | The placed target bones (orange), with a trail (`a.StepAhead.Debug.TrailLength`). |
| `a.StepAhead.Debug.Root 1` | The root bone (magenta) and the component (white). |
| `a.StepAhead.Debug.Traces 1` | The ground traces (yellow; red: the previous trace, or no hit), hits and normals (cyan). |
| `a.StepAhead.Debug.Terrain 1` | Each step's type (flat silver, slope green, stairs pink) with height and angles, the terrain in between (white) and the step-over arc (orange). |
| `a.StepAhead.Debug.Reachability 1` | The capsule sweeps to each step (lavender, red where blocked). |
| `a.StepAhead.Debug.TraceStats 1` | Queries per second and per step for all characters, and how many trace at which LOD. |
| `a.StepAhead.Debug.Curves 1` | Planted (green), step progress (orange) and placed height per foot for the viewed character (`CurvesNext` / `CurvesPrev` to switch, `CurveSeconds` for the length). |
| `a.StepAhead.Debug.LogJumps <cm>` | Logs placement updates whose height jumps by more than this, with their inputs. |
| `a.StepAhead.Debug.TraceLODOverride <n>` | Force a Trace LODs entry on every character (-1 = off). |
| `a.StepAhead.Debug.HoldStepProgress <0/1>` | Override Hold Step Progress on every character. |

Use `slomo 0.2` to watch single steps. StepAhead logs to `LogStepAhead`.

## Performance

### Few traces

Reactive foot IK traces under every foot every frame. StepAhead traces where each foot will land and keeps the result:

- **A trace is reused until its point moves** more than **Retrace Radius** (15 cm at LOD 0, 30 cm at LOD 1). With the animation matched to the movement speed, the landing spot stays put while the foot swings toward it, so a swinging foot is retraced only when the animation changes course, and a planted foot is usually traced once.
- **The terrain in between is cheap:** Obstacle Search runs once per step (one trace on clear ground, a few per obstacle); Other Foot traces nothing.
- **Reachability queries run once per new trace**, not every frame.
- **Characters you can't see trace nothing:** not rendered recently or at a distant LOD. Once blended out, their update only checks whether to trace again.

`a.StepAhead.Debug.TraceStats 1` shows the numbers in your own level.

### Threads and profiling

- Traces are asynchronous by default.
- Placement runs on the animation worker threads and reads the component's results without copying them.
- Dedicated servers don't trace (foot placement is cosmetic and isn't replicated).
- Unreal Insights (`-trace=cpu`) shows each part under `UPredictiveFootPlacementComponent::`, `FNextStepSnapshot::UpdatePlacement` and the node names.

## Troubleshooting

**The feet don't move.** Check that the mesh has the `nextstep_*` bones, the playing animations have the `Planted_*` / `StepProgress_*` curves, and the names in **Feet** match. `a.StepAhead.Debug.NextStep` shows no markers without prediction data.

**The prediction bones slide away from their spot in the world.** The animation's speed doesn't match the character's in-game speed, so the baked landing spots move with the character instead of staying put. This also costs extra traces, since a moving prediction is retraced. Use root motion or distance matching, or match the animation's play rate to the movement speed. With **Speed Warping**, make sure the prediction bones are warped along with the feet. `a.StepAhead.Debug.NextStep` shows the predictions.

**A foot jumps once per cycle.** The loop was baked with **Loop** off; see [Looping animations](#looping-animations).

**The traces work, but the legs don't follow.** Put the node before your leg IK, with the **Target Bones** as the IK targets.

**In a Control Rig, every foot stays planted.** Import the curves into the rig (see [Control Rig unit](#control-rig-unit)).

**A foot doesn't step onto a ledge.** The step is probably unreachable; check `a.StepAhead.Debug.Reachability` or the movement's **Max Step Height**.

**The feet go back to the animation while turning sharply.** That's the Facing Check: raise **Max Angle**, or turn it off if your character turns instantly.

**The feet drop while jumping or in custom movement modes.** That's **Only On Ground**; for a custom ground mode, override `IsMovingOnGround()`.

**After a teleport the feet blend from the old spot.** Call **Reset Traces**.

## Limitations

- **Baked locomotion only.** Un-baked animations get no placement.
- **Blending a run straight into an idle** (without a stop animation) can look abrupt on stairs.
- **Moving ground is followed, not predicted:** a platform moving fast between traces can be a frame behind.
- **Windows only** for now.
