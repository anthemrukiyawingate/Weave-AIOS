# WEAVE AIOS

Self-hosted memory and routing for AI agents that work as a team: verifiable recall from their session transcripts, calibrated "knows / doesn't know" answers, a tiered model router, a hash-chained decision record, and per-agent fine-tuned adapters.

**See it run:** open this repository's GitHub Pages site (the link is in the About panel of the repository). The page runs the recall, router, lattice, consolidation, intention, redaction and decision-log components in the browser on sample records.

## What is here

This repository holds this README, the page (`docs/index.html`) and the licence. The source, tests, evaluation files and documentation are in the release archive `weave-aios-1.0.0.tar.gz`, attached to the release.

## How it works

| Layer | What it does |
|---|---|
| Archive (L0) | Agent session transcripts, kept unedited, inventoried and deduplicated across machines, each session attributed to the agent that spoke |
| Palace (L1) | Dialogue cut into turn-aligned drawers with secrets redacted; each drawer stores its SHA-256 and source lines, so a recalled passage can be rebuilt and checked. SQLite with full-text search, plus a Qdrant vector store |
| Search | BM25 and dense vectors (Qwen3-Embedding-0.6B) fused by reciprocal rank, 1:1; copies folded into their earliest drawer |
| Lattice codes | Each 1,024-dimension vector quantised block by block to the E8 lattice: a compact resident shortlist with exact re-rank, and lattice cells as rooms for consolidation |
| Meta-memory | A calibrated threshold on the palace's confidence, so recall abstains when it holds no answer |
| Router | Code, then recall, then a local model, then a frontier model, then the principal; a judge model scores each answer before it is accepted, and purchases, publishing and destructive acts always go to the principal |
| Decision stream | Every routing decision and agent action appended to a hash-chained log |
| Adapters | Per-agent training sets (time-split held-out data) and evaluation gates for LoRA adapters |
| Agents' access | An MCP server, small HTTP endpoints, and Claude Code mods for recall, redaction and the decision stream |

## The release at a glance

| | |
|---|---|
| Version | 1.0.0 |
| Tests | 38 passing, 8 skipped on a fresh install (they need an indexed palace or a running model server) |
| Retrieval (LongMemEval-S, 470 questions) | evidence session in the top 5 for 0.979 of questions; NDCG@10 0.916 (hybrid) |
| Licence | MIT |

## Run it yourself

Unpack the archive, then:

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
cp config/site.example.toml config/site.toml      # every installation-specific setting lives here
cp config/site.env.example config/site.env        # the same for the shell scripts
python -m pytest -q tests
```

Then `scripts/qdrant.sh up` starts the vector store, and `inventory.py`, `sessions.py`, `drawers.py`, `index.py` and `consolidate.py` (in `scripts/`) build the palace from the transcript sources named in `config/site.toml`. Requires Python 3.11 or newer and Docker for Qdrant. The model tiers need OpenAI-compatible endpoints; their addresses go in `config/site.toml`.

## Known limits

- One host holds the database and the vector store.
- The router's judge and the adapter evaluation expect a judge model with a calibrated yes/no readout; other judges work through log-probabilities and are less well calibrated.
- The fact layer (`scripts/facts.py`, Graphiti over FalkorDB) is included and is not part of the measured results: with an 8B local extraction model it fails on long records.
- Tests that need an indexed palace or a running model server skip on a fresh install, and say why.
