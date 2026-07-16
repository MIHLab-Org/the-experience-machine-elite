
# The Experience Machine: Elite Dangerous — Build Doc

A weekend-scope Elite Dangerous mod that lets you speak, in natural language, to a philosopher-persona NPC — a station kiosk, a dodgy mission giver — and get an in-character reply, spoken aloud with an on-screen subtitle. Same family as the Skyrim and Fallout 4 builds (`the-experience-machine-ii`, `the-experience-machine-fo4`), but the underlying mechanism is different, and in most ways simpler: Elite Dangerous has no Creation Kit or Papyrus, so instead of scripting the game engine, this build rides on Frontier's own sanctioned Journal file and the EDMC plugin ecosystem.

## 0. Read this before you start: what's actually different here

Elite Dangerous is closed-source. There is no official modding toolkit, and there's no way to reach into an NPC's dialogue tree the way Creation Kit + Papyrus lets you in Bethesda games. On-foot kiosk and mission-giver NPCs have fixed, hardcoded dialogue — walk up, press E, canned menu — and that can't be rewritten from outside the game process.

What makes this possible anyway is the same trick behind [COVAS:NEXT](https://www.nexusmods.com/elitedangerous/mods/8) (the ship-computer voice assistant this project is modeled on): Elite Dangerous continuously writes a plain-text [Journal](https://elite-journal.readthedocs.io/) of everything you do — dock, accept a mission, approach a settlement — as line-delimited JSON in `%userprofile%\Saved Games\Frontier Developments\Elite Dangerous`. That's a documented, Frontier-sanctioned interface for third-party tools, not a hack. On top of it, [EDMC](https://github.com/EDCD/EDMarketConnector) (Elite Dangerous Market Connector) is a widely-used companion app that already tails the Journal and exposes a plugin API, and [EDMCModernOverlay](https://github.com/SweetJonnySauce/EDMCModernOverlay) is an existing plugin that paints text directly onto the game screen. Note: despite descending from the classic EDMCOverlay's JSON/TCP protocol, EDMCModernOverlay does **not** run a socket listener for outside connections — it only accepts payloads from code running inside EDMC's own process (see Phase 3, step 4).

So instead of building a mod that changes what the kiosk says, you're building an EDMC plugin: it watches the Journal to know your context (which station, which mission board, which faction), gives you a way to type or speak a line, sends it to an LLM playing your philosopher persona, and displays the reply as an overlay subtitle with TTS audio — timed to feel like it's coming from the NPC in front of you, even though mechanically it's a companion app, not a rewritten game character.

**Practical bonus:** because EDMC and your plugin are all one Python process, there's no cross-process file-bridge to build at all (unlike the Papyrus↔Python JSON handoff in the Skyrim/FO4 builds) — the whole thing can be a single plugin file. And you need Odyssey (currently $39.99, frequently discounted) for on-foot settlements/kiosks specifically; the base game alone doesn't include on-foot NPCs.

## 1. Architecture

```
 Player walks up to a kiosk / mission giver, presses a hotkey
        │
        ▼
 EDMC plugin (Python, running inside EDMC)
        │
        ├─ journal_entry() callback already fired on Docked /
        │  ApproachSettlement / MissionAccepted — plugin knows
        │  current station, faction, nearby mission givers
        │
        ├─ small popup captures the player's typed/spoken line
        │
        ├─ calls LLM API with philosopher persona system prompt
        │  (selected by context: kiosk persona vs. mission-giver persona)
        │
        ├─ calls Piper TTS → reply.wav → plays it locally
        │
        └─ imports EDMCOverlay's compatibility layer in-process
           and calls send_raw() → subtitle drawn on the game screen
```

Nothing here touches Elite Dangerous' game files or memory. EDMC reads the Journal (a file Frontier already writes for this purpose); EDMCModernOverlay draws text via a Python compatibility module (`EDMCOverlay.edmcoverlay`) that your plugin imports directly — not a network socket. Everything else is normal Python.

## 2. Tools you need

| Tool | Why | Link |
|---|---|---|
| Elite Dangerous + Odyssey | The game; Odyssey is required for on-foot kiosks/settlements | Steam / Frontier Store |
| EDMC (Elite Dangerous Market Connector) | Tails the Journal for you and provides the plugin framework — this is your "PapyrusUtil" | https://github.com/EDCD/EDMarketConnector |
| EDMCModernOverlay | Draws subtitle text directly on the game screen. Your plugin talks to it by importing `EDMCOverlay.edmcoverlay` in-process (not a socket) | https://github.com/SweetJonnySauce/EDMCModernOverlay |
| Python 3.11+ | EDMC plugins are plain Python; also runs your LLM/TTS calls | python.org |
| Piper TTS | Local, free, fast text-to-speech (same as the other two builds). Install via `pip install piper-tts` — the old standalone binary repo has moved and is no longer the recommended path | https://github.com/OHF-Voice/piper1-gpl |
| An LLM API key | OpenRouter or OpenAI — powers each persona's reply | openrouter.ai |
| `keyboard` (Python package) | Optional — global hotkey to open the "talk" popup without alt-tabbing | PyPI |

No GPU required. Piper runs on CPU. No SKSE/F4SE equivalent, no Creation Kit — EDMC is a standard installer, and your plugin is a folder you drop into EDMC's `plugins` directory.

## 3. Phase 1 — Environment setup

1. Install Elite Dangerous and the Odyssey expansion; launch once and confirm you can walk around a settlement or station concourse.
2. Confirm the Journal is being written: play for a minute, then check `%userprofile%\Saved Games\Frontier Developments\Elite Dangerous` for a `Journal.*.log` file growing with new lines.
3. Install EDMC, launch it alongside the game, and confirm it shows your commander name and current system — this proves it's reading the Journal correctly.
4. Install EDMCModernOverlay as an EDMC plugin per its README. Test it with the built-in `!ovr test` chat command (type it in the in-game chat panel) to confirm the overlay itself renders. Do **not** test with a standalone socket script — EDMCModernOverlay doesn't listen on a port; see Phase 3, step 4 for the correct way to send it a message from code.
5. Set up Python: `python -m venv venv`, activate it, `pip install requests keyboard piper-tts`. Download a voice with `python -m piper.download_voices en_US-lessac-medium`, then confirm `python -m piper -m en_US-lessac-medium -f test.wav -- "Hello, CMDR."` produces a `test.wav` that actually plays.
   > **Watch out:** if you use the older standalone `piper` binary instead of the `piper-tts` pip package, it reads the sentence to speak from **stdin**, not from a flag — `piper --model <voice>.onnx --output_file test.wav` with nothing piped in will "succeed" and write a WAV file with no real audio in it, which shows up as "corrupted" when you try to play it. Either pipe text in (`echo "Hello" | piper --model <voice>.onnx --output_file test.wav`) or use the `piper-tts` command above, which takes the text as a real argument so this can't happen silently.
6. Decide your first two personas (e.g. a Stoic-inspired station kiosk, a Machiavellian mission broker) and sketch a 3–5 sentence system prompt for each — you'll wire these in Phase 3.

**Done when:** EDMC shows live game state, EDMCModernOverlay renders the `!ovr test` box, and Piper produces a `.wav` file that actually plays back audio (not just a file that exists).

## 4. Phase 2 — The trigger (EDMC plugin skeleton)

Unlike Skyrim/FO4, there's no in-game item or spell to create — the trigger lives entirely in your plugin.

1. Create a folder in EDMC's `plugins` directory containing a `load.py` with the minimal EDMC plugin skeleton (a `plugin_start3()` function returning the plugin name, and optionally `plugin_app()` to add a button to EDMC's window).
2. Add a "Talk" button to EDMC's main window (simplest trigger — no hotkey library needed for the weekend version) that opens a small Tkinter popup for typed input.
3. Implement `journal_entry(cmdr, is_beta, system, station, entry, state)` to store the latest `StationName`, `StationType`, and any `MissionAccepted`/`Missions` data in a module-level variable, so the plugin always knows "where" the player is talking from.

