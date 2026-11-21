
# Contact-Conditioned Dynamic Graph Reinforcement Learning for Humanoid Motion Control

**DGRL** · Humanoid whole-body motion tracking · Dynamic graph reinforcement learning · Contact-aware control · Temporal coordination


## Overview

Humanoid robots have a fixed kinematic structure, yet their functional coordination changes across walking, turning, single-leg support, and recovery. **Contact-Conditioned Dynamic Graph Reinforcement Learning (DGRL)** models this changing coordination through three coupled mechanisms:

1. **Contact-conditioned interaction selection:** activate support-related graph edges without changing the fixed skeleton.
2. **State-adaptive relational weighting:** modulate the strength of interactions using motion state and relation-aware attention.
3. **Temporal coordination memory:** retain the history of body-part coordination using a gated recurrent unit (GRU).

The resulting representation conditions a Proximal Policy Optimization (PPO) actor–critic controller.

![Motivation and overview of DGRL](assets/figure_01.png)

*Figure 1. Contact determines relevant interactions, motion state adjusts relation strengths, and memory supplies coordination history.*

## Key Results

| Evaluation | DGRL result | Comparison / context |
|---|---:|---|
| Seen-motion pose error, five seeds ↓ | **36.9 ± 1.4** | Static-GCN: 54.3 ± 2.1 (**32.0% lower**) |
| Seen-motion success rate, five seeds ↑ | **90.0 ± 1.8%** | Static-GCN: 80.0 ± 2.5% |
| Unseen-motion velocity error, five seeds ↓ | **0.119 ± 0.010** | Lowest among evaluated policy-architecture baselines |
| Per-step computation ↓ | **1.2 GFLOPs** | Static-GCN: 1.5; GAT-PPO: 1.6 |
| Recovery time after 10 N push ↓ | **0.42 ± 0.04** | Without dynamic graph: 0.71 ± 0.06 (**40.8% shorter**) |
| Real-robot success ↑ | **22 / 30 trials (73.3%)** | Three evaluated motions; passive safety tether used |

## Architecture

![Overall DGRL architecture](assets/figure_02.png)

*Figure 2. Contact-conditioned graph construction, motion-dependent relation weights, GRU memory, and PPO policy learning.*

### 1. Contact-conditioned graph construction

The relation-attention implementation uses **14 body-indexed graph nodes** (pelvis, leg joints, torso, and arm joints), distinct from the robot's **29 actuated degrees of freedom**. The candidate typed-edge list contains:

- **26 directed kinematic edges** from the fixed skeletal chains;
- **12 directed functional edges** from six predefined pairs;
- **182 directed contact-edge candidates**, gated by predicted node activity.

Thus, computation reserves **220 typed edge slots** (38 always-enabled fixed slots plus 182 contact candidates). Contact activation uses thresholded predicted node scores, rather than directly measured binary foot-contact states:
$$
p_{i,t}=\operatorname{sigmoid}(f_c(X_{i,t})),\qquad b_{i,t}=\mathbb{1}[p_{i,t}>0.5].
$$

A contact-type edge between distinct nodes is enabled only if both endpoint scores are active. Edge validity is implemented through a **mask**; it does not remove parallel fixed kinematic or functional edges.

### 2. State-adaptive relational reasoning

A learned adjacency gate and relation-aware attention jointly determine message strength:
$$
A_t(i,j)=\sigma\bigl(\phi_A(h_{i,t},h_{j,t},c_t,m_t)\bigr),
$$

$$
\alpha_{ij,t}=\frac{\exp(e_{ij,t})}{\sum_{k\in\mathcal N_t(i)}\exp(e_{ik,t})},
\qquad \omega_{ij,t}=A_t(i,j)\alpha_{ij,t}.
$$

Invalid typed edges are excluded from attention and message aggregation. Adjacency gates control effective interaction strength, while attention distributes information among valid neighboring relations.

### 3. Temporal coordination memory and PPO

The graph representation is processed through recurrent memory:
$$
\widetilde M_t=\operatorname{GNN}(V_t,E_t,M_{t-1}),\quad
M_t=\operatorname{GRU}(M_{t-1},\widetilde M_t),\quad
z_t=\operatorname{Pool}(M_t).
$$

The graph-conditioned policy is trained using PPO, together with value, entropy, sparsity, and temporal-consistency terms. The latter penalize redundant adjacency weights and abrupt graph changes.

## Experimental Results

