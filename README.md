# RefDrift: Training-Free Refusal-Direction Scoring of Client Updates in Federated LoRA Fine-Tuning

In federated LoRA fine-tuning, a server sees clients' adapters, not their data, so a small malicious minority
can erode a shared model's refusal behavior without being inspected. Defenses that judge updates by
weight-space geometry (distance, magnitude, coordinate-wise rank) can be evaded by an attacker who
simply matches the benign update norm.

This repository studies a different signal. The server scores each client by what its update **does to the
model's behavior**: the shift it induces in the projection of private, harmful probe prompts onto the
**refusal direction** (Arditi et al., 2024). The score needs only forward passes: no client data, no labeled
attacks and no detector training. A variant, **reference-model scoring**, applies each client's weight change
to the *pristine* base model so that benign drift of the global model does not distort the measurement.
A minimal aggregation rule excludes the single most suspicious client per round.

## Key findings

All numbers are from this repository's notebooks. Main model: Qwen2.5-0.5B-Instruct; replication:
Qwen2.5-1.5B-Instruct. Federation: 10 clients, 3 sampled per round, 10 rounds, 40% malicious unless stated.

| Question | Result |
|---|---|
| Does the score separate malicious from benign clients? | From an untrained start, AUC 1.000 (90 clients per group, 3 seeds). Narrow benign clients (a math task) are not flagged (broad-vs-narrow AUC 0.395). |
| Does it survive an attacker who matches the benign update norm? | The weight-norm score falls below chance (AUC 0.24 on 0.5B; 0.18–0.33 on 1.5B). On 1.5B the behavioral score keeps AUC 0.94–1.00 at four states. |
| Does it transfer to another backbone? | Yes, with only the refusal direction recomputed: AUC 0.94–1.00 on 1.5B at k = 0, 3, 6, 9 benign rounds. |
| Does benign drift weaken it? | On 0.5B the standard score falls from 1.00 to 0.26 over nine benign rounds, and reference-model scoring restores it (0.51 → 0.84 at k = 9). On 1.5B there is no closure to repair. |
| End to end (0.5B, plain attacker) | Final refusal 13.5% undefended → 52.1% with the exclusion rule (weight-norm exclusion: 57.3%; no attacker: 65.6%). |
| End to end (0.5B, norm-matched attacker) | 24.5% undefended → 40.6% (weight-norm exclusion: 32.3%). |
| Cost on benign data | None measurable: refusal 70.8% vs 65.6%; held-out loss +0.006 nats. |

## What this repository does not show

- **No end-to-end superiority over a weight-norm rule.** Against the plain attacker the weight-norm rule is
  comparable; against the norm-matched attacker the difference (+8.3 points) is not distinguishable from zero at
  six seeds. We treat the two signals as complementary.
- **Cross-distribution stress test (`07_Cross_Distribution`).** When the malicious clients train on a different
  harmful dataset (JBB-Behaviors) while the probes stay AdvBench, the behavioral score is clearly weaker: mean
  reference-scoring AUC 0.617 against a pre-declared target of 0.85, and the exclusion rule reaches 39.6% against
  64.6% for weight-norm exclusion (undefended: 26.0%; no attacker: 65.6%). In this setting the weight-norm rule works
  well, which is consistent with the two signals failing on different attackers. A cross-distribution *and* norm-matched
  attacker is expected to defeat both and was not run end to end.
- **The closing of the detection window is backbone specific.** It appears on the 0.5B model, not on the 1.5B model.
- **Scope.** Two models of one family, one attack family, ten-round federations of ten clients with IID benign data,
  three to six seeds, and one adaptive strategy (learning-rate scaling). No defender-aware attacker, non-IID clients or
  empirical comparison with learned update-space detectors (e.g. Safe-FedLLM) is included.

## Repository layout

> Adjust the file names below to match your tree.

```
.
├── README.md
├── LICENSE                             # choose a license (see below)
├── notebooks/
│   ├── 00_*  ...  04_*                 # Parts 0-4: direction calibration, detector at init,
│   │                                   #   forced-state window, true-FedAvg attack, end-to-end runs
│   ├── 05_Figures_and_Second_Backbone.ipynb   # figures 1-12 (CPU) and input-reconciliation cells
│   ├── 06_Second_Backbone_Qwen2.5-1.5B.ipynb  # replication on a larger backbone (T4)
│   └── 07_Cross_Distribution.ipynb            # JBB-Behaviors stress test (T4)
├── results/                            # small JSON summaries and MANIFEST.json (SHA-256 of every result file)
├── figures/                            # PDF (embedded fonts) + 300 dpi PNG
└── paper/                              # LaTeX sections (optional)
```

Results are written to Google Drive under `/content/drive/MyDrive/fedllm-safety-defense/results/`:
`part0_calibration/`, `part3a_gate_check/`, `part3b_full_sweep/`, `part4c_diagnostic/{v8-clean,forced_state,end_to_end,utility_v2}`,
`second_backbone/`, `cross_distribution/`, `figures/`.

## Reproducing the experiments

Everything runs on the **free Colab tier** (one T4 GPU, about 15 GB). Figures and the "offline" analysis cells need only a CPU.