**Done when:** clicking "Talk" in EDMC pops up a text box, and printing `state`/`station` to the console shows accurate current context.

## 5. Phase 3 — Persona bridge

1. **Pick the persona.** Use the stored station/context data from Phase 2 to choose which philosopher system prompt to use (e.g., station services kiosk vs. a mission-giver NPC name pulled from `Missions` journal data).
2. **Call the LLM.** Send the system prompt plus the player's typed line to your LLM API of choice. Keep responses short (2–4 sentences) so TTS playback stays snappy.
3. **Generate speech.** Pipe the reply into Piper, save as `reply.wav`, play it locally (`winsound`/`playsound`).
4. **Show the subtitle.** Import EDMCModernOverlay's compatibility layer directly in your plugin's code — `from EDMCOverlay import edmcoverlay` — and call `edmcoverlay.Overlay().send_raw({...})` with the reply text, a reasonable `ttl`, and a screen position. This runs in-process (EDMC and EDMCModernOverlay share the same Python process), not over a network socket — there is no `127.0.0.1:5010` listener to connect to anymore. This is your Fuz Ro D-oh / F4z Ro D-oh equivalent, except it's a real in-game overlay instead of a simulated subtitle.
5. Run the LLM/TTS calls in a background thread so they don't freeze EDMC's UI while waiting on the API.

