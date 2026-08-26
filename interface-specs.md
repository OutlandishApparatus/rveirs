# RVEIRS — Interface Specification
**v0.1 · August 22, 2026**

*What the client actually looks like and does. Written against the existing demo (`eddy-calculator-v3.html`) where it exists, and specified where it doesn't yet.*

---

## 0. Scope and Status

This describes the **client** — the single-file web app a participant uses. It does not cover the loop server, mesh configuration, or cluster servers.

| Marker | Meaning |
|---|---|
| **[built]** | Exists and works in the current demo |
| **[spec]** | Designed here, not yet built |
| **[later]** | Acknowledged, deliberately deferred |

**The demo and the live version are the same software.** The only difference is whether it's paired to a mesh node. Everything below should work offline, standalone, with no network at all — the network only adds other people's data.

---

## 1. Shell

**[built]** Single page. Tabbed interface, persistent results panel.

```
┌─────────────────────────────────────────────┐
│  RVEIRS                    [☀ light] [✕ clear]│
├─────────────────────────────────────────────┤
│  [ Outflows ] [ Inflows ] [ Zone Settings ]  │
├─────────────────────────────────────────────┤
│                                             │
│              (active tab body)              │
│                                             │
├─────────────────────────────────────────────┤
│              SCORES PANEL                   │
│         (always visible, live)              │
└─────────────────────────────────────────────┘
```

**The scores panel never hides.** Whatever tab is active, the current numbers stay on screen and update as you type. This is the single most important interaction in the whole tool — you watch the score move as you enter a row, which teaches the concept without a word of explanation.

**Theme toggle** — light/dark. **[built]**
**Clear** — wipes all entered data. Should confirm before firing. **[built, confirm is spec]**

---

## 2. Tab: Outflows

The primary screen. Where the work happens.

### Layout **[built]**

A small spreadsheet — rows with an **+ add row** button beneath. Not a form-per-entry wizard; a grid you can move through quickly.

| Column | Field | Notes |
|---|---|---|
| Amount | number | placeholder `0.00` |
| Counterparty | text | placeholder `counterparty`. Optional, display-only, **never transmitted** |
| Zip | text | placeholder `zip` |
| Tag | *auto* | live LCFV label, not user-editable |
| — | ✕ | remove row |

### Live zone tagging **[built]**

As a zip is typed, the row's tag updates immediately: **LOCAL / CLOSE / FAR / VOID**, color-coded. No submit step, no recalculation button.

**Blank zip = VOID.** No separate "unknown" control. Leaving it empty is the honest answer and should feel like a normal thing to do, not an error state. Do not validate zips as required. Do not show a warning icon on void rows.

### Keyboard **[spec — currently relies on browser defaults]**

Tab moves between fields. Browser autofill handles vendor repetition — good enough, and deliberately not reimplemented. Adding custom autocomplete is a known temptation and is explicitly out of scope for v1.

**Worth adding:** Enter on the last row creates a new row and focuses its first field. Small change, large effect on entry speed.

### Import **[spec / later]**

- **Paste / CSV import** — desirable, spec'd, not built. Expected columns: `amount, name, zip`.
- **Accounting-package extractor** (QuickBooks etc.) — **[later]**. Show the button, greyed, labeled "coming soon." Signals intent and sets expectation without promising a date.

---

## 3. Tab: Inflows

**[built]** Same grid, two columns — amount and source. No zip, no tag.

Inflows exist for exactly one reason: **the total is the throughput denominator.** Nothing else uses them.

Because of that, itemization is optional. A single row with the period's total is a completely valid inflow entry, and the interface should make that obvious rather than implying a full ledger is expected.

*[later] Once the network is dense, inflows become the basis of two-part verification — your inflows are someone else's outflows. Not built, but it's why the field exists in a structured form rather than as a single number box.*

---

## 4. Tab: Zone Settings

**[built]** Configured once, rarely touched.

- **Business name** (optional) — display only
- **Home zip** — L0
- **Local zips** — list, with **+ add local zip**
- **Close zips** — list, with **+ add close zip**
- **Presets** — one-click regional configs. Currently *Carroll County / Eureka Springs* and *Fort Smith / River Valley*

`far` and `void` are automatic and not configurable.

**This config never leaves the device.** It's how *you* read data, not how you report it. Worth stating on the screen itself — it's the single most common misunderstanding about how the system works.

*[spec] Add a line of guidance near the local list: "when in doubt, draw local smaller." A tight zone with honest scores beats a generous one that flatters.*

---

## 5. Scores Panel

Always visible. Three regions.

### 5.1 The vector — four honest numbers **[built, needs relabeling]**