### Main comparison: five independent seeds (Table 3)

Values are mean ± standard deviation. Lower is better except success rate (SR).

| Method | Seen pose ↓ | Seen MPJPE ↓ | SR (%) ↑ | Unseen pose ↓ | Unseen velocity ↓ | Unseen MPJPE ↓ | p vs. DGRL |
|---|---:|---:|---:|---:|---:|---:|---:|
| PPO-MLP | 48.6 ± 1.8 | 0.622 ± 0.021 | 80.4 ± 2.9 | 80.0 ± 2.6 | 0.260 ± 0.015 | 0.700 ± 0.026 | 0.008 |
| PPO-LSTM | 49.4 ± 1.6 | 0.758 ± 0.024 | 80.8 ± 2.1 | 64.7 ± 2.3 | 0.286 ± 0.017 | 0.708 ± 0.029 | 0.016 |
| Static-GCN | 54.3 ± 2.1 | 0.944 ± 0.030 | 80.0 ± 2.5 | 59.1 ± 1.9 | 0.471 ± 0.021 | 0.949 ± 0.035 | 0.008 |
| GAT-PPO | 53.7 ± 1.9 | 0.766 ± 0.027 | 80.8 ± 2.2 | 56.4 ± 1.7 | 0.501 ± 0.024 | 1.105 ± 0.042 | 0.029 |
| **DGRL** | **36.9 ± 1.4** | **0.563 ± 0.018** | **90.0 ± 1.8** | **56.2 ± 1.5** | **0.119 ± 0.010** | **0.676 ± 0.022** | — |

*p-values: Mann–Whitney U test on five-seed seen-motion pose error. MPJPE: mean per-joint position error.*

### Representative single-seed architecture comparison (Table 2a)

| Method | Seen pose ↓ | Seen velocity ↓ | Seen MPJPE ↓ | Seen MPKE ↓ | SR (%) ↑ | Unseen pose ↓ | Unseen velocity ↓ | Unseen MPJPE ↓ | Unseen MPKE ↓ | GFLOPs/step ↓ |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| PPO-MLP | 48.52 | 0.467 | 0.620 | 28.94 | 80.0 | 80.12 | 0.258 | 0.697 | 56.42 | 0.5 |
| PPO-LSTM | 49.32 | 0.188 | 0.761 | 31.83 | 80.0 | 64.59 | 0.283 | 0.704 | 54.67 | 0.8 |
| Static-GCN | 54.42 | 0.233 | 0.946 | 32.48 | 80.0 | 58.97 | 0.469 | 0.952 | 58.74 | 1.5 |
| GAT-PPO | 53.62 | 0.221 | 0.763 | 43.76 | 80.0 | 56.32 | 0.506 | 1.112 | 56.49 | 1.6 |
| **DGRL** | **36.82** | 0.300 | **0.560** | 42.00 | **90.0** | 56.34 | **0.116** | **0.673** | **46.60** | 1.2 |



### Published-controller comparison (Table 2b)

| Method | Seen pose ↓ | Seen velocity ↓ | Seen MPJPE ↓ | Seen MPKE ↓ | SR (%) ↑ | Unseen pose ↓ | Unseen velocity ↓ | Unseen MPJPE ↓ | Unseen MPKE ↓ |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| ExBody2 | 373.90 | 0.256 | 0.684 | 53.03 | 90.97 | 468.20 | 0.416 | 0.654 | 62.78 |
| GMT | 292.10 | 0.187 | 0.526 | 45.28 | 90.07 | 397.10 | 0.354 | 0.574 | 58.89 |
| VMS | 48.15 | 0.176 | 0.499 | 42.59 | **92.68** | 57.92 | 0.216 | 0.568 | 52.95 |
| **DGRL** | **36.82** | 0.300 | 0.560 | **42.00** | 90.00 | **56.34** | **0.116** | 0.673 | **46.60** |

*These are representative re-evaluations on a shared platform. The source does not provide GFLOPs/step values for the published-controller rows.*

### Component ablations (Table 2c)

