# IBM FlashSystem SV 9.1.2 Sizing Calculator

## Technical Methodology, Architecture, and Validation Guide

**Applies to:** `fs-sv912-calculator-v6.html`  
**Document purpose:** Explain what the calculator is designed to do, how its calculations work, how Storage Insights data is incorporated, what the outputs mean, and where validation is still required.

---

## 1. Purpose and Design Intent

The calculator began as a lightweight pre-sales aid for IBM peers who needed a fast way to create a ballpark FlashSystem sizing from incomplete customer information. The original design principle was simple:

> Enter what is known, allow clearly labeled defaults to fill the gaps, and show enough arithmetic that another technical person can follow the result.

The application subsequently expanded in two directions:

1. **Does-it-fit assessment**  
   It evaluates whether a proposed workload fits within the selected FlashSystem model across four separate dimensions: local performance, effective capacity, replication, and system memory.

2. **Measured-data interpretation and client visualization**  
   It imports IBM Storage Insights performance CSV files, aggregates multiple systems, identifies peak operating points, and places those measured values onto explanatory performance and headroom charts that can be used in client discussions and proposals.

The result is not intended to replace IBM Storage Modeller, IBM Storage Insights, IBM Configurator, or formal configuration-limit documentation. It is a transparent teaching, estimation, and communication tool that sits between initial discovery and authoritative sizing.

### 1.1 Primary objectives

The application is intended to help a technical seller or architect:

- Create an initial sizing when only partial workload information is available.
- Separate customer-provided values from assumed defaults.
- Determine whether a workload appears to fit a selected FS9600, FS7600, or FS5600.
- Identify the likely binding constraint rather than relying on a single headline specification.
- Explain the relationship among IOPS, transfer size, throughput, latency, capacity, and replication bandwidth.
- Import measured Storage Insights data instead of relying only on estimates.
- Compare measured current-state peaks with a proposed system's comfort target and saturation point.
- Export charts suitable for proposals, presentations, and technical discussions.

### 1.2 What the tool does not do

The application does not:

- Produce an authoritative IBM sizing commitment.
- Generate a final bill of materials.
- Model every workload characteristic that affects controller utilization or latency.
- Predict application latency as an SLA.
- Quantify controller overhead added by active PBR or PBHA replication.
- Replace measured telemetry or a matched IBM Storage Modeller run.
- Model the recovery system as a separate configuration.
- Model initial replication seeding time.
- Validate host, fabric, port, protocol, queue-depth, or pathing limits.

---

## 2. Application Structure

The v6 application is a self-contained HTML file with embedded CSS and JavaScript. It runs locally in a modern browser and does not require a server or external dependency.

The application has four functional layers:

1. **Workload input and assumptions**
2. **Four fit-assessment cards**
3. **Performance scaling and measured-data visualization**
4. **Proposal and chart export**

### 2.1 Workload inputs

The main workload inputs are:

- FlashSystem model
- Total IOPS
- Read/write percentage
- Read transfer size
- Write transfer size
- Data today
- Capacity growth
- Data-reduction ratio
- Target capacity utilization
- Round-trip time to the recovery site
- WAN link speed
- Replication stream-compression ratio

When an input is blank, the application can use a model or shared default. Assumed values are visually identified so that the user can distinguish measured or customer-provided facts from planning assumptions.

### 2.2 Model defaults

| Model | Machine type | Datasheet 4 KiB read-hit peak | Mixed-workload saturation | Base service time | Bandwidth ceiling | Maximum effective capacity | Node memory |
|---|---:|---:|---:|---:|---:|---:|---:|
| FlashSystem 9600 | 5078-A40 | 6,300,000 IOPS | 1,500,000 IOPS | 0.113 ms | 86 GB/s | 11.8 PBe | 1,536 GB |
| FlashSystem 7600 | 5075-A30 | 4,300,000 IOPS | 850,000 IOPS | 0.125 ms | 55 GB/s | 7.2 PBe | 768 GB |
| FlashSystem 5600 | 5127-A20 | 2,600,000 IOPS | 450,000 IOPS | 0.140 ms | 30 GB/s | 2.5 PBe | 512 GB |

