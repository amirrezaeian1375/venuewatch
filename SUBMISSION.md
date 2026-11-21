# VenueWatch - submission notes

An Intelligent Contract that reads a vendor's published terms page under GenLayer
consensus and settles whether the page agrees with a dispute policy the caller
pinned, storing the page's own sentence as the evidence.

Two topics, four verdicts, and one comparison that runs in the contract rather
than in a model.

## Live on studionet

Contract: `0x2b6AB6B7dfeB045f1969c23B69d9c6330700c251`
Deploy transaction: `0x7ae93cde434d603adfed9a242aca6b4a5f3a3a508c2f70312afdfa1f7b16049d`
Deployer wallet: `0x1C95Ba0bC5EccCe362eaEb8aD5aD5F60B9604d4f`

Explorer: https://explorer-studio.genlayer.com/address/0x2b6AB6B7dfeB045f1969c23B69d9c6330700c251

The code stored at that address was retrieved with `eth_getTransactionByHash` and
compared against the file in `contracts/venuewatch.py`. After the CRLF
normalisation that Studio applies on the way in, the two are byte identical:
25,891 bytes, md5 `ed1285b23c2b84bb928505fab2a08a73`.

Every transaction below is `FINALIZED` with `result_name` `MAJORITY_AGREE`, sent
from the deployer wallet to the deployed address, and its calldata was decoded to
confirm which method and which arguments it carried.

### The demo: all four verdicts, on four live vendor pages

The policy being checked is the one a company actually carries: California law,
and an arbitration seat in San Francisco if there is arbitration at all.

| check | page | topic | pinned policy | verdict | stated |
|---|---|---|---|---|---|
| V1 | anthropic.com/legal/consumer-terms | GOVERNING_LAW | State of California | **MATCHES** | the State of California |
| V2 | anthropic.com/legal/consumer-terms | ARBITRATION_VENUE | San Francisco, California | **NOT_STATED** | (none) |
| V3 | vercel.com/legal/terms | ARBITRATION_VENUE | San Francisco, California | **MATCHES** | San Francisco, California |
| V4 | cloudflare.com/terms/ | GOVERNING_LAW | Ireland | **DIFFERS** | State of California |
| V5 | stripe.com/legal/ssa | ARBITRATION_VENUE | San Francisco, California | **AMBIGUOUS** | (none) |

| step | transaction |
|---|---|
| open V1 | `0x81aeb3a7f50d4f486e2d66017696a08fa2faa30037d13dbca461d6e5de250157` |
| open V2 | `0x092020c92021bcd3cd78e36477cf0e07f331272948fd86e0ddab56b213539aa6` |
| open V3 | `0xe5fd3e647fcedae2473341ad18687dc2012a7823f28fa953c32fb5a0a51107a3` |
| open V4 | `0x82f2f981363344d0e59df2821b08f2caba8dd644fe37a1a299bc12e0fe7ee696` |
| open V5 | `0xe80b58a62d58ec8828e3cfc14d17060fe3332f6d42fc20100a9da1ed9970d215` |
| resolve V1 | `0x531a3e423f8b3338c13b589efe1c7e08245677169f43cad4f3454dedb14f5795` |
| resolve V2 | `0xde5a03620178b8f7ecb1832774904ebca99aedda1600c53c2d8b0446d1a4401a` |
| resolve V3 | `0xe395bb797b63f6194bcae5580d89a2dc34bfa4c0fd7d7bef1be7460b30f62ef8` |
| resolve V4 | `0x27edca311f0066b2e9c2da60e4abd331fe4df233b23808476e34861a330de03d` |
| resolve V5 | `0xe533ec8f26f52c9a5ce29d6e74963d064d0c65198bcdda44fbabf442b7cbf235` |

`count()` returns 5 and `list_checks()` returns all five records with the
verdicts above.

### What the chain stored, verbatim

V1, `GOVERNING_LAW`, MATCHES:

> Our Terms will be governed by, and construed and interpreted in accordance with, the laws of the State of California without giving effect to conflict of law principles.

V3, `ARBITRATION_VENUE`, MATCHES:

> The arbitration shall take place in San Francisco, California or at such other venue (and pursuant to such procedures) as is mutually agreed upon.

