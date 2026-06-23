# DRAFT reply for PR #733 — not yet posted

> This is a **draft** for review. Nothing has been posted to GitHub. Post manually once
> you're happy with the wording. Intended as a reply to `@alfredocontreras88`'s comment.

---

Thanks @alfredocontreras88 — completely agree, and I'd rather extend the existing base than
add a parallel power hierarchy.

I dug into the repo and found `PowerSystemResourcesPhysical` (`psrp`) under
`Domains/2-DisciplinePhysical/PowerSystemResources` — same layer as this proposal — which
already defines the base classes I was effectively re-implementing (`Pole`, `Conductor`,
`Transformer`/`PowerTransformer`, `Insulator`, `GuyWire`, `SpanGuy`, `AnchorCage`, plus the
`Equipment`/`Structure`/`Foundation` roots). I also see `SubstationPhysical` extends `psrp`
directly (e.g. `DistributionTransformer` → `psrp:PowerTransformer`), so that's a clear
pattern to follow.

My plan is to re-base `ElectricDistribution` so its physical classes subclass `psrp`
instead of `bis:PhysicalElement`, and keep only the genuinely distribution-specific content
on top (NESC/GO-95 structural loading and clearance analysis, field inspection, vegetation
and avian-protection context, grounding). A few things I'd like to align on before I rework it:

1. A couple of mappings need a decision — e.g. `Crossarm` has no exact `psrp` equivalent
   (`CableSupports` vs a new `Structure` subtype), and whether pole-mounted hardware should
   adopt the `ElectricalPort`/`StructuralPort` connectivity model rather than just the
   `dsys` mixins.
2. `psrp` is currently marked `NotForProduction` — is there a timeline for it going
   production, given I'd be taking a dependency on it?
3. Whether you'd want matching `*PhysicalType` catalog classes (Conductor, Insulator,
   Transformer) to stay consistent with `psrp`/Substation.

Happy to set up a call with you and Diego Diaz to walk through the class mapping — I've
written up a full class-by-class proposal I can share ahead of time. What's the best way to
reach you both?
