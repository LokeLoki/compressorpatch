# Hermes Compaction Hardening — patch `v3.4.5`

A drop-in patch for **Hermes Agent**'s context compressor. It fixes the
failure modes that appear in long-running sessions (many compaction
generations): summary snowball, section duplication, hallucinated session
counters, and prompt-cache thrashing from micro-compaction.

Built and field-tested on a single 7M-token session (~80 folds, ~37 h) — every
knob and floor in it is calibrated against measured telemetry from that run,
not guesses.

| | |
|:---|:---|
| **Applies to** | stock Hermes Agent at base commit `71d34bad10` (v0.21.x era) |
| **Patch file** | [`compressorpatch.patch`](https://lokeloki.github.io/compressorpatch/compressorpatch.patch) |
| **sha256** | `4484c7928612fb93278a7664ca35ca4ce7517d8935d1c052357fe537856e44a5` |
| **Scope** | 3 files, 439 insertions / 14 deletions — no new dependencies |
| **Verified** | clean `git apply` at base, compiles, and passes behavior assertion suites against the patched tree |

---

## What changes vs stock Hermes

Stock Hermes runs a head–torso–tail compaction loop that works well for short
sessions and degrades past ~10 folds, because each fold's summary is fed back
into the next one, and micro-compaction edits the middle of the window.

| Area | Stock | Patched (v3.4.5) |
|:---|:---|:---|
| **Handoff size across generations** | Unbounded. The previous summary re-feeds every fold, so summary + appendages snowball (observed ~4K → ~19K chars over 13 generations). | Hard ceiling: `min(15% of context, 10K tokens)`. Evicts oldest entries from low-priority sections first (Session Log → Completed Actions → Anchor Index), never the state sections or verbatim user block. Each eviction leaves a `session_search` pointer. |
| **Summarizer output drift** | Models invent headings (`Next Steps`), nest the old summary inside the new one, or state the same fix in three sections. | Output contract enforced in code: unrequested headings are stripped (body kept), surviving headings canonicalized, cross-section duplicate lines deduped — all fence-aware, so code blocks are never parsed as structure. `ONE HOME PER FACT` in the prompt. |
| **Contradictions & retractions** | An early wrong guess stays recorded as fact forever. | Truth-revision rule: corrections outrank earlier assertions; superseded claims are dropped or tagged `(superseded: <old>)`. |
| **Session counters in the summary** | Hallucinated drift. Token/hour counts live in the protected tail, outside the summarizer's input, so a stale "5.3M tokens" copy-propagates across dozens of folds with nothing able to contradict it. | `⚙ Live session counters` line is **machine-stamped from the session DB at fold time** — exact in/out tokens, message count, elapsed hours. The model never decides these numbers; code overwrites them every fold. |
| **Truncation amnesia** | Cut quotes and dropped bullets leave no trail. | Recovery pointers: truncated quotes and evicted entries carry the `session_id` + `session_search` breadcrumb. |
| **Micro-compaction vs KV cache** | A micro splice can fire the turn after a big fold (two cold re-prefills stacked), the rolling memo can outgrow its own output budget, and passes fire for ~0 net savings — paying a cache break for nothing. | Rolling memo obeys the same structural contract as a batch handoff (3K ceiling); a macro fold resets the micro cadence clock; any micro pass whose **measured** reclaim falls under `micro_compact_min_reclaim_tokens` is rolled back — memo restored, no splice, no cache break, cursor advances. |

### Design principle

Everything **mechanical** (headings, dedup, caps, counters, anchors) is stamped
by deterministic code at commit time; everything **interpretive** (what matters,
what's stale) is left to the summarizer. The model never gets a vote on a
number or an identifier. That split is what makes each fold a verifiable
snapshot instead of a game of telephone.

Why the thresholds are what they are — the measured cache-break economics,
the full 18-pass micro telemetry, the dead-knob list, and the traps for
re-implementers — is written up in **[DESIGN.md](DESIGN.md)**.

---

## How to apply

> Apply from your terminal, not inside a live Hermes chat session. Restart
> Hermes afterward — Python loads these modules once at startup.

```bash
# 1. Download
curl -sL https://lokeloki.github.io/compressorpatch/compressorpatch.patch -o compaction.patch

# 2. From a hermes-agent checkout AT THE BASE COMMIT:
git checkout 71d34bad10
git apply --check compaction.patch   # dry run: RC 0 means it lands cleanly
git apply compaction.patch
```

Plain `git apply` is atomic: it either applies 100% or refuses with zero
changes. **Do not use `--3way`** — conflict markers can land inside multi-line
prompt-string constants, where Python parses them as plain text and the
summarizer contract is silently corrupted with no crash to warn you.

**Already applied an earlier hermes-compaction patch (v3.3–v3.4.4)?** Revert
first — this one patch supersedes them all and is not stackable:

```bash
git checkout -- agent/context_compressor.py agent/micro_compaction.py agent/agent_init.py
git apply compaction.patch
```

**Revert any time:**

```bash
git apply -R compaction.patch
```

Newer stock HEADs may context-reject the patch (plain apply is strict by
design). If `--check` fails, you are not at the base commit — `git checkout
71d34bad10` first, or rebase your checkout onto it.

---

## Recommended `config.yaml`

The patch is **behavior-neutral by default** — stock config keeps stock
economics. Below is the field-tested preset from the 7M-token run, tuned for a
**200K context window** (window breathes in an ~80–135K band between folds).
Stock defaults are in comments; if your context window differs, the
`threshold`/`target_ratio` pair is the first thing to re-check.

```yaml
compression:
  micro_compact: true                           # stock: false
  micro_compact_every_n_turns: 6                # stock: 1; 6 ≈ 2.8 passes per fold runway,
                                                #      each pass buys ~2.5 turns
  micro_compact_min_reclaim_tokens: 2500        # v3.4.4; 0 = off (stock behavior).
                                                #      Field split: passes under ~1.4K saved
                                                #      were noise, 2.5K+ were real.
  context_timeout_seconds: 240                  # stock: 120; headroom under the 300s aux budget
  threshold: 0.87                               # stock: 0.50 — fold at 87% of 200K
  target_ratio: 0.45                            # stock: 0.20
  protect_last_n: 60                            # stock: 20
  proactive_prune_tokens: 40000                 # stock: 0 (off); cheap insurance vs tool-output floods
```

Keys intentionally **not** recommended:

- `micro_compact_defrag_threshold_tokens` — leave the default. The rolling memo
  is capped at 3,000 tokens before this threshold can ever be reached, so any
  value above 3000 is inert.
- Anything you see elsewhere named `hygiene_hard_message_limit` or
  `progress_notices` — those keys do not exist in the compressor.

---

## Changelog (what each version added)

| v | Change |
|:--|:--|
| v3.1 | span-based eviction, fence-aware dedup, lean-heading allow-list |
| v3.2 | verbatim-quote cap accounting, single suffix, hkey alignment |
| v3.3 | canonical heading re-emit, fence-balanced eviction, loop-normalized heading matching |
| v3.4 | micro memo gets the same hardening pipeline as batch handoffs (structure contract, strip/dedup/cap chain, output budget bound) |
| v3.4.1 | handoff ceiling 12K → 10K tokens (aligns with summarizer output budget) |
| v3.4.2 | `⚙` machine-stamped live session counters at every fold |
| v3.4.3 | macro fold resets the micro cadence clock (no fold→micro cache-break stacking) |
| v3.4.4 | measured reclaim floor — micro absorbs only what actually pays for its cache break |
| v3.4.5 | section-cap eviction matches micro-short *and* macro-long headings (external-audit find) |
