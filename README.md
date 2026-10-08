# Policy Hacker — AI Policy & Government-News Digest

Monitoring Latvian government and policy sources by hand is slow, and 95% of what you read
is irrelevant to any given startup. **Policy Hacker** reads seven official sources end to
end — full bill text, meeting protocols, agency news — and produces a short, Latvian-language
digest of only the items that actually matter to startups: new funding to apply for, and
regulatory changes to react to.

What makes it more than a scraper-plus-keyword-filter:

- A **two-tier model pipeline** — cheap classification at scale, expensive verification
  only where it pays off.
- A **deep-verification agent** that, for legislative items, goes and reads the *real*
  primary law text with domain-restricted web tools before trusting a relevance claim.
- **Two-stage deduplication** — string similarity *and* an LLM "same real-world event?"
  judgment — so the same story on three sources shows up once.
- **Reliability engineering** that treats a wrong "relevant" verdict as the thing to
  design against: deterministic code backstops behind the prompt, confidence filtering,
  and fail-closed verification, all pinned by **107 tests and a 31-case eval suite**.

**▶ Live demo:** **<https://smalkaisk-hash.github.io/Policy-hacker/>** — rebuilt
automatically every week by GitHub Actions (no server).
**📄 Example output:** a full digest from a real run on live data →
**[`sample_digest/digest_example.md`](sample_digest/digest_example.md)** (renders inline
right here on GitHub).

---

## How it works (the 20-second version)

```
fetch (7 sources, full text) → dedupe (string) → classify (Haiku) → dedupe (LLM)
   → deep-verify legislative items (Sonnet + web tools) → render Latvian digest
```

Three parts are worth calling out:

1. **Model tiering — Haiku for breadth, Sonnet for depth.** Every candidate item is
   classified for relevance by **Claude Haiku** (`claude-haiku-4-5`) in fast, cheap batches
   of 10. Only the small subset that is *both* relevant *and* from a law/government-decision
   source escalates to **Claude Sonnet 5** for deep verification — the expensive stage is
   bounded to where it earns its cost, not run on every candidate.

2. **A deep-verification agent with domain-restricted web tools.** A Saeima committee
   agenda often reads as nothing but a routing list of bill titles — no description of what
   any amendment actually *does*. So the verifier gives Sonnet live `web_search` /
   `web_fetch` server tools **locked to official Latvian government/legal domains**
   (`likumi.lv`, `saeima.lv`, `tapportals.mk.gov.lv`, `data.gov.lv`, `mk.gov.lv`,
   `likumprojekti.lv`, `vestnesis.lv`), hands it the scraped bill/document number as a
   search key, and has it find and read the *real* primary text before confirming or
   rejecting the startup-relevance claim. It **fails closed**: an item it can't confirm
   against real primary text is held back, not shown with a caveat. Confirmed items carry a
   "Pārbaudīts pret oriģinālo tekstu" line linking the source it actually read.

