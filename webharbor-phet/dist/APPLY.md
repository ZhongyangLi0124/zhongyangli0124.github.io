# Apply the PhET contribution to your WebHarbor fork

The sandbox couldn't push directly to `ZhongyangLi0124/WebHarbor` (the
git proxy here only authorizes the personal-site repo, and rejects
writes to any other repo with `502 repository not authorized`). So the
work is shipped as a patch + asset tarball that you apply locally.

## Files in this directory

- `0001-feat-phet_simulations-add-new-site.patch` — git-formatted patch
  containing the full commit (Flask app, 16 templates, CSS/JS, tasks.jsonl,
  plus the 3-line patches to `Dockerfile`, `control_server.py`,
  `websyn_start.sh`). Plain text, 117 KB.
- `phet_simulations.tar.gz` — HF dataset asset bundle (17 KB) containing
  `instance_seed/phet_simulations.db` (143 KB seed) and the placeholder
  static asset dirs.

## Apply on your machine

```bash
# 1. clone your fork
git clone git@github.com:ZhongyangLi0124/WebHarbor.git
cd WebHarbor
git checkout -b add-phet-simulations

# 2. apply the code patch
git am /path/to/0001-feat-phet_simulations-add-new-site.patch

# 3. drop the seed DB in place so the local build can run
mkdir -p sites/phet_simulations/instance_seed
tar -xzf /path/to/phet_simulations.tar.gz \
    -C sites \
    phet_simulations/instance_seed/phet_simulations.db
ls -la sites/phet_simulations/instance_seed/

# 4. local smoke test (optional but recommended)
./scripts/check_assets.sh                         # phet_simulations should pass
./scripts/build.sh
docker run -d --rm --name wh-test \
    -p 8101:8101 -p 40000-40015:40000-40015 webharbor:dev
sleep 5
curl -fsS http://127.0.0.1:40015/_health          # → {"ok": true, "site": "phet_simulations"}
curl -fsS -X POST http://127.0.0.1:8101/reset/phet_simulations
docker exec wh-test md5sum \
    /opt/WebSyn/phet_simulations/instance{,_seed}/phet_simulations.db
# → both md5s MUST match (e094a2ee23369d3b60232f49f4ac691c)
docker stop wh-test

# 5. push the GitHub branch
git push -u origin add-phet-simulations

# 6. upload the asset tarball to your HF fork
#    (or to ChilleD/WebHarbor directly if you have write access — see CONTRIBUTING.md)
mkdir ../wh-static-pr
cp /path/to/phet_simulations.tar.gz ../wh-static-pr/
cd ../wh-static-pr
hf upload phet_simulations.tar.gz \
    <your-hf-username>/WebHarbor phet_simulations.tar.gz --repo-type dataset

# 7. after the HF PR merges, bump .assets-revision
cd ../WebHarbor
sed -i "s/^revision:.*/revision: <hf-merge-sha>/" .assets-revision
git add .assets-revision
git commit -m "chore: bump assets to <hf-merge-sha>"
git push

# 8. open the GitHub PR from the branch
```

## Verification already done in this sandbox

| Check                                                | Result |
| ---------------------------------------------------- | ------ |
| Cloned real `aiming-lab/WebHarbor` (your fork's base) | OK     |
| Site copied into real `sites/` tree                  | OK     |
| 3-file patch (Dockerfile / control_server / websyn_start) | OK |
| `./scripts/check_assets.sh sites/phet_simulations`   | OK (only PhET has its seed locally) |
| `./scripts/extract_assets.sh ../wh-static-pr/ phet_simulations` | OK (17 KB tarball) |
| App boots under exact Dockerfile pin set (Flask 3.1.0 etc.) | OK |
| 26 representative routes return 200 with non-empty body | OK |
| `/reset` cycle md5-identical (e094a2ee23369d3b60232f49f4ac691c) | OK |

## What's still up to you

- Docker build + `docker run` (this sandbox has no Docker daemon).
- HF dataset upload (needs your HF token, which is correctly NOT in the
  sandbox).
- Opening the actual GitHub PR.
- Submitting the [Contribution Request Form](https://forms.gle/ngcD1rzAfUEphNmRA)
  to claim the PhET slot before the PR — confirm it's still available on
  the [Contribution Track Sheet](https://docs.google.com/spreadsheets/d/1vZsrQjy9nJKze58fx4kbQtFi85NjVXIWCFyu3ShD7gk/edit).