The FS9600 mixed-workload saturation and base-service-time model were calibrated against a Storage Modeller reference workload. The FS7600 and FS5600 values are scaled estimates and must be treated as overridable planning values rather than published hardware specifications.

### 2.3 Shared defaults

| Input | Default |
|---|---:|
| Comfort target | 60% of mixed-workload saturation |
| PBHA synchronous RTT boundary | 1 ms |
| PBR asynchronous maximum RTT | 80 ms |
| Read percentage | 70% |
| Read transfer size | 8 KiB |
| Write transfer size | Same as read unless entered separately |
| Replication stream compression | 2:1 |
| Data-reduction ratio | 3:1 |
| Capacity target utilization | 80% |
| PBR change-volume reserve | 10% |
| Usable WAN percentage | 80% |
| OS and metadata memory reserve | 20% |
| Write-cache allocation | 15% of usable memory |

Defaults make a first-pass estimate possible, but they do not become facts. Any externally shared result should identify which influential inputs were assumed.

---

## 3. Overall Calculation Flow

The calculator follows this sequence:

1. Select the FlashSystem model and load its default ceilings.
2. Resolve entered values and defaults.
3. Split total IOPS into reads and writes.
4. Convert read and write operations into throughput using their respective transfer sizes.
5. Evaluate performance against mixed-workload saturation and bandwidth ceilings.
6. Evaluate effective-capacity demand, growth, headroom, and any PBR reserve.
7. Select a replication mode from RTT and compare the write-change rate with usable link capacity.
8. Estimate cache allocation and synchronous buffer impact.
9. Draw current and projected points on model curves.
10. If Storage Insights CSVs are imported, aggregate measured systems by aligned timestamps, find peaks, and overlay those peaks on the charts.

Each card can operate with partial data. Partial operation is intentional, but a partial result must not be mistaken for a complete design validation.

---

## 4. Card 1: Local Performance

### 4.1 Question answered

> Can the selected system handle the proposed host workload with reasonable operating headroom?

### 4.2 IOPS split

```text
Read IOPS  = Total IOPS x Read %
Write IOPS = Total IOPS x Write %
Write %    = 100% - Read %
```

Either read percentage or write percentage may be entered. The application calculates the complementary value.

### 4.3 Transfer-size-aware throughput

Read and write throughput are calculated separately because write transfer size may differ from read transfer size.

```text
Read MB/s  = Read IOPS x Read KiB x 1024 / 1,000,000
Write MB/s = Write IOPS x Write KiB x 1024 / 1,000,000
Total MB/s = Read MB/s + Write MB/s
Total GB/s = Total MB/s / 1000
```

The application uses binary KiB for operation size and displays the resulting byte rate in decimal MB/s. It also exposes a MiB/s equivalent in the performance card for comparison with tools that report binary throughput units.

### 4.4 Mixed-workload saturation

The primary performance denominator is not the datasheet's best-case 4 KiB read-hit peak. The application instead uses a mixed-workload saturation value representing the point at which controller resources are treated as fully consumed for the modeled workload.

```text
Core utilization = Total IOPS / Mixed-workload saturation x 100
Comfortable IOPS ceiling = Mixed-workload saturation x Comfort target
```

The datasheet IOPS value remains visible as a reference, but it is not used as the fit denominator.

### 4.5 Bandwidth utilization

```text
Bandwidth utilization = Total GB/s / Model bandwidth ceiling x 100
```

The performance verdict uses the higher of core utilization and bandwidth utilization. This prevents a large-block workload from appearing safe merely because its IOPS count is low.

### 4.6 Performance verdict thresholds

| Worst utilization | Verdict |
|---:|---|
| 0% through 60% | FITS |
| Greater than 60% through 80% | MARGINAL |
| Greater than 80% | OVER LIMIT |

These performance thresholds are intentionally more conservative than the general 70/90 thresholds used by the other cards.

### 4.7 Estimated latency curve

The application uses a simplified queueing response-time relationship:

```text
Estimated latency = Base service time / (1 - utilization)

where:
utilization = workload / saturation
```

Utilization is clamped just below 1.0 to keep the function finite at the saturation boundary.

