# IBM FlashSystem SV 9.1.2 Sizing Calculator

A self-contained, single-file pre-sales sizing tool for IBM FlashSystem (Storage Virtualize 9.1.2). Open `fs-sv912-calculator-v6.html` in any modern browser — no server, no install, no dependencies.

---

## Files

| File | Purpose |
|---|---|
| `fs-sv912-calculator-v6.html` | Main calculator — current version with all features |
| `StorM_aggregate.py` | Companion CLI script — aggregates Storage Insights CSVs into a peak-performance Excel report |
| `fs-sv912-calculator-v5.html` | Previous version (reference) |

---

## Calculator — Quick Start

1. Open **`fs-sv912-calculator-v6.html`** in Chrome, Safari, or Edge
2. Select a FlashSystem model (9600 / 7600 / 5600)
3. Enter what you know — all other fields default to IBM-recommended values (shown with a purple **assumed** tag)
4. Results update instantly across all four cards
5. Scroll down to the **Performance Scaling** section to see latency curves and import real measured data

---

## Four Sizing Cards

| Card | What it answers | Minimum input |
|---|---|---|
| **PERF** — Local Performance | Can the system handle this workload? | Total IOPS |
| **CAP** — Capacity | Will data fit with room to grow? | Data today (TB) |
| **REPL** — Policy-Based Replication & HA | Replication mode, bandwidth, buffer | RTT to DR site |
| **MEM** — System Memory | Cache and PBHA journal impact | IOPS and/or RTT |

---

## Performance Scaling & Modelling Section

### Latency Curves
Three FlashSystem models overlaid on a queueing-theory latency curve (calibrated to IBM Storage Modeller for the FS9600). Shows the "hockey stick" shape — where load goes from comfortable to latency-impacting.

**Visibility toggles** let you show/hide individual model lines, the NOW dot, the projected growth dot, and StorM measured peaks.

### Growth Projection
Enter an **IOPS growth rate (% / year)** and **projection horizon (years)** to place a projected workload dot on the curves — see where you'll be in N years.

### Editable Axis Ranges
Below each latency chart are **X max** and **Y max** override fields:

- **X max (IOPS chart)** — set a custom IOPS ceiling, e.g. `500000` to zoom into the client's operating window
- **Y max (IOPS chart)** — set a custom latency ceiling in ms, e.g. `2` to focus on the flat/green zone
- **X max (Throughput chart)** — set a custom GB/s ceiling
- **Y max (Throughput chart)** — set a custom latency ceiling in ms
- **Auto** button — clears overrides and restores auto-calculated ranges

### Zoom & Pan
Every chart has interactive zoom and pan controls:

| Action | Effect |
|---|---|
| Scroll wheel | Zoom in / out centred on cursor |
| Click + drag | Pan the chart |
| **+** / **−** buttons | Step zoom in / out |
| **⟳** button | Reset to default view |
| **↓** button | Export chart as PNG |

---

## StorM CSV Import — Real Measured Performance

The **StorM import panel** (orange section at the top of Performance Scaling) brings your actual Storage Insights telemetry directly into the calculator.

### How to use

1. Export per-system performance CSVs from **IBM Storage Insights** (standard export format)
2. Drop one or more CSV files onto the orange drop zone, or click to browse
3. The tool aggregates all systems, aligns 5-minute timestamps, and identifies peak moments
4. Results appear immediately in the chart

### What it produces

**Peak Summary Table** — four worst-case moments across all imported systems:

| Peak metric | What it shows |
|---|---|
| Peak Total I/O Rate | Busiest IOPS moment |
| Peak Total Data Rate | Highest throughput moment (MiB/s) |
| Peak Write I/O Rate | Write-heaviest IOPS moment |
| Peak Write Data Rate | Highest write throughput moment |

Each row shows: timestamp · Read % · Total IOPS · Avg Read transfer size · Avg Write transfer size · Weighted avg cache hit ratio

**Orange diamond markers** on the IOPS and Throughput latency curves — plots each measured peak directly on every model's curve so clients can see exactly where their current estate sits relative to the hockey-stick knee.

**Current Measured vs. New System Headroom chart** — a collapsible two-panel bar chart:

- **Left panel** — I/O Rate in IOPS: measured peak vs new system comfortable ceiling vs saturation
- **Right panel** — Data Rate in MiB/s: same comparison for throughput
- Each bar group shows a colour-coded **headroom %** (green > 30%, amber > 10%, red ≤ 10%)
- Click the title bar to collapse / expand the chart

### Multiple CSV files
Drop CSVs from multiple systems simultaneously. The tool sums IOPS and data rates across all systems (as if they were one large system) and computes a read-I/O-weighted average cache hit ratio — the same aggregation logic as `StorM_aggregate.py`.

