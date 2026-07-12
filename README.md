# IBM-FS-SV9.1.x-Calculator
This calculator gives a quick, defensible first-pass answer to all four for the current-generation **IBM FlashSystem** arrays — **FS9600**, **FS7600**, **FS5600**, and **SV3Nodes** — running **Storage Virtualize 9.1.2**.

# IBM FlashSystem — Storage Virtualize 9.1.2 Sizing Calculator
## Technical README & Plain-Language Guide

*What every number means, what the results are telling you, and how to use them to explain a storage design — written so that someone new to storage sizing can follow it.*

---

## 1. What this tool is for

When you buy or design an enterprise storage array, you have to answer four practical questions **before** you sign off on the hardware:

1. **Can the array keep up with the work?** (speed)
2. **Will the data fit, with room to grow?** (capacity)
3. **If the site burns down, how do we get a copy somewhere else, and how fresh will that copy be?** (replication / disaster recovery)
4. **Does the array have enough memory to do all of the above well?** (cache & buffers)

This calculator gives a quick, defensible first-pass answer to all four for the current-generation **IBM FlashSystem** arrays — **FS9600**, **FS7600**, and **FS5600** — running **Storage Virtualize 9.1.2**. You pick the model at the top of the panel and the tool measures your workload against that array's ceilings. Below the four result cards, a **Performance Scaling Charts** section overlays all three models on the same graph so you can instantly see which model gives the right amount of headroom and where growth eventually forces an upgrade or a FlashSystem Grid. This tool is a *pre-sales / planning* tool: it produces estimates good enough to shape a design and explain it, not the final bill of materials. The final configuration is always confirmed in the official IBM Configurator.

### The "enter what you know" philosophy

You are not expected to know every input. You type in the few things you do know (for example, "we do about 150,000 IOPS and have 200 TB of data"), and the tool fills in everything else with sensible **defaults**. Any value the tool assumed for you is tagged in **purple** with the word *assumed*. That tag is important: it tells the reader of your sizing exactly which numbers came from the customer and which were industry-standard guesses. When you can replace an assumed value with a real measured one, your sizing becomes more trustworthy.

### The traffic-light / MALI FLAG  system

Every result card shows a colored verdict so you can read it at a glance:

| Color | Meaning | Utilisation band |
|-------|---------|------------------|
| 🟢 **Green** | Comfortably within spec. The array has headroom. | 0–70% |
| 🟡 **Yellow** | Marginal. It works, but there's little room for spikes or growth. Review it. | 70–90% |
| 🔴 **Red** | Over the limit. The design needs a bigger or additional array. | 90%+ |

"Utilisation" simply means *how much of the array's maximum you are planning to use*. Lower is safer; you generally want to land in green so that a busy Monday morning, or next year's data growth, doesn't push you over the edge.

### Choosing your model

At the top of the input panel is a **FlashSystem model** selector covering the three current-generation arrays. Picking a model loads that array's performance, bandwidth, capacity, and memory ceilings, so every result is measured against the right hardware. The three are, from largest to smallest: **FS9600** (the flagship), **FS7600** (mid-to-upper enterprise), and **FS5600** (entry-to-midrange). The exact ceilings are listed in Section 7.3. The replication behaviour (sync/async RTT limits) and the mixed-workload derate are Storage Virtualize software characteristics and are the same on all three. If you're sizing a multi-array **FlashSystem grid**, override the capacity ceiling with the grid figure shown for that model.

---

## 2. A 60-second glossary (read this first if storage is new to you)

**IOPS** — *Input/Output Operations Per Second.* The number of individual read or write requests the array handles each second. Think of it as "how many small errands per second." Databases and virtual machines generate lots of small errands, so they care about IOPS.

**Block size** — How big each errand is, in kilobytes (KB). A bigger block moves more data per errand. Typical mixed/database workloads use about **8 KB**.

**Throughput / Bandwidth** — How much *total data* moves per second, measured in **MB/s** (megabytes/sec) or **GB/s** (gigabytes/sec). This is just `IOPS × block size`. Two systems can do the same IOPS but very different bandwidth if their block sizes differ. Backups and analytics care about bandwidth more than IOPS.

