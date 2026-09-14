# TODO / Roadmap

Durable feature notes — things we've agreed to add but haven't built yet. Keep
this file up to date as items land or change. Code references use function /
identifier names (line numbers drift); the architecture overview is in
`README.md`.

**Status (latest):** the user-driven-life batch landed — careers/roles,
unscheduled micro-events, show memory, appointments, more/less activity nudges,
the expanded 71-achievement list, and the auto-reply setup made a hidden unlock.
Follow-up landed too: **procedural careers** (any occupation the user names gets
a generated schedule — no fixed list), **shower vs. rinse** (showers only in the
morning + sometimes before bed, gated by `showerPlanned`/`once`; quick
`rinse` after gym/beach/friend's place), and new `beach`/`visiting` acts. See
README "User-driven life". Also landed: the **climax meter** — a small HUD card
that appears the moment an intimate session is detected from sex/masturbation
keywords, fills with session `heat` and bursts on climax (see README, "climax
meter"), with broader Indonesian trigger words (`sange`, `horny`, `ngetot`,
`ngewe`, `entot`, `pegang/elus/usap memek`).
Also landed: **consistent photo look** — every generated photo now opens with the
same canonical `charIntro()`/`CHAR_BODY` description in a fixed slot and repeats
it as `CHAR_TAIL` at the prompt end (start/end carry the most weight), with the
hedged "slim skinny … oversized breasts" wording removed (text encoders read
negation as its content) and body-drift words moved into `NEG_BASE`
(`skinny/athletic/straight hips/…`). A single persisted `photoSeed` (kv key
`photoSeed`, default on, toggle + "🔁 New look" in Settings → Photos) is reused
for all of her photos at `guidanceScale: 9`, so she's the same girl/body across
shots. Old album photos keep their old look; only new ones use the locked seed.
**Next up: Section A — Glasses Mode.**

---

## A. "Glasses Mode" — she puts on glasses and becomes a super-genius assistant

> **User's words:** *"a 'glasses mode' where she puts on a pair of glasses and
> suddenly becomes a super genius that can help with AI tasks and assist you with
> anything you need."*

### A1. The physical prop
- A real 3D glasses mesh, parented to the head so it moves with every animation.
  The model is a VRM 0.x with normalized bones; attach the group to
  `vrm.humanoid.getNormalizedBoneNode("head")` (helper `boneNode("head")`), with a
  head-local `position`/`rotation` offset onto the nose/ears and a scale matched to
  `modelFrame`. Iterate with a close-up `vision` render exactly like the
  rest-pose / hand-damping audits.
- Two ways to get the mesh, prefer the first:
  - **(a) Hand-built geometry** — two thin rounded lens rings + a bridge + temple
    arms from `THREE.TorusGeometry`/`TubeGeometry`/boxes. No asset, no upload,
    tiny, fully recolorable, and we control the pivot so it fits the face.
  - (b) Load a small glTF with the existing `GLTFLoader` — only if a generated
    mesh looks clearly better; must be CC0/shippable and credited (see README
    credits style).
- Look: dark metal frame + slightly transparent blue-tinted lenses (the classic
  "genius" read), optional faint emissive "smart display" strip.
- Put-on / take-off should be a beat, not a pop: fade/scale the prop in over
  ~0.3 s, and pair with a suitable existing clip (`pose` / `phone`) or a new
  `yui-*` clip if one reads as adjusting glasses.
- API + persistence, mirroring the `looks` / `thermostat` shape:
  `window.aoi3d.glasses = {on(), off(), toggle(), isOn(), setStyle(id)}`,
  flag in `localStorage["aoi_glasses"]` (simple) and/or kv.

### A2. The persona switch
- The persona lives in `A.sys` (`const A = {...}`, same block as `memIntro`,
  `taskExtract`, …) and is substituted by `personaSys()`.
  - Add `A.modes = { companion: <current A.sys>, genius: <new prompt> }` and a
    `currentMode`; `personaSys()` returns the active one (keep the
    `\baoi\b → charName` substitution).
  - **Genius prompt** register: precise and concise, structured answers
    (headings / numbered steps / code blocks when useful), asks a clarifying
    question when a request is ambiguous, states assumptions, admits uncertainty,
    cites concrete steps, no filler. Keeps her warmth and identity — she is
    still *her*, just fully switched-on. Drops the heavy text-speak.
  - Glasses **off** = byte-identical current behaviour. Same history, same
    `memories`, same relationship/mood — only the register/competence changes.
    She can acknowledge the switch ("ok, glasses on — what are we building?").
  - Note: flipping the system prompt invalidates the model's prefix cache; that's
    fine for a deliberate mode change (give it a short "thinking" beat).
- Wire the mode banner into `buildPromptParts()` and keep the **stable prefix
  first** (mode system text → memories → history → context lines → task) so
  repeated genius-mode turns stay prefix-cache-friendly. Genius mode may relax the
  "keep it fairly short" rule from `A.sys`.

### A3. Visual / behavioural cues while active
- Header/avatar gets a subtle "genius" treatment (thin cyan ring or a small
  ⌘/🧠 glyph) and a one-off toast on toggle; settings row to control it.
- Optional: cooler accent for her chat bubbles, "thinking"/"typing on a
  keyboard" idle, and a distinct TTS voice/rate for genius mode.
- In genius mode allow longer, structured replies and (later) let her act on the
  workspace tools in §B.

---

## B. The Workspace ("Desk") — a place to actually work on things, together

> **User's words:** *"some kind of space where you're able to work on stuff and
> she can help with lots of stuff."*

### B1. Layout
- A large overlay / full-screen panel `#deskPop` in the existing glass style
  (`#statPop` / `#imgPop` are the templates), opened from glasses mode and/or the
  header. Panes: **document editor** (centre), **task list** (right rail),
  **files / attachments** strip, and a **chat dock** for her (reuse the message
  renderer, or a parallel transcript scoped to the desk).
- Phones: collapse to tabs (Notes / Tasks / Files / Chat).
- Decide with the user whether the 3D scene stays visible (corner PiP) or the desk
  covers it; a "she's watching the desk" look-at state maps onto the existing
  gaze/lookat systems.

### B2. Tools to build (each persisted under `root.kv.aoi.<subfolder>`, `KV()` in
`index.html`)
- **Notes / documents** — autosaving markdown textarea (`contenteditable` optional),
  multiple docs with titles. AI actions on a selection: rewrite, summarise,
  expand, make bullet points, fix tone.
