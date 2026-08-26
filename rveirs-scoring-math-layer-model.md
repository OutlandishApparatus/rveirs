# RVEIRS Scoring Math — The Layer Model
*Replaces eddy_1 / eddy_2 / eddy_3 / depth_1–3. Last updated: August 11, 2026.*

---

## Contents
1. Plain language — what this measures and why
2. The layer model, formally
3. Worked example
4. Why this preserves privacy
5. What's fixed math vs. what's a deployment choice
6. Data flow / architecture
7. Terminology migration (old names → new)
8. Open questions

---

## 1. Plain Language

LM3 applies an ammeter at one point in the circuit. RVEIRS installs a network of real-time weather stations. This document is the schematic for how those stations actually talk to each other.

Every participant — a business, eventually a household — can answer one question about themselves: *of the money that left me, where did it go?* That answer, broken into zip codes or zones, is **Layer 1**. It's self-reportable. No network required. A diner knows what it paid its landlord and its produce supplier without needing anyone else's cooperation.

But money doesn't stop moving once it leaves the diner. The produce supplier spends *their* income too — on their own rent, their own staff, their own suppliers. Some of that stays local. Some doesn't. If the diner's local produce supplier is itself owned in another state and buys from national distributors, the diner's "locally spent" dollar might not actually be doing much local work at all — it just took one extra hop to leave.

**Layer 2** is what you get by asking: for everyone I paid, where did *they* send it? Layer 3 asks the same question one hop further, and so on. Each layer traces the dollar deeper into the regional economy. Every layer, something leaks out — to somewhere far away, or to somewhere untracked entirely. That leakage is not an error. It's the signal. A community where money bounces five hops before leaking is a fundamentally different economy than one where it leaves after one.

Two numbers come out of this, and they mean different things:

- **The score** — how much local economic activity a dollar generated before it fully left the region. This is *not* a percentage. It can be small (money leaves fast) or it can run into the double digits (money circulates for a long time before it's gone). Temperature, not grade.
- **Depth** — how much of the money we're tracing actually had a known destination at each step, versus disappearing into "we don't know." This *is* bounded, 0 to 1. It's the trust rating on the score above it. A high score built on low depth is a guess dressed up as a number.

**What gets shown by default:** Layer 1 — the business's own outbound flow to local zip codes. It needs no network, no cooperation from anyone else, and it's the one number a business can actually act on directly, because it's simply *where they chose to spend their own money.* Layers 2 and beyond are the deeper picture, available once enough of the network participates, but they're the "zoom in" view — not the front door.

---

## 2. The Layer Model, Formally

### Layer 0 — Self

L0 is a reference point, not a score. For now, it's just the entity's home zip code. (Flagged for later: for entities with split or absentee ownership, L0 could become a distribution across zips — e.g., a business 60% owned by a local resident and 40% by an out-of-state investor. Not needed yet. Worth remembering it's compatible with the model when it is.)

### Layer 1 — Contacts

Every entity reports its own outflow as fractions of total spend, by destination:

```
local_1, close_1, far_1, void_1        (four numbers, sum to 1.0)

local_1 = spend to zips in the "local" zone ÷ total spend
close_1 = spend to zips in the "close" zone ÷ total spend
far_1   = spend to any other known zip ÷ total spend
void_1  = spend to unknown / blank destination ÷ total spend
```

This is the exact computation the existing `eddy-calculator.html` already performs. Nothing about L1 changes — it's the same tool, just correctly understood as the *first* layer of a longer chain rather than the whole system.

At the raw, most granular level, L1 is actually a list of specific counterparties and their zips — `{0.04 → cook-bob, zip 67xxx}, {0.06 → manager-lisa, zip 68xxx}`, etc. The four-number local/close/far/void vector above is what you get when you classify that raw list against a zone definition. Both forms matter — see §6.

### Layer n (n > 1) — Recursion

For any entity A with confirmed, participating contacts B (where `ratio(A→B)` is A's fraction of spend going to B):

```
local_n(A) = Σ over confirmed contacts B:  ratio(A→B) × local_(n-1)(B)
close_n(A) = Σ over confirmed contacts B:  ratio(A→B) × close_(n-1)(B)
far_n(A)   = Σ over confirmed contacts B:  ratio(A→B) × far_(n-1)(B)

void_n(A)  = [ Σ over confirmed contacts B: ratio(A→B) × void_(n-1)(B) ]
           + [ Σ over UNCONFIRMED contacts: their share of A's outflow ]
```

In words: A's layer-n local credit is the sum, over everyone A pays, of "how much A paid them" times "how local *their* layer-(n-1) spending was." Anyone A pays who isn't a confirmed, reporting node contributes straight to void — we can't trace a dollar past a contact who isn't measuring their own flow.

This always sums to 1.0 by construction: `local_n + close_n + far_n + void_n = 1.0` at every layer, for every entity, automatically. (Proof is arithmetic, not assumption — see §4.)

### Depth

```
depth_n = local_n + close_n + far_n  =  1 − void_n
```

One formula. Same formula at every layer. This retires depth_1/depth_2/depth_3 as three separately-described scripts — they were always the same computation, just never stated that way.

### The score

Across however many layers have been computed (call that N):

```
raw_score(N)     = Σ (n = 1 to N) of local_n
total_depth(N)   = Σ (n = 1 to N) of depth_n
corrected_score  = raw_score(N) / total_depth(N)
```

`raw_score` is the "temperature" number — unbounded upward in principle, but in practice it converges, because something leaks to void at every layer (see the convergence note in §4). It answers "how much local economic activity did this dollar generate before it was gone."

`corrected_score` is bounded 0–1 — since `local_n ≤ depth_n` at every layer, `raw_score` can never exceed `total_depth`. It answers "of the money we could actually trace, how much of it stayed local." This is the trustworthy companion number, and it should always be reported alongside `raw_score` and N (how many layers deep the computation went) — a score with no depth attached is not a real number, it's a guess with decoration.

---

## 3. Worked Example

Starting from the existing reference case (Main Street Diner), extended one layer deeper.

**Layer 1 — Main Street Diner's own report** (unchanged from the existing spec):

```
Employee wages          $4,000   72901  → local
Commercial rent         $1,500   72901  → local
Local produce supplier    $800   72903  → close   (contact: "Riverside Produce")
Sysco (national)        $1,200   30328  → far
Entergy Arkansas          $400   72201  → far
Unknown misc               $300   —     → void
Total: $8,200

local_1 = 0.671   close_1 = 0.098   far_1 = 0.195   void_1 = 0.037
depth_1 = 0.963
```

**Layer 2 — one contact reports back.** Suppose Riverside Produce is the only one of Main Street Diner's contacts currently participating in the network, and Riverside Produce's own L1 report is:

```
local_B = 0.40   close_B = 0.30   far_B = 0.20   void_B = 0.10   (depth_B = 0.90)
```

Main Street Diner's ratio to Riverside Produce is 0.098 (the close_1 figure above). Everyone else Main Street Diner pays — the landlord, employees, Sysco, Entergy — isn't a confirmed node yet, so that entire remaining 0.902 of Layer 1 falls straight into void at Layer 2:

```
local_2 = 0.098 × 0.40 = 0.039
close_2 = 0.098 × 0.30 = 0.029
far_2   = 0.098 × 0.20 = 0.020
void_2  = (0.098 × 0.10) + 0.902 = 0.912

depth_2 = 0.039 + 0.029 + 0.020 = 0.088
```

**Combined, two layers deep:**

```
raw_score(2)    = local_1 + local_2   = 0.671 + 0.039 = 0.710
total_depth(2)  = depth_1 + depth_2   = 0.963 + 0.088 = 1.051
corrected_score = 0.710 / 1.051       = 0.675
average depth per layer = 1.051 / 2   = 0.53   (about 53%)
```

Notice depth_2 is low — not a bug, an honest reflection of an early-stage network where only one of Main Street Diner's several contacts has joined yet. As more contacts become confirmed nodes, depth_2 rises and the picture sharpens. This is the same "incomplete and honest beats complete and estimated" principle already stated elsewhere in the project — now it's a number, not just a sentiment.

---

## 4. Why This Preserves Privacy

Riverside Produce never had to tell anyone at Main Street Diner (or the network) *who* its own local customers or suppliers are. It only published four numbers: 0.40 / 0.30 / 0.20 / 0.10. And yet Main Street Diner's Layer 2 comes out exactly right. Here's why that's guaranteed, not lucky:

Say Riverside Produce's actual local spending (the 0.40) is really two payments under the hood — 0.25 to a farmer, 0.15 to a farm co-op, both local. Main Street Diner's true path-by-path local credit through Riverside Produce would be:

```
ratio(Diner→Produce) × ratio(Produce→Farmer)  +  ratio(Diner→Produce) × ratio(Produce→Co-op)
= 0.098 × 0.25  +  0.098 × 0.15
= 0.0245 + 0.0147
= 0.0392
```

Compare to what the layer formula actually computed: `0.098 × local_B = 0.098 × 0.40 = 0.0392`. Identical. Multiplication distributes over the sum — a contact's published aggregate already *contains* every itemized path a level down, so nobody upstream ever needs the itemized version. Each node only ever needs to publish its own four-number self-report. That's the whole privacy model, and it's a property of the arithmetic, not a policy promise.

**On convergence:** because every node leaks something to void at every layer (even a fully-confirmed, fully-local contact still has *some* far or void component somewhere, and most contacts aren't confirmed at all), the terms in `raw_score` shrink layer over layer. The infinite sum converges to a finite total for any real network — this is the same structure as compound interest or a Neumann series, not something new. That's also why "keep recursing until it stops mattering" is a mathematically sound stopping rule rather than an arbitrary one — see §5.

---

## 5. What's Fixed Math vs. What's a Deployment Choice

**Fixed — this is the math, it doesn't vary by community:**
- The recursive definition in §2
- `depth_n = 1 − void_n` at every layer, one formula
- `raw_score`, `total_depth`, `corrected_score` as defined above
- The invariant that each layer's four numbers sum to 1.0

**Configurable — decided per deployment or left to the user, not hardcoded:**
- **Local/close zone boundaries.** Already established elsewhere in the project — unchanged.
- **Zone weighting — NOT configurable. Listed here only because it used to be.** The old eddy_2 baked in `close × 0.5`. That weighting is gone from the core math *and* is not offered as a display option: local, close, far, and void travel as four honest, unblended numbers through every layer and stay unblended at the reading end. How much a nearby-but-not-local dollar is "worth" is a values judgment, and burying a values judgment in arithmetic presents it as a fact. Emphasis belongs in rendering, never in the numbers.
- **How many layers to compute.** Three options, all valid: a fixed system-wide N (e.g. always 5), a user-selectable depth, or recursing until depth_n becomes negligible (i.e., essentially everything left is void). The last option works cleanly *because* of the convergence property in §4 — there's no arbitrary cutoff needed, the terms genuinely stop mattering on their own.
- **Raw zip-level vs. pre-aggregated sharing.** A node can share its raw list of (fraction, destination zip) pairs — most flexible, lets zones be redefined later without re-collecting data — or pre-aggregate straight to the four local/close/far/void numbers before sharing anything — simpler, less data leaves the device, but locks in the zone definition at collection time. Both feed the same recursion; they differ only in when classification happens.

---

## 6. Data Flow / Architecture

```
[ entity's own device ]
   raw transaction data (amounts, counterparties, zips)
        ↓
   L0 (home zip) + L1 (own outflow, classified or raw — see above)
        ↓
   raw dollar amounts and identities are never transmitted past this point
        ↓
   L1 result exported in some shareable form (encrypted export — exact format TBD)
```

L0 and L1 are always computed entirely on the reporting entity's own device — this doesn't change from the existing design, and needs no network to produce a usable, publishable number (see §1, "what gets shown by default").

L2 and beyond require L1 data *from* an entity's own contacts — there's no way around that, it's structural, not a missing feature. How that data actually moves is a separate decision from the math and doesn't need to be settled to finish this spec:

- **Direct exchange** — contacts share their L1 report with each other directly, and each entity computes its own deeper layers locally.
- **Trusted hub** — every entity ships its encrypted L1 to a hub; the hub holds everyone's L1 and computes L2+ centrally, publishing results back out without exposing any node's data to any other node directly.

Both produce identical numbers. Which one (or which mix) a given deployment uses is a "distribution model" choice, and different variants of the network may reasonably choose differently. Nothing in §2–§5 depends on which one is picked.

---

## 7. Terminology Migration

| Old name | What it actually computed | Status now |
|---|---|---|
| `eddy_1` | `1 − void_ratio`, single node | = `depth_1` |
| `eddy_2` | `local + close×0.5`, single node, weighting hardcoded | = L1's raw vector; the ×0.5 is gone entirely — not in the core, not as a display option |
| `eddy_3` | `Σ ratio(A→B) × eddy_2(B)`, one hop, bounded | = `local_2` — one term feeding into `raw_score`, not a final score on its own |
| `depth_1` | `1 − void_ratio` | Correct as originally defined — kept, and now understood as one case of a general `depth_n` |
| `depth_2` | Named ("confidence in zone distribution"), no formula given | = `depth_n` at n=2, same formula as depth_1 |
| `depth_3` | Named ("confidence in extended scores"), no formula given | = `depth_n` at n=3, same formula |
| "Eddy Score" (book draft — unbounded, "20+") | Never formally defined | = `raw_score(N)` |
| "Network Score" (business plan) | Never formally defined | = `raw_score(N)` or `corrected_score`, depending which is being shown |
| "Share Score" (business plan) | = `eddy_1` | = `depth_1` |
| "Zone Score" (business plan) | = `eddy_2` | = L1's raw vector |

`crush_1` / `crush_2` (on-device hashing + ratio conversion) and `crush_3` (household-level aggregation) are unaffected by this document — they're implementation, not math, and the pipeline code for all of them is still unwritten regardless of this update.

---

## 8. Open Questions

Honestly flagged, not blocking:

- **Layer-count policy** — fixed N, user choice, or recurse-to-negligible. All three are mathematically sound; which one ships as the default is a product decision, not a math one.
- **Distribution model** — direct exchange vs. trusted hub vs. some mix, per §6. Doesn't change any formula above; changes what gets built.
- **Shared zone definitions across a network.** This document assumes one zone_config (local/close boundaries) per deployment, so a contact's own L1 report is already classified the same way the entity computing L2 would classify it. That's already how the project's existing zone-config examples work (one config per deployment), so it's stated here as a confirmed assumption rather than a new gap — but it's worth keeping explicit, since it's the reason contacts' raw reports can be used as-is without re-classification.