**Read/Write split** — What fraction of the errands are reads versus writes. A common default is **70% read / 30% write**. This matters because writes are heavier work for flash, and because **only writes need to be replicated** to a disaster-recovery site (reading data doesn't change it).

**Latency / RTT** — *Round-Trip Time.* How long, in milliseconds (ms), a signal takes to travel to the other site and back. Distance adds latency because light and electricity are not instant. RTT is the single most important number deciding *which kind* of replication you can use. On the performance charts, **latency (ms/op)** refers to per-I/O response time — how long the application waits for the array to complete a single read or write — which rises as the array approaches its ceiling.

**Data reduction (DRR)** — IBM FlashSystem squeezes data with **compression** and **deduplication** before storing it. A **3:1** ratio means 3 TB of your data fits in 1 TB of physical flash. Higher ratios mean you buy less hardware for the same data.

**Capacity flavors** — this trips up newcomers, so:
- **Raw / usable capacity** = the actual physical flash you bought (after RAID overhead). Measured in **PBr** (petabytes raw) in this tool.
- **Effective capacity** = how much *logical* data you can actually store *after* data reduction does its job. Measured in **PBe** (petabytes effective). This is the bigger, "marketing" number, and it's what you usually size against because it's what the customer's data actually consumes.

**Headroom / target utilisation** — You never fill a storage array to 100%; performance degrades and you have no buffer. A **target of 80%** means "plan to use at most 80% of the array," leaving 20% breathing room.

**RPO** — *Recovery Point Objective.* If disaster strikes, how much recent data can you afford to lose? **RPO = 0** means "no data loss at all." **RPO = seconds** means "we might lose the last few seconds of writes."

**RTO** — *Recovery Time Objective.* How fast must you be back up and running after a failure?

**Cache** — Fast memory (DRAM) inside the array that holds hot data so the system doesn't always have to go to flash. Split into **read cache** and **write cache**.

**Queueing / hockey-stick curve** — As any system (storage, network, checkout line) approaches its maximum capacity, response time rises slowly at first and then very steeply. This is a fundamental property of queueing systems. The Performance Scaling Charts visualise this effect, showing why an array running at 90% utilisation has dramatically worse latency than one at 60%, even though the difference in load looks small in percentage terms.

**FlashSystem Grid** — IBM's scale-out architecture allowing up to 32 FlashSystem nodes to be clustered together, multiplying effective capacity (and, to a degree, performance) while appearing as a single system to hosts. The charts show where a single array saturates and label that region "GRID →" to indicate where a Grid configuration would take over.

---

## 3. The replication concepts, explained simply

This is the part most people find confusing, so here it is in everyday terms. IBM gives you two related ways to keep a second copy of your data at another site.

**Policy-Based HA (PBHA) — the "two arms always in sync" option.**
Both sites are live at the same time and every write is mirrored to the other site *before the application is told "done."* Nothing is ever lost (**RPO = 0**) and failover is automatic and instant (**RTO = 0**). The catch: because the application waits for the far site to confirm every single write, the two sites must be **very close together with very low latency** — ideally well under a millisecond of round-trip time, i.e. metro-area distance. This is **synchronous** replication.

**Policy-Based Replication (PBR) — the "shipping copies behind the scenes" option.**
Writes are confirmed locally and a copy is sent to the far site shortly afterward. The far site is a few seconds (or more) behind, so **RPO is greater than zero** but you can put the two sites much farther apart. This is **asynchronous** replication, and IBM supports it up to about **80 ms** of round-trip time.

Within async PBR there are two automatic sub-modes:

- **Journaling mode** — the link is fast enough to keep up with the rate of change, so changes stream across continuously and the copy stays only seconds behind. Good. (RPO ≈ seconds.)
- **Cycling mode** — the link *can't* keep up, so the system falls back to sending periodic snapshots instead. The copy is further behind (larger RPO). This is a sign you need a faster link or a lower change rate.

You don't choose journaling vs cycling manually — the system picks based on whether your link can carry your write rate. The calculator predicts which one you'll land in.

**One crucial sizing insight:** ongoing replication bandwidth is driven by your **write rate**, *not* by how much total data you have. A 500 TB system that only changes 50 MB/s of data needs only enough link to carry 50 MB/s. (The exception is the very first **initial sync**, which must copy the *entire* dataset once — plan extra time or bandwidth for that one-time event; this tool sizes the steady-state, not the initial seed.)

---

## 4. Walk-through of each result card

Below, each card is explained as: **the question it answers → the inputs it uses → what each output number means → how to read the verdict.** The worked numbers use the built-in **Load Sample** scenario on the default **FS9600** model (150,000 IOPS, 70/30 read/write, 8 KB blocks, 200 TB data + 50 TB growth, 3.5:1 reduction, 80% target, 12 ms RTT, 10 Gbps link, 2:1 stream compression). Selecting FS7600 or FS5600 keeps the same workload but measures it against that model's smaller ceilings, so the utilisation percentages rise accordingly.

---

### Card 1 — Local Performance: *"Can the system handle this workload?"*

**Needs:** Total IOPS. **Assumes if blank:** read/write split, block size, and the array's speed ceilings.

**IOPS breakdown.** The tool splits your total IOPS into reads and writes using the read% you gave (or the 70% default).
- Read IOPS = `Total IOPS × read%` → 150,000 × 70% = **105,000**
- Write IOPS = `Total IOPS × write%` → 150,000 × 30% = **45,000**

*Why it matters:* the write portion is what later drives both replication bandwidth and write-cache needs.

**Throughput.** Converts errands-per-second into data-per-second:
> `(IOPS ÷ 1000) × block size in KB = MB/s`

→ (150,000 ÷ 1000) × 8 = **1,200 MB/s = 1.2 GB/s**. (For reference, 1,200 MB/s is about 9.6 Gbps of data movement.)

This card shows total, read, and write throughput separately so you can see whether your workload is "lots of tiny errands" (IOPS-bound) or "fewer big errands" (bandwidth-bound).

**System headroom.** This is the actual fit check. The array has two independent speed limits, and your workload must fit under **both**:

- **IOPS ceiling.** The array's peak IOPS figure is a *best-case* "4K read-hit" lab number. Real mixed workloads can't reach it, so the tool multiplies by a **derate** (default 60%) to get a realistic usable ceiling.
  > Usable IOPS = `peak IOPS × derate%`
  Then **IOPS utilisation** = `your IOPS ÷ usable IOPS`. Lower is better.

- **Bandwidth ceiling.** The most data-per-second the array can push. **Bandwidth utilisation** = `your GB/s ÷ ceiling GB/s`.

The card's verdict reflects **whichever of the two is higher** (the binding constraint). If you're at 5% on IOPS but 95% on bandwidth, you're bandwidth-bound and the verdict is red. This is exactly how a real bottleneck behaves — the tightest limit wins.

**Helper notes you may see:**
- *Write-heavy warning* (over 50% writes): flash wears and slows more under heavy writes ("write amplification"), so consider a more conservative derate.
- *Bandwidth-bottleneck warning*: your block size is large enough that you'll run out of GB/s before you run out of IOPS.

**How this justifies the sizing:** "At 150,000 IOPS and 1.2 GB/s, we're using under 5% of the array's realistic ceilings — green across the board — so this single FS9600 has ample performance headroom for this workload and years of growth."

---

### Card 2 — Capacity: *"Will the data fit, with room to grow?"*

**Needs:** Data today (TB). **Assumes if blank:** data reduction ratio, target utilisation, array max capacity. Growth is optional.

**Logical (effective) data.** Your real-world data, today plus expected growth:
> `Data today + Growth` → 200 + 50 = **250 TB logical**

"Logical" means the size as your applications see it. This logical figure is also your **effective footprint** — the amount of effective capacity it consumes on the array.

**Physical flash consumed (informational).** How much actual flash that data occupies once compression/dedup do their job:
> `Logical ÷ DRR` → 250 ÷ 3.5 = **71.4 TB** of physical flash

This line uses *your* data-reduction ratio and is useful for understanding how much NAND you're burning and your endurance headroom. It does not drive the fit verdict.

**Effective needed with headroom.** Because you never fill to 100%, the tool grosses the logical footprint up to your target:
> `Logical ÷ target%` → 250 ÷ 80% = **312.5 TB = 0.313 PBe**

**Array fit.** The tool compares this effective demand against the array's **effective** capacity ceiling and reports **capacity utilisation** (here 0.313 ÷ 11.8 PBe ≈ **2.6%**). Green means it fits comfortably; red means the dataset is too large for one array and you need a larger model or a FlashSystem grid of several arrays.

> **Why effective-vs-effective?** Effective capacity already accounts for data reduction, so the honest comparison is logical-data against effective-capacity (both "post-reduction" terms). The 11.8 PBe figure assumes a reference reduction ratio — if your data reduces better or worse than that, the realised effective capacity scales up or down accordingly. A single FS9600 enclosure tops out near 11.8 PBe; a grid of up to 32 systems scales to roughly 377 PBe.

**How this justifies the sizing:** "250 TB of data is a small fraction of the array's ~11.8 PBe effective capacity even after growth and headroom (about 2.6%), and it occupies only ~71 TB of physical flash — so capacity is not a constraint and there's substantial room to grow."

---

### Card 3 — Replication & HA: *"Which DR mode, how much link, how fresh a copy?"*

**Needs:** RTT to the DR site. **Uses if available:** IOPS (to derive write rate) and WAN link speed. **Assumes if blank:** stream compression, block size, read/write split.

**Replication mode.** The tool sorts your RTT into one of three bands and shows a colored mode badge:
- **≤ 1 ms → PBHA SYNC**, RPO = 0 (the "always in sync" option; metro distance only).
- **1 ms to 80 ms → PBR ASYNC**, RPO > 0 (the "shipping copies" option), refined into journaling or cycling once it knows your link speed.
- **> 80 ms → not supported** — beyond the documented RTT limit for asynchronous PBR, the card turns red and tells you replication isn't supported at that distance (you'd need externally orchestrated DR).

