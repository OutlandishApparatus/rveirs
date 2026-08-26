# RVEIRS — Design Document
**Version 0.2 · August 21, 2026**

*Published as a dated public record of the system's design. Free to read, build, fork, and deploy.*

---

## Contents

0. What this document is
1. Mission — what this is for
2. What RVEIRS is and isn't
3. Core concepts and vocabulary
4. The scoring math
5. Privacy architecture
6. Data formats
7. System architecture
8. The mesh / transport layer
9. Clustering — individuals, households, and larger structures
10. Deployment configuration
11. What's built, what isn't
12. Open questions
13. Licensing and provenance

---

## 0. What This Document Is

This is the complete design of RVEIRS as it stands today: the mission, the math, the privacy model, the data formats, and the architecture. It is written so that someone who has never spoken to the author could build a working implementation from it.

It has a second job. Publishing a dated, public description of how this works establishes prior art. RVEIRS is meant to stay free and un-enclosable — the way insulin's patent was sold for a dollar so that nobody could ever own it, and the way that promise was eventually broken anyway through routes that sale never anticipated. A public, timestamped record is one of several defenses against that outcome. It isn't sufficient alone, but it costs nothing and it's cumulative: every feature documented publicly as it's built is one more thing that can't later be claimed as someone else's invention.

**This document is a catalog, not a blessed path.** The system is meant to evolve to community needs, so the goal here is to describe as many working variants and options as possible and let a deployment assemble what it wants. Reference implementations and demos should aim to include every option that can be made to work, not one opinionated build.

Because of that, everything in here carries a maturity marker:

| Marker | Meaning |
|---|---|
| **[proven]** | Built and working, or mathematically proven |
| **[tested]** | Tried on real hardware, not yet at scale |
| **[lab]** | Works in synthetic testing only |
| **[design]** | Designed and reasoned through, not yet built |
| **[idea]** | Plausible, unvalidated, included for completeness |

Where this document and a later decision disagree, the later decision wins. This is a snapshot of a live design, not a specification frozen for compliance.

---

## 1. Mission — What This Is For

Money moves. Where it goes after it leaves your hand is invisible to you, and it's invisible to everyone else too. Not because anyone's hiding it — because nobody has ever built the instrument that would show it.

The existing tools measure the wrong thing. Standard economic indicators measure *wealth* — how much is held, where the totals sit. RVEIRS measures *current* — how much is moving, in which direction, through whom. A town with a big number attached to it and a town where money actually circulates before leaving are different towns, and today they look identical on paper.

The closest existing instrument is the local multiplier study (LM3 and relatives): pick a business, trace its spending, publish a number. That's an ammeter — one measurement, one point in the circuit, one moment in time, and it takes a researcher weeks. RVEIRS is a network of weather stations: many participants, continuously, each measuring only themselves, with the whole picture assembled from the parts.

**The thing being measured is leakage.** Every layer of spending loses some money to somewhere far away or somewhere untraceable. That loss is not an error in the data. It is the signal. A community where a dollar bounces five times before it leaves is structurally different from one where it's gone after a single hop — and until now, neither community had any way to know which one it was.

**What RVEIRS does with that information: nothing.** It doesn't recommend, rank, shame, or optimize. It reports what's true and stops. The design assumption is that people make better decisions when they can see more, and that deciding *for* them — however well-intentioned — is both presumptuous and less effective than making the picture legible and getting out of the way. This is a deliberate constraint, not an unfinished feature.

**Design commitments that follow:**

- **Cheap.** A basic node is about $20 in parts. Cost is an adoption barrier, and adoption is the whole game.
- **Low-friction.** A business doesn't restructure, incorporate differently, hire a lawyer, or sign anything. It reports where money it was already spending went.
- **Privacy by arithmetic, not by promise.** Policies get changed, subpoenaed, or quietly updated. The guarantee here is a property of the math (§5), which cannot be revised by whoever ends up running a server.
- **No political prerequisite.** RVEIRS doesn't require agreement about capitalism, policy, or what should be done. Only that someone wants to know where their money goes.
- **Un-enclosable.** Free, open, forkable, licensed so it stays that way (§13).

---

## 2. What RVEIRS Is and Isn't

