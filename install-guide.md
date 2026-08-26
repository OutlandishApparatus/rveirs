# RVEIRS — Install Guide
**v0.2 · August 21, 2026**
*How to stand up a local economic telemetry network*

---

## Who This Is For

Someone standing up RVEIRS in their own community. You don't need to be an engineer, but you should be comfortable flashing firmware, running a Linux box, and reading a log when something doesn't work.

If you want to know *why* any of this is shaped the way it is, that's the design guide. This document is what to do.

**Status honesty:** parts of this are proven, parts are designed but untested. Every section is marked. Don't skip the markers — building on an untested section means you're the one testing it, and you should know that going in.

| Marker | Meaning |
|---|---|
| **[proven]** | Working, deployed, or mathematically guaranteed |
| **[tested]** | Works on real hardware, not at scale |
| **[design]** | Reasoned through, not yet built |

---

## 1. What You're Building

Three pieces. They only make sense together.

**Nodes.** Cheap radios at participating businesses. Someone connects a phone, submits their zipsum, the node passes it into the mesh. About $20 each.

**A loop server.** One Linux box with two radios attached. It listens to everything published, records it, and rebroadcasts it on a cycle. This is the only piece that has to keep running.

**The client.** A single-file web app. Runs on any phone. Does all the math locally.

That's the whole machine.

**Note on what the servers do:** for a business network, the loop server does **no computation whatsoever**. It records and repeats. All scoring happens on user devices. This matters for security — there's nothing on that box worth attacking, because everything on it was already public.