The curve is useful for showing the shape of the latency knee and the relative runway among models. It is not a predicted customer latency or guaranteed response time. Actual latency depends on workload locality, cache behavior, sequentiality, queue depth, protocol, firmware, media behavior, host behavior, and many other factors that are not represented by a single equation.

### 4.8 Optional workload sensitivity

The application can calculate a separate sensitivity-adjusted saturation estimate from optional cache-efficiency inputs.

```text
Blended cache hit = Sequential fraction x Sequential cache efficiency
                  + Random fraction x Random cache efficiency

Sensitivity factor = clamp(1 + (Blended cache hit - 0.5) x 0.4, 0.75, 1.25)

Sensitivity-adjusted saturation = Standard saturation x Sensitivity factor
```

This does not replace the standard result. It is deliberately bounded and labeled as an estimate. Cache hit rate is a measured behavior, not something that can be reliably predicted from simple workload descriptors.

### 4.9 Performance limitations

The performance card reflects host I/O only. It does not add controller CPU consumption for:

- PBR change tracking
- PBR journaling
- PBR cycling snapshots
- PBHA processing
- Background migration
- Rebuild activity
- Data-reduction behavior
- Other concurrent system services

When replication or other material background activity is active, the displayed core utilization should be treated as a floor.

---

## 5. Card 2: Capacity

### 5.1 Question answered

> Does the selected model have enough effective capacity for the current data, planned growth, operating headroom, and applicable PBR change-volume reserve?

### 5.2 Logical data

```text
Total logical data = Data today + Growth
```

Growth is entered as an absolute TB value. The application does not calculate capacity growth from an annual percentage.

### 5.3 Informational physical-flash estimate

```text
Estimated physical flash consumed = Total logical data / Entered DRR
```

This value is informational. It illustrates the relationship between logical data and the entered data-reduction ratio, but it is not the basis of the effective-capacity fit verdict.

### 5.4 PBR change-volume reserve

When entered RTT places the workload in the asynchronous PBR band, the application adds a change-volume reserve:

```text
Change-volume reserve = Total logical data x Reserve %
```

The default reserve is 10%. The calculated reserve applies to the system being evaluated. The same reserve must also be considered on the recovery system, which is not modeled independently by this application.

The reserve is not added when:

- RTT is blank
- RTT falls in the synchronous PBHA band
- RTT exceeds the configured asynchronous maximum

### 5.5 Effective capacity required with headroom

```text
Effective TB required =
  (Total logical data + Change-volume reserve) / Target utilization

Effective PBe required = Effective TB required / 1000
```

### 5.6 Capacity utilization

```text
Capacity utilization = Effective PBe required / Model maximum PBe x 100
```

### 5.7 Capacity verdict thresholds

| Utilization | Verdict |
|---:|---|
| 0% through 70% | FITS |
| Greater than 70% through 90% | TIGHT |
| Greater than 90% | TOO LARGE |

### 5.8 Capacity interpretation boundary

The model maximum is an effective-capacity headline that assumes a reference reduction behavior. The entered DRR and the model's listed maximum effective capacity are not dynamically normalized to one common physical configuration. Therefore:

- The fit result is a directional effective-capacity comparison.
- The physical-flash estimate is useful context, not a configured usable-capacity calculation.
- A final design still requires the actual drive count, RAID layout, feature configuration, workload reduction behavior, and IBM Configurator output.

---

## 6. Card 3: Policy-Based Replication and HA

### 6.1 Question answered

> Which policy-based replication mode is indicated by RTT, how much steady-state bandwidth does the write workload require, and can the entered link carry it?

### 6.2 RTT mode selection

| RTT condition | Mode shown by application |
|---|---|
| RTT less than or equal to synchronous threshold | PBHA synchronous |
| RTT greater than synchronous threshold and less than or equal to asynchronous maximum | PBR asynchronous |
| RTT greater than asynchronous maximum | Beyond configured supported limit |

The default thresholds are 1 ms for synchronous classification and 80 ms for maximum asynchronous classification. Both are user-overridable planning parameters.

### 6.3 Replication change rate

Ongoing replication is sized from writes rather than total stored capacity.

```text
Write MB/s = Write IOPS x Write KiB x 1024 / 1,000,000
```

### 6.4 Estimated on-wire rate