3. **Dual deduplication.** The same story is often syndicated across sources. Stage one
   (`difflib` text similarity, ≥ 0.82 ratio) catches verbatim and near-verbatim copies for
   free. Stage two is an **LLM judgment call** — two independently *written* articles about
   the same event can share almost no wording, so a string compare misses them; Claude
   decides "same real-world event?" instead. Dedup never compares two items from one source
   (a source's own scrape is unique by construction), and keys on content rather than title
   alone (standing committees reuse the same agenda title on different days) — both learned
   the hard way.

Full step-by-step pipeline is in the [Pipeline in detail](#pipeline-in-detail) section.

---

## Repository layout

```
run_digest.py                  CLI entrypoint: orchestrates the whole pipeline
policy_digest/
  sources/                     one fetcher module per source, all returning a common Item
    base.py                      the Item dataclass (source, title, url, date, raw_text, …)
    tap_legal_acts.py            TAP portāls: data.gov.lv dataset + structuralizer/.docx text
    mk_meetings.py               VSS + MK protocol meeting feeds
    news_listing.py              EM + LIAA (shared gov CMS)
    altum_news.py                Altum via its open WordPress REST API
    saeima_committees.py         Saeima committee agendas via titania.saeima.lv
  classify.py                  Haiku relevance classification + deterministic backstops
  dedupe.py                    two-stage dedup (string similarity, then LLM)
  verify.py                    Sonnet deep-verification agent (domain-restricted web tools)
  digest.py                    Latvian/branded Markdown + HTML rendering
  state.py                     seen-before tracking (output/state.json)
evals/classification_evals.py  31 real-item fixtures the classifier must get right
tests/                         107 tests (sources, dedup, verify, reliability backstops)
.github/workflows/digest.yml   weekly GitHub Actions run → GitHub Pages
sample_digest/                 a checked-in example digest from a real run
```

## Sources covered

All 7 sources are implemented, each pulling the **full article/document body**, not just the
headline. Several deliberately use a source's own open API/dataset rather than scraping its
JavaScript front-end — more robust and no rate-limit games.

| Source | How it's fetched |
|---|---|
| **TAP portāls** | [Open dataset on data.gov.lv](https://data.gov.lv/dati/lv/dataset/tap-publicetie-tiesibu-akti) for metadata, plus full act text via TAP's public "structuralizer" preview endpoint, or a direct `.docx` attachment parsed with `python-docx` when no preview exists. |
| Valsts sekretāru sanāksme | `tapportals.mk.gov.lv/meetings/state_secretaries` — full agenda item text. |
| Ministru kabineta protokoli | Same mechanism, `tapportals.mk.gov.lv/meetings/cabinet_ministers`. |
| Ekonomikas ministrija | `em.gov.lv/lv/jaunumi` listing + each article's full body. |
| LIAA | Same CMS as EM, `liaa.gov.lv/lv/jaunumi`. |
| Altum | `altum.lv`'s open WordPress REST API (`/wp-json/wp/v2/posts`) — full pagination and server-side date filtering, so the lookback window isn't capped, and the full article body comes back in the same response (no second request needed). |
| Saeimas komisiju darba kārtības | Public agenda feed on `titania.saeima.lv` (reached via saeima.lv's own committee-agenda link), all committees' sittings per day. |

## What counts as "startup-relevant"

An item is flagged if it involves:
- **Funding & support programs** — grants, EU funds, accelerator/incubator programs,
  LIAA/Altum initiatives, investment/VC programs.
- **Regulatory or legal changes** affecting startups — company law, tax treatment, labor
  law, digital services/AI regulation, procurement rules for tech vendors.
- **Draft legislation or government initiatives** on innovation, digitalization, or
  entrepreneurship.

Routine administrative/personnel/ceremonial items and unrelated sector regulation are
excluded, along with events (contests, mentor calls, course cohorts) and generic
"business in general" programs with no startup/SME-specific eligibility scoping.
Definition lives in [`policy_digest/classify.py`](policy_digest/classify.py)
(`SYSTEM_PROMPT`, and `KEYWORDS` for the no-API-key fallback).

## Reliability engineering

The hard part of this project isn't fetching pages — it's making an LLM's "relevant / not
relevant" calls trustworthy enough that a reader doesn't have to re-check them against
primary sources. Three principles, each learned from a real failure caught by fact-checking
production output:

- **A prompt rule the model has already broken needs a deterministic code backstop, not
  stronger wording.** LLM output is probabilistic; re-phrasing the same instruction more
  forcefully doesn't reliably fix a pattern the model has demonstrably violated. So specific
  failure modes are enforced in code, independent of the prompt — e.g.
  `_is_criminal_procedure_cooperation_bill` (a cybercrime-convention ratification bill that
  kept getting justified as "affects digital service providers") and a discussion-only
  title gate (a minister *discussing* a regulation is not a policy action).
- **Exclude low-confidence output — don't flag it for review.** Items below
  `MIN_CONFIDENCE_TO_INCLUDE` (0.75) are held back from the digest entirely, not shown with
  a "check this manually" badge. Flagging just relocates the reliability problem onto a
  human.
- **Fail closed.** The deep-verification agent only lets a legislative item through if it
  can find and read the real primary text confirming the scope claim. Any failure mode — no
  source found, ambiguous result, API error — holds the item back rather than letting an
  unverified guess ship.

These are pinned by regression coverage, so a future prompt edit can't silently reopen a
closed hole — see [Tests & evals](#tests--evals).

## Pipeline in detail

1. **Fetch** — one module per source under `policy_digest/sources/`, returning normalized
   `Item`s with the real article/document body attached (truncated to ~4 000 chars for
   classification), not just a title. One source failing is logged and skipped; the run
   continues.
2. **Dedupe (seen-before)** — `policy_digest/state.py` tracks previously seen item URLs in
   `output/state.json`, so re-runs only surface genuinely new items. Items that failed to
   classify this run are *not* marked seen, so they're retried next run.
3. **Dedupe (cross-source)** — `policy_digest/dedupe.py` merges the same underlying
   article/document when it's published on more than one source: first by `difflib` text
   similarity (≥ 0.82), then an LLM pass (`claude-haiku-4-5`) for independently-written
   pieces about the same event. Only cross-source pairs are ever compared.
4. **Classify** — `policy_digest/classify.py` sends items to **Claude Haiku** in batches of
   10, via forced tool-use, for a relevance verdict, confidence, one-line reason, category
   (`funding` vs. everything else), and any extracted deadline. Falls back to a free keyword
   pre-filter if no `ANTHROPIC_API_KEY` is set. Includes the deterministic backstops above,
   plus defensive parsing (a numeric-string `index` won't silently drop a whole batch).
5. **Deep-verify (legislative items only)** — `policy_digest/verify.py` takes every item
   still marked relevant from the government-decision/law sources (TAP portāls, Saeima
   committees, the MK/VSS meeting feeds) and gives **Claude Sonnet 5** live
   `web_search`/`web_fetch` tools, restricted to official Latvian government/legal domains,
   to find and actually read the bill's real text — using the scraped bill/document number
   as the search key — and confirm or reject the relevance claim against that primary text,
   not just the (sometimes bare-titles-only) agenda snippet. A confirmed item gets a
   "Pārbaudīts pret oriģinālo tekstu" line linking the primary source read; an unconfirmable
   item is held back. News/funding sources (EM, LIAA, Altum) skip this step — they're already
   full-article text with no separate primary legal source to check against.
6. **Render** — `policy_digest/digest.py` writes a Latvian-language, startin.lv-branded
   digest as `output/digest_<date>.md` and `.html`, split into two sections — **Finansējuma
   iespējas** (funding, apply-for/deadline-driven) and **Regulējums un iniciatīvas**
   (monitor-and-react) — with source-level grouping inside each. A deadline found in an
   item's own text is shown next to it and flagged urgent inside 14 days. A coverage line
   lists every monitored source with its relevant-item count for the period, including zero,
   so a quiet source reads as "checked, nothing relevant" rather than "not checked".

## Setup

```bash
pip install -r requirements.txt
cp .env.example .env   # then fill in ANTHROPIC_API_KEY (optional — see below)
```

Requires Python 3.10+. Dependencies: `requests`, `beautifulsoup4`, `anthropic`,
`python-dotenv`, `python-docx`.

## Run it

```bash
python run_digest.py                # last 7 days, updates dedupe state
python run_digest.py --days 14       # wider window
python run_digest.py --no-state      # don't read/write dedupe state (repeatable demo runs)
```

Set `ANTHROPIC_API_KEY` in `.env` to get real LLM classification; omit it to run in free
keyword-only mode (no deep-verification — that stage requires the API). Output lands in
`output/digest_<today>.md` and `.html` (gitignored — a sample run generated **with** an API
key is checked into [`sample_digest/`](sample_digest/)).

## Tests & evals

```bash
python -m pytest                        # 107 tests: sources, dedup, verify, reliability backstops
python evals/classification_evals.py     # 31 real-item classifier fixtures
```

`tests/` covers each source fetcher, both dedup stages, the verification agent (mocked — no
live API calls), and the deterministic reliability backstops. `evals/classification_evals.py`
is a growing set of real items the classifier previously got wrong (and the carve-outs it
must *not* over-correct), run by hand against the live API after any `SYSTEM_PROMPT` change —
each past misclassification becomes a permanent fixture so it can't regress silently.

## Hosting a live version (GitHub Pages)

The [live demo](https://smalkaisk-hash.github.io/Policy-hacker/) is published by
[`.github/workflows/digest.yml`](.github/workflows/digest.yml), which runs the digest and
deploys it to GitHub Pages — free, no server. The workflow uses a 30-day window and
`--no-state` (each publish is a fresh, independent snapshot). One-time setup in the repo's
GitHub web UI:

1. **Add the API key as a secret**: Settings → Secrets and variables → Actions → New
   repository secret → name `ANTHROPIC_API_KEY`.
2. **Turn on Pages**: Settings → Pages → Build and deployment → Source: **GitHub Actions**.

After that, it publishes automatically every Monday, or on demand via the Actions tab →
"Publish policy digest" → Run workflow.