| Variant | Seen pose ↓ | Seen velocity ↓ | Seen MPJPE ↓ | Seen MPKE ↓ | SR (%) ↑ | Unseen pose ↓ | Unseen velocity ↓ | Unseen MPJPE ↓ | Unseen MPKE ↓ | GFLOPs/step ↓ |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| No contact edge | 67.57 | 1.206 | 2.041 | 66.54 | 80.0 | 86.31 | 1.047 | 2.521 | 68.89 | 1.6 |
| Without GRU | 65.28 | 1.118 | 0.453 | 60.45 | 80.0 | 90.45 | 0.785 | 0.707 | 40.00 | 1.0 |
| No dynamic graph | 81.98 | 1.019 | 1.031 | 52.78 | 80.0 | 93.08 | 1.014 | 1.680 | 56.69 | 1.5 |
| **DGRL** | **36.82** | **0.300** | 0.560 | **42.00** | **90.0** | **56.34** | **0.116** | **0.673** | 46.60 | 1.2 |

### Contact-aligned attention analysis

![Temporal evolution of attention entropy](assets/figure_03.png)

*Figure 3. Coordination entropy changes with gait phase; the plotted “sparsity” is actually a thresholded edge-density fraction, for which larger values mean more active edges.*

![Body-part attention heatmaps](assets/figure_04.png)

*Figure 4. Attention snapshots during double-support and single-support intervals, with corresponding kinematic traces.*

![Contact-aligned relational analysis](assets/figure_05.png)

*Figure 5. Motion variables, representative edge attention, relation-weight patterns, and coordination adaptivity across gait transitions.*

The observed changes in attention allocation need not imply a changing count of thresholded active edges; interaction **strength** can vary even at similar edge density.

## Robustness to Perturbations

![Robustness across perturbation categories](assets/figure_06.png)

*Figure 6. Mean completion ratio, foot-slip metric, and ZMP deviation under delay, dynamics changes, pushes, friction changes, and torque limits.*

### Detailed perturbation results (Table 4)

**MCR** = mean rollout completion ratio, the average fraction of the prescribed rollout horizon executed. **MCR = 1.00 is not the same as a 100% success rate.**

| Perturbation family | Policy | Condition | MCR ↑ | Tracking error ↓ | Foot slip ↓ | ZMP deviation ↓ | Reported failure type |
|---|---|---|---:|---:|---:|---:|---|
| Delay | DGRL | 0 ms | 1.00 | 0.850 | 2.511 | 0.564 | coordination_loss |
| Delay | DGRL | 30 ms | 1.00 | 0.914 | 3.201 | 0.593 | coordination_loss |
| Delay | DGRL | 60 ms | 1.00 | 0.930 | 3.604 | 0.607 | coordination_loss |
| Delay | No dynamic graph | 0 ms | 1.00 | 0.201 | 4.417 | 0.621 | — |
| Delay | No temporal memory | 0 ms | 1.00 | 1.055 | 3.069 | 0.628 | memory_decay |
| Mass/motor/inertia | DGRL | (1.00, 1.00, 1.00) | 1.00 | 0.850 | 2.511 | 0.564 | coordination_collapse |
| Mass/motor/inertia | DGRL | (0.85, 1.00, 1.20) | 1.00 | 0.751 | 2.498 | 0.526 | coordination_collapse |
| Mass/motor/inertia | DGRL | (1.15, 1.20, 1.00) | 0.37 | 0.239 | 1.823 | 0.166 | coordination_loss |
| Mass/motor/inertia | DGRL | (1.00, 0.70, 1.00) | 1.00 | 1.000 | 2.331 | 0.649 | coordination_collapse |
| Mass/motor/inertia | No dynamic graph | (1.00, 1.00, 1.00) | 1.00 | 0.201 | 4.417 | 0.621 | — |
| Mass/motor/inertia | No temporal memory | (1.00, 0.70, 1.00) | 1.00 | 1.002 | 2.505 | 0.621 | memory_failure |
| Push | DGRL | 0 N | 1.00 | 0.850 | 2.511 | 0.564 | coordination_collapse |
| Push | DGRL | 10 N | 1.00 | 0.867 | 2.494 | 0.562 | coordination_collapse |
| Push | DGRL | 15 N | 1.00 | 0.959 | 2.466 | 0.560 | coordination_collapse |
| Push | No dynamic graph | 10 N | 1.00 | 0.233 | 5.947 | 0.590 | — |
| Push | No temporal memory | 15 N | 1.00 | 1.055 | 3.126 | 0.632 | memory_failure |
| Friction | DGRL | μ = 0.3 | 1.00 | 0.786 | 3.301 | 0.631 | coordination_collapse |
| Friction | DGRL | μ = 0.7 | 1.00 | 0.852 | 2.530 | 0.539 | coordination_collapse |
| Friction | DGRL | μ = 1.1 | 1.00 | 0.890 | 2.619 | 0.578 | coordination_collapse |
| Friction | No contact-aware edge | μ = 0.3 | 1.00 | 0.854 | 2.950 | 0.759 | contact_build |
| Friction | No dynamic graph | μ = 0.7 | 1.00 | 0.195 | 4.411 | 0.153 | — |
| Torque limit | DGRL | τ = 0.6 | 1.00 | 0.850 | 2.511 | 0.564 | coordination_collapse |
| Torque limit | DGRL | τ = 1.0 | 1.00 | 0.850 | 2.511 | 0.564 | coordination_collapse |
| Torque limit | DGRL | τ = 0.4 | 1.00 | 0.850 | 2.511 | 0.564 | coordination_collapse |
| Torque limit | No dynamic graph | τ = 0.6 | 1.00 | 0.201 | 4.417 | 0.156 | — |
| Torque limit | No temporal memory | τ = 0.4 | 1.00 | 1.055 | 3.069 | 0.628 | memory_failure |

