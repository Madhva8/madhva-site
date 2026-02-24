---
title: "How ALICE Processes 3.5 TB/s of Collision Data in Real Time"
description: "Notes on David Rohr's talk at Kruger 2022 on the O2 computing framework, GPU-accelerated reconstruction, and the engineering behind Run 3 data processing."
pubDate: "Feb 24 2026"
heroImage: "/ALICE.png"
---

**Reference:** David Rohr (on behalf of the ALICE Collaboration), *ALICE Data Processing in LHC Run 3*, Kruger 2022: Discovery Physics at the LHC, 7 December 2022. [Slides](https://indico.tlabs.ac.za/event/113/contributions/2395/)

---

## Why This Talk Matters to Me

Working with O2Physics on lxplus every day, I interact with the output of this pipeline constantly — AO2D files, reconstructed tracks, TRD tracklets. But I had only a vague picture of what happens between the raw detector signal and the analysis-ready data I actually open. This talk by David Rohr fills that gap in a very concrete way.

It is also directly relevant to anyone working with TRD data. TRD tracking accounts for nearly **4% of the asynchronous reconstruction compute time** — a non-trivial slice — and is explicitly listed as a target for GPU offloading in the optimistic scenario. Understanding why that matters requires understanding the full pipeline first.

---

## The Core Problem: Too Much Data, Too Fast

Run 3 represents a fundamental shift in how ALICE operates. In Run 1 and 2, the detector used a triggered readout — only record data when something interesting happened. In Run 3 this is gone entirely. ALICE now runs in **continuous readout mode**, recording every collision without a hardware trigger.

The numbers are striking:

- **50,000 Pb-Pb collisions per second** (50 kHz interaction rate)
- **3.5 TB/s** of raw data coming off the detectors
- Tracks from different collisions overlap in the TPC drift time window — there is no clean event boundary

You cannot store 3.5 TB/s. A year of running would produce data volumes that are simply unarchivable. The solution is to compress and reconstruct the data *in real time*, before it ever hits permanent storage, reducing it to roughly **100 GB/s** of compressed reconstructed output.

This is what the O2 project (Online-Offline, or O²) is built to do.

---

## The Two-Phase Processing Architecture

Rohr describes a clean two-phase design that I found clarifying for understanding what happens to data before it reaches us on lxplus.

**Synchronous processing** happens in real time during data taking, on the EPN (Event Processing Node) farm at Point 2. It does three things:

1. **Calibration extraction** — extracting detector calibration constants that were previously computed in separate offline passes. In Run 3 this happens on the fly.
2. **Data compression** — the TPC dominates raw data volume. Tracks are reconstructed and cluster coordinates are stored as *residuals to track models* rather than absolute positions, then entropy-encoded. This is how 3.5 TB/s becomes 100 GB/s.
3. **Event reconstruction** — full TPC tracking for all collisions; tracking in other detectors for only ~1% of events (sufficient for calibration purposes).

**Asynchronous processing** happens after data taking, using the same EPN farm plus the CERN GRID. This is the full reconstruction with final calibration — all detectors, all tracks, producing the AOD files that end up in analysis.

The key insight is that the synchronous phase is completely **dominated by the TPC** — it accounts for 99.37% of CPU time. Everything else (ITS, TRD, TOF, EMCAL) is noise by comparison. This makes the GPU strategy straightforward: put the TPC on the GPU, and you have essentially solved the synchronous processing problem.

---

## The GPU Architecture

The EPN farm consists of roughly **2000 nodes**, each with multiple AMD MI50 GPUs. One MI50 replaces approximately 80 CPU cores for the synchronous TPC workload — a dramatic speedup.

Rohr describes two scenarios for GPU usage:

**Baseline scenario** (fully implemented and running in 2022): TPC clusterization, track finding, track merging, track fitting, and track-model compression on the GPU. This covers the synchronous processing requirement.

**Optimistic scenario** (work in progress): Full barrel tracking — ITS, TPC, TRD, and TOF — all on the GPU. For the asynchronous reconstruction, where the workload is more spread out (TPC is only ~60% of compute time rather than 99%), this matters much more. Without it, the GPUs sit mostly idle during async processing while the CPU is the bottleneck.

The code is written in a generic C++ framework compatible with CUDA (NVIDIA), HIP (AMD), and OpenCL, so the same reconstruction algorithms run on different hardware backends without rewriting.

---

## The TRD Connection

The asynchronous processing breakdown is where TRD appears directly:

| Process | Compute Time |
|---|---|
| TPC Tracking | 61.6% |
| ITS-TPC Matching | 6.1% |
| MCH Clusterization | 6.1% |
| TPC Entropy Decoder | 4.7% |
| ITS Tracking | 4.2% |
| TOF Matching | 4.1% |
| **TRD Tracking** | **3.95%** |

TRD tracking is listed as a target for GPU offloading under the optimistic scenario. Once TPC + ITS + TRD + TOF are all on the GPU, the coverage goes from 61.6% to 85% of async compute on GPU — which is important for making efficient use of the EPN farm during the no-beam periods when async processing runs.

This connects to my own work in an indirect but real way. The TRD tracklets I work with in O2Physics — the Q0/Q1/Q2 charge deposits, the track-to-tracklet matching — are the output of this TRD tracking step. The quality and efficiency of that reconstruction directly affects the training data available for the BDT I'm building for electron-pion separation.

---

## The 2022 Pb-Pb Run: First Real Test

The first stable Pb-Pb beam of Run 3 was processed online on November 18, 2022 — just weeks before this talk. Rohr reports it went smoothly, though the interaction rate was far below the 50 kHz design value, so it was not a real stress test.

What they *did* validate was the 50 kHz scenario using Monte Carlo data replay — confirming more than 20% margin on both CPU/GPU load and memory. So the system was ready; the beam just was not pushed hard yet.

For pp running, the farm handled up to **2.6 MHz inelastic interaction rate** with full processing — the limitation being CPU and host memory rather than GPU compute.

---

## What I Took Away

A few things stood out that I hadn't fully appreciated before:

**The calibration problem is subtle.** TPC space charge distortions — caused by ion buildup in the gas volume — change over time and must be corrected continuously. In Run 1/2 this required separate offline passes over the data. In Run 3 this calibration is extracted online and fed back into the reconstruction in near-real-time. The fact that I can do analysis on well-calibrated tracks without thinking about this is because of a lot of engineering work happening upstream.

**Track-model compression is clever.** Storing cluster positions as residuals to a fitted track is essentially using physics knowledge to compress the data. A random collection of hit positions has high entropy; hit positions constrained to lie near a helix have much lower entropy. The ANS entropy coder then squeezes this further.

**The synchronous/asynchronous split is a practical necessity, not an ideological choice.** During data taking you simply do not have time for full reconstruction of everything. The architecture is designed around that constraint.

---

## Questions I Still Have

- How does the online TRD tracking quality compare to the full offline reconstruction? Is there a significant difference in tracklet efficiency or position resolution?
- The talk mentions TPC space charge distortion correction as the most complicated calibration step. How does this affect TRD-matched tracks specifically — do distortions propagate into the TRD matching?
- With TRD tracking at ~4% of async compute, what is the actual bottleneck within TRD reconstruction — the tracklet finding, the track matching, or something else?

If anyone reading this has answers or pointers to relevant papers, I'd genuinely like to know.