```text
On-wire MB/s = Write MB/s / Stream-compression ratio
On-wire Gbps = On-wire MB/s / 125
```

The default 2:1 stream-compression value is an assumption, not a guarantee. If the transport does not provide the assumed compression behavior, the ratio should be set to 1:1.

### 6.5 Usable link rate

```text
Usable link MB/s = Link Gbps x 125 x 0.80
Link utilization = On-wire MB/s / Usable link MB/s x 100
```

The 80% factor is a planning allowance. It is not a measured protocol-efficiency guarantee.

### 6.6 Journaling and cycling indication

For asynchronous operation:

```text
If On-wire MB/s <= Usable link MB/s:
    Indicate journaling
Else:
    Indicate cycling
```

This is a simplified steady-state classification. It shows whether the calculated link can carry the calculated change rate. It does not model burst duration, backlog clearance, contention, packet behavior, recovery operations, or workload variability.

### 6.7 Replication verdict thresholds

When link speed is available, the card applies the general utilization bands:

| Link utilization | Verdict |
|---:|---|
| 0% through 70% | FEASIBLE |
| Greater than 70% through 90% | MARGINAL |
| Greater than 90% | OVER LINK |

If no link speed is entered, the application can only produce a partial bandwidth result.

### 6.8 Initial synchronization

The card sizes steady-state change traffic. Initial synchronization must transfer the required dataset separately and is not modeled. A design may fit steady-state replication while still requiring a separate seeding plan or temporary bandwidth for initial synchronization.

### 6.9 Transport scope

The application describes IP and Fibre Channel partnership possibilities, but its entered WAN-speed arithmetic is a generic Gbps-versus-MB/s comparison. Transport-specific validation remains required, including compression behavior, distance design, network sharing, FCIP or Ethernet implementation, and supported configuration details.

---

## 7. Card 4: System Memory

### 7.1 Question answered

> How does the application's simplified memory allocation compare with the selected node-memory default, and what is the estimated synchronous write-buffer impact?

### 7.2 Memory allocation heuristic

```text
OS and metadata reserve = Node memory x 20%
Usable memory = Node memory - OS and metadata reserve
Write cache = Usable memory x 15%
Read cache = Usable memory x 85%
```

These percentages are application heuristics. They are not a detailed internal Storage Virtualize memory model.

### 7.3 Synchronous buffer estimate

The current v6 implementation calculates:

```text
Buffer MB = Write MB/s x (RTT ms / 1000) x 2
```

However, the entered value is already labeled as round-trip time. Multiplying RTT by 2 again likely overstates the buffer by approximately two times. The technical intent should be resolved in code by either:

- Removing the final multiplication by 2 when the input remains RTT, or
- Relabeling the input as one-way latency if the multiplication is retained.

Until corrected, the displayed buffer is conservative but internally inconsistent with the RTT label.

### 7.4 Memory utilization

```text
Memory consumed = Write cache + Synchronous buffer
Memory utilization = Memory consumed / Usable memory x 100
```

### 7.5 Memory verdict thresholds

| Utilization | Verdict |
|---:|---|
| 0% through 70% | OK |
| Greater than 70% through 90% | REVIEW |
| Greater than 90% | TIGHT |

### 7.6 Memory interpretation boundary

Because write cache is assigned as a fixed 15% share, memory utilization normally starts at 15% before any synchronous buffer is added. The card is therefore best viewed as an explanatory heuristic and an exception detector, not as a full memory-sizing engine.

---

## 8. Performance Scaling Charts

### 8.1 Purpose

The charts convert a pass/fail sizing into a runway conversation. They help explain:

- Where the current workload sits
- Where projected growth places it
- How the selected model compares with alternatives
- Where latency begins to rise sharply
- When measured current-state performance approaches the proposed design target

### 8.2 IOPS chart

The IOPS chart uses each model's mixed-workload saturation as the curve ceiling. The selected model defines the background operating zones.

| Zone | Selected-model range |
|---|---:|
| Comfortable | 0% through comfort target, default 60% |
| Marginal | Comfort target through 87.5% |
| Latency-impacting | 87.5% through 100% |
| Grid territory | Beyond modeled single-system saturation |

