# RIXI + RFdiffusion3 (rfd3) on MSU HPCC (Foundry) — End-to-End Install + Runs

This repo documents a working, reproducible workflow to:
1) install **Foundry** (which provides **RFdiffusion3 `rfd3`**) in a local project folder  
2) download model **checkpoints**  
3) run an **original 1-design de novo length run** (RIXI-length)  
4) run a **RIXI-only loop remodel** job (1 design) via **Slurm GPU**  
5) compare outputs in **PyMOL** (align + color)

> Paths and filenames below match the working setup:
> - Working directory: `/mnt/gs21/scratch/henryaa3/rfdiffusion3`
> - Input files (must exist):
>   - `fold_1_rixi_model_0.pdb`
>   - `rixi.fasta`

---

## Contents
- [1. Folder Setup](#1-folder-setup)
- [2. Install Miniforge Locally](#2-install-miniforge-locally)
- [3. Create Python 3.12 Env + Install Foundry (rfd3)](#3-create-python-312-env--install-foundry-rfd3)
- [4. Download Foundry Checkpoints](#4-download-foundry-checkpoints)
- [5. Make an Activation Helper Script](#5-make-an-activation-helper-script)
- [6. Run 1 Design: De Novo by RIXI Length](#6-run-1-design-de-novo-by-rixi-length)
- [7. RIXI-Only Loop Remodel (A152–A170)](#7-rixi-only-loop-remodel-a152a170)
- [8. Submit Loop Remodel as a GPU Job (1 design)](#8-submit-loop-remodel-as-a-gpu-job-1-design)
- [9. Check Outputs](#9-check-outputs)
- [10. PyMOL: Align + Color Two Models](#10-pymol-align--color-two-models)

---

## 1. Folder Setup

```bash
cd /mnt/gs21/scratch/henryaa3/rfdiffusion3
pwd
ls -lh

You should see at least:

fold_1_rixi_model_0.pdb

rixi.fasta

2. Install Miniforge Locally

This avoids relying on system Python modules and ensures Python ≥ 3.12 is available.

cd /mnt/gs21/scratch/henryaa3/rfdiffusion3

MINIFORGE="/mnt/gs21/scratch/henryaa3/rfdiffusion3/miniforge3"

# Install only if not already present
if [ ! -d "$MINIFORGE" ]; then
  curl -L -o Miniforge3.sh \
    https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh
  bash Miniforge3.sh -b -p "$MINIFORGE"
fi

# Enable conda in current shell
source "$MINIFORGE/etc/profile.d/conda.sh"
conda config --set auto_activate_base false || true
3. Create Python 3.12 Env + Install Foundry (rfd3)

Foundry provides rfd3 via pip. Python ≥ 3.12 is required.

cd /mnt/gs21/scratch/henryaa3/rfdiffusion3
source /mnt/gs21/scratch/henryaa3/rfdiffusion3/miniforge3/etc/profile.d/conda.sh

ENV="/mnt/gs21/scratch/henryaa3/rfdiffusion3/foundry_py312"

if [ ! -d "$ENV" ]; then
  conda create -y -p "$ENV" python=3.12 pip
fi

conda activate "$ENV"

# Avoid contamination from PYTHONPATH if your environment sets it
unset PYTHONPATH || true

python --version
python -m pip install -U pip setuptools wheel

# Install Foundry (includes rfd3 / rf3 / mpnn)
python -m pip install "rc-foundry[all]"

# Verify rfd3 CLI exists
which rfd3
rfd3 --help | head -n 20
4. Download Foundry Checkpoints

This downloads model weights (large: multiple GB).

cd /mnt/gs21/scratch/henryaa3/rfdiffusion3
conda activate /mnt/gs21/scratch/henryaa3/rfdiffusion3/foundry_py312
unset PYTHONPATH || true

CKPTS="/mnt/gs21/scratch/henryaa3/rfdiffusion3/foundry_ckpts"
mkdir -p "$CKPTS"

foundry install base-models --checkpoint-dir "$CKPTS"

# Make checkpoints discoverable for runs
export FOUNDRY_CHECKPOINT_DIRS="$CKPTS"
5. Make an Activation Helper Script

Use this before every run.

cat > /mnt/gs21/scratch/henryaa3/rfdiffusion3/activate_rfd3.sh <<'SH'
#!/bin/bash
source /mnt/gs21/scratch/henryaa3/rfdiffusion3/miniforge3/etc/profile.d/conda.sh
unset PYTHONPATH
conda activate /mnt/gs21/scratch/henryaa3/rfdiffusion3/foundry_py312
export FOUNDRY_CHECKPOINT_DIRS=/mnt/gs21/scratch/henryaa3/rfdiffusion3/foundry_ckpts
SH
chmod +x /mnt/gs21/scratch/henryaa3/rfdiffusion3/activate_rfd3.sh

Test it:

cd /mnt/gs21/scratch/henryaa3/rfdiffusion3
source ./activate_rfd3.sh
which rfd3
echo $FOUNDRY_CHECKPOINT_DIRS
6. Run 1 Design: De Novo by RIXI Length

This generates a new structure only constrained by length (RIXI length from rixi.fasta).

Hydra configs are strict; use +specification.length=... (note the +).

cd /mnt/gs21/scratch/henryaa3/rfdiffusion3
source ./activate_rfd3.sh

L=$(grep -v '^>' rixi.fasta | tr -d '\n' | wc -c)
OUT_ORIG="/mnt/gs21/scratch/henryaa3/rfdiffusion3/out_rfd3_rixi_len_gpu"
mkdir -p "$OUT_ORIG"

rfd3 design \
  out_dir="$OUT_ORIG" \
  inputs=null \
  +specification.length=$L \
  n_batches=1 \
  diffusion_batch_size=1

Expected outputs include:

__0_model_0.cif.gz

__0_model_0.json

7. RIXI-Only Loop Remodel (A152–A170)

This remodels only residues 152–170 while keeping the rest of RIXI’s backbone fixed.

Exact change:

Freeze backbone for residues A1–151 and A171–275

Allow sequence changes only for A152–170

Allow meaningful remodeling with partial_t: 6.0

Create the input YAML:

cd /mnt/gs21/scratch/henryaa3/rfdiffusion3

cat > rixi_loop_remodel_A152_A170_v1.yaml <<'YAML'
rixi_loop_remodel_v1:
  dialect: 2
  input: /mnt/gs21/scratch/henryaa3/rfdiffusion3/fold_1_rixi_model_0.pdb

  contig: A1-275

  select_fixed_atoms:
    A1-151: BKBN
    A171-275: BKBN

  select_unfixed_sequence: A152-170

  partial_t: 6.0
YAML
8. Submit Loop Remodel as a GPU Job (1 design)

This creates a unique output directory so it does not overwrite other runs.

cd /mnt/gs21/scratch/henryaa3/rfdiffusion3

cat > submit_rfd3_rixi_loop_A152_A170_1design.sbatch <<'SBATCH'
#!/bin/bash
#SBATCH --job-name=rfd3RIXIloop
#SBATCH --partition=gpu
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=01:00:00
#SBATCH --output=/mnt/gs21/scratch/henryaa3/rfdiffusion3/log_rfd3_rixi_loop_A152_A170_%j.out
#SBATCH --error=/mnt/gs21/scratch/henryaa3/rfdiffusion3/log_rfd3_rixi_loop_A152_A170_%j.err

set -euo pipefail

cd /mnt/gs21/scratch/henryaa3/rfdiffusion3
source ./activate_rfd3.sh

OUT=/mnt/gs21/scratch/henryaa3/rfdiffusion3/out_rfd3_rixi_loop_A152_A170_1design_run1
mkdir -p "$OUT"

# Confirm GPU availability
nvidia-smi

# Exactly 1 design: n_batches * diffusion_batch_size = 1
rfd3 design \
  out_dir="$OUT" \
  inputs=/mnt/gs21/scratch/henryaa3/rfdiffusion3/rixi_loop_remodel_A152_A170_v1.yaml \
  n_batches=1 \
  diffusion_batch_size=1
SBATCH

sbatch submit_rfd3_rixi_loop_A152_A170_1design.sbatch
9. Check Outputs
List outputs
ls -lh /mnt/gs21/scratch/henryaa3/rfdiffusion3/out_rfd3_rixi_len_gpu
ls -lh /mnt/gs21/scratch/henryaa3/rfdiffusion3/out_rfd3_rixi_loop_A152_A170_1design_run1
Decompress .cif.gz for viewing
cd /mnt/gs21/scratch/henryaa3/rfdiffusion3/out_rfd3_rixi_loop_A152_A170_1design_run1
gunzip -k __0_model_0.cif.gz
View logs
cd /mnt/gs21/scratch/henryaa3/rfdiffusion3
ls -lt log_rfd3_rixi_loop_A152_A170_*.out | head
tail -n 80 log_rfd3_rixi_loop_A152_A170_*.out
10. PyMOL: Align + Color Two Models

If you load both CIFs into PyMOL, use:

Generic (replace names with your object names)
align remodeled_obj, original_obj
hide everything
show cartoon, original_obj
show cartoon, remodeled_obj
color gray70, original_obj
color cyan, remodeled_obj
orient
Example using the object names PyMOL created in our run

PyMOL object names observed:

Remodeled: rixi_loop_remodel_A152_A170_v1_rixi_loop_remodel_v1_0_model_0

Original: _0_model_0_1

align rixi_loop_remodel_A152_A170_v1_rixi_loop_remodel_v1_0_model_0, _0_model_0_1
hide everything
show cartoon, _0_model_0_1
show cartoon, rixi_loop_remodel_A152_A170_v1_rixi_loop_remodel_v1_0_model_0
color gray70, _0_model_0_1
color cyan, rixi_loop_remodel_A152_A170_v1_rixi_loop_remodel_v1_0_model_0
orient

Optional: highlight remodeled region (resi 152–170)

select remodel_loop, (rixi_loop_remodel_A152_A170_v1_rixi_loop_remodel_v1_0_model_0 and resi 152-170)
show sticks, remodel_loop
color orange, remodel_loop
Notes / Common Issues

If you see Hydra errors like: Could not override 'specification.length', use:

+specification.length=<value> (note the +)

Do not run rfd3 design on a dev/login node without a GPU driver; submit via Slurm to a GPU partition.

Foundry checkpoints are large; ensure you have enough storage in your chosen location.

::contentReference[oaicite:0]{index=0}