Local / Close / Far / Void, each with amount and percentage, plus the period total.

Currently labeled by zone name alone. **Should be explicitly labeled as Layer 1** now that deeper layers exist in the math.

**No blending.** The old `eddy_2` folded close in at half weight; that's removed. Four separate numbers, always.

### 5.2 Layer scores **[partially built — old model]**

Current demo shows `eddy_1`, `eddy_2` with a bar, depth, and a "self score." **All of this is superseded.** The replacement:

```
Layer 1    local 0.671   close 0.098   far 0.195   void 0.037   depth 0.963
Layer 2    local 0.039   close 0.029   far 0.020   void 0.912   depth 0.088
Layer 3    …

SCORE  1.20        DEPTH  2.20        LAYERS  3
```

**Score and depth are always displayed together, at the same size.** A score shown without its depth is a guess with decoration, and the interface should make separating them impossible.

*Optional, off by default:* `traced ratio` (score ÷ depth). Useful for comparing networks with very different depths. Not a headline.

### 5.3 Reading aids **[spec]**

- **Score is a multiplier, not a percentage.** Label it so. "1.20" should read as "each dollar generated $1.20 of local activity," and that sentence belongs on screen, not just in the docs.
- **Depth needs a plain-language gloss** — something like "you traced 2.2 layers of 3 attempted." Depth is the concept users misunderstand most.
- **Low depth is not an error.** Do not style it as a warning. Early networks have terrible depth and that's an honest picture, not a failure.

### 5.4 Zone weighting **[spec]**

Optional display control. Any zone, any weight, user-chosen, **off by default**, visible off switch.

Never touches the core numbers — it produces a separate, clearly-labeled blended figure alongside the honest four. In practice it's mostly unnecessary when all four are visible at once.

---

## 6. Payload Preview

**[built as an expandable panel — should become a modal]**

Currently a **▶ show payload** toggle revealing a text box. Works, but under-weights the moment.

**[spec] This should be a deliberate stop.** Before anything publishes, the user sees exactly what leaves:

```
┌────────────────────────────────────┐
│  This is what gets published       │
│                                    │
│  entity: a7f3…                     │
│  period: 2026-W34                  │
│                                    │
│    72901  →  0.6707                │
│    72903  →  0.0976                │
│    30328  →  0.1463                │
│    72201  →  0.0488                │
│    void   →  0.0366                │
│                                    │
│  No amounts. No names.             │
│  Nothing else leaves this device.  │
│                                    │
│      [ Cancel ]    [ Publish ]     │
└────────────────────────────────────┘
```

**Nothing publishes without an explicit press.** No auto-sync, no background upload, no "we'll send this later" toast.

This screen is the privacy guarantee made visible. It's worth more than any paragraph of policy text, because the user can count the fields themselves and confirm their vendors' names aren't there.

*[spec] The raw JSON should stay available behind a "show raw" link for anyone who wants it — but the default view is the human-readable list above.*

---

## 7. Map

**[spec — not built]**

Participants and the flows between them.

- **Data-as-of date, prominently.** Every view, always. A stale map is a real map of a different week — but only if the user knows which week.
- Geographic and network layouts, toggleable — same data, two arrangements.
- Node = entity. Edge = flow. Edge weight = ratio.
- Void should be *visible*, not omitted. Money leaving the known network is the most important thing on the map and the easiest thing to accidentally hide by only drawing what's traceable.

*[later] Timeline animation across periods.*

---

## 8. Cross-Cutting Rules

**Live, not submitted.** Everything recalculates as you type. There is no "calculate" button anywhere in the tool.

**Nothing leaves without a press.** One deliberate action, one clear preview. Everywhere.

**Void is never an error state.** No red, no warning icons, no validation blocking. Unknown is a legitimate, expected, honest answer and the interface's treatment of it teaches the whole philosophy.

**Score and depth are inseparable.** Never render one without the other.

**Offline-first.** Full functionality with no network. Network adds other people's data; it is never a prerequisite.

**Single file.** Distributable from a web host, a node's own WiFi, or a USB stick.

---

## 9. Open

- **Recursion UI.** L2+ needs contacts' zipsums loaded. How does a user acquire and manage those — automatic from the loop room, manual import, both? Unspecified.
- **Layer-count control.** Fixed-N (default 3), threshold, or converge. Which are exposed, and where.
- **Period selection.** Weekly default. Does the user set it, or the deployment?
- **Entity ID generation.** Where the stable hash comes from and how it survives a device change.
- **Cluster/household flows** — different enough to need their own spec.

---

*Interface Spec v0.1 · August 22, 2026*
