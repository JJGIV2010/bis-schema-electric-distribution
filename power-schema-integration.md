# Integrating ElectricDistribution with the Official BIS Power Schemas

**Status:** Analysis & alignment proposal — *no schema changes made yet.*
**Trigger:** PR review comment on [iTwin/bis-schemas#733](https://github.com/iTwin/bis-schemas/pull/733).
**Date:** 2026-06-23

---

## 1. Summary

On PR #733, **Alfredo Contreras** (`@alfredocontreras88`, Substation Plus team) wrote:

> "if creating or adding a new power system schema, please coordinate with other
> interested parties like substation plus team. we already have a base schema that
> should be extended for other power sub domains. you can contact me directly and
> should be coordinated with Diego Diaz."

He is right, and the "base schema" is concrete: **`PowerSystemResourcesPhysical`**
(alias `psrp`) already lives in `iTwin/bis-schemas` at the *same layer as ours*
(`2-DisciplinePhysical`). It defines abstract, subclassable base classes for the entire
power physical domain — including `Pole`, `Conductor`, `Transformer`, `Insulator`,
`GuyWire`, `SpanGuy`, and `AnchorCage`.

**The core finding:** our `ElectricDistribution` schema currently derives every class
directly from `bis:PhysicalElement` (+ `dsys` mixins) and re-invents concepts `psrp`
already provides. This creates a *parallel, duplicate* power hierarchy in the very repo
that already has a power base schema — exactly the disconnect Alfredo flagged.

**Recommendation (short version):** Re-base `ElectricDistribution` so it *extends* `psrp`
(following the proven `SubstationPhysical` precedent) rather than paralleling it — but
**coordinate with Alfredo and Diego Diaz first**, because `psrp` is currently marked
`NotForProduction` and the re-base involves real architectural decisions (below).

---

## 2. The existing official power schema stack

| Schema | Alias | Version | Layer | Role |
|---|---|---|---|---|
| `DistributionSystems` | `dsys` | 01.00.02–03 | 1-Common | Mixins: `IDistributionElement`, `IDistributionFlowElement`; ports & connections. *We already reference this.* |
| `PowerSystemResourcesPhysical` | `psrp` | 01.00.02 | 2-DisciplinePhysical | **The base schema Alfredo means.** Physical equipment/structure base classes. |
| `PowerSystemResourcesFunctional` | `psrf` | 01.00.02 | 3-DisciplineOther | Functional (schematic/connectivity) counterparts. |
| `SubstationPhysical` | `subphy` | 01.00.06 | 4-Application | Substation Plus product schema. **The precedent**: it references `psrp` and subclasses it. |

### What `psrp` already provides (verified from the schema XML)

Three abstract roots, each carrying the right mixins so subclasses inherit them:

- **`Equipment`** *(abstract)* → `bis:PhysicalElement`, `dsys:IDistributionFlowElement`,
  `spi:IStructuralElement` — active flow-carrying equipment.
- **`Structure`** *(abstract)* → `bis:PhysicalElement`, `spi:IStructuralAssembly` —
  support structures.
- **`Foundation`** *(abstract)* → `bis:PhysicalElement` — ground support.

Concrete classes already defined under them (all directly relevant to us):

| psrp class | Inheritance chain |
|---|---|
| `Pole` | `OverheadStructure` → `Structure` |
| `Conductor` | `Equipment` |
| `Transformer`, `PowerTransformer` | `Equipment` |
| `Insulator` | `AuxiliaryEquipment` → `Equipment` |
| `GuyWire` | `SupportWire` → `OverheadStructure` → `Structure` |
| `SpanGuy`, `DownGuy` | `Cable` → `Structure` |
| `AnchorCage` | `UndergroundStructure` → `Structure` |
| `CableSupports` | `Structure` |

### The precedent: how `SubstationPhysical` extends `psrp`

`SubstationPhysical` references `psrp` (`version="01.00.01" alias="psrp"`) and subclasses
its base classes for product-specific types — for example:

- `subphy:DistributionTransformer` → `psrp:PowerTransformer`
- `subphy:PowerCable` → `psrp:Conductor`
- `subphy:Supports` → `psrp:Structure`

This is the exact pattern `ElectricDistribution` should mirror.

> **Caveat:** `psrp` carries `<SupportedUse>NotForProduction</SupportedUse>`. That status
> directly affects whether a production-intended domain schema should base on it today —
> a key question for the coordination conversation.

---

## 3. Class-by-class mapping: `edist` → `psrp`

Today every `edist` physical class derives from `bis:PhysicalElement` (some add `dsys`
mixins). Proposed re-basing onto `psrp`:

| `edist` class | Current base | Proposed base (`psrp`) | Notes |
|---|---|---|---|
| `DistributionPole` | `bis:PhysicalElement` + `dsys:IDistributionElement` | **`psrp:Pole`** (→ `OverheadStructure` → `Structure`) | Inherits structural assembly mixin from `Structure`. |
| `OverheadConductor` | `bis:PhysicalElement` + `dsys:IDistributionFlowElement` | **`psrp:Conductor`** (→ `Equipment`) | `Equipment` already carries `IDistributionFlowElement`. |
| `DistributionTransformer` | `PoleAttachment` + `dsys:IDistributionFlowElement` | **`psrp:Transformer`** or **`psrp:PowerTransformer`** | Matches `subphy:DistributionTransformer` precedent. |
| `Insulator` | `PoleAttachment` | **`psrp:Insulator`** (→ `AuxiliaryEquipment`) | |
| `GuyWire` | `PoleAttachment` | **`psrp:GuyWire`** (→ `SupportWire` → `OverheadStructure`) | |
| `SpanGuy` | `PoleAttachment` | **`psrp:SpanGuy`** (→ `Cable` → `Structure`) | |
| `AnchorAssembly` | `bis:PhysicalElement` | **`psrp:AnchorCage`** (→ `UndergroundStructure`) | Confirm semantic fit ("cage" vs general anchor). |
| `Crossarm` | `PoleAttachment` | **`psrp:CableSupports`** or **`psrp:Structure`** | *Open — no exact `psrp` match.* |
| `PoleEquipment` | `PoleAttachment` | **`psrp:AuxiliaryEquipment`** / specific `Equipment` subtypes (e.g. `SurgeArrester`) | Map per equipment kind. |
| `DistributionPoleType` | `bis:PhysicalType` | **`psrp:PolePhysicalType`** | psrp pairs instances with `*PhysicalType`. |

**Keep `edist`-specific (no `psrp` equivalent — these are our genuine value-add):**

- Analytical: `PoleLoadingAnalysis`, `ClearanceCheck` (`anlyt:AnalyticalElement`)
- Information records: `PoleInspection`, `ClearanceViolation`
- Definitions: `AvianZone`, `LoadCaseDefinition`, `ClearanceRequirement`
- Aspects: `GroundingConfiguration`, `EnvironmentalDesignContext`, `VegetationContext`,
  `AvianProtectionContext`, `ClearanceSummary`, `PoleLoadingResult`, `InspectionFinding`,
  `ConductorSagCondition`
- All 25 enumerations and the two custom KOQs (`EDIST_RESISTANCE`, `EDIST_RESISTIVITY`)

These NESC/GO-95 engineering, inspection, clearance, vegetation, and avian-protection
concepts are precisely what `ElectricDistribution` contributes *on top of* the `psrp`
physical base — they are not duplicated anywhere upstream.

---

## 4. Architectural decisions to resolve with Alfredo / Diego

These are not mechanical; they need the team's input before any re-base.

1. **`PoleAttachment` vs. the `Equipment`/`Structure` split.** Our `PoleAttachment`
   abstract base unifies insulators, crossarms, guys, transformers, and equipment under
   one parent. `psrp` splits these across *different* branches — `Insulator`/`Transformer`
   are `Equipment`; `GuyWire`/`SpanGuy` are `Structure`. Re-basing each child onto its
   natural `psrp` branch means **`PoleAttachment` likely goes away** (or is repurposed as
   a mixin/aspect that just carries the shared `AttachmentHeight`/`Direction`/`InstallDate`
   properties).
2. **Connectivity model.** `psrp` models electrical connectivity with
   `ElectricalPort` / `StructuralPort` + `PortConnection`, and `SubstationPhysical` adds
   `LiveMember` / `GroundMember` owners. We currently rely on `dsys` mixins only.
   Decide whether to adopt ports for cross-product interoperability.
3. **Type–instance pattern.** `psrp`/`SubstationPhysical` pair every instance with a
   `*PhysicalType`. We only have `DistributionPoleType`. Decide whether to add
   `ConductorPhysicalType`, `InsulatorPhysicalType`, `TransformerPhysicalType`, etc.
4. **Aspect base classes.** Keep our aspects on `bis:Element*Aspect`, or re-base onto
   `psrp:EquipmentAspect` / `psrp:StructureAspect` for consistency with the domain.
5. **`NotForProduction` status.** Confirm `psrp`'s production roadmap/timeline. If a
   production schema cannot yet depend on it, agree on sequencing.

---

## 5. Reference / version changes required *if* we re-base

(For execution after coordination concludes — listed here so the impact is visible.)

- Add `ECSchemaReference name="PowerSystemResourcesPhysical" ... alias="psrp"`.
- Bump `BisCore` **01.00.15 → 01.00.25** (`psrp`'s floor).
- Add transitive references `StructuralPhysicalInterop` (`spi`) and
  `StructuralConnections` (`sc`), which `psrp` pulls in.
- Reconcile `DistributionSystems` version — we reference 01.00.03; `psrp` references
  01.00.02. Use the higher compatible version.
- Re-run `SchemaValidationRunner --wip` after the re-base; expect base-class and
  mixin-inheritance changes to ripple through relationship constraints.

---

## 6. Recommendation

1. **Coordinate first.** Reply on PR #733 (draft in `pr733-reply-draft.md`), connect with
   Alfredo and Diego Diaz, and resolve the five decisions in §4 — especially the
   `NotForProduction` status of `psrp`.
2. **Then re-base `edist` to extend `psrp`,** mirroring the `SubstationPhysical` pattern,
   keeping our analytical/inspection/clearance/vegetation/avian content as the
   distribution-specific layer.
3. **Address the PR's secondary blockers in parallel** (independent of the above):
   - CLA still pending (`@JJGIV2010` confirming internal IP authorization).
   - `@naveedkhan8067` notes the PR is from a **fork**, which blocks several checks, and
     asks to branch from `master` instead. Re-target to a branch on the upstream repo if
     possible.

---

## Appendix — sources

- PR comments: [iTwin/bis-schemas#733](https://github.com/iTwin/bis-schemas/pull/733)
- Schemas inspected (from `iTwin/bis-schemas@master`):
  `Domains/2-DisciplinePhysical/PowerSystemResources/PowerSystemResourcesPhysical.ecschema.xml`,
  `Domains/3-DisciplineOther/PowerSystemResources/PowerSystemResourcesFunctional.ecschema.xml`,
  `Domains/1-Common/DistributionSystems/DistributionSystems.ecschema.xml`,
  `Domains/4-Application/Substation/SubstationPhysical.ecschema.xml`
- Our schema: `ElectricDistribution.ecschema.xml`, `design.md`
- All base-class chains above were verified directly against the schema XML.
