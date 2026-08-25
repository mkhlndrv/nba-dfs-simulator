# NBA Daily Fantasy Simulator

A daily fantasy basketball platform built with Laravel 12 and Blade. Users draft salary-cap
lineups from real NBA player statistics and enter contests played for virtual points. Game
results are simulated from each player's season averages, so the whole loop runs locally with
no external API and no real money.

## Overview

For a user:

1. Register and receive 10,000 virtual points.
2. Pick a contest: 50/50, GPP or head-to-head, each with an entry fee and a lock time.
3. Build a lineup of exactly 8 players, one per slot (PG, SG, SF, PF, C, G, F, UTIL), under a
   $50,000 salary cap. Only players whose teams play on the contest date are offered, and the
   G and F slots check position eligibility.
4. Once the games are simulated, lineups are scored and ranked, prizes are paid into the
   points balance, and every point movement lands in a transaction history.

For an admin: create contests, cancel them (cancellation refunds every entry fee), simulate or
reset games one at a time or a whole date, curate team rosters, and re-import player or
schedule data, all from an admin panel behind admin middleware.

## Scoring and salaries

Fantasy points use a standard DFS formula: points x 1.0, rebounds x 1.25, assists x 1.5,
steals x 2.0, blocks x 2.0, turnovers x -0.5, plus 1.5 for a double-double and 3.0 for a
triple-double.

Salaries are generated from the same weights at import time: a player's per-game fantasy value
times 200, scaled by position (1.00 for point guards down to 0.88 for centers) and by minutes
played, clamped to the 3,000 to 12,000 range and rounded to the nearest 100. So the salary cap
is priced in the same currency the contests are scored in.

## Contests and payouts

The prize pool is 90 percent of the entry fee times the maximum entries; the remaining 10
percent is the rake.

- **50/50**: the top half of the field splits the pool evenly.
- **GPP**: top-heavy, 20 percent to 1st, 10 percent to 2nd, 7 percent to 3rd, then shrinking
  shares down to 30th place.
- **H2H**: the winner takes the whole pool.

The pool is fixed when the contest is created, from the maximum entries rather than the actual
count, so an under-filled contest pays out more than it collected, the way a guaranteed prize
pool does.

## Game simulation

There is no live NBA feed. A game takes the ten highest-minute active players from each team
and draws a per-player performance multiplier from a discrete distribution centred on an
average night: for players scoring 20 or more a game, 10 percent bad games (0.6 to 0.8x), 40
percent average, 10 percent great (1.3 to 1.6x); lower-volume scorers draw from a harsher
distribution that underperforms more often. The multiplier is applied to the player's season
averages with extra noise, each stat is clamped to a plausible range, and the box score is
stored per game. A game simulates once, and every contest on that date reads the same stored
results.

## Data

Two CSVs are committed so the app seeds without any external service:

- `nba_player_stats_update.csv`: per-game averages for 533 players from the 2024-25 season
  (points, rebounds, assists, steals, blocks, turnovers, minutes, position).
- `nba_full_calendar.csv`: 267 scheduled games from November 17 to December 31, 2025, with
  tip-off times and arenas.

`import:players` computes each player's salary as it loads the stats, `import:games` loads the
schedule, and `set:rosters` marks who is active.

## Usage

Requirements: PHP 8.3+, Composer, Node.js. SQLite is the default database.

```bash
composer install && npm install
cp .env.example .env
php artisan key:generate
touch database/database.sqlite
php artisan migrate
php artisan db:seed --class=AdminUserSeeder
php artisan import:players nba_player_stats_update.csv
php artisan import:games nba_full_calendar.csv
php artisan set:rosters
npm run build
php artisan serve
```

The seeded admin login is `admin@nba-fantasy-bet.test` / `password`, for local use only. New
users register at `/register`; the admin panel is at `/admin/dashboard`. For development,
`composer run dev` starts the server, queue worker, log tail and Vite together. `SETUP.md`
has the longer version, including troubleshooting, and `DATABASE_RELATIONSHIPS.md` maps the
nine tables.

## Tests

```bash
php artisan test
```

Pest feature tests cover authentication, contest entry (balance, lock time and per-user entry
limits), lineup validation (roster shape and the salary cap), leaderboards and the admin
actions; unit tests cover the contest, lineup and game models and the simulator. Tests run
against an in-memory SQLite database.

## Layout

```
app/Services/GameSimulator.php     stat simulation, fantasy scoring, contest resolution
app/Services/SalaryCalculator.php  the salary formula
app/Models/Contest.php             payout structures, prize distribution, cancellation
app/Console/Commands/              CSV imports and roster management
routes/web.php                     the user and admin route map
```

## Limitations

- Results are simulated from season averages, not real box scores, and the multiplier
  distribution is hand-tuned rather than fitted to historical variance.
- Player stats are a single snapshot of the 2024-25 season: no injury news, no matchup
  adjustment, and salaries never move.
- Contests resolve only when an admin triggers simulation; there is no scheduler.
- Points are virtual throughout. Nothing here handles money.