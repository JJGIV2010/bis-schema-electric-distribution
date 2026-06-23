# ElectricDistribution → BIS Power Schemas: Connector Impact Brief

> **Purpose of this document.** This is a **portable, self-contained** brief intended to
> be dropped into a *connector* repository (one that reads from / writes to the
> `ElectricDistribution` schema) so that repo's maintainers/agents can analyze how a
> proposed re-basing of `ElectricDistribution` onto the official BIS power schemas would
> impact their mapping code, element creation, and validation.
>
> **It describes a proposal, not a shipped change.** Nothing in `ElectricDistribution`
> has changed yet. Treat every "Proposed" row as a *candidate* breaking change to
> evaluate, not a fact on the ground. The re-base is pending coordination with the
> upstream Substation Plus team (Alfredo Contreras / Diego Diaz) and resolution of the
> open questions in §5.

**Source schema:** `ElectricDistribution` (alias `edist`), layer `2-DisciplinePhysical`.
**Target base schema:** `PowerSystemResourcesPhysical` (alias `psrp`), `iTwin/bis-schemas`,
same layer (`2-DisciplinePhysical`), version `01.00.02`, currently `NotForProduction`.
**Date:** 2026-06-23.

---

## 1. TL;DR for a connector author

Today `ElectricDistribution` derives every physical class **directly** from
`bis:PhysicalElement` (plus `dsys` mixins). The proposal re-bases those classes onto the
official `psrp` base classes (`Equipment` / `Structure` / `Foundation` and their
concrete subclasses), mirroring how `SubstationPhysical` already extends `psrp`.

**Why a connector should care, even though class *names* may not change:**

1. **Inherited properties and mixins shift.** A class that today inherits only from
   `bis:PhysicalElement` would instead inherit from, e.g., `psrp:Structure` →
   `bis:PhysicalElement` + `spi:IStructuralAssembly`. Properties/mixins you currently
   set explicitly may become inherited (or vice-versa), and new required references may
   appear.
2. **New schema references become mandatory.** The iModel/connector must import
   additional schemas (`psrp`, `spi`, `sc`) and a **higher `BisCore`** before
   `ElectricDistribution` will import.
3. **The `PoleAttachment` unifying base likely disappears.** `psrp` splits attachments
   across *different* branches (`Equipment` vs `Structure`). Any connector code that
   keys off the common `PoleAttachment` base class will break.
4. **Optional but likely: a type–instance pattern and a ports/connections model** get
   introduced, which means new element kinds the connector may be expected to create.

If your connector only ever **reads** instances by ECClass name and pulls a fixed set of
properties, impact is moderate (watch property inheritance + the `PoleAttachment` change).
If it **writes** instances, impact is higher (new references, possibly new `*PhysicalType`
and port elements).

---

## 2. The target schema stack (context)

| Schema | Alias | Version | Layer | Role |
|---|---|---|---|---|
| `DistributionSystems` | `dsys` | 01.00.02–03 | 1-Common | Mixins (`IDistributionElement`, `IDistributionFlowElement`), ports & connections. *Already referenced today.* |
| `PowerSystemResourcesPhysical` | `psrp` | 01.00.02 | 2-DisciplinePhysical | **Re-base target.** Physical equipment/structure base classes. `NotForProduction`. |
| `PowerSystemResourcesFunctional` | `psrf` | 01.00.02 | 3-DisciplineOther | Functional counterparts (not a physical-connector concern). |
| `SubstationPhysical` | `subphy` | 01.00.06 | 4-Application | Precedent: references `psrp` and subclasses it. |

`psrp` exposes three abstract roots, each carrying mixins subclasses inherit:

- **`Equipment`** *(abstract)* → `bis:PhysicalElement`, `dsys:IDistributionFlowElement`,
  `spi:IStructuralElement` — flow-carrying equipment.
- **`Structure`** *(abstract)* → `bis:PhysicalElement`, `spi:IStructuralAssembly` —
  support structures.
- **`Foundation`** *(abstract)* → `bis:PhysicalElement` — ground support.

