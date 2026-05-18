# PhET Interactive Simulations — WebHarbor mirror

A Flask + SQLite mirror of <https://phet.colorado.edu/> for the WebHarbor
agent benchmark. Browse the simulation catalog, filter by subject /
grade / language, drill into per-simulation pages, browse 28 language
translations, read teacher-submitted activities, and exercise the
account-gated save flow.

## What the snapshot contains

| Filter            | Records | Notes                                        |
| ----------------- | ------: | -------------------------------------------- |
| Simulations       |      98 | Spans 5 subjects, 4 grade bands, 28 languages |
| Subjects          |       5 | physics, chemistry, math, biology, earth-science (each ≥20 sims) |
| Grade levels      |       4 | elementary, middle, high, university (each ≥20 sims) |
| Languages         |      28 | Includes RTL (Arabic, Hebrew, Persian)       |
| Teacher activities|      14 | Linked to specific simulations               |
| Benchmark users   |       4 | Pre-saved sims on teacher account            |

## Routes

| Route                                         | Purpose                          |
| --------------------------------------------- | -------------------------------- |
| `/`                                           | Home: featured / new / popular   |
| `/simulations`                                | Catalog with filters + pagination |
| `/simulations/category/<subject>`             | Subject landing page             |
| `/simulation/<slug>`                          | Per-sim detail page              |
| `/search?q=<term>`                            | Full-text-ish search             |
| `/translations`                               | Language picker                  |
| `/translations/<code>`                        | Sims in a language               |
| `/teachers`                                   | For-teachers landing             |
| `/teachers/activities`                        | Activity index (grade filter)    |
| `/teachers/activity/<id>`                     | Activity detail                  |
| `/about`, `/accessibility`                    | Static info pages                |
| `/register`, `/login`, `/logout`, `/account`  | Auth + saved sims                |
| `/api/save-sim`, `/api/unsave-sim`            | AJAX (JSON, CSRF-protected)      |
| `/_health`                                    | `{"ok": true, "site": ...}`      |

## Benchmark credentials

| Email                     | Password              | Role       | Pre-saved sims |
| ------------------------- | --------------------- | ---------- | -------------: |
| `teacher@phet.test`       | `phet-teacher-pass`   | teacher    | 4              |
| `student@phet.test`       | `phet-student-pass`   | student    | 0              |
| `research@phet.test`      | `phet-research-pass`  | researcher | 0              |
| `demo@phet.test`          | `phet-demo-pass`      | teacher    | 0              |

## Local development

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
PORT=40015 .venv/bin/python app.py
# → http://127.0.0.1:40015/
```

The first boot creates `instance/phet_simulations.db` and runs all seed
functions. Subsequent boots are no-ops — seed functions early-return when
their target table is non-empty.

## Rebuilding the seed DB

```bash
rm -rf instance/phet_simulations.db
.venv/bin/python -c "from app import app"   # triggers seed
cp instance/phet_simulations.db instance_seed/phet_simulations.db
md5sum instance{,_seed}/phet_simulations.db  # both should match
```

## Idempotency contract

After `/reset/phet_simulations` the WebSyn control plane copies
`instance_seed/` over `instance/`. Re-importing `app` runs `seed_all()`
but every `seed_*` function checks `.query.count() > 0` and returns
early, so the database file is byte-identical to the seed after restart.
Verified with `md5sum` (see test script in repo root).
