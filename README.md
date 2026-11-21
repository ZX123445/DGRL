# Contact-Conditioned Dynamic Graph Reinforcement Learning for Humanoid Motion Control

![DGRL motivation and overview](assets/figures/figure_01.png)

<p align="center">
  <strong>Contact-conditioned interaction topology · State-adaptive relation weights · Temporal coordination memory</strong>
</p>

<p align="center">
  <a href="#method"><b>Method</b></a> ·
  <a href="#experimental-results"><b>Results</b></a> ·
  <a href="#all-figures"><b>Figures</b></a> ·
  <a href="#all-tables"><b>Tables</b></a>
</p>


## Overview

This project presents **Contact-Conditioned Dynamic Graph Reinforcement Learning for Humanoid Motion Control (DGRL)**. Humanoid motion tracking requires coordination among body parts that changes with **contact configurations and motion phases**, even though the underlying kinematic structure remains fixed. DGRL explicitly models **which body parts exchange information**, **how strongly they interact**, and **how recent coordination history affects the current action**.

DGRL combines three mechanisms within a **Proximal Policy Optimization (PPO)** actor–critic framework:

1. **Contact-conditioned graph construction:** preserve skeletal and functional edges while activating additional support-dependent relation slots from predicted node contact scores.
2. **State-adaptive relation-aware reasoning:** use motion- and contact-conditioned adjacency weights together with masked attention for inter-body message passing.
3. **Temporal coordination memory:** use a **gated recurrent unit (GRU)** to encode evolving relational context, supported by adjacency sparsity and temporal consistency regularization.

> **Project status:** This package contains the project overview and its figures and tables. A training or inference implementation is **not** included in this documentation package; no installation or reproduction commands are asserted here.

## Highlights

- **32.0% lower seen-motion pose error** than Static-GCN across five training seeds (36.9 ± 1.4 versus 54.3 ± 2.1).
- **90.0% mean success rate** on seen-motion evaluation, compared with 80.0–80.8% for the four architecture baselines.
- **0.119 ± 0.010 unseen-motion velocity error**, the lowest mean among the evaluated policy architecture baselines.
- **40.8% shorter recovery time** after event-aligned 10 N pushes than the no-dynamic-graph ablation (0.42 s versus 0.71 s).
- **22/30 successful real-robot trials (73.3%)** over three quantitative motion tasks.

## Method

![DGRL overall architecture](assets/figures/figure_02.png)

*Figure 2. Overall architecture: contact-conditioned topology, state-dependent relation weights, GRU memory and PPO optimization.*

### Contact-conditioned interaction graph

The implementation described for relation-attention diagnostics uses **14 body-indexed nodes** (pelvis, legs, torso and arms), distinct from the robot's **29 actuated degrees of freedom**. It reserves **38 fixed directed edge slots** (26 skeletal and 12 functional) plus **182 candidate directed contact slots**, for **220 slots** in total. A node is active when its predicted sigmoid contact score exceeds 0.5; a directed contact-type relation is enabled when both endpoints are active. Fixed edges remain valid independently of contact activations.

### Dynamic relation weighting

DGRL learns continuous state-conditioned adjacency weights and masked relation-attention coefficients. Their product sets effective message strengths. The architecture distinguishes adding/removing *eligible relation slots* from varying the *strengths of enabled messages*.

### Temporal policy optimization

The graph-level state is augmented by recurrent GRU memory, and PPO optimizes the resulting policy. Regularizers constrain the magnitude and step-to-step changes of the continuous adjacency weights.

## Experimental Results

### Five-seed policy comparison

Values below are **mean ± standard deviation across five independent training seeds** (Table 3). SR is reported in percent.

| Method | Seen pose ↓ | Seen MPJPE ↓ | SR (%) ↑ | Unseen pose ↓ | Unseen velocity ↓ | Unseen MPJPE ↓ |
|:--|--:|--:|--:|--:|--:|--:|
| PPO-MLP | 48.6 ± 1.8 | 0.622 ± 0.021 | 80.4 ± 2.9 | 80.0 ± 2.6 | 0.260 ± 0.015 | 0.700 ± 0.026 |
| PPO-LSTM | 49.4 ± 1.6 | 0.758 ± 0.024 | 80.8 ± 2.1 | 64.7 ± 2.3 | 0.286 ± 0.017 | 0.708 ± 0.029 |
| Static-GCN | 54.3 ± 2.1 | 0.944 ± 0.030 | 80.0 ± 2.5 | 59.1 ± 1.9 | 0.471 ± 0.021 | 0.949 ± 0.035 |
| GAT-PPO | 53.7 ± 1.9 | 0.766 ± 0.027 | 80.8 ± 2.2 | 56.4 ± 1.7 | 0.501 ± 0.024 | 1.105 ± 0.042 |
| **DGRL** | **36.9 ± 1.4** | **0.563 ± 0.018** | **90.0 ± 1.8** | **56.2 ± 1.5** | **0.119 ± 0.010** | **0.676 ± 0.022** |