### Multi-seed robustness (Table 6)

| Condition | Policy | SR (fraction) ↑ | Tracking error ↓ | Foot slip ↓ | ZMP deviation ↓ |
|---|---|---:|---:|---:|---:|
| Nominal | **DGRL** | **0.85 ± 0.04** | 0.45 ± 0.04 | **2.51 ± 0.12** | **0.56 ± 0.02** |
| Nominal | No dynamic graph | 0.75 ± 0.10 | **0.20 ± 0.02** | 4.42 ± 0.20 | 0.62 ± 0.03 |
| Nominal | No contact-aware edges | 0.80 ± 0.15 | 0.91 ± 0.05 | 3.80 ± 0.18 | 0.68 ± 0.03 |
| 10 N push | **DGRL** | **0.80 ± 0.02** | 0.87 ± 0.05 | **2.49 ± 0.12** | 0.66 ± 0.02 |
| 10 N push | No dynamic graph | **0.80 ± 0.14** | **0.24 ± 0.03** | 5.95 ± 0.26 | **0.59 ± 0.03** |
| 10 N push | No contact-aware edges | 0.75 ± 0.16 | 0.91 ± 0.05 | 3.80 ± 0.17 | 0.72 ± 0.04 |
| μ = 0.3 | **DGRL** | **0.85 ± 0.11** | 0.79 ± 0.04 | 3.30 ± 0.16 | **0.63 ± 0.03** |
| μ = 0.3 | No dynamic graph | 0.75 ± 0.22 | **0.20 ± 0.02** | 4.41 ± 0.21 | 0.70 ± 0.03 |
| μ = 0.3 | No contact-aware edges | 0.70 ± 0.26 | 0.85 ± 0.05 | **2.95 ± 0.14** | 0.76 ± 0.04 |

*The advantage depends on the metric and condition; DGRL does not achieve the smallest tracking-error or foot-slip value in every row.*

### Event-aligned contact transitions and 10 N push (Table 7)

| Event | Policy | Stance-foot planar speed ↓ | Peak torso tilt ↓ | Tracking error ↓ | Recovery time ↓ | SR (fraction) ↑ |
|---|---|---:|---:|---:|---:|---:|
| Contact switch | **DGRL** | **0.09 ± 0.01** | **3.8 ± 0.3** | **0.82 ± 0.04** | — | **0.96 ± 0.02** |
| Contact switch | No dynamic graph | 0.18 ± 0.02 | 5.6 ± 0.4 | 1.20 ± 0.06 | — | 0.88 ± 0.10 |
| Contact switch | No contact-aware edges | 0.15 ± 0.02 | 5.0 ± 0.4 | 1.08 ± 0.05 | — | 0.90 ± 0.06 |
| 10 N push | **DGRL** | **0.10 ± 0.01** | **6.2 ± 0.5** | **0.87 ± 0.05** | **0.42 ± 0.04** | **0.98 ± 0.02** |
| 10 N push | No dynamic graph | 0.24 ± 0.03 | 9.1 ± 0.7 | 1.35 ± 0.07 | 0.71 ± 0.06 | 0.92 ± 0.07 |
| 10 N push | No contact-aware edges | 0.19 ± 0.02 | 8.0 ± 0.6 | 1.18 ± 0.06 | 0.63 ± 0.05 | 0.94 ± 0.03 |