- **Task list** — add / complete / reorder; she can break a stated goal into
  tasks and tick them off as work proceeds (AI writes into the store).
- **Files & attachments** — drag-drop or paste images / text / PDFs; images feed
  the vision path (§C), text opens in the editor.
- **Prompt / scratch board** — prompt → result cards, side by side. Stream the
  result with an animated loading indicator and a stop button (`.stop()`), per the
  house rule for async output.
- **Doc / URL reader** — open a URL and read or annotate it together. Needs
  `superFetch` (see §D).
- **Agentic "do this for me"** — she plans (writes the task list), executes step
  by step with visible progress, and writes results back to the desk. Each step is
  a `generateText` call; keep a step log; stream partial output; show token/usage
  from the plugin's meta object where useful.
- API sketch: `window.aoiDesk = {open, close, notes:{list,create,read,write,delete},
  tasks:{list,add,toggle,remove,propose}, files:{add,list,read}, ask(prompt)}`.

### B3. "Help with lots of stuff"
- Selection actions: summarise, explain, rewrite, translate, extract tasks,
  generate code, critique, plan, turn notes into a doc.
- A visible "she's working" affordance near the streaming output + stop.
- Telemetry: `aoiTrack.count("desk_*")` for opens/actions — **new counter names
  must also be added to the server whitelist** (`trackPing` filter) or they're
  dropped, per README's stats section.

---

## C. Screen viewing — "even viewing your screen"

### C1. On-demand screen capture → vision
- **Verified available:** in the live preview this session,
  `navigator.mediaDevices.getDisplayMedia` exists, the context is secure, and the
  iframe's `display-capture` feature policy **allows** it. So the real path works.
- Implement a "👁 Look at my screen" button (desk / genius mode). On click
  (user gesture) request `getDisplayMedia({video:true})`, draw the current frame to
  a canvas, `canvas.toBlob` a JPEG (cap ~1280 px, quality ~0.8), and hand it to
  her with the existing vision path:
  `root.generateText({instruction: ["<question>", blob]})` — the same one-image
  Blob attachment already used for chat images (`A.imgNote`).
