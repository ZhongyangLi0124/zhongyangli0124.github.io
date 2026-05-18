# PhET Simulations contribution to WebHarbor — Handoff

> **Read this if you are a fresh Claude Code (or human) session working
> inside a clone of <https://github.com/ZhongyangLi0124/WebHarbor>.**
> All prior design work is done; this is the operational runbook.

## TL;DR

You will:
1. Apply a single git patch (~117 KB) that adds `sites/phet_simulations/`
   and updates 3 existing files.
2. Extract a 17 KB tarball that places the 143 KB seed database at
   `sites/phet_simulations/instance_seed/phet_simulations.db`.
3. Smoke-test the container.
4. Push the branch to the fork.
5. Upload the asset tarball to Hugging Face (the dataset side of WebHarbor).
6. Bump `.assets-revision` once the HF PR merges, then open the GitHub PR.

The patch and tarball are hosted on the contributor's personal site repo —
URLs given below. Both have been validated end-to-end against the real
`aiming-lab/WebHarbor` codebase (Dockerfile-pinned Flask 3.1.0,
SQLAlchemy 2.0.36, etc.).

---

## Context

**Project:** [aiming-lab/WebHarbor](https://github.com/aiming-lab/WebHarbor)
is a benchmark of Flask + SQLite mirror sites for evaluating web agents.
Each `sites/<name>/` is a self-contained Flask app that mimics a real
website closely enough that an agent that works on the real site also
works on the mirror. The container launches all sites on
`40000 + index`, with a control plane on `:8101` that handles
`/reset/<site>` for deterministic re-seeding.

**Contribution:** PhET Interactive Simulations
(<https://phet.colorado.edu/>) — University of Colorado Boulder's free
interactive math and science simulations. Selected from the open-tasks
list because it's a stable, CC-BY-licensed academic site with no
commercial anti-bot defenses, clean taxonomy (subjects × grades ×
languages), and modest catalog size.

**Port slot:** **40015** (the 16th site — appended after `espn`).

**Seeded catalog:**

| Filter             | Records | Notes                                  |
| ------------------ | ------: | -------------------------------------- |
| Simulations        |      98 | Spans all 5 subjects + 4 grade bands   |
| Subjects           |       5 | physics / chemistry / math / biology / earth-science (each ≥20 sims) |
| Grade levels       |       4 | elementary / middle / high / university (each ≥20 sims) |
| Languages          |      28 | Includes RTL (Arabic, Hebrew, Persian) |
| Teacher activities |      14 | Linked to specific simulations         |
| Benchmark users    |       4 | One has pre-saved sims                 |
| Benchmark tasks    |      43 | `tasks.jsonl`                          |

---

## Prerequisites on your machine

| Tool            | Version           | Install                            |
| --------------- | ----------------- | ---------------------------------- |
| `git`           | any modern        | system                             |
| `docker`        | any with daemon   | system                             |
| `python3`       | 3.10+             | system                             |
| `hf` CLI        | **0.x preferred** | `pip install -U "huggingface_hub<1"` (see "Known issues") |
| HF auth         | logged in         | `hf auth login` (paste token)      |

The fork is at `ZhongyangLi0124/WebHarbor`. SSH host-key warnings on
first clone may indicate stale `~/.ssh/known_hosts` entries — see
"Known issues" if you hit that.

---

## Step 1 — Clone the fork and create a branch

```bash
git clone https://github.com/ZhongyangLi0124/WebHarbor.git
cd WebHarbor
git checkout -b add-phet-simulations
```

**HTTPS is recommended** over SSH for the first clone — it avoids the
known_hosts churn from GitHub's 2023 RSA key rotation.

---

## Step 2 — Fetch the patch + asset tarball

The artifacts live on the contributor's personal site repo, branch
`claude/evaluate-docker-setup-KFBgT`. Direct raw URLs:

```bash
curl -L -o /tmp/phet.patch \
  "https://raw.githubusercontent.com/ZhongyangLi0124/zhongyangli0124.github.io/claude/evaluate-docker-setup-KFBgT/webharbor-phet/dist/0001-feat-phet_simulations-add-new-site.patch"

curl -L -o /tmp/phet_simulations.tar.gz \
  "https://raw.githubusercontent.com/ZhongyangLi0124/zhongyangli0124.github.io/claude/evaluate-docker-setup-KFBgT/webharbor-phet/dist/phet_simulations.tar.gz"

# Sanity check sizes
ls -la /tmp/phet.patch /tmp/phet_simulations.tar.gz
# Expected: ~117 KB patch, ~17 KB tarball
```

---

## Step 3 — Apply the patch

```bash
git am /tmp/phet.patch
```

Expected: a single new commit on `add-phet-simulations` titled
`feat(phet_simulations): add new site`, touching **26 files / +2575 / -6**.

If `git am` fails with "patch does not apply", run `git am --abort`
and check that you are on a clean working tree of `main` of the fork
(no other local edits). The patch is rebased onto the same `main` tip
that the fork was synced from on **2026-05-18**; if upstream has moved
significantly since then, do `git rebase main` after applying.

---

## Step 4 — Drop the seed DB into place

The seed DB is the runtime source of truth. Per the project convention,
it's gitignored (`sites/*/instance_seed/`) and ships via the Hugging
Face dataset, not GitHub. For local testing you extract it manually
from the bundled tarball:

```bash
mkdir -p sites/phet_simulations/instance_seed
tar -xzf /tmp/phet_simulations.tar.gz -C sites \
    phet_simulations/instance_seed/phet_simulations.db

# Verify
ls -la sites/phet_simulations/instance_seed/
md5sum sites/phet_simulations/instance_seed/phet_simulations.db
# Expected: e094a2ee23369d3b60232f49f4ac691c  ...phet_simulations.db
```

If the md5 doesn't match, the tarball is corrupted or truncated — refetch.

---

## Step 5 — Fetch the other 15 sites' assets

The Dockerfile copies the whole `sites/` tree into the image, so any
site without an `instance_seed/<name>.db` will fail to boot. You need
the other 15 sites' seed DBs from the HF dataset before `build.sh`
will succeed.

### Recommended path (works around the huggingface_hub 1.x breakage)

```bash
./scripts/fetch_assets.sh
./scripts/check_assets.sh
# Expected: "all sites have instance_seed/ (... warnings ...)"
```

### If `fetch_assets.sh` reports "0 site(s) extracted"

Symptom seen on `huggingface_hub` 1.15.0: `hf download` succeeds but
places files in an unexpected subdirectory of `sites/.cache/tarballs/`,
so the bash glob `$CACHE_DIR/*.tar.gz` matches nothing.

**Diagnose first:**

```bash
ls sites/.cache/tarballs/
find sites/.cache/tarballs -name '*.tar.gz' | head -5
hf --version
```

**Fix A — downgrade `hf` CLI to 0.x (the version the script was written against):**

```bash
pip install -U "huggingface_hub<1"
rm -rf sites/.cache/tarballs
./scripts/fetch_assets.sh
```

**Fix B — keep 1.x and do the extraction manually:**

```bash
# Make sure the tarballs are actually on disk
hf download ChilleD/WebHarbor --repo-type dataset \
    --revision main --include "*.tar.gz" \
    --local-dir sites/.cache/tarballs

# Extract every tarball you find, no matter how nested
find sites/.cache/tarballs -name '*.tar.gz' -exec \
    tar -xzf {} -C sites/ \;

./scripts/check_assets.sh
```

**Fix C — only smoke-test PhET, skip the other 15 sites locally**

If you cannot get HF auth working at all but you want to verify PhET in
isolation, you can temporarily edit `websyn_start.sh`, `control_server.py`,
and `Dockerfile` to list only `phet_simulations`. **Do NOT commit
those edits to `add-phet-simulations`** — they're for local smoke
testing only. Use `git stash` or a throwaway branch.

---

## Step 6 — Build and smoke-test the container

```bash
./scripts/build.sh
docker run -d --rm --name wh-test \
    -p 8101:8101 -p 40000-40015:40000-40015 webharbor:dev
sleep 8
```

### Probe PhET specifically

```bash
curl -fsS http://127.0.0.1:40015/_health
# Expected: {"ok": true, "site": "phet_simulations"}

curl -fsS -o /dev/null -w "GET /                          %{http_code}\n" \
    http://127.0.0.1:40015/
curl -fsS -o /dev/null -w "GET /simulations               %{http_code}\n" \
    http://127.0.0.1:40015/simulations
curl -fsS -o /dev/null -w "GET /simulation/build-an-atom  %{http_code}\n" \
    http://127.0.0.1:40015/simulation/build-an-atom
curl -fsS -o /dev/null -w "GET /translations/zh-cn        %{http_code}\n" \
    http://127.0.0.1:40015/translations/zh-cn
curl -fsS -o /dev/null -w "GET /teachers                  %{http_code}\n" \
    http://127.0.0.1:40015/teachers
curl -fsS -o /dev/null -w "GET /search?q=gravity          %{http_code}\n" \
    http://127.0.0.1:40015/search?q=gravity
# Expected: all 200
```

### Verify control plane sees PhET

```bash
curl -fsS http://127.0.0.1:8101/health | python3 -m json.tool | \
    grep -A 3 phet_simulations
# Expected: phet_simulations is alive on port 40015
```

---

## Step 7 — Verify `/reset/phet_simulations` is byte-idempotent

This is the canonical hard requirement from `CONTRIBUTING.md`:

```bash
curl -fsS -X POST http://127.0.0.1:8101/reset/phet_simulations
# Wait briefly for respawn to finish:
sleep 3

docker exec wh-test md5sum \
    /opt/WebSyn/phet_simulations/instance/phet_simulations.db \
    /opt/WebSyn/phet_simulations/instance_seed/phet_simulations.db
# Expected: both md5s identical -> e094a2ee23369d3b60232f49f4ac691c
```

If the md5s differ, something is mutating the DB during seed phase. The
sandbox already verified this passes with the shipped seed; a mismatch
would mean either a botched tarball extract or local divergence — go
back to Step 4.

### Also verify across a full container restart

```bash
docker restart wh-test && sleep 8
docker exec wh-test md5sum \
    /opt/WebSyn/phet_simulations/instance/phet_simulations.db \
    /opt/WebSyn/phet_simulations/instance_seed/phet_simulations.db
# Still must match.
```

Tear down:

```bash
docker stop wh-test
```

---

## Step 8 — Upload the asset tarball to Hugging Face

The GitHub repo and the HF dataset PR together form one contribution.
The HF side goes first because the GitHub PR references the HF merge
commit SHA in `.assets-revision`.

```bash
# Stage the tarball in a separate dir (mirrors the dataset's flat layout)
mkdir -p ../wh-static-pr
cp /tmp/phet_simulations.tar.gz ../wh-static-pr/

# Upload to YOUR HF fork (you'll need to fork
# https://huggingface.co/datasets/ChilleD/WebHarbor first via the HF UI)
hf upload <your-hf-username>/WebHarbor \
    ../wh-static-pr/phet_simulations.tar.gz \
    phet_simulations.tar.gz \
    --repo-type dataset
```

Then open a PR on `huggingface.co/datasets/ChilleD/WebHarbor` from your
fork. Once it's merged, copy the **merge commit SHA** for Step 9.

---

## Step 9 — Bump `.assets-revision` and push

```bash
# In the WebHarbor clone
sed -i.bak "s/^revision:.*/revision: <hf-merge-sha>/" .assets-revision
rm .assets-revision.bak

git add .assets-revision
git commit -m "chore(phet_simulations): bump assets to <hf-merge-sha>"

# Push the branch
git push -u origin add-phet-simulations
```

Open the GitHub PR from `add-phet-simulations` against
`aiming-lab/WebHarbor:main`. The PR description should include:

- Real site mirrored: <https://phet.colorado.edu/>
- 98 simulations, 5 subjects, 4 grade bands, 28 languages
- Link to the merged HF PR
- Paste output of `curl -X POST http://127.0.0.1:8101/reset/phet_simulations`
  showing `"ready": true`
- Paste both md5sums from Step 7

---

## Reference — what the patch changes

```
 Dockerfile                                      |    2 +-
 control_server.py                               |    2 +-
 websyn_start.sh                                 |    8 +-
 sites/phet_simulations/_health.py               |   11 +
 sites/phet_simulations/app.py                   | 1263 ++++++++++++++++++++
 sites/phet_simulations/static/css/style.css     |  433 +++++++
 sites/phet_simulations/static/js/main.js        |   59 +
 sites/phet_simulations/tasks.jsonl              |   43 +
 sites/phet_simulations/templates/*.html         |  ~800 lines across 16 files
 26 files changed, 2575 insertions(+), 6 deletions(-)
```

### The 3-file integration patches (already in the .patch — listed here for review)

```diff
--- a/Dockerfile
-EXPOSE 8101 40000-40014
+EXPOSE 8101 40000-40015

--- a/control_server.py
     'huggingface', 'wolfram_alpha', 'cambridge_dictionary',
-    'coursera', 'espn',
+    'coursera', 'espn', 'phet_simulations',
 ]

--- a/websyn_start.sh
-SITES=(allrecipes amazon apple arxiv bbc_news booking github
-       google_flights google_map google_search huggingface wolfram_alpha
-       cambridge_dictionary coursera espn)
+SITES=(allrecipes amazon apple arxiv bbc_news booking github
+       google_flights google_map google_search huggingface wolfram_alpha
+       cambridge_dictionary coursera espn phet_simulations)
@@
-echo "[WebSyn] Starting 15 sites on ports ${BASE_PORT}-$((BASE_PORT + 14))..."
+echo "[WebSyn] Starting 16 sites on ports ${BASE_PORT}-$((BASE_PORT + 15))..."
@@
-    echo "  [${elapsed}/${max_wait}s] ${ready}/15 sites ready"
-    if [ $ready -eq 15 ]; then
+    echo "  [${elapsed}/${max_wait}s] ${ready}/16 sites ready"
+    if [ $ready -eq 16 ]; then
```

---

## Reference — seeded credentials (for testing the auth flow)

The seed creates 4 deterministic users. The teacher account has 4
pre-saved simulations so `/account` is non-empty.

| Email                 | Password              | Role       | Pre-saved sims |
| --------------------- | --------------------- | ---------- | -------------: |
| `teacher@phet.test`   | `phet-teacher-pass`   | teacher    | 4              |
| `student@phet.test`   | `phet-student-pass`   | student    | 0              |
| `research@phet.test`  | `phet-research-pass`  | researcher | 0              |
| `demo@phet.test`      | `phet-demo-pass`      | teacher    | 0              |

---

## Reference — site architecture (quick summary)

**Stack:** Flask 3.1.0 + Flask-SQLAlchemy 2.0.36 + Flask-Login 0.6.3 +
Flask-WTF 1.2.2 + Flask-Bcrypt 1.0.1. Versions match the project's
root `Dockerfile` — no per-site `requirements.txt` is shipped (project
convention).

**Models (7):** `User`, `Subject`, `GradeLevel`, `Language`,
`Simulation`, `Activity`, `SavedSimulation`. All defined inline in
`app.py`.

**Routes (17 public + 2 JSON + /_health):**

| Path                                          | Purpose                       |
| --------------------------------------------- | ----------------------------- |
| `/`                                           | Featured / new / popular sims |
| `/simulations` (`?subject=&grade=&language=&sort=&page=`) | Filtered catalog  |
| `/simulations/category/<subject-slug>`        | Per-subject landing           |
| `/simulation/<slug>`                          | Detail page                   |
| `/search?q=`                                  | Title + topic search          |
| `/translations`                               | Language picker               |
| `/translations/<code>`                        | Sims in a language            |
| `/teachers`                                   | Teacher resources landing     |
| `/teachers/activities` (`?grade=`)            | Activities index              |
| `/teachers/activity/<id>`                     | Activity detail               |
| `/about`, `/accessibility`                    | Static info                   |
| `/register`, `/login`, `/logout`, `/account`  | Auth + saved sims             |
| `/api/save-sim`, `/api/unsave-sim`            | CSRF-protected JSON           |
| `/_health`                                    | `{"ok": true, "site": ...}`   |

**Idempotency:** every `seed_*` function early-returns on a populated
table — *before* any `db.session.add()` or `commit()`. This avoids the
"empty-transaction bumps SQLite metadata" trap noted in
`CONTRIBUTING.md`.

---

## Known issues you may hit

### "REMOTE HOST IDENTIFICATION HAS CHANGED" on `git clone`

GitHub rotated its RSA host key in **March 2023**. Stale local
`~/.ssh/known_hosts` entries cause the warning. Either:

```bash
ssh-keygen -R github.com
# Re-clone; you'll be prompted to accept the new key.
# Verify it matches GitHub's published RSA fingerprint:
#   SHA256:uNiVztksCsDhcc0u9e8BujQXVUpKZIDTMczCvj3tD2s
```

Or just use HTTPS (recommended above) and avoid SSH entirely for this
operation.

### `fetch_assets.sh` reports "0 site(s) extracted"

`huggingface_hub` ≥ 1.0 changed how `hf download --local-dir` lays out
files. See Step 5 Fix A / Fix B above.

### `git am` fails with "patch does not apply"

The patch was generated against `aiming-lab/WebHarbor@main` as of
2026-05-18 (commit `af0765d`, "Merge pull request #4 from
hqhq1025/codex/fix-booking-homepage-images"). If upstream has moved:

```bash
git am --abort
git fetch origin main
git rebase origin/main      # if no conflicts you're fine
# Otherwise: apply the patch, then resolve conflicts in the standard way.
```

### Commit signing fails when you push

If your local `git config commit.gpgsign true` blocks the commit you'd
need to make to bump `.assets-revision` (Step 9), the simplest workaround:

```bash
GIT_AUTHOR_NAME="$(git config user.name)" \
GIT_AUTHOR_EMAIL="$(git config user.email)" \
git -c commit.gpgsign=false commit --no-gpg-sign \
    -m "chore(phet_simulations): bump assets to <hf-merge-sha>"
```

The cherry-on-top fix is to sign the commit; GitHub will show
"unverified" otherwise, which is OK for a contribution PR but cleaner
to fix later via `git commit --amend -S`.

### Container fails to start with "missing instance_seed"

You skipped Step 4 or Step 5 — go back. The Dockerfile blindly copies
the whole `sites/` tree; missing seed DBs aren't detected until a site
process tries to open SQLite at runtime.

---

## What is intentionally NOT in scope here

- **Real images / screenshots.** The simulation cards use CSS-only
  gradient placeholders. PhET's real screenshots are CC-BY-licensed and
  could be added later if reviewers ask; for the initial PR they were
  omitted to keep the asset bundle under 20 KB and the contribution
  reviewable.
- **Actual simulation playback.** The detail-page "Play" button shows
  a modal explaining the mirror is a static snapshot. Agents that
  search / filter / read sim metadata will all work; agents that
  expect to physically *run* a PhET HTML5 sim will not.
- **Translations of the UI itself.** Only the *catalog of translated
  sims* is mirrored; the surrounding navigation stays in English. This
  matches the spec — tasks are about discovering which sims are
  available in which languages, not interacting with them in those
  languages.

---

## If you're a fresh Claude Code session reading this

You have everything you need. Don't re-design anything — the patch
encodes the design decisions. Your job is purely operational: fetch,
apply, verify, push, upload, PR. If you hit something unexpected,
check "Known issues" first; if it's not listed, escalate to the user
before improvising.

The contributor's original development branch
(`claude/evaluate-docker-setup-KFBgT` on
`ZhongyangLi0124/zhongyangli0124.github.io`) contains the source of
truth for everything in `webharbor-phet/`, including this handoff.
