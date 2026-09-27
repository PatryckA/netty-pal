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
| **Admin** | One account per team. Whoever creates the team starts as admin. Transferable to another member. | Everything: edit team, seasons, roster, fixtures, assign roles, remove members, regenerate the season code, transfer admin. |
| **Coach** | Assigned by admin. | Manage lineups and positions (before and during games), view all summaries. |
| **Scorer** | Assigned by admin. | Score games (one scorer per game; can differ game to game), make lineup changes during play if needed. |
| **Viewer** | Anyone who joined with the season code and has no role. | Follow live scores, view game and season summaries. Read-only. |

- One person can be both Coach and Scorer.
- Anyone signed in with Google can create a new team (and becomes its admin).
- A user can belong to several teams (e.g. siblings).

### Season code
- One code per team season. **Never expires.**
- Joining: sign in with Google, enter code, become a Viewer of that season.
- Admin can remove a member and regenerate the code if it gets passed around.

---

## 3. Teams, seasons, roster

- **Team**: name, admin.
- **Season**: name (e.g. "Winter 2027"), competition name, default format
  (**quarters** or **halves**), default period length (**7 minutes**), season code.
- **Roster** per season: player names. Fill-in players are simply added to the
  roster and count in season totals like everyone else. Players can be marked
  inactive rather than deleted, so past stats stay intact.

---

## 4. Fixtures

- Admin can add fixtures for the season: round, date, time, venue/court, opponent.
- **Import**: upload a PDF or paste text from the competition draw. The app pulls
  out rows that look like fixtures and shows an **editable table to review** before
  saving. Nothing is saved without review.
- Import is built last and needs a real fixture file from the competition to design
  against (not yet supplied).
- Manual "add fixture" and "new game on the day" are always available.
- Each game inherits the season's format and period length, and can override both.

---

## 5. Game day

### Setup
1. Pick the fixture (or create a new game).
2. Confirm format (quarters/halves) and period length (default 7 min).
3. Scorer claims the game (only one scorer per game).
4. Set the starting lineup on the court board: GS, GA, WA, C, WD, GD, GK.
5. Choose who takes the first centre pass (us or opposition).

### Clock
- Start, pause, resume, end period. Pause is for injuries and umpire stoppages.
- **Court time only accrues while the clock is running.**
- Court time is calculated from timestamped events (on/off, clock start/stop), not
  from a ticking counter, so a locked phone or backgrounded app doesn't lose time.

### Lineup changes
- Allowed **at any time, including mid-quarter**.
- Coach does this normally; Scorer can also do it.
- Swap a bench player on, or swap two players' positions.

### Scoring
- Two taps: **position bib, then stat**. The stat is credited to whoever is in that
  position at that moment.
- Our team stats (starting set):
  - Goal
  - Miss (missed shot)
  - Intercept
  - Held ball (penalty against)
  - Obstruction (penalty against)
  - Contact (penalty against)
- Opposition: **goal** and **miss** (team-level, no players).
- **Undo** for every action.
- More stats can be added later (rebounds, deflections, turnovers were discussed).

### Centre pass
- Automatic: alternates after every goal (either team), and each period starts
  with the team that did not take the first centre of the previous period.
- Manual override button if it gets out of sync.
- Always visible on the scoring screen.

### Live view
- Everyone in the season (including Viewers) can follow the live score, period and
  clock.

---

## 6. Summaries (Coaches and Viewers)

### Per game
- Final score, by period.
- Court time per player, split by position.
- Per player: goals, misses, shooting % (goals / (goals + misses)), intercepts,
  penalties against (held ball + obstruction + contact, with breakdown).
- Opposition goals, misses and shooting %.

### Season to date
- The same figures totalled across all completed games.
- Court time totals per player and per position (to help keep court time fair).

---

## 7. Draft data model (Firestore)

```
users/{uid}                          name, email
joinCodes/{code}                     teamId, seasonId
teams/{teamId}                       name, adminUid
  seasons/{seasonId}                 name, competition, format, periodMinutes (7), joinCode
    members/{uid}                    name, roles: ["coach","scorer"] (empty = viewer)
    players/{playerId}               name, active
    games/{gameId}                   round, date, time, venue, opponent,
                                     format, periodMinutes, scorerUid,
                                     status: scheduled | live | final,
                                     firstCentre: us | them, score summary
      events/{eventId}               type, period, clockMs, playerId, position,
                                     createdBy, createdAt, undone
```

Event types: `clockStart`, `clockStop`, `periodEnd`, `lineup` (player on/off/position),
`goal`, `miss`, `intercept`, `heldBall`, `obstruction`, `contact`, `oppGoal`,
`oppMiss`, `centreOverride`.

All summaries are calculated from the event list, so undo just marks an event as
undone and everything recalculates.

---

## 8. Build order

1. **Foundations**: Google sign-in, create team, seasons, roster, season code,
   joining, roles, remove member, regenerate code, admin transfer. Security rules.
2. **Game day**: fixtures (manual), clock, court board, lineup changes, scoring pad,
   centre pass, undo, live view.
3. **Summaries**: per game and season to date.
4. **Fixture import**: PDF or pasted text, review table. Needs a sample fixture file.

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