Both thresholds (the 1 ms sync limit and the 80 ms async ceiling) are overridable in the Advanced section if IBM's guidance changes.

**Replication bandwidth.** Replication only carries **writes**, so:
> Write rate = `Write IOPS × block size` → 45,000 IOPS × 8 KB ≈ **360 MB/s**

Then it subtracts what compression saves on the wire:
> On-wire = `write rate ÷ stream compression` → 360 ÷ 2 = **180 MB/s = 1.4 Gbps**

This "on-wire" number is the actual traffic your network link must carry between sites.

**Link capacity & utilisation.** Network links are quoted in Gbps (gigabits), but data is measured in MB/s (megabytes). The conversion is `1 Gbps = 125 MB/s`. The tool also reserves ~20% of the link for overhead and assumes ~80% is usable:
> Usable link = `link Gbps × 125 × 80%` → 10 × 125 × 0.8 = **1,000 MB/s**
> Link utilisation = `on-wire ÷ usable link` → 180 ÷ 1,000 = **18%**

Low link utilisation (green) means the pipe comfortably keeps up, so async replication will run in the good *journaling* mode and stay just seconds behind. High utilisation (red) means the link is the bottleneck and you'll fall into *cycling* mode (larger RPO) — fix it with a faster link or by reducing change rate.

