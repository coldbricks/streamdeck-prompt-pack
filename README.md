# Programming a Stream Deck with an AI — the prompt pack

**Rendered version with the deck tour:** https://finallyvr.com/streamdeck-prompt-pack/

Plain-text companion to the build pack page. Send prompts one at a time, in order,
reading the reply before sending the next. If you only send two, send 1 and 3.

Written and verified on Windows. On a Mac, read section 0 first — the format laws
hold everywhere, the plumbing translates.

---

## 0 — If you're on a Mac

Everything format-level in this pack is platform-independent and stays true: the package layout, the package.json shape, the UUID case law, the hidden blank Default page, the Controllers/Settings/States shapes, the action UUIDs, the icon sources, and all of the GIF math.

The plumbing around it is Windows. Tell your AI to translate these four things and ground-truth each one on your machine during recon — never let it trust a path claim it hasn't read:

- The profile store is not under %APPDATA%. Expect it inside ~/Library/Application Support/com.elgato.StreamDeck/ — find the ProfilesV3 folder there in prompt 1 and confirm the layout matches before believing anything.
- There is no .cmd. Launcher wrappers become executable shell scripts (chmod +x); prove ONE fires from a key before building thirty.
- The quit-and-elevation notes in prompt 6 (--quit, taskkill, ShellExecute, WinError 740, staying non-elevated) are Windows plumbing. Find the clean quit on macOS (try osascript telling the app to quit), prove the process is actually gone, and keep the rest of the ritual exactly: quit → back up → install → verify → restart → verify.
- The PowerShell/BOM law doesn't apply; the argv-limit habit (cd into the frames dir, assemble with a glob) still does.

---

## 1 — Recon (send first)

I have an Elgato Stream Deck. I want YOU to program it by writing its profile files directly, not by me clicking around in the Elgato app. This first message is read-only recon — do not write or change anything yet.

Find and report:

1. My device. Read %APPDATA%\Elgato\StreamDeck\ProfilesV3\ — list every *.sdProfile folder, and for each one open manifest.json and report "Name", "Device.Model" (the model string, e.g. 20GBX9901 = Stream Deck + XL) and "Device.UUID" (it embeds vid/pid/serial).

2. The grid. From any page manifest, report the highest "col,row" key coordinate and how many Encoder (dial) slots exist. Confirm the exact key count so we never build a page that doesn't fit.

3. The pages. For each page folder: its "Name", how many keypad and encoder actions it holds, and the distinct action UUIDs in use (that tells us which plugins I already have).

4. My tools. Check whether python, magick (ImageMagick) and Chrome are on PATH. Report versions or MISSING — don't install anything yet.

5. Whether the Stream Deck app is currently running, where the exe lives, and its version — the build step later embeds the app version in the package, so capture it now.

Then give me a table of what's on my deck today, the exact backup command you'd run before touching anything, and your plan in one paragraph. Ask me before you write a single file.

---

## 2 — Design the deck before building it

Now design my deck on paper before you build anything. Ask me at most 8 questions to learn: which apps I live in, what I do twenty times a day, what deserves one key versus a whole page, whether I want text/prompt macros, and what I want on the dials.

Then propose a page plan as literal ASCII grids — one grid per page, one cell per key, at my deck's real dimensions — plus:

- A section color scheme. Group keys into families (agents / dev / apps / projects / whatever fits me) and give each family its own ground color, so I find keys by color and never have to read the deck.
- Two navigation keys pinned to the SAME position on every single page: back and forward. Page turns must never move under my thumb. The last page's forward key goes home. Those two slots are reserved on every page — nothing else ever lives there.
- A dial map where every dial does something on EVERY page. A dial that's dead on page 3 feels like broken hardware. Reserve the leftmost dial as a zoom knob everywhere: turn = zoom in / zoom out (Ctrl+= / Ctrl+-; Cmd on a Mac), press = reset to 100% (Ctrl+0). It earns that slot on every single page.
- A safe-arm interlock on anything destructive or irreversible: a deliberate arm step with a short timeout before the key will fire, and a cold press that does nothing but report. One fat thumb must never be enough.

Show me the grids and wait for my edits. Don't build yet.

---

## 3 — Build the profile (the format doctrine)

