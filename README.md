# VenueWatch

An Intelligent Contract that reads a vendor's published terms page under GenLayer
consensus and settles whether the page agrees with the dispute policy you pinned.

Two questions come up before every vendor signature, and both are answered in one
sentence on one public page:

* **which law governs** the agreement, and
* **where arbitration is held**, if the page sends disputes to arbitration at all.

A company carries both as policy, for example "California law" and "we do not
accept an arbitration seat outside San Francisco". Today the check is a human
reading that sentence, pasting a screenshot into a thread, and typing
"California, no arbitration" into a spreadsheet. Nobody can recompute that later.
The vendor republishes the page, the spreadsheet keeps the old answer, and the
only surviving record of what the page said is a picture.

VenueWatch stores the check instead of the screenshot: the page, the topic, and
the policy string go in, and the page's own sentence comes back as evidence with
a verdict attached, re-derivable by anyone at any time.

## Live on studionet

Contract: `0x2b6AB6B7dfeB045f1969c23B69d9c6330700c251`
Deploy transaction: `0x7ae93cde434d603adfed9a242aca6b4a5f3a3a508c2f70312afdfa1f7b16049d`
Explorer: https://explorer-studio.genlayer.com/address/0x2b6AB6B7dfeB045f1969c23B69d9c6330700c251

Five checks were run against four live vendor pages, one policy throughout:
California law, and an arbitration seat in San Francisco if there is arbitration
at all. All ten transactions are FINALIZED with MAJORITY_AGREE, and the code
stored at that address is byte identical to `contracts/venuewatch.py` after
Studio's CRLF normalisation.

| check | page | topic | pinned policy | verdict | stated |
|---|---|---|---|---|---|
| V1 | anthropic.com/legal/consumer-terms | GOVERNING_LAW | State of California | MATCHES | the State of California |
| V2 | anthropic.com/legal/consumer-terms | ARBITRATION_VENUE | San Francisco, California | NOT_STATED | (none) |
| V3 | vercel.com/legal/terms | ARBITRATION_VENUE | San Francisco, California | MATCHES | San Francisco, California |
| V4 | cloudflare.com/terms/ | GOVERNING_LAW | Ireland | DIFFERS | State of California |
| V5 | stripe.com/legal/ssa | ARBITRATION_VENUE | San Francisco, California | AMBIGUOUS | (none) |

Two of those rows are the reason the design looks the way it does. V5: Stripe
publishes one global agreement whose regional terms name five seats, so a yes or
no answer would have to pick one and be wrong for four regions. V2: Anthropic's
consumer terms contain no arbitration clause at all, and the contract settles
that without calling a model, because an empty evidence region is a fact every
validator rebuilds identically from its own fetch.

`SUBMISSION.md` carries every transaction hash, the verbatim stored quotes, and
the audit rule by rule.

## The two topics

| topic | reads | example of what a page prints |
|---|---|---|
| `GOVERNING_LAW` | the jurisdiction whose law governs the agreement | "governed by the laws of the State of California" |
| `ARBITRATION_VENUE` | the place where arbitration under the agreement is held | "Arbitration will be held in San Francisco, California" |

## The four verdicts

| verdict | meaning |
|---|---|
| `MATCHES` | the page names a jurisdiction, and your policy string sits inside it |
| `DIFFERS` | the page names a jurisdiction, and your policy string does not sit inside it |
| `NOT_STATED` | the page makes no statement on this topic at all |
| `AMBIGUOUS` | the page names more than one, depending on region or domicile |

`NOT_STATED` is not a shrug. A vendor whose terms send every dispute to court has
no arbitration seat, and that absence is a fact the contract can prove. A policy
of "no arbitration" reads `NOT_STATED` as its pass.

`AMBIGUOUS` is the finding that ends most real reviews. A vendor that publishes
one global page with regional terms does not have one governing law: it has
California for the United States, Ireland for Europe, England and Wales for the
UK, New South Wales for Australia, and so on. A spreadsheet cannot hold that. A
verdict that says the answer depends on where the customer sits, with the
sentence that proves it, can.