**Journal buffer (sync only).** In synchronous mode, the array briefly holds writes in memory while waiting for the far site to confirm. The amount of memory needed is the "bandwidth-delay product":
> Buffer ≈ `write rate × RTT` (a small number for low-latency links)

**How this justifies the sizing:** "At 12 ms RTT we replicate asynchronously with PBR. The workload changes ~360 MB/s, which compresses to ~180 MB/s on the wire — only 18% of a 10 Gbps link — so the copy stays seconds-fresh in journaling mode with plenty of link headroom."

---

### Card 4 — System Memory: *"Is there enough DRAM for cache and replication buffers?"*

**Needs:** IOPS (for write-cache context) and/or RTT (for sync buffer). **Assumes if blank:** node memory, read/write split, block size.

**Node memory allocation.** Starts from the array's total memory and carves out:
- **OS + metadata reserve** (~20%) — memory the system keeps for itself.
- **Usable for cache** — what's left for data caching.

**Cache allocation.** The usable cache is split into a smaller **write cache** (~15%) and a larger **read cache** (~85%). Reads benefit from more cache (hot data served instantly); the write cache mainly absorbs bursts before flushing to flash.

**PBHA journal buffer.** If you're in synchronous mode, the in-flight write buffer (from Card 3) also lives in memory and is added on top.

**Memory utilisation.** Adds write cache + journal buffer and expresses it as a percent of usable memory. With the FS9600's large DRAM, this almost always lands comfortably green — the card mainly exists to flag the rare case where a *very* high write rate over a sync link makes the journal buffer large enough to crowd out read cache. If you see that warning, consider whether async (PBR) is acceptable instead.

**How this justifies the sizing:** "Cache and replication buffers consume a small fraction of node memory, so the standard memory configuration is sufficient — no memory upgrade is required for this workload."

---

## 5. Performance Scaling Charts — full guide and legend

The Performance Scaling section appears below the four result cards (collapsed/expanded with the toggle arrow). It contains two side-by-side charts — one for IOPS, one for throughput — that plot all three FlashSystem models on the same axes simultaneously. The goal is to show not just whether today's workload fits, but **where on the latency curve that workload sits, and at what growth point each model starts to become long in the tooth**.

Both charts update live as you type, just like the result cards.

---

### 5.1 The two charts

**Chart 1 — IOPS vs. Estimated Latency**
- X-axis: Total IOPS, from 0 to beyond the FS9600 ceiling (up to ~7.4 million, 18% past the FS9600 ceiling).
- Y-axis: Estimated per-I/O latency in milliseconds (ms).
- Three curves: one for each model, from zero load up to that model's own ceiling (where the curve ends).

**Chart 2 — Throughput vs. Estimated Latency**
- X-axis: Total throughput in GB/s, from 0 to beyond the FS9600 ceiling (~101 GB/s).
- Y-axis: Same estimated latency scale as the IOPS chart.
- Three curves: one for each model, to that model's bandwidth ceiling.

The throughput figure is derived from your IOPS and block size: `(IOPS ÷ 1000) × block KB ÷ 1000 = GB/s`. Both charts update together because they share the same workload inputs.

---

