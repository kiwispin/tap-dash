# Tournament Dash — Architecture & Technical Reference

> Complete reference for the **tap-dash** codebase. This single document covers every screen, function, Firebase path, and design decision so any developer (or AI) can understand the full system without reading the source.

---

## 1. Overview

**Tournament Dash: School Edition** is a real-time tapping tournament game. Students on their phones tap as fast as they can, and a host screen (projected on the big screen) displays live race bars. Built as a **single `index.html` file** (~3,050 lines) with inline CSS and JS. No build step, no framework, no bundler.

### Tech Stack

| Layer | Technology |
|---|---|
| Structure | HTML5 (single file) |
| Style | Inline `<style>` with CSS custom properties |
| Logic | Vanilla JavaScript (inline `<script>`) |
| Database | Firebase Realtime Database (SDK 8.10.0) |
| Font | Google Fonts — [Orbitron](https://fonts.google.com/specimen/Orbitron) (400, 700, 900) |
| Hosting | GitHub Pages (static file) |

### Firebase Config
- **Project**: `tap-dash-de913`
- **Database URL**: `https://tap-dash-de913-default-rtdb.asia-southeast1.firebasedatabase.app`
- **Security Rules** (`database.rules.json`): Wide open (`".read": true, ".write": true`). Acceptable for this use case because there is no sensitive data — it's a school tapping game.

---

## 2. Screens & Navigation

The app has **6 screens**, managed by toggling the `.hidden` class. Only one screen is visible at a time.

```
┌─────────────────────────────────────────────────┐
│                  #setup                          │
│  (Team Select, 1v1 Duel btn, Host Controls btn) │
│                                                  │
│  ┌─ Team btn ──→ #client (Player tapping)       │
│  ├─ HOST btn ──→ #pin-modal → #host (Admin)     │
│  ├─ BRACKET ──→ #bracket-screen (Read-only*)    │
│  └─ DUEL btn ──→ #duel-screen (Lobby/Arena)     │
└─────────────────────────────────────────────────┘
```

### Screen IDs

| ID | Purpose | Access |
|---|---|---|
| `#setup` | Team selection, entry point | Everyone |
| `#client` | Player tapping screen (phone) | After selecting a team |
| `#pin-modal` | PIN entry modal overlay | Everyone (PIN blocks non-admins) |
| `#confirm-modal` | "SHUT DOWN" confirmation overlay | Admin only |
| `#host` | Admin scoreboard + match controls | Admin only (PIN: defined in JS) |
| `#bracket-screen` | Tournament bracket (read-only for non-admin, editable for admin) | Everyone views, admin edits |
| `#duel-screen` | 1v1 duel lobby, waiting room, and arena | Everyone |

---

## 3. CSS Architecture

### Design System (CSS Custom Properties)

```
:root
├── --bg-color: #050510
├── --text-main: #fff
├── Team Colors (7 teams):
│   ├── --pink: #ff00cc     (Aranui)
│   ├── --yellow: #fff01f   (Tainui)
│   ├── --orange: #ffaa00   (Mahi Tahi)
│   ├── --green: #39ff14    (Kia Kaha)
│   ├── --blue: #00f3ff     (Te Aroha)
│   ├── --white: #ffffff    (Year 9)
│   └── --purple: #9d00ff   (Staff)
└── Utility:
    ├── --cyan: #00f3ff
    ├── --cyan-dim: rgba(0, 243, 255, 0.3)
    └── --cyan-glow: ...
```

### Visual Effects
- **Grid background**: `.grid-bg` — perspective grid lines, fading gradient at top
- **Glassmorphism**: `backdrop-filter: blur()` used on bracket button, host elements
- **Neon glow**: `box-shadow` with colour variables throughout
- **Clip-path polygons**: Angled corners on team buttons, host button, bracket button
- **Animations**: `pulse` (tap buttons), `float` (winner trophy), `fadeIn` (winner overlay), `duelPulse` (waiting indicator)
- **Immersive mode**: `body.immersive` hides admin controls and cursor — toggled by pressing `H`

### Key CSS Classes
- `.hidden` — `display: none !important` (screen toggle)
- `.screen` — Full viewport flex container
- `.lane` / `.lane.inactive` — Scoreboard race lanes (visible/hidden per team)
- `.locked` — Client screen locked during standby (hides real tap button, shows practice)
- `.disabled-team` — Greyed out team button when not active

---

## 4. HTML Structure

```
<body>
├── .grid-bg                    (background decoration)
├── #setup                      (team selection screen)
│   ├── .btn-bracket-corner     (top-right bracket link)
│   ├── h1 "Select Team"
│   ├── .btn-grid               (7 team buttons)
│   ├── .btn-duel               (⚡ 1v1 PRACTICE DUEL)
│   └── .btn-host               (HOST CONTROLS)
├── #pin-modal                  (PIN entry overlay)
├── #confirm-modal              (exit confirmation overlay)
├── #bracket-screen             (tournament bracket)
│   ├── .bracket-wrapper        (4 rounds of matches)
│   └── .bracket-controls       (BACK, POPULATE, CLEAR)
├── #host                       (admin host screen)
│   ├── #winner-overlay         (winner announcement)
│   ├── #match-setup            (team checkboxes, director mode)
│   ├── #race-lights            (5 F1-style lights)
│   ├── .host-header            (LIVE badge + timer)
│   ├── #scoreboard             (race lanes built by JS)
│   └── #controls               (ENGAGE, RESET, NEW MATCH, EXIT)
├── #client                     (player screen)
│   ├── .btn-client-exit        (EXIT — visible in standby only)
│   ├── #msg                    (STANDBY / ENGAGED)
│   ├── #tps-display            (taps per second counter)
│   ├── #practice-btn           (practice tap — no score sent)
│   └── #tap-btn                (real tap — sends to Firebase)
└── #duel-screen                (1v1 duel)
    ├── .duel-back-btn          (← EXIT)
    ├── #duel-lobby             (name input, CREATE/JOIN/WATCH btns)
    ├── #duel-waiting           (room code display, player tags, START)
    └── #duel-arena             (timer, lanes, tap zone, result)
        └── #duel-spectator-badge  (SPECTATING — spectators only)
```

---

## 5. JavaScript Architecture

### Global State

| Variable | Type | Purpose |
|---|---|---|
| `myTeam` | `string\|null` | Currently selected team ID |
| `isHost` | `boolean` | Whether current user has admin access |
| `teams` | `Array<{id, name, color}>` | All 7 teams |
| `currentScores` | `Object` | Live scores by team ID |
| `teamWins` | `Object` | Win dots by team ID |
| `winningScore` | `number` (300) | Bar fills at 300 taps |
| `ROUND_DURATION` | `number` (15) | Seconds per tournament round |
| `pendingTaps` | `number` | Batched tap count awaiting flush |
| `isSequenceRunning` | `boolean` | Prevents double-start |
| `audioCtx` | `AudioContext\|null` | Web Audio API context |

### Duel-Specific State

| Variable | Type | Purpose |
|---|---|---|
| `DUEL_DURATION` | `number` (10) | Seconds per duel round |
| `DUEL_WINNING_SCORE` | `number` (200) | Bar width reference |
| `duelRoomCode` | `string\|null` | Current 4-char room code |
| `duelMyName` | `string\|null` | Player name (null for spectators) |
| `duelIsSpectator` | `boolean` | Whether viewing as spectator |
| `duelListeners` | `Array` | Firebase listener refs for cleanup |
| `duelPendingTaps` | `number` | Batched duel taps |
| `duelState` | `string` | `idle\|waiting\|countdown\|playing\|finished` |

### Director Mode State

| Variable | Purpose |
|---|---|
| `bracketMatches` | Array of match definitions (id, inputs, target, bestOf) |
| `currentDirectorMatch` | Currently active bracket match |
| `targetWins` | Number of wins needed (varies by round: 2 or 3) |

---

## 6. Function Reference

### Navigation
| Function | Description |
|---|---|
| `showSetup()` | Return to team selection screen |
| `showBracket()` | Show bracket (locks inputs for non-admin) |
| `goToSetup()` | Return from bracket to setup |
| `exitClient()` | Leave client screen, return to setup |
| `exitHost()` | Show exit confirmation modal |
| `confirmExit()` | Actually exits host mode |

### Client (Player) Functions
| Function | Description |
|---|---|
| `joinGame(teamId)` | Select a team, show client screen, register Firebase listeners |
| `handleTap()` | Increment `pendingTaps` + record tap timestamp (gameplay) |
| `handlePracticeTap()` | Record tap timestamp only (no score sent — standby practice) |
| `flushPendingTaps()` | Send batched taps to `scores/{teamId}` via transaction (every 300ms) |

### Host (Admin) Functions
| Function | Description |
|---|---|
| `showPinModal()` / `closePin()` / `checkPin()` | PIN entry flow |
| `initHost()` | Build scoreboard lanes, attach Firebase listeners for scores/wins/status |
| `updateActiveTeams()` | Sync active team checkboxes to Firebase + toggle lane visibility |
| `startSequence()` | Reset scores → 5-second F1 countdown → call `startGameplay()` |
| `runCountdownBeat(num)` | Light up one race light + play beep |
| `startGameplay()` | Set Firebase status to `playing`, start 15s timer |
| `endRound()` | Set status to `paused`, determine winner |
| `determineWinner()` | Find highest scoring active team, increment wins, show overlay |
| `resetScores()` | Reset score bars to 0 (keeps win dots) |
| `resetMatch()` | Reset everything including win dots |
| `showWinnerOverlay()` | Display winner announcement with trophy animation |

### Tournament Director Functions
| Function | Description |
|---|---|
| `evaluateBracketState()` | Scan bracket for next playable match, auto-advance BYEs |
| `activateMatch(match)` | Set active teams to the two teams in the match, reset scores |
| `directorHandleWin(teamId)` | Write winner to bracket slot, re-evaluate for next match |

### Bracket Functions
| Function | Description |
|---|---|
| `saveBracket(id, val)` | Write a bracket slot to Firebase (admin only) |
| `populateBracket()` | Shuffle teams into QF slots randomly |
| `clearBracket()` | Remove all bracket data from Firebase |

### Audio Functions
| Function | Description |
|---|---|
| `initAudio()` | Create/resume AudioContext |
| `playBeep(type, val)` | Play synth sound: `count` (countdown), `go` (start), `win` (victory fanfare) |

### Duel Functions
| Function | Description |
|---|---|
| `showDuelLobby()` | Show duel lobby, reset state |
| `showJoinInput()` | Reveal JOIN code input row |
| `showWatchInput()` | Reveal WATCH code input row |
| `createDuelRoom()` | Generate 4-char code, write room to `duels/{code}/`, listen |
| `joinDuelRoom()` | Validate code, register as player, listen |
| `watchDuelRoom()` | Validate code, join as **read-only spectator** (no Firebase writes) |
| `showDuelWaiting()` | Show waiting room with room code |
| `listenToDuelRoom()` | Attach listeners for `players`, `status` |
| `startDuel()` | Write `status: countdown` to Firebase |
| `enterDuelArena()` | Show arena view (hides tap zone for spectators) |
| `runDuelCountdown()` | 3-2-1 countdown with beeps |
| `startDuelGameplay()` | Start 10s timer, begin tap flushing |
| `handleDuelTap()` | Increment `duelPendingTaps` |
| `flushDuelTaps()` | Send batched taps to `duels/{code}/scores/{name}` (every 300ms) |
| `endDuel()` | Stop timers, determine winner, show result |
| `rematchDuel()` | Reset and restart |
| `exitDuel()` | Clean up listeners, remove player (skip for spectators), return to setup |

### Utility
| Function | Description |
|---|---|
| `generateRoomCode()` | Random 4-char code from `ABCDEFGHJKMNPQRSTUVWXYZ23456789` |
| `setLights(state)` | Control F1-style race lights (0=off, -1=green) |
| `updateDots(teamId)` | Update win dot indicators |

---

## 7. Firebase Data Structure

```
root/
├── game/
│   ├── status                "idle" | "playing" | "paused"
│   └── activeTeams/
│       ├── aranui            true | false
│       ├── tainui            true | false
│       ├── mahitahi          true | false
│       ├── kiakaha           true | false
│       ├── tearoha           true | false
│       ├── year9             true | false
│       └── staff             true | false
│
├── scores/
│   ├── aranui                0..n (tap count)
│   ├── tainui                0..n
│   └── ...
│
├── wins/
│   ├── aranui                0..n (round wins)
│   ├── tainui                0..n
│   └── ...
│
├── bracket/
│   ├── b-q1-a                "Team name"
│   ├── b-q1-b                "Team name"
│   ├── b-s1-a                "Team name"
│   ├── b-f1-a                "Team name"
│   ├── b-winner              "Team name"
│   └── ...                   (14 bracket slot IDs total)
│
└── duels/
    └── {roomCode}/           (4-char alphanumeric)
        ├── players/
        │   ├── ALPHA          true
        │   └── BRAVO          true
        ├── scores/
        │   ├── ALPHA          0..n
        │   └── BRAVO          0..n
        ├── status             "waiting" | "countdown" | "playing" | "finished"
        ├── created            (server timestamp)
        └── winner             "ALPHA" (set on finish)
```

### Real-Time Sync Pattern
- **Tap batching**: Clients accumulate taps locally, then flush to Firebase via `transaction()` every **300ms**. This reduces writes from ~10/sec to ~3/sec per client.
- **Score listeners**: Host screen uses `.on('value')` to get live updates.
- **Status-driven UI**: Client screens switch between STANDBY/ENGAGED based on `game/status`.

---

## 8. Tap Batching Detail

Both tournament and duel modes use the same pattern:

```
Client taps → pendingTaps++ (local)
              ↓ (every 300ms)
         flushPendingTaps()
              ↓
  db.ref('scores/{team}').transaction(c => c + count)
```

This `transaction()` ensures concurrent taps from multiple phones on the same team are safely merged.

---

## 9. Tournament Director (Auto-Run)

When `#chk-auto-director` is checked:

1. `evaluateBracketState()` scans `bracketMatches` top-to-bottom
2. Automatically advances BYE matchups
3. Finds the first match where both inputs have values but the target is empty
4. `activateMatch()` checks the correct team checkboxes and resets the scoreboard
5. When a team wins enough rounds (`bestOf / 2`, rounded up), `directorHandleWin()` writes the winner's name to the next bracket slot
6. Re-evaluates after 2 seconds to find the next match

### Bracket Match Definitions
| Match | Inputs | Target | Best Of |
|---|---|---|---|
| q1 | b-q1-a, b-q1-b | b-s1-a | 3 |
| q2 | b-q2-a, b-q2-b | b-s1-b | 3 |
| q3 | b-q3-a, b-q3-b | b-s2-a | 3 |
| q4 | b-q4-a, b-q4-b | b-s2-b | 3 |
| s1 | b-s1-a, b-s1-b | b-f1-a | 3 |
| s2 | b-s2-a, b-s2-b | b-f1-b | 3 |
| f1 | b-f1-a, b-f1-b | b-winner | 5 |

---

## 10. Client Dual-Button System

The client screen has **two** circular tap buttons that swap visibility:

| State | Visible Button | Behaviour |
|---|---|---|
| **STANDBY** (locked) | `#practice-btn` | Taps recorded locally for TPS display only. No Firebase write. EXIT button visible. |
| **ENGAGED** (unlocked) | `#tap-btn` | Taps batched and flushed to Firebase. EXIT button hidden. |

This is controlled by CSS:
- `#client.locked #tap-btn` → hidden
- `#client:not(.locked) #practice-btn` → hidden
- `#client:not(.locked) .btn-client-exit` → hidden

---

## 11. Duel Spectator Mode

Spectators join via **👁 WATCH** → enter room code. They are **read-only**:

| Aspect | Player | Spectator |
|---|---|---|
| Writes to `players/` | ✅ | ❌ |
| Writes to `scores/` | ✅ | ❌ |
| Sees START button | ✅ | ❌ |
| Sees tap button | ✅ | ❌ |
| Sees progress bars | ✅ | ✅ |
| Sees winner | ✅ | ✅ |
| Spectator badge | ❌ | ✅ (pulsing cyan) |
| On EXIT, cleans Firebase | ✅ | ❌ (nothing to clean) |

---

## 12. Keyboard Shortcuts (Host Only)

| Key | Action |
|---|---|
| `Space` | Start round (same as ENGAGE button) |
| `R` | Reset current scores |
| `H` | Toggle immersive mode (hide admin controls + cursor) |

---

## 13. Teams

| ID | Display Name | CSS Class | Colour Variable |
|---|---|---|---|
| `aranui` | Aranui | `.btn-aranui` | `--pink` (#ff00cc) |
| `tainui` | Tainui | `.btn-tainui` | `--yellow` (#fff01f) |
| `mahitahi` | Mahi Tahi | `.btn-mahitahi` | `--orange` (#ffaa00) |
| `kiakaha` | Kia Kaha | `.btn-kiakaha` | `--green` (#39ff14) |
| `tearoha` | Te Aroha | `.btn-tearoha` | `--blue` (#00f3ff) |
| `year9` | YEAR 9 | `.btn-year9` | `--white` (#ffffff) |
| `staff` | STAFF | `.btn-staff` | `--purple` (#9d00ff) |

---

## 14. File Inventory

| File | Purpose |
|---|---|
| `index.html` | **The entire app** — CSS, HTML, and JS |
| `database.rules.json` | Firebase security rules (open read/write) |
| `README.md` | User-facing documentation |
| `ARCHITECTURE.md` | This document |