The background zones are explanatory, not formal IBM support boundaries.

### 8.3 Throughput chart

The throughput chart uses each model's bandwidth ceiling as the curve denominator and applies the same queueing-shaped visualization. A bandwidth ceiling and an IOPS saturation point are different constraints; the application displays them as separate charts so the user can see which one is more relevant.

### 8.4 Current and projected workload

```text
Projected IOPS = Current IOPS x (1 + Annual growth rate) ^ Years
```

Projected throughput assumes the same read/write mix and transfer sizes as the present workload.

### 8.5 Model visibility and chart controls

The application supports:

- Per-model visibility toggles
- Current-workload marker
- Projected-workload marker
- Measured StorM marker
- Editable X and Y ranges
- Zoom and pan
- Individual PNG export

These controls change presentation only. They do not alter the underlying fit calculations.

---

## 9. Storage Insights CSV Import

### 9.1 Purpose

The StorM import function changes the discussion from estimated workload to observed workload. Multiple Storage Insights exports can be imported and treated as one combined estate.

### 9.2 Fields consumed

The v6 parser uses fixed CSV column positions:

| Index | Field |
|---:|---|
| 1 | Timestamp |
| 7 | Read I/O rate |
| 8 | Write I/O rate |
| 9 | Total I/O rate |
| 10 | Read-cache hits |
| 13 | Read data rate |
| 14 | Write data rate |
| 15 | Total data rate |

This positional design assumes the imported export matches the expected Storage Insights CSV layout.

### 9.3 Timestamp alignment

Each timestamp is moved forward to the next five-minute boundary unless it is already on a boundary. The rounded timestamp becomes the aggregation key across imported systems.

This is a ceiling operation, not nearest-interval rounding.

### 9.4 Multi-system aggregation

For each aligned timestamp, the application:

- Sums read IOPS
- Sums write IOPS
- Sums total IOPS
- Sums read, write, and total data rates
- Computes read-I/O-weighted cache-hit percentage
- Derives average read and write transfer sizes
- Derives read percentage

```text
Weighted read-cache hit =
  Sum(Read cache % x Read IOPS) / Sum(Read IOPS)

Average read transfer KiB/op =
  Read MiB/s x 1024 / Read IOPS

Average write transfer KiB/op =
  Write MiB/s x 1024 / Write IOPS
```

The application rounds several aggregate values upward to whole numbers.

### 9.5 Peak selection

Four independent peak timestamps are identified:

1. Peak total I/O rate
2. Peak total data rate
3. Peak write I/O rate
4. Peak write data rate

These peaks can occur at different timestamps. The table preserves the workload characteristics associated with each selected peak.

### 9.6 Chart overlay

The application places measured values on the modeled curves using orange markers and creates a headroom chart comparing:

- Measured peak
- Selected model comfortable ceiling
- Selected model saturation

The measured marker does not transform the modeled curve into measured latency. It places a measured load value onto an illustrative latency model.

### 9.7 Headroom calculation

```text
Headroom % = (Comfortable ceiling - Measured peak) / Comfortable ceiling x 100
```

In the bar chart, negative headroom is visually clamped to 0%. The comparison should therefore be read together with the bar heights and not treated as a complete overload magnitude.

---

## 10. Export Behavior

### 10.1 PNG export

The application serializes each SVG chart, renders it to a white canvas at two times the SVG dimensions, and downloads a PNG. The chart visualization is suitable for reuse in proposals and presentations, but it retains the same directional limitations as the on-screen chart.

### 10.2 Proposal PDF

The PDF function opens the browser print workflow and adds a proposal header containing the selected model and generation date. The resulting PDF depends on the browser's print implementation and user-selected print settings.

### 10.3 Excel export status in the supplied v6 application

The supplied v6 HTML does not currently implement Excel generation. The button handler writes the following browser-console warning:

```text
Excel export: StorM integration pending.
```

Therefore, the README or user interface should not claim that v6 currently produces a multi-sheet Excel workbook unless the missing export code is restored or completed.

---

## 11. Known Technical Risks and Recommended Corrections

The following items were identified directly from the supplied v6 implementation and should be addressed before the application is represented as fully complete.

### 11.1 Correct the synchronous buffer formula