### 5.2 Complete element legend

#### Background zone fills

The chart background is divided into four colored bands. Boundaries are anchored to the **FS9600** (the widest model) so all three curves can be read against the same reference scale. The zone labels appear at the top of each band.

| Zone label | Color | X-axis boundary (IOPS chart) | X-axis boundary (BW chart) | What it means |
|---|---|---|---|---|
| **COMFORTABLE** | Faint green | 0 → derate threshold (default: 0 → 3,780,000 IOPS) | 0 → 51.6 GB/s | Load is below the mixed-workload derate point. Latency is low and flat. This is the target operating region. |
| **MARGINAL** | Faint amber | Derate threshold → 87.5% of FS9600 ceiling (3,780,000 → 5,512,500 IOPS) | 51.6 → 75.25 GB/s | Above the derate point, latency begins rising. The system still works but there is little headroom for bursts or growth. |
| **LATENCY↑** | Faint red | 87.5% → 100% of FS9600 ceiling (5,512,500 → 6,300,000 IOPS) | 75.25 → 86 GB/s | Latency is climbing steeply. The queueing hockey-stick is in full effect. Operating here degrades application response times and risks saturation under any spike. |
| **GRID →** | Faint gray | Beyond FS9600 ceiling (> 6,300,000 IOPS) | > 86 GB/s | A single FS9600 is saturated. A FlashSystem Grid (up to 32 nodes) continues the scaling story from this point. |

> The derate threshold uses whatever value is set in the Advanced panel (default 60%). If you override the derate, the green/amber boundary shifts accordingly, and the zone coloring updates live.

---

#### Model curves

Three performance curves are drawn, one per model. Each curve runs from zero load to **that model's own ceiling** — the curve ends there (it does not continue as a horizontal plateau into saturation). This is intentional: beyond a model's ceiling, behavior is undefined, and the vertical dashed marker (see below) signals where that boundary is.

| Curve color | Model | IOPS ceiling | BW ceiling | Baseline latency at zero load |
|---|---|---|---|---|
| **Cyan** `#00b4d8` | FlashSystem 9600 | 6,300,000 IOPS | 86 GB/s | 0.10 ms |
| **Purple** `#7c3aed` | FlashSystem 7600 | 4,300,000 IOPS | 55 GB/s | 0.12 ms |
| **Amber/gold** `#f59e0b` | FlashSystem 5600 | 2,600,000 IOPS | 30 GB/s | 0.15 ms |

Reading the curves together: the FS5600 curve ends first (shorter runway), FS7600 next, FS9600 last. For any given workload, the gap between where your NOW dot lands on each curve shows you the difference in latency impact across models at that load level — and how much additional runway each larger model provides before the hockey stick steepens.

---

#### Vertical dashed ceiling markers

A dashed vertical line in each model's color runs from the top to the bottom of the chart (and slightly below the X-axis) at that model's ceiling. A small label — **FS9600**, **FS7600**, or **FS5600** — appears below the X-axis at each line. These markers show exactly where each model's curve ends and make it easy to read off "if my workload were X IOPS, which models can still handle it?"

---

#### NOW dot (filled circle)

A filled circle appears on **each model's curve** at the X-position corresponding to your current workload (IOPS or throughput) as soon as you enter an IOPS value in the input panel. The dot's color matches the model's curve. A **"NOW"** label appears above the topmost dot (the model with the most headroom, since it has the lowest latency at the same load).

Reading the NOW dots: they show you simultaneously where today's workload sits on all three models. If the NOW dot on FS5600 is in the red zone but the NOW dot on FS7600 is in the green zone, the workload is too heavy for FS5600 and FS7600 is the minimum viable model. If all three NOW dots are deep in the green zone, any model is viable and the selection can be driven by capacity or other factors.

If your IOPS exceed a model's ceiling, the NOW dot for that model is plotted at that model's ceiling (the rightmost end of its curve), indicating saturation.

---

#### +Nyr dot (hollow circle)

A hollow circle (open ring) appears on each model's curve at the X-position representing your **projected future workload**. It appears only when both growth inputs are filled in:
- **IOPS growth rate (% per year):** how fast you expect the workload to grow annually.
- **Projection horizon (years):** how many years out you want to project.

Projected IOPS = `Current IOPS × (1 + growth rate / 100) ^ years` — standard compound annual growth.

A **"+Nyr"** label (where N is the number of years) appears above the topmost projected dot. The arrow from NOW dot to +Nyr dot is implicit: you read the story as "today we're here, in N years we'll be here, and that's the latency impact of that growth on each model."

If the +Nyr dot on a model lands in the amber or red zone while the NOW dot is in the green zone, that model has insufficient runway for the projected growth period and an upgrade or Grid conversation is warranted before that horizon.