Build the profile now by generating a .streamDeckProfile package (it's just a ZIP) with a script I can re-run — not by hand-editing the live store. Write the script so every key's label, icon and command live in ONE list at the top of the file. I'll be editing that list for years.

Two gates before you write anything: if this conversation never did the read-only recon, go read the live store and app version now — never invent those values. And if we never agreed a page plan, show me one and wait for my yes.

Package layout (this is ground truth, don't improvise it):

    package.json                                                    at the zip root
    Profiles/<PROFILE-UUID>.sdProfile/manifest.json
    Profiles/<PROFILE-UUID>.sdProfile/Profiles/<PAGE-UUID>/manifest.json
    Profiles/<PROFILE-UUID>.sdProfile/Profiles/<PAGE-UUID>/Images/*.png

package.json is not empty — a real one from a working package looks like this; match the shape and fill in the values you found during recon:

    {"AppVersion": "<installed Stream Deck version>", "DeviceModel": "<model string>",
     "DeviceSettings": null, "FormatVersion": 1, "OSType": "Windows",
     "OSVersion": "<from the OS>",
     "RequiredPlugins": ["<every action UUID your pages actually use, exactly as the live manifests spell them>"]}

Rules already paid for in someone else's lost weekend:

- Page folder names on disk are UPPERCASE UUIDs; the "Pages" array in the profile manifest references them in lowercase. Get this backwards and pages silently vanish after the app reloads.
- "Pages.Default" must point at a HIDDEN BLANK template page whose UUID is NOT in the "Pages" array. The blank page manifest is exactly:
  {"Controllers":[{"Actions":null,"Type":"Keypad"},{"Actions":null,"Type":"Encoder"}],"Icon":"","Name":""}
  If Default points at a real visible page, the import path silently DROPS a page and you'll spend hours thinking your builder is broken.
- Every page manifest has "Controllers": a list containing an "Encoder" entry and a "Keypad" entry. Keys are addressed by string keys of the form "col,row".
- Every UUID must parse as a real UUID.
- Bake the label INTO the key image and set "ShowTitle": false — note ShowTitle lives in the action's States[0] object next to "Image", NOT in Settings. The app's own title chrome is ugly and can't be styled.
- To launch something: action UUID "com.elgato.streamdeck.system.open", Settings = {"openInBrowser": false, "path": "<full path to a .cmd wrapper>"}. The Open action cannot run a .ps1 directly — always wrap PowerShell in a .cmd. And make launcher wrappers focus-or-launch: if the app's window already exists, raise it instead of spawning a second instance — a launcher key pressed twice should never mean two copies running. Save every wrapper as ASCII or UTF-8 with BOM — PowerShell 5.1 reads a BOM-less file as ANSI, and one stray em dash becomes a bogus parse error pointing at the wrong line.
- To type something: action UUID "com.elgato.streamdeck.system.text", full text in "pastedText", "isTypingMode": false (clipboard paste, instant), and "isSendingEnter" true or false depending on whether that key should submit.
- Dials that run a command are wired as the SAME direct Open action as a key, with the image set on the Open action itself. Never wrap a single command in an Action Wheel: a one-child wheel opens a picker UI on press and the command never fires — the dial just looks dead.

Then write me a validator script that opens the built package and fails loudly on: any action referencing a missing image, a Default page that appears in Pages[], case-mismatched folder names, non-UUID strings, dead dials, and any key whose label doesn't match the command it fires. Run it and show me it passing before we go anywhere near the live store.

---

## 4 — Real logos, never an approximation

Now make the key art properly, with real vendor marks — not your approximations of them.

Sources, in this order:

1. Simple Icons for brand marks:
   https://raw.githubusercontent.com/simple-icons/simple-icons/develop/icons/<slug>.svg
   One clean monochrome path at viewBox "0 0 24 24". Also pull
   https://raw.githubusercontent.com/simple-icons/simple-icons/develop/data/simple-icons.json
   and use each brand's OFFICIAL hex to tint its key, so nothing is arbitrarily colored.

2. Google Material Symbols for anything that isn't a brand:
   https://raw.githubusercontent.com/google/material-design-icons/master/symbols/web/<name>/materialsymbolsoutlined/<name>_24px.svg
   Careful: these use viewBox "0 -960 960 960" and are FILLED paths — set fill:currentColor and stroke:none or you'll render an empty key.

3. If a brand isn't in Simple Icons (Adobe apps, some AI tools, niche software), pull the official SVG or a 512px-or-larger PNG from that company's own press/brand page, and tell me which URL you used.

Those URL shapes are correct as of this writing. If one 404s, the repo has moved things around — find the new path and tell me. A dead URL is never a reason to fall back to approximating.

Hard rule: if you cannot find the real mark, STOP and tell me which one is missing and where you looked. Never substitute a similar-looking icon, and never hand-draw SVG path data. A wrong logo is worse than a blank key, and a hand-drawn one always looks hand-drawn at this size.

Render pipeline: compose each key as a small HTML file (gradient ground, inline SVG glyph, baked label), screenshot it with headless Chrome at 3x supersample, then downsample with ImageMagick using -filter Lanczos -resize 288x288 -strip. Author at 288px even though the panel is smaller — let the software downscale. Art authored at panel resolution looks visibly soft.

Go wide on color: these panels are vivid, so use deep saturated grounds with one high-contrast glyph. Keys read at a glance by color first, glyph second, text last.

One more art-direction rule: name every mechanism by its real engineering term, never by a metaphor. Ask for a "safe-arm interlock" and you get clean instrument chrome; describe the same thing in figurative language and you get that figure of speech drawn literally on every key. The vocabulary I give you becomes the artwork, so keep it technical.

---

## 5 — Animated keys at ~50 FPS

Animated GIFs work on Stream Deck keys AND on the dial / touch-strip segments, and they look incredible. Animate my hero keys — HERO keys, not all of them.

Restraint first, because this is the mistake everyone makes: DO NOT animate every icon. A deck where thirty keys are moving is a slot machine — nothing reads, my eye has nothing to land on, and the one key that actually needed attention is now invisible. Static keys are the ground that makes a moving key mean something.

Animate a key only when motion is doing a job:
- it carries STATE I need to see without reading — armed, live, recording, connected, running, locked
- it's a destructive or irreversible key I want my eye pulled to before I press it
- it's the single signature key of a page, and the page has a lot of quiet around it

Do not animate: app launchers, folder shortcuts, text macros, anything I press dozens of times a day, and anything sitting in a block of similar keys. Those want to be instantly recognizable, which means they want to hold still.

Budget: at most two or three animated surfaces visible at once (the dial strip counts as one). If you think a fourth earns it, tell me why first. Also propose which keys should animate CONDITIONALLY — swapping to the GIF only when that thing is actually running or armed, and sitting on a static PNG otherwise. That's better than a permanent loop in almost every case (the swap itself is an install-time choice or a proper plugin's job — never a script hot-editing the live store while the app runs).

The math, which is where everyone gets burned — GIF frame delays are in CENTIseconds:

    delay 2  = 20ms/frame = ~50 FPS   <-- the practical ceiling. Use this.
    delay 1  = 10ms/frame = ~100 FPS  <-- same cycle length needs twice the frames, twice the bytes, zero visible gain
    delay 10 = 100ms      = 10 FPS    <-- visibly steppy on a small bright panel

True 60 FPS would need 16.67ms, which the GIF format simply cannot express. Don't chase it — 50 FPS is the max and it is smooth.

Rules:

- Assemble with FULL frames at the key's full pixel size. Do NOT run ImageMagick's -layers optimize on the final GIF: it collapses frames into 1x1 partial deltas, and Stream Deck then rejects the file or shows garbage.
- Windows argv limit: don't pass hundreds of frame paths to one magick call or you'll hit CreateProcess error 206. cd into the frames directory and assemble with a glob like f*.png.
- Fast frame rate, SLOW cycle. Example: 240 frames at delay 2 is a 4.8-second breathe running at 50 FPS. High FPS buys smoothness, not speed.
- If two animated surfaces sit near each other, give one twice the frames so its tempo is half. Matching tempos read as mechanical; offset tempos read as alive.
- Fade all the way to pure black rather than to a dim tint — a partial fade looks like a rendering bug on this panel.
- Always also emit a static PNG of the mid-frame as a fallback, and reference the GIF exactly like the PNGs: "Images/name.gif" on the action's Image field.

Nothing installs in this step — deployment is the next prompt's ritual. Once it's deployed, verify: report the installed GIF's real pixel dimensions (1x1 means it got optimized and is broken) and confirm it's looping on the hardware.

---

## 6 — Deploy, verify, roll back

Deploy time. Follow this ritual exactly — installing while the app is running is how profiles get corrupted.

1. FULLY QUIT the Stream Deck software first, not minimize to tray. The clean way is running StreamDeck.exe with the --quit flag. Notes: taskkill without /F gets Access Denied (it's a tray app with no top-level window), and the exe's manifest demands elevation, so a plain CreateProcess call (Python subprocess) throws WinError 740 — launch it via ShellExecute instead (os.startfile in Python, Start-Process in PowerShell).

2. STAY NON-ELEVATED FOR THE BUILD AND THE INSTALL. The profile store lives under %APPDATA% — your own user already owns every byte of it, and elevated shells break this flow both ways: a UAC shell can come up with a different PATH (python and magick vanish) and profile writes get denied FROM the elevated side. The only thing that ever needs elevation is StreamDeck.exe itself, and ShellExecute raises that UAC prompt for you. If a write to the store is denied, the app is still alive — go back to step 1 instead of reaching for Administrator.

3. Back up %APPDATA%\Elgato\StreamDeck\ProfilesV3 to a timestamped folder. Every time. Print the path you used.

4. Install by replacing the CONTENTS of the existing UUID.sdProfile folder, keeping the live folder's UUID name. Which profile the device boots into is stored in an opaque blob under HKCU\Software\Elgato Systems GmbH\StreamDeck\Devices — never patch that; reuse the existing UUID instead. (Note that importing through the GUI does not replace by name: it lands as "<name> copy".)

5. Start the app, wait 30 seconds, then read %APPDATA%\Elgato\StreamDeck\logs\StreamDeck.log and confirm: it used the existing profile store, the device connected, and there is NO "Profile not found" and NO import of the factory DefaultProfiles package. That factory import only fires when the device has no valid profile — if you see it, we broke something.

6. Quit and restart once more, then re-verify the page count. Pages that survive one load but vanish on the second are the UUID-case bug from the build prompt.

7. Finally: tell me exactly what to physically press to test it, key by key, and write a ROLLBACK.txt next to the backup containing the literal restore commands.

---

## 7 — A page of prompt macros

Add a page of prompt keys — one key equals one long instruction pasted straight into whatever text box I'm focused on.

Use the action UUID "com.elgato.streamdeck.system.text" with the whole prompt in "pastedText" and "isTypingMode": false. Simulated typing is unusably slow for anything longer than a sentence; clipboard paste is instant. It does overwrite my clipboard — that's the accepted trade. (Paste mode needs app 6.9 or later — you captured my version during recon, so check it.)

Split the page by intent, and be consistent about it:

- SEND keys ("isSendingEnter": true) — self-contained instructions that should fire the moment I press them.
- APPEND keys ("isSendingEnter": false) — modifiers that stack onto something I've already typed. Start each of those payloads with ", " so they graft cleanly onto a sentence in progress.

Never mix the two behaviors on one page. Muscle memory can't hold the exception.

Keep each key's label and its prompt body in the SAME data structure entry. If labels live in one list and bodies in another and you pair them by index, you WILL ship the wrong prompt under the wrong key, and it takes an hour of confusion to notice.

Encoding trap that has shipped a real bug before: pastedText must carry real newlines. If your writer double-escapes them, the key pastes a literal backslash-n into my text box. Have the validator scan every payload for that.

This page obeys the same laws as every other page: the two reserved navigation keys stay in their corners, no dial goes dead, and it ships through the same build → validate → deploy ritual — never pasted into the live store while the app is running.

Give every prompt on the page a shared footer contract so outputs are comparable across keys — for example, end every one of them by demanding: VERDICT / EVIDENCE / OPEN RISKS / NEXT ACTION / STOP CONDITION.

---

## The laws (pocket version)

| Symptom | Cause | Fix |
|---|---|---|
| Pages vanish after reload | Folder UUID case | UPPERCASE folder on disk, lowercase in `Pages[]` |
| A page disappears on import | `Pages.Default` points at a real page | Default = hidden blank page, not listed in `Pages[]` |
| Store corrupts mid-edit | App was running | Quit → install → start → verify → restart → verify |
| Install fails / python+magick vanish | Build ran in an elevated shell | Stay non-elevated — the store is under `%APPDATA%`; only StreamDeck.exe elevates itself (ShellExecute) |
| "Not a gif" / garbage key | `-layers optimize` made 1×1 partials | Full frames only |
| magick CreateProcess 206 | Argv too long | cd into frames dir, assemble with a glob |
| Key does nothing | `.ps1` on a System > Open action | Wrap in a `.cmd` |
| Wrong prompt under a key | Labels and bodies paired by index | One record per key |
| Key pastes literal `\n` text | Double-escaped newlines in `pastedText` | Real newlines; validator scans every payload |
| Dial press does nothing | It's a single-child Action Wheel | Direct Open action instead |
| Bogus PowerShell parse error | PS 5.1 read a BOM-less file as ANSI | Save UTF-8 with BOM, or stay ASCII |

---

D. Lombardo / Coldbricks — verified against a live Stream Deck + XL profile store, Windows 11, 2026-08-10.
