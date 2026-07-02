# PR #29 review fixes — apply & land runbook

One patch fixes all five review points from MufanQiu. Apply it on top of
your existing `add-phet-simulations` branch (head `b790166`), re-upload
the regenerated seed tarball to Hugging Face, repin `.assets-revision`,
push, and reply to the review.

## Files

- `0001-fix-phet_simulations-address-PR-29-review-feedback.patch`
  — git-am patch, 4 files changed (+93/−34)
- `phet_simulations.tar.gz` — carries the regenerated seed DB
  (new md5 `48ca438b8ac6dab37a6503a4e5574503`). ⚠️ Seed-only bundle:
  extract the .db from it, but repack the HF tarball from your local
  tree so your site images are preserved (see step 2).

## What the patch fixes (mapping to the review)

| Review point | Fix |
| --- | --- |
| BLOCKER: homepage renders zero sims | `index.html` now renders featured / new_sims / most_played as three sim-card rows (20 cards) |
| MAJOR: 'New' filter decorative | `simulations()` reads `release=new` (is_new) and `release=updated` (released ≥ 2024-01-01); checkbox state, chips, sort, and pagination all carry the param |
| MAJOR: DB mutates on GET | play_count / download_count increments removed; counts are fixed seed data |
| MINOR: task --12 download tie | activities now have explicit pairwise-distinct downloads; unique max = Net Force Investigation (8742) → Forces and Motion: Basics |
| MINOR: tasks --6/--24 multiple answers | reworded with tighter constraints → exactly one answer each (Plinko Probability / Magnet and Compass), verified programmatically |
| MINOR: task --28 all versions 1.0.0 | versions derived deterministically from seed constants (47 distinct; Wave Interference = 1.5.2); task reworded to name the sim |
| BLOCKER: .assets-revision pinned to `main` | step 4 below — needs the HF merge SHA, which only exists after your HF PR merges |

Also re-verified: 66/66 routes return 200; reset cycle stays
byte-idempotent (`md5 48ca438b8ac6dab37a6503a4e5574503`); two GETs of the
same page leave the DB untouched.

## Steps

```bash
# 1. Apply the fix on your branch
cd WebHarbor
git checkout add-phet-simulations
git am ~/Downloads/0001-fix-phet_simulations-address-PR-29-review-feedback.patch

# 2. Swap ONLY the seed DB into your local tree, then REPACK the tarball
#    from your working copy.
#
#    ⚠️ DO NOT upload the delivered phet_simulations.tar.gz directly to HF —
#    it contains only instance_seed/. Your site's images
#    (static/images/ui/*.svg, static/images/sims/*.png) live in your local
#    tree and in your current HF tarball; uploading the seed-only tarball
#    would drop them. Extract just the .db and repack locally:
tar -xzf ~/Downloads/phet_simulations.tar.gz -C sites \
    phet_simulations/instance_seed/phet_simulations.db
md5sum sites/phet_simulations/instance_seed/phet_simulations.db
# → 48ca438b8ac6dab37a6503a4e5574503

./scripts/extract_assets.sh ../wh-static-pr/ phet_simulations
# packs instance_seed + static/images + external_cache from YOUR tree
tar -tzf ../wh-static-pr/phet_simulations.tar.gz | head   # sanity: images present

hf upload <your-hf-username>/WebHarbor \
    ../wh-static-pr/phet_simulations.tar.gz \
    phet_simulations.tar.gz --repo-type dataset
# If your HF PR is still open this updates it in place.
# Ask the maintainer to merge the HF PR, then copy the MERGE COMMIT SHA.

# 3. Local re-test (optional but recommended — paste output in PR reply)
mkdir -p sites/phet_simulations/instance_seed
tar -xzf ~/Downloads/phet_simulations.tar.gz -C sites \
    phet_simulations/instance_seed/phet_simulations.db
./scripts/build.sh
docker run -d --rm --name wh-test \
    -p 8101:8101 -p 40000-40015:40000-40015 webharbor:dev
sleep 8
curl -fsS http://127.0.0.1:40015/ | grep -c 'class="sim-card"'   # expect 20
curl -fsS -X POST http://127.0.0.1:8101/reset/phet_simulations
docker exec wh-test md5sum \
    /opt/WebSyn/phet_simulations/instance{,_seed}/phet_simulations.db
# both md5s = 48ca438b8ac6dab37a6503a4e5574503
docker stop wh-test

# 4. Repin .assets-revision to the HF MERGE SHA (blocker #1)
sed -i.bak "s/^revision:.*/revision: <HF-MERGE-SHA>/" .assets-revision && rm .assets-revision.bak
git add .assets-revision
git commit -m "chore(phet_simulations): pin assets to merged HF revision <HF-MERGE-SHA>"

# 5. Push — the PR updates automatically
git push origin add-phet-simulations
```

## Draft reply to post on PR #29

> Thanks for the thorough review — all five points addressed:
>
> 1. **Homepage** now renders the route's `featured` / `new_sims` /
>    `most_played` as three sim grids (20 cards), so task --41 is
>    solvable from `/`.
> 2. **'New' filter** is functional: `release=new` filters `is_new`,
>    `release=updated` filters release date ≥ 2024-01-01; checkbox
>    state, active chip, sort, and pagination all persist the param.
>    Task --21 is solvable via UI (7 New sims, all released 2025).
> 3. **No more GET mutations** — the `play_count` / `download_count`
>    increments are gone; counts are fixed seed data, so --6/--31/--41
>    are deterministic.
> 4. **Task ambiguities**: activities now carry explicit
>    pairwise-distinct download counts (unique max: Net Force
>    Investigation, 8742 → Forces and Motion: Basics, task --12); sim
>    versions are now varied and deterministic (47 distinct, task
>    --28); tasks --6/--24/--28/--31 reworded so each has exactly one
>    valid answer — verified programmatically against the seed.
> 5. **`.assets-revision`** repinned to the merged HF revision
>    `<HF-MERGE-SHA>` (the seed DB changed with these fixes — new
>    tarball uploaded to the HF PR, md5
>    `48ca438b8ac6dab37a6503a4e5574503`).
>
> Re-verified locally: 66/66 routes 200, reset cycle byte-idempotent,
> repeated GETs leave the DB untouched.

## Order matters

Do **step 2 (HF upload + merge) before step 4** — the repin needs the
merge SHA. If the maintainer prefers to merge the HF PR only together
with the GitHub PR, push steps 1+5 first with `.assets-revision` still
on `main`, note in the reply that the repin commit lands as soon as the
HF side merges, and ask them to confirm the order they want.