---

#### Y-axis: Estimated latency (ms)

The Y-axis scale is dynamic — it auto-sizes to fit the data. It always shows enough range to make the NOW and +Nyr dots visible, and expands if the current workload is at a high utilisation where latency is elevated. Tick labels use two decimal places below 1 ms (e.g., "0.10"), one decimal place between 1 ms and 10 ms (e.g., "2.5"), and whole numbers above 10 ms.

The Y-axis range is anchored primarily to the 90th-percentile latency of each model's curve (roughly the latency at 90% of ceiling), then expanded to ensure current and projected dots are always visible. If all three dots are in the comfortable zone, the scale is tight and the flat region of the curve is clearly visible; as load increases toward the ceiling, the scale expands to show the hockey-stick shape.

---

#### X-axis: IOPS or GB/s

The X-axis extends from 0 to approximately 118% of the FS9600 ceiling, leaving room to display the "GRID →" zone and making the FS9600 ceiling marker clearly visible before the right edge. Tick intervals are automatically rounded to clean values (e.g., 1M, 2M, 3M... IOPS or 20, 40, 60... GB/s).

---

### 5.3 The latency model — what it is and what it is not

The latency curves use a **queueing-theory-inspired model** (M/D/1 queue, deterministic service time):

> `L(u) = L₀ × (1 + 1.5 × u² / (1 − 0.98u)^1.1)`

where:
- `L₀` = baseline latency at near-zero load for that model (0.10 ms for FS9600, 0.12 ms for FS7600, 0.15 ms for FS5600).
- `u` = utilisation = `load ÷ ceiling` (clamped to 0.9995 at the ceiling).
"The absolute breaking point of this system happens at 98% traffic."

**What this model gets right:** the shape — flat at low load, modest rise through the middle, steep hockey-stick above roughly 80–90% of ceiling. This shape is a fundamental property of any system with a finite service rate, not an IBM-specific quirk. It correctly conveys *why* you derate to 60% (the curve is still flat there) and *why* operating above 90% is dangerous (the curve goes nearly vertical).

**What this model does not do:** produce accurate latency numbers. The actual measured latency of a specific FlashSystem array depends on cache hit rates, workload sequentiality, queue depth, FlashCore Module generation, firmware version, and dozens of other factors not present in this tool. The curve values are deliberately illustrative — treat them as showing **relative behavior** across models and utilisation levels, not as latency SLAs or specifications.

The bottom of the chart legend includes a disclaimer to this effect for client-facing conversations.

---

### 5.4 The growth projection inputs

Two fields appear above the charts:

**IOPS growth rate (% per year):** the expected annual compound growth in IOPS demand. Leave blank if you want to show only the current-workload NOW dots without a projection. Common ranges: 10–20% for moderate enterprise growth, 30–50%+ for environments with active expansion or AI/analytics workload growth.

**Projection horizon (years):** how far out to project. Default assumption is 3 years if the field is left blank but a growth rate is entered. Valid range is 1–10 years.

Both fields are cleared by the **Clear** button. They do not affect any of the four result cards — they are purely for the chart's +Nyr dot.

---

### 5.5 How to use the charts in a client conversation

**Model selection:** when the customer hasn't decided which FlashSystem model they need, pull up the charts before entering any numbers. Walk them through the three curves and explain that each one represents a different "runway" — FS5600 gives you the shortest, FS9600 the longest. Then enter their IOPS and let the NOW dots land. If all three NOW dots are in green, you have options; if FS5600's dot is in red, that model is off the table before you've discussed price.

**Growth story:** enter the customer's projected growth rate and a 3-year horizon. Move the NOW and +Nyr dots in front of them and ask: "Today you're here. In three years, you're here. On the FS7600 that puts you in the marginal zone — on the FS9600 you're still comfortable. That's the conversation about whether the premium for the FS9600 is worth the headroom." This is a more credible framing than "you'll hit the ceiling in 3 years" because it shows the trajectory, not just a binary pass/fail.

**Grid discussion:** if the +Nyr dot crosses into the "GRID →" gray zone, the single-array story is over at that horizon. The gray zone is your opening to introduce FlashSystem Grid — point to it on the chart and explain that IBM's grid architecture picks up where the single enclosure leaves off, scaling to 32 nodes.

---

## 6. How to present a sizing using these results

A clean way to explain a design to a customer or manager, in their language:

> *"Your workload of 150,000 IOPS and 250 TB of data fits on a single FlashSystem 9600 with room to spare. Performance runs under 5% of the array's realistic limits, capacity uses a small fraction of available flash even after growth, and your data changes slowly enough that a standard 10 Gbps link keeps a near-real-time copy at your DR site. We sized one array, not two, because every dimension — speed, space, replication, and memory — sits comfortably in the green with headroom for several years of growth."*

