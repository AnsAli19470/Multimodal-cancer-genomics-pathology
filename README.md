
# Multimodal Cancer PAM50 MIL

Multimodal deep learning for PAM50 breast cancer subtype prediction using **TCGA-BRCA** gene expression and whole-slide pathology images. The pipeline combines genomics and histopathology via attention-based multiple-instance learning (MIL) and intermediate fusion, with Grad-CAM visualizations for model interpretability.

---

## Overview

This repository contains a complete, end-to-end pipeline for predicting PAM50 intrinsic subtypes (Basal, Her2, LumA, LumB, Normal) from multimodal data:

- **Genomics:** Bulk RNA-seq gene expression (TPM) from TCGA-BRCA.
- **Pathology:** Whole-slide images (WSI) tiled into patches, embedded with a frozen ResNet-18 backbone.
- **Models:**
  - **Attention Fusion Model:** Combines genomics and pathology features via a cross-attention layer.
  - **Gated-Attention MIL:** Pathology-only model using gated attention pooling (Ilse et al., 2018).
- **Explainability:** Grad-CAM heatmaps on high-attention patches.

The pipeline is designed to run in **Google Colab**, using Google Drive for persistent storage. It is **resumable** — if a download or cache exists, it is skipped.

---

## Key Features

- Automated download of clinical, gene expression, and WSI data from GDC and cBioPortal.
- Resumable data pipeline: skips already downloaded files and cached matrices.
- Patch extraction from WSI with tissue filtering (white ratio threshold).
- Frozen ResNet-18 feature extraction for pathology patches.
- 5-fold cross-validation with leak-free preprocessing (feature selection and scaling fit only on training folds).
- Class-weighted loss to handle imbalanced PAM50 subtypes.
- Grad-CAM visualization for the MIL model.
- Saved model head and class names for inference.

---

## Results

| Model | Accuracy | Macro F1 | Weighted F1 |
|-------|----------|----------|-------------|
| Attention Fusion (genomics + pathology) | **0.75** | 0.63 | 0.74 |
| Gated-Attention MIL (pathology only) | 0.53 | 0.21 | 0.44 |
| Majority-class baseline (LumA) | 0.533 | — | — |

- **Attention Fusion** achieves 75% accuracy, significantly above the majority-class baseline.
- **Pathology-only MIL** performs near baseline, likely due to limited patches per slide and frozen backbone.
- Average attention weights: genomics ≈ 0.71, pathology ≈ 0.29 (fusion model).

---

## Repository Structure

```
multimodal-cancer-pam50-mil/
├── notebooks/
│   └── TCGA_BRCA_v4_FastMIL.ipynb   # Main pipeline
├── trained_mil_model/
│   ├── mil_head.pt                  # Trained gated-attention MIL head
│   └── class_names.json             # Class names in label order
├── README.md
├── requirements.txt
├── .gitignore
└── LICENSE
```

---

## Data Sources

Raw data is **not** included in this repository. The notebook downloads it automatically from:

- **GDC Data Portal** – clinical, gene expression, and WSI files.
- **cBioPortal** – PAM50 subtype (`SUBTYPE`) and AJCC pathologic stage.

All data is publicly available. Please refer to the TCGA data usage policies.

---

## Requirements

### Python packages
Install with:
```bash
pip install -r requirements.txt
```

`requirements.txt` includes:
```
numpy
pandas
pyarrow
scikit-learn
matplotlib
requests
torch
torchvision
openslide-python
Pillow
```

### System packages
OpenSlide is required for WSI reading:
```bash
apt-get install openslide-tools -y
```

### Hardware
- A GPU runtime is recommended for feature extraction and training (Colab GPU works).
- Sufficient Google Drive storage for WSI files and caches.

---

## How to Run

1. Open the notebook `notebooks/TCGA_BRCA_v4_FastMIL.ipynb` in **Google Colab**.
2. Mount your Google Drive. The notebook will create a project folder at:
   ```
   /content/drive/MyDrive/tcga_brca_project
   ```
3. Run the cells sequentially. The pipeline is divided into:
   - **Part A:** Data pipeline (clinical, expression, WSI, patches, features).
   - **Part B:** Attention Fusion model (5-fold CV).
   - **Part C:** Gated-Attention MIL model (5-fold CV, Grad-CAM).
   - **Save Model:** Saves `mil_head.pt` and `class_names.json`.
4. The notebook is resumable: re-running will skip existing downloads and caches.

---

## Model Details

### Attention Fusion Model
- **Inputs:** Top 500 genes by variance (selected per fold) + 2048-dim ResNet-18 pathology features.
- **Architecture:**
  - Gene encoder: Linear → ReLU → Dropout → Linear → ReLU.
  - Pathology encoder: same structure.
  - Cross-attention: query from genomics, keys/values from both modalities.
  - Classifier: concatenation of attended genomics and pathology → MLP.
- **Training:** Adam, weight decay 1e-3, 80 epochs, class-weighted cross-entropy, gradient clipping.

### Gated-Attention MIL
- **Input:** Set of 2048-dim patch embeddings per patient.
- **Architecture:** Gated attention pooling (tanh + sigmoid gates) → bag representation → linear classifier.
- **Training:** Adam, weight decay 1e-3, 80 epochs, class-weighted cross-entropy with label smoothing, early stopping on validation macro-F1.
- **Precomputed embeddings:** ResNet-18 (ImageNet pretrained, frozen) used for speed.

### Class Names
The label encoder sorts classes alphabetically. The order is:
```json
["Basal", "Her2", "LumA", "LumB", "Normal"]
```

---

## Grad-CAM Visualization

Grad-CAM is applied to the MIL model by wrapping the frozen backbone and trained head into a single module. For a given patient bag, gradients are backpropagated to the last convolutional layer of ResNet-18. The resulting heatmaps highlight regions the model considers important for the predicted subtype.

---

## Disclaimer

This project is for **research and educational purposes only**. It is not intended for clinical diagnosis or treatment decisions. The models are trained on public TCGA data and may not generalize to other cohorts.

---

## License

This project is licensed under the MIT License. See `LICENSE` for details.

---

## Acknowledgements

- The Cancer Genome Atlas (TCGA) Research Network.
- cBioPortal for providing curated clinical data.
- OpenSlide for whole-slide image handling.
- PyTorch and torchvision for deep learning tools.
- Ilse et al. (2018) for the gated attention MIL architecture.

---

## Citation

If you use this code in your work, please cite the TCGA-BRCA project and the original attention MIL paper:

```
Ilse, M., Tomczak, J. M., & Welling, M. (2018). Attention-based Deep Multiple Instance Learning. ICML.
```

---

## Contact

For questions or issues, please open an issue on GitHub.
```
