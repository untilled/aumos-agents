---
name: ct-event-sweep
description: "How this desk finds an event it did not already know about: the whole-market OpenDART index call the sweep is made of, the corp_code join every per-issuer read owes, the classification it has to do itself because the filings index has no type filter, the difference between status 013 and status 020, the policy-gazette web lane through observation:file, and how the candidate ledger is written back with a hash. Read this before sweeping, before recording a vendor failure, and before moving the cursor."
---

# The sweep, and the four places it goes wrong quietly

`PROMPT.md` governs. This document is the **procedure** behind its Stage 2, written down because the
sweep has four failure modes that all look like success: a cursor that walked over a range nobody
read, an empty answer mistaken for an exhausted quota, a candidate built from a search-result
summary, and a ledger written over another run's.

## 1. The store for documents, the vendor for the index

⛔ **Do not build a second store in private memory.** The host keeps a per-fund source cache and #305
is explicit that this package reuses it rather than copying documents into `manager-memory`. Two
tools: `source_cache_read` to read what is held, `source_cache_refresh` to go and get more.

⚠️ **The one thing the store cannot answer is the index, so the index call is yours and it is
allowed.** `source_cache_read` and `filings_list` return `documentKey` (= `rcept_no`), `version`,
`publishedAt` and `capturedAt` — and **no `report_nm`**, which is the only field §3 classifies from
(measured in `run_28dc224c3d984252a10eeb2597471ec4`, and filed with Aumos; until it lands the vendor
is the route). So the two divide by job: **classification is a direct `source_request` to `open-dart`
`/api/list.json`**, which is on this package's allowlist and is §3's main road; **the documents of a
named issuer are collected and kept through `source_cache_*`**. What #305 forbids is the private
copy, not the call.

⚠️ **Incrementality is store-side, and it is not a cursor.** There is **no `since` parameter and no
vendor cursor.** What you get instead is a **state**, and the four words are the whole contract:

| `source_cache_read.state` | what it means | your lane |
|---|---|---|
| `never-fetched` | nobody has ever asked for this document | `unstated` |
| `refresh-failed` | somebody asked and the vendor refused | `dark` |
| `fresh` | held, and younger than the `freshFor` you required | `open` |
| `stale` | held, and older than you required | `partial` |

| `source_cache_refresh.state` | what it means | your lane |
|---|---|---|
| `observed` | fetched, and there are documents | `open` |
| `observed-empty` | fetched, and there are none | `open` — ⚠️ **this one is a fact about the issuer** |
| `satisfied` | already fresh enough; nothing was fetched | `open` |
| `failed` | the vendor refused | `dark` |

⚠️ **`freshFor` is required on every read.** There is no default, and "how fresh is fresh enough" is
a decision this desk makes rather than one it inherits.

## 2. The join, and the sweep that does not need it

A filings request **for one issuer** is keyed by **`corp_code`**, which is OpenDART's own identifier
and is *not* the six-digit KRX ticker. The mapping lives in the `open-dart/corp-codes` document,
whose `vendorId` is the stock code (`005930`) and whose document key is `corpcode:<corp_code>`.

```text
symbol (005930)  →  open-dart/corp-codes  →  corp_code  →  open-dart/filings
```

⛔ **Wherever a request names an issuer, this is a required step and never a fallback.** A run that
"tries the ticker first and falls back to the corp-code list" will, for a subset of issuers, get an
answer that is somebody else's filings or no filings at all — and the second is indistinguishable
from an issuer that filed nothing. If the corp-code document cannot be read, the filing lane is
`dark` for that issuer. It is not empty.

⚠️ **§3's sweep names no issuer, so it owes this join nothing.** The join is what a **deep read** of
one company costs — three to five of them in a run — and never what *looking* costs. Paying it per
ticker before anything has been classified turns KOSPI's 804 issuers into 2,412 calls, which is not a
slower sweep but a sweep that cannot finish; the run that tried it read **one** name.

## 3. The whole-market index, and what you have to do yourself

The filings index is an **index**, not a set of documents. Per row: `rcept_no`, `rcept_dt`,
`corp_name`, `corp_cls`, `report_nm`. ⛔ No figures. No normalised fields. **No type filter.**

