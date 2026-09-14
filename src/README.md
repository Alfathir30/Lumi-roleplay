# Lumi — architecture notes for agents

A 3D VRM anime AI companion ("Lumi", formerly aoi) with chat, a simulated daily
life, wardrobe/shop, photos, achievements and an ending. Perchance generator
name: `aoi-official`.

If you only read one thing: **almost the whole app is `index.html`.** `main.pjs`
is a thin config/import file. Line numbers drift constantly (12k+ lines) — grep
for the identifier names below instead of trusting positions.

## Files
| path | role |
| --- | --- |
| `index.html` | The entire app: markup + one big `<style>` block + one big classic `<script>` (all game logic; top-level names are globals) + one `<script type="module">` (three.js / VRM rendering, exposed as `window.aoi3d`). |
| `main.pjs` | `$meta`, plugin imports (`ai-text-plugin`, `text-to-image-plugin`, `kv-plugin`, `upload-plugin`, `server-plugin`), a few config lists. |
| `src/anim/*.vrma` | VRMA animation clips. Clip names are the file stems; play them with `window.aoi3d.play("agreeing")`. See `VRMA-GUIDE.md`. |
| `src/vendor/piper/*` | Local text-to-speech (Piper voice model/config). |
| `src/TODO.md` | Durable roadmap (unbuilt features + user's own words). |
| `src/VRMA-GUIDE.md` | How the animation clips were sourced/retargeted. |

## System map (grep these)
- **Persona / prompting** — `const A = {`, `personaSys()`, `buildPromptParts()`,
  `memIntro`, `taskExtract`. `personaSys()` does the `\baoi\b → charName`
  substitution that keeps the persona greppable. Keep the prompt prefix stable
  (prefix caching matters for reply latency).
- **Chat** — `send()`, `addMsg()`, `spontaneousReply()`, `isAsleepNow()`,
  `relationMood()` (mood gates many behaviours, e.g. intimate sessions only start
  when `relationMood() === "warm"`).
- **Life simulation** — `lifeNow()`, `startLifeTick()`, `renderLifeChip()`,
  `lifeRole`, `actBias`, `inPublicNow()`, `publicStripGate()`.
- **Wardrobe / shop** — `OUTFIT_PRESETS`, `applyOutfitPreset()`,
  `renderOutfitPresets()`, `ownedItems`, `equipped`, `clothesOff`.
- **Photos** — `parsePhotoRequest()`, `PHOTO_*_RE`, `sendRequestedPhoto()`,
  `takeSelfie()`, `captionFor()`, `INTIMATE_CAPTIONS`. Photos live in `photos`
  (`{url, cap, ts}`) and are thumbnailed in the album popovers.
- **Intimacy (sex / masturbation sessions)** — `intimacyIntent()`,
  `startIntimacy()`, `advanceIntimacy()`, `intimacyClimax()`, `endIntimacy()`,
  `intimacyDecorate()` (moans), `intimacyContextLine()`, the `SEX_START_RE` /
  `MAST_START_RE` / `CLIMAX_RE` / `INTIMACY_END_RE` / `INTIMACY_HARD_RE` /
  `INTIMACY_SOFT_RE` / `INTIMACY_HOT_RE` regexes, `INTIMACY_MAX_HEAT` (6),
  `MOAN_SHORT/LONG/GASP`, `MOAN_*`. Session state is `intimacy`
  (`{kind, heat, pos, startedAt, lastAt, climaxes, round}`), persisted under the
  kv key `intimacy` (30-minute staleness window in `loadIntimacy()`).
  Semantics: a session starts on a sex/masturbation keyword; `heat` ramps per
  turn (hard +2 / hot +1 / soft −1, clamped 1–6) and drives moan length;
  **climax only fires when the message matches `CLIMAX_RE`** while a session is
  live.
- **Climax meter (HUD)** — `climaxMeterEl()`, `renderClimaxMeter()`,
  `flashClimaxMeter()`, `hideClimaxMeter()` + the `.climaxMeter` CSS. Shows only
  while a session is live, fills with `heat`, bursts on climax. Desktop:
  `right:16px; top:44px`. Phone (`max-width:640px`): `left:10px;
  bottom:calc(48vh + 18px)` so it lines up with `#camDock` and never collides
  with the right-hand HUD column (`#relMeter`, `#thermoW`, `#devMsgBtn`) — the
  thermometer's collapsed/expanded state changes what is free on the right.
- **Achievements** — `ACH_DEFS`, `checkAchievements()`, `unlockedAch`,
  `renderAch()`, `showDopamine()`. Lewd ones: `firstTime`, `climax1`, `climax10`,
  `intimatePhoto1`, `intimatePhoto10`.
- **Stats / persistence** — `stats`, `saveState()`, `saveStatsSoon()`, `KV()`
  (kv-plugin). `saveState()` lists every persisted key — add new state there.
- **Ending** — `ENDGAME_RE`, `endgameReady()`, `endgameStats()`,
  `completeGame()`, `showEnding()`, `.ending` CSS. Auto-triggers at
  `relationshipLevel >= 4.5 && stats.msgCount >= 150 && unlockedAch.length >= 10`.
- **3D / rendering** — `window.aoi3d` (module script): `play()`, `flash()`,
  `burst()`, `clothing`, `boneNode()`. `#sceneCtn`, `#camDock`, `html.perfLite`
  (low-end fallback that drops backdrop blur).

## Testing / verifying (learned the hard way)
- Edits only reach the live page via `page_refresh`; then verify with
  `page_eval`. Check `syntaxErrors` / `perchanceErrors` in the result.
- Layout work: `set_viewport_size` to **390×844** and **1280×800** and check
  `getBoundingClientRect()` overlaps against the other fixed HUD elements
  (`#relMeter`, `#thermoW`, `#devMsgBtn`, `#camDock`, `#lifeChip`) — that is
  cheaper and more reliable than screenshots.
- This page is heavy (VRM + TTS). **Full-page `snapshot.js` captures frequently
  trip the 15 s "page frozen" watchdog**, especially right after a reload while
  the VRM is still loading. Prefer `snapshot.capture(element, {scale:2})` on the
  specific subtree, wait several seconds after `page_refresh` first, and expect
  occasional freezes (refresh to recover).
- **Don't drive the game state carelessly.** Calling `startIntimacy()` /
  `intimacyClimax()` directly will (a) increment `stats.sessions` / `stats.climaxes`
  and persist them, (b) **unlock achievements** via `checkAchievements()` (so it
  keeps firing from timers even if you stub the global — always re-check
  `unlockedAch` before you finish), and (c) inside `intimacyClimax()` roll a 55 %
  chance of an unprompted "afterglow" photo through `sendRequestedPhoto()`, which
  really generates an image and posts it to the user's chat/album. Serialise
  `stats`, `unlockedAch` and `intimacy` before a test and restore them after, and
  blank `pendingAction` (it leaks into the user's next real reply).
- Keep the user's real save intact: restore `stats` + `intimacy`, then call
  `saveIntimacy()` and `saveStatsSoon()`/`saveState()`; verify the counters match
  what you captured before reporting done.
