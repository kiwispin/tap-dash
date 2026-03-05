# Tournament Dash: School Edition

A real-time, neon-themed tapping tournament game built with vanilla HTML/CSS/JS and Firebase Realtime Database. Students join on their phones, tap as fast as they can, and the host screen displays live race bars.

## How It Works

### For Players (Phones)
1. Open the app URL on your phone
2. Select your team (Aranui, Tainui, Mahi Tahi, Kia Kaha, Te Aroha, Year 9, Staff)
3. Wait for "ENGAGED" — then tap as fast as you can!
4. Your taps-per-second (TPS) is shown during standby for practice

### For the Host (Big Screen)
1. Click **HOST CONTROLS** → enter the admin PIN
2. Select which teams are active for this round
3. Press **ENGAGE** (or Spacebar) to start the countdown + round
4. The live scoreboard shows all team bars racing in real-time
5. Press **H** to toggle HUD (hide admin controls for a cleaner projection)

### Tournament Bracket
- Click **TOURNAMENT BRACKET** from the team selection screen to view
- The host can populate, edit, and clear bracket slots
- Non-admin users see a read-only bracket view

## 1v1 Practice Duel

A self-contained head-to-head mode that runs outside the tournament. No host or admin access needed.

1. Click **⚡ 1v1 PRACTICE DUEL** from the team selection screen
2. **CREATE** a room (generates a 4-character code) or **JOIN** with an existing code
3. Once both players are in, either can hit **START DUEL**
4. 3-second countdown → 10-second tapping round → winner announced
5. **REMATCH** to go again, or **EXIT** to return

All duel data lives under `duels/{roomCode}/` in Firebase — completely isolated from tournament data.

### Spectator Mode (Big Screen Projection)

Teams can project a duel on the big screen without admin access:

1. Open the app on the projector/big-screen device
2. Click **⚡ 1v1 PRACTICE DUEL** → click **👁 WATCH**
3. Enter the 4-character room code → **GO**
4. The spectator sees a clean, tap-free arena view: timer, both progress bars, live scores, and the winner announcement
5. A pulsing **SPECTATING** badge appears in the top corner
6. Spectators are read-only — they never write to Firebase and don't affect the room

## Tech Stack

- **Frontend**: Single `index.html` file (HTML + CSS + JS)
- **Database**: Firebase Realtime Database (batched tap transactions every 300ms)
- **Font**: [Orbitron](https://fonts.google.com/specimen/Orbitron) (Google Fonts)
- **No build step** — just open or deploy the HTML file

## Firebase Structure

```
├── game/
│   ├── status          (idle | playing)
│   └── activeTeams/    (which teams are enabled)
├── scores/             (live tap counts per team)
├── wins/               (round win tallies)
├── bracket/            (tournament bracket data)
└── duels/
    └── {roomCode}/
        ├── players/    (player names)
        ├── scores/     (live duel tap counts)
        ├── status      (waiting | countdown | playing | finished)
        └── winner
```