That paragraph is exactly what the four green cards are telling you, translated into a business justification. The **assumed/purple** tags tell you which parts of that story rest on industry defaults and would be worth confirming with real measurements (block size, read/write split, and achievable data-reduction ratio are the three that most influence the outcome).

For the charts, a complementary line for client conversations: *"The performance curve shows this workload is sitting in the flat, comfortable part of the FS9600's range — well before the hockey stick. Even projecting 20% annual IOPS growth over three years, we stay in the green zone. That's why we're recommending the FS9600: not just for today, but for the next refresh cycle."*

---

## 7. Accuracy & calibration notes (important — please read)

The arithmetic in the tool is sound and internally consistent. This section documents the methodology choices and the corrections applied in this version, plus the few minor items still worth your attention. **The capacity basis, the RTT thresholds, and the FS9600 ceilings described below have been corrected in this build.**

### 7.1 Capacity card uses effective-vs-effective (corrected)

Earlier the capacity card reduced logical data by the DRR to a *physical/raw* demand and then divided it by an *effective* ceiling — mixing two different capacity units and understating utilisation for large datasets. This is now fixed: the fit verdict compares your **logical (effective) data plus headroom** against the array's **effective** capacity. Your DRR still appears, now driving the "physical flash consumed" informational line, which tells you how much NAND the data occupies.

> Capacity utilisation = `(logical data ÷ target%) ÷ (effective capacity in TB) × 100`

Note that the array's effective-capacity figure assumes a reference data-reduction ratio. If your workload reduces better or worse than that reference, the realised effective capacity scales with it — so treat the headline percentage as "good if your data reduces about as expected," and use the physical-flash line plus your measured DRR for a closer look.

### 7.2 Synchronous/asynchronous RTT bands (corrected)

The tool previously flipped from "PBHA synchronous (RPO 0)" to "PBR asynchronous" at 80 ms — which wrongly labelled metro-and-beyond links as zero-data-loss. It now uses three correct bands:
- **≤ 1 ms → PBHA synchronous.** IBM guidance is sub-millisecond ideal, because the application waits for the far site on every write.
- **1 ms to 80 ms → PBR asynchronous.** 80 ms is IBM's documented upper RTT limit for asynchronous PBR.
- **> 80 ms → not supported.** Beyond 80 ms the tool flags replication as unsupported rather than silently sizing it.

Both thresholds are exposed as overridable Advanced fields (PBHA sync RTT limit, PBR async max RTT) so you can adjust them if IBM updates its supported limits.

### 7.3 Array ceilings — current-generation models (FS9600 / FS7600 / FS5600)

The model selector loads the single-enclosure ceilings below. These replace the previous-generation values that were originally hard-coded, and you can override any of them in the Advanced panel.

| Ceiling | **FS9600** (5078-A40) | **FS7600** (5075-A30) | **FS5600** (5127-A20) |
|---|---|---|---|
| Peak IOPS (4K read-hit) | 6,300,000 | 4,300,000 | 2,600,000 |
| Bandwidth | 86 GB/s | 55 GB/s | 30 GB/s |
| Effective capacity (single enclosure) | 11.8 PBe | 7.2 PBe | 2.5 PBe |
| Effective capacity (32-system grid) | ~377 PBe | ~230 PBe | ~77 PBe |
| Node memory (default) | 1,536 GB | 768 GB | 512 GB |
| Form factor / drive slots | 2U / 32 | 2U / 32 | 1U / 12 |
| Largest FlashCore Module | 105.6 TB | 52.8 TB | 52.8 TB |

Notes on the figures:
- **IOPS** are IBM's best-case 4K read-hit numbers; the tool then applies the **60% mixed-workload derate** (shared across all three) before reporting utilisation. The derate threshold is also the green/amber boundary in the performance charts.
- **Bandwidth** is peak read bandwidth per single enclosure (the larger "grid" bandwidth figures sometimes quoted are 32-system aggregates — divide by 32 for a single box).
- **Effective capacity** assumes a reference data-reduction ratio baked into the FlashCore Modules; realised capacity scales with how well your data actually reduces (see 7.1).
- **Node memory** defaults to a documented, commonly-shipped configuration per model. The FS9600 figure (1,536 GB) is the per-node maximum; FS7600 and FS5600 also offer other memory options — override the field if your system differs. The memory model (20% OS reserve, 15% write cache) is a heuristic and applies identically across models.
- **Shared software limits** (PBHA sync ≤ 1 ms, PBR async ≤ 80 ms, 60% derate) are Storage Virtualize 9.1.2 behaviours, not model-specific.
- **Chart baseline latencies** used in the performance scaling curves (FS9600: 0.10 ms, FS7600: 0.12 ms, FS5600: 0.15 ms) are illustrative reference points, not published IBM specifications. They represent approximate NVMe all-flash response times at near-zero load for similarly-classed arrays and are used solely to give the latency curves a realistic shape.