If any imported file has missing 5-minute windows (export gaps), a **⚠ partial coverage warning** appears above the peaks table when more than 1% of timestamps are affected — peaks for those windows may be understated.

### CSV format
The parser handles standard IBM Storage Insights exports including **quoted fields containing commas** (e.g. system names like `"SYS,01"`) and files with a UTF-8 BOM. Relevant columns:

| Column | Metric |
|---|---|
| 1 | Timestamp (supports `M/D/YY HH:mm`, `M/D/YYYY HH:mm`, and ISO-8601 formats; rounded to nearest 5-minute boundary) |
| 7 | Read I/O Rate (ops/s) |
| 8 | Write I/O Rate (ops/s) |
| 9 | Total I/O Rate (ops/s) |
| 10 | Read Cache Hits (%) |
| 13 | Read Data Rate (MiB/s) |
| 14 | Write Data Rate (MiB/s) |
| 15 | Total Data Rate (MiB/s) |

---

## Export Options

| Button | Output |
|---|---|
| **Export to CSV** | Downloads a `.csv` file — sections for Inputs, Performance, Capacity, Replication, Memory, and StorM Peaks (if loaded). Opens in any spreadsheet app (Excel, Numbers, Google Sheets). |
| **Export Proposal PDF** | Browser print dialog — pre-sales proposal layout with key verdicts in the header; charts and inputs print cleanly |
| **Export Charts PNG** | Downloads each visible chart (IOPS Latency, Throughput Latency, Headroom if loaded) as individual high-resolution PNG files — ready to paste into PowerPoint or Word proposals |

---

## Advanced / Override Inputs

Expand **"[Model] array ceilings (override)"** (label updates when you switch the model) to adjust:

| Field | Default | Notes |
|---|---|---|
| 4K read-hit peak | model-specific | Datasheet theoretical ceiling — not reachable for mixed I/O |
| Mixed saturation | model-specific | Calibrated to IBM Storage Modeller for FS9600; scaled estimates for 7600/5600 |
| Comfort % of saturation | 60% | IBM Storage Modeller amber threshold |
| BW ceiling | model-specific | GB/s |
| Max effective capacity | model-specific | PBe |
| PBHA sync RTT limit | 1 ms | |
| PBR async max RTT | 80 ms | |
| Change volume reserve | 10% | IBM guideline; applied to both production and recovery systems |
| Node memory | model-specific | GB |

Expand **"Workload sensitivity"** to add an optional cache-efficiency estimate (does not affect the main derate — flagged separately).

---

## Model Defaults

| Model | Sat. IOPS | BW ceiling | Max eff. cap | Node memory |
|---|---|---|---|---|
| FlashSystem 9600 (5078-A40) | 1,500,000 | 86 GB/s | 11.8 PBe | 1,536 GB |
| FlashSystem 7600 (5075-A30) | 850,000 | 55 GB/s | 7.2 PBe | 768 GB |
| FlashSystem 5600 (5127-A20) | 450,000 | 30 GB/s | 2.5 PBe | 512 GB |

Saturation IOPS = mixed-workload saturation point (70/30 R/W, 8/16 KiB, ~55% cache hit). FS9600 calibrated to IBM Storage Modeller; FS7600/FS5600 are scaled estimates — override per model.

---

## StorM Aggregate CLI (StorM_aggregate.py)

A standalone Python script for batch aggregation of Storage Insights CSVs into a formatted Excel report. Useful when you want a shareable `.xlsx` rather than the in-browser view.

### Requirements
```
pip install openpyxl
```

### Usage
```
python StorM_aggregate.py
```
Enter CSV file paths when prompted (comma-separated). The script:
1. Parses each CSV and snaps timestamps to 5-minute boundaries
2. Sums IOPS and data rates across all systems
3. Computes read-I/O-weighted average cache hit ratio
4. Identifies peak moments for Total IOPS, Total Data Rate, Write IOPS, Write Data Rate
5. Writes `StorM_aggregate_report.xlsx` with a styled summary table
6. Prints a plain-text summary to the console

### Output columns
Peak Description · Date/Time · Read I/O % · Total I/O Rate (ops/s) · Avg Read Transfer Size (KiB/op) · Avg Write Transfer Size (KiB/op) · Wtd Avg Read Cache Hit Ratio (%)

---

## Disclaimer

**Teaching and estimation tool — not a substitute for IBM Storage Modeller.**
All values are directional. Validate any committed figure in IBM Storage Modeller (authoritative report) or IBM Storage Insights (measured telemetry) before quoting a BOM. The performance curve is calibrated to Storage Modeller for the FS9600 mixed-workload profile; FS7600/FS5600 saturation points are scaled estimates.