**Is:** a live, local-multiplier-focused economic dashboard. Runs on ordinary personal devices. Passes small data over a local mesh. Owned by its users. Primary purpose: calculating local multipliers to quantify recirculation and extraction.

**Isn't:**

- Not a bank connection, an accounting package, or anything touching accounts.
- Doesn't model interest, inflation, debt service, or investment returns. Anything requiring policy or legislation to change is out of scope — it's just part of the flow.
- Doesn't store transaction records, dollar amounts, vendor names, or account identifiers anywhere on the network.
- Doesn't rate businesses as good or bad. A high score at a chain and a low score at a beloved local institution isn't a contradiction — it's information.
- Doesn't require internet connectivity to function.
- Not a co-op, and doesn't require anyone to become one. Cooperation as a *behavior* — where you choose to spend — requires none of the legal machinery that cooperative *ownership* does. That gap is precisely the barrier RVEIRS is designed not to have.

---

## 3. Core Concepts and Vocabulary

**Entity** — a participating business, organization, household, or cluster. The unit that reports.

**Zipsum** — an entity's own outbound flow expressed as fractions of throughput, keyed by destination zip code. The atomic unit of data in the system. No dollar amounts, no counterparty names, no transaction records. **Zipsums are public by design.**

**Throughput** — total inflow over the reporting period. The denominator for ratios. Not transmitted for businesses; transmitted only within clustering (§9), where it's required and then destroyed.

**Zone** — a classification of destination zips *relative to whoever is reading*:

| Zone | Definition |
|---|---|
| `local` | The tightest circuit — home zip and immediate neighbors |
| `close` | Regional, meaningfully connected but not core |
| `far` | Any other *known* destination |
| `void` | No known destination — blank or `99999`. Not missing data. Signal. |

**Zones live at the reading end, not the reporting end.** A zipsum contains raw zips; the consumer collapses them into zones using their own config. Four zones is the default set, not a requirement — a deployment wanting more or different bands can define them, because nothing in the transmitted data assumes any particular set.

Zones are a *radius* classification, unrelated to layers. An older draft numbered them Z1–Z4; they are now always named, never numbered, to prevent collision with layer numbering.

**Layer** — recursion depth. How many hops from the reporting entity a measurement follows the money. L1 is "where I spent it." L2 is "where the people I paid spent it."

**Score** — how much local economic activity a dollar generated before fully leaving the region. Unbounded upward in principle. *Temperature, not percentage.*

**Depth** — how much of the traced money had a known destination, versus disappearing into void. Bounded 0–1. The trust rating on the score above it. **A score reported without its depth is a guess with decoration**, and implementations should treat omitting depth as a bug.

---

## 4. The Scoring Math

**[proven]** — this section is fixed and does not vary by deployment.

### 4.1 Layer 0 — Self

L0 is a reference point, not a score: the entity's home zip.

*[idea] For entities with split or absentee ownership, L0 could become a distribution across zips — e.g. 60% resident-owned, 40% out-of-state investor. The model accommodates this without modification whenever it's wanted.*

### 4.2 Layer 1 — Contacts

An entity's L1 is its zipsum: outbound flow as fractions of throughput, keyed by destination zip.

```
{ "72901": 0.6707, "72903": 0.0976, "30328": 0.1463,
  "72201": 0.0488, "void": 0.0366 }
```

**Classification into zones happens at the consumer, not the reporter.** A reader applies their own zone config to the raw zips:

```
local_1 = Σ fractions where zip ∈ local zone
close_1 = Σ fractions where zip ∈ close zone
far_1   = Σ fractions where zip is known but in neither
void_1  = the void fraction
```

This is what preserves regional variability. Two deployments reading the same zipsum legitimately produce different zone vectors, because "local" is a different place for each of them. Collapsing at the sender would destroy that permanently and irreversibly — the raw zips can always be re-collapsed, but a collapsed vector can never be re-expanded.

**L1 requires no network and no cooperation from anyone.** A business knows what it paid its landlord and its produce supplier without anyone else's participation. This is the front door of the entire system and the default view.

### 4.3 Layer n (n > 1) — Recursion

For entity A with confirmed participating contacts B, where `ratio(A→B)` is A's fraction of throughput going to B:

```
local_n(A) = Σ over confirmed contacts B:  ratio(A→B) × local_(n-1)(B)
close_n(A) = Σ over confirmed contacts B:  ratio(A→B) × close_(n-1)(B)
far_n(A)   = Σ over confirmed contacts B:  ratio(A→B) × far_(n-1)(B)

void_n(A)  = [ Σ over confirmed B: ratio(A→B) × void_(n-1)(B) ]
           + [ Σ over UNCONFIRMED contacts: their share of A's outflow ]
```

In words: A's layer-n local credit is the sum, over everyone A pays, of *how much A paid them* times *how local their layer-(n−1) spending was*. Anyone A pays who isn't a confirmed reporting node contributes entirely to void — a dollar can't be traced past someone who isn't measuring their own flow.

**Invariant:** `local_n + close_n + far_n + void_n = 1.0` at every layer, for every entity, automatically. An implementation violating this has a bug.

**Where this runs: on the user's own device.** Once a user holds their contacts' published zipsums, the recursion is a handful of multiplications — well within a phone's capability. No server-side computation, no scheduling, no spare-cycle queue. An earlier design assumed recursion would be expensive enough to need offloading to servers computing gradually in the background; the distributive property (§5.1) makes that unnecessary. **Servers in this system do no math at all.**

### 4.4 Depth

```
depth_n = local_n + close_n + far_n  =  1 − void_n
```

One formula, every layer.

### 4.5 The score

Across N computed layers:

```
raw_score(N)     = Σ (n = 1..N) local_n
total_depth(N)   = Σ (n = 1..N) depth_n
```

- **`raw_score`** — the temperature number. "How much local activity did this dollar generate before it was gone."
- **`total_depth`** — how much of that was actually traced rather than assumed.

**Report both, always, along with N.** Depth is the clarifier; a score without it is meaningless.

*Optional derived number:* `traced_ratio = raw_score / total_depth`, bounded 0–1 — "of the money we could trace, how much stayed local." Useful mainly for comparing networks with very different depths, where raw scores aren't directly comparable. Not a headline number, and nothing depends on it. *(Previously called `corrected_score` — renamed because nothing is being corrected. Nothing was wrong.)*

### 4.6 No blended weighting in the core

Local, close, far, and void travel as separate, unblended numbers through every layer. An earlier version hardcoded `local + (close × 0.5)` into the score itself. That is removed and should not be reintroduced.

The reason is a design principle, not a preference: *how much a nearby-but-not-local dollar is worth* is a values judgment, and burying a values judgment inside arithmetic presents it as a fact.

Any zone may be given any weight, by the user, at display time — clearly labeled as a chosen weighting, with a visible off switch, defaulting to none. In practice, when all zone values are visible at once, weighting is largely unnecessary; a blended number mostly hides information the display was already showing.

### 4.7 Convergence and stopping

Every node leaks something to void at every layer. Even a fully-confirmed, fully-local contact has some far or void component somewhere, and most contacts aren't confirmed at all. Terms shrink layer over layer and the infinite sum converges to a finite total for any real network — the same structure as compound interest or a Neumann series.

Three valid stopping rules, all **[proven]** sound:

1. **Fixed N** — always compute exactly N layers.
2. **Threshold** — stop when `depth_n` drops below a set fraction (e.g. 10%) — everything remaining is essentially void.
3. **Converge** — recurse until terms stop mattering at all. Well-defined precisely because of the convergence property above.

Expose all three. See §10 for defaults.

### 4.8 Worked example

**[proven]** — hand-checkable; use as an implementation conformance test.

**Layer 1 — Main Street Diner, home zip 72901.** Reader's zone config: local = {72901}, close = {72903, 72904, 72956}.

```
Employee wages          $4,000   72901    → local
Commercial rent         $1,500   72901    → local
Local produce supplier    $800   72903    → close   (contact: Riverside Produce)
Sysco (national)        $1,200   30328    → far
Entergy Arkansas          $400   72201    → far
Unknown misc              $300   (blank)  → void
Total: $8,200

local_1 = 0.671   close_1 = 0.098   far_1 = 0.195   void_1 = 0.037
depth_1 = 0.963
```

**Layer 2 — Riverside Produce is the diner's only participating contact.** Its zipsum, collapsed against the same config:

```
local_B = 0.40   close_B = 0.30   far_B = 0.20   void_B = 0.10   (depth_B = 0.90)
```

