# IBM FlashSystem — Storage Virtualize 9.1.2 Sizing Calculator
## Technical README & Plain-Language Guide — **v5**

*What every number means, what the results are telling you, and how to use them to explain a storage design — written so that someone new to storage sizing can follow it.*

> **What changed in v5 (read this if you used an earlier build).** The performance engine was recalibrated against **IBM Storage Modeller** output. Three things moved materially: (1) the latency curve is now a textbook queueing model instead of an ad-hoc curve; (2) the IOPS "fit" is measured against a realistic **mixed-workload saturation point** rather than a flat derate of the datasheet peak — which lowered the usable-IOPS ceiling roughly 3–4× and made it match IBM's tooling; and (3) throughput now sizes **reads and writes separately** by transfer size. Because these change the output numbers, **v5 exports should not be mixed with earlier ones.** Full detail in Section 7.

---

## 1. What this tool is for

When you buy or design an enterprise storage array, you have to answer four practical questions **before** you sign off on the hardware:

1. **Can the array keep up with the work?** (speed)
2. **Will the data fit, with room to grow?** (capacity)
3. **If the site burns down, how do we get a copy somewhere else, and how fresh will that copy be?** (replication / disaster recovery)
4. **Does the array have enough memory to do all of the above well?** (cache & buffers)

This calculator gives a quick, defensible first-pass answer to all four for the current-generation **IBM FlashSystem** arrays — **FS9600**, **FS7600**, and **FS5600** — running **Storage Virtualize 9.1.2**. You pick the model at the top of the panel and the tool measures your workload against that array's ceilings. Below the four result cards, a **Performance Scaling Charts** section overlays all three models on the same graph so you can instantly see which model gives the right amount of headroom and where growth eventually forces an upgrade or a FlashSystem Grid.

### This is a teaching-and-estimation tool, not a substitute for Storage Modeller

A banner at the top of the tool states this plainly, and it matters: **this calculator shows the formulas behind the numbers so you can explain *how you got there*.** It is directional and not court-defensible. IBM Storage Modeller (and Storage Insights, for measured telemetry) remain the authoritative sources — Storage Modeller produces the report, the graph, and the number you commit to; this tool complements it by making the arithmetic visible in a client conversation. The two are designed to agree: for the calibrated reference workload, the calculator's FS9600 results now land within a few percent of Storage Modeller (see Section 7). Always confirm the final configuration in the official IBM Configurator and the current *Configuration limits* documentation before quoting a bill of materials.

### The "enter what you know" philosophy

You are not expected to know every input. You type in the few things you do know (for example, "we do about 150,000 IOPS and have 200 TB of data"), and the tool fills in everything else with sensible **defaults**. Any value the tool assumed for you is tagged in **purple** with the word *assumed*. That tag is important: it tells the reader of your sizing exactly which numbers came from the customer and which were industry-standard guesses. When you can replace an assumed value with a real measured one, your sizing becomes more trustworthy.

### The traffic-light system

Every result card shows a colored verdict so you can read it at a glance:

| Color | Meaning |
|-------|---------|
| 🟢 **Green** | Comfortably within spec. The array has headroom. |
| 🟡 **Yellow / Amber** | Marginal. It works, but there's little room for spikes or growth. Review it. |
| 🔴 **Red** | Over the limit. The design needs a bigger or additional array. |

"Utilisation" simply means *how much of the array's maximum you are planning to use*. Lower is safer; you generally want to land in green.

**The bands differ by card, on purpose:**

