# The Experience Machine: Elite Dangerous — Build Doc

A weekend-scope Elite Dangerous mod that lets you speak, in natural language, to a philosopher-persona NPC — a station kiosk, a dodgy mission giver — and get an in-character reply, spoken aloud with an on-screen subtitle. Same family as the Skyrim and Fallout 4 builds (`the-experience-machine-ii`, `the-experience-machine-fo4`), but the underlying mechanism is different, and in most ways simpler: Elite Dangerous has no Creation Kit or Papyrus, so instead of scripting the game engine, this build rides on Frontier's own sanctioned Journal file and the EDMC plugin ecosystem.

## 0. Read this before you start: what's actually different here

Elite Dangerous is closed-source. There is no official modding toolkit, and there's no way to reach into an NPC's dialogue tree the way Creation Kit + Papyrus lets you in Bethesda games. On-foot kiosk and mission-giver NPCs have fixed, hardcoded dialogue — walk up, press E, canned menu — and that can't be rewritten from outside the game process.

What makes this possible anyway is the same trick behind [COVAS:NEXT](https://www.nexusmods.com/elitedangerous/mods/8) (the ship-computer voice assistant this project is modeled on): Elite Dangerous continuously writes a plain-text [Journal](https://elite-journal.readthedocs.io/) of everything you do — dock, accept a mission, approach a settlement — as line-delimited JSON in `%userprofile%\Saved Games\Frontier Developments\Elite Dangerous`. That's a documented, Frontier-sanctioned interface for third-party tools, not a hack. On top of it, [EDMC](https://github.com/EDCD/EDMarketConnector) (Elite Dangerous Market Connector) is a widely-used companion app that already tails the Journal and exposes a plugin API, and [EDMCOverlay](https://github.com/inorton/EDMCOverlay) is an existing plugin that paints text directly onto the game screen over a simple local JSON/TCP protocol.

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
        └─ sends JSON over TCP to EDMCOverlay → subtitle drawn
           directly on the game screen
```

Nothing here touches Elite Dangerous' game files or memory. EDMC reads the Journal (a file Frontier already writes for this purpose); EDMCOverlay draws text via its own documented local socket. Everything else is normal Python.

## 2. Tools you need

| Tool | Why | Link |
|---|---|---|
| Elite Dangerous + Odyssey | The game; Odyssey is required for on-foot kiosks/settlements | Steam / Frontier Store |
| EDMC (Elite Dangerous Market Connector) | Tails the Journal for you and provides the plugin framework — this is your "PapyrusUtil" | https://github.com/EDCD/EDMarketConnector |
| EDMCOverlay (or EDMCModernOverlay) | Draws subtitle text directly on the game screen via a local JSON/TCP API | https://github.com/inorton/EDMCOverlay |
| Python 3.11+ | EDMC plugins are plain Python; also runs your LLM/TTS calls | python.org |
| Piper TTS | Local, free, fast text-to-speech (same as the other two builds) | https://github.com/rhasspy/piper |
| An LLM API key | OpenRouter or OpenAI — powers each persona's reply | openrouter.ai |
| `keyboard` (Python package) | Optional — global hotkey to open the "talk" popup without alt-tabbing | PyPI |

No GPU required. Piper runs on CPU. No SKSE/F4SE equivalent, no Creation Kit — EDMC is a standard installer, and your plugin is a folder you drop into EDMC's `plugins` directory.

## 3. Phase 1 — Environment setup

1. Install Elite Dangerous and the Odyssey expansion; launch once and confirm you can walk around a settlement or station concourse.
2. Confirm the Journal is being written: play for a minute, then check `%userprofile%\Saved Games\Frontier Developments\Elite Dangerous` for a `Journal.*.log` file growing with new lines.
3. Install EDMC, launch it alongside the game, and confirm it shows your commander name and current system — this proves it's reading the Journal correctly.
4. Install EDMCOverlay (or EDMCModernOverlay) as an EDMC plugin per its README; test it with the sample script that draws "Hello" on screen.
5. Set up Python: `python -m venv venv`, activate it, `pip install requests keyboard`. Download a Piper voice model and confirm `piper --model <voice>.onnx --output_file test.wav` works from the command line.
6. Decide your first two personas (e.g. a Stoic-inspired station kiosk, a Machiavellian mission broker) and sketch a 3–5 sentence system prompt for each — you'll wire these in Phase 3.

**Done when:** EDMC shows live game state, EDMCOverlay can draw test text on screen, and Piper can turn a string into a `.wav` from the command line.

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
4. **Show the subtitle.** Send a JSON message to EDMCOverlay's TCP socket (`127.0.0.1:5010`) with the reply text, a reasonable `ttl`, and a screen position — this is your Fuz Ro D-oh / F4z Ro D-oh equivalent, except it's a real in-game overlay instead of a simulated subtitle.
5. Run the LLM/TTS calls in a background thread so they don't freeze EDMC's UI while waiting on the API.

**Done when:** typing a line into the popup produces a spoken reply and a matching on-screen overlay subtitle, with the persona matching your current station/mission context.

## 6. Phase 4 — Persona spec sheet

This is the part unique to your concept, and worth its own phase rather than folding it into Phase 3.

1. Write a short spec sheet per persona: name, philosophical tradition, tone, 2–3 example lines, and any topics they refuse or deflect (keeps replies in-character and bounded).
2. Store personas as a simple JSON/YAML file (`personas.json`) keyed by station type or NPC role, so adding a third or fourth persona later is a data change, not a code change.
3. Test each persona against a handful of player lines to check tone consistency before moving on.

**Done when:** you have at least two working personas (kiosk + dodgy mission giver) with distinct, consistent voices.

## 7. Phase 5 — Integration and end-to-end test

1. Launch Elite Dangerous, launch EDMC with your plugin loaded, fly to a station with on-foot access, walk to a kiosk or mission board.
2. Click "Talk," type a line, confirm the full loop: context detected → correct persona selected → LLM reply → TTS → overlay subtitle, all while standing at the right NPC.
3. Test at a second location (different station or a settlement) to confirm persona switching works from Journal context alone.
4. Time the round trip; if it feels slow, drop to a faster/cheaper LLM model before considering architecture changes.

**Done when:** you can walk between two different NPCs and get two different, contextually appropriate philosopher personas without restarting EDMC.

## 8. Phase 6 — Stretch goals (beyond the weekend)

- **Per-NPC memory:** persist a short conversation log per station/persona so repeat visits build continuity.
- **Companion API (CAPI) integration:** authenticate against Frontier's Companion API for richer context (full mission list, faction standing, market data) beyond what the Journal alone provides.
- **Voice input:** replace the typed popup with local Whisper transcription, triggered by the same hotkey.
- **More personas:** expand `personas.json` to cover more station types, factions, or named mission givers.
- **Packaging:** publish as a proper EDMC plugin (zip + README) so others can drop it into their own `plugins` folder.

## 9. Timeline

| When | What |
|---|---|
| Day 1 morning | Phase 1 (environment: EDMC, EDMCOverlay, Piper, personas sketched) |
| Day 1 afternoon | Phase 2 + 3 (plugin skeleton, trigger, LLM/TTS/overlay wired end-to-end) |
| Day 2 morning | Phase 4 (persona spec sheets, multiple personas tested) |
| Day 2 afternoon | Phase 5 (full integration + multi-location test pass) |
| Beyond | Phase 6 (pick 1–2 stretch goals, not all of them) |

## 10. Troubleshooting

- **EDMC doesn't show live game state:** confirm the Journal folder path in EDMC's settings matches where the game is actually writing (`Saved Games\Frontier Developments\Elite Dangerous`), and that EDMC was launched after the game.
- **Plugin doesn't load:** check EDMC's own log (Help menu) for a Python traceback — a bad `plugin_start3()` signature is the most common cause.
- **Overlay text doesn't appear:** confirm EDMCOverlay's process is running (it's a separate executable EDMC launches) and that Elite Dangerous is running in Borderless or Windowed mode — true Fullscreen Exclusive can block overlays.
- **Context is wrong/stale:** the plugin's stored station/mission state only updates on new Journal events — test by actually docking/undocking rather than assuming a stale session will refresh.
- **LLM replies feel slow:** switch to a smaller/faster model, and confirm the LLM and TTS calls are running in a background thread, not blocking EDMC's main loop.
