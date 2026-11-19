# TRAC-DTA

Official research-code repository for **TRAC-DTA: Drug–Target Affinity Prediction**.

This repository currently contains the KIBA random-interaction-split training and analysis script for **seed 430**. The script implements a two-stage training workflow: an absolute affinity predictor followed by target-conditioned relational residual refinement. It also exports matched ABS/FINAL predictions and analysis outputs for TCMF and TAPC.

> **Scope and reproducibility note**
>
> The code currently included here is the supplied KIBA seed-430 experiment script. It is not, by itself, a complete reproduction package for every experiment reported in the manuscript (for example, all KIBA seeds, DAVIS scaffold splits, cold-start settings, external Human evaluation, and all baselines). Additional scripts/configurations should be added before claiming that the entire paper is fully reproducible from this repository.

## Repository structure

```text
TRAC-DTA/
├── trac_dta_seed430_rm2_fixed_analysis.py
├── requirements.txt
├── LICENSE
├── .gitignore
├── Data/
│   └── KIBA/
│       └── plm_features_SOTA/
│           └── .gitkeep
└── Pretrained_Data/
    └── .gitkeep
```

The `Data/` and `Pretrained_Data/` directories are intentionally kept empty in this public repository. Dataset files and precomputed pretrained embeddings are not distributed here.

## Method overview

The supplied script includes:

- Molecular and protein graph encoders based on GIN/GAT layers.
- Bidirectional interactive pooling for sequence/graph positional readout.
- Target-Conditioned Modality Fusion (TCMF).
- Target-Conditioned Affinity Prototype Calibration (TAPC).
- A two-stage training procedure:
  1. Train the absolute affinity predictor.
  2. Freeze the absolute prediction pathway and train the target-conditioned relational residual branch.
- Same-target training-reference construction. Validation and test references are selected from the training split only; reference affinity labels are not supplied as prediction inputs.
- Export and diagnostic routines for ABS/FINAL predictions, TCMF weights, TAPC analysis, and result bundles.

Please consult the source code and the manuscript for the exact mathematical definitions and experimental protocol.

## Requirements

### Software

The script imports the following packages:

- Python
- PyTorch
- NumPy
- pandas
- tqdm
- NetworkX
- scikit-learn
- SciPy
- RDKit

Matplotlib is used for optional plot previews in the result-export workflow.

A previously used environment for this project was Python 3.9.25 with PyTorch 2.4.1 and CUDA 11.8. The exact compatible PyTorch wheel depends on the local operating system, NVIDIA driver, and GPU. The code also attempts to use `torch.compile`; if compilation is unavailable or fails, it falls back to eager execution.

### Create an environment

```bash
conda create -n trac_dta python=3.9 -y
conda activate trac_dta
```

Install PyTorch using the official selector for your platform and CUDA configuration:

https://pytorch.org/get-started/locally/

For the previously used PyTorch 2.4.1 / CUDA 11.8 stack, an installation command may be:

```bash
pip install torch==2.4.1 --index-url https://download.pytorch.org/whl/cu118
```

Then install the remaining dependencies:

```bash
pip install -r requirements.txt
```

If you use a CPU-only environment, install the CPU build of PyTorch instead. GPU training is recommended for practical runtime.

## Data preparation

### 1. KIBA source files

Place the following files in `Data/KIBA/`:

```text
Data/
└── KIBA/
    ├── drug_smiles.json
    ├── protein_seq.json
    ├── Y.txt
    └── plm_features_SOTA/
        └── plm_embeddings_sota_final.pt
```

Expected input formats:

- `drug_smiles.json`: JSON object mapping each drug identifier to its SMILES string.
- `protein_seq.json`: JSON object mapping each protein identifier to its amino-acid sequence.
- `Y.txt`: numeric affinity matrix. The script accepts either drug-by-protein or protein-by-drug orientation and transposes the matrix when its dimensions match the latter.
- `plm_embeddings_sota_final.pt`: PyTorch serialized feature dictionary containing the following keys:
  - `drug_id2idx`
  - `prot_id2idx`
  - `drug_feats`
  - `prot_feats`
  - `drug_masks`
  - `prot_masks`

