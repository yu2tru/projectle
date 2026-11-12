# projectle

A daily 3D estimation game. Each round shows an object and a target ring. Drag back to add launchers (fans, springs, rockets or leaf blowers), let go, and try to land on the ring. 5 rounds a day, the same for everyone.

Everything is in **`index.html`**. Open it in a browser or host it as a static file. Three.js (0.168.0) and, if enabled, supabase-js (2.45.4) load from pinned jsDelivr URLs.

## Where to tune things

| What | Where in `index.html` |
| --- | --- |
| Launch date (puzzle No. 1) | `LAUNCH_DATE` in the CORE script (section 1) |
| Launcher strength | `LAUNCHERS` (section 2): `strength`, plus `falloff` for fans and blowers, `stroke`/`angle` for springs, `burn`/`angle` for rockets |
| Objects (mass, drag, friction, bounce…) | `OBJECTS` (section 3). Add an entry here and a matching builder in `MODELS` |
| Ice friction for leaf-blower rounds | `ICE_MULT` (section 4) |
| Answer range per round (difficulty curve) | `ROUND_RANGES` (section 5) |
| Scoring curve | `score()` in CORE |
| Leaderboard keys | `SUPABASE_URL`, `SUPABASE_ANON_KEY` at the top of the module script |
| Drag distance per launcher | `PX_PER_LAUNCHER` |
| Backdrops per scene | `THEMES` (and `buildProps`) |

Any change to `LAUNCHERS`, `OBJECTS`, `ICE_MULT` or `ROUND_RANGES` changes every day's answers. If you change them after launch, players who already played that day will see a different puzzle.

## How rounds are made

- The seed is `LAUNCH_DATE + '|' + local date`. A seeded generator picks 5 different objects, sorted by `difficulty`, and gives them launcher types so that each day uses at least 3 types.
- For each round, the game simulates every count from 1 to 20 with a fixed 1/120 s timestep. It picks an answer from that round's `ROUND_RANGES` entry, but only a count whose resting spot is at least 1.15 object-lengths from every other count's resting spot. The target goes exactly where that count comes to rest.
- The physics is pure IEEE arithmetic with its own sin/cos (no `Math.sin`, `Math.exp` or `Math.random`), so the same count gives exactly the same result in every browser.

## Supabase table (`projectle_scores`)

```sql
create table if not exists public.projectle_scores (
  id         bigint generated always as identity primary key,
  created_at timestamptz not null default now(),
  player_id  text        not null check (char_length(player_id) between 8 and 64),
  puzzle_no  integer     not null check (puzzle_no > 0),
  day        date        not null,
  total      smallint    not null check (total between 0 and 500),
  rounds     smallint[]  not null check (array_length(rounds, 1) = 5 and 0 <= all(rounds) and 100 >= all(rounds)),
  counts     smallint[]           check (counts is null or (array_length(counts, 1) = 5 and 1 <= all(counts) and 20 >= all(counts))),
  unique (player_id, puzzle_no)
);

create index if not exists projectle_scores_puzzle_total on public.projectle_scores (puzzle_no, total);

alter table public.projectle_scores enable row level security;

create policy "projectle: anyone can read"
  on public.projectle_scores for select
  to anon, authenticated
  using (true);

create policy "projectle: anyone can submit"
  on public.projectle_scores for insert
  to anon, authenticated
  with check (
    total between 0 and 500
    and day between current_date - 1 and current_date + 1
  );
```

The client inserts with `on conflict (player_id, puzzle_no) do nothing`, so it needs only the insert policy (there is no update or delete policy). If the keys are empty or a request fails, the leaderboard line stays hidden and the rest of the game still works.
