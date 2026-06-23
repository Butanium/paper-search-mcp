# paper-search-mcp — Butanium audited fork

This is **our fork** of [`openags/paper-search-mcp`](https://github.com/openags/paper-search-mcp)
(an MCP server for searching/downloading academic papers). It is consumed as a
**git submodule** inside `~/.claude` at `mcp/paper-search-mcp`, and the Claude Code
MCP config runs it directly from this source tree:

```
uv run --project ~/.claude/mcp/paper-search-mcp python -m paper_search_mcp.server
```

## Why we forked (not just `uv --with paper-search-mcp`)

1. **Security:** we run only code we've read. PyPI / git HEAD could change under us;
   a pinned, audited fork can't. See the audit record below.
2. **Control:** we can fix/improve in-place and contribute back upstream.

We did **not** fork to fix the original arXiv-search bug — upstream already fixed
that. (PyPI `0.1.3` shipped `search_arxiv` with `sortBy=submittedDate` + a raw
multi-word query, which arXiv's date-sorted path silently ignores, returning the
newest papers regardless of the query. Upstream commit `6d1f7ef5`, 2026-04-03,
switched the default to `sortBy=relevance` and made sort configurable — that fix
is in this fork. PyPI just hadn't released it as of June 2026.)

## Audit record

- **2026-06-23 — commit `dba2c743` — VERDICT: CLEAN.** Full read of all 34 source
  files + grep/URL/env sweep. No `eval`/`exec`/`subprocess`/`pickle`/`socket`/
  `base64`/dynamic-import. No `.post()` anywhere — every network call is a GET to a
  legitimate academic host (arxiv, NCBI, crossref, openalex, semanticscholar, HAL,
  zenodo, CORE, …). Every env var read is a named API credential sent only to its
  own host; no blanket `os.environ` dump. Downloaded content is only parsed
  (XML/JSON/HTML/PDF), never deserialized into code. `config.py` is a standard
  dotenv loader (`setdefault`, never transmits env).
- **Re-audit on every upstream sync** (see workflow). Only the *new* commits need
  reading, not the whole tree again.

### Known flags (audited, accepted as-is — no local code change)
- **Sci-Hub is exposed** via `download_scihub` and `download_with_fallback`
  (the latter defaults `use_scihub=True`). Sci-Hub is a copyright-piracy mirror —
  opt-in per call; the LLM won't hit it unless asked. Revisit if we want it off by default.
- **TLS-downgrade-on-retry** (`verify=False`) in `openaire.py` / `citeseerx.py` on
  SSLError, and unconditionally in `sci_hub.py`. Scoped to the legit host; weakens
  MITM protection, returns only parsed metadata. Hygiene, not exfil.
- **Weak filename sanitizers** on caller-supplied `paper_id` in `iacr.py`/`chemrxiv.py`
  (and none in `arxiv.py`). Not remote-injected; `..` w/o a separator can't traverse.
- **Bug (not security):** `citeseerx.py` `download_pdf` uses `os.*` without importing
  `os` → `NameError`. That download path is dead.

## Tracking upstream — ALWAYS check before assuming we're current

Upstream is active (1900+★). To pull improvements:

```bash
cd ~/.claude/mcp/paper-search-mcp
git fetch upstream
git log --oneline HEAD..upstream/main      # what's new since our pin
# review the diff of the new commits (security re-audit of NEW code only):
git diff HEAD..upstream/main
git merge upstream/main                     # or cherry-pick
git push origin main                        # update our fork
uv sync                                      # refresh deps
# then in ~/.claude: git add mcp/paper-search-mcp && commit (bump submodule pointer)
```

When a proper PyPI release > 0.1.3 lands, we *could* drop this fork and go back to a
versioned pin — but only if we're happy giving up the read-before-run guarantee.

## Local changes vs upstream

**None.** This fork is a pure audited mirror of upstream + this `CLAUDE.md`. Keep it
that way when possible — zero code diff = trivial upstream merges. If you must change
code, record it here so future merges are conflict-aware, and prefer upstreaming the
fix as a PR over carrying a private diff.

### Candidate upstream PR (not yet done)
- Question-form NL queries (e.g. `"what is a sparse autoencoder"`) still return junk
  because `search_arxiv` sends `all:<full query>` without dropping stopwords. Keyword
  queries work fine. A small stopword-stripping / per-term `all:x AND all:y` reformat
  would fix it — better contributed upstream than carried locally.
