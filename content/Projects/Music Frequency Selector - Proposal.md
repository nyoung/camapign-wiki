---
type: project-proposal
tags: [project, music, browser-extension]
status: draft
---

# Music Frequency Selector — Proposal

A browser extension for Amazon Music that builds a personal listening schedule driven by play/skip feedback, so songs you love surface more often and songs you tire of naturally recede.

---

## Core Concept

Each song in your library has a **cooldown** value (a positive integer, minimum 1). After a song is played or skipped, it becomes ineligible to play again until `cooldown²` hours have passed. Listening to a song decreases its cooldown (plays sooner); skipping increases it (plays later).

### Cooldown Mechanics

| Event | Effect |
|-------|--------|
| Listen (>30s played) | `cooldown -= 1` (floor: 1) |
| Skip | `cooldown += 1` |

**Cooldown → wait time examples:**

| Cooldown | Next eligible |
|----------|--------------|
| 1 | 1 hour |
| 2 | 4 hours |
| 3 | 9 hours |
| 5 | ~1 day |
| 8 | ~2.5 days |
| 10 | ~4 days |

New songs enter at a default cooldown of **5** (eligible again in ~25 hours), giving them a chance to prove themselves before settling into a rhythm.

---

## Song Database

A plain text file on disk, human-readable and editable. Managed via the browser's **File System Access API** (user picks the file once; the extension remembers it).

### Format

```
# title | artist | cooldown | last_played_unix_epoch
Lateralus | Tool | 1 | 1750809600
The Less I Know the Better | Tame Impala | 3 | 1750780000
Redbone | Childish Gambino | 5 | 0
```

- `last_played = 0` means never played / always eligible
- Lines beginning with `#` are comments
- Fields are pipe-separated, trimmed of whitespace

---

## Shuffle & Selection Algorithm

On startup (or when a new song is needed):

1. Parse the database file
2. Shuffle all songs randomly
3. Walk the shuffled list; return the **first song** where:
   `now - last_played >= cooldown² hours`
4. If **no song is eligible** (all cooldowns still active):
   - Play the song with the **smallest remaining wait time** (earliest to become eligible)
   - This edge case also triggers a **discovery check** (see below)
5. After a song plays or is skipped, update its record in the file immediately

The shuffle happens once per session start; within a session songs are served in the shuffled order (re-shuffled on next launch or on explicit "reshuffle").

---

## Song Discovery

To keep the library growing without manual curation, the extension periodically finds new songs similar to your most-listened tracks.

### Trigger Conditions
- No eligible songs exist (all on cooldown) — idle time becomes exploration time
- On demand via a "Discover" button in the popup

### Process
1. Collect the **top 10 lowest-cooldown songs** (your favourites) as seeds
2. For each seed, call the **Last.fm API** (`track.getSimilar`) to get similar tracks
3. Filter out songs already in the database
4. Add candidates to the database at **cooldown = 5** with `last_played = 0`
5. Surface one of the new tracks immediately for a first listen

### Last.fm API
- Free, no OAuth required for read operations
- Endpoint: `https://ws.audioscrobbler.com/2.0/?method=track.getSimilar&artist=…&track=…&api_key=…`
- Requires a free Last.fm API key (user provides in extension settings)

---

## Browser Extension Architecture

**Target:** Chrome / Chromium (Manifest V3)  
**Host page:** `https://music.amazon.com/*`

### Components

```
extension/
├── manifest.json          # MV3 manifest
├── background/
│   └── service-worker.js  # File I/O, scheduling, Last.fm calls
├── content/
│   └── amazon-music.js    # DOM observer on music.amazon.com
├── popup/
│   ├── popup.html         # Extension popup UI
│   ├── popup.js
│   └── popup.css
└── storage/
    └── file-handle.js     # File System Access API wrapper
```

### Content Script Responsibilities (`amazon-music.js`)
- Observe DOM for **now-playing title/artist changes** (MutationObserver on the player bar)
- Detect **skip events** (click on skip button or song changes within ~5 seconds)
- Detect **listen events** (song plays for >30 seconds without a skip)
- Send events to the service worker via `chrome.runtime.sendMessage`

### Service Worker Responsibilities
- Maintain the database file via File System Access API
- Run the shuffle/selection algorithm
- Make Last.fm API calls for discovery
- Respond to popup requests for current state

### Popup UI
- **Now playing:** title + artist
- **Cooldown status:** current value + time until next eligible (`cooldown² - elapsed hours`)
- **Controls:** Play / Pause / Skip (simulated clicks injected into the page)
- **Queue preview:** next 3–5 eligible songs
- **Actions:** Reshuffle | Discover | Open database file

---

## Settings

Stored in `chrome.storage.sync`:

| Setting | Default | Description |
|---------|---------|-------------|
| `listenThresholdSeconds` | 30 | Seconds before a play counts as a listen |
| `defaultCooldown` | 5 | Cooldown assigned to newly added songs |
| `lastfmApiKey` | — | User-provided Last.fm API key |
| `discoverySeeds` | 10 | Number of favourite songs used as discovery seeds |
| `databaseFilePath` | — | File System Access handle (persisted separately) |

---

## Prototype Scope

The first working prototype should demonstrate the core loop only:

- [ ] Content script detects listen and skip events on `music.amazon.com`
- [ ] Database file is read and written correctly (File System Access API)
- [ ] Shuffle + eligibility selection algorithm works
- [ ] Popup shows now-playing info and cooldown value
- [ ] Cooldown updates on listen/skip and persists to file

Discovery (Last.fm integration) and the full popup queue view are **out of scope** for the prototype.

---

## Open Questions

1. **Amazon Music DOM stability** — the player bar selectors will need to be found by inspection and may break on Amazon Music updates. Consider a selector config in settings.
2. **File System Access in service workers** — MV3 service workers have limited filesystem access; the file handle may need to be managed from the popup context and passed to the worker. Needs a spike.
3. **Skip detection heuristic** — distinguishing a skip from a natural song-end-then-next requires timing. 5 seconds is a guess; needs tuning.
4. **Cooldown floor at 1** — with many songs at cooldown=1, you could cycle a small set hourly. A minimum library size recommendation may be useful.