**Done when:** typing a line into the popup produces a spoken reply and a matching on-screen overlay subtitle, with the persona matching your current station/mission context.

## 6. Phase 3.5 — Mission engine (recommended before Phase 4)

Phase 3 gets you single-turn Q&A: player asks a question, persona answers. If you want an NPC to *assign a task* — "turn off your HUD and dock manually, then come back and tell me how it went" — and have the debrief reference what happened, you need a small state layer on top of the persona bridge. That's this phase.

**One thing to know first: not everything the player does is verifiable.** Elite Dangerous's Journal and `Status.json` expose a lot of real-time state — `FlightAssist Off`, `Silent Running`, `Landing Gear Down`, `Shields Up`, `Docked`, on-foot flags, plus Journal events like `FSDJump`, `Scan`, `DockingGranted` (see the [Status File docs](https://elite-journal.readthedocs.io/en/latest/Status%20File/) for the full flag bitfield) — but there is **no flag for "HUD turned off."** That's a purely visual, client-side setting Frontier doesn't expose to any external tool. For a mission like the one above, the plugin can't programmatically confirm the player actually did it — the NPC has to take their word for it in conversation. Given this project's theme (testing subjective intuition, not scoring a puzzle), that's arguably the right design anyway: a self-report check-in, not a graded test. Reserve objective verification (`status_flag` / `journal_event`) for missions where it's actually meaningful and available — "fly with FlightAssist off," "dock without shields," "scan an anomaly."

1. **Define missions as data, not code.** Create a `missions/` folder next to `personas.json`, one YAML file per mission. Your team edits these directly — no Python required to add or tweak a mission. Suggested schema:

   ```yaml
   id: extended_cognition_01
   persona: stoic_kiosk              # matches a key in personas.json
   trigger_keywords: ["extended mind", "cognitive faculties", "intuition"]

   stages:
     assign:
       npc_says: >
         Some say the mind stops at the skull. Others say it leaks into the
         tools we use to think. Disable your HUD, fly manually, then come
         back and tell me what changed.
       pre_survey:
         - prompt: "Scale of 1-10, how much do you trust your instincts without instruments?"
           store_as: pre_confidence
       next: in_progress

     in_progress:
       completion: self_report        # self_report | status_flag | journal_event
       # completion: status_flag      # example of an objectively-verifiable alternative
       # flag: FlightAssistOff
       next: debrief

     debrief:
       post_survey:
         - prompt: "Same scale — now?"
           store_as: post_confidence
       npc_reflection: true           # ask the LLM to generate an in-character closing line using stored answers
   ```

2. **Add a mission loader.** On plugin start — and ideally on a "reload missions" button, so your team can test edits without restarting EDMC — read every file in `missions/` into memory, keyed by `id`.
3. **Add a state machine.** Each mission instance tracks its current stage (`assign` / `in_progress` / `debrief` / `complete`) per player. A stage advances when either the LLM detects a self-report confirmation in the player's next line ("I did it," "just docked manually"), or the plugin polls `Status.json`/the Journal for the configured flag or event.
4. **Persist per-player mission state.** Write a small `mission_state.json` next to the plugin (mission id, current stage, timestamp assigned, and any `store_as` survey answers), so a mission that spans a real play session — fly somewhere, do the thing, come back — survives EDMC restarts and station changes.
5. **Wire the debrief reflection.** When a mission reaches `debrief`, feed the persona's system prompt plus the stored survey answers (e.g. `pre_confidence`/`post_confidence`) back to the LLM so it generates a persona-voiced line that actually references the change, instead of a static string.

