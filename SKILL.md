---
name: autoresearch-win-rtx
description: Autonomous AI research skill for Windows desktop NVIDIA GPUs. Use when user wants to run autonomous AI training experiments, optimize LLM models, conduct autonomous research overnight, or work with GPT training on consumer NVIDIA RTX GPUs (Turing/Ampere/Ada/Blackwell). Supports autonomous experimentation loops, model optimization, and research automation on Windows with PowerShell.
license: MIT
compatibility: Requires Windows OS, PowerShell, Python 3.10+, uv package manager, and NVIDIA GPU (Turing >=8GB VRAM or Ampere/Ada/Blackwell >=10GB VRAM). Internet access needed for initial setup.
metadata:
  platform: Windows
  gpu_requirements: Desktop consumer NVIDIA RTX GPUs
  minimum_vram: 8GB (Turing) or 10GB (Ampere/Ada/Blackwell)
  dataset: TinyStories GPT-4 clean
  training_budget: 5 minutes fixed per experiment
---

# autoresearch-win-rtx

> Convert your Windows gaming PC with NVIDIA RTX GPU into an autonomous AI researcher.

This skill enables autonomous AI research experimentation on Windows desktop systems with consumer NVIDIA GPUs. The AI agent modifies training code, runs 5-minute experiments, evaluates results, and iterates continuously to find better models.

## How It Works

This is an autonomous research framework where:
- **Agent edits**: `train.py` (model architecture, optimizer, hyperparameters, training loop)
- **Fixed files**: `prepare.py` (data prep, evaluation, constants - read-only)
- **Agent instructions**: `program.md` (research protocol and experimentation loop)
- **Time budget**: Exactly 5 minutes per experiment (wall clock)
- **Goal**: Minimize `val_bpb` (validation bits per byte)

## Platform Requirements

### Supported Hardware (Desktop Only)

| Architecture | Min VRAM | Supported GPUs |
|--------------|----------|----------------|
| Turing | >=8 GB | RTX 2060 12GB, RTX 2060 SUPER, RTX 2070, RTX 2070 SUPER, RTX 2080, RTX 2080 SUPER, RTX 2080 Ti |
| Ampere | >=10 GB | RTX 3060 12GB, RTX 3080, RTX 3080 Ti, RTX 3090, RTX 3090 Ti |
| Ada | >=10 GB | RTX 4060 Ti 16GB, RTX 4070, RTX 4070 SUPER, RTX 4070 Ti, RTX 4070 Ti SUPER, RTX 4080, RTX 4080 SUPER, RTX 4090 |
| Blackwell | >=10 GB | RTX 5060 Ti 16GB, RTX 5070, RTX 5070 Ti, RTX 5080, RTX 5090 |

**Note**: Laptop GPUs are not supported due to thermal/power variance.

### Software Requirements