- Stop the stream tracks after grabbing unless "keep watching" is on. Show an
  explicit, always-visible "she can see your screen" indicator while live, with a
  one-tap Stop — privacy first.
- **Keep-watching mode** (optional): periodic grabs (20–60 s, or only on request),
  strictly opt-in, obvious indicator; skip near-identical frames (cheap diff) to
  save cost.
- Fallbacks if a browser blocks the picker: paste an image (desk paste handler),
  drag-drop a screenshot, or a file input. Also a **same-page snapshot** option —
  capture the app itself with the `.../files/snapshot.js` helper and hand that to
  her ("what's wrong with this screen?").

### C2. Screen-aware behaviours
- **Guide me** — she reads the screen and gives click-by-click instructions;
  optionally draw a translucent highlight box over her suggested region and show
  it back.
- **Error flow** — grab screen, ask vision "what does this error say / what's
  wrong here?", she explains and proposes a fix.
- Grabs stay local until the user asks; never silently upload screen frames.

---

## D. Shared plumbing / integration points
- Mode + desk state persist (`localStorage` for flags, kv `root.kv.aoi.<sub>` for
  docs/tasks) and load on boot alongside `applyLooks()` etc.
- `personaSys()` + `buildPromptParts()` are the two functions to extend for mode
  text and desk/file context — keep the model prefix stable.
- Settings: add a **Modes / Glasses** `setSection` to `#settingsPanel` (toggle +
  style picker + screen-access controls), plus a top-bar button once it's ready.
- New telemetry counters → add to the server whitelist (see the stats section of
  README).
- Add `superFetch = {import:super-fetch-plugin}` to `main.pjs` if/when we build the
  URL/doc reader.
- Verification: `vision`-check the glasses prop close-up and at distance;
  `vision`-check desk layouts at 390×844 and 1920×1080; drive every `generateText`
  path with a real call and confirm the loading indicator.
- Any downloaded model/mesh must be CC0/shippable, credited like the VRMA sources.

---

## H. "Sex appeal" slider — breast size (USER-OWNED, DO NOT IMPLEMENT)

> **User's words:** *"we want to introduce a 'sex appeal' slider which will change
> her breasts size. i will create the blender files or something to actually do
> that because its not that easy to just do that."*

- **Do NOT build this.** The user will author the Blender files / shape-key morphs
  themselves; this is recorded only so it isn't forgotten.
- When the user delivers the morph assets, the slider should blend between
  breast shape-keys (or apply a bone/morph scale) on the VRM, persist like the
  other appearance settings (`localStorage` / kv), and ideally feed the selfie
  prompt's figure description. Note aoi is a **VRM 0.x** model with two dedicated
  breast spring joints (see the hair-spring note in README) — those must keep
  their authored spring settings while any morph is driven.
- Until then: **no code, no placeholder UI.**

---

## E. Stretch ideas (later)
- **Pair programming** — she reads pasted code and proposes diffs in a diff pane.
- **Shared whiteboard / canvas** pane both of you can draw on (could share via the
  `server-plugin`, like the gallery).
- **Projects** — a project owns its docs/tasks/files/chat and can be switched; a
  home screen of recent projects.
- **Voice desk** — walkie-talkie conversation using the existing TTS + STT.
- She comments on the 3D scene she's standing in (partly possible already).
- **Export** the desk (docs + tasks) as markdown / JSON.

---

## F. Open questions for the user
- Persistent toggle, or a timed "focus session"?
- One shared conversation with her companion personality, or a separate assistant
  thread?
- Where should the desk live — full-screen overlay, resizable side panel, or a
  separate room/scene?
- How much screen access is comfortable: only when explicitly asked, or an opt-in
  "keep watching" mode?
- Different voice for genius mode?
- Which AI tasks do you do most, so the workspace nails those first?

---

## G. Suggested build order
1. Glasses prop + toggle + persistent flag (3D only), vision-checked.
2. Genius persona switch in `A` / `personaSys()` + header cue + settings row.
3. Desk shell: Notes + Tasks (kv-backed) + her chat dock + streaming helper.
4. Contextual "help me with this" selection actions.
5. Screen grab → vision ("look at my screen"), then optional keep-watching.
6. Files pane, URL/doc reader (`superFetch`), agentic task runner.
7. Polish: telemetry, mobile tabs, export, README/TODO updates.