The representative **single-seed** comparisons with policy baselines, published controllers (ExBody2, GMT and VMS), and component ablations are preserved in [Table 2](#table-2-single-seed-comparisons). DGRL does **not** lead every metric: for example, VMS has a higher success rate in the published-controller comparison, and GAT-PPO has a marginally lower unseen pose error in the representative policy-baseline run.

### Contact-aligned relation analysis

![Coordination entropy and graph edge density](assets/figures/figure_03.png)

*Figure 3. Coordination entropy across support phases and the thresholded edge-density fraction (called “sparsity” in the original plot).*

![Relation-attention maps](assets/figures/figure_04.png)

*Figure 4. Body-part attention maps at selected time steps, with motion-state traces.*

![Contact-aligned coordination across policy architectures](assets/figures/figure_05.png)

*Figure 5. Changing relational attention across gait phases and contact switches.*

### Robustness and disturbance recovery

DGRL is evaluated under control delay, mass/motor/inertia scaling, horizontal pushes, friction variation and torque-limit scaling. **Mean rollout completion ratio (MCR) is not the same as trial success rate**: an MCR of 1.00 indicates that the prescribed rollout horizon was executed, not necessarily stable or failure-free tracking. The paper reports metric-specific tradeoffs across ablations and perturbations; see Tables 4 and 6.

![Robustness under dynamics perturbations](assets/figures/figure_06.png)

*Figure 6. Robustness with respect to completion ratio, foot slip and ZMP deviation.*

For matched event-aligned evaluations (Table 7), DGRL recovers from a **10 N push in 0.42 ± 0.04 s**, versus **0.71 ± 0.06 s** without dynamic graph modeling and **0.63 ± 0.05 s** without contact-aware edges.

### Real-robot deployment

A physical humanoid evaluation includes **30 trials** over three quantified motion tasks, with **22 successes (73.3%)**. The overall averages are **0.108 rad** joint-angle error, **3.72°** base orientation error and **0.025 m** foot slip. A passive safety tether was used for fall arrest.

![Real robot whole body motions](assets/figures/figure_07.png)

*Figure 7. Real-robot sequence illustrations: (a) Back Kick, (b) Single-Leg Support, (c) Dialogue Gestures, (d) Swing Leg, (e) EricCamper. These illustrations are qualitative; quantitative results cover three tasks.*

### Qualitative zero-shot terrain and motion demonstrations

The paper presents illustrative evaluation on uneven ground and obstacles after flat-ground training, as well as motion sequences in IsaacSim. These visualizations are **qualitative** and should not be interpreted as numerical benchmark results.

![Uneven terrain and obstacle test](assets/figures/figure_08.png)

*Figure 8. Zero-shot terrain generalization examples.*

![Sequential humanoid movement simulation](assets/figures/figure_09.png)

*Figure 9. Sequential motion demonstrations in IsaacSim.*

## All Figures

All nine figures are displayed inline below.

### Figure 1. Motivation and overview

![Figure 1 — Motivation and overview](assets/figures/figure_01.png)

### Figure 2. Overall architecture

![Figure 2 — Overall architecture](assets/figures/figure_02.png)

### Figure 3. Coordination entropy and edge density

![Figure 3 — Coordination entropy and edge density](assets/figures/figure_03.png)

### Figure 4. Body-part attention snapshots

![Figure 4 — Body-part attention snapshots](assets/figures/figure_04.png)

### Figure 5. Contact-aligned policy comparisons

![Figure 5 — Contact-aligned policy comparisons](assets/figures/figure_05.png)

### Figure 6. Dynamics perturbation robustness

![Figure 6 — Dynamics perturbation robustness](assets/figures/figure_06.png)

### Figure 7. Real-world deployment

![Figure 7 — Real-world deployment](assets/figures/figure_07.png)

### Figure 8. Zero-shot terrain demonstration

![Figure 8 — Zero-shot terrain demonstration](assets/figures/figure_08.png)

### Figure 9. IsaacSim motion sequences

![Figure 9 — IsaacSim motion sequences](assets/figures/figure_09.png)

## All Tables

All **eight tables** are embedded as **high-resolution PNG crops from the study** to preserve mathematical symbols, multi-level headers and precise alignment on GitHub. The five-seed main results are additionally provided as an accessible Markdown table above.

### Table 1. Reward parameter settings

![Table 1. Parameter settings during pre-training](assets/tables/table_01.png)

### Table 2. Single-seed comparisons

![Table 2. Policy architectures, published humanoid controllers and component ablations](assets/tables/table_02.png)

### Table 3. Five-seed statistical evaluation

![Table 3. Five-seed statistical comparison](assets/tables/table_03.png)

### Table 4. Perturbation robustness

![Table 4. Robustness under delay, dynamics changes, pushes, friction and torque limits](assets/tables/table_04.png)

### Table 5. Real-robot evaluation

![Table 5. Quantitative real-robot evaluation](assets/tables/table_05.png)

### Table 6. Multi-seed robustness

![Table 6. Multi-seed robustness against ablations](assets/tables/table_06.png)

### Table 7. Event-aligned motion and recovery

![Table 7. Contact switch and 10 N push recovery analysis](assets/tables/table_07.png)

### Table 8. Contact/phase-cue comparison

![Table 8. Contact and phase input configuration comparison](assets/tables/table_08.png)

## Repository Structure

```text
DGRL/
├── README.md
└── assets/
    ├── figures/
    │   ├── figure_01.png
    │   ├── ...
    │   └── figure_09.png
    └── tables/
        ├── table_01.png
        ├── ...
        └── table_08.png
```

**GitHub upload:** Put `README.md` and the entire `assets/` directory at the **root of the same repository**. Keep filenames and capitalization unchanged. Uploading only `README.md` will break the image links.



## Real  Robot

<img width="514" height="689" alt="92a1455d5969bca86df5cd7f3b0f2cb4" src="https://github.com/user-attachments/assets/f9861300-41f9-4de2-8c75-70733ed457ae" />

<img width="514" height="689" alt="6f398df406d553fcaaca2fc439d5b4cf" src="https://github.com/user-attachments/assets/5aa0a84e-8ffa-49fa-a6fa-c57e17161181" />

<img width="481" height="719" alt="5063a6b1e05b4f959dc8c9f6fe024182" src="https://github.com/user-attachments/assets/35c5c168-be98-4305-b9c0-b4a448202c98" />