### Contact and gait-phase cues (Table 8)

| Configuration | Seen pose ↓ | SR (%) ↑ | Foot slip ↓ | ZMP deviation ↓ |
|---|---:|---:|---:|---:|
| **Full DGRL** | **36.8 ± 1.4** | **90.0 ± 1.8** | 3.51 ± 0.12 | **0.56 ± 0.02** |
| Static adjacency | 54.4 ± 2.1 | 80.0 ± 2.0 | 4.42 ± 0.20 | 0.62 ± 0.03 |
| No contact or phase cue | 67.6 ± 2.5 | 80.0 ± 2.6 | 4.16 ± 0.19 | 0.71 ± 0.04 |
| Phase cue, no contact cue | 45.2 ± 1.8 | 86.0 ± 2.0 | **3.10 ± 0.15** | 0.65 ± 0.03 |

*These four configurations do not constitute a full factorial isolation of input cues and adjacency dynamics.*

## Real-Robot Deployment

![Hardware demonstrations](assets/figure_07.png)

*Figure 7. Qualitative real-robot sequences: (a) back kick, (b) single-leg support, (c) dialogue gestures, (d) swing leg, and (e) EricCamper.*

### Hardware performance (Table 5)

| Motion | Trials | SR ↑ | Falls / terminations ↓ | Joint-angle error ↓ | Base-orientation error ↓ | Foot slip ↓ | Tether engagement ↓ |
|---|---:|---:|---:|---:|---:|---:|---:|
| Single-leg support | 10 | 70.0% | 1 | 0.120 rad | 3.21° | 0.019 m | 10.0% |
| Back kick | 10 | 80.0% | 2 | 0.097 rad | 4.36° | 0.034 m | 20.0% |
| Swing leg | 10 | 70.0% | 1 | 0.108 rad | 3.58° | 0.023 m | 10.0% |
| **Overall / average** | **30** | **73.3%** | **4** | **0.108 rad** | **3.72°** | **0.025 m** | **13.3%** |

The experiments use a **passive fall-arrest tether**. Tether engagement means the tether became load-bearing following a loss of balance or a safety intervention.

## Generalization and Motion Skills

![Zero-shot terrain generalization](assets/figure_08.png)

*Figure 8. Qualitative transfer from flat-ground training to uneven terrain and obstacles without additional training.*

![Diverse martial-art motion skills](assets/figure_09.png)

*Figure 9. IsaacSim demonstrations: left crescent kick, back kick, right crescent kick, and EricCamper.*

## Training Reward Weights (Table 1)

| Group | Reward / penalty item | Weight |
|---|---|---:|
| Limit penalty | DOF position limit | -10.00 |
| Limit penalty | DOF velocity limit | -5.00 |
| Limit penalty | Torque limit | -5.00 |
| Limit penalty | Episode termination | -200.00 |
| Regularization | Torque | -1 × 10⁻⁶ |
| Regularization | Action rate | -0.50 |
| Regularization | Foot orientation | -2.00 |
| Regularization | Sliding | -1.00 |
| Task reward | Body position | 1.00 |
| Task reward | 3D keypoints | 1.60 |
| Task reward | Foot position | 2.10 |
| Task reward | Body rotation | 0.50 |
| Task reward | Body angular velocity | 0.50 |
| Task reward | Body velocity | 0.50 |
| Task reward | DOF position | 0.75 |
| Task reward | DOF velocity | 0.50 |





## Repository Layout and GitHub Display

```text
.
├── remade.md
└── assets/
    ├── figure_01.png
    ├── figure_02.png
    ├── figure_03.png
    ├── figure_04.png
    ├── figure_05.png
    ├── figure_06.png
    ├── figure_07.png
    ├── figure_08.png
    └── figure_09.png
```




## Real  Robot

<img width="514" height="689" alt="92a1455d5969bca86df5cd7f3b0f2cb4" src="https://github.com/user-attachments/assets/f9861300-41f9-4de2-8c75-70733ed457ae" />

<img width="514" height="689" alt="6f398df406d553fcaaca2fc439d5b4cf" src="https://github.com/user-attachments/assets/5aa0a84e-8ffa-49fa-a6fa-c57e17161181" />

<img width="481" height="719" alt="5063a6b1e05b4f959dc8c9f6fe024182" src="https://github.com/user-attachments/assets/35c5c168-be98-4305-b9c0-b4a448202c98" />