V4, `GOVERNING_LAW`, DIFFERS, with the policy pinned to Ireland:

> Governing Law This Agreement will be governed by the laws of the State of California without regard to conflict of law principles.

V5, `ARBITRATION_VENUE`, AMBIGUOUS:

> (a) Arbitration will be held in San Francisco, California.

V5 is the interesting one. Stripe publishes one global agreement whose regional
terms name five different seats, San Francisco, Dublin, London, Sydney, and
Toronto. A check that could only answer yes or no would have to pick one and be
wrong for four regions. The contract settles `AMBIGUOUS` and stores one of the
sentences that proves it.

V2 is the other one worth reading carefully. Anthropic's consumer terms contain
no arbitration clause at all, so disputes go to court. The contract settles
`NOT_STATED` and it does so **without calling a model**, because the evidence
region it builds for that topic is empty and every validator rebuilds the same
empty region from its own fetch. Silence is a deterministic fact here, not an
opinion, and it can never be promoted into a pass.

## Why this belongs in Intelligent Contracts

**Real consensus logic, not a stored model answer.** Three separate things are
decided by code that every participant runs: which sentences constitute the
evidence, whether the transcribed phrase is bound to that evidence, and which of
the four verdicts the comparison produces. The model contributes one
transcription. Two validators that isolate the phrase differently ("the State of
California" and "California") still agree, because both bind their phrase to
their own fetch and both reach MATCHES. There is a test for exactly that case.

**A clear state design.** One record per page, topic, and policy string, with
ids V1, V2 and a cap of 2000. Opening the same triple twice returns the same id,
so a check is a durable object rather than an event. States are PENDING, SETTLED,
UNRESOLVED. UNRESOLVED always carries a reason and never carries a verdict, so a
failed read is structurally incapable of looking like a pass. Re-resolving a
settled check re-reads the page and bumps `version`, which is how a vendor that
moves its seat becomes visible instead of being silently overwritten.

**Thoughtful validators and equivalence checks.** The validator does not inspect
a flag. It re-fetches, rebuilds the evidence region, re-grounds the quote,
re-binds the phrase to the quote, re-runs the whole read including its own model
call, and requires an identical state, verdict, and reason. A claimed NOT_STATED
is only certified when the validator's own region is empty too. A validator that
cannot read the page rejects a settled claim rather than abstaining into
acceptance.

**Use beyond a demo.** Vendor review is a recurring, quantifiable task: every new
supplier, every renewal, every terms republish. The stored record is the artifact
a procurement team actually needs, because it carries the sentence, the
jurisdiction as printed, the policy it was checked against, and the moment it was
checked. `rules()` publishes the marker lists and every limit, so a reviewer can
see the selection rule without reading the source.

**Readable source and real tests.** 695 lines including a long module docstring
that states the honest limits. 59 direct VM tests, linter clean at 9 methods
(7 view, 2 write), and no test reaches the network: every page is a hermetic
fixture shaped like the real vendor pages this was built against.

## How this differs from work already published

| published contract | what it does | what VenueWatch does differently |
|---|---|---|
| genlayer-sweep | proves a clause is absent, with verbatim quotes and a contest window | stores a named jurisdiction and compares it against a pinned policy string; absence is one of four verdicts, not the goal |
| terms-audit-desk | multi page documents against a checklist of yes/no criteria, with coverage | one page, one topic, one value; the answer is a jurisdiction, not a boolean |
| genlayer-clause-checker | three fixed yes/no clarity questions | caller supplied policy string, caller supplied page, two topics |
| license-compatibility-gate | extracts an SPDX id and applies a deterministic policy bitmask | extracts a jurisdiction phrase and applies word sequence containment; the policy is the caller's, not a table in the contract |
| LifecycleQuorum | cross checks a lifecycle record between two sources | single pinned source, and the value is prose on a legal page rather than a structured record |

The distinguishing mechanism is the middle column of the design: a **named value
extracted from prose**, bound word for word to a quoted sentence, then compared
against a **policy the caller pinned**. Presence checks have no way to express
`AMBIGUOUS`, and `AMBIGUOUS` is the verdict that ends most real vendor reviews.

The evidence region is also built differently. The contract splits the page into
sentences and keeps only those carrying the topic's published marker words, so a
model reads eight sentences rather than a hundred thousand characters, and
neither the model nor the caller chooses what is read. On stripe.com/legal/ssa,
which renders to 127,928 characters of text, the arbitration region is eight
sentences and 661 characters.

## The audit, rule by rule

Every rejection this account has received was turned into a standing check. This
is where each one lands.

**Rule 1, evidence must prove the claim and not merely exist.** A stored phrase
is checked three ways: the quoted sentence must exist inside the evidence region
this reader built, it must carry the topic marker, and the phrase must be printed
inside it word for word in order. A quoted heading such as "Governing law and
exclusive jurisdiction." grounds and carries the marker, and it is still
rejected, because it prints no jurisdiction. That was a real hole in the first
draft of this contract: the phrase was bound to the evidence region as a whole
rather than to the quoted sentence, so a worthless quote could ride along on a
phrase taken from a different sentence. It is closed, and
`test_the_validator_rejects_a_quote_that_proves_nothing` is the test that closes
it.

**Rule 2, the validator recomputes and never trusts.** Nothing in the leader's
payload is believed. The validator rebuilds the region, re-grounds, re-binds,
re-reads, and compares state, verdict, and reason. No boolean in calldata stands
between a lie and acceptance, because there is no boolean in calldata.

**Rule 3, a failed check must not silently become a pass.** Every failure path
lands in UNRESOLVED with a reason from a published list, and `matches(id)` is
true only for a settled MATCHES. A second resolve that fails clears the previous
quote and phrase rather than leaving stale evidence attached to a fresh
`checked_at`.

**Rule 4, test the attack and not only the happy path.** The suite includes a
quote that is on the page but belongs to a different topic, a quote on the right
topic that prints a different jurisdiction than the one stored, an invented
quote, a phrase too long or too short to be a jurisdiction, a heading offered as
proof, a forged verdict, a forged silence claim on a page that speaks, a forged
ambiguous claim that cannot be grounded, an unresolved claim the validator's own
read settles, and a validator that cannot fetch at all.

**Rule 5, prove it on the real chain.** Ten transactions above, all
FINALIZED/MAJORITY_AGREE, all four verdicts, on four live vendor pages, with the
deployed bytecode diffed against the tested file.

**Rule 6, read it as a skeptic.** What can a malicious leader do, and who catches
it: it can name a jurisdiction the page does not print, and every validator
catches it by binding the phrase to its own fetch. It can attach a real but
worthless quote, and the phrase binding catches it. It can claim silence on a
page that speaks, and the validator's own non-empty region catches it. What
happens when the page changes after the record is stored: the record stays true
about `checked_at`, and a re-resolve produces a new verdict, a new phrase, and a
higher version; there is a test where the validator reads a republished page and
rejects the old claim. Where could a correct looking answer carry worthless
evidence: only in the quote, and the quote is now required to print the stored
phrase. Which check could report a pass without proof: none, because MATCHES
requires a phrase, a quote, a grounding, a marker, and a containment that the
contract computes itself.

## Limits, stated plainly

* The evidence region is selected by a fixed marker list per topic. A page that
  states its governing law without `governed by` or `governing law`, or its
  arbitration seat without `arbitrat` plus a place marker, settles NOT_STATED.
  Both lists are published by `rules()`.
* Matching is word sequence containment, so a short policy string can sit inside
  several jurisdictions: "Ireland" also matches a page that prints "Northern
  Ireland". Pin the full name when the distinction matters.
* The caller pins the URL, so the caller owns the consequence of pinning a page
  that is not the vendor's real terms.
* One page, one topic, one policy string per check. A vendor with separate
  consumer and commercial terms is two checks.
* A verdict is a claim about the page at `checked_at`. Vendors republish.
* Legal effect is out of scope. The contract reports what a page prints and
  whether it satisfies a string the caller pinned. It does not advise, and the
  prompt tells the model not to.

## Reproducing

```
pip install -r requirements.txt
python -m pytest tests/direct -q          # 59 passed
genvm-lint check contracts/venuewatch.py  # Lint passed, Validation passed, 9 methods
```

Reading the live contract needs no key:

```
readContract(address, "rules", [])
readContract(address, "count", [])
readContract(address, "list_checks", [])
readContract(address, "get_check", ["V5"])
```
