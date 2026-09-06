# rveirs zipsum calculator — v0.3

Computes a business's local / close / far / void breakdown from its own outflows,
exports zipsum files, imports counterparty zipsums, and recurses (L2+, depth, eddy pair).
Everything runs on the device. Nothing is transmitted anywhere.

Built to `spec_zipsum-calculator-v4.md`. Supersedes the eddy-calculator v3 demo but does
not replace it — v3 stays published as the dated, honest artifact it is.

## Run it

- Quickest: open `index.html` in any browser. Works from a file, works offline.
- As a PWA: host the folder on any static server over https, open it once, and the
  service worker makes it fully offline; browsers will offer install-to-home-screen.

## What's real and what's placeholder

- The math is tested against the generated Carroll County network (seed 42):
  Harvest Table L1 = 0.774 / 0.000 / 0.226 / 0.000, depth 2 after imports,
  shared-node convergence sums correctly.
- The city/county crosswalk is a **placeholder** covering Carroll, Sebastian, and
  Crawford counties, unverified, pending a real HUD import. The resolved zip list is
  always inspectable in the Zones panel. "My state" uses standard USPS prefix ranges.
- Imported figures are displayed as reported, never as confirmed. No verification
  exists yet by design.

## Try the import flow

Load the "Harvest Table" example, then drop the four files from `sample_imports/`
onto the import box. L2 appears, depth goes to 2, and the eddy pair arrives.

## File format (zipsum v0.3)

    zipsum,v0.3
    name,Harvest Table Restaurant
    zip,72632
    period,week,2026-W36
    zones,local=custom,close=custom
    rows,day,rate,zip
    2026-09-01,0.232519,72632
    ...
    end,zipsum

Rates are shares of the period's outflow. No dollar amounts appear anywhere. Plain text,
readable without the software.

**Removed in v0.2:** `layer` (position is derived from the data, not declared), `depth`
and `eddy` (the sender's view of a network the receiver recomputes from their own
position — and a small leak about who else the sender has been talking to),
`includeclose` (a display setting for an eddy no longer in the file).

**Removed in v0.3:** `throughput`. It bought weighted aggregates and clustering, both of
which need a tier that aggregates across firms — which a serverless per-device tool does
not have. So it purchased nothing here while being the largest leak in the file: rate ×
throughput is an exact dollar figure into a zip, and in a thinly populated zip that names
the recipient and the amount. Reintroduce it only alongside a tier that can actually
spend it.

Older files still import. Removed fields are read and discarded, and the import list says
so.

## Zones are a network-level choice

A zonal statistic is only comparable to another one drawn on the same partition. Move the
lines and the same books produce a different number — the modifiable areal unit problem,
in an unusually direct form, since here the lines are chosen rather than inherited.

The scope this instrument claims: within one network with fixed zones, firms are
comparable to each other, and a firm is comparable to itself over time. Across networks
drawn differently, they are not, and nothing here claims otherwise. Importing a file whose
zone tokens differ from yours raises a warning — its rows are still usable, since they are
raw zips, but its published figures are not comparable to yours.

## Attribution is approximate past L2

A zipsum row names a zip, not a business. When several of your counterparties send money
into the same zip, nothing in the data says which of them paid the business whose file
you are holding, or whether more than one did. Deep matches sum every inbound path and
are labelled approximate in the import list.

This is the privacy model showing up as a limit on attribution. The same arithmetic that
makes a record safe to publish is what makes entity-level tracing impossible. It is a
property, not a defect — but it means L3 and deeper read as a regional shape rather than
a supply chain.

AGPL. v0.3.
