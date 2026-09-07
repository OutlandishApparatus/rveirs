# RVEIRS — User Guide
**v0.2 · August 21, 2026**
*Where does the money go?*

---

## Where Does the Money Go?

Your town has an economy. You already know this. You live in it.

Money comes in — paychecks, tourists, contracts, transfers. Money goes out — rent, supplies, utilities, food. Somewhere in between, your town either grows or it doesn't. Businesses either stay or they don't. Your kids either have somewhere to work when they grow up or they don't.

Nobody knows where the money goes.

Not the mayor. Not the chamber of commerce. Not the bank. Not the business owner writing the checks. The data that exists is survey data — someone asked someone who guessed. Or it's model data — someone ran a simulation on national averages that may or may not describe your specific town at all.

There is no instrument. There has never been an instrument.

This is that instrument.

---

## What It Isn't

Before anything else, the short list. It saves time.

Not a company. Not a nonprofit. Not fintech. Not a protest. Not a policy proposal. Not an app that touches your bank.

It doesn't want your account numbers, your allegiance, your vote, or your money. It won't ask you to restructure your business, incorporate differently, join anything, or agree with anyone about anything.

It's a way to finally see something nobody has ever been able to see.

---

## The Instrument

Small, cheap, community-owned. The hardware is about the size of a deck of cards and plugs into a wall at a participating business. It sits there quietly.

When you want to report where your money went — suppliers, rent, utilities, whatever — you open a page on your phone, enter zip codes, and submit. That's it. No amounts leave. No names. No account numbers. Just: money went here, and here, and here.

Put enough of those on a map and you can see the economy. Not a model of it. Not a survey approximation. The actual shape of where the money flows.

---

## The Five Screens

The whole tool is five screens. You'll use two of them regularly.

### 1. Setup — done once

Your business name and your home zip.

Then your **zone config** — which zip codes you consider *local* and which you consider *close*. Everything else with a known zip is *far*, and anything with no destination at all is *void*. Those last two are automatic.

There's no correct answer for local and close. It's a judgment about your own town. Two rules of thumb: when in doubt, draw local **smaller** — a tight zone with honest numbers tells you more than a generous one that flatters. And once you pick, leave it alone; changing it mid-year makes your own scores incomparable to each other.

Your deployment probably ships with a preset for your area. Start there, adjust if it's wrong.

**This config never leaves your device.** It's how *you* read the data, not how you report it.

### 2. Entry — the one you'll actually use

Two lists.

**Money in.** What came in this period. You don't need it itemized — a total is fine. This is only used as the denominator, and then it's gone.

**Money out.** Each payment: an amount, a zip code, and optionally a name for your own reference. The name is display-only and never leaves your device.

Don't know a zip? Leave it blank. That's not a failure — blank means *void*, and void is real information. It's the part of your money you can't account for, and knowing how big that is matters as much as anything else on the screen.

The tool colors each row as you type so you can see where a payment lands before you finish entering it.

### 3. Scores — what comes back

Covered in the next section. This is the screen you'll look at when you're curious rather than when you're working.

### 4. Payload — what's about to leave

Before anything is published, you see exactly what's going out. A list of zip codes and fractions. That's the whole thing.

**Nothing publishes without you pressing the button.** Read it, then push it.

### 5. Map — the town

Everyone who's participating, and how the money moves between them. Interesting immediately, genuinely useful once enough of the town is on it.

Every view carries a **data as of** date. Always check it. A stale map is still a real map, but it's a map of a different week.

---

## What the Numbers Mean

### Four numbers, not one

Your spending splits four ways:

| | |
|---|---|
| **local** | stayed in your local zone |
| **close** | stayed regional |
| **far** | went somewhere known, but out of the area |
| **void** | no known destination |

They always add to 1.0. Nothing is hidden and nothing is blended together for you — an older version of this tool quietly counted a "close" dollar as half a "local" one, which is a judgment call dressed up as arithmetic. If you want to weight them, there's a control for it, and it's off unless you turn it on.

### The score