**Done when:** you can trigger the extended-cognition mission by asking the kiosk about it, get sent off with a pre-survey answer captured, fly around, come back, and get a debrief that references your own before/after answers in the persona's voice.

## 7. Phase 4 — Persona spec sheet

This is the part unique to your concept, and worth its own phase rather than folding it into Phase 3.

1. Write a short spec sheet per persona: name, philosophical tradition, tone, 2–3 example lines, and any topics they refuse or deflect (keeps replies in-character and bounded).
2. Store personas as a simple JSON/YAML file (`personas.json`) keyed by station type or NPC role, so adding a third or fourth persona later is a data change, not a code change.
3. Test each persona against a handful of player lines to check tone consistency before moving on.

**Done when:** you have at least two working personas (kiosk + dodgy mission giver) with distinct, consistent voices.

## 8. Phase 5 — Integration and end-to-end test

1. Launch Elite Dangerous, launch EDMC with your plugin loaded, fly to a station with on-foot access, walk to a kiosk or mission board.
2. Click "Talk," type a line, confirm the full loop: context detected → correct persona selected → LLM reply → TTS → overlay subtitle, all while standing at the right NPC.
3. Test at a second location (different station or a settlement) to confirm persona switching works from Journal context alone.
4. Time the round trip; if it feels slow, drop to a faster/cheaper LLM model before considering architecture changes.

**Done when:** you can walk between two different NPCs and get two different, contextually appropriate philosopher personas without restarting EDMC.

## 9. Phase 6 — Stretch goals (beyond the weekend)

- **Per-NPC memory:** persist a short conversation log per station/persona so repeat visits build continuity (note: `mission_state.json` from Phase 3.5 already gives you this for mission-linked conversations — this stretch goal is for open-ended chat continuity beyond missions).
- **Companion API (CAPI) integration:** authenticate against Frontier's Companion API for richer context (full mission list, faction standing, market data) beyond what the Journal alone provides.
- **Voice input:** replace the typed popup with local Whisper transcription, triggered by the same hotkey.
- **More personas:** expand `personas.json` to cover more station types, factions, or named mission givers.
- **Packaging:** publish as a proper EDMC plugin (zip + README) so others can drop it into their own `plugins` folder.

## 10. Timeline

| When | What |
|---|---|
| Day 1 morning | Phase 1 (environment: EDMC, EDMCOverlay, Piper, personas sketched) |
| Day 1 afternoon | Phase 2 + 3 (plugin skeleton, trigger, LLM/TTS/overlay wired end-to-end) |
| Day 2 morning | Phase 3.5 + Phase 4 (mission engine wired, persona spec sheets, multiple personas tested) |
| Day 2 afternoon | Phase 5 (full integration + multi-location test pass) |
| Beyond | Phase 6 (pick 1–2 stretch goals, not all of them) |

## 11. Troubleshooting

- **EDMC doesn't show live game state:** confirm the Journal folder path in EDMC's settings matches where the game is actually writing (`Saved Games\Frontier Developments\Elite Dangerous`), and that EDMC was launched after the game.
- **Plugin doesn't load:** check EDMC's own log (Help menu) for a Python traceback — a bad `plugin_start3()` signature is the most common cause.
- **Overlay text doesn't appear:** first confirm EDMCModernOverlay itself is working by typing `!ovr test` in the game chat — you should see a black test box. If that works but your plugin's own messages don't show, you're not actually reaching the overlay: confirm your plugin does `from EDMCOverlay import edmcoverlay` and calls `.send_raw()`/`.send_message()` directly (in-process), not a raw socket connection to port 5010 — EDMCModernOverlay doesn't listen on that port for outside connections. Also confirm Elite Dangerous is running in Borderless or Windowed mode — true Fullscreen Exclusive can block overlays.
- **Context is wrong/stale:** the plugin's stored station/mission state only updates on new Journal events — test by actually docking/undocking rather than assuming a stale session will refresh.
- **LLM replies feel slow:** switch to a smaller/faster model, and confirm the LLM and TTS calls are running in a background thread, not blocking EDMC's main loop.
- **Mission won't advance:** check whether the mission's `completion` type actually matches what you're testing — `self_report` requires the LLM to recognize a confirmation in the player's line, `status_flag`/`journal_event` require that flag/event to actually exist (not everything is exposed — see Phase 3.5's note on HUD state).
