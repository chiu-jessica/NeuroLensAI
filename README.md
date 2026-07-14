# NeuroLensAI: Pediatric High-Grade Glioma Subtype Classification from MRI

A transfer-learning CNN (ResNet50) that classifies pediatric high-grade glioma
subtype, **high-grade astrocytoma** vs. **diffuse midline glioma (DMG/DIPG)**, 
from a single MRI sequence (T1-contrast-enhanced), using the BraTS-PEDs
imaging dataset with subtype labels derived from CBTN (via Kids First DRC)
and the international DIPG/DMG Registry.

## Motivation

AI-assisted brain tumor classification tools have historically been
developed and validated almost entirely on adult imaging datasets, which
don't reliably transfer to pediatric cases due to differences in tumor
location, imaging characteristics, and disease biology. No FDA-cleared or
widely marketed pediatric-specific brain tumor imaging AI product currently
exists. This project is a small-scale contribution toward that gap.

## Dataset

- **Imaging**: [BraTS-PEDs](https://www.cancerimagingarchive.net/collection/brats-peds/)
  (ASNR-MICCAI BraTS Pediatric Brain Tumor Challenge dataset), via The Cancer
  Imaging Archive (TCIA). Multi-institutional, expert-curated, restricted to
  histologically confirmed high-grade glioma.
- **Labels**: not included in the imaging release. Derived separately:
  - Patients sourced from the **DIPG/DMG Registry (DIPGr)** were labeled
    `DMG_DIPG` directly, since this registry exists specifically for that
    diagnosis.
  - Patients sourced from **CBTN** were linked to diagnosis data via the
    [Kids First Data Resource Center](https://kidsfirstdrc.org) portal,
    matched on external participant ID, filtered to proband-only records
    (to exclude parent/family trio records), and bucketed from the MONDO
    diagnosis field.
  - Patients from DFCI-BCH-BWH-PEDs-HGG, Yale, and Duke had no public
    diagnosis file available and were excluded from the labeled cohort.
- Final two-class labels (`DMG_DIPG`, `high_grade_astrocytoma`) saved as
  `subtype_labels.csv`.

## Notebook Structure

`NeuroLensAI.ipynb` contains the full, unedited development history,
organized under markdown section headers:

1. **Initial Setup** — first data pull attempt (smaller, outdated BraTS-PEDs
   release via Synapse; later found to mismatch the labels file's cohort size)
2. **Adding Labels to Dataset** — CBTN/Kids First linkage, proband filtering,
   diagnosis bucketing, DIPGr direct-labeling, combining into the final
   labels CSV
3. **Failed Model Building Attempt** — using the smaller/outdated imaging
   set, which produced a severely limited and class-imbalanced usable cohort
4. **Getting New, Larger BraTS-PEDs Dataset** — replacing the outdated
   imaging download with the full TCIA release matching the labels file
5. **Final, Successful Model Building Attempt** — dataset loader, stratified
   split, ResNet50 transfer learning, class-weighted training, evaluation
6. **Testing New Slicing Method** — diagnosing and fixing segmentation-label
   convention inconsistencies for tumor-guided slice selection
7. **Grad-CAM Building** — model attention visualization for a correctly
   classified test patient

This history is left in intentionally, including the failed attempt, as a
record of the actual data-engineering process rather than a cleaned-up final
version only.

## Method

- **Model**: ResNet50 (ImageNet-pretrained), frozen backbone, new
  classification head trained for two classes.
- **Input**: one 2D axial slice per patient (T1c sequence), selected using
  the expert segmentation mask to find the slice with the most tumor-core/
  enhancing tissue, rather than a fixed anatomical center slice.
- **Class imbalance**: handled via class-weighted cross-entropy loss,
  reflecting genuine population-level imbalance in the source registries
  (DIPGr structurally over-represents DMG/DIPG cases).
- **Split**: stratified 70/15/15 train/validation/test.

## Results

| Version | Test Accuracy | Notes |
|---|---|---|
| Baseline (center slice, per-slice normalization) | 0.67 | |
| Tumor-guided slice selection + volume-level normalization | **0.88** | |

Full classification report and confusion matrices are in the notebook
(evaluation cells under the "Final, Successful Model Building Attempt"
section).

## Grad-CAM

Model attention was visualized using Grad-CAM for a correctly classified
test patient, with the segmentation mask shown alongside for comparison. See
the "Grad-CAM Building" section of the notebook for the full process,
including the fix required (enabling gradient tracking on the input tensor,
since the backbone is frozen) and the reasoning behind the chosen target
layer.

## Limitations

- Small test set (~24 patients) — reported accuracy should be treated as an
  initial estimate, not a precise measurement. 
- Labeled cohort excludes patients from DFCI-BCH-BWH-PEDs-HGG, Yale, and Duke
  (no public diagnosis data available for these sources).
- Single 2D slice, single MRI sequence (T1c) used per patient — no 3D volume
  or multi-sequence information.
- Possible shortcut-learning consideration: since slice selection is guided
  by enhancing tumor presence, and enhancement correlates with subtype, part
  of model performance may reflect this correlation rather than deeper
  tissue-level differences.
- No external validation cohort.

## Data Access

BraTS-PEDs imaging requires a data use agreement via TCIA. CBTN clinical
data requires a Kids First DRC account. This repository contains code only —
no patient data is included or redistributed, and all authentication tokens
should be treated as placeholders, not real credentials, if found in any
committed notebook version.
