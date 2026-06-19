# ElectricDistribution BIS Schema

A proposed [BIS](https://www.itwinjs.org/bis/guide/) domain schema for overhead electric distribution infrastructure — poles, attachments, conductors, transformers, structural loading analysis, field inspection, and clearance checking.

**Layer:** 2-DisciplinePhysical  
**Schema name:** `ElectricDistribution` · alias `edist` · version `01.00.00`  
**Status:** WIP — passes `SchemaValidationRunner` with zero violations (validated 2026-06-19); pre-PR to iTwin/bis-schemas

---

## Contents

### Physical Elements

| Class | Base | Notes |
|---|---|---|
| `DistributionPole` | `bis:PhysicalElement` + `dsys:IDistributionElement` | Primary structural host |
| `PoleAttachment` | `bis:PhysicalElement` (abstract) | Base for all pole-mounted hardware |
| `Crossarm` | `PoleAttachment` | Horizontal timber or steel arm |
| `Insulator` | `PoleAttachment` | Pin, disc, or deadend insulator |
| `SpanGuy` | `PoleAttachment` | Guy attachment point on the pole |
| `GuyWire` | `bis:PhysicalElement` | Tensioned cable from pole to anchor |
| `AnchorAssembly` | `bis:PhysicalElement` | Ground anchor terminating a guy wire |
| `OverheadConductor` | `bis:PhysicalElement` + `dsys:IDistributionFlowElement` | Primary, neutral, or secondary wire |
| `DistributionTransformer` | `bis:PhysicalElement` + `dsys:IDistributionFlowElement` | Single- or three-phase transformer |
| `PoleEquipment` | `PoleAttachment` | Capacitors, regulators, switches, arresters |

### Type Catalog

| Class | Base |
|---|---|
| `DistributionPoleType` | `bis:PhysicalType` |

### Analytical Elements

| Class | Base |
|---|---|
| `PoleLoadingAnalysis` | `anlyt:AnalyticalElement` |
| `ClearanceCheck` | `anlyt:AnalyticalElement` |

### Information Records

| Class | Base |
|---|---|
| `PoleInspection` | `bis:InformationRecordElement` |
| `ClearanceViolation` | `bis:InformationRecordElement` |

### Aspects

| Class | Kind | Host |
|---|---|---|
| `GroundingConfiguration` | UniqueAspect | `DistributionPole` |
| `EnvironmentalDesignContext` | UniqueAspect | `DistributionPole` |
| `VegetationContext` | UniqueAspect | `DistributionPole` |
| `AvianProtectionContext` | UniqueAspect | `DistributionPole` |
| `ClearanceSummary` | UniqueAspect | `DistributionPole` |
| `PoleLoadingResult` | MultiAspect | `PoleLoadingAnalysis` |
| `InspectionFinding` | MultiAspect | `PoleInspection` |
| `ConductorSagCondition` | MultiAspect | `OverheadConductor` |

### Definition Elements

`AvianZone`, `LoadCaseDefinition`, `ClearanceRequirement`

### Enumerations (25)

`PoleMaterial`, `PoleConditionRating`, `CrossarmMaterial`, `CrossarmSide`, `AnchorType`, `InsulatorType`, `ConductorUsageGroup`, `ConductorSagConditionType`, `GuyGrade`, `InspectionMethod`, `InspectionVerdict`, `FindingType`, `FindingSeverity`, `ClearanceDimension`, `ClearanceSeverity`, `AnalysisCode`, `NescGrade`, `GO95Grade`, `NescDistrict`, `GO95Zone`, `ConstructionType`, `TransformerCoolingClass`, `TransformerPhaseConfig`, `EquipmentType`, `FireThreatZone`

### Custom KindOfQuantity

| Name | Persistence unit | Notes |
|---|---|---|
| `EDIST_RESISTANCE` | `u:OHM` | Grounding resistance |
| `EDIST_RESISTIVITY` | `u:OHM_M` | Soil resistivity |

All other quantities use `AECU:` KOQs from AecUnits.

---

## Schema References

| Schema | Version | Alias |
|---|---|---|
| BisCore | 01.00.15 | `bis` |
| Analytical | 01.00.02 | `anlyt` |
| DistributionSystems | 01.00.03 | `dsys` |
| AecUnits | 01.00.03 | `AECU` |
| Units | 01.00.06 | `u` |
| Formats | 01.00.00 | `f` |
| CoreCustomAttributes | 01.00.03 | `CoreCA` |
| BisCustomAttributes | 01.00.00 | `bisCA` |

BisCore 01.00.15 is required because Analytical 01.00.02 depends on it. Units 01.00.06 is the minimum released version containing `u:OHM_M` (ELECTRICAL_RESISTIVITY).

---

## Validating Locally

```bash
# 1. Clone the BIS schemas repo alongside this one
git clone --depth 1 https://github.com/iTwin/bis-schemas.git

# 2. Install (--ignore-scripts skips the playwright postinstall)
cd bis-schemas
npm install --ignore-scripts

# 3. Place the schema under the correct layer
mkdir -p Domains/2-DisciplinePhysical/ElectricDistribution
cp ../bis-schema-electric-distribution/ElectricDistribution.ecschema.xml \
   Domains/2-DisciplinePhysical/ElectricDistribution/

# 4. Validate
node ./tools/SchemaValidation/SchemaValidationRunner.js \
     --BisRoot . --name ElectricDistribution --wip
```

Expected output: `Schema Validation Succeeded. No rule violations found.`

---

## Design

See [design.md](./design.md) for:

- Full property tables for every class
- UPM (Universal Pole Model) field mapping
- Enumeration value rationale
- Relationship diagram descriptions
- Open design decisions and alternatives considered

---

## Validated Against

The schema property set and enumeration values were validated against the data
models of the three dominant overhead distribution analysis tools:

| Tool | Vendor | Format                             |
| --- | --- |------------------------------------|
| SPIDAcalc | Bentley Systems | SPIDA/JSON (`.spida`)              |
| PLS-POLE / PLS-CADD | Bentley Systems | Text-structured (`.pole` / `.lco`) |
| O-Calc Pro | Osmose Utilities Services | PPLX/XML (`.ocalc`)                |

---

## UPM Ground Truth

Properties and enumeration values are aligned with the Universal Pole Model
(UPM), a proprietary Pydantic domain model.
Field names in the schema match UPM field names 1-to-1 so the BIC connector
can map without aliasing.

The UPM is the canonical internal representation for a suite of 140+
engineering automation tools and is covered by 825+ domain-validated tests
spanning real-world pole configurations across NESC and GO-95 jurisdictions.
This test coverage is the primary evidence that the schema's property set and
enumeration values handle production edge cases, not just nominal inputs.

Access to the UPM reference implementation is available on request.

---

## PR Target

This schema is intended for submission to
[iTwin/bis-schemas](https://github.com/iTwin/bis-schemas) under
`Domains/2-DisciplinePhysical/ElectricDistribution/`.

---

Proposed by Julius Guay IV (Bentley Systems Solutions Engineer) · [linkedin.com/in/juliusguay](https://www.linkedin.com/in/juliusguay/)
