# Netty Pal: specification

A web app for junior netball teams: manage rosters, track court time by position,
score games live, and let parents help by joining a team with a season code.

Status: agreed with the owner on 2026-09-27. Nothing built yet.

---

## 1. Tech approach (same pattern as Packing Pal)

- A single-page web app (`index.html`), at **https://nettypal.au**, hosted on **Vercel**
  (moved from GitHub Pages in October 2026; the old address forwards to the new one).
- Sign-in emails come from `noreply@nettypal.au` (Firebase custom email domain, with
  SPF, DKIM and DMARC records at Porkbun). Firebase project is on the Blaze plan.
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
- (Added 2026-10-13) **Email and password**: "Continue with email" now signs in with an email
  and password (Firebase Email/Password), so nobody has to leave the app; on iPhones an emailed
  link opens in Safari, not the installed app. "Create an account" (name, email, password of at
  least 6 characters), and "Forgot password?" emails a link to set a new one (this is also how
  someone who used email links gets a password). The emailed sign-in link stays as a backup
  ("Email me a sign-in link"). A note at the top of the form tells people who used emailed
  links before to tap Forgot password to set one. Typing a Gmail address shows "Using Gmail? Continue with Google
  is quicker". Problems show in the form. Needs Email/Password switched on in Firebase console >
  Authentication > Sign-in method.
- **Install on your phone**: a home screen card (until installed or put away) and an item in
  the account menu (tap your photo/initials; hidden once installed) open step-by-step instructions for iPhone/iPad and Android.
  Android browsers that support it get a one-tap Install button.
- **Owner: delete a team.** In "Owner: show every team", each team has a Delete button
  (two taps). It deletes every season and everything in it (members, players, games, game
  events, invite links, codes) and then the team. Seasons with nobody in them show a
  "No users" pill, and each season shows how many users it has. Only the owner can delete
  seasons, teams and game events (enforced in `firestore.rules`).
- The install card also shows on the sign-in screen (not when arriving from an invite
  link, so the invite isn't lost, and not inside Facebook/Instagram's own browser).
- (Added 2026-10-10) **Archive this team** (Support Crew tab, anyone in the team): hides that
  team season from the person's own home screen only. They stay a member; it's listed under
  "Show archived teams" on the home screen, with Unarchive (also a banner inside the team).
  Stored as `archived` on their own `users/{uid}/memberships/{seasonId}`; no rule changes.
- (Added 2026-10-11) **Press and hold** a team on the home screen: Archive / Unarchive (anyone);
  the app owner also gets Delete team (two taps; deletes it for everyone). Press and hold a game
  (Coach/Managers): Archive / Restore, or Delete game (two taps).
- (Added 2026-10-11) **Help**: a short FAQ in a slide-up panel, from the sign-in screen (link
  next to Privacy) and the pink Help button in the top bar on every signed-in screen. Sections fold away; only relevant ones
  show: signed out sees Getting started; signed in adds Following a game and Tidying up;
  Scorers add Helping out; Coach/Managers (in any team) add Coach/Managers. Contact email
  hello@nettypal.au (Porkbun email forwarding).
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

### Preferred positions and the Players list (added 2026-10-13)
- Each player can have up to **3 preferred positions**, most preferred first, stored as
  `positions: ['GS','GA']` on the player. Coach/Managers tap **Set preferred positions** above
  the list, then tap up to 3 bibs per player (numbered 1 to 3); changes save straight away.
- The list shows them as little coloured bibs by each name, and on the player's stats page.
  Everyone in the team can see them.
- When picking who plays a position (lineup planner, and subs during a game), players who
  prefer that position are listed first in each group (1st choice, then 2nd, then 3rd), with
  a star, "Prefers GS (1st choice)" and their bibs; then least time in that position, as before.
- Tap a player to open their stats page. Coach/Managers **press and hold** a player for
  Rename, Remove from team / Add back (keeps their stats), and **Delete player** (two taps),
  only for players who haven't been in a game that started; deleting also takes them out of
  upcoming games' Who's playing and lineups. The "Remove from team" button in each row is gone.
