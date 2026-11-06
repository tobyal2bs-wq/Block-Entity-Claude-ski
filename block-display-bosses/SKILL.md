---
name: "block-display-bosses"
description: "Build any creature, boss, item or prop as a Minecraft block-display model (vanilla blocks only) from a picture or a short prompt, textured and animated, for CBosses/CQuests."
---

# Block-display models from a picture or a short prompt

Every model is made of vanilla block displays: no resource pack. The goal is a clean, readable model that looks like the reference, stands centred on the ground at the right size, has real limbs, looks alive, and looks **the same in game as in the preview**. Code lives in CBosses `tools/models/` (gen_models.py, anims.py, palette.py, roster.py, v9_minis.py, v10_fixes.py, v11_new.py, v12_final.py).

## 0. In game must match the preview (most important)

- A block display draws translation x leftRotation x scale x rightRotation. Turn a posed cube matrix into that with `Displays.transformation` (CBosses `com.cbosses.model`, CQuests `com.cquests.worldboss`):
  - a block that is only turned and stretched (square edges): rotation = the matrix's **columns divided by their lengths**, right rotation = identity, so the rotation stays steady from frame to frame and interpolation never spins a block the long way round;
  - a block skewed by a parent bone's squash while turned: an exact SVD (left rotation, scale, right rotation).
- Never use JOML `getNormalizedRotation` on a stretched block: it normalises rows, so every long arm, leg or plate is drawn at the wrong angle as soon as it turns and the boss falls apart in game while the preview looks fine.
- Compare both rotations in the "same transform, skip the update" check.
- BossDataTest `blocksShownInGameMatchTheModel` rebuilds every cube from its display transform and fails above 0.01 blocks. Keep it passing.
- ModelRig and Displays need only gson, joml and a stub `org.bukkit.util.Transformation`, so you can compile them locally with jars from Maven Central and check every model in every animation, even when Paper's repo is blocked.
- Keep pure helpers free of server registries, or the unit tests cannot load them.

## 1. What the user likes

- **Clean colour beats detail.** Big parts in one or two tones of one hue, on grain blocks. A patchwork of mixed-hue mottled cells looks broken; plain concrete everywhere looks like plastic.
- **Minis are simple.** No block minimum for minis (25+ is fine). Never pad to a count.
- **Real legs and arms**, nothing floating, unless it is a ghost or a plant by design.
- **Entrances:** the whole model grows in at once with a burst in its theme's colours and its roar (`entrance()` in gen_models). No blocks flying in one by one, no staggered builds, no rising out of the ground except the zombie (Rotting Giant, `CLIMBERS`). Clucksworth keeps her egg (shell bones).
- **No green sparkle particles** (HAPPY_VILLAGER). Nature uses cherry leaves, toxic uses slime and smoke.
- **Boss bar:** name and health only.
- **Change only what the user names.** Don't redraw models they have not called bad. When a redraw is rejected, restore the old one and redo only the named ones (`FIX` in v12_final).
- When the user says stop, stop and undo unreleased local changes.

## 2. Read the picture (or prompt) first

Write down: silhouette and proportions; palette (2–4 colours per part, one hue each); the one feature that makes it recognisable; what moves.

## 3. Build like Blockbench

- A few big boxes per part, integer sizes. Axes: x left, y up, z front. Use real mob proportions when based on a mob.
- Helpers: v11 `tb`, `legs2`, `arms2`, `boss_recipe`, `chapter()`, `remini()`. v12_final: `tb`/`T` (mottling at 0.3×), `face(...)`, `kid(...)`, `quad4(...)`, `ghost(...)`, `toadstool(...)`, `idle`/`sway`/`bob`; decorators `big`, `mini`, `remake`, `drop(old)`; `partner(...)` registers a standalone mini.
- Bones are added parent first.
- Held weapons: pivot at the hand, item pointing up, `m.hold`.
- Hoods and helmets as plates, never one box over the face. Long arms end in plain fists, no tiny fingers.
- People are people: a head on a real body, clothes, flat feet.

## 4. Texture, lightly

- Grain blocks: `texturize()` turns `X_concrete` into powder (not black or lit). Also wool, terracotta, mud, dirt, granite, diorite, andesite, tuff, logs, planks, bone block, moss. Add new blocks to palette.py.
- Mottling, if any: low rate, same hue, chunky cells. Seeds via `zlib.crc32(repr(...))`, never `hash()`.
- Colour traps: blue_terracotta renders purple, light_blue_terracotta lavender.

## 5. Faces

Use `face()`: lighter top row, eyes with a white catch light, brows, mouth, shaded bottom row. Face states are flat `plane="front"`, `shell=False` plates tagged `face:<state>`.

## 6. Alive, not posed

- Idles breathe, shift weight and look around. Loose parts get their own bone and lag.
- Bending necks and joints: one bone per segment with its own keys. `m.knees()` for IK legs.
- Check `only=` selectors hit the right bone; an old `{"hat": "head"}` rewrite made Mayor Grumble's head float off.

## 7. Size, centre, hitbox

gen_models `normalize()`: centroid to x = z = 0, lowest cube to y = 0, scale by class (mini 1.5–2.0, big 3.0–3.5, mythic 4.0–5.0); span = max(top, 0.7 × width); hitbox 0.9 if tall, else top/span (min 0.45). Never import gen_models (it runs the build). For CQuests world bosses keep the original vertical placement and change only scale, x/z centring and hitbox.

## 8. Roster and data

- roster.builders() sorts by module rank so newer modules win.
- Removing a boss: take it out of the roster lists (or `drop()` it), add it to `REMOVED` in v12_final (bosses.yml `removed:`; CQuests `RETIRED`), and update the finale and the tests. Servers drop removed ids on start.
- Chapter numbers are fixed per boss (`CHAPTER_NO`), because player progress is stored by chapter.
- Redesigned entries get a higher `revision` so servers take them and keep rewards and on/off.

## 9. Checks and pipeline

- Pipeline: `gen_models.py`, `tools/gen_bosses_yml.py`, `tools/gen_messages.py`, `tools/viewer/build_viewer.py`, `tools/gen_readme.py`. Render with `ZOOM=0.75 MODS=v12_final bash msheet.sh <ids>`.
- Pre-check sizes, centring and rig bones in Python; run the display-transform check (section 0).
- When the boss count changes, update BossDataTest, CQuests WorldBossDefTest and both smoke-test.sh files.
- Sync the working copy to git with tar excluding `.git`, `build`, `tools/models/tex` and `.github`.

## 10. Ops and shipping

- Ops: `/cbosses edit|enable|disable|mythic` and `/cbosses settings [page]` (dialog on 1.21.6+, chest before). Add a setting with one line in SettingsMenu.pages().
- All saves go through SafeFiles.
- Release only from a green Build run: copy the last release workflow (version, SOURCE_SHA, CHECKED_RUN, unit test count, title). Verify with `gh api repos/<owner>/<repo>/releases/tags/vX`.
- Commit as the user, with no Claude attribution.

## Lessons from what the user called bad

- "They come as random blocks, glitch and break": wrong rotation maths in game (section 0) plus staggered entrances. Check the in-game transform, not only the preview.
- "Textures are shit": mixed-hue mottling. "Whole white": plain concrete. Use clean grain blocks.
- "Too much detail broke it": minis padded to 150+ blocks.
- "All went back in time": reverting everything when only some were disliked.
- Floating heads, missing limbs, weapons under feet, hoods over faces: render and check.
- Two bodies glued into one boss confused players; the little ones work better as their own minis.