Concrete `psrp` classes relevant here: `Pole` (→`OverheadStructure`→`Structure`),
`Conductor` (→`Equipment`), `Transformer`/`PowerTransformer` (→`Equipment`),
`Insulator` (→`AuxiliaryEquipment`→`Equipment`), `GuyWire`
(→`SupportWire`→`OverheadStructure`→`Structure`), `SpanGuy`/`DownGuy` (→`Cable`→`Structure`),
`AnchorCage` (→`UndergroundStructure`→`Structure`), `CableSupports` (→`Structure`).

---

## 3. Class-by-class impact map (the heart of the analysis)

For each `edist` class: its current base, the proposed `psrp` base, and the **connector
impact** to evaluate. "Inheritance change" almost always implies a *property-inheritance*
change — verify which properties move between explicitly-set and inherited.

| `edist` class | Current base | Proposed base (`psrp`) | Connector impact to assess |
|---|---|---|---|
| `DistributionPole` | `bis:PhysicalElement` + `dsys:IDistributionElement` | `psrp:Pole` (→`OverheadStructure`→`Structure`) | Gains `spi:IStructuralAssembly`. Check for new structural-assembly relationships the connector must populate. |
| `OverheadConductor` | `bis:PhysicalElement` + `dsys:IDistributionFlowElement` | `psrp:Conductor` (→`Equipment`) | `Equipment` already carries `IDistributionFlowElement` — the mixin you set explicitly becomes inherited. Confirm no duplicate-set issues. |
| `DistributionTransformer` | `PoleAttachment` + `dsys:IDistributionFlowElement` | `psrp:Transformer` or `psrp:PowerTransformer` | **`PoleAttachment` parent removed.** Re-point any base-class queries; matches `subphy:DistributionTransformer` precedent. |
| `Insulator` | `PoleAttachment` | `psrp:Insulator` (→`AuxiliaryEquipment`→`Equipment`) | Moves from "attachment" branch to `Equipment` branch. |
| `GuyWire` | `PoleAttachment` | `psrp:GuyWire` (→`SupportWire`→`OverheadStructure`) | Moves to `Structure` branch. |
| `SpanGuy` | `PoleAttachment` | `psrp:SpanGuy` (→`Cable`→`Structure`) | Moves to `Structure` branch. |
| `AnchorAssembly` | `bis:PhysicalElement` | `psrp:AnchorCage` (→`UndergroundStructure`) | Confirm semantic fit ("cage" vs general anchor) before re-mapping. |
| `Crossarm` | `PoleAttachment` | `psrp:CableSupports` or `psrp:Structure` | **Open — no exact `psrp` match.** Mapping may stay custom; flag for connector. |
| `PoleEquipment` | `PoleAttachment` | `psrp:AuxiliaryEquipment` / specific `Equipment` subtypes (e.g. `SurgeArrester`) | Map per equipment kind — may split one connector mapping into several. |
| `DistributionPoleType` | `bis:PhysicalType` | `psrp:PolePhysicalType` | psrp pairs instances with `*PhysicalType`; see §4. |

### Unchanged — `edist`-specific value-add (no `psrp` equivalent)

These are **not** affected by the re-base and remain `edist`-owned. A connector mapping
these keeps working as-is:

- Analytical: `PoleLoadingAnalysis`, `ClearanceCheck` (`anlyt:AnalyticalElement`).
- Information records: `PoleInspection`, `ClearanceViolation`.
- Definitions: `AvianZone`, `LoadCaseDefinition`, `ClearanceRequirement`.
- Aspects: `GroundingConfiguration`, `EnvironmentalDesignContext`, `VegetationContext`,
  `AvianProtectionContext`, `ClearanceSummary`, `PoleLoadingResult`, `InspectionFinding`,
  `ConductorSagCondition`.
- All 25 enumerations and the two custom KOQs (`EDIST_RESISTANCE`, `EDIST_RESISTIVITY`).

---

## 4. New constructs the connector may need to produce

These are *consequences* of adopting `psrp` conventions. Each is a decision (see §5), but
if adopted, a **writing** connector is affected:

1. **Type–instance pattern.** `psrp`/`SubstationPhysical` pair every instance with a
   `*PhysicalType`. Today only `DistributionPoleType` exists. Adoption could add
   `ConductorPhysicalType`, `InsulatorPhysicalType`, `TransformerPhysicalType`, etc. —
   new `bis:PhysicalType` elements the connector would create and relate.