⛔ **The sweep is one market-wide call, and `corp_code` is the parameter you leave out of it.** With
no issuer named, `/api/list.json` answers for every issuer that filed in the window; `total_count`
says how many rows stand behind the answer and `page_no` walks them 100 at a time. `bgn_de` is the
day after the range your cursor already holds and `end_de` is `asOf` — the five-year window belongs
to a **deep read** of one issuer, not to looking. Measured, in
`run_28dc224c3d984252a10eeb2597471ec4`: `bgn_de=20260814`, `end_de=20260913`, `corp_cls=Y`,
`pblntf_ty=B`, `sort=date`, `sort_mth=desc`, `page_no=1`, `page_count=100` → status `000`,
`total_count` 137, 100 rows in hand, **one call**.

| `corp_cls` | market | | `pblntf_ty` | what is filed under it, and what this desk reads it for |
|---|---|---|---|---|
| `Y` | 유가증권 (KOSPI) | | `A` | 정기공시 — the report a confirming indicator is finally read out of |
| `K` | 코스닥 | | `B` | 주요사항보고 — ⚠️ **this desk's lane**: 유상증자, 회사채 발행, 자산 양수도, 회생절차, 영업정지 |
| `N` | 코넥스 | | `C` · `D` | 발행공시 · 지분공시 — the raise once B announced it; ownership is another desk's entrance |
| `E` | 기타 | | `E` · `F` | 기타공시 · 외부감사관련 — an audit opinion is a survivability fact |
| | | | `I` | 거래소공시 — ⚠️ 실적공시 and 조회공시 해명, where a narrowing loss shows up before the report |
| | | | `G` `H` `J` | 펀드공시 · 자산유동화 · 공정위공시 — not this desk |

⚠️ **The run is a fan-out, and the two halves do not cost the same.** `discoveryIndexPages` cheap
whole-market pages (3 by default) → classify every row from `report_nm` → carry **three to five**
names into the expensive per-issuer work, which is §2's join and then the filings and the financials.
⛔ Inverted, the sweep cannot finish at any budget, and a desk whose every run therefore reports
`discovery_incomplete` has a status word that carries no information.

Three consequences, and all three are yours to handle:

1. **You classify from `report_nm`.** There is no `type=` parameter to ask for tariff revisions. Read
   the report name, place it in the sweep vocabulary, and when you cannot place it, say so — an
   unclassified row is something to look at again and never something that qualified. The vocabulary
   is `lib/discovery.mjs`'s `SWEEP_EVENT_KINDS`, and it is #305's four bullets:

   | #305's bullet | sweep kinds |
   |---|---|
   | 요금·단가 정상화 | `tariff-or-price-normalisation` |
   | 차환 확정 · 부채 감소 · 자산 매각 | `refinancing-secured`, `debt-reduction`, `asset-disposal` |
   | 생산 재개 · 구조조정 완료 | `production-restart`, `restructuring-completion` |
   | 미수금·재고·운전자본 정상화 / 손실 축소 | `working-capital-normalisation`, `loss-narrowing` |

   Each one carries the catalyst kind it becomes if it is ever registered, and the recovery channel it
   claims. ⚠️ Two events naming one channel are **one** improving channel, not two — that is
   `minImprovingChannels`' whole point and it starts here.

2. **The page is the unit and the receipt is the boundary.** Because the index is sorted
   newest-first, incrementality is: read down the page until you reach the `rcept_no` your ledger's
   cursor holds, and stop. Everything above it is this run's range. The cursor you write back is the
   **newest receipt in the newest range you finished**, `{ "kind": "dart-receipt", "value": "<rcept_no>" }`.

3. **`rcept_dt` is the receipt date and not the period.** The two instants a filing has — the period it
   covers and the moment it became public — are never interchangeable in this package, and the index
   gives you only the second. The first comes out of the document.

## 4. `013` is not `020`, and the difference is the record

| status | what OpenDART means | what you do |
|---|---|---|
| `000` | success | record the range as succeeded; move the cursor |
| `013` | **no matching document** | ⚠️ record the range as **succeeded and empty**; move the cursor. The host's own cache turns this into `observed` with zero documents |
| `020` | **the key's daily quota is spent** | ⛔ record the range as **failed**; the cursor does **not** move; the range goes into `failedRanges` and is the first thing the next run retries |
| anything else | vendor refusal | as `020` |

⛔ **Read one as the other and one of two silent things happens.** `013` read as a failure parks the
cursor on an empty stretch forever and reports `discovery_incomplete` on a complete sweep. `020` read
as an empty answer moves the cursor over a range nobody read, and those filings are never seen again
by anything.