These remain *estimates*. Always confirm current limits on IBM's official *Configuration limits* page for v9.1.x and in the IBM Configurator before quoting a bill of materials, since marketing figures and formally supported limits can differ.

### 7.4 Minor notes

- **Decimal units throughout.** The tool uses decimal conventions (1 MB = 1000 KB, 1 GB/s = 125 MB/s, 1 PB = 1000 TB), which matches how IBM markets bandwidth and capacity. Just be aware these are ~2–7% smaller than the binary (1024-based) equivalents an OS might report.
- **Journal buffer multiplies RTT by 2.** The field is labelled "RTT" (already round-trip), and the buffer formula multiplies it by 2 again. If RTT is genuinely round-trip, this over-sizes the buffer roughly 2×. It errs on the safe (over-provisioned) side and the amounts are tiny, but the intent should be made consistent: either label the input one-way latency, or drop the ×2.
- **Performance ceiling vs block size.** The IOPS ceiling is a 4K-read-hit figure but the workload may use larger blocks. The tool correctly compensates by also checking bandwidth and taking the worse of the two, so this is handled — just know that the IOPS-utilisation line alone shouldn't be read in isolation for large-block workloads.
- **Chart zone boundaries use the FS9600 as the reference model.** The green/amber/red/gray zone fills are fixed to FS9600's ceiling so all three curves can be compared on a single background. This means the FS7600 curve ends inside the amber zone (its ceiling is at ~68% of the IOPS X-axis) and the FS5600 curve ends in the green zone (ceiling at ~41%). This is intentional — it visually shows that FS7600 and FS5600 "run out" before the X-axis does, reinforcing why larger models or a Grid are needed beyond those points.
- **Unused constant.** A `perVolumeKB` (per-volume metadata) constant is defined in the memory model but never used; harmless, but it implies a per-volume overhead the card doesn't actually compute. If per-volume metadata matters for your sizing, wire it in; otherwise remove it to avoid confusion.

---

## 8. Quick reference: every formula in one place

| Result | Formula (as implemented) |
|---|---|
| Read IOPS | `Total IOPS × read%` |
| Write IOPS | `Total IOPS × (100 − read%)` |
| Throughput (MB/s) | `(IOPS ÷ 1000) × block KB` |
| Throughput (GB/s) | `MB/s ÷ 1000` |
| Usable IOPS ceiling | `peak IOPS × derate%` |
| IOPS utilisation | `IOPS ÷ usable IOPS × 100` |
| Bandwidth utilisation | `GB/s ÷ ceiling GB/s × 100` |
| Logical (effective) data | `data today + growth` |
| Physical flash consumed (info) | `logical ÷ DRR` |
| Effective needed with headroom | `logical ÷ target%` |
| Capacity utilisation | `(effective-with-headroom in PBe) ÷ array max PBe × 100` |
| Replication mode | `RTT ≤ 1 ms → PBHA sync` · `1–80 ms → PBR async` · `> 80 ms → not supported` |
| Write rate (MB/s) | `write IOPS × block KB ÷ 1000` |
| On-wire (after compression) | `write MB/s ÷ stream compression` |
| On-wire (Gbps) | `on-wire MB/s ÷ 125` |
| Usable link (MB/s) | `link Gbps × 125 × 0.80` |
| Link utilisation | `on-wire MB/s ÷ usable link × 100` |
| Sync journal buffer (MB) | `write MB/s × (RTT ms ÷ 1000) × 2` |
| Usable memory | `node memory × (1 − 0.20)` |
| Write cache | `usable memory × 0.15` |
| Read cache | `usable memory × 0.85` |
| Memory utilisation | `(write cache + journal buffer) ÷ usable memory × 100` |
| **Chart: current throughput (GB/s)** | `(IOPS ÷ 1000) × block KB ÷ 1000` |
| **Chart: projected IOPS** | `current IOPS × (1 + growth% ÷ 100) ^ years` |
| **Chart: estimated latency** | `L₀ × (1 + 1.5 × u² ÷ (1 − 0.98u)^1.1)` where `u = load ÷ ceiling`, `L₀` = model baseline ms |

---

*Estimates for pre-sales planning. Always confirm final performance, capacity, and replication limits with the IBM Configurator and the current IBM Storage Virtualize 9.1.2 configuration-limits documentation before quoting a bill of materials.*
