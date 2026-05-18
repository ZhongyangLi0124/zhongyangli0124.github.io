# Integration patches for the WebHarbor fork

PhET is the 16th site, so it claims port slot **40015** (= `40000 + 15`,
appended to the existing 15-site list). The three control-plane files
need matching updates.

## 1. `websyn_start.sh`

Append `phet_simulations` to the `SITES` array. Also bump the count
strings in the log messages so they reflect 16 sites.

```diff
 SITES=(allrecipes amazon apple arxiv bbc_news booking github
        google_flights google_map google_search huggingface wolfram_alpha
-       cambridge_dictionary coursera espn)
+       cambridge_dictionary coursera espn phet_simulations)
 BASE_PORT=40000
 ...
-echo "[WebSyn] Starting 15 sites on ports ${BASE_PORT}-$((BASE_PORT + 14))..."
+echo "[WebSyn] Starting 16 sites on ports ${BASE_PORT}-$((BASE_PORT + 15))..."
 ...
-    echo "  [${elapsed}/${max_wait}s] ${ready}/15 sites ready"
-    if [ $ready -eq 15 ]; then
+    echo "  [${elapsed}/${max_wait}s] ${ready}/16 sites ready"
+    if [ $ready -eq 16 ]; then
         break
     fi
```

## 2. `control_server.py`

Append `'phet_simulations'` to the `SITES` list. The list **must match
the order in `websyn_start.sh` exactly** — per-site port is derived from
the list index.

```diff
 SITES = [
     'allrecipes', 'amazon', 'apple', 'arxiv', 'bbc_news', 'booking',
     'github', 'google_flights', 'google_map', 'google_search',
     'huggingface', 'wolfram_alpha', 'cambridge_dictionary',
-    'coursera', 'espn',
+    'coursera', 'espn', 'phet_simulations',
 ]
```

## 3. `Dockerfile`

Extend the `EXPOSE` range by one port to cover 40015.

```diff
-EXPOSE 8101 40000-40014
+EXPOSE 8101 40000-40015
```

## 4. Asset upload (HF dataset side)

After local testing succeeds, the seed database `instance_seed/phet_simulations.db`
ships via the Hugging Face dataset, **not** GitHub. Run from the WebHarbor
fork root:

```bash
./scripts/extract_assets.sh ../wh-static-pr/ phet_simulations
cd ../wh-static-pr
hf upload-large-folder <your-fork>/WebHarbor . --repo-type dataset
```

Once the HF PR merges, record the merge SHA in `.assets-revision` and
open the GitHub PR referencing it.

## Verification checklist

After building the container, all four checks must pass:

```bash
# (a) Site responds on its assigned port
curl -fsS http://127.0.0.1:40015/_health
# → {"ok": true, "site": "phet_simulations"}

# (b) Control plane sees PhET in healthy set
curl -fsS http://127.0.0.1:8101/health | jq '.sites.phet_simulations'

# (c) Reset endpoint succeeds
curl -fsS -X POST http://127.0.0.1:8101/reset/phet_simulations

# (d) After reset, instance/ and instance_seed/ are byte-identical
docker exec wh-test md5sum \
    /opt/WebSyn/phet_simulations/instance/phet_simulations.db \
    /opt/WebSyn/phet_simulations/instance_seed/phet_simulations.db
# → both md5s MUST match
```