## 5. Before you write down a vendor failure, test the route

⚠️ **A refusal on one path is not a dead source.** Aumos's own refusals (`source-failed`,
`as-of-missing`, `as-of-in-future`, `post-as-of-timestamp`) mean the *request* was wrong, and the
vendor's refusals arrive with a status of their own. So before recording a lane as `dark`:

- try a **sibling route** on the same source — a different document for the same issuer, or the same
  document for an issuer you know is held. If the sibling answers, what failed is this request;
- read the code. A time-window refusal is not a rate limit and ⛔ **is not retried with a different
  `asOf`**;
- one path per call. There is **no batch partial-failure envelope**: a call that fails, fails whole,
  so a per-symbol failure has to come from a per-symbol call or it is not a per-symbol failure.

Only then does the range go into `failedRanges` with a `reasonCode`, and only then is a lane `dark`.

## 6. The web lane: a gazette is a source, a search result is not

Half of these cases begin outside OpenDART — a ministry notice, a tariff decision, a regulator's own
calendar. That lane is required by this strategy, and it has one route.

⛔ **Never build a candidate from a search result's title or snippet.** File the original through
`observation:file` and carry five things with it:

| field | what it is |
|---|---|
| `url` | the original document. Not a search page, not a homepage |
| `title` | the document's own title |
| `excerpt` | the source's **own words**, verbatim, at most 64000 characters — over the cap is a refusal, not a truncation |
| `publishedAt` | the publication date. ⛔ Omit it if you do not know it; never guess. Later than `asOf` is a refusal |
| `reading` | **your** interpretation, kept separate from the excerpt on purpose |

The call returns an **`evidenceId`**, and that id is what a catalyst row and a candidate cite.

⚠️ **The round trip spans two runs.** Evidence filed during a run that has not ended is not committed
until the run finishes, so an observation filed in run *N* is citable in run *N+1*. Plan for it:
write the id into the candidate ledger this run with the question it answers, and cite it next run.
A run that files a reading and immediately builds a proposal on it is a run whose citation may not
resolve.

⛔ **Never mint an evidence id yourself.** A catalyst with an invented citation is worse than an
unregistered one.

## 7. Writing the ledger back

The candidate ledger is **one document** — `state/candidate-ledger.json` — with the cursor inside it.
That is not a style choice: `manager-memory` has no multi-file atomic write, so a cursor in a second
file is a cursor that can disagree with the candidates it belongs to.

1. `files_read` the document, and keep the **`hash`** the listing gave you.
2. Compute the next document.
3. `files_write` with `expectedHash` set to that hash — or to `null` when the file did not exist.

⚠️ **`expectedHash` is the whole concurrency story.** A mismatch comes back as `revision-conflict`,
which means somebody wrote between your read and your write: **re-read, re-apply, write again.** Do
not force the write. A `file-lease-held` error is a different thing — another writer holds the 60
second advisory lease — and it is retried rather than re-applied.

⛔ **What may never be written into it**: prices, bars, candles, filing bodies or excerpts; holdings,
quantities, weights, cash, average costs or target weights; pending proposals. Those are re-read from
the host every run. `lib/candidate-memory.mjs` refuses them at `blocked`, and the reason it is a rule
rather than advice is that a stale private copy reads exactly like a fresh one.

## 8. What the sweep must still produce when it produces nothing

- the **discovery record**, always, with the four-status verdict and both cursors;
- ⛔ `no_candidate_qualified` only when the universe was declared, every required lane is `open` and
  no symbol failed. Otherwise the answer is `discovery_not_run` or `discovery_incomplete`;
- ⚠️ and when two of the four hold at once, the precedence decides:
  `discovery_not_run` > `discovery_incomplete` > `candidates_produced` > `no_candidate_qualified`.
  A failed range **or** a required lane that came back `partial` outranks a name that came through
  the gate — the name is still counted and still written to the ledger, but the run is reported as
  `discovery_incomplete` and the cursor stays where it was found;
- the failed ranges, so the next run starts by retrying them;
- and if the status is `discovery_not_run`, the token `discovery_not_run` verbatim in one
  `uncertainty` entry of the proposal.

A sweep that read one gazette and nothing else is a complete run with a finding in it. A sweep that
read nothing and reports a clean screen is the one thing this document exists to stop.