- The **Performance card** now uses **IBM Storage Modeller's own core thresholds**: green up to **60%** (IBM's amber line), amber **60–80%** (IBM's red line), red above **80%**. This is deliberately conservative and lets you cross-check a verdict directly against a Storage Modeller run.
- The **Capacity, Replication, and Memory cards** use the general **70% / 90%** bands (green ≤ 70%, amber 70–90%, red > 90%).

### Choosing your model

At the top of the input panel is a **FlashSystem model** selector covering the three current-generation arrays: **FS9600** (the flagship), **FS7600** (mid-to-upper enterprise), and **FS5600** (entry-to-midrange). Picking a model loads that array's performance, bandwidth, capacity, and memory ceilings — and, in v5, its calibrated **saturation point** and **base latency** — so every result and every chart zone is measured against the right hardware. The exact ceilings are in Section 7.3. The replication behaviour (sync/async RTT limits) and the mixed-workload model are Storage Virtualize software characteristics shared across all three (and shared with SVC SV3 nodes, which run the same software). If you're sizing a multi-array **FlashSystem Grid**, override the capacity ceiling with the grid figure shown for that model.

---

## 2. A 60-second glossary (read this first if storage is new to you)

**IOPS** — *Input/Output Operations Per Second.* The number of individual read or write requests the array handles each second. Think of it as "how many small errands per second." Databases and virtual machines generate lots of small errands, so they care about IOPS.

**Transfer size / block size** — How big each errand is, in kibibytes (KiB). A bigger block moves more data per errand. Typical mixed/database reads are about **8 KiB**; **writes are often larger** (16 KiB is common), which is why v5 lets you size reads and writes separately.

**Throughput / Bandwidth** — How much *total data* moves per second, in **MB/s** or **GB/s**. In v5 it's computed per operation type: `read IOPS × read size + write IOPS × write size`. Two systems can do the same IOPS but very different bandwidth if their block sizes differ. Backups and analytics care about bandwidth more than IOPS.

**Read/Write split** — What fraction of the errands are reads versus writes. A common default is **70% read / 30% write**. This matters because writes are heavier work for flash, and because **only writes need to be replicated** to a disaster-recovery site (reading data doesn't change it).

**Latency / response time** — How long, in milliseconds (ms), the application waits for the array to complete a single read or write. It rises as the array approaches its ceiling — slowly at first, then very steeply (the "hockey stick," below).

**RTT** — *Round-Trip Time.* How long a signal takes to travel to the DR site and back. Distance adds RTT because signals aren't instant. RTT is the single most important number deciding *which kind* of replication you can use.

**Saturation point (v5)** — the workload level at which the array's controller core effectively hits 100% for a **typical mixed workload** — i.e. where latency runs away. This is the honest ceiling for real workloads, and it's *much* lower than the datasheet "peak IOPS" figure (which is a best-case 4K-read-hit lab number). The calculator's FS9600 saturation point is calibrated directly to IBM Storage Modeller.

**Comfort target** — the percentage of the saturation point you're willing to plan up to (default **60%**, matching IBM's amber line). Comfortable ceiling = `saturation × comfort%`.

**Data reduction (DRR)** — FlashSystem squeezes data with **compression** and **deduplication** before storing it. A **3:1** ratio means 3 TB of your data fits in 1 TB of physical flash.

**Capacity flavors:**
- **Raw / usable capacity** = the actual physical flash you bought (after RAID overhead), in **PBr** (petabytes raw).
- **Effective capacity** = how much *logical* data you can store *after* data reduction, in **PBe** (petabytes effective). This is what you usually size against, because it's what the customer's data actually consumes.

**Headroom / target utilisation** — You never fill an array to 100%. A **capacity target of 80%** means "plan to use at most 80% of the array."

**RPO** — *Recovery Point Objective.* How much recent data you can afford to lose. **RPO = 0** means no loss; **RPO = seconds** means you might lose the last few seconds of writes.

**RTO** — *Recovery Time Objective.* How fast you must be back up after a failure.

**Cache** — Fast memory (DRAM) inside the array holding hot data. Split into read cache and write cache.

**Cache hit rate** — the fraction of I/O served from cache instead of flash. It is **emergent, not calculable from workload specs alone** — it depends on working-set size, locality, and time-of-day, and can only be *measured* (e.g. with Storage Insights). v5 exposes optional cache-efficiency fields as a **sensitivity** lever, not a prediction (Section 4, Card 1).

**Queueing / hockey-stick curve** — As any system nears its maximum, response time rises slowly, then steeply. The latency formula (`service time ÷ (1 − utilisation)`) is the textbook expression of this and is why you plan to a comfort target well below saturation.

**FlashSystem Grid** — IBM's scale-out architecture clustering up to 32 FlashSystem nodes into one system, multiplying effective capacity (and, to a degree, performance). The charts label the region past a single array's ceiling "GRID →".

---

## 3. The replication concepts, explained simply

IBM gives you two related ways to keep a second copy of your data at another site.

**Policy-Based HA (PBHA) — the "two arms always in sync" option.** Both sites are live and every write is mirrored to the other site *before the application is told "done."* Nothing is lost (**RPO = 0**) and failover is automatic (**RTO = 0**). The catch: the application waits for the far site on every write, so the sites must be **very close, with very low RTT** — sub-millisecond ideal, metro distance. This is **synchronous** replication.

**Policy-Based Replication (PBR) — the "shipping copies behind the scenes" option.** Writes are confirmed locally and copied to the far site shortly after, so **RPO > 0** but the sites can be much farther apart. This is **asynchronous** replication, supported up to about **80 ms** RTT.

Within async PBR there are two automatic sub-modes:

- **Journaling** — the link keeps up with the change rate; the copy stays only seconds behind (RPO ≈ seconds). Good.
- **Cycling** — the link *can't* keep up, so the system falls back to periodic snapshots; the copy is further behind (larger RPO). A sign you need a faster link or lower change rate.

You don't choose journaling vs cycling — the system picks based on whether your link can carry your write rate. The calculator predicts which one you'll land in.

**One crucial sizing insight:** ongoing replication bandwidth is driven by your **write rate**, not by how much total data you have. A 500 TB system that changes 50 MB/s of data needs only enough link to carry 50 MB/s. (The exception is the one-time **initial sync**, which copies the whole dataset once — plan extra time or bandwidth for that; this tool sizes steady-state, not the seed.)

---

## 4. Walk-through of each result card

Each card is explained as: **the question it answers → the inputs it uses → what each output means → how to read the verdict.** The worked numbers use the built-in **Load Sample** scenario on the default **FS9600**: 150,000 IOPS, 70/30 read/write, **8 KiB reads / 16 KiB writes**, 200 TB data + 50 TB growth, 3.5:1 reduction, 80% capacity target, 12 ms RTT, 10 Gbps link, 2:1 stream compression.

---

### Card 1 — Local Performance: *"Can the system handle this workload?"*

**Needs:** Total IOPS. **Assumes if blank:** read/write split, block sizes, and the array's speed ceilings.

**IOPS breakdown.**
- Read IOPS = `Total IOPS × read%` → 150,000 × 70% = **105,000**
- Write IOPS = `Total IOPS × write%` → 150,000 × 30% = **45,000**

The write portion is what later drives replication bandwidth and write-cache needs.

**Throughput (v5: transfer-size aware).** v5 sizes reads and writes separately, using each block size, and computes the true byte rate (binary KiB) displayed as decimal MB/s:
> `(read IOPS × read KiB + write IOPS × write KiB) × 1024 ÷ 1,000,000 = MB/s`

For the sample:
- Read throughput = 105,000 × 8 KiB → **860 MB/s**
- Write throughput = 45,000 × 16 KiB → **737 MB/s**
- **Total throughput = 1,597 MB/s = 1.6 GB/s** (which is **1,523 MiB/s**, shown in parentheses so you can cross-check against Storage Modeller, which reports MiB/s)

This is a genuine v5 correction: the old build applied one flat 8 KB block to everything and reported 1,200 MB/s. Sizing the 16 KiB writes separately raises the honest figure to 1,597 MB/s — matching Storage Modeller's byte rate exactly.

**System headroom (v5: saturation-based).** The array has two independent speed limits, and your workload must fit under **both**:

- **Mixed-workload saturation.** Instead of derating a best-case peak, v5 measures against the IOPS level where the controller core saturates for a real mixed workload. For FS9600 this is **1,500,000 IOPS** (calibrated to Storage Modeller). The datasheet "4K read-hit peak" (6,300,000) is still shown, labelled as such, purely as a theoretical-max reference.
  - **Comfortable ceiling** = `saturation × comfort%` → 1,500,000 × 60% = **900,000 IOPS**
  - **Core utilisation** = `your IOPS ÷ saturation` → 150,000 ÷ 1,500,000 = **10.0%**. This figure is designed to line up with Storage Modeller's **"System Core %"** — you can read it straight across.
  - **Estimated latency at this load** = `base service time ÷ (1 − utilisation)` → 0.113 ÷ (1 − 0.10) = **0.13 ms** (Storage Modeller's exact figure for this point is 0.126 ms).

- **Bandwidth ceiling.** The most data-per-second the array can push. **Bandwidth utilisation** = `your GB/s ÷ ceiling GB/s` → 1.6 ÷ 86 = **1.9%**.

The verdict reflects **whichever of the two is higher** (the binding constraint), against the IBM 60% / 80% bands. At 10% core and 1.9% bandwidth, the sample is **FITS** (green).

**Optional — Workload sensitivity.** A collapsed, optional section lets you enter **Sequential I/O %**, **Cache efficiency – random %**, and **Cache efficiency – sequential %**. Leave them blank and they do nothing — the standard result is unchanged. If you populate the cache-efficiency fields, the tool adds a *separate, clearly-flagged* "sensitivity-adjusted saturation" line showing how much the ceiling would move for a more- or less-cache-friendly workload (bounded to ±25% so it can't masquerade as hardware-grade precision). Every appearance carries the caveat that **cache hit rate is emergent and must be validated against Storage Insights** before it informs a commitment. This is a reasoning lever for "how sensitive is my sizing to a number I haven't measured yet," not a prediction.

**Helper notes you may see:**
- *Write-heavy warning* (over 50% writes): flash write amplification lowers effective IOPS, so the real saturation may sit below the estimate — validate in Storage Modeller.
- *Bandwidth-bottleneck warning*: your block size is large enough that you'll run out of GB/s before IOPS.

**How this justifies the sizing:** "At 150,000 IOPS the FS9600 core sits at 10% utilisation — the same figure Storage Modeller's System Core reports — with latency near its 0.11 ms floor and bandwidth under 2%. Green across the board, with the comfortable ceiling (900k IOPS) leaving years of runway."

---

### Card 2 — Capacity: *"Will the data fit, with room to grow?"*

**Needs:** Data today (TB). **Assumes if blank:** data reduction ratio, target utilisation, array max capacity. Growth is optional.

**Logical (effective) data.** Your real-world data, today plus growth:
> `Data today + Growth` → 200 + 50 = **250 TB logical**

This logical figure is your **effective footprint** — the effective capacity it consumes on the array.

**Physical flash consumed (informational).** How much actual flash the data occupies once compression/dedup work:
> `Logical ÷ DRR` → 250 ÷ 3.5 = **71.4 TB** of physical flash

This uses *your* DRR and helps you understand NAND burn and endurance headroom. It does not drive the verdict.

**Effective needed with headroom.** Because you never fill to 100%:
> `Logical ÷ target%` → 250 ÷ 80% = **312.5 TB = 0.313 PBe**

**Array fit.** Compared against the array's **effective** capacity ceiling:
> Capacity utilisation = `(logical ÷ target%) ÷ effective-capacity` → 0.313 ÷ 11.8 PBe ≈ **2.6%**

Green means it fits comfortably; red means you need a larger model or a Grid.

> **Why effective-vs-effective?** Effective capacity already accounts for data reduction, so the honest comparison is logical-data against effective-capacity (both post-reduction terms). The 11.8 PBe figure assumes a reference reduction ratio — if your data reduces better or worse, realised effective capacity scales accordingly. A single FS9600 enclosure tops out near 11.8 PBe; a 32-system grid scales to roughly 377 PBe.

**How this justifies the sizing:** "250 TB is ~2.6% of the array's ~11.8 PBe effective capacity even after growth and headroom, occupying only ~71 TB of physical flash — capacity is not a constraint, with substantial room to grow."

---

### Card 3 — Replication & HA: *"Which DR mode, how much link, how fresh a copy?"*

**Needs:** RTT to the DR site. **Uses if available:** IOPS (to derive write rate) and WAN link speed. **Assumes if blank:** stream compression, block sizes, read/write split.

**Replication mode.** The tool sorts RTT into three bands:
- **≤ 1 ms → PBHA SYNC**, RPO = 0 (metro only).
- **1 ms to 80 ms → PBR ASYNC**, RPO > 0, refined into journaling or cycling once it knows your link speed.
- **> 80 ms → not supported** — beyond the documented RTT limit for async PBR; the card turns red.

Both thresholds are overridable in Advanced if IBM's guidance changes.

**Replication bandwidth (v5: sized on write transfer size).** Replication carries **writes only**, and v5 sizes them at the *write* block size:
> Write rate = `write IOPS × write KiB` → 45,000 × 16 KiB ≈ **737 MB/s**

Then subtract what compression saves on the wire:
> On-wire = `write rate ÷ stream compression` → 737 ÷ 2 ≈ **369 MB/s = 2.9 Gbps**

(Note: with 8 KiB writes this would be ~369 MB/s / ~185 MB/s on-wire — the sample's 16 KiB writes double it. This is exactly the kind of thing the read/write split now surfaces honestly.)

**Link capacity & utilisation.** Links are quoted in Gbps, data in MB/s (`1 Gbps = 125 MB/s`), and the tool reserves ~20% for overhead:
> Usable link = `Gbps × 125 × 80%` → 10 × 125 × 0.8 = **1,000 MB/s**
> Link utilisation = `on-wire ÷ usable link` → 369 ÷ 1,000 ≈ **36.9%**

Green means the pipe keeps up and async runs in the good *journaling* mode; high utilisation (red) means you'll fall into *cycling* (larger RPO) — fix with a faster link or lower change rate.

**Journal buffer (sync only).** In synchronous mode the array briefly holds writes in memory awaiting far-site confirmation: `Buffer ≈ write rate × RTT`. At 12 ms the sample is async, so this is 0.

**How this justifies the sizing:** "At 12 ms RTT we replicate asynchronously with PBR. The workload's 45,000 writes at 16 KiB change ~737 MB/s, compressing to ~369 MB/s on the wire — about 37% of a 10 Gbps link — so the copy stays seconds-fresh in journaling mode, with room to spare."

---

### Card 4 — System Memory: *"Is there enough DRAM for cache and replication buffers?"*

**Needs:** IOPS (for write-cache context) and/or RTT (for sync buffer). **Assumes if blank:** node memory, read/write split, block sizes.

**Node memory allocation.** From the array's total memory (FS9600 default **1,536 GB**), carve out:
- **OS + metadata reserve** (~20%) → 307 GB
- **Usable for cache** → 1,229 GB

**Cache allocation.** Usable cache splits into a smaller **write cache** (~15% → 184 GB) and a larger **read cache** (~85% → 1,045 GB). Reads benefit from more cache; the write cache absorbs bursts before flushing.

**PBHA journal buffer.** In synchronous mode the in-flight buffer from Card 3 also lives in memory and is added on top. (Async → 0.)

**Memory utilisation.** Write cache + journal buffer as a percent of usable memory → sample **15.0%**. With the FS9600's large DRAM this almost always lands green; the card mainly flags the rare case where a very high write rate over a sync link makes the journal buffer large enough to crowd read cache.

**How this justifies the sizing:** "Cache and replication buffers use ~15% of usable node memory, so the standard configuration is sufficient — no memory upgrade required."

---

## 5. Performance Scaling Charts — full guide and legend

The Performance Scaling section (below the four cards, expandable) contains two charts — one for IOPS, one for throughput — plotting all three FlashSystem models on the same axes. The goal is to show not just whether today's workload fits, but **where on the latency curve it sits, and at what growth point each model runs out of runway**. Both charts update live as you type.

---

### 5.1 The two charts

**Chart 1 — IOPS vs. Estimated Latency**
- X-axis: Total IOPS. In v5 the domain is anchored to the largest model's **saturation point** (~1.5M for FS9600), expanding if your current or projected workload is higher.
- Y-axis: Estimated per-I/O latency (ms), auto-scaled.
- Three curves, one per model, each running to that model's own **saturation point**.

**Chart 2 — Throughput vs. Estimated Latency**
- X-axis: Total throughput (GB/s), to beyond the FS9600 bandwidth ceiling (~86 GB/s).
- Y-axis: same latency scale.
- Three curves, one per model, to each model's bandwidth ceiling.

The throughput figure uses the same transfer-size-aware math as Card 1.

---

### 5.2 Complete element legend

#### Background zone fills — **now follow the selected model (v5 fix)**

The chart background is divided into colored bands. **In v5 the boundaries follow the model you've selected**, so the shaded bands mean the same thing as the Performance card's verdict for that model. (Earlier builds anchored the bands to FS9600 regardless of selection, which made a smaller selected model look mis-scaled.)

| Zone label | Color | X-axis boundary (selected model) | What it means |
|---|---|---|---|
| **COMFORTABLE** | Faint green | 0 → comfort% of saturation (default 0 → 60%) | Below the comfort target. Latency is low and flat. Target operating region. |
| **MARGINAL** | Faint amber | comfort% → 87.5% of saturation | Latency begins rising. Works, but little headroom for bursts or growth. |
| **LATENCY↑** | Faint red | 87.5% → 100% of saturation | Latency climbing steeply — the queueing hockey-stick. Risky under any spike. |
| **GRID →** | Faint gray | Beyond the model's saturation | A single array is effectively saturated; a FlashSystem Grid continues the scaling story. |

For the FS9600 sample, that puts the green band at 0 → 900,000 IOPS, amber 900,000 → 1,312,500, red 1,312,500 → 1,500,000, then GRID beyond.

> If you override the comfort% in the Advanced panel, the green/amber boundary shifts and the coloring updates live.

---

#### Model curves — **selected model emphasised (v5)**

Three curves are drawn, one per model. In v5 the **selected model's curve is bold and fully opaque; the other two are faded to thin context lines**, so your eye tracks the curve the verdict and zones actually describe. Each curve runs from zero load to that model's own saturation point (where it ends).

| Curve color | Model | Saturation (mixed) | Datasheet peak (ref) | BW ceiling | Base latency (~0 load) |
|---|---|---|---|---|---|
| **Cyan** `#00b4d8` | FlashSystem 9600 | **1,500,000 IOPS** (calibrated) | 6,300,000 | 86 GB/s | 0.113 ms |
| **Purple** `#7c3aed` | FlashSystem 7600 | **850,000 IOPS** (estimate) | 4,300,000 | 55 GB/s | 0.125 ms |
| **Amber/gold** `#f59e0b` | FlashSystem 5600 | **450,000 IOPS** (estimate) | 2,600,000 | 30 GB/s | 0.140 ms |

Reading them together: FS5600 saturates first (shortest runway), FS7600 next, FS9600 last. The **datasheet peak** columns are the theoretical 4K-read-hit maxima and are *not* the axis — they're shown only as the contrast that motivates the mixed-workload saturation.

> **Why the weaker model can *look* better on the raw curves.** Because each curve blows up near *its own* saturation and the Y-axis is capped, a lower-saturation model (say FS7600) shoots off the top of the chart early, leaving only its comfortable left segment visible — while FS9600, being stronger, stays on-chart longer and is the one you *see* climbing into the amber/red bands. At any shared IOPS point the bigger model is always lower-latency; the v5 emphasis (bold selected curve, per-model saturation markers) is what keeps this from being misread.

---

#### Vertical dashed saturation markers

A dashed vertical line in each model's color runs the height of the chart at that model's **saturation point**, with a small label below the X-axis (**FS9600 / FS7600 / FS5600**). The **selected** model's marker is emphasised and tagged with a ◄ pointer. These make it easy to read off "if my workload were X IOPS, which models can still handle it?"

#### NOW dot (filled circle)

A filled circle sits on **each model's curve** at your current workload's X-position as soon as you enter IOPS. Colors match the curves; a **"NOW"** label sits above the topmost (lowest-latency) dot. If your IOPS exceed a model's saturation, its NOW dot is pinned at that model's saturation point (rightmost end of its curve).

#### +Nyr dot (hollow circle)

A hollow ring appears on each curve at your **projected** workload, when both growth inputs are filled:
- **IOPS growth rate (% per year)** and **Projection horizon (years)**.
- Projected IOPS = `Current IOPS × (1 + growth% ÷ 100) ^ years`.

A **"+Nyr"** label sits above the topmost projected dot. If a model's +Nyr dot lands in amber/red while NOW is green, that model lacks runway for the projected horizon — an upgrade or Grid conversation is warranted.

#### Axes

- **Y-axis (latency, ms)** auto-sizes to keep the NOW and +Nyr dots visible, anchored near the 90th-percentile latency of the visible curves and expanding as load rises toward saturation. Tick labels: 2 dp below 1 ms, 1 dp from 1–10 ms, whole numbers above 10 ms.
- **X-axis (IOPS or GB/s)** runs from 0 to ~118% of the reference saturation/ceiling, with clean rounded tick intervals.

---

### 5.3 The latency model — what it is and what it is not

v5 uses the **textbook queueing response-time formula** (M/M/1 sojourn time with constant service time):

> `L(u) = base service time ÷ (1 − u)`   where `u = load ÷ saturation` (clamped just below 1)

- `base` = base service time at near-zero load (FS9600 0.113 ms, FS7600 0.125 ms, FS5600 0.140 ms).
- `saturation` = the model's mixed-workload saturation IOPS (or its bandwidth ceiling, on the throughput chart).

**What this model gets right:** it is calibrated. For FS9600 it reproduces IBM Storage Modeller's response-time curve to within about **5%** across the operating range — e.g. at 150k IOPS it gives 0.126 ms vs Storage Modeller's 0.1259 ms, and the FITS / MARGINAL / OVER transitions land on Storage Modeller's own amber (60%) and red (80%) core thresholds. The shape — flat, then a knee, then near-vertical — is a fundamental property of any finite-service-rate system, and it's why you plan to a comfort target well below saturation.

**What this model does not do:** replace Storage Modeller. It runs slightly *optimistic* in the mid-range (a few percent below Storage Modeller's latency around 40–60% load), which is inherent to the simple M/M/1 form. Real measured latency also depends on cache hit rate, sequentiality, queue depth, FlashCore Module generation, and firmware — none of which this single formula captures. Treat the curve as **calibrated-directional**: good enough to explain relative behaviour and land in the right ballpark, not a latency SLA. The chart legend carries a disclaimer to this effect for client-facing use.

---

### 5.4 The growth projection inputs

Two fields above the charts:

- **IOPS growth rate (% per year)** — expected annual compound growth. Leave blank to show only the NOW dots. Common ranges: 10–20% moderate; 30–50%+ for active expansion or AI/analytics growth.
- **Projection horizon (years)** — how far out to project (default 3 if a rate is entered; valid 1–10).

Both are cleared by **Clear** and affect only the chart's +Nyr dots, not the four result cards.

---

### 5.5 How to use the charts in a client conversation

**Model selection:** before entering numbers, walk the customer through the three curves — each is a different "runway," FS5600 shortest, FS9600 longest. Enter their IOPS and let the NOW dots land. All green → you have options; FS5600 in red → it's off the table before price comes up.

**Growth story:** enter their growth rate and a 3-year horizon. "Today you're here; in three years you're here. On FS7600 that's marginal — on FS9600 you're still comfortable. That's the conversation about whether the FS9600 premium buys worthwhile headroom." More credible than a binary pass/fail because it shows the trajectory.

**Grid discussion:** if a +Nyr dot crosses into "GRID →", the single-array story ends at that horizon — your opening to introduce FlashSystem Grid.

---

## 6. How to present a sizing using these results

A clean way to explain a design, in business language:

> *"Your workload of 150,000 IOPS and 250 TB of data fits on a single FlashSystem 9600 with room to spare. The controller core runs at about 10% — the same figure IBM's own Storage Modeller reports — capacity uses under 3% of available effective flash even after growth, and your data changes slowly enough that a 10 Gbps link keeps a seconds-fresh copy at your DR site. We sized one array, not two, because every dimension — speed, space, replication, and memory — sits comfortably in the green with headroom for several years of growth."*

The **assumed/purple** tags tell you which parts rest on defaults worth confirming (block sizes, read/write split, and achievable DRR are the three that most influence the outcome). And because the performance numbers are now calibrated to Storage Modeller, you can add: *"these figures line up with a Storage Modeller run for this workload — this tool just shows the arithmetic behind them."*

For the charts: *"The performance curve shows this workload in the flat, comfortable part of the FS9600's range — well before the hockey stick. Even at 20% annual IOPS growth over three years, we stay green. That's why we recommend the FS9600: not just for today, but for the next refresh cycle."*

---

## 7. Accuracy & calibration notes (important — please read)

The arithmetic is sound and internally consistent, and every formula is on-screen so you can audit it. This section documents the v5 methodology and its honest limits.

### 7.1 The performance engine is calibrated to IBM Storage Modeller (v5)

The headline change. The tool was validated against an actual **IBM Storage Modeller** export for an FS9600 mixed workload (70/30, 8/16 KiB, ~55% cache hit). Three corrections came out of it:

1. **Realistic saturation, not a derated peak.** Earlier builds took the datasheet 4K-read-hit peak (6.3M IOPS) and multiplied by a flat 60% derate to get a "usable" ~3.78M ceiling. Storage Modeller shows the FS9600 core actually saturates near **1.5M IOPS** for a real mixed workload — roughly **3–4× lower**. v5 measures against that calibrated saturation point; the 6.3M peak is retained only as a labelled theoretical-max reference. Core utilisation now equals `IOPS ÷ saturation` and reads directly against Storage Modeller's "System Core %."
2. **Queueing latency.** The ad-hoc curve was replaced with `service time ÷ (1 − utilisation)`, which reproduces Storage Modeller's FS9600 response times to within ~5% (Section 5.3).
3. **Transfer-size-aware throughput.** Reads and writes are sized separately by block size, and the byte rate uses binary KiB — so throughput matches Storage Modeller's figure exactly (1,597 MB/s = 1,523 MiB/s for the sample) instead of the old flat-block 1,200 MB/s.

**Honest caveats:**
- Only **FS9600 is calibrated directly** to this Storage Modeller file. **FS7600 (850k) and FS5600 (450k) saturation points are scaled estimates**, labelled as such and overridable per model in the Advanced panel. If you obtain Storage Modeller runs for those two at a matched workload, drop the real saturation numbers into the override fields and they become exact too.
- The FS7600 tab in the source export used a *different* cache-hit assumption (75%) than FS9600 (~51%) — a different workload, not just different hardware — so FS9600 is the anchor and the others are scaled from it rather than read directly.
- The queueing model is slightly **optimistic** (a few percent low on latency) in the 40–60% range. It's disclaimed, and it's a rounding error next to the old 3–4× ceiling error.

### 7.2 Capacity card uses effective-vs-effective

The fit verdict compares your **logical (effective) data plus headroom** against the array's **effective** capacity (both post-reduction terms). Your DRR drives the informational "physical flash consumed" line.
> Capacity utilisation = `(logical ÷ target%) ÷ (effective capacity in TB) × 100`

The effective-capacity figure assumes a reference DRR; if your data reduces better or worse, realised effective capacity scales with it — so treat the headline percentage as "good if your data reduces about as expected," and use the physical-flash line plus your measured DRR for a closer look.

### 7.3 Array ceilings — current-generation models

The model selector loads the single-enclosure ceilings below; override any of them in Advanced.

| Ceiling | **FS9600** (5078-A40) | **FS7600** (5075-A30) | **FS5600** (5127-A20) |
|---|---|---|---|
| Datasheet peak IOPS (4K read-hit) | 6,300,000 | 4,300,000 | 2,600,000 |
| **Mixed-workload saturation (v5)** | **1,500,000** (calibrated) | **850,000** (est.) | **450,000** (est.) |
| Base service time (chart, ~0 load) | 0.113 ms | 0.125 ms | 0.140 ms |
| Bandwidth | 86 GB/s | 55 GB/s | 30 GB/s |
| Effective capacity (single enclosure) | 11.8 PBe | 7.2 PBe | 2.5 PBe |
| Effective capacity (32-system grid) | ~377 PBe | ~230 PBe | ~77 PBe |
| Node memory (default) | 1,536 GB | 768 GB | 512 GB |
| Form factor / drive slots | 2U / 32 | 2U / 32 | 1U / 12 |
| Largest FlashCore Module | 105.6 TB | 52.8 TB | 52.8 TB |

Notes:
- **Datasheet peak IOPS** are IBM's best-case 4K-read-hit numbers — a reference only. The tool sizes against the **mixed-workload saturation** row and the **comfort %** (default 60%, also the green/amber chart boundary).
- **Bandwidth** is peak read bandwidth per single enclosure (larger "grid" figures are 32-system aggregates — divide by 32 for one box).
- **Effective capacity** assumes a reference DRR; realised capacity scales with how well your data reduces (see 7.2).
- **Node memory** defaults to a documented, commonly-shipped configuration; FS9600's 1,536 GB is the per-node maximum. Override if your system differs. The memory heuristic (20% OS reserve, 15% write cache) applies identically across models.
- **Shared software limits** (PBHA sync ≤ 1 ms, PBR async ≤ 80 ms, the comfort/derate model) are Storage Virtualize 9.1.2 behaviours, shared across the three FlashSystem models and with SVC SV3.
- **Base service times** and the FS7600/FS5600 saturation points are calibrated/scaled estimates, not published IBM specifications — validate per model in Storage Modeller before committing.

Always confirm current limits on IBM's official *Configuration limits* page for v9.1.x and in the IBM Configurator before quoting a bill of materials.

### 7.4 Synchronous/asynchronous RTT bands

Three correct bands, both thresholds overridable in Advanced:
- **≤ 1 ms → PBHA synchronous** (RPO 0) — sub-millisecond ideal, because the application waits for the far site on every write.
- **1 ms to 80 ms → PBR asynchronous** — 80 ms is IBM's documented upper RTT limit for async PBR.
- **> 80 ms → not supported** — flagged rather than silently sized.

### 7.5 Fixes and remaining minor notes

- **Excel export is repaired (v5).** The generated `.xlsx` previously had a malformed zip central directory (a missing "last-modified date" field) that strict readers — openpyxl, Google Sheets, automated pipelines — rejected, even though Excel/LibreOffice opened it via a lenient fallback. This is fixed; the export now opens cleanly everywhere.
- **Export chart artifact fixed (v5).** The Growth Projection series had an off-by-one range that appended a phantom `(0,0)` point, producing a "boomerang" sweep to the origin. The range is corrected and the projection series draw as straight year-to-year segments.
- **Chart zones follow the selected model (v5).** Previously the shaded bands were fixed to FS9600; they now scale to whichever model is selected, and the selected curve is emphasised with the others faded to context (Section 5.2).
- **Units.** Capacity and bandwidth *ceilings* use decimal conventions (1 GB/s = 125 MB/s, 1 PB = 1000 TB), matching how IBM markets them. Throughput **transfer sizes** are binary KiB (1 KiB = 1024 B) so the byte rate matches Storage Modeller; the result is displayed in decimal MB/s, with the MiB/s equivalent shown alongside for cross-checking.
- **Journal buffer multiplies RTT by 2.** The input is labelled "RTT" (already round-trip) and the buffer formula multiplies by 2 again, so it over-sizes the sync buffer roughly 2×. It errs safe (over-provisioned) and the amounts are tiny, but if you want it exact, either treat the input as one-way latency or drop the ×2.
- **Sensitivity fields are advisory only.** The optional Sequential/Cache-efficiency inputs never change the standard result; they only add a clearly-flagged, ±25%-bounded sensitivity estimate that must be validated against Storage Insights (Section 4, Card 1).

---

## 8. Quick reference: every formula in one place

| Result | Formula (as implemented in v5) |
|---|---|
| Read IOPS | `Total IOPS × read%` |
| Write IOPS | `Total IOPS × (100 − read%)` |
| Read throughput (MB/s) | `read IOPS × read KiB × 1024 ÷ 1,000,000` |
| Write throughput (MB/s) | `write IOPS × write KiB × 1024 ÷ 1,000,000` |
| Total throughput (MB/s) | `read throughput + write throughput`  (÷1000 → GB/s; ÷1.048576 → MiB/s) |
| Mixed-workload saturation | per-model calibrated value (FS9600 1,500,000; FS7600 850,000; FS5600 450,000) |
| Comfortable ceiling | `saturation × comfort%` |
| **Core utilisation** | `IOPS ÷ saturation × 100`  (≈ Storage Modeller "System Core %") |
| Bandwidth utilisation | `GB/s ÷ ceiling GB/s × 100` |
| **Estimated latency** | `base service time ÷ (1 − u)`  where `u = load ÷ saturation` |
| Performance verdict bands | `FITS ≤ 60%` · `MARGINAL 60–80%` · `OVER > 80%` (of core/BW, worst-of) |
| Logical (effective) data | `data today + growth` |
| Physical flash consumed (info) | `logical ÷ DRR` |
| Effective needed with headroom | `logical ÷ target%` |
| Capacity utilisation | `(logical ÷ target%) ÷ array max PBe × 100` |
| Replication mode | `RTT ≤ 1 ms → PBHA sync` · `1–80 ms → PBR async` · `> 80 ms → not supported` |
| Write rate (MB/s) | `write IOPS × write KiB × 1024 ÷ 1,000,000` |
| On-wire (after compression) | `write MB/s ÷ stream compression`  ( ÷125 → Gbps ) |
| Usable link (MB/s) | `link Gbps × 125 × 0.80` |
| Link utilisation | `on-wire MB/s ÷ usable link × 100` |
| Sync journal buffer (MB) | `write MB/s × (RTT ms ÷ 1000) × 2` |
| Usable memory | `node memory × (1 − 0.20)` |
| Write cache | `usable memory × 0.15` |
| Read cache | `usable memory × 0.85` |
| Memory utilisation | `(write cache + journal buffer) ÷ usable memory × 100` |
| Sensitivity factor (optional) | `clamp(1 + (blended cache hit − 0.5) × 0.4, 0.75, 1.25)`, applied to saturation as a separate estimate |
| **Chart: current throughput (GB/s)** | transfer-size aware total ÷ 1000 |
| **Chart: projected IOPS** | `current IOPS × (1 + growth% ÷ 100) ^ years` |
| **Chart: estimated latency** | `base ÷ (1 − u)` where `u = load ÷ saturation` |

---

*Teaching and estimation tool for pre-sales planning — it shows the formulas behind the numbers, and its performance curve is calibrated to IBM Storage Modeller for the FS9600 mixed-workload profile. It is not a substitute for Storage Modeller (authoritative report & graph) or Storage Insights (measured telemetry), and is not court-defensible. Always confirm final performance, capacity, and replication limits with the IBM Configurator and the current Storage Virtualize 9.1.2 configuration-limits documentation before quoting a bill of materials.*
