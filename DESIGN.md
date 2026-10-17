# Design notes — the economics behind the patch

This repo's README says *what* the patch changes. This file records *why the
numbers are what they are* — the parts a future harness re-implementer is most
likely to get wrong or optimize away. Everything here is measured from one
7,274,774-token session (~80 macro folds, 18 micro passes, 36.9 h, 200K window,
`threshold 0.87`).

---

## 1. Doctrine: claims vs receipts

A summarizer model's output is a **claim**. Code-written facts are
**receipts**. Every fact that must survive many generations of
summarize-the-summary gets mechanically witnessed:

| Fact | Receipt mechanism |
|:---|:---|
| What the user actually said | verbatim user-message block in the protected tail |
| Session scale (tokens/messages/hours) | `⚙` line stamped from the session DB at fold time, regex-wiped and re-stamped every fold (self-healing even if a model mangles it mid-run) |
| Identifiers (paths, commits, PRs) | Anchor Index, mechanically extracted, never paraphrased |
| Retracted claims | `(superseded: <old>)` quarantine — the final position is the only fact |
| Section structure | code-side strip/canonicalize/dedup/cap chain, not prompt trust |

The model keeps exactly one job: deciding **what matters**. It never gets a
vote on a number or an identifier. The consequence: every fold is a dated,
notarized snapshot — "true as of stamp time" — instead of prose that silently
rots. A stale copy of `5.3M tokens` persisted ~35 folds before this fix,
because *nothing inside the summarizer's input window could contradict it*.
Facts the fold input can't see cannot be self-corrected; stamp them from
outside. That is the whole v3.4.2 lesson, generalized.

## 2. The cache-break economy (why floors exist at all)

Any edit to the middle of the window — a micro splice, a fold — invalidates the
provider's prompt-cache prefix beyond the edit point. Each micro pass therefore
pays a **fixed toll**: one cold re-prefill of the post-splice tail (tens of
seconds at a 200K window; observed 50–65 s per pass including the aux merge).
The *savings* scale with what the merge reclaims, but the *cost is per-pass*.
So every pass must be judged as: `measured reclaim vs one tail re-prefill`.

This is also why v3.4.3 exists: a macro fold already paid a re-prefill; a
micro splice on the next turn stacked a second cold prefill. Fold resets the
micro cadence clock so the two cache breaks can never be adjacent.

## 3. Measured: exchange size does NOT predict savings

The trap we walked into: "gate micro passes by *exchange size* — small
exchanges must save little." The telemetry says the opposite. All 18 passes,
`saved = tokens_before − tokens_after` (negative = the pass GREW the window):

| time | exchange tok | saved tok |
|:---|---:|---:|
| 02:47 | 6,738 | **43,413** |
| 03:26 | 3,073 | 4,783 |
| 11:21 | 1,935 | 3,783 |
| 12:09 | 4 | **−2,032** |
| 13:03 | 4 | **−1,374** |
| 16:12 | **771** | **9,972** |
| 16:38 | 519 | 694 |
| 17:02 | **175** | **8,971** |
| 18:18 | **346** | **7,100** |
| 18:40 | 225 | 1,409 |
| 20:06 | 255 | 510 |
| 20:28 | 61 | 234 |
| 21:14 | **956** | **9,149** |
| 22:29 | 4 | 381 |
| 23:01 | 433 | 1,098 |
| 23:15 | 822 | 2,638 |
| 00:56 | 474 | **6,898** |

A 175-token exchange reclaimed 8,971 — because the merge collapses accumulated
tool-result chains and supersedes stale marker rows across the *whole memo*,
not just the new exchange. A 4-token exchange grew the window by 2,032. An
exchange-size floor would have skipped most of the seven best passes. **The
verdict must be measured after the merge, never predicted before it.** That is
why v3.4.4 gates on post-merge `tokens_delta` and rolls back (memo restored,
no splice, no cache break, cursor advances past the crumb — no re-summarize
loop).

The savings split has a clean valley: ≤1,409 were noise/negative, ≥2,638 were
real. Hence `micro_compact_min_reclaim_tokens: 2500` (field floor at the time;
the running box now operates at **2700**, set alongside v3.4.7's gate — the
valley's exact depth is harness-specific, as always). On a different harness
or window, re-measure the valley — don't copy the constant; copy the method.

v3.4.7 adds the one exception to "never predict before measuring": the
**pre-generation gate**, which skips a window only when the *best case* —
whole exchange absorbed AND the summary shrinking to nothing — still can't
clear the floor. That bound is one-sided and provable, so it can never skip a
run that could pass the floor; genuine winners (a fat summary collapsing to a
thin memo) are never predicted dead and still generate. Everything cheaper
than the memo call is free to gate; the memo call itself is where the measured
verdict resumes.

v3.4.8 widens the gate's sight: the bound is computed over the **whole pending
backlog**, not just the next exchange — still one-sided (a bigger window is a
more generous bound, so provability holds). On a skip the cursor is now
**held**, so a streak of sub-floor exchanges accumulates; when the accumulated
backlog clears the floor it generates **once** and splices **once** — one cache
break for the streak instead of one per cadence tick. The window's user rows
are re-inserted verbatim after the marker, per the micro contract.

## 4. Cadence and interactions

- **Every-N-turns** is a floor on pass frequency, not a trigger: demand-gating
  (is there an unabsorbed finished exchange?) does most of the work at human
  conversation pace. Field rows were 32–75 minutes apart at chat pace — the
  cadence knob only bites during dense tool loops. 6 (of stock 1) bought ~2.8
  passes per fold-runway at ~2.5 turns each and stopped fold-adjacent collisions
  when paired with the v3.4.3 reset. 6 vs 7 measured as noise.
- **Rolling-memo ceiling (3,000 tok, `out_budget // 2` bound) is the real
  governor** of memo growth. A defrag threshold above the ceiling is *inert
  config* — it can never fire. Several audits keep recommending defrag values
  (5,000 / 20,000) that the ceiling makes unreachable.
- **Proactive prune (stock layer) fired zero times** across the session:
  micro + fold evicted material before prune eligibility. Three stock
  layers → two doing everything. Don't add more layers without telemetry
  showing a gap.
- **Occupancy above the fold trigger is not a bug.** Late-session rows show
  124–160% occupancy held without folding: rearm hysteresis + tail protection
  queue the fold; micro grazing keeps the re-prefill cheap until it lands.

## 5. Traps for a new harness

1. A length-truncated merge must be **discarded**: storing a half-rewrite
   permanently stalls absorption. Bound the memo ceiling under half the merge
   call's output budget so truncation is structurally impossible at steady state.
2. Eviction must be **span-based and fence-aware**: an entry spans to the next
   entry line (sub-bullets drop with their parent); a drop span with an odd
   fence count extends to the closing fence, never bisecting a code block.
3. Heading matching must handle **short and long heading variants** — the cap
   pass used the long canonical prefix and silently could not evict the
   micro contract's short `## Anchor Index` (v3.4.5, external-audit find).
   v3.4.8 fixes it at the root: a canonicalization map folds the short variants
   to the exact canonical headings *before* dedup and cap run, so Pass 1 merges
   them and the cap's prefix match can never miss.
4. Cap-before-commit for anything that re-feeds a future prompt: the handoff
   snowballs precisely because it is the input of the next generation
   (observed 4K → 19K chars over 13 generations, no ceiling).
5. Prompt injection: the exchange being summarized is **data, never
   instructions** — say so in the merge prompt, and keep the mechanical
   contract in code so injected headings don't become structure.
