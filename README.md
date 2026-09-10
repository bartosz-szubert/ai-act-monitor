# AI Act & Poland Implementation Monitor

An n8n agent that watches the EU AI Act (Regulation (EU) 2024/1689), its implementing acts, and its implementation in Poland, and sends me a plain-language weekly email — no legal jargon, no digging through EUR-Lex myself.

**Status: running weekly.** Built and verified end-to-end against live data on 4 September 2026 — node by node first, then repeated full runs while the fixes below were applied. Several of those fixes came from reading the code against its own description, not from running it. What follows is what actually happened while building it, not just the intended design.

## The problem

The AI Act isn't one fixed text. It's a regulation, a set of amendments (the Digital Omnibus), a growing pile of implementing acts and guidelines from the European Commission, and a parallel national law in Poland with its own timeline. Deadlines get postponed. New guidance appears. A national supervisory body gets created in stages. Missing any of this quietly is expensive if you work anywhere near AI compliance.

Reading five legal-tracker sites a week isn't realistic, and the sources that move fastest — law-firm blogs, industry newsletters — are exactly the ones I don't want to treat as ground truth without a second, official source confirming them.

## What it does

Every Saturday at 09:00 it checks three things — the AI Act's own timeline, new implementing acts and guidance, and Poland's implementation of the law — and sends one email with the full current status of all of them, in plain language. Unlike my MotoGP agent, it does **not** stay silent when nothing changed: it always sends the full digest, with a "what changed this week" section at the top when there is something to report, and a separate block of unverified signals at the bottom when the Poland branch has found something it cannot vouch for.

## How it works

1. **Schedule trigger** — weekly, Saturday 09:00 Europe/Warsaw
2. **Read state** — previous known status from a JSON file on disk
3. **Three independent branches**, each with its own Perplexity Sonar query, domain filter and JSON schema:
   - **Milestones** — checks the 7 known AI Act application dates against official and industry sources
   - **Implementing acts** — open-ended discovery of new executive acts, delegated acts and guidelines
   - **Poland implementation** — checks the known implementation items (law entry into force, KRiBSI formation, the national regulatory sandbox) and surfaces anything else it finds as an unverified signal, without adding it to the tracked state