Once enough of your suppliers are on the network, the tool follows your money past the first hop.

Your dollar leaves. Some of it lands local. The businesses it landed on spend it too, and some of *that* stays local. And so on.

Add up the local portion at every step and you get a **multiplier**. Not a percentage — a factor.

> **1.2** means each dollar you spent generated $1.20 of local activity before it finally leaked away.

Bigger is better. There's no ceiling. **Temperature, not a grade.**

### Depth — the number that keeps the score honest

Every layer, some money disappears into void — paid to someone who isn't reporting, or with no destination recorded at all. Depth is how much you actually managed to track.

A depth of 2.2 across three layers means you traced 2.2 layers' worth of real flow. The rest was smoke.

**Always read them together.** A score without its depth is a guess with decoration. A 1.2 built on a depth of 2.2 is a real measurement. A 1.2 built on a depth of 0.4 is mostly noise wearing a number.

Low depth early on isn't a bug. It's an honest picture of a network that's still filling in. As your suppliers join, depth climbs and the picture sharpens.

**Incomplete and honest beats complete and estimated.** That's not a slogan here, it's the number.

---

## Follow the Dollar

Two restaurants. Same town. Very different scores.

**Mama's Diner.** Linda's been behind the counter for 22 years. Everybody knows her. The food is good, the prices are fair, and when something in the community needs support, Linda's name is on the donation list.

Her money leaves fast.

Not because Linda doesn't care. Because Sysco is cheaper than the local distributor. Because her landlord lives in Dallas. Because the supply chain she inherited wasn't built for her town — it was built for efficiency at scale.

**Rio Rojo.** Dallas-owned chain. The sign is corporate, the menu is corporate, the ownership is definitely not local.

Their money sticks around.

Manager Carlos sources produce locally because it's fresher and the relationships are good. What he spends gets spent again by those suppliers, and again by theirs. Four hops before it finally leaks out.

---

**The instrument doesn't care who owns the building. It measures where the money goes.**

That's the whole point. And it's why a low score isn't an accusation. It might mean your supply chain was built somewhere else and you inherited it. It might mean there's no local alternative for something you need.

The score shows you the shape of the problem. What you do with that is yours.

A high score at a chain and a low score at a beloved local institution is not a contradiction. It's information.

Follow the dollar. See what you see.

---

## Privacy by Math

Most privacy promises sound like this: *we take your data seriously and will never share it with third parties.*

That's a policy. Policies get changed, subpoenaed, hacked, or quietly updated on a Tuesday when nobody's looking.

This is different.

The network never receives dollar amounts. Never receives vendor names. Never receives anything that could identify a transaction. Not because anyone promised to delete it — because it never travels in the first place. Your device does the math. Fractions leave. Everything else stays home.

**There's nothing to leak because there's nothing there.**

And here's the part that surprises people: your suppliers never have to tell you who *their* suppliers are, and you still get an accurate number. The arithmetic works out so that a business's published summary already contains everything you'd need from the detail underneath it. Nobody upstream ever needs the itemized version.

That's not a promise. It's multiplication.

---

## Reporting Rhythm

Report once a period — a week in most deployments.

Each period has a closing date. Report before it and you're in. Miss it and you catch the next one; nothing accumulates or backfills. That keeps the whole thing a snapshot of something moving, rather than a permanent ledger of everyone's business.

Results follow about a period behind. Log a week, collect a week, see it on the map in two.

---

## Get Involved

Participation is simple. Join, report where your money went once a week, get your scores back. That's the whole transaction. No fees. No lock-in. Nobody owns your data but you.

The map gets more useful with every participant. One business is a data point. A neighborhood is a pattern. A whole town is something a community can actually use — to decide where to spend, who to source from, what to ask for, what to build next.

If you want to run your own instance for your own community, the install guide is the next document. If you want to know exactly how the math works, the design guide has all of it, including the parts that aren't finished.

If you want to support the project, pay what you can.

**The map is already being drawn. Come help fill it in.**

---

*RVEIRS User Guide v0.2 · August 21, 2026*