Everyone else the diner pays isn't a confirmed node, so that remaining 0.902 falls to void:

```
local_2 = 0.098 × 0.40 = 0.039
close_2 = 0.098 × 0.30 = 0.029
far_2   = 0.098 × 0.20 = 0.020
void_2  = (0.098 × 0.10) + 0.902 = 0.912
depth_2 = 0.088
```

**Combined:**

```
raw_score(2)    = 0.671 + 0.039 = 0.710
total_depth(2)  = 0.963 + 0.088 = 1.051
traced_ratio    = 0.710 / 1.051 = 0.675
avg depth/layer = 1.051 / 2     = 0.53
```

`depth_2` is low. Not a bug — an honest reflection of an early network where one contact of several has joined. As contacts join, depth rises and the picture sharpens. **Incomplete and honest beats complete and estimated**, and here that principle is a number rather than a sentiment.

---

## 5. Privacy Architecture

### 5.1 The guarantee

**[proven]**

Riverside Produce never tells anyone — not the diner, not the network — *who* its suppliers or customers are. It publishes a zipsum. The diner's Layer 2 still comes out exactly right.

Why this is guaranteed rather than lucky: suppose Riverside's 0.40 local is really 0.25 to a farmer plus 0.15 to a co-op. True path-by-path credit:

```
0.098 × 0.25  +  0.098 × 0.15  =  0.0245 + 0.0147  =  0.0392
```

What the layer formula computes:

```
0.098 × 0.40 = 0.0392
```

Identical. Multiplication distributes over addition, so a contact's published aggregate *already contains* every itemized path one level down. Nobody upstream ever needs the itemized version.

**This is the whole privacy model, and it is a property of arithmetic rather than a policy promise.** No operator, court order, or change of management can revise it.

It's also what makes §4.3 cheap: because the aggregate already contains the detail, recursion is multiplication rather than graph traversal.

### 5.2 The hard architectural line

```
[ entity's own device ]
   raw transaction data (amounts, counterparties, zips)
        ↓
   L0 (home zip) + L1 (zipsum)
        ↓
   ══════ raw amounts and identities never cross this line ══════
        ↓
   zipsum published
```

**On-device computation before transmission is non-negotiable.** Any implementation that moves raw financial data across a network before converting to ratios is not RVEIRS, whatever it calls itself. This is the one place in the design where "that's a deployment choice" does not apply.

### 5.3 Operator access — scope

Three commonly-conflated guarantees:

1. **Transport encryption** protects data in flight. Not at rest, not in use.
2. **Network isolation** protects against remote intrusion. Not local access.
3. **Operator access** — whoever runs the computation sees whatever it reads, regardless of (1) and (2).

**For businesses, (3) does not apply.** Servers hold only published zipsums — data that is public by design. There is nothing private on a business-side server for an operator to misuse, and no confidentiality claim to break.

**For clustering (§9), (3) applies fully.** Cluster servers briefly hold decrypted, still-identified data with throughput attached, mid-processing. That window is the one genuinely sensitive point in the system, and §9 is built around minimizing and controlling it.

### 5.4 Where the guarantee is weaker

- **Small-network inference.** In a network with few participants, a published zipsum plus local knowledge may identify a business. The math prevents *derivation*; it doesn't prevent *inference* by someone who already knows the town.
- **Cluster differencing.** See §9.7.
- **Business connection disclosure** (§7.6) deliberately trades privacy for benefit — which is why it's opt-in and dual-consent.

---

## 6. Data Formats

### 6.1 Input (on-device, never transmitted)

Two CSVs per entity:

```
amount, name, zip
4000.00, Employee wages, 72901
1500.00, Commercial rent, 72901
800.00, Local produce supplier, 72903
1200.00, Sysco, 30328
300.00, Unknown misc,
```

- `inflows.csv` — sum = throughput = the ratio denominator.
- `outflows.csv` — where money went.
- `name` is optional, display-only, never transmitted.
- `amount` becomes a denominator, then is discarded.
- Blank or `99999` zip = void.

### 6.2 The zipsum (the published unit)

```json
{
  "entity_id": "<stable hashed pseudonymous ID>",
  "period":    "2026-08",
  "version":   1,
  "seq":       1,
  "flows": {
    "72901": 0.6707,
    "72903": 0.0976,
    "30328": 0.1463,
    "72201": 0.0488,
    "void":  0.0366
  }
}
```