4. **Merge and ratchet**, one per branch — see below
5. **Merge node** (n8n's built-in Merge, append mode) — physically joins the three branches into one graph path. This turned out to be necessary, not optional: see "What went wrong"
6. **Combine** all three sections into one state object
7. **Write state** — overwrite the JSON file, once, after all three branches have merged
8. **Build HTML email** — plain template, no model call at this stage
9. **Send** — Gmail

## The rule that makes it useful

Every tracked item carries a status, and for the two closed-list categories (milestones, Poland implementation) **only an official source can confirm a date change**:

| Source tier | Domains | Can set... |
|---|---|---|
| **Official** | eur-lex.europa.eu, digital-strategy.ec.europa.eu, gov.pl, uodo.gov.pl, sejm.gov.pl, eli.gov.pl | confirmed change (`przesuniete` / `zaktualizowany`) |
| **Industry** | artificialintelligenceact.eu, iapp.org, euractiv.com, lexology.com | early signal only (`zagrozony_zmiana`) |

The table is the union across branches; each branch passes only the subset that makes sense for it to `search_domain_filter`.

This is the same principle as my MotoGP agent's confirmed/unconfirmed/rumour ladder, adapted to two tiers instead of three, because regulatory status doesn't really have a "rumour" level — either a law firm is speculating about a delay, or the Official Journal has published one.

For implementing acts (an open list), the status can only move forward — `zapowiedziany` (announced) → `w_konsultacjach` (in consultation) → `przyjety_opublikowany` (adopted and published) — and only an official-source hit can push it to the final stage. New acts are matched against known ones by a normalized key derived from the title: lowercased, diacritics stripped, filler words removed, first six words. That is the same `normalizeKey()` pattern as in the MotoGP agent, for the same reason: don't trust a language model to produce a stable identifier across runs.

## Decisions and trade-offs

**Why "always send" instead of "only on change".** The MotoGP agent stays silent by default and only speaks up when something moves. This one always sends the full weekly digest. The volume here is far lower — a handful of tracked items, not dozens of race weekends — so a standing digest is more useful than a notification stream, and it doubles as a sanity check that the pipeline is still alive.

**Why one shared state file with three independent branches instead of three files.** Simpler to back up and to inspect, at the cost of having to be careful about write ordering. The three branches read the same in-memory state, parsed once at the start of the run, and process independently; the physical write to disk happens exactly once, after all three have finished, so that one branch's write cannot clobber another's. This replaced an earlier design with three separate physical read/write cycles — same separation of concerns, without the race condition.

**Why static descriptions for known items and model-generated text only for new ones.** The plain-language requirement is easiest to guarantee for the 12 known items (7 milestones + 5 Poland implementation items) by writing each description once, by hand, and never letting the model touch it again. Only genuinely new items get a model-generated `opis_prostym_jezykiem`, with an explicit instruction to avoid jargon and keep it to two sentences. That limits the blast radius of the model drifting back into formal language to the cases where there is no alternative.

**Why Perplexity Sonar again.** Same reasoning as MotoGP: `response_format: json_schema` and `search_domain_filter` enforce structure and source quality at the API level, which is more reliable than a well-worded prompt. A broader model comparison done for an earlier agent found that at this call volume the cost differences between models are negligible, so reliability decided it, not price.

## What went wrong

**`Extract from File` nests its output under `data`, not at the root.** The first thing that broke. Every downstream node reading the previous state (`state.kamienie_milowe`, `$('Parse State').first().json.akty_wykonawcze`) had to become `state.data.kamienie_milowe`, `$('Parse State').first().json.data.akty_wykonawcze`. The node's `fromJson` operation puts the parsed content under a destination key, default `data`, rather than merging it into the item root. Worth checking for rather than assuming, on any workflow that parses a file this way.

**A broad "official" domain let community content in.** `ec.europa.eu` was on the official-source list to cover `digital-strategy.ec.europa.eu`. But the validation used substring matching, so `futurium.ec.europa.eu` — the Commission's community forum, not an official publication channel — matched too, and the implementing-acts branch picked up a community playbook and a "how to navigate" explainer as if they were legal instruments.

The fix was to drop the bare parent domain and keep only the specific subdomain. **The more useful part of this story is that the first fix only landed in one of the two places that used the list**: the implementing-acts branch was corrected, the milestones branch still carried `ec.europa.eu` in both its search filter and its official-source check — the branch where a bad "official" match does the most damage, because it can move a legal deadline. A later line-by-line review caught it. Lesson recorded twice over: a domain filter meant to signal "official" has to name the actual publication channel, and a fix to a duplicated list has to be applied to every copy of it.

**The model can correctly say something doesn't qualify and include it anyway.** This happened three times in one testing session, in three different shapes:

- A synthetic "no results found" item invented and placed in the `tytul` field, instead of the empty array the schema allowed.
- A GDPR/privacy statement classified as `przyjety_opublikowany`, while its own `opis_prostym_jezykiem` field said, in Polish, that it was not a standalone implementing act.
- A general-information item whose own `status` field read "informacja ogólna, bez nowego formalnego aktu" — a self-disqualifying sentence, sitting in the status field, on an item that was still returned.

None of this is a schema violation. The structure was valid every time and mostly on-topic. It is a categorization failure that structure alone cannot catch, because "is this genuinely a new implementing act" is a judgment call, not a type constraint. `response_format: json_schema` guarantees shape; it does not guarantee the model applies its own stated criteria to its own output.

The fix was a code-level keyword filter, applied after parsing and before the merge, looking for self-disqualifying phrases. It reads slightly differently in the two branches, and the difference is deliberate: in the implementing-acts branch `status` is a schema enum, so nothing can hide there and the filter only checks title and description; in the Poland branch `status` was at that point a free-form string, which is exactly why a self-disqualifying sentence ended up in it, so there the filter checks that field too. It is an enum now, for reasons the next-but-one section explains. Worth knowing which of your fields are enum-constrained before deciding where to look for junk.

This is a blunt, whack-a-mole safeguard rather than a real solution — it will miss phrasings I haven't seen yet. Accepted as a reasonable trade-off for a weekly digest a human reads and can sanity-check. Worth revisiting if the false-positive rate climbs once it runs unattended for a few months.

**`$('Node Name').item` needs a real graph connection, and breaks across a Merge node.** Two failures in sequence. First, the three branches were built as dead-end parallel paths, joined only by `$('Node Name')` expression references inside the final combine step, on the assumption that any node which had executed in the run would be resolvable that way. It isn't — n8n requires the referenced node to be an actual ancestor in the connections graph, even indirectly. Fixed by adding a Merge node (append mode, three inputs) that physically joins the branches before the combine step.

That surfaced the second failure. `.item` relies on n8n's paired-item tracking to know which upstream item corresponds to the one being processed. After an append-mode Merge of three unrelated branches that pairing is meaningless, and `.item` failed with `Cannot read properties of undefined (reading 'error')` — an error message that points nowhere near the actual cause. Fixed by switching to `.first()`, which takes the node's first (and here, only) output item without needing a pairing relationship. Every cross-node reference in the workflow now uses `.first()`, not just the one that broke.

**Two bugs that testing would not have found, only review would.** Both were caught by reading the code against its own description rather than by running it:

- *New Poland items had no identity.* Items discovered outside the known list were stored with `id: null` and appended to the state. The search window is a month while the schedule is weekly, so the same new regulation would be found again the following Saturday and appended a second time, then a third. Fixed first by giving discovered items a `normalizeKey()`-derived id — the same mechanism the implementing-acts branch already used — and skipping any whose key was already in the state. That held, but only against identical titles, which is what pushed the redesign described next.
- *A standing rumour was reported as new every week.* When an industry source flagged a possible change, the item was marked as changed on every run for as long as that source stayed inside the search window, so the "what changed this week" section would repeat the same line for a month. Fixed by marking a change only when the status or the note actually differs from what is already stored. This is the same failure the MotoGP agent's ratchet exists to prevent — a notification that fires on noise trains you to ignore it — arriving in a different disguise.

**Where prompt and schema stopped being enough, and the design had to change.** The Poland branch does two things: it checks a closed list of known items, and it discovers new ones. The first half is reliable. The second half kept returning things that are not implementation acts at all — a call for applications, a set of draft checklists, a ministerial announcement of a science programme — and doing so in ways no filter could catch:

- One item's own `opis_prostym_jezykiem` read "wygląda na materiał roboczy, nie formalne obwieszczenie lub akt" — a self-disqualification, in a phrasing my keyword list did not have. Adding that phrasing would only postpone the next one.
- Another described the same fact as an already-tracked item, under a different title. No key derivation catches that, because the key is derived from the title and the titles are genuinely different strings.
- A third arrived through a self-contradiction split across two fields of one response: the item appeared in `nowe_elementy` while a note in the `items` array of the same response said it was not a new formal action. The filter reads the item, not its neighbour, so it cannot see the contradiction.

Two structural changes instead of a longer blocklist. First, `status` in this branch became a schema enum, the same as in the implementing-acts branch. It had been a free-form string, which is precisely why sentences like "informacja ogólna, bez nowego formalnego aktu" could live in it. Second, and more important: **discovered items no longer enter the tracked state at all.** They go into the email as a separate, clearly labelled block of unverified signals. If one turns out to matter, it gets added by hand as a permanent item with a written description — the same rule the README already applied to everything else.

That removed a whole class of failures at once rather than patching them one at a time: nothing can duplicate a curated item, nothing accumulates, and the model can no longer promote a press release into a tracked legal fact. The general lesson: a schema can guarantee shape and an enum can constrain a field, but neither can make a judgment call. When the model's output is a judgment, keep it out of the data you trust and put it where a human sees it labelled as what it is.

**Gmail OAuth2 token needed reconnecting mid-session.** Unrelated to everything above: the credential's refresh token had expired. Reconnected through Credentials → the credential → Reconnect. Worth a periodic check, since a token expiring silently means no mail goes out until somebody notices.

## A note on the seed data

The state file ships with real facts checked by web search while building this agent, not recalled from a model's training data:

- The AI Act's high-risk deadlines were pushed back by the Digital Omnibus, **Regulation (EU) 2026/1744**, published in the Official Journal on 24 July 2026 and in force since 27 July 2026: standalone high-risk systems (Annex III) moved from 2 August 2026 to **2 December 2027**, and product-embedded systems (Annex I) to **2 August 2028**.
- Poland's AI law — *ustawa z 3 lipca 2026 r. o systemach sztucznej inteligencji*, Dz.U. 2026 poz. 1003 — is not a pending bill. It has been **in force since 11 August 2026**, with a second wave of chapters (opinions and explanations, inspections, proceedings, administrative fines) applying from **28 October 2026**. The national supervisory body, KRiBSI, must have a chair within two months of entry into force (by 11 October 2026) and a full commission within three (by 11 November 2026).

That is a good demonstration of why this agent exists. The picture moved substantially in the eight months before the agent was even built, and every one of those dates would have been wrong if I had trusted a model's recall instead of checking.

## Known rough edges

Left in deliberately, so that this file matches what actually runs:

- **The implementing-acts list grows without bound.** Nothing ages items out of the state, so the digest gets longer over time. Fine for now, at a handful of acts; it will need a retention rule eventually.
- **The keyword filter is a blocklist.** It catches the phrasings I have seen. A new way of saying "this doesn't really qualify" will get through until I add it. It matters less now that discovered items are only signals, not tracked state.
- **A signal repeats while it stays in the search window.** Signals are not remembered between runs, so the same lead can appear in the digest for up to a month. Deliberate: they are labelled as current leads rather than news, and not storing them keeps the state file to curated facts only.
- **No retry on the three HTTP calls.** A failed request means that branch reports nothing that week and the state is not advanced, which is the safe direction.
- **`search_recency_filter` is set to a month against a weekly schedule.** Deliberate overlap so nothing falls between runs; the ratchet and the key matching absorb the repeats.

## Running it yourself

The workflow is exported without personal identifiers or credential keys. It arrives **inactive**.

1. **Import** `ai-act-monitor.json` into n8n (Workflows → Import from File).
2. **Attach two credentials.** The workflow references them by name, not by key, and no secret is stored in this file:
   - *Perplexity API* → the three `HTTP Request - Sonar ...` nodes
   - *Gmail account* → the `Gmail - Wyslij Podsumowanie` node
3. **Set the recipient.** Replace `TWOJ_EMAIL@example.com` in the Gmail node with your own address.
4. **Create the state file** at `/home/node/.n8n-files/ai_act_monitor_state.json` using the contents of `ai_act_monitor_state_seed.json` in this repository. The read node does not bootstrap it.
5. **Timezone** is already `Europe/Warsaw` in the workflow settings; check it matches what you want from the Saturday 09:00 trigger.
6. **Run it node by node first, not the whole workflow at once.** That is exactly what surfaced the failures above — none of them would have been obvious from a single full-workflow run with a generic error.
7. **Activate only after a clean end-to-end run.**

## Stack

Hostinger VPS (KVM) · Docker · n8n self-hosted (v2.33.6) · Perplexity Sonar API · Gmail (OAuth2) · state as JSON on disk

---

*Sibling repositories: [motogp-calendar-monitor](https://github.com/bartosz-szubert/motogp-calendar-monitor) — the same ratchet idea, applied to a race calendar instead of EU regulation. [human-gated-agent](https://github.com/bartosz-szubert/human-gated-agent) — the pattern both of these were extracted into.*