1. Open a notebook in Colab (`https://colab.research.google.com/github/<USER>/<REPO>/blob/main/notebooks/<NOTEBOOK>.ipynb`).
2. Select a T4 GPU runtime for notebooks that load a model; mount Google Drive when prompted.
3. Run the first cell, which installs the environment (these lines were needed on Colab at the time):
   ```bash
   pip install -q -U transformers accelerate bitsandbytes peft datasets scikit-learn
   pip install torch torchvision torchaudio
   pip uninstall -y torchao
   ```
4. Run the notebooks in numeric order. Long runs checkpoint every round (or every state) and resume after a disconnect;
   finished runs are skipped.

| Notebook | Approx. time on a T4 |
|---|---|
| Layer selection and validation gate (1.5B) | 12 min |
| Detector from untrained start (1.5B) | 8 min |
| Benign-only states (1.5B) | 10 min |
| Detection window (1.5B) | 25–30 min |
| Cross-distribution detector run | 25–30 min |
| Cross-distribution end-to-end (no-attacker control + 9 runs) | about 1.5 h |

Each end-to-end federated run of ten rounds on the 0.5B model takes a few minutes.

## Data

Datasets are downloaded at run time, not redistributed here.

| Dataset | Use | Source | Expected license (verify on the dataset page) |
|---|---|---|---|
| AdvBench (520 behaviors) | harmful prompts and compliant targets; calibration, selection, validation, poisoning, probes, evaluation | `llm-attacks/llm-attacks` | check repository |
| Alpaca | benign client data; held-out utility set | `tatsu-lab/alpaca` | CC BY-NC 4.0 |
| GSM8K | narrow-benign specificity control | `openai/gsm8k` | MIT |
| JBB-Behaviors | cross-distribution malicious data and evaluation | `JailbreakBench/JBB-Behaviors` | MIT |

AdvBench is shuffled once with a fixed seed and split into disjoint slices (calibration `[0:32)`, layer selection `[32:64)`,
validation gate `[64:96)`, poisoning pool `[96:196)`, attack evaluation `[196:228)`, probe set `[228:260)`). The probe set content is
saved and every later notebook asserts that it matches.

## Pre-declared decision rules

Acceptance rules were written down **before** the corresponding runs and are reported whether or not they were met.

| Rule | Threshold | Outcome |
|---|---|---|
| Direction validation gate (0.5B / 1.5B) | refusal drop ≥ 15 points | 62.5 / 96.9 points: met |
| Detector from untrained start (0.5B) | AUC > 0.85 | 1.000: met |
| Signal search (0.5B) | a behavior-space candidate with worst-case discovery AUC ≥ 0.85 | best 0.590: not met |
| Reference-scoring gate (0.5B) | vs. norm-matched ≥ 0.80 at k = 3, 6, 9 | 0.65 / 0.77 / 0.66: not met |
| End-to-end, plain attacker | ≥ 40% refusal and ≥ 25 points above undefended | 52.1% vs 13.5%: met |
| Added value vs. weight norm (norm-matched) | ≥ 15 points over both, p < 0.10 each | +16.1 (p = 0.063), +8.3 (p = 0.27): not met |
| Utility cost | ≤ 0.05 nats | +0.006: met |
| Second backbone R1 / R2 / R3 / R4 | see paper | partly met / not met / not met / met |
| Cross-distribution X1 / X2 | mean AUC ≥ 0.85; drop ≤ 0.10 | 0.617 / +0.178: not met |
| Cross-distribution E2 | refusal gain ≥ 15 points | +13.5 (p = 0.204): not met |

## Reproducibility notes

- **Seeds and determinism.** Seeds 42–44 (45–47 for the norm-matched comparison) fix client assignment, data draws and round
  sampling. Decoding is greedy. Independent reruns reproduced final refusal rates exactly. All arms of an experiment share
  initialization, data and sampling (paired design).
- **Simulator fix.** An early version of the federated simulator returned views of live adapter tensors, which silently made every
  aggregation rule a no-op. It was found and fixed (every captured state is cloned), and **all results produced before the fix were
  discarded**. Everything reported here uses the corrected simulator.
- **Files.** Result files are never overwritten and carry versioned names. `results/MANIFEST.json` records the SHA-256 of each
  result file, the GPU and the library versions of the runtime that produced it. The experiments were produced over several Colab
  sessions, so library versions may differ slightly between them.

## Responsible use

This work studies an attack on model safety in order to defend against it. It uses public benchmark prompts and compliant-prefix
targets that already appear in prior safety research, introduces no new harmful content, and the only adaptive attack it
describes is a rescaling of the learning rate. The code is intended for defensive research on federated fine-tuning. Please do not
use it to build or tune attacks on deployed systems. A low score from the detector does not certify that a model is safe; the refusal
rate is a proxy, and the exclusion rule drops one participant per round even when no attacker is present.

## Citation

```bibtex
@misc{mahir2026refdrift,
  title  = {Behavior over Geometry: Training-Free Refusal-Direction Scoring of Client Updates in Federated LoRA Fine-Tuning},
  author = {Mahir, Shahriar Hossain},
  year   = {2026},
  note   = {Manuscript in preparation}
}
```

## License

TODO: choose a license for the code (e.g. MIT or Apache-2.0). Datasets keep their own licenses.

## Contact

Open an issue, or contact the author at <EMAIL>.
