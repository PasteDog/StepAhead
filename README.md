# StepAhead

**Predictive foot placement for Unreal Engine.** StepAhead bakes where each foot will land next into your animations. It then places the feet on the real ground before they land, so stairs, slopes and ledges don't make the feet snap at contact. It works from the Animation Blueprint, Control Rig or Blueprints.

This repository holds the plugin's documentation and its issue tracker. The plugin itself is sold on Fab; the listing link will be added here once it is live.

## Documentation

- **[User Guide](https://pastedog.github.io/StepAhead/)**: setup, the bake, every setting, debugging and troubleshooting.
- The same guide ships inside the plugin, at `Plugins/StepAhead/Docs/UserGuide.md`.

## Quick start

1. **Bake:** open **Tools → StepAhead**, add one rule per foot (mesh, foot bone, animation folder) and click **Bake Next Steps**.
2. **Add:** add a **Next Step Predictor** component to your character and list the feet (foot bone and IK target bone).
3. **Place:** add the **Predictive Foot Placement** node to your AnimBP, or the unit to your Control Rig, before your leg IK.

The [Quick start section of the guide](https://pastedog.github.io/StepAhead/#quick-start) walks through it on the UE5 mannequin.

## Support

- **Found a bug?** [Open a bug report](https://github.com/PasteDog/StepAhead/issues/new?template=bug_report.yml).
- **Have an idea?** [Suggest a feature](https://github.com/PasteDog/StepAhead/issues/new?template=feature_request.yml).
- **Stuck on setup?** Check [Troubleshooting](https://pastedog.github.io/StepAhead/#troubleshooting) first, then [ask a question](https://github.com/PasteDog/StepAhead/issues/new?template=question.yml).

Search the [existing issues](https://github.com/PasteDog/StepAhead/issues?q=is%3Aissue) first; your question may already have an answer.

## Compatibility

| | |
|---|---|
| Engine versions | Unreal Engine 5.4, 5.8 |
| Platforms | Windows (Win64) |
| Dependencies | Control Rig (engine plugin) |