States are `PENDING`, `SETTLED`, and `UNRESOLVED`. `UNRESOLVED` always carries a
reason and never carries a verdict, so a failed read can never be mistaken for a
pass.

## How a check settles

1. `open_check(source_url, topic, wanted)` validates the URL, the topic, and the
   policy string, then stores one record and returns its id. Opening the same
   triple twice returns the same id, so a check is a durable thing and not an
   event.
2. `resolve(id)` fetches the pinned page under consensus.
3. **The contract builds the evidence region.** The page is split into sentences
   and the ones carrying the topic's own marker words are kept, numbered, up to
   eight sentences and four thousand characters. For `GOVERNING_LAW` the markers
   are `governed by` and `governing law`. For `ARBITRATION_VENUE` a sentence must
   carry `arbitrat` **and** a place marker such as `held in`, `seat of`, `take
   place in`, or `venue`. Both marker lists are published by `rules()`.
4. **The model transcribes one field from that region.** It returns the
   jurisdiction exactly as printed plus the sentence it came from, verbatim. It
   is never told the policy string, and it never chooses what it is shown.
5. **The contract checks the transcription three ways.** The quoted sentence must
   exist inside the evidence region, it must carry the topic marker, and the
   stored phrase must be printed inside it, word for word, in order.
6. **The contract compares two strings.** Matching is word sequence containment
   on lowercased words, so "State of California" matches a page that prints "the
   laws of the State of California", and does not match a page that prints "the
   laws of the Federal Republic of Germany". No model, no score, no judgement.
7. **Every validator repeats all of it** from its own fetch and must reach the
   same state, verdict, and reason.

Step 3 is what keeps this cheap and honest at the same time: a model reads eight
sentences rather than a hundred thousand characters, and a caller cannot widen
the region to smuggle a jurisdiction into it.

## Why this is not a prompt wrapper

* **The policy string never reaches the model.** It is used in one place, the
  comparison in step 6, which runs in the contract. The model cannot be steered
  toward `MATCHES` because it cannot see what anyone wanted to hear. There is a
  test that registers a poison reading which fires only if the policy string ever
  appears in a prompt.
* **The evidence proves the claim, not just exists.** A quoted heading such as
  "Governing law and exclusive jurisdiction." is printed on the page and carries
  the marker, and it proves nothing. It is rejected, because the stored phrase
  has to be printed inside the quoted sentence itself. A fragment such as "or at
  such other venue as is mutually agreed upon." is rejected for the same reason:
  it grounds, but it carries no arbitration word. A governing law sentence offered
  as evidence for an arbitration seat is rejected because it never entered that
  topic's evidence region.
* **The validator recomputes and never trusts.** It rebuilds the evidence region
  from its own fetch, re-grounds the quote, re-binds the phrase, re-runs the whole
  read, and compares state, verdict, and reason. A leader that names a
  jurisdiction its own page does not print is rejected. A leader that claims
  silence on a page that speaks is rejected. A validator that cannot read the page
  rejects a settled claim rather than waving it through.
* **Silence is deterministic.** When the evidence region is empty the contract
  settles `NOT_STATED` without calling a model at all, because an empty region is
  a fact every reader rebuilds identically from its own fetch. There is a test
  that resolves with no model registered and still settles.
* **Two readers may phrase the same finding differently and still agree.** One
  may isolate "the State of California" and another only "California". Both bind
  their phrase to their own fetch and both reach `MATCHES`, so the claim is
  certified instead of deadlocked. A reader that reaches a different verdict is a
  no vote.
* **An oversized page is refused, not truncated.** A partial read can never look
  like a complete one.

## How this differs from a clause presence check

Contracts that answer "does this page contain an arbitration clause" return a
boolean about a clause. VenueWatch returns a **named value** and compares it
against a **policy the caller pinned**, which is a different mechanism end to
end:

| | clause presence check | VenueWatch |
|---|---|---|
| model output | yes or no | the jurisdiction, as printed, plus its sentence |
| stored evidence | that a clause was found | a phrase bound word for word to a quoted sentence |
| decision | was it there | does what was there satisfy a pinned policy string |
| verdicts | present or absent | `MATCHES`, `DIFFERS`, `NOT_STATED`, `AMBIGUOUS` |
| selection | whole document or caller chunks | the contract's own marker rule over the page's sentences |

The `AMBIGUOUS` verdict has no meaning in a presence check, and it is the answer
that matters most on the multi-region pages that large vendors publish.

## API

Writes:

```
open_check(source_url: str, topic: str, wanted: str) -> str   # "V1", "V2", ...
resolve(check_id: str) -> str                                  # the new state
```

Views:

```
get_check(check_id) -> dict   # every stored field, including quote and checked_at
verdict(check_id) -> str      # MATCHES / DIFFERS / NOT_STATED / AMBIGUOUS / ""
stated(check_id) -> str       # the jurisdiction exactly as the page printed it
matches(check_id) -> bool     # true only for a settled MATCHES
list_checks() -> list         # id, topic, wanted, state, verdict, stated
count() -> int
rules() -> dict               # topics, marker lists, verdicts, reasons, every limit
```

`get_check` returns:

```json
{
  "id": "V1",
  "source_url": "https://www.anthropic.com/legal/consumer-terms",
  "topic": "GOVERNING_LAW",
  "wanted": "State of California",
  "owner": "0x...",
  "state": "SETTLED",
  "verdict": "MATCHES",
  "reason": "",
  "stated": "the State of California",
  "quote": "Our Terms will be governed by, and construed and interpreted in accordance with, the laws of the State of California without giving effect to conflict of law principles.",
  "checked_at": "2026-11-21T15:02:11.482913Z",
  "version": 1
}
```

Resolving a settled check re-reads the page and bumps `version`, so a vendor that
moves its seat from San Francisco to Dublin shows up as a moved phrase and a new
`checked_at` instead of an overwritten history.

### Errors and reasons

Reverts at open time: `URL_INVALID` (https only, no spaces, no separator),
`TOPIC_INVALID`, `WANTED_INVALID` (3 to 80 characters), `CAPACITY_REACHED`,
`UNKNOWN_CHECK`.

Reverts at resolve time: `SOURCE_EMPTY_OR_UNREADABLE` (fewer than 200 characters
came back), `SOURCE_TOO_LARGE` (more than 400000).

Stored on an `UNRESOLVED` check: `EXTRACTION_FAILED`, `STATED_INVALID`,
`QUOTE_INVALID`, `QUOTE_NOT_GROUNDED`, `QUOTE_NOT_SUPPORTING`,
`STATED_NOT_IN_QUOTE`.

## Limits, stated plainly

* The evidence region is selected by a fixed marker list per topic. A page that
  states its governing law without `governed by` or `governing law`, or its
  arbitration seat without `arbitrat` plus a place marker, settles `NOT_STATED`.
  Both lists are in `rules()` rather than buried in code.
* Matching is word sequence containment, so a short policy string can sit inside
  several jurisdictions: "Ireland" also matches a page that prints "Northern
  Ireland". Pin the full name when the distinction matters.
* The caller pins the URL, so the caller owns the consequence of pinning a page
  that is not the vendor's real terms. The contract witnesses what the pinned page
  says; it does not know which page a vendor meant to publish.
* One page, one topic, one policy string per check. A vendor with separate
  consumer and commercial terms is two checks.
* A verdict is a claim about the page at `checked_at`. Vendors republish, and
  re-resolving is how you see that they did.
* Legal effect is out of scope. The contract reports what a page prints and
  whether it satisfies a string you pinned. It does not advise, and the model is
  instructed not to.

## Running it

```
pip install -r requirements.txt
python -m pytest tests/direct -q
genvm-lint check contracts/venuewatch.py
```

59 tests pass, the linter reports 9 methods (7 view, 2 write), and every fixture
page is served from a hermetic host so no test reaches the network. The suite
covers all four verdicts, both `NOT_STATED` paths, every reason code, every
revert, the forged payload cases from the audit list, the prompt leak test, and
re-reads after a page is republished.