2. **Ports / connections model.** `psrp` models electrical connectivity with
   `ElectricalPort` / `StructuralPort` + `PortConnection` (`SubstationPhysical` adds
   `LiveMember`/`GroundMember` owners). Today `edist` relies on `dsys` mixins only.
   If adopted, the connector must emit port + connection elements to represent topology.
3. **Aspect re-basing.** Aspects may move from `bis:Element*Aspect` onto
   `psrp:EquipmentAspect` / `psrp:StructureAspect`. Aspect *class names* may be stable but
   their base/owner constraints could tighten — verify the connector's aspect placement.

---

## 5. Open decisions that change the impact surface

The final blast radius depends on how these resolve. Flag each in your connector analysis
as "depends on upstream decision":

1. **`PoleAttachment` fate** — removed entirely vs. repurposed as a mixin/aspect carrying
   the shared `AttachmentHeight` / `Direction` / `InstallDate` properties. Determines
   whether those properties stay co-located or scatter onto different `psrp` branches.
2. **Whether ports/connections are adopted** (see §4.2).
3. **Whether the full type–instance pattern is adopted** (see §4.1).
4. **Aspect base classes** — keep `bis:Element*Aspect` or re-base to `psrp` aspect bases.
5. **`psrp` `NotForProduction` status** — if a production connector cannot depend on a
   `NotForProduction` schema, the re-base may be sequenced/delayed. This gates *timing*,
   not just shape.

---

## 6. Reference & version changes (the import-time impact)

If the re-base proceeds, the iModel the connector targets must satisfy these *before*
`ElectricDistribution` will import. A connector that provisions schemas needs to handle:

- **Add** `ECSchemaReference name="PowerSystemResourcesPhysical" ... alias="psrp"`.
- **Bump `BisCore` 01.00.15 → 01.00.25** (`psrp`'s floor). *This is the most likely
  hard blocker for an existing connector/iModel — verify the target BisCore.*
- **Add transitive references** `StructuralPhysicalInterop` (`spi`) and
  `StructuralConnections` (`sc`) that `psrp` pulls in.
- **Reconcile `DistributionSystems`** — `edist` references 01.00.03; `psrp` references
  01.00.02. The higher compatible version wins; confirm the connector's `dsys` usage is
  compatible.
- After re-base, re-run schema validation; expect base-class and mixin-inheritance
  changes to ripple through relationship constraints (and therefore any constraint a
  connector relies on when inserting relationships).

---

## 7. Suggested analysis checklist for the connector repo

Hand this list to the connector repo's reviewer/agent:

- [ ] Does any code query or branch on the **`PoleAttachment`** base class? (Will break.)
- [ ] Does any code rely on a property being **explicitly set** that would become
      **inherited** (or move to a different branch) under the new bases?
- [ ] Does the target iModel/profile meet **`BisCore` 01.00.25** and have `psrp`/`spi`/`sc`
      available to import?
- [ ] Does the connector **write** instances of any re-based class? List them and confirm
      the new base-class constraints (mixins, relationships) are satisfiable.
- [ ] Will the connector be expected to create **`*PhysicalType`** instances or
      **port/connection** elements (decisions §4)? If yes, scope that work.
- [ ] Are any **aspects** placed/owned in a way that conflicts with `psrp` aspect bases?
- [ ] Confirm none of the **unchanged `edist`-specific** classes (§3) are in scope — those
      need no connector changes.
- [ ] Note the **`NotForProduction`** gate: does your connector's release process allow a
      dependency on a non-production schema?

---

## Appendix — provenance

- Re-base target verified against `iTwin/bis-schemas@master`:
  `Domains/2-DisciplinePhysical/PowerSystemResources/PowerSystemResourcesPhysical.ecschema.xml`,
  `Domains/1-Common/DistributionSystems/DistributionSystems.ecschema.xml`,
  `Domains/4-Application/Substation/SubstationPhysical.ecschema.xml`.
- Source schema: `ElectricDistribution.ecschema.xml`.
- All base-class chains were verified directly against the schema XML.
- This brief is derived from the internal analysis `power-schema-integration.md` and
  reframed for a connector-impact audience. **No schema changes have been made.**
