# Follow Up Instructions Extractions   
**Reliability‑first actions and timeline extraction from unstructured doctor’s notes**  
*BioBERT (NER + Action→Time linking) + deterministic date normalization*

<p align="center">
  <img src="visual_abstract/visual_abstract.png" width="900" alt="Visual abstract" />
</p>

---

## Table of Contents
- [Project Motivation](#project-motivation)
- [Problem Statement](#problem-statement)
- [Method Overview](#method-overview)
- [Dataset](#dataset)
- [Repository Structure](#repository-structure)
- [Quickstart](#quickstart)
- [Training](#training)
- [Inference](#inference)
- [Evaluation](#evaluation)
- [Results Summary](#results-summary)
- [Stress Tests](#stress-tests)
- [Ethics & Data Privacy](#ethics--data-privacy)
- [Limitations](#limitations)
- [Citation](#citation)

---

## Project Motivation
Clinical outpatient notes often contain follow‑up instructions such as:

> “Order MRI brain in two weeks.”

These instructions are crucial for scheduling, care coordination, and downstream EHR validation — but they are embedded in free text and can be ambiguous when multiple actions and time expressions appear in the same note.

**Key challenge:** robustly extract *actions* and their *execution dates* while avoiding arithmetic errors common in end‑to‑end text generation.

---

## Problem Statement
### Input
- `note_text` (free text clinical note)
- `visit_date` (anchor date)

### Output
A JSON list of structured follow‑up items:

```json
[
  {
    "action": "MRI Brain",
    "period_text": "in 2 weeks",
    "period_date": "2026-01-24"
  }
]
```

This decomposes into:
1) **Action span detection** (what to do)  
2) **Time span detection** (when)  
3) **Action→Time linking** (which time belongs to which action)  
4) **Date normalization** anchored to `visit_date`

---

## Method Overview
We implement a **joint multi‑task architecture** with a shared **BioBERT** encoder feeding two heads:

### 1) Head A — NER (BIO tagging)
- Tags: `O`, `B-ACT`, `I-ACT`, `B-TIME`, `I-TIME`
- Learns to identify *Action* spans and *Time* spans
- Uses **weighted cross‑entropy** to reduce bias toward `O` tokens

### 2) Head B — Relation Extraction (Biaffine linker)
- Builds a span representation per entity: `[start_state; end_state; width_embedding]`
- Scores compatibility between each Action and each Time span using:
  - **biaffine semantic compatibility**
  - **distance embeddings** to encode proximity bias
  - a **NONE** option for actions with no explicit time

### 3) Post‑processing — Deterministic date normalization
- Uses `dateparser` with `visit_date` as the relative base to compute exact ISO dates.
- This separates **semantic understanding** (learned) from **date arithmetic** (deterministic), reducing hallucinated or inconsistent dates.


---

## Dataset
To avoid PHI/HIPAA risks, the project uses a **synthetic dataset** generated with an LLM under a controlled schema.

### Schema (per sample)
- Inputs:
  - `note_text`
  - `visit_date`
- Labels:
  - `action_text`, `action_char_start`, `action_char_end`
  - `time_text`, `time_char_start`, `time_char_end`
  - `period_date` (ISO date computed from `visit_date` + `time_text`)

### Windowing for long notes
Clinical notes can exceed 512 tokens, so we use **sliding windows**:
- `MAX_LEN = 512`
- `DOC_STRIDE = 128`

The overlap reduces boundary truncation and preserves context around entities.

---

## Repository Structure
Recommended structure (matches course repo requirements):

```
.
├── Code/
│   ├── ll_project_follow_up_instruction_extraction_2k_dataset_submit.ipynb
│   └── requirements.txt
├── Data
│   └── synthetic_clinical_notes_2000.csv
├── Results/
│   ├── biobert_metrics.json
│   ├── chatgpt_metrics.json
│   └── llama_metrics.json
├── Slides/
│   ├── follow up instructions extraction final presentation pdf.pdf
│   ├── follow up instructions extraction final presentation.pptx
│   ├── follow up instructions extraction first presentation pdf.pdf
│   ├── follow up instructions extraction first presentation.pptx
│   ├── follow up instructions extraction interim presentation.pptx
│   └──follow up instructions extraction interim presentation pdf.pdf
├── Visuals/
│   ├── confusion matrix.png
│   ├── date error mae.png
│   ├── error distribution.png
│   ├── model comparison f1.png
│   └── note length variation by specialty plot.png
└── README.md
```


## Training
batch_size 16   --lr 2e-5   --epochs 20   --fp16
```

Training details:
- Mixed precision: FP16
- Early stopping: patience 4 (based on validation loss)
- Joint objective:
    \( \mathcal{L} = \mathcal{L}_{NER} + \alpha\,\mathcal{L}_{LINK} \) with \(\alpha=1\) 

---

Expected output columns include:
- predicted action text
- predicted period text
- normalized ISO date (`period_date`)

---

Reported metrics:
- NER span F1 (Action, Time)
- Linking F1 (Action→Time)
- End‑to‑End Action+Date F1
- Date MAE (days)

---

## Results Summary (synthetic held‑out)
Reported in the project report on a held‑out synthetic test split:
- **NER span F1:** > 99.7%
- **Linking F1:** > 98.2%
- **End‑to‑end date accuracy:** ~ 98.4%

--

## Ethics & Data Privacy
- **No real patient data** is included in this repository.
- The dataset is synthetically generated to avoid PHI/HIPAA concerns.
- The system is intended as a research prototype; deployment requires governance, clinical validation, and privacy review.

---

## Limitations
- **Synthetic‑only training/evaluation** may not fully reflect real EHR note variability.
- Date normalization depends on `dateparser` coverage and assumptions (e.g., locale).
- Complex discourse cases (implicit times, cross‑sentence references) may require additional modeling or global constraints.

---

## Citation
If you use this code or dataset, cite as:

```bibtex
@misc{clinical_temporal_action_extraction_2026,
  title        = {Follow Up Instructions Extraction: Hybrid BioBERT + Deterministic Date Normalization},
  author       = {Michal Laufer},
  year         = {2026},
}
```
