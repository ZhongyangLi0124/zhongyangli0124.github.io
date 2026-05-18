# WebHarbor contribution: PhET Interactive Simulations

This directory contains a self-contained Flask + SQLite mirror of
[PhET Interactive Simulations](https://phet.colorado.edu/), packaged for
contribution to [aiming-lab/WebHarbor](https://github.com/aiming-lab/WebHarbor)
under Track A (new website mirror).

## Why PhET

Picked from the WebHarbor open-tasks list because it scores well against
the "easy-to-mirror" criteria:

- **University-hosted, no commercial reCAPTCHA / anti-bot.**
- **No payment flows, no third-party OAuth.**
- **Open CC-BY content** — safe to mirror.
- **Stable layout** — academic site, not redesigned monthly.
- **Modest catalog size** — ~100 sims, fits in a 150 KB SQLite seed.
- **Clear filter taxonomy** — subjects × grade levels × languages
  (≥20 records per filter, satisfying the spec).

## Layout

```
webharbor-phet/
├── README.md                   ← this file
├── INTEGRATION.md              ← exact diffs for the 3 WebHarbor files
└── sites/
    └── phet_simulations/       ← drop into <WebHarbor-fork>/sites/
        ├── app.py              ← Flask app: 7 models, 17 routes, full seed
        ├── _health.py
        ├── tasks.jsonl         ← 43 benchmark prompts (agent eval input)
        ├── README.md           ← per-site notes
        ├── .gitignore
        ├── templates/          ← 16 Jinja2 templates
        ├── static/css/         ← single hand-rolled stylesheet, no CDNs
        ├── static/js/          ← vanilla JS for save / play handlers
        ├── instance/.gitkeep   ← runtime DB (gitignored, recreated at boot)
        ├── instance_seed/      ← seed DB (.db gitignored, ships via HF)
        └── scraped_data/.gitkeep
```

## How to submit it

1. Fork [aiming-lab/WebHarbor](https://github.com/aiming-lab/WebHarbor)
   and [aiming-lab/WebHarbor](https://huggingface.co/datasets/aiming-lab/WebHarbor)
   (the HF dataset).
2. Copy `sites/phet_simulations/` into the fork at `sites/phet_simulations/`.
3. Apply the three patches in `INTEGRATION.md`:
   - `websyn_start.sh` (add to `SITES=(...)`)
   - `control_server.py` (add to `SITES = [...]`)
   - `Dockerfile` (extend `EXPOSE` range to include the new port)
4. Build and run the container:
   ```bash
   ./scripts/build.sh
   docker run -d --rm --name wh-test \
       -p 8101:8101 -p 40000-40015:40000-40015 webharbor:dev
   curl http://127.0.0.1:40015/_health
   ```
5. Verify seed idempotency:
   ```bash
   curl -X POST http://127.0.0.1:8101/reset/phet_simulations
   docker exec wh-test md5sum \
       /opt/WebSyn/phet_simulations/instance{,_seed}/phet_simulations.db
   ```
6. Extract assets to the HF fork:
   ```bash
   ./scripts/extract_assets.sh ../wh-static-pr/ phet_simulations
   cd ../wh-static-pr
   hf upload-large-folder <your-fork>/WebHarbor . --repo-type dataset
   ```
7. After the HF PR merges, update `.assets-revision` in the GitHub fork
   with the merge commit SHA and open the GitHub PR.

## Local smoke test (without WebHarbor)

```bash
cd sites/phet_simulations
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt
PORT=40015 .venv/bin/python app.py
# → http://127.0.0.1:40015/
```