- No rule changes (Coach/Managers can already write players).

### Lineup tab without the court (changed 2026-10-15)
- The Lineup tab no longer shows the quarter-by-quarter court (or its quarter picker, bench,
  Clear Q1 / Reset to match buttons and position picker). It's now **Fill lineup for me**,
  **Clear lineup** (two taps, before the game) and the **Whole game** table, where spots are
  tapped for suggestions or held and dragged to swap. The court stays on the Game tab (lineup
  before the game, and during play). During a game, quarters that have started are read-only
  here, with a note to use Sub Player(s) on the Game tab.

- (Added 2026-10-15) Every planned lineup has a small note underneath: "This is a plan only.
  Lineups often change in the moment." It shows under the Whole game table (while any quarter
  is still to come), the Fill lineup preview, and the Game tab's court before the game and at
  breaks (everyone sees it, parents included). Not in practice games or finished games.

### Whole game table on the Lineup tab (added 2026-10-14)
- Under the court, a **Whole game** table lists every player playing with their position in
  each quarter (– on the bench) and their total quarters; it replaces the "Quarters on court
  this game" chips. The quarter on the court is highlighted, and tapping a quarter heading
  shows it on the court. Grey bibs aren't one of the player's preferred positions (players
  with no preferences show coloured bibs). Before the game, Coach/Managers and Scorers also see
  "Avg mins after": season minutes per game available, counting this game. The same table is
  used in the Fill lineup preview.
- (Added 2026-10-14) Tap a **name** in the Whole game table (and the Fill lineup preview) to show
  that player's preferred positions under it; tap again to hide. Redraws keep an open panel
  scrolled where it was, so "Try another" and the options don't jump back to the top.