The supplied script expects drug pretrained features with 768 dimensions and protein pretrained features with 2560 dimensions. Drug and protein sequence masks must correspond to the feature tensors and identifier mappings.

### 2. Pretrained features

The pretrained feature file is **not included** in this repository. Place the compatible feature file at:

```text
Data/KIBA/plm_features_SOTA/plm_embeddings_sota_final.pt
```

This repository script consumes precomputed embeddings; it does not generate the pretrained embeddings. The feature extraction procedure, model checkpoint, and preprocessing settings should be documented and made available separately if required for exact reproduction.

### 3. Data access and licensing

Obtain KIBA and any pretrained resources from their original providers or the sources cited in the manuscript. Follow the original dataset and pretrained-model terms of use. Do not upload third-party datasets, pretrained weights, or derived files unless their licenses and redistribution terms permit it.

The public repository intentionally contains empty data directories only. Users must obtain and prepare the required files themselves.

## Training and reproduction

From the repository root, after preparing all required files:

```bash
python -u trac_dta_seed430_rm2_fixed_analysis.py
```

The script uses the KIBA random interaction split and seed 430. The nominal split is 80% training, 10% validation, and 10% test. The script:

1. Loads the KIBA interaction matrix, drug SMILES, protein sequences, and pretrained features.
2. Builds or loads graph caches.
3. Constructs same-target training references.
4. Trains Stage 1 (absolute affinity prediction).
5. Selects the Stage-1 checkpoint using validation performance and initializes Stage 2 from the validation-selected Best-CI checkpoint.
6. Freezes the absolute prediction pathway and trains the relational residual modules.
7. Evaluates and exports ABS/FINAL predictions and TCMF/TAPC analysis outputs.
8. Creates a results bundle under `Data/KIBA/cache_trac_dta_v31_seed430_rm2_fixed_analysis/`.

A normal run performs training and export automatically. To re-export outputs from checkpoints produced by this exact script, use:

```bash
python -u trac_dta_seed430_rm2_fixed_analysis.py --export-only
```

The `--export-only` option requires the corresponding saved checkpoints, split/reference file, and run manifest from a previous run. It does not train a new model.

### Configuration

The main hyperparameters and paths are defined near the beginning of the Python script. The supplied configuration includes:

- Random seed: `430`
- Batch size: `256`
- Stage 1 maximum learning rate: `1.8e-4`
- Stage 1 maximum epochs: `200`
- Stage 2 maximum learning rate: `1e-4`
- Stage 2 maximum epochs: `60`
- Same-target reference count (`K`): `4`
- Number of TAPC prototypes: `8`

Do not change these values if the goal is to reproduce the supplied seed-430 configuration. Any modification should be reported as a changed experimental setting.

### Reproducibility notes

- The script records source-data SHA-256 hashes, split indices, fixed training-reference indices, and a run manifest.
- Validation/test reference features are constructed only from training interactions.
- The validation set is used for checkpoint selection. Test-set monitoring is also present in the supplied training loop; for a strictly blind final evaluation protocol, ensure test results are not used to make training or model-selection decisions.
- GPU model, driver, CUDA/PyTorch build, and library versions can affect runtime and numerical behavior.
- Full-paper reproduction requires the corresponding scripts and configurations for the other datasets, seeds, split protocols, and external evaluations. These are not included in this initial repository package.

## Outputs

The script writes checkpoints, split/reference metadata, prediction exports, diagnostics, and a compressed analysis bundle under:

```text
Data/KIBA/cache_trac_dta_v31_seed430_rm2_fixed_analysis/
```

Graph preprocessing cache files are written under:

```text
Data/KIBA/cache_kiba_max_speed/
```

These generated files are excluded from version control by `.gitignore`.

## Citation

If you use this code, please cite the associated TRAC-DTA paper:

```text
[Insert the final published or preprint citation here.]
```

## License

This repository is distributed under the MIT License. See `LICENSE`.

Third-party datasets, pretrained models, and pretrained feature files are not covered by this repository license; their original terms remain applicable.

## Contact

For questions about the implementation, please contact the corresponding author listed in the associated TRAC-DTA paper.
