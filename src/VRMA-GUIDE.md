# .vrma animation files (aoi's motion system)

**All of aoi's body animations are real recorded `.vrma` clips** (VRM Animation —
a glTF/GLB binary with a `VRMC_vrm_animation` extension). The old hand-authored
procedural `ANIMS` table still exists in the code as an **empty object** — every
name in `listAnims()` (**66** of them) resolves to a file in `src/anim/`. The
recorded clips look dramatically better than the procedural poses, so all of
them were replaced.

**21 clips are currently outside the roster.** 17 were pruned after a retarget
audit: `nod, shake, bow, dismiss, excited, point, shrug, stretch, thanks, walk,
sadwalk, run, jog, standup, kneel, drink` all self-intersected on aoi (hands
clipping through the head or through each other, legs crossing mid-stride, parts
of the body folding away) and `shy` was a duplicate of `blush`. The other 4 —
`wave`, `agree`, `walkNeutral`, `laugh` — were deleted at the user's request
(each one visibly broke her hips). All of their files are still in `src/anim/`,
and the removed names are listed in a comment above `VRMA_ANIM`, so any clip can
be restored by re-adding one mapping line. `clap` was kept but fixed with
`VRMA_ROT_SCALE` (see §2).

The staging area is currently **empty** — the two candidate batches that were
staged in `src/anim/candidates/` and registered in `VRMA_NEW` (§5) have both been
fully vetted (9 of 26 promoted from the first, all 24 of the second), so nothing
is reachable only through the secret menu's "🆕 New VRMAs" preview right now.

This guide covers where the clips came from, how they're wired in, the
retargeting gotchas specific to aoi's model, and licensing.

> aoi is a **VRM 0.x** model, and most of these clips are **VRMA 0.x**.
> `createVRMAnimationClip` retargets any spec version onto aoi's own humanoid
> bones, so the source model only affects proportions (see gotchas).

---

## 1. Where clips came from (the actual files we ship)

Everything in `src/anim/` (~95 files) came from one of these, all committed to
the repo so the generator never hotlinks:

| Source | What we took | Notes |
|---|---|---|
| [`aikeyaorg/aikeya`](https://github.com/aikeyaorg/aikeya) | `idle`, `talking`, `greeting`, `spin`, + the `emote-*` set | **MIT** |
| [`not-elm`](https://github.com/not-elm) VRM samples | `idle-maid` | Apache-2.0 / MIT |
| [Quaternius](https://quaternius.com) via [`3dkit-online`](https://github.com/3dkit-io/3dkit-online) / three.js examples | `dance` | **CC0** |
| [`tk256ailab/vrm-viewer`](https://github.com/tk256ailab/vrm-viewer) | `emote-*.vrma` (wave/cheer/etc.) | **MIT** — see loader patch note below |
| [`Undi95/Hanami`](https://huggingface.co/Undi95/Hanami) (Overte + Rocketbox + Quaternius) | the walk/run/jog/sit batches (`walk`, `walk-slow`, `walk-fast`, `walk-back`, `walk-neutral`, `sadwalk`, `run`, `jog`, `sit-enter`, `sit-exit`, `sit-idle`, `sit-lean`, `sit-legs`) | mixed permissive (CC0/CC-BY); Overte & Rocketbox |
| [`yw0nam/YUI`](https://github.com/yw0nam/YUI) (Mate Engine) | `sleep.vrma`, `sulk.vrma` | **NON-COMMERCIAL + attribution** — swap before commercial use |

**Loader patch:** the `tk256ailab/vrm-viewer` clips shipped without a
`specVersion` inside their `VRMC_vrm_animation` extension, so three-vrm's loader
silently skipped them. The GLB JSON was patched to add `"specVersion":"1.0"` to
that extension; after that they load fine. Seven of the **first candidate
batch** (§5, the `へすい/rerofumi` pack) had exactly the same defect and were
patched the same way. If you add a clip and it "loads" but never plays, check
for a missing `specVersion` (the extension must contain it) before anything
else.

### Where to find more

- **Booth.pm** — the biggest source. Search `VRMA` or `VRM モーション`.
  Filter by 無料 (free) / VRM 0.x. Free + paid dances/idles/walk/emotes.
- **VRoid Hub** (hub.vroid.com) — mostly models, but some motion sets.
- **Nizima** (nizima.com) — motions category.
- **GitHub / HuggingFace** — search `vrma` for community collections; the
  official three-vrm test clip is at
  `https://raw.githubusercontent.com/pixiv/three-vrm/dev/packages/three-vrm-animation/examples/models/test.vrma`.
- Google: `vrma file free download`, `VRM モーション 無料`.

A `.vrma` is a GLB container, so you can inspect/patch its JSON with any glTF
tool. Quick sanity check before wiring one up: the official drag-and-drop viewer
at `https://pixiv.github.io/three-vrm/packages/three-vrm-animation/examples/dnd.html`
(drag a .vrm + .vrma; if it plays there, it'll work here).

## 2. How the clips are wired into aoi

The loader package (version must match the existing `@pixiv/three-vrm@3.x` line):

```js
import { VRMAnimationLoaderPlugin, createVRMAnimationClip } from "https://esm.sh/@pixiv/three-vrm-animation@3.1.5?deps=three@0.160.0";
```

The plugin is registered once on the app's `GLTFLoader`
(`loader.register(parser => new VRMAnimationLoaderPlugin(parser));`), then each
file is loaded like a model and retargeted:

```js
loader.load(VRMA_DIR + file, (gltf) => {
  const vrma = gltf.userData.vrmAnimations[0];
  const clip = createVRMAnimationClip(vrma, vrm);   // retargets onto aoi's bones
  // -> an AnimationMixer(vrm.scene) action drives it
});
```

The clip layer is **mixed on top of** the procedural pose in quaternion space so
it crossfades with the pointer-gaze/idle overlays. The relevant pieces in
`index.html`:

- `VRMA_ANIM` — action name → file. `VRMA_ALIAS` remaps legacy names the
  game/AI still ask for. `VRMA_LOOP` marks gaits/long poses that should play N
  repeat cycles instead of a single pass (so they don't freeze mid-stride).
  `VRMA_FACE` adds a matching facial expression per emote.
- `VRMA_IDLE_FILES` — the idle loops (`idle`, `idle2`, `idle3`, `idle5`),
  crossfaded at random 17–31 s intervals. (`happyIdle` and `idle-maid` were
  removed from the rotation on request; the files stay on disk.)
  `idle4.vrma` is intentionally **excluded** (its retarget twists both hands
  into claws — dropping individual tracks doesn't fix it). `VRMA_DROP_TRACKS`
  exists for partly-broken clips you want to keep (it drops whole limb tracks so
  they fall back to rest), but idle4's damage is in the hand tracks themselves,
  not a stray one. For a clip that's only *slightly* over-rotated,
  `VRMA_ROT_SCALE` is gentler — it **scales down** the rotation of matching
  tracks instead of dropping them: the live entry
  `{ "clap.vrma": { re: /Normalized_(Left|Right)_(arm|elbow|wrist)/, f: 0.75 } }`
  shrinks her arm/wrist swing to 75% so the clap's hands meet without passing
  through each other (min wrist separation 0.024 → 0.066; `f:1` untouched,
  `0` = rest). The same mechanism fixes the **clasped/folded-hand clips**, whose
  hand meshes fused into a single blob on this rig:
  `sit-idle.vrma`/`sit-lean.vrma`/`sit-legs.vrma f:0.84`, `sulk.vrma f:0.88`
  (in `sulk` the crossed forearms also sank into the torso). (`idle-maid.vrma`,
  which also fused, was removed on request.) Targeting ≥ ~0.20 m
  between the two hand origins gives a visible gap; see §2/README for the
  measured before/after distances. `aoi3d._dbg.rotScale` and
  `aoi3d._dbg.reload(file)` let you tweak an entry and re-load one clip
  (no page reload) to A/B render it. `VRMA_TALK_FILE` (`talking.vrma`)
  crossfades in while she's speaking.
- `playVRMAClip(name)` / `startVRMAEmote(name, entry)` — one-shot emotes clamp,
  then fade back to idle over ~0.5 s.
- `initVRMAClips()` — staggered preload at startup (emotes first, idles after,
  so the first frame isn't delayed). **Playing a clip before preload settles can
  silently no-op** — in tests, poll until `aoi3d.vrma().cached.length` stops
  growing before asserting a pose.
- `window.aoi3d.previewAllAnims()` cycles every clip; `aoi3d.dumpClip(name)` and
  `aoi3d.vrma()` expose cache/blend state for debugging.

### Gotchas specific to aoi

- **Hips translation is retargeted by hand.** three.js's `AnimationMixer`
  position-track binding onto `Normalized_Hips` is **unreliable** — it
  intermittently resolves to nothing, which made sit/sleep clips render standing
  while the rotations worked. The fix: `stepVRMAClips`/`stepStillClip` read the
  clip's own hips-position keyframes with a binary search (`sampleHipsPos`) and
  write the blended value onto BOTH the normalized hips node and the raw hips
  bone (`writeHips`), never trusting the mixer's position track. The old code
  that zeroed `hips.position.y` every frame (and on load) was removed — that was
  the bug.
- **The pose loop fights a naive mixer.** The pose system writes the normalized
  bones every frame, so a VRMA clip must be run *as the driver* for its duration
  (the app blends pose↔clip in quaternion space rather than running two
  competing writers).
- **Proportions.** Clips from a much taller/shorter source model can float feet
  or misplace hands. aoi's normalized rest hips are at y ≈ `1.453`. Prefer clips
  from similar-height female VRM 0.x models.
- aoi has **no `upperChest` bone** — the retarget skips it safely.
- `createVRMAnimationClip` logs one harmless warning about a missing
  `VRMLookAtQuaternionProxy`; ignore it. If a clip has a `lookAtTrack` you do
  want, add a `VRMLookAtQuaternionProxy` to `vrm.scene` and set
  `vrm.lookAt.autoUpdate = vrma.lookAtTrack != null`.

## 3. Adding a new clip

1. Drop the `.vrma` into `src/anim/` (no need to `upload_file` — `src/` ships
   with the generator).
2. Add a `name: "file.vrma"` entry to `VRMA_ANIM` in `index.html`
   (optionally `VRMA_FACE` for an expression, `VRMA_LOOP` for a looping
   gait/pose, `VRMA_ALIAS` for a legacy name, `VRMA_ROT_SCALE` to damp a
   slightly over-rotated limb).
3. Add the name to the AI prompt's inline animation list (the `sys:` string near
   the chat code — it's hand-maintained) and, for energetic clips,
   `GROOVE_BEAT` / `window.__aoiLight.onPlay`.
4. Reload. `listAnims()`, the input listener, and the `**command**` parser all
   read `VRMA_ANIM` automatically.

## 4. Licensing

Read each item's terms. Most free VRMA content permits use on any VRM model,
including commercially, but some restrict redistribution. The clips we ship that
do **not** allow commercial use are the `yw0nam/YUI` set — `sleep.vrma`,
`sulk.vrma` and the 22 promoted YUI motion clips (`yui-dance1/2…13`,
`yui-happy`, `yui-laugh`, `yui-calm`, `yui-idle1…9`, `yui-sit2/sit4`,
`yui-sitdown`, `yui-standup`, `yui-kneel1/kneel3`, `yui-walk`, `yui-jump`) — all
Mate Engine (Shiny) exports under non-commercial terms requiring attribution.
Replace every clip whose file starts with `yui-`, plus `sleep.vrma`/`sulk.vrma`
(still the lying-sleep `STILL_CLIP_FILE` and the sulk emote) before shipping a
commercial product. The first-batch voxavatar/3dkit clips are CC0. Keep creator
credit where required.

## 5. Candidate clips (staging — not approved)

`src/anim/candidates/` is where clips go for vetting before they join the
roster. It is **empty right now** — the second batch of 24 YUI clips was approved
and promoted, and the first batch of 26 was vetted before it. Candidates are
registered in the `VRMA_NEW` map in `index.html` (name →
`"candidates/<file>.vrma"`), which is
deliberately separate from `VRMA_ANIM`: candidates never appear in
`listAnims()`, the `**command**` parser, the input listener, the AI prompt, or
the idle rotation, and they are never preloaded at startup (they load lazily
when the preview reaches them). The owner vets them with the secret menu's
**"🆕 New VRMAs"** button (`window.aoi3d.previewNewVRMAs({hold, cap, gap})`,
which drives the same floating preview bar, mode line `new`). `VRMA_NEW_LOOP`
gives looping candidates a few repeat cycles so gaits/idles don't freeze.

### Approving / rejecting

- **Approve:** move the file from `src/anim/candidates/` up into `src/anim/`,
  add a `name: "file.vrma"` entry to `VRMA_ANIM` (plus `VRMA_FACE` / `VRMA_LOOP`
  / `VRMA_ROT_SCALE` as needed — §2/§3), add the name to the AI prompt's `sys:`
  animation list, and delete its `VRMA_NEW` line.
- **Reject:** delete the file and its `VRMA_NEW` line.

### The vetted candidate sets

**Second batch (2026-09), all from
[`yw0nam/YUI`](https://github.com/yw0nam/YUI) `public/motions/`** — 24 clips,
**all approved and promoted** into `VRMA_ANIM` (same **non-commercial +
attribution** terms as the shipped `sleep`/`sulk`; mostly Mate Engine (Shiny)
exports).

| Group | Clips (now in `VRMA_ANIM`) |
|---|---|
| Dances | `yui-dance2`, `yui-dance3`, `yui-dance4`, `yui-dance5`, `yui-dance6`, `yui-dance8`, `yui-dance9`, `yui-dance11`, `yui-dance12`, `yui-dance13` |
| Idle loops | `yui-idle2` … `yui-idle9` |
| Seated / transitions | `yui-sit2`, `yui-sitdown`, `yui-standup`, `yui-kneel3` |
| Gaits / misc | `yui-walk`, `yui-jump` |

The **first batch** (26 clips — 9 promoted, 17 rejected) is
documented in `src/README.md` §"Candidate VRMAs". Its sources were:
[`SanHsien/voxavatar`](https://github.com/SanHsien/voxavatar) (three explicitly
**CC0** BOOTH motion packs), `yw0nam/YUI`, the `pixiv/three-vrm` official test
clip, and `3dkit-online/3d-motion-avatar-viewer-cases` (**CC0**).

Anything that retargets badly onto aoi should be rejected — that is exactly what
this staging area is for.