Fractions, never dollars. Raw zips, never pre-collapsed zones (§4.2). No throughput for businesses.

### 6.3 Identity

`entity_id` is a stable, hashed, pseudonymous identifier for the *entity*. Explicitly **not** a mesh node ID — a node ID identifies a radio, and radios get swapped, shared, replaced, and moved. Binding economic identity to hardware identity breaks the moment someone changes a board. The loop server (§7.3) also repeats **per entity**, not per node, for the same reason.

### 6.4 Zone config (reader-side)

```json
{
  "deployment": "carroll-county",
  "local": ["72632","72661","72616","72631","72611"],
  "close": ["72653","72756","72745","72752","72717"]
}
```

`far` and `void` are automatic. This config never leaves the reader — it's how *you* interpret zipsums, not how anyone reports them. Changing it changes your own scores' comparability over time, so version it if it changes; historical zipsums can always be re-collapsed under the new config, since nothing was lost at collection.

### 6.5 Temporal resolution

**Default: monthly.** Frequent enough to be live, infrequent enough not to be a burden.

Each zipsum is tagged with its period, version, and sequence number — so a receiver can tell "I have 1 and 3, missing 2" rather than silently holding an incomplete picture it believes is complete.

*Open, see §12: whether periods are discrete non-overlapping chunks or cumulative dated snapshots, and how to merge flow rates collected at differing resolutions. These are rates, not quantities — they don't average directly. This affects the final schema.*

---

## 7. System Architecture

### 7.1 The shape

**Data** — an entity enters where money went, on their own device, staying there.

**Math** — on the same device, amounts become ratios; amounts and names are discarded. What travels is a zipsum.

**Publication** — zipsums are broadcast into a shared pool. They're public. Anyone can grab what they want.

**Interpretation** — each user's PWA collapses raw zips into their own zones, and recurses through their own contacts' zipsums to compute their own deeper layers, locally.

There is no central scoring authority. Servers distribute; devices compute.

### 7.2 The network is a group chat of CSVs

**[design]** — the core architectural simplification.

- Users publish their zipsum into a **live room**.
- A **loop server** — a Linux box attached to a mesh node by USB serial — records everything it hears and rebroadcasts it into a **loop room**, repeating each entity's most recent zipsum on a cycle.
- Clients listening to the loop room receive the current state of the network without asking anyone for anything.

That's the whole server role. No computation, no queue, no scheduling logic, no database of anything private.

### 7.3 Sync: broadcast plus loop

Broadcast is safe here because nothing on the wire is private. But broadcast alone is fire-and-forget — a client offline during a transmission misses it permanently, with no retry.

The loop closes that gap without a request/response handshake. A late-joining client just listens to a room that's already repeating everything. Same job a pull-based sync would do, none of the client-side bookkeeping.

- **Live room** — real-time, if there's traffic right now.
- **Loop room** — default. Always current, always repeating.

**Give the loop server two radios.** §8.2: the SX1262 is half-duplex and physically cannot transmit and receive simultaneously. One radio listens on live, one transmits on loop. This isn't a convenience — it's the hardware workaround for a datasheet-level constraint.

*Note: an earlier design bridged a Room Server's circular buffer to permanent storage via MQTT. The loop server records everything to disk and is itself the permanent storage, so that bridge has nothing left to bridge.*

### 7.4 The client (PWA)

- Builds the CSV at the operating temporal resolution.
- Presents the finished payload to the user with an explicit **ask before pushing to the mesh**. Nothing publishes silently.
- Collapses raw zips into the user's own zones.
- Recurses locally through contacts' zipsums.

**The demo and the live version are the same software.** The only difference is whether the client is paired to a live mesh node. A business uses the real tool on day one of the cheapest possible pilot; nothing gets rebuilt when it scales.

Single-file distribution — from the web, from physical media, or from a local server's own WiFi. No app store, no internet dependency.

*[idea] Ship the PWA with a recent DB snapshot bundled, so a first-time user opens it looking at a live town rather than an empty screen waiting to sync. Requires a prominent "data as of" date — which is worth having on every view regardless, not just bundled ones.*

### 7.5 Business connection disclosure (optional)

