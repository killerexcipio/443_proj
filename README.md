# Deep Learning-Based Genomic Prediction with LD-Aware Encoding

This repository contains a research implementation of a deep learning pipeline for **genomic prediction**, where quantitative phenotypes are predicted from genotype marker data. The project adapts an EBMGP-style workflow using Elastic Net-selected SNPs, linkage disequilibrium-aware tokenization, BERT-style embeddings, convolutional feature extraction, and multiple pooling strategies.

The main implementation is in:

```text
pipeline (our work latest).ipynb
```

## Project Summary

Genomic prediction helps estimate phenotypic traits directly from genotype data, which can accelerate breeding and selection decisions in plant and animal genetics. This project explores whether LD-aware deep learning representations can improve prediction performance compared with traditional genomic prediction baselines.

The implemented pipeline converts SNP genotype data into ordered token sequences, enriches those tokens with local LD structure, embeds them using a BERT-style embedding layer, and trains a convolutional regression model to predict standardized phenotype values.

## Key Features

- End-to-end genomic prediction workflow in Jupyter Notebook
- Elastic Net-based SNP feature selection
- LD-aware SNP token construction using adjacent-marker `R²`
- BERT-style token, position, and segment embeddings
- 1D convolutional neural network for phenotype regression
- Pooling strategy comparison:
  - Multi-Scale Attention Pooling / MAP
  - Average Pooling
  - Max Pooling
  - Local Importance Pooling / LIP
- LD threshold ablation across multiple threshold values
- 5-fold cross-validation
- Benchmarking against classical and machine learning baselines
- Evaluation using Pearson correlation, RMSE, R², and MAE

## Repository Structure

```text
443_proj/
├── pipeline (our work latest).ipynb   # Main implementation notebook
└── README.md                          # Project documentation
```

## Methods

### 1. Data Loading

The notebook loads genotype matrices, phenotype labels, and Elastic Net SNP-ranking files for each species and trait.

The project evaluates genomic prediction across three species:

| Species | Traits |
|---|---|
| Rice | SW, FLW, PH, AC, SNPP |
| Sorghum | MO, YLD, HT |
| Holstein Bulls | MS, NMSP, VE |

Each sample represents one individual. The input is a sequence of selected SNP markers, and the target is a standardized quantitative phenotype value.

### 2. Elastic Net SNP Selection

To reduce the dimensionality of genotype data, the pipeline uses precomputed Elastic Net rankings to select the most informative SNPs.

The main experiments use:

```text
Top-T SNPs = 5000
```

This keeps the model focused on high-value genomic markers while reducing the computational cost of training.

### 3. LD-Aware Tokenization

For each selected SNP, the pipeline combines two pieces of information:

1. **Genotype state**
   - `H`: major allele
   - `M`: heterozygous allele
   - `L`: minor allele

2. **LD-derived gap label**
   - `J`: adjacent SNP pair has `R² >= threshold`
   - `Y`: adjacent SNP pair has `R² < threshold`
   - `N`: chromosome boundary

These are combined into categorical SNP tokens such as:

```text
HJ, HY, HN, LJ, LY, LN, MJ, MY, MN
```

This produces a compact SNP vocabulary that preserves both genotype state and local linkage structure.

### 4. BERT-Style Embedding Layer

The tokenized SNP sequences are passed through a BERT-style embedding module. The embedding representation combines:

- Token embeddings
- Positional embeddings
- Segment/type embeddings based on LD labels

This allows the model to learn from both SNP identity and SNP order.

### 5. Convolutional Regression Model

The embedded SNP sequence is processed by a 1D convolutional backbone. The network uses several convolutional blocks with alternating kernel sizes and downsampling.

The final representation is passed into a regression head that predicts one phenotype value per sample.

### 6. Pooling Strategy Comparison

The project compares several pooling strategies inside the convolutional backbone:

| Pooling Method | Description |
|---|---|
| MAP | Attention-style pooling for learning weighted local feature aggregation |
| AVG | Standard average pooling |
| MAX | Standard max pooling |
| LIP | Local importance-based pooling using learned importance weights |

This comparison tests whether learned pooling improves genomic prediction performance over fixed pooling operations.

### 7. LD Threshold Ablation

The notebook evaluates how different LD thresholds affect prediction performance.

Tested thresholds include:

```text
0.2, 0.4, 0.6, 0.8
```

This experiment measures whether stricter or looser LD labeling improves predictive accuracy.

## Model Architecture

The deep learning model follows this high-level structure:

```text
Input SNP Tokens
      ↓
Token + Position + Segment Embeddings
      ↓
1D Convolutional Blocks
      ↓
Pooling Layer
      ↓
Adaptive Average Pooling
      ↓
Linear Regression Head
      ↓
Predicted Phenotype
```

Main architecture settings:

| Component | Value |
|---|---|
| SNP budget | 5000 |
| Token vocabulary size | 9 SNP tokens + padding |
| Type vocabulary size | 3 LD labels + padding |
| Embedding size | 64 |
| Convolution type | 1D convolution |
| Dropout | 0.3 |
| Output | Single continuous phenotype value |

## Training Setup

The model is trained using:

| Setting | Value |
|---|---|
| Loss function | L1 Loss / MAE |
| Optimizer | AdamW |
| Learning rate | `5e-4` |
| Weight decay | `1e-5` |
| Batch size | 32 |
| Epochs | Up to 100 |
| Scheduler | CosineAnnealingLR |
| Validation | 5-fold cross-validation |

Early stopping is used based on validation Pearson correlation.

## Baseline Models

The deep model is compared against classical and machine learning baselines, including:

- RidgeCV as a GBLUP-style linear shrinkage baseline
- BayesianRidge as a Bayesian regression proxy
- Histogram-based gradient boosting / LightGBM-style baseline
- Additional tabular baselines where applicable

All baselines are evaluated using the same SNP subsets and fold splits for fair comparison.

## Evaluation Metrics

The project reports:

| Metric | Purpose |
|---|---|
| Pearson Correlation | Measures predictive agreement between observed and predicted phenotypes |
| RMSE | Measures prediction error magnitude |
| R² | Measures explained variance |
| MAE | Used as the training loss |

Pearson correlation is treated as the primary evaluation metric because genomic prediction often prioritizes ranking and association between predicted and observed phenotype values.

## Results Summary

The experiments show that:

- Pooling strategy has a meaningful effect on model performance.
- LIP and MAX pooling are often competitive with MAP.
- LD threshold choice has a smaller effect than pooling strategy in most experiments.
- The deep model is especially competitive on several rice traits.
- Classical genomic prediction baselines remain strong for some species and traits.
- LD-aware encoding provides a biologically motivated way to structure SNP sequence input for neural networks.

## Installation

Clone the repository:

```bash
git clone https://github.com/killerexcipio/443_proj.git
cd 443_proj
```

Install the required Python packages:

```bash
pip install numpy pandas scikit-learn torch torchmetrics transformers einops matplotlib tqdm
```

Depending on the experiments you run, you may also need:

```bash
pip install xgboost scipy
```

## Running the Project

Open the notebook:

```bash
jupyter notebook "pipeline (our work latest).ipynb"
```

Then run the cells in order.

Before running experiments, update the local data paths inside the notebook so they point to the correct genotype, phenotype, and Elastic Net ranking files.

Expected local data organization may look like:

```text
443_proj/
├── data/
│   ├── rice/
│   ├── sorghum/
│   └── bulls/
├── EN/
│   ├── rice/
│   ├── sorghum/
│   └── bulls/
└── pipeline (our work latest).ipynb
```

## Example Workflow

```python
# Load genotype, phenotype, and Elastic Net ranking files
# Select Top-T SNPs
# Compute LD labels
# Convert SNPs into genotype + LD tokens
# Train EBMGP-style model
# Evaluate with 5-fold cross-validation
# Compare pooling strategies and baseline models
```

## Project Outputs

Depending on which notebook cells are run, the project may generate:

- Cross-validation results
- LD threshold ablation tables
- Pooling comparison tables
- Benchmark comparison tables
- Training and validation loss curves
- Predicted vs. observed scatter plots
- Feature-importance visualizations for selected SNPs

## Technical Highlights

This project demonstrates experience with:

- Deep learning for biological sequence-style data
- PyTorch model development
- Custom `Dataset` and `DataLoader` pipelines
- Genomic feature engineering
- SNP tokenization
- Transformer-style embeddings
- Convolutional neural networks
- Model ablation studies
- Cross-validation
- Regression evaluation
- Scientific machine learning experimentation
- Benchmarking against classical ML models

## Limitations

- The repository depends on external/preprocessed genotype and phenotype files.
- Data files are not bundled directly in the repository.
- The notebook assumes local file paths that may need to be changed before execution.
- Elastic Net SNP rankings are treated as precomputed inputs.
- Full reproducibility requires access to the same processed data and feature-ranking files used during the experiments.

## Future Work

Potential improvements include:

- Refactoring the notebook into reusable Python modules
- Adding command-line scripts for training and evaluation
- Recomputing Elastic Net feature selection within each training fold
- Testing larger SNP budgets
- Adding richer haplotype-aware encodings
- Comparing against additional genomic prediction models
- Adding experiment configuration files
- Publishing saved result tables and plots

## Authors

Md. Rezaur Rahman Bhuiyan  
Jareen Tasneem Khondaker  
Tasfia Zaman

## Acknowledgment

This project was developed as part of a computational biology / genomic prediction research assignment. It adapts and evaluates ideas from EBMGP-style deep genomic prediction methods using LD-aware SNP representations and pooling-based neural architectures.

## License

No license is currently specified. Add a license file before public reuse or distribution.