**Current behavior:** RTT is multiplied by 2 even though the input is already RTT.  
**Recommendation:** Remove the extra factor of 2 or change the input and documentation to one-way latency.

### 11.2 Align the legend with performance thresholds

**Current behavior:** The main legend describes yellow as 70% to 90%, while Card 1 uses 60% to 80%.  
**Recommendation:** State that performance uses 60/80 and the other cards use 70/90, or give Card 1 its own legend.

### 11.3 Update the bottom throughput tip

**Current behavior:** The bottom tip shows a single-block-size decimal shortcut:

```text
(IOPS / 1000) x block KB = MB/s
```

The actual v6 engine uses separate read and write KiB values with binary conversion.  
**Recommendation:** Replace the tip with the actual transfer-size-aware formula.

### 11.4 Restore or remove Excel export claims

**Current behavior:** The Excel button is present, but its code only reports that integration is pending.  
**Recommendation:** Implement the workbook export or label the function unavailable in v6.

### 11.5 Make advanced overrides flow into all charts

**Current behavior:** The result cards honor several entered overrides, but chart model definitions and the headroom chart use static model constants. Changing mixed saturation or bandwidth in Advanced can therefore make the cards and charts disagree.  
**Recommendation:** Build chart limits from the active resolved model values, including user overrides.

### 11.6 Use one throughput unit basis consistently

**Current behavior:** Host workload throughput is displayed as decimal GB/s, StorM data is reported in MiB/s, and some chart conversions divide or multiply by 1024 while comparing with decimal GB/s ceilings.  
**Recommendation:** Choose a canonical internal unit, preferably bytes per second, and convert only for display. Clearly label GB/s versus GiB/s and MB/s versus MiB/s.

### 11.7 Harden CSV parsing

**Current behavior:** Each line is split on commas with `split(',')`. Quoted fields containing commas would be parsed incorrectly.  
**Recommendation:** Use a CSV parser that supports quoted fields, embedded commas, different line endings, and optional headers.

### 11.8 Validate the CSV schema by header name

**Current behavior:** The parser relies on fixed column indexes.  
**Recommendation:** Resolve columns from normalized header names and fail clearly when required fields are absent.

### 11.9 Make missing-sample behavior explicit

**Current behavior:** Systems are summed only when a row exists for a rounded timestamp. A missing row contributes nothing, which can understate an aggregate peak if exports have gaps or nonaligned sampling.  
**Recommendation:** Detect missing samples and report data completeness. Do not silently treat absent rows as measured zero.

### 11.10 Revisit timestamp handling

**Current behavior:** Browser date parsing is locale dependent, and timestamps are always rounded upward to the next five-minute boundary.  
**Recommendation:** Parse the documented Storage Insights timestamp format explicitly, preserve timezone information, and choose a documented alignment policy such as nearest interval or exact interval matching.

### 11.11 Avoid rounding all measured values upward

**Current behavior:** Several measured aggregates are rounded with `Math.ceil`.  
**Recommendation:** Retain full precision internally and round normally for presentation. Always rounding upward can bias summaries, ratios, and derived transfer sizes.

### 11.12 Clarify Grid language

**Current behavior:** The chart labels all load beyond a single modeled saturation as “Grid territory.”  
**Recommendation:** Treat this as a conversation cue, not a conclusion that Grid is the required or automatically scalable answer. Workload distribution, host access, topology, and supported design still require validation.

### 11.13 Separate fit from recommendation

**Current behavior:** A green result establishes that the workload is below a selected threshold, but it does not establish that the selected model is the best commercial or architectural choice.  
**Recommendation:** Use language such as “fits the modeled constraints” rather than “recommended” unless capacity configuration, ports, resilience, growth, service requirements, and Storage Modeller have also been reviewed.

---

## 12. Validation Strategy

A disciplined validation process should include four levels.

### Level 1: Arithmetic tests

Use fixed sample inputs and independently verify:

- Read/write split
- Read and write throughput
- Capacity headroom
- Change-volume reserve
- Replication bandwidth
- Link utilization
- Growth projection
- Memory arithmetic

### Level 2: Cross-component consistency tests

Confirm that:

- Cards, charts, PDF, PNG, and any future Excel export use the same resolved inputs.
- Advanced overrides change every dependent output.
- Decimal and binary unit conversions remain consistent.
- Clearing or changing model selection does not leave stale chart state.

### Level 3: Reference-tool calibration

For matched workloads, compare calculator outputs with:

- IBM Storage Modeller
- IBM Storage Insights telemetry
- IBM Configurator
- Current Storage Virtualize configuration limits

Calibration must use the same read/write ratio, transfer sizes, cache behavior, and system model. A comparison using different workload assumptions is not a valid model-to-model calibration.

### Level 4: Field review

Have an IBM storage SME review:

- Whether the assumptions are appropriate for the opportunity
- Whether imported telemetry represents a true design peak
- Whether concurrent systems should be summed
- Whether the recovery design and transport assumptions are valid
- Whether business growth and failure scenarios are represented

---

## 13. Recommended Client-Facing Use

The strongest client use is not to present the calculator as an answer engine. It should be used to support a transparent engineering conversation:

1. Import or enter the best available workload evidence.
2. Show which values are measured and which are assumed.
3. Explain the binding constraint.
4. Show current load, projected growth, comfort target, and saturation separately.
5. Export a clean chart for the proposal.
6. State what still requires authoritative validation.

A defensible summary should sound like this:

> Based on the entered workload and the measured Storage Insights peaks, the proposed system fits the modeled performance, bandwidth, capacity, and replication constraints with the displayed headroom. The calculator exposes the assumptions and arithmetic used for that first-pass conclusion. Final sizing and configuration remain subject to IBM Storage Modeller, Storage Insights validation, IBM Configurator, and current support limits.

This language is useful because it is strong enough to explain the design while remaining honest about what the application has and has not proven.

---

## 14. Formula Reference

| Result | Formula |
|---|---|
| Read IOPS | Total IOPS x Read % |
| Write IOPS | Total IOPS x Write % |
| Read MB/s | Read IOPS x Read KiB x 1024 / 1,000,000 |
| Write MB/s | Write IOPS x Write KiB x 1024 / 1,000,000 |
| Total MB/s | Read MB/s + Write MB/s |
| Core utilization | Total IOPS / Mixed saturation x 100 |
| Comfortable IOPS | Mixed saturation x Comfort % |
| Bandwidth utilization | Total GB/s / Bandwidth ceiling x 100 |
| Estimated latency | Base service time / (1 - utilization) |
| Logical data | Data today + Growth |
| Physical flash estimate | Logical data / DRR |
| Change-volume reserve | Logical data x Reserve % |
| Effective TB needed | (Logical data + Reserve) / Target utilization |
| Capacity utilization | Required PBe / Maximum PBe x 100 |
| On-wire MB/s | Write MB/s / Stream compression |
| On-wire Gbps | On-wire MB/s / 125 |
| Usable link MB/s | Link Gbps x 125 x 0.80 |
| Link utilization | On-wire MB/s / Usable link MB/s x 100 |
| Current v6 sync buffer | Write MB/s x RTT seconds x 2 |
| Usable memory | Node memory x 0.80 |
| Write cache | Usable memory x 0.15 |
| Read cache | Usable memory x 0.85 |
| Memory utilization | (Write cache + Sync buffer) / Usable memory x 100 |
| Projected IOPS | Current IOPS x (1 + Growth rate) ^ Years |
| StorM weighted cache | Sum(Cache % x Read IOPS) / Sum(Read IOPS) |
| Read transfer KiB/op | Read MiB/s x 1024 / Read IOPS |
| Write transfer KiB/op | Write MiB/s x 1024 / Write IOPS |
| Headroom | (Comfortable ceiling - Measured peak) / Comfortable ceiling x 100 |

---

## 15. Document Control and Trust Statement

This document describes the supplied v6 HTML implementation, including its current logic and known gaps. It should be revised whenever the application's formulas, defaults, supported imports, model constants, or export behavior change.

The calculator is best treated as:

- **More transparent than a rule-of-thumb spreadsheet**
- **More approachable than an authoritative sizing tool**
- **More useful when paired with measured Storage Insights data**
- **Still subordinate to IBM's formal sizing, configuration, and support-validation processes**