A business may publicly list which other business IDs it flows to — no amounts, unlike the zipsum — as informal advertising of a relationship.

**Dual-consent required.** Both parties agree before a named connection appears. One party cannot unilaterally disclose a relationship the other never agreed to expose.

**Generalize this:** any downstream use of RVEIRS data beyond what a participant originally agreed to — research access, derived products, anything — requires its own explicit, case-by-case opt-in. **Consent does not inherit.**

---

## 8. The Mesh / Transport Layer

*Least settled part of the system. Current direction, not decided architecture.*

### 8.1 What the mesh is for

LoRa carries **small zipsum updates only** — never the PWA itself, never a full DB sync. Those move over WiFi or physical media. LoRa is low-bandwidth by design, and any design forgetting this will fail.

```
WiFi (local, at a node) → LoRa mesh → loop server via USB serial → recorded, rebroadcast
WiFi → PWA delivery and full DB download
```

### 8.2 Hardware constraints

**[proven]** — confirmed from documentation, not assumed.

- **BLE is one active connection per node at a time.** Fundamental to BLE peripherals, firmware-independent.
- **The SX1262 radio (Heltec V3, most common LoRa boards) is half-duplex** — per Semtech's datasheet. Cannot transmit and receive simultaneously. This is why a repeater must fully receive a packet before retransmitting, and why the loop server wants two radios (§7.3).

### 8.3 WiFi for users, Bluetooth for the owner

**Default: users connect to a node over WiFi.** An access point handles many clients natively; BLE handles one. This sidesteps the BLE limit entirely for public-facing nodes.

**Bluetooth stays free for the node's owner** — a guaranteed way in even when the WiFi side is saturated or busy.

*Open, see §12: how many simultaneous WiFi clients one bare node actually handles.*

### 8.4 Firmware direction

**Current plan: build and test on MeshCore.** Large real-world meshes already validate it in the field — lower risk than debugging a new protocol and the rest of the system simultaneously.

MeshCore offers three roles, all firmware on the same board:

| Role | Function |
|---|---|
| Companion Radio | Client. Sends/receives, doesn't repeat. |
| Repeater | Infrastructure. Forwards packets, no UI. |
| Room Server | Repeater plus stored message history. |

**[design] — status: unverified.** Prior testing was on Meshtastic, which has no such role structure. These roles are the intended direction pending real MeshCore testing, and this section should be revised from experience rather than documentation. Room Server is flagged by MeshCore's own project as less mature than the other two.

**Custom firmware is deliberately avoided.** Staying on an actively maintained platform means fixes and community support arrive for free, rather than one person maintaining a radio stack on top of everything else.

**Reticulum is the long-term candidate**, to be revisited deliberately. Architecturally a better fit — blends LoRa, WiFi/LAN, and internet into one network without gateways, and has a genuine no-install browser client. Tradeoff: typically needs a real host driving the radio rather than running standalone on a bare ESP32.

### 8.5 Cost

| Item | Cost |
|---|---|
| Basic node (indoor, USB-powered) | ~$20 |
| Solar/battery repeater (battery, housing, panel) | ~$100 |
| Pilot: server on hand + handful of nodes | under $200 |
| First real batch: 40–50 basic nodes | ~$1,000 |
| Full town saturation: ~500 nodes | $12–15k |

To be clear about what that top figure buys: not a transformed economy. The tools to do it yourself.

*Note: an earlier draft proposed a hybrid dense-WiFi-downtown / LoRa-edges topology to fight latency and bandwidth. With the loop repeating indefinitely, patience is cheaper than infrastructure. Dropped.*

---

## 9. Clustering — Individuals, Households, and Larger Structures

*Deferred. Businesses come first. Documented so the design exists, not because it's next to build.*

### 9.1 Why clustering exists

A household's flows are more identifying than a business's, in a smaller population, with no professional expectation of disclosure. Individual zipsums are therefore never published individually — they're folded into a cluster first, and the cluster publishes.

But clustering isn't only a privacy mechanism. **It's how the system scales structurally.** A cluster publishes a zipsum and a summed throughput, which is exactly what any other entity needs to treat it as a contact in their own recursion. Clusters of clusters work identically — neighborhoods into districts, districts into a city, mixed-source clusters combining households, businesses, and government. **The structures are yours to define.** The math doesn't care.