- (Added 2026-10-14) **Drag to swap** in the Whole game table (Coach/Managers, quarters that
  haven't started): press and hold a spot (about a quarter of a second) until it lifts, then
  drag it onto another spot in the same quarter. Two positions swap; a bench spot (–) dropped
  on a position brings that player on (and the other off), and the other way round. A quick
  swipe still scrolls and a quick tap still opens suggestions. Saves that quarter only.
  The same drag works in the **Fill lineup preview** (added 2026-10-15): it changes the preview
  only (a note says it's been changed by hand), **Use this lineup** saves the changed version,
  and **Try another** or changing an option starts again.
- (Added 2026-10-14) **Suggest a change**: Coach/Managers tap a player's spot in the table (any
  quarter that hasn't started) for suggestions: swap with someone on court that quarter, bring
  a bench player on, or bring them on from the bench. Each option is scored on the whole game
  like Fill lineup (preferred positions, even season minutes, nobody out two quarters in a
  row, only GS/GA choosers at GS/GA, no empty positions). The best few are shown with reasons;
  "Make this change" saves that quarter (later quarters that followed it keep their lineup).
  If nothing improves the game it says so and shows the closest options.

### Fill lineup for me (added 2026-10-13)
- On a game's Lineup tab (before it starts), Coach/Managers tap **Fill lineup for me**. A preview
  shows each player's position per quarter and their season average minutes (per game
  available) now and after this game; **Use this lineup** saves it, **Try another** shows the
  next-best plan. Anything can still be changed by hand afterwards.
- **Fair court time**: everyone playing gets the even share of quarters (rounded down). Spare
  quarters go one at a time to whoever has the lowest season average (from finished games'
  saved summaries), so anyone behind catches up.
- **Bench**: nobody sits out two quarters in a row unless it can't be avoided (noted).
- **Shooters**: only players with GS or GA as a preferred position play GS or GA. If there
  aren't enough, they play extra quarters, or the position is left empty. When that makes court
  time uneven, the preview shows a warning box naming who plays extra and who plays fewer
  (rows marked + and −), with how to fix it by hand.
- **Short of players**: WD is left empty first, then WA, then GA.
- (Added 2026-10-14) **Sharing out playing out of position**: when someone has to play a
  position they didn't choose, Fill lineup and suggestions pick whoever has done it least this
  season (minutes out of preferred positions per game available). Each game's saved summary
  keeps a snapshot of players' preferences from its first save (`summary.prefs`), so changing
  preferences later doesn't rewrite past games; games saved before this use today's
  preferences. Players with no preferences never count as out of position. The figure isn't
  shown anywhere; a suggestion may say who "has done it less this season".
- (Added 2026-10-14) **Few position changes**: a position change is playing a different
  position from the last quarter a player was on court (bench quarters don't count). The first
  change in a game is free; each extra one costs points (about half a preference step), so
  plans and suggestions keep players to fewer than 2 changes where they can. Not a hard rule.
  Variety mode rotates players once (e.g. GA, GA, WA, WA) rather than every quarter.
- **Prioritise**: *Top preferences* (1st choice where possible, then 2nd, then 3rd) or
  *Variety* (rotates players through their preferred positions across the quarters).
- **Lineup already set**: *Keep it, fill the gaps* (positions already set stay) or *Start again*.
- Positions within each quarter are worked out exactly (best arrangement); many candidate
  quarter patterns are tried and the best are offered. Saves `plan` (and `available` if it
  wasn't set). No rule changes.

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
- (Added 2026-10-10.) **Hand over scoring** is also on the live screen during play, next to
  Sub Player(s), for the scorer and Coach/Managers. The clock keeps running.
- Only the person scoring can start, pause, resume or end a period (enforced in the
  security rules). Coach/Managers and Scorers can all make substitutions.
- Breaks are named Quarter time, Half time, Three quarter time and Full time, and each
  break offers **Share score**. The shared picture says how far through the game it is.
  (Changed 2026-10-10.) Sharing sends only the picture, no text. Where a phone or computer
  can't share pictures, the picture is saved instead.

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
- The full-time **Timeline** is shown only to Coach/Managers and Scorers (viewers
  don't see it). Scorer or Coach can **correct the timeline** (add, remove or reassign events),
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
    minutes per player. Detailed splits the time by position: in minutes, for this game
    or the season so far (changed 2026-10-10: the season view used to count periods). Viewers never see Detailed.
  - The choices are remembered while moving between games (until the app reloads).
- Team **Stats** tab: record (played, won, lost, drawn, goals for : against, both
  teams' shooting %), results list (tap to open a game), players (games on court,
  minutes, goals, %) and court time by position in minutes.
- Season totals include games that have reached full time, finalised or not; games
  in progress are left out and noted. They're worked out from each game's events,
  loaded when the Stats tab is opened. Everyone in the season can see them.

### Track more stats (added 2026-10-10)
- Off by default. **Team Settings > This season > Track more stats** (folded away at the
  bottom) picks extra stats for the season; **Game settings** can choose differently for one
  game (it only keeps its own choice if it differs from the season's).
- Goal is always tracked (it makes the score). **Miss can be unticked**: then there are no
  Miss buttons, no shooting accuracy, and misses already recorded don't count. Season accuracy
  only uses games that tracked misses. Extras, by category:
  Shooting: Feed. Gains: Rebound, Intercept, Deflection. Losses: Bad pass, Footwork,
  Held ball, Out of court, Error. Penalties: Contact, Obstruction, Offside.
- **Track for**: Our team, Other team, or Both. The other team's stats are for the team
  as a whole (no names). Feeds are for our team only.
- **Live scoring**: with extras for our team, every position gets a tile. Goal/Miss first
  shows the extras as smaller buttons under Goal and Miss; Player first opens a panel with
  that player's stats, grouped. After each of our goals, if Feed is tracked, the scorer is
  asked "Who fed it?" (or Skip). A feed stops counting if its goal is undone.
- **Voice scoring** understands tracked extras for any position ("WD intercept",
  "their contact") and "GS goal fed by WA".
- **Stats**: Game stats and Season Stats get a "More stats" table (one column per stat,
  grouped, with our team total and the other team's row). A **Simple stats / All stats**
  switch (remembered on each phone, All to start with) shows or hides it. Season totals
  count each stat only from games that tracked it.
- Stored as `stats { extra:[keys], teams:'us'|'them'|'both' }` on the season (and optionally
  the game). Events: extras for our players use the stat key as `type` (with `pos`,
  `playerId`); `feed` also has `goal` (the goal's event id); the other team's are
  `{ type:'oppStat', stat }`. No security rule changes were needed.

---

### Player awards (added 2026-10-10)
- **Off by default.** Switched on for the season in Team Settings (folded "Player awards"
  section), or for one game in Game settings (which otherwise follows the season).
- Up to **5 awards**, named by the Coach/Manager ("Coaches Award" to start with).
- **Only Coach/Managers see awards** (enforced in `firestore.rules`: the season's `awards`
  collection is readable and writable by Coach/Managers only). Parents and Scorers never see them.
- Per game: on Game stats, tap players to give each award (joint winners allowed). Players
  are sorted so anyone who hasn't had that award yet this season comes first ("not yet").
  At full time a nudge offers "Give awards".
- Season: Season Stats shows a "Season awards" table per player and "Not yet given an award".
  Archived games and games with awards switched off aren't counted.
- No share card or players' player voting (decided with the owner).
- Stored as `awards/settings { enabled, types[] }` and `awards/{gameId} { winners, enabled? }`.

---

### UX review changes (added 2026-10-11)
- **Top bar**: a pink **Help** on every signed-in screen, and your photo/initials, which opens a
  menu with your name, email, **Install on your phone** (hidden once installed) and **Sign out**.
  The top bar is hidden for the scorer while a period is being played.
- **Live score cards**: a game being played shows at the top of the home screen and the Games
  tab, with the score, period and clock; one tap opens its Live tab. The score is counted from
  the game's events, so these listen only to live games (a `status == 'live'` query per season
  on the home screen) and only while the card is on screen. No rule changes.
- **Next up** card at the top of the Games tab: the next game that hasn't started, with
  Today / Tomorrow / the day, time, venue (a Google Maps link for directions) and court. Coach/
  Managers also see whether the lineup is ready, with Finalise lineup. The venue on a game's
  page is a directions link too.
- **Follow a player**: tap any player's name in a stats table to open their page (games,
  minutes, average, goals, accuracy, positions played, game by game). **Follow** saves them on
  that phone for that season (`localStorage` `nettyPalFollow`); they're starred and highlighted
  in lineups and stats, with a card at the top of Season Stats. Awards are never shown there.
- **Season Stats players table**: Player, Mins and Goals to start with, **All columns** for
  Games, Avg mins and Accuracy; tap a heading to sort. The court-time-by-position grid is for
  Coach/Managers and Scorers (everyone can see one player's positions on their page).
- **Scoring screen**: End period is at the top right (two taps), away from Goal/Miss; Pause is
  bigger. Game stats shows the other team's shooting by position only when there's more than one.
- **Results** in the games list have a W / L / D badge and colour (a loss was green before).
- **Support Crew**: Remove is behind a "⋯" button next to each person.
- **Games tab**: one **+ Add games** button (add one game, or import the season's draw). The
  Coach/Manager guide and Try a practice game are also in Help while a team is open.
- Buttons, tabs and links are at least 44px tall, small text is bigger and grey text darker.
- **Team tabs**: Games · Stats · Team. Team has a switch for Players (Coach/Managers and
  Scorers), Support Crew (everyone) and Settings (Coach/Managers), and reopens the part used
  last. Parents only see Support Crew there, so no switch.
- Tab rows that are wider than the screen (a game's tabs for Coach/Managers) show a thin scroll
  bar and fade at the edge that has more tabs.
- The install card is no longer on the home screen (Install is in the account menu); it still
  shows on the sign-in screen. On the Games tab, "Want to help score?" and the practice-game
  card sit below the games, not above them. The scorer's "? Help" on the scoreboard shows
  during breaks too.

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
  site's address (now `nettypal.au` and `www.nettypal.au`) as an authorised domain.
- Paste the Firebase web config (not a secret; it identifies the project) into the app.

---

## 10. Working conventions

- Owner is not a professional developer: explain decisions in plain English.
- Work on a branch and open a pull request; the owner merges.
- No em dashes in copy or comments.
- Never commit passwords, tokens or private keys.