(Cluster servers, used for household participation, *do* compute and *do* briefly hold sensitive data. That's a different deployment with different requirements — §8.)

---

## 2. Before You Touch Hardware

### Define your zones **[proven]**

The one decision you can't easily undo. Your community picks which zip codes count as **local** and which count as **close**.

```
Carroll County / Eureka Springs
  local — 72632, 72661, 72616, 72631, 72611
  close — 72653, 72756, 72745, 72752, 72717

River Valley / Fort Smith
  local — 72901–72956 corridor and surrounding River Valley zips
  close — rest of Arkansas
```

Local is the tightest circuit — where spending genuinely recirculates before leaving. Usually a home zip plus immediate neighbors. **When in doubt, draw it smaller.** A tight zone with honest scores is worth more than a loose one that flatters the numbers.

Close is regional. County lines are a reasonable start. Adjust for how your economy actually moves, not how the map looks.

There is no correct answer. Document what you choose and keep it consistent — changing zone definitions mid-deployment makes scores incomparable over time. (You *can* recover, since raw zipsums retain original zips and can be re-collapsed under a new config. But version it and re-derive rather than mixing.)

### Pick your period **[design]**

Weekly is the default. Log a week, collect a week, publish. Results land about two weeks behind live.

Monthly also works and is less burden on participants. Pick one and hold it.

### Find your anchors

Not technical, but it determines whether this works. A handful of locally trusted businesses matter more than a large number of interested strangers. Awareness doesn't convert to installed hardware on its own — a well-liked owner walking a neighbor through it does.

---

## 3. Hardware

### Shopping list

| Item | Qty | ~Cost |
|---|---|---|
| LoRa board (Heltec V3 or similar SX1262) | 1 per node | $20 |
| LoRa board for loop server | 2 | $40 |
| Linux box (Pi 4, Pi Zero, old laptop, anything) | 1 | on hand |
| USB cables | as needed | — |
| Solar + battery + weatherproof housing (repeaters only) | as needed | ~$80 each |

**Pilot:** a server you already own plus a handful of nodes. Under $200.
**First real batch:** ~$1,000 for 40–50 nodes.
**Town saturation:** ~500 nodes, $12–15k.

To be clear about what that last number buys: not a transformed economy. The tools to do it yourself.

### Two constraints that shape everything **[proven]**

**BLE is one connection at a time.** A Bluetooth peripheral serves one client. Firmware doesn't change this.

**The SX1262 is half-duplex.** Straight from Semtech's datasheet — it physically cannot transmit and receive simultaneously. This is why a repeater must fully receive a packet before forwarding it, and why the loop server needs two radios.

Design around these. Don't try to firmware your way past them.

---

## 4. Node Setup

**[design]** — role structure is MeshCore's; not yet validated on real hardware by this project. Prior testing was on Meshtastic, which has no equivalent role system. Expect to correct this section from experience.

### Firmware

Flash MeshCore. Three roles, all the same board:

| Role | Use |
|---|---|
| **Companion** | A business node. Sends and receives, doesn't forward. |
| **Repeater** | Extends range. Solar hilltop or anything on mains power. |
| **Room Server** | Repeater that also holds recent history. Flagged by MeshCore as the least mature of the three. |

Most nodes are Companions. Add Repeaters where geography demands.

**Don't write custom firmware.** Staying on a maintained platform means fixes and community support arrive for free instead of one person maintaining a radio stack alone.

### Connectivity: WiFi for users, Bluetooth for the owner **[design]**

Configure the node's WiFi access point as the way customers and staff connect. An AP handles many clients natively; BLE handles one. This sidesteps the BLE limit for anything public-facing.

**Leave Bluetooth free for whoever owns the node.** It's a guaranteed way in even when WiFi is saturated.

*Untested: how many simultaneous WiFi clients one bare node actually handles. The one figure floating around (3–5) was measured through a software proxy, not a raw node. Test small and watch for where it breaks.*

### Identity

A node ID identifies a **radio**, not a business. Radios get swapped, shared, and replaced. Each participating entity needs its own stable hashed identifier, independent of whatever hardware carries its traffic today.

The loop server repeats **per entity**, not per node, for exactly this reason.

---

## 5. The Loop Server

**[design]** — the core of the architecture and the piece most worth getting right.

### What it is

A Linux box with two LoRa radios on USB serial.

- **Radio A** listens on the **live room** — where clients publish.
- **Radio B** transmits on the **loop room** — rebroadcasting each entity's most recent zipsum on a cycle.

Two radios because of half-duplex. One radio cannot do both jobs.

### What it does

Records everything it hears to disk. Repeats the current state of the network, forever, on a loop.

That's all. It computes nothing.

### Why a loop instead of a request/response sync

Broadcast is fire-and-forget. A client that's offline when something is published misses it permanently — there's no retry.

The loop fixes that without any handshake. A client joining late just listens to a room that's already repeating everything. Same outcome as "ask the server what I missed," none of the bookkeeping.

- **Live room** — real-time, if there's traffic right now.
- **Loop room** — the default. Always current, always repeating.

### Storage

The loop server's disk is the permanent record. Everything published, timestamped, kept.

*An earlier design bridged a Room Server's circular buffer to permanent storage over MQTT. The loop server records to disk directly, so that bridge has nothing left to bridge. If you see MQTT in older notes, it's superseded.*

### Placement

Anywhere with power and radio reach into the mesh. It doesn't need internet. It doesn't need to be publicly reachable. A closet works.

---

## 6. The Client

Single HTML file. Distribute however:

- Served from the loop server's own WiFi
- Hosted on the web
- Physical media — a USB stick works fine

**The demo and the live version are the same software.** The only difference is whether it's paired to a live node. A business uses the real tool on day one of the cheapest possible pilot, and nothing gets rebuilt when it scales.

*[design] Consider bundling a recent database snapshot so a first-time user opens the app looking at a live town rather than an empty screen. Requires a prominent "data as of" date — which every view should carry anyway.*

---

## 7. The Data Pipeline

### On the device **[proven math, design implementation]**

```
inflows + outflows (raw, on device)
        ↓
  throughput = total inflow
  each outflow → fraction of throughput, keyed by destination zip
        ↓
  ══════ raw amounts and names never cross this line ══════
        ↓
  zipsum published
```

**This line is not negotiable.** Any implementation that moves raw financial data across a network before converting to fractions is not RVEIRS, whatever it calls itself.

### The zipsum

```json
{
  "entity_id": "<stable hashed pseudonymous ID>",
  "period":    "2026-W34",
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

**Publish raw zip codes, not pre-collapsed zones.** Two deployments reading the same zipsum legitimately produce different zone vectors, because "local" is a different place for each of them. Raw zips can always be collapsed later; a collapsed vector can never be expanded back.

### Chunking

One zipsum per message, tagged with sequence and version. Fits packet limits and isolates failure — a receiver can tell "I have 1 and 3, missing 2" rather than silently holding an incomplete picture it believes is complete.

### Recursion runs on the client **[proven]**

Once a user holds their contacts' published zipsums, computing deeper layers is a handful of multiplications. Well within a phone.

No server-side computation. No scheduling. No queue.

---

## 8. Household Clustering — A Different Deployment

**[design]** — deferred. Businesses first. Read this before enabling it, not after.

Households don't publish individually. They fold into a cluster, and the cluster publishes.

**A cluster server is not a loop server.** It computes, and it briefly holds decrypted, still-identified data with throughput attached. That processing window is the one genuinely sensitive point in the whole system.

If you're running one:

- **Host it somewhere locally accountable** — a chamber, a credit union, a church, whoever your community actually trusts. Users should be able to choose which cluster gets their key, and get a rough sense of whose money they're pooling with.
- **Enforce a minimum cluster size** before computing or displaying anything. A score from two or three households is identifiable data about those specific people.
- **Never append records.** Insert each new record at a **random index**. If the roster and the throughput list both grow by appending, row N of one is the same person as row N of the other, and the entire separation is defeated by opening both files side by side. This is a security requirement, not a style preference.
- **Delete on schedule.** Individual zipsums and the prior cluster state go as soon as the fold completes. The roster goes at period close.
- **Document who has operator access.** Even if the answer is one person.

The full file structure and fold procedure are in the design guide, §9.

---

## 9. Verification

Before you trust a deployment, verify the math.

**Hand-checkable test case:**

```
Business: Main Street Diner
Home zip: 72901
Local: 72901        Close: 72903, 72904, 72956

  Employee wages          $4,000    72901    → local
  Commercial rent         $1,500    72901    → local
  Local produce supplier    $800    72903    → close
  Sysco (national)        $1,200    30328    → far
  Entergy Arkansas          $400    72201    → far
  Unknown misc              $300    (blank)  → void
  Total: $8,200

Expected:
  local_1 = 0.671   close_1 = 0.098
  far_1   = 0.195   void_1  = 0.037
  depth_1 = 0.963
```

If your pipeline produces these numbers, L1 is correct.

**Invariant to assert everywhere:** `local + close + far + void = 1.0` at every layer, for every entity, always. If it doesn't, you have a bug. Check this in code, not by eye.

**Synthetic networks:** `generate-network.py` produces full multi-hop test networks with known values, including shared-node cases where two paths converge on the same business. Run it. Verify your recursion against it before pointing it at real data.

---

## 10. Deployment Sequence

1. Zones defined, written down, agreed on.
2. Period length chosen.
3. Loop server running, recording, looping. Verify with one test node.
4. Client distributed, verified against the hand-check above.
5. Anchor businesses onboarded in person. Walk them through the first submission.
6. Watch the first two periods. Nothing about the numbers matters yet — you're watching whether the pipeline holds.
7. Grow.

Expect depth to be terrible at first. That's correct. It's an honest picture of a network that's still filling in, and it climbs as participation does.

---

## 11. Known Gaps

Things that will bite you, listed so they don't surprise you:

- **MeshCore roles are unvalidated by this project.** §4 is documentation-derived, not experience-derived.
- **Simultaneous WiFi clients per node is unknown.** Test it.
- **Transport at scale is undemonstrated.** Hundreds of participants over LoRa hasn't been done. Payloads are small, so requirements are modest — modest isn't proven.
- **Temporal merging is unsolved.** If you run different periods in different parts of one network, there is currently no correct way to merge them. Flow rates aren't quantities and don't average. Pick one period for the whole deployment.
- **Minimum cluster size has no recommended number yet.** It likely varies with population density.

---

## 12. License

AGPL (pending legal review). Free to build, run, fork, and deploy. Modified versions offered as a network service must share their changes.

Nobody can take this away from anybody. That's the point of the license, and it's the point of the project.

---

*RVEIRS Install Guide v0.2 · August 21, 2026*
*The instrument observes without judging. Absorbs without extracting. Makes visible without exposing.*