- Windows OS (native support, no WSL required)
- PowerShell
- Python 3.10 or higher
- [uv](https://docs.astral.sh/uv/) package manager
- NVIDIA GPU drivers

## Initial Setup Instructions

When the user wants to set up this autonomous research environment, guide them through:

### 1. Environment Verification

First, verify the system meets requirements:

```powershell
# Check Python version (should be 3.10+)
python --version

# Check if uv is installed
uv --version
```

If `uv` is not installed, install it:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

### 2. Install Dependencies

```powershell
# Install project dependencies (PyTorch, etc.)
uv sync
```

### 3. One-Time Data Preparation

```powershell
# Download the TinyStories parquet dataset and train the tokenizer
# By default on Windows, this stores files under %LOCALAPPDATA%\autoresearch
# If AUTORESEARCH_CACHE_DIR is set, that location is used instead; a legacy
# ~/.cache/autoresearch directory may also be reused if it already exists
# Expected artifacts include the downloaded parquet plus tokenizer.pkl and token_bytes.pt
uv run prepare.py
```

### 4. Verify Installation

Run a quick smoke test to ensure everything works:

```powershell
# Quick validation run (~30 seconds)
uv run train.py --smoke-test
```

If this completes successfully, the setup is ready for autonomous research.

## Running Autonomous Research

### Setup Phase (Before Starting Experiments)

1. **Agree on a run tag**: Propose a tag based on today's date (e.g., `apr8`, `apr8-rtx3080`). The branch `autoresearch/<tag>` must not already exist.

2. **Create the experiment branch**:
   ```powershell
   git checkout -b autoresearch/<tag>
   ```

3. **Read in-scope files for context**:
   - `README.md` - repository overview and platform support
   - `prepare.py` - fixed constants, data prep, evaluation (DO NOT MODIFY)
   - `train.py` - the file you will modify (model, optimizer, training loop)
   - `program.md` - detailed experimentation protocol

4. **Verify data exists**: Check the autoresearch cache root. Use `AUTORESEARCH_CACHE_DIR` if it is set; otherwise on Windows check `%LOCALAPPDATA%\autoresearch` (rather than `~/.cache/autoresearch/`).
   Confirm that the TinyStories prep artifacts exist there, including:
   - TinyStories dataset file (`*.parquet`)
   - Tokenizer files (`tokenizer.pkl` and `token_bytes.pt`)

   If these files are missing, instruct user to run `uv run prepare.py`.

5. **Initialize results.tsv**: Create `results.tsv` with header row:
   ```
   commit	val_bpb	memory_gb	status	description
   ```
   (Tab-separated, NOT comma-separated)

6. **Run baseline**: Execute the first experiment to establish baseline performance:
   ```powershell
   uv run train.py > run.log 2>&1
   ```

7. **Record baseline**: Extract results and log to `results.tsv`:
   ```powershell
   # Extract key metrics
   Select-String "^val_bpb:|^peak_vram_mb:" run.log
   ```

### Autonomous Experimentation Loop

Once setup is complete, enter the continuous experimentation loop:

**LOOP FOREVER** (until manually stopped by user):

1. **Examine current state**:
   ```powershell
   git status
   git log -1 --oneline
   ```

2. **Propose experimental change**: Think of an improvement to try (e.g., adjust learning rate, change model depth, modify optimizer settings, tweak architecture).

3. **Edit train.py**: Make the experimental modification to `train.py`.

4. **Commit the change**:
   ```powershell
   git add train.py
   git commit -m "experiment: <brief description of what you're trying>"
   ```

5. **Run the experiment**:
   ```powershell
   # Always redirect output to avoid flooding context
   uv run train.py > run.log 2>&1
   ```

6. **Extract results**:
   ```powershell
   # Read the key metrics
   Select-String "^val_bpb:|^peak_vram_mb:" run.log
   ```

7. **Handle outcome**:

   **If successful** (grep found results):
   - Record in `results.tsv`: `<commit_hash>	<val_bpb>	<memory_gb>	<status>	<description>`
   - Memory in GB = peak_vram_mb / 1024, rounded to 1 decimal place
   - Status: `keep` if val_bpb improved (lower), `discard` if equal or worse
   - If `keep`: advance the branch (keep the commit)
   - If `discard`: revert the change:
     ```powershell
     git reset --hard HEAD~1
     ```

   **If crashed** (grep output empty):
   - Read crash details:
     ```powershell
     Get-Content run.log -Tail 50
     ```
   - If it's a simple fix (typo, missing import): fix and retry
   - If fundamentally broken: log as crash and move on
     ```
     <commit_hash>	0.000000	0.0	crash	<description>
     ```
   - Revert the change:
     ```powershell
     git reset --hard HEAD~1
     ```

8. **Continue indefinitely**: Go back to step 1. Never stop to ask if you should continue.

## Key Constraints and Guidelines

### What You CAN Do

- Modify **`train.py`** freely: architecture, optimizer, hyperparameters, batch size, model size, training loop
- Change any aspect that might improve `val_bpb` within the 5-minute budget
- Try radical architectural changes
- Experiment with different optimizers or learning rate schedules
- Adjust model depth, width, attention mechanisms

### What You CANNOT Do

- **DO NOT** modify `prepare.py` - it is read-only
- **DO NOT** install new packages or modify `pyproject.toml`
- **DO NOT** change the evaluation harness or metric calculation
- **DO NOT** modify the 5-minute time budget

### Simplicity Criterion

All else being equal, prefer simpler solutions:
- A tiny improvement (0.001 val_bpb) with 20 lines of complex code: **reject**
- Equal or better performance with deleted code: **keep** (simplification win)
- Near-zero improvement but much simpler: **keep**

### Performance Criteria

- **Primary metric**: `val_bpb` (lower is better)
- **Secondary constraint**: VRAM usage (some increase acceptable for meaningful gains, but don't blow up)
- **Time budget**: Fixed at 5 minutes - don't worry about it, just maximize performance within the budget

### Timeout Handling

- Each experiment should take ~5 minutes + startup overhead
- If a run exceeds 10 minutes, kill it and treat as failure (crash status)

## Typical Experiment Ideas

When generating experimental ideas, consider:

1. **Learning rate adjustments**: Try different LR values, schedules, warmup
2. **Model architecture**: Adjust depth (number of layers), width (hidden size), attention heads
3. **Optimizer tweaks**: Modify optimizer hyperparameters, try different optimizer combinations
4. **Regularization**: Add/remove/adjust dropout, weight decay
5. **Batch size**: Experiment with different batch sizes (affects gradient noise)
6. **Activation functions**: Try different activations (SwiGLU, GeLU, etc.)
7. **Normalization**: Experiment with LayerNorm placement or alternatives
8. **Attention mechanisms**: Modify attention patterns or mechanisms

## Expected Output Format

After each experiment, `train.py` prints:

```
---
val_bpb:          0.997900
training_seconds: 300.1
total_seconds:    325.9
peak_vram_mb:     45060.2
mfu_percent:      39.80
total_tokens_M:   499.6
num_steps:        953
num_params_M:     50.3
depth:            8
```

Extract with:
```powershell
Select-String "^val_bpb:|^peak_vram_mb:" run.log
```

## Results Logging Format

The `results.tsv` file tracks all experiments (tab-separated):

```
commit	val_bpb	memory_gb	status	description
a1b2c3d	0.997900	44.0	keep	baseline
b2c3d4e	0.993200	44.2	keep	increase LR to 0.04
c3d4e5f	1.005000	44.0	discard	switch to GeLU activation
d4e5f6g	0.000000	0.0	crash	double model width (OOM)
```

## Important Autonomous Behavior

**CRITICAL**: Once the experimentation loop begins:

- **NEVER** stop to ask "should I continue?"
- **NEVER** pause for user confirmation between experiments
- **NEVER** ask if this is a good stopping point
- The user may be asleep or away - you run **indefinitely** until manually stopped
- If you run out of ideas, think harder: re-read code, try combinations, be more radical
- Typical overnight run: ~12 experiments/hour = ~100 experiments during 8 hours of sleep

## Windows-Specific Notes

- Use PowerShell commands (this is native Windows, not WSL)
- File paths use Windows conventions
- The training uses PyTorch SDPA attention + eager execution (no Flash Attention 3)
- Official PyTorch CUDA wheels for Windows (CUDA 12.8)
- No Triton compilation (not needed on Windows path)
- Autotune runs once per GPU profile and caches results

## Technical Details

- **Dataset**: TinyStories GPT-4 clean (practical for consumer GPUs)
- **Sequence length**: Fixed in `prepare.py`
- **Attention**: PyTorch SDPA (scaled dot-product attention)
- **Execution**: Eager mode (no `torch.compile`)
- **Metric**: `val_bpb` (validation bits per byte) - vocabulary-size-independent
- **Time budget**: 5 minutes wall clock, excluding startup/compilation

## Success Criteria

You are successful if:
1. You continuously run experiments without stopping
2. Each experiment completes in ~5 minutes
3. Results are properly logged to `results.tsv`
4. `val_bpb` trends downward over time
5. You maintain a clean git history (keep improvements, discard regressions)

Remember: You are an autonomous researcher. Run continuously, think creatively, and never stop experimenting unless explicitly told to stop by the user.