### 9.2 Who runs a cluster

**[idea]** Cluster servers hosted by locally accountable institutions — chamber of commerce, churches, credit unions, whoever a community actually trusts.

Users choose which cluster gets their key. That converts the §5.3 operator-access risk from something imposed into something chosen, and gives a user a rough sense of whose money they're blending with — which is itself information they might want.

### 9.3 Membership and floors

- Membership requires sharing the same zip at L0 — physically same zip, not self-selected. Harder to game than open groups.
- **A hard minimum cluster size is required** before any cluster score is computed or shown. Same shape as k-anonymity: a score from two or three households is identifiable data about those specific people. A dense urban zip satisfies this trivially; a sparse rural zip may not satisfy it at all.

### 9.4 Why throughput is required

A plain unweighted mean of ratios treats a household moving $49k/year and one moving $1M/year as equally representative. For "what's a typical household's experience," that's correct. For "what's actually happening to money in this zip," it's badly wrong:

> A $1M household at 3% local, a $49k household at 60% local. Unweighted mean: 31.5% local. Weighted by actual dollars: `(0.03 × 1,000,000 + 0.60 × 49,000) ÷ 1,049,000 ≈ 5.7%`.

Not a rounding difference — a different claim.

Throughput is needed twice, and the second reason matters more:

1. To weight each household's ratios correctly when folding into the cluster's zipsum.
2. **To give the cluster its own throughput**, so it can participate in anyone else's recursion as a properly weighted contact. Without it, the cluster is a ratio with no mass.

The fold works by converting ratios back into amounts using each household's throughput, summing into the new cluster, and re-deriving ratios from the combined total.

### 9.5 The five files

Split so no single artifact carries both identity and content.

| # | File | Fate |
|---|---|---|
| 1 | Incoming personal zipsum + throughput | Deleted after fold |
| 2 | Reporting roster (IDs only, no flow data) | Deleted at period close |
| 3 | De-identified throughput list | Retained or pooled upward — see below |
| 4 | Prior cluster zipsum + throughput | Deleted after fold |
| 5 | New cluster zipsum + throughput | **Published** |

Sequence: de-identify → add to roster → fold weighted zipsum into cluster → delete 1, 4 → publish 5 → delete 2 at period close.

Only the current roster matters — nobody's time-travelling, so it resets each period.

The throughput list (3) is genuinely useful for research, and is the natural candidate for pooling upward into a cluster-of-clusters before any public release. Absent that, delete it.

### 9.6 Random insertion is a security requirement

**Never append.** If the roster and the throughput list both grow by appending, row N of one is the same person as row N of the other, and the entire separation is defeated by opening both files side by side.

**Insert each new record at a random index.** The same applies to any order-preserving accumulation anywhere in the pipeline — arrival order is itself an identifier.

### 9.7 Residual risk

Pooled totals are subtractable. If someone can determine N−1 members of a small cluster, the pooled total yields the Nth by subtraction — and raw throughput is far more identifying than a percentage. The minimum-size floor must cover throughput pooling, not only the published ratio.

### 9.8 Mean vs. median

Different problems, not alternatives:

- **Throughput-weighted mean** fixes *accuracy* — whether bigger households count proportionally.
- **Median** fixes *robustness* — whether one extreme household distorts or exposes the group.

A running mean needs only a total and a count, so values can be discarded on arrival. **Median cannot** — it needs the actual list at computation time. File 3 makes median possible; that's a deliberate accepted tradeoff, not an oversight.

### 9.9 Related deferred feature

Once individuals participate, showing a user their own wage as a factor of their region's living wage is a natural extension — a private mirror for that individual, not an assessment of any employer. RVEIRS does not evaluate wages and this doesn't change that.

---

## 10. Deployment Configuration

| Decision | Default | Notes |
|---|---|---|
| Zone boundaries | Community-defined, no shipped default | Reader-side. When in doubt, draw local *smaller* — a tight zone with honest scores beats a loose one that flatters. |
| Zone weighting | **None** | Available to users at display time. Off by default; blending mostly hides what the display already showed. |
| Layer count | **Max-N, default 3** | Threshold and converge also available. |
| Sharing format | Raw zip-level zipsums | Preserves regional variability. |
| Distribution | Broadcast + loop | Trusted cluster server required for §9. |
| Temporal resolution | Monthly | |

