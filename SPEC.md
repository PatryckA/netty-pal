# Netty Pal: specification

A web app for junior netball teams: manage rosters, track court time by position,
score games live, and let parents help by joining a team with a season code.

Status: agreed with the owner on 2026-09-27. Nothing built yet.

---

## 1. Tech approach (same pattern as Packing Pal)

- A single-page web app (`index.html`), hosted on **GitHub Pages**.
- Installable on a phone home screen (a PWA, with a small `sw.js` service worker).
- **Firebase** for:
  - **Google sign-in** (Firebase Authentication)
  - **Cloud Firestore** as the shared, live-updating database
- New GitHub repo and new Firebase project, separate from Packing Pal.
- No build step, no framework. Firebase is loaded from Google's CDN.
- Mobile reception at courts is fine, so offline scoring is **not** a requirement
  (Firestore's built-in short-term offline cache is a bonus, not a design goal).
- Permissions are enforced by **Firestore security rules** (on Google's side), not
  only by hiding buttons in the page.

---

## 2. People and roles

| Role | Who | Can do |
|---|---|---|
| **Coach/Manager** | Whoever creates the team starts as Coach/Manager. A team can have several; any Coach/Manager can make others. | Runs the team: edit team and seasons, roster, fixtures, lineups and positions, assign roles, remove members, regenerate the season code, unlock finalised games. |
| **Scorer** | Assigned by a Coach/Manager. | Score games (one scorer per game; can differ game to game), make lineup changes during play if needed. |
| **Viewer** | Anyone who joined with the season code and has no role. | Follow live scores, view game and season summaries. Read-only. |

- One person can be both Coach/Manager and Scorer.
- Anyone signed in can create a new team (and becomes its Coach/Manager).
- A user can belong to several teams (e.g. siblings).
- (Changed 2026-10-03.) There is no separate Admin role any more: it was merged into
  Coach/Manager. A team always keeps at least one Coach/Manager (the app won't let the
  last one be removed, step down or leave).
- **App owner**: one Google account (set in `index.html` and `firestore.rules`) can read
  and manage every team without being a member, and can list every team from the home
  screen. Nobody else sees this.
- Any member can send the invite link from the home screen ("Invite someone to a team").
  A Coach/Manager of the chosen team is asked which role: Viewer (the team link), Scorer or
  Coach/Manager (a one-time role invite link, below).
- (Added 2026-10-03.) **Role invite links**: in Support Crew, a Coach/Manager can send
  "Invite a Coach/Manager" or "Invite a Scorer". It's a one-time link (a long random code
  in the usual `?join=` link): the first person to use it joins with that role, and the
  link stops working. Unused links are listed there and can be sent again or cancelled.
  Stored as `roleInvites/{code}` (teamId, seasonId, role) plus the season's
  `invites/{code}` (role, createdAt, createdByName); both are deleted when it's used.

### Season code
- One code per team season. **Never expires.**
- Joining: sign in with Google, enter code, become a Viewer of that season.
- A Coach/Manager can remove a member and regenerate the code if it gets passed around.

### Your account (added 2026-10-03)
- **Delete my account** (bottom of the home screen): leaves every season (removes the
  person's member record and their list of teams), deletes the players, games, invite links
  and code of any season where nobody else is a member, then deletes the sign-in account.
  Season and team names stay (the rules don't allow deleting them). Not allowed while they're
  the only Coach/Manager of a season other people are in. Google and Facebook users confirm
  with a sign-in popup first; email-link users are asked to sign in again. No rule changes
  were needed: it only uses permissions members and Coach/Managers already have.
- **Install on your phone**: a home screen card (until installed or put away) and a link at
  the bottom of the home screen open step-by-step instructions for iPhone/iPad and Android.
  Android browsers that support it get a one-tap Install button.
- **Owner: delete a team.** In "Owner: show every team", each team has a Delete button
  (two taps). It deletes every season and everything in it (members, players, games, game
  events, invite links, codes) and then the team. Seasons with nobody in them show a
  "No users" pill, and each season shows how many users it has. Only the owner can delete
  seasons, teams and game events (enforced in `firestore.rules`).
- The install card also shows on the sign-in screen (not when arriving from an invite
  link, so the invite isn't lost, and not inside Facebook/Instagram's own browser).
- "Delete my account" is a small text link at the bottom of the home screen, next to Privacy.
- Confirmation buttons read "Confirm [action]" on the second tap (e.g. "Confirm delete game").

---

## 3. Teams, seasons, roster

- **Team**: name, who created it.
- **Season**: name (e.g. "Winter 2027"), competition name, default format
  (**quarters** or **halves**), default period length (**7 minutes**), season code.
- **Roster** per season: player names. Fill-in players are simply added to the
  roster and count in season totals like everyone else. Players can be marked
  inactive rather than deleted, so past stats stay intact.

---

## 4. Fixtures

- A Coach/Manager can add fixtures for the season: round, date, time, venue/court, opponent.
- **Import** (built, option A: no AI): upload a PDF or CSV, or paste text from the
  competition draw. Only games involving our team are kept (the name in the draw is
  editable, with a pick-list of teams found). Understands round headings with games
  under them, one game per line, and tables with a heading row (PDF columns are
  matched by position). Our byes are kept as bye rounds; games and byes already added are unticked. The
  app shows an **editable list to review** before saving. Nothing is saved without
  review. PDFs are read in the browser with PDF.js from the jsDelivr CDN.
- Built without a real fixture from the competition; tune it once one is available.
  Screenshot import (AI, option B) was considered and deferred.
- Manual "add fixture" and "new game on the day" are always available.
- Each game inherits the season's format and period length, and can override both.
- (Added 2026-10-03.) A fixture can be marked as a **bye** (tick "This round is a bye" when adding
  it or in Game settings, before it starts). A bye keeps its round and date, shows as "Bye" in
  the games list, has no lineup or scoring, and is left out of stats and Time on court.

---

## 5. Game day

### Before the game (decided 2026-09-27)
- **Availability**: Coach ticks which players are playing. Only they appear on the
  bench, and season stats can show games played. (Parents don't mark their own child.)
- **Lineup plan**: Coach can plan positions for every period ahead of time (e.g. the
  night before). A **fairness view** shows who has played where so far this season
  to help spread positions and court time. On the day the plan loads each period and
  can still be changed.
- Positions may be left **empty** (e.g. only 6 players turn up). No time or stats are
  counted for an empty position.

### Setup on the day
1. Pick the fixture (or create a new game).
2. Confirm format (quarters/halves) and period length (default 7 min).
3. Scorer claims the game (only one scorer per game).
4. Confirm the starting lineup on the court board: GS, GA, WA, C, WD, GD, GK.
5. Choose who takes the first centre pass (us or opposition).

### Scorer handover
- Anyone with the Scorer or Coach role can tap **Take over scoring**. The previous
  scorer's screen switches to view-only. Nothing is lost because everything saves live.
- (Added 2026-10-03.) Only one person scores at a time. At each break the scorer (or a
  Coach/Manager) can tap **Hand over scoring** and pick the next scorer; Coach/Managers
  can also do this from the Support Crew tab while a game is live.
- Only the person scoring can start, pause, resume or end a period (enforced in the
  security rules). Coach/Managers and Scorers can all make substitutions.
- Breaks are named Quarter time, Half time, Three quarter time and Full time, and each
  break offers **Share score**. The shared picture says how far through the game it is.

### Clock
- **Counts down** from the period length (e.g. 7:00). At 0:00 it stops, the phone
  vibrates and the scoreboard shows "Time". Scoring stays open so a shot from the
  whistle can still be recorded (at 0:00); the scorer then taps End to end the period.
- Start, pause, resume, end period early. Pause is for injuries and umpire stoppages.
- **Court time only accrues while the clock is running.**
- Court time is calculated from timestamped events (on/off, clock start/stop), not
  from a ticking counter, so a locked phone or backgrounded app doesn't lose time.

### Lineup changes
- Allowed **at any time, including mid-quarter**.
- Coach does this normally; Scorer can also do it.
- Swap a bench player on, or swap two players' positions.

### Scoring (revised by the owner after step 2b)
- **Only goals and misses are recorded**, by GS and GA, for both teams. Intercepts
  and penalties were dropped. (Older games may still hold them; they are ignored.)
- The scorer's in-play screen shows only **our GS and GA** (highlighted, with the
  player's name and our team name), then one tile for the other team. Below is an
  event log, newest first. Everyone else sees **Your team's lineup** on the court
  diagram, with the bench (no court time; that's on the Game stats tab).
- Two scoring modes; the scorer swipes left/right (or taps the tabs) to switch, and
  the choice is remembered on that phone:
  - **Player first**: tap a shooter, then Goal or Miss.
  - **Event first**: tap Goal or Miss, then who (our GS/GA, their GS/GA/Team).
- The scorer makes substitutions with "Sub Player(s)". Coach/Managers and Scorers
  who aren't scoring can also tap a position on the court to make one.
- The stat is credited to whoever is in that position at that moment. Opposition
  shots record GS, GA or just the team.
- Floating **Undo** and **Redo**.

### After the game
- Scorer or Coach can **correct the timeline** (add, remove or reassign events),
  then tap **Finalise** to lock the game. A Coach/Manager can unlock it if needed.

### Centre pass
- Automatic: alternates after every goal (either team), and each period starts
  with the team that did not take the first centre of the previous period.
- Manual override button if it gets out of sync.
- Always visible on the scoring screen.

### Live view
- Everyone in the season (including Viewers) can follow, read-only: live score,
  period, clock and who is on court in which position. Stats are on the Game stats
  tab (there's no stats section on the Live tab).

---

## 6. Summaries (Coaches and Viewers)

### Per game
- Final score, by period.
- Court time per player, split by position.
- Per player: goals, misses, shooting % (goals / (goals + misses)).
- Opposition goals, misses and shooting % (by GS / GA where recorded).

### Season to date
- The same figures totalled across all completed games.
- Court time totals per player and per position (to help keep court time fair).

### As built (step 3)
- Game **Summary** tab (once a game has started): result, score by period, shooting
  (our shooters with positions, opposition by GS / GA / Team, team totals), court
  time per player per position (players marked as playing with no court time are
  highlighted).
- (Revised 2026-10-04.) The separate Time on court tab is now part of **Game stats**,
  which (outside practice games) is there before the game too. One **Time on court**
  table at a time, chosen with switches:
  - **Game / Season** (everyone; default Game). Before the game starts only Season
    is shown; practice games have only Game.
  - **Simple / Detailed** (Coach/Managers and Scorers only; default Simple). Simple is
    minutes per player. Detailed splits the time by position: minutes this game, or
    periods (quarters or halves) so far this season. Viewers never see Detailed.
  - The choices are remembered while moving between games (until the app reloads).
- Team **Stats** tab: record (played, won, lost, drawn, goals for : against, both
  teams' shooting %), results list (tap to open a game), players (games on court,
  minutes, goals, %) and court time by position in minutes.
- Season totals include games that have reached full time, finalised or not; games
  in progress are left out and noted. They're worked out from each game's events,
  loaded when the Stats tab is opened. Everyone in the season can see them.

---

## 7. Draft data model (Firestore)

```
users/{uid}                          name, email
joinCodes/{code}                     teamId, seasonId
teams/{teamId}                       name, adminUid (creator only), editSeason
  seasons/{seasonId}                 name, competition, format, periodMinutes (7), joinCode
    members/{uid}                    name, roles: ["coach","scorer"] (empty = viewer)
    players/{playerId}               name, active
    games/{gameId}                   round, date, time, venue, court, opponent,
                                     format, periodMinutes, available[], plan{},
                                     status: scheduled | live | final,
                                     scorerUid, scorerName,
                                     live { period, phase: running | paused | ended,
                                            startedAt, elapsedMs, firstCentre: us | them },
                                     summary { us, them, courtTime { playerId: { pos: ms } } }
      events/{eventId}               type, period, t (ms of play into the period),
                                     at, by, undone, plus per type below
```

Event types (as built in step 2b): `lineup` (full lineup map; `start: true` at a
period start), `goal`, `miss`, `intercept`, `held`, `obstruction`, `contact` (with
`pos` and `playerId`), `oppGoal`, `oppMiss`, `centre` (team taking the next centre),
`periodEnd`. The clock itself lives on the game doc (`live`), not in events.
A period with no planned changes starts from the actual lineup at the end of the
previous period, so substitutions carry forward.

All summaries are calculated from the event list, so undo just marks an event as
undone and everything recalculates.

---

## 8. Build order

1. **Foundations**: Google sign-in, create team, seasons, roster, season code,
   joining, roles, remove member, regenerate code, admin transfer. Security rules.
2. **Game day**, in two parts:
   - **2a. Before the game**: fixtures (manual add/edit), availability, lineup
     planner per period, fairness view.
   - **2b. The live game**: clock, court board, lineup changes, scoring pad, centre
     pass, undo, scorer handover, live view, post-game corrections and Finalise.
3. **Summaries**: per game and season to date. (Built.)
4. **Fixture import**: PDF, CSV or pasted text, review table. (Built; tune with a real draw.)

---

## 9. Owner setup tasks (need the owner's own accounts)

- Create a new GitHub repo (`netty-pal`).
- Create a new Firebase project, enable Google sign-in and Firestore, and add the
  GitHub Pages address as an authorised domain.
- Paste the Firebase web config (not a secret; it identifies the project) into the app.

---

## 10. Working conventions

- Owner is not a professional developer: explain decisions in plain English.
- Work on a branch and open a pull request; the owner merges.
- No em dashes in copy or comments.
- Never commit passwords, tokens or private keys.
