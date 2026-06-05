# LLM-Based Phishing Email Detection — Group 72

**Applied LLM Term Project** — A cybersecurity Proof of Concept: a two-tier **BERT → Qwen 2.5 7B cascade** that classifies emails as phishing or safe, with a rigorous cross-corpus evaluation against classical baselines.

## Team & Roles

| Member | Role |
|--------|------|
| **Hien** | Dataset curation, BERT fine-tuning (`1_Model-Finetune.ipynb`) |
| **Nguyen** | Cascade pipeline: prompt engineering, Qwen CoT escalation, inference (`2_run_pipeline_test.ipynb`) |
| **LE TRUNG KIEN** (M11415803) | Evaluation, baselines, metric design, error analysis (`3_evaluation_pipeline.ipynb`) |

## Architecture

```
                        ┌──────────────────────┐
 incoming email ──────► │  BERT (fine-tuned)   │── confidence ≥ 0.65 ──► label  (~21 ms)
                        │  bert-base-uncased   │
                        └──────────┬───────────┘
                                   │ confidence < 0.65  (1.2% of emails)
                                   ▼
                        ┌──────────────────────┐
                        │  Qwen 2.5 7B (local) │──► chain-of-thought ──► label  (~29 s)
                        │  SOC-analyst CoT     │
                        └──────────────────────┘
```

## Pipeline — run the notebooks in order

| # | Notebook | What it does | Hardware |
|---|----------|--------------|----------|
| 1 | `1_Model-Finetune.ipynb` | Fine-tunes `bert-base-uncased` on [drorrabin/phishing_emails-data](https://huggingface.co/datasets/drorrabin/phishing_emails-data) (26,946 emails), evaluates on its held-out test split, and documents why the resulting 100% in-distribution score is a **dataset artifact**, not model quality. Published checkpoint: [harrynguyen5/phishing-bert-model](https://huggingface.co/harrynguyen5/phishing-bert-model) | GPU |
| 1b | `1b_Model-Finetune-v2.ipynb` | Controlled re-train applying all documented fixes (body-only input, 512 tokens, artifact removed, fixed seed). Produces the v2 checkpoint for the before/after comparison | GPU |
| 2 | `2_run_pipeline_test.ipynb` | Runs the BERT→Qwen cascade over a **disjoint** 18,650-email test corpus (`Phishing_Email.csv`), producing `pineline_full_predictions.csv` with per-row predictions, confidence, and latency | GPU |
| 3 | `3_evaluation_pipeline.ipynb` | Data-integrity audit, 8-metric panel + bootstrap CIs, McNemar test, three classical baselines (cross-corpus), error analysis, label-suffix experiment | CPU only |

```bash
uv sync          # or: pip install (deps auto-install inside each notebook)
jupyter lab      # run notebooks 1 → 2 → 3
```

Notebook 3 runs standalone on the included `pineline_full_predictions.csv` — the training corpus is auto-downloaded from Hugging Face if `train.parquet` is absent.

## Headline Results (N = 18,460, cross-corpus)

All models are trained on the training corpus and evaluated on a **disjoint** test corpus — identical conditions throughout.

| Model | Accuracy | F1 | Recall | Precision | ROC-AUC |
|-------|----------|-----|--------|-----------|---------|
| Majority | 60.50% | 0.00% | 0.00% | 0.00% | 0.500 |
| Length-only LR | 60.50% | 0.00% | 0.00% | 0.00% | 0.566 |
| TF-IDF + LR | 61.83% | 6.55% | 3.39% | 99.20% | 0.837 |
| **BERT (fine-tuned)** | **80.71%** | **71.22%** | 60.43% | 86.70% | **0.884** |
| **Cascade (BERT + Qwen)** | **81.07%** | **71.94%** | 61.45% | 86.75% | 0.884 |

## Key Findings

- **The training dataset's own benchmark is invalid — and we proved it.** The fine-tuned BERT scores a suspicious 100% on the dataset's held-out test split. Investigation revealed the two classes come from different source corpora: a single string rule (`"ceas-challenge" in text → phishing`) scores **92.7% with no ML at all**. The in-distribution score measures source-corpus detection, not phishing detection — which is why this project evaluates **cross-corpus only**.
- **We diagnosed the flaws, fixed them, and re-measured (v1 → v2).** A controlled retrain (`1b`) on body-only text at 512 tokens removed the artifact (one-rule classifier 92.7% → 48.9% = chance). Cross-corpus result: default-threshold accuracy stayed flat (80.7% → 80.4%), **ruling out the artifact/truncation as the generalisation bottleneck** — the ~80% ceiling is intrinsic distribution shift. But v2 is a genuinely better model: **ROC-AUC 0.884 → 0.919**, and at a tuned threshold its F1 (0.76 → 0.80) and recall (78% → 84%) both improve. The gain is masked at the default operating point by BERT's miscalibration — connecting the fix back to the calibration finding.
- **Pre-trained representations generalise; lexical features don't.** Under identical cross-corpus conditions, TF-IDF collapses to 61.8% (near the majority floor) while BERT holds 80.7% — a 19-point gap that isolates the value of the LLM approach.
- **The cascade earns its keep on uncertain emails** — net +66 correct on 223 escalated rows (McNemar p < 0.0001) — at 29.4 s mean latency per escalation (0.36 s amortised per email).
- **Honest caveats:** BERT's phishing recall is 60.4% (misses ~4 in 10 phishing emails) and it is poorly calibrated (ECE 0.178) — a milestone, not yet a deployable detector.
- **Root causes traced to code:** training truncation at 128 tokens explains the length-degradation (F1 85% → 41% across length deciles); a train/inference input-format mismatch explains much of the cross-corpus gap; partial label-suffix leakage was hypothesised, **tested**, and shown to be a minor factor (−0.4pp in a controlled experiment).

## Known Limitations / Next Iteration

1. Re-train with the prompt prefix and label suffix stripped, `max_length=512`, fixed seed.
2. Use one canonical input format (subject + body) in both training and inference.
3. Post-hoc calibrate BERT (Platt scaling) so the cascade threshold operates on real probabilities.
4. The test corpus's "phishing" is promotional/bulk spam, not spear-phishing — scope is a coarse unwanted-mail detector.

## Artifacts

- Fine-tuned model: [`harrynguyen5/phishing-bert-model`](https://huggingface.co/harrynguyen5/phishing-bert-model)
- Training data: [`drorrabin/phishing_emails-data`](https://huggingface.co/datasets/drorrabin/phishing_emails-data)
- Predictions: `pineline_full_predictions.csv` (18,650 rows: labels, confidence, latency, escalation flag)
- Evaluation reports: generated into `reports/` by notebook 3 (JSON + CSV + PNG, all seeded and reproducible)
