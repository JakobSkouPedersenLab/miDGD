# miDGD

**miDGD** is a deep generative model that jointly learns latent representations of mRNA and miRNA expression from bulk RNA-seq data. It uses a shared decoder with modality-specific Negative Binomial output heads and a Gaussian Mixture Model (GMM) prior over the latent space, enabling cross-modal prediction: given mRNA expression of a new sample, the model infers its latent code and predicts miRNA expression.

---

## Repository structure

```
miDGD/
├── base/
│   ├── data/          # Dataset classes (GeneExpressionDatasetCombined, etc.)
│   ├── dgd/           # Core model: DGD class, GMM prior, NB output head
│   ├── engine/        # Training and prediction loops
│   ├── model/         # Decoder architecture
│   ├── plotting/      # Latent-space and loss-curve utilities
│   └── utils/         # Helpers (set_seed, get_activation, …)
├── data/              # TCGA expression matrices and sample annotations
├── models/            # Saved model checkpoints (created at runtime)
├── setup_train.ipynb  # Train a miDGD model from scratch
└── setup_predict.ipynb # Load a trained model and predict miRNA expression
```

---

## Data

The `data/` folder contains a minimal example of stratified subset of the TCGA pan-cancer cohort (3 train + 2 val + 2 test samples per cancer type, 32 types):

| File | Description |
|---|---|
| `tcga_mrna.tsv` | Raw mRNA counts — samples × 18 393 genes |
| `tcga_mirna.tsv` | Raw miRNA counts — samples × 755 miRNAs |
| `tcga_anno.tsv` | Per-sample metadata for all splits |
| `TCGA_train_anno.tsv` | Training-split sample IDs and metadata |
| `TCGA_val_anno.tsv` | Validation-split sample IDs and metadata |
| `TCGA_test_anno.tsv` | Test-split sample IDs and metadata |
| `gene_anno.tsv` | Ensembl ID → gene symbol mapping |

All count matrices are raw (not normalised); library-size scaling is handled inside the model during training.

---

## Installation

```bash
conda create -n midgd python=3.10
conda activate midgd
pip install -r requirement.txt
```

---

## Workflow

### 1. Train — `setup_train.ipynb`

Trains a miDGD model on the TCGA data and saves the checkpoint.

**Key steps inside the notebook:**

1. **Load data** — reads `tcga_mrna.tsv`, `tcga_mirna.tsv`, and the three annotation files from `data/`.
2. **Build datasets and loaders** — wraps the sliced DataFrames in `GeneExpressionDatasetCombined`, which computes per-sample library sizes on the fly.
3. **Define the model** — constructs a `Decoder` (shared MLP backbone) with two `NB_Module` heads (one for mRNA, one for miRNA) and wraps it in a `DGD` object that adds the GMM prior.
4. **Train** — calls `train_midgd`, which alternates between optimising the decoder/GMM parameters and the per-sample latent representations for 100 epochs (configurable).
5. **Save outputs** — writes three files to `models/`:
   - `midgd_minimal.pt` — the full trained model
   - `loss_minimal.pt` — per-epoch loss and correlation metrics
   - `mrna_genes.csv` — ordered list of genes the model was trained on (needed by the predict notebook to select the same features)
6. **Plot** — loss curves (mRNA recon, miRNA recon, GMM) and correlation metrics (R², Pearson, Spearman) across epochs.

---

### 2. Predict — `setup_predict.ipynb`

Loads a trained model and predicts miRNA expression for the held-out test set.

**Run `setup_train.ipynb` first** — this notebook requires `models/midgd_minimal.pt` and `models/mrna_genes.csv`.

**Key steps inside the notebook:**

1. **Load model** — loads `models/midgd_minimal.pt` with `torch.load`.
2. **Load and align data** — reads the same expression files from `data/`, then selects the exact gene columns stored in `models/mrna_genes.csv` so the input dimensionality matches the trained decoder.
3. **Learn new representations** — calls `learn_new_representation`, which freezes the decoder weights and optimises a fresh `RepresentationLayer` for the test samples using only the mRNA reconstruction loss (50 epochs by default). This is the inference-time analogue of the training-time representation optimisation.
4. **Decode to miRNA** — passes the learned latent codes through the frozen decoder to obtain predicted miRNA probabilities, then scales by each sample's total miRNA library size to get predicted counts.
5. **Evaluate** — computes per-miRNA Spearman correlation between predicted and observed counts and plots the distribution.
6. **Visualise latent space** — projects the test representations to 2D with PCA, coloured by cancer type.

---


## Citation

If you use this code, please cite the original miDGD paper.
