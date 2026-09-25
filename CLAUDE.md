# paper-search-mcp — Butanium audited fork

This is **our fork** of [`openags/paper-search-mcp`](https://github.com/openags/paper-search-mcp)
(an MCP server for searching/downloading academic papers). It is a **nested git
submodule**: it lives inside the `custom-claude-mcps` repo (`~/.claude/mcp`), which is
itself a submodule of `~/.claude`. It runs from this source tree as a shared
streamable-HTTP server, not per-session stdio: `~/.claude/mcp/serve_http.py paper-search`
(`uv run --project` into this venv, then serves `paper_search_mcp.server` on
`127.0.0.1:8874/mcp`) under the systemd user unit `claude-mcp@paper-search`.

**Code changes take effect only after `systemctl --user restart claude-mcp@paper-search`.**
Logs (e.g. the arXiv non-200 warnings): `journalctl --user -u claude-mcp@paper-search`.

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
git remote get-url upstream || git remote add upstream https://github.com/openags/paper-search-mcp
git fetch upstream
git log --oneline HEAD..upstream/main      # what's new since our pin
# review the diff of the new commits (security re-audit of NEW code only):
git diff HEAD..upstream/main
git merge upstream/main                     # or cherry-pick
git push origin main                        # update our fork
uv sync                                      # refresh deps
systemctl --user restart claude-mcp@paper-search
# bump the pointer one level up, in custom-claude-mcps:
cd ~/.claude/mcp && git add paper-search-mcp && git commit && git push
# then in ~/.claude: git add mcp && commit (bump the custom-claude-mcps pointer)
```

When a proper PyPI release > 0.1.3 lands, we *could* drop this fork and go back to a
versioned pin — but only if we're happy giving up the read-before-run guarantee.

## Local changes vs upstream

Keep the code diff minimal: zero diff = trivial upstream merges. Record every local
change here so merges are conflict-aware, and prefer upstreaming a fix as a PR over
carrying a private diff.

- **`academic_platforms/arxiv.py` (2026-09-25): no-ALPN handshake + loud failures.**
  From 2026-09-25 (worked 2026-09-16) arXiv's export API edge answered **HTTP 406, empty
  body**, to every cache-missing request from this venv's Python (uv CPython 3.13 /
  OpenSSL 3.5.6), via requests, httpx and urllib alike, so `search_arxiv` returned `[]`
  for every query. Verified cause: the TLS ClientHello, not HTTP. The same raw HTTP
  bytes sent over Python `ssl` get 406 when the handshake offers ALPN `http/1.1` and
  200 without ALPN; request headers (UA, Accept, Accept-Encoding, Connection) make no
  difference; curl (any HTTP version) and the system Python 3.14 (OpenSSL's default
  30-suite cipher list + `compress_certificate`, vs 3.13's own 17-suite list) get 200
  even with ALPN. Gotcha when re-testing: a CDN cache HIT (`X-Cache: ..., HIT`) returns
  200 to any client, so vary the URL (e.g. `start=`) per probe. Changes:
  1. `BASE_URL` is https (http only 301s to https; upstream made the same change).
  2. The session mounts `_NoALPNAdapter` for `https://export.arxiv.org/`. It is an
     `SSLContext` subclass whose `set_alpn_protocols` is a no-op, because urllib3 sets
     ALPN on every context it wraps. Certificate and hostname verification are
     unchanged (checked against badssl.com).
  3. A non-200 response or an exhausted network retry now logs a warning (status, URL,
     body) and **raises** instead of returning `[]`, so the tool call errors visibly and
     `search_papers` records it in `errors`. Upstream issue openags#121 reports the same
     406 as intermittent, and upstream `main` (checked at `808e462`) still returns `[]`.
  If arXiv changes the rule again, the symptom is now a
  `RuntimeError: arXiv API returned HTTP <code>` from `search_arxiv`, not silent zero
  results.

### Known quirk (no action planned)
- arXiv relevance search is stopword-sensitive: a question-form query like
  `"what is a sparse autoencoder"` returns junk, while the keyword form
  `"sparse autoencoder"` works. Not worth fixing — the caller is an LLM, which
  passes keywords, not natural-language questions.