**On the layer default:** starting at 3 is deliberate. Early networks won't have the depth to justify more, and it sets a visible bar for the network to outgrow — a challenge to force raising it, rather than a ceiling pretending to be a decision.

**Worked zone examples:**

```
River Valley / Fort Smith
  local — 72901–72956 corridor and surrounding River Valley zips
  close — rest of Arkansas

Carroll County / Eureka Springs
  local — 72632, 72661, 72616, 72631, 72611
  close — 72653, 72756, 72745, 72752, 72717
```

Different scales, identical structure. Only the lists change.

---

## 11. What's Built, What Isn't

**Working:**
- L1 calculator — submission interface and live scoring, on-device. **[tested]**
- Synthetic network generator — full multi-hop test networks, hand-checkable values, shared-node convergence cases. **[lab]**
- Meshtastic transport. **[tested]**

**Designed, not built:**
- Ln recursion in shipped code. Math complete and proven; implementation deferred. **[proven / not built]**
- Full on-device pipeline (hashing, ratio conversion, publication). **[design]**
- Loop server. **[design]**
- MeshCore deployment and role validation. **[design]**
- Zipsum transport, chunking, loop rebroadcast. **[design]**
- Clustering (§9). **[design]**
- Network visualization — force-directed layout, geographic/financial toggle, timeline animation. **[design]**

**Known outdated in current code:** the shipped calculator computes `eddy_1`/`eddy_2` with hardcoded `close × 0.5` weighting, collapses to four zones at the sender rather than publishing raw zipsums, and stops at L1. It predates this document. It works and it's honest about what it measures; it just isn't current.

---

## 12. Open Questions

Genuine invitations. Several are questions a community should answer for itself rather than inherit.

**Needs testing:**
- **MeshCore roles and server structures.** Untested. Real hardware supersedes §8.4.
- **Simultaneous WiFi clients per node.** Unknown in vivo. The one available data point (3–5) was measured through a software proxy, not a bare node.
- **Transport at scale.** Hundreds of participants over LoRa is undemonstrated. Payloads are small, so requirements are modest — but modest isn't proven.

**Needs math:**
- **Temporal merging.** Discrete deltas vs. cumulative dated snapshots — and how to merge flow rates collected at differing resolutions. These are rates, not quantities; they don't average directly. Duration-weighting is probably the answer. Blocks finalizing the CSV schema.
- **The per-household weighted fold** into a cluster zipsum (§9.4). The approach is clear; the exact formulation needs writing down.

**Needs a community decision:**
- **Zone boundaries.** No correct answer exists to inherit.
- **Minimum cluster size.** Likely varies with zip population density rather than being one global constant.
- **Whether a blended headline number is wanted at all.**

**Unbuilt extensions:**
- **Non-monetary exchange.** The math doesn't care what the units are — labor, food, skills, time would work identically. Nobody's built it.

---

## 13. Licensing and Provenance

**Intent:** the code, the math, and the hardware designs stay free and un-enclosable. Anyone can build one. Anyone can run one. Anyone can fork it. Nobody can take it away from anyone else.

**Leading license: AGPL.** Permissive licenses (MIT, Apache) leave a loophole where a modified version offered as a network service never has to share its changes. AGPL closes it. *Not legal advice — a real lawyer should review this before it's load-bearing.*

**Why this matters and isn't paranoia:** insulin's discoverers sold the patent for a token dollar specifically so it could never be enclosed. It was enclosed anyway, decades later, through routes that sale never anticipated. Good intentions in a license aren't enough; the license has to close the specific doors.

**Funding does not depend on restricting the software.** The tool is free. The mission is funded directly by people who want it to exist — pay-what-you-want distribution and direct patronage, open-book. That separates "how does this sustain itself" from "who's allowed to use it," which is why the license can be maximally permissive about use while strict about enclosure.

**Prior art as ongoing practice.** Publishing a dated public description of a feature as it's built blocks later patent claims on that specific thing. It does not cover future improvements unless those are also published. This is a habit, not a filing — which is why this document has a version number and a date, and will have successors.

---

*RVEIRS — Design Document v0.2 · August 21, 2026*
*The instrument observes without judging. Absorbs without extracting. Makes visible without exposing.*
